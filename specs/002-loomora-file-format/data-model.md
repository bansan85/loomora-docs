# Data Model: The `.loomora` Project

Logical model of a project (what the format layer holds and the C API exposes). The byte-level encoding is in [contracts/project-format.md](contracts/project-format.md); the JSON entries follow [contracts/schemas/](contracts/schemas/). Units: millimetres; coordinate system as FR-015 (origin at the centre of the frame's bounding box, X right, Y up, Z toward the viewer, front view).

## Overview

```text
Project 1 ──* Frame 1 ──* Layout 1 ──* Nail
   │            └─ (3D) Mesh
   ├──* Image
   ├──* Spool
   └──* Result ── 1 Frame, 1 Layout of that frame, 1..n Images (views on 3D),
                  a Spool selection (spool + chosen length), a Weaving sequence
```

Application data (progress, bookmarks) is **not** part of the model; it refers to `project_id` and `result.id`.

## Identifiers

- `project_id`: UUID v4, supplied by the caller at creation, never changed (FR-006).
- Item identifiers (`frame`, `layout`, `image`, `spool`, `result`): unsigned 32-bit integers, unique **within their kind**, allocated from `next_id[kind]` of the manifest, which only increases. A removed item's identifier is never reused (FR-014b). `nail.id` is unique **within its layout** (not necessarily contiguous, any `u32`).
- Display order of each kind: the `order` list of the manifest (a permutation of the existing identifiers).

## Entities

### Project (manifest)

| Field | Type | Rule |
|---|---|---|
| `format` | `{name:"loomora", major:int, minor:int}` | `name` fixed; major and minor ≥ 0 |
| `min_reader` | string `"M.m"` | not above `format` |
| `project_id` | UUID v4 string | lowercase, canonical form |
| `title` | string (UTF-8) | up to 1,024 bytes, may be empty |
| `created_at`, `modified_at` | ISO 8601 UTC, `YYYY-MM-DDTHH:MM:SS[.fff]Z` | `modified_at` ≥ `created_at` is not enforced; both supplied by the caller (R6, FR-004). The shape (month 01-12, day 01-31, hour 00-23, minute 00-59, second 00-60) decides acceptance; a time that is not a real date (day beyond the month with the Gregorian leap rule, second 60) still opens and gets the warning `warn.timestamp` |
| `generator` | `{application, version, platform, architecture}` strings | informational |
| `profile` | `full` or `light` | `light` implies `editable:false` |
| `editable` | boolean | false for `light` |
| `counts` | `{frames, layouts, images, spools, results}` | equal to the number of items (checked) |
| `next_id` | `{frame, layout, image, spool, result}` | each greater than every existing identifier of its kind (checked) |
| `order` | `{frames:[id], layouts:[id], images:[id], spools:[id], results:[id]}` | each a permutation of the items of the kind |
| `entries` | map path → `{size, sha256}` | every archive entry except `mimetype` and `manifest.json`, each exactly once; `size` is the uncompressed size |
| `required_features` | list of strings | subset of `surface_mesh`, `string_levels`, `multi_view_target`; derived from the content (checked) |
| `extensions` | object, keys `namespace.key` | preserved, never interpreted |

### Frame

| Field | Type | Rule |
|---|---|---|
| `id`, `name` | u32, string | name up to 256 bytes |
| `kind` | `2d` or `3d` | |
| `dimensions` | `{width_mm, height_mm}` (2D) or `{width_mm, height_mm, depth_mm}` (3D bounding box) | finite, > 0 |
| `background` | `#rrggbb` | |
| `outline` | `{type, params, polyline}` | `type` one of `circle ellipse rectangle heart star svg polygon`; `polyline` = resolved closed polygon `[[x,y],…]` of at least 3 points; `params` free object per type |
| `exclusions` | list of the same shape | each inside the outline (checked, tolerance 0.01 mm) |
| `surface` | `plane` (2D); `mesh`, `primitive`, `extrusion`, `revolution`, `csg` (3D) | 2D requires `plane` |
| `body` | `solid`, `shell`, `guide` | 3D only |
| `mesh` | mesh description (below) | 3D only; required when `surface` is `mesh` |

**Mesh description** (in `frame.json`, data in `mesh.bin`): `vertices`, `triangles`, `watertight`, `manifold`, `has_normals`, `has_uvs` (default false), `source` `{name, sha256}` (the imported file it came from, optional), `construction` (parametric description, optional, **full only**), `attached_source` (path of `mesh-source.<ext>`, optional, **full only**). `mesh.bin` always holds positions (f32 ×3) and triangle indices (u32 ×3); normals (f32 ×3 per vertex) when `has_normals`; texture coordinates (UV, f32 ×2 per vertex, finite) when `has_uvs`, **full profile only**: a `light` file has `has_uvs` false and a `mesh.bin` rewritten without them (FR-017).

### Layout (nail layout)

| Field | Type | Rule |
|---|---|---|
| `id`, `name` | u32, string | |
| `frame` | frame id | exists; fixed for the life of the layout (a layout cannot move to another frame) |
| `nail_count` | integer | equals the number of nails in `nails.json` (checked) |
| nails | array in `nails.json` | below |

**Nail**: `{id, x, y, diameter, kind, label?}`; 3D layouts add `z, axis[3], length, side, face:{triangle, u, v}` (`side` `exterior` or `interior`; `axis` unit vector within 1e-6; `face.triangle` < triangle count of the frame's mesh; `u, v ≥ −1e-6`, `u + v ≤ 1 + 1e-6`). `kind`: `boundary`, `exclusion_boundary`, `interior`. Rules: `id` unique in the layout; all numbers finite; `diameter > 0`; 2D nail inside or on the outline and outside every exclusion, within 0.01 mm; the distance between two nails is at least the sum of their radii minus 1e-6 mm, otherwise a **warning**.

### Image

| Field | Type | Rule |
|---|---|---|
| `id`, `name` | u32, string | |
| `color_mode` | `bw`, `gray`, `color` | |
| `shape` | `circle`, `landscape`, `portrait`, `square`, `free` | informational for the application |
| `width_px`, `height_px` | integer ≥ 1 | of the source image |
| `source` | `{entry, format, size}` | **full only**; format `png`, `jpeg`, `webp`; metadata already stripped |
| `target` | same, PNG | **full only**, optional (the processed image) |
| `thumbnail` | `{entry}` | both profiles; PNG, within 256 × 256 px and 64 KiB |

### Spool

Stored together in `spools.json` as a list. `{id, name, color, opacity, diameter_mm, available_length_mm?, material?, domain?}`; `color` `#rrggbb`; `opacity` in (0, 1]; `diameter_mm > 0`; `domain` `exterior`, `interior` or `both` (default `exterior`).

### Result

| Field | Type | Rule |
|---|---|---|
| `id`, `name` | u32, string | |
| `created_at` | ISO 8601 UTC | supplied by the caller |
| `frame` | frame id | exists |
| `layout` | layout id | exists **and belongs to `frame`** |
| `images` | list of `{image, view?}` | 2D frame: exactly one, no view; 3D frame: one or more, each with `view` `{position[3], target[3], up[3], projection (orthographic or perspective), size_or_fov, weight}`; every image exists |
| `spools` | list of `{spool, length_mm}` (the **spool selection**) | at least one; spool exists; no repeat; `length_mm > 0` and not above `available_length_mm` when set |
| `steps` | integer | equals the record count of `sequence.bin` (checked) |
| `levels` | boolean | `sequence.bin` header bit 0; true requires the `string_levels` feature |
| `thumbnail` | entry | both profiles |
| `settings` | entry `settings.json`, **full only** | algorithm, seed, resolution, preprocessing, engine versions (free object) |

**Weaving sequence** (`sequence.bin`): list of records `{nail, rotation (CW or CCW), type (start, segment, end), spool_index, level?}`. `nail` is an identifier of the result's layout; `spool_index` is a position in the result's spool selection; for a `segment`, a previous record of the same spool exists; a spool starts with a `start` and ends with an `end` or is left open at the end of the list. Decoding rules and the reconstruction of the machine step list are in the contract.

### Report (validation)

`Issue {severity: error|warning, code: string, entry: path or empty, item_kind: frame|layout|image|spool|result|project|none, item_id: u32 or none, message: UTF-8}`. A project is valid when it has no error. Codes are listed in [contracts/c-api-project.md](contracts/c-api-project.md).

## Relationships and dependency rules

| Edit of … | Removes the results that … | Also removes | Keeps |
|---|---|---|---|
| Frame removed | belong to the frame | its layouts and their results | other frames |
| Frame geometry changed (outline, exclusions, dimensions, surface, mesh) | use any layout of the frame | nothing else | the layouts (a nail may now be outside: validation reports it) |
| Frame renamed or background changed | none | | everything |
| Layout removed or its nails changed | use the layout | | the frame, other layouts |
| Layout renamed | none | | |
| Image removed or its pixels, shape or color mode changed | use the image | | the image slot of the result if only the thumbnail changed |
| Image renamed or thumbnail replaced | none | | |
| Spool removed, or color, opacity or diameter changed | select the spool | | |
| Spool renamed, `available_length_mm` changed, material or domain changed | none (a length now above the available one is a validation **error** the application must resolve before saving) | | |
| Result removed | none | | everything else |

**Which changes count.** The rows above apply to these fields (the editor compares the old and the new value; a change to any other field removes nothing):

| Item | Fields that remove dependent results when they change | Fields that never do | Refused (remove and add instead) |
|---|---|---|---|
| Frame | `outline`, `exclusions`, `dimensions`, `surface`, `body`, the mesh (positions, indices, normals, UVs) | `name`, `background` | `kind` |
| Layout | the nails (position, diameter, kind, 3D fields; adding or removing a nail) | `name`, a nail `label` | `frame` |
| Image | the pixels of source or target, `color_mode`, `shape`, `width_px`, `height_px` | `name`, the thumbnail | |
| Spool | `color`, `opacity`, `diameter_mm` | `name`, `available_length_mm`, `material`, `domain` | |
| Result | none | `name` | `frame`, `layout`, `images`, `spools` (a result is removed and recomputed, which gives a new identifier) |

A change to a nail `label` is part of the layout's data but removes nothing: the sequence refers to nail identifiers, which a label does not change.

Operations available to the application on bulk data: replace the JSON of an item; replace a layout's nails; replace a frame's mesh; replace an image (JSON, pixels, thumbnail); replace the thumbnail of an image or result (never has dependents); remove an item. `dependents(item, operation)` returns what the same operation would remove, so the application can ask the person first.

A removal of dependents is **performed only when the caller confirms** (`confirm_dependents = 1`); without it, the operation fails with the list of dependents (`LOOMORA_ERR_HAS_DEPENDENTS`) and changes nothing (User Story 3, scenarios 4 and 5).

## State transitions

```text
[no project] --create(project_id, title, times)--> Editing(full, empty)
Editing --add/replace/remove/rename item--> Editing        (dependents rule above)
Editing --save(path | buffer)--> Saved(full)               (atomic; modified_at from caller)
Saved(full) --open_for_use--> Reading(read-only)           (never writes)
Saved(full) --open_for_edit--> Editing
Saved(full) --export(profile=light | full, result subset)--> Saved(new file, same project_id)
Saved(light) --open_for_use--> Reading(read-only)
Saved(light) --open_for_edit--> refused (LOOMORA_ERR_READ_ONLY_PROJECT)
Saved(light) --export(light, subset)--> Saved(light)      (full impossible: data missing)
```

## Constants (`core/src/format/limits.hpp`)

| Constant | Value | Used by |
|---|---|---|
| warn nails per layout | 100,000 | FR-025 |
| warn steps per result | 5,000,000 | FR-025 |
| warn triangles per mesh | 5,000,000 | FR-025 |
| warn decompressed total | 256 MiB | FR-025 |
| warn items per kind | 1,000 | FR-025 |
| bomb ratio (after 1 MiB) | 250 : 1 | FR-026 |
| thumbnail max side / size | 256 px / 64 KiB | FR-011, FR-017 |
| nail-in-frame tolerance | 0.01 mm | FR-024 |
| nail overlap tolerance | 1e-6 mm | FR-024 |
| frame origin tolerance (bounding-box centre) | 0.01 mm (warning) | FR-015, FR-024 |
| JSON depth / entry JSON size | 64 / 1 GiB (SAX for arrays) | FR-027 |
| entry name length | 512 bytes | FR-026 |
| progress callback interval | every 4 MiB processed | R11 |

All are tunable without a format change; the writer and the reader use the same header.
