# visual-inspiration-skill — working notes

One skill, and the repo root **is** the skill: `SKILL.md` sits at the top level. This checkout is also
the live installation (see Layout).

Public: https://github.com/gabrielhidalgow/visual-inspiration-skill

## Layout — there is no build, deploy, or sync step

```
~/Desktop/Projects/visual-inspiration skill/
  visual-inspiration-skill/          ← this repo, and the skill itself (edit here)
  visual-inspiration-experiments/    ← test runs and scratch, NOT in the repo (see below)
~/.claude/skills/visual-inspiration  →  symlink to the repo root   ← what /visual-inspiration loads
```

The symlink means both paths are the same file. An edit here is live on the next invocation, with no
reinstall and no copy. The one non-automatic step is a stale clone: pull first if you edited on github.com
or another machine.

```
SKILL.md                  the workflow (brief → sources → collect → download → curate → analyse → board)
references/sources.md     medium→source table + measured per-site behaviour and extraction recipes
references/images.md      download, MIME validation, screening (size + dHash dedupe), palette sampling
references/board.md       refs.json schema + the board generator
```

## Experiments

They live **outside this repo**, in `../visual-inspiration-experiments/`, one dated folder each with a
`NOTES.md`. The symlink points at the repo root, so anything beside `SKILL.md` is inside the skill
directory and would be read as part of it. Git-ignoring it doesn't help. The same lesson came from the
pinterest skill. Do not recreate `experiments/` here.

## Testing a change

Run the skill on a real brief, with the working directory set to a new dated folder under
`../visual-inspiration-experiments/`, so `./inspiration/<slug>/` lands there and not in some project. The
reference run is `2026-09-28-sydney-design-meetup-flyer` ("bold, editorial-style flyer for a Sydney design
meetup").

Test the scripts as written by extracting them from the reference files rather than retyping them:

```bash
python3 -c "import re,pathlib,sys;b=re.findall(r\"<<'PY'[^\n]*\n(.*?)\nPY\n\",pathlib.Path(sys.argv[1]).read_text(),re.S);[pathlib.Path(f'/tmp/snip{i}.py').write_text(x) for i,x in enumerate(b)]" references/images.md
# snip0 = screen, snip1 = palette; board.md has one block = the generator
```

To view a board in a browser pane, serve the folder (`python3 -m http.server`). `file://` is often blocked.

**Re-probe `sources.md` when a source misbehaves**, not on a schedule. Probing means one curl plus one web
fetch per site, not a crawl.

## Invariants — do not break these without a reason

- **Only retrieved data goes on the board.** Title, creator, URL and image all come from a response in
  this run. `Creator not stated` beats a plausible guess. This is the whole trust model of the board.
- **Respect bot walls.** Behance, Dribbble and Land-book block automation (403 / AWS WAF challenge). The
  skill logs them and moves on. It never retries with header spoofing, proxies, "reader" services or a
  browser. The legitimate route to that work is Are.na blocks whose `source.url` points back to it.
- **Images stay local; the board is never published.** It embeds third-party work.
- **Writes only `./inspiration/<slug>/`.** This deliberately differs from pinterest, which writes nothing,
  because the user asked for a board to reopen while designing. Rejected candidates stay in the scratchpad.
- **Curate by looking.** The mechanical screen (MIME, size, dHash) only removes junk. The 41→20 cut in the
  reference run was made by eye, and that is where the value is.
- **Validate by MIME type, never by size.** A lesson paid for in the pinterest skill.

## Gotchas already paid for

- **Fonts In Use has two media URL shapes** (`use-media/…` and `static/use-media-items/…`). Matching only
  one silently loses about 70% of images.
- **Fonts In Use use pages carry a site-wide typeface nav.** A naive `/typefaces/` regex returns the same
  alphabetical list (Acumin, Adobe Caslon…) for every use. Take typefaces from the search listing.
- **Are.na `/v2/search/blocks` sits behind a Cloudflare challenge**, while `/v2/search` and
  `/v2/search/channels` work. Easy to misread as "Are.na is down".
- **godly.website now redirects to recent.design**, which renders with JavaScript.
- **SiteInspire returns 429 to curl but works through web fetch.**
- **Lazy-loaded images without `width`/`height` make CSS-columns masonry reflow** as you scroll. The
  generator writes dimensions from `screen.tsv`, so keep `w`/`h` in `refs.json`.
- **`curl` inside `while read` eats stdin.** `</dev/null` is load-bearing.
- **Black-on-transparent marks vanish on a dark board.** Logobook SVGs rendered invisible in dark mode
  until the generator gave `.svg`/`.png` cards a white `.flat` panel. Check a logo board in dark mode.
- **Not every board is 4:5 posters.** Logo runs mix SVG marks with photos of applications. Prefer the
  image where the mark is the subject; storefronts and merch shots were most of the cuts in the coffee run.

## Committing

Edit → live run in the experiments folder → commit → push. The repo is the source of truth.
