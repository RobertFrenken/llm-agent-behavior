# RQ2: Correction Decay Curve — Step-by-Step Methodology

## Research Question

At what point in a session does agent compliance degrade, and what is the shape of that degradation?

## Why This First

This is the only RQ that requires zero new instrumentation. The data already exists across 408 sessions in usage.db. The output — an empirical friction threshold — immediately informs whether a session-break hook is worth building and what its trigger value should be.

---

## Step 1: Define the Population

Pull all sessions from usage.db that have both quantitative metadata (tool call counts, token totals, duration, interruptions) and qualitative facets (outcome, helpfulness, friction_detail). This is the intersection of the `sessions` and `facets` tables — approximately 166 sessions based on current coverage.

Record how many sessions fall into each project. If any project has fewer than 15 sessions with facets, note it but don't exclude it yet — the initial analysis should be pooled across projects.

## Step 2: Operationalize "Session Length"

Session length is not a single number. Compute four candidate measures for each session:

- **Tool call count**: total rows in `tool_calls` for that session_id
- **Token volume**: sum of input_tokens and output_tokens from `sessions`
- **Wall-clock duration**: duration_min from `sessions`
- **User message count**: user_messages from `sessions`

These will be compared later to determine which is the best predictor. Do not commit to one measure upfront.

## Step 3: Operationalize "Compliance Degradation"

Define two outcome variables:

**Binary friction indicator**: Did the session have a poor outcome? Encode `facets.outcome` as: fully_achieved and mostly_achieved = 0 (good), partially_achieved and not_achieved = 1 (friction). This gives a binary variable suitable for logistic regression.

**Continuous proxy**: `sessions.user_interruptions` divided by `sessions.user_messages`. This is an interruption rate — how often the user had to intervene relative to how much they were participating. Available for all 408 sessions, not just the 166 with facets.

## Step 4: Exploratory Binning

Before fitting any curve, look at the raw data. Bin sessions by tool call count into 5 groups:

- 0–50 calls
- 51–100 calls
- 101–200 calls
- 201–400 calls
- 401+ calls

For each bin, compute:
- Number of sessions
- Proportion with friction (binary indicator = 1)
- Mean interruption rate
- Mean outcome score (ordinal: fully=4, mostly=3, partially=2, not=1)

This produces a table that should visually confirm or contradict the hooks.md claim (8% friction at 0-100, 62% at 400-600). If the pattern doesn't hold, stop here and investigate why before fitting models.

## Step 5: Check for Confounders

Before attributing friction to session length, check whether longer sessions are simply harder tasks. For each bin, also compute:

- Mean files_modified (proxy for task scope)
- Mean lines_added + lines_removed (proxy for task complexity)
- Project distribution (are long sessions concentrated in one project?)
- Mean assistant_messages (does the agent talk more in long sessions, or does the user drive more?)

If long sessions are systematically associated with larger tasks or specific projects, note this as a confounder. It does not invalidate the analysis, but it changes interpretation from "length causes friction" to "length and complexity jointly predict friction."

## Step 6: Compute Behavioral Diversity

For each session, compute the tool-type distribution: what fraction of tool calls were Read, Edit, Bash, Grep, Glob, Write, Agent, etc. From this distribution, compute Shannon entropy — a single number measuring how varied the agent's tool usage was.

Compare entropy across the bins from Step 4. The context-momentum-drift document predicts that long sessions will show lower diversity (the agent gets stuck in a rut). If entropy decreases with session length, that's supporting evidence for the momentum hypothesis.

## Step 7: Fit the Curve

Using the binary friction indicator from Step 3 and tool call count from Step 2, fit a logistic regression: P(friction) = logistic(β₀ + β₁ × tool_calls).

From this model, extract the inflection point — the tool call count where P(friction) = 0.50. This is the candidate threshold for a session-break recommendation.

Repeat with the other three length measures (tokens, duration, user messages) as the predictor. Compare model fit (AIC or log-likelihood) to determine which measure best predicts friction.

## Step 8: Validate with the Continuous Proxy

Repeat Steps 4–7 using the interruption rate (available for all 408 sessions) instead of the binary friction indicator (available for 166). If the two analyses agree on the general shape and threshold range, confidence increases. If they disagree, investigate which outcome measure is more reliable.

## Step 9: Check for Project-Specific Effects

Re-run the logistic regression with a project indicator variable. Does the friction threshold differ by project? KD-GAT sessions involve GPU training and complex configs — they may hit friction earlier. Home sessions are lighter coordination tasks — they may tolerate more length.

If project matters, report project-specific thresholds rather than a single global number.

## Step 10: Interpret and Decide

At this point you have:

1. A binned frequency table showing friction rates by session length
2. A fitted curve with a specific inflection point
3. A comparison of four length measures showing which best predicts friction
4. A behavioral diversity analysis showing whether tool entropy decreases with length
5. A confounder check showing whether length or complexity drives friction
6. A project-specific analysis showing whether one threshold fits all

The decision is: should a PostToolUse hook count tool calls and inject a session-break suggestion? If so, at what count? The answer comes directly from the inflection point, adjusted for the confounder analysis.

If the confounder analysis shows complexity dominates over length, the hook should factor in task scope (files_modified) rather than raw call count. If behavioral diversity drops before friction rises, the diversity metric is an earlier warning signal than the call count.

## What This Does NOT Answer

- Whether session breaks actually reduce friction (that requires an intervention study)
- Whether the friction is caused by context window saturation, model fatigue, or user fatigue
- Whether different models (Opus vs. Sonnet vs. Haiku) have different decay curves
- Whether the curve has shifted over time as hooks were added

These are follow-up questions, not prerequisites. Get the baseline curve first.
