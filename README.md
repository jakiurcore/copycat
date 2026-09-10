# copycat

Paste clipboard content directly to disk, like pasting an image into Signal —
except it lands as a file instead of a chat message.

## Status

- **Images:** done. `bin/copycat` reads whatever image is on the Wayland
  clipboard (`image/png`, `jpeg`, `webp`, `gif`, `bmp`, `tiff`, `svg`) and
  writes it to disk.
- **Text / files / other mime types:** not yet — planned next.

## How it decides where to save

1. Looks at the currently focused Hyprland window.
2. Finds its shell's working directory via `/proc/<pid>/cwd` (so if you're
   sitting in a terminal at `~/notes`, the image lands there).
3. If that can't be determined (focused window isn't a terminal), falls back
   to `~/Pictures/Clipboard`.

## Usage

- Hotkey: `SUPER + SHIFT + V` (bound in `~/.config/hypr/bindings.lua`) —
  copy an image anywhere, focus a terminal, hit the hotkey.
- Or run directly: `./bin/copycat`

Requires: `wl-clipboard`, `hyprctl`, `jq`, `notify-send` (all already on this
machine).

## Next up

Extend `save()`-style dispatch in `bin/copycat` to also handle:
- `text/plain` → write a `.txt` file
- `text/uri-list` (copied files in a file manager) → copy the referenced
  files into the target directory
