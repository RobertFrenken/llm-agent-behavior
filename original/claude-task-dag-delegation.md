# Claude Task-DAG Delegation: Nudging Claude to Decompose, Chunk, and Delegate

> Researched: 2026-03-21

## Context

The goal is to create a policy (via CLAUDE.md, skills, subagent definitions, and/or hooks) that nudges Claude Code to treat complex prompts as task lists — decomposing them into chunks, delegating chunks to subagents, and executing them following a DAG (directed acyclic graph) of dependencies. This builds on prior work in `~/plans/agent-nudging-design.md` (value confabulation nudging) and `~/plans/tool-routing-optimization.md` (tool routing via hooks).

### Why this matters

Without explicit nudging, Claude tends to execute complex multi-step prompts monolithically — streaming through all steps in the main context window. This causes:
- **Context pollution**: exploration output, intermediate results, and verbose tool output fill the window
- **No parallelism**: independent subtasks run sequentially
- **No isolation**: a failure in step 5 can corrupt context needed for step 6
- **Poor resumability**: if the session compacts or crashes, there's no external record of progress

The mechanisms now exist natively in Claude Code (subagents, agent teams, Task tool with dependency tracking, skills, hooks) — the question is how to make Claude **default to using them** for non-trivial work.

## Options Considered

### Option 1: CLAUDE.md Policy Nudge (Prompt-Only)

**What**: Add instructions to CLAUDE.md that tell Claude to decompose multi-step prompts into a task list before executing, and delegate independent chunks to subagents.

**Implementation**: A `<task_decomposition>` block in CLAUDE.md with rules like:
```
Before executing any prompt with 3+ distinct steps or actions:
1. Parse the prompt into discrete tasks
2. Identify dependencies between tasks (what blocks what)
3. Group independent tasks into parallel waves
4. Present the task plan to the user for approval
5. Execute each task via subagent delegation, respecting the DAG order
```

**Pros**:
- Zero infrastructure cost — just markdown
- Immediately active in every session
- Easy to iterate on wording
- Works with existing Claude Code capabilities

**Cons**:
- Suffers the same paradox as value-nudging (see `agent-nudging-design.md`): Claude must attend to the rule, but confidence causes it to skip the decomposition step and just start executing
- No enforcement mechanism — Claude can ignore it entirely
- Adds to CLAUDE.md bloat, diluting attention on existing rules
- Opus 4.6 is already more proactive about subagents; aggressive nudging may cause overuse on simple tasks

**Effort**: Small

### Option 2: Skill-Based Orchestrator (`/plan-and-execute`)

**What**: Create a Claude Code skill (`.claude/skills/plan-and-execute/`) that implements a structured decomposition-then-delegation workflow. The user invokes it explicitly via `/plan-and-execute <prompt>`, or Claude invokes it when it detects a complex prompt.

**Implementation**: A skill markdown file that:
1. Parses the user's prompt into tasks
2. Writes a `tasks.json` to a scratchpad directory with task IDs, descriptions, dependencies, status
3. Presents the DAG to the user for approval
4. Executes tasks in dependency order, delegating each to a subagent
5. Updates `tasks.json` after each completion
6. Synthesizes results when all tasks are done

**Pros**:
- Structured, repeatable workflow
- External state (`tasks.json`) survives compaction and can be resumed
- User retains approval gate before execution
- Can be shared across projects via `~/.claude/skills/`
- Skill descriptions guide Claude on when to invoke it

**Cons**:
- Requires user to remember to invoke it (unless combined with a nudge)
- Skill content is injected into context, consuming tokens even when not needed
- Skills run in the main conversation context — they don't inherently isolate
- More complex to maintain than a CLAUDE.md rule

**Effort**: Medium

### Option 3: Orchestrator Subagent (`orchestrator` agent definition)

**What**: Define a custom subagent in `.claude/agents/orchestrator.md` whose sole job is task decomposition and delegation. The main Claude delegates to the orchestrator for complex prompts, and the orchestrator spawns further subagents for each task.

**Implementation**: An agent markdown file with:
- Description: "Decomposes complex, multi-step prompts into a DAG of tasks and coordinates their execution via specialized subagents"
- System prompt with the decomposition protocol
- Tools: Agent (to spawn workers), Read, Bash, TaskCreate, TaskUpdate, TaskList
- Model: inherit (needs full capability for planning)

**Pros**:
- Isolated context — decomposition reasoning doesn't pollute the main window
- Can spawn worker subagents for each task chunk
- Description field guides Claude on when to delegate automatically
- Reusable across projects if placed in `~/.claude/agents/`

**Cons**:
- **Critical limitation**: subagents cannot spawn other subagents. The orchestrator would need to use the Task tool (which creates general-purpose subagents), not custom typed subagents
- Adds a layer of indirection — user prompt → main Claude → orchestrator → workers
- The orchestrator's context is separate from the main conversation, so it may miss relevant earlier context
- Harder to debug when things go wrong (nested delegation)

**Effort**: Medium

### Option 4: Agent Team with Shared Task List

**What**: Use Claude Code's experimental agent teams feature to spawn a team lead + teammates that coordinate via a shared task list with dependency tracking.

**Implementation**:
- Enable `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- Prompt Claude to "form a team for this task" or define a skill that triggers team formation
- Team lead decomposes the prompt, creates tasks with `TaskCreate`, sets dependencies with `TaskUpdate(addBlockedBy)`
- Teammates self-claim unblocked tasks and execute them
- Git-based coordination handles file conflicts

**Pros**:
- True parallelism — teammates run in separate 1M-token context windows
- Built-in dependency tracking with auto-unblocking
- Peer-to-peer messaging for coordination
- Best for large, genuinely parallelizable work (multi-module refactors, parallel analysis)

**Cons**:
- Experimental feature — API may change
- Requires Opus 4.6 model
- 2-3x token cost (each teammate is a full context window)
- Heavy coordination overhead for small tasks
- Write-heavy tasks cause merge conflicts
- Overkill for most day-to-day prompts

**Effort**: Medium-Large (setup + learning curve)

### Option 5: Hybrid — Tiered Nudge + Skill + Subagent

**What**: Combine Options 1-3 in a tiered system where the complexity of the prompt determines the response:

| Prompt Complexity | Mechanism | Example |
|---|---|---|
| 1-2 steps | Main conversation (no decomposition) | "Fix the import error in foo.py" |
| 3-5 steps | Skill-based decomposition (`/plan-and-execute` or auto-detected) | "Refactor the config system: extract constants, update imports, add tests" |
| 6+ steps or multi-module | Orchestrator subagent with worker delegation | "Implement the full KD pipeline: VGAE training, GAT distillation, DQN fusion, evaluation, SLURM scripts" |

**Implementation**:
1. CLAUDE.md policy nudge (lightweight, always-on) with complexity thresholds
2. `/plan-and-execute` skill for medium complexity
3. `orchestrator` subagent for high complexity
4. Optional: PreToolUse hook that detects multi-step Edit/Write sequences and suggests decomposition

**Pros**:
- Right-sized response for each situation
- Graceful degradation — if the nudge is ignored, the skill/subagent still works when invoked
- Modular — each tier can be improved independently
- Matches how human project managers think: small tasks just do, medium tasks plan first, big tasks delegate

**Cons**:
- Most complex to set up and maintain
- Complexity thresholds are fuzzy — Claude may misjudge
- Three mechanisms to keep in sync

**Effort**: Medium (incremental — build tier by tier)

## Recommendation

**Option 5 (Hybrid Tiered)**, built incrementally starting with the cheapest tier.

The reason is pragmatic: no single mechanism reliably changes Claude's behavior for all prompt complexities. The CLAUDE.md nudge alone will be ignored for exactly the prompts where decomposition matters most (the paradox from `agent-nudging-design.md`). But adding the skill and subagent as escalation paths gives the user explicit control when the nudge fails, and gives Claude a well-described delegation target when it does attend to the nudge.

**Build order:**
1. **Week 1**: CLAUDE.md nudge + `/plan-and-execute` skill (covers 80% of cases)
2. **Week 2**: `orchestrator` subagent (for the 20% that need full delegation)
3. **Week 3**: Evaluate and tune — measure how often each tier fires, adjust thresholds

The agent teams option (Option 4) should be reserved for genuinely large parallel workloads (multi-day refactors, full pipeline implementations) and invoked manually rather than as a default nudge.

## Implementation Sketch

### Tier 1: CLAUDE.md Nudge

Add to project or global CLAUDE.md:

```markdown
## Task Decomposition Policy

For prompts with 3+ distinct actions or steps:
1. Before executing, list the discrete tasks and their dependencies
2. Present the task list to the user with estimated complexity
3. For independent tasks, use subagents to run them in parallel
4. Track progress externally (scratchpad file) so work survives compaction

For prompts with 6+ steps or cross-module scope, use the orchestrator agent
or /plan-and-execute skill instead of working monolithically.

Do NOT decompose simple, single-purpose prompts. The overhead isn't worth it
for "fix this bug" or "add this test."
```

### Tier 2: `/plan-and-execute` Skill

Create `.claude/skills/plan-and-execute/SKILL.md`:

```markdown
---
name: plan-and-execute
description: Decomposes a complex prompt into a DAG of tasks, gets user approval, then executes each task via subagent delegation
---

## Protocol

1. Parse the user's prompt into discrete tasks
2. For each task, identify: ID, description, dependencies (blocked_by), estimated effort
3. Write the task list to `.claude/scratchpad/tasks.json`
4. Present the DAG to the user as a table (ID, description, depends_on, status)
5. On approval, execute tasks in topological order:
   - Independent tasks: delegate to subagents in parallel
   - Dependent tasks: wait for blockers to complete
6. After each task completes, update tasks.json status
7. When all tasks are done, synthesize a summary of changes
```

### Tier 3: Orchestrator Subagent

Create `.claude/agents/orchestrator.md`:

```markdown
---
name: orchestrator
description: Decomposes complex multi-step prompts into a DAG of tasks and coordinates execution via worker subagents. Use for prompts with 6+ steps or cross-module scope.
tools: Agent, Read, Glob, Grep, Bash
model: inherit
maxTurns: 50
---

You are a task orchestrator. Your job is to decompose complex prompts into
discrete tasks, identify dependencies, and coordinate execution.

## Protocol

1. Analyze the prompt and decompose into atomic tasks
2. Build a dependency graph (what blocks what)
3. Group independent tasks into parallel waves
4. For each wave, spawn worker subagents to execute tasks
5. Collect results and pass context to the next wave
6. Synthesize final results and report to the caller

## Rules

- Never execute tasks yourself — always delegate to workers
- Each worker gets a focused, self-contained prompt
- Track progress in a structured format
- If a task fails, report the failure and suggest remediation
- Minimize the number of waves (maximize parallelism)
```

## Source Files (read during implementation)

| File | Why |
|------|-----|
| `~/.claude/agents/` | Location for user-level subagent definitions |
| `.claude/agents/` | Location for project-level subagent definitions |
| `.claude/skills/` | Location for skill definitions |
| `~/KD-GAT/CLAUDE.md` | Project CLAUDE.md to add the nudge policy |
| `~/.claude/CLAUDE.md` | Global CLAUDE.md for cross-project nudge |
| `~/plans/agent-nudging-design.md` | Prior work on the nudge paradox — the confidence-error correlation |
| `~/plans/tool-routing-optimization.md` | Prior work on hook-based behavior steering |

## Open Questions

1. **Threshold calibration**: "3+ steps" and "6+ steps" are arbitrary. Need to observe real prompts and see where the natural breakpoints are. Could instrument with a PostToolUse hook that logs prompt complexity vs. whether decomposition was used.

2. **Subagent nesting limitation**: Subagents cannot spawn other subagents. The orchestrator subagent can only use the Task/Agent tool to spawn general-purpose workers, not typed custom subagents. Is this sufficient, or do workers need specialized system prompts? If so, the orchestrator must inline the specialization into the task prompt.

3. **State format**: `tasks.json` vs. markdown checklist vs. git-tracked progress file. JSON is machine-readable but harder for Claude to update reliably. Markdown is easier to write but harder to parse for status. Need to test both.

4. **Auto-detection vs. explicit invocation**: Should Claude auto-detect complex prompts and invoke the skill/subagent, or should the user always invoke explicitly? Auto-detection risks the nudge paradox; explicit invocation adds friction. The hybrid approach (nudge suggests, user confirms) may be the sweet spot.

5. **Cost awareness**: Subagent delegation multiplies token usage. For OSC work with limited allocation, need a cost-conscious mode that decomposes but executes sequentially in the main window rather than spawning subagents.

6. **Interaction with existing `/research` skill**: The `/research` skill already does planning-only work. Should `/plan-and-execute` call `/research` for the planning phase, or are they independent? Likely independent — `/research` produces a document, `/plan-and-execute` produces executable tasks.

## Cross-Repo Impact

- **`~/dotfiles`**: If the CLAUDE.md nudge goes in `~/.claude/CLAUDE.md`, it applies to all projects via chezmoi. Need to template it so it only activates on machines where Claude Code is installed.
- **`~/KD-GAT`**: Primary testing ground. The 3-stage pipeline (VGAE → GAT → DQN) is a natural multi-step DAG.
- **`~/lab-setup-guide`**: Low complexity — nudge should correctly identify most prompts here as below threshold.

## References

- [Claude Code: Create Custom Subagents](https://code.claude.com/docs/en/sub-agents) — official docs on subagent definitions, tool control, hooks, memory
- [Claude Code Workflow Orchestration plugin](https://github.com/barkain/claude-code-workflow-orchestration) — community plugin for two-stage plan+execute with wave-based parallel execution
- [Agent Teams Lite (Spec-Driven Development)](https://github.com/Gentleman-Programming/agent-teams-lite) — orchestrator + 9 specialized sub-agents, explore→propose→spec→tasks→apply→verify→archive DAG
- [Claude Code Swarm Orchestration](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea) — patterns for TaskCreate with dependency tracking, parallel specialists, sequential pipelines, self-organizing swarms
- [Agent Teams Guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide/blob/main/guide/workflows/agent-teams.md) — lead-teammate model, git-based coordination, shared task lists
- [Claude 4.6 Prompting Best Practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices) — official guidance on subagent orchestration, parallel tool calling, long-horizon reasoning
- [The Task Tool: Claude Code's Agent Orchestration System](https://dev.to/bhaidar/the-task-tool-claude-codes-agent-orchestration-system-4bf2) — breakdown of Task tool internals
