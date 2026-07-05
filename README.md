# 3D Gaussian Splatting — from camera math to a browser viewer

A ground-up study of 3D Gaussian Splatting: the classical vision underneath it, a differentiable renderer written from scratch, a real object reconstructed from photos I took myself, and a browser viewer to see the result.

**Live:** https://leeyunhome.github.io/splatting-viewer/ · **Viewer:** https://leeyunhome.github.io/splatting-viewer/viewer/

한국어 안내는 라이브 페이지의 **KO/EN 토글**을 사용하세요 (페이지 자체가 한·영 병기).

---

## What's here

This repo is both a **case-study landing page** and the **interactive viewer** behind it.

- **`index.html`** — the case study. Five stages, in the order they were actually done:
  1. **Foundations** — camera model, lens distortion, features & matching, triangulation
  2. **A renderer from scratch** — 2D Gaussian rasterization, differentiable rendering (hard vs. soft edge), adaptive density control, L1 + D-SSIM loss
  3. **Photos → a model** — 26 photos I shot of the Mechander figure → COLMAP SfM → 3DGS training (30k, depth reg.) → live in the viewer
  4. **In the browser** — reading the Spark.js render pipeline and shipping a viewer
  5. **Roadmap** — Isaac Sim integration (in progress), outdoor/large scenes, `.spz` compression
- **`viewer/index.html`** — a [Spark.js](https://sparkjs.dev/) + Three.js WebGL viewer. Loads bundled presets or your own `.ply` / `.splat` / `.spz` / `.ksplat` file, with orbit/pan/zoom.

## Bundled model

| Model | File | Splats | Notes |
| --- | --- | --- | --- |
| Mechander | [`models/mechander.ply`](models/mechander.ply) | 388,261 | reconstructed from 26 photos; trained 30k iterations with depth regularization (92 MB) |

Each splat carries 62 attributes (position, covariance via scale + rotation quaternion, opacity, and spherical-harmonic color coefficients).

## Viewer controls

| Action | Input |
| --- | --- |
| Rotate | Left drag |
| Pan | Right drag / arrow keys |
| Zoom | Mouse wheel / middle drag |

## Local development

No build step. The landing page opens directly; the viewer needs an HTTP server (ES modules + importmap don't run over `file://`):

```bash
python -m http.server 8000
# landing → http://localhost:8000/
# viewer  → http://localhost:8000/viewer/
```

## Docs

- [`spark-rendering-pipeline.md`](spark-rendering-pipeline.md) — how Spark.js projects a 3D Gaussian to the screen (3D covariance → Jacobian → 2D projection → ellipse quad → Gaussian falloff → alpha blend), with source links.
- [`HOSTING.md`](HOSTING.md) — GitHub Pages hosting layout, deploy steps, and the model file-size watch list.

## Credits

Built on [Spark.js](https://sparkjs.dev/), [Three.js](https://threejs.org/), [COLMAP](https://colmap.github.io/), and gsplat / Nerfstudio. Course materials remain the property of their respective owners.
