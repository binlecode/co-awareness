# ARCHITECTURE — As-Built System Architecture & Technical Specifications

> **Canonical as-built architecture document for MenuBar Load Runner (v1.25.0).**
> **Source ground truth:** `MenuBarLoadRunner.swift`, `menubar-load-runner` (launcher), `gifs/presets.json`.
> **Scope:** Complete architectural specifications, subsystem topologies, concurrency models, telemetry algorithms, and system invariants.

---

## 0. Document Routing & ADLC Taxonomy (Canonical Ground Truth)

All repository documentation lives in `docs/`. The repository root holds only `README.md` (user entry point) and `CLAUDE.md` / `AGENTS.md` (agent instructions).

**This document (`docs/ARCHITECTURE.md`) is the canonical routing umbrella and as-built architectural ground truth.** Detailed subsystem designs and invariants reside here rather than in `CLAUDE.md`.

```
   docs/
   +-- ARCHITECTURE.md          <-- You are here: system topology, subsystem specs (§1–§13),
   |                              telemetry algorithms, invariants, parameter reference, release hygiene.
   +-- ROADMAP.md                 The standing product tracker: candidate backlog (R<n>),
   |                              and declined proposals with rationale.
   +-- RESEARCH-<topic>.md        External peer surveys and ecosystem research (e.g. peer-survey.md);
   |                              facts-only, dated evidence outside the ADLC landing chain.
   +-- PLAN-<topic>.md            Active feature / issue design and implementation plans;
   |                              distilled into ARCHITECTURE.md upon landing, then `git rm` deleted.
   +-- cover.html, media/         Public landing page and visual assets.
   +-- .claude/skills/<name>/     Agent-executable operational workflows (build-visuals, etc.).
```

### Document Taxonomy & Lifecycle Table

| Document Type | Naming Convention | Lifecycle & Purpose |
|---|---|---|
| **Roadmap** | `docs/ROADMAP.md` | Single standing tracker for open candidate backlog (`R<n>`) and declined proposals. Never deleted. |
| **Plan / Proposal** | `docs/PLAN-<topic>.md` | Active feature design, options, and verification checklist. **Absorbed into `docs/ARCHITECTURE.md` upon landing, then immediately deleted (`git rm`)**. |
| **As-Built Architecture** | `docs/ARCHITECTURE.md` | Canonical description of what the code actually is and why. Updated only after code lands and stabilizes. |
| **External Research** | `docs/RESEARCH-<topic>.md` | Public surveys and external benchmarks. Does not enter the ADLC landing cycle; retained as dated evidence. |

- **Strict Document Kinds:** No invented prefixes (`REPORT-`, `DESIGN-`, `TODO-`, `LESSONS`, `RUNBOOK-`, `JOURNAL`).
- **One Fact, One Place:** Do not duplicate subsystem contracts across README, help text, and docs. State facts once and cross-reference.
- **Pure ASCII Diagrams:** All system diagrams must use standard ASCII characters (`+ - | = v ^ < >`), avoiding box-drawing characters.
- **Editing Rule:** Always use the dedicated Edit tool rather than `sed` to avoid mangling formatting.

---

## 1. System Topology & Architectural Philosophy

MenuBar Load Runner is a single-file, unbundled native macOS menu bar application written in Swift and AppKit. It visualizes real-time hardware telemetry by driving the playback rate of an animated status-bar GIF and providing an integrated live diagnostic dashboard with built-in sleep inhibition.

There are **two entry paths and one set of readers**. The GUI is the app; `--once` is a single-shot snapshot for anything that is not a pair of eyes (§ 4.7). The paths never meet: they share `TelemetryCore` and nothing else.

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

### Core Design Tenets

1. **Read-Only Telemetry & Self-Throttling:** The application observes the system without modifying system settings or CPU governors. When system load or thermal conditions escalate, the application throttles its own rendering footprint to avoid exacerbating contention.
2. **Zero-Xcode Single-File Architecture:** The entire runtime resides in `MenuBarLoadRunner.swift` (~6.6k lines) compiled via `swiftc` with complete concurrency checking (`-strict-concurrency=complete`).
3. **No Mocks / Non-Privileged Execution:** Every metric is collected via unprivileged public Mach, IOKit, and SMC APIs without root privileges, background daemons, or kernel extensions.
4. **Jitter-Free Menu Bar Real Estate:** Status item widths are strictly reserved using figure-space padding (U+2007) and monospaced digits, ensuring that value oscillations never cause lateral layout jitter.

---

## 2. Launcher & Compilation Lifecycle

Execution is governed by the `menubar-load-runner` zsh script, which manages compilation, process singletons, and detached execution.

```
                      +----------------------------+
                      |     Execution Request      |
                      +-------------+--------------+
                                    |
                                    v
                      +----------------------------+
                      |  Per-User Singleton Check  |
                      |   (pgrep -U <uid> -f ...)  |
                      +-------------+--------------+
                                    |
                      +-------------+--------------+
             Instance Running?             No Instance Running
                      |                            |
                      v                            v
         [Exit with notice / --extra]   +----------------------------+
                                        | Is Source Newer than Mach-O|
                                        +-------------+--------------+
                                                      |
                                           +----------+----------+
                                         Stale                Up to date
                                           |                     |
                                           v                     |
                                +---------------------------+    |
                                |   swiftc -O -strict-      |    |
                                |   concurrency=complete    |    |
                                |   -o MenuBarLoadRunner.new|    |
                                +----------+----------------+    |
                                           |                     |
                                           v                     |
                                +---------------------------+    |
                                | rename(2) atomically over |    |
                                |    MenuBarLoadRunner      |    |
                                +----------+----------------+    |
                                           |                     |
                                           +---------------------+
                                           v
                                +---------------------------+
                                | Launch Process (Detached  |
                                |      or Foreground)       |
                                +---------------------------+
```

### Compilation Mechanics

- **Atomic Rename:** When compiling, the launcher outputs to `MenuBarLoadRunner.new` before invoking `mv` (`rename(2)`) over `MenuBarLoadRunner`. This guarantees that an existing live process paging from the Mach-O binary does not crash during a rebuild.
- **Precompilation Hook (`--precompile`):** Exposes the compilation branch without launching the process. Used by the in-app self-updater to build newly pulled source code while the current instance remains live.
- **Strict Concurrency Safety:** Compiled with Swift 5 `-strict-concurrency=complete`. All UI and state-managing classes are annotated `@MainActor`.
- **Interpreted Fallback & Singleton Scope:** If `swiftc` compilation fails or toolchain elements are unavailable, the launcher falls back to interpreted execution via `swift MenuBarLoadRunner.swift`. This degraded emergency fallback is intentionally not singleton-guarded: the launcher's singleton check (`pgrep -U "$(id -u)" -f "/MenuBarLoadRunner( |$)"`) explicitly matches the compiled binary path to avoid false positives against editors holding `MenuBarLoadRunner.swift` open or background `swiftc` builds, while the interpreted fallback process executes directly under `/usr/bin/swift`.

---

## 3. Rendering Pipeline & Game Loop Engine

The visualizer transforms raw GIF frames into an aspect-ratio-fitted, vsync-aligned status bar animation.

### 3.1 Transparent Padding Trimming (`trimTransparentPadding`)

Many raw pixel-art GIFs contain uniform transparent margins. At initialization:
1. Every frame in the GIF is scanned at the raw pixel level (`CGImage` provider).
2. Alpha components below `Tuning.alphaVisibleThreshold` (3) are treated as transparent.
3. A single union bounding box is computed across all frames to prevent frame-to-frame shifting.
4. Frames are cropped to this union bounding box, deriving the canonical aspect ratio (`currentGifAspect`).

### 3.2 Dynamic Aspect & Slot Length Derivation

Width is not a fixed constant. It is derived at runtime:
$$\text{slotLength} = \max\left(\text{menuBarHeight} \times \text{clamp}(\text{aspect}, \text{minAspect}, \text{maxIconAspect}), \text{minBaseSlotWidth}\right)$$
- `Tuning.minAspect` = `0.01`
- `Tuning.maxIconAspect` = `6.0`
- `Tuning.minBaseSlotWidth` = `18.0 pt`
- Sizing is refreshed whenever a new GIF preset is loaded via `applySizing()`, triggering `updateRenderedFrames()`.

### 3.3 Vsync-Aligned Game Loop

On macOS 14+, the animation is driven by a `CADisplayLink` bound to the status item's button view (`NSView.displayLink(target:selector:)`). On older macOS releases, a 60 Hz fallback `Timer` (`Tuning.gameLoopFallbackInterval`) is used.

```swift
// Abridged from MenuBarLoadRunner.advanceFrames(now:) — elided: the empty-frames guard
// and the first-tick latch (lastTickTime == 0 returns without advancing).
let delta = now - lastTickTime
lastTickTime = now
// A backwards jump or a gap larger than maxFrameAdvanceDelta is DROPPED, not replayed and
// not resynced here: the next tick simply resumes from the current frame.
guard delta > 0, delta <= Tuning.maxFrameAdvanceDelta else { return }

accumulatedFrameTime += delta
var advanced = false
while true {
    let baseDelay = baseDurations[frameIndex]                              // this frame's GIF delay
    let requiredDelay = max(baseDelay / speedMultiplier, Tuning.minGifFrameDelay)
    if accumulatedFrameTime >= requiredDelay {
        accumulatedFrameTime -= requiredDelay
        frameIndex = (frameIndex + 1) % baseDurations.count
        advanced = true
    } else { break }
}
if advanced { renderCurrentFrame() }                                        // layer.contents = image
```

- **Speed is a divisor on each frame's own delay**, not a multiplier on elapsed time, and the result
  floors at `Tuning.minGifFrameDelay` (0.02s) so a fast preset can't outrun the display link.
- **`speedMultiplier` is read live**, so a speed change takes effect on the next tick without
  restarting the driver.
- **Sleep/Occlusion gaps are dropped, not replayed:** a delta over `Tuning.maxFrameAdvanceDelta` (1.0s) — sleep, occlusion, a clock jump — returns without advancing, so the animation resumes from the current frame instead of catching up in a burst. An explicit `resetGameLoopTiming()` exists for the *frame-source switch* and driver-(re)start paths, not for this one.

---

## 4. Hardware Telemetry & Scaling Subsystems

The application includes nine unprivileged telemetry monitors, owned by `TelemetryCore` (§ 4.7) and sampled by the GUI every 2 seconds (`Tuning.loadSampleInterval`).

```
+----------------------------------------------------------------------------------------+
|                                   Telemetry Monitors                                   |
+------------------------+---------------------------+-----------------------------------+
| Monitor Class          | Primary Kernel/Mach API   | Normalization / Scaling Model     |
+------------------------+---------------------------+-----------------------------------+
| CPULoadMonitor         | host_processor_info()     | Exponential Moving Average (EMA)  |
| MemoryLoadMonitor      | host_statistics64()       | Composite: max(RAM%, ScaledSwap)  |
| GPULoadMonitor         | IORegistry IOAccelerator  | Direct Percentage (0.0 .. 1.0)    |
| SMCClient (Fan)        | AppleSMCKeysEndpoint      | RPM / Max RPM (Average of Fans)   |
| SMCClient (Temp)       | AppleSMCKeysEndpoint      | Fixed 30°C .. 100°C Window        |
| NetworkLoadMonitor     | getifaddrs() (AF_LINK)    | ThroughputScaler (Bytes/Sec)      |
| DiskLoadMonitor        | IOBlockStorageDriver      | ThroughputScaler (Bytes/Sec)      |
| BatteryLoadMonitor     | IOKit Power Sources       | ThroughputScaler (Discharge mA)   |
| ANELoadMonitor         | IOReport (Energy Model)   | ThroughputScaler (Watts)          |
+------------------------+---------------------------+-----------------------------------+
```

### 4.1 CPU Load Monitoring (`CPULoadMonitor`)

- Mach call: `host_processor_info(mach_host_self(), PROCESSOR_CPU_LOAD_INFO, ...)`.
- Computes tick deltas across all system cores:
  $$\Delta \text{total} = \Delta \text{user} + \Delta \text{system} + \Delta \text{nice} + \Delta \text{idle}$$
  $$\text{rawFraction} = \frac{\Delta \text{user} + \Delta \text{system} + \Delta \text{nice}}{\Delta \text{total}}$$
- Smoothed via EMA with smoothing factor $\alpha = 0.20$ (`Tuning.cpuSmoothingAlpha`):
  $$\text{smoothedUsage}_t = \alpha \times \text{rawFraction} + (1 - \alpha) \times \text{smoothedUsage}_{t-1}$$

### 4.2 Composite Memory & Swap Monitoring (`MemoryLoadMonitor`)

- **RAM Used Fraction:** Derived from `host_statistics64(HOST_VM_INFO64)` — measured from what is
  *reclaimable*, not from a sum of "used" buckets, which is why it reads higher than Activity Monitor
  (a deliberate approximation, documented at the call site):
  $$\text{availablePages} = \text{free\_count} + \text{purgeable\_count} + \text{external\_page\_count}$$
  $$\text{rawFraction} = 1 - \frac{\text{availablePages} \times \text{pageSize}}{\text{totalPhysicalRAM}}$$
  $$\text{adjustedRAMFraction} = \frac{\max(0, \text{rawFraction} - \text{memoryIdleFloor})}{1.0 - \text{memoryIdleFloor}}$$
  (`Tuning.memoryIdleFloor` = `0.55`, compensating for OS cache retention).
- **Swap Paging Rate:** Measures delta of `swapins` + `swapouts` over elapsed wall time $\Delta t$, normalized through `ThroughputScaler` (`swapFloorBytesPerSec` = 1 MiB/s).
- **Composite Driver:**
  $$\text{currentMemoryLoad} = \max(\text{adjustedRAMFraction}, \text{scaler.scale}(\text{swapRateBytesPerSec}))$$

### 4.3 SMC Sensor Architecture (`SMCClient`)

`SMCClient` is a thread-safe singleton communicating with `AppleSMCKeysEndpoint` using the standard 80-byte `SMCKeyData` protocol structure.

```
                          +---------------------------+
                          |   SMCClient.ensureOpen()  |
                          | IOServiceOpen("AppleSMC")  |
                          +-------------+-------------+
                                        |
                                        v
                          +---------------------------+
                          | Discover Key Count (#KEY) |
                          +-------------+-------------+
                                        |
                                        v
                          +---------------------------+
                          | Binary Search Key Table   |
                          |   for Target Prefix (Tp*) |
                          +-------------+-------------+
                                        | ~115 calls (20ms) vs 3385 full scan (600ms)
                                        v
                          +---------------------------+
                          | Filter & Read Sensor Data |
                          | (Command 8 / readBytes)   |
                          +---------------------------+
```

- **Binary Search Key Table Discovery:** Instead of sequential table scans (~600ms) or guesswork, `SMCClient.floatKeys(withPrefix:)` uses binary search across the ascending SMC key table (~20ms), discovering dynamic fan (`F{n}Ac`) and temperature (`Tp**`) keys across Intel and Apple Silicon chips.
- **Die Temperature Mapping:** Reads hottest cluster maxima (`Tpx*` cluster maximum sensors on Apple Silicon). Bounded strictly between $30^\circ\text{C}$ and $100^\circ\text{C}$ (`Tuning.temperatureFloorCelsius` to `temperatureCeilingCelsius`):
  $$\text{loadFraction} = \text{clamp}\left(\frac{T_{\text{max}} - 30.0}{100.0 - 30.0}, 0.0, 1.0\right)$$

### 4.4 The Adaptive `ThroughputScaler` (btop Hysteresis)

For unbounded rates (network bytes/sec, disk bytes/sec, swap bytes/sec, battery discharge mA), `ThroughputScaler` provides jitter-free normalization:

1. **Sliding Window Ceiling:** Computes rolling average over the last $N=5$ samples (`Tuning.scalerWindow`).
2. **Asymmetric Headroom:**
   - Headroom Up: $1.3\times$ (`Tuning.scalerHeadroomUp`)
   - Headroom Down: $3.0\times$ (`Tuning.scalerHeadroomDown`)
3. **Hysteresis Counters:** Rescaling requires $5$ consecutive out-of-band samples (`Tuning.scalerRescaleCount`) tracked in `overCount` and `underCount` registers, preventing single bursts from oscillating the display.

### 4.5 Neural Engine Power Monitoring (`ANELoadMonitor`)

The ninth reader, and the only one built on a private API. It exists because the NPU is the one busy
state the other eight cannot see: on-device inference (Apple Intelligence, CoreML, MLX's ANE backend,
Vision) runs the Neural Engine while CPU and GPU utilization sit near idle, so every other source
reports a quiet machine while it is working hard.

**Why `IOReport` does not breach the unprivileged tenet.** `IOReport` is the user-space interface
`powermetrics` reads energy from — no root, no kext, no entitlement, no system state mutated. What it
lacks is a public *header*, not permission. The entry points are therefore bound at runtime with
`dlopen`/`dlsym` behind `@convention(c)` signatures, and every failure along that path (missing dylib,
missing symbol, missing group, missing rail) collapses into `isAvailable == false` — never a trap, and
never a fabricated reading. This keeps the single-file, zero-dependency build intact: no bridging
header, no linker flag, no `Package.swift`.

**The dylib path is load-bearing.** As of macOS 26 there is no `IOReport.framework` on disk *or in the
dyld shared cache*, so `dlopen` of the framework path fails outright. The symbols ship in
`/usr/lib/libIOReport.dylib`.

**Stateful delta sampling, not a point read.** Unlike every other monitor, `IOReport` is a
subscription API. One subscription is opened over the `"Energy Model"` channel group for the life of
the process (the `SMCClient` precedent — one long-lived handle, never a per-sample open), and a
reading is the difference between two samples:

```
  bind: dlopen -> dlsym -> IOReportCopyChannelsInGroup("Energy Model")
          |                         |
          |                   filter to the "ANE" row alone
          v                         v
  IOReportCreateSubscription(nil, channels, &subscribed, 0, nil)   <-- see note below
          |
          v
  tick:  S1 = IOReportCreateSamples(...)        (t1)
         S2 = IOReportCreateSamples(...)        (t2)
         D  = IOReportCreateSamplesDelta(S1, S2)
         P  = joules(D["ANE"], unit) / (t2 - t1)     -> Watts
         S1 <- S2
```

- **The subscribed-channels out-parameter is not optional.** Passing `nil` makes
  `IOReportCreateSubscription` return `nil` on hardware that supports the API perfectly well — which
  would read as "no Neural Engine" on *every* Apple Silicon Mac. It must receive a real pointer.
- **Unit labels are read per row, never assumed.** One sample mixes units across rails — measured on an
  M4 Max, 322 of 328 channels report `mJ`, five `uJ`, and the GPU's own row `nJ`.
- **The channel list is filtered to the ANE rail before subscribing.** The group cannot be narrowed by
  subgroup (every energy row reports an empty one), so the desired-channel list is filtered by hand.
  This is a small win, not a large one — 3.6 ms → 2.8 ms per sample+delta, because the cost is the
  kernel round trip and not the row count — but it states in code exactly which rail this reader may see.
- **No sleep-gap special case is needed.** `elapsed` comes from `systemUptime`, which does not advance
  while the machine is asleep, and neither does the energy counter.

**Normalization.** Watts are unbounded and vary by chip tier, so the reading goes through the same
`ThroughputScaler` as the other rate sources rather than a hardcoded per-chip cap. `Tuning.aneFloorWatts`
= `1.0` W is the minimum ceiling, and unlike the rate floors it is not there to reject idle chatter — a
power-gated NPU reads a hard, exact `0` — but to keep a *trivial* inference from reading as a busy one.
Measured on an M4 Max: `0` W idle, 0.24–0.32 W under Vision accurate text recognition, and 1.4–3.7 W
under image-featureprint, saliency, and body-pose inference.

**Accepted cost.** This is the most expensive reader in the suite. `IOReportCreateSamples` costs ~2.9 ms
per call and is effectively the entire per-tick cost (the delta and the single-row scan measure 0.001 ms
each), which no filtering on this side reduces. Measured on the real binary over a 20-sample window:
0.40% of a core with `--load-source ane` against 0.12% for `cpu`/`fan` and 0.18% for `temperature`. It is
paid only when ANE is the active source or the Other Sources list is expanded — active-only sampling is
unchanged — and it buys the one signal nothing else in the app can report.

### 4.6 Battery Telemetry & Static Diagnostics (`BatteryDiagnosticsReader`)

Battery telemetry operates across two distinct time domains to honor the unprivileged, minimal-footprint tenet:

1. **High-Frequency Dynamic Telemetry (`BatteryLoadMonitor`)**:
   - Sampled periodically during active telemetry.
   - Reads instantaneous discharge current (`kIOPSCurrentKey`) and battery charge percentage via unprivileged `IOPSCopyPowerSourcesInfo()`.
   - While on battery, discharge current (mA) normalizes through `ThroughputScaler` (`Tuning.batteryFloorMilliamps` = 500 mA). While plugged into AC, discharge draw is 0, driving the animation at idle speed.
2. **Static Health & Capacity Diagnostics (`BatteryDiagnosticsReader`, R22)**:
   - Battery health and cycle counts are slow-moving hardware metrics that degrade over months, not seconds.
   - **Strict Zero-Polling Invariant**: Health diagnostics are **never** queried in the background 2s telemetry loop or vsync game loop.
   - **Menu-Gated Single Read**: Queried unprivileged from `AppleSmartBattery` via `IOServiceGetMatchingService(kIOMainPortDefault, IOServiceMatching("AppleSmartBattery"))` strictly once on `menuWillOpen(_:)`, and cleared on `menuDidClose(_:)`.
   - **Normalizations**: Automatically maps Apple Silicon normalized `MaxCapacity` percentages (0..100) and legacy Intel raw mAh capacities against `DesignCapacity`. Computes service recommendation conditions when health falls below 80% or `PermanentFailureStatus != 0`.
   - **Presentation**: Enriches the battery status row (`Battery: 80% · AC · 100% health · 113 cycles`), `stateItem` (`Battery State: Normal · 8478/8579 mAh`, replacing the drain band while the menu is open), and provides multi-line AppKit tooltips with granular mAh capacities.
   - **Graceful Desktop Omission**: On AC-only desktop Macs lacking an `AppleSmartBattery` service, the reader cleanly returns `nil`, and the menu seamlessly preserves standard desktop AC status with zero visual defects or overhead.
   - **Observability without behavior change**: `MENUBAR_LOAD_RUNNER_LOG_BATTERY_DIAGNOSTICS=1` prints one line at launch and one per menu open. It reads a throwaway copy and never assigns the cached field, so the hook cannot make the menu render the diagnostics branch at a moment the menu was never open — the reason it is not wired into `sampleSystemLoad()` where the other `LOG_*` hooks sit.

`MENUBAR_LOAD_RUNNER_FORCE_BATTERY=<pct>[:battery|:ac]` pins the charge and power state on `BatteryLoadMonitor` itself, not on a caller. Two places in the app read `IOPSCopyPowerSourcesInfo` — Keep Awake's suspension policy (§ 7.2) and this reader — and a hook honored by only one of them would let them disagree about the same battery in the same run. Current (mA) is left real: the hook simulates a charge and a power state, nothing else. On a desktop it also makes the reader *answer*, which is the only way a machine with no battery can exercise the path at all.

### 4.7 Telemetry Core & the `--once` Snapshot (R24)

Nine unprivileged readers used to run inside a status item, so the only consumer of a reading was a pair of human eyes. `TelemetryCore` is the type that owns them; `--once` is the one way anything else asks.

**Module boundaries.** Each row's *not its business* column names the canonical owner, so nothing has to be inferred:

| Module | Owns | Not its business — canonical owner |
|---|---|---|
| `TelemetryCore` | The nine readers, their availability probes, their scalers, `sampleSource(_:elapsed:)`, `isSourceAvailable(_:)`, and one `snapshot()` returning physical units | Speed mapping, menu text, labels, Keep Awake, `state.json` — all `MenuBarLoadRunnerApp` |
| `MenuBarLoadRunnerApp` | Everything on screen and every intent that persists; asks the core for readings | How a reading is taken — `TelemetryCore` |
| `menubar-load-runner` (launcher) | Singleton guard, `compile_if_stale`, detach — for **GUI launches only** | Telemetry; and on the `--once` path, compiling anything |

The core never imports a display concept. It returns MB/s, °C, W, RPM, %, A — never the 0..1 driver value. Normalization to 0..1 is a speed-mapping question, which is why the scalers (§ 4.4) stay *inside* the readers, where they are how a rate reader produces its own number, and why nothing in a snapshot reads one. The core also never touches `state.json`: a snapshot describes the machine, not this app's intent, and a second writer would break the single-writer model (§ 8.2).

**Interface.** One flag, one schema, one sampling window:

| Property | Contract |
|---|---|
| Argument form | `--once` **must be the only argument.** Any other flag with it is a usage error, not a silent ignore — every other flag configures a GUI this path does not build |
| Output | Exactly one line on stdout, a JSON object, newline-terminated. Nothing else on stdout, ever — the usage error above prints to stderr *without* the usage block for this reason |
| Unavailable source | Its keys are **absent**. Never `null`, never a zero standing in for "no reading" |
| Side effects | None. No `NSApplication`, no status item, no `state.json` read or write, no `caffeinate`, no update check, no compile |
| Concurrency | Safe while a GUI instance runs, and safe in parallel with itself. It holds nothing and writes nothing |
| Exit | `0` a snapshot was printed (even if degraded) · `1` usage error · `2` binary not built (launcher only, names `--precompile`) |
| Latency | One process start plus one sampling window. ~290 ms wall measured on an M4 Max, dominated by the window |

The rate readings (network, disk, swap, battery current, ANE) are counter deltas and do not exist at a single instant, so the path samples, waits `Tuning.snapshotWindow`, samples again against the *measured* gap, and prints. The budget is therefore a window, not a syscall: a sub-10 ms snapshot could only carry the point readings, and splitting the schema into fast keys and slow keys would be two schemas.

**Schema.** `v` is the contract version and the only field always present; every other key appears when its reader answered. Names carry their unit — `_mibs` is MiB/s (the menu writes "MB/s" as display shorthand: same number, not a second fact). Adding a field is not a version bump; removing one or changing what it means is, and consumers read by key and ignore what they do not know.

```json
{"v":1,"cpu_pct":14.2,"mem_pct":41.0,"swap_mibs":0.00,"gpu_pct":28.0,"net_rx_mibs":1.40,"net_tx_mibs":0.20,"disk_read_mibs":0.00,"disk_write_mibs":3.10,"fan_rpm":[2160],"battery_pct":96.0,"battery_a":0.80,"temp_c":78.0,"thermal":"nominal","ane_w":0.00}
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
| `battery_a` | A, discharge positive | `BatteryLoadMonitor` | not discharging (AC, or no current reading) |
| `temp_c` | °C, hottest die sensor | `TemperatureLoadMonitor` (SMC) | no readable `Tp**` cluster |
| `thermal` | `nominal` · `fair` · `serious` · `critical` | `KernelThermalPressure` | never |
| `ane_w` | W | `ANELoadMonitor` (IOReport) | channel absent |

Precision follows the reader, not the field: percentages and °C carry one decimal, rates, watts and amps two, RPM none — each finer than the hardware's own resolution. The line is assembled by hand rather than by `JSONEncoder`, because the contract fixes the key order and the per-unit precision and an encoder gives neither.

**Launcher interception.** `--once` is handled in the first statement of argument handling, ahead of both the singleton guard and `compile_if_stale`, and `exec`s the binary with argv unchanged (exclusivity is the binary's to enforce, since it owns the usage text). The source being newer than the binary is deliberately not consulted: a reading from the previous build is still a true reading, and compiling here would put a `swiftc` race back in front of the very guard that exists to prevent one (§ 2). A missing binary is one stderr line naming `--precompile` and exit 2.

**Headless is measured, not assumed.** The GPU (IOAccelerator), SMC and IOReport readers are kernel-side and were *expected* to answer with no WindowServer connection. `tests/qa.sh` §2a is what turns that into a measurement: it runs in the core tier, which never boots a GUI, and asserts the always-present keys are there rather than only that the JSON parses — a snapshot degrading to `{"v":1}` would otherwise pass "absent when unavailable" while telling the truth about nothing.

---

## 5. Power Management, Occlusion & Accessibility

The engine reduces its own footprint under thermal or battery strain.

```
                                  +--------------------------+
                                  |   Environmental Event    |
                                  +------------+-------------+
                                               |
              +--------------------------------+--------------------------------+
              |                                |                                |
              v                                v                                v
   [Occlusion State Change]       [Thermal / Power / Memory]         [Reduce Motion Toggle]
 (NSWindow Occlusion Notification)  (ProcessInfo / DispatchSource)   (NSWorkspace / Menu Toggle)
              |                                |                                |
              v                                v                                v
    Is Fully Occluded?              Is Under Power Pressure?             Is Animation Frozen?
     * Notch Coverage               * Low Power Mode                     * System Reduce Motion
     * Inactive Space               * Thermal Serious/Critical           * Manual Settings Toggle
     * Display Sleep                * Memory Warning/Critical                   |
              |                                |                                |
              v                                v                                v
    Halt Game Loop (0% CPU)         Cap Speed Multiplier             syncGameLoopRunning():
    Resume when visible             at Preset Midpoint (0.5×)        Hold frame, handoff to label
```

### 5.1 Occlusion Pausing (0% CPU)

- Listens for `NSWindow.didChangeOcclusionStateNotification` on the status item window.
- When `occlusionState` does not contain `.visible` (e.g. hidden behind a MacBook notch, displaced by menu bar crowding, situated on an inactive macOS Space, or display asleep), the `CADisplayLink` / `Timer` is stopped completely.
- Frame rendering and rasterization drop to **0.0% CPU**.

### 5.2 Power, Thermal & Memory Throttling

- **Triggers:**
  - `NSProcessInfo.isLowPowerModeEnabled` (`.NSProcessInfoPowerStateDidChange`)
  - `ProcessInfo.thermalState` is `.serious` or `.critical` (`ProcessInfo.thermalStateDidChangeNotification`)
  - Memory Pressure is `.warning` or `.critical` (`DispatchSource.makeMemoryPressureSource`)
- **Action:** Speed multiplier is capped at `Tuning.constrainedSpeedCeilingFraction` ($0.5\times$ range midpoint). Immediate recalculation bypasses standard 2s hysteresis.

**Kernel throttling is a separate fact from this self-throttling, and the menu states them separately (R23).** `KernelThermalPressure` maps `ProcessInfo.thermalState` to a level and annotates the *temperature* row with `· Thermal Throttling` at `.serious` / `.critical`, replacing the sensor count rather than extending the row. Three constraints shape it:

- **Level only, never a percentage.** Apple Silicon manages clocks on-die via CLPC and publishes no unprivileged frequency cap: `IOPMCopyCPUPowerStatus` answers `kIOReturnNotFound` (probed on M4 Max; `pmset -g therm` agrees). Intel's `CPU_Speed_Limit` was declined rather than special-cased: no Intel hardware is available to this project, so both the IOKit read and the row it would annotate would ship unverified — and § 4.3's claim that `Tp**` discovery spans Intel is itself untested here (`floatKey` accepts only `flt `-typed keys, so an Intel Mac may answer with a different type rather than a different name). Deriving a percentage from a level would be a fabricated reading.
- **No new telemetry source, no new timer, no new sample.** It reads a property the app already observes, inline in `refreshMenuMetrics()` — so it is evaluated on that method's existing cadence (the 2s tick and `menuWillOpen`) and adds no cache, no timer of its own, and no IOKit call.
- **`.fair` does not annotate.** That is headroom narrowing, not the kernel clocking anything down.

It also closes a gap: `throttleStatusItem` is set only in the `isAutoSpeed` branch, so on a fixed `--speed-multiplier` the menu previously carried no thermal indication at all — and its wording could not simply be un-gated, since in that mode the app is not slowing anything.

**Wiring invariant.** `loadReductionReasons` and `isUnderPowerPressure` read `ProcessInfo.thermalState` directly and must never consult `KernelThermalPressure`. The type is display-only, which is what keeps its `MENUBAR_LOAD_RUNNER_FORCE_THERMAL` hook an input simulator rather than a hook that moves a business decision (§ 10). `MENUBAR_LOAD_RUNNER_LOG_THERMAL` prints the level, the derived gate, the row it produced and the self-throttle row's state together, so a headless test asserts the separation instead of trusting it.

### 5.3 Reduce Motion & Manual Freeze

- **System Accessibility:** Listens for `NSWorkspace.accessibilityDisplayOptionsDidChangeNotification` to observe `NSWorkspace.shared.accessibilityDisplayShouldReduceMotion`.
- **Manual Toggle:** `Settings ▸ Freeze Animation` (persisted in `state.json`).
- **Unified Decider (`syncGameLoopRunning()`):** If either system reduce motion or manual freeze is active, the game loop stops, holding the current frame.
- **Label Handoff:** While frozen, if the menu bar label is configured to `off`, it automatically switches to `.value` mode temporarily so the user still receives live telemetry.

---

## 6. Status Bar Layout & Dual-Slot Label Model

To prevent lateral jitter and enable flexible placement, the application implements a dual status-item architecture.

```
                           macOS Menu Bar Item Creation Order
                   (Oldest Created = Rightmost Placement on Screen)
                   
          +---------------------+---------------------+---------------------+
          |   labelItemRight    |     statusItem      |    labelItemLeft    |
          |   (Status Item 1)   |   (Status Item 2)   |   (Status Item 3)   |
          |   Created First     |   Created Second    |   Created Third     |
          +----------+----------+----------+----------+----------+----------+
                     |                     |                     |
                     v                     v                     v
          [Right Slot (Active)]    [Animated GIF Art]   [Left Slot (Hidden)]
              length = 87 pt         length = 47 pt         length = 0 pt
```

### 6.1 Fixed-Width Reservation & Jitter Elimination

Auto-sizing status items causes neighboring items to jitter on every telemetry update. MenuBar Load Runner guarantees $0\text{ pt}$ jitter:
1. **Monospaced Digits:** Uses `NSFont.monospacedDigitSystemFont(ofSize:weight:)`.
2. **Figure Space Padding (U+2007):** Padded with U+2007 (whose glyph width exactly matches numeric digits):
   ```swift
   func labelField(_ value: Double, ceiling: Double, decimals: Int) -> String
   ```
3. **Reserved Template Width (`labelSlotWidth`):** Computes maximum string dimension based on worst-case template bounds + `Tuning.labelSlotPadding` (4 pt). Slot length is fixed to this reservation.

### 6.2 Left/Right Placement Architecture & Menu Bar Congestion

macOS orders status items right-to-left based on creation time with no reordering API. To allow dynamic side switching at runtime without rebuilding the animation layer:
- Two label items are allocated at launch: `labelItemRight` (before animation item) and `labelItemLeft` (after animation item).
- The active side gets the computed reservation length; the inactive side is collapsed to `length = 0`.
- **Full Bar Adjacency Boundary (Scatter):** macOS owns status-item placement and provides no reorder or relative group-pinning API. Creation order decides *intent*; the WindowServer decides actual screen placement. On crowded menu bars (notably notched built-in displays with high item density), status items may scatter with foreign menu items interleaved between label and icon. The $0\text{ pt}$ jitter guarantee (§ 6.1) remains unaffected; `tests/qa.sh` §3c geometrically detects scatter and emits a `NOTE` rather than a spurious failure, as adjacency cannot be verified on a saturated bar.

### 6.3 Keep Awake Window Countdown Display

When Keep Awake is armed with a windowed duration (`keepAwakeDeadline != nil`):
- **Menu Bar Presentation:** The countdown timer (`MM:SS` or `HH:MM:SS`) is rendered in the active status bar slot (`activeLabelItem`).
  - When `labelMode == .off`: The slot dynamically reveals the countdown (e.g., `29:58`), collapsing back to length 0 upon timer expiry or disarming.
  - When `labelMode == .value` or `.custom`: Both telemetry/custom text and the countdown are displayed together (e.g., `CPU 45%  29:58` when placed left of the icon, or `29:58  CPU 45%` when placed right), positioning the countdown immediately adjacent to the runner icon.
- **Zero-Jitter Template Reservation:** `labelSlotWidth` accounts for the countdown template (`88:88` or `88:88:88`), ensuring that second-by-second decrements introduce $0\text{ pt}$ lateral shift.
- **1-Second Countdown Ticker:** A unified 1-second timer (`syncKeepAwakeCountdownTicker()`) drives live updates while a windowed countdown is active on the bar, stopping when disarmed or expired to preserve the self-throttling footprint.
- **Occlusion Gate:** The bar branch of that ticker reads the same `statusItemOccluded` verdict the frame driver does (§5), so a hidden item (notch, overflow, another Space, display off) costs 0 measure/relayout passes per second rather than 1 — a countdown exists to be looked at, and an 8-hour window is the case that makes the difference material. The *menu* branch is deliberately ungated: an open menu is its own window, visible whatever the status item is doing. On resume `updateAnimationForOcclusion()` redraws through `refreshKeepAwakeCountdown()` before restarting the timer, so the slot never shows the second it went dark on for up to a tick.

### 6.4 Click Dispatch & Modifier Routing

All three slots present the same `infoMenu`, but none of them owns it. A permanently attached
`NSStatusItem.menu` makes AppKit handle the mouse-down itself and never fires the button's action —
which is the only place a modifier can be read — so the menu is attached only for the duration of a
plain click and each slot carries `handleStatusItemClick(_:)` the rest of the time.

```
                [ Click on any of the three status item slots ]
                                      |
                                      v
                       handleStatusItemClick(_:) reads
                          NSApp.currentEvent modifiers
                                      |
                  +-------------------+-------------------+
                  |                                       |
        [ leftMouseUp + exactly Option ]        [ anything else ]
                  |                                       |
                  v                                       v
        toggleKeepAwakeQuick()                   presentInfoMenu(from:)
        (no menu is ever shown)                  item.menu = infoMenu
                  |                              button.performClick(nil)   <- modal
                  v                              item.menu = nil            <- on close
        arm/disarm via the submenu's                       |
        own two functions (§ 7.6)                          v
                                                 menuWillOpen / menuDidClose
                                                 fire unchanged (delegate is
                                                 on the menu, not the item)
```

- **Why not `popUpMenu(_:)`:** it does the same job without the attach/detach, but has been deprecated
  since macOS 11 and this build is gated warning-clean (`tests/qa.sh` § 1).
- **No re-entrancy:** `NSStatusItem` intercepts `performClick` below target/action, so the popup does not
  re-invoke `handleStatusItemClick`. The button's target/action also survives attaching and detaching
  `menu`, so there is nothing to re-wire — both verified against the real AppKit on macOS 26, not assumed
  from the widely-copied idiom, which asserts the opposite.
- **Modifier exclusivity:** the toggle requires *exactly* `.option` on `leftMouseUp`. Right-click and
  Control-click stay the conventional "show me the menu" gesture and must never arm anything; Command is
  left untouched so the system keeps its own drag-to-rearrange modifier.
- **Accepted cost:** an attached `menu` opens on mouse-*down*; an action fires on mouse-*up*. The dropdown
  therefore now appears on release rather than on press. This is the price of reading the modifier at all,
  and it is the same trade every status-bar app that supports modifier clicks makes.

---

## 7. Integrated Sleep Prevention (`SleepPreventer`) & Assertion Monitor

Sleep inhibition integrates directly into the visualizer while observing system-wide power management.

```
                                  +--------------------------+
                                  |   User Arms Keep Awake   |
                                  +------------+-------------+
                                               |
                                               v
                                  +--------------------------+
                                  | StateStore Persists      |
                                  | Target Deadline (JSON)   |
                                  +------------+-------------+
                                               |
                                               v
                                  +--------------------------+
                                  |  Evaluate Safety Release |
                                  +------------+-------------+
                                               |
               +-------------------------------+-------------------------------+
               |                                                               |
               v                                                               v
   [Battery ≤ 5% Hard Floor OR]                                    [Safe Operating State]
   [Battery ≤ Configured Threshold without Override]                           |
               |                                                               v
               v                                                   +---------------------------+
   KeepAwake Suspended / Paused                                    | Spawns Subprocess:        |
   (caffeinate terminated, intent kept)                            | caffeinate -di -w <pid>   |
                                                                   |            -t <seconds>   |
                                                                   +---------------------------+
```

### 7.1 Child Process Binding & Intent Separation

- **Subprocess Execution:** Spawns `/usr/bin/caffeinate -di -w <app_pid> [-t <seconds>]`.
- **Display + Idle Sleep:** Uses `-di` (preventing display and idle sleep).
- **Intent vs Running State:** `SleepPreventer` maintains `isEnabled` (user intent) separately from `isRunning` (child process active). When suspended by low battery or thermal events, intent remains set, and the child respawns automatically when conditions normalize.
- **Menu Bar Countdown Display:** Windowed assertions display a real-time seconds-resolution countdown on the active status item slot (see § 6.3).

### 7.2 Safety Floor & Configurable Thresholds

- **Configurable Battery Release:** Defaults to 20% (`Tuning.batteryLowThresholdDefault`), adjustable via CLI `--battery-threshold` or menu. The accepted range is **6%–100%** (`Tuning.batteryThresholdMin`…`batteryThresholdMax`), or `Never` / `0`; the menu offers 10 / 15 / 20 / 30% as rows plus `Custom…` for any whole percent in range. The minimum sits **above** the 5% floor on purpose, so no setting can reach it.
- **5% Hard Critical Floor:** `Tuning.batteryCriticalThreshold` (0.05). If battery drops to $\le 5\%$, all sleep assertions release immediately, overriding manual user overrides.
- **Low Battery Override Floor:** Arming Keep Awake while already below the threshold sets `keepAwakeBatteryOverride`, honoring user intent down to the 5% hard floor. An explicit "arm anyway" override is honored strictly between the configured threshold (e.g. 20%) and 5%; it deliberately never extends below the 5% physical floor into a hard power-off.

### 7.3 System Assertion Telemetry (`SleepAssertionMonitor`)

Reads machine-wide power management state via `IOPMCopyAssertionsByProcess`:
- **Noise Filtering:** `Tuning.assertionNoiseOwners` is **two named exceptions**, `powerd` and `WindowServer` — not an is-a-daemon test. (`powerd`'s assertion is literally "Prevent sleep while display is on", so it is present whenever anyone could read the menu.) `backupd` and `sharingd` are signal and must keep showing; don't grow this list into a policy.
- **Display vs Idle Classification:** Isolates display sleep assertions (`PreventUserIdleDisplaySleep`, `NoDisplaySleepAssertion`) from idle-only assertions.
- **Machine Hold State (`AwakeHold`):** Determines if any external application is keeping the Mac awake, reporting holder name and scheduled release times in the menu.
- **3-Tier Visual Indicator:**
  1. Full Tone ($1.0\alpha$): App's own sleep hold running.
  2. Foreign Tone ($0.45\alpha$, `keepAwakeBarForeignAlpha`): Mac held awake by an external process.
  3. Paused Tone ($0.22\alpha$, `keepAwakeBarPausedAlpha`): Armed but suspended due to battery/thermal conditions.

### 7.4 Process-Bound Windows (`--keep-awake-pid`)

A window whose end is an **event** rather than a clock: the hold is released when a named process
exits. That is the shape an unattended terminal job actually has — an agent run, a long build, a
render — where a fixed duration is a guess in both directions, and the guess that ends early is the
one that costs the job.

- **Exclusive with a timed window by construction.** `KeepAwakeLaunchOption.boundPID` is a third case
  beside `.off` / `.window`, so launch precedence stays the one switch in `applyLaunchKeepAwakeState`,
  and arming either kind clears the other's state. A bound hold therefore has no deadline, spawns
  `caffeinate` with no `-t`, and renders no countdown. When both flags are given the pid wins: it is
  the more specific intent, and the one whose stopping condition the caller can point at.
- **Release is event-driven with a poll underneath.** A `DispatchSourceProcess` `.exit` watch is
  primary — it fires the instant the target exits, and it is attached to the *process*, so a recycled
  pid cannot fool it. `ProcessProbe.isAlive` (`kill(pid, 0)`, where only `ESRCH` is death) is
  re-checked on the existing 2s sample tick to cover the one case the watch cannot: a registration
  that failed outright. A zombie still answers `kill(pid, 0)` until it is reaped, so the fallback can
  only ever fire late, never early. The exit ends the **intent**, not just the child, exactly as an
  elapsed window does.
- **Never resumed after a reboot, and never baked into a login item.** Pids are recycled, so a
  restored one could bind to an unrelated process or to nothing. This needs no rule of its own in the
  restore path: a binding saves as `enabled: true` with no `deadline`, which is already the one shape
  the restore refuses (§ 8.2). It *is* forwarded across an in-app restart, which does not reboot the
  Mac — the job is still running and its pid still means what it meant a second ago.
- **The subject is named on every surface** (`Keep Awake: claude (41293)`, and `until claude (41293)
  exits` in the status row). An unattributable hold is the exact problem § 7.3 exists to fix, and this
  app must not become that for its own.
- **A pid is not something a user has by hand,** so the menu prompt (`Keep Awake ▸ Until a process
  exits…`) takes a pid *or* a name, resolving a name to the newest match among the calling user's own
  processes (`ProcessProbe.newestMatch`, via `KERN_PROC_UID` + `proc_pidpath`). Newest is the right
  tie-break for the case this exists for — the job just started — and the pid it settled on is then
  shown, so a wrong guess is visible rather than silent. Same uid scope as the launcher's singleton
  guard, for the same reason: the menu bar is per-session.
- **The safety floor is untouched.** A bound hold suspends on the battery band and the 5% floor like
  any other (§ 7.2): the binding says when to stop holding, never that the floor stops applying.

### 7.5 OS Power & Multi-User Boundaries

- **Machine-Wide Sleep Semantics:** Sleep is a system-wide hardware/kernel state managed by `powerd`. A `caffeinate` assertion prevents the entire Mac from sleeping, so holds from two concurrent user sessions do not compose independently. User intent remains cleanly partitioned per user via per-account `state.json` (§ 8.2).
- **Fast User Switching GUI Isolation:** Windows and status items belonging to a background login session are invisible from the foreground session due to macOS WindowServer security boundaries. Background Keep Awake assertions deliberately continue running across user switches so unattended jobs finish without interruption.
- **Unprivileged Clamshell Sleep Boundary:** Subprocess `caffeinate -di -w <pid>` cannot prevent clamshell (closed-lid) sleep on battery power. Inhibiting clamshell sleep on battery requires mutating system-wide NVRAM power settings via root-privileged `pmset disablesleep`, which violates the unprivileged execution tenet and risks leaving sleep permanently disabled if the process terminates abnormally. Supported closed-lid operation requires Apple's standard clamshell conditions (AC power + external display).

### 7.6 Option-Click Quick Toggle (R21)

Option-clicking any of the three status item slots (§ 6.4) arms or disarms Keep Awake without opening
the dropdown, matching the modifier-click convention of the system's own Wi-Fi and Battery items.

The gesture is deliberately a **shortcut through** the submenu's two existing paths, never a second
implementation of them — `toggleKeepAwakeQuick()` calls `armKeepAwake(with:)` and `disarmKeepAwake()`,
the same functions the duration rows and the `Off` row call. That is what makes every guarantee below
inherited rather than re-argued:

- **The 5% floor still holds.** Arming routes through `updateSleepPrevention()` →
  `SleepPreventer.applyConditions(suspend:)`, and `KeepAwakeSuspension.batteryCritical` is not
  overridable by construction. No gesture can reach past it (§ 7.2).
- **The battery-override rule is unchanged.** The gesture passes `isUserGesture: true`, so it grants the
  arm-anyway override on exactly the same condition a menu click does: only while an overridable
  suspension is actually in force.
- **Single-writer persistence holds.** Both branches end in `persistState()` (§ 8.2).
- **The window is never invented.** The toggle arms `keepAwakeSelectedDuration` as it stands, which is
  `.indefinite` unless a window is live — disarming runs `clearKeepAwakeWindow()`, which resets the
  selection to `.indefinite`, the same default the submenu's first duration row carries. There is no
  separate "last duration" memory and no state added to `state.json`.

`disarmKeepAwake()` exists so that "off" has exactly one meaning across both surfaces: intent withdrawn,
any armed window dropped, any process binding released, and the arm-anyway override withdrawn with them.

**Verification boundary:** the gesture itself is not machine-testable here. Injecting a modified click
needs `CGEvent` posting under an Accessibility (TCC) grant, and every observability hook in this project
exists precisely to avoid requiring one. Its *effects* are covered — the arm and disarm paths are the
same ones `tests/qa.sh` § 3a/§ 3f already drive — but that Option-click, plain click and right-click each
route correctly is an eyes-and-hands check in the manual walk (§ 13), not a scripted assertion.

---

## 8. In-Menu Dashboard & State Persistence

### 8.1 Live Sparkline View (`LoadHistoryView`)

- Implemented as an `NSView` inside the status dropdown menu.
- Holds a circular buffer of 30 samples $\times$ 2s tick $\approx$ 60 seconds of history (`Tuning.loadHistoryCapacity`).
- Draws color-coded vertical bars:
  - Green: Load $< 0.30$ (`Tuning.cpuStateLowThreshold`)
  - Yellow: Load $0.30 \le x < 0.70$
  - Red: Load $\ge 0.70$ (`Tuning.cpuStateMediumThreshold`)

### 8.2 State Persistence (`StateStore`)

State is persisted to `~/Library/Application Support/menubar-load-runner/state.json`:

```json
// `deadline` is a Swift `Date` under JSONEncoder's default strategy: seconds since the
// 2001 reference date, NOT the Unix epoch.
{
  "version": 1,
  "keepAwake": {
    "enabled": true,
    "tint": 0,
    "deadline": 780000000.0
  },
  "settings": {
    "labelMode": "value",
    "labelSide": "left",
    "batteryThreshold": 0.20,
    "freezeAnimation": false
  }
}
```

- **Fail-Silent:** Corrupt, missing, or unwritable files fall back to system defaults without surfacing dialogs.
- **Atomic Persistence:** `data.write(to:options: .atomic)` — Foundation writes a temp file and renames it into place.
- **Single-Writer Rule:** `persistState()` is the sole disk writer, assembling memory state atomically to avoid race conditions.

---

## 9. Preset Registry & Self-Updating

### 9.1 Data-Driven Presets (`gifs/presets.json`)

Built-in preset identities are decoupled from Swift code into `gifs/presets.json`:

```json
{
  "defaultPreset": "horse-white",
  "presets": [
    {
      "key": "horse-white",
      "menuTitle": "Horse (White)",
      "file": "running-horse-white.gif",
      "speed": {
        "label": "horse",
        "min": 0.45,
        "max": 2.30,
        "responseExponent": 1.0
      }
    }
  ]
}
```

At launch, `JSONDecoder` hydrates `allPresets: [PresetDescriptor]`, determining menu items, keywords, and speed curves dynamically.

### 9.2 Git-Native In-App Update Engine

- **Update Probe (`UpdateChecker`):** Executes `git ls-remote --tags --refs origin 'v*'` against the origin remote, comparing the highest strict three-component SemVer against `AppInfo.version`.
- **Precompile Before Restart (`Builder`):** On user confirmation, runs `git pull --ff-only` followed by `menubar-load-runner --precompile`.
- **Supervisor-Preserving Relaunch (`Restarter`):** Dispatches a detached `/bin/sh` script waiting for the old process PID to terminate, then relaunches via either `launchctl kickstart` (for LaunchAgent jobs) or the original launcher command line.

---

## 10. Key Invariants & System Guardrails

| Subsystem | Hard Invariant | Architectural Rationale |
|---|---|---|
| **Process Model** | Single binary execution per UID (`pgrep -U`) | Prevents duplicate menu bar status items across accidental terminal launches while supporting Fast User Switching. |
| **SMC Access** | Exactly one `io_connect_t` instance | `SMCClient.shared` holds process-lifetime connection without opening redundant kernel handles. |
| **Private API** | `IOReport` bound only by `dlopen`/`dlsym`, never linked | The one unheadered API in the build. Runtime binding keeps the zero-dependency single-file compile intact and makes every absence (dylib, symbol, group, rail) a clean `isAvailable == false` instead of a launch failure (§ 4.5). |
| **Sleep Assertion** | Hard 5% critical battery floor | Sleep assertions unconditionally terminate at $\le 5\%$ battery, protecting laptop hardware from deep discharge (§ 7.2). |
| **Clamshell Sleep** | Unprivileged sleep inhibition | Uses PID-bound `caffeinate -di -w <pid>`; never mutates system-wide NVRAM sleep policy via `pmset` (§ 7.5). |
| **Menu Layout** | Static slot width reservation | Status items must never resize based on live data values to guarantee zero layout jitter on the menu bar (§ 6.1). WindowServer owns ultimate placement under congestion (§ 6.2). |
| **Game Loop** | Occlusion stops driver completely | Full occlusion (notch, inactive space, display off) must reduce render CPU utilization to exactly 0.0%. |
| **State File** | Single-writer centralized save | `persistState()` is the only function permitted to write `state.json`, eliminating partial block overwrites. |
| **Snapshot Path** | `--once` writes nothing and holds nothing | Side-effect freedom is what makes it safe beside a live instance and in parallel with itself; it is also why it is exempt from the singleton guard and the compile (§ 4.7). |
| **Telemetry Core** | Physical units out, no display concepts in | `TelemetryCore` never returns a 0..1 driver value from `snapshot()` and never reads AppKit, Keep Awake or `state.json`, so one set of readers serves both entry paths without either defining the other (§ 4.7). |

---

## 11. Tuning & Parameter Reference

Comprehensive reference of values defined in `Tuning`:

| Constant Name | Value | Unit | Functional Role |
|---|---|---|---|
| `loadSampleInterval` | `2.0` | Seconds | Telemetry sampling and dashboard refresh period |
| `snapshotWindow` | `0.2` | Seconds | Delta window between the two samples `--once` takes; the snapshot's whole latency budget (§ 4.7) |
| `cpuSmoothingAlpha` | `0.2` | Fraction | Exponential moving average alpha for CPU load smoothing |
| `speedUpdateHysteresis` | `0.08` | Fraction | Minimum load delta required to adjust animation speed |
| `constrainedSpeedCeilingFraction` | `0.5` | Fraction | Animation speed cap under thermal/power/memory pressure |
| `maxFrameAdvanceDelta` | `1.0` | Seconds | Threshold to trigger game loop clock resync after sleep/pause |
| `scalerWindow` | `5` | Samples | Window size for adaptive throughput rolling average |
| `scalerRescaleCount` | `5` | Samples | Hysteresis consecutive sample count before rate rescaling |
| `scalerHeadroomUp` | `1.3` | Multiplier | Ceiling headroom when expanding rate scale |
| `scalerHeadroomDown` | `3.0` | Multiplier | Floor headroom when contracting rate scale |
| `temperatureFloorCelsius` | `30.0` | °C | Lower anchor for die temperature speed mapping |
| `temperatureCeilingCelsius` | `100.0` | °C | Upper anchor for die temperature speed mapping |
| `memoryIdleFloor` | `0.55` | Fraction | Baseline RAM fraction subtracted before speed scaling |
| `aneFloorWatts` | `1.0` | Watts | Minimum Neural Engine speed-scale ceiling (keeps trivial inference off full speed) |
| `labelWattCeiling` | `9.9` | Watts | Reserved label width for the `ANE` readout |
| `batteryLowThresholdDefault` | `0.20` | Fraction | Default battery release point for Keep Awake (20%) |
| `batteryCriticalThreshold` | `0.05` | Fraction | Hard safety release floor for Keep Awake (5%) |
| `keepAwakeBarForeignAlpha` | `0.45` | Alpha | Opacity of track line when machine is held awake externally |
| `keepAwakeBarPausedAlpha` | `0.22` | Alpha | Opacity of track line when Keep Awake is armed but suspended |
| `assertionRetentionSeconds` | `8.0` | Seconds | Hysteresis retention time for external sleep assertion display |
| `assertionRowCap` | `4` | Rows | Maximum external assertion rows displayed before overflow row |
| `labelSlotPadding` | `4.0` | Points | Slack padding added to reserved status item label widths |

Parameters that are not `Tuning` constants because they live on an interface rather than in the binary's interior:

| Name | Default | Unit | Role | Lives in |
|---|---|---|---|---|
| `--once` | off | flag | Single-shot JSON snapshot; exclusive of every other argument (§ 4.7) | binary and launcher |
| `v` | `1` | int | Snapshot contract version | output |
| `MENUBAR_LOAD_RUNNER_FORCE_UNAVAILABLE` | unset | source keys | Marks readers unavailable; honored on both paths, so QA can assert an absent snapshot key | binary |
| `MENUBAR_LOAD_RUNNER_FORCE_BATTERY` | unset | `pct[:battery\|:ac]` | Pins charge and power state on the reader itself (§ 4.6); honored on both paths | binary |
| `MENUBAR_LOAD_RUNNER_FORCE_THERMAL` | unset | level | Pins the kernel thermal level; display-only, honored on both paths | binary |

`MENUBAR_LOAD_RUNNER_EXIT_AFTER` has no meaning on the snapshot path and is ignored there: it already exits on its own.

---

## 12. Capability Lineage (As-Shipped)

How the current architecture was reached. Each stage is complete and in the binary today; this section
exists so an implementer can see *why* a subsystem has the shape it has, without reading the release
history. Forward-looking candidates live in `docs/ROADMAP.md`, never here.

One line: a load *visualizer* that earns each new capability through unprivileged reads and
self-restraint — it only ever reads the system, and the only thing it throttles is itself.

| Stage | What landed | What it established |
|---|---|---|
| **v1.0** — the thesis | A GIF whose playback speed *is* the load readout: five unprivileged readers (CPU / memory / GPU / network / disk), btop-style adaptive scaling for unbounded rates, self-throttle under power/thermal/memory pressure | One source file, no bundle, no Xcode (§ 1, § 2). The read-only + self-throttle tenet everything since is measured against |
| **v1.6** — distribution without a bundle | MIT license, one-line installer, git-checkout self-update | The standing decision every later one leans on — no `.app`, no notarization, so the update engine is git-native (§ 9.2) and the interface stays CLI-first |
| **v1.8 → v1.13** — the first *action*: Keep Awake | From a checkbox spawning `caffeinate` to the intent/running split, timed windows, persistence across relaunches, and battery/thermal release | The visualizer learned to hold state responsibly: intent vs. running state (§ 7.1), the safety floor (§ 7.2), and single-writer persistence (§ 8.2) |
| **v1.10 → v1.16** — the second slot | The live-value label as its own status item: reserved width, figure-space padding, the no-jitter guarantee | The dual-slot layout model and the jitter-free contract (§ 6, § 6.1, § 6.2); the dropdown becomes a live dashboard (§ 8) |
| **v1.17 → v1.19** — from *our* hold to *the machine's* | Other sleep assertions, the machine-hold row, brightness-tracks-the-hold tint | Report the whole truth about sleep, not just this app's part of it (§ 7.3); the submenu's subject-grouped layout |
| **v1.20** — the sensor tier | A shared `SMCClient` opened fan, then die temperature | The family of hardware readings the app can keep growing through without privileges (§ 4.3) |
| **v1.21 → v1.22** — restart cost, and standing still | Build-before-restart in the update path; Freeze Animation honoring Reduce Motion (R17) | The compile moved out of the window where the app is gone (§ 9.2); a single stop/start decider total over occlusion + freeze (§ 5.1, § 5.3) |
| **v1.23** — a hold that isn't a guess | Keep Awake bound to a process instead of a clock (R19); the timed window counting down on the menu bar itself | A hold can take its end condition from the job rather than from a guessed duration (§ 7.4); the countdown became a glance, under the same occlusion gate the animation obeys (§ 6.3) |
| **v1.24** — the gesture, and the battery's own history | Option-click on any slot toggles Keep Awake without the menu (R21); the dropdown reports battery health, cycle count and capacity (R22); the temperature row names the kernel's own throttling (R23) | The first action reachable without opening anything — routed *through* the submenu's own arm/disarm so the 5% floor and the override rule cannot drift from it (§ 6.4, § 7.6); the first reading the app does not poll at all, gated entirely on menu open (§ 4.6); and the line between what the kernel does to the machine and what this app does about it, drawn in the menu and enforced in the wiring (§ 5.2) |

---

## 13. Release Hygiene & Delivery Discipline

A version bump moves five surfaces together: `AppInfo.version`, the `CHANGELOG.md` heading, the
`README.md` version line, the `docs/cover.html` badge, and the git tag `UpdateChecker` reads.
`tests/qa.sh` §2 enforces the first four; the tag is deliberately unchecked (qa.sh runs before it
exists), so **pushing the commit and tag is the last manual step**. One rule with no other home: **the
tag gates the prompt; the branch carries the code** — a behavioral change committed *after* its release
tag is an undeliverable fix (`UpdateChecker` sees equal versions and offers nothing), resolved only by
the next bump across all five surfaces. Never re-cut a published tag.

**Cutting a release, in this order.** Steps 2, 3 and 6 look skippable and are not — each fails
silently, and nothing upstream catches it:

1. Bump `AppInfo.version`; move `CHANGELOG.md`'s `[Unreleased]` items into a dated section.
   MAJOR/MINOR/PATCH follow the public-API definition at the top of `CHANGELOG.md` (CLI flags, env
   vars, preset keywords + `presets.json` schema, observable behavior).
2. Give `docs/cover.html` the release's **prose**, not just its badge, then redeploy (the
   `publish-cover` skill; `npx wrangler whoami` first — the OAuth token usually persists, so no
   interactive login is needed). qa.sh §2 greps the badge and nothing else, so a cover describing the
   *previous* release passes every check while the badge reads as proof it is current. Its interactive
   demos need the same pass and fail differently: they model app behavior in JS, so a stale demo goes
   on *reproducing* the bug this release fixed — prose drift reads as stale, a stale demo reads as a
   specification. Click the demo for whatever you changed.
3. **Re-run `tests/qa.sh --core`** — §2 can only compare the version surfaces *after* you have edited
   them; skip it and a mistyped surface ships silently.
4. Tag `vX.Y.Z` — the fifth surface, and the only unchecked one.
5. Push the commit **and** the tag: `git push origin main vX.Y.Z`.
6. **Confirm it shipped with the command the app itself polls:**
   ```bash
   git ls-remote --tags origin 'v*' | sed 's|.*refs/tags/||; s|\^{}||' | sort -uV | tail -3
   ```
   A tag that never left your machine tells every installed copy the old version is current, silently,
   with the local repo looking fully released. Sort `-V`, never plain `sort`: lexically `v1.9.1` beats
   `v1.19.2`, which stopped being hypothetical at `v1.10.0`.

Ship when `tests/qa.sh` says ALL PASS, and **no NOTE covers what
this release changed** — a NOTE is an unanswered case, not an accepted one, and neighbouring cases
passing is not cover for it. Before signing off, check for a leaked keep-awake child by its `-w <pid>`
signature, never by name (your own instance holds one legitimately), with `pgrep -fl caffeinate | grep -- "-w <pid>"`:

```bash
pgrep -fl caffeinate 2>/dev/null | grep -- '-w ' | while read -r _ pid; do
  ps -p "$pid" >/dev/null 2>&1 || echo "  LEAK: caffeinate -w $pid"; done
```

Don't record the cover's published state in docs — fetch it. A `curl` of the live badge settles it in one command; the flow, including the two
checks that read as a failed deploy and aren't, is the `publish-cover` skill's.
