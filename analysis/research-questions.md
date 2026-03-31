# LLM Agent Behavior — Research Questions & Experimental Designs

Generated 2026-03-31 from cross-document analysis of 9 compacted research notes.

## Methodology

### Data Sources

1. **9 research documents** in `llm-agent-behavior/` covering hooks, nudging, enrichment, tool routing, context management, plan compliance, and task delegation — written over ~5 weeks of daily Claude Code usage.
2. **hooks.md** — canonical inventory of 18 active hooks with behavior types, introduction dates, and empirical findings from 273 sessions.
3. **Session logging pipeline** — DuckDB database (`~/.cache/claude-usage/usage.db`) with 408 sessions, 90k tool calls, 166 LLM-generated quality facets, and 5.9k hook latency records. Date range: 2026-02-20 through 2026-03-31.

### Analysis Pipeline

**Step 1: Compaction.** Each of the 9 source documents was independently compacted by a dedicated subagent into Core Findings / Key Insights / Takeaways structure, reducing to ~30-50% of original length while preserving specific numbers and thresholds.

**Step 2: Cataloging.** Each compacted document was independently processed by a dedicated subagent that decomposed it into atomic claims (one point = one finding, insight, or recommendation). Each point was assigned 1-3 category tags from a controlled vocabulary of 26 tags. Total: 232 tagged points.

**Step 3: Structural analysis.** The 232-point catalog was analyzed for:
- **Tag frequency** — which categories dominate the corpus (hooks: 62 points, agent-behavior: 42, measurement: 37)
- **Tag co-occurrence** — which categories appear together on the same point (top: enforcement+hooks at 14, hooks+measurement at 8)
- **Cross-source convergence** — which tags appear in 6+ of 9 sources (measurement: 9/9, agent-behavior: 8/9, hooks: 8/9)
- **Tag gaps** — which expected tag pairs never co-occur, revealing blind spots (agent-behavior+measurement: 0, enforcement+measurement: 0)
- **Point type classification** — heuristic labeling as empirical (15%), prescriptive (31%), or interpretive (54%)

**Step 4: Pattern identification.** Three patterns emerged from the intersection of structural analysis results and available logging infrastructure:
- Pattern 1 surfaced from the tag gap analysis: `measurement` is universal but never connects to behavioral tags
- Pattern 2 surfaced from co-occurrence + hooks.md findings: correction-resistance claims cite the same friction-vs-length data but nobody fitted the curve
- Pattern 3 surfaced from co-occurrence concentration: `enrichment+measurement` is the strongest thematic pairing but every point admits the measurement is missing

**Step 5: Grounding.** Each pattern was checked against the actual DuckDB schema (12 tables), hook inventory (18 hooks with known introduction dates), and logging coverage (408 sessions, 166 facets, 90k tool calls) to determine whether the research question is answerable with existing data or requires new instrumentation.

---

## RQ1: Do hard-block hooks improve session outcomes compared to advisory hooks?

### Origin

Tag gap analysis revealed that `enforcement + measurement` never co-occur across 232 points — enforcement is discussed extensively (25 points, 6 sources) and measurement is discussed everywhere (37 points, 9 sources), but no point actually measures enforcement effectiveness. hooks.md states "Hard blocks work" based on anecdotal evidence ("zero ambiguity"), not statistical comparison. The `enforcement + hooks` co-occurrence is the strongest in the corpus (14 points), making this the most-discussed-but-least-validated claim.

### Variables

- **IV:** Hook behavior type (hard-block vs. advisory vs. silent) — already categorized in hooks.md and inferable from hook_latency records by tool/pattern
- **DV:** Session outcome (`facets.outcome`: fully/mostly/partially/not achieved), friction (`facets.friction_detail`), tokens consumed (`sessions.input_tokens + output_tokens`)

### Design

Natural experiment. Each hook has a known introduction date (hooks.md `Introduced` column). Compare session quality distributions *before vs. after* each hook's introduction, controlling for project (`sessions.project`) and session length (`sessions.duration_min`, tool call count). No intervention needed — the data already exists across 408 sessions.

### Power & Constraints

- 408 sessions total, 166 with quality facets (~41% coverage)
- Focus on projects with sufficient n: home (213 sessions), KD-GAT (94), Map-Visualizations (63)
- Confounders: user skill growth over time, task difficulty variation, multiple hooks introduced in same period
- Can only detect large effects (>15% outcome shift) given sample size

### Data Query Path

`sessions` JOIN `facets` ON session_id, filtered by `start_time` relative to hook introduction dates. Group by before/after × project. Compare outcome distributions.

---

## RQ2: What is the functional relationship between session length and agent compliance?

### Origin

`correction-resistance + nudging` co-occurs on 5 points across the corpus. Multiple documents independently claim that corrections decay with context volume, and hooks.md provides the only quantitative anchor: 8% high-friction at 0-100 tool calls, 62% at 400-600. But nobody has fitted the actual curve, identified the inflection point, or determined whether friction is driven by tool call count, token volume, time, or behavioral diversity loss. The `context-window` tag (24 points, 7 sources) and `session-management` tag (8 points) both reference session length as a risk factor without providing a threshold.

### Variables

- **IV:** Session length operationalized four ways:
  - Tool call count (from `tool_calls` grouped by session)
  - Total tokens (`sessions.input_tokens + output_tokens`)
  - Wall-clock duration (`sessions.duration_min`)
  - User message count (`sessions.user_messages`)
- **DV:**
  - User interruptions (`sessions.user_interruptions`)
  - Outcome quality (`facets.outcome` ordinal-encoded)
  - Behavioral diversity: Shannon entropy over tool-type distribution per session
  - Friction events (parsed from `facets.friction_detail`)

### Design

Observational / correlational. Query `sessions` joined to `facets` and `tool_calls`. For each session compute: total calls, tool diversity index, interruption rate, and outcome. Bin by tool-call count (0-50, 50-100, 100-200, 200-400, 400+) and fit a logistic curve to the friction probability (P(outcome ∈ {partially, not_achieved}) ~ f(tool_calls)).

### Actionable Output

An empirical threshold for "suggest session break" — the tool-call count where friction probability exceeds 50%. Currently the session-break recommendation in the corpus is asserted without a number. This threshold would parameterize a PostToolUse hook that counts calls and injects a session-break suggestion.

### Power & Constraints

- 408 sessions provide reasonable coverage across the IV range
- Only 166 have facet ratings — the outcome DV is limited to this subset
- User interruptions (available for all 408) serve as a proxy DV with full coverage
- Primary confounder: task complexity (harder tasks are both longer AND more friction-prone)
- Mitigation: include `files_modified` and `lines_added` as proxy controls for task scope

### Data Query Path

`sessions` LEFT JOIN `facets` ON session_id. `tool_calls` GROUP BY session_id for per-session tool counts and tool-type entropy. Logistic regression or binned proportion plot.

---

## RQ3: Does per-tool-call graph enrichment change agent file-access patterns or just add tokens?

### Origin

`enrichment + measurement` co-occurs on 6 points — the strongest thematic pairing proportional to tag size. But every one of those 6 points *acknowledges the measurement doesn't exist yet*. The enrichment-ab-testing document reports 826 injections over 10 days with "zero measurement of whether any changed agent decisions." It also reports outcome quality degraded from 80% good (W09) to 65% good (W11) after adding enrichment, but attributes this to confounders. The document designed a 4-config A/B test but hasn't run it. Meanwhile, `hook_latency` already tracks `enrichment_chars` and `elapsed_ms` per call (5,904 records), providing the cost side of the equation — only the benefit side is missing.

### Variables

- **IV:** `CLAUDE_ENRICHMENT_CONFIG` environment variable:
  - A = Full enrichment (PageRank + GitNexus-on-Read + GitNexus-on-Search/Edit + read-count blocking)
  - B = Blocking-only (read-count blocking, no context injection)
  - C = Search-enrichment-only (GitNexus on Grep/Glob/Edit, no Read enrichment)
  - D = Read-enrichment-only (GitNexus on Read, no search enrichment)
- **DV:**
  - Re-read rate (from `smart-context.sh` logs at `/tmp/claude-reads-{session}.jsonl`)
  - Enrichment tokens injected per session (from `hook_latency.enrichment_chars`)
  - Unique files accessed per session (from `tool_calls` WHERE tool = 'Read')
  - Session outcome (`facets.outcome`, `facets.helpfulness`)
  - Token efficiency: outcome quality / total tokens consumed

### Design

Randomized experiment. Set `CLAUDE_ENRICHMENT_CONFIG` per session via `.bashrc` randomization or daily rotation. Log active config in `session-end.sh` metadata so it flows into `usage.db`. Run for ~4 weeks (~50 sessions per config at ~8 sessions/day).

### Key Test

If Config B (blocking-only) has equivalent outcomes to Config A (full enrichment), the enrichment pipeline can be stripped, saving ~500-700ms per call on gitnexus-hook.cjs. This directly addresses the corpus's strongest open question.

### Power & Constraints

- ~8 sessions/day × 28 days = ~224 sessions, ~56 per config
- Detectable effect size: >15% difference in outcome rate (from enrichment-ab-testing analysis)
- Cannot detect small effects — accept this limitation upfront
- Dominant confounder: task difficulty varies day-to-day and cannot be controlled
- Mitigation: randomize (not rotate) config per session to decorrelate from daily task patterns

### New Instrumentation Required

1. Add `CLAUDE_ENRICHMENT_CONFIG` randomization to session startup
2. Modify `gitnexus-hook.cjs` to read the env var and conditionally skip enrichment layers
3. Log the active config value in `session-end.sh` output (one new field)

---

## Cross-Cutting Observation: The Behavior–Measurement Disconnect

The tag gap analysis reveals that `agent-behavior + measurement` **never co-occur** across all 232 points. This is the single largest blind spot in the corpus: 42 points describe what agents do wrong in rich qualitative detail, 37 points describe how to measure things, and zero points connect the two.

The data to close this gap already exists in the logging pipeline — it's a query problem, not a collection problem. All three RQs above are designed to bridge this disconnect by connecting behavioral observations (from the corpus) to measurable outcomes (from the DuckDB pipeline).

### Evidence for the disconnect

| Behavioral claim (from corpus) | Measurable proxy (from DuckDB) | Currently measured? |
|-------------------------------|-------------------------------|-------------------|
| "Corrections decay with context volume" | user_interruptions ~ tool_call_count | No |
| "Hard blocks work" | outcome before/after hook introduction | No |
| "Enrichment may be counterproductive" | outcome ~ enrichment_config | No |
| "~40% Bash calls duplicate structured tools" | Bash grep/cat/ls count / total Bash | Yes (in tool_calls) |
| "Advisory hooks have diminishing returns" | friction_detail correlation with advisory hook count | No |
| "Session length is a risk factor" | P(poor outcome) ~ duration/calls | Partially (hooks.md cites 8%/62%) |

### Recommended priority

1. **RQ2 (correction decay curve)** — purely observational, zero new infrastructure, highest immediate utility (parameterizes session-break hook)
2. **RQ1 (hook effectiveness)** — natural experiment, zero new infrastructure, validates or invalidates the most-discussed claim in the corpus
3. **RQ3 (enrichment A/B)** — requires modest instrumentation, answers the most expensive open question (gitnexus-hook adds 500-700ms/call)
