## Core Findings

- **33,900 tokens (17% of 200K) consumed at boot** by auto-loaded rules/CLAUDE.md before any user interaction. Target: <8,000 tokens via trigger tables (proven 54% reduction).
- **Massive re-read waste**: rules files re-read 22-31x/session (e.g., MEMORY.md: 31x), source files up to 90x (export.py: 90x, cli.py: 80x). Root cause: Claude lacks session-scoped memory of prior reads -- every reasoning turn starts fresh.
- **Only 4 mechanically enforceable control surfaces exist**: (1) system prompt (probabilistic, degrades past ~150-200 instructions), (2) PreToolUse hooks (deterministic -- deny/rewrite/inject), (3) PostToolUse hooks (observational only, cannot modify results), (4) agent isolation (separate context windows).
- **Text generation cannot be hooked** -- no interception during reasoning/analysis. PostToolUse cannot modify tool results (Anthropic: NOT_PLANNED). Multi-hook on same matcher silently drops earlier `updatedInput` (bug #15897).
- **Auto-compaction triggers at 83.5%** (configurable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`, 1-100). The widely cited ~50% figure is incorrect/outdated.
- **Three targeted 40K-token sessions outperform one saturated 180K session**: 2.1s vs 8.2s response time, 94% vs 72% relevance.
- **Structured prompts (tables, YAML, bullets) preserve 92% fidelity during compaction** vs 71% for narrative prose.
- **75% context utilization is the quality sweet spot** -- performance degrades beyond this.

## Key Insights

- **`updatedInput` rewriting is strictly better than blocking reads**: Claude still gets scoped content (no retry loops), same hook count, 50-90% less context consumed. Graph-informed scoping determines relevance rather than hardcoded thresholds.
- **Never narrow first reads** -- arXiv 2505.13353 confirms success rates halve when partial reads degrade full logic flow understanding. Safe only for re-reads of unchanged files or isolated edits (variable rename, docstring fix).
- **Haiku hallucinates filenames and misses bugs that Sonnet catches; Opus finds subtle bugs (resource leaks, concurrency) that both miss.** Use Haiku only for read/summarize (Explore-style), Sonnet/inherit for design decisions.
- **Subagent context is completely isolated** -- main context session cache doesn't know about subagent reads (by design, prevents context pollution). A file must be read twice if both contexts need it.
- **Masking beats summarization for compaction** -- cheaper, no trajectory elongation, +2.6% solve rate (JetBrains research). Every competitor (Aider, Cursor, Cody) returns snippets, never full files.
- **Graph + vector hybrid wins for code retrieval**, but BM25 is surprisingly competitive -- Sourcegraph abandoned embeddings for adapted BM25.
- **Hooks cannot validate agent final output** -- only tool calls within the agent. Output contracts must be enforced in agent system prompts (probabilistic, but only available lever).
- **Filesystem offloading (read summary, then Edit) is dangerous**: Claude won't have exact text for `old_string` matching, causing coherence failures. Blocked by bug + edit coherence risk.
- **Meta-MCP not worth it for heterogeneous servers** -- flattening gitnexus (KuzuDB), lab-docs (BM25), memory (entity-relation) into 2 generic tools loses domain-specific ranking and query expressiveness.
- **Agent Teams experimental, not production-ready** -- session resumption broken, task coordination lag. Subagents are the reliable path.

## Takeaways

- **Consolidate to one hook per tool matcher** (multi-hook bug requires it). Build `smart-context.sh` for Read: log first reads, warn on 2nd, narrow/deny on 3rd+ re-reads of unchanged files.
- **Replace verbose rules files with trigger tables** ("When you need X -> Use Y") to cut boot context by 54%+. Move detailed architecture content into agent system prompts and skill references.
- **Cap 3-4 domain expert agents max** -- more creates decision overhead. Each agent should enforce an output contract (structured summary with file:line refs, max 500 words) in its system prompt.
- **Add payload caps to all search tools**: `head_limit: 20` on Grep, `head_limit: 50` on Glob, `| head -100` on unbounded Bash output -- prevents context floods with no behavioral change from Claude.
- **Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75`** to trigger compaction earlier, preserving more working space post-compaction.
- **Use `context: fork` on exploration skills** (`/research`, `/navigate`) to prevent their reads from consuming main context. Low-risk, high-impact.
- **Write PreCompact hook** that serializes the session read cache to a recovery file, so post-compaction context includes "you already read these files: [list]" -- directly attacks the re-read problem.
- **Use Node.js (not Python) for latency-sensitive hooks** -- Node.js warm startup is ~20-50ms for cache lookup vs Python's ~100-200ms.
- **Instrument before optimizing** (Phase 0): add read logging to PreToolUse, track path/count/mtime/session position. Baseline data is prerequisite for all subsequent phases.
- **Keep Context Mode MCP and aggressive output compression away from code editing** -- they lose exact error messages, line numbers, variable names, and stack traces. Only suitable for research-only sessions.
