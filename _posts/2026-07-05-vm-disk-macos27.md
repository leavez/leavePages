---

layout:     post
title:      VM Disk Performance on macOS 27
category:   []
tags:       [小品]
published:  True
date:       2026-7-5

---

> Written by AI, supervised by myself

A follow-up to [Disk Passthrough Performance in Apple Virtualization.framework]({% post_url 2026-04-12-vm-disk-test %}). Same question, newer OS.

The earlier post ended with three conclusions: the macOS Virtio block driver caps out at roughly 40K random IOPS regardless of backend; NVMe (`VZNVMExpressControllerDeviceConfiguration`) crashes at `vm.start()` when attached to a real block device; and file-backed disks lose heavily on random writes.

WWDC 2026 session 224 brought two disk-related changes: the **ASIF** sparse image format (via DiskImageKit), and a **custom Virtio device** API. I re-ran the tests on macOS 27 (beta 2), across three guests: macOS 27, macOS 26.5, and Linux.

## Test Environment

* M4 Pro / 24GB / Apple SSD / macOS 27.0 beta 2 (26A5368g)
* Test partition: disk0s3 (21GB), unmounted, raw access via `/dev/rdisk0s3`
* Guests: macOS 27.0 (26A5368g, same build as the host) and macOS 26.5, 4 vCPU / 8GB; Fedora 43 (Linux 6.17), 4 vCPU / 4GB
* fio 3.42 (host and macOS guests) / 3.40 (Fedora), `direct=1`, `ioengine=posixaio`
* 4K tests at `iodepth=32`, 1M tests at `iodepth=4`
* 3 runs per config, median reported.

Configurations, per guest:

| Config | Virtual Controller | Storage Backend |
|--------|--------------------|-----------------|
| **Host** | — | Direct raw partition access |
| **Virtio Raw (part)** | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → /dev/rdisk0s3 |
| **Virtio NBD (part)** | VZVirtioBlockDevice | VZNetworkBlockDeviceAttachment → nbd://127.0.0.1 → qemu-nbd (`--cache=none`) → /dev/rdisk0s3 |
| **Virtio Raw (img)** | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → /dev/rdiskN of an hdiutil-attached raw image |
| **Virtio File** | VZVirtioBlockDevice | VZDiskImageAttachment → raw sparse file |
| **Virtio ASIF** | VZVirtioBlockDevice | VZDiskImageAttachment → ASIF image |
| **NVMe ...** | VZNVMExpressController | (see the NVMe section) |

The VMs themselves are custom-written — a bit over a hundred lines of code, compiled against the macOS 27 SDK.

About the ASIF image: created blank with `diskutil image create blank --format ASIF`, then attached whole to the VM. It's a flat single-layer sparse file — space allocated in 1 MB chunks tracked by a bitmap, no backing-file chain or internal layering, and no encryption.

## Results

### macOS guest (27, same build as host)

| Test | Host | Virtio File | Virtio ASIF | Virtio Raw (part) | Virtio NBD (part) | Virtio Raw (img) |
|------|-----:|------------:|------------:|------------------:|------------------:|-----------------:|
| seq write 1M | 428 MB/s | 447 (104%) | 416 (97%) | 507 (118%) | 501 (117%) | 417 (97%) |
| seq read 1M | 4,391 MB/s | 2,902 (66%) | 989 (23%) | 4,380 (100%) | 4,097 (93%) | 2,720 (62%) |
| seq write 4K | 147K IOPS | 43.0K (29%) | 54.0K (37%) | 60.5K (41%) | 52.2K (36%) | 13.2K (9%) |
| seq read 4K | 195K IOPS | 66.3K (34%) | 36.2K (19%) | 58.9K (30%) | 48.0K (25%) | 39.4K (20%) |
| rand write 4K | 63.2K IOPS | 29.4K (47%) | 57.1K (90%) | 52.1K (83%) | 46.0K (73%) | 7.2K (11%) |
| rand read 4K | 141K IOPS | 14.7K (10%) | 54.4K (39%) | 55.6K (40%) | 42.2K (30%) | 12.0K (9%) |
| randrw 4K write | 115 MB/s | 17 (15%) | 61 (53%) | 50 (43%) | 43 (37%) | 11 (10%) |
| randrw 4K read | 268 MB/s | 41 (15%) | 141 (53%) | 116 (43%) | 100 (37%) | 27 (10%) |

*(Percentages are vs Host — the raw partition. 1M / randrw in MB/s, 4K in IOPS. randrw = 70/30 mixed.)*

Notable findings:

* **Sequential is near native; the gap is in small I/O.** Virtio Raw does 100–118% of host on 1M, then the 4K numbers flatten between 52K and 60K while the host does 63–195K on the same partition. This is the original's ~40K ceiling, higher on M4 + a 27 guest.
* **What backs the block device dominates.** The same guest drops from 52–56K random on the partition to 7–12K when the block device is an hdiutil-attached image — a bigger swing than any controller or format choice.
* **File and ASIF go partly through the host cache.** ASIF random write reaches 90% of the partition-host and File random write 47%, but their honest disk figure is random read (10%, 39%), where the cache can't help. Read the block-device columns (Raw part / NBD) for real-disk behavior.
* **ASIF's only real weakness now is sequential read.** Random is healthy (54K read), but sequential read collapses to 989 MB/s — a third of what the plain raw sparse file manages (2,902). (On a 26.5 guest ASIF collapsed on *everything*; see below.)
* **No NVMe column:** `validate()` rejects NVMe for Mac guests — "NVM Express Controller device must be configured with a generic platform".

### The guest version matters more than the APIs

Running the same tests on a **macOS 26.5** guest (older Virtio driver, same host and disk) changed three things:

| 4K, partition | 26.5 guest | 27 guest |
|---------------|-----------:|---------:|
| rand read | 47.4K | 55.6K |
| rand write | 42.0K | 52.1K |
| NBD rand read | 54.3K | 42.2K |
| ASIF rand read | 19.6K | 54.4K |
| ASIF seq read 1M | 691 MB/s | 989 MB/s |

Three separate effects:

* **The 27 guest's block driver is faster.** On the direct partition it beats 26.5 outright — random read 47.4K → 55.6K, write 42.0K → 52.1K.
* **The ASIF collapse was a 26.5-guest bug.** Gone on 27; random is back to normal.
* **NBD and the direct path swap places.** On 26.5, NBD beat the direct attachment (54K vs 47K); on 27 the direct path caught up and passed it (56K vs 42K).

So the ceiling isn't the host OS's call alone — the guest's driver version moves it too, and it's what decides which backend wins.

### Linux guest

| Test | Host | Virtio Raw (part) | Virtio Raw (img) | Virtio File | Virtio ASIF | NVMe Raw (img) | NVMe File |
|------|-----:|------------------:|-----------------:|------------:|------------:|---------------:|----------:|
| seq write 1M | 428 MB/s | 618 (144%) | 3,360 (785%) | 892 (208%) | 609 (142%) | 596 (139%) | 334 (78%) |
| seq read 1M | 4,391 MB/s | 2,284 (52%) | 2,524 (57%) | 1,381 (31%) | 789 (18%) | 1,758 (40%) | 727 (17%) |
| seq write 4K | 147K IOPS | 27.7K (19%) | 12.2K (8%) | 24.8K (17%) | 21.5K (15%) | 11.9K (8%) | 24.2K (16%) |
| seq read 4K | 195K IOPS | 26.3K (14%) | 18.9K (10%) | 18.5K (9%) | 14.7K (8%) | 17.8K (9%) | 25.8K (13%) |
| rand write 4K | 63.2K IOPS | 26.5K (42%) | 8.7K (14%) | 22.4K (35%) | 19.5K (31%) | 8.4K (13%) | 17.2K (27%) |
| rand read 4K | 141K IOPS | 15.8K (11%) | 7.6K (5%) | 15.4K (11%) | 11.0K (8%) | 8.4K (6%) | 12.2K (9%) |
| randrw 4K write | 115 MB/s | 21 (18%) | 8 (7%) | 19 (17%) | 13 (11%) | 10 (9%) | 15 (13%) |
| randrw 4K read | 268 MB/s | 49 (18%) | 19 (7%) | 43 (16%) | 30 (11%) | 24 (9%) | 36 (13%) |

*(Same Host baseline and units. NVMe + partition is missing because it crashes — see the NVMe section. NBD was only tested with the macOS guest.)*

Notable findings (only where Linux differs from the macOS guest):

* **NVMe exists here** — generic-platform-only — but ≈ Virtio on the same backend: 8.7K/7.6K (virtio) vs 8.4K/8.4K (NVMe) on random 4K. Multiple NVMe queues, no difference. The limit is not the number of queues.
* **Large sequential writes get coalesced much harder.** Virtio Raw 1M writes hit 144% of the disk on the partition, 3,360 MB/s (785%) on the image, where the macOS guest stayed near native (118%). The Linux virtio driver batches aggressively; macOS's doesn't.
* **The 4K ceiling is lower than the macOS guest's** — Linux ~16–28K on the partition against macOS's ~52–56K. Different guest driver, different ceiling.
* **ASIF trails moderately** — 13–43% behind the raw file (worst on sequential read), not the 26.5-macOS collapse.

## Discussion

### NVMe: still no real passthrough

The original post's crash is unchanged. NVMe + `VZDiskBlockDeviceStorageDeviceAttachment` on the physical partition fails at `vm.start()` with the same "Internal Virtualization error", on macOS 27, reproduced verbatim:

| Guest | Attachment | Target | 15.5 / 26.3.1 | **27.0** |
|-------|-----------|--------|:-------------:|:--------:|
| Linux | Block device | physical partition | ❌ crash | ❌ crash |
| Linux | Block device | disk image via hdiutil | — | ✅ boots, namespace works |
| Linux | File | regular file | ✅ | ✅ |
| macOS | any | — | ❌ not supported | ❌ rejected at `validate()` |

Two details did move. A block device backed by an hdiutil-attached *disk image* now boots with a working namespace at `/dev/nvme0n1` — but the I/O still goes through the host's disk-image machinery, so it's not passthrough, and its numbers sit on top of Virtio's. And for macOS guests, what used to be an undocumented "doesn't work" is now an explicit configuration error: "NVM Express Controller device must be configured with a generic platform."

### Where Virtio is limited

Session 224 made two Virtio changes that could affect performance: the implementation now follows **VIRTIO 1.4**, and the new custom Virtio device API allows a custom queue count (covered below). The ceiling has always been pinned on Virtio, so these two changes are a good excuse to walk through where it could actually be the limit.

The queue count first. In the guest, virtio-blk negotiates exactly **one** virtqueue — `VIRTIO_BLK_F_MQ` isn't set. If a single queue were the bottleneck, adding queues should help; but NVMe comes up with multiple I/O queues and performed the same:

| Device | Guest queues |
|--------|:------------:|
| virtio-blk | 1 |
| NVMe · 6 vCPU | 5 |
| NVMe · 4 vCPU | 3 |

(The counts come straight from the guest: virtio-blk's single queue from the negotiated virtio feature bits, the NVMe figures by counting blk-mq hardware queues under `/sys/block/nvme0n1/mq/`.)

If more queues change nothing, a single queue is not the limit.

The spec version next. The features that matter for block performance are multi-queue (`VIRTIO_BLK_F_MQ`) and packed virtqueues (`VIRTIO_F_RING_PACKED`), and both predate 1.4 — packed rings appeared in VIRTIO 1.1. What the revisions since add (admin virtqueues and device groups in 1.3, new device types along the way) is control-plane machinery, outside the virtio-blk data path. macOS 27 already negotiates `VIRTIO_F_RING_PACKED`, and rebuilding the launcher against the 27 SDK changed nothing.

The Raw (part) columns give the cross-guest view: on the same partition where the host does 63–195K on 4K, a macOS 27 guest flattens around 52–56K, a macOS 26.5 guest around 42–47K, and a Linux guest around 16–27K. The same disk gives three different ceilings, and the biggest variable is the guest's own driver version — not the controller, the format, or the VIRTIO spec. What stays below native is the per-request cost of the virtualization I/O path, and it sits in the guest driver as much as in the host.

### NBD

Network Block Device (NBD) is a minimal protocol from the late-90s Linux world for serving "a disk" over TCP: a server exports something — a file, a partition, a qcow2 image — as raw readable/writable sectors, and a client presents it to the local system as an ordinary block device. It's much simpler than iSCSI (just read/write/flush/trim), which is why the virtualization world leans on it. Here the client is Virtualization.framework itself: `VZNetworkBlockDeviceStorageDeviceAttachment` (macOS 14+) takes an `nbd://` URL and the guest sees a plain Virtio disk — it never knows the backend is a socket.

#### Using NBD locally

Since it's a supported protocol, we can try it: point that client at a qemu-nbd server on localhost exporting the same partition (`--cache=none`, so zero real network), and compare it to the direct attachment. On a **26.5** guest, NBD actually beat the direct block-device attachment (54K vs 47K random) — the old direct path was weak enough that routing through localhost NBD was faster. On a **27** guest the direct path caught up, and NBD sits *behind* it: 42.2K/46.0K random read/write against 55.6K/52.1K direct, about 76–88%. And it isn't free on CPU:

| Path (27 guest, sustained 4K random read) | Random read | Host CPU |
|-------------------------------------------|------------:|---------:|
| Virtio direct (partition) | 55.6K | VM 521% |
| Virtio NBD (partition) | 42.2K | VM 381% + qemu-nbd 166% = **547%** |

NBD does *less* work for *more* total CPU — about 13% per 1K IOPS against 9% for the direct path, once the qemu-nbd process is counted.

#### Pairing the VM with a cloud disk

Pairing NBD with a cloud-disk backend — compute and storage decoupled — makes for a flexible deployment: the backend can be detached, migrated, snapshotted, or shared between machines. NBD is the framework's one formal interface for network storage, and the performance is respectable — no order-of-magnitude gap.

So what can serve an `nbd://` URL? Almost nothing speaks NBD natively (cloud block volumes attach only to their own hypervisor's VMs; a NAS speaks iSCSI/NFS/SMB), so you run a small gateway that re-exports your storage as NBD. The simplest viable one is qemu-nbd's `rbd` driver — `qemu-nbd rbd:pool/image` turns a Ceph RBD volume into an `nbd://` URL in one command (`rbd-nbd` won't do — it goes the other way, mapping RBD to a local `/dev/nbdX`).

The relay is two hops, though: Mac → gateway (NBD) → Ceph (RBD). Keep the gateway next to the cluster, so the only hop carrying the RTT that random IOPS is sensitive to is Mac → gateway; a box floating far from both pays the latency twice. And librbd is effectively Linux-only, so the gateway won't run on the Mac itself.

### Any other options

`VZVirtioBlockDeviceConfiguration` exposes a single property in the 27 runtime — `blockDeviceIdentifier`. There is no queue-count setting.

The framework offers two other routes, and neither is a drop-in replacement:

- **`VZCustomVirtioDeviceConfiguration`**, the custom Virtio device from session 224. It does expose `virtioQueueCount` and custom feature sets, but you implement the block-device protocol yourself and provide a matching guest driver — a real disk driver. On Linux that's a well-trodden path: a kernel module modeled on virtio-blk, itself about a thousand lines. On a macOS guest it's effectively a dead end: block drivers still live in kext territory, kexts have been deprecated for years, signing one requires a special certificate from Apple, and Apple Silicon won't load third-party kexts without lowering the security policy.
- **PCI passthrough**, which exists as a private SPI (`_pciPassthroughDevices` on `VZVirtualMachineConfiguration`). But an Apple Silicon laptop has no device to pass through — the only NVMe is the boot SSD.

## Conclusion

- The backend and the guest driver both strongly affect performance.
- OS 27 guest + Virtio raw partition is the best-performing combination; OS 27 guest + the ASIF image format performs about the same except on sequential read, while offering more flexibility — and without opening an order-of-magnitude gap against the host.
- NBD offers more flexibility at acceptable performance; with a cluster on hand, it's worth integrating.


<br />
<br />

---

<br />
<br />

(本文由 AI 撰写，由我把关。)

这是 [Disk Passthrough Performance in Apple Virtualization.framework]({% post_url 2026-04-12-vm-disk-test %}) 的后续。同一个问题，换了更新的系统。

上一篇有三个结论：macOS 的 Virtio 块驱动无论后端如何，随机 IOPS 都卡在约 40K；NVMe（`VZNVMExpressControllerDeviceConfiguration`）接真实块设备时在 `vm.start()` 崩溃；文件后端的随机写损失很大。

WWDC 2026 的 session 224 有两个和磁盘相关的改动：DiskImageKit 提供的 **ASIF** 稀疏镜像格式，和一个 **custom Virtio device** API。我在 macOS 27（beta 2）上重跑了一遍测试，并覆盖 guest os：macOS 27、macOS 26.5 和 Linux。

## 测试环境

* M4 Pro / 24GB / Apple SSD / macOS 27.0 beta 2 (26A5368g)
* 测试分区：disk0s3（21GB），未挂载，直接操作 `/dev/rdisk0s3`
* Guest：macOS 27.0（26A5368g，和宿主同一个 build）与 macOS 26.5，4 vCPU / 8GB；Fedora 43（Linux 6.17），4 vCPU / 4GB
* fio 3.42（宿主和 macOS guest）/ 3.40（Fedora），`direct=1`，`ioengine=posixaio`
* 4K 测试 `iodepth=32`，1M 测试 `iodepth=4`
* 每组跑 3 轮，取中位数。


每个 guest 的配置：

| 配置 | 虚拟控制器 | 存储后端 |
|--------|--------------------|-----------------|
| **Host** | — | 直接访问 raw 分区 |
| **Virtio Raw (part)** | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → /dev/rdisk0s3 |
| **Virtio NBD (part)** | VZVirtioBlockDevice | VZNetworkBlockDeviceAttachment → nbd://127.0.0.1 → qemu-nbd（`--cache=none`）→ /dev/rdisk0s3 |
| **Virtio Raw (img)** | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → hdiutil 挂载的 raw 镜像的 /dev/rdiskN |
| **Virtio File** | VZVirtioBlockDevice | VZDiskImageAttachment → raw 稀疏文件 |
| **Virtio ASIF** | VZVirtioBlockDevice | VZDiskImageAttachment → ASIF 镜像 |
| **NVMe ...** | VZNVMExpressController | （见 NVMe 一节） |

实际 VM 由自己编写，一百多行代码，并使用 macOS 27 SDK 编译。

其中 ASIF 镜像的具体形式：用 `diskutil image create blank --format ASIF` 创建的空白镜像，整盘挂给 VM。它是平铺的单层稀疏文件——空间按 1MB chunk 分配、位图记录，没有 backing 链或内部分层，也没加密。


## 结果

### macOS guest（27，和宿主同 build）

| Test | Host | Virtio File | Virtio ASIF | Virtio Raw (part) | Virtio NBD (part) | Virtio Raw (img) |
|------|-----:|------------:|------------:|------------------:|------------------:|-----------------:|
| seq write 1M | 428 MB/s | 447 (104%) | 416 (97%) | 507 (118%) | 501 (117%) | 417 (97%) |
| seq read 1M | 4,391 MB/s | 2,902 (66%) | 989 (23%) | 4,380 (100%) | 4,097 (93%) | 2,720 (62%) |
| seq write 4K | 147K IOPS | 43.0K (29%) | 54.0K (37%) | 60.5K (41%) | 52.2K (36%) | 13.2K (9%) |
| seq read 4K | 195K IOPS | 66.3K (34%) | 36.2K (19%) | 58.9K (30%) | 48.0K (25%) | 39.4K (20%) |
| rand write 4K | 63.2K IOPS | 29.4K (47%) | 57.1K (90%) | 52.1K (83%) | 46.0K (73%) | 7.2K (11%) |
| rand read 4K | 141K IOPS | 14.7K (10%) | 54.4K (39%) | 55.6K (40%) | 42.2K (30%) | 12.0K (9%) |
| randrw 4K write | 115 MB/s | 17 (15%) | 61 (53%) | 50 (43%) | 43 (37%) | 11 (10%) |
| randrw 4K read | 268 MB/s | 41 (15%) | 141 (53%) | 116 (43%) | 100 (37%) | 27 (10%) |

*(括号内为 vs Host 百分比，Host 为 raw 分区直接访问。1M / randrw 单位 MB/s，4K 单位 IOPS。randrw = 70/30 混合。)*

几点关注：

* **顺序接近原生，差距在小 I/O。** Virtio Raw 的 1M 到宿主的 100–118%，但 4K 全压在 52K–60K 之间，而宿主在同一块分区上是 63K–195K。这就是原文的 ~40K 天花板，在 M4 + 27 guest 上更高一些。
* **块设备背后是什么，起决定作用。** 同一个 guest，块设备从分区换成 hdiutil 挂载的镜像，随机就从 52–56K 掉到 7–12K——比任何控制器或格式的差别都大。
* **File 和 ASIF 有一部分走了宿主缓存。** ASIF 随机写到分区宿主的 90%、File 到 47%，但它们老实的磁盘数字是随机读（10%、39%），那里缓存帮不上忙。看块设备列（Raw part / NBD）才是真实磁盘表现。
* **ASIF 现在唯一的真短板是顺序读。** 随机已经正常（读 54K），但顺序读塌到 989 MB/s——只有同为文件后端的 raw 稀疏文件（2,902）的三分之一。（26.5 guest 上 ASIF 是*全线*塌方，见下。）
* **没有 NVMe 列**：`validate()` 直接拒绝 Mac guest 配 NVMe——"NVM Express Controller device must be configured with a generic platform"。

### guest 版本比新 API 更关键

同样的测试换到 **macOS 26.5** guest（更旧的 Virtio 驱动，宿主和盘不变），有三处变化：

| 4K，分区 | 26.5 guest | 27 guest |
|---------|-----------:|---------:|
| 随机读 | 47.4K | 55.6K |
| 随机写 | 42.0K | 52.1K |
| NBD 随机读 | 54.3K | 42.2K |
| ASIF 随机读 | 19.6K | 54.4K |
| ASIF 顺序读 1M | 691 MB/s | 989 MB/s |

三个独立的效应：

* **27 guest 的块驱动更快。** 在直连分区上就是比 26.5 快一截——随机读 47.4K → 55.6K，写 42.0K → 52.1K。
* **ASIF 的塌方是 26.5 guest 的 bug。** 27 上没有了，随机恢复正常。
* **NBD 和直连的高下反了过来。** 26.5 上 NBD 快过直连（54K vs 47K），27 上直连追了上去、反超（56K vs 42K）。

所以天花板不是宿主 OS 一家说了算——guest 的驱动版本也在挪它，还决定了哪个后端更快。

### Linux guest

| Test | Host | Virtio Raw (part) | Virtio Raw (img) | Virtio File | Virtio ASIF | NVMe Raw (img) | NVMe File |
|------|-----:|------------------:|-----------------:|------------:|------------:|---------------:|----------:|
| seq write 1M | 428 MB/s | 618 (144%) | 3,360 (785%) | 892 (208%) | 609 (142%) | 596 (139%) | 334 (78%) |
| seq read 1M | 4,391 MB/s | 2,284 (52%) | 2,524 (57%) | 1,381 (31%) | 789 (18%) | 1,758 (40%) | 727 (17%) |
| seq write 4K | 147K IOPS | 27.7K (19%) | 12.2K (8%) | 24.8K (17%) | 21.5K (15%) | 11.9K (8%) | 24.2K (16%) |
| seq read 4K | 195K IOPS | 26.3K (14%) | 18.9K (10%) | 18.5K (9%) | 14.7K (8%) | 17.8K (9%) | 25.8K (13%) |
| rand write 4K | 63.2K IOPS | 26.5K (42%) | 8.7K (14%) | 22.4K (35%) | 19.5K (31%) | 8.4K (13%) | 17.2K (27%) |
| rand read 4K | 141K IOPS | 15.8K (11%) | 7.6K (5%) | 15.4K (11%) | 11.0K (8%) | 8.4K (6%) | 12.2K (9%) |
| randrw 4K write | 115 MB/s | 21 (18%) | 8 (7%) | 19 (17%) | 13 (11%) | 10 (9%) | 15 (13%) |
| randrw 4K read | 268 MB/s | 49 (18%) | 19 (7%) | 43 (16%) | 30 (11%) | 24 (9%) | 36 (13%) |

*(Host 基线和单位同上。没有 NVMe + 分区一列，因为它会崩溃——见 NVMe 一节。NBD 只在 macOS guest 上测了。)*

几点关注（只列和 macOS guest 不同的部分）：

* **这里有 NVMe**——generic 平台专属——但同后端下 ≈ Virtio：随机 4K 上 8.7K/7.6K（virtio）对 8.4K/8.4K（NVMe）。多个 NVMe 队列，没有差别。限制不在队列数。
* **大块顺序写被合并得凶得多。** Virtio Raw 的 1M 写在分区上是宿主的 144%，镜像上到 3,360 MB/s（785%），而 macOS guest 停在接近原生（118%）。Linux 的 virtio 驱动激进地攒批，macOS 的不这么做。
* **4K 天花板比 macOS guest 低**——Linux 在分区上约 16–28K，macOS 约 52–56K。guest 驱动不同，天花板不同。
* **ASIF 落后得不算多**——比 raw 文件低 13–43%（顺序读最差），不像 26.5 的 macOS 那样塌方。

## 讨论

### NVMe：依旧没有真直通

原文的崩溃没有变。NVMe + `VZDiskBlockDeviceStorageDeviceAttachment` 接物理分区，在 macOS 27 上照样在 `vm.start()` 报同一个 "Internal Virtualization error"：

| Guest | Attachment | 目标 | 15.5 / 26.3.1 | **27.0** |
|-------|-----------|------|:-------------:|:--------:|
| Linux | 块设备 | 物理分区 | ❌ 崩溃 | ❌ 崩溃 |
| Linux | 块设备 | hdiutil 挂的磁盘镜像 | — | ✅ 启动，namespace 可用 |
| Linux | 文件 | 普通文件 | ✅ | ✅ |
| macOS | 任意 | — | ❌ 不支持 | ❌ `validate()` 拒绝 |

有变化的是两个细节。用 hdiutil 挂载的*磁盘镜像*做块设备后端，现在能启动了，`/dev/nvme0n1` 带可用的 namespace——但 I/O 仍然走宿主的磁盘镜像那套机制，不是直通，数字也和 Virtio 基本重合。另外 macOS guest 这边，过去文档上查不到的"不能用"，现在是一条明确的配置错误："NVM Express Controller device must be configured with a generic platform"。

### Virtio 的限制在哪

session 224 在 Virtio 上有两个可能影响性能的改动：实现对齐到 **VIRTIO 1.4**，以及新的 custom Virtio device API 可以自定义队列数（后面说）。天花板一直被归在 Virtio 上，正好借这两个改动，把 Virtio 上可能构成限制的地方过一遍。

先看队列数。在 guest 里读协商结果，virtio-blk 只有**一个** virtqueue——`VIRTIO_BLK_F_MQ` 没有被协商。单队列自然是首先怀疑的。但 NVMe 起来就带多个 I/O 队列，结果没有差别：

| 设备 | Guest 队列 |
|--------|:------------:|
| virtio-blk | 1 |
| NVMe · 6 vCPU | 5 |
| NVMe · 4 vCPU | 3 |

（队列数都是在 guest 里直接读的：virtio-blk 的一条来自协商出来的 virtio feature bit，NVMe 的几条是数 `/sys/block/nvme0n1/mq/` 下的 blk-mq 硬件队列目录得到的。）

队列多了数字没有变化，说明限制不在单队列。

再看 spec 版本。对块设备性能有影响的两个 feature 是多队列（`VIRTIO_BLK_F_MQ`）和 packed virtqueue（`VIRTIO_F_RING_PACKED`），都早于 1.4——packed ring 在 VIRTIO 1.1 就有了。之后各版新增的内容（1.3 的 admin virtqueue 和 device groups，以及一路加进来的新设备类型）都是控制面的机制，不在 virtio-blk 的数据路径上。macOS 27 已经协商了 `VIRTIO_F_RING_PACKED`；把 launcher 对 27 SDK 重新编译，数字没有变化。

Raw (part) 列给出跨 guest 的视角：同一块分区，宿主 4K 是 63–195K，macOS 27 guest 压在 52–56K，macOS 26.5 guest 在 42–47K，Linux guest 在 16–27K。同一块盘，三个 guest 落在三个不同的天花板上，而这里最大的变量是 guest 自己的驱动版本，不是控制器、格式，也不是 VIRTIO spec。低于原生的那部分，是虚拟化 I/O 路径的单请求开销，它既在宿主、也在 guest 驱动里。

### NBD

NBD(Network Block Device)是上世纪 90 年代末从 Linux 来的一个极简协议，用来把"一块盘"通过 TCP 提供出去：服务端把某样东西——一个文件、一个分区、一个 qcow2 镜像——按可读写的裸扇区导出，客户端再把它当成一块普通块设备呈现给本地系统。它比 iSCSI 简单得多（只有读/写/flush/trim），所以虚拟化圈爱用它。这里的客户端就是 Virtualization.framework 自己：`VZNetworkBlockDeviceStorageDeviceAttachment`（macOS 14+）接一个 `nbd://` URL，guest 看到的就是一块普通 Virtio 盘，根本不知道背后是个 socket。


#### 本地使用 NBD 协议

既然它是一种支持的协议，我们也可以尝试一下：把这个 NBD 客户端指向 localhost 上的 qemu-nbd、导出同一块分区（`--cache=none`，零真实网络），再和直连比。在 **26.5** guest 上，NBD 确实快过直连块设备（随机 54K vs 47K）——旧的直连路径弱到绕一圈 localhost NBD 反而更快。到了 **27** guest，直连追了上来，NBD 落到它后面：随机读/写 42.2K/46.0K 对直连的 55.6K/52.1K，约 76–88%。而且它在 CPU 上不是免费的：

| 路径（27 guest，持续 4K 随机读） | 随机读 | 宿主 CPU |
|-------------------------------|------:|---------:|
| Virtio 直连（分区） | 55.6K | VM 521% |
| Virtio NBD（分区） | 42.2K | VM 381% + qemu-nbd 166% = **547%** |

NBD 做的活更少，占的总 CPU 反而更多——算上 qemu-nbd 进程，每 1K IOPS 约 13%，直连是 9%。

#### VM 配合云盘

使用 NBD 配合后端云盘，计算存储分离，在业务上是一个灵活的选择。后端可拆卸、可迁移、可快照、可在机器间共享。NBD 是框架给网络存储准备的唯一正式接口，并且有着还可以的性能，并没有数量级的差距。

什么东西能提供 `nbd://` 的服务呢：几乎没有存储服务原生对外提供 NBD（公有云块存储只挂自家 hypervisor 的 VM，NAS 走的是 iSCSI/NFS/SMB），所以得跑一个小网关，把手上的存储重新导出成 NBD。最简单可行的是 qemu-nbd 的 `rbd` 驱动——`qemu-nbd rbd:pool/image` 一条命令，就把一个 Ceph RBD 卷变成一个 `nbd://` URL（`rbd-nbd` 不行，它是反方向的，把 RBD 映射成本机的 `/dev/nbdX`）。

不过这套中转是两跳：Mac →（NBD）网关 →（RBD）Ceph 集群。网关要贴着集群放——真正拖累随机 IOPS 的那点 RTT，就只落在 Mac→网关这一跳上；网关要是飘在中间、离两边都远，延迟就得付两遍。而且 librbd 基本只在 Linux 上，网关跑不到 Mac 本机。

### 还有什么办法

`VZVirtioBlockDeviceConfiguration` 在 27 的运行时里只有一个属性——`blockDeviceIdentifier`，没有队列数的设置。

框架里还有两条路，都不是现成的替代品：

- **`VZCustomVirtioDeviceConfiguration`**，session 224 的 custom Virtio device。它有 `virtioQueueCount` 和自定义 feature set，但块设备协议要自己实现，guest 里还要配套驱动——真的要写一个磁盘驱动。Linux 这边是成熟套路，照着 virtio-blk（本身千行上下）写个内核模块就行；macOS guest 这边基本是死路：块设备驱动仍然是 kext 的领地，kext 已被 Apple 弃用多年，签名需要专门向 Apple 申请证书，Apple Silicon 上还得降低安全策略才能加载第三方 kext。
- **PCI 直通**，以私有 SPI 存在（`VZVirtualMachineConfiguration` 的 `_pciPassthroughDevices`）。但 Apple Silicon 笔记本没有可以直通的设备——唯一的 NVMe 是启动盘。

## 结论

- 后端和 guest 驱动都会对性能有很大影响
- OS 27 guest + virtio raw 分区是性能最好的组合；OS 27 guest + ASIF 文件格式，除了顺序读之外，性能基本类似，但提供了更好的灵活性，并且没有与 host 拉开数量级别的差距
- NBD 在可接受的性能下，提供了更好的灵活性，集群情况下可以考虑集成
