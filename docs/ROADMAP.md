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

| Claim | State |
|---|---|
| The `ANE` energy channel's name and unit across chips | **One machine.** The reader's contract — an `"Energy Model"` group holding a channel literally named `ANE`, reporting accumulated energy in a labeled unit — is verified on an M4 Max / macOS 26.5.2 only. Units are read per row rather than assumed, and every absence degrades to `isAvailable == false` with a fallback to CPU, so a differently-named rail on another chip is a silent *missing source*, not a wrong reading. `qa.sh` cannot close this: it can only assert the shape the local machine reports. Needs a reading from an M1/M2/M3 and an Ultra to confirm the name generalizes. |
| Keep Awake resume after a condition-suspend | **No check.** When a battery/thermal suspend lifts, the respawn must pass the **remaining** window (`keepAwakeRemainingSeconds`), not the original length, and the tint mark must not move. Unreachable from `qa.sh`: `FORCE_BATTERY` pins one static value per process, so no run can cross the threshold mid-flight — forcing it needs a real draining battery or a hook that changes a decision, which is barred. |
| Battery Threshold (R5) click-path residue | Code complete and mechanically verified for the flag/env/persistence halves (`qa.sh` §3a/§3b, 18 cases each). **Click-only and unverified:** apply-immediately, override-retirement, the red band's *rendering*, the mark's position (AX can't read a custom `onStateImage`), and the threshold submenu's own row list (`menu-dump` descends one level; that submenu is two deep). |
| Per-user single-instance guard (`pgrep -U`) | Flag semantics proven in isolation; same-user rejection tested live. **Cross-user unverified** — needs a second account + fast user switching. |
| Keep Awake radio-mark glyphs, and the `This App` header's styling | Eyes-only. `AXMenuItemMarkChar` is empty (custom `onStateImage`), so AX can't read the mark; a synthesized `CGEvent` click renders the menu for a screenshot but needs screen coordinates AX reports unreliably across displays — deliberately checked ad hoc by looking, not by a script. Menu *structure* is mechanical (`tests/menu-dump.applescript`) and the v1.19.1 two-section order verified 2026-07-31. |
| R13 is category-first ("no other menu-bar util tells you") | **KeepingYouAwake 1.6.8**, **RunCat Neo 1.0.3**, **Stats v3.0.15**, **SiliconScope**, and **adrafinil** were inspected (September 2026); none inspects `IOPMCopyAssertionsByProcess` for foreign holders. Amphetamine remains unchecked. The claim stays out of `README.md` and `docs/cover.html` until settled — `pmset -g assertions` is one command, so a wrong category claim is worse than none. |
| `ThroughputScaler` hysteresis, `SemVer` / `highestTag` parsing, `Restarter`'s argv + mode mapping | **No check at all.** These have no reachable functional path yet: the scaler needs sustained synthetic net/disk load to move its ceiling; the tag parse needs a checkout whose `origin` has canned `v*` tags; `Restarter`'s launchd branch needs a real LaunchAgent job. Don't "restore coverage" by re-adding a re-ported copy of the logic — see `CLAUDE.md` § 测试与回归约定. |
| The in-app update sequence: pull → `Builder.precompile` → Restart | **Click-only, unverified.** The pieces the launcher owns *are* covered — `qa.sh` §6 asserts `--precompile` builds and launches nothing, that a live instance survives the rebuild (the temp+rename), and that a rejected launch doesn't compile. What no run reaches is the modal path that drives them: the `Building vX.Y.Z…` row, the build-failed wording, and the restart actually being fast afterwards. Reaching it needs a checkout whose `origin` carries a newer canned tag — the same harness the row above wants — so it is walked by hand on the *previous* version's binary: `Check for Updates…` must become `Update available: vX.Y.Z`, the app must keep running while it pulls and compiles (reopen the menu: the row reads `Building vX.Y.Z…`), and **Restart** must come back within a second or two leaving exactly one instance. `rm -rf "$(getconf DARWIN_USER_CACHE_DIR)/clang/ModuleCache"` first to exercise the cold path — the *build* should then take ~30s and the restart still shouldn't. The failure being guarded against is silent and bimodal: with a warm module cache a deferred build costs ~7s and looks fine, so a regression only shows on the cold path. |
| Four §3e cases gated on a **quiet machine** (no display holder / no assertions at all) | §3e NOTEs each rather than faking a quiet system: R7's `tint=paused`, R16's `Nothing holding sleep`, attribution when another display holder sorts first, and the idle-only reading. Their states differ — R7's tone **is** confirmed by hand (2026-08-01: three consecutive `tint=paused` ticks, no `-w` child), so don't read its NOTE as the tone being unverified; the other three are unverified locally. The method for R7's tone, since anything holding the display masks a paused line entirely: `screencapture -x -R<x>,<y>,<w>,<h>` a strip of the bar (screen coords from `LOG_SLOTS`) and composite `bg×(1−α) + tint×α` to rank candidate alphas — a **proxy**, and where it and a direct look disagree the look decides (it did: the shipped `0.22` over the ratio's `0.30`). Don't soften any of them to make a run pass. |
| `SMCClient` holds exactly one `io_connect_t` | **Structural only** — IOKit connections aren't visible to `lsof`/`ps`, and counting `IOServiceOpen` calls needs dtrace (root + SIP off), so *exactly one* rests on the code's shape: one `static let shared`, one `openSMC()` call site, one write to `connection`. The weaker sharing-works claim **is** covered: fan and temperature driven together in one process, both correct (2026-08-01). Don't upgrade that into connection-counting, and don't "cover" the remainder with a re-ported copy of the client. |
| Temperature reader — three dark spots | (1) `Tpx*`-are-cluster-maxima verified on **one chip only** — re-run the ramp comparison on any new hardware; a wrong chip silently under-reads and no in-process sample can tell. (2) The all-clusters-parked branch (v1.20.1) has **no functional check and none is available** — hardware state, and forcing it needs a decision-changing hook, which is barred; reachable, not theoretical. (3) The map above ~78 °C is exercised by arithmetic only — the test machine peaks there. Don't cover any of these with a re-ported copy, and don't make the map adaptive. |
| Status-item occlusion pauses (frame driver, and the countdown ticker) | **No functional check, and none is available.** Occlusion is the window server's verdict, not the test's — forcing it needs a decision-changing hook, which is barred. Both consumers read one `statusItemOccluded` property, so what is untested is that single reading, not each consumer: everything downstream (the countdown's content, slot width, expiry collapse and resume redraw) is covered by `qa.sh` §3c on a visible bar. Checked by eye instead: arm a window, drag the item behind the notch or sleep the display, and `sample` the process — the 1Hz relayout should disappear and the countdown should be correct, not stale, the moment it reappears. |
| Thermal pause rendering | No way to force a thermal state. The battery reasons share the code path and are covered by `tests/qa.sh` §3a; only the thermal *trigger* is untested. |
| Reduce Motion trigger (R17) | **No functional check.** The pref is the machine's, not the test's; forcing it needs a decision-changing hook, which is barred (`FORCE_BATTERY` got in only because a desktop has *no* battery to read — every Mac has a real, readable Reduce Motion setting). Everything downstream of the observer (freeze rendering, label handoff, persistence) is covered by `qa.sh` §3g via the manual toggle; only the notification→reading wiring is eyes-only: toggle System Settings → Accessibility → Display → Reduce Motion against a running instance — the icon freezes/resumes with no relaunch and the row reads `Freeze Animation — on via Reduce Motion` with the checkmark still tracking your own toggle. |
| Label slot order is "creation order", assumed not guaranteed | Confirmed on a roomy bar 2026-08-01 (§3c passed all six cases). The notched built-in display inverts the order **deterministically** (3/3, modified and unmodified binary) — environmental, not a code change, and contiguous-but-wrong, so §3c **FAILs** there: read an adjacency FAIL against which screen the bar was on before believing it. Deliberately not softened into a NOTE, or a real ordering regression hides behind it. Lead if it recurs: `NSStatusItem.autosaveName` is never set, and ⌘-drag positions persist against it — probe that first. |
| Everything behind a menu **click** | **Unscriptable, and checked by hand or nowhere.** No shell can click an `NSMenu`: preset switching via `Presets ▸`, the Other Sources ▸/▾ expand and its row-click source switch, `Settings ▸ Menu Bar Label` mode + `Position`, `Freeze Animation`, `Start at Login` (writes the real LaunchAgent with the *current* args), the Keep Awake tint pick / `Duration` / `Custom…` (`0 hr 0 min` cancels), `Until a process exits…` (its prompt, and a *name* resolving to the newest match — the pid form and everything downstream of the binding is covered by `qa.sh` §3f), and the low-battery arm **override**, which by design only a live click can set. Menu *structure* is a diff, not a squint — `osascript tests/menu-dump.applescript "$(pgrep -U "$(id -u)" -f '/MenuBarLoadRunner( |$)')"` (needs Accessibility for your terminal, resolve by **pid**, never by name); it cannot see selection marks, the menu bar itself, or what a click does. |
