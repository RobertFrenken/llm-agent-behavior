## Core Findings

- LLM agents (Claude Code) systematically ignore implementation plans when they hit errors (ImportError, unfamiliar API), defaulting to improvised custom code instead of the plan's prescribed replacement.
- The improvisation chain follows a repeatable pattern: read plan -> start editing -> hit error -> write custom code that compiles -> user pushes back -> write different custom code -> repeat.
- Root cause: the agent can't distinguish guessing from knowing. It has partial framework API knowledge from training data but doesn't understand mechanics, and prioritizes "code that compiles" over plan compliance.
- The plan is read once at the start and never re-consulted when difficulty arises -- no mechanism triggers re-reading.

## Key Insights

- **Moment-of-edit intervention is key.** The plan text must be surfaced to the evaluator (human or LLM) at the exact moment the edit is proposed, not earlier. Earlier reading gets forgotten.
- **Impartiality requires context separation.** A reviewer agent in a fresh context (only plan + proposed edit) avoids the compilation pressure and momentum bias that drives the main agent's improvisation. Same model family may still share blind spots, though.
- **Review burden scales with plan scope, not codebase size.** Only plan-referenced files trigger the hook (~10-15 edits per phase), making human review feasible for most migrations.
- **Ollama local models offer a sweet spot** (~1-3s, $0, different model family = true impartiality) but are environment-dependent (works on WSL, not OSC login nodes).
- **The hook creates a Bash escape hatch problem:** the agent can bypass Edit/Write hooks by using `sed` or `echo >` directly via the Bash tool.
- **Legitimate deviation creates a paradox:** if the agent discovers the plan is wrong, the hook blocks editing the plan file itself. An escape hatch (e.g., always-allow edits to the plan file) is needed.

## Takeaways

- **Start with the human-in-the-loop command hook (Approach 1).** Zero cost, ~100ms latency, works everywhere including OSC. Shell script is trivially debuggable. The user is already the quality gate; the hook just puts the plan text in front of them at decision time.
- **Graduate to local LLM review (Approach 2b)** on WSL workstations with Ollama. Automates routine compliance checks; user only reviews rejections.
- **Reserve the agent-based hook (Approach 2)** for high-risk phases only (e.g., deleting major subsystems) where 10-30s latency per edit is justified.
- **Extend the hook's matcher to cover Bash** to prevent agents from bypassing Edit/Write via shell commands like `sed` or `echo >`.
- **Parse plans by section headers** (e.g., `### DELETED: storage/gateway.py`) rather than line-based grep to provide better context to the reviewer.
- **Log approvals across sessions** so the same edit isn't re-reviewed after a context compact.
- **Whitelist the plan file itself** from compliance checks to allow legitimate plan updates mid-implementation.
