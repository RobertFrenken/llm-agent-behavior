## Core Findings

- **PreToolUse hook on Agent tool (Approach C) is the strongest output-side steering mechanism** because it fires deterministically, requires no model "choice" to be critical, injects review context before the agent runs (shaping the entire response), and adds zero extra agent calls -- it modifies the existing one.
- **Stop hooks (Approach A) are too late** -- they fire after the response is already displayed, making review post-hoc rather than inline.
- **Skill-based assessment (Approach B) is probabilistic** -- it depends on the model choosing to invoke it, which is unreliable.
- **Input-side mediation via `/steer` skill is preferred over a `user-prompt-submit` hook** because the user sees the restructured prompt (transparency), whereas the hook is invisible to the user. However, the hook is mechanically stronger since it fires before the model sees the prompt at all.
- **Adversarial injection should be selective** -- only for recommendation/design agents, not search/explore agents. Detection method (keyword vs. always-on) is an open question.
- **Leading prompt framing (e.g., "isn't X the right approach?") triggers agreement bias** and should be reframed as neutral evaluation requests.

## Key Insights

- **Bidirectional steering requires both input and output intervention** -- neither alone is sufficient. Input-side normalizes prompts to reduce bias triggers; output-side forces self-critique on recommendations. The compound effect is hypothesized to be greater than either alone, but controlled measurement is hard in real work sessions.
- **Transparency and mechanical strength are in tension**: the skill approach gives the user visibility into prompt restructuring, but the hook approach has stronger guarantees (fires before the model processes the prompt). This is an unresolved design tradeoff.
- **Existing hook infrastructure can be extended, not replaced**: payload control (head_limit injection), context freshness (staleness detection), and tool nudging (value registry with DENY + correct value) are incremental improvements to already-working hooks.
- **The adversarial injection pattern -- modifying an existing tool call's context rather than spawning a second agent -- is a general technique** for adding verification without latency cost. The three injected checks (verifiable vs. assertion labeling, omission listing, project-ceiling scaling) are domain-portable.
- **User fatigue is a real risk**: if `/steer` invocation declines, it's ambiguous whether the feature is unnecessary or annoying. Both invocation rate and session quality must be tracked.

## Takeaways

- **Start with low-risk extensions (payload control, context freshness) before new patterns** -- the implementation order explicitly prioritizes extending existing hooks (steps 1-2) before adding new ones (steps 3-6).
- **For output verification, inject adversarial requirements into agent prompts via PreToolUse rather than adding post-hoc review agents.** This is zero-cost (no extra calls) and deterministic (not dependent on model self-selection).
- **Build a value registry (`value-registry.yaml`) of known-risky infrastructure values** (SLURM partitions, module versions, account IDs) and DENY + inject corrections on mismatch. Scope to one project first (KD-GAT).
- **Measure with concrete signals**: agreement-bias reversals (target: fewer), verifiable-vs-assertion ratio (target: >70% verifiable), token consumption per session (payload control should reduce waste), and user invocation frequency over time.
- **For context freshness, add staleness detection** (file modified externally since last read) and a session-scoped summary cache (after 3rd read of same file, DENY + inject cached summary).
- **The `/steer` skill should extract four things**: core technical goal, relevant constraints, what a verifiable answer looks like, and flags for emotional valence / leading framing / scope ambiguity. It should reframe leading questions as neutral evaluations, not censor or remove informative frustration.
- **Open problem**: measuring whether bidirectional (input + output) outperforms either alone requires synthetic evaluation, since controlled comparison in real sessions is impractical.
