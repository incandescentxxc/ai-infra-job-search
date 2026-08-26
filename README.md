# AI Infra Job Search

A purpose-built job-search tool for **AI infrastructure, LLM infrastructure, and LLM inference/serving roles**. Unlike a generic job scraper, this repo treats a growing, curated **company track-list** as a first-class object: it's built to search a known set of AI-infra-relevant employers thoroughly and to grow that list over time, rather than to search the whole job market broadly.

## What this does

- Maintains a company track-list (`company-tracker/companies.md` for humans, `company-tracker/track_list.json` for the agent) of employers known to hire AI/LLM infra talent.
- Searches each tracked company's open roles via portal CLIs (LinkedIn, freehire.me) or direct ATS APIs (Greenhouse, Ashby).
- Classifies roles against an AI/LLM-infra-specific rubric: distinguishes genuine infra/inference/serving roles from adjacent-but-different roles (product engineering, security engineering, forward-deployed/customer-facing roles, research-scientist roles, and Staff-tier seniority) - by reading the actual job description, not just the title.
- Batch-scores newly found roles so a human can triage a shortlist instead of reading every posting.
- Aggregates prep signal across all high-fit postings: technical and soft skills reframed as what to prepare (not a relabeled requirements list), frequency stats ("N of M high-fit JDs mention X"), and named open-source projects/papers/techniques worth reading or building toward for interviews and project experience.

## What this does NOT do (yet, by design)

- No candidate-specific fit scoring, CV tailoring, or cover letters - this is a market-mapping tool, not a personal application assistant.
- No open-ended "find AI infra companies not on the list" discovery - the track-list is curated and grown deliberately, not auto-expanded.

## Repo structure

- `company-tracker/companies.md` - human-editable list of tracked companies + careers page URLs + a one-line note each. Keep this light and skimmable.
- `company-tracker/track_list.json` - machine state per company: ATS/portal used, resolution slug, status, when added and why, when last scraped. The agent reads and updates this; humans mostly edit `companies.md` and let the agent reconcile.
- `.agents/skills/` - portal search CLIs (`linkedin-search`, `freehire-search`), zero-dependency, run via `bun`.
- `.claude/skills/` - `ai-infra-role-classifier` (the fit rubric), `jd-insights-extractor` (per-JD prep-signal extraction schema), `web-research` (retrieval + ghost-job playbook).
- `.claude/commands/` - `/add-portal` (scaffold a new portal CLI), `/search-roles` (search the track-list, filter, and classify against the AI-infra rubric - the single entry point for finding roles), `/summarize-roles` (aggregate prep signal across all high-fit roles found so far).
- `prep-briefs/` - dated, regeneratable snapshot reports from `/summarize-roles` (tracked in git - these are a deliverable, not local state).
- `tools/` - `robots_check.py` (compliance gate for the browser-header retry), `lint_skills.py` (skill/command file linter).

## How to Use

### 1. The company track-list (`companies.md`)

`company-tracker/companies.md` is the human-editable list of companies to search - just name, careers page, and a one-line note. To track a new company, add a row. To stop tracking one, delete its row (or move it under "Paused" to keep a record of why without deleting).

You don't need to know the company's ATS, ID, or anything technical - the agent resolves that automatically into `company-tracker/track_list.json` the next time `/search-roles` runs, and keeps it updated (last-scraped date, portal quirks, ghost-link warnings) from then on. `companies.md` is the only file you should need to hand-edit day to day.

### 2. The skills in this repo

- **`/search-roles [company | all | health]`** - the main entry point. Searches the tracked companies (or one named company), filters obvious noise, reads the full job description of everything else, and classifies it against the AI/LLM-infra rubric. Only returns roles with a real verdict (`strong_fit` or `adjacent`) - never a bare title list.
  - `/search-roles fireworks` - search one tracked company (matches by name substring)
  - `/search-roles all` - search every active company on the track-list
  - `/search-roles health` - check whether the portal CLIs are still working, without doing a full search
- **`/add-portal`** - scaffold a new portal search CLI when a company can't be resolved through freehire, LinkedIn, or a direct Greenhouse/Ashby API call. Investigates the target site, generates a CLI following this repo's portal-skill contract, and test-runs it against a live query before registering it.
- **`/summarize-roles [company | --include-adjacent]`** - once `/search-roles` has found some `strong_fit` roles, this reads their full JDs and aggregates prep signal across all of them: technical and soft skills framed as what to prepare (not a relabeled requirements list), frequency stats ("N of M high-fit JDs mention X"), and named open-source projects/papers/techniques worth reading or building toward. Writes a dated report to `prep-briefs/`.

## Contributions Welcome

The company track-list is the main thing this repo gets better by growing. If you know of a company hiring for AI infrastructure, LLM infrastructure, or LLM inference/serving roles that isn't in `company-tracker/companies.md` yet, add it - that file is intentionally kept light (name, careers page, one-line note) so contributing a company is a one-line PR, not a research project. `/search-roles` reconciles the richer `track_list.json` metadata (ATS, portal resolution, scrape history) automatically on its next run.

Also welcome:
- New portal skills via `/add-portal`, especially for ATS platforms not yet covered (Lever, Workday, etc.) or company-specific career pages that need bespoke handling.
- Refinements to `.claude/skills/ai-infra-role-classifier/SKILL.md`'s calibration table - if you find a posting that the rubric misclassifies, add it as a new calibration example with the reasoning, the same way the existing examples are recorded.
- Fixes to portal-specific quirks recorded in `company-tracker/track_list.json` (e.g. an ATS migration that makes a portal's cached URLs go stale) - these notes are what keep `/search-roles` from repeating mistakes.

## Acknowledgements

Adapted from [ai-job-search](https://github.com/MadsLorentzen/ai-job-search). Thanks to the authors of ai-job-search.
