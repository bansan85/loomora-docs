# ADR 0005: GTK version policy of the Linux Modern UI, a 3.10 floor and newer GTK behind macros

- **Status**: accepted
- **Date**: 2026-11-13
- **Spike / requirement**: T057 / FR-021, FR-022 (fallbacks), research R7 and R12

> **Amended by ADR 0006 (2026-11-14).** The Linux Modern UI is now C++ with gtkmm built with CMake. The policy below (GTK 3.10 floor, newer features behind one macro each, one `fallbacks.json` entry per macro, the floor build, the GTK 4 option) is unchanged; the files and options it names are now: `gtk_compat.hpp` for `gtk_compat.h`, `main.cpp` / `main_window.cpp` for `main.c`, `CMakeLists.txt` with `LOOMORA_GTK_API` and `LOOMORA_GTK_FLOOR` for `meson_options.txt` with `gtk_api` and `gtk_floor` (the static C++ runtime option was removed by ADR 0007), and the C calls (`gtk_label_set_xalign`, ...) are their gtkmm methods. ADR 0006 has the mapping and what the floor build proves with gtkmm. The text and the measurements below describe the C UI and the Meson build of 2026-11-13 and are kept as they were.

## Context

The Linux Modern UI (`ui/modern/linux-gtk3`) was written against the GTK 3.10 API subset and enforced it for every build: `GDK_VERSION_MIN_REQUIRED` and `GDK_VERSION_MAX_ALLOWED` were both pinned to 3.10, so the compiler refused any newer symbol, whichever GTK the machine had. That keeps the floor (about Ubuntu 14.04) but gives a recent distribution nothing from its newer GTK, and rules out GTK 4 completely. The Fallbacks rule of `CLAUDE.md` says a restriction applies only to the case that needs it.

## Decision

1. **The floor stays GTK 3.10.** The UI must always build and run with only the 3.10 API. `GDK_VERSION_MIN_REQUIRED` is 3.10 in every GTK 3 build.
2. **Newer features are allowed, up to GTK 4 and later, each behind one macro.** `gtk_compat.h` defines `LOOMORA_GTK_HAS_<FEATURE>` from the version of the headers being built against (`GTK_CHECK_VERSION`, `GTK_MAJOR_VERSION`); without the macro the code takes the 3.10 path (`#else`). Code outside `gtk_compat.h` never tests a GTK version by itself, it tests the macro. Every macro is declared in `validation/fallbacks.json`, one entry per macro (the Fallbacks rule, FR-021).
3. **Two CMake options** (`ui/modern/linux-gtk3/CMakeLists.txt`):
   - `-DLOOMORA_GTK_API=3|4` (default `3`): gtkmm-3.0 (GTK 3.10 or later) or gtkmm-4.0 (GTK 4). The default build keeps the GTK 3 UI used in every trial.
   - `-DLOOMORA_GTK_FLOOR=ON` (default off): the **floor build**. It pins `GDK_VERSION_MAX_ALLOWED` to 3.10 and compiles with `-Werror=deprecated-declarations`. GTK marks a symbol newer than `MAX_ALLOWED` with a deprecation attribute, so any such symbol is a build error. The same pin makes every `LOOMORA_GTK_HAS_*` macro of `gtk_compat.h` undefined (each one also tests `GDK_VERSION_MAX_ALLOWED`), so the floor build compiles exactly the 3.10 code path. It cannot be combined with `LOOMORA_GTK_API=4`.

   The default build does not pin `MAX_ALLOWED`: it uses the headers' own version and turns the newer features on.

### Version-gated features

| Macro | Needs | With the macro | Without it (3.10 path) |
|---|---|---|---|
| `LOOMORA_GTK_HAS_LABEL_XALIGN` | GTK 3.16 (always in GTK 4) | `gtk_label_set_xalign(label, 0)` for the text of the end-of-life warning | `gtk_misc_set_alignment(label, 0, 0.5)` (GtkMisc, deprecated in 3.14, gone in GTK 4); same left-aligned text |
| `LOOMORA_GTK_HAS_OVERLAY_SCROLLING` | GTK 3.16 (always in GTK 4) | `gtk_scrolled_window_set_overlay_scrolling(FALSE)`: the scrollbar of a long greeting stays visible beside the text | not called; a real 3.10 has no overlay scrollbar, so it is always visible; on a 3.16+ runtime it would be an overlay hidden until the pointer moves |
| `LOOMORA_GTK4` | `GTK_MAJOR_VERSION >= 4`, built with `-Dgtk_api=4` | the GTK 4 branch of the window and the main loop | the GTK 3 UI (default) |

Each entry is in `validation/fallbacks.json` (`gtk-label-xalign`, `gtk-overlay-scrolling`, `gtk4-ui`). No platform of `validation/platforms/` takes the fallback of the first two: `linux-ubuntu-24.04-x86_64` has GTK 3.24.41 headers. It does take the fallback of `gtk4-ui` (the deliverable is the GTK 3 UI). The 3.10 path of the first two is exercised only by the floor build, on a GTK 3.24 runtime.

### The GTK 4 branch

`gtk_compat.h` also hides the calls that differ without being optional (`gtk_init_check`, the text of a `GtkEntry`) and `gtk_compat.c` the main loop (`gtk_main` in GTK 3, a `GMainLoop` in GTK 4). `main.c` has a `#ifdef LOOMORA_GTK4` branch for the window construction: `gtk_window_set_child`, `gtk_box_append`, `gtk_scrolled_window_set_child`, margins for the 12 pixel border, `gtk_label_set_wrap`, `gtk_window_present`. Behaviour kept: same content, 1,000 character limit, greeting, scrolled label, probe (`probe.c` is shared), exit semantics.

What is not reproduced exactly, on purpose:

- **End-of-life warning.** `GtkInfoBar` is deprecated since GTK 4.10 and has no replacement. The GTK 4 branch uses a plain box with the `dialog-warning-symbolic` icon, the text and a flat close button. It has no warning-coloured background and no message type.
- **First display for the probe.** GTK 4 has no `draw` signal. `display_ms` is taken at the first `after-paint` of the frame clock of the window (end of the first painted frame), a little later than the first draw of GTK 3.
- **Look.** GTK 4 draws the entry and the selection with its own theme (lighter selection colour, same layout and fonts).

The GTK 4 build has no trial verdict and no deliverable record; it is a verified build, not a validated target.

## Rejected alternatives

- **Pin 3.10 for everybody** (the previous state): safe, but permanently denies every newer GTK feature to the systems that have them. Against the Fallbacks rule.
- **Runtime checks of the GTK version** (`gtk_check_version`, `dlsym`): the 3.10 build would still need the newer headers and symbols to compile, and a program built against the 3.10 headers cannot call them. A compile-time macro keeps the floor build honest.
- **GTK 4 selected by the macros alone, no CMake option**: the headers decide, and a machine with both `gtk+-3.0` and `gtk4` can only build one UI at a time. A build option says which one is wanted.
- **A separate source tree for GTK 4**: duplicates the probe contract and the greeting code; one tree with one branch keeps the contract in one place.

## Consequences

- The executable of the default build changes (new code, no more pinned headers), so the executable hash recorded by the existing Linux trial evidence no longer matches a rebuild: the trial must be redone after this change.
- `tools/build.py linux modern` builds the default variant. To build the other two: `cmake -S . -B <dir> -G Ninja -DLOOMORA_GTK_FLOOR=ON` or `-DLOOMORA_GTK_API=4`, then `cmake --build <dir>`. `tools/run_vectors.py` takes the executable of `$LOOMORA_BUILD_DIR/linux-modern/loomora-gtk3`.
- A new newer-than-3.10 feature means a new macro in `gtk_compat.h`, a use of it in the code, its `#else` path in 3.10, and an entry in `fallbacks.json`; the floor build then fails if the 3.10 path used anything newer.
- GTK 4 under WSLg prints Mesa EGL warnings on standard error (`libEGL warning: DRI3 error` on the X11 backend, `failed to get driver name` / `MESA-LOADER` on the default Wayland backend: 134 and 265 bytes). They come from the OpenGL renderer, not from the UI. On the default Wayland backend they disappear with `GSK_RENDERER=cairo` or `LIBGL_ALWAYS_SOFTWARE=1` (0 bytes); on the X11 backend `GSK_RENDERER=cairo` does not remove them. The GTK 3 builds print nothing.

## Verified (2026-11-13, WSLg, Ubuntu 24.04, GCC 13.3, GTK 3.24.41 and GTK 4.14.5)

| | floor (`-Dgtk_floor=true`) | default (GTK 3.24) | GTK 4 (`-Dgtk_api=4`) |
|---|---|---|---|
| warnings at build | 0, with `-Werror=deprecated-declarations` | 0 | 0 |
| `run_vectors.py` | 15 of 15 | 15 of 15 | 15 of 15 |
| `--probe`, `--probe-input "Zoé 日本 😀"`, `check_probe.py` | OK, OK | OK, OK | OK, OK |
| window 4 s without `--probe` | still running, stderr 0 bytes | still running, stderr 0 bytes | still running, stderr 134 bytes of EGL warnings (X11 backend) |
| typing "Marie Curie" (XTest) | `Bonjour Marie Curie` | same | same |
| symbols imported | `gtk_misc_set_alignment`, no `xalign` or `overlay_scrolling` | `gtk_label_set_xalign`, `gtk_scrolled_window_set_overlay_scrolling` | same two, `gtk_box_append`, `gtk_window_set_child` |

The floor build rejects a newer symbol: `gtk_label_set_xalign` compiled with the floor flags fails with `'gtk_label_set_xalign' is deprecated: Not available before 3.16 [-Werror=deprecated-declarations]`, and compiles without them.

Screenshots (WSLg, X11 backend, GTK 4 with `GSK_RENDERER=cairo`): same window content in the three builds (entry, greeting at 1.8 times the font size, Noto fonts, accents, CJK, emoji, Arabic and Hebrew). With a 1,000-character greeting the floor build shows no scrollbar (overlay scrollbar hidden), the default and GTK 4 builds show it. With the end-of-life warning (`uname` faked to 2.6.32 with `LD_PRELOAD`, probe `eol_warning_shown` true) the GTK 3 builds show the coloured `GtkInfoBar` and the GTK 4 build the plain box.

## Not verified

- A real GTK 3.10 system: the floor build proves, at compile time, that the 3.10 path uses no newer symbol, but it was run on a GTK 3.24 runtime.
- Other distributions and GTK versions between 3.10 and 3.24, GTK 4 versions other than 4.14.5.
- The end-of-life warning is shown only with a faked kernel version (the real kernel is not end of life); closing it with the button was not tried in the GTK 4 branch.
- Dark and high-contrast themes, Wayland-specific behaviour of the GTK 4 branch (screenshots use X11).
- Pre-existing, not changed here: with the end-of-life warning the GTK 3 window opens 430 pixels high instead of 240 (also true of the code before this change).
