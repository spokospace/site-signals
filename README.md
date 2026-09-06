# site-signals

Continuous technical SEO and AI-readiness monitoring for small sites.

The checks themselves are not new — most of them exist as one-off build scripts in
half my repositories, and I run the rest by hand whenever something feels off. What
is missing is **the difference between two runs**. "Four meta descriptions went over
the limit this week", "two pages lost their `.md` alternate", "`/pl/portfolio.md`
started returning 404" — each of those sat unnoticed for months on my own site until
I happened to look.

So this is not an auditing tool. It is the same audit, run on a schedule, with a
memory.

## Status

Early. Nothing works yet.

## What it checks

Starting from the checks that actually caught something on a live site:

- **Meta** — rendered `<title>` and description lengths against the limits Google
  applies, not the ones people quote.
- **Markdown layer** — every URL in the sitemap has a `.md` alternate, and each one
  is declared with `<link rel="alternate" type="text/markdown">`. Relevant now that
  `llms.txt` conventions promise exactly that.
- **Structured data** — JSON-LD parses, and its URLs resolve.
- **Internal links** — nothing in the rendered HTML points at a 404, including
  redirect rules.
- **hreflang** — declared alternates exist and reciprocate.

## How it is built, and why

Three constraints shaped the architecture, all of them about cost:

**The app runs on shared hosting behind Phusion Passenger.** Passenger sleeps an app
after 24 hours without requests and wakes it on the next one, so there is no resident
scheduler. Crawls are triggered from outside — by cron, or over HTTP — never by a
timer inside the process.

**The host is FreeBSD**, which rules out a local headless browser and makes native
bindings a gamble. Fetch and parse cover every check above without rendering. Anything
that genuinely needs a browser runs elsewhere and posts its result back.

**Elsewhere is GitHub Actions.** Standard runners are free without a minute
allowance on public repositories, and they ship with Chromium — which is one of the
reasons this repository is public. Two caveats worth writing down: scheduled workflows
are best-effort and can be delayed or skipped, and GitHub disables them after 60 days
of repository inactivity. Neither matters for a daily check; both matter if you mistake
`schedule:` for a clock.

## Licence

MIT.
