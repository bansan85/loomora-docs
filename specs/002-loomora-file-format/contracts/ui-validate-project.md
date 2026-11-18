# Contract: UI probe modes `--validate-project` and `--roundtrip-project`

An addition to the probe mode of the native UIs of spec 001 ([../../001-loomora-cross-platform/contracts/ui-contract.md](../../001-loomora-cross-platform/contracts/ui-contract.md)); the same rules apply to every UI, Legacy and Modern, on every platform: no application logic (FR-029 of spec 001), no network connection, no administrator rights, and nothing written but the result line. Every UI calls only the functions of [c-api-project.md](c-api-project.md) (FR-027, SC-010 of this spec). The two modes prove that each UI reads a `.loomora` file and writes a copy of it through the backend, with the same result as the command line ([cli-validate.md](cli-validate.md)).

## Launch conventions

| Platform | Validate mode | Round-trip mode | File given as |
|---|---|---|---|
| Windows, macOS, Linux | `--validate-project <path>` | `--roundtrip-project <path>` | a path on the command line |
| Android | intent extra `validate_project` | intent extra `roundtrip_project` | a path readable by the app (the launcher pushes the file first) |
| iOS | launch argument `-validate_project <path>` | launch argument `-roundtrip_project <path>` | a path inside the app container or the app bundle |
| Web | URL parameter `?validate_project=<name>` | URL parameter `?roundtrip_project=<name>` | `<name>` is a file bundled with the page, fetched by the page and passed to the backend as a buffer |

`<path>` and `<name>` are percent-encoded UTF-8 where the platform needs it (Web URL, Android extra). The two modes are mutually exclusive with `--probe`; if several are given, the UI exits with status 2 and writes nothing.

## Mode `validate-project`

1. The UI opens the file: `loomora_project_open_path` (Windows, macOS, Linux, Android, iOS), or `loomora_project_open_memory` after reading the whole file into a buffer (Web, and any UI whose platform gives the file only as a buffer). The buffer stays valid until `loomora_project_close`.
2. It prints **one line, identical byte for byte to the line `loomora validate --quiet <file>` prints for the same file** (the summary line of [cli-validate.md](cli-validate.md), without the issue lines): the verdict, the file name as given, the error count and the warning count. When the open fails with a report (`LOOMORA_ERR_CORRUPT`, `_NOT_A_PROJECT`, `_UNSUPPORTED_VERSION`, ...), the line is the failing summary line built from `loomora_report_error_count` and `loomora_report_count`; when it fails without a report (`LOOMORA_ERR_IO`, `_NO_MEMORY`), the UI prints the line `FAIL <file>: error <code>` and uses the exit status 2.
3. It closes the project and the report (`loomora_project_close`, `loomora_report_free`) and exits.

The UI holds no list of codes and no wording of its own for the counts: it reads them from the report. The display name of the file is the argument as it was given (no normalization), so the launcher can predict the line.

## Mode `roundtrip-project`

1. The UI opens the file as in the validate mode. If the open fails, it prints the failing line of the validate mode and exits with status 1 (2 for an I/O or memory failure); nothing else.
2. It writes a **full copy of every result** into memory: `loomora_project_export_stream(project, profile, NULL, 0, modified_at, 0, sink, user, &report)` with `result_ids = NULL` and `n_results = 0` (all results), a `sink` callback that appends each chunk to a growing memory buffer, `options = 0`, and `profile = LOOMORA_PROFILE_FULL` for a `full` file or `LOOMORA_PROFILE_LIGHT` for a `light` file (`loomora_project_is_light`; a full export from a light project is refused, `LOOMORA_ERR_READ_ONLY_PROJECT`). `modified_at` is the value of `modified_at` the manifest already has (read through `loomora_project_manifest`), so the UI reads no clock. No file is created.
3. It validates the copy with `loomora_validate_memory(buffer, len, NULL, NULL, &report)`.
4. It prints `roundtrip ok` followed by one space and the validation line of the copy (the same line as the validate mode, with the file name as given for the original), when the export succeeded and the copy has no error; otherwise it prints `roundtrip FAIL` followed by one space and the validation line of the copy, or `roundtrip FAIL export error <code>` when the export itself failed.
5. It frees the buffer, the report and the project, and exits.

The launcher (`tools/run_vectors.py`, T088) compares the part after `roundtrip ok ` with the line `loomora validate --quiet` prints for the **original** file: a lossless copy has the same counts. For a file with a result subset or with extension entries the comparison is by counts, not by entries.

## Where the line goes

As the probe record of spec 001: standard output (Windows Modern, macOS, Linux); a file next to the executable for the Windows Legacy GUI executable (`loomora-validate-project.txt`, one line, UTF-8, with `OutputDebugString` as well); `logcat` with tag `LOOMORA_VALIDATE` (Android); `NSLog` (iOS); `console.log` and a `<pre id="validate-project">` element (Web). The Windows Legacy executable has no console, so a launcher reads the file. The UI exits by itself with status 0 for a valid file (warnings allowed), 1 when the file has an error, 2 for a usage error or a file that cannot be read at all, as the command line does (Web: no exit; the `<pre>` element carries a `data-status` attribute with the status).

## What the UI must not do

Neither mode does anything else: no window content beyond what the UI shows at startup, no greeting, no file picker, no save, no edit, no retry, no cache, no second validation of the original beyond the one of the open, and no interpretation of the report (the UI never shows or filters issues; it prints the summary line only). Edit-mode functions (`loomora_editor_*`) are not called. The UI opens no network connection and writes nothing but the line (and, for the Windows Legacy GUI, its file).

## Tests

`python tools/run_vectors.py <platform> <variant> --validate-project FILE` and `--roundtrip-project FILE` (T088) start the built UI in the mode, read the line as the probe launcher does, and compare it with `loomora validate --quiet` on the same file: the valid reference files (line identical, status 0), the invalid corpus (line identical, status 1) and a missing path (status 2). Per the PoC scope of spec 001, Legacy UIs run on the lowest testable system, Modern UIs on one system and one 64-bit processor; the other rows stay `not_tested`.
