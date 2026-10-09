# hyprgrid — Claude Agent Instructions

## Project

Bash scripts that spawn literal N×M tiled terminal grids in Hyprland via dwindle tree manipulation. No floating windows.

## Key Architecture

- **`grid`** — core engine. Bands-first build: ROWS equal full-width bands, then COLS equal columns per band. Focus tracked by window address.
- **`grid-ssh`** — same build + per-cell spawn commands. Auto-balanced dimensions (minimize empty cells, then closest to 16:9). Host *i* lands at row-major cell (`i/COLS`, `i%COLS`); cells past the host list spawn a plain terminal.
- **`grid-hosts`** — `grid-hosts [-n] PATTERN [FILE]`: Host aliases from an ssh config (default `~/.ssh/config`) matching an awk regex → grid-ssh. Only `Host` lines are read, wildcard aliases skipped, output sorted; `-n` lists without spawning. Personal host groups live in private wrappers outside the repo — never hardcode inventories or hostnames here.

## Hyprland 0.56 + Lua config (DO NOT FORGET)

The user runs **hyprland.lua**, not hyprlang. hyprctl is now Lua-evaluated — every legacy syntax breaks:

| Legacy (0.55) | 0.56 replacement |
|---|---|
| `hyprctl keyword …` | `hyprctl eval 'hl.config({…})'` (keyword fails: "use eval") |
| `hyprctl dispatch exec "cmd"` | `hyprctl dispatch 'hl.dsp.exec_cmd("cmd")'` (runs via bash -c) |
| `dispatch layoutmsg "preselect r"` | `hl.dsp.layout("preselect r")` / `"preselect d"` |
| `dispatch movefocus r` | `hl.dsp.focus({ direction = "right" })` |
| focus by window | `hl.dsp.focus({ window = "address:0x…" })` — **must** prefix `address:`; bare `0x…` or `{window="0x…"}` → "window not found" |
| `dispatch resizeactive "-33%" 0` | `hl.dsp.window.resize({ x = <pixels>, y = <pixels>, relative = true })` |

Dispatch semantics to remember:

- Any bare word in a dispatch is parsed as Lua → `')' expected near 'X'`. All strings must be quoted (helper `lua_str`). A dispatch is just `hl.dispatch(<your-lua-expr>)`.
- **One dispatch = exactly one operation.** Dispatcher constructors (`hl.dsp.*`) do NOT execute on construction; only the single value returned by the expression is run, so you cannot chain ops in one `hyprctl dispatch` (a `;` at top level is a parse error, and `hl.dsp.exec_cmd("a"), hl.dsp.exec_cmd("b")` spawns only one window). Each op is its own hyprctl round-trip.
- `window.resize` numeric args **without** `relative` are ABSOLUTE sizes (negative → "Invalid size"). With `relative = true` it drags the split border by a pixel delta; positive shrinks the freshly-spawned right/bottom child. 0.55's integer-percent behavior is gone — deltas are computed in px.
- **Focus does NOT auto-return to pos 0 after a resize** (unlike 0.55 `resizeactive`). You must `focus({window="address:…"})` the persistent seed/band window before every split.
- **Pacing — no fixed sleeps.** `focus`/`preselect` are synchronous and need no wait. Only two steps are genuinely async, and both are **polled adaptively**: after `exec` wait for the window to map + grab focus (`wait_map`), after `window.resize` wait until geometry is stable across two reads (`wait_stable`). Set `animations.enabled=false` for the build so sizes snap (a fixed sleep was only needed to out-wait open animations). ~4× faster: 3×3 in ~1.9 s, 4×4 ~3.4 s, 6×6 ~8 s, geometry ≤3px.
- **Restore config-file values, never live ones.** On exit the scripts re-apply the `input:follow_mouse` / `animations.enabled` values *parsed from `hyprland.lua`/`hyprland.conf`* (with Hyprland built-in defaults as fallback), via a `trap`. Do NOT snapshot with `getoption` and restore that: another tool or an interrupted run can mutate the live value, and restoring it would cement the drift (a test harness once left `animations.enabled=true` live, then scripts kept re-applying `true` even though the config says `false`). Parsing keeps the end state deterministic and self-healing.

## Equal-grid build (validated 2×2…5×3, 6×6; ≤3px on 0.56.2)

```bash
# dpct = (2R/(L+R) - 1)*100   (negative)
# delta = round(size * -dpct / 100)   -- width for columns, height for rows
#   ROWS bands: split seed downward ROWS-1 times -> equal bands
#   then per band: split rightward COLS-1 times    -> equal columns
# refocus the seed / band window by address before EVERY split.
```

**Split↔visual-cell mapping** (deterministic; dwindle nests left) — a split with remaining count `L` puts its new window at visual index `L`:

- band split `r` (2..ROWS) → cell row `ROWS - r + 1`, col 0. Creation fills rows `0, ROWS-1, …, 1`.
- column split `c` (2..COLS) within band row → cell col `COLS - c + 1`. Creation fills cols `0, COLS-1, …, 1`.
- Full creation order: seed `(0,0)`; bands `r=2..ROWS`; then per band (in creation order) columns `c=2..COLS`.
- grid-ssh precomputes this as a per-spawn command array and advances a pointer (`next_cmd`) so host *i* is placed at row-major cell *i*.

## Bash Pitfalls

- **`$()` runs in a subshell — mutations are lost.** `next_cmd(){ sp=$((sp+1)); echo …; }` used as `"$(next_cmd)"` never advances `sp` (every window got cmds[0]). Advance a global inside a plain call, then read the variable.
- **`local` in bash shadows variables in called functions.** `build_dim` uses `local N` which overrides the global `N` (host count) inside `spawn()`. Rename to `local count`.
- **`1e9` not valid in bash arithmetic.** Use `999999` instead.
- **`replace_all` with `sed` is dangerous** — `S=1` → `S=0.15` also hit `ROWS=1` → `ROWS=0.15`.
- lua strings: escape `\` then `"` (helper `lua_str`).

## Working Rules

- No fixed sleeps. Sync ops (`focus`, `preselect`) are un-paced; async ops are adaptively polled (`wait_map` after exec, `wait_stable` after resize). Keep the generous poll caps as a safety net.
- Windows are tiled — never use `[float]` rules.
- Grid runs on current workspace. Aborts if non-floating windows exist.
- Test on workspace 6 only; kill everything there before each test; return to original workspace immediately. Test harnesses are throwaway scripts (keep them out of the repo): save the original workspace, close all ws6 windows on exit, then restore it. Check grid-ssh placement by spawning `alacritty --title HOST<i>` instead of ssh and reading titles from `hyprctl -j clients`.
- `input.follow_mouse` must be 0 during build, restored to the config-file value after (via `hyprctl eval 'hl.config({…})'`).
- `hl.dsp.exec_cmd` runs the string through `bash -c` → grid-ssh can pass `alacritty --command ssh <host>` unquoted.

## Tested Layouts

2×2, 2×3, 3×2, 3×3, 3×5, 4×2, 4×4, 5×3, 6×6 — equal within 5px with the adaptive-pacing build. grid-ssh 3×2 host placement verified by window title (host *i* at row-major cell *i*).

