# hyprgrid — Claude Agent Instructions

## Project

Bash scripts that spawn literal N×M tiled terminal grids in Hyprland via dwindle tree manipulation. No floating windows.

## Key Architecture

- **`grid`** — core engine. No `order()` or `leaf[]` tracking needed: always split position 0 (leftmost/topmost window). After `resizeactive`, focus returns to pos 0 — zero navigation.
- **`grid-ssh`** — same algorithm + auto-balanced grid dimensions (prioritize fewest empty cells, then closest to 16:9).
- **`grid-rpc`/`grid-drpc`** — pipe hostnames from `$HOME/.ssh/hosts` to grid-ssh.

## Hyprland 0.55 Findings (DO NOT FORGET)

| Issue | Detail |
|---|---|
| `splitratio` | Removed. Use `resizeactive` with integer percentages. |
| `misc:focus_on_activate` | Broken. New windows always steal focus. |
| Decimal percentages | Ignored by `resizeactive`. Must use integers (e.g., `-33%` not `-33.3%`). |
| Fraction resize | `resizeactive -0.33 0` does nothing. Only `%` works. |
| Minimum sleep | `S=0.11` minimum. Fails at `S=0.10`. Default `S=0.15` (~30% margin). |
| `activewindow.addr` | Always `null` in 0.55. Cannot track focus by address. |
| `movefocus l` after resize | Broken (addr null, no effect). |
| `movefocus r` wrap | Works from non-rightmost. From rightmost: unreliable after resize. |
| Focus after resize | With `S≥0.11`: focus reliably on LEFT/TOP window after `resizeactive`. |

## Algorithm (zero-navigation)

```bash
# Always split position 0 (leftmost/topmost).
# L = N - c + 1 (remaining in first window), R = 1.
# After resizeactive, focus returns to pos 0.
for ((c=2; c<=N; c++)); do
    L=$((N - c + 1)); R=1
    layoutmsg preselect r
    exec "$CMD"; sleep "$S"
    resizeactive (shrinks right window so left = L/(L+R))
done
```

All N columns/rows end up at exactly 1/N of total. Works for any N.

## Bash Pitfalls

- **`local` in bash shadows variables in called functions.** `build_dim` uses `local N` which overrides the global `N` (host count) inside `spawn()`. Rename to `local count`.
- **`1e9` not valid in bash arithmetic.** Use `999999` instead.
- **`replace_all` with `sed` is dangerous** — `S=1` → `S=0.15` also hit `ROWS=1` → `ROWS=0.15`.

## Working Rules

- `S` is configurable at script top. Default: `0.15`.
- Every `hyprctl dispatch` must have `sleep "$S"` after it. No exceptions.
- Windows are tiled — never use `[float]` rules.
- Grid runs on current workspace. Aborts if non-floating windows exist.
- Test via `/tmp/grid-run.sh` (never edit, saves ws, switches to 6, cleans, runs test, screenshots, returns) + `/tmp/grid-test.sh` (edit this, defines what to run).
- Only workspace 6 for testing. Kill everything there before each test.
- Return to original workspace immediately after screenshot.
- `input:follow_mouse` must be 0 during build, restored to 1 after.
- `dwindle:force_split 2`, `preserve_split true`, `default_split_ratio 1`.

## Tested Layouts

2×2, 3×2, 4×2, 5×2, 3×3, 6×6 — all equal within 5px.

