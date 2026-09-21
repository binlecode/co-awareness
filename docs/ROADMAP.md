# ROADMAP

The product tracker: what is open and what was declined. Shipped work and system
boundaries are not tracked here — they are described as-built in `docs/ARCHITECTURE.md`.

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

**Shipped capability is not tracked here.** Which stage established which invariant is as-built and
lives in `docs/ARCHITECTURE.md` § 12; a completed item's durable outcome moves there and its row
leaves this file. Nothing in this file records what changed, or when.

One item is open: R26, the name, held until an outside caller has exercised the CLI surface that `--once` and `--status` now make up.

## Open

Ordered by ROI, highest first — value against cost, not just the priority band. Each row is one line of
tracking: what is wrong, and where the design lives. Mechanism, schema, parameters and verification live
in the design the row points to — never restated here.

| ID | Item | Pri | Blocked by |
|---|---|---|---|
| R26 | **Rename to `co-load-runner`.** A name, not a capability, and confined to this repo — nothing in `actop` moves with it. Held below (§ R26) with its cost, until an outside caller has exercised the CLI surface. | P4 | — |

### R26 — the rename, and what it costs

No plan file: nothing here is a design, it is one decision held open.

**The case for it.** `co-cli`, `co-s2s` and `co-asciiball` are one family in `~/workspace_genai/repos.yaml`;
this binary would be its hardware sense and its lifecycle guard. `actop` (Python, PyPI, CI, `actop.pages.dev`)
reads the same chip through the same unprivileged interfaces; that overlap says where this binary sits
in the family, and is not an argument that either of them has to go.

**Why it is not bundled with a capability.** Integration is a *contract*, not a name — `co-cli` can call any
binary. The rename buys nothing the `--once` schema does not already buy, and it is charged separately:
the `MENUBAR_LOAD_RUNNER_*` hook names that `AGENTS.md` and `tests/qa.sh` hold as canonical, the
`state.json` directory, the launcher filename, the LaunchAgent label in `scripts/`, the self-update
remote, README and the cover page.

**Scope: this repo only.** R26 is the surfaces above and nothing else. `actop` is not touched, not
deprecated and not a dependency of this item — it is a published package with its own users, and
whatever becomes of it is decided in that repo. A rename here neither needs nor implies a move there,
so the two never have to be sequenced.

**If it is taken:** one name, one env prefix, one state path with a single silent migration. No permanent
alias and no dual-prefix fallback — the compatibility layer is the expensive half and it never gets
deleted. **Decide only when** an outside caller (such as `co-cli`) has been running against the
`--once` + `--status` contract long enough for it to have stopped moving.

## Declined

A row earns its place here only by guarding a boundary the **current** design rests on, and only if the
reason lives nowhere else — a decline whose argument is already in `AGENTS.md`, or that merely records
something once cut for scope, is history and belongs in git. Re-propose only with a concrete report of
the behavior being missed.

| Item | Why not |
|---|---|
| Single-field extraction — `--query <key>` (proposed with R27) | **Slower than the pipe it would replace:** each call is a fresh process that must re-open the same `Tuning.snapshotWindow` delta window, so three fields cost three spawns and three windows where `--once \| jq` costs one, and the tokens saved off a ~300-byte line are noise against that. It is also a second entrance to a fact `--once` already carries, with a `--json` variant of its own output — the one-thing-two-formats surface `AGENTS.md` § 红线 9 forbids. Re-propose with a caller that cannot pipe. |
| Pre-flight admission gate — `--check` (proposed with R27) | **The threshold is the caller's, and the verdict would corrupt a test hook.** What counts as "too hot to start" is a property of the job about to run, not of the observed machine; the gate would also have reused the keep-awake battery threshold — a sleep-prevention intent — as an admission threshold, conflating two numbers. Worse, its verdict reads `KernelThermalPressure`, which `docs/ARCHITECTURE.md` §10 holds display-only precisely so `MENUBAR_LOAD_RUNNER_FORCE_THERMAL` stays an input simulator rather than a hook that moves a business decision. `--once` already publishes `thermal` and `battery_pct`. Re-propose as the caller's own threshold applied by the caller. |
| Agent tool-schema export — `--tool-spec` (proposed with R27) | **Nothing consumes it:** `co-cli` registers native tools from decorated Python in its own registry and reaches an external binary through `shell_exec`, which reads `--help`; the only other schema path is an MCP server, declined below. The export would also be a second hardcoded copy of the `jsonLine` field set, drifting from it at the first field change. Re-propose with a caller that ingests an external binary's schema. |
| First-class local custom-GIF presets / runtime import (R9) | Over-design: custom GIF import is not a core feature or user requirement. The system only requires a reproducible SOP to add whatever art is needed (add optimized GIF to `gifs/` and register in `gifs/presets.json`, per `AGENTS.md`); a raw positional path (`./menubar-load-runner /path/to.gif`) remains the unbundled escape hatch. |
| GPU power and SoC package power readers (the rest of R10) | Energy Model channels nest arbitrarily with cross-chip schema debt while closely correlating with CPU/GPU utilization and temperature readers, offering negligible ROI compared to the isolated ANE leaf rail. |
| A numeric CPU speed-limit percentage on the temperature row (the other half of R23) | Apple Silicon publishes a discrete pressure level, not a frequency cap — `IOPMCopyCPUPowerStatus` answers `kIOReturnNotFound` (probed) — so any percentage would be derived from a level rather than measured. Intel's `CPU_Speed_Limit` is real, but no Intel hardware is available to this project — both the read and the temperature row it would annotate would ship unverified. Re-propose with a reading from a real Intel Mac. |
| Periodic update polling (R18) | Launch-time discovery plus on-demand menu checks are sufficient for non-critical releases without adding background network timers or lifecycle complexity. |
| An `.app` bundle · notarization · Homebrew cask · Sparkle · a URL scheme / automation interface | Stays an unbundled source-built binary to preserve direct CLI argv execution, zero-cost ad-hoc signing, and git-native in-place updates without $99/yr notarization overhead. |
| Closed-lid (clamshell) sleep prevention via `pmset disablesleep` (like modafinil) | **Violates unprivileged execution and clean-teardown tenets:** Mutating system-wide sleep policy via `pmset` requires root and risks leaving sleep permanently disabled on crash, unlike PID-bound `caffeinate`. |
| Per-process CPU/RAM breakdown table (like Stats v3 or Activity Monitor) — including an on-demand `--proc-list` that only walks the task list when asked | **Violates self-throttling and minimal-footprint tenets:** Continuously walking the Mach task list for per-PID statistics consumes 1–3% CPU, turning the observer into the load it measures. Making it on-demand answers the cost and not the need: nothing yet names a decision that a whole-machine reading leaves unanswerable, and `top` already answers per-process. Re-propose with that decision, not with a cheaper implementation. |
| Online community asset store / in-app GIF downloader (like RunCat Runner Gallery) | **Violates the self-contained dotfile philosophy:** Remote asset downloads introduce network attack surfaces and untrusted runtime ingestion, whereas local Git/directory presets remain transparent and auditable. |
| Arbitrary external metric polling / custom JSON telemetry (like RunCat Neo Custom Metrics) | **Out of scope:** Ingesting arbitrary external files introduces schema maintenance, file watching, and unbounded failure modes that dilute the direct unprivileged kernel/Mach/SMC telemetry core. |
| Embedded Model Context Protocol (MCP) server or background daemon | **Violates the single-process, CLI-first architecture:** Background daemon sockets and protocol parsers add runtime complexity when agents already integrate natively via standard CLI flags, `state.json`, and `--keep-awake-pid`. |
| Release Keep Awake on fast user switching | **Contradicts chosen semantics:** A Keep Awake hold is a time or process promise, so unattended background tasks must continue running when another user switches in. |
| Any transient announcement of a Keep Awake event (HUD panel, notification) | **Ineffective and technically barred:** Transient panels are missed during unattended battery events (which already wear a persistent paused tone), while native notifications require an application bundle. |

