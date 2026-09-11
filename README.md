# copycat

Paste clipboard content directly to disk, like pasting an image into Signal —
except it lands as a file instead of a chat message.

## Status

- **Images:** done. `bin/copycat` reads whatever image is on the Wayland
  clipboard (`image/png`, `jpeg`, `webp`, `gif`, `bmp`, `tiff`, `svg`) and
  writes it to disk.
- **Text / files / other mime types:** not yet — planned next.

## Supported systems

| Compositor | Hotkey wiring | Focused-terminal cwd detection |
|---|---|---|
| Hyprland (Omarchy / Arch) | yes (`~/.config/hypr/bindings.lua`) | yes (`hyprctl`) |
| Sway | yes (`~/.config/sway/config`) | yes (`swaymsg`) |
| COSMIC (Pop!_OS) | yes (see below) | no — falls back to `~/Pictures/Clipboard` |
| GNOME / KDE / other | not automated — bind `copycat` to a key manually | no |

| Package manager | Distros |
|---|---|
| `pacman` (`omarchy pkg add` if present) | Arch, Omarchy |
| `apt` | Debian, Ubuntu, **Pop!_OS** |
| `dnf` | Fedora |

## How it decides where to save

- **Hotkey / bare CLI** (`copycat`): looks at the currently focused window
  (via whichever of `hyprctl`/`swaymsg` is available — see table above),
  finds its shell's working directory via `/proc/<pid>/cwd` (so if you're
  sitting in a terminal at `~/notes`, the image lands there). Falls back to
  `~/Pictures/Clipboard` if that can't be determined (unsupported
  compositor, or the focused window isn't a terminal).
- **Explicit target** (`copycat <dir>`): saves straight into `<dir>`,
  bypassing the detection above. This is how the Nautilus integration below
  gets it right for file-manager windows.

### Pop!_OS / COSMIC specifics

COSMIC has no equivalent of `hyprctl` for querying the focused window, so
on Pop!_OS the hotkey always saves to `~/Pictures/Clipboard` rather than a
terminal's cwd. The hotkey itself *is* wired up, though — `install.sh`
writes a `Spawn(...)` entry directly into COSMIC's own keybinding config
(`~/.config/cosmic/com.system76.CosmicSettings.Shortcuts/v1/custom`, RON
format — reverse-engineered from the `cosmic-comp`/`cosmic-settings-daemon`
source, not from a live COSMIC session, since this was built on an
Arch/Hyprland machine). It should apply live; if the hotkey doesn't respond
right after install, log out and back in once.

COSMIC Files (COSMIC's default file manager) has no scripts/extensions
mechanism yet, so there's no Nautilus-Scripts equivalent to install there.

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

- Detects your package manager (`pacman`/`omarchy`, `apt`, or `dnf`) and
  installs missing dependencies (`wl-clipboard`, `jq`, and the platform's
  `notify-send` package) through it.
- Symlinks `bin/copycat` to `~/.local/bin/copycat`, so `copycat` works as a
  plain CLI command from any shell.
- Detects your compositor (Hyprland, Sway, or COSMIC, checked in that order)
  and binds `SUPER + SHIFT + V` to copycat in its native config, inside a
  clearly marked auto-generated block:
  - **Hyprland**: `~/.config/hypr/bindings.lua` (`hl.unbind` + `o.bind`),
    validated with `hyprctl configerrors`.
  - **Sway**: `~/.config/sway/config` (`bindsym ... exec`), applied with
    `swaymsg reload`.
  - **COSMIC**: the RON shortcuts file — see the Pop!_OS section above.
  In every case, if that key is already bound to something else, it's
  detected and overridden. Rolls back to a backup automatically if the
  compositor reports the new config as invalid. Re-running never duplicates
  the block; it just replaces it in place.
- Installs the Nautilus "Scripts" entry (see below) if `nautilus` is present.

To use a different key combo: `./install.sh "SUPER + ALT + V"`.

Every step degrades gracefully if its target isn't present: unknown package
manager -> prints what to install manually; no supported compositor ->
hotkey step skipped (bind `copycat` to a key yourself); no `nautilus` ->
script step skipped. The CLI install always happens.

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
