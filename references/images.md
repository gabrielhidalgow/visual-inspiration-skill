# Downloading, screening and sampling images

`$W` is the working directory in the session scratchpad (SKILL.md Step 1). Everything here stays in
`$W`, and nothing is written into the project. `UA` is the browser user agent from `references/sources.md`.

## 1. Download every candidate

```bash
jq -r '[.id, .image_url] | @tsv' "$W/candidates.jsonl" > "$W/dl.tsv"
while IFS=$'\t' read -r id url; do
  curl -sL --max-time 30 -A "$UA" -e "$(dirname "$url")/" -o "$W/cand/$id.bin" "$url" </dev/null
done < "$W/dl.tsv"
```

The `</dev/null` is **load-bearing**. Inside a `while read` loop, curl otherwise eats the loop's stdin
and the run hangs or silently skips lines.

## 2. Validate by MIME type, never by size

CDNs answer missing or forbidden files with an HTML or XML error body, and curl saves it under your
filename. A non-empty file proves nothing.

```bash
for f in "$W/cand/"*.bin; do
  id=$(basename "$f" .bin)
  case "$(file -b --mime-type "$f")" in
    image/jpeg) mv "$f" "$W/cand/$id.jpg" ;;
    image/png)  mv "$f" "$W/cand/$id.png" ;;
    image/webp) mv "$f" "$W/cand/$id.webp" ;;
    image/gif)  mv "$f" "$W/cand/$id.gif" ;;
    image/avif) mv "$f" "$W/cand/$id.avif" ;;
    image/svg+xml) mv "$f" "$W/cand/$id.svg" ;;
    *) echo "$id — not an image ($(file -b --mime-type "$f")); dropped" >> "$W/run-log.md"; rm "$f" ;;
  esac
done
```

A failed download is not a reason to invent a replacement URL. Try the page's `og:image` once (see
`sources.md`); if that fails too, drop the candidate.

## 3. Screen: resolution and near-duplicates

```bash
uv run --quiet --with pillow python - "$W/cand" 600 <<'PY' > "$W/screen.tsv"
import sys, pathlib
from PIL import Image
d, MIN = pathlib.Path(sys.argv[1]), int(sys.argv[2])

def dhash(im, n=8):
    px = list(im.convert("L").resize((n + 1, n)).tobytes())
    return sum(1 << i for i in range(n * n) if px[(i // n) * (n + 1) + i % n] > px[(i // n) * (n + 1) + i % n + 1])

rows = []
for f in sorted(d.iterdir()):
    if f.suffix == ".svg":
        rows.append([f.stem, 0, 0, None, "ok-vector", f.name]); continue
    try:
        im = Image.open(f); im.seek(0); w, h = im.size
    except Exception as e:
        rows.append([f.stem, 0, 0, None, "unreadable", f.name]); continue
    rows.append([f.stem, w, h, dhash(im), "ok" if max(w, h) >= MIN else "low-res", f.name])

# Near-duplicates: Hamming distance <= 6 on a 64-bit dHash. Keep the larger one.
for i, a in enumerate(rows):
    for b in rows[i + 1:]:
        if a[3] is None or b[3] is None or not a[4].startswith("ok") or not b[4].startswith("ok"):
            continue
        if bin(a[3] ^ b[3]).count("1") <= 6:
            small, big = (a, b) if a[1] * a[2] < b[1] * b[2] else (b, a)
            small[4] = f"dupe-of-{big[0]}"

print("id\tw\th\tstatus\tfile")
for r in rows:
    print(f"{r[0]}\t{r[1]}\t{r[2]}\t{r[4]}\t{r[5]}")
PY
column -t "$W/screen.tsv"
```

Use `400` instead of `600` for a logo brief. Everything with status `ok` or `ok-vector` goes on to the
visual review in SKILL.md Step 6. Log the rest in one line each.

## 4. Sample a palette from the 9 picks

Run this after `$W/order.txt` holds the picks (see `contact-sheet.md`). Reference numbers come from that
order.

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
