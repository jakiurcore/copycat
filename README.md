# copycat

Paste clipboard content directly to disk, like pasting an image into Signal —
except it lands as a file instead of a chat message.

## Status

- **Images:** done. `bin/copycat` reads whatever image is on the Wayland
  clipboard (`image/png`, `jpeg`, `webp`, `gif`, `bmp`, `tiff`, `svg`) and
  writes it to disk.
- **Text / files / other mime types:** not yet — planned next.

## How it decides where to save

- **Hotkey / bare CLI** (`copycat`): looks at the currently focused Hyprland
  window, finds its shell's working directory via `/proc/<pid>/cwd` (so if
  you're sitting in a terminal at `~/notes`, the image lands there). Falls
  back to `~/Pictures/Clipboard` if that can't be determined (e.g. the
  focused window isn't a terminal).
- **Explicit target** (`copycat <dir>`): saves straight into `<dir>`,
  bypassing the detection above. This is how the Nautilus integration below
  gets it right for file-manager windows.

### Why file managers need a different path

There's no external API to ask an arbitrary GUI window "what folder are you
showing" on Wayland — confirmed by introspecting Nautilus's D-Bus interface
(no queryable location) and its window title (just shows the folder's
basename, e.g. "Downloads" — too ambiguous to guess a full path from safely).
So the hotkey can't reliably target a file manager's open folder.

Nautilus does expose it through its own **Scripts** mechanism, though: a
script placed in `~/.local/share/nautilus/scripts/` shows up under
right-click → *Scripts*, and Nautilus sets `NAUTILUS_SCRIPT_CURRENT_URI` to
the exact folder being viewed when it runs the script. `install.sh` installs
`nautilus-scripts/Paste Clipboard Image Here` there, which resolves that URI
and calls `copycat <dir>` with the real folder — no guessing.

Right-click a folder's background in Nautilus → **Scripts** → **Paste
Clipboard Image Here**.

## Install

```
./install.sh
```

This is idempotent — safe to re-run any time (e.g. after moving the repo, or
just to make sure everything's still wired up). It:

- Installs missing dependencies (`wl-clipboard`, `jq`, `libnotify`) via
  `omarchy pkg add` or `pacman`.
- Symlinks `bin/copycat` to `~/.local/bin/copycat`, so `copycat` works as a
  plain CLI command from any shell.
- Binds `SUPER + SHIFT + V` in `~/.config/hypr/bindings.lua`, inside a
  clearly marked auto-generated block. If that key is already bound to
  something else — an Omarchy default or your own binding — it prints what
  it was and overrides it (`hl.unbind` + `o.bind`, the safe Omarchy pattern).
  Re-running never duplicates the block; it just replaces it in place.
- Validates the Hyprland config after editing (`hyprctl configerrors`) and
  rolls back automatically if anything's wrong.
- Installs the Nautilus "Scripts" entry (see below) if `nautilus` is present.

To use a different key combo: `./install.sh "SUPER + ALT + V"`.

Each step degrades gracefully if its target isn't present: no `hyprctl` ->
hotkey step skipped; no `nautilus` -> script step skipped. The CLI install
always happens.

## Usage

- Hotkey: `SUPER + SHIFT + V` — copy an image anywhere, focus a terminal,
  hit the hotkey. Saves into that terminal's current directory.
- Nautilus: right-click a folder's background → Scripts → **Paste Clipboard
  Image Here**. Saves into the folder you're viewing.
- Or run directly: `copycat` (after install) / `./bin/copycat`, optionally
  with a target directory: `copycat ~/some/folder`.

## Next up

Extend `save()`-style dispatch in `bin/copycat` to also handle:
- `text/plain` → write a `.txt` file
- `text/uri-list` (copied files in a file manager) → copy the referenced
  files into the target directory
