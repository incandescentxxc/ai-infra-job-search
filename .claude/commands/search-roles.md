# /search-roles - Find and Classify AI/LLM Infra Roles at Tracked Companies

This is the single entry point for this repo: it searches the companies in `company-tracker/`, filters obvious noise cheaply, then reads the full JD of everything that survives and classifies it against `.claude/skills/ai-infra-role-classifier/SKILL.md`. **It only returns classified (triaged) results** - nothing gets presented to the user without a full-JD-backed verdict.

Internally this runs in two phases (a cheap title-level pass, then an expensive full-JD classification pass) purely for cost efficiency - don't skip the cheap phase and read every JD, and don't present the cheap phase's output as if it were a final verdict. Both phases are this command's job; there is no separate command to invoke afterward.

`$ARGUMENTS` may contain a company name (e.g. `/search-roles fireworks`), `all` to run every tracked company, or `health` to run the portal health check only.

---

## Step 0: Load State

1. Read `company-tracker/companies.md` and `company-tracker/track_list.json`.
2. **Reconcile:** any company present in `companies.md` but missing from `track_list.json` gets a new entry (`ats: "unverified"`, `portal_resolution: {"freehire_company_slug": null, "notes": "Not yet resolved."}`, `status: "active"`, `added_date`: today, `added_source: "companies.md"`, `last_scraped_date: null`). Any company in `track_list.json` but removed from `companies.md` gets flagged to the user - ask whether to mark it `paused` or delete it; never silently drop tracked state.
3. Read `scrape_state/seen_roles.json` (create with `{"seen": {}}` if missing).
4. Determine scope: `$ARGUMENTS` names one company -> resolve it against `track_list.json` by name (case-insensitive substring match); `all`/`broad` -> every company with `status: "active"`; nothing -> ask the user which company or `all`.

---

## Step 1: Resolve Each Company to a Portal

**Direct ATS API is the primary path once a company's ATS is known - freehire is a discovery mechanism for companies that aren't resolved yet, not a trusted primary source.** This was a confirmed failure mode: for a company already known to be on Ashby, a freehire company-scoped search returned 17 results against 75 actually-live postings on Ashby's own API - not a date-filter artifact, freehire's index for that company was just badly incomplete. Never treat freehire's result count as the true size of a company's job board once the direct API is available.

For each company in scope, in order of preference:

1. **Known ATS + slug.** If `track_list.json` already records a confirmed `ats` (`greenhouse` or `ashby`) and slug, go straight to the direct API - skip freehire entirely for this company:
   - Greenhouse: `curl -s https://boards-api.greenhouse.io/v1/boards/<slug>/jobs?content=true`
   - Ashby: `curl -s https://api.ashbyhq.com/posting-api/job-board/<slug>`
2. **Unresolved company - use freehire for discovery.** Try `bun run .agents/skills/freehire-search/cli/src/cli.ts search -q "<company name>" --limit 5 --format json` to find a `company_slug` and infer the likely ATS from the `url` field's domain (`job-boards.greenhouse.io` or `jobs.ashbyhq.com`). Once you have a slug and ATS guess, verify it against the direct API (step 1's URLs) before trusting it - if the direct API 404s, fall back to using freehire itself as the search source for this company and note in `track_list.json` that direct-API resolution failed.
3. **Neither works.** If the ATS is something other than Greenhouse/Ashby (Lever, Workday, a custom careers page) and freehire has no data either, use `WebSearch`/`WebFetch` per `.claude/skills/web-research/SKILL.md`. This is the case `/add-portal` exists for - if a company keeps landing here across runs, that's the signal to scaffold it a dedicated CLI instead of repeating ad hoc WebFetch every time.

If a company can't be resolved by any path, say so plainly to the user and move on - don't silently skip it. Write whatever was learned (confirmed ATS + slug, or confirmed freehire incompleteness) back into `track_list.json` regardless of which path worked.

---

## Step 2: Search Each Company

**When using a direct ATS API (Step 1.1):** fetch the full current listing - no date filter. The API only returns currently-open postings, so there is nothing to filter for availability, and these companies commonly run long-lived "evergreen" listings (a role open for a year is not stale, it's just still open) - a recency filter would silently discard real, current postings. Use the `department`/`departments` field from the response as the primary noise filter (see Step 3) instead of a date cutoff.

**When falling back to freehire (Step 1.2/1.3, no direct API available):** apply a **30-day** `--jobage` filter as a bounded compromise - freehire doesn't expose a department field, so without some filter a large company's result set is unmanageable. Known limitation: freehire tags some postings with a bulk re-index timestamp rather than the true posting date, which can still misfile things either direction; if a company's freehire-sourced result count looks suspiciously low, re-run with no `--jobage` filter as a disclosed one-off widening.

Cap large companies (Google, Meta, NVIDIA, AWS) to a keyword-scoped query (e.g. `-q "inference"` or `-q "ML infrastructure"`) rather than pulling the whole company's job board - `track_list.json`'s notes field flags which companies need this, regardless of which path resolved them.

---

## Step 3: Fast Pre-Filter (cheap pass, internal only)

Not the final verdict - a cost-control step before the expensive full-JD read in Step 5.

**Department filter (direct-API path only, apply first).** Ashby's `department` field and Greenhouse's `departments` array classify each posting - drop anything not tagged as an engineering/product/design-type department (Ashby's `EPD` is the common label; Greenhouse varies by company, check the actual values returned). This alone typically removes well over half a large company's listings (GTM, sales, G&A, recruiting) before any title parsing, and it's far more reliable than keyword matching - it's what a title-only pass has no way to know when it excludes "GTM Engineer" one string at a time.

**Title-level filter (apply to what survives the department filter, or to everything on the freehire-only path):**

- **Immediate exclude** on title alone: "Forward Deployed Engineer", "Security Engineer", "GRC Analyst", "Security Operations Lead", any Sales/Marketing/Recruiting/Product Manager/Product Designer/Solutions Architect/Account Executive/Customer Success/Facilities/Accounting title, "Staff Engineer"/"Staff Software Engineer"/"Staff ML Engineer"/"Staff Platform Engineer" (explicit Staff seniority tier - but **not** "Member of Technical Staff", which is a generic IC title, not a Staff tier), "Engineering Manager"/"Manager"/"Head of Engineering" or any people-management title (management track is excluded regardless of the team's domain - see the classifier rubric's pattern 6).
- **Keep for Step 5:** anything with Engineer/Research Engineer/Member of Technical Staff/Platform/Infrastructure/Inference in the title that isn't caught by an exclude rule above. When genuinely unsure, keep it - Step 5's full rubric is the real filter, and **title framing is not reliable in either direction**: "Software Engineer - Dedicated Inference" and "AI Inference Engineer" have both turned out, on full JD read, to be product-engineering and forward-deployed roles respectively despite infra-sounding titles, while "Software Engineer - Model Products" turned out to be core serving infra despite a product-sounding title. Don't let a promising or discouraging title substitute for Step 5.

## Step 4: Ghost-Job Check on Kept Results

Before fetching full JDs in Step 5, do a light-touch URL check per `.claude/skills/web-research/SKILL.md`'s ghost-job section on anything kept in Step 3: if the company's `track_list.json` notes flag it as ATS-migrating, verify each kept URL resolves and cross-reference the live ATS API directly for a corrected URL if any 404. For companies without that flag, spot-check is enough - a full verify only if something looks wrong.

## Step 5: Full-JD Classification (the real filter)

For everything that survived Steps 3-4, dispatch parallel `general-purpose` agents via the Agent tool, ~5 roles per agent (a single agent is fine for ≤5).

- Pass each agent the classifier rubric **inline in the prompt** (the hard-exclude patterns, the calibration table, the verdict scale, from `.claude/skills/ai-infra-role-classifier/SKILL.md`) - do not make agents re-read the SKILL.md file themselves.
- Agents fetch each posting's full JD with WebFetch and classify **only from actually fetched content**, following the escalation order in `.claude/skills/web-research/SKILL.md` before giving up (robots-checked curl retry, then WebSearch for the employer's current posting, then `expired`).
- If a URL 404s or redirects to an unrelated page, the agent must attempt cross-ATS recovery (check Greenhouse/Ashby directly) before marking `expired`.

Each agent returns a JSON array, one object per role:

```json
{
  "key": "<the role's URL as searched>",
  "status": "triaged" | "expired",
  "corrected_url": "<only if the original URL was a ghost job and a live one was found>",
  "verdict": "strong_fit" | "adjacent" | "excluded",
  "verdict_reason": "<one sentence, quoting the JD if the reasoning isn't obvious from the title>",
  "matched_exclude_pattern": "<one of the 6 hard-exclude pattern names, only if verdict is excluded>"
}
```

## Step 6: Mass-Posting Detection

If two or more results from the same company (or sharing a req/job ID visible in the URL) have substantially the same description and differ only in city, consolidate into one row and note the spread (e.g. "posted identically across 6 cities"). This is a caution signal about distribution pattern, never an accusation.

## Step 7: Store and Update Track-List

Write every result from this run (kept and excluded, at every stage) into `scrape_state/seen_roles.json`:

```json
{
  "seen": {
    "<url>": {
      "title": "...",
      "company": "...",
      "url": "...",
      "first_seen": "YYYY-MM-DD",
      "deadline": "YYYY-MM-DD" | null,
      "title_filter": "kept" | "excluded",
      "exclude_reason": "<pattern name, only if excluded at the title stage>",
      "status": "triaged" | "excluded" | "expired",
      "verdict": "strong_fit" | "adjacent" | "excluded" | null,
      "verdict_reason": "<one sentence, only once triaged>",
      "matched_exclude_pattern": "<only if verdict is excluded>",
      "triage_date": "YYYY-MM-DD",
      "portal": "<portal skill or 'direct-api' or 'websearch'>",
      "source": "cli" | "websearch"
    }
  }
}
```

A role excluded at the title-filter stage (Step 3) never reaches Step 5, so it keeps `verdict: null` - the title pattern alone was reason enough. A role that reaches Step 5 always gets a real verdict or `expired`.

Then update that company's `last_scraped_date` in `track_list.json` to today, and fill in any newly-resolved `portal_resolution` fields from Step 1, plus any URL-reliability notes discovered in Step 4.

Only present roles not already presented in a prior run (check `scrape_state/seen_roles.json` for an existing `verdict` before re-triaging, unless `--all` was passed).

---

## Step 8: Present Results

**Only present roles with a real verdict from Step 5** (`strong_fit` or `adjacent`) in the main results. Title-filter exclusions and Step-5 exclusions are summarized as counts, with the excluded list available on request (for auditability) rather than shown by default.

```
## Search Results - <company or "all tracked companies"> - YYYY-MM-DD

Found X postings. Y strong fit, Z adjacent, W excluded (title filter: a, full-JD read: b), V expired/ghost.

### Strong Fit
| # | Title | Company | Location | Why | URL |
|---|-------|---------|----------|-----|-----|
| 1 | ... | ... | ... | <one-line reason> | [Link](...) |

### Adjacent
(same table shape - roles worth knowing about but with a named gap)
```

If any company could not be resolved to a working portal (Step 1 failure), list it separately. If asked, show the excluded list with each one's matched pattern or JD-based reason for auditability.

---

## Important Rules

1. Never fabricate postings or JD content - only present what a CLI/API/WebSearch/WebFetch call actually returned.
2. Always dedup against `scrape_state/seen_roles.json` before presenting.
3. The Step 3 title filter is a cost-control pre-filter, never the final verdict - a role only gets `strong_fit`/`adjacent`/`excluded` after Step 5's full-JD read.
4. Company-scoped searches use a 30-day recency filter by default (Step 2) - widen to unfiltered only as a deliberate, disclosed one-off when a company's yield looks suspiciously low.
5. `companies.md` is the human-editable source of truth for *which* companies to track; `track_list.json` is the agent-maintained record of *how* to search them. Keep them reconciled every run (Step 0.2).
6. Every hard-exclude verdict names which of the 6 patterns triggered it, so a human can spot-check the classifier's judgment later.
