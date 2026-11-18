# Contract: `loomora validate`

A host program (`cli/loomora/main.cpp`, built for Windows, Linux and macOS hosts; not for Android, iOS or Web). It links the backend and calls `loomora_validate_path`; it has no logic of its own (FR-030).

```text
loomora validate [--json] [--quiet] [--max-issues N] [--set-limit name=value] <file> [<file> ...]
loomora --version
```

## Behaviour

- Validates each file with the full validation (container, checksums, schemas, references, numbers, nails, binary entries) and prints one line per issue, then one summary line per file.
- Opens the file **read-only**; never creates or changes a file.
- A progress line goes to standard error only when `--progress` is given.

## Output (default, UTF-8)

```text
error   entry.sha256        entry=frames/1/layouts/2/nails.json            expected a1b2…  found c3d4…
error   ref.missing         entry=results/3/result.json  result=3          image 9 does not exist
warning warn.count          entry=frames/1/layouts/2/nails.json layout=2   250000 nails (threshold 100000)
FAIL project.loomora: 2 errors, 1 warning
```

`--json` prints one JSON object per file on one line: `{"file":"…","valid":false,"errors":2,"warnings":1,"issues":[{"severity":"error","code":"…","entry":"…","item":{"kind":"result","id":3},"message":"…"}]}`.

`--set-limit name=value` (test only, not listed in `--help`) lowers a warning threshold or the ratio guard for this run (`warn_nails`, `warn_steps`, `warn_triangles`, `warn_total_bytes`, `warn_items`, `bomb_ratio`), so the large-file corpus can be small; it calls the setters of the internal limits object and is never read from the environment.

`--quiet` prints only the summary lines. `--max-issues N` stops printing after N issues (the counts stay exact).

## Exit status

| Status | Meaning |
|---|---|
| 0 | every file valid (warnings allowed) |
| 1 | at least one file has an error |
| 2 | usage error, or a file that could not be read at all (missing, not a file, unreadable) |

When several files are given the status is the highest of their statuses. The convention is that of the project's host tools (ADR 0015).

## Tests

`tools/verify.py quick` runs the command on the valid reference files (status 0), on the invalid corpus (status 1, the expected `code` for each file in a table of the test) and on a missing path (status 2).
