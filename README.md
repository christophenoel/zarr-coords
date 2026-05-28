# Coordinates Convention

- **UUID**: 6ca4454a-658a-4348-a667-b39ced0e58cb
- **Name**: "coords"
- **Namespace**: `coords:`
- **Schema URL**: "<https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json>"
- **Spec URL**: "<https://github.com/zarr-conventions/coords/blob/v1/README.md>"
- **Extension Maturity Classification**: Proposal
- **Owner**: @christophenoel

## Description

Generic mapping between Zarr array index space and coordinate space — declares
dimensions, locates coordinate descriptors, and composes with the `spatial`
convention for affine transforms or CF-style explicit arrays for non-spatial
dimensions.

This convention provides a single, generic mechanism to associate each
**dimension** of a Zarr array (or tuple of dimensions, for curvilinear
coordinates) with a **coordinate descriptor** that says *how* the
coordinate values are represented and *where* they live.

This specification is written against **Zarr v3**. Dimension *names* are
**not** redeclared by this convention — they are taken from the Zarr v3
array's `dimension_names` field (top-level array metadata, sibling of
`zarr_format` and `node_type`). `coords:coordinates` keys MUST match those
existing names.

Zarr v2 datasets can use this convention with a small adaptation — see
[Zarr v2 compatibility](#zarr-v2-compatibility) at the end of this
document.

It deliberately does not invent a new coordinate model. Instead it offers a
small set of descriptor shapes that cover the dominant ecosystems:

- **Explicit coordinate arrays** (the NetCDF / CF / Xarray model: `lat(lat)`,
  `time(time)`, curvilinear `lat(y, x)`, …).
- **Affine spatial transforms** (the GeoTIFF / GDAL model, as already
  captured by the [`spatial`](https://github.com/zarr-conventions/zarr-spatial)
  convention).
- **References to other conventions** for temporal, vertical, spectral,
  lookup-based, or other domain-specific coordinate types — without
  enlarging this spec.

All properties use the `coords:` namespace prefix and are placed at the root
`attributes` level following the [Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

## Motivation

- Provides a uniform way to declare *what each axis of a Zarr array means*
  and *where its coordinate values come from*, regardless of whether the
  axis is spatial, temporal, spectral, or domain-specific.
- Bridges two ecosystems:
  - **NetCDF/CF/Xarray** — coordinates as explicit arrays, with dimension
    names carried by the Zarr v3 `dimension_names` array field. CF-style
    semantic metadata (`standard_name`, `units`, `axis`, `grid_mapping`)
    is out of scope here and may be formalized by a future `cf:`
    convention.
  - **GeoTIFF/GDAL/GeoZarr** — coordinates as affine transforms, expressed
    by the [`spatial`](https://github.com/zarr-conventions/zarr-spatial)
    convention.
- Composable with [`spatial`](https://github.com/zarr-conventions/zarr-spatial)
  (affine georeferencing),
  [`proj`](https://github.com/zarr-conventions/zarr-proj) (CRS),
  and [`multiscales`](https://github.com/zarr-conventions/multiscales)
  (each level can independently declare its own coordinates).
- Uses **integer-major + URL pin** versioning: the `schema_url` carries the
  major (`/refs/tags/v1/schema.json`); all v1.x changes are additive.

### Composes with

- **[`spatial`](https://github.com/zarr-conventions/zarr-spatial)** — when an
  axis is governed by an affine transform, declare it with
  `{type: "affine", convention: "spatial"}` and let the existing
  `spatial:transform` attribute carry the matrix.
- **[`proj`](https://github.com/zarr-conventions/zarr-proj)** — provides the
  CRS for spatial axes; orthogonal to `coords:`, applied on the same node.
- **[`multiscales`](https://github.com/zarr-conventions/multiscales)** —
  domain-agnostic per-axis resampling pyramid. Each level independently
  carries its own `dimension_names` and `coords:coordinates`, so pyramids
  can downsample non-spatial axes (time, band, …) just as uniformly as
  spatial ones. See
  [Relationship with the `multiscales` convention](#relationship-with-the-multiscales-convention).

## Convention Registration

The convention must be registered in `zarr_conventions`:

```json
{
  "zarr_conventions": [
    {
      "schema_url": "https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json",
      "spec_url": "https://github.com/zarr-conventions/coords/blob/v1/README.md",
      "uuid": "6ca4454a-658a-4348-a667-b39ced0e58cb",
      "name": "coords",
      "description": "Generic mapping between Zarr array index space and coordinate space — declares dimensions, locates coordinate descriptors, and composes with the spatial convention for affine transforms or CF-style explicit arrays for non-spatial dimensions."
    }
  ]
}
```

## Applicable To

This convention can be used with these parts of the Zarr hierarchy:

- [x] Group
- [x] Array

On **arrays**, `coords:coordinates` keys reference the array's own
Zarr v3 `dimension_names`. On **groups**, `coords:coordinates` can act as
a group-level catalogue of coordinate descriptors shared by child arrays;
keys reference dimension names used by those children.

## Dimension names — relying on Zarr v3 `dimension_names`

This convention deliberately does **not** introduce a new field for
dimension names. Every Zarr v3 array carries a top-level
`dimension_names` field alongside `zarr_format`, `node_type`, `shape`,
etc.:

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time", "y", "x"],
  "attributes": { /* coords:coordinates lives here */ }
}
```

All `coords:coordinates` map keys MUST resolve to entries in that array's
`dimension_names`. Tuple keys for multi-D coordinates (e.g. `"y,x"` for
`lat(y, x)`) are comma-joined dimension names from that same source.

For Zarr v2 datasets, see [Zarr v2 compatibility](#zarr-v2-compatibility).

## Properties

All properties use the `coords:` namespace prefix and are placed at the root
`attributes` level.

| Field Name           | Type    | Required                | Description |
|----------------------|---------|-------------------------|-------------|
| `coords:coordinates` | `object` | Optional               | Map from dimension name (or comma-joined tuple for multi-D coordinates) to a [coordinate descriptor](#coordinate-descriptors). Keys reference Zarr v3 `dimension_names` — see [Dimension names](#dimension-names--relying-on-zarr-v3-dimension_names). |
| `coords:version`     | `integer` | Optional              | Major version pin (currently `1`). Optional because `schema_url` already pins the major. |

### Additional Properties

Additional properties are allowed.

### `coords:coordinates`

Map from a dimension name — or a comma-joined tuple of dimension names for
multi-dimensional coordinates such as `lat(y, x)` (`"y,x"`) — to a
**coordinate descriptor**.

- **Type**: object (map)
- **Required**: no
- **Keys**: dimension name from the array's Zarr v3 `dimension_names`, or
  a comma-joined tuple like `"y,x"` for curvilinear / multi-D coordinates.
- **Values**: a [Coordinate descriptor](#coordinate-descriptors).

### `coords:version`

Optional integer pinning the major version of the convention this metadata
was authored against. Currently `1`. Readers MAY use it as a sanity check
in addition to the `schema_url`. Omitting it is fine — `schema_url` already
pins the major.

## Coordinate descriptors

Each value in `coords:coordinates` is one of the following shapes,
distinguished by the `type` field. The fields defined below are the only
ones this convention specifies; descriptors carry `additionalProperties:
true`, so other ecosystems can attach their own metadata without
conflicting — see [Coordinate semantics are out of
scope](#coordinate-semantics-are-out-of-scope).

### `type: "array"` — explicit coordinate array

```json
{
  "type": "array",
  "path": "../time"
}
```

- `path` is a **Zarr-relative path** to a sibling array holding the
  coordinate values.
- Multi-dimensional coordinate arrays (`lat(y, x)`, swath geometries) are
  supported: the target array's own Zarr v3 `dimension_names` declares its
  shape, and the map key in the parent's `coords:coordinates` is a
  comma-joined tuple (e.g. `"y,x"`).

### `type: "affine"` — delegate to the `spatial` convention

```json
{
  "type": "affine",
  "convention": "spatial"
}
```

Indicates that the coordinate is derived from the `spatial:transform`
affine matrix declared on the same node (or an ancestor group). Use this
when georeferencing is already expressed via the `spatial` convention and
you only need to map a generic dimension name to it.

### `type: "reference"` — delegate to another convention

```json
{
  "type": "reference",
  "convention": "proj"
}
```

Delegates this coordinate's semantics to a named sibling convention.
Intended for future coordinate families — temporal calendars, vertical
levels, spectral bands, lookup tables, or domain-specific axes — without
expanding this convention.

### `type: "inline"` — embed small coordinate vectors

```json
{
  "type": "inline",
  "values": [0.490, 0.560, 0.665, 0.842]
}
```

Embeds the coordinate values directly in the metadata. Intended for short
auxiliary axes (e.g. a 4-band spectral axis) where allocating a separate
Zarr array would be wasteful. Keep these small — readers MAY refuse to
interpret very large inline arrays.

## Coordinate semantics are out of scope

This convention is deliberately limited to *locating* coordinates. It does
**not** define how to describe their *semantics* — units of measure,
calendar, axis role, standard name, etc. Those concerns belong to other
specifications:

- The [CF conventions](https://cfconventions.org/) define
  `standard_name`, `long_name`, `units`, `axis`, `calendar`,
  `grid_mapping`, and related metadata.
- A future dedicated **`cf:`** Zarr convention may formalize a CF-aligned
  attribute namespace for use alongside `coords:`. Until such a convention
  exists, CF-style fields MAY appear directly inside a coordinate
  descriptor — descriptors are `additionalProperties: true`, so readers
  that understand CF can pick them up while `coords:` validators ignore
  them.

### Example — CF metadata carried alongside a `type: "array"` descriptor

This is the only example in this spec that includes CF fields. It is shown
to illustrate the future composition path; the underlined fields below are
**not** defined by `coords:` and are passed through verbatim:

```json
{
  "type": "array",
  "path": "../time",
  "standard_name": "time",
  "long_name": "observation time",
  "units": "seconds since 2020-01-01",
  "axis": "T",
  "calendar": "proleptic_gregorian"
}
```

A future `cf:` convention would register the CF vocabulary explicitly
(via its own `schema_url` in `zarr_conventions`) and validate these
fields. For now, treat them as opportunistic interop metadata rather than
part of `coords:`.

## Relationship with the `spatial` convention

This convention is intentionally **broader** than `spatial`:

- [`spatial`](https://github.com/zarr-conventions/zarr-spatial) describes
  affine georeferencing (origin + cell size + CRS hook) for the spatial
  dimensions of a grid.
- `coords:` describes *any* dimension — spatial, temporal, vertical,
  spectral, or domain-specific — and how to find its coordinate
  representation.

The two compose cleanly:

1. **Routing an existing affine grid.** A dataset already using
   `spatial:transform` with `dimension_names: ["y", "x"]` can adopt
   `coords:` to reuse the affine transform via
   `coords:coordinates: {"y": {"type": "affine", "convention": "spatial"},
   "x": {"type": "affine", "convention": "spatial"}}`.
2. **Mixed spatial + non-spatial dimensions.** A `(time, y, x)` cube can
   carry a `time` axis described as an explicit coordinate array and `y` /
   `x` axes delegated to the `spatial` convention's affine transform — all
   declared uniformly inside one `coords:coordinates` map.
3. **Irregular / curvilinear / swath data.** When the affine model does
   not apply, declare the spatial axes via `type: "array"` with paths to
   2-D `lat(y, x)` / `lon(y, x)` arrays. The `spatial` convention is
   simply not used in that case.
4. **Future coordinate families.** Temporal, vertical, spectral, lookup,
   or domain-specific coordinate types can plug in either by extending the
   `type` enum in a future major version of this convention, or by adding
   their own sibling convention and being referenced via
   `type: "reference"`.

In short: `spatial:` remains the authoritative declaration of an affine
georeferencing matrix. `coords:` is the index-space → coordinate-space
*router* — it names the axes and tells readers where to look for each
axis's coordinate values, whether that is the affine transform, an
explicit array, an inline vector, or another convention.

## Relationship with the `multiscales` convention

The
[`multiscales`](https://github.com/zarr-conventions/multiscales)
convention is **domain-agnostic** by design: each level's
`transform.scale` and `transform.translation` are per-axis arrays of length
equal to the array rank, not assumed to be spatial. A pyramid can
therefore downsample (or upsample) **any** dimension — temporal,
spectral, vertical, or domain-specific — not only spatial ones.

`coords:` slots in cleanly because each multiscales level *is its own
Zarr array (or group) node*, so each level independently carries:

- its Zarr v3 `dimension_names`, and
- its own `coords:coordinates` map.

Readers therefore route each level's axes through `coords:` exactly as
they would for a single-level array. The two conventions stay orthogonal:
`multiscales` describes the resampling relationship *between* levels;
`coords:` describes the index-to-coordinate mapping *within* each level.

### Composition pattern

The group declares the pyramid layout; each level array declares its own
dimension names and `coords:coordinates`. The descriptor `path` values
inside `coords:coordinates` are level-local — typically each level has its
own coordinate arrays sized to that level.

```text
my_cube/                    # group: zarr_conventions = [multiscales, coords]
├── 0/                      # native resolution
│   ├── data                # array: dimension_names=[time,y,x], coords:coordinates={...}
│   ├── time                # array: 1-D time coordinate at native step
│   ├── y                   # array: 1-D y coordinate
│   └── x                   # array: 1-D x coordinate
├── 1/                      # half-res in y/x, 7-day temporal aggregation
│   ├── data
│   ├── time                # 1-D time coordinate at weekly step
│   ├── y
│   └── x
└── 2/                      # quarter-res in y/x, monthly temporal aggregation
    ├── data
    ├── time
    ├── y
    └── x
```

### Example — N-D pyramid that also downsamples a non-spatial axis

Group node (`my_cube/zarr.json`):

```json
{
  "zarr_format": 3,
  "node_type": "group",
  "attributes": {
    "zarr_conventions": [
      { "name": "multiscales", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/multiscales/refs/tags/v1/schema.json" },
      { "name": "coords",      "schema_url": "https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json" }
    ],
    "multiscales": {
      "layout": [
        { "asset": "0/data", "transform": { "scale": [1, 1, 1] } },
        { "asset": "1/data", "derived_from": "0/data", "transform": { "scale": [7,  2, 2] } },
        { "asset": "2/data", "derived_from": "1/data", "transform": { "scale": [4,  2, 2] } }
      ],
      "resampling_method": "mean"
    }
  }
}
```

The `transform.scale` arrays carry one entry per dimension. The first
slot (`7`, then `4`) is the temporal downsampling factor — daily → weekly
→ monthly — exactly the same machinery that downsamples `y`/`x`.

Native-resolution level array (`my_cube/0/data/zarr.json`):

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time", "y", "x"],
  "attributes": {
    "zarr_conventions": [
      { "name": "coords", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json" }
    ],
    "coords:coordinates": {
      "time": { "type": "array", "path": "../time", "standard_name": "time", "units": "days since 2020-01-01", "axis": "T" },
      "y":    { "type": "affine", "convention": "spatial", "axis": "Y" },
      "x":    { "type": "affine", "convention": "spatial", "axis": "X" }
    }
  }
}
```

Downsampled level (`my_cube/1/data/zarr.json`) is structurally identical
but its sibling `../time` array holds the weekly-step values, and its
`spatial:transform` (declared on the same level node) reflects the 2× y/x
scaling:

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time", "y", "x"],
  "attributes": {
    "zarr_conventions": [
      { "name": "coords", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/coords/refs/tags/v1/schema.json" }
    ],
    "coords:coordinates": {
      "time": { "type": "array", "path": "../time", "units": "days since 2020-01-01", "axis": "T" },
      "y":    { "type": "affine", "convention": "spatial", "axis": "Y" },
      "x":    { "type": "affine", "convention": "spatial", "axis": "X" }
    }
  }
}
```

Notes on this composition:

- **Per-level coordinate independence.** Each level keeps its own
  `coords:coordinates`, so coordinates can change shape, path, or even
  representation type across levels (e.g. a level with explicit `time`
  array, a coarser level with an `inline` time vector).
- **Non-spatial pyramids work too.** Nothing in `coords:` or in
  `multiscales` requires a spatial axis. A pure `(time, band)` cube can
  carry a `(time-aggregation, band-thinning)` pyramid by setting the
  per-axis `transform.scale` entries accordingly.
- **Mixed downsampling factors are first-class.** The `transform.scale`
  vector can use `1` for axes that are *not* downsampled at a given level
  — e.g. `[1, 2, 2]` downsamples only `y`/`x`, leaving `time` untouched.

## Examples

See the [examples](examples/) directory for complete Zarr convention
metadata examples:

- [examples/coords.json](examples/coords.json) — minimal example mixing an
  explicit time coordinate array with affine `y`/`x` axes delegated to the
  `spatial` convention.

## Versioning and Compatibility

This convention follows the **integer-major with URL pin** contract
(contract #4 in the [Zarr Conventions Guidance Implementation Contracts](https://zarr-conventions.github.io/zarr-conventions-guidance/implementation-contracts)):

- The `schema_url` and `spec_url` carry the integer major version
  (`/refs/tags/v1/...`, `/blob/v1/...`).
- All v1.x changes are additive: new optional fields, new `type` values in
  the coordinate descriptor enum, and broadened ranges only.
- Breaking changes (renaming, retyping, removing, or semantically shifting
  existing fields) require a new major: tag `v2` and publish a fresh
  schema under `/refs/tags/v2/schema.json`.
- Readers SHOULD tolerate unknown `type` values in coordinate descriptors
  and unknown additional fields per the conventions framework's
  safely-ignorable principle.

## Zarr v2 compatibility

This convention is specified against Zarr v3. Zarr v2 datasets can use it
with two adaptations, both already established in the Zarr / Xarray
ecosystem:

1. **Dimension names live in attributes, not at the top level.** Zarr v2
   has no `dimension_names` field. The de-facto equivalent — established
   by Xarray — is the `_ARRAY_DIMENSIONS` attribute on each array, an
   ordered list of strings stored under `.zattrs`. Implementations
   consuming `coords:` on a Zarr v2 array MUST treat `_ARRAY_DIMENSIONS`
   as the source of dimension names that `coords:coordinates` keys
   reference.

2. **Convention metadata location is unchanged.** The `zarr_conventions`
   registration array and the `coords:coordinates` / `coords:version`
   fields live in the same attribute location as in v3 (i.e. under
   `.zattrs` rather than under the `attributes` key of a v3 array's
   metadata document).

Side-by-side:

| Concept              | Zarr v3                                              | Zarr v2 (with Xarray convention)                |
|----------------------|------------------------------------------------------|-------------------------------------------------|
| Dimension names      | Top-level `dimension_names` field on array metadata  | `_ARRAY_DIMENSIONS` attribute in `.zattrs`      |
| Convention registry  | `attributes.zarr_conventions`                        | `.zattrs.zarr_conventions`                      |
| `coords:coordinates` | `attributes["coords:coordinates"]`                   | `.zattrs["coords:coordinates"]`                 |
| Descriptor `path`    | Zarr-relative path to a sibling array                | Same                                            |
| Multi-D tuple keys   | Comma-joined names from `dimension_names`            | Comma-joined names from `_ARRAY_DIMENSIONS`     |

Everything else in this specification — the four descriptor `type` values,
the composition with `spatial` / `proj` / `multiscales`, the
out-of-scope status of CF semantics — applies unchanged to Zarr v2.

The JSON Schema in this repository validates Zarr v3 metadata documents.
Validating Zarr v2 datasets is left to v2-aware tooling: a v2 validator
would synthesize an equivalent document from the v2 array's `.zarray` and
`.zattrs` files and then apply the same schema.

## Acknowledgements

The template is based on the [STAC extensions template](https://github.com/stac-extensions/template/blob/main/README.md).

Coordinate-array semantics are inspired by the
[CF conventions](https://cfconventions.org/) and the Xarray data model.
The affine spatial composition path mirrors the
[Zarr `spatial` convention](https://github.com/zarr-conventions/zarr-spatial).
