# IFC to USD DiY

Open your vibe coding tool, command "develop it under PRD.md".

# Summary
In reference, this Python CLI tool that converts IFC (Industry Foundation Classes) models into
USD (Universal Scene Description) scenes.

It does more than convert: it **measures what was preserved and what was lost, and
writes it to a report**. Coordinate precision, property retention, class coverage
and round-trip validation error are recorded on every run.

```bash
pip install ifc2usd
ifc2usd convert model.ifc -o out/
```

## Features

- **Single file or whole folder** — batch-convert many IFC files into one output tree
- **Precision-safe coordinates** — sub-millimetre error even in projected CRS
- **Semantics preserved** — GUID, class, property sets and quantity sets as USD customData
- **IFC spatial hierarchy kept** — Site / Building / Storey structure carried into USD
- **Conversion reports** — `report.json` (machine-readable) and `report.md` (human-readable)
- **Built-in validation** — re-reads the written USD, compares against the source, signals threshold breaches through the exit code
- **One config file** — every behaviour driven by `config.json`

## Installation

```bash
pip install ifc2usd
ifc2usd doctor      # check the runtime environment
```

Requires Python 3.10 or later, plus `ifcopenshell`, `usd-core` and `numpy`.

> **ARM64 (aarch64) note.** `usd-core` publishes no ARM64 wheel on PyPI. Install a
> distribution that bundles pxr (Isaac Sim, for example) first, then
> `pip install ifc2usd --no-deps`. `ifc2usd doctor` verifies the setup.

## Quick start

### Single file

```bash
ifc2usd convert model.ifc -o out/
cat out/model/report.md
```

### Whole folder

```bash
ifc2usd batch data/ifc -o out/ --workers 4
cat out/summary.md
```

Subfolders are discovered recursively and the same structure is recreated under the
output folder. If one file fails, the rest still convert.

### Using a config file

```bash
ifc2usd init                                # write config.json
ifc2usd batch data/ifc --config config.json
```

## Output layout

```
out/
  summary.json          batch aggregate (machine-readable)
  summary.md            batch aggregate (human-readable)
  convert.log           execution log
  model_a/
    scene.usda          converted USD scene
    report.json         conversion report (machine-readable)
    report.md           conversion report (human-readable)
  sub/model_b/
    ...
```

## Reports

Example `report.md`:

```
# ifc2usd conversion report - Duplex_A_20110907.ifc

## Input
- SHA-256          : 8f3c1a...  (12.4 MB)
- IFC schema       : IFC2X3
- Authoring tool   : Revit 2024

## Elements
- target 286 / converted 286 / skipped 0
- 14 classes (IfcWallStandardCase 87, IfcSpace 42, IfcDoor 33, ...)

## Coordinates
- offset mode      : auto_center
- offset           : (12.938, 8.204, 0.000) m
- max magnitude    : 22.2 m
- est. quantization: 0.0017 mm

## Properties
- source 15,489 / preserved 15,489 / dropped 0   (retention 100.0%)

## Validation
- round-trip vertex error : max 0.0006 mm, RMS 0.0003 mm   -> PASS
- property retention      : 100.0%                         -> PASS

## Execution
- 4.71 s / ifc2usd 0.1.0 / ifcopenshell 0.8.5
```

`summary.md` carries a one-line entry per file plus the batch totals.

## CLI

```
ifc2usd convert  <input.ifc> [-o OUT]             convert a single file
ifc2usd batch    <input_dir> [-o OUT]             convert a whole folder
ifc2usd inspect  <input.ifc>                      statistics only, no conversion
ifc2usd validate <scene.usda> --source <in.ifc>   verify an existing artifact
ifc2usd report   <out_dir>                        regenerate reports
ifc2usd init                                      write a default config.json
ifc2usd doctor                                    check the runtime environment
```

Common options

| Option | Description |
|---|---|
| `--config PATH` | Configuration file (default `config.json`) |
| `-o, --output DIR` | Output folder |
| `--json` | Print the machine-readable result to stdout |
| `--workers N` | Batch parallelism |
| `--verbose` / `--quiet` | Log verbosity |

Exit codes: `0` success, `1` conversion failure, `2` validation threshold exceeded,
`3` configuration error.

In CI, branch on `--json` output and the exit code:

```bash
ifc2usd batch data/ifc -o out/ --json || echo "conversion or validation failed"
```

## Configuration (config.json)

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

Key settings

| Key | Default | Description |
|---|---|---|
| `geometry.origin_offset.mode` | `auto_center` | `auto_center`, `explicit` or `none` |
| `geometry.origin_offset.preserve_z` | `true` | Leave elevations untouched |
| `semantics.max_props_per_element` | `0` | 0 means unlimited; when set, drops are reported |
| `scene.hierarchy` | `spatial` | `spatial`, `by_class` or `flat` |
| `batch.workers` | `4` | Auto-capped at half the physical cores |
| `batch.on_error` | `continue` | One failure does not stop the batch |
| `batch.skip_unchanged` | `true` | Skip when the input hash matches and outputs exist |
| `validation.max_vertex_error_mm` | `1.0` | Exceeding this exits with code 2 |

Precedence is **CLI flags > config.json > defaults**. Every key is optional.

## IFC to USD mapping

| IFC | USD |
|---|---|
| `IfcProject` | `/World` (defaultPrim, `metersPerUnit`, up axis Z) |
| `IfcSite` / `IfcBuilding` / `IfcBuildingStorey` | `Xform` hierarchy |
| `IfcProduct` (with Representation) | `UsdGeom.Mesh` |
| `GlobalId` | `customData["ifc_guid"]` |
| `is_a()` | `customData["ifc_class"]` |
| `Name`, `Description` | `customData["ifc_name"]`, `["ifc_description"]` |
| `IfcPropertySet` | `customData["pset:<set>.<prop>"]` |
| `IfcElementQuantity` | `customData["qto:<set>.<qty>"]` |
| — | `customLayerData["origin_offset"]` |

Prim paths follow the IFC spatial structure by default:

```
/World/Site/Building/Storey_2F/IfcWallStandardCase/g_3ObOfBTNz2dhgWxUzdmDpl
```

Prim names are normalised to valid USD identifiers, but the original GUID stays in
`customData`, so round-trip lookup by GUID always works.

## Coordinate precision

USD `points` is float32. Writing projected-CRS coordinates directly makes the error
grow with coordinate magnitude. `ifc2usd` applies a local origin offset and keeps
float64 up to the moment the offset is subtracted.

| Coordinate magnitude | Without offset | ifc2usd |
|---|---|---|
| 1,000 m | 0.063 mm | 0.0006 mm |
| 100,000 m | 5.313 mm | 0.0006 mm |
| 200,000 m | 10.760 mm | 0.0006 mm |
| 500,000 m | 21.749 mm | 0.0006 mm |

The offset is recorded in USD `customLayerData` and in the report, so the original
coordinate frame can always be restored. Elevations are left untouched by default.

## Limitations

- No USD to IFC reverse conversion
- Materials and textures are not converted (geometry and semantics only)
- IFC validation (IDS, mvdXML) is out of scope
- Curved surfaces are tessellated; the parameters used are recorded in the
  configuration snapshot inside the report
- IFC4X3 infrastructure models are often sparse in semantics. Reports distinguish
  "absent in the source" from "lost during conversion"

## Documentation

- [PRD.md](PRD.md) — product requirements document
- [README.ko.md](README.ko.md) / [PRD.ko.md](PRD.ko.md) — Korean versions

## Author

Taewook Kang (laputa99999@gmail.com)
