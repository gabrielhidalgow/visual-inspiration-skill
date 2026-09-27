# The reference board

A single self-contained `board.html` next to an `images/` folder. It works offline, with relative paths
and no external requests. It is generated from `refs.json`, so every card traces back to a record.

## refs.json

```json
{
  "brief": {
    "title": "Bold, editorial flyer for a Sydney design meetup",
    "medium": "Flyer / poster",
    "keywords": ["bold", "editorial", "typographic"],
    "context": "Design community event, Sydney",
    "date": "2026-09-28",
    "sources": ["Fonts In Use", "typo/graphic posters", "Are.na"]
  },
  "summary": "Two or three sentences: what the set says overall.",
  "patterns": [
    {"label": "Layout", "text": "… (3, 7, 12)"},
    {"label": "Type", "text": "…"},
    {"label": "Colour", "text": "…"},
    {"label": "Composition", "text": "…"},
    {"label": "Texture", "text": "…"},
    {"label": "Motion", "text": "n/a — static print"}
  ],
  "directions": [
    {"name": "Type as architecture", "text": "2–3 sentences.", "refs": [1, 4, 9]}
  ],
  "palette": [
    {"hex": "#F2EFE8", "role": "Ground", "share": "~50%", "refs": [2, 5, 8]}
  ],
  "type": "Type direction paragraph. Name families only when sourced, and label suggestions as suggestions.",
  "gaps": "What this set can't tell you.",
  "log": ["Behance — 403 on all pages; skipped", "Dribbble — AWS WAF challenge; skipped"],
  "refs": [
    {
      "n": 1,
      "file": "images/01-fonts-in-use-park-session.jpg",
      "title": "park:session poster",
      "label": "Two-colour BMX contest poster, stacked condensed caps",
      "creator": "Maa Luvs",
      "creator_url": "https://fontsinuse.com/designers/4527/maa-luvs",
      "source": "Fonts In Use",
      "url": "https://fontsinuse.com/uses/10326/park-session-poster-1",
      "w": 1400, "h": 1981
    }
  ]
}
```

- `n` values are unique and match the file prefix and every number cited in the analysis.
- `creator` is exactly as retrieved, or `"Creator not stated · saved to Are.na by <user>"`. Never guess.
- `creator_url` and `label` are optional. `share` is free text (`"~50%"`, `"accent"`).
- `w` / `h` are the pixel dimensions from `screen.tsv`. They are optional, but include them: the board uses them to
  reserve each image's space, so lazy-loaded images don't make the masonry columns jump while scrolling.

## Generator

Stdlib only; no Pillow needed.

```bash
python3 - "$OUT/refs.json" "$OUT/board.html" <<'PY'
import sys, json, html, pathlib
data = json.loads(pathlib.Path(sys.argv[1]).read_text())
out = pathlib.Path(sys.argv[2])
e = lambda s: html.escape(str(s or ""), quote=True)
b = data["brief"]
refs = {r["n"]: r for r in data["refs"]}

def ink_on(hex_):
    h = hex_.lstrip("#"); r, g, bb = (int(h[i:i + 2], 16) / 255 for i in (0, 2, 4))
    lum = lambda c: c / 12.92 if c <= 0.03928 else ((c + 0.055) / 1.055) ** 2.4
    return "#111" if 0.2126 * lum(r) + 0.7152 * lum(g) + 0.0722 * lum(bb) > 0.45 else "#fff"

def chips(ns):
    return "".join(
        f'<a class="chip" href="#ref-{n}" title="{e(refs[n]["title"])}">'
        f'<img src="{e(refs[n]["file"])}" alt="" loading="lazy"><span>{n}</span></a>'
        for n in ns if n in refs)

patterns = "".join(f'<div class="pat"><dt>{e(p["label"])}</dt><dd>{e(p["text"])}</dd></div>'
                   for p in data.get("patterns", []))
directions = "".join(
    f'<article class="dir"><p class="dir-k">Direction {chr(65 + i)}</p><h3>{e(d["name"])}</h3>'
    f'<p>{e(d["text"])}</p><div class="chips">{chips(d.get("refs", []))}</div></article>'
    for i, d in enumerate(data.get("directions", [])))
palette = "".join(
    f'<li><span class="sw" style="background:{e(p["hex"])};color:{ink_on(p["hex"])}">{e(p["hex"].upper())}</span>'
    f'<span class="sw-role">{e(p.get("role"))}</span><span class="sw-meta">{e(p.get("share"))}'
    f'{" · refs " + ", ".join(str(n) for n in p.get("refs", [])) if p.get("refs") else ""}</span></li>'
    for p in data.get("palette", []))
log = "".join(f"<li>{e(x)}</li>" for x in data.get("log", []))

def dims(r):   # reserve space so lazy images don't reflow the masonry columns
    return f' width="{int(r["w"])}" height="{int(r["h"])}"' if r.get("w") and r.get("h") else ""

def card(r):
    creator = (f'<a href="{e(r["creator_url"])}" target="_blank" rel="noopener">{e(r["creator"])}</a>'
               if r.get("creator_url") else e(r.get("creator") or "Creator not stated"))
    label = f'<p class="label">{e(r["label"])}</p>' if r.get("label") else ""
    return (f'<figure class="card" id="ref-{r["n"]}"><a class="img" href="{e(r["file"])}" target="_blank">'
            f'<img src="{e(r["file"])}" alt="{e(r.get("label") or r["title"])}"{dims(r)} loading="lazy"></a>'
            f'<figcaption><span class="n">{r["n"]}</span><div><h4>{e(r["title"])}</h4>'
            f'<p class="by">{creator}</p>{label}<p class="src">{e(r["source"])} · '
            f'<a href="{e(r["url"])}" target="_blank" rel="noopener">View original ↗</a></p></div></figcaption></figure>')

cards = "".join(card(r) for r in sorted(data["refs"], key=lambda r: r["n"]))
meta = " · ".join(x for x in [b.get("medium"), ", ".join(b.get("keywords", [])), b.get("context")] if x)

out.write_text(f"""<!doctype html>
<html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>{e(b["title"])} — references</title>
<style>
:root{{--bg:#f4f3ef;--paper:#fff;--ink:#141414;--mute:#6b6a66;--line:#dedcd5;--accent:#141414;color-scheme:light}}
@media (prefers-color-scheme:dark){{:root{{--bg:#111110;--paper:#1b1b1a;--ink:#ededea;--mute:#9a9994;--line:#2e2e2c;--accent:#ededea;color-scheme:dark}}}}
*{{box-sizing:border-box}}html{{-webkit-text-size-adjust:100%}}
body{{margin:0;background:var(--bg);color:var(--ink);font:15px/1.55 ui-sans-serif,system-ui,-apple-system,"Helvetica Neue",Arial,sans-serif}}
a{{color:inherit}}
.wrap{{max-width:1480px;margin:0 auto;padding:48px 32px 80px}}
header{{border-bottom:2px solid var(--ink);padding-bottom:28px;margin-bottom:40px}}
.eyebrow{{font:600 12px/1 ui-monospace,SFMono-Regular,Menlo,monospace;letter-spacing:.08em;text-transform:uppercase;color:var(--mute);margin:0 0 18px}}
h1{{font-size:clamp(34px,5.4vw,76px);line-height:.98;letter-spacing:-.035em;margin:0 0 18px;max-width:18ch;font-weight:750}}
.meta{{color:var(--mute);margin:0}}
.summary{{font-size:clamp(18px,1.6vw,22px);line-height:1.45;max-width:62ch;margin:0 0 48px;letter-spacing:-.01em}}
h2{{font:600 12px/1 ui-monospace,SFMono-Regular,Menlo,monospace;letter-spacing:.08em;text-transform:uppercase;color:var(--mute);margin:0 0 16px;padding-top:14px;border-top:1px solid var(--line)}}
section{{margin-bottom:52px}}
dl.pats{{display:grid;grid-template-columns:repeat(auto-fit,minmax(260px,1fr));gap:20px 32px;margin:0}}
.pat dt{{font-weight:650;margin-bottom:4px}}.pat dd{{margin:0;color:var(--ink);opacity:.86}}
.dirs{{display:grid;grid-template-columns:repeat(auto-fit,minmax(280px,1fr));gap:16px}}
.dir{{background:var(--paper);border:1px solid var(--line);border-radius:6px;padding:20px 20px 16px;display:flex;flex-direction:column}}
.dir-k{{font:600 11px/1 ui-monospace,Menlo,monospace;letter-spacing:.08em;text-transform:uppercase;color:var(--mute);margin:0 0 10px}}
.dir h3{{font-size:21px;line-height:1.15;letter-spacing:-.02em;margin:0 0 8px}}.dir p{{margin:0 0 16px}}
.chips{{display:flex;flex-wrap:wrap;gap:6px;margin-top:auto}}
.chip{{position:relative;display:block;width:52px;height:52px;border-radius:4px;overflow:hidden;background:var(--line);outline:1px solid var(--line)}}
.chip img{{width:100%;height:100%;object-fit:cover;display:block}}
.chip span{{position:absolute;left:3px;bottom:3px;font:700 10px/1 ui-monospace,Menlo,monospace;background:var(--ink);color:var(--bg);padding:3px 4px;border-radius:2px}}
.chip:hover,.chip:focus-visible{{outline:2px solid var(--accent)}}
.two{{display:grid;grid-template-columns:minmax(0,1.1fr) minmax(0,1fr);gap:40px}}
ul.pal{{list-style:none;margin:0;padding:0;display:grid;grid-template-columns:repeat(auto-fill,minmax(130px,1fr));gap:12px}}
.sw{{display:flex;align-items:flex-end;height:92px;border-radius:4px;padding:8px;font:600 12px/1 ui-monospace,Menlo,monospace;box-shadow:inset 0 0 0 1px rgba(128,128,128,.25)}}
.sw-role{{display:block;font-weight:600;margin-top:8px}}.sw-meta{{display:block;color:var(--mute);font-size:13px}}
.note{{color:var(--mute);font-size:13px;margin:10px 0 0}}
details{{margin-top:18px;color:var(--mute)}}details ul{{margin:8px 0 0;padding-left:18px}}
.grid{{columns:4 280px;column-gap:20px}}
.card{{break-inside:avoid;margin:0 0 20px;background:var(--paper);border:1px solid var(--line);border-radius:6px;overflow:hidden;scroll-margin-top:24px}}
.card:target{{outline:3px solid var(--accent);outline-offset:2px}}
.card .img{{display:block;background:var(--line)}}.card img{{display:block;width:100%;height:auto}}
figcaption{{display:flex;gap:12px;padding:14px 16px 16px}}
.n{{flex:none;font:700 12px/1 ui-monospace,Menlo,monospace;background:var(--ink);color:var(--bg);border-radius:3px;padding:5px 6px;height:max-content}}
.card h4{{margin:0 0 2px;font-size:15px;line-height:1.3;letter-spacing:-.005em}}
.by{{margin:0;font-weight:500}}.label{{margin:6px 0 0;color:var(--mute);font-size:13px;line-height:1.4}}
.src{{margin:8px 0 0;font-size:13px;color:var(--mute)}}.src a{{color:var(--ink)}}
footer{{border-top:1px solid var(--line);padding-top:16px;color:var(--mute);font-size:13px}}
@media (max-width:760px){{.wrap{{padding:28px 16px 56px}}.two{{grid-template-columns:1fr;gap:0}}.grid{{columns:2 150px;column-gap:12px}}.card{{margin-bottom:12px}}figcaption{{padding:10px 12px 12px;gap:8px}}}}
@media (max-width:420px){{.grid{{columns:1}}}}
</style></head><body><div class="wrap">
<header><p class="eyebrow">Reference board · {e(b.get("date"))} · {len(refs)} references</p>
<h1>{e(b["title"])}</h1><p class="meta">{e(meta)}<br>Sources: {e(", ".join(b.get("sources", [])))}</p></header>
<p class="summary">{e(data.get("summary"))}</p>
<section><h2>Recurring patterns</h2><dl class="pats">{patterns}</dl></section>
<section><h2>Creative directions</h2><div class="dirs">{directions}</div></section>
<div class="two">
<section><h2>Palette — sampled, approximate</h2><ul class="pal">{palette}</ul></section>
<section><h2>Type direction</h2><p>{e(data.get("type"))}</p>
<h2 style="margin-top:28px">What this set can't tell you</h2><p>{e(data.get("gaps"))}</p>
{f'<details><summary>Run log</summary><ul>{log}</ul></details>' if log else ""}</section>
</div>
<section><h2>References</h2><div class="grid">{cards}</div></section>
<footer>All work © its creators, collected for reference and inspiration only. Not for reproduction. Follow each link for the original and full credits.</footer>
</div></body></html>
""")
print(out)
PY
```

## Checking it

- Every `<img>` resolves: `grep -oE 'src="images/[^"]+"' board.html | sed 's/src="//;s/"$//' | sort -u | while read f; do [ -f "$OUT/$f" ] || echo "missing $f"; done`
- Numbers in the directions and palette exist as cards; the generator silently skips unknown numbers in
  chips, so compare counts by eye.
- View it at desktop and phone width if the host has a browser or preview pane. The grid collapses to two
  columns under 760 px and one under 420 px.
