# Feature Specification: The `.loomora` File Format

**Feature Branch**: `feat/loomora-format`

**Created**: 2026-11-17

**Status**: Draft

**Input**: User description (translated from French): "I want to implement the .loomora format. Look at the specs in docs/requirements-specification.md."

> The format is first described in [docs/requirements-specification.md](../../docs/requirements-specification.md), §9.1 (decision 16.5). This feature builds the ability to **write, read, validate and exchange** it, and **changes §9.1 on four points** decided on 2026-11-17 (see Clarifications): a project holds several frames, nail layouts, images and results; the file holds no weaving progress; the number of elements is not limited; 2D and 3D frames live side by side. §9.1 keeps the byte-level details that are not contradicted (field names, `sequence.bin` layout); where it disagrees with this document, this document wins and §9.1 is updated (Assumptions).

## Definitions

- **Project file**: a `.loomora` file. It is a **project**: one self-describing archive that holds everything the person created. It is changed **only by the application's edit mode**, on an explicit save; it holds **no weaving progress**, and use mode never changes it.
- **Edit mode**: the application mode in which the person builds a project file: defines frames, defines nail layouts (nail positions), imports images, and runs the computations that produce the string paths (results). It reads and writes project files.
- **Use mode**: the application mode in which the person only selects a result (a string path) of a project and follows the weaving progress. It only **reads** project files; the progress it records is application data.
- **Frame** (French "cadre"): the physical support the nails go in, with its outline, exclusion zones, dimensions and background. A frame is **2D** (flat) or **3D** (surface, mesh or primitive). Its nails are held by its nail layouts, not by the frame itself. A project holds **zero or more** frames, of either kind.
- **Nail layout** (French "disposition de clous"): one arrangement of nails on a frame, for example a circle of nails or a heart of nails on the same 50 cm × 50 cm 2D frame. A layout belongs to **one frame**; a frame holds **zero or more** layouts, which may differ in number and positions of nails.
- **Image**: a picture the person brought or prepared (source image, processed target image, thumbnail). It has its own color mode (for example black and white, or color) and its own shape (for example circle, landscape). A project holds **zero or more** images.
- **Spool** (French "bobine"): a reel of string, defined **once per project** with its color, opacity, diameter and optional material, domain and available length. A project holds **zero or more** spools.
- **Result**: one outcome of the path computation (the reconstruction of a picture with strings). It is the combination of **one frame and one of its nail layouts**, and points to the image(s) it reproduces (a 3D frame uses several images, one per view); it carries the **selection of project spools it uses, with the length chosen for each**, its weaving sequence and the parameters used. A project holds **zero or more** results. A combination of frame, layout and image that was never computed simply has no result; this is normal, because the application lets the person compute the missing ones.
- **Entry**: one named item inside the archive.
- **Writer / Reader**: the shared backend capability that produces / consumes project files; it exists once, in the shared backend, and every UI uses it (no logic in a UI, as in spec 001).
- **Profile**: `full` (everything, editable) or `light` (for use mode, small terminals and Bluetooth): without the source and processed images, without the computation parameters and without what only editing needs, but with a small thumbnail per image and per result and with everything needed to show and follow a result, including the 3D mesh geometry. A `light` project is **not editable**.
- **Required feature**: a capability a reader must have to open a file correctly (for example `surface_mesh`, `string_levels`). A file listing one the reader lacks is rejected.
- **Weaving sequence**: the ordered list of nail crossings (nail, winding direction, spool, optionally pass height) of one result. It is **stored, never recomputed** when a file is opened.
- **Application data**: data the application keeps for the person, outside the project file, such as the weaving progress (position, string consumed, bookmarks). It refers to a project and to a result by their stable identifiers.
- **Warning threshold**: a size or count above which the application warns the person on opening and on saving. It never blocks.
- **Validation**: the checks a reader runs on every file it opens.

## Clarifications

### Session 2026-11-17

- Q: How far does this feature go on 3D supports (mesh, views, string levels, 3D nails), which §3.6 marks P2? → A: **The whole format, 3D included.** The writer, the reader and the validation cover every kind of frame, flat or 3D. The 3D features are advertised through required features so that a reader without a capability rejects the file rather than misreading it.
- Q: Are there limits on the number of elements (nails, steps, triangles, ...)? → A: **No limit.** There is only a **warning, shown on opening and on saving**, above thresholds. This replaces the hard limits of §9.1 (100,000 nails, 5,000,000 steps, 5,000,000 triangles), which become warning thresholds.
- Q: Must the format support 2D frames? → A: **Yes, 2D and 3D frames**, in the same project, together.
- Q: What is a `.loomora` file? → A: **A project file.** It can hold several images (for example one black and white and one in color, one circle and one landscape), several frames (2D and 3D) and several results of the path computation (2D and 3D).
- Q: Must a file that decompresses into a huge amount of data (a "decompression bomb") be refused, or only warned about? → A: **No fixed size ceiling.** The file is refused only when the decompression ratio is abnormal (a tiny archive expanding enormously) or when an allocation or a write fails (a defined error, nothing damaged); otherwise a large file only raises the warnings of the previous answer. This replaces the 256 MB decompressed limit of §9.1.
- Q: Is the project file something the application never changes? → A: **No, it is changed by the application's edit mode.** The application has two modes: **edit mode** (define frames, define nail positions, import images, run the computations that give the string paths) reads and writes project files; **use mode** (select a string path and follow the progress) only reads them. The earlier wording "never changed by the application" was wrong; the real rule is that the file never holds, and use mode never changes, anything related to the weaving progress.
- Q: When a frame, a nail layout or an image is modified in edit mode, what becomes of the results computed from it? → A: **They are deleted** (the application asks the person to confirm first). A saved project therefore never holds a result that no longer matches its frame, layout and images; no "stale result" state or content fingerprint is needed. A recomputed result is a new result with a new identifier, and identifiers of deleted items are never reused in the project, so application data (progress) of a deleted result can never attach itself to another one.
- Q: Are spools defined per result or once for the project? → A: **Once per project.** The project holds the spools (color, opacity, diameter, optional material, domain and available length). To compute a result, the person **selects which spools to use and the length to use of each**; the result stores that selection and its sequence. Removing a spool, or changing a property that the computation used (color, opacity, diameter), removes the results that use it, like any other item (previous answer).
- Q: Must the `light` profile keep thumbnails for use mode? → A: **Yes.** `light` keeps a small thumbnail (maximum size set by a project constant) of each image and of each result, so the person recognizes the right string path in use mode; it drops the source images, the processed target images and the computation parameters.
- Q: Can a `light` export contain only some of the results, and can a `light` project be edited? → A: A `light` export contains **only a chosen selection of results** (with the frames, layouts, images and spools they need). A `light` project is **not editable**: edit mode cannot modify and re-save it, because the data editing needs is missing.
- Q: Does `light` keep the 3D mesh, which can be large? → A: **Yes, unmodified.** The shape of a string drawn from nail to nail on a curved frame (for example around a cylinder) depends on the mesh, so the `light` profile keeps the mesh geometry exactly (positions and triangle indices, never simplified) and drops only what use mode does not need: the attached copy of the file the person imported the mesh from (STL, OBJ, ...; the mesh of the frame is already extracted from it and stays), the parametric construction (editing only) and the texture coordinates.
- Q: Does the file hold the weaving progress? → A: **No.** The file holds only what the person made and the application does not change by itself. The weaving progress is **application data**, outside the project file. This removes `progress.json` and the progress merge rules of §9.1.
- Q: Can several results of one frame use different nail layouts? → A: **Yes.** Nails are held by **nail layouts**, separate items that belong to a frame (one 50 cm × 50 cm 2D frame can have a circle layout and a heart layout). A result is the combination of a frame and one of its layouts. Not every combination needs to be computed: the person can compute the missing ones in the application.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Save a project and open it again without loss (Priority: P1)

A person who has a project saves it to a `.loomora` file and, later or on another device, opens it. Every frame, nail layout, image, result (with its spools and weaving sequence) and setting comes back exactly as saved. No path is recomputed.

**Why this priority**: this is the base of everything else (sharing, the workshop player). Without a reliable save and open there is no product.

**Independent Test**: build a reference project in memory, write it, read it back, compare field by field and compare each weaving sequence step by step. Needs no network, no UI and no other feature.

**Acceptance Scenarios**:

1. **Given** a project with one 2D frame, one layout, one image and one result, **When** it is saved then opened, **Then** the reopened project is identical to the saved one, with no recomputation.
2. **Given** a project saved with the `full` profile, **When** it is opened, **Then** the source image, the processed target image and the thumbnail of every image are available.
3. **Given** a result using more than 31 spools, **When** it is saved and reopened, **Then** each step still refers to the right spool.
4. **Given** the same project content, **When** it is saved twice on one platform, **Then** the two files are byte-identical; **When** it is saved on two different platforms or architectures, **Then** the two files have the same entries in the same order and the same decompressed content (their compressed bytes may differ).
5. **Given** a saving interrupted by a crash or a power cut, **When** the person looks for their project, **Then** the previous file is intact (a partial file never replaces a good one).

---

### User Story 2 - Use a project in use mode without changing it (Priority: P1)

A person opens a project in use mode, picks one result (a string path) and follows their weaving. The project file is only read: it stays exactly as it was, and what they do while weaving is stored elsewhere as application data.

**Why this priority**: use mode is how the project is used in the workshop, often on a small or old device; a project that is never modified by use cannot be corrupted, and can be shared read-only (removable media, transfer).

**Independent Test**: open a reference file read-only (file marked read-only), list its results, load one, then check the file bytes and modification time are unchanged and no other file appeared beside it.

**Acceptance Scenarios**:

1. **Given** a project file on a read-only location, **When** it is opened for use, **Then** it opens and every result can be read.
2. **Given** a project opened for use, **When** a result is selected and read, **Then** the file bytes and modification time are unchanged afterwards.
3. **Given** a project opened for use while edit mode saves a new version, **When** the reader reads it, **Then** it sees the complete old or the complete new file, never a mix.

---

### User Story 3 - Keep several frames, layouts, images and results in one project (Priority: P1)

A person keeps in one project: a black-and-white circular image and a color landscape image; a 50 cm × 50 cm 2D frame with a circle layout and a heart layout, a landscape 2D frame, and a 3D frame; and several results (for example the circle image on the circle layout, the landscape image on the landscape frame). They add, remove and reorder these items and reopen the project with each result still tied to the right frame, layout and image. Some combinations (for example the heart layout) have no result yet.

**Why this priority**: the project file is a whole project, not one computation. Without this the format cannot hold the person's real work, and changing the model later would break every file already written.

**Independent Test**: write a reference project with 3 frames (two 2D, one 3D), 5 layouts, 4 images (mixed color mode and shape) and 5 results referring to them; read it back; check every reference; remove one frame and check what is reported about its layouts and results.

**Acceptance Scenarios**:

1. **Given** a 50 cm × 50 cm 2D frame with a circle layout and a heart layout and a result for the circle layout only, **When** the project is saved and opened, **Then** both layouts are present, the result is tied to the circle layout, and the missing heart combination is not reported as an error.
2. **Given** two results on the same frame and the same layout with different images, **When** the project is opened, **Then** both are available, each with its own spools, weaving sequence and parameters.
3. **Given** a result that refers to a frame, a layout or an image missing from the file, or a layout that does not belong to the frame of the result, **When** the file is validated, **Then** the broken reference is reported with the result and the missing item named.
4. **Given** a frame with layouts and results, **When** the person asks to remove the frame (or a layout), **Then** the format layer reports which layouts and results depend on it, so the application can ask before it removes them; once confirmed, they are all removed together.
5. **Given** a result that depends on a frame, a layout or an image, **When** that item is modified (nail moved, outline changed, image replaced) and the person confirms, **Then** the dependent results are removed from the project, and none remains in the saved file.
6. **Given** a result removed and a new one computed, **When** the project is saved, **Then** the new result has an identifier never used before in this project.
7. **Given** a project with no result (only a frame, a layout and an image), or with only images, **When** it is saved and opened, **Then** it is valid.
8. **Given** a result computed for a 3D frame from several views, **When** it is opened, **Then** each view is tied to its own image.

---

### User Story 4 - Open files from any machine, including old and big-endian ones (Priority: P1)

A file produced on a powerful 64-bit machine is opened on a 15-year-old 32-bit machine, and the reverse; the same holds on big-endian machines. The result is the same everywhere.

**Why this priority**: the format exists so that a path computed on a strong machine can be woven from a weak one (§9). Cross-architecture identity is the reason the format fixes the byte order.

**Independent Test**: write the reference files on one platform, read them on every other platform of the validation (including a 32-bit one and, when available, a big-endian one), and compare with the expected content.

**Acceptance Scenarios**:

1. **Given** reference files written on each architecture, **When** they are read on every other architecture, **Then** the content read is identical to the expected content.
2. **Given** a file written by a newer minor version (extra fields, extra entries), **When** an older reader opens it, **Then** it opens, ignores what it does not know and, when it rewrites the file, keeps the unknown entries when possible.
3. **Given** a file whose major version is newer than the reader, **When** it is opened, **Then** it is rejected with an explicit message that states the minimum reader version needed.
4. **Given** a file of any previous major version, **When** a recent reader opens it, **Then** it opens.

---

### User Story 5 - Refuse damaged, hostile or unsupported files safely, warn on very large ones (Priority: P1)

A person opens a file that is truncated, altered, crafted to harm, or that needs a capability this version lacks. The application never crashes, never writes outside its own area, and says precisely what is wrong. A file that is merely very large is not refused: the person is warned, on opening and on saving, and continues.

**Why this priority**: files are exchanged by network, Bluetooth and sharing, so any of them may be corrupt or hostile.

**Independent Test**: run a corpus of invalid and hostile files (one per rule of the validation) and check each is rejected with the expected reason and no crash; run a corpus of very large files and check each opens with the expected warning; run the reader under fuzzing for a fixed budget with no crash.

**Acceptance Scenarios**:

1. **Given** an entry whose content no longer matches its recorded checksum, **When** the file is opened, **Then** it is rejected and the error names the entry.
2. **Given** an archive containing an absolute path, a `..` path or a symbolic link, **When** it is opened, **Then** it is rejected and nothing is written outside the destination.
3. **Given** a "decompression bomb" (a tiny archive that expands at an abnormal ratio), **When** it is opened, **Then** it is rejected before it is expanded in memory or on disk (reading only streams and writes nothing); **Given** a legitimately large file with a normal ratio, **Then** it opens with a warning only.
4. **Given** a file with more elements than a warning threshold (nails, steps, triangles), **When** it is opened, **Then** it opens and the person is warned with the element and its count; **When** it is saved, **Then** the same warning is given.
5. **Given** a file that lists a required feature the reader does not handle, **When** it is opened, **Then** it is rejected with that feature named.
6. **Given** nails with a duplicate identifier in a layout, a reference to a missing nail, spool, frame, layout or image, a number that is not finite, or a nail outside its frame, **When** the file is validated, **Then** the problem is reported; a nail-to-nail distance below the minimum is reported as a warning, not a rejection.
7. **Given** a file that contains executable content, **When** it is opened, **Then** it is never executed and is ignored or rejected as documented.
8. **Given** a machine that cannot hold a very large file in memory, **When** it is opened, **Then** the failure is a defined error, never a crash.

---

### User Story 6 - Validate a file from the command line (Priority: P2)

A technician, a test script or a server runs `loomora validate <file>` and gets a clear pass or fail, the list of problems and warnings, and an exit status usable in scripts.

**Why this priority**: it makes the format checkable without any UI and is the tool of the regression suite; the UIs work without it.

**Independent Test**: run the command on the valid corpus (exit 0) and on the invalid corpus (non-zero, with the expected message for each).

**Acceptance Scenarios**:

1. **Given** a valid file, **When** `loomora validate` runs, **Then** it reports success and exits with status 0.
2. **Given** an invalid file, **When** it runs, **Then** it lists each problem with the entry and the item (frame, layout, image, result) concerned and exits with a non-zero status.
3. **Given** a file with warnings only (nails closer than the minimum, an element count above a threshold), **When** it runs, **Then** it reports the warnings and still exits with success.

---

### User Story 7 - Share a lightweight copy of a project (Priority: P3)

A person exports a `light` copy (no image pixels, no computation parameters) for a small terminal or a short-range transfer, and a `full` copy for archiving. They may choose which results (with the frames, layouts and images they need) go in the copy.

**Why this priority**: the light profile makes Bluetooth and small devices practical, but the transports are other features.

**Independent Test**: write both profiles of the same project, check the light file has no image pixels and no parameters and is smaller, and that both open to the same frames, layouts, spools and sequences.

**Acceptance Scenarios**:

1. **Given** a project with a 5 MB source image, **When** it is exported with the `light` profile, **Then** the file holds no source or processed image and no parameters, and opens with the same frames, layouts, spools and sequences; each image is still listed (name, color mode, shape) with a small thumbnail, and each result has its thumbnail.
2. **Given** a `light` project, **When** edit mode tries to modify and save it, **Then** the attempt is refused with a message saying it is a light copy; **When** use mode opens it, **Then** it works normally.
3. **Given** a result on a cylindrical 3D frame, **When** it is exported with the `light` profile, **Then** the mesh geometry in the file is identical to the original and the strings are drawn identically; the attached copy of the imported mesh file, the parametric construction and the texture coordinates are not present.
4. **Given** a project with 5,000 steps in a result, **When** its weaving sequence is stored, **Then** it takes about 15 KB (about 3 bytes per step).
5. **Given** an image carrying GPS or camera metadata, **When** it is embedded in either profile, **Then** that metadata is not present in the file.
6. **Given** a project with three results, **When** the person exports only one, **Then** the copy holds that result and only the frame, layout, image(s) and spools it needs, and is a valid project.

---

### Edge Cases

- **Empty project**: no frame, no layout, no image, no result; or frames, layouts and images only. All valid. A frame with no layout is valid.
- **Light export of a very large mesh**: the geometry is kept whole and never simplified; the size is only warned about.
- **Same frame, several layouts**; **same layout, several results**; **same image, several results**; one result per layout.
- **Reference to a missing item** (frame, layout, image, nail, spool, view), a layout attached to a frame other than the result's, a **reference cycle**, a **duplicate item identifier** (two frames, two layouts or two results with the same identifier).
- **Sparse identifiers**: nail identifiers not contiguous, a label different from the identifier, an identifier at the 32-bit maximum; nail identifiers are unique **within a layout**, not across the project.
- **Spool 31 and above**: the escape code carries the reference; a result may select more than 31 spools.
- **Spool selection**: a result selecting a spool absent from the project, a length above the spool's available length, a step using a spool outside the selection, and a spool shared by results that choose different lengths; all handled as validation rules or valid cases.
- **Very large element counts or sizes** (hundreds of thousands of nails, tens of millions of steps or triangles, gigabytes decompressed at a normal ratio): opens with a warning; the only refusals are an abnormal decompression ratio and a defined out-of-memory or out-of-disk error.
- **Truncated file**: cut at every byte boundary of a small reference file; every cut is rejected cleanly.
- **Duplicate entry names**, an entry listed in the manifest but absent, an entry present but not listed.
- **Text in many scripts**: titles, names and labels with accents, right-to-left text and emoji round-trip.
- **Unknown extension keys**: preserved on rewrite when possible, ignored otherwise; never an error.
- **Time values**: UTC, ISO 8601; a date in the future or without fractions of seconds still opens.
- **Same file opened twice**, or opened (for example in use mode on another device or window) while edit mode saves it: the reader sees either the complete old file or the complete new file, never a mix (FR-028).
- **Mirror**: a file is never mirrored implicitly; a mirrored export is an explicit, separate step.
- **Application data of a deleted result**: the application may keep progress for a result that no longer exists in the file; stable, never-reused identifiers make this detectable and safe, and what the application does with it is its own business.
- **Modifying an item under results**: moving one nail, changing a frame outline or replacing an image removes every dependent result; a frame whose layout is modified keeps its other layouts' results.

## Requirements *(mandatory)*

### Functional Requirements

**Container**

- **FR-001**: The writer MUST produce a single archive with extension `.loomora` whose first entry is `mimetype`, stored uncompressed, containing `application/vnd.loomora+zip`.
- **FR-002**: The writer MUST store each frame, nail layout, image and result as its own set of entries, so that adding, removing or replacing one of them does not rewrite the others' content. The exact entry layout is fixed by the plan and written into §9.1.
- **FR-003**: The reader MUST accept entries stored uncompressed or compressed, and the writer MUST be able to produce either.
- **FR-004**: Writing the same project content with the same options MUST produce the same file structure (entries, names, order, compression method of each entry) and the same decompressed content of every entry, whatever the platform, architecture, byte order or time of saving. The compressed bytes of a deflated entry MAY differ between platforms (they depend on the zlib build); on one platform, the same content and options MUST give byte-identical files.
- **FR-005**: All text MUST be UTF-8 and all binary numbers explicit little-endian; no part of the file may depend on the memory layout of the machine.

**Project, frames, layouts, images, results**

- **FR-006**: The manifest MUST carry: the format name and its major and minor version, the minimum reader version, a stable project identifier (UUID v4, unchanged by every export), title, creation and modification times (UTC, ISO 8601), the generator (application, version, platform, architecture), the profile, the counts of frames, layouts, images, spools and results (the number of nails of a layout and of steps of a result are in `layout.json` and `result.json`), the list of entries with size and checksum, the required features, and a free `extensions` object with namespaced keys.
- **FR-007**: A project MUST be able to hold any number of frames (2D and 3D mixed), nail layouts, images, spools and results; each has a stable identifier, unique in the project, a name chosen by the person, and is never renumbered by saving.
- **FR-008**: A **2D frame** MUST be stored with its outer shape and exclusion shapes (each with type, parameters **and** resolved polyline, so a reader can display it without knowing the shape type), its dimensions and its background color.
- **FR-009**: A **3D frame** MUST be stored with everything a 2D frame has plus its surface kind and body kind and its mesh (positions, triangle indices, optional normals and texture coordinates, flags, fingerprint of the file it was imported from, optional parametric construction) and, optionally, an attached copy of the imported file.
- **FR-010**: A **nail layout** MUST belong to exactly one frame and hold its nails. Each nail MUST carry its identifier (unique within the layout, not necessarily contiguous), position, diameter and kind (boundary, exclusion boundary, interior), and optionally a label; on a 3D frame, also its height, axis, length, side and attachment face.
- **FR-011**: Each image MUST carry its name, its color mode (for example black and white, or color), its shape (for example circle, landscape, or free) and its dimensions, and a thumbnail (in both profiles) and, in the `full` profile, the pixels of the source image and of the processed target image when they exist.
- **FR-012**: Each result MUST carry the identifier of its frame, of one of that frame's layouts, and of the image(s) it reproduces (on a 3D frame, several, one per view with its camera and weight); the selection of project spools it uses, with the length chosen for each (never more than the spool's available length when one is set); its weaving sequence; a thumbnail of the result (in both profiles); and, in the `full` profile, the computation parameters (algorithm, seed, resolution, preprocessing, engine versions).
- **FR-012a**: Each **spool** MUST carry its identifier (unique in the project), name, color, opacity and diameter, and optionally its available length, material and domain (exterior, interior, both).
- **FR-013**: The weaving sequence of a result MUST be stored in the compact binary encoding of §9.1 (nail, winding direction, record type start / segment / end, spool with an escape for 31 and above, optionally the pass height, and an integrity code), and the step list used for machine export MUST be reconstructible from it with no loss.
- **FR-014**: A result MUST be stored as computed and MUST NOT be recomputed when a file is opened. A frame and layout combination with no result MUST NOT be an error.
- **FR-014a**: The format layer MUST be able to list, for a frame, a layout or an image, every layout and result that depends on it. Modifying a frame, a layout, an image or a spool (removal, or a change of color, opacity or diameter) through the format layer MUST remove the results that depend on it (after the caller confirmed, the confirmation itself being the application's), so that a saved project never contains a result inconsistent with its frame, layout or images.
- **FR-014b**: The identifier of an item removed from a project MUST never be given to another item of the same project, so application data kept for a removed result can never apply to a different one.
- **FR-015**: The geometric convention (millimetre, origin at the centre of the frame's bounding box, X right, Y up, Z toward the viewer, front view, CW/CCW seen from the nail tip) MUST apply to every frame, layout and result of every file.
- **FR-016**: The file MUST NOT contain the weaving progress, nor any other data the application changes while the person weaves in use mode; such data is application data, outside the file, and refers to a project and a result by their stable identifiers.
- **FR-016a**: The reader MUST be able to open a file **read-only**, as use mode does: opening, selecting a result and reading it MUST NOT write to the file, change its modification time or create any file next to it. Only an explicit save from edit mode writes a project file, and the modification time (`modified_at`) changes only then.

**Profiles and media**

- **FR-017**: The `full` profile MUST include everything; the `light` profile MUST omit the source and processed images, the computation parameters, the attached copy of the file a mesh was imported from, the parametric construction of a mesh and the texture coordinates, MUST keep (in every 3D frame it retains, see FR-017a) the mesh of the frame itself, and MUST keep the description of each image and a thumbnail (size bounded by a project constant) of each image and each result, and MUST remain a valid project that opens and can be read for use.
- **FR-017a**: The `light` profile MUST keep the mesh geometry of every 3D frame used by a retained result exactly as in the `full` profile (positions, triangle indices and normals, no simplification; only the optional texture coordinates are dropped), so that the drawing of strings is identical.
- **FR-017b**: A `light` project MUST NOT be editable: the format layer MUST refuse any modification of it that edit mode would save (frame, layout, image, spool or result), with a message saying the project is a light copy. Reading it, and exporting a further `light` selection from it, remain allowed; turning it back into `full` is impossible.
- **FR-018**: An export, in particular a `light` one, MUST be able to include only a chosen subset of results, together with the frames, layouts, images and spools those results need, and the copy MUST be a valid project with the same project identifier.
- **FR-019**: Location (GPS) and camera (EXIF) metadata of any embedded image MUST always be removed when the file is written, in both profiles.

**Versioning**

- **FR-020**: A reader MUST ignore unknown fields of a known minor version and SHOULD keep unknown entries when it rewrites a file.
- **FR-021**: A reader MUST reject a file whose major version it does not know, with a message stating the minimum reader version required; a reader of version N MUST be able to read every earlier major version.
- **FR-022**: A reader MUST reject a file that lists a required feature it does not handle, naming the feature; a project whose frames are all 2D MUST list no required feature that only 3D needs.

**Validation, safety, warnings**

- **FR-023**: On every open, the reader MUST verify each entry against its recorded size and checksum and each sequence against its integrity code, and reject the file on mismatch, naming the entry.
- **FR-024**: On every open, the reader MUST verify: identifiers of frames, layouts, images, spools and results are unique in the project, and nail identifiers are unique within their layout; every reference (layout to frame; result to frame, layout, image, view and spool; step to a nail of the result's layout and to a spool of the result's selection; nail to face) is valid and a result's layout belongs to the result's frame; all numbers are finite; nails lie within their frame (with a tolerance); and MUST warn, not reject, when the minimum nail-to-nail distance is not respected or when the centre of a frame's bounding box is not at the origin (FR-015).
- **FR-025**: The reader and the writer MUST NOT refuse a project because of the number of its elements. They MUST report a **warning**, on opening and again on saving, when the count of an element kind (nails of a layout, steps of a result, triangles of a mesh, frames, layouts, images, spools, results), or the total decompressed size of the archive, is above its warning threshold; the warning names the element and its count or size. The thresholds are project constants recorded with the format documentation.
- **FR-026**: The reader MUST reject absolute paths, `..` components and symbolic links in entry names (archive path traversal), MUST guard against decompression bombs, and MUST NOT execute or load any content from a file. The guard has **no fixed size ceiling**: it rejects a file only when the decompression ratio is abnormal. Because the reader streams and a save writes to a temporary file, memory and disk are never predicted: an allocation or a write that fails is reported as a defined error (`LOOMORA_ERR_NO_MEMORY` or `LOOMORA_ERR_IO`) with nothing damaged and any previous file intact; the ratio threshold is a project constant recorded with the format documentation.
- **FR-027**: No input, however malformed, may crash the reader or make it read or write outside its buffers or its destination; a machine without enough memory MUST get a defined error; every failure MUST come with a message a person can act on.
- **FR-028**: A failure while saving MUST leave any previous file at that location intact.

**Access and tooling**

- **FR-029**: Write, read and validate MUST be available to every UI, in both Legacy and Modern variants of every platform, through the existing single boundary of the backend (the stable C interface), with no format logic in any UI. Warnings and errors are returned to the UI, which shows them.
- **FR-030**: A command `loomora validate <file>` MUST run the full validation without any UI, print each problem and warning with the entry and the item concerned, and exit with status 0 on success (warnings allowed) and non-zero on failure.
- **FR-031**: The format code MUST be part of the single C++11 backend (FR-019 of spec 001): no second implementation, no language standard above C++11, and it MUST build with the oldest toolchain of every floor of spec 001, with each fallback declared as the project rule requires.
- **FR-032**: Reading and writing MUST work offline, without graphics acceleration and without elevated rights.

**Tests**

- **FR-033**: The project MUST provide reference files covering every entry kind, both profiles, mixed 2D and 3D frames, several layouts per frame, several images and several results, produced on each architecture of the validation and read back on all the others, including a 32-bit one and, when one is available, a big-endian one.
- **FR-034**: The project MUST provide a corpus of invalid and hostile files with one case per validation rule, a corpus of very large files for the warnings, a lossless round-trip test for every reference file, and a fuzzing run of the reader with no crash.

### Key Entities

- **Project file**: the archive; identified by a stable project identifier; has a format version, a profile, and lists of frames, layouts, images, spools and results.
- **Manifest**: the self-description of a file; lists entries with checksums, counts, required features and extensions.
- **Frame (2D or 3D)**: the physical support with its outline, exclusion zones, dimensions and background; a 3D frame adds a surface mesh.
- **Nail layout**: an arrangement of nails belonging to one frame.
- **Nail**: a point of a layout, with identifier, position, diameter, kind, and for 3D its axis, length, side and face.
- **Image**: a picture with a name, color mode, shape and dimensions, and its pixels in the `full` profile.
- **Result**: one path computation; the combination of a frame and one of its layouts, with one or more images, a spool selection, a weaving sequence and parameters.
- **Spool**: a string reel of the project, with color, opacity, diameter and optional available length, material and domain; shared by the results that select it.
- **Spool selection**: the spools a result uses and the length chosen for each.
- **Weaving sequence**: the ordered records that make the path of a result.
- **View** (3D): a camera tied to an image and a weight, used by a result on a 3D frame.
- **Validation report**: the list of errors and warnings, each tied to an entry and an item.
- **Reference corpus**: valid files per architecture, invalid files per rule, and very large files, used as the regression baseline.
- **Application data** (outside the file): the weaving progress and other data the application changes, keyed by project and result identifiers.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For every reference project (2D, 3D and mixed; one or several frames, layouts, images and results; both profiles), saving then opening yields a project with no difference in any field, in 100% of cases and on every platform of the validation.
- **SC-002**: Every reference file written on one architecture is read with identical content on every other architecture of the validation (including 32-bit and, when available, big-endian), and the same content written twice produces byte-identical files.
- **SC-003**: 100% of the files in the invalid and hostile corpus (one per validation rule, plus truncation at every byte boundary of a small reference file) are rejected with the expected reason and no crash.
- **SC-004**: A fuzzing run of the reader of at least 10 million inputs (or one hour per platform, whichever comes first) ends with no crash, no hang beyond 5 seconds on one input, and no out-of-bounds access.
- **SC-005**: A project with 5,000 nails and 100,000 steps per result is validated and opened on a current desktop in under 1 second; a project 100 times larger is opened with its warnings and is not refused for its size, in 100% of the large-file corpus on machines with enough memory.
- **SC-006**: A weaving sequence of 5,000 steps takes under 20 KB; the `light` profile of a typical project that has image pixels is at least 50% smaller than its `full` profile.
- **SC-007**: Opening a file written by a newer minor version works in 100% of the version-skew test cases; a newer major version is rejected in 100% of them with the minimum reader version in the message.
- **SC-008**: A save interrupted at any point (simulated at every write step) never destroys the previous file, in 100% of runs.
- **SC-009**: No embedded image in any file produced by the test suite carries GPS or camera metadata.
- **SC-010**: Every UI of the validation (Legacy and Modern, each family) opens the reference file, writes a full copy of it, and validates both with the same result as the command-line validation.
- **SC-011**: A project with 3 frames (two 2D, one 3D), 5 layouts (two on the same 2D frame), 4 images (mixed color mode and shape), 6 spools (shared by several results with different selections and lengths) and 5 results keeps every reference intact after a save, an open, the removal of one result and a second save; frame and layout combinations without result raise no error.
- **SC-012**: No file produced by the writer contains any weaving progress; opening a project in use mode, selecting results and recording progress never changes a single byte of the project file or its modification time, in 100% of the tests.
- **SC-013**: A project saved from edit mode after adding one result to an existing project keeps every other frame, layout, image and result byte-identical in content (same checksums for their entries).
- **SC-014**: In 100% of the tests, a `light` project is refused for editing and saving; for every 3D reference project the mesh geometry (positions, triangle indices, normals) of its `light` export is identical to the `full` one; a `light` export of one result out of several holds no item the retained result does not need.

## Assumptions

- The format is described in §9.1 of the requirements document; this feature implements it with the changes of the Clarifications. The text of §9.1 (the single `support`, `nails`, `spools`, `sequence.bin` entries, `progress.json`, the limits), §9.2 (the `progress` route), §9.3 and §10.2 is updated to match, as part of this feature; the entry layout of the multi-item project is decided in the plan.
- The two application modes (edit and use) and their screens are specified on their own; this feature only guarantees what the format needs from them: edit mode reads and writes whole project files, use mode only reads.
- The weaving progress and its synchronization between devices become application data, with their own specification; they are out of scope here. The format only guarantees stable identifiers for projects and results so the application can attach data to them.
- The format lives in the shared backend, like the rest of the project: one C++11 implementation, one stable C interface, no Rust (the requirements text still names crates in places; spec 001 and ADR 0001 supersede it).
- Frames, layouts, images and results are produced by other features (shape and nail generation, image processing, path computation). This feature stores and reads them; it does not compute them.
- Each UI asks the person where to save and open; the backend writes and reads through a file path or a memory buffer. The atomic write guarantee (FR-028) applies to the file path case.
- Writing is deterministic: the creation and modification times come from the project, not from a clock read inside the format code.
- Compression, when used, is the standard ZIP deflate; the writer's choice (stored or deflate) is an option, and FR-004 holds for a given option: the same decompressed content on every platform, byte-identical files on one platform.
- Embedded images are PNG, JPEG or WebP (§4); removing location and camera metadata does not change the pixels.
- Warning thresholds start at the old limits of §9.1 (100,000 nails per layout, 5,000,000 steps per result, 5,000,000 triangles per mesh, 256 MB decompressed) and may be tuned from measurements. A warning never blocks.
- The compact weaving sequence refers to spools through the position in the result's spool selection, which keeps it near 3 bytes per step whatever the project's spool identifiers; the plan fixes the exact encoding and §9.1 is updated.
- Renaming a spool or changing its available length does not remove results; only the properties the computation used (color, opacity, diameter) do.
- The tolerance for "nail within its frame" and the minimum nail-to-nail distance are project constants recorded with the format documentation.
- **Out of scope**: the weaving progress and its synchronization; the HTTP server and its routes (§9.2), the Bluetooth transfer (§9.3), the QR pairing; file association in each operating system, PWA file handling and share targets; the open and save dialogs and the dialogs that show the warnings; mesh import from STL/OBJ/PLY/3MF/glTF; path computation; PDF, G-code and other exports (§8). They use this feature but are specified on their own.
- The validation environment, reference platforms and "one platform stands for the range" rule of spec 001 (Session 2026-11-15) apply to this feature's tests and records.
