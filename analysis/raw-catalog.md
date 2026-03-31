# Raw Catalog — All Points Tagged by Source

## Source: context-momentum-drift
context-window, agent-behavior | Context momentum is an emergent property of autoregressive generation: accumulated tool-call patterns in conversation history create implicit behavioral bias that overrides explicit user corrections, hook reminders, and direct commands.
context-window, agent-behavior | Context momentum is distinct from training-prior errors (wrong values); it causes wrong behavior patterns that resist correction.
context-window, agent-behavior | After 10 turns of successful file deletions, the agent reinterpreted "refactor components to use shadcn primitives" as "delete more files," inlining component contents as raw HTML and destroying working abstractions.
context-window, correction-resistance | Corrections decay relative to context volume: a correction at turn 5 (10 prior tool calls) lands effectively; the same correction at turn 25 (100+ tool calls) is overwhelmed because the noise floor rises, not because the correction weakens.
correction-resistance, agent-behavior | Selective compliance emerges: the agent follows instructions aligned with current momentum and resists opposing ones, acknowledging hook reminders verbally ("read sibling files first") but not changing tool-call behavior.
correction-resistance, agent-behavior | Escalation blindness: increasing user frustration does not proportionally increase correction weight; "stop" is processed as "pause briefly" because overwhelming context says "keep going."
context-window, agent-behavior | Conversation history functions as the model's working memory, and like human working memory it creates inertia; volume of prior successful actions outweighs a single corrective instruction.
context-window, correction-resistance | Paradox of late correction: the longer a session runs in one mode, the harder it is to change modes mid-session; by the time the user notices the wrong track, context momentum may already be too strong for verbal correction.
context-window, agent-behavior | The agent interprets new instructions through the lens of established patterns (pattern matching over intent); the same instruction in a fresh session would be interpreted correctly.
hooks, correction-resistance | Instructional reminders (hooks, system prompts) are necessary but insufficient -- the agent can read, acknowledge, and then ignore them when they conflict with accumulated behavioral context.
session-management, context-window | Start new sessions for mode changes: if you've been deleting code and now want to refactor, begin a fresh session with no accumulated context pulling the wrong way.
prompt-design, correction-resistance | Front-load constraints: "Refactor components to use shadcn primitives -- do NOT delete any component files" works better than correcting after deletion has started.
prompt-design, correction-resistance | Don't escalate -- redirect: calm, specific alternative instructions provide a new pattern to follow rather than negating the current one.
enforcement, hooks | Corrections need mechanical enforcement, not reminders: a blocking hook that refuses writes until siblings are read would have prevented the damage; a pre-hook reminder was acknowledged and ignored.
session-management, measurement | Session length is a risk factor: surface warnings or suggest session breaks after N turns of the same tool-call pattern, especially if user corrections have appeared.
observability, tool-routing | Track behavioral diversity as a health signal: if 90%+ of recent tool calls are Edit/Write with no Read/Grep, the agent is likely in a momentum rut and a hook could flag this.
agent-behavior, enforcement | Architecture-level fixes needed: recency weighting for user instructions, explicit mode-transition mechanisms, and correction amplification (user "no"/"stop" should receive outsized attention vs. prior approvals).
enforcement, agent-behavior | The solution is not better instructions but mechanical constraints that make wrong behavior impossible rather than merely discouraged.

## Source: agent-nudging-design
infrastructure-values, correction-resistance | The agent is most likely to skip tool lookups when most confident from training priors, but project-specific infrastructure values (SLURM partitions, module versions, paths, config keys) are exactly where training priors are most wrong.
infrastructure-values, correction-resistance | High confidence correlates with high error rate for values that deviate from common patterns — the confidence-error correlation is inverted.
hooks, correction-resistance | PreToolUse hooks only fire when the agent already decided to use a tool, but the problem is the agent skips tools when confident — so the correction mechanism requires the very behavior it aims to create.
hooks, nudging | Hooks end up "cleaning up the agent's mess" rather than changing behavior; the agent does not learn from blocks across sessions.
infrastructure-values, agent-behavior | executor.py was written with partition_cpu="serial" despite the agent having read scripts showing "cpu" earlier in the same session.
infrastructure-values, agent-behavior | The serial-vs-cpu failure pattern recurs daily across SLURM params, module versions (cuda/12.4 vs cuda/12.4.1), paths, and config field names.
hooks, enforcement | Approach 1 (Edit/Write validation hooks): catches errors for registered values but is pure error correction, requires manual registry upkeep, does not change behavior.
prompt-design, enforcement | Approach 2 (rules "always look up"): zero cost but no enforcement; confidence overrides rule attendance due to the hook paradox.
prompt-design, enforcement | Approach 3 (citation requirements before edits): forces lookup cycles but no hook point exists before the agent composes edit content; agent can cite files it read yet still generate from priors.
infrastructure-values, agent-behavior | Approach 4 (codebase constants file): eliminates the problem structurally for code files but doesn't cover docs, plans, or new projects.
hooks, context-boot | Approach 5 (pre-composition context injection): right timing but no reliable signal for "about to edit" vs "just reading"; if the agent skips Read entirely, back to the paradox.
agent-behavior, token-efficiency | Generation is chosen over lookup because it's faster — the agent avoids tool-call round-trips when confident.
nudging, agent-behavior | Making generation "feel slower or riskier" could flip the default from generate to lookup, but no mechanism exists for this yet.
correction-resistance, nudging | The paradox is recursive: every mitigation that depends on the agent attending to something is undermined by the same overconfidence that causes the original problem.
nudging, correction-resistance | The nudge must be attended to, but confidence causes inattention — creating a recursive loop.
agent-behavior, infrastructure-values | Codebase design for agent consumption is a double-edged sword: constants files have independent maintainability benefits for humans, making them defensible.
agent-behavior, infrastructure-values | Designing codebases primarily to work around agent limitations is described as a "new and possibly misguided pattern."
hooks, token-efficiency | Reactive block-and-retry may be "good enough" pragmatically: catches errors reliably for registered values, preserves output correctness, but wastes tokens composing wrong content.
hooks, infrastructure-values | The question for reactive approach is whether the registry maintenance cost is sustainable.
measurement, observability | Distinguishing "agent looked it up" from "agent generated correctly by chance" from "agent generated wrong and got blocked" is necessary to evaluate any intervention but no instrumentation exists yet.
measurement, observability | Measurement is an unsolved prerequisite for evaluating any nudging intervention.
hooks, infrastructure-values | Start with approach 1 (Edit/Write validation hook + value registry) immediately — low effort, stops the bleeding for known-risky values.
infrastructure-values, agent-behavior | Centralize infrastructure constants (approach 4) where it makes independent sense — SLURM constants in KD-GAT benefit human maintainability regardless of agent behavior.
measurement, hooks | Instrument the validation hook to measure block rates by value category — high block rates identify which categories need stronger nudging.
hooks, enforcement | Investigate whether a hook point can fire after "decide to edit" but before "compose edit content" — this is the key missing primitive.
prompt-design, correction-resistance | The citation approach (3) is the most promising proactive strategy but requires the missing pre-composition hook point to enforce.
prompt-design, correction-resistance | Do not rely on rules alone — without enforcement, rules compete with training priors and lose when the agent is confident.
nudging, agent-behavior | "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026) confirms small prompt changes shift behavior significantly, but the nudge-attention paradox limits applicability.
context-boot, agent-behavior | "Codified Context" paper (arXiv 2602.20478) addresses context injection but not the upstream tool-use avoidance problem.

## Source: bidirectional-steering-hooks
hooks, agent-behavior | PreToolUse hook on Agent tool (Approach C) is the strongest output-side steering mechanism because it fires deterministically, requires no model choice, injects review context before the agent runs, and adds zero extra agent calls.
hooks, agent-behavior | Stop hooks (Approach A) are too late because they fire after the response is already displayed, making review post-hoc rather than inline.
skill-routing, agent-behavior | Skill-based assessment (Approach B) is probabilistic because it depends on the model choosing to invoke it, which is unreliable.
prompt-design, hooks | Input-side mediation via /steer skill is preferred over user-prompt-submit hook for transparency (user sees restructured prompt), but the hook is mechanically stronger since it fires before the model sees the prompt.
hooks, nudging | Adversarial injection should be selective -- only for recommendation/design agents, not search/explore agents.
agreement-bias, prompt-design | Leading prompt framing (e.g., "isn't X the right approach?") triggers agreement bias and should be reframed as neutral evaluation requests.
hooks, prompt-design | Bidirectional steering requires both input and output intervention -- neither alone is sufficient.
hooks, prompt-design | The compound effect of bidirectional steering is hypothesized to be greater than either alone, but controlled measurement is hard in real work sessions.
prompt-design, hooks | Transparency and mechanical strength are in tension: skill approach gives user visibility, hook approach has stronger guarantees; unresolved design tradeoff.
hooks, infrastructure-values | Existing hook infrastructure can be extended, not replaced: payload control, context freshness, and tool nudging are incremental improvements.
hooks, agent-behavior | The adversarial injection pattern -- modifying an existing tool call's context rather than spawning a second agent -- is a general technique for adding verification without latency cost.
hooks, enforcement | The three injected adversarial checks (verifiable vs. assertion labeling, omission listing, project-ceiling scaling) are domain-portable.
hooks, measurement | User fatigue is a real risk with /steer: if invocation declines, it's ambiguous whether the feature is unnecessary or annoying.
hooks, enforcement | Start with low-risk extensions (payload control, context freshness) before new patterns.
hooks, agent-behavior | For output verification, inject adversarial requirements into agent prompts via PreToolUse rather than adding post-hoc review agents -- zero-cost and deterministic.
infrastructure-values, enforcement | Build a value registry (value-registry.yaml) of known-risky infrastructure values and DENY + inject corrections on mismatch.
measurement, hooks | Measure with concrete signals: agreement-bias reversals, verifiable-vs-assertion ratio (>70%), token consumption, user invocation frequency.
context-window, hooks | For context freshness, add staleness detection and session-scoped summary cache (after 3rd read, DENY + inject cached summary).
prompt-design, agreement-bias | The /steer skill should extract: core technical goal, relevant constraints, what a verifiable answer looks like, and flags for emotional valence / leading framing / scope ambiguity.
prompt-design, agreement-bias | /steer should reframe leading questions as neutral evaluations, not censor or remove informative frustration.
measurement, hooks | Open problem: measuring whether bidirectional (input + output) outperforms either alone requires synthetic evaluation.

## Source: advanced-claude-configuration
context-window, token-efficiency, context-boot | 33,900 tokens (17% of 200K) consumed at boot by auto-loaded rules/CLAUDE.md before any user interaction; target is <8,000 tokens via trigger tables (proven 54% reduction).
re-read-waste, context-window, measurement | Rules files re-read 22-31x per session (e.g., MEMORY.md: 31x); source files up to 90x (export.py: 90x, cli.py: 80x).
re-read-waste, model-limitations | Root cause of re-read waste: Claude lacks session-scoped memory of prior reads; every reasoning turn starts fresh.
enforcement, hooks, infrastructure-values | Only 4 mechanically enforceable control surfaces exist: system prompt (probabilistic), PreToolUse hooks (deterministic), PostToolUse hooks (observational), agent isolation.
hooks, model-limitations | Text generation cannot be hooked -- no interception during reasoning/analysis.
hooks, model-limitations | PostToolUse cannot modify tool results (Anthropic status: NOT_PLANNED).
hooks, model-limitations | Multi-hook on same matcher silently drops earlier updatedInput (bug #15897).
compaction, context-window | Auto-compaction triggers at 83.5% (configurable via CLAUDE_AUTOCOMPACT_PCT_OVERRIDE); the widely cited ~50% figure is incorrect/outdated.
session-management, context-window, latency | Three targeted 40K-token sessions outperform one saturated 180K session: 2.1s vs 8.2s response time, 94% vs 72% relevance.
compaction, token-efficiency | Structured prompts (tables, YAML, bullets) preserve 92% fidelity during compaction vs 71% for narrative prose.
context-window, measurement | 75% context utilization is the quality sweet spot; performance degrades beyond this.
hooks, tool-routing, token-efficiency | updatedInput rewriting is strictly better than blocking reads: same hook count, 50-90% less context consumed.
hooks, tool-routing | Graph-informed scoping determines relevance for updatedInput rewriting rather than hardcoded thresholds.
tool-routing, agent-behavior | Never narrow first reads of a file -- arXiv 2505.13353 confirms success rates halve when partial reads degrade full logic flow understanding.
tool-routing | Narrowing reads is safe only for re-reads of unchanged files or isolated edits.
subagents, model-limitations | Haiku hallucinates filenames and misses bugs that Sonnet catches; Opus finds subtle bugs (resource leaks, concurrency) that both miss.
subagents, skill-routing | Use Haiku only for read/summarize (Explore-style); use Sonnet/inherit for design decisions.
subagents, context-window | Subagent context is completely isolated -- main context session cache doesn't know about subagent reads (by design, prevents context pollution).
subagents, re-read-waste | A file must be read twice if both main agent and subagent contexts need it.
compaction, token-efficiency | Masking beats summarization for compaction -- cheaper, no trajectory elongation, +2.6% solve rate (JetBrains research).
token-efficiency, agent-behavior | Every competitor (Aider, Cursor, Cody) returns snippets, never full files.
tool-routing, infrastructure-values | Graph + vector hybrid wins for code retrieval, but BM25 is surprisingly competitive -- Sourcegraph abandoned embeddings for adapted BM25.
hooks, enforcement | Hooks cannot validate agent final output -- only tool calls within the agent.
enforcement, prompt-design | Output contracts for agents must be enforced in agent system prompts (probabilistic, but only available lever).
tool-routing, model-limitations | Filesystem offloading (read summary, then Edit) is dangerous: Claude won't have exact text for old_string matching, causing coherence failures.
infrastructure-values, tool-routing | Meta-MCP not worth it for heterogeneous servers -- loses domain-specific ranking and query expressiveness.
subagents, infrastructure-values | Agent Teams experimental, not production-ready -- session resumption broken, task coordination lag; subagents are the reliable path.
hooks, enforcement | Consolidate to one hook per tool matcher (multi-hook bug requires it).
hooks, re-read-waste | Build smart-context.sh for Read: log first reads, warn on 2nd, narrow/deny on 3rd+ re-reads of unchanged files.
context-boot, token-efficiency, prompt-design | Replace verbose rules files with trigger tables to cut boot context by 54%+.
context-boot, subagents | Move detailed architecture content into agent system prompts and skill references (out of boot context).
subagents, decomposition | Cap 3-4 domain expert agents max -- more creates decision overhead.
subagents, enforcement, prompt-design | Each agent should enforce an output contract (structured summary with file:line refs, max 500 words) in its system prompt.
token-efficiency, tool-routing | Add payload caps to all search tools: head_limit: 20 on Grep, head_limit: 50 on Glob, | head -100 on unbounded Bash output.
token-efficiency, agent-behavior | Payload caps prevent context floods with no behavioral change from Claude.
compaction, context-window | Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75 to trigger compaction earlier, preserving more working space post-compaction.
subagents, context-window, skill-routing | Use context: fork on exploration skills to prevent their reads from consuming main context.
compaction, re-read-waste, hooks | Write PreCompact hook that serializes session read cache to a recovery file so post-compaction context includes "you already read these files."
hooks, latency | Use Node.js (not Python) for latency-sensitive hooks -- Node.js warm startup is ~20-50ms vs Python's ~100-200ms.
observability, measurement, hooks | Instrument before optimizing (Phase 0): add read logging to PreToolUse, track path/count/mtime/session position.
token-efficiency, model-limitations | Keep Context Mode MCP and aggressive output compression away from code editing -- they lose exact error messages, line numbers, variable names.
token-efficiency | Context Mode MCP and aggressive output compression are only suitable for research-only sessions.

## Source: tool-routing-optimization
bash-duplication, measurement | ~40% of Bash calls duplicate structured tools: 612 ls, 349 grep, 130 cat out of 3,995 total Bash calls, stable at 37-40% across time and resistant to instructions alone.
enrichment, tool-routing | Structured tool calls get full hook enrichment (PageRank weighting 0.95, graph context, re-read blocking, session caching, payload caps); Bash equivalents bypass all enrichment and are untracked.
token-efficiency, tool-routing | Edit sends diffs (token-efficient) while sed via Bash sends full file output.
observability, infrastructure-values | No external observability tool natively imports SQLite or JSONL; all require a backfill script.
observability, infrastructure-values | Arize Phoenix is the only evaluated observability tool that runs on OSC (pip install, single process, SQLite backend, no Docker).
infrastructure-values, cost | Langfuse requires 4 containers and ~32GB RAM, making it impractical on HPC.
infrastructure-values | OSC has both Podman and Apptainer pre-installed, but multi-container stacks on compute nodes are operationally heavy.
enrichment, hooks | Hook enrichment is the differentiator, not just tool structure; the three-layer model (Raw Bash -> Structured Tool -> Tool + Hook) shows value is in the enrichment pipeline.
enrichment, hooks | Phase 3 of the experiment isolates the effect of hooks by comparing structured tools without hooks vs. with hooks.
nudging, agreement-bias | Nudge hypersensitivity is a real risk: MIT ICLR 2026 paper shows LLMs respond far more strongly to nudges than humans.
nudging, token-efficiency | Each additionalContext injection is 20-50 tokens; if it prevents one re-read (~2,000 tokens) the ROI is 40-100x, but ignored nudges are pure waste.
nudging, correction-resistance | Over-nudging creates brittleness in agent behavior.
enrichment, tool-routing | No existing system combines per-tool-call graph enrichment + session caching + re-read tracking; this is a genuine gap.
enrichment | Aider does PageRank but pre-prompt (not per-call); VS Code Copilot Hooks are architecturally identical but lack graph DB backing.
tool-routing, measurement | OTC (RL-based tool optimization) achieved 68% fewer tool calls and 215% better tool productivity.
measurement, tool-routing | Correlation vs. causation problem: sessions with higher structured-tool usage may just be simpler tasks.
measurement | Proxy controls (files touched, prompt length) are imperfect; the A/B design is the only way to get causal evidence.
measurement, plan-compliance | Start with Phase 1 (observational SQL analysis) before building anything.
hooks, latency | Do NOT add Bash-to-Read nudge hooks until Phase 1 results are in; a pattern-check hook costs 50-100ms per call with no proven benefit.
measurement, tool-routing | For the A/B experiment, use disallowedTools: [Read, Grep, Glob] on one agent to force Bash-only.
infrastructure-values, observability | Prefer cloud free tiers over self-hosting multi-container stacks on HPC.
measurement, token-efficiency | Use TEFS and tool productivity as primary metrics; they penalize correct-but-inefficient patterns.
observability, infrastructure-values | OTel instrumentation is the portable path forward: instrument hooks once with otel-cli, redirect spans to any backend.

## Source: enrichment-ab-testing
enrichment, measurement | 826 enrichment injections fired over 10 days with zero measurement of whether any changed agent decisions.
enrichment, agent-behavior | PageRank weight injection may be counterproductive -- it reinforces existing read patterns instead of informing structural decisions.
enrichment, agent-behavior | The memory nudge from PageRank encourages memorizing current state rather than questioning it.
enrichment, measurement | Outcome quality degraded after adding enrichment infrastructure: W09 80% good to W11 65% good, though confounders prevent attribution.
enrichment, agent-behavior | External evidence confirms context injections affect LLM behavior, but direction of effect is unknown for this system.
measurement | Detectable effect size is limited to >15% differences given ~8 sessions/day from a single user.
measurement | METR needed 246 issues across 16 developers to detect a 19% effect.
measurement, agreement-bias | Self-assessed outcomes (facets) may be unreliable -- METR found developers misjudged AI helpfulness by ~40%.
enrichment, re-read-waste | Read-count blocking is the only enrichment component with demonstrated value.
enrichment, measurement | All four experimental configs keep read-count blocking enabled.
enrichment, decomposition | The experiment decomposes enrichment into four independently testable components: PageRank injection, GitNexus-on-Read, GitNexus-on-Search/Edit, and read-count blocking.
token-efficiency, enrichment | Payload caps (head_limit) should stay on regardless of enrichment config.
enrichment, context-window | The "enrichment might hurt" hypothesis is plausible: bloated tool context is a known failure mode.
enrichment, agent-behavior | Reinforcing existing patterns through enrichment could suppress exploration of unfamiliar but important files.
enrichment, measurement | Run the 4-config A/B test using CLAUDE_ENRICHMENT_CONFIG=A|B|C|D.
observability, enrichment | Log the active config in session-end metadata so it flows into usage.db.
measurement, enrichment | Need ~50 sessions per config (~4 weeks) to reach minimum statistical power.
measurement, enrichment | Measure good_rate, wrong_approach friction count, and avg_tokens per config.
measurement | Accept this will only catch large effects -- small improvements/degradations invisible at this sample size.
measurement | Task difficulty is the dominant confounder and cannot be fully controlled.
enrichment, infrastructure-values | Default to "blocking only" (Config B) if no config shows significant advantage.

## Source: plan-compliance-hooks
plan-compliance, agent-behavior, correction-resistance | LLM agents systematically ignore implementation plans when they hit errors, defaulting to improvised custom code instead of the plan's prescribed replacement.
plan-compliance, agent-behavior | The improvisation chain follows a repeatable pattern: read plan -> start editing -> hit error -> write custom code that compiles -> user pushes back -> repeat.
agent-behavior, model-limitations | Root cause of plan deviation: the agent cannot distinguish guessing from knowing -- prioritizes "code that compiles" over plan compliance.
context-window, plan-compliance | The plan is read once at session start and never re-consulted when difficulty arises -- no mechanism triggers re-reading.
hooks, enforcement | Moment-of-edit intervention is key: plan text must be surfaced at the exact moment the edit is proposed, not earlier.
hooks, subagents | Impartiality requires context separation: a reviewer agent in a fresh context avoids compilation pressure and momentum bias.
hooks, subagents, model-limitations | Same model family used for review may still share blind spots with the main agent.
hooks, measurement | Review burden scales with plan scope not codebase size (~10-15 edits per phase), making human review feasible.
hooks, infrastructure-values, cost | Ollama local models offer a sweet spot (~1-3s, $0, different model family for true impartiality) but are environment-dependent.
hooks, bash-duplication | The hook creates a Bash escape hatch problem: the agent can bypass Edit/Write hooks by using sed or echo > via Bash.
hooks, plan-compliance | Legitimate deviation creates a paradox: if the agent discovers the plan is wrong, the hook blocks editing the plan file itself.
hooks, enforcement | Start with human-in-the-loop command hook (Approach 1): zero cost, ~100ms latency, works everywhere.
hooks, enforcement | Graduate to local LLM review (Approach 2b) on WSL with Ollama to automate routine compliance checks.
hooks, subagents, latency | Reserve the agent-based hook (Approach 2) for high-risk phases only where 10-30s latency per edit is justified.
hooks, bash-duplication, enforcement | Extend the hook's matcher to cover Bash to prevent agents from bypassing Edit/Write via shell commands.
hooks, prompt-design | Parse plans by section headers rather than line-based grep for better reviewer context.
hooks, session-management, compaction | Log approvals across sessions so the same edit isn't re-reviewed after a context compact.
hooks, enforcement | Whitelist the plan file itself from compliance checks to allow legitimate plan updates mid-implementation.

## Source: claude-code-tool-utilization-reliability
bash-duplication | ~40% of Bash calls duplicate structured tools (Read, Grep, Glob, Edit) -- Claude defaults to raw Bash for file operations despite purpose-built alternatives.
context-window, tool-routing | Context window saturation degrades tool awareness as context fills up.
context-window, skill-routing | Skill description budget is 2% of context (16K char fallback); bloated CLAUDE.md files cause Claude to ignore instructions.
tool-routing, prompt-design | Tool descriptions are the primary selection signal for which tool Claude chooses to invoke.
tool-routing, measurement | Tool Search Tool improved Opus 4 accuracy from 49% to 74%.
tool-routing, measurement | Tool Use Examples improved parameter handling accuracy from 72% to 90%.
prompt-design, enforcement | CLAUDE.md routing tables ("When you need X, use Y") are advisory, not deterministic -- followed ~80% of the time.
prompt-design, context-window | Recommended CLAUDE.md length for reliable compliance is ~12 lines.
tool-routing, model-limitations | Lower effort levels cause Claude to skip MCP tools in favor of faster built-in alternatives.
tool-routing | alwaysThinkingEnabled: true helps counteract MCP-skip at low effort but is insufficient alone.
skill-routing | Skill auto-invocation depends on fuzzy name+description matching.
skill-routing | Overlapping skill descriptions cause wrong-skill selection.
skill-routing | Missing user-vocabulary keywords in skill descriptions prevent triggering entirely.
nudging, agent-behavior | LLMs are hypersensitive to nudges: weak cues have disproportionately large effects (ICLR 2026 paper).
hooks, nudging | Hook-injected additionalContext is a direct implementation of the nudge effect.
nudging, token-efficiency | Over-nudging creates brittleness and token waste (~20-50 tokens per injection).
enforcement, hooks | Enforcement hierarchy: hooks are deterministic and mechanical; CLAUDE.md is advisory and degrades with context length.
enforcement, prompt-design | Move enforcement to hooks, knowledge to on-demand skills, keep CLAUDE.md to routing-only.
skill-routing, prompt-design | Skill descriptions should be written for user vocabulary, not developer vocabulary.
skill-routing, prompt-design | Skill descriptions should include trigger phrases users actually say and be mutually exclusive.
token-efficiency, tool-routing | Tool deferred loading (Tool Search Tool pattern) cut token usage 85% while improving accuracy.
token-efficiency | Deferred tool loading is already partially implemented via available-deferred-tools.
session-management, tool-routing | Cross-session tool-routing memory doesn't persist by default; Claude re-learns tool preferences every session.
session-management | Tool routing preferences persist across sessions only if stored in memory MCP or MEMORY.md.
measurement, observability | Each pattern changes context dynamics -- implementing multiple simultaneously makes effects unattributable.
measurement | Observe 5-10 sessions between changes to isolate effects.
prompt-design, context-window | Prune CLAUDE.md aggressively: root CLAUDE.md under 30 lines, project CLAUDE.md under 50 lines.
prompt-design | For each CLAUDE.md line ask "Would removing this cause mistakes?" -- if no, cut it.
prompt-design | Move domain knowledge to skills; move enforcement to hooks.
skill-routing, prompt-design | Rewrite all skill descriptions with natural-language trigger phrases and make them mutually exclusive.
skill-routing | Test skill descriptions with "What skills are available?" to verify discoverability.
skill-routing, enforcement | Side-effect skills (deploy, commit) must set disable-model-invocation: true.
hooks, nudging | Implement a UserPromptSubmit nudge hook that pattern-matches prompts and injects tool suggestions.
hooks, nudging | Start nudge hook with 3-5 high-confidence patterns: code structure -> gitnexus, docs -> lab-docs/context7, memory -> memory MCP.
tool-routing, prompt-design | Move the routing table from CLAUDE.md to a tool-guide skill with user-invocable: false.
skill-routing, enforcement | Set effort: high on skills that depend on MCP tools to counteract the effort-level MCP skip pattern.
decomposition, measurement | Implement changes sequentially, not simultaneously; priority order: prune CLAUDE.md -> audit skill descriptions -> hook nudging -> routing skill.
tool-routing | Verify MCP tool names are descriptive enough for ToolSearch matching.

## Source: claude-task-dag-delegation
decomposition, nudging, agent-behavior | Without explicit nudging, Claude executes complex multi-step prompts monolithically, causing context pollution, zero parallelism, no failure isolation, and poor resumability.
nudging, correction-resistance, agent-behavior | No single mechanism reliably changes Claude's behavior across all prompt complexities; CLAUDE.md nudges suffer the "nudge paradox."
subagents, model-limitations, infrastructure-values | Subagents cannot spawn other subagents; an orchestrator can only use Task/Agent tool for general-purpose workers.
subagents, model-limitations, agent-behavior | Worker specialization must be inlined into the task prompt because subagent nesting is blocked.
subagents, cost, context-window | Agent teams (experimental) provide true parallelism with separate 1M-token context windows but at 2-3x token cost.
subagents, agent-behavior | Agent teams are overkill for most day-to-day prompts.
measurement, decomposition | Complexity thresholds ("3+ steps", "6+ steps") are arbitrary and need empirical calibration.
measurement, hooks | A PostToolUse hook could instrument prompt complexity to calibrate decomposition thresholds.
decomposition, prompt-design | The tiered approach matches how human PMs think: small tasks just do, medium tasks plan first, big tasks delegate.
decomposition, infrastructure-values | Three mechanisms map to the tiers: CLAUDE.md nudge, /plan-and-execute skill, orchestrator subagent.
nudging, correction-resistance | The nudge paradox creates a graceful-degradation argument for the hybrid: the nudge may be ignored, but skill and subagent remain as explicit escalation paths.
session-management, compaction | External state (tasks.json) is the key differentiator over monolithic execution; it survives compaction, enables resumption.
session-management, agent-behavior | JSON is harder for Claude to update reliably than markdown; format choice for external state needs testing.
nudging, decomposition | Auto-detection vs. explicit invocation is an unsolved tension: auto-detection risks the nudge paradox, explicit invocation adds friction.
nudging, prompt-design | The hybrid "nudge suggests, user confirms" may be the sweet spot for triggering decomposition.
cost, subagents, token-efficiency | Cost-conscious mode needed for constrained environments (OSC): decompose for clarity but execute sequentially to avoid multiplied token usage.
decomposition, infrastructure-values | Build incrementally, cheapest tier first: Week 1 = CLAUDE.md nudge + /plan-and-execute, Week 2 = orchestrator subagent, Week 3 = evaluate.
subagents, agent-behavior | Reserve agent teams for genuinely large parallel workloads; invoke manually, never as a default.
nudging, prompt-design, agent-behavior | The CLAUDE.md nudge must include a "do NOT decompose" guardrail for simple single-purpose prompts.
subagents, prompt-design | Orchestrator workers need specialization inlined into task prompts since subagent nesting is blocked.
measurement, hooks, observability | Instrument before tuning: add hooks or logging to measure prompt complexity vs. whether decomposition was actually used.
infrastructure-values, prompt-design | For cross-project deployment via chezmoi, template the CLAUDE.md nudge so it only activates on machines where Claude Code is installed.
