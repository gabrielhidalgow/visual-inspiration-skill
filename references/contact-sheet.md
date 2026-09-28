# The contact sheet

One numbered image of the 9 picks, shown inline in the conversation. It is the whole visual deliverable:
no HTML, no files in the project. It is adapted from the pinterest skill's sheet, with justified rows,
nothing cropped and nothing letterboxed, plus the two things mixed design references need:
**SVG rasterising** and **transparent images on white** (black logo marks vanish on a dark sheet).

## Order

Write the picks, in the order you want them numbered, to `$W/order.txt` with one candidate id per line.
That file is the single source of numbering: the sheet and the `n — title (creator)` links both read it.
For a "more like N" round, **append** the new ids and pass `start_n` (10, then 19…), so numbers never
collide.

## Rasterise SVGs first

```bash
mkdir -p "$W/rast"
for f in "$W/cand/"*.svg; do [ -e "$f" ] || continue
  qlmanage -t -s 1200 -o "$W/rast" "$f" >/dev/null 2>&1 \
    || rsvg-convert -w 1200 "$f" -o "$W/rast/$(basename "$f").png"
done
```

`qlmanage` (macOS Quick Look) writes `<name>.svg.png`. The script below looks there for any `.svg` pick.

## Build the sheet

```bash
uv run --quiet --with pillow python - "$W" "$W/sheet.jpg" [start_n] <<'PY'
import sys, pathlib
from PIL import Image, ImageDraw, ImageFont

W, out_path = pathlib.Path(sys.argv[1]), pathlib.Path(sys.argv[2])
start_n = int(sys.argv[3]) if len(sys.argv) > 3 else 1   # "more like" rounds start at 10, 19, ...
order = [l.strip() for l in (W / "order.txt").read_text().split() if l.strip()][start_n - 1:][:9]

def load(cid):
    f = next((W / "cand").glob(f"{cid}.*"))
    if f.suffix == ".svg":
        f = W / "rast" / (f.name + ".png")
    vector = f.suffix == ".png" and f.parent.name == "rast"
    im = Image.open(f); im.seek(0)
    im = im.convert("RGBA")
    bg = Image.new("RGBA", im.size, (255, 255, 255, 255))   # transparent marks sit on white
    im = Image.alpha_composite(bg, im).convert("RGB")
    if vector:                                               # marks are cropped to their edges: give them air
        p = round(max(im.size) * 0.12)
        framed = Image.new("RGB", (im.width + 2 * p, im.height + 2 * p), (255, 255, 255))
        framed.paste(im, (p, p)); im = framed
    return im

ims = [load(c) for c in order]
SHEET_W, GAP, MARGIN, PER_ROW = 2400, 14, 18, 3             # 9 -> 3x3; Claude Code's viewer favours ~0.6-1.0 aspect
BG = (14, 14, 16)
inner = SHEET_W - 2 * MARGIN

# Justified rows: one height per row, widths follow true aspect, row fills the width.
rows = [ims[i:i + PER_ROW] for i in range(0, len(ims), PER_ROW)]
sizes, full_h = [], None
for row in rows:
    ars = [im.width / im.height for im in row]
    h = (inner - GAP * (len(row) - 1)) / sum(ars)
    if len(row) < PER_ROW and full_h:                        # a short last row must not balloon
        h = min(h, full_h)
    full_h = full_h or h
    sizes.append([(round(a * h), round(h)) for a in ars])

SHEET_H = 2 * MARGIN + sum(s[0][1] for s in sizes) + GAP * (len(rows) - 1)
sheet = Image.new("RGB", (SHEET_W, SHEET_H), BG)
draw = ImageDraw.Draw(sheet)
try:
    font = ImageFont.truetype("/System/Library/Fonts/Helvetica.ttc", 46)
except OSError:
    font = ImageFont.load_default(size=46)

n, y = start_n, MARGIN
for row, row_sizes in zip(rows, sizes):
    x = MARGIN
    for im, (w, h) in zip(row, row_sizes):
        sheet.paste(im.resize((w, h), Image.LANCZOS), (x, y))
        bx, by, r = x + 14, y + 14, 36                        # badge drawn AFTER the image
        draw.ellipse([bx, by, bx + 2 * r, by + 2 * r], fill=(0, 0, 0), outline=(255, 255, 255), width=3)
        tb = draw.textbbox((0, 0), str(n), font=font)
        draw.text((bx + r - (tb[2] - tb[0]) / 2, by + r - (tb[3] - tb[1]) / 2 - tb[1]),
                  str(n), fill=(255, 255, 255), font=font)
        x += w + GAP; n += 1
    y += row_sizes[0][1] + GAP

sheet.save(out_path, "JPEG", quality=84)
print(out_path, f"{SHEET_W}x{SHEET_H}", f"{out_path.stat().st_size // 1024}KB", f"{len(ims)} images")
PY
```

## Showing it

- In Claude Code: `SendUserFile` with `display: render` on `$W/sheet.jpg`. In any other agent, use its
  image-display call. With no image channel at all, print the absolute path and say the user needs to
  open it. Never write the analysis as though they have seen it.
- The sheet is 2400 px wide. It downscales cleanly, and its 3-across layout keeps each reference about
  240 px wide in Claude Code's viewer, which is legible for logos, type and layout.
- Portrait-heavy sets give a tall sheet and landscape-heavy sets a wide one. Both are fine at 9; don't
  crop to force a shape.
