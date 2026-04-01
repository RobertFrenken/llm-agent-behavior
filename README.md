# llm-agent-behavior

Observations, analysis, and experimental designs from daily use of Claude Code as a research programming assistant.

## What's Here

| Directory | Contents |
|-----------|----------|
| `original/` | 9 source documents — raw research notes written during real work sessions |
| `compacted/` | Condensed versions — core findings, insights, takeaways only |
| `analysis/` | Cross-document data mining — tagged catalog, co-occurrence, research questions |
| `methodology/` | Step-by-step experimental procedures — how to execute each RQ |

## Scope

This repo covers **behavioral patterns of LLM coding agents** observed empirically through instrumented Claude Code sessions. Topics include:

- Hook-based steering (enforcement vs. advisory)
- Context window dynamics (momentum, compaction, re-read waste)
- Tool routing reliability (structured tools vs. Bash duplication)
- Enrichment effectiveness (graph context injection)
- Task decomposition and delegation
- Nudging, correction resistance, and agreement bias

## Not In Scope

- Claude Code configuration files or hook source code (those live in `~/.claude/`)
- Training scripts, model weights, or ML experiment tracking
- General LLM benchmarking or prompt engineering guides
- Session logs or raw data (those live in `~/.cache/claude-usage/`)

## Status

> **This repo's purpose and structure are still being defined.** See `repo-questionnaire.md` for the open questions that will shape its direction.

## Contributing a New Document

Before adding a new file, check:

1. **Does it belong here?** If it's about agent behavior patterns observed during real use, yes. If it's a how-to guide, config file, or project-specific note, it belongs elsewhere.
2. **Which directory?** Raw observations → `original/`. If compacted → `compacted/`. If it synthesizes across documents → `analysis/`. If it describes how to run an experiment → `methodology/`.
3. **Tag it.** Use the 26-tag vocabulary from `analysis/raw-catalog.md` so future analysis can incorporate it.
