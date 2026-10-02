# PRD — ifc2usd: IFC to USD Conversion Pipeline

| | |
|---|---|
| **Product** | ifc2usd |
| **Version** | 0.3 (implementation contract and verified development workflow) |
| **Date** | 2026-10-02 |
| **Purpose** | Product requirements at a level of detail that lets a developer start implementing from this document alone |

---

## 0. Start Here: Development Contract

This document applies equally to human developers and coding agents, including
Codex and Claude Code. It describes an existing implementation, not a greenfield
scaffold. Read repository instructions and inspect the working tree before editing.
Preserve unrelated local changes. Do not add AI attribution to commits, PRs or logs;
commits are authored by the user alone.

Resolve specification conflicts in this order: current owner instructions,
repository instructions, this revision, then older examples. The owner has selected
**English-only UI**. Korean UI defaults, language switching and bilingual acceptance
criteria from the previous revision are superseded. Do not reintroduce them.
Requirement IDs below are retained for traceability. A requirement is not evidence
of completion; section 14 identifies what has actually been checked and what remains.
The specification revision is 0.3; the package version remains 0.2.0 unless a
separate release change updates both package metadata and `ifc2usd.__version__`.

### 0.1 Environment and first run

Use the existing `venv_lmm` Conda environment on this checkout. Do not create another
virtual environment or silently switch to system Python or the old `.venv` directory.
The recorded Windows interpreter is
`C:\Users\ktw\.conda\envs\venv_lmm\python.exe`; this is machine-specific, not a path to
hard-code into application code. On other machines select a compatible environment.

Run from the repository root in PowerShell:

```powershell
conda activate venv_lmm
python -c "import sys; print(sys.executable)"
python -m ifc2usd doctor
# Install only if the environment is missing the editable project or dependencies:
python -m pip install --no-cache-dir -e ".[web,test]" -c constraints-tested.txt
python -m ifc2usd web input -o out --no-open-browser
```

Open the printed URL, normally `http://127.0.0.1:5000`. Keep the server process alive
while using the workspace. A shell session with a running server is expected.
Verify `/api/workspace` reports this checkout's `input` and `out` roots before testing.
If a server is already present, inspect its roots first; do not assume that a working
URL points to the intended files, and do not terminate an unrelated process.
If the assistant's embedded browser is unavailable, provide the local URL and use a
supported desktop browser. Browser availability is not a conversion-engine failure.

| Path | Meaning |
|---|---|
| `input/Office_A_20110811.ifc` | Real user-supplied model for integration/visual checks |
| `out/` | Normal local outputs; ignored by Git |
| `.test-output/browser-input/` | Synthetic browser fixtures only; never present these as the user's models |
| `.test-output/` | Disposable investigation/test evidence; preserve unrelated existing files |
| `data/ifc` | Library configuration default; not the sample directory in this checkout |

Input and output roots must be separate and neither may contain the other. Roots are
locked at server startup. The v0.3 UI browses within those roots; it does not offer an
OS-wide folder picker, file uploads or runtime root changes. Disabled root fields are
intentional and must explain how to restart with another path.

### 0.2 Build and change discipline

- Python CLI and Flask must call the same reader, writer, validation and batch core.
  Do not implement a separate web conversion path.
- Edit `frontend/*.ts` and `frontend/workspace.css`, then run `npm run build`.
  The Flask server serves `ifc2usd/web/static/dist/`; changing TypeScript alone does
  not update the delivered UI. Do not hand-edit generated JavaScript.
- Reuse `package-lock.json` with `npm ci` when dependencies are missing. Node is for
  development, not a runtime requirement for Python users. No CDN dependencies.
- Restart Python after backend changes and reload the page after asset changes.
  A server restart invalidates browser sessions/CSRF tokens and in-memory jobs.
- Changing authored USD semantics requires a conversion-key writer schema/version
  change; changing preview serialization requires an `ADAPTER_VERSION` change.
  Otherwise old artifacts may be incorrectly reused despite a code fix.
- Use a fresh output folder when comparing options. Do not delete or overwrite
  existing user outputs just to make a test pass. Explain explicit overwrite and
  the reuse rules in section 7.

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
is subtracted** produced the small-model measurements above. These are reference
measurements, not universal guarantees: a model's extent and unshifted elevation
still determine float32 precision. Every enabled validation measures actual error.

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
| G2 | Preserve precision with a local origin and measure the actual error; sub-millimetre results depend on model extent and preserved elevations |
| G3 | Batch-convert every IFC file in a folder |
| G4 | Emit machine-readable (JSON) and human-readable (Markdown) reports |
| G5 | Reproducible runs driven by a single `config.json` |
| G6 | First successful conversion within five minutes of install |
| G7 | Browse IFC inputs and USD outputs, run conversions and inspect results in a local three-panel web workspace |
| G8 | Provide a polished, resizable workspace with dark/light themes and English UI |

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
| FR-C9 | Preserve `IfcSpace`, `IfcOpeningElement` and `IfcOpeningStandardCase` meshes and semantics, but author them with USD `purpose=guide`. These are spatial/void volumes, not opaque building components. Include them in conversion and validation counts. |
| FR-C10 | A `scene.meters_per_unit` override must rescale authored coordinates from SI metres into that unit, preserving physical size. Changing metadata alone is incorrect. |

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
| FR-B6 | Reuse only when source hash, conversion configuration/version hash and every recorded artifact hash match (`skip_unchanged`, default true); existence alone is insufficient |
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
| FR-W10 | All authored UI labels, menus, tooltips, validation messages and statuses are English (`en`). Preserve source names, GUIDs and raw dependency messages verbatim. No language switcher in this version. |
| FR-W11 | Show queued/running/succeeded/warning/failed/cancelled states, per-file and overall progress, elapsed time, warnings and a collapsible live log; reconnecting must recover the active job snapshot |
| FR-W12 | Distinguish empty folders, no selection, loading, no renderable geometry, conversion failure, preview failure and unavailable WebGL2; show actionable status and retain artifact/report access |
| FR-W13 | Folder-name activation expands/collapses; its checkbox selects that folder for recursive conversion. File names and checkboxes toggle selection consistently. Provide input refresh, clear-selection and search across unexpanded subfolders. Select-visible acts only on displayed entries. |
| FR-W14 | Surface per-file failure reasons automatically, including existing outputs with overwrite disabled. Invalid option text must remain visible and block both run commands until corrected. |

### 5.8 Web UX and Interaction Contract

**Layout and visual system**

```text
+--------------------------------------------------------------------------+
| Run / Cancel / Validate / Report | Conversion options | Theme            |
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
  use labeled buttons for execution, labeled projection controls,
  menus for render modes, toggles for booleans and sliders/numeric inputs for
  opacity and numeric settings. Provide visible keyboard focus, tree navigation,
  accessible splitter values and keyboard equivalents for camera actions.
- Trees truncate long names with full-path tooltips. Toolbars wrap or overflow
  into menus without covering the canvas or each other. Canvas dimensions follow
  splitter/container changes without resetting the camera or stretching output.
- Persist theme, panel sizes and last render settings in browser-local
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

- Hide guide volumes by default; expose a `Spaces / openings` toggle without
  modifying the USD. Restore-all respects this toggle. A successful round-trip
  check alone does not prove that a building is visually readable: acceptance
  must include a real IFC containing spaces and openings.
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

### 7.0 Coordinate and unit contract

Let `u` be output metres-per-unit: the explicit positive unit override, or the IFC
length-unit scale when null. IfcOpenShell geometry is returned in SI metres.

1. Read tessellated points as float64. If `use_world_coords=false`, apply the
   product's shape transform once to recover world-space SI coordinates; this
   flag must not scatter products at the origin or alter the physical model.
2. Convert to stage units: `p_stage = p_world_metres / u`.
3. Compute `origin_offset` in stage units. Auto-center uses the complete target
   bounding box, including guide geometry. Explicit values use the same units.
   `preserve_z=true` forces offset Z to zero even in explicit mode.
4. Subtract offset in float64. For Z-up keep `(x,y,z)`; for Y-up write `(x,z,-y)`.
   Only then cast authored mesh points to float32.
5. Author `metersPerUnit=u`, the stage up axis and the original pre-axis-rotation
   offset in layer custom data. Do not add the offset again as a scene transform.
6. Validation reverses the axis rotation, adds the offset, and compares in
   millimetres. Preview converts local stage points to metres and to glTF Y-up
   once; it does not restore the large georeferenced offset into the camera scene.

Example: one metre is 1 stage unit at `u=1`, and 1,000 stage units at `u=0.001`.
Both must look the same physical size. A metadata-only unit change fails FR-C10
even if a self-consistent round-trip test passes. Test against independently
measured source coordinates in metres, not just the converted in-memory model.

### 7.1 Defaults

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
    "locale": "en",
    "viewer": { "render_mode": "shaded", "transparent": false, "opacity": 0.35 }
  }
}
```

Precedence is **CLI flags > config.json > defaults**. Every key is optional.
On a schema violation, exit with code 3 and name the offending key.

`web` is optional and ignored by headless conversion commands. Its host is
loopback-only in v1; port must be 1-65535, theme `dark|light|system`, locale
`en` only, render mode `shaded|edges|wireframe|normals`, and opacity 0.05-1.00.
Input/output roots come from `input.path` and `output.dir`; web launch requires
an input directory. Web form edits override the resolved conversion settings
only for the next job, using the same schema validation. Exclude `web` and all
browser preferences from conversion cache keys and artifact determinism checks.
Changing conversion options must invalidate skip-unchanged reuse even when the
input hash is unchanged.

### 7.2 Reuse and output publication

- `convert model.ifc -o out` writes `out/model/scene.usda`, not `out/scene.usda`.
  Batch mirrors relative folders and removes the `.ifc` suffix for each output
  directory. Explicit input paths are recorded in the configuration snapshot.
- Relative paths in `config.json` resolve against the configuration file's
  directory; explicit CLI paths resolve against the current working directory.
- Each completed directory contains `.ifc2usd.json` with source/configuration
  hashes, artifact hashes, result code and report. It is required for reuse,
  report regeneration and the source link used by web validation. It is not a
  user-facing tree entry. Do not delete it independently of its result.
- Reuse is checked before overwrite. Unchanged matching results may be reused
  when overwrite is false. A changed format, configuration, writer schema or
  artifact hash requires a new output directory or explicit overwrite.
- If reuse fails and a destination contains files, overwrite=false returns a
  clear error. It must not appear to the user as an unresponsive Run button.
- Build into a sibling staging directory, validate and close logs before
  publication. Retain the old result until the replacement is complete. Do not
  display staging directories as completed outputs.
- `validation.enabled=false` means `passed=null`, never a validation pass.
  Zero source properties means retention=null, not 0% or a fabricated 100%.

### 7.3 Option behavior

Run-selected expands folder selections using recursion/pattern and deduplicates
overlapping selections. An individually selected IFC is an explicit target;
filename patterns govern folder discovery. Search only changes the displayed
tree, not the discovery configuration or Run-all scope. Selection count counts
selected entries, while job total counts resolved unique files.

Basic and advanced options must stay synchronized. Invalid numeric/array text
must not be silently replaced with the last valid value on opening the dialog.
Disable both Run-selected and Run-all until corrected. In-flight jobs use frozen
configuration. Validate/Report act on the selected completed USD and its recorded
source/configuration, not on edited options for the next conversion. Merely
opening a report does not select a different USD for these actions.

The generated per-file CLI commands are convenience examples: independently
running `convert` for a nested file does not recreate the web root's entire
relative output hierarchy. For an exact multi-file/nested replay use `batch`
with the same root and exported configuration, and document the target set.

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
ifc2usd web input -o out --host 127.0.0.1 --port 5000 --theme dark --locale en
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
| `--locale en` | `web.locale` / `en` | English only; reject other values |
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
| 1 | Any conversion failure or cancellation; partial batch failure is non-zero |
| 2 | Quality warning: skipped geometry, failed validation or property retention below threshold |
| 3 | Configuration error |

---

## 9. Architecture

```
ifc2usd/
  cli.py             argparse entry point, exit-code decision
  config.py          default merge, schema validation, path resolution
  reader/
    ifc_reader.py    ifcopenshell parsing: units, spatial structure, properties
    geometry.py      float64 local-origin offset computation
  writer/
    usd_writer.py    stage creation, hierarchy, customData
  validate.py        USD re-read and round-trip error
  report.py          JSON/Markdown reports and aggregation
  batch.py           folder discovery, parallel execution, summary
  logging_setup.py   file and console loggers
  core.py            single-file pipeline, cache checks, staging and publication
  cancellation.py    cooperative cancellation signal
  web/
    app.py          Flask app factory, local API, session and root restrictions
    jobs.py         background process jobs, cancellation, progress snapshots
    preview.py      USD-to-GLB adapter, metadata index, preview cache
    templates/      Jinja2 workspace shell
    static/         bundled CSS, JavaScript, fonts, icons, locales and Three.js
tests/
docs/
frontend/            main.ts UI, viewer.ts rendering, locales.ts English labels
scripts/             synthetic fixtures and real-office web checks
```

### 9.1 Module Interfaces

```python
# reader/ifc_reader.py
def read_ifc(path: Path, cfg: dict, cancel=None) -> IfcModel: ...

@dataclass
class IfcModel:
    schema: str                       # "IFC2X3" | "IFC4X3" | ...
    meters_per_unit: float
    elements: list[Element]           # conversion targets
    skipped: list[SkipRecord]         # guid, class, reason
    spatial: dict[str, list]          # guid -> ordered (guid, name) parent chain
    stats: dict                       # source property totals, etc.

@dataclass
class Element:
    guid: str
    ifc_class: str
    name: str
    verts: np.ndarray                 # (N, 3) float64  <- never float32
    faces: np.ndarray                 # (M, 3) int32
    props: dict[str, str | int | float | bool]
    description: str
    spatial: list                    # ordered (guid, name) parent chain

# writer/usd_writer.py
def write_usd(model: IfcModel, out: Path, cfg: dict, cancel=None) -> WriteResult: ...
# WriteResult: scene_path, origin_offset, n_prims, prop_written, prop_dropped, collisions

# validate.py
def validate(scene: Path, model: IfcModel, cfg=None) -> dict: ...
# Includes max_error_mm, rms_error_mm, property_retention, passed,
# preserved_property_count, missing_guids, mismatched_guids and enabled.
```

### 9.2 Dependencies

`ifcopenshell`, `usd-core`, `numpy`, `psutil` (Python 3.10 or later).
Supported version ranges are in `pyproject.toml`; the recorded tested Python 3.12
environment is in `constraints-tested.txt`, not a guarantee for every version.

Optional `web` extra: `Flask`, `waitress` (cross-platform local WSGI server),
`pygltflib`. Frontend: Jinja2 shell, TypeScript modules, CSS design tokens,
Three.js with `OrbitControls`/`GLTFLoader`, Lucide icons and English labels.
Use Vite for development/build only; ship built assets in the Python wheel so
end users need neither Node.js nor CDN access. Pin compatible dependency versions.
Run conversion/preview work outside request threads in bounded process workers;
Flask debug mode and its development server are not the distributed runtime.

> Verify wheel availability for the target architecture before installation.
> If unavailable, use an environment that supplies `pxr` and document its provenance;
> `ifc2usd doctor` checks imports. Do not assume x86-64 results establish ARM support.

### 9.3 Local Web API and Safety

| Endpoint | Responsibility |
|---|---|
| `GET /api/workspace` | Roots, validated configuration, schema/defaults and active job snapshot |
| `GET /api/tree?root=input\|output&path=...&query=...` | Lazy listing; a nonempty query searches descendants. Returns server-issued IDs, relative paths and statuses. |
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

### 9.4 Concrete API sequence

Use a single same-origin session for the following calls. `GET /api/workspace`
returns `csrf`; send it as `X-CSRF-Token` on every POST. Obtain entry IDs from the
tree API, never construct them from paths in frontend code.

```json
{
  "action": "convert",
  "all": false,
  "entries": ["server-issued-input-entry-id"],
  "configuration": {"output": {"format": "usda"}},
  "confirm_overwrite": false
}
```

Submit to `POST /api/jobs`. HTTP 202 returns `{"id":"job-id"}`. Poll
`GET /api/jobs/job-id` until status leaves `queued|running`; display individual
`files`/`result.results` errors and warnings even if the job has no top-level error.
`all=true` converts the root. Validate/Report use output scene IDs instead of input
IDs. HTTP 409 `confirm_overwrite` requires a user decision, then a retry with
`confirm_overwrite=true`; an `active_job` conflict is a different 409 case.

For preview, submit `{"entry":"output-scene-id"}` to `POST /api/previews`, poll
its returned ID, then load the ready `glb` and `metadata` URLs. The metadata index
maps prim paths to IFC identity/properties and the `guide` flag. Ignore an obsolete
preview response if the user has selected another scene in the meantime.
All errors must remain actionable without disabling artifact download.

---

## 10. Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR-1 | Reference small-model tests at 500 km have round-trip error at or below 0.01 mm; report actual error for arbitrary models rather than promise a universal bound |
| NFR-2 | Benchmark target, not yet established: 10,000 elements within five minutes on a single core |
| NFR-3 | Future benchmark/design target: 100,000 elements within 8 GB. Current reader and USD stage retain the model in memory; streaming is not implemented. |
| NFR-4 | Byte-identical USD output for identical input and configuration |
| NFR-5 | Linux and Windows, Python 3.10 or later |
| NFR-6 | `ifc2usd convert model.ifc` works in one line after `pip install` |
| NFR-7 | Web workspace supports current stable Chrome/Edge/Firefox on desktop Windows/Linux with WebGL2; narrow layouts retain conversion/report access and unsupported rendering shows a fallback |
| NFR-8 | On a documented reference desktop and 100,000-triangle preview fixture, target at least 30 FPS during orbit at 1920x1080; job status updates within 2 s and cached tree/menu actions within 200 ms; record hardware/browser and cold-preview timing separately |
| NFR-9 | Target WCAG 2.2 AA contrast/keyboard requirements, reduced motion and 200% zoom in English with both themes; full conformance requires an audit |
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
| AC-12 | Theme changes preserve active jobs, selections and camera; preferences persist with explicit CLI overrides taking priority; all authored labels/errors are English | Playwright reload and launch-precedence tests |
| AC-13 | Mouse/keyboard splitters respect limits and persist; no overlap at 1920x1080, 1280x720 and 390x844, or at 200% zoom | Playwright screenshots and accessibility checks in both themes |
| AC-14 | Web and CLI jobs with equivalent resolved settings produce identical USD and equivalent conversion metrics; UI-only changes do not invalidate conversion reuse, conversion-option changes do | Cross-entry-point integration and cache regression tests |
| AC-15 | UI stays responsive during conversion; cancel/reconnect/partial failure preserve completed outputs; preview failure leaves successful conversion intact | Background-job lifecycle and failure-injection tests |
| AC-16 | Traversal, junction/symlink escapes, external USD references and cross-origin mutation are rejected; missing WebGL2 and empty scenes show correct fallback states | Security integration and browser fallback tests |
| AC-17 | Bundled web assets work offline, headless install remains usable without the extra, and reference-scene responsiveness meets NFR-8 | Packaging smoke tests and recorded browser performance run |
| AC-18 | `input/Office_A_20110811.ifc` converts all 1,083 represented products and 50,782 properties; its 99 spaces and 181 openings remain inspectable guides while the physical building is visible by default | Real-file web conversion, USD re-read, preview and report checks |
| AC-19 | An output-unit override preserves world coordinates in metres; malformed options block execution; folder/file overlap creates one target per source and existing-output failures show their reason | Unit and browser regression tests |

---

## 12. Milestones

| Stage | Scope | Deliverable |
|---|---|---|
| M1 | Single-file conversion, coordinate precision, basic report | `convert`, `report.json` |
| M2 | Full semantics preservation, spatial hierarchy, validation | `validate`, retention metrics |
| M3 | Batch conversion, parallelism, aggregate report | `batch`, `summary.md` |
| M4 | Flask workspace, shared configuration, trees, job API and CLI launch options | `ifc2usd web`, optional web extra, conversion workflow tests |
| M5 | USD preview, guide-volume visibility, render modes/transparency, splitters, themes and English UI | Three-panel viewer, accessibility and browser acceptance tests |
| M6 | Conversion/viewer performance tuning, local security, offline packaging, documentation, release | PyPI package, CLI/web user guide, acceptance evidence |

---

## 13. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| No `usd-core` wheel on aarch64 | Install fails | `doctor` pre-check, document alternative distributions |
| Large models exceed memory | Conversion aborts | Measure current memory limits; streaming/chunking is future work, not an existing guarantee |
| Geometry varies with tessellation parameters | Reproducibility suffers | Include the configuration snapshot in the report |
| Diverse property value types | Serialisation failures | Serialise non-scalars to strings, count failures |
| Prim path collisions | Elements lost | Suffix disambiguation and collision count (FR-S5) |
| Schema differences across IFC versions | Mapping gaps | Maintain IFC2X3 / IFC4 / IFC4X3 regression sets |
| Browser cannot directly display generated USD | Blank or incorrect preview | Server-side USD-to-GLB adapter, output-format fixtures and GUID/transform checks |
| Large previews or overlapping transparent surfaces | GPU exhaustion, low FPS or sorting artifacts | Lazy previews, GPU resource disposal, measured scene limits and explicit fidelity warnings |
| Local filesystem/API exposed through the browser | Unauthorized reads or conversions | Loopback binding, Host/Origin checks, CSRF, root confinement and registered artifacts only |
| Web and CLI settings diverge | Irreproducible conversions | Shared schema/core, immutable job snapshots and cross-entry-point tests |

---

## 14. Verification Evidence and Remaining Work

### 14.1 Recorded baseline (2026-10-02)

The real-model baseline applies to the exact file SHA-256
`3ccc7df480d7ae9d1af148d13634fe1af8f1a75949fdf3cb1f87c60464df4700`.
Do not enforce these counts on another model or a modified copy.

| Measurement | Recorded result |
|---|---|
| Source | IFC2X3, Autodesk Revit Architecture 2011, 4,099,307 bytes |
| Target / converted / skipped products | 1,083 / 1,083 / 0 |
| Preserved source properties | 50,782 / 50,782 |
| Vertices / triangular faces | 109,599 / 57,755 |
| Guides retained | 99 `IfcSpace` + 181 `IfcOpeningElement` |
| Maximum / RMS round-trip error | approximately 0.001289 / 0.000452 mm |
| Default output | Z-up, 1 metre per unit, auto-center with Z preserved |
| Real web workflow | conversion, preview, guide toggle, validation and report viewing exercised in Edge; no page errors |
| Python tests | 28 passing, including output-unit and guide-volume regressions |
| Browser regressions | 4 passing: conversion/viewer/preferences, mobile layout, selection/search/invalid options, and folder deduplication/format/failure details |

Evidence: generated `out/Office_A_20110811/report.json`,
`.test-output/office-web-check.json`, `.test-output/office-web.png` and
`.test-output/office-web-guides.png`. These are local ignored artifacts, not
checked-in acceptance certificates. Recreate them with the workflow below.
The roughly 13-15 second observed conversion time is a local observation, not a
performance guarantee. Software versions are recorded in the report.

### 14.2 Claims that are not established by this baseline

- Current validation checks coordinates and property values for source GUIDs.
  It does not independently certify topology, normals, all IFC placements,
  material fidelity or visual similarity. It also does not exhaustively detect
  every extra/duplicate output mesh. Add targeted tests before making those claims.
- Guide purpose prevents void/space solids from appearing as opaque building
  components in ordinary rendering. External viewers may deliberately enable
  guides. Verify their purpose settings before calling this lost geometry.
- Original IFC materials/textures, transparency and appearance are out of scope.
  The preview uses generated class colors. Missing original colors do not imply
  a geometry conversion failure.
- The reader retains all element geometry and the USD stage in memory. NFR-2,
  NFR-3 and NFR-8 need measured large-file/scene benchmarks and possibly redesign.
- Edge automation plus synthetic fixtures does not establish Firefox/Linux,
  `usdview`, all IFC schemas or a full WCAG audit. AC-7 and broader compatibility
  work remain acceptance tasks, not assumed passes.
- The partial-failure test currently covers 20 valid synthetic files plus one
  corrupt file. AC-4's 29+1 example still needs a dedicated run if that exact
  release criterion is claimed. Repeated copies are not diverse real BIM models.
- Cancellation is cooperative between elements. It cannot interrupt a blocked
  native tessellation call immediately. Validate/Report cancellation is not a
  guarantee of immediate interruption either. Jobs survive browser reconnection,
  not server restart.
- Only the snapshot at job start is frozen; browser conversion options and
  selections are not promised to survive reload. Theme/panels/render preferences
  have separate browser-local persistence.

### 14.3 Required verification workflow

Use the selected Python environment throughout:

```powershell
python -m pytest -q
# Only when frontend dependencies are not already installed:
npm ci
npm run build
python scripts/browser_fixture.py
npm run test:e2e
```

Playwright starts its own server on port 5094 by default, uses installed Microsoft
Edge and gives each run a new synthetic output folder. Override
`IFC2USD_PYTHON`, `IFC2USD_BROWSER` or `IFC2USD_TEST_PORT` for another environment.
Do not silently reuse a server on that port: it may run stale assets or different
roots. If a test leaves a process behind, identify the exact test-owned process
before stopping it. A port conflict is not a product regression.

Permission errors on an existing test cache, manifest or OS temporary directory
are environment failures. Record the exact error and use the approved permission
flow or a documented fresh test directory; do not disable path confinement or
delete unrelated outputs. Do not claim test success just because the UI loaded.

For the real model, run the normal server from section 0 in a separate terminal,
then execute:

```powershell
node scripts/check_office_web.mjs
python -m ifc2usd validate out/Office_A_20110811/scene.usda --source input/Office_A_20110811.ifc --json
```

The real-file script expects the standard `input -> out` workspace at port 5000
and creates/reuses the office result. If those outputs already exist with different
settings or a prior writer schema, first preserve them and select a fresh output
root, adapting the script, or explicitly authorize overwrite. The script must not
silently delete existing outputs. Inspect both captured screenshots: an empty
canvas or visible space/void blocks is not a satisfactory preview test.

### 14.4 Completion checklist for any subsequent change

1. Record the reproduced trigger, expected behavior and affected requirement IDs.
2. Fix the shared implementation and add a focused regression at the level where
   the defect occurred. Do not replace real-file verification with synthetic tests.
3. Rebuild frontend artifacts when relevant; restart the backend and verify roots.
4. Run the relevant tests, then the real-file flow when conversion/preview changes.
   Inspect output metrics and a rendered image, not merely process exit status.
5. Report test commands/results and remaining limitations. Keep verified facts,
   product requirements and future targets distinct when updating this document.
6. Do not commit or publish unless requested; do not add agent attribution.

## 15. Troubleshooting Without Guesswork

| Symptom | First check | Correct response |
|---|---|---|
| Expected IFC absent | `/api/workspace` roots | Launch `web input -o out`; do not use synthetic fixture roots |
| Folder name does not select contents | Folder name vs checkbox | Name expands; checkbox selects; Run resolves/deduplicates files |
| New file absent | Input refresh and active search | Refresh, clear filter if needed; remain within configured root |
| Run seems ineffective | Job result and visible error/log | Existing output may block overwrite; fix options or choose fresh destination |
| Options change physical scale | Unit conversion formula | Convert SI points to stage units, not metadata alone |
| Building looks like solid blocks | Space/opening volumes | Preserve as guides; hide by default; verify toggle and USD purpose |
| Correct report but blank canvas | Preview job result and WebGL2 | Check USD-to-GLB and browser errors separately from conversion |
| TS edit has no visible effect | Built asset timestamp and server | Run build, reload; restart for backend changes |
| Old result survives a writer change | Conversion key/schema and manifest | Invalidate writer schema; preserve old output and reconvert deliberately |
| Old preview survives an adapter change | Adapter cache version | Bump adapter version and regenerate preview |
| CSRF/session error after restart | Browser session | Reload; do not remove CSRF protection |
| Tests pass but real building looks wrong | Real-file screenshot and independent units | Investigate visual/semantic omissions beyond current round-trip checks |
