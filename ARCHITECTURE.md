# Architecture notes

Orientation for working on this repository: how to build and verify, and the
parts of the single-file design that take reading several thousand lines to
piece together.

## Build and run

```sh
make                    # builds ./fetch from fetch.c
make clean
sudo make install       # PREFIX=~/.local make install for a user install
./fetch --frames 1      # single frame, useful for eyeballing a render change
./fetch --no-info       # logo only
./fetch --infinite      # run until keypress
```

There is no test suite and no linter. Verification is manual: build, run, look at
the output. `--frames N` bounds a run so it can be used non-interactively.

The Makefile injects `FETCH_VERSION` (from the `VERSION` file), `FETCH_CODENAME`,
`FETCH_ARCH` and `FETCH_OS` as `-D` defines. Building `fetch.c` by hand without
them falls back to `"dev"` for the version and fails to compile `--version`, so
always go through `make`. On Darwin the Makefile adds `-framework IOKit
-framework CoreFoundation`; `-DLEGACY_IOKIT` compiles out the IOKit power-source
path for old SDKs.

## Architecture

One file: `fetch.c`, ~4.6k lines, libm the only dependency. Linux and macOS are
split with `#ifdef __APPLE__` / `#ifndef __APPLE__` — there is no `__linux__`
guard anywhere, Linux is always the else branch. Anything new that reads `/proc`,
`/sys` or DRM needs an `#ifdef __APPLE__` counterpart or a stub.

The frame pipeline, in `main()` order:

1. Parse argv, then `config_defaults()` + `load_config()`. CLI flags win over
   config; config wins over defaults.
2. Load the logo. Precedence: `-l <name>` (fastfetch) > `~/.config/fetch/logo.txt`
   > detected distro via fastfetch > each token of `ID_LIKE` > built-in Gentoo.
3. `process_logo()` splits the raw rows into per-cell codepoints and strips ANSI
   into a per-cell color index.
4. `gather_*()` for each enabled field, then optional `box_wrap_lines()`, then
   `apply_layout()`.
5. `build_points()` turns the logo into a heightmap (`char_weight_utf8()` maps a
   codepoint to visual density), samples it into a point cloud with normals from
   finite differences, and fills `PX/PY/PZ`, `NX/NY/NZ`, `PCOLOR`.
6. The render loop: rotate + project every point into `zbuf`/`lumbuf`/`colorbuf`,
   `cell_glyph()` collapses each cell's sub-samples to one glyph, the whole frame
   is assembled into one reused buffer and flushed with a single `write()`.

### Info fields

A field is five places that must stay in sync: the `F_*` enum, `field_map[]`,
`config_defaults()`'s `defaults[]`, the `fns[]` dispatch table in `main()`, and a
`gather_<name>()`. Plus the `--help` text, README and `docs/configuration.md`.

`add_info()` is the only way to emit a labeled line. It records the line index in
`field_line[current_field]` on the first pass, so the refresh tick can rewrite
that row in place. A gather that emits several rows (multi-GPU, multi-disk) works
because only the first pass appends.

Live refresh happens every 20 frames (~1s) and is limited to uptime, memory and
swap **because those are pure `/proc` reads**. Do not add a `popen()`-backed
field to the refresh block — it stalls the animation. Battery, disk and IP are
deliberately static.

`box_wrap_lines()` runs once, after every gather, and rewrites `fetch_lines[]`
while shifting `field_line[]`; `add_info()` re-pads to `box_width` on refresh.

### Layout and resizing

`apply_layout()` is the single sizing authority, called at startup and again on
`SIGWINCH` (the handler only sets `term_resized`). Three regimes: full 60-col
canvas beside the info, shrunken canvas when the terminal is narrower, and
stacked (info below a full-width logo) when shrinking would make the canvas
unreadable. Info lines clip, never wrap. `render_height` covers the info rows;
`logo_height` is capped separately at `anim_width * 3/5` because the projection
needs ~5/3 the columns of its rows.

### Terminal input — keypress passthrough

The exit path is load-bearing and easy to break. `FIONREAD` is checked before any
`read()`: if exactly one byte is pending it is a keypress, and the loop breaks
**without consuming it** so the shell receives it (this is what makes fetch usable
as a startup fetch, and replaced an older `TIOCSTI` approach). Only escape
sequences are read and parsed, for SGR mouse reporting (`\033[<btn;x;yM`) used by
drag-to-rotate. Never add an unconditional `read()` to that loop.

Raw mode, `\033[?25l` and mouse modes 1002/1006 are undone by `cleanup()`, wired
through `atexit()`. Handlers go in right after argv parsing, before the
`gather_*()` calls — those spend a couple of seconds in `popen()`, and a signal
there would otherwise kill the process outright.

`handle_signal()` branches on the `animating` flag, set once the render loop owns
the screen. Before that nothing has been drawn, so it just cleans up and exits.
During the animation the first signal only raises `interrupted`, which the loop
checks, so SIGINT leaves through the normal path and prints the settled logo
instead of calling the non-async-signal-safe `cleanup()` from signal context; a
second signal does call it and `_exit()`, as the escape hatch.

### Exiting — the flat logo

`print_flat_logo()` renders the final frame: the logo flat, from the parsed
`logo_cells[]` and their ANSI colors, instead of the 3D relief frozen at a
random rotation. It mirrors `apply_layout()`'s three regimes so the info block
does not shift between the last animated frame and the settled one — change one
and the other needs the same change. The flat logo cannot be scaled like the 3D
one, so when space is tight it is drawn whole, clipped evenly top and bottom, or
omitted, never cut off at one end. Row usage stops one short of the terminal
height to leave the prompt its line.

### Shading modes

`ascii` is the default and the intended look; `blocks` (2×2) and `sextants` (2×3)
are opt-in sub-cell modes. `select_shading()` sets `sub_rows`/`sub_cols`, which
scale the render buffers and the projection. Sextant glyphs are generated
arithmetically over U+1FB00..U+1FB3B, skipping the two masks Unicode already has
as half blocks. `MAX_POINTS` (200k) is sized for sextants at `--size 3`.

### ANSI-aware strings

Logo and info lines carry embedded escapes, so column math never uses `strlen()`.
Use `visible_width()` to measure, `emit_clipped()` to write clipped output (it
re-emits a reset if it cut a color short), `skip_ansi()` to step over an escape,
`utf8_char_len()` to step over a codepoint.

### Fixed caps

Static buffers throughout: `MAX_LOGO_ROWS` 64, `MAX_LOGO_COLS` 128,
`MAX_FETCH_LINES` 32, `MAX_LINE_LEN` 512, `MAX_HEIGHT` 200, `MAX_POINTS` 200000,
`MAX_EXTRA_DISKS` 8. Adding fields or bigger logos may mean raising one of these
rather than the code being wrong.

### Config file

`~/.config/fetch/config`. Its mere presence resets `field_enabled[]` and takes the
field order entirely from the file — a config listing three fields shows three
fields. `key=value` lines set appearance/3D options; bare names are fields;
`disk=/path` appends a mount point. `strip_inline_hint()` lets a value carry a
trailing `" (...)"` hint, so users can paste the commented reference block
verbatim.

### fastfetch integration

fastfetch is optional, invoked through `popen()`, and every call passes `-c none`
so a user's fastfetch config cannot alter the output being parsed. Preferred path
is colored logo output (per-character colors preserved); `--print-logos` is the
fallback for older versions; `--json` is used for distro detection because it
beats `/etc/os-release` on derivatives.

## Releasing / version bumps

`VERSION` is the source of truth for `make` and for `nix/package.nix`
(`lib.fileContents ../VERSION`). These are **not** derived and must be bumped by
hand alongside it: `fetch.spec` (`Version:` plus a `%changelog` entry),
`debian/changelog` (new stanza), and the `filename` in `_service`. Packaging for
Fedora/openSUSE/Debian is maintained by outside contributors — see the README's
package manager section.
