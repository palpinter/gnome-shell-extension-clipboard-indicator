# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Clipboard Indicator — a GNOME Shell extension (UUID `clipboard-indicator@tudmotu.com`) written in GJS using ES modules (GNOME 45+ extension API). Supported shell versions are listed in `metadata.json` (currently 46–51). No build step for JS, no package manager, no test suite, no linter config.

## Commands

- `make` — compile translations (`.po` → `.mo`) and GSettings schema (`schemas/gschemas.compiled`)
- `make install` — build and copy into `~/.local/share/gnome-shell/extensions/clipboard-indicator@tudmotu.com/`
- `make nested-session` — launch a nested Wayland GNOME Shell for manual testing (enable the extension inside it with `gnome-extensions enable clipboard-indicator@tudmotu.com`)
- `make update-po-files` — regenerate `clipboard-indicator.pot` from `*.js` and merge into every `locale/*/LC_MESSAGES/*.po`
- `make bundle` — produce `bundle.zip` for extensions.gnome.org

Compiled artifacts (`*.mo`, `schemas/gschemas.compiled`) are committed to git, so rerun `make` after changing `.po` files or the schema. Logs from the extension go to the shell journal (`journalctl -f -o cat /usr/bin/gnome-shell`); prefs logs come from the `gjs` prefs process.

## Architecture

- `extension.js` — almost all runtime logic. `ClipboardIndicatorExtension` (the `Extension` subclass) just creates/destroys `ClipboardIndicator`, a `PanelMenu.Button` subclass that owns the popup menu, history/favorites sections, search, notifications, keybindings, private mode, excluded apps, interval clearing, and tag/edit/image-preview dialogs.
- `registry.js` — persistence. `Registry` reads/writes the history as JSON to `~/.cache/clipboard-indicator@tudmotu.com/registry.txt`; image entries are stored as separate files in the same dir, named by the hash of their bytes. If the registry exceeds the `cache-size` setting (MB) on load it is moved to `registry.txt~` and history starts empty. `ClipboardEntry` wraps mimetype + `GLib.Bytes` + favorite flag + optional tag.
- `prefs.js` — Adw/GTK4 preferences window (runs in a separate process; cannot import Shell/St/Clutter modules).
- `constants.js` — `PrefsFields`, the map from JS names to GSettings keys. Shared by `extension.js`, `prefs.js`, `registry.js`.
- `keyboard.js` — virtual keyboard device used to paste: after setting the clipboard it sends Shift+Insert, or Ctrl+Shift+Insert when the focused input's content purpose is TERMINAL.
- `confirmDialog.js` — modal confirm dialog wrapper (`DialogManager`).
- `schemas/org.gnome.shell.extensions.clipboard-indicator.gschema.xml` — all settings.

### Key patterns

- **Settings are mirrored into module-level `let` globals** at the top of `extension.js` (e.g. `MAX_REGISTRY_LENGTH`, `PRIVATEMODE`, `CACHE_ONLY_FAVORITE`). `_fetchSettings()` refreshes them; `_onSettingsChange()` reacts to changes. Adding a setting means: schema key → `PrefsFields` entry → global + `_fetchSettings()` line → UI in `prefs.js` → `make` to recompile the schema.
- **Clipboard tracking** listens to `Meta.Selection` `owner-changed` (`_setupListener`) → `_refreshIndicator()`, which skips private mode, excluded apps (by focused window `wm_class`), and duplicates (selecting/moving the existing item instead).
- **Menu items are the source of truth**: `clipItemsRadioGroup` holds menu items each carrying an `.entry` (`ClipboardEntry`); the registry is rewritten from this list via `_updateCache()` / `#addToCache()` after mutations.
- Signal connections and timeouts must be cleaned up in `destroy()`/`disable()` (GNOME extension review requirement).
- User-visible strings use `_()` (gettext domain `clipboard-indicator`); translations live in `locale/<lang>/LC_MESSAGES/`.

## Contribution notes (from README)

Upstream (Tudmotu) does not accept unsolicited PRs except translation updates; feature work should be discussed in an issue first. Releases loosely track GNOME's twice-yearly cycle.
