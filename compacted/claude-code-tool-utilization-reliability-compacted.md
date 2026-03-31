## Core Findings

- **~40% of Bash calls duplicate structured tools** (Read, Grep, Glob, Edit) -- Claude defaults to raw Bash for file operations despite having purpose-built alternatives.
- **Context window saturation degrades tool awareness.** Skill description budget is 2% of context (16K char fallback). Bloated CLAUDE.md files cause Claude to ignore instructions.
- **Tool descriptions are the primary selection signal.** Tool Search Tool improved Opus 4 accuracy from 49% to 74%. Tool Use Examples improved parameter handling from 72% to 90%.
- **CLAUDE.md routing tables ("When you need X, use Y") are advisory, not deterministic** -- followed ~80% of the time. Recommended CLAUDE.md length: ~12 lines for reliable compliance.
- **Lower effort levels cause Claude to skip MCP tools** in favor of faster built-in alternatives. `alwaysThinkingEnabled: true` helps but is insufficient alone.
- **Skill auto-invocation depends on fuzzy name+description matching.** Overlapping descriptions cause wrong-skill selection; missing user-vocabulary keywords prevent triggering entirely.

## Key Insights

- **LLMs are hypersensitive to nudges** (ICLR 2026 paper): weak cues have disproportionately large effects. Hook-injected `additionalContext` is a direct implementation -- but over-nudging creates brittleness and token waste (~20-50 tokens per injection).
- **Enforcement hierarchy matters**: hooks are deterministic and mechanical; CLAUDE.md is advisory and degrades with context length. Move enforcement to hooks, knowledge to on-demand skills, and keep CLAUDE.md to routing-only.
- **Skill descriptions should be written for user vocabulary, not developer vocabulary.** Include trigger phrases users actually say. Make descriptions mutually exclusive across skills.
- **Tool deferred loading (Tool Search Tool pattern) cut token usage 85%** while improving accuracy -- already partially implemented via `<available-deferred-tools>`.
- **Cross-session tool-routing memory doesn't persist by default.** Claude re-learns tool preferences every session unless they're stored in memory MCP or MEMORY.md.
- **Each pattern changes context dynamics** -- implementing multiple simultaneously makes effects unattributable. Observe 5-10 sessions between changes.

## Takeaways

1. **Prune CLAUDE.md aggressively** (highest ROI, zero risk). Root CLAUDE.md under 30 lines, project CLAUDE.md under 50 lines. For each line: "Would removing this cause mistakes?" If no, cut it. Move domain knowledge to skills, enforcement to hooks.
2. **Rewrite all skill descriptions with natural-language trigger phrases.** Make descriptions mutually exclusive. Test with "What skills are available?" Side-effect skills (deploy, commit) must set `disable-model-invocation: true`.
3. **Implement a UserPromptSubmit nudge hook** that pattern-matches prompts and injects tool suggestions via `additionalContext`. Start with 3-5 high-confidence patterns (code structure -> gitnexus, docs -> lab-docs/context7, memory -> memory MCP).
4. **Move the routing table from CLAUDE.md to a `tool-guide` skill** with `user-invocable: false` -- loads on-demand instead of consuming budget every session.
5. **Set `effort: high` on skills that depend on MCP tools** (gitnexus, research, navigate) to counteract the effort-level MCP skip pattern.
6. **Implement sequentially, not simultaneously.** Priority order: prune CLAUDE.md -> audit skill descriptions -> hook nudging -> routing skill -> deferred loading / memory / effort tuning.
7. **Verify MCP tool names are descriptive enough for ToolSearch matching** (e.g., `mcp__memory__search_nodes` is good; opaque names fail search).
