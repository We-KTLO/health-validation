# Draft upstream issue and PR: B300 architecture (not filed)

> This file is a staging note for the fork and is **not part of the proposed change**. Drop this commit before
> opening the PR. Nothing here has been filed upstream.
>
> **Base.** Branch `feat/b300-architecture`, rebased onto `main@0ed4778` (2026-11-05). The original patches were
> written against `catalog/v0.3.1`; see "Reconciling with main" for what changed.
>
> **Signing.** CONTRIBUTING requires a DCO sign-off (present) **and** a cryptographic signature (`git commit -S`).
> The commits on this branch are not signed with a key; re-sign them (`git rebase --exec 'git commit --amend
> --no-edit -S -s' upstream/main`) before opening the PR.

---

## Draft issue (feature request form)

**Title:** `feat(catalog): support HGX B300 (B300 architecture)`

**Problem / motivation**

The catalog has no `B300` architecture, and the toolkit maps the B300 product name (`NVIDIA B300 SXM6 AC`) to
`unknown`. Every architecture-scoped check therefore skips on HGX B300 nodes, so the catalog gives no verdict there.

**Proposed solution**

Recognize `B300` in the toolkit, and list it on the checks that apply to one HGX B300 node (8 × B300 SXM per x86_64
host, NVLink 5 inside the node through the baseboard NVSwitches, node-to-node traffic over RDMA, no multi-node
NVLink domain, IMEX or Grace C2C), the same way B200 was added in #225. Per-architecture defaults come from
measurements on B300 nodes (see the PR).

**Alternatives considered**

Fleets can override thresholds today, but they cannot make an architecture-scoped check run on an architecture
it does not list, so an override alone does not help.

**Scope**

Catalog check definitions and the toolkit's architecture mapping. No new signals and no new checks. The
domain-scoped NVLink fabric checks stay off B300 (follow-up below).

---

## Draft PR

**Title:** `feat(catalog): add the B300 architecture to the checks that apply to HGX B300`

### Summary

The catalog has no B300 architecture, so every architecture-scoped check skips on HGX B300 nodes.

This PR lists `B300` on the checks that apply to one HGX B300 node, next to `B200` where B200 already runs. Where a
check needs per-architecture defaults, it adds B300 values measured on B300 nodes. The toolkit now maps B300
product names to `B300`.

**Why.** KTLO, a Kubernetes GPU-health platform, evaluates this catalog as one of its passive checks. On B300 today
the catalog skips. KTLO grades B300 nodes with its own calibrated checks and wants NVIDIA's verdict on the same
nodes too, under NVIDIA's own rules and recommended actions. Each threshold below is a `DefaultThreshold`, so
fleets can still override it.

### Changes

Three commits:

1. `feat(catalog): add B300 to the checks that apply to HGX B300`
   - `gpu_count`: B300 added, `PerArchitecture` 8 (as B200).
   - `gpu_power_limit`: B300 added, `expected_power_limit_w` 1100 W. (B200 is not on this check; see the SKU
     caveat.)
   - `nvlink_bandwidth_count_speed`: B300 added, `expected_link_count` 18 and `expected_speed_mbytes_per_sec`
     53125.
   - Added with their shared defaults: `gpu_average_power_draw`, `gpu_power_state`, `gpu_sensor_throttle`,
     `gpu_compute_apps`, `nvlink_active`, `nvlink_p2p`, `rdma_linkflap_fatal`.
   - Xid checks: B300 joins `allArchitectures`, `blackwellArchitectures` (including the NVLink5 144-150 family),
     `hopperBlackwellArchitectures` and `hgxArchitectures`; it does not join `graceBlackwellArchitectures`
     (Xid 142, NVENC3).
   - `toolkit/app`: `averagedPowerDrawArchitectures` gains B300, so `gpu_average_power_draw` may read the power
     draw there (NVML documents power usage as a one-second average on Ampere except GA100, and newer).
   - `nvlink_fabric` and `nvlink_fabric_health` stay off B300, with the reason in a comment (domain-scoped).
   - `TestCatalog_B300Coverage` pins the included and excluded lists; B300 cases added to the `gpu_count`,
     `gpu_power_limit`, `nvlink_bandwidth_count_speed`, `gpu_average_power_draw`, `rdma_linkflap_fatal` and Xid
     coverage tests; `TestXidErrors_B300` (Xid 79 and fatal NVLink5 Xid 146 are critical on B300).
   - README: a "B300 support" note next to the GB300 one. `docs/reference` regenerated.
2. `feat(gpu): recognize B300 GPUs by product name` — `gpuinventory.ArchitectureFromName` maps `B300` (after
   `GB300`, because `"NVIDIA GB300"` contains `"B300"`), with tests in `gpuinventory` and `toolkit/nvml`.
3. `feat(catalog): add B300 to the NUMA node count and NVLink capability checks` — `numa_node_count` (B300: 2) and
   `nvlink_capabilities`, both measured (below). `docs/reference` regenerated.

### Type

- [ ] Bug fix
- [x] New check / check enhancement
- [ ] Refactor / cleanup
- [ ] Documentation
- [ ] Dependency update

### Per-check applicability

| Check | B300 | Why |
| --- | --- | --- |
| `cpu_count` | already runs | Unrestricted (no `Architectures`); no change. |
| `numa_node_count` | yes (2) | A B300 host is two-socket x86_64 with no Grace C2C, so its count is the CPU NUMA layout, not GB300's 34. Measured: `/sys/devices/system/node/online` = `0-1` on both nodes (firmware default, one node per socket); a fleet that partitions sockets further overrides it. |
| `gpu_count` | yes (8) | 8 GPUs per HGX baseboard, as B200. |
| `gpu_power_limit` | yes (1100 W) | See thresholds and the SKU caveat. |
| `gpu_average_power_draw` | yes | The shared 65 W / 1500 W band holds on B300 (idle about 237 W, load about 1075 W median). |
| `gpu_power_state` | yes | B300 idles at P0 with persistence on, so the shared P0 default holds. |
| `gpu_sensor_throttle` | yes | Same NVML clock-event reasons. HW, thermal and power-brake slowdown read Not Active on a healthy B300. |
| `gpu_compute_apps` | yes | Architecture-independent (leftover compute contexts). |
| `nvlink_active` | yes | 18 of 18 NVLink 5 links active on every GPU. Shared budget 0. |
| `nvlink_bandwidth_count_speed` | yes | 18 links at 53125 MB/s, the same NVLink 5 rate as GB300 and B200. |
| `nvlink_capabilities` | yes | Measured: `nvidia-smi nvlink -c` on both nodes reports all six capabilities supported on every link (8 GPUs × 18 links × 6 = 864 of 864, driver 595). |
| `nvlink_p2p` | yes | All-to-all through the baseboard NVSwitches (`NV18` between every GPU pair). Shared budget 0. |
| `nvlink_fabric` | **no** | `ConditionScope: DOMAIN` assumes a multi-node NVLink domain. On HGX B300 the fabric is the node's own baseboard. |
| `nvlink_fabric_health` | **no** | The same scope reason, and its remediation ladder ends in a domain-wide `hold`, which has no meaning for a node-local fabric. |
| `rdma_linkflap_fatal` | yes | B300 nodes carry NVIDIA ConnectX-8 (mlx5) RDMA rails. The journal outage parse and shared thresholds apply unchanged. |
| `gpu_xid_<code>` (all-architectures, Blackwell, Hopper-plus-Blackwell lists) | yes | Blackwell Xids. A check fires only when its Xid is logged. |
| `gpu_xid_144`–`gpu_xid_150` (NVLink5) | yes | NVLink 5 is present: 18 ports per GPU to the baseboard NVSwitches. |
| `gpu_xid_54` (HGX list) | yes, **please confirm** | B300 follows B200 as an HGX baseboard. We have no B300-specific entry from the public Xid catalog for this code; if maintainers read it differently, drop B300 from `hgxArchitectures`. |
| `gpu_xid_142` (Grace-Blackwell only) | no | NVENC3 Xid marked GraceBlackwell; does not apply to an HGX baseboard. |

### Thresholds and their sources

All measurements are on HGX B300 nodes whose GPUs report the product name `NVIDIA B300 SXM6 AC`.

| Threshold | B300 value | Source |
| --- | --- | --- |
| `gpu_count.expected_gpu_count` | **8** | `nvidia-smi -L` and `lspci -d 10de:` (device `10de:3182`) on two validation nodes, driver 595.71.05, 2026-09-26. 8 allocatable GPUs on every GPU node of a 16-node cluster (2026-10-13); 8 × B300 per node across a 323-node cluster (2026-10-10). |
| `gpu_power_limit.expected_power_limit_w` | **1100 W** | `nvidia-smi -q` GPU Power Readings: Current = Default = Max power limit 1100 W (min 200 W) on every GPU of two nodes, driver 595.71.05, 2026-09-26. A paired `nvidia-smi -q -x` capture on one node, driver 595.91.07, 2026-10-13: current, default, max and requested limits all 1100 W on 8/8 GPUs. A 59-node, 472-GPU segment of the larger cluster (driver 595.91.07, 2026-10-11) had every GPU at the 1100 W default. Under a sustained bf16 matmul the GPUs held a median of 1093 W against the limit, with the SW power cap active. |
| `nvlink_bandwidth_count_speed.expected_speed_mbytes_per_sec` | **53125** | `nvidia-smi nvlink -s`: 18 of 18 links at 53.125 GB/s on every GPU of two nodes (144/144 links), 2026-09-26; again on 8/8 GPUs of the 2026-10-13 capture. |
| `nvlink_bandwidth_count_speed.expected_link_count` | **18** | As above. |
| `numa_node_count.expected_numa_node_count` | **2** | `/sys/devices/system/node/online` = `0-1` on two nodes, firmware default. |

The shared defaults that B300 now inherits were checked against the same data:

| Threshold (shared) | Value | B300 data |
| --- | --- | --- |
| `gpu_average_power_draw.min_power_w` / `max_power_w` | 65 / 1500 W | **Idle:** median 237 W and p90 245 W per GPU over about 460 GPUs (DCGM power, 15 s samples, 2026-10-10); 230–245 W on the 2026-10-13 capture. **Under `gpu_burn`:** median 1075 W, max sample 1117 W (DCGM, 30 s). NVML readings of 1.16–1.48 kW were also seen between scrapes, inside 1500 W but close to it; this check grades the one-second average. |
| `gpu_power_state.expected_gpu_pstate` | 0 (P0) | P0 on 8/8 GPUs of an idle node, persistence enabled (2026-10-13 capture). |
| `nvlink_active.max_inactive_nvlinks` | 0 | 144/144 links active on two nodes. |
| `nvlink_p2p.max_unhealthy_p2p_peers` | 0 | `nvidia-smi topo -m` reads `NV18` between every GPU pair. `nvbandwidth` device-to-device pair minimum about 762 GB/s (read) on 32 nodes (2026-09-26) and 58 nodes (2026-10-10). |
| `rdma_linkflap_fatal.min_outage_seconds` / `fatal_flap_threshold` | 30 s / 1 | No B300-specific value proposed. The rails are mlx5 (ConnectX-8), as on GB300. |

**SKU caveat for 1100 W.** We measured one B300 board variant (`SXM6 AC`). Another B300 variant with a different
default power limit would fail `gpu_power_limit` until the fleet overrides `expected_power_limit_w`. #225 left B200
off `gpu_power_limit` for the same reason (air- and liquid-cooled power budgets differ). If maintainers prefer
that, drop `B300` from `gpu_power_limit` and keep the rest.

### Reconciling with main (done on this branch)

The patches were written against `catalog/v0.3.1`. On `main@0ed4778`:

- **#216 (single module):** the root `go.mod` now covers `catalog/` and `toolkit/`; tests run from the root.
- **#166 (`ValueType` on every threshold):** the B300 entries live in existing `PerArchitecture` maps that already
  declare a `ValueType`, so nothing new was needed. `PerArchitecture` maps must now cover every listed
  architecture, so B300 was added to `expected_link_count` too.
- **#194 (per-code Xid architecture lists):** B300 was added to the shared lists rather than per check (see the
  table).
- **#225 (B200, merged):** every conflict was B200 and B300 touching the same `Architectures` lists, comments and
  tests; resolved by listing both. B300 follows B200 everywhere except `gpu_power_limit` (B300 is measured) and
  `numa_node_count` / `nvlink_capabilities` (B300 is measured; B200 is not on them).
- **VR200 (#238–#245):** kept as is; B300 goes next to B200 in the lists.
- **#262 (toolkit DCGM/NVML collect):** `architectureFromName` in `toolkit/nvml` now delegates to
  `toolkit/internal/gpuinventory`, so the B300 name case moved there (following #238). The new
  `averagedPowerDrawArchitectures` gate needed B300, or `TestAveragePowerDrawChecksStayOnAveragingArchitectures`
  fails.

### Follow-ups (not this PR)

1. **Node-scoped NVLink fabric checks.** HGX B300 GPUs do report fabric state. On two nodes (driver 595.91.07,
   2026-10-23) and the 2026-10-13 capture, NVML shows `State=Completed`, `Status=Success`, health summary
   `Healthy`, with the bandwidth and route fields `N/A`, `CliqueId` 0 and an all-zero `ClusterUUID`. A node-scoped
   variant of `nvlink_fabric`, with a node-level action, would cover HGX (B200 and B300). It should not reuse the
   domain-scoped definition.

### Test plan

- [x] `go test ./...` at the repository root (Go 1.27.3, linux/arm64): 53 packages pass, none fail.
- [x] `go vet ./...` clean; `gofmt -l catalog toolkit collect` clean.
- [x] `docs/reference` regenerated with the same pipeline as `//docs:reference_update`
  (`healthdump` → `healthschema` → `gencheckpages`), run with `go run`.
- [ ] `bazel test //...` not run (no Bazel on the authoring host), so `//docs:reference_update_test`,
  `//tools:check_spdx` and nogo were not run.
- [ ] `bazel run //tools/format` (gofumpt) not run. No struct layouts change, so `fieldalignment` is unaffected.
- [x] Changed checks have table-driven tests covering PASS and CRITICAL on B300.
- [ ] Tested on hardware: not yet with this patch. The values come from B300 measurements; the patched catalog has
  not been run against live B300 telemetry.

### Checklist

- [x] PR title follows Conventional Commits.
- [x] Commit messages describe why.
- [x] Every commit has a DCO `Signed-off-by` trailer.
- [ ] Every commit is cryptographically signed (re-sign before opening, see the top of this file).
- [x] No internal tooling references, URLs, or credentials in the diff.
- [x] No dependency change (`mod tidy` not needed).
- [x] No struct field changes.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
