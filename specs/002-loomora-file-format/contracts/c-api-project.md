# Contract: C API of the project format (`loomora_project.h`)

An addition to the C API of spec 001 ([../../001-loomora-cross-platform/contracts/c-api.md](../../001-loomora-cross-platform/contracts/c-api.md)); the same rules apply: valid as C89 and C++98, `extern "C"`, `LOOMORA_API`, no C++ type or exception across the boundary, UTF-8 strings, caller-supplied buffers, negative error codes. The header `loomora_project.h` is included by `loomora.h`. `LOOMORA_ABI_VERSION` **stays 1** for now (owner decision 2026-11-17): the new functions are additive, no existing function or constant changes, and a caller that needs the project API checks `loomora_format_version` (it links only against a library that has it).

**Extension of ADR 0001 rule 3** (ADR 0023): the library also creates and destroys **opaque handles** (`loomora_project`, `loomora_editor`, `loomora_report`). A handle exposes no memory. Every byte of data still crosses in a buffer owned by the caller.

## Conventions

- **Size queries.** Functions that return variable-size data return `int64_t`: the **total length needed** (≥ 0), or a negative error. They write only when `cap` is at least that length (nothing is written otherwise), so `cap = 0, buf = NULL` asks for the size. No NUL is written.
- **Ranges.** Bulk data is read as `first` (index of the first element) and `count` into an array of `count` elements; the function returns the number of elements written (it can be less than `count` at the end of the data) or an error.
- **Identifiers** are `uint32_t`; kinds: `LOOMORA_KIND_FRAME 1`, `LAYOUT 2`, `IMAGE 3`, `SPOOL 4`, `RESULT 5`.
- **Thread safety.** A handle may be used from one thread at a time; two handles are independent. Read-only `loomora_project` handles may be shared by several threads for read calls.
- **Time** values are UTF-8 `YYYY-MM-DDTHH:MM:SSZ` strings supplied by the caller; the library reads no clock.
- **Progress callback**: `typedef int32_t (*loomora_progress_fn)(void *user, uint64_t done, uint64_t total);` called about every 4 MiB processed; a non-zero return cancels the operation with `LOOMORA_ERR_CANCELLED` and, for a save, leaves the previous file intact.

## Error codes

Existing: `-1` TOO_LONG, `-2` CAPACITY, `-3` INVALID (null pointer, bad argument), `-4` INTERNAL. New (`LOOMORA_ERR_…`):

| Code | Name | Meaning |
|---|---|---|
| −10 | `IO` | read, write, flush, rename, disk full |
| −11 | `NOT_A_PROJECT` | not a ZIP archive, or `mimetype` missing or wrong |
| −12 | `CORRUPT` | container, checksum, schema or reference error (the report lists each) |
| −13 | `UNSUPPORTED_VERSION` | major version above the library's; message carries `min_reader` |
| −14 | `UNSUPPORTED_FEATURE` | a required feature the library does not handle |
| −15 | `BOMB` | decompression ratio above the guard |
| −16 | `NO_MEMORY` | allocation failed |
| −17 | `READ_ONLY_PROJECT` | edit of a `light` project |
| −18 | `NOT_FOUND` | unknown kind, identifier or entry |
| −19 | `CANCELLED` | the progress callback asked to stop |
| −20 | `HAS_DEPENDENTS` | the edit needs confirmation: dependents are listed, nothing changed |
| −21 | `INVALID_ITEM` | the item passed to an edit does not satisfy the data model |
| −22 | `TOO_LARGE_FOR_PLATFORM` | an offset the platform cannot address (a declared fallback) |

## Library

```c
uint32_t loomora_format_version(int32_t *major, int32_t *minor);   /* 1, 0; returns 1 */
uint64_t loomora_format_limit(int32_t which);                      /* warning thresholds, see data-model */
int32_t  loomora_uuid4_from_random(const uint8_t random16[16], char out[37]); /* NUL-terminated */
```

## Reading (use mode)

```c
typedef struct loomora_project loomora_project;
typedef struct loomora_report  loomora_report;

/* Opens, validates (streaming) and returns the handle. Never writes, never creates a file. On success
   *report (if not NULL) holds the warnings; on failure *project is NULL and *report holds the errors
   (when the failure happened after the container could be read). The memory variant does not copy:
   the buffer must stay valid until loomora_project_close. */
int32_t loomora_project_open_path(const char *path, loomora_progress_fn cb, void *user,
                                  loomora_project **project, loomora_report **report);
int32_t loomora_project_open_memory(const uint8_t *data, size_t len, loomora_progress_fn cb,
                                    void *user, loomora_project **project, loomora_report **report);
void    loomora_project_close(loomora_project *project);

/* Validation only (no handle kept); the same report. */
int32_t loomora_validate_path(const char *path, loomora_progress_fn cb, void *user,
                              loomora_report **report);
int32_t loomora_validate_memory(const uint8_t *data, size_t len, loomora_progress_fn cb,
                                void *user, loomora_report **report);

int64_t loomora_project_manifest(const loomora_project *p, uint8_t *buf, size_t cap); /* manifest.json */
int32_t loomora_project_is_light(const loomora_project *p);          /* 1 = light, 0 = full */
int64_t loomora_project_count(const loomora_project *p, int32_t kind);
int64_t loomora_project_ids(const loomora_project *p, int32_t kind, uint32_t *ids, size_t cap); /* display order; returns the number of ids, writes them only if cap >= that number */
int64_t loomora_project_item_json(const loomora_project *p, int32_t kind, uint32_t id,
                                  uint8_t *buf, size_t cap);          /* frame.json, layout.json, image.json, result.json; spool: its object from spools.json */
int64_t loomora_project_entry_size(const loomora_project *p, const char *entry);          /* uncompressed size */
int64_t loomora_project_read_entry(const loomora_project *p, const char *entry,
                                   uint64_t offset, uint8_t *buf, size_t cap);            /* range of any entry: thumbnails, source images, settings.json, mesh-source */
```

Bulk data, fixed layouts (sizes in bytes, no hidden padding; the header gives the offsets):

```c
typedef struct {            /* 88 bytes, 8-byte aligned */
  uint32_t id;  uint32_t kind;        /* kind: 1 boundary, 2 exclusion_boundary, 3 interior */
  uint32_t flags;                     /* bit 0: 3D fields valid, bit 1: has label */
  uint32_t side;                      /* 3D: 0 exterior, 1 interior */
  double   x, y, z;
  double   diameter, length;
  float    axis[3];  float face_u;
  float    face_v;   uint32_t face_triangle;   uint32_t reserved0, reserved1;
} loomora_nail;
typedef struct { uint32_t nail; uint32_t spool_index; uint8_t rotation; /* 0 CW, 1 CCW */
                 uint8_t type; /* 0 segment, 1 start, 2 end */ uint8_t level; uint8_t reserved; } loomora_step; /* 12 bytes */
#define LOOMORA_MESH_NORMALS 1   /* flags of loomora_mesh_info */
#define LOOMORA_MESH_UVS     2
typedef struct { uint64_t vertices; uint64_t triangles; uint32_t flags; uint32_t reserved; } loomora_mesh_info; /* 24 bytes */

int64_t loomora_layout_nail_count(const loomora_project *p, uint32_t layout);
int64_t loomora_layout_read_nails(const loomora_project *p, uint32_t layout, uint64_t first,
                                  size_t count, loomora_nail *out);
int64_t loomora_layout_nail_label(const loomora_project *p, uint32_t layout, uint32_t nail_id,
                                  uint8_t *buf, size_t cap);
int64_t loomora_result_step_count(const loomora_project *p, uint32_t result);
int64_t loomora_result_read_steps(const loomora_project *p, uint32_t result, uint64_t first,
                                  size_t count, loomora_step *out);
int32_t loomora_frame_mesh_info(const loomora_project *p, uint32_t frame, loomora_mesh_info *info);
int64_t loomora_frame_read_vertices(const loomora_project *p, uint32_t frame, uint64_t first, size_t count, float *xyz);  /* 3 floats each */
int64_t loomora_frame_read_triangles(const loomora_project *p, uint32_t frame, uint64_t first, size_t count, uint32_t *abc);
int64_t loomora_frame_read_normals(const loomora_project *p, uint32_t frame, uint64_t first, size_t count, float *xyz);
int64_t loomora_frame_read_uvs(const loomora_project *p, uint32_t frame, uint64_t first, size_t count, float *uv);  /* 2 floats each; 0 written when the frame has none */
```

Machine export steps (§8.3 of the requirements) are rebuilt from `loomora_step` by the caller's export feature; the format guarantees the rebuild is lossless.

## Report

```c
int64_t loomora_report_count(const loomora_report *r);
int64_t loomora_report_error_count(const loomora_report *r);
/* severity: 1 warning, 2 error; item_kind: 0 none/project, or LOOMORA_KIND_*; item_id 0 when none */
int32_t loomora_report_issue(const loomora_report *r, size_t index, int32_t *severity,
                             int32_t *item_kind, uint32_t *item_id);
/* field: 1 code (e.g. "ref.layout_of_other_frame"), 2 entry path, 3 message (UTF-8, English) */
int64_t loomora_report_text(const loomora_report *r, size_t index, int32_t field,
                            uint8_t *buf, size_t cap);
void    loomora_report_free(loomora_report *r);
```

Issue codes (stable, the CLI prints them; messages are for people): `zip.*` (container), `entry.size`, `entry.sha256`, `entry.crc`, `entry.missing`, `entry.unlisted`, `entry.bomb`, `manifest.*`, `version.major`, `feature.unsupported`, `schema.*`, `id.duplicate`, `ref.missing`, `ref.layout_of_other_frame`, `ref.spool_not_selected`, `num.not_finite`, `nail.outside_frame`, `nail.duplicate_id`, `seq.header`, `seq.crc`, `seq.count`, `mesh.crc`, `mesh.index_range`, `spool.length_over_available` (errors); `warn.count` (with the element and its count), `warn.nail_distance`, `warn.origin_offset`, `warn.thumbnail_size`, `warn.timestamp` (a manifest `created_at` / `modified_at` or a result's `created_at` that has the right shape but is not a real date and time, such as 2026-02-31 or second 60: one warning per field, naming the entry; the file still opens), `warn.light_project`, `warn.unknown_entry`, `warn.leftover_file` (a save that replaced the file but could not delete the previous copy it moved aside, which only happens on Windows: no entry, the message names the leftover `<target>.old-xxxxxxxx` file, which can be deleted by hand; returned in the report of the save or export, never by validation) (warnings). Each issue carries the entry and the item when they apply.

## Editing (edit mode)

```c
typedef struct loomora_editor loomora_editor;

int32_t loomora_editor_create(const uint8_t project_id_random16[16], const char *title,
                              const char *created_at, loomora_editor **editor);
int32_t loomora_editor_open_path(const char *path, loomora_progress_fn cb, void *user,
                                 loomora_editor **editor, loomora_report **report);   /* LOOMORA_ERR_READ_ONLY_PROJECT for light */
void    loomora_editor_close(loomora_editor *e);

/* Items are passed as UTF-8 JSON following the schemas; the id is allocated by the editor and
   returned (the "id" field of the JSON, if present, is ignored). */
int32_t loomora_editor_add_spool(loomora_editor *e, const char *json, uint32_t *id);
int32_t loomora_editor_add_frame(loomora_editor *e, const char *json, uint32_t *id);
int32_t loomora_editor_frame_set_mesh(loomora_editor *e, uint32_t frame, const loomora_mesh_info *info,
                                      const float *xyz, const uint32_t *abc,
                                      const float *normals_or_null, const float *uvs_or_null);
                                      /* first mesh of a frame that has none; use replace_frame_mesh afterwards */
int32_t loomora_editor_add_layout(loomora_editor *e, uint32_t frame, const char *json,
                                  const loomora_nail *nails, size_t count, uint32_t *id);
/* Optional extras of an existing item; they change nothing a result uses, so they have no
   dependents. replace_layout_nails keeps the labels of the nails whose identifier remains. */
int32_t loomora_editor_layout_set_label(loomora_editor *e, uint32_t layout, uint32_t nail_id,
                                        const char *label_or_null);   /* NULL removes the label */
int32_t loomora_editor_frame_set_attached_source(loomora_editor *e, uint32_t frame,
                                                 const char *name, const uint8_t *data,
                                                 size_t len);   /* mesh-source copy, full only; NULL data removes it */
int32_t loomora_editor_add_image(loomora_editor *e, const char *json,
                                 const uint8_t *source, size_t source_len,   /* metadata is stripped */
                                 const uint8_t *target_png, size_t target_len,
                                 const uint8_t *thumb_png, size_t thumb_len, uint32_t *id);
int32_t loomora_editor_add_result(loomora_editor *e, const char *json, const loomora_step *steps,
                                  size_t step_count, const uint8_t *thumb_png, size_t thumb_len,
                                  const char *settings_json_or_null, uint32_t *id);

/* Replace = same identifier, new content; applies the dependency rules of the data model.
   replace_item changes the JSON part of an item (the fields of its schema); the bulk data of an
   item (nails, mesh, image pixels) is replaced by the three functions after it. */
int32_t loomora_editor_replace_item(loomora_editor *e, int32_t kind, uint32_t id, const char *json,
                                    int32_t confirm_dependents);
int32_t loomora_editor_replace_layout_nails(loomora_editor *e, uint32_t layout,
                                            const loomora_nail *nails, size_t count,
                                            int32_t confirm_dependents);
int32_t loomora_editor_replace_frame_mesh(loomora_editor *e, uint32_t frame,
                                          const loomora_mesh_info *info, const float *xyz,
                                          const uint32_t *abc, const float *normals_or_null,
                                          const float *uvs_or_null, int32_t confirm_dependents);
int32_t loomora_editor_replace_image(loomora_editor *e, uint32_t image, const char *json,
                                     const uint8_t *source, size_t source_len,
                                     const uint8_t *target_png, size_t target_len,
                                     const uint8_t *thumb_png, size_t thumb_len,
                                     int32_t confirm_dependents);
/* Thumbnail of an image or of a result: never has dependents. */
int32_t loomora_editor_replace_thumbnail(loomora_editor *e, int32_t kind, uint32_t id,
                                         const uint8_t *png, size_t len);
int32_t loomora_editor_remove_item(loomora_editor *e, int32_t kind, uint32_t id,
                                   int32_t confirm_dependents);
/* Lists what an operation on an item would remove: ids written as pairs kind, id (layouts and
   results). operation: LOOMORA_OP_REMOVE 1 (removal of the item), LOOMORA_OP_CHANGE 2 (a change
   of the item that the data model says affects results: the same list the matching replace_*
   function would remove). Returns the number of pairs, or the number needed. */
#define LOOMORA_OP_REMOVE 1
#define LOOMORA_OP_CHANGE 2
int64_t loomora_editor_dependents(const loomora_editor *e, int32_t kind, uint32_t id,
                                  int32_t operation, uint32_t *pairs, size_t cap_pairs);
int32_t loomora_editor_set_order(loomora_editor *e, int32_t kind, const uint32_t *ids, size_t n);
int32_t loomora_editor_set_title(loomora_editor *e, const char *title);

/* Validates the staged project, writes a new archive, replaces atomically. modified_at is the
   caller's time. options: bit 0 = store every entry uncompressed. */
int32_t loomora_editor_save_path(loomora_editor *e, const char *path, const char *modified_at,
                                 uint32_t options, loomora_progress_fn cb, void *user,
                                 loomora_report **report);
typedef int32_t (*loomora_sink_fn)(void *user, const uint8_t *data, size_t len);
int32_t loomora_editor_save_stream(loomora_editor *e, const char *modified_at, uint32_t options,
                                   loomora_sink_fn sink, void *user, loomora_report **report);
```

**Saving to a symbolic link** (FR-028): when `path` of `loomora_editor_save_path` or `loomora_project_export_path` is a symbolic link (or a chain of links), the save follows it: the file the link finally points to is replaced atomically, the temporary file is made in the directory of that file, and the link stays a link. A failed save leaves the old file intact, as for any path. On POSIX systems a link to a file that does not exist yet creates that file, and a loop of links, or a link that cannot be read, returns `LOOMORA_ERR_IO` and changes nothing. On Windows the link is followed with `GetFinalPathNameByHandleW` when the target exists, and by reading the link text (`FSCTL_GET_REPARSE_POINT`, relative or absolute, at most 40 links in a chain) when it is missing, so that a link to a missing file creates that file as on POSIX; both are found at run time, so only when the system has them (Windows Vista and later; Windows 2000 and XP have no symbolic links). A junction (mount point), a loop of links, a chain of more than 40 links or a link that cannot be read is not followed: the path is replaced by the new file, as any other path.

Every `replace_*` and `remove_item` call with `confirm_dependents = 0` returns `LOOMORA_ERR_HAS_DEPENDENTS` when results or layouts would be removed, changes nothing, and `loomora_editor_dependents` (with the matching operation) lists them. With `confirm_dependents = 1` the item is changed and the dependents are removed in the same call.

**Which changes count** (the field table of the data model decides; summarized): a change to a field that affects results (frame outline, exclusions, dimensions, surface, body or mesh; layout nails; image pixels, color mode, shape or dimensions; spool color, opacity or diameter) removes the dependent results; a change to any other field (`name`, `background`, `available_length_mm`, `material`, `domain`, thumbnails) removes nothing. `replace_item` refuses with `LOOMORA_ERR_INVALID_ITEM` a change of the `frame` of a layout, of the `frame`, `layout`, `images` or `spools` of a result, or of the `kind` of a frame: those items are removed and added again instead. The only change allowed on a result is its `name`; a recomputed result is a new result with a new identifier.

## Export (any profile, any result subset)

```c
#define LOOMORA_PROFILE_FULL  1
#define LOOMORA_PROFILE_LIGHT 2
int32_t loomora_project_export_path(const loomora_project *p, const char *path, int32_t profile,
                                    const uint32_t *result_ids, size_t n_results, /* 0 = all */
                                    const char *modified_at, uint32_t options,
                                    loomora_progress_fn cb, void *user, loomora_report **report);
int32_t loomora_project_export_stream(const loomora_project *p, int32_t profile,
                                      const uint32_t *result_ids, size_t n_results,
                                      const char *modified_at, uint32_t options,
                                      loomora_sink_fn sink, void *user, loomora_report **report);
```

A `full` export from a `light` project returns `LOOMORA_ERR_READ_ONLY_PROJECT`. The written file is atomically replaced like a save.

## Export lists

`loomora.exp`, `loomora.map`, `bridge/jni` and `bridge/wasm/exports*.txt` list every function above; `tools/check_std.py` fails when the shared build exports a symbol that is not `loomora_*` or misses one of the header.
