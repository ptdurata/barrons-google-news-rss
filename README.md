# barrons-google-news-rss

**Feed URL:** https://ptdurata.github.io/barrons-google-news-rss/barrons.xml

An hourly static mirror of Google News' own public search-as-RSS feed for
`site:barrons.com`. A GitHub Actions job fetches it every hour and GitHub
Pages serves the result as a plain file.

## Why this exists

Barron's publishes no reachable RSS feed of its own — every path (homepage,
official feed, individual articles) sits behind bot detection that blocks
plain automated requests, including from Cloudflare's edge and from GitHub
Actions runners trying to reach Barron's directly. (A companion project,
[`barrons-homepage-rss`](https://github.com/ptdurata/barrons-homepage-rss),
tried the homepage-scraping route and documents exactly why that one is a
dead end: Barron's `robots.txt` disallows the homepage for every crawler
except a named allowlist.)

Google News' own `news.google.com/rss/search?q=site:barrons.com` endpoint is
a different story — it's Google's own public product, returns real, fresh
Barron's headlines with working links (the `<link>` is a Google reader
wrapper that resolves to the real `barrons.com` article URL in an actual
browser), and is reachable from GitHub Actions runners (verified) even though
it isn't reliably reachable from Cloudflare Workers (also verified — Google
503s or times out requests from Cloudflare's shared edge IP range
specifically). This repo exists purely to get that already-public feed onto
infrastructure that can actually reach it, so the Durata dashboard's `news.js`
can fetch a plain static file instead of hitting Google directly.

## What this is not

This never touches barrons.com. It makes one request per run, to Google's own
news search endpoint, the same URL anyone can open in a browser. No login, no
cookies, no browser automation, no bot-detection workaround of any kind —
there's nothing here to get past, since nothing on the request path blocks it.

## Schedule

`.github/workflows/update.yml` runs at the top of every hour and can also be
triggered manually (Actions → **Update Barron's RSS mirror** → **Run
workflow**, or `gh workflow run update.yml -R ptdurata/barrons-google-news-rss`).

A run only commits `barrons.xml` if the fetched content actually changed, so
the repo's history reflects real updates, not hourly no-op noise.

## Safety checks before publishing

A run refuses to overwrite `barrons.xml` — leaving the last good feed in
place — if any of the following happen:

- The fetch doesn't return HTTP 200.
- The response is suspiciously small (under 5KB — a block page or empty body
  is far smaller than a real ~100-item feed).
- The response isn't well-formed XML.
- Fewer than 5 `<item>` elements are found.

## Files

| File                            | Purpose                                      |
|----------------------------------|-----------------------------------------------|
| `.github/workflows/update.yml`   | Hourly + manual job; fetches and commits the feed if changed |
| `barrons.xml`                    | The published feed (generated, not hand-edited) |
| `.nojekyll`                      | Serve files as-is on GitHub Pages            |
