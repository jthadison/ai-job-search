---
name: usajobs-search
version: 1.0.0
description: >
  Use this skill to search USAJOBS, the official job board of the US federal
  government, covering every federal agency. Reach for it when the user wants
  government work, federal jobs, civil service roles, GS-grade positions, or
  jobs at a named agency (IRS, NASA, VA, DoD, Treasury). Requires a free API key
  in USAJOBS_API_KEY / USAJOBS_EMAIL — this skill is disabled until those are
  set. Trigger phrases: federal jobs, government jobs, USAJOBS, civil service,
  GS-13, work for the government, agency jobs, federal hiring, public sector jobs.
context: fork
enabled: true  # requires USAJOBS_API_KEY + USAJOBS_EMAIL in the repo-root .env
allowed-tools: Bash(bun run .agents/skills/usajobs-search/cli/src/cli.ts *)
---

# USAJOBS Search Skill

Search [USAJOBS](https://www.usajobs.gov), the official US federal government job
board, via its [public Search API](https://developer.usajobs.gov/api-reference/get-api-search).
**Zero runtime dependencies.**

Unlike the other portals in this fork, USAJOBS filters **entirely server-side** —
keyword, title, location, agency, remote and recency are all real API parameters.
That makes it the most precise portal here, once it is set up.

## ⚠️ Setup required: a free API key (~2 minutes)

USAJOBS is the only authenticating portal in this fork. Without credentials every
request returns HTTP 401, and an always-failing portal in `/scrape` is worse than
an absent one — so `enabled` tracks whether credentials are actually present.

**In this fork it is `enabled: true`**: the key is configured in the repo-root
`.env` and verified against live data. If you clone this fork and do not have your
own USAJOBS key, set `enabled: false` until you do.

1. Register at <https://developer.usajobs.gov/apirequest/> (free, email-confirmed).
2. Put both values in **`.env` at the repo root** (the file already exists; copy
   `.env.example` if it doesn't). The registered email is sent as the
   `User-Agent` header, which is how USAJOBS identifies callers:

   ```dotenv
   USAJOBS_API_KEY=<key from the confirmation email>
   USAJOBS_EMAIL=<the email you registered>
   ```

   Bun auto-loads `.env` from the directory it is run in, and the portal skills
   are invoked from the repo root — so **repo root is the location that works**.
   A `.env` inside `cli/` is silently ignored when the skill runs. (Verified:
   run from the root the CLI reaches USAJOBS; run from `cli/` it reports
   `NO_CREDENTIALS`.)

   A shell `export` works too and takes precedence, but does not persist across
   terminals.

3. Set `enabled: true` in this file's frontmatter. (Already done here — this step
   is for a fresh clone that started from `enabled: false`.)

`.env` and `.env.*` are gitignored, so the key is never committed. Credentials
are read from the environment only and never written anywhere by the CLI.

### Verification status

**Verified against live data on 2026-08-05**, with a real key. Both `search` and
`detail` return correctly parsed results.

That live run mattered: the parser was originally built to USAJOBS' *published*
schema, and three defects only surfaced against real responses —

1. `PositionSchedule[].Name` is **always empty**; only `Code` is populated, so
   `schedule` was permanently `null`. Codes are now resolved against USAJOBS'
   own `positionscheduletypes` codelist.
2. `RateIntervalCode` is a terse `"PA"` while a sibling `Description` field holds
   `"Per Year"` — salaries rendered as `"$85,447 - $187,093 PA"`.
3. `PositionURI` carries an explicit `:443` port. `/scrape` dedupes on URL, so
   the `:443` and bare forms would have counted as two distinct jobs.

All three are fixed and pinned by regression tests whose fixture uses the real
live shape rather than the idealised schema. Treat this as a caution for any
future portal: schema-derived parsers look correct right up until they meet the
API.

## Commands

### Search job listings

```bash
bun run .agents/skills/usajobs-search/cli/src/cli.ts search [flags]
```

Key flags (**all server-side**):
- `--query <text>` / `-q <text>` — keyword across the whole announcement (matches synonyms too).
- `--title <text>` / `-t <text>` — match within the job title only ("contains").
- `--location <text>` / `-l <text>` — city or military installation. USAJOBS wants the **full state name**: `"Austin, Texas"`, not `"Austin, TX"`.
- `--organization <code>` — agency subelement code, e.g. `TR` for Treasury.
- `--remote` — only remote/telework-eligible roles.
- `--onsite` — exclude remote roles. Mutually exclusive with `--remote`; omitting both includes remote by default.
- `--jobage <days>` — posted within N days. **The API maximum is 60**; larger values are clamped rather than rejected.
- `--page <n>` — 1-indexed page.
- `--limit <n>` / `-n <n>` — results per page, up to 500. Default 25.
- `--format json|table|plain` — default `json`.
- **`--limit` is a TOTAL OUTPUT CAP here** (this portal paginates server-side, so page
  size is fixed by the API). `--page` selects the API's page; `--limit` trims what is
  emitted. For "just the top N", pass `--limit N` and omit `--page`.


### Fetch full job detail

```bash
bun run .agents/skills/usajobs-search/cli/src/cli.ts detail <control-number|announcement-number|url> [--format json|plain]
```

Accepts the numeric control number (the `id` from search results), the
announcement number (`positionId`, e.g. `ST-12345678-26-XY`), or a
`usajobs.gov/job/...` URL. Returns the job summary, major duties, requirements,
qualification summary, salary band, grade and apply link.

## Usage examples

```bash
# Remote data science roles across all agencies
bun run .agents/skills/usajobs-search/cli/src/cli.ts search -q "data scientist" --remote --format table

# Software engineering roles in Austin
bun run .agents/skills/usajobs-search/cli/src/cli.ts search -t "Software Engineer" -l "Austin, Texas" --format table

# Anything at Treasury posted in the last two weeks
bun run .agents/skills/usajobs-search/cli/src/cli.ts search --organization TR --jobage 14 -n 50 --format table

# Full details for one announcement
bun run .agents/skills/usajobs-search/cli/src/cli.ts detail 828810900 --format plain
```

## Output formats

| Format | Best for |
|--------|----------|
| `json` | Default — programmatic use, includes `meta.apiTotal` |
| `table` | Quick human-readable scanning |
| `plain` | Reading a single job's full detail (`detail` command) |

All errors are written to **stderr** as `{ "error": "...", "code": "..." }` and the process exits with code `1`.
`NO_CREDENTIALS` means the environment variables are unset; a translated 401 means they are wrong.

## Notes

- **Only currently open announcements are returned.** USAJOBS' search API does not serve closed postings, so a `detail` lookup on an expired announcement returns `NOT_FOUND` even when the id is correct.
- **`WhoMayApply=Public` is always sent** — those are the roles US citizens can apply for. The `All` and `Status` values require additional authorization that a standard key does not carry, so federal-employee-only ("Status") postings are not visible here.
- **There is no fetch-by-id endpoint.** Per-announcement text lives behind a separate `/api/announcementtext` API needing extra authorization, so `detail` searches for the identifier as a keyword and selects the **exact** id match — a fuzzy keyword hit is never returned as the answer.
- **Location needs full state names.** `"Austin, TX"` will not match; use `"Austin, Texas"`.
- Salary arrives as a min/max band plus a rate interval, not a single number — it is formatted as `"$99,200 - $153,354 Per Year"`.
- Flag validation runs **before** the credentials check, so a typo reports as a typo rather than as a missing key.
- Grades (`GS-13` etc.) drive federal salary. If you are not already in the federal system, filtering by salary is usually easier than by grade.
