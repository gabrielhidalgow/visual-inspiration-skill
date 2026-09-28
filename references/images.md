# Thumbnails, review, full-size fetch, palette

`$W` is the working directory in the session scratchpad (SKILL.md Step 1). Everything here stays in
`$W`, and nothing is written into the project. `UA` is the browser user agent from `references/sources.md`.

The flow is **screen by image, fetch big only for the picks**: download a small thumbnail for every
candidate (30–40 of them, cheap), look at them all on one labelled review sheet, choose 9, and only then
fetch full-size images for those 9.

## 1. Download thumbnails for every candidate

Each candidate line has `thumb_url` (listing thumbnail, WP `medium_large`, or og:image) and
`image_url` (full size, which may still be unknown for sources that need a project-page visit).

```bash
mkdir -p "$W/thumbs"
# The possibly-empty field goes LAST: `read` treats consecutive tabs as one separator, so an empty middle
# field would shift the next column into it.
jq -r '[.id, .url, (.thumb_url // .image_url // "")] | @tsv' "$W/candidates.jsonl" > "$W/thumbs.tsv"
while IFS=$'\t' read -r id page url; do
  if [ -z "$url" ]; then   # no listing image (e.g. Brand New posts without featured media): use the page's og:image
    url=$(curl -sL --max-time 20 -A "$UA" "$page" </dev/null | tr '\n' ' ' \
      | grep -oE 'property="og:image" content="[^"]+' | head -1 | sed 's/.*content="//')
    [ -z "$url" ] && { echo "$id — no image found on $page; dropped" >> "$W/run-log.md"; continue; }
  fi
  curl -sL --max-time 25 -A "$UA" -e "$(dirname "$url")/" -o "$W/thumbs/$id.bin" "$url" </dev/null
done < "$W/thumbs.tsv"
```

An og:image found this way is usually the full-size lead image too. Check that it isn't a generic
site card (see `sources.md`), and write it back as the candidate's `image_url` if picked.

The `</dev/null` is **load-bearing**. Inside a `while read` loop, curl otherwise eats the loop's stdin
and the run hangs or silently skips lines.

## 2. Validate by MIME type, never by size (use for both folders)

CDNs answer missing or forbidden files with an HTML or XML error body, and curl saves it under your
filename. A non-empty file proves nothing.

```bash
validate() {   # $1 = folder of <id>.bin files
  for f in "$1/"*.bin; do [ -e "$f" ] || continue
    id=$(basename "$f" .bin)
    case "$(file -b --mime-type "$f")" in
      image/jpeg) mv "$f" "$1/$id.jpg" ;;  image/png)  mv "$f" "$1/$id.png" ;;
      image/webp) mv "$f" "$1/$id.webp" ;; image/gif)  mv "$f" "$1/$id.gif" ;;
      image/avif) mv "$f" "$1/$id.avif" ;; image/svg+xml) mv "$f" "$1/$id.svg" ;;
      *) echo "$id — not an image ($(file -b --mime-type "$f")); dropped" >> "$W/run-log.md"; rm "$f" ;;
    esac
  done
}
validate "$W/thumbs"
```

Then rasterise any SVGs (logo marks), so the review sheet and the final sheet can show them:

```bash
mkdir -p "$W/rast"
for f in "$W/thumbs/"*.svg "$W/cand/"*.svg; do [ -e "$f" ] || continue
  qlmanage -t -s 1200 -o "$W/rast" "$f" >/dev/null 2>&1 \
    || rsvg-convert -w 1200 "$f" -o "$W/rast/$(basename "$f").png"
done
```

## 3. Review sheet: every thumbnail, labelled, with duplicates flagged

One image to look at before choosing anything. Curated candidates (staff pick, award winner) get a
yellow dot after their id, and near-duplicates (by perceptual hash) are listed so you pick only one of each pair.

```bash
uv run --quiet --with pillow python - "$W" <<'PY'
import sys, json, pathlib
from PIL import Image, ImageDraw, ImageFont
W = pathlib.Path(sys.argv[1])
meta = {json.loads(l)["id"]: json.loads(l) for l in open(W / "candidates.jsonl")}

def load(f):
    if f.suffix == ".svg":
        f = W / "rast" / (f.name + ".png")
    im = Image.open(f); im.seek(0); im = im.convert("RGBA")
    return Image.alpha_composite(Image.new("RGBA", im.size, "white"), im).convert("RGB")

def dhash(im, n=8):
    px = list(im.convert("L").resize((n + 1, n)).tobytes())
    return sum(1 << i for i in range(n * n) if px[(i // n) * (n + 1) + i % n] > px[(i // n) * (n + 1) + i % n + 1])

items = []
for f in sorted((W / "thumbs").iterdir()):
    try: items.append((f.stem, load(f)))
    except Exception: print(f"{f.stem}: unreadable, skipped")
hashes = {cid: dhash(im) for cid, im in items}
ids = list(hashes)
for i, a in enumerate(ids):
    for b in ids[i + 1:]:
        if bin(hashes[a] ^ hashes[b]).count("1") <= 6: print(f"near-duplicate: {a} ~ {b}")

CELL, COLS, LABEL = 360, 6, 46
font = ImageFont.truetype("/System/Library/Fonts/Helvetica.ttc", 30)
rows = -(-len(items) // COLS)
sheet = Image.new("RGB", (COLS * CELL, rows * (CELL + LABEL)), (28, 28, 30)); d = ImageDraw.Draw(sheet)
for k, (cid, im) in enumerate(items):
    im.thumbnail((CELL - 12, CELL - 12)); x, y = (k % COLS) * CELL, (k // COLS) * (CELL + LABEL)
    sheet.paste(im, (x + (CELL - im.width) // 2, y + 6 + (CELL - 12 - im.height) // 2))
    d.text((x + 10, y + CELL + 6), cid, fill=(255, 214, 0), font=font)
    if meta.get(cid, {}).get("curated"):                  # drawn marker: Helvetica has no ★ glyph
        tx = x + 18 + d.textlength(cid, font=font)
        d.ellipse([tx, y + CELL + 14, tx + 18, y + CELL + 32], fill=(255, 214, 0))
sheet.save(W / "review.jpg", quality=80)
print(W / "review.jpg", f"{len(items)} candidates")
PY
```

Read `$W/review.jpg`, which is for you, not the user. Open any borderline candidate's thumbnail on its own
before deciding.

## 4. Fetch full size for the 9 picks

After writing the 9 ids to `$W/order.txt` (SKILL.md Step 6): if a pick has no `image_url` yet, get it from
its project page first (the per-source recipe in `sources.md`, e.g. the Fonts In Use use page, or
`og:image`), and record it in `candidates.jsonl`. Then:

```bash
mkdir -p "$W/cand"
while read -r id; do
  url=$(jq -r --arg id "$id" 'select(.id==$id) | .image_url // empty' "$W/candidates.jsonl")
  [ -n "$url" ] && curl -sL --max-time 40 -A "$UA" -e "$(dirname "$url")/" -o "$W/cand/$id.bin" "$url" </dev/null
done < "$W/order.txt"
validate "$W/cand"
uv run --quiet --with pillow python -c "
import sys, pathlib; from PIL import Image
W = pathlib.Path(sys.argv[1]); MIN = int(sys.argv[2])
for cid in (W / 'order.txt').read_text().split():
    f = next((W / 'cand').glob(cid + '.*'), None)
    if f is None: print(cid, 'MISSING'); continue
    if f.suffix == '.svg': print(cid, 'ok-vector'); continue
    w, h = Image.open(f).size; print(cid, f'{w}x{h}', 'ok' if max(w, h) >= MIN else 'LOW-RES')
" "$W" 600
```

Use `400` instead of `600` for a logo brief. For any pick that is **MISSING** or **LOW-RES**: copy its
thumbnail into `$W/cand/` if the thumbnail itself clears the threshold. Otherwise swap in the next-best
candidate from the review sheet and repeat for that one. Never invent a replacement URL.

## 5. Sample a palette from the 9 picks

Reference numbers come from the order in `$W/order.txt`.

```bash
uv run --quiet --with pillow python - "$W" <<'PY'
import sys, pathlib
from PIL import Image
W = pathlib.Path(sys.argv[1])
order = [l.strip() for l in (W / "order.txt").read_text().split() if l.strip()]
clusters = []   # [r, g, b, weight, set(ref numbers)]
for n, cid in enumerate(order, 1):
    f = next((W / "cand").glob(f"{cid}.*"))
    if f.suffix == ".svg": continue                      # vector marks are flat ink; name their colour by eye
    im = Image.open(f).convert("RGB"); im.thumbnail((160, 160))
    q = im.quantize(colors=6, method=Image.Quantize.MEDIANCUT)
    pal, total = q.getpalette(), im.width * im.height
    for count, idx in q.getcolors():
        r, g, b = pal[idx * 3: idx * 3 + 3]; share = count / total
        if share < 0.04: continue
        for c in clusters:
            if abs(c[0] - r) + abs(c[1] - g) + abs(c[2] - b) < 48:
                w = c[3] + share
                c[0], c[1], c[2] = [round((c[k] * c[3] + v * share) / w) for k, v in enumerate((r, g, b))]
                c[3] = w; c[4].add(n); break
        else:
            clusters.append([r, g, b, share, {n}])
clusters.sort(key=lambda c: -c[3])
for r, g, b, w, refs in clusters[:10]:
    print(f"#{r:02X}{g:02X}{b:02X}\tweight {w:.2f}\trefs {sorted(refs)}")
PY
```

This gives candidates, not the answer. Pick **4–6** that describe the set: a ground, an ink, and accents
that recur across **several** references (a colour in only one image is that image's palette, not the
set's). Label them **approximate**. JPEG re-encoding shifts gradients and photos more than flat graphic
colour.
