# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **`dimensions` Zarr convention** — a specification, not an application.
It is an independent git repository (remote is still `github.com/christophenoel/zarr-coords`,
to be published under `github.com/zarr-conventions/dimensions`), and is **not** part
of the surrounding `geozarr/` multi-project tree despite living inside it. Do not apply
the parent `geozarr/CLAUDE.md` (Quarto/Python/notebook) workflow here. (The convention
was formerly named `coords`; the directory is still named `coords/` but the shipped
name, namespace, and URLs are all `dimensions`.)

The convention describes each **axis** (dimension) of a Zarr array along two lines:
*axis semantics* — what the axis is (`type`, `axis`, `unit`) — and *coordinate
encoding* — how/where its values live, carried in a nested `coords` descriptor.

## The three artifacts that must stay in sync

Almost all work here is keeping these consistent with each other:

1. **`README.md`** — the human-readable specification (the primary deliverable).
   Prose, tables, and JSON snippets describing every descriptor type.
2. **`schema.json`** — the JSON Schema (draft-07) that formally validates Zarr v3
   metadata documents against the convention. The top-level maps live under
   `$defs.dimensionsAttributes` (`dimensions:axes`, `dimensions:coordinates`,
   `dimensions:version`). Axis objects are `$defs.axisObject`; auxiliary
   coordinates are `$defs.coordinateObject`; the four encoding descriptor shapes
   are the `oneOf` in `$defs.coordDescriptor` (referenced by the `coords` field
   of both).
3. **`examples/*.json`** — complete, valid Zarr metadata documents (`dimensions*.json`)
   that exercise each descriptor type and double as the test corpus.

When you change a descriptor (add a `type`, add/rename a field, change a
constraint), you almost always touch all three: the prose in README, the
`oneOf` branch in `$defs.coordDescriptor`, and at least one example.

## Build / validate

```bash
npm install        # installs ajv (only dependency)
npm test           # validates every examples/*.json against schema.json
```

`npm test` runs `test.js`, which shells out to `validate.js` (ajv, `allErrors`)
once per example file. To validate a single document:

```bash
node validate.js schema.json examples/dimensions.json
```

This is ESM (`"type": "module"` in package.json). `package-lock.json` and
`node_modules/` are gitignored.

## Core design invariants (don't break these)

- **Two roles per axis: semantics vs. encoding.** An axis object carries STAC-aligned
  *semantics* (`type` = spatial/temporal/…, `axis` = x/y/z, `unit`, `description`) and a
  nested `coords` object carrying the *encoding*. Both are `additionalProperties: true`
  so foreign ecosystems (e.g. CF `standard_name`, `calendar`) can ride along — coordinate
  *semantics* beyond `type`/`axis`/`unit` are deliberately out of scope (future `cf:`).
- **Encoding descriptors are distinguished by `coords.type`** and validated with `oneOf`:
  `array`, `reference`, `inline`, `interval` (in `$defs.coordDescriptor`). `coords` is
  optional on an axis; its absence = implicit integer index `0..n-1`.
- **`dimensions:axes` keys are dimension names** from Zarr v3 `dimension_names`.
  Auxiliary / multi-dimensional coordinates (not themselves dimensions) live in the
  separate **`dimensions:coordinates`** map, keyed by coordinate name, each carrying
  `indexed_by` (the dimensions it varies along) + a `coords` descriptor — the CF
  auxiliary-coordinate model: `lat(sample)`, `lat(y,x)`.
- **Spatial delegation:** affine-georeferenced spatial axes are *not* described
  here; their `coords` is `{type: "reference", convention: "spatial"}`, delegating to
  the sibling `spatial` convention. Explicit `lat`/`lon` arrays use `coords.type: array`.
  CRS is delegated to `proj` (no `reference_system` field here).
- **Versioning is integer-major + URL pin.** The major version lives in the URLs
  (`/refs/tags/v1/schema.json`, `/blob/v1/README.md`) and in the schema's `const`
  values (schema_url, spec_url, uuid, name, description are all pinned with
  `const`). All v1.x changes must be **additive** (new optional fields, new
  `type` enum values, broadened ranges). Renaming/retyping/removing a field
  requires a `v2` tag and a fresh schema — never edit v1 destructively.
- The convention UUID `6ca4454a-658a-4348-a667-b39ced0e58cb` is permanent.

## Zarr v2 vs v3

The schema validates **v3** documents (`dimension_names` at top level). v2
support is spec-only: dimension names come from the `_ARRAY_DIMENSIONS`
attribute and metadata lives under `.zattrs`. See the "Zarr v2 compatibility"
section in README.md. Don't add v2-specific branches to schema.json.

## Conventions when editing

- README JSON snippets and `examples/*.json` must remain valid against
  `schema.json` — run `npm test` after edits.
- Keep the cross-references between README sections and example filenames
  intact (README's "Examples" section names each file and what it demonstrates).
- Match existing prose style: RFC-style MUST/SHOULD/MAY, `dimensions:`-prefixed
  field names, and the existing table formats.
