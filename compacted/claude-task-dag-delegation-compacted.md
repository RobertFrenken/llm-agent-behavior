## Core Findings

- **Without explicit nudging, Claude executes complex multi-step prompts monolithically** -- causing context pollution, zero parallelism, no failure isolation, and poor resumability after compaction/crash.
- **No single mechanism reliably changes Claude's behavior across all prompt complexities.** CLAUDE.md nudges suffer the "nudge paradox": Claude's confidence causes it to skip decomposition and just start executing, especially on the exact prompts where decomposition matters most.
- **Subagents cannot spawn other subagents** -- a critical limitation. An orchestrator subagent can only use the Task/Agent tool for general-purpose workers, not typed custom subagents. Worker specialization must be inlined into the task prompt.
- **Agent teams (experimental) provide true parallelism** with separate 1M-token context windows, but at **2-3x token cost** and with merge conflict risk on write-heavy tasks. Overkill for most day-to-day prompts.
- **Complexity thresholds ("3+ steps", "6+ steps") are arbitrary** and need empirical calibration via observed real prompts. A PostToolUse hook could instrument this.

## Key Insights

- **The tiered approach matches how human PMs think**: small tasks just do, medium tasks plan first, big tasks delegate. This maps to three mechanisms: CLAUDE.md nudge (always-on, lightweight), `/plan-and-execute` skill (medium complexity, user-invocable), orchestrator subagent (high complexity, full delegation).
- **The nudge paradox creates a graceful-degradation argument for the hybrid**: the nudge may be ignored, but the skill and subagent remain available as explicit escalation paths when it fails.
- **External state (`tasks.json`) is the key differentiator** over monolithic execution -- it survives compaction, enables resumption, and provides a progress audit trail. But JSON is harder for Claude to update reliably than markdown; format choice needs testing.
- **Auto-detection vs. explicit invocation is an unsolved tension**: auto-detection risks the nudge paradox; explicit invocation adds friction. The hybrid "nudge suggests, user confirms" may be the sweet spot.
- **Cost-conscious mode needed for constrained environments (OSC)**: decompose for clarity but execute sequentially in the main window instead of spawning subagents, to avoid multiplied token usage.

## Takeaways

- **Build incrementally, cheapest tier first**: Week 1 = CLAUDE.md nudge + `/plan-and-execute` skill (covers ~80% of cases). Week 2 = orchestrator subagent (remaining 20%). Week 3 = evaluate and tune thresholds.
- **Reserve agent teams for genuinely large parallel workloads** (multi-day refactors, full pipeline implementations); invoke manually, never as a default.
- **The CLAUDE.md nudge must include a "do NOT decompose" guardrail** for simple single-purpose prompts -- otherwise Opus 4.6's proactive subagent use causes overuse on trivial tasks.
- **Orchestrator workers need specialization inlined into task prompts** since subagent nesting is blocked -- the orchestrator must embed per-worker system prompt content directly.
- **Instrument before tuning**: add hooks or logging to measure prompt complexity vs. whether decomposition was actually used, before adjusting thresholds.
- **For cross-project deployment via chezmoi**, template the CLAUDE.md nudge so it only activates on machines where Claude Code is installed.
