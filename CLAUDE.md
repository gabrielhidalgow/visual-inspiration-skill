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

The symlink means both paths are the same file, so an edit is live on the next invocation. The one
non-automatic step is a stale clone: pull first if you edited elsewhere.

```
SKILL.md                    the workflow: brief → sources → collect (curated first) → review thumbs → 9 → sheet → reply
references/sources.md       medium→source table + measured per-site behaviour and extraction recipes
references/images.md        thumbnails, MIME validation, review sheet (dHash dupes), full-size for picks, palette
references/contact-sheet.md SVG rasterise + the 3×3 numbered sheet script
```

## Why the output is one sheet, not a board

Version 1 wrote `./inspiration/<slug>/board.html` + `refs.json` + an images folder, with a long analysis
and 12–20 references from about 60 candidates. After two live runs Gabriel judged it too heavy for
in-session use: he wants a visual idea in the project session, not a document. The HTML also broke in the
Claude Code preview, where relative image paths don't resolve. The redesign, mirroring the pinterest
skill: **9 references on one inline sheet, about 10 lines of direction plus credited links, and nothing
written into the project.** Don't drift back towards a board without a new reason.

## Experiments

They live **outside this repo**, in `../visual-inspiration-experiments/`, one dated folder each with a
`NOTES.md`. The symlink points at the repo root, so anything beside `SKILL.md` would be read as part of
the skill. Git-ignoring it doesn't help (same lesson as pinterest). Don't recreate `experiments/` here.

## Testing a change

Run the skill on a real brief with the working directory set to a new dated experiments folder, then
confirm it created **no files there**. For sheet or layout changes, rebuild from candidates already in a
session scratchpad instead of fetching again.

Extract scripts from the reference files rather than retyping them. Each file's `<<'PY'` blocks, in
order, are: images.md → review sheet, palette; contact-sheet.md → final sheet.

```bash
python3 -c "import re,pathlib,sys;b=re.findall(r\"<<'PY'[^\n]*\n(.*?)\nPY\n\",pathlib.Path(sys.argv[1]).read_text(),re.S);[pathlib.Path(f'/tmp/snip{i}.py').write_text(x) for i,x in enumerate(b)]" references/images.md
```

**Re-probe `sources.md` when a source misbehaves**, not on a schedule: one curl plus one web fetch per
site, not a crawl.

## Invariants — do not break these without a reason

- **Nothing written into the project; 9 refs; one sheet; direction in the same reply.**
- **Only retrieved data.** Title, creator, URL and image all come from a response in this run.
  `creator not stated` beats a plausible guess. This is the whole trust model.
- **Respect bot walls.** Behance, Dribbble and Land-book block automation (403 / AWS WAF challenge). Skip
  them. Never retry with spoofing, proxies, "reader" services or a browser. Are.na saves that link back
  to them are the legitimate route.
- **Never publish the sheet.** It shows third-party work.
- **Curate by looking.** The screen (MIME, size, dHash) only removes junk; the pick is made by eye.
- **Validate by MIME type, never by size.**

## Gotchas already paid for

- **WordPress REST search sorts by date unless told otherwise.** Always `orderby=relevance`: on Brand New
  `coffee`, date order gave 0/8 relevant and relevance gave 8/8. Missing this made BP&O and Brand New look
  like weak sources in the first runs, when they weren't.
- **Fonts In Use's staff-picks filter is `&filters=staff-picks-only`**, hidden in a base64 `data-js-link`.
- **Choosing from titles misses good work.** Candidates are now picked from a thumbnail review sheet, and
  full-size images are fetched only for the 9 picks.

- **Fonts In Use has two media URL shapes** (`use-media/…` and `static/use-media-items/…`). Matching only
  one silently loses about 70% of images.
- **Fonts In Use use pages carry a site-wide typeface nav.** A naive `/typefaces/` regex returns the same
  alphabetical list for every use. Take typefaces from the search listing.
- **Are.na `/v2/search/blocks` sits behind a Cloudflare challenge**; `/v2/search` and
  `/v2/search/channels` work.
- **godly.website redirects to recent.design** (JS-rendered). **SiteInspire returns 429 to curl** but
  works through web fetch.
- **`curl` inside `while read` eats stdin.** `</dev/null` is load-bearing.
- **Black-on-transparent marks vanish on a dark background.** Logobook SVGs were invisible on the v1
  dark board. The sheet composites every image onto white before pasting.
- **Pick the image where the work is the subject.** Logo projects come with storefronts, merch and
  portraits, which were most of the cuts in the coffee run. One image per project keeps the candidate count low.
- **Nested heredocs:** a `<<'PY'` inside another `<<'PY'` ends the outer one early. Use the Edit tool or a
  script file when editing the reference docs.

## Committing

Edit → live run in the experiments folder → commit → push. The repo is the source of truth.
