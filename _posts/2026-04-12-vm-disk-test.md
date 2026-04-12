---

layout:     post
title:      Disk Passthrough Performance in Apple Virtualization.framework
category:   []
tags:       [小品]
published:  True
date:       2026-4-12

---

> This article was written with the assistance of AI

Disk I/O is the main bottleneck for macOS VMs. This post benchmarks three configurations and discusses NVMe passthrough.

## Test Environment

* M1 Pro / 32GB / Apple SSD 2TB / macOS 15.5
* Test partition: disk0s5 (994.6GB), unmounted, raw access via `/dev/rdisk0s5`
* Guest: macOS 15.6.1, 6 vCPU / 8GB RAM
* fio 3.42, `direct=1`, `ioengine=posixaio`
* 4K tests at `iodepth=32`, 1M tests at `iodepth=4`

Three configurations:

| Config | Virtual Controller | Storage Backend |
|--------|-------------------|-----------------|
| **Host** | — | Direct raw partition access |
| **Virtio Raw** | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → /dev/rdisk0s5 |
| **Virtio File** | VZVirtioBlockDevice | VZDiskImageAttachment → APFS sparse file |

3 runs per config, median reported.

## Results

| Test | Host | Virtio Raw | Virtio File |
|------|-----:|-----------:|------------:|
| seq write 1M | 3,351 MB/s | 6,649 (198%) | 5,403 (161%) |
| seq read 1M | 5,769 MB/s | 4,293 (74%) | 3,618 (63%) |
| seq write 4K | 156K IOPS | 42K (27%) | 38K (24%) |
| seq read 4K | 90K IOPS | 38K (43%) | 36K (40%) |
| rand write 4K | 34K IOPS | 39K (114%) | 13K (39%) |
| rand read 4K | 39K IOPS | 25K (65%) | 21K (54%) |
| randrw 4K write | 43 MB/s | 28 (65%) | 20 (47%) |
| randrw 4K read | 102 MB/s | 67 (66%) | 46 (45%) |

*(Percentages are vs Host. randrw = 70/30 mixed read/write)*

Notable findings:

* **seq write 1M**: Virtio Raw hits 2x the Host throughput. The macOS Virtio driver aggressively coalesces large writes, achieving higher effective concurrency than Host posixaio at iodepth=4.

* **rand write 4K**: The largest gap between Raw and File. Raw slightly exceeds Host, while File drops to 39% — APFS Copy-on-Write requires extra block allocation, metadata updates, and journaling on every random write.

* **seq write/read 4K**: Raw and File are nearly identical (~38-42K IOPS), indicating the bottleneck is the macOS Virtio driver itself — roughly a **40K IOPS ceiling**, regardless of backend.

## NVMe Virtual Controller: Not Working

### Why NVMe

The results above show a ~40K IOPS ceiling in the macOS Virtio driver. Virtualization.framework also provides `VZNVMExpressControllerDeviceConfiguration`. NVMe protocol has lower per-I/O overhead than Virtio, so it could potentially break through this limit.

### Test Results

None of the combinations worked (same results on macOS 15.5 and 26.3.1):

| Controller | Attachment | Target | Result |
|-----------|-----------|--------|--------|
| NVMe | `VZDiskBlockDeviceAttachment` | /dev/rdisk0s5 | ❌ vm.start() crashes |
| NVMe | `VZDiskImageAttachment` | /dev/rdisk0s5 | ❌ Controller present, no namespace |
| NVMe | `VZDiskImageAttachment` | /dev/disk0s5 | ❌ Controller present, no namespace |
| NVMe | `VZDiskImageAttachment` | Regular file | ✅ Works |

> In macOS, `/dev/disk0s5` is a buffered block device (goes through system cache), while `/dev/rdisk0s5` is a raw character device (bypasses cache for direct access). Both point to the same physical partition but differ in I/O path. Performance tests typically use `rdisk` to avoid cache interference.

`VZNVMExpressControllerDeviceConfiguration` + `VZDiskBlockDeviceStorageDeviceAttachment` (the correct way to do physical device passthrough) always throws "Internal Virtualization error" at `vm.start()`. Tried synchronizationMode none and full, tried readOnly — all failed. The API-level config validation passes, but it crashes at actual startup.

Passing the device path to `VZDiskImageStorageDeviceAttachment` avoids the crash. The guest kernel recognizes the NVMe controller (`/dev/nvme0`), but there are no namespaces registered under it. In NVMe, a namespace is the actual logical disk, mapped to `/dev/nvme0n1`. No namespace means the controller is empty — like having an NVMe adapter card installed but no drive attached. The guest can see the controller but cannot use it.

NVMe only works correctly when the backend is a regular file. But that means all I/O goes through the host's APFS — this is not passthrough, and performance is worse than Virtio. Also, macOS guests do not support this controller at all; it only works with Linux guests on the Generic platform.

### What Is This API For 

`VZNVMExpressControllerDeviceConfiguration` was introduced in macOS 14 Sonoma (WWDC 2023). Apple's stated reason: some Linux kernels lack Virtio drivers but have NVMe drivers, so a standard controller interface is needed.

In practice, the community mostly uses it to work around a different issue: Virtio block devices with default caching mode corrupt ext4 filesystems in Linux guests. UTM ([PR #5919](https://github.com/utmapp/UTM/pull/5919)) and Lima ([#1957](https://github.com/lima-vm/lima/issues/1957)) both encountered this. NVMe serves as a workaround. However, UTM found NVMe to be **15-20% slower** than Virtio, and it doesn't fix the corruption on all kernels.

This API is designed for compatibility (booting more Linux kernels), not performance. It only works with file-backed images and cannot be used for physical device passthrough.


<br />
<br />

---

<br />
<br />

(文章由 AI 辅助生成)

磁盘性能是 macOS VM 的主要瓶颈。这篇文章测了三种配置下的实际表现，并且讨论了一下 NVMe 直通。

## 测试环境

* M1 Pro / 32GB / Apple SSD 2TB / macOS 15.5

* 测试分区：disk0s5 (994.6GB)，未挂载，直接操作 `/dev/rdisk0s5`

* Guest：macOS 15.6.1，6 vCPU / 8GB RAM

* fio 3.42，`direct=1`，`ioengine=posixaio`

* 4K 测试 `iodepth=32`，1M 测试 `iodepth=4`

三种配置：

| 配置              | 虚拟控制器               | 存储后端                                        |
| --------------- | ------------------- | ------------------------------------------- |
| **Host**        | —                   | 直接访问 raw 分区                                 |
| **Virtio Raw**  | VZVirtioBlockDevice | VZDiskBlockDeviceAttachment → /dev/rdisk0s5 |
| **Virtio File** | VZVirtioBlockDevice | VZDiskImageAttachment → APFS sparse file    |

每组跑 3 轮，取中位数。

## 结果

| Test            |       Host |   Virtio Raw |  Virtio File |
| --------------- | ---------: | -----------: | -----------: |
| seq write 1M    | 3,351 MB/s | 6,649 (198%) | 5,403 (161%) |
| seq read 1M     | 5,769 MB/s |  4,293 (74%) |  3,618 (63%) |
| seq write 4K    |  156K IOPS |    42K (27%) |    38K (24%) |
| seq read 4K     |   90K IOPS |    38K (43%) |    36K (40%) |
| rand write 4K   |   34K IOPS |   39K (114%) |    13K (39%) |
| rand read 4K    |   39K IOPS |    25K (65%) |    21K (54%) |
| randrw 4K write |    43 MB/s |     28 (65%) |     20 (47%) |
| randrw 4K read  |   102 MB/s |     67 (66%) |     46 (45%) |

*(括号内为 vs Host 百分比。randrw = 70/30 混合读写)*

几点关注：

* **seq write 1M**：Virtio Raw 跑到 Host 的 2 倍。macOS Virtio 驱动对大块写入做了激进的合并，实际并发度高于 Host posixaio 在 iodepth=4 时的表现。

* **rand write 4K**：Raw 直通略超 Host，而 File 只有 Host 的 39%，性能差距最大。

* **seq write/read 4K**：Raw 和 File 几乎一样（~38-42K IOPS），说明瓶颈在 macOS Virtio 驱动层，约 **40K IOPS 天花板**，与后端无关。

## NVMe Virtual Controller：直通物理设备不可用

### 为什么需要 NVMe

前面的结果显示 macOS Virtio 驱动有 ~40K IOPS 天花板。Virtualization.framework 还提供了 `VZNVMExpressControllerDeviceConfiguration`，NVMe 协议比 Virtio 开销更低，理论上有可能突破这个限制。

### 实际测试

没有实现目的 （macOS 15.5 和 26.3.1 上结果一样）：

| Controller               | Attachment                    | 目标            | 结果                 |
| ------------------------ | ----------------------------- | ------------- | ------------------ |
| NVMe | `VZDiskBlockDeviceAttachment` | /dev/rdisk0s5 | ❌ vm.start() 崩溃    |
| NVMe | `VZDiskImageAttachment`       | /dev/rdisk0s5 | ❌ 有控制器，无 namespace |
| NVMe | `VZDiskImageAttachment`       | /dev/disk0s5  | ❌ 有控制器，无 namespace |
| NVMe | `VZDiskImageAttachment`       | 普通文件          | ✅ 正常工作             |

> macOS 中 `/dev/disk0s5` 是 buffered 块设备（经过系统缓存），`/dev/rdisk0s5` 是 raw 字符设备（绕过缓存直接访问）。两者指向同一物理分区，只是 I/O 路径不同。性能测试一般用 `rdisk` 避免缓存干扰。

`VZNVMExpressControllerDeviceConfiguration` + `VZDiskBlockDeviceStorageDeviceAttachment`（物理设备直通该用的方式）在 `vm.start()` 时一律报 "Internal Virtualization error"。试了 synchronizationMode none 和 full，试了 readOnly，都失败了。API 层面 config validation 能通过，但实际启动时崩溃。

把设备路径传给 `VZDiskImageStorageDeviceAttachment` 可以绕过崩溃，guest 内核能识别到 NVMe 控制器（`/dev/nvme0`），但控制器下没有注册任何 namespace。NVMe 协议中 namespace 是实际的逻辑磁盘，对应 `/dev/nvme0n1`。没有 namespace 意味着控制器是空的，相当于装了 NVMe 适配卡但没接硬盘，guest 中没有可用的块设备。

只有后端是普通文件的时候 NVMe 才正常工作。但这不是直通，性能甚至不如 Virtio。而且它不支持 macOS 作为 guest OS，只能对 Linux 使用。

### 那这个 API 是干什么的

`VZNVMExpressControllerDeviceConfiguration` 是 macOS 14 Sonoma（WWDC 2023）引入的。Apple 说的理由是，有些 Linux 内核没有 Virtio 驱动但有 NVMe 驱动，得提供一个能用的控制器。

实际上社区更多是用它来绕过另一个问题：Virtio 块设备在默认缓存模式下会导致 Linux guest 的 ext4 文件系统损坏。UTM（[PR #5919](https://github.com/utmapp/UTM/pull/5919)）和 Lima（[#1957](https://github.com/lima-vm/lima/issues/1957)） 等项目都遇到了。NVMe 控制器是一种 workaround。并且 UTM 测试发现 NVMe 比 Virtio **慢 15-20%**，且并非在所有内核上都能解决损坏问题。

所以这个 API 设计目标是兼容性（让更多 Linux 内核能启动）。它只能配合文件镜像使用，无法用于物理设备直通。
