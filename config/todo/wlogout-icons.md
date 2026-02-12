# wlogout Power Menu — Replace Icons

## Status
New — not yet attempted

## Description
The power menu (ctrl+alt+p) icons look bad and need replacing.

## What We Know
- Power menu app: **wlogout**, launched via `~/.config/hypr/scripts/Wlogout.sh`
- User icon directory: `~/.config/wlogout/icons/` (takes priority over system defaults)
- Current icons in user dir: png files (hibernate, lock, logout, power, restart, sleep — plus some
  alternates like `moon_865813.png`, `sleep2.png`)
- System fallback icons: `/usr/share/wlogout/icons/` (hibernate, lock, logout, reboot, shutdown, suspend)
- Layout config: `~/.config/wlogout/layout`
- Styling: `~/.config/wlogout/style.css`

## Fix
Replace the PNG files in `~/.config/wlogout/icons/` with preferred icons.
Names must match what `~/.config/wlogout/layout` references.

Check which names the layout uses before replacing:
```bash
cat ~/.config/wlogout/layout
```

Then drop new PNGs in with matching filenames:
```bash
cp /path/to/new/icon.png ~/.config/wlogout/icons/logout.png
# etc.
```

No restart needed — wlogout reads icons fresh each time it opens.

## Notes
- SVG is not supported by wlogout, must be PNG
- Hover variants follow the `-hover` naming convention (e.g. `logout-hover.png`)
- The JaKooLits setup ships its own icons that override the system ones — editing the
  system path (`/usr/share/wlogout/icons/`) would be overwritten on package updates
