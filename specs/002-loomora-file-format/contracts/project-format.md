# Contract: The `.loomora` project file format (version 1.0)

This document is the byte-level specification of the format. It replaces §9.1 of the requirements document (which is rewritten from it) and is the reference of the writer, the reader, the validator and the independent checker. Words in capitals follow RFC 2119.

## 1. Container

- A ZIP archive (PKWARE APPNOTE 6.3), extension `.loomora`, MIME type `application/vnd.loomora+zip`.
- Compression methods: **0 (stored)** and **8 (deflate)** only. No encryption, no multi-disk archive, no archive comment, no per-entry comment.
- ZIP64 MUST be used when and only when an entry size, an offset or the entry count does not fit the classic fields (4 GiB − 1 and 65,535 entries); a reader MUST accept ZIP64 whenever present.
- The central directory is authoritative. A reader checks each local header against it (name, method, sizes, CRC-32) and rejects a mismatch. Data descriptors MAY appear in files from other writers and are accepted; this writer never writes them.
- **Entry names**: UTF-8, at most 512 bytes, relative, `/` as separator, no empty, `.` or `..` component, no leading `/`, no `\`, no `:` in the first component, no NUL, no control character; two names that are equal after ASCII case folding are rejected. Names the format defines use only `[A-Za-z0-9._-]` and `/`. A name ending with `/` (a directory entry) is accepted and ignored; the writer never writes one. An entry whose Unix mode marks a symbolic link, a device or a socket is rejected.
- **Overlap and bounds**: entries MUST NOT overlap and MUST lie before the central directory; the reader rejects a file otherwise.
- **First entry** `mimetype`: stored, no extra field, content exactly `application/vnd.loomora+zip` (no newline), at offset 0 (so the type can be sniffed at a fixed offset).

### 1.1 Writer determinism (FR-004)

For one set of options the writer MUST produce the same bytes from the same project. It sets: version made by `0x0314` (Unix, 2.0), version needed 20 (45 when ZIP64), general flags 0 (bit 11 only when the name is not ASCII), DOS time `0x0000` and date `0x0021` (1980-01-01), no extra field other than ZIP64, external attributes `0x81A40000` (regular file, 0644), the entry order of §2, and for deflate the fixed parameters of research R3 (level 6, window 15, memLevel 8, default strategy; zlib version pinned in `third_party/zlib/VENDORED.md`). Stored entries are copied byte for byte.

Default compression by entry: deflate for every `.json`; stored for `mimetype`, `*.bin`, `*.png`, `*.jpg`, `*.jpeg`, `*.webp`. The writer option `all_stored` stores everything.

## 2. Entries and their order

The writer writes entries in this order; a reader accepts any order but `mimetype` MUST be first.

```text
mimetype
manifest.json
spools.json
frames/<fid>/frame.json
frames/<fid>/mesh.bin                       3D frames with a mesh
frames/<fid>/mesh-source.<ext>              optional, full profile only
frames/<fid>/layouts/<lid>/layout.json
frames/<fid>/layouts/<lid>/nails.json
images/<iid>/image.json
images/<iid>/source.<ext>                   full profile only (ext: png, jpg, webp)
images/<iid>/target.png                     full profile only, optional
images/<iid>/thumb.png
results/<rid>/result.json
results/<rid>/sequence.bin
results/<rid>/thumb.png
results/<rid>/settings.json                 full profile only
<entries unknown to this version, in their original relative order>
```

Frames are listed in increasing `fid`, and inside a frame the layouts in increasing `lid`; images, results likewise. `<fid>`, `<lid>`, `<iid>`, `<rid>` are the decimal identifier without leading zeros. Entries the format does not define are **unknown entries**: they MUST have a name under a namespace folder (`x-<namespace>/…`) or be listed in `manifest.json`; a reader keeps them listed and copies them on rewrite (FR-020).

## 3. `manifest.json`

A JSON object (schema: [schemas/manifest.schema.json](schemas/manifest.schema.json)); keys in this order:

```json
{
  "format": { "name": "loomora", "major": 1, "minor": 0 },
  "min_reader": "1.0",
  "project_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "title": "Living room",
  "created_at": "2026-11-17T09:00:00Z",
  "modified_at": "2026-11-17T09:00:00Z",
  "generator": { "application": "Loomora", "version": "0.1.0", "platform": "windows", "architecture": "x86_64" },
  "profile": "full",
  "editable": true,
  "counts": { "frames": 1, "layouts": 2, "images": 1, "spools": 3, "results": 1 },
  "next_id": { "frame": 2, "layout": 3, "image": 2, "spool": 4, "result": 2 },
  "order": { "frames": [1], "layouts": [1, 2], "images": [1], "spools": [1, 2, 3], "results": [1] },
  "required_features": [],
  "entries": {
    "frames/1/frame.json": { "size": 812, "sha256": "…64 hex…" }
  },
  "extensions": {}
}
```

- `entries` lists every archive entry except `mimetype` and `manifest.json`, with the **uncompressed** size and the SHA-256 (lowercase hex). The reader checks both on every entry (FR-023).
- Counts, `next_id` and `order` follow [data-model.md](../data-model.md).
- `required_features` is derived from the content: `surface_mesh` when a 3D frame has a mesh, `string_levels` when a `sequence.bin` has levels, `multi_view_target` when a result uses more than one image. A reader that does not know one rejects the file naming it (FR-022).
- The manifest is written with compact JSON; the keys of every object in the order shown.

## 4. JSON entries

All JSON is UTF-8 without BOM, compact (no whitespace outside strings), numbers as in §6, object keys in the order of the schema's `properties`. A reader accepts any key order and any whitespace, ignores unknown keys (FR-020), and rejects duplicate keys, a number that is not finite and a depth over 64.

### 4.1 `spools.json`

```json
{ "spools": [ { "id": 1, "name": "Black", "color": "#000000", "opacity": 1, "diameter_mm": 0.3,
                "available_length_mm": 200000, "material": "cotton", "domain": "exterior" } ] }
```

### 4.2 `frame.json`

```json
{ "id": 1, "name": "Square 50", "kind": "2d",
  "dimensions": { "width_mm": 500, "height_mm": 500 }, "background": "#ffffff",
  "outline": { "type": "rectangle", "params": { "width_mm": 500, "height_mm": 500 },
               "polyline": [[-250,-250],[250,-250],[250,250],[-250,250]] },
  "exclusions": [], "surface": "plane" }
```

A 3D frame adds `"kind": "3d"`, `"surface"`, `"body"`, and `"mesh": { "vertices": n, "triangles": t, "watertight": true, "manifold": true, "has_normals": true, "has_uvs": false, "source": {"name": "…", "sha256": "…"}, "construction": {…}, "attached_source": "frames/2/mesh-source.stl" }` (`construction` and `attached_source` omitted in `light`). For a 3D frame `outline` is the footprint used for display and may be an empty polyline when the surface is closed.

### 4.3 `layout.json` and `nails.json`

`layout.json`: `{ "id": 1, "name": "Circle 200", "frame": 1, "nail_count": 200 }`.

`nails.json`: `{ "nails": [ {"id":0,"x":250,"y":0,"diameter":2,"kind":"boundary"}, … ] }`. A 3D nail adds `z`, `axis` `[x,y,z]`, `length`, `side`, `face` `{ "triangle": n, "u": f, "v": f }`; `label` is optional. One nail per array element, written on the same line as the array (no line break between nails).

### 4.4 `image.json`

```json
{ "id": 1, "name": "Portrait", "color_mode": "bw", "shape": "circle", "width_px": 1200, "height_px": 1200,
  "source": { "entry": "images/1/source.jpg", "format": "jpeg", "size": 345678 },
  "target": { "entry": "images/1/target.png", "format": "png", "size": 123456 },
  "thumbnail": { "entry": "images/1/thumb.png" } }
```

`source` and `target` are absent in a `light` project.

### 4.5 `result.json`

```json
{ "id": 1, "name": "Portrait on circle", "created_at": "2026-11-17T09:30:00Z",
  "frame": 1, "layout": 1,
  "images": [ { "image": 1 } ],
  "spools": [ { "spool": 1, "length_mm": 150000 }, { "spool": 2, "length_mm": 80000 } ],
  "steps": 5000, "levels": false,
  "thumbnail": { "entry": "results/1/thumb.png" },
  "settings": { "entry": "results/1/settings.json" } }
```

On a 3D frame an `images` element carries `"view": { "position": [x,y,z], "target": [x,y,z], "up": [x,y,z], "projection": "orthographic", "size": 400, "weight": 1.0 }` (`"projection": "perspective"` uses `"fov_deg"` instead of `"size"`). `settings` is absent in a `light` project.

### 4.6 `settings.json`

A free JSON object; the format recommends `algorithm`, `seed`, `resolution`, `preprocessing`, `engine_versions`. It is not interpreted.

## 5. Binary entries

All integers are little-endian; floats are IEEE 754 binary32 (little-endian byte order). Readers MUST decode byte by byte (never by casting memory).

### 5.1 `sequence.bin`

```text
"LMSQ" | version u8 (=1) | header flags u8 (bit 0: levels present) | count varint (LEB128) | records… | CRC32 u32
record: nail_id varint | flags u8 | level u8 (only if header bit 0; pass height, 0 = base)
flags:  bit 0     = rotation at this nail (0 = CW, 1 = CCW)
        bits 1–2  = type (0 segment, 1 start, 2 end, 3 reserved: rejected)
        bits 3–7  = spool index 0–30 into result.json "spools"; 31 = escape, followed by a varint index
```

- The CRC-32 (zlib polynomial) covers every preceding byte of the entry. A varint is at most 5 bytes for a `u32` value; a longer or non-minimal encoding is rejected.
- `count` MUST equal `result.steps` and the number of records; `nail_id` MUST be a nail of the result's layout; the spool index MUST be below the length of the selection.
- Semantics are those of the requirements document §9.1: records are in weaving order; `start` attaches the spool's string to the nail; `segment` goes from the nail of the spool's previous record to `nail_id`, winding direction as indicated at `nail_id`, the rotation at the starting nail being that of the spool's previous record; `end` cuts or fixes the string. The step list of the machine export (§8.3 of the requirements) is rebuilt from it without loss.
- About 3 bytes per step without levels.

### 5.2 `mesh.bin`

```text
"LMMS" | version u8 (=1) | flags u8 (bit 0: normals present, bit 1: texture coordinates present) | reserved u16 (=0)
vertex_count u64 | triangle_count u64
positions: vertex_count × 3 × f32 (mm)
indices:   triangle_count × 3 × u32        (each < vertex_count)
normals:   vertex_count × 3 × f32          (only if bit 0; unit length within 1e-3)
uvs:       vertex_count × 2 × f32          (only if bit 1; finite; full profile only)
block table: for each 1 MiB block of the data above: CRC32 u32 ; then table CRC32 u32
```

The counts MUST match `frame.json` (`vertices`, `triangles`), and the flags MUST match its `has_normals` and `has_uvs`. In a `light` file the entry is **rewritten without the `uvs` section**: bit 1 is clear and the block table is recomputed, so the entry's SHA-256 differs from the full file's while positions, indices and normals are byte-for-byte the same values. A reader verifies a block's CRC-32 whenever it reads from it (range reads) and the SHA-256 of the whole entry during validation.

## 6. Numbers

- Integers are written as decimal integers without sign for unsigned values; values above 2^53 are not used (identifiers are 32-bit).
- A floating-point value is written as a decimal that parses back to exactly the same double, produced by the project's fixed algorithm (research R4, R5: RapidJSON's Grisu2 conversion and its text layout, at the version pinned in `third_party/rapidjson/VENDORED.md`); `-0` is normalized to `0` before writing. The exact text for a given double is therefore a property of the pinned library, and the reference hashes change only when it is updated.
- A reader parses decimals with full precision and rejects `NaN`, `Infinity` and out-of-range values.

## 7. Profiles (FR-017)

| Content | `full` | `light` |
|---|---|---|
| frames, layouts, nails, spools, results, sequences | yes | yes (selected results only) |
| 3D mesh geometry (`mesh.bin`: positions, indices, normals) | yes | yes, **identical values** (the entry is rewritten without UVs, so its SHA-256 differs) |
| mesh texture coordinates (UV, optional) | yes, if the frame has them | no |
| `mesh-source.<ext>`, mesh `construction` | yes (both optional) | no |
| `images/*/source.*`, `target.png` | yes | no |
| `images/*/thumb.png`, `results/*/thumb.png` | yes | yes |
| `results/*/settings.json` | yes | no |
| `editable` | true | false |

An export with a result subset (FR-018) writes the chosen results and exactly the frames, layouts, images and spools they use; `project_id` is kept and counts, `next_id` and `order` describe the file.

## 8. Reading rules (summary of validation)

In this order (research R11): container checks (§1), manifest parse, version and features, per-entry size and SHA-256 and ZIP CRC-32, JSON schema checks, identifiers and references, numeric checks, nails against frames (and the frame origin, a warning), `sequence.bin` and `mesh.bin` decoding, thresholds, time stamp (`warn.timestamp`) and nail distance warnings. The first error does not stop the pass when it is safe to continue; every issue is reported (CLI) and the project is valid only if there is no error.

## 9. Versioning (FR-020, FR-021)

- A **minor** change only adds optional fields, optional entries or new values of open enumerations; readers ignore what they do not know.
- A **major** change is any other change; `min_reader` says which reader version can open the file; a reader of a lower major rejects it with `This file needs Loomora format version M.m or newer`.
- A reader of major N reads every major up to N.
