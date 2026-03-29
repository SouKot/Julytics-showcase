<div align="center">

<h1>Julytics</h1>
<p><strong>Interactive data visualization and analysis platform built with Julia</strong></p>

![Julia](https://img.shields.io/badge/Julia-1.12.3-9558B2?style=for-the-badge&logo=julia&logoColor=white)
![WGLMakie](https://img.shields.io/badge/WGLMakie-GPU%20Accelerated-4CAF50?style=for-the-badge)
![Genie](https://img.shields.io/badge/Genie-Reactive%20UI-1565C0?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Alpha-orange?style=for-the-badge)
![AI Assisted](https://img.shields.io/badge/AI%20Assisted-Audited%20%26%20Tested-blueviolet?style=for-the-badge)

</div>

> **⚠️ Alpha — Active Development**
> Julytics is currently in alpha. Core features are functional but the API, UI, and data formats are subject to change. Feedback and bug reports are welcome.

---

## 🎬 Demo

> Click the image below to watch the demo video.

<div align="center">
  <a href="https://raw.githubusercontent.com/SouKot/Julytics-showcase/main/docs/assets/julytics.mp4">
    <img src="docs/assets/home_tab.png" alt="Click to watch Julytics demo" width="85%"/>
  </a>
  <br/>
  <sub>▶ Click to watch demo</sub>
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
      <img src="docs/assets/data_explorer.png" alt="Data Explorer" width="100%"/>
      <sub><b>Data Explorer — Drag-and-drop column mapping with live preview</b></sub>
    </td>
    <td align="center">
      <img src="docs/assets/bench_mode.png" alt="Bench Mode" width="100%"/>
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
- Click-to-snap: snaps to the nearest data point
- Dashed leader line with automatic pixel-precise gap from text edge
- Optional text box background (toggle on/off per annotation)
- Annotations persist across sessions via JSON config
- **Vectorized Array Buffer architecture**: all annotations share 4 static GPU nodes — zero scene-graph mutations, zero UI stutter on create/delete

### 📊 Statistical Analysis
Julytics ships a built-in statistical analysis panel backed by Julia's native statistics ecosystem:

| Test | Type | Use case |
|------|------|----------|
| **One-Way ANOVA** | Parametric | Compare means across ≥ 3 groups |
| **t-Test** | Parametric | Compare means of 2 groups |
| **Mann-Whitney U** | Non-parametric | 2-group alternative to t-Test |
| **Kruskal-Wallis** | Non-parametric | Multi-group alternative to ANOVA |
| **Pearson Correlation** | Correlation | Measure linear relationship strength |
| **Linear Regression** | Regression | Fit Y ~ X, report coefficients & R² |

Results are displayed in a live **descriptive statistics table** (statistic, p-value, significance) that updates as soon as analysis is run. Dependent and independent variables are selected directly from the loaded dataset columns — no coding required.

> Additional tests (e.g. Welch's t-Test, Shapiro-Wilk normality, χ² independence, ANCOVA) are planned for upcoming releases.

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
| Statistics | [HypothesisTests.jl](https://github.com/JuliaStats/HypothesisTests.jl), [GLM.jl](https://github.com/JuliaStats/GLM.jl), [StatsBase.jl](https://github.com/JuliaStats/StatsBase.jl) |
| Frontend | Vue 3 (managed by Stipple) |
| Persistence | JSON config snapshots with content-hash deduplication |

---

## Architecture Highlights

### 1. Reactive Model — Single Source of Truth
The entire application state lives in one `Model` struct declared with Stipple's `@app` DSL.
Each field is annotated `@in` (client → server write) or `@out` (server → client read-only),
and Stipple automatically serialises changes over WebSocket to the Vue 3 frontend.
There are no manual fetch calls, no polling, and no state duplication between layers.

```julia
@app Model begin
    @in  layout_config::LayoutConfig = LayoutConfig(...)  # grid + all cell/layer/annotation state
    @in  analysis_test_anova::Bool   = true               # UI checkbox bound to backend
    @out hypothesis_results::DataTable = DataTable()      # pushed to table on run
    # ... 60+ reactive fields
end
```

### 2. Dual-Server Architecture
Julytics runs two servers side-by-side, bridged via dynamic port negotiation:

| Server | Role |
|--------|------|
| **Genie HTTP** (port 8001) | Serves the main HTML page, routes, WebSocket for Stipple reactivity |
| **Bonito WS** (port 9303+) | Dedicated WebSocket for WGLMakie → WebGL GPU buffer updates |

At startup, `get_free_port(9303)` probes for the first available port so multiple instances
never collide. The Makie port is injected into the page template so the browser connects
to the correct WebGL endpoint.

### 3. LayoutConfig — Composable Grid State
All layout, cell, layer, and annotation data is expressed in a single serializable struct tree:

```
LayoutConfig
└── cells::Vector{CellConfig}       # N×M grid cells with (row, col, rowspan, colspan)
    ├── visuals / margins / labels
    ├── layers::Vector{LayerConfig}  # Multi-layer plots per cell
    │   └── x_var, y_var, color_var, type, color, width, opacity
    └── annotations::Vector{AnnotationConfig}  # persisted per cell
```

All mutations follow an **immutable-update** pattern — controllers `deepcopy` the current
config, apply changes, then write the new struct back — guaranteeing Stipple's reactive
graph fires exactly once:

```julia
conf = deepcopy(model.layout_config[])
conf.cells[idx].layers[lid].opacity = val
model.layout_config[] = conf          # single reactive notification
```

An `is_loading_cell` boolean guard prevents cascading re-saves during programmatic load.

### 4. N×M Grid Engine with Span Support
Cells behave like **CSS grid items** — each has (row, col, rowspan, colspan). The grid
engine supports:
- **Directional expansion/contraction** (8 triggers: expand/contract × up/down/left/right)
- **`compact_layout!()`** — reindexes a sparse grid by remapping only occupied row/column
  indices, preserving span correctness
- **Presentation / Workbench modes** — the splitter position is stored in the model and
  toggled via `current_tab` observer

### 5. Non-Reactive Global Data Store
DataFrame storage is deliberately kept outside the reactive model:

```julia
const GLOBAL_DATA = Ref{Union{DataFrame, Nothing}}(nothing)
```

This avoids serialising potentially large datasets over WebSocket on every state change.
Plot generation reads `GLOBAL_DATA[]` directly, while only lightweight metadata
(column names, row counts) is synced to the frontend.

### 6. Smart Layer Diffing
When `layout_config` changes, the plotting backend does a **label-ID-based scan** of the
existing Makie scene before re-rendering:

- Plots labelled `L{id}_{type}` are matched against incoming layers
- Unchanged layers are left untouched on the GPU
- Type-changed layers (e.g. lines → scatter) are replaced by label since the ID changes
- Stale layers are hidden via `p.visible[] = false` — never `delete!()`

This eliminates the full-scene teardown that caused WebSocket stutter in earlier versions.

### 7. Vectorized Annotation Buffer (VA Architecture)
Annotations use **4 static Makie nodes** per axis, backed by shared `Observable{Vector}` arrays:

```
VA_ANC  (scatter!)        — anchor dots at clicked data points
VA_LINE (linesegments!)   — dashed leader lines with pixel-precise text-edge gap
VA_BOX  (poly!, px-space) — optional white background box per label
VA_TXT  (text!)           — annotation labels in data space
```

`_rebuild_vbuf!()` is the single O(N) sync function that pushes all annotation state to GPU
arrays in one pass. Zero scene-graph mutations occur after initialisation — create, edit, and
delete are all array updates.

### 8. MVC Component Separation

| Layer | Modules |
|-------|--------|
| **Model** | `AppModel.jl` — all reactive `@in`/`@out` state |
| **Controllers** | `LayoutController`, `PersistenceController`, `PlotController`, `StatisticsController`, `HypothesisController` |
| **Handlers** | `LayoutHandler`, `StatisticsHandler`, `HypothesisHandler` — event dispatch & trigger management |
| **Components** | `InteractivePlot`, `DataViewer`, `DataExplorer`, `Workspace`, `Inspector` — UI rendering |
| **Types** | `Types.jl` — all shared structs (`LayoutConfig`, `AnnotationVBuffer`, `LayerData`, …) |

### 9. Live Script Console *(Early Stage)*

A sandboxed Julia REPL is embedded in the workbench. User code runs inside an isolated
`ScriptContext` module with the reactive model injected as `model`. Results are written
out via `log("msg")` to avoid stdout capture issues:

```julia
# Inside the script console:
log(describe(GLOBAL_DATA[]))  # inspect loaded dataset
model.status_msg[] = "Script done"  # update UI directly
```

> **⚠️ Early Stage.** The script console is functional at a basic level but requires significant additional work before it is genuinely useful:
> - No persistent variable scope between runs
> - No output streaming (output appears only after full execution)
> - No syntax highlighting or autocomplete
> - Error messages are minimal
>
> Improving the scripting experience is on the roadmap.

### 10. Session Persistence Strategy
Layout, layer mappings, and annotations are fully serialised through Stipple's JSON
mechanism. To avoid slow disk I/O on every request, `GenieSession.persist` is monkey-patched
at startup to a no-op, keeping all session state in-memory for the process lifetime.

---

## Architectural Roadmap

The current architecture is intentionally a **work in progress**. The codebase is subject to significant change as the project matures. The primary long-term goal is a progressively more **performant, stable, and extensible** application — the items below reflect a formal internal audit organised by priority.

---

### ⚡ Performance

| # | Issue | Plan |
|---|-------|------|
| 1.1 | **Type instability in data processing** — dynamic column extraction triggers Julia's dynamic dispatch, slowing aggregations on large datasets | Introduce function barriers (`compute_core(vec::Vector{Float64})`) so the compiler can specialize and run inner loops at native speed |
| 1.2 | **Single-threaded processing** — CSV parsing and statistical aggregations block the UI thread | Enable `JULIA_NUM_THREADS=auto` and apply `Threads.@threads` / `CSV.File(..., ntasks=N)` to parallelise independent operations |
| 1.3 | **WebSocket serialization bottleneck** — rendering 100k+ points simultaneously can freeze the WebGL socket | Implement server-side GPU Datashading / Hexbinning, returning a single PNG overlay instead of raw point geometry |
| 1.4 | **Vue DOM repaint overhead** — deeply nested reactive dictionaries in `AppModel` can trigger full-subtree repaints | Ensure all mutations are in-place Observable updates; batch WebSocket notifications using `Stipple.@js_watch` |

---

### 🛡️ Stability

| # | Issue | Plan |
|---|-------|------|
| 2.1 | ~~**Memory leaks & stutter from Makie scene graph mutations**~~ | ✅ **Resolved** — replaced `delete!(ax, plot)` with the Vectorized Array Buffer (VA) architecture; 4 static GPU nodes, zero scene-graph mutations after init |
| 2.2 | **Dangling Observable listeners** — `on(obs)` callbacks created for mouse interactions are never explicitly destroyed, accumulating invisible CPU load | Standardize a lifecycle pattern where all temporary tool listeners are returned and explicitly torn down on tool exit |

---

### 🖱️ User Ergonomics

| # | Issue | Plan |
|---|-------|------|
| 3.1 | **No column type casting** — numeric codes (1, 2, 3) for categorical groups are silently treated as continuous, causing incorrect statistical routing | Add a "Column Type" dropdown (Categorical / Continuous / String) next to each variable in the Data Drawer |
| 3.2 | **Blind off-canvas interactions** — users must look away from the plot to configure trace properties | Implement right-click context menus on the canvas, surfacing only the contextually relevant parameters for the clicked element |

---

### 🧪 Testing & Benchmarks

| # | Issue | Plan |
|---|-------|------|
| 4.1 | **No GPU render validation** — current CI tests verify DOM load but cannot detect a broken Makie render that produces a white screen | Spin up a headless GL server in GitHub Actions and perform pixel-level comparisons against golden baseline images using `ReferenceTests.jl` |
| 4.2 | **Benchmark blind spots** — benchmarks only measure ANOVA CPU speed | Expand to track memory allocations (`@benchmarkable ... allocs=true`) and stress-test with large Mixed Linear Models |

---

### 📊 Statistical Features

These additions would close the gap against established tools like GraphPad Prism and Jamovi:

| # | Feature | Priority |
|---|---------|----------|
| 5.1 | **Automated assumption checking** — run Shapiro-Wilk (normality) + Levene's (variance homogeneity) automatically before displaying ANOVA results | Critical |
| 5.2 | **Post-hoc multiple comparisons** — Tukey's HSD and Bonferroni corrections to identify which specific groups differ after a significant ANOVA | Critical |
| 5.3 | **Non-linear curve fitting** — IC50 / dose-response (4PL, 5PL) and exponential decay fitting via `LsqFit.jl` | High |
| 5.4 | **Repeated measures & mixed models** — track random effects for longitudinal biological data using `MixedModels.jl` | High |
| 5.5 | **Survival analysis** — Kaplan-Meier curves and Log-Rank tests via `Survival.jl` for clinical/oncology workflows | Medium |

---

> Architecture and feature priorities are re-evaluated continuously as the project evolves. All design decisions are driven by correctness, rendering performance, and the usability needs of scientific end-users.

---

## A Note on AI-Assisted Development

This project was built with the assistance of AI coding tools. However, to ensure **reliability, correctness, and performance**, every AI-generated contribution was subject to:

- ✅ Manual code review and architectural audit
- ✅ Rigorous testing of edge cases and interaction states
- ✅ Performance profiling and regression checks
- ✅ Domain-specific validation of numerical and rendering correctness

AI was used as a productivity accelerator — all design decisions, trade-off evaluations, and quality gates were driven by human judgement.

---

## Source Code

The full source is maintained in a **private repository**.

---

<div align="center">
  <sub>Built with ❤️ using Julia — Alpha release, actively developed</sub>
</div>
