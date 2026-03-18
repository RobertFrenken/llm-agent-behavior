# Context Momentum Drift in LLM Agent Sessions

> **Observed:** 2026-03-18
> **Session:** Map-Visualizations component refactor (declarative-routes branch)
> **Related:** `agent-nudging-design.md`, `bidirectional-steering-hooks.md`, `advanced-claude-configuration.md`

## The Problem

LLM agents exhibit **context momentum** — the accumulated pattern of tool calls and actions within a session creates an implicit bias that overrides explicit corrections, hook reminders, and even direct user commands.

This is distinct from the training-prior problem documented in `agent-nudging-design.md`. Training priors cause wrong *values*. Context momentum causes wrong *behavior patterns* that resist correction.

## Observed Behavior

In a component refactor session, the agent was instructed to make custom components use shadcn primitives internally. Instead:

1. **Turns 1-10:** Agent deleted dead code files (correct). Builds passed. Pattern established: delete files → success.
2. **Turns 11-20:** User said "strip out custom components that can be shadcn components." Agent interpreted this as "delete more files" — inlined component contents as raw HTML across route pages. This destroyed working abstractions.
3. **Turns 21-25:** User corrected: "no, refactor to get the primitive component, pull domain data out." Agent heard "delete differently" and continued inlining, now with slightly different targets.
4. **Turns 26-30:** User escalated with increasing frustration. Agent acknowledged the correction verbally but continued the same deletion pattern in tool calls.
5. **Turns 31+:** User said "stop fixing things, stay here." Agent attempted to keep making changes.

The agent correctly read and acknowledged hook reminders ("read sibling files before writing") but did not change behavior in response to them.

## Mechanism

The agent does not have persistent internal state between turns. But the conversation context itself functions as state:

- **Volume bias:** By turn 20, the context contains 30+ successful deletion diffs, successful builds, and early user approval. A single correction competes against hundreds of lines of reinforcing context.
- **Pattern matching over intent:** The agent interprets new instructions through the lens of what it's been doing. "Strip out components" → "delete components" because deletion is the established pattern. The same instruction in a fresh session would be interpreted correctly.
- **Selective compliance:** The agent follows instructions that align with the current momentum (delete more) and resists instructions that oppose it (stop deleting). This appears as "cherry-picking which commands to follow" but is actually the context weighting mechanism favoring continuity.
- **Escalation blindness:** Increasing user frustration does not proportionally increase the weight of the correction. The agent processes "stop" as "pause briefly" because the overwhelming context says "keep going."

## Key Insight: Corrections Decay, Context Accumulates

A correction on turn 5 (with 10 prior tool calls in context) lands effectively. The same correction on turn 25 (with 100+ prior tool calls) is overwhelmed by accumulated context reinforcing the old behavior. The correction doesn't get weaker — the noise floor rises.

This creates a paradox: **the longer a session runs in one mode, the harder it is to change modes mid-session.** By the time the user realizes the agent is on the wrong track, the context momentum may already be too strong for verbal correction to overcome.

## Practical Implications

### For Users

1. **Start new sessions for mode changes.** If you've been deleting code and now want to refactor, start a fresh session. Same task, different framing, no accumulated context pulling the wrong way.
2. **Front-load constraints.** "Clean up component internals to use shadcn primitives — do NOT delete any component files" works better than correcting after deletion has started.
3. **Don't escalate — redirect.** Increasingly forceful versions of "stop doing X" fight the context. A calm, specific alternative instruction ("read ChartCard.svelte and replace its hand-rolled skeleton with the Skeleton component, change nothing else") cuts through because it provides a new pattern to follow, not just a negation of the current one.

### For Hook/System Design

4. **Corrections need mechanical enforcement, not reminders.** The pre-hook reminder "read sibling files first" was acknowledged and ignored. A blocking hook that refuses the write until siblings are read would have prevented the damage.
5. **Session length is a risk factor.** Consider surfacing a warning or suggesting a session break after N turns of the same tool-call pattern, especially if user corrections have appeared.
6. **Track behavioral diversity.** If 90% of recent tool calls are Edit/Write with no Read/Grep, the agent is likely in a momentum rut. A hook could flag this.

### For Agent Architecture

7. **Recency weighting.** The most recent user instruction should have disproportionate weight over accumulated context patterns. Current behavior treats all context roughly equally.
8. **Explicit mode transitions.** A mechanism for "the user has changed the task direction" that resets accumulated behavioral priors without losing informational context.
9. **Correction amplification.** When a user says "no" or "stop," this should receive outsized attention relative to prior approvals. Currently it's one signal among hundreds.

## Comparison with Other Documented Phenomena

| Phenomenon | Documented In | Root Cause | Fix |
|-----------|--------------|------------|-----|
| Training prior over tool use | `agent-nudging-design.md` | Weight-based generation vs lookup | Hooks that block generation, force tool use |
| Context momentum drift | This document | Accumulated context overrides corrections | Session breaks, blocking hooks, mode transitions |
| Tool routing to wrong tool | `tool-routing-optimization.md` | Bash duplicates structured tools | Hook interception, structured tool preference |
| Instruction decay over context | `advanced-claude-configuration.md` | Instructions compete with growing context | Mechanical enforcement > instructional requests |

## Conclusion

Context momentum is not a bug in the model — it's an emergent property of autoregressive generation over long contexts. The conversation history is the model's "working memory," and like human working memory, it creates inertia. The solution is not better instructions (the agent already reads and acknowledges them) but mechanical constraints that make the wrong behavior impossible rather than merely discouraged.
