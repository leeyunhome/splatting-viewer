# Splatting Viewer

Browser-based 3D Gaussian Splatting viewer built on [Spark.js](https://sparkjs.dev/) and Three.js.

**Live demo:** https://leeyunhome.github.io/splatting-viewer/

## Features

- Load `.ply` / `.splat` / `.spz` / `.ksplat` files from your local disk
- Orbit / pan / zoom with mouse, keyboard, and touch
- Auto-rotate and Y-axis invert toggles
- Default Butterfly sample loaded from Spark.js CDN

## Usage

Open the live demo URL, then either:

1. Click **"Load Default Butterfly"** to view the sample model, or
2. Use the file picker to load your own splat file.

## Controls

| Action | Input |
| --- | --- |
| Rotate | Left drag |
| Pan | Right drag / arrow keys |
| Zoom | Mouse wheel / middle drag |

## Local development

No build step needed — open `index.html` in a browser (or serve the folder with any static server).

```
python -m http.server 8000
# then open http://localhost:8000/
```

## Notes

See [`spark-rendering-pipeline.md`](spark-rendering-pipeline.md) for an explanation of the rendering pipeline this viewer uses.
