# PLAN — Telemetry core and the `--once` snapshot (R24)

**Status:** design settled, nothing implemented. Grounded against `MenuBarLoadRunner.swift` and the
launcher on 2026-09-20.

**Lifecycle:** when R24 ships, distill the module boundary and the snapshot contract into
`docs/ARCHITECTURE.md`, drop R24 from `docs/ROADMAP.md`, and `git rm` this file the same commit.

---

## 0. The system after R24

The topology once this is done, and what `docs/ARCHITECTURE.md` § 1 gets redrawn to on landing. Two
entry paths, one set of readers, and nothing shared between the paths except the core.

```
                      +---------------------------------+
                      |       menubar-load-runner       |
                      |         (Zsh launcher)          |
                      +---------------------------------+
                                       |
                       +---------------+---------------------+
                       |  --once, intercepted first          |  GUI launch
                       |  no guard, no compile               |  singleton -> compile -> detach
                       v                                     v
      +---------------------------------+   +---------------------------------+
      | snapshot path                   |   | MenuBarLoadRunnerApp (GUI)      |
      |                                 |   |                                 |
      | sample, wait, sample again      |   | status items, menu, labels      |
      | one JSON line, exit 0           |   | Keep Awake, state.json          |
      |                                 |   | CADisplayLink game loop         |
      | no NSApplication                |   | speed mapping 0..1              |
      | no state.json                   |   |                                 |
      +---------------------------------+   +---------------------------------+
                       |                                     |
                       +---------------+---------------------+
                                       v
              +--------------------------------------------------+
              | TelemetryCore                                    |
              |                                                  |
              | the nine readers, their probes and scalers       |
              | one snapshot(), physical units only              |
              |                                                  |
              | not its business: AppKit, speed mapping,         |
              | Keep Awake, state.json                           |
              +--------------------------------------------------+
                                       |
         +-----------------+-----------+-----+-----------------+
         v                 v                 v                 v
  +---------------+ +---------------+ +---------------+ +---------------+
  | Mach          | | IOKit         | | SMCClient     | | IOReport      |
  | CPU, memory   | | GPU, disk,    | | fan, die      | | ANE watts     |
  |               | | network,      | | temperature   | |               |
  |               | | battery       | |               | |               |
  +---------------+ +---------------+ +---------------+ +---------------+
```

Read it as two claims. **Downward:** nothing above the core can be reached from below — a reader cannot
ask what the menu is showing or whether Keep Awake is armed. **Sideways:** the two paths never meet. The
snapshot path builds no `NSApplication` and touches no file, which is what lets it run beside a live GUI
instance, in parallel with itself, over SSH, with nothing to lock.

## 1. What is wrong

Nine unprivileged readers run inside an app that has to be on screen to answer. A script, `co-cli`, or an
agent about to start a 40-minute build cannot ask this machine what its thermal, bandwidth or battery
headroom is — the only consumer of the readings is a pair of human eyes watching a GIF.

Two things follow, and only two: the readers must be usable without AppKit, and there must be one way to
ask them for a reading.

## 2. Modules and boundaries

| Module | Owns | Not its business — canonical owner |
|---|---|---|
| `TelemetryCore` | The nine readers, their availability probes, their scalers, and one `snapshot()` that returns physical units | Speed mapping, menu text, labels, Keep Awake, `state.json` — all `MenuBarLoadRunnerApp` |
| `MenuBarLoadRunnerApp` | Everything on screen and every intent that persists; asks the core for readings | How a reading is taken — `TelemetryCore` |
| `menubar-load-runner` (launcher) | Singleton guard, `compile_if_stale`, detach — for **GUI launches only** | Telemetry; and on the `--once` path, compiling anything |

**The core never imports a display concept.** It returns MB/s, °C, W, RPM, %, A — never the 0..1 driver
value. Normalization to 0..1 is a speed-mapping question, so the scalers stay inside the readers (they
are how a rate reader produces its own number) while the snapshot carries only what the hardware said.

**The core never reads or writes `state.json`.** A snapshot describes the machine, not this app's
intent, and a second writer would break the single-writer model (`ARCHITECTURE.md` § 8.2).

## 3. Interface

One flag. `--once` prints one line of JSON to stdout and exits.

| Property | Contract |
|---|---|
| Argument form | `--once` **must be the only argument.** Any other flag with it is a usage error, not a silent ignore — every other flag configures a GUI that this path does not build |
| Output | Exactly one line on stdout, a JSON object, newline-terminated. Nothing else on stdout, ever |
| Unavailable source | Its keys are **absent**. Never `null`, never a zero standing in for "no reading" |
| Side effects | None. No `NSApplication`, no status item, no `state.json` read or write, no `caffeinate`, no update check, no compile |
| Concurrency | Safe while a GUI instance runs, and safe in parallel with itself. It holds nothing and writes nothing |
| Exit | `0` a snapshot was printed (even if degraded) · `1` usage error · `2` binary not built (launcher only, names `--precompile`) |
| Latency | One process start plus one sampling window. About 210 ms wall, dominated by the window |

Rate readings (network, disk, swap, battery current, and bandwidth once R25 lands) are counter deltas
and do not exist at a single instant, so the path samples, waits `Tuning.snapshotWindow`, samples again,
and prints. This is why the budget is a window and not a syscall — a "sub-10 ms" snapshot could only
contain the point readings, and splitting the schema into fast keys and slow keys would be two schemas.

### Snapshot schema

`v` is the contract version and is the only field always present. Everything else appears when its
reader answered. Names carry their unit; `_mibs` is MiB/s (the menu writes "MB/s" as display shorthand —
same number, not a second fact).

```json
{"v":1,"cpu_pct":14.2,"mem_pct":41.0,"swap_mibs":0.0,"gpu_pct":28.0,"net_rx_mibs":1.4,"net_tx_mibs":0.2,"disk_read_mibs":0.0,"disk_write_mibs":3.1,"fan_rpm":[2160],"battery_pct":96.0,"battery_a":0.8,"temp_c":78.0,"thermal":"nominal","ane_w":0.0}
```

| Field | Unit | Reader | Absent when |
|---|---|---|---|
| `v` | int | — | never |
| `cpu_pct` | % | `CPULoadMonitor` | never (Mach always answers) |
| `mem_pct` | % | `MemoryLoadMonitor` raw used fraction | never |
| `swap_mibs` | MiB/s | `MemoryLoadMonitor` swap rate | swap counters unreadable |
| `gpu_pct` | % | `GPULoadMonitor` | no readable accelerator |
| `net_rx_mibs` · `net_tx_mibs` | MiB/s | `NetworkLoadMonitor` | — |
| `disk_read_mibs` · `disk_write_mibs` | MiB/s | `DiskLoadMonitor` | — |
| `fan_rpm` | RPM, one entry per fan | `FanLoadMonitor` (SMC) | fanless machine |
| `battery_pct` | % | `BatteryLoadMonitor` | desktop, no battery |
| `battery_a` | A, discharge positive | `BatteryLoadMonitor` | not discharging |
| `temp_c` | °C, hottest die sensor | `TemperatureLoadMonitor` (SMC) | no readable `Tp**` cluster |
| `thermal` | `nominal` · `fair` · `serious` · `critical` | `KernelThermalPressure` | never |
| `ane_w` | W | `ANELoadMonitor` (IOReport) | channel absent |

Adding a field is not a version bump; removing or re-meaning one is. Consumers read by key and ignore
what they do not know.

## 4. Parameters

| Name | Default | Unit | Role | Lives in |
|---|---|---|---|---|
| `--once` | off | flag | Single-shot snapshot; exclusive of every other argument | binary and launcher |
| `Tuning.snapshotWindow` | `0.2` | Seconds | Delta window between the two samples on the snapshot path | binary |
| `v` | `1` | int | Snapshot contract version | output |
| `MENUBAR_LOAD_RUNNER_FORCE_UNAVAILABLE` | unset | source key | Existing simulator, honored on this path so QA can assert an absent key | binary |
| `MENUBAR_LOAD_RUNNER_FORCE_BATTERY` | unset | `pct:state` | Existing simulator, honored on this path | binary |
| `MENUBAR_LOAD_RUNNER_FORCE_THERMAL` | unset | level | Existing simulator, display-only, honored on this path | binary |

`MENUBAR_LOAD_RUNNER_EXIT_AFTER` has no meaning here and is ignored: the path already exits on its own.

## 5. Implementation order

1. **Extract `TelemetryCore`** — move the nine monitors, `isSourceAvailable`, and `sampleSource(_:elapsed:)`
   out of `MenuBarLoadRunnerApp` into a `@MainActor` type the app holds one of. Every display function
   stays where it is and reads through the core. No behavior change: the whole existing `tests/qa.sh`
   run is the proof, and it must pass untouched before step 2.
2. **`TelemetrySnapshot`** — a struct of optionals in physical units, plus `jsonLine`. Built by the core
   from the same readers the menu rows use, so a row and a field can never disagree.
3. **`Config.parse()`** — add a `.snapshot` result, returned only when `--once` is the sole argument;
   any companion argument returns the existing usage failure.
4. **Top-level `switch`** — a third case that builds the core, samples, waits `Tuning.snapshotWindow`,
   samples again, writes `jsonLine` to `FileHandle.standardOutput`, and exits 0. `NSApplication` is
   never touched on this path.
5. **Launcher** — intercept `--once` in the first statement of argument handling, ahead of the singleton
   guard and `compile_if_stale`, and `exec` the binary. Missing binary: one stderr line naming
   `--precompile`, exit 2. Source newer than the binary is not consulted — a reading from the previous
   build is still a true reading, and compiling here would put a `swiftc` race back in front of the
   guard that the red line exists to prevent.
6. **`--help`** — one line, in both the launcher usage block and the app's help text.

## 6. Verification (real binary, `tests/qa.sh --core`)

| Command | Assertion |
|---|---|
| `$BIN --once` | Exit 0; stdout is exactly one line; `plutil -convert json -o - -` accepts it on stdin |
| `$BIN --once` (core tier, no WindowServer) | Same — this tier is the headless proof, no `ssh` harness needed |
| `$BIN --once` | Wall time under 500 ms |
| `$BIN --once --label value` | Exit 1, stderr carries a usage line, stdout empty |
| `FORCE_UNAVAILABLE=gpu $BIN --once` | Exit 0, no `gpu_pct` key |
| `FORCE_BATTERY=15:battery $BIN --once` | `battery_pct` is 15 |
| `$BIN --once` with a GUI instance running | Exit 0, and `state.json` mtime is unchanged afterwards |
| Launcher tier: `--once` with the binary removed | Exit 2, stderr names `--precompile`, nothing compiled |

## 7. Open before implementation

- **Every reader headless.** The GPU (IOAccelerator), SMC and IOReport readers are kernel-side and are
  expected to answer with no WindowServer connection, but this is expectation, not measurement. The
  `--core` tier row above is what turns it into measurement; if a reader comes back empty there, its
  keys are absent and the gate stays honest rather than the flag being declared broken.
- **One delta is enough.** The rate readers produce a raw rate from a single delta, which is what the
  snapshot carries. The scalers need several samples to settle, but nothing in the snapshot reads a
  scaler. Confirm on implementation that `ANELoadMonitor`'s IOReport subscription yields a usable delta
  over a 0.2 s window — it is the one reader whose delta is a subscription rather than a counter read.
- **`fan_rpm` as an array** is the only shape decision that a later reading could force open (per-fan
  utilization is also known). It stays RPM-only until something asks.
