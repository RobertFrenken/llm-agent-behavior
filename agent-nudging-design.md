# Agent Nudging: Tool Use vs Weight-Based Generation

> Brainstormed: 2026-03-10

## The Problem

LLM agents (Claude Code) generate infrastructure values from training priors instead of looking them up in the codebase. This causes stale or wrong values when project conventions diverge from common training data patterns.

**Concrete example:** `executor.py` was written with `partition_cpu="serial"` when every SLURM script in the repo uses `"cpu"`. The agent had read the correct scripts earlier in the session but generated from training priors instead.

This is not a one-off. The pattern recurs daily across different value categories: SLURM partition names, account IDs, module versions, file paths, config keys.

### The confidence-error correlation

The agent is MOST likely to skip tool use when MOST confident from training priors. But project-specific values are exactly where training priors are most likely to be wrong. High confidence correlates with high error rate for infrastructure values that deviate from common patterns.

Categories most affected:
- SLURM parameters (partitions, accounts, constraints)
- Module names and versions (`cuda/12.4` vs `cuda/12.4.1`)
- Project-specific paths and constants
- Config schema field names
- Version pins in build files

## The Paradox

PreToolUse hooks can inject real-time context, but only fire when the agent already decided to use a tool. The problem is the agent does NOT use tools when it feels confident -- so the correction mechanism requires the behavior it is trying to create.

```
Agent confident about value
  -> skips lookup (no tool call)
  -> composes Edit/Write with wrong value
  -> PreToolUse on Edit fires, but damage is done (wrong value already in new_string)
  -> Hook can block, but this is error correction, not behavior change
```

Hooks are "cleaning up the agent's mess," not making the agent better. The agent does not learn from the block -- it just retries with corrected values, and will make the same mistake next session.

## Design Approaches

### 1. Reactive: Edit/Write validation hooks

PreToolUse hook on Edit/Write scans `new_string` for known-risky tokens (partition names, account IDs, module versions, paths). Maintains a registry of values that must match codebase sources. On mismatch: block the tool call and inject the correct value.

**Implementation:** Extend existing auto-format hook or add a sibling hook. Registry is a JSON/YAML file mapping value categories to correct values and source files.

**Strengths:**
- Works with existing hook infrastructure
- Catches errors before they reach the filesystem
- No false negatives for registered values

**Weaknesses:**
- Pure error correction -- does not change underlying behavior
- Requires maintaining a registry of risky values (manual upkeep)
- Agent already spent tokens composing the wrong content
- Does not generalize to new value categories until they are registered

### 2. Proactive: Induce uncertainty via rules

Rules that say "never generate from memory, always look up" for infrastructure values. E.g., "Before writing any SLURM parameter, Read the existing sbatch scripts in the project."

**Strengths:**
- Zero implementation cost
- Addresses root cause (overconfidence)

**Weaknesses:**
- Suffers the same paradox: the agent must attend to the rule, but confidence from priors causes it to skip the rule
- Unclear how much weight rules carry vs training priors
- Adding more rules dilutes attention to existing rules
- No enforcement mechanism

### 3. Structural: Citation/source requirements before Edit/Write

Agent must produce a "sources" block before any Edit/Write, listing the files it consulted for each value in the edit. A validation hook checks that the agent actually Read those files recently (cross-referencing the session read cache from smart-context.sh).

```
Agent: I will edit executor.py. Sources:
  - partition_cpu: read from scripts/train.sbatch (line 8)
  - account: read from scripts/train.sbatch (line 5)
[Edit tool call follows]
```

Hook validates: did the agent actually Read `scripts/train.sbatch` in this session? If not, block.

**Strengths:**
- Forces a lookup cycle before generation
- Makes the agent's reasoning auditable
- Uses existing session cache infrastructure

**Weaknesses:**
- Requires structured output format that the agent may not reliably produce
- Adds friction to every Edit/Write, including ones where values are correct
- Agent could cite a file it read but still generate from priors for specific values within it
- No current hook point fires before the agent starts composing Edit content

### 4. Codebase design: Make confabulation impossible

All infrastructure values live in one constants file. All scripts import/reference that file. The agent cannot confabulate because there is no inline value to generate -- it must use the import.

```python
# config/constants.py
SLURM_PARTITION_CPU = "cpu"
SLURM_PARTITION_GPU = "gpu"
SLURM_ACCOUNT = "PAS1266"

# scripts/train.sbatch references {{ SLURM_PARTITION_GPU }}
# executor.py does: from config.constants import SLURM_PARTITION_CPU
```

**Strengths:**
- Eliminates the problem at the source for code files
- Single source of truth benefits humans too
- Agent must Read the constants file or get an import error

**Weaknesses:**
- Only works for code files, not plans, docs, or new files
- Puts burden on codebase design to work around agent limitations
- Not all values are centralizable (e.g., paths computed at runtime)
- Does not help when writing new projects from scratch

### 5. Pre-composition context injection

A hook that fires BEFORE the agent starts composing Edit/Write content, injecting relevant infrastructure values proactively. This would require hooking into something the agent does earlier in its workflow -- e.g., when it reads a file it is about to edit, inject a sidebar of "values you will need."

**Strengths:**
- Addresses root cause at the right time
- Could use existing Read hook (smart-context.sh) to inject values when the agent reads a file it is likely to edit

**Weaknesses:**
- No reliable signal for "about to edit" vs "just reading"
- If the agent skips the Read step (edits from memory), back to the paradox
- Increases context size for every Read, even informational ones

## Comparison Matrix

| Approach | Catches errors | Changes behavior | Maintenance | Generality | Hook point exists |
|----------|---------------|-----------------|-------------|------------|-------------------|
| 1. Edit/Write validation | Yes | No | High (registry) | Registered values only | Yes |
| 2. Rules | No enforcement | Maybe | Low | All values | N/A |
| 3. Citation requirement | Yes | Partially | Medium | All values | No (needs new format) |
| 4. Codebase constants | N/A (prevents) | Yes (structural) | Medium | Code files only | N/A |
| 5. Pre-composition inject | Yes | Yes | Medium | All values | Partial (Read hook) |

## Open Questions

1. **Path of least resistance:** How do you make tool use the default behavior instead of weight-based generation? The agent chooses generation because it is faster (no tool call round-trip). Can you make generation feel slower or riskier?

2. **Intermediate lookup steps:** Can you structurally require them? The citation approach (3) tries this but lacks enforcement. A "plan then execute" protocol might work -- agent must produce an explicit plan with source references, validated by hook, before any multi-edit session.

3. **Reducing false certainty:** Is there a way to make the agent uncertain about specific value categories? Training-time interventions are out of scope, but prompt-based uncertainty calibration might help for known-risky categories.

4. **Codebase conforming to agent limitations:** Should it? Constants files (approach 4) have independent benefits for maintainability. But designing codebases primarily for agent consumption is a new and possibly misguided pattern.

5. **Reactive "good enough":** The block-and-retry path (approach 1) catches errors reliably for registered values. If the registry is maintained, is the proactive path worth the complexity? The agent wastes tokens on wrong compositions, but the output is correct.

6. **Measurement:** How would you measure whether a nudging intervention actually changes behavior vs just catching more errors? Need to distinguish "agent looked it up" from "agent generated correctly by chance" from "agent generated wrong and got blocked."

## Recommended Next Steps

1. **Immediate (low effort):** Implement approach 1 -- Edit/Write validation hook with a registry of known-risky values. Start with SLURM parameters and module versions. This stops the bleeding.

2. **Short-term:** Implement approach 4 where it makes sense -- centralize SLURM constants in KD-GAT. This has independent maintainability benefits.

3. **Medium-term:** Instrument approach 1 to measure how often it fires. If the block rate is high for specific value categories, that data informs which categories need stronger nudging.

4. **Research:** Evaluate whether approach 3 (citation requirements) can be enforced via the existing hook system. The key question is whether there is a hook point that fires after the agent decides to edit but before it composes the edit content.

## Context

- Emerged from a concrete bug in KD-GAT `executor.py` (partition "serial" vs "cpu")
- Related to prior work on tool routing optimization (`~/plans/tool-routing-optimization.md`)
- Existing hook infrastructure: `smart-context.sh` (Read enrichment), `gitnexus-hook.cjs` (search enrichment), auto-format hooks (Edit/Write)
- The "Codified Context" paper (arXiv 2602.20478) studies a similar pattern but focuses on context injection, not the upstream problem of tool use avoidance
- "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026) suggests small prompt changes can significantly shift agent behavior -- but the nudge must be attended to, which circles back to the paradox
