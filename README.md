# hyprgrid

Hyprland helper scripts for spawning terminal grids — tiled matrices of floating terminal windows sized and positioned to fit the active monitor's usable area.

![screenshot](20260617_19h15m51s_grim.png)

## Architecture

```
bin/
├── grid          # generic N×M terminal grid
├── grid-ssh      # SSH-terminal grid, auto-balanced to ~16:9
├── grid-rpc      # host-list wrapper: filters example-rpc-* hosts → grid-ssh
└── grid-drpc     # host-list wrapper: filters *.drpc hosts → grid-ssh
```

All four scripts are standalone Bash programs. `grid-rpc` and `grid-drpc` are thin wrappers that parse a host inventory file and delegate to `grid-ssh`. The two core engines — `grid` and `grid-ssh` — share the same geometry machinery but differ in how they determine grid dimensions and what command each cell spawns.

### Geometry engine (shared by `grid` and `grid-ssh`)

Both core scripts use identical `load_reserved_area` and `grid_rule` functions:

1. **`load_reserved_area`** — queries the active Hyprland monitor via `hyprctl -j` and `jq` to read the reserved area (panels, bars, gaps) so the grid fits within the usable region.
2. **`grid_rule`** — emits a Hyprland window-rule string of the form `[float; size W H; move X Y]`, computing per-cell dimensions and offsets from the reserved-area inset, column count, and row count.
3. **`spawn`** (or `spawn_cell`) — calls `hyprctl dispatch exec "<rule> <terminal-cmd>"` and sleeps `$DELAY` seconds to give the compositor time to map each window before the next one lands.

Windows are floated individually via `exec` rules — no reliance on Dwindle/Master split history, and no persistent changes to compositor settings.

---

## Script specifications

### `bin/grid`

**Purpose:** spawn an arbitrary N×M grid of terminal windows.

**Signature:**

```
grid [cols] [rows] [_workspace] [cmd]
```

| Argument     | Position | Default      | Notes                                      |
|-------------|----------|--------------|---------------------------------------------|
| `cols`       | 1        | `3`          | Columns (positive integer)                  |
| `rows`       | 2        | `3`          | Rows (positive integer)                     |
| `_workspace` | 3        | _(reserved)_ | Placeholder — workspace switching is commented out |
| `cmd`        | 4        | `alacritty`  | Terminal emulator command                   |

**Grid logic:**
- Grid dimensions are user-specified (`COLS × ROWS`).
- Iterates `idx` from `0` to `COLS * ROWS - 1`.
- Cell column = `idx % COLS`, row = `idx / COLS`.
- Every cell gets the same `$CMD`.

**Input validation:**
- `cols` and `rows` are checked against `^[1-9][0-9]*$` (positive integer, no leading zero).

**Known rough edge:** the workspace argument persists at position 3 for backward compatibility, but workspace switching (`hyprctl dispatch workspace`) is commented out. The terminal command is therefore argument 4, not 3.

---

### `bin/grid-ssh`

**Purpose:** spawn one SSH terminal per host, auto-balanced into a grid that stays close to 16:9 while minimizing empty panes.

**Signature:**

```
grid-ssh host1 host2 ...
```

**Grid-dimension selection algorithm:**

1. Enumerate every possible row count from `1` to `N` (the number of hosts).
2. For each candidate `(rows, cols = ceil(N / rows))`:
   - Compute `unused = rows × cols - N` (empty panes).
   - Compute the aspect ratio `cols / rows`.
   - Score the candidate by `diff = |ratio - 16/9|`.
3. Pick the candidate with the **smallest `diff`**. On tie (within `1e-6`), pick the one with **fewer unused panes**.
4. If no candidates exist (`N == 0`), exit with a usage message.

**Cell spawning:**
- Hosts fill cells `0` through `N-1` with `alacritty --command ssh <host>`.
- Remaining cells `N` through `COLS * ROWS - 1` get a plain `alacritty` (no SSH).

**Host quoting:** `printf -v host_q '%q'` is used to shell-escape hostnames before embedding them in the `--command` argument.

---

### `bin/grid-rpc`

**Purpose:** spawn an SSH grid for hosts matching the `example-rpc-` pattern.

**Signature:**

```
grid-rpc
```
(no arguments — reads `$HOME/.ssh/hosts`)

**Pipeline:**

```
awk '/^[[:space:]]*#/ { next } /example-rpc-/ { print $2 }' "$HOST_FILE" \
    | sort \
    | xargs -r "$SCRIPT_DIR/grid-ssh"
```

- Skips comment lines.
- Matches lines containing `example-rpc-`, extracting the second whitespace-delimited field as the hostname.
- Sorts alphabetically.
- Passes all matched hosts to the repo-relative `grid-ssh` via `xargs`.

---

### `bin/grid-drpc`

**Purpose:** spawn an SSH grid for hosts whose names end in `drpc.`.

**Signature:**

```
grid-drpc
```
(no arguments — reads `$HOME/.ssh/hosts`)

**Pipeline:**

```
awk '/^[[:space:]]*#/ { next } /drpc.$/ { print $2 }' "$HOST_FILE" \
    | sort \
    | xargs -r "$SCRIPT_DIR/grid-ssh"
```

Same structure as `grid-rpc`, but the awk pattern matches `.drpc` at end-of-line rather than `example-rpc-` anywhere in the line.

Both wrappers resolve `grid-ssh` relative to their own directory (`$SCRIPT_DIR`) so they work regardless of `$PWD`.

---

## Host inventory file format

`grid-rpc` and `grid-drpc` read `$HOME/.ssh/hosts`. The expected format is whitespace-delimited fields, where the second field is a hostname. Lines starting with `#` (with optional leading whitespace) are ignored.

Example:

```
# production nodes
192.168.1.10    example-rpc-us-east
192.168.1.11    example-rpc-eu-west
# disaster recovery
10.0.0.5        nyc-backup.drpc.
10.0.0.6        sfo-backup.drpc.
```

---

## Requirements

| Dependency   | Used by           | Purpose                              |
|-------------|-------------------|--------------------------------------|
| `bash`       | all               | Arrays, arithmetic, `printf %q`      |
| `hyprctl`    | `grid`, `grid-ssh`| Window placement, monitor geometry   |
| `jq`         | `grid`, `grid-ssh`| Parse `hyprctl -j` JSON output       |
| `awk`        | `grid-rpc`, `grid-drpc` | Parse host inventory file     |
| `sort`       | `grid-rpc`, `grid-drpc` | Sort hostnames              |
| `xargs`      | `grid-rpc`, `grid-drpc` | Pass host list to grid-ssh   |
| `alacritty`  | all (default)      | Terminal emulator                    |
| `ssh`        | `grid-ssh`         | SSH client                           |

---

## Environment

- **OS:** Linux
- **Compositor:** Hyprland (Wayland)
- **Monitor reservation:** scripts read `hyprctl -j monitors` reserved area and inset the grid accordingly — works with waybar, eww, or any panel that reports reservation through `wlr-layer-shell`.

---

## Internal constants

| Constant        | File(s)            | Value       | Meaning                                  |
|----------------|--------------------|-------------|------------------------------------------|
| `DELAY`         | `grid`, `grid-ssh` | `0.18`      | Seconds between spawns; tune if windows fail to map in time |
| `TARGET_RATIO`  | `grid-ssh`         | `1.7777778` | 16/9; target aspect ratio for grid dimensions |
| `RESERVED_*`    | `grid`, `grid-ssh` | `0` (init)  | Populated at runtime from `hyprctl`; inset from monitor edges |

---

## Testing (static)

```bash
# Syntax check
bash -n bin/grid bin/grid-ssh bin/grid-rpc bin/grid-drpc

# Lint
shellcheck bin/grid bin/grid-ssh bin/grid-rpc bin/grid-drpc
```

For behavior tests, stub `hyprctl` and `sleep` in the shell to inspect generated `exec` rules without spawning real windows.

---

## Notes

- Windows are placed as **floating** via Hyprland window rules — they do not participate in tiling layout. This produces a clean, regular matrix independent of split history.
- No compositor settings are mutated. If a future change needs to toggle a runtime option, save the pre-run value and restore it through a `trap EXIT` handler.
- The `grid` script's argument 3 (workspace) is a reserved placeholder; workspace switching code is commented out.
