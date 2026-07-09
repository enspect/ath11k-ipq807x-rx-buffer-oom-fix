# ath11k RX-buffer memory exhaustion on low-RAM IPQ807x (Linksys MX4200) — diagnosis & fix

**TL;DR:** On OpenWrt with `ath11k`, low-RAM IPQ807x/IPQ8074 devices (e.g. the Linksys
MX4200, ~512 MB installed / ~372 MB usable) can **run out of RAM under sustained
Wi-Fi/mesh load and hang**. The culprit is ath11k's **RX buffer pools**, which are
**page-fragment-backed** (2 KB buffers carved from order-3 32 KB pages ≈ **16×
amplification**) and sized with an idr cap of `bufs_max × 3`. Under load they fill toward
that cap and exhaust RAM → OOM → the device locks up. **`kmemleak` and `/proc/slabinfo`
both miss it** because the consumed space is page-allocator memory, not slab.

The fix — a small OpenWrt patch that **halves the ath11k RX/monitor ring sizes** — caps
the footprint. In testing it took a 4-node MX4200 mesh from *locking up during a single
Plex movie* to **hours of streaming with no incident**, free memory rising from a ~30 MB
death-spiral to a stable **115–190 MB**. **No fix for this has been merged into the
OpenWrt tree** as of writing — see [Prior art & credit](#prior-art--credit) for related
independent work.

---

## A note on the symptom: hang, leading to a manual reboot

The native failure mode is a **hard hang** — the AP stops forwarding, becomes
unresponsive, and needs a power cycle to recover. If you install a low-RAM watchdog (a
cron job that reboots the device when `MemFree` drops below a threshold), the same
underlying OOM instead presents as **repeated reboots under load**. If you're seeing an
ipq807x AP either freeze or reboot-cycle when busy, this is a strong candidate.

## Affected setups

- **SoC:** Qualcomm IPQ8074 / `qualcommax` (ipq807x) — most exposed on ~512 MB boards.
- **Device seen on:** Linksys MX4200 v1 (~372 MB usable RAM).
- **Driver:** `ath11k` (tested via mac80211-backports 6.18.26 on OpenWrt 25.12.4 `r32933`).
- **Trigger:** sustained RX load — heavy streaming, many associated clients, and/or mesh
  (batman-adv) forwarding. Worsens as client count / throughput grows.

Higher-RAM ipq807x boards are far less likely to hit this because they have the headroom
to absorb the oversized pools.

## Symptoms

- `MemAvailable` / `MemFree` sawtooths downward under load, trending toward a few tens of MB.
- Eventually: OOM → **hang** (or **reboot**, if a low-RAM watchdog is installed).
- **`Slab` in `/proc/meminfo` stays flat** while `MemFree` bleeds away.
- **Nothing** shows in `kmemleak`; **nothing** obvious in `/proc/slabinfo`.

## Root cause

ath11k's RX buffer pools — the data path (`ath11k_dp_rxbufs_replenish`), the
monitor-status ring, and the copy-engine — allocate skbs via `dev_alloc_skb()`, whose
data area comes from the kernel **page-fragment cache**: order-3 (32 KB) pages sliced into
~2 KB fragments. **A single referenced fragment pins its entire 32 KB page**, so the
pools' effective memory footprint is a large multiple of their nominal byte count. With
the pools' idr capacity of `bufs_max × 3`, they fill under sustained load and the pinned
32 KB pages exhaust the ~372 MB RAM.

This is a **bounded-but-oversized-for-the-RAM** condition, not a classic unbounded leak.
The skb *heads* are slab-backed (flat, accounted); the skb *data buffers* are
page-allocator-backed (unaccounted by slab tooling) — which is exactly why `kmemleak` and
`/proc/slabinfo` are blind to it.

## How it was diagnosed (reproducible)

1. Build the image with `CONFIG_PAGE_OWNER=y` and boot with `page_owner=on`.
2. Under load, aggregate `/sys/kernel/debug/page_owner` by allocating call stack.
3. `__page_frag_alloc_align` dominates, tracing to the ath11k RX buffer replenish paths.
4. `/proc/meminfo` confirms the growth is **page-allocator**, not slab (`Slab` flat while
   `MemFree` falls).

## The fix

An OpenWrt patch under `package/kernel/mac80211/patches/ath11k/` that **halves the
RX/monitor ring sizes** in `drivers/net/wireless/ath/ath11k/dp.h`:

| define | stock | patched |
|---|---|---|
| `DP_RXDMA_BUF_RING_SIZE` | 4096 | **2048** |
| `DP_RXDMA_REFILL_RING_SIZE` | 2048 | **1024** |
| `DP_RXDMA_MON_STATUS_RING_SIZE` | 1024 | **512** |
| `DP_RXDMA_MONITOR_BUF_RING_SIZE` | 4096 | **2048** |
| `DP_RXDMA_MONITOR_DST_RING_SIZE` | 2048 | **1024** |
| `DP_RXDMA_MONITOR_DESC_RING_SIZE` | 4096 | **2048** |

The patch file is [`950-ath11k-shrink-rx-buffer-rings.patch`](./950-ath11k-shrink-rx-buffer-rings.patch)
in this repo. Because it lives in the build tree, it **survives future firmware rebuilds**.

## Applying it

```sh
# in your OpenWrt build tree:
mkdir -p package/kernel/mac80211/patches/ath11k
cp 950-ath11k-shrink-rx-buffer-rings.patch package/kernel/mac80211/patches/ath11k/
# rebuild the image as usual, flash config-preserving (sysupgrade without -n)
```

## Results

Measured end-to-end on a 4-node MX4200 mesh under a real Plex movie:

| | before | after |
|---|---|---|
| free RAM under load | ~30 MB death-spiral | stable **115–190 MB** |
| behaviour under load | hang / (watchdog) reboot mid-movie | **hours of streaming, no incident** |

## Trade-offs & tuning

Halving the rings reduces the RX buffering headroom. On the ~372 MB MX4200 this is a clear
net win (stability >> a theoretical peak-throughput ceiling that the box can't sustain
anyway). **Higher-RAM or very high-throughput ipq807x boards may prefer the stock (larger)
rings** — so if you adapt this for a wider audience, consider conditioning the sizes on
available RAM (or a build/DT option) rather than halving unconditionally. Values here are a
known-good starting point, not a universally optimal one.

## Prior art & credit

This is **not the first time** someone has shrunk the ath11k RX rings to survive on a
low-RAM board — and it's worth being clear about that.

- **Yanko Yankulov** independently arrived at the same lever (reducing the ring sizes in
  `ath11k/dp.h`) for the **ipq5018 / 256 MB** Mercusys MR80X, published ~March 2026:
  - forum: *"A simple patch to get ath11k working on 256MiB ram"* —
    <https://forum.openwrt.org/t/a-simple-patch-to-get-ath11k-working-on-256mib-ram/247147>
  - commit: <https://github.com/yanko-yankulov/openwrt-mr80x/commit/ee7747f48999cae66e686d464dceaf838398309e>

I reached the same mechanism independently, which is really **mutual validation** that
ath11k's RX-buffer sizing is the culprit on low-RAM boards. What this repo adds on top of
that prior work is:

1. **ipq807x / 512 MB (MX4200)-specific tuning** — different SoC and RAM budget than the
   256 MB ipq5018 case, so different ring values (a gentle halving rather than the more
   aggressive cuts that suit a 256 MB board).
2. **A documented root-cause method** — the `page_owner` diagnosis showing the
   page-fragment 16× amplification and *why* `kmemleak` / `/proc/slabinfo` miss it.

Neither patch is in the upstream OpenWrt tree; a properly mergeable fix would likely need
to be conditioned on available RAM (see *Trade-offs & tuning*).

## License

- **Patch** (`*.patch`): GPL-2.0, matching the Linux kernel / OpenWrt source it modifies.
- **Documentation** (this README and other prose): CC0-1.0 (public domain) — copy, adapt,
  and reuse freely.

See [`LICENSE`](./LICENSE).

---

*Diagnosed and written up from a real deployment. No fix was merged in the OpenWrt tree at
time of writing (see [Prior art & credit](#prior-art--credit) for related independent
work); this is offered so other ipq807x/ath11k users hitting the same hang have a root
cause and a working mitigation to start from.*

*Diagnosis and writeup by **enspect**, with an AI assistant (Claude Code).*
