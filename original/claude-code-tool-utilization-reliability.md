# Claude Code Tool Utilization Reliability

> Researched: 2026-03-21
> Related: `advanced-claude-configuration.md`, `tool-routing-optimization.md`

## Context

Claude Code has access to a rich ecosystem of tools: built-in structured tools (Read, Grep, Glob, Edit), MCP servers (memory, gitnexus, lab-docs, context7, obsidian-vault, playwright), custom skills (12 user-defined), subagents, and hooks. Despite this, Claude frequently ignores available tools — falling back to raw Bash for file operations (~40% of Bash calls duplicate structured tools), skipping MCP servers that could answer questions directly, and not invoking skills that match the user's request.

This plan consolidates research on **why** tools get ignored and **what configuration patterns** reliably improve utilization, drawing from Anthropic's official documentation, the advanced tool use engineering blog, community patterns, and academic research on LLM tool selection.

## The Problem: Five Failure Modes

### 1. Context Window Saturation Drowns Tool Awareness

Claude Code loads tool descriptions, skill descriptions, CLAUDE.md, rules files, and hook-injected context into the same finite window. As context fills, Claude's attention to tool availability degrades. Anthropic's own docs state: "LLM performance degrades as context fills. Claude may start forgetting earlier instructions."

**Evidence**: The skill description budget is 2% of context window (fallback 16,000 chars). With many skills + MCP tools + CLAUDE.md, descriptions compete for attention. Bloated CLAUDE.md files cause Claude to "ignore your actual instructions" (Anthropic best practices).

### 2. Tool Description Quality Determines Selection

Claude selects tools based on name + description matching against the current task. Poor descriptions lead to poor selection. Anthropic's advanced tool use blog reports:
- Tool Search Tool improved Opus 4 accuracy from 49% to 74%
- Tool Use Examples improved parameter handling from 72% to 90%
- The key insight: **tool descriptions are the primary signal for selection**

For MCP servers, tool descriptions are defined by the server author. If the description doesn't match how the user phrases their request, Claude won't select the tool.

### 3. CLAUDE.md Routing Tables Are Advisory, Not Deterministic

The "When you need X, use Y" table pattern (present in `~/CLAUDE.md`) provides routing hints. But CLAUDE.md is advisory — Claude follows it ~80% of the time. The routing pattern article (dev.to/builtbyzac) recommends keeping CLAUDE.md to ~12 lines precisely because longer files get partially ignored.

**Current state**: `~/CLAUDE.md` has a routing table but also substantial content. Project CLAUDE.md files add more. The cumulative effect may dilute routing signals.

### 4. Effort Level Affects MCP Tool Utilization

At lower effort levels, Claude takes shortcuts and skips MCP tools in favor of faster built-in alternatives. The BSWEN analysis (2026-03-13) documents that MCP tools require higher effort levels to be reliably utilized. This is particularly relevant when `alwaysThinkingEnabled` is true but effort isn't explicitly set to high.

### 5. Skill Description Matching Is Fuzzy

Skills auto-invoke based on name + description matching against conversation content. If two skills have overlapping descriptions, Claude picks the wrong one. If the description doesn't use keywords the user naturally says, the skill never triggers. The Anthropic skills docs confirm: "If Claude doesn't use your skill when expected, check the description includes keywords users would naturally say."

## Proven Patterns for Improving Tool Utilization

### Pattern 1: Minimize CLAUDE.md, Maximize Routing Precision

**Source**: Anthropic best practices, CLAUDE.md routing pattern

**Principle**: For each line in CLAUDE.md, ask "Would removing this cause Claude to make mistakes?" If not, cut it.

**Specific actions**:
- Keep root CLAUDE.md under 30 lines — routing table + critical constraints only
- Move domain knowledge to skills (loaded on demand, not every session)
- Move enforcement rules to hooks (deterministic, not advisory)
- Use `@path/to/import` syntax to delegate to files only when needed
- Prune rules/ files of anything Claude already does correctly without instruction

**Expected impact**: Higher signal-to-noise ratio in always-loaded context means routing instructions get more attention.

### Pattern 2: Write Tool-Trigger-Optimized Skill Descriptions

**Source**: Anthropic skills docs, advanced tool use blog

**Principle**: Descriptions are the primary matching signal. Write them for the user's vocabulary, not the developer's.

**Specific actions**:
- Include trigger phrases users actually say: "Use when the user asks 'how does X work?'" or "Use when exploring unfamiliar code"
- Make descriptions mutually exclusive across skills — no overlapping trigger conditions
- Test by asking "What skills are available?" and verifying Claude lists them
- For skills that should auto-invoke, keep `disable-model-invocation: false` (default) and write rich descriptions
- For side-effect skills (deploy, commit), always set `disable-model-invocation: true`

**Expected impact**: Anthropic reports moving from generic to specific descriptions is the single most effective fix for skill non-triggering.

### Pattern 3: Hook-Based Nudging via additionalContext

**Source**: Anthropic hooks docs, MIT "Hypersensitive to Nudges" paper (ICLR 2026), existing smart-context.sh

**Principle**: Hooks fire deterministically. `additionalContext` injects text Claude sees alongside tool results. This is the mechanically enforceable equivalent of CLAUDE.md routing.

**Specific actions**:
- **SessionStart hook**: Inject a concise tool capability summary (which MCP servers are available and what they do) as `additionalContext`. This refreshes tool awareness at session start.
- **UserPromptSubmit hook**: Pattern-match user prompts against tool capabilities. If the user asks about "code structure" or "dependencies", inject "Consider using gitnexus MCP tools for this query." If they ask about "docs" or "setup", inject "lab-docs MCP can answer this."
- **PreToolUse hook on Bash**: When Claude uses `cat`, `grep`, or `find` via Bash, inject `additionalContext` suggesting the structured tool equivalent (Read, Grep, Glob). The existing gitnexus hook partially does this.

**Expected impact**: The ICLR 2026 nudging paper shows LLMs are far more responsive to nudges than humans — weak cues have disproportionately large effects. Hook-injected context is a direct implementation of this finding.

**Caution**: Over-nudging creates brittleness and token waste. Each `additionalContext` injection is ~20-50 tokens. Only nudge when the pattern match is high-confidence.

### Pattern 4: Use the Tool Search Tool Pattern for Large Tool Libraries

**Source**: Anthropic advanced tool use blog

**Principle**: Instead of loading all tool definitions upfront (which consumes context), use deferred tool loading. Claude discovers tools on-demand.

**Current state**: This is already partially implemented — the system uses `<available-deferred-tools>` for MCP tools. Claude calls `ToolSearch` to fetch schemas when needed.

**Specific actions**:
- Verify all MCP server tools are properly deferred (not eagerly loaded)
- Ensure tool names are descriptive enough for search matching (e.g., `mcp__memory__search_nodes` is good; `mcp__x__do_thing` is bad)
- In CLAUDE.md, mention the ToolSearch tool and when to use it: "If you need a tool that isn't loaded, use ToolSearch to find it"

**Expected impact**: Anthropic reports 85% reduction in token usage with Tool Search Tool, and accuracy improvements from 49% to 74% for Opus 4.

### Pattern 5: Memory MCP for Cross-Session Tool Learning

**Source**: memory MCP server, Anthropic memory tool docs

**Principle**: If Claude learns a tool routing pattern in session N, persist it so session N+1 starts with that knowledge.

**Specific actions**:
- Use the memory MCP to store tool-routing observations: "For KD-GAT architecture questions, gitnexus_query is more effective than grep"
- In SessionStart hook, read key memory nodes and inject as context
- The existing `~/.claude/projects/.../memory/MEMORY.md` partially serves this purpose but is static

**Expected impact**: Prevents re-learning tool preferences every session. Compounds over time.

### Pattern 6: Explicit "When to Use What" in Skill Descriptions (Not CLAUDE.md)

**Source**: Anthropic skills docs ("Reference content adds knowledge Claude applies")

**Principle**: Move the routing table from CLAUDE.md (always loaded, competes for attention) to a skill with `user-invocable: false` that Claude loads automatically when relevant.

**Specific actions**:
- Create a skill like `tool-guide` with `user-invocable: false` and a description like "Tool selection guide for this project. Use when deciding which tool to use for code exploration, documentation lookup, or codebase navigation."
- The skill content contains the detailed routing table currently in CLAUDE.md
- CLAUDE.md keeps only a one-liner: "For tool selection guidance, the tool-guide skill loads automatically."

**Expected impact**: Routing knowledge loads on-demand instead of consuming budget every session. The skill's description triggers loading precisely when tool selection is ambiguous.

### Pattern 7: Effort Level Configuration

**Source**: BSWEN analysis, Anthropic skill frontmatter docs

**Principle**: MCP tool utilization correlates with effort level.

**Specific actions**:
- For skills that depend on MCP tools (gitnexus, research, navigate), set `effort: high` in frontmatter
- Consider setting session-level effort to high when MCP-heavy workflows are expected
- The `alwaysThinkingEnabled: true` setting (already configured) helps but is not sufficient alone

**Expected impact**: Direct fix for the observed pattern of Claude skipping MCP tools at default effort.

## Recommendation

Implement patterns in priority order:

1. **Pattern 1 (Prune CLAUDE.md)** — highest ROI, zero risk. Audit all CLAUDE.md and rules files. Delete anything Claude does correctly without instruction. Target: root CLAUDE.md under 30 lines, KD-GAT CLAUDE.md under 50 lines.

2. **Pattern 2 (Skill descriptions)** — audit all 12 skill descriptions. Rewrite with trigger phrases. Make descriptions mutually exclusive. Test with "What skills are available?"

3. **Pattern 3 (Hook nudging)** — implement a UserPromptSubmit hook that pattern-matches prompts and injects tool suggestions. Start with 3-5 high-value patterns (code structure -> gitnexus, docs -> lab-docs/context7, memory -> memory MCP).

4. **Pattern 6 (Routing skill)** — move the routing table from CLAUDE.md to a `tool-guide` skill. Reduce CLAUDE.md to a pointer.

5. **Patterns 4, 5, 7** — lower priority, implement as needed.

Do NOT implement all patterns simultaneously. Each one changes context dynamics. Implement one, observe for 5-10 sessions, then add the next.

## Implementation Sketch

### Phase 1: Prune (1 hour)

1. Read all CLAUDE.md files and rules/*.md files
2. For each line, ask: does Claude already do this correctly without it?
3. Delete or move to skills anything that isn't critical-path
4. Target: 50% reduction in always-loaded text

### Phase 2: Skill Description Audit (30 min)

1. Read all 12 SKILL.md files
2. Rewrite descriptions with natural-language trigger phrases
3. Ensure no two skills have overlapping trigger conditions
4. Test auto-invocation with representative prompts

### Phase 3: UserPromptSubmit Nudge Hook (1 hour)

1. Create `~/.claude/hooks/tool-nudge.sh`
2. Pattern-match on UserPromptSubmit input for keywords
3. Return `additionalContext` suggesting the right tool/MCP
4. Add to settings.json hooks config
5. Monitor via tool-usage.jsonl for 10 sessions

### Phase 4: Routing Skill (30 min)

1. Create `~/.claude/skills/tool-guide/SKILL.md`
2. Move routing table from CLAUDE.md
3. Set `user-invocable: false`
4. Replace CLAUDE.md table with one-line pointer

## Source Files (read during implementation)

| File | Why |
|------|-----|
| `~/.claude/CLAUDE.md` | Prune target — global instructions |
| `~/KD-GAT/CLAUDE.md` | Prune target — project instructions |
| `~/.claude/settings.json` | Hook configuration, effort settings |
| `~/.claude/skills/*/SKILL.md` | All 12 skill descriptions to audit |
| `~/.claude/hooks/smart-context.sh` | Existing hook pattern to follow for new nudge hook |
| `~/.claude/hooks/gitnexus/gitnexus-hook.cjs` | Existing Bash-nudge pattern |
| `~/plans/tool-routing-optimization.md` | Prior research on structured vs Bash tool routing |
| `~/plans/advanced-claude-configuration.md` | Prior research on mechanical enforcement |

## Open Questions

1. **What is the actual skill description character budget?** The docs say 2% of context window with 16K char fallback. With 1M context on Opus 4.6, that's 20K chars — generous. But does it scale linearly or is there a hard cap?

2. **Can UserPromptSubmit hooks see the full prompt or just metadata?** The docs show `tool_input` for PreToolUse but the UserPromptSubmit input schema isn't fully documented. Need to test whether the user's prompt text is available for pattern matching.

3. **Does `effort: high` on a skill override the session effort only for that skill's execution, or for the remainder of the session?** The docs say "Overrides the session effort level" but the scope is unclear.

4. **How does the Tool Search Tool interact with MCP server descriptions?** If MCP tool names are opaque (e.g., `mcp__gitnexus__query`), does Tool Search match on name only or also on description? Testing needed.

5. **Is there a measurable difference in tool selection quality between Opus 4.6 effort levels?** The BSWEN article claims yes but provides no controlled data. The Phase 1 experiment from `tool-routing-optimization.md` would answer this.

## Cross-Repo Impact

| Repo | Impact |
|------|--------|
| `dotfiles` | If CLAUDE.md pruning changes `~/.claude/CLAUDE.md`, it propagates to all projects via chezmoi |
| `KD-GAT` | Project CLAUDE.md and skills live here — direct edit target |
| `lab-setup-guide` | Has its own CLAUDE.md; pruning pattern should apply there too |
| All projects | Hook changes in `~/.claude/settings.json` affect every project |

## Key Sources

- [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) — Anthropic official
- [Hooks Reference](https://code.claude.com/docs/en/hooks) — PreToolUse, additionalContext, updatedInput
- [Extend Claude with Skills](https://code.claude.com/docs/en/skills) — description matching, frontmatter, invocation control
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) — Tool Search Tool, programmatic calling, examples
- [Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) — API-level best practices
- [LLM Agents Are Hypersensitive to Nudges](https://arxiv.org/abs/2505.11584) — ICLR 2026, supports hook injection
- [Codified Context: Infrastructure for AI Agents](https://arxiv.org/abs/2602.20478) — three-tier context architecture
- [MCP Tools Effort Levels](https://docs.bswen.com/blog/2026-03-13-mcp-tools-effort/) — effort level correlation
- [Troubleshooting MCP](https://www.arsturn.com/blog/why-is-claude-ignoring-your-mcp-prompts-a-troubleshooting-guide) — four failure modes
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) — Anthropic official
