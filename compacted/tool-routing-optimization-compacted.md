## Core Findings

- **~40% of Bash calls duplicate structured tools**: 612 `ls`, 349 `grep`, 130 `cat` out of 3,995 total Bash calls. This rate is stable across time (37-40%) and resistant to instructions alone.
- **Structured tool calls get full hook enrichment; Bash equivalents get none.** A `Read` call receives PageRank weighting (0.95), graph context (callers/callees), re-read blocking (3rd+ blocked), session caching (survives compaction), and payload caps. A `cat` via Bash bypasses all of this -- it is an untracked, unenriched read.
- **Edit sends diffs (token-efficient); `sed` sends full file output.** This is likely the clearest structured-tool win.
- **No external observability tool natively imports SQLite or JSONL.** All require a backfill script. Existing internal tracking (usage.db, tool-usage.jsonl, trace_analysis.py) already contains the needed data.
- **Arize Phoenix is the only evaluated tool that runs on OSC** (pip install, single process, SQLite backend, no Docker). Langfuse requires 4 containers and ~32GB RAM -- impractical on HPC.
- **OSC has both Podman (aliased as `docker`) and Apptainer pre-installed**, but multi-container stacks on compute nodes are operationally heavy (die with SLURM job, ephemeral images, burns allocation on infra).

## Key Insights

- **Hook enrichment is the differentiator, not just tool structure.** The three-layer model (Raw Bash -> Structured Tool -> Tool + Hook) shows that the value is in the enrichment pipeline, not merely in using a named tool. Phase 3 of the experiment isolates this: structured tools without hooks vs. with hooks.
- **Nudge hypersensitivity is a real risk.** MIT ICLR 2026 paper shows LLMs respond far more strongly to nudges than humans -- weak cues have disproportionately large effects. Over-nudging creates brittleness. Each `additionalContext` injection is 20-50 tokens; if it prevents one re-read (~2,000 tokens) the ROI is 40-100x, but ignored nudges are pure waste.
- **No existing system combines per-tool-call graph enrichment + session caching + re-read tracking.** Aider does PageRank but pre-prompt (not per-call). VS Code Copilot Hooks are architecturally identical but lack graph DB backing. This is a genuine gap in the tooling landscape.
- **OTC (RL-based tool optimization) achieved 68% fewer tool calls and 215% better tool productivity** -- suggesting that fewer, better-routed calls beat more calls, reinforcing the value of nudging toward structured tools rather than tolerating Bash duplication.
- **The correlation vs. causation problem is real.** Sessions with higher structured-tool usage may just be simpler tasks. Proxy controls (files touched, prompt length) are imperfect. The A/B design (same prompt, one agent forced to Bash-only) is the only way to get causal evidence.

## Takeaways

- **Start with Phase 1 (observational SQL analysis) before building anything.** Query existing usage.db to check whether structured-tool ratio even correlates with session quality. If no correlation, stop -- hooks are still valuable for tracking/capping, but nudging isn't worth the overhead.
- **Do NOT add Bash-to-Read nudge hooks until Phase 1 results are in.** Adding a pattern-check hook to every Bash call costs 50-100ms per call with no proven benefit.
- **For the A/B experiment, use `disallowedTools: [Read, Grep, Glob]` on one agent** to force Bash-only, then compare tool calls, tokens, re-reads, errors, and wall-clock time across 5-10 task pairs (search, architecture, debugging, refactoring).
- **Prefer cloud free tiers over self-hosting multi-container stacks on HPC.** Braintrust Cloud (1M free spans) for A/B comparison; Phoenix (pip install) for local trace visualization. Reserve Apptainer for single-image workloads.
- **Use TEFS (Task Efficiency Finish Score) from MCPAgentBench and tool productivity (correct outcomes / total calls) from OTC** as the primary metrics -- they penalize correct-but-inefficient patterns, which is exactly what unrouted Bash calls represent.
- **OTel instrumentation is the portable path forward.** Instrument hooks once with otel-cli, then redirect spans to any backend (Phoenix, Langfuse, Jaeger) without re-instrumenting.
