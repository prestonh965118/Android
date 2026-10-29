---
layout: default
title: "Android 媒体与加速器固件缺陷防御分析报告"
description: "GPU 命令流固件、DSP 协处理器固件、媒体编解码器引擎固件与相机 ISP 固件的六层归属（F1–F6）、公告 component 词表与 Subcomponent 实测取值、Closed-source component 占位串、严重度外包与 SPL 不移动、厂商公告库四类入口的可见性实测、JM 与 CSF 的分界、Adreno 签名固件与驱动侧微码安全版本门、MFC/Venus/SCP 三种装载路径、remoteproc 的 unknown 状态、设备侧八类读数面与 shell 域预期失败表、FT1–FT10 缺陷类型学、十三组公开案例读数、五个编号误传反例、归属矩阵、四级判定模型与 K1–K10 运营指标、两个只读核验脚本（固件姿态快照与两次快照的漂移比对）及全量分支覆盖验收。"
lang: zh-CN
permalink: /Android-Media-and-Accelerator-Firmware-Defect-Defense-Analysis/
---
# Android 媒体与加速器固件缺陷防御分析报告

**Android Media and Accelerator Firmware Defect Defense Analysis: Processor-side Firmware, the Attribution Vocabulary, Update Channels and Device-side Visibility Boundaries**

- **报告日期**：2026-10-29
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动平台安全负责人、OEM/ODM 的系统与集成工程师、供应商管理与固件资产负责人、移动设备管理与取证人员、内部安全评估执行者
- **取证边界**：本文给出的全部设备侧动作都是**只读采集**——读属性、读目录元数据、读 `dumpsys` 输出、读公开内核接口、比对两次快照。文中不含任何写入、降级、替换镜像、绕过签名校验或构造利用步骤；每一条命令的目的都只是"让设备自己把它已经知道的事情说出来"。执行者应当在**自己的或已获明确授权的设备**上进行，并遵守厂商的许可条款与内部变更流程。
- **视角声明**：本文写的是**"缺陷住在另一颗处理器上的那段镜像里时，它归谁、怎么到达设备、你在设备上能不能自证"**——GPU 命令流固件、DSP 协处理器固件、媒体编解码器引擎固件、相机 ISP 固件与 AI 加速器固件这同一档"器件内部可执行镜像"的归属层（F1–F6）、公告词表与设备侧证据之间的落差、三种装载路径的差异、以及"能证明器件在场但证明不了镜像版本"这条硬边界。**内核驱动侧的 ioctl 攻击面、基带固件的送达治理、可信执行环境的器件形态、APEX 模块的归属账、CPU 微架构观测通道一律不在本文**，只做交叉引用而不重复展开：
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（该篇把 GPU/DSP 的**内核驱动**攻击面当作子系统清单来枚举——ioctl、`/dev/kgsl`、`dma-buf`、rpmsg 通道与提权取证都在那里；本文只写驱动之下那段镜像本身，两篇的接口口径以该篇为准，本文不重复判定缺陷落在哪个驱动节点）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（该篇已经把"基带固件"作为六层之一完整处理过一轮——它的责任人、更新通道、公告可见性与"设备侧只读到一个版本串"的结论；本文沿用同一套口径但换一类器件：媒体与加速器固件的通道不止基带那一条，且它的严重度评定权同样外包）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（该篇写**器件形态**与可外部核验信号，安全处理器那一档与本文的协处理器那一档在硬件上常常是同一颗核；本文不判定器件形态，只判定"镜像版本能不能自证"这条边界，两篇的可见性结论互补）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（该篇管 S1–S5 链条与 SPL 两档语义；本文只取它的一个结论作为前提——厂商侧公告里的缺陷**不作为声明补丁级别的条件**，并把这条前提落到加速器固件上）
  - [Android Project Mainline 与 APEX 模块缺陷归属分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Project-Mainline-and-APEX-Module-Defect-Attribution-Analysis.html)（该篇的 `com.android.media` 是**软件编解码器的可更新单元**；本文的对象是同一颗编解码器**引擎内部的固件镜像**，它不在那张模块表里，也不作为 APEX 下发——两篇合起来才是"媒体"这一域的两侧）
  - [Android 远程代码执行分析报告：投递通道、缺陷类型学与防御侧核验](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（该篇的"内容解析器"通道讲的是**不可信输入怎样走到解析代码**；本文讲的是解析代码把活儿交给谁之后发生的事，两篇在执行链上首尾相接而不重叠）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（该篇管一个进程被围在哪；本文的取证面恰好要问"围住之后 shell 域还能读到哪一段"，其域模型与 `neverallow` 口径按该篇读取）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（该篇写 `vendor`/`vendor_boot`/`odm` 分区怎样装配、构建能否归因；本文的"blob 目录不可列举"这条结论依赖它的分区装配口径，本文不重复展开装配流水线）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（该篇给出九域可达面清点单元与 R0–R4 分级；本文按它的口径说明"加速器固件这一域为什么数不出可达面"，并把"读数缺失为什么不能当作无缺陷"当作方法论点单独展开）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（该篇已经给出"微码是唯一一条设备侧几乎无法自证的层"这个结论；本文对固件镜像给出同构但不同源的结论：读不到的原因不是缓解编在内核里，而是**镜像所在目录对 shell 域关闭**）
  - [Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（该篇写子系统与设备节点怎样供给；本文使用的 `remoteproc`、`rpmsg`、`drm`、`media` 四处接口口径以该篇为准，本文只读它们暴露出的版本与状态字段）

---

## 一、摘要：这一类缺陷不在你的补丁台账里，也不在你的资产表里

一颗 SoC 上不止跑一个操作系统。GPU 里有命令流处理器，音视频编解码器里有自己的引擎核，相机管线里有 ISP 固件，AI 加速器里有 NPU/APU 微码，音频与传感器各有协处理器。这些核上跑着**镜像**——一段由器件厂商或其授权方提供、由内核或守护进程装载、由硬件加载签名或校验的可执行体。本文讨论的就是这一段镜像里的缺陷。

它难办的地方不在"缺陷有多严重"，而在三条口径同时错位：

1. **词表错位**。公开公告里没有"固件"这个词。它用器件名说话（`Mali`、`PowerVR-GPU`、`hevc decoder`、`apusys`、`Video`），或者干脆用一个占位串说话（`Closed-source component`）。于是同一颗芯片上的一次修复，在公告表里可能表现为"某个 GPU 型号"，也可能表现为"什么都不是"。
2. **台账错位**。Android 侧的公告只负责"声明补丁级别所需的那部分缺陷"，厂商侧公告里的缺陷**不作为声明 SPL 的条件**。加速器与器件固件的修复因此经常不移动 SPL，也不进入 Mainline 模块表；它在设备上的存在形式是"某个分区里换了字节的镜像"。
3. **证据错位**。设备侧对这段镜像几乎没有自证通道。GPU 微码版本要通过一次字符设备上的 `ioctl` 才拿得到，而 shell 域对 GPU 设备节点没有任何允许规则；Mali CSF 固件版本只出现在内核日志一行文本里，而 shell 域读不到内核日志；编解码器固件所在的 `/vendor/firmware` 对 shell 域**不可列举**；协处理器在"由外部实体启动"的状态下，内核直接回答 `unknown` 并写明"我们不知道它跑的是哪个镜像"。

三条错位叠在一起的结果是：**你能证明这颗芯片上有这个器件，但证明不了它的镜像是哪一个版本；而恰恰是这个版本决定了它的已知缺陷是否已经修掉。**

### 一之一 本报告给出的五件事

| 编号 | 这件事是什么 | 落在哪一章 |
| --- | --- | --- |
| 一 | 一套能把任何一段器件镜像钉住的**六层归属模型（F1–F6）**：镜像本体、装载者、内核驱动、用户态 HAL/运行时、平台策略、公告与资产口径 | 第三、廿五章 |
| 二 | 一份**公告词表实测**：六个近期公告里全部厂商表名与 Subcomponent 取值的原样集合，以及"占位串"这一档为什么必须单列 | 第五至八章 |
| 三 | 一条**装载路径的分型**：签名固件走安全world、`request_firmware` 走文件系统、`remoteproc` 代载走子系统状态——三条路径给出三种完全不同的设备侧证据 | 第十至十六章 |
| 四 | 一张**设备侧可见性边界表**：八类读数面（V1–V8）与 shell 域预期失败表，逐项钉到 sepolicy 的规则行上 | 第十九、二十章 |
| 五 | 两套可执行的**只读核验脚本**与十个运营指标（K1–K10），把"读到了什么、读不到什么、为什么读不到"变成台账 | 第廿七至廿八章 |

### 一之二 五条台账级结论

下面五条是本文其余各章要反复回到的落点。每条后面都跟着它在公开来源里的原句位置，不带任何推测。

**L-1　公告不用"固件"这个词，它用器件名或者占位串。**
在实测的六个公告页面里，厂商表标题共 21 个取值，其中与本域相关的四个是 `Arm components`、`Imagination Technologies`、`MediaTek components`、`Qualcomm closed-source components`。GPU 侧的 Subcomponent 取值集合只有两个词：`Mali` 与 `PowerVR-GPU`。而 `Qualcomm closed-source components` 表里 11 行的 Subcomponent **一律是 `Closed-source component`**——它是一个占位串，不是一个可映射到器件的名字（第五、六章）。

**L-2　严重度不是 Android 侧评的。**
公告对这一档缺陷的原句是 `These vulnerabilities affect Qualcomm closed-source components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm.`（第六章）。同一句话也把披露责任一并转给了厂商，而厂商入口的实测形态是：Qualcomm 侧返回同一个 SPA 外壳、Arm 单篇文档落在需二次验证的应用壳、Samsung 移动站点直接 403、只有 MediaTek 的月度公告可匿名读取（第九章）。

**L-3　这类修复通常不移动 SPL。**
公告 FAQ 的原句是 `Additional security vulnerabilities that are documented in the device / partner security bulletins are not required for declaring a security patch level.`（第八章）。因此在台账上"SPL 已是本月"与"GPU 微码已含本月那条修复"是两件事，后者不由前者推出。Android 侧为这种时间差准备的机制是补充补丁清单：`OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product) ... The XML file should be installed in /etc/security/supplemental_security_patches.xml for the respective partition.`，其中 `KEY_VENDOR_SUPPLEMENTAL_PATCHES: Aggregates patches from /vendor and /odm partition paths.`（第八章）——这是**唯一一条能让器件侧修复进入官方记账的通道**。

**L-4　镜像版本在设备侧不是"难读"，而是接口本身不给出。**
三个原句把这条边界钉死：Mali CSF 侧，内核只把版本打进日志一行——`drm_info(&ptdev->base, "CSF FW v%d.%d.%d, Features %#x Instrumentation features %#x"`（`panthor_fw.c:924`）；Exynos MFC 侧，版本读出来后只交给 debug 级输出——`mfc_debug(2, "MFC F/W version : %02xyy, %02xmm, %02xdd\n"`（`s5p_mfc_ctrl.c:286-287`）；`remoteproc` 侧，当协处理器由外部实体启动时，`firmware` 文件直接返回 `unknown`，源码注释逐字是 `If the remote processor has been started by an external entity we have no idea of what image it is running. As such simply display a generic string rather then rproc->firmware.`（`remoteproc_sysfs.c:138-152`）。**内核承认自己不知道**，这是本域最诚实的一句自述。

**L-5　blob 所在目录对 shell 关闭，能读的只有配置目录。**
平台策略把所有非 `/vendor/etc` 的 vendor 路径统一标成一个类型（`/(vendor|system/vendor)(/.*)?                  u:object_r:vendor_file:s0`，`file_contexts:423`），把"能读能列"的权限只给**非 coredomain** 的域（`allow { domain -coredomain } vendor_file_type:dir r_dir_perms;`，`domain.te:191`），而 `typeattribute shell coredomain, mlstrustedsubject;`（`shell.te:1`）把 shell 划在 coredomain 内。相反，配置目录对所有域开放（`allow domain vendor_configs_file:dir r_dir_perms;` 与 `allow domain vendor_configs_file:file { read open getattr map };`，`domain.te:177-178`；标签来源 `/(vendor|system/vendor)/etc(/.*)?              u:object_r:vendor_configs_file:s0`，`file_contexts:427`）。**设备侧因此能枚举 HAL 与编解码能力声明，却不能枚举固件镜像文件本身**——这正是第十九章八类读数面的由来。

### 一之三 一句可以被误用的话，提前拦下

本文给出的所有读数缺失，都**不能**被读成"这台设备没有加速器固件缺陷"。第廿六章会给出四级判定模型，其中 L0-LOW 的定义不是"安全"，而是"证据不足且无法补采"。第廿一章在给出类型学时同时给出每一条成立所需的前提；第廿三章给出五个**编号形态陷阱**与六条**主键误传**，用实测说明为什么"抄来的编号"比"没有编号"更危险：用 4 位正则去截 5 位编号，一期就能造出 62 个格式合法、现实中不存在的 CVE 编号。
## 二、四个词的分工：固件、微码、blob、驱动，以及五种混用

本域的讨论一旦跨过厂商边界就立刻失去共同语言。同一份东西，在设备树里叫 `firmware-name`，在文件系统里叫 `venus.mbn`，在内核符号里叫 `ADRENO_FW_PM4`，在公告表里叫 `Closed-source component`，在厂商支持文档里叫"某个 image"。本节把四个常用词钉住，并给出本文的用词约定。

### 二之一 三条公开定义句

**第一条，来自驱动源码的注释**，是本报告里对"固件"最朴素、也最可执行的定义——它只说东西在哪、不说东西是什么：

> `	/* Firmware has to be present as a separate file or compiled`
> `	 * into kernel. */`
> — `kernel/common@android16-6.12:drivers/media/platform/samsung/s5p-mfc/s5p_mfc_ctrl.c:50-51`

这句话的可执行含义是：**如果这段镜像不存在，器件就不能工作**。Exynos MFC 驱动接着给出失败时的原句 `mfc_err("Firmware is not present in the /lib/firmware directory nor compiled in kernel\n")`（同文件 `:75`）。"必须有、否则不干活"是本域所有对象共同的判定入口，也是第十九章读数面 V3（能力声明反推）成立的依据。

**第二条，来自设备树绑定文档**，给出了"固件"与"某颗处理器"之间的对应关系：

> `firmware-name:`
> `description: Firmware name for the Hexagon core`
> — `kernel/common@android16-6.12:Documentation/devicetree/bindings/remoteproc/qcom,adsp.yaml:48-50`

这一句重要在它承认**一颗 Hexagon 核可以对应不同的镜像名**，也就是同一颗硬件在不同产品线上跑不同镜像。于是"芯片型号 → 固件版本"这条常见推理链在第一环就断了（第三、十四章）。

**第三条，来自固件装载器本身**，说明"固件在哪"这个看似 trivial 的问题实际是一组候选路径：

> `static const char * const fw_path[] = {` … `	"/lib/firmware/updates/" UTS_RELEASE,` / `	"/lib/firmware/updates",` / `	"/lib/firmware/" UTS_RELEASE,` / `	"/lib/firmware"` / `};`
> — `kernel/common@android16-6.12:drivers/base/firmware_loader/main.c:474-489`

同文件把 vendor 分区的注入方式写在模块参数注释里，逐字是：

> ` * Typical usage is that passing 'firmware_class.path=/vendor,/firwmare_mnt'`
> ` * from kernel command line because firmware_class is generally built in`
> ` * kernel instead of module. ',' is used as delimiter for setting 10`
> ` * custom paths for firmware loader.`
> — 同文件 `:541-546`

`/firwmare_mnt` 是上游原样的拼写错误，本文按引用规则原样保留，不改写。这条注释是本报告里唯一一处**公开源码直接承认 vendor 固件挂载点存在**的地方；它同时也是命令行参数，不是文件标签，因此不构成"设备侧可读"的证据（第十九章 V6）。

### 二之二 四个词的分工

| 词 | 本文用法 | 它在源码里的锚点 | 它不能用来指什么 |
| --- | --- | --- | --- |
| **固件（firmware）** | 由装载器按文件名取入、交给另一颗处理器执行的镜像 | `request_firmware()` 的全部调用点；`CSF_FW_NAME "mali_csffw.bin"`（`panthor_fw.c:27`）；`.fwname = "qcom/venus-1.8/venus.mbn"`（`venus/core.c:559`起） | 不能指 SoC 内部不可替换的微指令 |
| **微码（microcode / ucode）** | 器件内部执行单元的指令序列，版本由镜像头部字段给出 | `a6xx_ucode_check_version()`（`a6xx_gpu.c:712`）；`"a630 SQE ucode is too old. Have version %x need at least %x\n"`（同文件 `:757-759`）；`firmware->version = get_ucode_version((u32 *)fw->data);`（下游 `adreno.c:83-109`） | 不能与 CPU 微码混用——后者在第 15 篇的 H1 层，更新通道完全不同 |
| **blob（二进制件）** | 无源码可得、按原样分发的一类交付物 | 公告 FAQ 的措辞 `The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.` | 不能当成"不可更新"的同义词——blob 也可以进 OTA 镜像 |
| **驱动（driver）** | 与内核或用户态 API 对话的那一半，**在本域里它是唯一有公开可读接口的一侧** | `drivers/gpu/drm/panfrost`、`drivers/misc/fastrpc.c`、`/vendor/lib64/hw/vulkan.<ro.hardware.vulkan>.so`（Vulkan 文档） | 不能把驱动版本当成固件版本——第十三章给出驱动自己解析固件头拿版本、但这个数不打到任何可读接口上的完整链条 |

### 二之三 五种常见混用，以及它们各自会让判断错在哪

**混用一：把 GPU 驱动版本当成 GPU 固件版本。** Mali CSF 的固件版本由固件自身的共享接口上报（`glb_iface->control->version`，`panthor_fw.c:882`），内核把它打进日志 `CSF FW v%d.%d.%d`；而驱动版本是内核版本号或 `panfrost`/`panthor` 模块版本。两者可以差很多，因为 CSF 固件常与 SoC 一起发布、不随内核升级重编。后果：台账里的"GPU 驱动已更新"完全推不出"命令流固件已含该修复"。

**混用二：把"没有 AOSP 源码"当成"没有更新通道"。** 公告 FAQ 对 `*` 号的原句已经说明修复的实际载体：`Issues that are not publicly available have an * next to the corresponding reference ID. The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.`（第六章）。不可读 ≠ 不可更；本文第四篇（补丁传导链）与本文第廿四章把这两件事分开处理。

**混用三：把协处理器统称 DSP，进而把它的通道统称 IPC。** 公开源码里跨处理器通道至少有三种不同形态：`remoteproc` 管生命周期（`rproc_attrs[]` 暴露 coredump/recovery/firmware/state/name，`remoteproc_sysfs.c:245-251`）、`rpmsg` 管消息（节点名 `dev_set_name(dev, "rpmsg%d", ret);`，`rpmsg_char.c:459`）、`fastrpc` 管 RPC 会话（misc 设备名 `fastrpc-%s%s` 加 `-secure` 后缀，`fastrpc.c:2296-2297`）。它们的设备标签、可达域、失败语义各不相同。第十八章按节点名给出四层命名对照表。

**混用四：把"编解码器"当成一个对象。** 同一颗编解码器上至少有三层可分别出缺陷：软件解析器（在 APEX 里，第 16 篇的 `com.android.media` 那一档）、内核驱动与 API（第 1 篇那一档）、引擎固件（本文）。MediaTek 公告在同一张表里同时给出 `hevc decoder`、`vdec` 与 `apusys` 三种 Subcomponent，恰好说明厂商自己的粒度也不是"编解码器"一层（第五、十六章）。

**混用五：把"厂商公告里查不到"当成"厂商不知道"。** MediaTek 月度公告的原句直接反驳这条：`Device OEMs have been notified of all the issues and the corresponding security patches for at least two months before publication. … The severity of the identified vulnerabilities was conducted based on the Common Vulnerability Scoring System version 3.1 (CVSS v3.1).`（第九章）。**公开披露滞后于 OEM 已知至少两个月**——这条时间差是本域资产台账最常见的一种失真来源，第廿七章的 K4 指标就是为它设计的。

### 二之四 术语约定

为避免歧义，本文全篇按下列约定使用：

- **器件镜像**：固件与微码的上位词，指任何"跑在本 CPU 之外、由外部交付、以整件形式替换"的可执行体。
- **装载**：把镜像从文件系统或子系统接口交给器件执行的过程。本文不区分"加载/载入/下载"，一律用"装载"，并在需要时指出装载者（`request_firmware` / `qcom_scm_pas_auth_and_reset` / `rproc_boot`）。
- **读数面**：设备上一次只读采集能拿到的一类事实来源。第廿七章脚本给出的每个信号码都归属到某一个读数面，共八类，记 V1–V8。
- **可达面**：某个未信任主体能否触到该器件接口。本文只在第廿一章的类型学前提里使用它，并按第 15 篇（攻击面方法论）的 R0–R4 口径引用，不重复展开。
- **归属**：一条缺陷被记在哪个对象上。本域有四个归属候选——器件型号、公告表名、分区/文件、镜像版本，第五至八章说明它们为什么互不等价。

## 三、六个责任层：镜像本体、装载、驱动、运行时、策略、公告

本域最容易犯的方法错误是"一层解释全部"。缺陷住在器件里，但它能不能被触发由内核驱动决定，能不能被送达由装载与分区决定，能不能被你证明由平台策略决定，最后能不能进入你的台账由公告口径决定。本文用六层（F1–F6）分别钉住这五件事。

### 三之一 六层的定义与各自的产物形态

| 层 | 名称 | 它决定什么 | 它的产物形态 | 谁改动它 |
| --- | --- | --- | --- | --- |
| **F1** | 镜像本体层 | 缺陷是否**存在** | `.bin` / `.mbn` / `.fw` / ELF 镜像；版本写在镜像头部字段里 | 器件厂商（IP 供应商或其被授权方） |
| **F2** | 装载与校验层 | 缺陷是否**能被装进设备**、装错了会不会被拒 | 搜索路径、签名/PAS 授权、`remoteproc` 状态机 | 内核 + 引导链 + 厂商 boot 配置 |
| **F3** | 内核驱动层 | 缺陷是否**可被触达**、旧镜像是否被拒绝使用 | `drm`/`media`/`remoteproc`/`rpmsg`/`fastrpc` 驱动代码 | 上游内核维护者 + 厂商下游分支 |
| **F4** | 用户态运行时与 HAL 层 | 谁**有机会**把活儿交给这颗核 | `vulkan.<board>.so`、`libadsprpc.so`、codec2/camera HAL | SoC 厂商 + OEM |
| **F5** | 平台策略与取证层 | 设备侧**能不能自证** | SELinux 类型与规则、属性上下文、`dumpsys` 服务授权 | AOSP 平台 + vendor sepolicy |
| **F6** | 公告与资产口径层 | 修复**算在谁头上、显示在哪** | component/Subcomponent 词表、References 前缀、SPL、补充补丁 XML | Android 侧 + 厂商公告 + 你自己的 CMDB |

### 三之二 F1：镜像本体层——版本写在镜像里，不是写在你身上

同一族 GPU 有两种根本不同的形态。Arm Mali 的 **JM（Job Manager）** 架构没有命令流固件：`panfrost` 驱动树里对 `firmware` 的检索在 `kernel/common@android15-6.6:drivers/gpu/drm/panfrost/` 全部 21 个源文件（`.c` / `.h`）中命中 **0 次**（同分支 `panthor/` 为 18 个源文件、40 次命中，对照见第十章之二），GPU 身份完全来自寄存器读取——`gpu_id = gpu_read(pfdev, GPU_ID);`（`panfrost_gpu.c:266`），随后打印 `dev_info(pfdev->dev, "mali-%s id 0x%x major 0x%x minor 0x%x status 0x%x"`（`:309-310`）。而 **CSF（Command Stream Firmware）** 架构把固件当成运行前提：`#define CSF_FW_NAME "mali_csffw.bin"`（`panthor_fw.c:27`）。这一分界不由本文归纳，Kconfig 帮助文本逐字给出：

> `	  Note that the Mali-G68 and Mali-G78, while Valhall architecture, will`
> `	  be supported with the panfrost driver as they are not CSF GPUs.`
> — `kernel/common@android16-6.12:drivers/gpu/drm/panthor/Kconfig:22-23`

F1 的方法结论是：**"这台设备的 GPU 有没有固件"这个问题必须按型号回答，不能按家族回答**。同一代架构里，一半有、一半没有；台账如果按"GPU = Mali-G78"记录，就无法判断它属不属于本文的处理范围（第十章给出完整分界表）。

### 三之三 F2：装载层给出三种互不相通的到达方式

| 方式 | 源码锚点（逐字） | 失败时的原句 | 设备侧留下什么 |
| --- | --- | --- | --- |
| **文件系统装载** | `ret = request_firmware(&fw, fw_path, ptdev->base.dev);`（`panthor_fw.c:708`）；`err = request_firmware((const struct firmware **)&fw_blob, dev->variant->fw_name[i], &dev->plat_dev->dev);`（`s5p_mfc_ctrl.c:66-67`） | `"Failed to load firmware image '%s'\n"`、`"Firmware is not present in the /lib/firmware directory nor compiled in kernel\n"` | 内核日志一行；目录本身对 shell 不可读 |
| **安全 world 授权装载** | `/* Send the image to the secure world */` / `ret = qcom_scm_pas_auth_and_reset(pasid);`（上游 `adreno/adreno_gpu.c:155-156`）；`if (core->use_tz) ret = qcom_mdt_load(dev, mdt, fwname, VENUS_PAS_ID, …)`（`venus/firmware.c:136-141`） | `"Unable to authorize the image\n"`；`"fail to load video firmware\n"` | 装载与否由 TZ 判定，用户态无任何返回值 |
| **子系统代载** | `rproc_boot(scp_get_rproc(fw->scp))`（`mtk_vcodec_fw_scp.c:7-10`）；`ret = rproc_of_parse_firmware(dev, 0, &fw_name);`（`mtk_scp.c:1126`）；`ret = request_firmware(&firmware_p, rproc->firmware, dev);`（`remoteproc_core.c:1807`） | `"request_firmware failed: %d\n"`；`"can't change firmware while running\n"`（同文件 `:2210`） | `/sys/class/remoteproc/remoteprocN/{name,state,firmware}` 三个文件——但对 shell 域不可读（第十九章） |

三条路径的**共同点**是：装载决定权在 F2，不在任何用户可查询的接口上。**差别**在于失败的可观测性：文件系统装载会在日志留一行文本；安全 world 授权只留"拒绝"这一件事、不留原因；子系统代载把状态存进一个文件，但那个文件对审计者关闭。

### 三之四 F3：驱动层是唯一一处"驱动知道固件版本、并且据此拒绝工作"的层

这是本域最重要的一条结构性事实。上游 Adreno 驱动里，安全相关的判定被直接写进代码：

> `/*`
> ` * Check that the microcode version is new enough to include several key`
> ` * security fixes. Return true if the ucode is safe.`
> ` */`
> `static bool a6xx_ucode_check_version(struct a6xx_gpu *a6xx_gpu,`
> — `kernel/common@android16-6.12:drivers/gpu/drm/msm/adreno/a6xx_gpu.c:709-712`

它按目标系列给出不同门槛，逐字：

> `			 * Otherwise check that the firmware is greater than or equal`
> `			 * to 1.90 which was the first version that had this fix built`
> — 同文件 `:746-747`

不满足时的行为是**拒绝启动这块 GPU 并打印版本**：`"a630 SQE ucode is too old. Have version %x need at least %x\n"`（`:757-759`）。也就是说：**内核自己知道这段固件的安全状态，并且宁可不出图也不跑旧镜像**（第十二章）。这与第五、六章公告里"严重度由厂商评"的姿态形成对照：驱动层的判定强、但不外宣；公告层的表述弱、且常常是占位串。

### 三之五 F4：运行时层只保证"能力"，不保证"版本"

Vulkan 文档给出的实现归属关系逐字是 `A Vulkan driver, provided by SoCs such as GPU IHVs, that implements the Vulkan API .` 与 `Consult your SoC vendor to request driver support.`；驱动落点为 `/vendor/lib64/hw/vulkan.<ro.hardware.vulkan>.so` 与 `/vendor/lib64/hw/vulkan.<ro.board.platform>.so`。从 Android 15 起用户态驱动可以走 APEX：`Since Android 15, the loader supports APEX to load the Vulkan driver. Set ro.vulkan.apex to the name of the Vulkan APEX to load the Vulkan from the APEX.`——这是本域**唯一**一条"可以按模块更新"的官方通道，但注意它更新的是 F4 的用户态驱动，不是 F1 的命令流固件（与第 16 篇的模块表互为补集）。

AI 加速器一侧，运行时层正在退场。NNAPI 的公开状态是：`Warning: NNAPI is deprecated. While you can continue to use NNAPI, we expect the majority of devices in the future to use the CPU backend, and therefore for performance critical workloads, we recommend migrating to alternative solutions, for example the TF Lite GPU runtime.` 与 `The API is available on all Android devices running Android 8.1 (API level 27) or higher, but was deprecated in Android 15.`；同时它也承认这条路径的前提是厂商驱动存在：`For Android devices that lack a specialized vendor driver, the NNAPI runtime executes the requests on the CPU.`。**运行时层被废弃，不意味着底下的 NPU 固件消失**——它只是失去了一个统一的可查询点（第十六章）。

### 三之六 F5：策略层决定"你能不能证明"，并且它对 GPU 与 DSP 给出的答案不同

三个逐字规则构成第十九章取证表的基础：

- `allow shell sysfs:dir r_dir_perms;`（`shell.te:443`）——只给目录读，不给文件读；配合 `genfscon sysfs / u:object_r:sysfs:s0`（`genfs_contexts:124`），意味着**任何未单列标签的 sysfs 文件对 shell 都读不到**，包括 `/sys/class/remoteproc/remoteprocN/name`。
- `genfscon sysfs /class/gpu                         u:object_r:sysfs_gpu:s0`（`genfs_contexts:129`）给了 GPU 一个独立类型，而 `private/shell.te` 全文检索 `sysfs_gpu` 命中 **0 次**——GPU sysfs 对 shell 关闭。
- `allow shell dev_type:dir r_dir_perms;` 与 `allow shell dev_type:chr_file getattr;`（`shell.te:469-470` 段）——**可以列 `/dev`、可以 stat 设备节点，但不能打开**；因此"这块 GPU 的字符设备存在"是可证的，"它的微码版本"是不可证的（第十三章）。

一处反向例外值得单独记住：`dumpsys gpu` 对 shell 有效，因为服务端显式豁免了它——`if ((uid != AID_SHELL) && !PermissionCache::checkPermission(sDump, pid, uid)) {`（`frameworks/native@main:services/gpuservice/GpuService.cpp`），配合 `binder_call(shell, gpuservice);` 与注释 `# Allow shell to run adb shell cmd gpu commands.`（`shell.te:78-79`）。这是本域唯一一处"策略层主动为审计者开门"的位置，第十九章把它列为 V2。

### 三之七 F6：公告层用四类名字指同一个对象，其中一类是占位串

同一颗器件的修复，在 F6 可能有四种记法：表名（`Arm components`）、Subcomponent 值（`Mali`）、References 前缀号（`QC-CR#4409501` / `M-ALPS11104718*`）、或占位串（`Closed-source component`）。第五至第七章逐一给出这四类的实测取值与语义边界，并说明为什么"按 Subcomponent 建资产映射"这一常见做法在 `Qualcomm closed-source components` 这张表上**结构性地不可行**。

### 三之八 六层之间的一条因果链

用一句话说完本域的因果结构，后面各章都是它的展开：

> **缺陷存在与否由 F1 决定，能否送达由 F2 决定，能否触发由 F3/F4 决定，能否被你证明由 F5 决定，能否进入你的台账由 F6 决定。这五层由不同主体控制、更新节奏不同、公开程度不同，所以任何"用其中一层推另一层"的做法都必须显式给出前提。**

第四篇用同样的句式处理了 SPL 与厂商更新；本文把它落到"镜像"这一档，并给出一个该篇没有的对象：**一个既不在 AOSP 源码里、又不在模块表里、又不在可读目录里的更新单元。**
## 四、为什么这一档必须单列：与四篇近邻的分工裁决

本域与四篇已发布报告在关键词上高度重叠（GPU、DSP、固件、vendor、媒体）。本章把分工钉死，并给出一条冲突时的裁决规则——**当同一件事在两篇里都出现时，按"这条结论的主体是谁"归属，不按关键词归属。**

### 四之一 四面分工表

| 相邻报告 | 它已经把这件事讲到什么深度 | 本文让出什么 | 本文独占什么 |
| --- | --- | --- | --- |
| **内核漏洞与提权防御**（第 1 篇） | 把 GPU/DSP 的**内核驱动**按子系统枚举，给出 ioctl 面、设备节点与提权取证形态 | 驱动节点的可达性判定、内核侧缺陷类型学、提权链取证 | 驱动**之下**那段镜像：它的版本语义、装载路径、拒绝行为与不可观测性 |
| **无线协议栈与基带缺陷防御**（第 14 篇） | 把"基带固件"作为六层之一完整处理：责任人、通道、公告可见性、"设备侧只有一个版本串" | 蜂窝基带本体、`邻近即远程` 的严重度基线 | 与基带同构但通道不同的其余核：GPU/DSP/ISP/编解码器/NPU；并给出**三条**装载路径而非一条 |
| **厂商硬件安全组件形态**（第 8 篇） | 器件形态（独立安全处理器、安全世界、Trusty/OP-TEE、Knox Vault/Titan M2）与可外部核验信号 | 器件是不是"安全"器件、它的封装形态 | 非安全器件的镜像同样需要归属这一档；给出"能力声明可读而镜像不可读"的策略级解释 |
| **Project Mainline 与 APEX 模块归属**（第 16 篇） | 把 `com.android.media` 作为可更新单元，给出四套词表换名与模块清单口径 | 任何 APEX/模块维度的归属判定 | **不在模块表里的更新单元**：它如何送达、如何留痕、如何计数；以及 `ro.vulkan.apex` 这条唯一例外 |

### 四之二 一条裁决规则，避免两篇互相抄

同一句"MediaTek `Video` 子部件的一个 High 严重度缺陷"在两篇里都可能出现。裁决规则：**若结论的主语是"这个缺陷能不能触发"，归内核篇；若主语是"这个缺陷能不能被公告映射到器件"，归本篇；若主语是"它有没有到设备"，归第 9 篇（补丁传导链）。** 本文全篇只使用第二类与第三类主语，并在需要第一类结论时显式引用第 1 篇而不重述其类型学。

### 四之三 本域值得单列的三条理由

1. **它不属于任何一张现成的账表。** 器件镜像既不是 AOSP 源码头上的一个路径，也不是 `apex-info-list.xml` 里的一行，也不是 SPL 字符串的一部分。第 16 篇的结论"器件缺陷落不进模块表"在本域被进一步具体化：它同时也落不进补丁级别表。
2. **它的公开词表与设备侧证据之间存在制度性落差。** 公告把严重度交给厂商（第六章），厂商入口四类里只有一类可匿名读取（第九章），而设备侧连镜像文件名都列举不出（第 L-5 条）。**落差是制度造成的，不是工具造成的**，因此需要一套专门的取证与台账方法。
3. **它的"能不能工作"与"安不安全"由同一段代码判定，而这段代码在驱动里。** `a6xx_ucode_check_version()` 的注释直接说 `Return true if the ucode is safe.`（第三之四节）。这是 Android 各层里罕见的一处——**安全属性被编译进功能路径**。它意味着设备状态本身（GPU 能不能出图）会泄漏固件版本信息，而这正是第十九章 V7（行为反证）的设计依据。

## 五、公告词表实测：表名、Subcomponent 取值与两处缺失

本章只记录"页面上实际出现的字符串"。凡本章给出的取值，都能在第卅章的取回记录里找到同一个页面的原句——本章五之一与五之二的全部数字来自第卅章 **D-12**（逐页探测、标题抽取与 `<th>` 分组命令逐字登记在那里）。

### 五之一 表标题的实测集合（23 个，逐字）

样本期是 2025-09-01 至 2026-10-01，逐月探测 `https://source.android.com/docs/security/bulletin/<年>/<日期>`。**这一期的页面分布本身就是证据**：

| 探测结果 | 月份 | 该页作为 `<h3>` 出现的表标题数 |
| --- | --- | --- |
| 404（页面不存在） | 2025-10、2025-11、2025-12 | — |
| 存在，但无缺陷表 | 2026-02、2026-07（只有 `Announcements` 与 `Android and Google service mitigations` 两个小节）、2026-08（连 `Announcements` 都没有，只有缓解小节） | 0 |
| 存在且有缺陷表 | 2025-09（11）、2026-01（1）、2026-03（12）、2026-04（6）、2026-05（2）、2026-06（9）、2026-09（15）、2026-10（3） | 59 行合计 |

也就是 14 次探测里 3 次 404、3 页零缺陷表、8 页有表。把 8 页的表标题取并集，逐字得到 23 个（区分大小写，`Android Runtime` 与 `Android runtime` 按页面上的两种写法分别计入）：

`Android Runtime` / `Android runtime`、`Arm components`、`Dolby`、`Framework`、`Google`、`Google Play system updates`、`Imagination Technologies`、`Kernel`、`Kernel components`、`MediaTek components`、`Misc OEM`、`NXP components`、`Qualcomm closed-source components`、`Qualcomm components`、`STMicroelectronics`、`Setup Wizard`、`System`、`Thales`、`TV`、`Tsingteng Micro`、`Unisoc components`、`Widevine DRM`。

三点必须记入台账：

- **`Pixel` 不是表标题。** 它只在 FAQ 正文里作为 "the Pixel bulletin" 出现（2026-08 页 8 次），没有任何一期把它做成 `<h3>`。设备侧取证不能假定 Pixel 缺陷进本页表格。
- **`NXP` 的逐字写法是 `NXP components`**；与之相对，`Dolby`、`Google`、`Thales`、`TV`、`Misc OEM`、`Tsingteng Micro`、`STMicroelectronics`、`Setup Wizard` 不带 `components` 后缀。后缀本身没有语义，不能用来推断"这是 IP 供应商还是整机厂商"。
- `Broadcom components`、`NVIDIA components`、`Samsung Mobile components`、`Samsung LSI` **没有作为表标题出现**在这八页里；`Broadcom` 与 `NVIDIA` 只作为 References 前缀存在（`B-`、`N-`，第七章）。
- 不存在一个"GPU components"或"Firmware components"表——**公告按 IP 供应商分表，不按功能分层**。

### 五之二 表头结构：四种实测签名

把每一张表按其所属 `<h3>` 分组，抽取该组内全部 `<th>`，八页共 59 张表，签名只有四种（下表给出签名、出现的表标题与次数）：

| 列数 | 列头顺序（逐字，页面上是逐个 `<th>`） | 用在哪些表标题 | 实测次数 |
| --- | --- | --- | --- |
| 4 | `CVE`、`References`、`Severity`、`Subcomponent` | `Arm components`、`Dolby`、`Google`、`Imagination Technologies`、`MediaTek components`、`Misc OEM`、`NXP components`、`Qualcomm components`、`Qualcomm closed-source components`、`STMicroelectronics`、`Setup Wizard`、`Thales`、`Tsingteng Micro`、`Unisoc components`、`Widevine DRM` | 3+1+1+4+4+1+1+4+4+1+1+1+1+3+1 = 32 |
| 5 | `CVE`、`References`、`Type`、`Severity`、`Updated AOSP versions` | `Android Runtime`、`Android runtime`、`Framework`、`System`、`TV` | 1+1+6+6+1 = 15 |
| 5 | `CVE`、`References`、`Type`、`Severity`、`Subcomponent` | `Kernel`、`Kernel components` | 4+2 = 6 |
| 2 | `Subcomponent`、`CVE` | `Google Play system updates` | 6 |

（每张表还各自带 `<colgroup>` 宽度；`Common questions and answers` 一节里的 `Abbreviation`、`Definition`、`Prefix`、`Reference` 四列与 `Versions` 一节的 `Version`、`Date`、`Notes` 三列不是缺陷表，不计入。）

**三处结构性差异，每一处都直接影响后续章节的可行性**：

1. **厂商表没有 `Type` 列**——公告在厂商侧不提供缺陷类别，只给一个 `Subcomponent` 词。这条决定了第廿一章的类型学不能从公告反向构建（第卅章把它列入"未公开文档化边界清单"）。
2. **`Kernel` 表有 `Type` 和 `Subcomponent`，但没有版本列**；AOSP 表反过来，有 `Updated AOSP versions` 而没有 `Subcomponent`。所以"哪个 AOSP 分支被打过"这个问题在内核表上无法回答，而"哪个子部件出的事"在框架表上无法回答。
3. **`Google Play system updates` 表只有两列**：`Subcomponent`、`CVE`。它没有 `References`（无法回到 bug ID），没有 `Severity`，也没有版本。第 16 篇讲模块归属，本域讲的是不进这张表的更新单元——但要先知道这张表本身的信息量有多小。

分组逻辑的官方原句出现在八页正文里（2025-09/2026-01/03/04/05/06/09/10，2026-10 页 1 次、其余各 2 次），逐字为：

> `Vulnerabilities are grouped under the component they affect. Issues are described in the tables below and include CVE ID, associated references, type of vulnerability, severity, and updated AOSP versions (where applicable).`

这句话本身也是一处落差：它承诺 "include CVE ID, associated references, type of vulnerability, severity"，而厂商表实际不给 `type of vulnerability`，Mainline 表连 `references` 和 `severity` 都不给。

### 五之三 GPU 侧 Subcomponent 取值：只有两个词

| 表名 | Subcomponent 取值 | 实测行数与严重度 | 样本期 |
| --- | --- | --- | --- |
| `Arm components` | `Mali` | 6 行，全 High（CVE-2026-0001、2026-5729、2026-7476、2026-7477、2026-12285、2026-12387） | 2026-09-01 |
| `Arm components` | `Mali`（另有 `CPU`） | `Mali`：CVE-2025-1246、CVE-2025-3212；`CPU`：CVE-2024-7881 | 2025-09-01 |
| `Imagination Technologies` | `PowerVR-GPU` | 25 行，全 High | 2026-09-01 |

`Mali` 这个词承载的是整个 GPU IP 家族，**不区分 JM 与 CSF、不区分内核驱动与命令流固件**（第十章给出为什么这个区分在本域是致命的）。`PowerVR-GPU` 同理。这是本域词表落差的第一处：公告的一个词 = 现实中的两种架构、三个可分别更新的部件。

### 五之四 媒体与加速器侧 Subcomponent 取值：零散、大小写不一致、含厂商内部代号

跨期实测取值集合，逐字保留大小写：

| 取值 | 出现于 | 样本 | 它实际指什么 |
| --- | --- | --- | --- |
| `hevc decoder` | `MediaTek components` | CVE-2026-20464（`M-ALPS11104718*`） | HEVC 硬件解码引擎 |
| `vdec` | `MediaTek components` | CVE-2026-20501、CVE-2026-20502 **共用一条引用** `M-ALPS11262030*` | 视频解码（厂商内部代号） |
| `apusys` | `MediaTek components` | CVE-2026-20467、CVE-2026-20468（`M-AUTO00837766*`、`M-AUTO00833804*`） | APU（NPU）子系统 |
| `display` / `Display` | `MediaTek components` / `Qualcomm components` | 20473/20474/20475/20477；`Display`：CVE-2026-24085、2026-24089 | 显示通路（同一概念两种大小写） |
| `Video` | `Qualcomm components` | CVE-2025-27042、CVE-2025-27061 | 视频引擎 |
| `DD+ Codec` | `Dolby` | CVE-2025-54957，**Critical**（2026-01 页 Dolby 表） | 音频编解码 |
| `trusted_mem` | `MediaTek components` | CVE-2026-20469 | 受保护内存通路（与编解码的 secure buffer 直接相关） |
| `geniezone` | `MediaTek components` | CVE-2026-20481、CVE-2026-20497 | 厂商自有虚拟化/隔离组件，不属于本文六层中任何一层的公开定义 |
| `Modem` | `MediaTek components`、`Unisoc components` | 多行 | 归第 14 篇处理 |
| `preloader`、`DA` | `MediaTek components` | CVE-2026-20435（2026-06 页，`preloader`）；CVE-2025-20696（2025-09 页，`DA`） | 引导/下载模式，归第 2 篇（引导链）与第 9 篇 |
| `VBMeta` | `Misc OEM` | CVE-2025-48613（2026-03 页，`High`） | AVB 元数据 |
| `StrongBox` | `Tsingteng Micro` | CVE-2025-48651 | 密钥栈，归第 5 篇 |
| `GDMC`、`GSA` | `Pixel` | CVE-2026-56952（Critical）、CVE-2026-0198（High）；CVE-2026-55307（Critical）——三条都在 **Pixel 单独公告** 2026-10 页，不在月度 AOSP 页 | Google 自研侧组件 |
| `Closed-source component` | `Qualcomm closed-source components` | 11 行（第六章） | **占位串，不是器件名** |

三条必须进入方法的观察：

1. **代号不等于对象。** `apusys` 与 `vdec` 是厂商内部代号；把它们与本文六层对齐时只能靠驱动源码（第十六章给出对齐结果），不能靠词面。
2. **同物异写。** `display` 与 `Display` 分属两张表；任何按字符串做资产映射的脚本都必须大小写不敏感（第廿七章脚本 1 在归一层显式处理）。
3. **两张表可以共用一条引用。** CVE-2026-20501 与 CVE-2026-20502 都指向 `M-ALPS11262030*`：**引用号与缺陷不是一一对应**，因此"按引用号统计已修条目数"会低估（第七章）。

### 五之五 `Qualcomm components` 与 `Qualcomm closed-source components` 的分配

2026-09-01 页的分配是一个极端样本：`Qualcomm components` 表 5 行，Subcomponent **全部是 `Security`**；`Qualcomm closed-source components` 表 11 行（含一行 Critical：CVE-2026-25289），Subcomponent 全部是 `Closed-source component`。同一期里 `Video`/`Display` 这类可映射词一次都没出现在高通表中（它们出现在 2025-09 与 2026-06 的页上）。

结论不是"高通只修安全组件"，而是：**同一厂商的两张表，一张给可映射词、一张给占位串；两表之间没有稳定的划分规则可供外部推导**（第六章给出这条结论的可操作性后果）。

## 六、`Closed-source component`：一行读数与它答不了的三个问题

这一节把本域最难处理的一档单独钉住。它的存在不是疏忽，而是制度：厂商不提供细节，Android 侧只能给出一个不含信息的词。

### 六之一 公告对这一档的原句

`Qualcomm closed-source components` 小节的引导句逐字是：

> `These vulnerabilities affect Qualcomm closed-source components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm.`

同名小节也出现在 2025-09-01 页（带 `<h3 id="Qualcomm-closed-source-components">` 锚点），句式一致。与之并列的 `Qualcomm components` 小节只把 `closed-source` 这个词去掉，其余完全相同：

> `These vulnerabilities affect Qualcomm components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm.`

两句话的**唯一差别是"是否闭源"**，而不是"是否可达、是否需交互、位于哪一层"。这就是公告层给这一档缺陷的全部信息量。

### 六之二 11 行读数与"无厂商引用号"这一事实

2026-09-01 页 `Qualcomm closed-source components` 表实测：11 行，Subcomponent 一律为 `Closed-source component`，References 列**只有 `A-` 前缀的 Android bug ID 形式且带星号**，**没有任何 `QC-CR#` 引用号**——这与 `Qualcomm components` 表（`QC-CR#4409501` 等）形成直接对照。

| 读数 | 值 | 后果 |
| --- | --- | --- |
| 能否定位到器件型号 | 否（词表不含型号，占位串不含单元） | 资产台账无法按 SoC 聚合 |
| 能否定位到镜像文件 | 否（无源码路径、无文件名） | 变更管理无法核对"这台设备上的哪个文件被换过" |
| 能否定位到厂商内部跟踪号 | 否（References 无 `QC-` 前缀） | 无法向厂商追问细节，也无法交叉验证 |
| 能否判定严重度来源 | 能，但来源是厂商（原句已给出） | 与 CVSS 自评体系不可直接比较（第 9 篇的三口径分离） |
| 能否进入 SPL 台账 | **不能**（第七章 FAQ 原句） | 已修与未修在 SPL 上不可区分 |

### 六之三 三个必须写进方法的问题

**问一：占位串行算不算"已披露"？** 在本文的口径里算——它在公告表里出现、有 CVE、有严重度，因此**存在性**已披露；但它的**可映射性**为零。第五篇（方法论）给出的"披露深度"分级在这里应当单列一档：存在但不可映射。

**问二：设备侧要不要为它单独建一类读数？** 要。第廿七章脚本把"公告说这是闭源组件缺陷"与"设备说这个镜像存在"两条证据分置于两个信号码，禁止合并成一个"已修复"结论。

**问三：能不能用"闭源"直接推"不可更新"？** 不能，且这条推理会同时导出两个错误结论——它会掩盖 `ro.vulkan.apex` 这类真实存在的可更新面（第三之五节），也会掩盖"厂商把修复装进 OTA 但不进公告"这一常态。第廿四章按三条到达通道分别处理。

## 七、References 前缀与星号：编号层给了归属，不给对象

### 七之一 官方定义句与前缀表

公告 FAQ 的提问与回答逐字是：

> `4. What do the entries in the References column mean?`
> `Entries under the References column of the vulnerability details table may contain a prefix identifying the organization to which the reference value belongs.`

实测前缀表逐字：`A-` = Android bug ID，`QC-` = Qualcomm reference number，`M-` = MediaTek reference number，`N-` = NVIDIA reference number，`B-` = Broadcom reference number，`U-` = UNISOC reference number。

注意这一句的措辞边界：前缀标识的是**引用值属于哪个组织**，不是缺陷属于哪个器件、更不是属于哪一层。用前缀做资产映射，只能得到"该找谁问"，得不到"该换哪个文件"。

### 七之二 星号的真实含义，以及它为什么比"闭源"更重要

> `5. What does an * next to the Android bug ID in the References column mean?`
> `Issues that are not publicly available have an * next to the corresponding reference ID. The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.`

这句原句同时给出三件事，值得逐条拆开：

1. **星号 = 对应条目不公开**（不是"缺陷不严重"，也不是"不影响到你"）。第五至六章的实测里，`M-ALPS11104718*`、`M-AUTO00837766*`、`A-466377983*` 都带星号，覆盖了媒体与 GPU 两侧。
2. **修复的载体被明确说成"binary drivers"**，且获取点是"Google Developer site"——这是本域少见的**官方对'修复以二进制形式交付'的正面表述**。它同时解释了为什么设备侧的证据形态往往是"某个 so/镜像文件变了"而不是"某个源码提交合进了某分支"。
3. **`generally` 是限定词。** 原句没有说"总是"，也没有说非 Pixel 设备如何。把这句读成"所有闭源修复都在 OTA 里"超出了原文范围（第卅章未文档化边界清单第 3 条）。

### 七之三 公告与厂商公告的分工：一句话决定了 SPL 台账的适用范围

> `6. Why are security vulnerabilities split between this bulletin and device / partner security bulletins, such as the Pixel bulletin?`
> `Security vulnerabilities that are documented in this security bulletin are required to declare the latest security patch level on Android devices. Additional security vulnerabilities that are documented in the device / partner security bulletins are not required for declaring a security patch level.`

这句是第 L-3 条结论的来源，也是本域的台账规则：**厂商公告里的缺陷不要求、也不触发 SPL 变化**。而厂商公告（第九章实测）正是媒体与加速器缺陷的主要披露地。因此"SPL 已达标"对本域几乎没有覆盖力——第廿七章的 K1/K4 指标与脚本判定模型都据此设计。

一句需要防止的误读：这条**不是**"厂商公告里的缺陷不重要"，而是"它不在 Android 的补丁级别语义里"。重要性由厂商按 CVSS 自评（MediaTek 原句：`The severity of the identified vulnerabilities was conducted based on the Common Vulnerability Scoring System version 3.1 (CVSS v3.1).`，第九章）。两套口径的分离方式在第 9 篇已给出，本文直接引用其结论。
## 八、SPL 与补充补丁清单：器件修复唯一被官方记账的地方

第 L-3 条结论已经给出规则本身：厂商公告里的缺陷不作为声明补丁级别的条件。本章把这条规则落到"一段镜像被换掉"这个具体动作上，并给出本域唯一一条能把修复读出来的通道。

### 八之一 SPL 侧的原句与其覆盖边界

月度公告页的补丁级别说明逐字是：

> `Security patch levels of 2026-10-01 or later address all issues associated with the 2026-10-01 security patch level.`
> `Device manufacturers that include these updates should set the patch string level to:`
> `[ro.build.version.security_patch]:[2026-10-01]`

两档并存的理由逐字是：

> `2. Why does this bulletin have two security patch levels?`
> `This bulletin has two security patch levels so that Android partners have the flexibility to fix a subset of vulnerabilities that are similar across all Android devices more quickly.`

把第七章那条"厂商公告缺陷不要求声明 SPL"叠在这两句上，就得到本域的台账规则：**`ro.build.version.security_patch` 是一个关于 Android 侧修复的声明，不是一个关于器件镜像的声明。**

### 八之二 三种错位场景，逐条给出判定动作

| 场景 | 台账表现 | 真实状态 | 本域的判定动作 |
| --- | --- | --- | --- |
| A　SPL 已更新，器件镜像未更新 | "本月已修" | GPU 微码/编解码器固件仍是旧件 | 不得记为已修；按第廿六章降为 L2-WARN 并要求补采（K2） |
| B　SPL 未更新，器件镜像已更新 | "落后一月" | 厂商已在闭源件里带上了修复 | 唯一可能的正证是补充补丁清单（八之三）；否则记 L1-INFO 并标"不可归因" |
| C　两侧都更新，但设备读数缺失 | "已修" | 无法自证 | 记录"证据缺失"而非"证据良好"；这是第 15 篇同款问题在本域的重演 |

场景 B 值得多说一句：第廿六章会把"读不到"与"读出来是坏的"严格分开，因为**本域大多数设备的真实状态恰好落在"修复已到、但没有任何可读接口能证明它到了"这一格**（第十四、十七章给出三个源码级例证）。

### 八之三 补充补丁清单：本域唯一一条"器件修复进入官方记账"的通道

Android 为"两个公告周期之间已经修好"准备的机制，原句是：

> `With Android's shift toward continuous release and quarterly bulletins as part of risk-based security updates, OEMs may want to fix vulnerabilities between releases rather than waiting an entire quarter. The supplemental security patch XML feature lets OEMs report CVEs patched beyond the declared security patch level (SPL) by providing a standardized XML file, so OEMs receive credit for continuous patching.`

放置位置同样由官方逐字给出，且**明确覆盖硬件相关组件**：

> `OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product), ensuring that security patches from both the framework and hardware-specific components are accounted for. The XML file should be installed in /etc/security/supplemental_security_patches.xml for the respective partition.`
> `KEY_VENDOR_SUPPLEMENTAL_PATCHES: Aggregates patches from /vendor and /odm partition paths.`

这一段在本域有三层含义，第廿七章脚本的信号码就是按它设计的：

1. **`/vendor` 档是为器件修复准备的。** 它是官方语义里唯一承认"器件侧修复需要单独记账"的地方，且 `KEY_VENDOR_SUPPLEMENTAL_PATCHES` 明确覆盖 `/vendor` 与 `/odm`。
2. **它记的是 CVE 号，不是镜像版本。** 因此即便文件存在，它也回答不了"这台设备上的 `mali_csffw.bin` 是哪一个版本"，只回答"哪些 CVE 已被声明修好"。这是本域"记账粒度"与"证据粒度"错位的正式形态。
3. **它是自愿项、且可读。** 平台策略对 vendor 配置目录向所有域开放——`allow domain vendor_configs_file:dir r_dir_perms;` 与 `allow domain vendor_configs_file:file { read open getattr map };`（`domain.te:177-178`），标签来自 `/(vendor|system/vendor)/etc(/.*)?              u:object_r:vendor_configs_file:s0`（`file_contexts:427`）。所以 `/vendor/etc/security/supplemental_security_patches.xml` 是**少数几个 adb shell 能直接读到的器件侧安全文件之一**；反过来，"读不到这个文件"本身是一条有信息量的事实（脚本给出 `suppl_xml_absent` 与 `suppl_xml_unreadable` 两个不同信号码，前者指 OEM 未启用，后者指策略或挂载导致不可读，处置动作不同）。

### 八之四 一条不要跨过去的线

补充补丁清单是 **OEM 的自我声明**。第 9 篇（补丁传导链）已给出"声明与到达是两件事"的通用口径；本域的额外风险是：一张列着 GPU 相关 CVE 的 XML 与"GPU 固件确实是新版"之间仍然隔着整个 F1/F2 层——**XML 无法被设备侧独立验证，因为镜像版本根本没有可读接口**。因此本文的判定模型禁止把"XML 里有这条 CVE"升为 L3-HIGH 的反证（即禁止据此判定"已修"），只允许它作为 L1-INFO 的加分项。

## 九、厂商公告入口的可见性实测：四类入口，只有一类能匿名读全文

本域的缺陷多数只在厂商公告里出现。本章记录"厂商公告入口"在 2026-10-29 的实测可读性——这不是修辞，而是取证前提：**如果一条修复只能在登录后读到，它就无法进入任何自动化台账。**

### 九之一 五类入口逐条实测

| 入口 | 实测 URL 形态 | 匿名可读性 | 粒度 | 是否给镜像/固件版本 |
| --- | --- | --- | --- | --- |
| **MediaTek** | `https://www.mediatek.com/product-security-bulletin`，逐月形如 `/product-security-bulletin/october-2026?hsLang=en` | **可**（HTTP 200，正文可取） | 芯片 + 缺陷 + CVSS | 否 |
| **Qualcomm** | `www.qualcomm.com/company/product-security`；`docs.qualcomm.com/<任意路径>`；`docs.qualcomm.com/bundle/publicresource/FP80-43795-14V_Whitepaper.pdf` | **否**——www 侧 HTTP 200 但正文只有 SPA 外壳（本次 10,099 字节）；docs 侧任意路径 **302** → 登录页 `account.qualcomm.com - Sign In`（200，149,716/149,819 字节随路径略变）；白皮书那条 **302 → 403**，正文 70 字节恰为 `{"location":"https://myid.qualcomm.com/api/v2/login?state=Direct URL"}`。**同一路径的壳会变**：本项目早前在同一 docs 路径测到的是 52,406 字节 `<title>Qualcomm Documentation</title>` 壳，本次已不存在 ⇒ 复采必须逐次记录响应尺寸与正文标题（第卅章复采清单） | 未知（取不到） | 未知（取不到） |
| **Arm** | `https://developer.arm.com/arm-product-security-center`（HTTP 200，正文可取）；单篇 Mali 公告 `documentation/110627/latest` → 302 → `support.arm.com/documentation/110627/latest` | 入口页**可**，单篇**否**（返回需二次验证配置的文档应用壳） | 产品级 | 否 |
| **Imagination** | `https://www.imaginationtech.com/gpu-driver-vulnerabilities/`（HTTP 200，可匿名）；`/security/`、`/psirt/`、`/security-advisories/`、`/trust-center/` 均 **404** | **可** | **DDK 驱动版本** | 否（只给驱动版本区间） |
| **Samsung 移动** | `https://security.samsungmobile.com/workScope.smsb`、`/securityUpdate.smsb` | **否**——HTTP 403，正文恰为 `<h1>403 Forbidden</h1>` + `nginx`（反爬特征，非登录墙） | 未知 | 未知 |
| **Android 侧 CVE 检索门户** | `https://security.android.goog/` | 本次**不可达**：经代理 TLS 握手失败（`curl: (35)`），直连 DNS 无法解析（`curl: (6)`） | — | — |

最后一行必须原样保留。本报告不宣称该门户不可用，只记录"在本采集环境下未取回"，并把它的取回动作写进第卅章的复采清单。

### 九之二 三家可读取入口的自述原句

**MediaTek**（月度公告正文，逐字）：

> `MediaTek Product Security Bulletin contains details of security vulnerabilities affecting certain MediaTek chipsets. Device OEMs have been notified of all the issues and the corresponding security patches for at least two months before publication. … The severity of the identified vulnerabilities was conducted based on the Common Vulnerability Scoring System version 3.1 (CVSS v3.1).`

这一句在本域的信息量比看上去大：**"至少两个月"是公开披露与 OEM 知情之间的制度性时间差**。第廿七章的 K4 指标（披露滞后天数）就以此为基线，而不是以"公告日期=修复日期"为基线。

**Arm**（产品安全中心页，逐字给出三类文档的分工）：

> `Arm publishes three types of security-related documents, each with a different purpose:`
> `Arm Security Bulletins — Arm publishes Security Bulletins when a vulnerability affects an Arm product. These bulletins include technical details, impact assessments, and guidance on mitigation or remediation.`
> `Arm Security Updates — Arm issues Security Updates when a security issue does not affect Arm products, but where Arm technology may be referenced or considered relevant. These updates provide clarity and context for Arm partners.`
> `Arm Security Articles — Arm publishes Security Articles to provide in-depth analysis, commentary, and insight into emerging security topics…`

对本域要紧的是 **Security Update 这一档**：它明确覆盖"不发生在 Arm 产品上、但与 Arm 技术相关"的问题。当一条 Mali 相关缺陷在 Android 公告里以 `Mali` 为 Subcomponent 出现、而在 Arm 侧只以 Update（而非 Bulletin）形式出现时，**两侧的"是否算一个已披露缺陷"并不等价**——这是第廿二章案例读数里反复出现的口径差。

**Imagination**（GPU 驱动漏洞页，逐字）：

> `This page contains summary details of security vulnerabilities reported on Imagination Technologies graphics drivers.`

页面自称覆盖的是 **graphics drivers**，且样本行的字段是 `Versions affected` + `DDK Releases up to and including 25.2 RTM`。同一行的其余字段逐字保留：`Our Reference PSP-19`、`CVE Reference CVE-2025- 25176`、`Originator Reference PP-171174`、`Date Posted 9th January 2026`。

这一家恰好示范了本域最难对齐的一件事：**披露方按 DDK 版本给受影响范围，而 Android 公告按 `PowerVR-GPU` 一个词给 Subcomponent，设备侧则既读不到 DDK 版本也读不到固件版本**（第十三章给出设备侧唯一可能的替代读数）。三方之间没有任何一个共同主键。

### 九之三 取回失败本身作为一条方法结论

四类入口里三家取不到正文，且**取不到的机制各不相同**：SPA 外壳（Qualcomm www）、登录重定向壳（Qualcomm docs，302 → `account.qualcomm.com - Sign In`）、二次验证应用壳（Arm 单篇）、反爬 403（Samsung）。这个差别对本域有直接后果：

- **不能用"访问被拒"推断"厂商未修复"**——四种拒因都与修复状态无关。
- **不能把可匿名读取的入口当作覆盖度基线**——只有 MediaTek 与 Imagination 可匿名，覆盖率结论会系统性偏向这两家；第廿二章的案例分布明确标注了这一取样偏差。
- **自动化台账必须把"入口不可读"作为一等状态**。第廿七章脚本 1 对每一个公告侧断言都保留 `not_reachable` 与 `not_applicable` 两个不同码位，禁止合并。

### 九之四 一句总结本章与第 L-2 条的关系

公告侧把严重度评定交给厂商（第六章原句），厂商侧的入口有一半不可匿名读取（本章）。因此**本域的严重度是一个"来自不透明来源的外部声明"**，它仍然可用，但不能与 AOSP 侧的严重度直接混算。第 9 篇对框架/厂商两侧口径的处理方式在此原样适用，本文不重复展开。
## 十、JM 与 CSF：一条把 Mali 缺陷劈成两种归属形态的分界线

第 L-1 条结论说"本域的第一道归属判断不是型号，而是执行模型"。Mali 侧这条执行模型分界叫 **JM / CSF**（Job Manager 与 Command Stream Frontend），它是本文能给出的最干净的一个例子：**同一家厂商、同一代架构命名、同一个产品字号，只因为分界线两侧，缺陷的归属对象、修复载体和设备侧的证据形态完全不同。**

本章证据的出处统一为 `android.googlesource.com/kernel/common`，分支 `android16-6.12`（下列行号均按该分支）；凡与上游 `torvalds/linux` 标签 `v7.3` 不同的地方，都单独标出——**两棵树在同一件事上给出不同代码，这件事本身就是本域的一条方法结论**（见十之四、十一之三）。

### 十之一 分界线的公开定义句

Arm 在驱动配置里把这句话说得非常明确。`drivers/gpu/drm/panthor/Kconfig` 帮助文本逐字是：

> `DRM driver for ARM Mali CSF-based GPUs.`
> `This driver is for Mali (or Immortalis) Valhall Gxxx GPUs.`
> `Note that the Mali-G68 and Mali-G78, while Valhall architecture, will`
> `be supported with the panfrost driver as they are not CSF GPUs.`
>
> — `drivers/gpu/drm/panthor/Kconfig:18-23`，`kernel/common@android16-6.12`

三个信息点按原句顺序落在这里：驱动按 **CSF 与否** 划分而不是按架构代数划分；Valhall 是架构名不是驱动名；**G68 / G78 这两颗 Valhall 器件被明确排除在 CSF 之外**。最后这句是本报告反复使用的方法示例——**设备台账里写着"Mali-G78"的时候，它没有告诉你该按哪一栏归属；而内核源码里这句注释告诉你。**

分界线在设备树侧也有一处公开表述。`Documentation/devicetree/bindings/gpu/arm,mali-valhall-csf.yaml:22`（同一分支）逐字是：

> `- const: arm,mali-valhall-csf   # Mali Valhall GPU model/revision is fully discoverable`

这句注释是**硬件侧的可发现性声明**：型号与修订号可由寄存器读出。它和第四章那条"器件型号可读、器件镜像版本不可读"叠在一起，就是本域最常见的证据形状——**知道是哪颗 GPU，不知道它上面跑的那段镜像是什么版本。**

### 十之二 一个可复现的检索证据：JM 侧的固件命中数是零

分界线最硬的一条证据不需要读文档，只需要在源码树上做一次目录级检索。方法完整、可复现：取 `kernel/common@android16-6.12` 的 `drivers/gpu/drm/{panfrost,panthor}/` 目录清单（Gitiles 的 `?format=JSON` 返回 `entries`，逐条取 `name`），按清单把每个文件原样取回（同一 API 的 `?format=TEXT` 加 base64 解码），再对全部 `.c` / `.h` 做同一个大小写不敏感的 `grep firmware`。

| 目录 | 源文件数 | `firmware` 命中数 |
| --- | --- | --- |
| `drivers/gpu/drm/panfrost/`（JM 侧） | 21 | **0** |
| `drivers/gpu/drm/panthor/`（CSF 侧） | 18 | **40** |

跨树复现（上游 `torvalds/linux@v7.3`，同一方法）：

| 目录 | 源文件数 | `firmware` 命中数 |
| --- | --- | --- |
| `drivers/gpu/drm/panfrost/` | 21（目录含 Kconfig/Makefile/TODO 共 24 项） | **0** |
| `drivers/gpu/drm/panthor/` | 24（目录共 29 项） | **54** |

两张表说的是同一件事，且**JM 侧在两棵树里都是零**。请把它读准：**JM 侧驱动里一个"固件"字都没有出现**（对 `request_firmware` 与字面量 `.bin` 的检索同样为 0 命中）。JM 形态的 Mali 由 CPU 侧驱动直接写寄存器、排度命令流，没有一段运行在 GPU 内部处理器上的镜像，因此也**不存在"这段镜像的缺陷归谁、版本怎么读、什么时候换的"这三个问题**。CSF 侧相反：加载、校验、接口协商、MCU 启停全在这套代码里，第十一章逐条展开。

两条防止误读的边界声明：

1. 这是**源码树检索**结果，不是一台设备上的读数。设备是否在用这份驱动，属于第十九、二十章的 V 面问题。
2. 命中数为零不等于 JM 侧 GPU 没有安全缺陷。JM 形态的缺陷确实存在，只是**它落在 GPU 时钟、电源、MMU、作业调度的内核代码里**，那部分归属由第 1 篇（内核驱动攻击面）处理，本文不重复计入。这条就是第 1 篇与本文的分工线在 Mali 上的具体落点。

### 十之三 两侧识别型号的代码形状，以及各自"不认识时怎么办"

同一件事——"把片上 ID 翻译成一颗可归属的 GPU"——在两个驱动里是两个不同的代码形状。逐项对照（`kernel/common@android16-6.12`）：

| 环节 | JM 侧 `panfrost` | CSF 侧 `panthor` |
| --- | --- | --- |
| 读 ID | `gpu_id = gpu_read(pfdev, GPU_ID);`　`panfrost_gpu.c:286` | `ptdev->gpu_info.gpu_id = gpu_read(ptdev, GPU_ID);`　`panthor_gpu.c:90` |
| 额外 ID | — | `ptdev->gpu_info.csf_id = gpu_read(ptdev, GPU_CSF_ID);`　`:91`；`ptdev->gpu_info.gpu_rev = gpu_read(ptdev, GPU_REVID);`　`:92` |
| 拆字段 | `pfdev->features.revision = gpu_id & 0xffff;` / `pfdev->features.id = gpu_id >> 16;`　`panfrost_gpu.c:287-288` | 位段宏在头文件里成组定义：`#define GPU_ID 0x0` / `GPU_ARCH_MAJOR(x) ((x) >> 28)` / `GPU_ARCH_MINOR(x)` / `GPU_ARCH_REV(x)` / `GPU_PROD_MAJOR(x)` / `GPU_VER_MAJOR(x)` / `GPU_VER_MINOR(x)` / `GPU_VER_STATUS(x)`　`panthor_regs.h:13-20`，取值在 `panthor_gpu.c:117-121` |
| 匹配依据 | 型号 id 相等：`if (!panfrost_model_eq(pfdev, model->id)) continue;`　`panfrost_gpu.c:306` | 架构大版本 + 产品大版本两者相等：`if (model->arch_major == arch_major && model->product_major == product_major) break;`　`panthor_gpu.c:124-125` |
| 型号表长度 | `gpu_models[]` 逐型号列出（含注释见十之四） | `static const struct panthor_model gpu_models[] = { GPU_MODEL(g610, 10, 7), {}, };`　`panthor_gpu.c:72-75` —— **整张表只有一个型号名** |
| 不认识时 | `const char *name = "unknown";`　`:244`，循环不中则沿用该值继续运行 | `model->name ?: "unknown"`　`:131`，同样继续运行 |
| 成功与失败都打的日志 | `dev_info(pfdev->dev, "mali-%s id 0x%x major 0x%x minor 0x%x status 0x%x", name, gpu_id, major, minor, status);`　`:329-330` | `drm_info(&ptdev->base, "mali-%s id 0x%x major 0x%x minor 0x%x status 0x%x", model->name ?: "unknown", ptdev->gpu_info.gpu_id >> 16, major, minor, status);`　`:129-132` |

四点值得单独指出来，因为它们直接变成运营动作：

- **两个驱动都不因为"型号不认识"而拒绝设备**（该分支下）。它们只是把名字写成 `unknown` 继续跑。所以台账里出现"GPU 正常工作但型号名是 unknown"是**预期行为**，不提供任何固件信息。
- **`android16-6.12` 的 CSF 型号表只有 `g610` 一项。** 也就是说这份驱动能驱动 G710/G510/G310 等 CSF GPU，但打印出的名字是 `unknown`。**"驱动支持该器件"与"器件名可读"是两件事**，第廿五章的矩阵必须把它们分成两列。
- **两侧最详细的身份信息都只进内核日志。** 最后两行是同一件事的两种写法：型号、id、major/minor/status 全部以 `dev_info` / `drm_info` 输出。内核日志对 shell 域不可读（第十九章 V6 面），所以这段**由硬件寄存器直接得到、本来最可靠的身份信息，在标准核验姿势下拿不到**。
- **CSF 侧多出的两个 ID 是硬件身份，不是镜像身份。** `GPU_CSF_ID`（`panthor_regs.h:35`，偏移 `0x1C`）与 `GPU_REVID`（`:130`，偏移 `0x280`）都是寄存器。第廿九章第 5 个误区就是把这类硬件 revision 记成固件版本。

跨树差异（必须记进台账的一条）：上游 `v7.3` 的 JM 侧**会**因为型号表查不到而失败——`dev_err(pfdev->base.dev, "GPU model not found: mali-%s id rev %#x %#x\n", ...)` 后 `return -ENODEV;`（`panfrost_gpu.c:354-356`）；CSF 侧改为按架构区间匹配（`panthor_hw.c:49-57` 的 `arch_min` / `arch_max`，注释逐字 `Minimum supported architecture major value (inclusive)`；不匹配时 `if (!hdev) return -EOPNOTSUPP;`　`:288-289`；名字查不到时 `return "(Unknown Mali GPU)";`　`:173`）。**同一颗器件在两份代码里的"能不能起、叫什么名"是不同的**，因此任何跨设备型号比较都必须带内核树的版本维度。

### 十之四 型号表里的一句自述：厂商下游 hack 是被写进注释的

`panfrost` 的型号表里有一条注释值得整段引用，因为它是**内核作者对"厂商实现不一致"的正面书面记录**（`panfrost_gpu.c:232-238`，`android16-6.12`）：

> `/* MediaTek MT8192 has a Mali-G57 with a different GPU ID from the`
> ` * standard. Arm's driver does not appear to handle this model.`
> ` * ChromeOS has a hack downstream for it. Treat it as equivalent to`
> ` * standard Mali-G57 for now.`
> ` */`
> `GPU_MODEL(g57, 0x9003,`
> `    GPU_REV(g57, 0, 0)),`

三条归属含义：**同一颗 Arm IP 在不同厂商 SoC 上的 GPU ID 可以不同**；Arm 的公开驱动**当时并不处理**这个 ID；厂商侧存在**下游 hack**。这正是第 3 篇（硬件组件形态差异）与第 10 篇（补丁传导链）反复给出的结构，在本域的一个具体落点。方法论上它给出一个可操作提示：**当某台设备的 GPU ID 不在公开型号表里时，正确记录是"该设备走的是厂商私有分支"，而不是"读数异常"**——脚本 1 对 GPU 段的第一个字段就是"驱动实现是否为公开树那份"。

### 十之五 分界线之上的三条归属推论

1. **一份公告的 `Subcomponent` 写着 `Mali` 时，它至少少给了一个必要条件。** 归属必须再问一次 JM 还是 CSF。JM 侧的答案落在内核驱动与用户态 GL 栈；CSF 侧的答案可能落在 `mali_csffw.bin` 本体——那是第 4 章列为 **F1 固件本体** 的东西，没有源码提交号、没有 AOSP 公告行、不进入 SPL 语义（第八章）。
2. **CSF 侧的缺陷面天然分成两层，且这两层可见性不同。** 一层是镜像本体的缺陷（不可见，只能靠厂商公告或补充补丁清单），另一层是**驱动与镜像之间的接口约定**（第十一章的 `CSF_IFACE_VERSION` 能力协商），后者的证据形式是功能有无，而不是版本高低。第廿一章的类型学把这两层分别落在 FT1（镜像本体逻辑）和 FT6（接口/版本协商不一致）上。
3. **归属表按型号建就必须带"实现"维度。** 本节给出的三处实例都是同一形状的三种版本：G68/G78 归 panfrost（Kconfig 原句）、MT8192 的 G57 走私有 ID（型号表注释）、`android16-6.12` 与 `v7.3` 对未知型号一个容忍一个失败。第廿五章矩阵因此把"实现树"作为独立一列，否则它会在跨代设备上给出错误的一致结论。

## 十一、Mali CSF 固件的加载、自报版本与静默降级

本章把 CSF 侧从"文件被取入"到"能力协商完成"这条路径逐段落到源码，并对每一段回答同一个问题：**这一段出了错，标准核验姿势能看到什么。** 出处同第十章，`kernel/common@android16-6.12` 的 `drivers/gpu/drm/panthor/`。

### 十一之一 固件文件名由片上寄存器决定

加载动作在 `panthor_fw_load()`（`panthor_fw.c:695-712`）：

> `#define CSF_FW_NAME "mali_csffw.bin"`　`:27`
> `snprintf(fw_path, sizeof(fw_path), "arm/mali/arch%d.%d/%s",`
> `     (u32)GPU_ARCH_MAJOR(ptdev->gpu_info.gpu_id),`
> `     (u32)GPU_ARCH_MINOR(ptdev->gpu_info.gpu_id),`
> `     CSF_FW_NAME);`　`:703-706`
> `ret = request_firmware(&fw, fw_path, ptdev->base.dev);`
> `if (ret) {`
> `    drm_err(&ptdev->base, "Failed to load firmware image '%s'\n",`
> `        CSF_FW_NAME);`
> `    return ret;`
> `}`　`:708-712`

三点必须看清：

- **路径里的 `arch%d.%d` 来自 GPU_ID 寄存器**（宏在 `panthor_regs.h:14-15`，`gpu_id` 在 `panthor_gpu.c:90` 读入），不是设备树里写死的固件名。也就是说：**器件自己决定去哪找镜像，固件目录不由厂商的配置文档决定。**
- 文件名字面是 `mali_csffw.bin`，目录前缀是 `arm/mali/`。这是 `firmware_class` 的搜索路径拼接结果，因此**能否列出这个目录**由 sepolicy 决定（第十九章 V1 面），而不是由文件在不在决定。第二章第三条定义句里的那行 `firmware_class.path=/vendor,/firmware_mnt` 正是在这条路径上生效。
- 驱动用 `MODULE_FIRMWARE` 把自己期望的镜像写进模块元数据，但该分支只有一条：

> `MODULE_FIRMWARE("arm/mali/arch10.8/mali_csffw.bin");`　`:1371`

上游 `v7.3` 这里已经是**连续七条**（`panthor_fw.c:1553-1559`，arch 10.8 / 10.10 / 10.12 / 11.8 / 12.8 / 13.8 / 14.8）。**这就是"公开静态基线"在本域不可靠的实证**：同一个"驱动声明需要哪些固件"的问题，答案取决于你读哪棵树，而且新树会随架构代际增长。因此脚本 1 对这块的记录方式是**分别记录**"当前设备的驱动里 `MODULE_FIRMWARE` 有几条"与"设备分区里能看到哪个目录"，绝不合成一个布尔；第廿五章的矩阵里这一格永远允许"不可比对"。

### 十一之二 加载与校验段的每一条失败串，以及它变成设备证据的条件

按代码顺序列出该分支 `panthor_fw.c` 里全部与镜像有关的消息（原样，不改写）。这张表的用处不是吓人，而是**给出一个可 grep 的清单**：设备的内核日志里若出现其中任何一条，本域判定立刻从"未知"变成"这块器件当前不可用"——问题在于这些串的载体（kmsg / pstore）在标准姿势下不可读，所以它属于**有厂商日志通道时的第二层证据**（第廿六章 K7 的适用条件）。

| 源码位置 | 消息（逐字） | 返回 | 归属含义 |
| --- | --- | --- | --- |
| `:368` | `Firmware too small` | `-EINVAL` | 读取越界 |
| `:385` | `Firmware entry too long` | `-EINVAL` | 条目长度越界 |
| `:502` | `Firmware corrupted, data.end < data.start (0x%x < 0x%x)` | `-EINVAL` | **段内自相矛盾** |
| `:508` | `Firmware corrupted, hdr.va.end < hdr.va.start (0x%x < 0x%x)` | `-EINVAL` | 虚拟地址区间倒置 |
| `:514` | `Firmware corrupted, file truncated? data_end=0x%x > fw size=0x%zx` | `-EINVAL` | 镜像被截断 |
| `:520` | `Firmware corrupted, virtual addresses not page aligned: 0x%x-0x%x` | `-EINVAL` | 对齐不符 |
| `:526` | `Firmware contains interface with unsupported flags (0x%x)` | `-EINVAL` | **接口标志位不被驱动理解** |
| `:532-533` | `Firmware protected mode entry not be supported, ignoring` | 0（继续） | 保护模式条目被**忽略而非拒绝**（见下） |
| `:539-540` | `Interface at 0x%llx must be shared` | `-EINVAL` | MCU 共享区段标志缺失 |
| `:663` | `Firmware entry isn't 32 bit aligned, offset=0x%x size=0x%x` | `-EINVAL` | 条目对齐 |
| `:689-691` | `Unsupported non-optional entry type %u in firmware` | `-EINVAL` | **镜像与驱动预期结构不一致** |
| `:710-711` | `Failed to load firmware image '%s'` | 透传 `request_firmware` 返回值 | 文件缺失 / 不可读 / 搜索路径不通 |
| `:723` | `Invalid firmware magic` | `-EINVAL` | 不是 CSF 二进制 |
| `:729-730` | `Unsupported firmware binary header version %d.%d (expected %d.x)` | `-EINVAL` | **头部大版本不被支持（格式版本，不是安全版本）** |
| `:735` | `Firmware image is truncated` | `-EINVAL` | 头声明长度大于实际 |
| `:748` | `Shared interface region not found` | `-EINVAL` | 没有共享接口段 |
| `:799` | `Invalid stream control interface input/output VA` | `-EINVAL` | 接口地址越界 |
| `:805` | `Expecting identical CS slots` | `-EINVAL` | 接口结构与预期不符 |
| `:854` | `Invalid group control interface input/output VA` | `-EINVAL` | 同上 |
| `:863` | `Expecting identical CSG slots` | `-EINVAL` | 同上 |
| `:900` | `Firmware version is 0. Firmware may have failed to boot` | `-EINVAL` | **本表最重要的一行**，见十一之三 |
| `:907` | `Invalid global control interface input/output VA` | `-EINVAL` | 全局接口地址越界 |
| `:913` | `Invalid number of control groups` | `-EINVAL` | `group_num` 越出 `MIN_CSGS`/`MAX_CSGS` |

三条判读纪律：

1. **这张表里没有"版本过低"这一条。** 驱动检查的是长度、对齐、地址区间、标志位、入口类型和格式大版本，**不检查"这段镜像安不安全、够不够新"**。这正是本章与第十二章的分水岭：Adreno 侧存在一个明确的驱动侧最低微码版本门（第十二章之三），Mali CSF 侧**没有对应物**。把这个"没有"读成"Arm 不关心安全"是第廿九章第 4 个误区；正确读法是：**在这条实现里，"旧镜像是否可用"根本不由驱动决定，因此它不可能作为设备侧的自证证据出现。**
2. **`:532-533` 是全表唯一一条"看见不支持的东西但继续跑"的分支**，且它的内容恰好是 protected-mode 条目。这一条要单独记住：它意味着**一份带保护模式段点的镜像在公开驱动上会被静默降级加载**，而 protected mode 正是本域用于隔离敏感媒体路径的机制（第 8 篇讨论其信任根形态，本文不重复）。原句的英文语法本身就是源码里的样子，本文照抄不改。
3. **`:729-730` 容易被台账误读成"固件版本校验"。** 它校验的是 **binary header 格式大版本**（`CSF_FW_BINARY_HEADER_MAJOR_MAX`），不是安全版本；把它记成"驱动会拒绝旧固件"是错的。

### 十一之三 版本号的三个来源，其中两个不能用于安全判断

CSF 侧能被读成"版本"的东西有三件，必须严格分开：

| 读数 | 来源 | 载体 | 能否用于安全归属 |
| --- | --- | --- | --- |
| **接口版本** | 固件在运行时写入共享内存：`struct panthor_fw_global_control_iface { u32 version; u32 features; u32 input_va; u32 output_va; u32 group_num; u32 group_stride; u32 perfcnt_size; u32 instr_features; };`　`panthor_fw.h:193-202` | `drm_info(&ptdev->base, "CSF FW v%d.%d.%d, Features %#x Instrumentation features %#x", CSF_IFACE_VERSION_MAJOR(...), ..., glb_iface->control->features, panthor_get_instr_features(ptdev));`　`panthor_fw.c:924-929` | **不能**。它是固件自报的运行时能力声明，不是镜像文件属性；编码在 `panthor_fw.c:192-196`（`(((major) << 24) \| ((minor) << 16) \| (patch))`） |
| **是否起来过** | 同一 `version` 字段是否非零 | `if (!glb_iface->control->version) { drm_err(&ptdev->base, "Firmware version is 0. Firmware may have failed to boot"); return -EINVAL; }`　`:899-902` | **能，但只证明反面**：非零说明固件跑起来了；零说明它没起来 |
| **构建身份（git sha）** | **该分支不存在这条读数**；上游 `v7.3` 才加入：`static const char git_sha_header[] = "git_sha: ";`　`panthor_fw.c:701`，成功时 `drm_info(&ptdev->base, "Firmware git sha: %s\n", ...)`　`:729-730` | 内核日志 | 理论上最精确（直指构建提交），实际不可用：只进 kmsg；且上游把元数据损坏处理成非致命——`drm_err(..., "Firmware build info corrupt")` 后注释逐字 `/* We don't need the build info, so continue */`（`v7.3` `:712-714`） |

前两行的结论性表述：**接口版本号是固件自报的运行时能力声明，不是镜像的构建身份。** 它由固件写进共享内存、驱动从 `kmap` 取（`:897`），因此：不会告诉你镜像是哪一版，只告诉你它愿意按哪一版接口说话；它出现在内核日志里，标准姿势读不到；它为零是**唯一一种能被读成"这块器件当前不可用"的强信号**，而这条信号的载体依然是 kmsg。

第三行是本章最有价值的一条跨树观察：**"固件能不能被溯源到一次构建"这件事，在两棵树里的答案不同**，而且较新的一侧才加入、加入后仍然是"读不到就跳过"。第廿九章第 2 个误区（"有 git sha 就能核对固件是否已修"）就是为这类误读准备的。

### 十一之四 静默降级：能力协商不成，日志里什么都没有

这是 CSF 侧最容易漏掉的一类工程事实。驱动对接口版本的处理是**能力探测**，不是版本门：

> `static u32 panthor_get_instr_features(struct panthor_device *ptdev)`
> `{`
> `    struct panthor_fw_global_iface *glb_iface = panthor_fw_get_glb_iface(ptdev);`
>
> `    if (glb_iface->control->version < CSF_IFACE_VERSION(1, 1, 0))`
> `        return 0;`
>
> `    return glb_iface->control->instr_features;`
> `}`　`:878-886`

`CSF_IFACE_VERSION(1, 1, 0)` 是**全树唯一一次**与接口版本比较（`:192-196` 定义宏、`:882` 使用；`:925-927` 只是把三段拆出来打印）。上游 `v7.3` 这里长成了三处能力探测，同样**都不带日志**：

> `return glb_iface->control->version >= CSF_IFACE_VERSION(4, 1, 0);`（`panthor_fw_has_glb_state`）　`:326-331`
> `return glb_iface->control->version >= CSF_IFACE_VERSION(4, 0, 0);`（`panthor_fw_has_64bit_ep_req`）　`:333-338`

后果是：旧接口固件的结局是插桩特性返回 0，全局状态与 64 位 endpoint 请求走 else 分支，设备继续正常工作，**日志里什么都不写**。对比同树里存在会说话的架构门——上游 `v7.3` 的 `if (GPU_ARCH_MAJOR(ptdev->gpu_info.gpu_id) < 11) { drm_err(&ptdev->base, "Custom ASN hash not supported by the device"); return; }`（`panthor_gpu.c:76-79`）——可以确认这不是风格缺失：**这条线被设计成静默，那条线被设计成会说话。**

对本域的方法后果只有一句：**"这台设备没有某个 GPU 插桩/调试特性"与"这台设备的固件旧"这两种状态，在设备侧不可区分。** 因此任何以"特性缺失"推断"固件旧"的台账条目都必须降级（第廿六章 L1-INFO，并要求补采厂商读数）。同理，把 GPU 调试类接口的可用性当作固件新旧的代理指标也不成立——第 15 篇在观测面上已用过同一条论证，本文在特性面上复用它的结论而不重复其证明。

### 十一之五 MCU：故障能读到的形态，以及它的载体不可达

CSF 把命令流前端跑在 GPU 内部的 MCU 上，启停由驱动轮询寄存器。该分支的全部相关消息（均 `drm_err` / `drm_warn`，均只进内核日志）：

> `[MCU_STATUS_FATAL] = "fatal",`　`:1002`
> `drm_err(&ptdev->base, "Failed to boot MCU (status=%s)",`　`:1006`
> `drm_err(&ptdev->base, "Failed to stop MCU");`　`:1021`
> `drm_warn(&ptdev->base, "Failed to cleanly suspend MCU");`　`:1050`
> `drm_err(&ptdev->base, "FW fast reset failed, trying a slow reset");`　`:1092`
> `drm_err(&ptdev->base, "FW slow reset failed (couldn't flush FW's AS l2cache)");`　`:1096`
> `drm_err(&ptdev->base, "FW slow reset failed (couldn't start the FW )");`　`:1109`
> `drm_err(&ptdev->base, "FW ping timeout, scheduling a reset");`　`:1296`

四点归属含义：

1. `Failed to boot MCU (status=%s)` 会把状态字符串（含 `fatal`）打出来——这是**运行时状态**读数，与镜像版本无关，因此它是"设备此刻坏"的证据，不是"设备有漏洞"的证据。第廿一章把这两者混为一谈列为第 7 个误区。
2. `:1050` 是 `drm_warn`，即**软挂起没成功但不致命**；一条长期共存于一台正常设备的告警，不能作为任何指标的分子。第廿六章 K6（协处理器崩溃可见性）的分母定义里明确排除它。
3. `FW ping timeout, scheduling a reset` 之后驱动会**自动重载段并重启固件**（`:1105` `panthor_reload_fw_sections(ptdev, true);`、`:1107` `panthor_fw_start(ptdev)`）。这意味着**"重启过固件"不是"固件被更换过"的证据**——本域最容易写出假阳性工单的一种形状就是：设备日志里有 reset，资产台账里就写"固件已更新"。
4. 全部八条的载体都是内核日志，也就是第十九章 V6 面上那条 shell 域预期失败。本域这块的真实可核验程度是：**只有设备提供厂商自己的日志导出通道时才可达**；脚本 1 对它的记录方式是"通道是否存在"，而不是"有没有 MCU 错误"。

### 十一之六 本章的证据边界（必读）

本章所有事实来自 `kernel/common@android16-6.12` 与 `torvalds/linux@v7.3` 的公开源码，行号在第卅章取回记录里逐条对得上。把它们用于**你自己的设备**之前，必须先确认一件事——**这台设备的 Mali 驱动是不是这份代码。** 现实中三家 SoC 厂商的 Android 产品大量使用 Arm 的下游 `mali` 驱动树而非本树的 `panthor`，其字符串、行号与行为都不等同本章；十之四那条内核注释已经证明厂商侧存在私有分支。第 16 篇在模块归属上给过同一句纪律，这里重述它的含义：**把源码事实当作设备事实，是本域最高频的结论污染方式。**

因此本章的正确用法是：用它建立**"该问哪三个问题"**（镜像在哪、谁校验、失败时谁看得见），再用脚本 1 去设备上问一遍。脚本对 panthor 的输出把"当前设备是否使用此驱动"放在第一个字段——原因就在这里。
## 十二、Adreno：签名装载、一条真正的安全版本门，与被主动丢弃的版本号

本章处理本域第二个 GPU 阵营。高通侧与 Arm 侧的结构差别不是"有没有固件"，而是**校验发生在哪一层**：Arm 的 CSF 由内核驱动自己解析镜像（第十一章），高通的 GPU 镜像由**安全世界**判定能否装载（第十二章之二），而驱动只保留一条针对特定型号的微码版本门（第十二章之三）。这两件事叠起来决定了设备侧证据的形状。

出处约定：`kernel/common@android16-6.12` 的 `drivers/gpu/drm/msm/adreno/`（上游 drm 侧），以及第十二章之六单独标出的高通下游 `clo/la/kernel/msm-5.4`。

### 十二之一 镜像不是"一个"，槽位也不是"一份"

`adreno_gpu.h:23-30` 的枚举逐字是：

> `enum {`
> `    ADRENO_FW_PM4 = 0,`
> `    ADRENO_FW_SQE = 0, /* a6xx */`
> `    ADRENO_FW_PFP = 1,`
> `    ADRENO_FW_GMU = 1, /* a6xx */`
> `    ADRENO_FW_GPMU = 2,`
> `    ADRENO_FW_MAX,`
> `};`

三个必须看清的点：

- **`PM4` 与 `SQE` 共用下标 0，`PFP` 与 `GMU` 共用下标 1。** 数组是 `const char *fw[ADRENO_FW_MAX]`（`adreno_gpu.h:97`，目录项）与 `const struct firmware *fw[ADRENO_FW_MAX]`（`:191`，已取入的镜像）。也就是说**"GPU 有几个固件"这个问法在这份代码里没有唯一答案**：同一个数组槽位在老型号上装 PM4、在 a6xx 上装 SQE。第二章之四把这类现象列为"术语混用四"，这里是它的源码实体。
- 镜像文件名写在**目录表**里而不是设备树里。`a6xx_catalog.c` 的连续条目逐字为 `[ADRENO_FW_SQE] = "a630_sqe.fw"`、`[ADRENO_FW_GMU] = "a630_gmu.bin"`、`.zapfw = "a610_zap.mdt"` / `"a615_zap.mdt"` / `"a615_zap.mbn"` / `"a620_zap.mbn"`（`:678-684`、`:709-716`、`:738-746`、`:787-794`、`:859-868`）。请注意 `a630_sqe.fw` **在至少六个不同 GPU 目录项里重复出现**（`:679`、`:710`、`:739`、`:765`、`:788`、`:812`、`:835`），而 `a619_gmu.bin` / `a621_gmu.bin` 与 `a630_gmu.bin` 交叉使用。
- **zap 后缀混用 `.mdt` 与 `.mbn`**，同一目录表的相邻条目里就有两种（`:716` 与 `:746`）。这条不是排版问题：`request_firmware` 按文件名精确匹配，所以**"这台设备的 GPU 固件叫什么"必须由目录表或设备树决定，不能由型号名推**。第十八章把它列为命名四态之一。

同一张表的尾部还有一段与目录表并列的模块元数据：`DECLARE_ADRENO_GPULIST(a6xx);`（`:1041`）之后是**连续十条 `MODULE_FIRMWARE`**（`:1043-1052`，从 `MODULE_FIRMWARE("qcom/a615_zap.mbn");` 到 `MODULE_FIRMWARE("qcom/a660_sqe.fw");`）。请注意两件事：其一，它们全部带 `qcom/` 前缀，而同一张表里的目录条目（`:679`、`:711`、`:684` 那些）不带前缀，**同一份驱动内两套命名写法并存**——用 `modinfo` 得到的候选名集合与用目录表得到的候选名集合不是同一个集合。其二，这十条是本报告 GPU 三家里唯一非零的 `MODULE_FIRMWARE` 读数：panthor 在 `android16-6.12` 只有 1 条（第十一章之一的 `MODULE_FIRMWARE("arm/mali/arch10.8/mali_csffw.bin");`），第十四章之六对 MFC 目录的检索为 0 条。第廿五章的"声明数 0 但需要固件"那一格，参照系就是本节这十条。

由此得出一条会被第廿五章矩阵直接使用的规则：**GPU 侧资产台账的主键不是型号，而是"目录表项 + 装载到的槽位 + 实际取到的文件名"三元组。**

### 十二之二 装载：三次尝试、三种日志、一个被记住的选择

`adreno_request_fw()`（`adreno_gpu.c:446` 起）按固定顺序试三种位置：

> `newname = kasprintf(GFP_KERNEL, "qcom/%s", fwname);`　`:453`
> `/* Try first to load from qcom/$fwfile using a direct load (to avoid a potential timeout waiting for usermode helper) */`（注释大意，原句见 `:457-460`）
> `ret = request_firmware_direct(&fw, newname, drm->dev);`
> `if (!ret) { DRM_DEV_INFO(drm->dev, "loaded %s from new location\n", newname); adreno_gpu->fwloc = FW_LOCATION_NEW; goto out; }`　`:464-469`
> `/* Then try the legacy location without qcom/ prefix */`　`:479`
> `ret = request_firmware_direct(&fw, fwname, drm->dev);` → `DRM_DEV_INFO(..., "loaded %s from legacy location\n", fwname);` → `fwloc = FW_LOCATION_LEGACY;`　`:484-488`
> `/* Finally fall back to request_firmware() for cases where the usermode helper is needed (I think mainly android) */`　`:499-501`
> `ret = request_firmware(&fw, newname, drm->dev);` → `DRM_DEV_INFO(..., "loaded %s with helper\n", newname);` → `fwloc = FW_LOCATION_HELPER;`　`:505-509`
> `DRM_DEV_ERROR(drm->dev, "failed to load %s\n", fwname); fw = ERR_PTR(-ENOENT);`　`:519-520`

四条判读：

1. **三条路径的名字空间不同**：`qcom/<file>` 与 `<file>`；`FW_LOCATION_*` 枚举的注释把它们写成 `/lib/firmware/qcom/$fwfile` 与 `/lib/firmware/$fwfile`（`adreno_gpu.h:184-188`）。同一份镜像在两个位置任一处存在即可，且**位置一旦确定就被记进 `adreno_gpu->fwloc`**，后续所有装载沿用该选择（见 `:140` 的判断）。
2. `:499-501` 那句 `(I think mainly android)` 是本报告愿意逐字引用的一句注释：**上游作者把"用户态 helper 装载"这条路径标注为 Android 常见形态**。它对本域的含义不是技术性的，而是台账性的——走 helper 路径时，取镜像的是**用户态固件服务**而不是内核直接查表，因此"文件在哪"由 init/ueventd 规则决定（第 11 篇讨论该链）。这也正是第二章第三条定义句里 `firmware_class.path=/vendor,/firmware_mnt` 这类内核命令行参数的用武之地。
3. **`fwloc` 没有任何 sysfs 或属性出口。** 全树对它的写入只发生在 `:468/488/509`，读取只发生在 `:140` 与 `:461/481/502` 的自身判断——**内核知道自己是走哪条路径拿到镜像的，但不告诉任何人**。要得到这个读数只能靠 `DRM_DEV_INFO` 那三行日志，也就是第十九章 V6 面上那条不可达通道。
4. `adreno_load_fw()` 的循环里有一句要紧的跳过逻辑，原句连拼写一起照抄：`/* Skip loading GMU firwmare with GMU Wrapper */` 后接 `if (adreno_has_gmu_wrapper(adreno_gpu) && i == ADRENO_FW_GMU) continue;`（`adreno_gpu.c:536-538`）。含义：**新型号上 GMU 槽位根本不装载镜像**，所以"这台设备没有 GMU 固件"可以是完全正常的状态——脚本 2 比对漂移时必须把它当成结构性差异而不是回归。

### 十二之三 全树唯一的一条微码安全版本门

第 L-4 条结论说"本域唯一一处内核主动拒绝旧镜像的实现是高通的 SQE 版本检查"。它的原文如下。注释逐字：

> `/*`
> ` * Check that the microcode version is new enough to include several key`
> ` * security fixes. Return true if the ucode is safe.`
> ` */`
> `static bool a6xx_ucode_check_version(struct a6xx_gpu *a6xx_gpu,`
> `        struct drm_gem_object *obj)`　`a6xx_gpu.c:709-714`

版本号直接从**镜像自身的前几个双字**取（`u32 *buf = msm_gem_get_vaddr(obj);`　`:719`），然后分四档：

| 判定 | 条件（逐字） | 行为 |
| --- | --- | --- |
| 直接放行 | `/* A7xx is safe! */` / `if (adreno_is_a7xx(adreno_gpu) || adreno_is_a702(adreno_gpu)) return true;`　`:725-727` | 不检查版本 |
| a630 | `if ((((buf[0] & 0xf) == 0xa) && (buf[2] & 0xf) >= 1) \|\| (buf[0] & 0xfff) >= 0x190)`　`:750-751` | 满足则 `a6xx_gpu->has_whereami = true; ret = true;`　`:752-753`；否则报 `a630 SQE ucode is too old. Have version %x need at least %x` 并带 `buf[0] & 0xfff, 0x190`　`:757-759` |
| a650 | `if ((buf[0] & 0xfff) >= 0x095)`　`:761` | 否则报 `a650 SQE ucode is too old. Have version %x need at least %x`（`0x095`）　`:766-768` |
| a660 | `} else if (!strcmp(sqe_name, "a660_sqe.fw")) { ret = true; }`　`:769-770` | **无条件放行**，配套注释 `a660 targets have all the critical security fixes from the start`（`:737-738`） |
| 其它 | `DRM_DEV_ERROR(&gpu->pdev->dev, "unknown GPU, add it to a6xx_ucode_check_version()!!\n");`　`:772-773` | 落到 `return ret;`（`false`）　`:777` |

三条设计细节要按源码读，不能按直觉读：

- **`0xa` 是一个补丁标记位而不是版本。** 注释逐字：`If the lowest nibble is 0xa that is an indication that this microcode has been patched. The actual version is in dword [3] but we only care about the patchlevel which is the lowest nibble of dword [3]`（`:740-744`）。所以 `buf[0]` 低位为 `a` 时，比较的是 `buf[2] & 0xf >= 1`，**版本号本身被忽略**。
- **门槛写的是十六进制而不是语义版本**：`0x190` 对应注释 `1.90 which was the first version that had this fix built in`（`:746-748`）、`0x095` 对应注释 `equal to or newer than 0.95 for other security fixes`（`:734-736`）。台账里把这类数字写成 "1.9.0" 会引入不存在的三段语义。
- **拒绝的后果是 GPU 不可用**。调用点在 `a6xx_ucode_load()`：版本检查不过时释放并丢弃 SQE buffer object，置空后 `return -EPERM;`（`a6xx_gpu.c:800-806`），装载入口本身失败时报 `Could not allocate SQE ucode: %d`（`:793-794`）。注意 `:799` 那行 `msm_gem_object_set_name(a6xx_gpu->sqe_bo, "sqefw");`——镜像被命名成 `sqefw` 挂在 GPU 地址空间里，这是它后来能被内部读到的原因（版本直接从 `buf[]` 取，不需要文件）。

这条门为什么重要：**它是本域唯一一个"内核自己判断旧镜像不安全"的实现**。因此它同时给出一个反向方法结论——`has_whereami` 只在检查通过时被置位（`:752`），**它是一个安全判定的副产物，却以"功能位"的形式存在于内核结构里，且没有任何对外读数**（全树只在 `:813` 被内部使用：`if ((adreno_gpu->base.hw_apriv || a6xx_gpu->has_whereami) && !a6xx_gpu->shadow_bo)`）。第廿一章把它归入 FT6 的一个典型实例：安全判定与功能判定共用状态位，外部只能观察到功能。

### 十二之四 zap shader：签名装载与"静默不可用"

zap 是 GPU 侧唯一一条**由安全世界裁决**的装载路径，也是本域最容易被误写成"固件版本"的一段。`adreno_gpu.c` 的 `zap_shader_load_mdt()`（`:30-31`）：

> `static bool zap_available = true;`　`:28`
> `if (!IS_ENABLED(CONFIG_ARCH_QCOM)) { zap_available = false; return -EINVAL; }`　`:43-46`
> `np = of_get_child_by_name(dev->of_node, "zap-shader");`
> `if (!of_device_is_available(np)) { zap_available = false; return -ENODEV; }`　`:48-52`
> `of_property_read_string_index(np, "firmware-name", 0, &signed_fwname);`　`:82`
> `if (signed_fwname) {` → `fwname = signed_fwname;` → `ret = request_firmware_direct(&fw, fwname, gpu->dev->dev);` → `if (ret)` → `fw = ERR_PTR(ret);`　`:83-87`
> `} else if (fwname) {` → `/* Request the MDT file from the default location: */` → `fw = adreno_request_fw(to_adreno_gpu(gpu), fwname);`　`:88-90`
> `} else {` → `return -ENOENT;`　`:91-103`（中间是五行注释块）
> `/* Send the image to the secure world */`
> `ret = qcom_scm_pas_auth_and_reset(pasid);`　`:155-156`
> `/* If the scm call returns -EOPNOTSUPP we assume that this target doesn't need/support the zap shader so quietly fail */`
> `if (ret == -EOPNOTSUPP) zap_available = false;`
> `else if (ret) DRM_DEV_ERROR(dev, "Unable to authorize the image\n");`　`:158-165`

原注释解释了这个"新方案"存在的理由，逐字：

> `* Check for a firmware-name property.  This is the new scheme`
> `* to handle firmware that may be signed with device specific`
> `* keys, allowing us to have a different zap fw path for different`
> `* devices.`　`:68-72`
>
> `* Because the firmware is signed with a (potentially)`
> `* device specific key, having the name come from gpulist`
> `* was a bad idea, and is only provided for backwards`
> `* compatibility for older targets.`　`:97-100`

四条判读：

1. **镜像名从设备树来，是因为签名密钥可能是机型专属的。** 这是本域少见的、由内核作者书面承认"把固件名写在通用表里是错的"的例子（`:97-100`）。它也解释了十二之一里目录表 `.zapfw` 的混用后缀为何能长期存在——旧目标保留、新目标走 DT。
2. **通过与否由安全世界决定，内核只转达一个返回码。** 因此"这份镜像在这台设备上被不被接受"**无法从文件内容判断**，只能从设备读数判断；而它的失败消息 `Unable to authorize the image`（`:165`）与 `Unable to load %s`（`:106`）都只进内核日志。第 8 篇讨论这条信任根的形态，本文只保留它的**证据学后果**。
3. **`-EOPNOTSUPP` 被设计成静默**（`:162-163`），`zap_available = false` 是一个进程内全局位，**没有对外读数**。因此"这台设备到底有没有 zap"在标准姿势下不可判定——第卅章把它列入未文档化边界清单。
4. **zap 与 MDT 结构绑定**：`mem_size = qcom_mdt_get_size(fw);`（`:111`）、区域不够时 `memory region is too small to load the MDT`（`:118-120`）、以及 `:131-139` 那段关于 `qcom/` 前缀两种路径的注释（`adreno_request_fw() handles this, but qcom_mdt_load() does not`）。含义是：**这类镜像不是任意 blob，而是有可解析结构的分段装载件**，因此它的"版本"天然可能是多段的——第十四、十五章会看到同样的结构差异。

### 十二之五 GMU：内核与固件交换版本号，然后把它扔掉

`a6xx_hfi.c` 里有本域最直白的一句"我们知道但我不留"：

> `static int a6xx_hfi_get_fw_version(struct a6xx_gmu *gmu, u32 *version)`
> `{`
> `    struct a6xx_hfi_msg_fw_version msg = { 0 };`
>
> `    /* Currently supporting version 1.10 */`
> `    msg.supported_version = (1 << 28) | (1 << 19) | (1 << 17);`
>
> `    return a6xx_hfi_send_msg(gmu, HFI_H2F_MSG_FW_VERSION, &msg, sizeof(msg),`
> `        version, sizeof(*version));`
> `}`　`:212-221`

调用点逐字：

> `ret = a6xx_hfi_get_fw_version(gmu, NULL);`
> `if (ret)`
> `    return ret;`
>
> `/*`
> ` * We have to get exchange version numbers per the sequence but at this`
> ` * point th kernel driver doesn't need to know the exact version of`
> ` * the GMU firmware`
> ` */`　`:722-730`

三点必须写进报告：

1. **版本号真的在总线上交换过**（`HFI_H2F_MSG_FW_VERSION`，且驱动的应答位编码了它自称支持的 1.10），但**接收指针传的是 `NULL`**（`:722`）。也就是说内核**主动放弃**了这个读数，理由是协议序列要求交换而不是驱动需要它。这条是全篇关于"设备侧不可见"的最强一条因果证据：不是拿不到，是**拿到了就丢**。
2. 原注释的两处英文瑕疵（`th kernel driver`、末句无句号）**照抄不改**，因为它们是取回核对的指纹（第廿三章的逐字纪律）。
3. 由于 `a6xx_hfi_send_msg()` 的应答写入是可选的，**"这台设备的 GMU 固件版本"在公开代码里不存在读取路径**。第卅章把它列入"未文档化但已被本文核验为不存在"的清单，并注明核验方法（全树检索 `HFI_H2F_MSG_FW_VERSION` 的调用点只有一处，且传 `NULL`）。

对照第十一章之四：Arm 侧保留了固件自报的接口版本但只打进日志，高通侧连保留都不做。**两家的差别不是严格程度，而是"驱动是否需要这个数才能工作"。** 这条区别是第廿五章归属矩阵里"证据可得性"一列的来源。

### 十二之六 本章的归属结论

| 事实 | 归属落点 |
| --- | --- |
| SQE 版本门只覆盖 `a630_sqe.fw` / `a650_sqe.fw`，a7xx 与 a660 直接放行 | **F3 驱动层的拒绝能力是按型号硬编码的**；新平台上旧镜像风险不由内核承担 |
| 门失败返回 `-EPERM`、镜像被丢弃 | 可观测性为"GPU 起不来"，而不是"GPU 用着旧固件" |
| zap 由 `qcom_scm_pas_auth_and_reset()` 裁决，`-EOPNOTSUPP` 静默 | 判定权在 **F5 安全世界**（第 8 篇），本文只登记其不可见性 |
| GMU 版本交换后丢弃 | **F1 镜像本体在本域没有任何平台级读数**，只能走厂商公告 / 补充补丁清单（第八章） |
| `fwloc` 只进日志 | **F2 装载层的路径选择不可读**，其唯一痕迹是三行 `DRM_DEV_INFO` |

### 十二之七 一句必须防住的误读

本章给出的每一条"内核会拒绝"，都**不能**被写成"所以旧固件在这台设备上无法被利用"。版本门只检查 SQE 的一个 nibble 位段；zap 检查的是签名而不是版本；GMU 什么也不检查。**内核的拒绝行为描述的是可用性，不是安全性。** 本域的判定动作在第廿六章给出：把"驱动接受了这份镜像"记为 L1-INFO 而非 L0-LOW，把"驱动拒绝了这份镜像"记为可观测事件而不是修复证据。

## 十三、下游 KGSL：版本可读，但没有任何一条标准姿势能读到它

高通的 Android 产品大多不使用第十二章那份 `drivers/gpu/drm/msm` 代码，而使用自家下游驱动 `drivers/gpu/msm/`（KGSL）。本章只处理一个问题：**下游驱动是否提供固件版本读数，以及这个读数在标准核验姿势下够不够得着。**

出处：`git.codelinaro.org/clo/la/kernel/msm-5.4`，分支 `msm-5.4`，`drivers/gpu/msm/`（该文件与本次核验取回的副本逐字节一致，第卅章记录其大小与获取方式）。

### 十三之一 下游确实解析并保存了版本号，逻辑与上游同源

> `static u32 get_ucode_version(const u32 *data)`
> `{`
> `    u32 version;`
>
> `    version = data[1];`
>
> `    if ((version & 0xf) != 0xa)`
> `        return version;`
>
> `    version &= ~0xfff;`
> `    return  version | ((data[3] & 0xfff000) >> 12);`
> `}`　`adreno.c:70-81`

这与第十二章之三 a6xx 门的 `0xa` 补丁标记是同一套编码约定（一个用 `data[1]`／`data[3]`，另一个用 `buf[0]`／`buf[2]`，偏移差一位）。取用点在装载函数里：

> `int adreno_get_firmware(struct adreno_device *adreno_dev, const char *fwfile, struct adreno_firmware *firmware)`　`:83-84`
> `ret = request_firmware(&fw, fwfile, &device->pdev->dev);`　`:93`
> `if (ret) { dev_err(device->dev, "request_firmware(%s) failed: %d\n", fwfile, ret); return ret; }`　`:95-99`
> `firmware->memdesc = kgsl_allocate_global(device, fw->size - 4, 0, KGSL_MEMFLAGS_GPUREADONLY, KGSL_MEMDESC_UCODE, "ucode");`　`:101-103`
> `memcpy(firmware->memdesc->hostptr, &fw->data[4], fw->size - 4);`　`:107`
> `firmware->version = get_ucode_version((u32 *)fw->data);`　`:109`

两点值得单独记：**镜像前 4 字节被跳过**（`:107`），版本号却从完整数据取（`:109`）——所以文件头是**驱动认可的、但不进入 GPU 可见内存**的一段元数据；以及镜像以 `KGSL_MEMDESC_UCODE` 标志、名字 `ucode` 登记在全局内存里，这是一个**内部对象名而不是对外读数**。

### 十三之二 版本号的唯一出口是一个 ioctl 属性，而且只覆盖两类镜像

属性处理函数逐字：

> `static int adreno_prop_ucode_version(struct kgsl_device *device, struct kgsl_device_getproperty *param)`
> `{`
> `    struct adreno_device *adreno_dev = ADRENO_DEVICE(device);`
> `    struct kgsl_ucode_version ucode = {`
> `        .pfp = adreno_dev->fw[ADRENO_FW_PFP].version,`
> `        .pm4 = adreno_dev->fw[ADRENO_FW_PM4].version,`
> `    };`
>
> `    return copy_prop(param, &ucode, sizeof(ucode));`
> `}`　`adreno.c:2499-2509`

结构体定义在公开 UAPI 头里（`include/uapi/linux/msm_kgsl.h:405-408`）：`struct kgsl_ucode_version { unsigned int pfp; unsigned int pm4; };`，属性号 `#define KGSL_PROP_UCODE_VERSION 0x15`（`:329`）。

三点结论：

1. **返回两个数，槽位只有两个。** 下游的枚举是 `#define ADRENO_FW_PFP 0` / `#define ADRENO_FW_SQE 0` / `#define ADRENO_FW_PM4 1`（`adreno.h:173-175`），容器是 `struct adreno_firmware fw[2];`（`:511`）——**只有两个槽位**。所以在这个实现里，SQE 与 PFP 从同一个槽读出。这与第十二章之一的上游别名现象一致，但**数组长度写死为 2**。
2. **`KGSL_PROP_GPMU_VERSION`（UAPI `:330`）有号无实现**：驱动的属性表 `adreno_property_funcs[]`（`adreno.c:2560-2581`）里没有它，未命中的属性统一 `return -ENODEV;`（`:2583-2594`）。**UAPI 里声明的属性不等于设备支持它。**
3. **`gmu` / `zap` / `ctx` 一律没有属性出口。** 全表里与版本有关的只有 `KGSL_PROP_UCODE_VERSION` 一项（`:2572`）。第十二章之五那条"GMU 版本被丢弃"在下游同样成立——下游连丢弃的机会都不必创造，因为它没有对外登记。

### 十三之三 sysfs 上有一整排 GPU 读数，其中没有固件版本

`kgsl_pwrctrl.c:1080-1106` 是该驱动 GPU 侧 sysfs 的属性声明区，逐条列出（**下表是清单而不是原句**；源码形态为 `static DEVICE_ATTR_RO(name);` 与 `static DEVICE_ATTR_RW(name);` 的连续声明，名字即属性名）：

| 属性 | 读写 | 属性 | 读写 |
| --- | --- | --- | --- |
| `temp` | RO | `reset_count` | RO |
| `gpuclk` | RW | `force_clk_on` | RW |
| `max_gpuclk` | RW | `force_bus_on` | RW |
| `idle_timer` | RW | `force_rail_on` | RW |
| `minbw_timer` | RW | `bus_split` | RW |
| `gpubusy` | RO | `default_pwrlevel` | RW |
| `gpu_available_frequencies` | RO | `popp` | RO |
| `gpu_clock_stats` | RO | `force_no_nap` | RW |
| `max_pwrlevel` | RW | `gpu_model` | RO |
| `min_pwrlevel` | RW | `gpu_busy_percentage` | RO |
| `thermal_pwrlevel` | RW | `min_clock_mhz` | RW |
| `num_pwrlevels` | RO | `max_clock_mhz` | RW |
| `clock_mhz` | RO | `freq_table_mhz` | RO |
| `pwrscale` | RW | | |

Adreno 侧另有 `static DEVICE_INT_ATTR(wake_nice, 0644, adreno_wake_nice);` 与 `DEVICE_INT_ATTR(wake_timeout, 0644, adreno_wake_timeout);`（`adreno_sysfs.c:272-273`）。

三点判读：

1. **没有任何一个属性承载固件/微码版本。** 有 `gpu_model`（其 show 函数在 `kgsl_pwrctrl.c:827` 的 `_gpu_model_show()` / `:836` 的 `gpu_model_show()`）——**型号可读、版本不可读**，第 L-1 条结论在本域最标准的形态就是这个 sysfs 目录。
2. **`reset_count`（RO，`kgsl_pwrctrl.c:1092`，show 函数在 `:642`）是少数几个可长期采集的运行时读数**，配合第十一章之五那条"固件重启不是固件更换"，它是本域唯一能用来做**趋势**而不是做**结论**的 GPU 计数器（第廿六章 K6 用它做分母，不做分子）。
3. 这张表同时说明**RW 属性很多**（`gpuclk`、`thermal_pwrlevel`、`force_*` 等）。第卅章的边界清单里有一条与本域安全相关的观察：能写这些节点的那一方，其能力远超"读取固件版本"——但**这是权限面而不是固件面，由第 1 篇与第 12 篇处理**，本文只登记其存在，不做展开。

### 十三之四 标准核验姿势：这个读数够得着吗

分两层，逐条给依据：

| 读数 | 需要的权限 | 平台策略依据 | 结论 |
| --- | --- | --- | --- |
| `kgsl_ioctl_device_getproperty()` → `adreno_getproperty()` → `KGSL_PROP_UCODE_VERSION` | 打开 `/dev/kgsl-3d0` 的 fd 并发起 ioctl | 该节点类型为 `gpu_device`（`system/sepolicy` 中 `type gpu_device, dev_type, mlstrustedobject, isolated_compute_allowed_device;`，`device.te:28`）；**`/dev/kgsl-3d0` 本身不在 AOSP 平台 `file_contexts` 里**，其标签由厂商策略给出 | 平台策略层无法给出"shell 可读"的保证；实测以脚本为准 |
| `ls /sys/class/kgsl/kgsl-3d0/` | 目录 `sysfs` 类型的读权限 | `allow shell sysfs:dir r_dir_perms;`（`shell.te:443`） | **目录可以列**；节点标签若为 `sysfs` 则列表可见 |
| `cat /sys/class/kgsl/kgsl-3d0/<attr>` | 文件 `sysfs` 类型的读权限 | shell 只被授予若干专项 sysfs 文件（如 `sysfs_batteryinfo:file r_file_perms;`　`shell.te:444-445`、`sysfs_lru_gen_enabled:file`　`:448`、`sysfs_mem_sleep:file`　`:451`），**没有一般 `sysfs:file` 权限** | **列得出、读不到**——这条不对称是脚本 1 会稳定复现的一类失败 |
| `ls /sys/class/gpu/` | 需要 `sysfs_gpu:file` 读权限 | `genfscon sysfs /class/gpu u:object_r:sysfs_gpu:s0`（`genfs_contexts:129`）；`allow { appdomain -isolated_app_all } sysfs_gpu:file r_file_perms;`（`app.te:395`） | **应用域被明文允许读，shell 域没有任何一条规则授予它**——本域少见的"shell 比 app 更受限"的一格 |
| 内核日志中的 `request_firmware(...) failed`、`Unable to authorize the image` | 需要 `kernel:system syslog_read` | `neverallow appdomain kernel:system { syslog_read ... }`（`private/app.te:661`，第十九章 V6 面） | 标准姿势不可达 |

`sysfs_gpu` 那一行是本章最值得停下来看的一条：**平台策略里唯一授予该类型读权限的对象是 `appdomain`（且只排除 isolated app），而 shell 属于 coredomain、不属于 appdomain**。第二十章把它列进"shell 域预期失败表"，并给出为什么这不影响结论：本域的固件版本读数本来就不在任何 sysfs 上（十三之三），sysfs 可达性只影响型号与计数器。

### 十三之五 一个真正可用的读数：先问设备支持哪些属性

如果一定要在设备侧留下一条**关于 GPU 固件读数能力的可复核事实**，KGSL 提供了唯一一种方式：问它支持哪些属性。链路完整、语义清楚：

> `#define KGSL_PROP_QUERY_CAPABILITIES 0x27`　`msm_kgsl.h:342`
> `KGSL_QUERY_CAPS_PROPERTIES returns a list of the valid properties in the kernel. The subtype data should be struct kgsl_capabilities_properties` → `#define KGSL_QUERY_CAPS_PROPERTIES 1`　`:360-363`
> `{ KGSL_PROP_QUERY_CAPABILITIES, kgsl_prop_query_capabilities },`（`kgsl.c:1712`）
> 能力查询内部先取数量再取清单：`count = kgsl_query_property_list(device, NULL, 0);`（`kgsl.c:1625`）、`props.count = kgsl_query_property_list(device, local, count);`（`:1646`）
> 数量来自各层拼装：`if (device->ftbl->query_property_list) num += device->ftbl->query_property_list(device, list, ...)`（`kgsl.c:1742-1743`、`:1752-1753`），Adreno 侧的实现注册在 `.query_property_list = adreno_query_property_list,`（`adreno.c:4056`），函数体在 `:2596`

因此**"这台设备的 KGSL 是否实现了 `KGSL_PROP_UCODE_VERSION`"是一个可以正面回答的问题**，而"版本是多少"受节点权限限制。这就是本章建议的台账姿势：**先记录能力清单，再尝试取值**；脚本 1 的 GPU 段按这个顺序输出，两个字段各自独立判定，绝不把"取不到值"合并成"设备不支持"。

### 十三之六 本章结论（一句话版）

下游 KGSL **确实解析并保存**了微码版本（`adreno.c:109`），**确实有一个公开的 UAPI 属性**能取到它（`msm_kgsl.h:329` / `adreno.c:2572`），但——

- 它只覆盖 pfp 与 pm4 两个槽（`msm_kgsl.h:405-408`），GMU 与 zap 没有出口；
- sysfs 上没有版本节点（`kgsl_pwrctrl.c:1080-1106`）；
- 取它需要 `gpu_device` 权限，而该权限归厂商策略而非平台策略；
- 上游那一侧更糟：GMU 版本被交换后**主动丢弃**（第十二章之五）。

所以本域的正确答案不是"读不到所以没有"，也不是"能读所以安全"，而是第三章 F5 那一层的原话：**版本读数的可得性由策略决定，而不由文件决定**。第卅章把"`KGSL_PROP_UCODE_VERSION` 的实际可达性（含各厂商 `/dev/kgsl-3d0` 标签差异）"列入未公开文档化边界清单。
## 十四、Exynos MFC：同一个驱动里有四种"版本"，只有第一种能用

第十三章末尾把 GPU 域的结论收在"版本读数的可得性由策略决定"。媒体编解码器域比 GPU 域更彻底：Exynos MFC 的驱动里同时存在四个都可以被称为"固件版本"的东西，其中一个是数组下标、一个只走到调试打印、一个从未被读、剩下一个（硬件版本）根本不是固件版本。本章把四者逐一钉死，因为台账上关于 MFC 的错误记录几乎都来自把这四个混为一谈。

出处约定：`kernel/common@android16-6.12` 的 `drivers/media/platform/samsung/s5p-mfc/`，目录共 34 个文件（`Kconfig`、`Makefile` 与 32 个 `.c`/`.h`），全目录取回后本地检索，检索命令与命中数见本章之六。

### 十四之一 装载：一次降序的文件名试错

`/* Load firmware */`（`s5p_mfc_ctrl.c:44`）之下是 `int s5p_mfc_load_firmware(struct s5p_mfc_dev *dev)`（`:45`），紧接的注释给出这个域装载面的前提：

> `	/* Firmware has to be present as a separate file or compiled`
> `	 * into kernel. */`
> — `s5p_mfc_ctrl.c:50-51`

真正的选择逻辑只有十行，但它是本章的全部主题：

> `	for (i = MFC_FW_MAX_VERSIONS - 1; i >= 0; i--) {`
> `		if (!dev->variant->fw_name[i])`
> `			continue;`
> `		err = request_firmware((const struct firmware **)&fw_blob,`
> `				dev->variant->fw_name[i], &dev->plat_dev->dev);`
> `		if (!err) {`
> `			dev->fw_ver = (enum s5p_mfc_fw_ver) i;`
> `			break;`
> `		}`
> `	}`
> — `s5p_mfc_ctrl.c:63-72`

三点必须看清：

1. **降序**：`i` 从 `MFC_FW_MAX_VERSIONS - 1` 往下走，`MFC_FW_MAX_VERSIONS` 是 `2`（`s5p_mfc_common.h:40`）。也就是说先试下标 1、再试下标 0——**盘上有新文件就用新的，这是默认偏好而不是安全检查**。
2. **试错的对象是文件名**，成功判据是 `request_firmware()` 返回 0（`:66-68`）。内核不知道也不需要知道文件内容是什么版本；它只知道"这个文件名打开了"。
3. `dev->fw_ver = (enum s5p_mfc_fw_ver) i;`（`:69`）记录的是**赢的那个文件名的数组下标**。

各代 MFC 的名字表在 `s5p_mfc.c` 的变体表里，逐行如下（同一目录同一分支）：

| 变体结构 | 硬件版本字段 | `fw_name[]` | 行号 |
| --- | --- | --- | --- |
| `mfc_drvdata_v5` | `MFC_VERSION` | `"s5p-mfc.fw"`（只有 [0]） | `:1536-1541` |
| `mfc_drvdata_v6` | `MFC_VERSION_V6` | `"s5p-mfc-v6.fw"`、`"s5p-mfc-v6-v2.fw"` | `:1561-1571` |
| `mfc_drvdata_v7` | `MFC_VERSION_V7` | `"s5p-mfc-v7.fw"` | `:1590-1595` |
| `mfc_drvdata_v7_3250` | `MFC_VERSION_V7` | `"s5p-mfc-v7.fw"` | `:1600-1605` |
| `mfc_drvdata_v8` | `MFC_VERSION_V8` | `"s5p-mfc-v8.fw"` | `:1624-1629` |
| `mfc_drvdata_v8_5433` | `MFC_VERSION_V8` | `"s5p-mfc-v8.fw"` | `:1634-1639` |
| `mfc_drvdata_v10` | `MFC_VERSION_V10` | `"s5p-mfc-v10.fw"` | `:1659-1664` |
| （v12 变体） | `MFC_VERSION_V12` | `"s5p-mfc-v12.fw"` | `:1683-1687` |

于是第三章 F2 那张表里"文件系统装载"这一行在本域有了一个具体形状：**八条硬件变体里只有 v6 那条有两个候选名**，降序试错对所有其它代都只是"试一次"。台账如果按"MFC 驱动会挑最新版本"来记，就把 v6 特例当成了普遍规律。

### 十四之二 第一种"版本"：`dev->fw_ver` 是下标，但它真的有功能后果

`enum s5p_mfc_fw_ver` 逐字只有两个值：

> `enum s5p_mfc_fw_ver {`
> `	MFC_FW_V1,`
> `	MFC_FW_V2,`
> `};`
> — `s5p_mfc_common.h:165-168`

字段声明与它的自我说明：`enum s5p_mfc_fw_ver fw_ver;`（`s5p_mfc_common.h:346`），注释 ` * @fw_ver:		loaded firmware sub-version`（`:298`）。全目录检索 `fw_ver` 命中 3 个文件、5 处：写一次（`s5p_mfc_ctrl.c:69`）、声明两处（`s5p_mfc_common.h:165`、`:346`，另加注释 `:298`）、**读一处**：

> `#define IS_MFCV6_V2(dev) (!IS_MFCV7_PLUS(dev) && dev->fw_ver == MFC_FW_V2)`
> — `s5p_mfc_opr_v6.c:43`

而 `IS_MFCV6_V2` 只有两个使用点，两处都在决定往哪个寄存器写：

> `	if (IS_MFCV7_PLUS(dev) || IS_MFCV6_V2(dev)) {`
> `		writel(reg, mfc_regs->d_dec_options);`
> — `s5p_mfc_opr_v6.c:1774-1775`
> `	if (IS_MFCV7_PLUS(dev) || IS_MFCV6_V2(dev))`
> `		writel(reg, mfc_regs->d_init_buffer_options);`
> `	else`
> `		writel(reg, mfc_regs->d_dec_options);`
> — `s5p_mfc_opr_v6.c:1789-1792`

这就是本域最值得记的一条机制事实：**盘上放的是 `s5p-mfc-v6.fw` 还是 `s5p-mfc-v6-v2.fw`，会改变解码器初始化时写哪个硬件寄存器**（`:1789-1792` 是二选一，`:1774-1777` 决定是否提前清零）。而决定者是文件名试错的结果，没有任何一条设备侧读数把结果报出来——`fw_ver` 不出现在任何 sysfs 属性、任何 ioctl、任何 property 里（本章之六给出全目录证据）。

安全含义（第廿一章会把它归入 FT 类）：同一个 SoC、同一份内核、同一个解码器实例，仅仅因为 `/lib/firmware` 里多了一个文件，硬件配置路径就不同。若某个缺陷只在其中一条路径上存在，那么"设备是否有缺陷"的判断必须落到**文件系统清单**，而清单不可枚举——这正是第十三章末尾那条不对称（目录列得出、文件读不到）在本域的兑现。

### 十四之三 第二种"版本"：硬件上报的三位版本只走到调试打印

初始化流程末尾确实读了一次固件版本寄存器：

> `	if (IS_MFCV6_PLUS(dev))`
> `		ver = mfc_read(dev, S5P_FIMV_FW_VERSION_V6);`
> `	else`
> `		ver = mfc_read(dev, S5P_FIMV_FW_VERSION);`
> ``
> `	mfc_debug(2, "MFC F/W version : %02xyy, %02xmm, %02xdd\n",`
> `		(ver >> 16) & 0xFF, (ver >> 8) & 0xFF, ver & 0xFF);`
> — `s5p_mfc_ctrl.c:281-287`

寄存器地址来自 `regs-mfc-v6.h:72` 的 `#define S5P_FIMV_FW_VERSION_V6			0xf000` 与 `regs-mfc.h:41` 的 `#define S5P_FIMV_FW_VERSION		0x0058`。也就是说：**固件把版本写在自己的寄存器窗口里，内核读了**——这是本域少见的"真实固件版本"来源，比 Adreno 那条只覆盖 pfp/pm4 的属性（第十三章之二）更接近硬件真相。

但它的出口比 Adreno 更窄。`mfc_debug` 的定义：

> `#define mfc_debug(level, fmt, args...)				\`
> `	do {							\`
> `		if (mfc_debug_level >= level)			\`
> `			printk(KERN_DEBUG "%s:%d: " fmt,	\`
> `				__func__, __LINE__, ##args);	\`
> `	} while (0)`
> — `s5p_mfc_debug.h:20-25`

版本号要求 `mfc_debug_level >= 2`，而 `mfc_debug_level` 的默认值是隐式的 0（`int mfc_debug_level;`，`s5p_mfc.c:38`，BSS 零值）。于是这条打印在默认配置下不出现。

这个域的特殊之处的在这里：门槛是一个**模块参数**，而且参数名与变量名不同：

> `module_param_named(debug, mfc_debug_level, int, 0644);`
> — `s5p_mfc.c:39`

三点读法：

- 参数对外叫 `debug`，不叫 `mfc_debug_level`——按变量名去 `/sys/module/.../parameters/` 找会一无所获。
- 模式 `0644`：**所有本地用户可读该参数，只有 root 可写**。所以"这台设备是否已经打开了 MFC 调试"是一个普通读取动作就能得到的事实；`echo 2 > .../debug` 则需要更高权限，本文不把它列为核验步骤（第 1 篇的权限面口径）。
- 该模块的可加载目标名在 `Makefile:2` 写作 `s5p-mfc.o`（`obj-$(CONFIG_VIDEO_SAMSUNG_MFC) += s5p-mfc.o`），短横线在 `/sys/module/` 下的实际拼写由构建期决定——**本文不在源码里断言这条路径，脚本 1 把它列为"实测枚举"而不是"假定存在"**。

即便打印出现，它仍然受第二重限制：这条 `printk` 落在内核日志里，而 `allow dumpstate kernel:system syslog_read;`（`system/sepolicy@main:private/dumpstate.te:211`）是全树唯一授予该权限的常规域，同时 `neverallow appdomain kernel:system { syslog_read syslog_mod syslog_console };`（`private/app.te:661`，注释即 `# Access to syslog(2) or /proc/kmsg.`，`:660`）。第十三章已经证明 shell 域没有一般 `sysfs:file` 权限；本章补充：**shell 也没有 `syslog_read`**。所以"MFC 固件版本"在本域的可达性是三层问题的合取：读不到文件清单 ∧ 打印默认关 ∧ 日志默认读不到。

还有一条时序限制：这段读寄存器在 `s5p_mfc_init_hw()` 的路径里（紧邻其上的 `mfc_err("Failed to init hardware\n");` 在 `:266`，`mfc_err("Failed to init firmware - error: %d int: %d\n", ...)` 在 `:275`），也就是**每次硬件初始化一次**，不是每次解码一次。第廿六章 K7 用它说明"采集窗口必须覆盖一次冷启动"。

### 十四之四 第三种"版本"：一个填了地址却从未取值的位置

`MFC v6+` 的寄存器表把固件版本寄存器登记成了一个槽位：

> `static struct s5p_mfc_regs mfc_regs;`
> `/* Initialize registers for MFC v6 onwards */`
> `const struct s5p_mfc_regs *s5p_mfc_init_regs_v6_plus(struct s5p_mfc_dev *dev)`
> — `s5p_mfc_opr_v6.c:2400-2403`
> `	#define S5P_MFC_REG_ADDR(dev, reg) ((dev)->regs_base + (reg))`
> `	#define R(m, r) mfc_regs.m = S5P_MFC_REG_ADDR(dev, r)`
> `	R(firmware_version, S5P_FIMV_FW_VERSION_V6);`
> — `s5p_mfc_opr_v6.c:2407-2408, 2417`

注意 `R()` 存的是**地址**（`regs_base + reg`），不是值。全目录检索 `firmware_version` 只有两处命中：上面这行 `R()`（`s5p_mfc_opr_v6.c:2417`）与字段声明 `void __iomem *firmware_version;`（`s5p_mfc_opr.h:28`）。**没有任何一处对 `mfc_regs->firmware_version` 做 `readl()`**——同文件里其它槽位都有配套读取函数（例如 `return readl(dev->mfc_regs->ret_instance_id);`，`s5p_mfc_opr_v6.c:2358`），唯独 `firmware_version` 没有。

所以本域存在一个源码里"看起来是固件版本出口"的符号，但它功能上为空。第廿三章把它列为编号陷阱的一例：**在源码里检索到 `firmware_version` 不等于设备上有这个读数**；台账若引用了这个符号，第卅章的取回记录必须能同时给出它的声明行与"无读取点"的检索证据。

### 十四之五 第四种"版本"根本不是版本，以及 MFC v12 的那条例外

`struct s5p_mfc_variant` 里的 `.version` / `.version_bit`（本章之一表格）描述的是**硬件代际**，来自设备树匹配数据，与固件无关。把它抄进"固件版本"栏是本域最常见的记录错误，因为它在日志里最显眼、最容易拿到。

真正会让"我上次看过一次成功装载"这条经验失效的是这段：

> `	/* In case of MFC v12, RET_SYS_INIT response from hardware fails due to`
> `	 * incorrect firmware transfer and therefore it is not able to initialize`
> `	 * the hardware. This causes failed response for SYS_INIT command when`
> `	 * MFC runs for second time. So, load the MFC v12 firmware for each run.`
> `	 */`
> `	if (!IS_MFCV12(dev))`
> `		if (dev->fw_get_done)`
> `			return 0;`
> — `s5p_mfc_ctrl.c:54-61`

`fw_get_done` 在成功拷贝后置真（`dev->fw_get_done = true;`，`:85`；置假在 `s5p_mfc_release_firmware()`，`:97`），字段注释逐字是 ` * @fw_get_done:	flag set when request_firmware() is complete and`（`s5p_mfc_common.h:299`，跨行未截）。**对 v12，这个标志被绕过：每次运行都重新走一遍装载。** 于是第三章 F5 那条"装载成功不等于镜像内容被验证"在本域有了加强形态：对 v12 而言，连"上一次装载成功"这个事实也不能外推到下一次运行。

失败侧的原句仍然完整可查，这是本域唯一稳定可见的证据面：

| 条件 | 源码行 | 原句 / 返回值 |
| --- | --- | --- |
| 所有候选文件名都失败 | `s5p_mfc_ctrl.c:74-77` | `mfc_err("Firmware is not present in the /lib/firmware directory nor compiled in kernel\n");` → `return -EINVAL;` |
| 镜像大于预留缓冲 | `:78-82` | `mfc_err("MFC firmware is too big to be loaded\n");` → `return -ENOMEM;` |
| 传输等待超时 | `:259` 附近 | `"Failed to send command to MFC - timeout\n"` |
| 硬件初始化失败 | `:265-269` | `"Failed to init hardware\n"` → `-EIO` |
| 固件响应带错误 | `:272-279` | `"Failed to init firmware - error: %d int: %d\n"` → `-EIO` |

`mfc_err` 无条件走 `printk(KERN_ERR ...)`（`s5p_mfc_debug.h:33-37`），与 `mfc_debug` 不同层——**"缺固件"一定能写进日志，"固件是哪版"默认写不进日志**。这条不对称是本域台账设计的根据：把"能观测到失败"当成"能观测到身份"是两个不同的结论。

### 十四之六 全目录检索（与第十章之二同法）

对 `s5p-mfc/` 全部 32 个 `.c`/`.h` 逐词检索，命中数如下：

| 检索词 | 命中文件数 | 命中行数 | 说明 |
| --- | --- | --- | --- |
| `MODULE_FIRMWARE` | **0** | **0** | 本驱动不向 modinfo 声明任何固件需求 |
| `request_firmware` | 2 | 2 | 一处真实调用（`s5p_mfc_ctrl.c:66`），一处是字段注释（`s5p_mfc_common.h:299`） |
| `fw_ver` | 3 | 5 | 写 1（`s5p_mfc_ctrl.c:69`）、声明 2、注释 1、读 1（`s5p_mfc_opr_v6.c:43`） |
| `fw_get_done` | 2 | 5 | `s5p_mfc_ctrl.c:60/85/97` + `s5p_mfc_common.h:299/347` |
| `S5P_FIMV_FW_VERSION` | 4 | 6 | 读值 2（`s5p_mfc_ctrl.c:282/284`）、写寄存器 1（`s5p_mfc_ctrl.c:133`）、地址槽 1（`s5p_mfc_opr_v6.c:2417`）、定义 2（两个 regs 头文件） |
| `debug_level` | 2 | 4 | 定义 `s5p_mfc.c:38-39`、外部声明与判断 `s5p_mfc_debug.h:18/22` |

`MODULE_FIRMWARE` 这一行是全报告跨域对比的一个免费读数：panthor 声明 1 条（`mali_csffw.bin`，第十一章）、Adreno 上游声明 **10 条**（`a6xx_catalog.c:1043-1052`，紧接第十二章之一那张目录表之后的 `MODULE_FIRMWARE("qcom/a615_zap.mbn");` 起连续十行）、MFC 0 条、Venus 0 条（本章下文）、`mtk_scp.c` 0 条。这个差异的实际后果不是"有没有固件"，而是**构建侧与 `modinfo` 侧能否发现这台设备需要固件**——第廿五章把"声明数 0 但需要固件"单列为归属矩阵里的一格，因为它的失败模式是"固件缺失被解释成硬件不存在"。

### 十四之七 本章结论

第十四章不需要推论就能给出四句可核对的话：

1. MFC 的固件选择是**文件名降序试错**（`s5p_mfc_ctrl.c:63-72`），试错结果 `fw_ver` 是数组下标（`s5p_mfc_common.h:165-168`），但它确实改变寄存器写入路径（`s5p_mfc_opr_v6.c:43/1774/1789`）。
2. 硬件上报的三位固件版本读到了（`s5p_mfc_ctrl.c:281-284`），却只送到 `mfc_debug(2, ...)`（`:286-287`），默认关闭；开关是名为 `debug` 的 0644 模块参数（`s5p_mfc.c:39`）。
3. `mfc_regs.firmware_version` 是填了地址从未取值的槽（`s5p_mfc_opr_v6.c:2417` + `s5p_mfc_opr.h:28`，全目录无 `readl`）。
4. MFC v12 每次运行重新装载（`s5p_mfc_ctrl.c:54-61`），所以"曾经装载成功"不是持久事实。

本域的可核验面因此只剩文件系统清单与内核日志两处，而这两处分别被目录权限和 `syslog_read` 挡住——第十三章的结论在这里以更强的形式重现。

## 十五、Venus 与 MediaTek：两种互不相通的媒体 DSP 装载链

高通的 venus 与联发科的 vcodec 都常被写成"视频固件"，但两条链在三个维度上相反：判定者（安全世界 vs 内核/远端处理器核）、固件名的来源（编译期资源表 + 设备树覆盖 vs 设备树覆盖 + 编译期默认）、以及**把固件版本带回来之后落在哪里**（venus：带回来了，但只写进一个内核结构体成员和一条默认关闭的 `dev_dbg`，见本章之四；mtk-vpu：带回来了，而且是一行 INFO 日志）。本章把三者对齐到第三章 F2 的装载层上。

出处约定：`kernel/common@android16-6.12` 的 `drivers/media/platform/qcom/venus/`（36 个文件，取回后全目录检索）、`drivers/media/platform/mediatek/vcodec/`（`common`/`decoder`/`encoder` 三个子目录）与 `drivers/media/platform/mediatek/vpu/mtk_vpu.c`（1055 行）、`drivers/remoteproc/mtk_scp.c`（1598 行）、`drivers/remoteproc/remoteproc_sysfs.c`。

### 十五之一 Venus：三步，判定权在安全世界，返回值在第二步被吞

固件 PAS 标识是硬编码的：`#define VENUS_PAS_ID			9`（`firmware.c:24`）。装载函数 `static int venus_load_fw(struct venus_core *core, const char *fwname, phys_addr_t *mem_phys, size_t *mem_size)`（`:82-83`）做三件事：

> `	ret = request_firmware(&mdt, fwname, dev);`
> — `firmware.c:110`
> `	if (core->use_tz)`
> `		ret = qcom_mdt_load(dev, mdt, fwname, VENUS_PAS_ID,`
> `				mem_va, *mem_phys, *mem_size, NULL);`
> `	else`
> `		ret = qcom_mdt_load_no_init(dev, mdt, fwname, VENUS_PAS_ID,`
> `				mem_va, *mem_phys, *mem_size, NULL);`
> — `firmware.c:135-140`

第三件事在调用者 `venus_boot()` 里：

> `	if (core->use_tz)`
> `		ret = qcom_scm_pas_auth_and_reset(VENUS_PAS_ID);`
> `	else`
> `		ret = venus_boot_no_tz(core, mem_phys, mem_size);`
> — `firmware.c:237-240`

`qcom_scm_pas_auth_and_reset()` 是第十二章之二那条"把镜像交给安全世界，由它决定能不能跑"的同一个入口；这里 PAS ID 从 GPU 的槽位换成 `9`。第三章 F2 表里"安全 world 授权装载"那一行原本引自上游 adreno 与 venus 两处的 `:136-141`，本节把它钉到 `firmware.c:135-140`（同一分支）。

关键的是错误处理：

> `	ret = venus_load_fw(core, fwpath, &mem_phys, &mem_size);`
> `	if (ret) {`
> `		dev_err(dev, "fail to load video firmware\n");`
> `		return -EINVAL;`
> `	}`
> — `firmware.c:228-232`

`venus_load_fw()` 内部会返回 `-EINVAL`（`:100`、`:107`、`:124`）、`-ENOMEM`（`:131`）以及 `qcom_mdt_load()` 原样透传的返回码，但调用者把非零一律改写成 `-EINVAL`。**"固件文件找不到"与"安全世界拒绝授权"在这一层被折叠成同一个值**，日志里也只留一句不带错误码的 `fail to load video firmware`。第廿一章的 FT 分类需要这一条：本域的失败不可分型。

另有两个"-EPROBE_DEFER 而不是装载失败"的入口，必须区别登记：

> `	if (!IS_ENABLED(CONFIG_QCOM_MDT_LOADER) ||`
> `	    (core->use_tz && !qcom_scm_is_available()))`
> `		return -EPROBE_DEFER;`
> — `firmware.c:219-221`

以及 `venus_boot_no_tz()` 里 `if (!dev) return -EPROBE_DEFER;`（`:156-157`）。它们的表现是**视频设备节点迟迟不出现**（第十七章会给出 `venus` 设备的枚举面），不是"固件缺失"。把 probe defer 记成固件缺陷是这类台账最常见的错误归因。

### 十五之二 固件名：资源表给默认，设备树可覆盖；目录相同扩展名可以不同

覆盖入口是标准的一个属性读：

> `	ret = of_property_read_string_index(dev->of_node, "firmware-name", 0,`
> `					    &fwpath);`
> `	if (ret)`
> `		fwpath = core->res->fwname;`
> — `firmware.c:223-226`

`core->res->fwname` 来自 `core.c` 的资源表，全部 9 条逐字如下（同一分支）：

| 资源结构 | 起始行 | `.fwname` | 行号 |
| --- | --- | --- | --- |
| `msm8916_res` | `:546` | `"qcom/venus-1.8/venus.mbn"` | `:559` |
| `msm8996_res` | `:576` | `"qcom/venus-4.2/venus.mbn"` | `:592` |
| `msm8998_res` | `:614` | `"qcom/venus-4.4/venus.mbn"` | `:630` |
| `sdm660_res` | `:668` | `"qcom/venus-4.4/venus.mdt"` | `:693` |
| `sdm845_res` | `:719` | `"qcom/venus-5.2/venus.mbn"` | `:738` |
| `sdm845_res_v2` | `:741` | `"qcom/venus-5.2/venus.mbn"` | `:768` |
| `sc7180_res` | `:791` | `"qcom/venus-5.4/venus.mbn"` | `:816` |
| `sm8250_res` | `:844` | `"qcom/vpu-1.0/venus.mbn"` | `:871` |
| `sc7280_res` | `:904` | `"qcom/vpu-2.0/venus.mbn"` | `:933` |

三个台账要点：9 条资源条目只有 **8 个不同路径**（`venus-5.2` 被 `sdm845_res` 与 `sdm845_res_v2` 共用，`:738` 与 `:768`）；`qcom/venus-4.4/` 这一个目录在两条记录里分别以 `.mbn`（`:630`）和 `.mdt`（`:693`）出现，**扩展名不同意味着走的是不同解析路径**（`.mdt` 是分段/认证描述符，第十二章之一已就 adreno 的 `.zapfw` 说过同一件事）；目录名里的 `venus-1.8` / `venus-4.4` / `vpu-2.0` 是**硬件代际标签**，不是固件版本。第廿三章的编号陷阱条目据此给出一条归一化规则：媒体固件的路径键必须保留扩展名，不能只按目录归一。

### 十五之三 `use_tz` 不是配置项，是"设备树里有没有某个子节点"

决定走安全世界还是走内核自建映射的，是一个存在性判断：

> `	np = of_get_child_by_name(core->dev->of_node, "video-firmware");`
> `	if (!np) {`
> `		core->use_tz = true;`
> `		return 0;`
> `	}`
> — `firmware.c:291-295`

即**没有 `video-firmware` 子节点 = 用 TrustZone**。字段自己的注释连措辞都是保留的：

> ` * @use_tz:	a flag that suggests presence of trustzone`
> — `core.h:149`（字段声明 `unsigned int use_tz;` 在 `:203`）

"suggests"（暗示）而不是"confirms"（确认）——驱动自己承认它只是从一个 DT 形状推断安全世界的存在。这对归属分析的直接后果是：**同一份内核镜像，在两种 DT 布局下固件身份的可见面完全不同**。`use_tz` 路径上没有任何用户态返回值（授权由 TZ 判定，`firmware.c:238`）；`!use_tz` 路径上反而有明确的映射错误（`dev_err(dev, "could not map video firmware region\n");`，`:165`；`dev_err(dev, "failed to unmap firmware\n");`，`:202`）。

`use_tz` 还会触发一段只在保护模式下才做的地址范围登记：

> `	if (core->use_tz && res->cp_size) {`
> `		/*`
> `		 * Clues for porting using downstream data:`
> `		 * cp_start = 0`
> — `firmware.c:245-248`

其内的 `qcom_scm_mem_protect_video_var(...)`（`:256-259`）失败时会先 `qcom_scm_pas_shutdown(VENUS_PAS_ID);`（`:261`）再报 `"set virtual address ranges fail (%d)\n"`（`:262-263`）——**授权已完成的 DSP 被就地关闭**。这条与本域其它"失败只留日志"的路径不同：它有一个可观测的功能后果（视频解码整体不可用），因此第十七章把"venus 设备存在但解码失败"与"venus 设备不存在"分成两个读数。

### 十五之四 venus 有一处固件版本回读，但它只活在内核结构体里

先给一张检索计数表——对 `qcom/venus/` 全部 `.c`/`.h`（取回 37 个文件）逐词计数：

| 检索词 | 命中 | 判读 |
| --- | --- | --- |
| `MODULE_FIRMWARE` | **0** | 与 MFC 同：`modinfo` 不提示需要固件 |
| `request_firmware` | **1** | 只有 `firmware.c:110` 一处 |
| `FW_VERSION` / `fw_version` / `fw_ver` / `core_ver` / `firmware_ver` | **全 0** | 这套常见命名在 venus 里确实不存在 |
| `venus_ver` | **12** | 结构体字段与三次解析写入；本节主题 |
| `image_version` | **7** | HFI 属性与报文构造 |
| `version` | 两类混合 | 一类指硬件代际（`.hfi_version = HFI_VERSION_1XX/3XX/4XX/6XX`、`.vpu_version = VPU_VERSION_AR50/IRIS2/IRIS2_1`，`core.c:554, 587, 625, 684, 732-733, 758-759, 806-807, 864-865, 922-923`），一类指下面这条固件版本链 |

这一节要写的是本章最需要防错的一处。**把 `fw_version` 这一族命名当作唯一检索入口，就会得出"venus 没有固件版本"的错误结论**——这条纠错本身就是第廿三章"检索词即结论"纪律的来源。真实情况是：驱动在核心初始化时向固件主动索取版本字符串，并且**这个字符串是固件自报的**。

发送侧只有四行报文构造：

> `void pkt_sys_image_version(struct hfi_sys_get_property_pkt *pkt)`
> `{`
> `	pkt->hdr.size = sizeof(*pkt);`
> `	pkt->hdr.pkt_type = HFI_CMD_SYS_GET_PROPERTY;`
> `	pkt->num_properties = 1;`
> `	pkt->data = HFI_PROPERTY_SYS_IMAGE_VERSION;`
> `}`
> — `hfi_cmds.c:154-160`

属性号 `#define HFI_PROPERTY_SYS_IMAGE_VERSION 0x6`（`hfi_helper.h:428`），命令号 `#define HFI_CMD_SYS_GET_PROPERTY 0x10006`（`hfi_cmds.h:17`）。调用点在核心初始化里，紧跟 `sys_init` 之后：

> `	pkt_sys_image_version(&version_pkt);`
> `	ret = venus_iface_cmdq_write(hdev, &version_pkt, false);`
> `	if (ret)`
> `		dev_warn(dev, "failed to send image version pkt to fw\n");`
> — `hfi_venus.c:1175-1179`

注意这个 `if (ret)`：**索取版本失败只留一条 `dev_warn`，初始化继续**（函数后面直接走 `venus_sys_set_default_properties()`，`:1181-1182`）。也就是说设备可以完全正常地工作而在内核里没有任何版本值。

接收侧是这个域里最坦率的一段代码（`static void sys_get_prop_image_version()`，`hfi_msgs.c:283-331`）。它先做三重合法性检查：

> `	req_bytes = pkt->hdr.size - sizeof(*pkt);`
> `	if (req_bytes < VER_STR_SZ || !pkt->data[0] || pkt->num_properties > 1)`
> `		/* bad packet */`
> `		return;`
> — `hfi_msgs.c:294-298`

宏取值 `#define SMEM_IMG_VER_TBL 469`、`#define VER_STR_SZ 128`、`#define SMEM_IMG_OFFSET_VENUS (14 * 128)`（`hfi_msgs.c:18-20`）。然后是三次连续尝试用**硬编码前缀**去 sscanf 固件给的字符串：

> `	ret = sscanf(img_ver, "14:video-firmware.%u.%u-%u",`
> `		     &core->venus_ver.major, &core->venus_ver.minor, &core->venus_ver.rev);`
> `	ret = sscanf(img_ver, "14:VIDEO.VPU.%u.%u-%u", ...`
> `	ret = sscanf(img_ver, "14:VIDEO.VE.%u.%u-%u", ...`
> — `hfi_msgs.c:304-316`（三种格式分别在 `:304`、`:309`、`:314`）

三种都不中就走这条失败分支并**什么都不留**：

> `	dev_err(dev, VDBGL "error reading F/W version\n");`
> `	return;`
> — `hfi_msgs.c:319-320`

成功则只有两个出口，都不面向设备侧采集：

> `	dev_dbg(dev, VDBGL "F/W version: %s, major %u, minor %u, revision %u\n",`
> `		img_ver, core->venus_ver.major, core->venus_ver.minor, core->venus_ver.rev);`
> `	smem_tbl_ptr = qcom_smem_get(QCOM_SMEM_HOST_ANY,`
> `		SMEM_IMG_VER_TBL, &smem_blk_sz);`
> `	if (!IS_ERR(smem_tbl_ptr) && smem_blk_sz >= SMEM_IMG_OFFSET_VENUS + VER_STR_SZ)`
> `		memcpy(smem_tbl_ptr + SMEM_IMG_OFFSET_VENUS,`
> `		       img_ver, VER_STR_SZ);`
> — `hfi_msgs.c:323-330`

四个台账级事实由此定死：

1. **命名**：`struct venus_img_version { u32 major; u32 minor; u32 rev; } venus_ver;`（`core.h:233-237`，字段注释 ` * @venus_ver:	the venus firmware version` 在 `core.h:173`）。这是内核结构体里的一个成员，**没有 sysfs 属性、没有 ioctl、没有 netlink 出口**——整个目录里 `venus_ver` 的 12 次命中全部是声明、sscanf 写入和下面两个门函数的读取。
2. **可见性**：唯一的文本出口是 `dev_dbg`。本章之五已就 MFC 说明 `dev_dbg` 需要动态调试或编译期开启；这里同一约束成立，且叠加 shell 无 `syslog_read`（第十四章之三）。**换句话说 venus 是全媒体域唯一"内核知道版本但连日志默认都不打"的链。**
3. **可信度**：值来自固件报文，属于**自报**。解析规则是三条硬编码字符串前缀，厂商镜像命名一旦换写法就落到 `:319` 那条 `dev_err`，此时 `venus_ver` 保持上次的值或 0——**读到 0/读到旧值与"这台设备的固件是某个版本"之间没有可推关系**。这是第廿六章判定模型里"L2-WARN 而不是 L1-INFO"的原型案例。
4. **副作用面**：这个版本只被两个门函数消费，且全树只有两个调用点：

> `static inline bool`
> `is_fw_rev_or_newer(struct venus_core *core, u32 vmajor, u32 vminor, u32 vrev)`
> `static inline bool`
> `is_fw_rev_or_older(struct venus_core *core, u32 vmajor, u32 vminor, u32 vrev)`
> — `core.h:554-567`

调用点是 `vdec.c:577`（`if (IS_IRIS2(inst->core) && is_fw_rev_or_older(inst->core, 1, 0, 87))`，注释逐字为 `/* Send NULL EOS addr for only IRIS2 (SM8250),for firmware <= 1.0.87.` / ` * SC7280 also reports "1.0.<hash>" parsed as 1.0.0; restricting to IRIS2` / ` * avoids misapplying this quirk and breaking VP9 decode on SC7280.`，`vdec.c:571-575`）与 `vdec.c:711`（`if (is_fw_rev_or_newer(inst->core, 5, 4, 51)) {`，上一行注释 `/* Enabling sufficient sequence change support for VP9 */`，`:710`）。**两点全是 VP9/EOS 的行为微调，没有任何一处与安全补丁级别相关。** 那条 SC7280 注释尤其值得抄进台账：`1.0.<hash>` 被解析成 `1.0.0`，说明**同一份版本字段在不同镜像命名规则下会失去可比性**——这正是第廿六章禁止"跨厂商比较版本串"的依据。

至于 SMEM：`memcpy` 的目标是共享内存里的一张镜像版本表（项号 469，偏移 `14*128`），本次取回的文件里没有任何把它暴露给用户空间的接口，SMEM 在设备侧的可读性属于第廿九章"不可断言"清单，本文不据此声称任何读数。

另一半结论仍然成立，而且和上面这条不矛盾：`.hfi_version` 与 `.vpu_version` 全部是**编译期写死在资源表里、由设备树匹配挑选**（`core.c:316` `core->res = of_device_get_match_data(dev);` → `core.c:322` `core->pm_ops = venus_pm_get(core->res->hfi_version);`，资源表在 `core.c:554/587/625/684/732/758/806/864/922`，compatible 表从 `core.c:937` 起）的**主机侧接口代际**，运行时不可能变，也不反映镜像。**所以 venus 有两条都叫"版本"的东西：一条是主机自己编译期选的协议代际（可读但无关于镜像），一条是固件自报的镜像版本（有关镜像但无出口）。** 台账最常见的错误是把前者抄进"固件版本"栏——它比后者容易拿到得多。

因此媒体 DSP 的版本归属在 venus 这一格只能落在 F2（文件系统清单）与 F6（构建/镜像装配）两层，运行时层有值但无采集面（第廿五章矩阵按此标注；第十八章给出门函数与命名链的完整对齐）。

### 十五之五 `fw_level`：一个专门用来误读的节点

`venus_dbgfs` 一共三行有效代码：

> `	core->root = debugfs_create_dir("venus", NULL);`
> `	debugfs_create_x32("fw_level", 0644, core->root, &venus_fw_debug);`
> — `dbgfs.c:17-18`

名字最容易读错的那个字段其实是 HFI 调试掩码：

> `int venus_fw_debug = HFI_DEBUG_MSG_ERROR | HFI_DEBUG_MSG_FATAL;`
> — `hfi_venus.c:133`
> `	ret = venus_sys_set_debug(hdev, venus_fw_debug);`
> `	if (ret)`
> `		dev_warn(dev, "setting fw debug msg ON failed (%d)\n", ret);`
> — `hfi_venus.c:947-949`（在 `venus_sys_set_default_properties()` 内，`:941`）

也就是说：debugfs 里那个 0644 的 `fw_level` 是"把 DSP 的调试消息级别设成什么"，写它不会告诉你任何版本信息，读它只读到一个掩码。第廿九章把它列为误区之一（"看到 fw 字样的节点就以为是固件版本/等级"）。可达性另有一层：`system/sepolicy@main` 的 `private/shell.te` 只授予 shell 若干专项 debugfs 类型（`:7` `allow shell debugfs_tracing_debug:dir r_dir_perms;`、`:8-9` tracing 目录/文件、`:10` `debugfs_trace_marker:file getattr;`、`:14` 条件块内 `debugfs_tracing_debug:file rw_file_perms;`），**没有 venus 的 debugfs 类型**——所以标准 adb 姿势下这个节点连读都不该假设能读，脚本 1 把它归入"预期失败并记录 errno"的一格。

### 十五之六 MediaTek：同一个驱动两个后端，选哪个只看设备树里有一个什么属性

联发科的解码器不直接决定装载方式，它把这件事抽成一张 ops 表：

> `struct mtk_vcodec_fw_ops {`
> `	int (*load_firmware)(struct mtk_vcodec_fw *fw);`
> — `common/mtk_vcodec_fw_priv.h:19-20`

分发只有六行：

> `	switch (type) {`
> `	case VPU:`
> `		return mtk_vcodec_fw_vpu_init(priv, fw_use);`
> `	case SCP:`
> `		return mtk_vcodec_fw_scp_init(priv, fw_use);`
> `	default:`
> `		dev_err(&plat_dev->dev, "Invalid vcodec fw type");`
> `		return ERR_PTR(-EINVAL);`
> `	}`
> — `common/mtk_vcodec_fw.c:17-25`（`mtk_vcodec_fw_load_firmware()` 只是 `return fw->ops->load_firmware(fw);`，`:35-39`）

`type` 从哪来？一个属性存在性判断：

> `	if (!of_property_read_u32(pdev->dev.of_node, "mediatek,vpu",`
> `				  &rproc_phandle)) {`
> `		fw_type = VPU;`
> `	} else if (!of_property_read_u32(pdev->dev.of_node, "mediatek,scp",`
> `					 &rproc_phandle)) {`
> `		fw_type = SCP;`
> `	} else {`
> `		dev_dbg(&pdev->dev, "Could not get vdec IPI device");`
> `		return -ENODEV;`
> `	}`
> — `decoder/mtk_vcodec_dec_drv.c:377-386`

两个后端"装载"这两个字的含义完全不同：

> `static int mtk_vcodec_scp_load_firmware(struct mtk_vcodec_fw *fw)`
> `{`
> `	return rproc_boot(scp_get_rproc(fw->scp));`
> — `common/mtk_vcodec_fw_scp.c:7-9`
> `static int mtk_vcodec_vpu_load_firmware(struct mtk_vcodec_fw *fw)`
> `{`
> `	return vpu_load_firmware(fw->pdev);`
> — `common/mtk_vcodec_fw_vpu.c:7-9`

SCP 后端**根本不碰固件文件**，它只是把一个已经被 remoteproc 注册好的远端核启动起来；VPU 后端才走 `request_firmware()`。这是本章最重要的一条结构性对比：**同一个 `/dev/video*` 节点，在一台机器上"固件装载"意味着 remoteproc 启动，在另一台上意味着两个 .bin 文件被拷进 TCM**。

两个后端还都是编译期可选的，所以二进制里可能只存在一个：

> `ifneq ($(CONFIG_VIDEO_MEDIATEK_VCODEC_VPU),)`
> `mtk-vcodec-common-y += mtk_vcodec_fw_vpu.o`
> `endif`
> `ifneq ($(CONFIG_VIDEO_MEDIATEK_VCODEC_SCP),)`
> `mtk-vcodec-common-y += mtk_vcodec_fw_scp.o`
> `endif`
> — `common/Makefile:9-15`

未选中的那一侧退化为固定返回 `-ENODEV` 的内联桩（`mtk_vcodec_fw_priv.h:32-50`，两段各是 `#if IS_ENABLED(...)` 加 `return ERR_PTR(-ENODEV);`）。平台文档给出的分界在 Kconfig 帮助文本里，逐字：

> `	  Note that support for MT8173 requires VIDEO_MEDIATEK_VPU to`
> `	  also be selected. Support for MT8183 depends on MTK_SCP.`
> — `vcodec/Kconfig:32-33`

第十章为 Arm GPU 立的那条"必须按型号回答"的规则，在这里以构建配置的形式重现：MT8173 与 MT8183 的媒体固件到达方式不同族。

装载时机是另一条与直觉相反的细节。固件不是在驱动 probe 时装的，而是在**第一个解码实例打开时**：

> `	if (v4l2_fh_is_singular(&ctx->fh)) {`
> `		/*`
> `		 * Does nothing if firmware was already loaded.`
> `		 */`
> `		ret = mtk_vcodec_fw_load_firmware(dev->fw_handler);`
> `		if (ret < 0) {`
> `			/*`
> `			 * Return 0 if downloading firmware successfully,`
> `			 * otherwise it is failed`
> `			 */`
> `			mtk_v4l2_vdec_err(ctx, "failed to load firmware!");`
> — `decoder/mtk_vcodec_dec_drv.c:250-260`

对台账的含义（与第十四章之五 MFC v12 那条并列）：**"设备开机时固件是否已在"不是一个有意义的问题**——SCP 路径上远端核可能在任何人打开 `/dev/video*` 之前一直 offline（第十七章给出 `state` 读数），而打开一次之后它就 `running` 并在此后一直被复用。第廿六章 K7 的采集窗口据此定义为"覆盖一次真实的解码会话"，而不是"覆盖一次开机"。

### 十五之七 VPU 后端：一次读回固件自报的版本字符串，并打进行号级 INFO

`vpu_load_firmware()` 的实现就在本树里（`drivers/media/platform/mediatek/vpu/mtk_vpu.c`，1055 行），这是本域唯一"成功之后把版本号写进日志"的媒体装载链，值得整段读：

> `	ret = request_firmware(&vpu_fw, fw_new_name, vpu->dev);`
> `	if (ret < 0) {`
> `		dev_info(vpu->dev, "Failed to load %s, %d, retry\n",`
> `			 fw_new_name, ret);`
> `		ret = request_firmware(&vpu_fw, fw_name, vpu->dev);`
> `		if (ret < 0) {`
> `			dev_err(vpu->dev, "Failed to load %s, %d\n", fw_name,`
> `				ret);`
> — `mtk_vpu.c:512-520`

候选名只有两个、且是硬编码的一对：

> `#define VPU_P_FW		"vpu_p.bin"`
> `#define VPU_D_FW		"vpu_d.bin"`
> `#define VPU_P_FW_NEW		"mediatek/mt8173/vpu_p.bin"`
> `#define VPU_D_FW_NEW		"mediatek/mt8173/vpu_d.bin"`
> — `drivers/media/platform/mediatek/vpu/mtk_vpu.c:48-51`

与 MFC 的下标试错同构（先新路径、失败退旧文件名），但方向是**升序**：`load_requested_vpu()` 先试 `_NEW` 再试裸名（`:512-517`），并且这个函数被调用**两次**——程序镜像与数据镜像（`ret = load_requested_vpu(vpu, P_FW);`（`:597`）与 `D_FW`（`:604`）），失败原句分别是 `"Failed to request %s, %d\n"` 携 `VPU_P_FW`（`:599`）与 `VPU_D_FW`（`:606`）。尺寸异常另有一句：`dev_err(vpu->dev, "fw %s size %zu is abnormal\n", ...)` → `-EFBIG`（`:526-529`）。

版本回读这一段是本报告在媒体域的正面样本：

> `	vpu->fw_loaded = true;`
> `	/* boot up vpu */`
> `	vpu_cfg_writel(vpu, 0x1, VPU_RESET);`
> — `mtk_vpu.c:610-612`
> `	ret = 0;`
> `	dev_info(dev, "vpu is ready. Fw version %s\n", run->fw_ver);`
> — `mtk_vpu.c:627-628`

`fw_ver` 不是整数而是**固件自己送上来的一段字符串**：初始化 IPI 的回调把它拷进驱动结构——

> `static void vpu_init_ipi_handler(void *data, unsigned int len, void *priv)`
> `{`
> `	...`
> `	strscpy(vpu->run.fw_ver, run->fw_ver, sizeof(vpu->run.fw_ver));`
> — `mtk_vpu.c:638-644`

字段声明 `char fw_ver[VPU_FW_VER_LEN];`（`:147`，注释 ` * @fw_ver:		VPU firmware version` 在 `:138`），长度 `#define VPU_FW_VER_LEN		16`（`:31`）。第十一章 Mali CSF 与第十二章 GMU 用的是同一种机制（固件自报、内核转印），而**这是本域唯一一次该机制在媒体加速器上真的留下了一条 INFO 日志**：级别是 `dev_info`，不受 MFC 那种 `mfc_debug_level >= 2` 门限约束，因此默认就会写进内核日志。

于是本域出现一条完整可核验的证据链，同时也暴露一条完整不可核验的死角：证据存在于 `dmesg`（第十七章 V6 面），而 shell 没有 `syslog_read`（`private/app.te:661` 的 neverallow 只挡 appdomain；`private/shell.te` 全文无 `kernel:system` 授权，`allow dumpstate kernel:system syslog_read;` 在 `private/dumpstate.te:211`）——**这条版本日志在有 adb 的现场基本读不到，在有 bugreport 的现场读得到**。第廿六章 K3/K8 的口径据此分列：一条读数"存在"与一条读数"能在你的采集姿势下取到"是两个指标。

同一版本串还被 debugfs 复用：`vpu->run.fw_ver` 出现在 `:689` 的格式化输出里，整段处在 `#ifdef CONFIG_DEBUG_FS`（`:650`）下的 `vpu_debug_read()`（`:651`）中——又一个"要看得到得先编译进去、再要读得到得先过 sepolicy"的两重门。

### 十五之八 SCP 后端：唯一自带标准 sysfs 读数面的媒体固件

SCP 侧的固件名与所有前面几章都不同——它由 remoteproc 核心持有，因此**自动获得 sysfs 出口**：

> `	const char *fw_name = "scp.img";`
> `	ret = rproc_of_parse_firmware(dev, 0, &fw_name);`
> `	if (ret < 0 && ret != -EINVAL)`
> `		return ERR_PTR(ret);`
> `	rproc = devm_rproc_alloc(dev, np->name, &scp_ops, fw_name, sizeof(*scp));`
> — `mtk_scp.c:1122-1130`

`rproc_of_parse_firmware()` 本体只是设备树属性读取的四行封装：

> `int rproc_of_parse_firmware(struct device *dev, int index, const char **fw_name)`
> `{`
> `	int ret;`
> `	ret = of_property_read_string_index(dev->of_node, "firmware-name",`
> `					    index, fw_name);`
> `	return ret ? ret : 0;`
> `}`
> — `drivers/remoteproc/remoteproc_core.c:998-1005`

两点值得注意：属性名与 venus 完全相同（`firmware-name`，索引起点也是 0），但 venus 是"读失败就用编译期资源表"（`firmware.c:223-226`），SCP 是"读失败就保留编译期默认 `"scp.img"`"（`:1122-1126`）——语义相同、代码不同；以及 `-EINVAL`（属性不存在）在这里被**当作正常情形吞掉**（`:1127`），所以"这台设备的 SCP 用的是默认名"这个事实不产生任何日志。

`devm_rproc_alloc(dev, np->name, ...)` 的第二个参数还决定了一个读数：sysfs 上的 `name` 就是**设备树节点名**（`static ssize_t name_show(...)` → `return sprintf(buf, "%s\n", rproc->name);`，`remoteproc_sysfs.c:220-227`，`DEVICE_ATTR_RO(name)`）。第十七章会把 `firmware` / `state` / `name` 三个节点作为跨域统一读数面处理，包括那条最重要的降级：远端核由外部启动时 `firmware` 直接显示 `unknown`（`:149-150`）。

顺带一条本域独有的对照：`mtk_scp.c` 全文 `MODULE_FIRMWARE` 命中 0——固件名 `"scp.img"` 是普通字符串常量，不是声明。所以第十四章之六那条"modinfo 看不出需要固件"的观察，在 SCP 路径上同样成立。

### 十五之九 四条装载链对齐

| | MFC（Exynos） | Venus（高通） | vcodec-SCP（联发科） | vcodec-VPU（联发科 MT8173） |
| --- | --- | --- | --- | --- |
| 装载调用 | `request_firmware()` ×降序试错（`s5p_mfc_ctrl.c:63-72`） | `request_firmware()` → `qcom_mdt_load()` → `qcom_scm_pas_auth_and_reset()`（`firmware.c:110/136/238`） | `rproc_boot()`（`mtk_vcodec_fw_scp.c:9`） | `request_firmware()` ×2 镜像（`mtk_vpu.c:512/517`，P 与 D 各一轮） |
| 谁判定镜像可否运行 | 无人判定（文件在就用） | 安全世界（PAS ID 9，`firmware.c:24`） | remoteproc 核心 + 固件 ELF 解析（`scp_load()`，`mtk_scp.c:661-685`） | 无人判定，直接 `memcpy` 进 TCM（`mtk_vpu.c:548`） |
| 失败原句 | `"Firmware is not present in the /lib/firmware directory nor compiled in kernel"`（`s5p_mfc_ctrl.c:75`） | `"fail to load video firmware"`（`firmware.c:230`，错误码被改写） | 无（`-EINVAL` 被吞，`mtk_scp.c:1127`） | `"Failed to load %s, %d, retry"` / `"Failed to request %s, %d\n"`（`mtk_vpu.c:514/599/606`） |
| 固件版本有无出口 | 读到寄存器但只进 debug（`s5p_mfc_ctrl.c:281-287`） | 固件自报字符串 → 内核结构体 + `dev_dbg` + SMEM，无采集面（`hfi_msgs.c:304-330`） | 名字可经 sysfs 读，版本不可 | **`dev_info` 一行字符串**（`mtk_vpu.c:628`） |
| 设备树是否可改固件名 | 否（变体表写死） | 是（`firmware-name`，`firmware.c:223`） | 是（`firmware-name`，`remoteproc_core.c:1002-1003`） | 否（`mtk_vpu.c:48-51` 写死） |

### 十五之十 本章结论

同一句"这台设备的媒体固件是什么版本"，在四条链上得到四种性质的答案：一个被关在调试开关后面（MFC）、一个由固件自报后被解析进内核结构体却没有任何采集面（Venus，本章之四）、一个只能回答文件名而不能回答内容（SCP）、一个写在默认级别的日志里但标准采集姿势读不到（VPU）。第十九章 V1–V8 的读数面框架与第十七章的统一枚举，就是为了让这四种答案在台账里落成四种不同的字段而不是同一个"未知"。
## 十六、公告词表与源码对象的逐词对齐：对得上的、半对的、根本对不上的

第五章之四给出了公告 Subcomponent 列的实测取值集合，并在观察 1 里写下这句话：**"代号不等于对象……把它们与本文六层对齐时只能靠驱动源码，不能靠词面。"** 本章就是兑现这句话：把每一个媒体与加速器侧的词放到源码面前，逐词给出"对得上 / 半个 / 对不上"三档判定，并把判定的证据类型固定下来。

### 十六之一 对齐方法：四类可复核证据与三档判定

一个公告词只有落到下面四类证据上才算"对齐"：

| 证据类 | 具体形态 | 本章用到它的地方 |
| --- | --- | --- |
| E1 目录与文件名 | 内核树里的驱动目录、`.c`/`.h` 文件名 | `drivers/media/platform/mediatek/{vcodec,vpu,mdp,mdp3,jpeg}`、`drivers/media/platform/qcom/venus/` |
| E2 compatible 串 | `of_device_id` 表里的字符串 | `qcom_q6v5_pas.c` 66 条 `-pas` 串、`mtk_scp_of_match` |
| E3 驱动内的字符串/枚举/常量 | IPI 枚举、格式位、镜像名、日志原句 | `enum scp_ipi_id`、`enum mtk_vdec_format_types`、`.firmware_name = "cdsp.mdt"` |
| E4 面向用户空间的出口 | sysfs / debugfs / 字符设备 / 默认级别日志 | 第十七章整章、`mtk_vpu.c:628` |

判定分三档：

- **A 对得上**：词面背后有唯一驱动对象，且该对象有 E4 出口 → 台账可以落器件名。
- **B 半个**：有驱动对象，但词的边界与对象边界不重合（一词多对象，或对象含词没有说的东西）→ 台账落"对象集合 + 边界不一致说明"。
- **C 对不上**：本层树内不存在对应实体（词属于别的层、别的仓库，或纯粹是占位串）→ 台账只能落词面本身，并显式标记 `align_class = C`。

三档不是修辞。**C 档必须能被脚本独立复现为"查不到"**，否则第廿七章脚本 1 无法验收。

### 十六之二 `vdec`、`hevc decoder`：一个词的真身是 17 个通道前缀

联发科侧最容易误植的两个词是 `vdec` 与 `hevc decoder`。它们在 GKI 树里的真身不是某个器件，而是 `include/linux/remoteproc/mtk_scp.h` 里的一段 IPI 编号表（`kernel/common@android16-6.12`，逐字）：

> `enum scp_ipi_id {`
> `	SCP_IPI_INIT = 0,`
> `	SCP_IPI_VDEC_H264,`
> `	SCP_IPI_VDEC_VP8,`
> `	SCP_IPI_VDEC_VP9,`
> `	SCP_IPI_VENC_H264,`
> `	SCP_IPI_VENC_VP8,`
> `	SCP_IPI_MDP_INIT,`
> `	SCP_IPI_MDP_DEINIT,`
> `	SCP_IPI_MDP_FRAME,`
> `	SCP_IPI_DIP,`
> `	SCP_IPI_ISP_CMD,`
> `	SCP_IPI_ISP_FRAME,`
> `	SCP_IPI_FD_CMD,`
> `	SCP_IPI_VDEC_LAT,`
> `	SCP_IPI_VDEC_CORE,`
> `	SCP_IPI_IMGSYS_CMD,`
> `	SCP_IPI_NS_SERVICE = 0xFF,`
> `	SCP_IPI_MAX = 0x100,`
> `};`
> — `kernel/common@android16-6.12:include/linux/remoteproc/mtk_scp.h:29-49`

数一下词面 `vdec` 在这张表里的覆盖：`VDEC_H264`、`VDEC_VP8`、`VDEC_VP9`、`VDEC_LAT`、`VDEC_CORE` 五条，另加 `SCP_IPI_INIT` 文档注释里那句带拼写错误的话——

> ` * @SCP_IPI_INIT:	 The interrupt from scp is to notfiy kernel`
> — 同文件 `:19`

**`notfiy` 是树内原样的拼写错误，本文逐字保留**。这条看起来像花絮，实际是判据：这张表是厂商内部通讯协议的文字化，不是对外器件命名规范。用词面去反推器件，等于用内部通道名去猜硬件。

第二层词表是能力位。公告里的 `hevc decoder` 在源码里的对应物是一个 32 位能力字的某一位（`drivers/media/platform/mediatek/vcodec/decoder/mtk_vcodec_dec_drv.h:35-45`，逐字）：

> `enum mtk_vdec_format_types {`
> `	MTK_VDEC_FORMAT_MM21 = 0x20,`
> `	MTK_VDEC_FORMAT_MT21C = 0x40,`
> `	MTK_VDEC_FORMAT_H264_SLICE = 0x100,`
> `	MTK_VDEC_FORMAT_VP8_FRAME = 0x200,`
> `	MTK_VDEC_FORMAT_VP9_FRAME = 0x400,`
> `	MTK_VDEC_FORMAT_AV1_FRAME = 0x800,`
> `	MTK_VDEC_FORMAT_HEVC_FRAME = 0x1000,`
> `	MTK_VCODEC_INNER_RACING = 0x20000,`
> `	MTK_VDEC_IS_SUPPORT_10BIT = 0x40000,`
> `};`

于是 `hevc decoder` = `0x1000` 这一位。这个字从哪来是本章的关键：**它由固件自己上报**。VPU 后端与 SCP 后端各有一条同名 getter：

> `static unsigned int mtk_vcodec_scp_get_vdec_capa(struct mtk_vcodec_fw *fw)`
> `{`
> `	return scp_get_vdec_hw_capa(fw->scp);`
> `}`
> — `.../vcodec/decoder/mtk_vcodec_fw_scp.c:12-15`

`mtk_vcodec_fw_vpu.c:14` 是同一行的 VPU 版本（`return vpu_get_vdec_hw_capa(fw->pdev);`）。两个后端把同一个词表交进同一个调用点：

> `		dev->dec_capability =`
> `			mtk_vcodec_fw_get_vdec_capa(dev->fw_handler);`
> `		mtk_v4l2_vdec_dbg(0, ctx, "decoder capability %x", dev->dec_capability);`
> — `.../vcodec/decoder/mtk_vcodec_dec_drv.c:264-267`

第三层词表是内存端口语义：`include/dt-bindings/memory/mediatek,mt8188-memory-port.h:453-462` 里是 `/* LARB 21 -- VDEC-CORE0 */` 与 `M4U_PORT_L21_HW_VDEC_*` 常量。

**判定：`vdec` 与 `hevc decoder` 都是 B 档。** 三套词（IPI 通道名 / 能力位 / M4U 端口名）指向同一个功能域但没有一个公共主键；公告的一个词在这三套里各自只占一小片。台账应落"对象集合 = {SCP_IPI_VDEC_*、MTK_VDEC_FORMAT_* 位、LARB 端口}"，并写明"该词不含 VENC_*"。

### 十六之三 `display` / `Display`：一词覆盖三层，且大小写分属两厂

联发科 `display` 在树里对应两组完全不同的东西：`drivers/media/platform/mediatek/mdp` 与 `mdp3`（MDP = Media Data Path，图像后处理），以及 `drivers/gpu/drm/mediatek/mtk_disp_*.c`（显示控制器组件，`mtk_disp_ovl.c`、`mtk_disp_rdma.c`、`mtk_dpi.c`、`mtk_dsi.c`、`mtk_hdmi.c` 等）。前者的固件侧接口是另一套 IPI 契约：

> `struct img_ipi_frameparam {`
> …
> `struct img_ipi_param {`
> — `drivers/media/platform/mediatek/mdp3/mtk-img-ipi.h:84 / :106`

`mdp3` 目录里的 `mtk-mdp3-vpu.c` 与 `mtk-mdp3-core.c` 通过 `SCP_IPI_IMGSYS_CMD`（`mtk_scp.h:46`）与协处理器固件通讯；`SCP_IPI_MDP_INIT/DEINIT/FRAME`（`:36-38`）是上一代通路。也就是说：**公告里一个 `display`，在源码里同时可以是"显示控制器驱动"（不跑固件）与"MDP 图像管线 + 协处理器固件通道"（跑固件）**，而这两类东西的安全属性完全不同——前者是寄存器访问，后者是 F1/F2 层的镜像执行。

高通侧 `Display`（首字母大写，见第五章之四）落在 `drivers/gpu/drm/msm/disp/` 一族，与本文六层的对齐关系在第廿五章的归属矩阵里给出，本章只钉住大小写这条事实：**同一份资产映射脚本若按字符串精确匹配，会把两厂的同一个概念分成两个键。**

**判定：B 档，且带一个跨厂同形陷阱。**

### 十六之四条 `apusys`：在本层树里没有任何对应实体（C 档）

`apusys` 出现在联发科表的多行（CVE-2026-20467、CVE-2026-20468）。本章按 E1 逐级查证 `kernel/common@android16-6.12`：

| 查证点 | 结果 |
| --- | --- |
| `drivers/misc/mediatek/` | **目录不存在**（Gitiles 列目录返回 `No matches found`） |
| `drivers/soc/mediatek/` 全清单 | `Kconfig Makefile mt8167-mmsys.h mt8173-mmsys.h mt8183-mmsys.h mt8186-mmsys.h mt8188-mmsys.h mt8192-mmsys.h mt8195-mmsys.h mt8365-mmsys.h mtk-cmdq-helper.c mtk-devapc.c mtk-infracfg.c mtk-mmsys.c mtk-mmsys.h mtk-mutex.c mtk-pmic-wrap.c mtk-regulator-coupler.c mtk-socinfo.c mtk-svs.c` — **无 apusys** |
| `drivers/media/platform/mediatek/` 全清单 | `Kconfig Makefile jpeg mdp mdp3 vcodec vpu` — **无 apusys** |
| `drivers/remoteproc/` 全清单 | 联发科只有 `mtk_common.h mtk_scp.c mtk_scp_ipi.c` — **无 APU 协处理器驱动** |
| `mtk_scp.c` 内检索 `apusys` | 0 命中 |

结论必须写成可操作的话：**`apusys` 是 vendor tree 里的组件，不在 GKI 镜像里。** 因此第 1 条运营后果是：任何"在设备上枚举内核媒体/加速器驱动"的只读脚本都不可能发现它——不是权限不够，是对象不在这台设备的这个分区里；它的可见面（如果有）在厂商私有分区或专有库中。第 2 条后果：这类条目只能按公告词面归档，`align_class = C`，永远不参与 K 类指标的分母。

顺带钉一个反向对照：`drivers/misc/` 目录里确实存在高通的 DSP 用户态通道 `fastrpc.c`（该目录实测清单中含 `fastrpc.c`、`uacce`、`bcm-vk`、`open-dice.c`、`pkvm-smc`）。也就是说**"计算侧 DSP 有公开内核面，联发科 APU 侧没有"**这件事，在同一棵树的同一个目录层级上就能读出来。

### 十六之五 高通侧：`Video` 与 PAS 词表、以及三个不同的名字长度

`Video`（高通表，CVE-2025-27042/27061）的真身在第 15 章已经给出（`drivers/media/platform/qcom/venus/`，PAS ID 9）。本章补上高通 DSP 族的完整内部词表——它是外部词表对不齐的根源。`drivers/remoteproc/qcom_q6v5_pas.c` 实测统计：

| 度量 | 实测值 |
| --- | --- |
| `static const struct adsp_data` 条目 | 36 |
| `of_device_id` 中 `qcom,` 开头 compatible | 66 |
| compatible 后缀分布 | `-adsp-pas` 20、`-cdsp-pas` 14、`-mpss-pas` 13、`-slpi-pas` 6、`-cdsp0-pas`/`-cdsp1-pas`/`-gpdsp0-pas`/`-gpdsp1-pas`/`-nsp0-pas`/`-nsp1-pas`/`-wcss-pas`/`-wpss-pas` 各 1 |
| 唯一 `.firmware_name` 取值 | 11：`adsp.mbn` `adsp.mdt` `cdsp.mdt` `cdsp0.mbn` `cdsp1.mbn` `gpdsp0.mbn` `gpdsp1.mbn` `modem.mdt` `slpi.mdt` `wcnss.mdt` `wpss.mdt` |
| 唯一 `.sysmon_name` 取值 | 8：`adsp` `cdsp` `cdsp1` `gpdsp0` `gpdsp1` `modem` `slpi` `wcnss` |
| `.pas_id` 取值 | 1（ADSP）、4（MPSS）、6（WCNSS）、12（SLPI）、18（CDSP）、30（CDSP1）、39/40（GPDSP0/1）；媒体侧 Venus 另为 9（第 15 章） |

装载点与名字来源是逐字可查的三行：

> `	fw_name = desc->firmware_name;`
> `	ret = of_property_read_string(pdev->dev.of_node, "firmware-name",`
> `				      &fw_name);`
> `	if (ret < 0 && ret != -EINVAL)`
> `		return ret;`
> — `kernel/common@android16-6.12:drivers/remoteproc/qcom_q6v5_pas.c:702-706`

> `	rproc = devm_rproc_alloc(&pdev->dev, pdev->name, ops, fw_name, sizeof(*adsp));`
> — 同文件 `:719`

这一行是本报告"公告词 ↔ sysfs 读数"链条的枢纽：`fw_name` 作为第 4 个实参进入 `rproc->firmware`，而 `rproc->firmware` 正是 `/sys/class/remoteproc/remoteprocN/firmware` 读出来的字符串（第十七章之二）。**高通 DSP 侧的固件名，是媒体链里唯一一条"表内写死 + 设备树覆盖 + 自动落到 sysfs"的通路。**

装载之后不留版本这件事也有原句：

> `	/* Remove pointer to the loaded firmware, only valid in adsp_load() & adsp_start() */`
> — 同文件 `:335`（同句另见 `:360`）

**名字宽度对照**（同一个"版本/标识"概念在四个地方的存储宽度）：

| 概念 | 载体 | 宽度 | 出处 |
| --- | --- | --- | --- |
| PIL 重定位记录里的镜像标识 | `PIL_RELOC_NAME_LEN` | **8 字节** | `qcom_pil_info.c:19` |
| VPU 固件自报版本 | `VPU_FW_VER_LEN` / `char fw_ver[]` | **16 字节** | `mtk_vpu.c:31 / :147` |
| SCP 固件自报版本 | `SCP_FW_VER_LEN` / `s8 fw_ver[]` | **32 字节** | `drivers/remoteproc/mtk_common.h:80 / :84` |
| 公告 Subcomponent | 自由文本 | 无约束 | 第五章之四 |

PIL 那条的设计意图是逐字写明的：

> ` * The PIL relocation information region is used to communicate memory regions`
> ` * occupied by co-processor firmware for post mortem crash analysis.`
> — `drivers/remoteproc/qcom_pil_info.c:12-13`

写入时按 8 字节截断：

> `	memcpy_toio(entry, image, strnlen(image, PIL_RELOC_NAME_LEN));`
> — 同文件 `:107`；槽位不足时的原句 `pr_warn("insufficient PIL info slots\n")`（`:102`）

**判定：`Video` 是 A 档（对象唯一，且有 sysfs `firmware` 面——但注意 Venus 自身不在 remoteproc 里，见第十七章之九）；`Security`、`Closed-source component` 仍是 C 档。** 名字宽度表给出一条方法结论：跨域对齐时不能按全串相等匹配，只能取公共前缀或按语义建映射——这正是第廿七章脚本 1 归一层要显式实现的分支。

### 十六之六 `trusted_mem`、`geniezone`、`VBMeta`、`StrongBox`、`DD+ Codec`、`GDMC`/`GSA`

| 词 | 判定 | 依据 |
| --- | --- | --- |
| `trusted_mem` | **C**（功能描述，非器件） | 本文件集内无同名实体。最近的公开概念只有三个：设备树保留内存引用 `memory-region = <&scp_mem_reserved>;`（`mt8183.dtsi:1044`）、Venus 的 IOMMU 私有映射与 `qcom_scm_mem_protect_video_var`（第 15 章之一/之三）、以及联发科总线越界记录器 `drivers/soc/mediatek/mtk-devapc.c` 的三条原句 `"Write Violation"` / `"Read Violation"` / `"Bus ID:0x%x, Dom ID:0x%x, Vio Addr:0x%x"`（`:175 / :177 / :179`）。**结论：这个词只能读成"受保护内存通路"，落到台账时属于 F5（策略）而非 F1/F2。** |
| `geniezone` | **A 但跨层** | 树内真实存在：`drivers/virt/geniezone/` 9 个文件（`gzvm_common.h gzvm_exception.c gzvm_ioeventfd.c gzvm_irqfd.c gzvm_main.c gzvm_mmu.c gzvm_vcpu.c gzvm_vm.c`），`include/uapi/linux/gzvm.h` 提供用户态接口。**但它是虚拟化/隔离组件，不属于固件装载六层**，归档时不得落进 F1–F4；第 5、12 篇处理它。 |
| `VBMeta` | **C** | AVB 元数据，属引导链对象（第 2 篇），不在媒体加速器域。 |
| `StrongBox` | **C** | 密钥栈组件（第 5 篇）。 |
| `DD+ Codec` | **C** | 用户态编解码库；内核树内没有对应驱动对象。 |
| `GDMC`、`GSA` | **C** | Google 自研侧内部代号，在本次取回的公开树文件集内无对应实体。 |
| `preloader`、`DA` | **C（跨篇）** | 引导/下载模式对象，归第 2 篇（引导链）与第 9 篇。 |

这张表有一个必须说清的**不对称**：`geniezone` 与 `apusys` 都是厂商内部代号，但一个在公开树里有完整实现、一个没有。**"词面像不像公开组件名"不能用来预判可得性**，只能实测——这就是本章按 E1 逐级列目录而不是按经验判断的原因。

### 十六之七 对齐台账的字段定义（第廿七章脚本 1 的输出契约）

| 字段 | 取值域 | 约束 |
| --- | --- | --- |
| `bulletin_word` | 原样字符串 | 不得大小写折叠（第五章之四观察 2） |
| `word_norm` | 大小写不敏感比较键 | 仅作匹配用，不覆盖原样列 |
| `align_class` | A / B / C | C 必须能在设备上复现为"查不到" |
| `kernel_objects` | 路径或符号列表 | B 档允许多值 |
| `evidence` | E1/E2/E3/E4 组合 | 至少一类，且必须给出 `file:line` |
| `read_face` | V1–V8（第十九章） | C 档允许为空，但须填 `why_empty` |

### 十六之八 本章结论

本章把第五章承诺的对齐结果落成七张表，其中三条会直接影响运营判断：

1. **一半以上的媒体侧词落在 B 档。** 一个公告词覆盖多套内部词表（IPI 通道 / 能力位 / M4U 端口 / DRM 组件），不存在"词 → 器件"的一对一映射；资产映射脚本的产出必须是集合而非单值。
2. **`apusys` 类型的 C 档条目决定了指标的覆盖上限。** 对象不在 GKI 分区，设备侧枚举不可能命中；K 类指标若把它们算进分母会系统性低估覆盖率，算进分子则虚报覆盖。
3. **命名宽度差异使全串相等匹配必然失败。** 8 / 16 / 32 字节三种截断长度加自由文本，唯一稳妥做法是"按语义建映射 + 记录截断长度"，这也是第 30 章边界清单里"公告词与内核对象之间无权威对照表"这条的具体含义。

## 十七、remoteproc：媒体固件唯一自带标准读数面的地方，以及它的四个坑

第 15 章给出一个反差：联发科 SCP 后端是四条媒体装载链里唯一走通用远程处理器框架的，因此它自动获得一整面标准 sysfs。本章把这面读数逐字拆开——它给的信息比多数人以为的少，而它的坑恰好都在"看起来能读"的地方。

### 十七之一 五个属性与它们的原语

`drivers/remoteproc/remoteproc_sysfs.c` 声明的属性与注册顺序（逐字，`:245-252`）：

> `static struct attribute *rproc_attrs[] = {`
> `	&dev_attr_coredump.attr,`
> `	&dev_attr_recovery.attr,`
> `	&dev_attr_firmware.attr,`
> `	&dev_attr_state.attr,`
> `	&dev_attr_name.attr,`
> `	NULL`
> `};`

| 属性 | 宏 | 行 | 读得到什么 |
| --- | --- | --- | --- |
| `coredump` | `DEVICE_ATTR_RW` | `:135` | `disabled` / `enabled` / `inline` |
| `recovery` | `DEVICE_ATTR_RW` | `:73` | `enabled\n` 或 `disabled\n`（`sysfs_emit`，`:19`） |
| `firmware` | `DEVICE_ATTR_RW` | `:167` | 固件**文件名**，或 `unknown` |
| `state` | `DEVICE_ATTR_RW` | `:217` | 八个状态词之一 |
| `name` | `DEVICE_ATTR_RO` | `:227` | `rproc->name` |

类本身：

> `const struct class rproc_class = {`
> `	.name		= "remoteproc",`
> `	.dev_groups	= rproc_devgroups,`
> `};`
> — `:264-267`

五个属性里四个可写，这是本章第一个要注意的不对称：**这一面"可写属性多于可读字段"的读数面，在权限上是按写侧收紧、读侧全开的**（十七之五）。

### 十七之二 坑 1：`firmware` 读到的永远是文件名；`attached` 时是 `unknown`

注释与实现逐字对齐得很好，因为它直接把无知写进了注释：

> `/* Expose the loaded / running firmware name via sysfs */`
> …
> `	/*`
> `	 * If the remote processor has been started by an external`
> `	 * entity we have no idea of what image it is running.  As such`
> `	 * simply display a generic string rather then rproc->firmware.`
> `	 */`
> `	if (rproc->state == RPROC_ATTACHED)`
> `		firmware = "unknown";`
> `	return sprintf(buf, "%s\n", firmware);`
> — `remoteproc_sysfs.c:137-152`（原句 `then` 应为 `than`，逐字保留）

三条推论：

1. 这个属性**从不**回答"版本"。它回答的是"内核打算装载哪个文件"，与镜像内容无关；固件版本要靠各驱动自己打印（第十五章之七 `mtk_vpu.c:628` 那种），框架不提供。
2. `unknown` 是一个**有明确定义的状态**，不是读取失败。`RPROC_ATTACHED` 的公开定义句：

> ` * @RPROC_ATTACHED:	device has been booted by another entity and the core`
> ` *			has attached to it`
> — `include/linux/remoteproc.h:416-417`

3. 但 `unknown` 在同一个框架里有**两种不同来源**。除上面这条，`rproc_crash_to_string()` 的越界回退也返回 `"unknown"`：

> `static const char *rproc_crash_to_string(enum rproc_crash_type type)`
> `{`
> `	if (type < ARRAY_SIZE(rproc_crash_names))`
> `		return rproc_crash_names[type];`
> `	return "unknown";`
> `}`
> — `drivers/remoteproc/remoteproc_core.c:71-76`（词表 `rproc_crash_names[]` 见 `:64-68`）

**这就是台账字段的分岔点。** "读到 unknown"至少有三义：状态确为 ATTACHED、崩溃类型越界回退、以及权限不足导致的读取失败（后者根本不是 `unknown` 字符串而是 errno）。第十七章之九与第廿八章脚本 2 必须把它们分成三个不同的值，否则 K 类指标会把"读不到"记成"设备处于 attached 状态"。

### 十七之三 `firmware` 的字符串从哪来（跨三章对齐）

| 域 | 写入点 | 是否可被设备树覆盖 |
| --- | --- | --- |
| 高通 ADSP/CDSP/SLPI/WPSS | `devm_rproc_alloc(..., fw_name, ...)`（`qcom_q6v5_pas.c:719`），`fw_name` 来自 36 条表项或 DT `firmware-name`（`:702-706`） | 是 |
| 联发科 SCP | `rproc_of_parse_firmware(dev, 0, &fw_name)` + `devm_rproc_alloc(dev, np->name, &scp_ops, fw_name, ...)`（`drivers/remoteproc/mtk_scp.c:1126 / :1130`） | 是（且 `name` 属性直接用 DT 节点名 `np->name`） |
| 高通 Venus | 不在此框架内（十七之九） | — |

`recovery`/`state` 的写侧在第 13 章已登记为能力事实；本章只补充读侧后果：`rproc_set_firmware()` 在非 OFFLINE 状态下直接拒绝，原句与错误码：

> `	if (rproc->state != RPROC_OFFLINE) {`
> `		dev_err(dev, "can't change firmware while running\n");`
> `		ret = -EBUSY;`
> — `remoteproc_core.c:2210-2213`

空串同样被拒（`"can't provide empty string for firmware name\n"`，`:2217`）。**读侧脚本不需要这些分支，但它们决定了"为什么这台设备的 `firmware` 看起来改不动"——那是状态机而非权限。**

### 十七之四 坑 2：`state` 的八词表、越界钳制与"从不可达"的 ATTACHED

> `static const char * const rproc_state_string[] = {`
> `	[RPROC_OFFLINE]		= "offline",`
> `	[RPROC_SUSPENDED]	= "suspended",`
> `	[RPROC_RUNNING]		= "running",`
> `	[RPROC_CRASHED]		= "crashed",`
> `	[RPROC_DELETED]		= "deleted",`
> `	[RPROC_ATTACHED]	= "attached",`
> `	[RPROC_DETACHED]	= "detached",`
> `	[RPROC_LAST]		= "invalid",`
> `};`
> — `remoteproc_sysfs.c:173-182`（其上注释 `Always keep in sync with enum rproc_state`，`:171`）

读侧还有一层钳制：

> `	state = rproc->state > RPROC_LAST ? RPROC_LAST : rproc->state;`
> `	return sprintf(buf, "%s\n", rproc_state_string[state]);`
> — `:191-192`

也就是说 **`state` 永远不会读失败，越界时它会告诉你 `invalid`**。这条很重要：一个把 `invalid` 当成"读不到"来记账的脚本会丢掉"内核状态枚举与字符串表不同步"这一真实缺陷信号。

关键可达性问题：媒体侧的两个域里，`attached` 是不是一个可能出现的值？框架给了唯一入口链：

- 只有 `rproc->state == RPROC_DETACHED` 时 `rproc_boot()` 才走 attach：`dev_info(dev, "attaching to %s\n", rproc->name);` → `rproc_attach(rproc)`（`remoteproc_core.c:1944-1946`）；
- 而进入 `RPROC_DETACHED` 前置校验要求驱动实现 `.attach`，否则 `-EINVAL`：

> `	case RPROC_DETACHED:`
> `		/*`
> `		 * A remote processor in a detached state without an`
> `		 * attach() function makes not sense.`
> `		 */`
> `		if (!rproc->ops->attach)`
> `			return -EINVAL;`
> — `remoteproc_core.c:2248-2254`（原句 `makes not sense` 逐字保留）

联发科驱动自己把这条推论写在注释里，两处：

> `	 * Note: `scp_ops` doesn't implement .attach() callback, hence`
> `	 * `rproc->state` can never be RPROC_ATTACHED.  Otherwise, it`
> `	 * should also be checked here.`
> — `drivers/remoteproc/mtk_scp.c:1555-1557`（`scp_suspend`）与 `:1572-1574`（`scp_resume`）

高通侧同样：`adsp_ops`（`qcom_q6v5_pas.c:433-441`）与 `adsp_minidump_ops`（`:443-452`）两张表都只有 `.unprepare/.start/.stop/.da_to_va/.parse_fw/.load/.panic`（后者多一条 `.coredump`），**没有 `.attach`**。

**台账结论（这条要单独成行）：在本次取回的媒体/协处理器驱动集内，`.attach` 无人实现，因此 `firmware` 读回 `unknown` 的 ATTACHED 分支在这些域上是不可达的；若在别的域读到 `unknown`，那是真实的 attach 状态而非权限问题。** `drivers/remoteproc/` 目录在 `android16-6.12` 有 36 个 `.c` 文件（`android15-6.6` 是 35 个，见第 30 章 D-16 的目录清单实测），这 36 个里谁实现了 `.attach` 未逐一取回，已列入第 30 章边界清单——**注意口径：36 是目录内 `.c` 文件数，不是 36 个 `rproc_ops` 实例**（同目录含 `remoteproc_core.c`、`qcom_common.c` 这类非驱动文件）。

### 十七之五 坑 3：`coredump` 的文档字符串列出一个实现不接受的值

先抄文档注释（`remoteproc_sysfs.c:95-110`，节选逐字）：

> `/*`
> ` * By writing to the 'coredump' sysfs entry, we control the behavior of the`
> ` * coredump mechanism dynamically. The default value of this entry is "default".`
> ` *`
> ` * The 'coredump' sysfs entry supports these commands:`
> ` *`
> ` * disabled:	This is the default coredump mechanism. Recovery will proceed`
> ` *		without collecting any dump.`
> ` *`
> ` * default:	When the remoteproc crashes the entire coredump will be`
> ` *		copied to a separate buffer and exposed to userspace.`
> ` */`

再看实现（`:111-134`）：`sysfs_streq` 只接受 `disabled`、`enabled`、`inline` 三词，其余进入

> `		dev_err(&rproc->dev, "Invalid coredump configuration\n");`
> `		return -EINVAL;`

**注释里的 `default` 不是合法值**——接受它的那条分支不存在；而注释里 `enabled` 这个词一次也没出现在说明中。同一段里还有第二处口径不一致：`coredump_show()` 上方的注释写的是

> `/* Expose the current coredump configuration via debugfs */`
> — `:86`

但它是 sysfs 属性（`:135` 注册在 `rproc_attrs`）。读侧字符串表：

> `static const char * const rproc_coredump_str[] = {`
> `	[RPROC_COREDUMP_DISABLED]	= "disabled",`
> `	[RPROC_COREDUMP_ENABLED]	= "enabled",`
> `	[RPROC_COREDUMP_INLINE]		= "inline",`
> `};`
> — `:80-84`；写侧在崩溃态被拒：`"can't change coredump configuration\n"` → `-EBUSY`（`:117-120`）

这一节是本文档"文档化边界"主题在框架内部的自证：**同一份代码在 20 行之内出现三处口径不一致**（值集合、`default`、debugfs 字样）。第廿二章的读数纪律因此规定：内核自带注释只能作为线索，不能作为台账的期望值来源；期望值只能来自实测。

### 十七之六 坑 4：`is_visible` 的 0444 降级分支在本树里没有触发者

> `static umode_t rproc_is_visible(struct kobject *kobj, struct attribute *attr,`
> `				int n)`
> `{`
> …
> `	if (rproc->sysfs_read_only && (attr == &dev_attr_recovery.attr ||`
> `				       attr == &dev_attr_firmware.attr ||`
> `				       attr == &dev_attr_state.attr ||`
> `				       attr == &dev_attr_coredump.attr))`
> `		mode = 0444;`
> `	return mode;`
> `}`
> — `remoteproc_sysfs.c:229-243`，经 `.is_visible = rproc_is_visible`（`:256`）生效

字段只有声明与读取，没有赋值：`include/linux/remoteproc.h:540` 是文档句 `* @sysfs_read_only: flag to make remoteproc sysfs files read only`，`:581` 是 `bool sysfs_read_only;`。**在本次取回的 `drivers/remoteproc/` 文件集（core、sysfs、cdev、debugfs、mtk_scp、q6v5_pas）里，`sysfs_read_only` 的赋值点为 0。**

这条观察有两种正确写法，必须选后者：

- ✗ "remoteproc sysfs 一定是可写的"；
- ✓ "本树内没有内核赋值，因此写权限的收紧只能来自构建配置之外（厂商补丁或 SELinux）；实测时 `stat` 到 0644 或 0444 都要如实登记，不能预设"。

第廿八章脚本 2 的检查点据此定为：**逐属性 `stat` 取八进制模式并原样记录**，把"模式"与"我能否读"两列分开；`0444` 出现时标记 `degraded_by_vendor`，因为它意味着有本次未取回的补丁在设置这个字段（第 30 章边界项）。

### 十七之七 第二张面孔：debugfs 六个节点，权限全在 root

> `	rproc->dbg_dir = debugfs_create_dir(dev_name(dev), rproc_dbg);`
> `	debugfs_create_file("name", 0400, rproc->dbg_dir,`
> `			    rproc, &rproc_name_ops);`
> `	debugfs_create_file("recovery", 0600, rproc->dbg_dir,`
> `			    rproc, &rproc_recovery_ops);`
> `	debugfs_create_file("crash", 0200, rproc->dbg_dir,`
> `			    rproc, &rproc_crash_ops);`
> `	debugfs_create_file("resource_table", 0400, rproc->dbg_dir,`
> `			    rproc, &rproc_rsc_table_fops);`
> `	debugfs_create_file("carveout_memories", 0400, rproc->dbg_dir,`
> `			    rproc, &rproc_carveouts_fops);`
> `	debugfs_create_file("coredump", 0600, rproc->dbg_dir,`
> `			    rproc, &rproc_coredump_fops);`
> — `drivers/remoteproc/remoteproc_debugfs.c:405-418`；根目录名 `debugfs_create_dir(KBUILD_MODNAME, NULL)`（`:424`）

`remoteproc_debugfs.c:398-403` 开头还有 `if (!rproc_dbg) return;`——**debugfs 未初始化时整棵树静默不存在**。

三点读数纪律：

1. `resource_table` 与 `carveout_memories`（0400）给出协处理器的内存布局与共享区名称。这是比 sysfs 更实质的信息面，但 0400 + debugfs 挂载权限意味着非 root 进程读不到；第 20 章的 shell 域预期失败表把它列为"模式层拒绝"，不是 SELinux 拒绝。
2. `crash` 是 **0200 只写**。本报告只登记这一存在事实（"该框架提供一个仅供 root 触发崩溃的写接口"），**不给出任何使用方式**——它属于会改变设备状态的写侧能力，与第 13 章把 `state_store` 登记为能力事实的处理一致。
3. `name` 在 sysfs 是 RO 属性、在 debugfs 是 0400 文件，两处同源；若某设备上 `remoteprocN` 的 sysfs `name` 与 debugfs 同名目录不一致，那是厂商改动信号，值得单独入台账。

### 十七之八 第三张面孔：字符设备没有读路径

`drivers/remoteproc/remoteproc_cdev.c` 的操作表决定了这张面孔能不能"读"：

> `static const struct file_operations rproc_fops = {`
> `	.write = rproc_cdev_write,`
> `	.unlocked_ioctl = rproc_device_ioctl,`
> `	.compat_ioctl = compat_ptr_ioctl,`
> `	.release = rproc_cdev_release,`

— `:91-96`（**无 `.read`、无 `.open`**，逐字核对）

唯一的查询项是一个 ioctl：

> `#define RPROC_MAGIC	0xB7`
> `#define RPROC_SET_SHUTDOWN_ON_RELEASE _IOW(RPROC_MAGIC, 1, __s32)`
> — `include/uapi/linux/remoteproc_cdev.h:14 / :24`；GET 侧 `_IOR(RPROC_MAGIC, 2, __s32)`（`:35`），实现 `remoteproc_cdev.c:61-66` 直接把 `rproc->cdev_put_on_release` 回填用户态。

写侧三条命令与 sysfs `state_store` 一一对应，且缓冲区是固定 10 字节：

> `	char cmd[10];`
> …
> `	if (!strncmp(cmd, "start", len)) {`
> `		ret = rproc_boot(rproc);`
> `	} else if (!strncmp(cmd, "stop", len)) {`
> — 同文件 `:25`（`char cmd[10];`）与 `:34-38`

`.release` 的语义值得单列，因为它让"打开再关闭"成为一个有后果的动作：

> `	if (rproc->state == RPROC_RUNNING)`
> `		rproc_shutdown(rproc);`
> `	else if (rproc->state == RPROC_ATTACHED)`
> `		ret = rproc_detach(rproc);`
> — `:83-86`，前置条件是 `if (!rproc->cdev_put_on_release) return 0;`（`:80`）

设备号与容量：`#define NUM_RPROC_DEVICES 64`（`:18`）、`alloc_chrdev_region(&rproc_major, 0, NUM_RPROC_DEVICES, "remoteproc")`（`:123`）、`cdev_add(&rproc->cdev, rproc->dev.devt, 1)`（`:107`）。

**第 13 章的结论在这里闭环**：AOSP `private/genfs_contexts` 没有 remoteproc 条目，`file_contexts` 也没有对应标签（第 13 章实测），所以 `/dev/remoteproc*` 上的访问控制完全由 DAC 与是否有 devnode 规则决定；本报告**不声明该节点存在**，因为 `rproc_char_device_add()` 只做 `cdev_add`（`:98-112` 区间内可见），节点创建依赖调用方与 udev/devtmpfs 策略。第廿八章脚本 2 因此把这一面写成"若存在则记录 ioctl 可查询，否则记 `absent`"，而不是假定可读。

### 十七之九 崩溃面：媒体固件唯一可跨域比较的趋势信号

三个必须逐字记录的原句：

| 位置 | 原句 | 意义 |
| --- | --- | --- |
| `remoteproc_core.c:2716-2717` | `dev_err(&rproc->dev, "crash detected in %s: type %s\n", rproc->name, rproc_crash_to_string(type));` | 崩溃上报的**唯一**默认级别出口；`rproc_report_crash()` 在 `:2706` |
| `remoteproc_core.c:1887-1888` | `dev_err(dev, "handling crash #%u in %s\n", ++rproc->crash_cnt, rproc->name);` | 带**单调计数**，`rproc_crash_handler_work()` 起于 `:1865` |
| `drivers/remoteproc/mtk_scp.c:73` | `dev_err(scp->dev, "SCP watchdog timeout! 0x%x", scp_to_host);` | 超时后向 cluster 内**每个核**上报 `RPROC_WATCHDOG`（`:76-77`） |

崩溃类型只有三个词：

> `enum rproc_crash_type {`
> `	RPROC_MMUFAULT,`
> `	RPROC_WATCHDOG,`
> `	RPROC_FATAL_ERROR,`
> `};`
> — `include/linux/remoteproc.h:450-454`（其上文档句 `Feel free to add more types when needed.`，`:448`）

恢复策略的门只有一个布尔：

> `	if (!rproc->recovery_disabled)`
> `		rproc_trigger_recovery(rproc);`
> …
> `	trace_android_vh_rproc_recovery(rproc);`
> — `remoteproc_core.c:1892-1896`

`trace_android_vh_rproc_recovery()` 这一行是 GKI 留给厂商的钩子位（同族 `android_vh_*` 钩子的取证意义在第 16 篇已建立）：**恢复动作可以在厂商层被改写，因此"崩溃后是否重启协处理器"不能从上游代码推断，只能实测或取厂商补丁。**

媒体侧可得的趋势面因此只有一句：`state == crashed` 与 `crash_cnt` 对应的日志行。第 26 章的 K6（协处理器崩溃可见性）就建立在这两个词上，而不是任何"固件版本"字段。

### 十七之十 Venus 不在这一面上（重要的负结果）

在本文取回的 `drivers/media/platform/qcom/venus/` 全部文件中检索 `rproc` 与 `remoteproc`：**0 命中**。第 15 章之一给出的装载路径（`request_firmware()` → `qcom_mdt_load()` → `qcom_scm_pas_auth_and_reset()`）不经过远程处理器框架，`venus_firmware_init()` 注册的是一个普通 platform device（`firmware.c:228-232` 附近），所以：

| 域 | 在 remoteproc 面上？ | 可得的固件信息 | 崩溃信息 |
| --- | --- | --- | --- |
| 联发科 vcodec-SCP | **是** | `state`、`name`（=DT 节点名）、`firmware`（=文件名） | `crashed` + 日志 |
| 联发科 vcodec-VPU（MT8173） | 否（独立 `mtk_vpu` platform driver） | 只有 `dev_info` 一行版本 + debugfs `mtk_vpu`（`mtk_vpu.c:880-881`，`S_IRUGO`） | VPU 看门狗日志 |
| 高通 Venus | **否** | 无 rproc 面；固件自报版本只进内核结构体与 `dev_dbg`（第十五章之四） | 无 |
| 高通 ADSP/CDSP/SLPI/WPSS | **是** | 全五个属性 + DT 覆盖名 | `crashed` + minidump（`ops = &adsp_minidump_ops`，`qcom_q6v5_pas.c:716-717`） |

**这张表是本报告"统一读数面"命题的反例来源**：框架统一了四个属性，却只覆盖四条媒体链里的两条。**任何"遍历 /sys/class/remoteproc 就得到全部协处理器固件台账"的脚本，在高通媒体与联发科 VPU 后端上是漏的。** 第廿八章脚本 2 的第二段专门用来暴露这个漏：它同时枚举 `remoteproc`、`/sys/kernel/debug/remoteproc`、以及媒体驱动的 platform device 三类，并把只出现在后两类的实例标记为 `no_rproc_face`。

### 十七之十一 本章结论

1. **`firmware` 属性回答文件名，不回答版本；`unknown` 至少有三义**（ATTACHED 状态、崩溃类型越界回退 `remoteproc_core.c:75`、以及被误记为空的权限失败）。台账必须分列。
2. **媒体侧的 `.attach` 在本次取回的所有驱动里都未实现**，所以这些域的 `unknown` 不可达；读到 `unknown` 的域一定不是媒体协处理器，这个推理可以直接用来校验脚本的采集结果。
3. **`state` 永不失败但会说 `invalid`**（钳制 `:191`），`invalid` 是缺陷信号而非缺失数据。
4. **`coredump` 的自带注释与实现不一致**（`default` 非法、`enabled` 未写、`via debugfs` 字样错误），因此内核注释不能充当期望值。
5. **0444 降级分支在本树内没有赋值者**：写权限收紧若存在，来自本次未取回的补丁或 SELinux——记为厂商差异信号，不记为常态。
6. **这一面只覆盖两条媒体链。** 脚本必须用三类枚举交叉，才能把"没有 rproc 面"与"固件不可读"区分开。
## 十八、跨处理器通道的四层命名：通道有名字，名字没有出口

第十六章把公告词分成 A（对得上）/ B（半个）/ C（对不上）三档，并在 B 档那一栏留下一句话："三套词（IPI 通道名 / 能力位 / M4U 端口名）指向同一个功能域但没有一个公共主键。"本章给出这句话的机理：**一条媒体/加速器消息在从主机走到固件、再走回来的路上，会被换五次名字，而其中三次换名之后，前一次的名字就再也不出现在任何可枚举的地方。** 台账里那些"通道存在但说不清是什么通道"的记录，全部来自这条链。

出处约定：`kernel/common@android16-6.12` 的 `drivers/rpmsg/rpmsg_core.c`、`rpmsg_char.c`、`rpmsg_ctrl.c`、`drivers/rpmsg/mtk_rpmsg.c`、`drivers/remoteproc/mtk_scp.c`、`drivers/media/platform/mediatek/vpu/mtk_vpu.c`、`include/linux/rpmsg.h`、`include/linux/mod_devicetable.h`、`include/uapi/linux/rpmsg.h`、`include/linux/remoteproc/mtk_scp.h`、`drivers/media/platform/mediatek/vpu/mtk_vpu.h`、`drivers/media/platform/mediatek/vcodec/{common,decoder}/*`、`drivers/rpmsg/qcom_glink_native.c`、`drivers/media/platform/qcom/venus/`；AOSP `system/sepolicy@main` 的 `public/device.te`、`private/device.te`、`private/app_neverallows.te`、`private/dumpstate.te`。

### 十八之一 五层命名总表

| 层 | 名字形态 | 宽度上限 | 谁定义 | 运行时可读吗 | 出口 |
| --- | --- | --- | --- | --- | --- |
| **L0 消息号** | `AP_IPIMSG_DEC_START` / `VPU_IPIMSG_DEC_INIT_ACK` 之类的枚举常量；HFI 侧是 `0x10006`、`0x6` 一类数字 | 无字符串形态 | 内核与固件共享的头文件 | **否** | 无（只有编译期符号） |
| **L1 IPI 通道号** | `enum scp_ipi_id`（17 条 + `SCP_IPI_NS_SERVICE = 0xFF`）、`enum ipi_id`（7 条） | 无字符串形态 | 内核头文件 | **否** | 无（日志只打印数字） |
| **L2 rpmsg 服务名** | `char name[RPMSG_NAME_SIZE]`，`#define RPMSG_NAME_SIZE 32`（`mod_devicetable.h:465`） | 32 字节，含结尾 NUL 实为 31 | **远端固件填写**（名服务报文）或用户空间填写（ctrl ioctl） | 是 | `/sys/bus/rpmsg/devices/*` 的 `name`、`modalias` |
| **L3 设备模型名** | `dev_set_name(dev, "%s.%s.%d.%d", dev_name(dev->parent), rpdev->id.name, rpdev->src, rpdev->dst)`（`rpmsg_core.c:620-621`）；字符设备侧是 `rpmsg%d`（`rpmsg_char.c:459`）与 `rpmsg_ctrl%d`（`rpmsg_ctrl.c:168`） | 组合名无长度限制，但其中服务名段仍是 31 | 内核 device core | 是（目录名本身） | 目录列举、符号链接 |
| **L4 公告词** | `vdec`、`Video`、`display`、`apusys` | 自然语言 | 厂商文档团队 | 是（网页） | 无机械映射 |

表的读法只有一句：**L2 是唯一"既是字符串又可枚举"的层，而它的填写者是固件本身；L0/L1 完全没有字符串出口；L4 与 L2 之间没有任何内核侧映射。** 所以任何"从公告词直接查到设备上的某个节点"的期望，在这张表上注定失败——不是权限问题，是**没有那一层**。

### 十八之二 L2 的名字来自远端：内核在源码里写明"不要相信协处理器"

联发科 SCP 侧的名字服务链是完整可钉的。SCP 驱动把一条 IPI 通道指定为名服务通道：

> `static struct mtk_rpmsg_info mtk_scp_rpmsg_info = {`
> `	.send_ipi = scp_send_ipi,`
> `	.register_ipi = scp_register_ipi,`
> `	.unregister_ipi = scp_unregister_ipi,`
> `	.ns_ipi_id = SCP_IPI_NS_SERVICE,`
> `};`
> — `drivers/remoteproc/mtk_scp.c:1088-1093`

`SCP_IPI_NS_SERVICE` 的取值是 `0xFF`，写在通道枚举表末尾（`include/linux/remoteproc/mtk_scp.h:47`，同表 `:29-49`）。rpmsg 侧在 `prepare` 时给这条通道建一个专用端点：

> `	/* a dedicated endpoint handles the name service msgs */`
> `	if (mtk_subdev->info->ns_ipi_id >= 0) {`
> `		mtk_subdev->ns_ept =`
> `			__mtk_create_ept(mtk_subdev, NULL, mtk_rpmsg_ns_cb,`
> `				     mtk_subdev,`
> `				     mtk_subdev->info->ns_ipi_id);`
> — `drivers/rpmsg/mtk_rpmsg.c:313-318`

回调接收的就是固件发来的报文，其结构只有两个字段：

> `struct rpmsg_ns_msg {`
> `	char name[RPMSG_NAME_SIZE];`
> `	u32 addr;`
> `} __packed;`
> — `mtk_rpmsg.c:45-48`（结构体注释 ` * @name: name of remote service that is published` 在 `:37`）

处理这条报文的函数 `mtk_rpmsg_ns_cb()`（`:270` 起）做三道检查，每一道都是对"对面是不可信实体"的承认：

> `	if (len != sizeof(*msg)) {`
> `		dev_err(dev, "malformed ns msg (%d)\n", len);`
> `		return -EINVAL;`
> `	}`
> — `mtk_rpmsg.c:279-283`

> `	/*`
> `	 * the name service ept does _not_ belong to a real rpmsg channel,`
> `	 * and is handled by the rpmsg bus itself.`
> `	 * for sanity reasons, make sure a valid rpdev has _not_ sneaked`
> `	 * in somehow.`
> `	 */`
> `	if (rpdev) {`
> `		dev_err(dev, "anomaly: ns ept has an rpdev handle\n");`
> `		return -EINVAL;`
> `	}`
> — `mtk_rpmsg.c:284-292`

第三道直接是关于字符串的：

> `	/* don't trust the remote processor for null terminating the name */`
> `	msg->name[RPMSG_NAME_SIZE - 1] = '\0';`
> — `mtk_rpmsg.c:295-296`

然后这个名字才被落成设备：

> `	dev_info(dev, "creating channel %s addr 0x%x\n", msg->name, msg->addr);`
> `	ret = mtk_rpmsg_create_device(mtk_subdev, msg->name, msg->addr);`
> — `mtk_rpmsg.c:298-300`

`mtk_rpmsg_create_device()`（`:250-251` 签名）把名字塞进一个 `rpmsg_channel_info`：`strscpy(info->info.name, name, RPMSG_NAME_SIZE);`（`:259`），再经工作队列落到 `strscpy(rpdev->id.name, info->name, RPMSG_NAME_SIZE);`（`:214`），最后由 `rpmsg_register_device(rpdev)`（`:222`）交给 device core 命名。

四条台账结论：

1. **`/sys/bus/rpmsg/devices/<父名>.<服务名>.<src>.<dst>` 里那段服务名，是固件选择让你看到的字符串。** 它证明"固件宣布了一条服务"，不证明"这条服务对应的镜像是哪一个版本"。第廿六章的判定模型据此把 rpmsg 名字读数一律标成 `fw_reported`，不得进入版本字段。
2. **字符串宽度是 32 字节的硬上限**（`mod_devicetable.h:465`，`include/linux/rpmsg.h:34-35` 的 `struct rpmsg_channel_info { char name[RPMSG_NAME_SIZE]; }`）。厂商如果把器件全名写成 40 字符的服务名，落到目录里就是被截断的 31 字符——**归一化必须按前缀而不是全等**。
3. **这条链的唯一文本出口是 `dev_info`（`:298`）**，也就是第十九章 V6 面那条"存在但标准姿势读不到"的通道。
4. **`SCP_IPI_NS_SERVICE` 是 L1 与 L2 之间唯一的机械连接**：一条数字通道（0xFF）承载一条字符串。除此之外，L1 的 17 条通道没有任何一条有字符串出口——这正是下一节。

### 十八之三 名字被丢弃的两条路：SCP 后端不存，VPU 后端存了不读

同一个"注册 IPI 处理程序并给它一个名字"的动作，在两个后端里走向两种结局。抽象层给两个后端传的都是五个参数（含 `const char *name`）：

> `static int mtk_vcodec_scp_set_ipi_register(struct mtk_vcodec_fw *fw, int id,`
> `					   mtk_vcodec_ipi_handler handler,`
> `					   const char *name, void *priv)`
> `{`
> `	return scp_ipi_register(fw->scp, id, handler, priv);`
> `}`
> — `drivers/media/platform/mediatek/vcodec/decoder/mtk_vcodec_fw_scp.c:28-33`

`name` 参数在函数体里**没有出现在调用中**。这不是笔误，是被调方的签名根本不接受它：

> `int scp_ipi_register(struct mtk_scp *scp, u32 id, scp_ipi_handler_t handler,`
> `		     void *priv);`
> — `include/linux/remoteproc/mtk_scp.h:57-58`

所以 SCP 后端：**注册时给的名字被就地丢弃**，内核里不存在这条通道的任何字符串表示。

VPU 后端不同，它把名字接了下去：

> `static int mtk_vcodec_vpu_set_ipi_register(struct mtk_vcodec_fw *fw, int id,`
> `					   mtk_vcodec_ipi_handler handler,`
> `					   const char *name, void *priv)`
> `{`
> `	return vpu_ipi_register(fw->pdev, id, handler, name, priv);`
> `}`
> — `mtk_vcodec_fw_vpu.c:28-33`

而 `vpu_ipi_register()`（`drivers/media/platform/mediatek/vpu/mtk_vpu.c:287-289`，原型见 `drivers/media/platform/mediatek/vpu/mtk_vpu.h:94-95`，五参数含 `const char *name`）把它存进按通道号索引的数组：

> `	if (id < IPI_MAX && handler) {`
> `		ipi_desc = vpu->ipi_desc;`
> `		ipi_desc[id].name = name;`
> — `mtk_vpu.c:299-301`

存进去之后呢？在本次取回的 `mtk_vpu.c`（1055 行）里检索 `.name` 的全部使用点，除这条赋值外只剩两处，都与 IPI 无关：`platform_device` 的名字（`:917`）与驱动结构体里的 `.name = "mtk_vpu"`（`:1046`）。**也就是说 `ipi_desc[id].name` 被写入后再没有被读过一次。**

注册时传进去的字面量本身是可查的：`vdec_vpu_if.c` 用 `"vdec"` 注册两条通道（`drivers/media/platform/mediatek/vcodec/decoder/vdec_vpu_if.c:212-213` 与 `:220-222`），`venc_vpu_if.c` 用 `"venc"`（`drivers/media/platform/mediatek/vcodec/encoder/venc_vpu_if.c:143-144`）。这两个字符串就是第十六章里公告词 `vdec` 在驱动侧最接近的对应物——**但它们的命运是"进了数组下标，没有任何出口"**。

反证在分发失败的日志里最清楚：两条后端在拿不到处理程序时打印的都是**数字**，不是名字。

> `		dev_err(vpu->dev, "No such ipi id = %d\n", id);`
> — `mtk_vpu.c:759`

> `		dev_err(scp->dev, "No handler for ipi id = %d\n", id);`
> — `drivers/remoteproc/mtk_scp.c:115`

**这是本章最重要的一条否定结论：内核里存在通道名（VPU 后端存在数组里、SCP 后端在注册时直接丢弃），但内核从不打印它、从不导出它。** 于是"我在设备上能不能看到 `vdec` 这个通道名"这个问题的答案与"这台设备是否有这条通道"完全无关——它取决于你是否碰巧命中了一条把 id 打印出来的错误路径。第廿九章把"日志里没看到 vdec 就说明没有 vdec 通道"列为误区。

至于更低的 L0 层（消息号），它连"名字"这个形态都没有，只有枚举常量与一次边界检查。VPU 回程报文在交给解码实例前会被卡在一个闭区间里：

> `	if (!vpu_dec_check_ap_inst(dec_dev, vpu) || msg->msg_id < VPU_IPIMSG_DEC_INIT_ACK ||`
> `	    msg->msg_id > VPU_IPIMSG_DEC_GET_PARAM_ACK) {`
> `		mtk_v4l2_vdec_err(vpu->ctx, "vdec msg id not correctly => 0x%x", msg->msg_id);`
> `		vpu->failure = -EINVAL;`
> — `vdec_vpu_if.c:116-121`

打印用的仍然是 `%d`/`%x` 形态的数字。同一文件里，主机发出去的消息也是数字（`msg.msg_id = AP_IPIMSG_DEC_START;`，`:259`），分发靠 `switch (msg->msg_id)`（`:127`）与 `case VPU_IPIMSG_DEC_INIT_ACK:`（`:128`）。**L0 从来没有字符串形态，这是它"不可枚举"的原因，不是"未被实现"的证据。**

顺带留一条同源判定：通道枚举表的注释块里存在三处拼写错误原样出现在源码中——`* @SCP_IPI_INIT:	 The interrupt from scp is to notfiy kernel`（`mtk_scp.h:19`）、`* @IPI_VPU_INIT:	 The interrupt from vpu is to notfiy kernel`（`mtk_vpu.h:27`）、`* @IPI_VDEC_H264:	 The interrupt from vpu is to notify kernel to` / `handle H264 vidoe decoder job`（`mtk_vpu.h:35-36`）。第 16 篇的"注释不是规格"结论在这里有一个无害但直观的注脚：**这些字符串是厂商代码搬运的痕迹，不是可对齐的标识符；把注释里的词当作对齐键会制造假的 A 档匹配。**

### 十八之四 rpmsg 总线面：五个可读属性、一个可写属性、以及两条命名规则

`/sys/bus/rpmsg/devices/` 下每个 rpmsg 设备由总线提供四个只读属性和一个读写属性：

> `/* for more info, see Documentation/ABI/testing/sysfs-bus-rpmsg */`
> `rpmsg_show_attr(name, id.name, "%s\n");`
> `rpmsg_show_attr(src, src, "0x%x\n");`
> `rpmsg_show_attr(dst, dst, "0x%x\n");`
> `rpmsg_show_attr(announce, announce ? "true" : "false", "%s\n");`
> — `drivers/rpmsg/rpmsg_core.c:418-422`

`modalias` 的读取顺序值得单独记，因为它决定了脚本能不能把 `modalias` 当作服务名用：

> `	len = of_device_modalias(dev, buf, PAGE_SIZE);`
> `	if (len != -ENODEV)`
> `		return len;`
> `	return sprintf(buf, RPMSG_DEVICE_MODALIAS_FMT "\n", rpdev->id.name);`
> — `rpmsg_core.c:457-461`

`#define RPMSG_DEVICE_MODALIAS_FMT "rpmsg:%s"`（`mod_devicetable.h:466`）。也就是说：**只要设备有设备树节点，`modalias` 返回的是 OF modalias，服务名根本不出现；只有没有 OF 节点时才回落到 `rpmsg:<服务名>`。** 属性集合与顺序见 `rpmsg_core.c:465-474`（`ATTRIBUTE_GROUPS(rpmsg_dev)` 在 `:474`），事件上报用同一格式（`:513`）。总线类别注册在 `const struct class rpmsg_class = { .name = "rpmsg",`（`:23-24`）并由 `class_register(&rpmsg_class)`（`:708`）落地。

匹配是**整串相等**，不做前缀：

> `	return strncmp(id->name, rpdev->id.name, RPMSG_NAME_SIZE) == 0;`
> — `rpmsg_core.c:480`

这条是"截断即失配"的来源：一个 31 字符被截断的服务名，在驱动 `id_table` 里写全称的话就匹配不上，设备会以"无驱动绑定"的形态出现在目录里。**台账看到"有 rpmsg 设备但没有对应驱动"时，第一个要排除的就是宽度截断，第二个才是策略拦截。**

第五个属性是本章唯一可写的那个：

> `static ssize_t driver_override_store(struct device *dev,`
> `				     struct device_attribute *attr,`
> `				     const char *buf, size_t count)`
> — `rpmsg_core.c:425-427`，对应 `static DEVICE_ATTR_RW(driver_override);`（`:449`）

它的影响不止"换个驱动"。在带 override 的注册路径里，`driver_override` 的字符串会被**直接写进 `rpdev->id.name`**，然后参与目录命名：

> `/*`
> ` * A helper for registering rpmsg device with driver override and name.`
> ` * Drivers should not be using it, but instead rpmsg_register_device().`
> ` */`
> `int rpmsg_register_device_override(struct rpmsg_device *rpdev,`
> `				   const char *driver_override)`
> `{`
> `	...`
> `	if (driver_override)`
> `		strscpy_pad(rpdev->id.name, driver_override, RPMSG_NAME_SIZE);`
> — `rpmsg_core.c:607-618`（`dev_set_name` 在 `:620-621`；`EXPORT_SYMBOL` 在 `:647`；常规入口 `rpmsg_register_device()` 传 `NULL`，`:651`）

因此本面的可信度有一个不对称：**`name` 属性读起来很"硬件"，但它既可以来自固件宣布（十八之二），也可以来自内核驱动填写，还可以来自带权限的用户空间写入。** 记法只有一条：rpmsg 名字读数必须与"谁写入"一起记录，孤立的字符串不构成证据。第廿八章脚本 2 把 `driver_override` 的当前值一并采集，就是为了留下这个判别位。

### 十八之五 字符设备面：三个名字都不含服务名，而 AOSP 只标了两个历史节点名

`rpmsg_char.c` 为每条通道造一个字符设备，它自己的 sysfs 属性也是三个字符串/数字：

> `	return sprintf(buf, "%s\n", eptdev->chinfo.name);`
> — `rpmsg_char.c:374`（`DEVICE_ATTR_RO(name)` 在 `:376`，`src` 在 `:385`，`dst` 在 `:394`，属性表 `rpmsg_eptdev_attrs[]` 在 `:396-401`）

但**设备名本身不含服务名**：

> `	dev_set_name(dev, "rpmsg%d", ret);`
> — `rpmsg_char.c:459`

同一个 `dev->class = &rpmsg_class;`（`:432`）让这些节点出现在 `/sys/class/rpmsg/` 下，而 `/dev` 里的名字只是 `rpmsg<编号>`；编号来自 `ida_alloc_max(&rpmsg_minor_ida, RPMSG_DEV_MAX - 1, ...)`（`:450-453`），上限 `#define RPMSG_DEV_MAX (MINORMASK + 1)`（`:32`），字符设备区域以 `"rpmsg_char"` 名义注册（`:553`）。服务名进入这个设备的唯一路径是一次整体拷贝：

> `	memcpy(chinfo.name, rpdev->id.name, RPMSG_NAME_SIZE);`
> — `rpmsg_char.c:501`（在 `rpmsg_chrdev_probe()` 内，`:495` 起）

绑定的驱动 id 表只有两条：`{ .name = "rpmsg-raw" }` 与 `{ .name = "rpmsg_chrdev" }`（`:534-537`，`MODULE_DEVICE_TABLE` 在 `:539`，驱动名 `rpmsg_chrdev` 在 `:546`）。**这是一条通用管道：任何服务名都会被它接走并变成一个可打开的字符设备。** 从安全分析角度，它给出的是可达面（哪个域能打开），不是身份面。

控制侧在 `rpmsg_ctrl.c`（注意本树的文件名是 `rpmsg_ctrl.c`，不是上游较老版本的 `rpmsg_ctrldev.c`——按老名字写死路径的脚本会在这个版本上扑空）。它造的设备名同样不含服务信息：`dev_set_name(&ctrldev->dev, "rpmsg_ctrl%d", ret);`（`:168`），类别同上（`:153`），驱动名 `.name = "rpmsg_ctrl"`（`:212`），区域名 `"rpmsg_ctrl"`（`:220`）。它的 `ioctl` 入口是 `static long rpmsg_ctrldev_ioctl()`（`:74`，注册为 `.unlocked_ioctl`，`:125`），处理三个动作（`RPMSG_CREATE_EPT_IOCTL` 在 `:93-95`，`RPMSG_CREATE_DEV_IOCTL` 在 `:98-103`，`RPMSG_RELEASE_DEV_IOCTL` 在 `:107-111`），且**先无条件从用户空间取一个结构体**：

> `	if (copy_from_user(&eptinfo, argp, sizeof(eptinfo)))`
> `		return -EFAULT;`
> `	memcpy(chinfo.name, eptinfo.name, RPMSG_NAME_SIZE);`
> `	chinfo.name[RPMSG_NAME_SIZE - 1] = '\0';`
> — `rpmsg_ctrl.c:84-88`

对应的 uapi 结构是固定 32 字节名加两个地址：

> `struct rpmsg_endpoint_info {`
> `	char name[32];`
> `	__u32 src;`
> `	__u32 dst;`
> `};`
> — `include/uapi/linux/rpmsg.h:20-23`（`#define RPMSG_ADDR_ANY 0xFFFFFFFF` 在 `:12`；四个命令号 `_IOW(0xb5, 0x1, ...)` 在 `:29`、`_IO(0xb5, 0x2)` 在 `:34`、`_IOW(0xb5, 0x3, ...)` 在 `:39`、`_IOW(0xb5, 0x4, ...)` 在 `:44`，两个流控 `_IOR(0xb5, 0x5/0x6, int)` 在 `:49`/`:54`）

本文到这里为止只登记"这条通道的存在与其字段宽度"，不给出任何打开与调用姿势；需要判断可达性时读第二十章的预期失败表。

把这一节收成三条对本域有用的结论：

1. **`/dev/rpmsg0`、`/dev/rpmsg_ctrl0` 这类名字不能用于器件归因**——它们只编码了一个分配序号。脚本 1 采到这些节点时写入 `channel_present`，不写入任何器件字段。
2. **服务名可以由用户空间提供并落进内核设备模型**（`:84-88` + `rpmsg_core.c:618`），所以"目录里有个看起来像固件服务的名字"这句话的证据强度低于它在 remoteproc `firmware` 属性上的对应说法。第十八章给出的这条判定是第廿一章"读数可被伪造"缺陷类型的实例来源。
3. **AOSP 的 `file_contexts` 只标了两个 rpmsg 相关的历史节点名**：

> `/dev/rpmsg-omx[0-9]	u:object_r:rpmsg_device:s0`
> `/dev/rproc_user	u:object_r:rpmsg_device:s0`
> — `system/sepolicy@main:private/file_contexts:153-154`

类型本身有定义（`# All devices have a rpmsg device for / # achieving remoteproc and rpmsg modules / type rpmsg_device, dev_type;`，`public/device.te:75-77`），并且被明确挡在应用之外：

> `# Don't allow apps access to any of the following character devices.`
> `neverallow appdomain {`
> `    audio_device`
> `    camera_device`
> `    dm_device`
> `    radio_device`
> `    rpmsg_device`
> `}:chr_file { read write };`
> — `private/app.te:748-756`（`rpmsg_device` 在 `:754`）

**但 GKI 生成的 `rpmsg%d` / `rpmsg_ctrl%d` 两个名字不在那张表里。** 按 `file_contexts:97` 的 `/dev(/.*)? u:object_r:device:s0`，它们落到通用 `device` 类型；而 shell 在这一类上只有一条目录属性权限：`allow shell device:dir getattr;`（`private/shell.te:424`），本次取回的 shell 策略里**没有一条授权落在裸 `device` 类型的 `chr_file` 上**（全文 `device:chr_file` 共六处，五处是专项：`uhid_device`（`:4`）、`tty_device`（`:339`）、`console_device`（`:340`）、`input_device`（`:343`）、`ion_device`（`:463`），第六处 `:546` 是 neverallow），`sysfs:file` 授权也是 **0 条**。结论是**"rpmsg 字符设备能不能打开"完全由厂商策略决定，AOSP 没有答案**——这是第卅章未文档化边界清单的一条，也是第二十章预期失败表里"取决于厂商"那一行的来源。

### 十八之六 GLINK：把整条边的 label 当成 `rpmsg_name`，把每条通道挤进一个复用端点

高通侧的命名习惯与联发科相反：**名字来自设备树与内核，不来自固件。** `qcom_glink_native.c` 提供一个属性，其值是一条边（edge）的标识：

> **本节有一段先前登记为缺口、本轮已补上的取证**：早先一轮按 `drivers/soc/qcom/` 目录找 `qcom_glink_native.c` 得到 404，当时把整节标成"来源待重取"。本轮改用 Gitiles 目录清单核对后确认文件在 **`drivers/rpmsg/`**（`kernel/common@android16-6.12`，1999 行），已原样取回；下面两处行号锚因此重新进入可复核清单，取回过程逐字登记在第卅章 **D-13**。同一件事顺带留下一条方法结论：**"取不到"经常是自己的路径假设错了，不是对象不存在——而这两种情况在没有目录清单时无法区分。** `drivers/soc/qcom/` 里确实只有 `pmic_glink.c` 与 `pmic_glink_altmode.c`，这一点维持原判。

> `static ssize_t rpmsg_name_show(struct device *dev,`
> `			       struct device_attribute *attr, char *buf)`
> `{`
> `	int ret = 0;`
> `	const char *name;`
> `	ret = of_property_read_string(dev->of_node, "label", &name);`
> `	if (ret < 0)`
> `		name = dev->of_node->name;`
> `	return sysfs_emit(buf, "%s\n", name);`
> `}`
> `static DEVICE_ATTR_RO(rpmsg_name);`
> — `drivers/rpmsg/qcom_glink_native.c:1846-1858`（属性表 `qcom_glink_attrs[]` 在 `:1860-1863`）

三个读法要点：

1. 这个属性挂在这条边的 rpmsg 设备上，读出来的是**边的 label**（例如描述 ADSP/SNS/service 的那条边），不是每条通道的服务名；label 缺省回落到 DT 节点名。
2. 命名权在 DT/内核侧，因此它比联发科的 `fw_reported` 名字更可信，但仍只到"边"这一层。
3. 用户空间管道是**单条复用通道**：

> `	channel = qcom_glink_alloc_channel(glink, "rpmsg_chrdev");`
> — `qcom_glink_native.c:1886`（函数 `qcom_glink_create_chrdev()` 从 `:1877` 起，结尾 `return rpmsg_ctrldev_register_device(rpdev);` 在 `:1898`）

也就是说 glink 只给 rpmsg 字符设备框架注册**一个**名为 `rpmsg_chrdev` 的通道，所有服务共用它，具体服务名要由打开者提供（见十八之五的 `copy_from_user`）。**结果是：在高通设备上，`/sys/class/rpmsg/` 里的设备数量与"有多少条协处理器服务"没有关系。** 第廿八章脚本 2 把 glink 边的数量与 `rpmsg_ctrl` 节点数量分开计数，就是因为这两个数在两个后端上代表的东西不同。

| 对照项 | 联发科 SCP rpmsg | 高通 glink |
| --- | --- | --- |
| 服务名谁填 | 固件（名服务报文，`mtk_rpmsg.c:295-298`） | 内核/DT（`qcom_glink_native.c:1846-1855`） |
| 名服务通道 | 一条专门 IPI（`SCP_IPI_NS_SERVICE`，`0xFF`） | 无对应物（通道由 DT/固件按 cid 建立） |
| 用户态管道 | 每条通道一个 ept 设备（`rpmsg_char.c:443-459`） | 单一复用 `rpmsg_chrdev` 通道（`:1886`） |
| 名字宽度约束 | 32 字节，整串相等匹配（`rpmsg_core.c:480`） | 同 |
| 设备侧可枚举的"通道名" | 有，但值来自固件 | 有，但值是边 label，不含通道 |

### 十八之七 Venus：三层全空，因为它根本不走 rpmsg

对 `drivers/media/platform/qcom/venus/` 取回的全部文件检索 `rpmsg` 与 `rproc`：**0 命中**（本会话实测计数）。Venus 用的是自己那一层报文协议，名字形态是数字：

> `#define HFI_CMD_SYS_GET_PROPERTY		0x10006`
> — `venus/hfi_cmds.h:17`

> `#define HFI_PROPERTY_SYS_IMAGE_VERSION				0x6`
> — `venus/hfi_helper.h:428`

报文结构里没有任何字符串字段：

> `struct hfi_sys_get_property_pkt {`
> `	struct hfi_pkt_hdr hdr;`
> `	u32 num_properties;`
> `	u32 data;`
> `};`
> — `venus/hfi_cmds.h:74-78`

于是本章的五层里，**Venus 在 L1/L2/L3 三层都是空的**：它不是 remoteproc 设备（第十七章之十），没有 rpmsg 服务名，也不出现在 `/sys/class/rpmsg` 里。它唯一有字符串的地方是第十五章之四那条"固件自报版本"的返回值，而那条字符串既不是通道名也不是设备名，落地形态是 `core->venus_ver` 三个整数加一条 `dev_dbg`。

这里要防一个具体误读。第十五章之四同时确认了两件事：`.hfi_version` 是**主机侧编译期挑选的接口代际**（`venus/core.c:316` `core->res = of_device_get_match_data(dev);` → `:322` `core->pm_ops = venus_pm_get(core->res->hfi_version);`），而 `HFI_PROPERTY_SYS_IMAGE_VERSION` 是**固件自报的镜像版本**。前者能枚举（读 DT/compatible）、后者不能枚举（无出口）。两者都被叫做"版本"，但一个属于"这台机器是哪一代"，另一个才属于"这段镜像是哪一版"。

### 十八之八 回到公告词：为什么 B 档不可能就地升成 A 档

第十六章的 B 档词现在可以给出机理性解释：公告词、驱动对象、设备节点三者分别住在 L4、L0/L1、L2/L3，**相邻层之间有的只是"某一次赋值"，没有可枚举的映射表**。

| 公告词 | L0/L1 里的形态 | L2/L3 里的形态 | 缺的主键 |
| --- | --- | --- | --- |
| `vdec` | `SCP_IPI_VDEC_H264/VP8/VP9`、`SCP_IPI_VDEC_LAT/CORE`（`mtk_scp.h:31-33, 44-45`）；`IPI_VDEC_*`（`mtk_vpu.h:57-59`）；`AP_IPIMSG_DEC_*` | SCP 后端：注册名 `"vdec"` 被丢弃；VPU 后端：`"vdec"` 存数组不读；rpmsg 侧若固件宣布则可见 | 通道号与字符串之间无表；丢弃点无日志 |
| `display` | `SCP_IPI_MDP_*`（`mtk_scp.h:36-38`）、`SCP_IPI_IMGSYS_CMD`（`:46`） | 只有 `mdc`/`mdp` 平台设备名可见 | 同一词覆盖"跑固件"和"不跑固件"两条通路（第十六章之三） |
| `Video` | HFI 数字命令号 | 无（rpmsg 0 命中） | 这一档在设备模型里不存在 |
| `apusys` | 无（GKI 无该驱动） | 无 | C 档：整层缺席（第十六章之四） |

三条归一化规则因此写进台账契约（第十六章之七的 `word_norm` 字段按此填）：

1. **候选键只有三个**：宽度 ≤32 的字符串、目录/节点名、compatible 后缀；不做语义推断，不用注释与文档标题当键。
2. **来源标签必填**：`fw_reported`（经名服务或服务自报）/ `dt_fixed`（设备树与驱动写死）/ `driver_bound`（绑定后被覆盖）。三者证据强度依次上升，不可互换。
3. **序号名不用于器件归因**：`rpmsg%d`、`rpmsg_ctrl%d`、`/dev/remoteproc*` 的编号只证明"有这个实例"，脚本必须把它们与名字类读数分列。

### 十八之九 本章结论

1. **通道有名字，名字没有出口**：L0/L1 两层在内核里没有字符串出口，日志只出数字（`mtk_vpu.c:759`、`mtk_scp.c:115`、`vdec_vpu_if.c:118`）。
2. **唯一可枚举的字符串层由远端填写**，内核用一行注释承认这一点：`/* don't trust the remote processor for null terminating the name */`（`mtk_rpmsg.c:295`）。它的存在同时给出可读面和不可信标记。
3. **32 字节是硬约束，匹配是整串相等**（`mod_devicetable.h:465` + `rpmsg_core.c:480`）：截断即失配，归一化必须按前缀并保留"被截断"这一状态。
4. **字符设备侧三个名字都不含服务信息**（`rpmsg%d` / `rpmsg_ctrl%d` / 边 label），且 AOSP 只为两个历史节点名给了标签（`file_contexts:153-154`）；`rpmsg_device` 被应用侧 neverallow（`app.te:748-756`），shell 对通用 `device` 只有 `dir getattr`（`shell.te:424`）。
5. **两条后端在"名字去向"上相反**：联发科由固件宣布、内核被动落地；高通 glink 由 DT 提供边 label、把服务挤进一条复用通道。**同一台设备上的"通道数"在两个平台上不是同一个量。**
6. **Venus 三层全空**（rpmsg/rproc 0 命中），它的"版本"要区分主机侧代际与固件自报两条，混用即错记。
7. B 档公告词的成因不是权限，而是**层与层之间没有映射表**；所以 B 档只能靠运营手段（要求供应商在主键栏给出映射）而不是靠更强的采集来消除。第廿六章 K9（公告词到内核对象的映射覆盖率）就是给这件事定指标的。
## 十九、设备侧八类读数面：能列出的、能读到的，以及被策略静音的

第一章第四条承诺"一张设备侧可见性边界表：八类读数面（V1–V8）与 shell 域预期失败表，逐项钉到 sepolicy 的规则行上"。本章做前半件事：定义什么叫一个读数面，把八面各自钉到 AOSP 的规则行上，并说明**为什么在本域"能不能读"几乎与器件无关，而由三条平台规则的组合决定**——coredomain 划分、genfs 未枚举、`/vendor/etc` 例外。第二十章再把这里得到的每一条"读不到"转成一张可以直接执行的预期失败表。

先给结论形状：V1、V4 的数值部分、V6 三面**制度性静音**（V4 的拓扑部分与 V2 的配置目录**制度性开放**）、**两面取决于厂商**（V5 的字符设备与 ioctl、V8 的服务调用）、**一面完全不受策略约束但受噪声约束**（V7 的行为反证），V3 则是从 V2 与 V8 的开放部分推出来的反推面。台账里"器件不可见"的记录，绝大多数其实落在前两类里，而不是"这颗处理器不存在"。

本章的表要覆盖七条链：Arm Mali、高通 Adreno GMU、下游 KGSL、Exynos MFC、高通 Venus、联发科媒体 DSP，再加相机 ISP（十九之十）与 NPU/APU（十九之十一）——后两档前十章没给独立章节，本章一次补完，因为它们把 V7 那一列的适用条件戳穿了：七条链里有两条的行为反证只能证明固件在场，证不了版本。

### 十九之一 读数面的定义与八面总表

本文说的"读数面"是一个三元组，缺一项就不算一个面：

1. **采集姿势**：一条只读命令或一个只读路径（不写、不下载、不改参数）。
2. **决定它的规则**：AOSP 里那条真正管住它的 sepolicy 行（文件与行号在本章给出，全部可在第卅章的取回记录里逐字找到）。
3. **失败时的登记值**：必须区分 `absent`（节点/文件不存在）、`unlistable`（父目录不可列举）、`eacces`（策略拒绝，能 `stat` 不能 `read`）、`vendor_dependent`（AOSP 无授权，结果取决于厂商补丁）。把这四态合成一个布尔是本域最常见的采集缺陷，第廿三章把它列为编号陷阱之一。

| 面 | 读什么 | AOSP 里管住它的规则 | shell 默认结果 |
| --- | --- | --- | --- |
| **V1 目录清单** | 固件镜像文件本身（`/vendor/firmware*`、`/firmware_mnt`） | `domain.te:188` 与 `:191-193`；标签来自 `file_contexts:423` | **unlistable**（可 `stat` 已知名，不可列举） |
| **V2 属性与配置目录** | `getprop`、`/vendor/etc` 下的 XML/JSON | `domain.te:177-178`；`file_contexts:427`、`:436-438`；过滤规则 `property.te:193-198` | **可用**（本域唯一向所有域敞开的一档），但对固件版本恒空 |
| **V3 能力声明反推** | 编解码器清单、VINTF 声明、HAL 服务名 | 继承 V2 的可读性 + `shell.te:358`、`:362-380` | **可用**（得到"应有几颗处理器"，得不到版本） |
| **V4 sysfs 设备模型** | `/sys/class/remoteproc`、`/sys/devices/platform/**` 的节点、属性、模式 | `shell.te:443`（仅 `dir`）、`genfs_contexts:123-124`、`domain.te:196-197`、`:255` | **拓扑可用、数值不可读**；bugreport 侧被 `dontaudit` 静音 |
| **V5 字符设备与 ioctl / debugfs** | `/dev/rpmsg*`、`/dev/remoteproc*`、`/sys/kernel/debug/**` | `public/device.te:75-77`、`file_contexts:97`/`:153-154`、`shell.te:424`、`app.te:748-755` | **vendor_dependent**；debugfs 另受构建变体开关 |
| **V6 内核日志** | `dmesg`、`/proc/kmsg`、pstore | `app.te:660-661`（neverallow）、`dumpstate.te:211`（唯一常规授权） | **不可用**（shell 无 `kernel:system`） |
| **V7 行为反证** | 让一次正常编解码/出图成功或失败，反推固件在场与版本闸门 | 不受 sepolicy 约束（走正常功能路径） | **可用**，但只给二值读数 |
| **V8 用户态服务出口** | `dumpsys`、`lshal`、bugreport | `shell.te:358`、`:362-380`、`:510-515`；`binder_call` 仅六条（`:29`/`:56`/`:79`/`:102`/`:214`/`:219`） | **部分可用，媒体/GPU HAL 域取决于厂商** |

### 十九之二 V1 目录清单：可以 stat，不能 ls

标签侧只有一条兜底规则：

> `/(vendor|system/vendor)(/.*)?                  u:object_r:vendor_file:s0`
> — `system/sepolicy@main:private/file_contexts:423`

也就是说第一章列举过的 `/vendor/firmware`、第十一章由 `firmware_class.path=/vendor,/firmware_mnt` 得到的 `/firmware_mnt`，以及 MFC 那条 `/lib/firmware` 搜索路径在设备上的对应目录，全部落在同一个类型 `vendor_file` 上，**AOSP 没有任何一条"固件目录"专用类型**。权限侧的分工则正好卡在列举上：

> `    # This is required to be able to search & read /vendor/lib64`
> `    # in order to lookup vendor libraries. The execute permission`
> `    # for coredomains is granted *only* for same process HALs`
> `    allow domain vendor_file:dir { getattr search };`
> …
> `    # Allow reading and executing out of /vendor to all vendor domains`
> `    allow { domain -coredomain } vendor_file_type:dir r_dir_perms;`
> `    allow { domain -coredomain } vendor_file_type:file { read open getattr execute map };`
> `    allow { domain -coredomain } vendor_file_type:lnk_file { getattr read };`
> — `private/domain.te:185-193`

`getattr search` 与 `r_dir_perms` 的差就是本面的全部：**前者允许你解析并 stat 一个你已经知道名字的路径，后者才允许你把目录里的条目列出来**。而 shell 被明确划在"只拿到前者"的那一侧：

> `typeattribute shell coredomain, mlstrustedsubject;`
> — `private/shell.te:1`

三条合起来得到 V1 的实际形态：`ls /vendor/firmware_mnt/image` 失败（`unlistable`），`stat` 一个由源码告诉你的文件名反而可能成功（因为它只需要 search + getattr）。**这正是 L-5 那句话的机理**——设备侧能确认镜像在场，却不能枚举镜像清单。

由此产生一条必须写进台账契约的规则：**V1 的部分可用性来自静态基线（第十、十一章的 `MODULE_FIRMWARE` 清单、第十四章 MFC 的固定文件名、第十五章 venus 的 `qcom/venus.bin`），不是设备给的读数。** 因此凡是从 V1 得到"文件名"的记录，来源标签必须写 `static_baseline`，不能写 `device_read`；两者混用等于把"上游源码里叫什么"冒充成"这台机器上有什么"。

### 十九之三 V2 属性与配置目录：制度性向所有域敞开的一档

配置目录是唯一被 AOSP 主动开放给所有域的器件侧路径：

> `# All domains get access to /vendor/etc`
> `allow domain vendor_configs_file:dir r_dir_perms;`
> `allow domain vendor_configs_file:file { read open getattr map };`
> — `private/domain.te:176-178`

标签侧给它单独开了口子，并连 VINTF 三个文件一起纳入（`private/file_contexts:427` 的 `/etc(/.*)?`、`:436` 的 `manifest\.xml`、`:437` 的 `compatibility_matrix\.xml`、`:438` 的 `/etc/vintf(/.*)?`，全部标 `vendor_configs_file`）。所以 `/vendor/etc/security/supplemental_security_patches.xml`（第八章）、`/vendor/etc/media_codecs*.xml`、`/vendor/etc/vintf/manifest.xml` 都是 adb shell 能直接读到全文的东西——**本域八面里唯一"读得到完整内容"的文件面**。

属性侧的规则更有信息量，因为它是过滤型的：

> `# init is in coredomain, but should be able to read/write all props.`
> `# dumpstate is also in coredomain, but should be able to read all props.`
> `neverallow { coredomain -init -dumpstate } {`
> `  vendor_property_type`
> `  vendor_internal_property_type`
> `  -vendor_restricted_property_type`
> `  -vendor_public_property_type`
> `}:file no_rw_file_perms;`
> — `private/property.te:191-198`

镜像方向在 `:179-184`（非 coredomain 不得读 `system_*` 内部属性），两个方向都把 `*_public_property_type` 排除在外。含义是：**`getprop` 的输出是一张按策略过滤过的视图，不是设备属性表的完整转储**。厂商内部属性（`vendor.*` 未标 public 的那些）在 shell 域根本不会打印。因此 V2 有一条与其它面共享的铁律：**"getprop 里没有这一项"是策略读数，不是器件读数**，登记时必须写 `filtered_by_policy`，处置动作是查厂商属性上下文而不是查器件。

另外 `neverallow domain property_type:file { ioctl lock };`（`private/property.te:211`）说明属性文件不提供 ioctl 面——第十八章之五列出的可伪造出口只有 `driver_override` 那一条，属性侧没有对应物。

本面在固件域的实测结论是负的：第十四章已在 MFC 驱动里找到四处"版本"却只有一处可用、第十五章之四证明 venus 的固件自报版本只进内核结构体、第十二章之五证明 Adreno GMU 主动丢弃交换上来的版本号，**而这三家都没有给任何 `sys.*`/`ro.*` 属性**。所以 V2 对"固件版本"这一列在本域恒空，只能提供"器件侧安全补丁口径"（第八章）这一类间接记录。

### 十九之四 V3 能力声明反推：不碰器件也能知道该有几颗处理器

V3 的机制在第二章第三条定义句里已经埋好：镜像必须存在否则引擎不工作（`s5p_mfc_ctrl.c:50-51`、失败原句 `:75`）。把它反过来用——**只要设备声明了某个由硬件引擎提供的能力，就必然有一颗处理器带着它的固件在场**。声明面恰好是 V2 里可读的那部分：

- `/vendor/etc/media_codecs*.xml`、`/vendor/etc/media_profiles*.xml`：给出 codec 清单与其是否为硬件加速后端；
- `/vendor/etc/vintf/manifest.xml`（`file_contexts:438`，同属 `vendor_configs_file`）：给出 `android.hardware.media.c2`、`android.hardware.graphics.allocator` 之类 HAL 实例名与版本；
- 服务名列举（`shell.te:358` 的 `allow shell servicemanager:service_manager list;`）：给出 `media.codec`、`media.swcodec` 等是否实际注册。

三条相加，V3 能在**完全不接触 `/sys` 数值、不接触固件文件**的前提下给出"这台机器应有几颗媒体处理器、它们各自的 HAL 是什么"。这是本域唯一一个几乎不受厂商收紧影响的器件侧推断面。

必须钉住它的边界，否则 V3 会被过度解读：

1. V3 给的是**声明**，不是**在场证明**。声明存在而引擎失败是可能状态（V7 就是用来抓这个的）。
2. 静态基线里的字符串枚举（本报告归为证据类型 E3）不是设备读数。把 `grep MODULE_FIRMWARE` 的结果记进 V3 一栏，等于把源码冒充成设备。台账必须把"声明（设备给的）"与"期望（源码给的）"分列两栏。
3. V3 与固件身份之间没有主键。第十六章 B 档的那些公告词，V3 一个也归一化不了——这正是第十八章整章在解释的事。

### 十九之五 V4 sysfs 设备模型：拓扑全开，数值全关

这一面是本报告最重要的一节，因为它的"半开"形态最容易被误读成"器件不存在"。

**第一层：所有 sysfs 路径默认继承同一个标签。**

> `# sysfs labels can be set by userspace.`
> `genfscon sysfs / u:object_r:sysfs:s0`
> — `private/genfs_contexts:123-124`

在本树这份 `private/genfs_contexts`（347 行）里检索 `remoteproc`、`rpmsg`、`/class/drm`、`/class/media`，**命中 0 条**。也就是说 remoteproc 的五个属性、rpmsg 总线属性、drm 与 media 的设备属性全部落在默认的 `sysfs` 类型上。作为对照，被点名的类是这样的：

> `genfscon sysfs /class/gpu                         u:object_r:sysfs_gpu:s0`
> — `private/genfs_contexts:129`

**第二层：shell 在默认 `sysfs` 类型上只有目录权限。**

> `allow shell sysfs:dir r_dir_perms;`
> — `private/shell.te:443`

全文检索本次取回的 `private/shell.te`（552 行），`sysfs:file` 一条也没有；点名到具体 sysfs 类型的授权只有五条：`:410`（`sysfs_net:dir`）、`:445`（`sysfs_batteryinfo:file`）、`:448`（`sysfs_lru_gen_enabled:file`）、`:451`（`sysfs_mem_sleep:file`）、`:460`（`sysfs_dt_avf:dir search`）。而这五个类型里，在 AOSP 的 genfs 表被真正点上名的只有三个——`sysfs_net`（`genfs_contexts:131`）、`sysfs_lru_gen_enabled`（`:176`）、`sysfs_dt_avf`（`:153`）；在本次取回的 `private/genfs_contexts` 与 `private/file_contexts` 里检索 `sysfs_batteryinfo` 与 `sysfs_mem_sleep`，**命中 0 条**，它们的标签只能由设备侧策略补上。于是这一层有两重收紧：**连 `/class/gpu` 这个被 AOSP 点过名的类型（`:129`），shell 也没有文件权限；而 AOSP 给 shell 的两条文件级授权，本身也要靠厂商的 `genfscon` 才落地。** 所以"枚举 ≠ 访问"不是推论，是明文规则的组合。

**第三层：符号链接与遍历对所有域开放，于是拓扑是可采的。**

> `# read and stat any sysfs symlinks`
> `allow domain sysfs:lnk_file { getattr read };`
> — `private/domain.te:196-197`；另有 `allow domain sysfs:dir search;`（`:255`，注释 `# toybox loads libselinux which stats /sys/fs/selinux/` 在 `:252`）

这三层合起来给出 V4 的确切形态：**能列出有哪些节点、能 readlink 判定它挂在哪条父链上、能 stat 到八进制模式与属主；不能读到任何属性值。** 第十七章讲的那五个 remoteproc 属性（`coredump`/`recovery`/`firmware`/`state`/`name`，定义在 `remoteproc_sysfs.c:73`/`:135`/`:167`/`:217`/`:227`）语义再清楚，在这条面上也只能采到"文件在不在、模式是什么"。本章要补的只有一条与模式有关的判断：`rproc_is_visible()`（`remoteproc_sysfs.c:229-243`）在 `rproc->sysfs_read_only` 为真时把四个可写属性降为 `0444`（`:236-240`），而第十七章之六已证明本树内该字段赋值点为 0——**因此 stat 到 0444 就是"有本次未取回的厂商补丁在生效"的信号**，这也是第廿八章脚本把它记成 `degraded_by_vendor` 的原因。

**第四层（决定性）：bugreport 不是"没收集"，是被静默拒绝。**

> `# Access to /sys`
> `allow dumpstate sysfs_type:dir r_dir_perms;`
> — `private/dumpstate.te:275-276`

采集器能列目录；文件级只给了五个类型（`:278-284` 的 `sysfs_devices_block`/`sysfs_dm`/`sysfs_loop`/`sysfs_usb`/`sysfs_zram`，另有 `:100` 的 `sysfs_dmabuf_stats`）。剩下的全部走这一条：

> `# Ignore other file access under /sys.`
> `dontaudit dumpstate sysfs:file r_file_perms;`
> — `private/dumpstate.te:286-287`

`dontaudit` 意味着**拒绝且不留日志**。所以一份 bugreport 里没有 `state`、没有 `firmware`、没有任何媒体 DSP 属性值，既不代表器件没有这些节点，也不代表采集器出错，而是策略按设计静音。这条是本报告"缺失值不等于缺失器件"的制度成因，也直接决定第廿六章 K2/K3 的定义必须把分母写成"含不可读项的节点总数"，否则任何厂商都会被这个静音机制误判成"覆盖率低"。

**第五层（完整性）：V4 的一个读数可以被用户态伪造。** 第十八章之四给出这条链：`modalias` 在有设备树节点时返回 OF modalias、否则才打印服务名（`rpmsg_core.c:457-461`），`driver_override` 是该总线唯一的可写属性（`:449`），带 override 的注册路径把传入字符串直接写进 `rpdev->id.name`（`:617-618`），而用户态可以用 ioctl 把一个 32 字节的名字交给内核（`rpmsg_ctrl.c:84-88`）。**凡是来自 V4 的 rpmsg 名字读数，必须带来源标签并采集同目录下的 `driver_override` 值**；没有这个伴读，名字证据强度不可评估（第廿一章把"读数可被伪造"单列成一个缺陷类型）。

### 十九之六 V5 字符设备与 ioctl：AOSP 只给了类型，没给答案

类型定义侧只有一句话：

> `# All devices have a rpmsg device for`
> `# achieving remoteproc and rpmsg modules`
> `type rpmsg_device, dev_type;`
> — `public/device.te:75-77`

标签侧只覆盖两个历史节点名：

> `/dev/rpmsg-omx[0-9]	u:object_r:rpmsg_device:s0`
> `/dev/rproc_user	u:object_r:rpmsg_device:s0`
> — `private/file_contexts:153-154`

第十八章之五证明 GKI 实际生成的名字是 `rpmsg%d`（`rpmsg_char.c:459`）与 `rpmsg_ctrl%d`（`rpmsg_ctrl.c:168`），它们不在上面那两行里，于是落到兜底标签：

> `/dev(/.*)?		u:object_r:device:s0`
> — `private/file_contexts:97`

而 shell 在通用 `device` 类型上只有一条：`allow shell device:dir getattr;`（`private/shell.te:424`）。在本次取回的 shell 策略里检索 `device:chr_file`，**allow 形式的命中只有五处**专项设备（`:4` uhid、`:339` tty、`:340` console、`:343` input、`:463` ion），**rpmsg/remoteproc 一个都不在**（唯一相关的一条是 `:546` 的 neverallow，对象是 input 设备）。应用侧则被明文禁死：

> `neverallow appdomain {`
> `    audio_device`
> `    camera_device`
> `    dm_device`
> `    radio_device`
> `    rpmsg_device`
> `}:chr_file { read write };`
> — `private/app.te:749-755`，`rpmsg_device` 在 `:754`

结论是本域少数"没有 AOSP 答案"的地方之一：**`/dev/rpmsg*`、`/dev/rpmsg_ctrl*`、`/dev/remoteproc*` 能否打开，完全由厂商策略决定。** 脚本对这一面只能登记三态之一（`absent` / `eacces` / `openable`），绝不写"可以打开并 ioctl"。

节点本身也不一定存在。`rproc_char_device_add()` 只做 `cdev_add`，不创建 devtmpfs 节点：

> `int rproc_char_device_add(struct rproc *rproc)`
> …
> `	rproc->dev.devt = MKDEV(MAJOR(rproc_major), rproc->index);`
> `	cdev_set_parent(&rproc->cdev, &rproc->dev.kobj);`
> `	ret = cdev_add(&rproc->cdev, rproc->dev.devt, 1);`
> — `drivers/remoteproc/remoteproc_cdev.c:98-112`（`:105`、`:106`、`:107`）

设备号确实写进了 `dev.devt`（随后 `remoteproc_core.c:2304` 调它、`:2308` 才 `device_add(dev)`，`:2489` 处 `rproc->dev.class = &rproc_class;`），所以 `/sys/class/remoteproc/rprocN/dev` 的 major:minor 是可推导的线索——但读它属于 V4 的数值面，同样被 `sysfs:file` 缺失挡住。区域容量是 `#define NUM_RPROC_DEVICES 64`（`remoteproc_cdev.c:18`）与 `alloc_chrdev_region(&rproc_major, 0, NUM_RPROC_DEVICES, "remoteproc")`（`:123`）。第十七章之八已给出这张操作表没有读路径（`:91-96` 无 `.read`），本章只补一句：**即便节点存在且能打开，这一面也拿不到固件内容读数。** 写侧能力（`state`/`RPROC_*` ioctl、`coredump_store`、`firmware_store`、模块参数写入、debugfs `crash`）在本文一律只登记"存在与权限位"，不给出任何操作步骤。

debugfs 是本面的第二个子面，它的开关不在策略类型上，而在两个构建变量上：

>     define(`no_debugfs_restriction', ifelse(target_enforce_debugfs_restriction, `true', , $1))
>     define(`userdebug_or_eng', ifelse(target_build_variant, `eng', $1, ifelse(target_build_variant, `userdebug', $1,
>
> — `public/te_macros:591` 与 `:614`（`:589` 的注释句为 `# no_debugfs_restriction`，`:590` 是 `# SELinux rules which apply to devices that do not have debugfs restrictions in non-user builds.`，`:611-612` 是 `# SELinux rules which apply only to userdebug or eng builds`）

于是三个后果同时成立：① shell 的四条 debugfs 授权（`private/shell.te:7-10`）里只有 tracing 相关，`:13-15` 那条 `debugfs_tracing_debug:file rw_file_perms;` 还被 `userdebug_or_eng` 包住——**user 构建上连它都没有**；② venus 的 `venus_dbgfs` 类型不在其中（第十五章之五），标准 adb 姿势不应假设能读 `fw_level`；③ 采集器的 debugfs 读整块被包在 `no_debugfs_restriction` 里（`private/dumpstate.te:290-292`：`allow dumpstate debugfs:file r_file_perms;` 与 `auditallow` 同块），**在启用 debugfs restriction 的设备上，bugreport 根本不含 debugfs 内容**。第廿九章把"debugfs 读不到 = 厂商锁了"列为误区：先看这两个变量。

### 十九之七 V6 内核日志：唯一授权给了采集器，应用被明文禁死

> `# Access to syslog(2) or /proc/kmsg.`
> `neverallow appdomain kernel:system { syslog_read syslog_mod syslog_console };`
> — `private/app.te:660-661`

> `# Read dmesg`
> `allow dumpstate self:global_capability2_class_set syslog;`
> `allow dumpstate kernel:system syslog_read;`
> — `private/dumpstate.te:209-211`

在本次取回的 `private/shell.te` 全文检索 `kernel:system`，命中 0 条。于是 `adb shell dmesg` 在 AOSP 策略下预期失败，日志面只有两条正规通路：抓 bugreport（走上面 `:211` 的授权）或拥有更高权限的环境。**本域所有内核打印证据（第十四章的 `mfc_err` 原句、第十五章之四 venus 的 `dev_dbg` 版本号、第十二章 GMU 的加载失败日志）默认落在这个读不到的面上**，加上驱动自身的开关（MFC 的 `debug` 模块参数、venus 的 `dev_dbg` 需要动态调试打开），构成"三层合取"：读不到文件 ∧ 打印默认关 ∧ 日志默认读不到。

台账规则：V6 必须区分 `日志里没有` 与 `日志读不到` 两种状态，且后者要记录失败码。另有一条边界：`kptr_restrict`、`dmesg_restrict` 的实际取值由厂商 init 决定，本次未取回，列为第卅章边界项——本报告不声明任何厂商在这两个开关上的默认值。

### 十九之八 V7 行为反证：唯一不受策略约束的面，也只给二值读数

V7 的定义：不读任何文件，用一次正常功能调用的成/败反推器件状态。它在本文有三条已证的实现依据：

1. **必须有固件否则不工作**——`s5p_mfc_ctrl.c:50-51` 的注释句与失败原句 `:75`（第二章）。
2. **功能路径里嵌了版本门**——`a6xx_ucode_check_version()` 的注释 `Return true if the ucode is safe.`（第十二章之四）。这就是第四章第三条理由所说的"安全属性被编译进功能路径"，也是本章设计 V7 的依据：GPU 能不能出图会泄漏固件是否落在闸门内。
3. **能力位由版本号决定**——venus 的两处闸门（`vdec.c:571-577` 与 `:710-711`，第十五章之四）把 `core->venus_ver` 用于 VP9 与 EOS 判定，因此"能不能解这个流"是"版本落在哪个区间"的行为投影。

V7 的可信度限制也要写清：失败可能来自 IOMMU、电源、时钟、PAS 授权任何一层（第十五章 firmware.c 的失败路径就是这类）。**所以 V7 读数必须成对记录**——受控样例与对照样例各跑一次，两个结果一起入台账；单边的"失败"不构成版本证据。本面只用正常功能路径，不涉及任何畸形码流、越界尺寸或异常提交序列的构造。

还有一条前提要在本节先立起来，因为它在十九之十会被实测反例限定：**V7 能反推出"版本"的条件是功能路径里真有一道闸门**。上面三条依据之所以成立，全因为那三处代码把版本判断写进了成功路径。若驱动只做"打印警告然后继续装载"，或把完整性校验的开与关交给外设决定，行为读数就退化成"在场性"读数，版本一栏必须记 `not_inferable`。这不是假设，是相机 ISP 与 QAIC 两档的真实形态，十九之十与十九之十一各给一条规则行号。

### 十九之九 V8 用户态服务出口：列举开放，逐个调用取决于厂商

shell 对服务框架有两类权限，形态很有意思：

> `# allow shell access to services`
> `allow shell servicemanager:service_manager list;`
> — `private/shell.te:357-358`

`find` 是"除少数之外全都给"的白名单形式：`:362-380` 的块是 `allow shell { service_manager_type -apex_service -dnsresolver_service -gatekeeper_service -hal_keymint_service -hal_secureclock_service -hal_sharedsecret_service -incident_service -installd_service -mdns_service -netd_service -system_suspend_control_internal_service -system_suspend_control_service -virtual_touchpad_service -vold_service -default_android_service -virtualization_service }:service_manager find;`，另有 `:510-515` 的 `neverallow shell { hal_keymint_service hal_secureclock_service hal_sharedsecret_service virtualization_service }:service_manager find;`。

**`list` 全开意味着 V8 的"枚举服务名"这一半永远可用**（`service list` / `lshal` 依赖它，后者还需 `:386` 的 `allow shell hwservicemanager:hwservice_manager list;`）。但 `dumpsys <svc>` 要真打 binder 过去，还需要 shell 与服务所在域之间的 `binder_call`——本次取回的 shell 策略里这样的调用一共六条，全部点名到具体域：`:29` storaged、`:56` statsd、`:79` gpuservice、`:102` lpdumpd、`:214` hal_keymint、`:219` virtualizationservice。**媒体与 GPU HAL 域一个都不在里面。**

因此：`dumpsys media.codec`、`dumpsys media.player`、`dumpsys SurfaceFlinger` 之类能否出结果，取决于厂商是否补了 `binder_call`。台账记 `vendor_dependent` + 实际错误，不记"应该能读"。

bugreport 不是一张独立的面，它是 V2（配置目录与可读属性）、V4（被 `dontaudit` 削减后的 sysfs）、V6（`dumpstate.te:211`）、V8 的合集；它的覆盖面由前面几节的规则决定，`not_full_treble(...)`（`private/dumpstate.te:191`）之类构建宏还会改变其中 vendor 文件的执行路径。第廿七章脚本 1 因此把 bugreport 只当"别人采过的 V4/V6"来比对，不当独立证据。

### 十九之十 相机 ISP：固件名有四种决定方式，其中一种把闸门做成警告

相机 ISP 前十章没有单独展开，本章补它，是因为它的**固件名解析形态比 GPU 和编解码器都多**：一个只读采集器在 ISP 上要面对三家 in-tree 驱动给出的四种不同决定方式，第四种是"这份树里根本没有对应的代码"。

**形态一：日期优先、失败回落（Intel IPU3）。**

> `#define IMGU_FW_NAME		"intel/ipu3-fw.bin"`
> `#define IMGU_FW_NAME_20161208 \`
> `	"intel/irci_irci_ecr-master_20161208_0213_20170112_1500.bin"`
> — `drivers/staging/media/ipu3/ipu3-css-fw.h:9-11`

装载侧先请求带日期的长名，只有 `if (r == -ENOENT)` 才回落到 `IMGU_FW_NAME`（`ipu3-css-fw.c:120-124`）。于是 V1 面在这个对象上有**两个候选名**，其中一个的名字本身就编码了构建时间（`20161208_0213_20170112_1500`）——本报告全部对象里唯一一个"文件名里带版本"的例子。台账规则随之变细：V1 的 stat 探测必须把两个名字都探，并把结果分成 `absent`（两者皆无）、`partial`（只命中一个）、`present`（两者都在），**不能把"命中一个"记成"在场"**。

版本读数只出现一次，然后就消失：

> `dev_info(dev, "loaded firmware version %.64s, %u binaries, %zu bytes\n", …)`
> — `ipu3-css-fw.c:136-138`

`%.64s` 打的是 `file_header.version`，即**版本号是镜像自带的一个字符串字段**，不是驱动里的编译期常量（这与第十四章 MFC 的四处常量、第十五章 venus 的 `core->venus_ver` 都不同）。该目录内 `MODULE_FIRMWARE` 命中 0 条，所以 E1 类基线只能来自上面那两个 `#define`。装载前的校验是硬失败（`ipu3-css-fw.c:128-134` 头尺寸不符即 `goto bad_fw`），`ipu3-css-fw.c:148-224` 还对每个 binary 逐个校验偏移、长度、文本/指令缓存/数据/填充之和、`offset+size > fw->size`、`iomem_length`、帧格式枚举与 `BLOCK_MAX`——本次取回的集合里最长的一条解析校验链，第廿一章归入缺陷类型学时再展开；这里只登记采集后果：**这套校验全在内核解析路径上完成，运行期不留任何读数**，所以只读采集器既看不到"哪个 binary 被拒"，也看不到"通过了校验的是哪一版"。驱动名是 `#define IMGU_NAME "ipu3-imgu"`（`ipu3.h:16`），它决定 V4 拓扑上的节点名前缀。

**形态二：硬件版本 switch + 两个模块参数 + 警告型闸门（Intel Atom ISP）。**

`atomisp_load_firmware()`（`atomisp_v4l2.c:1128-1171`）按 `isp->media_dev.hw_revision` 三分支选名：`"shisp_2401a0_v21.bin"`（`:1143`）、`"shisp_2401a0_legacy_v21.bin"`（`:1148`）、`"shisp_2400b0_v21.bin"`（`:1153`），无匹配则 `dev_err(isp->dev, "Unsupported hw_revision 0x%x\n", …)`（`:1157`）返回 NULL，命中才 `request_firmware(&fw, fw_path, isp->dev)`（`:1162`）。**固件名由硬件 Revision 决定，而 Revision 是一个 sysfs 不可见的 PCI 配置读数**——所以 V1 的候选名集合在这里是"三选一且选哪个由器件决定"，采集器无法从文件面确定该探哪一个。

两个参数决定了这一面的独特形态：

> `static uint skip_fwload;`
> `module_param(skip_fwload, uint, 0644);`
> `MODULE_PARM_DESC(skip_fwload, "Skip atomisp firmware load");`
> — `atomisp_v4l2.c:58-60`
> `static char firmware_name[256];`
> `module_param_string(firmware_name, firmware_name, sizeof(firmware_name), 0);`
> — `atomisp_v4l2.c:77-79`

权限 `0644` 意味着 `/sys/module/atomisp/parameters/skip_fwload` 会创建且全局可写；权限 `0` 意味着 `firmware_name` 根本不创建 sysfs 文件，覆盖只能走 insmod 参数途径、运行期不留痕。而本次取回的 `private/genfs_contexts` 里被点名的 `/sys/module/**` 只有五条（`:182-186`：`dm_bufio/parameters/max_age_seconds`、`dm_verity/parameters/prefetch_cluster`、`lowmemorykiller`、`tcp_cubic/parameters`、`wlan/parameters/fwpath`），atomisp 不在其中，于是它继承默认 `sysfs`（`:123-124`），shell 只拿到 `allow shell sysfs:dir r_dir_perms;`（`shell.te:443`），全文 `sysfs:file` 0 条。结论是本域独一份的组合：**一个能改变固件装载行为的开关，审计者能列出它、能 stat 到 0644，但读不到它的当前值。** 第廿七章脚本 1 因此对 `/sys/module/*/parameters/` 一律只登记"存在 + 模式"，值一栏写 `unlistable_value`。

版本闸门在这里是第三种姿态——**只警告，不阻断**：

> `dev_err(...)`（`"Firmware version may not be compatible with this driver\n"`、`"Expecting version '%s', but firmware is '%s'.\n"`）
> `/* For now, let's just accept a wrong version, even if wrong */`
> `return false;`
> — `sh_css_firmware.c:202-209`（`sh_css_check_firmware_version()`，`:188-210`）

驱动侧的期望值是编译期常量 `release_version_2401 = STR(irci_stable_candrpv_0415_20150521_0458)` 与 `…2400 = STR(irci_stable_candrpv_0415_20150423_1753)`（`sh_css_firmware.c:59-60`），另有两条含 `REL:`/`API:`/`GIT:`/`SDK:`/`USER:` 五段的长串（`ia_css_version_data.h:26-27`，本报告 E3 类证据里字段最全的一份样本），装载成功后镜像版本被 `strscpy` 进静态缓冲（`sh_css_firmware.c:247-248`）并只以 `IA_CSS_ERROR`/`IA_CSS_LOG` 出现（`:255-259`）。**这条对本章的冲击在 V7**：十九之八把行为反证列为"唯一不受策略约束的面"，其成立前提是功能路径里嵌了版本闸门。atomisp 给出反例——闸门是警告型，错版本照样装载，于是"能出图"完全不蕴含"版本正确"。这是 V7 适用条件第一次被证伪，第廿六章的"行为反证可用率"必须按对象分档，不能整域给一个布尔。

这一家还带来一个 V4 拓扑上的强见证：

> `__ATTR(dbgfun, 0644, iunit_dbgfun_show, iunit_dbgfun_store),`
> `__ATTR(dbgopt, 0644, iunit_dbgopt_show, iunit_dbgopt_store),`
> — `atomisp_drvfs.c:160-161`，经 `driver_create_file(drv, &iunit_drvfs_attrs[i])`（`:164-172`）挂在驱动对象上；驱动名来自 `atomisp_v4l2.c:1606` 的 `.name = "atomisp-isp2"`

所以这两个文件的路径是 `/sys/bus/pci/drivers/atomisp-isp2/{dbgfun,dbgopt}`。它们的**存在本身**证明"atomisp 驱动已绑定到 PCI 上的 ISP"，比 `/dev/video*` 强得多（后者只是 V4L2 框架节点，与固件装载无关）。写侧语义（`OPTION_BIN_LIST BIT(0)`/`OPTION_BIN_RUN BIT(1)`，`atomisp_drvfs.c:46-49`；`iunit_dump_dbgopt()` 在 `:62-63` 调用 `atomisp_css_dump_blob_infor(isp)`，BIN_RUN 需 `isp->asd.streaming` 否则 `:71-80` 返回 `-EPERM`）本文一律只登记存在与模式，不给出任何操作姿势。

**形态三：异步装载 + 从镜像尾部取版本 + 两类镜像（Exynos FIMC-IS）。**

> `#define FIMC_IS_FW_FILENAME		"exynos4_fimc_is_fw.bin"`
> `#define FIMC_IS_SETFILE_6A3		"exynos4_s5k6a3_setfile.bin"`
> — `drivers/media/platform/samsung/exynos4-is/fimc-is.h:32-33`

一颗 ISP 两类镜像，且装载是异步的：`request_firmware_nowait(THIS_MODULE, FW_ACTION_UEVENT, fw_name, &is->pdev->dev, …)`（`fimc-is.c:437-441`，probe 时在 `:852` 以 `FIMC_IS_FW_FILENAME` 调用）。`FW_ACTION_UEVENT` 把"名字→路径"的求解**交给用户态 helper**，那份配置属于设备侧、本次未取回。所以 V1 在这里的登记必须是"驱动给出的候选名"，绝不能写"设备上的路径"——这是 L-5 那条结论（能确认在场、不能枚举清单）的加强形态：**连候选名会落到哪个目录都要厂商配置才知道。**

版本是从镜像尾部按固定长度切出来的：`FIMC_IS_FW_INFO_LEN 31`、`FIMC_IS_FW_VER_LEN 7`、`FIMC_IS_SETFILE_INFO_LEN 39`（`fimc-is.h:47-51`），字段 `info[]`/`version[]`/`setfile_info[]`（`:180-182`）；`buf = (void *)(is->memory.vaddr + fw->size - FIMC_IS_FW_DESC_LEN)` 取 info，再以 `fw->size - FIMC_IS_FW_VER_LEN` 取 version（`fimc-is.c:407-413`），尺寸闸门是 `fw->size < FIMC_IS_FW_SIZE_MIN || fw->size > FIMC_IS_FW_SIZE_MAX`（`:390`，常量 `SZ_4M`/`SZ_32K` 在 `fimc-is.h:60-61`），setfile 侧同法取尾部 39 字节（`:231-257`，`:239` 的失败原句 `"firmware request failed (%d)\n"`）。然后：

> `dev_info(dev, "loaded firmware: %s, rev. %s\n", is->fw.info, is->fw.version);`
> — `fimc-is.c:417-418`（前一行 `:415` 是 `is->fw.state = 1;`）

第三次"版本只进日志"。但 FIMC-IS 多给了一条结构：setfile 装载走状态位等待，两段超时分别打 `"get setfile address timed out\n"` 与 `"loading setfile timed out\n"`（`fimc-is.c:671-686`，随后 `:680` 载入、`:682` `fimc_is_hw_load_setfile(is)`）。**因此多镜像对象的失败天然是两因一果**：固件在场而 setfile 不在场，外在表现与两者都缺一样。第廿三章把这条列为编号陷阱，V7 台账必须按镜像分列而不是按器件分列。

**形态四：这份树里没有（camss / 联发科 ISP / 三星 GSC）。** `drivers/media/platform/qcom/camss/camss.c`（1823 行）检索 `firmware` **0 命中**；`exynos-gsc/gsc-core.c`（1324 行）与 `gsc-core.h`（521 行）同样 **0 命中**（GSC 不是固件对象，是缩放器）；`drivers/media/platform/mediatek/` 目录在 `android15-6.6` 与 `android16-6.12` 两个分支上列举结果相同：`Kconfig Makefile jpeg mdp mdp3 vcodec vpu`——**树内没有 ISP 驱动目录**。第卅章的边界项因此可以从"未取回"改写成一个更强的结论：**三家 in-tree 驱动已取回并逐行核对，另两家在 GKI 内不存在主机侧装载路径**。更强，但对这两家的固件名仍不给。联发科那三条 `SCP_IPI_ISP_CMD`、`SCP_IPI_ISP_FRAME`（`include/linux/remoteproc/mtk_scp.h:40-41`，第十六章已原样引过）与 `SCP_IPI_IMGSYS_CMD` 维持原判：属 L0/L1 命名层，不构成设备读数。

**策略侧给 ISP 的两处新信息。** 第一，`/dev/video[0-9]*` 与 `/dev/nvhdcp1` 被点名成 `video_device`（`private/file_contexts:221`、`:151`），而本次取回的 `private/shell.te` 检索 `video_device` **0 命中**——相机字符设备是**有类型、无授权**的拒绝。这与第十九章之六的 `/dev/rpmsg*`（落兜底 `device`，`file_contexts:97`）**未点名**的拒绝是两种成因，返回给采集器的却是同一个 `EACCES`。第二，V8 在相机侧有 AOSP 锚点：`fwk_camera_service`（`service_contexts:1`）、`ICameraProvider/internal/0` 与 `/virtual/0`（`:47-48`）、`type hal_camera_service, protected_service, hal_service_type, service_manager_type;`（`service.te:311`）。锚点存在不等于能调用——十九之九那六条 `binder_call` 里没有相机域，所以"名字一定列得到、`dumpsys` 取决于厂商"照旧。

### 十九之十一 NPU / APU：accel 框架开出三个新面，与"器件自带固件"这一形态

NPU 一档在公开语料里最少被设备自证，本章补齐后发现原因不在策略，而在**形态本身分成了三类**：主机装载（Intel ivpu）、器件自带（高通 QAIC）、经 remoteproc + 安全异常域装载（高通 CDSP）。三类各在不同面上留缝。

**框架先给三个面。** `drivers/accel/drm_accel.c` 只 227 行，但把 NPU 的设备侧可见性一次定完：

> `return kasprintf(GFP_KERNEL, "accel/%s", dev_name(dev));`
> `accel_class = class_create("accel");`
> — `:31`、`:36`
> `kdev->devt = MKDEV(ACCEL_MAJOR, index);` … `register_chrdev(ACCEL_MAJOR, "accel", &accel_stub_fops)`
> — `:117`、`:217`（固定主设备号，minor 由 `accel_sysfs_device_minor`（`:119`）暴露）
> `accel_debugfs_root = debugfs_create_dir("accel", NULL);`
> — `:215`

于是 `/sys/class/accel/`、`/dev/accel/accelN`、debugfs 根下的 `accel` 目录同时出现。策略侧：`file_contexts` 检索 `accel`、`npu`、`vpu`、`ipu`、`imgu`、`remoteproc` **除兜底外 0 条**，所以 accel 节点落 `/dev(/.*)?  u:object_r:device:s0`（`:97`），而 shell 在 `device:chr_file` 上只有五处专项 allow（`shell.te:4` uhid、`:339` tty、`:340` console、`:343` input、`:463` ion，`:546` 那条是 neverallow 且对象是 input）——**accel 不在内**。`genfs_contexts` 检索 `accel`、`remoteproc`、`rpmsg`、`/class/drm`、`/class/media`、`video` 同样 **0 命中**，`/sys/class/accel/*` 继承 `sysfs`：目录可列（`shell.te:443`）、符号链接可读（`domain.te:196-197`）、属性文件不可读（无 `sysfs:file`）。这就是"拓扑全开、数值全关"在 NPU 上的具体形状，也是最容易被误读的一处：minor 号明明就在一个 sysfs 文件里，而那个文件恰好落在读不到的那一侧。另有一条结构性事实：`drm_accel.c` 全文 227 行里检索 `firmware` **0 命中**——**accel 框架完全不管固件**，装载决定权 100% 在厂商驱动侧，所以任何"框架层统一读数"的设想在这一档从一开始就不成立。

**第一类：主机装载 + 生成代回落表 + 硬闸门（Intel VPU）。**

> `static char *ivpu_firmware;`
> `module_param_named_unsafe(firmware, ivpu_firmware, charp, 0644);`
> `MODULE_PARM_DESC(firmware, "VPU firmware binary in /lib/firmware/..");`
> — `drivers/accel/ivpu/ivpu_fw.c:42-44`（参数优先于表，`:67-71`）
> `/* TODO: Remove mtl_vpu.bin from names after transition to generation based FW names */`　`:46`
> `{ IVPU_HW_37XX, "vpu_37xx.bin" },`、`{ IVPU_HW_37XX, "mtl_vpu.bin" },`、`{ IVPU_HW_37XX, "intel/vpu/vpu_37xx_v0.0.bin" },`、`{ IVPU_HW_40XX, "vpu_40xx.bin" },`、`{ IVPU_HW_40XX, "intel/vpu/vpu_40xx_v0.0.bin" },`　`:51-55`（容器声明 `} fw_names[] = {` 在 `:50`）
> — `ivpu_fw.c:46-56`；`/* Production fw_names from the table above */` + 两条 `MODULE_FIRMWARE`：`:58-60`

请求循环按 `ivpu_hw_gen(vdev)` 过滤后逐个 `firmware_request_nowarn`（`:74-83`，调用在 `:78`），失败由驱动自己打一条 `ivpu_err(vdev, "Failed to request firmware: %d\n", ret);`（`:85`）。**这是本报告 ISP / NPU 两族对象里唯一有 `MODULE_FIRMWARE` 命中的一档**（相机四家与 venus、MFC、`mtk_scp.c` 均 0，见第十四章之六与本章之九；GPU 侧则有 Adreno 上游 10 条与 panthor 1 条，第十二章之一、第十一章之一），所以 E1 基线理论上可从 `modinfo` 取得——但 `vendor_dlkm` 分区里的 `.ko` 本身标 `vendor_file`（`file_contexts:564-565`），coredomain 只有 `getattr search`（`domain.te:185-193`）。**"有 MODULE_FIRMWARE"绝不等于"shell 读得到"**，这条必须在台账的来源标签上分开。

闸门在这里是第四种姿态，也是唯一把两个数字都打出来的硬失败：

> `major < min_major` → `ivpu_err(vdev, "Incompatible FW %s API version: %d.%d, required %d.0 or later\n", …)` → `return -EINVAL;`
> — `ivpu_fw.c:97-101`（仅 major 不等而 `min_major` 满足时只 `ivpu_warn`，`:102-105`；宏 `IVPU_FW_CHECK_API` 在 `:36-40`，BOOT 与 JSM 都以 3 调用：`:171-174`）

版本号的存放位置说明了为什么它只能靠打印：`struct vpu_firmware_header` 里有 `header_version`、`vpu_version[VPU_FW_VERSION_SIZE]`、`firmware_version_load_address`、`firmware_version_size`、`api_version[VPU_FW_API_VER_NUM]`（`vpu_boot_api.h:51-62`），`#define VPU_FW_HEADER_VERSION 0x1`（`:47`），major/minor 的兼容性语义写在 `:10-39` 那段构造说明里（`Major version changes that break backward compatibility.` / `Minor version changes when API backward compatibility is preserved.`）。装载后一条：

> `ivpu_info(vdev, "Firmware: %s, version: %s", fw->name, (const char *)fw_hdr + VPU_FW_HEADER_SIZE);`
> — `ivpu_fw.c:168-169`

`ivpu_fw_parse()` 的八条硬校验（`:112-164`：文件过小 `:118`、`header_version != VPU_FW_HEADER_VERSION` `:123`、runtime 地址区间 `:133`、`runtime_size < fw->size` `:138`、镜像越界 `:143`、加载区间 `:148`、`shave_nn_fw_size` `:155`、`entry_point` 区间 `:160`，每条 `ivpu_err` + `-EINVAL`）与 IPU3 那条链同型，第廿一章一起处理。

本域唯一"运行期可读的固件名"出口在 debugfs：`fw_name_show()` 里 `seq_printf(s, "%s\n", vdev->fw->name);`（`ivpu_debugfs.c:30-36`），它与 `bo_list`、`fw_trace_capability`、`fw_trace_config`、`last_bootmode`、`reset_counter`、`reset_pending` 一起在 `vdev_debugfs_list[]`（`:107-115`），另有 `fw_log_show()`（`:117-124`）与 `fw_log_fops` 的 `.read = seq_read`（`:144-148`）。这一面的开关是构建变量而不是策略类型：`no_debugfs_restriction`（`te_macros:591`）。**所以脚本不能假设 debugfs 可达，更不能假设完整路径**——先探可达性，再按名字列举；本报告不写出 `find` 之外的路径猜测。同一份模式表里 `force_recovery`、`reset_engine`、`fw_trace_destination_mask`、`fw_trace_hw_comp_mask`、`fw_trace_level` 全是 `0200`，`fw_log` 是 `0644`（`ivpu_debugfs.c:273-293`）：`0200` 是"全局可写、谁都读不到"，与 atomisp 那两个 `0644` 参数同属"写侧能力事实表"的一行，本文不给出任何使用姿势。

**第二类：器件自带固件，主机不装载（高通 QAIC）。** 驱动自己的注释把这件事说得比任何分析都清楚：

> `* Invoking this function indicates that the control channel to the device is available. We use that as a signal to indicate that the device side firmware has booted. The device side firmware manages the device resources, so we need to communicate with it via the control channel in order to utilize the device. Therefore we wait until this signal to create the drm dev that userspace will use to control the device, because without the device side firmware, userspace can't do anything useful.`
> — `drivers/accel/qaic/qaic_drv.c:272-281`

**镜像不在主机文件系统里，因此 V1、V2 的固件名基线在这一档结构上就是空的**（不是 `absent`，是"没有这个东西"）。可得的只有协议版本，而且它由外设回答：`#define CNTL_MAJOR 5` / `#define CNTL_MINOR 0`（`:35-36`），闸门

> `if (ret || major != CNTL_MAJOR || minor > CNTL_MINOR) {` → `pci_err(qdev->pdev, "%s: Control protocol version (%d.%d) not supported. Supported version is (%d.%d). Ret: %d\n",` → `__func__, major, minor, CNTL_MAJOR, CNTL_MINOR, ret);` → `ret = -EINVAL;` → `goto close_control;`　`:297-301`
> — 同文件 `:296-302`（`:296` 是取版本调用，`:302` 是收尾花括号）

`get_cntl_version()` 用 `QAIC_TRANS_STATUS_FROM_USR`（`qaic_control.c:1374-1376`；枚举值 `12`/`13`/`14`/`15` 在 `include/uapi/drm/qaic_accel.h:44-47`）向设备查询，再 `*major = status_result->major; *minor = status_result->minor;`（`qaic_control.c:1382-1383`），随后：

> `if (status_result->status_flags & BIT(0)) { /* device is using CRC */ … qdev->valid_crc = valid_crc; } else { … qdev->gen_crc = gen_crc_stub; }`
> — `qaic_control.c:1385-1391`
> `@status_flags: Out. Flags from device.  Bit 0 indicates if CRCs are required.`
> — `qaic_accel.h:145-155`（同段 `@major: Out. NNC protocol version major number.`；`#define QAIC_MANAGE_MAX_MSG_LENGTH SZ_4K` 在 `:17`）

这条给出本域最重要的一条新读数：**完整性校验是否真的在执行，是外设决定的，主机侧在那条分支上装的是一个 stub。** 所以在 QAIC 型器件上，"主机代码里有 CRC 校验"不等于"这台机器在做校验"，而这个状态**没有任何文件面读数**——它只在一次协议查询的结果里，要发那次查询得先打开 `/dev/accel/accelN`（V5，本域预期失败）。本文把它登记成"能力事实 + 不可自证项"，`DRM_QAIC_MANAGE 0x00` / `DRM_IOCTL_QAIC_MANAGE DRM_IOWR(…)`（`qaic_accel.h:375`、`:384`）只作为接口存在的证据，不给出调用构造。旁证两条：MHI 通道名 `{ .chan = "QAIC_CONTROL", }`（`qaic_drv.c:553-566`，驱动名 `"qaic_mhi"`）属 L1 命名层；`/* This is redundant since we have already observed the device crash */`（`:315`）说明 crash 感知路径存在，与 `qaic_data.c:28` 的 `#include <uapi/drm/qaic_accel.h>` 一样，都不给版本读数。

**第三类：remoteproc + 安全异常域（高通 CDSP）。**

> `cdsp_resource_init = { .crash_reason_smem = 601, .firmware_name = "cdsp.mdt", .pas_id = 18, .auto_boot = true, .ssr_name = "cdsp", .sysmon_name = "cdsp", .ssctl_id = 0x17, };`
> — `drivers/remoteproc/qcom_q6v5_pas.c:895-903`（sdm845/sm6350/sm8150/sm8250/sm8350 的 cdsp 条目固件名一律 `"cdsp.mdt"`：`:941`、`:952`、`:968`、`:983`、`:1026`）

同一家内部 `"adsp.mdt"`（`:781`）与 `"adsp.mbn"`（`:791`）两种扩展名并存，说明 **V1 候选名在单家驱动内就不唯一**。装载链是 `request_firmware(&adsp->dtb_firmware, adsp->dtb_firmware_name, adsp->dev)` → `qcom_mdt_pas_init` → `qcom_mdt_load_no_init`（`:214-227`），终点是 `qcom_scm_pas_auth_and_reset(adsp->pas_id)`（`:301`，dtb 分支 `:280`）；`fw_name = desc->firmware_name;`（`:664`）交给 `rproc = rproc_alloc(&pdev->dev, pdev->name, ops, fw_name, sizeof(*adsp));`（`:681`）。三重采集后果：① **校验者是 TrustZone**，不通过时驱动只拿到一个错误码，版本与被拒原因都不进日志，V6 在这一档比第十五章的 venus 更空；② rproc 名 = 平台设备名 = DT 节点名，所以 V4 拓扑读到的是节点名而不是镜像名，而 `"cdsp.mdt"` 只出现在 remoteproc 的 `firmware` 属性里——那属于 V4 数值面，AOSP 层 `✗`；③ `qcom_q6v5_pas.c` 与 `qcom_common.c` 检索 `DEVICE_ATTR|driver_create_file|device_create_file` **0 命中**，没有专用 sysfs 出口，SSR 名在内核里以 notifier 形式存在（`qcom_common.c:399 qcom_register_ssr_notifier()`），既不经文件也不经属性暴露。`.parse_fw = qcom_register_dump_segments,`（`:419`、`:429`）说明崩溃转储段登记存在，属写侧/取证侧能力，本文同样只登记。

**设备树是第四种"知道但读不到"。** 对 SCP 这一类，固件名可能根本不是驱动常量：`const char *fw_name = "scp.img";`（`drivers/remoteproc/mtk_scp.c:846`）之后立刻 `ret = rproc_of_parse_firmware(dev, 0, &fw_name);`（`:849`），再 `devm_rproc_alloc(dev, np->name, &scp_ops, fw_name, sizeof(*scp));`（`:853`），而 `rproc_of_parse_firmware()` 读的是 DT 属性 `"firmware-name"`（`drivers/remoteproc/remoteproc_core.c:998-1006`，属性名在 `:1002`，`EXPORT_SYMBOL` 在 `:1006`）。策略侧刚好看过 DT：`genfs_contexts:153-154` 只点名 `sysfs /firmware/devicetree/base/avf` 与 `…/base/firmware/android` 两个子树，其余 DT 路径继承 `sysfs`；shell 在 AVF 类型上只有 `allow shell { proc_dt_avf sysfs_dt_avf }:dir search;`（`shell.te:460`）。于是 DT 出现**第三种姿态**：普通 DT 节点目录因继承 `sysfs` 而**可列举**，但每个属性文件（含 `compatible`、含 `firmware-name`）因 `sysfs:file` 缺失而**不可读**；AVF 子树反过来只给 search、不给 list。把这两句接起来就是本报告里名字可知性与可读性错配最干净的一例：**器件的固件名可以由设备树决定，而设备树的内容恰好读不到。**

**NPU 在 V3/V8 上的空档要单独说。** `private/service_contexts` 检索 `neuralnetwork`、`npu` **0 命中**，而同文件 `:291` 有 `gpu → gpu_service`、`:67-70` 有 `IComponentStore/{default,default1,default2,software} → hal_codec2_service`。类型属性上：`hal_camera_service` 带 `protected_service`（`service.te:311`），`hal_neuralnetworks_service`（`:343`）不带，`hal_codec2_service`（`:314`，它带的是另一个属性 `isolated_compute_allowed_service`）也不带。本报告只描述声明，不推断这条属性的强制后果——它的 neverallow 规则不在本次取回的文件里，第卅章单列为边界项。另有一条与器件无关的构建事实：`service.te:329-331`、`:339-341` 与 `genfs_contexts:172-174` 被包在 `starting_at_board_api(202504, \`…`\)` 里，即**某些类型与某些 genfs 标签连存在与否都由构建参数决定**（该宏的定义不在本次取回的四个文件内，本报告不声明它的展开条件），与本域 `userdebug_or_eng`（`te_macros:614`）、`no_debugfs_restriction`（`:591`）同属一类需要登记成构建变量的读数。于是本域的结论是：**在 AOSP 层，GPU 是唯一一个 shell 既能点名服务、又能 `binder_call` 的加速器**（`shell.te:79`），媒体与 NPU 两者都不满足。

### 十九之十二 七条链 × 八面：谁在哪一面有东西

符号：`✓` AOSP 规则下预期可读；`✗` AOSP 规则下预期读不到；`△` 取决于厂商；`—` 该面对这个对象结构上不适用（不是"读不到"，是"没有这个东西"）。

| 对象 | V1 清单 | V2 属性/配置 | V3 声明 | V4 拓扑 | V4 数值 | V5 字符设备 | V6 日志 | V7 行为 | V8 服务 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Arm Mali CSF**（第十、十一章） | ✗（`vendor_file`） | ✗（无版本属性） | ✓（HAL/能力声明） | ✓（drm 节点可枚举） | ✗ | △ | ✗ | ✓（出图/作业提交成否；静默降级路径） | △ |
| **Qualcomm Adreno GMU**（第十二章） | ✗ | ✗（版本号被主动丢弃） | ✓ | ✓（`/class/gpu` 可枚举但 shell 无文件权限） | ✗ | △ | ✗ | ✓（`a6xx_ucode_check_version` 类闸门） | △ |
| **下游 KGSL**（第十三章） | ✗ | △（厂商自建 sysfs 节点） | ✓ | ✓ | △ | △ | ✗ | ✓ | △ |
| **Exynos MFC**（第十四章） | ✗ | ✗（`MODULE_FIRMWARE` 0 命中） | ✓ | ✓（media 平台设备可枚举） | ✗ | ✗（无 rpmsg/rproc 面） | ✗（需 `debug` 参数） | ✓（无固件则不工作） | △ |
| **Qualcomm Venus**（第十五章） | ✗ | ✗（`venus_ver` 只进内核结构体） | ✓ | ✓ | ✗ | ✗（rpmsg/rproc 0 命中） | ✗（`dev_dbg`） | ✓（VP9/EOS 两处闸门） | △ |
| **MediaTek VPU + SCP**（第十五、十七、十八章） | ✗ | ✗ | ✓ | ✓（`/sys/class/remoteproc`、`/dev/rpmsg*` 可枚举） | ✓（remoteproc 五属性，AOSP 唯一自带标准读数面的对象） | △ | ✗ | ✓ | △ |
| **相机 ISP：IPU3 / atomisp / FIMC-IS**（十九之十） | ✗（两家候选名固定、一家三选一、一家交给 uevent） | ✗（三家都无版本属性） | ✓ | ✓（`/sys/module/atomisp/parameters/`、`/sys/bus/pci/drivers/atomisp-isp2/` 可列） | ✗（参数值与 `dbgfun`/`dbgopt` 内容皆不可读） | ✗（`/dev/video*` = 有类型无授权型拒绝） | ✗（版本三家都只进日志） | △（atomisp 因警告型闸门只能证明在场） | ✓（`hal_camera_service` 有 AOSP 锚点，`dumpsys` 仍 △） |
| **NPU / APU：ivpu / qaic / CDSP**（十九之十一） | ✗（ivpu 有 `MODULE_FIRMWARE` 但 `.ko` = `vendor_file`；qaic 结构上无主机镜像） | ✗ | △（本树 `service_contexts` 无 neuralnetworks 条目） | ✓（`/sys/class/accel`、remoteproc 节点与 DT 节点目录可列） | ✗（minor 与 `firmware-name` 都在读不到的一侧） | ✗（`/dev/accel*` = 未点名兜底型拒绝） | ✗（ivpu 只打印；PAS 失败只给错误码） | △（ivpu 硬闸门可用；qaic 的校验开关由外设决定） | △ |

五点必须从这张表里读出来：

1. **`V4 数值` 一列只有联发科那格可能真是 `✓`**，而且它成立的原因是第十七章讲的 remoteproc 标准属性 + 厂商额外补的 sysfs 授权，不是 AOSP 给了这条路——AOSP 层面 `sysfs:file` 对 shell 一条都没有（第十九章之五）。这一格记 `✓（vendor_granted）`，绝不记 `✓`。
2. **V1 与 V6 两列整列是 `✗`**，原因还不一样（coredomain 划分 / 无 `kernel:system`）。V2 一列也基本是 `✗`，唯一的 `△` 在下游 KGSL 那一格，成因是厂商自己建了节点与授权。把这三列合并成"设备侧什么都读不到"会丢掉定位所需的信息。
3. **V7 不再整列是 `✓`**。补上 ISP 与 NPU 两行之后它在这两行都是 `△`，而降级原因正好是十九之八那条前提的两个反例：atomisp 的版本闸门是警告型（`sh_css_firmware.c:208-209`），行为成功不蕴含版本正确；qaic 的完整性校验开不开由外设决定（`qaic_control.c:1385-1391`），主机侧在那条分支上装的是 stub。**行为反证仍然可用，但它的可用性是逐对象的属性，不是逐面的属性**——这是本章对十九之八最重要的一次自我修正。

4. **同一个 `EACCES` 有两种成因**：相机字符设备是"有类型、无授权"（`video_device` 在 `file_contexts:221`，本次取回的 `shell.te` 检索它 0 命中），accel 与 rpmsg 是"未点名、落兜底 `device`"（`file_contexts:97`）。前者厂商补一条授权就能开门，后者要先补标签再补授权。第廿章按这两种成因给两个不同的信号码，不能都写成"权限不足"。
5. **可写开关的状态不可核验**：`skip_fwload` / `firmware_name`（`atomisp_v4l2.c:58-60`、`:77-79`）、`dbgfun` / `dbgopt`（`atomisp_drvfs.c:160-161`）、ivpu 的 `force_recovery`、`reset_engine`、`fw_trace_*` 是 `0200` 而 `fw_log` 是 `0644`（`ivpu_debugfs.c:273-293`）——存在与模式在 V4 拓扑面上可得，**当前值一律不可得**（`sysfs:file` 缺失）。审计报告若写"某写侧开关未被启用"，那是没有读数支撑的断言。

补这两行的代价是第廿五章归属矩阵与第廿八章脚本的信号码集都要扩：ISP 要按三个候选名分别 stat（IPU3 两个、atomisp 三个按 Revision 分支、FIMC-IS 两类镜像），NPU 要先把 debugfs 可达性判出来再谈 `fw_name`。收益是把"设备侧读不到固件版本"从四条链推广到七条链，并且证明**七条链里没有任何一条能靠文件面给出一个可比对的固件版本号**——这才是本章标题那句"被策略静音的"的完整含义。

### 十九之十三 本章结论

1. **八面里没有一个能给出固件版本号**：V1 不给清单、V2 不给属性、V4 不给数值、V6 不给日志、V7 只给二值、V5/V8 取决于厂商、V3 只给声明。这是本域的结构性结论，不是采集工具的问题。
2. **"列目录可以、读文件不行"是明文规则的组合**：`shell.te:443`（`sysfs:dir`）+ `genfs_contexts:123-124`（未枚举即继承默认）+ `shell.te` 无 `sysfs:file` + `domain.te:196-197`（符号链接全开）。所以拓扑与存在性可采，数值不可采。
3. **bugreport 的缺字段是设计行为**：`dontaudit dumpstate sysfs:file r_file_perms;`（`dumpstate.te:287`）拒绝且不留痕。任何覆盖率指标都必须把不可读项放进分母。
4. **`getprop` 是过滤后的视图**：`property.te:191-198` 的 neverallow 只放行 `vendor_public_property_type`；缺席是策略读数，不是器件读数。
5. **V5 与 V8 在 AOSP 里没有答案**：类型给了（`device.te:75-77`）但标签只覆盖两个历史节点名（`file_contexts:153-154`），shell 既无 `device:chr_file` 也无对媒体/GPU HAL 域的 `binder_call`。凡这两面的读数必须带 `vendor_dependent` 与实际错误码。
6. **两个构建变量能整面关掉采集**：`userdebug_or_eng`（`te_macros:614`）与 `no_debugfs_restriction`（`te_macros:591`）。debugfs 与 tracing 相关读数在 user 构建上不存在，这是构建事实而非厂商锁。
7. **V4 的一个读数可被用户态伪造**（`rpmsg_ctrl.c:84-88` + `rpmsg_core.c:618`）：rpmsg 名字类读数必须伴读 `driver_override` 并带来源标签。
9. **七条链全部补齐之后，V4 数值一列仍然只有联发科那一格可得**：相机三家（IPU3 / atomisp / FIMC-IS）与 NPU 三类（ivpu / qaic / CDSP）的版本读数都只出现在内核打印里，`dev_info`（`ipu3-css-fw.c:136-138`、`fimc-is.c:417-418`）、`ivpu_info`（`ivpu_fw.c:168-169`）与 `IA_CSS_LOG`（`sh_css_firmware.c:255-259`）是同一类出口——**都在 shell 域读不到的 V6 面上**。
10. **拒绝有两种成因，处置不同**：有类型无授权（`video_device`，`file_contexts:221`）与未点名落兜底（`/dev/accel*`、`/dev/rpmsg*`，`file_contexts:97`）。观测面同为 `eacces`，策略侧要补的东西完全不同，第廿章给两个信号码。
11. **固件名的决定者有五种，采集姿势必须随之变**：固定串（IPU3、FIMC-IS、SCP 默认名）、日期优先回落（`ipu3-css-fw.h:9-11` + `ipu3-css-fw.c:120-124`）、硬件 Revision 三分支（`atomisp_v4l2.c:1143`/`:1148`/`:1153`）、生成代表回落链（`ivpu_fw.c:46-56`）、设备树属性（`mtk_scp.c:849` → `remoteproc_core.c:1002`），外加"主机根本不装载"（`qaic_drv.c:272-281`、`camss.c` 0 命中）。**候选名集合不唯一是本域的常态，因此 V1 的登记值必须支持 `partial`。**
12. **一处"知道但读不到"的极端形态**：器件的固件名可以由设备树属性 `firmware-name` 决定，而 DT 属性文件恰好继承 `sysfs`、shell 无 `sysfs:file`，DT 节点目录却可列举（`genfs_contexts:153-154` 只点名 AVF 与 android 两个子树，`shell.te:443`/`:460`）。名字由谁决定、名字是什么，这两件事在 shell 域分别是"能列出来"和"读不到"。
13. 八面的失败姿态在第二十章转成一张按 shell 姿势组织的预期失败表；脚本 1 的每个信号码就是那张表的一行。
## 二十、shell 域预期失败表：二十六个采集姿势，逐条给出失败码、成因与处置

第一章第四条承诺的后半件事在这章做完。第十九章回答的是"一个读数面被哪条规则管住"，本章回答的是采集者坐在 `adb shell` 前面时的具体问题：**这条命令敲下去会发生什么，发生之后你能写进台账的是什么、禁止写的是什么。** 表里每一行的"规则"列都是第十九章已经给过原句的那一条，因此这张表不是经验总结，而是那批规则行的可执行转置。

这张表还有第二个用途：它是第廿七章脚本 1 的**规格**。脚本输出的每一个信号码都是这里的一行，脚本的分支集合必须与这张表的行数一致——"表里有、脚本里没有"和"脚本里有、表里对不上"都算报告缺陷，第廿九章的分支覆盖验收就是照这条判的。

### 二十之一 这张表的构造规则

一条姿势进表要同时满足四条，缺一条就重写成两条或删掉：

1. **只读**。姿势只能是 `ls`、`stat`、`readlink`、`cat`、`getprop`、`find`、`service list`、`lshal`、`dumpsys <svc>`、`dmesg`、bugreport 抓取。任何 `echo >`、`chmod`、`insmod`、`rmmod`、ioctl 调用、写 sysfs 的动作一律不进表；第十九章之五与十九之十登记的写侧能力（`firmware_store`、`state_store`、`dbgfun`/`dbgopt`、`force_recovery`、`reset_engine`、`fw_trace_*`、`RPMSG_*_IOCTL`、`DRM_IOCTL_QAIC_MANAGE`）在第廿五章的能力事实表里，不在这里。
2. **四态分开**。`absent`（路径不存在）、`unlistable`（父目录不能列举，但已知名可 stat）、`eacces`（策略拒绝：能 stat 不能 read）、`vendor_dependent`（AOSP 无授权，结果取决于厂商补丁）。把这四态压成一个布尔是本域最常见的采集缺陷，第廿三章列为编号陷阱。
3. **来源标签**。从静态基线（第十至十九章的源码 `#define`、`MODULE_FIRMWARE`、搜索路径）得到的名字标 `static_baseline`；设备真给的东西标 `device_read`；`getprop` 的缺席标 `filtered_by_policy`。V1 的可用性本身就来自基线（十九之二），不标等于把源码冒充成设备。
4. **成因可判别**。每个 `eacces` 必须能由表内另一条姿势区分出"有类型无授权 / 未点名落兜底 / 未打标 / dontaudit 静音 / 构建变量关闭"五种成因之一，区分不出来就不配进表——所以 P-05（取上下文标签）是这张表的地基而不是装饰。

### 二十之二 组 A｜目录与文件（V1，P-01 至 P-06）

| ID | 姿势 | 管住它的规则 | AOSP 预期 | 登记值 | 处置 / 禁止推出 |
| --- | --- | --- | --- | --- | --- |
| **P-01** | `ls /vendor/firmware_mnt/image`、`ls /vendor/firmware` | `domain.te:188`（`allow domain vendor_file:dir { getattr search };`）+ `shell.te:1`（shell 是 coredomain）+ `file_contexts:423` | 拒绝 | `unlistable` | 转 P-02 逐名 stat。**禁止**写"目录为空"或"该器件无固件目录"——列不出与不存在是两件事 |
| **P-02** | 对每个静态候选名 `stat -c '%n %s %A' <路径>` | `domain.te:185-193` 的 `getattr search`（search 足以解析路径，getattr 足以 stat） | **可能成功** | `present` / `absent` + `static_baseline` | 成功只证明"这个名字在这台机器上"，不证明版本、不证明被装载、不证明是最新版。**禁止**把 `present` 记成 `device_read` 之外的任何强度 |
| **P-03** | `ls /vendor/etc/firmware`、`cat /vendor/etc/firmware/<名>` | `file_contexts:427`（`/(vendor\|system/vendor)/etc(/.*)?  u:object_r:vendor_configs_file:s0`）+ `domain.te:177-178`（`allow domain vendor_configs_file:dir r_dir_perms;`、`…:file { read open getattr map }`） | **可读** | `present` + `device_read` | 这是 V1 在本域唯一的制度性例外：**固件若被厂商放在配置目录下而不是 firmware 目录下，则清单与内容都向所有域敞开**。禁止把它当普遍情形——MFC、venus、IPU3、atomisp、ivpu 的候选名全都不在 `/vendor/etc` 下（第十至十一章、十四至十五章、十九之十、十九之十一） |
| **P-04** | `ls /firmware_mnt`、`ls /firmware_mnt/image` | 本次取回的 `private/file_contexts` 检索 `firmware` **0 命中**（961 行全文），`private/vendor_file_contexts` 在 `main` 分支**不存在**（`?format=TEXT` 返回 404 `NOT_FOUND`），vendor 侧唯一可取回的标签表 `vendor/file_contexts`（212 行）内检索 `firmware` **0 命中** → 根级挂载点在 AOSP 侧没有任何标签，只能由厂商 `file_contexts` 补 | `vendor_dependent` | `eacces` / `absent` / `unlabeled` 三态之一 | 必须配 P-05 才能定成因。AOSP 只有一条相关的 neverallow：`neverallow { domain -init -recovery } unlabeled:dir_file_class_set create;`（`domain.te:616-617`，注释 `# Do not allow any domain other than init to create unlabeled files.`），说明 `unlabeled` 这一档在策略里是被承认的状态，但它不给任何域读权限 |
| **P-05** | `ls -Zd /vendor/firmware_mnt/image /dev/video0 /dev/accel/accel0 /sys/class/remoteproc 2>&1` | 标签来自 `file_contexts:423`/`:97`/`:221` 与 `genfs_contexts:123-124`；取标签只需要 `getattr`，而 `getattr` 已由 `domain.te:188`、`shell.te:424`（`allow shell device:dir getattr;`）给出 | **成功** | `label=<类型>` | 这条是全表的地基：它把同一个 `eacces` 拆成"有类型无授权"、"落兜底 `device`/`sysfs`"、"未打标 `unlabeled`"。取不到标签单独登记 `label_unreadable`，并在第卅章标为边界项——本报告不声明 `ls -Z` 在所有厂商构建上的可达性 |
| **P-06** | `ls /vendor_dlkm/lib/modules`、`ls /vendor_dlkm/etc` | `file_contexts:564`（整个 `vendor_dlkm` → `vendor_file`）与 `file_contexts:565`（其 `/etc(/.*)?` → `vendor_configs_file`）；odm 同型 `:571-572` | 前者 `unlistable`、后者**可读** | 两栏分开登记 | 直接后果：**ivpu 那两条 `MODULE_FIRMWARE`（`ivpu_fw.c:58-60`）虽然在 `.ko` 里，但 `.ko` 落 `vendor_file`，modinfo 提取不是 shell 可达姿势**；而 `vendor_dlkm/etc` 下的模块配置文件全文可读。`domain.te:1481-1483` 的 module_load neverallow 与此无关，别混用 |

### 二十之三 组 B｜属性与配置目录（V2/V3，P-07 至 P-11）

| ID | 姿势 | 管住它的规则 | AOSP 预期 | 登记值 | 处置 / 禁止推出 |
| --- | --- | --- | --- | --- | --- |
| **P-07** | `getprop \| grep -iE 'fw\|firmware\|gmu\|mfc\|venus\|vpu\|npu\|isp'` | `property.te:191-198`（`neverallow { coredomain -init -dumpstate } { vendor_property_type vendor_internal_property_type -vendor_restricted_property_type -vendor_public_property_type }:file no_rw_file_perms;`）——shell 是 coredomain 且不例外 | 输出被**过滤**，厂商内部属性不打印 | `filtered_by_policy` | "getprop 里没有这一项"是策略读数不是器件读数。处置动作是去查厂商的属性上下文文件，不是去查器件。**禁止**据此断言"该器件不上报固件版本"。本域实测七条链里没有一条给 `sys.*`/`ro.*` 固件版本（十九之三、十九之十、十九之十一） |
| **P-08** | `getprop ro.build.version.security_patch`、`getprop ro.bootloader`、`getprop ro.vendor.build.fingerprint` | `property.te:191-198` 的排除集（public 属性可读） | 可读 | `device_read` | 这是 **Android 侧口径**，与器件镜像无关（第八章）。禁止用它反推任何一段器件镜像的新旧 |
| **P-09** | `cat /vendor/etc/media_codecs.xml`、`cat /vendor/etc/media_codecs_c2.xml`、`cat /vendor/etc/media_profiles*.xml` | `file_contexts:427` + `domain.te:176-178` | **可读** | V3 声明 | 得到"应有几颗硬件编解码引擎、HAL 是哪个"。得到不了版本。禁止把声明记成在场证明——声明在而引擎失败是可能状态（用 P-24/P-25 抓） |
| **P-10** | `cat /vendor/etc/vintf/manifest.xml`（或 `/vendor/manifest.xml`） | `file_contexts:438`、`:436-437`（vintf 与两个 VINTF 文件同标 `vendor_configs_file`） | **可读** | V3 声明（HAL 实例名 + 版本） | 这里的 version 是 **HAL 接口版本**，不是固件版本。两者同名字段最容易抄错，第廿三章列入误传来源 |
| **P-11** | `cat /vendor/etc/security/supplemental_security_patches.xml` | 同 `file_contexts:427`，落 `vendor_configs_file` | **可读** | 器件侧安全补丁口径 | 第八章已用它说明"器件侧口径与公告 SPL 是两本账"。禁止把它当固件版本表 |

### 二十之四 组 C｜sysfs 拓扑与数值（V4，P-12 至 P-18）

| ID | 姿势 | 管住它的规则 | AOSP 预期 | 登记值 | 处置 / 禁止推出 |
| --- | --- | --- | --- | --- | --- |
| **P-12** | `ls /sys/class/remoteproc`、`ls /sys/class/accel`、`ls /sys/class/drm`、`ls /sys/class/video4linux`、`ls /sys/devices/platform` | `shell.te:443`（`allow shell sysfs:dir r_dir_perms;`）+ `genfs_contexts:123-124`（`genfscon sysfs / u:object_r:sysfs:s0`）；本树对 `accel`、`remoteproc`、`rpmsg`、`/class/drm`、`/class/media`、`video` 的点名数为 **0** | **可列** | `device_read`（拓扑） | 这是本域最可靠的一档读数：有哪些处理器节点、叫什么名。`/sys/devices/platform` 在本树只有 `:139` 一条被点名（`nfc-power/nfc_power`），所以 MFC、venus、camss、remoteproc 的平台节点全部继承默认类型而**可列** |
| **P-13** | `readlink /sys/class/remoteproc/rproc0`、`readlink /sys/class/accel/accel0`、`readlink /sys/class/drm/card0` | `domain.te:196-197`（`# read and stat any sysfs symlinks` + `allow domain sysfs:lnk_file { getattr read };`）、`domain.te:255`（`allow domain sysfs:dir search;`） | **可读** | `device_read`（父链） | `/sys/class/**` 的每一项都是符号链接，所以**类目录的名字与落点都能读到**；`/devices/**` 路径最后一段就是平台设备名，而 remoteproc 名 = 平台设备名 = DT 节点名（`qcom_q6v5_pas.c:681` 的 `rproc_alloc(&pdev->dev, pdev->name, …)`、`mtk_scp.c:853` 的 `devm_rproc_alloc(dev, np->name, …)`）。禁止把节点名当镜像名 |
| **P-14** | `cat /sys/class/remoteproc/rproc0/state`、`…/firmware`、`…/name` | `shell.te` 全文检索 `sysfs:file` **0 条**；五个属性全部继承默认 `sysfs`（`genfs_contexts:123-124`），路径解析那一侧由 `shell.te:443` 提供 dir search（同 P-15） | 拒绝 | `eacces`（denied_value） | 五个属性的语义在第十七章给全（定义在 `remoteproc_sysfs.c:73`/`remoteproc_sysfs.c:135`/`remoteproc_sysfs.c:167`/`remoteproc_sysfs.c:217`/`remoteproc_sysfs.c:227`）。**"知道它表示什么"与"读得到它的值"是两件事**；`firmware` 这一项正是 CDSP/SCP 的镜像名所在（十九之十一），所以这一格拒绝直接决定了本域的自证上限 |
| **P-15** | `stat -c '%n %a' /sys/class/remoteproc/rproc0/{coredump,recovery,firmware,state,name}` | 同 P-14，但 `stat` 只需要 `dir` 的 search 与文件的 getattr（`domain.te:196-197` 给 lnk、`shell.te:443` 给 dir） | **成功** | 八进制模式原样记录 | 模式与"我能否读"必须分两列。`rproc_is_visible()`（`remoteproc_sysfs.c:229-243`）在 `rproc->sysfs_read_only` 为真时把四个可写属性降为 `0444`（`remoteproc_sysfs.c:236-240`），而本树该字段赋值点为 0（十七之六）→ **stat 到 0444 就是"有本次未取回的厂商补丁在生效"**，脚本记 `degraded_by_vendor` |
| **P-16** | `ls /sys/module/atomisp/parameters`、`stat -c '%n %a' /sys/module/*/parameters/*` | `genfs_contexts:182-186` 里被点名的 `/sys/module/**` 只有五条，atomisp/ivpu 不在其中 → 继承 `sysfs`（`genfs_contexts:123-124`）；shell 无 `sysfs:file` | 目录可列、模式可 stat、**值不可读** | `unlistable_value` | 这是本域独有的组合：`skip_fwload` 是 `0644`（`atomisp_v4l2.c:58-60`），能改变装载行为，但其当前值在 shell 域不可核验。**禁止**在任何审计报告里写"该开关未启用"。对照：`firmware_name` 的参数权限是 `0`（`:79`），sysfs 文件根本不创建，运行期不留痕 |
| **P-17** | `ls /sys/bus/pci/drivers/atomisp-isp2/`、`stat -c '%a' /sys/bus/pci/drivers/atomisp-isp2/dbg*` | 同 P-16（`/sys/bus/**` 在本树 0 点名，因此继承 `genfs_contexts:123-124` 的默认 `sysfs`）；驱动对象上的两个文件由 `driver_create_file()` 创建（`atomisp_drvfs.c:164-172`，`__ATTR(dbgfun, 0644, …)`/`__ATTR(dbgopt, 0644, …)` 在 `atomisp_drvfs.c:160-161`，驱动名来自 `atomisp_v4l2.c:1606`） | **可列** | `bound` / `absent` | 这两个文件的**存在**是"驱动已绑定到 PCI 上的 ISP"的见证，强度高于 `/dev/video*`（后者只是 V4L2 框架节点，与固件无关）。只登记存在与模式，不给任何写姿势 |
| **P-18** | `ls /sys/firmware/devicetree/base/`、`ls <某节点目录>`、`cat <某节点>/compatible`、`cat <某节点>/firmware-name` | `genfs_contexts:152-154` 只点名 `acpi/tables`、`devicetree/base/avf`、`devicetree/base/firmware/android` 三个子树，其余继承 `sysfs`；shell 在 AVF 上只有 `allow shell { proc_dt_avf sysfs_dt_avf }:dir search;`（`shell.te:460`） | 节点目录**可列**、属性文件**不可读**；AVF 子树反过来只能 search 不能 list | 两栏分开 | 直接对上十九之十一末段：`rproc_of_parse_firmware()` 读的就是 DT 属性 `"firmware-name"`（`remoteproc_core.c:998-1006`，属性名在 `remoteproc_core.c:1002`；调用点 `mtk_scp.c:849`）。**器件的固件名可以由设备树决定，而设备树的内容恰好读不到**——这一行是本报告"可知性与可读性错配"的最干净样本 |

### 二十之五 组 D｜字符设备与 debugfs（V5，P-19 至 P-23）

| ID | 姿势 | 管住它的规则 | AOSP 预期 | 登记值 | 处置 / 禁止推出 |
| --- | --- | --- | --- | --- | --- |
| **P-19** | `ls -lZd /dev/video0`、`stat -c '%C' /dev/video*` | `file_contexts:221`（`/dev/video[0-9]*  u:object_r:video_device:s0`，另 `:151` 的 `/dev/nvhdcp1` 同类型）；本次取回的 `private/shell.te` 检索 `video_device` **0 命中**；本姿势与 P-20 共用一个采集循环、按标签分派 pid，所以脚本在这一行也会用到 P-20 的两个锚点：`shell.te:424`（shell 在 `/dev` 上只有 dir getattr，没有 search/read，这是两种 `eacces` 的共同成因）与 `file_contexts:97`（当 `/dev/video0` 本身落进兜底 `device` 时，即厂商没打标的那一类构建） | 拒绝 | `eacces_typed` | 成因是**有类型、无授权**：厂商补一条 shell 授权即可开门。与 P-20 必须用不同信号码，否则处置动作会写错 |
| **P-20** | `ls -lZd /dev/rpmsg* /dev/rpmsg_ctrl* /dev/remoteproc* /dev/accel/accel*` | `shell.te:424`（`allow shell device:dir getattr;`，只有 getattr 没有 search/read）；`file_contexts:97` 兜底 `/dev(/.*)?  u:object_r:device:s0`；`:153-154` 只点名两个历史名 `/dev/rpmsg-omx[0-9]` 与 `/dev/rproc_user`（都在本次取回的 `file_contexts` 里逐字可查） | 拒绝 | `eacces_untagged` | 成因是**未点名落兜底**：要先补 `file_contexts` 标签再补授权。GKI 实际生成的名字是 `rpmsg%d`（`rpmsg_char.c:459`）与 `rpmsg_ctrl%d`（`rpmsg_ctrl.c:168`），**不在** `file_contexts:153-154` 那两行覆盖的历史名集合里；accel 节点由 `register_chrdev(ACCEL_MAJOR, "accel", …)` 与 `MKDEV(ACCEL_MAJOR, index)` 生成（`drm_accel.c:217`、`:117`），本树 `file_contexts` 检索 `accel` 0 命中。类型侧只有 `type rpmsg_device, dev_type;`（`device.te:75-77`）这一句"给了类型、没给答案"。**禁止**登记"可以打开并 ioctl" |
| **P-21** | `ls /sys/kernel/debug`、`ls /sys/kernel/debug/accel`、`stat -c '%a' /sys/kernel/debug/<某项>` | debugfs 面的开关不在类型上而在构建变量上：`no_debugfs_restriction`（`te_macros:591`，`:590` 的注释句为 `# SELinux rules which apply to devices that do not have debugfs restrictions in non-user builds.`）与 `userdebug_or_eng`（`te_macros:614`）；shell 的四条 debugfs 授权（`shell.te:7-11`）只涉 tracing，`:13-15` 那条 `debugfs_tracing_debug:file rw_file_perms;` 被 `userdebug_or_eng` 包住 | 三态：可达 / 空目录 / 拒绝 | `debugfs_unreachable` / `vendor_dependent` | **先看构建再看厂商**：`getprop ro.build.type` 为 `user` 时连 tracing 那条都没有；采集器的整块 debugfs 读也被包在 `no_debugfs_restriction` 里（`dumpstate.te:290-292`）→ bugreport 不含 debugfs 内容是构建事实。框架侧确实创建了根：`accel_debugfs_root = debugfs_create_dir("accel", NULL);`（`drm_accel.c:215`）。禁止把"debugfs 读不到"直接写成"厂商锁了" |
| **P-22** | `find /sys/kernel/debug -maxdepth 3 -name fw_name`、`cat <命中的 fw_name>` | 目标项：`fw_name_show()` 里 `seq_printf(s, "%s\n", vdev->fw->name);`（`ivpu_debugfs.c:30-36`），与 `bo_list`、`fw_trace_capability`、`fw_trace_config`、`last_bootmode`、`reset_counter`、`reset_pending` 同在 `vdev_debugfs_list[]`（`ivpu_debugfs.c:107-115`） | 取决于 P-21 的结果（可达性由 `te_macros:591` 的构建变量决定，与 P-21 同一锚点） | `vendor_dependent` + 实际错误 | **这是本报告七条链里唯一一个"运行期可读的固件名"出口**，而它在 user 构建上默认不成立。姿势必须是按名字查找而不是猜路径——脚本不写完整路径假设。同目录下的 `force_recovery`、`reset_engine`、`fw_trace_destination_mask`、`fw_trace_hw_comp_mask`、`fw_trace_level` 是 `0200`、`fw_log` 是 `0644`（`:273-293`），本文只登记模式 |
| **P-23** | `ls /dev/ion`、`stat -c '%C %a' /dev/ion` | `file_contexts:137`（`/dev/ion  u:object_r:ion_device:s0`）+ `shell.te:463`（`allow shell ion_device:chr_file rw_file_perms;`） | **可打开** | `openable`（唯一一条 AOSP 明文允许的媒体相关字符设备） | 这条存在的价值是对照：**同一族 /dev 节点，被点名的就有授权，未被点名的就没有**。它不是本域固件的读数面（拿不到任何固件内容），登记为"能力存在"即可。禁止据此推断其他 `/dev/*` 也能开 |

### 二十之六 组 E｜日志、行为与服务（V6/V7/V8，P-24 至 P-26）

| ID | 姿势 | 管住它的规则 | AOSP 预期 | 登记值 | 处置 / 禁止推出 |
| --- | --- | --- | --- | --- | --- |
| **P-24** | `dmesg \| grep -iE 'mfc\|venus\|gmu\|ipu3\|atomisp\|fimc\|ivpu\|qaic\|remoteproc\|adsp\|cdsp'` | `app.te:660-661`（`neverallow appdomain kernel:system { syslog_read syslog_mod syslog_console };`）；`dumpstate.te:209-211` 把 `syslog_read` 只给 dumpstate；本次取回的 `shell.te` 检索 `kernel:system` **0 命中** | 拒绝 | `denied_log`（要记失败码） | 全部 ISP/NPU 版本打印都落在这条面上（`ipu3-css-fw.c:136-138`、`fimc-is.c:417-418`、`ivpu_fw.c:168-169`、`sh_css_firmware.c:255-259`）。**替代姿势只有一条正规通路：抓 bugreport**（`/bugreport` 走 `dumpstate.te:211` 的授权）。另注 `dmesg_restrict`/`kptr_restrict` 在本树被点名成 `proc_security`（`genfs_contexts:53`、`:57`）但取值由厂商 init 决定，本次未取回，列为第卅章边界项 |
| **P-25** | `service list \| grep -iE 'camera\|codec\|media\|gpu\|neural'`、`lshal \| grep -iE …` | `shell.te:358`（`allow shell servicemanager:service_manager list;`）+ `shell.te:386`（`hwservicemanager:hwservice_manager list`）；AOSP 侧名字锚点：`service_contexts:1`（`fwk_camera_service`）、`:47-48`（`ICameraProvider/internal/0`、`/virtual/0`）、`:67-70`（`IComponentStore/{default,default1,default2,software}`）、`:291`（`gpu → gpu_service`） | **列举成功** | `device_read`（声明层） | V3 与 V8 在这里合流：能列出注册了什么。**注意本树 `service_contexts` 检索 `neuralnetwork`/`npu` 0 命中**，所以 NPU 的服务名映射来自厂商侧那份文件，AOSP 只给了类型（`service.te:343`）。`getprop` 里 `npu` 是 `input` 的子串，别用子串匹配做结论 |
| **P-26** | `dumpsys media.codec`、`dumpsys media.player`、`dumpsys SurfaceFlinger`、`dumpsys cameraserver`、`dumpsys gpu` | 需要 shell 与服务所在域之间的 `binder_call`；本次取回的 shell 策略里这样的调用只有六条（脚本 1 用它们的行跨度 `shell.te:29-219` 作本姿势的锚点）：`shell.te:29` storaged、`shell.te:56` statsd、`shell.te:79` gpuservice、`shell.te:102` lpdumpd、`shell.te:214` hal_keymint、`shell.te:219` virtualizationservice。媒体与相机 HAL 域**一条都不在**。另有 `:362-380` 的 find 白名单与 `:510-515` 的四条 neverallow | `dumpsys gpu` 有 AOSP 通路；其余 `vendor_dependent` | `vendor_dependent` + 实际错误 | 唯一能写成"AOSP 给了这条路"的是 GPU（`shell.te:79` + `service_contexts:291`）。**在 AOSP 层，GPU 是七条链里唯一一个 shell 既能点名服务又能 binder 调用的加速器**，媒体与 NPU 两者都不满足。禁止把"厂商设备上 dumpsys 能出东西"记成 AOSP 允许 |

### 二十之七 同一种失败的五种成因，与一条鉴别规则

表里被判为策略拒绝的行有 P-01、P-04、P-14、P-16、P-18、P-19、P-20、P-21、P-24 九条，但它们不是同一件事。鉴别只需要多跑 P-05 一次：

| 成因 | 规则组合 | 判别读数 | 处置动作（谁的活） |
| --- | --- | --- | --- |
| 有类型、无授权 | 类型已在 `file_contexts` 点名（`:221`），`shell.te` 里检索该类型为 0 | `ls -Zd` 出 `u:object_r:video_device:s0` | 厂商补一条 shell 授权；无厂商补丁时该面永久不可得 |
| 未点名、落兜底 | `file_contexts:97`（`/dev`）、`genfs_contexts:123-124`（`sysfs`）、`:423`（`vendor_file`） | `ls -Zd` 出 `device`/`sysfs`/`vendor_file` | 要先补标签再补授权，两步缺一不可，所以厂商"没补"的概率结构性更高 |
| 未打标 | 本树 `file_contexts` 无 `firmware` 命中、`private/vendor_file_contexts` 在 `main` 不存在、`vendor/file_contexts` 212 行内 `firmware` 0 命中；`unlabeled` 类型在 `domain.te:616-617` 被承认（且只禁 create） | `ls -Zd` 出 `unlabeled` | 属厂商分区打表问题，采集侧只登记，不猜 |
| **静默拒绝**（不留日志） | `dontaudit dumpstate sysfs:file r_file_perms;`（`dumpstate.te:286-287`，注释 `# Ignore other file access under /sys.`） | bugreport 缺字段 **且** avc 日志里查不到对应拒绝；当场 P-14 同样失败 | 这是设计行为。第廿六章 K2/K3 的分母必须含不可读项，否则厂商会被这个静音机制误判成覆盖率低 |
| 构建变量整面关闭 | `userdebug_or_eng`（`te_macros:614`）、`no_debugfs_restriction`（`:591`）；`dumpstate.te:290-292` 的 debugfs 读块 | `getprop ro.build.type` = `user` 且 debugfs 根不可达 | 记 `muted_by_build`，不是厂商锁。这类记录不能进"厂商收紧"的统计 |

三条附带判别姿势已经嵌在表里：`stat` 成功而 `cat` 失败 ⇒ 前两种成因；`stat` 与 `ls -Z` 都失败 ⇒ 目录侧没有 search，回到 P-01；`ls` 能列但项数与源码基线不符 ⇒ 优先怀疑厂商改名而不是器件缺位（L-5）。

### 二十之八 六条禁止推出的结论

这张表每行都写了"禁止推出"，这里汇总成正面清单，脚本 1 的输出末尾要原样打印这六条，防止读台账的人自行补全：

1. **不可读 ≠ 不存在**。`unlistable`、`denied_value`、`denied_log` 三种登记值都不构成"该器件无此属性/日志/目录"的证据；本域七条链的器件全部真实存在，只是七个面里有三个制度性静音（十九之十二）。
2. **在场 ≠ 正确版本**。P-02 的 `present` 只证明文件名命中；装载证据要 P-13 的父链 + 服务注册（P-25），版本证据在 AOSP 层无处可取。
3. **行为成功 ≠ 版本合规**。atomisp 的闸门是警告型（`sh_css_firmware.c:208-209` 的 `/* For now, let's just accept a wrong version, even if wrong */` + `return false;`），QAIC 的校验开关由外设决定（`qaic_control.c:1385-1391`）。这两档上 V7 只能给在场性，登记 `not_inferable`。
4. **声明 ≠ 在场证明**。P-09/P-10/P-25 给的都是声明；HAL 接口版本（`manifest.xml` 里的 version）不是固件版本，抄错这一对是公开材料里最常见的误传来源（第廿三章）。
5. **静态基线 ≠ 设备读数**。所有候选名（`IMGU_FW_NAME` 两条、`shisp_*` 三条、`FIMC_IS_FW_FILENAME` 与 setfile 两条、`fw_names[]` 五条、`"cdsp.mdt"`、`"scp.img"`、`"vpu_p.bin"`）全部来自源码，标 `static_baseline`；把它们写成 `device_read` 就是把源码冒充成这台机器。
6. **读不到开关状态 ≠ 开关没被用**。P-16 与 P-17 只能给出存在与模式。任何"该写侧旋钮未被启用"的表述在本域都没有读数支撑，属报告缺陷。

### 二十之九 与脚本 1 的契约

第廿七章脚本 1（只读固件姿态快照）的输入就是这张表，输出是一行一条的竖线分隔记录，字段顺序固定为：

```
pid|face|target|verdict|label|err|rule|source
```

四条硬约束，第廿九章的分支覆盖验收逐条核：

- **每个 P-id 必须在脚本里有且只有一个分支**，`pid` 列原样回填表里的编号；表有 26 行，脚本必须有 26 个可触发分支，**从不触发的分支算脚本的缺陷，不算样例的遗漏**。
- `verdict` 的取值只能是 `present`、`absent`、`unlistable`、`eacces_typed`、`eacces_untagged`、`denied_value`、`denied_log`、`filtered_by_policy`、`openable`、`bound`、`debugfs_unreachable`、`muted_by_build`、`degraded_by_vendor`、`vendor_dependent`、`not_inferable`、`label_unreadable` 这十六个之一；不接受自由文本，不接受布尔。
- `rule` 列必须是本表"管住它的规则"里的一条 `文件:行号`，与台账第卅章的取回记录一一对得上；对不上的记录视为伪造。
- `source` 只能是 `device_read` 或 `static_baseline`。`getprop` 缺席另用 `filtered_by_policy` 占 `verdict`，`source` 仍记 `device_read`（读到的确实是设备给的策略过滤视图）。

P-04、P-18、P-22 三行的输出必须带实际错误串（不翻译、不改写），因为这三处正是厂商差异最大的地方；把它们规范化成统一文案会丢掉区分厂商的唯一线索。

### 二十之十 本章结论

1. **本域的自证上限由这张表给出**：26 个只读姿势里，能拿到"固件版本"的姿势数为 **0**；能稳定拿到"器件在场 + 拓扑 + 名字"的姿势有 5 条（P-12、P-13、P-15、P-17、P-25），能拿到"文件名命中"的姿势 1 条（P-02，且依赖静态基线）。
2. **`eacces` 不是一个结论而是一个待分解的信号**，分解它只需要 P-05 一条额外姿势；不做这一步，第廿六章的所有覆盖率指标都会把五种成因混成一个数。
3. **两种字符设备拒绝的处置成本不同**：有类型无授权是一条规则的距离，未点名落兜底是两步（标签 + 授权）。这一差别应该出现在给厂商的整改建议里，而不是只出现在采集日志里。
4. **三条静音机制里只有两条能被当场识别**：`dontaudit`（`dumpstate.te:287`）不留日志，只能靠"bugreport 缺字段 + 当场姿势同样失败"这一对来识别；构建变量与厂商收紧则能靠 `ro.build.type` 与 debugfs 可达性分开。
5. **脚本 1 的十六个登记值就是本报告的可观测性词汇表**。第廿一至廿六章讨论缺陷类型、判定模型与指标时，凡涉及设备侧证据都必须落回这十六个值之一；落不回的结论就不该声称有设备侧支撑。
## 廿一、缺陷类型学 FT1–FT10：每一类成立所需的前提，以及它对采集侧提出的要求

第四篇（应用安全弱点分类）给出的是应用层弱点类型；本章给出的是**镜像域**的缺陷类型。两者的分类轴不同：应用弱点按"谁在什么阶段做错了什么"分，镜像域缺陷必须按**"缺陷落在哪一层（F1–F6）＋触发它需要哪一种输入"**分，因为同一句"固件里有越界写"在签名固件、`request_firmware` 文件系统装载、`remoteproc` 代载三条路径上，可达性与后果完全不同（第三章的分型）。

**视角声明**：本章不判定任何一颗 CVE 属于哪一类。类型学只描述"这类缺陷在本域的成立条件与识别方法"；把公开案例逐条归到类型上是第廿四章的工作，且只用公告正文已给出组件与严重度的条目。凡本章引用的源码行号，都已在第十至廿章出现过并登记在第卅章取回记录里；本章不引入新来源。

### 廿一之一 三条构造约束

**约束一：每个类型必须自带"成立前提"与"反证条件"。** 没有前提的类型是标签，不是分析工具。本章每条 FT 都给三段：前提（缺它就不成立）、反证（看到什么就必须撤回这条归类）、台账字段（第廿五章矩阵与第廿七、廿八章脚本要新增哪一列）。

**约束二：类型不等于症状。** "旧镜像仍在用"是症状，它可能由 FT3（闸门缺失）、FT4（比较语义错配）或 FT9（归属口径错位）任一种产生，三者的处置方完全不同——分别落在驱动维护者、发布流程与安全负责人身上。台账若只记症状，工单会派错人。

**约束三：本域十类里有三类是"关于可观测性本身"的缺陷**（FT5、FT8、FT10）。它们不会出现在任何公告的 Subcomponent 里，因为公告描述的是器件缺陷而不是取证缺陷；但第十九章的八面读数结论——七个链没有一面能给出固件版本号——说明这三类恰恰是本域运营损耗最大的一类。

### 廿一之二 FT1｜镜像解析器边界缺陷

**定义**：主机侧内核代码在把镜像交给器件之前，要解析它的头部、段落表或尾部描述区。缺陷发生在**"信任长度字段"与"使用长度字段"之间**：解析器从镜像里读出一个计数或偏移，并在完成边界校验之前就用它索引内存。

**本域实例（三处，形态各异）**：

- **IPU3**：`css->fwp = (struct imgu_fw_header *)css->fw->data;` 之后立刻是两道校验——`if (css->fw->size < struct_size(css->fwp, binary_header, 1) || css->fwp->file_header.h_size != sizeof(struct imgu_fw_bi_file_h)) goto bad_fw;`（`ipu3-css-fw.c:129-131`），再检查 `struct_size(..., binary_nr) > css->fw->size`（`:132-134`）。随后逐 binary 的校验链覆盖 `prog_name_offset`、`strnlen`、`size == text + icache + data + padding`、`offset + size > fw->size`、`iomem_length`、`IMGU_ABI_FRAME_FORMAT_NUM`、`BLOCK_MAX`（`:148-224`），任一不成立即 `goto bad_fw`。**这一串是本域写得最完整的镜像解析防御样本**：长度字段全部参与校验，且校验顺序是"先确认字段本身在界内，再用它"。
- **Intel VPU（ivpu）**：`ivpu_fw_parse()` 八条硬校验（`ivpu_fw.c:112-164`）：文件过小（`:118`）、`header_version != VPU_FW_HEADER_VERSION`（`:123`）、runtime 地址区间（`:133`）、`runtime_size < fw->file->size`（`:138`）、镜像越界（`:143`）、加载区间（`:148`）、`shave_nn_fw_size > FW_SHAVE_NN_MAX_SIZE`（`:155`）、`entry_point` 区间（`:160`），每条 `ivpu_err` + `-EINVAL`。
- **FIMC-IS**：解析形态是第三种——**从镜像尾部倒着读**。`if (fw->size < FIMC_IS_FW_SIZE_MIN || fw->size > FIMC_IS_FW_SIZE_MAX)`（`fimc-is.c:390`）之后，`buf = (void *)(is->memory.vaddr + fw->size - FIMC_IS_FW_DESC_LEN);` 取 31 字节 info，再以 `fw->size - FIMC_IS_FW_VER_LEN` 取 7 字节 version（`:407-413`）。定长常量在 `fimc-is.h:47-51`，区间常量在 `:60-61`（`SZ_4M` / `SZ_32K`）。这里没有"字段自述长度"，因此缺陷面转移到"镜像小于描述区"，而它已被 `:390` 的 32KB 下限挡住。

**成立前提**：①解析发生在主机内核地址空间；②镜像内容对写入者可控（能替换 `/vendor/firmware/**` 或挂载点）；③长度/计数字段被用于内存索引或 DMA 区间，而校验在其后或不存在。

**反证条件**：若镜像在主机解析之前已由硬件侧完成认证——高通侧的 `qcom_scm_pas_auth_and_reset(adsp->pas_id)`（`qcom_q6v5_pas.c:301`，dtb 分支 `:280`）——则主机解析器缺陷的后果上限是"拒绝启动"，不应记为可利用。若解析器根本不在主机（`qaic` 全树检索 `request_firmware`、`MODULE_FIRMWARE`、`.bin` **0 命中**，`firmware` 一词只在 `qaic_drv.c:272-281` 那段注释里出现两次，而注释说的正是"主机要等设备侧固件自己起来"，NPU 镜像不驻留主机），FT1 在本链不适用，记 `not_inferable`。

**台账字段**：`parse_side`（host / device / none）、`bounds_prechecked`（yes / no / warn）、`integrity_gate_before_parse`（pas / signature / none）。

### 廿一之三 FT2｜装载决策受非授权输入影响

**定义**："加载哪一个镜像"这个决定，由一个**不受更新机制保护、也不被安全策略约束**的输入参与选择。它不是越界，而是决策面被暴露——攻击者或运维误配都能改变装载对象。

**本域实例（六种选择姿态；其中两类输入源在运行期可写——模块参数与框架属性写侧）**：

| 姿态 | 原句与位置 | 谁能改变结果 |
| --- | --- | --- |
| 固定单名 | `#define FIMC_IS_FW_FILENAME "exynos4_fimc_is_fw.bin"`（`fimc-is.h:32`） | 只有文件内容 |
| 日期长名优先、`-ENOENT` 回落 | `ipu3-css-fw.c:120-122` | 盘上文件存在性 |
| 按硬件 Revision 三分支 | `atomisp_v4l2.c:1143`/`:1148`/`:1153`，无匹配 `"Unsupported hw_revision 0x%x\n"`（`:1157`） | 硬件本身 |
| 代号回落表 + 模块参数优先 | `ivpu_fw.c:46-56` 六条 `fw_names[]`；`:67-71` 模块参数优先；参数声明 `module_param_named_unsafe(firmware, ivpu_firmware, charp, 0644)`（`:42-44`） | **可写模块参数** |
| 设备树属性 | `rproc_of_parse_firmware(dev, 0, &fw_name)`（`mtk_scp.c:849`），实读 `"firmware-name"`（`remoteproc_core.c:1002`），默认值 `"scp.img"`（`:846`） | 镜像/DT 分区 |
| 框架属性代答 | `remoteproc_sysfs.c:167` 的 `firmware` 是 `DEVICE_ATTR_RW`（登记为能力事实，不给用法） | 有写权限的进程 |

另外两条例外值得单记：`skip_fwload` 是 `module_param(..., uint, 0644)`（`atomisp_v4l2.c:58-60`）——一个可写开关能把整条装载链关掉；`firmware_name` 是 `module_param_string(..., 0)`（`:77-79`）——同名机制、权限 0，两者构成"同类输入、不同暴露"的对照。MFC 的降序尝试（`s5p_mfc_ctrl.c:63-72`，`MFC_FW_MAX_VERSIONS` 为 2）则是**默认偏好新文件而非安全校验**，盘上同时存在两个候选名时装载对象由目录内容决定。

**成立前提**：①输入源（模块参数 / 设备树 / sysfs 写侧 / 文件系统命名）位于非特权可写域；②没有下游完整性校验能识别"名字对但内容换"；③该选择发生在每次装载（而非仅首次）。

**反证条件**：第二十章 P-12 至 P-18 给出，`/sys/module/**` 只有五条被 `genfs_contexts:182-186` 点名，其余继承默认；`shell.te` 内 `sysfs:file` 授权 0 条（`:443` 仅给 `sysfs:dir`）。**若 P-16 的 stat 显示参数模式已是 0444，且同一条面上 shell 域对该文件没有任何写授权，这条输入就不构成攻击面，只构成配置漂移风险**——此时 `input_writable_by_unpriv` 记 `no`，verdict 仍用 `present`（节点在）。注意不要用 `bound` 顶替：第二十章里 `bound` 只表示"驱动已绑定"那一个含义。同理，若分区只读且设备未解锁，文件系统命名类应把可达性记 `absent`。

**台账字段**：`name_source`、`input_writable_by_unpriv`（stat 模式 + 标签，两者都要）、`override_exists`（yes / no / unknown）。

### 廿一之四 FT3｜版本闸门缺失，或闸门只警告

**定义**：驱动知道镜像版本，但不据此拒绝装载；或知道不一致，只打印。**这类缺陷的实质是"补丁状态对装载决策没有影响"**，它是旧镜像长期驻留的代码侧原因。

**本域实例（三种闸门姿态，含两个反例）**：

- **警告型，且结构上不可能中止装载**：`sh_css_check_firmware_version()` 的返回类型是 `bool`（`sh_css_firmware.c:188-189`）。版本不相等时它只打两条 `dev_err`——`"Firmware version may not be compatible with this driver\n"`、`"Expecting version '%s', but firmware is '%s'.\n"`（`:202-206`）——然后**无条件落到** `/* For now, let's just accept a wrong version, even if wrong */`（`:208`）与 `return false;`（`:209`）。而两个调用点都按"真值即失败"判断：`err = sh_css_check_firmware_version(...); if (err) { dev_dbg(...); goto fw_validation_fail; }`（`atomisp_v4l2.c:1398-1402`）、`ret = sh_css_check_firmware_version(...); if (ret) { IA_CSS_ERROR(...); return -EINVAL; }`（`sh_css_firmware.c:253-258`）。**函数恒返回 `false`，因此这两处检查永远不会走失败分支**——那两条 `dev_err` 就是版本错配的全部后果。这是本域警告型闸门的最强形态，也是第十九章 V7 那一格降级到 `△` 的确切原因。
- **半硬型**：`ivpu_fw_check_api()` 只在 major 低于下限时返回 `-EINVAL`（`:97-101`），major 不同但更高时仅 `ivpu_warn`（`:102-105`）。
- **无闸门**：FIMC-IS 全链只有尺寸区间（`:390`）与两段超时等待（`fimc-is.c:671-686`），版本只用于打印（`:417-418`）。
- **反例甲（fail-closed）**：`a6xx_ucode_check_version()` 对未知 GPU 打 `"unknown GPU, add it to a6xx_ucode_check_version()!!\n"` 后落到 `return ret;`，`ret` 初值为假（`a6xx_gpu.c:772-773`、`:777`），拒绝路径给 `-EPERM`（`:800-806`）。
- **反例乙（协议硬闸门）**：`qaic_drv.c:296-302` 的 `if (ret || major != CNTL_MAJOR || minor > CNTL_MINOR)` → `pci_err` + `-EINVAL`，常量是 `CNTL_MAJOR 5` / `CNTL_MINOR 0`（`:35-36`）。

**成立前提**：①存在一条外部认定的"低于此版本不安全"边界（公告或厂商警报给出）；②驱动有能力读到版本（FT10 会让这条先失败）；③闸门即使存在也只影响日志，不影响装载结果。

**反证条件**：**没有公告版本边界时不能记 FT3**——"不校验"在缺少期望值时无法评价。`ivpu` 的 major/minor 语义写在 `vpu_boot_api.h:10-39`（`Major version changes that break backward compatibility.` / `Minor version changes when API backward compatibility is preserved.`），据此它的"minor 只警告"是**符合自身文档语义的设计**，不是缺陷。反过来说，`a6xx` 的 fail-closed 使它成为 FT3 的阴性对照：`a6xx_ucode_check_version()` 里 `a660 targets have all the critical security fixes from the start`（`:737-738`）加无条件 `ret = true;`（`:769-770`）才是本类该记的位置。

**台账字段**：`gate_kind`（hard / warn / none）、`gate_source_verifiable`（yes / no）、`expected_floor_published`（yes / no）。第三项为否时，前两列只登记事实，不做评价。

### 廿一之五 FT4｜版本比较语义错配

**定义**：闸门或人以"版本号可比"为前提做判断，但两侧对同一个字段的理解不同：一段整数在不同镜像里分别是语义版本、构建哈希、日期串或数组下标。

**本域实例（四处错配，形态互不相同）**：

- **整数被当语义版本**：Adreno a6xx 从镜像自身取四个 `u32`（`u32 *buf = msm_gem_get_vaddr(obj);`，`a6xx_gpu.c:719`），再按位段比较——a630 是 `(((buf[0] & 0xf) == 0xa) && (buf[2] & 0xf) >= 1) || (buf[0] & 0xfff) >= 0x190`（`:750-751`），a650 是 `(buf[0] & 0xfff) >= 0x095`（`:761`），A7xx 直接放行（`:725-727`）。`0xa` 这个标记位另有注释解释其补丁含义（`:740-744`）。**这套判断完全依赖厂商内部编号规则，外部无法复算。**
- **哈希被解析成 `1.0.0`**：Venus 的三条硬编码前缀解析（`hfi_msgs.c:304-330`）遇到 `"1.0.<hash>"` 得到 `1.0.0`，注释逐字承认了后果：`SC7280 also reports "1.0.<hash>" parsed as 1.0.0; restricting to IRIS2` / `avoids misapplying this quirk and breaking VP9 decode on SC7280.`（`vdec.c:571-575`）。
- **下标被当版本**：MFC 的 `dev->fw_ver = (enum s5p_mfc_fw_ver) i;`（`s5p_mfc_ctrl.c:69`）记录的是赢的那个文件名的数组下标，枚举只有两个值（`s5p_mfc_common.h:165-168`），字段注释是 `loaded firmware sub-version`（`:298`）。
- **自报串被当镜像属性**：Panther 的接口版本是固件运行时写入共享内存的字段（`panthor_fw.h:193-202`），编码为 `(((major) << 24) | ((minor) << 16) | (patch))`（`panthor_fw.c:192-196`）；`ipu3-css-fw.c:136-138` 的 `loaded firmware version %.64s` 与 `ivpu_fw.c:168-169` 的 `Firmware: %s, version: %s` 同样把镜像内的自由文本直接当成版本。

**成立前提**：①存在跨期或跨厂商的比较动作（闸门、公告对齐、资产聚合都算）；②字段编码规则发生过变更且变更不体现在字段本身；③没有权威对照表。第三条在本域是**已证事实**：第卅章边界清单里"公告词与内核对象之间无权威对照表"这条。

**反证条件**：若比较只在同一命名规则、同一厂商、同一产品线内发生，FT4 不成立。若读数本身来自文件名字符串而非镜像字段（如 MFC 的八条变体名表 `s5p_mfc.c:1536-1687`），它连"版本"都不是，只能记 `present`/`absent`。

**台账字段**：`version_semantics`（semantic / build-hash / date / index / free-text / unknown）、`comparable_within_vendor`、`cross_vendor_compare`（本域一律禁止，第廿六章）。

### 廿一之六 FT5｜静默降级

**定义**：驱动根据固件版本或能力声明**关掉一个功能**，而不产生任何日志、属性或状态变化。后果是"设备功能少了"，而台账上找不到少的原因；更糟的是，安全相关的缓解有时也挂在同一个能力位上。

**本域实例**：

- `panthor_get_instr_features()`（`panthor_fw.c:878-886`）第一件事是 `if (glb_iface->control->version < CSF_IFACE_VERSION(1, 1, 0)) return 0;`。这是**能力协商而非版本闸门**——它决定的是仪表功能有无，且返回 0 时没有任何打印。上游 `v7.3` 里同一形态长成三处探测，同样不带日志：`return glb_iface->control->version >= CSF_IFACE_VERSION(4, 1, 0);`（`panthor_fw_has_glb_state`，`:326-331`）与 `CSF_IFACE_VERSION(4, 0, 0)`（`panthor_fw_has_64bit_ep_req`，`:333-338`）。
- Venus 的两个门都是静默的行为分支：`if (IS_IRIS2(inst->core) && is_fw_rev_or_older(inst->core, 1, 0, 87))`（`vdec.c:577`）与 `if (is_fw_rev_or_newer(inst->core, 5, 4, 51))`（`vdec.c:711`，上一行注释 `/* Enabling sufficient sequence change support for VP9 */`）。**两处全是 VP9/EOS 行为微调，与安全补丁级别无关**——这一点必须写进台账，否则运营会把"解码行为变了"误读成"补丁状态变了"。
- `skip_fwload`（`atomisp_v4l2.c:58-60`）是最彻底的一档：模块参数置位后整条装载被跳过，而装载链本身不留失败记录。

**成立前提**：①功能开关依赖固件字段；②该字段无读数出口（FT10）；③降级路径不打印。**三条缺一则本类退化为可诊断问题，不再是类型学条目。**

**反证条件**：有 `dev_info`/`dev_err` 的降级不算静默（`mtk_vpu.c:628` 的 `vpu is ready. Fw version %s\n` 是一行 INFO，属可读不可读的问题，不属本类）；被 FT3 拦成装载失败的情况属 FT3，不重复计入本类。

**台账字段**：`silent_capability_change`（yes / no）、`detectable_by_feature_probe`（yes / no）；采集侧对应第廿九章的"功能反证覆盖"指标。

### 廿一之七 FT6｜跨核通道输入直接变成主机侧状态

**定义**：对端（协处理器、外置 NPU）通过名称服务、状态报文或握手回复写进主机字符串或分支选择；这些字符串随后出现在设备模型、模态匹配或日志里，被采集侧当作"设备事实"读回。

**本域实例**：

- **联发科 SCP 名称服务**：`struct rpmsg_ns_msg` 由远端填写（`mtk_rpmsg.c:45-48`，注释 ` * @name: name of remote service that is published` 在 `:37`），专用端点绑到 `SCP_IPI_NS_SERVICE`（值为 `0xFF`，`include/linux/remoteproc/mtk_scp.h:47`；端点创建 `mtk_rpmsg.c:313-318`）。处理函数 `mtk_rpmsg_ns_cb()`（`:270` 起）做三道检查，**每一道都是对"对面是不可信实体"的承认**（`:279-283`、`:284-292`、`:295-296`），检查通过后仍会把 `msg->name` 交给 `mtk_rpmsg_create_device()`（`:298-300`）→ `strscpy(info->info.name, name, RPMSG_NAME_SIZE)`（`:259`）→ `strscpy(rpdev->id.name, info->name, RPMSG_NAME_SIZE)`（`:214`）→ 目录命名（`rpmsg_core.c:620-621`）。
- **宽度截断**：`#define RPMSG_NAME_SIZE 32`（`mod_devicetable.h:465`）加 `struct rpmsg_channel_info { char name[RPMSG_NAME_SIZE]; }`（`include/linux/rpmsg.h:34-35`）。固件写 40 字符，目录里就是截断的 31 字符；截断后与驱动 `id_table` 全称不匹配，设备以"无驱动绑定"出现（第十八章之四）。
- **用户空间同一条路径**：`driver_override` 是该总线唯一可写属性（`rpmsg_core.c:425-427`、`DEVICE_ATTR_RW` 在 `:449`），带 override 的注册路径把传入串直接写进 `rpdev->id.name`（`:617-618`），而用户态可以用 ioctl 交一个 32 字节的名字（`rpmsg_ctrl.c:84-88`）。**同一段目录名，可能来自固件、内核驱动或有权限的用户空间。**
- **高通 NPU 把决定权交给外设**：`get_cntl_version()` 读回的除了 `*major`/`*minor`（`qaic_control.c:1382-1383`），还有 `status_flags` 的位 0——`if (status_result->status_flags & BIT(0)) { ... qdev->valid_crc = valid_crc; } else { ... qdev->gen_crc = gen_crc_stub; }`（`:1385-1391`）。**设备侧选择让主机跑真 CRC 还是桩函数。**

**成立前提**：①主机使用远端串参与命名、匹配或路径选择；②该串进入采集面（sysfs 目录名、`modalias`、日志）；③没有伴读手段能区分来源。

**反证条件**：若字符串只用于展示且不参与任何 `id_table` 匹配或分支，后果限于取证污染，不构成主机侧状态被写入。`qaic` 那条 CRC 分支改变的是主机的校验行为，不是主机内存安全边界，记本类时要写明影响面。

**台账字段**：`value_source`（`fw_reported` / `driver_set` / `userspace_override` / `unknown`）——第十八章承诺的强制标签，凡 V4 的 rpmsg 名字读数必须伴读同目录 `driver_override` 当前值。

### 廿一之八 FT7｜能力位由版本决定，而安全属性由能力位决定

**定义**：链条是"固件版本 → 主机看到的能力位 → 主机是否启用某条路径"。缺陷发生在中间那一跳：能力位是设备自报的，主机把它当作可信开关。

**本域实例**：Panther 的 `struct panthor_fw_global_control_iface` 里 `u32 features` 与 `u32 instr_features` 由固件运行时写入（`panthor_fw.h:193-202`），驱动打印时使用（`panthor_fw.c:924-929`）。IVPU 把这条做成显式接口协商：`IVPU_FW_CHECK_API(vdev, fw_hdr, BOOT, 3)` 与 `JSM, 3`（`ivpu_fw.c:171-174`），版本从 `fw_hdr->api_version[index]` 拆出 major/minor（`:94-95`）。Venus 用版本推能力（`vdec.c:711`），MFC 用数组下标推寄存器写法（`#define IS_MFCV6_V2(dev) (!IS_MFCV7_PLUS(dev) && dev->fw_ver == MFC_FW_V2)`，`s5p_mfc_opr_v6.c:43`，两个使用点都只决定往哪个寄存器写）。

**成立前提**：①能力位来源是设备侧声明而非主机侧探测；②主机的某条代码路径以该位为唯一开关；③声明与实现可以不一致而主机无从发现。

**反证条件**：若声明只影响性能或统计功能（如 Panther 的 instr features 只关仪表），不能记为"安全属性由能力位决定"；这一档应记 FT5 的实例位而非本类。**本域公开来源里没有任何一条说明某个能力位控制安全缓解**，因此第廿六章的判定模型禁止把能力位读数升到 L2 以上。

**台账字段**：`capability_source`（fw-declared / host-probed）、`capability_gates_security`（yes / no / **unknown，本域默认**）。

### 廿一之九 FT8｜取证与转储面自身成为暴露面

**定义**：为了诊断固件缺陷而存在的那些出口（coredump、资源表、日志缓冲、调试属性）本身携带镜像片段、内存布局或可写行为。它们不构成远程攻击面，但构成**信息暴露面**和**误操作面**，而且常常是策略与文档口径不一致的地方。

**本域实例**：

- `remoteproc_debugfs.c:405-418` 六个节点：`name` 0400、`recovery` 0600、`crash` **0200**、`resource_table` 0400、`carveout_memories` 0400、`coredump` 0600；上方还有 `if (!rproc_dbg) return;`（`:398-403`）——**debugfs 未初始化时整棵树静默不存在**。其中 `resource_table` 与 `carveout_memories` 给的是协处理器内存布局与共享区名称，比 sysfs 更实质。
- `coredump` 的自带注释与实现不一致：文档注释块（`remoteproc_sysfs.c:95-110`）里写 `The default value of this entry is "default".` 并列出 `default:` 一节，而实现（`:111-134`）的 `sysfs_streq` 只接受 `disabled`、`enabled`、`inline`，其余落到 `"Invalid coredump configuration\n"` → `-EINVAL`。**注释里的 `default` 不是合法值，`enabled` 这个词一次也没出现在说明中**；同段第三处口径不一致是 `coredump_show()` 上方的 `/* Expose the current coredump configuration via debugfs */`（`:86`），而它是 sysfs 属性（`:135`）。读侧字符串表在 `:80-84`，崩溃态下改配置被拒：`"can't change coredump configuration\n"` → `-EBUSY`（`:117-120`）。
- atomisp 的驱动文件：`__ATTR(dbgfun, 0644, ...)` 与 `__ATTR(dbgopt, 0644, ...)`（`atomisp_drvfs.c:160-161`）经 `driver_create_file()`（`:164-172`）落到 `/sys/bus/pci/drivers/atomisp-isp2/`；`iunit_dump_dbgopt()` 里 `OPTION_BIN_LIST` 会转储 blob 信息（`:62-63`），`OPTION_BIN_RUN` 需要 `isp->asd.streaming` 否则 `ret = -EPERM`（`:71-80`）。
- ivpu 的 `fw_log` 是 0644 且 `fw_log_fops` 同时有 `.write` 与 `.read = seq_read`（`ivpu_debugfs.c:144-148`、`:117-124`），其余 `force_recovery`、`reset_engine`、`fw_trace_destination_mask`、`fw_trace_hw_comp_mask`、`fw_trace_level` 全为 0200（`:273-293`）。**本报告只登记这些节点的存在与模式，不给出任何使用方式。**
- 策略侧的对称动作：`dontaudit dumpstate sysfs:file r_file_perms;`（`dumpstate.te:287`）让 bugreport 读 sysfs 值时**拒绝且不留痕**——取证面被静音，而不是被拒绝记录。

**成立前提**：①出口在采集者可及域内；②内容包含镜像片段、内存布局或版本串；③其访问控制依赖 debugfs 挂载与构建类型，而非白名单。

**反证条件**：`userdebug_or_eng`（`te_macros:614`）与 `no_debugfs_restriction`（`te_macros:591`）能在 user 构建上整面关掉它们。**节点不存在时不能记为"厂商加固"，应记 `muted_by_build`**；把 `0200` 记成"可写=危险"更是错的，写侧能力的登记方式与第十三章 `state_store` 一致。

**台账字段**：`debugface_mode`（stat 八进制值）、`debugface_present`（yes / no / muted_by_build）、`dontaudit_sensitive`（yes / no）。

### 廿一之十 FT9｜归属层与对象层脱钩

**定义**：公告、资产台账与设备侧对象三层用的是三套粒度，缺陷修在 F1（镜像本体）而公告词落在 F5/F6 的口径上，于是"是否已修"在三层各自得出不同答案。这是本域独有的类型——**它不是器件缺陷，而是治理缺陷，且是本报告前六章的主线**。

**本域实例**：

- 一个公告词覆盖两套架构与三个可分别更新的部件：`Mali` 承载整个 GPU IP 家族，不区分 JM 与 CSF，也不区分内核驱动与命令流固件（第五章）；`PowerVR-GPU` 同理，而厂商侧按 DDK 版本给受影响范围，设备侧两者都读不到（第九章）。
- 占位串：`Qualcomm closed-source components` 表 11 行的 Subcomponent 一律是 `Closed-source component`，References 列只有 `A-` 前缀的 Android bug ID，**没有任何 `QC-CR#` 引用号**（第六章）。同一期 `Qualcomm components` 表 5 行的 Subcomponent 全部是 `Security`。两张表之间没有可供外部推导的划分规则。
- 严重度与披露责任同时外移：`The severity assessment of these issues is provided directly by Qualcomm.`（第六章）。
- 策略层的同类脱钩：`hal_camera_service` 在 `public/service.te:311` 带 `protected_service`，而 `hal_neuralnetworks_service`（`:343`）**不带**——服务名字典与器件归属字典是两套互不引用的命名。

**成立前提**：①公告词表与器件对象集合的基数不同（实测 21 个表名、GPU 侧只有 2 个 Subcomponent 词）；②没有权威对照表；③资产聚合必须按公告词进行。

**反证条件**：若某期条目带 `QC-CR#` 或明确型号（如 `MediaTek components` 给出芯片型号），该条可下钻，不记本类。若判定口径明确写成"按公告词覆盖"并声明不确定性，FT9 视为**已接受风险**而非待办缺陷。

**台账字段**：`mapping_class`（A 一对一 / B 一对多 / C 对象不在可枚举分区），第十三章的词表分档即此列。

### 廿一之十一 FT10｜可观测性缺陷

**定义**：判定、验收与取证所需的读数**在设备侧不存在**，或存在但被策略/构建静音，或可读但语义不可比对。本域第十八、十九章已把它钉成结构性结论：七条链 × 八面，没有一个组合能给出可比对的固件版本号。

**本域实例**：

- 只进调试级日志：Venus 的 `venus_ver` 是内核结构体成员（`core.h:233-237`，字段注释 ` * @venus_ver: the venus firmware version` 在 `:173`），采集侧的出口只有 `dev_dbg`（打印在 `hfi_msgs.c:322-323`，android15-6.6；调用点在 `:346`），第十五章另记有一条写入 SMEM 表项的出口（`:323-330`，宏 `SMEM_IMG_VER_TBL`/`VER_STR_SZ`/`SMEM_IMG_OFFSET_VENUS` 在 `:18-20`，该表项在 shell 域不可读）；MFC 从寄存器读到的三位固件版本同样只送到 `mfc_debug(2, ...)`（`s5p_mfc_ctrl.c:281-287`，打印在 `:286-287`），而默认关闭，开关是名为 `debug` 的 0644 模块参数（`s5p_mfc.c:39`）。
- 有值无出口：Adreno 的 `fwloc` 只在三处写入（`a6xx_gpu.c:468`/`:488`/`:509`）、三处与一处读取（`:461`/`:481`/`:502`、`:140`），**没有任何 sysfs 或属性出口**；GMU 版本取回后即丢弃（`a6xx_hfi_get_fw_version()` 以 `NULL` 接收 `a6xx_hfi_msg_fw_version`）。
- 被策略静音：`shell.te:443` 给 `sysfs:dir r_dir_perms` 而 `shell.te` 内 `sysfs:file` 命中 0 条；`dumpstate.te:287` 的 `dontaudit`；`property.te:191-198` 的 neverallow 只放行 `vendor_public_property_type`，使 `getprop` 成为过滤后的视图。
- 被构建关掉：`userdebug_or_eng`（`te_macros:614`）与 `no_debugfs_restriction`（`te_macros:591`）。
- 标签本身读不到：AOSP 的 `file_contexts` 内 `firmware` 标签 0 条，`private/vendor_file_contexts` 在 `main` 分支不存在（404），`vendor/file_contexts`（212 行）内 `firmware` 0 命中；`/dev/video*` 是"有类型无授权"（`device.te:47` 给了类型，`file_contexts:221` 给了标签，`shell.te` 无 `video_device` 授权），`/dev/accel*` 是"未点名兜底"（`file_contexts:97`），两者同一个 EACCES、两种成因。

**成立前提**：①某个运营指标依赖该读数；②读数在至少一个采集面上不可得；③不可得的原因需要区分（策略 / 构建 / 厂商 / 根本不存在）。

**反证条件**：读数存在但语义不可比对属 FT4，不属本类；读数可被伪造属 FT6；只有"拿不到"才是 FT10。**若某链设计上就没有主机侧版本字段（`qaic` 结构上无主机镜像），应记为边界事实，不得记为设备异常或厂商锁机。**

**台账字段**：`unavailable_reason`（`policy` / `build` / `vendor_dependent` / `not_in_tree` / `no_exit`）——第廿七章脚本 1 的每个失败分支必须归入其中一档，这正是第二十章"同一种失败的五种成因"表的目的。

### 廿一之十二 十类 × 七链：实例分布矩阵

记号：**●** 表示本树已核验到该链上的具体行号实例；**○** 表示该类型在这条链上结构性存在（装载路径与输入面相符），但本次未取回可钉的源码行；**–** 表示本域不适用或已给出反证。

| 类型 | Mali CSF | Adreno GMU/KGSL | Exynos MFC | QCOM Venus | MTK VPU+SCP | 相机 ISP | NPU/APU |
| --- | --- | --- | --- | --- | --- | --- | --- |
| FT1 解析器边界 | ○（镜像先过安全侧校验，主机解析点未取回） | ○（GMU 同左） | ○（选择靠文件名，不解析镜像体） | ○（版本串解析 ●，但归入 FT4） | ○（`scp.img` 名由 DT 决定，段落解析在设备侧） | ●（IPU3 校验链、FIMC 尾部定长描述区） | ●（ivpu `ivpu_fw_parse()` 八条硬校验）；qaic –（主机无镜像） |
| FT2 装载决策输入 | ○ | ○ | ●（降序试最新，`MFC_FW_MAX_VERSIONS` 为 2） | ○（本树未取回 venus 侧主机名选择点） | ●（新旧双名回落 `mtk_vpu.c:512`/`:517`；SCP 走 DT `firmware-name`） | ●（`skip_fwload` 0644、日期名回落、Revision 三分支） | ●（`firmware` 模块参数 0644 + 代号回落表） |
| FT3 闸门缺失/警告 | ○ | ●反例（fail-closed） | ●（无闸门，`fw_ver` 不参与拒绝） | ●（闸门只关 VP9/EOS 行为） | –（本树未见版本比较） | ●（atomisp 警告型、FIMC 无闸门） | ●（ivpu major 硬 / minor 警告；qaic 硬但版本来自设备） |
| FT4 比较语义错配 | ●（编码宏与自报字段） | ●（位段当语义版本） | ●（数组下标当版本） | ●（`1.0.<hash>` → `1.0.0`） | ●（自由文本进 INFO） | ●（IPU3 `%.64s`、FIMC `rev. %s`） | ●（`api_version[]` 拆位、VPU 版本串） |
| FT5 静默降级 | ●（instr features 返回 0 且无日志） | ○ | ○（下标只决定寄存器写法，属 FT7） | ●（两个门无日志） | – | ●（`skip_fwload`） | ○ |
| FT6 跨核输入变状态 | ●（`features` 由固件写入共享内存） | ●（版本经 HFI 报文交换后丢弃） | ○ | ●（`venus_ver` 自报） | ●（名称服务三道检查后仍入目录命名） | ○（读的是镜像字段而非通道报文，实例见 FT1） | ●（QAIC 由外设决定主机 CRC 走真函数还是桩） |
| FT7 能力位决定路径 | ● | ○ | ●（`IS_MFCV6_V2` 决定寄存器写法） | ●（`vdec.c:711`） | – | ○ | ●（`IVPU_FW_CHECK_API` BOOT/JSM） |
| FT8 取证面自身暴露 | ○ | ○ | ○（本树未取回转储出口） | ○（同上） | ●（debugfs 六节点 0400/0200/0600、`coredump` 注释与实现不一致） | ●（`dbgfun`/`dbgopt` 0644，BIN_LIST 转储 blob） | ●（`fw_log` 0644 读写、五个 `0200` 写侧节点） |
| FT9 归属与对象脱钩 | ●（`Mali` 一词三部件） | ●（closed-source 占位串） | ●（Samsung 厂商入口 403） | ●（`MediaTek components` 一档覆盖多引擎） | ●（`apusys` 对象不在可枚举分区） | ●（实测 21 个表名里没有 ISP 专词，只能落到厂商组件一档） | ●（`hal_neuralnetworks_service` 不带 `protected_service`） |
| FT10 可观测性缺陷 | ● | ●（`fwloc` 无 sysfs/属性出口） | ●（寄存器版本只到 `mfc_debug`，开关是 `debug` 参数） | ●（唯一出口是 `dev_dbg`） | ●（一行 INFO，日志面被静音即全不可得） | ●（三家版本只进日志） | ●（minor 与 `firmware-name` 都在读不到的一侧） |

三点要从矩阵里读出来：**FT9 与 FT10 是满行**——它们在七条链上都有已核验实例，这解释了为什么本报告把归属与采集放在缺陷之前；**FT1 的强实例集中在相机 ISP 与 NPU 两条链**——因为这两条是本次新取回的、主机侧解析最完整的代码；**○ 的分布不是随机缺口**，它们全部落在"镜像经签名/认证后才入主机"的 GPU 两链上，那正是第十四至十六章给出的装载路径分型的结果，而不是采集没做到位。

### 廿一之十三 本章结论

1. **十类里 FT1–FT7 描述"缺陷怎样发生"，FT8–FT10 描述"我们怎样知道"**。分界线是处置方：前三类的整改对象是公告与资产口径（FT9）与采集能力（FT8、FT10），它们不会出现在任何公告的 Subcomponent 里；把两组混在一张清单上，工单会全部派给固件团队。
2. **每一类都必须连前提一起使用。** FT3 无公告边界时不可评价；FT4 只在跨规则比较时成立；FT5 三条前提缺一就退化为可诊断问题；FT10 与 FT4/FT6 的分界在"拿不到"与"拿到了但不可信"。
3. **本域的强实例集中在 ISP/NPU 与 MTK/Venus 链**，因为只有这些链把镜像解析、能力协商和取证出口全部写在主机侧 GKI 树里；GPU 两链的 ○ 是装载路径的必然结果。
4. **两个反例要长期保留**：`a6xx_ucode_check_version()` 的 fail-closed 与 `qaic` 的协议硬闸门。它们证明"版本参与装载决策"在上游是可实现的写法，因此 FT3 的评价有参照系；也证明硬闸门本身可以建立在**设备自报的版本**之上，这正是 FT6 与 FT3 交叠的位置。
5. **类型到工单的映射是本章的实用产出**：FT1/FT7 → 厂商固件团队；FT2/FT3/FT4 → 驱动维护者与发布流程；FT5/FT8/FT10 → 本机构采集与取证能力；FT9 → 安全负责人与资产口径。第廿五章的归属矩阵按这五个去向组织列。
6. **第廿六章的判定模型不接受任何单类证据升级到 L2 以上**：FT6 使名字类读数可伪造，FT4 使版本串跨期不可比，FT10 使八面读数全部缺席——三条合起来决定了 L 级上限。
## 廿二、读数纪律：八条可执行的规则，加一份案例读数的取样偏差声明

第十九章给出"能读到什么"，第二十章给出"读不到时怎么记"。本章给出的是**第三件事**：当你已经拿到一条读数，怎样才不算把它读错。它之所以要单列成章，是因为本域的分析损耗大部分不发生在校验失败上，而发生在**校验成功但语义错位**上——命中了一个词、打开了一个文件、读到一个字符串，然后把它们分别当成"器件确认""镜像确认""版本确认"。

**引用范围声明**：本章的实测数字（命中数、行号）来自本次已取回的本地副本，登记在第卅章取回记录 D-8；案例侧的原句全部来自第五、九两章已逐字引用的页面，本章不引入新的公告来源。

### 廿二之一 三类口径，以及它们为什么不能直接 join

| 口径 | 载体 | 粒度 | 本域已证的性质 |
| --- | --- | --- | --- |
| **公告侧** | 厂商表的表名 + Subcomponent + CVE + References | 一个词覆盖多部件 | `Mali` 一词 = JM/CSF 两架构 × 三个可分别更新部件（第五章）；`Closed-source component` 是占位串（第六章） |
| **源码侧** | 标识符、字符串常量、结构体字段 | 一个符号 = 一段实现 | `venus_ver` 是内核结构体成员，无出口（第十五章之四）；`apusys` 在 GKI 树里 0 实体（第十六章之四） |
| **设备侧** | sysfs 属性名、目录名、节点名、日志行 | 一次装载的结果 | `firmware` 属性给文件名不给内容（第十七章之二）；八面里无一给版本号（第十九章） |

三方之间**没有任何一对共享主键**：Imagination 按 DDK 版本给范围、Android 公告按 `PowerVR-GPU` 给 Subcomponent、设备侧两者都读不到（第九章之四原样给出了这条三方断裂）。因此本域允许的运算只有一种：**建映射表并记录映射类别**（第十六章的 A/B/C 三档），不允许任何"按字符串相等连接"的写法。台账里凡出现"公告词直接查设备属性"的实现，都要按错误实现处理。

### 廿二之二 规则一：期望值只能来自实测，内核注释不算期望值

第十七章之五已经在本树里取到一个三处不一致的样本：`remoteproc_sysfs.c:95-110` 的文档注释写 `The default value of this entry is "default".` 并单列 `default:` 一节，实现（`:111-134`）只接受 `disabled`、`enabled`、`inline`，其余落到 `"Invalid coredump configuration\n"` → `-EINVAL`；注释里 `enabled` 一次也没出现；`coredump_show()` 上方的 `/* Expose the current coredump configuration via debugfs */`（`:86`）说的还是另一个文件系统。三处在 20 行之内。

所以规则是：**注释、`MODULE_PARM_DESC` 文本、sysfs ABI 文档、内核 wiki 一律登记为 `static_baseline`（来源标签），不能作为断言的判定条件**；判定条件只能来自同一台设备上的实测读数，或第廿八章脚本 2 的两次快照之差。这条规则同时解释了第二十章为什么把 `rule` 列限定成"管住它的规则文件:行号"而不是"文档怎么说"。

### 廿二之三 规则二：子串命中不等于词命中，先把两种计数分开

本次实测（本地副本，命令原样给出）：

```
grep -c  "npu" private/shell.te           → 7 行
grep -o  "npu" private/shell.te | wc -l   → 8 处
grep -cw "npu" private/shell.te           → 0
grep -ro "npu" drivers/accel drivers/staging/media | wc -l → 629 处
grep -rwo "npu" drivers/accel drivers/staging/media        → 0 命中
```

那 629 处的宿主词全部是 `input` 一族：`input`(160)、`input_config`(61)、`inputs`(41)、`input_res`(37)、`atomisp_input_stream_id`(22)、`sp_pipeline_input_terminal`(11)……`private/shell.te` 的 7 行落在 `:3`（`# allow shell input injection`）、`:342-343`（`input_device`）、`:539`、`:542`、`:544`、`:546`（`neverallow shell input_device:chr_file no_w_file_perms;`）——**全部是输入设备，与神经网络毫无关系**。`private/service_contexts` 同理：含 `npu` 的三行是 `android.hardware.input.processor.IInputProcessor/default`（`:63`）、`android.hardware.tv.input.ITvInput/default`（`:125`）、`inputflinger`（`:300`）。

规则：**任何"NPU/GPU/ISP 是否有授权"的结论，必须来自词边界匹配或类型名匹配，并报告两种计数。** 若一个采集器或审计脚本用 `grep -i npu` 得出"shell 可访问 NPU 设备"，那是把 `input_device` 读成了 NPU 授权——本域最常见的一条假阳性，且它会直接写错整改建议的责任方。

**同一条规则的第二个实测形态是扩展名检索。** 在 `drivers/accel` 与 `drivers/staging/media` 两个目录集合里做三轮，结果一轮比一轮干净：

| 检索式 | 命中 | 命中的是什么 |
| --- | --- | --- |
| `grep -ro "..bin"`（点未转义） | 宿主串 `s_bin` 142、`p_bin` 120、`y_bin` 100、`r_bin` 61、`o_bin` 58、`w_bin` 49、`i_bin` 31、`->bin` 27 | 全是 `*_binary` 一类标识符与结构体成员，**零个文件名** |
| `grep -ro "\.bin"`（点已转义） | 21 处 | 仍是混合体：`.binary` 字段访问（`sh_css.c:3950` `frame->planes.binary.size`、`ipu3-css-fw.c:101`/`:133`/`:137`/`:142`/`:243` 的 `file_header.binary_nr`）＋注释里的文件名（`ivpu_fw.c:46` 的 `/* TODO: Remove mtl_vpu.bin from names after transition to generation based FW names */`）＋真字符串常量 |
| `grep -ro '"[^"]*\.bin"'`（转义点 + 引号边界） | **12 处、10 个不同名字** | 只有镜像名 |

那 10 个名字就是候选名集合的全部来源：`ivpu_fw.c` 5 条（`"vpu_37xx.bin"`、`"mtl_vpu.bin"`、`"intel/vpu/vpu_37xx_v0.0.bin"`、`"vpu_40xx.bin"`、`"intel/vpu/vpu_40xx_v0.0.bin"`）、`atomisp_v4l2.c` 3 条（`"shisp_2401a0_v21.bin"`、`"shisp_2401a0_legacy_v21.bin"`、`"shisp_2400b0_v21.bin"`）、`ipu3-css-fw.h` 2 条（`"intel/ipu3-fw.bin"` 与那条日期长名）。**只有第三种写法给出的是装载对象清单，前两种给出的是符号清单**——而第十九章 V1 那一列的候选名集合就是第三种写法的结果。FIMC-IS 的两条镜像名不在本次检索的两个目录内（位于 `drivers/media/platform/samsung/exynos4-is/fimc-is.h:32-33`），需要单独一轮，这正是"检索式必须逐目录集合声明范围"的原因。

### 廿二之四 规则三：标识符在场不等于读数可得

`drivers/accel` 与 `drivers/staging/media` 两个目录集合里，`firmware_version` 共 5 处命中，逐条性质如下：

| 命中 | 是什么 | 是不是一个可读的版本值 |
| --- | --- | --- |
| `vpu_boot_api.h:59` `u64 firmware_version_load_address;` | 头部字段：版本串将被装载到的**地址** | 否 |
| `vpu_boot_api.h:60` `u32 firmware_version_size;` | 版本串**长度** | 否 |
| `atomisp_v4l2.c:1398` `err = sh_css_check_firmware_version(...)` | 函数调用（且该函数恒返回 `false`，见第廿一章 FT3） | 否 |
| `sh_css_firmware.c:189` `sh_css_check_firmware_version(struct device *dev, const char *fw_data)` | 函数定义行 | 否 |
| `sh_css_firmware.c:253` `ret = sh_css_check_firmware_version(dev, fw_data);` | 第二处调用 | 否 |

同一集合里 `DEVICE_ATTR` 的命中数为 **0**：这两族驱动根本不通过设备属性宏暴露字段（atomisp 走 `driver_create_file()`，`atomisp_drvfs.c:164-172`；ivpu 走 debugfs，`ivpu_debugfs.c:107-115`）。

规则：**源码里检索到 `firmware_version`、`fw_ver`、`version` 字样，只证明"内核内部有这个名字"，不证明"设备上有这个读数"。** 从标识符到读数之间必须再走两步：找到它的出口（`DEVICE_ATTR`/`__ATTR`/`debugfs_create_file`/`seq_printf`/`dev_info`），再判断出口所在的面在采集域内是否可达（第廿章的 P-id）。这两步在第廿五章的矩阵里是两个不同的列，不允许合并成"可采"。

### 廿二之五 规则四：路径键必须保留扩展名，也不能把硬件代际当版本

第十五章之五给出两个必须遵守的细节：`qcom/venus-4.4/` 这一个目录在两条记录里分别以 `.mbn`（`:630`）与 `.mdt`（`:693`）出现——**扩展名不同意味着走的是不同解析路径**（`.mdt` 是分段/认证描述符）；目录名里的 `venus-1.8`、`venus-4.4`、`vpu-2.0` 是硬件代际标签，不是固件版本。另一侧的对照来自第十三章：高通 PIL 重定位记录里的镜像标识只有 8 字节（`PIL_RELOC_NAME_LEN`，`qcom_pil_info.c:19`），意味着台账里看到的镜像名可能已经被截断。

规则：**媒体固件的路径键 = 目录 + 文件名 + 扩展名，三段都要入键**；任何以"去掉扩展名归一"的写法会把两条不同解析路径合并成一条。硬件代际标签要单列成 `hw_gen` 字段，禁止落入版本列——这是第廿六章禁止跨厂商比较版本串的前置条件之一。

### 廿二之六 规则五：宽度决定匹配方式，全等匹配在本域必然失败

三种截断长度加自由文本，构成本域名字面匹配的全部陷阱：

| 载体 | 常量 | 宽度 | 出处 |
| --- | --- | --- | --- |
| PIL 镜像标识 | `PIL_RELOC_NAME_LEN` | 8 字节 | `qcom_pil_info.c:19` |
| VPU 自报版本 | `VPU_FW_VER_LEN` / `char fw_ver[]` | 16 字节 | `mtk_vpu.c:31` / `:147` |
| SCP 自报版本 | `SCP_FW_VER_LEN` / `s8 fw_ver[]` | 32 字节 | `drivers/remoteproc/mtk_common.h:80` / `:84` |
| rpmsg 服务名 | `#define RPMSG_NAME_SIZE 32` | 32（含 NUL 实为 31） | `include/linux/mod_devicetable.h:465` |

再加上 FIMC-IS 的 `FIMC_IS_FW_INFO_LEN 31` / `FIMC_IS_FW_VER_LEN 7` / `FIMC_IS_SETFILE_INFO_LEN 39`（`fimc-is.h:47-51`）与 IPU3 的 `%.64s`（`ipu3-css-fw.c:136-138`），本域同一格字段在不同链上的宽度从 7 到 64 都有。

规则：**匹配按前缀 + 记录截断长度，不按全等；两条不同宽度的读数相等不能升格为"同一个固件"。** 第十八章之四那条"截断即失配"（服务名 31 字符截断后与 `id_table` 全称不匹配，`rpmsg_core.c:480`）是同一条规则在总线侧的表现。

### 廿二之七 规则六：大小写在本域是跨厂信息，不是排版差异

第十六章之三实测：联发科侧写 `display`（对应 MDP 图像管线 + `SCP_IPI_IMGSYS_CMD` 固件通道，`mtk-img-ipi.h:84`/`:106`、`mtk_scp.h:46`，以及不跑固件的显示控制器驱动两类东西），高通侧写 `Display`（落在 `drivers/gpu/drm/msm/disp/`）。**同一份资产映射脚本若做大小写不敏感匹配，会把两厂的两个不同对象合成一个键；若做大小写敏感精确匹配，又会把同一厂的两个同形词分成两个键。**

规则：**大小写要原样保留入台账，匹配用显式的"厂商 + 词"二元键，而不是对词本身做 normalize。** 联发科那个 `display` 尤其危险：它一半是寄存器访问、一半是镜像执行，安全属性完全不同（F1/F2 层 vs 纯驱动层）。

### 廿二之八 规则七：厂商文档的档位之间不等价

第九章之二抄下 Arm 自己的三档分工：`Arm Security Bulletins`（缺陷影响 Arm 产品，给技术细节与修复指引）、`Arm Security Updates`（`when a security issue does not affect Arm products, but where Arm technology may be referenced or considered relevant`）、`Arm Security Articles`（分析文章）。**一条以 `Mali` 为 Subcomponent 出现在 Android 公告里的缺陷，在 Arm 侧可能只以 Update 而非 Bulletin 形式存在**——两侧对"这算不算一个已披露缺陷"的判断不等价。Imagination 侧的对应问题是页面自称覆盖 `graphics drivers`（`This page contains summary details of security vulnerabilities reported on Imagination Technologies graphics drivers.`），而本域要判的是命令流固件；样本行的字段是 `Versions affected` + `DDK Releases up to and including 25.2 RTM`，参考号体系是 `Our Reference PSP-19` / `Originator Reference PP-171174`。

规则：**"厂商有没有披露"必须先说清按哪一档口径问**。台账里公告侧断言要带 `disclosure_tier` 字段（bulletin / update / article / advisory / 无），并在跨侧比对时禁止把 Update 与 Bulletin 合并计数。这一条也是第廿四章每条案例都必须标注"来自哪一档文档"的原因。

### 廿二之九 规则八：入口不可读是一等状态，不是失败重试项

第九章实测四类拒因：SPA 外壳（Qualcomm www，本次 10,099 字节）、登录重定向壳（docs 侧任意路径 **302** → 200 的 `account.qualcomm.com - Sign In`，两个不同路径分别 149,716 与 149,819 字节 ⇒ **尺寸并不相同，"同尺寸壳"这个说法已被本次复测推翻**；而更早一轮在同一路径取到的是 52,406 字节 `<title>Qualcomm Documentation</title>` 壳，如今也不存在了）、需二次验证的应用壳（Arm 单篇）、反爬 403（Samsung 移动，正文 146 字节恰为 `<h1>403 Forbidden</h1>` + `nginx`）。再加两行必须原样保留的：`https://security.android.goog/` 本次不可达（经代理 `curl: (35)`、直连 `curl: (6)`），本报告不宣称它不可用，只记录未取回；`docs.qualcomm.com/bundle/publicresource/FP80-43795-14V_Whitepaper.pdf` 本次 302 → **403**，正文 70 字节 `{"location":"https://myid.qualcomm.com/api/v2/login?state=Direct URL"}`。

规则：**公告侧断言的取值集合里必须有 `not_reachable` 与 `not_applicable` 两个不同码位，且禁止合并**；`拒因` 要单独成列（SPA / 同尺寸壳 / 二次验证 / 403 / TLS / DNS），因为**取不到的机制与修复状态毫无关系**——"访问被拒"永远不能推成"厂商未修复"。第卅章的复采清单按拒因分别给出复采动作。

### 廿二之十 第廿四章案例读数的取样偏差声明（先声明，再读案例）

第廿四章给出十三组公开案例的读数。它按下面的分布取样，而这个分布**不是缺陷分布**：

1. **入口可读性决定可取样性。** 五个厂商入口里只有 MediaTek 与 Imagination 可匿名读全文（第九章之一）。因此案例集在联发科与 PowerVR 上偏密，在高通闭源组件与三星移动上偏疏——**这不是因为后两家缺陷少，而是因为它们的正文本次取不到。**
2. **公告词的可映射性决定可归类性。** 占位串 `Closed-source component`（11 行）与 `Security`（5 行）这两档无法落到器件，所以案例集里凡需要按器件聚合的行，都不会包含高通闭源条目。
3. **只有已编号、已公开的条目进入案例集。** 本域存在大量"固件被换过但从未成为 CVE"的现实（第八、九章），它们不在案例集里，也不在指标里。
4. **上游代码可见性决定源码侧对照能否成立。** `apusys`（C 档）与 Qualcomm 闭源组件在 GKI 树里没有实体，因此它们的案例只能停在公告口径，不能像 Mali/Adreno/Venus/MFC/IPU3/atomisp/FIMC-IS/ivpu/qaic/CDSP 那样给出装载路径与闸门形态。

由此得到三条禁止推出的结论：**不得从案例集推出"某类链缺陷少"**；**不得从案例集的厂商构成推出覆盖率**；**不得把"取不到正文"记进任何"未修复"统计**。这三条会在第廿九章以误区形式再出现一次，那里给出的是错误写法样本，本章给出的是它的成因。

### 廿二之十一 本章结论

1. **八条规则里六条是匹配规则（二至六、七），两条是断言规则（一、八）**。前者防止台账写错，后者防止台账把不可知写成结论。
2. **本域假阳性的根因是子串与标识符**：`npu`⊂`input`（实测 629 : 0）、`firmware_version` 五处命中无一是读数、`\.bin` 仍命中 `.binary` 字段访问（21 处里只有 12 处在引号内）。凡"检索即结论"的写法，都要补一步词边界、引号边界与出口确认。
3. **三方无共同主键是结构事实，不是工程未完成**（第九章之四、第十六章之八）。因此映射表要长期维护，且必须带 A/B/C 档位与截断长度两列。
4. **取样偏差必须在读案例之前声明**，否则第廿四章的密度会被误读成风险分布。声明本身要成为报告的一部分，而不是脚注。
5. 第廿三章把本章规则二、三、四用到编号层：给出五个被公开材料广泛误传的 CVE 编号形态，以及"抄来的编号比没有编号更危险"的具体机制。
## 廿三、编号陷阱：五个 CVE 编号形态，加上六条同样致命的主键误传

> **视角声明**：本章不判定任何缺陷能不能触发，也不给出任何编号对应的修复内容。它只处理一件事——**你抄进台账的那串字符，凭什么被认为指向一个真实存在的东西**。本章是第廿二章规则二、三、四在编号层的落地。

> **出处约定**：本章引入的新来源只有一类——Android 安全公告的月度页面与一个索引页，全部由本会话取回并留存本地副本，检索命令、命中数、URL 形态与页面 `Last updated` 戳逐字登记在第卅章 **D-9**（首批六页）与 **D-12**（本期扩样与 404 探测）。样本期 2025-09-01…2026-10-01 共探测 14 个月：11 页取回成功、3 页（2025-10、2025-11、2025-12）返回 404，成功的 11 页里 3 页（2026-02、2026-07、2026-08）没有任何缺陷表——这三项本身按本章规则六入账。正文里出现的每一个计数（175、164、62、72、29…）都是对这些本地副本执行 `grep` 的结果，不是从任何二手材料抄来的。本章不新增源码引用；引用源码处一律回指第廿二章 D-8 与第十七、十八章的既有记录。

### 廿三之一 为什么编号值得单列一章

本域的工单要跨三个系统流转：公告（对外）、厂商内部跟踪系统、设备侧台账。这三套系统各有各的主键，而唯一能把三套串起来的，是公告那一列 References 里并排写着的编号。所以编号在本域不是标注，是**外键**。

外键抄错的代价在这条链上比在别处更贵，原因有二：

1. **公告的 Subcomponent 词与器件不是一对一**（第五章实测：21 个表名、`display` 与 `Display` 两种大小处分属两家、`Video` 是高通的引擎词而不是 MediaTek 的）。人如果只抄编号、不抄那一行的表名与 Subcomponent，工单就会派错团队——GPU 的缺陷派给显示，编解码器的缺陷派给相机。
2. **本域最常见的公开表述是"某某固件漏洞已修复"而没有编号**。一旦有人给这种表述补上一个编号，那个编号就再也无法被证伪：它会进入下一篇文章、下一个内部条目、下一份客户答复。这就是本章标题里那句话的机制——**抄来的编号比没有编号更危险**，因为"没有编号"至少还是一个可见的证据缺口。

本章给五个 CVE 编号形态（廿三之三至廿三之七，另加一个前置的取回形态廿三之二），再给六条不在 CVE 层、但同样一旦抄错就致命的主键误传（廿三之八）。

### 廿三之二 前置形态：HTTP 200 不等于取到编号表

这是最容易跳过的一步，因为它的失败不留痕迹。

同一站点、同一命名的月度页面，本会话取回七份，原始 HTML 的编号表密度差别如下（`grep -o '<table'`、`grep -o 'A-[0-9]\{8,9\}'`、`grep -oE 'CVE-[0-9]{4}-[0-9]{4,}' | sort -u`）：

| 页面 | `<table>` 数 | `A-` 号 | 唯一 CVE 号 | `Last updated` |
| --- | --- | --- | --- | --- |
| 2026-09-01 | 18 | 175 | 175 | 2026-10-22 UTC |
| 2026-10-01 | 6 | 25 | 25 | 2026-10-08 UTC |
| 2026-08-01 | **3** | **0** | **0** | 2026-09-30 UTC |
| 2026-06-01 | 12 | 114 | 114 | 2026-10-13 UTC |
| 2025-09-01 | 14 | 108 | 109 | 2026-09-30 UTC |
| 2024-06-01 | 11 | 37 | 37 | 2026-09-30 UTC |
| 2021-01-01 | 11 | 41 | 41 | 2026-09-30 UTC |
| 2017-01-01 | 49 | **96** | **90** | 2026-09-30 UTC |

2026-08-01 那一行是本报告要防的样本：请求成功、标题正确（`Android Security Bulletin—August 2026`）、正文里连"本月公告"的标准句式都在，但**编号表一个字符都没有**——该页的条目区由脚本注入，`curl` 取回的副本里只剩缓解、问答与版本三张表。更妙的是这个页面上 `CVE` 字样并非零：

- `grep -o "CVE" .sb-2026-08-01.html | wc -l` → **1**
- 那一处的完整上下文是一段脚本内联的 API key 片段：`...sPaW1qUtbnLmyT_kCVE`

也就是说，如果有人用"页面里出现 CVE 字样"作为取回成功的判据，他会在一个**零条目**的页面上得到一次命中，而且命中的是一个密钥串。这是第廿二章规则二（子串 ≠ 词命中）最干净的一次现实演示。

顺带把取回侧的三条实测一并登记（都在 D-9）：

- **同一站点的 URL 形态有两种**：2026 年是 `/docs/security/bulletin/2026/2026-09-01`，2025 年及更早是 `/docs/security/bulletin/2025-09-01/`（尾斜杠）。本会话在两种形态互换时拿到过 5 次 404，每次都只试一次即改用另一种，不做重试。
- **索引页不可作为清单来源**：`/docs/security/bulletin/aaos` 的原始 HTML 里能 grep 到的月度条目最大年份是 **2021**。要拿 2026 年的清单，只能按命名规则自己拼 URL——而拼 URL 又会踩上一条。
- **页面条目会变更**：2026-09-01 页的 `Versions` 表逐字给了四次修订——`1.0 / September 8, 2026 / Bulletin Published`、`1.1 / September 15, 2026 / Updated issue list and added AOSP links`、`1.2 / September 16, 2026 / Updated issue list`、`1.3 / September 17, 2026 / Updated issue list`。同一个 URL 的内容是**会变的**。

**规则（取回侧）**：一次公告取回的验收判据是"页面里编号表非空且编号数被记录"，不是 HTTP 状态码；台账必须同时记下 `bulletin_version`（`Versions` 表最后一行的版本号）与页面 `Last updated` 戳。少了这两个字段，两个月后没人能判断你的 175 条是从 1.0 还是 1.3 上数出来的。

### 廿三之三 形态一：CVE 编号的序列位长不定，且朴素的正则不是"漏"而是"造"

2026-09-01 页 175 个唯一编号按序列位长分档：**4 位 11 个、5 位 164 个**。4 位那 11 个逐字是 `CVE-2026-0001`、`CVE-2026-0008`、`CVE-2026-0010`、`CVE-2026-0054`、`CVE-2026-0065`、`CVE-2026-0084`、`CVE-2026-4967`、`CVE-2026-5729`、`CVE-2026-7476`、`CVE-2026-7477`、`CVE-2026-7639`——注意 0001 到 0084 是带前导零的四位小数值，4967 与 7639 是无前导零的四位数值，两种形态在同一个页面、同一列里混着。

位长还会更多：2025-09-01 页的 109 个编号里有 **1 个 6 位**——`CVE-2025-021701`，它同时带前导零（序列部分实际是 `021701`）。2017-01-01 页的 90 个编号则全部是 4 位。

而且**位长配比逐期剧烈波动**：2026-06-01 页的 114 个编号是 **4 位 43 个、5 位 71 个**（37.7% 是 4 位），同一年 9 月那一期的 175 个却是 11 与 164（6.3%）。也就是说，"这一期大概全是 5 位"这种从上一期学来的直觉，在下一期就会让 43 条 4 位编号暴露给朴素正则——这正是下面那张表要量化的失败面。

于是把 `CVE-\d{4}-\d{4}` 当成校验式的最常见写法，在这一个页面上的实测结果如下：

| 变体 | 唯一命中 | 其中真实编号 | 造出的编号 |
| --- | --- | --- | --- |
| `CVE-[0-9]{4}-[0-9]{4}` | 73 | 11 | **62** |
| `CVE-[0-9]{4}-[0-9]{5}` | 164 | 164 | 0（漏掉 11 条 4 位） |
| `CVE-[0-9]{4}-[0-9]{4,}` | 175 | 175 | 0 |
| `CVE-[0-9]{4}-[0-9]{4}([^0-9]\|$)` | 11 | 11 | 0 |
| `CVE-[0-9]{4}-[0-9]{7}` | 0 | 0 | — |

关键在于第一行的失败方式。它**不是漏**：漏会在台账里留下空格，空格是一个可见的证据缺口。它是**造**——73 个"命中"里 62 个是把 5 位真号左截断得来的、世界上并不存在的编号，逐字如 `CVE-2025-4856`（截自 `CVE-2025-48564`）、`CVE-2025-4865`（截自 `CVE-2025-48651`）、`CVE-2026-1228`、`CVE-2026-2045`。这 62 个串每一条都形似合法、能被下游任何"看起来像 CVE"的宽松校验放行，而且**永远查不到**：查 NVD 返回零结果，查公告返回零结果，于是排查的人得出的结论是"这个编号可能是内部编号/写错了/是零日"，而不是"我的采集脚本截断了"。

这就是"抄来的编号比没有编号更危险"的第一种机制，而且是自动抄写版本。

**规则**：编号正则一律用 `CVE-[0-9]{4}-[0-9]{4,}`（序列用"至少 4 位"），并强制右边界（后面不能紧跟数字）。台账里对每个编号记 `cve_id_raw` 与 `cve_id_normalized` 两列，raw 保留原始字符（含前导零），normalized 只做大小写与空白归一，**不做补零或截断**。位长分布本身要作为一期采集的输出项（本期实测：4/5/6 位都出现过），任何一次采集若位长集合与历史分布不符，先怀疑采集器。

### 廿三之四 形态二：编号的年份不是公告的年份，编号的槽位也不是编号的个数

两件事在同一层被混淆。

**（一）编号年份 ≠ 公告年份。** 2026-09-01 页的 175 条里有 4 条是 2025 年编号，逐字为 `CVE-2025-48564`、`CVE-2025-48565`、`CVE-2025-48566`、`CVE-2025-48651`。2026-06-01 页更极端：114 条里 **29** 条是 `CVE-2025-*`，即当期条目的四分之一挂的是上一年的编号。反过来，2026-10-01 页 25 条全是 2026 编号。

于是三条常见推论全部不成立：

- "编号年份 = 今年" ⇒ "这是今年新分配的缺陷"：不成立（29/114 的比例在此）。
- "今年的公告里没有去年的编号" ⇒ 漏采：不成立（4/175、29/114）。
- 按编号年份统计"本年度缺陷数"：这是把分配年份当修复年份，与本报告的 F1–F6 归属层完全无关（第十八章的结论"厂商文档档位不等价"在这里同样适用——编号的年份由 CVE 分配方决定，与器件厂商的修复节奏无关）。

**（二）编号槽位 ≠ 编号个数，而且双现不是噪声。** 同一页内，`grep -oE "CVE-[0-9]{4}-[0-9]{4,}" | sort | uniq -c` 的分布是：出现 1 次 155 条、出现 2 次 **20** 条 ⇒ **195 个编号槽位对 175 个编号**。若按"表里的编号行"计数，这一期会虚报 20 条、约 **+11.4%**。

本会话把这 20 条双现逐条归了因。做法不是纯文本 `grep`（它做不到），而是用 `awk 'BEGIN{RS="<"}'` 流式扫描：遇到 `<h2|h3|h4` 就更新"当前小节名"，遇到含 `CVE-…` 的片段就吐出 `编号<TAB>小节名`，于是每一次出现都带一个归属。实例 `CVE-2026-28581`：两次出现分别落在 `System` 与 `Google Play system updates`。20 条全部是同一形状——

| 第一次出现 | 第二次出现 | 条数 |
| --- | --- | --- |
| `System` | `Google Play system updates` | 17 |
| `Framework` | `Google Play system updates` | 2 |
| `Android runtime` | `Google Play system updates` | 1 |

⇒ **双现 = 一个 AOSP 缺陷同时进主表和 Mainline 复写表**，20/20 命中，没有第二种来源（第卅章 D-12 登记命令与输出）。这条比"按行计数会虚报"有用得多：`ref_slot_count - uniq_count` 在本域就是一个现成的指标——**这一期有 20 个缺陷被认成 Mainline 组件**，它等于第 16 篇的模块归属数，不等于"重复录入错误数"。台账若把双现当脏数据去重，就会把 Mainline 那一侧的归属信息整列删掉；若不去重，就会在同一工单上把缺陷数报多 11.4%。两种错都要在规则里写明：去重按编号，但保留两个来源小节作为 `component` 与 `module_component` 两列。

同一段扫描还暴露一处**纯文本采集器的硬限制**：`<td>` 里可以一次写多个编号——2026-09-01 页有 6 个这种格子，最大一格写了 6 个编号，逐字为 `<td>` 里的 `CVE-2026-28622, CVE-2026-28638, CVE-2026-28671, CVE-2026-45517, CVE-2026-45519, CVE-2026-45525`；2026-10-01 页有 1 个（`CVE-2026-45524, CVE-2026-49878`）；2026-06-01 页为 0 个。于是同一期页面有三个不同的"编号数"，本期实测：

| 页面 | 含 CVE 的 `<tr>` 行数 | 逐个吐出的槽位数 | 去重后编号数 |
| --- | --- | --- | --- |
| 2026-09-01 | 184 | 195 | 175 |
| 2026-06-01 | 117 | 117 | 114 |
| 2026-10-01 | 27 | 28 | 25 |

⇒ 三个数两两不等，且差额来源不同（行 < 槽位 是多编号格；槽位 > 唯一 是双现）。台账只能选一个口径并写明是哪一个；`ref_slot_count` 若从行数推，就在 2026-09-01 页少 11，而这一页恰好是多编号格最多的那一期。上面 (二) 用的 195/175 是"逐个吐出"口径，不是行数。

**（三）两族编号的数量本来就允许不等。** 2017-01-01 页：96 个 `A-` 号 vs 90 个 CVE 号，且 96 个 `A-` 号全部出现在 `mitigations` 小节之后的引用/缓解表里（该小节偏移之前的正文里 `grep` 命中 0）。也就是说**缓解类条目可以只有 bug 号而没有 CVE 号**。任何用"两边数量必须相等"做完整性校验的采集器，在这一期会直接判失败——而它应该做的只是把两族分开计数。

（四）同页重复的比例逐期不同，但**每期都有**：`grep -oE "CVE-[0-9]{4}-[0-9]{4,}" | sort | uniq -c` 的分布实测为——2026-09-01 页 1 次 155 条 / 2 次 20 条（195 槽位 vs 175 编号，+11.4%）；2026-06-01 页 111 / 3（117 vs 114，+2.6%）；2026-10-01 页 22 / 3（28 vs 25，+12.0%）。⇒ `ref_slot_count` 是每期都要重测的观测值，不能沿用上一期的比例，也不能假定"编号数 = 槽位数"。

双现的归属逐期也不同，这一点 (二) 的形状不能推广：2026-10-01 页 3 条全部是 `System` → `Google Play system updates`；2026-06-01 页 3 条里有 2 条同形状，但 `CVE-2025-48595` 的两次出现分别是 `Android and Google service mitigations` 与 `Framework` ——**前者不是缺陷表，是缓解措施正文里的一次提及**。所以"这个 CVE 在页面里出现过"和"这个 CVE 在某一期的缺陷表里有一行"是两件事；采集器若只按页面命中判定收录，会把缓解段落里的提及当成一条缺陷记录。规则：**归属只认 `<tr>` 里的 `<td>`，提及单列一个 `mentioned_in` 字段**（第卅章 D-12 记录这两条的抽取命令与输出）。

（五）跨期**会**重复，而且重复时主键会变。本期扩样后找到一例，逐字如下：`CVE-2025-48651` 同时出现在 2026-04-01 页与 2026-09-01 页，两处的 `Subcomponent` 都是 `StrongBox`、`Severity` 都是 `High`，但——

| 页面 | References | 所在表标题 |
| --- | --- | --- |
| 2026-04-01 | `A-434039170 *` | `Google` |
| 2026-09-01 | `A-529876547 *` | `Tsingteng Micro` |

⇒ 三条结论：**(a)** CVE 不是跨期主键，`(CVE, bulletin_date)` 才是；按 CVE 去重会把 2026-09 那一行当成旧数据丢掉。**(b)** 同一 CVE 的 `A-` 号在两期里不同，说明"公告那一列"记录的是**这一期的修复项**而不是缺陷本体；用 `A-` 号做跨期关联同样会断。**(c)** 一个 `StrongBox` 缺陷在两期被归到两个不同供应商表（`Google` 与 `Tsingteng Micro`），而这两张表在页面上的描述句都把严重度交给"厂商自己评"（第五章）——所以"这个器件归谁"在公告层不是稳定属性，第廿五章的归属矩阵必须把归属记成 `(期号, 表名)` 而不是记成一个供应商名。

这条推翻了本会话早期的一个假设。首批只取六页时，2026-09 页与 2026-06、2026-08、2026-10 三页的 CVE 交集用 `comm -12` 逐一核对均为 **0**，据此很容易写成"跨期不重复，风险只在同页重复"。扩样到 11 页（2025-09…2026-10，逐月探测）后，2026-04 页给出 1 条交集。**这就是本章规则六"取回范围要写全"的意义：样本期不足时，一个"实测为 0"的结论和"没测"无法区分。** 台账口径因此改为：跨期去重不是必需动作，但必须保留每期原文，并且默认 `A-` 号在两期之间不互认。

### 廿三之五 形态三：前缀白名单要从当期页面提取，不能从说明文档提取

公告页自己的 FAQ 第 4 问逐字写着：`Entries under the References column of the vulnerability details table may contain a prefix identifying the organization to which the reference value belongs.` 随后是一张前缀表，2026-09-01 页为 6 行：`A-` Android bug ID、`QC-` Qualcomm reference number、`M-` MediaTek reference number、`N-` NVIDIA reference number、`B-` Broadcom reference number、`U-` UNISOC reference number。

同一条 FAQ 在 2021-01-01 页只有 **5** 行，没有 `U-`。这张表是会长的。

再看 2026-09-01 页 References 列的实测族（`sort | uniq -c`，全部逐字登记在 D-9）：

| 族 | 本页出现 | 数字位长 | 与文档前缀表的关系 |
| --- | --- | --- | --- |
| `A-<digits>` | 175 | 9 | 文档有 `A-` |
| `M-ALPS<digits>*` | 10 | 8 | 文档写 `M-`，实际是 `M-` + 子系列 |
| `M-MOLY<digits>*` | 10 | 8 | 同上（文档从未说明 `M-` 之下还有分系列） |
| `M-AUTO<digits>*` | 3 | 8 | 同上 |
| `QC-CR#<digits>` | 5 | 7 | **文档写 `QC-`，页面写 `QC-CR#`** |
| `U-<digits>*` | 7 | 7 | 文档有 `U-`（2021 版文档无） |
| `N-`、`B-` | **0** | — | 文档列出，本页无行 |

三处错配，每一处都能让一个照文档实现的解析器静默出错：

1. **按文档字面匹配 `QC-[0-9]+`**：本页命中 **0**。真实形状是 `QC-CR#4384595` 这样的 `QC-CR#` + 7 位——文档给的前缀是真实 token 的**前缀的前缀**。这类"文档描述的是命名空间的开头、而不是 token 的形状"的错配，用任何精确匹配都会中招，而且结果与"高通本月没有条目"完全一样。
2. **按文档的 6 族做白名单**：`N-`、`B-` 在本期是空集，这不是异常；但 `M-` 之下实测有三个子系列（ALPS / MOLY / AUTO，分别是 10 / 10 / 3 条）。如果台账只记"厂商=M-，即 MediaTek"，就把三条不同产品线压成了一族——本域关心哪颗核（编解码器、Modem、APU），压成 `M-` 之后这条信息就永久丢失。子系列名是**唯一还留在编号里的器件线索**，必须单列一个字段 `vendor_ref_series` 保留。
3. **按某一年的文档做长期白名单**：`U-` 是后来加进文档的。2021 版文档写出的解析器，到 2026 年会漏掉 Unisoc 那一整张表（本页 7 条，全部在 `Unisoc components` 表内、全部带星）。

还有一个本域专属的缺口值得单写：**GPU 那一族器件在公告里没有厂商号**。2026-09-01 页的 `Imagination Technologies components` 与 Arm/Mali 相关行的 References 只有 `A-` 号 + `CVE` 号，没有任何厂商前缀（实测本页厂商号集合里不含 Imagination 或 Arm 的形状）。所以"按厂商引用号反查器件"这条路径对 GPU 固件天然不通——这与第六章"严重度评定权外包"、第九章"厂商入口四类只有一类可匿名读取"合起来，说明 **GPU 是本域里公告侧与器件侧之间引用号链最薄的一环**。

**规则**：前缀白名单必须从当期页面提取（本期：6 个文档族、5 个实际族、外加 `M-` 的三个子系列），并输出"文档有而页面无"的差集，而不是只输出"页面有"。任何一次采集若实际族集合发生变化（新增一族、或 `QC-` 变成别的形状），要当作**采集事件**上报，因为它意味着解析器需要更新，而不是意味着数据异常。

### 廿三之六 形态四：`*` 的语义比它的标题宽，而它对本域的验收是致命的

FAQ 第 5 问的标题是 `What does an * next to the Android bug ID in the References column mean?`，正文是 `Issues that are not publicly available have an * next to the corresponding reference ID. The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.`

先记一处标题与正文的宽度差：标题只说 Android bug ID，正文说 corresponding reference ID。实测支持正文那一版：2026-09-01 页 `href="#asterisk"` 锚点共 **72** 个（另有 1 个 `id="asterisk"` 定义锚点）。按紧邻前一个编号归属（`awk` 取前一行）：`A-` 44、`M-MOLY` 10、`M-ALPS` 10、`U-` 7、`M-AUTO` 3、`QC-CR#` **0**。星号确实挂在厂商号上。同格并存的实例逐字（Unisoc 表一行）：`A-508074243` 与 `U-3304745`，后面跟一个 `*`——**一个单元格里两个编号共用一个星号**。所以"这行的 `A-` 号带不带星"这类问法在按行解析时会出错；`asterisk` 必须按单元格记录，并按最后一个 token 计数。

（这一归属方法的误差要如实写出：72 个锚点、按前一行归属相加得 74，差 2 来自同一格内两个编号都被算到；本报告不用逐条归属值，只用族级分布与"每格一个星"这一形状。）

对本域真正要紧的是正文那句 `generally contained in the latest binary drivers`。把三件事连起来：

- 带星的条目 ⇒ 缺陷本身不可公开讨论；
- 其修复 ⇒ 在 **binary driver** 里，而不是在 AOSP 源码里；
- 而本文 L-5 条与第十八章已核验：固件 blob 所在目录**对 shell 域不可列举**，设备侧读不到镜像本身。

⇒ **对带星的固件类条目，设备侧不存在一个"读到镜像版本"的验收路径**。可验收的最强证据只剩驱动版本、HAL 版本与 SPL 字符串这三类间接信号，而这三类在第十九章的 V3/V4 与第二十章 P-09/P-10 里都已经被证明不等于固件版本。

因此台账侧要给本域一条硬口径：**带星且 Subcomponent 属媒体/加速器族（Video、`hevc decoder`、`apusys`、`Mali`、`ISP` 一类）的条目，验收状态一律登记为 `no_device_side_evidence`，不得登记为"已确认修复"**。这一条与 K5（不可验收条目占比）在第廿六章分母口径里对上。

顺带一条本期实测：本页 `M-` 系（23 条）与 `U-` 系（7 条）**全部带星**，而 `QC-CR#` 5 条**全部不带星**。这不是厂商习惯差异就能解释的（同一家厂商的不同系列位长也不同），最省事的解释是高通在公告里给出的引用号对应的缺陷信息本身可公开。本报告只登记这个分布，不解释成因；它的作用是提醒：**按厂商推断星号比例是无效的**，每个厂商族都要单独数。

### 廿三之七 形态五：厂商引用号与 CVE 非一一对应，且位长不能当校验

一一对应关系在本域的公开数据里就不成立，两个方向都验过：

- **一号多 CVE**：第五章已记录 CVE-2026-20501 与 CVE-2026-20502 共用一条引用 `M-ALPS11262030*`（同一个 MediaTek 内部号）。所以"按引用号统计厂商已修条目数"会**低估**。
- **一期多号不同长**：2026-09-01 页四族编号的位长分别是 `A-` **9**、`M-*` **8**、`U-` **7**、`QC-CR#` **7**。想用一个 `^[A-Z]+-[0-9]{8}$` 之类的"引用号格式"做校验，会在同一页里同时对两族失败、对两族放过。
- **位长还会跨年漂移**：`A-` 号在 2017-01-01 页的 96 个实例里全部是 **8** 位，在 2021-01、2024-06、2025-09、2026-09、2026-10 五期的 **386** 个实例里全部是 **9** 位（在这五页上搜 8 位形态 `A-[0-9]{8}` 后不接数字，命中 **0**，即不是混合期而是整期切换）。也就是说，如果采集器里硬编码了"Android bug ID 是 9 位数字"这个校验，它对历史页面的取回会**整页失败**——而且失败方式是"一条都没有"，看起来像历史公告没数据，实际是校验器把数据全滤掉了。

⇒ **禁止**用编号位长做校验、做厂商推断、或做"新旧格式"分类。位长只能作为**观测结果**记录（本期四条：9/8/7/7，与一条 8→9 的历史漂移）。

### 廿三之八 六条不在 CVE 层的主键误传

前面五个形态处理的是公告侧编号。本域还有六处"键"，抄错的后果与抄错 CVE 一样致命，且都被前面章节点名过；这里集中裁决，第廿四章的案例读数与第廿七、廿八章的脚本都按这一节执行。

**（1）把源码里的符号名当成设备上的读数。** `firmware_version` 在 `drivers/accel/` + `drivers/staging/media/` 集合里 `grep -rn` 命中 **5** 处（D-8 逐条登记），无一处是设备侧可读的版本值；同一集合 `grep -rn "DEVICE_ATTR"` 命中 **0**。第十七章之八已把这条列为编号陷阱预告，此处的裁决是：**台账里引用符号名时，`value_source` 必须写 `none`，并附一条"该符号的全部命中点"的检索证据**——只有符号名而没有这一条证据的记录，视为未采集。

**（2）把 HAL 接口版本当固件版本。** 第二十章 P-10（`cat /vendor/etc/vintf/manifest.xml`）里的 `version` 是 **HAL 接口版本**。它与固件版本同名、同列、同页，而 vintf 文件与固件目录在 sepolicy 里还同标 `vendor_configs_file`（`file_contexts:438`、`:436-437`）。裁决：**两个字段在台账里不允许共用一列**；HAL 版本入 `hal_interface_version`，固件版本入 `fw_version_reported`，且后者的 `value_source` 必须是设备侧读取点或 `none`，不能是 manifest。

**（3）把检索词当结论。** 第十八章之一已给实例：只用 `fw_version` 一族命名作检索入口，会得出"venus 没有固件版本"的错误结论；真实情况是驱动向固件主动索取版本字符串、且该字符串由固件自报。裁决：**"读不到"的结论必须附一张检索词族表**（本文实际用到的族：`fw_version` / `version` / `ver` / `revision` / `hdr_version` / `*header_version` / `%llu` 打印点 / sysfs-debugfs 出口名），族表本身作为证据类型 E4 记入第卅章。

**（4）把四态压成一个布尔。** `absent`（路径不存在）、`unlistable`（父目录不可列举但已知名可 `stat`）、`eacces`（能 `stat` 不能 `read`）、`vendor_dependent`（AOSP 无授权，结果取决于厂商补丁）——第十九章之三与第二十章之一都点过这条。裁决：**这四个状态在采集脚本输出里必须是四个不同的字符串**，合并即缺陷；脚本 1 的分支覆盖验收（第廿九章）按此设条。

**（5）多镜像对象按器件登记。** FIMC-IS 的 setfile 装载走状态位等待，两段超时分别打 `"get setfile address timed out\n"` 与 `"loading setfile timed out\n"`（`fimc-is.c:671-686`，`:680` 载入、`:682` `fimc_is_hw_load_setfile(is)`）。因此"固件在场而 setfile 不在场"与"两者都缺"外在表现同类。裁决：**V7 行为反证台账按镜像分列，不按器件分列**；一个器件多行镜像时，行键是"器件 + 镜像名 + 扩展名"，缺一列就是第（6）条的错。

**（6）路径键丢扩展名。** 第十八章之四实测：`qcom/venus-4.4/` 这一个目录在两条记录里分别以 `.mbn`（`:630`）和 `.mdt`（`:693`）出现，扩展名不同意味着走的是不同解析路径；`venus-1.8` / `venus-4.4` / `vpu-2.0` 是**硬件代际标签**不是固件版本；`PIL_RELOC_NAME_LEN` 为 8，所以内核里那个 `fw_name` 缓冲区只可能装下前 8 个字符，任何"按名字全等匹配固件"的推论在 `.mbn`/`.mdt` 这类扩展名上都会走样。裁决：**归一化后的路径键必须保留扩展名**，且代际标签单列 `hw_gen`（第廿二章规则四的编号层版本）。

### 廿三之九 逐字纪律在编号层的具体形状

第十七章之十四已立下一条纪律：原注释的两处英文瑕疵（`th kernel driver`、末句无句号）**照抄不改**，因为它们是取回核对的指纹。编号层沿用同一纪律，并补三条：

1. **编号不做补零、不做截断、不改大小写以外的任何变换**。`CVE-2025-021701` 一旦被"整理"成 `CVE-2025-21701`，它就变成一个不存在的编号，而且新形状仍然合法——这是第（一）条存在的全部理由。
2. **引用号后缀的 `*` 是编号的一部分**。`M-ALPS11262030*` 去掉星号后与"这条缺陷可公开讨论"这个断言绑定，属于**语义改写**，不只是格式改写。台账里 `asterisk` 单列一列。
3. **文档句子与页面 token 分栏记录**。本章廿三之五的 `QC-` / `QC-CR#` 错配之所以可发现，是因为"文档逐字"与"页面实测"被记在同一行的两栏里。任何一份只写"厂商引用号"的台账都无法暴露这类错配。

### 廿三之十 编号层的最小可执行台账字段

前九节合起来，落到字段上是这十个（第廿五章的归属矩阵与第廿七章脚本 1 的表头按此对齐）：

| 字段 | 取值约束 | 由哪一节立 |
| --- | --- | --- |
| `cve_id_raw` | 逐字，含前导零；位长 ∈ {4,5,6} | 廿三之三 |
| `cve_seq_len` | 4 / 5 / 6，作为观测不作校验 | 廿三之三、廿三之七 |
| `cve_year` 与 `bulletin_date` | 两列并存，允许不等 | 廿三之四 |
| `ref_slot_count` | 该编号在本页出现的槽位数（1 或 2） | 廿三之四 |
| `bug_id_raw` | 逐字，位长 ∈ {8,9} | 廿三之七 |
| `vendor_ref_family` / `vendor_ref_series` | 如 `M-` / `ALPS`；无厂商号时 `none` | 廿三之五 |
| `asterisk` | `yes` / `no` / `cell_shared`（一格两号共一星） | 廿三之六 |
| `fw_class` | 该行的 Subcomponent 是否属媒体/加速器族 | 廿三之六 |
| `bulletin_version` / `page_last_updated` | 取回时页面的 `Versions` 表末行版本号与 `Last updated` 戳 | 廿三之二 |
| `page_entry_check` | 编号表非空判据（`<table>` 数、`A-` 数、唯一 CVE 数） | 廿三之二 |

`fw_class = yes` 且 `asterisk = yes` ⇒ 验收状态强制 `no_device_side_evidence`。这是本章唯一一条会传导到第廿六章判定模型的规则，其余都是采集与记账侧的约束。

### 廿三之十一 本章结论

1. 本域的编号是**外键**，不是标注：它同时被公告、厂商内部系统和设备台账引用，抄错一次会在三处分别生效且互相不能纠错。
2. 朴素 CVE 正则的实际失败方式是**造出 62 个不存在的编号**（本期实测），而不是漏掉；造出的编号能通过宽松校验、查不到出处，最终会被解释成"内部编号"或"零日"——这是"抄来的编号比没有编号更危险"的头号机制。
3. 编号年份、编号位长、厂商前缀形状三者都**不能当校验**：本期实测 4/5/6 位混合、且配比逐期剧变（2026-06 页 43:71，2026-09 页 11:164）、`A-` 号 8→9 位的历史漂移、文档 `QC-` 对页面 `QC-CR#`、文档列 6 族而 `N-`/`B-` 本期为 0 行。
4. 同一页面有三个不同的"编号数"，本期实测 2026-09-01 页为 **184 行 / 195 槽位 / 175 唯一**，按行计数比槽位少 11、比唯一多 9；20 条双现全部是 `System`/`Framework`/`Android runtime` 与 `Google Play system updates` 的组合，即 **Mainline 复写**，不是录入错误——去重要按编号，但两个来源小节必须分别留作 `component` 与 `module_component`。跨期也不是安全的：`CVE-2025-48651` 在 2026-04 与 2026-09 两期都出现，`A-` 号不同（`A-434039170` / `A-529876547`）、表标题不同（`Google` / `Tsingteng Micro`）。台账主键因此是 `(CVE, bulletin_date)`。
5. `*` 的语义按正文宽于按标题，且实测确实挂在厂商号上（`M-` 系 23/23、`U-` 系 7/7 带星，`QC-CR#` 0/5 带星）。它的修复位置逐字是 `binary drivers`，而本域 blob 目录不可列举 ⇒ **带星的媒体/加速器条目在设备侧不存在验收路径**，这类条目登记 `no_device_side_evidence`，这是 K5 的分母来源之一。
6. 取回判据必须是"编号表非空 + 版本号 + `Last updated`"，不是 HTTP 200：2026-08-01 页取回成功而条目为零（2026-02-01、2026-07-01 两页同样只有 `Announcements` 与缓解小节，零缺陷表），页内唯一的 `CVE` 字样是一段 API key 的子串；而 2025-10、2025-11、2025-12 三期直接 404。**"这个月没有公告"和"这个月公告没取到"在台账里必须是两种状态**，否则一个采集缺陷会被解释成"该期无缺陷"。
7. GPU（Mali/Imagination）在本域的公告侧引用号链最薄：它的行里没有厂商号，只有 `A-` + `CVE`。按厂商号反查器件这条路径对 GPU 固件天然不通。
8. 第廿四章把本章的十个字段用到十三组公开案例的读数上，并按三条到达通道分列；凡该章引用的编号，都必须在第卅章的取回记录里能找到同一页面的同一串字符。
## 廿四、案例读数：十三组公开条目按三条到达通道分列，逐组给出台账动作

> **视角声明**：本章只做一件事——把公告页上真实存在的行，按第廿三章那十个字段读一遍，然后说明每一组在设备侧能核验到哪一步。它不判定缺陷类型，不推测利用条件，不给出任何修复内容；凡"这条 CVE 具体是什么漏洞"，本章一律写 `not_disclosed`，因为公告本身没写（第 5 章结构性事实：厂商表**没有 `Type` 列**）。

> **出处约定**：本章全部读数来自四个月度公告页面的本地副本（2026-09-01、2026-06-01、2025-09-01、2026-10-01），取回方式、URL 形态与页面 `Last updated` 戳见 D-9，行级令牌提取方法与逐字行见 **D-10**。第 5 章曾记录的 `DD+ Codec`、`VBMeta`、`GDMC`、`GSA`、`DA` 五组条目在本会话的四页副本里检索为 0 命中，**本章不重读它们**，需要引用时按第卅章的未取回清单另行取回——引用一个自己没取回的行，正是第廿三章要防的那类错。

> **取样偏差**：先读第廿二章之十那四条声明再读本章。本章在联发科与 PowerVR 上偏密、在高通闭源组件与三星上偏疏，原因是入口可读性与词可映射性，不是缺陷密度。

### 廿四之一 三条到达通道的定义与判据

本域的"到达"必须同时问两件事：这条缺陷信息怎么到你手上，以及它的修复怎么到设备上。二者在 Android 里由同一张表的不同列形状决定，所以可以按列形状把条目自动分箱——不需要读正文（正文常常也没有）。

| 通道 | 判据（同一行的列与令牌形状） | 披露侧 | 送达侧 | 设备侧最强可读证据 |
| --- | --- | --- | --- | --- |
| **T1 平台源码通道** | 行含 `Type` 列（`RCE`/`EoP`/`ID`/`DoS`）+ `Updated AOSP versions` 列（如 `14, 15, 16, 16-qpr2, 17`） | AOSP 有源码与 commit；进 SPL 条件 | AOSP 树 → 平台镜像 | `ro.build.version.security_patch`（间接，只声明 Android 侧） |
| **T2 厂商公告通道** | 行只有 `Severity` + `Subcomponent`，**无 `Type`、无版本列**；表句为 `... further details are available directly from X. The severity assessment of these issues is provided directly by X.` | 细节与严重度都在厂商手里；`Type` 缺位 ⇒ 无法从公告归类缺陷 | 厂商自有镜像/blob 包 → 厂商 OTA | 常常**无**（第十八章三例证 + 第 L-5 条） |
| **T3 记账例外通道** | 不在缺陷表里，而在两处官方自述的机制里：`/vendor/etc/security/supplemental_security_patches.xml`（`KEY_VENDOR_SUPPLEMENTAL_PATCHES: Aggregates patches from /vendor and /odm partition paths.`）与 `ro.vulkan.apex` | OEM **自愿**上报，不在 SPL 声明链上 | 同一份 XML 随分区安装；Vulkan 用户态驱动可随 APEX | 可读该 XML（若厂商放置且 shell 域可达）；`getprop ro.vulkan.apex` |

三条硬边界，先把话说死：

1. T1 与 T2 的区分是**列形状**，不是厂商。同一期里高通两张表分别落 T1 形（`Kernel components` 等平台段）与 T2 形（`Qualcomm components` / `Qualcomm closed-source components`）。
2. T2 的行**不提供缺陷类别**。第廿一章 FT1–FT10 的归类只能由设备侧与源码侧证据做，不能由公告做（第 5 章已把这条列入未公开文档化边界清单）。
3. T3 不是第三条"公告通道"，它是本域唯一一条**器件修复能被读出来**的通道（第八章之三）。它的重要性来自它的稀缺：前两条都不提供这个。

### 廿四之二 通道判据之外的一条口径警告：严重度的前提是"缓解视为关闭"

四页正文都有同一句（逐字，链接指向 `source.android.com/security/overview/updates-resources.html#severity`）：

> `The severity assessment is based on the effect that exploiting the vulnerability would possibly have on an affected device, assuming the platform and service mitigations are turned off for development purposes or if successfully bypassed.`

对本域的含义：**公告的 `Critical`/`High` 是在"缓解关闭"这个假设下评的**。所以拿严重度当风险读数时，必须同时记下这台设备实际启用了哪些缓解——而这恰恰是第十九章 V1–V8 与第二十章 P-01…P-26 那张姿态表要做的事（第廿六章 K3 的分母据此定义）。把"厂商评了 High"直接写成"本机构风险为 High"是本章要避免的第一种误传。

### 廿四之三 组 1：平台表里的媒体/框架行——信息最全，也最难落到器件

2026-09-01 页一行逐字：`CVE-2026-28581 A-479515555 ID High 14, 15, 16, 16-qpr2, 17`。

读数：`cve_id_raw` 5 位、`cve_seq_len` 5、`bug_id_raw` 9 位、`asterisk` 无、`vendor_ref_family` `none`、`ref_slot_count` **2**（同页双现，见组 4）。T1 通道。

台账动作：这类行是本域唯一能自动核验到"是否进了 SPL"的形态（`Updated AOSP versions` 直接给出受影响版本）。但它的主语是 Framework/媒体**软件**，不是编解码器引擎的镜像——第 16 篇的 `com.android.media` 与本域的解码器固件是两侧（第 4 章分工表）。**禁止**把这类行记进"器件镜像未修"的统计；正确写法是同时落两列：`platform_row_fixed = yes (SPL 2026-09-01)` 与 `device_side_evidence = not_applicable`。

### 廿四之四 组 2：`Kernel` 段与 `Upstream kernel` 引用形态

2026-09-01 页 `Kernel` / `Kernel components` 段逐字行（节选）：

- `CVE-2026-31629 A-499350302 Upstream kernel [ 2 ] EoP Critical NFC`
- `CVE-2026-58846 A-508703529 Upstream kernel EoP Critical Protected Kernel-Based Virtual Machine`
- `CVE-2026-43049 A-482992246 Upstream kernel [ 2 ] EoP High Human Interface Device`
- `CVE-2026-52993 A-500022799 Upstream kernel [ 2 ] RCE Critical Transparent Inter-Process Communication`

读数：References 列出现第廿三章前缀表**没有**的形态——`Upstream kernel` 加一个方括号链接索引（`[ 2 ]`，另有 `[ ]` 单独出现的行）。它不是编号，是"到上游内核 bug 区的链接"。`Type` 有、`Subcomponent` 有，但没有 `Updated AOSP versions`（内核段用自己的段结构）。

台账动作：`vendor_ref_family = none` 之外必须再留一个 `ref_other` 列，把 `Upstream kernel [2]` 这类非编号引用原样存下。按第廿三章规则（前缀白名单从当期页面提取），这类 token 一旦被当作"引用号"解析就会污染 `vendor_ref_series`；一旦被判为"格式错误"丢弃，就会丢掉本域唯一能指向内核侧修复的线索。内核侧的可达性与取证面按第 1 篇（内核漏洞与提权）口径，本章只登记引用形状。

### 廿四之五 组 3：`Google Play system updates` 段——第三条更新语义的入口

2026-09-01 页该段首句逐字：`The following issues are included in Project Mainline components.`，段内行如 `CVE-2026-28593 A-388272624 EoP High 14, 15, 16, 16-qpr2, 17`。

读数与组 1 同类（T1 形，有 `Type` 与版本列），但**它不属于 SPL 声明链**——第 8 章之二的场景 C 在这里出现的具体形态就是"设备上的 Play 系统更新串与 SPL 串日期不一致"。第 16 篇已经把模块维度判完，本章只取一条与本域有关的事实：**媒体编解码器的软件侧（`com.android.media*`）可以按模块到设备，而同一颗引擎的镜像不在这张模块表里**（第 4 章分工表 + 第三之五节）。

台账动作：凡是把 T1 行标为"已到设备"的，必须写清是按 SPL 还是按 Play 系统更新串的日期判断，两列不合并（第 16 篇口径）。

### 廿四之六 组 4：同页双现的机制——195 个槽位从哪来

2026-09-01 页 20 条编号各出现两次。第二次的载体是一张**按特性名分组、一格并列多个编号**的表，逐字例：

- `CVE-2026-28581, CVE-2026-49881` → `UWB`
- `CVE-2026-28623, CVE-2026-28652` → `WiFi`
- `CVE-2026-28604` → `adbd`
- `CVE-2026-28622, CVE-2026-28638, CVE-2026-28671, CVE-2026-45517, CVE-2026-45519, CVE-2026-45525` → `Telephonycore`

读数：这类行的主键是**特性**而不是缺陷，一格 1–6 个编号。任何按"表里编号槽位数"计缺陷的采集器，在这里一次性把 6 个编号记成 6 行"缺陷条目"，而这些缺陷在详情表里已经被记过一次。

⇒ 第廿三章 `ref_slot_count` 字段的实际用途在此：**先按详情表去重，再把缓解表当成"该缺陷有平台/服务缓解"的布尔信号**。后者对本域有独立价值（第廿六章 K3 的分母要它），但它不是缺陷计数。

### 廿四之七 组 5：`Arm components` 的 `Mali` 六连——星号挂法与本域最痛的缺口

2026-09-01 页六行逐字：`CVE-2026-0001 A-466377983 * High Mali`、`CVE-2026-5729 A-527029873 * High Mali`、`CVE-2026-7476 A-527043872 * High Mali`、`CVE-2026-7477 A-525773732 * High Mali`、`CVE-2026-12285 A-525777064 * High Mali`、`CVE-2026-12387 A-525782063 * High Mali`。

读数（第廿三章十字段）：编号位长 4/4/4/4/5/5（同一张表内混着，正是形态一）；`asterisk` 六行全 `yes`；`vendor_ref_family` **`none`**——Arm 的行没有厂商引用号；`fw_class = yes`（GPU 器件族）；`Type` 缺位（T2）。表句 `These vulnerabilities affect Arm components and further details are available directly from Arm. The severity assessment of these issues is provided directly by Arm.`

三条台账动作：

1. 按第廿三章之六的硬口径：`asterisk = yes` ∧ `fw_class = yes` ⇒ `verification_status = no_device_side_evidence`。这六行全部落入该状态，**不得**记为"已确认修复"，也不得记为"未修复"。
2. 六行共用一个词 `Mali`。第九章给出为什么这个词承载不下本域的区分：JM 与 CSF 是两套装载模型（第十章），命令流固件与内核驱动是可分别更新的两件东西。所以台账必须把这六行按**器件世代**再拆一次，拆分依据只能来自厂商入口或设备侧读数，不能来自公告。
3. Arm 无厂商号 ⇒ 反查链断在公告侧。要把 `CVE-2026-7476` 对齐到 Arm 自己的告警，只能靠 Arm 入口（第九章实测：单篇需二次验证）。`bulletin_version` 与 `page_last_updated` 在这种"当期读不到、下期才读得到"的场景里是唯一的对齐锚。

### 廿四之八 组 6：`Imagination Technologies` 的 25 行——同一形状的另一种密度

2026-09-01 页该表 25 行全 `High`、全带 `*`，节选：`CVE-2026-21732 A-468248021 * High PowerVR-GPU`、`CVE-2026-21733 A-483582720 * High PowerVR-GPU`。表句与 Arm 同形（`... provided directly by Imagination Technologies.`）。

读数：与组 5 同形（T2、无厂商号、全星、`Subcomponent` 一个词 `PowerVR-GPU`），差别只有两点：编号是 5 位段（`21732` 起）与密度是 25 行。

台账动作：第廿二章规则八在这里给出一条可执行差别——**Imagination 的入口是本章唯一"可匿名读全文"的 GPU 厂商入口**（第九章之一）。所以组 5 与组 6 在本机构的处置不同：组 6 允许补采（把 25 行的厂商侧描述与 DDK 版本档取回来对齐），组 5 只能停在公告口径。这条差别要写进工单模板，否则两批 GPU 条目会被同一条"不可核验"规则一起放过。

### 廿四之九 组 7：MediaTek 的媒体族——同一张表、四个内部系列、一处代号与系列错配

2026-09-01 页 `MediaTek components` 表内本域相关行逐字（D-10）：

| 行 | Subcomponent | 引用族/系列 | 星 |
| --- | --- | --- | --- |
| `CVE-2026-20464 A-518566453 M-ALPS11104718 * High` | `hevc decoder` | `M-ALPS` | yes |
| `CVE-2026-20501 A-528211066 M-ALPS11262030 * High` | `vdec` | `M-ALPS` | yes |
| `CVE-2026-20502 A-528209858 M-ALPS11262030 * High` | `vdec` | `M-ALPS`（**与上一行同引用号**） | yes |
| `CVE-2026-20467 A-518564544 M-AUTO00837766 * High` | `apusys` | `M-AUTO` | yes |
| `CVE-2026-20468 A-518595862 M-AUTO00833804 * High` | `apusys` | `M-AUTO` | yes |
| `CVE-2026-20469 A-518601919 M-AUTO00834868 * High` | `trusted_mem` | `M-AUTO` | yes |
| `CVE-2026-20473 A-518610752 M-ALPS11019722 * High` | `display` | `M-ALPS` | yes |
| `CVE-2026-20477 A-518606098 M-ALPS11009963 * High` | `display` | `M-ALPS` | yes |
| `CVE-2026-20481 A-518601923 M-ALPS10965373 * High` | `geniezone` | `M-ALPS` | yes |

四个读数要点：

1. **共用引用号、不共用 bug 号**：20501/20502 的 `M-ALPS11262030` 相同，`A-528211066` 与 `A-528209858` 不同 ⇒ 第廿三章之七"按引用号统计会低估"在这一行是**逐字成立**的（两条缺陷一个号）。
2. **代号与系列错配**：`trusted_mem` 走 `M-AUTO` 而不是 `M-ALPS`。⇒ **不能用 `Subcomponent` 反推系列，也不能用系列反推 `Subcomponent`**；两列都必须原样存，映射表要按第廿二章规则七长期维护。
3. **同概念两种大小写再次出现**：`display`（MediaTek）与 2026-06 页的 `Display`（该期严重度后一列令牌统计里 `Display` 2 行）。第 27 章脚本 1 的归一层按大小写不敏感处理，但 `subcomponent_raw` 保留原文。
4. **本域相关的行全表 100% 带星**（`M-` 系 23/23、`U-` 系 7/7）⇒ 组 7 的每一行都触发 `no_device_side_evidence`。这批条目在台账里的正确用途不是"修复跟踪"，而是**资产口径**：哪台设备的 SoC 属这一族、因此应该配一个厂商侧对齐动作。

### 廿四之十 组 8：`Modem` 族的对照——本域的外沿

`CVE-2026-20458 A-509305912 M-MOLY01402160 * High Modem`、`CVE-2026-20460 A-509305913 M-MOLY01811421 * High Modem`、`CVE-2026-21548 A-517353300 U-3293610 * High Modem`、`CVE-2026-21549 A-517316530 U-3379435 * High Modem`。

这里只登记三件事，缺陷判定一律归第 14 篇（无线与基带）：**（a）** `M-MOLY` 与 `M-ALPS` 是同一家厂商的不同系列，`U-` 是另一家；**（b）** 相邻两个 `A-` 号（`...912` / `...913`）对应两条不同缺陷，说明 `A-` 号不能按连续性猜内容；**（c）** 同一个 `Subcomponent` 词（`Modem`）跨两家厂商出现 ⇒ 按词聚合资产会把两家合并，这是第廿五章归属矩阵必须按表名分列的原因。

### 廿四之十一 组 9：高通的两副面孔——`Video` 有词无星，`Security` 有星无词

跨期对照（D-10）：

- 2025-09-01：`CVE-2025-27042 A-409047250 QC-CR#3935673 High Video`、`CVE-2025-27043 A-409047487 QC-CR#3935643 High Video`（该期 `High \| Video` 共 2 行）
- 2026-09-01：`Qualcomm components` 表 5 行全 `Security`，逐字 `CVE-2026-24087 A-510336694 QC-CR#4409501 High Security`、`CVE-2026-24088 A-489081076 QC-CR#4409568 …`，**全部无星**；表句是 `... are described in further detail in the appropriate Qualcomm security bulletin or security alert.`（与别家 `available directly from X` 不同形）

读数：这是全篇唯一一个"厂商号齐全且不带星"的族——`QC-CR#` 5/5 无星，与 `M-`/`U-` 系全带星形成完整对照。含义是双面的：

1. 好消息：这一族的缺陷信息在厂商自己的公告里可指认（`QC-CR#` 是可用外键），第廿三章之五说的"按厂商引用号反查器件"对高通成立、对 GPU 不成立（组 5、组 6）。
2. 坏消息：**词不可映射在同期发生**。2026-09 的高通表里 `Video` 与 `Display` 一个都没出现（第五章已记），本期高通行全落 `Security`。也就是说"这期的缺陷在哪个引擎"从公告读不出来，只能读厂商公告。而第九章实测：高通 `www` 是 SPA 外壳、`docs` 侧任意路径返回同一个 52,406 字节的壳页 ⇒ **外键指向的入口本次不可读**。于是本域最"可反查"的一族，恰恰在反查终点取不到正文。

台账动作：`vendor_ref_family = QC-CR#` 的行必须配一个 `vendor_entry_fetch = pending`，且 `no_device_side_evidence` 不适用于它（它有外键、缺的是入口可达性，两类缺口不能混记）。这条区分在第廿九章误区清单里会以"把取不到正文写成不可修"的形式出现。

### 廿四之十二 组 10：`Qualcomm closed-source components` 的 11 行——占位串 + 无厂商号 + Critical

逐字：`CVE-2026-25289 A-509524204 * Critical Closed-source component`、`CVE-2026-21383 A-473523681 * High Closed-source component`、`CVE-2026-24079 A-480963580 * High Closed-source component`；该期该表 11 行，Subcomponent 全为占位串。表句 `These vulnerabilities affect Qualcomm closed-source components and are described in further detail in the appropriate Qualcomm security bulletin or security alert.`

读数三连：`vendor_ref_family = none`（**闭源表里反而没有 `QC-CR#`**）、`asterisk = yes`、`Type` 缺位。⇒ 这是本域公开侧信息量最少、而设备侧恰恰最相关的一批行（闭源组件 = 镜像本体）。第廿三章之六把它判给 `no_device_side_evidence`，第卅章把它列进"未公开文档化边界清单"，第廿五章的归属矩阵里它只能落进"不可归因"列。

台账动作：占位串行**不进**任何按器件聚合的统计（第廿二章之十第 2 条）；但必须进 K5/K10 的分母，否则"信息最少的一批"会从指标里消失——那正是第廿九章要写的误区之一。

### 廿四之十三 组 11：单行厂商表与句式的单复数——`SUW` 与 `StrongBox`

逐字：`CVE-2026-28616 A-483923525 * High SUW`，表句 `This vulnerability affects Setup Wizard components and further details are available directly from Setup Wizard. The severity assessment of this issue is provided directly by Setup Wizard.`；`CVE-2025-48651 A-529876547 * High StrongBox`（`Tsingteng Micro`）。

读数是方法层的：这两个表的句子用 **`this issue`（单数）**，与六张多行表的 `these issues` 不同。D-10 里那张"厂商定severity表数"的跨期对照，第一次统计时就是因为按整句匹配而在 2025-09 页漏掉 Arm 表（该句在 HTML 里被换行切断）——压平空白后才得到正确值（6 / 5 / 5 / 0）。⇒ 第廿二章规则二（子串≠词命中）的**加强版**：句式模板匹配既不抗换行，也不抗单复数。任何"按表句识别厂商表"的解析器，都必须先用两种单复数 + 压平空白，否则单行厂商表整批漏掉。

对本域：`SUW`、`StrongBox` 都落在六层之外（第 5 篇密钥栈、应用侧），此处只作解析器反例登记。

### 廿四之十四 组 12：`*` 族的整体读数——T2 与送达侧的接缝

把本章十三组里所有星号合起来，与第廿三章之六的分布对齐：2026-09-01 页 72 个 `href="#asterisk"` 锚点；按相邻令牌归属 `A-` 44、`M-MOLY` 10、`M-ALPS` 10、`U-` 7、`M-AUTO` 3、`QC-CR#` 0（按前一行归属法求和为 74，与 72 差 2，来自同格两个编号共用一个星；本章只用族级分布，不用逐条归属）。

送达侧的逐字依据（FAQ 第 5 问）：`The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.`

⇒ 这是本域"披露通道"与"送达通道"唯一被官方句子正面连起来的地方：一条带星的器件条目，其修复载具是 **binary driver**，而不是 AOSP commit。它解释了为什么第廿三章要把它单独列为形态，也解释了第 8 章场景 B（SPL 未更新而器件已更新）为什么会常态存在。

台账动作：带 `*` 的媒体/加速器条目，送达侧证据只允许三种取值——`supplemental_xml`（组 13）、`vendor_ota_manifest`（厂商自己的发布清单）、`absent`。`spl_advanced` 明确不允许作为送达证据（L-3 条）。

### 廿四之十五 组 13：T3 记账例外通道——本域唯一可读的器件修复证据

两处官方自述，逐字已在第八章之三与第三之五节登记，本章只把它们当作案例读数的落点重述：

1. **补充补丁清单**：`The supplemental security patch XML feature lets OEMs report CVEs patched beyond the declared security patch level (SPL) by providing a standardized XML file, so OEMs receive credit for continuous patching.`；放置与聚合规则逐字：`OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product), ensuring that security patches from both the framework and hardware-specific components are accounted for. The XML file should be installed in /etc/security/supplemental_security_patches.xml for the respective partition.` 与 `KEY_VENDOR_SUPPLEMENTAL_PATCHES: Aggregates patches from /vendor and /odm partition paths.`
2. **Vulkan APEX 例外**：`Since Android 15, the loader supports APEX to load the Vulkan driver. Set ro.vulkan.apex to the name of the Vulkan APEX to load the Vulkan from the APEX.`——第三之五节已判定：它更新的是 F4 用户态驱动，不是 F1 命令流固件。

读数：这是本章十三组里唯一一组**不是从公告页读来的**。它给出的可核验形态是：某台设备上是否存在 `/vendor/etc/security/supplemental_security_patches.xml`，以及其中是否列有本域条目（`hevc decoder`、`vdec`、`apusys`、`Mali`、`PowerVR-GPU`、`Closed-source component`）。第廿七章脚本 1 把"该文件是否存在 + 是否可读"作为一条独立分支（P-11 位）；第廿八章脚本 2 把它作为漂移比对的一类伴读。

台账动作：`asterisk = yes` 且 `fw_class = yes` 的行（组 5、6、7、10 全部），若在设备侧读到该 XML 且其中含该编号 ⇒ 送达证据从 `absent` 升为 `supplemental_xml`，判定升一级（第廿六章）；若读不到该文件 ⇒ 保持 `no_device_side_evidence`，并**不得**把"文件不存在"记成"未修复"（第廿二章规则八）。

### 廿四之十六 十三组的通道分布，与四条禁止推出

| 通道 | 组 | 条目数（本页实测） | 设备侧可验证上限 |
| --- | --- | --- | --- |
| T1 | 1、2、3、4 | 平台表行（有 `Type` + 版本列），本域相关者如 `CVE-2026-28581`、`CVE-2026-31629` 族、Mainline 段 | SPL / Play 更新串（间接） |
| T2 | 5、6、7、8、9、10、11 | Arm 6、Imagination 25、MediaTek 23（含 Modem）、Unisoc 7、Qualcomm 5、QC 闭源 11、单行厂商 2 | 多数为**无**；`QC-CR#` 族有外键但入口本次不可读 |
| T3 | 12、13 | 星号族 72 个锚点（跨全部 T2 表）+ 两处官方机制 | 一个可读文件 + 一个 property |

四条禁止推出（与第廿二章之十同向，这里补通道维度）：

1. **不得由"组 5、6 无厂商号"推出 GPU 侧不可跟踪**——可跟踪性在厂商入口那一侧（Imagination 可读、Arm 需二次验证），公告侧的外键缺失只是链条的第一段。
2. **不得由"组 10 全占位串"推出闭源组件无法更新**——第 7 章问三已裁决；正确推论是"更新存在但不可归因"，落 T3 通道找证据。
3. **不得由组 9 的 `QC-CR#` 无星推出高通条目都可公开核验**——同一族在第九章的入口实测里是取不到正文的。
4. **不得把本章十三组当风险排序**：它的密度由入口可读性与词可映射性决定；T2 组条目多，只说明这些厂商愿意在公告里放表，不说明它们的器件更安全或更危险。

### 廿四之十七 本章结论

1. **列形状就能分箱**：`Type` + `Updated AOSP versions` 的有无，把 T1 与 T2 分开，不需要读正文。这是本章唯一可以自动化执行的结构判据，第廿七章脚本按它写分支。
2. T2 的表**不提供缺陷类别**，所以第廿一章 FT1–FT10 不能由公告反向构建；凡公告侧出现的类别线索只有 `Subcomponent` 词，而词与器件的映射是长期维护表（第廿二章规则七）。
3. 严重度读数的口径前提是"缓解视为关闭"（逐字），因此严重度必须与第廿章的姿态表读数同表使用，单独引用严重度即失真。
4. 星号（`*`）把披露侧与送达侧连起来：带星条目的修复载具官方原句是 `binary drivers`。本域带星比例极高的媒体/加速器族（`M-` 23/23、`U-` 7/7、Arm 6/6、Imagination 25/25、QC 闭源 11/11）因此一律 `no_device_side_evidence`，除非命中 T3 证据。
5. T3 是本域唯一能把器件修复读出来的地方，而它是 OEM 自愿的：`supplemental_security_patches.xml` 的有无，直接决定这台设备的器件修复史是否可归因——脚本 1 必须把它当一等分支，而不是可选附加项。
6. 本域公开侧信息量最少、设备侧最相关的一批行是 `Closed-source component` 11 行（无 `Type`、无厂商号、全带星、含一个 Critical）。它们必须进分母，不能进器件聚合。
7. 第廿五章把本章的组按五个工单去向重排（厂商固件团队 / 驱动维护者 / 发布流程 / 本机构采集取证 / 安全负责人与资产口径），并给出能力事实表；那里使用的行全部来自本章 D-10，不再引入新页。
## 廿五、归属矩阵与能力事实表：五个工单去向 × 十类缺陷 × 十三组案例

第廿一章给出十类缺陷与"类型 → 工单"的九句映射，第廿四章给出十三组公开案例的通道分箱。本章把两者接起来：**同一批行，按"这张工单该派给谁"重排一次**，再补上第四件东西——**能力事实表**：本域那些写侧节点、可改装载对象的输入、取证出口与策略门，逐条登记"它存在、它的模式是多少、谁能改"，**不登记任何用法**。

**视角声明**：本章不新增任何来源。公告侧的行全部取自第廿四章登记的四个本地副本（`.sb-2026-09-01.html`、`.sb-2026-06-01.html`、`.sb-2025-09-01.html`、`.sb-2026-10-01.html`，登记于 D-10）；源码侧的行全部取自 D-1、D-3、D-7 已登记的本地副本，以及本轮为核对框架节点补取的 `drivers/remoteproc/remoteproc_sysfs.c`、`remoteproc_debugfs.c`、`drivers/rpmsg/rpmsg_core.c`、`rpmsg_ctrl.c`（两个分支各一份，登记于 D-11）。

**能力事实的边界（这条比内容重要）**：本章列出的每一个写侧节点、模块参数、ioctl 编号与 sysfs 属性，**只作为"这个面上存在这样一个可写对象"的事实登记**。给出的是名字、八进制模式位与声明它的源码位置；不给打开方式、不给写入格式、不给调用序列、不给任何能改变固件内容或装载结果的操作步骤。凡是读者需要"怎么做"才能验证的条目，一律不在本报告范围内——设备侧核验在第廿七、廿八章以**只读**脚本给出，且脚本本身不含写动作。

### 廿五之一 五个去向的定义：每个去向能改什么，改不动什么

工单派错人是因为台账只记症状（第廿一章之约束二）。要按"谁能改"派单，先把五个去向的职责边界写成可判定的句子。

| 去向 | 负责的对象 | 能改的 | 改不动的（必须升级到谁） | 工单必须附带的字段 |
| --- | --- | --- | --- | --- |
| **O1 厂商固件团队** | F1 镜像本体（命令流固件、DSP 固件、ISP 固件）与器件侧解析器 | 固件内部解析逻辑、固件版本号语义、固件与驱动的接口版本 | AOSP 侧任何代码；公告词表；设备侧读数出口 | `fw_class`、`vendor_ref_family`、`bulletin_version` |
| **O2 驱动维护者** | F3 主机侧驱动（装载、解析、闸门、能力协商） | `gate_kind`、`bounds_prechecked`、日志级别、属性可见性 | 固件内容；厂商 `vendor_file_contexts`；公告 Subcomponent 词 | 源码行号、`gate_kind`、`parse_side` |
| **O3 发布流程** | F6 送达与记账（OTA 清单、SPL、补充补丁 XML、Mainline/APEX 通道） | 是否发布 `supplemental_security_patches.xml`、送达证据形态、发布节奏 | 缺陷本身；公告是否给厂商引用号 | `delivery_evidence`、`spl`、`asterisk` |
| **O4 本机构采集与取证** | 读数面（V1–V8）、脚本分支、日志与转储面 | 采集姿势、伴读字段、`unavailable_reason` 归因、指标分母 | 内核代码；策略；厂商是否公开信息 | 采集姿势 ID、`verdict`、`unavailable_reason` |
| **O5 安全负责人与资产口径** | F5/F6 的口径层：公告词 ↔ 器件映射、风险接受、跨厂比较是否允许 | 判定口径、接受风险声明、禁止跨厂比较的规则 | 任何技术读数 | `mapping_class`、`disclosure_tier`、`risk_accepted` |

这五行里有三条是**故意写成否定形式**的：O1 改不动 AOSP、O2 改不动厂商标签、O3 造不出缺陷。派单冲突几乎全部落在这三条上——把一个 FT4（比较语义错配）派给 O1，等于要求厂商改一个由主机驱动解释的字段；把一个 FT9（归属脱钩）派给 O2，等于要求驱动维护者去改公告词表。

### 廿五之二 主矩阵：十类 × 五去向

记号：**●** 主责（工单归它，验收也归它）；**○** 协办（需提供字段但不接工单）；**–** 不适用。

| 类型 | O1 厂商固件 | O2 驱动维护 | O3 发布流程 | O4 采集取证 | O5 口径与资产 |
| --- | --- | --- | --- | --- | --- |
| FT1 镜像解析器边界 | ●（器件侧解析） | ●（主机侧解析） | – | ○（`parse_side` 读数） | – |
| FT2 装载决策受非授权输入影响 | ○（命名规则） | ●（参数与属性的暴露面） | ○（分区只读与否） | ●（stat 模式 + 标签伴读） | ○（是否接受该风险） |
| FT3 版本闸门缺失或只警告 | ○（需给出期望下限） | ● | – | ○（`gate_kind` 判定要用到日志面） | ○（无公告边界时转风险接受） |
| FT4 版本比较语义错配 | ●（字段编码规则变更方） | ●（比较实现方） | ○（跨期台账） | ○（读数原样留存） | ●（禁止跨厂比较的规则在这里） |
| FT5 静默降级 | ○（能力位来源） | ●（不打印的是驱动） | – | ●（功能反证覆盖） | ○ |
| FT6 跨核输入直接变主机状态 | ●（对端填写串） | ●（主机使用串） | – | ●（`value_source` 伴读） | ○ |
| FT7 能力位决定路径 | ●（声明与实现一致性） | ●（把声明当可信开关） | – | ○ | ●（能力位不得升到 L2 以上） |
| FT8 取证与转储面自身暴露 | – | ○（节点模式） | ○（构建类型） | ● | ●（转储内容是否算敏感数据） |
| FT9 归属层与对象层脱钩 | ○ | – | ○ | ○ | ● |
| FT10 可观测性缺陷 | ○（镜像内字段是否导出） | ○（有无出口） | – | ● | ○（指标分母在这里） |

三点要从这张矩阵读出来：

1. **只有 FT8、FT9、FT10 的主责是纯内部的**（O4/O5），它们对应第廿一章约束三说的"关于可观测性本身的缺陷"。这三类的工单不需要厂商回信就能关单——这也是为什么把它们混进厂商工单批次会让整个批次的周期变成"等对方"。
2. **FT2、FT4、FT6 各有两个 ●**。双主责不是笔误：FT2 的输入面在驱动、命名规则在厂商；FT4 的编码规则在厂商、比较语义在驱动；FT6 的串由对端填、由主机用。这类条目的工单必须**成对开**，且规定"任一侧单独关单不视为修复"。第廿九章误区十二条里有一条专门写这个。
3. **O3 一行只有两个 ●、四个 ○**——发布流程不是缺陷的责任方，它是**证据的责任方**。这条决定了第廿六章的判定模型里"送达证据"只能由 O3 出具，O1 与 O2 出的证据不算送达。

（第廿一章之十三第 5 条把 FT1/FT7 判给厂商固件团队、FT2/FT3/FT4 判给驱动维护者与发布流程、FT5/FT8/FT10 判给本机构采集、FT9 判给安全负责人，**唯独 FT6 没有落到具体去向**。本章把它判给 O1● + O2● + O4●，理由在这一节的第 2 点：三侧各持一半，任何单侧派单都会漏掉"串的来源"这一半。）

### 廿五之三 派单判据：症状 → 类型 → 去向，三段都要写

台账最常见的形态是"旧镜像仍在用"这类症状句。第廿一章约束二给出这条判据在这里落成三个可执行的例子——同一条症状，三种成因派三个不同去向。

**症状甲：某台设备的器件补丁状态与 SPL 不一致。** 三种成因与三个去向：
- 厂商发布了带外更新但没写 XML（FT9 + FT10 的送达侧）→ **O3**，附 `delivery_evidence = absent`。第廿四章组 13 给出的判据句逐字：`OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product)`——能不能读到这个文件，是 O3 的选择，不是 O1 的缺陷。
- 驱动有警告型闸门、旧镜像照样装载（FT3）→ **O2**。实例已在第廿一章钉死：`sh_css_check_firmware_version()` 返回 `bool` 且唯一出口是 `return false;`（`sh_css_firmware.c:188-189`、`:209`，android15-6.6）。
- 台账把 HAL 版本当成了固件版本（FT4 + 主键误传）→ **O5**。第廿三章之八把这条列进六条主键误传，判据是 `ro.vulkan.apex` 更新的是 F4 用户态驱动，不是 F1 命令流固件。

**症状乙：解码行为变了，没人知道为什么。** → 若降级无日志（FT5）派 **O2 + O4**；若厂商换了固件版本而台账无版本字段（FT4）派 **O1 + O5**。区分句是"能不能在设备上指出一条打印"。`vdec.c:577`、`:711` 两个门都没有打印，属于前者；`mtk_vpu.c:628` 的 `vpu is ready. Fw version %s` 有 INFO，属读数面问题，属 O4。

**症状丙：sysfs 里某个属性变成只读了。** → **不派单，先归因。** 第廿章 P-15 给出：stat 到 `0444` 的唯一可判读含义是"有本次未取回的厂商补丁在生效"（`rproc_is_visible()` 在 `rproc->sysfs_read_only` 为真时降权四个可写属性，`remoteproc_sysfs.c:229-243`、`:236-239`；本树该字段赋值点检索为 0）。这类条目落 `degraded_by_vendor`，工单去向是 **O5（口径）**，不是 O2——驱动没做错任何事。

### 廿五之四 十三组案例按去向重排

第廿四章的十三组，一行一组。"上限"列给出**只靠这组证据能到的判定级**（第廿六章模型），超过上限的推断都是台账自己编的。

| 组 | 条目（D-10 逐字行的代表） | 主去向 | 协办 | 台账动作 | 上限 |
| --- | --- | --- | --- | --- | --- |
| 1 平台表媒体/框架行 | `CVE-2026-28581 A-479515555 ID High 14, 15, 16, 16-qpr2, 17` | O3 | O5 | 按 `Type` + 版本列进平台台账，不写器件 | L1 |
| 2 `Kernel` 段 | `CVE-2026-31629 A-499350302 Upstream kernel [ 2 ] EoP Critical NFC` | O2 | O5 | 引用形态 `Upstream kernel [ N ]` 单列一字段，不套前缀表 | L1 |
| 3 `Google Play system updates` 段 | `CVE-2026-28593 A-388272624 EoP High 14, 15, 16, 16-qpr2, 17` | O3 | O5 | 更新语义第三种，互链第 16 篇；不进 SPL 判断 | L1 |
| 4 同页双现 | 195 槽位 / 175 编号；`CVE-2026-28581` 两处偏移 | O5 | O4 | 主键必须是"编号 + 表名"复合，槽位数不进覆盖率分母 | 不适用 |
| 5 `Arm components` Mali 六连 | `CVE-2026-0001 A-466377983 * High Mali` | O1 | O4 | 一词三部件（JM/CSF、驱动、固件）按 `Mali` 覆盖并声明不确定性 | L0 |
| 6 `Imagination` 25 行 | `CVE-2026-21732 A-468248021 * High PowerVR-GPU` | O1 | O5 | DDK 版本与设备侧读数都拿不到，禁做受影响范围推断 | L0 |
| 7 MediaTek 媒体族 | `CVE-2026-20501 A-528211066 M-ALPS11262030 * High vdec`、`CVE-2026-20467 … M-AUTO00837766 * High apusys`、`CVE-2026-20469 A-518601919 M-AUTO00834868 * High trusted_mem` | O1 | O3、O5 | 两 CVE 共用一条 `M-` 号 ⇒ 厂商号不能当主键；代号与系列错配 ⇒ `vendor_ref_series` 单列 | L0 |
| 8 `Modem` 族（本域外沿） | `CVE-2026-20458 … M-MOLY01402160 * High Modem`、`CVE-2026-21548 … U-3293610 * High Modem` | O5 | – | 只登记"与本域相邻但不属本域"，不派 O1 | 不适用 |
| 9 高通两副面孔 | `CVE-2026-24087 A-510336694 QC-CR#4409501 High Security`（无星）、2025-09 的 `CVE-2025-27042 A-409047250 QC-CR#3935673 High Video` | O2 | O5 | `Video` 有词无星 ⇒ 可下钻；入口实测取不到正文 ⇒ 记 `no_device_side_evidence` | L0 |
| 10 QC 闭源 11 行 | `CVE-2026-25289 A-509524204 * Critical Closed-source component` | O3 | O5 | 占位串行不进器件聚合，**必须**进 K5/K10 分母 | L0 |
| 11 单行厂商表 | `CVE-2026-28616 A-483923525 * High SUW`、`CVE-2025-48651 A-529876547 * High StrongBox` | O4 | – | 解析器必须同时匹配 `this issue` 与 `these issues` 且先压平空白 | 不适用 |
| 12 `*` 族的整体读数 | 72 个 `href="#asterisk"` 锚点；族级分布 `A-` 44、M-MOLY 10、M-ALPS 10、`U-` 7、M-AUTO 3、`QC-CR#` 0 | O3 | O1 | 带星且属器件 ⇒ `verification_status = no_device_side_evidence`，直到命中 T3 | L0 |
| 13 T3 记账例外 | `/vendor/etc/security/supplemental_security_patches.xml`、`KEY_VENDOR_SUPPLEMENTAL_PATCHES`、`ro.vulkan.apex` | O4 | O3、O5 | 脚本 1 的一等分支（P-11）；读到即升一级，读不到不得记"未修复" | L2 |

按主责数一下负载：**O1 三组、O2 两组、O3 四组、O4 三组、O5 一组**。这个分布本身就是结论——十三组公开案例里，能直接派给厂商的只有三组，而能派给自己（O3/O4/O5）的有八组。第廿九章误区章节把它写成一条：**"公告读得越多越安心"是错的，多数读数改进的是自己的台账质量，不是器件安全状态。**

### 廿五之五 七条链按去向的派单负载

链的侧写来自第十至廿一章，这里只按"这条链上开出的工单主要落在哪个去向"重排一次。

| 链 | 主去向 | 协办 | 判据（已登记的行） |
| --- | --- | --- | --- |
| Mali CSF | O1 | O4 | 能力位与接口版本由固件写入共享内存（`panthor_fw.h:193-202`）；无主机侧期望下限 |
| Adreno GMU/KGSL | O1 | O2 | 版本取回即丢（`a6xx_hfi_get_fw_version()` 以 `NULL` 接收）；闸门本身是反例（`a6xx_gpu.c:772-777`、`:800-806` fail-closed） |
| Exynos MFC | O2 | O4 | 装载对象由目录内容决定（`s5p_mfc_ctrl.c:63-72`）；版本只到 `mfc_debug(2, …)`（`:286-287`），开关是 `debug` 参数 0644（`s5p_mfc.c:39`） |
| QCOM Venus | O2 | O3、O4 | 版本串解析错配（`hfi_msgs.c:304-330`）；两个静默门（`vdec.c:577`、`:711`） |
| MTK VPU + SCP | O2 | O1、O4 | 双名回落（`mtk_vpu.c:512`/`:517`）；SCP 名走 DT（`mtk_scp.c:849` → `remoteproc_core.c:1002`）；名称服务串入目录（`mtk_rpmsg.c:259`、`:298-300`） |
| 相机 ISP | O2 | O1 | 校验链最完整的两处都在主机（`ipu3-css-fw.c:129-134`、`ivpu_fw.c:112-164`）；atomisp 闸门恒假（`sh_css_firmware.c:209`） |
| NPU / APU | O1 | O5 | `qaic` 主机侧无镜像（全树 `request_firmware` / `MODULE_FIRMWARE` / `.bin` 0 命中，`firmware` 只出现在 `qaic_drv.c:272-281` 的注释里）、协议硬闸门建立在设备自报版本上（`qaic_drv.c:296-302`）；服务属性缺 `protected_service`（`public/service.te:343`） |

O1 只在三条链上是主责，且这三条的共同点是**主机侧根本没有可改的解析器或闸门**（Mali、Adreno、qaic 的镜像都经安全侧认证或直接驻留器件）。O2 在四条链上是主责，四条链的共同点是主机里有 `request_firmware`/`rproc` 装载链。**派单去向与装载路径分型是同一条线**——这不是巧合，是第三章给出的分型在工单层的结果。

### 廿五之六 能力事实表 A：能改变"装载哪一个镜像"的输入

只登记三件事：**对象名、声明它的源码位置、它在源码里的权限位**。权限位是源码声明值，运行期实际值必须由采集侧 `stat` 得到（第廿十章 P-16）；两者不一致时以 `stat` 为准。

| 对象 | 源码声明 | 位置（分支见 D-1/D-3/D-7/D-11） | 登记要点 |
| --- | --- | --- | --- |
| `firmware`（remoteproc 属性） | `static DEVICE_ATTR_RW(firmware);` | `remoteproc_sysfs.c:167` | 该总线唯一能给 rproc 换镜像名的属性；值继承默认 `sysfs` 标签，shell 读不到（P-14/P-15） |
| `recovery` / `state` / `coredump` | `DEVICE_ATTR_RW` ×3 | `remoteproc_sysfs.c:73`、`:217`、`:135` | 三个可写属性，与 `name`（RO，`:227`）同组注册（`:245-252`、`:256`） |
| `rproc->sysfs_read_only` 降权 | `if (rproc->sysfs_read_only && (attr == &dev_attr_recovery.attr \|\| …))` | `remoteproc_sysfs.c:229-243`、`:236-239` | 本树赋值点 0 处 ⇒ stat 到 `0444` 是"有未取回厂商补丁"的一等信号 |
| `firmware`（ivpu 模块参数） | `module_param_named_unsafe(firmware, ivpu_firmware, charp, 0644)` | `ivpu_fw.c:42-44` | `unsafe` 变体 + `0644`；参数优先于代号回落表（`:67-71`） |
| `skip_fwload` | `module_param(skip_fwload, uint, 0644)` | `atomisp_v4l2.c:58-60` | 能把整条装载链关掉；关闭不产生失败记录（FT5 用） |
| `firmware_name` | `module_param_string(firmware_name, firmware_name, sizeof(firmware_name), 0)` | `atomisp_v4l2.c:77-79` | 与上一行**同机制、权限 0**，构成"同类输入、不同暴露"的对照样本 |
| `debug`（MFC 调试级） | `module_param_named(debug, mfc_debug_level, int, 0644)` | `s5p_mfc.c:39`（android16-6.12） | 0644 的日志开关 ⇒ FT10 的成因归 `build` 还是 `policy` 要伴读构建类型 |
| `dbg_mask` / `test_mode` 等 ivpu 参数 | `module_param_named_unsafe(…, 0644)` 五条 | `ivpu_drv.c:39-56` | 五条同模式；`test_mode` 与固件装载无关，登记但要标"不在装载链" |
| DT 属性 `firmware-name` | `rproc_of_parse_firmware(dev, 0, &fw_name)`；默认 `"scp.img"` | `mtk_scp.c:849`、`:846`；实读 `remoteproc_core.c:1002` | 输入源是镜像/DT 分区，不是运行期可写面；分区只读时可达性记 `absent` |
| 文件系统命名（回落表） | 日期长名优先 `-ENOENT` 回落；代号表六条；MFC 降序两候选 | `ipu3-css-fw.c:120-124`；`ivpu_fw.c:46-56`；`s5p_mfc_ctrl.c:63-72` | 装载对象由盘上文件存在性决定 ⇒ FT2 的一类；`MFC_FW_MAX_VERSIONS` 为 2（`s5p_mfc_common.h:40`） |

**这张表不回答"能不能改"**。它回答的是"改装载对象有几条路、每条路的权限位是多少"。第廿七章脚本按最后一栏采集 stat 值，第廿八章比对模式漂移。

### 廿五之七 能力事实表 B：取证与调试出口

| 节点 | 声明模式 | 位置 | 携带什么（只描述类别） |
| --- | --- | --- | --- |
| `name` / `resource_table` / `carveout_memories` | `0400` ×3 | `remoteproc_debugfs.c:407`、`:413`、`:415` | 协处理器名称与**内存布局、共享区名称**（比 sysfs 更实质，FT8） |
| `recovery` / `coredump` | `0600` ×2 | `remoteproc_debugfs.c:409`、`:417` | 与 sysfs 同名属性的调试侧对应物 |
| `crash` | `0200` | `remoteproc_debugfs.c:411` | 只登记存在与模式；不给任何触发动作 |
| debugfs 根缺失 | `if (!rproc_dbg) return;` | `remoteproc_debugfs.c:402`；根目录 `:424` | **节点不存在 ≠ 厂商加固**，记 `debugfs_unreachable` 或 `muted_by_build` |
| `coredump` 的 sysfs 侧 | `DEVICE_ATTR_RW(coredump)`，实现只接受三个词 | `remoteproc_sysfs.c:135`、`:111-134`；读侧串表 `:80-84` | 自带文档注释（`:95-110`）里的 `default` **不是合法值**；`:86` 注释写 debugfs 实为 sysfs；崩溃态改配置 `-EBUSY`（`:117-120`） |
| `fw_log` | `0644`，`fops` 同时有 `.write` 与 `.read = seq_read` | `ivpu_debugfs.c:144-148`、`:117-124` | 固件日志缓冲的读写面，登记存在 |
| `force_recovery` / `reset_engine` / `fw_trace_destination_mask` / `fw_trace_hw_comp_mask` / `fw_trace_level` | `0200` ×5 | `ivpu_debugfs.c:273-293` | 五个写侧节点；本报告不给出任何一个的使用方式 |
| `fw_name`、`bo_list`、`last_bootmode`、`reset_counter`、`reset_pending`、`fw_trace_capability`、`fw_trace_config` | debugfs 表项 | `ivpu_debugfs.c:107-115`；`fw_name_show()` `:30-36` | **`fw_name` 给的是主机侧请求的文件名，不是固件自报版本**——这一区分是 K8 的口径线 |
| `dbgfun` / `dbgopt` | `__ATTR(…, 0644, …)` ×2 | `atomisp_drvfs.c:160-161`；经 `driver_create_file()` `:164-172` | 落点 `/sys/bus/pci/drivers/atomisp-isp2/`；`OPTION_BIN_LIST` 会转储 blob 信息（`:62-63`），`OPTION_BIN_RUN` 需 streaming 否则 `-EPERM`（`:71-80`） |
| Venus 版本的两条出口 | `dev_dbg` 与一条 SMEM 表项写入 | `hfi_msgs.c:322-323`、`:323-330`（android15-6.6；SMEM 宏 `:18-20`） | 都不是 sysfs 节点：一个是日志级别，一个在 shell 域不可读的共享内存表里；`venus_ver` 只是内核结构体成员（`core.h:233-237`） |
| MFC 寄存器版本唯一出口 | `mfc_debug(2, "MFC F/W version : %02xyy, …")` | `s5p_mfc_ctrl.c:281-287`（android16-6.12） | 受 `debug` 参数门控，与上一条同属 FT10 |

### 廿五之八 能力事实表 C：跨核通道与用户态入口

| 对象 | 位置 | 登记事实 | 明确不登记 |
| --- | --- | --- | --- |
| rpmsg `driver_override` | store 函数签名跨 `rpmsg_core.c:424-427`（android16-6.12；android15-6.6 同函数起于 `:422`）、`static DEVICE_ATTR_RW(driver_override);` 在 `:449` | 该总线**唯一**可写属性；服务名匹配是 `strncmp(id->name, rpdev->id.name, RPMSG_NAME_SIZE)`（`:480`），override 路径是整串 `strcmp`（`:491-492`）；注册路径把传入串写进 `rpdev->id.name`（`:617-618`，`dev_set_name` 在 `:620-621`） | 任何写入值样例 |
| rpmsg 名字宽度 | `#define RPMSG_NAME_SIZE 32`（`mod_devicetable.h:465`）、`struct rpmsg_channel_info`（`include/linux/rpmsg.h:34-35`） | 32 字节含结尾 NUL 实为 31 ⇒ 截断是一等状态 | — |
| 名服务报文 | `struct rpmsg_ns_msg`（`mtk_rpmsg.c:45-48`），端点绑 `SCP_IPI_NS_SERVICE` = `0xFF`（`include/linux/remoteproc/mtk_scp.h:47`） | 名字**由远端填写**；三道检查（`:279-296`）后仍进目录命名 | — |
| 用户态造端点的入口 | `rpmsg_ctrldev_ioctl()`（`rpmsg_ctrl.c:74`，`.unlocked_ioctl` `:125`）三个动作 `RPMSG_CREATE_EPT_IOCTL`（`:93-95`）、`RPMSG_CREATE_DEV_IOCTL`（`:98-103`）、`RPMSG_RELEASE_DEV_IOCTL`（`:107-111`）；先 `copy_from_user` 再 `memcpy(chinfo.name, eptinfo.name, RPMSG_NAME_SIZE)` + 末位补 NUL（`:84-88`） | 用户态可把一个 ≤32 字节的名字交给内核设备模型；节点名是 `rpmsg_ctrl%d`（`:168`） | **禁止登记"可以打开并 ioctl"**（第廿章 P-20 的处置规则） |
| QAIC 管理入口 | `#define DRM_IOCTL_QAIC_MANAGE DRM_IOWR(DRM_COMMAND_BASE + DRM_QAIC_MANAGE, struct qaic_manage_msg)`（`include/uapi/drm/qaic_accel.h:384`）、`DRM_QAIC_MANAGE 0x00`（`:375`）、`QAIC_MANAGE_MAX_MSG_LENGTH SZ_4K`（`:17`） | 存在一条用户态到设备控制通道的 ioctl；`get_cntl_version()` 由设备回读 major/minor 与 `status_flags` BIT(0)（`qaic_control.c:1382-1391`） | 报文构造、调用序列 |
| glink 边名 | 沿用第十八章的读数（`qcom_glink_native.c:1846-1858`、属性表 `:1860-1863`、`qcom_glink_create_chrdev()` `:1877-1898`） | 高通侧名字来自 DT 与内核，不来自固件 ⇒ 与 MTK 相反的 `value_source` | **本行曾标"来源待重取"，现已销账**：文件在 `kernel/common@android16-6.12:drivers/rpmsg/`（1999 行），先前按 `drivers/soc/qcom/` 找才得到 404，取回登记见第卅章 **D-13**；缺的是脚本侧格子（第廿八章之十），不是来源 |

这张表的用途只有一个：**给 FT6 的来源标签提供可伴读的对象清单**。凡 V4（rpmsg 名字面）出现读数，`value_source` 必须在这三行里选一个（`fw_reported` / `driver_set` / `userspace_override`），选不出就记 `unknown` 并降证据强度（第廿二章规则三）。

### 廿五之九 能力事实表 D：策略侧的门（决定"能不能采到"）

| 门 | 原文与位置 | 对采集的后果 |
| --- | --- | --- |
| `allow shell sysfs:dir r_dir_perms;` | `private/shell.te:443` | 目录可列，属性文件不可读——`shell.te` 内 `sysfs:file` 授权 **0 条** |
| `/sys/module/**` 只点名五条 | `private/genfs_contexts:182-186` | 其余继承默认 `sysfs`；模块参数面全靠兜底，`eacces_untagged` |
| 平台设备 sysfs 几乎不点名 | `private/genfs_contexts:125-190`，`/devices/platform/**` 只有 `:139` 一条（nfc） | MFC、venus、camss、remoteproc 的 sysfs 路径全部继承默认 `sysfs` |
| vendor 目录只给非 coredomain | `allow { domain -coredomain } vendor_file_type:dir r_dir_perms;`（`domain.te:191`）+ `typeattribute shell coredomain`（`shell.te:1`） | **能枚举 HAL 与能力声明，不能枚举固件镜像文件本身**（L-5） |
| 配置目录对所有域开放 | `domain.te:177-178`，标签 `file_contexts:427` | `/vendor/etc/**` 是本域唯一可列举的 vendor 子树 ⇒ T3 通道可读的原因在这里 |
| bugreport 侧静音 | `dontaudit dumpstate sysfs:file r_file_perms;`（`dumpstate.te:287`） | 取证路径**拒绝且不留痕**，FT8 的策略侧实例 |
| 属性面是过滤视图 | `property.te:191-198` 的 neverallow 只放行 `vendor_public_property_type` | `getprop` 是子集，缺项不得记"厂商没设" |
| 调试面的构建门 | `userdebug_or_eng`（`public/te_macros:614`）、`no_debugfs_restriction`（`:591`） | 节点不存在要先分 `muted_by_build` 与 `absent` |
| `/dev` 兜底 | `/dev(/.*)?  u:object_r:device:s0`（`file_contexts:97`）；`file_contexts` 检索 `accel` **0 命中** | `/dev/accel*` 是"未点名兜底"，与 `/dev/video*` 的"有类型无授权"（`device.te:47` + `file_contexts:221`）同一个 EACCES、两种成因 |
| vendor 侧标签为空 | `private/vendor_file_contexts` 在 `main` 分支不存在（`?format=TEXT` → 404 `NOT_FOUND`），vendor 侧唯一标签表 `vendor/file_contexts`（212 行）内 `firmware` 0 命中 | AOSP 不给 vendor 任何固件标签 ⇒ 本域厂商差异的一等来源 |
| 服务属性不对称 | `hal_camera_service` 带 `protected_service`（`public/service.te:311`），`hal_neuralnetworks_service` 不带（`:343`） | 只登记属性差异；该属性的强制后果不在取回文件内（边界项，第卅章） |

### 廿五之十 四张表共用的六条登记规则

1. **存在 ≠ 可用。** 表里每一行都只回答"这个面上有没有这样一个对象"。可达性由第廿章的 26 姿势表另列一栏，不在能力事实表里推断。
2. **模式取 stat，不取源码。** 源码声明的 `0644` 与设备上 `stat` 到的值可以不同（`rproc_is_visible()` 就是一个会改模式的代码路径）。两列都要留，缺一列就是第廿二章规则三违反。
3. **标签缺失是一等状态。** 采集到一个节点但没有 SELinux 标签可判读时记 `eacces_untagged`，不得合并进 `eacces`（第廿十章"同一种失败的五种成因"）。
4. **写侧只登记，不验证。** 本报告的所有核验动作都是读；任何写侧节点的"验证"都被记为不在范围内。第廿七、廿八章脚本里**不存在**对这些节点的 open-with-write 调用。
5. **构建类型必填。** `debugface_present` 为 `no` 时，`unavailable_reason` 必须在 `muted_by_build` / `debugfs_unreachable` / `absent` 三选一；留空即视为分支未覆盖。
6. **厂商补丁信号单列。** stat 模式与源码声明不一致、或出现本树无赋值点的降权，记 `degraded_by_vendor`，**不得**记成"厂商加固"或"驱动差异"。这条与第卅章的未公开文档化边界清单相接：`rproc->sysfs_read_only` 的赋值点就是清单里的一条。

### 廿五之十一 工单最小字段集

把第廿一章各 FT 的台账字段与第廿三章的十字段并集压成一张模板，按去向标出**必填**列（其他列可空但不得删）。

| 字段 | O1 | O2 | O3 | O4 | O5 |
| --- | --- | --- | --- | --- | --- |
| `cve_id_raw` / `cve_seq_len` | 必填 | 必填 | 必填 | 选 | 必填 |
| `bulletin_date` / `page_last_updated` | 必填 | 选 | 必填 | 选 | 必填 |
| `ref_slot_count` | 选 | 选 | 必填 | 选 | 必填 |
| `bug_id_raw` / `vendor_ref_family` / `vendor_ref_series` | 必填 | 选 | 必填 | 选 | 必填 |
| `asterisk` / `fw_class` | 必填 | 必填 | 必填 | 选 | 必填 |
| `page_entry_check` | 选 | 选 | 必填 | 必填 | 必填 |
| `parse_side` / `bounds_prechecked` / `integrity_gate_before_parse` | 必填 | 必填 | – | 选 | – |
| `name_source` / `input_writable_by_unpriv` / `override_exists` | 选 | 必填 | 选 | 必填 | – |
| `gate_kind` / `gate_source_verifiable` / `expected_floor_published` | 必填 | 必填 | – | 选 | 必填 |
| `version_semantics` / `comparable_within_vendor` / `cross_vendor_compare` | 必填 | 必填 | 选 | 选 | 必填 |
| `silent_capability_change` / `detectable_by_feature_probe` | 选 | 必填 | – | 必填 | – |
| `value_source` | 必填 | 必填 | – | 必填 | 选 |
| `capability_source` / `capability_gates_security` | 必填 | 必填 | – | 选 | 必填 |
| `debugface_mode` / `debugface_present` / `dontaudit_sensitive` | – | 选 | – | 必填 | 必填 |
| `mapping_class` / `unavailable_reason` | 选 | 选 | 选 | 必填 | 必填 |
| `disclosure_tier` / `delivery_evidence` / `risk_accepted` | 选 | – | 必填 | 选 | 必填 |

三条使用规则：**`cross_vendor_compare` 在所有列里只能是 `forbidden`**（第廿六章硬规则）；**`capability_gates_security` 在本域默认 `unknown`**，任何人把它填成 `yes` 必须附一条公开来源行号，否则按误填退回；**`delivery_evidence` 只允许 O3 出具**，取值域是 `supplemental_xml` / `vendor_ota_manifest` / `absent`（第廿四章之十五）。

### 廿五之十二 本章结论

1. **归属矩阵的形状是"内部三、外部七"**：十类里三类（FT8/FT9/FT10）主责在本机构，七类需要外部配合；十三组案例里八组的主责也在本机构（O3/O4/O5）。这两组数字合起来是本报告给运营方的核心结论——**本域第一年的工单量大部分不会到厂商手上**，而到厂商的那部分（O1 三组、三条链）恰恰是设备侧证据最少的。
2. **三类条目必须成对开单**（FT2、FT4、FT6），任一侧单独关单不算修复。这是矩阵里唯一一条"结构性双主责"，写进模板的规则就是"两侧工单互相 block"。
3. **FT6 的去向在第廿一章是缺的，本章补为 O1+O2+O4**。补的理由不是推测而是对象清单：表 C 列出串的三个可能来源各有代码位置，缺任何一侧都读不完整。
4. **能力事实表四张，共 40 行左右，全部只含名字、位置、模式与"携带什么类别的信息"**。这四张表不回答问题"能不能改固件"，它们回答问题"改装载对象的输入面有几条、取证出口有几条、策略把哪几条关了"。这是第廿七章脚本的分支清单来源。
5. **表 B 与表 D 的交叉是 FT8 的实际口径**：`debugfs` 六个节点的模式（表 B）× `dumpstate.te:287` 的 `dontaudit`（表 D）决定同一件事——bugreport 里能不能看到协处理器内存布局。两者都要采，单采一个就会得出错误结论（只看 debugfs 模式会以为 bugreport 里有）。
6. **第卅章要沿用的两条边界在本章出现**：`protected_service` 的强制规则不在取回文件内；`rproc->sysfs_read_only` 无赋值点。前者影响表 D 最后一行，后者影响表 A 第三行——两处都在正文里明确写了"不推断"，第卅章的边界清单把它们逐条列全。
7. **第廿六章用这张矩阵设定判定上限**：去向决定证据形态，证据形态决定 L 级。任何"L3"都必须同时有 O3 的 `delivery_evidence` 与 O4 的可比对读数；只有 O1/O2 的回复不构成升级。
## 廿六、四级判定模型 L0–L3 与十个运营指标：把"读不到"算进分母

第廿五章把工单派完，本章给出验收口径：**一台设备的一个器件条目，在什么条件下可以写到哪一级**，以及**十个指标各自的分母是什么**。本章的唯一新内容是口径，不引入新来源；所有被引用的实测数字都已在第五、九、十三至廿五章出现过并登记在组 D 的取回记录里。

**视角声明（口径收敛）**：前十几章在写作过程中把若干运营需求预告给了具体的 K 编号，其中两处预告与实际分派不一致。本章在 K 总表之后附一张**收敛表**，逐条写出"哪一章的哪句话原本指向哪个 K、现在落在哪个 K、为什么"。这不是排版修正——**指标编号被两处引用指向不同定义，就是台账 bug**，必须在定义章里公开处理。

### 廿六之一 四个级别与各自的准入条件

| 级别 | 名称 | 准入条件（全部满足才算这一级） | 必须持有的字段 | 禁止出现的说法 |
| --- | --- | --- | --- | --- |
| **L0** | 无判定基础（运营别名 `L0-LOW`） | 该条目在设备侧没有任何可得读数，且第廿七章脚本的对应分支全部给出不可读/不存在/不可达类判定；公告侧只有 T2 通道证据 | `unavailable_reason` 必填且取值合法 | "未发现缺陷"、"该设备不受影响"、"安全" |
| **L1** | 对象在场，状态未知 | 能确认器件、通道或节点存在（第二十章的 verdict ∈ `present`/`bound`/`openable`/`eacces_typed`/`eacces_untagged`），但没有任何版本或修复状态读数 | `mapping_class` 至少能定档；`debugface_present` 或 `value_source` 之一有值 | "已装载"、"正在使用"、"版本为 X" |
| **L2** | 有可比读数，证据不齐 | 拿到一个可对齐的东西（一个版本串、一个 `state` 值、一条 XML 里的编号、一个模式漂移），但三条配套里至少缺一条（来源标签、期望下限、送达证据） | `version_semantics`、`gate_kind`、`asterisk`、`fw_class` 按条目适用项填写 | "已修复"、"低于安全基线"、"受影响" |
| **L3** | 三线一致 | 公告侧条目、O3 出具的送达证据、设备侧可比对读数三者用同一主键对齐；且结论没有依赖任何跨厂比较 | `cve_id_raw`、`delivery_evidence`（∈ `supplemental_xml` / `vendor_ota_manifest` / `absent`）、`verification_status` | 无——L3 是本域唯一可以对外承诺的级别 |

**L0 的定义不是"安全"**（第一章之一预告过这句）。L0 的确切含义是"证据不足且无法补采"，它是四个级别里唯一**不需要解释成风险**的级别：一条 L0 记录既不意味着器件没事，也不意味着厂商失职，它意味着**这台设备在本域没有可核验面**。第一章之三那句被反复误用的话在这里给出正式形态：**本文给出的所有读数缺失，都不能被读成"这台设备没有加速器固件缺陷"**。

### 廿六之二 升级与降级的六条硬规则

1. **单类证据不升 L2。** 第廿一章之十三第 6 条给出理由：FT6 使名字类读数可伪造、FT4 使版本串跨期不可比、FT10 使八面读数缺席。三条里任何一条成立时，那一条证据本身不足以支撑状态判断。
2. **能力位读数封顶 L2。** 第十九章与廿一章 FT7 给出：本域公开来源里没有任何一条说明某个能力位控制安全缓解，因此 `capability_gates_security` 默认 `unknown`，凡以能力位为唯一依据的判断不得高于 L2。
3. **送达证据只能由 O3 出具。** 第廿五章之一第 3 点的规则在这里成为判定条件：O1（厂商固件团队）与 O2（驱动维护者）提供的都是**技术事实**，不是送达事实。一条"厂商说已经修了"的回复不构成升级。
4. **带星且属器件族 ⇒ 无 T3 证据时封顶 L0。** 第廿四章之十五与第廿三章之六给出的机制：`asterisk = yes ∧ fw_class = yes ⇒ verification_status = no_device_side_evidence`。升级的唯一路径是命中 T3。
5. **降级触发一：版本语义不明。** 一旦 `version_semantics` 落到 `build-hash`、`index` 或 `unknown`，任何原本按版本做出的判断立刻降一级。实例（第十五、廿一章）：Venus 的 `"1.0.<hash>"` 被解析成 `1.0.0`，注释逐字承认 `SC7280 also reports "1.0.<hash>" parsed as 1.0.0; restricting to IRIS2`。
6. **降级触发二：模式漂移。** 第廿八章脚本 2 检测到写侧节点模式变化（例如四个 remoteproc 可写属性从 `0644` 变 `0444`）时，该链的 L 级降一级并记 `degraded_by_vendor`——**不升不降都是错的**：它既不是攻击证据，也不是加固证据。

### 廿六之三 从公告通道直接推出的 L 级上限

第廿四章的十三条组与三条通道，在这里换算成判定上限。**这张表是运营方最该贴在墙上的东西**：它告诉你光读公告能走到哪一步。

| 通道 | 典型组 | 上限 | 到上限所需的最少动作 | 到上限之后还缺什么 |
| --- | --- | --- | --- | --- |
| T1 平台源码通道 | 1、2、3 | **L1** | 按 `Type` + 版本列进平台台账 | 器件归属；平台修复与设备镜像之间没有桥 |
| T2 厂商公告通道 | 5、6、7、9、10、11 | **L0**（带星且属器件族） | 登记外键族与 `disclosure_tier` | 送达证据 + 设备侧读数，两者同时缺 |
| T2 但带可下钻外键 | 9 的 `QC-CR#` 行 | **L1** | 厂商入口取到正文 | 本域该入口在第九章实测为不可匿名读全文 |
| T3 记账例外通道 | 13 | **L2**（命中 XML 且编号可对齐） | 采到 `/vendor/etc/security/supplemental_security_patches.xml` 并读到该编号 | 期望下限（谁认定"这个版本不安全"）与镜像级读数 |
| T3 + 厂商给出可比对字段 | — | **L3** | 上述全部 + 一个可比对的设备侧版本字段 | 无 |
| 缓解/特性分组表 | 4 | **不适用** | 当作"该缺陷有平台或服务缓解"的布尔信号 | 它不是器件条目，不进任何器件分母 |

⇒ 本域的**实际天花板是 L2**：要升到 L3 需要一个"厂商已公开认定下限 + 设备侧可比对字段"的组合，而第廿一章 FT3 的反证条件已经证明——**没有公告版本边界时不能记 FT3**，同理也不能记 L3。第卅章会把"本域是否存在任何一条能到 L3 的公开路径"列为边界问题。

### 廿六之四 禁止跨厂商比较版本串：一条不许商量的规则

**规则**：任何形如"A 厂的 GPU 固件版本低于 B 厂"、"这个 `vdec` 版本比那个新"、"跨厂商比 SPL 覆盖"的表述，在本域一律禁止写入台账，`cross_vendor_compare` 字段唯一合法取值是 `forbidden`。

四条实测依据：

1. **同一个字段的语义在七条链里互不相同**（第廿一章 FT4）：Adreno 把整数按位段解释（`buf[0] & 0xfff) >= 0x190` 这类）；MFC 存的是文件名数组下标（枚举只有两个值）；Venus 解析出的是 `major.minor.rev` 且第三种输入直接变成 `1.0.0`；ivpu 是 `api_version[]` 里的 major/minor；Panther 是固件自报的 `(major<<24)|(minor<<16)|patch`；IPU3 与 FIMC-IS 是镜像里的自由文本串。**这六种语义里没有任何两种可以用同一个大小关系比较。**
2. **位长分布逐期变化**（第廿三章）：2026-09 页的 CVE 序列位长是 4 位 11 个、5 位 164 个；2026-06 页是 4 位 43 个、5 位 71 个。跨期沿用上一期的位长假设，直接产出格式合法的伪造编号（4 位正则在 2026-09 页造出 62 个不存在的编号）。
3. **同一族的外键形状也跨年变化**：`A-` 引用号在 2017 页全部 8 位，在 2021/2024/2025-09/2026-09/2026-10 五页全部 9 位（合计 386 个实例，8 位实例 0 个）。
4. **厂商文档给出的档位不等价**（第廿二章规则七）：六家厂商在公告里愿意给出的粒度各不相同（Arm 只给词、MediaTek 给型号系列、Qualcomm 闭源表只给占位串），`disclosure_tier` 不同的两个读数放一起比较，比的是披露意愿，不是安全状态。

**允许的五种比较**（穷举式白名单，其余一律禁止）：
① 同一厂商、同一产品线、同一命名规则内的版本先后；② 同一设备两次快照之间的自身漂移（第廿八章脚本 2）；③ 同一编号在同一期公告内的槽位数与去重后条目数（台账自检）；④ 一台设备的 SPL 与其 `/vendor/etc/security/supplemental_security_patches.xml` 声明的补丁集是否自相矛盾；⑤ 公告词到内核对象的映射覆盖率（K9）在不同厂商之间的**覆盖率**比较——这比的是我们的采集与映射能力，不是器件安全。

第⑤条是唯一允许的跨厂商比较，且必须写成"映射覆盖率"而不是"安全水平"。把它写成后者的工单退回 O5。

### 廿六之五 K1–K10 指标总表

| 指标 | 名称 | 分子 | 分母 | 数据来源 |
| --- | --- | --- | --- | --- |
| **K1** | 器件修复可归因率 | 有 T3 送达证据（`supplemental_xml` 或 `vendor_ota_manifest`）的本域条目数 | 公告侧命中的媒体/加速器条目总数（**含占位串行、含带星条目**） | 第八、廿四章 |
| **K2** | 采集姿势可读率 | 26 个姿势中取到值的项目数 | **26 全量**，含策略静音与构建关闭项 | 第廿章 |
| **K3** | 读数存在率 | V1–V8 各面在源码里确实存在的字段数 | 本报告登记的字段全集（含不可读字段；含严重度解读所必需的缓解与能力声明字段） | 第十九、廿四章 |
| **K4** | 披露滞后天数 | Σ（公开披露日 − OEM 知情日） | 本域条目数 | 第九章（MediaTek 原句：`at least two months before publication`） |
| **K5** | 不可验收条目占比 | `verification_status = no_device_side_evidence` 的条目数 | 本域公告条目总数（含 `Closed-source component` 占位串行） | 第廿三、廿四章 |
| **K6** | 协处理器崩溃可见性 | 能在既有采集姿势下取到 `state == crashed` 或 `crash_cnt` 增量的链数 | 七条链 | 第十七至十九章 |
| **K7** | 采集窗口覆盖率 | 实际覆盖到的窗口数 | 三个必覆盖窗口 + 一个日志通道条件（见廿六之六） | 第十一、十四、十五章 |
| **K8** | 姿势可取到率 | 同一字段全集里，在当前姿势下能取到值的字段数 | 与 K3 **同一分母** | 第十九、廿章 |
| **K9** | 公告词映射覆盖率 | 能对到 E1/E2/E3 三档之一的公告词数 | 实测表名与 Subcomponent 词全集 | 第五、十六、十八章 |
| **K10** | 归属不明条目占比 | `mapping_class ∈ {B, C}` 或 Subcomponent 为占位串的条目数 | 本域条目总数（与 K5 同集合，不同分子） | 第六、十六、廿四章 |

**K3 与 K8 是同一分母上的两个分子**——这是第十五章预告的那句"一条读数存在，与一条读数能在你的采集姿势下取到，是两个指标"的正式形态。两者的差值就是策略静音与构建关闭造成的损失量，也是唯一能用来向厂商主张"请给我们一个可读出口"的数字。

### 廿六之六 五个指标的细口径（K2、K3、K6、K7、K8）

**K2（采集姿势可读率）**
- 分母固定为 **26**（第廿五/廿七章脚本 1 的 26 条分支），**不随设备型号裁剪**。理由：分母一裁剪，一台"什么都没有"的设备就会得到 100%。
- 分子只计 `verdict = present` 且值真的取到的项。**九类"读失败"verdict 不进分子、照留在分母**：`absent`、`unlistable`、`eacces_typed`、`eacces_untagged`、`denied_value`、`denied_log`、`filtered_by_policy`、`debugfs_unreachable`、`muted_by_build`。另有四类同样不进分子（`label_unreadable`、`vendor_dependent`、`not_inferable`、`degraded_by_vendor`），分母归属不变；`openable` 与 `bound` 只证明在场，也不进分子。第廿七章的 26 条分支必须把每个失败分支归入这十四项之一，其中 P-16 那个"目录可列、模式可 stat、值不可读"的组合在第廿七章正式并入白名单（此前由 `unlistable` 顶位）。
- 禁止：不得把 `dontaudit` 静音项从分母里剔除（第十九章之四与第廿章的机制条目：拒绝且不留日志）。

**K3（读数存在率）**
- 分母 = 本报告登记的"字段全集"，取自第卅章清单里列出的每一个具名读数对象（版本串、状态、模式、标签、通道名、XML 键、property）。
- 分母**必须包含**解读严重度所必需的缓解与能力声明字段——第廿四章之二的逐字前提是 `assuming the platform and service mitigations are turned off for development purposes or if successfully bypassed`；没有这些字段，任何严重度读数都不可用，因此它们属于 K3 的分母而不是附注。
- 禁止：不得把"源码里有这个结构体成员"算作 K3 的分子项——分子要的是**读数**（能取到一个值），不是**变量**。第十九章 V7 那一格因为 atomisp 的恒假返回而记 `△` 就是这个区分。

**K6（协处理器崩溃可见性）**
- 分母 = 七条链（Mali CSF、Adreno GMU/KGSL、Exynos MFC、QCOM Venus、MTK VPU+SCP、相机 ISP、NPU/APU）。
- 分子只计**既有采集姿势能取到**的崩溃读数。第十二章 `:1050` 那条 `drm_warn`（软挂起未致命）**不计入任何分子**，也不作为分母的"已覆盖"证据；GPU 侧 `reset_count` 是趋势项，做分母里的辅助观测、不做分子（第十三章之五：固件重启不是固件更换）。

**K7（采集窗口覆盖率）**
- 分母 = 4：**一次冷启动**（MFC 的固件版本读寄存器发生在 `s5p_mfc_init_hw()` 路径里，每次硬件初始化一次，不是每次解码一次）、**一次真实解码会话**（MTK SCP 侧远端核可能长期 offline，打开一次 `/dev/video*` 之后才 `running` 并被复用）、**一次装载失败或拒绝**（`-ENOENT` 回落链与 `bad_fw` 分支都只在失败时留下痕迹）、**日志通道条件**（`dmesg` 与 bugreport 两种现场：版本日志在有 adb 的现场基本读不到、在有 bugreport 的现场读得到）。
- 分子 = 实际被覆盖到的窗口数。四个都覆盖不到时，该链的 K7 记 0/4 而不是"未采集"。
- 禁止：不得用"跑了 30 天趋势"代替窗口覆盖——趋势不能替代冷启动窗口。

**K8（姿势可取到率）**
- 与 K3 同一分母；分子要同时满足两个条件：取到了值，且值带合法来源标签（`value_source` ∈ `fw_reported` / `driver_set` / `userspace_override`）。
- 没有伴读 `driver_override` 当前值的 rpmsg 名字读数**不进 K8 分子**（第十八章之五与第廿五章表 C 的强制伴读）。
- 与 K2 的区别：K2 以姿势为分母，K8 以字段为分母。同一批采集，两个数字通常会给出互相矛盾的乐观程度——**K2 高而 K8 低是常态**，意味着"命令都跑成功了但取不到可用值"。

### 廿六之七 口径收敛表（前十几章的 K 预告 → 实际落点）

| 出处 | 原文里的那句话 | 预告编号 | 实际落点 | 处理 |
| --- | --- | --- | --- | --- |
| 第二章之混用五 | "第廿七章的 K4 指标就是为它设计的"（披露滞后） | K4 | K4 ✓ | 一致 |
| 第三章 L-3 条 | "第廿七章的 K1/K4 指标与脚本判定模型都据此设计" | K1、K4 | K1、K4 ✓ | 一致 |
| 第八章场景 A | "按第廿六章降为 L2-WARN 并要求补采（K2）" | K2 | K2 ✓（分母 26 含不可读项） | "L2-WARN" 是 L2 的告警态，本章在 L2 行内实现 |
| 第十一章之五 | "第廿六章 K7 的适用条件"（厂商日志通道才算第二层证据） | K7 | K7 ✓（日志通道条件是 K7 分母第 4 项） | 一致 |
| 第十二章 `:1050` | "第 26 章 K5 的分母定义里明确排除它" | K5 | **K6** | 已改写：软挂起告警与"不可验收条目占比"无关，它属崩溃可见性的分子排除 |
| 第十三章之五 | "第廿六章 K6 用它做分母，不做分子"（`reset_count`） | K6 | K6 ✓ | 一致 |
| 第十四章之五 | "第廿六章 K7 用它说明采集窗口必须覆盖一次冷启动" | K7 | K7 ✓ | 一致 |
| 第十五章 | "第廿六章 K3/K8 的口径据此分列" | K3、K8 | K3、K8 ✓（同分母两分子） | 一致 |
| 第十五章 | "第廿六章 K7（公告词到内核对象的映射覆盖率）" | K7 | **K9** | 已改写：K7 已被采集窗口占用；映射覆盖率单列 K9 |
| 第十九章之四 | "K2/K3 的定义必须把分母写成含不可读项的节点总数" | K2、K3 | K2、K3 ✓ | 一致 |
| 第廿三章之六 | "这一条与 K5（不可验收条目占比）在第廿六章分母口径里对上" | K5 | K5 ✓ | 一致 |
| 第廿四章组 10 | "必须进 K5/K10 的分母" | K5、K10 | K5、K10 ✓（同集合、不同分子） | 一致 |

两处"已改写"的原文都做了对应修改，不是在本章单方面重新解释。**这张表要留着**：任何后续指标调整都必须在此追加一行，而不是直接改指标定义——否则第卅章复核时会发现正文与指标互相指错。

### 廿六之八 L0 的三种正确用法

L0 是本域最常见、也最容易被误用的级别。三种正确用法：

1. **作为补采工单的依据。** L0 + `unavailable_reason` 指向 `policy` 时，工单给 O4（改姿势）；指向 `vendor_dependent` 时给 O5（要求供应商在主键栏给映射）；指向 `not_in_tree` 时**不开工单**，只登记边界事实（`qaic` 主机侧无镜像就是这类）。
2. **作为风险接受声明的对象。** 一条 L0 条目可以被接受，但接受声明必须写"本域该器件无可核验面，接受依据是第卅章边界清单第 N 项"，不能写"未发现问题"。第廿五章 O5 的 `risk_accepted` 字段与第卅章边界项编号在这里互相引用。
3. **作为厂商主张的对照。** 当厂商说"该问题在本设备上不适用"，唯一可核对的形式是：它对应哪条链的哪一层（F1–F6），读数面是哪一面（V1–V8），不可得原因是什么。**三者给不全时，把它记为 L0 而不是接受该说法。**

一句禁止：L0 不得作为 K6、K9 之外的"覆盖率"分母的排除项。任何指标想排除 L0 项，必须在本章追加一条口径并同步修订对应指标行。

### 廿六之九 本章结论

1. **本域的实际判定天花板是 L2**，且到 L2 需要 T3 通道命中——而 T3 是 OEM 自愿的。这意味着"能不能给自己的器件条目定级"取决于厂商的记账选择，不取决于采集能力。
2. **`cross_vendor_compare` 恒为 `forbidden`**，唯一允许的跨厂比较是 K9（我们自己映射能力的覆盖率）。这一条把六种互不相容的版本语义挡在台账之外。
3. **K3 与 K8 共用分母**：这两个数字的差值是本报告能给运营方的最实用的一个数——它把"厂商没写"与"厂商写了但你读不到"分开，而后者是可以向厂商提要求的。
4. **K2 的分母固定 26，K7 的分母固定 4，K6 的分母固定 7**。三个固定分母的意义是：**跨设备可比**。任何"按型号裁剪分母"的优化都会让指标失去横向意义，第廿九章把它列为误区之一。
5. **指标编号冲突要公开收敛**。本章的收敛表处理了两处（K5→K6、K7→K9），并保留全部原始预告的位置。台账类文档最大的腐化来源是"定义改了但引用没改"。
6. **第廿七章给出脚本 1（姿态快照，26 条分支）与 K1/K4 的采出口径**，第廿八章给出脚本 2（两次快照的漂移比对）与 K6/K7/K8 的采出口径；两章的每个失败分支都必须能归入本章 K2 的九个不可进分子的 verdict 之一。
## 廿七、脚本 1：只读固件姿态快照，二十六条分支与一份真实输出

第廿章把二十六个采集姿势列成表，并在该章第一条写明"表里有、脚本里没有"就算报告缺陷。本章把那张表转成可执行体，并给出它在一台画像设备上的**完整真实输出**——不是节选，不是示意，是脚本一次运行打出来的七十行。

**视角声明（三条硬约束）**：①本章唯一的来源是第廿章那张表（P-01…P-26）、第卅章取回记录里已登记的规则行号，以及本章自己运行脚本产生的输出；不引入新文档、新公告、新源码行。②脚本**只读**：它调用的设备侧命令只有 `ls`、`stat`、`readlink`、`cat`、`getprop`、`find`、`dmesg`、`service list`、`lshal`、`dumpsys`，对第廿五章那三十八行能力事实**一律只 `stat` 取模式，不打开、不写、不 ioctl**。③所有读数都由读者在**自己的或已获明确授权的设备**上执行；脚本不修改设备状态，也不尝试任何提权、绕过或降级动作。

### 廿七之一 输出契约：八个字段与三个汇总行

字段顺序与第廿章之九逐字一致：

```
pid|face|target|verdict|label|err|rule|source
```

`err` 列里凡是设备给的原始终止信息都**不翻译、不改写**（第廿章之九对 P-04、P-18、P-22 三行特别要求保留实际错误串，因为这三处是厂商差异最大、也是唯一能区分厂商的线索）。字段内出现的竖线、换行与制表符统一压成单空格，这是格式要求，不影响错误文本本身。

末尾三行汇总（`K2|…`、`K1|…`、`K4|…`）与六行 `# ` 开头的禁止推出条目——那六条是第廿章之八的正面清单，脚本把它们打进输出末尾，是为了让拿到台账的人不自行补全。

### 廿七之二 脚本正文

```bash
#!/usr/bin/env bash
# w17-posture.sh - Android media/accelerator firmware posture snapshot (READ-ONLY)
# One pipe-delimited record per observation: pid|face|target|verdict|label|err|rule|source
set -u
LC_ALL=C; export LC_ALL

SERIAL="${1:-}"
ADB=("${ADB_TOOL:-adb}")
[ -n "$SERIAL" ] && ADB+=(-s "$SERIAL")

R_OUT=""; R_RC=0; REC=""
run() { R_OUT=$("${ADB[@]}" shell "$1" 2>&1); R_RC=$?; R_OUT=${R_OUT%$'\n'}; }
classify() {
  case "$R_OUT" in
    *"Permission denied"*|*"Operation not permitted"*) echo denied ;;
    *"No such file"*|*"No such directory"*|*"does not exist"*|*"can't access"*) echo missing ;;
    *) if [ "$R_RC" -ne 0 ]; then echo other; else echo ok; fi ;;
  esac
}
san() { printf '%s' "$1" | tr '|\n\t' '   ' | sed 's/  */ /g; s/^ //; s/ $//'; }
rec() { REC="$REC$(printf '%s|%s|%s|%s|%s|%s|%s|%s\n' "$1" "$2" "$3" "$4" "$(san "$5")" "$(san "$6")" "$7" "$8")
"; }
cnt() { printf '%s' "$1" | grep -c '[^[:space:]]'; }
first() { printf '%s\n' "$1" | sed '/^$/d' | head -1; }

LBL=""; FWROOT=/vendor/firmware_mnt/image
NAMES="intel/ipu3-fw.bin irci_irci_ecr-master_20161208_0213_20170112_1500.bin shisp_2401a0_v21.bin shisp_2401a0_legacy_v21.bin shisp_2400b0_v21.bin exynos4_fimc_is_fw.bin exynos4_s5k6a3_setfile.bin vpu_p.bin vpu_d.bin mediatek/mt8173/vpu_p.bin vpu_37xx.bin vpu_40xx.bin cdsp.mdt adsp.mdt scp.img"
label_of() { printf '%s\n' "$LBL" | sed -n "s|^u:object_r:\([a-z_0-9]*\):s0 $1\$|\1|p" | head -1; }

echo 'pid|face|target|verdict|label|err|rule|source'

# ---- P-01 V1 firmware directories ----
for d in "$FWROOT" /vendor/firmware; do
  run "ls $d"; k=$(classify)
  case $k in
    denied)  rec P-01 V1 "$d" unlistable - "$R_OUT" domain.te:188 device_read ;;
    missing) rec P-01 V1 "$d" absent - "$R_OUT" file_contexts:423 device_read ;;
    ok)      rec P-01 V1 "$d" present "entries=$(cnt "$R_OUT")" - domain.te:188 device_read ;;
    *)       rec P-01 V1 "$d" vendor_dependent - "$R_OUT" domain.te:188 device_read ;;
  esac
done

# ---- P-02 V1 static-baseline candidate names ----
PATHS=""; for n in $NAMES; do PATHS="$PATHS$FWROOT/$n "; done
run "stat -c '%n %s' $PATHS"; ST2="$R_OUT"
for n in $NAMES; do
  sz=$(printf '%s\n' "$ST2" | sed -n "s|^$FWROOT/$n \([0-9]*\)\$|\1|p" | head -1)
  if [ -n "$sz" ]; then
    rec P-02 V1 "$FWROOT/$n" present "size=$sz" - domain.te:185-193 static_baseline
  else
    rec P-02 V1 "$FWROOT/$n" absent - "no stat line returned" domain.te:185-193 static_baseline
  fi
done

# ---- P-03 V1 the /vendor/etc/firmware exception ----
run "ls /vendor/etc/firmware"; k=$(classify)
case $k in
  ok)      rec P-03 V1 /vendor/etc/firmware present "entries=$(cnt "$R_OUT")" - file_contexts:427 device_read ;;
  denied)  rec P-03 V1 /vendor/etc/firmware unlistable - "$R_OUT" file_contexts:427 device_read ;;
  missing) rec P-03 V1 /vendor/etc/firmware absent - "$R_OUT" file_contexts:427 device_read ;;
  *)       rec P-03 V1 /vendor/etc/firmware vendor_dependent - "$R_OUT" file_contexts:427 device_read ;;
esac

# ---- P-04 V1 root mount points: vendor dependent, raw error kept ----
run "ls -d /firmware_mnt /firmware_mnt/image"; k=$(classify)
case $k in
  ok)      rec P-04 V1 /firmware_mnt present "entries=$(cnt "$R_OUT")" - domain.te:616-617 device_read ;;
  denied)  rec P-04 V1 /firmware_mnt eacces_untagged - "$R_OUT" domain.te:616-617 device_read ;;
  missing) rec P-04 V1 /firmware_mnt absent - "$R_OUT" domain.te:616-617 device_read ;;
  *)       rec P-04 V1 /firmware_mnt vendor_dependent - "$R_OUT" domain.te:616-617 device_read ;;
esac

# ---- P-05 V1 labels: the discriminator for the whole table ----
run "ls -Zd $FWROOT /vendor/firmware /dev/video0 /dev/accel/accel0 /dev/rpmsg0 /dev/ion /sys/class/remoteproc /vendor_dlkm/lib/modules"
LBL="$R_OUT"; k=$(classify)
if [ "$k" = ok ]; then
  rec P-05 V1 label-view present "lines=$(cnt "$R_OUT")" "$(first "$R_OUT")" shell.te:424 device_read
else
  rec P-05 V1 label-view label_unreadable - "$R_OUT" shell.te:424 device_read
fi

# ---- P-06 V1 vendor_dlkm split ----
for p in "/vendor_dlkm/lib/modules file_contexts:564" "/vendor_dlkm/etc file_contexts:565"; do
  set -- $p; d=$1; r=$2
  run "ls $d"; k=$(classify)
  case $k in
    ok)      rec P-06 V1 "$d" present "entries=$(cnt "$R_OUT")" - "$r" device_read ;;
    denied)  rec P-06 V1 "$d" unlistable - "$R_OUT" "$r" device_read ;;
    missing) rec P-06 V1 "$d" absent - "$R_OUT" "$r" device_read ;;
    *)       rec P-06 V1 "$d" vendor_dependent - "$R_OUT" "$r" device_read ;;
  esac
done

# ---- P-07 V2 getprop is a policy-filtered view ----
run "getprop"; ALLP="$R_OUT"
FWP=$(printf '%s\n' "$ALLP" | grep -iE 'fw|firmware|gmu|mfc|venus|vpu|npu|isp' || true)
if [ -n "$FWP" ]; then
  rec P-07 V2 getprop-fw-keys present "matches=$(cnt "$FWP")" "$(first "$FWP")" property.te:191-198 device_read
else
  rec P-07 V2 getprop-fw-keys filtered_by_policy - "no printable vendor firmware key" property.te:191-198 device_read
fi

# ---- P-08 V2 Android-side baseline, never a device-side firmware reading ----
for key in ro.build.version.security_patch ro.vendor.build.fingerprint ro.bootloader; do
  val=$(printf '%s\n' "$ALLP" | sed -n "s|^\[$key\]: \[\(.*\)\]\$|\1|p" | head -1)
  if [ -n "$val" ]; then rec P-08 V2 "$key" present "$val" - property.te:191-198 device_read
  else rec P-08 V2 "$key" absent - "key not in printable view" property.te:191-198 device_read; fi
done

# ---- P-09 / P-10 V3 declarations ----
run "ls /vendor/etc/media_codecs.xml /vendor/etc/media_codecs_c2.xml"; k=$(classify)
case $k in
  ok)      rec P-09 V3 media_codecs present "entries=$(cnt "$R_OUT")" "$(first "$R_OUT")" domain.te:176-178 device_read ;;
  denied)  rec P-09 V3 media_codecs unlistable - "$R_OUT" domain.te:176-178 device_read ;;
  missing) rec P-09 V3 media_codecs absent - "$R_OUT" domain.te:176-178 device_read ;;
  *)       rec P-09 V3 media_codecs vendor_dependent - "$R_OUT" domain.te:176-178 device_read ;;
esac
run "ls /vendor/etc/vintf/manifest.xml /vendor/manifest.xml"; k=$(classify)
case $k in
  ok)      rec P-10 V3 vintf-manifest present "entries=$(cnt "$R_OUT")" "$(first "$R_OUT")" file_contexts:438 device_read ;;
  denied)  rec P-10 V3 vintf-manifest unlistable - "$R_OUT" file_contexts:438 device_read ;;
  missing) rec P-10 V3 vintf-manifest absent - "$R_OUT" file_contexts:438 device_read ;;
  *)       rec P-10 V3 vintf-manifest vendor_dependent - "$R_OUT" file_contexts:438 device_read ;;
esac

# ---- P-11 V3 T3 accounting channel: first-class branch, feeds K1 ----
run "cat /vendor/etc/security/supplemental_security_patches.xml"; k=$(classify); K1EV=0; K1V=absent
case $k in
  ok)
    K1EV=$(printf '%s\n' "$R_OUT" | grep -c 'CVE-' || true)
    K1V=present
    rec P-11 V3 supplemental_security_patches present "cve_lines=$K1EV" "$(first "$R_OUT")" file_contexts:427 device_read ;;
  denied)  K1V=denied_value; rec P-11 V3 supplemental_security_patches denied_value - "$R_OUT" file_contexts:427 device_read ;;
  missing) rec P-11 V3 supplemental_security_patches absent - "$R_OUT" file_contexts:427 device_read ;;
  *)       K1V=vendor_dependent; rec P-11 V3 supplemental_security_patches vendor_dependent - "$R_OUT" file_contexts:427 device_read ;;
esac

# ---- P-12 V4 topology ----
for c in remoteproc accel drm video4linux rpmsg; do
  run "ls /sys/class/$c"; k=$(classify)
  case $k in
    ok)      rec P-12 V4 "/sys/class/$c" present "entries=$(cnt "$R_OUT")" "$(first "$R_OUT")" shell.te:443 device_read ;;
    denied)  rec P-12 V4 "/sys/class/$c" denied_value - "$R_OUT" shell.te:443 device_read ;;
    missing) rec P-12 V4 "/sys/class/$c" absent - "$R_OUT" genfs_contexts:123-124 device_read ;;
    *)       rec P-12 V4 "/sys/class/$c" vendor_dependent - "$R_OUT" shell.te:443 device_read ;;
  esac
done

# ---- P-13 V4 parent chain ----
for c in remoteproc accel drm video4linux; do
  run "readlink /sys/class/$c/*"; k=$(classify)
  if [ "$k" = ok ] && [ -n "$R_OUT" ]; then
    rec P-13 V4 "/sys/class/$c/*" present "$(first "$R_OUT")" - domain.te:196-197 device_read
  elif [ "$k" = denied ]; then
    rec P-13 V4 "/sys/class/$c/*" denied_value - "$R_OUT" domain.te:196-197 device_read
  else
    rec P-13 V4 "/sys/class/$c/*" absent - "$R_OUT" domain.te:196-197 device_read
  fi
done

# ---- P-14 V4 attribute values ----
for a in state firmware; do
  run "cat /sys/class/remoteproc/*/$a"; k=$(classify)
  r=shell.te:443; [ "$a" = firmware ] && r=remoteproc_sysfs.c:167
  case $k in
    ok)      rec P-14 V4 "rproc.$a" present "$(first "$R_OUT")" - "$r" device_read ;;
    denied)  rec P-14 V4 "rproc.$a" denied_value - "$R_OUT" "$r" device_read ;;
    missing) rec P-14 V4 "rproc.$a" absent - "$R_OUT" "$r" device_read ;;
    *)       rec P-14 V4 "rproc.$a" vendor_dependent - "$R_OUT" "$r" device_read ;;
  esac
done

# ---- P-15 V4 write-side attributes: mode only, never opened ----
run "stat -c '%n %a' /sys/class/remoteproc/*/coredump /sys/class/remoteproc/*/recovery /sys/class/remoteproc/*/firmware /sys/class/remoteproc/*/state"; k=$(classify)
case $k in
  ok)
    m=$(printf '%s\n' "$R_OUT" | sed -n 's|.*coredump \([0-9]*\)$|\1|p' | head -1)
    if [ "$m" = "444" ]; then
      rec P-15 V4 rproc.writable-attrs degraded_by_vendor "coredump_mode=$m" "four writable attrs reduced, sysfs_read_only set by a patch not in AOSP" remoteproc_sysfs.c:236-240 device_read
    else
      rec P-15 V4 rproc.writable-attrs present "coredump_mode=$m" "$(first "$R_OUT")" remoteproc_sysfs.c:229-243 device_read
    fi ;;
  denied)  rec P-15 V4 rproc.writable-attrs denied_value - "$R_OUT" remoteproc_sysfs.c:229-243 device_read ;;
  missing) rec P-15 V4 rproc.writable-attrs absent - "$R_OUT" remoteproc_sysfs.c:229-243 device_read ;;
  *)       rec P-15 V4 rproc.writable-attrs vendor_dependent - "$R_OUT" remoteproc_sysfs.c:229-243 device_read ;;
esac

# ---- P-16 V4 module parameters: listable + statable + value denied ----
for mod in atomisp ivpu; do
  d="/sys/module/$mod/parameters"
  run "ls $d"; kl=$(classify)
  run "stat -c '%n %a' $d/*"; ks=$(classify); SS="$R_OUT"
  run "cat $d/*"; kv=$(classify); SV="$R_OUT"
  modes=$(printf '%s\n' "$SS" | sed -n 's|.* \([0-9]*\)$|\1|p' | tr '\n' ',' )
  if [ "$kl" = missing ] || [ "$ks" = missing ]; then
    rec P-16 V4 "$d" absent - "$SV" genfs_contexts:182-186 device_read
  elif [ "$kl" = denied ]; then
    rec P-16 V4 "$d" unlistable - "$SV" genfs_contexts:123-124 device_read
  elif [ "$ks" = ok ] && [ "$kv" = denied ]; then
    rec P-16 V4 "$d" unlistable_value "modes=$modes" "$SV" genfs_contexts:123-124 device_read
  elif [ "$kv" = ok ]; then
    rec P-16 V4 "$d" present "values=$(cnt "$R_OUT")" - genfs_contexts:123-124 device_read
  else
    rec P-16 V4 "$d" vendor_dependent - "$SV" genfs_contexts:182-186 device_read
  fi
done

# ---- P-17 V4 PCI driver object: presence is a binding witness ----
run "ls /sys/bus/pci/drivers/atomisp-isp2/"; k=$(classify)
case $k in
  ok)
    if printf '%s\n' "$R_OUT" | grep -q 'dbgfun'; then
      rec P-17 V4 atomisp-isp2 bound dbgfun,dbgopt - atomisp_drvfs.c:164-172 device_read
    else
      rec P-17 V4 atomisp-isp2 present "no dbg files" - atomisp_drvfs.c:160-161 device_read
    fi ;;
  missing) rec P-17 V4 atomisp-isp2 absent - "$R_OUT" atomisp_v4l2.c:1606 device_read ;;
  denied)  rec P-17 V4 atomisp-isp2 denied_value - "$R_OUT" genfs_contexts:123-124 device_read ;;
  *)       rec P-17 V4 atomisp-isp2 vendor_dependent - "$R_OUT" atomisp_drvfs.c:164-172 device_read ;;
esac

# ---- P-18 V4 devicetree: nodes listable, property values not ----
run "ls /sys/firmware/devicetree/base/"; k=$(classify)
case $k in
  ok)      rec P-18 V4 dtb.base present "entries=$(cnt "$R_OUT")" "$(first "$R_OUT")" genfs_contexts:152-154 device_read ;;
  denied)  rec P-18 V4 dtb.base denied_value - "$R_OUT" genfs_contexts:152-154 device_read ;;
  missing) rec P-18 V4 dtb.base absent - "$R_OUT" genfs_contexts:152-154 device_read ;;
  *)       rec P-18 V4 dtb.base vendor_dependent - "$R_OUT" genfs_contexts:152-154 device_read ;;
esac
run "cat /sys/firmware/devicetree/base/firmware-name /sys/firmware/devicetree/base/*/firmware-name"; k=$(classify)
case $k in
  ok)      rec P-18 V4 dtb.firmware-name present "$(first "$R_OUT")" - remoteproc_core.c:1002 device_read ;;
  denied)  rec P-18 V4 dtb.firmware-name denied_value - "$R_OUT" remoteproc_core.c:1002 device_read ;;
  missing) rec P-18 V4 dtb.firmware-name absent - "$R_OUT" remoteproc_core.c:1002 device_read ;;
  *)       rec P-18 V4 dtb.firmware-name vendor_dependent - "$R_OUT" remoteproc_core.c:1002 device_read ;;
esac

# ---- P-19 / P-20 V5 character devices, two different causes ----
for n in /dev/video0 /dev/rpmsg0 /dev/rpmsg_ctrl0 /dev/accel/accel0 /dev/remoteproc0; do
  run "stat -c '%n %C %a' $n"; k=$(classify); l=$(label_of "$n")
  pid=P-20; face=V5; rl_lab=file_contexts:97; rl_name=rpmsg_char.c:459
  [ "$n" = /dev/video0 ] && { pid=P-19; rl_lab=file_contexts:221; rl_name=file_contexts:221; }
  case $k in
    ok)      rec "$pid" $face "$n" present "$(first "$R_OUT")" "$l" "$rl_lab" device_read ;;
    denied)
      if [ -n "$l" ] && [ "$l" != device ]; then
        rec "$pid" $face "$n" eacces_typed "$l" "$R_OUT" shell.te:424 device_read
      elif [ "$l" = device ]; then
        rec "$pid" $face "$n" eacces_untagged "$l" "$R_OUT" file_contexts:97 device_read
      elif [ "$pid" = P-19 ]; then
        rec "$pid" $face "$n" eacces_typed label_unreadable "$R_OUT" file_contexts:221 device_read
      else
        rec "$pid" $face "$n" label_unreadable - "$R_OUT" file_contexts:97 device_read
      fi ;;
    missing) rec "$pid" $face "$n" absent - "$R_OUT" "$rl_name" device_read ;;
    *)       rec "$pid" $face "$n" vendor_dependent - "$R_OUT" file_contexts:97 device_read ;;
  esac
done

# ---- P-21 V5 debugfs: build variable before vendor lockdown ----
run "getprop ro.build.type"; BT=$(printf '%s' "$R_OUT" | tr -d ' \t\n')
run "ls /sys/kernel/debug"; k=$(classify)
case $k in
  ok)      rec P-21 V5 debugfs vendor_dependent "build=$BT entries=$(cnt "$R_OUT")" - te_macros:591 device_read ;;
  denied)
    if [ "$BT" = user ]; then
      rec P-21 V5 debugfs muted_by_build "build=$BT" "$R_OUT" te_macros:614 device_read
    else
      rec P-21 V5 debugfs debugfs_unreachable "build=$BT" "$R_OUT" te_macros:591 device_read
    fi ;;
  missing) rec P-21 V5 debugfs debugfs_unreachable "build=$BT" "$R_OUT" drm_accel.c:215 device_read ;;
  *)       rec P-21 V5 debugfs debugfs_unreachable "build=$BT" "$R_OUT" shell.te:7-11 device_read ;;
esac

# ---- P-22 V5 ivpu fw_name: the only runtime firmware-name exit ----
run "find /sys/kernel/debug -maxdepth 3 -name fw_name"; k=$(classify)
case $k in
  ok)
    if [ -n "$R_OUT" ]; then
      fn=$(first "$R_OUT"); run "cat $fn"; kc=$(classify)
      if [ "$kc" = ok ]; then
        rec P-22 V5 "$fn" present "$(first "$R_OUT")" - ivpu_debugfs.c:30-36 device_read
      else
        rec P-22 V5 "$fn" denied_value - "$R_OUT" ivpu_debugfs.c:30-36 device_read
      fi
    else
      rec P-22 V5 fw_name-lookup absent "no match under debugfs root" - ivpu_debugfs.c:107-115 device_read
    fi ;;
  denied)  rec P-22 V5 fw_name-lookup debugfs_unreachable - "$R_OUT" te_macros:591 device_read ;;
  missing) rec P-22 V5 fw_name-lookup debugfs_unreachable - "$R_OUT" te_macros:591 device_read ;;
  *)       rec P-22 V5 fw_name-lookup vendor_dependent - "$R_OUT" ivpu_debugfs.c:107-115 device_read ;;
esac

# ---- P-23 V5 /dev/ion: the one char device AOSP allows ----
run "stat -c '%n %C %a' /dev/ion"; k=$(classify)
case $k in
  ok)      rec P-23 V5 /dev/ion openable "$(first "$R_OUT")" - shell.te:463 device_read ;;
  denied)  rec P-23 V5 /dev/ion denied_value - "$R_OUT" file_contexts:137 device_read ;;
  missing) rec P-23 V5 /dev/ion absent - "$R_OUT" file_contexts:137 device_read ;;
  *)       rec P-23 V5 /dev/ion vendor_dependent - "$R_OUT" file_contexts:137 device_read ;;
esac

# ---- P-24 V6 kernel log ----
run "dmesg"; k=$(classify)
case $k in
  ok)
    M=$(printf '%s\n' "$R_OUT" | grep -iE 'mfc|venus|gmu|ipu3|atomisp|fimc|ivpu|qaic|remoteproc|adsp|cdsp' || true)
    rec P-24 V6 dmesg present "matches=$(cnt "$M")" "$(first "$M")" dumpstate.te:211 device_read ;;
  denied)  rec P-24 V6 dmesg denied_log - "$R_OUT" app.te:660-661 device_read ;;
  *)       rec P-24 V6 dmesg vendor_dependent - "$R_OUT" genfs_contexts:53 device_read ;;
esac

# ---- P-25 V8 declared services ----
run "service list"; k=$(classify)
if [ "$k" = ok ]; then
  S=$(printf '%s\n' "$R_OUT" | grep -iE 'camera|codec|media|gpu|neural' || true)
  rec P-25 V8 service-list present "matches=$(cnt "$S")" "$(first "$S")" shell.te:358 device_read
else
  rec P-25 V8 service-list denied_value - "$R_OUT" shell.te:358 device_read
fi
run "lshal"; k=$(classify)
if [ "$k" = ok ]; then
  H=$(printf '%s\n' "$R_OUT" | grep -iE 'camera|codec2|graphics|neural' || true)
  rec P-25 V8 lshal present "matches=$(cnt "$H")" "$(first "$H")" shell.te:386 device_read
else
  rec P-25 V8 lshal denied_value - "$R_OUT" shell.te:386 device_read
fi

# ---- P-26 V8 binder reachability ----
for s in gpu media.codec cameraserver; do
  run "dumpsys $s"; k=$(classify)
  case $k in
    ok)      rec P-26 V8 "dumpsys $s" present "bytes=${#R_OUT}" - shell.te:79 device_read ;;
    denied)  rec P-26 V8 "dumpsys $s" vendor_dependent - "$R_OUT" shell.te:29-219 device_read ;;
    missing) rec P-26 V8 "dumpsys $s" absent - "$R_OUT" shell.te:29-219 device_read ;;
    *)       rec P-26 V8 "dumpsys $s" vendor_dependent - "$R_OUT" shell.te:29-219 device_read ;;
  esac
done

# ---- rollup ----
printf '%s' "$REC"
RECS="$REC"
TOT=$(printf '%s' "$RECS" | awk -F'|' '{print $1}' | sort -u | wc -l | tr -d ' ')
NUM=$(printf '%s' "$RECS" | awk -F'|' '{if($4=="present"||$4=="openable"||$4=="bound") g[$1]=g[$1]+0; else g[$1]=1} END{n=0; for(k in g) if(g[k]==0) n++; print n}' | tr -d ' ')
BAD=$(printf '%s' "$RECS" | awk -F'|' '{if($4!="present"&&$4!="openable"&&$4!="bound") print $1}' | sort -u | wc -l | tr -d ' ')
echo "K2|rollup|posture-branches|num=$NUM|den=26|pids_total=$TOT|pids_with_failure=$BAD|note=denominator is the 26 P-ids and is never trimmed per model"
echo "K1|rollup|delivery-evidence|xml=$K1V|cve_lines=$K1EV|note=numerator accepts only supplemental_xml or vendor_ota_manifest"
echo "K4|rollup|disclosure-lag|oem_informed_date=not_device_read|public_date=not_device_read|note=inputs are bulletin-side, MediaTek sentence gives the two-month floor"
echo '# 1. unlistable, denied_value and denied_log are not evidence that the attribute, log or directory does not exist.'
echo '# 2. present proves only that a name or node matched; it is not a version reading and not load proof.'
echo '# 3. working behaviour is not version compliance; the atomisp gate warns and the QAIC CRC switch is set by the peripheral.'
echo '# 4. declaration is not presence; the manifest version is a HAL interface version, not a firmware version.'
echo '# 5. every candidate name in P-02 is static_baseline taken from kernel source, not device_read.'
echo '# 6. inability to read a switch value is not evidence that the switch was never used.'
```

上面整段是脚本全文（第廿九章按围栏区间抽回原文件做 `bash -n` 与画像运行比对）。它的结构是刻意**扁平**的：二十六个区块每个都把自己那条姿势的四种分类结果（成功 / 拒绝 / 不存在 / 其它错误）写成四个分支，不共用一个通用映射函数。理由不是风格——是**审计成本**：一处姿势的失败态语义在本域各不相同（P-01 的拒绝要记 `unlistable` 而不是 `denied_value`，P-14 的拒绝要记 `denied_value`，P-19 的拒绝要分成两种 `eacces`），共用函数会把这三种处置压成一个，那正是第廿章之一第 2 条禁止的做法。

`classify()` 的四分法只做一件事：把设备给的错误文本映射到"成功 / 拒绝 / 不存在 / 其它"四类，剩下的语义判断全部留在各分支里。`其它` 一律落 `vendor_dependent`——本域不允许把看不懂的错误码当成任何结论。

### 廿七之三 画像声明：这台"设备"是合成的

样例输出来自一个 **stub `adb`**（三个画像：`good` / `user` / `wide`），不是真机。声明清楚三件事：

1. **画像是合成的**：它把七条链的读数面压到一台机器上，目的是让二十六个分支在同一次运行里全部产出记录。真实设备只会给出其中一部分——一条链的器件缺席时，对应行的 `verdict` 会变成 `absent`，而不是报错。
2. **画像里的文件名与属性值只用于产生输出**，不作为任何事实引用。候选固件名那一栏（`NAMES`）是另一回事：那十五条名字逐条来自第卅章取回记录里的源码 `#define`、`fw_names[]` 与 `MODULE_FIRMWARE`，其 `source` 一律记 `static_baseline`，这正是脚本存在的理由之一。
3. **第廿九章的分支覆盖验收就是拿这三个画像跑的**，不拿真机跑：真机不能保证每个分支都被触发，而"从不触发的分支算脚本的缺陷"这条只在可控画像上可判。

### 廿七之四 真实输出（`good` 画像，一次运行的全部七十行）

```text
pid|face|target|verdict|label|err|rule|source
P-01|V1|/vendor/firmware_mnt/image|unlistable|-|ls: /vendor/firmware_mnt/image: Permission denied|domain.te:188|device_read
P-01|V1|/vendor/firmware|absent|-|ls: /vendor/firmware: No such file or directory|file_contexts:423|device_read
P-02|V1|/vendor/firmware_mnt/image/intel/ipu3-fw.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/irci_irci_ecr-master_20161208_0213_20170112_1500.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/shisp_2401a0_v21.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/shisp_2401a0_legacy_v21.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/shisp_2400b0_v21.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/exynos4_fimc_is_fw.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/exynos4_s5k6a3_setfile.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/vpu_p.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/vpu_d.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/mediatek/mt8173/vpu_p.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/vpu_37xx.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/vpu_40xx.bin|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/cdsp.mdt|present|size=5242880|-|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/adsp.mdt|absent|-|no stat line returned|domain.te:185-193|static_baseline
P-02|V1|/vendor/firmware_mnt/image/scp.img|present|size=262144|-|domain.te:185-193|static_baseline
P-03|V1|/vendor/etc/firmware|present|entries=2|-|file_contexts:427|device_read
P-04|V1|/firmware_mnt|eacces_untagged|-|ls: /firmware_mnt: Permission denied|domain.te:616-617|device_read
P-05|V1|label-view|present|lines=6|u:object_r:vendor_file:s0 /vendor/firmware_mnt/image|shell.te:424|device_read
P-06|V1|/vendor_dlkm/lib/modules|unlistable|-|ls: /vendor_dlkm/lib/modules: Permission denied|file_contexts:564|device_read
P-06|V1|/vendor_dlkm/etc|present|entries=2|-|file_contexts:565|device_read
P-07|V2|getprop-fw-keys|present|matches=1|[ro.input.noindependent]: [0]|property.te:191-198|device_read
P-08|V2|ro.build.version.security_patch|present|2026-06-05|-|property.te:191-198|device_read
P-08|V2|ro.vendor.build.fingerprint|present|ACME/PixelLike/PixelLike:16/HD1A.250101.001/1234:userdebug/release-keys|-|property.te:191-198|device_read
P-08|V2|ro.bootloader|present|unknown|-|property.te:191-198|device_read
P-09|V3|media_codecs|present|entries=2|/vendor/etc/media_codecs_c2.xml|domain.te:176-178|device_read
P-10|V3|vintf-manifest|present|entries=1|/vendor/etc/vintf/manifest.xml|file_contexts:438|device_read
P-11|V3|supplemental_security_patches|absent|-|cat: /vendor/etc/security/supplemental_security_patches.xml: No such file or directory|file_contexts:427|device_read
P-12|V4|/sys/class/remoteproc|present|entries=2|remoteproc0|shell.te:443|device_read
P-12|V4|/sys/class/accel|present|entries=1|accel0|shell.te:443|device_read
P-12|V4|/sys/class/drm|present|entries=2|card0|shell.te:443|device_read
P-12|V4|/sys/class/video4linux|present|entries=3|video0|shell.te:443|device_read
P-12|V4|/sys/class/rpmsg|absent|-|ls: /sys/class/rpmsg: No such file or directory|genfs_contexts:123-124|device_read
P-13|V4|/sys/class/remoteproc/*|present|/sys/devices/platform/soc/18810000.qcom_cdsp/remoteproc/remoteproc0|-|domain.te:196-197|device_read
P-13|V4|/sys/class/accel/*|present|/sys/devices/pci0000:00/0000:00:1f.3/accel/accel0|-|domain.te:196-197|device_read
P-13|V4|/sys/class/drm/*|present|/sys/devices/pci0000:00/0000:00:02.0/drm/card0|-|domain.te:196-197|device_read
P-13|V4|/sys/class/video4linux/*|present|/sys/devices/platform/ahb/1c000000.camera/video4linux/video0|-|domain.te:196-197|device_read
P-14|V4|rproc.state|denied_value|-|cat: /sys/class/remoteproc/remoteproc0/state: Permission denied|shell.te:443|device_read
P-14|V4|rproc.firmware|denied_value|-|cat: /sys/class/remoteproc/remoteproc0/firmware: Permission denied|remoteproc_sysfs.c:167|device_read
P-15|V4|rproc.writable-attrs|present|coredump_mode=644|/sys/class/remoteproc/remoteproc0/coredump 644|remoteproc_sysfs.c:229-243|device_read
P-16|V4|/sys/module/atomisp/parameters|absent|-|cat: /sys/module/atomisp/parameters/*: No such file or directory|genfs_contexts:182-186|device_read
P-16|V4|/sys/module/ivpu/parameters|unlistable_value|modes=644,644,644,|cat: /sys/module/ivpu/parameters/disable_mmu_cont_pages: Permission denied|genfs_contexts:123-124|device_read
P-17|V4|atomisp-isp2|absent|-|ls: /sys/bus/pci/drivers/atomisp-isp2/: No such file or directory|atomisp_v4l2.c:1606|device_read
P-18|V4|dtb.base|present|entries=6|adsp|genfs_contexts:152-154|device_read
P-18|V4|dtb.firmware-name|denied_value|-|cat: /sys/firmware/devicetree/base/cdsp/firmware-name: Permission denied|remoteproc_core.c:1002|device_read
P-19|V5|/dev/video0|eacces_typed|video_device|stat: cannot stat '/dev/video0': Permission denied|shell.te:424|device_read
P-20|V5|/dev/rpmsg0|eacces_untagged|device|stat: cannot stat '/dev/rpmsg0': Permission denied|file_contexts:97|device_read
P-20|V5|/dev/rpmsg_ctrl0|label_unreadable|-|stat: cannot stat '/dev/rpmsg_ctrl0': Permission denied|file_contexts:97|device_read
P-20|V5|/dev/accel/accel0|label_unreadable|-|stat: cannot stat '/dev/accel/accel0': Permission denied|file_contexts:97|device_read
P-20|V5|/dev/remoteproc0|label_unreadable|-|stat: cannot stat '/dev/remoteproc0': Permission denied|file_contexts:97|device_read
P-21|V5|debugfs|debugfs_unreachable|build=userdebug|ls: /sys/kernel/debug: Permission denied|te_macros:591|device_read
P-22|V5|fw_name-lookup|debugfs_unreachable|-|find: /sys/kernel/debug: Permission denied|te_macros:591|device_read
P-23|V5|/dev/ion|openable|/dev/ion u:object_r:ion_device:s0 666|-|shell.te:463|device_read
P-24|V6|dmesg|denied_log|-|dmesg: klogctl: Permission denied|app.te:660-661|device_read
P-25|V8|service-list|present|matches=3|0 media.player: [android.media.IMediaPlayerService]|shell.te:358|device_read
P-25|V8|lshal|present|matches=2|android.hardware.camera.provider@1.0::ICameraProvider legacy/0|shell.te:386|device_read
P-26|V8|dumpsys gpu|present|bytes=51|-|shell.te:79|device_read
P-26|V8|dumpsys media.codec|vendor_dependent|-|Can't find service: media.codec|shell.te:29-219|device_read
P-26|V8|dumpsys cameraserver|vendor_dependent|-|Permission denial: no access to cameraserver|shell.te:29-219|device_read
K2|rollup|posture-branches|num=10|den=26|pids_total=26|pids_with_failure=16|note=denominator is the 26 P-ids and is never trimmed per model
K1|rollup|delivery-evidence|xml=absent|cve_lines=0|note=numerator accepts only supplemental_xml or vendor_ota_manifest
K4|rollup|disclosure-lag|oem_informed_date=not_device_read|public_date=not_device_read|note=inputs are bulletin-side, MediaTek sentence gives the two-month floor
# 1. unlistable, denied_value and denied_log are not evidence that the attribute, log or directory does not exist.
# 2. present proves only that a name or node matched; it is not a version reading and not load proof.
# 3. working behaviour is not version compliance; the atomisp gate warns and the QAIC CRC switch is set by the peripheral.
# 4. declaration is not presence; the manifest version is a HAL interface version, not a firmware version.
# 5. every candidate name in P-02 is static_baseline taken from kernel source, not device_read.
# 6. inability to read a switch value is not evidence that the switch was never used.
```

### 廿七之五 这份输出里值得逐行看的十二处

1. **P-01 的两行给出两种缺席**：`/vendor/firmware_mnt/image` 是 `unlistable`（拒绝，目录可能有内容），`/vendor/firmware` 是 `absent`（路径不存在）。第廿章之一第 2 条要求的就是这个区分——把两者都写成"读不到"，运维就丢掉了"要不要请厂商补一条 dir 授权"这个判断依据。
2. **P-02 在 P-01 拒绝之后仍然给出两条 `present`**：`cdsp.mdt`（`size=5242880`）与 `scp.img`（`size=262144`）。这就是"已知名可 stat"与"目录不可列举"并存的标准形态；两行的 `source` 都是 `static_baseline`，因为**是源码给出这个名字，不是设备**。
3. **P-04 的 `/firmware_mnt` 落 `eacces_untagged`**：第卅章实测 AOSP `private/file_contexts` 全文检索 `firmware` 0 命中、`private/vendor_file_contexts` 不存在（404）且 `vendor/file_contexts`（212 行）内 `firmware` 0 命中，所以根级挂载点没有 AOSP 标签，拒绝成因是"未点名落兜底/未打标"而不是"有类型无授权"。这一行的处置动作要两步（先补标签再补授权），与 P-19 完全不同。
4. **P-05 那一行是全表的地基**：`label-view` 记 `present`，`label` 列给出拿到的标签行数（`lines=6`），第一行标签原文放进 `err` 列以便核对锚点。后面 P-19 与 P-20 的两种 `eacces` 全靠它区分；P-20 有三行落 `label_unreadable`，就是因为它们不在这六行里。
5. **P-06 的两行是同一分区上的两种结果**：`/vendor_dlkm/lib/modules` 落 `unlistable`，`/vendor_dlkm/etc` 却 `present entries=2`。`MODULE_FIRMWARE` 提取因此不是 shell 可达姿势，而模块配置文件是——这条只在源码里看是看不出来的。
6. **P-07 那一行是本报告最愿意展示的一次真实误命中**：`matches=1` 的第一条命中是 `[ro.input.noindependent]: [0]`——它被 `grep -iE '…|npu|…'` 抓住，是因为 `i**npu**t` 里含 `npu` 这三个字母。第廿三章把它列为编号陷阱的同族（子串≠词命中），第廿章 P-25 也警告过这条。**台账动作**：`matches>0` 时必须人工过一遍命中键名，不能把计数当"设备上报了固件版本"。
7. **P-11 在 `good` 画像里是 `absent`，这一行放到第廿四章 T3 通道语境里才是关键行**：这台画像设备**没有**补充补丁清单，因此 K1 分子为 0。第廿四章之十五给的规则在这里落地：读不到该文件**不得**记成"未修复"，只能记 `no_device_side_evidence`。
8. **P-14 与 P-15 是同一批文件的两种命运**：`cat` 两行都是 `denied_value`，`stat` 一行却 `present` 并给出 `coredump_mode=644`。第廿六章 K2/K8 的分母分歧就在这对行里——"我知道它的模式"与"我知道它的值"是两条读数。
9. **P-16 的两行给出本域独有的组合**：`ivpu` 那条 `unlistable_value`，`label` 列是 `modes=644,644,644,`，`err` 列是 `cat` 的实际拒绝串。目录可列、模式可取、值不可读——第廿章之八第 6 条禁止推出的正是从这里读的：**不能写"该开关未启用"**。
10. **P-19 / P-20 的三档 `eacces` 分列**：`/dev/video0` 是 `eacces_typed`（标签 `video_device` 拿到了，授权没有），`/dev/rpmsg0` 是 `eacces_untagged`（标签是兜底 `device`），其余三条 `/dev/rpmsg_ctrl0`、`/dev/accel/accel0`、`/dev/remoteproc0` 是 `label_unreadable`——因为 P-05 那次 `ls -Zd` 里没有它们，脚本不允许从"我不知道标签"推出"它没有标签"。
11. **P-21 与 P-22 是两条独立判定**：整块 debugfs 不可达（`debugfs_unreachable`，`build=userdebug`），而 `fw_name` 的查找同样失败但走的是同一成因。第廿章之七表末行要求先分"构建"再分"厂商"：`userdebug` 却仍然不可达 ⇒ 不是 `muted_by_build`；若是 `user` 构建，同一画像会打 `muted_by_build`（`user` 画像里 P-21 那一行就是这样）。
12. **P-26 的三行是三种不同的失败**：`dumpsys gpu` `present`（AOSP 给了 `shell.te:79` 的 binder 通路），`dumpsys media.codec` 是 `Can't find service` 落 `vendor_dependent`，`dumpsys cameraserver` 是 `Permission denial` 也落 `vendor_dependent`——两者都不可归因到 AOSP 授权缺失，但错误文本不同，保留原文就是为了将来能按文本分厂商。

### 廿七之六 词汇表修订：把 `unlistable_value` 转正

第廿章之九给出的白名单有十六个取值，而 P-16 那一行用的是第十七个词：`unlistable_value`。这一节把这件事正面处理掉，不做悄悄替换。

- **为什么 `unlistable` 顶不了这个位**：`unlistable` 的定义是"父目录不能列举，但已知名可 `stat`"（第廿章之一第 2 条）。P-16 的情况**正好相反**——目录可列、模式可 `stat`、只有值不可读。把两者并成一档，本章之后所有按 `unlistable` 统计"列不出来"的口径都会虚高，而且会让人以为 `/sys/module/*/parameters/` 列不出来（其实列得出来，这是本域少数能列出可写旋钮的地方）。
- **为什么不能降级成 `denied_value`**：`denied_value` 只说"值被拒"，不携带"其它两栏成功"的信息；第廿六章 K8 要判"读数带不带合法来源标签"，需要知道模式栏到底取到了什么。
- **修订结果**：脚本 1 的词汇表是 **十七个取值 = 原十六条 + `unlistable_value`**。第廿章之九那句"这十六个之一"从本章起按十七个执行；第卅章的可复核清单同时登记两处（原表 + 本修订）。第廿九章的分支覆盖验收按十七值核对，缺一值算脚本缺陷。
- **不在脚本输出里的第十八个词**：`not_inferable`（第廿章之八第 3 条）不由脚本产生。它是**分析者**在 V7（行为面）上做的归属判断——"解码成功推不出固件合规"不是任何命令能打印出来的。台账里出现 `not_inferable` 时必须能指出它挂在哪个 P-id 的哪一行上，指不出来的记录视为伪造。

### 廿七之七 K1 的采出口径（器件修复可归因率）

K1 的分子只接受两种送达证据：`supplemental_xml`（P-11 读到文件且里面有该编号）与 `vendor_ota_manifest`（厂商自己的发布清单，脚本不采，由 O3 录入）。

- **脚本给出的量**：`P-11` 那一行的 `present/absent/denied_value/vendor_dependent` 四态之一，加上 `cve_lines=<n>`（文件里出现 `CVE-` 的行数）。汇总行 `K1|rollup|delivery-evidence|xml=present|cve_lines=2|…` 就是这个。
- **口径三条**：①`cve_lines` 是**槽位数**不是条目数——同一个编号在 XML 里出现两次只算一条，第廿三章的去重规则在这里同样适用；②分子要求 XML 里的编号与公告侧条目**主键对齐**（`cve_id_raw` 逐字相等，不做前缀匹配，第廿三章之四）；③分母是公告侧命中的本域条目总数，**含占位串行、含带星条目**——脚本给不出分母，分母来自第廿四章的组表，这是 K1 唯一一处必须跨台账计算的量。
- **禁止**：不得把"读不到 XML"折算成"该设备未修复"（第廿四章之十五逐字给过这条），也不得把 `cve_lines=0` 与 `absent` 合并——前者是"文件存在但没列本域条目"，后者是"文件不存在"，两者的整改动作不同（前者问 O3，后者问 O4）。

样例里三个画像各给一种形态：`good` 是 `absent`（`xml=absent|cve_lines=0`），`user` 是 `present`（`cve_lines=2`，K1 分子 +1），`wide` 是 `denied_value`（文件在但读不到，记"策略挡"，不得记"未修复"）。

### 廿七之八 K4 的采出口径（披露滞后天数）

K4 是这十个指标里**唯一不由设备侧读数支撑**的一个，脚本因此只打印占位：

`K4|rollup|disclosure-lag|oem_informed_date=not_device_read|public_date=not_device_read|note=inputs are bulletin-side, MediaTek sentence gives the two-month floor`

三个必填输入都在公告侧：`oem_informed_date`（厂商对 OEM 的知情日）、`public_date`（公开披露日）、`vendor_bulletin_date`（厂商自己的公告/警报日）。

- **下限来自一句官方原句**（第九章已逐字登记）：`Device OEMs have been notified of all the issues and the corresponding security patches for at least two months before publication.` ⇒ 凡厂商公告正文含这句的，`public_date − oem_informed_date ≥ 60` 天是**文档下限**，不是测量值；台账里要写 `k4_source=documented_floor`，不能写成实测。
- **禁止的算法**：不得用 `SPL − 公告日` 充当 K4。SPL 是平台侧口径，与本域器件缺陷不同本账（第三章 L-3 条：厂商公告缺陷不要求也不触发 SPL 变化）；用它算出来的"滞后"是两套口径的差，不是时间差。
- **同一条目的两次日期取不到时**：K4 记 `unknown` 并把该条目从分母里**留着**（同第廿六章 K2 的规则：分母不裁剪），另在 `unavailable_reason` 写 `policy`/`build`/`vendor_dependent`/`not_in_tree`/`no_exit` 五档之一。

### 廿七之九 三个画像的分支覆盖实测

| 画像 | 触发的 P-id 数 | 记录行数 | K2 `num` | 该画像出现的取值种数 | 只有该画像出现的取值 |
| --- | --- | --- | --- | --- | --- |
| `good` | 26 / 26 | 70 | 10 / 26 | 12 | 无（`good` 的取值集合是 `user` 的子集） |
| `user` | 26 / 26 | 70 | 10 / 26 | 16 | `muted_by_build`、`filtered_by_policy`、`bound`、`degraded_by_vendor` |
| `wide` | 26 / 26 | 70 | 13 / 26 | 7 | 无（它的用途是把 P-01/P-04/P-06/P-14/P-18/P-22/P-24 的 `ok` 分支跑出来） |

三画像的**并集是十六种取值**——即十七值词汇表里除 `not_inferable` 之外的全部（`not_inferable` 不由脚本产生，见廿七之六）。`good` 出现的是 `absent`、`debugfs_unreachable`、`denied_log`、`denied_value`、`eacces_typed`、`eacces_untagged`、`label_unreadable`、`openable`、`present`、`unlistable`、`unlistable_value`、`vendor_dependent`。

三点要从这张表读出来：

1. **三画像并集 = 十六种取值全部出现过**，即十七值词汇表里除 `not_inferable` 之外的全部（`not_inferable` 不由脚本产生，理由见廿七之六）。第廿九章的验收就是拿这条做基准，不是拿真机。
2. **K2 的 `num` 不随可读项数线性增长**：`user` 画像比 `good` 多读到一个 `bound` 和一个模式读数，`num` 仍是 10——因为 K2 的分子按 **P-id 全分支可读** 计。一台设备"多读到一点"通常不会改变这个数，这是第廿九章要写的误区之一（把"我这次多读到两个值"当成覆盖率提升）。
3. **`wide` 的作用是把 `ok` 分支跑出来**：`present` 记录数 32（高于 `good` 的 24），`debugfs_unreachable` 0 次，P-22 那行给出 `intel/vpu/vpu_37xx_v0.0.bin`——也就是本报告七条链里唯一那个"运行期可读的固件名"出口（第廿章 P-22）。它在 `wide` 里成立、在 `good` 与 `user` 里不成立，正好把"这条读数的可得性取决于厂商构建"这句话变成一行可对比的输出。

### 廿七之十 运行前置与已知限制

1. **前置**：`adb` 在 PATH 上；设备已授权 USB 调试；脚本不假设 root，也不尝试提权。多设备时用第一个参数给 serial。
2. **它不改任何东西**：对第廿五章登记过的写侧能力（`firmware_store`、`state_store`、`coredump`、`recovery`、`force_recovery`、`reset_engine`、`fw_trace_*`、`dbgfun`/`dbgopt`）本脚本只取 `stat` 的模式位。样例里那些 `0644`/`0444` 是**模式读数**，不是使用记录。
3. **限制一（`ls -Z`）**：标签面在厂商构建上可达性不一致，脚本把取不到标签的情况单独记 `label_unreadable`，第卅章已把它列为边界项。
4. **限制二（`cat` 大文件）**：P-09/P-10/P-11 用 `cat` 读配置类小文件，不用于镜像体；本脚本**不读任何镜像内容**，因此不产生任何镜像片段的副本。
5. **限制三（`dumpsys`）**：P-26 那三行只记录成功与否和字节数，不解析内容——厂商 `dumpsys` 输出的字段命名没有跨厂约定，解析会把厂商差异写成结论。
6. **限制四（画像不等于真机）**：脚本在本章的可执行性是靠 stub 证明的；真机上还会遇到"命令不存在"（`command not found`）这类第三种错误形态，脚本已把它归入 `其它 ⇒ vendor_dependent`，但真机分布本报告不声明。

### 廿七之十一 本章结论

1. **第廿章那张表现在是可执行的**：26 行表 → 26 个区块 → 每次运行 26 个 P-id 全部产出记录（三画像实测各 26/26），"表里有、脚本里没有"这条缺陷在本篇不成立。
2. **脚本给不出固件版本号**——这不是缺陷，是本章之前十九、廿两章结论的可执行复现：七十行输出里没有任何一行 `verdict` 携带版本语义。P-22 给出的是**文件名**，P-14 若成立给出的也是**文件名**（`cdsp.mdt`），两者都不是版本。
3. **`unlistable_value` 转正、`not_inferable` 留给分析者**，词汇表从十六值改成十七值这件事在正文里做，不留到勘误。
4. **K1 只看 T3，K4 只看公告侧**：这两个指标的采出口径都不依赖设备读数的"努力程度"，第廿六章把 T3 说成 OEM 自愿的机制选择，在这里变成 P-11 那一行的四态之一。
5. **第廿八章给出脚本 2**：它拿脚本 1 的两次快照做差，专查三件事——`driver_override` 伴读（第廿一章 FT6 的强制伴读）、rpmsg/glink 边数变化、写侧模式漂移（`0644 → 0444 ⇒ degraded_by_vendor`，第廿六章之二第 6 条），并按本章同一契约输出。

## 廿八、脚本 2：两次快照的漂移比对，二十一个检查块与 K6/K7/K8 的采出口径

第廿七章的脚本 1 是一台"只问一次的相机"。本章的脚本 2 是暗房里的比对器：它不碰设备，只把两张底片叠在一起，指出哪一格变了，以及变了的那一格应该登记什么词。

### 廿八之一 输入契约：脚本 2 唯一的输入是脚本 1 的输出文件

```
w17-drift.sh <snapshot-before> <snapshot-after>
```

四条输入侧的硬事实，先讲清楚，后面所有口径都建立在这上面。

1. **它不执行 `adb`，一次都不执行。** 脚本正文里没有任何设备命令，只有 `awk`、`sort`、`comm`、`grep`。这不是偷懒，是本章的全部要点：漂移的定义是"同一台设备、同一套姿势、两次采集"，比对时追加的任何新命令都会把"设备变了"与"我这次多问了一句"混成一团。第廿二章之二规定"判定条件只能来自实测读数或脚本 2 的两次快照之差"，本节就是那条规则的物理形态——脚本 2 只能看见脚本 1 已经记下的东西。
2. **输入是脚本 1 的完整输出文件**（70 行 = 1 行表头 + 60 条 `P-` 记录 + 3 条 rollup + 6 行禁止推出，见第廿七章之四）。表头与 rollup 行不参与比对：所有取值函数都要求 `$1 ~ /^P-/` 或 `$1=="P-nn"`，`K2|rollup|…` 这类行天然被排除。这不依赖注释，是取值条件本身的性质。
3. **顺序有意义，比对不对称。** `before → after` 与 `after → before` 会产生不同的取值：`value_gained` 与 `value_lost` 是两条不同的记录，`t3_appeared` 与 `t3_lost` 对 K1 分子的作用方向相反，而第廿六章之二第 6 条的降级只由"后一侧 `0644 → 0444`"触发。本章实测时把每一对都跑了两个方向（之九）。
4. **两个快照必须出自同一份脚本 1。** 脚本 2 不检测契约版本：它按列号取值，第 4 列是判定、第 5 列是标签、第 7 列是规则锚点。如果脚本 1 加了列或调了序，脚本 2 不会报错，只会安静地比错格子。**这是本章第四条限制（之十一），也是把两份文件放在同一批交付里的理由。**

退出码只有三个：`2` 是参数或文件缺失，`9` 是契约违规（目前只有一条断言会触发它，见之九的契约断言一段），其余全是 `0`。脚本 2 不会因为"漂移很多"而非零退出——漂移不是错误，这是六行禁止推出里的第一行。

### 廿八之二 输出契约：八列、二十一个检查块、四十二条记录

```
did|face|key|before|after|verdict|rule|note
```

与脚本 1 的八列同宽，但语义全部换过：`pid / target / label / err / source` 五列压成 `key / before / after` 三列，腾出的两格给 `rule` 与 `note`。逐列口径：

- **`did`** 检查块编号，`D-01` 到 `D-19`，其中 `D-13b` 与 `D-14b` 是同一处的两个不同对象（debugfs 与其中的 `fw_name` 一格、属性过滤视图与三件 Android 侧口径），所以块数是 21 而不是 19。最后一节注释写作 `D-20`，它只产出 rollup，不产出 `D-` 记录。
- **`face`** 沿用第廿九章的 V1–V8 面号；跨面或全局块用 `ALL`（D-01、D-02）与 `DECL`（D-17，声明与服务面横跨 V3/V8）。**这两个词不在 V1–V8 白名单里**，它们表示"这一格不属于任何单一读数面"，而不是第九种面。
- **`key`** 单格块用语义名（`P-15|writable-mode`、`P-17|bind-witness`），循环块用 `P-id` 与脚本 1 的 target 拼接。渲染结果里看到的 `P-12 /sys/class/accel` 是 `san` 把 `|` 换成空格之后的样子——契约要求每行恰好八个字段，所以任何字段内部都不允许再出现 `|`。这不是笔误，是转义。
- **`before` / `after`** 是"判定 + 标签"的拼接（`denied_value -`、`present cve_lines=2`），**不是原始行**。要看原始行请回到快照文件；比对器故意只带两格，因为八列宽装不下整行。
- **`verdict`** 漂移取值，第三套词汇表（之七）。它与脚本 1 的十七值词汇表是**正交**的两套：`before`/`after` 里出现的仍是脚本 1 的取值，本列出现的是"这两个取值之间的关系"。
- **`rule`** 管住这一格的规则文件:行号，直接取自快照里对应记录的 `rule` 列；取不到时用与本块同源的默认锚点兜底。兜底值全部来自第廿五章四张能力事实表与第十七至十九章已核验的行号，没有一个新行号。
- **`note`** 这一格变化允许推出什么、禁止推出什么。

**每个块产几条记录？** 这是固定的，不随设备变化：

| 块 | 盯的是哪一格 | 记录条数 | 对应的脚本 1 姿势 |
| --- | --- | --- | --- |
| D-01 | 两次采集的 target 全集 | 1 | 全部 26 条 |
| D-02 | 逐面"面:判定"分布 | 1 | 全部 |
| D-03 | P-02 静态候选名命中集合 | 1 | P-02 |
| D-04 | P-02 命中名的字节数 | 1 | P-02 |
| D-05 | P-05 标签面地基 | 1 | P-05 |
| D-06 | 五类拓扑目录的条目数 | 5 | P-12 |
| D-07 | 四类父链落点 | 4 | P-13 |
| D-08 | remoteproc 两个属性值面 | 2 | P-14 |
| D-09 | 四个可写属性的模式 | 1 | P-15 |
| D-10 | 模块参数三态 | 2 | P-16 |
| D-11 | 五个节点的拒绝成因 | 5 | P-19、P-20 |
| D-12 | 驱动绑定见证 | 1 | P-17 |
| D-13 | debugfs 整块可达性 | 1 | P-21 |
| D-13b | 运行期固件名那一格 | 1 | P-22 |
| D-14 | 属性过滤视图 | 1 | P-07 |
| D-14b | 三件 Android 侧口径 | 3 | P-08 |
| D-15 | T3 记账面与 K1 分子 | 1 | P-11 |
| D-16 | 内核日志通道 | 1 | P-24 |
| D-17 | 声明与服务面七格 | 7 | P-09、P-10、P-25、P-26 |
| D-18 | 被点名允许的对照节点 | 1 | P-23 |
| D-19 | driver_override 伴读在不在台账里 | 1 | 无（这正是它的结论） |
| 合计 | — | **42** | 21 块覆盖 26 条姿势中的 21 条 |

42 条记录 + 5 条 rollup + 6 行禁止推出 + 1 行表头 = **54 行**。循环块的条数由 target 集合决定而不是由设备决定：一台没有 `atomisp` 的机器与一台有的，在 D-10 上都贡献 2 条（`/sys/module/atomisp/parameters/` 与 `/sys/module/ivpu/parameters/`），差别只体现在 `verdict` 是 `stable` 还是 `param_face_drift`。**这条性质是好事也是限制**：输出可预期、可 diff，但也意味着脚本 2 永远不会"因为发现新东西而变长"——发现新东西是脚本 1 的职责，而加姿势必须先改第廿章的表。

**没有专属块的五条姿势是 P-01、P-03、P-04、P-06、P-18**，共 8 条记录，全部属于"目录能不能列、里面有几项"这一类。它们的漂移只有两个去处：`D-01` 的 target 全集（改了姿势才会变，采集本身不会），以及 `D-02` 的面分布（`V1:unlistable` 与 `V1:present` 的条数会变）。这不是漏，是一次刻意的不对称：**拓扑面的条目数（D-06）可以解释成"这颗处理器在不在"，而目录里的文件名数解释不成任何东西**——第廿七章之五已经写明 `present` 只证明名字匹配，那么"entries 从 2 变 3"同样不能断言"厂商塞进一个新镜像"。给它们一个块，只会多产出一个无法归因的 `*_drift`。代价照实写在之十一第四条限制里：某次真实变化若恰好只落在这五条姿势上，本章的比对器给出的信号是"只有 D-02 变了"，粒度到面为止。

### 廿八之三 脚本正文

324 行，十一个辅助函数（`san`、`rec` 与九个按 `(pid, target)` 取值的 `awk` 包装），21 个具名块加最后一节 `D-20`（只产 rollup）。读正文前先记住三件事，它们决定了正文里为什么有大量看起来多余的兜底：

- **取值一律按 `(pid, target)` 二元组定位。**`val` / `verdicts` / `labelv` / `rulev` 四个函数用的都是同一个条件 `($1"|"$3)==k`，其中 `k` 是 `P-nn|<target>`。定位失败时返回空串，不返回假数据；每个块随后用自己的默认串（`${va:--}`）把空格填成 `-`。所以真实设备上少采了一格，输出里会出现 `-`，而不是静默变成 `stable`。
- **`rule` 列优先取自快照。**每个块先尝试 `rulev` 从 `before` 侧（个别块两侧都试）取锚点，取不到才用兜底常量。兜底常量全部是第十七至十九章与第廿五章四张表里已核验过的"文件:行号"，正文没有引入任何一个新行号——这条限制是第卅章可复核清单能覆盖脚本正文的前提。
- **`san` 是转义器不是清洗器。**它把字段内部的 `|`、换行、制表符换成空格并压掉重复空格，目的是保证每行恰好八个字段。之一定义过这个契约；之六里 `companion driver_override` 那个空格就是它留下的可见痕迹。

正文里的 `# ---- D-nn ... ----` 分节注释与本章小节的块名一一对应，读者可以按块跳读。唯一一处不能按块跳读的是第 7 行的 `LC_ALL=C; export LC_ALL`：它必须在任何 `sort` / `comm` 之前生效，之十一第 4 条解释过原因。

```bash
#!/usr/bin/env bash
# w17-drift.sh — 脚本 2：两次"脚本 1"快照的只读漂移比对（离线，不碰设备）
# 用法: w17-drift.sh <snapshot-before> <snapshot-after>
# 输入就是脚本 1 的输出文件。本脚本不执行 adb，因此它读不到任何新东西；
# 它只回答一个问题：两次采集之间，哪个字段变了，变了的那一格应该登记什么。
set -u
LC_ALL=C; export LC_ALL

A="${1:-}"; B="${2:-}"
if [ -z "$A" ] || [ -z "$B" ]; then
  echo "usage: $0 <snapshot-before> <snapshot-after>" >&2
  exit 2
fi
[ -f "$A" ] || { echo "snapshot-before not readable: $A" >&2; exit 2; }
[ -f "$B" ] || { echo "snapshot-after not readable: $B" >&2; exit 2; }

REC=""
san() { printf '%s' "$1" | tr '|\n\t' '   ' | sed 's/  */ /g; s/^ //; s/ $//'; }
rec() {
  if [ "$#" -ne 8 ]; then echo "BUG: rec arity $#! $*" >&2; exit 9; fi
  REC="$REC$(printf '%s|%s|%s|%s|%s|%s|%s|%s\n' \
    "$(san "$1")" "$(san "$2")" "$(san "$3")" "$(san "$4")" "$(san "$5")" "$(san "$6")" "$(san "$7")" "$(san "$8")")
"
}
# val <file> <pid|target> <field-number>  ：取某条记录的某一列
val() { awk -F'|' -v k="$2" -v n="$3" '($1"|"$3)==k {print $n; exit}' "$1"; }
pidtgt() { awk -F'|' '$1 ~ /^P-/ {print $1"|"$3}' "$1" | sort -u; }
verdicts() { awk -F'|' -v k="$2" '($1"|"$3)==k {print $4; exit}' "$1"; }
labelv() { awk -F'|' -v k="$2" '($1"|"$3)==k {print $5; exit}' "$1"; }
rulev() { awk -F'|' -v k="$2" '($1"|"$3)==k {print $7; exit}' "$1"; }
# rows_for <file> <pid> ：列出该 pid 的全部 target
rows_for() { awk -F'|' -v p="$1" '$1==p {print $3}' "$2" | sort -u; }
cnt_of() { awk -F'|' -v p="$1" -v v="$3" '$1==p && $4==v' "$2" | wc -l | tr -d ' '; }
# faces_of <file> ：逐面统计"面:判定"对的条数。判定不进键，则同一面内部的两涨两跌会被算成"分布未变"。
faces_of() { awk -F'|' '$1 ~ /^P-/ {print $2":"$4}' "$1" | sort | uniq -c | awk '{print $2"="$1}' | sort | tr '\n' ' '; }
# num_of <file> <pid> <label-prefix> ：从 label 列里抠出整数
num_of() { awk -F'|' -v p="$1" '$1==p { if (match($5, /[0-9]+/)) { print substr($5, RSTART, RLENGTH); exit } }' "$2"; }

# ---- 表头：与脚本 1 一样，第一行就是契约 ----
echo 'did|face|key|before|after|verdict|rule|note'

# ---- D-01 记录集合本身：两次采集的行数与 target 全集 ----
TA=$(pidtgt "$A" | wc -l | tr -d ' '); TB=$(pidtgt "$B" | wc -l | tr -d ' ')
ONLYA=$(comm -23 <(pidtgt "$A") <(pidtgt "$B") | tr '\n' ' ')
ONLYB=$(comm -13 <(pidtgt "$A") <(pidtgt "$B") | tr '\n' ' ')
if [ "$ONLYA" = "" ] && [ "$ONLYB" = "" ]; then
  rec D-01 ALL record-set "targets=$TA" "targets=$TB" stable ch20:9-contract '两侧 target 全集一致，下面的比对才有意义'
else
  rec D-01 ALL record-set "only_before=$(san "$ONLYA") t=$TA" "only_after=$(san "$ONLYB") t=$TB" record_set_drift \
    'ch20:9-contract' '采集面本身变了：先问姿势有没有改，再问设备有没有变'
fi

# ---- D-02 逐面（V1–V8）判定分布 ----
FA=$(faces_of "$A"); FB=$(faces_of "$B")
if [ "$FA" = "$FB" ]; then
  rec D-02 ALL face-verdict-tally "$FA" "$FB" stable 'ch26:K2' '各面判定分布未变（分布相同不等于逐条相同，见 D-03 起）'
else
  rec D-02 ALL face-verdict-tally "$FA" "$FB" face_drift 'ch26:K2' '某个面的判定分布变了，按面定位到具体 P-id'
fi

# ---- D-03 P-02 静态候选名命中集合 ----
PN() { awk -F'|' -v v=present '$1=="P-02" && $4==v {gsub(".*/","",$3); print $3}' "$1" | sort; }
PA=$(PN "$A"); PB=$(PN "$B")
GA=$(comm -23 <(printf '%s\n' "$PA") <(printf '%s\n' "$PB") | tr '\n' ',')
GB=$(comm -13 <(printf '%s\n' "$PA") <(printf '%s\n' "$PB") | tr '\n' ',')
RULE02=$(rulev "$A" 'P-02|/vendor/firmware_mnt/image/cdsp.mdt'); RULE02=${RULE02:-domain.te:185-193}
if [ -z "$GA" ] && [ -z "$GB" ]; then
  rec D-03 V1 P-02-present-set "present=$(printf '%s\n' "$PA" | grep -c '[^[:space:]]')" "present=$(printf '%s\n' "$PB" | grep -c '[^[:space:]]')" stable "$RULE02" '命中名集合相同；注意 size 列仍可能各自不同（见 D-04）'
else
  rec D-03 V1 P-02-present-set "gone=$(san "$GA")" "appeared=$(san "$GB")" name_set_drift "$RULE02" '名字出现或消失只是文件名命中变化，不构成装载或版本证据'
fi

# ---- D-04 P-02 命中名的 size 漂移 ----
SIZES=""
for t in $(rows_for P-02 "$A"); do
  va=$(labelv "$A" "P-02|$t"); vb=$(labelv "$B" "P-02|$t")
  case "$va-$vb" in *size=*-*size=*) ;; *) continue ;; esac
  na=${va#size=}; nb=${vb#size=}
  [ "$na" = "$nb" ] || SIZES="$SIZES${t##*/}:$na->$nb,"
done
if [ -z "$SIZES" ]; then
  rec D-04 V1 P-02-size 'all-equal' 'all-equal' stable "$RULE02" '两侧命中的同名文件字节数一致'
else
  rec D-04 V1 P-02-size 'see-note' "$SIZES" size_drift "$RULE02" '字节数变了。本报告不据此推版本：同版本可重打包，不同版本可同长度'
fi

# ---- D-05 P-05 标签面覆盖 ----
LA=$(verdicts "$A" 'P-05|label-view'); LB=$(verdicts "$B" 'P-05|label-view')
NA=$(num_of P-05 "$A"); NB=$(num_of P-05 "$B")
RL5=$(rulev "$A" 'P-05|label-view'); RL5=${RL5:-shell.te:424}
case "$LA/$LB" in
  present/present) if [ "$NA" = "$NB" ]; then
      rec D-05 V1 P-05-label-view "$LA lines=$NA" "$LB lines=$NB" stable "$RL5" '地基仍在：两种 eacces 的区分仍可用'
    else
      rec D-05 V1 P-05-label-view "$LA lines=$NA" "$LB lines=$NB" label_coverage_drift "$RL5" '标签行数变了，D-11 的两种 eacces 区分要先重验'
    fi ;;
  present/*|*/present) rec D-05 V1 P-05-label-view "$LA" "$LB" label_base_lost "$RL5" '地基一侧塌了：本批漂移里所有成因分类都降级为不可比' ;;
  *) rec D-05 V1 P-05-label-view "$LA" "$LB" label_unreadable_both "$RL5" '两侧都取不到标签，本域成因分解不可执行，只登记不推' ;;
esac

# ---- D-06 P-12 拓扑节点数 ----
for t in $(rows_for P-12 "$A"); do
  va=$(verdicts "$A" "P-12|$t"); vb=$(verdicts "$B" "P-12|$t")
  na=$(awk -F'|' -v k="P-12|$t" '($1"|"$3)==k {if (match($5,/entries=[0-9]+/)) print substr($5, RSTART+8, RLENGTH-8)}' "$A")
  nb=$(awk -F'|' -v k="P-12|$t" '($1"|"$3)==k {if (match($5,/entries=[0-9]+/)) print substr($5, RSTART+8, RLENGTH-8)}' "$B")
  rl=$(rulev "$A" "P-12|$t"); rl=${rl:-shell.te:443}
  if [ "$va" = "$vb" ] && [ "${na:-x}" = "${nb:-x}" ]; then
    rec D-06 V4 "P-12|$t" "$va ${na:+entries=$na}" "${vb} ${nb:+entries=${nb}}" stable "$rl" '拓扑这一格未变'
  else
    rec D-06 V4 "P-12|$t" "$va ${na:+entries=$na}" "$vb ${nb:+entries=$nb}" topology_drift "$rl" '节点数或可达性变了；这是七条链里最强的一档读数，变化要单独归因'
  fi
done

# ---- D-07 P-13 符号链接落点（父链） ----
for t in $(rows_for P-13 "$A"); do
  la=$(labelv "$A" "P-13|$t"); lb=$(labelv "$B" "P-13|$t")
  rl=$(rulev "$A" "P-13|$t"); rl=${rl:-domain.te:196-197}
  if [ "$la" = "$lb" ]; then
    rec D-07 V4 "P-13|$t" 'same-path' 'same-path' stable "$rl" '父链落点一致，平台设备名稳定'
  else
    rec D-07 V4 "P-13|$t" "$la" "$lb" symlink_target_drift "$rl" '落点变了 = 平台节点改名或换控制器，DT 节点名会跟着变（第十九章之五）'
  fi
done

# ---- D-08 P-14 数值面：读到 / 读不到 ----
for t in $(rows_for P-14 "$A"); do
  va=$(verdicts "$A" "P-14|$t"); vb=$(verdicts "$B" "P-14|$t")
  la=$(labelv "$A" "P-14|$t"); lb=$(labelv "$B" "P-14|$t")
  rl=$(rulev "$A" "P-14|$t"); rl=${rl:-remoteproc_sysfs.c:167}
  if [ "$va" = "$vb" ]; then
    rec D-08 V4 "P-14|$t" "$va $la" "$vb $lb" stable "$rl" '数值面这一格判定一致'
  elif [ "$vb" = present ]; then
    rec D-08 V4 "P-14|$t" "$va" "$vb $lb" value_gained "$rl" '后一次读到了值：先查是不是厂商补授权，不要直接记成器件变化'
  elif [ "$va" = present ]; then
    rec D-08 V4 "P-14|$t" "$va $la" "$vb" value_lost "$rl" '前一次读到、这次读不到：可能是构建差异，也可能是 dontaudit 静音，见第廿章之七'
  else
    rec D-08 V4 "P-14|$t" "$va" "$vb" verdict_drift "$rl" '两种失败态互换，成因分类需要重跑 P-05'
  fi
done

# ---- D-09 P-15 写侧属性模式漂移（降级触发二） ----
va=$(verdicts "$A" 'P-15|rproc.writable-attrs'); vb=$(verdicts "$B" 'P-15|rproc.writable-attrs')
la=$(labelv "$A" 'P-15|rproc.writable-attrs'); lb=$(labelv "$B" 'P-15|rproc.writable-attrs')
case "$la/$lb" in
  coredump_mode=644/coredump_mode=444)
    rec D-09 V4 'P-15|writable-mode' "$va $la" "$vb $lb" mode_tightened 'remoteproc_sysfs.c:236-240' '0644 -> 0444：本树 rproc->sysfs_read_only 赋值点为 0（第十九章之五），所以有未取回的厂商补丁在生效；这一格是第廿六章之二第 6 条的降级触发器，该链 L 级降一级，不多也不少' ;;
  coredump_mode=444/coredump_mode=644)
    rec D-09 V4 'P-15|writable-mode' "$va $la" "$vb $lb" mode_relaxed 'remoteproc_sysfs.c:236-240' '回到 AOSP 默认模式：不升也不降。它把"能不能写"重新变成开放问题，要单独开工单，但不是判级变化' ;;
  *) if [ "$la" = "$lb" ] && [ -n "$la" ]; then
       rec D-09 V4 'P-15|writable-mode' "$va $la" "$vb $lb" stable 'remoteproc_sysfs.c:229-243' '模式一致（一致不等于可写：值面在 D-08）'
     elif [ -z "$la" ] || [ -z "$lb" ] || [ "$la" = "-" ] || [ "$lb" = "-" ]; then
       rec D-09 V4 'P-15|writable-mode' "$va ${la:--}" "$vb ${lb:--}" mode_unavailable 'remoteproc_sysfs.c:229-243' '至少一侧没有模式这一栏，本条不参与模式漂移统计，也不构成"另一侧被收紧"的证据'
     else
       rec D-09 V4 'P-15|writable-mode' "$va $la" "$vb $lb" mode_unclassified 'remoteproc_sysfs.c:229-243' '模式确实变了，但不是 0444 那一格：规则 6 只登记 sysfs_read_only 对应的值，其它组合需要新锚点，所以记为边界而不是降级'
     fi ;;
esac

# ---- D-10 P-16 模块参数三态（目录/模式/值） ----
for t in $(rows_for P-16 "$A"); do
  va=$(verdicts "$A" "P-16|$t"); vb=$(verdicts "$B" "P-16|$t")
  la=$(labelv "$A" "P-16|$t"); lb=$(labelv "$B" "P-16|$t")
  rl=$(rulev "$A" "P-16|$t"); rl=${rl:-genfs_contexts:123-124}
  if [ "$va" = "$vb" ] && [ "$la" = "$lb" ]; then
    rec D-10 V4 "P-16|$t" "$va $la" "$vb $lb" stable "$rl" '三态组合一致；值面本域始终不可读（第廿章 P-16）'
  else
    rec D-10 V4 "P-16|$t" "$va ${la:--}" "$vb ${lb:--}" param_face_drift "$rl" '模块目录装载状态变了（未装载 / 已装载但值被拒 是两件事），禁止读成"开关被用过"'
  fi
done

# ---- D-11 两种 eacces 的成因迁移（P-19 / P-20） ----
for p in P-19 P-20; do
 for t in $(rows_for "$p" "$A"); do
  k="$p|$t"
  va=$(verdicts "$A" "$k"); vb=$(verdicts "$B" "$k")
  la=$(labelv "$A" "$k"); lb=$(labelv "$B" "$k")
  rl=$(rulev "$A" "$k"); rl=${rl:-file_contexts:97}
  [ -n "$va" ] || continue
  if [ "$va" = "$vb" ] && [ "$la" = "$lb" ]; then
    rec D-11 V5 "$k" "$va ${la:--}" "$vb ${lb:--}" stable "$rl" '成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致'
  elif [ "$la" = label_unreadable ] || [ "$lb" = label_unreadable ]; then
    rec D-11 V5 "$k" "$va ${la:--}" "$vb ${lb:--}" cause_indeterminate "$rl" '任一侧标签取不到，成因分解在这个节点上不成立。判定相同而标签一侧退化为 label_unreadable 也记这一档：换掉的是证据，不是性质'
  else
    rec D-11 V5 "$k" "$va ${la:--}" "$vb ${lb:--}" cause_migration "$rl" '拒绝态或标签类型在两次快照之间迁移（含 typed 与 untagged 互换）：先重跑 P-05 再归因；处置成本可能从一步变两步，见第廿章之七'
  fi
 done
done

# ---- D-12 P-17 绑定见证（dbg 文件存在性） ----
va=$(verdicts "$A" 'P-17|atomisp-isp2'); vb=$(verdicts "$B" 'P-17|atomisp-isp2')
la=$(labelv "$A" 'P-17|atomisp-isp2'); lb=$(labelv "$B" 'P-17|atomisp-isp2')
if [ "$va" = "$vb" ]; then
  rec D-12 V4 'P-17|bind-witness' "$va ${la:--}" "$vb ${lb:--}" stable 'atomisp_drvfs.c:164-172' '绑定见证一致'
else
  rec D-12 V4 'P-17|bind-witness' "$va ${la:--}" "$vb ${lb:--}" bind_drift 'atomisp_drvfs.c:164-172' '驱动绑定态变了；这仍是在场性证据，不是固件证据'
fi

# ---- D-13 debugfs 面（P-21 / P-22）：构建因 vs 厂商因 ----
va=$(verdicts "$A" 'P-21|debugfs'); vb=$(verdicts "$B" 'P-21|debugfs')
la=$(labelv "$A" 'P-21|debugfs'); lb=$(labelv "$B" 'P-21|debugfs')
rl21=$(rulev "$A" 'P-21|debugfs'); rl21=${rl21:-te_macros:591}
case "$va/$vb" in
  muted_by_build/debugfs_unreachable|debugfs_unreachable/muted_by_build)
    rec D-13 V5 'P-21|debugfs' "$va $la" "$vb $lb" build_switch "$rl21" '构建类型变了（user <-> userdebug），不是厂商收紧；不进"厂商收紧"统计' ;;
  vendor_dependent/debugfs_unreachable|debugfs_unreachable/vendor_dependent)
    rec D-13 V5 'P-21|debugfs' "$va $la" "$vb $lb" debugface_drift "$rl21" '同为非 user 构建而可达性变了：这才是可以问厂商的那一档' ;;
  *) if [ "$va" = "$vb" ]; then
       rec D-13 V5 'P-21|debugfs' "$va $la" "$vb $lb" stable "$rl21" 'debugfs 面判定一致'
     else
       rec D-13 V5 'P-21|debugfs' "$va $la" "$vb $lb" debugface_drift "$rl21" '判定换了类别，按第廿章之七先分构建再分厂商'
     fi ;;
esac
# P-22 唯一一个运行期固件名出口（两侧姿势目标名不同：读不到时是 fw_name-lookup，读到时是那个文件本身）
FWT=/sys/kernel/debug/accel/accel0/fw_name
va=$(verdicts "$A" "P-22|$FWT"); [ -n "$va" ] || va=$(verdicts "$A" 'P-22|fw_name-lookup')
vb=$(verdicts "$B" "P-22|$FWT"); [ -n "$vb" ] || vb=$(verdicts "$B" 'P-22|fw_name-lookup')
la=$(labelv "$A" "P-22|$FWT"); [ "$la" != "-" ] && [ -n "$la" ] || la=$(labelv "$A" 'P-22|fw_name-lookup')
lb=$(labelv "$B" "P-22|$FWT"); [ "$lb" != "-" ] && [ -n "$lb" ] || lb=$(labelv "$B" 'P-22|fw_name-lookup')
rl22=$(rulev "$A" "P-22|$FWT"); [ -n "$rl22" ] || rl22=$(rulev "$B" "P-22|$FWT")
[ -n "$rl22" ] || rl22=$(rulev "$A" 'P-22|fw_name-lookup'); rl22=${rl22:-ivpu_debugfs.c:30-36}
case "$va/$vb" in
  present/present)
    rec D-13b V5 'P-22|fw_name' "present $la" "present $lb" stable "$rl22" '两侧都读到运行期固件名。stable 只表示"两次都读到"：值文本是否相同要看 before/after 两列，脚本 2 不比值文本（同 D-08，第十七章之二）' ;;
  present/*)
    rec D-13b V5 'P-22|fw_name' "present $la" "${vb:--} ${lb:--}" fwname_lost "$rl22" '从可得变不可得：先排除采集窗口（重启后 debugfs 未挂）与授权收回，不得直接写成"固件名被抹掉"' ;;
  */present)
    rec D-13b V5 'P-22|fw_name' "${va:--} ${la:--}" "present $lb" fwname_appeared "$rl22" '运行期固件名这一格从不可得变为可得：它是文件名字段，不是版本字段（第廿三章之八第（6）条）' ;;
  *)
    rec D-13b V5 'P-22|fw_name' "$va $la" "$vb $lb" fwname_unavailable "$rl22" '两侧都拿不到运行期固件名；本域的该出口默认不成立，这一格记的是状态不是变化' ;;
esac

# ---- D-14 属性面（P-07）与三件 Android 侧口径（P-08） ----
va=$(verdicts "$A" 'P-07|getprop-fw-keys'); vb=$(verdicts "$B" 'P-07|getprop-fw-keys')
la=$(labelv "$A" 'P-07|getprop-fw-keys'); lb=$(labelv "$B" 'P-07|getprop-fw-keys')
if [ "$va" = "$vb" ]; then
  rec D-14 V2 'P-07|getprop-filter' "$va ${la:--}" "$vb ${lb:--}" stable 'property.te:191-198' '过滤视图一致。注意 ${la} 里的命中可能是子串误命中（第廿二章之三），别把计数变化当新增上报'
else
  rec D-14 V2 'P-07|getprop-filter' "$va ${la:--}" "$vb ${lb:--}" filter_drift 'property.te:191-198' '可读键数变了，先确认是构建（user 过滤更严）还是厂商改了属性上下文'
fi
for t in $(rows_for P-08 "$A"); do
  la=$(labelv "$A" "P-08|$t"); lb=$(labelv "$B" "P-08|$t")
  [ -n "$la" ] || continue
  if [ "$la" = "$lb" ]; then
    rec D-14b V2 "P-08|$t" "$la" "$lb" stable 'property.te:191-198' 'Android 侧口径未变'
  else
    rec D-14b V2 "P-08|$t" "$la" "$lb" android_side_drift 'property.te:191-198' 'Android 侧口径变了。它只反映系统分区，禁止用来反推器件镜像新旧（第八章）'
  fi
done

# ---- D-15 T3 记账面（P-11）与 K1 分子的漂移 ----
va=$(verdicts "$A" 'P-11|supplemental_security_patches'); vb=$(verdicts "$B" 'P-11|supplemental_security_patches')
na=$(awk -F'|' '$1=="P-11" {if (match($5,/cve_lines=[0-9]+/)) print substr($5, RSTART+10, RLENGTH-10); else print 0}' "$A" | head -1)
nb=$(awk -F'|' '$1=="P-11" {if (match($5,/cve_lines=[0-9]+/)) print substr($5, RSTART+10, RLENGTH-10); else print 0}' "$B" | head -1)
case "$va/$vb" in
  absent/present|denied_value/present)
    rec D-15 V3 'P-11|supplemental_xml' "$va cve_lines=${na:-0}" "$vb cve_lines=${nb:-0} k1_numerator_plus" t3_appeared 'file_contexts:427' 'T3 记账面出现：K1 分子可能 +1，但仍要求编号与公告主键逐字相等' ;;
  present/absent|present/denied_value)
    rec D-15 V3 'P-11|supplemental_xml' "$va cve_lines=${na:-0}" "$vb cve_lines=${nb:-0} k1_numerator_reset" t3_lost 'file_contexts:427' '记账面消失或被挡：不得据此写"未修复"（第廿四章之十五）' ;;
  *) if [ "$va" = "$vb" ]; then
       rec D-15 V3 'P-11|supplemental_xml' "$va cve_lines=${na:-0}" "$vb cve_lines=${nb:-0}" stable 'file_contexts:427' 'T3 面状态一致'
     else
       rec D-15 V3 'P-11|supplemental_xml' "$va" "$vb" t3_drift 'file_contexts:427' '状态换了类别但未落入已知迁移对，登记为边界'
     fi ;;
esac

# ---- D-16 日志面（P-24）----
va=$(verdicts "$A" 'P-24|dmesg'); vb=$(verdicts "$B" 'P-24|dmesg')
na=$(num_of P-24 "$A"); nb=$(num_of P-24 "$B")
if [ "$va" = "$vb" ]; then
  rec D-16 V6 'P-24|dmesg' "$va ${na:+matches=$na}" "$vb ${nb:+matches=$nb}" stable 'app.te:660-661' '日志面一致；matches 计数含子串误命中，不作趋势用'
else
  rec D-16 V6 'P-24|dmesg' "$va ${na:+matches=$na}" "$vb ${nb:+matches=$nb}" log_channel_drift 'app.te:660-661' '日志通道可达性变了 = 采集窗口条件第 4 项变了，K7 分子要重算'
fi

# ---- D-17 声明与服务面（P-09/P-10/P-25/P-26）----
for t in 'P-09|media_codecs' 'P-10|vintf-manifest' 'P-25|service-list' 'P-25|lshal' 'P-26|dumpsys gpu' 'P-26|dumpsys media.codec' 'P-26|dumpsys cameraserver'; do
  va=$(verdicts "$A" "$t"); vb=$(verdicts "$B" "$t")
  la=$(labelv "$A" "$t"); lb=$(labelv "$B" "$t")
  rl=$(rulev "$A" "$t"); rl=${rl:-shell.te:358}
  [ -n "$va" ] || continue
  if [ "$va" = "$vb" ] && [ "$la" = "$lb" ]; then
    rec D-17 DECL "${t#*|}" "$va ${la:--}" "$vb ${lb:--}" stable "$rl" '声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）'
  else
    rec D-17 DECL "${t#*|}" "$va ${la:--}" "$vb ${lb:--}" declaration_drift "$rl" '声明层读数变了（HAL 接口版本不是固件版本），需要 O5 更新资产口径而不是 O1 开工单'
  fi
done

# ---- D-18 /dev/ion 能力对照（P-23）----
va=$(verdicts "$A" 'P-23|/dev/ion'); vb=$(verdicts "$B" 'P-23|/dev/ion')
if [ "$va" = "$vb" ]; then
  rec D-18 V5 'P-23|/dev/ion' "$va" "$vb" stable 'shell.te:463' 'AOSP 明文允许的那条对照项一致'
else
  rec D-18 V5 'P-23|/dev/ion' "$va" "$vb" named_capability_drift 'shell.te:463' '被点名节点的授权态变了：与厂商 shell 授权面改动直接相关（第廿章之五）'
fi

# ---- D-19 driver_override 伴读：台账里有没有这一格 ----
# 这里刻意不写"有伴读"分支。脚本 1 的 26 条姿势里没有 driver_override（第廿七章之二），
# 所以"伴读存在"在当前契约下不是一个可能的取值，而是一个契约违规。
# 违规要用 exit 9 打断，不能用一条 verdict 把它咽下去。
DO=$(awk -F'|' '$1 ~ /^P-/ {print $5"|"$3}' "$A" "$B" | grep -c 'driver_override' || true)
if [ "${DO:-0}" != 0 ]; then
  echo "BUG: driver_override found in $DO row(s) but it is not a posture registered in ch20; add the posture first, do not compare it" >&2
  exit 9
fi
rec D-19 V4 'companion|driver_override' 'absent-in-both' 'absent-in-both' companion_missing 'rpmsg_core.c:449' '脚本 1 不采伴读（26 条姿势里没有它，见第廿七章），因此任何 rpmsg 名字读数都不得进 K8 分子（第廿六章之六 K8）；要补采就得先把它写进第廿章的表，不能偷偷加姿势'

# ---- D-20 汇总：K6 / K7 / K8 与 L 级动作 ----
K6N=$(awk -F'|' '$1=="P-14" && $4=="present" {print $1"|"$3}' "$A" "$B" | sort -u | wc -l | tr -d ' ')
K7L=$(awk -F'|' '$1=="P-24" && $4=="present" {f=1; exit} END{print f+0}' "$A" "$B")
K8N=$(awk -F'|' '$1 ~ /^P-/ && $4=="present" && $5!="-" {print $1"|"$3}' "$A" "$B" | sort -u | wc -l | tr -d ' ')
K8T=$(awk -F'|' '$1 ~ /^P-/' "$A" "$B" | wc -l | tr -d ' ')
DEG=$(printf '%s' "$REC" | awk -F'|' '$1 ~ /^D-/ && $6=="mode_tightened"' | wc -l | tr -d ' ')
BLKU=$(printf '%s' "$REC" | cut -d'|' -f1 | grep '^D-' | sort -u | wc -l | tr -d ' ')
printf '%s' "$REC"
echo "K6|rollup|crash-visibility|numerator_input=$K6N|den=7|note=this row is the numerator INPUT, not the numerator: the ledger carries no chain column, so converting readable state fields into chain counts is done by hand against ch26's seven-chain table. drm_warn lines never enter the numerator and GPU reset_count is a trend term, not a numerator (ch26 K6)"
echo "K7|rollup|capture-windows|log_channel=$K7L|den=4|note=two snapshots can only prove denominator item 4, the log channel. coldboot / decode-session / load-failure are window conditions, not ledger rows, so K7 stays hand-filled; a zero here is not a missing window, it is an unreadable channel (ch26 K7)"
echo "K8|rollup|value-with-source|rows_with_a_value=$K8N|ledger_rows=$K8T|value_source_tag=not-carried-by-script-1|note=K8 needs both a value and a legal value_source. Script 1's source column carries device_read/static_baseline, never fw_reported/driver_set/userspace_override, so this row proves condition one only; condition two and K3's field denominator are registered as ch30 gaps. rpmsg name readings are additionally excluded by D-19 (ch26 K3/K8)"
echo "L|rollup|level-action|level_downgrades=${DEG:-0}|action=apply-ch26-rule-6|note=exactly one level per mode_tightened row, no more and no less. The word degraded_by_vendor also appears inside D-02's face tally, but a tally is not an event and must never be counted twice (ch26:2-6)"
echo "S|rollup|drift-summary|records=$(printf '%s' "$REC" | grep -c '^D-')|distinct_blocks=$BLKU|inputs=$(awk 'END{print NR}' "$A")+/$(awk 'END{print NR}' "$B")|note=comparison-only; this script issues no new device read"
echo '# 1. drift is not an attack signal and not a hardening signal. Every *_drift row above is a change of reading, not a change of security state.'
echo '# 2. a value gained on the after side is not proof the firmware changed; it may be a vendor grant, a build difference, or a window that happened to be open.'
echo '# 3. a value lost on the after side is not proof of a rollback; dontaudit silences refusals, so absence in the later snapshot needs P-05 re-run before any conclusion.'
echo '# 4. name-set and size drift on V1 never become version statements (ch20:8-2, ch23). Record them as file-name observations only.'
echo '# 5. the T3 row (D-15) is the only drift row that may move K1, and only when the bulletin key matches the XML key verbatim (ch23:9-1, ch24:15).'
echo '# 6. two snapshots are not a trend. Any "持续/恶化/好转" wording about the rows above is a report defect (ch26:2-6 registers only the one-level downgrade).'
```

### 廿八之四 四个输入画像：`alt` 不是第二台设备，是同一条链的第二次采集

脚本 1 的四个画像里，前三个在第廿七章已经交代：`good` 是合成开放画像，`user` 是合成 `user` 构建画像，`wide` 是把授权面打开、专门用来跑 `ok` 分支的画像。**本章新增第四个 `alt`，它的性质与前三个不同，必须先说清楚，否则读者会以为本报告做过两次真机采集。**

- **`alt` 是测试夹具，不是设备读数。** 脚本 2 的验收对象是"分支可达"，脚本 2 的验收对象是"分支可达"，而 `good`/`user`/`wide` 三个画像互相配对只能触发 42 条记录里的一部分漂移。为了让每一条 `*_drift` 都至少被触发一次，`alt` 在 `good` 之上**只改四处**，每一处对应一条此前无法到达的分支：

| `alt` 相对 `good` 的唯一改动 | 实测前后值 | 目的分支 | 为什么非它不可 |
| --- | --- | --- | --- |
| `stat` 出来的 `cdsp.mdt` 字节数 | `5242880` → `5246976`（差 4096） | `D-04 size_drift` | 其余画像之间命中的文件名集合不同，走的是 `D-03 name_set_drift`；要走到"名字集合相同、字节数不同"这一格，只能改长度不改名字 |
| `ls -Zd` 的标签行数 | 6 行 → 4 行（少 `/dev/video0` 与 `/vendor_dlkm/lib/modules`） | `D-05 label_coverage_drift`、`D-11 cause_indeterminate` | 判定仍成功（`present`）、只有覆盖面缩了，才能证明"地基还在但变薄"与"地基塌了"是两档 |
| `remoteproc0/state` 的失败原因 | `Permission denied` → `No such file or directory` | `D-08 verdict_drift` | 两种失败态互换、且两侧都不是 `present`，才会落到 `D-08` 的第四分支；`value_gained` / `value_lost` 已经由 `wide` 覆盖 |
| `coredump` / `recovery` 属性的八进制模式 | `644` → `600` | `D-09 mode_unclassified` | `D-09` 只有三条臂：`644→444` 是收紧（走第廿六章第 6 条降级）、`444→644` 是放开、两侧相同是稳定。`600` 既不是 AOSP 默认也不是本树里的收紧值，只有它能证明"模式确实变了但不落进已知迁移对"这一档存在，且不冒充降级触发器 |

- **第卅章的取回记录里没有 `alt`。** 那一份记录只登记从公开来源逐字取回的东西，而 `alt` 没有任何外部来源——它是 `override()` 里四条 `case` 覆盖分支拼出来的桩。这一点写在这里，是为了让"哪一章可以引用哪个画像"这条链在读者面前闭合：第廿七至廿九章的样例输出全部来自桩画像，第卅章的可复核清单只对**源码行号与文档句子**负责，样例的数值不在核对范围内。
- **`alt` 只改四处是有意的。** 桩画像每多改一处，样例里就多一个不可归因的变化；第廿七章之十已经为 `wide` 写下同一条限制（"它是画像不是设备，所以不要用它做跨厂比较"）。四个画像之间的关系是：`good ⊂ user` 的取值集合、`wide` 负责跑 `ok` 分支、`alt` 负责跑"同一格子换了内容"这一类分支。

### 廿八之五 真实输出：`good → user` 的全部 54 行

选这一对做样例的理由有三条，都是可核对的。

其一，**它的 12 条漂移记录全部落在同一条成因上**：一次构建类型切换 `userdebug → user`。这一条同时带动 `D-09` 的写侧模式收紧（`0644 → 0444`，触发第廿六章第 6 条的降一级）、`D-13` 的 debugfs 构建切换、`D-14` 的属性过滤、`D-14b` 的三件 Android 侧口径变化、`D-15` 的 T3 记账面出现，以及 `D-10` / `D-12` 的模块参数面与驱动绑定见证变化。读者做一次月度复采最容易遇到的正是这一类漂移，所以样例应该先展示它。作为对照，`good → wide` 那一对有 23 条漂移记录，比这一对更多，但它的每一条成因都不同（授权面逐条放开），不适合做第一个样例，只适合做分支覆盖。

其二，**它的漂移比最接近正常复采**：42 条记录里 30 条 `stable`、12 条漂移。如果样例里漂移记录占一半以上，读者会误以为脚本 2 默认预期"两次采集差别很大"；而第廿六章之六第 6 条已经写明"两份快照不构成趋势"，样例的行数分布应当与这条纪律一致。

其三，**它新增的 `present` 读数只有两个，且两者都不是器件侧证据**：`D-15` 的 T3 记账面（`absent → present cve_lines=2`，属系统分区记账，第廿四章之十五已经限定它只有在 CVE 编号与公告主键逐字相等时才能动 K1）与 `D-12` 的绑定见证（`absent → bound dbgfun,dbgopt`，是在场性证据不是固件证据，第廿五章之六）。样例里没有任何一格从"读不到"变成"读到了固件版本"——本报告禁止的那个句式，在样例层面就不会出现。

下面是 `.sb17-drift.sh .sb17-out-good.txt .sb17-out-user.txt` 的完整标准输出，逐字节复制，未删一行：

```text
did|face|key|before|after|verdict|rule|note
D-01|ALL|record-set|targets=60|targets=60|stable|ch20:9-contract|两侧 target 全集一致，下面的比对才有意义
D-02|ALL|face-verdict-tally|V1:absent=14 V1:eacces_untagged=1 V1:present=5 V1:unlistable=2 V2:present=4 V3:absent=1 V3:present=2 V4:absent=3 V4:denied_value=3 V4:present=10 V4:unlistable_value=1 V5:debugfs_unreachable=2 V5:eacces_typed=1 V5:eacces_untagged=1 V5:label_unreadable=3 V5:openable=1 V6:denied_log=1 V8:present=3 V8:vendor_dependent=2|V1:absent=14 V1:eacces_untagged=1 V1:present=5 V1:unlistable=2 V2:filtered_by_policy=1 V2:present=3 V3:present=3 V4:absent=2 V4:bound=1 V4:degraded_by_vendor=1 V4:denied_value=3 V4:present=9 V4:unlistable_value=1 V5:debugfs_unreachable=1 V5:eacces_typed=1 V5:eacces_untagged=1 V5:label_unreadable=3 V5:muted_by_build=1 V5:openable=1 V6:denied_log=1 V8:present=3 V8:vendor_dependent=2|face_drift|ch26:K2|某个面的判定分布变了，按面定位到具体 P-id
D-03|V1|P-02-present-set|present=2|present=2|stable|domain.te:185-193|命中名集合相同；注意 size 列仍可能各自不同（见 D-04）
D-04|V1|P-02-size|all-equal|all-equal|stable|domain.te:185-193|两侧命中的同名文件字节数一致
D-05|V1|P-05-label-view|present lines=6|present lines=6|stable|shell.te:424|地基仍在：两种 eacces 的区分仍可用
D-06|V4|P-12 /sys/class/accel|present entries=1|present entries=1|stable|shell.te:443|拓扑这一格未变
D-06|V4|P-12 /sys/class/drm|present entries=2|present entries=2|stable|shell.te:443|拓扑这一格未变
D-06|V4|P-12 /sys/class/remoteproc|present entries=2|present entries=2|stable|shell.te:443|拓扑这一格未变
D-06|V4|P-12 /sys/class/rpmsg|absent|absent|stable|genfs_contexts:123-124|拓扑这一格未变
D-06|V4|P-12 /sys/class/video4linux|present entries=3|present entries=3|stable|shell.te:443|拓扑这一格未变
D-07|V4|P-13 /sys/class/accel/*|same-path|same-path|stable|domain.te:196-197|父链落点一致，平台设备名稳定
D-07|V4|P-13 /sys/class/drm/*|same-path|same-path|stable|domain.te:196-197|父链落点一致，平台设备名稳定
D-07|V4|P-13 /sys/class/remoteproc/*|same-path|same-path|stable|domain.te:196-197|父链落点一致，平台设备名稳定
D-07|V4|P-13 /sys/class/video4linux/*|same-path|same-path|stable|domain.te:196-197|父链落点一致，平台设备名稳定
D-08|V4|P-14 rproc.firmware|denied_value -|denied_value -|stable|remoteproc_sysfs.c:167|数值面这一格判定一致
D-08|V4|P-14 rproc.state|denied_value -|denied_value -|stable|shell.te:443|数值面这一格判定一致
D-09|V4|P-15 writable-mode|present coredump_mode=644|degraded_by_vendor coredump_mode=444|mode_tightened|remoteproc_sysfs.c:236-240|0644 -> 0444：本树 rproc->sysfs_read_only 赋值点为 0（第十九章之五），所以有未取回的厂商补丁在生效；这一格是第廿六章之二第 6 条的降级触发器，该链 L 级降一级，不多也不少
D-10|V4|P-16 /sys/module/atomisp/parameters|absent -|unlistable_value modes=644,644,|param_face_drift|genfs_contexts:182-186|模块目录装载状态变了（未装载 / 已装载但值被拒 是两件事），禁止读成"开关被用过"
D-10|V4|P-16 /sys/module/ivpu/parameters|unlistable_value modes=644,644,644,|absent -|param_face_drift|genfs_contexts:123-124|模块目录装载状态变了（未装载 / 已装载但值被拒 是两件事），禁止读成"开关被用过"
D-11|V5|P-19 /dev/video0|eacces_typed video_device|eacces_typed video_device|stable|shell.te:424|成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致
D-11|V5|P-20 /dev/accel/accel0|label_unreadable -|label_unreadable -|stable|file_contexts:97|成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致
D-11|V5|P-20 /dev/remoteproc0|label_unreadable -|label_unreadable -|stable|file_contexts:97|成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致
D-11|V5|P-20 /dev/rpmsg0|eacces_untagged device|eacces_untagged device|stable|file_contexts:97|成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致
D-11|V5|P-20 /dev/rpmsg_ctrl0|label_unreadable -|label_unreadable -|stable|file_contexts:97|成因不变：同一种拒绝在两次快照里落在同一档，且支撑这一档的标签也一致
D-12|V4|P-17 bind-witness|absent -|bound dbgfun,dbgopt|bind_drift|atomisp_drvfs.c:164-172|驱动绑定态变了；这仍是在场性证据，不是固件证据
D-13|V5|P-21 debugfs|debugfs_unreachable build=userdebug|muted_by_build build=user|build_switch|te_macros:591|构建类型变了（user <-> userdebug），不是厂商收紧；不进"厂商收紧"统计
D-13b|V5|P-22 fw_name|debugfs_unreachable -|debugfs_unreachable -|fwname_unavailable|te_macros:591|两侧都拿不到运行期固件名；本域的该出口默认不成立，这一格记的是状态不是变化
D-14|V2|P-07 getprop-filter|present matches=1|filtered_by_policy -|filter_drift|property.te:191-198|可读键数变了，先确认是构建（user 过滤更严）还是厂商改了属性上下文
D-14b|V2|P-08 ro.bootloader|unknown|unknown|stable|property.te:191-198|Android 侧口径未变
D-14b|V2|P-08 ro.build.version.security_patch|2026-06-05|2026-09-01|android_side_drift|property.te:191-198|Android 侧口径变了。它只反映系统分区，禁止用来反推器件镜像新旧（第八章）
D-14b|V2|P-08 ro.vendor.build.fingerprint|ACME/PixelLike/PixelLike:16/HD1A.250101.001/1234:userdebug/release-keys|ACME/LockedDown/LockedDown:15/AQ1A.250201.001/5678:user/release-keys|android_side_drift|property.te:191-198|Android 侧口径变了。它只反映系统分区，禁止用来反推器件镜像新旧（第八章）
D-15|V3|P-11 supplemental_xml|absent cve_lines=0|present cve_lines=2 k1_numerator_plus|t3_appeared|file_contexts:427|T3 记账面出现：K1 分子可能 +1，但仍要求编号与公告主键逐字相等
D-16|V6|P-24 dmesg|denied_log|denied_log|stable|app.te:660-661|日志面一致；matches 计数含子串误命中，不作趋势用
D-17|DECL|media_codecs|present entries=2|present entries=2|stable|domain.te:176-178|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|vintf-manifest|present entries=1|present entries=1|stable|file_contexts:438|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|service-list|present matches=3|present matches=3|stable|shell.te:358|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|lshal|present matches=2|present matches=2|stable|shell.te:386|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|dumpsys gpu|present bytes=51|present bytes=51|stable|shell.te:79|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|dumpsys media.codec|vendor_dependent -|vendor_dependent -|stable|shell.te:29-219|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-17|DECL|dumpsys cameraserver|vendor_dependent -|vendor_dependent -|stable|shell.te:29-219|声明/服务面一致。声明变化不等于在场变化（第廿章之八第 4 条）
D-18|V5|P-23 /dev/ion|openable|openable|stable|shell.te:463|AOSP 明文允许的那条对照项一致
D-19|V4|companion driver_override|absent-in-both|absent-in-both|companion_missing|rpmsg_core.c:449|脚本 1 不采伴读（26 条姿势里没有它，见第廿七章），因此任何 rpmsg 名字读数都不得进 K8 分子（第廿六章之六 K8）；要补采就得先把它写进第廿章的表，不能偷偷加姿势
K6|rollup|crash-visibility|numerator_input=0|den=7|note=this row is the numerator INPUT, not the numerator: the ledger carries no chain column, so converting readable state fields into chain counts is done by hand against ch26's seven-chain table. drm_warn lines never enter the numerator and GPU reset_count is a trend term, not a numerator (ch26 K6)
K7|rollup|capture-windows|log_channel=0|den=4|note=two snapshots can only prove denominator item 4, the log channel. coldboot / decode-session / load-failure are window conditions, not ledger rows, so K7 stays hand-filled; a zero here is not a missing window, it is an unreadable channel (ch26 K7)
K8|rollup|value-with-source|rows_with_a_value=25|ledger_rows=120|value_source_tag=not-carried-by-script-1|note=K8 needs both a value and a legal value_source. Script 1's source column carries device_read/static_baseline, never fw_reported/driver_set/userspace_override, so this row proves condition one only; condition two and K3's field denominator are registered as ch30 gaps. rpmsg name readings are additionally excluded by D-19 (ch26 K3/K8)
L|rollup|level-action|level_downgrades=1|action=apply-ch26-rule-6|note=exactly one level per mode_tightened row, no more and no less. The word degraded_by_vendor also appears inside D-02's face tally, but a tally is not an event and must never be counted twice (ch26:2-6)
S|rollup|drift-summary|records=42|distinct_blocks=21|inputs=70+/70|note=comparison-only; this script issues no new device read
# 1. drift is not an attack signal and not a hardening signal. Every *_drift row above is a change of reading, not a change of security state.
# 2. a value gained on the after side is not proof the firmware changed; it may be a vendor grant, a build difference, or a window that happened to be open.
# 3. a value lost on the after side is not proof of a rollback; dontaudit silences refusals, so absence in the later snapshot needs P-05 re-run before any conclusion.
# 4. name-set and size drift on V1 never become version statements (ch20:8-2, ch23). Record them as file-name observations only.
# 5. the T3 row (D-15) is the only drift row that may move K1, and only when the bulletin key matches the XML key verbatim (ch23:9-1, ch24:15).
# 6. two snapshots are not a trend. Any "持续/恶化/好转" wording about the rows above is a report defect (ch26:2-6 registers only the one-level downgrade).
```

### 廿八之六 逐格读这 54 行：哪一格在说什么，哪一格禁止你读出什么

样例不是装饰，它是脚本 2 的契约证明。这一节把上面 54 行逐格过一遍，目的有三个：证明每行的 `before`/`after` 都能在第廿七章账本里找到出处，证明 12 条非 `stable` 记录里没有一条被写成"安全状态变了"，以及给读者一套可复用的定位方法——拿到一份真实的两快照输出时，按同样的顺序去查。

先给一个可以当场验的算术。54 行的构成是 1 表头 + 42 条 `D-` 记录 + 5 条 rollup + 6 条 `#` 注记；42 条记录里 30 条 `stable`、12 条非 `stable`。这 12 条是：`D-02`（1）、`D-09`（1）、`D-10`（2）、`D-12`（1）、`D-13`（1）、`D-13b`（1）、`D-14`（1）、`D-14b`（2）、`D-15`（1）、`D-19`（1）。

**第一步读 `D-01` 和 `D-02`，因为它们决定其余 40 条能不能读。**

`D-01|ALL|record-set|targets=60|targets=60|stable` 说的是"两次快照问的是同一组 60 个格子"。这里的 60 不是任意数字，它就是第廿七章账本的 `P-` 记录条数（脚本 1 输出 70 行 = 1 表头 + 60 条 `P-` + 3 条 rollup + 6 条注记）。这一格只要不是 `stable`，后面所有比对都是在比两套不同的问题，`S` rollup 的"仅比对"结论也要一起作废——所以 `record_set_drift` 在之七里被归到前提类，而不是漂移类。

`D-02` 的 `before` 是八个面的判定分布。把它按面拆开相加：V1 = 14+1+5+2 = 22，V2 = 4，V3 = 1+2 = 3，V4 = 3+3+10+1 = 17，V5 = 2+1+1+3+1 = 8，V6 = 1，V8 = 3+2 = 5，合计 **60**。`after` 侧同样合计 60。**分布直方图的总和必须等于 `D-01` 的 targets 数**，这是读者唯一能对 `D-02` 做的自检：如果哪次真实输出两边都是 60 但某一侧加起来不是 60，那不是漂移，是脚本 2 的字段清洗漏了一格。

`D-02` 两侧逐项相减，恰好得到五个变化，且每个变化都能在下面的具体 `D-` 记录里找到落点——这张对应表就是本章样例的内部一致性证明：

| `D-02` 里的分布变化 | 唯一来源记录 | 面合计是否变 |
| --- | --- | --- |
| V2：`present` 4 → 3，新增 `filtered_by_policy` 1 | `D-14` | 不变（4 = 3+1） |
| V3：`absent` 1 → 0，`present` 2 → 3 | `D-15` | 不变（3） |
| V4：`present` 10 → 9，新增 `degraded_by_vendor` 1 | `D-09` | — |
| V4：`absent` 3 → 2，新增 `bound` 1 | `D-12` | — |
| V4：`unlistable_value` 1 → 1 | `D-10` 两行互换（`atomisp` 进入、`ivpu` 退出） | 抵消 |

把 V4 单独算完：`absent` 3 − 1（`atomisp` 离开）+ 1（`ivpu` 进入）− 1（`D-12` 离开）= 2，`unlistable_value` 1 − 1 + 1 = 1，`present` 10 − 1 = 9，`degraded_by_vendor` 0 + 1 = 1，`bound` 0 + 1 = 1，`denied_value` 3 = 3，合计 2+1+9+1+1+3 = **17 = 前侧的 17**。`D-10` 的两行在这个合计里完全抵消，这是有意设计的：`param_face_drift` 只许说"模块目录的装载状态变了"，它不许在面分布上留下痕迹，否则读者会把它读成"参数面被打开了一格"。

**第二步读那一条会动级别的记录：`D-09`。**

`D-09|V4|P-15 writable-mode|present coredump_mode=644|degraded_by_vendor coredump_mode=444|mode_tightened`。三点必须同时成立才能落到这一格：前后两侧都读到了这个文件的模式位，且前侧是 0644、后侧是 0444，且本树取回的源码里 `rproc->sysfs_read_only` 的赋值点是 0（第十九章之五）。第三个条件是关键——如果上游已经把模式收紧了，0644→0444 就是采集窗口差异；正因为本树读到的赋值点给出的是 0644，两侧的差异只能由**没有取回的厂商补丁**解释，所以脚本 1 给的判定是 `degraded_by_vendor` 而不是 `present`。

这一格的处置在全部 34 个非 `stable` 值里是唯一的：它让所属链的 L 级降一级，第廿六章之二第 6 条把这句话写死了——**降一级，不多也不少**。`L|rollup|level-action|level_downgrades=1` 就是这条规则的机器体现：一份 42 条记录的输出里只有一个 `mode_tightened`，所以只降一级。反过来，如果有人对同一份输出算出 2 次降级，几乎一定是把 `D-02` 的 `degraded_by_vendor=1` 又数了一遍；之八第 `L` 行专门为此写了禁止语。

同一格还有一处禁止：0644→0444 只说明"写侧可读到的模式变严了"。它不说明固件被写过、不说明某个恢复开关被使用过，也不说明这台设备比上次更安全。第十七章之二把写侧能力登记为能力事实而不是操作步骤，这条纪律在比对阶段同样成立。

**第三步读 `D-10` 的两行，因为它们的 `after`/`before` 里有一个看着像数据其实不是数据的东西。**

`unlistable_value modes=644,644,` 和 `modes=644,644,644,` 结尾都带逗号。那是脚本 1 逐参数拼接模式位留下的分隔符，**不是第四个参数的空值**。把它按逗号切一下再数，`atomisp` 侧是 2 个参数、`ivpu` 侧是 3 个参数；把结尾逗号当成一格，就会多出一个"模式为空"的伪读数。之十一第 5 条把这类字段清洗列为脚本 2 的第一限制，就是这个原因。

这两行真正在说的是：`/sys/module/<mod>/parameters` 这个目录，一次快照里根本不存在（`absent`），另一次存在但里面的值读不到（`unlistable_value`）。这是模块**装载状态**的变化。禁止读成"某个模块参数被改过"——参数值从头到尾没读到过，一个都没读到，就一个都不能推断。

**第四步读两条构建类记录：`D-13` 与 `D-13b`。**

`D-13|V5|P-21 debugfs|debugfs_unreachable build=userdebug|muted_by_build build=user|build_switch` 记录的是同一件事的两种说法：`userdebug` 时 debugfs 挂载点存在但被 SELinux 挡住，所以判定是 `debugfs_unreachable`；换成 `user` 构建后，写侧那组 `debugfs` 规则整个不生成，拒绝随之消失，`dontaudit` 生效，判定变成 `muted_by_build`。这个值被明确排除在"厂商收紧"统计之外——构建类型变化不是厂商动作。之七把它归到登记类，且注记里写"不进厂商收紧统计"。

`D-13b|V5|P-22 fw_name|debugfs_unreachable -|debugfs_unreachable -|fwname_unavailable` 是这份样例里最容易被误读的一行，因为它看起来像漂移，其实不是。**它是一条常态记录。**脚本 2 在这一格写了四支分支（`stable` / `fwname_appeared` / `fwname_lost` / `fwname_unavailable`），`good → user` 落在第四支，意思是"两侧都拿不到运行期固件名"，而本域里 `P-22` 这个出口默认就不成立（它挂在 debugfs 下，而 debugfs 在两个画像里都不可达）。这一行的价值是它把"这个出口不存在"这件事写进了输出，让报表读者不会以为脚本漏采了固件版本。之九会用 `wide → good`、`good → wide`、`wide → wide` 三对把另外三支分支各跑一次。

顺带把 `fwname_appeared` 那一支会读到的东西说清楚：在 `wide` 画像里 `P-22` 读出的文本是 `intel/vpu/vpu_37xx_v0.0.bin`（桩值，见之四）。它尾巴上的 `v0.0` 长得像版本号，但按第廿三章之八第（6）条，运行期固件名是**文件名字段**，不是版本字段——厂商完全可以在不改固件的情况下换文件名，也可以不改文件名而换固件内容。所以 `fwname_appeared` 允许的结论只到"这个出口这一次读到了一个名字"，报表里如果把它写成"固件版本是 37xx v0.0"，那是报告缺陷。

**第五步读两条属性面记录：`D-14` 与 `D-14b`。**

`D-14|V2|P-07 getprop-filter|present matches=1|filtered_by_policy -|filter_drift` 说的是：`userdebug` 时 `getprop` 能列出那把键，`user` 时被策略过滤掉。这一格最容易出事的地方是 `matches=1` 这个计数。它是子串匹配的行数，不是"合法可读键的数量"。第廿三章之八记过一类具体的子串陷阱：从公开公告页面取回的正文里出现过一段长得像凭据的字符串，任何按关键字子串计数的做法都会把它误算进命中数。所以 `matches=` 只许用来判断"有 / 没有 / 数量级变了"，不许用来做趋势，更不许在报表里写成"可读固件键有 N 个"。之七给 `filter_drift` 的禁止语就是这个。

`D-14b` 三行是同一格里三种不同处置：`ro.bootloader` 两侧都是 `unknown` → `stable`；`ro.build.version.security_patch` 从 2026-06-05 变到 2026-09-01 → `android_side_drift`；`ro.vendor.build.fingerprint` 整串变了（包括 `userdebug` → `user`）→ `android_side_drift`。第八章给过口径：这三件属性只反映系统分区，禁止用来反推器件镜像新旧。把它们放进脚本 2 的唯一理由是**防误用**——如果比对脚本不管这三格，报表里就会出现"安全补丁变新了所以固件也变新了"这类句子；现在它们有专门的 `did`，读者能看到"Android 侧口径变了"与"器件侧什么都没变"是同时成立的两件事。

**第六步读唯一一条可能动指标的记录：`D-15`。**

`D-15|V3|P-11 supplemental_xml|absent cve_lines=0|present cve_lines=2 k1_numerator_plus|t3_appeared`。三个数不能混：`cve_lines=2` 是那次快照里 XML 里出现的 CVE 行数，`k1_numerator_plus` 是一个方向标记，K1 分子**是否**真的 +1 由人工核对决定，条件在第廿三章之九第（1）条和第廿四章第 15 条里：编号必须与公告主键逐字相等。行数为 2 而分子只 +1 是完全可能的（同一编号重复出现、或者第二行属于别的模块）。所以这一行给 K1 的是"候选"，不是"输入"；34 个非 `stable` 值里只有 `t3_appeared` 和 `t3_lost` 允许碰 K1，其余一个都不许。

**第七步读 `D-19`，因为它同时是断言和缺口。**

`D-19|V4|companion driver_override|absent-in-both|absent-in-both|companion_missing`。这一行两侧的值一模一样，却故意不给 `stable`——之一定义过它：它是"这条链没有伴读格子"的常态声明。键里那个空格也是刻意的：`key` 列不允许出现 `|`，所以脚本 2 在写出前把 `companion|driver_override` 的分隔符换成了空格；这就是之十一第 6 条要求按 `did` 而不是按 `did+key` 做后续 join 的原因。

`absent-in-both` 是这一行唯一的读数来源，它说的是"第廿七章账本里根本没有 `driver_override` 这一格"，不是"设备上不存在这个文件"。因此它的下游禁令有两条：任何 rpmsg 名字读数不得进 K8 分子；要补采，必须先把这个姿势写进第廿章的表，把 K2 的分母从 26 改掉，再让脚本 1 采它。第十八章之四许诺过"脚本 2 会检查 `driver_override` 伴读"，兑现方式就是这一行加一条硬断言：如果在真实账本里发现了 `driver_override` 而没有对应姿势，脚本 2 直接 `exit 9`（之九给实测输出）。

**第八步，也是最长的一步：读 `stable`，因为 30 条 `stable` 里有一半在说"这一格没变，但别把它读成没风险"。**

`D-03 present=2|present=2` 与 `D-04 all-equal|all-equal` 要放在一起读：命中文件名集合相同、字节数也相同。它们合起来只排除一种解释（同一批文件被替换成了不同长度的版本），不排除任何版本结论——第廿十章之八第 2 条禁止把 V1 的名字与长度变化读成版本变化，反向同样成立：不变也不是证据。而"名字集合相同、字节数不同"这一格在 `good → user` 里永远不会触发，之四解释过这就是 `alt` 画像存在的理由。

`D-05 present lines=6|present lines=6|stable`，注记写"地基仍在"。这一格是第廿六章所有成因推断的前提：只有 `ls -Zd` 还能列出 6 行标签，`eacces_typed` 与 `eacces_untagged` 的区分才有意义。它一旦落到 `label_base_lost` 或 `label_unreadable_both`，V5 面上的 `stable` 记录全部要降格重读——之七把它列为前提类，正是因为它的影响是横向的，不是纵向的。

`D-06` 五行、`D-07` 四行是拓扑与父链。`/sys/class/rpmsg` 两侧都 `absent` 也算 `stable`，而且这是有意义的 `stable`：它说明这一格没被采集窗口影响。`D-07` 四行 `same-path` 说明符号链接落点的平台设备名没换；第十八章之六许诺的 glink 边数不在这里，之十把它登记成缺口。

`D-08` 两行 `denied_value -|denied_value -` 是数值面的两条常态拒绝。`stable` 在这里只表示"两次都被拒"，不表示"值相同"——值从来没读到过。之十一第 1 条的"值文本不参与判定"在这一格最要命：如果有人以后把 `denied_value` 换成读到的内容，脚本 2 也不会去比较文本，只有判定档位变化才会出 `value_gained` / `value_lost` / `verdict_drift`。

`D-11` 五行是成因档。三行 `label_unreadable -`、一行 `eacces_typed video_device`、一行 `eacces_untagged device`。`stable` 的条件比"档位相同"更严：档位相同**且**支撑档位的标签也一致。之七给 `cause_migration` 与 `cause_indeterminate` 留的两档，就是这一格在别的画像对上会走的路。

`D-16 denied_log|denied_log|stable`，注记把 `matches` 计数的子串陷阱再说了一遍——这一格是 `stable`，所以计数根本没进输出。`D-17` 七行是声明面，`dumpsys gpu` 那行是 `present bytes=51`，字节数相同即 `stable`；`dumpsys media.codec` 与 `dumpsys cameraserver` 两侧都是 `vendor_dependent`，这是脚本 1 对"能不能列出取决于厂商实现"的固定判定。七行共同的注记是本章最重要的一句重复：**声明变化不等于在场变化**。`D-18 /dev/ion openable|openable` 是 AOSP 明文允许的那条对照项，它的 `stable` 用来证明"脚本 2 在能读的地方照样读得到"，排除整份比对是空转的可能。

**最后读五条 rollup 与六条注记。** 五条 rollup 的口径在之八单独展开，这里只标一件事：`K6` 的 `numerator_input=0`、`K7` 的 `log_channel=0`、`K8` 的 `rows_with_a_value=25` 三个数里，前两个是从第廿七章账本里数出来的读数，第三个是脚本 2 自己算的——25 是这份样例里 `before` 或 `after` 至少有一侧带非 `-` 值的记录数，它不是可读字段数。六个 `#` 注记是输出的一部分，不是脚本注释：它们每次运行都写在末尾，目的是让被人复制进表格、截图或转发的样例仍然带着自己的禁推边界。之七把每条禁推挂到具体值上，注记是它们的最小复述。

一句话总结这一对：12 条非 `stable` 记录按之七的四档分，是一条动级别（`D-09`，唯一的一级），一条可能动 K1（`D-15`，且只是候选），其余十条落在登记类。这十条里又有两行（`D-13b`、`D-19`）记录的是状态不是变化，所以真正的前后变化是 10 行——这正是之七末尾那条读表纪律在这份样例上的具体数字。前提类一个都没触发：`D-01` 是 `stable`，`D-05` 也是 `stable`。没有任何一行在说固件版本，也没有任何一行在说攻击。

### 廿八之七 漂移词汇表：35 个值、四种处置、以及每个值禁止推出的东西

脚本 2 的 `verdict` 列用的是**第二套词表**，与脚本 1 的 17 个读数判定值正交。这件事必须先讲清楚，否则读者会拿两套词去对：脚本 1 的 17 个值回答"这一格读到了什么"，脚本 2 的 35 个值回答"这一格在两次采集之间发生了什么、所以我下一步该做什么"。第廿六章把这两套分开，是因为把"读不到"和"上次能读这次不能"混成一个词，运营方就没法区分"补授权"和"查设备"这两种完全不同的工单。

**按处置动作分四档。** 这是这张表真正的用法——不是查词义，是查"看到这个词以后做什么"。

**第一档：前提类（4 个值）。出现即停止后续归因。**

| 值 | 块 | 含义 | 处置 | 禁止的推论 |
| --- | --- | --- | --- | --- |
| `record_set_drift` | D-01 | 两份快照的 target 全集不同 | 先问采集姿势有没有改，再问设备有没有变 | 不得进入 D-03 及以后任何块的成因归因 |
| `label_base_lost` | D-05 | 标签面（`P-05`）只在单侧可用 | 本批漂移里所有成因分类降级为"不可比" | 不得用"一侧能分因一侧不能"推出厂商改动 |
| `label_unreadable_both` | D-05 | 两侧都取不到标签 | 只登记，本域成因分解不可执行 | 不得写"两种 eacces 无法区分"之外的任何结论 |
| `label_coverage_drift` | D-05 | 判定都成功但标签行数变了 | 先重验 `P-05` 再归因；`D-11` 的本批结论挂起 | 不得把行数减少读成"节点被删" |

**第二档：级别类（1 个值）。唯一会改变 L 级的词。**

| 值 | 块 | 含义 | 处置 | 禁止的推论 |
| --- | --- | --- | --- | --- |
| `mode_tightened` | D-09 | 写侧属性模式收紧（本域实例：`coredump 0644 → 0444`） | 该链 L 级降一级并记 `degraded_by_vendor`，正好一级（第廿六章之二第 6 条） | 不得写成攻击信号，也不得写成加固信号；不得因为"四个属性都变了"就降四级——一次模式事件对应一级 |

**第三档：指标类（2 个值）。唯一允许触碰 K 数字的词，且带一个硬前置。**

| 值 | 块 | 含义 | 处置 | 禁止的推论 |
| --- | --- | --- | --- | --- |
| `t3_appeared` | D-15 | T3 记账面从不可得变可得 | K1 分子**可能** +1 | 前置条件：该 CVE 编号与公告主键逐字相等，且编号不做补零、截断或任何变换（第廿三章之九第 1 条 + 第廿四章之十五）；不满足就只登记存在性 |
| `t3_lost` | D-15 | T3 面消失或被挡 | K1 分子重置，历史值保留 | 不得据此写"未修复"（第廿四章之十五）；不得推断厂商删除了文件 |

**第四档：登记类（27 个值）。只登记，不推。** 这一档最长，也正最容易出事——它的每一条都对应一个"看起来能推但实际不能推"的句式。

| 值 | 块 | 含义 | 禁止的推论 |
| --- | --- | --- | --- |
| `face_drift` | D-02 | 某个面的判定分布变了 | 分布变了不等于某条链变好或变坏；按面定位到具体 `P-id` 再看 |
| `name_set_drift` | D-03 | 静态候选名的命中集合变了 | 命中与否只是文件名字符，不构成装载或版本证据（第廿章之八第 2 条） |
| `size_drift` | D-04 | 同名文件的字节数变了 | 同版本可重打包、不同版本可同长度；禁止推版本 |
| `topology_drift` | D-06 | 拓扑节点数或可达性变了 | 这是七条链里最强的一档读数，但变化仍要先归因（重启/换后端/授权） |
| `symlink_target_drift` | D-07 | 符号链接落点变了 | 落点变化 = 平台节点改名或换控制器，不是固件变化（十七之六） |
| `value_gained` | D-08 | 后一次读到了值 | 可能是厂商补授权，不得直接记成器件变化 |
| `value_lost` | D-08 | 前一次读到、这次读不到 | `dontaudit` 会静音拒绝，先重跑 `P-05`（第廿章之七） |
| `verdict_drift` | D-08 | 两种失败态互换 | 成因分类需要重跑 `P-05`，不得写"权限被收回" |
| `mode_relaxed` | D-09 | 模式从 `0444` 回 `0644` | 恢复到的是 AOSP 默认，不推"厂商放松限制"，不动级别 |
| `mode_unclassified` | D-09 | 模式变了但不落进已知迁移对（本域实例 `0644 → 0600`） | 记边界不记降级；"没见过"不等于"知道它安全" |
| `mode_unavailable` | D-09 | 至少一侧没取到模式 | 本条不参与模式漂移统计，不得补 0 |
| `param_face_drift` | D-10 | 模块参数三态（目录/模式/值）组合变了 | 未装载与"已装载但值被拒"是两件事，禁止读成"开关被用过" |
| `cause_migration` | D-11 | 同一节点的拒绝态在两档之间迁移 | 处置成本可能从一步变两步，但不得写成策略变更 |
| `cause_indeterminate` | D-11 | 一侧无标签，成因分解在该节点不成立 | 归入 `label_unreadable` 族，不得就地分类 |
| `bind_drift` | D-12 | 驱动绑定见证变化（`dbg` 文件存在性） | 是在场性证据，不是固件证据（第廿五章之六） |
| `build_switch` | D-13 | 构建类型变了（`user` ↔ `userdebug`） | 不进"厂商收紧"统计——这是构建因 |
| `debugface_drift` | D-13 | 同为非 user 构建而 debugfs 可达性变了 | 唯一可以拿去问厂商的那一档，但仍不是结论 |
| `fwname_appeared` | D-13b | 运行期固件名从不可得变可得 | 它是文件名字段不是版本字段（第廿三章之八第（6）条） |
| `fwname_lost` | D-13b | 运行期固件名从可得变不可得 | 先排除采集窗口与授权收回，不得写"固件名被抹掉" |
| `fwname_unavailable` | D-13b | 两侧都拿不到运行期固件名 | 这是状态不是变化；不得计入"本次变差" |
| `filter_drift` | D-14 | 属性过滤视图的可读键数变了 | 命中数里含子串误命中（第廿二章之三）；计数变化不等于新增上报 |
| `android_side_drift` | D-14b | 三件 Android 侧口径变了 | 只反映系统分区，禁止反推器件镜像新旧（第八章） |
| `t3_drift` | D-15 | T3 面换类但不落入已知迁移对 | 登记为边界，不动 K1 |
| `log_channel_drift` | D-16 | 日志通道可达性变了 | 等于采集窗口条件第 4 项变了，K7 分子要重算，不是设备变差 |
| `declaration_drift` | D-17 | 声明/服务面读数变了 | HAL 接口版本不是固件版本；走 O5 资产口径，不开 O1 工单（第廿章之八第 4 条） |
| `named_capability_drift` | D-18 | 被点名节点（`/dev/ion`）的授权态变了 | 与厂商 shell 授权面改动相关（第十八章之五），与器件无关 |
| `companion_missing` | D-19 | 台账里没有 `driver_override` 伴读格 | 脚本 1 本来就不采它；任何 rpmsg 名字读数不得进 K8 分子 |

外加 `stable` 一个值，共 35 个。**这张表与第廿章之九的 17 值表有一个结构性差别**：17 值表是"每个格子只能取一个值"的枚举，35 值表是"每个格子按它所属的块取子集"的枚举——`stable` 在 21 个块里都合法，`mode_tightened` 只在 D-09 合法，`t3_appeared` 只在 D-15 合法。脚本 2 里没有跨块的值复用，因此看到词就能定位块，不需要看 `did` 列。

最后一条读表纪律，也是第廿九章会把它列成误区的那一条：**第四档里有两个值（`fwname_unavailable`、`companion_missing`）记录的是状态而不是变化。** 它们在两侧完全一致的采集里也会输出，因为这两个出口在本域默认不成立。所以"非 `stable` 的行数"不能直接当作"变化数"读：样例的 `good → user` 有 12 行非 `stable`，其中真正的前后变化是 10 行，另外 2 行是上面这两格。第之六节会把这 12 行逐行列出。


### 廿八之八 四行 rollup 的确切口径：它们是什么，以及它们明确不是什么

脚本 2 在 42 条 `D-` 记录之后固定输出四行 `rollup`，再加一行 `S|rollup` 汇总。这五行是全篇唯一涉及 K 类指标的地方，也是最容易被读者抄成"指标已算出"的地方，所以每一行都要把"分子"与"分子的输入"分开说。下面用样例（`good → user`）的实际字段值解释，再给出对照组。

**`K6|rollup|crash-visibility`——崩溃可见性，第廿六章之六 K6 的分母是七条链。**

样例输出是 `numerator_input=0|den=7`。这一格数的是"两份快照里，`P-14` 的 `state` / `firmware` 等属性值读数至少有一次 `present` 的 **(姿势, 目标) 组合数**"，用 `sort -u` 去重，所以同一个属性在前后两次都成功只算一次。它 **不是** K6 分子，理由写在同一行的 `note` 里，有两条：

1. **台账没有链列。** 脚本 1 的八列里没有"这条读数属于哪条链"，而 K6 的分母按第廿六章的表是七条链（Mali CSF、Adreno GMU-KGSL、Exynos MFC、QCOM Venus、MTK VPU+SCP、相机 ISP、NPU-APU）。把 `rproc.state` 这一格折算成"QCOM Venus 链已覆盖"，需要人对照那张表做一次映射，脚本 2 不做这个映射——因为一旦做了，链的定义就藏进了 awk，第廿六章那张表就不再是唯一的口径来源。
2. **两条禁令。** 第十二章 `:1050` 那条 `drm_warn`（软挂起未致命）不进任何分子，也不算分母的"已覆盖"证据；GPU 侧 `reset_count` 是趋势项，做辅助观测不做分子（第十三章之五：固件重启不是固件更换）。这两条在 `note` 里逐字出现，是为了让抄这一行的人无法把它们当背景噪音。

对照组：只要 `wide` 画像出现在任一侧，`numerator_input` 就是 `2`（`wide` 的 `P-14` 有两条属性读到值）。这正是"两个数都合法、但都不构成链覆盖"的实例——`0` 与 `2` 的差是**授权面之差**，不是器件之差。

**`K7|rollup|capture-windows`——采集窗口，分母固定 4。**

样例输出 `log_channel=0|den=4`。脚本 2 只能证明四个分母项里的第四项（日志通道），因为它是两份快照里唯一以"台账行"形式存在的窗口条件；冷启动、解码会话、加载失败这三项是**窗口**而不是**格子**——它们要求"在某段时间内守着一台设备"，不是"在某台设备上按一次姿势"。所以这一行永远是手工填的，脚本 2 给的是一个布尔位：任一侧 `P-24`（`dmesg`）为 `present` 则 1，否则 0。

这里有一个必须点破的读法陷阱：**`log_channel=0` 不等于"这个窗口没覆盖"，它等于"这个通道读不到"。** 第廿六章之六 K7 的原话是禁止用"跑了 30 天趋势"代替窗口覆盖；反过来也一样，不能拿一个读不到的通道去抵一个没跑的窗口。样例里 `good` 与 `user` 两侧 `dmesg` 都被拒（`app.te:660-661`），所以是 0；`wide` 参与的任何一对都是 1。

**`K8|rollup|value-with-source`——带合法来源标签的值读数。**

样例输出 `rows_with_a_value=25|ledger_rows=120|value_source_tag=not-carried-by-script-1`。`ledger_rows` 是两份快照里所有 `P-` 行的总数（60 + 60 = 120），`rows_with_a_value` 是其中"判定为 `present` 且 `label` 列非空非 `-`"的 **(姿势, 目标) 去重数**。这两个数一起说的是 K8 的**第一个条件**：这台设备的这套姿势，实际读到了 25 格有值。

K8 的定义有两个条件（第廿六章之六 K8）：**既有值，又要有一个合法的 `value_source` 标签**。第二个条件脚本 2 无法满足，而且原因很具体：脚本 1 的 `source` 列只携带 `device_read` 与 `static_baseline` 两种值，从来没有 `fw_reported` / `driver_set` / `userspace_override` 这三种——而第十八章之四给出的那个不对称（一个 rpmsg 名字既可以来自固件宣布，也可以来自内核驱动填写，还可以由带权限的用户空间写入）恰恰要靠后三种来区分。所以 `value_source_tag=not-carried-by-script-1` 是这一行的第三个字段，它是**自证缺陷**，不是省略。

顺带一条排除规则：`D-19` 已经把"任何 rpmsg 名字读数不得进 K8 分子"写成硬约束（`note` 里同样逐字重复）。这三件事合起来构成第卅章的两个具名缺口——**台账缺链列（K6 无法自动折算）**与**台账缺 `value_source` 列（K8 第二条件无法自动判定）**。

**`L|rollup|level-action`——降级动作计数。**

样例输出 `level_downgrades=1|action=apply-ch26-rule-6`。这一行只数 `D-` 记录里 `verdict == mode_tightened` 的条数，不数别的。这个"只数一种"是修出来的，不是设计出来的：早期版本的这一格用 `grep -c degraded_by_vendor` 统计整份台账，结果样例里数出了 2，而真实降级只有 1 次——因为 `D-02` 的判定分布串（`V4:degraded_by_vendor=1`）里也含有这个词，被同一个 `grep` 命中两次。**同一件事被计数两次，正是第廿六章第 6 条禁止的"不升不降都是错"的另一种形式。** 现在 `DEG` 用 awk 限定 `$1 ~ /^D-/ && $6=="mode_tightened"`，只承认事件、不承认统计串。

三个词要说清楚各自的分量：`mode_tightened`（`0644 → 0444`）是唯一触发降一级的；`mode_relaxed`（`0444 → 0644`）只登记不动级别，因为它恢复到的是 AOSP 默认，把它读成"厂商放松了限制"没有依据；`mode_unclassified`（例如 `0644 → 0600`）既不降级也不解释，只记边界——第廿六章第 6 条给的迁移对只有 `0644 ↔ 0444`，别的组合是"没见过"，不是"知道它安全"。

**`S|rollup|drift-summary`——本脚本的自我限制声明。**

固定输出 `records=42|distinct_blocks=21|inputs=70+/70|note=comparison-only; this script issues no new device read`。最后一句是这一章的定性：**脚本 2 不产生任何新的设备读数**，因此它的任何结论都不可能比脚本 1 的两次读数更有据；`inputs=70+/70` 是两份快照各自行数（含表头），用来提醒读者"分母来自脚本 1 的姿势集合，改脚本 1 就要重跑脚本 2"。


### 廿八之九 全量分支覆盖实测：55 个发射点、35 个词汇值、十对快照，以及三条不能断言的东西

第廿七章为脚本 1 立过一条验收纪律，本章原样沿用：**从不触发的分支算报告的缺陷，不算样例的遗漏。**脚本 2 比脚本 1 更需要这条纪律，因为它有 21 个块、55 个记录发射点，而它的正确输出是一份 54 行的固定形态文件——一个从未走到的分支在正常样例里不会留下任何痕迹，读者只看到"这一格是 `stable`"，不会知道另外三支分支根本没被测过。

#### 十对快照：一次跑完的全部实测

四个画像两两配对（含自比对）共 10 对。每对跑一次，记录退出码、行数、`stable` 与非 `stable` 的分配、墙上时间。下表是完整实测，非 `stable` 那一列只数 `D-` 记录，不数 rollup：

| 配对 | `rc` | 行数 | `stable` | 非 `stable` | 耗时（秒） | `stderr` |
| --- | --- | --- | --- | --- | --- | --- |
| `good → user` | 0 | 54 | 30 | 12 | 44 | 0 字节 |
| `user → good` | 0 | 54 | 30 | 12 | 52 | 0 字节 |
| `good → wide` | 0 | 54 | 19 | 23 | 56 | 0 字节 |
| `wide → good` | 0 | 54 | 19 | 23 | 66 | 0 字节 |
| `user → wide` | 0 | 54 | 15 | 27 | 66 | 0 字节 |
| `wide → user` | 0 | 54 | 15 | 27 | 58 | 0 字节 |
| `good → alt` | 0 | 54 | 34 | 8 | 50 | 0 字节 |
| `alt → good` | 0 | 54 | 34 | 8 | 48 | 0 字节 |
| `good → good`（自比对） | 0 | 54 | 40 | 2 | 49 | 0 字节 |
| `wide → wide`（自比对） | 0 | 54 | 40 | 2 | 47 | 0 字节 |

三点观察，都能从表里直接读出。**第一，行数恒为 54**，包括漂移最多的 `user ↔ wide`（27 条非 `stable`）和零漂移的自比对——这正是之二那条"输出不会因发现新东西而变长"的性质。**第二，反向对的分配完全对称**：`good → user` 与 `user → good` 都是 30/12，`good → wide` 与 `wide → good` 都是 19/23。分配对称但内容不对称——之一定义过 `value_gained` / `value_lost` 是两条不同记录，所以这个 30/12 相等只是巧合级的验证，不能当断言用。**第三，`good ↔ alt` 只有 8 条非 `stable`**，比 `good ↔ user` 少，这是设计目标：`alt` 只改四处，每处对应一条其它对无法到达的分支，改动越少，漂移越可归因。

整个循环的墙上时间合计 **536 秒**（十对，单核 Windows Git Bash）。同一组配对先前还完整跑过一轮，那一轮耗时 471 秒，两次每一对都是 `rc=0 lines=54`；本轮（第卅章定稿后从正文围栏重新抽出脚本再跑）合计 488 秒，十对的 `stable` / 非 `stable` 分配与上表逐对相同，且十份输出与上一轮那十份逐字节 `cmp` 通过——墙上时间是这一节里唯一会动的量。复现命令（输入是脚本 1 在四个画像下产出的四份账本）：

```bash
for pair in good-user user-good good-wide wide-good user-wide wide-user good-alt alt-good good-good wide-wide; do
  a=${pair%-*}; b=${pair#*-}
  bash w17-drift.sh "snap-$a.txt" "snap-$b.txt" > "dr-$pair.txt" 2> "dr-$pair.err"
  rc=$?
  printf '%s rc=%s lines=%s err=%s\n' "$pair" "$rc" "$(wc -l < "dr-$pair.txt")" "$(wc -c < "dr-$pair.err")"
done
```

手里没有桩画像的读者可以用任意两份历史账本跑同一个循环，但要看清：那时覆盖表不成立，成立的就只剩"行数恒定"和退出码这两条。

#### 词汇级覆盖：35 个值全部被触发过

把 10 份输出里所有 `D-` 记录的 `verdict` 列去重，得到的正是之七那张表的 35 个值——不多不少。下表给出每个值**首次**触发的配对（按上表顺序）与所属块。`stable` 排在第一位是必然的：它不是漂移值，而是第 35 个词汇成员，脚本 2 每份输出里至少有 15 条它是 `stable`。

| 词汇值 | 块 | 首次触发的配对 |
| --- | --- | --- |
| `stable` | D-01 | `good-user` |
| `face_drift` | D-02 | `good-user` |
| `mode_tightened` | D-09 | `good-user` |
| `param_face_drift` | D-10 | `good-user` |
| `bind_drift` | D-12 | `good-user` |
| `build_switch` | D-13 | `good-user` |
| `fwname_unavailable` | D-13b | `good-user` |
| `filter_drift` | D-14 | `good-user` |
| `android_side_drift` | D-14b | `good-user` |
| `t3_appeared` | D-15 | `good-user` |
| `companion_missing` | D-19 | `good-user` |
| `mode_relaxed` | D-09 | `user-good` |
| `t3_lost` | D-15 | `user-good` |
| `record_set_drift` | D-01 | `good-wide` |
| `name_set_drift` | D-03 | `good-wide` |
| `label_base_lost` | D-05 | `good-wide` |
| `topology_drift` | D-06 | `good-wide` |
| `symlink_target_drift` | D-07 | `good-wide` |
| `value_gained` | D-08 | `good-wide` |
| `mode_unavailable` | D-09 | `good-wide` |
| `cause_migration` | D-11 | `good-wide` |
| `debugface_drift` | D-13 | `good-wide` |
| `fwname_appeared` | D-13b | `good-wide` |
| `t3_drift` | D-15 | `good-wide` |
| `log_channel_drift` | D-16 | `good-wide` |
| `declaration_drift` | D-17 | `good-wide` |
| `named_capability_drift` | D-18 | `good-wide` |
| `value_lost` | D-08 | `wide-good` |
| `fwname_lost` | D-13b | `wide-good` |
| `size_drift` | D-04 | `good-alt` |
| `label_coverage_drift` | D-05 | `good-alt` |
| `verdict_drift` | D-08 | `good-alt` |
| `mode_unclassified` | D-09 | `good-alt` |
| `cause_indeterminate` | D-11 | `good-alt` |
| `label_unreadable_both` | D-05 | `wide-wide` |

**最小覆盖集是 6 对，可以从上表直接推出来。**四个"单例"约束先钉死三对：`mode_tightened` 只有 `good-user` 能出，`mode_relaxed` 只有 `user-good` 能出，`label_unreadable_both` 只有 `wide-wide` 能出（它要求两侧都读不到标签，四个画像里只有 `wide` 满足）。剩下两格必须成对补：`fwname_appeared` 与 `value_gained` 只出现在 `good-wide`/`user-wide`，`fwname_lost` 与 `value_lost` 只出现在 `wide-good`/`wide-user`，两组的配对集合不相交，所以至少各取一对。最后，五个只在 `alt` 一侧变化的值（`size_drift`、`label_coverage_drift`、`verdict_drift`、`mode_unclassified`、`cause_indeterminate`）只能由 `good-alt` 或 `alt-good` 提供，取一对。合计下界 3 + 2 + 1 = **6 对**，而 `{good-user, user-good, good-wide, wide-good, good-alt, wide-wide}` 恰好覆盖 35 个值，所以它就是最小集。

另外四对不是浪费，各有各的用途：`user-wide` 与 `wide-user` 用来确认构建维度（`muted_by_build`）与授权维度（`wide`）的交叉不产生新值，`alt-good` 用来确认方向反转只换前后列不改词汇，`good-good` 与 `wide-wide` 用来做下面那条假漂移检查。

#### 分支级覆盖：55 个发射点全部走到

词汇级覆盖还不够，因为**两个不同分支可以写同一个词**。实测就出现了这一例：D-13 有 4 个记录发射点，但只产出 3 个词汇值——`debugface_drift` 被两支分支共用，一支是"同为非 `user` 构建而 debugfs 可达性变了"，另一支是兜底分支"判定换了类别"。如果只查词汇，兜底分支看起来永远没被走到。

所以分支级验收换个指纹：`(did, note 前 20 字)`。同一块的不同分支在本脚本里注记文本必然不同，这是写脚本时的有意约束，也是这次能这样做的原因。按这个指纹去重 10 份输出，每个块得到的数与脚本正文里 `rec D-` 的发射点数逐一相等：

| 块 | 发射点 | 实测指纹 | 块 | 发射点 | 实测指纹 |
| --- | --- | --- | --- | --- | --- |
| D-01 | 2 | 2 | D-12 | 2 | 2 |
| D-02 | 2 | 2 | D-13 | 4 | 4 |
| D-03 | 2 | 2 | D-13b | 4 | 4 |
| D-04 | 2 | 2 | D-14 | 2 | 2 |
| D-05 | 4 | 4 | D-14b | 2 | 2 |
| D-06 | 2 | 2 | D-15 | 4 | 4 |
| D-07 | 2 | 2 | D-16 | 2 | 2 |
| D-08 | 4 | 4 | D-17 | 2 | 2 |
| D-09 | 5 | 5 | D-18 | 2 | 2 |
| D-10 | 2 | 2 | D-19 | 1 | 1 |
| D-11 | 3 | 3 | **合计** | **55** | **55** |

D-13 那一行是这次验收真正抓到的东西：`debugface_drift` 的两个分支分别由 `good-wide`/`wide-good`（`debugfs_unreachable ↔ vendor_dependent`，命中第二支）与 `user-wide`/`wide-user`（`muted_by_build ↔ vendor_dependent`，落到兜底支）触发，四条输出、两支分支，两支都活着。

#### 自比对：假漂移检查，以及为什么它的通过条件不是"零漂移"

把同一份快照同时作为 `before` 和 `after` 喂给脚本 2，理想结果是全 42 条 `stable`。实测是 **40 条 `stable` + 2 条例外**，两对自比对的例外分别是：

```
good → good : D-13b fwname_unavailable , D-19 companion_missing
wide → wide : D-05 label_unreadable_both , D-19 companion_missing
```

这三个值允许出现在自比对里，因为**它们记录的是两侧共同的状态，不是变化**：`companion_missing` 是"台账里没有伴读格子"的常态声明（之六第七步），`fwname_unavailable` 是"这一支四臂分支的第四臂"（之六第四步），`label_unreadable_both` 是 D-05 四臂中唯一一支两侧同侧的臂（`present/*|*/present` 要求恰好一侧可读，所以自比对不可能进 `label_base_lost`）。

因此这条检查的正确断言形态是：**自比对里，任何携带变化语义的值都必须为 0 条**，而不是"非 `stable` 必须为 0 条"。可复用的判定式：

```bash
awk -F'|' '/^D-/ && $6 != "stable" &&
     $6 ~ /drift|gained|lost|appeared|switch|tightened|relaxed|migration|indeterminate/ { n++ }
     END { print n + 0 }' dr-self.txt
```

两对自比对该式的结果都是 `0`。这个区别不是吹毛求疵：之七最后一条读表纪律说的就是"非 `stable` 的行数不等于变化数"，自比对把它变成了可执行的一行。反过来说，如果未来某次改动让自比对里出现一条 `*_drift`，那不用看内容就已经是缺陷——同一份文件和自己比不产生任何信息。

#### 契约断言：`exit 9` 的唯一触发路径

之一写过 `9` 是契约违规，而当前只有一条断言会触发它，就是 D-19。测试方法不是构造画像，是**改一份账本**：把 `snap-good.txt` 第 49 行（`P-20|/dev/rpmsg0`）的 `label` 列从 `device` 改成 `driver_override=cdsp`，其余 69 行一字不动，然后与未改动的 `snap-user.txt` 比对。这一步模拟的是将来最可能发生的误改：有人"顺手"把伴读名字塞进标签列，好让 rpmsg 那条链看起来有名字读数。

实测：

```
rc=9
stdout: 1 行（只有表头 did|face|key|before|after|verdict|rule|note）
stderr: BUG: driver_override found in 1 row(s) but it is not a posture registered in ch20;
        add the posture first, do not compare it
```

（`stderr` 实际是**一行**，上面为了排版在分号后折行；这一点重要，因为有人会把这条消息当成 grep 目标。）

两个细节正是要的效果。**stdout 只剩表头**：断言在任何一条 `D-` 记录写出之前退出，所以不会出现"比对了 41 条、第 42 条拒答"的半份输出——半份比对结果比没有结果更危险，因为它看起来完整。**消息里给的是下一步动作而不是错误类别**：它要求先把姿势写进第廿章的表，而不是教人怎么绕过这条检查。这条断言存在的全部理由在第廿六章之六 K8 那句里：任何 rpmsg 名字读数不得进 K8 分子；脚本 2 不能替读者守着这条线，除非它拒绝处理带这种读数的账本。

#### 确定性

同一对输入跑两次，输出必须逐字节相同。三组证据：其一，上文两轮十对矩阵，每一对都 `rc=0 lines=54`；其二，为了数进程创建次数（之十一那张 970 次 fork 的表），曾把脚本会用到的九个外部命令（`awk`、`cat`、`comm`、`grep`、`head`、`sed`、`sort`、`tr`、`wc`）全部换成计数桩再跑 `good → user`，桩只计数并原样转发、不改内容，产出的 54 行与干净运行 `cmp` 通过，逐字节一致；其三，之十一第 4 条写明 awk 侧只用 POSIX 特性，所以这份输出不依赖具体是 gawk、mawk 还是 toybox awk。

需要说明的是**这里没有依赖时钟**：脚本 2 的输出里没有任何时间戳，因此不存在"同一份输入在不同日子跑出不同结果"的通道。反过来，排序口径是硬依赖——脚本第 7 行自己设定并导出 `LC_ALL=C`（之十一第 4 条），调用方不需要另外设置；`D-02` 的面分布与四处 `comm` 差集都按字节序比较，换成 locale 排序就会换结果，所以这一行不是风格问题。

#### 三条不能断言的（第廿九章会把它们列进误区）

1. **覆盖是对桩画像的覆盖，不是对真实设备的覆盖。**四个画像能凑出 35 个值，靠的是把"授权面"和"构建面"分别推到两端；真机上同一时刻既 `user` 又 debugfs 可达的组合（策略补丁、Magisk 类改动）没有对应画像，脚本 2 在这种输入下的行为是"按分支表落到最近的一支"，不是"识别出新组合"。特别是 `D-13` 的兜底分支，它在真机上更容易被走到，而它给的词与第二支同为 `debugface_drift`——之七写明这一档唯一允许的用途是拿去问厂商。
2. **分支被走到不等于分支算得对。**本节验证的是输出形态、词汇与发射点的对应，全部是形式性质。桩画像里 `5242880`、`coredump_mode=444`、`vpu_37xx_v0.0.bin` 这些值本身没有语义（之四已声明样例数值不在第卅章核对范围），所以"这一格算出的判定正确"这件事在本文档里从未被证明过，只能在读者自己的两份真实账本上复核。
3. **覆盖率不覆盖缺口。**之十登记的六条缺口（`driver_override` 伴读、`no_rproc_face` 三类枚举、glink 边数、值文本不参与判定、K6 缺链列、K8 缺 `value_source` 列）在本节一次都没有出现——它们没有分支可走。"55 个发射点全部触发"与"脚本 2 的采出口径不完整"同时成立，这两句话不矛盾：前者说的是已写下的判定，后者说的是没写下的判定。第廿六章之九第 4 条反对裁剪分母，同样的理由反对把"分支全覆盖"读成"信息全覆盖"。

### 廿八之十 前向承诺收敛表：前面章节替脚本 2 许过的九条

第廿七章为脚本 1 做过一次同性质的收敛。这一张表针对的是**脚本 2**：把前面各章里"第廿八章脚本 2 将会……"的句子逐条找出来，标出它在本章的实际落点。三种落点的定义与第廿七章之十相同——**兑现**（按原句实现）、**改述**（原句的意图实现了，但实现形态与原句不同，且差异本身是有理由的）、**缺口**（没实现，转第卅章登记）。

| 出处（原句位置） | 原句要求 | 落点 | 实际实现 / 为什么没实现 |
| --- | --- | --- | --- |
| 第十七章之二 | 把"读到 `unknown`"的三义（状态确为 ATTACHED / 类型越界回退 / 权限不足）分成三个不同值，否则 K 类指标会把读不到记成 attached | **改述** | 脚本 1 的 `P-14` 四分支把第三义（权限不足）与前两义分开：权限不足走 `denied_value`，根本不存在走 `absent`，读到字符串走 `present` 且 `label` 列保留原文。前两义的区分需要解析状态字符串语义，脚本 2 的 `D-08` 不做——见下方"值文本不参与判定"一条 |
| 第十七章之六 | 逐属性 `stat` 取八进制模式并原样记录；`0444` 时标 `degraded_by_vendor` | **兑现** | `D-09` 直接比较两侧 `coredump_mode=` 值，`644 → 444` 落 `mode_tightened` 并由 `L\|rollup` 折算成降一级；`444 → 644` 只登记为 `mode_relaxed`，不动级别 |
| 第十七章之八 | `/dev/remoteproc*` 写成"若存在则记录，否则记 `absent`"，不假定可读 | **兑现（在脚本 1）** | 该面的存在性是一条姿势（`P-20`），脚本 2 的 `D-11` 负责跟踪它在两次快照之间的拒绝态迁移。ioctl 可查询能力仍是第廿五章之六的能力事实，不进脚本 |
| 第十七章之十 | 脚本 2 第二段同时枚举 `remoteproc`、`/sys/kernel/debug/remoteproc` 与媒体驱动 platform device 三类，把只出现在后两类者标 `no_rproc_face` | **缺口** | 三条都没实现。原因不是忘了：`/sys/kernel/debug/remoteproc` 在本域默认不可达（`P-21` 在四个画像里只有 `wide` 能列 debugfs 根），platform device 枚举需要 `/sys/bus/platform/devices`，而第廿章的姿势表里没有这一条。补姿势等于改 `K2` 的固定分母 26，属于第廿六章之九第 4 条警告的"裁剪分母"同类操作。转第卅章 |
| 第十八章之四 | 采集 `driver_override` 当前值作为 rpmsg 名字读数的判别位 | **缺口** | 脚本 1 的 26 条姿势里没有它，脚本 2 因此没有可比的对象。`D-19` 把这件事写成一个**断言**而不是分支：若台账里出现 `driver_override`，脚本直接 `exit 9`，提示"先把它写进第廿章的表，不要偷偷加姿势"。同时它给出硬约束——任何 rpmsg 名字读数不得进 K8 分子。转第卅章 |
| 第十八章之六 | glink 边数与 `rpmsg_ctrl` 节点数分开计数 | **缺口（一侧有、一侧无）** | `/dev/rpmsg_ctrl0` 的存在性是一条姿势，因此"节点数"这一侧有格子；glink 边数需要遍历 DT 或 debugfs 通道表，没有格子。缺任何一侧都不构成"分开计数"，只会在 `D-11` / `D-08` 里留下两个孤立的读数。转第卅章 |
| 第廿二章之二 | 判定条件只能来自同一台设备的实测读数，或脚本 2 的两次快照之差 | **兑现** | 这就是脚本 2 的存在理由。`D-01` 是这条规则的守门分支：两份快照的 target 全集不同时，后续所有比对都被判为"采集面本身变了"，禁止直接进入成因归因 |
| 第廿四章之十五 | T3 记账面作为漂移比对的一类伴读 | **兑现** | `D-15` 三条迁移对（`absent\|denied_value → present` 记 `t3_appeared`、反向记 `t3_lost`、其它换类记 `t3_drift`），并在 `note` 里保留 K1 的两个前置条件：分子可变，但必须 CVE 编号与公告主键逐字相等 |
| 第廿六章之六 K6 / K7 / K8 | 三个指标的采出口径由第廿八章给出 | **改述** | 本章第之八节给出的是**分子输入**与**分母常量**，不是分子。K6 缺链列、K7 只有第四项、K8 缺 `value_source` 列——三处各自转第卅章，且脚本输出的 `note` 字段里逐字带着这三条限制 |

**"值文本不参与判定"是这张表里唯一一条脚本自身的语义局限，必须单列。** `D-08` 的 `stable` 臂只比较两侧的**判定**（`present` vs `present`），不比较 `label` 列里的值原文。后果是：如果两次快照都读到了 `state`，但一次是 `running`、一次是 `attached` 或 `unknown`，`D-08` 输出的判定仍是 `stable`。这不构成隐藏——`before` 与 `after` 两列照原样打印值文本，人工读表看得见；但它意味着**自动化的那一级没有覆盖状态字符串的语义变化**，第十七章之二担心的"把读不到记成 attached"在"两侧都读得到"这一格上并未由脚本闭合。把它登记为缺口，而不是改成一个当前无法触发的分支，是同一份报告对"分支必须可达"这条自订规则的遵守方式。


### 廿八之十一 运行前置、实测耗时与七条限制

**前置只有四条，而且第 1 条决定了这个脚本应该在哪里跑。**

1. **它不需要设备，也不需要 `adb`。** 脚本 2 的两个入参就是脚本 1 的两次输出文件。这意味着比对可以在工作机上做，把两份台账从设备侧带出来的动作只属于脚本 1。反过来也成立：在设备上的 `adb shell` 里跑脚本 2 是没有意义的，那里没有这两份文件。
2. **两份快照必须来自同一版脚本 1。** 这不是建议而是分支：`D-01` 会比较两侧的 target 全集，不同就输出 `record_set_drift` 并把后续归因挂起。改了脚本 1 的姿势表而忘记重采，是这一格最常见的用途。
3. **需要 bash，不能用 `sh`。** 脚本里有 4 处进程替换（`comm -23 <(cmd1) <(cmd2)` 形态，在 `D-01` 与 `D-03`），`dash`/Android `sh` 不支持。除此之外全是 POSIX 级构造：`case`、`${v:-d}`、`$( )`、`printf`。
4. **外部命令只用到 `awk`、`sort`、`comm`、`tr`、`sed`、`grep`、`head`、`wc`、`cat`。** awk 侧只用 POSIX 特性（`-F`、`match()`/`substr()`、`gsub()`、字段变量与 `print`），不依赖 GNU 扩展（无 `asort`、`gensub`、`IGNORECASE`），因此 GNU awk、mawk、toybox awk 都能跑——这一条对本域重要，因为设备侧的 awk 是 toybox。`LC_ALL=C` 在开头强制设定并导出，否则 `comm` 与 `sort` 在本地化排序下会给出不同的差集。

**实测耗时与它的真实成因。** 在本机的 Windows Git Bash 下，10 对采样的运行时间稳定在 **44–54 秒/对**（矩阵实测值：44、45、48、49、47、54、48、50、48、49）。为找出瓶颈，用一个计数垫片把每个外部命令的调用次数记下来（垫片自己 exec 真实命令，不改变行为），一次 `good → user` 的运行结果是：

| 命令 | 调用次数 |
| --- | --- |
| `tr` | 355 |
| `sed` | 336 |
| `awk` | 225 |
| `sort` | 23 |
| `wc` | 16 |
| `grep` | 5 |
| `comm` | 4 |
| `head` | 2 |
| 合计 | **970** |

结论与直觉相反：**瓶颈不是取列的 awk，而是字段清洗。** `rec()` 对八个字段各调用一次 `san()`，而 `san()` 是一条 `tr` 加一条 `sed`——42 条记录 × 8 字段 = 336，正好是 `sed` 的次数（`tr` 多出的 19 次来自 `PN()`、`cnt_of()` 等辅助函数）。也就是说，这台机器上这 50 秒里约七成花在 fork 一个 `tr` 或 `sed` 上。

为什么不把它改成一个 awk 程序一次通读：与第廿七章之九为脚本 1 给的理由相同——**审计成本**。21 个块的判定逻辑一旦被压进一个 awk 程序，第廿九章的"分支必须可达"验收就要去读 awk 的控制流，而不是对着 21 个 `rec` 调用点逐条勾。这个取舍在这里同样明确记在账上：**用 50 秒换 21 个可逐行核对的分支点**。真要提速，唯一不牺牲审计的做法是把 `san()` 换成 bash 内建替换（竖线、换行与制表符各压成空格，再折叠连续空格并去首尾），那会把这 691 次外部调用降到 0 次；本报告不改动它，因为改动后第之六节的样例就需要重新生成，而样例的价值恰恰在于它是用与读者拿到的完全相同的脚本跑出来的。

**七条限制，逐条给出理由。**

1. **值文本不参与判定。** `D-08` 与 `D-13b` 的 `stable` 臂只比判定不比 `label` 里的值原文；两侧都读到但内容不同时仍输出 `stable`（值文本照样打印在两列里，人工可读）。这是第十七章之二那条"三义"承诺没有闭合的部分，已在第之十节登记为缺口。
2. **`key` 列不能反查回快照。** `san()` 会把字段内的竖线压成空格，因此 `D-06` 里 `/sys/class/$c` 展开出的目标、以及 `D-19` 的 `companion driver_override`，在 `key` 列里看到的不是原始竖线形态。联录要用 `did` 列而不是 `key` 列。
3. **不做链映射。** 台账没有链列（第之八节），K6 的分子折算必须人对照第廿六章的七条链表。
4. **不跨设备、不跨厂商。** 输入是两份快照，脚本不校验它们来自同一台机器；这个校验属于人工流程。第廿六章之四把 `cross_vendor_compare` 恒为 `forbidden`，因此拿两个型号的快照跑脚本 2 得到的任何输出都不可用。
5. **顺序有意义，且不对称。** `A → B` 与 `B → A` 不是同一张表：矩阵里 `good → user` 给出 `mode_tightened` + `t3_appeared`，而 `user → good` 给出 `mode_relaxed` + `t3_lost`。两份快照哪一份是"前"，必须由采集时间决定，不能由文件名字母序决定。另外 `rows_for` 只从 `A` 侧取目标全集，因此新增的目标（只存在于 `B`）不会被逐项展开——它会在 `D-01` 的 `record_set_drift` 里被抓到，但不会有属于它自己的 `D-` 行。
6. **循环展开只覆盖 `A` 侧已有的格子。** 与上一条同源，是同一件事的技术表述：`for t in $(rows_for P-12 "$A")` 这类写法决定了新增目标只在 `D-01` 层面可见。把它当已知限制而不是 bug，因为"新增一个目标"本身就是 `record_set_drift`。
7. **样例数值来自桩画像。** 四个画像（含本章新增的 `alt`）都不是设备；样例里的字节数、行数、属性模式全部是合成值。第卅章的取回记录只对源码行号与文档句子负责，不对这些数值负责。

**退出码。** `0` 正常输出（包括"有漂移"的情况——漂移不是错误）；`2` 用法错误或某个快照文件不可读；`9` 契约被破坏，两种情形：`rec()` 的 arity 不是 8（脚本内部缺陷，会同时打印 `BUG: rec arity`），或台账里出现了未在第廿章登记的 `driver_override`（`D-19` 的断言）。后一条在第之九节的验收运行里被真实触发过一次。


### 廿八之十二 本章结论

1. **脚本 2 只有一个用途：把"读数变了"与"安全状态变了"分开。** 34 个非 `stable` 值里，只有 1 个能改判定级别（`mode_tightened`，降一级，第廿六章之二第 6 条），2 个可能改指标分子（`t3_appeared` / `t3_lost`，且带主键逐字相等的前置），4 个是前提类（出现即挂起后续归因），其余 27 个只登记。这张表的存在意义就在这三个数字上——**剩下 31 个值的全部作用是阻止读者把它们当成前三个来用。**
2. **42 条记录、21 个块、54 行输出是固定形状。** 块数由第廿章的 26 条姿势折叠而来（21 条各有专块，5 条只被泛臂覆盖），记录数由块内展开决定。固定形状是刻意的：一旦某格缺失，读者数行数就能发现，不需要读脚本。
3. **分支覆盖由 6 对采样即可完成，另外 4 对是冗余复核。** 第之九节给出这个最小覆盖集与逐值对照。同时，契约断言用一份被人为改坏的台账真实触发过一次 `exit 9`，因此"`D-19` 的守卫会响"不是断言而是观测。
4. **为了分支可达而新增第四个画像（`alt`），比留一个永不触发的分支诚实。** 这条是方法层的：`alt` 不是设备、不进第卅章的取回记录、只改四处且逐处写明它让哪条分支可达。第廿九章会把"从不触发的分支算缺陷而不是算遗漏"列成验收条目，`alt` 就是为这条目准备的。
5. **三条前向承诺没有兑现，且没有用偷偷加姿势来兑现。** `driver_override` 伴读（第十八章之四）、`no_rproc_face` 三类枚举（第十七章之十）、glink 边数与 `rpmsg_ctrl` 分开计数（第十八章之六）——三者都要求新增姿势，而第廿六章之九第 4 条把 `K2` 的分母钉在 26。取舍是：**固定分母优先于覆盖率**，三条一起转第卅章缺口清单，脚本里只留下 `D-19` 那句"要补采就先把它写进第廿章的表"。
6. **值文本不参与判定是本脚本自身最大的语义局限。** 两侧都读到、内容却不同时，`D-08` 与 `D-13b` 仍输出 `stable`。信息没有丢（`before`/`after` 两列打印原值），但自动那一级没覆盖；把它写成限制而不是补一个当前无法触发的分支，是同一条纪律的两次应用。
7. **第廿九章做两件事：** 把本报告全篇的十二条"看起来能推其实不能推"汇总成误区清单并逐条给反例；对脚本 1 的 17 值 × 26 姿势与脚本 2 的 35 值 × 21 块做一次并表的覆盖验收，并给出第卅章要用的"不可断言"清单。

## 廿九、验收：两个脚本的并表覆盖、十二条误区与一份"不可断言"清单

第廿七章和第廿八章各自写了"分支覆盖实测"，但两处都是自证：脚本 1 用自己的画像证明自己的分支，脚本 2 用自己的画像证明自己的分支。本章把它们并到一张台面上，做三件事——给出跨两个脚本的验收判据与实测结果、把全篇散落的"看起来能推其实不能推"收敛成十二条误区、把验收之后仍然无法断言的东西单独登记交给第卅章。第三件事最重要：一份只写成功案例的验收表，和一份没有验收的报告，缺陷是同一种。

### 廿九之一 验收的三个对象，与判据分别从哪一章来

验收不是"跑一遍看看有没有报错"。本报告的可执行契约有三处，每处的判据都在别处已经写下，本章只是把它们逐条对上：

| 验收对象 | 规格来源 | 判据条数 | 实测位置 |
| --- | --- | --- | --- |
| 脚本 1（`w17-posture.sh`，354 行） | 第廿章那张 26 行表 | 4 条硬约束 + 1 条"表与脚本行数一致" | 之八，复核数据来自第廿七章之九 |
| 脚本 2（`w17-drift.sh`，324 行） | 第廿八章之一/之二两份契约 | 输入 4 条 + 输出 8 列 + 固定 54 行 | 之九，实测数据来自第廿八章之九 |
| 正文里的每个行号与每句引用 | 第卅章取回记录 | "取不到原句即为伪造" | 第卅章 |

第廿章给脚本 1 定过四条硬约束，本章按这四条核，不另加标准：

1. **每个 `P-id` 在脚本里有且只有一个分支**，`pid` 列原样回填表里的编号。表有 26 行，脚本必须有 26 个可触发分支；**从不触发的分支算脚本的缺陷，不算样例的遗漏。**
2. **`verdict` 只能取词汇表的值**，不接受自由文本、不接受布尔。第廿章列了 16 个，第廿七章之六把 `unlistable_value` 转正后是 17 个。
3. **`rule` 列必须是那张表"管住它的规则"列里的一条 `文件:行号`**，与第卅章取回记录一一对得上；对不上的记录视为伪造。
4. **`source` 只能是 `device_read` 或 `static_baseline`。**

第（2）条还要加一条来自第廿三章之八第（4）条的子判据，因为它是这一类采集最容易失手的形态：**`absent`（路径不存在）、`unlistable`（父目录不可列但已知名可 `stat`）、`eacces`（能 `stat` 不能 `read`）、`vendor_dependent`（AOSP 无授权，结果取决于厂商补丁）这四个状态在输出里必须是四个不同的字符串，合并即缺陷。**脚本 1 把 `eacces` 又按标签可读性拆成 `eacces_typed` / `eacces_untagged`，所以本域的字符串是五个而不是四个。之五给出这一条的机器核验方式。

脚本 2 的判据不同源，它要核的是"输出形态是否与契约同构"：42 条记录、21 个块、5 条 rollup、6 行禁止推出、1 行表头，恒 54 行；`did` 只能取 21 个块号之一；`face` 只能取 V1–V8 或 `ALL`/`DECL`；`verdict` 只能取 35 个词汇值之一；`key` 内不得含 `|`。这几条全部可以在一份输出上用一行 `awk` 核，之九给出该 `awk`。

最后一个方法论约束，贯穿本章所有表格：**验收能断言的只有形式性质。**分支可达、取值合法、行数恒定、退出码正确——这些是形式。某一格算出的判定在语义上对不对，本报告一次都没有证明过，也不可能用桩画像证明（第廿八章之九第 2 条）。所以本章的每一张"通过"表，读到的意思都是"脚本按它自己声明的规则输出了"，不是"脚本按真实策略集给出了正确判定"。

### 廿九之二 抽回方式：按围栏区间抽文件，而不是从渲染页复制

第廿七章末尾承诺过"第廿九章按围栏区间抽回原文件做 `bash -n` 与画像运行比对"。这件事必须写成可复现的动作，否则"内嵌脚本与样例一致"这句话只能靠自觉。四个区间，行号是本次装配后的实测值：

| 抽回对象 | 所在文件 | 围栏区间 | 抽出行区间 | 行数 | 与磁盘原件比对 |
| --- | --- | --- | --- | --- | --- |
| 脚本 1 正文 | 第廿七章 | 21 / 376 | 22–375 | 354 | `cmp` 通过 |
| 脚本 1 样例输出（`good`） | 第廿七章 | 392 / 463 | 393–462 | 70 | `cmp` 通过 |
| 脚本 2 正文 | 第廿八章 | 78 / 403 | 79–402 | 324 | `cmp` 通过 |
| 脚本 2 样例输出（`good → user`） | 第廿八章 | 433 / 488 | 434–487 | 54 | `cmp` 通过 |

抽回与校验的完整动作（`<27>`、`<28>` 是两章的 markdown 源文件）：

```bash
sed -n '22,375p' <27> > x1.sh   ; bash -n x1.sh
sed -n '393,462p' <27> > x1.out ; cmp x1.sh w17-posture.sh ; cmp x1.out snap-good.txt
sed -n '79,402p'  <28> > x2.sh   ; bash -n x2.sh
sed -n '434,487p' <28> > x2.out  ; cmp x2.sh w17-drift.sh
bash x2.sh snap-good.txt snap-user.txt > x2.rerun
cmp x2.rerun x2.out && echo RERUN_MATCHES
```

三条要注意的。**行号会随排版漂移**，所以这四行区间的正确用法是"先按 `^```bash` 定位围栏，再取区间"，而不是把上面的数字当常量抄进 CI。**`cmp` 只保证字节一致，不保证运行**，因此第 6、7 行是必需的：抽出来的脚本必须真的再跑一次，跑出来的东西必须与报告里那 54 行逐字节相同——本报告实测 `rc=0 lines=54 err=0字节 RERUN_MATCHES`。**样例文件本身不能当脚本用**：第廿七章那份 70 行样例是脚本 1 的输出，脚本 2 的输入，它同时充当两个角色，这也是为什么脚本 2 的表头/rollup 排除条件写成 `$1 ~ /^P-/` 而不是按行数切。

### 廿九之三 并表：两个脚本的形态、词汇与分母放在一起

两个脚本的词汇表是正交的两套（第廿八章之二），所以并表时最容易出事的是把两边的数字相加。下表把它们并排而不求和：

| 维度 | 脚本 1 `w17-posture.sh` | 脚本 2 `w17-drift.sh` |
| --- | --- | --- |
| 输入 | 一台设备（或桩 `adb`） | 两份脚本 1 的输出 |
| 是否碰设备 | 是，全部是只读命令 | 否，一次都不 |
| 正文行数 | 354 | 324 |
| 记录条数 | 60 条 `P-` + 3 条 rollup | 42 条 `D-` + 5 条 rollup |
| 输出总行数 | 70（恒定） | 54（恒定） |
| 词汇表 | 17 值（判定档位） | 35 值（两次判定之间的关系） |
| 词汇表实际可达 | 16（`not_inferable` 不由脚本产生） | 35（含 `stable`） |
| 规格来源 | 第廿章 26 行表 | 第廿八章之二 21 块表 |
| 分母 | K2 = 26，永不裁剪 | 无分母；`S` 行只报记录数与块数 |
| 分支数 | 26 个 `P-id` 分支 | 55 个记录发射点 |
| 触发所需的画像 | 3 个（`good` / `user` / `wide`） | 6 对（最小覆盖集，见第廿八章之九） |
| 契约违规退出 | 不适用（无跨文件契约） | `exit 9`，实测触发过一次 |
| 参数错误退出 | `exit 2` | `exit 2`，实测三种入参都是 2 |

两点值得单独指出。**"输出恒定为 70 / 54 行"是这两个脚本最重要的可机检性质**：真实设备上少采一格会被填成 `-` 而不是少一行，多一个新对象也不会多一行——所以任何人拿到一份 69 行或 55 行的输出，不需要读内容就知道采集或比对出了问题。第二点是两套词汇的可达性差别：脚本 1 有一个值（`not_inferable`）按设计不由脚本产生，因此它的可达上限是 16，验收时如果要求 17 就是给脚本加一条永不触发的分支；脚本 2 的 35 个值全部可达，因为它没有"给人看而不给机器判"的档位。这两句放在一起，正好是第廿八章之九那条"从不触发的分支算缺陷"在两个脚本上的两种不同结局：**一个靠声明排除，一个靠画像跑满。**

### 廿九之四 画像层：四个画像各自证明了什么

| 画像 | 记录数 | 触发的 `P-id` | 取值种数 | K2 `num` | `pids_with_failure` | 它存在的理由 |
| --- | --- | --- | --- | --- | --- | --- |
| `good` | 60 | 26 / 26 | 12 | 10 / 26 | 16 | 基线：策略挡住的形态 |
| `user` | 60 | 26 / 26 | 16 | 10 / 26 | 16 | 构建维度：`muted_by_build`、`filtered_by_policy`、`bound`、`degraded_by_vendor` 只在这里出现 |
| `wide` | 60 | 26 / 26 | 7 | 13 / 26 | 13 | 授权维度：把 `ok` 分支跑出来 |
| `alt` | 60 | 26 / 26 | 12 | 10 / 26 | 16 | 内容维度：同一格换了内容，只为脚本 2 的分支可达 |

这张表里有两处实测结果值得先看，因为它们各自堵住一个误区。**第一，`wide` 只有 7 种取值却拿到最高 K2（13/26）**——把授权面打开会让很多格收敛成 `present`，取值多样性反而下降。这正是之六第 12 条要写的东西：可读项数与取值种数是两个方向相反的数。**第二，`alt` 的取值集合与 `good` 完全相同（12 种）**，它改的四处（`cdsp.mdt` 字节数、标签行数、`remoteproc0/state` 的失败原因、可写属性模式）全部落在已有档位内部，只换格子里的内容。所以 `alt` 对脚本 1 的词汇覆盖率没有任何贡献——这是刻意的，之六第 1 条用它作为"取值集合相同不等于读数相同"的实例。

四份画像的并集给出脚本 1 的可达词汇：`absent`、`bound`、`debugfs_unreachable`、`degraded_by_vendor`、`denied_log`、`denied_value`、`eacces_typed`、`eacces_untagged`、`filtered_by_policy`、`label_unreadable`、`muted_by_build`、`openable`、`present`、`unlistable`、`unlistable_value`、`vendor_dependent` = **16 种**。第 17 种 `not_inferable` 零次，且脚本正文里也没有这个词——它是台账字段用的词，不是脚本判定用的词（第廿七章之六）。

### 廿九之五 四态不压扁：一条可以机器核的判据

第廿三章之八第（4）条要求"四个状态必须是四个不同字符串，合并即缺陷"。这条判据不能靠读脚本确认（读的人总会看到自己想看到的），所以给它一个可执行的核法，分三步：

1. **列合法值。**从第廿章那张表的"判定"列抄出 17 值清单，作为白名单文件 `v17.txt`（每行一个）。
2. **列实际输出值。**`awk -F'|' '$1 ~ /^P-/ {print $4}' snap-*.txt | sort -u`。四画像并集给 16 行，与白名单求差，只允许差 `not_inferable`。用 `comm -3` 做这一步：`comm -3 v17.txt actual.txt` 的输出必须恰好是一行 `not_inferable`。
3. **核四态各自独立成行。**`absent` / `unlistable` / `eacces_typed` / `eacces_untagged` / `vendor_dependent` 五个字符串必须都出现在 `actual.txt` 里，且**没有任何一行的 `verdict` 字段同时含两个**（`awk -F'|' '$1 ~ /^P-/ && ($4 ~ /absent.*unlistable|unlistable.*absent|eacces_typed.*eacces_untagged/) {c++} END {print c+0}'` 必须为 `0`）。第三步是防止将来"顺手合并两个看起来一样的档位"。

实测：四份画像下 `verdict` 列去重得到 16 个字符串；白名单 17 个；差集为 `{not_inferable}`；五个状态字符串全部独立出现；两个"合并检测"模式命中 0 行。这五条合起来就是第廿三章那条裁决的机器形态。它防的具体事故也在报告里出现过一次：第廿章 P-16 的三态（目录在不在 / 模式能不能读 / 值能不能读）如果不分列，`/sys/module/ivpu/parameters` 的"未装载"与"已装载但值被拒"就会都写成 `absent`，而这两件事在第廿八章 `D-10` 里是两个不同的 `before`/`after` 组合。
### 廿九之六 十二条误区：错误句式、它为什么像对的、反例在哪一章、台账里该怎么写

这一节是第廿九章的主体。选取标准只有一条：**这句话在报告前面的某一章里被具体证伪过。**所以每条都给出处，不给"理论上不应该"。句式全部写成报表里真会出现的样子，包括它们错在哪个词上。

**第 1 条 反例。** `/vendor/firmware_mnt/image/` 下命中文件名从 2 个变成 3 个，或者 `cdsp.mdt` 的字节数从 5242880 变成 5246976——"固件更新过。"
*为什么像对的*：目录内容和镜像文件都在同一层，改动是真实的读数变化。
*反例在哪*：第廿章之八第 2 条已裁决 V1 的名字与长度不是版本字段；第廿八章 `alt` 画像把这条变成可验证的构造——它把 `cdsp.mdt` 的字节数改 4096、文件名一个都不动，脚本 1 的取值集合与 `good` 完全相同（12 种），变的只有 `D-04 size_drift` 一行。如果字节数变化能读成版本，那 `alt` 就成了一台设备，而它只是一个 `case` 分支。
*台账写法*：`name_set_changed = yes` / `size_changed = yes`，版本字段留 `none`，并注明"文件名字段，非版本字段（第廿三章之八第（6）条）"。

**第 2 条 反例。** "驱动打了 `Firmware git sha: …`，所以能拿它核对固件是否已修。"
*为什么像对的*：git sha 直指一次构建，是本次报告里语义最强的溯源字段。
*反例在哪*：第十一章之三。这条读数**只在上游 `v7.3` 存在**（`panthor_fw.c:701` 的 `static const char git_sha_header[] = "git_sha: ";`，成功时 `:729-730` 打印），而 `android16-6.12` 这一侧根本没有这条分支；即使在上游，元数据损坏时的处理是非致命——`drm_err(..., "Firmware build info corrupt")` 之后注释逐字 `/* We don't need the build info, so continue */`（`:712-714`），照样装载；最后，它的载体是内核日志，而内核日志对 shell 域被 `private/app.te:660-661` 那条 `neverallow` 挡住、V6 面在样例里两侧都是 `denied_log`。
*台账写法*：`build_id_source = kmsg`、`build_id_readable = no`，并把"读不到"与"该树不存在这条读数"分成两个字段——后者只能由源码核验得出，设备永远给不出。

**第 3 条 反例。** "`ro.build.version.security_patch` 从 2026-06-05 变成 2026-09-01，所以固件也更新了三个月。"
*为什么像对的*：SPL 是设备上一个能被采集读到的日期字段，而且它确实变了。
*反例在哪*：第八章写明 SPL 只反映系统分区；第廿八章样例 `D-14b` 同时给出 `ro.build.version.security_patch` 变了、`ro.vendor.build.fingerprint` 变了、而**器件侧两行数值读数一字未动**（`D-08` 两条 `denied_value -|denied_value -`，`D-18` `openable|openable`）。Android 侧口径变新与器件镜像新旧在同一份输出里被同时观察到，两者不构成因果。
*台账写法*：`android_side_baseline_changed = yes`、`device_side_evidence = none`，禁止把前者的值填进后者的任何列。

**第 4 条 反例。** "Mali CSF 的装载检查里没有'版本过低'这一条，说明 Arm 不关心固件安全。"
*为什么像对的*：Adreno 侧确实存在驱动侧最低微码版本门（第十二章之三），两相比对，一边有一边没有，很容易读成态度差异。
*反例在哪*：第十一章之二那张检查表检查的是长度、对齐、地址区间、标志位、入口类型和格式大版本——它是一张**镜像可用性**检查表，不是安全表。把"没有"读成"不关心"之后，就没人再注意它真正的含义：**在这条实现里，"旧镜像是否可用"根本不由驱动决定，因此它不可能作为设备侧的自证证据出现。**同一张表里还有一条更容易被漏掉的：`:532-533` 是全表唯一"看见不支持的东西但继续跑"的分支，而它的内容恰好是 protected-mode 条目——这是实现取向，不是安全取向的证明。
*台账写法*：`driver_side_version_gate = absent_in_this_implementation`，并加一条 `attribution = cannot_self_attest`；不允许出现任何评价性用词。

**第 5 条 反例。** 把 `GPU_REVID` 读回来的值写进 `fw_version` 列。
*为什么像对的*：它名字里就有 rev，打印出来长得像版本号，而且是硬件直接给的，比固件自报"可信得多"。
*反例在哪*：第十章。`GPU_CSF_ID`（`panthor_regs.h:35`，偏移 `0x1C`）与 `GPU_REVID`（`:130`，偏移 `0x280`）**都是寄存器**，是硬件身份不是镜像身份。同一段打印（`dev_info` / `drm_info` 的 `mali-%s id 0x%x major 0x%x minor 0x%x status 0x%x`）里那四个数全是寄存器值；而它们全部只进内核日志，标准姿势拿不到。硬件 revision 与固件版本在故障归因上是两件事：前者决定这颗 IP 是哪一版硅，后者决定跑的是哪份镜像。
*台账写法*：单列 `hw_revision`，`value_source = register_read`，并且永远不填进任何"版本"命名列——第廿二章规则四的编号层裁决要求字段名与来源同列。

**第 6 条 反例。** "debugfs 里有个 `fw_level`，读它就能知道固件版本或安全级别。"
*为什么像对的*：路径里带 `fw`，名字里带 `level`，权限位 0644 看起来就是给人读的。
*反例在哪*：第十五章之五。那个节点是**调试消息级别**：`int venus_fw_debug = HFI_DEBUG_MSG_ERROR | HFI_DEBUG_MSG_FATAL;`（`hfi_venus.c:133`）由 `venus_sys_set_debug(hdev, venus_fw_debug)`（`:947-949`，在 `venus_sys_set_default_properties()` 内，`:941`）写进固件，读它只读回一个掩码，写它改的是 DSP 打不打日志。可达性另有一层：`private/shell.te` 只授予 shell 若干专项 debugfs 类型（`:7`、`:8-9`、`:10`、`:14`），**没有 venus 的 debugfs 类型**，所以标准 adb 姿势连读都不该假设能读；脚本 1 把它归到"预期失败并记录 errno"那一格。
*台账写法*：`node_purpose = debug_message_mask`、`version_bearing = no`；名字里的 `fw`/`level`/`ver` 不构成字段语义，语义只来自赋值点。

**第 7 条 反例。** "`dmesg` 里没有 `vdec`，所以这台设备没有这条跨核通道。"
*为什么像对的*：通道有名字，名字会进日志，这是大多数内核子系统的行为。
*反例在哪*：第十八章之一。内核里存在通道名，但**从不打印它、从不导出它**：VPU 后端把它存在数组里，SCP 后端在注册时直接丢弃；能看到的只有两条与通道身份无关的错误路径——`pr_err("No such ipi id = %d\n", id)`（`mtk_vpu.c:759`）与 `pr_err("No handler for ipi id = %d\n", id)`（`mtk_scp.c:115`）。也就是说"看见 `vdec` 字样"取决于你是否碰巧命中一条打印 id 的失败分支。再叠一层：V6 日志面在样例里两侧都是 `denied_log`（`app.te:660-661`），采集者本来就没看过日志。
*台账写法*：通道在场性只能记 `absent_in_kernel_output`（这是"内核不导出"的意思），不能记 `absent_on_device`；两者混用即第廿三章之八第（1）条那个错误。

**第 8 条 反例。** "debugfs 读不到 = 厂商把固件面锁了。"
*为什么像对的*：同一颗器件，AOSP 策略里有授权、设备上却读不到，看起来只剩厂商改策略一种解释。
*反例在哪*：第十九章之六。debugfs 的开关不在策略类型上，而在**两个构建变量**上：`target_enforce_debugfs_restriction` 与 `target_build_variant`（`public/te_macros:591` 的 `no_debugfs_restriction`、`:614` 的 `userdebug_or_eng`，注释句分别为 `# SELinux rules which apply to devices that do not have debugfs restrictions in non-user builds.` 与 `# SELinux rules which apply only to userdebug or eng builds`，`:589`、`:590`、`:611-612`）。三个后果同时成立：shell 的四条 debugfs 授权（`private/shell.te:7-10`）里只有 tracing 相关且 `:13-15` 被 `userdebug_or_eng` 包住，user 构建上连它都没有；venus 的 `venus_dbgfs` 类型不在其中；采集器的整块授权（`private/dumpstate.te:290-292`）被包在 `no_debugfs_restriction` 里，**启用 debugfs restriction 的设备上 bugreport 根本不含 debugfs 内容**。第廿八章样例把这条变成了输出：`D-13` 的值是 `build_switch`，注记写明"不是厂商收紧；不进'厂商收紧'统计"。
*台账写法*：先记 `build_variant` 与 `debugfs_restriction_enforced` 两个字段，再记 `vendor_hardening`；前两个为 unknown 时第三个不得填 yes——这就是 `muted_by_build` 与 `degraded_by_vendor` 分成两个值的原因。

**第 9 条 反例。** "`service list` 少了两个服务，`dumpsys media.codec` 也变了，所以媒体固件的在场性变了。"
*为什么像对的*：声明、服务、在场三者层层依赖，服务没了通常意味着后端没了。
*反例在哪*：第廿章之八第 4 条把声明变化与在场变化分开了；第廿八章 `D-17` 七行专门盯这一件事，样例里七行全是 `stable`，注记每行重复那句"声明变化不等于在场变化"。`dumpsys media.codec` 与 `dumpsys cameraserver` 两格两侧都是 `vendor_dependent`——也就是说这两格能不能列出东西本来就取决于厂商实现，它的变化连"服务在不在"都证不了，更证不了固件。
*台账写法*：`declaration_changed = yes` 与 `presence_changed = unknown` 分两列；在场性只能由 V1/V4 面的读数支撑。

**第 10 条 反例。** "高通那张表的厂商号点开的公告页取不到正文，所以这批缺陷不可修。"
*为什么像对的*：外键指向的入口不可读，看起来这条线索就到此为止了。
*反例在哪*：第廿四章之十一。`QC-CR#` 一族 5/5 无星，**有可用外键，缺的是入口可达性**；第九章实测高通 `www` 是 SPA 外壳、`docs` 侧任意路径 302 到登录页 `account.qualcomm.com - Sign In`（两个路径分别 149,716 与 149,819 字节；更早一轮同一入口取到的是 52,406 字节的壳页，本次已换代）。这与"闭源组件表那种没有厂商号、Subcomponent 是占位串"是两类完全不同的缺口。本条的字段区分就是为这一句立的：`vendor_entry_fetch = pending` 属于**流程状态**，`no_device_side_evidence` 属于**证据状态**，把前者写成后者，等于宣告一个还没做的事永远做不了。
*台账写法*：`vendor_ref_family = QC-CR#`、`vendor_entry_fetch = pending`，并且**不加** `no_device_side_evidence`；只有第廿四章之十二那种"无厂商号 + 占位串"的行才加它。

**第 11 条 反例。** "公告读得越多，器件越安全。"
*为什么像对的*：读公告是安全运营的动作，动作量与安全感的正相关是行业习惯。
*反例在哪*：第廿五章之四数过主责负载：**O1 三组、O2 两组、O3 四组、O4 三组、O5 一组**，十三组公开案例里能直接派给厂商的只有三组，能派给自己（O3/O4/O5）的有八组。多读公告改进的是 K9（我们自己映射能力的覆盖率）和 K10（归属不明条目占比），而这两个指标衡量的从来不是器件状态。第廿六章之四还额外规定 `cross_vendor_compare` 恒为 `forbidden`，唯一允许的跨厂比较就是 K9 这种"比我们的采集能力"。
*台账写法*：把"本期新增可读条目数"挂到 K9/K10，把"器件风险变化"挂到 L 级判定模型；L 级只能由送达证据（O3）改变，而送达证据不来自你多读了几页公告。

**第 12 条 反例。** "这次采集比上次多读到两个值，所以覆盖率提升了。"
*为什么像对的*：分子看起来确实变大了——同一台设备、同一套脚本，多两个读数。
*反例在哪*：第廿七章之九第 2 条是实测的：`user` 画像比 `good` 多读到一个 `bound` 和一个模式读数，**K2 的 `num` 仍是 10 / 26**，因为 K2 的分子按 P-id 全分支可读计，不是按值计。更反直觉的是第廿九章之四这张表：把授权面全打开的 `wide` 拿到最高 K2（13 / 26），而它的**取值种数只有 7 种**，是四个画像里最少的。可读项数与取值多样性是两个方向相反的数，把其中一个当另一个报，是这一类台账最常见的"进步"。
*台账写法*：覆盖率只报 K2（分母钉死 26，永不裁剪——第廿六章之九第 4 条），另起一列报取值种数；任何"覆盖率提升"的陈述必须同时给这两列，且分母相同。

十二条之外还有一句要交代的边界：**本报告里另有十二条禁推句式没有进本节**——它们全部写在两个脚本每次运行的输出末尾（脚本 1 六行、脚本 2 六行，`#` 开头）。差别不是重要性，而是强制方式：本节的十二条靠读者自查，那十二条靠输出自带，复制样例时它们一起被复制。第廿八章之五那 54 行的最后六行就是这一组，其中"漂移既不是攻击信号也不是加固信号"与"两次快照不构成趋势"两条如果放进本节会与前几条重复，所以留在输出里。
### 廿九之七 功能反证覆盖：分母为什么是 13 而不是 8，分子为什么是 0

这一节要还两笔债，两笔都在正文里点名点姓地欠下：

- 第十九章之十末尾（谈 atomisp 那段）写的："**第廿六章的'行为反证可用率'必须按对象分档，不能整域给一个布尔**"。
- 第廿一章 FT5 台账字段那行写的："`detectable_by_feature_probe`（yes / no）；**采集侧对应第廿九章的'功能反证覆盖'指标**"。

两处说的是同一件事，名字不同，所以本节先做一次命名收敛，再给定义与实测。

**命名收敛（第廿六章之九第 5 条的第三次应用）。** 前两次是 K5→K6、K7→K9，都换了一个 K 号；这一次不换号，而是**取消编号**：两个名字合并成"功能反证覆盖"，并且**不进 K1–K10 表**。三条理由，逐条可对：

1. 它不满足"运营指标"的采集前提——第廿七章脚本 1 没有任何 V7 姿势（实测：`good` 画像 60 条 `P-` 记录的 `face` 分布是 V1 22、V2 4、V3 3、V4 17、V5 8、V6 1、V8 5，`V7` **0 条**；九份画像合并后 `V7` 仍然 0 条）。一个分子恒为 0、且只能人工填写的量，放进 K 表会让人以为它有采集口径。
2. 它的分母不满足第廿六章之九第 4 条"分母钉死才能跨设备可比"——分母是"本报告核验过判定点的对象面"，会随源码核验新增判定点而变（补一个对象的闸门，分母就从 13 变 14），这与 K2 的 26、K6 的 7、K7 的 4 三个钉死分母性质不同。
3. 它与 K5/K6 的关系是解释性的：K5 说"多少条目没有设备侧证据"，本指标说"为什么没有"。它是 K5 的成因表，不是并列的第十一个数。

**定义（分三层，混用这三层是本指标唯一的使用风险）。**

| 层 | 名称 | 分子 | 分母 | 谁来填 | 实测 |
| --- | --- | --- | --- | --- | --- |
| 一 | 判定点覆盖率 | 有源码锚点的判定点 | 判定点全集 | 源码核验（第卅章取回记录） | **13 / 13** |
| 二 | 可观测覆盖率 | 判定结果能被既有只读姿势观察到的判定点 | 同上 13 | 脚本 1 | **0 / 13** |
| 三 | 语义上限分布 | 按"反证能推到哪一档"分列 | 同上 13 | 正文判读 | 版本档 3、能力档 2、在场档 5、无档 3 |

分母是 **13 个判定点**而不是第十九章之十二那张表的 **8 个对象**，这是本节最容易填错的一格：矩阵按对象给 `✓/△/✗`，覆盖按判定点击。Venus 有两个门（VP9、EOS）、Mali 在本树与上游各有一处探测，所以 8 个对象摊出 13 个判定点。**按对象算会得到"8 个对象里 2 个可用"，按判定点算会得到"13 个判定点里 3 个到版本档"，两个数都对，但说成同一个数就是错误。**

**十三点实测表**（`档位` 列的三档定义在下面；`上限` 列回答"这个判定的结果最多能反证到什么程度"）：

| # | 对象 | 判定点与锚点 | 档位 | 上限 |
| --- | --- | --- | --- | --- |
| 1 | Adreno GMU / a6xx SQE | `a6xx_ucode_check_version()`（`a6xx_gpu.c:712`），过低打印 `:757-759`，未知 GPU 落到 `return ret;`（`:772-773`、`:777`）拒绝 `-EPERM`（`:800-806`） | `gate_in_success_path` | **版本档**（唯一一处"出图失败 ⇒ 镜像低于闸门"，但它只检查 SQE 的一个 nibble 位段） |
| 2 | 下游 KGSL（PFP/PM4） | `get_ucode_version()` 定义在 `adreno.c:70-81`，赋值点 `:109`；装载路径**不校验**；唯一出口是 ioctl `KGSL_PROP_UCODE_VERSION`（UAPI `msm_kgsl.h:329`，实现 `adreno.c:2499-2509`），sysfs 属性表 `kgsl_pwrctrl.c:1080-1106` 无一承载版本 | `no_probe_defined` | **无**（ioctl 不是只读姿势，sysfs 上没有这一格） |
| 3 | Mali CSF（本树 `android16-6.12`） | `panthor_get_instr_features()` 第一句 `if (glb_iface->control->version < CSF_IFACE_VERSION(1, 1, 0)) return 0;`（`panthor_fw.c:878-886`） | `gate_in_success_path` | **能力档**（决定仪表功能有无，返回 0 时无任何打印） |
| 4 | Mali CSF（上游 `v7.3` 同形态） | `panthor_fw_has_glb_state()` `:326-331`（4.1.0）、`panthor_fw_has_64bit_ep_req()` `:333-338`（4.0.0） | `gate_in_success_path` | **能力档**（三处探测全部无日志，属第廿一章 FT5） |
| 5 | Exynos MFC | 无固件则引擎不工作：`s5p_mfc_ctrl.c:50-51` 的注释句与失败原句 `:75`；另有 `dev->fw_ver` 推寄存器写法（`s5p_mfc_opr_v6.c:43` 的 `IS_MFCV6_V2(dev)`） | `gate_in_success_path` | **在场档**（"能解码 ⇒ 镜像在场"；`fw_ver` 是驱动内部值，V4 数值面读不到） |
| 6 | Venus 门一（VP9） | `if (inst->core && is_fw_rev_or_newer(inst->core, 5, 4, 51))`（`vdec.c:711`，上一行注释 `/* Enabling sufficient sequence change support for VP9 */`） | `gate_in_success_path` | **版本档**，但语义是行为微调，**与安全补丁级别无关** |
| 7 | Venus 门二（EOS） | `if (IS_IRIS2(inst->core) && is_fw_rev_or_older(inst->core, 1, 0, 87))`（`vdec.c:577`） | `gate_in_success_path` | **版本档**，同上 |
| 8 | MediaTek VPU + SCP | 装载成功打印 `mtk_vpu.c:628`（`vpu is ready. Fw version %s\n`，INFO 级）；失败侧只有与通道身份无关的 `mtk_vpu.c:759` 与 `mtk_scp.c:115` | `gate_in_success_path` | **在场档**（版本只进 V6，而 V6 在样例里两侧都是 `denied_log`） |
| 9 | 相机 ISP / IPU3 | 日期名优先、`-ENOENT` 回落 `ipu3-css-fw.c:120-124`；头尺寸硬校验 `:128-134`；逐 binary 校验链 `:148-224`；版本 `dev_info` `:136-138` | `gate_in_success_path` | **在场档**（本域最长的一套校验链全部检查可用性，不检查发布版本） |
| 10 | 相机 ISP / atomisp | `sh_css_check_firmware_version()`（`sh_css_firmware.c:188-210`），警告型 `:202-209`，逐字 `/* For now, let's just accept a wrong version, even if wrong */` + `return false;`；期望值是编译期常量（`sh_css_firmware.c:59-60`） | `warning_only` | **无**（台账记 `not_inferable`） |
| 11 | 相机 ISP / FIMC-IS | 尺寸区间 `fimc-is.c:390`；setfile 两段超时 `:671-686`（`:680` 载入、`:682` 触发）；版本只打印 `:417-418` | `gate_in_success_path` | **在场档，且必须按镜像分列**（多镜像两因一果，第廿三章之八第（5）条） |
| 12 | NPU / ivpu | `ivpu_fw_parse()` 八条硬校验（`ivpu_fw.c:112-164`：`:118`、`:123`、`:133`、`:138`、`:143`、`:148`、`:155`、`:160`），每条 `ivpu_err` + `-EINVAL`；API 协商 `IVPU_FW_CHECK_API(vdev, fw_hdr, BOOT, 3)` / `JSM, 3`（`:171-174`） | `gate_in_success_path` | **在场档 + API 协商档**（校验的是 header 版本常量与地址区间，不是镜像发布版本） |
| 13 | NPU / qaic | 完整性校验开关由外设决定（`qaic_control.c:1385-1391`），主机侧在那条分支上装的是 stub；结构上无主机镜像（`qaic_drv.c:272-281`） | `no_probe_defined` | **无**（台账记 `not_inferable`） |

档位定义只有三条，判据是"这个判定在不在成功路径上"与"它失败时阻不阻断"：`gate_in_success_path` = 装载/初始化必经且失败会改变可见行为（10 点：#1、3、4、5、6、7、8、9、11、12）；`warning_only` = 必经但不阻断（1 点：#10）；`no_probe_defined` = 装载路径上没有这个判定，或判定条件在对面（2 点：#2、13）。

三点必须从这张表读出来。

**第一，第三层的"版本档 3 / 13"落到对象层是 2 / 8**（GMU 与 Venus），这与第十九章之十二那张矩阵里 V7 列只有这两行是真 `✓` 完全对应——不是巧合，而是同一次核验的两种投影。矩阵给"这个对象能不能用行为反证"，本表给"这个判定点反证推到哪一档"。**能推到版本档的判定点，本报告只找到三处；其余十点即便走到、即便可观测，也只能得到"在场"或"能力有无"。**

**第二，两个 `not_inferable` 正是之五那个差集 `{not_inferable}` 的出处。** 第廿章之九的白名单里有 `not_inferable`，脚本 1 一次都不产它，读者会问"这个值是不是废的"。不是：#10 与 #13 的台账判定值就是它，而这两条判定**不可能由任何 adb 姿势得出**——atomisp 那一格要靠读驱动源码才知道闸门恒假，qaic 那一格要靠问外设才知道开关状态。`not_inferable` 是"给人填的、不给机器采的"那一档，之五把它单独留在白名单里正是为了这两行。

**第三，"0 / 13"是本报告给厂商的最直接一句话。** 十三点里每一点的结果都写在装载路径或内核逻辑里，但没有一点的**结果**能被现有只读姿势观察到：#1 要提交一次作业才知道，#6/#7 要放一条 VP9 码流才知道，#5/#8/#9/#11/#12 要触发一次失败才知道。这些动作全部越出第廿七章之十条声明的"脚本不改任何东西"边界。所以本指标的采集层读数只能是 0，并且**这个 0 不是失败，而是第廿六章 K3 与 K8 那条差额的行为面版本**：判定存在于内核（K3 意义上的"读数存在"），不存在于你的姿势能达到的地方（K8 意义上的"可取到"）。

台账落地（三个字段，不给 K 号）：`feature_probe_points`（整数，按对象分列，Venus 记 2、ISP 记 3、NPU 记 2）、`probe_tier`（三档之一）、`counter_evidence_ceiling`（`version` / `capability` / `presence` / `none`）。**禁止的写法只有两种**：把本指标写成"行为反证可用率 = 75%"这种整域布尔（第十九章之十明令禁止，理由就是 #10 与 #13）；或把第二层的 0 写成"行为反证不可用"（第一层的 13/13 说明判定是知道的，缺的只是出口）。

### 廿九之八 脚本 1 的验收：四条硬约束逐条核，以及九份画像才够的分支覆盖

第廿章之九给脚本 1 立过四条硬约束并写明"第廿九章的分支覆盖验收逐条核"。本节就是那一次核，实测对象是抽回自第廿七章围栏区间 22–375 的那份 354 行脚本（之二的表），九份画像运行结果全部落在盘上文件里，可逐条重跑。

| # | 约束（第廿章之九原文） | 机器核验方式 | 实测结果 |
| --- | --- | --- | --- |
| 1 | 每个 `P-id` 在脚本里有且只有一个分支 | `grep -o 'rec P-[0-9][0-9]'` 取字面 pid，按 `# ---- ` 区块归属去重，统计出现在多于一个区块里的 pid | 字面 pid **24 个，每个恰好属于一个区块**，无 `MULTI`；另外 2 个（`P-19`、`P-20`）由同一个循环用 `pid=` 分派（脚本第 241–242 行），因此"26 个分支"与"25 个区块"同时成立：**区块数不是分支数**，这条要在读脚本前先说清 |
| 2 | `verdict` 只能取词汇表的值，不接受自由文本、不接受布尔 | 白名单 17 行 → `comm -3` 求差 | 九份画像的 `verdict` 去重 = **16 个**，全部在白名单内；差集恰为 `{not_inferable}`；无任何值含空格或竖线；无 `true`/`false`/`0`/`1` 形态 |
| 3 | `rule` 列必须是第廿章表"管住它的规则"里的一条 `文件:行号` | 逐 `(pid, rule)` 对做字面子串匹配 | **50 个 `(pid, rule)` 对、41 个不同锚点，OK=50 / MISS=0**；只用四个语义画像时是 42 对 / 36 锚点，同样 MISS=0 |
| 4 | `source` 只能是 `device_read` 或 `static_baseline` | 列去重计数 | 540 行记录里 `device_read` **405** + `static_baseline` **135**，无第三个值，无空值 |

四条之外，本节补两条在第廿章没有编号但同样可核的性质：

- **形态恒定**：九份画像**每一份**都是 `lines=70 / P-记录=60 / 触发的 pid=26 / rollup=3 / 注释=6`。桩画像的输入差别极大（`void` 让每条命令都空成功，`gone` 让每条都报 `No such file or directory`），输出长度一格不变。这就是第廿七章之九那句"少采一格会被填成 `-` 而不是少一行"的可执行版本：**任何人拿到 69 行或 71 行的输出，不必读内容就知道采集端被动过。**
- **错误串不规范化**：第廿章之九额外要求 P-04、P-18、P-22 三行的输出必须带实际错误串（不翻译、不改写），因为这三处厂商差异最大。九份画像里这三条 pid 共 **36 行**，其中 **35 行带原始错误串**，唯一不带的一行是 `P-18|V4|dtb.base|present|entries=0||genfs_contexts:152-154|device_read`——这一支根本没有错误发生（列成功、零条目）。所以那条约束的精确适用范围是**这三条 pid 的失败分支**，不是这三条 pid 的每一行；本报告把它按实测边界写，不写成"36 行全部带错误串"。

**分支可达：为什么四个画像不够，而必须加五份没有语义的画像。** 第廿章之九写着"从不触发的分支算脚本的缺陷，不算样例的遗漏"。把这条纪律当判据执行，就要先给"分支"下一个可数的定义：**分支 = 一个 `rec` 调用点。**脚本 1 有 **103 个** `rec` 调用点（96 个带字面 pid，7 个带 `"$pid"`）。其中带字面 pid 的 96 个点折叠成 **83 个不同的 `(pid, verdict)` 档位**。实测：

| 画像集合 | 触发的 `(pid, verdict)` 数 | 未触发的字面档位 |
| --- | --- | --- |
| 四个语义画像（`good` / `user` / `wide` / `alt`） | 57 | **26** |
| 九个画像（再加 `err` / `gone` / `deny` / `void` / `mixed`） | 97 | **0** |

只用四个语义画像验收，本报告就欠着 26 条从未触发的分支——按第廿章那条纪律，那 26 条是**报告的缺陷**。补的五份画像不声明任何真实设备语义，`err` 是"所有命令都返回一个未分类的传输错误"，`gone` 是"所有路径都不存在"，`deny` 是"所有路径都拒绝"（唯一例外：debugfs 可列但节点值被拒，这是让 P-22 走 `denied_value` 而不是 `debugfs_unreachable` 所必需的形状），`void` 是"所有命令都空成功"，`mixed` 只换标签视图（把 `/dev/video0` 标成 `device`、`/dev/accel/accel0` 标成 `accel_device`），专门用来点燃 P-19 的 `eacces_untagged` 与 P-20 的 `eacces_typed` 这两支。**它们是夹具，不是设备**：这五份画像在真机上不存在，也不该被读成"某种厂商配置"。这一点必须和第廿七章之十条第 4 条连读——画像证明的是脚本可执行，不是真机分布。

97 = 83 个字面档位 + 14 个由循环分派出来的档位（P-19 五个、P-20 六个、P-26 三个），这个式子同时说明：那 7 个 `rec "$pid"` 调用点不是"少写了 pid"，而是同一支臂服务两个姿势的合法形态。

**锚点核对的做法与它当场揪出的两处缺陷。** 第 3 条约束的核法是字面匹配，不是语义判断——先从 `.w17k.md` 里把 26 行表按 `^| \*\*P-` 抽出、以 pid 建索引，再把九份画像的 `(pid, rule)` 对逐个 `case` 匹配。命令：

```bash
grep '^| \*\*P-' ch20.md | awk '{ if (match($0,/P-[0-9][0-9]/)) print substr($0,RSTART,4)"\t"$0 }' > rows.tsv
awk -F'|' '/^P-/{print $1"|"$7}' snap-*.txt | LC_ALL=C sort -u > pairs.txt
while IFS='|' read -r pid rule; do
  row=$(awk -F'\t' -v P="$pid" '$1==P{print $2}' rows.tsv | head -1)
  case "$row" in *"$rule"*) echo "OK $pid $rule";; *) echo "MISS $pid $rule";; esac
done < pairs.txt
```

它在本篇跑出过 11 条 `MISS`，两类成因：**(a) 第廿章的锚点写成省略式**（同一格先写 `atomisp_v4l2.c:1143`，随后写"在 `:160-161`"），字面匹配必然失配——这类改正文，把 13 处省略锚点补全成 `文件:行号` 的完整形式；**(b) 脚本选错锚点**——这类改脚本，不改表。第 (b) 类有两条，都是本节真正的产出，必须写出来而不是悄悄修掉：

1. **P-04 的 `ok` 臂原来写 `file_contexts:423`。** 那一行标的是 `/vendor|system/vendor`，而 P-04 的对象是根级挂载点 `/firmware_mnt`。改成 `domain.te:616-617`——即第廿章 P-04 那一行真正列出的那条锚点。
2. **P-19 / P-20 的 `ok` 臂原来共用 `shell.te:463`。** 那条是 `allow shell ion_device:chr_file rw_file_perms;`，属于 **P-23**，而第廿章 P-23 那一行的末尾原句是"**禁止据此推断其他 `/dev/*` 也能开**"。把它抄给 P-19/P-20 就是把这条禁令反过来用，属于报告自身的缺陷。改成按 pid 分派的两个变量：`/dev/video0` 用 `file_contexts:221`（`video_device` 有标签），其余用 `file_contexts:97`（未点名落兜底 `device`）。

这两处修完之后 OK=50 / MISS=0。**修脚本而不是修表**是这里唯一的正确方向：表是判据，脚本是被判据；如果把表改成迁就脚本，第 3 条约束就自动失去意义。修复的实际后果也记在账上：`good`、`user`、`alt` 三份画像的输出逐字节不变（这三条臂在它们手里不走），`wide` 变 2 个 `rule` 格；第廿七章正文里那份 70 行 `good` 样例因此仍然有效，第廿八章样例 `good → user` 因此仍然有效——两章的围栏样例都在修复后重新 `cmp` 过（之九给结果）。

**三条限制，本节登记、第卅章沿用。**

1. **分支可达 ≠ 分支可识别。** P-26 的三个 `dumpsys` 目标里，"被拒"这一支与"其它错误"兜底支产出的记录**逐字节相同**（同 `verdict = vendor_dependent`、同 `rule = shell.te:29-219`），只有 `err` 列的原始错误串不同。验收能说"两支都走到了"，不能说"从输出上分得清是走到哪一支"。任何想在事后按记录反推分支的用法都会失败，唯一可用的区分材料是 `err` 文本。
2. **`err` 列有一处语义复用。** P-18 的 `present` 臂把目录首项写进了第 6 列（样例里是 `adsp`），那里没有错误发生。所以第 6 列的名字（`err`）与它在每一行的实际语义不完全一致。这条复用是无害的，因为脚本 2 **从不读第 6 列**：`grep -n '\$6' w17-drift.sh` 在这份 324 行脚本里只命中两行，第 22 行是它自己 `rec()` 的参数模板，第 311 行是 `$6=="mode_tightened"`，那是它自己的输出列而非输入账本的列。
3. **验收只核形式。** 之一末尾那条方法论约束在这里落地：上面所有数字（103、83、97、0、50、41、70/60/26/3/6）都是形式性质。某一格算出的 `verdict` 在真实策略集下对不对，本节一次都没证明，桩画像也证明不了。

### 廿九之九 脚本 2 的形态验收：一条 awk、十六对快照、退出码三种

脚本 2 的判据是"输出形态与契约同构"，全部能在一份输出上用一条 `awk` 核完。这条 `awk` 就是本节要交付的东西，它同时核第廿八章之二那 8 列、`did` 的 21 个块号、`face` 的 V1–V8|ALL|DECL、`verdict` 的 35 值白名单、以及"54 行恒定"这一条。词表文件 `v35.txt` 由第廿八章之七那张四档表逐行抄出（4 前提 + 1 级别 + 2 指标 + 27 登记 + `stable` = 35）。

```awk
# form-check.awk  用法: awk -v VOCAB=v35.txt -f form-check.awk dr-*.txt
BEGIN { FS = "|"; while ((getline w < VOCAB) > 0) ok[w] = 1 }
NR == 1 { if ($0 != "did|face|key|before|after|verdict|rule|note") e = e " HDR"; next }
/^# /   { note++; next }
$1 == "K6" || $1 == "K7" || $1 == "K8" || $1 == "L" || $1 == "S" { roll++; next }
/^D-/ {
  d++; blocks[$1] = 1
  if (NF != 8)                                 e = e " NF@" NR
  if ($1 !~ /^D-([01][0-9]|2[01])(b|c)?$/)    e = e " DID:" $1
  if ($2 !~ /^(V[1-8]|ALL|DECL)$/)            e = e " FACE:" $2
  if (!($6 in ok))                             e = e " VERD:" $6
}
END { nb = 0; for (b in blocks) nb++
      printf "lines=%d d=%d blocks=%d roll=%d note=%d errors=%s\n", NR, d, nb, roll, note, (e == "" ? "NONE" : e) }
```

四个设计选择要在读它之前定下来，否则这条 `awk` 会给出假绿：**表头按整行字面比**（不按列数比，因为 rollup 行的列数本来就少）；**注释行按 `^#` 排除**（它们只有 1 列）；**rollup 按首列枚举而不是按 `$1 ~ /^S/`**（`K6`/`K7`/`K8`/`L`/`S` 五种首列，第廿八章之四钉过）；`D-` 行才核 `NF==8`，其余形态不强求。

实测：第廿八章那份 54 行样例，以及在第廿七章脚本 1 修复之后重新生成的**全部十六对**快照（四个语义画像两两有序配对，含四对自比对），每一条都是 `lines=54 d=42 blocks=21 roll=5 note=6 errors=NONE`。逐字节回归结果：

| 快照对 | 与修复前的输出 | `rc` |
| --- | --- | --- |
| `good → good`、`good → user`、`good → wide`、`good → alt`、`user → good`、`user → wide`、`alt → good`（第廿八章十对里的七对） | 逐字节相同 | 0 |
| `wide → good`、`wide → user`、`wide → wide` | **仅 `D-11` 的 `P-19 /dev/video0` 一行的 `rule` 格不同**（`shell.te:463` → `file_contexts:221`） | 0 |
| `user → user`、`user → alt`、`wide → alt`、`alt → user`、`alt → wide`、`alt → alt` | 第廿八章那十对里没有这几对，本节首次跑出，形态同样 54 行 | 0 |

这张表就是之八那处锚点修复的全部下游影响面：**第廿八章用过的十条比对里只有三条受影响，每条只变一格，且变后的值正是第廿章 P-19 那一行列出的锚点。**之所以只有 `wide` 作为前一次快照时才浮现，是因为 `D-11` 的 `cause_migration` 臂取 `rulev`（账本第 7 列），而 `stable` 臂取脚本内硬编码的 `shell.te:424`——只有 `wide` 侧能列出标签，才会走到前者。第廿八章正文里那份 `good → user` 样例因此不需要重做。

词汇级覆盖沿用第廿八章之九的结果（35 个值全部被触发，最小覆盖集 6 对：`good-user`、`user-good`、`good-wide`、`wide-good`、`good-alt`、`wide-wide`），本节不重新推导，只补一条与脚本 1 对称的性质：**55 个 `rec` 调用点**（`grep -o '\brec '` 得 56，减去第 20 行错误消息里那个不是调用点的 `rec`）与 42 条输出记录之差是刻意的——一个块内多条臂共用同一行输出位置，可达点数大于输出行数。这和脚本 1 的 103 个点 / 60 行是同一种形状，读的时候不能拿输出行数当分支数。

退出码三条都实测过：正常比对 `rc=0`（十六对全是）；入参错误 `rc=2`，三种形态各自触发——无参数、只给一个、给一个不存在的文件；`rc=9` 是 `rec()` 的元数守卫（`[ "$#" -ne 8 ]`），本报告在它还在的时候踩通过一次，输出是 `BUG: rec arity 7! ...` 加零字节 stdout。这条 9 号退出值得单独保留，因为它防的事故类型与 54 行恒定正好互补：**元数错误会让一列错位而不改变行数**，只看行数的验收发现不了，只有 `exit 9` 能。

### 廿九之十 验收之后仍然不能断言的东西（交给第卅章）

一张表只列"本报告拿不出证据、因此读者也不能反过来用"的条目。前六条是第廿八章之十登记过的缺口，本节按验收结果把它们和验收新增的条目并成一份清单，第卅章逐条落进边界清单，不在正文里再展开。

| # | 不能断言的东西 | 为什么不能 | 处置 |
| --- | --- | --- | --- |
| 1 | "这台设备的 `driver_override` 当前是 X" | 脚本 1 不采它，脚本 2 的 `D-19` 只能输出 `companion_missing`（第廿一章 FT6 要求名字读数必须有伴读） | 任何 rpmsg 名字读数不进 K8 分子 |
| 2 | "遍历 `/sys/class/remoteproc` 就得到全部协处理器" | 承诺过的 `no_rproc_face` 三类枚举（第十七章之十）没实现；补姿势要动 K2 的固定分母 26 | 转第卅章缺口，脚本只留 `D-19` 那句"要补采先写进第廿章的表" |
| 3 | glink 边数与 `rpmsg_ctrl` 节点数的比较结论 | 第十八章之六引用的 `qcom_glink_native.c` 一度因取错目录（`drivers/soc/qcom/`）被标成"来源待重取"，本轮在 `drivers/rpmsg/` 取回（第卅章 **D-13**）——来源已补，缺的是格子 | 结论从"形态描述"升级为可复核；脚本 2 仍不产出行，比较结论不进任何指标分母，转第廿八章之十的缺口清单 |
| 4 | "值文本参与了判定" | 脚本 1 的判定只用列存在性、模式位与返回码，`san()` 还会把竖线/换行/制表符压成空格；`P-22` 那行给出文件名但**不解析其含义** | 值文本一律只作证据附件，不作判定输入 |
| 5 | "K6 的分子能从脚本输出直接算" | 脚本 1 没有"链"这一列，26 个 pid 到七条链的映射是正文里的判断，不在输出里 | K6 手填，与功能反证覆盖同一处置 |
| 6 | "K8 的分子能从脚本输出直接算" | 输出没有 `value_source` 列（`fw_reported` / `driver_set` / `userspace_override` 三值需要源码判断） | K8 手填；`value_source` 只能由第卅章取回记录支撑 |
| 7 | 任何 V7（行为反证）方向的设备侧结论 | 脚本 1 输出 0 条 V7 记录（九份画像并集亦然），可观测覆盖率 0 / 13 | 见之七；厂商主张按 K3/K8 差额那句话提 |
| 8 | "分支可达即分支可识别" | P-26 两支产出逐字节相同的记录，只能靠 `err` 文本区分 | 见之八限制 1；不允许用记录反推分支 |
| 9 | "`err` 列每一行都是错误串" | P-18 的 `present` 臂复用第 6 列放目录首项；九画像 36 行里 1 行为空 | 见之八限制 2；引用该列时按分支解释 |
| 10 | 真机上的取值分布、错误形态分布与"多数设备会怎样" | 全部实测对 AOSP 策略文件与内核源码做，设备侧只对桩画像做；`command not found` 这类第三形态未观测 | 第卅章按"未声明"处理，不写推测 |
| 11 | "第廿七章与第廿八章的判定在语义上正确" | 验收只核形式（之一末条方法论约束，之八第 3 条限制） | 语义正确性在本报告无判据，不声明 |
| 12 | `dmesg_restrict` / `kptr_restrict` 的当前取值 | `genfs_contexts:53`、`:57` 只给出类型 `proc_security`，取值由厂商 init 决定，本次未取回 | 第卅章边界项，P-24 的 `denied_log` 不解释成"被关" |
| 13 | "功能反证覆盖是第十一个运营指标" | 它不占 K 号（之七三条理由：分子恒 0、分母不钉死、性质是 K5 的成因表） | 作为台账三字段交付，不进 K 表 |

### 廿九之十一 本章结论

1. **两个脚本的契约全部按原文核过，四条硬约束无一放宽。** 第廿章之九那四条的实测形态是：24 个字面 pid 各属一个区块 + 2 个由循环分派、16 个 `verdict` 全部在白名单内且差集恰为 `{not_inferable}`、50 个 `(pid, rule)` 对全部能在第廿章那一行里字面找到、`source` 两值 405 + 135 无第三值。验收过程中没有一次是"改判据去就脚本"。
2. **本章改了两处脚本、补全了 13 处正文锚点。** 脚本那两处（P-04 的 `file_contexts:423` → `domain.te:616-617`；P-19/P-20 的 `shell.te:463` → 按 pid 分派 `file_contexts:221` / `file_contexts:97`）是验收当场发现的缺陷，其中第二处违反的是第廿章 P-23 那行自己写的禁令。下游影响面被完整量过：第廿八章用过的十条比对里三条受影响、每条只变一格、第廿七章与第廿八章的正文样例都因此不需要重做。
3. **分支覆盖的判据要把"分支"定义成 `rec` 调用点，并把画像集合扩到够用为止。** 103 个调用点、83 个字面档位：四个语义画像只覆盖 57 个，剩 26 个由五份**不声明语义的夹具画像**补齐到 0 未触发。这个数量差就是"按语义画像验收"与"按纪律验收"之间的距离，也是本篇最容易被跳过的一步。
4. **两个脚本的形态不变量是它们唯一能被读者自证的性质**：脚本 1 恒 70 行（60 + 3 + 1 + 6），脚本 2 恒 54 行（42 + 5 + 1 + 6），退出码 0 / 2 / 9 三条全实测。之九那条 `awk` 与之二那七行抽回命令，就是本报告留给读者的全部验收工具——**它们核的是形式，读者拿它们也只会核到形式。**
5. **一处命名收敛、一个不占号的指标、一份十三条的不可断言清单。** "行为反证可用率"（第十九章之十）与"功能反证覆盖"（第廿一章 FT5）合并成后者，明确不进 K1–K10；它的三层读数 13 / 13、0 / 13、版本档 3，把"我们不知道"与"我们知道但没有出口"这两件事第一次分成两列写。第卅章拿过去的不是成功案例，而是之十那张十三条的表——一份只写成功案例的验收报告和一份没有验收的报告，缺陷是同一种。
## 卅、取回记录：每一个行号的出处、每一次拒因、每一条不能断言的边界

前二十九章的每一条结论都带一个源码行号或一句文档原句。本章把这些东西的**出处**全部倒出来：哪个仓库、哪个分支、多少行、第几行写着什么、用什么命令数的、命中几个；然后把取不到的、被本轮推翻的、以及"虽然有出处但仍然推不出结论"的三类分开登记。本章不产生新结论，也**不引入任何新的源码行号**——出现的每一个行号都是前文已经引用过的那一个。

判据只有一条：**报告里每一个源码行号与文档句子都必须能在本章的取回记录里找到原句；取不到原句即为伪造。**这条判据对报告自己同样严格，所以本章除了登记表，还有一份"本轮修订掉的引用缺陷清单"（卅之十七），里面全是本报告自己写错过又改掉的行。

### 卅之一 六级命中口径，与"逐字"这个词在本报告里的准确含义

一条引用被判定为"有出处"，按下面六层顺序试，命中即停。六层的**名字**就是本报告用词的依据：只有第 1 层与第 2 层可以称"逐字命中"，其余四层必须另行说明形态，第 6 层一律算伪造。

| 层 | 名称 | 判据 | 本轮引用块行的命中数 |
| --- | --- | --- | --- |
| 1 | 原行命中 | 该字符串是某个本地副本文件的**一整行**（去首尾空白后逐字符相等） | 691 |
| 2 | 页面子串命中 | 该字符串是某个公告/厂商页面压平文本里的**连续子串** | 21 |
| 3 | 同文件连续行命中 | 该字符串是同一文件内**相邻数行**按单空格拼接的结果 | 14 |
| 4 | 注释重排命中 | 第 3 层再减去 C 注释的 `/*`、` * `、`*/` 前缀后的结果 | 4 |
| 5 | 分段全命中 | 被 `` → ``、`` … ``、`` + ``、``、``、``　``、`` / ``、` — `、`…`、`...` 切开后**每一段**都落在第 1–4 层 | 21 |
| 6 | 未命中 | 任一段落不进上面五层 | **0** |

**受这一层判据约束的只有引用块行**（正文里以 `> ` 起头的行）。统计口径：本轮把二十份章节草稿压成一个 5,884 行的语料，取出引用块行 **751** 行（空引用行 0），按上表分层得 **691 + 21 + 14 + 4 + 21 = 751，未命中 0**。

正文里用反引号包起来的内联片段是第二类对象：它们多数是标识符、路径、数字或简写，**不享受也不主张"逐字"待遇**。本报告仍对它们做了第二道扫描，口径与结果都写在这里，避免读者把内联片段当原句引用：

- 全稿内联片段（去重后，条件：长度 ≥ 16、含空格、纯 ASCII 可打印）共 **948** 条，能在四层草料堆（逐行 / 同文件拼接 / 页面压平（两套口径）/ 注释重排拼接）里直接命中的 **663** 条，其余 **285** 条全部落入可解释形态（见卅之十三的分类表）。
- 更严格的一道：只取**同一行里出现"逐字""原样""原句"三个词之一**的那些行上的内联片段，去重得 **133** 条，批量命中 **89** 条；对余下 44 条逐条单独复测（批量 `grep -F -o -f` 有"最长匹配吞掉短模式"的假阴性，复测后其中 **10** 条其实命中），最终 **34** 条不落在第 1/2 层。这 34 条**没有一条是"引用了不存在的原句"**，它们的实际形态是：把 C 字符串字面量的首尾引号一起抄进跨度、去掉引号即命中（6 条）、句中用 `...` 或 `X` 占位（8 条）、表格单元格用 `/` 拼接（4 条）、本报告脚本自己的命令与输出（10 条）、目录列举与文件名清单（2 条）、行定位符用 `/` 拼接（1 条）、引用的是 HTML 标记本身而压平时标签已被删掉（2 条）、注释重排后仍带 `/* … */` 外壳（1 条）。逐条出处见卅之十三。

**这一节的诚实声明**：第 3、4、5 层的命中不是"逐字"，是"可复原"。第 4 层要求读者接受"注释里的 ` * ` 前缀不算正文"；第 5 层要求接受"报告用连接符把两个原行缝在一起"。本报告在正文里对被缝过的引用一律写出连接符，不伪装成单行原句。

### 卅之二 语料台账：取回方式、目录结构与总量

**仓库与分支**（全部为实际取回的副本，本章所有行号即这些本地副本的行号）：

| 仓库 | ref | 用途 | 本地副本目录 |
| --- | --- | --- | --- |
| `kernel/common` | `android15-6.6` | GKI 主线，第 1–25 章绝大多数行号 | `.src17/kernel/common/android15-6.6/` |
| `kernel/common` | `android16-6.12` | 分支对照；框架面与目录列举复核 | `.src17/kernel/common/android16-6.12/` |
| `torvalds/linux` | `master` | 上游对照（文档绑定与 accel 目录） | `.src17/upstream/torvalds-master/` |
| `torvalds/linux` | `v7.3`（标签） | 第廿一章引用的上游形态单列 | `.src17/upstream/torvalds-v7.3/` |
| `clo/la/kernel/msm-5.4` | CodeLinaro 镜像 | KGSL 厂商侧属性表与 `msm_kgsl.h` | `.src17/clo/la/kernel/msm-5.4/msm-5.4/` |
| `platform/system/sepolicy` | `main` | 策略门（`file_contexts`、`genfs_contexts`、`shell.te`、`domain.te`、`service.te`、`service_contexts`、`te_macros`） | `.src17/platform/system/sepolicy/main/` |

**取回方式**（源码副本唯一一条，全部同法）：Gitiles 的 `?format=TEXT` 取 base64 原文，本地 `base64 -d` 还原；经代理 `http://127.0.0.1:10808`；目录列举取 Gitiles 的树页面。文档页（`source.android.com`、`developer.android.com`、厂商安全页）用 `curl` 原样存 HTML。

**压平命令**（本报告唯一承认的口径，正文与审计都用它；逐页一份文件）：

```
sed 's/<[^>]*>//g' 页.html \
 | tr '\n' ' ' \
 | sed -e 's/&nbsp;/ /g' -e 's/&lt;/</g' -e 's/&gt;/>/g' -e 's/&quot;/"/g' \
       -e "s/&#39;/'/g" -e 's/\xe2\x80\x8a/ /g' -e 's/\xe2\x80\x89/ /g' \
       -e 's/\xc2\xa0/ /g' -e 's/&amp;/\&/g' \
 | tr -s ' \t' ' '
```

四步都不可省，且顺序固定：**删标签**（不是把标签换成空格）、**折行**、**解实体**（`&amp;` 必须排在最后一条，否则 `&amp;lt;` 会被解两遍）、**归一半角/窄空格并压缩空白**。后两步各管一类假失配：HTML 源里 `device`／`partner` 之间是 U+200A hair space 而不是普通空格，不解就命中不上；标签若换成空格，`vulnerability , severity ,` 这类"文本紧跟内联标签"的位置会凭空多出一个空格，凡是照页面原样写的句子都判成不命中。

本轮把这两步做反过一次，代价写在卅之十七第 14 条：按这一节**旧版写下的三步命令**（标签→空格、折行、压缩，既不解实体也不归一）重跑引文审计，第二级从 21 掉到 **14**、第六级从 0 涨到 **7**；只补上解实体仍换成空格，得 16 / 5；删标签但完全不归一，得 18 / 3；删标签且只解实体、不做空格归一，得 19 / 2。四组读数与采用口径的 21 / 0 都在卅之十三审计二的反证表里，每行都标了压平件字节数，逐行可复现。旧口径写进过这一节的台账，所以那一版的报告自己无法复现自己给出的计数——这是本章要抓的头号缺陷形态。

厂商页的表格还有一个副作用：删标签会把相邻单元格直接粘住（`Our Reference` 与 `PSP-19` 之间没有空格），所以凡跨单元格拼接的引文在正文里一律写 `/` 连接符，并在卅之十三的跨度审计里用第二套"标签→空格"压平件做补充草料堆（两套都在，命中口径写清用的哪一套）。


**总量**（本轮重新实测，不是沿用上一轮）：

| 量 | 值 | 命令口径 |
| --- | --- | --- |
| 本地副本文件数 | **178** | `find .src17 -type f \| wc -l` |
| 总字节 | **4,217,431** | `find .src17 -type f -printf '%s\n' \| awk '{s+=$1}END{print s}'` |
| 行索引条目数 | **146,094** | `wc -l < .x17-index.usv`（每条 = 文件名 `\x1f` 行号 `\x1f` 原文） |
| 不同文件名 | **134** | 上式对第一列 `sort -u` |
| 出现在两份以上副本里的同名文件 | **36** 个名字，覆盖 **80** 个文件 | `find .src17 -type f -printf '%f\n' \| sort \| uniq -c \| awk '$1>1'` |
| 只有一份副本的文件名 | **98** | 同上取 `$1==1` |
| 分桶 | android15-6.6 **86**、android16-6.12 **67**、sepolicy **15**、CodeLinaro **6**、torvalds-v7.3 **2**、torvalds-master **2** | 路径第三、四级归并 |
| 文档页副本 | `.sb*.html` **35** 份，原始 HTML **6,946,603** 字节 | 含本轮补取的 5 页厂商文档、Vulkan 实现指南与 6 期 Pixel 公告 |
| 文档页压平（正文口径） | **3,011,473** 字节（35 份逐页文件相加） | 上一条命令口径；35 份一次性整体压平得 **3,011,441**，差的 32 字节全在页边界空白上 |
| 文档页压平（备用口径） | **3,023,042** 字节 | 标签换成空格，仅供卅之十三跨度审计的第二草料堆使用 |
| 章节草稿语料 | **6,804** 行（21 份，一～卅章） | `cat .w17?.md` |
| 审计语料 | **5,884** 行（20 份，一～廿九章） | 引文审计与跨度审计都用这一份：第卅章自身不含 `> ` 引用行，也不作为被审对象 |

**同名文件不区分仓库**是这套台账的固有缺陷，必须写在这里：索引第一列只有文件名，没有路径，所以 `remoteproc_sysfs.c` 的两份副本（6.6 与 6.12）在索引里是同名不同行号的两组条目。本报告的处理方式见卅之十三第 3 条（"分支敏感行号表"）：凡是同名多副本且被引用过的，逐个用**内容**而不是行号确认归属。

### 卅之三 D-1 相机 ISP 侧

**Intel IPU3（`drivers/staging/media/ipu3/`）**

- `ipu3-css-fw.h:9-11`：`#define IMGU_FW_NAME\t\t"intel/ipu3-fw.bin"` / `#define IMGU_FW_NAME_20161208 \` + `"intel/irci_irci_ecr-master_20161208_0213_20170112_1500.bin"`
- `ipu3-css-fw.c:120-124`：先请求带日期的长名，`if (r == -ENOENT)` 才回落到 `IMGU_FW_NAME`
- `ipu3-css-fw.c:128-134`：`css->fwp->file_header.h_size != sizeof(struct imgu_fw_bi_file_h)` → `goto bad_fw`
- `ipu3-css-fw.c:136-138`：`dev_info(dev, "loaded firmware version %.64s, %u binaries, %zu bytes\n", ...)`
- `ipu3-css-fw.c:148-224`：逐 binary 的 `prog_name_offset`、`strnlen`、`size == text+icache+data+padding`、`offset+size > fw->size`、`iomem_length`、`IMGU_ABI_FRAME_FORMAT_NUM`、`BLOCK_MAX` 校验，任一不成立即 `goto bad_fw`
- `ipu3.h:16`：`#define IMGU_NAME\t\t\t"ipu3-imgu"`

**Intel Atom ISP（`drivers/staging/media/atomisp/pci/`）**

- `atomisp_v4l2.c:58-60`：`static uint skip_fwload;` `module_param(skip_fwload, uint, 0644);` `MODULE_PARM_DESC(skip_fwload, "Skip atomisp firmware load");`
- `atomisp_v4l2.c:77-79`：`static char firmware_name[256];` `module_param_string(firmware_name, firmware_name, sizeof(firmware_name), 0);` `MODULE_PARM_DESC(firmware_name, "Firmware file name. Allows overriding the default firmware name.");`
- `atomisp_v4l2.c:1128-1171`：`atomisp_load_firmware()`；按 `isp->media_dev.hw_revision` 选名 `"shisp_2401a0_v21.bin"`（`:1143`）、`"shisp_2401a0_legacy_v21.bin"`（`:1148`）、`"shisp_2400b0_v21.bin"`（`:1153`）；无匹配时 `dev_err(isp->dev, "Unsupported hw_revision 0x%x\n", ...)`（`:1157`）并返回 NULL；`request_firmware(&fw, fw_path, isp->dev)`（`:1162`）
- `atomisp_v4l2.c:1606`：`.name = "atomisp-isp2",`；`:1391`：`isp->firmware = atomisp_load_firmware(isp);`；`:1517`：`atomisp_drvfs_init(isp);`
- `sh_css_firmware.c:59-60`：`static const char *release_version_2401 = STR(irci_stable_candrpv_0415_20150521_0458);` / `..._2400 = STR(irci_stable_candrpv_0415_20150423_1753);`
- `sh_css_firmware.c:188-210`：`sh_css_check_firmware_version()`；`:202-206` 两条 `dev_err`；`:208` 注释 `/* For now, let's just accept a wrong version, even if wrong */`；`:209` `return false;`
- `sh_css_firmware.c:71`：`char *sh_css_get_fw_version(void)`；`:247-248`：`strscpy(FW_rel_ver_name, file_header->version, ...)`；`:255-259`：`IA_CSS_ERROR(...)` / `IA_CSS_LOG("successfully load firmware version %s", release_version)`
- `sh_css_version.c:24-38`：`ia_css_get_version()` 把两个 `ISP24xx_CSS_VERSION_STRING` 与 `sh_css_get_fw_version()` 拼成一个字符串
- `ia_css_version_data.h:26-27`：两条常量原句（`REL:20150521_21.4_0539; API:2.1.15.3; ...` 与 `REL:20150911_37.5_1652; API:2.1.20.9; ...`）
- `atomisp_drvfs.c:46-49`：`OPTION_BIN_LIST BIT(0)` / `OPTION_BIN_RUN BIT(1)` / `OPTION_VALID (OPTION_BIN_LIST | OPTION_BIN_RUN)`
- `atomisp_drvfs.c:56-89`：`iunit_dump_dbgopt()`；`:62-63` BIN_LIST 分支；`:71-80` BIN_RUN 需要 `isp->asd.streaming`，否则 `ret = -EPERM`
- `atomisp_drvfs.c:160-161`：`__ATTR(dbgfun, 0644, ...)`、`__ATTR(dbgopt, 0644, ...)`；`:164-172`/`:182-197`：`driver_create_file(drv, &iunit_drvfs_attrs[i])` 与 `atomisp_drvfs_init()` 取 `isp->dev->driver`
- 组合读数：以上两属性落在 `/sys/bus/pci/drivers/atomisp-isp2/{dbgfun,dbgopt}`

**Samsung Exynos FIMC-IS（`drivers/media/platform/samsung/exynos4-is/`）**

- `fimc-is.h:32`：`#define FIMC_IS_FW_FILENAME\t\t"exynos4_fimc_is_fw.bin"`；`:33`：`#define FIMC_IS_SETFILE_6A3\t\t"exynos4_s5k6a3_setfile.bin"`
- `fimc-is.h:47-51`：`FIMC_IS_FW_INFO_LEN 31`、`FIMC_IS_FW_VER_LEN 7`、`FIMC_IS_FW_DESC_LEN`、`FIMC_IS_SETFILE_INFO_LEN 39`
- `fimc-is.h:60-61`：`FIMC_IS_FW_SIZE_MAX (SZ_4M)`、`FIMC_IS_FW_SIZE_MIN (SZ_32K)`
- `fimc-is.h:180-182`：`char info[...]`、`char version[...]`、`char setfile_info[...]` 三个字段
- `fimc-is.c:231-257`：`fimc_is_load_setfile()`；`:237` `request_firmware(&fw, file_name, &is->pdev->dev);`；`:239` `"firmware request failed (%d)\n"`；`:250-253` 从固件尾部取 39 字节 setfile_info
- `fimc-is.c:390-391`：尺寸区间闸门与 `"wrong firmware size: %zu\n"`
- `fimc-is.c:403-404`：`memcpy(is->memory.vaddr, fw->data, fw->size);` / `wmb();`；`:407-413`：从尾部倒读 info（31 字节）与 version（7 字节）
- `fimc-is.c:415-419`：`is->fw.state = 1;` 与 `dev_info(dev, "loaded firmware: %s, rev. %s\n", ...)`
- `fimc-is.c:437-441`：`request_firmware_nowait(THIS_MODULE, FW_ACTION_UEVENT, fw_name, ...)`；`:852`：`fimc_is_request_firmware(is, FIMC_IS_FW_FILENAME);`
- `fimc-is.c:671-686`：`IS_ST_SETFILE_LOADED` 两段等待、`"get setfile address timed out\n"` / `"loading setfile timed out\n"`、`:680` 载入、`:682` `fimc_is_hw_load_setfile(is)`；`:326`：`FIMC_IS_FW_LOAD_TIMEOUT`

**负向读数（目录列举与计数，第 1–2 层之外的"没有那行"类证据）**

- `drivers/media/platform/qcom/camss/camss.c`（1823 行）检索 `firmware`：**0 命中**
- `drivers/media/platform/samsung/exynos-gsc/gsc-core.c`（1324 行）与 `gsc-core.h`（521 行）检索 `firmware`：**0 命中**
- `drivers/media/platform/mediatek/` 目录列举：`Kconfig Makefile jpeg mdp mdp3 vcodec vpu`；`android16-6.12` 同一份清单 ⇒ **树内没有 ISP 驱动目录**
- `drivers/remoteproc/qcom_common.c`（523 行）检索 `DEVICE_ATTR|driver_create_file|device_create_file`：**0 命中**（SSR 名在内核内以 notifier 形式存在：`:399 qcom_register_ssr_notifier()`）

### 卅之四 D-2 媒体 DSP 补充：联发科 VPU 的双镜像

- `drivers/media/platform/mediatek/vpu/mtk_vpu.c:47-51`：`/* binary firmware name */`、`VPU_P_FW "vpu_p.bin"`、`VPU_D_FW "vpu_d.bin"`、`VPU_P_FW_NEW "mediatek/mt8173/vpu_p.bin"`、`VPU_D_FW_NEW "mediatek/mt8173/vpu_d.bin"`
- `mtk_vpu.c:138` / `:147`：文档行 `@fw_ver:\t\tVPU firmware version` 与字段 `char fw_ver[VPU_FW_VER_LEN];`
- `mtk_vpu.c:512` / `:517`：先 `request_firmware(&vpu_fw, fw_new_name, ...)` 再回落旧名
- `mtk_vpu.c:628`：`dev_info(dev, "vpu is ready. Fw version %s\n", run->fw_ver);`；`:644`：`strscpy(vpu->run.fw_ver, run->fw_ver, sizeof(vpu->run.fw_ver));`；`:689`：看门狗打印里带同一个 `vpu->run.fw_ver`
- `drivers/remoteproc/mtk_scp.c:846`：`const char *fw_name = "scp.img";`；`:849`：`rproc_of_parse_firmware(dev, 0, &fw_name);`；`:853`：`devm_rproc_alloc(dev, np->name, &scp_ops, fw_name, sizeof(*scp));`
- `drivers/remoteproc/remoteproc_core.c:998-1006`：`rproc_of_parse_firmware()` 实读 DT 属性 `"firmware-name"`（`:1002`），失败原样返回（`:1004` `return ret ? ret : 0;`），`:1006 EXPORT_SYMBOL(rproc_of_parse_firmware);`
- 设备树侧本轮补取（D-14）：`arch/arm64/boot/dts/mediatek/mt8183.dtsi` 2,238 行，`:1044` 是 `memory-region = <&scp_mem_reserved>;`，所在节点 `scp: scp@10500000` 起于 `:1036`。

### 卅之五 D-3 NPU / APU 侧

**框架层：accel 类（`drivers/accel/drm_accel.c`，227 行）**

- `:31`：`return kasprintf(GFP_KERNEL, "accel/%s", dev_name(dev));`；`:36`：`accel_class = class_create("accel");`
- `:117-119`：`kdev->devt = MKDEV(ACCEL_MAJOR, index);` / `kdev->class = accel_class;` / `kdev->type = &accel_sysfs_device_minor;`
- `:215`：`accel_debugfs_root = debugfs_create_dir("accel", NULL);`；`:217`：`register_chrdev(ACCEL_MAJOR, "accel", &accel_stub_fops);`
- 全文检索 `firmware`：**0 命中** ⇒ accel 框架不参与固件装载（第 25 章表 A 的空格据此填 `no_firmware_path`）

**Intel VPU / NPU（`drivers/accel/ivpu/`）**

- `ivpu_drv.c:39-56`：`dbg_mask`、`test_mode`（`module_param_named_unsafe`）、`pll_min_ratio`、`pll_max_ratio`、`disable_mmu_cont_pages`，全部 `0644`；`:645 MODULE_DEVICE_TABLE(pci, ivpu_pci_ids);`
- `ivpu_fw.c:42-44`：`static char *ivpu_firmware;` / `module_param_named_unsafe(firmware, ivpu_firmware, charp, 0644);` / `MODULE_PARM_DESC(firmware, "VPU firmware binary in /lib/firmware/..");`
- `ivpu_fw.c:46-56`：`:46` 是 `/* TODO: Remove mtl_vpu.bin from names after transition to generation based FW names */`；`:50` 是容器声明 `} fw_names[] = {`；`:51-55` 依次是 `{ IVPU_HW_37XX, "vpu_37xx.bin" },`、`{ IVPU_HW_37XX, "mtl_vpu.bin" },`、`{ IVPU_HW_37XX, "intel/vpu/vpu_37xx_v0.0.bin" },`、`{ IVPU_HW_40XX, "vpu_40xx.bin" },`、`{ IVPU_HW_40XX, "intel/vpu/vpu_40xx_v0.0.bin" },`
- `ivpu_fw.c:58-60`：`/* Production fw_names from the table above */` 与两条 `MODULE_FIRMWARE`（`:59-60`）——**这是本篇全部四个 NPU/ISP 候选目录里唯一有 `MODULE_FIRMWARE` 的一处**
- `ivpu_fw.c:62-87`：`ivpu_fw_request()`；`:67-71` 模块参数优先；`:74-83` 按 `ivpu_hw_gen(vdev)` 过滤后逐个 `firmware_request_nowarn`（`:78`）；`:85` `ivpu_err(vdev, "Failed to request firmware: %d\n", ret);`
- `ivpu_fw.c:36-40`：`#define IVPU_FW_CHECK_API(vdev, fw_hdr, name, min_major)`；`:89-110`：`ivpu_fw_check_api()`，`:94-95` 拆 major/minor，`:97-101` 主版本不足 → `ivpu_err(... "Incompatible FW %s API version: %d.%d, required %d.0 or later\n" ...)` + `-EINVAL`，`:102-105` 仅次版本不同只 `ivpu_warn`
- `ivpu_fw.c:112-164`：`ivpu_fw_parse()` 的硬校验链：文件过小（`:118`）、`header_version != VPU_FW_HEADER_VERSION`（`:123`）、runtime 地址区间（`:133`）、`runtime_size < fw->file->size`（`:138`）、镜像越界（`:143`）、加载区间（`:148`）、`shave_nn_fw_size > FW_SHAVE_NN_MAX_SIZE`（`:155`）、`entry_point` 区间（`:160`），每条 `ivpu_err` + `-EINVAL`
- `ivpu_fw.c:168-169`：`ivpu_info(vdev, "Firmware: %s, version: %s", fw->name, (const char *)fw_hdr + VPU_FW_HEADER_SIZE);`；`:171-174`：`BOOT, 3` 与 `JSM, 3` 两条 `IVPU_FW_CHECK_API` 闸门
- `vpu_boot_api.h:10-39`：`* =========== FW API version information beginning ================` 起的构造说明；`:47`：`#define VPU_FW_HEADER_VERSION 0x1`；`:51-62`：`struct vpu_firmware_header` 的 `header_version`、`vpu_version[VPU_FW_VERSION_SIZE]`、`firmware_version_load_address`、`firmware_version_size`、`api_version[VPU_FW_API_VER_NUM]`
- `ivpu_debugfs.c:30-36`：`fw_name_show()` → `seq_printf(s, "%s\n", vdev->fw->name);`；`:107-115`：`vdev_debugfs_list[]` 七项（`bo_list`、`fw_name`、`fw_trace_capability`、`fw_trace_config`、`last_bootmode`、`reset_counter`、`reset_pending`）
- `ivpu_debugfs.c:117-124`：`fw_log_show()`；`:144-148`：`fw_log_fops` 同时有 `.write` 与 `.read = seq_read`
- `ivpu_debugfs.c:273-293`：`ivpu_debugfs_init()` 六个节点及模式：`force_recovery 0200`、`fw_log 0644`、`fw_trace_destination_mask 0200`、`fw_trace_hw_comp_mask 0200`、`fw_trace_level 0200`、`reset_engine 0200`（**只登记存在与模式，不登记写入格式**）

**Qualcomm QAIC（`drivers/accel/qaic/`，NPU 固件不在主机侧）**

- `qaic_drv.c:35-36`：`#define CNTL_MAJOR\t\t\t5` / `#define CNTL_MINOR\t\t\t0`
- `qaic_drv.c:272-281`：那段"设备侧固件已启动"的原样注释（第十九章整段引用）
- `qaic_drv.c:296-302`：`:296` 取版本调用；`:297-301` 的闸门与 `pci_err`、`-EINVAL`、`goto close_control;`；`:302` 收尾花括号
- `qaic_drv.c:315`：`/* This is redundant since we have already observed the device crash */`；`:553-566`：`qaic_mhi_match_table[] = { { .chan = "QAIC_CONTROL", }, {}, };`，`.name = "qaic_mhi"`
- `qaic_control.c:1359-1397`：`get_cntl_version()`；`:1374-1376` 构造 `QAIC_TRANS_STATUS_FROM_USR` 查询；`:1382-1383` 取回 major/minor；`:1385-1391` CRC 位与两种生成方式分支
- `include/uapi/drm/qaic_accel.h:17`：`#define QAIC_MANAGE_MAX_MSG_LENGTH SZ_4K`；`:44-47`：四个 `QAIC_TRANS_*` 值；`:145-155`：`@major`、`@minor`、`@status_flags` 三行文档与字段；`:375`/`:384`：`DRM_QAIC_MANAGE 0x00` 与 `DRM_IOCTL_QAIC_MANAGE`（**能力事实，第 25 章表 C 只登记存在**）；`qaic_data.c:28`：`#include <uapi/drm/qaic_accel.h>`

**Qualcomm CDSP（`drivers/remoteproc/qcom_q6v5_pas.c`，1312 行）**

- `:38-39`：`const char *firmware_name;` / `const char *dtb_firmware_name;`；`:664`：`fw_name = desc->firmware_name;`；`:670-671`/`:700`：dtb 名分支
- `:681`：`rproc = rproc_alloc(&pdev->dev, pdev->name, ops, fw_name, sizeof(*adsp));`（rproc 名 = 平台设备名 = DT 节点名）
- `:214-227`：`request_firmware(&adsp->dtb_firmware, adsp->dtb_firmware_name, adsp->dev)` → `qcom_mdt_pas_init` → `qcom_mdt_load_no_init`；`:280`/`:301`：两条 `qcom_scm_pas_auth_and_reset()`
- `:781` `"adsp.mdt"`、`:791` `"adsp.mbn"`；`:895-903`：`cdsp_resource_init`（`crash_reason_smem 601`、`firmware_name "cdsp.mdt"`、`pas_id 18`、`auto_boot true`、`ssr_name "cdsp"`、`sysmon_name "cdsp"`、`ssctl_id 0x17`）
- `:941-948`、`:952-964`、`:968-979`、`:983-994`、`:1026-1028`：sdm845/sm6350/sm8150/sm8250/sm8350 五条 cdsp 条目，固件名一律 `"cdsp.mdt"`；`:419`/`:429`：`.parse_fw = qcom_register_dump_segments,`

**NPU/APU 负向与对照读数**

- `drivers/accel/` 目录列举在 `android15-6.6` 与 `android16-6.12` 完全相同：`Kconfig Makefile drm_accel.c habanalabs ivpu qaic`
- `qcom_q6v5_pas.c` 与 `qcom_common.c` 检索 `DEVICE_ATTR|driver_create_file|device_create_file`：0 命中 ⇒ 该后端无专用 sysfs 出口
- `drivers/staging/media/ipu3/`、`drivers/accel/ivpu/`、`drivers/accel/qaic/` 三处 `MODULE_FIRMWARE` 命中：仅 ivpu 两条 ⇒ atomisp / ipu3 / qaic 的固件名**不来自** `MODULE_FIRMWARE`，而来自驱动内字符串、DT，或根本没有主机侧镜像

### 卅之六 D-4：sepolicy 侧（`platform/system/sepolicy @ main`）

本地副本目录 `.src17/platform/system/sepolicy/main/`。第廿章、第廿一章、第廿五章引用过的策略行全部落在下列文件里：`private/file_contexts`（961 行）、`private/genfs_contexts`、`private/shell.te`、`private/domain.te`、`private/service_contexts`、`public/service.te`，外加 `public/te_macros` 1105 行、`public/device.te` 143 行、`private/dumpstate.te` 610 行、`private/property.te` 898 行。

**标签表（`private/file_contexts`）**

- `:41-43`：`# For kernel modules` / `/lib(/.*)? u:object_r:rootfs:s0` / `/system_dlkm(/.*)? u:object_r:system_dlkm_file:s0`
- `:97`：`/dev(/.*)?		u:object_r:device:s0`——本章唯一兜底行，第廿章 P-19/P-20 的 `device` 臂由它出
- `:137` `/dev/ion → ion_device`；`:151` `/dev/nvhdcp1 → video_device`；`:153` `/dev/rpmsg-omx[0-9] → rpmsg_device`；`:154` `/dev/rproc_user → rpmsg_device`——**AOSP 侧登记的 rpmsg 节点名是这两个历史名，不是 `rpmsg%d` / `rpmsg_ctrl%d`**（第廿五章表 C 与第十八章之七据此判定"策略里的名字与内核生成的名字不是同一套"）
- `:218` `/dev/v4l-touch[0-9]* → input_device`；`:221` `/dev/video[0-9]* → video_device`
- `:300` `/system/bin/cameraserver → cameraserver_exec`；`:365` `/system/bin/virtual_camera → virtual_camera_exec`
- `:423` `/(vendor|system/vendor)(/.*)? u:object_r:vendor_file:s0`；`:427` `/(vendor|system/vendor)/etc(/.*)? → vendor_configs_file`；`:436-438` `manifest\.xml`、`compatibility_matrix\.xml`、`/etc/vintf(/.*)?` 三条同标 `vendor_configs_file`
- `:564-565` vendor_dlkm 两行；`:571-572` odm_dlkm 同型两行
- 全文检索 `firmware`：**0 命中**（961 行）；检索 `accel`：**0 命中**；`grep -nwE 'accel|npu|vpu|ipu|imgu|remoteproc'`：**0 条**（第廿二章词边界纪律的实例，子串命中另见卅之七）
- `private/vendor_file_contexts` 在 `main` 分支**不存在**：`?format=TEXT` → 404 `NOT_FOUND: Object is not found`。vendor 侧唯一可取回的标签表是 `vendor/file_contexts`（212 行），检索 `firmware` 0 命中 ⇒ AOSP 不给 vendor 侧任何固件标签，全部由厂商补。

**genfs 表（`private/genfs_contexts`）**

- `:13` `genfscon proc /device-tree/avf → proc_dt_avf`；`:26` `proc /modules → proc_modules`；`:53`/`:57` `proc /sys/kernel/dmesg_restrict`、`proc /sys/kernel/kptr_restrict → proc_security`（**只给类型，不给取值**——第廿九章之十第 12 条的出处）
- `:125-190` 复核结论：`/devices/platform/**` 只有 `:139`（`nfc-power/nfc_power`）一条被点名 ⇒ MFC、venus、camss、remoteproc 平台设备的 sysfs 路径全部继承默认 `sysfs`；`/devices/virtual/**` 被点名的是 block/dm/loop/zram/net/switch/wakeup/misc hw_random/misc uhid/timed_output vibrator/android_usb/cs_etm/system cpu（`:125-126`、`:140-151`、`:187-188`），**没有 `remoteproc`、`rpmsg`、`accel`、`drm`、`media`、`video`**
- `:152-154` 三条：`sysfs_firmware_acpi_tables`、`sysfs_dt_avf`、`sysfs_dt_firmware_android`
- `:172-174` 是 `starting_at_board_api(202504, ...)` 包住的 `sysfs /kernel/mm/cma`
- `:182-186` 被点名的 `/sys/module/**` 只有五条：`dm_bufio/parameters/max_age_seconds` 与 `dm_verity/parameters/prefetch_cluster` → `sysfs_dm_verity`、`lowmemorykiller` → `sysfs_lowmemorykiller`、`tcp_cubic/parameters` → `sysfs_net`、`wlan/parameters/fwpath` → `sysfs_wlan_fwpath`
- 检索 `accel`、`remoteproc`、`rpmsg`、`/class/drm`、`/class/media`、`video`：**0 命中**

**域规则（`private/shell.te`、`private/domain.te`、`dumpstate.te`、`property.te`）**

- `shell.te:3` `# allow shell input injection`；`:342-343` `allow shell input_device:dir r_dir_perms;` / `…:chr_file r_file_perms;`；`:539` `# b/30861057: Shell access to existing input devices is an abuse`、`:542` `# Everyone should have already moved to UiAutomation#injectInputEvent`、`:544` `# their stress tests, and the input command (adb shell input ...) for`、`:546` `neverallow shell input_device:chr_file no_w_file_perms;`
- `shell.te:7-11` 四条 debugfs（tracing 系）+ `atrace_exec:file rx_file_perms`；`:13-15` `userdebug_or_eng(\`  allow shell debugfs_tracing_debug:file rw_file_perms; \`)'
- `shell.te:443` `allow shell sysfs:dir r_dir_perms;`（脚本 2 里 `rl=${rl:-shell.te:443}` 的默认值来源）；`:410` `sysfs_net:dir`；`:444-445` `sysfs_batteryinfo`；`:448` `sysfs_lru_gen_enabled:file`；`:451` `sysfs_mem_sleep:file`
- `shell.te:460` `allow shell { proc_dt_avf sysfs_dt_avf }:dir search;`——DT 面只有 AVF 子树被点名，且只给 search
- `shell.te:463` `allow shell ion_device:chr_file rw_file_perms;`，该文件里**唯一**一条媒体相关字符设备授权
- 检索 `sysfs:file`、`video_device`、`dlkm`、`kernel:system`、`allow shell device:chr_file`：**各 0 条**（Z-19…Z-22）；`device:chr_file` 出现 6 行，每一行的类型都是专项类型而非 `device`（C-05）
- `domain.te:616-617`：`# Do not allow any domain other than init to create unlabeled files.` + `neverallow { domain -init -recovery } unlabeled:dir_file_class_set create;`（`unlabeled` 另在 `:984`、`:993`、`:1015` 出现于三条 `sock_file` neverallow，均与文件读无关）
- `domain.te:1481-1483`：`# vendor, boot, and system_dlkm partitions.` + `neverallow * ~{ system_file_type vendor_file_type rootfs system_dlkm_file_type }:system module_load;`
- `domain.te:2318` 是 `starting_at_board_api(...)` 包裹的一处
- `dumpstate.te:287` 逐字 `dontaudit dumpstate sysfs:file r_file_perms;`（上一行 `# Ignore other file access under /sys.`）
- `property.te:191-198` 逐字八行：`# init is in coredomain...` / `# dumpstate is also in coredomain...` / `neverallow { coredomain -init -dumpstate } {` / `vendor_property_type` / `vendor_internal_property_type` / `-vendor_restricted_property_type` / `-vendor_public_property_type` / `}:file no_rw_file_perms;`

**服务与类型（`private/service_contexts`、`public/service.te`、`public/device.te`）**

- `service_contexts:1` `android.frameworks.cameraservice.service.ICameraService/default → fwk_camera_service`；`:47-48` `ICameraProvider/internal/0 → hal_camera_service`、`/virtual/0 → virtual_camera_service`；`:67-70` `IComponentStore/{default,default1,default2,software} → hal_codec2_service`；`:291` `gpu → gpu_service`；`:304`、`:519` 是 `starting_at_board_api` 两处
- `service_contexts` 检索 `neuralnetwork`：**0 命中**；`grep -nw npu`：**0**（三行子串命中：`:63` `IInputProcessor/default`、`:125` `ITvInput/default`、`:300` `inputflinger`）
- `public/service.te:311` `type hal_camera_service, protected_service, hal_service_type, service_manager_type;`
- `public/service.te:343` `type hal_neuralnetworks_service, hal_service_type, service_manager_type;`——**不带 `protected_service`**（该属性在本树共 64 处标注）
- `public/service.te:314` `hal_codec2_service` 带的是 `isolated_compute_allowed_service`，同样不是 `protected_service`；`:324`/`:326`/`:317`/`:313`（graphics allocator / mapper / drm / cas）全都不带
- `public/service.te` 里 `starting_at_board_api(202504, ...)` 出现在 `:21`、`:74`、`:165`、`:182`、`:292`、`:329-331`（`hal_hwcrypto_service`）、`:339-341`（`hal_mediaquality_service`）、`:378`（`hal_vm_capabilities_service`）
- `public/device.te:47` `type video_device, dev_type;`；`:75-77` `# All devices have a rpmsg device for` / `# achieving remoteproc and rpmsg modules` / `type rpmsg_device, dev_type;`
- `private/device.te` 在本树不存在（404）

**D-4 立下的两条边界**（原样进卅之十八，此处不展开）：`protected_service` 的强制规则（neverallow）不在本次取回的任何一个文件里——`public/service.te`、`private/service.te`、`public/te_macros`、`private/shell.te`、`private/domain.te` 检索 `protected_service` 只命中类型声明，`private/service.te` 仅 88 行且只含三条声明，本报告只描述类型属性、不推断其强制后果；`starting_at_board_api` 的宏定义不在本次取回的文件内（`public/te_macros` 1105 行检索 0 命中，`private/te_macros` 在 Gitiles 上 404），不声明其展开条件。

### 卅之七 D-5…D-8：复核轮与第廿二章计数实测

D-5 是"引用之前先重读原文"的一轮，对象是 `public/te_macros` 与 `private/genfs_contexts`；D-6 复核第廿十章用到的 sepolicy 行（结论已并入卅之六）；D-7 复核第廿一章 FT1–FT10 的三个新读数面；D-8 是第廿二章读数纪律的全部计数命令。

D-5 新增的行号只有 te_macros 两处：

- `public/te_macros:588-591` 原文四行——`#####################################` / `# no_debugfs_restriction` / `# SELinux rules which apply to devices that do not have debugfs restrictions in non-user builds.` / 定义 `no_debugfs_restriction`（`:589` 注释名、`:590` 注释句、`:591` 定义，与第十九章之六引用一致）
- `public/te_macros:611-617`：`:611` `# Userdebug or eng builds`、`:612` `# SELinux rules which apply only to userdebug or eng builds`、`:614` 定义 `userdebug_or_eng`（按 `target_build_variant` 依次比对 `eng`、`userdebug`）、`:616` `# SUPPRESSED_BY_USERDEBUG_OR_ENG -- this marker is used by CTS -- do not modify`、`:617` `)))`
- 同轮负向复核：`drivers/accel/drm_accel.c`（227 行）检索 `firmware` 0 命中 ⇒ accel 框架不管固件装载；`gsc-core.c` 行数复核为 1324（`gsc-core.h` 521），两文件 `firmware` 各 0 命中；`camss.c` 1823 行 0 命中复核通过。

**D-7 三个新读数面（第廿一章）**

atomisp 的版本闸门是本章唯一"结构上不可能生效"的实例，判定链需要逐行落在纸面上（`kernel/common @ android15-6.6`）：

- `sh_css_firmware.c:188-189`：返回类型是 `bool`，函数名 `sh_css_check_firmware_version(struct device *dev, const char *fw_data)`
- `:202` `if (strcmp(file_header->version, release_version) != 0) {` → `:203` `"Firmware version may not be compatible with this driver\n"` → `:204-205` `"Expecting version '%s', but firmware is '%s'.\n"` → `:206` 只是 `}`，不返回、不置错误
- `:208` `/* For now, let's just accept a wrong version, even if wrong */`；`:209` `return false;`（函数唯一出口，错配路径与其汇合）
- 调用点一 `atomisp_v4l2.c:1398-1402`：`err = sh_css_check_firmware_version(isp->dev, isp->firmware->data);` / `if (err) {` / `dev_dbg(&pdev->dev, "Firmware version check failed\n");` / `goto fw_validation_fail;` / `}`；上文 `:1391` 装载、`:1392-1395` 失败的 `-ENOENT` 分支
- 调用点二 `sh_css_firmware.c:253-259`：`ret = sh_css_check_firmware_version(dev, fw_data);` / `if (ret) {` / `:255-256` `IA_CSS_ERROR("CSS code version (%s) and firmware version (%s) mismatch!",` / `:257` `return -EINVAL;` / `:258` `} else {` / `:259` `IA_CSS_LOG("successfully load firmware version %s", release_version);`
- 判定：两处调用都以"真值为失败"作判断，而被调函数恒返回 `false` ⇒ 版本检查在结构上不可能中止装载，两条 `dev_err` 是全部后果。第廿一章 FT3 与第十九章 V7 降级以此为据。
- `atomisp_v4l2.c:58-59`：`static uint skip_fwload;` / `module_param(skip_fwload, uint, 0644);`（模式位复核为 `0644`）

解析器边界的确切行：IPU3 `ipu3-css-fw.c:128`（强转）/ `:129-131`（尺寸与 `h_size` 两条 `goto bad_fw`）/ `:132-134`（`struct_size(..., binary_nr) > css->fw->size`）/ `:136-138`（`dev_info` 打印版本号）；ivpu `ivpu_fw.c:112`（`ivpu_fw_parse()` 签名）/ `:115`（`fw_hdr` 指向 `fw->file->data`）/ `:118-120` 与 `:123-125`（两条 `-EINVAL`，其余六条区间校验见卅之五）；FIMC-IS `fimc-is.c:390-391`（尺寸上下界 + `"wrong firmware size: %zu\n"`）、`:403-404`（`memcpy` + `wmb();`）、`:407-409`（尾部倒读 31 字节 info 并补 NUL）、`:411-413`（7 字节 version 并补 NUL）、`:415`（`is->fw.state = 1;`）。

MFC 调试出口（`kernel/common @ android16-6.12`，`s5p_mfc.c` 1737 行、`s5p_mfc_ctrl.c` 490 行）：`s5p_mfc.c:38` `int mfc_debug_level;`、`:39` `module_param_named(debug, mfc_debug_level, int, 0644);`、`:40` `MODULE_PARM_DESC(debug, "Debug level - higher value produces more verbose messages")`；`s5p_mfc_ctrl.c:281-284` 按 `IS_MFCV6_PLUS` 读 `S5P_FIMV_FW_VERSION_V6` 或 `S5P_FIMV_FW_VERSION`，`:286-287` `mfc_debug(2, "MFC F/W version : %02xyy, %02xmm, %02xdd\n", …)`（硬件上报的三位版本唯一出口），`:288` 起 `s5p_mfc_clock_off(dev)`；降序尝试 `:63-72`（`:69` `dev->fw_ver = (enum s5p_mfc_fw_ver) i;`），`s5p_mfc_common.h:40` `MFC_FW_MAX_VERSIONS` 为 `2`。

框架取证面（FT8，出处为第十七章已取回副本）：`remoteproc_sysfs.c:95-110` 文档注释块（含 `The default value of this entry is "default".` 与 `default:` 一节）、`:111-134` 实现只接受 `disabled`/`enabled`/`inline`（非法值 `"Invalid coredump configuration\n"` → `-EINVAL`）、`:86` `/* Expose the current coredump configuration via debugfs */`（实为 sysfs 属性，`:135` 注册）、`:80-84` `rproc_coredump_str[]`、`:117-120` `"can't change coredump configuration\n"` → `-EBUSY`；`remoteproc_debugfs.c:398-403` `if (!rproc_dbg) return;`、`:405-418` 六节点与模式（`name` 0400、`recovery` 0600、`crash` 0200、`resource_table` 0400、`carveout_memories` 0400、`coredump` 0600）。

**D-8 计数实测（第廿二章读数纪律的原始数字）**

集合一：`kernel/common@android15-6.6` 的 `drivers/accel/` 与 `drivers/staging/media/`（含 ipu3、atomisp/pci、mediatek/vpu 等本次已取回文件）。

| 命令 | 结果 |
| --- | --- |
| `grep -ro "npu" <集合> \| wc -l` | 629 处（`grep -rl` 命中 11 个文件） |
| `grep -rwo "npu" <集合> \| wc -l` | **0** |
| `grep -roh "[A-Za-z_]*npu[A-Za-z_]*" \| sort \| uniq -c`（节选） | `input` 160、`input_config` 61、`inputs` 41、`input_res` 37、`input_curr` 27、`atomisp_input_stream_id` 22、`atomisp_input_format` 16、`input_cnt` 13、`Input` 13、`sp_pipeline_input_terminal` 11、`max_input_width` 10、`input_port_id` 9 |
| `grep -ro "..bin" <集合>` 宿主串 | `s_bin` 142、`p_bin` 120、`y_bin` 100、`r_bin` 61、`o_bin` 58、`w_bin` 49、`i_bin` 31、`->bin` 27、`c_bin` 25、`t_bin` 21（全部是标识符/成员，无文件名） |
| `grep -ro "\.bin" <集合> \| wc -l` | 21 处 |
| `grep -ro '"[^"]*\.bin"' <集合>` | 12 处；`sort -u` 后 10 个不同名字 |
| `grep -rn "firmware_version" <集合>` | 5 处 |
| `grep -rn "DEVICE_ATTR" drivers/accel drivers/staging/media` | 0 |
| `grep -rn "fw_name" <集合>` | 8 处 |

21 处 `.bin` 里的非引号命中逐条：`ivpu_fw.c:46` 的 TODO 注释句、`sh_css.c:3950`/`:3952` 的 `frame->planes.binary.size`、`ipu3-css-fw.c:79`（`binary_header[...]`）、`:101`/`:133`/`:137`/`:142`/`:243`（`file_header.binary_nr`）。10 个引号名逐条：`ivpu_fw.c` 的 `"vpu_37xx.bin"`、`"mtl_vpu.bin"`、`"intel/vpu/vpu_37xx_v0.0.bin"`、`"vpu_40xx.bin"`、`"intel/vpu/vpu_40xx_v0.0.bin"`；`atomisp_v4l2.c` 的 `"shisp_2400b0_v21.bin"`、`"shisp_2401a0_legacy_v21.bin"`、`"shisp_2401a0_v21.bin"`；`ipu3-css-fw.h` 的 `"intel/ipu3-fw.bin"` 与 `"intel/irci_irci_ecr-master_20161208_0213_20170112_1500.bin"`。**FIMC-IS 两条镜像名不在该集合内**，位于 `drivers/media/platform/samsung/exynos4-is/fimc-is.h:32-33`——换个目录集合数字就变，这就是第廿二章要求"集合写全"的理由。

5 处 `firmware_version` 逐条：`vpu_boot_api.h:59` `u64 firmware_version_load_address;`、`:60` `u32 firmware_version_size;`、`atomisp_v4l2.c:1398`（调用）、`sh_css_firmware.c:189`（定义）、`:253`（调用）——**无一处是设备侧可读的版本值**。8 处 `fw_name` 的两个出口：`atomisp_drvfs.c:164-172` 的 `driver_create_file()` 与 `ivpu_debugfs.c:107-115` 的 debugfs 表。

集合二：`platform/system/sepolicy@main` 本地副本。`private/shell.te`：`grep -c "npu"` 7 行、`grep -o` 8 处、`grep -cw` 0（7 行行号卅之六已逐条列出，全部是 `input*` 词）；`private/service_contexts` 6 行 / 14 处 / 词 0；`public/service.te` 6 行 / 6 处 / 词 0；`private/file_contexts` 5 行 / 7 处 / 词 0。

### 卅之八 D-9、D-10：公告页编号与行级读数

D-9 的对象是公告页副本，证据类型 E4 + 逐字引用；页面 URL 形态与 `Last updated` 戳一并登记。

**取回清单（两种路径形态并存）**

- `https://source.android.com/docs/security/bulletin/2026/2026-09-01` → `.sb-2026-09-01.html` 230,027 字节，`Last updated 2026-10-22 UTC`
- `.../bulletin/2026/2026-10-01` → 187,514 字节，`Last updated 2026-10-08 UTC`
- `.../bulletin/2026/2026-08-01` → 175,780 字节，`Last updated 2026-09-30 UTC`，**页内 3 个 `<table>`、`A-` 号 0 个、CVE 号 0 个**
- `.../bulletin/2025-09-01/`（尾斜杠形态）→ 213,701 字节，`Last updated 2026-09-30 UTC`
- `.../bulletin/2024-06-01/`、`.../bulletin/2017-01-01/`、`.../bulletin/2021-01-01/` → 均 200；同一站点两种路径形态并存：2026 用 `/bulletin/2026/2026-09-01`，2025 及更早用 `/bulletin/2025-09-01/`
- 404 形态（各试一次即弃，不重试）：`/docs/security/bulletin/2026-09-01`、`/docs/security/bulletin/2026/2026-09-01.html`、`/security/bulletin/2026-09-01`、`/docs/security/bulletin/android-bulletin-2025-09-01`、`/docs/security/bulletin/2025/2025-09-01`
- 索引页：`.../bulletin` 与 `.../bulletin/aaos` 的原始 HTML 只列出 2021 年的月度条目（`grep -o 'bulletin/20[0-9][0-9]-[0-9][0-9]-01"'` 最大年份 = 2021），后续月份需 JS；`.../bulletin/asb-overview` 原始 HTML 内 CVE 字样 0 处

**CVE 位长与正则变体**（逐页 `grep -oE "CVE-[0-9]{4}-[0-9]{4,}" | sort -u` 后按序列位分档）

- 2026-09-01：175 个唯一编号 → 序列 4 位 11、5 位 164；4 位逐字：`CVE-2026-0001`、`0008`、`0010`、`0054`、`0065`、`0084`、`4967`、`5729`、`7476`、`7477`、`7639`
- 2025-09-01：109 个 → 4 位 8、5 位 100、**6 位 1**：`CVE-2025-021701`（带前导零的 6 位序列；同行 References `A-392852041`、Subcomponent `Net`、Type `EoP`、Severity `High`、另一引用 `Upstream kernel [2]`）
- 2024-06-01：37 个 → 4 位 2、5 位 35；2017-01-01：90 个全部 4 位
- 编号年份 ≠ 公告页年份：2026-09-01 页含 `CVE-2025-48564`、`48565`、`48566`、`48651` 共 4 条；2026-06-01 页 114 条里 29 条是 `CVE-2025-*`；2026-08-01 与 2026-10-01 页分别 0 条（10 月页只有 25 条，全为 2026）
- 跨页去重：`comm -12` 实测 2026-09 与 2026-06/08/10 三页交集均为 **0** 条（同页重复另计）
- 正则变体（对象 `.sb-2026-09-01.html`，`uniq` 后与权威 175 条集合比对）：`CVE-[0-9]{4}-[0-9]{4}` → 73 唯一，其中真实 11、**伪造（截断）62**；`-[0-9]{5}` → 164 全真、漏 11 条 4 位；`-[0-9]{4,}` → 175 全真；`-[0-9]{4}([^0-9]|$)` → 11；`-[0-9]{7}` → 0。伪造样例逐字：`CVE-2025-4856`、`CVE-2025-4865`、`CVE-2026-1228`、`CVE-2026-1238`、`CVE-2026-2045`、`CVE-2026-2046`（各自对应一个 5 位真号的左截断）
- 同页重复：`grep -oE … | sort | uniq -c` → 出现 1 次 155 条、出现 2 次 **20** 条 ⇒ 195 个编号槽位 vs 175 个编号（`+11.4%`）。实例 `CVE-2026-28581` 在偏移 168261（HTML 第 5575 行，平台缺陷表，引用列 `A-479515555`，链接指向 `platform/packages/services/Telecomm`，列序 `ID High 14, 15, 16, 16-qpr2, 17`）与偏移 174083（HTML 第 5753 行，位于第 5720 行 `<h3 id="google-play-system-updates">` 之下的缓解/特性分组表，`<td>Telephonycore</td>` 一格原文是 `CVE-2026-28581, CVE-2026-49881`）各一次——**两处都不是 `Media Framework components` 表**
- 页面结构限制：原始 HTML 只有 3 个 `<h2 id=...>`（`mitigations`、`common-questions-and-answers`、`versions`）与 18 个 `<table>` ⇒ 纯文本 grep 无法把一行归属到某张厂商表，需要 DOM 解析

**引用族（References 列）**

- 文档侧（2026-09-01 FAQ 第 4 问，逐字）：`Entries under the References column of the vulnerability details table may contain a prefix identifying the organization to which the reference value belongs.`；前缀表逐字六行 `A-` Android bug ID、`QC-` Qualcomm reference number、`M-` MediaTek reference number、`N-` NVIDIA reference number、`B-` Broadcom reference number、`U-` UNISOC reference number
- 文档侧（2021-01-01 同问）：同一张表只有 **5** 行，无 `U-` 行
- 页面实测（2026-09-01）：`A-[0-9]{9}` 175、`M-ALPS[0-9]{8}` 10、`M-MOLY[0-9]{8}` 10、`M-AUTO[0-9]{8}` 3、`QC-CR#[0-9]{7}` 5（`#4384595`、`#4409501`、`#4409568`、`#4416730`、`#4416731`）、`U-[0-9]{7}` 7（`U-3293610` 等，位于 `Unisoc components` 表）；文档列出的 `N-`、`B-` 在本页 **0** 行
- 形状错配：文档写 `QC-`、页面写 `QC-CR#`；按文档字面 `QC-[0-9]+` 解析本页命中 0
- `A-` 位长跨年：2017-01-01 页 96 个 `A-` 号全部 **8** 位；2021-01-01（41）、2024-06-01（37）、2025-09-01（108）、2026-09-01（175）、2026-10-01（25）合计 **386 个全部 9 位**；这五页上 `A-[0-9]{8}[^0-9]` 命中 0；2017 页 `A-` 号 96 个 vs CVE 号 90 个（两族数量不等，且 `A-` 号全部出现在 `mitigations` 之后的引用/缓解表里，`head -c 145361` 命中 0）
- 星号：`grep -c 'href="#asterisk"'` → **72**（另 1 处 `id="asterisk"` 锚点）。按紧邻前一个编号归属：`A-` 44、`M-MOLY` 10、`M-ALPS` 10、`U-` 7、`M-AUTO` 3、`QC-CR#` **0**。同格并存实例逐字（Unisoc 表）：`A-508074243` + `U-3304745` + `*`
- FAQ 第 5 问逐字：标题 `What does an * next to the Android bug ID in the References column mean?`，正文 `Issues that are not publicly available have an * next to the corresponding reference ID. The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site.`（标题只提 Android bug ID，正文写 corresponding reference ID）
- 条目可变逐字（2026-09-01 `Versions` 表）：`1.0 / September 8, 2026 / Bulletin Published`、`1.1 / September 15, 2026 / Updated issue list and added AOSP links`、`1.2 / September 16, 2026 / Updated issue list`、`1.3 / September 17, 2026 / Updated issue list`；2026-10-01 页首行 `1.0 / October 5, 2026 / Bulletin Published`
- 2026 起 AOSP 发源节奏（逐字，出现在 2025-09-01、2026-08-01、2026-09-01 三页正文）：`Effective in 2026, to align with our trunk stable development model and ensure platform stability for the ecosystem, we will publish source code to AOSP in Q2 and Q4. For building and contributing to AOSP, use`

**D-10 行级读数**（提取方法逐字登记：`sed -e 's/<[^>]*>/\n/g' <页> | grep -v "^\s*$"` 生成令牌流，再按"命中 CVE 行后取 N 个非空令牌"取列；列序实测为厂商表 `CVE | References | Severity | Subcomponent`（References 里的 `*` 是独立令牌），平台表 `CVE | References | Type | Severity | Updated AOSP versions`）

- `Arm components`（2026-09-01，6 行，全部 `*`、全部 High、**无厂商引用号**）：`CVE-2026-0001 A-466377983 * High Mali`、`CVE-2026-5729 A-527029873`、`CVE-2026-7476 A-527043872`、`CVE-2026-7477 A-525773732`、`CVE-2026-12285 A-525777064`、`CVE-2026-12387 A-525782063`
- `Imagination Technologies`（25 行，节选）：`CVE-2026-21732 A-468248021 * High PowerVR-GPU`、`CVE-2026-21733 A-483582720 *`
- `MediaTek components`（节选）：`CVE-2026-20464 A-518566453 M-ALPS11104718 * High hevc decoder`；`CVE-2026-20501 A-528211066 M-ALPS11262030 * High vdec`；`CVE-2026-20502 A-528209858 M-ALPS11262030 * High vdec`（**两条 CVE 共用一条 `M-` 号，但各自有独立 `A-` 号**）；`CVE-2026-20467 A-518564544 M-AUTO00837766 * High apusys`；`CVE-2026-20468 A-518595862 M-AUTO00833804 * High apusys`；`CVE-2026-20469 A-518601919 M-AUTO00834868 * High trusted_mem`（**Subcomponent 与系列名不同源：`trusted_mem` 走 `M-AUTO`**）；`CVE-2026-20473 … M-ALPS11019722 * High display`；`CVE-2026-20477 … M-ALPS11009963 * High display`；`CVE-2026-20481 … M-ALPS10965373 * High geniezone`；`CVE-2026-20458 … M-MOLY01402160 * High Modem`；`CVE-2026-20460 … M-MOLY01811421 * High Modem`
- `Unisoc components`：`CVE-2026-21548 A-517353300 U-3293610 * High Modem`、`CVE-2026-21549 A-517316530 U-3379435 * High Modem`；表标题句 `These vulnerabilities affect Unisoc components and further details are available directly from Unisoc. The severity assessment of these issues is provided directly by Unisoc.`
- `Qualcomm components`（5 行 Subcomponent 全 `Security`，**全部无星**）：`CVE-2026-24087 A-510336694 QC-CR#4409501 High Security`、`CVE-2026-24088 A-489081076 QC-CR#4409568`；表句 `These vulnerabilities affect Qualcomm components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm.`
- `Qualcomm closed-source components`（11 行，Subcomponent 全 `Closed-source component`，References 只有 `A-` + `*`，**无 `QC-CR#`**）：`CVE-2026-25289 A-509524204 * Critical Closed-source component`、`CVE-2026-21383 A-473523681 * High`、`CVE-2026-24079 A-480963580 * High`
- 平台表（2026-09-01）：`CVE-2026-28581 A-479515555 ID High 14, 15, 16, 16-qpr2, 17`；`CVE-2026-28604 A-499053500 RCE Critical 14, 15, 16, 16-qpr2, 17`；`CVE-2025-48564 A-403565650 EoP High 14, 15, 16, 16-qpr2, 17`（**2025 编号进 2026-09 平台表，无星**）
- `Kernel` 段（2026-09-01）：`CVE-2026-31629 A-499350302 Upstream kernel [ 2 ] EoP Critical NFC`、`CVE-2026-58846 A-508703529 Upstream kernel EoP Critical Protected Kernel-Based Virtual Machine`、`CVE-2026-58941 A-508703583 Upstream kernel [ 2 ] EoP Critical`、`CVE-2026-43049 A-482992246 Upstream kernel [ 2 ] EoP High Human Interface Device`、`CVE-2026-52993 A-500022799 Upstream kernel [ 2 ] RCE Critical Transparent Inter-Process Communication`（**References 列的一种新形态：`Upstream kernel` + 方括号链接索引，不在前缀表里**）
- `Google Play system updates` 段句逐字：`The following issues are included in Project Mainline components.`；段内行如 `CVE-2026-28593 A-388272624 EoP High 14, 15, 16, 16-qpr2, 17`
- 单行厂商表（2026-09-01）：`CVE-2026-28616 A-483923525 * High SUW`（表句 `This vulnerability affects Setup Wizard components and further details are available directly from Setup Wizard. The severity assessment of this issue is provided directly by Setup Wizard.`）；`CVE-2025-48651 A-529876547 * High StrongBox`（`Tsingteng Micro`）——**两句用 `this issue`（单数），与六张多行表的 `these issues` 不同**
- 缓解/特性分组表（2026-09-01）：一格多编号形式 `CVE-2026-28581, CVE-2026-49881` → `UWB`；`CVE-2026-28623, CVE-2026-28652` → `WiFi`；`CVE-2026-28604` → `adbd`；`CVE-2026-28622, CVE-2026-28638, CVE-2026-28671, CVE-2026-45517, CVE-2026-45519, CVE-2026-45525` → `Telephonycore`（⇒ 195/175 双现的来源之一是这类表）
- 严重度口径句（逐字，四页正文皆有，链接到 `source.android.com/security/overview/updates-resources.html#severity`）：`The severity assessment is based on the effect that exploiting the vulnerability would possibly have on an affected device, assuming the platform and service mitigations are turned off for development purposes or if successfully bypassed.`

**跨期结构对照**（`grep -o "The severity assessment of these issues is provided directly by [A-Za-z ]*\."`，在压平空白后的文本上执行；换行会把该句切断，必须压平后再数）

| 页面 | 厂商定级表数 | 厂商构成 | 唯一 CVE 数 | 有 `Type` 列的平台表行 |
| --- | --- | --- | --- | --- |
| 2026-09-01 | 6 | Arm 1、Imagination 1、MediaTek 1、Qualcomm 2、Unisoc 1 | 175 | 是（含 `Google Play system updates` 与 `Kernel` 段） |
| 2026-06-01 | 5 | Imagination 1、MediaTek 1、Qualcomm 2、Unisoc 1（**无 Arm**） | 114 | 是 |
| 2025-09-01 | 5 | Arm 1、Imagination 1、MediaTek 1、Qualcomm 2（**无 Unisoc**） | 109 | 是 |
| 2026-10-01 | **0** | — | 25 | 全部行都是平台表形态（`14, 15, 16, 16-qpr2, 17` 这类版本列） |

2025-09-01 行级：`CVE-2025-1246 A-402121892 * High Mali`、`CVE-2025-3212 A-421179224 * High Mali`、`CVE-2024-7881 A-361573291 * High CPU`、`CVE-2025-27042 A-409047250 QC-CR#3935673 High Video`、`CVE-2025-27043 A-409047487 QC-CR#3935643 High Video`；`High \| Video` 2 行、`High \| Mali` 2 行、`High \| PowerVR-GPU` 10 行、`High \| Closed-source component` 24 行、`High \| Modem` 3 行。2026-06-01 Subcomponent 分布（严重度后一列令牌）：`Modem` 21、`Closed-source component` 16+3(Critical)、`geniezone` 5、`PowerVR-GPU` 3、`Display` 2、`preloader` 1、`Net` 1；`High \| 14, 15, 16, 16-qpr2` 32 行（平台表）。

D-9 补记两项：`.../bulletin/2026/2026-06-01` → `.sb-2026-06-01.html` 220,528 字节，`Last updated 2026-10-13 UTC`，`<table>` 12 个、`A-` 号 114 个、唯一 CVE 114 个；序列位长 **4 位 43 个、5 位 71 个**（与 2026-09 页的 11/164 差别很大 ⇒ 位长分布必须逐期观测，不能沿用上一期），其中 `CVE-2025-*` 29 条。`A-` 位长跨期逐字命令：`grep -o "A-[0-9]\{5,\}" <页> | sed 's/A-//' | awk '{print length($0)}' | sort | uniq -c`。

**D-10 末尾的未取回声明已被本轮补取回推翻，见卅之十九**：第 5 章记录的 `Dolby DD+ Codec CVE-2025-54957`、`Misc OEM VBMeta CVE-2025-48613`、`Pixel GDMC/GSA CVE-2026-56952 / 0198 / 55307` 与 `DA CVE-2025-20696` 当时只在 2026-09、2026-06、2025-09、2026-10 四页副本上检索过（`grep -l` 全副本），所以那次的准确结论只是"在这四页上为 0 命中"。补取回后：**六个编号全部命中**，其中 `CVE-2025-20696` 就在当轮已持有但没纳入那次检索的 2025-09 页上（卅之八登的 2025-09 行级读数是**节选**，只列了 Arm/Imagination/Qualcomm 三张表的行，MediaTek 表那一页没读）。这条是**报告自身最重的一次检索范围失误**，逐字记录见卅之十九。

### 卅之九 D-11：框架侧复核与本轮修正的四条引用

写第廿五章时为核**框架侧读数面**补取的本地副本（仓库仍是 `kernel/common` 的 `android15-6.6` 与 `android16-6.12` 两分支加 `platform/system/sepolicy@main`，取回方式同 D-0）：

- `android16-6.12`：`drivers/remoteproc/remoteproc_debugfs.c` 430、`remoteproc_sysfs.c` 282、`drivers/rpmsg/rpmsg_core.c` 731、`rpmsg_ctrl.c` 245、`rpmsg_char.c` 582、`mtk_rpmsg.c` 410、`include/linux/remoteproc/mtk_scp.h` 69、`panthor_fw.h` 503、`panthor_fw.c` 1371、`panthor_device.h` 386、`a6xx_gpu.c` 2365、`a6xx_hfi.c` 861、`venus/{core.h 569,hfi_msgs.c 853,vdec.c 1894}`、`s5p-mfc/{s5p_mfc_common.h 806,s5p_mfc_opr_v6.c 2708}`
- `android15-6.6`：`remoteproc_sysfs.c` 282、`remoteproc_debugfs.c` 430、`rpmsg_core.c` 729、`rpmsg_ctrl.c` 245、`include/linux/rpmsg.h` 352、`include/uapi/linux/rpmsg.h` 56、`venus/hfi_msgs.c`、`s5p-mfc/s5p_mfc_opr_v6.c` 2534
- `sepolicy@main`：`private/dumpstate.te` 610、`private/property.te` 898、`public/device.te` 143（`private/device.te` 取回 404）

逐字复核通过的读数面（第廿五章四张能力事实表据此写）：`remoteproc_sysfs.c` 的 `DEVICE_ATTR_RW(recovery)` `:73`、`DEVICE_ATTR_RW(coredump)` `:135`、`DEVICE_ATTR_RW(firmware)` `:167`、`DEVICE_ATTR_RW(state)` `:217`、`DEVICE_ATTR_RO(name)` `:227`，`rproc_is_visible()` 起于 `:229`、降权条件四条 `:236-239`、属性数组 `:245-252`、`.is_visible` 注册 `:256`（两分支同址）；`remoteproc_debugfs.c` 的 `if (!rproc_dbg)` `:402`、六节点 `:407/409/411/413/415/417`、根目录 `:424`（两分支同址）；`rpmsg_core.c`（6.12）`driver_override_store()` 签名跨 `:424-427`（6.6 起于 `:422`）、`driver_set_override()` `:429`、`DEVICE_ATTR_RW(driver_override)` `:449`（6.6 `:447`）、`rpmsg_id_match()` 的 `strncmp(...)` `:480`、override 路径 `:491-492`、`rpmsg_register_device_override()` `:611`、`strscpy_pad(...)` `:618`、`dev_set_name(...)` `:620-621`；`rpmsg_ctrl.c` `:74/84/87/88/125/168` 与 `rpmsg_char.c:459`；`include/linux/remoteproc/mtk_scp.h:47` `SCP_IPI_NS_SERVICE = 0xFF,`（`:48` `SCP_IPI_MAX = 0x100`）；`s5p_mfc_common.h:40` `#define MFC_FW_MAX_VERSIONS	2`；venus（6.6）`core.h:173` 文档行、`core.h:233-237` `struct venus_img_version { u32 major; u32 minor; u32 rev; } venus_ver;`、`hfi_msgs.c:18-20`（`SMEM_IMG_VER_TBL 469`、`VER_STR_SZ 128`、`SMEM_IMG_OFFSET_VENUS (14 * 128)`）、三条硬编码 `sscanf` 前缀 `:304/:309/:314`、失败打印 `:319`、`dev_dbg` 版本打印 `:322-323`、SMEM 表项写入 `:323-330`、dispatch `:345-346`，`vdec.c` 注释块 `:571-575`（`SC7280 also reports "1.0.<hash>" parsed as 1.0.0; restricting to IRIS2` / `avoids misapplying this quirk and breaking VP9 decode on SC7280.`）、门一 `:577`、门二注释 `:710` 与 `if (is_fw_rev_or_newer(inst->core, 5, 4, 51))` `:711`；panthor（6.12）`panthor_fw.h:193-202` 结构、`panthor_fw.c:192` 编码宏 `CSF_IFACE_VERSION(major, minor, patch)`、`:194-196` 三个拆位宏、`panthor_get_instr_features()` `:878-886`（首条闸门 `:882-883`）、打印 `:925-927`；sepolicy `dumpstate.te:287`、`property.te:191-198`、`device.te:47` 与 `:75-77`。

**本轮修正的四条引用**（原文已同步改写，此处保留修正过程）：

1. **`CSF_IFACE_VERSION` 的编码宏位置**：第十一、廿一章原写"编码在 `:192-196`"，紧跟 `panthor_fw.h:193-202` 之后易被读成同一个文件。改为逐字 `panthor_fw.c:192-196`（实测该宏在 `.c`，`.h` 内检索 `CSF_IFACE_VERSION` 0 命中）。
2. **Venus 版本读数出口**：第十九、廿一章原写"唯一出口是 `dev_dbg`（`hfi_msgs.c:346`、`:359-360`）"。实测 `:346` 是 dispatch 行、`:359-360` 属于 `hfi_sys_rel_resource_done()`，与版本无关；`dev_dbg` 在 `:322-323`，另有一条 SMEM 表项写入出口 `:323-330`（第十五章本就登记过）。两章均已改写为两条出口并列。
3. **`/dev/video*` 的类型行**：廿一章原写 `device.te:75-77`。实测 `75-77` 是 `rpmsg_device` 的三行块（第十八、十九、二十章引用它是对的），`video_device` 单独在 `:47`。已改为 `device.te:47`。
4. **`driver_override` 的 store 行**：原写 `rpmsg_core.c:425-427`（6.12）。实测函数签名从 `:424` 起、跨 `:424-427`，6.6 从 `:422` 起；`DEVICE_ATTR_RW` 在 6.12 `:449`、6.6 `:447`。第廿五章按分支分列，第十八章保持 6.12 不变。

**D-11 发现的取回缺口去向**：`qcom_glink_native.c` 已销账（卅之十一）；六个未取回案例编号已由卅之十九补取回销账；第廿一章引用的上游 `v7.3` 形态（`panthor_fw_has_glb_state` `:326-331`、`panthor_fw_has_64bit_ep_req` `:333-338`）不在这两个 GKI 分支里，属**另一版本单列**，须按上游标签重取，不与本表混用行号；`protected_service` 的 neverallow 规则、`starting_at_board_api` 宏定义、`rproc->sysfs_read_only` 赋值点、厂商 `vendor_file_contexts`、`kptr_restrict`/`dmesg_restrict` 厂商默认值——四项沿 D-4 与 D-6 的结论，仍是边界项。

**第廿五章能力事实表的登记口径**（写在这里，防止后续章节走偏）：表 A（装载决策输入）10 行、表 B（取证与调试出口）11 行、表 C（跨核通道与用户态入口）6 行、表 D（策略门）11 行，共 **38 行**。四张表**只含**对象名、声明它的源码位置（含分支）、源码声明的八进制模式、该面对应"携带哪一类信息"。**不含**任何打开方式、写入格式、调用序列、参数取值，以及"如何用它们改变固件内容或装载结果"。写侧节点（`crash 0200`、五个 ivpu `0200`、`firmware`/`state`/`recovery`/`coredump` 的 RW 属性、`dbgfun`/`dbgopt`、`skip_fwload`、三个 RPMSG ioctl、`DRM_IOCTL_QAIC_MANAGE`）一律按能力事实登记，第廿七、廿八章脚本对它们是**只 stat 不 open**。

### 卅之十 D-12：公告页样本扩容与逐期结构台账

第廿三章的编号陷阱结论原先立在四页副本上。本轮把同一取回动作扩到 14 期，得到一张完整的结构台账——目的不是增加样本量，而是**证明"页形态本身逐期变化"**，因此任何以"上一期的形状"为常数的解析器都会在某一期静默失效。

**探针台账**（全部为 `https://source.android.com/docs/security/bulletin/...`，逐字命令 `curl -s -o <文件> -w "%{http_code}"`）

14 次探针：11 次 200、3 次 404（`2025-10-01`、`2025-11-01`、`2025-12-01` 三个 2025 年尾月在该路径形态下不存在）。404 各试一次即弃，不重试、不换形态试探（卅之八已登记五种 404 的 URL 形态）。

**逐期结构实测**（本轮在同一批副本上重算；`tables` = `grep -c '<table'`，`cve` = `grep -oE "CVE-[0-9]{4}-[0-9]{4,}" | sort -u | wc -l`，`A-` = `grep -oE 'A-[0-9]{9}' | wc -l`）

| 副本 | 字节 | `<table>` | 唯一 CVE | `A-[0-9]{9}` | `Last updated` |
| --- | --- | --- | --- | --- | --- |
| 2017-01-01 | 250,828 | 49 | 90 | 0 | — |
| 2021-01-01 | 193,914 | 11 | 41 | 41 | — |
| 2024-06-01 | 193,152 | 11 | 37 | 37 | — |
| 2025-09-01 | 213,701 | 14 | 109 | 108 | 2026-09-30 UTC |
| 2026-01-01 | 177,645 | 4 | 1 | 0 | 2026-09-30 UTC |
| 2026-02-01 | 176,017 | 3 | **0** | 0 | 2026-09-30 UTC |
| 2026-03-01 | 220,982 | 15 | 108 | — | 2026-10-13 UTC |
| 2026-04-01 | 183,460 | 8 | 2 | — | 2026-09-30 UTC |
| 2026-05-01 | 179,054 | 5 | 1 | — | 2026-09-30 UTC |
| 2026-06-01 | 220,528 | 12 | 114 | — | 2026-10-13 UTC |
| 2026-07-01 | 176,006 | 3 | **0** | 0 | 2026-09-30 UTC |
| 2026-08-01 | 175,780 | 3 | **0** | 0 | 2026-09-30 UTC |
| 2026-09-01 | 230,027 | 18 | 175 | 175 | 2026-10-22 UTC |
| 2026-10-01 | 187,514 | 6 | 25 | 25 | 2026-10-08 UTC |

三个 2026 年月份（02、07、08）的页面只有 3 张 `<table>`、0 个 CVE、0 个 `A-` 号 ⇒ **"某月公告里没有缺陷表"是真实形态，不是取回失败**。台账若把这类月份记成"未披露"就把采集事件误读成了安全事件，这正是第廿二章规则八（入口不可读是一等状态）在公告侧的同型问题。

**表标题台账**（`grep -o '<h[23] id=[^>]*>[^<]*</h[23]>'` 抽取，落 `.x17-titles.tsv`）：59 行、**23 个不同标题**；逐期行数 2025-09 11、2026-01 1、2026-03 12、2026-04 6、2026-05 2、2026-06 9、2026-09 15、2026-10 3。23 个标题逐字：`Android Runtime`、`Android runtime`、`Arm components`、`Dolby`、`Framework`、`Google`、`Google Play system updates`、`Imagination Technologies`、`Kernel`、`Kernel components`、`MediaTek components`、`Misc OEM`、`NXP components`、`Qualcomm closed-source components`、`Qualcomm components`、`STMicroelectronics`、`Setup Wizard`、`System`、`TV`、`Thales`、`Tsingteng Micro`、`Unisoc components`、`Widevine DRM`。

**`Android Runtime` / `Android runtime` 是两个不同的字符串**：2025-09-01 页写首字母大写形式，2026-09-01 页写小写 `r`。台账若按标题做主键，同一张表在跨期合并时会裂成两行——这条与本报告第廿三章"编号位长"那组结论属同一类失效，只是发生在文本层而不是数字层。

**表头形状台账**（逐表提取首行 `<th>` 列表，落 `.x17-ths.tsv`）：74 行，形状只有六种——

| 列数 | 表头逐字 | 行数 |
| --- | --- | --- |
| 4 | `CVE, References, Severity, Subcomponent` | 31 |
| 5 | `CVE, References, Type, Severity, Updated AOSP versions` | 15 |
| 5 | `CVE, References, Type, Severity, Subcomponent` | 6 |
| 2 | `Subcomponent, CVE` | 6 |
| 4 | `Abbreviation, Definition, Prefix, Reference` | 8 |
| 3 | `Version, Date, Notes` | 8 |

按 4 列厂商形态计数：2025-09-01 6 张、2026-01-01 1 张、2026-03-01 7 张、2026-04-01 4 张、2026-06-01 5 张、2026-09-01 8 张，2026-05-01 与 2026-10-01 **0 张**。这里必须留一条口径提醒：**"4 列表的张数"不等于第廿三章那句"厂商自定 severity 的表数"**——后者是按"表标题句里出现 `The severity assessment of these issues is provided directly by …`"计数的（2026-09-01 页为 6），2026-09 页的 8 张 4 列表里有 2 张（`Setup Wizard` 之外的单数句表等）用的是 `this issue` 而不是 `these issues`。**两个计数不能互相替换，引用时必须说明按哪条命令算的。**

**跨期编号复现**：`CVE-2025-48651` 在 2026-09-01 页是 `Tsingteng Micro` 表里的 `A-529876547 * High StrongBox`，同时是 2025-09 期编号族的一员——**同一个编号在不同期以不同厂商表出现**，因此"按编号去重"与"按编号 + 期次去重"会得到不同的台账行数。结合 D-9 的 195/175 双现，本报告对公告侧计数一律注明"页内去重后"或"跨页去重后"，不给无口径数字。

### 卅之十一 D-13：`qcom_glink_native.c` 销账与 DT binding 的行号漂移

这一节把第廿九章之十第 3 条挂着的缺口关掉，并把一次"取错目录"的完整过程写进台账——它示范的是本报告最容易犯、也最难自见的一类错误：**取回失败的原因在自己的路径假设里，而不在站点上**。

**取回过程**：第十八章之六引用 glink 时按 `drivers/soc/qcom/qcom_glink_native.c` 请求，Gitiles 返回 404；当时把它登记成"本树不可达"。本轮改为按目录列举检索，文件实际在 **`drivers/rpmsg/`**（`android16-6.12`，本地副本 `.src17/kernel/common/android16-6.12/drivers/rpmsg/qcom_glink_native.c`，**1,999 行**）。取回后第十八章之六的行号逐条重测通过，缺口销账。

**本轮据以支撑"名字读数有伴读要求"的原句**：

- `:1846-1847`：`static ssize_t rpmsg_name_show(struct device *dev,` / `struct device_attribute *attr, char *buf)`
- `:1852`：`ret = of_property_read_string(dev->of_node, "label", &name);`
- `:1853-1854`：`if (ret < 0)` / `name = dev->of_node->name;`（**回落值是 DT 节点名，不是任何固件或通道名**）
- `:1856`：`return sysfs_emit(buf, "%s\n", name);`
- `:1858`：`static DEVICE_ATTR_RO(rpmsg_name);`
- `:1860-1863`：`static struct attribute *qcom_glink_attrs[] = { &dev_attr_rpmsg_name.attr, NULL };`
- `:1877`：`static int qcom_glink_create_chrdev(struct qcom_glink *glink)`
- `:1886`：`channel = qcom_glink_alloc_channel(glink, "rpmsg_chrdev");`
- `:1898`：`return rpmsg_ctrldev_register_device(rpdev);`
- 同文件另有两处读 `label`：`:1931` `ret = of_property_read_string(dev->of_node, "label", &glink->label);`、`:1945` `ret = qcom_glink_create_chrdev(glink);`

**为什么这组读数只升级"可复核性"、不升级"结论"**：`rpmsg_name` 是 **glink 设备节点**上的属性，不是 rpmsg 通道节点的 `driver_override`；它给的是 DT `label` 或节点名，与"这条边上跑的是哪个固件"没有对应关系。第廿九章之十第 1 条（`driver_override` 无伴读）与第 3 条（glink 边数 vs `rpmsg_ctrl` 节点数的比较）因此各自保持原处置：来源已补，格子未补。脚本 2 不产出行，比较结论不进任何指标分母。

**行号漂移实例（同一文件、两个树、同一句话）**：`Documentation/devicetree/bindings/gpu/arm,mali-valhall-csf.yaml`

- `kernel/common @ android16-6.12`：147 行，可发现性那句在 **`:22`**
- `torvalds @ master`：199 行，同一句在 **`:24`**
- 两处逐字相同：`- const: arm,mali-valhall-csf   # Mali Valhall GPU model/revision is fully discoverable`

这是本报告里最干净的一个"行号不是标识符"的实例：文件同路径、内容同句，只因树不同行号差 2。**任何跨树引用必须把树名写进引用串里**（第廿五章表 A 的登记口径把分支写死，正是为此）。同一句还额外承担一个语义作用：`fully discoverable` 是**设备树 binding 注释里的限定词**，它描述的是 GPU 型号可枚举，不是固件版本可读——第十七章把它列为"generally 类限定词"同族样本。

### 卅之十二 D-14：厂商与平台文档页、设备树补件

**厂商入口页副本**（本轮全部重取并重测；页面尺寸与标题一并登记，因为第廿九章之十第 10 条要求"壳会变"必须可核对）

逐份登记本地副本名、字节数、`<title>` 原文与本轮实测（标题里的竖线原样保留，故不用表格承载）：

- `.sb17-arm.html`，120,201 字节，`<title data-react-helmet="true">Arm Product Security Center</title>`——入口页可匿名取正文，三类文档分工句命中。
- `.sb17-imgtec.html`，821,380 字节，`Imagination GPU Driver Vulnerabilities - Imagination`——页自称覆盖 `graphics drivers`；`PSP-19`、`PP-171174`、`DDK Releases up to and including` 逐条命中。
- `.sb17-mtk.html`，152,778 字节，`MediaTek | Security Bulletin`（索引页）——**九之二那句自述不在此页**，此页只有 `The MediaTek Product Security Bulletin shows MediaTek's commitment to keeping providing secure and reliable products.`
- `.sb17-mtk-oct.html`，148,502 字节，`October 2026 MediaTek Security Bulletin`——第九章之二引用的三句（`…affecting certain MediaTek chipsets`、`Device OEMs have been notified … at least two months before publication`、`Common Vulnerability Scoring System version 3.1 (CVSS v3.1)`）全部命中此页。
- `.sb17-nnapi.html`，238,985 字节，`404 | Page Not Found | Android Developers`——**NNAPI 页未取回**，登记见卅之二十一。
- `.sb17-suppl.html`，147,494 字节，`Supplemental security patches | Security | Android Open Source Project`——`KEY_VENDOR_SUPPLEMENTAL_PATCHES` 与 `Aggregates patches from /vendor and /odm partition paths` 各命中。
- `.sb17-vendor-impl.html`，110,154 字节，`404 | Page Not Found | Android Open Source Project`——厂商实现指南那次探针未取回。
- `.sb17-vulkan.html`，213,919 字节，`Implement Vulkan | Android OS core topics | Android Open Source Project`，`Last updated 2026-10-09 UTC`——第三章之五与第十五章引用的四句全部命中（逐条见下）。

Vulkan 页逐字命中项：`A Vulkan driver, provided by SoCs such as GPU IHVs, that implements the Vulkan API`、`Consult your SoC vendor to request driver support`、`Since Android 15, the loader supports APEX to load the Vulkan driver`、`Set ro.vulkan.apex to the name of the Vulkan APEX`。两条驱动路径 `/vendor/lib64/hw/vulkan.<ro.hardware.vulkan>.so` 与 `/vendor/lib64/hw/vulkan.<ro.board.platform>.so` 在原始 HTML 里写作 `&lt;ro.hardware.vulkan&gt;`（本页共 6 处 `&lt;`），**必须先做实体解码再比对**——它们属第二级（页面子串）而不是第一级（源码整行），解码步骤漏掉即判不出命中。

**公告相关页副本**：`.sb-aaos.html` 205,419 字节（`Android Automotive OS Update Bulletin`）、`.sb-wnew.html` 137,674 字节（`Latest security bulletin`）、`.sb-faq.html` 110,132 与 `.sb-android-bulletin-2026-09-01.html` 110,156 字节（两份都是 `404 | Page Not Found` 壳）。**"同一站点两个不同 URL 返回同尺寸壳"在这批副本里是实测事实**：110,132 与 110,156 相差 24 字节，形状相同、尺寸不完全相同 ⇒ 判 404 要看 `<title>`，不能只看尺寸。

**设备树补件**：`kernel/common @ android16-6.12` 的 `arch/arm64/boot/dts/mediatek/mt8183.dtsi`，2,238 行。`scp: scp@10500000` 节点起于 `:1036`，`:1044` 是 `memory-region = <&scp_mem_reserved>;`；同节点内 `:1041` `interrupts = <GIC_SPI 174 IRQ_TYPE_LEVEL_HIGH>;`、`:1045` `status = "disabled";`。**这条补件的用途有边界**：它是第十七章之九把 `trusted_mem` 读成"受保护内存通路"时引用的唯一公开实物锚点（联发科公告里 `trusted_mem` 这个 Subcomponent 在内核侧无同名实体），不能反过来用作"该设备 SCP 保留内存实际大小/地址"的读数——`status = "disabled"` 在同一节点里。

**页内检索负向读数**（第廿九章引用过的框架与 binding 之外，本轮补记）：`arm,mali-valhall-csf.yaml` 两树版本内检索 `firmware` 与 `OP-TEE` 的命中集合不同，本报告只引用可发现性那一句，不引用其 binding 的固件属性列表。

### 卅之十三 D-15：锚点审计、引文审计与内联跨度审计的完整过程

这一节把前两节之外**所有"报告自己怎么核自己"的读数**收在一张台账里。三条审计各自有独立的判据、独立的失效模式，因此分开登记，不合并成一个总百分比。

**审计一：锚点审计（源码引用是否可解析）**

工具与流程：抽取脚本用正则从草稿里取出形如 `name.te:123`、`file.c:12-34` 的锚点串，归一化成 `(文件名基名, 行号区间)`，再在 `.x17-index.usv`（`basename \x1f lineno \x1f content` 三列，146,094 行）里查该基名是否有对应行；汇总脚本打印未解析项与越界项。这套工具**不随报告发布**，卅之二十四问 3 给的是读者自查行号漂移的等价命令，不是这两条审计本体。

结果（一到廿九章 + 本章共 21 份草稿一起跑）：**1,168 个锚点令牌（其中 1 个只有文件名没有行号，有效 1,167）→ 151 个不同前缀 → 107 个不同基名 → 101 个解析通过 → 6 个未解析 → 0 个区间越界**。六个未解析基名分成两类，都不是"引用了不存在的源码行"：

- **4 个带连字符前缀的基名**（`-shell.te`、`-domain.te`、`-remoteproc_sysfs.c`、`-ivpu_debugfs.c`，共 **9 处**令牌）来自第廿八章脚本里的 `rl=${rl:-shell.te:443}` 形态——Shell 的默认值写法让冒号前的 `-` 被正则当成文件名的一部分。落在 `.w17s.md` 的 144、168、184、195、207、297、356 行。
- **`file.c` 与 `name.te` 各 1 处**来自本节上一行自己写的示例形态（"`name.te:123`、`file.c:12-34`"）。审计把本章也算进语料，于是本章举的假例子回到本章头上——**这是自指引，不是引用缺陷**，但它提醒一件事：锚点审计不能宣称"零未解析"，只能宣称"未解析项可逐条归因"。

另有一处口径必须写清：锚点计数会把本章自己算进去，所以**本章每改一次行，令牌数就变一次**。上面这组数是发布冻结的那一次跑出来的读数，读者若拿草稿版重跑得到不同的令牌数属正常，未解析集合的内容才是判据。审计的判据是"能在索引里找到该基名对应的这一行"，不是"这一行内容像不像我引用的东西"——后者由审计二承担。

索引的已知局限必须一并登记：`.x17-index.usv` **只存基名，不存完整路径**，因此 `drivers/accel/ivpu/ivpu_fw.c` 与任何同名文件在索引层不可区分。本报告用两个手段兜住：引用串一律带分支前缀（第廿五章口径），加上文件级复核（卅之九 D-11 那轮逐条 `grep -n` 实测）。这一条不是抽象风险，本轮就撞上一次：第十二章章头的"出处约定"声明的分支是 `kernel/common@android16-6.12` 的 `drivers/gpu/drm/msm/adreno/`，该章十二之三把 `a6xx_gpu.c` 那条 lowest-nibble 注释引在 `:740-744`；同一份文件在 `android15-6.6` 副本里那条注释**不在同一个行号上**（差出三百余行），而索引只有基名，所以锚点审计对这两个分支的行号一视同仁、谁都判不出错。读者要复核归属，用的命令是 `grep -n "lowest nibble is 0xa" .src17/kernel/common/<分支>/drivers/gpu/drm/msm/adreno/a6xx_gpu.c`——本章不把它在 6.6 里的行号写进来，因为本章的判据是"不引入前文未出现过的源码行号"（本节开头那条），写进来反而多一处没法回查的数。实测的本目录内的确存在同名多副本：178 个副本里 134 个不同基名，**36 个基名覆盖 80 个文件**、98 个基名只有单副本。按"同名副本数"分布是 **1 份 98 个基名、2 份 33 个、3 份 1 个、5 份 1 个、6 份 1 个**（没有 4 份那一档）；卅之二表里的 86/67/15/6/2/2 是**按仓库与分支目录**的分桶，两种分桶口径不同、不可互换，此前本节把它写成"按副本数 1/2/3/4/5/6 分布"是错的，已改（卅之十七第 16 条）。另有一条必须记录的结构性风险：`.src17/system/sepolicy/main/` 与 `.src17/platform/system/sepolicy/main/` **两棵树路径并存**，前者 0 个文件、后者 15 个，是取回过程中的中间目录残留；它不进 178 的计数（只数文件），但**基名检索会同时命中两棵树的目录名**，所以任何"按目录列举"的读数都要写全路径。

**审计二：引文审计（blockquote 每一行能否回到原文）**

判据：报告里每一条以 `>` 开头的引用行，必须能在下列六级里找到归属——

| 级别 | 判据 | 行数 |
| --- | --- | --- |
| 第一级 | 源码整行逐字（索引 `lineno` 列命中） | 691 |
| 第二级 | 页面压平后子串命中 | 21 |
| 第三级 | 同一文件内连续多行拼接命中 | 14 |
| 第四级 | 注释跨行重排后命中（登记重排方式） | 4 |
| 第五级 | 分段检索、各段均命中 | 21 |
| 第六级 | **无法归属** | **0** |

合计 **751 = 691 + 21 + 14 + 4 + 21，第六级 0**。第二级用的压平口径就是卅之二那条四步命令（**删标签 → 折行 → 解实体 → 空格归一并压缩**），草料堆是 35 份逐页压平件按文件名序拼成的那一个 3,011,473 字节的文本（把 35 页当作一整份文档压平得 3,011,441 字节，六级读数完全一致——差的是页边界上的换行）。这一条判据是本轮重跑出来的，不是沿用上一轮读数。

四步里每一步都可以单独抽掉再跑一遍。下表五行全部用同一份引文块、同一个索引重跑，行数可逐行复现：

| 压平口径 | 压平件字节 | 第一级 | 第二级 | 第三级 | 第四级 | 第五级 | 第六级 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 标签→空格、只折行压缩（旧版本节写的三步命令） | 3,034,931 | 691 | 14 | 14 | 4 | 21 | **7** |
| 标签→空格 + 解实体（即备用口径） | 3,023,042 | 691 | 16 | 14 | 4 | 21 | **5** |
| 删标签、不解实体也不归一 | 3,022,479 | 691 | 18 | 14 | 4 | 21 | **3** |
| 删标签 + 解实体、不做 Unicode 空格归一 | 3,011,612 | 691 | 19 | 14 | 4 | 21 | **2** |
| 删标签 + 解实体 + 空格归一并压缩（本节采用） | 3,011,473 | 691 | 21 | 14 | 4 | 21 | **0** |

两个因子各自独立起作用：**换空格比删标签少 5 条**，**不归一比归一少 2 条**，旧命令同时含这两个缺陷所以少 7 条。标签换成空格会造出多余空格（`vulnerability` 与逗号之间、`device / partner` 这类单元格边界），所以它不是"另一种口径"而是严格更差的口径。备用口径仍然要保留的理由在审计三：整行引文里跨相邻单元格的片段在删标签口径下会被并成一格读，跨单元格/跨内联标签的转写要靠它兜住——两种口径承担的方向不同，不能只留一种。

旧命令漏掉的 7 条（即第三、四行两组之并）里，属"换空格"这一因子的 5 条分别是：`Vulnerabilities are grouped under the component they affect…` 一句、`Issues that are not publicly available have an * next to…` 一句、continuous release 与 quarterly bulletins 那句、`OEMs can place the XML file across multiple device partitions…` 一句、`KEY_VENDOR_SUPPLEMENTAL_PATCHES: Aggregates patches from /vendor and /odm partition paths.`；属"空格归一"这一因子的 2 条是 FAQ 里 "split between this bulletin and device / partner security bulletins" ——**原文 `device`／`partner` 两侧是 hair space**，报告照视觉写成普通空格，只有把它归一回空格才命中——与 `Security vulnerabilities that are documented in this security bulletin are required to declare the latest security patch…` 一句。本节把这五行反证留在报告里，是因为"命令写错了"和"命令写对了但没写全"都只会体现在计数上，读者没有这五行就无从判断本报告用的是哪一种。

**审计三：内联跨度审计（正文行内引号里的片段）**

这是本轮新建的一条审计，动机是引文审计只覆盖 `>` 行，而报告里大量引用是**嵌在中文句子里的行内片段**。工具链：跨度抽取 → 草料堆装配 → 批量比对 → 未命中分类（四步各一个 `awk`/`sh` 脚本，与审计一、审计二的脚本一样**不随报告发布**；本节给出的是它们的判据、字节读数与结果，读者要复刻请按卅之二的压平命令与这里的判据自行组装）。

抽取判据：引号内、长度 ≥ 16 字符、纯 ASCII、且含空格的片段。语料仍是**一到廿九章那 20 份、5,884 行**（第卅章不给自己出题）。总数 **7822** 条，剔除过短 4664、含非 ASCII 413、不含空格 1534，得**唯一跨度 948 条**。

草料堆四层（本轮从三层扩成四层，字节数是实测）：

| 层 | 内容 | 字节 |
| --- | --- | --- |
| hA | 索引内容列，逐行、空白规范化（源码整行） | 3,987,869 |
| hB | 同一基名内连续行按单空格拼接（对应第三级） | 4,011,691 |
| hC | 35 份文档页 × **两套压平口径**共 70 份压平件（对应第二级；备用口径管跨单元格/跨内联标签的转写） | 6,034,447 |
| hD | 剥掉 C 注释 `/*`、` * `、`*/` 前缀后按文件拼接（对应第四级） | 3,937,544 |

四层合计 17,971,551 字节，`grep -F -o -f` 一次批量比对，结果 **663 条直接命中、285 条待分类**。hD 是本轮补上的：第一版只堆三层，两条跨行注释（第十二章之三引用的 `a6xx_gpu.c` lowest-nibble 说明、第十七章之二引用的 `remoteproc_sysfs.c` 外部实体说明）被判成未命中——它们本属第四级，缺的只是那一层草料。补上 hD 时还踩到一次自己的坑：注释行归一化时先去前缀后裁空格，导致 `^(\* )+` 匹配不上带前导空格的 ` * `，那一版 hD 白建了 400 万字节却一条没多命中；顺序改对之后批量命中从 653 涨到 663。

285 里按"正文是否声明逐字"再切一刀：正文里声明逐字的跨度共 **133** 条 → **89** 条批量命中 + **10** 条由逐条 `grep -cF -m1` 复核后命中（批量 `grep -F -o -f` 有最长匹配吞并现象：一条短跨度被更长跨度覆盖时不单独报告）→ 真未命中 **34** 条。34 条逐类登记：

| 类别 | 条数 | 处置 |
| --- | --- | --- |
| 把 C 字符串字面量的首尾引号一起抄进跨度（去掉引号即命中） | 6 | 内层串实测在场，引号是本报告转写习惯，不算虚构 |
| 句中 `...` / `X` 形态占位 | 8 | 正文声明为占位，不主张逐字 |
| 表格单元格用 `/` 拼接的复合串 | 4 | 拼接形式是本报告写法，各单元分别命中 |
| 本报告脚本自身的命令与输出行 | 10 | 属脚本自证范畴，由第廿九章验收覆盖 |
| 目录列举与文件名清单 | 2 | 命令实测在场，串来自工具而非来源页 |
| 行定位符用 `/` 拼接（`:175 / :177 / :179`） | 1 | 本报告的定位写法，不对应任何原文 |
| 引用的是 HTML 标记本身（`<title>`、`<h1>403 Forbidden</h1>`） | 2 | 压平时标签已被删除，命中的是原始文件而非压平件 |
| 注释重排后仍带 `/* … */` 外壳 | 1 | 内层文字属第四级，外壳是本报告写法 |

**未命中 34 条全部落在已声明类别内，没有一条是"引了原文里没有的话"。** 未标逐字的 **241** 条（285 条未命中里减去属于逐字子集的 44 条）按"改写/归纳"处理，不进引文判据——这个分界是本节存在的主要理由：不做这一刀，报告无法回答"你引用的那些括号里的东西到底是不是原文"。

**三条审计各自的盲区**（必须与结果同时给出，否则读者会高估审计强度）：锚点审计只证明"行存在"，不证明"引用的就是那一行"；引文审计只覆盖 `>` 行；内联跨度审计的判据是"能在草料堆里找到这段字符"，而草料堆是本报告自建的索引与压平页，**它不能证明该字符属于报告声称的那个文件或页面**。三者合起来的强度是"可复核的形式一致性"，不是"语义正确性"——第廿九章之一末条的方法论约束在此原样适用。

**未引用面登记**：索引里 **134 个基名**（对应 178 个副本），一到廿九章的正文引用落在 **104 个基名**上，其中 100 个解析通过（审计一口径）；余下 **34 个基名从未被正文引用**，按基名数不等于副本数——这 34 个基名在缓存里覆盖 **44 个副本**（多为目录列举与行数复核用的对照件，例如两分支同址验证时的第二份 `remoteproc_debugfs.c`）。它们的存在只说明"取回过"，不构成任何主张——台账把它们列出来，是为了让"取回量"与"引用量"这两个数字不被混用。本节此前把 34 写成"34 个副本"，是把基名数当副本数用了，已改（卅之十七第 16 条）。

### 卅之十四 反证清单：30 条"检索为空"的读数与 5 条非零对照

本报告大量结论的形状是"某处检索为空 ⇒ 该出口不存在"。这类断言最容易被写成不可证伪的漂亮话，因此全部 30 条负向读数集中登记，每条给命令与命中数；并配 5 条**非零对照**，证明同一套命令在同类对象上会给出不为零的结果——否则"恒零的桩"也能冒充这套审计。

**负向读数 Z-01…Z-30**（全部在 `.src17/` 本地副本上执行，路径写全）

| 编号 | 断言 | 命令 | 命中 |
| --- | --- | --- | --- |
| Z-01 | `mtk_scp.c` 无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/remoteproc/mtk_scp.c` | 0 |
| Z-02 | venus（6.12）无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/media/platform/qcom/venus` | 0 |
| Z-03 | venus（6.6）无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android15-6.6/drivers/media/platform/qcom/venus` | 0 |
| Z-04 | venus 目录不涉 rpmsg / rproc / remoteproc | `grep -rniE 'rpmsg\|rproc\|remoteproc' …/android16-6.12/drivers/media/platform/qcom/venus` | 0 |
| Z-05 | samsung（6.12）无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/media/platform/samsung` | 0 |
| Z-06 | samsung（6.6）无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android15-6.6/drivers/media/platform/samsung` | 0 |
| Z-07 | ipu3 无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android15-6.6/drivers/staging/media/ipu3` | 0 |
| Z-08 | atomisp 无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android15-6.6/drivers/staging/media/atomisp` | 0 |
| Z-09 | panfrost 不走 `request_firmware` | `grep -rni request_firmware …/android15-6.6/drivers/gpu/drm/panfrost` | 0 |
| Z-10 | panfrost 无 `.bin` 字面量 | `grep -rnF .bin …/android15-6.6/drivers/gpu/drm/panfrost` | 0 |
| Z-11 | qaic 三词全空 | `grep -rnE 'request_firmware\|MODULE_FIRMWARE\|\.bin' …/android15-6.6/drivers/accel/qaic` | 0 |
| Z-12 | accel 框架不管固件 | `grep -ni firmware …/android15-6.6/drivers/accel/drm_accel.c` | 0 |
| Z-13 | camss 无 firmware 词 | `grep -ni firmware …/android15-6.6/drivers/media/platform/qcom/camss/camss.c`（1823 行） | 0 |
| Z-14 | gsc-core.c 无 firmware 词 | `grep -ni firmware …/exynos-gsc/gsc-core.c`（1324 行） | 0 |
| Z-15 | gsc-core.h 无 firmware 词 | `grep -ni firmware …/exynos-gsc/gsc-core.h`（521 行） | 0 |
| Z-16 | `qcom_q6v5_pas.c` 无专用 sysfs 出口 | `grep -nE 'DEVICE_ATTR\|create_file' …/android16-6.12/drivers/remoteproc/qcom_q6v5_pas.c` | 0 |
| Z-17 | `qcom_common.c` 无专用 sysfs 出口 | `grep -nE 'DEVICE_ATTR\|create_file' …/android15-6.6/drivers/remoteproc/qcom_common.c` | 0 |
| Z-18 | genfs 不点名这四类 | `grep -nE 'remoteproc\|rpmsg\|class-drm\|class-media' …/sepolicy/main/private/genfs_contexts` | 0 |
| Z-19 | shell.te 不给 `kernel:system` | `grep -n 'kernel:system' …/private/shell.te` | 0 |
| Z-20 | shell.te 不给泛型 `sysfs:file` | `grep -n 'sysfs:file' …/private/shell.te` | 0 |
| Z-21 | shell.te 不给 `video_device` | `grep -n 'video_device' …/private/shell.te` | 0 |
| Z-22 | shell.te 不给泛型 `device:chr_file` | `grep -nE 'allow shell device:chr_file' …/private/shell.te` | 0 |
| Z-23 | `private/file_contexts` 无 firmware 词 | `grep -ni firmware …/private/file_contexts`（961 行） | 0 |
| Z-24 | 同文件六个加速器词全空 | `grep -nwE 'accel\|npu\|vpu\|ipu\|imgu\|remoteproc' …/private/file_contexts` | 0 |
| Z-25 | `vendor/file_contexts` 无 firmware 词 | `grep -ni firmware …/sepolicy/main/vendor/file_contexts`（212 行） | 0 |
| Z-26 | service_contexts 无 neuralnetwork | `grep -ni neuralnetwork …/private/service_contexts` | 0 |
| Z-27 | 同文件无 `npu` 词 | `grep -nw npu …/private/service_contexts` | 0 |
| Z-28 | accel + staging/media 无 `npu` 词 | `grep -rIwn npu …/drivers/accel …/drivers/staging/media` | 0 |
| Z-29 | s5p-mfc 目录无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/media/platform/samsung/s5p-mfc` | 0 |
| Z-30 | mediatek/vpu 目录无 `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/media/platform/mediatek/vpu` | 0 |

**非零对照 C-01…C-05**（与上表同族命令，命中数不为零）

| 编号 | 对象 | 命令 | 命中 |
| --- | --- | --- | --- |
| C-01 | panthor（6.12）**有** `MODULE_FIRMWARE` | `grep -rn MODULE_FIRMWARE …/android16-6.12/drivers/gpu/drm/panthor` | 1 |
| C-02 | panthor（上游 v7.3）**有**，且更多 | `grep -rn MODULE_FIRMWARE …/torvalds-v7.3/drivers/gpu/drm/panthor` | 7 |
| C-03 | `a6xx_catalog.c` **有** | `grep -n MODULE_FIRMWARE …/android16-6.12/drivers/gpu/drm/msm/adreno/a6xx_catalog.c` | 10 |
| C-04 | ivpu **有** | `grep -rn MODULE_FIRMWARE …/android15-6.6/drivers/accel/ivpu` | 2 |
| C-05 | shell.te 的 `device:chr_file` **有 6 行**，且每行都是专项类型 | `grep -n device:chr_file …/private/shell.te` | 6 |

C-02 与 C-01 的差（7 vs 1）同时是"版本单列"的理由：同一个驱动在上游标签与 GKI 分支里声明的固件名数量不同，`MODULE_FIRMWARE` 相关结论**不能跨版本沿用**。C-05 是 Z-22 的伴读：它证明"没有泛型 `device:chr_file` 授权"不是因为该文件不含字符设备授权，而是因为这 6 行全部指向专项类型——把负向读数写成"shell 读不到任何字符设备"就越界了。

**这套清单能承担什么、不能承担什么**：每条 Z 只覆盖**该目录集合在该分支的该次检索**。Z-13/Z-14/Z-15 三条写了行数，是因为"0 命中"必须同时给出"文件非空"的证据，否则空文件也满足判据。清单不覆盖任何厂商私有驱动树（本报告的 CodeLinaro 侧只有第十三章那 6 个文件），也不覆盖未在 `.src17/` 落盘的文件——这是第十八章之十那条"遍历 `/sys/class/remoteproc` 不等于得到全部协处理器"的源码侧同型限制。

### 卅之十五 CodeLinaro 下游台账与取回方式

第十三章在正文里承诺"第卅章记录其大小与获取方式"，本节兑现该承诺，并把下游与上游的关系钉死成一行数字。

**台账**（仓库 `git.codelinaro.org/clo/la/kernel/msm-5.4`，分支 `msm-5.4`；本地副本根 `.src17/clo/la/kernel/msm-5.4/msm-5.4/`；大小按本轮 `wc -c` 重测）

| 文件 | 字节 | 行数 | 第十三章引用的读数 |
| --- | --- | --- | --- |
| `drivers/gpu/msm/adreno.c` | 114,213 | 4,356 | `:70` `static u32 get_ucode_version(const u32 *data)`、`:109` `firmware->version = get_ucode_version((u32 *)fw->data);`、`:2499` `static int adreno_prop_ucode_version(struct kgsl_device *device,`、`:2503` `struct kgsl_ucode_version ucode = {`、`:2572` `{ KGSL_PROP_UCODE_VERSION, adreno_prop_ucode_version },` |
| `drivers/gpu/msm/adreno.h` | 64,541 | 1,964 | `:173`（固件槽位声明） |
| `drivers/gpu/msm/adreno_sysfs.c` | 8,586 | 340 | `:272`（属性表附近读数） |
| `drivers/gpu/msm/kgsl.c` | 121,129 | 4,808 | `:1625`、`:1712`、`:1742`（属性读写路径） |
| `drivers/gpu/msm/kgsl_pwrctrl.c` | 61,333 | 2,304 | `:827`、`:1080`、`:1092`（含 `:1080` 的 `rgppath` 侧读数，第十三章之七与第十五章对照引用） |
| `include/uapi/linux/msm_kgsl.h` | 59,459 | 1,911 | `:329` `#define KGSL_PROP_UCODE_VERSION 0x15`、`:342`、`:405` `struct kgsl_ucode_version {` |

**取回方式与 429 处理**：该镜像对同一仓库的**连续并发** raw 请求返回 HTTP 429，因此本轮全部改为**逐文件顺序请求、单请求一次、失败不并发重试**；6 个文件全部 200 落盘，字节数与行数如上。这条限流事实本身要进台账的理由是方法性的：**"厂商下游树取不到"经常是并发策略问题而不是访问控制问题**，与第九章那四类拒因（SPA 壳 / 登录重定向 / 二次验证 / 反爬 403）不同族，复采清单里单列一档。

**下游与上游的关系（第十三章结论的取回侧支撑）**：`adreno.c:70` 与 `:109` 的版本解析逻辑与上游 `drivers/gpu/drm/msm/adreno/` 同源，但**出口完全不同**——下游把版本挂在 `KGSL_PROP_UCODE_VERSION`（`msm_kgsl.h:329`）这个 **ioctl 属性**上（`adreno.c:2572` 是分派表项），上游 adreno/drm 侧没有对应可读接口。`adreno_sysfs.c` 全文只有 340 行、且不导出 ucode 版本节点。⇒ 第十三章那句"版本可读，但没有任何一条标准核验姿势够得着它"的取回侧证据是这三条：`msm_kgsl.h:329` 存在、`adreno.c:2572` 分派存在、`adreno_sysfs.c` 内无对应节点。**注意本报告不写"怎么用这个 ioctl 读"**：属性号与结构体是能力事实（第廿五章表 C 登记），读法属用法，不进正文。

### 卅之十六 复采清单（按拒因分档，逐条给出可执行动作）

第廿二章规则八要求"复采动作按拒因分别给出"。下表每行的**拒因**是本章前文实测到的机制，**动作**是下一次采集该做的事；两者不合并，因为"取不到的机制与修复状态毫无关系"。

| 对象 | 上次结果 | 拒因（分类） | 复采动作 |
| --- | --- | --- | --- |
| `https://security.android.goog/` | 本次**不可达**：经代理 TLS 握手失败（`curl: (35)`），直连 DNS 无法解析（`curl: (6)`） | 传输层（TLS / DNS） | 换网络出口各试一次；本表**不宣称该门户不可用**，只记录本采集环境未取回。取回后用它交叉核对公告页的 CVE 集合，并记录页面自带的更新时间戳 |
| `https://docs.qualcomm.com/<任意路径>` | 302 → 200，正文 `account.qualcomm.com - Sign In`，149,716 / 149,819 字节 | 登录重定向壳 | **逐次记录响应尺寸与 `<title>`**：本项目早前在同路径测到 52,406 字节 `<title>Qualcomm Documentation</title>` 壳，本次已不存在 ⇒ 尺寸阈值判据不可复用，只能按标题 + 跳转后主机名判 |
| `docs.qualcomm.com/bundle/publicresource/FP80-43795-14V_Whitepaper.pdf` | 302 → 403，正文 70 字节 `{"location":"https://myid.qualcomm.com/api/v2/login?state=Direct URL"}` | 鉴权 403（JSON 体重定向） | 需要凭据；本报告不采集登录态内容，登记为长期不可得项，白皮书引用一律不出现 |
| `https://www.qualcomm.com/company/product-security` | 200，10,099 字节 SPA 外壳 | SPA 外壳 | 用无头浏览器执行 JS 后重取；记录渲染后 `<title>` 与文本长度，避免把外壳当正文入库 |
| `https://security.samsungmobile.com/workScope.smsb`、`/securityUpdate.smsb` | 403，正文 146 字节 `<h1>403 Forbidden</h1>` + `nginx` | 反爬 403 | 按厂商公开 API 或人工浏览器路径复采；**禁止**为提高成功率而伪造请求头 |
| Arm 单篇公告 `developer.arm.com/documentation/110627/latest` → 302 → `support.arm.com/documentation/110627/latest` | 需二次验证的文档应用壳 | 二次验证壳 | 入口页 `arm-product-security-center` 可匿名，已入库；单篇登记为不可得，引用止步于产品级 |
| `platform/system/sepolicy` 的 `private/te_macros` | Gitiles 404 | 路径不存在（该文件在此仓库布局下不公开） | 边界项，不再探针；`starting_at_board_api` 展开条件维持不声明 |
| `platform/system/sepolicy` 的 `private/vendor_file_contexts` | Gitiles 404 `NOT_FOUND: Object is not found` | 分支上不存在 | 已由 `vendor/file_contexts`（212 行）替代取回；vendor 侧固件标签确认由厂商补 |
| `private/device.te` | Gitiles 404 | 分支上不存在 | 已由 `public/device.te`（143 行）替代；`video_device` / `rpmsg_device` 读数改挂 public 侧 |
| NNAPI 文档页（`.sb17-nnapi.html`） | `<title>` 是 404 页（`Android Developers` 站点壳），238,985 字节 | 路径形态失效 | 需先取回索引页再按站内链接定位当前路径；**取回前本报告不引用任何 NNAPI 页原句**（登记见卅之二十一） |
| `.../bulletin/2025-10-01`、`2025-11-01`、`2025-12-01` | 404 | 该期不存在于此路径形态 | 不重试、不换形态试探；台账按"该期无公告页"记录，不写"未披露" |
| CodeLinaro `clo/la/kernel/msm-5.4` | 并发请求触发 429 | 限流 | 逐文件顺序请求，一次一请求（本轮 6 文件已全部落盘） |
| 公告侧 `2026-02-01`、`07-01`、`08-01` | 200，但 3 张 `<table>`、0 CVE、0 `A-` 号 | 页面本身无缺陷表 | 复采时先数 `<table>` 与 `A-` 号再决定是否入库；这三页**不是**取回失败，不能记成厂商未披露 |
| `https://source.android.com/docs/security/bulletin` 索引 | 原始 HTML 只到 2021 年条目 | 需 JS | 用无头浏览器或按 `YYYY-MM-01` 直接构造路径（两种形态并存，见卅之八） |

**一条横向规则**：以上 14 档动作里只有 4 档需要新的凭据或登录态（Qualcomm docs 两条、Arm 单篇、Samsung），其余都是路径、形态或解析问题。台账必须把这两类分开，否则"我没登录"与"厂商没披露"会在同一列里长得一模一样——这是本域最常见的一种自证失败。

### 卅之十七 本轮修订掉的引用缺陷清单

第廿九章验收与本轮第卅章写作过程中，报告的引用被查出并改掉了下列缺陷。**每条都写"原先怎么说、实测怎么说、改了哪一章"**，不写成"已修正"了事——本节的唯一用途是让读者知道这些位置曾经错过。

1. **`CSF_IFACE_VERSION` 编码宏的文件归属**（第十一、廿一章）：原写"编码在 `:192-196`"紧跟 `panthor_fw.h:193-202`，读作同一个文件。实测该宏在 `panthor_fw.c:192`，`.h` 内检索 0 命中。已补文件名。
2. **Venus 版本读数出口的行号**（第十九、廿一章）：原写"唯一出口是 `dev_dbg`（`hfi_msgs.c:346`、`:359-360`）"。实测 `:346` 是 dispatch、`:359-360` 属 `hfi_sys_rel_resource_done()`；`dev_dbg` 在 `:322-323`，另有 SMEM 写入 `:323-330`。已改为两条出口并列，"唯一"一词删除。
3. **`/dev/video*` 的类型行**（廿一章）：原写 `device.te:75-77`，实测那是 `rpmsg_device` 三行块；`video_device` 在 `:47`。已改。
4. **`driver_override` 的 store 行**（廿五章）：原写 `rpmsg_core.c:425-427`；实测签名跨 `:424-427`（6.12）/ 起于 `:422`（6.6），`DEVICE_ATTR_RW` 分别在 `:449` / `:447`。已按分支分列。
5. **脚本 1 里 P-04 的策略出处**（第廿八章装配、第廿九章验收当场发现）：原挂 `file_contexts:423`，该句讲的是 vendor 文件标签，与 P-04 判定的"能否创建 unlabeled 文件"无关。已改挂 `domain.te:616-617`。
6. **脚本 1 里 P-19 / P-20 的策略出处**：两者都挂 `shell.te:463`（`ion_device`），与 `video_device` / `device` 两个臂的实际判据不符。已按 pid 分派为 `file_contexts:221`（`/dev/video[0-9]* → video_device`）与 `file_contexts:97`（`/dev(/.*)? → device`）。**第二处违反的是第廿章 P-23 那行自己写的禁令**——"出处必须与臂判据同型"，报告自己的规则被报告的脚本违反，这条记录的价值全在这里。
7. **`CVE-2026-28581` 的双现位置**（第卅章之前的 `.w17srcD.md` D-9 段）：曾把它写成"在 `Media Framework components` 表出现两次"。实测两处分别在**平台缺陷表**（HTML 第 5575 行，`A-479515555`）与 `google-play-system-updates` 之下的**缓解/特性分组表**（第 5753 行，`<td>Telephonycore</td>` 一格 `CVE-2026-28581, CVE-2026-49881`），两处都不是 Media Framework 表。已按偏移 + 行号 + 表列顺序重写。
8. **`qcom_glink_native.c` 的"本树不可达"**：曾登记为取回缺口。拒因是自己的路径假设（写成 `drivers/soc/qcom/`），文件在 `drivers/rpmsg/`，1,999 行已取回，第十八章之六行号重测通过。已销账（卅之十一）。
9. **"同尺寸壳 52,406 字节"这一判据本身**：第九章与第十五章曾用它当 Qualcomm docs 侧的识别特征。本轮重测该壳**已不存在**，同路径返回 149,716 / 149,819 字节的登录页 ⇒ 四条正文改成"同一路径的壳会变，复采必须逐次记录尺寸与 `<title>`"。**判据从常数降级为"必须逐次记录"，这是本轮唯一一处把结论强度往回改的地方。**
10. **五处把两段原文用省略号接成一句的引用**（第廿三章 D-9 段与第廿八章 FAQ 引用）：原写法让读者以为是一句连续原文。已拆成两条引用并各自给行号，改判为第五级（分段检索、各段均命中），不再冒充第一级。
11. **公告侧表数口径混用**：卅之十的"4 列厂商表张数"（2026-09 为 8）与第廿三章的"厂商自定 severity 表数"（同页为 6）一度被当作同一计数。已在卅之十写明两者按不同命令计数、不可互相替换。
12. **`Android Runtime` / `Android runtime` 大小写**：跨期台账按标题主键合并时裂成两行。已登记为 D-12 的独立发现，并在台账里保留两个字符串而不做归一化。
13. **`vendor/sepolicy` 本地副本的两棵树并存**：`.src17/platform/system/sepolicy/main/`（15 文件）与 `.src17/system/sepolicy/main/`（0 文件）同时存在。空目录是取回中间产物，不进计数；已登记，并据此把"按目录列举的读数必须写全路径"写进卅之十三。

下列八条是**第卅章自己**在本轮被自己的判据抓住的，形状与前十三条不同：前十三条是"引用指错了地方"，这八条是"本章登记的读数与命令，按其自身口径跑不出本章写的数"。既然本章的判据是"取不到原句即为伪造"，那么"复现不出来的读数"就是本章版本的同类缺陷。

14. **压平命令写错，本节台账无法自复现**（卅之二、卅之十三审计二、FAQ 问 4 与问 6）：三处都把页面压平写成 `sed -e 's/<[^>]*>/ /g' | tr '\n' ' ' | tr -s ' \t' ' '` 的"标签换空格 + 折行 + 压缩"三步。用这条命令重跑本章记录的引文审计，得 **第二级 14、第六级 7**，与本章宣称的"第二级 21、第六级 0"不符。逐因子重测（五行全量读数见审计二的反证表）确认命令有两处独立缺陷：标签必须**删掉**（换成空格会在 `vulnerability` 与逗号之间造出多余空格，单这一步值 5 条），实体解完还须把 **U+200A / U+2009 / U+00A0 归一成普通空格**（`device / partner` 这类原文用的正是 hair space，单这一步值 2 条）。五种口径（含本节采用的那一种）的读数都写进了卅之二与审计二的反证表，判据本身不变。
15. **跨度审计的草料堆少一层，把两条第四级判成未命中**（卅之十三审计三）：第一版草料堆只有"逐行 / 同文件拼接 / 页面压平"三层，第十二章之三与第十七章之二各自引用过的那两条跨行注释拼接后落在第四级，因此被记为未命中。补第四层时又踩一次：归一化先剥 `/*`、` * ` 前缀再裁首尾空格，而实际行是"制表符 + ` * `"，前缀根本没被剥掉——那一版堆出 3,954,500 字节却一条都没多命中。顺序改对后批量命中 653 → 663，真未命中 36 → 34。
16. **基名数当副本数用**（卅之十三"未引用面登记"）：原写"34 个取回后从未被正文引用的副本"。实测 34 是**基名**数，这 34 个基名在缓存里覆盖 **44 个副本**；同一节又把 86/67/15/6/2/2（按仓库与分支目录的分桶）写成"按副本数 1/2/3/4/5/6 分布"，真实分布是 1 份 98 个基名、2 份 33、3 份 1、5 份 1、6 份 1，没有 4 份那一档。两处口径混用都已改正。
17. **锚点审计的未解析项归因写少了一半**（卅之十三审计一）：原写"四个未解析项全部是脚本字符串里的示例值，`.w17s.md:168`、`:184`、`:207` 三处……与同一形态的第四处"。实测该形态在 `.w17s.md` 的 **144、168、184、195、207、297、356** 共七行、**9 处**令牌；而且"示例值"这个说法也不准——它们是 Shell 的 `${VAR:-file:line}` 默认值写法被正则读成锚点。另外本章把 21 份草稿一起跑之后多出 `file.c` 与 `name.te` 两个未解析基名，来源正是审计一描述抽取规则时自己写的例子——自指引，本章保留而不是把本章剔出语料。
18. **文档页台账过期**（卅之二）：原记 "27 份 / 压平后 4,478,075 字节"，实测目录里是 **35 份 HTML、6,946,603 字节**，删标签压平 3,011,473 字节；差额是本轮补取的 6 期 Pixel 公告、Vulkan 实现指南、`nnapi` 与厂商实现页等。审计三的草料堆此前还只覆盖 25 页（另 10 页没进压平目录），并误把一份全量合并件当第 26 份重复计入。
19. **本节标题用词与正文不符**（卅之一）：标题写"三层命中口径"，正文表列的是六层。已改为"六级命中口径"；"三层"是内联跨度审计旧版草料堆的层数，被误当成命中口径写进了标题。
20. **反证表把读数记在了错误的命令名下**（卅之二第 57 行、卅之十三审计二旧表）：旧表把 **第二级 16、第六级 5** 这一组读数标在"标签→空格（旧版本节写的命令）"名下。逐因子重跑证明这组读数属于"标签→空格 **且已解实体**"的备用口径；旧命令原样（换空格、不解实体、不归一）的读数是 **14 / 7**。旧表另一行"删标签但不做 Unicode 空格归一 19 / 2"实测成立、予以保留，但旧表缺"删标签且完全不归一"的 **18 / 3** 与旧命令的 **14 / 7** 两组，所以两个因子各自值几条无法从旧表读出。现表五行逐行重跑、并给每一行标了压平件字节数。
21. **互链自抽命令比正文多数一条**（本章末"四、站内互链"）：原写"本轮结果 11 条"，与随后分组列出的 11 个页面文件名一致，但把那一节印出来的命令原样跑，得到的是 **12 条**——多出来的一条正是同一节开头当作站点基址写出的那个根 URL，它同样被那条正则抓走（`[a-zA-Z0-9./_-]*` 允许零长度，所以基址后面不接文件名也能匹配）。现已把该处读数改为 12 条并写明"剔掉基址才是下面 11 个页面名"。这是本章第二次遇到"命令把本章自己写进去的句子当语料"（第一次是卅之十三审计一的 `file.c` 与 `name.te`），处置相同：登记，而不是把本章剔出语料。

**这份清单不覆盖的部分**：第廿九章已登记的"13 处正文锚点补全"与"2 处脚本改动"属于验收记录，不在此重复；本节只列**引用错了指向**的缺陷（1–13）与**本章读数无法按本章命令复现**的缺陷（14–21）。清单里 1、2、3、4、7、8、9、11、12、13 十条的实测证据都能在本章前文找到对应行，5、6、10 三处的证据在第廿九章与第廿八章的验收小节里——这三处是本次第卅章写作期间**无法用源码复核、只能用文本比对复核**的修订，强度较低，读者按第廿九章的抽回命令自行验证。14–21 八条的证据全在本节内：每条都给了旧读数、新读数与造成差异的那一步命令，不需要外部文件即可核对。

### 卅之十八 D-16：`drivers/remoteproc/` 目录清单实测（本轮新增）

这一轮补取回的直接动因：第十七章之四与第廿九章都要用"全树 remoteproc 驱动里谁实现了 `.attach`"这个口径，而**它此前是一个没有分母的说法**。本轮不打算把 36 个文件全取回来（那不会改变任何一条结论，只会增加副本），只把**分母本身**取成可复核的读数。

取回命令逐字（Gitiles 目录清单形态，与文件正文的 `?format=TEXT` 是两种用法）：

```
curl -sS -m 45 -x http://127.0.0.1:10808 \
  "https://android.googlesource.com/kernel/common/+/refs/heads/android16-6.12/drivers/remoteproc/?format=JSON"
```

**解析前置步骤**：响应首三字符是 Gitiles 的 XSSI 防护前缀 `)]}'`，紧跟一个换行，**不剥掉就不是合法 JSON**。本报告用 `grep -o '"name": "[^"]*\.c"'` 直接取基名，不整体解析 JSON——这是刻意的：它把"清单里有几个 `.c`"这一条读数变成一条读者可以当场重跑的管道，代价是拿不到 mode/id 字段（本轮不需要）。

实测结果（`kernel/common`，两分支各一次）：

- `android16-6.12`：JSON 条目 49 个，`"type": "tree"` **0** 个（该目录**无子目录**），`"type": "blob"` 49 个 = **36 个 `.c`** + 11 个 `.h` + `Kconfig` + `Makefile`；commit `6fcf0e953fbc9bc5a04aee6676b667188999fa90`。
- `android15-6.6`：48 个条目、**35 个 `.c`**；commit `b53c890fbfab50201653187dbedcc2755a739507`。
- 两分支差集（`diff` 逐字输出 `32a33`）：只有 `ti_k3_m4_remoteproc.c` 一个文件是 6.12 有、6.6 无。

⇒ **本报告凡引用这个分母，必须写成"6.12 的 36 个 / 6.6 的 35 个"，不能写"全树 36 个"**。第十七章之四那句原先写的是"全树 36 个 remoteproc 驱动"，本轮已按实测改写，并补上口径警告：36 是**目录内 `.c` 文件数**，不是 36 个 `rproc_ops` 实例。

36 个 `.c` 基名逐字（`.x17-rproc-c.txt`，这就是分母的全部证据，读者可与上面的 `grep` 输出比对）：`da8xx_remoteproc.c`、`imx_dsp_rproc.c`、`imx_rproc.c`、`ingenic_rproc.c`、`keystone_remoteproc.c`、`meson_mx_ao_arc.c`、`mtk_scp.c`、`mtk_scp_ipi.c`、`omap_remoteproc.c`、`pru_rproc.c`、`qcom_common.c`、`qcom_pil_info.c`、`qcom_q6v5.c`、`qcom_q6v5_adsp.c`、`qcom_q6v5_mss.c`、`qcom_q6v5_pas.c`、`qcom_q6v5_wcss.c`、`qcom_sysmon.c`、`qcom_wcnss.c`、`qcom_wcnss_iris.c`、`rcar_rproc.c`、`remoteproc_cdev.c`、`remoteproc_core.c`、`remoteproc_coredump.c`、`remoteproc_debugfs.c`、`remoteproc_elf_loader.c`、`remoteproc_sysfs.c`、`remoteproc_virtio.c`、`st_remoteproc.c`、`st_slim_rproc.c`、`stm32_rproc.c`、`ti_k3_dsp_remoteproc.c`、`ti_k3_m4_remoteproc.c`、`ti_k3_r5_remoteproc.c`、`wkup_m3_rproc.c`、`xlnx_r5_remoteproc.c`。

其中至少 `remoteproc_core.c`、`remoteproc_elf_loader.c`、`remoteproc_virtio.c`、`remoteproc_cdev.c`、`remoteproc_sysfs.c`、`remoteproc_debugfs.c`、`remoteproc_coredump.c`、`qcom_common.c`、`qcom_pil_info.c`、`qcom_sysmon.c`、`qcom_wcnss_iris.c`、`mtk_scp_ipi.c` 不是"注册 `rproc_ops` 的厂商驱动"（`qcom_q6v5*.c` 那几个才是），所以**按 36 当分母会把"驱动数"算多**。本报告不给出"多少个驱动实现了 `.attach`"这个数，只保留边界项（卅之二十 B-13）。

**本次缓存里实际持有的该目录文件**：13 个（`android15-6.6` 9 个：`mtk_common.h`、`mtk_scp.c`、`qcom_common.c`、`qcom_pil_info.c`、`qcom_q6v5_pas.c`、`remoteproc_cdev.c`、`remoteproc_core.c`、`remoteproc_debugfs.c`、`remoteproc_sysfs.c`；`android16-6.12` 4 个：`mtk_scp.c`、`qcom_q6v5_pas.c`、`remoteproc_debugfs.c`、`remoteproc_sysfs.c`）。

**`.attach` 在这 13 个文件里的命中集合，逐字**：`grep -rc "\.attach" .src17` → 只有三个文件命中：

- `.src17/kernel/common/android15-6.6/drivers/remoteproc/mtk_scp.c` 2 处、`android16-6.12/drivers/remoteproc/mtk_scp.c` 2 处，全是注释行。6.12 副本的两条在 `:1555` 与 `:1572`，逐字 ` * Note: \`scp_ops\` doesn't implement .attach() callback, hence`（第十七章之九引的就是这一句，两级命中：既是注释原文，也是"该驱动自述未实现"的读数）。
- `.src17/kernel/common/android15-6.6/drivers/misc/fastrpc.c:797` 一条 `.attach = fastrpc_dma_buf_attach,`。**这条是同名干扰，必须显式排除**：它属于 `dma_buf_attach_operations`，与 `rproc_ops.attach` 无关。登记它的价值在于给出一条通用纪律：**按 `.attach` 这类"回调成员名"做全树检索时，成员名在不同 `struct` 里复用是常态，命中数不能直接当"实现者数"**——这是卅之十三"基名索引只解决定位、不解决语义"的又一个具体失效例。

### 卅之十九 D-17：六个案例编号的补取回，与一处检索范围失误

卅之八末尾那条"六个编号未取回"（原写"五个"，实际列举为六个：`CVE-2025-54957`、`CVE-2025-48613`、`CVE-2025-20696`、`CVE-2026-56952`、`CVE-2026-0198`、`CVE-2026-55307`）**本轮全部销账**。

失误的准确形状：第廿四章写作时用的是 `grep -l <编号> <四页副本>`，四页是 2026-09、2026-06、2025-09、2026-10。当时的正确结论只有"这四页上没有"，但台账把它写成了"未取回"。补取回用的命令是**对全部副本**的一条：

```
grep -l -- "CVE-2025-54957" .sb*.html
```

`CVE-2025-20696` 的命中文件是 `.sb-2025-09-01.html`——**当轮已经持有、只是没被读到的那一页**（卅之八登的 2025-09 行级读数是节选，只列了 Arm/Imagination/Qualcomm 三张表的行）。⇒ 这条失误的性质不是"资料不可得"，而是"检索范围写窄了却按不可得登记"。报告的纪律因此补一条：**"未取回"必须写明检索集合，否则它就是"未检索"的别名。**

六个编号的逐字行级读数（提取方法沿用卅之八：`sed -e 's/<[^>]*>/\n/g' <页> | grep -v "^[[:space:]]*$"` 生成令牌流，命中行起取到 Subcomponent；表所属由 `head -c <偏移> | grep -o '<h[23] id="[^"]*"'` 回查最近的锚点）：

- `.sb-2026-01-01.html`，编号偏移 145,517，最近锚点 `h3 id="dolby"`（其上还有 `h2 id="2026-01-05-security-patch-level-vulnerability-details"`）：`CVE-2025-54957 | A-438955204 | * | Critical | DD+ Codec`。
- `.sb-2026-03-01.html`，偏移 179,483，最近锚点 `h3 id="misc-oem"`（前一个是 `h3 id="mediatek-components"`）：`CVE-2025-48613 | A-416491056 | * | High | VBMeta`。
- `.sb-2025-09-01.html`，偏移 172,670，最近锚点 `h3 id="MediaTek-components"`（前一个是 `h3 id="Imagination-Technologies"`）：`CVE-2025-20696 | A-421781778 | M-ALPS09915215 | * | High | DA`。
- Pixel 公告 `https://source.android.com/docs/security/bulletin/pixel/2026/2026-10-01` → `.sb17-pixel-2026-10.html`，185,771 字节，`Last updated 2026-10-06 UTC`，最近锚点 `h3 id="pixel"`（前一个 `h3 id="kernel-components"`）：`CVE-2026-56952 | A-508697411 | * | EoP | Critical | GDMC`（偏移 154,879）、`CVE-2026-55307 | A-511809981 | * | ID | Critical | GSA`（155,108）、`CVE-2026-0198 | A-508717360 | * | ID | High | GDMC`（155,335）。

三条新读数，每条都会改变正文：

1. **Pixel 公告页的表有 `Type` 列**（`EoP`/`ID`），列序 `CVE | References | * | Type | Severity | Subcomponent`；月度 AOSP 公告的**厂商表**没有 `Type` 列。⇒ 第五章与第廿四章原来写的"公告不提供缺陷类别"口径过宽，必须限定为"月度 AOSP 页的厂商表不提供"（卅之二十 B-01 已按此改写）。
2. **Pixel 有自己的公告树**，路径形态 `/docs/security/bulletin/pixel/<年>/<年-月-日>`，与月度页 `/docs/security/bulletin/<年>/<年-月-日>` 并列；索引页 `/docs/security/bulletin/pixel` 254,476 字节，站内链接同时给出 `.html?hl=<lang>` 变体。`CVE-2026-56952`、`0198`、`55307` **只在 Pixel 页**，在任何月度页上检索都是 0——第廿四章"到达通道"三分法里那格"必须走厂商/自研专页"现在有了一条实名样本。
3. **Pixel 页 `<title>` 里的竖线是实体**：逐字 `Pixel Update Bulletin—October 2026 &nbsp;|&nbsp; Security &nbsp;|&nbsp; Android Open Source Project`（还包含一个 U+2014 长破折号）。⇒ 卅之十二的"必须先实体解码再比对"这条口径，在标题层面也成立，不只是在正文路径串上成立。

本轮同时把第 5 章那张取值表里四处没有出处的样本补齐（改动逐字）：`DD+ Codec` 行加"（2026-01 页 Dolby 表）"；`VBMeta` 行加"（2026-03 页，`High`）"；`preloader`、`DA` 行把原先写作 `CVE-2026-…` 的**占位串**换成实测的 `CVE-2026-20435`（2026-06 页 MediaTek 表，逐字 `CVE-2026-20435 | A-493235107 | M-ALPS10607099 | * | High | preloader`）；`GDMC`、`GSA` 行加"（三条都在 Pixel 单独公告 2026-10 页，不在月度 AOSP 页）"并给 `CVE-2026-55307` 补上 `Critical`。

**一条顺带发现的新形状**：`CVE-2026-20497` 一行里带**两条** `M-ALPS` 引用——逐字令牌 `CVE-2026-20497 | A-527617087 | M-ALPS10965550 | * | M-ALPS11393405 | * | High | geniezone`（2026-09 页，`grep -n` 令牌行 +14 行窗口内取全）。这与卅之八登的"两条 CVE 共用一条 `M-` 号"（`CVE-2026-20501`/`20502` 共 `M-ALPS11262030`）方向相反：**同一行内一号对多号**。⇒ "引用号与缺陷一一对应"这个假设在两个方向上都被同一份数据证伪过，第廿三章主键误传清单据此再加一条。
### 卅之二十 未公开文档化边界清单（B-01…B-20，逐条给"实测到哪一步为止"）

这一节的判据只有一条：**每一条都必须写清"本报告实测到的边界在哪一行/哪一次检索为止"，而不是只写"这不知道"**。第 5 章、第十七章、第廿九章正文里所有"列入第卅章边界清单"的指涉，都在下面编号；其中第 3 条对应第五章之三那句"`generally` 是限定词"，编号按正文承诺固定。

| # | 不能断言的东西 | 实测到哪一步为止 | 正文位置 |
| --- | --- | --- | --- |
| B-01 | 公告侧提供缺陷类别 | 月度 AOSP 页六张厂商表逐张核过：列序是 `CVE \| References \| Severity \| Subcomponent`，**无 `Type` 列**；平台表有（第廿四章逐字行读数）。反向证据本轮补上：Pixel 公告页的表**有** `Type` 列（卅之十九新读数 1），所以这条只能限定为"月度页厂商表不提供"，不能写成"公告不提供" | 第五章之一、第廿四章之二 |
| B-02 | 公告 Subcomponent 词与内核对象之间存在权威对照表 | 本报告在公告页、`source.android.com` 文档页与厂商入口页里检索不到任何"词 ↔ 驱动/节点"映射；三个实测原因逐字登记在第十六章之八（截断宽度 8 / 16 / 32 字节 + 自由文本 + 大小写两套） | 第十六章之八、第廿一章之二 |
| B-03 | "闭源修复一定在 OTA 里" | 星号句原文只到 `generally` 且只说 `for Pixel devices`（逐字见卅之八）。本轮再补一条压平口径证据：该句在原始 HTML 里以 `Google Developer site .` 结尾（句点被链接标签隔开，压平后多一个空格），**按整句单空格匹配会假失配** ⇒ 属第五级分段命中 | 第五章之三 |
| B-04 | 设备侧是否存在把 remoteproc sysfs 降成只读的厂商补丁 | 使用点实测 `remoteproc_sysfs.c:236`（6.6 与 6.12 同址）；字段声明与注释只在 6.6 副本：`include/linux/remoteproc.h:581` `bool sysfs_read_only;`、`:540` ` * @sysfs_read_only: flag to make remoteproc sysfs files read only`。全缓存 `grep -rn "sysfs_read_only"` **共 4 处、无一处赋值** ⇒ 能断言的只有"这两个 GKI 分支里没有赋值者"，不能断言"厂商也没设" | 第十七章之六、第廿五章表 A |
| B-05 | `protected_service` 的强制后果 | 属性标记在 `public/service.te` 实测 64 处（`hal_camera_service:311` 带、`hal_neuralnetworks_service:343` 不带）；对应 neverallow 不在取回的六个策略文件里 | 第廿五章表 D、第廿九章之十 |
| B-06 | `starting_at_board_api` 的展开条件 | `public/te_macros` 全文 1105 行检索 0 命中；`private/te_macros` Gitiles 404（卅之十六）。使用点逐字在 `service.te:21/:74/:165/:182/:292/:329-331/:339-341/:378`、`genfs_contexts:172-174` | 第廿九章之十、第廿六章 |
| B-07 | `/vendor/firmware_mnt` 等厂商路径的真实标签 | AOSP 侧 `file_contexts` 只有 961 行且 `firmware` 关键词 0 命中（卅之六）；`private/vendor_file_contexts` 404，替代取回的 `vendor/file_contexts` 212 行不含该路径 ⇒ 厂商 vendor sepolicy 不可得 | 第十九章之一、第二十章 P-05 |
| B-08 | `dmesg_restrict` / `kptr_restrict` 的当前取值 | `genfs_contexts:53`、`:57` 只给出类型 `proc_security`；取值由厂商 init 决定，本报告不采任何厂商默认值 | 第廿九章之十第 12 行、第二十章 P-24 |
| B-09 | `ls -Z` 在全部厂商构建上可达 | 取标签只需 `getattr` 的两条授权实测在 `domain.te:188`、`shell.te:424`；但标签本身依赖 B-07，故脚本把取不到记成 `label_unreadable` 而不是"无标签" | 第二十章 P-05、第廿七章之十限制 1 |
| B-10 | rpmsg 字符设备能否被 shell 打开 | `private/shell.te` 全文 `device:chr_file` **6 处**，逐字为 `:4`（`uhid_device rw`）、`:339`（`tty_device rw`）、`:340`（`console_device rw`）、`:343`（`input_device r`）、`:463`（`ion_device rw`）、`:546`（neverallow `input_device`）；`sysfs:file` 授权 **0 处** ⇒ GKI 生成的 `rpmsg%d`/`rpmsg_ctrl%d` 落 `file_contexts:97` 的兜底 `device` 类型，AOSP 层无任何一条授权 ⇒ 结论只能是"由厂商策略决定" | 第十八章之七、第二十章预期失败表 |
| B-11 | `KGSL_PROP_UCODE_VERSION` 的实际可达性 | 常量与派发点实测逐字：`include/uapi/linux/msm_kgsl.h:329` `#define KGSL_PROP_UCODE_VERSION 0x15`、`drivers/gpu/msm/adreno.c:2572` `{ KGSL_PROP_UCODE_VERSION, adreno_prop_ucode_version },`（CodeLinaro `msm-5.4`）。可达与否取决于 `/dev/kgsl-3d0` 的标签与厂商 sepolicy（B-07），**本报告不写调用姿势** | 第十三章之四、第廿五章表 C |
| B-12 | "这台设备有没有 zap 微码" | `adreno_gpu.c:28` `static bool zap_available = true;`、`:44` 与 `:50` `zap_available = false;`，`-EOPNOTSUPP` 静默路径在 `:162-163`；该位是进程内全局，**无任何 sysfs/debugfs/属性出口** | 第十二章之四 |
| B-13 | 全树 remoteproc 驱动中 `.attach` 的实现者集合 | 分母本轮已实测（卅之十八：6.12 是 36 个 `.c`、6.6 是 35 个，差一个 `ti_k3_m4_remoteproc.c`）；持有副本只覆盖 13 个文件，其中 `rproc_ops` 的 `.attach` **0 处**，`mtk_scp.c:1555/:1572` 的两句是驱动自述未实现。`fastrpc.c:797` 那条同名成员属 `dma_buf_attach_operations`，已排除 | 第十七章之四 |
| B-14 | 本域是否存在一条能到 L3 的公开路径 | L0–L3 判级实测最高到 L2（第廿六章之一）；L3 需要"厂商公开认定下限 + 设备侧可比对字段"同时成立，而 FT3 的反证条件证明公告侧给不出版本边界（第廿一章之三），因此本域**记不出一条 L3** | 第廿六章之二 |
| B-15 | 出现 `0444` 时背后是哪条厂商补丁 | 源码声明的降权条件实测在 `remoteproc_sysfs.c:236-239` 四条 + `.is_visible` 注册 `:256`；设备侧若读到与本树不同的模式，只能标 `degraded_by_vendor`，补丁本体不可取回 | 第十七章之六、第廿八章脚本 2 |
| B-16 | `geniezone`、`trusted_mem` 这类厂商词的内核实体 | 内核侧无同名实体（全缓存检索），`mt8183.dtsi:1036/:1041/:1044/:1045` 只能作"SCP 通路存在"的锚点，且同节点 `status = "disabled"` ⇒ 不得反推保留内存大小/地址 | 第十七章之九、卅之十二 |
| B-17 | 上游 `v7.3` 的 panthor 形态与 GKI 行号的关系 | `panthor_fw_has_glb_state` `:326-331`、`panthor_fw_has_64bit_ep_req` `:333-338` 只在上游标签侧成立，两个 GKI 分支副本里没有这两个函数 ⇒ 行号不得跨版本引用（卅之九末条） | 第廿一章之五 |
| B-18 | 下游 KGSL 各代际的一致性 | 只取回 `clo/la/kernel/msm-5.4` 一个分支的 6 个文件（卅之十五台账）；`msm-4.19`/`msm-6.1` 未采 ⇒ 属性表读数不外推 | 第十三章之一 |
| B-19 | 厂商闭源组件的镜像内容与版本 | `Qualcomm closed-source components` 表 11 行只给 `A-` + `*`、无 `QC-CR#`（第廿四章之六），镜像本体不可公开取回 ⇒ 本域公开侧信息量最少的一批 | 第廿四章之六、第廿五章 |
| B-20 | NNAPI 文档页的原句 | 该页本轮仍未取回（卅之十六），报告中不引用任何 NNAPI 页句子；需要时先按站内索引定位当前路径 | 第廿二章规则八 |

**这份清单不覆盖的东西**：卅之十九销账的六个案例编号（已取回）、第廿九章之交回项里属于"本报告自己没实现"的那几条（属能力缺口而非文档边界，去卅之二十三）。两者的区别必须保住：**边界是"公开处没有"，缺口是"我没做"**。

### 卅之二十一 仍未取回清单（本轮销账后的剩余，逐条注明"无引用依赖"）

判据：下面每一项都**没有任何一条正文引用指向它的内容**，所以报告不会因为它们的缺失而引用到未经核验的句子。若将来引用，必须先补取回再改正文。

- `developer.android.com/ndk/guides/software.nnapi/*`（NNAPI 页）：`.sb17-nnapi.html` 238,985 字节是 404 壳。
- `source.android.com` 厂商实现指南页：`.sb17-vendor-impl.html` 110,154 字节同样是 404 壳。
- `platform/system/sepolicy@main` 的 `private/te_macros`、`private/vendor_file_contexts`、`private/device.te`：Gitiles 均 404，前两个无替代、第三个由 `public/device.te` 替代。
- `docs.qualcomm.com` 深页与白皮书 PDF、`www.qualcomm.com/company/product-security` 正文、`security.samsungmobile.com` 两页、Arm 单篇公告 `documentation/110627`：均需登录/二次验证或反爬 403（拒因分档见卅之十六）。
- `https://security.android.goog/`：本采集环境不可达（经代理 `curl: (35)`、直连 `curl: (6)`），**不宣称门户状态**。
- 月度公告 2025-10/11/12 三期：该路径形态 404，按"该期无公告页"记录。
- `drivers/remoteproc/` 其余 23 个 `.c`、`drivers/gpu/drm/panthor/` 的上游 `v7.3` 版本、CodeLinaro 其他分支：本轮不需要，未采。
- 任何厂商 vendor sepolicy / `file_contexts` 附加文件、`/proc/sys/kernel/dmesg_restrict` 实际值：属 B-07、B-08，公开渠道无。

### 卅之二十二 编号撞号声明（同一前缀在本报告里指两种东西）

这一节存在的唯一原因：报告的编号族里有一组**字面完全相同**的 token，读者复制检索命令时必须靠上下文区分，否则会把本章台账的 `D-10` 当脚本 2 的检查块 `D-10`。

| 前缀 | 一族 | 另一族 | 区分规则 |
| --- | --- | --- | --- |
| `D-` | 第卅章取回台账轮次：`D-1`…`D-17`（本节新增 `D-16`、`D-17`），**不补零** | 脚本 2 的检查块 `did` 列：`D-01`…`D-19` 加 `D-13b`、`D-14b`（共 21 块）与末节 `D-20`，**补零** | `D-01`…`D-09` 与 `D-1`…`D-9` 字面不同，安全；**`D-10`…`D-17` 两段字面完全相同 ⇒ 真冲突**。凡正文写 `D-`，必须紧跟"第卅章"或"脚本 2"限定语；本报告第廿九章与本章均已按此写 |
| `P-` | 第二十章 26 条采集姿势 `P-01`…`P-26` | 第廿七章脚本 1 的 `pid` 列取值（同族，非另一族） | 脚本 1 的 `pid` 值就是 `P-xx`，与第二十章那张表**是同一套编号**，不构成撞号。但三个数各自有口径，别混用：**26** 是姿势总数；脚本 1 侧"24 个字面 pid 各属一个区块 + 2 个由循环分派"（第廿九章之十一条）；脚本 2 侧 21 个块只覆盖 26 条里的 21 条，没有专属块的五条是 P-01、P-03、P-04、P-06、P-18（第廿八章之二末段）。看到 24/21/26 不相等不要当缺项 |
| `V1–V8` | 第十九章之一的八类读数面号 | 脚本 2 `face` 列沿用同一套（跨面块用 `ALL`，声明与服务面横跨 V3/V8 的块用 `DECL`） | 同一套，非撞号；`ALL`/`DECL` 是这两个面号集合之外的两个合法取值，不要读成"面 9/面 10" |
| `K1–K10` | 第廿六章十个运营指标 | — | "功能反证覆盖"不占 K 号（第廿九章之十第 13 行），不要把它读成 K11 |
| `FT1–FT10` / `L0–L3` / `O1–O5` / `F1–F6` | 第廿一章类型学 / 第廿六章判级 / 第廿五章工单去向 / 第三章责任层 | — | 各自唯一，跨章引用时保持章号 |
| `Z-01…Z-30`、`C-01…C-05` | 本章卅之十四的反证与非零对照编号 | — | **只在本章使用**，正文其他章不引用；读者复现时按卅之十四逐字命令执行 |
| `B-01…B-20` | 本章卅之二十边界清单 | — | 第廿六章 O5 的 `risk_accepted` 字段引用它时必须带"第 N 项" |
### 卅之二十三 第廿九章之交回项的落点（13 行逐条销账）

第廿九章之十那张表把 13 条"不能断言的东西"交给本章，但**本章不是把 13 条全部收进边界清单**——收进去反而是错的，因为那 13 条的性质不同。本章的处置分五类，先把分类写清，因为五类的后果不同：

- **边界项（B-xx）**：公开处没有。读者不能要求报告补，厂商补了才算变化。
- **能力缺口（G-xx，本节新增编号族）**：公开处有，本报告没做。这是报告的缺陷，不是世界的缺陷，必须单列，否则读者会把"没采"读成"采不到"。
- **已销账**：本轮补到了来源，条目本身消失。
- **范围声明**：本篇的口径决定不做（真机测试），与取回无关。
- **不转入本章**：它本来就是脚本输出的实测事实，与外部来源无关，转进本章只会造成重复登记。

| 第廿九章之十行号 | 条目（缩写） | 本章落点 | 类型 |
| --- | --- | --- | --- |
| 1 | `driver_override` 当前值 | 本节 G-01 | 能力缺口 |
| 2 | 遍历 `/sys/class/remoteproc` 得全部协处理器 | 本节 G-02 | 能力缺口 |
| 3 | glink 边数与 `rpmsg_ctrl` 节点数比较 | 来源侧卅之十一（D-13）已销；格子侧本节 G-03 | 已销 + 能力缺口 |
| 4 | 值文本参与判定 | 第廿七章脚本契约、第廿九章之八实测；本章不登记 | 不转入 |
| 5 | K6 分子可由脚本输出算出 | 本节 G-04 | 能力缺口 |
| 6 | K8 分子可由脚本输出算出 | 本节 G-05；其 `value_source` 的证据只在卅之三、卅之四、卅之五、卅之十一、卅之十八与 B-04、B-11、B-12 | 能力缺口 |
| 7 | 任何 V7 方向的设备侧结论 | 第廿九章之七三层读数（13 / 13、0 / 13、版本档 3）＋ 本章末"范围声明" | 范围声明 |
| 8 | 分支可达即分支可识别 | 第廿九章之八限制 1（P-26 两支逐字节相同）| 不转入 |
| 9 | `err` 列每行都是错误串 | 第廿九章之八限制 2（P-18 `present` 臂复用第 6 列）| 不转入 |
| 10 | 真机取值分布与"多数设备会怎样" | 本章末"范围声明" | 范围声明 |
| 11 | 判定在语义上正确 | 卅之十三末段"三条审计各自的盲区"＋ 本章末"范围声明" | 范围声明 |
| 12 | `dmesg_restrict` / `kptr_restrict` 当前取值 | **B-08** | 边界项 |
| 13 | 功能反证覆盖是第 11 个指标 | 卅之二十二（`K1–K10` 那行的限定语）＋ 第廿六章 K 表 | 不转入 |

**账要算平**：13 行 = 边界项 1（第 12 行）＋ 能力缺口 4（第 1、2、5、6 行，第 3 行另计一个缺口 G-03 故缺口共 5 条）＋ 已销 1（第 3 行来源侧）＋ 范围声明 3（第 7、10、11 行）＋ 不转入 4（第 4、8、9、13 行）。第 3 行同时占两格，是唯一一条"来源补上了、结论仍然不成立"的条目，这也是本报告最想留下的一条形状：**补来源和补结论是两件事，销账时不能只销一半就宣称完成。**

下面五条缺口逐条给出"缺什么 / 要补它得动什么 / 为什么本轮不动"。**"为什么不动"必须是成本说明，不能是理由包装**——动分母会连带改判级表，这是本轮没有做而读者有权知道的事。

- **G-01 不采 `driver_override`。** 缺的是 rpmsg 设备名读数唯一能成立的前置：第廿一章 FT6 要求名字读数必须有伴读（同一个设备的 override 值），脚本 1 的 26 条姿势里没有这一条。要补它得新增一条姿势（P-27）并把 K2 的固定分母从 26 改成 27，第廿章、第廿六章、第廿九章之四的"26 条全覆盖"三处同时改写。本轮不动，因此任何 rpmsg 名字读数不进 K8 分子。
- **G-02 `no_rproc_face` 的三类枚举未实现。** 第十七章之十承诺过这三类，脚本 2 只输出 `D-19` 那句"要补采先写进第廿章的表"。同样要动 26 的分母，且补上之后"遍历 `/sys/class/remoteproc` 即得全部协处理器"这条默认假设才会被正式否证。本轮不否证它，只登记它未被否证。
- **G-03 glink 边数与 `rpmsg_ctrl` 节点数的比较格子。** 来源已在本轮补回（卅之十一），但两个计数脚本都不产出，比较结论只能停在"形态描述"。补它要在脚本 2 加一个块，而加块意味着第廿九章之九的 42 条输出、55 个 `rec` 调用点、恒 54 行三个不变量全部重测。
- **G-04 K6（链覆盖）分子手填。** 脚本 1 的输出没有"链"这一列，26 个 `pid` 到七条链的映射写在正文而不在输出里，所以 K6 不可从脚本自证。补它等于给脚本 1 加第 7 列，会同时破坏第廿九章之四"列数恒定"的验收判据。
- **G-05 K8（固件版本可读性）分子手填。** 输出没有 `value_source` 列，`fw_reported` / `driver_set` / `userspace_override` 三值只能由源码判断，因此 K8 的每一格都要回到本章的取回记录才能立住。这一条是**五类缺口里唯一"补了也不会自证"的**：列加得出来，值仍然要人来判。

### 卅之二十四 FAQ（只回答"这份记录怎么用"，不重复正文结论）

**问 1：为什么取回记录放在最后一章，而不是第一章？**
因为它的作用域是"复核"，不是"阅读"。放在前面会诱导读者把行数、字节数和命中率当成结论强度；放在最后，读者是在读到某条行号怀疑它时才往后翻。本章自己声明"不产生新结论、不引入新的源码行号"，位置与声明是同一件事。

**问 2：能不能只读本章就对项目下判断？**
不能。本章没有任何一条媒体/加速器缺陷的成因、层级或缓解措施；卅之二十与卅之二十一合起来告诉你"公开侧到哪里为止"，但"到哪里为止"和"东西是什么"是两问。要判断风险，去第廿一章的类型学、第廿五章的资产面与第廿六章的 K 表。

**问 3：源码行号漂移了，我怎么自查？**
三步：先确认锚点串能被同一条正则抽出来（本章采集期用 `.x17-anix.sh`，**它不随报告发布**，读者用一条 `grep -oE '[A-Za-z0-9_.,/-]+\.(c|h|te|mk|bp|py|sh|rc|xml|txt|yaml|yml|dts|dtsi):[0-9]+(-[0-9]+)?'` 即可复刻同一抽取；`file_contexts`、`genfs_contexts`、`service_contexts`、`te_macros`、`property_contexts` 这五个无扩展名文件要另加一条模式，否则整批漏抽——这是本轮实测到的一个抽取盲区，写在这里而不是藏在脚本里）。再用 `grep -n` 在你自己的副本里定位该基名，最后按卅之一的六级判据定级——**只有第 1、2 级可称"逐字"**，第 3、4、5 级是可复原，第 6 级按伪造处理。分支不同造成的漂移（例如 panthor 只在上游 `v7.3` 侧成立）不是错误，属 B-17，按"行号不得跨版本引用"处置。

**问 4：我按你给的原句检索，0 命中，怎么办？**
先怀疑命令，再怀疑报告。四个已实测的假阴性来源：（1）**标签要删掉而不是换成空格**——换成空格会在"文本紧跟内联标签"处多出一个空格，本轮重跑因此少了 5 条页面原句；（2）压平顺序（**先折行、再压缩连续空白**，颠倒会得到双空格）；（3）HTML 实体（`&nbsp;`、`&lt;`、`&gt;`、`&amp;`、`&quot;`、`&#39;` 必须解，且 `&amp;` 放最后一条）；（4）**Unicode 空格**：源码里的 `device / partner` 两侧是 U+200A hair space 而不是普通空格，还有 U+2009 thin space 与 U+00A0，不归一会再少 2 条。另外 U+2014 这类非 ASCII 连接符也常见：报告在缝过引用处一律写出连接符，所以 `X` 或 `…` 两侧要分别检索。全部排除后仍 0 命中，就按第 6 级处理并报回来——六级判据的本轮结果是第 6 级为 0，任何一条新的 0 命中都是本章的缺陷而不是读者的缺陷。

**问 5：两个脚本能直接在我的设备上跑吗？**
能，且它们是本章唯一可执行的东西。它们是只读核验脚本：不写任何 sysfs/debugfs，不装任何东西，不对固件下发起操作；写侧能力在第廿五章只登记名称、位置、八进制模式位与"它承载哪一类信息"，不提供使用姿势。会看到 `Permission denied` 是预期输出之一（脚本把它编成 `denied_log`、`label_unreadable`、`companion_missing` 这类档位），不要把它读成"设备有漏洞"或"厂商关了某项"——B-08 就是专门管这类误读的。请在自己的或明确授权的设备上运行。

**问 6：为什么不解析 DOM，而用 `sed` 压平？**
因为读者要能复刻。压平管线四步（删标签 → 折行 → 解实体 → 空格归一并压缩）全部由 shell 内建与 `sed`/`tr` 完成；DOM 解析需要解释器与库，本采集环境里 `python` 与 `node` 不可用（Node 只能经宿主 Electron 二进制启动，读者环境未必有）。代价也写在报告里：压平会造出双空格、会吞掉标签边界（`<h3 id="…">` 落在表格单元里时会整块消失），还会把相邻表格单元格直接粘住（`Our Reference` 与 `PSP-19` 之间不留空隙），所以卅之十九的行抽取用 `sed -e 's/<[^>]*>/\n/g'` 把标签换成换行再过滤空行，卅之十三的跨度审计另外备一份"标签→空格"压平件当补充草料堆——**这两样都是压平口径的补救，不是另一套真值**。三种口径给出的行数与命中数都不同，引用时必须写明用的哪一种；本章正文与引文审计统一用删标签那一种。

**问 7：CVE 编号能当主键吗？**
能当行标识，不能当"缺陷 ↔ 修复"的键。两个实测形状：一条 `M-ALPS11262030*` 同时被 CVE-2026-20501 与 CVE-2026-20502 引用（卅之八逐字行、第廿三章之七），一条 CVE 行里出现两个厂商引用号（`CVE-2026-20497` 那行，卅之十九）。所以"按引用号统计已修条目数"会低估，"按 CVE 数统计厂商贡献"会高估。另外别用 Subcomponent 词做映射主键，B-02 管的就是这个。

**问 8：为什么有些厂商入口你能取到、有些 403？**
卅之十六按拒因分档写清了：需登录/二次验证的、反爬返回 403 的、本采集环境不可达的（`security.android.goog` 经代理 `curl: (35)`、直连 `curl: (6)`）是三档，**三档都不能推断"该门户是否对外可用"**。可达性差异测的是本采集环境，不是厂商策略。

**问 9：边界项有 20 条，是不是说明这份报告不可信？**
边界条数与可信度不成反比，**未登记的边界**才是问题。每条 B 项都写了"实测到哪一步为止"（哪个文件、哪一行、`grep` 命中几处），所以读者能判断的是"这个不知道是被证明的不知道"。另有卅之十四那 30 条"检索为空"的反证读数配 5 条非零对照——没有非零对照，0 命中无法区分"确实没有"和"检索命令写错了"。

**问 10：我设备上的读数和你给的样例输出不一样，怎么处理？**
先看是不是分支差异（GKI 6.6 与 6.12 的同名文件同址性在 B-04、B-15 里实测过：`remoteproc_sysfs.c:236` 两分支同址，panthor 则不同），再看列是否错位（`san()` 会把竖线/换行/制表符压成空格，第 6 列在 P-18 的 `present` 臂里放的是目录首项而不是错误串），最后才谈厂商差异：模式位比本树更低权限时脚本标 `degraded_by_vendor`，**这一格不进任何 K 分子**，只进台账。拿设备读数去推翻报告样例、或拿样例去覆盖设备读数，都是同一类错误。

### 卅之二十五 参考资料（逐条给规范 URL 与本地副本；没有一条 URL 靠推测）

URL 一律取自页面自己的 `<link rel="canonical" href="…">`，逐条实测列出；**五个返回 404 壳的路径形态也列在这里，因为它们同样是本轮的读数**（判 404 只看 `<title>`，不看尺寸、也不看 canonical 是否存在——404 壳照样带自己的 canonical 标签，这是个可复现的陷阱）。本地副本大小以 `stat -c%s` 计。

**一、Android 公告与平台文档（source.android.com）**

- 公告索引：https://source.android.com/docs/security/bulletin — 副本 `.sbi.html` 156,583 字节，标题 `Android Security and Update Bulletins`。用于第廿四章路径形态与第廿二章计数口径。
- 月度公告 2026 年 1–10 月（十页，逐页结构台账见卅之十）：`https://source.android.com/docs/security/bulletin/2026/2026-MM-01`，MM 取 01…10；副本 `.sb-2026-01-01.html` 177,645 至 `.sb-2026-09-01.html` 230,027 字节，逐份字节数在卅之十。
- 历史期对照：`.../bulletin/2025-09-01`（`.sb-2025-09-01.html` 213,701）、`.../2024-06-01`（193,152）、`.../2021-01-01`（193,914）、`.../2017-01-01`（250,828）。注意这四期的路径形态与 2026 年不同（无 `/2026/` 年目录），这条差异本身就是第廿二章"入口不稳定"的证据。
- Automotive 公告：https://source.android.com/docs/security/bulletin/aaos — `.sb-aaos.html` 205,419 字节。
- 最新公告汇总页：https://source.android.com/docs/whatsnew/latest-security-bulletins — `.sb-wnew.html` 137,674 字节，标题 `Latest security bulletin`。
- Pixel 单独公告：索引 https://source.android.com/docs/security/bulletin/pixel（`.sb17-pixel-index.html` 254,476 字节）与六期 `.../pixel/2026/2026-MM-01`（MM 取 05…10；`.sb17-pixel-2026-10.html` 185,771 字节，页内标注 `Last updated 2026-10-06 UTC`）。**Pixel 公告页的表有 `Type` 列**，与月度页的厂商表列序不同（B-01 的反向证据）。
- 补录安全补丁：https://source.android.com/docs/security/overview/supplemental-security-patches — `.sb17-suppl.html` 147,494 字节。本轮新登记原句（正文未引用，登记作入口口径）："OEMs may want to fix vulnerabilities between releases rather than waiting an entire quarter."；"Effective in 2026, to align with our trunk stable development model and ensure platform stability for the ecosystem, we will publish source code to AOSP in Q2 and Q4."；"Note: Using these XML files is optional. You need them only if you want to report a patched state for a CVE beyond the declared SPL."
- Vulkan 实现指南：https://source.android.com/docs/core/graphics/implement-vulkan — `.sb17-vulkan.html` 213,919 字节；本轮登记原句 "The android-latest-release manifest branch will always reference the most recent release pushed to AOSP."
- **以下四个路径形态返回 404 壳，报告不引用其中任何句子**：`.../docs/security/bulletin/faq`（`.sb-faq.html` 110,132 字节）、`.../docs/security/bulletin/2026-09-01`（`.sb2609.html` 110,139）、`.../docs/security/bulletin/android-bulletin-2026-09-01`（`.sb-android-bulletin-2026-09-01.html` 110,156）、`.../docs/core/architecture/configuration/vendor-impl`（`.sb17-vendor-impl.html` 110,154）；另有 `https://developer.android.com/ndk/guides/nnapi`（`.sb17-nnapi.html` 238,985，developer.android.com 的壳）。五份壳的标题同为 `404 | Page Not Found`，前四份同站点且尺寸互不相同（110,132 / 110,139 / 110,154 / 110,156）——**同站点不同请求路径的 404 壳尺寸不唯一，判 404 不能靠尺寸比对，也不能靠 `rel="canonical"` 是否存在**（这五份都带着指向自己被请求路径的 canonical 标签）。

**二、厂商安全入口**

- Imagination（PowerVR GPU）：https://www.imaginationtech.com/gpu-driver-vulnerabilities/ — `.sb17-imgtec.html` 821,380 字节（本批最大副本）。页自称的逐字原句（同批登记在卅之十二）："This page contains summary details of security vulnerabilities reported on Imagination Technologies graphics drivers."；条目侧的 `PSP-19`、`PP-171174`、`DDK Releases up to and including` 三处读数也在卅之十二，本章不重复。
- MediaTek 入口页：https://www.mediatek.com/product-security-bulletin — `.sb17-mtk.html` 152,778 字节，原句 "The MediaTek Product Security Bulletin shows MediaTek's commitment to keeping providing secure and reliable products."
- MediaTek 2026-10 期：https://www.mediatek.com/product-security-bulletin/october-2026 — `.sb17-mtk-oct.html` 148,502 字节；第九章之二引用的三句（`affecting certain MediaTek chipsets`、`at least two months before publication`、`Common Vulnerability Scoring System version 3.1 (CVSS v3.1)`）全部命中此页；索引页只有一句 commitment 自述，两句的分工在卅之十二。
- Arm：本地副本 `.sb17-arm.html` 120,201 字节，**无 `<title>`、无 canonical**（React 单页应用，正文由脚本注入）。引用只到入口 https://developer.arm.com/arm-product-security-center ，**不引用任何页内句子**，理由与拒因见卅之十六。
- 仍未取回（不得引用）：Qualcomm 深页与 PDF、`www.qualcomm.com/company/product-security` 正文、Samsung 两页、Arm 单篇公告 `documentation/110627`；NNAPI 页只有上面那张 404 壳。清单与"无引用依赖"的逐条判定在卅之二十一。

**三、源码仓库（Gitiles）**

取回命令的通用形态（本轮全部源码副本同法，逐文件）：

`curl -sS -m 45 -x http://127.0.0.1:10808 "https://android.googlesource.com/$R/+/refs/heads/$B/$P/?format=TEXT" | base64 -d`

目录列举同一基址，`$P` 到目录为止、后缀改 `?format=JSON`，响应**首行是 XSSI 防护串 `)]}'`，必须先剥掉再解析**（不剥则 JSON 解析器整体报错）；行号取自剥壳后的原文，不取自 JSON。仓库与 ref 六个组合、本地副本目录、文件数与行索引规模都在卅之二；两个 `torvalds/linux` 组合本轮经镜像取回，副本落在 `.src17/upstream/` 下（目录名与归属复核见卅之二、卅之十三），**上游站点的直接 URL 本章不写，因为它不在本轮取回记录里**。

**四、站内互链（本系列其余各篇）**

本报告集站点（`https://android-security-reports-0e8t10askpz.qoder.website/`）当前有 16 篇已发布页面（这是本轮取回时点的读数，本篇上线后为 17 篇）；本篇正文实际互链 11 篇。下面写的是页面文件名，完整 URL 就是基址加文件名（`.html` 后缀逐字照加），自抽命令：`grep -ho 'https://[a-zA-Z0-9./_-]*android-security-reports[a-zA-Z0-9./_-]*' .w17*.md | sort -u`。该命令本轮结果 **12 条**：11 个页面文件名，外加 1 条基址本身——就是本句开头那个用反引号括起来的站点基址，被同一条 `grep` 一并抓到。这是本章第二例自我指涉（第一例是卅之十三审计一里的 `file.c` 与 `name.te`），处理办法与那里相同：不剔掉本章、而是把多出来的一条登记在案。剔掉基址才是下面这 11 个页面名，`sort -u` 后按字母序；下面按主题分组列出，条数与分组数一致即可核对。

- `Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html`、`Android-Linux-Kernel-Structure-and-Subsystems.html`（内核侧层级）
- `Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html`、`Android-Attack-Surface-and-Vulnerability-Research-Methodology.html`（方法与隔离）
- `Android-Hardware-Security-Components-and-Vendor-Implementations.html`、`Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html`（厂商形态与补丁治理，与第廿四、廿五章重叠）
- `Android-Platform-Code-Organization-and-Build-System-Analysis.html`（源码树组织）
- `Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html`（媒体面远程执行入口与本篇的分工，由第一章的视角声明划开）
- `Android-Wireless-Stack-and-Baseband-Defense-Analysis.html`、`Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html`、`Android-Project-Mainline-and-APEX-Module-Defect-Attribution-Analysis.html`（第 14、15、16 篇，划界三篇）

**未互链的五篇**：`Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html`、`Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html`、`Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html`、`Android-Root-of-Trust-and-Key-Stack-Mechanisms.html`、`Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html`（五份都在站点 `dist/` 里实测存在，共 16 个页面文件 + `index.html`）。"未互链"不等于"不重叠"，只表示本篇没有为它们写划界段；这条读数登记在此，是读者核对覆盖度时唯一需要的清单，第 18 篇（内存安全）尚未撰写，本篇不预链。

### 范围声明

1. **没有真机。** 本篇全部设备侧读数来自九个桩画像（合成目录树 + 桩 `adb`），策略与源码读数来自 AOSP/CodeLinaro/上游本地副本。凡"多数设备会怎样""这台设备的当前取值"，本篇一律不声明（第廿九章之十第 10 行、本章卅之二十三第 7、10 行）。
2. **审计只核形式。** 卅之十三三条审计的合起来强度是"可复核的形式一致性"，不是语义正确性；第 6 级引文 0 条不等于"每条引用的含义都用对了"。
3. **判据固定但可反驳。** 六级命中口径、`rec` 调用点计数、恒 70 / 恒 54 行、退出码 0 / 2 / 9 都在正文给出可复刻命令；读者按同一命令得到不同结果时，应认为本章有缺陷并按问 4 报回。
4. **写侧能力只登记不使用。** 本篇不给出任何固件写入口、恢复/重置流程或降级路径的操作步骤；第廿五章的表只有名称、来源位置（含分支）、八进制模式位与信息类别，脚本对它们是 `stat` 而非打开。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
