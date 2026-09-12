# RESEARCH — Peer Survey: macOS Status Bar Load Visualizers & Telemetry Monitors

> **Scope:** Comprehensive technical peer survey and architectural comparison of open-source macOS status bar system monitors, load visualizers, and sleep inhibitors (≥100 GitHub stars) in 2026.
> **Peer set:** `RunCatNeo` (runcat-dev), `menubar_runcat` (Kyome22), `stats` (exelban), `Hot` (macmade), `Fanny` (DanielStormApps), `SiliconScope` (kennss), `KeepingYouAwake` (newmarcel), `adrafinil` (kageroumado), `modafinil` (narcotic-sh).
> **Methodology:** Public README / documentation review, live GitHub repository metadata checks (stars, tags, release histories), plus
> **direct source-code inspection of `newmarcel/KeepingYouAwake` 1.6.8 only** (§6). Everything said about the
> other peers is documentation-level and repository-metadata-level, not source-verified, and none of them was runtime-profiled — so no
> claim here is a category-first claim.
> **Snapshot:** September 11, 2026 (live re-verification via GitHub API and web inspection; strictly pruned to established peers with ≥100 stars).
> Star counts, activity dates, and feature sets are point-in-time reads of other people's projects and **go stale on their schedule, not ours**.
> **Re-check or prune before 2027-03-01**, and before any public use of a comparative claim: confirm each
> peer's current release, then either re-date this line or delete the rows you did not re-verify. A survey
> that has silently aged is worse than no survey — per `CLAUDE.md`, a research file is evidence, and
> evidence carries the date it was taken.

---

## 1. Executive Summary & 2026 Ecosystem Context

The macOS menu bar customization and system-monitoring category has experienced a developer-centric renaissance through 2026. While the proprietary App Store application **RunCat** remains the popular standard-bearer for CPU-driven status bar animations, open-source developers have built specialized alternatives addressing distinct architectural niches.

In 2026, the ecosystem exhibits four primary evolutions:

1. **Modernization & Extensibility of Animated Visualizers:** The launch of **RunCat Neo** (`runcat-dev/RunCatNeo`, 841 stars, v1.0.3) represents the active open-source successor from the RunCat ecosystem. Written in Swift 6.2 for macOS 26 Tahoe, it introduces **Custom Metrics** (polling local JSON files to visualize AI agent token usage such as Claude Code and Codex, cryptocurrency prices, or custom scripts) alongside a community **Runner Gallery** for pixel-art keyframes.
2. **Major Milestone in General Monitors:** The premier open-source system monitor, **Stats** (`exelban/stats`), reached 41,742 GitHub stars and released **v3.0.0** (latest v3.0.15 in September 2026), initiating a new architectural chapter for comprehensive multi-module telemetry.
3. **Specialization in Power Management & Sleep Supervision:** Beyond classic `caffeinate` GUI wrappers like **KeepingYouAwake** (6,898 stars, v1.6.8), the category has branched into specialized sub-genres:
   - **AI-agent session guards:** Tools like **adrafinil** (`kageroumado/adrafinil`, 471 stars) keep Macs awake specifically while autonomous AI coding agents are executing tasks.
   - **Closed-lid/clamshell sleep management:** Utilities like **modafinil** (`narcotic-sh/modafinil`, 151 stars) manage power assertions and prevent sleep when MacBook lids are closed during long-running background tasks.
4. **App Store Sandboxing Backlash & Specialized Unprivileged Telemetry:** Power users and terminal-centric developers increasingly favor lightweight, unbundled, CLI-driven utilities that run without Xcode project overhead, script cleanly via user LaunchAgents, and read unprivileged Mach, IOKit, and SMC telemetry directly. Tools like **SiliconScope** (`kennss/SiliconScope`, 943 stars) demonstrate growing demand for sudoless Apple Silicon NPU (ANE) and memory bandwidth visibility, while **Hot** (`macmade/Hot`, 3,038 stars) and **Fanny** (`DanielStormApps/Fanny`, 1,384 stars) focus on thermal throttling and fan RPM.

Within this landscape, **MenuBar Load Runner** (v1.23.0) occupies a specialized, command-line-first niche: an unbundled single-file Swift script backed by a Zsh launcher, featuring adaptive rate scaling (`btop` hysteresis), power-throttling, occlusion-awareness, machine-wide sleep assertion inspection, process-bound sleep inhibition (`--keep-awake-pid`), and a direct menu-bar countdown readout.

---

## 2. Peer Catalog (≥100 Stars)

### 2.1 MenuBar Load Runner (Baseline Profile)

* **What it is:** A native, CLI-first macOS status bar load visualizer and lightweight diagnostic monitor for developers and power-users, written in unbundled Swift and AppKit.
* **Core mechanics:** Animates a dynamic GIF in the status bar whose playback speed correlates with a smoothed hardware metric across 8 unprivileged readers (CPU, memory+swap, GPU, network, disk, fan speed, battery discharge, die temperature). Full architectural specification: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md).
* **Distinctive profile:**
  - Single-file unbundled build compiled via `swiftc -O -strict-concurrency=complete`; zero external package or Xcode project overhead.
  - Adaptive `ThroughputScaler` (ported from Linux `btop`) normalizing unbounded streams (network, disk, swap) with asymmetric headroom and hysteresis counters.
  - Power-aware 0% CPU occlusion pausing under the notch, off-screen, or display sleep; self-throttles under thermal or battery pressure; honors Reduce Motion and manual freeze.
  - Integrated Keep Awake: PID-bound `caffeinate -di -w <pid>`, process-bound exit watch (`--keep-awake-pid`), direct status-bar countdown surface, and machine-wide foreign assertion inspection (`IOPMCopyAssertionsByProcess`).
* **Trade-offs:** No graphical settings window (configured via CLI/env or menu dropdown), source-based distribution, and static disk GIF registry (first-class local import planned in R9).

### 2.2 runcat-dev / RunCatNeo (841 Stars, v1.0.3)

* **What it is:** The active open-source next-generation RunCat application for macOS, launched in 2026 by the RunCat developer community (`runcat-dev`, Kyome22).
* **Core mechanics:** Animates a running cat or community runner in the menu bar based on system CPU metrics, while supporting external JSON metric cards.
* **Strengths:** Built with Swift 6.2 and the LUCA architecture for macOS 26 Tahoe; Custom Metrics engine polling user-specified JSON files (e.g. Claude Code token usage, Codex sessions, cryptocurrency); centralized Runner Gallery for pixel-art keyframes; distributed via Mac App Store and GitHub.
* **Weaknesses:** Requires full Xcode 26.5+ build environment; packaged as compiled `.app` bundle; core animation remains tied to CPU percentage thresholds (no adaptive rate scaling for unbounded streams); lacks sleep management, process-exit watches, CLI flags, or headless automation.

### 2.3 Kyome22 / menubar_runcat (510 Stars)

* **What it is:** A lightweight, reduced open-source edition of the original App Store RunCat app in native Swift and AppKit.
* **Core mechanics:** Animates a running cat in the menu bar; frame interval scales inversely with CPU load.
* **Strengths:** Historical pedigree; minimal memory and CPU footprint due to stripped-down AppKit codebase.
* **Weaknesses:** Strictly limited to CPU load; archived read-only repository (superseded by RunCat Neo); requires Xcode to compile; `.app` bundle packaging.

### 2.4 exelban / stats (41,742 Stars, v3.0.15)

* **What it is:** The benchmark open-source macOS system monitor.
* **Core mechanics:** Comprehensive multi-module dashboard monitoring CPU (per-core), GPU, RAM, network, disks, battery health, and fans.
* **Strengths:** Modular architecture rewritten in v3.0; extensive graphical plots; detailed per-process breakdowns; highly configurable.
* **Weaknesses:** Heavy runtime footprint (multi-module framework with persistent background daemon); no load-scaled animation; no integrated sleep management; GUI-centric configuration.

### 2.5 macmade / Hot (3,038 Stars, v1.9.4)

* **What it is:** Specialized native menu bar utility monitoring CPU thermal limits.
* **Core mechanics:** Checks system temperature sensors and detects whether macOS kernel is throttling CPU clock frequency due to thermal or electrical constraints.
* **Strengths:** Highly focused single-purpose diagnostic; minimal memory overhead.
* **Weaknesses:** Narrow scope (thermal throttling only); no rate scaling, animations, or sleep management.

### 2.6 DanielStormApps / Fanny (1,384 Stars)

* **What it is:** Compact macOS status bar and Notification Center widget for cooling diagnostics.
* **Core mechanics:** Reads SMC fan speeds (RPM) and CPU/GPU temperatures.
* **Strengths:** Clean visual layout; lightweight footprint for hardware cooling checks.
* **Weaknesses:** Limited telemetry scope (fans and temperatures only); legacy project architecture; no CLI or power automation.

### 2.7 kennss / SiliconScope (943 Stars)

* **What it is:** Sudoless Apple Silicon system monitor with deep hardware telemetry.
* **Core mechanics:** Queries `IOReport` and `IOHIDEventSystem` without root privileges to track Apple Neural Engine (ANE) power, Media Engine activity, and unified memory bandwidth alongside CPU and GPU metrics.
* **Strengths:** Native SwiftUI interface; surfaces hardware signals invisible to standard Activity Monitor (ANE power, RAM bandwidth) without `sudo`.
* **Weaknesses:** Packaged as full Xcode `.app` bundle; popover GUI only (no animated load visualizer); no CLI automation or sleep inhibition.

### 2.8 newmarcel / KeepingYouAwake (6,898 Stars, v1.6.8)

* **What it is:** Dedicated macOS menu bar sleep inhibitor wrapping `caffeinate`.
* **Core mechanics:** Toggles display and idle sleep prevention via subprocess `caffeinate -di -w <pid>`, supporting timed windows and custom durations.
* **Strengths:** Mature, polished UI; custom URL scheme for automation; localization across 21 languages; multi-tab Preferences window.
* **Weaknesses:** No telemetry or load visualization; no process-bound exit triggers; requires menu traversal to inspect remaining countdown time (analyzed in depth in §6).

### 2.9 kageroumado / adrafinil (471 Stars) & narcotic-sh / modafinil (151 Stars)

* **What they are:** Specialized modern keep-awake utilities tailored for automated workflows.
* **Core mechanics:** **adrafinil** monitors autonomous AI coding agent sessions (Claude Code, Codex, Cursor) and maintains sleep assertions only while tasks run; **modafinil** manages clamshell power assertions to keep MacBooks awake with the lid closed.
* **Strengths:** Solves specific unattended developer pain points (agent execution, headless clamshell).
* **Weaknesses:** Single-purpose utilities requiring separate status-bar slots; modafinil alters global power assertions.

---

## 3. Deep Architectural Comparisons

| Architectural Pillar | MenuBar Load Runner | runcat-dev / RunCatNeo | exelban / stats | kennss / SiliconScope | newmarcel / KeepingYouAwake |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Primary Goal** | Scriptable load runner & dashboard | Ecosystem cat runner & custom cards | Comprehensive system monitoring | Sudoless Apple Silicon telemetry & ANE | Dedicated sleep inhibitor |
| **Build Philosophy** | Single-file Swift script | Multi-file Xcode project (LUCA) | Multi-module Swift framework | Multi-file SwiftUI app | Multi-file Objective-C / Swift app |
| **Packaging & Execution** | Unbundled; launcher script | Compiled `.app` bundle | Compiled `.app` bundle | Compiled `.app` bundle | Compiled `.app` bundle |
| **Toolchain Requirement** | `swiftc` CLI only (no Xcode required) | Xcode 26.5+, Swift 6.2 | Xcode / CocoaPods | Xcode / Swift | Xcode |
| **Telemetry Inputs** | 8 unprivileged Mach/IOKit/SMC readers | CPU + Custom JSON metric files | Full hardware telemetry suite | CPU, GPU, ANE, Media Engine, RAM bandwidth | None (internal state only) |
| **Scaling Logic** | Adaptive hysteresis scaler (`btop`) | Fixed step thresholds | Numerical & graph plots | Numerical & progress gauges | Binary / timer countdown |
| **In-Menu Readout** | 60s sparkline + numeric + load avgs | Custom JSON metric cards | Full graphs/temps/per-process | Popover window with live charts | Countdown menu row |
| **Hardware Sensors / SMC** | SMCClient (Fan RPM, Max Die Temp) | External metric files / none | Comprehensive SMC & IOKit | IOReport & IOHIDEventSystem | None |
| **Power Throttling** | Occlusion pause (0% CPU) + thermal/LPM/RAM cap + Reduce Motion | None (standard app lifecycle) | None (continuous polling) | None (continuous polling) | None |
| **Sleep Supervision** | Built-in Keep Awake (`caffeinate`), process watch (`--keep-awake-pid`), surface countdown, auto-disengage, assertion inspection | None | None | None | Core function (`caffeinate -di`), URL scheme |
| **CLI & Automation** | Native launcher flags, env vars, LaunchAgents | JSON card format only | AppleScript / Defaults | None | URL Scheme (`keepingyouawake:///`) |
| **Update Mechanism** | Git tag check + `git pull --ff-only` (precompiles before restart) | App Store / GitHub release | Sparkle / GitHub release | GitHub release | Sparkle / GitHub release |

---

## 4. Key Differentiators vs. Established Peers

### 4.1 Toolchain & Dotfile Integration: Unbundled CLI vs. Heavy `.app` Bundles
Unlike GUI-first utilities (RunCat Neo, Stats, SiliconScope) that require full Xcode installations, project files, and application bundles, MenuBar Load Runner operates as an unbundled single file (`MenuBarLoadRunner.swift`) compiled on-demand via `swiftc -O -strict-concurrency=complete`. This enables frictionless dotfile tracking, version control via standard git workflows, and headless installation via user LaunchAgents (`scripts/install-login-item.sh`). Details: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) §1–§2.

### 4.2 Rate Normalization: Adaptive Stream Hysteresis vs. Discrete Thresholds
Monitors like RunCat and zoomies map metrics to animation speed using static percentage thresholds or discrete step functions. This approach fails on unbounded stream telemetry (network throughput, disk I/O, memory swap rate), which has no static ceiling. MenuBar Load Runner ports the adaptive `ThroughputScaler` from Linux `btop`, employing rolling ceiling discovery, asymmetric scaling headroom (1.3x up, 3.0x down), and debounce hysteresis counters to smoothly normalize unbounded rates to a 0…1 playback range without jitter. Details: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) §4.2.

### 4.3 Power Restraint: 0% CPU Occlusion Pausing & Self-Throttling
Continuous menu bar animation can consume significant resources under load. MenuBar Load Runner implements three-layer self-restraint:
1. **Occlusion pausing:** Completely suspends frame rasterization and timer loops when occluded by the MacBook notch, menu overflow, or display sleep, dropping CPU usage to **0%**.
2. **System pressure capping:** Automatically halves its speed ceiling during thermal pressure, Low Power Mode, or memory constraints.
3. **Accessibility compliance:** Honors the system **Reduce Motion** setting (freezing on the current frame) and supports manual freeze with automatic telemetry handoff to the label slot. Details: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) §3, §5.

### 4.4 Status-Bar Ergonomics: Dedicated Coordinated Slots vs. Baked Overlays
While peers bake text into rasterized icons or crowd single status items, MenuBar Load Runner utilizes a dual-slot model with coordinated `NSStatusItem` instances. Telemetry labels (`--label value`) and Keep Awake countdowns render in native system typography using reserved template widths and figure spaces (U+2007) to eliminate horizontal jitter. When animation is frozen, the live reading seamlessly transfers to the adjacent label slot. Details: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) §6.

### 4.5 Integrated Power Supervision: Process-Bound Watch & Assertion Visibility
Rather than requiring separate sleep inhibitor tools (KeepingYouAwake, adrafinil), sleep supervision is built directly into the telemetry runtime:
- **Process-bound exit watch:** `--keep-awake-pid <pid>` uses kernel-level `DispatchSourceProcess .exit` monitoring to hold the Mac awake until an unattended build or AI agent completes, then releases cleanly.
- **Surface countdown:** Active hold deadlines count down directly on the status bar (88:88:88 template, 1Hz ticker) without opening dropdown menus.
- **Machine assertion inspection:** Queries `IOPMCopyAssertionsByProcess` to surface foreign sleep locks, attribution, and scheduled deadlines. Details: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) §7.

---

## 5. Decision & Workflow Trade-offs

| If your primary need is... | Recommended Peer | Why |
|---|---|---|
| **Terminal workflows, dotfiles, unprivileged monitoring, animated load visualizer, integrated sleep watch** | **MenuBar Load Runner** | Single-file architecture, zero Xcode, adaptive `btop` scaling, process-bound keep-awake, 0% CPU occlusion pausing. |
| **Community keyframe gallery, pixel-art runner sharing, custom JSON dashboard cards** | **RunCat Neo** (`runcat-dev`) | Centralized Runner Gallery ecosystem, LUCA architecture, App Store convenience. |
| **Deep per-core graphs, per-PID process tables, battery cycle health, complex dashboards** | **Stats** (`exelban`) | Comprehensive multi-module framework designed for detailed system diagnostics. |
| **Sudoless Apple Silicon NPU (ANE) power & unified memory bandwidth tracking** | **SiliconScope** (`kennss`) | Native SwiftUI tool dedicated to unprivileged `IOReport` Apple Silicon rail profiling. |
| **Standalone sleep inhibitor with URL schemes, preferences window, and multi-language UI** | **KeepingYouAwake** (`newmarcel`) | Dedicated `caffeinate` wrapper with mature localization, Shortcuts/URL integration, and tabbed preferences. |

---

## 6. Deep Dive: Integrated Sleep Supervision vs. Dedicated Sleep Utilities (KeepingYouAwake)

Because sleep inhibition is a core subsystem of MenuBar Load Runner alongside telemetry, below is a grounded head-to-head comparison against **KeepingYouAwake** (`newmarcel/KeepingYouAwake`), the standard dedicated open-source macOS sleep inhibitor.

### 6.1 Comparative Grounding

Grounded in direct source-code review of `newmarcel/KeepingYouAwake` 1.6.8 against MenuBar Load Runner v1.23.0:

| Capability | MenuBar Load Runner | KeepingYouAwake | Comparison Note |
|---|---|---|---|
| **Inhibition Mechanism** | `caffeinate -di` | `caffeinate -di` (configurable to `-i`) | Both prevent display and idle sleep by default |
| **Process Binding** | PID-bound (`-w <pid>`) | PID-bound (`-w <pid>`) | Identical kernel cleanup safety model |
| **Timed Durations** | `-t <secs>`; presets (30m–8h) + custom | `-t <secs>`; preset list + user-editable list | Both leverage native `caffeinate` timer |
| **Live Countdown Surface** | Active countdown rendered in adjacent status item (`88:88:88` fixed template, 1Hz) | None on menu bar; menu row only | MLR displays time remaining at a glance without opening menu |
| **Process-Exit Watch** | `--keep-awake-pid <pid>` / menu `Until a process exits…` (`DispatchSourceProcess .exit` + `kill(pid, 0)`) | None | MLR natively watches external compilers, test runs, or AI agents to finish |
| **Low-Battery Disengage** | Configurable threshold (6–100% or Never) + 5% hard floor | Configurable slider (10–90%) | Both protect battery; MLR enforces hard 5% floor |
| **Thermal Disengage** | Automatic suspension on `.serious` / `.critical` thermal state | None | MLR releases sleep assertions under thermal load |
| **Low-Battery Arm Override** | Explicit menu arm honored above 5% floor | Not supported below cutoff | MLR supports intentional override with hard safety floor |
| **Machine Assertion Visibility** | Queries `IOPMCopyAssertionsByProcess` to show foreign holders and assertion list | Shows internal application state only | MLR surfaces system-wide sleep assertion state |
| **State vs Intent Preservation** | Decoupled (`isEnabled` vs `isRunning`); respawns with remaining time | Deactivation clears timer state | MLR resumes remaining duration after condition suspension |
| **Relaunch Persistence** | Persists target deadline timestamp to local state JSON | Persists activate-on-launch preference | MLR restores remaining bounded window without clock reset |
| **Launch-Time Scripting** | `--keep-awake <duration>`, `--keep-awake-pid` / env vars | URL Scheme (`keepingyouawake:///activate`) | Different automation models (CLI vs URL handler) |

### 6.2 Architectural Differences

1. **Automation Interfaces & Process Lifecycles:**
   - **KeepingYouAwake** registers a custom URL scheme (`keepingyouawake:///activate?seconds=...`, `/deactivate`, `/toggle`), enabling integration with `open`, Shortcuts, and web automation without special permissions.
   - **MenuBar Load Runner** provides launch-time flags (`--keep-awake`, `--keep-awake-pid`, `--battery-threshold`) and environment variables designed for shell scripts and LaunchAgent configurations. Crucially, it can bind its sleep hold to the lifetime of an arbitrary process (`--keep-awake-pid <pid>`), holding the machine awake until a long-running build or AI coding agent exits without needing artificial clock guesses.

2. **At-a-Glance Visibility vs. Menu Traversal:**
   - **KeepingYouAwake** relies on a binary status bar icon (coffee cup empty vs full) to denote state. Checking remaining duration requires clicking to open the dropdown menu.
   - **MenuBar Load Runner** projects the active countdown directly onto the menu bar in a dedicated adjacent status item slot (with an `88:88:88` fixed-width template and 1Hz ticker derived from the deadline), allowing engineers to verify hold status at a glance without breaking focus.

3. **Configuration Surface:**
   - **KeepingYouAwake** provides a dedicated multi-tab Preferences window (General, Battery, Durations, Advanced, Updates, About) and localization across 21 languages.
   - **MenuBar Load Runner** concentrates settings in the status dropdown (`Settings ▸` submenu) to maintain a compact, single-file codebase without bundle dependencies.

4. **Assertion Policy & System Visibility:**
   - **MenuBar Load Runner** queries machine-wide power management assertions via `IOPMCopyAssertionsByProcess`, presenting external sleep locks (such as terminal `caffeinate` sessions or media players) and their scheduled release times directly in the menu.

### 6.3 Practical Summary

For developers managing build tasks, downloads, and autonomous AI coding agent sessions within command-line environments, integrated sleep prevention in MenuBar Load Runner covers timed and process-bound assertion requirements while eliminating the need for a secondary menu bar utility. Users requiring custom URL schemes, multi-language localization, or standalone preferences windows continue to be well-served by dedicated tools like KeepingYouAwake.
