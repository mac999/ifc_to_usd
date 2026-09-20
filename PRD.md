# PRD — ifc2usd: IFC to USD Conversion Pipeline

| | |
|---|---|
| **Product** | ifc2usd |
| **Version** | 0.2 (local web workspace specification) |
| **Date** | 2026-09-20 |
| **Purpose** | Product requirements at a level of detail that lets a developer start implementing from this document alone |

---

## 1. Product Overview

`ifc2usd` is a Python CLI tool with an optional Flask-based local web workspace
that converts IFC (Industry Foundation Classes) models into USD (Universal Scene
Description) scenes. The CLI and web workspace share the same conversion engine,
configuration schema, validation and reporting behavior.

Each conversion produces three artifacts:

1. `scene.usda` — the USD scene carrying geometry and semantics
2. `report.json` / `report.md` — conversion statistics and loss accounting
3. `convert.log` — the execution log

What sets the product apart is that it **verifies the result and records what was
lost**. Instead of reporting "finished without exceptions", it reports
"max vertex error 0.0006 mm, property retention 100%".

---

## 2. Design Rationale

Three constraints the implementation must honour, and the measurements behind them.

### 2.1 Read coordinates as float64; narrow to float32 only at write time

USD `points` is float32. Writing projected-CRS coordinates directly (a Korean TM
easting is around 200,000 m) produces a quantization error that grows with
coordinate magnitude.

| Coordinate magnitude | float32 ULP | Measured round-trip max error |
|---|---|---|
| 1,000 m | 0.061 mm | 0.063 mm |
| 10,000 m | 0.977 mm | 0.529 mm |
| 100,000 m | 7.813 mm | 5.313 mm |
| 200,000 m | 15.625 mm | 10.760 mm |
| 500,000 m | 62.500 mm | 21.749 mm |

Applying a local origin offset **while keeping float64 up to the point the offset
is subtracted** holds the error at 0.0006 mm regardless of coordinate magnitude.

> **Implementation note.** The offset does not undo precision that was already
> discarded. A single `dtype=np.float32` cast anywhere upstream defeats it, and
> reading vertices as float32 in the parser is the most common way this happens.

### 2.2 Do not cap property count by default

Capping properties per element (for example at 64) drops data silently. In a real
building model (286 products, 15,489 properties), 41.6% of elements exceeded the
cap and 7.7% of all properties were discarded with no signal to the user.

The default must be unlimited. When a cap is configured, the number of dropped
properties must appear in the report.

### 2.3 Distinguish "absent in source" from "lost in conversion"

IFC4X3 infrastructure models are often sparse in semantics — one road model
contained zero `IfcPropertySet` entities. Reporting that as "0% retention" is
misleading. Reports must carry `source_property_count` and
`preserved_property_count` as separate fields.

---

## 3. Goals and Non-Goals

### 3.1 Goals

| ID | Goal |
|---|---|
| G1 | Convert IFC geometry, semantics and identifiers to USD with measurable loss accounting |
| G2 | Guarantee sub-millimetre precision for projected-CRS input |
| G3 | Batch-convert every IFC file in a folder |
| G4 | Emit machine-readable (JSON) and human-readable (Markdown) reports |
| G5 | Reproducible runs driven by a single `config.json` |
| G6 | First successful conversion within five minutes of install |
| G7 | Browse IFC inputs and USD outputs, run conversions and inspect results in a local three-panel web workspace |
| G8 | Provide a polished, resizable workspace with dark/light themes and Korean/English UI |

### 3.2 Non-Goals (out of scope for v1)

- USD to IFC reverse conversion
- Material and texture conversion
- IFC validation (IDS, mvdXML)
- Publicly hosted, multi-user web service, remote filesystem browsing or cloud storage integration; the local web workspace is in scope

---

## 4. Users and Scenarios

| User | Purpose | Scenario |
|---|---|---|
| BIM engineer | Hand a design model to a real-time viewer | Convert one IFC, inspect in usdview |
| Digital twin developer | Assemble many models into one scene | Batch-convert 30 IFC files in a folder |
| QA engineer | Check conversion quality | Review loss and error figures in the report |
| CI pipeline | Convert automatically on model update | Parse `--json`, branch on exit code |
| Local workspace user | Convert and visually review without leaving the browser | Select IFC files in the left tree, run with visible options, open generated USD from the right tree in the central viewer |

---

## 5. Functional Requirements

### 5.1 Conversion (FR-C)

| ID | Requirement |
|---|---|
| FR-C1 | Convert a single IFC file to USD (`.usda` or `.usdc`) |
| FR-C2 | Process every `IfcProduct` that has a `Representation` |
| FR-C3 | Skip elements whose geometry fails, **counting them with a reason**; never abort the whole run |
| FR-C4 | Keep coordinates in float64 from parsing until the offset is applied; cast to float32 only immediately before writing |
| FR-C5 | Support local origin offset modes `auto_center`, `explicit`, `none` |
| FR-C6 | Record the offset in both USD `customLayerData` and the report |
| FR-C7 | Read the IFC length unit into USD `metersPerUnit`; default up axis Z |
| FR-C8 | With `preserve_z: true` (default), leave the vertical offset component at zero so elevations are preserved |

### 5.2 Semantics Mapping (FR-M)

| ID | Requirement |
|---|---|
| FR-M1 | `GlobalId` to customData `ifc_guid`, verbatim |
| FR-M2 | `is_a()` to customData `ifc_class` |
| FR-M3 | `Name`, `Description` to customData `ifc_name`, `ifc_description` |
| FR-M4 | Property sets to customData `pset:<PsetName>.<PropName>` |
| FR-M5 | Quantity sets to customData `qto:<QtoName>.<QuantityName>` |
| FR-M6 | `max_props_per_element` defaults to 0 (unlimited); when set, record the dropped count in the report |
| FR-M7 | Serialise non-scalar property values to strings; count both serialised and failed values |

### 5.3 Scene Structure (FR-S)

| ID | Requirement |
|---|---|
| FR-S1 | Default hierarchy follows the IFC spatial structure: `/World/<Site>/<Building>/<Storey>/<class>/g_<guid>` |
| FR-S2 | Elements outside the spatial structure go under `/World/Unassigned/` |
| FR-S3 | Hierarchy strategy is switchable: `spatial` (default), `by_class`, `flat` |
| FR-S4 | Normalise prim names to valid USD identifiers while preserving the original GUID in customData |
| FR-S5 | Resolve prim path collisions with a suffix and report the collision count |

### 5.4 Batch Conversion (FR-B)

| ID | Requirement |
|---|---|
| FR-B1 | Discover input files in a folder (`recursive`, `pattern`) and convert them all |
| FR-B2 | Mirror the input folder's relative structure in the output folder |
| FR-B3 | One file's failure must not stop the run (`on_error: continue \| stop`) |
| FR-B4 | Process-level parallelism (`workers`, default 4, auto-capped at half the physical cores) |
| FR-B5 | Emit `summary.json` and `summary.md` when the batch finishes |
| FR-B6 | Skip a file whose input hash matches the previous run and whose outputs exist (`skip_unchanged`, default true) |
| FR-B7 | Show progress on the console, suppressed by `--quiet` |

### 5.5 Reporting and Logging (FR-R)

| ID | Requirement |
|---|---|
| FR-R1 | Write `report.json` and `report.md` per input file |
| FR-R2 | Include every field listed below |
| FR-R3 | With `--json`, print the machine-readable result to stdout |
| FR-R4 | Write an execution log to `convert.log` at a configurable level |

**FR-R2 required report fields**

| Group | Fields |
|---|---|
| Input | path, SHA-256, file size, IFC schema, authoring application |
| Elements | target count, converted count, skipped count by reason, class distribution |
| Geometry | vertex count, face count, bounding box, maximum coordinate magnitude |
| Coordinates | offset mode and value, estimated maximum quantization error |
| Properties | source count, preserved count, dropped count, retention rate, elements over cap |
| Validation | round-trip vertex error (max, RMS), pass or fail |
| Execution | elapsed time, tool and dependency versions, configuration snapshot |

### 5.6 Validation (FR-V)

| ID | Requirement |
|---|---|
| FR-V1 | Re-read the written USD, compare vertex coordinates against the source, and record max and RMS error |
| FR-V2 | Warn and exit with code 2 when the error exceeds `validation.max_vertex_error_mm` (default 1.0) |
| FR-V3 | Warn when property retention falls below `validation.min_property_retention` (default 1.0) |
| FR-V4 | Allow validation to be disabled via `validation.enabled: false` for large workloads |

### 5.7 Local Web Workspace (FR-W)

| ID | Requirement |
|---|---|
| FR-W1 | Launch an optional local Flask web workspace via `ifc2usd web`; existing headless commands must work without web dependencies |
| FR-W2 | Left panel: lazy-loaded input folder/file tree with IFC filtering, search, expand/collapse, multi-selection and per-file conversion status; selecting a folder applies the configured recursion and pattern rules |
| FR-W3 | Right panel: output folder/file tree mirroring FR-B2, showing `.usda`, `.usdc`, reports and logs; refresh on completed artifacts, link each result to its source and preserve selection on refresh |
| FR-W4 | Center panel: interactive 3D canvas for a selected output USD with orbit, pan, zoom, fit-all, fit-selection, standard views, perspective/orthographic projection and an orientation indicator |
| FR-W5 | Support shaded, shaded-with-edges, wireframe and normals rendering; provide an independent transparency toggle with opacity slider from 0.05 to 1.00, default 0.35 when enabled |
| FR-W6 | Picking a rendered element exposes its GUID, IFC class, name, USD prim path and properties in a collapsible inspector; support isolate, hide and restore-all without editing artifacts |
| FR-W7 | Top menu panel exposes run-selected, run-all, cancel, validate-result and regenerate-report commands, plus visible effective conversion options and a collapsible advanced-options area |
| FR-W8 | Two draggable splitters resize the input tree, canvas and output tree; support keyboard resizing, panel collapse, double-click reset and per-browser persistence |
| FR-W9 | Provide `dark`, `light` and OS-following `system` themes, with dark as the initial default; update canvas, grids, menus, trees and focus/selection states together |
| FR-W10 | Switch all UI labels, menus, tooltips, validation messages and status text between Korean (`ko`, default) and English (`en`) without reload; preserve selection, camera, options and active jobs |
| FR-W11 | Show queued/running/succeeded/warning/failed/cancelled states, per-file and overall progress, elapsed time, warnings and a collapsible live log; reconnecting must recover the active job snapshot |
| FR-W12 | Distinguish empty folders, no selection, loading, no renderable geometry, conversion failure, preview failure and unavailable WebGL2; show actionable status and retain artifact/report access |

### 5.8 Web UX and Interaction Contract

**Layout and visual system**

```text
+--------------------------------------------------------------------------+
| Run / Cancel / Validate / Report | Conversion options | Theme | KO / EN   |
+------------------+------------------------------------+------------------+
| Input IFC tree   |                                    | Output USD tree  |
| Search / Select |          3D canvas viewer           | Search / Refresh |
| Folder / Files   |          View toolbar              | Scenes / Reports |
|                  |          Element inspector         | Logs             |
+------------------+------------------------------------+------------------+
| Job status / Progress / Elapsed time / Warnings / Expandable log          |
+--------------------------------------------------------------------------+
```

- Use a compact engineering workspace, not a landing page: a 56 px top bar,
  initial 260 px side panels, 6 px splitters and a flexible unframed canvas.
  Side panels resize within 180-480 px while keeping the canvas at least 320 px
  wide. Below 960 px, trees become mutually exclusive drawers; below 640 px,
  options move into a sheet and the top bar retains run/cancel and menu access.
- Use restrained neutral surfaces, teal selection accents and distinct semantic
  success/warning/error colors, with text or icons in addition to color. Use
  bundled Noto Sans KR for Korean/Latin UI and a monospace face for paths/logs;
  keep section headings compact, letter spacing zero and corner radii at 4-8 px.
- Use Lucide icons with localized accessible names/tooltips for viewer tools;
  use labeled buttons for execution, segmented controls for language/projection,
  menus for render modes, toggles for booleans and sliders/numeric inputs for
  opacity and numeric settings. Provide visible keyboard focus, tree navigation,
  accessible splitter values and keyboard equivalents for camera actions.
- Trees truncate long names with full-path tooltips. Toolbars wrap or overflow
  into menus without covering the canvas or each other. Canvas dimensions follow
  splitter/container changes without resetting the camera or stretching output.
- Persist theme, locale, panel sizes and last render settings in browser-local
  preferences. First launch uses `web` configuration defaults; explicit web CLI
  presentation flags override saved preferences for that launch. Thereafter UI
  changes take effect immediately and update preferences, not conversion config.
- Localize presentation only: filesystem names, GUIDs, IFC classes, raw dependency
  logs, configuration keys and machine-readable report fields remain unchanged.
  Store application statuses as stable codes with localized display messages.

**Execution and conversion options**

- Always display input/output roots, selection count, output format, worker count
  and validation-enabled state. Advanced controls expose recursion/pattern,
  overwrite, skip-unchanged, error policy, origin mode/value/preserve-Z, world
  coordinates, vertex welding, hierarchy, up axis, unit override, semantic
  preservation/property cap, validation thresholds, report and logging settings
  from section 7. Show defaults and inline errors; disable run until valid.
- Run-selected converts selected IFC files, expanding selected folders under the
  configured discovery rules and deduplicating paths. Run-all uses the input root.
  A job captures an immutable effective configuration and target list; subsequent
  edits affect only the next job. Export the conversion snapshot as `config.json`
  and show a copyable equivalent `convert`/`batch` command (one command per file
  for arbitrary selections), using shell-appropriate quoting.
- Confirm destructive overwrites and cancellation. Allow one active conversion
  job per workspace, using the existing batch workers internally. Cancellation
  stops scheduling new files, signals workers and records cancelled targets;
  completed artifacts remain usable and incomplete artifacts are not published.
  Show completed/total counts when byte-level progress is unavailable.
- Enable validate/report actions only for applicable completed outputs. They use
  the shared core and job status model. UI rendering, transparency and camera
  state never alter USD, reports, precision validation or conversion hashes.
- Clicking a completed USD opens its preview; reports/logs open a readable tab
  or drawer without losing the camera state. A running conversion never replaces
  the currently viewed result automatically. Refreshing the browser must not
  cancel a job; shutting down the server performs controlled cancellation.

**USD preview contract**

- Three.js must not be assumed to load arbitrary USD directly. A Flask-side
  preview adapter reads the generated `.usda`/`.usdc` using `usd-core`, evaluates
  the supported mesh hierarchy and transforms, and emits a cached GLB plus an
  element metadata index. Three.js loads it through `GLTFLoader`; use a maintained
  glTF serialization library such as `pygltflib` for the adapter.
- Preserve the GUID/prim-path mapping for picking; honor USD `metersPerUnit`,
  up axis and local origin. Key the preview cache by source artifact content and
  adapter version, outside deliverable folders. Never modify the source USD or
  include preview files in conversion success/precision measurements.
- Scope preview support to static mesh scenes produced by this pipeline. Default
  display materials and class colors are viewer-only, not IFC material/texture
  conversion. Report unsupported USD features and transparent-surface sorting
  limitations explicitly; do not imply production-renderer fidelity.
- Generate previews in background workers, dispose prior GPU resources on scene
  changes and show loading/error states. A preview failure must not mark an
  otherwise successful conversion as failed; keep USD download and report access.

---

## 6. IFC to USD Mapping

| IFC | USD | Notes |
|---|---|---|
| `IfcProject` | `/World` (defaultPrim, Xform) | `metersPerUnit`, up axis Z |
| `IfcSite` / `IfcBuilding` / `IfcBuildingStorey` | `Xform` | hierarchy per FR-S1 |
| `IfcProduct` (with Representation) | `UsdGeom.Mesh` | triangulated |
| `GlobalId` | `customData["ifc_guid"]` | verbatim |
| `is_a()` | `customData["ifc_class"]` | |
| `IfcPropertySet` | `customData["pset:<set>.<prop>"]` | |
| `IfcElementQuantity` | `customData["qto:<set>.<qty>"]` | |
| `IfcUnitAssignment` (length) | `metersPerUnit` | length units only in v1 |
| — | `customLayerData["origin_offset"]` | for coordinate restoration |

---

## 7. Configuration Schema (config.json)

```json
{
  "input":  { "path": "data/ifc", "recursive": true, "pattern": "*.ifc" },
  "output": { "dir": "out", "format": "usda", "overwrite": false },
  "geometry": {
    "use_world_coords": true,
    "weld_vertices": false,
    "origin_offset": { "mode": "auto_center", "value": [0, 0, 0], "preserve_z": true }
  },
  "semantics": {
    "psets": true,
    "quantities": true,
    "max_props_per_element": 0,
    "serialize_non_scalar": true
  },
  "scene":   { "hierarchy": "spatial", "up_axis": "Z", "meters_per_unit": null },
  "batch":   { "workers": 4, "on_error": "continue", "skip_unchanged": true },
  "validation": {
    "enabled": true,
    "max_vertex_error_mm": 1.0,
    "min_property_retention": 1.0
  },
  "report":  { "json": true, "markdown": true, "per_file": true, "summary": true },
  "logging": { "level": "INFO", "file": "convert.log" },
  "web": {
    "host": "127.0.0.1",
    "port": 5000,
    "open_browser": true,
    "theme": "dark",
    "locale": "ko",
    "viewer": { "render_mode": "shaded", "transparent": false, "opacity": 0.35 }
  }
}
```

Precedence is **CLI flags > config.json > defaults**. Every key is optional.
On a schema violation, exit with code 3 and name the offending key.

`web` is optional and ignored by headless conversion commands. Its host is
loopback-only in v1; port must be 1-65535, theme `dark|light|system`, locale
`ko|en`, render mode `shaded|edges|wireframe|normals`, and opacity 0.05-1.00.
Input/output roots come from `input.path` and `output.dir`; web launch requires
an input directory. Web form edits override the resolved conversion settings
only for the next job, using the same schema validation. Exclude `web` and all
browser preferences from conversion cache keys and artifact determinism checks.
Changing conversion options must invalidate skip-unchanged reuse even when the
input hash is unchanged.

---

## 8. CLI Specification

```
ifc2usd convert  <input.ifc> [-o OUT] [--config C] [--json]
ifc2usd batch    <input_dir> [-o OUT] [--config C] [--workers N] [--json]
ifc2usd inspect  <input.ifc>                     statistics only, no conversion
ifc2usd validate <scene.usda> --source <in.ifc>  verify an existing artifact
ifc2usd report   <out_dir>                       regenerate reports from artifacts
ifc2usd init                                     write a default config.json
ifc2usd doctor                                   check the runtime environment
ifc2usd web      [input_dir] [-o OUT] [--config C] start the local web workspace
```

Common options: `--config` (default `config.json`), `--json`, `--verbose`,
`--quiet`, `--version`

### 8.1 Web Launch Options

Install the optional stack with `pip install "ifc2usd[web]"`. Example:

```bash
ifc2usd web data/ifc -o out --host 127.0.0.1 --port 5000 --theme dark --locale ko
ifc2usd web --config config.json --no-open-browser --render-mode edges --transparent --opacity 0.25
```

| Option | Default | Contract |
|---|---|---|
| `[input_dir]` | `input.path` | Existing input root directory; readable IFC files only |
| `-o, --output DIR` | `output.dir` | Output root; create if missing, validate write access |
| `--host HOST` | `web.host` / `127.0.0.1` | Only loopback addresses or `localhost`; reject public bindings in v1 |
| `--port PORT` | `web.port` / `5000` | Fail clearly on conflicts; do not silently choose another port |
| `--open-browser / --no-open-browser` | `web.open_browser` / `true` | Open the local URL after readiness; browser-open failure leaves the server running and prints the URL |
| `--theme dark\|light\|system` | `web.theme` / `dark` | Initial launch theme |
| `--locale ko\|en` | `web.locale` / `ko` | Initial UI language |
| `--render-mode shaded\|edges\|wireframe\|normals` | `web.viewer.render_mode` / `shaded` | Initial render mode |
| `--transparent / --no-transparent` | `web.viewer.transparent` / `false` | Initial transparency state |
| `--opacity FLOAT` | `web.viewer.opacity` / `0.35` | Transparent-mode opacity, 0.05-1.00 |
| `--workers N` | `batch.workers` / `4` | Same validation and physical-core cap as batch mode |

Web mode accepts `--config`, `--verbose` and `--quiet` but rejects `--json`
with code 3: job results are served through the API, not a long-running stdout
result stream. Always print the local URL and startup failures; `--quiet`
suppresses routine access/progress logs. Ctrl+C performs graceful shutdown and
returns 0; runtime/startup failure returns 1, invalid configuration returns 3.
Conversion exit codes are recorded per job and do not terminate the web server.

**Exit codes**

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Conversion failure (for batch, when every file failed) |
| 2 | Validation threshold exceeded |
| 3 | Configuration error |

---

## 9. Architecture

```
ifc2usd/
  cli.py             argparse entry point, exit-code decision
  config.py          default merge, schema validation, path resolution
  reader/
    ifc_reader.py    ifcopenshell parsing: units, spatial structure, properties
    geometry.py      tessellation, float64 handling, offset computation
  writer/
    usd_writer.py    stage creation, hierarchy, customData
  validate.py        USD re-read and round-trip error
  report.py          JSON/Markdown reports and aggregation
  batch.py           folder discovery, parallel execution, summary
  logging_setup.py   file and console loggers
  web/
    app.py          Flask app factory, local API, session and root restrictions
    jobs.py         background process jobs, cancellation, progress snapshots
    preview.py      USD-to-GLB adapter, metadata index, preview cache
    templates/      Jinja2 workspace shell
    static/         bundled CSS, JavaScript, fonts, icons, locales and Three.js
tests/
docs/
```

### 9.1 Module Interfaces

```python
# reader/ifc_reader.py
def read_ifc(path: Path, cfg: dict) -> IfcModel: ...

@dataclass
class IfcModel:
    schema: str                       # "IFC2X3" | "IFC4X3" | ...
    meters_per_unit: float
    elements: list[Element]           # conversion targets
    skipped: list[SkipRecord]         # guid, class, reason
    spatial: dict[str, SpatialNode]   # guid -> parent chain
    stats: dict                       # source property totals, etc.

@dataclass
class Element:
    guid: str
    ifc_class: str
    name: str
    verts: np.ndarray                 # (N, 3) float64  <- never float32
    faces: np.ndarray                 # (M, 3) int32
    props: dict[str, str | int | float | bool]

# writer/usd_writer.py
def write_usd(model: IfcModel, out: Path, cfg: dict) -> WriteResult: ...
# WriteResult: scene_path, origin_offset, n_prims, prop_written, prop_dropped

# validate.py
def validate(scene: Path, model: IfcModel) -> ValidationResult: ...
# ValidationResult: max_error_mm, rms_error_mm, property_retention, passed
```

### 9.2 Dependencies

`ifcopenshell`, `usd-core`, `numpy` (Python 3.10 or later)

Optional `web` extra: `Flask`, `waitress` (cross-platform local WSGI server),
`pygltflib`. Frontend: Jinja2 shell, TypeScript modules, CSS design tokens,
Three.js with `OrbitControls`/`GLTFLoader`, Lucide icons and `ko`/`en` dictionaries.
Use Vite for development/build only; ship built assets in the Python wheel so
end users need neither Node.js nor CDN access. Pin compatible dependency versions.
Run conversion/preview work outside request threads in bounded process workers;
Flask debug mode and its development server are not the distributed runtime.

> `usd-core` publishes no aarch64 wheel on PyPI. On ARM, a distribution that
> bundles pxr (such as Isaac Sim) must be installed first; `ifc2usd doctor`
> checks for this.

### 9.3 Local Web API and Safety

| Endpoint | Responsibility |
|---|---|
| `GET /api/workspace` | Roots, validated configuration, schema/defaults and active job snapshot |
| `GET /api/tree?root=input\|output&path=...` | Lazy directory listing with server-issued entry IDs and statuses |
| `POST /api/jobs` | Validate action (`convert`, `validate`, `report`), entry IDs and configuration overrides; return job ID with HTTP 202 |
| `GET /api/jobs/<id>` | Poll status, monotonic progress, per-file results and log entries since a cursor; poll about once per second while active |
| `POST /api/jobs/<id>/cancel` | Request idempotent cancellation |
| `POST /api/previews` | Request/cache preview for an output entry; return a preview ID with HTTP 202 |
| `GET /api/previews/<id>` | Preview status, error details and ready GLB/metadata URLs |
| `GET /api/artifacts/<id>` | Read/download an authorized output artifact or cached preview |

Return structured errors (`code`, `message_key`, `details`), HTTP 400 for invalid
options, 403 for forbidden paths, 404 for missing entries and 409 for an active-job
conflict. Job snapshots retain numerical CLI result codes and reports; partial
batch failures must remain visible, not be flattened into a success toast.

Bind only to loopback, validate Host/Origin and disable cross-origin API access.
Use same-origin session cookies and CSRF protection for mutating endpoints.
Resolve and authorize all paths against startup input/output roots, including
symlink/junction resolution; prevent traversal and external USD reference reads.
Serve only registered artifacts/previews, escape filenames/properties/logs, and
never build shell commands from UI input. Root changes require a server restart;
v1 offers no arbitrary filesystem picker, uploads or remote access.

---

## 10. Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR-1 | Round-trip vertex error at or below 0.01 mm for coordinates up to 500 km |
| NFR-2 | Convert 10,000 elements within five minutes on a single core |
| NFR-3 | Handle a 100,000-element model within 8 GB via per-element streaming writes |
| NFR-4 | Byte-identical USD output for identical input and configuration |
| NFR-5 | Linux and Windows, Python 3.10 or later |
| NFR-6 | `ifc2usd convert model.ifc` works in one line after `pip install` |
| NFR-7 | Web workspace supports current stable Chrome/Edge/Firefox on desktop Windows/Linux with WebGL2; narrow layouts retain conversion/report access and unsupported rendering shows a fallback |
| NFR-8 | On a documented reference desktop and 100,000-triangle preview fixture, target at least 30 FPS during orbit at 1920x1080; job status updates within 2 s and cached tree/menu actions within 200 ms; record hardware/browser and cold-preview timing separately |
| NFR-9 | UI meets WCAG 2.2 AA contrast and keyboard requirements, respects reduced motion, and supports 200% zoom in both languages/themes without overlapping controls |
| NFR-10 | Installed web UI works offline with bundled assets; optional web dependencies do not affect headless installation or conversion results |

---

## 11. Acceptance Criteria

| ID | Criterion | Verification |
|---|---|---|
| AC-1 | 20 or more sample files all convert, geometry failure rate below 5% | `summary.json` after `batch` |
| AC-2 | Property retention is 100% with `max_props_per_element: 0` | `report.json` |
| AC-3 | Round-trip max error at or below 0.01 mm at 500 km | precision regression test |
| AC-4 | With 1 of 30 files failing, the other 29 complete and the exit code is non-zero | corrupted-file injection test |
| AC-5 | Report JSON contains every FR-R2 field | schema validation test |
| AC-6 | Two runs on identical input produce byte-identical USD | hash comparison |
| AC-7 | Output USD loads in usdview without errors | manual check |
| AC-8 | A prim can be looked up round-trip by GUID | unit test |
| AC-9 | `ifc2usd web` serves the workspace; root, port, theme, locale and viewer flags follow documented precedence; invalid flags/ports fail clearly | CLI integration tests including occupied port and missing web extra |
| AC-10 | Selecting IFC files/folders on the left runs the expected deduplicated targets; completed outputs appear on the right and open in the central canvas | Playwright end-to-end test with nested IFC fixtures and both USD formats |
| AC-11 | Every render mode and transparency toggle/slider works; picking resolves GUID/prim metadata; fit, isolate, hide and restore work without changing artifact hashes | Browser interaction, canvas-pixel and metadata checks |
| AC-12 | Theme and Korean/English changes cover labels/errors and preserve active jobs, selections and camera; preferences persist with explicit CLI overrides taking priority | Playwright reload and launch-precedence tests |
| AC-13 | Mouse/keyboard splitters respect limits and persist; no overlap at 1920x1080, 1280x720 and 390x844, or at 200% zoom | Playwright screenshots and accessibility checks in both languages/themes |
| AC-14 | Web and CLI jobs with equivalent resolved settings produce identical USD and equivalent conversion metrics; UI-only changes do not invalidate conversion reuse, conversion-option changes do | Cross-entry-point integration and cache regression tests |
| AC-15 | UI stays responsive during conversion; cancel/reconnect/partial failure preserve completed outputs; preview failure leaves successful conversion intact | Background-job lifecycle and failure-injection tests |
| AC-16 | Traversal, junction/symlink escapes, external USD references and cross-origin mutation are rejected; missing WebGL2 and empty scenes show correct fallback states | Security integration and browser fallback tests |
| AC-17 | Bundled web assets work offline, headless install remains usable without the extra, and reference-scene responsiveness meets NFR-8 | Packaging smoke tests and recorded browser performance run |

---

## 12. Milestones

| Stage | Scope | Deliverable |
|---|---|---|
| M1 | Single-file conversion, coordinate precision, basic report | `convert`, `report.json` |
| M2 | Full semantics preservation, spatial hierarchy, validation | `validate`, retention metrics |
| M3 | Batch conversion, parallelism, aggregate report | `batch`, `summary.md` |
| M4 | Flask workspace, shared configuration, trees, job API and CLI launch options | `ifc2usd web`, optional web extra, conversion workflow tests |
| M5 | USD preview, render modes/transparency, splitters, themes and Korean/English UI | Three-panel viewer, accessibility and browser acceptance tests |
| M6 | Conversion/viewer performance tuning, local security, offline packaging, documentation, release | PyPI package, CLI/web user guide, acceptance evidence |

---

## 13. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| No `usd-core` wheel on aarch64 | Install fails | `doctor` pre-check, document alternative distributions |
| Large models exceed memory | Conversion aborts | Per-element streaming writes, chunked processing |
| Geometry varies with tessellation parameters | Reproducibility suffers | Include the configuration snapshot in the report |
| Diverse property value types | Serialisation failures | Serialise non-scalars to strings, count failures |
| Prim path collisions | Elements lost | Suffix disambiguation and collision count (FR-S5) |
| Schema differences across IFC versions | Mapping gaps | Maintain IFC2X3 / IFC4 / IFC4X3 regression sets |
| Browser cannot directly display generated USD | Blank or incorrect preview | Server-side USD-to-GLB adapter, output-format fixtures and GUID/transform checks |
| Large previews or overlapping transparent surfaces | GPU exhaustion, low FPS or sorting artifacts | Lazy previews, GPU resource disposal, measured scene limits and explicit fidelity warnings |
| Local filesystem/API exposed through the browser | Unauthorized reads or conversions | Loopback binding, Host/Origin checks, CSRF, root confinement and registered artifacts only |
| Web and CLI settings diverge | Irreproducible conversions | Shared schema/core, immutable job snapshots and cross-entry-point tests |
