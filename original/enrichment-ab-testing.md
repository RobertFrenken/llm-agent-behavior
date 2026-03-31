# Enrichment A/B Testing Design

## Problem

826 enrichment injections fired over 10 days with zero measurement of whether any changed a decision. The hooks log that they ran, but nothing observes what Claude did differently because of the injection.

PageRank weight injection may be actively counterproductive — it reinforces existing read patterns ("this file is important because you read it a lot") rather than informing structural decisions. The message "consider keeping key details in memory" nudges toward memorizing current state rather than questioning it.

## Evidence Gap

- GitClear, METR, and ICLR 2026 nudging paper all show context injections affect LLM behavior
- But direction of effect is unknown for this specific system
- Facet data shows outcomes got worse after infrastructure was added (W09: 80% good → W11: 65% good) but confounders are too strong (longer sessions, harder tasks)
- No controlled comparison exists

## Proposed Experiment

### 4 Configurations

| Config | PageRank | Graph on Read | Graph on Search | Read blocking |
|--------|----------|--------------|-----------------|---------------|
| A: Full (current) | on | on | on | on |
| B: Blocking only | off | off | off | on |
| C: Search enrichment only | off | off | on | on |
| D: Read enrichment only | on | on | off | on |

Read-count blocking (2nd advise, 3rd deny) stays on in all configs — demonstrated value in live usage (blocked wasteful re-reads during cleanup session 2026-03-20).

### Components

1. **PageRank weight injection** — `smart-context.sh` lines 138-148. Injects "PageRank weight: X (read Nx across N sessions)" on first read of any file.
2. **GitNexus graph context on Read** — `smart-context.sh` lines 151-177. Queries graph for related nodes on first read of indexed files.
3. **GitNexus enrichment on Search/Edit** — `gitnexus-hook.cjs`. Fires on Grep, Glob, Bash, Write, Edit. Injects graph context + enforces payload caps (head_limit).
4. **Read-count blocking** — `smart-context.sh` lines 180-198. Advisory on 2nd read, deny on 3rd+ of unchanged files.

### Implementation

Single env var checked by both hooks:

```bash
# CLAUDE_ENRICHMENT_CONFIG=A|B|C|D
# Set at session start
```

`smart-context.sh` checks the var to decide whether to run PageRank lookup (steps 1) and GitNexus graph query (step 2). `gitnexus-hook.cjs` checks it to decide whether to inject enrichment (step 3). Payload caps (head_limit) should stay on regardless — they prevent token waste independent of enrichment value.

### Assignment

Rotate by day of week: Mon=A, Tue=B, Wed=C, Thu=D, Fri=A, Sat=B, Sun=C. Or assign randomly per session via `$(( RANDOM % 4 ))` in a SessionStart hook.

Log the active config in `session-end.sh` metadata so it appears in usage.db.

### Measurement

After ~50 sessions per config (~4 weeks at current rate):

```sql
SELECT config,
       COUNT(*) as sessions,
       AVG(CASE WHEN outcome IN ('fully_achieved','mostly_achieved') THEN 1.0 ELSE 0.0 END) as good_rate,
       SUM(json_extract(friction_counts, '$.wrong_approach')) as wrong_approach,
       AVG(total_tokens) as avg_tokens
FROM sessions s
JOIN facets f ON s.session_id = f.session_id
JOIN session_metadata m ON s.session_id = m.session_id
GROUP BY config
```

### Caveats

- Sample sizes are small (one user, ~8 sessions/day)
- Task difficulty varies and is the dominant confounder
- METR study needed 246 issues across 16 developers to detect a 19% effect
- This will only detect large effects (>15% difference in good_rate)
- Self-assessment of outcomes (facets) may be unreliable (METR found developers misjudged AI helpfulness by ~40%)

## References

- "LLM Agents Are Hypersensitive to Nudges" (ICLR 2026, arXiv 2505.11584)
- METR RCT: 19% slowdown, 40% perception gap (arXiv 2507.09089)
- GitClear 2025: 211M lines, refactoring halved, duplication 8x (gitclear.com)
- Anthropic context engineering guide: bloated tool sets as failure mode
