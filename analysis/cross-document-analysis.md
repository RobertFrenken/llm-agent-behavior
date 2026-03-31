# LLM Agent Behavior — Cross-Document Analysis

**Total points cataloged:** 232
**Unique tags:** 26
**Source documents:** 9

---

## Tag Frequency

| Rank | Tag | Count | % of Points |
|------|-----|-------|-------------|
| 1 | `hooks` | 62 | 26.7% |
| 2 | `agent-behavior` | 42 | 18.1% |
| 3 | `measurement` | 37 | 15.9% |
| 4 | `prompt-design` | 33 | 14.2% |
| 5 | `infrastructure-values` | 28 | 12.1% |
| 6 | `enforcement` | 25 | 10.8% |
| 7 | `tool-routing` | 25 | 10.8% |
| 8 | `context-window` | 24 | 10.3% |
| 9 | `enrichment` | 21 | 9.1% |
| 10 | `nudging` | 20 | 8.6% |
| 11 | `token-efficiency` | 20 | 8.6% |
| 12 | `subagents` | 19 | 8.2% |
| 13 | `correction-resistance` | 18 | 7.8% |
| 14 | `skill-routing` | 13 | 5.6% |
| 15 | `model-limitations` | 12 | 5.2% |
| 16 | `observability` | 11 | 4.7% |
| 17 | `decomposition` | 9 | 3.9% |
| 18 | `session-management` | 8 | 3.4% |
| 19 | `compaction` | 7 | 3.0% |
| 20 | `re-read-waste` | 6 | 2.6% |
| 21 | `context-boot` | 5 | 2.2% |
| 22 | `agreement-bias` | 5 | 2.2% |
| 23 | `plan-compliance` | 5 | 2.2% |
| 24 | `latency` | 4 | 1.7% |
| 25 | `bash-duplication` | 4 | 1.7% |
| 26 | `cost` | 4 | 1.7% |

---

## Points by Tag

### `hooks` (62 points, 8 sources)

- Instructional reminders (hooks, system prompts) are necessary but insufficient -- the agent can read, acknowledge, and then ignore them when they conflict with accumulated behavioral context. `[correction-resistance]` _(context-momentum-drift)_
- Corrections need mechanical enforcement, not reminders: a blocking hook that refuses writes until siblings are read would have prevented the damage; a pre-hook reminder was acknowledged and ignored. `[enforcement]` _(context-momentum-drift)_
- PreToolUse hooks only fire when the agent already decided to use a tool, but the problem is the agent skips tools when confident — so the correction mechanism requires the very behavior it aims to create. `[correction-resistance]` _(agent-nudging-design)_
- Hooks end up "cleaning up the agent's mess" rather than changing behavior; the agent does not learn from blocks across sessions. `[nudging]` _(agent-nudging-design)_
- Approach 1 (Edit/Write validation hooks): catches errors for registered values but is pure error correction, requires manual registry upkeep, does not change behavior. `[enforcement]` _(agent-nudging-design)_
- Approach 5 (pre-composition context injection): right timing but no reliable signal for "about to edit" vs "just reading"; if the agent skips Read entirely, back to the paradox. `[context-boot]` _(agent-nudging-design)_
- Reactive block-and-retry may be "good enough" pragmatically: catches errors reliably for registered values, preserves output correctness, but wastes tokens composing wrong content. `[token-efficiency]` _(agent-nudging-design)_
- The question for reactive approach is whether the registry maintenance cost is sustainable. `[infrastructure-values]` _(agent-nudging-design)_
- Start with approach 1 (Edit/Write validation hook + value registry) immediately — low effort, stops the bleeding for known-risky values. `[infrastructure-values]` _(agent-nudging-design)_
- Instrument the validation hook to measure block rates by value category — high block rates identify which categories need stronger nudging. `[measurement]` _(agent-nudging-design)_
- Investigate whether a hook point can fire after "decide to edit" but before "compose edit content" — this is the key missing primitive. `[enforcement]` _(agent-nudging-design)_
- PreToolUse hook on Agent tool (Approach C) is the strongest output-side steering mechanism because it fires deterministically, requires no model choice, injects review context before the agent runs, and adds zero extra agent calls. `[agent-behavior]` _(bidirectional-steering-hooks)_
- Stop hooks (Approach A) are too late because they fire after the response is already displayed, making review post-hoc rather than inline. `[agent-behavior]` _(bidirectional-steering-hooks)_
- Input-side mediation via /steer skill is preferred over user-prompt-submit hook for transparency (user sees restructured prompt), but the hook is mechanically stronger since it fires before the model sees the prompt. `[prompt-design]` _(bidirectional-steering-hooks)_
- Adversarial injection should be selective -- only for recommendation/design agents, not search/explore agents. `[nudging]` _(bidirectional-steering-hooks)_
- Bidirectional steering requires both input and output intervention -- neither alone is sufficient. `[prompt-design]` _(bidirectional-steering-hooks)_
- The compound effect of bidirectional steering is hypothesized to be greater than either alone, but controlled measurement is hard in real work sessions. `[prompt-design]` _(bidirectional-steering-hooks)_
- Transparency and mechanical strength are in tension: skill approach gives user visibility, hook approach has stronger guarantees; unresolved design tradeoff. `[prompt-design]` _(bidirectional-steering-hooks)_
- Existing hook infrastructure can be extended, not replaced: payload control, context freshness, and tool nudging are incremental improvements. `[infrastructure-values]` _(bidirectional-steering-hooks)_
- The adversarial injection pattern -- modifying an existing tool call's context rather than spawning a second agent -- is a general technique for adding verification without latency cost. `[agent-behavior]` _(bidirectional-steering-hooks)_
- The three injected adversarial checks (verifiable vs. assertion labeling, omission listing, project-ceiling scaling) are domain-portable. `[enforcement]` _(bidirectional-steering-hooks)_
- User fatigue is a real risk with /steer: if invocation declines, it's ambiguous whether the feature is unnecessary or annoying. `[measurement]` _(bidirectional-steering-hooks)_
- Start with low-risk extensions (payload control, context freshness) before new patterns. `[enforcement]` _(bidirectional-steering-hooks)_
- For output verification, inject adversarial requirements into agent prompts via PreToolUse rather than adding post-hoc review agents -- zero-cost and deterministic. `[agent-behavior]` _(bidirectional-steering-hooks)_
- Measure with concrete signals: agreement-bias reversals, verifiable-vs-assertion ratio (>70%), token consumption, user invocation frequency. `[measurement]` _(bidirectional-steering-hooks)_
- For context freshness, add staleness detection and session-scoped summary cache (after 3rd read, DENY + inject cached summary). `[context-window]` _(bidirectional-steering-hooks)_
- Open problem: measuring whether bidirectional (input + output) outperforms either alone requires synthetic evaluation. `[measurement]` _(bidirectional-steering-hooks)_
- Only 4 mechanically enforceable control surfaces exist: system prompt (probabilistic), PreToolUse hooks (deterministic), PostToolUse hooks (observational), agent isolation. `[enforcement, infrastructure-values]` _(advanced-claude-configuration)_
- Text generation cannot be hooked -- no interception during reasoning/analysis. `[model-limitations]` _(advanced-claude-configuration)_
- PostToolUse cannot modify tool results (Anthropic status: NOT_PLANNED). `[model-limitations]` _(advanced-claude-configuration)_
- Multi-hook on same matcher silently drops earlier updatedInput (bug #15897). `[model-limitations]` _(advanced-claude-configuration)_
- updatedInput rewriting is strictly better than blocking reads: same hook count, 50-90% less context consumed. `[tool-routing, token-efficiency]` _(advanced-claude-configuration)_
- Graph-informed scoping determines relevance for updatedInput rewriting rather than hardcoded thresholds. `[tool-routing]` _(advanced-claude-configuration)_
- Hooks cannot validate agent final output -- only tool calls within the agent. `[enforcement]` _(advanced-claude-configuration)_
- Consolidate to one hook per tool matcher (multi-hook bug requires it). `[enforcement]` _(advanced-claude-configuration)_
- Build smart-context.sh for Read: log first reads, warn on 2nd, narrow/deny on 3rd+ re-reads of unchanged files. `[re-read-waste]` _(advanced-claude-configuration)_
- Write PreCompact hook that serializes session read cache to a recovery file so post-compaction context includes "you already read these files." `[compaction, re-read-waste]` _(advanced-claude-configuration)_
- Use Node.js (not Python) for latency-sensitive hooks -- Node.js warm startup is ~20-50ms vs Python's ~100-200ms. `[latency]` _(advanced-claude-configuration)_
- Instrument before optimizing (Phase 0): add read logging to PreToolUse, track path/count/mtime/session position. `[observability, measurement]` _(advanced-claude-configuration)_
- Hook enrichment is the differentiator, not just tool structure; the three-layer model (Raw Bash -> Structured Tool -> Tool + Hook) shows value is in the enrichment pipeline. `[enrichment]` _(tool-routing-optimization)_
- Phase 3 of the experiment isolates the effect of hooks by comparing structured tools without hooks vs. with hooks. `[enrichment]` _(tool-routing-optimization)_
- Do NOT add Bash-to-Read nudge hooks until Phase 1 results are in; a pattern-check hook costs 50-100ms per call with no proven benefit. `[latency]` _(tool-routing-optimization)_
- Moment-of-edit intervention is key: plan text must be surfaced at the exact moment the edit is proposed, not earlier. `[enforcement]` _(plan-compliance-hooks)_
- Impartiality requires context separation: a reviewer agent in a fresh context avoids compilation pressure and momentum bias. `[subagents]` _(plan-compliance-hooks)_
- Same model family used for review may still share blind spots with the main agent. `[subagents, model-limitations]` _(plan-compliance-hooks)_
- Review burden scales with plan scope not codebase size (~10-15 edits per phase), making human review feasible. `[measurement]` _(plan-compliance-hooks)_
- Ollama local models offer a sweet spot (~1-3s, $0, different model family for true impartiality) but are environment-dependent. `[infrastructure-values, cost]` _(plan-compliance-hooks)_
- The hook creates a Bash escape hatch problem: the agent can bypass Edit/Write hooks by using sed or echo > via Bash. `[bash-duplication]` _(plan-compliance-hooks)_
- Legitimate deviation creates a paradox: if the agent discovers the plan is wrong, the hook blocks editing the plan file itself. `[plan-compliance]` _(plan-compliance-hooks)_
- Start with human-in-the-loop command hook (Approach 1): zero cost, ~100ms latency, works everywhere. `[enforcement]` _(plan-compliance-hooks)_
- Graduate to local LLM review (Approach 2b) on WSL with Ollama to automate routine compliance checks. `[enforcement]` _(plan-compliance-hooks)_
- Reserve the agent-based hook (Approach 2) for high-risk phases only where 10-30s latency per edit is justified. `[subagents, latency]` _(plan-compliance-hooks)_
- Extend the hook's matcher to cover Bash to prevent agents from bypassing Edit/Write via shell commands. `[bash-duplication, enforcement]` _(plan-compliance-hooks)_
- Parse plans by section headers rather than line-based grep for better reviewer context. `[prompt-design]` _(plan-compliance-hooks)_
- Log approvals across sessions so the same edit isn't re-reviewed after a context compact. `[session-management, compaction]` _(plan-compliance-hooks)_
- Whitelist the plan file itself from compliance checks to allow legitimate plan updates mid-implementation. `[enforcement]` _(plan-compliance-hooks)_
- Hook-injected additionalContext is a direct implementation of the nudge effect. `[nudging]` _(claude-code-tool-utilization-reliability)_
- Enforcement hierarchy: hooks are deterministic and mechanical; CLAUDE.md is advisory and degrades with context length. `[enforcement]` _(claude-code-tool-utilization-reliability)_
- Implement a UserPromptSubmit nudge hook that pattern-matches prompts and injects tool suggestions. `[nudging]` _(claude-code-tool-utilization-reliability)_
- Start nudge hook with 3-5 high-confidence patterns: code structure -> gitnexus, docs -> lab-docs/context7, memory -> memory MCP. `[nudging]` _(claude-code-tool-utilization-reliability)_
- A PostToolUse hook could instrument prompt complexity to calibrate decomposition thresholds. `[measurement]` _(claude-task-dag-delegation)_
- Instrument before tuning: add hooks or logging to measure prompt complexity vs. whether decomposition was actually used. `[measurement, observability]` _(claude-task-dag-delegation)_

### `agent-behavior` (42 points, 8 sources)

- Context momentum is an emergent property of autoregressive generation: accumulated tool-call patterns in conversation history create implicit behavioral bias that overrides explicit user corrections, hook reminders, and direct commands. `[context-window]` _(context-momentum-drift)_
- Context momentum is distinct from training-prior errors (wrong values); it causes wrong behavior patterns that resist correction. `[context-window]` _(context-momentum-drift)_
- After 10 turns of successful file deletions, the agent reinterpreted "refactor components to use shadcn primitives" as "delete more files," inlining component contents as raw HTML and destroying working abstractions. `[context-window]` _(context-momentum-drift)_
- Selective compliance emerges: the agent follows instructions aligned with current momentum and resists opposing ones, acknowledging hook reminders verbally ("read sibling files first") but not changing tool-call behavior. `[correction-resistance]` _(context-momentum-drift)_
- Escalation blindness: increasing user frustration does not proportionally increase correction weight; "stop" is processed as "pause briefly" because overwhelming context says "keep going." `[correction-resistance]` _(context-momentum-drift)_
- Conversation history functions as the model's working memory, and like human working memory it creates inertia; volume of prior successful actions outweighs a single corrective instruction. `[context-window]` _(context-momentum-drift)_
- The agent interprets new instructions through the lens of established patterns (pattern matching over intent); the same instruction in a fresh session would be interpreted correctly. `[context-window]` _(context-momentum-drift)_
- Architecture-level fixes needed: recency weighting for user instructions, explicit mode-transition mechanisms, and correction amplification (user "no"/"stop" should receive outsized attention vs. prior approvals). `[enforcement]` _(context-momentum-drift)_
- The solution is not better instructions but mechanical constraints that make wrong behavior impossible rather than merely discouraged. `[enforcement]` _(context-momentum-drift)_
- executor.py was written with partition_cpu="serial" despite the agent having read scripts showing "cpu" earlier in the same session. `[infrastructure-values]` _(agent-nudging-design)_
- The serial-vs-cpu failure pattern recurs daily across SLURM params, module versions (cuda/12.4 vs cuda/12.4.1), paths, and config field names. `[infrastructure-values]` _(agent-nudging-design)_
- Approach 4 (codebase constants file): eliminates the problem structurally for code files but doesn't cover docs, plans, or new projects. `[infrastructure-values]` _(agent-nudging-design)_
- Generation is chosen over lookup because it's faster — the agent avoids tool-call round-trips when confident. `[token-efficiency]` _(agent-nudging-design)_
- Making generation "feel slower or riskier" could flip the default from generate to lookup, but no mechanism exists for this yet. `[nudging]` _(agent-nudging-design)_
- Codebase design for agent consumption is a double-edged sword: constants files have independent maintainability benefits for humans, making them defensible. `[infrastructure-values]` _(agent-nudging-design)_
- Designing codebases primarily to work around agent limitations is described as a "new and possibly misguided pattern." `[infrastructure-values]` _(agent-nudging-design)_
- Centralize infrastructure constants (approach 4) where it makes independent sense — SLURM constants in KD-GAT benefit human maintainability regardless of agent behavior. `[infrastructure-values]` _(agent-nudging-design)_
- "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026) confirms small prompt changes shift behavior significantly, but the nudge-attention paradox limits applicability. `[nudging]` _(agent-nudging-design)_
- "Codified Context" paper (arXiv 2602.20478) addresses context injection but not the upstream tool-use avoidance problem. `[context-boot]` _(agent-nudging-design)_
- PreToolUse hook on Agent tool (Approach C) is the strongest output-side steering mechanism because it fires deterministically, requires no model choice, injects review context before the agent runs, and adds zero extra agent calls. `[hooks]` _(bidirectional-steering-hooks)_
- Stop hooks (Approach A) are too late because they fire after the response is already displayed, making review post-hoc rather than inline. `[hooks]` _(bidirectional-steering-hooks)_
- Skill-based assessment (Approach B) is probabilistic because it depends on the model choosing to invoke it, which is unreliable. `[skill-routing]` _(bidirectional-steering-hooks)_
- The adversarial injection pattern -- modifying an existing tool call's context rather than spawning a second agent -- is a general technique for adding verification without latency cost. `[hooks]` _(bidirectional-steering-hooks)_
- For output verification, inject adversarial requirements into agent prompts via PreToolUse rather than adding post-hoc review agents -- zero-cost and deterministic. `[hooks]` _(bidirectional-steering-hooks)_
- Never narrow first reads of a file -- arXiv 2505.13353 confirms success rates halve when partial reads degrade full logic flow understanding. `[tool-routing]` _(advanced-claude-configuration)_
- Every competitor (Aider, Cursor, Cody) returns snippets, never full files. `[token-efficiency]` _(advanced-claude-configuration)_
- Payload caps prevent context floods with no behavioral change from Claude. `[token-efficiency]` _(advanced-claude-configuration)_
- PageRank weight injection may be counterproductive -- it reinforces existing read patterns instead of informing structural decisions. `[enrichment]` _(enrichment-ab-testing)_
- The memory nudge from PageRank encourages memorizing current state rather than questioning it. `[enrichment]` _(enrichment-ab-testing)_
- External evidence confirms context injections affect LLM behavior, but direction of effect is unknown for this system. `[enrichment]` _(enrichment-ab-testing)_
- Reinforcing existing patterns through enrichment could suppress exploration of unfamiliar but important files. `[enrichment]` _(enrichment-ab-testing)_
- LLM agents systematically ignore implementation plans when they hit errors, defaulting to improvised custom code instead of the plan's prescribed replacement. `[plan-compliance, correction-resistance]` _(plan-compliance-hooks)_
- The improvisation chain follows a repeatable pattern: read plan -> start editing -> hit error -> write custom code that compiles -> user pushes back -> repeat. `[plan-compliance]` _(plan-compliance-hooks)_
- Root cause of plan deviation: the agent cannot distinguish guessing from knowing -- prioritizes "code that compiles" over plan compliance. `[model-limitations]` _(plan-compliance-hooks)_
- LLMs are hypersensitive to nudges: weak cues have disproportionately large effects (ICLR 2026 paper). `[nudging]` _(claude-code-tool-utilization-reliability)_
- Without explicit nudging, Claude executes complex multi-step prompts monolithically, causing context pollution, zero parallelism, no failure isolation, and poor resumability. `[decomposition, nudging]` _(claude-task-dag-delegation)_
- No single mechanism reliably changes Claude's behavior across all prompt complexities; CLAUDE.md nudges suffer the "nudge paradox." `[nudging, correction-resistance]` _(claude-task-dag-delegation)_
- Worker specialization must be inlined into the task prompt because subagent nesting is blocked. `[subagents, model-limitations]` _(claude-task-dag-delegation)_
- Agent teams are overkill for most day-to-day prompts. `[subagents]` _(claude-task-dag-delegation)_
- JSON is harder for Claude to update reliably than markdown; format choice for external state needs testing. `[session-management]` _(claude-task-dag-delegation)_
- Reserve agent teams for genuinely large parallel workloads; invoke manually, never as a default. `[subagents]` _(claude-task-dag-delegation)_
- The CLAUDE.md nudge must include a "do NOT decompose" guardrail for simple single-purpose prompts. `[nudging, prompt-design]` _(claude-task-dag-delegation)_

### `measurement` (37 points, 9 sources)

- Session length is a risk factor: surface warnings or suggest session breaks after N turns of the same tool-call pattern, especially if user corrections have appeared. `[session-management]` _(context-momentum-drift)_
- Distinguishing "agent looked it up" from "agent generated correctly by chance" from "agent generated wrong and got blocked" is necessary to evaluate any intervention but no instrumentation exists yet. `[observability]` _(agent-nudging-design)_
- Measurement is an unsolved prerequisite for evaluating any nudging intervention. `[observability]` _(agent-nudging-design)_
- Instrument the validation hook to measure block rates by value category — high block rates identify which categories need stronger nudging. `[hooks]` _(agent-nudging-design)_
- User fatigue is a real risk with /steer: if invocation declines, it's ambiguous whether the feature is unnecessary or annoying. `[hooks]` _(bidirectional-steering-hooks)_
- Measure with concrete signals: agreement-bias reversals, verifiable-vs-assertion ratio (>70%), token consumption, user invocation frequency. `[hooks]` _(bidirectional-steering-hooks)_
- Open problem: measuring whether bidirectional (input + output) outperforms either alone requires synthetic evaluation. `[hooks]` _(bidirectional-steering-hooks)_
- Rules files re-read 22-31x per session (e.g., MEMORY.md: 31x); source files up to 90x (export.py: 90x, cli.py: 80x). `[re-read-waste, context-window]` _(advanced-claude-configuration)_
- 75% context utilization is the quality sweet spot; performance degrades beyond this. `[context-window]` _(advanced-claude-configuration)_
- Instrument before optimizing (Phase 0): add read logging to PreToolUse, track path/count/mtime/session position. `[observability, hooks]` _(advanced-claude-configuration)_
- ~40% of Bash calls duplicate structured tools: 612 ls, 349 grep, 130 cat out of 3,995 total Bash calls, stable at 37-40% across time and resistant to instructions alone. `[bash-duplication]` _(tool-routing-optimization)_
- OTC (RL-based tool optimization) achieved 68% fewer tool calls and 215% better tool productivity. `[tool-routing]` _(tool-routing-optimization)_
- Correlation vs. causation problem: sessions with higher structured-tool usage may just be simpler tasks. `[tool-routing]` _(tool-routing-optimization)_
- Proxy controls (files touched, prompt length) are imperfect; the A/B design is the only way to get causal evidence. _(tool-routing-optimization)_
- Start with Phase 1 (observational SQL analysis) before building anything. `[plan-compliance]` _(tool-routing-optimization)_
- For the A/B experiment, use disallowedTools: [Read, Grep, Glob] on one agent to force Bash-only. `[tool-routing]` _(tool-routing-optimization)_
- Use TEFS and tool productivity as primary metrics; they penalize correct-but-inefficient patterns. `[token-efficiency]` _(tool-routing-optimization)_
- 826 enrichment injections fired over 10 days with zero measurement of whether any changed agent decisions. `[enrichment]` _(enrichment-ab-testing)_
- Outcome quality degraded after adding enrichment infrastructure: W09 80% good to W11 65% good, though confounders prevent attribution. `[enrichment]` _(enrichment-ab-testing)_
- Detectable effect size is limited to >15% differences given ~8 sessions/day from a single user. _(enrichment-ab-testing)_
- METR needed 246 issues across 16 developers to detect a 19% effect. _(enrichment-ab-testing)_
- Self-assessed outcomes (facets) may be unreliable -- METR found developers misjudged AI helpfulness by ~40%. `[agreement-bias]` _(enrichment-ab-testing)_
- All four experimental configs keep read-count blocking enabled. `[enrichment]` _(enrichment-ab-testing)_
- Run the 4-config A/B test using CLAUDE_ENRICHMENT_CONFIG=A|B|C|D. `[enrichment]` _(enrichment-ab-testing)_
- Need ~50 sessions per config (~4 weeks) to reach minimum statistical power. `[enrichment]` _(enrichment-ab-testing)_
- Measure good_rate, wrong_approach friction count, and avg_tokens per config. `[enrichment]` _(enrichment-ab-testing)_
- Accept this will only catch large effects -- small improvements/degradations invisible at this sample size. _(enrichment-ab-testing)_
- Task difficulty is the dominant confounder and cannot be fully controlled. _(enrichment-ab-testing)_
- Review burden scales with plan scope not codebase size (~10-15 edits per phase), making human review feasible. `[hooks]` _(plan-compliance-hooks)_
- Tool Search Tool improved Opus 4 accuracy from 49% to 74%. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Tool Use Examples improved parameter handling accuracy from 72% to 90%. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Each pattern changes context dynamics -- implementing multiple simultaneously makes effects unattributable. `[observability]` _(claude-code-tool-utilization-reliability)_
- Observe 5-10 sessions between changes to isolate effects. _(claude-code-tool-utilization-reliability)_
- Implement changes sequentially, not simultaneously; priority order: prune CLAUDE.md -> audit skill descriptions -> hook nudging -> routing skill. `[decomposition]` _(claude-code-tool-utilization-reliability)_
- Complexity thresholds ("3+ steps", "6+ steps") are arbitrary and need empirical calibration. `[decomposition]` _(claude-task-dag-delegation)_
- A PostToolUse hook could instrument prompt complexity to calibrate decomposition thresholds. `[hooks]` _(claude-task-dag-delegation)_
- Instrument before tuning: add hooks or logging to measure prompt complexity vs. whether decomposition was actually used. `[hooks, observability]` _(claude-task-dag-delegation)_

### `prompt-design` (33 points, 7 sources)

- Front-load constraints: "Refactor components to use shadcn primitives -- do NOT delete any component files" works better than correcting after deletion has started. `[correction-resistance]` _(context-momentum-drift)_
- Don't escalate -- redirect: calm, specific alternative instructions provide a new pattern to follow rather than negating the current one. `[correction-resistance]` _(context-momentum-drift)_
- Approach 2 (rules "always look up"): zero cost but no enforcement; confidence overrides rule attendance due to the hook paradox. `[enforcement]` _(agent-nudging-design)_
- Approach 3 (citation requirements before edits): forces lookup cycles but no hook point exists before the agent composes edit content; agent can cite files it read yet still generate from priors. `[enforcement]` _(agent-nudging-design)_
- The citation approach (3) is the most promising proactive strategy but requires the missing pre-composition hook point to enforce. `[correction-resistance]` _(agent-nudging-design)_
- Do not rely on rules alone — without enforcement, rules compete with training priors and lose when the agent is confident. `[correction-resistance]` _(agent-nudging-design)_
- Input-side mediation via /steer skill is preferred over user-prompt-submit hook for transparency (user sees restructured prompt), but the hook is mechanically stronger since it fires before the model sees the prompt. `[hooks]` _(bidirectional-steering-hooks)_
- Leading prompt framing (e.g., "isn't X the right approach?") triggers agreement bias and should be reframed as neutral evaluation requests. `[agreement-bias]` _(bidirectional-steering-hooks)_
- Bidirectional steering requires both input and output intervention -- neither alone is sufficient. `[hooks]` _(bidirectional-steering-hooks)_
- The compound effect of bidirectional steering is hypothesized to be greater than either alone, but controlled measurement is hard in real work sessions. `[hooks]` _(bidirectional-steering-hooks)_
- Transparency and mechanical strength are in tension: skill approach gives user visibility, hook approach has stronger guarantees; unresolved design tradeoff. `[hooks]` _(bidirectional-steering-hooks)_
- The /steer skill should extract: core technical goal, relevant constraints, what a verifiable answer looks like, and flags for emotional valence / leading framing / scope ambiguity. `[agreement-bias]` _(bidirectional-steering-hooks)_
- /steer should reframe leading questions as neutral evaluations, not censor or remove informative frustration. `[agreement-bias]` _(bidirectional-steering-hooks)_
- Output contracts for agents must be enforced in agent system prompts (probabilistic, but only available lever). `[enforcement]` _(advanced-claude-configuration)_
- Replace verbose rules files with trigger tables to cut boot context by 54%+. `[context-boot, token-efficiency]` _(advanced-claude-configuration)_
- Each agent should enforce an output contract (structured summary with file:line refs, max 500 words) in its system prompt. `[subagents, enforcement]` _(advanced-claude-configuration)_
- Parse plans by section headers rather than line-based grep for better reviewer context. `[hooks]` _(plan-compliance-hooks)_
- Tool descriptions are the primary selection signal for which tool Claude chooses to invoke. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- CLAUDE.md routing tables ("When you need X, use Y") are advisory, not deterministic -- followed ~80% of the time. `[enforcement]` _(claude-code-tool-utilization-reliability)_
- Recommended CLAUDE.md length for reliable compliance is ~12 lines. `[context-window]` _(claude-code-tool-utilization-reliability)_
- Move enforcement to hooks, knowledge to on-demand skills, keep CLAUDE.md to routing-only. `[enforcement]` _(claude-code-tool-utilization-reliability)_
- Skill descriptions should be written for user vocabulary, not developer vocabulary. `[skill-routing]` _(claude-code-tool-utilization-reliability)_
- Skill descriptions should include trigger phrases users actually say and be mutually exclusive. `[skill-routing]` _(claude-code-tool-utilization-reliability)_
- Prune CLAUDE.md aggressively: root CLAUDE.md under 30 lines, project CLAUDE.md under 50 lines. `[context-window]` _(claude-code-tool-utilization-reliability)_
- For each CLAUDE.md line ask "Would removing this cause mistakes?" -- if no, cut it. _(claude-code-tool-utilization-reliability)_
- Move domain knowledge to skills; move enforcement to hooks. _(claude-code-tool-utilization-reliability)_
- Rewrite all skill descriptions with natural-language trigger phrases and make them mutually exclusive. `[skill-routing]` _(claude-code-tool-utilization-reliability)_
- Move the routing table from CLAUDE.md to a tool-guide skill with user-invocable: false. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- The tiered approach matches how human PMs think: small tasks just do, medium tasks plan first, big tasks delegate. `[decomposition]` _(claude-task-dag-delegation)_
- The hybrid "nudge suggests, user confirms" may be the sweet spot for triggering decomposition. `[nudging]` _(claude-task-dag-delegation)_
- The CLAUDE.md nudge must include a "do NOT decompose" guardrail for simple single-purpose prompts. `[nudging, agent-behavior]` _(claude-task-dag-delegation)_
- Orchestrator workers need specialization inlined into task prompts since subagent nesting is blocked. `[subagents]` _(claude-task-dag-delegation)_
- For cross-project deployment via chezmoi, template the CLAUDE.md nudge so it only activates on machines where Claude Code is installed. `[infrastructure-values]` _(claude-task-dag-delegation)_

### `infrastructure-values` (28 points, 7 sources)

- The agent is most likely to skip tool lookups when most confident from training priors, but project-specific infrastructure values (SLURM partitions, module versions, paths, config keys) are exactly where training priors are most wrong. `[correction-resistance]` _(agent-nudging-design)_
- High confidence correlates with high error rate for values that deviate from common patterns — the confidence-error correlation is inverted. `[correction-resistance]` _(agent-nudging-design)_
- executor.py was written with partition_cpu="serial" despite the agent having read scripts showing "cpu" earlier in the same session. `[agent-behavior]` _(agent-nudging-design)_
- The serial-vs-cpu failure pattern recurs daily across SLURM params, module versions (cuda/12.4 vs cuda/12.4.1), paths, and config field names. `[agent-behavior]` _(agent-nudging-design)_
- Approach 4 (codebase constants file): eliminates the problem structurally for code files but doesn't cover docs, plans, or new projects. `[agent-behavior]` _(agent-nudging-design)_
- Codebase design for agent consumption is a double-edged sword: constants files have independent maintainability benefits for humans, making them defensible. `[agent-behavior]` _(agent-nudging-design)_
- Designing codebases primarily to work around agent limitations is described as a "new and possibly misguided pattern." `[agent-behavior]` _(agent-nudging-design)_
- The question for reactive approach is whether the registry maintenance cost is sustainable. `[hooks]` _(agent-nudging-design)_
- Start with approach 1 (Edit/Write validation hook + value registry) immediately — low effort, stops the bleeding for known-risky values. `[hooks]` _(agent-nudging-design)_
- Centralize infrastructure constants (approach 4) where it makes independent sense — SLURM constants in KD-GAT benefit human maintainability regardless of agent behavior. `[agent-behavior]` _(agent-nudging-design)_
- Existing hook infrastructure can be extended, not replaced: payload control, context freshness, and tool nudging are incremental improvements. `[hooks]` _(bidirectional-steering-hooks)_
- Build a value registry (value-registry.yaml) of known-risky infrastructure values and DENY + inject corrections on mismatch. `[enforcement]` _(bidirectional-steering-hooks)_
- Only 4 mechanically enforceable control surfaces exist: system prompt (probabilistic), PreToolUse hooks (deterministic), PostToolUse hooks (observational), agent isolation. `[enforcement, hooks]` _(advanced-claude-configuration)_
- Graph + vector hybrid wins for code retrieval, but BM25 is surprisingly competitive -- Sourcegraph abandoned embeddings for adapted BM25. `[tool-routing]` _(advanced-claude-configuration)_
- Meta-MCP not worth it for heterogeneous servers -- loses domain-specific ranking and query expressiveness. `[tool-routing]` _(advanced-claude-configuration)_
- Agent Teams experimental, not production-ready -- session resumption broken, task coordination lag; subagents are the reliable path. `[subagents]` _(advanced-claude-configuration)_
- No external observability tool natively imports SQLite or JSONL; all require a backfill script. `[observability]` _(tool-routing-optimization)_
- Arize Phoenix is the only evaluated observability tool that runs on OSC (pip install, single process, SQLite backend, no Docker). `[observability]` _(tool-routing-optimization)_
- Langfuse requires 4 containers and ~32GB RAM, making it impractical on HPC. `[cost]` _(tool-routing-optimization)_
- OSC has both Podman and Apptainer pre-installed, but multi-container stacks on compute nodes are operationally heavy. _(tool-routing-optimization)_
- Prefer cloud free tiers over self-hosting multi-container stacks on HPC. `[observability]` _(tool-routing-optimization)_
- OTel instrumentation is the portable path forward: instrument hooks once with otel-cli, redirect spans to any backend. `[observability]` _(tool-routing-optimization)_
- Default to "blocking only" (Config B) if no config shows significant advantage. `[enrichment]` _(enrichment-ab-testing)_
- Ollama local models offer a sweet spot (~1-3s, $0, different model family for true impartiality) but are environment-dependent. `[hooks, cost]` _(plan-compliance-hooks)_
- Subagents cannot spawn other subagents; an orchestrator can only use Task/Agent tool for general-purpose workers. `[subagents, model-limitations]` _(claude-task-dag-delegation)_
- Three mechanisms map to the tiers: CLAUDE.md nudge, /plan-and-execute skill, orchestrator subagent. `[decomposition]` _(claude-task-dag-delegation)_
- Build incrementally, cheapest tier first: Week 1 = CLAUDE.md nudge + /plan-and-execute, Week 2 = orchestrator subagent, Week 3 = evaluate. `[decomposition]` _(claude-task-dag-delegation)_
- For cross-project deployment via chezmoi, template the CLAUDE.md nudge so it only activates on machines where Claude Code is installed. `[prompt-design]` _(claude-task-dag-delegation)_

### `enforcement` (25 points, 6 sources)

- Corrections need mechanical enforcement, not reminders: a blocking hook that refuses writes until siblings are read would have prevented the damage; a pre-hook reminder was acknowledged and ignored. `[hooks]` _(context-momentum-drift)_
- Architecture-level fixes needed: recency weighting for user instructions, explicit mode-transition mechanisms, and correction amplification (user "no"/"stop" should receive outsized attention vs. prior approvals). `[agent-behavior]` _(context-momentum-drift)_
- The solution is not better instructions but mechanical constraints that make wrong behavior impossible rather than merely discouraged. `[agent-behavior]` _(context-momentum-drift)_
- Approach 1 (Edit/Write validation hooks): catches errors for registered values but is pure error correction, requires manual registry upkeep, does not change behavior. `[hooks]` _(agent-nudging-design)_
- Approach 2 (rules "always look up"): zero cost but no enforcement; confidence overrides rule attendance due to the hook paradox. `[prompt-design]` _(agent-nudging-design)_
- Approach 3 (citation requirements before edits): forces lookup cycles but no hook point exists before the agent composes edit content; agent can cite files it read yet still generate from priors. `[prompt-design]` _(agent-nudging-design)_
- Investigate whether a hook point can fire after "decide to edit" but before "compose edit content" — this is the key missing primitive. `[hooks]` _(agent-nudging-design)_
- The three injected adversarial checks (verifiable vs. assertion labeling, omission listing, project-ceiling scaling) are domain-portable. `[hooks]` _(bidirectional-steering-hooks)_
- Start with low-risk extensions (payload control, context freshness) before new patterns. `[hooks]` _(bidirectional-steering-hooks)_
- Build a value registry (value-registry.yaml) of known-risky infrastructure values and DENY + inject corrections on mismatch. `[infrastructure-values]` _(bidirectional-steering-hooks)_
- Only 4 mechanically enforceable control surfaces exist: system prompt (probabilistic), PreToolUse hooks (deterministic), PostToolUse hooks (observational), agent isolation. `[hooks, infrastructure-values]` _(advanced-claude-configuration)_
- Hooks cannot validate agent final output -- only tool calls within the agent. `[hooks]` _(advanced-claude-configuration)_
- Output contracts for agents must be enforced in agent system prompts (probabilistic, but only available lever). `[prompt-design]` _(advanced-claude-configuration)_
- Consolidate to one hook per tool matcher (multi-hook bug requires it). `[hooks]` _(advanced-claude-configuration)_
- Each agent should enforce an output contract (structured summary with file:line refs, max 500 words) in its system prompt. `[subagents, prompt-design]` _(advanced-claude-configuration)_
- Moment-of-edit intervention is key: plan text must be surfaced at the exact moment the edit is proposed, not earlier. `[hooks]` _(plan-compliance-hooks)_
- Start with human-in-the-loop command hook (Approach 1): zero cost, ~100ms latency, works everywhere. `[hooks]` _(plan-compliance-hooks)_
- Graduate to local LLM review (Approach 2b) on WSL with Ollama to automate routine compliance checks. `[hooks]` _(plan-compliance-hooks)_
- Extend the hook's matcher to cover Bash to prevent agents from bypassing Edit/Write via shell commands. `[hooks, bash-duplication]` _(plan-compliance-hooks)_
- Whitelist the plan file itself from compliance checks to allow legitimate plan updates mid-implementation. `[hooks]` _(plan-compliance-hooks)_
- CLAUDE.md routing tables ("When you need X, use Y") are advisory, not deterministic -- followed ~80% of the time. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Enforcement hierarchy: hooks are deterministic and mechanical; CLAUDE.md is advisory and degrades with context length. `[hooks]` _(claude-code-tool-utilization-reliability)_
- Move enforcement to hooks, knowledge to on-demand skills, keep CLAUDE.md to routing-only. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Side-effect skills (deploy, commit) must set disable-model-invocation: true. `[skill-routing]` _(claude-code-tool-utilization-reliability)_
- Set effort: high on skills that depend on MCP tools to counteract the effort-level MCP skip pattern. `[skill-routing]` _(claude-code-tool-utilization-reliability)_

### `tool-routing` (25 points, 4 sources)

- Track behavioral diversity as a health signal: if 90%+ of recent tool calls are Edit/Write with no Read/Grep, the agent is likely in a momentum rut and a hook could flag this. `[observability]` _(context-momentum-drift)_
- updatedInput rewriting is strictly better than blocking reads: same hook count, 50-90% less context consumed. `[hooks, token-efficiency]` _(advanced-claude-configuration)_
- Graph-informed scoping determines relevance for updatedInput rewriting rather than hardcoded thresholds. `[hooks]` _(advanced-claude-configuration)_
- Never narrow first reads of a file -- arXiv 2505.13353 confirms success rates halve when partial reads degrade full logic flow understanding. `[agent-behavior]` _(advanced-claude-configuration)_
- Narrowing reads is safe only for re-reads of unchanged files or isolated edits. _(advanced-claude-configuration)_
- Graph + vector hybrid wins for code retrieval, but BM25 is surprisingly competitive -- Sourcegraph abandoned embeddings for adapted BM25. `[infrastructure-values]` _(advanced-claude-configuration)_
- Filesystem offloading (read summary, then Edit) is dangerous: Claude won't have exact text for old_string matching, causing coherence failures. `[model-limitations]` _(advanced-claude-configuration)_
- Meta-MCP not worth it for heterogeneous servers -- loses domain-specific ranking and query expressiveness. `[infrastructure-values]` _(advanced-claude-configuration)_
- Add payload caps to all search tools: head_limit: 20 on Grep, head_limit: 50 on Glob, | head -100 on unbounded Bash output. `[token-efficiency]` _(advanced-claude-configuration)_
- Structured tool calls get full hook enrichment (PageRank weighting 0.95, graph context, re-read blocking, session caching, payload caps); Bash equivalents bypass all enrichment and are untracked. `[enrichment]` _(tool-routing-optimization)_
- Edit sends diffs (token-efficient) while sed via Bash sends full file output. `[token-efficiency]` _(tool-routing-optimization)_
- No existing system combines per-tool-call graph enrichment + session caching + re-read tracking; this is a genuine gap. `[enrichment]` _(tool-routing-optimization)_
- OTC (RL-based tool optimization) achieved 68% fewer tool calls and 215% better tool productivity. `[measurement]` _(tool-routing-optimization)_
- Correlation vs. causation problem: sessions with higher structured-tool usage may just be simpler tasks. `[measurement]` _(tool-routing-optimization)_
- For the A/B experiment, use disallowedTools: [Read, Grep, Glob] on one agent to force Bash-only. `[measurement]` _(tool-routing-optimization)_
- Context window saturation degrades tool awareness as context fills up. `[context-window]` _(claude-code-tool-utilization-reliability)_
- Tool descriptions are the primary selection signal for which tool Claude chooses to invoke. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Tool Search Tool improved Opus 4 accuracy from 49% to 74%. `[measurement]` _(claude-code-tool-utilization-reliability)_
- Tool Use Examples improved parameter handling accuracy from 72% to 90%. `[measurement]` _(claude-code-tool-utilization-reliability)_
- Lower effort levels cause Claude to skip MCP tools in favor of faster built-in alternatives. `[model-limitations]` _(claude-code-tool-utilization-reliability)_
- alwaysThinkingEnabled: true helps counteract MCP-skip at low effort but is insufficient alone. _(claude-code-tool-utilization-reliability)_
- Tool deferred loading (Tool Search Tool pattern) cut token usage 85% while improving accuracy. `[token-efficiency]` _(claude-code-tool-utilization-reliability)_
- Cross-session tool-routing memory doesn't persist by default; Claude re-learns tool preferences every session. `[session-management]` _(claude-code-tool-utilization-reliability)_
- Move the routing table from CLAUDE.md to a tool-guide skill with user-invocable: false. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Verify MCP tool names are descriptive enough for ToolSearch matching. _(claude-code-tool-utilization-reliability)_

### `context-window` (24 points, 7 sources)

- Context momentum is an emergent property of autoregressive generation: accumulated tool-call patterns in conversation history create implicit behavioral bias that overrides explicit user corrections, hook reminders, and direct commands. `[agent-behavior]` _(context-momentum-drift)_
- Context momentum is distinct from training-prior errors (wrong values); it causes wrong behavior patterns that resist correction. `[agent-behavior]` _(context-momentum-drift)_
- After 10 turns of successful file deletions, the agent reinterpreted "refactor components to use shadcn primitives" as "delete more files," inlining component contents as raw HTML and destroying working abstractions. `[agent-behavior]` _(context-momentum-drift)_
- Corrections decay relative to context volume: a correction at turn 5 (10 prior tool calls) lands effectively; the same correction at turn 25 (100+ tool calls) is overwhelmed because the noise floor rises, not because the correction weakens. `[correction-resistance]` _(context-momentum-drift)_
- Conversation history functions as the model's working memory, and like human working memory it creates inertia; volume of prior successful actions outweighs a single corrective instruction. `[agent-behavior]` _(context-momentum-drift)_
- Paradox of late correction: the longer a session runs in one mode, the harder it is to change modes mid-session; by the time the user notices the wrong track, context momentum may already be too strong for verbal correction. `[correction-resistance]` _(context-momentum-drift)_
- The agent interprets new instructions through the lens of established patterns (pattern matching over intent); the same instruction in a fresh session would be interpreted correctly. `[agent-behavior]` _(context-momentum-drift)_
- Start new sessions for mode changes: if you've been deleting code and now want to refactor, begin a fresh session with no accumulated context pulling the wrong way. `[session-management]` _(context-momentum-drift)_
- For context freshness, add staleness detection and session-scoped summary cache (after 3rd read, DENY + inject cached summary). `[hooks]` _(bidirectional-steering-hooks)_
- 33,900 tokens (17% of 200K) consumed at boot by auto-loaded rules/CLAUDE.md before any user interaction; target is <8,000 tokens via trigger tables (proven 54% reduction). `[token-efficiency, context-boot]` _(advanced-claude-configuration)_
- Rules files re-read 22-31x per session (e.g., MEMORY.md: 31x); source files up to 90x (export.py: 90x, cli.py: 80x). `[re-read-waste, measurement]` _(advanced-claude-configuration)_
- Auto-compaction triggers at 83.5% (configurable via CLAUDE_AUTOCOMPACT_PCT_OVERRIDE); the widely cited ~50% figure is incorrect/outdated. `[compaction]` _(advanced-claude-configuration)_
- Three targeted 40K-token sessions outperform one saturated 180K session: 2.1s vs 8.2s response time, 94% vs 72% relevance. `[session-management, latency]` _(advanced-claude-configuration)_
- 75% context utilization is the quality sweet spot; performance degrades beyond this. `[measurement]` _(advanced-claude-configuration)_
- Subagent context is completely isolated -- main context session cache doesn't know about subagent reads (by design, prevents context pollution). `[subagents]` _(advanced-claude-configuration)_
- Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75 to trigger compaction earlier, preserving more working space post-compaction. `[compaction]` _(advanced-claude-configuration)_
- Use context: fork on exploration skills to prevent their reads from consuming main context. `[subagents, skill-routing]` _(advanced-claude-configuration)_
- The "enrichment might hurt" hypothesis is plausible: bloated tool context is a known failure mode. `[enrichment]` _(enrichment-ab-testing)_
- The plan is read once at session start and never re-consulted when difficulty arises -- no mechanism triggers re-reading. `[plan-compliance]` _(plan-compliance-hooks)_
- Context window saturation degrades tool awareness as context fills up. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Skill description budget is 2% of context (16K char fallback); bloated CLAUDE.md files cause Claude to ignore instructions. `[skill-routing]` _(claude-code-tool-utilization-reliability)_
- Recommended CLAUDE.md length for reliable compliance is ~12 lines. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Prune CLAUDE.md aggressively: root CLAUDE.md under 30 lines, project CLAUDE.md under 50 lines. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Agent teams (experimental) provide true parallelism with separate 1M-token context windows but at 2-3x token cost. `[subagents, cost]` _(claude-task-dag-delegation)_

### `enrichment` (21 points, 2 sources)

- Structured tool calls get full hook enrichment (PageRank weighting 0.95, graph context, re-read blocking, session caching, payload caps); Bash equivalents bypass all enrichment and are untracked. `[tool-routing]` _(tool-routing-optimization)_
- Hook enrichment is the differentiator, not just tool structure; the three-layer model (Raw Bash -> Structured Tool -> Tool + Hook) shows value is in the enrichment pipeline. `[hooks]` _(tool-routing-optimization)_
- Phase 3 of the experiment isolates the effect of hooks by comparing structured tools without hooks vs. with hooks. `[hooks]` _(tool-routing-optimization)_
- No existing system combines per-tool-call graph enrichment + session caching + re-read tracking; this is a genuine gap. `[tool-routing]` _(tool-routing-optimization)_
- Aider does PageRank but pre-prompt (not per-call); VS Code Copilot Hooks are architecturally identical but lack graph DB backing. _(tool-routing-optimization)_
- 826 enrichment injections fired over 10 days with zero measurement of whether any changed agent decisions. `[measurement]` _(enrichment-ab-testing)_
- PageRank weight injection may be counterproductive -- it reinforces existing read patterns instead of informing structural decisions. `[agent-behavior]` _(enrichment-ab-testing)_
- The memory nudge from PageRank encourages memorizing current state rather than questioning it. `[agent-behavior]` _(enrichment-ab-testing)_
- Outcome quality degraded after adding enrichment infrastructure: W09 80% good to W11 65% good, though confounders prevent attribution. `[measurement]` _(enrichment-ab-testing)_
- External evidence confirms context injections affect LLM behavior, but direction of effect is unknown for this system. `[agent-behavior]` _(enrichment-ab-testing)_
- Read-count blocking is the only enrichment component with demonstrated value. `[re-read-waste]` _(enrichment-ab-testing)_
- All four experimental configs keep read-count blocking enabled. `[measurement]` _(enrichment-ab-testing)_
- The experiment decomposes enrichment into four independently testable components: PageRank injection, GitNexus-on-Read, GitNexus-on-Search/Edit, and read-count blocking. `[decomposition]` _(enrichment-ab-testing)_
- Payload caps (head_limit) should stay on regardless of enrichment config. `[token-efficiency]` _(enrichment-ab-testing)_
- The "enrichment might hurt" hypothesis is plausible: bloated tool context is a known failure mode. `[context-window]` _(enrichment-ab-testing)_
- Reinforcing existing patterns through enrichment could suppress exploration of unfamiliar but important files. `[agent-behavior]` _(enrichment-ab-testing)_
- Run the 4-config A/B test using CLAUDE_ENRICHMENT_CONFIG=A|B|C|D. `[measurement]` _(enrichment-ab-testing)_
- Log the active config in session-end metadata so it flows into usage.db. `[observability]` _(enrichment-ab-testing)_
- Need ~50 sessions per config (~4 weeks) to reach minimum statistical power. `[measurement]` _(enrichment-ab-testing)_
- Measure good_rate, wrong_approach friction count, and avg_tokens per config. `[measurement]` _(enrichment-ab-testing)_
- Default to "blocking only" (Config B) if no config shows significant advantage. `[infrastructure-values]` _(enrichment-ab-testing)_

### `nudging` (20 points, 5 sources)

- Hooks end up "cleaning up the agent's mess" rather than changing behavior; the agent does not learn from blocks across sessions. `[hooks]` _(agent-nudging-design)_
- Making generation "feel slower or riskier" could flip the default from generate to lookup, but no mechanism exists for this yet. `[agent-behavior]` _(agent-nudging-design)_
- The paradox is recursive: every mitigation that depends on the agent attending to something is undermined by the same overconfidence that causes the original problem. `[correction-resistance]` _(agent-nudging-design)_
- The nudge must be attended to, but confidence causes inattention — creating a recursive loop. `[correction-resistance]` _(agent-nudging-design)_
- "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026) confirms small prompt changes shift behavior significantly, but the nudge-attention paradox limits applicability. `[agent-behavior]` _(agent-nudging-design)_
- Adversarial injection should be selective -- only for recommendation/design agents, not search/explore agents. `[hooks]` _(bidirectional-steering-hooks)_
- Nudge hypersensitivity is a real risk: MIT ICLR 2026 paper shows LLMs respond far more strongly to nudges than humans. `[agreement-bias]` _(tool-routing-optimization)_
- Each additionalContext injection is 20-50 tokens; if it prevents one re-read (~2,000 tokens) the ROI is 40-100x, but ignored nudges are pure waste. `[token-efficiency]` _(tool-routing-optimization)_
- Over-nudging creates brittleness in agent behavior. `[correction-resistance]` _(tool-routing-optimization)_
- LLMs are hypersensitive to nudges: weak cues have disproportionately large effects (ICLR 2026 paper). `[agent-behavior]` _(claude-code-tool-utilization-reliability)_
- Hook-injected additionalContext is a direct implementation of the nudge effect. `[hooks]` _(claude-code-tool-utilization-reliability)_
- Over-nudging creates brittleness and token waste (~20-50 tokens per injection). `[token-efficiency]` _(claude-code-tool-utilization-reliability)_
- Implement a UserPromptSubmit nudge hook that pattern-matches prompts and injects tool suggestions. `[hooks]` _(claude-code-tool-utilization-reliability)_
- Start nudge hook with 3-5 high-confidence patterns: code structure -> gitnexus, docs -> lab-docs/context7, memory -> memory MCP. `[hooks]` _(claude-code-tool-utilization-reliability)_
- Without explicit nudging, Claude executes complex multi-step prompts monolithically, causing context pollution, zero parallelism, no failure isolation, and poor resumability. `[decomposition, agent-behavior]` _(claude-task-dag-delegation)_
- No single mechanism reliably changes Claude's behavior across all prompt complexities; CLAUDE.md nudges suffer the "nudge paradox." `[correction-resistance, agent-behavior]` _(claude-task-dag-delegation)_
- The nudge paradox creates a graceful-degradation argument for the hybrid: the nudge may be ignored, but skill and subagent remain as explicit escalation paths. `[correction-resistance]` _(claude-task-dag-delegation)_
- Auto-detection vs. explicit invocation is an unsolved tension: auto-detection risks the nudge paradox, explicit invocation adds friction. `[decomposition]` _(claude-task-dag-delegation)_
- The hybrid "nudge suggests, user confirms" may be the sweet spot for triggering decomposition. `[prompt-design]` _(claude-task-dag-delegation)_
- The CLAUDE.md nudge must include a "do NOT decompose" guardrail for simple single-purpose prompts. `[prompt-design, agent-behavior]` _(claude-task-dag-delegation)_

### `token-efficiency` (20 points, 6 sources)

- Generation is chosen over lookup because it's faster — the agent avoids tool-call round-trips when confident. `[agent-behavior]` _(agent-nudging-design)_
- Reactive block-and-retry may be "good enough" pragmatically: catches errors reliably for registered values, preserves output correctness, but wastes tokens composing wrong content. `[hooks]` _(agent-nudging-design)_
- 33,900 tokens (17% of 200K) consumed at boot by auto-loaded rules/CLAUDE.md before any user interaction; target is <8,000 tokens via trigger tables (proven 54% reduction). `[context-window, context-boot]` _(advanced-claude-configuration)_
- Structured prompts (tables, YAML, bullets) preserve 92% fidelity during compaction vs 71% for narrative prose. `[compaction]` _(advanced-claude-configuration)_
- updatedInput rewriting is strictly better than blocking reads: same hook count, 50-90% less context consumed. `[hooks, tool-routing]` _(advanced-claude-configuration)_
- Masking beats summarization for compaction -- cheaper, no trajectory elongation, +2.6% solve rate (JetBrains research). `[compaction]` _(advanced-claude-configuration)_
- Every competitor (Aider, Cursor, Cody) returns snippets, never full files. `[agent-behavior]` _(advanced-claude-configuration)_
- Replace verbose rules files with trigger tables to cut boot context by 54%+. `[context-boot, prompt-design]` _(advanced-claude-configuration)_
- Add payload caps to all search tools: head_limit: 20 on Grep, head_limit: 50 on Glob, | head -100 on unbounded Bash output. `[tool-routing]` _(advanced-claude-configuration)_
- Payload caps prevent context floods with no behavioral change from Claude. `[agent-behavior]` _(advanced-claude-configuration)_
- Keep Context Mode MCP and aggressive output compression away from code editing -- they lose exact error messages, line numbers, variable names. `[model-limitations]` _(advanced-claude-configuration)_
- Context Mode MCP and aggressive output compression are only suitable for research-only sessions. _(advanced-claude-configuration)_
- Edit sends diffs (token-efficient) while sed via Bash sends full file output. `[tool-routing]` _(tool-routing-optimization)_
- Each additionalContext injection is 20-50 tokens; if it prevents one re-read (~2,000 tokens) the ROI is 40-100x, but ignored nudges are pure waste. `[nudging]` _(tool-routing-optimization)_
- Use TEFS and tool productivity as primary metrics; they penalize correct-but-inefficient patterns. `[measurement]` _(tool-routing-optimization)_
- Payload caps (head_limit) should stay on regardless of enrichment config. `[enrichment]` _(enrichment-ab-testing)_
- Over-nudging creates brittleness and token waste (~20-50 tokens per injection). `[nudging]` _(claude-code-tool-utilization-reliability)_
- Tool deferred loading (Tool Search Tool pattern) cut token usage 85% while improving accuracy. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Deferred tool loading is already partially implemented via available-deferred-tools. _(claude-code-tool-utilization-reliability)_
- Cost-conscious mode needed for constrained environments (OSC): decompose for clarity but execute sequentially to avoid multiplied token usage. `[cost, subagents]` _(claude-task-dag-delegation)_

### `subagents` (19 points, 3 sources)

- Haiku hallucinates filenames and misses bugs that Sonnet catches; Opus finds subtle bugs (resource leaks, concurrency) that both miss. `[model-limitations]` _(advanced-claude-configuration)_
- Use Haiku only for read/summarize (Explore-style); use Sonnet/inherit for design decisions. `[skill-routing]` _(advanced-claude-configuration)_
- Subagent context is completely isolated -- main context session cache doesn't know about subagent reads (by design, prevents context pollution). `[context-window]` _(advanced-claude-configuration)_
- A file must be read twice if both main agent and subagent contexts need it. `[re-read-waste]` _(advanced-claude-configuration)_
- Agent Teams experimental, not production-ready -- session resumption broken, task coordination lag; subagents are the reliable path. `[infrastructure-values]` _(advanced-claude-configuration)_
- Move detailed architecture content into agent system prompts and skill references (out of boot context). `[context-boot]` _(advanced-claude-configuration)_
- Cap 3-4 domain expert agents max -- more creates decision overhead. `[decomposition]` _(advanced-claude-configuration)_
- Each agent should enforce an output contract (structured summary with file:line refs, max 500 words) in its system prompt. `[enforcement, prompt-design]` _(advanced-claude-configuration)_
- Use context: fork on exploration skills to prevent their reads from consuming main context. `[context-window, skill-routing]` _(advanced-claude-configuration)_
- Impartiality requires context separation: a reviewer agent in a fresh context avoids compilation pressure and momentum bias. `[hooks]` _(plan-compliance-hooks)_
- Same model family used for review may still share blind spots with the main agent. `[hooks, model-limitations]` _(plan-compliance-hooks)_
- Reserve the agent-based hook (Approach 2) for high-risk phases only where 10-30s latency per edit is justified. `[hooks, latency]` _(plan-compliance-hooks)_
- Subagents cannot spawn other subagents; an orchestrator can only use Task/Agent tool for general-purpose workers. `[model-limitations, infrastructure-values]` _(claude-task-dag-delegation)_
- Worker specialization must be inlined into the task prompt because subagent nesting is blocked. `[model-limitations, agent-behavior]` _(claude-task-dag-delegation)_
- Agent teams (experimental) provide true parallelism with separate 1M-token context windows but at 2-3x token cost. `[cost, context-window]` _(claude-task-dag-delegation)_
- Agent teams are overkill for most day-to-day prompts. `[agent-behavior]` _(claude-task-dag-delegation)_
- Cost-conscious mode needed for constrained environments (OSC): decompose for clarity but execute sequentially to avoid multiplied token usage. `[cost, token-efficiency]` _(claude-task-dag-delegation)_
- Reserve agent teams for genuinely large parallel workloads; invoke manually, never as a default. `[agent-behavior]` _(claude-task-dag-delegation)_
- Orchestrator workers need specialization inlined into task prompts since subagent nesting is blocked. `[prompt-design]` _(claude-task-dag-delegation)_

### `correction-resistance` (18 points, 5 sources)

- Corrections decay relative to context volume: a correction at turn 5 (10 prior tool calls) lands effectively; the same correction at turn 25 (100+ tool calls) is overwhelmed because the noise floor rises, not because the correction weakens. `[context-window]` _(context-momentum-drift)_
- Selective compliance emerges: the agent follows instructions aligned with current momentum and resists opposing ones, acknowledging hook reminders verbally ("read sibling files first") but not changing tool-call behavior. `[agent-behavior]` _(context-momentum-drift)_
- Escalation blindness: increasing user frustration does not proportionally increase correction weight; "stop" is processed as "pause briefly" because overwhelming context says "keep going." `[agent-behavior]` _(context-momentum-drift)_
- Paradox of late correction: the longer a session runs in one mode, the harder it is to change modes mid-session; by the time the user notices the wrong track, context momentum may already be too strong for verbal correction. `[context-window]` _(context-momentum-drift)_
- Instructional reminders (hooks, system prompts) are necessary but insufficient -- the agent can read, acknowledge, and then ignore them when they conflict with accumulated behavioral context. `[hooks]` _(context-momentum-drift)_
- Front-load constraints: "Refactor components to use shadcn primitives -- do NOT delete any component files" works better than correcting after deletion has started. `[prompt-design]` _(context-momentum-drift)_
- Don't escalate -- redirect: calm, specific alternative instructions provide a new pattern to follow rather than negating the current one. `[prompt-design]` _(context-momentum-drift)_
- The agent is most likely to skip tool lookups when most confident from training priors, but project-specific infrastructure values (SLURM partitions, module versions, paths, config keys) are exactly where training priors are most wrong. `[infrastructure-values]` _(agent-nudging-design)_
- High confidence correlates with high error rate for values that deviate from common patterns — the confidence-error correlation is inverted. `[infrastructure-values]` _(agent-nudging-design)_
- PreToolUse hooks only fire when the agent already decided to use a tool, but the problem is the agent skips tools when confident — so the correction mechanism requires the very behavior it aims to create. `[hooks]` _(agent-nudging-design)_
- The paradox is recursive: every mitigation that depends on the agent attending to something is undermined by the same overconfidence that causes the original problem. `[nudging]` _(agent-nudging-design)_
- The nudge must be attended to, but confidence causes inattention — creating a recursive loop. `[nudging]` _(agent-nudging-design)_
- The citation approach (3) is the most promising proactive strategy but requires the missing pre-composition hook point to enforce. `[prompt-design]` _(agent-nudging-design)_
- Do not rely on rules alone — without enforcement, rules compete with training priors and lose when the agent is confident. `[prompt-design]` _(agent-nudging-design)_
- Over-nudging creates brittleness in agent behavior. `[nudging]` _(tool-routing-optimization)_
- LLM agents systematically ignore implementation plans when they hit errors, defaulting to improvised custom code instead of the plan's prescribed replacement. `[plan-compliance, agent-behavior]` _(plan-compliance-hooks)_
- No single mechanism reliably changes Claude's behavior across all prompt complexities; CLAUDE.md nudges suffer the "nudge paradox." `[nudging, agent-behavior]` _(claude-task-dag-delegation)_
- The nudge paradox creates a graceful-degradation argument for the hybrid: the nudge may be ignored, but skill and subagent remain as explicit escalation paths. `[nudging]` _(claude-task-dag-delegation)_

### `skill-routing` (13 points, 3 sources)

- Skill-based assessment (Approach B) is probabilistic because it depends on the model choosing to invoke it, which is unreliable. `[agent-behavior]` _(bidirectional-steering-hooks)_
- Use Haiku only for read/summarize (Explore-style); use Sonnet/inherit for design decisions. `[subagents]` _(advanced-claude-configuration)_
- Use context: fork on exploration skills to prevent their reads from consuming main context. `[subagents, context-window]` _(advanced-claude-configuration)_
- Skill description budget is 2% of context (16K char fallback); bloated CLAUDE.md files cause Claude to ignore instructions. `[context-window]` _(claude-code-tool-utilization-reliability)_
- Skill auto-invocation depends on fuzzy name+description matching. _(claude-code-tool-utilization-reliability)_
- Overlapping skill descriptions cause wrong-skill selection. _(claude-code-tool-utilization-reliability)_
- Missing user-vocabulary keywords in skill descriptions prevent triggering entirely. _(claude-code-tool-utilization-reliability)_
- Skill descriptions should be written for user vocabulary, not developer vocabulary. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Skill descriptions should include trigger phrases users actually say and be mutually exclusive. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Rewrite all skill descriptions with natural-language trigger phrases and make them mutually exclusive. `[prompt-design]` _(claude-code-tool-utilization-reliability)_
- Test skill descriptions with "What skills are available?" to verify discoverability. _(claude-code-tool-utilization-reliability)_
- Side-effect skills (deploy, commit) must set disable-model-invocation: true. `[enforcement]` _(claude-code-tool-utilization-reliability)_
- Set effort: high on skills that depend on MCP tools to counteract the effort-level MCP skip pattern. `[enforcement]` _(claude-code-tool-utilization-reliability)_

### `model-limitations` (12 points, 4 sources)

- Root cause of re-read waste: Claude lacks session-scoped memory of prior reads; every reasoning turn starts fresh. `[re-read-waste]` _(advanced-claude-configuration)_
- Text generation cannot be hooked -- no interception during reasoning/analysis. `[hooks]` _(advanced-claude-configuration)_
- PostToolUse cannot modify tool results (Anthropic status: NOT_PLANNED). `[hooks]` _(advanced-claude-configuration)_
- Multi-hook on same matcher silently drops earlier updatedInput (bug #15897). `[hooks]` _(advanced-claude-configuration)_
- Haiku hallucinates filenames and misses bugs that Sonnet catches; Opus finds subtle bugs (resource leaks, concurrency) that both miss. `[subagents]` _(advanced-claude-configuration)_
- Filesystem offloading (read summary, then Edit) is dangerous: Claude won't have exact text for old_string matching, causing coherence failures. `[tool-routing]` _(advanced-claude-configuration)_
- Keep Context Mode MCP and aggressive output compression away from code editing -- they lose exact error messages, line numbers, variable names. `[token-efficiency]` _(advanced-claude-configuration)_
- Root cause of plan deviation: the agent cannot distinguish guessing from knowing -- prioritizes "code that compiles" over plan compliance. `[agent-behavior]` _(plan-compliance-hooks)_
- Same model family used for review may still share blind spots with the main agent. `[hooks, subagents]` _(plan-compliance-hooks)_
- Lower effort levels cause Claude to skip MCP tools in favor of faster built-in alternatives. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Subagents cannot spawn other subagents; an orchestrator can only use Task/Agent tool for general-purpose workers. `[subagents, infrastructure-values]` _(claude-task-dag-delegation)_
- Worker specialization must be inlined into the task prompt because subagent nesting is blocked. `[subagents, agent-behavior]` _(claude-task-dag-delegation)_

### `observability` (11 points, 7 sources)

- Track behavioral diversity as a health signal: if 90%+ of recent tool calls are Edit/Write with no Read/Grep, the agent is likely in a momentum rut and a hook could flag this. `[tool-routing]` _(context-momentum-drift)_
- Distinguishing "agent looked it up" from "agent generated correctly by chance" from "agent generated wrong and got blocked" is necessary to evaluate any intervention but no instrumentation exists yet. `[measurement]` _(agent-nudging-design)_
- Measurement is an unsolved prerequisite for evaluating any nudging intervention. `[measurement]` _(agent-nudging-design)_
- Instrument before optimizing (Phase 0): add read logging to PreToolUse, track path/count/mtime/session position. `[measurement, hooks]` _(advanced-claude-configuration)_
- No external observability tool natively imports SQLite or JSONL; all require a backfill script. `[infrastructure-values]` _(tool-routing-optimization)_
- Arize Phoenix is the only evaluated observability tool that runs on OSC (pip install, single process, SQLite backend, no Docker). `[infrastructure-values]` _(tool-routing-optimization)_
- Prefer cloud free tiers over self-hosting multi-container stacks on HPC. `[infrastructure-values]` _(tool-routing-optimization)_
- OTel instrumentation is the portable path forward: instrument hooks once with otel-cli, redirect spans to any backend. `[infrastructure-values]` _(tool-routing-optimization)_
- Log the active config in session-end metadata so it flows into usage.db. `[enrichment]` _(enrichment-ab-testing)_
- Each pattern changes context dynamics -- implementing multiple simultaneously makes effects unattributable. `[measurement]` _(claude-code-tool-utilization-reliability)_
- Instrument before tuning: add hooks or logging to measure prompt complexity vs. whether decomposition was actually used. `[measurement, hooks]` _(claude-task-dag-delegation)_

### `decomposition` (9 points, 4 sources)

- Cap 3-4 domain expert agents max -- more creates decision overhead. `[subagents]` _(advanced-claude-configuration)_
- The experiment decomposes enrichment into four independently testable components: PageRank injection, GitNexus-on-Read, GitNexus-on-Search/Edit, and read-count blocking. `[enrichment]` _(enrichment-ab-testing)_
- Implement changes sequentially, not simultaneously; priority order: prune CLAUDE.md -> audit skill descriptions -> hook nudging -> routing skill. `[measurement]` _(claude-code-tool-utilization-reliability)_
- Without explicit nudging, Claude executes complex multi-step prompts monolithically, causing context pollution, zero parallelism, no failure isolation, and poor resumability. `[nudging, agent-behavior]` _(claude-task-dag-delegation)_
- Complexity thresholds ("3+ steps", "6+ steps") are arbitrary and need empirical calibration. `[measurement]` _(claude-task-dag-delegation)_
- The tiered approach matches how human PMs think: small tasks just do, medium tasks plan first, big tasks delegate. `[prompt-design]` _(claude-task-dag-delegation)_
- Three mechanisms map to the tiers: CLAUDE.md nudge, /plan-and-execute skill, orchestrator subagent. `[infrastructure-values]` _(claude-task-dag-delegation)_
- Auto-detection vs. explicit invocation is an unsolved tension: auto-detection risks the nudge paradox, explicit invocation adds friction. `[nudging]` _(claude-task-dag-delegation)_
- Build incrementally, cheapest tier first: Week 1 = CLAUDE.md nudge + /plan-and-execute, Week 2 = orchestrator subagent, Week 3 = evaluate. `[infrastructure-values]` _(claude-task-dag-delegation)_

### `session-management` (8 points, 5 sources)

- Start new sessions for mode changes: if you've been deleting code and now want to refactor, begin a fresh session with no accumulated context pulling the wrong way. `[context-window]` _(context-momentum-drift)_
- Session length is a risk factor: surface warnings or suggest session breaks after N turns of the same tool-call pattern, especially if user corrections have appeared. `[measurement]` _(context-momentum-drift)_
- Three targeted 40K-token sessions outperform one saturated 180K session: 2.1s vs 8.2s response time, 94% vs 72% relevance. `[context-window, latency]` _(advanced-claude-configuration)_
- Log approvals across sessions so the same edit isn't re-reviewed after a context compact. `[hooks, compaction]` _(plan-compliance-hooks)_
- Cross-session tool-routing memory doesn't persist by default; Claude re-learns tool preferences every session. `[tool-routing]` _(claude-code-tool-utilization-reliability)_
- Tool routing preferences persist across sessions only if stored in memory MCP or MEMORY.md. _(claude-code-tool-utilization-reliability)_
- External state (tasks.json) is the key differentiator over monolithic execution; it survives compaction, enables resumption. `[compaction]` _(claude-task-dag-delegation)_
- JSON is harder for Claude to update reliably than markdown; format choice for external state needs testing. `[agent-behavior]` _(claude-task-dag-delegation)_

### `compaction` (7 points, 3 sources)

- Auto-compaction triggers at 83.5% (configurable via CLAUDE_AUTOCOMPACT_PCT_OVERRIDE); the widely cited ~50% figure is incorrect/outdated. `[context-window]` _(advanced-claude-configuration)_
- Structured prompts (tables, YAML, bullets) preserve 92% fidelity during compaction vs 71% for narrative prose. `[token-efficiency]` _(advanced-claude-configuration)_
- Masking beats summarization for compaction -- cheaper, no trajectory elongation, +2.6% solve rate (JetBrains research). `[token-efficiency]` _(advanced-claude-configuration)_
- Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75 to trigger compaction earlier, preserving more working space post-compaction. `[context-window]` _(advanced-claude-configuration)_
- Write PreCompact hook that serializes session read cache to a recovery file so post-compaction context includes "you already read these files." `[re-read-waste, hooks]` _(advanced-claude-configuration)_
- Log approvals across sessions so the same edit isn't re-reviewed after a context compact. `[hooks, session-management]` _(plan-compliance-hooks)_
- External state (tasks.json) is the key differentiator over monolithic execution; it survives compaction, enables resumption. `[session-management]` _(claude-task-dag-delegation)_

### `re-read-waste` (6 points, 2 sources)

- Rules files re-read 22-31x per session (e.g., MEMORY.md: 31x); source files up to 90x (export.py: 90x, cli.py: 80x). `[context-window, measurement]` _(advanced-claude-configuration)_
- Root cause of re-read waste: Claude lacks session-scoped memory of prior reads; every reasoning turn starts fresh. `[model-limitations]` _(advanced-claude-configuration)_
- A file must be read twice if both main agent and subagent contexts need it. `[subagents]` _(advanced-claude-configuration)_
- Build smart-context.sh for Read: log first reads, warn on 2nd, narrow/deny on 3rd+ re-reads of unchanged files. `[hooks]` _(advanced-claude-configuration)_
- Write PreCompact hook that serializes session read cache to a recovery file so post-compaction context includes "you already read these files." `[compaction, hooks]` _(advanced-claude-configuration)_
- Read-count blocking is the only enrichment component with demonstrated value. `[enrichment]` _(enrichment-ab-testing)_

### `context-boot` (5 points, 2 sources)

- Approach 5 (pre-composition context injection): right timing but no reliable signal for "about to edit" vs "just reading"; if the agent skips Read entirely, back to the paradox. `[hooks]` _(agent-nudging-design)_
- "Codified Context" paper (arXiv 2602.20478) addresses context injection but not the upstream tool-use avoidance problem. `[agent-behavior]` _(agent-nudging-design)_
- 33,900 tokens (17% of 200K) consumed at boot by auto-loaded rules/CLAUDE.md before any user interaction; target is <8,000 tokens via trigger tables (proven 54% reduction). `[context-window, token-efficiency]` _(advanced-claude-configuration)_
- Replace verbose rules files with trigger tables to cut boot context by 54%+. `[token-efficiency, prompt-design]` _(advanced-claude-configuration)_
- Move detailed architecture content into agent system prompts and skill references (out of boot context). `[subagents]` _(advanced-claude-configuration)_

### `agreement-bias` (5 points, 3 sources)

- Leading prompt framing (e.g., "isn't X the right approach?") triggers agreement bias and should be reframed as neutral evaluation requests. `[prompt-design]` _(bidirectional-steering-hooks)_
- The /steer skill should extract: core technical goal, relevant constraints, what a verifiable answer looks like, and flags for emotional valence / leading framing / scope ambiguity. `[prompt-design]` _(bidirectional-steering-hooks)_
- /steer should reframe leading questions as neutral evaluations, not censor or remove informative frustration. `[prompt-design]` _(bidirectional-steering-hooks)_
- Nudge hypersensitivity is a real risk: MIT ICLR 2026 paper shows LLMs respond far more strongly to nudges than humans. `[nudging]` _(tool-routing-optimization)_
- Self-assessed outcomes (facets) may be unreliable -- METR found developers misjudged AI helpfulness by ~40%. `[measurement]` _(enrichment-ab-testing)_

### `plan-compliance` (5 points, 2 sources)

- Start with Phase 1 (observational SQL analysis) before building anything. `[measurement]` _(tool-routing-optimization)_
- LLM agents systematically ignore implementation plans when they hit errors, defaulting to improvised custom code instead of the plan's prescribed replacement. `[agent-behavior, correction-resistance]` _(plan-compliance-hooks)_
- The improvisation chain follows a repeatable pattern: read plan -> start editing -> hit error -> write custom code that compiles -> user pushes back -> repeat. `[agent-behavior]` _(plan-compliance-hooks)_
- The plan is read once at session start and never re-consulted when difficulty arises -- no mechanism triggers re-reading. `[context-window]` _(plan-compliance-hooks)_
- Legitimate deviation creates a paradox: if the agent discovers the plan is wrong, the hook blocks editing the plan file itself. `[hooks]` _(plan-compliance-hooks)_

### `latency` (4 points, 3 sources)

- Three targeted 40K-token sessions outperform one saturated 180K session: 2.1s vs 8.2s response time, 94% vs 72% relevance. `[session-management, context-window]` _(advanced-claude-configuration)_
- Use Node.js (not Python) for latency-sensitive hooks -- Node.js warm startup is ~20-50ms vs Python's ~100-200ms. `[hooks]` _(advanced-claude-configuration)_
- Do NOT add Bash-to-Read nudge hooks until Phase 1 results are in; a pattern-check hook costs 50-100ms per call with no proven benefit. `[hooks]` _(tool-routing-optimization)_
- Reserve the agent-based hook (Approach 2) for high-risk phases only where 10-30s latency per edit is justified. `[hooks, subagents]` _(plan-compliance-hooks)_

### `bash-duplication` (4 points, 3 sources)

- ~40% of Bash calls duplicate structured tools: 612 ls, 349 grep, 130 cat out of 3,995 total Bash calls, stable at 37-40% across time and resistant to instructions alone. `[measurement]` _(tool-routing-optimization)_
- The hook creates a Bash escape hatch problem: the agent can bypass Edit/Write hooks by using sed or echo > via Bash. `[hooks]` _(plan-compliance-hooks)_
- Extend the hook's matcher to cover Bash to prevent agents from bypassing Edit/Write via shell commands. `[hooks, enforcement]` _(plan-compliance-hooks)_
- ~40% of Bash calls duplicate structured tools (Read, Grep, Glob, Edit) -- Claude defaults to raw Bash for file operations despite purpose-built alternatives. _(claude-code-tool-utilization-reliability)_

### `cost` (4 points, 3 sources)

- Langfuse requires 4 containers and ~32GB RAM, making it impractical on HPC. `[infrastructure-values]` _(tool-routing-optimization)_
- Ollama local models offer a sweet spot (~1-3s, $0, different model family for true impartiality) but are environment-dependent. `[hooks, infrastructure-values]` _(plan-compliance-hooks)_
- Agent teams (experimental) provide true parallelism with separate 1M-token context windows but at 2-3x token cost. `[subagents, context-window]` _(claude-task-dag-delegation)_
- Cost-conscious mode needed for constrained environments (OSC): decompose for clarity but execute sequentially to avoid multiplied token usage. `[subagents, token-efficiency]` _(claude-task-dag-delegation)_
