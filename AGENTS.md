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
- Flow in `fr-window.c`: on `GtkDragSource::prepare` the `FrDragContentProvider` allocates a temp dir and starts extracting the selection asynchronously (`fr_drag_content_provider_start_extraction`). The drop target is served the cached uri-list only after the extraction completes (`fr_drag_content_provider_extraction_finished`), via `write_mime_type_async`; write requests that arrive mid-extraction are queued (`pending_writes`). Nothing pumps the main context — dragging never blocks the UI.
- Extraction waits for the archive to be free: if `priv->action != FR_ACTION_NONE` or `priv->dnd_extract_is_running`, `start_extraction` re-arms a 200 ms timeout until a 60 s deadline (`start_deadline`, set on first busy hit, not in `schedule_start`). Only one DnD extraction runs at a time; the window keeps a ref in `priv->dnd_extract_provider` and clears the running flag in `fr_window_dnd_extraction_finished`.
- `archive_extraction_ready_cb` matches a completed extraction to the provider by comparing the destination against `dnd_extract_provider->tmp_dir_path` (not the `drag_tmp_dir`/`drag_old_tmp_dir` slots — a new drag may have taken either slot). A dir no longer in the current slot is moved to `drag_old_tmp_dir` and the 60 s cleanup timeout is re-armed.
- Do not break the deferred temp-dir cleanup: `drag_old_tmp_dir` is kept because the drop target copies the files only after the drop. `rotate_drag_tmp_dir` (used by drag-end, drag-cancel and `set_tmp_dir`) keeps the old slot occupied while an extraction runs; `drag_cleanup_timeout_cb` returns `G_SOURCE_CONTINUE` (without losing its id) while `dnd_extract_is_running` or `clipboard_extract_is_running`.
- The provider's `GTask`s must not take the provider as source object (`g_task_new (NULL, ...)`), or pending writes would ref the provider and block its finalize forever.
- An encrypted archive without a stored password shows its password dialog from `start_extraction`; cancelling it calls `fr_window_dnd_extraction_finished (window, TRUE)` from `dlg-ask-password.c`.
- The system `/usr/bin/file-roller` (stock Arch 44.7) does NOT have this feature — test the locally built `build/src/file-roller`.
- The working tree may carry uncommitted WIP in `src/fr-window.c` (drops originating from the archive's own temp dir).

## Architecture
- `HACKING` documents the format plugin model: `FrArchive` subclasses for libarchive-backed formats, `FrCommand` subclasses for external-tool formats, all registered in `src/fr-init.c`.
