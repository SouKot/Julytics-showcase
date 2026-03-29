<div align="center">

<h1>Julytics</h1>
<p><strong>Interactive data visualization and analysis platform built with Julia</strong></p>

<video src="docs/assets/julytics.mp4" controls width="100%"></video>

<br/>

![Julia](https://img.shields.io/badge/Julia-1.9+-9558B2?style=for-the-badge&logo=julia&logoColor=white)
![WGLMakie](https://img.shields.io/badge/WGLMakie-GPU%20Accelerated-4CAF50?style=for-the-badge)
![Genie](https://img.shields.io/badge/Genie-Reactive%20UI-1565C0?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)

</div>

---

## What is Julytics?

Julytics is a browser-based data exploration and charting platform powered by Julia. It combines the numerical performance of Julia with a reactive web UI, enabling smooth, GPU-accelerated interactive plots — all without leaving the browser.

Built for analysts who want the power of a full programming language with the immediacy of a drag-and-drop interface.

---

## Screenshots

<table>
  <tr>
    <td align="center">
      <img src="docs/assets/home_tab.png" alt="Home Tab" width="100%"/>
      <sub><b>Home — Load datasets and manage your workspace</b></sub>
    </td>
    <td align="center">
      <img src="docs/assets/data_explorer.png" alt="Data Explorer" width="100%"/>
      <sub><b>Data Explorer — Drag-and-drop column mapping with live preview</b></sub>
    </td>
  </tr>
  <tr>
    <td align="center" colspan="2">
      <img src="docs/assets/bench_mode.png" alt="Bench Mode" width="60%"/>
      <sub><b>Bench Mode — Multi-panel analysis layout</b></sub>
    </td>
  </tr>
</table>

---

## Key Features

### 🎨 Interactive Plotting
- **GPU-accelerated rendering** via WGLMakie (WebGL) — smooth 60 FPS pan/zoom
- **Multiple chart types** — line, scatter, bar, boxplot
- **Smart layer diffing** — only mutates scene objects that actually changed, avoiding WGLMakie stutter

### 🔍 Data Explorer
- Drag-and-drop axis mapping (X, Y, Color)
- Live column preview with type inference
- Supports CSV and structured data files

### 📝 Annotation System
- Click-to-snap: circle snaps to the nearest data point
- Dashed leader line with automatic pixel-precise gap from text edge
- Optional text box background (toggle on/off per annotation)
- Annotations persist across sessions via JSON config
- Selection state (red = selected, black = unselected)
- **Vectorized Array Buffer architecture**: all annotations share 4 static GPU nodes — zero scene-graph mutations, zero UI stutter on create/delete

### 📐 Flexible Layout
- Multi-subplot grid with configurable rows/columns
- Bench mode for deep-dive analysis
- Presentation mode for clean exports

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | [Julia](https://julialang.org/) |
| Plotting | [WGLMakie](https://docs.makie.org/stable/) — WebGL via Observables |
| Reactive UI | [Genie.jl](https://genieframework.com/) + [Stipple.jl](https://github.com/GenieFramework/Stipple.jl) |
| Frontend | Vue 3 (managed by Stipple) |
| Geometry | [GeometryBasics.jl](https://github.com/JuliaGeometry/GeometryBasics.jl) |
| Persistence | JSON config snapshots with content-hash deduplication |

---

## Architecture Highlights

### Vectorized Annotation Buffer
Rather than creating and deleting Makie plot objects per annotation (which causes WebSocket serialization overhead and GPU stuttering), Julytics uses four **persistent static nodes** bound to shared `Observable{Vector}` arrays:

```
VA_ANC  — scatter!       anchor dots      (data space)
VA_LINE — linesegments!  leader lines     (data space, pixel-gap computed)
VA_BOX  — poly!          text backgrounds (pixel space)
VA_TXT  — text!          labels           (data space)
```

All annotation CRUD operations update the arrays in-place. Zero scene-graph mutations after initialization.

### Smart Layer Diffing
The plotting backend compares incoming layer configs against existing scene objects by label ID before rendering. Only changed layers are redrawn — unchanged layers are left untouched on the GPU.

---

## Source Code

The full source is maintained in a **private repository**. If you are interested in collaboration, licensing, or a deeper technical walkthrough, feel free to reach out.

---

<div align="center">
  <sub>Built with ❤️ using Julia</sub>
</div>
