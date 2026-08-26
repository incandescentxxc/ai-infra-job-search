# /summarize-roles - Aggregate Prep Signal Across High-Fit Roles

`/search-roles` finds and classifies roles. `/summarize-roles` is the next step: it reads the full JD of every `strong_fit` posting already found, extracts candidate-prep signal from each with `.claude/skills/jd-insights-extractor/SKILL.md`, and aggregates across all of them into one report - technical and soft skills to prepare, quantitative frequency stats, and named projects/techniques worth reading or building toward.

This command never searches for new roles - it only works with what `/search-roles` has already classified in `scrape_state/seen_roles.json`.

`$ARGUMENTS` may contain: a company name (scope to that company's `strong_fit` roles), nothing (`all` tracked companies), or `--include-adjacent` (widen the pool to `strong_fit` + `adjacent` - useful when `strong_fit` alone is too small a sample).

---

## Step 1: Select the Pool

1. Read `scrape_state/seen_roles.json`. Select entries with `status: "triaged"` and `verdict: "strong_fit"` (plus `verdict: "adjacent"` if `--include-adjacent` was passed), scoped to the named company if one was given.
2. **Sample-size check:** if the pool has fewer than 3 roles, tell the user the aggregation will be low-confidence (a "mentioned in 1/2 JDs" stat isn't meaningful) and ask whether to proceed, widen with `--include-adjacent`, or run `/search-roles` on more companies first.
3. State the pool size and which companies/roles are included before proceeding.

---

## Step 2: Fetch Full JDs

For each role in the pool, get the full JD text:

1. If `scrape_state/jd_cache/<url-hash>.txt` exists and is less than 14 days old, use the cached text (this command may run repeatedly as the pool grows; no need to re-fetch a JD that hasn't changed). Hash the URL with a stable, short hash (e.g. first 16 hex chars of SHA-256) for the cache filename.
2. Otherwise, fetch per `.claude/skills/web-research/SKILL.md`'s escalation order (WebFetch, then robots-checked curl retry, then WebSearch for a corrected URL if it's a ghost job). Cache the result to `scrape_state/jd_cache/<url-hash>.txt` on success.
3. If a JD genuinely can't be retrieved after the full escalation, drop that role from the pool for this run and note it in the final report rather than silently shrinking the sample size without explanation.

---

## Step 3: Per-JD Extraction

Dispatch parallel `general-purpose` agents via the Agent tool, ~5 JDs per agent (a single agent is fine for ≤5).

- Pass each agent `.claude/skills/jd-insights-extractor/SKILL.md`'s extraction schema and framing rule **inline in the prompt**, plus that agent's slice of already-fetched JD text (from Step 2 - agents should not re-fetch).
- Each agent returns one JSON object per JD, following the schema in that skill file: `url`, `company`, `title`, `technical_skills`, `soft_skills`, `named_projects` (each with `name` + `context`), `technique_keywords`.

---

## Step 4: Aggregate

1. **Technical skills:** cluster near-duplicate phrasings across JDs by underlying concept (e.g. "FP8/INT4 quantization" and "quantization (FP8, INT4)" are the same cluster). For each cluster, count how many distinct JDs mention it and compute the percentage of the pool. Rank by frequency, descending. Keep the candidate-prep phrasing from Step 3 - don't flatten back into recruiter-speak.
2. **Soft skills:** same clustering approach, but soft skills cluster more thematically than technical ones (e.g. "works with customers alongside FDEs" and "partners directly with customer engineering teams" are the same theme even with different wording) - use judgment rather than exact-string matching.
3. **Named projects/techniques:** deduplicate by project/technique name across JDs (keep the name verbatim), merge the `context` from each mention, and record which companies/roles mentioned it. Do not compute a percentage for these - list them with their attribution; the fact that a niche project is mentioned even once by a single high-fit posting is itself useful signal.
4. **Quantitative insight lines:** turn the technical-skill and technique-keyword frequencies into plain sentences, e.g. "3 of 4 high-fit JDs (75%) mention speculative decoding" - these are the headline stats for the report.

---

## Step 5: Write the Report

Write to `prep-briefs/<scope>-<YYYY-MM-DD>.md` (`<scope>` is the company name or `all-companies`), creating the directory if needed:

```markdown
# Prep Brief: <scope> - <date>

Aggregated from N strong-fit roles across <company list>. <Note --include-adjacent if used, and any roles dropped in Step 2.>

## Technical Skills to Prepare
(ranked by frequency, each with its %/count and which companies mentioned it)

## Soft Skills to Prepare
(thematic clusters)

## Quantitative Insights
(the frequency sentences from Step 4.4)

## Notable Projects, Papers, and Techniques to Read or Reference
(name, context, which company/role mentioned it - framed as: this is what to read, cite, or build toward for interview/project-experience prep)

## Source Roles
(table: title, company, URL, verdict - full traceability back to the postings this brief was built from)
```

This file is a dated snapshot, not live state - it will go stale as the pool grows or JDs change, which is exactly why it's dated and disposable to regenerate rather than edited in place.

---

## Step 6: Present

Show the user the same content as the written file, condensed - lead with the quantitative insights and named projects (the highest-value, least-obvious output), then the ranked skill lists. Tell them where the full report was saved.

---

## Important Rules

1. This command classifies nothing new - it only works from roles `/search-roles` already verdicted as `strong_fit`/`adjacent`. If the pool is empty, tell the user to run `/search-roles` first.
2. Extraction is candidate-prep framing, never a bare relabeling of the JD's requirements list - see the framing rule in `.claude/skills/jd-insights-extractor/SKILL.md`.
3. A named project/technique is recorded with its context (what the company is doing with it), never just the bare name.
4. Quantitative stats always state their denominator ("3 of 4", not just "75%") so the reader can judge how much to trust a small sample.
5. The JD cache (`scrape_state/jd_cache/`) is local, disposable state - like the rest of `scrape_state/`, it is gitignored and never committed.
