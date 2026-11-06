# appimage-install

A small helper script for [Omarchy](https://omarchy.org/) (and any Arch Linux +
Hyprland setup) that installs AppImages as **first-class desktop apps**, the
same way Omarchy itself installs its own AppImage-based apps (see the
`omarchy-install-t3code-app` script: `/usr/bin/omarchy-install-t3code-app`).

Point it at an AppImage file and it:

1. Verifies the file is a real AppImage (embedded `AI` magic at bytes 8-9)
2. Places it in a **stable, versionless home**: `~/.local/share/<app>/<Name>.AppImage`
   - version numbers are stripped from the file name (`WowUp-CF-2.25.4` → `WowUp-CF`)
   - the name and path never change again after that, so an app with a
     self-updater (e.g. anything using electron-updater) can replace its own
     file **in place** on every update, and the registered launcher keeps
     working forever
3. Extracts the bundled `.desktop` entry and all hicolor icon sizes out of
   the image itself (via `--appimage-extract`)
4. Rewrites the entry for the stable path: quoted absolute `Exec=`
   (preserving app-specific flags like `--no-sandbox %U`), adds `TryExec=`,
   drops `X-AppImage-Version=` and relative `Path=`
5. Registers it with `update-desktop-database`, so it appears in the app
   launcher (Super+Space on Omarchy) like any native app

No AppImage daemon or extra tooling: just `fuse3`, a fixed path, and a
registered desktop entry.

## Install

```bash
# from this repo (the script lives here):
ln -s "$PWD/appimage-install" ~/.local/bin/appimage-install
```

`~/.local/bin` must be on `PATH` (it is on Omarchy by default).

Requirements:

- `fuse3` (`omarchy pkg add fuse3`) — most AppImages mount through fusermount
- `update-desktop-database`, `find`, and ImageMagick (`magick`) help but the
  script degrades gracefully when optional tools are missing

## Usage

```bash
# install (moves the file out of place):
appimage-install ~/Downloads/TradeSkillMaster.AppImage

# install, keep the original file instead:
appimage-install --copy ~/Downloads/TradeSkillMaster.AppImage

# install every *.AppImage sitting in ~/Downloads:
appimage-install

# uninstall (asks for confirmation; id matching is case-insensitive:
# "tradeskillmaster" matches "TradeSkillMaster"):
appimage-install --remove tradeskillmaster

# help:
appimage-install --help
```

Multiple AppImages can be given at once. Non-zero exit on any failure, zero
when everything succeeded.

## Design notes

- **Why a versionless file name?** Self-updaters like electron-updater replace
  the running AppImage in place at the path it knows about (the `APPIMAGE`
  env var set by the AppImage runtime). A name that carries a version would
  change on update and orphan the registered launcher. Publishing/installing
  versionless keeps the path frozen.
- **Why `~/.local/share/<app>/`?** User-writable, survives cleanups of
  `~/Downloads`, and matches Omarchy's own convention for T3 Code.
- **Why extract the desktop entry from the image?** The bundled entry is the
  source of truth for the app's name, categories, and required launch flags;
  rewriting it minimizes drift from upstream changes.
- **Why launch via the desktop entry?** Omarchy's launcher starts apps under
  the systemd user session (`uwsm-app -- gtk-launch ...`) — better resource
  handling and a consistent environment.

## Troubleshooting

- **"error: target not found" with `omarchy pkg add appimagelauncher`:**
  that tool is AUR-only and unmaintained; this script replaces it.
- **App won't launch:** make sure `fuse3` is installed (`pacman -Q fuse3`).
- **Icon missing/wrong in the launcher:** Omarchy's shell (Quickshell) builds
  an icon index from `~/.local/share/icons` on shell start — run
  `omarchy restart shell` after an install if the tile looks stale.
- **`gtk-update-icon-cache` fails with "No theme index file":** the local
  hicolor tree needs an `index.theme`; copy one from the system:
  `cp /usr/share/icons/hicolor/index.theme ~/.local/share/icons/hicolor/`.

## Tested with

- TradeSkillMaster.AppImage (Electron + electron-updater, generic provider)
- WowUp-CF-2.25.4.AppImage (Electron)
- T3 Code.AppImage (upstream equivalent installs it the same way)

## License

MIT - see [LICENSE](LICENSE).
