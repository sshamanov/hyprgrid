# AGENTS.md

## Project

Bash scripts for spawning literal tiled terminal grids in Hyprland via dwindle tree manipulation — no floating windows.

- `bin/grid` — core engine: ROWS equal full-width bands, then COLS equal columns per band.
- `bin/grid-ssh` — same build, one SSH host per cell (row-major), auto-balanced to ~16:9.
- `bin/grid-hosts` — Host aliases from an ssh config matching a regex → grid-ssh.

Full Hyprland 0.56 / Lua API notes, the build algorithm and known pitfalls are in `CLAUDE.md` — read it before changing the scripts.

## Working Rules

- Keep changes small and script-focused. Bash arrays, arithmetic, functions.
- Hyprland 0.56 with a Lua config: use `hyprctl dispatch 'hl.dsp.…'` and `hyprctl eval 'hl.config({…})'`; legacy `keyword`/dispatch syntax fails.
- No fixed sleeps: poll for window map after exec and for stable geometry after resize.
- Windows are tiled (no `float` rules). Runs on the current workspace, which must have no tiled windows.
- `input.follow_mouse` and `animations.enabled` are off during the build and restored to the config-file values on exit.
- Never commit host inventories, hostnames, IPs, credentials or screenshots of real sessions.
- Do not run scripts interactively unless the user asks; test only on workspace 6.

## Expected Environment

- Linux, Hyprland ≥ 0.56 (Lua config), `hyprctl`, `jq`, `awk`, `alacritty`, `ssh`
