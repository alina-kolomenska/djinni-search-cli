# djinni-search

A command-line tool and Claude Code skill for searching IT jobs on [Djinni](https://djinni.co/jobs/), the Ukrainian tech job board, and reading full postings. The repo is laid out as a portal skill for [ai-job-search](https://github.com/ai-job-search-dev/ai-job-search): clone it into `.agents/skills/djinni-search` and `/scrape` uses it.

- **TypeScript on Bun, zero runtime dependencies.** Plain `fetch` and HTML/JSON-LD parsing; `bun install` only pulls dev types.
- **Filters:** keywords, Djinni category, remote/part-time/office, candidate region, city, salary, public salary only, posting age.
- **Output:** JSON (default), a table, or plain text. Errors go to stderr as JSON with exit code 1.
- **Polite and defensive:** an honest User-Agent, retries with exponential backoff on 429/5xx, and hidden text (HTML comments, invisible elements, zero-width and bidi characters) stripped from listings before output.
- **Tested:** 56 tests: 55 on synthetic fixtures and a mocked `fetch`, so they run offline, plus an opt-in live smoke test that checks the real site.

> **Personal use only.** Djinni's `robots.txt` allows the `/jobs/` pages this tool reads, and the tool never touches candidate data. Keep volume low: a few searches per session, never a crawl. You are responsible for how you use it.

## Install into ai-job-search

From the root of your ai-job-search workspace:

```bash
git clone https://github.com/alina-kolomenska/djinni-search-cli.git .agents/skills/djinni-search
```

Then add this line to the portal permissions in **both** `.claude/settings.json` (`permissions.allow`) and `tools/security_guards.py` (the reviewed allowlist, next to the other `.agents/skills/*-search` lines):

```
"Bash(bun run .agents/skills/djinni-search/cli/src/cli.ts:*)",
```

`/scrape` discovers every `*-search` skill on its own. To keep the skill installed but skipped, set `enabled: false` in `SKILL.md`. The clone leaves a nested `.git`; delete `.agents/skills/djinni-search/.git` if you'd rather commit the folder into your own workspace.

For plain Claude Code without ai-job-search, clone it into `.claude/skills/djinni-search` instead and change the paths in `SKILL.md` to match.

## Standalone CLI

```bash
git clone https://github.com/alina-kolomenska/djinni-search-cli.git
cd djinni-search-cli/cli
bun install          # optional: TypeScript dev types only

# Remote React roles open to candidates in Ukraine
bun run src/cli.ts search -q React --employment remote --region ukraine --format table

# Python roles, remote, $3000+/month, posted in the last 7 days
bun run src/cli.ts search -c Python --employment remote --salary 3000 --jobage 7 --format table

# Full detail for one posting (id or djinni.co URL)
bun run src/cli.ts detail 855979 --format plain
```

## Commands

| Command | Description |
|---------|-------------|
| `search` | Search job listings; every filter is optional |
| `detail <id\|url>` | Full description, workplace, languages and salary for one posting |

### Search flags

| Flag | Alias | Description |
|------|-------|-------------|
| `--query` | `-q` | Keywords |
| `--search-type` | | `basic` (default), `full-text` or `title`; needs `--query` |
| `--category` | `-c` | Djinni category, e.g. `"React.js"`, `JavaScript`, `Fullstack`; unknown names are rejected |
| `--employment` | | `remote`, `parttime` or `office` |
| `--region` | | Candidate countries: `ukraine`, `eu`, `worldwide`, `other` |
| `--location` | `-l` | City slug for office roles, e.g. `kyiv`, `lviv` |
| `--salary` | | Monthly salary filter, USD |
| `--public-salary` | | Only postings that show a salary |
| `--jobage` | | Posted within N days (filtered client-side) |
| `--page` | | 1-indexed page, 15 results per page |
| `--limit` | `-n` | Cap the number of results |
| `--format` | | `json`, `table` or `plain` |

Each result has `id`, `title`, `company`, `location`, `date` and `url` (`null` when missing), plus `remoteOnly`, `offices`, `employmentType`, `experienceYears`, `english`, `salary`, `salaryTier` and `category`.

### Deviations from the ai-job-search portal contract

- **Fixed page size:** 15 results per page, set by Djinni.
- **`--jobage` is client-side:** Djinni has no posting-age parameter, so the fetched page is filtered (results are newest first).
- **`--employment remote` is wider than remote-only:** it also returns office and hybrid roles that offer remote as one option. `remoteOnly: true` marks the strict ones.
- **Labels are Ukrainian:** Djinni renders anonymous pages in Ukrainian, so `location` and `detail`'s `workplace` read like `Україна` or `Тільки віддалено` (remote only).

## Compared with other Djinni skills

[room2speak/ai-job-search](https://github.com/room2speak/ai-job-search) shipped the first Djinni skill for ai-job-search. It reads Djinni's RSS feed. This one reads the search pages, which carry more per posting:

| | RSS-based (room2speak) | This skill |
|---|---|---|
| Company and location in search results | at `detail` | in every result |
| Work format filter (remote / part-time / office) | - | `--employment` |
| Candidate region filter (Ukraine / EU / worldwide) | - | `--region` |
| City filter | - | `--location` |
| Salary filters | - | `--salary`, `--public-salary` |
| Keyword search | client-side match on the feed | Djinni's own search: basic, full-text or title |
| Category names | exact, case-sensitive key | case-insensitive; unknown names rejected with suggestions |
| Hidden-text stripping | - | yes |

## Development

```bash
cd cli
bun run typecheck
bun run test                  # offline
DJINNI_LIVE=1 bun run test    # also runs the live smoke test against djinni.co
```

`url-reference.md` records the endpoints, parameters and parsing anchors, for when Djinni changes its markup. Tested against ai-job-search `master` at `2562379` (2026-11-05).

## How it was built

Built AI-first with [Claude Code](https://claude.com/claude-code): I set the requirements and the constraints (zero dependencies, offline tests, hidden-text filtering), checked the results against them, and tested the parser against live Djinni pages before relying on it. It started as a portal plugin for the open-source [ai-job-search](https://github.com/ai-job-search-dev/ai-job-search) framework and reuses that project's portal-CLI conventions and text-sanitising helpers.

## License

MIT. See [LICENSE](LICENSE). Portions are adapted from ai-job-search (MIT, copyright Mads Lorentzen).
