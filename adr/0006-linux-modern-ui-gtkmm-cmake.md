# ADR 0006: Linux Modern UI in C++ with gtkmm, built with CMake

- **Status**: accepted (amends ADR 0005, which describes the C UI and the Meson options as decided on 2026-11-13; amended by ADR 0007: no static C++ runtime option, the `istringstream` parsers stay for their own sake)
- **Date**: 2026-11-14
- **Spike / requirement**: owner decision of 2026-11-14 / FR-017 (backend in C++11 behind the C API), FR-021, FR-022 (fallbacks), research R7 and R12

## Context

The Linux Modern UI (`ui/modern/linux-gtk3`) was written in C against the GTK C API and built with Meson (ADR 0005, `main.c`, `probe.c`, `gtk_compat.c/.h`, `meson.build`, `meson_options.txt`). The owner decided on 2026-11-14 that it is C++ with gtkmm, using the system packages, and built with CMake like the rest of the repository; Meson is dropped. The same decision fixes the C++ standards of the project:

- the **Legacy UIs** are C++11;
- the **Modern UIs** may use any standard up to C++17;
- the **backend** (`core/`, the library `libloomora` and its tests) is **always C++11**, whatever the standard of the UI that links it, and the UI talks to it only through the C API of `core/include/loomora.h`.

The GTK version policy of CLAUDE.md and ADR 0005 stays: minimum GTK 3.10 API, newer features only behind compile-time macros, each declared as a fallback in `validation/fallbacks.json`.

## Decision

### 1. Language and structure

The UI is written as C++ classes on gtkmm; no manual memory management (widgets are members, the window is owned by a `std::unique_ptr`, the application by a `Glib::RefPtr`), text is `std::string` and `Glib::ustring`, handlers are sigc++ slots made from lambdas.

| File | Content |
|---|---|
| `main.cpp` | `Application` (derived from `Gtk::Application`: activation, the 2 second probe timer, exit status) and `main()` (ABI check, probe arguments, exceptions stopped at the top) |
| `main_window.hpp/.cpp` | `MainWindow` (derived from `Gtk::Window`: field, scrolled greeting label, first display) and `EolBar` (the end-of-life banner) |
| `probe.hpp/.cpp` | `parse_probe_args`, `platform_id`, `Probe` (launch and first-display times, `loomora_probe_record`, the stdout line); no GTK, no gtkmm |
| `gtk_compat.hpp` | the version policy: `LOOMORA_GTK4`, `LOOMORA_GTK_HAS_LABEL_XALIGN`, `LOOMORA_GTK_HAS_OVERLAY_SCROLLING`, and the names of the enumerations that gtkmm 3 declares as plain enums and gtkmm 4 as scoped ones |
| `CMakeLists.txt` | the target `loomora_gtk3` (executable `loomora-gtk3`), the options, `pkg_check_modules` |

`gtk_compat.c` (the main loop of the GTK 4 branch) is gone: `Gtk::Application::run` is the main loop of both versions.

Behaviour kept: window 480 x 240, title "Loomora", 12 pixel border, field prefilled "Coucou" with a limit of 1,000 characters (counted in characters by `Gtk::Entry::set_max_length`, not in bytes), label "Bonjour " + text from `loomora_greet` at 1.8 times the theme font, wrapped on words and characters, centred, in a scrolled window (no horizontal scroll, vertical scrollbar when needed), end-of-life banner from `loomora_is_eol` and `loomora_eol_warning`, error text when the greeting fails, probe mode (`--probe`, `--probe-input`, usage error with exit status 2, one stdout line, exit after 2 seconds, status 1 when the record cannot be built).

### 2. Build: Meson options become CMake cache options

`tools/build.py linux modern` configures the whole repository (`cmake -S . -B ~/loomora-build/linux-modern -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DLOOMORA_BUILD_TESTS=OFF -DLOOMORA_BUILD_LINUX_MODERN=ON`), like `windows legacy`; the top-level `CMakeLists.txt` adds the subdirectory only when `LOOMORA_BUILD_LINUX_MODERN` is ON, so a host without gtkmm is not affected. The executable is `<builddir>/loomora-gtk3`, the path `tools/run_vectors.py` looks at.

| Meson (ADR 0005) | CMake |
|---|---|
| `-Dgtk_api=3` (default) | `-DLOOMORA_GTK_API=3`: `pkg_check_modules(gtkmm-3.0>=3.10)` |
| `-Dgtk_api=4` | `-DLOOMORA_GTK_API=4`: `gtkmm-4.0` |
| `-Dgtk_floor=true` | `-DLOOMORA_GTK_FLOOR=ON`: `GDK_VERSION_MAX_ALLOWED` 3.10 and `-Werror=deprecated-declarations`; still refused with `LOOMORA_GTK_API=4` |
| `warning_level=2`, `buildtype=release` | `-Wall -Wextra` on the UI target, `CMAKE_BUILD_TYPE=Release` |

`GDK_VERSION_MIN_REQUIRED` is 3.10 in every GTK 3 build, as before.

### 3. C++ standards of this UI

- The target `loomora_gtk3` has the property `CXX_STANDARD` set **on that target only** (`CXX_STANDARD_REQUIRED ON`, `CXX_EXTENSIONS OFF`); `CMAKE_CXX_STANDARD` is never set. `libloomora` keeps the properties of `core/CMakeLists.txt` (11, required, no extensions).
- **gtkmm-3.0: C++11.** The headers of gtkmm 3.24.9, glibmm 2.66, pangomm 1.4, giomm and libsigc++ 2.12 compile with `-std=c++11` (tried: no error, no warning), and the UI needs nothing newer than C++11 (lambdas, `std::function`, `std::unique_ptr`, `std::array`, `constexpr`, `override`). C++11 is the lowest standard those headers accept (glibmm 2.66 requires it), the lowest the project allows for a UI, and the same as the backend, so there is no reason to go higher and a distribution with an older gcc stays served.
- **gtkmm-4.0: C++17.** gtkmm 4.10, glibmm 2.68 and libsigc++ 3 require it: the same headers fail under `-std=c++11` inside `sigc++-3.0/sigc++/functors/slot.h`.
- Checked on the compile commands of every build (`compile_commands.json`, and again by `tools/build.py linux modern`, which fails otherwise): the six `core/src/*.cpp` sources are compiled with exactly `-std=c++11`, the three UI sources with `-std=c++11` (gtkmm-3.0) or `-std=c++17` (gtkmm-4.0). `tools/check_std.py` keeps checking the backend at strict C++11 with GCC and Clang and has a new step 5: no source under `ui/` includes a header of `core/src`, only `loomora.h`.
- **No exception crosses the C API.** The backend never calls back into the UI, so no UI exception can enter it; the UI calls only `loomora_*` functions, which do not throw (`core/tests/boundary_check.cpp`, run by `check_std.py`). On the UI side the slots called from GTK's C code (`show_greeting`, the probe timer) catch what the standard library may throw, and `main()` stops anything left (`Glib::Error` in gtkmm 3, which is not a `std::exception`, and `std::exception`).

### 4. GTK version policy with gtkmm

- The macros are defined in `gtk_compat.hpp` from the version of the **GTK headers** (`GTK_CHECK_VERSION`) and, in GTK 3, also of the **gtkmm headers** (`LOOMORA_GTKMM_AT_LEAST`, `GTKMM_MAJOR_VERSION`/`GTKMM_MINOR_VERSION`): gtkmm 3.x wraps GTK 3.x with the same minor number, so a call to `Gtk::Label::set_xalign` needs gtkmm 3.16 as well as GTK 3.16. Their names and the entries of `fallbacks.json` do not change (`gtk-label-xalign`, `gtk-overlay-scrolling`, `gtk4-ui`).

| Macro | With the macro | Without it (3.10 path) |
|---|---|---|
| `LOOMORA_GTK_HAS_LABEL_XALIGN` | `Gtk::Label::set_xalign(0)` | `Gtk::Misc::set_alignment(0, 0.5)` |
| `LOOMORA_GTK_HAS_OVERLAY_SCROLLING` | `Gtk::ScrolledWindow::set_overlay_scrolling(false)` | not called |
| `LOOMORA_GTK4` | the GTK 4 branch (gtkmm-4.0, C++17) | the GTK 3 UI |

- **What the floor build proves with gtkmm, and what it does not.** `LOOMORA_GTK_FLOOR=ON` pins `GDK_VERSION_MAX_ALLOWED` to 3.10: every GTK or GDK C symbol newer than 3.10 is then an error (`gtk_label_set_xalign` fails with `'gtk_label_set_xalign' is deprecated: Not available before 3.16 [-Werror=deprecated-declarations]`, tried in a scratch file with the gtkmm headers included), and every `LOOMORA_GTK_HAS_*` macro is off, so the 3.10 path is the one compiled (the floor executable imports `Gtk::Misc::set_alignment` and neither `Gtk::Label::set_xalign` nor `Gtk::ScrolledWindow::set_overlay_scrolling`). A **gtkmm method is not marked by GTK**: `label.set_xalign(0)` compiles in the floor build. The compiler therefore does not prove that the gtkmm calls of the 3.10 path exist in gtkmm 3.10; that is checked by hand with the table below, and was not compiled against gtkmm 3.10 headers (none is available).
- gtkmm methods used in the 3.10 path, with the GTK version of the function each one wraps: `Gtk::Window` (`set_title`, `set_default_size`), `Gtk::Container::set_border_width` and `add`, `Gtk::Box::pack_start`, `Gtk::Entry` (`set_max_length`, `set_text`, `get_text`, `signal_changed`), `Gtk::Label` (`set_text`, `set_line_wrap`, `set_line_wrap_mode`, `set_justify`, `set_attributes`), `Gtk::Widget::set_halign` / `set_valign` (3.0), `Gtk::Misc::set_alignment` (the 3.10 fallback), `Gtk::ScrolledWindow::set_policy`, `Gtk::InfoBar` (`set_message_type`, `get_content_area`, `set_show_close_button` since 3.10, `signal_response`), `Pango::AttrList` and `Pango::Attribute::create_attr_scale`, `Gtk::Application` (3.0) with `Gio::APPLICATION_NON_UNIQUE | Gio::APPLICATION_HANDLES_COMMAND_LINE`, `Glib::signal_timeout().connect_once`. All exist in the GTK 3.10 API; whether gtkmm 3.10 wraps each of them under the same name was not verified.
- **A binary built here runs against the gtkmm of its build.** `libgtkmm-3.0.so.1` is linked to the GTK 3.24 symbols, so the executable cannot run on a system whose gtkmm is older; the floor of a system is "a distribution whose gtkmm-3.0 builds the source", which is where the 3.10 path matters (Ubuntu 14.04 ships gtkmm 3.10 and C++98-era glibmm; whether that glibmm accepts `-std=c++11` was not tried).

### 5. Differences from the C UI

- **Main loop and start.** `Gtk::Application` with the flags `NON_UNIQUE | HANDLES_COMMAND_LINE` replaces `gtk_init_check` + `gtk_main` (GTK 3) and the `GMainLoop` (GTK 4): nothing is registered on the session bus (a probe run leaves no process behind) and `--probe` / `--probe-input` are read by the program from `argv` (raw bytes, UTF-8 checked with `Glib::ustring::validate`), not by GApplication. Unknown arguments, among them GTK's own (`--display`), are accepted as before. **A display that cannot be opened** still ends the program with status 1, but the message is GTK's (`Gtk-WARNING ... cannot open display`) instead of `loomora: cannot open the display`.
- **First display.** GTK 3: the `on_draw` override calls `Gtk::Window::on_draw` and then notifies, the same moment as the `draw` signal connected with `connect_after`. GTK 4: gtkmm 4.10 does not wrap the `after-paint` signal of the frame clock, so the notification is connected to its `paint` signal after GTK's own handlers (`connect(slot, true)`), a little before the end of the frame that `after-paint` marked; only the first call counts.
- **Info bar.** `Gtk::Window::show_all()` does not show the GTK 3 `Gtk::InfoBar` (the first gtkmm build opened a 480 x 430 window with no banner); `show_window()` shows it explicitly. The GTK 4 banner is a plain box (icon, text, flat close button) as before.
- **Number parsing.** The kernel version and `VmHWM` are read with `std::istringstream` instead of `std::sscanf` / `std::strtoll` (first done to avoid `__isoc23_*` / GLIBC_2.38; kept by ADR 0007 because it is the natural C++ code, the glibc reason being void now that binaries are built on the machine that runs them).
- **`Glib::ustring(const char*, n)` counts characters.** The greeting is built from a `std::string` of the exact byte count the backend returned.

### 6. The static C++ runtime option was removed

The CMake option `LOOMORA_STATIC_CXX_RUNTIME` that this ADR first kept was removed on 2026-11-14 (ADR 0007): libstdc++ and libgcc are always linked dynamically, gtkmm needs `libstdc++.so.6` anyway.

## Rejected alternatives

- **Keep Meson for the UI.** Two build systems for one repository, and the options of the UI could not be seen from the CMake presets; the owner dropped it.
- **`Gtk::Main` for gtkmm 3.** It does not exist in gtkmm 4, so the GTK 4 build would need another path. `Gtk::Application` is one code path for both.
- **Let GApplication parse `--probe`.** Not tried: from the documentation of GOption, a string option is converted from the locale charset (which fails for non-ASCII input in the C locale), a missing value is reported by GOption with exit status 1 instead of the usage error 2, and an unknown option is an error, none of which the C UI did.
- **C++14 or C++17 for the gtkmm-3.0 build.** Nothing in the UI needs it and it would exclude compilers that accept C++11 only.
- **Compile the gtkmm 3 UI as C++17 everywhere to have one standard.** The rule is per case: the lowest the headers accept, per target.
- **A separate source tree for GTK 4.** As in ADR 0005: one tree with one branch keeps the contract in one place.

## Consequences

- Measured, Ubuntu 24.04, GCC 13.3, gtkmm 3.24.9 / GTK 3.24.41 and gtkmm 4.10.0 / GTK 4.14.5, `Release`, 0 warnings in every build. Before (C, Meson, same machine, rebuilt from the previous commit) and after (C++, CMake):

| Build | Size before | Size after | Libraries resolved by `ldd` before / after | Highest GLIBC, GLIBCXX, CXXABI after |
|---|---|---|---|---|
| default (GTK 3) | 54,016 | 167,192 | 71 / 79 | 2.34, 3.4.21, 1.3 |
| floor | 53,952 | 167,120 | 71 / 79 | 2.34, 3.4.21, 1.3 |
| GTK 4 | 54,480 | 169,920 | 69 / 75 | 2.34, 3.4.21, 1.3.9 |

- New direct run-time dependencies of the GTK 3 build: `libgtkmm-3.0.so.1`, `libatkmm-1.6.so.1`, `libgiomm-2.4.so.1`, `libpangomm-1.4.so.1`, `libglibmm-2.4.so.1`, `libsigc-2.0.so.0` (before: `libgtk-3.so.0`, `libpango-1.0.so.0`, `libgobject-2.0.so.0`, `libglib-2.0.so.0`; `libstdc++.so.6`, `libgcc_s.so.1` and `libc.so.6` in both). GTK 4 build: `libgtkmm-4.0.so.0`, `libpangomm-2.48.so.1`, `libgiomm-2.68.so.1`, `libglibmm-2.68.so.1`, `libsigc-3.0.so.0`. The distribution packages `libgtkmm-3.0-1t64` (or `libgtkmm-4.0-0`) must be installed to run the UI, not only GTK.
- A probe run uses about 8 MB more memory (peak_rss_kb 45,820 to 46,600 kB against 37,716 to 38,336 kB) and starts as fast (display_ms 90 to 152 against 91 to 239).
- The executable hash recorded by the earlier Linux trial evidence no longer matches: the earlier evidence (`validation/evidence/*/linux-ubuntu-24.04-x86_64/`) is for the C UI and stays as history, the new evidence is in `validation/evidence/gtkmm-cmake/linux-ubuntu-24.04-x86_64/` and the verdict says which is which. `tools/trials/linux-ubuntu-24.04-x86_64/gtk-floor-and-runtime-check.sh` and ADR 0004 (findings 12 and 13) describe the Meson build and the C objects and are superseded by this ADR for the UI.
- A new newer-than-3.10 feature means a macro in `gtk_compat.hpp`, its use, its 3.10 `#else` path, a `fallbacks.json` entry and a line in the table of section 4.
- GTK 4 under WSLg still prints Mesa EGL warnings on standard error (ADR 0005); the GTK 3 builds print nothing.

## Verified and not verified

Verified and not verified are in the verdict `verdicts/linux-ubuntu-24.04-x86_64-modern.json` (field `restrictions`) and in the evidence of `validation/evidence/gtkmm-cmake/`. Not verified at all: a real GTK 3.10 / gtkmm 3.10 system (the 3.10 path is only the floor build on a GTK 3.24 runtime), any other distribution, `clang++` as the UI compiler, a system with the gtkmm 3 headers older than 3.24.
