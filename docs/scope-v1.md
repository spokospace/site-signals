# Scope — v1

## What v1 has to prove

That the loop works end to end: crawl a site on a schedule, keep what it found, and
say what changed since last time. Everything else is a feature request against a thing
that already runs.

The bar for "done" is concrete: **site-signals runs daily against spoko.space and
uper.pl, and the next time something regresses I find out from an email rather than by
stumbling into it.** Three real regressions on spoko.space went unnoticed for months —
`/pl/portfolio.md` linking to 404s, a CSS class that matched no rule, twenty-four meta
descriptions over the limit. Each one would have shown up in a diff on the day it
shipped.

## The loop

```
sitemap → fetch each URL (bounded concurrency) → extract signals → snapshot
                                                                      ↓
                                                     diff against previous snapshot
                                                                      ↓
                                                         dashboard + email if changed
```

A **target** is a site plus its sitemap URL. A **run** is one pass over that sitemap.
A **snapshot** is what one run saw for one page. A **finding** is a difference between
the newest snapshot and the one before it.

## Checks in v1

Only checks that have already caught something real:

| Check | What it records |
|---|---|
| Status | HTTP status of every sitemap URL |
| Title | rendered `<title>`, decoded, length against 60 |
| Description | `<meta name="description">`, length against 160, presence |
| Canonical | present, and where it points |
| Markdown alternate | `<link rel="alternate" type="text/markdown">` present, and the `.md` returns 200 |
| hreflang | declared alternates resolve, and reciprocate |
| JSON-LD | every block parses |

Attribute order matters here: `astro-compress` emits both `<meta content="…" name="…">`
and the reverse on the same site, so match the tag and then read `content` regardless of
position. A regex that assumes one order reports false "missing description" hits.

## Not in v1

The list that keeps this shippable. Every item is defensible later; none of it is
needed to prove the loop.

- **Screenshots and visual diffing.** None of the regressions above needed rendering.
  This is the single most likely thing to swallow the project.
- **Crawling beyond the sitemap.** Checking every internal link means ~30 000 requests
  per run on spoko.space against ~120 for the sitemap. v1 checks the pages the sitemap
  declares; link crawling is its own milestone with its own budget.
- **Search Console and GA4.** A second data source doubles the surface and answers a
  different question — what Google *thinks*, not what the site *is*.
- **Users, auth, billing, multi-tenancy.** Config lives in a table; there is one
  operator and he has SSH.
- **Rendering JavaScript.** The sites being watched are static.
- **Inngest.** Cron plus a jobs table does this at this scale. Adding Inngest later is
  a deliberate choice about what the CV should show, not a requirement the load creates —
  worth writing down so the dependency's reason survives.

## Shape

**`bin/run.js`** — the whole loop as a CLI. This is what cron calls. Running the crawl
outside the web process sidesteps Passenger sleeping an idle app, and keeps the slow
work off the request path.

**`app.js`** — Passenger entry. Serves the dashboard and a token-protected
`POST /internal/run` so a run can also be triggered over HTTP later, from a GitHub
Action or by hand.

**Storage** — Postgres on the same hosting. Four tables: `targets`, `runs`,
`page_snapshots`, `findings`.

**Dependencies** — pure JavaScript only. FreeBSD rarely has prebuilt native binaries,
and a failed `node-gyp` build on a shared host is a bad afternoon. `undici` for fetching,
a JS HTML parser, `pg`, `nodemailer` against the host's own SMTP.

**Concurrency** — a bounded queue, not `Promise.all` over the whole sitemap. Somewhere
around 6–8 in flight, with a real User-Agent. This is the part that actually demonstrates
Node doing what Node is for.

## Milestones

**M1 — the loop, headless.** `bin/run.js` reads a sitemap, fetches every URL with
bounded concurrency, extracts the signals above, writes a run and its snapshots. No UI.
Verified by running it against spoko.space and reading the rows.

**M2 — memory.** Diff the newest run against the previous one, store findings, and put
a dashboard in front of it: last run, what changed, one page per run.

**M3 — it tells you.** Email when a run produces findings. Cron on MyDevil, daily.
Domain pointed at the app.

Each milestone is useful on its own, and any of them is a reasonable place to stop.

## Risks worth naming

**Native dependencies on FreeBSD.** Mitigated by the pure-JS rule above; if something
turns out to need a binary, it does not go in.

**Passenger sleeping the app.** Mitigated by making cron call the CLI directly rather
than an HTTP endpoint. The web process only needs to be awake when someone is looking.

**Scope.** The "Not in v1" list is the mitigation. It is worth re-reading before
starting M2.
