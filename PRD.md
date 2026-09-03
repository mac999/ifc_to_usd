# PRD — ifc2usd: IFC to USD Conversion Pipeline

| | |
|---|---|
| **Product** | ifc2usd |
| **Version** | 0.1 (for development kickoff) |
| **Date** | 2026-09-03 |
| **Purpose** | Product requirements at a level of detail that lets a developer start implementing from this document alone |

---

## 1. Product Overview

`ifc2usd` is a Python CLI tool that converts IFC (Industry Foundation Classes)
models into USD (Universal Scene Description) scenes.

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

### 3.2 Non-Goals (out of scope for v1)

- USD to IFC reverse conversion
- Material and texture conversion
- IFC validation (IDS, mvdXML)
- GUI or web service

---

## 4. Users and Scenarios

| User | Purpose | Scenario |
|---|---|---|
| BIM engineer | Hand a design model to a real-time viewer | Convert one IFC, inspect in usdview |
| Digital twin developer | Assemble many models into one scene | Batch-convert 30 IFC files in a folder |
| QA engineer | Check conversion quality | Review loss and error figures in the report |
| CI pipeline | Convert automatically on model update | Parse `--json`, branch on exit code |

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
  "logging": { "level": "INFO", "file": "convert.log" }
}
```

Precedence is **CLI flags > config.json > defaults**. Every key is optional.
On a schema violation, exit with code 3 and name the offending key.

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
```

Common options: `--config` (default `config.json`), `--json`, `--verbose`,
`--quiet`, `--version`

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

> `usd-core` publishes no aarch64 wheel on PyPI. On ARM, a distribution that
> bundles pxr (such as Isaac Sim) must be installed first; `ifc2usd doctor`
> checks for this.

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

---

## 12. Milestones

| Stage | Scope | Deliverable |
|---|---|---|
| M1 | Single-file conversion, coordinate precision, basic report | `convert`, `report.json` |
| M2 | Full semantics preservation, spatial hierarchy, validation | `validate`, retention metrics |
| M3 | Batch conversion, parallelism, aggregate report | `batch`, `summary.md` |
| M4 | Performance tuning, documentation, release | PyPI package, user guide |

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
