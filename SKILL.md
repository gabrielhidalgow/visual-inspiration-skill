---
name: visual-inspiration
description: Quick, credited visual references for a design task, shown in the session. Picks the 3–4 curated sources that fit the medium (Fonts In Use, typo/graphic posters, BP&O, Brand New, Identity Designed, The Dieline, Logobook, Awwwards, SiteInspire, Minimal Gallery, Are.na and others), chooses the 9 strongest references, and shows them as one numbered contact sheet in the conversation with a short direction (2–3 directions, palette, type) and credited links. Writes nothing into the project. Covers logos, brand identities, flyers and posters, packaging, typography, illustration, websites and social graphics. Use when asked to "find inspiration for", "find references for a logo / flyer / poster / packaging / website", "moodboard for", or "what are good examples of" a design piece. For Pinterest specifically, use the pinterest skill; for app screens and UX flows, use the Mobbin MCP or refero-design.
license: MIT
compatibility: Needs web search and web fetch tools, a shell with bash, curl, jq and file, and uv (or any Python 3 with Pillow), plus open network access. SVG rasterising uses macOS qlmanage or rsvg-convert. Not usable in the Claude chat sandbox, which has no network access.
---

# Visual inspiration

**Purpose: visual grounding for the work in progress, fast.** Nine strong, credited references from the
sources designers trust, on one sheet, with just enough direction to act on, so the project moves
straight on.

One reply: brief → sources → collect → look → **9 on one sheet** → short direction + links.

## Boundaries — read once, apply always

- **Nothing is written into the project.** Candidates, rejects and the sheet all live in the session
  scratchpad. No reference folder, no HTML, no markdown. The deliverable is your reply.
- **9 references, one sheet, direction in the same reply.** Don't stop to ask which images they like;
  they can say "more like 4" afterwards.
- **Inspiration, not copying.** Every reference is third-party copyrighted work. It informs direction,
  and is never traced, never shipped inside a deliverable, and never fed to an image generator as a
  style target. Describe principles, never "do what X did".
- **Credit everyone, link everything.** Each number maps to a title, a creator and the original page.
- **Never make anything up.** Every title, creator, URL and image comes from a response you retrieved in
  this run. If a creator isn't stated, say `creator not stated`. An empty slot beats a guessed name.
- **Research scale.** 3–4 sources × 4–6 candidates, about 15–24 in total. One image per project: the one
  where the work is the subject.
- **Respect bot walls.** Behance, Dribbble and Land-book block automation (403 / bot challenge). Skip
  them. Never retry with spoofed headers, proxies, "reader" services or a browser.
- **Tools.** Web search and web fetch for discovery; `curl` for images and raw `<meta og:*>` tags. No paid
  APIs, keys or logins.

## Step 1 — Brief, without an interview

Pull out the **medium** (logo, identity, poster/flyer, packaging, type, illustration, web, social), 2–3
**style words**, the **subject or industry**, and any **constraints**. If it's vague, infer and say so.
Ask only if the medium truly can't be inferred. If there's a project in progress, anchor to it: what the
reference is *for*.

```bash
W="<scratchpad>/visual-inspiration/<slug>"; mkdir -p "$W/cand"
```

Use the host's session scratchpad; fall back to `mktemp -d`. Never write inside the project.

## Step 2 — Pick 3–4 sources

Use the medium table in **`references/sources.md`**, which also records how each site actually behaves
(API, web search only, or blocked). Prefer sources marked **reliable**. Mobbin is out of scope.

## Step 3 — Queries

Use **an artifact noun plus 1–2 style words** (`coffee logo`, `editorial poster`, `wine label minimal`),
at most about four words. Brief language ("flyer for a Sydney design meetup") matches nothing. The
subject and location go into your direction, not the query.

## Step 4 — Collect ~15–24 candidates

Work down each source's ladder in `references/sources.md` (API → listing → `site:` search → project
page) until you have **4–6 per source**. One line each in `$W/candidates.jsonl`:

```json
{"id":"c07","source":"Fonts In Use","title":"Two Bean Coffee","creator":"Two Bean Coffee","url":"https://fontsinuse.com/uses/65315/two-bean-coffee","image_url":"https://assets.fontsinuse.com/…/@2x/…"}
```

`image_url` must be a URL you saw in a response. Pre-filter on titles before downloading, so off-brief
projects never get fetched. If a source fails, note it in one line and move on; if fewer than three
sources produced anything, add the next best one.

## Step 5 — Download and screen

Follow **`references/images.md`**: curl into `$W/cand/` (with `</dev/null` in the loop), **validate by
MIME type, never by size**, then run the screen (resolution: 600 px long edge, or 400 for logos; near-
duplicates).

## Step 6 — Look, and pick 9

Actually look. A quick labelled contact sheet of all survivors is the efficient first pass (see
`references/contact-sheet.md` for the rasterise step), then open borderline ones individually.

Pick **9** that give **range across 2–3 directions**, with no more than 4 from one source. Drop anything
off-brief, weak, redundant, or where the work isn't the subject (storefronts, merch, portraits, stock
mockups). Write the 9 ids, in the order to number them, to `$W/order.txt`.

## Step 7 — Build and show the sheet

Build it with **`references/contact-sheet.md`** (3×3, numbered, SVG and transparent marks on white), then
**show it inline**: in Claude Code, `SendUserFile` with `display: render` on `$W/sheet.jpg`. If the host
can't display images, print the path. Never proceed as though the user has seen it.

## Step 8 — The reply: short, in the same message as the sheet

Sample the palette with the script in `references/images.md` (on the 9 picks), then write only this:

```
**Brief:** minimalist coffee brand logo (roaster/café inferred)

**Directions**
- **Bean as letterform:** a bean doing structural work inside the initial (1, 2, 3)
- **Quiet spaced wordmark:** one colour, wide-tracked caps, no symbol (4, 5, 6)
- **Monogram roundel:** initials in a thin-line circle, used as a stamp (7, 8)

**Palette** (approx.): `#0A0A0A` · `#F9F9F7` · `#E2D5BD` · `#0D4FA0`
**Type:** wide-tracked caps in slab, mono or geometric sans; Fonts In Use names TT Fors (4), FK Grotesk (6)

1 — [Danesi](url) (Sergio Salaroli)
2 — [Café Palheta](url) (creator not stated)
…
9 — [Canyon Coffee](url) (Studio L'Ami)

Say **more like 4** and I'll search around it.
```

- Keep each direction to one line citing numbers, with at least 2 references per direction.
- Name a typeface only when a source names it (Fonts In Use does), or it is unmistakable.
- Add one line for failed sources **only if one failed and it matters** (e.g. "Behance blocked, so the
  set leans European").
- Then connect the direction to the actual files or components in play, and carry on with the work if
  that was asked. Don't wait for approval of the direction.

## Step 9 — "More like N"

Search from that reference's own context: the designer's other work, the same Fonts In Use typeface, the
Are.na channel it was saved to, or the same Logobook category. Append the new ids to `order.txt`, build a
second sheet with `start_n` 10 (then 19…), and reply in the same short format.
