# Hosting plan — GitHub Pages

How this project is hosted, why it's laid out this way, and what to watch as the models grow.

---

## 1. How the rest of the site is hosted (the pattern this follows)

`leeyunhome`'s site is a **user page + many project pages**, all on free GitHub Pages:

| Piece | Repo | URL |
|---|---|---|
| Hub (landing) | `leeyunhome/leeyunhome.github.io` | `https://leeyunhome.github.io/` |
| Each mini-app | its own repo (e.g. `pdf-merger`, `smart-qr`, `ai_chatbot_hodu_ai`) | `https://leeyunhome.github.io/<repo>/` |
| Career portfolio | `portfolio` (MkDocs) | `https://leeyunhome.github.io/portfolio/` |
| **This project** | `splatting-viewer` | `https://leeyunhome.github.io/splatting-viewer/` |

Convention: **one repo per project → served at `leeyunhome.github.io/<repo>/`**, and the hub links out to each. This project keeps that convention — no new hosting model to learn.

## 2. Layout of this repo

```
splatting-viewer/                → https://leeyunhome.github.io/splatting-viewer/
├── index.html                   # portfolio / case-study landing page
├── viewer/index.html            → /splatting-viewer/viewer/  (the interactive WebGL viewer)
├── models/                      # trained 3DGS .ply files (served statically)
│   └── mechander.ply  (92 MB)
├── assets/                      # case-study media (training/orbit videos, stills)
├── spark-rendering-pipeline.md  # rendering pipeline write-up (linked from the page)
├── README.md
└── HOSTING.md                   # this file
```

The landing page (`/`) is the case study; the viewer (`/viewer/`) is the app. The landing's **Launch viewer** button and the viewer's **← Project overview** link connect them. Everything is static — no build step, no server code.

## 3. GitHub Pages settings

One-time, in the repo on GitHub:

1. **Settings → Pages**
2. **Source:** *Deploy from a branch*
3. **Branch:** `main` · folder `/ (root)` → **Save**
4. Wait 1–2 min → live at `https://leeyunhome.github.io/splatting-viewer/`

No workflow file needed (plain static site). The importmap in `viewer/index.html` pulls Three.js and Spark.js from CDNs at runtime — GitHub Pages only has to serve the HTML, the `.ply` models, and the media.

## 4. Deploy

```bash
cd splatting-viewer
git add index.html viewer/ assets/ README.md HOSTING.md
git commit -m "Add case-study landing page; move viewer to /viewer/"
git push origin main
```

Pages redeploys automatically on push to `main` (usually live within a minute).

## 5. File-size watch list ⚠️

GitHub enforces per-file limits that matter here because 3DGS models are large:

| Threshold | Effect |
|---|---|
| **50 MB** | push **warning** (`GH001: Large files detected`) — `mechander.ply` (92 MB) already triggers this; push still succeeds |
| **100 MB** | **hard block** — a push containing a file this big is rejected |

`mechander.ply` at 92 MB is under the block but close. **Before adding any model ≥ ~90 MB, compress first.** Options, cheapest first:

1. **`.spz` / `.splat` compression** — Spark.js loads these natively; ~10× smaller (92 MB → ~10 MB) and much faster page loads. Best option; on the roadmap.
2. **GitHub Releases attachment** — up to 2 GB per file; keep the repo light and point the viewer at the release download URL.
3. **Git LFS** — works for storage, but GitHub Pages does **not** reliably serve LFS files, so avoid for models that must load in-browser.

Also soft, rarely hit here: Pages sites have a ~1 GB soft size cap and ~100 GB/month soft bandwidth. Current total (~92 MB, one model) is fine; revisit if many large models pile up.

## 6. Local preview

The landing page is plain HTML and opens directly. The viewer uses ES modules + an importmap, which browsers block over `file://`, so serve over HTTP:

```bash
cd splatting-viewer
python -m http.server 8000
# landing → http://localhost:8000/
# viewer  → http://localhost:8000/viewer/
```

## 7. Optional: custom domain

If a domain is ever added, drop a `CNAME` file (containing the domain) at the repo root and set the DNS `CNAME`/`A` records per GitHub's docs. Not required — the `github.io` URL works as-is.

## 8. Hub link

The hub (`leeyunhome.github.io`) already has a card pointing to `https://leeyunhome.github.io/splatting-viewer/`, which now lands on this case-study page. No hub change is required for the URL; only the card's description was refreshed to reflect the fuller project.
