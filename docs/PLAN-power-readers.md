# PLAN — Hardware Power & Apple Neural Engine (ANE) Telemetry Readers (R10)

Status: implementation-ready proposal, grounded 2026-09-11. No runtime code has landed yet.

Lifecycle: when R10 is implemented and verified, distill the durable result into
`docs/ARCHITECTURE.md`, update the public surfaces, remove R10 from `docs/ROADMAP.md`, and immediately
`git rm` this file. This plan is not an as-built claim.

---

## 1. Decision Summary

R10 expands MenuBar Load Runner's telemetry suite beyond utilization percentages and I/O rates by
introducing real-time hardware power readers in Watts, specifically targeting:

1. **Apple Neural Engine Power / Load (`--load-source ane`)**: Measures active power draw and utilization
   of Apple Silicon's Neural Engine (NPU). This delivers a genuinely novel, non-redundant signal:
   when local AI models (e.g. Apple Intelligence, CoreML, MLX with ANE backends, Whisper, Stable
   Diffusion) execute on the NPU, CPU and GPU utilization remain near 0%, making existing visualizers
   appear completely idle. A dedicated ANE source turns NPU inference activity into a visible load runner.
2. **SoC Package Power (`--load-source power`)**: Measures total Apple Silicon SoC package power
   (the sum of CPU, GPU, ANE, DRAM, and fabric energy rails) in Watts. Unlike the existing Battery source
   (which only reports discharge current on battery and is unavailable or zero on AC), Package Power
   provides continuous power visibility on both AC power and battery.

### Core Architectural Invariants Maintained:
- **Zero Root / Zero Sudo / Unprivileged Execution**: Does not call `powermetrics` (which requires `sudo`),
  does not load kernel extensions, and does not alter system power policies. Reads exclusively through
  the unprivileged user-space `IOReport` private framework.
- **Unbundled Single-File Swift**: All logic resides within `MenuBarLoadRunner.swift`. Private symbols
  are resolved via dynamic linking (`dlopen`/`dlsym`) with type-safe `@convention(c)` function pointers,
  preserving strict compiler independence with `swiftc -O -strict-concurrency=complete` and zero warnings.
- **Single-Subscription Shared Client (`IOReportClient`)**: Follows the precedent of `SMCClient`: exactly
  one long-lived subscription manages energy channel sampling, avoiding redundant kernel channel queries.
- **Adaptive Normalization (`ThroughputScaler`)**: Power in Watts is an unbounded stream metric that
  varies significantly across chip tiers (e.g. 20W max on base M-series, 60–100W on M-Max, 150W+ on
  M-Ultra). Using `ThroughputScaler` with asymmetric headroom and hysteresis counters normalizes Watts
  into a smooth 0…1 animation speed range without artificial hardcoded caps.
- **Graceful Hardware Degradation**: Non-Apple-Silicon systems (Intel Macs) or systems where `IOReport`
  energy channels are absent probe `isAvailable == false` at launch, cleanly disabling the menu items
  and falling back to CPU if requested via CLI.

---

## 2. Grounded Current Behavior

The plan builds on these existing symbols and architectural conventions in `MenuBarLoadRunner.swift`:

- `LoadSource` (enum, Int, CaseIterable): Defines the 8 existing sources:
  `cpu (0)`, `memory (1)`, `gpu (2)`, `network (3)`, `disk (4)`, `fan (5)`, `battery (6)`, `temperature (7)`.
  Each case defines `key` (CLI/config token) and `menuTitle`.
- `LoadSource.from(key:)`: Case-insensitive factory mapping CLI strings to enum cases.
- `MenuBarLoadRunnerApp.sampleSource(_:elapsed:)`: Central sampling switch routing to respective monitor
  classes (`loadMonitor`, `memoryMonitor`, `gpuMonitor`, `networkMonitor`, `diskMonitor`, `fanMonitor`,
  `batteryMonitor`, `temperatureMonitor`).
- `ThroughputScaler`: Used by `NetworkLoadMonitor` (bytes/s), `DiskLoadMonitor` (bytes/s), and
  `MemoryLoadMonitor` (swap bytes/s) to track rolling maximums, apply asymmetric scaling headroom
  (1.3x up, 3.0x down), and debounce transient spikes.
- `SMCClient`: Precedent for shared, low-level hardware access. Demonstrates safe dynamic probing,
  single-instance lifetime, structured binary layout guards, and zero-allocation point sampling.
- `GPULoadMonitor`: Precedent for capability probing:
  `var isAvailable: Bool` checks hardware availability once, caches the result, and disables the menu item
  when unavailable. If specified via CLI on an unsupported machine, `Config.loadSource` falls back to CPU.
- `MenuBarLabel`: Handles secondary status-item slots (`--label value`). Formatters use monospaced digits,
  fixed-character templates, and figure-space padding (U+2007) to guarantee zero menu-bar jitter.
- `LoadHistoryView`: 60-second in-menu sparkline (30 samples × 2s) displaying normalized 0…1 fractions
  color-coded by state (green/yellow/red).

---

## 3. Goals and Non-Goals

### 3.1 Goals
- Add `LoadSource.ane` (`ane`) and `LoadSource.power` (`power` / `pkg-power`) as fully integrated sources.
- Dynamically bind `IOReport` functions at runtime via `dlopen`/`dlsym`, requiring zero C bridging headers
  or external build configuration.
- Implement `IOReportClient` as a shared singleton managing the channel subscription and delta samples.
- Compute delta energy over elapsed wall time: $P = \frac{\Delta E}{\Delta t}$ in Watts.
- Map unbounded wattage to 0…1 animation speed using `ThroughputScaler`.
- Provide compact status-bar labels: `ANE 4.2W`, `PWR 18.5W` with fixed template reservation.
- Display detailed in-menu breakdowns:
  - For ANE: live wattage, peak wattage, rolling average.
  - For Package Power: total package wattage with breakdown: `CPU: X.X W · GPU: Y.Y W · ANE: Z.Z W`.
- Provide complete fail-safe degradation on Intel Macs and unsupported hardware without logging errors
  or throwing runtime traps.
- Ensure strict zero-leak memory management by releasing all CoreFoundation dictionaries and arrays
  created during `IOReport` sampling.

### 3.2 Non-Goals
- No `sudo` or privileged helper daemon (rejects `powermetrics` subprocess model).
- No per-process energy attribution (avoids walking task lists, which violates self-throttling tenets).
- No private Intel MSR / RAPL kernel drivers.
- No continuous high-frequency polling (< 1 Hz); sampling aligns with the existing 2-second telemetry tick.
- No battery health cycle counts or system voltage profiling.

---

## 4. Technical Architecture: `IOReport` Integration

### 4.1 The `IOReport` Subsystem
Apple Silicon exposes hardware energy counters to non-root user processes through the private framework
`/System/Library/PrivateFrameworks/IOReport.framework/IOReport`.

Unlike Mach point reads (`host_statistics64`), `IOReport` is a **stateful delta-sampling API**:
1. Query available channels in the `"Energy Model"` group.
2. Create an `IOReportSubscriptionRef`.
3. Take Sample $S_1$ at time $t_1$.
4. Take Sample $S_2$ at time $t_2$.
5. Generate Delta $D = S_2 - S_1$.
6. Extract energy increments $\Delta E$ per channel, convert to Watts via duration $\Delta t = t_2 - t_1$,
   and advance $S_1 \leftarrow S_2$.

### 4.2 Dynamic Linking & Type Definitions
To preserve the single-file Swift architecture without Xcode headers, all symbols are resolved dynamically:

```
+------------------------------------------------------------------------------------+
|                         IOReport Dynamic Binding Architecture                      |
|                                                                                    |
|  /System/Library/PrivateFrameworks/IOReport.framework/IOReport                     |
|                                     |                                              |
|                                dlopen()                                            |
|                                     v                                              |
|                       IOReportPrivateSymbols (struct)                              |
|   +-----------------------------------------------------------------------------+  |
|   | copyChannelsInGroup       : @convention(c) (CFString, ...) -> CFDictionary  |  |
|   | mergeChannels             : @convention(c) (CFDictionary, ...) -> Void      |  |
|   | createSubscription        : @convention(c) (UnsafeRawPointer?, ...) -> Ref  |  |
|   | createSamples             : @convention(c) (Ref, CFMutableDictionary) -> CF |  |
|   | createSamplesDelta        : @convention(c) (CFDictionary, ...) -> CFDict    |  |
|   | channelGetGroup           : @convention(c) (CFDictionary) -> CFString       |  |
|   | channelGetChannelName     : @convention(c) (CFDictionary) -> CFString       |  |
|   | channelGetUnitLabel       : @convention(c) (CFDictionary) -> CFString       |  |
|   | simpleGetIntegerValue     : @convention(c) (CFDictionary, Int32) -> Int64   |  |
|   +-----------------------------------------------------------------------------+  |
+------------------------------------------------------------------------------------+
```

Dynamic function signatures:
```swift
private typealias IOReportCopyChannelsInGroupFunc = @convention(c) (
    CFString, CFString?, UInt64, UInt64, UInt64
) -> Unmanaged<CFDictionary>?

private typealias IOReportMergeChannelsFunc = @convention(c) (
    CFDictionary, CFDictionary, CFTypeRef?
) -> Void

private typealias IOReportCreateSubscriptionFunc = @convention(c) (
    UnsafeMutableRawPointer?,
    CFMutableDictionary,
    UnsafeMutablePointer<Unmanaged<CFMutableDictionary>?>?,
    UInt64,
    CFTypeRef?
) -> UnsafeMutableRawPointer?

private typealias IOReportCreateSamplesFunc = @convention(c) (
    UnsafeMutableRawPointer, CFMutableDictionary, CFTypeRef?
) -> Unmanaged<CFDictionary>?

private typealias IOReportCreateSamplesDeltaFunc = @convention(c) (
    CFDictionary, CFDictionary, CFTypeRef?
) -> Unmanaged<CFDictionary>?

private typealias IOReportChannelGetGroupFunc = @convention(c) (
    CFDictionary
) -> Unmanaged<CFString>?

private typealias IOReportChannelGetChannelNameFunc = @convention(c) (
    CFDictionary
) -> Unmanaged<CFString>?

private typealias IOReportChannelGetUnitLabelFunc = @convention(c) (
    CFDictionary
) -> Unmanaged<CFString>?

private typealias IOReportSimpleGetIntegerValueFunc = @convention(c) (
    CFDictionary, Int32
) -> Int64
```

### 4.3 Energy Channel Identification & Aggregation
In the `"Energy Model"` group, Apple Silicon energy channels appear under standard naming conventions
across M1, M2, M3, M4, and M5 generations:

| Channel Pattern | Hardware Domain | Aggregation Target |
|---|---|---|
| `CPU Energy` / `*CPU*` | Efficiency + Performance CPU clusters | `cpuWatts` |
| `GPU Energy` / `*GPU*` | Integrated Graphics Core | `gpuWatts` |
| `ANE*` / `ANE Energy` / `ANE0 Energy` | Apple Neural Engine (NPU) | `aneWatts` |
| `DRAM Energy` / `*DRAM*` | Unified Memory Controller | `dramWatts` |
| `GPU SRAM Energy` | Graphics L2 / SRAM cache | `gpuWatts` (or package total) |

**Total SoC Package Power**:
$$\text{Package Power (Watts)} = P_{\text{CPU}} + P_{\text{GPU}} + P_{\text{ANE}} + P_{\text{DRAM}} + P_{\text{SRAM}}$$

### 4.4 Unit Normalization
Raw energy increments $\Delta E$ are scaled by unit label string:
- `"mJ"`: $\Delta E \times 10^{-3} \text{ Joules}$
- `"uJ"`: $\Delta E \times 10^{-6} \text{ Joules}$
- `"nJ"`: $\Delta E \times 10^{-9} \text{ Joules}$

$$P = \frac{\Delta E_{\text{Joules}}}{\Delta t_{\text{Seconds}}} \quad (\text{Watts})$$

---

## 5. Detailed Component Design

### 5.1 `IOReportClient` (Shared Core Client)
A singleton class `@MainActor private final class IOReportClient` manages:
- One-time `dlopen` of `IOReport.framework`.
- Channel subscription creation targeting `"Energy Model"`.
- Stateful sample retention: `previousSample: CFDictionary?`, `previousTimestamp: Double?`.
- Delta sample calculation on every 2-second tick.
- Structured output struct:
  ```swift
  struct PowerSample {
      let cpuWatts: Double
      let gpuWatts: Double
      let aneWatts: Double
      let dramWatts: Double
      let totalPackageWatts: Double
      let elapsedSeconds: Double
  }
  ```
- Graceful error cleanup: if `IOReportCreateSubscription` returns nil (e.g. Intel Macs), `isAvailable`
  becomes `false` and subsequent calls immediately return `nil` without overhead.

### 5.2 `ANELoadMonitor` & `PackagePowerLoadMonitor`
Two lightweight monitor classes consume `IOReportClient`:

```swift
@MainActor
private final class ANELoadMonitor {
    private(set) var currentWatts: Double = 0
    private(set) var currentLoad: Double = 0
    private(set) var hasSample = false
    private var scaler = ThroughputScaler(floor: Tuning.aneFloorWatts)

    var isAvailable: Bool { IOReportClient.shared.isAvailable && IOReportClient.shared.hasANEChannel }

    func sampleUsage(elapsed: Double?) -> Double? {
        guard let sample = IOReportClient.shared.sampleEnergy(elapsed: elapsed) else {
            hasSample = false
            return nil
        }
        currentWatts = sample.aneWatts
        hasSample = true
        currentLoad = scaler.normalize(speed: currentWatts)
        return currentLoad
    }
}

@MainActor
private final class PackagePowerLoadMonitor {
    private(set) var currentWatts: Double = 0
    private(set) var currentLoad: Double = 0
    private(set) var currentBreakdown: (cpu: Double, gpu: Double, ane: Double, dram: Double) = (0, 0, 0, 0)
    private(set) var hasSample = false
    private var scaler = ThroughputScaler(floor: Tuning.packagePowerFloorWatts)

    var isAvailable: Bool { IOReportClient.shared.isAvailable }

    func sampleUsage(elapsed: Double?) -> Double? {
        guard let sample = IOReportClient.shared.sampleEnergy(elapsed: elapsed) else {
            hasSample = false
            return nil
        }
        currentWatts = sample.totalPackageWatts
        currentBreakdown = (sample.cpuWatts, sample.gpuWatts, sample.aneWatts, sample.dramWatts)
        hasSample = true
        currentLoad = scaler.normalize(speed: currentWatts)
        return currentLoad
    }
}
```

### 5.3 Normalization Tuning Baseline
Added to `enum Tuning`:
- `static let aneFloorWatts: Double = 0.05`: Ignores micro-watt leakage noise when the NPU is idle.
- `static let packagePowerFloorWatts: Double = 0.5`: Baseline idle floor for total SoC power.
- `ThroughputScaler` parameters reuse standard headroom (`scalerHeadroomUp = 1.3`, `scalerHeadroomDown = 3.0`)
  and sample window (`scalerWindow = 5`), ensuring smooth transitions as heavy model workloads begin or end.

### 5.4 Menu Bar Label & Dropdown Readout

1. **Status Bar Value Mode (`--label value`)**:
   - For ANE: `ANE 4.2W`, `ANE 12.8W` (or `ANE 0.0W` when idle).
   - For Package Power: `PWR 18.4W`, `PWR 45W`.
   - Reserved template width ensures zero menu-bar jitter.

2. **In-Menu Diagnostic Block**:
   ```
   [Active: ANE]
   +---------------------------------------------+
   | [=== 60s ANE Load History Sparkline ======] |
   | ANE Power: 6.4 W   (Load: 48%, Speed: 2.1x) |
   | State: Normal · Load Average: 1.82 1.45 1.10|
   +---------------------------------------------+

   [Active: Package Power]
   +---------------------------------------------+
   | [=== 60s Package Power Sparkline =========] |
   | Package: 24.2 W   (Load: 55%, Speed: 2.3x)  |
   | CPU: 12.1W · GPU: 4.8W · ANE: 6.4W · RAM: 0.9W|
   | State: Normal · Load Average: 1.82 1.45 1.10|
   +---------------------------------------------+
   ```

---

## 6. Failure Modes, Edge Cases & Guardrails

| Failure Mode / Edge Case | Risk | Mitigation Strategy |
|---|---|---|
| **Intel Mac (x86_64) Hardware** | `IOReport` lacks `"Energy Model"` group; calls would return null | `IOReportClient.isAvailable` returns false on first check. Menu items disabled; CLI flag falls back cleanly to CPU with stdout note. |
| **System Sleep / Wake Discontinuity** | Time gap $\Delta t$ spans hours; raw $\Delta E / \Delta t$ or clock skew produces erratic spike | If `elapsed > 5.0` seconds (e.g. system wake from sleep), discard the delta, record the new baseline, and return nil for that tick. |
| **Dynamic Framework Relocation** | Future macOS changes private framework location | `dlopen` checks fallback paths; if all fail, gracefully sets `isAvailable = false`. Never crashes or triggers fatalError. |
| **CoreFoundation Memory Leaks** | Continuous 2s allocation of CFDictionary/CFArray results in memory bloat | Every call to `IOReportCreateSamples` and `IOReportCreateSamplesDelta` wrapped with explicit `Unmanaged.release()` / `CFRelease`. Verified via memory footprint assertions. |
| **Zero ANE Activity** | During pure CPU tasks, ANE counter delta is 0 | When $\Delta E \le \text{aneFloorWatts}$, driver returns 0.0, speed sits at minimum preset rate, sparkline stays at baseline. |
| **Simultaneous Load Sources** | User switches between ANE and Package Power in menu | `IOReportClient` is shared; both monitors read from the single synchronized delta sample without double-querying the kernel. |

---

## 7. Symbol-Level Implementation Order

When scheduled for implementation, modifications will proceed in this strict order:

1. **`enum Tuning` additions**: Define `aneFloorWatts` and `packagePowerFloorWatts`.
2. **`IOReportClient` implementation**:
   - Dynamic linking loader for `/System/Library/PrivateFrameworks/IOReport.framework/IOReport`.
   - Subscription creation and delta parsing.
   - CoreFoundation memory release guarantees.
3. **`ANELoadMonitor` & `PackagePowerLoadMonitor` implementation**:
   - Integration with `ThroughputScaler`.
   - Availability checking (`isAvailable`).
4. **`enum LoadSource` expansion**:
   - Add `.ane = 8` and `.power = 9`.
   - Add string mappings: `"ane"` and `"power"`.
   - Update `menuTitle`: `"Apple Neural Engine"` / `"ANE"`, `"Package Power"`.
5. **App Wiring (`MenuBarLoadRunnerApp`)**:
   - Instantiate monitors in app initialization.
   - Extend `sampleSource`, `activeSourceHasSample`, `activeSourceCurrentUsage`.
   - Update `refreshMenuMetrics()` to render power rows and rail breakdowns.
   - Update `menuBarLabelText()` for `ANE` and `PWR` formatting.
6. **CLI & Launcher Updates**:
   - Add `--load-source ane` and `--load-source power` to help text and option parser.
   - Update shell completion / launcher help in `menubar-load-runner`.

---

## 8. Real-Binary QA & Verification Matrix

In accordance with project testing tenets (Zero Mocks / Real Binary Assertions), testing will be
implemented in `tests/qa.sh`:

### 8.1 Headless Core Gate (`tests/qa.sh --core`)
- Verify warning-free build under `swiftc -O -strict-concurrency=complete`.
- Verify CLI acceptance:
  - `./tmp/mblr-check --load-source ane --help` exits 0.
  - `./tmp/mblr-check --load-source power --help` exits 0.
- Verify invalid option rejection:
  - `./tmp/mblr-check --load-source invalid_source` exits 1.

### 8.2 Real-Binary Hardware Verification
- **On Apple Silicon**:
  - Run with `MENUBAR_LOAD_RUNNER_EXIT_AFTER=4 ./tmp/mblr-check --load-source power`:
    Must sample successfully (`hasSample == true`), report wattage > 0W in debug log, and exit 0.
  - Run with `MENUBAR_LOAD_RUNNER_EXIT_AFTER=4 ./tmp/mblr-check --load-source ane`:
    Must sample successfully and exit 0 without crashing.
  - Label verification:
    Run with `--label value --load-source power`: assert label text matches regex `^PWR [0-9]+(\.[0-9]+)?W$`.
- **On Intel Hardware / Non-Apple Silicon CI**:
  - Verify that `--load-source ane` prints the standard availability fallback NOTE and defaults to CPU
    without crashing or asserting.

### 8.3 Memory Stability Verification
- Run process under sustained 60-second telemetry polling.
- Assert resident memory (RSS) remains completely flat ($\Delta \text{RSS} < 500\text{KB}$ over 30 ticks),
  proving zero CoreFoundation leaks in the delta calculation loop.
