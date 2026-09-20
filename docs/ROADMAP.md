# ROADMAP

The product tracker: what is open and what was declined. Shipped work and system
boundaries are not tracked here — they are described as-built in `docs/ARCHITECTURE.md`. Created 2026-07-26;
current as of v1.25.0.

Items are `R<n>`, assigned once, never reused. **P1** user-visible defect or silent failure · **P2**
real capability gap · **P3** nice to have · **P4** parity for its own sake. Nothing here is a
commitment or a date.

Keep this at tracking altitude — item, priority, blocker, and any catch an implementer would trip
over. Design and technical rationale live in `docs/ARCHITECTURE.md`; release history lives in `CHANGELOG.md`
and git. Refer to code by **symbol, never by line number** — anchors rot every release.

---

## Where this sits

One line: a load *visualizer* that earns each new capability through unprivileged reads and
self-restraint — it only ever reads the system, and the only thing it throttles is itself.

**Shipped capability is not tracked here.** How the app got from v1.0 to today — which stage
established which invariant — is as-built and lives in `docs/ARCHITECTURE.md` § 12; a completed item's
durable outcome moves there and its row leaves this file.

The Open items sit on the same arc: R24/R25 let something other than a pair of eyes read what the
readers already know; R9 extends preset identity beyond the repo.

## Open

Ordered by ROI, highest first — value against cost, not just the priority band. Each row is one line of
tracking: what is wrong, and where the design lives. Mechanism, schema, parameters and verification are
in the linked plan — never restated here.

| ID | Item | Pri | Blocked by |
|---|---|---|---|
| R24 | **The readings exist but only a pair of eyes can get at them.** Nine unprivileged readers run behind a GUI that has to be on screen; nothing else on the machine — a script, `co-cli`, an agent about to start a long build — can ask what the hardware is doing. Extract the readers into a telemetry core the GUI owns but does not define, and add exactly one flag: `--once`, a single-line JSON snapshot of every available source, then exit. One flag, one schema, one sampling window, no `NSApplication`, no `state.json`, no launcher singleton or compile. Module boundary, snapshot contract and parameter table: [`PLAN-telemetry-core-and-snapshot.md`](PLAN-telemetry-core-and-snapshot.md). | P2 | — |
| R25 | **Three readings the hardware publishes and the app does not take.** DRAM bus bandwidth — the actual ceiling during local LLM inference, invisible to a RAM-capacity reader; the CPU P/E cluster split, which says *which half* of the chip is busy; the GPU Renderer/Tiler split. Bandwidth is a new `--load-source` enum value, the other two are fields on rows that already exist — no new flags. Mechanisms, the one unverified assumption and its probe: [`PLAN-telemetry-depth.md`](PLAN-telemetry-depth.md). | P3 | R24 |
| R26 | **Rename to `co-load-runner`, retire `actop`.** A name and a public deprecation, not a capability. Held below (§ R26) with its cost, until the `--once` contract has a caller that actually exercises it. | P4 | R24, R25 |
| R9 | **Custom GIFs are launch-only, not reusable presets.** Add first-class local animated-GIF presets under `~/.config/menubar-load-runner/presets/`, with drop-in discovery plus native menu/CLI import that validates, alpha-union crops, bounds, normalizes, atomically installs, and immediately selects the result. Static images and non-GIF animation formats stay out of the first cut; a raw positional GIF path remains the no-install escape hatch. Full accepted-input contract, resource budgets, identity/merge rules, failure semantics, symbol-level implementation order, and real-binary QA matrix: [`PLAN-custom-gifs.md`](PLAN-custom-gifs.md). | P4 | — |

### R26 — the rename, and what it costs

No plan file: nothing here is a design, it is one decision held open.

**The case for it.** `co-cli`, `co-s2s` and `co-asciiball` are one family in `~/workspace_genai/repos.yaml`;
this binary would be its hardware sense and its lifecycle guard. `actop` (Python, PyPI, CI, `actop.pages.dev`)
reads the same chip through the same unprivileged interfaces, and once R25 lands, the overlap is most of it.

**Why it is not bundled into R24/R25.** Integration is a *contract*, not a name — `co-cli` can call any
binary. The rename buys nothing the `--once` schema does not already buy, and it is charged separately:
the `MENUBAR_LOAD_RUNNER_*` hook names that `AGENTS.md` and `tests/qa.sh` hold as canonical, the
`state.json` directory, the launcher filename, the LaunchAgent label in `scripts/`, the self-update
remote, README and the cover page. Retiring `actop` is a public act on a published package with its own
users, decided in that repo, not recorded here.

**If it is taken:** one name, one env prefix, one state path with a single silent migration. No permanent
alias and no dual-prefix fallback — the compatibility layer is the expensive half and it never gets
deleted. **Decide only when** an outside caller has been running against `--once` long enough for its
schema to have stopped moving.

## Declined

A row earns its place here only by guarding a boundary the **current** design rests on, and only if the
reason lives nowhere else — a decline whose argument is already in `AGENTS.md`, or that merely records
something once cut for scope, is history and belongs in git. Re-propose only with a concrete report of
the behavior being missed.

| Item | Why not |
|---|---|
| GPU power and SoC package power readers (the rest of R10) | **Decided 2026-09-12:** Energy Model channels nest arbitrarily with cross-chip schema debt while closely correlating with CPU/GPU utilization and temperature readers, offering negligible ROI compared to the isolated ANE leaf rail. |
| A numeric CPU speed-limit percentage on the temperature row (the other half of R23) | **Decided 2026-09-12:** Apple Silicon publishes a discrete pressure level, not a frequency cap — `IOPMCopyCPUPowerStatus` answers `kIOReturnNotFound` (probed) — so any percentage would be derived from a level rather than measured. Intel's `CPU_Speed_Limit` is real, but no Intel hardware is available to this project — both the read and the temperature row it would annotate would ship unverified. Re-propose with a reading from a real Intel Mac. |
| Periodic update polling (R18) | **Decided 2026-09-10:** Launch-time discovery plus on-demand menu checks are sufficient for non-critical releases without adding background network timers or lifecycle complexity. |
| An `.app` bundle · notarization · Homebrew cask · Sparkle · a URL scheme / automation interface | **Decided 2026-07-26:** Stays an unbundled source-built binary to preserve direct CLI argv execution, zero-cost ad-hoc signing, and git-native in-place updates without $99/yr notarization overhead. |
| Closed-lid (clamshell) sleep prevention via `pmset disablesleep` (like modafinil) | **Violates unprivileged execution and clean-teardown tenets:** Mutating system-wide sleep policy via `pmset` requires root and risks leaving sleep permanently disabled on crash, unlike PID-bound `caffeinate`. |
| Per-process CPU/RAM breakdown table (like Stats v3 or Activity Monitor) — including an on-demand `--proc-list` that only walks the task list when asked | **Violates self-throttling and minimal-footprint tenets:** Continuously walking the Mach task list for per-PID statistics consumes 1–3% CPU, turning the observer into the load it measures. Making it on-demand answers the cost and not the need: nothing yet names a decision that a whole-machine reading leaves unanswerable, and `top` already answers per-process. Re-propose with that decision, not with a cheaper implementation. |
| Online community asset store / in-app GIF downloader (like RunCat Runner Gallery) | **Violates the self-contained dotfile philosophy:** Remote asset downloads introduce network attack surfaces and untrusted runtime ingestion, whereas local Git/directory presets remain transparent and auditable. |
| Arbitrary external metric polling / custom JSON telemetry (like RunCat Neo Custom Metrics) | **Out of scope:** Ingesting arbitrary external files introduces schema maintenance, file watching, and unbounded failure modes that dilute the direct unprivileged kernel/Mach/SMC telemetry core. |
| Embedded Model Context Protocol (MCP) server or background daemon | **Violates the single-process, CLI-first architecture:** Background daemon sockets and protocol parsers add runtime complexity when agents already integrate natively via standard CLI flags, `state.json`, and `--keep-awake-pid`. |
| Release Keep Awake on fast user switching | **Contradicts chosen semantics:** A Keep Awake hold is a time or process promise, so unattended background tasks must continue running when another user switches in. |
| Any transient announcement of a Keep Awake event (HUD panel, notification) | **Ineffective and technically barred:** Transient panels are missed during unattended battery events (which already wear a persistent paused tone), while native notifications require an application bundle. |

