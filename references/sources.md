# Sources: which to pick, and how each one actually behaves

Everything under "Behaviour" was **measured on 2026-09-28** with plain curl (browser UA) and the built-in
web fetch. Sites change. When a source behaves differently from what is written here, log it, adapt, and
update this file afterwards.

## Pick by medium

Choose 3–5. **Bold** means reliable to automate; *italic* means fallback-only (usually blocked, or low
yield).

| Medium | First choices | Also good | Fallback-only |
|---|---|---|---|
| Branding and logos | **BP&O**, **Brand New**, **Identity Designed** | **Logobook** (marks only), **Fonts In Use** (branding format) | *Behance*, *Dribbble* |
| Posters, flyers, print | **Fonts In Use** (posters and flyers), **typo/graphic posters** | **Are.na** | *Behance* |
| Typography | **Fonts In Use**, **Typewolf** | **typo/graphic posters** | |
| Packaging | **The Dieline**, **BP&O** | **Fonts In Use** (packaging), **Identity Designed** | *Behance* |
| Web | **Awwwards**, **Minimal Gallery**, **SiteInspire** (via web fetch) | **Typewolf** (site of the day) | *Land-book*, *Godly → recent.design* |
| Mobile and app UI | **Fonts In Use** (mobile/software formats), **Are.na** | **Awwwards** (mobile) | *Dribbble*, *Behance* — **not Mobbin** |
| Illustration, social, mixed, moodboard | **Are.na** | **Fonts In Use** (art/illustration, social formats) | *Behance*, *Dribbble* |

Behance and Dribbble are the obvious names, but both block automated access (see below). Are.na often
holds re-saved Behance and Dribbble work, with the original link in `source.url`, so it is the practical
route to that material.

## Shared recipes

**Browser UA** for every curl call:

```bash
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0 Safari/537.36"
```

**og: tags from a project page.** Use this when you have a page URL but no image, or need the canonical
title:

```bash
og() { curl -sL --max-time 20 -A "$UA" "$1" </dev/null | tr '\n' ' ' \
  | grep -oE '<meta[^>]+(og:image"|og:title|name="author")[^>]*>' | sed -E 's/.*content="([^"]*)".*/\1/'; }
```

Watch for **generic og:images**: site-wide share cards (`social.jpg`, `og.png`, a logo) or composite cards
(Fonts In Use's `cards.fontsinuse.com/cardshot…` overlays the typeface on the image). Reject those and
take the real media from the page body instead.

**WordPress REST search.** Several galleries run on WordPress. Its REST API returns title, link and
featured image in one JSON call, with no HTML scraping:

```bash
curl -s --max-time 25 -A "$UA" "$BASE/wp-json/wp/v2/posts?search=<q>&per_page=12&_embed=wp:featuredmedia" </dev/null \
  | jq -c '.[] | {title: .title.rendered, url: .link,
                  image_url: (._embedded["wp:featuredmedia"][0].source_url // null)}'
```

Titles come back HTML-encoded (`&#8217;`), so decode them before display. A null `image_url` means
there is no featured image; fall back to `og` on the post URL. The **creator is usually in the title**
("… by Pentagram", "New Identity for X by TEMPLO"). If it isn't, read it from the post body with web
fetch. Never infer it.

**`site:` fallback.** When a listing page fails, run web search `site:<domain> <artifact noun> <style>`.
Keep only results that are individual project pages (not search, tag or "hire" pages). Then use `og` or
web fetch on each.

---

## Per source

### Fonts In Use — fontsinuse.com — **reliable**
- **Search:** `https://fontsinuse.com/search?terms=<q>` works with curl and returns about 50–60 uses. Each
  result is a `class="fiu-galleryItem"` block holding everything needed to screen without opening the page:
  `href="/uses/<id>/<slug>"`, `__headline">Title`, `__date">Year`, `__designers"><li>Name</li>…` and a
  `fiu-sampleList` of `/typefaces/<id>/<slug>"><img … alt="Typeface">`. Split the HTML on
  `class="fiu-galleryItem"` and regex each chunk. Pick from this list, then open only the picks.
- **Query words:** plain event nouns work (`design conference`, `design event poster`). Results rank by
  text match, so expect websites, logos and books mixed in; filter titles before opening pages.
- **Format listings:** `https://fontsinuse.com/in/2/formats/<id>/<name>`, for example `12/posters-flyers`,
  `9/branding-identity`, `4/packaging`, `3/web`, `16/mobile-tablet`, `17/software-apps`,
  `18/art-illustration`, `59/social-media`, `6/books`, `11/magazines-periodicals`, `72/album-art`.
  `?terms=` and `/tags/…` style URLs return 404 or 500.
- **Big images:** the use page embeds 1400 px `@2x` media in **two URL shapes**; match both:
  `assets.fontsinuse.com/use-media/<n>/upto-700xauto/<hash>/@2x/jpeg/<file>.jpeg` and
  `assets.fontsinuse.com/static/use-media-items/<a>/<b>/upto-700xauto/<hash>/@2x/<k>.png`.
  Regex: `https://assets\.fontsinuse\.com/(?:static/use-media-items|use-media)/[^" ]*?/upto-700xauto/[^/]+/@2x/[^" ]+?\.(?:jpe?g|png)`.
  The first match is the lead image. Listing thumbs (`/thumb/…/@2x/…`) are only about 440 px, too small.
- **Creator:** on the use page, `href="/designers/<id>/<slug>" title="View all Uses filed under <Name>"`.
  Read the name from the `title` attribute; the link text is wrapped in spans. Some uses list none.
- **Typefaces:** take them from the search-listing block, **not** the use page. The use page also carries
  a site-wide typeface nav, so a naive `/typefaces/` regex returns the same alphabetical list (Acumin,
  Adobe Caslon…) for every use. Named typefaces feed the type direction with facts.
- Pages are UTF-8 but occasionally contain stray bytes, so decode with `errors="replace"`.
- The og:image is a composite card. **Don't use it.**

### typo/graphic posters — typographicposters.com — **reliable via web search**
- Listing and search pages are a JavaScript app; web fetch sees only the heading, and `?s=` returns no
  posters.
- **Route:** web search `site:typographicposters.com <q>`. Results include poster pages
  (`/<designer>/<24-hex-id>`) and designer pages (`/<designer>`).
- **Poster pages** have clean og tags. For example, `og:title` is `"<Title>", <year>, by <Designer> -
  typo/graphic posters`, and `og:image` is `https://images.typographicposters.com/poster/<designer>/…jpg`
  at full size. Parse the creator from the title.
- Designer pages have og:image set to a studio cover, not a poster, so use them only to find poster links.
- Poster images are about **560×800**: enough for the board, under Fonts In Use's 1400. `" for <X>"` at
  the end of an og:title names the client or organiser, not a designer; strip it from the creator. The URL's
  first path segment is the designer's profile (`/studiodobra`), except for organiser accounts such as
  `/100besteplakate` or `/weltformat`.
- Web search returns about 10 results per query, around 5 of them posters, so run 2–3 phrasings to get
  10+ candidates.

### Are.na — api.are.na/v2 — **reliable, but check attribution**
- `GET /v2/search?q=<q>&per=40` returns `channels[]` and `blocks[]`. No key needed.
- `GET /v2/search/channels?q=<q>` works. `GET /v2/channels/<slug>/contents?per=40` lists a channel's
  blocks.
- **`/v2/search/blocks` sits behind a Cloudflare challenge**: don't use it. `/v3/…` needs auth.
- Blocks of class `Image`, `Link` or `Media` carry `image.original.url` (full size) and
  `image.display.url`. `source.url` is where the image came from; `user.full_name` is **who saved it**,
  not who made it. Block page: `https://www.are.na/block/<id>`.
- **Attribution rule.** If `source.url` points at a creator page (Behance, a studio site, Instagram), link
  it and name the creator only if the title or source names them. Otherwise write
  `Creator not stated · saved to Are.na by <user>`. Titles are often filenames
  (`405331192_…_n.jpg`), so replace those with your own factual label and mark the creator unknown.
- The best path is channels: search channels for the brief, pick 2–3 well-curated ones (high `length`,
  on-topic title), then pull their contents.
- **Filter for attributable blocks** before looking: drop titles that are filenames or empty, and drop
  `source.url`s on instagram/pinterest/twitter (reposts rarely name the maker). What survives is often
  titled by a careful curator as `"<Creator>, <Work> (<year>)"` or `"<Work> — <Studio>"`. That is a stated
  credit, so use it, and add `via Are.na, saved by <user>` so the provenance is visible.
- Download `image.large.url`: `original` files run to 10–20 MB.

### BP&O — bpando.org — **reliable**
- WordPress REST works: `BASE=https://bpando.org`. Titles often name the studio ("Studio South merges …");
  when they don't, the post slug usually does (`…-branding-by-studio-mut`).
- Search is fuzzy: `coffee` returned 3 coffee brands in 20. Filter titles and links before downloading.
- Listing thumbnails are 150×150 crops; always use the featured image or og:image.

### Brand New — underconsideration.com/brandnew — **reliable**
- WordPress REST works: `BASE=https://www.underconsideration.com/brandnew`. Titles follow the pattern
  `New Logo and Identity for <Client> by <Studio>`.
- Some posts have no featured media: fall back to `og` on the post page.
- **The REST response starts with PHP warnings** (`<b>Warning</b>: Undefined variable…`) before the
  JSON, so pipe it through `sed -n '/^\[/,$p'` before `jq`.
- Search is loose, and whole-word matching is absent: `coffee` returned 1 coffee identity in 15 (the rest
  were "Friday Likes" roundups). Expect low yield for niche subjects.

### Identity Designed — identitydesigned.com — **reliable**
- WordPress REST works: `BASE=https://identitydesigned.com`. Titles are the client name only. The credit
  is consistent in the post HTML: `Designed by <a href="<studio url>">Studio</a>, City`, so regex the
  `<a>` right after "Designed by". Don't regex the tag-stripped text: inline CSS precedes it and a lazy
  match swallows the stylesheet.
- Each post has 10–30 images. The first few are usually the logo and hero applications. Every post also
  embeds the **same 300×357 site-wide image** (drop it by size, which the screen does). Skip `-NNNxNNN.`
  resized variants.
- Images named `…-low.jpg` are 600 px. Try the same URL without `-low` for a larger file, and keep it if
  it validates.

### The Dieline — thedieline.com — **reliable**
- WordPress REST works: `BASE=https://thedieline.com`. Featured images reach about 2000 px. Agency
  credits are in the post body.

### Logobook — logobook.com — **reliable, marks only**
- Search: `https://logobook.com/?s=<q>`. Categories: `/letter/<x>/`, `/shape/<name>/`, `/object/<x>/`,
  `/nature/<x>/`, `/business/<industry>/`.
- Logo pages (`/logo/<slug>/`) credit `<a href="…/designer/<slug>/">Name</a>` (about half have none).
  The mark itself is **not in an `<img>`**: take it from the JSON-LD, `"contentUrl":"…/uploads/…_logo.svg"`.
- Marks are black SVGs on transparent. The screen passes them as `ok-vector`; to view them, rasterise
  with `qlmanage -t -s 900 -o <dir> *.svg` (macOS) or `rsvg-convert`. The board puts SVG/PNG on a white
  panel so they stay visible in dark mode.
- The collection skews mid-century European. It is excellent for reduced symbol logic, not for current
  trends.

### Typewolf — typewolf.com — **reliable, web/type only**
- Static HTML. `https://www.typewolf.com/site-of-the-day` lists `<img src="/assets/img/sotd/<date>.png"
  srcset="…-2x.png 2x" alt="<Site>">` at about 984×590 (use the `-2x`). Each entry names the fonts.
- There's no search; browse or `site:typewolf.com <q>`.

### Awwwards — awwwards.com — **reliable**
- `https://www.awwwards.com/websites/?text=<q>` returns about 30 `/sites/<slug>` links (curl).
- Site pages have `og:image` at `assets.awwwards.com/awards/submissions/…jpg`, and `og:title`
  `<Site> - Awwwards SOTD`. Web fetch on the listing also returns the creator per site.

### SiteInspire — siteinspire.com — **web fetch only**
- curl gets **429**; web fetch works. Ask it for per-site name, studio, `/website/<id>-<slug>` link and
  preview URL. Previews live on `r2.siteinspire.com/cdn-cgi/image/width=1920,…/<file>.jpg`.

### Minimal Gallery — minimal.gallery — **reliable**
- WordPress REST works: `BASE=https://minimal.gallery`. Search terms match site names and descriptions,
  so use site-type words (`portfolio`, `studio`, `agency`), not style words. Featured images are full
  screenshots.

### Behance — behance.net — *blocked*
- Search, gallery and project pages all return **403** to curl and to web fetch. `site:behance.net` web
  search mostly returns search and "hire" pages, not projects.
- **Don't fight it.** Log it, and reach Behance work through Are.na blocks whose `source.url` is a Behance
  gallery. The image comes from Are.na, and the link goes to the Behance project.

### Dribbble — dribbble.com — *blocked*
- Every page returns **HTTP 202 with `x-amzn-waf-action: challenge`**, an AWS WAF bot challenge, and web
  fetch sees an empty page. Log it and skip. The Are.na route applies here too.

### Land-book — land-book.com — *blocked*
- **403** to curl and web fetch.

### Godly — godly.website → recent.design — *low yield*
- godly.website now 301-redirects to `recent.design`, a JavaScript-rendered mixed gallery. Web fetch
  returns item names without links or images. Use it only as a last resort for web or motion briefs.
