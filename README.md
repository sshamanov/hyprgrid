# hyprgrid

Hyprland scripts for spawning tiled terminal grids — literal N×M matrices of equal-size windows via dwindle tree manipulation. No floating.

![screenshot](20260617_19h15m51s_grim.png)

## Architecture

```
bin/
├── grid          # core: N×M identical terminals
├── grid-ssh      # SSH grid: auto-balanced to ~16:9
├── grid-rpc      # wrapper: example-rpc-* hosts → grid-ssh
└── grid-drpc     # wrapper: *.drpc hosts → grid-ssh
```

All spawn logic lives in `grid`. `grid-ssh` wraps it with host→SSH-command transformation and automatic grid dimension selection.

## How it works

1. **Prep:** switch to workspace 6, clear it, set dwindle options:
   - `force_split 2` — always split right/bottom
   - `preserve_split true` — keep direction after resize
   - `focus_on_activate` broken in Hyprland 0.55, so focus stays on new window
2. **Build columns:** seed first window, then repeatedly split the leftmost window.
   After each split, `resizeactive` shrinks the right window so the left section spans (N-1)/N of the total — all N columns are equal width.
3. **Build rows:** for each column, split the top window repeatedly — all rows equal height.
4. **Restore** `input:follow_mouse`.

No focus navigation needed — after `resizeactive`, focus returns to the left/top window naturally.

Default sleep between actions: `S=0.15` seconds (minimum safe value for Hyprland 0.55; fails at ≤0.10 on 6×6).

Tested layouts: 2×2, 3×2, 4×2, 5×2, 3×3, 6×6 — all equal within 5px.

## Scripts

### `grid [COLS] [ROWS] [CMD]`

Defaults: 3 columns, 3 rows, alacritty.

```bash
grid                # 3×3 alacritty
grid 5 2 foot       # 5×2 foot terminals
grid 4 3 "alacritty -e htop"
```

### `grid-ssh HOST...`

One SSH terminal per host. Grid dimensions auto-chosen to stay close to 16:9. Empty panes get a plain terminal.

### `grid-rpc` / `grid-drpc`

Read hosts from `$HOME/.ssh/hosts`, filter by pattern, pipe to `grid-ssh`.

## Requirements

- Hyprland ≥ 0.55, `hyprctl`, `jq`, `awk`
- `alacritty` (default), `ssh` (grid-ssh)
- Configurable: set `S` (sleep) and `WS` (workspace) at top of scripts

## Testing

```bash
bash /tmp/grid-run.sh    # wrapper: ws save/restore, run test, screenshot
bash /tmp/grid-test.sh   # test logic — edit this
```
