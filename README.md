# barrons-google-news-rss

Hourly static mirrors of Google News' own public search-as-RSS feed, for
sources that publish no reachable RSS feed of their own. Kept the original
repo name (and `barrons.xml`'s existing URL) for backward compatibility even
though it now covers more than one source.

**Feed URLs:**
- `https://ptdurata.github.io/barrons-google-news-rss/barrons.xml` — `site:barrons.com`
- `https://ptdurata.github.io/barrons-google-news-rss/brookings.xml` — `site:brookings.edu`

A single GitHub Actions job fetches every source on the list each hour and
GitHub Pages serves the results as plain files.

## Why this exists

- **Barron's** publishes no reachable RSS feed at all — every path (homepage,
  official feed, individual articles) sits behind bot detection that blocks
  plain automated requests, including from Cloudflare's edge and from GitHub
  Actions runners trying to reach Barron's directly. (A companion project,
  [`barrons-homepage-rss`](https://github.com/ptdurata/barrons-homepage-rss),
  tried the homepage-scraping route and documents exactly why that one is a
  dead end: Barron's `robots.txt` disallows the homepage for every crawler
  except a named allowlist.)
- **Brookings** did publish a real feed at `brookings.edu/feed/`, but that
  path now 302s to the homepage via an intentional "Safe Redirect Manager"
  rule (confirmed live — the feed was killed in a redesign, not blocked by
  bot detection), and `brookings.edu/rss` just redirects into the same dead
  `/feed/`. No working path exists on their own domain.

Google News' own `news.google.com/rss/search?q=site:<domain>` endpoint sides
around both problems — it's Google's own public product, returns real, fresh
headlines with working links (the `<link>` is a Google reader wrapper that
resolves to the real article URL in an actual browser), and is reachable from
GitHub Actions runners (verified for both sources) even though it isn't
reliably reachable from Cloudflare Workers (also verified — Google 503s or
times out requests from Cloudflare's shared edge IP range specifically). This
repo exists purely to get that already-public feed onto infrastructure that
can actually reach it, so the Durata dashboard's `news.js` can fetch a plain
static file instead of hitting Google directly.

## What this is not

This never touches barrons.com or brookings.edu. It makes one request per
source per run, to Google's own news search endpoint, the same URL anyone can
open in a browser. No login, no cookies, no browser automation, no
bot-detection workaround of any kind — there's nothing here to get past,
since nothing on the request path blocks it.

## Adding another source

Add a `"name:site:domain.com"` line to the `SOURCES` array in
`.github/workflows/update.yml`. Everything else (fetch, validation,
publish-on-change) is shared across every entry.

## Schedule

`.github/workflows/update.yml` runs at the top of every hour and can also be
triggered manually (Actions → **Update Google News RSS mirrors** → **Run
workflow**, or `gh workflow run update.yml -R ptdurata/barrons-google-news-rss`).

A run only commits a given source's file if its fetched content actually
changed, so the repo's history reflects real updates, not hourly no-op noise.
A problem with one source (blocked, empty, malformed) only skips that
source's file — it never blocks or rolls back the others.

## Safety checks before publishing (per source)

A run refuses to overwrite a source's `.xml` file — leaving the last good
feed in place — if any of the following happen:

- The fetch doesn't return HTTP 200.
- The response is suspiciously small (under 5KB — a block page or empty body
  is far smaller than a real ~100-item feed).
- The response isn't well-formed XML.
- Fewer than 5 `<item>` elements are found.

## Files

| File                            | Purpose                                      |
|----------------------------------|-----------------------------------------------|
| `.github/workflows/update.yml`   | Hourly + manual job; fetches and commits each source's feed if changed |
| `barrons.xml`                    | Published Barron's feed (generated, not hand-edited) |
| `brookings.xml`                  | Published Brookings feed (generated, not hand-edited) |
| `.nojekyll`                      | Serve files as-is on GitHub Pages            |
