# site-signals

Continuous technical SEO and AI-readiness monitoring for small sites. Read
`docs/scope-v1.md` before proposing work — it defines what v1 is and, more
usefully, what it is deliberately not.

## Where this runs, and what that forbids

The app is hosted on **MyDevil shared hosting**, which changes three decisions:

- **FreeBSD.** Prebuilt native binaries are usually missing and `node-gyp` on a
  shared host is a bad afternoon. **Pure-JavaScript dependencies only.** No
  Playwright, Puppeteer, sharp, better-sqlite3 or anything else that compiles.
- **Phusion Passenger** serves the app from `public_nodejs/app.js` and sleeps it
  after 24 hours without requests. **No scheduler inside the web process** — a
  `setInterval` there is a clock that sometimes does not tick. Crawls run from
  `bin/run.js`, called by cron.
- **Node 22** is the default (`node22`, `npm22`); 16/18/20/23/24 also available.

Anything that genuinely needs a browser runs elsewhere and posts its result back.
GitHub Actions is the intended host for that: standard runners are free without a
minute allowance **because this repository is public**, and they ship with Chromium.
Two caveats to design around — scheduled workflows are best-effort and can be delayed
or skipped, and GitHub disables them after 60 days of repository inactivity.

## Conventions

- **Never commit to `main`.** Branch → PR → squash merge. A hook enforces this.
- Code comments and PR descriptions in **English**, always, even when the
  conversation is in Polish.
- Measure against **rendered HTML**, not source. Attribute order varies: match the
  tag, then read `content` regardless of position — `<meta content="…" name="…">`
  and the reverse both occur in the wild.
- Bounded concurrency for fetching, never `Promise.all` over a whole sitemap.

## Verifying

There is no staging. A change to the crawl is verified by running it against a real
sitemap (`spoko.space` has ~120 URLs and a known history of regressions) and reading
the rows it wrote.
