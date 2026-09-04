# cache-warmup

Walk a sitemap and request every page, so the server or CDN cache is primed
before real visitors arrive.

The point of difference: **it checks that what it warmed is not broken.** A
page that dies halfway through rendering still returns `HTTP 200`, and your
cache will happily store it and serve it to everyone. A naive warmer reports
that as a success.

```
cache-warmup https://example.com/sitemap.xml
```

```
sitemap index: https://example.com/sitemap.xml
sitemap: https://example.com/wp-sitemap-posts-page-1.xml (20 urls)
warming 53 urls as a browser, one at a time
[ 1/53] 200   0.14s    320K  https://example.com/
[ 2/53] 200   0.19s    275K  https://example.com/about/
...
checking 87 stylesheet/script url(s) referenced by those pages

53 urls, avg 0.15s, slowest 0.30s
  2xx          53
  median page 282K
  87 stylesheet/script urls, 0 missing
```

## Install

Requires `bash`, `curl` and `awk` — everything is present on a stock macOS or
Linux box. Drop it anywhere on your `PATH`:

```sh
curl -o ~/.local/bin/cache-warmup \
  https://raw.githubusercontent.com/joachim-tecklenburg/cache-warmup/main/cache-warmup
chmod +x ~/.local/bin/cache-warmup
```

## Usage

```
cache-warmup [options] <sitemap-url|domain>
```

A bare domain is expanded to `https://<domain>/sitemap.xml`, so
`cache-warmup example.com` is usually all you need. Sitemap indexes are
followed recursively (up to 3 levels), plain sitemaps work too, `.xml.gz` is
decompressed, and URLs are deduplicated before anything is requested.

`robots.txt` is read first and every sitemap it announces is walked as well:

```
robots.txt lists 3 sitemap(s)
sitemap: https://example.com/sitemap-de.xml (48 urls)
sitemap: https://example.com/sitemap-en.xml (48 urls)
sitemap: https://example.com/sitemap-fr.xml (31 urls)
```

This matters on multilingual sites. WPML, Polylang and most of the
multi-site setups publish one sitemap per language and list them all in
`robots.txt`, while `/sitemap.xml` carries only the default language.
Warming that one file leaves every other language cold — and a cold
language is exactly where a visitor is most likely to be the unlucky one
who triggers a bad cache write. `Sitemap:` is matched case-insensitively,
site-relative paths are accepted, and each sitemap is walked once no matter
how many times it is named. Entries pointing at a **different host** are
reported and skipped; someone else's `robots.txt` is not a reason to go and
hammer their server. Use `-N` to warm strictly the sitemap you named.

| Option | Meaning |
| --- | --- |
| `-P N` | request N pages in parallel (default: 1, strictly sequential) |
| `-d SEC` | pause between pages; `auto` scales it with render time, `0` disables (default: `auto`) |
| `-t SEC` | per-request timeout (default: 120) |
| `-A STR` | User-Agent (default: a current desktop Chrome) |
| `-a LANG` | Accept-Language |
| `-H 'K: V'` | extra request header, repeatable |
| `-F` | do not follow redirects |
| `-r` | do not retry failed or suspect pages |
| `-s` | skip the stylesheet/script check |
| `-C` | skip the cross-page consistency check |
| `-N` | do not read `robots.txt`; warm only the sitemap given |
| `-l` | list the URLs only, don't request them |
| `-q` | quiet: only the summary |
| `-h` | help |

Exit status is `1` if any page or asset came back broken, so it drops into a
deploy hook or cron job unchanged.

## What it checks

Four failures produce a broken layout while still returning `HTTP 200`, and
each has a check:

- **Truncated pages.** A page that ran out of memory or time mid-render is
  stored by the cache as a complete 200. Every response body is checked for a
  closing `</html>`.
- **Short pages.** A page can close its tags and still have rendered only a
  fraction of its content. Every page is measured against the median page size
  of the run and flagged below 40%.
- **Missing stylesheets and scripts.** A page whose CSS 404s renders as
  unstyled HTML. Every same-origin `<link rel=stylesheet>` and `<script src>`
  is collected from the page bodies, deduplicated across the whole run, and
  fetched once to prove it exists. Quoted, unquoted and single-quoted
  attributes are handled, as is `rel=preload as=style`; third-party hosts and
  inline scripts are ignored. A file that fails is asked for again before it
  is believed — a live server was measured returning 404 for a stylesheet that
  was plainly there about once in every sixty requests, and one bad answer is
  not evidence a file is missing. Files that fail and then load are reported
  separately and do not fail the run, because real visitors hit the same odds.
- **Pages that do not load CSS their neighbours load.** The nastiest one,
  because the page is complete, the right size, and every asset it points at
  resolves — it is simply not pulling something in. A page built while the
  backend was still warming up can silently omit a generated `<style>` block,
  and the cache stores that copy for good. Every page is fingerprinted with
  the stylesheets it links and every `url()` its inline CSS references — fonts
  above all. Anything 90% of the run pulls in is expected everywhere, and
  pages missing something are named along with the exact file.

  The fingerprint deliberately keys on **what the CSS delivers, not which
  `<style>` block delivers it**. An earlier version compared `<style id=…>`
  labels and produced a false positive on the first real site it ran against:
  Elementor emitted the same `@font-face` rules under one id on some pages and
  two on others, and the tool called a perfectly good page broken. Pages whose
  markup is shaped differently but which load the same files are now reported
  as a note and do not fail the run.

Anything flagged is retried once, sequentially. Whatever is still broken is
listed at the end, because a bad page already in the cache will not be fixed
by warming — the cache answers without ever asking the backend. Purge those
URLs first, then warm again.

## Why it paces itself

PHP does not stop working when the response is sent. Transients, options and
the cache file itself are written after the last byte goes out. Requesting the
next page the instant the previous one completes starts a fresh render on top
of that shutdown work — and a generated `<style>` block whose transient is
mid-write can come back empty, which is how a page ends up cached with its
fonts or its custom CSS missing.

Note that this is not a concurrency problem: at the default `-P 1` there is
only ever one request in flight. A browser actually hits the server *harder*
per page, because it fetches every stylesheet and script in parallel right
after the HTML. What a browser has is a gap between one page and the next.
`-d` is that gap.

A flat number is the wrong instrument, so the default is `auto`: the pause is
30% of how long the page took, capped at 1.5s, and skipped entirely below 0.5s
because a response that fast came from the cache and rendered nothing. On a
site that is already warm this costs about 2 seconds across 125 pages. On a
cold site it scales the pause to the pages that actually need it.

Use `-d 0` to turn it off, or `-d 2` for a fixed pause if a backend needs more.

## Why sequential by default

Warming with many parallel requests is how you break the thing you are trying
to speed up. Several workers rendering heavy pages at once on a cold backend
can push one into a memory or time limit; it emits a partial page with status
200, and the cache keeps it. Pages are fetched one at a time unless `-P` says
otherwise.

## Why it impersonates a browser

Requests carry a real desktop Chrome `User-Agent` plus the `Accept`,
`Accept-Language` and `Sec-Fetch-*` headers a browser sends, follow redirects,
and accept compressed responses. This matters more than it looks:

- Bare `curl` sends no `Accept-Encoding` at all. On a site sending
  `Vary: Accept-Encoding`, that warms the uncompressed variant while every
  real visitor still gets a cold miss.
- A `curl/8.x` User-Agent triggers server-side branching that a browser never
  sees — mobile/device cache buckets, bot detection in security and consent
  plugins, WAF challenges. An interstitial or a stripped bot page returns 200
  and gets cached for everyone.

Two differences remain and cannot be fixed here: no JavaScript is executed, so
client-rendered content is not warmed, and the TLS fingerprint is still
curl's, which a determined bot protection can spot.

## License

MIT — see [LICENSE](LICENSE).
