# Dimensions Convention

> 📖 New to this convention? Start with the
> [introduction article](https://medium.com/@christophe.noel/zarr-coords-convention-65036eeab7b4)
> for the motivation and a guided walkthrough (published under the convention's
> former name, `coords`).

- **UUID**: 6ca4454a-658a-4348-a667-b39ced0e58cb
- **Name**: "dimensions"
- **Namespace**: `dimensions:`
- **Schema URL**: "<https://raw.githubusercontent.com/zarr-conventions/dimensions/refs/tags/v1/schema.json>"
- **Spec URL**: "<https://github.com/zarr-conventions/dimensions/blob/v1/README.md>"
- **Extension Maturity Classification**: Proposal
- **Owner**: @christophenoel

## Description

Domain-agnostic mapping between a Zarr array's index space (its **axes**) and
coordinate space (how each axis's coordinate values are **encoded**).

This convention describes each **axis** (dimension) of a Zarr array along two
distinct lines:

- **Axis semantics** — *what the axis is*: its kind (`spatial`, `temporal`, …),
  its spatial role (`x` / `y` / `z`), its unit. This is the STAC datacube
  *dimension* view, minus the data-variable part.
- **Coordinate encoding** — *how and where the axis's coordinate values live*:
  an explicit sibling array, an inline vector, a regularly spaced interval, or a
  delegation to another convention. This is carried in a nested `coords` object
  and is the same uniform machinery for every axis kind.

It deliberately does not invent a new coordinate model. The `coords` encoding
offers a small set of descriptor shapes:

- **Explicit coordinate arrays** — a sibling Zarr array holds the per-index
  values for the dimension (the NetCDF / Xarray model). 1-D for regular
  axes such as `time(time)`, `level(level)`, or `band(band)`; N-D auxiliary
  coordinates (see [`dimensions:coordinates`](#dimensionscoordinates)) where
  coordinates depend on more than one index dimension.
- **Inline coordinate values** — short value vectors embedded directly in
  the metadata (for example, a 4-band spectral axis where allocating a
  separate array would be wasteful).
- **Implicit regularly spaced values** — a compact `start` / `end` / `step`
  descriptor for axes that are uniformly spaced, covering both numeric
  domains (angles, distances, frequencies, levels) and ISO 8601 time
  intervals, without enumerating every value.
- **Delegation via composition** — for coordinate kinds that have their
  own Zarr convention, the descriptor delegates to it via a generic
  `reference` descriptor instead of carrying the values. Future conventions
  for temporal calendars, vertical levels, spectral bands, lookup tables,
  or other domains plug in this way.
- **Geospatial use** — spatial coordinates follow one of two traditions:
  explicit `lat` / `lon` (or projected `x` / `y`) arrays — the NetCDF / CF
  model, sometimes paired with `grid_mapping` — use the `array` descriptor
  above; an affine geotransform on a regular grid — the GeoTIFF / GDAL
  model — uses a `reference` descriptor delegating to the
  [`spatial`](https://github.com/zarr-conventions/spatial) convention.

All properties use the `dimensions:` namespace prefix and are placed at the root
`attributes` level following the [Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

## Motivation

- Provides a uniform way to declare *what each axis of a Zarr array means*
  (its semantics) and *where its coordinate values come from* (its encoding),
  regardless of whether the axis is spatial, temporal, spectral, or
  domain-specific.
- Aligns the axis object with the STAC datacube
  [`cube:dimensions`](https://github.com/stac-extensions/datacube) *dimension*
  vocabulary (`type`, `axis`, `unit`) — deliberately leaving out the
  data-variable part — so datacube ↔ Zarr crosswalks are structural.
- In the geospatial domain, bridges two ecosystems for spatial coordinates:
  - **NetCDF / CF / Xarray** — explicit `lat` / `lon` (or projected
    `x` / `y`) arrays, often paired with the CF `grid_mapping` attribute.
    CF semantic metadata (`standard_name`, `units`, `calendar`, `grid_mapping`)
    is out of scope here and may be formalized by a future `cf:` convention.
  - **GeoTIFF / GDAL / GeoZarr** — affine geotransform expressed by the
    [`spatial`](https://github.com/zarr-conventions/spatial)
    convention.
- Composable with [`spatial`](https://github.com/zarr-conventions/spatial)
  (affine georeferencing),
  [`proj`](https://github.com/zarr-conventions/proj) (CRS),
  and [`multiscales`](https://github.com/zarr-conventions/multiscales)
  (each level can independently declare its own axes).
- Uses **integer-major + URL pin** versioning: the `schema_url` carries the
  major (`/refs/tags/v1/schema.json`); all v1.x changes are additive.

### Composes with

- **[`spatial`](https://github.com/zarr-conventions/spatial)** — when a
  spatial axis is represented by an affine geotransform (the GeoTIFF / GDAL
  model), declare its `coords` as `{type: "reference", convention: "spatial"}`
  and let the existing `spatial:transform` attribute carry the matrix. Spatial
  axes represented as explicit `lat` / `lon` (or projected-`x` /
  projected-`y`) arrays — the NetCDF / CF model, sometimes paired with
  `grid_mapping` — use the `array` descriptor instead.
- **[`proj`](https://github.com/zarr-conventions/proj)** — provides the
  CRS for spatial axes; orthogonal to `dimensions:`, applied on the same node.
  CRS is **not** carried here (no `reference_system` field) — it is delegated
  to `proj`.
- **[`multiscales`](https://github.com/zarr-conventions/multiscales)** —
  domain-agnostic per-axis resampling pyramid. Each level independently
  carries its own `dimension_names` and `dimensions:axes`, so pyramids
  can downsample non-spatial axes (time, band, …) just as uniformly as
  spatial ones. See
  [Relationship with the `multiscales` convention](#relationship-with-the-multiscales-convention).

## Convention Registration

The convention must be registered in `zarr_conventions`:

```json
{
  "zarr_conventions": [
    {
      "schema_url": "https://raw.githubusercontent.com/zarr-conventions/dimensions/refs/tags/v1/schema.json",
      "spec_url": "https://github.com/zarr-conventions/dimensions/blob/v1/README.md",
      "uuid": "6ca4454a-658a-4348-a667-b39ced0e58cb",
      "name": "dimensions",
      "description": "Domain-agnostic mapping between a Zarr array's index space (axes) and coordinate space (coordinate encoding)."
    }
  ]
}
```

## Applicable To

This convention can be used with these parts of the Zarr hierarchy:

- [x] Group
- [x] Array

On **arrays**, `dimensions:axes` keys reference the array's own
Zarr v3 `dimension_names`; auxiliary coordinates named in `dimensions:coordinates`
use `indexed_by` to tie themselves to those dimensions. On **groups**,
`dimensions:axes` can act as a group-level catalogue of axes shared by child
arrays; keys reference dimension names used by those children.

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
  "attributes": { /* dimensions:axes lives here */ }
}
```

Each `dimensions:axes` map key MUST resolve to an entry in that array's
`dimension_names` — it names the dimension the axis object describes. Auxiliary
coordinates that are *not* themselves dimensions live in a separate
[`dimensions:coordinates`](#dimensionscoordinates) map and reference their
dimensions via `indexed_by`.

For Zarr v2 datasets, see [Zarr v2 compatibility](#zarr-v2-compatibility).

## Dimension-axis semantics vs. coordinate encoding

This convention keeps two concerns cleanly separated, and it is worth stating
their roles explicitly because they answer different questions:

- **The axis (dimension) — semantics: *what the axis is.*** Carried by the
  fields of an axis object (`type`, `axis`, `unit`, `description`). This is the
  identity layer: it says an axis is temporal, or is the projected `x` axis in
  metres, etc. It reuses the STAC datacube dimension vocabulary and maps onto
  the OGC dimension-aspect layer.
- **The coordinate — encoding: *how and where the values live.*** Carried by
  the nested `coords` object (an [encoding descriptor](#coordinate-encoding-descriptors)).
  This is the storage layer: it says the values are in a sibling array at
  `../time`, or are a regular interval, or are supplied by the `spatial`
  convention. `coords` is the **only** part of this convention that locates
  values, and it works identically for every axis kind.

Everything beyond these two roles is delegated: **CRS** to
[`proj`](https://github.com/zarr-conventions/proj), the **affine geotransform**
to [`spatial`](https://github.com/zarr-conventions/spatial), and richer
**coordinate semantics** (standard name, calendar, cell bounds) to the CF
conventions / a future `cf:` convention — see
[Coordinate semantics are out of scope](#coordinate-semantics-are-out-of-scope).

## Properties

All properties use the `dimensions:` namespace prefix and are placed at the root
`attributes` level.

| Field Name               | Type      | Required     | Description |
|--------------------------|-----------|--------------|-------------|
| `dimensions:axes`        | `object`  | **Required** | Map from a dimension name (a key in Zarr v3 `dimension_names`) to an [axis object](#axis-object). |
| `dimensions:coordinates` | `object`  | Optional     | Map from a coordinate name to an [auxiliary coordinate object](#dimensionscoordinates) — the CF auxiliary-coordinate model (`lat(y,x)`, `lat(sample)`). |
| `dimensions:version`     | `integer` | Optional     | Major version pin (currently `1`). Optional because `schema_url` already pins the major. |

### Additional Properties

Additional properties are allowed.

### `dimensions:axes`

Map from a **dimension name** to an **axis object**.

- **Type**: object (map)
- **Required**: yes — a node that registers this convention MUST carry a
  `dimensions:axes` map.
- **Keys**: a **dimension name** from the array's Zarr v3 `dimension_names`.
- **Values**: an [axis object](#axis-object).

#### Axis object

| Field         | Type     | Required | Description |
|---------------|----------|----------|-------------|
| `type`        | `string` | Optional | Axis kind. Recommended values follow STAC datacube: `spatial`, `temporal`, `geometry`, `other`. Open string — domain-specific values allowed. |
| `axis`        | `string` | Optional | Spatial axis role, when applicable. Recommended: `x`, `y`, `z`. |
| `unit`        | `string` | Optional | Unit of measurement for the coordinate values (UCUM code / UDUNITS symbol). |
| `description` | `string` | Optional | Human-readable description of the axis. |
| `coords`      | `object` | Optional | [Coordinate encoding descriptor](#coordinate-encoding-descriptors). When omitted, the axis is an implicit integer index `0..n-1`. |

Axis objects are `additionalProperties: true`, so other ecosystems (e.g. CF
`standard_name`) can attach their own metadata without conflicting — see
[Coordinate semantics are out of scope](#coordinate-semantics-are-out-of-scope).

The keys alone — semantics and `coords` omitted here for clarity — look like
this for an array with `dimension_names: ["time", "y", "x"]`:

```jsonc
"dimensions:axes": {
  "time": { … },   // key = dimension name
  "y":    { … },   // key = dimension name
  "x":    { … }    // key = dimension name
}
```

### `dimensions:coordinates`

Optional map for **auxiliary / multi-dimensional coordinates** — named
coordinates that vary along one or more axes but are not themselves dimensions.
This is the CF **auxiliary coordinate** model. It covers point observations,
trajectories, profiles, stations, and curvilinear grids — where coordinates
like `lat`, `lon`, or `time` vary along some other dimension (`sample`,
`station`, …) or along several dimensions at once (`y`, `x`).

- **Keys**: the coordinate's own name (e.g. `lat`), *not* a dimension name.
- **Values**: an auxiliary coordinate object.

#### Auxiliary coordinate object

| Field         | Type       | Required     | Description |
|---------------|------------|--------------|-------------|
| `indexed_by`  | `[string]` | **Required** | Dimension(s) the coordinate varies along. Each entry MUST be a dimension name from `dimension_names`. e.g. `["sample"]` for `lat(sample)`, or `["y","x"]` for `lat(y,x)`. |
| `type`        | `string`   | Optional     | Coordinate kind (same meaning as axis `type`). |
| `unit`        | `string`   | Optional     | Unit of measurement for the coordinate values. |
| `description` | `string`   | Optional     | Human-readable description. |
| `coords`      | `object`   | Optional     | [Coordinate encoding descriptor](#coordinate-encoding-descriptors) — normally a `type: "array"` pointing at the (possibly N-D) coordinate array. |

```json
{
  "indexed_by": ["sample"],
  "unit": "degrees_north",
  "coords": { "type": "array", "path": "../lat" }
}
```

Keyed as `lat` in `dimensions:coordinates`, this describes a latitude coordinate
whose values are indexed by the `sample` dimension:

```text
lat(sample)
```

Two or more dimensions give a multi-dimensional / curvilinear coordinate —
e.g. `indexed_by: ["y", "x"]` for `lat(y, x)`. Each name in `indexed_by` MUST
be one of the array's `dimension_names`, and the referenced coordinate array's
own `dimension_names` MUST match `indexed_by` in order. Because each entry is
keyed by the coordinate name, several auxiliary coordinates can share the same
dimensions (`lat(y, x)` *and* `lon(y, x)`) without colliding.

### `dimensions:version`

Optional integer pinning the major version of the convention this metadata
was authored against. Currently `1`. Readers MAY use it as a sanity check
in addition to the `schema_url`. Omitting it is fine — `schema_url` already
pins the major.

## Coordinate encoding descriptors

The `coords` object — on an axis or an auxiliary coordinate — is one of the
following shapes, distinguished by its `type` field (distinct from the axis
`type`). The fields defined below are the only ones this convention specifies;
descriptors carry `additionalProperties: true`, so other ecosystems can attach
their own metadata without conflicting — see
[Coordinate semantics are out of scope](#coordinate-semantics-are-out-of-scope).

### `type: "array"` — explicit coordinate array

```json
{
  "type": "array",
  "path": "../time"
}
```

- `path` is a **Zarr-relative path** to a sibling array holding the
  coordinate values.
- On an axis, the target array is 1-D along that dimension (a CF coordinate
  variable). On an auxiliary coordinate in
  [`dimensions:coordinates`](#dimensionscoordinates), the target array's shape
  matches the coordinate's `indexed_by` (e.g. 2-D for `lat(y, x)`).

### `type: "reference"` — delegate to a sibling convention

```json
{
  "type": "reference",
  "convention": "spatial"
}
```

Delegates the coordinate to a named sibling convention that supplies or defines
it. The canonical case is `convention: "spatial"`, whose affine transform
(`spatial:transform`, on the same node or an ancestor group) georeferences the
axis — the GeoTIFF / GDAL model. The same mechanism covers temporal, vertical,
spectral, lookup, or other future domain-specific conventions.

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

### `type: "interval"` — implicit regularly spaced values

```json
{
  "type": "interval",
  "start": 0,
  "end": 10,
  "step": 2
}
```

→ `0, 2, 4, 6, 8, 10`

A compact descriptor for axes whose coordinate values form a regularly
spaced sequence. Equivalent to an `inline` descriptor enumerating
`start, start + step, ..., end`, but avoids materializing the values.

- `start` is the first value of the sequence.
- `end` is **inclusive**: the last value of the sequence reaches `end`
  exactly when `(end − start)` is an integer multiple of `step`. Authors
  SHOULD ensure this; readers MAY ignore any residual fraction.
- `step` is the increment between successive values. It MUST be non-zero;
  it MAY be negative for descending sequences.

Two value domains are supported, distinguished by JSON type:

**Numeric** — `start`, `end`, `step` are all JSON numbers. Applicable to
any ordered numeric axis: integers, floating-point values, angles,
distances, frequencies, elevations, levels:

```json
{ "type": "interval", "start": 1.0, "end": 2.0, "step": 0.25 }
```

→ `1.0, 1.25, 1.5, 1.75, 2.0`

```json
{ "type": "interval", "start": 0, "end": 360, "step": 15 }
```

→ `0°, 15°, 30°, …, 360°`

**ISO 8601 (temporal)** — `start` and `end` are ISO 8601 date-time
strings; `step` is an ISO 8601 duration (e.g. `P1D`, `PT1H`, `P1M`).
This is the temporal equivalent of the numeric form, standardized by
ISO 8601:

```json
{
  "type": "interval",
  "start": "2026-01-01T00:00:00Z",
  "end":   "2026-01-31T00:00:00Z",
  "step":  "P1D"
}
```

Conceptually equivalent to the ISO 8601 interval notation
`2026-01-01T00:00:00Z/2026-01-31T00:00:00Z/P1D`, but kept as three
discrete fields so readers do not need to parse a compound string.

Calendar semantics (e.g. the CF `calendar` choice, or non-Gregorian
calendars whose step is not an ISO 8601 duration) remain out of scope — see
[Coordinate semantics are out of scope](#coordinate-semantics-are-out-of-scope).

### Putting it together

A complete array node: the Zarr v3 `dimension_names` declare the axes, and
`dimensions:axes` gives each one its semantics plus a `coords` encoding — here
an explicit `time` array alongside `y` / `x` axes whose coordinates are
delegated to the `spatial` convention.

```json
{
  "zarr_format": 3,
  "node_type": "array",
  "dimension_names": ["time", "y", "x"],
  "attributes": {
    "zarr_conventions": [
      { "name": "dimensions", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/dimensions/refs/tags/v1/schema.json" }
    ],
    "dimensions:axes": {
      "time": { "type": "temporal",                "coords": { "type": "array",     "path": "../time" } },
      "y":    { "type": "spatial", "axis": "y",     "coords": { "type": "reference", "convention": "spatial" } },
      "x":    { "type": "spatial", "axis": "x",     "coords": { "type": "reference", "convention": "spatial" } }
    },
    "dimensions:version": 1
  }
}
```

This is [examples/dimensions.json](examples/dimensions.json) (shown there with
the full `zarr_conventions` registration block). For the implicit regularly
spaced form, see [examples/dimensions-interval.json](examples/dimensions-interval.json).

## Coordinate semantics are out of scope

This convention describes an axis's *kind* (`type`, `axis`, `unit`) and
*locates* its coordinate values (`coords`). It does **not** define the finer
*semantics* of those values — standard name, calendar, cell bounds,
`grid_mapping`, etc. Those concerns belong to other specifications:

- The [CF conventions](https://cfconventions.org/) define
  `standard_name`, `long_name`, `units`, `calendar`, `bounds`,
  `grid_mapping`, and related metadata.
- A future dedicated **`cf:`** Zarr convention may formalize a CF-aligned
  attribute namespace for use alongside `dimensions:`. Until such a convention
  exists, CF-style fields MAY appear directly on an axis object (or its
  `coords` descriptor) — both are `additionalProperties: true`, so readers
  that understand CF can pick them up while `dimensions:` validators ignore
  them.

### Example — CF metadata carried alongside an axis object

This is the only example in this spec that includes CF fields. It illustrates
the future composition path; the fields below beyond `type` / `coords` are
**not** defined by `dimensions:` and are passed through verbatim:

```json
{
  "type": "temporal",
  "coords": { "type": "array", "path": "../time" },
  "standard_name": "time",
  "long_name": "observation time",
  "units": "seconds since 2020-01-01",
  "calendar": "proleptic_gregorian"
}
```

A future `cf:` convention would register the CF vocabulary explicitly
(via its own `schema_url` in `zarr_conventions`) and validate these
fields. For now, treat them as opportunistic interop metadata rather than
part of `dimensions:`.

## Relationship with the `spatial` convention

Spatial coordinates have two well-established representations in the wider
ecosystem:

- **GeoTIFF / GDAL** — an affine geotransform on a regular grid (origin +
  cell size + CRS hook). This is what the
  [`spatial`](https://github.com/zarr-conventions/spatial) convention
  captures, and what a `coords` descriptor with `{type: "reference",
  convention: "spatial"}` delegates to.
- **NetCDF / CF / Xarray** — explicit `lat` / `lon` (or projected `x` /
  `y`) coordinate arrays, often paired with a CF `grid_mapping` attribute.
  In this spec these are just regular explicit coordinate arrays, declared
  via the `array` descriptor.

This convention is intentionally **broader** than `spatial`:

- [`spatial`](https://github.com/zarr-conventions/spatial) describes
  the GeoTIFF / GDAL affine model for the spatial dimensions of a grid.
- `dimensions:` describes *any* axis — temporal, vertical, spectral,
  categorical, spatial, or domain-specific — its semantics, and how to find
  its coordinate representation, including both spatial representations above.

The two compose cleanly:

1. **Routing an existing affine grid.** A dataset already using
   `spatial:transform` with `dimension_names: ["y", "x"]` can adopt
   `dimensions:` to reuse the affine transform by giving each spatial axis a
   `{"type": "reference", "convention": "spatial"}` encoding.
2. **Mixed spatial + non-spatial dimensions.** A `(time, y, x)` cube can
   carry a `time` axis whose `coords` is an explicit coordinate array and `y` /
   `x` axes whose `coords` delegate to the `spatial` convention's affine
   transform — all declared uniformly inside one `dimensions:axes` map.
3. **Irregular / curvilinear / swath data.** When the affine model does
   not apply, declare the spatial coordinates as auxiliary coordinates in
   `dimensions:coordinates` (`lat`, `lon`) with `indexed_by: ["y", "x"]`
   pointing at 2-D `lat(y, x)` / `lon(y, x)` arrays. The `spatial`
   convention is simply not used in that case.
4. **Future coordinate families.** Temporal, vertical, spectral, lookup,
   or domain-specific coordinate types can plug in either by extending the
   encoding `type` enum in a future major version of this convention, or by
   adding their own sibling convention and being referenced via
   `type: "reference"`.

In short: `spatial:` remains the authoritative declaration of an affine
georeferencing matrix. `dimensions:` names the axes and their semantics, and
its `coords` encoding tells readers where to look for each axis's coordinate
values — whether that is the affine transform, an explicit array, an inline
vector, or another convention.

## Relationship with the `multiscales` convention

The
[`multiscales`](https://github.com/zarr-conventions/multiscales)
convention is **domain-agnostic** by design: each level's
`transform.scale` and `transform.translation` are per-axis arrays of length
equal to the array rank, not assumed to be spatial. A pyramid can
therefore downsample (or upsample) **any** dimension — temporal,
spectral, vertical, or domain-specific — not only spatial ones.

`dimensions:` slots in cleanly because each multiscales level *is its own
Zarr array (or group) node*, so each level independently carries:

- its Zarr v3 `dimension_names`, and
- its own `dimensions:axes` map.

Readers therefore route each level's axes through `dimensions:` exactly as
they would for a single-level array. The two conventions stay orthogonal:
`multiscales` describes the resampling relationship *between* levels;
`dimensions:` describes the axis semantics and index-to-coordinate mapping
*within* each level.

### Composition pattern

The group declares the pyramid layout; each level array declares its own
dimension names and `dimensions:axes`. The `coords` descriptor `path` values
are level-local — typically each level has its own coordinate arrays sized to
that level.

```text
my_cube/                    # group: zarr_conventions = [multiscales]
├── 0/                      # native resolution
│   ├── data                # array: dimension_names=[time,y,x], dimensions:axes={...}
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
      { "name": "multiscales", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/multiscales/refs/tags/v1/schema.json" }
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
      { "name": "dimensions", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/dimensions/refs/tags/v1/schema.json" }
    ],
    "dimensions:axes": {
      "time": { "type": "temporal",            "coords": { "type": "array", "path": "../time" } },
      "y":    { "type": "spatial", "axis": "y", "coords": { "type": "reference", "convention": "spatial" } },
      "x":    { "type": "spatial", "axis": "x", "coords": { "type": "reference", "convention": "spatial" } }
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
      { "name": "dimensions", "schema_url": "https://raw.githubusercontent.com/zarr-conventions/dimensions/refs/tags/v1/schema.json" }
    ],
    "dimensions:axes": {
      "time": { "type": "temporal",            "coords": { "type": "array", "path": "../time" } },
      "y":    { "type": "spatial", "axis": "y", "coords": { "type": "reference", "convention": "spatial" } },
      "x":    { "type": "spatial", "axis": "x", "coords": { "type": "reference", "convention": "spatial" } }
    }
  }
}
```

Notes on this composition:

- **Per-level coordinate independence.** Each level keeps its own
  `dimensions:axes`, so coordinates can change shape, path, or even
  encoding type across levels (e.g. a level with an explicit `time`
  array, a coarser level with an `inline` time vector).
- **Non-spatial pyramids work too.** Nothing in `dimensions:` or in
  `multiscales` requires a spatial axis. A pure `(time, band)` cube can
  carry a `(time-aggregation, band-thinning)` pyramid by setting the
  per-axis `transform.scale` entries accordingly.
- **Mixed downsampling factors are first-class.** The `transform.scale`
  vector can use `1` for axes that are *not* downsampled at a given level
  — e.g. `[1, 2, 2]` downsamples only `y`/`x`, leaving `time` untouched.

## Examples

See the [examples](examples/) directory for complete Zarr convention
metadata examples:

- [examples/dimensions.json](examples/dimensions.json) — minimal example mixing
  an explicit `time` coordinate array with affine `y`/`x` axes delegated to the
  `spatial` convention.
- [examples/dimensions-interval.json](examples/dimensions-interval.json) —
  implicit axes using `type: "interval"`: an ISO 8601 daily time interval plus
  numeric elevation and azimuth axes.
- [examples/dimensions-point-cloud.json](examples/dimensions-point-cloud.json) —
  auxiliary coordinates over a `sample` index dimension (`lat(sample)`,
  `lon(sample)`, `time(sample)`) in `dimensions:coordinates`.
- [examples/dimensions-curvilinear.json](examples/dimensions-curvilinear.json) —
  multi-dimensional coordinates `lat(y, x)` / `lon(y, x)` on a curvilinear
  grid, keyed by name in `dimensions:coordinates` with `indexed_by: ["y", "x"]`.

## Versioning and Compatibility

This convention follows the **integer-major with URL pin** contract
(contract #4 in the [Zarr Conventions Guidance Implementation Contracts](https://zarr-conventions.github.io/zarr-conventions-guidance/implementation-contracts)):

- The `schema_url` and `spec_url` carry the integer major version
  (`/refs/tags/v1/...`, `/blob/v1/...`).
- All v1.x changes are additive: new optional fields, new `type` values in
  the coordinate encoding enum, and broadened ranges only.
- Breaking changes (renaming, retyping, removing, or semantically shifting
  existing fields) require a new major: tag `v2` and publish a fresh
  schema under `/refs/tags/v2/schema.json`.
- Readers SHOULD tolerate unknown `type` values (axis kinds and encoding
  descriptors alike) and unknown additional fields per the conventions
  framework's safely-ignorable principle.

## Zarr v2 compatibility

This convention is specified against Zarr v3. Zarr v2 datasets can use it
with two adaptations, both already established in the Zarr / Xarray
ecosystem:

1. **Dimension names live in attributes, not at the top level.** Zarr v2
   has no `dimension_names` field. The de-facto equivalent — established
   by Xarray — is the `_ARRAY_DIMENSIONS` attribute on each array, an
   ordered list of strings stored under `.zattrs`. Implementations
   consuming `dimensions:` on a Zarr v2 array MUST treat `_ARRAY_DIMENSIONS`
   as the source of dimension names that `dimensions:axes` keys reference.

2. **Convention metadata location is unchanged.** The `zarr_conventions`
   registration array and the `dimensions:axes` / `dimensions:coordinates` /
   `dimensions:version` fields live in the same attribute location as in v3
   (i.e. under `.zattrs` rather than under the `attributes` key of a v3
   array's metadata document).

Side-by-side:

| Concept                  | Zarr v3                                              | Zarr v2 (with Xarray convention)                |
|--------------------------|------------------------------------------------------|-------------------------------------------------|
| Dimension names          | Top-level `dimension_names` field on array metadata  | `_ARRAY_DIMENSIONS` attribute in `.zattrs`      |
| Convention registry      | `attributes.zarr_conventions`                        | `.zattrs.zarr_conventions`                      |
| `dimensions:axes`        | `attributes["dimensions:axes"]`                      | `.zattrs["dimensions:axes"]`                    |
| Encoding `coords.path`   | Zarr-relative path to a sibling array                | Same                                            |
| Auxiliary `indexed_by`   | Dimension names from `dimension_names`               | Dimension names from `_ARRAY_DIMENSIONS`        |

Everything else in this specification — the four coordinate encoding `type`
values, the composition with `spatial` / `proj` / `multiscales`, the
out-of-scope status of CF semantics — applies unchanged to Zarr v2.

The JSON Schema in this repository validates Zarr v3 metadata documents.
Validating Zarr v2 datasets is left to v2-aware tooling: a v2 validator
would synthesize an equivalent document from the v2 array's `.zarray` and
`.zattrs` files and then apply the same schema.

## Acknowledgements

The template is based on the [STAC extensions template](https://github.com/stac-extensions/template/blob/main/README.md).
The axis object reuses the [STAC datacube](https://github.com/stac-extensions/datacube)
dimension vocabulary (minus the data-variable part).

Coordinate-array semantics are inspired by the
[CF conventions](https://cfconventions.org/) and the Xarray data model.
The affine spatial composition path delegates to the
[Zarr `spatial` convention](https://github.com/zarr-conventions/spatial)
via a `reference` descriptor.
