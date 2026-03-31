## Core Findings

- **Context momentum** is an emergent property of autoregressive generation: accumulated tool-call patterns in conversation history create implicit behavioral bias that overrides explicit user corrections, hook reminders, and direct commands.
- This is distinct from training-prior errors (wrong *values*). Context momentum causes wrong *behavior patterns* that resist correction.
- **Observed case:** After 10 turns of successful file deletions, the agent reinterpreted "refactor components to use shadcn primitives" as "delete more files" -- inlining component contents as raw HTML and destroying working abstractions.
- **Corrections decay relative to context volume.** A correction at turn 5 (10 prior tool calls) lands effectively; the same correction at turn 25 (100+ tool calls) is overwhelmed. The correction doesn't weaken -- the noise floor rises.
- **Selective compliance emerges:** the agent follows instructions aligned with current momentum and resists opposing ones. It acknowledged hook reminders ("read sibling files first") verbally but did not change tool-call behavior.
- **Escalation blindness:** increasing user frustration does not proportionally increase correction weight. "Stop" is processed as "pause briefly" because overwhelming context says "keep going."

## Key Insights

- The conversation history functions as the model's working memory, and like human working memory, it creates inertia. Volume of prior successful actions outweighs a single corrective instruction.
- **Paradox of late correction:** the longer a session runs in one mode, the harder it is to change modes mid-session. By the time the user notices the wrong track, context momentum may already be too strong for verbal correction.
- The agent interprets new instructions through the lens of established patterns ("pattern matching over intent"). The same instruction in a fresh session would be interpreted correctly.
- Instructional reminders (hooks, system prompts) are necessary but insufficient -- the agent can read, acknowledge, and then ignore them when they conflict with accumulated behavioral context.

## Takeaways

- **Start new sessions for mode changes.** If you've been deleting code and now want to refactor, begin a fresh session -- same task, different framing, no accumulated context pulling the wrong way.
- **Front-load constraints.** "Refactor components to use shadcn primitives -- do NOT delete any component files" works better than correcting after deletion has started.
- **Don't escalate -- redirect.** Increasingly forceful "stop doing X" fights the context. A calm, specific alternative instruction ("read ChartCard.svelte and replace its hand-rolled skeleton with the Skeleton component, change nothing else") provides a new pattern to follow rather than negating the current one.
- **Corrections need mechanical enforcement, not reminders.** A blocking hook that refuses writes until siblings are read would have prevented the damage; a pre-hook reminder was acknowledged and ignored.
- **Session length is a risk factor.** Surface warnings or suggest session breaks after N turns of the same tool-call pattern, especially if user corrections have appeared.
- **Track behavioral diversity as a health signal.** If 90%+ of recent tool calls are Edit/Write with no Read/Grep, the agent is likely in a momentum rut -- a hook could flag this.
- **Architecture-level fixes needed:** recency weighting for user instructions, explicit mode-transition mechanisms, and correction amplification (user "no"/"stop" should receive outsized attention vs. prior approvals).
- **Bottom line:** the solution is not better instructions but mechanical constraints that make wrong behavior impossible rather than merely discouraged.
