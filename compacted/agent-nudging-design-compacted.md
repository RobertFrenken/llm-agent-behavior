## Core Findings

- **Confidence-error correlation is inverted for infrastructure values.** The agent is most likely to skip tool lookups when most confident from training priors, but project-specific values (SLURM partitions, module versions, paths, config keys) are exactly where training priors are most wrong. High confidence = high error rate for values that deviate from common patterns.

- **The hook paradox.** PreToolUse hooks only fire when the agent already decided to use a tool. The problem is the agent skips tools when confident -- so the correction mechanism requires the very behavior it aims to create. Hooks end up "cleaning up the agent's mess" rather than changing behavior. The agent does not learn from blocks across sessions.

- **Concrete failure mode.** `executor.py` was written with `partition_cpu="serial"` despite the agent having read scripts showing `"cpu"` earlier in the same session. This pattern recurs daily across SLURM params, module versions (`cuda/12.4` vs `cuda/12.4.1`), paths, and config field names.

- **Five approaches compared, none fully solve it:**
  1. **Edit/Write validation hooks** -- catches errors for registered values but is pure error correction, requires manual registry upkeep, does not change behavior
  2. **Rules ("always look up")** -- zero cost but no enforcement; the paradox means confidence overrides rule attendance
  3. **Citation requirements before edits** -- forces lookup cycles but no hook point exists before the agent composes edit content; agent can cite files it read yet still generate from priors
  4. **Codebase constants file** -- eliminates the problem structurally for code files but doesn't cover docs, plans, or new projects
  5. **Pre-composition context injection** -- right timing but no reliable signal for "about to edit" vs "just reading"; if the agent skips Read entirely, back to the paradox

## Key Insights

- **Generation is chosen over lookup because it's faster.** The agent avoids tool-call round-trips when confident. Making generation "feel slower or riskier" could flip the default, but no mechanism exists for this yet.

- **The paradox is recursive.** Every mitigation that depends on the agent attending to something (rules, citation requirements, structured output) is undermined by the same overconfidence that causes the original problem. The nudge must be attended to, but confidence causes inattention.

- **Codebase design for agent consumption is a double-edged sword.** Constants files (approach 4) have independent maintainability benefits for humans, making them defensible. But designing codebases primarily to work around agent limitations is a "new and possibly misguided pattern."

- **Reactive may be "good enough" pragmatically.** Block-and-retry catches errors reliably for registered values. The agent wastes tokens composing wrong content, but output correctness is preserved. The question is whether the registry maintenance cost is sustainable.

- **Measurement is an unsolved prerequisite.** Distinguishing "agent looked it up" from "agent generated correctly by chance" from "agent generated wrong and got blocked" is necessary to evaluate any intervention, but no instrumentation exists for this yet.

## Takeaways

- **Start with approach 1 (Edit/Write validation hook + value registry) immediately.** Low effort, stops the bleeding for known-risky values (SLURM params, module versions). No behavior change, but reliable error prevention.

- **Centralize infrastructure constants (approach 4) where it makes independent sense.** SLURM constants in KD-GAT are a good candidate -- the benefit to human maintainability justifies the effort regardless of agent behavior.

- **Instrument the validation hook to measure block rates by value category.** High block rates identify which categories need stronger nudging and provide data for prioritizing further work.

- **Investigate whether a hook point can fire after "decide to edit" but before "compose edit content."** This is the key missing primitive. The citation approach (3) is the most promising proactive strategy but requires this hook point to enforce.

- **Do not rely on rules alone.** Without enforcement, rules compete with training priors and lose when the agent is confident. Rules are necessary context but insufficient as a standalone intervention.

- **Related work to leverage:** "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026) confirms small prompt changes shift behavior significantly, but the nudge-attention paradox limits applicability. The "Codified Context" paper (arXiv 2602.20478) addresses context injection but not the upstream tool-use avoidance problem.
