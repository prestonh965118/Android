---
layout: default
title: "Android Linux 内核结构与子系统机制分析报告"
description: "GKI 内核镜像形态、模块与子系统、设备节点供给、安全控制落点与能力边界的机制分析"
lang: zh-CN
permalink: /Android-Linux-Kernel-Structure-and-Subsystems/
---
# Android Linux 内核结构与子系统机制分析报告

## The Android Linux Kernel: Build Forms, Subsystems and Where Security Controls Live

- 版本：v1.0（2026-10-23）
- 适用对象：ROM/OEM 平台工程、内核与驱动维护、移动安全评估、设备完整性审计、漏洞管理负责人
- 视角声明：本文写的是**内核的结构与机制**——这块代码从哪来、被打进哪个镜像、由谁加载、暴露成哪个节点、能力被哪一层限制。**缺陷类型学、CVE 案例、SPL 指标与提权取证不在本文**，那些内容在姊妹篇里，本文只做交叉引用而不重复：
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（攻击面分级、缺陷分类、公开案例、缓解矩阵、补丁治理指标、提权取证）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（六段分层、进程与 IPC、HAL/Treble、存储与更新架构）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（Framework 层的身份/权限缺陷）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（binder 驱动/协议词表、binderfs 节点供给、SELinux binder 类与 service_manager 判定）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（ACK 分支命名与上游优先、GKI 的二进制交付与 KMI/hermetic manifest、构建产物如何被签名与装配；本文讲内核的结构与落点，该篇讲它的构建与交付口径）
- 范围声明：全文仅描述公开文档化的机制与其设计意图，不提供任何漏洞利用代码、偏移量、绕过步骤或可操作的攻击流程。

---

## 一、摘要

1. **Android 的"内核"不是一个二进制，而是三种角色的叠加**：Android Common Kernel（ACK，LTS + Android 补丁的下游分支）、GKI（Google 构建并签名的通用内核二进制 + 稳定 KMI）、厂商内核/模块（SoC 相关代码，以 `.ko` 形式挂在 vendor 侧）。审计时必须先问"这段代码属于哪一层"，因为它决定谁发补丁、多久发一次。
2. **装载形态已被工程化固定**：Android 12 起 `boot` 镜像是"通用内核 + generic ramdisk"，SoC 相关进入 `vendor_boot`，设备树叠加进 `dtbo`；boot image header v4 提供 `boot_signature` 可用于内核与 ramdisk 的完整性检查，AVB 描述符负责跨阶段校验。
3. **KMI 是内核侧的 Treble**：GKI 用符号列表 + `CONFIG_MODVERSIONS` 固定模块接口，厂商代码必须移出核心内核；GKI 模块由 Google 的构建期密钥签名、且只与构建它的那个 GKI 内核兼容；未签名模块被允许加载的前提是只使用白名单符号，模块文件必须位于只读分区（如 `/vendor/lib/modules`）。
4. **分支体系与生命周期是可计算的**：ACK 分支名形如 `android14-6.1`、`android16-6.12`，KMI 由"内核版本 + Android 平台版本"唯一确定；自 kernel 6.6 起 ACK 采用 4 年支持策略；Android 15（2024）起每个内核版本只新增一个 GKI。**设备出厂内核与当前受支持分支的差值，就是"内核通道是否还在收补丁"的直接度量**。
5. **Android 与主线 Linux 的差距在系统性收敛**：`ION` 在 Android 12 / GKI 2.0 被 DMA-BUF heaps 取代（每个 heap 是独立字符设备，权限模型更清晰）；`ashmem` 自 Linux v5.18（2022）从 staging 移除、Android 侧持续迁向 `memfd`；内核内 `lowmemorykiller` 早已移除，改由用户态 `lmkd` 基于 PSI/`vmpressure` 决策。**这些迁移的共同收益是把"特权逻辑搬出内核、把接口搬进可审计的字符设备"**。
6. **安全控制在内核里有明确落点，而不是抽象概念**：LSM/SELinux（策略在 init 阶段由平台 + vendor 映射编译载入）、seccomp-bpf（Android 8.0 起应用于所有应用进程）、UID/capabilities/`no_new_privs`、cgroup（内存/冻结/优先级）、VFS 层的 FBE 加密策略与块层的 `dm-verity`/`fs-verity`/`blk-crypto`。评估一台设备时，"哪个落点缺失"比"有没有漏洞"更能描述真实风险。
7. **供应商驱动是内核侧真正的长尾风险**：GPU/NPU/modem/Wi-Fi 驱动在 vendor 分区、用私有 ioctl ABI、不进 AOSP 代码树、按厂商节奏更新。GKI 把"核心内核"变短，因此"厂商可出错的地方"更集中也更容易被度量（模块列表、版本、加载路径）。
8. **可观测性决定审计上限**：`pstore`/`ramoops`（崩溃前写入 RAM）、`dmesg`/`logd`、SELinux `avc: denied`、`ftrace`/Perfetto、已加载模块列表、`/proc/cmdline` 构成了"事后能看到什么"的边界。第十三章给出未 root 条件下可采的内核结构指纹与断言。

---

## 二、内核从哪里来：ACK、GKI 与厂商内核

### 2.1 名词表

| 术语 | 全称/含义 | 结构作用 |
| --- | --- | --- |
| ACK | Android Common Kernel，LTS 内核 + Android 补丁的下游分支（如 `android12-5.10`、`android14-6.1`、`android16-6.12`） | 补丁的**来源通道**；厂商应从此合并 |
| GKI | Generic Kernel Image，Google 统一构建/签名的通用内核二进制 | 把"核心内核"变成**共享、可独立更新的产物** |
| KMI | Kernel Module Interface，由符号列表 + 结构体布局定义的稳定 ABI | 决定"厂商模块能否跨内核小版本复用" |
| DLKM | Dynamically Loadable Kernel Module，厂商可加载模块 | SoC/设备代码的**合法存放位置** |
| VINTF / LLNDK | 接口兼容性矩阵 / 底层原生库接口 | 用户态的对偶机制，帮助理解 KMI 的定位 |
| `boot` / `vendor_boot` / `dtbo` | 通用内核+generic ramdisk / 厂商 ramdisk+DLKM / 设备树叠加 | 三个**独立签名与独立更新**的镜像 |

### 2.2 三种内核角色

```
Google:  LTS upstream ──┐
                        ├─► ACK (androidNN-X.Y) ──► GKI 构建（签名、KMI 固定）
AOSP:    Android 补丁 ──┘                                   │
                                                            ▼
OEM/ODM: SoC 驱动 ──► DLKM（.ko，/vendor/lib/modules）+ vendor_boot ──► 设备 boot 链
```

- **谁发补丁**：核心内核由 Google 的 GKI release 构建提供（`gki-release-builds` 页面按月列出产物）；厂商模块由 OEM/SoC 供应商发；设备树与板级配置由 OEM 发。
- **结构含义**：同一台设备上"内核核心版本"与"厂商模块集合"是两个可以分别落后的组件。任何只用 `uname -r` 描述设备内核的做法都会掩盖这一点。

### 2.3 分支与生命周期（可计算的指标）

| 事实 | 来源表述 | 审计用法 |
| --- | --- | --- |
| ACK 分支命名形如 `android14-6.1`；KMI 由"内核版本 + Android 平台版本"唯一确定 | Android common kernels 文档 | 用 `uname -r` + `ro.build.version.release` 反查是否落在受支持分支 |
| 自 kernel 6.6 起 ACK 采用 4 年支持策略 | 同上 | "出厂内核已超出支持期"应作为独立风险项 |
| Android 15（2024）起每个内核版本只新增一个 GKI（不存在 `android15-6.1` 之类并行分支） | 同上 | 减少分支组合爆炸，也意味着"选错内核版本"的代价更大 |
| Android 12 起，出厂内核 5.10+ 的设备必须使用 GKI | GKI 文档 | 未走 GKI 的 5.10+ 设备通常不是 AOSP 合规构建 |

### 2.4 Android 相对主线 Linux 的差异要点

1. **补丁集**：调度（EAS/schedtune 类）、内存回收与 `lmkd` 协作、`staging` 里的 Binder/ION（历史上）、`f2fs`、`blk-crypto` 相关、GKI 配置与符号列表。
2. **移除/替换的方向**：`ION` → DMA-BUF heaps；`ashmem` → `memfd`；内核内 `lowmemorykiller` → 用户态 `lmkd`。共同动机是**缩小内核里的"特权策略代码"**。
3. **上游化压力**：Google 的公共分支要求厂商"先改上游、再回移"，这也是过去五年 Android 内核攻击面收敛慢但方向明确的原因。

---

## 三、镜像形态：内核如何被打包、校验与加载

### 3.1 分区与镜像

| 镜像 | 内容 | 引入/约束 |
| --- | --- | --- |
| `boot` | GKI 内核 + generic ramdisk（first-stage init） | Android 12 起 `boot` 即"通用内核镜像"的容器 |
| `vendor_boot` | 厂商 ramdisk、DLKM、厂商 cmdline 片段 | ARM64 出厂 Android 11 / 5.4 的设备即被要求支持 `vendor_boot` 与新 boot 格式 |
| `dtbo` | 设备树叠加 | 独立分区、独立 AVB 描述符 |
| `vbmeta*` | 校验元数据（含链式分区公钥） | 由前一阶段签名链验证 |

### 3.2 校验链的三个接触点

1. **boot image header**：v4 头提供 `boot_signature`，可用于检查内核与 ramdisk 的完整性（Android 12）。
2. **AVB 描述符**：`boot`/`vendor_boot`/`dtbo` 以 hash/hashtree 描述符进入可信链，最终状态（green/yellow/orange/red）被写进密钥证明的 `RootOfTrust`（详见架构报告第 3.2 节）。
3. **cmdline 交接**：引导加载程序把 `androidboot.*` 参数（含 `verifiedbootstate`、`slot_suffix`、`bootimage` 相关）交给内核，first-stage init 读取后决定挂载与 verity 模式——**内核自身不判定可信性，判定在链的上游**。

### 3.3 模块装载的结构规则

| 规则 | 表述 | 后果 |
| --- | --- | --- |
| GKI 模块由 Google 构建期密钥签名 | "GKI modules are signed by Google using the kernel build time key pair"，且只与构建它们的 GKI 内核兼容 | 混用不同构建的模块会被拒 |
| 未签名模块仍可加载，但**只能使用白名单符号** | loadable kernel modules 文档 | 白名单符号集是"厂商能碰内核什么"的实际边界 |
| 模块必须放在只读分区（如 `/vendor/lib/modules`） | 同上 | 可写分区里放 `.ko` = 配置异常，属高价值检测信号 |
| ABI 通过 `CONFIG_MODVERSIONS` + KMI 符号列表固定 | 同上 | 内核小版本升级不必重编所有模块，代价是符号列表本身要维护 |

**审计要点**：模块清单是"第三方代码进内核"的唯一常规入口。采集 `lsmod`/`/sys/module/*` 与 `/vendor/lib/modules/*.ko` 的存在性比对（第十三章），能识别出"运行了镜像里没有的模块"这类改造痕迹（需要相应读取权限，生产设备多不可读，按数据质量记录）。

### 3.4 模块的分区化：`vendor_dlkm` / `odm_dlkm` / `system_dlkm`

- Android 11 引入 **`vendor_dlkm` 与 `odm_dlkm`** 分区，把厂商/ODM 内核模块从 `vendor` 镜像里独立出来；它们作为动态分区位于 `super` 内，挂载后模块路径仍是 `/vendor/lib/modules` 或 `/odm/lib/modules`（不是分区路径本身）。
- `system_dlkm` 承载系统侧模块；`vendor_dlkm` 里依赖 `system_dlkm` 的模块要通过 `modules.dep` 正确建立引用关系，装载顺序才可靠。
- 结构收益：模块集合可以**独立于整个 vendor 分区**更新，这是 GKI 合规下"只补模块不动平台"能落地的前提；代价是"模块集合"从此成为必须单独版本化的组件。
- 审计含义：把 `vendor_dlkm`/`odm_dlkm` 当作与 `vendor` 同级的独立条目——它们的构建日期与签名链可能落后于平台分区。

### 3.5 构建产物与配置约束

| 事实 | 表述 | 用途 |
| --- | --- | --- |
| GKI release 构建按月发布，产物含正式内核与 debug 内核 | `gki-release-builds` 及各分支页（`android12-5.10`、`android15-6.6`、`android16-6.12`） | 核对设备内核是否来自官方构建系列，是判定"是否被替换内核"的比对基准 |
| 内核构建走 Bazel（Kleaf）等新构建方式 | Build kernels 文档 | 配置变更可被构建系统记录，"随手改 defconfig"的空间被压缩 |
| 需要启用隐藏配置时，须通过 `init/Kconfig.gki` 的 `select` 与 `CONFIG_GKI_HACKS_TO_FIX` 机制 | Develop kernel code for GKI 文档 | 厂商想动 GKI 配置必须留下可见痕迹——**"配置违规"从隐性变成显性** |
| 提供可 FIPS 140-3 认证的 GKI 加密模块 | FIPS 140-3 certifiable GKI crypto module 文档 | 合规场景（政府/金融）可把内核加密路径纳入认证范围 |


---

## 四、内存管理：分配、共享与回收

### 4.1 三层分配视图

```
用户态：malloc/scudo（Android 11 起原生默认分配器） ─┐
内核态：buddy（页）+ slab/slub（对象）              ├─► 物理内存
设备态：DMA-BUF heaps（system/cma/…）→ dma-buf 句柄 ─┘   （跨进程/跨设备共享）
```

区分这三层很重要：**堆损坏类缺陷的缓解多数在用户态分配器（scudo/GWP-ASan），而共享缓冲区的管理错误在内核侧**，两者的修复通道完全不同。

### 4.2 DMA-BUF heaps 取代 ION

- Android 12 / GKI 2.0 用 **DMA-BUF heaps** 替换 `ION`。官方给出的动机之一是**安全性**：每个 heap 是独立的字符设备（`/dev/dma_heap/<heap>`），因而权限、SELinux 类型、访问控制可以按设备节点分别表达，而不必集中在一个 `/dev/ion` 里靠 ioctl 参数区分。
- 审计含义：看到 `/dev/ion` 说明这是旧内核形态；看到 `dma_heap` 节点则应逐个核对属主/组/SELinux 标签，因为**图形/相机/NPU 缓冲区的跨进程共享能力就落在这些节点上**。

### 4.3 共享内存：ashmem → memfd

- `ashmem` 自 Linux v5.18（2022）从 staging 移除，方向是 `memfd`（匿名、可 fd 传递、可 seal 的文件内存）；Android 侧因 vendor 与应用兼容性做**渐进迁移**，过渡期内两条路径并存。
- 机制意义：`memfd` 走普通 VFS 与文件描述符语义，因此继承 DAC/SELinux/`noexec` 等约束；`ashmem` 是专用字符设备 + ioctl，策略表达面更窄。迁移等于把"私有接口"换成"内核通用机制"。

### 4.4 低内存与回收

| 组件 | 事实 | 结构含义 |
| --- | --- | --- |
| `lmkd` | Android 10+ 使用用户态 `lmkd`，基于 **PSI** 或 `vmpressure` 决策；内核内 `lowmemorykiller` 已移除（文档标注 4.12+） | 终止策略是**用户态进程的行为**：可被观测、可被配置错误影响 |
| zram + `mmd` | `mmd` 做 ZRAM writeback：把空闲页整体换出到 `/data` 上的 backing 文件 | 交换内容可落到闪存 → 与加密策略（FBE 是否覆盖 swap 文件）相关 |
| Memory Limiter | 通过 cgroup `memory.swap.max` 等限制单进程可用交换与内存 | 进程级配额在内核 cgroup 层实现，用户态写策略 |

### 4.5 内核侧执行/内存加固的定位

编译期与硬件强制缓解（PAN/PXN、Shadow Call Stack、CFI/kCFI、PAC/BTI、MTE、StackProtector、init_on_alloc、KFENCE 等）的**清单与攻防含义在姊妹篇第七章**；本文只指出它们的落点属性：

- 它们是 **GKI 构建配置的一部分**，因此"设备是否具备"取决于 GKI 版本 + SoC 能力，而不是应用或 ROM 开关。
- 它们**不改变可达面**（哪些 syscall/节点可访问），只改变"缺陷被触发后的可利用性"。做风险模型时不要把二者混为同一层。

---

## 五、进程、调度与资源控制

### 5.1 进程创建的内核路径

Android 应用进程由 Zygote `fork`（`clone`）后特化：改 UID/GID、capabilities、seccomp 过滤器、cgroup 归属、SELinux 上下文，然后进入 `ActivityThread.main()`。**关键点：能力的授予发生在从特权父进程分裂出去的那一刻**，此后进程无法自行扩大（`no_new_privs`、SELinux 域固定）。

### 5.2 cgroup 层次

| 控制器 | 用途 | 安全/可用性相关性 |
| --- | --- | --- |
| `cpuset` / `cpu` / `stune` | 绑核、份额、schedtune 参数 | 前台/后台差异导致"同一 APK 在不同档位行为不同" |
| `memory` | 内存高/低水位、`memory.swap.max` | Memory Limiter 的执行处 |
| `freezer` / 近版本基于 PSI 的 cached apps freezing | 冻结后台进程 | 冻结期间设备侧无行为上报，检测时间窗需容忍 |
| `pid` | 进程数上限 | 资源耗尽类缺陷的兜底 |

### 5.3 命名空间

Android 并不给每个应用一套完整 namespace；隔离主力是 **UID + SELinux**。命名空间主要用于挂载视图隔离（`/mnt`、`/apex`、`/data/user` 的私有挂载视图）与容器/调试类场景。因此**"用 namespace 加固"在 Android 上不是默认答案**，评估时不要把它当作缺失项。

### 5.4 调度特性对安全评估的间接影响

EAS/schedtune/WALT 一类的 Android 调度补丁影响时序与功耗窗口，其安全意义主要是：竞态类缺陷的触发窗口随负载变化而变化（复现困难），以及"后台限频"使持久化行为（恶意保活）在功耗统计上可被识别——后者是一条可用的检测线索。

---

## 六、文件系统与块层：只读、加密与增量

### 6.1 只读完整性：两种 verity 的分工

| 机制 | 粒度 | 典型用途 | 关键差异 |
| --- | --- | --- | --- |
| `dm-verity` / inline-verity（含 FEC） | 块设备 | `system`/`vendor`/`product` 等 AVB 保护的分区 | 由 AVB 描述符建立根哈希；`ro.boot.veritymode` 决定拒绝策略 |
| `fs-verity` | 单个文件 | APK 完整性（与 APK 签名方案 v4 的 Merkle 树结构一致）、设备端签名架构 | 文件系统按页校验，读到时验证；不受整盘挂载模式影响 |

含义：**"分区只读"和"文件可信"是两回事**。前者可被 `veritymode=logging`/解锁降级，后者依赖签名与安装流程正确性。两者都应出现在检测项里。

### 6.2 IncFS：把"未完成的 APK"放进内核

`Incremental File System`（Android 11 引入）允许系统接收**流式 APK**（例如通过 ADB 安装）：文件按页到位、缺失页可读为错误状态，完成后才变得可执行。安全上它的价值是把"安装中"的半可信文件限制在一个受控文件系统里，而不是放在 `/data` 普通目录。

### 6.3 FBE 的内核落点

```
用户态：vold/system_server 生成/解锁密钥  ──► 内核 keyring
内核态：VFS 文件加密策略（per-file/目录默认策略）+ 内容/文件名算法
块层：  inline crypto（blk-crypto，硬件密钥槽）/ dm-default-key（元数据加密）
```

- CE（凭据加密）与 DE（设备加密）的区分在**策略层**，而不是路径字符串：路径只是策略的体现。审计时应看"是否设置了加密策略"，而非"文件在哪个目录"。
- 元数据加密保护目录树、文件名等 FBE 不覆盖的部分；两者由不同层实现，缺一个就是不同的信息泄露面。
- **信任边界**：密钥由用户态服务生成/递送、由内核使用，最终保护根在硬件后端（KeyMint/TEE/Weaver）。这条链上任何一环用软件回退，设备的"静态数据保护"结论就要降级。

### 6.4 挂载与节点权限

常见形态：`/data` `noatime` 等选项 + `nosuid`/`nodev`，`/dev` 由 devtmpfs + `ueventd` 权限表赋权，`/proc`、`/sys` 大量节点受 SELinux 与属主限制。**"节点权限表"（`ueventd.rc`）是内核暴露面与用户态权限模型之间的官方映射**，值得在设备基线里逐条比对；异常新增的可读写节点是改造信号。

### 6.5 受支持的文件系统与强制要求

Android 内核侧支持 `ext4`、`f2fs`、`EROFS`、`exfat`、`fuse`、`incfs`、`vfat`，并且**这些文件系统都被要求具备 `fscrypt` 与 `fs-verity` 支持**——即"加密与按文件验证"不是可选附加项，而是文件系统能否进入 Android 的门槛条件之一。

| 文件系统 | 常见用途 | 结构特征 |
| --- | --- | --- |
| `ext4` | 传统 `/data`、部分只读分区 | 通用性强；写放大与碎片由工具链管理 |
| `f2fs` | `/data`（闪存友好） | 面向闪存的日志结构；checkpoint 机制与 OTA/回滚配合 |
| `EROFS` | 只读镜像（紧凑、低内存开销） | 只读形态与 verity 校验天然契合 |
| `incfs` | 流式 APK（见 6.2） | 专门承载"未完整文件"的内核态缓冲 |
| `fuse` | 外部存储/文档提供器等用户态文件系统 | **策略在用户态守护进程里**，其正确性等价于一个服务的正确性 |

审计含义：`fuse` 一类把文件系统逻辑放到用户态的形态，其信任边界不再由内核 VFS 单独保证，评估时要连带看承载它的进程身份与权限（通常是 `mediaprovider`/`sdcardfs` 演进后的用户态实现）。


---

## 七、Binder、设备节点与 socket：内核里的 IPC 底座

### 7.1 驱动侧数据结构

Binder 在内核里维护 **process（打开 `/dev/binder` 的上下文）— thread（每线程一个小缓冲区）— node（服务对象）— ref（跨进程引用）** 四类对象，事务在内核中完成拷贝与身份注入：

- **调用方 UID/PID 由内核写入事务头**，这是"身份可信"的物理来源；用户态无法伪造（这也是 Framework 层鉴权必须使用 `getCallingUid()` 的前提）。
- **引用计数决定对象生命周期**：strong/weak ref 与死亡通知由驱动维护。
- **每个进程的事务缓冲区大小有上限**，异步空间独立配额；超出即失败——这是设计约束，也是稳定性问题的来源（详见架构报告第六章）。

### 7.2 三个 binder 设备域

`/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder` 是**同一驱动的三个上下文**，服务管理器分别对应用户态进程；域的选择由 Treble 边界与 SELinux 类型决定（`binder` 类的域间通信权限，逐向放行）。`binderfs` 提供把 binder 上下文以文件形式挂载到目录的能力（容器/调试场景），生产设备上的节点位置应以 `ls -Z /dev/*binder*` 实测为准。

### 7.3 `/dev/socket/*`：内核之外的早期 IPC

`property_service`、`logd`、`tombstoned` 等通过 `init` 创建的 UNIX socket 提供服务；这些 socket 的**属主/组与 SELinux 类型**决定了谁能写日志、谁能读属性、谁能接收崩溃信息。对安全评估而言，`ueventd.rc` + `*.rc` 里的 socket 声明合起来就是"内核外的一层攻击面清单"。

### 7.4 内核日志与用户态日志的接缝

`printk` → 环形缓冲区 → `/dev/kmsg` / `/proc/kmsg`（受 `dmesg_restrict` 等限制）→ `logd` 落入 `main/system/crash` 缓冲；崩溃前内容另由 `pstore`/`ramoops` 写入持久 RAM 区域。这决定了**"内核里发生过什么"的可观测深度**，也是第十三章采集设计的依据。

### 7.5 调试与遥测通道：Android 12 之后 debugfs 不再是生产接口

- **DebugFS 只在 `userdebug` 构建上挂载，生产（`user`）构建不挂载**；"One-time DebugFS files" 指开机初始化阶段挂载、使用后立即卸载的场景。
- 官方指引是：**发布镜像里的内核遥测应使用 eBPF / Kernel Telemetry 通道**，而不是 debugfs（`docs/core/architecture/kernel/bpf`、`Collect kernel telemetry with eBPF`）。
- 审计含义（两条都要记住）：
  1. 在生产设备上看不到 `/sys/kernel/debug/*` **是正常状态**，不能当作"日志被关闭"的证据；
  2. eBPF 程序运行在内核里、由平台装载，属于"可更新的准代码"，其装载与生命周期应与模块同等纳入变更观察（`ro.build.type` 与可用调试通道的差异，也直接决定取证深度：`user` 与 `userdebug` 的采集能力不是一套）。

---

## 八、安全子系统在内核里的落点

| 控制 | 内核中的实现 | Android 侧配置点 | 失效时的直接表现 |
| --- | --- | --- | --- |
| MAC（SELinux） | LSM 钩子 + 策略数据库（类型/属性/MLS 类别） | `init` 载入平台策略与 vendor 映射（CIL 编译）；`getenforce`；`neverallow` 在构建期约束 | 权限判定退化到 DAC；`avc: denied` 消失并不代表安全，可能是策略被删 |
| syscall 过滤（seccomp-bpf） | `SECCOMP_SET_MODE_FILTER`，进程级过滤器 | Zygote 为应用进程安装（Android 8.0 起所有应用）；原生服务可用 `minijail` | 过滤器缺失/为 allow-all → 内核可达面等于主线 Linux 全部 |
| DAC（UID/GID/capabilities） | 凭证与文件属主检查 | 每应用唯一 UID；`init` 按服务声明授予 cap | 能力继承错误 = 长期提权驻留 |
| 执行限制 | `no_new_privs`、`noexec` 挂载、W^X | 应用进程默认 | 可执行数据页出现 → 内存损坏类缺陷后果放大 |
| 完整性 | AVB（上游）、`dm-verity`、`fs-verity`、模块签名 | `vbmeta` 描述符、`/vendor/lib/modules` 只读 | 篡改被拒 → 若 verity 为 logging/disabled 则不再拒 |
| 数据保护 | VFS 加密策略、`blk-crypto`、`dm-default-key`、内核 keyring | FBE CE/DE、元数据加密 | 凭据未解锁即可读文件 = CE 面被拉平到 DE |
| 资源隔离 | cgroup、`pid` 限制、OOM 决策接口 | `lmkd`/Memory Limiter/冻结 | 单进程可耗尽全局资源（可用性） |

一个实用结论：**Android 的安全控制是"内核机制 + 用户态策略"的配对**。只有机制没有策略（SELinux 存在但域被放宽、seccomp 存在但白名单过宽）时，`getenforce` 一类的单一字段会给出过于乐观的读数——检测脚本应尽可能采"配对两侧"。

---

## 九、驱动与 HAL：内核与硬件的接缝

### 9.1 结构要点

1. **形态**：GPU/NPU/modem/Wi-Fi/蓝牙等驱动以 `.ko` 或 built-in 形式存在于 vendor 侧，暴露为字符设备节点或网络接口，用户态经 `hal_*` 进程访问。
2. **可审计性最低**：私有 ioctl ABI、无公开源码、厂商自定义数据结构——这是内核评估里真正的长尾。可行的替代方法是**结构比对**（节点是否新增、模块是否在镜像清单内、`ro.vendor.build.date` 是否过旧）而不是代码审查。
3. **modem 与 secure world 不在 AVB 链内**：基带、TEE/Trustzone（Trusty 等安全 OS）、KeyMint/门限认证实现是**独立更新通道**，其完整性不受 Android 镜像校验覆盖；企业采购与风险评估应把它们列为单独条目。
4. **GKI 带来的结构性收益**：核心内核变短 + SoC 代码进模块 → 补丁可以只更新模块而不动 GKI（或反之），使"内核落后"从不可分指标变成两个可分指标（GKI 版本 / vendor 模块版本）。

### 9.2 安全世界（secure world）与内核的关系

- **Trusty** 是开源的安全 OS，提供 Android 的 TEE；它**运行在与 Android 相同的处理器上**，通过硬件特权分级（ARM TrustZone 一类机制）与正常世界隔离，Android 侧经内核驱动与之通信。
- 结构定位：内核在这里只是**通道维护者**——真正的密钥材料、门限认证（gatekeeper）、生物识别匹配逻辑位于安全世界或独立安全芯片内。因此：
  1. 内核被攻破**不自动等于**硬件密钥被取出（这也是攻防报告里"提权层级"区分的原因）；
  2. 但内核可以破坏**请求的完整性**（伪造调用序列、干扰可用性）；
  3. 安全世界的代码与固件**不在 AVB/OTA 覆盖范围内**，其更新通道与补丁责任属于芯片/整机厂商，采购评估需单列。
- 可验证性出口：密钥证明（Key Attestation）与 RKP 远程配置把"是否硬件后端、实现版本"等信息以签名形式外化，服务端可据此判定；属性字段只能作运维参考（见架构报告第十二章）。

---

## 十、其他基础子系统（结构视角简表）

| 子系统 | Android 侧形态 | 值得记录的结构性事实 |
| --- | --- | --- |
| 时间 | `boottime`/`monotonic`/RTC；唤醒由驱动与电源管理决定 | 属性 `ro.*` 与时间戳组合可用于判断"是否曾长期关机"，但对取证只能算弱证据 |
| 随机数 | `getrandom(2)` + 硬件熵源（RNG 驱动多为 vendor 代码） | 熵源实现属于厂商私有面，早期启动阶段的 RNG 是历史上出过问题的位置 |
| 网络 | netlink + `netd`/eBPF 做策略与统计（`docs/core/architecture/kernel/bpf`） | eBPF 程序在内核里运行，是"平台自己写的可扩展逻辑"，需要与模块同等对待 |
| 电源/挂起 | suspend 状态由内核与固件共同完成；锁屏与密钥可用性联动 | 挂起期间内存内容仍在（物理攻击面），但用户未解锁时 CE 密钥不应在内存中 |
| 崩溃处理 | `panic` → `pstore`/`ramoops`；用户态 `tombstoned`/`dropbox` | 崩溃历史是设备稳定性与安全改造检测的双重线索 |

---

## 十一、版本演进时间线（结构变化 → 安全含义）

| 内核/版本节点 | 结构变化 | 安全含义 |
| --- | --- | --- |
| 3.18 / 4.x 时代 | 单一厂商内核、`ION`、`/dev/ion`、`ashmem`、内核内 LMK、策略集中在厂商 | 内核不可独立打补丁，审计只能靠设备行为 |
| Android 8 / Treble（4.9 起） | HAL 与 vendor 分区分离、SELinux 双策略、binder 域拆分 | 边界可静态校验，供应商代码首次被"分区化" |
| Android 10/11（5.4） | `vendor_boot` + 新 boot 格式（ARM64 出厂 5.4 需支持）、`lmkd` 基于 PSI、IncFS（Android 11） | 早期启动阶段与共享内存通道开始标准化 |
| Android 12 / GKI 2.0（5.10 强制） | 通用内核 + DLKM、KMI 符号列表、DMA-BUF heaps 取代 ION、boot header v4 `boot_signature` | **内核核心可被 Google 直接更新**；共享缓冲区获得按节点表达权限的能力 |
| 6.1 / 6.6 / 6.12（Android 14–16 分支） | ACK 4 年支持策略（6.6 起）、每内核版本单一 GKI（Android 15 起） | 支持窗口明确可计算；分支收敛降低"选错内核"的空间 |
| `ashmem` 从主线移除（v5.18，2022） | 迁移到 `memfd`（Android 渐进推进） | 私有设备接口 → 通用 VFS 语义，策略表达面统一 |

---

## 十二、结构缺陷分布矩阵（与攻防报告的分界线）

> 类型学与案例见内核攻防报告第四、五章；这里只回答"结构上哪一层最容易承载哪类问题"，便于把发现落到正确的所有者。

| 结构位置 | 承载的问题类别 | 谁负责修 | 可观测信号 |
| --- | --- | --- | --- |
| ACK/GKI 通用代码（网络、VFS、`binder`、`ion/dma-buf` 核心） | 通用内存损坏、逻辑鉴权缺失 | Google → OEM 集成 | GKI 版本 / `uname -r` 与受支持分支差 |
| vendor DLKM 与私有 ioctl | 驱动鉴权缺失、越界拷贝、竞态 | SoC/OEM | 模块清单、vendor 构建日期 |
| 内核策略层（SELinux 策略、seccomp 过滤器） | 可达面过宽、域过权 | OEM（平台）+ 平台合规检查 | `getenforce`、域集合、过滤器加载记录 |
| 完整性链（AVB/verity/模块签名） | 篡改与降级 | OEM/构建流程 | `verifiedbootstate`、`veritymode`、模块加载路径 |
| 加密实现（FBE/元数据/keyring） | 密钥可用性错误、策略缺失 | 平台 + 硬件后端 | 加密策略、DE/CE 目录内容可读时序 |

---

## 十三、检测与采集：内核结构指纹

> 全部命令在**未 root** 的 user 版设备上运行；标注 `# 需特权` 的项在生产设备上预期读不到，按数据质量记录而不是安全结论。

### 13.1 采集脚本

```bash
#!/usr/bin/env bash
# kstruct-fingerprint.sh —— 采集设备内核结构事实（只读）
set -u
dev=${1:-}
[ -n "$dev" ] || { echo "usage: $0 <serial>"; exit 2; }
sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

echo "== identity =="
echo "release=$(p ro.build.version.release) sdk=$(p ro.build.version.sdk)"
echo "spl=$(p ro.build.version.security_patch) type=$(p ro.build.type)"
echo "fp=$(p ro.build.fingerprint)"

echo "== kernel =="
echo "uname=$(sh 'uname -r')"
echo "version=$(sh 'cat /proc/version')"
echo "cmdline=$(sh 'cat /proc/cmdline')"
echo "gki=$(p ro.kernel.android.gki) gki_ver=$(p ro.gki.version)"   # 属性名随 OEM 变化，缺失不等于未用 GKI

echo "== memory =="
echo "heaps=$(sh 'ls /dev/dma_heap 2>/dev/null | tr "\n" " "')"
echo "ion_present=$(sh 'ls -l /dev/ion 2>/dev/null')"
echo "zram=$(sh 'ls /dev/block/zram* 2>/dev/null | tr "\n" " "')"
echo "swap=$(sh 'grep -c . /proc/swaps 2>/dev/null')"
echo "lmkd=$(sh 'getprop ctl.info lmkd')"

echo "== ipc =="
echo "binder_nodes=$(sh 'ls /dev/binder /dev/hwbinder /dev/vndbinder 2>/dev/null | tr "\n" " "')"
echo "binderfs=$(sh 'ls /dev/binderfs 2>/dev/null | tr "\n" " "')"   # 需特权/部分设备
echo "sockets=$(sh 'ls /dev/socket 2>/dev/null | tr "\n" " "')"

echo "== modules / integrity =="
echo "mods=$(sh 'ls /sys/module 2>/dev/null | tr "\n" " "')"   # 需特权，未授权通常为空
echo "ko_files=$(sh 'ls /vendor/lib/modules 2>/dev/null | wc -l')"            # 需特权
echo "selinux=$(sh 'getenforce')"
echo "verity=$(p ro.boot.veritymode) vbstate=$(p ro.boot.verifiedbootstate)"
echo "locked=$(p ro.boot.flash.locked) vbmeta_alg=$(p ro.boot.vbmeta.hash_alg)"

echo "== observability =="
echo "pstore=$(sh 'ls /sys/fs/pstore 2>/dev/null | tr "\n" " "')"   # 需特权
echo "kptr=$(sh 'cat /proc/sys/kernel/kptr_restrict 2>/dev/null')"
echo "dmesg_restrict=$(sh 'cat /proc/sys/kernel/dmesg_restrict 2>/dev/null')"
echo "perf=$(sh 'cat /proc/sys/kernel/perf_event_paranoid 2>/dev/null')"
```

### 13.2 断言：什么算结构异常

```bash
#!/usr/bin/env bash
# kstruct-assert.sh —— 对第 13.1 节的采集结果做保守断言
set -u
dev=${1:-?}
sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

k=$(sh 'uname -r')
case "$k" in
  '')      echo "LOW   读不到 uname -r" ;;
  *.*)     ;;
  *)       echo "MED   内核版本字符串异常: $k" ;;
esac

# 与官方支持分支的粗略比对：主.次版本
maj=$(printf '%s' "$k" | cut -d. -f1)
min=$(printf '%s' "$k" | cut -d. -f2 | sed 's/[^0-9].*//')
if [ -n "$maj" ] && [ -n "$min" ]; then
  if [ "$maj" -lt 5 ] || { [ "$maj" -eq 5 ] && [ "$min" -lt 4 ]; }; then
    echo "HIGH  内核 ${k} 早于 GKI/ACK 主流支持形态（5.4 以下），补丁通道存疑"
  fi
fi

rel=$(p ro.build.version.release)
spl=$(p ro.build.version.security_patch)
if [ -n "$spl" ]; then
  age=$(( ( $(date +%s) - $(date -d "$spl" +%s) ) / 86400 ))
  [ "$age" -gt 120 ] && echo "HIGH  release=${rel} 但 SPL 落后 ${age} 天（${spl}）"
fi

[ "$(sh 'getenforce')" = "Permissive" ] && echo "HIGH  SELinux 全局 permissive"
case "$(p ro.boot.veritymode)" in
  logging|disabled|'') echo "HIGH  verity 非 enforcing（=$(p ro.boot.veritymode)）：块设备篡改不再被拒" ;;
esac
[ "$(p ro.boot.flash.locked)" = "0" ] && echo "HIGH  引导加载程序未锁定"
[ -n "$(sh 'ls -l /dev/ion 2>/dev/null')" ] && echo "INFO  仍存在 /dev/ion（旧 ION 形态，共享缓冲区权限表达集中在 ioctl）"

# swap 是否落到 /data 上的 backing 文件：与 FBE 覆盖面相关，需结合厂商说明判断
sw=$(sh 'cat /proc/swaps 2>/dev/null')
case "$sw" in
  *zram*) ;;
  '')     echo "INFO  读不到 /proc/swaps（权限）" ;;
  *)      echo "INFO  存在非 zram 交换，核对是否被 FBE 覆盖" ;;
esac
```

### 13.3 建议入库的字段

| 字段 | 命令 | 用途 |
| --- | --- | --- |
| `uname -r`、`/proc/version` | shell | 判断是否落在受支持的 ACK/GKI 分支形态 |
| `ro.gki.version` / `ro.kernel.android.gki` | getprop | 是否真为 GKI 构建（属性名以设备实际为准） |
| `/proc/cmdline` | shell | `androidboot.*` 与 verity/槽位状态交叉验证 |
| `/dev/dma_heap/*`、`/dev/ion` 存在性 | shell | 共享内存通道形态（新/旧） |
| `/sys/module/*`、`/vendor/lib/modules` 数量 | shell（需特权） | 模块与镜像清单一致性 |
| `getenforce` + `avc: denied` 是否有记录 | shell / logcat | 机制与策略"配对两侧"核对 |
| `/sys/fs/pstore` 内容存在性 | shell（需特权） | 崩溃历史可用性；重启原因分析 |
| `kptr_restrict`、`dmesg_restrict`、`perf_event_paranoid` | shell | 信息泄露面收敛程度（部分设备默认即可读） |

### 13.4 应用自身进程内的内核自检字段

设备外的采集受权限限制，但**应用在自己的进程里可以读到内核给出的真实状态**，适合做成 SDK 上报或合规巡检。`/proc/self/status` 中的相关字段（语义见 proc(5)）：

| 字段 | 含义 | 期望值 | 异常含义 |
| --- | --- | --- | --- |
| `Seccomp` | 0=关闭，1=strict，**2=filter** | 2（Android 8.0 起应用进程装载 seccomp-bpf 过滤器） | 0 → 该进程未进入受过滤状态（或被特殊域启动） |
| `Seccomp_filters` | 已装载的过滤器数量 | ≥1 | 0 与上表 0 同义 |
| `NoNewPrivs` | 禁止通过 exec/setuid 获得新特权 | 1 | 0 → 存在能力继承路径 |
| `CapEff` / `CapBnd` | 有效/可继承 capability 集（十六进制位掩码） | 应用进程应为 `0000000000000000` | 非零 → 该进程被当作系统服务启动，能力异常 |
| `Uid` / `Gid` | 真实/有效/保存/文件系统 ID 四元组 | 与安装时分配的 app UID 一致 | 与包管理器记录不一致 → 身份传递或启动路径异常 |
| `Cpus_allowed` / `Mems_allowed` | 可运行的 CPU/内存节点掩码 | 随进程档位变化（cpuset 控制器） | 只用于稳定性分析，不作安全结论 |

```bash
# 在 adb shell 中读取"shell 自身进程"的同名字段做形态对照（应用侧应读 /proc/self/status）
adb shell 'grep -E "^(Uid|Gid|Seccomp|Seccomp_filters|NoNewPrivs|CapEff|CapBnd)" /proc/self/status'
```

用法与边界：这些字段证明的是**"当前这个进程"的内核侧状态**，不能外推为"整机所有进程如此"；把它与服务端可校验的密钥证明（不同链路）配合使用，才能同时覆盖"运行时被降权/被放行"和"设备被改造"两类异常。

### 13.5 与另两篇报告的采集差异

- 内核攻防报告第八章采集的是**补丁水位**（SPL、公告映射、在野利用匹配）。
- 架构报告第十四章采集的是**设备完整性与更新通道**。
- 本文第十三章采集的是**内核形态**（分支/GKI/节点/模块/加固 sysctl）。
  三者字段不重叠，合成后就是"这台设备的内核层能说什么"的完整答案。

---

## 十四、常见误区

1. **"内核版本号 = 安全性"**：`uname -r` 不告诉你 GKI 构建、模块集合或补丁水位；必须三个字段一起看。
2. **"GKI 意味着 Google 修所有内核漏洞"**：GKI 覆盖核心内核代码；厂商模块、固件（基带/TEE）不在其中。
3. **"模块签名 = 内核只运行官方代码"**：GKI 模块由 Google 密钥签名，但未签名模块在使用白名单符号的前提下仍可加载；签名不是"全量校验"。
4. **"SELinux 开着就安全"**：`getenforce` 只证明机制在；策略是否被放宽要看域与 `avc: denied` 记录是否配对存在。
5. **"seccomp 限制了攻击面"**：它限制 syscall 集合，不限制被允许接口内部的缺陷（历史上多数 Android 内核提权正是走允许的系统调用）。
6. **"`/system` 只读所以内容可信"**：只读是挂载属性，可信来自 AVB + verity enforcing + 锁定状态三者同时成立。
7. **"分区只读等价于内核也校验"**：`fs-verity`/`dm-verity` 覆盖文件与块设备，内核镜像的校验在 boot header/AVB 层，是不同机制。
8. **"删除 `avc: denied` 日志说明没问题"**：更可能说明策略被裁剪或日志通道被限制，属于反向信号。
9. **"ION/ashmem 还在就等于有漏洞"**：它们是旧形态（策略表达集中在 ioctl），不等于存在缺陷；但在采购评估里应视为"可观测性/最小权限较差"的加分项。
10. **"内核 panic 少 = 内核稳定"**：`pstore` 为空可能只是该通道未启用或不可读，不能反推历史无崩溃。
11. **"设备上读不到 `/sys/kernel/debug` 说明调试被关闭"**：Android 12 起 debugfs 只在 `userdebug` 构建挂载，生产构建本就不挂；生产遥测的正确通道是 eBPF/内核遥测。
12. **"内核被攻破 = 硬件密钥丢了"**：密钥材料、门限认证与生物识别匹配在安全世界或独立安全芯片内；内核失陷破坏的是调用完整性与可用性，取密钥需要另一层缺陷。
13. **"模块可以随内核小版本自动复用"**：模块只与构建它的 GKI 内核/KMI 兼容，跨内核版本需要随 `vendor_dlkm`/`odm_dlkm` 一起更新，`modules.dep` 的依赖关系也要成立。

---

## 十五、趋势判断（2024 → 2026）

1. **内核形态进一步收敛**：ACK 支持策略明确化（6.6 起 4 年）、每个内核版本单一 GKI，使"这台设备还能不能收到内核补丁"成为可自动判定的字段。
2. **私有接口继续被通用机制替代**：`ION`→DMA-BUF heaps 已完成，`ashmem`→`memfd` 在推进；下一步是厂商自定义字符设备/ioctl 的收敛（难度更高，因为多在 vendor 侧）。
3. **策略与机制的分离更明显**：核心内核由 Google 维护、模块与策略由 OEM 维护，安全评估的重心随之从"平台版本"转向"vendor 构建新鲜度 + 模块清单"。
4. **可观测性作为产品能力**：`pstore`、Perfetto/ftrace、崩溃与重启原因在设备管理侧被更系统地采集，使"内核被改造/长期失联"的检测窗口缩短。
5. **硬件能力进入判定链**：MTE、PAC/BTI、独立安全芯片（密钥/认证）逐步成为可被证明的属性；固件（基带/安全 OS）的更新与证明仍是行业短板。
6. **Rust 与上游化**：新代码倾向于以受控 ABI 引入并可独立更新，长期效果是把"内核里的新攻击面"限制在更小、更可验证的集合内。

---

## 十六、结论

从结构看，Android Linux 内核在过去十年完成的不是"加了更多防护"，而是**把特权逻辑拆分成三层可分别验证、分别更新的组件**：核心内核（ACK/GKI，Google 维护、KMI 稳定）、厂商模块与驱动（vendor 分区、只读、签名规则明确）、用户态策略进程（`init`/`lmkd`/`apexd`/`vold`/安全后端）。安全评估的正确问法因此是：

1. **这台设备的核心内核是否还在受支持分支形态上？**（第十三章字段 1–3）
2. **厂商模块与 vendor 构建的新鲜度如何？**（模块清单、`ro.vendor.build.date`）
3. **控制是否成对存在？**（SELinux 机制 ↔ 策略与 `avc` 记录；seccomp 机制 ↔ 过滤器来源；只读 ↔ verity enforcing；FBE 机制 ↔ 密钥保护根）
4. **崩溃与日志通道是否可用？**（决定事后能否解释）

把这四个问题回答完整，比逐条对照 CVE 更早一步暴露真实风险——这也是本文与攻防报告分工的意义：一个描述"结构应该是什么"，一个描述"结构被破坏时发生了什么"。

---

## 十七、常见问题

**Q1：GKI 和"厂商定制内核"如何在一台设备上共存？**
核心内核是 Google 构建的 GKI 二进制；厂商代码以 `.ko` 形式在 `vendor_boot`/`/vendor/lib/modules` 侧加载，二者通过 KMI 符号接口连接。

**Q2：为什么"设备树"单独占一个分区？**
`dtbo` 独立便于按机型/硬件版本叠加，且能被 AVB 单独描述与更新，避免改动整个 boot 镜像。

**Q3：`dma-buf` 与 `ION` 的差异为什么算安全改进？**
每个 heap 是独立字符设备节点，权限可以按节点表达（属主/组 + SELinux 类型），而 `ION` 把各类分配集中在一个节点里靠 ioctl 区分，策略表达粒度更粗、可见性更差。

**Q4：`memfd` 取代 `ashmem` 后应用还能共享内存吗？**
能。`memfd` 创建匿名文件内存并通过 fd 传递（Binder 支持传 FD），因此共享能力不变，策略走 VFS/DAC/SELinux 通用路径。

**Q5：低内存杀进程是内核做的吗？**
Android 10+ 由用户态 `lmkd` 基于 PSI/`vmpressure` 决策，内核内 `lowmemorykiller` 已移除。杀错进程/策略被配置影响，都属于用户态问题。

**Q6：`fs-verity` 与 `dm-verity` 需要同时存在吗？**
是不同粒度：`dm-verity` 保护只读分区整体，`fs-verity` 保护单个文件（如 APK）。设备基线里应分别核对。

**Q7：交换区会不会绕过文件加密？**
需要看实现：`mmd` 的 ZRAM writeback 目标是 `/data` 上的 backing 文件，该路径受 FBE 覆盖；非 zram 交换则要结合厂商实现与加密策略确认（第十三章脚本会把这种情况标为 INFO 让人核对）。

**Q8：怎么判断"内核可观测性"是否被关闭？**
交叉看三点：`pstore` 目录可读性与内容、`dmesg`/logcat 是否连续、`avc: denied` 是否长期为零。三者同时异常通常是 ROM 改造特征。

**Q9：`vendor_dlkm` 和把模块放在 `vendor` 分区里有什么区别？**
`vendor_dlkm`/`odm_dlkm`（Android 11 引入）是 `super` 内的独立动态分区，挂载后模块路径仍是 `/vendor/lib/modules`、`/odm/lib/modules`。区别在于**可以只更新模块集合而不重建整个 vendor 镜像**，因此"模块通道"要单独版本化。

**Q10：能不能只看 `uname -r` 判断设备内核有没有在受支持分支上？**
不能。`uname -r` 只给版本串，不区分 GKI 构建、厂商自建、也不反映模块集合；应与 `/proc/cmdline`、`ro.build.type`、SPL、`ro.vendor.build.date` 以及官方 GKI release 产物清单交叉比对（第十三章）。

---

## 十八、参考资料（公开来源，本文事实均来自以下页面）

1. AOSP — Kernel overview（内核类型与术语）：https://source.android.com/docs/core/architecture/kernel
2. AOSP — Android common kernels（分支命名、KMI 标识、支持生命周期）：https://source.android.com/docs/core/architecture/kernel/android-common
3. AOSP — Generic Kernel Image (GKI)：https://source.android.com/docs/core/architecture/kernel/generic-kernel-image
4. AOSP — GKI release builds：https://source.android.com/docs/core/architecture/kernel/gki-release-builds
5. AOSP — android16-6.12 release builds：https://source.android.com/docs/core/architecture/kernel/gki-android16-6_12-release-builds
6. AOSP — Loadable kernel modules（签名规则、模块位置、白名单符号）：https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules
7. AOSP — Kernel modules overview：https://source.android.com/docs/core/architecture/kernel/modules
8. AOSP — Transition from ION to DMA-BUF heaps（Android 12 / GKI 2.0）：https://source.android.com/docs/core/architecture/kernel/dma-buf-heaps
9. AOSP — Incremental File System（Android 11）：https://source.android.com/docs/core/architecture/kernel/incfs
10. AOSP — Extend the kernel with eBPF：https://source.android.com/docs/core/architecture/kernel/bpf
11. AOSP — Generic boot partition（Android 12 GKI boot 内容）：https://source.android.com/docs/core/architecture/partitions/generic-boot
12. AOSP — Vendor boot partitions：https://source.android.com/docs/core/architecture/partitions/vendor-boot-partitions
13. AOSP — Boot image header（v4 `boot_signature`）：https://source.android.com/docs/core/architecture/bootloader/boot-image-header
14. AOSP — Low memory killer daemon（lmkd / PSI）：https://source.android.com/docs/core/perf/lmkd
15. AOSP — Memory management daemon（ZRAM writeback）：https://source.android.com/docs/core/perf/mmd
16. AOSP — Memory Limiter（cgroup `memory.swap.max`）：https://source.android.com/docs/core/perf/memory-limiter
17. AOSP — Application Sandbox（每应用 UID、SELinux 5.0、seccomp-bpf 8.0、Android 9 per-app 沙箱）：https://source.android.com/docs/security/app-sandbox
18. AOSP — SELinux：https://source.android.com/docs/security/features/selinux
19. AOSP — Android Verified Boot：https://source.android.com/docs/security/features/verifiedboot
20. AOSP — On-device signing architecture（fs-verity）：https://source.android.com/docs/security/features/verifiedboot/on-device-signing-architecture
21. AOSP — File-based encryption：https://source.android.com/docs/security/features/encryption/file-based
22. AOSP — Metadata encryption：https://source.android.com/docs/security/features/encryption/metadata
23. AOSP — Build kernels：https://source.android.com/docs/setup/build/building-kernels
24. Linux Kernel docs — Ramoops oops/panic logger：https://docs.kernel.org/admin-guide/ramoops.html
25. Linux Kernel docs — fs-verity：https://docs.kernel.org/filesystems/fsverity.html
26. Linux Kernel docs — /proc/sys/kernel（`dmesg_restrict`、`kptr_restrict`）：https://docs.kernel.org/admin-guide/sysctl/kernel.html
27. LWN — Destaging ION：https://lwn.net/Articles/792733/
28. LPC 2025 — Transitioning Android from ashmem to memfd：https://lpc.events/event/19/contributions/2117/attachments/1727/3782/LPC%202025_%20Transitioning%20Android%20from%20ashmem%20to%20memfd.pdf
29. Chromium OS minijail（Android 原生服务沙箱工具）：https://chromium.googlesource.com/chromiumos/platform/minijail/
30. platform/system/core — libcutils/ashmem-dev.cpp（memfd 迁移过渡说明）：https://android.googlesource.com/platform/system/core/+/refs/heads/main/libcutils/ashmem-dev.cpp
31. AOSP — Trusty TEE（与 Android 同处理器上运行的安全 OS）：https://source.android.com/docs/security/features/trusty
32. AOSP — Vendor and ODM DLKM partitions（Android 11 引入、挂载后路径）：https://source.android.com/docs/core/architecture/partitions/vendor-odm-dlkm-partition
33. AOSP — Implement a GKI module partition（`system_dlkm` 与 `modules.dep` 依赖）：https://source.android.com/docs/core/architecture/partitions/gki-partitions
34. AOSP — Partitions overview：https://source.android.com/docs/core/architecture/partitions
35. AOSP — Develop kernel code for GKI（隐藏配置须经 `init/Kconfig.gki` / `CONFIG_GKI_HACKS_TO_FIX`）：https://source.android.com/docs/core/architecture/kernel/kernel-code
36. AOSP — Use DebugFS in Android 12（debugfs 仅 `userdebug` 挂载、生产改用 eBPF 遥测）：https://source.android.com/docs/core/architecture/kernel/using-debugfs-12
37. AOSP — Android kernel file system support（`ext4`/`f2fs`/`EROFS`/`exfat`/`fuse`/`incfs`/`vfat`，要求 `fscrypt`/`fs-verity`）：https://source.android.com/docs/core/architecture/android-kernel-file-system-support
38. AOSP — android15-6.6 release builds：https://source.android.com/docs/core/architecture/kernel/gki-android15-6_6-release-builds
39. AOSP — Debug features with GKI（内核调试与遥测入口集合）：https://source.android.com/docs/core/architecture/kernel/debugging-with-gki
40. man7 — proc_pid_status(5)（`Seccomp`/`NoNewPrivs`/`CapEff` 字段语义）：https://man7.org/linux/man-pages/man5/proc_pid_status.5.html
41. man7 — seccomp(2)：https://man7.org/linux/man-pages/man2/seccomp.2.html

---

咨询ios系统请咨询 telegram：https://t.me/one00190
