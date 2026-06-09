# Studio Vectorizer — Project Description

**Status:** Handoff brief, ready to build · **Date:** 2026-06-09 · **Owner:** cAI™

A studio-wide internal web tool that converts raster images to vector SVG, splits them into layers, and exports flexibly — including **G-code** to feed the robotic paint pipeline. Built as a visual node-graph editor so seniors can save and share reusable pipelines.

---

## 1. Decisions locked

| Question | Decision |
|---|---|
| Interface model | **Visual node-graph editor** — wire operations as nodes (load → vectorize → split → export), like Substance Designer / Nuke / ComfyUI. The graph is the saved, shareable document. |
| Layer split (v1) | **Color/region only** — group traced SVG paths by fill color. Deterministic, no ML, no GPU. |
| Export formats | **SVG** in v1 (per-layer ZIP, PNG/JPG, PDF are easy adds). **G-code deferred to Phase 2.** |
| Hosting | **Simplest managed deploy**, CPU-only. No SSO requirement; lightweight shared login. |
| Vectorization engine | **vtracer** (visioncortex), open-source, `pip install vtracer`. |
| Ruled out | Figma MCP — it embeds the bitmap, does no real tracing. |

G-code export is the longer-term connection to the **robotic paint pipeline** — the vectorizer becomes the front end that turns artwork into machine toolpaths — but it's **deferred to Phase 2** to keep v1 simple and avoid machine-specific complexity up front.

---

## 2. Scope

### Function A — Raster → SVG (vtracer)
True vector paths with color support. Strong on logos, flat art, and line art; bloated on photographs (expected, document it). Exposed parameters:

- `colormode`: `color` | `binary`
- `mode`: `spline` | `polygon` | `pixel`
- `color_precision`: 1–8 (lower = fewer colors)
- `filter_speckle`: higher removes more specks
- `corner_threshold`

Presets: **Logo/flat**, **Illustration**, **B&W line art**.

### Function B — Split into layers (v1: color/region)
After tracing, group SVG `<path>` elements by `fill` color into named layers. Pure post-processing on the SVG DOM — fast, deterministic, no ML. Each layer is independently visible, exportable, and routable to its own export node (e.g. one G-code file per paint color).

> Semantic/object split (person vs background via rembg / U²-Net / SAM) is **deferred to Phase 2** — it needs a GPU and more infra. Designed as a drop-in node type so it can be added without reworking the graph.

### Function C — Export (v1: SVG)
- **SVG** — combined, or per-layer.
- Cheap adds: per-layer **ZIP**, **PNG/JPG**, **PDF**.
- **G-code** — deferred to Phase 2. See §4 for the planned path.

---

## 3. Architecture

```
┌─────────────────────────────────────────┐
│  Browser — node-graph editor (React Flow)│
│  Load → Vectorize → Split → Export nodes  │
│  Graph saved as JSON = shareable document │
└───────────────┬───────────────────────────┘
                │ REST / JSON
┌───────────────▼───────────────────────────┐
│  Python backend (Flask/FastAPI)            │
│  Heavy nodes run here:                      │
│   • vectorize  → vtracer                     │
│   • split      → SVG fill-color grouping     │
│   • gcode      → vpype + vpype-gcode (Phase 2)│
└────────────────────────────────────────────┘
```

**Front end:** **React Flow (`@xyflow/react`)** for the node canvas — most mature, best docs, large ecosystem. (Alternatives: Rete.js, LiteGraph.js.) The node graph serializes to JSON; that JSON is the saved document, so seniors build reusable studio pipelines and juniors run them.

**Back end:** light operations (color split, file packaging) can run client-side or server-side; heavy operations (vtracer, future segmentation) run on the Python server. Each node maps to one backend endpoint or one step in a graph-execution endpoint.

**Why node-based earns its cost here:** the studio's real workflow is a *pipeline* (trace → split by color → one G-code file per color → robot). A node graph captures that pipeline once, names it, and lets anyone re-run it on new artwork.

---

## 4. SVG → G-code path (Phase 2)

Planned approach when G-code lands in Phase 2. Recommended: **`vpype` + `vpype-gcode` (`gwrite`)**. Both open-source and actively maintained (commits/issues through 2025).

Why vpype rather than a raw converter: vpype is purpose-built for plotter/pen toolpaths and includes **path optimization** — `linemerge`, `linesort`, and reordering — which cuts travel moves and pen/brush lifts. For *painting*, fewer lifts and shorter travel directly improve speed and finish.

Pipeline:

```
vpype read input.svg linemerge linesort gwrite --profile <machine> output.gcode
```

- `--profile` selects a machine profile (feed rates, pen/brush up-down commands, units). Profiles live in `~/.vpype.toml` or a project config; define a **studio profile per machine**.
- Set `vertical_flip = true` in the profile — G-code origin is bottom-left, SVG origin is top-left.
- **Color layers map cleanly to G-code:** export one G-code file per color layer so each maps to a paint/tool change.

Fallback: **`sameer/svg2gcode`** (Rust; CLI, library, and WASM) supports circular interpolation (G02/G03) and basic SVG shapes.

> **Robotic-painting caveat to resolve early:** if the rig is a multi-axis robot arm rather than a Cartesian plotter, generic pen-plotter G-code may not map 1:1 to robot motion. Confirm the controller's expected format. Likely need a **machine-specific post-processor / vpype profile** tuned to the robot — flag as a dedicated task and align with whatever the robotic paint pipeline already consumes.

---

## 5. Active bug to fix first

Testing produced: `Failed to execute 'postMessage' on 'Window': FormData object could not be cloned.`

This is a **client-side sandbox/preview error**, not an app bug — a proxied preview frame can't serialize FormData. The app is a real Flask app and must run on an actual server:

```
pip install -r requirements.txt && python app.py
# then open http://localhost:8000   (not as a static file, not in a preview frame)
```

If it must run somewhere that proxies `fetch`, change the upload from `FormData` to a **JSON body with the image base64-encoded**. Carry this fix forward into the new build's upload/load node so it never recurs in a hosted/proxied context.

---

## 6. Roadmap

**Phase 1 — Core, shippable**
- Node editor (React Flow): Load, Vectorize (vtracer), Color-Split, Export-SVG nodes.
- Presets + fine-tune controls; before/after preview; "show paths" toggle.
- Save/load graph as JSON. Lightweight shared login. Managed CPU deploy.

**Phase 2 — Power & reach**
- **G-code export** (vpype + vpype-gcode); studio vpype profile for the target machine; per-color G-code export. Connects to the robotic paint pipeline.
- Robot-specific G-code post-processor if the rig isn't a plain Cartesian plotter.
- Semantic/object layer split (rembg / SAM) as new node type (needs GPU host).
- Extra exports: per-layer ZIP, PNG/JPG, PDF.
- Saved/shared pipeline library; senior-authored templates for juniors.

---

## 7. Open items still to confirm

1. **Users & technical level** — who runs this day-to-day, and how technical? Drives how much we lean on prebuilt templates vs. free-form node wiring.
2. **Target machine for G-code** — Cartesian pen-plotter vs. robot arm; controller's expected G-code dialect. Determines whether a custom post-processor is needed.
3. **G-code semantics for paint** — pen-up/down vs. brush dip/pressure/Z-height; how color changes are signaled to the operator/robot.
4. **Login** — how lightweight is acceptable (shared password vs. per-user accounts) for "studio-wide."

---

## 8. Tech reference

**vtracer (Python)**
- `convert_raw_image_to_svg(img_bytes, img_format=, colormode=, mode=, color_precision=, filter_speckle=, corner_threshold=)` → SVG string
- `convert_image_to_svg_py(in, out, ...)` for files · CLI: `vtracer --input x.png --output x.svg`
- WASM build available for client-side tracing.

**Node-editor libraries:** React Flow (`@xyflow/react`) — recommended · Rete.js · LiteGraph.js

**SVG → G-code:** `vpype` + `vpype-gcode` (recommended) · `sameer/svg2gcode` (fallback)

**Existing prototype files:** `app.py`, `templates/index.html`, `requirements.txt` (flask, vtracer, pillow, gunicorn), `Dockerfile`, `README.md`.

---

### Sources
- [vpype-gcode (plottertools)](https://github.com/plottertools/vpype-gcode)
- [vpype (abey79)](https://github.com/abey79/vpype)
- [svg2gcode (sameer)](https://github.com/sameer/svg2gcode)
