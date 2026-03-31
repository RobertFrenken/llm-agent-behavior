## Core Findings

- 826 enrichment injections fired over 10 days with **zero measurement** of whether any changed agent decisions. Hooks log execution but not behavioral impact.
- PageRank weight injection may be **counterproductive** -- it reinforces existing read patterns ("file is important because you read it a lot") instead of informing structural decisions. The memory nudge encourages memorizing current state rather than questioning it.
- Outcome quality **degraded** after adding enrichment infrastructure: W09 80% good outcomes -> W11 65% good, though confounders (longer sessions, harder tasks) prevent attribution.
- External evidence confirms context injections affect LLM behavior (GitClear, METR, ICLR 2026 nudging paper), but direction of effect is unknown for this system.
- Detectable effect size is limited to **>15%** differences given ~8 sessions/day from a single user. METR needed 246 issues across 16 developers to detect a 19% effect.
- Self-assessed outcomes (facets) may be unreliable -- METR found developers misjudged AI helpfulness by ~40%.

## Key Insights

- **Read-count blocking is the only component with demonstrated value** -- it blocked wasteful re-reads during a live cleanup session (2026-03-20). All four experimental configs keep it enabled.
- The experiment decomposes enrichment into **four independently testable components**: PageRank injection, GitNexus-on-Read, GitNexus-on-Search/Edit, and read-count blocking. This isolates which (if any) enrichment layer actually helps.
- **Payload caps (head_limit) should stay on regardless of enrichment config** -- they prevent token waste independent of whether enrichment content has decision value.
- The "enrichment might hurt" hypothesis is plausible: bloated tool context is a known failure mode (Anthropic context engineering guide), and reinforcing existing patterns could suppress exploration of unfamiliar but important files.

## Takeaways

- **Run the 4-config A/B test** (Full / Blocking-only / Search-enrichment-only / Read-enrichment-only) using a single env var `CLAUDE_ENRICHMENT_CONFIG=A|B|C|D`, rotated daily or randomized per session.
- **Log the active config in session-end metadata** so it flows into `usage.db` for analysis.
- **Need ~50 sessions per config (~4 weeks)** to reach minimum statistical power; measure `good_rate`, `wrong_approach` friction count, and `avg_tokens` per config.
- **Accept this will only catch large effects** -- small improvements/degradations will be invisible at this sample size. Task difficulty is the dominant confounder and cannot be fully controlled.
- **Default to "blocking only" (Config B) if no config shows significant advantage** -- it has the lowest injection overhead and the only proven-valuable component.
