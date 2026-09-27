---
name: visual-inspiration
description: Find strong visual and graphic design references for a design task and build a local HTML reference board with an analysis. Covers logos, brand identities, flyers and posters, packaging, typography, illustration, websites, mobile/app UI and social graphics. Picks the 3–5 best-fit curated sources for the medium (Fonts In Use, typo/graphic posters, BP&O, Brand New, Identity Designed, The Dieline, Awwwards, SiteInspire, Minimal Gallery, Are.na and others), downloads 12–20 credited references into ./inspiration/<slug>/, and writes board.html with recurring patterns, 3–5 creative directions, and a palette and type direction. Use when asked to "find inspiration for", "find references for a logo / flyer / poster / packaging / website / app", "make a moodboard for", "build a reference board", or "what are good examples of" a design piece. For Pinterest specifically, use the pinterest skill; for app screens and UX flows, use the Mobbin MCP or refero-design.
license: MIT
compatibility: Needs web search and web fetch tools, a shell with bash, curl, jq and file, and uv (or any Python 3 with Pillow), plus open network access. Not usable in the Claude chat sandbox, which has no network access.
---

# Visual inspiration

**Purpose: a credited, curated reference board for a design brief, and a point of view on it.** Twelve to
twenty strong references from the sources designers actually trust. You say what they have in common and
which directions they open up, so the user starts designing from a position instead of a blank page.

One pass: brief → pick sources → collect → download → look → keep the best → board + analysis.

## Boundaries — read once, apply always

- **Inspiration, not copying.** Every reference is third-party copyrighted work. It informs direction.
  It is never traced, never shipped inside a deliverable, and never fed to an image generator as a
  style target. Directions describe principles ("oversized condensed grotesk bleeding off two edges"),
  never "do what X did".
- **Credit everyone, link everything.** Every card carries the creator (or studio) and the original page.
- **Never make anything up.** Every title, creator, URL and image in the board comes from a page or API
  response you actually retrieved in this run. If a creator is not stated, write `Creator not stated`
  and say where it came from (for example "saved to Are.na by …"). An empty slot beats a guessed name.
- **Images stay local.** The board is a file on disk. Never publish it as an Artifact, upload it, or host
  it: it embeds other people's work.
- **This skill writes into the project, on purpose.** Output goes to `./inspiration/<slug>/` because the
  board is meant to be reopened while designing. Nothing else in the project is touched.
- **Research scale.** Three to five sources, one pass, around 30–40 candidates. No crawling, no
  scheduled runs.
- **Respect bot walls.** Some sites (Behance, Dribbble, Land-book) answer automated requests with a 403
  or a bot challenge. Log it and move on. Never retry with spoofed headers, proxies, third-party
  "reader" services or a browser to get past a block.
- **Tools.** Web search and web fetch do the discovery. `curl` downloads images and reads a page's
  raw `<meta og:*>` tags when web fetch's summary drops them. No paid APIs, no API keys, no logins.

## Step 1 — Clarify the brief, without an interview

From the request, pull out:

- **Medium** — what is being designed: logo, identity, poster/flyer, packaging, type, illustration, web,
  app, social.
- **Style keywords** — 2–4 mood or style words ("bold", "editorial", "brutalist", "warm minimal").
- **Industry or subject** — "design meetup", "coffee roaster", "fintech".
- **Constraints** — colours, formats, must-haves.

If the brief is vague, infer sensible keywords from what is given and say so. Ask a question only if the
**medium** genuinely cannot be inferred. State the brief back in one line and derive a short
kebab-case slug (`editorial-flyer-sydney-design-meetup`).

```bash
OUT="./inspiration/<slug>"; W="<scratchpad>/visual-inspiration/<slug>"
mkdir -p "$OUT/images" "$W/cand"
```

`$W` is working space for candidates and rejects, and is never inside the project. Use the host's session
scratchpad; fall back to `mktemp -d`. Only kept images and the board land in `$OUT`.

## Step 2 — Pick 3–5 sources for the medium

Use the medium table in **`references/sources.md`**. It also records how each source actually behaves
when fetched: which have an API, which need web search, and which block. Choose the 3–5 that fit the
brief best, not all of them. Prefer sources marked **reliable**, and add one **fallback-only** source at
most. Say which you picked in one line.

Mobbin is out of scope; the user runs it separately.

## Step 3 — Build queries

Use one query per source, in the language people tag with: **an artifact noun plus 1–2 style words**
(`editorial poster`, `typographic event poster`, `coffee packaging minimal`). Past about four words,
matching degrades. Adjective soup with no artifact noun (`bold modern clean`) returns noise. Brief
language (`flyer for a Sydney design meetup`) matches nothing, because nobody tags their work that way.
Put the subject and location into the analysis, not the query.

Run a second, adjacent query on a strong source only if the first comes back thin.

## Step 4 — Collect candidates

For each chosen source, work down its ladder in `references/sources.md` (API → listing page →
`site:` web search → project pages) until you have **6–12 candidates from that source**. Record each one
as a line of `$W/candidates.jsonl`:

```json
{"id":"c07","source":"Fonts In Use","title":"park:session poster","creator":"Maa Luvs","creator_url":"https://fontsinuse.com/designers/4527/maa-luvs","url":"https://fontsinuse.com/uses/10326/park-session-poster-1","image_url":"https://assets.fontsinuse.com/use-media/…/@2x/…jpeg"}
```

- `url` is the project page on the source site. `image_url` must be a URL you saw in a response. If you
  only have the project page, curl it for `og:image` (the recipe is in `references/sources.md`).
- When web fetch is used for extraction, ask it for **exact URLs as they appear**, and treat any URL it
  returns as a claim to verify: the download step checks it.
- **If a source fails** (403, bot challenge, empty JS shell, zero results), write one line to
  `$W/run-log.md` (`Behance — 403 on search and project pages; skipped`) and go to the next source.
  Never stop the run for one source. If fewer than three sources produced anything, add the next best
  source from the table.

## Step 5 — Download and screen mechanically

Follow **`references/images.md`**:

1. Download every candidate to `$W/cand/<id>.<ext>` with curl (browser UA, `</dev/null` in the loop).
2. **Validate by MIME type, never by size.** CDNs return HTML or XML error bodies under image names.
3. Run the screening script. It reports dimensions, drops anything whose long edge is under **600 px**
   (400 px for logos) and flags near-duplicates by perceptual hash. Keep the higher-resolution copy of
   each duplicate pair.

## Step 6 — Look at every survivor, and choose

Actually view each surviving image. This is the curation step. Its quality decides whether the board is
worth opening. For a first pass over 30–40 candidates, a labelled contact sheet (12 per sheet, with ids
under each image) is efficient. Then open the likely keepers individually, so labels describe what is
really there.

Keep **12–20**. Drop anything that is:

- **off-brief** — wrong medium, or a mockup or stock template instead of real work;
- **weak** — generic, dated in a way that doesn't serve the brief, or illegible at board size;
- **redundant** — a third near-identical take on the same idea;
- **not the work** — a site banner, a portrait of the designer, a typeface specimen instead of a use.

Aim for **range within the brief**: the set should support several distinct directions, not twenty
variations of one. Write a factual one-line label for each keeper ("two-colour condensed-type poster,
type bleeding off three edges"), and cap any single source at about 40% of the board.

Copy the keepers into `$OUT/images/` as `NN-source-slug.ext` (numbered in board order, grouped loosely
by direction).

## Step 7 — Analyse

Sample the palette with the script in `references/images.md` (run on the kept images). Then write:

- **Recurring patterns** — one or two sentences each for **layout, type, colour, composition, texture,
  motion** (motion only where the medium has it; say "n/a — static print" otherwise). Cite reference
  numbers.
- **3–5 creative directions** — distinct, not variations of each other. Each gets a short name, 2–3
  sentences on what it is and why it suits this brief, and the reference numbers that support it. Every
  direction needs at least two references.
- **Palette** — 5–7 hex values with role and rough proportion, each traced to the references it came
  from. Label them **sampled, approximate**.
- **Type direction** — classification, weight, width, case, spacing and hierarchy. Name a typeface only
  when a source names it (Fonts In Use does) or it is unmistakable. Otherwise describe it, and suggest
  2–3 comparable families as options, clearly labelled as suggestions.
- **What this set can't tell you** — gaps: sources that failed, a missing medium, few examples of a
  direction.

## Step 8 — Build the board

Write `$OUT/refs.json` using the schema in **`references/board.md`**, then run the generator from that
file. It writes `$OUT/board.html`: the analysis up top, palette swatches, direction cards linking to their
references, then a responsive grid of numbered cards (image, title, creator, link to the original). It
works offline with relative image paths and supports light and dark mode.

Check it before delivering. Open the board in a browser or preview pane if the host has one, and confirm
every image renders and the numbers match the analysis.

## Step 9 — Deliver

1. **Show the board** by whatever mechanism the host provides for displaying a local file. In Claude Code
   that is `SendUserFile` with `display: render` on `board.html`. If the host cannot display files, print
   the absolute path. Never proceed as though the user has seen it.
2. **In chat, in the same reply**: the one-line brief, the patterns, the directions (with reference
   numbers), the palette and the type direction. Keep it scannable.
3. **References** as `n — title (creator)` links to the original pages, never bare numbers.
4. **Run log** — one line per source that failed or was skipped, and why.
5. Offer one next step: "say **more like 4** or **push direction B** and I'll search around it."

**"More like 4"**: search again from that reference's own source, creator or tags (the designer's other
work, the same Fonts In Use typeface, the Are.na channel it was saved to). Continue numbering from the
current high-water mark and regenerate the board with the added cards.
