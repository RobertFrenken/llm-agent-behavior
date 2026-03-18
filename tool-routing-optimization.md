# Tool Routing Optimization: Structured Tools + Hooks vs Raw Bash

> Researched: 2026-03-08

## Motivation

Claude Code has two ways to interact with files: **built-in structured tools** (Read, Grep, Glob, Edit, Write) and **Bash** (cat, grep, find, sed). Anthropic's system prompt instructs Claude to prefer the structured tools, but ~40% of Bash calls are still file operations that duplicate what the structured tools do (612 `ls` calls, 349 `grep` calls, 130 `cat` calls out of 3,995 total Bash calls).

The question isn't whether this is messy — it's whether it **matters**.

### Why it might matter

We built a PreToolUse hook infrastructure (Phases 0-5) that enriches structured tool calls with graph context, PageRank weights, re-read tracking, payload caps, and session-scoped caching. **These hooks are tool-specific.** When Claude uses `Read`, it gets the full smart-context pipeline. When it `cat`s the same file via Bash, none of that fires:

| Action | PageRank | Graph Context | Re-read Tracking | Session Cache | Payload Cap |
|--------|----------|---------------|------------------|---------------|-------------|
| `Read file.py` | Yes (0.95) | Yes (callers/callees) | Yes (blocks 3rd+) | Yes (survives compaction) | N/A |
| `bash cat file.py` | No | No | No | No | `\| head -100` only |
| `Grep "pattern"` | N/A | Yes | N/A | N/A | `head_limit: 30` |
| `bash grep "pattern"` | N/A | Partial (only if pattern detected) | N/A | N/A | `\| head -100` (less precise) |
| `Edit old → new` | N/A | N/A | N/A | N/A | Sends diff only (token efficient) |
| `bash sed 's/old/new/'` | N/A | N/A | N/A | N/A | Sends full file output |

So every `cat` call is an **untracked, unenriched read** that bypasses the entire infrastructure. The session cache doesn't know about it, the pre-compact hook won't list it, and the 3rd-read blocker can't prevent it.

### Why it might not matter

- The 40% misuse rate is **stable** across time periods (39.7% → 40.1% → 37.1%). Instructions alone don't fix it.
- Adding nudge hooks adds latency to every Bash call (~50-100ms for the pattern check).
- Many `ls` and `find` calls are legitimately better in Bash (permissions, sizes, piping).
- If the hook enrichment doesn't actually improve Claude's downstream reasoning, the whole argument collapses — we'd be adding complexity for no measurable gain.

### The core question

Is there a measurable difference in session quality when Claude uses **structured tool + hook** vs **raw Bash** for file operations? If yes, nudging is worth the overhead. If no, the hooks are valuable but the routing doesn't matter.

## Conceptual Framework

### The Three Layers

```
Layer 0: Raw Bash          cat file.py → raw text enters context
Layer 1: Structured Tool   Read file.py → same text, but structured metadata
Layer 2: Tool + Hook       Read file.py → text + PageRank + graph + tracking
```

Each layer adds information. The experiment measures whether that information translates to better outcomes.

### Hook Mechanics (for reference)

A PreToolUse hook is a shell script that receives the tool call as JSON on stdin and can output:

```
1. Nothing        → tool proceeds normally
2. additionalContext → text injected alongside result (nudge/enrich)
3. updatedInput   → silently rewrite tool parameters (caps, scoping)
4. deny           → block the call with a reason message
```

A "nudge" and a "hook enrichment" are the same mechanism — just different JSON output from the same PreToolUse hook. There is no separate nudge system.

PostToolUse hooks fire after execution but **cannot modify results** (Anthropic: NOT_PLANNED). They can only observe and log.

## Experiment Design

### Hypothesis

**H1**: Sessions where Claude uses structured tools (Read/Grep/Glob) have fewer total tool calls, lower token consumption, and fewer re-reads than sessions where Claude uses Bash equivalents (cat/grep/find).

**H2**: Hook enrichment (PageRank, graph context) on structured tool calls reduces follow-up reads and searches compared to unenriched structured tool calls.

### Methodology

#### Phase 1: Observational Analysis (no code changes)

Query existing `usage.db` data to correlate tool choice patterns with session outcomes.

```sql
-- Classify sessions by structured-tool usage ratio
SELECT
    session_id,
    -- Ratio of structured reads vs bash reads
    COUNT(CASE WHEN tool = 'Read' THEN 1 END) as structured_reads,
    COUNT(CASE WHEN tool = 'Bash' AND command LIKE 'cat %' THEN 1 END) as bash_reads,
    -- Outcomes
    SUM(total across all tools) as total_tool_calls,
    session.input_tokens + session.output_tokens as total_tokens,
    session.duration_min,
    session.tool_errors
FROM tool_calls
GROUP BY session_id
```

Dimensions to measure per session:
- **Total tool calls** (fewer = more efficient)
- **Total tokens** (lower = more context-efficient)
- **Re-read ratio** (count of 2nd+ reads of same file / total reads)
- **Tool errors** (fewer = smoother)
- **Duration** (shorter for same task complexity = better)

Confound: sessions with more structured tool usage may also be sessions where the user asked simpler questions. Need to control for task complexity (proxy: number of files touched, first_prompt length).

#### Phase 2: A/B Experiment (code changes required)

Use the `/experiment` skill pattern — fork the same prompt into two parallel agents:

- **Agent A**: Normal tools + hooks (current setup)
- **Agent B**: `disallowedTools: [Read, Grep, Glob]` — forced to use Bash for all file operations

Give both agents the same task (e.g., "find all functions that call `export_results` and explain the data flow"). Compare:

| Metric | How to measure |
|--------|---------------|
| Tool calls | Count from agent transcript |
| Tokens consumed | `input_tokens + output_tokens` from agent result |
| Unique files accessed | Distinct file paths from tool calls |
| Re-reads | Files read more than once |
| Answer quality | Manual review (blinded) |
| Time to complete | Wall clock from agent start to finish |

Run 5-10 task pairs across different task types:
- Code search ("where is X defined?")
- Architecture understanding ("how does the pipeline flow?")
- Bug investigation ("why does this fail?")
- Refactoring ("rename X to Y across the codebase")

#### Phase 3: Hook Value Isolation

If Phase 2 shows structured tools win, isolate whether the **hook enrichment** matters:

- **Agent C**: Structured tools, hooks disabled (no smart-context, no gitnexus enrichment)
- **Agent A**: Structured tools + hooks (current setup)

Compare C vs A on the same metrics. This tells us whether it's the structure or the enrichment that drives the difference.

### Control Variables

- Same model (`model: inherit` for both agents)
- Same `maxTurns` limit
- Same task prompt (verbatim)
- Same working directory and codebase state
- Run both agents in the same session to minimize environmental variance

## Source Files (read during implementation)

| File | Why |
|------|-----|
| `~/.cache/claude-usage/usage.db` | Historical tool call data for observational analysis |
| `~/.claude/scripts/trace_analysis.py` | Extend with new analysis commands |
| `~/.claude/hooks/gitnexus/gitnexus-hook.cjs` | Where Bash nudge would be added |
| `~/.claude/hooks/smart-context.sh` | The Read hook whose value we're measuring |
| `~/.claude/skills/experiment/SKILL.md` | A/B fork pattern to adapt |

## Implementation Sketch

### Phase 1 (observational, ~1 hour)

1. Add `tool-routing` command to `trace_analysis.py` that runs the session-level correlation query
2. Output: table of sessions ranked by structured-tool ratio, with outcome metrics
3. Look for correlation (not causation) between tool choice and session quality

### Phase 2 (A/B experiment, ~2 hours)

1. Write two agent definitions:
   - `agents/tool-ab-structured.md` — normal tools + hooks
   - `agents/tool-ab-bash.md` — `disallowedTools: [Read, Grep, Glob]`
2. Design 5 task prompts spanning different task types
3. Run pairs, collect transcripts
4. Build comparison table

### Phase 3 (hook isolation, ~1 hour)

1. Temporarily disable smart-context and gitnexus hooks
2. Re-run the same 5 tasks with structured tools but no enrichment
3. Compare against Phase 2 results

## Open Questions

- **Is 5-10 task pairs enough for statistical significance?** Probably not for rigorous claims, but enough to see directional signal.
- **How to control for task complexity?** First_prompt length and files-touched count are imperfect proxies. May need manual task difficulty rating.
- **Does the nudge itself consume enough tokens to offset savings?** Each `additionalContext` injection is ~20-50 tokens. If it prevents one unnecessary re-read (~2,000 tokens), the ROI is 40-100x. But if the nudge is ignored, it's pure waste.
- **Can we measure "answer quality" without manual review?** Possibly: check if the agent's final answer references the correct files (from a ground-truth set). But this requires known-answer tasks.
- **Should we also test Edit vs sed?** Edit sends diffs (small); sed sends full output (large). This is likely the clearest win for structured tools and would strengthen the case.

## Recommendation

Start with **Phase 1** (observational). It requires no code changes beyond a new query in `trace_analysis.py` and will tell us whether there's even a correlation worth investigating. If the data shows no relationship between tool choice and session outcomes, we can stop there — the hooks are valuable for their own reasons (tracking, capping), and the nudge isn't worth adding.

If Phase 1 shows signal, proceed to Phase 2 for causal evidence.

Do not add Bash→Read nudges until Phase 1 results are in.

## Literature Review (2026-03-08)

### Directly Relevant Papers

**1. "LLM Agents Are Hypersensitive to Nudges"** (Cherep, Maes, Singh — MIT Media Lab, ICLR 2026)
- LLMs are far more responsive to nudges than humans — weak cues that slightly shift human behavior have disproportionately large effects on model choices.
- Directly supports hook-injected `additionalContext` pattern. Caution: over-nudging creates brittleness.
- arXiv: 2505.11584 | GitHub: `PapayaResearch/nudging`

**2. OTC: Optimal Tool Calls via RL** (Wang et al., April 2025)
- RL framework optimizing tool-call efficiency: **68.3% reduction in tool calls**, **215.4% improvement in tool productivity** (correct answers / total calls).
- Introduces "tool-integrated reward" jointly considering correctness and efficiency.
- arXiv: 2504.14870

**3. "Codified Context: Infrastructure for AI Agents in a Complex Codebase"** (Feb 2026)
- Three-tier context (hot/specialized/cold), trigger tables routing to domain agents. Tracked 283 sessions, 16,522 agent turns. Persistent specs prevented repeated debugging cycles.
- **Studies essentially the same pattern we've built.**
- arXiv: 2602.20478

**4. Tool RAG** (Red Hat, Kolchinsky, Nov 2025)
- RAG applied to tool descriptions — semantically searches tool schemas/usage patterns. Claims **3x tool invocation accuracy, 50% prompt reduction**.
- next.redhat.com/2025/11/26/tool-rag-the-next-breakthrough-in-scalable-ai-agents/

**5. START: Self-taught Reasoner with Tools** (March 2025, EMNLP 2025)
- "Hint-infer" — injects artificial hints at inference time to activate latent tool-use capabilities. Conceptually identical to `additionalContext` hook injection.
- arXiv: 2503.04625

**6. ToolRL: Reward is All Tool Learning Needs** (Qian et al., 2025)
- Reward-based RL for learning when/how to use tools.
- openreview.net/forum?id=eOLdGbXT6t

**7. LLM-Based Agents for Tool Learning: A Survey** (Springer, 2025)
- Organizes tool learning into four stages: task planning, tool selection, task execution, response generation. Hook enrichment targets the tool selection stage.
- link.springer.com/article/10.1007/s41019-025-00296-9

### Context Engineering as a Field

**Context Engineering for Agents** (LangChain blog, June 2025)
- Defines context engineering as "filling the context window with just the right information at each step of an agent's trajectory."
- Four strategies: write (persist), select (pull relevant), compress (summarize), isolate (split across sub-agents).
- Cognition (Devin) calls it "the #1 job of engineers building AI agents."
- blog.langchain.com/context-engineering-for-agents/

**Agentic RAG Survey** (Jan 2025, arXiv: 2501.09136)
- Agents dynamically manage retrieval strategies based on reasoning state. Hook pattern is this applied at infrastructure level rather than requiring agent decision.

### Existing Implementations

| System | Pattern | vs Our Approach |
|--------|---------|-----------------|
| **LangChain Middleware** (v1-Alpha) | `before_model`, `wrap_tool_call`, `after_model` — 13 built-in | Same primitives, no graph DB backing |
| **VS Code Copilot Hooks** | 8 lifecycle events, `updatedInput` + `additionalContext` | Architecturally identical to Claude Code hooks |
| **Aider repo map** | tree-sitter AST → dependency graph → PageRank ranking | Similar to GitNexus enrichment, but pre-prompt not per-call |
| **Devin** | Auto-indexed codebase wikis, multi-model swarm | Context enrichment but coarser-grained |
| **Cursor** | `.cursor/rules/*.mdc` injected at context start | Persistent injection, not per-tool-call |

**Unique differentiator**: No existing system combines per-tool-call graph-context enrichment via pre-execution hooks with session caching and re-read tracking.

### Benchmarks & Metrics (for experiment design)

| Benchmark | Key Metric | Relevance |
|-----------|-----------|-----------|
| **MCPAgentBench** (arXiv: 2512.24565) | Task Efficiency Finish Score (TEFS) — penalizes correct-but-inefficient | Directly applicable to our A/B |
| **ToolBench** (ICLR 2024) | Pass Rate + Win Rate, per-call annotation | Tool choice evaluation framework |
| **SWE-bench** | Token budget constraints, tool profiles | Strategic approach comparison |
| **API-Bank** (EMNLP 2023) | API call detection, retrieval, planning | Multi-tool planning evaluation |

Recommended metrics (literature-aligned):
1. Tool productivity (correct outcomes / total calls) — OTC
2. TEFS (correctness × efficiency) — MCPAgentBench
3. Token usage (input + output)
4. Re-read ratio (unique to our system)
5. Wall-clock time
6. Error/retry rate

### Key Tensions from Literature

1. **Enrich vs. reduce**: OTC shows best optimization is sometimes fewer calls, not enriched ones. Phase 3 (hook isolation) addresses this.
2. **Nudge hypersensitivity**: ICLR 2026 paper — agents may over-rely on injected context. Monitor for this.
3. **Context pressure**: Every enrichment adds tokens, degrading performance at long trajectories. Payload caps and compaction mitigate.

## Frameworks & Tools Landscape (2026-03-08)

### Hook Policy / Composition

**Invariant Guardrails** (invariantlabs.ai, acquired by Snyk)
- Purpose-built DSL for agent tool-call policies. Pattern-matching on tool calls and data flows.
- Example: `raise "msg" if: (call: ToolCall) call is tool:Read count(max=2) ...`
- Invariant Gateway acts as runtime proxy intercepting calls.
- **Fit**: Our deny/allow logic (re-read blocking, payload caps) could be expressed declaratively. Evaluate vs shell scripts.
- github.com/invariantlabs-ai/invariant

**Microsoft Agent Framework Middleware** (github.com/microsoft/agent-framework)
- Chain-of-responsibility pattern for tool calls. Three middleware layers: agent, chat client, function/tool.
- **Fit**: Design patterns for hook composition/ordering. "Build your own agent" only — patterns are portable.

### Context Injection

**No dedicated framework exists for pre-execution context enrichment.** This is the gap our hooks fill. Closest concepts:
- **Parlant** (parlant.io) — context-narrowing per turn, but conversation-level not tool-call-level
- **Aider repo map** — tree-sitter AST → PageRank file ranking, but pre-prompt not per-call
- **Anthropic/Manus context engineering guides** — describe the pattern abstractly; we've built it concretely

### Guardrails (deny/allow)

| Framework | Focus | Agent Model | Maturity |
|-----------|-------|-------------|----------|
| Invariant Guardrails | Tool-call policies (DSL) | Extend existing | High (Snyk-acquired) |
| NeMo Guardrails | Conversation flows (Colang DSL) | Build your own | High (NVIDIA) |
| LlamaFirewall | Security (jailbreak, alignment, code) | Build your own | High (Meta production) |
| APort Agent Guardrails | Pre-action authorization (passport model) | Extend existing | Newer (~40ms latency) |
| Guardrails AI | Output validation | Build your own | High |

### Observability

| Tool | Type | Key Feature for Us | Overhead |
|------|------|-------------------|----------|
| **Langfuse** | Open-source, self-hostable | Tracing + evals, correlate hook injection with outcomes | ~15% |
| **Braintrust** | Hybrid monitoring + QA | Production A/B testing, side-by-side comparison | Low |
| **Arize Phoenix** | Open-source | Drift detection, multi-step agent traces | Low |
| **AgentOps** | Agent-native | 400+ framework integrations, multi-step traces | ~12% |
| **OpenTelemetry GenAI** | Standard/convention | Semantic conventions for agent spans, tool calls | Minimal |

### Claude Code Community

- `disler/claude-code-hooks-mastery` — tutorials and hook patterns
- `ChrisWiles/claude-code-showcase` — hooks, skills, agents examples
- `karanb192/claude-code-hooks` — reusable hook collection
- `carlrannaberg/claudekit` — toolkit of commands/hooks/utilities

### Key Gap

No framework combines **enrichment + gating + tracking** as first-class concerns. Frameworks cover safety/deny OR observability OR orchestration. Our hooks sit in the intersection.

## Observability Deep Dive (2026-03-08)

### Current Internal Tracking

| Component | Data | Format | Location |
|-----------|------|--------|----------|
| PostToolUse hook | Every tool call (name, params, timestamp, session) | JSONL | `tool-usage.jsonl` |
| Usage dashboard | 192 sessions, 13,358 tool calls, views | SQLite | `~/.cache/claude-usage/usage.db` |
| Smart-context hook | Per-session file read log | JSONL | `/tmp/claude-reads-{session_id}.jsonl` |
| Trace analysis | hot-files, task-patterns, pagerank-weights, session-profile | Python script | `~/.claude/scripts/trace_analysis.py` |
| Graph drift | Stale nodes in codebase graphs | Python script | `~/.claude/scripts/graph-drift.py` |
| PreCompact hook | Session read summary injected before compaction | Shell script | `~/.claude/hooks/` |

### Tool Evaluation

#### Arize Phoenix — Best fit for OSC

- **Runs from `pip install arize-phoenix`** — single Python process, SQLite backend by default. No Docker, Redis, ClickHouse, or blob storage. The only tool that can run on OSC login nodes.
- Native OTel backend — receives spans via `http://localhost:6006/v1/traces`
- Custom spans via `arize-phoenix-otel` or any OTel SDK
- Trace visualization, waterfall views, structured analysis
- A/B comparison is basic (less polished than Braintrust/Langfuse)
- Open source (MIT)
- **Integration path**: `pip install arize-phoenix arize-phoenix-otel`, launch server, backfill from JSONL/SQLite, instrument hooks with OTel spans

#### Braintrust — Best A/B comparison

- Purpose-built experiment framework: define task dataset, run under two configs, side-by-side diff comparison with scoring
- Free tier: 1M spans, 10K scores (generous for our experiment)
- **Cloud-only** (self-host = Enterprise plan). Closed source.
- REST API (`/v1/project_logs/{id}/insert`) accepts custom events with input/output/scores/metadata
- BTQL (SQL-like) for analysis
- **Integration path**: backfill script to push existing data, use experiment framework for A/B

#### Langfuse — Best ecosystem

- Tracing + evals + experiments + datasets
- Multi-score comparison (Pearson, Cohen's Kappa, F1) in experiment compare view
- OTLP/HTTP endpoint for OTel-compatible ingestion
- **Cannot self-host on OSC** (requires PostgreSQL + ClickHouse + Redis + S3 + two containers, ~8GB RAM)
- Cloud free tier: 50K observations/month (fits our 13K tool calls)
- Shell hooks can emit via curl to REST API (Basic Auth)
- Custom scores: Numeric, Categorical, Boolean — attach to traces/observations/sessions
- **Integration path**: cloud account, backfill script, add emission to hooks

#### OpenTelemetry GenAI Conventions — Best long-term standard

- Agent-specific conventions (Development status, not stable yet):
  - `invoke_agent`, `create_agent`, `execute_tool` spans
  - `gen_ai.agent.name`, `gen_ai.operation.name`, token usage metrics
- **Shell hook emission** via `otel-cli` (single Go binary) or `opentelemetry-shell` (bash functions + curl)
- Collector available as standalone binary (~50MB, no Docker/root needed)
- Any backend consumes: Phoenix, Langfuse, Jaeger, Grafana Tempo
- **Integration path**: instrument hooks with otel-cli, send to Phoenix or Langfuse

#### AgentOps — Skip

- SDK-only auto-instrumentation, no JSONL/SQLite import, no REST API for custom ingestion
- No A/B comparison capability
- Designed for standard frameworks (CrewAI, LangChain), poor fit for custom hooks

### None of Them Import SQLite or JSONL Directly

All require a backfill script to re-emit existing data. No tool has a native SQLite/JSONL connector.

### Lightweight Alternatives (visualization on existing data)

| Tool | What it does | OSC-compatible | Effort |
|------|-------------|----------------|--------|
| **DuckDB SQL** | `SELECT * FROM read_json_auto('tool-usage.jsonl')` — query JSONL natively | Yes | Minimal |
| **Evidence.dev** | SQL + Markdown → static HTML dashboards. Reads DuckDB/SQLite/CSV/Parquet | Yes (`npx` or `pip`) | ~2 hours |
| **Mosaic/vgplot** | Interactive grammar of graphics via DuckDB-WASM (already used in KD-GAT Quarto) | Yes | ~2 hours |

### Recommended Approach

**Phase 1 experiment: Use what we have (Option A)**

No external tools needed. Our `usage.db` + `tool-usage.jsonl` already contain the data. Steps:
1. Add a `config` tag to sessions (hook-on vs hook-off vs bash-only)
2. Write DuckDB/SQL queries comparing sessions by configuration
3. Build an Evidence.dev dashboard or Quarto page for visualization

```sql
-- Compare tool efficiency by configuration
SELECT config,
       AVG(tool_calls) as avg_calls,
       AVG(tokens) as avg_tokens,
       AVG(re_reads) as avg_rereads,
       AVG(errors) as avg_errors
FROM experiment_sessions
GROUP BY config;
```

**If we want real-time observability going forward (Option B)**

Layer in **Arize Phoenix** (pip install, single process) as OTel collector + trace visualizer. Instrument hooks with **otel-cli** (single binary, no Docker). This gives:
- Waterfall trace views of agent sessions
- Span-level timing of hook enrichment
- Foundation for future A/B comparison

OTel instrumentation is portable — if Phoenix is insufficient, redirect spans to Langfuse Cloud or Braintrust without re-instrumenting.

**If we want the best experiment UX (Option C)**

Use **Braintrust Cloud** free tier for the A/B experiment specifically. Write a backfill script (~100 lines), use their experiment framework for structured comparison with diff views. Keep internal tracking for day-to-day observability.

## OSC Containerization (2026-03-08)

Container support on OSC means tools requiring Docker should NOT be crossed off — they can likely run via Podman or Apptainer.

### Available Runtimes

| Runtime | On OSC? | Docker image compat | Compose support | GPU | Best for |
|---------|---------|--------------------|-----------------|----|----------|
| **Apptainer** | Yes (pre-installed) | Yes (`apptainer pull docker://`) | No | `--nv` flag | Single-container GPU workloads |
| **Podman** | Yes (pre-installed, `docker` aliased) | Yes (native OCI) | Yes (`podman-compose`) | `--device nvidia.com/gpu=all` | Multi-container stacks |
| Charliecloud | No | — | — | — | — |
| Enroot (NVIDIA) | No | — | — | — | — |
| Pixi | N/A | N/A | N/A | N/A | Package manager, not container tool |

### Podman (Docker replacement)

- `docker` command on OSC is aliased to Podman — most Docker commands work as-is
- `podman-compose` pre-installed at `/usr/bin/podman-compose`
- Containers in a compose stack resolve each other by service name
- Fully rootless, no daemon
- **Caveat: images are ephemeral** (node-local `/tmp/container-user-<uid>`). Must `podman save`/`podman load` from shared storage across jobs.
- `podman-compose` is less mature than `docker-compose` — some Compose v2 features (health checks, depends_on conditions, restart policies) may not work

### Apptainer (Singularity)

- Converts Docker images to `.sif` single-file containers
- Shares host network by default (no port mapping needed — simpler than Docker)
- Auto-binds `$HOME`, CWD, `/fs/ess`, `/tmp`
- No compose equivalent — must manually orchestrate multiple containers via `apptainer instance start`

### Revised Tool Feasibility

| Tool | Requirement | OSC Path | Friction |
|------|------------|----------|----------|
| **Phoenix** | `pip install` | Direct — no containers | None |
| **Langfuse** | 4 containers (pg + clickhouse + redis + app) | `podman-compose` on compute node | High — ephemeral images, ~32GB mem, stack dies with SLURM job |
| **Braintrust** | Cloud-only (or Enterprise) | Cloud free tier | None (no containers) |
| **Jaeger** | Single binary or single container | Standalone binary or `apptainer pull` | Low |
| **OTel Collector** | Standalone Go binary | Direct — no containers | None |

### Multi-Container Stack Constraints on HPC

Running a compose stack (e.g., Langfuse) on a compute node is possible but operationally heavy:
1. Must run inside a SLURM job — stack dies when allocation ends
2. Images must be pre-saved to shared storage and reloaded each job
3. Data persistence requires bind-mounting directories from shared filesystem
4. Access via SSH tunnel to compute node ports
5. Burns SLURM allocation on infrastructure, not research

**Recommendation**: For multi-container tools, prefer cloud free tiers (Langfuse Cloud, Braintrust Cloud) over self-hosting on HPC. Reserve containerization for single-image workloads (ML training, Jaeger, standalone tools) where Apptainer excels.
