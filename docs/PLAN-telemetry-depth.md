# PLAN — Bandwidth, and the two cluster splits (R25)

**Status:** mechanisms grounded, one assumption unverified (§ 3). Nothing implemented.
Grounded 2026-09-20 against `MenuBarLoadRunner.swift` and a working reference implementation of the same
three readings in `~/workspace_fullstack/actop`.

**Lifecycle:** when R25 ships, the three readers join `docs/ARCHITECTURE.md` § 4 and their constants
§ 11, R25 leaves `docs/ROADMAP.md`, and this file is `git rm`'d in the same commit.

**Depends on R24** for the module boundary and for the snapshot that carries the new fields.

---

## 0. Where the three readings attach

R25 changes the shape of the core in exactly one place: the IOReport binding stops belonging to the ANE
reader and becomes something both readers hold. Everything else hangs off acquisition points that are
already there.

```
                      +--------------------------------------+
                      |         TelemetryCore (R24)          |
                      +--------------------------------------+
                                         |
                   +---------------------+---------------------+
                   v                     v                     v
        +---------------------+  +---------------+  +---------------------+
        | IOReportClient      |  | Mach ticks    |  | IOAccelerator       |
        | one dlopen, shared  |  | cluster slice |  | PerformanceStatis-  |
        | by both readers     |  |               |  | tics, already read  |
        +---------------------+  +---------------+  +---------------------+
             |           |               |                     |
             v           v               v                     v
           ANE W      BW GB/s         P and E          Renderer / Tiler
          (have)       (new)           (new)                 (new)
```

Three of the four arrows are new and only one box is: the GPU split costs no syscall, the cluster split
costs a `sysctl` at most, and bandwidth is the one reading that needs the histogram half of IOReport —
which is why the binding has to be shared rather than opened twice. The Mach column is provisional: § 3
has not decided whether the cluster split arrives from there or from a second IOReport group, and that
is the only structural question R25 still has open.

## 1. What is added, and what interface it costs

Three readings the hardware already publishes to unprivileged callers.

| Reading | Why it is not already covered | Public interface it adds |
|---|---|---|
| DRAM bus bandwidth | The memory reader measures *capacity and paging*. During local LLM inference the ceiling is the bus, and a machine can sit at 40% RAM while the bus is saturated — the two are unrelated numbers | One new `--load-source` enum value, `bandwidth`. No new flag |
| CPU P/E cluster split | `CPULoadMonitor` blends every core into one number, so "35%" cannot distinguish four saturated E-cores from four saturated P-cores — a different machine state with a different thermal future | None. A clause on the existing CPU row, a field in the snapshot |
| GPU Renderer/Tiler split | The same `PerformanceStatistics` dictionary already read for Device Utilization also carries the two pipeline halves. Not reading them is leaving a value in a dictionary already in hand | None. A clause on the existing GPU row, a field in the snapshot |

Total new public surface: **one enum value.** Everything else lands on rows and fields that exist.

## 2. Bandwidth — `BandwidthLoadMonitor`

**Mechanism.** IOReport group `("PMP", "DCS BW")`, channels whose name starts with `AMCC` and ends with
`RD+WR` — one per memory controller die. Each channel is a residency histogram whose state *names* are
bandwidth buckets (`"32GB/s"`, `"64GB/s"`, …) and whose values are time spent in that bucket.

Per channel: weighted mean of bucket **midpoints**, `Σ(midpoint · residency) / Σ(residency)`, where a
bucket's midpoint is the mean of its own edge and the previous one (first bucket's lower edge is 0).
Then sum the per-die means for a whole-chip figure.

> **The trap:** the bucket name is the bucket's *upper* edge. Weighting by that edge pins an idle Mac at
> a flat 32 GB/s, because nearly all of its residency sits in the bottom 0–32 GB/s bucket. Deriving
> midpoints from consecutive edges also survives a chip that spaces its buckets differently.

The result is already GB/s — a residency-weighted rate, not a counter — so it is **not** divided by the
sample interval.

**Shared IOReport binding.** `ANELoadMonitor` already `dlopen`s `/usr/lib/libIOReport.dylib`, but binds
only the simple-value path. Histograms need three more symbols — `IOReportStateGetCount`,
`IOReportStateGetNameForIndex`, `IOReportStateGetResidency` — plus `IOReportChannelGetGroup` and
`IOReportChannelGetSubGroup` to route channels. Extract the binding into one `IOReportClient` that both
readers hold, rather than a second `dlopen` beside the first. This is the one structural change in R25.

**Cost control.** The `DCS BW` group carries roughly ninety channels of thirty-two buckets each. Reading
per-state residency for all of them is thousands of round-trips per sample — well past what a 2 s tick
can spend. Residency is read **only** for channels whose name starts with `AMCC`; the rest are counted
and skipped.

**Normalization.** An unbounded rate, so it goes through `ThroughputScaler(floor:)` like network, disk
and swap — never a fixed ceiling, which would need a per-SoC table.

**Surfaces.** Label `185.2 GB/s` · menu row `Memory Bandwidth: 185.2 GB/s` · snapshot `bw_gbps`.

## 3. CPU P/E split — the one unverified assumption

Two mechanisms are available and they are not equally proven.

| Path | How | Standing |
|---|---|---|
| **A. Mach ticks, sliced** | `sysctlbyname("hw.perflevel0.logicalcpu")` / `perflevel1` for the cluster sizes, then slice the `PROCESSOR_CPU_LOAD_INFO` array `CPULoadMonitor` already reads | Zero new syscalls. **Assumes the logical-CPU index order groups by perflevel, and that perflevel0 is the P cluster.** Neither is documented, and the reference implementation does not use this path |
| **B. IOReport cluster residency** | Group `CPU Stats`, channels `E-Cluster_active` / `P-Cluster_active` | Proven in the reference implementation on this hardware. Nearly free once § 2 has extracted `IOReportClient` |

**Probe before choosing, not after.** Pin a known single-cluster load (`taskpolicy -b` for a background/E
job, a plain spin for P), read both paths in the same window, and compare. Take A only if it tracks B
within a few points under both loads; otherwise take B. Do not ship A on the strength of the index
ordering looking right on one machine — a wrong slice produces a plausible number, which is the failure
mode that survives review.

**Surfaces.** CPU menu row gains `· P 75% · E 12%` · snapshot `cpu_p_pct`, `cpu_e_pct`.

## 4. GPU Renderer/Tiler split

`GPULoadMonitor` already copies `PerformanceStatistics`; read two more keys from the dictionary in hand.
Zero additional syscalls.

Two rules carried over from the reference implementation: an entry with **no** `Device Utilization %`
key is not a usable accelerator and is skipped entirely (not treated as 0), and Renderer or Tiler
missing *individually* reads as 0. These are driver point reads, not interval integrations — the same
character as the Device figure beside them.

**Surfaces.** GPU menu row gains `· Renderer 46% · Tiler 19%` · snapshot `gpu_rend_pct`, `gpu_tiler_pct`.

## 5. Parameters

| Name | Default | Unit | Role | Lives in |
|---|---|---|---|---|
| `--load-source bandwidth` | — | enum value | Selects DRAM bandwidth as the speed driver | binary and launcher |
| `Tuning.bandwidthFloorGBps` | **to set** — see below | GB/s | `ThroughputScaler` floor: the scale's minimum ceiling, so an idle bus does not read as full speed | binary |
| `Tuning.labelBandwidthCeiling` | `999.9` | GB/s | Reserved label width for the `BW` readout (same role as `labelWattCeiling`) | binary |

**The floor is not guessable.** Measured on this machine 2026-09-20 with the reference implementation:
an idle bus reads **16.0 GB/s**. A floor at or below that makes an idle Mac drive the animation at full
speed — the exact failure `Tuning.aneFloorWatts` and `memoryIdleFloor` exist to prevent. Set it from a
measured idle/peak pair on this hardware before landing (a `mlx` or `llama.cpp` run gives the peak);
`50.0` is the working starting point, not a decision.

## 6. Verification (real binary)

| Command | Assertion |
|---|---|
| `$BIN --load-source bandwidth` | Exits 0, starts with the bandwidth reader engaged |
| `LOG_SLOTS=1 $BIN --load-source bandwidth --label value` | Label text matches `... GB/s` |
| `FORCE_UNAVAILABLE=bandwidth $BIN --load-source bandwidth` | Falls back like any other unavailable source, exit 0 |
| `$BIN --once` (R24) | `bw_gbps`, `cpu_p_pct`, `cpu_e_pct`, `gpu_rend_pct`, `gpu_tiler_pct` present on hardware that has them |
| Idle-bus sanity | `$BIN --once` on an idle Mac reports `bw_gbps` well below the busy figure taken during an inference run — the one assertion that catches the upper-edge weighting bug |
| CPU split probe (§ 3) | Under a pinned E-cluster load, `cpu_e_pct` is the higher of the two; under a spin, `cpu_p_pct` is |
| GPU split | With the menu open under GPU load, the row carries `Renderer` |
