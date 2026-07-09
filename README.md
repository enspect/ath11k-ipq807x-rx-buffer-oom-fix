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
death-spiral to a stable **115–190 MB**. There is **no upstream fix** as of writing.

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

## License

- **Patch** (`*.patch`): GPL-2.0, matching the Linux kernel / OpenWrt source it modifies.
- **Documentation** (this README and other prose): CC0-1.0 (public domain) — copy, adapt,
  and reuse freely.

See [`LICENSE`](./LICENSE).

---

*Diagnosed and written up from a real deployment. No upstream fix existed at time of
writing; this is offered so other ipq807x/ath11k users hitting the same hang (or
watchdog-reboot) have a root cause and a working mitigation to start from.*
