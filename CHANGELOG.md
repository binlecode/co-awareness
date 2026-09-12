# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Public API (what semver governs)

MenuBar Load Runner is a CLI-launched app; the surface that MAJOR / MINOR / PATCH bumps apply to is:

- **Launcher CLI** — the positional preset keyword or GIF path, and the flags
  `--speed-multiplier`, `--label`, `--load-source`, `--keep-awake`, `--keep-awake-pid`,
  `--battery-threshold`, `--no-update-check`,
  `--foreground` / `--no-detach`, `--detach`, `--extra`, `--precompile`, `-h` / `--help`.
- **Environment variables** — `MENUBAR_LOAD_RUNNER_PATH`, `MENUBAR_LOAD_RUNNER_LOAD_SOURCE`,
  `MENUBAR_LOAD_RUNNER_LABEL`, `MENUBAR_LOAD_RUNNER_KEEP_AWAKE`,
  `MENUBAR_LOAD_RUNNER_KEEP_AWAKE_PID`,
  `MENUBAR_LOAD_RUNNER_BATTERY_THRESHOLD`, `MENUBAR_LOAD_RUNNER_UPDATE_CHECK`,
  `MENUBAR_LOAD_RUNNER_LOG_FILE`, `MENUBAR_LOAD_RUNNER_BIN_NAME`, and the debug/QA hooks
  `MENUBAR_LOAD_RUNNER_EXIT_AFTER`, `MENUBAR_LOAD_RUNNER_FORCE_UNAVAILABLE`,
  `MENUBAR_LOAD_RUNNER_FORCE_BATTERY`, `MENUBAR_LOAD_RUNNER_STATE_FILE`,
  `MENUBAR_LOAD_RUNNER_LOG_SLOTS`, `MENUBAR_LOAD_RUNNER_LOG_ASSERTIONS`,
  `MENUBAR_LOAD_RUNNER_LOG_AWAKE`, `MENUBAR_LOAD_RUNNER_LOG_ANIMATION`,
  `MENUBAR_LOAD_RUNNER_LOG_BATTERY_DIAGNOSTICS`, `MENUBAR_LOAD_RUNNER_FORCE_THERMAL`, and
  `MENUBAR_LOAD_RUNNER_LOG_THERMAL`.
- **Built-in preset keywords** and the `gifs/presets.json` manifest schema.
- **Observable behavior** — the status menu structure, the default preset, and the load-adaptive
  speed contract.

Internal implementation details (Swift types, `Tuning` constants, file structure) are **not** part
of the public API and may change in any release.

## [1.25.0] - 2026-09-12

### Added

- **The temperature row now says when macOS has started throttling the machine.** Once the system
  reports serious or critical thermal pressure, the row reads
  `Temperature: 98 °C · P-cores 92–98 °C · Thermal Throttling` — the sensor count steps aside, since
  a die reading alone can't tell you whether the kernel has begun clocking the hardware down, which
  is the thing that number is watched for. It is the *kernel's* throttling, kept distinct from this
  app slowing its own animation, which keeps its own `Slowing animation — …` line; and it shows on a
  fixed `--speed-multiplier` too, where that line is hidden by design and the menu previously said
  nothing about heat at all. No percentage comes with it: Apple Silicon manages clocks on-die and
  publishes a pressure level rather than a frequency cap, so a percentage here would be a number
  nothing measured. Nothing new is polled — the reading is one the app already observed.

## [1.24.0] - 2026-09-12

### Added

- **Option-click the menu bar item to turn Keep Awake on and off without opening the menu.** ⌥-clicking
  the creature — or either number slot — arms an indefinite hold, or releases whatever hold is running
  along with its window and its low-battery override. It is a shortcut *through* the submenu's own arm
  and disarm paths, not a second copy of them, so the battery pause, the arm-anyway rule and the 5%
  floor all apply to it unchanged. A plain click still opens the menu, a right- or Control-click still
  opens the menu, and ⌘-drag still rearranges the item.
- **The dropdown now reports the battery's own history: health, cycle count and capacity.** On a Mac
  with a battery, the `Battery` row reads `Battery: 80% · AC · 100% health · 113 cycles`, the state row
  names the condition and the capacity behind it (`Normal · 8478/8579 mAh`, or `Service Recommended`
  once health falls under 80% or the hardware reports a permanent failure), and hovering either row
  gives the full breakdown including the raw measured capacity. It reads `AppleSmartBattery` from the
  IORegistry with no privileges and, unlike every other reading here, **is never polled** — health and
  cycles move over months, so it is read once when you open the menu and dropped when you close it.
  Desktop Macs with no battery are unaffected: the reader finds no service and the menu stays as it was.

### Changed

- **The dropdown now opens on mouse release rather than on mouse press.** Reading a modifier requires
  the status item's button action, which only fires on release; an attached menu fires on press but
  never reports the modifier. This is the cost of the gesture above.

## [1.23.1] - 2026-09-11

### Fixed

- **A timed Keep Awake kept a 1 Hz redraw running while the icon was hidden.** The menu-bar countdown
  measured and relaid out its slot every second even with the status item behind the notch, in overflow,
  on another Space, or with the display asleep. The countdown now reads the same occlusion verdict the
  animation does and stops with it, redrawing on resume so the slot never shows a stale second. A
  countdown in an open menu is unaffected.

## [1.23.0] - 2026-09-10

### Added

- **Keep Awake bound to a process (`--keep-awake-pid <pid>`).** Holds the Mac awake until that process
  exits, then releases automatically (`DispatchSourceProcess .exit` watch + 2s polling fallback). Menu
  item `Keep Awake ▸ Until a process exits…` resolves by pid or process name. Honors the battery band
  and 5% floor; deliberately never resumed after a reboot.
- **Surface countdown on the menu bar.** Active Keep Awake timed windows render a seconds-resolution
  countdown (`88:88` / `88:88:88` template, 0 pt jitter) directly in the active status bar slot.
  Displays the paused tone when conditions suspend the hold, and collapses cleanly on expiry or disarm.

### Fixed

- Countdown no longer freezes at `00:01` on the bar when a window elapses during a low-battery pause.
- Externally killed `caffeinate` now cleanly releases process-bound hold intent.

## [1.22.0] - 2026-08-02

### Added

- **Reduce Motion & Manual Freeze.** Honors system `Reduce Motion` accessibility preference by freezing
  animation at 0% CPU. Added manual `Settings ▸ Freeze Animation` toggle. Live telemetry automatically
  hands off to the label slot when frozen, ensuring metrics remain readable without motion.
- Added `MENUBAR_LOAD_RUNNER_LOG_ANIMATION=1` QA hook to verify freeze state and frame cursor.

## [1.21.0] - 2026-08-02

### Fixed

- **Precompile update before restart.** Replaced long restart delays by compiling updates in the
  background while the app remains live; restart now takes ~1 second.
- **Singleton guard precedes compilation.** Duplicate launches check `pgrep` before triggering `swiftc`.
- **Atomic binary rebuilds.** Compiles to a temporary file before invoking `rename(2)` to prevent
  crashing running instances paging Mach-O segments.
- Added `--precompile` launcher flag for decoupled compilation.

## [1.20.1] - 2026-08-01

### Fixed

- **Idle core cluster handling in temperature reader.** All-parked core clusters reporting 0 are now
  classified as idle readings (`≤30 °C`) rather than spurious missing data.

## [1.20.0] - 2026-08-01

### Added

- **Eighth load source: Max Die Temperature (`--load-source temperature`).** Unprivileged leading thermal
  signal reading performance-core sensors via shared `SMCClient`. Fixed absolute scaling (30 °C idle →
  100 °C ceiling) following the hottest core cluster without excessive polling overhead.

## [1.19.4] - 2026-08-01

### Fixed

- **Shared `SMCClient` singleton.** Consolidated kernel handles so fan and temperature readers share
  a single process-lifetime connection without opening redundant endpoints.

## [1.19.3] - 2026-08-01

### Changed

- Paused Keep Awake track line alpha lightened to `0.22` for optimal contrast across dark and light appearances.

## [1.19.2] - 2026-07-31

### Fixed

- Suspended Keep Awake now wears a distinct faint track line (`0.22` alpha) rather than going completely
  dark like `Off`, clarifying the distinction between intent paused and intent cleared.
- Added `paused=` and `tint=` telemetry to `MENUBAR_LOAD_RUNNER_LOG_AWAKE=1`.

## [1.19.1] - 2026-07-30

### Fixed

- Reordered Keep Awake submenu: machine sleep state and `Other Assertions` sit in Section 1 (This Mac);
  controls and radio groups sit in Section 2 (This App) to avoid contradictory UI sandwiches.

## [1.19.0] - 2026-07-30

### Added

- **System assertion telemetry (`IOPMCopyAssertionsByProcess`).** Queries machine-wide power management
  state to surface external sleep locks, holder names, and release deadlines directly in the menu.
- Three-tier track line visual indicator: full tone ($1.0\alpha$) for own hold, foreign tone ($0.45\alpha$)
  for external hold, paused tone ($0.22\alpha$) when suspended.

## [1.18.0] - 2026-07-29

### Added

- **Configurable battery release threshold (`--battery-threshold <pct|off>`).** Keep Awake auto-releases
  at configured battery fraction (6%–100%, default 20%). Enforces unyielding 5% critical hardware floor.
- Low-battery arm override: explicitly arming below threshold honors intent down to the 5% hard floor.

## [1.17.1] - 2026-07-29

### Fixed

- Fixed 100% CPU lockup when clicking the status bar item while already selected.

## [1.17.0] - 2026-07-29

### Added

- **Dynamic label side selection (`Settings ▸ Label Position`).** Switch label between Left and Right
  of the animation at runtime with 0 pt jitter using dual-item slot architecture (`labelItemLeft`/`labelItemRight`).

## [1.16.0] - 2026-07-29

### Fixed

- **Zero-jitter layout guarantee.** Swapped auto-sizing status items for fixed template reservations,
  monospaced digits, and U+2007 figure-space padding, eliminating lateral jitter on value oscillations.

## [1.15.2] - 2026-07-28

### Fixed

- Fixed missing slot reservations causing adjacent menu bar items to jitter during rate changes.

## [1.15.1] - 2026-07-28

### Changed

- Expanded CLI parameter parsing to accept inline percentages and decimal values uniformly.

## [1.15.0] - 2026-07-28

### Added

- **Custom status bar text overlay (`--label <text>`).** Added static text label slot alongside runner.

## [1.14.0] - 2026-07-26

### Added

- **Atomic state persistence (`StateStore`).** Persists Keep Awake windows, label modes, and battery
  thresholds across relaunches to `~/Library/Application Support/menubar-load-runner/state.json`.

## [1.13.1] - 2026-07-26

### Fixed

- Hardened Keep Awake window recovery to prevent clock resets when resuming from condition suspensions.

## [1.13.0] - 2026-07-26

### Added

- **CLI launch-time Keep Awake (`--keep-awake <duration>`).** Accepts durations (`30m`, `2h`) at startup.
- Enforced 5% critical battery emergency safety floor.

## [1.12.0] - 2026-07-25

### Added

- **Timed Keep Awake duration presets.** Added preset windows (`30m`, `1h`, `2h`, `4h`, `8h`, `Custom…`)
  spawning `caffeinate -t <seconds>`.

## [1.11.2] - 2026-07-24

### Fixed

- Upgraded `caffeinate` invocation to `-di` (preventing display and idle sleep) for reliable Apple Silicon hold.
- Created status label slot up front to prevent wide GIF presets pushing numbers under the MacBook notch.

## [1.11.0] - 2026-07-16

### Added

- Merged Keep Awake on/off and color selection into a unified submenu with Graphite, Mauve, and Sage tints.

## [1.10.1] - 2026-07-16

### Changed

- Hardened standalone `install.sh` and `uninstall.sh` scripts; expanded test coverage.

## [1.10.0] - 2026-07-15

### Added

- **Seventh load source: Battery discharge rate (`--load-source battery`).** Unprivileged `IOPS` reader
  mapping discharge milliamps to animation speed via `ThroughputScaler`.

## [1.9.1] - 2026-07-15

### Fixed

- Fixed menu metric row formatting when switching between unbounded and percentage load sources.

## [1.9.0] - 2026-07-15

### Added

- Low battery auto-disengage at 20% charge fraction protecting laptop batteries from unattended drain.

## [1.8.0] - 2026-07-15

### Added

- **Integrated Keep Awake (first cut).** Spawns child `/usr/bin/caffeinate -w <pid>` bound to process lifetime
  with automatic thermal disengagement (`.serious` / `.critical`).

## [1.7.1] - 2026-07-10

### Changed

- Optimized vector tracing for runner silhouettes and fine-tuned aspect-ratio clamps.

## [1.7.0] - 2026-07-10

### Added

- Added live 60-second vertical bar sparkline (`LoadHistoryView`) inside status dropdown menu.

## [1.6.1] - 2026-07-10

### Fixed

- Corrected ThroughputScaler contraction headroom to prevent animation jitter under bursty I/O.

## [1.6.0] - 2026-07-10

### Added

- **Adaptive ThroughputScaler (ported from Linux `btop`).** Normalizes unbounded network, disk, and
  swap rates with asymmetric headroom (1.3x up, 3.0x down) and hysteresis debounce counters.

## [1.5.1] - 2026-07-10

### Changed

- Separated active load throttle indicator into dedicated menu status line.

## [1.5.0] - 2026-07-09

### Changed

- Subtracted 55% idle RAM baseline floor so macOS wired memory doesn't pin idle animation to full speed.

## [1.4.0] - 2026-07-09

### Changed

- Network, Disk, and Fan sources report independent axes (rx/tx, read/write, individual fan RPMs).
- Unified GIF frame cropping to single alpha union bounding box, eliminating frame-to-frame aspect wobbles.

## [1.3.0] - 2026-07-09

### Added

- **Sixth load source: Fan tachometers (`--load-source fan`).** Unprivileged SMC reader for fan RPMs.

## [1.2.2] - 2026-07-09

### Changed

- Vector-traced dog silhouette presets with `potrace` for smooth Retina edges.
- Added positional argument forgiveness for CLI load-source keywords.

## [1.2.1] - 2026-07-09

### Added

- Added project cover page (`docs/cover.html`) and asset build tooling.

## [1.2.0] - 2026-07-07

### Added

- Chihiro walk-cycle presets (`chihiro`, `chihiro-white`, `chihiro-black`).

## [1.1.3] - 2026-07-07

### Fixed

- Resolved relative resource paths for LaunchAgent `launchd` compatibility; aspect-fitted About dialog icon.

## [1.1.2] - 2026-07-07

### Changed

- Modularized LaunchAgent install documentation.

## [1.1.1] - 2026-07-07

### Fixed

- Eliminated race condition between `launchctl bootout` and `bootstrap` during login item re-installation.

## [1.1.0] - 2026-07-07

### Added

- Start-at-login LaunchAgent installer and uninstaller (`scripts/install-login-item.sh`).

## [1.0.0] - 2026-07-07

### Added

- Initial stable release.
- Native macOS status bar load visualizer (Swift + AppKit, single-file unbundled build, zero Xcode).
- 5 unprivileged load readers (CPU, Memory+Swap, GPU, Network, Disk).
- 9 built-in presets in `gifs/presets.json` (dog, horse, Totoro).
- Occlusion pausing (0% CPU under notch/sleep), vsync `CADisplayLink` game loop, thermal/power self-throttling.
- Zsh launcher with singleton enforcement and detached execution.
