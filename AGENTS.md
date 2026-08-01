# AGENTS.md

Fork `file-roller-DnD` of GNOME File Roller 44.7 (latest upstream release). The entire point of this fork is the drag-and-drop extraction feature, which lives only in `src/fr-window.c`. Upstream: https://gitlab.gnome.org/GNOME/file-roller (this remote points to the user's GitHub fork).

## Build
- Meson + ninja. `build/` is already configured (gitignored, buildtype=debug).
  - Configure: `meson setup build` (README says `_build`; the real dir here is `build/`)
  - Build: `ninja -C build`
  - Run: `meson devenv -C build/ file-roller`
- README dependency list is stale (GTK3 era). Real deps in `meson.build`: `gtk4 >= 4.8.1`, `libadwaita >= 1.2`, `glib >= 2.38`, libportal/libportal-gtk4 (optional, only with `use_native_appchooser`), libarchive, json-glib.
- `run-in-place` meson option loads UI/data from the source tree instead of the install prefix.
- Version lives in `version:` in `meson.build` and `NEWS`; releases are git-tagged.
- CI builds via `nix-shell` (`.gitlab-ci.yml`). `PKGBUILD` packages this fork on Arch as `file-roller-dnd-git` (conflicts with stock `file-roller`); `flatpak/org.gnome.FileRoller.json` is the Flatpak manifest.

## Tests
- Only one real test: `safe-path`. Run with `meson test -C build`. `src/test-server.c` is a helper binary, not a test.

## Drag-and-drop feature (the fork's core)
- Flow in `fr-window.c`: on selection change, selected files are pre-extracted async to a temp dir; URIs are cached and returned synchronously in `GtkDragSource::prepare`.
- Do not break the deferred temp-dir cleanup: `drag_old_tmp_dir` is kept because the extraction thread may still be writing to the current temp dir. Stale-extraction detection compares destination paths. Extraction is blocked while another archive operation runs.
- The system `/usr/bin/file-roller` (stock Arch 44.7) does NOT have this feature — test the locally built `build/src/file-roller`.
- The working tree may carry uncommitted WIP in `src/fr-window.c` (drops originating from the archive's own temp dir).

## Architecture
- `HACKING` documents the format plugin model: `FrArchive` subclasses for libarchive-backed formats, `FrCommand` subclasses for external-tool formats, all registered in `src/fr-init.c`.
