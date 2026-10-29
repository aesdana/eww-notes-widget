# eww notes widget

A notes panel for [eww](https://github.com/elkowar/eww) that slides in from the left screen edge. It holds notes and to-do lists. You can tag them, search them by keyword or `#tag`, pin them and copy them to the clipboard. Each note is saved as a small Markdown file.

The widget needs a **patched eww**. The patch adds a multi-line `textarea` widget and selectable labels, and it makes text fields pass their text to commands on stdin. The script reads text from there, so nothing you type ever becomes part of a shell command line.

## Files

| File | What it is | Goes to |
|---|---|---|
| `eww-textarea.patch` | the eww patch | applied to the eww source |
| `notes.yuck` | the panel, its bar button and the window | `~/.config/eww/` |
| `notes.scss` | its styles | `~/.config/eww/` |
| `scripts/notes` | Python script: storage, search, everything the buttons do | `~/.config/eww/scripts/` |
| `notes-base.yuck` | what `notes.yuck` needs from a full config: the `eww` path variable, the `ic` icon set, the `look_seg` button | `~/.config/eww/` |
| `notes-base.scss` | what `notes.scss` needs from a full config: colours, fonts, the 3D mixins, the panel frame, the List / New buttons | `~/.config/eww/` |

## Requirements

- An X11 session. The widget is built and used on X11; see [Wayland](#wayland) for what else that needs.
- `python3` (standard library only).
- `xclip` for the copy button: `sudo apt install xclip`.
- The **Font Awesome 7 Free** desktop fonts, Solid and Regular. They are used for the icons; see step 2.
- Optional: the **Roboto Condensed** font (`sudo apt install fonts-roboto`). Without it, GTK uses its default font.

## 1. Build eww with the patch

Build tools and libraries (Debian / Ubuntu):

```bash
sudo apt install git build-essential pkg-config libgtk-3-dev libdbusmenu-gtk3-dev
```

Rust: install it with [rustup](https://rustup.rs). eww's `rust-toolchain.toml` pins the Rust version, and rustup downloads that version on the first build. A distribution `cargo` also works if it is recent; 1.95 builds it.

Get eww at the commit the patch was made for, apply the patch, and build the X11 version:

```bash
git clone https://github.com/elkowar/eww.git ~/src/eww
```
```bash
cd ~/src/eww && git checkout -b notes 48f5aa8 && git apply /path/to/eww-notes-widget/eww-textarea.patch
```
```bash
cargo build --release --no-default-features --features x11
```

Install the binary. The widget expects it at `~/bin/eww`; to use another place, see [The eww binary](#the-eww-binary).

```bash
install -Dm755 target/release/eww ~/bin/eww
```

The patch also applies to newer eww commits as long as `crates/eww/src/widgets/mod.rs` and `widget_definitions.rs` haven't changed much. `git apply --check /path/to/eww-textarea.patch` tells you before you change anything.

The patch doesn't break existing configs. `input` widgets work as before; their text is now *also* written to the command's stdin.

## 2. Install the icon font

Download "Free for Desktop" from [fontawesome.com/download](https://fontawesome.com/download), then:

```bash
mkdir -p ~/.local/share/fonts && unzip -j ~/Downloads/fontawesome-free-7.*-desktop.zip '*/otfs/*' -d ~/.local/share/fonts/fontawesome7 && fc-cache -f
```

Check that it was found. You should see `Font Awesome 7 Free` with the styles Solid and Regular:

```bash
fc-list | grep 'Font Awesome 7'
```

## 3. Copy the files

```bash
mkdir -p ~/.config/eww/scripts && cd /path/to/eww-notes-widget && cp notes.yuck notes.scss notes-base.yuck notes-base.scss ~/.config/eww/ && cp scripts/notes ~/.config/eww/scripts/ && chmod +x ~/.config/eww/scripts/notes
```

## 4. Add it to your eww config

**`~/.config/eww/eww.yuck`**: include both files, base first.

```lisp
(include "./notes-base.yuck")
(include "./notes.yuck")
```

**`~/.config/eww/eww.scss`**: import both, base first.

```scss
@import "notes-base";
@import "notes";
```

Then do one of these, or both:

- **Bar button.** If you have an eww bar, put `(notesw)` where the button should go.
- **Keyboard shortcut.** Bind a key to `~/.config/eww/scripts/notes toggle` in your desktop's keyboard settings.

The eww daemon must be running for either one. If eww doesn't run your bar yet, start it with `~/bin/eww daemon`, for example from your session's autostart.

**No eww config yet?** An `eww.yuck` and `eww.scss` with just the lines above are enough. The panel then opens with the keyboard shortcut.

**Clashes with your config.** `notes-base.yuck` defines the names `eww`, `ic` and `look_seg`. If your config already uses one of them, delete it from `notes-base.yuck`. Your definition must then provide the same things: the icon names used in `notes.yuck`, and `look_seg` with the same arguments. `notes-base.scss` defines the mixins `raised`, `hover` and `pressed`. If you have mixins with those names, import `notes-base` *before* your own files.

## 5. Use it

```bash
~/.config/eww/scripts/notes toggle
```

- **List**: search by keyword and `#tag`. Click a tag chip to filter by it. Use the buttons under each note to add items, edit, copy, edit tags, pin or delete.
- **New**: choose Note or To-do list, type, and press Ctrl+Enter (in a note) or Enter (in the title or tags field).
- In a multi-line field, Enter starts a new line and **Ctrl+Enter saves**.

Run `scripts/notes` without arguments to list all its commands.

## Configuration

### Where the notes are stored

By default, everything is in `~/Documents/ewwnotes`:

```
~/Documents/ewwnotes/notes/       one Markdown file per note (20261025-155500.md …)
~/Documents/ewwnotes/.data/       state.json (search, unsaved draft), trash/ (deleted notes)
```

To store them elsewhere, edit the lines near the top of `scripts/notes`:

```python
ROOT = f'{HOME}/Documents/ewwnotes'
NOTES = f'{ROOT}/notes'
DATA = f'{ROOT}/.data'
```

- Change `ROOT` to move both folders, for example `ROOT = f'{HOME}/Nextcloud/notes'`.
- Change `NOTES` or `DATA` on their own to split them. For example, keep the notes in a synced folder and the state locally with `DATA = f'{HOME}/.local/state/ewwnotes'`.

The folders are created on first use. To move existing notes, close the panel, move the folders, then change the lines.

The delete button's tooltip in `notes.yuck` names the trash path (`kept in ~/Documents/ewwnotes/.data/trash`). Update that text too if you move `DATA`.

A note file looks like this, and you can edit it by hand:

```
---
kind: todo
tags: home
pinned: false
created: 2026-10-25 16:02
---
# Groceries
- [ ] milk
- [x] bread
```

### The eww binary

The widget calls eww in two places:

- `notes-base.yuck`: `(defvar eww "$HOME/bin/eww")`, used by the List / New buttons.
- `scripts/notes`: `EWW = os.environ.get('EWW_BIN', f'{HOME}/bin/eww')`. You can either set `EWW_BIN` in the environment the eww daemon starts with, or edit the line.

If your eww is somewhere else, change both, or link it:

```bash
mkdir -p ~/bin && ln -s "$(command -v eww)" ~/bin/eww
```

### Size and position

The `notes` window at the end of `notes.yuck` is set up for a 1080 px high screen with a 46 px bar at the top:

```lisp
:geometry (geometry :x "0px" :y "46px" :width "340px" :height "1028px" :anchor "top left")
```

- `:y`: space above the panel. Use your bar's height plus a gap, or `0px` without a bar.
- `:height`: screen height minus `:y` minus a gap at the bottom. For example, `1440 - 46 - 6 = 1388px`.
- `:width`: if you change it, also change `:width 340` on the panel's box in `notes_panel`.
- `:monitor`: `'["<primary>", 0]'` means the primary monitor, or the first one if no monitor is primary.

### Colours and fonts

`notes-base.scss` contains the colours and fonts of my theme (Catpine), all set with `!default`. To override them, set the variables before the import in your `eww.scss`:

```scss
$accent: #89b4fa;
$font-text: "Inter";
@import "notes-base";
@import "notes";
```

## Wayland

The patch builds on Wayland too. Drop `--no-default-features --features x11`, and install `libgtk-layer-shell-dev` as well. The rest has only been used on X11:

- `xclip` in `scripts/notes` (`notes copy`) has to become `wl-copy`.
- The `notes` window's `:wm-ignore` and `:windowtype` are X11 options. On Wayland, set `:focusable "ondemand"` and remove the two.
