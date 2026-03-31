# Advanced Claude Code Configuration: Unified Strategy

> **Status**: Enhanced synthesis — merges architecture + runtime + new research (Mar 8)
> **Created**: 2026-03-08
> **Predecessors**: `context-efficient-architecture.md`, `runtime-architecture-observations.md`
> **Goal**: A single actionable reference for maximizing Claude Code effectiveness through mechanical enforcement, not instructional requests

## Executive Summary

Two investigations converged on the same conclusion: Claude Code's context management cannot be solved with markdown instructions. The runtime architecture reveals exactly **four mechanically enforceable control surfaces** — and a clear hierarchy of what each can and cannot do. This document synthesizes empirical runtime findings with architectural design into a unified strategy.

## The Problem (Quantified)

| Metric | Current | Waste Category |
|--------|---------|---------------|
| Pre-loaded context | 33,900 tokens (17% of 200K) | Boot overhead |
| Rules file re-reads | tooling.md: 23×, knowledge-bank.md: 22×, MEMORY.md: 31× | Double-loading (auto-injected + re-read) |
| Source file re-reads | export.py: 90×, cli.py: 80×, init_db.py: 71× | No session memory |
| Bash file inspection | 1,545 calls (cat/ls/head) | Wrong tool selection |
| Agent delegation | 1.1 calls/session, 0 custom agents | Under-delegation |
| GitNexus MCP usage | 1 call total (list_repos only) | Infrastructure waste |
| Empty ToolSearch | 410 calls | Runtime overhead |
| Read distribution | Q1:38.6%, Q2:28%, Q3:33.8%, Q4:32.7% | Sustained, not frontloaded |

**Core insight**: The read pattern is sustained throughout sessions because Claude lacks session-scoped memory of what it has already read. Every new reasoning turn starts fresh, and with 33K tokens of instructions saying nothing about what was already consumed, re-reading is the rational default.

## The Control Surface (What's Actually Possible)

### Empirically Verified Architecture

```
User message
    ↓
Runtime injects: system prompt + tool schemas + CLAUDE.md + rules/*.md + conversation history
    ↓
Model generates tokens (probabilistic — shaped by context, not deterministic routing)
    ↓
Runtime parser monitors output stream
    ├─ Plain text → display to user (NO interception possible)
    ├─ Tool call pattern detected → intercept
    │   ├─ PreToolUse hooks fire (synchronous, blocking)
    │   │   ├─ ALLOW / DENY / ASK
    │   │   ├─ updatedInput (rewrite tool parameters)
    │   │   ├─ additionalContext (inject text model sees)
    │   │   └─ BUG: multi-hook on same matcher → last hook wins
    │   ├─ Tool executes → result captured
    │   ├─ PostToolUse hooks fire (observational only, CANNOT modify results)
    │   ├─ Tool result injected into context
    │   └─ Model generates next response
    └─ Stop sequence → end generation
```

### The Four Control Surfaces

| # | Surface | Mechanism | Enforcement | Limitation |
|---|---------|-----------|-------------|------------|
| **1** | System prompt | CLAUDE.md, rules/*.md | Probabilistic (instruction-following) | Degrades beyond ~150-200 instructions |
| **2** | PreToolUse hooks | Shell scripts at tool boundaries | **Deterministic** — deny, rewrite params, inject context | Cannot invoke tools, cannot fire mid-generation, latency is blocking |
| **3** | PostToolUse hooks | Shell scripts after execution | Observational only — log, inject systemMessage | Cannot modify tool output (rejected by Anthropic) |
| **4** | Agent isolation | .claude/agents/*.md | Structural — separate context window | Model must "choose" to spawn (probabilistic); agent descriptions shape this |

### What Each Surface Is Best For

| Goal | Best Surface | Why |
|------|-------------|-----|
| Block re-reads | PreToolUse hook (#2) | Deterministic — fires every time, can DENY with cached summary |
| Inject graph context | PreToolUse hook (#2) | `additionalContext` appears before tool result |
| Narrow read scope | PreToolUse hook (#2) | `updatedInput` rewrites offset/limit silently |
| Reduce boot context | System prompt (#1) | Trigger tables replace verbose docs |
| Isolate exploration | Agent isolation (#4) | Subagent reads files in its own window, returns summary |
| Log usage patterns | PostToolUse hook (#3) | Observational logging doesn't affect behavior |
| Redirect to agent | PreToolUse hook (#2) | DENY + reason "use pipeline-expert agent instead" |
| Format enforcement | System prompt (#1) only | No hook fires during text generation |

### Hard Boundaries (Cannot Be Changed)

1. **Text generation cannot be hooked** — reasoning, analysis, explanations stream without interception points
2. **PostToolUse cannot modify results** — tool output enters context before hook fires (Anthropic: NOT_PLANNED)
3. **Multi-hook on same matcher drops earlier updatedInput** — last hook wins silently (bug #15897)
4. **Token-level stream interception impossible** at API consumer level
5. **Tool invocation from hooks impossible** — can only suggest via DENY reason

## Unified Architecture

### Layer 1: Thin Boot Context (Trigger Table)

**Current**: 33,900 tokens of rules files auto-loaded
**Target**: ~5,000 tokens via trigger table pattern (54% reduction proven by johnlindquist)

```markdown
## When you need...                → Use
Pipeline architecture/training     → pipeline-expert agent
Map-Viz charts/D3/data flow        → mapviz-expert agent
Config system/YAML resolution      → config-expert agent
Library docs (PyTorch, Ray, PyG)   → context7 MCP
Lab setup procedures               → lab-docs MCP
Code structure/dependencies        → gitnexus MCP
Cross-session learnings            → memory MCP
SLURM job management               → /hpc skill
Codebase navigation                → /navigate skill
```

**What stays in rules files**: trigger table, critical constraints (login node safety), environment facts, session behavior rules.
**What moves out**: architecture details → agent prompts, conventions → agent prompts, workflows → skills, troubleshooting → MCP responses.

### Layer 2: PreToolUse Hook Layer (Mechanical Enforcement)

**Single consolidated hook per tool matcher** (multi-hook bug requires this).

#### smart-context.sh (matches: Read)

Consolidates smart-read + gitnexus enrichment into one hook:

```
Read request arrives
    ├─ Is this a rules/*.md file?
    │   └─ DENY: "Already in your context (auto-loaded). Use offset/limit for specific section."
    ├─ First read of this file?
    │   ├─ Is file indexed in GitNexus?
    │   │   ├─ Yes → ALLOW + additionalContext: pruned graph paths (PathRAG-style)
    │   │   └─ No → ALLOW (no enrichment)
    │   └─ Log to session cache: path, mtime, timestamp
    ├─ File edited since last read? (mtime changed)
    │   └─ ALLOW, reset cache entry + additionalContext: "File changed since last read"
    ├─ Targeted read (offset/limit already set)?
    │   └─ ALLOW (user/Claude already scoped it)
    ├─ 2nd read of unchanged file?
    │   └─ ALLOW + additionalContext: "You read this file N minutes ago. Consider using offset/limit or delegating to [domain] agent."
    └─ 3rd+ read of unchanged file?
        └─ Strategy choice (configurable):
            Option A: DENY + cached summary + "delegate to agent or use offset/limit"
            Option B: ALLOW + updatedInput with narrowed offset/limit (graph-informed)
            NOTE: Never narrow first reads — degrades architectural understanding (arXiv 2505.13353)
```

**Session cache**: `/tmp/claude-reads-{transcript_hash}.jsonl`
```jsonl
{"path":"/home/rf15/KD-GAT/graphids/pipeline/export.py","count":2,"first_read":1709856000,"mtime":1709855000,"summary":"..."}
```

#### gitnexus-enrich.cjs (matches: Grep|Glob|Bash)

Existing hook — unchanged. Injects graph context on searches.

### Layer 3: Domain Expert Agents

**Purpose**: Isolate file reading into separate context windows. Main context sees 200-500 token summaries instead of 2,000+ token raw file dumps.

```yaml
# ~/.claude/agents/pipeline-expert.md
---
name: pipeline-expert
description: >
  KD-GAT pipeline domain expert. Use when investigating pipeline stages,
  orchestration, data flow, export, or training loop.
tools: [Read, Glob, Grep, Bash]
model: inherit  # Haiku only for Explore-style tasks; inherit/sonnet for design decisions
memory: project
mcpServers:
  gitnexus: {}
---
# System prompt with focused architecture knowledge
# Output contract: structured summary with file:line refs, max 500 words
```

| Agent | Domain | Trigger phrase |
|-------|--------|---------------|
| `pipeline-expert` | KD-GAT pipeline, stages, export, training | "pipeline", "training loop", "export", "Ray" |
| `mapviz-expert` | Map-Viz D3, Observable, DuckDB, charts | "map", "chart", "D3", "campaign finance" |
| `config-expert` | KD-GAT config, schema, YAML | "config", "schema", "hyperparameter" |

Design rules (from research):
- **3-4 agents max** — more creates decision overhead
- **Model selection by task**: `haiku` for read/summarize (Explore-style), `inherit`/`sonnet` for architectural reasoning (Haiku hallucinated filenames, missed bugs Sonnet caught)
- **Persistent memory** — agent learns across sessions, reducing cold-start reads
- **Output contract enforced in system prompt** — summaries with file:line refs, never raw dumps (hooks cannot validate agent final output, only tool calls within the agent)
- **MCP access** — each agent can query GitNexus for graph-informed reading

### Layer 4: Session Lifecycle Hooks

| Hook Event | Script | Purpose |
|------------|--------|---------|
| SessionStart | `inject-slurm-context.sh` | SLURM queue status (existing) |
| SessionStart | `session-health.sh` | Staleness + drift checks (existing) |
| PreCompact | `pre-compact.sh` | Inject recovery context + pointer to working memory file |
| Stop | `stop-verify.sh` | Uncommitted changes reminder (existing) |
| SessionEnd | `session-end.sh` | Log session metadata (existing) |

**New in PreCompact**: Before compaction, hook writes current session read cache to `~/plans/session-state.md` so post-compaction context includes "you already read these files: [list]".

### Layer 5: Payload Size Control (updatedInput Rewriting)

The most impactful lever that requires no behavioral change from Claude:

| Tool | Rewrite Rule | Savings |
|------|-------------|---------|
| **Read** re-reads | Add offset/limit scoped to relevant function (from graph) | 50-90% per read |
| **Read** re-reads | For edited files, return only changed lines (diff since last read) | Variable |
| **Grep** | Add `head_limit: 20` if no limit set | Prevents grep floods |
| **Glob** | Add `head_limit: 50` if no limit set | Prevents large listings |
| **Bash** | Append `| head -100` to commands without output limits | Prevents unbounded output |

This is **strictly better than blocking** because:
1. Claude still gets content (just scoped) — no retry loops
2. No quality degradation from missing context
3. Graph determines relevance, not hardcoded thresholds
4. Same number of hook firings, 90% less context consumed

## Research Validation

### Techniques That Directly Map to Our Architecture

| Research System | Technique | Our Implementation |
|----------------|-----------|-------------------|
| **Aider** repo map | Tree-sitter → PageRank → token budget fitting | GitNexus graph + pruned paths in additionalContext |
| **JetBrains** observation masking | Replace old tool outputs with placeholders | PreCompact hook + session read cache |
| **johnlindquist** trigger tables | Replace verbose docs with when→what | Trigger table in CLAUDE.md replacing rules content |
| **Prometheus** (SOTA SWE-bench) | Unified KG + multi-agent + memory | 3-layer architecture + domain expert agents |
| **Sourcegraph Cody** | 3-layer cache (stable/action/retrieval) | Boot context (stable) + session cache (action) + MCP (retrieval) |
| **PathRAG** | Flow-based path pruning over full subgraphs | GitNexus augment returns pruned paths, not neighborhoods |
| **InlineCoder** | Inline target into call chain | GitNexus additionalContext includes call chain on first read |
| **Nick Tune** | Respawn agents at iteration boundaries | Domain agents with clean context per invocation |

### Cross-Cutting Consensus (14 papers, 9 blog posts, 7 MCP servers, 6 competitor architectures)

1. **Graph + vector hybrid wins** — but BM25 is surprisingly competitive for code (Sourcegraph abandoned embeddings for adapted BM25)
2. **Every system returns snippets, never full files** — Aider truncates, Cursor chunks at function level, Cody ranks snippets
3. **Context budget is THE central design constraint** — binary search (Aider), hierarchical pruning (HCP), observation masking (JetBrains) all address this
4. **~150-200 instructions is the CLAUDE.md ceiling** — beyond this, LLMs lose consistency (Anthropic's own guidance)
5. **Masking > summarization** for compaction — masking is cheaper, no trajectory elongation, +2.6% solve rate (JetBrains)
6. **75% context utilization is the sweet spot** — quality degrades beyond this

## New Research Findings (Mar 8 Sweep)

### Meta-MCP Architecture (88% Tool Schema Reduction)

Instead of exposing 60+ individual MCP tool definitions (each consuming 550-850 tokens), create a meta-MCP server with just 2 tools: `search_docs` (search API definitions of child servers) and `execute_code` (run TypeScript/Python calling MCP bindings). Demonstrated: **36.6K → 4.4K tokens (88% reduction)**. Inspired by Cloudflare's Code Mode pattern (~1,000 tokens for an entire API).

> Source: [Meta-MCP Architecture](https://dev.to/tgfjt/a-practical-meta-mcp-architecture-for-claude-code-compressing-60-tools-into-just-two-oje)

**Relevance to us**: We have 6 MCP servers with ~50 deferred tools. Even with ToolSearch deferral, the tool name list in `<available-deferred-tools>` consumes tokens. A meta-MCP could consolidate gitnexus + lab-docs + memory into fewer endpoints.

### Context Mode MCP (98% Output Compression)

[Context Mode](https://news.ycombinator.com/item?id=47193064) is an MCP server that compresses tool outputs before they enter the context window: **315 KB raw → 5.4 KB compressed**. Sessions that previously died at 30 minutes ran for 3+ hours. This addresses the PostToolUse limitation (can't modify results) by operating at the MCP transport layer instead.

**Relevance to us**: Could wrap GitNexus or Grep results through a compression layer before they enter context. Operates outside the hook system entirely.

### Agent Teams / Swarms (Experimental)

Claude Code has an experimental Agent Teams feature (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` in settings.json):
- A lead agent coordinates specialists via shared task board + mailbox system
- Teammates communicate directly with each other (not just through lead)
- Each gets independent context windows and transcripts
- Quality gate hooks (exit code 2 blocks completion)
- Different from subagents: teams are **collaborators with shared state**, subagents are **isolated workers**

> Source: [Claude Code Agent Teams Docs](https://code.claude.com/docs/en/agent-teams), [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)

### Builder/Validator Pattern (disler/claude-code-hooks-mastery)

Two-agent workflow with mechanical separation of concerns:
- **Builder agent**: Full tool access, executes modifications
- **Validator agent**: Read-only tools (Read, Glob, Grep, safe Bash), audits quality without modification rights
- PostToolUse validators run ruff + type checking automatically
- Stop hook can block completion if validator finds issues

This addresses the "who watches the watchmen" problem — the validator literally cannot break things.

> Source: [claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery)

### Meta-Agent Pattern

An agent whose job is to **create and orchestrate other agents dynamically**. Instead of pre-defining all agents, a meta-agent spawns role-specific subagents based on task analysis. Enables adaptive agent composition per task type.

### Instinct-Based Learning (everything-claude-code)

Goes beyond static rules to **confidence-scored behavioral patterns** that evolve across sessions:
- SessionEnd hooks serialize learnings into persistent "instincts"
- SessionStart hooks reload them
- Patterns cluster into actionable domain knowledge with confidence scoring
- Import/export across projects

> Source: [everything-claude-code](https://github.com/affaan-m/everything-claude-code) (Anthropic hackathon winner)

### TAOR Runtime Model (Reverse-Engineered)

The runtime follows **Think-Act-Observe-Repeat** — "the runtime is dumb; the model is the CEO." Key insight: Claude Code implements a **six-tier permission resolver**:
1. Tool-level allow/deny (blocklist/allowlist)
2. Specifier patterns (glob matching for file paths)
3. Scope resolution (org/project/user hierarchies)
4. Static analysis before execution
5. Runtime validation post-execution
6. User approval gates

Auto-compaction triggers at **83.5% default** (configurable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env var, 1-100). The ~50% claim from this source was incorrect/outdated. Skills marked `context: fork` get isolated context.

> Source: [Claude Code Architecture (Reverse Engineered)](https://vrungta.substack.com/p/claude-code-architecture-reverse)

### Codex CLI's Diff-Based Forgetting

Codex CLI uses "diff-based forgetting memory" — remembering changes as diffs rather than full state. Compact representation that could inform PreCompact hook design: summarize the session as a series of diffs rather than narrative.

### cchook: YAML-Based Conditional Hooks

[cchook](https://github.com/syou6162/cchook) replaces complex JSON hook configurations with YAML, supporting template-based data access (`{.tool_input.file_path}`), conditional logic (file extension matching, directory conditions). Cleaner authoring than raw JSON in `settings.json`.

### Quantified Context Budget Allocation

From SFEIR Institute research:

| Category | Token Range | % of 200K |
|----------|-------------|-----------|
| System prompt + CLAUDE.md | 3,000-8,000 | 2-4% |
| Auto-read files | 10,000-50,000 | 5-25% |
| Conversation history | 30,000-120,000 | 15-60% |
| Current response | 5,000-20,000 | 3-10% |

Multi-session performance vs single saturated session:
- **Single session** (180K tokens): 8.2s response, 72% relevance
- **Three targeted sessions** (40K each): 2.1s response, **94% relevance**

Plan mode savings: 46-53% token reduction across code review, architecture analysis, and bug search tasks.

### Structured Prompts Survive Compaction Better

Structured prompts (tables, YAML, bullet lists) achieve **92% fidelity preservation** during compaction vs **71% for narrative text**. This validates our trigger table approach — tables compress better than prose.

### Subagent Hooks (Scoped Lifecycle)

Agents can define hooks in their YAML frontmatter that **only fire while that agent is active** and are cleaned up when it finishes. This enables agent-specific behavior without polluting the global hook configuration.

### Complete Subagent Frontmatter Reference

| Field | Options | Notes |
|-------|---------|-------|
| `name` | string | **Required** |
| `description` | string | **Required** — shapes when model spawns this agent |
| `model` | `sonnet`, `opus`, `haiku`, `inherit` | `haiku` for reading, `inherit` for quality-sensitive |
| `tools` | list | Explicit allowlist; omit = inherit all |
| `disallowedTools` | list | Evaluated after `tools` |
| `permissionMode` | `default`, `acceptEdits`, `dontAsk`, `plan`, `delegate`, `bypassPermissions` | `dontAsk` for autonomous agents |
| `mcpServers` | object | Agent-scoped MCP config |
| `hooks` | object | Agent-scoped lifecycle hooks (cleaned up on exit) |
| `maxTurns` | int | e.g., 25 |
| `skills` | list | Full skill content injected at startup |
| `memory` | string/bool | Persistent MEMORY.md (first 200 lines) |

## Implementation Roadmap

### Phase 0: Instrument (1 week, zero risk)
- Add read logging to a PreToolUse hook on Read (logging only, no blocking)
- Track: path, count, mtime, session position, offset/limit usage
- Measure GitNexus augment latency on common files
- **Quick win**: Test `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75` to trigger compaction earlier, giving more working space post-compaction
- **Output**: Baseline data for all subsequent phases

### Phase 1: Enrich (1-2 weeks, low risk)
- Consolidate into `smart-context.sh` matching Read
- On first read of indexed file: ALLOW + additionalContext with pruned graph paths
- No blocking — purely additive
- **Conservative narrowing only**: never narrow first reads (degrades architectural understanding). Only narrow re-reads of unchanged files where Claude already has full context.
- **Measure**: Does Claude read fewer related files when given neighborhood upfront?

### Phase 2: Delegate (2 weeks, medium risk)
- Create 3 domain expert agents in `~/.claude/agents/`
- **Model selection**: Use `model: inherit` or `sonnet` for agents making design-level decisions; Haiku only for Explore-style read/summarize tasks (Haiku hallucinated filenames, missed bugs Sonnet caught)
- Add trigger table to CLAUDE.md
- Move detailed architecture content from rules → agent system prompts
- Add `context: fork` to exploration skills (`/research`, `/navigate`, `/experiment`) to prevent their reads from consuming main context
- **Optional**: Add a read-only validator agent (`disallowedTools: [Edit, Write]`, `model: sonnet`) for semantic code review beyond ruff
- **Measure**: Agent delegation rate, main-context read count

### Phase 3: Intercept (2 weeks, higher risk)
- Enable re-read throttling (3rd+ read → narrowed via updatedInput OR denied)
- Block rules/*.md re-reads (already auto-loaded)
- Add payload caps to Grep/Glob/Bash
- **Measure**: Quality regression? Retry loops? Session completion rate?
- **Escape hatch**: Touch `/tmp/claude-unrestricted-{hash}` to disable blocking for current session

### Phase 4: Diet (ongoing, highest impact)
- Slim all rules files to trigger table format
- Move content to agent prompts, skills, MCP responses
- Use `claudeMdExcludes` in settings to skip rules that moved to agent prompts
- Expand path-scoped rules (YAML `paths:` frontmatter) to more rules files — already proven with slurm-hpc.md, d3-conventions.md
- Implement observation masking in PreCompact
- **Target**: Boot context <8,000 tokens

### Phase 5: Advanced (future)
- Symbol-level index (jCodeMunch-style tree-sitter → signatures → on-demand source via MCP)
- Personalized PageRank (weight graph by conversation relevance)
- Agent trace logging (learn which files Claude actually needs per task type)
- Embeddings via SLURM batch (GitNexus `--embeddings` blocked on login nodes)
- Augment Context Engine MCP evaluation

## Success Metrics

| Metric | Baseline | Phase 2 | Phase 4 |
|--------|----------|---------|---------|
| Boot context | 33,900 tok | ~25,000 | <8,000 |
| Reads/session (main) | 59.3 | <40 | <25 |
| Agent calls/session | ~1.1 | >3 | >5 |
| Re-reads of top files | 5-6 avg | <3 | <1 |
| Rules file re-reads | 23 (tooling.md) | <10 | 0 |
| GitNexus enrichment/session | 0 | >10 | >10 |
| Session quality | baseline TBD | no regression | no regression |

## Open Questions

### Resolved
- [x] Graph-based context helps? **Yes** — Prometheus, CodexGraph, LocAgent prove it
- [x] Snippets vs full files? **Snippets universally** — every competitor
- [x] Masking vs summarization? **Masking wins** — JetBrains research
- [x] Can updatedInput rewrite Read? **Yes, confirmed.** Bug: multi-hook last-wins
- [x] PostToolUse modify results? **No, rejected** by Anthropic
- [x] Token-level stream interception? **Impossible** at API consumer level
- [x] 33K pre-load too much? **Yes** — trigger tables achieve 54% reduction

### Resolved (continued — Mar 8 research sweep)
- [x] Hook latency for cache-only rewriting? **~20-50ms** for bash/warm Node.js doing JSONL cache lookup. Python has ~100-200ms startup — avoid for latency-sensitive hooks. Node.js (already warm for gitnexus-hook.cjs) is ideal.
- [x] Does narrowing reads degrade task quality? **YES for architectural/cross-file tasks.** arXiv 2505.13353 confirms success rates halve as context scales, partial reads degrade full logic flow understanding. **Safe only for isolated edits** (variable rename, docstring fix) or re-reads of unchanged files where Claude already has full context.
- [x] Claude API context editing beta usable alongside hooks? **NO, API-only.** `context-management-2025-06-27` beta is not accessible in Claude Code sessions. Claude Code uses auto-compaction instead. Not integrated.
- [x] Filesystem offloading pattern? **Blocked by bug + dangerous for edits.** Issue #15897: `updatedInput` silently ignored in some cases. Reading a summary then trying to Edit creates coherence failures — Claude won't have exact text for `old_string` matching. **Don't pursue.**
- [x] Haiku agent quality sufficient for design-level decisions? **Good for read/summarize, insufficient for architectural reasoning.** Haiku hallucinated filenames more often than Sonnet. Sonnet caught bugs (rebuild issues, missing disposes, async) that Haiku missed. Opus found subtle bugs (resource leaks, concurrency) that both missed. **Use `model: inherit` or Sonnet for design-level agents; Haiku only for Explore-style tasks.**
- [x] Cache coherence between agent reads and main-context reads? **Complete isolation.** Each subagent runs in a separate context window. Only final summary returns to main context. Main context session cache does NOT know about subagent reads. File must be read twice if both need it. By design (prevents context pollution).
- [x] Rules file auto-loading — can files be moved out? **YES.** Move files out of `.claude/rules/` (stops auto-loading), use `claudeMdExcludes` in settings, or add YAML frontmatter `paths: ["src/**/*.ts"]` for conditional loading. For Phase 4 "Context Diet," move detailed content to agent system prompts or skill references.
- [x] Cross-session read cache? **Defer to Phase 5.** Time-decay functions exist (0.05 decay per context shift in ECC), Redis+LangGraph patterns show viability, but Claude Code implementations are emerging. Session-scoped caching (Phase 0-3) is sufficient for now.
- [x] Meta-MCP feasibility? **Not worth it for heterogeneous servers.** MetaMCP exists as production aggregator, but gitnexus (KuzuDB), lab-docs (BM25), and memory (entity-relation JSON) have fundamentally different query models. Flattening to 2 tools loses domain-specific ranking and query expressiveness.
- [x] Context Mode MCP detail preservation? **Loses critical code details.** Compaction drops exact error messages, line numbers, variable names, stack traces. Only preserves high-level discoveries, task outcomes, key patterns. Not suitable for code editing tasks.
- [x] Agent Teams stability? **Experimental, not production-ready.** Session resumption broken (`/resume` doesn't restore teammates), task coordination lag, only one team per session, no nesting. **Skip; use subagents.** Revisit when session resumption is fixed.
- [x] Instinct system implementation? **ECC v2 has working 4-phase pipeline** (observe → analyze → codify → evolve) with quality safeguards (0.05 decay, contradiction cap, archive below 0.3). **Interesting but heavyweight.** Our existing knowledge-bank.md + memory MCP serves a similar role. Could adapt observation logging pattern without the full pipeline.
- [x] Auto-compaction threshold? **83.5% default, configurable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env var (1-100).** The reverse-engineering claim of ~50% was incorrect/outdated. For 200K window: triggers at ~167K tokens. Lower values (70%) give more working space post-compaction.
- [x] Subagent-scoped hooks for output contracts? **Hooks validate tool calls WITHIN the agent, not the agent's final output.** No hook fires when agent finishes and returns results to parent. **Enforce output contracts in agent system prompts only** (probabilistic but only available lever).
- [x] Builder/Validator quality gap? **Hooks catch ~60% (mechanical), agents add ~30-40% (semantic).** Ruff catches formatting/imports/type errors. Validator agent catches missing error handling, architectural violations, missing tests, security issues, cross-file consistency. arXiv 2603.00822 confirms semantic constraints require LLM judgment. **Worth implementing as Phase 2 addition** — read-only validator agent with `disallowedTools: [Edit, Write]` and `model: sonnet`.
- [x] `context: fork` on skills? **YES, officially supported.** YAML frontmatter `context: fork` runs skill in isolated sub-agent context. Skill does NOT access main conversation history. Results summarized and returned. **Skills like `/research`, `/navigate`, `/experiment` could benefit** — prevents file reads from consuming main context. Low-risk, high-impact optimization.
- [x] cchook worth the dependency? **No.** YAML is cleaner than JSON for hook configs, but our hooks are shell scripts — the value is marginal. Existing `settings.json` + scripts pattern works fine.

### Architecture Adjustments Based on Research Findings

| Original Plan | Finding | Revised Decision |
|---------------|---------|-----------------|
| Narrow reads via updatedInput offset/limit | Degrades quality for architectural tasks | Only narrow re-reads of unchanged files, not first reads |
| Haiku model for all domain agents | Insufficient for architectural reasoning | Use `model: inherit` or Sonnet for design-level agents; Haiku only for Explore-style tasks |
| Filesystem offloading pattern | Blocked by bug + edit coherence risk | Don't pursue; use additionalContext injection instead |
| Meta-MCP consolidation | Loses query precision for heterogeneous servers | Keep separate servers; optimize tool counts per server |
| Agent Teams | Session resumption broken | Skip; use subagents |
| Context Mode MCP | Loses code details | Not suitable for code editing; maybe for research-only sessions |
| Smart-read: aggressive narrowing | Quality degradation confirmed | Conservative narrowing: only for re-reads, always read full semantic units on first read |
| Subagent output validation via hooks | Not supported | Enforce output contracts in agent system prompts only |
| Cross-session read cache | Research phase | Defer to Phase 5; session-scoped caching is sufficient |

### New Opportunities Discovered

| Opportunity | Impact | Phase |
|-------------|--------|-------|
| `context: fork` on exploration skills | Prevents skill reads from consuming main context | Phase 2 |
| `claudeMdExcludes` for selective rule loading | Reduce boot context without moving files | Phase 4 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` tuning | Control when compaction fires (e.g., 75% for longer pre-compaction work) | Phase 0 |
| Builder/Validator agent split | Semantic code review beyond ruff | Phase 2 |
| Path-scoped rules (already partially used) | Expand to more rules files | Phase 4 |
| Observation logging (from ECC instinct system) | Deterministic tool call logging for pattern extraction | Phase 0 (already have via log-tool-usage.py) |

## Key References

### Academic (14 papers)
- Prometheus (SOTA SWE-bench 74.4%), CodexGraph, LocAgent, PathRAG, InlineCoder, SWE-Adept
- GraphRAG Survey, LEGO-GraphRAG, CodeRAG-Bench, RACG Survey
- Evaluating AGENTS.md, Agent READMEs, Mem0, AI Agentic Programming Survey

### Industry (15+ blog posts)
- Anthropic effective context engineering + Claude Code best practices
- JetBrains observation masking, 75% context sweet spot, trigger table 54% reduction
- Hook-driven workflows, Continuous-Claude-v3, context handoff plugin, 33K token buffer
- [Meta-MCP Architecture (88% reduction)](https://dev.to/tgfjt/a-practical-meta-mcp-architecture-for-claude-code-compressing-60-tools-into-just-two-oje)
- [Context Mode MCP (98% output compression)](https://news.ycombinator.com/item?id=47193064)
- [Claude Code Architecture Reverse-Engineered](https://vrungta.substack.com/p/claude-code-architecture-reverse)
- [SFEIR Institute Context Management](https://institute.sfeir.com/en/claude-code/claude-code-context-management/optimization/)
- [Claude Code Full Stack Explained](https://alexop.dev/posts/understanding-claude-code-full-stack/)
- [AI OS Blueprint](https://dev.to/jan_lucasandmann_bb9257c/claude-code-to-ai-os-blueprint-skills-hooks-agents-mcp-setup-in-2026-46gg)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)

### Tools (10+ MCP servers & utilities)
- Augment Context Engine, Code-Graph-RAG, jCodeMunch, RepoMapper
- claude-context (Zilliz), Code Pathfinder, OpenMemory
- [Context Mode MCP](https://news.ycombinator.com/item?id=47193064) — output compression
- [cchook](https://github.com/syou6162/cchook) — YAML-based conditional hooks
- [Cloudflare Code Mode](https://blog.cloudflare.com/code-mode-mcp/) — entire API in ~1K tokens

### Reference Implementations
- [disler/claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) — 13 lifecycle hooks, meta-agent, builder/validator teams
- [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) — instinct-based learning, cross-harness abstraction
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) — curated skills/hooks/plugins list
- [wesammustafa/Claude-Code-Everything](https://github.com/wesammustafa/Claude-Code-Everything-You-Need-to-Know) — comprehensive guide

### Competitors (6 architectures)
- Aider repo map, Cursor semantic search, Sourcegraph Cody 3-layer cache
- Continue.dev context providers, Context Engineering Framework, Qodo 3-layer retrieval
- Codex CLI diff-based forgetting, OpenCode multi-provider rules

> Full academic citations with URLs in `context-efficient-architecture.md`
