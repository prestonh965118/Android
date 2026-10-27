---
layout: default
title: "Android 远程代码执行（RCE）攻击面与防御分析：投递通道、缺陷类型与只读核验 | Android 安全报告集"
description: "把 Android 上的远程代码执行拆成四条可清点的投递通道（内容解析器、浏览器内核与 WebView、近场无线、基带空中接口），说明公告 RCE 口径与严重度判据、公开案例与在野利用统计、CFI/ShadowCallStack/HWASan/MTE/进程拆分等缓解机制各自能挡什么，并给出可运行的只读设备侧核验脚本。面向 Android 平台安全、移动设备管理与安全运营。"
lang: zh-CN
permalink: /Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis/
---
# Android 远程代码执行分析报告：投递通道、缺陷类型学与防御侧核验

**Android Remote Code Execution: Delivery Channels, Defect Typology, Mitigations and Read-only Verification**

- **报告日期**：2026-10-26
- **报告性质**：公开资料整理与防御性攻击面分析
- **适用对象**：Android 平台/ROM 安全负责人、移动安全运营与 EMM/MDM 管理员、应用与媒体/协议栈研发、漏洞管理与补丁治理负责人、第三方评估机构、移动取证与事件响应
- 视角声明：本文写的是**"远程"这段路**——不可信输入如何到达设备上的解析代码、缺陷属于哪一类、平台用什么机制削弱它、以及防御方在设备上能只读核验到哪一步、核验不到什么。落地之后的事情归姊妹篇：
  - [Android 跨域攻击面地图与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面清点单元与 R0–R4 分级、目标选择评分、模糊测试方法——本文的通道划分是它的一个切片，不复用它的评分表）
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核缺陷类型学与提权链——本文止于"进入用户态某个域"，不展开内核侧）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（EoP/ID 类缺陷与权限判定，与 RCE 的区分口径）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（跨进程输入如何被传递与反序列化——本文第 6.2 节的机制细节在其第 5、9 章）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层、进程模型、SELinux/seccomp/AID 身份；本文第 5 章在其之上叠加"执行落点"视角）
  - [Android Linux 内核结构与子系统分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（无线/网络子系统与设备节点供给）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（基带与安全处理器的器件侧形态；本文第 11 章负责"这一段的到达性证据"）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（SPL 语义与更新通道——本文第 10、13 章直接使用其结论，不重复论证）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（验证启动与度量状态，本文第 14 章作为执法状态证据采集）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（本文回答"外部输入怎么抵达解析代码"，该篇回答"代码进来之后落在哪个 UID、哪个 SELinux 域、哪份 seccomp 过滤之下"；执行落点的身份构成与放宽条件不在本文展开）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（本文回答外部输入怎么抵达解析代码，该篇回答这段代码所属的应用在清单里已经声明了哪些弱点类别，两者在内容解析组件这一格相接）
- 范围声明：全文只讨论**公开文档化**的公告口径、机制、公开研究论文与公开通告中的缺陷类别，并给出防御侧的清点、度量、核验与检测流程。**不提供任何漏洞利用代码、内存布局与偏移、原语构造、绕过步骤或可直接用于攻击真实设备的操作化流程。**文中所有"如何找/如何验证"均指研发自查、授权评估与检测覆盖建设，运行前提是你对目标设备或代码拥有所有权或书面授权。

---

## 一、摘要

"Android 远程代码执行"在公开资料里不是一个漏洞，而是**一组结构相同的问题**：某段解析不可信输入的代码（多数用 C/C++ 写成），运行在某个特权或半特权的进程里，输入可以从网络、短信、邻近无线或空中接口抵达，于是缺陷的后果不是崩溃而是执行。Android 公告体系本身就按这个后果组织——其类型列使用 `RCE: Remote code execution` 这样的标注，并在每个分区的说明里给出**后果与前提**（例如当期公告中反复出现的表述："The most severe vulnerability in this section could lead to remote code execution with no additional execution privileges needed. User interaction is needed for exploitation."）。

本文给出六个结论：

1. **可达面可以穷举成四条投递通道**：内容解析器（媒体/图像/字体/压缩包/浏览器文档）、浏览器内核与 WebView、近场无线（蓝牙/Wi-Fi/NFC）、基带与空中接口。每条通道都能回答四个问题：解析代码在哪个进程、在哪个 SELinux 域、需要什么前提（用户交互/邻近/已入网）、修复经哪条更新通道到达。**清点到这个粒度，"RCE 风险"就从形容词变成清单。**
2. **公开案例的分布是稳定的**：内容解析器（Stagefright 及其后的媒体框架加固，Google 安全博客的表述是"Starting in 2015 with Stagefright, a large number of Critical severity vulnerabilities were reported in Android's media framework"）、蓝牙栈（BlueBorne，CERT/CC 明确写出 Android 侧受影响的补丁级别界限）、Wi-Fi 协议实现（FragAttacks）、基带（Project Zero 2023-03 一次性公开 Exynos 调制解调器四个 **Internet-to-baseband remote code execution** 缺陷）。
3. **"远程"不等于"零点击"**。公告口径里 user interaction 是一个独立维度，同一个 RCE 分区在不同组件下的前提不同。把"RCE"直接读成"零点击"是对公告的误读，也是本文第 18 章误区清单的第一条。
4. **执行落点的权限由进程与域决定，而不是由漏洞名字决定**。媒体框架加固的做法是"Splitting AV pipeline components into app-specific sandboxed processes"并配合沙箱与系统调用过滤——同一段解析代码，落在哪个域里，决定了拿到的是媒体权限还是设备权限。这是本文与内核/Framework 篇的分工线。
5. **平台侧对这一类缺陷的削弱是有记录、可核对的**：CFI/ShadowCallStack/HWASan/MTE/IntSan 与 BoundSan 的逐版本引入（AOSP 的 Security enhancements 与专项页面）、内存安全语言的比例变化（AOSP 的内存安全页面记录非内存安全语言"represent over 70% of the Android platform code"；Google 安全博客记录内存安全类缺陷占比从 2019 年的 76% 降到 2022 年的 35%；2025-11 的官方博客记录 Rust 代码的内存安全漏洞密度相对 C/C++ "1000x reduction"）、Mainline/APEX 让部分组件可以不等整机 OTA 就更新。
6. **到达性是这套体系最薄弱的一段**，而且分两种：芯片/基带侧的修复是否落到你的机型，**在公开资料里通常无解**；设备侧能采到的只有版本字符串一类代理指标。因此本文第 13–15 章给出七项运营指标与两个只读脚本——它们把"这台设备的远程可达面按未修复计"变成一个可复核的判定，同时显式列出采不到的项。

交付物三件：**四通道 × 落点 × 更新通道的可达面表**（第 3–5 章）、**缓解机制与其失效面矩阵**（第 9–12 章）、**七项指标 + 两个只读脚本 + 基线文件格式**（第 13–15 章）。

---

## 二、口径：公告里的 RCE 到底标注了什么

要谈"Android RCE"，先固定三个公开口径，否则统计与比较都会失真。

### 2.1 类型列：后果分类，不是攻击路径分类

Android 安全公告在每条缺陷上给出类型标注，其图例把 `RCE` 解释为 Remote code execution，并与 EoP（Elevation of privilege）、ID（Information disclosure）、DoS（Denial of service）并列。这个类型列说的是**成功利用的最坏后果**，不说明输入从哪来、也不说明需要几步。因此：

- 一个 EoP 缺陷可能比一个 RCE 缺陷更值得处置——如果前者给了系统权限而后者只在深沙箱里执行。
- "RCE 标注在 Framework 分区"与"RCE 标注在 System 分区"的可核验度不同：前者可回到 AOSP 源码与提交，后者常常对应厂商实现。
- 一个网络输入触发的缺陷，其标注可能是 ID 或 DoS——**不能靠类型列反推可达性**，这正是第 3 章存在的理由。

### 2.2 严重度判据：Critical 档与"任意代码执行"

AOSP 的 Security updates and resources 页给出严重度判据表，其 Critical 档的后果描述包含在 TEE 或安全单元（SE）中任意执行代码这类情形；公告正文则按"最严重的漏洞可能导致的后果 + 是否需要额外执行权限 + 是否需要用户交互"三件套描述每个分区。**这三件套是可比较的结构**，做清单时应当逐条抽出来，而不是把严重度当成一个可排序的标量。

### 2.3 在野利用的官方措辞是保守的

公告对已被利用的缺陷使用固定表述，例如当期公告中的原文注记："There are indications that CVE-… may be under limited, targeted exploitation."（2026-03 公告对 CVE-2026-21385 即使用该句）。这句话有两层含义：

- 它**不是**"影响面很小"的意思——"targeted"意味着存在真实目标选择；
- 它**也不是**"只有被点名才要紧"的意思——措辞保守是披露政策，不是风险度量。第 8 章用 GTIG 的统计补上这一维度。

---

## 三、可达性模型：四条投递通道

本章是全文的骨架。每条通道给出：输入形态、解析代码所在的典型落点、触发前提、公开案例、设备侧可观测证据。**"可观测证据"这一列决定了第 14–15 章脚本采集哪些字段。**

| 通道 | 输入形态 | 解析代码的典型落点 | 触发前提 | 公开案例族 | 设备侧可观测证据 |
| --- | --- | --- | --- | --- | --- |
| **C1 内容解析器** | 一条 MMS/邮件/网页里的媒体文件、图像、字体、压缩包、文档；自动预加载即触发 | 媒体框架与编解码组件（历史上集中于 mediaserver，加固后按 AOSP 文档拆分为按应用的沙箱进程）、厂商/开源解析库 | 依组件而异：可无交互（预加载路径），也可需要打开文件 | Stagefright（2015-08 群聚披露，Project Zero 后续分析用到 CVE-2015-3864）；此后每期公告的 Media Framework / System 分区 | 框架 SPL、APEX 挂载情况（`com.android.media` 等）、构建类型 |
| **C2 浏览器内核与 WebView** | 网页内容、JS、WebAssembly；由 Chrome 或任何基于 WebView 的应用承载 | WebView 内核（Chromium）、渲染与 JS 引擎 | 通常需要访问页面；多进程沙箱内的渲染器落点 | 每期 Chrome Releases 安全更新；Android 公告中的 WebView 分区条目 | WebView 版本号（可从包信息只读读取）、Chromium 当期稳定版本的滞后跨度 |
| **C3 近场无线** | 蓝牙协议报文（SDP/L2CAP/ATT）、Wi-Fi 管理帧与分片帧、NFC 数据 | 蓝牙栈（含厂商实现与 BlueDroid 族）、Wi-Fi  supplicant/内核协议栈、NFC 栈 | 攻击者需在无线射程内，多数无需配对或用户交互 | BlueBorne（Android 侧 CVE-2017-0781、CVE-2017-0782；CERT/CC VU#240311）；FragAttacks（Wi-Fi 分片/聚合的设计与实现缺陷） | 蓝牙/Wi-Fi 开关状态（Settings.Global）、框架与厂商 SPL |
| **C4 基带与空中接口** | 空口消息（寻呼、短信的协议层处理）、SIM/USAT 消息 | 调制解调器固件（独立处理器，厂商实现） | 设备已入网且知道号码/IMSI 一类标识；射程即蜂窝网 | Project Zero 2023-03 公开的 Exynos 四个 Internet-to-baseband RCE（CVE-2023-24033、26496、26497、26498） | **只有版本字符串一类代理**（`gsm.version.baseband`），修复到达与否不可从设备判定 |

三点结构性读法：

- **C1/C2 的输入是"应用级"的**，因此防御面包括内容预加载策略、沙箱与进程拆分；**C3/C4 的输入是"协议级"的**，防御面主要是栈实现与固件，用户与 App 无法规避。
- **C4 的落点在其他处理器上**。Google 的基带加固博客把优先级理由写成基带"unique combination of running in an elevated privilege and parsing"不可信输入——即特权 + 不可信解析这两个属性同时成立，这是它被单列的原因，不是因为它 CVE 数量多。
- **同一 CVE 可能横跨通道**。一条空口缺陷若同时导致协议栈状态破坏，公告里可能标 DoS；反之一条媒体解析缺陷可能被 SMS 预加载触发。清点以"输入如何抵达"为主键，不以 CVE 为主键。

---

## 四、通道的横切属性：交互、权限、范围与"零点击"的读法

### 4.1 三个正交属性

任何一条 RCE 类缺陷，描述清楚这三个属性就足够做优先级判断：

1. **前提（precondition）**：需要用户交互吗？需要已加入某个网络/已配对吗？需要设备已开机入网吗？
2. **落点权限（landing privilege）**：执行发生在哪个进程/域，该域能做什么（见第 5 章）。
3. **射程（range）**：互联网可达 / 同一 Wi-Fi / 蓝牙射程 / 蜂窝空口。射程决定**缓解手段归谁**——互联网射程归服务端与内容策略，邻近射程归协议栈与无线开关，空口归基带固件与网络制式。

### 4.2 "零点击"是一个前提属性，不是严重度

公开讨论里"零点击"被当作严重度同义词，这是错的：零点击只说明**没有用户交互这道保险**，执行仍受落点域权限、ASLR/CFI/沙箱等机制约束；反过来，一个需要点击的 RCE 若落在特权解析器里，风险一样高。**公告的表述方式是"with no additional execution privileges needed"加上"user interaction is needed / not needed"两句**——做清单时把这两句原样抽出来即可，不必引入新术语。

### 4.3 为什么"预加载"是内容解析器风险的放大器

C1 通道的历史教训集中在这一点：如果系统会在用户打开之前自动解析收到的内容（预览、缩略图、元数据提取），那么"访问射程"就等价于"发送一条消息"。这也是加固方向从"修解析器"转向"把解析器搬进按应用隔离的沙箱进程"的原因——即使触发前提为无交互，落点权限也被压到单应用粒度。AOSP 的 Media framework hardening 页对此的表述即"Splitting AV pipeline components into app-specific sandboxed processes"并启用相应限制。

---

## 五、执行落点：RCE 之后拿到的是哪一格权限

本节回答"这条通道上的 RCE 值多少钱"。**不展开提权链**（内核篇负责），只建立落点模型。

| 落点类型 | 域/身份特征 | 执行后的直接能力面 | 防御侧含义 |
| --- | --- | --- | --- |
| **应用进程（渲染器、按应用拆分的媒体进程）** | 该应用的 UID、独立 SELinux 域、受限 seccomp | 该应用的数据与 API 能力，跨应用受沙箱约束 | 破坏的是**机密性与完整性**而非整机控制；仍需逃逸（Framework/Binder 报告负责） |
| **系统服务（surfaceflinger、system_server 侧原生组件等）** | 平台域，可访问系统级资源 | 广泛的信息读取与状态篡改，通常仍非 root | 需要平台 SPL + Mainline 通道双看 |
| **厂商 HAL / 守护进程** | vendor 域，可持有器件级权限 | 依实现而定，常涉及音频/显示/传感器/连接性控制 | **厂商 SPL 才是相关指标**；框架 SPL 漂亮不代表这一格已修 |
| **无线/连接栈** | 网络与配对相关权限 | 邻近设备的持久接触能力 | 关无线电是唯一的"立即缓解"，非补丁 |
| **基带固件** | 独立处理器，通常不共享应用侧沙箱 | 蜂窝侧长期驻留能力 | 设备侧不可核验到达性（第 11 章）；缓解靠制式策略与厂商固件 |

三条判断：

- **"RCE = 已 root"是错的**；"RCE 无关紧要因为沙箱会挡"也是错的。正确问法是：这个域能读什么、能持久化什么、能否作为提权链的第一段。
- **域是否真正执法，是可采集的**。SELinux 处于 Permissive 时，域内检查不生效，横向限制只剩沙箱本身——这正是第 14 章脚本把 `getenforce` 与 `ro.build.type` 列为高权重字段的原因：`userdebug`/`eng` 构建出现在生产资产上，同时意味着调试能力与更弱的执法默认值。
- **验证启动状态决定"落点是否可信"**。镜像被修改过的设备，讨论"这个域的权限"失去意义，因此脚本同时采集 `ro.boot.verifiedbootstate` 与 `ro.boot.flash.locked`（语义与密钥栈报告的度量部分一致）。

---

## 六、缺陷类型学：远程输入抵达的解析代码里会出什么错

按**后果可远程触发**的机制归类，与内核篇的类型学互补而不重复。

### 6.1 内存安全类（历史上 Android RCE 的主体）

| 类别 | 触发面 | 公开资料里的典型出处 | 被哪些机制削弱 |
| --- | --- | --- | --- |
| 越界写 / 堆溢出 | 解析长度字段与实收数据不一致 | 媒体框架历史通告、基带研究（Project Zero 的 Exynos 系列即分周公开各类内存缺陷） | CFI/KCFI、HWASan/MTE、BoundSan、分配器加固 |
| 越界读 | 元数据解析、字符串/字段提取 | 公告中的 ID 条目多来自此类，部分可与写原语组合 | 同上；内存安全语言直接消除 |
| 整数溢出 / 回绕 | 尺寸计算、分片长度累加 | 高通公告条目类型即包含 "Integer Overflow or Wraparound"（如 CVE-2026-21385 的芯片侧标题） | IntSan、编译器检查、类型收窄 |
| Use-After-Free | 对象生命周期与异步回调 | GTIG 报告对 2025 年 Intellexa 案例的描述即包含 Android Runtime 的 UAF 类内存损坏 | 引用计数/所有权模型、MTE（标签复用检测）、语言层 |
| 类型混淆 | 跨进程/跨语言边界的反序列化与强转 | Binder 与 Java/JNI 边界的公开研究 | 接口层校验、Stable AIDL（Binder 报告第 20 章）、语言层 |
| 栈破坏 / 返回地址覆盖 | 递归解析、固定缓冲区 | 历史媒体与协议栈案例 | ShadowCallStack（AOSP 页对其的定义：通过另存返回地址来保护 against return address overwrites） |
| 未初始化数据使用 | 状态机与错误分支 | 公告与 sanitizer 报告常见类别 | 初始化检查、MSan 类工具、语言层 |

### 6.2 非内存类（同样能导致执行，且常被认为"更安全"而被忽略）

1. **命令注入与解释器边界**：把不可信字符串拼接进 shell、`exec`、脚本或 SQL；跨进程输入未做类型校验时同样成立（Binder 报告负责传输层，本文负责"输入进了解析器之后"）。
2. **信任假设错误**：把调用方身份当作输入可信度（例如认为"只有系统会调用"）——公告里被标 RCE 但实际射程依赖此类假设的案例并不少。
3. **权限与协议逻辑缺陷**：认证前状态可达、重放与顺序混淆（Wi-Fi 分片/重组类问题即属此类：FragAttacks 的研究主页把其概括为 Wi-Fi 的三个设计缺陷加上广泛存在的实现缺陷）。
4. **不安全的组件暴露**：导出组件、可被远端调用的服务与调试接口。它本身归 EoP/ID 口径，但当暴露的组件接受代码或配置输入时，后果即执行。
5. **更新与安装边界**：签名校验缺失、降级允许。此类缺陷的后果常被标 EoP，但它是"远程投递的持久化"环节——到达性治理（姊妹篇）负责。

### 6.3 类型学的用途

**这张表是给检测覆盖用的，不是给叙事用的。**第 16 章的取证信号与研发侧的 sanitizer/模糊测试配置，都应按"哪个类别 × 哪个通道"打勾；空洞（例如某协议解析器未做 fuzz、某厂商 HAL 未开 CFI）就是下一季度的工程项。跨域方法论报告已给出目标选择评分与测试编排，本文不重复其流程。

---

## 七、公开案例与时间线

只列公开材料完整、可回链的案例族，并给出**它在本文模型里属于哪一格**。案例的价值是结构性的：几乎每一族都推动了平台机制的改变。

| 时间 | 案例族 | 通道 | 公开口径中的类别 | 结构性后果（平台侧改变） |
| --- | --- | --- | --- | --- |
| 2015-08 | Stagefright（媒体框架缺陷群，Android 2.2–5.1.1 版本区间被广泛报道） | C1 | RCE | 触发媒体框架长期加固：按应用拆分沙箱进程、更新节奏与月报机制强化；Google 安全博客后来直接把 2015 年 Stagefright 作为"Critical 媒体框架缺陷大量出现"的起点 |
| 2015-09 | Project Zero《Stagefrightened?》对 CVE-2015-3864 的分析 | C1 | RCE（补丁不完整的再修） | 确立"补丁到达性 ≠ 修复完成"这一验证要求（本文第 13 章指标 4 与其同源） |
| 2016 | QuadRooter（高通驱动族，Check Point 公开） | **不属于本文 RCE 通道** | 公开口径为**本地提权/获取 root**（恶意应用触发），非远程代码执行 | 常见误读来源；纠正见第 18 章误区 1 |
| 2017-09 | BlueBorne（蓝牙族；Android 侧公开编号含 CVE-2017-0781、CVE-2017-0782） | C3 | 远程执行类 | CERT/CC VU#240311 给出 Android 侧界限式表述："All versions of Android prior to September 9, 2017 Security Patch level contain a vulnerable implementation of SDP within the Android Bluetooth software stack"；US-CERT/CISA 同日发布通告 |
| 2021-03 | FragAttacks（Wi-Fi 分片/聚合；研究与工具公开） | C3 | 设计与实现缺陷，公开演示以外泄与注入为主 | 说明**协议设计层**缺陷不能被实现修复覆盖；厂商通告口径不一，需按机型核对 |
| 2023-03 | Exynos 调制解调器四个 Internet-to-baseband RCE（CVE-2023-24033、26496、26497、26498） | C4 | RCE（互联网到基带） | Project Zero 记录 Pixel 在 2023 年 3 月公告中把四条**全部**列出；推动 Google 2023-12 公开基带加固方向与制式侧策略 |
| 2023-12 | Google《Hardening cellular basebands in Android》 | C4 | 机制说明 | 把基带单列为优先加固对象，理由是其特权与不可信解析同时成立；并给出网络制式侧的收紧方向 |
| 2025 | GTIG 关于 Intellexa/Predator 的分析（含 Android Runtime 的 UAF 类内存损坏条目） | C1/落地执行 | 在野利用归因 | 说明商业间谍软件生态仍把移动端内存缺陷作为入口；与 SPL 治理指标（姊妹篇）配合读 |
| 2026-03 | CVE-2026-21385（Android 公告中的 Qualcomm 相关条目；高通公告标题为 "Integer Overflow or Wraparound in Graphics"） | 厂商侧内存损坏（GPU/显示路径，需经本地或内容触发） | 公告注记 "may be under limited, targeted exploitation" | 展示芯片侧缺陷的公告粒度问题：机型映射不公开（第 11 章） |

**读法**：C1 与 C3 的案例有明确的 SPL 界限可供自查；C4 的案例没有；2026 年的这条厂商侧案例则说明"公告里有编号"与"你能核验自己是否受影响"之间仍有距离。**这份时间线的用途是校准防御重心，不是复述攻击史。**

---

## 八、在野利用的统计口径：把"感觉很多"换成数

公开可引用的口径来自 Google Threat Intelligence Group（GTIG）年度与半年度分析：

| 口径 | 数值（公开表述） | 来源 |
| --- | --- | --- |
| 2024 年在野利用零日总数 | "GTIG tracked 75 zero-day vulnerabilities exploited in the wild in 2024" | GTIG 2024 年度报告（PDF） |
| 2025 年在野利用零日总数 | 约 90 起（媒体报道引 GTIG 表述） | GTIG 2025 年度回顾 |
| **移动相关零日的三年走势** | 2023 年 17 起 → 2024 年 9 起 → 2025 年 15 起 | GTIG 2025 年度回顾 |
| 零日利用的节奏 | 月均从 2025 年的 8 起上升到当期的 11 起（AI 时代趋势报告，2026-09-30） | GTIG 趋势报告 |

三条解释原则：

1. **移动零日数量下降不等于移动端风险下降**。分母是发现与披露量，且 GTIG 自身记录 2025 年回升；单年数字用于趋势而非结论。
2. **"在野利用"的归因粒度与公告不同**。年度报告按攻击者与平台归类，公告按 CVE 与组件归类；把两者对应起来要靠第 13 章的指标而不是靠印象。
3. **Android TAG 长期公开其 0-day 狩猎工作**（"routinely hunts for 0-day vulnerabilities exploited in-the-wild"），这意味着一旦某条 Android 缺陷被公开点名，通常已晚于可用的缓解窗口——防御侧的结论只能是：**别等被点名，按通道建设到达性。**

---

## 九、缓解机制矩阵：每层挡什么、挡不住什么

下表按"机制 → 被削弱的类别 → 不覆盖的部分"给出。版本归属以 AOSP 的 Security enhancements 页与专项页为准（该页按发布版本列出机制，例如把 HWASan 标注为 Android 10 引入），**厂商是否在你的机型上开启是另一回事**（硬件组件报告负责器件侧差异）。

| 机制 | 削弱什么 | 挡不住什么 | 防御侧可核验证据 |
| --- | --- | --- | --- |
| **进程拆分 + 按应用沙箱**（媒体框架加固） | 把 C1 通道的落点权限从系统级降到应用级 | 单应用内的执行本身；逃逸与逻辑缺陷 | 落点进程命名与域（有授权时）；机型与 SPL |
| **SELinux 强制模式** | 执行后的横向移动与资源访问 | 域内允许的行为；Permissive 机型 | `getenforce`（脚本 1 的 FIND_selinux） |
| **seccomp 系统调用过滤** | 落点域可用的内核能力面 | 过滤列表内的合法调用被滥用 | 构建版本与域策略（源码侧核对） |
| **CFI / KCFI** | 面向控制流劫持的利用（虚表/函数指针改写） | 不需要控制流劫持的缺陷（逻辑、信息泄露后再利用）；未编译进目标二进制的代码 | 由构建决定，设备侧不可直接读取；核对 ROM 构建说明 |
| **ShadowCallStack** | 返回地址覆盖类（栈溢出后果） | 非返回地址的写原语 | 同上 |
| **HWASan（Android 10 起）** | 堆栈越界与 UAF 的**检测与拦截**（生产侧此前主要用于测试） | 未插桩模块 | 测试覆盖记录 |
| **Arm MTE（内存标签扩展）** | 标签不匹配的使用（UAF/越界在标签粒度上被拒） | 未开启或未标记对象；硬件可用性依机型 | 机型/SoC 支持矩阵（硬件组件报告） |
| **IntSan / BoundSan** | 整数溢出与边界计算 | 语义错误（计算正确但假设错误） | 构建说明 |
| **内存安全语言（Rust/C++ 内存安全子集）** | 直接消除一类内存安全缺陷 | 非内存类（第 6.2 节全部）；unsafe 边界与 FFI | 组件实现语言（源码侧） |
| **Mainline/APEX 模块化更新** | 缩短修复到达时间 | 厂商替换/禁用的模块；基带与 HAL | `/apex` 挂载观察（脚本 1）+ 补丁链报告口径 |
| **验证启动与锁定状态** | 镜像完整性，防止落点被替换 | 运行时内存缺陷本身 | `ro.boot.verifiedbootstate`、`ro.boot.flash.locked` |
| **无线电/制式策略** | 邻近与空口射程（C3/C4） | 已入网路径与固件缺陷 | `bluetooth_on`、`wifi_on`、基带版本字符串 |

**读法**：矩阵右两列决定工程预算。若某条通道上的解析器既没有插桩也没有模块化更新（典型是厂商 HAL 与基带），那么对该通道**只能靠"关闭 + 固件到达性"两条**，其余机制的讨论对它不成立。

---

## 十、Mainline 与 APEX：RCE 修复的第二条到达路径

补丁链报告已论证通道划分，本节只谈对 RCE 清点的影响：

- **可经模块到达的组件**：媒体、网络栈（tethering）、TLS（conscrypt）、运行时（art）等以 APEX 形式提供更新；AOSP 的 Mainline 页面说明更新后的模块"can be packaged together and pushed to end-user devices, either by Google, using the Google Play system update feature, or by the"设备厂商。这与本文 C1/C3 通道高度重叠——**这些通道的修复到达时间可能早于 SPL**。
- **反向也成立**：SPL 显示当月，不代表厂商替换过的模块是最新版；模块被企业策略或 ROM 禁用时，这条腿直接消失（第 17 章）。
- **可观测边界**：无 root 的设备上能稳定采到的是 `/apex` 下的挂载项存在性，不是模块版本。因此脚本 1 对每个关注模块给出"观察到/未观察到"，并在未观察到时明确写"不得据此判定该模块已更新或未更新"——**把采不到记成数据缺口，而不是记成合规**。
- WebView 常被误当作 APEX 模块。本文不采纳该说法：脚本读取的是包版本信息，其可用性依机型与授权，采不到时同样单列（FIND_webview 的 L0-LOW 分支）。

---

## 十一、基带与近场无线：设备侧核验不到的一段

这是全篇唯一"防御方无法自证"的一段，必须写得比别处更明确。

**基带（C4）**

- **可达性**：射程即蜂窝网络。Project Zero 2023 年的系列标题直接写作 Internet-to-baseband，说明"从互联网到基带"的路径被公开确认过。
- **修复归属**：固件在厂商侧，公告常按技术域组织而不给机型映射（补丁链报告第 8 章论证过这条粒度差）。
- **设备侧能采到什么**：只有版本字符串一类字段（脚本 1 的 `gsm.version.baseband`）。脚本因此把它的判定写成"该字段只标识版本，不代表 SPL 覆盖基带，需与厂商公告逐条核对"；采不到则写"无法从设备判定，需向厂商索取机型映射"。
- **能立即做的缓解**：关闭/限制低代制式回退、拒绝未知来源的空中接口配置、按机型维护厂商固件说明与基带版本的对应表。Google 的基带加固博客记录了把基带单列优先加固的理由与收紧方向（其 2G 相关策略亦被后续厂商与媒体报道记录，本文标为二手来源）。

**近场无线（C3）**

- **可采、可控，但只有开关粒度**：`bluetooth_on` / `wifi_on` 可读（Settings.Global），可关。但"关闭"依赖使用者配合，是治理问题而不是技术边界。
- **协议实现层不可从设备核对**：SDP/ATT 的修复在栈实现里，只能靠 SPL + 机型公告。BlueBorne 之所以给出明确界限（2017-09-09 SPL 之前），是因为当时 Android 侧的蓝牙栈在 AOSP 中；**换成厂商栈实现后，同样的界限表述就不再存在**。这是第 18 章误区 6 的实质。

**结论**：C4 与 C3 的厂商实现部分，在设备侧只能采到代理指标。**任何声称"我们从设备上验证了基带已修复"的说法都应被要求出示厂商映射证据。**

---

## 十二、内存安全的宏观数据：为什么这一类缺陷曾经是主体

三组公开数字，按口径各自解释：

1. **代码构成**：AOSP 的 Memory safety 页记录，以 C、C++、汇编等非内存安全语言编写的原生代码"represent over 70% of the Android platform code"，并且这类代码在严重漏洞构成中占相当比例（具体百分比以该页当期文本为准）。**含义**：只要平台仍有大量原生解析器，C1/C3 的缺陷生成速率就不会归零。
2. **缺陷构成变化**：Google 安全博客（Android 13 内存安全语言一文）记录内存安全类缺陷占 Android 总漏洞的比例从 2019 年的 76% 降到 2022 年的 35%，并指出 2022 年是第一年（该表述指内存安全类不再是多数）。**含义**：这条曲线是"缓解有效"的最强证据，也说明剩余风险正在向非内存类（第 6.2 节）与厂商侧迁移。
3. **语言密度**：2025-11 的官方博客记录 Rust 代码的内存安全漏洞密度相对 C/C++ 有"1000x reduction"。**含义**：新组件的实现语言是**可询问、可核对**的工程属性，采购与自研评估都该把它列进问卷；但它不改变"边界与逻辑缺陷仍在"的事实。

**治理推论**：若一家 OEM 的公开材料里同时有"内存安全语言占比上升"与"厂商 SPL 长期落后"，那么真实风险取决于后者——机制再好，没推到设备上就不生效（补丁链报告的核心结论在本文的对应物）。

---

## 十三、面向 RCE 的七项运营指标

每项给定义、数据来源、建议阈值与**会被钻空子的地方**。阈值按企业池经验给出，需按业务基线校准。

| # | 指标 | 数据来源 | 建议阈值 | 会被钻空子的地方 |
| --- | --- | --- | --- | --- |
| 1 | **通道覆盖完整率** | 第 3 章四通道 × 机型清单 | 100%（缺格即数据缺口） | 只统计有 CVE 的通道，忽略采不到的 C4 |
| 2 | **框架 SPL 落后月数** | `ro.build.version.security_patch` | P1 设备 ≤1 个月 | 用 `-01` 档冒充 `-05` 档（补丁链报告第 2.1 节） |
| 3 | **厂商侧 SPL 与差值** | `ro.vendor.build.version.security_patch` | 落后 ≤2 个月，差值 ≤1 个月 | 属性缺失记 0；应单列为缺口 |
| 4 | **落点执法状态** | `getenforce`、`ro.build.type`、`ro.debuggable` | Permissive/userdebug/eng 出现即 P1 事件 | 用"测试设备"名义长期留在生产池（脚本 1 的 FIND_build_type 专治此项） |
| 5 | **C2 内核滞后跨度** | WebView 版本 vs 当期 Chromium 稳定版 | ≤1 个稳定周期 | 采不到版本时跳过；应记 MED 单列（脚本 2 的 `webview_unknown`） |
| 6 | **模块化通道可用性** | `/apex` 观察 + EMM 策略是否禁用 Play 系统更新 | 禁用需例外审批 | 只看 SPL 忽略这条更早到达的腿 |
| 7 | **C4 到达性证据率** | 机型 → 厂商基带固件说明的映射覆盖 | 覆盖 ≥90% 机型；未覆盖单列 | 把"设备采不到"读成"没问题"（脚本 2 的 `baseband_unobservable`） |

**指标间的取舍要写进政策**：1–3 回答"输入是否可达且未修"，4 回答"落点是否真被约束"，5–6 回答"两条更新腿是否都在走"，7 回答"我们承认自己看不见什么"。**只汇报第 2 项的 RCE 治理，等价于把 C3/C4 从报表里删掉。**

---

## 十四、设备侧核验之一：远程可达面与执法状态采集

**定位**：单台设备的一次只读快照，输出"采集值 + 可回溯判定"。用途有三：新机验收、事故分诊时快速定性、以及为指标 2–7 提供原始数据。全部字段只读，不写入设备、不安装代理。

**前置条件**：设备已授权调试连接（自有或获授权资产）；脚本不含任何触发式探测。

**用法**

```
用法: rce-surface.sh <设备序列号> <基准日期 YYYY-MM-DD>
示例: rce-surface.sh 9A1B2C 2026-10-05
输出: 先输出 key,value 采集行，再输出 FIND_* 判定行；判定可回溯到上面的采集值
退出码: 0 正常；1 设备不可达；2 基准日期格式错误
```

**脚本**

```bash
#!/usr/bin/env bash
# 脚本 1: 远程可达面与执法状态采集（只读）
# 用法: rce-surface.sh <设备序列号> <基准日期 YYYY-MM-DD>
# 输出: key,value 与 FIND_*；判定可回溯到上面的采集值
set -u
dev="${1:?用法: rce-surface.sh <serial> <YYYY-MM-DD>}"
ref="${2:?缺少基准日期，格式 YYYY-MM-DD}"

adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable" >&2; exit 1; }

p()  { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }
q()  { adb -s "$dev" shell "toybox $1" 2>/dev/null | tr -d '\r'; }
s()  { adb -s "$dev" shell settings get "$1" "$2" 2>/dev/null | tr -d '\r'; }
row() { printf '%s,%s\n' "$1" "$2"; }

mi() {
  case "$1" in
    [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]) printf '%s' $(( 10#${1:0:4} * 12 + 10#${1:5:2} )) ;;
    *) printf '' ;;
  esac
}

spl="$(p ro.build.version.security_patch)"
vspl="$(p ro.vendor.build.version.security_patch)"
btype="$(p ro.build.type)"
dbg="$(p ro.debuggable)"
model="$(p ro.product.model)"
fp="$(p ro.build.fingerprint)"
radio="$(p gsm.version.baseband)"
vb="$(p ro.boot.verifiedbootstate)"
lock="$(p ro.boot.flash.locked)"
enforce="$(adb -s "$dev" shell getenforce 2>/dev/null | tr -d '\r')"
apex_list="$(q "ls /apex")"
wv="$(adb -s "$dev" shell dumpsys package com.android.webview 2>/dev/null | grep -m1 -o 'versionName=[0-9.]*' | cut -d= -f2 | tr -d '\r')"
bt="$(s global bluetooth_on)"
wifi="$(s global wifi_on)"

row model "$model"; row spl "$spl"; row vendor_spl "$vspl"
row build_type "$btype"; row debuggable "$dbg"; row fingerprint "$fp"
row baseband "$radio"; row verifiedbootstate "$vb"; row flash_locked "$lock"
row selinux "$enforce"; row webview_version "$wv"
row bluetooth_on "$bt"; row wifi_on "$wifi"

ref_m="$(mi "$ref")"
[ -n "$ref_m" ] || { echo "ERR,bad_reference_date" >&2; exit 2; }

if [ -z "$spl" ]; then
  row FIND_spl "L0-LOW 未采集到框架 SPL，不得记为已修复"
else
  gap=$(( ref_m - $(mi "$spl") ))
  if [ "$gap" -gt 3 ]; then
    row FIND_spl "L3-HIGH 框架 SPL 落后基准约 ${gap} 个月，远程可达面按未修复计"
  elif [ "$gap" -gt 1 ]; then
    row FIND_spl "L2-WARN 框架 SPL 落后约 ${gap} 个月"
  else
    row FIND_spl "L1-INFO 框架 SPL 与基准同月或仅差 1 个月"
  fi
fi

case "$btype" in
  user)      row FIND_build_type "L1-INFO user 构建：调试面按最小化计" ;;
  userdebug) row FIND_build_type "L3-HIGH userdebug 构建出现在生产资产上：adb root 与 SELinux 关闭能力存在" ;;
  eng)       row FIND_build_type "L3-HIGH eng 构建：默认开放调试能力" ;;
  *)         row FIND_build_type "L0-LOW 未采集到构建类型，无法判断调试面" ;;
esac

case "$enforce" in
  Enforcing) row FIND_selinux "L1-INFO SELinux Enforcing" ;;
  Permissive) row FIND_selinux "L3-HIGH SELinux Permissive：域内检查不生效，进程内 RCE 的横向限制仅剩沙箱本身" ;;
  *) row FIND_selinux "L0-LOW 未采集到 SELinux 模式（授权受限），不得记为 Enforcing" ;;
esac

case "$vb" in
  green)  row FIND_lock "L1-INFO 验证启动 green 且锁定状态=${lock:-未知}" ;;
  orange) row FIND_lock "L2-WARN 验证启动 orange（引导加载器已解锁），镜像完整性不作保证" ;;
  red|yellow) row FIND_lock "L3-HIGH 验证启动 ${vb}：度量链异常" ;;
  *)     row FIND_lock "L0-LOW 未采集到验证启动状态，需按机型说明核对字段可用性" ;;
esac

if [ -z "$radio" ]; then
  row FIND_baseband "L0-LOW 未采集到基带版本字符串；基带固件是否含当期修复无法从设备判定，需向厂商索取机型映射"
else
  row FIND_baseband "L1-INFO 采到基带版本字符串（${radio}）；该字段只标识版本，不代表 SPL 覆盖基带，需与厂商公告逐条核对"
fi

if [ -z "$wv" ]; then
  row FIND_webview "L0-LOW 未采集到 WebView 版本（包名或授权差异）；浏览器内核是公开在野利用最多的入口之一，此项缺失必须单列而不是略过"
else
  row FIND_webview "L1-INFO WebView 版本 ${wv}；按当期 Chromium 稳定版本核对滞后跨度"
fi

for m in com.android.media com.android.tethering com.android.conscrypt com.android.art; do
  if printf '%s\n' "$apex_list" | grep -q "$m"; then
    row FIND_apex "$m 观察到挂载项"
  else
    row FIND_apex "L0-LOW 未观察到 ${m}；不得据此判定该模块已更新或未更新"
  fi
done

for k in bluetooth_on wifi_on; do
  v="$( [ "$k" = bluetooth_on ] && printf '%s' "$bt" || printf '%s' "$wifi" )"
  case "$v" in
    1|true|null|"") row FIND_surface "L1-INFO ${k}=${v:-未知}，按开启计入可达面" ;;
    0|false)        row FIND_surface "L1-INFO ${k}=${v}，该无线面在采集时关闭" ;;
    *)              row FIND_surface "L0-LOW ${k} 取值无法解析（${v}）" ;;
  esac
done
```

**设计说明**

- **L 级只是本脚本的处置便利记号**：L3-HIGH 需立即处置、L2-WARN 需查因、L1-INFO 记录、L0-LOW 数据缺口。**L0 不等于通过**，这是本脚本与常见检查脚本的最大差异。
- `mi()` 只取年月，判定粒度是**月**；要按天考核请在上层用 `spl` 与当期公告日做日期差。
- `null` 与空值按"开启"计入可达面：`settings get` 在键未设置时返回 `null`，而可达面应当按最保守方向计算。
- 基带项**刻意不给结论**。把"未采到"写成 L0-LOW 并要求厂商映射，是为了让这条无法自证的路径在报告里保持可见。

**样例输出（模拟设备桩验证，非真实设备数据）**

健康设备（SPL 与基准同月、user 构建、Enforcing、green、四个关注模块均观察到）：

```
FIND_spl,L1-INFO 框架 SPL 与基准同月或仅差 1 个月
FIND_build_type,L1-INFO user 构建：调试面按最小化计
FIND_selinux,L1-INFO SELinux Enforcing
FIND_lock,L1-INFO 验证启动 green 且锁定状态=1
FIND_baseband,L1-INFO 采到基带版本字符串（B22.26-1）；该字段只标识版本，不代表 SPL 覆盖基带，需与厂商公告逐条核对
FIND_webview,L1-INFO WebView 版本 139.0.7258.153；按当期 Chromium 稳定版本核对滞后跨度
FIND_apex,com.android.media 观察到挂载项
FIND_surface,L1-INFO bluetooth_on=0，该无线面在采集时关闭
FIND_surface,L1-INFO wifi_on=1，按开启计入可达面
```

异常设备（SPL 落后 18 个月、userdebug、Permissive、orange、模块与基带字段缺失）：

```
FIND_spl,L3-HIGH 框架 SPL 落后基准约 18 个月，远程可达面按未修复计
FIND_build_type,L3-HIGH userdebug 构建出现在生产资产上：adb root 与 SELinux 关闭能力存在
FIND_selinux,L3-HIGH SELinux Permissive：域内检查不生效，进程内 RCE 的横向限制仅剩沙箱本身
FIND_lock,L2-WARN 验证启动 orange（引导加载器已解锁），镜像完整性不作保证
FIND_baseband,L0-LOW 未采集到基带版本字符串；基带固件是否含当期修复无法从设备判定，需向厂商索取机型映射
FIND_apex,L0-LOW 未观察到 com.android.media；不得据此判定该模块已更新或未更新
```

不可达设备输出 `ERR,device_unreachable`（退出码 1），基准日期非法输出 `ERR,bad_reference_date`（退出码 2）。

---

## 十五、设备侧核验之二：fleet 基线断言

**定位**：把第 13 章指标 2、4、5、7 变成一次可回归的断言。每台设备按**型号**取基线行，逐项判定，最后给出汇总计数——用于验收、季度复审与事故池筛查。

**基线文件格式**（每行一型，字段留空表示该机型不断言此项）

```
# model,minSPL,minWebviewMajor,buildType,selinux
Pixel-G,2026-09-05,130,user,Enforcing
Pixel-B,2026-09-05,130,user,Enforcing
Pixel-C,,130,user,Enforcing
```

**用法**

```
用法: rce-fleet-assert.sh <基线文件> <设备序列号> [设备序列号 ...]
退出码: 0 正常（含发现不合规）；1 基线文件缺失或未提供设备序列号
```

**脚本**

```bash
#!/usr/bin/env bash
# 脚本 2: fleet 远程可达面基线断言（只读）
# 用法: rce-fleet-assert.sh <基线文件> <设备序列号> [设备序列号 ...]
# 基线文件每行: 型号,最低框架SPL,WebView最低版本,要求构建类型,要求SELinux
#   例: Pixel Test A,2026-09-05,130,user,Enforcing
#   某字段留空表示该机型不断言此项
set -u
base="${1:?用法: rce-fleet-assert.sh <baseline-file> <serial> ...}"
shift
[ -f "$base" ] || { echo "ERR,baseline_missing,$base" >&2; exit 1; }
[ "$#" -gt 0 ] || { echo "ERR,no_device" >&2; exit 1; }

p()  { adb -s "$1" shell getprop "$2" 2>/dev/null | tr -d '\r'; }
hi=0; md=0; lo=0; ok=0

for dev in "$@"; do
  if ! adb -s "$dev" get-state >/dev/null 2>&1; then
    printf 'LOW,%s,unreachable,设备不可达\n' "$dev"; lo=$((lo+1)); continue
  fi
  model="$(p "$dev" ro.product.model)"
  if [ -z "$model" ]; then
    printf 'LOW,%s,model_missing,型号属性为空，无法安全匹配基线行\n' "$dev"
    lo=$((lo+1)); continue
  fi
  line="$(grep -m1 -F "$model," "$base" || true)"
  if [ -z "$line" ]; then
    printf 'LOW,%s,no_baseline,型号=%s,基线未覆盖该型号\n' "$dev" "$model"
    lo=$((lo+1)); continue
  fi
  want_spl="$(printf '%s' "$line" | cut -d, -f2)"
  want_wv="$(printf '%s' "$line" | cut -d, -f3)"
  want_bt="$(printf '%s' "$line" | cut -d, -f4)"
  want_se="$(printf '%s' "$line" | cut -d, -f5)"

  spl="$(p "$dev" ro.build.version.security_patch)"
  btype="$(p "$dev" ro.build.type)"
  se="$(adb -s "$dev" shell getenforce 2>/dev/null | tr -d '\r')"
  wv="$(adb -s "$dev" shell dumpsys package com.android.webview 2>/dev/null | grep -m1 -o 'versionName=[0-9.]*' | cut -d= -f2)"
  radio="$(p "$dev" gsm.version.baseband)"

  if [ -z "$want_spl" ]; then
    printf 'LOW,%s,spl_baseline_absent,该项未设基线，不得记为通过\n' "$dev"; lo=$((lo+1))
  elif [ -z "$spl" ]; then
    printf 'LOW,%s,spl_missing,不得记为通过\n' "$dev"; lo=$((lo+1))
  elif [[ "$spl" < "$want_spl" ]]; then
    printf 'HIGH,%s,spl_below_baseline,got=%s,min=%s\n' "$dev" "$spl" "$want_spl"; hi=$((hi+1))
  else
    printf 'INFO,%s,spl_ok,got=%s\n' "$dev" "$spl"; ok=$((ok+1))
  fi

  if [ -n "$want_bt" ]; then
    if [ -z "$btype" ]; then
      printf 'LOW,%s,build_type_missing,要求=%s\n' "$dev" "$want_bt"; lo=$((lo+1))
    elif [ "$btype" != "$want_bt" ]; then
      printf 'HIGH,%s,build_type_mismatch,got=%s,want=%s,调试面与 adb root 能力随构建类型变化\n' "$dev" "$btype" "$want_bt"; hi=$((hi+1))
    else
      printf 'INFO,%s,build_type_ok,got=%s\n' "$dev" "$btype"; ok=$((ok+1))
    fi
  fi

  if [ -n "$want_se" ]; then
    case "$se" in
      "$want_se") printf 'INFO,%s,selinux_ok,got=%s\n' "$dev" "$se"; ok=$((ok+1)) ;;
      '')         printf 'LOW,%s,selinux_unknown,授权受限，不得记为 Enforcing\n' "$dev"; lo=$((lo+1)) ;;
      *)          printf 'HIGH,%s,selinux_mismatch,got=%s,want=%s\n' "$dev" "$se" "$want_se"; hi=$((hi+1)) ;;
    esac
  fi

  if [ -n "$want_wv" ]; then
    if [ -z "$wv" ]; then
      printf 'MED,%s,webview_unknown,最低要求=%s，采集不到版本要单列\n' "$dev" "$want_wv"; md=$((md+1))
    else
      major="$(printf '%s' "$wv" | cut -d. -f1)"
      if [ "$major" -lt "$want_wv" ] 2>/dev/null; then
        printf 'HIGH,%s,webview_outdated,got=%s,min=%s\n' "$dev" "$wv" "$want_wv"; hi=$((hi+1))
      else
        printf 'INFO,%s,webview_ok,got=%s\n' "$dev" "$wv"; ok=$((ok+1))
      fi
    fi
  fi

  if [ -z "$radio" ]; then
    printf 'LOW,%s,baseband_unobservable,基带侧到达性无法从设备判定\n' "$dev"; lo=$((lo+1))
  else
    printf 'INFO,%s,baseband_recorded,value=%s\n' "$dev" "$radio"; ok=$((ok+1))
  fi
done

printf 'SUMMARY,high=%s,med=%s,low=%s,ok=%s,devices=%s\n' "$hi" "$md" "$lo" "$ok" "$#"
```

**样例输出（模拟设备桩：一台合规、一台多项落后、一台未设 SPL 基线且 WebView 采不到、一台不可达）**

```
INFO,GOOD001,spl_ok,got=2026-10-05
INFO,GOOD001,build_type_ok,got=user
INFO,GOOD001,selinux_ok,got=Enforcing
INFO,GOOD001,webview_ok,got=139.0.7258.153
INFO,GOOD001,baseband_recorded,value=B22.26-1
HIGH,BAD001,spl_below_baseline,got=2025-04-05,min=2026-09-05
HIGH,BAD001,build_type_mismatch,got=userdebug,want=user,调试面与 adb root 能力随构建类型变化
HIGH,BAD001,selinux_mismatch,got=Permissive,want=Enforcing
HIGH,BAD001,webview_outdated,got=118.0.1234.5,min=130
LOW,BAD001,baseband_unobservable,基带侧到达性无法从设备判定
LOW,MIX001,spl_baseline_absent,该项未设基线，不得记为通过
MED,MIX001,webview_unknown,最低要求=130，采集不到版本要单列
LOW,GHOST001,unreachable,设备不可达
SUMMARY,high=4,med=1,low=3,ok=8,devices=4
```

**设计与边界**

- **按型号匹配，不按设备匹配**：`grep -m1 -F "$model,"` 用固定串并带上逗号，避免型号互为前缀时错配；型号属性为空单独报 `model_missing` 而不是落到"未匹配"分支，避免把采集失败当成合规失败。
- **基线留空 ≠ 通过**：`spl_baseline_absent` 记 LOW。这一条把"忘了设基线"从统计里显式暴露出来。
- **计数是断言数不是设备数**：`devices=4` 与 `high=4` 并存是正常的（一台设备可贡献多条 HIGH）。看板时用 `SUMMARY` 定位，用明细行处置。
- **本脚本只做正向断言**：不修改任何设置、不下发策略、不做触发式探测。策略下发属于 EMM，不属于验证脚本。

---

## 十六、检测与取证信号：一次成功的远程执行会留下什么

分两档：**设备侧可得的信号**与**必须依赖外部证据的信号**。前者是覆盖率建设的起点，后者决定你需不需要 MDM/网络侧日志。

### 16.1 设备侧可得

| 信号 | 采法（只读） | 解读与误报 |
| --- | --- | --- |
| **原生崩溃转储** | `adb bugreport`；tombstone/Dropbox 类记录（依权限） | 同一进程在短时间内反复崩溃，尤其在收到消息/内容之后，是内存安全类缺陷被触发的**最常见可观测痕迹**；误报源：不稳定的第三方库与厂商栈自身缺陷 |
| **落点进程异常重启** | `dumpsys` 快照对比、进程存在性 | 媒体/蓝牙/Wi-Fi 栈守护进程重启而无人操作，值得立案核查；不能单独定性 |
| **SELinux 拒绝与模式** | `getenforce`；`dmesg`/日志中的 avc 记录（依权限） | **异常静默**也可能是信号：本应产生 avc denied 的行为突然没有记录，需查策略与模式 |
| **SPL/构建/指纹不一致** | 脚本 1/2 | SPL 晚于构建日期、同机型同 SPL 但 incremental 分散，属"声明可信度"问题（补丁链报告第 12 章同源） |
| **异常持久化** | 无障碍服务/设备管理器/未知应用的安装记录、开机自启项 | 远程执行后的常见第二步是持久化；这一项在应用层可见，是最实用的用户侧线索 |
| **无线电与制式状态变化** | `settings get global bluetooth_on/wifi_on`、基带版本 | 与用户行为不符的开关变化，或基带版本回退，需要与固件更新记录区分 |
| **流量与功耗异常** | 网络侧统计（设备侧只读能力有限） | C2/C1 落地后的外联与数据回传常表现为后台流量抬升 |

### 16.2 必须依赖外部证据

- **投递内容本身**：邮件/短信/网页的原始副本与时间线；设备上没有"我解析过哪个文件"的完整日志。
- **基带与固件侧**：设备侧不可得，只能要厂商说明（第 11 章）。
- **利用代码存在性**：靠内存取证与专项工具，超出本文范围；本报告不含任何构造性细节。

### 16.3 覆盖矩阵（建议按此排工程）

四通道 × 三档信号（崩溃 / 状态变化 / 持久化）共十二格，逐格标注：已覆盖、部分覆盖、**不可观测**。经验上"不可观测"集中在 C4 全行与 C3 的固件侧——把它们写进风险接受声明，而不是假装覆盖。

---

## 十七、企业策略：能控与不能控

| 你想做的事 | 是否可由策略达成 | 依据与限制 |
| --- | --- | --- |
| 关闭/限制低代蜂窝制式回退 | 可以（Android 14 起的管理侧选项，Google 基带加固方向与其后媒体报道记录；EMM 字段名不统一） | 只影响 C4 的回退面，不影响已入网路径；需按机型验证策略实际生效 |
| 关闭蓝牙/Wi-Fi 并要求审批 | 可以 | 属 C3 立即缓解；脚本 1 可复核采集时状态 |
| 强制 Play 系统更新/模块化更新 | 部分可以 | 厂商禁用或未提供该通道时无效；脚本 1/2 用 `/apex` 存在性作粗粒度代理 |
| 固定 WebView 版本、阻止自动更新 | **不作为可达项**（未公开文档化为标准策略能力） | 需要的是"保证最新版本"而非"锁定版本"；核对方式取版本字符串，采不到单列 |
| 强制 SELinux Enforcing、拒绝 userdebug 构建 | 靠采购与验收达成，非运行时策略 | 脚本 1/2 把两者作为验收断言（FIND_selinux / FIND_build_type） |
| 确认基带固件已含某条修复 | **不能从设备达成** | 唯一路径是厂商的机型—固件映射；无映射则记为数据缺口（指标 7） |
| 验证某条 CVE 是否影响 fleet | 靠公告分区 + 组件 + 机型交叉 | 公告不给出机型清单；需与厂商更新说明联合判断 |

**一条实践建议**：把第 15 章的基线文件放进版本控制，和 fleet 台账同批评审。**基线变更本身应当是一次可追溯的决策**，而不是散落在邮件里的口头例外。

---

## 十八、常见误区

1. **"QuadRooter 是 Android 远程代码执行"**。公开口径是高通驱动族缺陷使恶意应用可**本地提权获取 root**（Check Point 2016 与后续厂商/媒体一致），不是网络侧执行。它属于第 5 章的"落点之后"，不属于第 3 章的任一通道。
2. **"标了 RCE 就是零点击"**。公告把"需要额外执行权限"与"需要用户交互"分开表述；RCE 只说后果。见第 4.2 节。
3. **"SPL 当月 = 远程可达面已修"**。SPL 不覆盖基带与安全侧固件，也不反映厂商替换过的模块实际版本（补丁链报告第 2.4 节的四个"不能"在此全部适用）。
4. **"Mainline/APEX 修了，所以 SPL 落后无所谓"**。方向对了一半：C1/C3 上确实可能已修；但**反向不成立**——SPL 漂亮不代表模块是最新，也不代表厂商没禁用该通道。
5. **"WebView 是 APEX 模块，所以能从 `/apex` 看到版本"**。本文不采纳该说法：版本从包信息读取，采不到时按数据缺口处理；把采集失败读成"没问题"是脚本设计要防的事。
6. **"关蓝牙就能防住 C3"**。采集时刻的开关状态只是快照，且历史案例（BlueBorne）里多数触发并不需要用户可见的操作；同时**蓝牙栈实现若来自厂商，SPL 界限式表述不再适用**——第 11 章。
7. **"基带缺陷只能等厂商，所以不用管"**。可达的缓解是制式策略、拒绝未知空中配置、以及把"无机型映射"写进风险声明；第 17 章。
8. **"Critical 一定是远程；本地缺陷不重要"**。Critical 判据包含 TEE/SE 中任意执行代码这类情形，与射程无关；本地落点如果给出安全侧权限，处置优先级可以高于一个沙箱内 RCE。
9. **"Permissive 只是测试环境的问题，不影响 RCE 风险"**。Permissive 时域内检查不生效，执行后的横向限制只剩沙箱本身——这正是脚本把它列为 L3-HIGH 的理由。
10. **"没有公开 PoC 就没有在野风险"**。GTIG 口径下的在野零日长期存在（2024 年 75 起、2025 年约 90 起，移动相关 17/9/15），而公告措辞是"indications … may be under limited, targeted exploitation"；防御排期不能以 PoC 公开为触发条件。
11. **"CPU/侧信道类缺陷属于 RCE"**。它们通常被归为信息泄露或跨特权边界读取，与"远程输入导致执行"不是同一类；混用会污染通道清单。
12. **"设备采不到 = 设备没问题"**。本报告的 L0-LOW/MED 分支全部为此而设：`spl_missing`、`selinux_unknown`、`webview_unknown`、`baseband_unobservable` 都要单列并进入处置队列。

---

## 十九、趋势判断

1. **内容解析器风险正在从"单点 CVE"转为"落点权限"问题**。按应用拆分沙箱之后，同一条解析缺陷的边际收益下降；相应地，**厂商 HAL 与闭源守护进程**成为同等可达但落点更高的部分，而这部分的公告粒度最粗。
2. **内存安全类占比继续下降，非内存类占比相对上升**。76% → 35% 的曲线与语言密度数据指向同一方向：下一批远程可达缺陷的构成会更偏向逻辑、信任假设与协议设计层（第 6.2 节），这些**不会被 CFI/MTE 削弱**，只能靠接口契约、状态机审查与差分测试。
3. **模块化更新会继续吸收 C1/C3 的一部分修复到达时间**，但这会加大"SPL 与真实状态不一致"的幅度，**指标必须按通道拆开**——只看一个日期的治理会在两个方向上同时出错。
4. **基带与卫星/新空口面会持续成为公开研究重点**。2023 年的 Exynos 公开把这一域从"研究者的角落"变成厂商必须回应的条目类型；随后的加固说明与制式策略是平台侧回应，而**厂商侧的机型映射公开程度不会自动改善**——采购问卷里要主动问。
5. **AI 辅助发现会提高披露密度**。GTIG 2026-09 趋势记录月均在野利用零日从 8 起升至 11 起；这意味着**验证到达性的频率**（而非公告阅读）成为防御侧的真实瓶颈——第 14–15 章脚本的价值在于可以把核验做成例行。
6. **企业侧的可控面在扩大但仍不含 C4**。制式策略与无线电策略是可下发的；固件到达性不是。**"能关的"与"能修的"要分开写进政策**，否则例外审批会集中在最不可见的一段。

---

## 二十、结论与分角色处置

**总判断**：Android 上的远程代码执行风险，**结构性地取决于三件事**——不可信输入抵达哪些解析器（第 3 章）、这些解析器落在什么权限与什么执法状态下（第 5 章、第 9 章），以及修复有没有真的推到设备上（第 10–11 章与补丁链姊妹篇）。第一件可以穷举，第二件可以核验，第三件里基带那一段目前只能要求供应商给证据。

**平台/ROM 研发**：先做四通道 × 解析器清单（含厂商闭源组件），再按第 6 章类型学核对每格的机制覆盖（CFI/SCS/MTE/sanitizer/fuzz/语言），最后把"不可观测格"写进风险声明。**新增解析器一律要求：给出落点域、给出输入到达路径、给出模糊测试覆盖。**

**设备安全与补丁治理**：以第 13 章七项指标为报表基线；**指标 1 与指标 7 必须存在**，否则报表会把 C3/C4 静默删除。验收脚本用第 15 章，并把基线文件纳入版本控制。

**安全运营 / EMM**：把无线电与制式策略作为**非补丁缓解**常态化；对 `userdebug`/`eng`、Permissive、SPL 落后 3 个月以上三类情况设自动告警；对 WebView 与模块版本采不到的设备建立例外清单而不是跳过。

**事件响应与取证**：第 16 章覆盖矩阵作为检测建设依据；发生疑似事件时先采脚本 1 快照与 bugreport，再对齐投递内容与厂商固件说明；**不要试图从设备上证明基带状态**，那需要厂商证据。

**第三方评估**：范围声明里的边界同样约束你——本文提供的是清点、度量与验证方法，不提供可用性证明；把"发现缺陷"与"确认受影响"分开出报告。

---

## 二十一、FAQ

**Q1：Android RCE 最常见的入口到底是哪个？**
按公开材料的历史密度，内容解析器（C1）与浏览器内核（C2）贡献最多条目；近场无线（C3）与基带（C4）贡献最"贵"的前提（无交互、射程特殊）。这个问题没有单一答案，但**必须按通道回答而不是按 CVE 数回答**。

**Q2：我的设备 SPL 是本月，还需要担心 RCE 吗？**
需要区分：C1/C2 大概率已覆盖；厂商 HAL 取决于 vendor SPL；C4 与 SPL 无关。第 13 章指标 2/3/7 就是为此拆分。

**Q3：为什么不直接给一份"高危组件列表"？**
组件的高危性依赖机型与厂商实现，同一组件在不同 ROM 上落点与执法状态不同；固定列表会给出虚假可比性。本文提供的是采集方法（脚本）与判定框架（通道 × 落点 × 到达性）。

**Q4：`/apex` 里没看到 `com.android.media`，是不是说明没更新？**
不是。未观察到挂载项可能是模块被合并、改名、禁用或采集权限受限。脚本因此写"不得据此判定该模块已更新或未更新"，把它记为数据缺口。

**Q5：能不能从设备确认某条基带 CVE 已修？**
不能。可采到的只是版本字符串；修复归属需要厂商的机型—固件映射。没有映射就按第 17 章的例外处理，并在风险声明里保留。

**Q6：零点击 Android 漏洞是不是很多？**
公开口径里"零点击"不是统计维度，公告用的是"是否需要用户交互/额外执行权限"。GTIG 统计的是在野利用零日数量（2024 年 75、2025 年约 90，移动相关 17/9/15），不直接给出交互维度分布。

**Q7：Rust 化之后是否还需要担心这一类风险？**
语言消除的是内存安全类缺陷，不消除第 6.2 节的逻辑与信任假设缺陷，也不改变"厂商侧实现未更新"的到达性问题。

**Q8：Permissive 模式在什么情况下可以接受？**
只在隔离的实验设备上、且不接入生产数据与生产网络时。生产池里的 Permissive/userdebug 组合按第 13 章指标 4 作 P1 事件处理。

**Q9：这两个脚本会不会影响设备状态？**
不会。全部为 `getprop`/`settings get`/`getenforce`/`toybox ls`/`dumpsys package` 只读调用，不写入、不安装、不下发策略、不做触发式探测。

**Q10：我该先读这个系列的哪一篇？**
按架构 → 内核结构 → Binder → 本文（远程输入）→ Framework（越权）→ 内核（提权）→ 硬件/密钥栈 → 方法论 → 补丁链的顺序。本文位置是"输入如何抵达"，之后的一切都在别处。

---

## 二十二、参考资料

**口径与公告**

1. Security updates and resources | Android Open Source Project — https://source.android.com/docs/security/overview/updates-resources （严重度判据；公告与 SPL 机制）
2. Android Security and Update Bulletins — https://source.android.com/docs/security/bulletin （公告索引与类型/组件分区组织）
3. Android Security Bulletin—March 2026 — https://source.android.com/docs/security/bulletin/2026/2026-03-01 （RCE 后果表述范式；对 CVE-2026-21385 的 "There are indications that … may be under limited, targeted exploitation" 注记）
4. Android 13 Security Release Notes — https://source.android.com/docs/security/bulletin/android-13 （按组件分区的示例：Media Framework / System / Platform 等）
5. Qualcomm March 2026 Security Bulletin — https://docs.qualcomm.com/securitybulletin/march-2026-bulletin.html （CVE-2026-21385 标题为 "Integer Overflow or Wraparound in Graphics"；芯片公告按技术域组织）
6. Qualcomm Description of Security Ratings — https://www.qualcomm.com/company/product-security/description-security-ratings （芯片侧严重度口径与 Android/AOSP 口径不同）

**机制与加固**

7. Media framework hardening | Android Open Source Project — https://source.android.com/docs/core/media/framework-hardening （"Splitting AV pipeline components into app-specific sandboxed processes"）
8. Data Driven Security Hardening in Android — https://security.googleblog.com/2021/01/data-driven-security-hardening-in.html （"Starting in 2015 with Stagefright, a large number of Critical severity vulnerabilities were reported in Android's media framework"）
9. Security enhancements | Android Open Source Project — https://source.android.com/docs/security/enhancements （按版本列出机制，含 HWASan 引入标注）
10. ShadowCallStack | Android Open Source Project — https://source.android.com/docs/security/test/shadow-call-stack （定义与保护目标）
11. Tagged pointers | Android Open Source Project — https://source.android.com/docs/security/test/tagged-pointers （HWASan/MTE 相关）
12. Arm Memory Tagging Extension | Android Open Source Project — https://source.android.com/docs/security/test/memory-safety/arm-mte （标签分配与运行时检查）
13. Memory safety | Android Open Source Project — https://source.android.com/docs/security/test/memory-safety （非内存安全原生代码"represent over 70% of the Android platform code"）
14. Memory Safe Languages in Android 13 — https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html （内存安全类缺陷占比 2019 年 76% → 2022 年 35%）
15. Rust in Android: move fast and fix things — https://blog.google/security/rust-in-android-move-fast-fix-things/ （相对 C/C++ "1000x reduction" in memory safety vulnerability density）
16. Protecting against code reuse in the Linux kernel — https://security.googleblog.com/2019/10/protecting-against-code-reuse-in-linux_30.html （内核侧 CFI 与 Shadow Call Stack 的公开记录）
17. Mainline | Android Open Source Project — https://source.android.com/docs/core/ota/modular-system （模块可"由 Google 通过 Google Play 系统更新推送，也可由"厂商推送）
18. Hardening cellular basebands in Android — https://security.googleblog.com/2023/12/hardening-cellular-basebands-in-android.html （基带优先加固理由：特权运行 + 解析不可信输入）

**缺陷与研究**

19. Stagefrightened? | Project Zero — https://projectzero.google/2015/09/stagefrightened.html （CVE-2015-3864 为"不完美补丁"的再利用分析）
20. Multiple Internet-to-baseband Remote Code Execution Vulnerabilities in Exynos Modems | Project Zero — https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html （CVE-2023-24033、26496、26497、26498；Pixel 三月公告将四条全部列出）
21. Multiple Bluetooth implementation vulnerabilities affect many platforms（BlueBorne）— https://www.kb.cert.org/vuls/id/240311 （"All versions of Android prior to September 9, 2017 Security Patch level contain a vulnerable implementation of SDP within the Android Bluetooth software stack"）
22. BlueBorne Bluetooth Vulnerabilities | CISA/US-CERT — https://www.cisa.gov/news-events/alerts/2017/09/12/blueborne-bluetooth-vulnerabilities （远程攻击者可夺取设备控制的通告口径）
23. FragAttacks: Security flaws in all Wi-Fi devices — https://www.fragattacks.com/ （三个 Wi-Fi 设计缺陷与广泛存在的实现缺陷）
24. FragAttacks（Black Hat 2021 材料）— https://papers.mathyvanhoef.com/blackhat2021-fragattacks.pdf
25. Unpatched "QuadRooter" Vulnerabilities Put Over 900 Million Android Devices At Risk — https://www.securityweek.com/unpatched-quadrooter-vulnerabilities-put-over-900-million-android-devices-risk/ （公开口径为本地提权获取 root，用于纠正误区 1）
26. The truth behind QuadRooter — https://blog.avast.com/the-truth-behind-quadrooter （"an attacker can trigger privilege escalations for the purpose of gaining root access to a device"）
27. Stagefright（bug）概览 — https://en.wikipedia.org/wiki/Stagefright_(bug) （受影响版本区间的二手汇总）

**在野利用统计**

28. GTIG：Hello, My Old Friend — 2024 Zero-Day Exploitation Analysis（PDF）— https://services.google.com/fh/files/misc/2024-zero-day-exploitation-analysis-en.pdf （2024 年 75 起在野零日）
29. Look What You Made Us Patch: 2025 Zero-Days in Review — https://cloud.google.com/blog/topics/threat-intelligence/2025-zero-day-review （移动相关零日 2023 年 17、2024 年 9、2025 年 15）
30. Vulnerability Discovery and Exploitation Trends in the AI Era — https://cloud.google.com/blog/topics/threat-intelligence/vulnerability-discovery-and-exploitation-trends-in-the-ai-era （月均在野利用从 8 起升至 11 起）
31. Intellexa's Prolific Zero-Day Exploits Continue — https://cloud.google.com/blog/topics/threat-intelligence/intellexa-zero-day-exploits-continue （含 Android Runtime UAF 类内存损坏条目）
32. Protecting Android users from 0-Day attacks（TAG）— https://blog.google/threat-analysis-group/protecting-android-users-from-0-day-attacks/ （"routinely hunts for 0-day vulnerabilities exploited in-the-wild"）

**厂商与第三方实现（二手来源，仅作交叉印证）**

33. Google adds new Pixel security features to block 2G exploits and baseband attacks — https://www.scyscan.com/news/google-adds-new-pixel-security-features-to-block-2g-exploits-and-baseband-attacks/ （Android 14 起管理员可关闭 2G 的报道口径；以厂商策略页与实际机型为准）
34. Android's March 2026 security patch fixes over 100 flaws — https://www.helpnetsecurity.com/2026-03-03/android-march-2026-security-patch-cve-2026-21385/ （当期公告规模与 CVE-2026-21385 位置）
35. Android Security Bulletin—December 2025 — https://source.android.com/docs/security/bulletin/2025-12-01 （严重度评估前提的表述范式）

> 检索与撰写过程中确认**未公开文档化**、因此本文未采纳为事实的说法：WebView 作为 APEX/Mainline 模块的官方归类（本文改以包版本只读采集，采不到即单列）；"厂商可单独更新基带固件"的 Android 侧统一口径；机型 → 芯片缺陷的公开映射（高通/联发科公告按技术域组织，不列机型）；企业策略中固定或锁定 WebView 版本的标准能力项；各 EMM 上 2G 策略字段与实际生效范围；从设备侧读取 APEX 模块精确版本的无 root 通用方法；当期 Chromium 与特定 Android 构建的对应关系；QuadRooter 的"远程"定性（公开口径为本地提权）。

---

*本报告基于截至 2026-10-26 的公开资料整理。公告口径、机型实现、模块化范围与厂商策略能力都会变化；任何结论在用于处置决策前都应在目标机型与目标固件上重新采集。本文两个脚本只做只读采集与正向断言，不含触发式探测、利用性内容或绕过步骤；文中关于缺陷类别的描述停留在机制与后果层面，不提供构造方法。*

---

咨询ios系统请咨询 telegram：https://t.me/one00190
