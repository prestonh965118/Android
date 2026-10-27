---
layout: default
title: "Android 系统架构分析报告：分层、进程与信任边界"
description: "Android 六段分层、进程模型、HAL/Treble、SELinux/seccomp/AID 身份与三条更新通道的信任边界分析"
lang: zh-CN
permalink: /Android-System-Architecture-Layers-Processes-and-Trust-Boundaries/
---
# Android 系统架构分析报告：分层、进程与信任边界

## Android System Architecture: Layers, Processes and Trust Boundaries

- 版本：v1.0（2026-10-23）
- 适用对象：Android 应用安全评估、设备/固件合规审计、移动安全管理（MDM/MAS）、威胁与风险建模人员
- 范围声明：本文只描述**公开文档化的架构机制**、其设计意图与失效后果，用于加固与检测。不提供任何漏洞利用代码、偏移量、绕过步骤或可操作的攻击流程。
- 姊妹篇：[Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)、[Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)、[Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)、[Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)、[Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（本文讲结构成型之后的信任边界，该篇讲这些结构是被哪条供给与装配链做出来的、构建身份在哪里形成与丢失）、[Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（本篇按"系统结构 + 信任边界"给每层机制定位，该篇按"一个应用进程被哪几层围住、这些层在什么条件下合并或放宽、放宽能否在设备侧被观察"展开）、[Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（该篇按"一个应用对外声明了哪些弱点形态、这些形态按哪五套公开口径归类与换算"切，落点只在应用层这一格）。本文与它们的区别在于**视角**：前两篇按"缺陷类型学 + 攻防"切，内核结构篇按"代码从哪来、打进哪个镜像、由谁加载"切，Binder 篇按"一次跨进程调用的协议与运行时"切，本篇按"系统结构 + 信任边界"切，回答的是"这块代码在哪个进程、哪层权限、谁校验它"。
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本篇按"系统结构 + 信任边界"横切并给每层机制定位；该篇把"无线"这一竖切放大成六层——芯片固件、内核无线子系统、用户态协议栈、HAL、Framework 权限、设备与企业策略——并逐层给出责任人与更新通道）

---

## 一、摘要

1. Android 不是一台"带应用的 Linux"，而是**六段结构 + 三条独立更新通道**的复合体：应用与 Framework（Java/Kotlin，跑在 ART 上）、原生库与守护进程（C/C++）、HAL（供应商实现，AIDL/HIDL）、Linux 内核（GKI）。它们的**变更速度、代码所有权和验证机制完全不同**，这是理解 Android 安全的首要事实。
2. 系统的**信任根不在 Framework，而在启动链**：BootROM → bootloader → AVB 2.0（vbmeta + hashtree/chain partition + 回滚保护）→ first-stage init → 第二个 init → Zygote → system_server。可信状态（green/yellow/orange/red）最终以 `verifiedBootState`/`verifiedBootHash` 的形式进入密钥证明（Key Attestation）与设备完整性判定，应用侧与服务端都能校验。
3. **进程模型是安全模型的骨架**：`init` 派生 `zygote`，`zygote` 通过 Unix socket 接受 system_server 的 fork 请求并"特化"出应用进程；每个应用获得唯一 UID（`10000+`，多用户以 `100000` 为步长偏移）、独立 SELinux 域与 seccomp 过滤器。Android 5.0 起 SELinux 在系统与应用之间实施 MAC，Android 8.0 起所有应用进程都被 seccomp-bpf 过滤，Android 9 起 `targetSdkVersion >= 28` 的非特权应用必须在**各自独立**的 SELinux 沙箱中运行。
4. **Binder 是唯一的跨进程主干**，并被 Treble 拆成三个域：`/dev/binder`（framework/app）、`/dev/hwbinder`（framework↔vendor 的 HIDL）、`/dev/vndbinder`（vendor↔vendor）。域的选择不是性能优化，而是**权限矩阵**——SELinux 按节点类型分别放行，混用会被拒。调用方身份（UID/PID）只在 binder 线程上下文中有效，这是 Framework 层大量越权缺陷的结构性来源。
5. **HAL 与内核是"最慢更新的两层，也是攻击最常落下的两层"**。VNDK 自 Android 15 起废弃（VNDK APEX、快照与版本属性移除），HIDL 自 Android 13 起废弃并由 AIDL 取代，GKI 让新设备内核统一（Android 12 起、内核 5.10+ 必须使用 GKI）——方向是把"供应商自由度"换成"可验证性与可更新性"。
6. **数据的生命周期由加密架构决定，而不是文件路径**：FBE 把 /data 分成 CE（凭据加密，需用户解锁）与 DE（设备加密，开机即用）两类存储，元数据（目录树/文件名）另由元数据加密保护；密钥链绑定到硬件后端的 KeyMint/Keystore 2.0（Android 12 引入 KeyMint HAL 取代 Keymaster），证明链可由 RKP 远程签发。
7. **更新是四条并行通道**：boot/system/vendor 镜像（A/B 自 Android 7、Virtual A/B 自 Android 11）、APEX/Mainline 模块（Android 10 引入 APEX 格式）、应用（Play）、安全补丁级别（SPL）。任何一条通道落后，都会形成"同一台设备安全水位不一致"的现象，这必须在资产数据里被度量。
8. 对防御者的可执行结论：**架构层的检测成本远低于漏洞层**。分区状态、SELinux 模式、verifiedBootState、SPL、APEX 版本、导出组件、无障碍/覆盖窗口授权这些字段都能在**不 root** 的条件下采集，且能覆盖绝大多数"设备被改造/被长期失联"的场景。第十四章给出脚本。

---

## 二、分层模型：一台 Android 设备的六段结构

### 2.1 层与所有权

| 层 | 主要语言 | 代表组件 | 代码所有者 | 更新通道 | 安全含义 |
| --- | --- | --- | --- | --- | --- |
| 应用 | Kotlin/Java | 三方与预装 APK | 应用开发者 | 应用商店 | 唯一可被"任意代码 + 任意外部输入"同时触及的层 |
| 应用框架 | Java/Kotlin（system_server） | AMS/ATMS/WMS/PMS/PowerManager 等 | AOSP/平台 | system 镜像 + 部分 APEX | 权限判定与身份传递都在这里；缺陷等于"合法 API 做非法的事" |
| 系统原生库/守护进程 | C/C++ | libc、libstagefright、surfaceflinger、audioserver、mediaswserver、cameraserver | AOSP + 供应商补丁 | system/vendor 镜像 | 历史高危面（媒体管线解析不可信输入） |
| Android 运行时（ART） | C++ + dex/oat | dex2oat、libart、Zygote | AOSP（`com.android.art` 为可更新 APEX） | APEX / system | JIT 可执行内存、类加载隔离、JNI 边界 |
| 硬件抽象层（HAL） | C++/Rust，AIDL/HIDL | `hal_camera`、`hal_wifi`、`hal_keymaster`/KeyMint、`hal_gatekeeper` | 供应商/OEM | vendor 镜像（节奏最慢） | 私有代码、审计资料少、长期落后 SPL |
| Linux 内核 | C（GKI + vendor 模块） | binder、selinux、kmod、文件系统、驱动 | Google（GKI）+ 供应商模块 | boot / vendor_boot | 提权的最终落点，见姊妹篇内核报告 |

### 2.2 分区视角：把"层"映射到"块设备"

| 分区 | 内容 | 挂载/验证 | 谁能改 |
| --- | --- | --- | --- |
| `boot` | 内核 + ramdisk（first-stage init） | AVB hashtree/raw 描述符 | OTA |
| `vendor_boot` / `dtbo` | 供应商 ramdisk、设备树叠加 | AVB | OTA（供应商节奏） |
| `super` | 动态分区容器：`system`/`system_ext`/`product`/`vendor`/`odm` | AVB + dm-verity/inline-verity | OTA |
| `vbmeta` / `vbmeta_system` / `vbmeta_vendor` | 校验元数据与链式分区公钥 | 由前一阶段签名链验证 | 解锁后易被替换（yellow/orange 状态） |
| `userdata` | 应用数据、媒体 | FBE + 元数据加密，不签名 | 运行时 |
| `metadata` | 加密策略、OTA 状态等 | 常由元数据加密层保护 | 系统 |
| APEX（在 `system`/`vendor` 内） | 模块化系统组件 | 独立 payload 签名 + 回滚保护 | Mainline/OTA |

要点：**"层"是逻辑视图，"分区"是更新与验证的单元**。审计时问"这段代码在哪个分区"比问"属于哪一层"更有可操作性——它直接决定更新通道、签名链与落后风险。

### 2.3 Treble：为什么"变更速度"要分层

Treble（Android 8 引入）把系统侧与供应商侧解耦，使 framework 更新不要求供应商重新移植整个 HAL。它带来了三个后续机制：稳定的 HAL 接口版本化（VNDK、linker namespace、VINTF 兼容性矩阵）、内核侧的 GKI，以及 SELinux 策略的 system/vendor 分离。安全侧的收益是**可验证性**：边界被固定成接口，就能被静态检查（兼容性测试、`neverallow` 规则）而不是靠约定。

### 2.4 与 iOS 的结构对照

| 维度 | Android | iOS |
| --- | --- | --- |
| 代码来源 | 大量设备特定代码来自供应商（私有） | 几乎全部由 Apple 掌握 |
| 更新通道 | 4 条并行（镜像/APEX/应用/SPL） | 基本 1 条（系统版本）+ 应用 |
| 隔离机制 | Linux UID + SELinux + seccomp + Binder 域 | 沙箱 + entitlement + 强签名链 |
| 侧载 | 可由用户开启 | 默认关闭（除 EU 特定变化） |
| 可观测性 | `adb` + `dumpsys` 提供丰富架构事实 | 无同等开放采集面 |
| 碎片化 | 高（同版本 ≠ 同实现） | 低 |

碎片化的真实含义不是"版本多"，而是**同一份 AOSP 描述在不同设备上由不同代码实现**——审计结论必须带设备型号与固件版本，否则不可复用。

---

## 三、启动链与可信根

### 3.1 阶段表

| # | 阶段 | 执行体 | 建立的事实 | 失败后果 |
| --- | --- | --- | --- | --- |
| 1 | BootROM / PBL | 芯片固化 | 硬件信任根、熔丝公钥 | 不可恢复 |
| 2 | bootloader（LK/U-boot/EDK2） | 已签 | 验证 boot 与 vbmeta | 变砖/降级 |
| 3 | AVB 校验 | bootloader | `vbmeta` 描述符（hash/hashtree/chain）、回滚索引 | 状态降级（orange）或拒启 |
| 4 | 内核启动 | kernel | cmdline 中的 verified-boot 参数、模块签名策略 | 崩溃循环 |
| 5 | first-stage init | ramdisk 内 init | 挂载 `super`、dm-verity/inline-verity、切换 root | 无法进入 second stage |
| 6 | second-stage init | `/system/bin/init` + `.rc` | 服务与 class 启动、SELinux 载入策略、属性上下文 | 关键守护进程缺失 |
| 7 | Zygote | `app_process64` | 预加载类与资源、socket 就绪 | 应用无法启动 |
| 8 | system_server | Zygote fork → `SystemServer` | 启动 bootstrap/core/other 服务群 | 系统半可用（安全服务缺失尤其危险） |
| 9 | 开机完成 / 用户认证 | ActivityManager + SystemUI | DE 可用 → 用户解锁后 CE 可用 | 数据仍锁定（这是特性） |

### 3.2 AVB 2.0 的机制

- **vbmeta 分区**承载顶层元数据：算法、公钥（或链式公钥）、描述符列表与回滚索引。
- **hashtree 描述符**保护可读写挂载的块设备（配合 `dm-verity` 或 `inline-verity`），**hash 描述符**保护只读镜像（如 `boot`）。
- **链式分区（chain partition）**让 `vbmeta_vendor` 等由供应商密钥独立验证，避免 Google 密钥直接接触供应商镜像——这是 Treble 在签名层的对应物。
- **回滚保护（anti-rollback）**：镜像里的 `rollback_index` 与熔丝/持久计数器比较，阻止刷回旧版本以规避补丁。
- **可验证状态**：`green`（全链可信且锁定）、`yellow`（自管密钥，锁定但用户信任链变化）、`orange`（解锁）、`red`（校验失败）。这些状态经 `ro.boot.verifiedbootstate`、`ro.boot.vbmeta.device_state`、`ro.boot.flash.locked` 等属性暴露，并被硬件后端写入密钥证明的 `RootOfTrust`（`verifiedBootKey`/`deviceLocked`/`verifiedBootState`/`verifiedBootHash`），服务端因此可远程判定完整性。

### 3.3 解锁与 verity 模式：审计上最有价值的两组字段

```bash
adb shell 'getprop | grep -E "verifiedbootstate|flash.locked|vbmeta.device_state|vbmeta.digest|vbmeta.hash_alg"'
adb shell 'getprop ro.boot.veritymode; getprop ro.boot.dm-verity.debuggable'
```

`ro.boot.veritymode` 的取值（`enforcing`/`eio`/`logging`/`disabled`）是"这台设备是否仍在拒绝被篡改的块"的最直接信号；`logging`/`disabled` 通常意味着自定义 ROM 或被解链，安全评估结论必须改写。

### 3.4 启动阶段的结构含义（防御侧清单）

1. 用户解锁前可用能力应假定"只有 DE 数据 + 最小服务"；任何"开机即需凭据"的业务逻辑都要能优雅降级。
2. 关键安全服务（keystore2、gatekeeper/gatekeeperd、weaver、生物识别 HAL、权限服务、包管理服务）都在 system_server 或其守护进程里；它们缺失或被降级时，"看起来正常"的界面并不代表安全属性成立——判定应回到第 3.3 节字段与 `getprop` 组合。
3. 恢复出厂/OTA 之后启动链一致，但**密钥可用性**变化最大：CE 数据依赖用户凭据派生，恢复后旧 CE 密钥不可用，等价于逻辑销毁。

---

## 四、进程模型

### 4.1 init

`init` 是 PID 1：解析 `.rc`（`/system/etc/init`、`/vendor/etc/init` …）、按 `class` 批量启停服务、处理触发器（`on property:`、`on boot`）、建立挂载与 cgroup、执行 early-prepare/finish 阶段、并在适当时刻 `exec` first→second stage 切换。**每一个系统服务都对应 `.rc` 中一段声明**，因此"这台设备上到底跑了什么"是可以静态列出的——这使 `init` 成为设备指纹与篡改检测的富矿。

### 4.2 Zygote 与"特化"

- Zygote 预加载常用类与资源，随后**不再加载新代码**，只 fork。
- system_server 需要新应用进程时，通过 **Unix domain socket** 把参数发给对应 Zygote；Zygote fork 后在子进程里做 specialization：改 UID/GID、capabilities、seccomp、cgroup、SELinux 上下文、`nice`/调度，最后 `ActivityThread.main()` 进入应用主循环。
- **USAP 池**（unspecialized app process）预生成未特化进程以缩短冷启动；池化意味着"一个尚未完成特化的进程"短暂存在，检测脚本看到 Zygote 子进程时不能假定它已具备目标应用身份。
- 变体：`zygote64`（默认）、**WebView Zygote**（渲染进程复用）、**App Zygote**（`isolatedProcess`、`app_process` 派生的沙箱实例，供需要额外进程隔离的组件使用）。隔离进程使用 `99000–99999` 段的 UID，无网络、无法读任何应用数据、无法使用内容提供程序——是"呈现不可信代码"这一需求的正确解，而不是把代码放主进程。

### 4.3 system_server 的内部结构

```
Zygote fork → AndroidRuntime::start("system_server")
  → com.android.server.SystemServer.main()
      startBootstrapServices()   # 基础：PowerManager/ActivityManager/PM 等骨架
      startCoreServices()        # 统计、数据库、AppOps 等
      startOtherServices()       # 绝大多数管理器：WMS/ATMS/Input/Notification/Job…
  → Looper.loop()                # 从此进入 binder/主线程调度
```

含义：**system_server 是单进程的数十个服务**。它一旦被攻破，等价于拿到 `system`（UID 1000）+ 平台签名权限的能力集合，但**不等于内核 root**——这正是 Framework 报告第八章"分层归因"的用途（`uid 1000` 与 `root` 的能力矩阵不同）。同时，单进程也意味着**一个坏服务能拖垮全部**（重启 `system_server` = 半软重启），因此稳定性缺陷与可用性攻击常共用同一代码路径。

### 4.4 常驻守护进程与它们的"域"

| 进程 | binder 域 | 典型 SELinux 域 | 为什么重要 |
| --- | --- | --- | --- |
| `surfaceflinger` | /dev/binder | `surfaceflinger` | 合成与可见性判定，覆盖/劫持类问题的落点 |
| `audioserver` | /dev/binder | `audioserver` | 录音权限的强制执行点之一 |
| `mediaswserver` / `media.swcodec`（Codec2） | /dev/binder | 媒体沙箱域（`mediaswserver`、`hal_codec*`，随版本变化） | 解码不可信媒体，历史上最高危的原生入口 |
| `cameraserver` | /dev/binder | `cameraserver` | 摄像头管线 + HAL 桥接 |
| `keystore2` | /dev/binder | `keystore2` | 密钥元数据与访问策略，后端接 KeyMint |
| `gatekeeperd` / `weaver` | /dev/binder | `gatekeeperd` / `weaver` | 凭据校验速率限制，抗暴力破解 |
| `hwservicemanager` / `servicemanager` / `vndservicemanager` | 各自域 | `servicemanager` / `vndservicemanager` | 服务发现与 ACL |
| `statsd` / `traced` / `incidentd` | /dev/binder | `statsd` / `traced` | 可观测性与取证数据来源 |
| `installd` / `vold` / `apexd` | /dev/binder | `installd` / `vold` / `apexd` | 安装、加密挂载、APEX 生命周期 |

（域名会随版本调整；表中值应在具体设备上用 `ps -Z` 核对，第十四章给出方法。）

### 4.5 资源与优先级层

Android 用 cgroup（`cpuset`、`cpu`、`stune`、`memory`、`pid`）与调度策略把进程分为若干可观测档位：原生系统进程、persistent、persistent service、前台、可见、可感知、服务、上一任务、缓存（Android 近版本改由 PSI 压力信息驱动的"cached apps freezer"）。安全相关性：

- **后台启动限制**与**前台服务类型**约束（Android 10+/12+/14+ 逐次收紧）是 Framework 层的滥用防线（伪装前台服务 = 常驻 + 保活）。
- 冻结与进程优先级交互会影响"检测窗口"：一个被冻结的进程不会及时上报行为，EDR/MDM 侧的"最后活动时间"因此必须区分"冻结"与"失联"。

---

## 五、身份与隔离模型

### 5.1 AID 体系

| 范围 | 值 | 说明 |
| --- | --- | --- |
| `root` | 0 | 仅极少数进程（如 `init`），现代 Android 上 ADB shell 不是 root |
| `system` | 1000 | system_server 与多数系统守护进程 |
| 角色账号 | 1001 radio、1002 bluetooth、1003 graphics、1041 audio、1013 media、1017 keystore、1019 drm … | 能力以 UID/GID 组划分 |
| `shell` | 2000 | ADB 上下文，权限受限 |
| 应用 | `10000 + n`（`FIRST_APPLICATION_UID=10000`，`LAST=19999`） | 显示为 `u0aNN` |
| 隔离进程 | `99000–99999` | `isolatedProcess` |
| 多用户偏移 | 每用户 `+100000` | 用户 12 的 `com.foo` = `12*100000 + n` |

**Android 为每个应用分配唯一 UID**，把 UNIX 用户隔离作为第一道防线；这是应用沙箱的根基，也是"跨应用读取默认失败"的原因。

### 5.2 身份的三种"共享"，及其风险

1. `sharedUserId`（= 同 UID + 同签名要求）：一旦使用，两个 APK 从此绑定（卸载/更新互相牵连），能力合并；近版本已在兼容性上持续收紧。
2. 签名/特权权限（`signature|privileged`）：由平台签名或 `privapp-permissions` 白名单授予；**声明-实现错位**（申请了但未在实现里鉴权）是 Framework 报告的常见缺陷模式。
3. 安装来源（installer package name）与"未知来源"：`installerPackageName=null` 表示侧载，MDM 应把它当作风险字段。

### 5.3 SELinux：域、类型、类别

- **MAC 早于策略复杂度**：Android 5.0 起 SELinux 在系统与应用之间提供强制访问控制；**Android 9 起所有 `targetSdkVersion >= 28` 的非特权应用必须在独立的 SELinux 沙箱中运行**（按应用计算的类别），使"同 UID 家族之外"的横向访问更难。
- 应用域示例：`u:r:untrusted_app:s0:c<x>,c<y>`（类别由 `seapp_contexts` 的 `levelFrom` 依据 UID/用户推导）。同一 UID 的多个进程可共享类别，跨应用则因类别不同而被拒。
- **`neverallow` 是平台级红线**：违反即无法通过 CTS/构建，因此"某能力被架构禁止"比"被策略禁止"更强——审计时区分这两者能避免误判（自定义 ROM 可能改策略，但改不动 `neverallow` 约束的构建）。
- Treble 把策略拆成 system/vendor 两份并规定供应商只能使用已声明的 public 类型（`vendor_*`、`hal_*` 域、`hwservice`），配合 `te_build` 检查；这就是"厂商域"与"平台域"的分离来源。

### 5.4 seccomp、capabilities、执行约束

| 机制 | 事实 | 版本锚点 |
| --- | --- | --- |
| seccomp-bpf | 应用进程运行在系统调用白名单过滤器内 | Android 8.0 起**所有应用**都设置 seccomp-bpf 过滤器 |
| capabilities | 进程默认丢弃绝大多数 cap；`init` 按 `.rc` 显式授予 | 平台 |
| `no_new_privs` | 禁止通过 setuid/exec 提权 | 应用进程默认 |
| 只读执行域 | 大量守护进程挂载 `nosuid nodev`，`/data` `noexec` | 平台 |
| 堆加固 | Scudo 硬化分配器 | Android 11 起所有原生代码默认使用（低内存设备除外，仍用 jemalloc） |
| 控制流/指针加固 | kCFI/CFI、Shadow Call Stack、PAC/BTI、MTE（详见内核报告） | 随 SoC/内核版本 |

### 5.5 隔离模型的真实边界（不写绕过）

结构性事实：UID 隔离保护**文件**；SELinux 保护**能力**；seccomp 限制**内核可达面**；Binder 域限制**可调用的服务**。四者互相补位，因此"某一项缺失"通常不会立刻等价于"完全裸奔"，但会让另外三者承受设计外的负载——评估报告应按"这一组合缺哪一环"来写结论，而不是笼统说"沙箱被打破"。

---

## 六、IPC 架构：Binder 的三个域

### 6.1 驱动模型要点

Binder 是**内核中的 RPC 层**：一次 `transaction` 把 Parcel 从一个进程搬到另一个进程，同时完成
（a）句柄表转换（`node`/`ref`：跨进程引用强/弱计数）、（b）**调用方身份注入**（UID/PID，由内核写入事务头，用户态无法伪造）、（c）文件描述符表转移。数据是"一次拷贝"进目标进程的映射区。含义：

- 身份可信的前提是**你处在 binder 线程上下文中**。离开 binder 线程（工作线程、协程池、`Handler`）再问 `Binder.getCallingUid()`，拿到的是自己进程的 UID——这一类错误在 Framework 报告里被列为"身份传递"缺陷模式。
- `clearCallingIdentity()` / `restoreCallingIdentity()` 是"以服务身份继续操作"的正规手段，但**其覆盖范围必须最小**；在 `clear` 状态下执行了基于外部输入的敏感动作，等于把 UID 边界抹掉。
- **同步事务缓冲区有限**（KB～MB 量级，常量随版本变化），`oneway` 异步事务另有独立配额；大块数据应走共享内存/FD 而非 Parcel，这是设计约束，也是"参数过大导致失败"的根因。
- 线程模型：服务端可增 binder 线程（驱动侧有上限），**嵌套调用与锁顺序**导致 binder 死锁是 system_server 类故障的经典形态（属可用性问题，非越权）。

### 6.2 三个设备节点与允许关系

| 节点 | 服务管理器 | 典型用途 | 约束来源 |
| --- | --- | --- | --- |
| `/dev/binder` | `servicemanager` | framework ↔ framework/app，AIDL 系统服务 | SELinux `binder` 类 + `servicemanager` `add/find` |
| `/dev/hwbinder` | `hwservicemanager` | framework ↔ vendor 的 HIDL 接口 | Treble 边界，vendor 侧必须经此域 |
| `/dev/vndbinder` | `vndservicemanager` | vendor ↔ vendor 的私有服务 | 禁止跨到 framework 域 |

要点：**跨域不是"性能选择"而是权限矩阵**。`/dev/vndbinder` 上的服务不会出现在 `service list` 里（那是 `servicemanager` 的视图），因此审计脚本必须分别枚举——这直接对应"厂商侧隐藏服务面"的可见性问题。

### 6.3 接口定义语言的三代

- **AIDL**（framework 侧长期标准）：`.aidl` → 生成 stub/proxy，`Parcel` 手写序列化（历史上类型不安全）。
- **HIDL**（Android 8 引入，Treble 用）：`@Vx.y` 版本化 + 内存布局稳定，配合 `/dev/hwbinder`；**Android 13 起废弃，新 HAL 用 AIDL**。
- **Stable AIDL**（Android 11 起用于 HAL）：在保持内存布局稳定、可跨镜像版本通信的同时，去掉 HIDL 的独立运行时，统一到 binder；Android 11 起允许用 AIDL 实现 HAL。

安全含义：接口版本化让"系统侧新、供应商侧旧"的组合可被校验（VINTF 兼容性矩阵），避免未定义行为；同时也意味着**旧接口不会因平台升级而自动变安全**——一个 `@V1.0` 的 HAL 缺陷在 5 年后仍在设备上。

### 6.4 Parcel 与"数据即指令"

Parcel 承载的不只是标量：FD、Binder 引用、Bundle（可携带对象引用）、`Intent`（可携带 `ClipData`/`Uri`）。当接收端把 Bundle 里的对象**当作可执行组件描述**去 `startActivity`/`sendBroadcast`，或把 FD 当作可信路径打开（未做规范化/所有者校验），就出现"数据即指令"缺陷。Framework 报告第七章把这一类列为独立模式并给出审查信号；本文只强调它的结构成因：**Binder 传输的是引用，而引用天然带有能力**。

---

## 七、HAL 与 Treble 的工程细节

### 7.1 实现形态

- **binderized HAL**（Android 8 起标准写法）：独立进程，经服务管理器发现，跨进程有 UID/SELinux 边界。
- **passthrough HAL**：客户端进程内 `dlopen` 厂商 `.so`，**没有独立进程与域**——它的代码运行在调用者（常是 system 侧守护进程）的权限上下文里。这是安全评估必须单独标记的形态：同一个"HAL 缺陷"，binderized 的失陷半径是 HAL 进程，passthrough 的失陷半径是宿主守护进程。

### 7.2 装载与命名空间

linker namespace（`default`/`vendor`/`product`/`sphal` 等）把不同分区的 `.so` 隔离在各自的搜索路径里：供应商进程只能用 VNDK/vendor 命名空间中的库，系统侧图形等 HAL 走 `sphal`。其效果是**把"随手链接内部符号"变成需要显式声明的动作**，从而可被静态检查。

### 7.3 版本变化（已核实）

| 主题 | 事实 |
| --- | --- |
| HIDL | Android 10 起进入废弃通道，Android 13 文档明确"已废弃，HAL 请改用 AIDL" |
| AIDL HAL | Android 11 引入用于 HAL 的能力，逐步成为默认 |
| VNDK | Android 15 起废弃，VNDK APEX、快照与版本属性被移除 |
| GKI | Android 12 起，出厂内核 5.10+ 的设备必须使用 GKI；SoC/设备相关代码移入可加载模块，模块不得含 SoC 专有代码 |
| KeyMint | Android 12 引入 KeyMint HAL 取代 Keymaster，功能对齐并扩展 |
| RKP | 远程密钥配置替代出厂私钥批量写入；Android 14 前由 `RemoteProvisioner` 应用 + Keystore 2.0 协作 |

### 7.4 为什么供应商层是安全评估的重点

1. **更新节奏不同**：平台月度补丁，vendor 常按季/半年。SPL 落后往往正是被 vendor 镜像拖住。
2. **代码不可见**：AOSP 之外的驱动与 HAL 无法用公开源码审查，只能靠行为观测与厂商通告。
3. **接口暴露**：`hal_camera`、`hal_wifi`、`hal_bluetooth`、`hal_nfc`、`hal_audio` 直接解析空中/线缆输入，可达性天然高于 framework。
4. **GKI/VNDK 的收敛正在改善它**：把可变部分收进可加载模块与稳定接口，让 Google 侧能统一打补丁——这是过去五年 Android 安全架构最有效的结构性改进之一。

---

## 八、框架服务层（结构速览）

> 细节与缺陷类型学见 Framework 报告；本节只提供"架构地图"，避免重复。

| 服务 | 进程 | 职责 | 安全相关点 |
| --- | --- | --- | --- |
| `ActivityManager` / `ActivityTaskManager` | system_server | 进程生命周期、组件启动、task 栈 | 后台启动限制、task 亲和（StrandHogg 类问题的结构根因） |
| `WindowManager` | system_server | 窗口层级、焦点、输入路由 | 覆盖窗口/触摸屏蔽（`SYSTEM_ALERT_WINDOW`） |
| `PackageManager` | system_server | 组件/权限/签名视图 | 导出判定、签名与权限安装期检查 |
| `Permission`/`AppOps` | system_server | 运行时授权 + 操作级策略 | 两者不一致时以 AppOps 兜底 |
| `UserManager` / `DevicePolicyManager` | system_server | 多用户、工作资料、设备管理 | 用户句柄 ≠ 用户 ID；Profile 隔离 |
| `NotificationManager` / `AccessibilityManager` | system_server | 通知与无障碍 | 高能力权限入口，滥用即数据面泄露 |
| `KeystoreService`（keystore2） | 独立进程 | 密钥策略与元数据 | 与 KeyMint/GK/Weaver 协作 |
| `RoleManager`/`WebViewUpdateService`/`DomainVerificationService` | system_server | 角色/引擎更新/域名验证 | 深链归属判定依赖此层正确性 |

应用侧对应结构：`Context`（`Application`/`Activity`/`Service`/`Provider`）是访问上述服务的唯一入口，`Context` 决定身份（包名、UID）与可见性（`createPackageContext`、`Context.DEVICE_*` 存储位置）——这是"同一个 APK 在 CE/DE 看到不同数据"的结构原因。

---

## 九、ART 与代码执行路径

### 9.1 从源码到指令

```
Kotlin/Java → javac/kotlinc → .class → d8 → classes.dex
   （安装/后台）dex2oat + profile → oat/.odex + vdex + art-app-image
运行时：AOT 机器码  /  JIT 编译码  /  解释器  三态混合
```

- Android 7 起 ART 采用 **AOT + JIT 混合**：安装或备份/空闲时用 profile 引导 `dex2oat`，后台作业（`bg-dexopt`）继续优化热路径；启动阶段使用 boot image，Zygote 可复用已编译缓存。
- 安全含义：**JIT 需要"可写也可执行"的受控区域**，运行时因此对代码缓存施加特殊保护（独立映射、不可被普通堆写路径触达）；同时 dex 的加载来源（APK、`PathClassLoader`、`DexClassLoader` 动态加载、`SplitLoader`）构成"代码从哪来"审计问题——**从可写路径动态加载 dex 等于把不可信输入当代码**，这是应用侧最常见的设计错误之一。

### 9.2 可更新性

`com.android.art` 与 `com.android.i18n`、`com.android.conscrypt`、`com.android.media`、`com.android.tzdata`、`com.android.sdkext` 等以 APEX 形式存在，可通过 Mainline 更新——意味着**运行时/加密库/媒体栈的补丁不依赖整机 OTA**。审计时"Android 版本"不足以描述设备风险，必须同时采集模块版本（第 14 章脚本）。

### 9.3 边界收紧

- **非 SDK 接口限制**：Android 9 起对隐藏 API 实施访问限制（名单随版本收紧），其结构意义是"Framework 内部不再是公共依赖"，让平台可安全重构。
- **原生隐式链接限制**：Android 7 起禁止随意 `dlopen`/链接非公开符号，NDK 与 VNDK/`sphal` 命名空间把可用符号集固定下来。
- **反射与序列化风险面**：`Parcel`/`Bundle` 与 `Intent` 的对象还原路径在近版本持续收紧（显式白名单方向），因为"反序列化即执行"的通用问题在 Android 上同样成立。

### 9.4 WebView / Chromium 子系统

WebView 是"嵌在应用里的浏览器内核"：独立渲染进程（经 app/WebView Zygote 特化）、GPU 进程、受限 syscall 策略，更新走 `com.android.webview`/`com.google.android.webview`（Mainline）或 Play。安全上必须按"第三方引擎"对待：应用若暴露 `addJavascriptInterface` 且加载不可信内容，等于把不可信 JS 与 Java 能力放进同一信任域；URL 白名单、`setAllowFileAccess(false)`、Cookie 作用域与 `WebSettings` 配置是标准缓解（Framework 报告第 9.1 节给出条目化建议）。

---

## 十、显示、输入与传感器管线

### 10.1 图形路径

`应用 UI（HWUI/RenderThread）→ BufferQueue/GRALLOC 缓冲 → SurfaceFlinger（合成 + RenderEngine）→ HWC HAL → 显示`。
关键安全属性：**"谁在这个坐标上"** 由 system_server（窗口层级/焦点）与 SurfaceFlinger（合成）共同决定，输入则由 `InputDispatcher` 按焦点窗口投递。覆盖窗口/事件注入类滥用的结构根因正是**呈现层与输入层来自不同判定者**：只要攻击者能让用户看到 A 却把点击送到 B，UI 就成了不可信信道。近版本对该面的收紧方向：覆盖窗口权限更难获得、敏感窗口可要求 `FLAG_SECURE`/触摸屏蔽（`Android: setFilterTouchesWhenObscured` 方向）、系统级对话框（权限/安装确认）优先合成于不可被覆盖的层级。

### 10.2 输入路径

`驱动 evdev → EventHub → InputReader（按设备分类）→ InputManager（system_server）→ InputDispatcher（焦点窗口 + 超时）→ 应用`。
传感器（加速度/陀螺/心率…）走 `sensors` HAL + `SensorManager`；Android 12+ 的隐私指示（麦克风/摄像头访问提示）与传感器隐私开关在架构上位于 framework 侧的强制策略层，配合 AppOps 记录访问事实——因此"是否有监听行为"在设备上是有**可查询痕迹**的。

### 10.3 媒体管线

`MediaMetadataRetriever / MediaExtractor / MediaCodec（Codec2）→ c2 组件 → omx 组件 → 解析器（libstagefright 及外部库）`。这是历史上"远程输入直达原生解析代码"的主通道（Stagefright 类问题即在此层），其架构特征是：**解析进程权限不高但攻击面极大**，且大量解析代码位于可更新 APEX（`com.android.media`）内，因此"整机 OTA 落后但媒体栈已更新"与相反情况都可能出现。

---

## 十一、存储与数据架构

### 11.1 动态分区与镜像形态

`super` 动态分区允许 OTA 调整 `system`/`vendor`/`product` 大小；system-as-root 使 `system` 成为挂载根；GSI 让 AOSP 镜像可跑在 Treble 设备上（评估"兼容性"时有用，但 GSI 通常非锁定状态）。

### 11.2 APEX：格式与生命周期

- zip 容器：`apex_manifest.json`（包名/版本）、`apex_payload.img`（**ext4 + dm-verity**）、`apex_pubkey` 与签名信息；安装时由 `apexd`/`installd` 校验签名与回滚版本后挂载到 `/apex/<id>`。
- `flatten_apex`（旧内核路径）把内容平铺到 `/system/apex`，代价是失去部分独立更新能力。
- 服务发现与链接：APEX 内的库通过命名空间暴露，`apex-info-list.xml` 记录已激活版本——这是**设备真实模块版本**的可信来源之一（第 14 章采集）。

### 11.3 加密分层

| 层 | 保护对象 | 机制 |
| --- | --- | --- |
| 元数据加密 | 块设备上的目录树/文件名/布局 | inline crypto（dm-default-key 类） |
| FBE 内容加密 | 每个文件内容（与文件名/长度） | AES-256-XTS（CE/DE 密钥） |
| FBE 文件名 | 文件名（策略相关） | AES-256-CTS |
| 密钥保护 | 存储密钥本身 | 由 synthetic password 派生的密钥包裹；硬件侧经 KeyMint/Weaver，绑定认证 |

- FBE 自 Android 7.0 支持、**Android 10 起为强制方向**；`/data/system_de`、`/data/user_de` 属 DE（开机即可用），`/data/data`、`/data/user`、`/data/media` 属 CE（须用户解锁后可用）。这就是"开机后闹钟/来电可用而相册不可用"的实现来源，也是取证上"重启后应用数据不可读"的原因。
- **DE 数据是低价值保护域**：任何放在 DE 的敏感凭据，其暴露时序从"开机"就开始。架构层建议是"仅把开机必需的最小状态放 DE"。

### 11.4 应用数据作用域

- `/data/data/<pkg>` 属应用 UID + `appdata_file` 类型；跨应用读写需显式授权。
- **Scoped Storage**（Android 10 引入、11 起生效方向）与 `MediaProvider`：媒体文件归属由系统提供器管理，应用不再需要全量存储权限。
- `MediaStore`/`Storage Access Framework`/`DocumentsProvider`/`DownloadProvider` 是"跨应用共享"的正规通道；`FileProvider` 通过 `content://` + 临时 URI 权限实现细粒度共享——**URI 授权的作用域（`Intent.FLAG_GRANT_*`、是否 `persistable`、是否随 task 传递）**是数据外泄类缺陷的高频审计点（Framework 报告第三章的 Provider 清单）。
- adb 备份/恢复通道在近版本被大幅限制，不应作为合规数据获取路径；有 MDM 需求应使用受管设备上的官方 API。

---

## 十二、更新架构

### 12.1 镜像更新

| 模式 | 引入 | 机制 | 失败处理 |
| --- | --- | --- | --- |
| 非 A/B（recovery） | 早期 | 单分区 + 差分包，更新中不可用 | 失败风险高，已弃用方向 |
| A/B（seamless） | Android 7 | 双槽位，写备用槽后切换 `slot_suffix` | 新槽启动失败自动回退 |
| Virtual A/B | Android 11 | 保留 A/B 语义但不占双倍空间：快照 + COW 文件 | 合并前可回滚，COW 完成后释放 |
| Retrofit | 面向旧设备 | 为没有双槽的分区（vendor/odm）准备快照 | 更新窗口更长 |

`update_engine` 校验 payload 签名与大小、写入后触发回滚索引与槽位切换；`ro.boot.slot_suffix`、`UPDATE_ENGINE` 日志与 `getprop` 是"这台设备到底升过什么"的观测面。

### 12.2 四条更新通道与其中风险

1. **boot / GKI**：内核与 ramdisk，受回滚索引保护，节奏由 OEM。
2. **super（system/vendor/product/odm）**：平台与 HAL，SPL 主要来源于此。
3. **APEX/Mainline**：ART、媒体、Wi-Fi、蓝牙栈、Conscrypt、APNs、WebView 等，Google 可直接推。
4. **应用**：Play/MDM。

风险来自**通道间的错配**：例如"平台 SPL 新但 vendor 镜像旧"（常见），或"Android 版本新但 `com.android.art`/WebView 落后"。因此设备风险评分至少要四个字段：Android 版本、SPL、vendor 构建日期、关键 APEX 版本。

### 12.3 SPL 与公开漏洞库

`ro.build.version.security_patch` / `Build.getSecurityPatchLevel()` 是设备侧唯一可自动化的"补丁水位"声明；配合 Android 安全公告的 CVE 列表与 CISA KEV（在野利用目录）可生成"这台设备暴露于已知在野漏洞"的判定。采集脚本与指标定义见内核报告第八章，本文第十四章只保留架构字段部分。

### 12.4 完整性对外声明

架构把"可验证性"外化成三件东西：**Key Attestation**（`RootOfTrust`：`verifiedBootKey`/`deviceLocked`/`verifiedBootState`/`verifiedBootHash`）、**Play Integrity**（设备/基础/强度判定 + APK 绑定）、**RKP**（避免出厂共享私钥的证明链）。防御侧的意义：服务端能在不依赖客户端自觉的前提下拒绝低完整性设备访问敏感接口——这是目前 Android 上性价比最高的架构级缓解。

---

## 十三、架构级信任边界地图

| 边界 | 跨越机制 | 校验点 | 典型失误（结构层面） |
| --- | --- | --- | --- |
| 应用 ↔ 应用 | Binder（经系统服务）、`content://` URI、文件 | UID + SELinux 类别 + URI 授权 | 把整目录挂出、URI 权限过宽/持久化 |
| 应用 ↔ system_server | Binder，`/dev/binder` | 权限位 + AppOps + 调用方 UID | 在非 binder 线程取调用方身份；`clearCallingIdentity` 覆盖过大 |
| 应用 ↔ 内核 | syscall | seccomp 白名单 + 内核权限检查 | 直接使用未过滤接口、驱动 `ioctl` 面 |
| 平台 ↔ 供应商 | Binder 域（`/dev/hwbinder`、`/dev/vndbinder`）、VNDK/命名空间 | VINTF + SELinux 双策略 + 接口版本 | passthrough HAL 把厂商代码放进系统宿主 |
| 不可信输入 ↔ 原生解析器 | 文件/网络 → MediaCodec/Extractor | 进程隔离 + 沙箱域 + APEX 更新 | 在特权进程内解析媒体 |
| 呈现层 ↔ 用户感知 | SurfaceFlinger 合成 + InputDispatcher | 窗口层级/焦点/触摸屏蔽 | 覆盖窗口 + 事件注入组合出的欺骗信道 |
| 启动链 ↔ 运行时 | AVB 描述符、回滚索引 | bootloader 熔丝链 + attestation | 解锁/换 `vbmeta` 后仍按受信设备提供服务 |
| 凭据 ↔ 数据 | 密钥链（SP → CE 密钥 → 文件密钥） | KeyMint/Weaver + 认证速率限制 | 敏感数据落 DE 或落可写公共目录 |
| 设备 ↔ 后端 | TLS + 密钥证明 + Integrity | 服务端策略 | 客户端声明当事实、未校验 `verifiedBootState` |
| 更新器 ↔ 分区 | `update_engine` + `apexd` | payload/APK 签名、版本回滚 | 关闭校验的自定义 ROM（`veritymode=logging`） |

用法：任何 Android 安全事件都应能被落到这张表的某一行；落不进去，通常说明对系统结构的理解有缺口（Framework 报告 8.5 节的"分层归因表"是同一思路的下游工具）。

---

## 十四、审计与检测：把架构事实采下来

> 全部命令在**未 root** 设备上可用（个别需 root 的已在注释标明）。脚本只做采集与断言，不写入设备。

### 14.1 设备架构指纹

```bash
#!/usr/bin/env bash
# arch-fingerprint.sh —— 采集一台设备的架构事实（只读）
set -u
dev=${1:-}
[ -n "$dev" ] || { echo "usage: $0 <serial>"; exit 2; }

sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

echo "== build =="
echo "release=$(p ro.build.version.release) sdk=$(p ro.build.version.sdk)"
echo "spl=$(p ro.build.version.security_patch) incremental=$(p ro.build.version.incremental)"
echo "fingerprint=$(p ro.build.fingerprint)"
echo "brand=$(p ro.product.brand) model=$(p ro.product.model) device=$(p ro.product.device)"
echo "board=$(p ro.product.board) vendor_sdk=$(p ro.product.vndk.version)"

echo "== boot chain =="
echo "vbstate=$(p ro.boot.verifiedbootstate) locked=$(p ro.boot.flash.locked)"
echo "vbmeta_state=$(p ro.boot.vbmeta.device_state) vbmeta_digest=$(p ro.boot.vbmeta.digest)"
echo "hash_alg=$(p ro.boot.vbmeta.hash_alg) vbmeta_size=$(p ro.boot.vbmeta.size)"
echo "veritymode=$(p ro.boot.veritymode) slot=$(p ro.boot.slot_suffix)"
echo "kernel=$(sh 'cat /proc/version')"

echo "== isolation =="
echo "selinux=$(sh 'getenforce')"
echo "kptr_restrict=$(sh 'cat /proc/sys/kernel/kptr_restrict')"   # 未授权时通常为空
echo "abi=$(p ro.product.cpu.abi) sealed=$(p ro.build.version.min_supported_target_sdk)"

echo "== modules =="
sh 'ls /apex 2>/dev/null | tr "\n" " "'
echo
sh 'pm list packages -s | wc -l'
```

注：`kptr_restrict` 等内核参数在未授权 shell 下通常读不到，属性名在不同 OEM 上也常缺失；**采不到不等于未启用**，把"字段缺失"记入数据质量而不是安全结论。

### 14.2 断言：什么算异常

```bash
#!/usr/bin/env bash
# arch-assert.sh —— 对第 14.1 节的字段做保守断言，输出风险行
set -u
dev=${1:-?}
sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

vb=$(p ro.boot.verifiedbootstate)
case "$vb" in
  green) ;;
  yellow) echo "MED   自管信任根（yellow）：AVB 链仍验证但使用用户密钥" ;;
  orange) echo "HIGH  引导加载程序已解锁（orange）：完整性判定不可用" ;;
  red)    echo "HIGH  校验失败（red）" ;;
  '')     echo "LOW   未暴露 verifiedbootstate（OEM 属性差异，记录数据质量）" ;;
  *)      echo "MED   未知 verifiedbootstate: $vb" ;;
esac

vm=$(p ro.boot.veritymode)
case "$vm" in
  enforcing) ;;
  eio)       echo "MED   dm-verity=eio（I/O 错误降级模式）" ;;
  logging)   echo "HIGH  dm-verity=logging：篡改块不再被拒绝" ;;
  disabled)  echo "HIGH  dm-verity=disabled" ;;
  '')        echo "LOW   未暴露 veritymode" ;;
  *)         echo "MED   未知 veritymode: $vm" ;;
esac

[ "$(p ro.boot.flash.locked)" = "0" ] && echo "HIGH  flash.locked=0（设备未锁定）"
[ "$(sh getenforce)" = "Permissive" ] && echo "HIGH  SELinux 全局 permissive"

spl=$(p ro.build.version.security_patch)
if [ -n "$spl" ]; then
  age=$(( ( $(date +%s) - $(date -d "$spl" +%s) ) / 86400 ))
  [ "$age" -gt 90 ]  && echo "HIGH  SPL 落后 ${age} 天（${spl}）"
  [ "$age" -gt 35 ] && [ "$age" -le 90 ] && echo "MED   SPL 落后 ${age} 天（${spl}）"
else
  echo "MED   无 security_patch 属性"
fi

# 域与 UID 一致性：需 root（userdebug/eng 构建）或在应用自身进程内采集；生产设备上本段会跳过
if [ "$(sh id -u)" = "0" ]; then
  sh 'ps -A -o PID=,UID=,NAME=' | awk '$2>=10000 && $2<20000' | while read -r pid uid name; do
    ctx=$(sh "cat /proc/$pid/attr/current" | tr -d '\r\n')
    case "$ctx" in
      *untrusted_app*|*isolated_app*) ;;
      '') ;;
      *) echo "MED   PID $pid（$name，UID $uid）SELinux 域不在预期集合: $ctx" ;;
    esac
  done
else
  echo "INFO  非 root，跳过进程域检查（应用侧可改读 /proc/self/attr/current 自行上报）"
fi

# VNDK/模块版本：GSI 或 vendor 构建日期过旧通常意味着 vendor 通道落后
echo "INFO  vendor build=$(p ro.vendor.build.date) vndk=$(p ro.product.vndk.version)"
```

### 14.3 值得入库的架构字段

| 字段 | 来源 | 用途 |
| --- | --- | --- |
| `ro.build.version.security_patch` | getprop | 补丁水位（第一排序键） |
| `ro.build.version.release` / `sdk` | getprop | 平台能力差异（导出声明强制、 PendingIntent 可变性等） |
| `ro.vendor.build.date` / `ro.product.vndk.version` | getprop | vendor 通道落后程度 |
| `ro.boot.verifiedbootstate` / `ro.boot.flash.locked` / `ro.boot.veritymode` | getprop | 完整性可信度、是否被改造 |
| `getenforce` | shell | SELinux 模式 |
| `/apex/*` 列表与 `apex-info-list.xml` 版本 | shell | Mainline 模块水位 |
| 已授予的高能力权限（无障碍/通知监听/设备管理员/覆盖窗口） | settings + dumpsys | 见 Framework 报告第八章 |
| `installerPackageName` | dumpsys package | 侧载判定 |

### 14.4 与 iOS 检测的差异

Android 提供**大量未授权可读的架构属性**（`getprop`、`dumpsys`、`/apex`），因此"设备状态断言"可以做得很细；代价是这些值由设备自己报告，**远端可信判定必须回到 Key Attestation/Play Integrity 的签名声明**，而不是采信客户端上报的字符串。写检测方案时，把"本地运维用 getprop"与"服务端信任用 attestation"明确分成两条链路，避免混用。

---

## 十五、常见误区

1. **"Android = Linux，所以 root 才是高危"**：`system`（UID 1000）与 `root` 能力集差异很大；把 system_server 失陷写成"内核 root"会高估影响，反之会低估横向能力。
2. **"targetSdk 只是兼容性开关"**：它决定平台侧强制收紧是否作用于你的应用（显式导出、PendingIntent 可变性、后台启动、Scoped Storage、非 SDK 接口限制）。
3. **"HAL 是驱动"**：HAL 是接口 + 实现，可 binderized（独立进程/域）也可 passthrough（宿主进程内）；后者把厂商代码放进系统权限上下文，风险模型完全不同。
4. **"SELinux 挡住一切"**：MAC 限制能力，seccomp 限制 syscall，UID 限制文件——三者正交，任一被削弱都不等于其他失效，但组合缺失会让剩余机制承担设计外负载。
5. **"分区只读 = 内容可信"**：只读还需 AVB 描述符 + verity 模式 + 锁定状态共同成立；`veritymode=logging` 时"只读"只是挂载选项。
6. **"Android 版本新 = 更安全"**：vendor 通道、APEX 模块、WebView 可能各自落后；单值风险评分不可靠。
7. **"DE 目录是'系统目录'所以安全"**：DE 在开机即解密，是低价值保护域；凭据放 DE 等于提前暴露时间窗。
8. **"content:// 只是文件路径"**：URI 携带临时授权与 task 传播语义，`persistable` 权限是长期能力。
9. **"WebView 是浏览器，安全由 Chrome 负责"**：WebView 由应用配置决定信任域，JS 接口与文件访问组合是应用自己的责任。
10. **"getprop 报告的就是事实"**：属性是本地自我声明，服务端的完整性判定必须依赖密钥证明链。
11. **"framework 与 vendor 边界由代码自觉维护"**：边界由 linker namespace、SELinux 双策略、VINTF 强制；绕过它的写法通常在构建期就失败。
12. **"binder 传输 = 参数拷贝"**：传输包含 FD 与对象引用，引用即能力；把外部 `Bundle` 当指令执行是最贵的错误。

---

## 十六、趋势判断（2024 → 2026）

1. **可变部分继续被"接口化"**：HIDL 退场（AIDL/Stable AIDL 统一）、VNDK 废弃、GKI 模块收敛；结果是供应商自由度下降、平台侧统一打补丁能力上升。
2. **Mainline 覆盖持续扩大**：运行时、加密库、媒体、连接栈的可更新性成为补丁的第一线；风险指标要从"OS 版本"转向"模块版本向量"。
3. **硬件能力进入策略层**：MTE/内存标签、PAC/BTI、独立安全芯片（密钥/认证隔离）逐步变成**可被 attestation 声明的属性**，服务端可据此做设备分级。
4. **呈现层信任被显式建模**：隐私指示、传感器开关、敏感窗口不可覆盖层级、事件注入防护——Android 正在把"用户看到的即真实"当作架构责任，而非应用责任。
5. **供应商差异收敛但不会消失**：GKI/VNDK 之后，vendor 面（驱动、固件、OEM 定制 Framework）仍是碎片化主要来源；跨机型结论必须带构建指纹。
6. **合规驱动架构**：`neverallow` + CTS + VINTF 让"安全属性"成为可测试项；企业客户在采购验收里应直接引用这些测试项，而不是笼统要求"通过安全测试"。

---

## 十七、结论

Android 的安全模型不是单点防护，而是**六段结构 × 四道正交隔离（UID / SELinux / seccomp / Binder 域）× 四条更新通道**的乘积。结构给出的核心判断有三条：

1. **可信性沿启动链建立，沿 attestation 外化**。任何绕过启动链的结论（解锁、verity 降级、`vbmeta` 替换）都必须使"这台设备的所有客户端声明"整体降权。
2. **供应商层是结构的短板，且正在被接口化收敛**。评估一台设备时，先问"哪条更新通道落后"，再问"哪个 CVE 影响我"。
3. **绝大多数架构事实可在不 root 的条件下采集**。第十四章的两个脚本足以支撑资产库、采购验收与事件响应初筛；把它们和 Framework 报告的权限面采集、内核报告的补丁水位指标合起来，就构成 Android 侧的三层观测面板。

---

## 十八、常见问题

**Q1：我应该按"版本"还是按"分区"描述风险？**
两者都要。版本决定能力集（`targetSdk`/平台强制项），分区决定更新通道与签名链。只报版本会漏掉 vendor 落后；只报分区会漏掉平台侧策略差异。

**Q2：为什么同一 CVE 公告在两台同版本设备上结论不同？**
因为 HAL/驱动实现不同（供应商代码）、GKI 版本不同、`neverallow` 之外的策略不同、可执行缓解（MTE/PAC/BTI）能力不同。架构层解释就是"同版本 ≠ 同实现"。

**Q3：passthrough HAL 现在还常见吗？**
在图形/音频等低延迟路径上仍存在。评估时把它标成"与宿主守护进程同权限域"，失陷半径按宿主计。

**Q4：seccomp 能替代 SELinux 吗？**
不能，方向不同：seccomp 限制"能调用哪些内核接口"，SELinux 限制"能对哪些对象做什么操作"。

**Q5：FBE 之后还需要应用层加密吗？**
需要区分场景：FBE 保护静态数据，但不限制"已解锁状态下具备 UID 的代码"的读取；敏感凭据仍应走密钥证明绑定 + 服务端策略。

**Q6：Mainline 更新能覆盖内核漏洞吗？**
不能。内核走 boot/GKI 通道；APEX 覆盖运行时与部分系统组件。这正是"SPL 落后"与"模块版本落后"要分别采集的原因。

**Q7：MDM 如何在不 root 的前提下判断设备被改造？**
组合 `verifiedbootstate` + `flash.locked` + `veritymode` + `getenforce` + 自定义 ROM 特征（`ro.build.type=userdebug`、异常 `ro.build.fingerprint`）+ 侧载标记；关键业务侧再加服务端 attestation 校验。

**Q8：本文与 Framework/内核报告重叠吗？**
重叠在结构描述，不重叠在结论：那两篇给缺陷类型学与检测清单，本文给"代码在哪、谁校验、走哪条更新通道"。三者组合使用。

---

## 十九、参考资料（公开来源，本文事实均来自以下页面）

1. Android Open Source Project — Architecture overview：https://source.android.com/docs/core/architecture
2. AOSP — About the Zygote processes：https://source.android.com/docs/core/runtime/zygote
3. AOSP — Configure ART（AOT/JIT 混合、profile 引导）：https://source.android.com/docs/core/runtime/configure
4. AOSP — Implement ART just-in-time compiler：https://source.android.com/docs/core/runtime/jit-compiler
5. AOSP — Hardware abstraction layer (HAL) overview（HIDL 自 Android 13 废弃、binderized HAL）：https://source.android.com/docs/core/architecture/hal
6. AOSP — HIDL：https://source.android.com/docs/core/architecture/hidl
7. AOSP — AIDL for HALs（Android 11 起支持）：https://source.android.com/docs/core/architecture/aidl/aidl-hals
8. AOSP — Use binder IPC（三个 binder 设备域与服务管理器）：https://source.android.com/docs/core/architecture/hidl/binder-ipc
9. AOSP — Vendor Native Development Kit (VNDK) overview（Android 15 起废弃）：https://source.android.com/docs/core/architecture/vndk
10. AOSP — Linker namespace：https://source.android.com/docs/core/architecture/vndk/linker-namespace
11. AOSP — Generic Kernel Image (GKI)：https://source.android.com/docs/core/architecture/kernel/generic-kernel-image
12. AOSP — APEX file format：https://source.android.com/docs/core/ota/apex
13. AOSP — Mainline（可更新模块）：https://source.android.com/docs/core/ota/modular-system
14. AOSP — Remote Key Provisioning：https://source.android.com/docs/core/ota/modular-system/remote-key-provisioning
15. AOSP — A/B updates：https://source.android.com/docs/core/ota/ab
16. AOSP — Virtual A/B（Android 11 引入）：https://source.android.com/docs/core/ota/virtual_ab
17. AOSP — Application Sandbox（每应用 UID、SELinux 5.0/9、seccomp-bpf 8.0）：https://source.android.com/docs/security/app-sandbox
18. AOSP — Android security features：https://source.android.com/docs/security/features
19. AOSP — SELinux：https://source.android.com/docs/security/features/selinux
20. AOSP — Android Verified Boot：https://source.android.com/docs/security/features/verifiedboot
21. AOSP — Encryption（总览）：https://source.android.com/docs/security/features/encryption
22. AOSP — File-based encryption：https://source.android.com/docs/security/features/encryption/file-based
23. AOSP — Metadata encryption：https://source.android.com/docs/security/features/encryption/metadata
24. AOSP — Scudo 硬化分配器（Android 11 起默认）：https://source.android.com/docs/security/test/scudo
25. AOSP — Hardware-backed Keystore / KeyMint（Android 12）：https://source.android.com/docs/security/features/keystore
26. Android Developers — Verify hardware-backed key pairs with key attestation：https://developer.android.com/privacy-and-security/security-key-attestation
27. Android Developers — WebView 与不可信原生桥接风险：https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges
28. CISA — Known Exploited Vulnerabilities Catalog：https://www.cisa.gov/known-exploited-vulnerabilities-catalog
29. AOSP — Sign builds for release（APEX/镜像签名）：https://source.android.com/docs/core/ota/sign_builds

---

咨询ios系统请咨询 telegram：https://t.me/one00190
