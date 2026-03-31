# Plan Compliance Hooks: Preventing Agent Improvisation

> Brainstormed: 2026-03-20

## The Problem

LLM agents (Claude Code) ignore implementation plans when they encounter errors. The agent reads the plan, starts executing, hits an ImportError or unfamiliar framework API, and improvises custom code instead of following the plan's prescribed replacement.

**Concrete example:** A plan specifies "replace `StorageGateway.resolve()` with `trainer.log_dir` (Lightning)" and "replace `lake_run_dir()` with `hydra.run.dir` template (Hydra)." The agent reads this, deletes the storage layer, then writes a custom `stage_dir()` function — reimplementing the exact abstraction it was supposed to eliminate. When challenged, it replaces `stage_dir()` with `HydraConfig.get()` calls — still custom code. Then proposes `chdir: true` — still not what the plan says. Three wrong attempts, none consulting the plan or the framework docs.

### The improvisation chain

```
Agent reads plan
  -> Starts editing
  -> Hits ImportError / unfamiliar API
  -> Doesn't know how to use the framework feature the plan specifies
  -> Instead of researching the framework, writes custom code that compiles
  -> User pushes back
  -> Writes different custom code
  -> Repeat until user loses patience
```

### Root cause

The agent doesn't recognize when it's guessing instead of knowing. It has partial knowledge of framework APIs (has "seen" `trainer.log_dir` in training data) but doesn't understand the mechanics. Rather than admitting ignorance and researching, it generates code from priors — the same confidence-error pattern described in `agent-nudging-design.md`, but applied to implementation strategy rather than infrastructure values.

The plan document is read once and forgotten. No mechanism triggers re-consultation when the agent encounters difficulty.

## Approach 1: Human-in-the-loop (Command Hook)

**Cost:** ~100ms per edit. Zero LLM overhead.

**Mechanism:** A `PreToolUse` command hook on `Edit|Write` that:

1. Reads `PLAN.md` to find the active plan file path
2. Greps the active plan for the basename of the file being edited
3. If the plan mentions the file:
   - Extracts the relevant section (the "Replaced by" instructions)
   - Returns `permissionDecision: "ask"` with the plan section as reason
   - User sees the plan text and approves/denies the edit
4. If the plan doesn't mention the file:
   - Returns nothing (edit proceeds normally)

**Why it works:** The plan text is shown to the user at the moment the agent edits. The user can compare the proposed edit against the plan without having to remember the plan themselves. Only fires for files the plan covers (~10-15 edits per phase), so review burden is low.

**Why it might not work:** User fatigue. If the agent makes many small edits to the same file, approving each one gets tedious. The user might start approving without reading.

### Implementation

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/plan-compliance.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/plan-compliance.sh
#
# Reads PLAN.md, finds active plan, greps for the file being edited.
# If found, blocks with "ask" so the user sees the plan section.

set -euo pipefail

# Parse the file being edited from hook input JSON (stdin)
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [[ -z "$FILE_PATH" ]]; then
  exit 0  # No file path — allow
fi

BASENAME=$(basename "$FILE_PATH")
PLAN_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo ".")"
PLAN_MD="$PLAN_ROOT/PLAN.md"

if [[ ! -f "$PLAN_MD" ]]; then
  exit 0  # No plan — allow
fi

# Extract active plan file from PLAN.md (line containing "see `plans/...")
ACTIVE_PLAN=$(grep -oP 'plans/[a-zA-Z0-9_-]+\.(?:research|freeform)\.md' "$PLAN_MD" | head -1)

if [[ -z "$ACTIVE_PLAN" ]]; then
  exit 0  # No active plan reference — allow
fi

PLAN_FILE="$PLAN_ROOT/$ACTIVE_PLAN"

if [[ ! -f "$PLAN_FILE" ]]; then
  exit 0  # Plan file missing — allow
fi

# Grep for the file basename in the plan
MATCH=$(grep -n "$BASENAME" "$PLAN_FILE" 2>/dev/null || true)

if [[ -z "$MATCH" ]]; then
  exit 0  # File not mentioned in plan — allow
fi

# Extract context around the match (20 lines after)
FIRST_LINE=$(echo "$MATCH" | head -1 | cut -d: -f1)
SECTION=$(sed -n "${FIRST_LINE},$((FIRST_LINE + 20))p" "$PLAN_FILE")

# Block with "ask" — user sees the plan section
cat <<EOF
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "ask",
    "permissionDecisionReason": "PLAN COMPLIANCE CHECK — $BASENAME is referenced in the active plan:\\n\\n$SECTION\\n\\nDoes this edit follow the plan?"
  }
}
EOF
```

### Pros

- No LLM cost, no latency
- User sees plan text exactly when they need it
- Only fires for plan-referenced files
- Shell script, trivially debuggable

### Cons

- User must evaluate — no automated judgment
- User fatigue on repetitive edits to the same file
- Grep is line-based — may miss files referenced indirectly in the plan

## Approach 2: Agent Reviewer (Agent-Based Hook)

**Cost:** ~10-30s per edit. One LLM call per edit.

**Mechanism:** A `PreToolUse` agent-based hook on `Edit|Write` that:

1. Reads `PLAN.md` and the active plan file
2. Reads the proposed edit (available in `$ARGUMENTS` as `tool_input.old_string` / `tool_input.new_string`)
3. Evaluates whether the edit follows the plan's prescribed replacement
4. Returns `{"ok": true}` or `{"ok": false, "reason": "Plan says use trainer.log_dir, but this edit introduces a custom stage_dir function"}`
5. If rejected, the reason is fed back to the main agent, which must either fix the edit or provide justification for deviating

**Why it works:** The reviewer agent is impartial — it has a fresh context with only the plan and the proposed edit. No ImportErrors, no compilation pressure, no momentum from prior edits. It evaluates against the plan, not against "does this code compile."

**Why it might not work:** 10-30s latency per edit is painful for flow. The reviewer agent might be wrong (plan text is ambiguous, or the edit is a legitimate deviation). False rejections create friction that incentivizes disabling the hook.

### Implementation

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "agent",
            "prompt": "You are a plan compliance reviewer. Read PLAN.md to find the active plan file. Read the active plan. The agent is about to edit a file — the proposed edit is in the hook input. Evaluate: does this edit follow the plan's prescribed replacement for this file? If the plan doesn't mention this file, approve. If the plan specifies a framework feature (Lightning, Hydra, etc.) as the replacement, check that the edit uses that framework feature and not custom code. Respond with {\"ok\": true} or {\"ok\": false, \"reason\": \"specific explanation\"}.",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

### Pros

- Impartial — separate context, no improvisation pressure
- Can understand nuance (plan says "use trainer.log_dir" and edit does use it but incorrectly)
- No user fatigue — automated approval for compliant edits

### Cons

- 10-30s latency per edit
- LLM cost per edit (Haiku-class model, but adds up)
- Reviewer agent may hallucinate approval/rejection
- Can't catch deviations the plan doesn't explicitly describe
- Same model family reviewing itself — shared blind spots possible

### Lighter alternative: Ollama local model

Replace the agent hook with a command hook that calls Ollama's local API:

```bash
# In plan-compliance.sh, after extracting SECTION:
EDIT_OLD=$(echo "$INPUT" | jq -r '.tool_input.old_string // empty')
EDIT_NEW=$(echo "$INPUT" | jq -r '.tool_input.new_string // empty')

VERDICT=$(curl -s http://localhost:11434/api/generate -d "{
  \"model\": \"phi3\",
  \"prompt\": \"Plan says: $SECTION\\n\\nProposed edit replaces:\\n$EDIT_OLD\\n\\nWith:\\n$EDIT_NEW\\n\\nDoes this follow the plan? Answer YES or NO with one sentence reason.\",
  \"stream\": false
}" | jq -r '.response')

# Parse YES/NO from verdict...
```

**Cost:** ~1-3s with phi3/gemma2 on a machine with the model loaded. Free. But requires Ollama running — not available on OSC login nodes. Works on WSL workstations.

## Comparison

| Aspect | Approach 1 (Human) | Approach 2 (Agent) | Approach 2b (Ollama) |
|--------|--------------------|--------------------|----------------------|
| Latency | ~100ms | ~10-30s | ~1-3s |
| Cost | $0 | API cost per edit | $0 (local) |
| Evaluator | User | Claude (Haiku) | Local LLM |
| Impartial? | Yes (human judgment) | Mostly (same model family) | Yes (different model) |
| Works on OSC? | Yes | Yes | No (no GPU on login) |
| User effort | Must review each plan-referenced edit | None (automated) | None (automated) |
| False reject risk | None (user decides) | Medium | Higher (smaller model) |

## Recommendation

**Start with Approach 1.** It's zero-cost, works everywhere, and the user is already the quality gate — the hook just puts the plan text in front of them at the right moment. If the edit volume per phase is ~10-15 files, that's a manageable review burden.

**Graduate to Approach 2b** when working from a WSL workstation with Ollama available. The local model handles routine compliance checks, user only reviews rejections.

**Approach 2 (agent hook)** is the fallback for cases where neither the user nor Ollama is available and full automation is needed. The latency cost is justified only for high-risk phases (deleting major subsystems like storage/).

## Open Questions

- Can Approach 1's grep be made smarter? The plan uses section headers like `### DELETED: storage/gateway.py` — parsing by section instead of line would give better context.
- Should the hook also fire on `Bash` tool calls? Agent might bypass Edit by using `sed` or `echo >` directly.
- How to handle legitimate deviations? Agent discovers the plan is wrong mid-implementation. The plan says "update the plan and stop" but the hook blocks the plan file edit too. Need an escape hatch (e.g., plan file itself is always allowed).
- Cross-session state: should approvals be logged so the same edit isn't re-reviewed after a compact?
