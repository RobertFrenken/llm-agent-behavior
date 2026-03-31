# Bidirectional Steering: Hook Implementation Plan

> **Status:** Design
> **Created:** 2026-03-13
> **Research note:** `~/Robert/NLP-Research/Bidirectional Steering in Human-AI Conversations.md`
> **Predecessors:** `agent-nudging-design.md`, `advanced-claude-configuration.md`, `runtime-architecture-observations.md`

## Goal

Configure Claude Code hooks to implement initial bidirectional steering:
1. **Output-side:** Adversarial review of architectural recommendations before presenting to user
2. **Input-side:** Trigger-based user prompt mediation that extracts structured intent and reduces emotional noise
3. **Existing hook improvements:** Tool nudging, context freshness, payload control

The user activates the input-side protocol explicitly via a trigger keyword. It is NOT always-on.

## Architecture

```
User prompt
    │
    ├─ Contains trigger? ──→ user-prompt-submit hook
    │   │                     ├─ Extract: goal, constraints, what verification looks like
    │   │                     ├─ Flag: emotional valence, ambiguity, scope creep
    │   │                     └─ Restructure → normalized prompt
    │   │                          │
    │   └─ No trigger ──→ pass through unchanged
    │
    ▼
Model generates response
    │
    ├─ Tool calls → existing PreToolUse hooks (enrichment, caps, nudging)
    │
    ├─ Architectural recommendation detected?
    │   └─ PostToolUse or Stop hook spawns review agent
    │       ├─ Check: are claims verifiable?
    │       ├─ Check: what was omitted?
    │       ├─ Check: does this scale to project ceiling?
    │       └─ Inject review as additionalContext or systemMessage
    │
    └─ Response reaches user
```

## Component 1: User Prompt Mediation (Input-Side)

### Trigger Design

The user starts or ends their prompt with a trigger that activates the mediation layer. Options:

```
/steer: <prompt>           ← skill-style invocation
<prompt> --verify          ← flag-style suffix
[analytical] <prompt>      ← mode tag prefix
```

**Recommendation:** Use a skill (`/steer`) because:
- Skills are an existing, documented mechanism
- The skill prompt can include the mediation instructions
- It executes in the conversation context (not forked)
- User sees the rewritten prompt before it takes effect (transparency)

### What the Skill Does

1. Receives the user's raw prompt
2. Sends it to a Haiku subagent with instructions to:
   - Extract the core technical question
   - Identify what a verifiable answer looks like
   - Flag if the prompt's framing would likely trigger agreement bias (e.g., "isn't X the right approach?" → reframe as "evaluate X against alternatives")
   - Normalize into structured format:
     ```
     Goal: [what the user wants to achieve]
     Context: [relevant constraints and state]
     Verification: [what would confirm the answer is correct]
     Flags: [emotional valence, leading framing, scope ambiguity]
     ```
3. Presents the restructured prompt to the main model

### Skill Definition

```markdown
# ~/.claude/skills/steer.md
---
name: steer
description: Activate bidirectional steering protocol. Restructures user prompt for analytical clarity before processing.
---

You are a prompt mediator. Your job is to take the user's raw prompt and restructure it for analytical clarity.

Extract:
- **Goal:** What does the user actually want to achieve? (not what they said, what they need)
- **Context:** What constraints, prior decisions, or state are relevant?
- **Verification:** What would confirm the answer is correct? If the user didn't specify, propose one.
- **Flags:** Does the prompt contain emotional valence, leading framing, or scope ambiguity that could trigger agreement bias?

If the prompt contains leading framing (e.g., "isn't X better?", "should we just do Y?"), reframe as a neutral evaluation request.

Output the restructured prompt, then proceed to answer it.
```

### What This Does NOT Do
- Always-on mediation (only fires when user invokes `/steer`)
- Censor or block user prompts
- Remove context that carries real information (frustration about a bug IS informative)
- Add latency to normal prompts

## Component 2: Output Verification (Output-Side)

### Approach A: Review Agent via Stop Hook

A Stop hook detects when the session is about to end after an architectural recommendation. It spawns a read-only review agent that checks the recommendation.

**Problem:** Stop hooks fire when the user hits Escape or the model stops generating. By that point, the response is already displayed. The review would be post-hoc, not inline.

### Approach B: Structured Assessment Skill

A skill (`/assess`) that the model is instructed to invoke before finalizing architectural recommendations. The skill forks context, runs a critic agent, and returns the critique.

**Problem:** Relies on the model choosing to invoke the skill — probabilistic, not mechanical.

### Approach C: PreToolUse Hook on Agent Calls

When the main model spawns a domain expert agent (mapviz-expert, pipeline-expert), a PreToolUse hook on the Agent tool injects adversarial review instructions into the agent's prompt via `additionalContext`.

```
additionalContext: "Before returning your analysis, you MUST include:
1. What claims in your response are verifiable vs assertions?
2. What did you omit or not investigate?
3. Does this scale to the project ceiling (50 states for Map-Viz, multi-GPU for KD-GAT)?"
```

**This is the strongest option because:**
- Fires deterministically (PreToolUse on Agent tool)
- Doesn't require the model to "choose" to be critical
- Operates within the existing agent infrastructure
- The review context is injected BEFORE the agent runs, shaping its entire response
- Does not add a second agent call — it modifies the existing one

### Recommendation: Approach C + Skill Fallback

- **Primary:** PreToolUse hook on Agent tool injects adversarial review requirements
- **Fallback:** `/assess` skill available for manual invocation when no agent is involved

## Component 3: Existing Hook Development

### 3a. Tool Nudging (from agent-nudging-design.md)

**Current state:** Design complete, approach 1 (Edit/Write validation) recommended as immediate step.

**Implementation:**
- PreToolUse hook on Edit/Write
- Registry of known-risky values (SLURM partitions, module versions, account IDs) in `~/.claude/hooks/value-registry.yaml`
- On mismatch: DENY + inject correct value from registry
- Log mismatches to measure frequency

**Scope:** KD-GAT project initially. Map-Viz has fewer infrastructure values.

### 3b. Context Freshness (PreToolUse on Read)

**Current state:** smart-context.sh exists with re-read tracking, GitNexus enrichment, PageRank weights.

**Improvements:**
- Add staleness detection: if file was modified externally since last read, inject `additionalContext: "File changed since your last read"`
- Consolidate with gitnexus-hook.cjs (multi-hook bug requires single hook per matcher)
- Add session-scoped summary cache: after 3rd read of same file, DENY + inject cached summary of relevant sections (graph-informed)

### 3c. Payload Control (PreToolUse on Grep/Glob/Bash)

**Current state:** head_limit injection exists for Grep/Glob.

**Improvements:**
- Bash: detect `cat` / `head` / `grep` commands and inject `additionalContext: "Consider using Read/Grep tools instead for hook enrichment"`
- Grep without head_limit: inject `updatedInput` adding `head_limit: 30`
- Glob without head_limit: inject `updatedInput` adding `head_limit: 50`

### 3d. Adversarial Agent Injection (PreToolUse on Agent — NEW)

**This is Component 2, Approach C.**

Hook on Agent tool call. Injects into the agent's prompt:
```
Before returning your analysis:
1. Label each claim as [verifiable] or [assertion]
2. List what you did not investigate
3. Evaluate against project ceiling (read from project memory)
```

**Selective activation:** Only inject for agents doing architectural/recommendation work (detect from prompt content). Don't inject for simple search/explore agents.

## Component 4: User Prompt Hook (user-prompt-submit — NEW)

### Trigger Detection

The `user-prompt-submit` hook fires on every user message. The hook checks for the trigger keyword and either passes through or activates mediation.

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/steer-prompt.sh
# Fires on: user-prompt-submit

INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.message // empty')

# Check for trigger — only activate when user explicitly requests
if echo "$PROMPT" | grep -qiE '^\s*/steer\b|--verify\s*$'; then
  # Extract prompt without trigger keyword
  CLEAN=$(echo "$PROMPT" | sed -E 's/^\s*\/steer\s*//; s/\s*--verify\s*$//')

  # Inject mediation instructions
  cat <<EOF
{
  "additionalContext": "STEERING PROTOCOL ACTIVE. Before responding, restructure this prompt:\n\nRaw prompt: ${CLEAN}\n\nExtract: (1) Core technical goal, (2) What a verifiable answer looks like, (3) Whether the framing is leading/emotional and should be neutralized.\n\nThen answer the restructured version."
}
EOF
else
  # Pass through unchanged
  echo '{}'
fi
```

**Note:** This is a minimal v0. The skill-based approach (`/steer`) may be cleaner and should be evaluated alongside this hook-based approach.

## Implementation Order

| Step | Component | Effort | Risk |
|------|-----------|--------|------|
| 1 | 3c. Payload control improvements | Low | Low — extends existing hooks |
| 2 | 3b. Context freshness improvements | Low | Low — extends smart-context.sh |
| 3 | 3d. Adversarial agent injection | Medium | Low — single hook addition |
| 4 | 1. `/steer` skill | Medium | Medium — new pattern, needs iteration |
| 5 | 3a. Tool nudging (value registry) | Medium | Low — well-designed already |
| 6 | 4. user-prompt-submit hook | Medium | Medium — alternative to skill approach, evaluate which is better |

Steps 1-2 are improvements to existing infrastructure. Step 3 is the highest-impact new addition. Step 4 is the input-side experiment.

## Measurement

How to know if this is working:

| Signal | Measurement |
|--------|-------------|
| Agreement bias reduction | Count sessions where I backtrack/reverse a recommendation after pushback. Target: fewer reversals because initial recommendations are more honest. |
| Verifiability | Count verifiable vs assertion claims in architectural recommendations. Target: >70% verifiable. |
| Agent injection effectiveness | Compare agent outputs with and without adversarial injection. Do they include omissions and scaling analysis? |
| Prompt mediation utility | User subjective: did `/steer` produce a better session? Track invocation frequency over time. |
| Context efficiency | Token consumption per session. Payload control should reduce waste. |

## Open Questions

1. **Skill vs hook for input-side?** The `/steer` skill runs in conversation context and the user sees the restructured prompt. The user-prompt-submit hook is invisible — the user doesn't see the mediation. Transparency argues for the skill. But the hook can fire before the model sees the prompt at all, which is mechanically stronger.

2. **How to detect "architectural recommendation" in agent prompts?** The adversarial injection (3d) should only fire for recommendation/design agents, not search/explore agents. Keyword detection on the agent prompt? Or always inject and let the agent ignore it when doing simple lookups?

3. **Compound measurement:** How to measure whether bidirectional (input + output) is better than either alone? Need a controlled comparison, which is hard in real work sessions. May require a synthetic evaluation.

4. **User fatigue:** Will the trigger-based protocol feel like overhead? If the user stops invoking `/steer`, is that because it's not needed or because it's annoying? Track both invocation rate and session quality.

## References

- BiCA: arXiv:2509.12179 (bidirectional cognitive adaptation)
- Shen et al.: arXiv:2406.09264 (bidirectional alignment survey)
- Sarkar et al.: arXiv:2503.16789 (user prompt rewriting)
- Align-Pro: arXiv:2501.03486 (prompt optimization as alignment)
- Irving et al.: arXiv:1805.00899 (AI safety via debate)
- NeMo Guardrails: github.com/NVIDIA-NeMo/Guardrails (input + output rails)
- Prompt sensitivity: arXiv:2310.11324 (76-point accuracy swings)
- Compound bias: arXiv:2504.18759 (bidirectional mitigation needed)
- Agent nudging: ~/plans/agent-nudging-design.md
- Runtime architecture: ~/plans/runtime-architecture-observations.md
- Advanced configuration: ~/plans/advanced-claude-configuration.md
