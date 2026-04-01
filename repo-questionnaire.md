# Repo Questionnaire

Answer these questions in a future session to define what this repo is and isn't. Write your answers inline — they'll be used to update the README and set structural constraints.

---

## 1. Identity

**What is this repo?** Pick one primary identity, or rank them.

- [ ] A **lab notebook** — append-only record of observations, dated, never edited after the fact
- [ ] A **knowledge base** — deduplicated, evolving, entries updated as understanding changes
- [ ] A **paper pipeline** — evidence accumulation toward a specific publication
- [ ] A **methods repository** — reusable experimental designs and analysis templates
- [ ] Something else: _______________

**Why does the answer matter?** A lab notebook never deletes; a knowledge base deduplicates aggressively; a paper pipeline has a target venue and deadline; a methods repo prioritizes reproducibility over narrative. The workflow for each is different and they conflict if mixed.

---

## 2. Audience

**Who reads this besides you?**

- [ ] Only me — personal reference
- [ ] Lab members — shared context for the MSL group
- [ ] Public/academic — intended for external readers (conference, blog, arxiv)
- [ ] Future-me — primarily a record for sessions 3+ months from now

**If public/academic:** Is there a target venue, format, or deadline?

> Answer: _______________

---

## 3. Relationship to Other Repos

**How does this relate to your other projects?**

- `~/dotfiles/` contains the actual hook implementations. Should this repo reference them, duplicate them, or stay independent?

> Answer: _______________

- `~/plans/` contains cross-project decision docs. Some overlap with `analysis/`. Should plans about agent behavior live here or there?

> Answer: _______________

- `~/.claude/` contains the logging pipeline (tool-usage.jsonl, session-end.sh, usage_db.py). Should analysis scripts that query this data live in this repo or in `osc-usage`?

> Answer: _______________

---

## 4. Lifecycle of a Document

**When you write a new observation, what should happen to it over time?**

- [ ] It stays as-is forever (lab notebook model)
- [ ] It gets compacted and the original is archived (current approach)
- [ ] It gets merged into a living document per topic (knowledge base model)
- [ ] It feeds into a structured dataset and the prose is secondary

**When an observation is proven wrong by later evidence, what happens?**

- [ ] Mark it as superseded but keep it (historical record)
- [ ] Delete or rewrite it (knowledge base stays current)
- [ ] Add a correction alongside it (academic errata model)

---

## 5. Analysis Artifacts

**The `analysis/` directory currently contains data-mined outputs (tag catalogs, co-occurrence, RQs). Should these be:**

- [ ] Regenerated from scratch each time new documents are added (reproducible pipeline)
- [ ] Manually curated and updated incrementally (living analysis)
- [ ] Versioned snapshots with dates (periodic reports)

**Should the analysis tooling (the Python/SQL used to generate catalogs) live in this repo?**

- [ ] Yes — `scripts/` directory with the analysis code
- [ ] No — it's ad-hoc, run in Claude sessions, doesn't need to be preserved
- [ ] Only if it becomes a reusable pipeline

---

## 6. Scope Boundaries

**Which of these belong in this repo? Check all that apply.**

- [ ] Observations about Claude Code behavior (current core)
- [ ] Observations about other LLM agents (Cursor, Copilot, Aider)
- [ ] Hook design rationale and evaluation
- [ ] Session analytics and dashboards
- [ ] Literature reviews of related papers
- [ ] Experimental results (data tables, plots)
- [ ] Draft paper sections

**What explicitly does NOT belong here?**

> Answer: _______________

---

## 7. Naming and Structure

**Current structure is `original/`, `compacted/`, `analysis/`, `methodology/`. Is this the right decomposition?**

- [ ] Yes, keep it
- [ ] No — I'd prefer: _______________

**Should documents be named by topic (current) or by date?**

- [ ] Topic (e.g., `context-momentum-drift.md`)
- [ ] Date-prefixed (e.g., `2026-03-18-context-momentum-drift.md`)
- [ ] Both (date prefix + topic suffix)

---

## 8. Quality Bar

**What's the minimum bar for a new document to enter `original/`?**

- [ ] Any observation worth writing down (low bar, high volume)
- [ ] Must contain at least one non-obvious finding (medium bar)
- [ ] Must contain empirical evidence — numbers, logs, or reproduction steps (high bar)

**Should compacted versions be human-reviewed before they replace/supplement originals?**

- [ ] Yes — I review each one
- [ ] No — the compaction process is trusted
- [ ] Only spot-check a sample

---

## 9. Integration with Logging Pipeline

**You have 408 sessions of data in DuckDB. Should this repo contain:**

- [ ] Only prose analysis (queries run ad-hoc, results described in text)
- [ ] Saved query results (CSV/tables committed alongside prose)
- [ ] The queries themselves (reproducible analysis)
- [ ] A full pipeline (ingest → analyze → report)

---

## 10. One-Year Test

**Imagine it's March 2027. You open this repo. What do you hope to find?**

> Answer: _______________

**What would make you regret not having structured it differently?**

> Answer: _______________
