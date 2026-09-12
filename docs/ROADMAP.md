# ROADMAP

The product tracker: what is open, what was declined, known limits, and verification debt. Shipped
work is not tracked here — it is described as-built in `docs/ARCHITECTURE.md`. Created 2026-07-26;
current as of v1.23.0 (updated September 2026).

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

The Open items sit on the same arc:
- R9 extends preset identity beyond the repo;
- R8 is parity only;
- R20 separates menu-open highlight from custom presets.

## Open

Ordered by ROI, highest first — value against cost, not just the priority band.

| ID | Item | Pri | Blocked by |
|---|---|---|---|
| R9 | **Custom GIFs are launch-only, not reusable presets.** Add first-class local animated-GIF presets under `~/.config/menubar-load-runner/presets/`, with drop-in discovery plus native menu/CLI import that validates, alpha-union crops, bounds, normalizes, atomically installs, and immediately selects the result. Static images and non-GIF animation formats stay out of the first cut; a raw positional GIF path remains the no-install escape hatch. Full accepted-input contract, resource budgets, identity/merge rules, failure semantics, symbol-level implementation order, and real-binary QA matrix: [`PLAN-custom-gifs.md`](PLAN-custom-gifs.md). | P4 | — |
| R8 | **English only** — zero `NSLocalizedString`. | P4 | — |
| R20 | **The menu-open highlight is not configurable.** This is an appearance preference, not part of custom-preset ingestion, and no longer shares R9's scope. | P4 | Define the desired highlighted and unhighlighted behavior against the layer-backed animation view before implementation. |

## Declined

A row earns its place here only by guarding a boundary the **current** design rests on, and only if the
reason lives nowhere else — a decline whose argument is already in `AGENTS.md`, or that merely records
something once cut for scope, is history and belongs in git. Re-propose only with a concrete report of
the behavior being missed.

| Item | Why not |
|---|---|
| GPU power and SoC package power readers (the rest of R10) | **Decided 2026-09-12:** Energy Model channels nest arbitrarily with cross-chip schema debt while closely correlating with CPU/GPU utilization and temperature readers, offering negligible ROI compared to the isolated ANE leaf rail. |
| Periodic update polling (R18) | **Decided 2026-09-10:** Launch-time discovery plus on-demand menu checks are sufficient for non-critical releases without adding background network timers or lifecycle complexity. |
| An `.app` bundle · notarization · Homebrew cask · Sparkle · a URL scheme / automation interface | **Decided 2026-07-26:** Stays an unbundled source-built binary to preserve direct CLI argv execution, zero-cost ad-hoc signing, and git-native in-place updates without $99/yr notarization overhead. |
| Closed-lid (clamshell) sleep prevention via `pmset disablesleep` (like modafinil) | **Violates unprivileged execution and clean-teardown tenets:** Mutating system-wide sleep policy via `pmset` requires root and risks leaving sleep permanently disabled on crash, unlike PID-bound `caffeinate`. |
| Per-process CPU/RAM breakdown table (like Stats v3 or Activity Monitor) | **Violates self-throttling and minimal-footprint tenets:** Continuously walking the Mach task list for per-PID statistics consumes 1–3% CPU, turning the observer into the load it measures. |
| Online community asset store / in-app GIF downloader (like RunCat Runner Gallery) | **Violates the self-contained dotfile philosophy:** Remote asset downloads introduce network attack surfaces and untrusted runtime ingestion, whereas local Git/directory presets remain transparent and auditable. |
| Arbitrary external metric polling / custom JSON telemetry (like RunCat Neo Custom Metrics) | **Out of scope:** Ingesting arbitrary external files introduces schema maintenance, file watching, and unbounded failure modes that dilute the direct unprivileged kernel/Mach/SMC telemetry core. |
| Embedded Model Context Protocol (MCP) server or background daemon | **Violates the single-process, CLI-first architecture:** Background daemon sockets and protocol parsers add runtime complexity when agents already integrate natively via standard CLI flags, `state.json`, and `--keep-awake-pid`. |
| Release Keep Awake on fast user switching | **Contradicts chosen semantics:** A Keep Awake hold is a time or process promise, so unattended background tasks must continue running when another user switches in. |
| Any transient announcement of a Keep Awake event (HUD panel, notification) | **Ineffective and technically barred:** Transient panels are missed during unattended battery events (which already wear a persistent paused tone), while native notifications require an application bundle. |

## Known limits — not gaps

| Limit | Why |
|---|---|
| Keep Awake's *effect* is machine-wide | `caffeinate` holds the whole Mac awake; sleep is machine-level, so two users' windows don't compose. Per-user *state* is already correct (per-account Application Support). |
| A window held by a background login session is invisible from the foreground one | Consequence of the above. Nothing to store differently. |
| Clamshell sleep can't be prevented | `caffeinate` cannot inhibit it. |
| Below 5% on battery the Mac sleeps regardless | Deliberate floor under the arm-anyway override: an explicit "anyway" is honored from 20% to 5%, not into a hard power-off. |
| The interpreted-`swift` fallback isn't singleton-guarded | Runs only when `swiftc` fails; the guard matches the compiled binary's path. |
| The menu-bar label may not sit adjacent to the icon on a **full** bar | macOS owns status-item placement and offers no reorder API — verified 2026-07-29 (6/6 scattered on a notched built-in display, 100% correct on a roomy external). Creation order decides *intent*; the bar decides the outcome. The v1.16.0 no-jitter guarantee is unaffected; `tests/qa.sh` §3c reports NOTE on scatter, so adjacency goes **unverified** on such a machine. Full account: `docs/ARCHITECTURE.md` § 6. |

## Verification debt

All automatable functional behaviors, state transitions, and condition recoveries are covered by `tests/qa.sh` (core + gui). No open verification debt. Manual eyes-only release checks are maintained in [`docs/ARCHITECTURE.md`](ARCHITECTURE.md) § 13.
