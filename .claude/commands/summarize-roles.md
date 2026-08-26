# /summarize-roles - Aggregate Prep Signal Across High-Fit Roles

`/search-roles` finds and classifies roles. `/summarize-roles` is the next step: it reads the full JD of every `strong_fit` posting already found, extracts candidate-prep signal from each with `.claude/skills/jd-insights-extractor/SKILL.md`, and aggregates across all of them into one **HTML report** - technical and soft skills to prepare, quantitative frequency stats, and named projects/techniques worth reading or building toward.

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

## Step 5: Write the Report (HTML)

Write a **self-contained HTML file** to `prep-briefs/<scope>-<YYYY-MM-DD>.html` (`<scope>` is the company name, kebab-cased, or `all-companies`), creating the directory if needed. Self-contained means no external stylesheets, fonts, or scripts - everything inline in one file, so it opens correctly straight from disk or from git.

Structure:

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Prep Brief: <scope> - <date></title>
<style>
  /* Inline CSS only. Support both light and dark viewing:
     @media (prefers-color-scheme: dark) { ... } alongside a light default.
     Keep it plain: system font stack, a max-width content column, simple
     table styling with borders and zebra striping, generous spacing.
     No JS. */
</style>
</head>
<body>
  <h1>Prep Brief: <scope></h1>
  <p class="meta">Generated <date> - aggregated from N strong-fit roles across <company list>. <Note --include-adjacent if used, and any roles dropped in Step 2.></p>

  <h2>Quantitative Insights</h2>
  <ul><!-- the frequency sentences from Step 4.4, e.g. "3 of 4 high-fit JDs (75%) mention speculative decoding" --></ul>

  <h2>Notable Projects, Papers, and Techniques to Read or Reference</h2>
  <!-- Split into three subsections by kind, not lumped into one table:
       - Concepts: a named technique/idea with no single canonical repo to send someone to
         (e.g. "speculative decoding", "Flash Attention", "disaggregated prefill/decode").
         Columns: Concept | Category note (e.g. "inference optimization technique",
         "GPU networking protocol") | Mentioned By (role title, linked to its posting URL).
       - Repositories: a named open-source project with an actual repo. Columns: Repository
         (linked to its real repo URL - verify the URL resolves before writing it, never guess) |
         Category note (e.g. "serving framework", "GPU kernel library", "training framework",
         "workflow orchestration") | Mentioned By (role title, linked to its posting URL).
       - Blogs: the target company's own engineering/research blog posts or case studies
         referenced in a JD (e.g. a "such-and-such achieved 2x faster inference with X" post).
         Columns: Post title (linked to the actual blog URL - pull it from the JD's own hyperlinks
         or embedded text, never fabricate one) | Category note (e.g. "case study: inference
         performance", "product announcement") | Mentioned By (role title, linked to its posting URL). -->
  <h3>Concepts</h3>
  <table><!-- Concept | Category note | Mentioned By --></table>
  <h3>Repositories</h3>
  <table><!-- Repository (linked) | Category note | Mentioned By --></table>
  <h3>Blogs</h3>
  <table><!-- Post (linked) | Category note | Mentioned By --></table>

  <h2>Technical Skills to Prepare</h2>
  <table><!-- columns: Skill (candidate-prep phrasing) | Mentioned In (count/%) | Companies -->
  </table>

  <h2>Soft Skills to Prepare</h2>
  <table><!-- columns: Skill (candidate-prep phrasing) | Mentioned In (count) | Companies -->
  </table>

  <h2>Source Roles</h2>
  <table><!-- columns: Title | Company | Verdict | Link (<a href> to the posting) - full traceability -->
  </table>
</body>
</html>
```

Order sections quantitative-insights-and-projects first, then the skill tables - the frequency stats and named projects are the highest-value, least-obvious output and shouldn't be buried under two skill tables first.

This file is a dated snapshot, not live state - it will go stale as the pool grows or JDs change, which is exactly why it's dated and disposable to regenerate rather than edited in place.

---

## Step 6: Present

Show the user the same content as the written file, condensed as chat-readable text/tables (not the raw HTML source) - lead with the quantitative insights and named projects, then the ranked skill lists. Tell them where the full HTML report was saved and that it can be opened directly in a browser.

---

## Important Rules

1. This command classifies nothing new - it only works from roles `/search-roles` already verdicted as `strong_fit`/`adjacent`. If the pool is empty, tell the user to run `/search-roles` first.
2. Extraction is candidate-prep framing, never a bare relabeling of the JD's requirements list - see the framing rule in `.claude/skills/jd-insights-extractor/SKILL.md`.
3. A named project/technique is recorded with its context (what the company is doing with it), never just the bare name.
4. Quantitative stats always state their denominator ("3 of 4", not just "75%") so the reader can judge how much to trust a small sample.
5. The JD cache (`scrape_state/jd_cache/`) is local, disposable state - like the rest of `scrape_state/`, it is gitignored and never committed.
