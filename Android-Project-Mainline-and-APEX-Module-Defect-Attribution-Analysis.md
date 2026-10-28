---
layout: default
title: "Android Project Mainline 与 APEX 模块缺陷归属分析报告"
description: "Project Mainline、APEX 容器与“模块”三个词的分工，公告 component 词表与 com.android.* 模块词表之间的对应落差，模块归属的六个面（归属单位、存在形态、更新通道、生命周期状态、契约表面、设备侧可证明性），Available modules 清单与 GMS/AOSP 前缀差异，APEX 四文件与 apex_manifest 版本语义，内置版本与已激活版本的分歧、/apex/apex-info-list.xml 的字段与属性类型的 SELinux 依据，模块契约（稳定 AIDL、@SystemApi、@hide 符号消失）与厂商禁用通道，13 组公告双表 CVE 读数与 Subcomponent 词表，M1–M10 缺陷归属类型学、归属矩阵、四级判定模型与十项运营指标、两个只读核验脚本（模块姿态快照与两次快照的漂移比对）"
lang: zh-CN
permalink: /Android-Project-Mainline-and-APEX-Module-Defect-Attribution-Analysis/
---
# Android Project Mainline 与 APEX 模块缺陷归属分析报告

**Android Project Mainline and APEX Module Defect-Attribution Analysis: Module Boundaries, the Bulletin-to-Module Vocabulary Gap, Lifecycle State, and Read-Only Verification**

- **报告日期**：2026-10-28
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动平台安全负责人、OEM/ODM 的系统与集成工程师、漏洞治理与补丁管理负责人、设备风控与资产清点团队、应用与 SDK 兼容性维护者、需要把公告条目派工到具体代码所有者的 SRC/PSIRT 成员、做 Mainline 模块回归与灰度的发布工程人员
- 视角声明：本文写的是**"一个缺陷怎样被归属到模块边界上，以及这条归属在设备侧能被证明到哪一步"**——公告的 component 词表与 `com.android.*` 模块词表之间的对应关系与落差、模块的存在形态与生命周期状态、模块与框架之间的责任切分、以及这些属性里哪些能被一次只读采集拿到。**APEX 的构建装配与签名口径、更新通道的治理指标、各子系统内部的缺陷类型学一律不在本文**，只做交叉引用而不重复展开：
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（`apex` / `prebuilt_apex` / `apex_key` 怎样装配、分区怎样落地、构建身份与 test-keys；本文第十一章只借用它的装配结论来问"归属为什么不能只看构建"）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（SPL 两档语义、S1–S5 链条与到达性实测；本文第十二章把 SPL 当作**不能解释模块状态的坐标**来用，第五章的通道治理指标不在本文重复）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层、进程与四条更新通道的整体坐标；本文是"模块"这一层的竖切放大图）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（公告 component 分档与 `-01`/`-05` 归因规则；本文第十六章沿用它的口径，但把问题从"哪一档"换成"哪一个模块"）
  - [Android 远程代码执行攻击面与防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（修复的第二条到达路径，以及"WebView 未被文档化为 APEX"这一拒绝归类；本文第十六章把它当对照组，说明**能点名模块**与**不能点名模块**的公告差别）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（`com.android.wifi` 可更新、Bluetooth 为**可选**模块、以及 mounted/absent/unknown 三态纪律；本文第十九章的 Wi-Fi 读数与该纪律同源）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面与 R0–R4 分级；本文的归属矩阵回答"这一格里站着哪个所有者"，与它的"这一格里站着谁能到达"互补）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（**反例**：缓解项为什么没有模块载体；本文第十八章正面使用它给出的"必须走伙伴 OTA"这条口径）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（UID/SELinux 域/seccomp 四层；本文第十三章讲模块自己的契约表面时，用它的身份模型解释"谁能读哪个读数面"）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（AVB 描述符与验证启动链；本文第七章的 APEX 内嵌 hashtree 描述符按该篇的 AVB 口径读取，`com.android.rkpd` 只作为模块实例出现）
  - [Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（GKI/VNDK 与内核侧装配；本文说明**公告里 Kernel 表与 Mainline 表互不相干**这一事实，内核缺陷不在本文）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（`apex_service` 的 binder 服务与 SELinux 判定；本文第十三章的"shell 拿不到 apex_service"按该篇的域标识口径解释）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（器件形态与厂商侧可见性；本文第十九章的 widevine/TPU/Cellular Modem 读数说明**器件缺陷为什么落不进模块表**）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（应用侧声明的弱点口径；本文第十五章的 `@hide` 符号消失问题与它的"目标版本改变默认值"同属**同一份代码在不同安装形态下行为不同**这一类）
- 范围声明：全文只讨论**公开文档化**的模块机制、公告口径、源码里可指认的字段与属性，以及在这些公开规则之上可以安全实施的**只读**清点与比对流程。**不提供任何可用的漏洞利用代码、提权或绕过步骤、模块替换或降级安装的操作方法、签名材料的制作流程；不给出任何针对具体机型或固件版本的攻防操作。**本文两个脚本，一个只做设备侧只读采集（`get-state` / `getprop` / `ls` / `cat` / `stat` / `dumpsys` / `cmd` 七类命令，全部指向公开路径与公开服务，且脚本自身对预期失败不隐藏错误），一个只做离线文本比对；两者都不写设备、不安装或卸载任何东西、不挂载或卸载文件系统、不访问网络、不触发任何安装会话。所有"核验"指读者对**自有或已明确获授权的测试设备**执行采集。

---

## 一、摘要：一个缺陷要经过四次换名字，才能落到"某个模块的某个版本"

在 Android 10 之后，"这个 CVE 修了吗"这个问题经常问不出答案，不是因为没人回答，而是因为**一句话里混着四件不同的事**：这段代码被归到哪个组件名下、它在设备上以哪种形态存在（内置 APEX、可更新 APEX、APK 模块，还是没有模块载体）、修复通过哪条通道到达这台设备、以及到达之后**它到底激活了没有**。这四件事的责任人不同、坐标轴不同、可证明性也不同：公告给的是组件名和日期，构建给的是分区和签名主体，更新通道给的是应用市场或 OTA 包，设备给的是一个挂载点和一份 XML。把它们塞进同一张"是否已修"的表，是这一层风险治理里最常见的失效模式——表上每行都有格子，实际上有的格子问代码归属、有的问分发路径、有的问本机状态。

官方文档把这件事的第一环写得很直白：**组件（component）是归属的唯一键，而且它决定三件事**。

> "The first task in handling a security vulnerability is to identify the severity of the bug and which component of Android is affected. The severity determines how the issue is prioritized, and the component determines who fixes the bug, who is notified, and how the fix gets deployed to users." — [Updates and resources](https://source.android.com/docs/security/overview/updates-resources)

也就是说，component 这个词同时决定了**谁修、通知谁、怎么分发**。而公告页面自己的分组声明只有一句：

> "Vulnerabilities are grouped under the component they affect." — [Android Security Bulletin 2024-03-01](https://source.android.com/docs/security/bulletin/2024-03-01)

问题从这里开始：**公告的 component 词表和 Project Mainline 的模块词表是两套词表，而且公告从不写模块名。**本文在核对 2024-06-01、2025-02-01、2025-09-01 三份公告页面时，检索 `com.android.*` 形式的包名，命中 **0 次**；公告里唯一专门写给 Mainline 的那张表只有两列——

> "Google Play system updates / The following issues are included in Project Mainline components. / Subcomponent | CVE" — [Android Security Bulletin 2024-06-01](https://source.android.com/docs/security/bulletin/2024-06-01)

这张表**没有类型、没有严重度、没有受影响版本、没有 AOSP 版本列**。换句话说，公告在"哪些修复属于 Mainline"这一格上，给出的是全系列中最弱的一条记录；而同一个 CVE 往往同时在 `Framework` 或 `System` 表里带完整字段出现——本文第十九章用 13 组这样的双表读数把这件事量化。

于是"缺陷归属到模块"不是一次查表，而是**四次换名字**：

1. 缺陷 → 公告里的 `Component`/`Subcomponent` 词（人写的字符串，如 `Conscrypt`、`WiFi`、`Documents UI`）；
2. 词 → `com.android.*` / `com.google.android.*` 模块名（要自己映射，且映射规则不在公告里）；
3. 模块名 → 设备上是否存在、以何种形态存在（APEX 还是 APK、内置还是已更新、激活还是仅安装）；
4. 已激活 → 具体版本号，以及这个版本是否真的包含那一行的修复。

**每一步都可能失败，而且失败的模式各不相同**——这是本文全部 30 章要展开的东西。管理上的直接后果是：一台设备的 Mainline 侧风险状态，**不能由 SPL 日期单独推出**，也不能由 `/apex` 目录里有没有某个文件单独推出，它需要一条完整归属链，而这条链上可被设备侧只读证明的部分是有限的、且哪些有限是写在 SELinux 策略里的（第十三章）。

为此本文做四件事：

1. 把"模块"这个词拆成六个归属面（**归属单位、存在形态、更新通道、生命周期状态、契约表面、可证明性**），逐面给出公开定义句、源码字段或策略行（第三、四章）；
2. 建立**公告词表与模块词表的差集矩阵**，并用真实公告行验证映射的四类失败模式（第六、十六、十九、二十章）；
3. 把模块的**生命周期状态**读成一等公民对象：内置版本与已激活版本的分歧、"已安装≠已生效"这条来自官方文档的时序、以及 staged 会话与原子回滚的语义（第八、九、十章）；
4. 给出两个只读核验脚本：一副**模块姿态快照**（覆盖设备侧可读的全部归属信号）和一次**两次快照的漂移比对**（把"消失≠已修"变成可执行的判定），每个码位都有等级标签与样例输出（第二十二、二十三章）。

---

## 二、三个词的分工：Mainline、APEX、模块，以及四种混用

这一层的讨论之所以困难，一半原因是三个词被长期混用，而它们在公开文档里各自有明确的、互不重叠的定义。

### 二之一 三条官方定义句

**Mainline 是一个把组件改造成可独立更新单元的项目**，它的定义句里包含"脱离版本周期"这个关键承诺：

> "Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle." — [Mainline](https://source.android.com/docs/core/ota/modular-system)
>
> "Mainline converts selected system components into modules." — 同页

**APEX 是一种容器格式，不是更新机制**，而且文档明确说了不是所有模块都用它：

> "Some Mainline modules use the APEX container format (introduced in Android 10) and some use the APK format." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

**模块是归属单位**，其存在形态至少有三种：可更新 APEX、APK 形态模块、以及**不可更新的 APEX**。最后这一种是本文反复要用的关键区分——同一格式承载两种完全不同的可更新性：

> "In Android 11, ART and libcore are packaged as non-updateable APEX." — [ART module](https://source.android.com/docs/core/ota/modular-system/art)

把这三条放在一起，就得到本文最重要的一个结构性结论：**"这台设备上有 APEX"不等于"这些代码可独立更新"，更不等于"这一行的修复已经到达"。**

### 二之二 四种常见混用，以及它们各自会让判断错在哪

1. **把 `/apex` 目录当模块清单。** 挂载点上是**当前激活**的版本，而且内置版本与更新版本会共享同一目录名（"the one that corresponds to the latest version is bind-mounted at `/apex/<apex_name>`"，[APEX file format](https://source.android.com/docs/core/ota/apex)）。用目录判断"有没有这个模块"，会把"存在但从未被更新"和"不存在"读成同一件事。本文第九章给出挂载点的完整语义（含 `@<version>` 实例目录）。
2. **把 APEX 当 Mainline。** APK 形态的模块（如清单里的 `com.android.documentsui`、`com.android.permissioncontroller`）没有 APEX 文件；它们的安装、验证与回滚路径完全不同，公告侧的可见性也不同。第十六章用一条 Wi-Fi 模块文档句说明契约差异，第十章用原子回滚句说明事务差异。
3. **把公告的 `Subcomponent` 当包名。** `WiFi`、`Conscrypt`、`Documents UI` 这些字符串是**人为书写**的分类词，与 `com.android.wifi`、`com.android.conscrypt`、`com.android.documentsui` 之间的对应关系不写在公告里，也不是机械的小写化（`Documents UI` 中间有空格、`Cell Broadcast` 对应 `com.android.cellbroadcast`、`MediaProvider` 对应 `com.android.mediaprovider`）。第十九章给出映射表的构造规则与它的四类失败。
4. **把 `com.google.android.*` 与 `com.android.*` 当成两个模块。** 它们是同一模块在**不同签名主体设备**上的两个名字：

   > "On Google Mobile Services devices, Mainline module packages are signed by Google and their names use the `com.google.android.*` preface. For devices signed with AOSP keys, package names are prefaced with `com.android.*`." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

   这条直接决定清点脚本的匹配规则：**按后缀名匹配，不能按完整包名匹配**，否则同一份资产在 GMS 设备和 AOSP 签名设备上会得到两套互不包含的清单。第廿二章脚本的 `apex_name_prefix` 类码位就是为这件事准备的。

### 二之三 术语约定

| 术语 | 本文含义 | 公开依据 |
| --- | --- | --- |
| 模块（module） | Mainline 转换出来的归属单元，公告侧用 `Subcomponent` 词指称 | Mainline 页定义句 |
| APEX 文件 | 未压缩 zip 容器，内含四个必备文件 | APEX 页 "The four files in an APEX file are: …" |
| 内置版本 | 出厂分区里的 APEX 文件；无更新时的兜底 | "The APEX manager defaults to using these packages if no updates are available." |
| 激活（active） | 已挂载并对其他进程可用；**安装成功不等于激活** | "the newly installed version of the APEX is only activated after reboot." |
| 扁平化 APEX | 解开为普通目录形态的构建方式，Android 15 起废弃，且**只能走 OTA** | "Flattened APEXes can be updated only via a regular OTA." |
| 契约表面 | 模块被允许依赖的接口集合：SDK/@SystemApi、稳定 C API、稳定 AIDL | "Module updates don't introduce new APIs. …" |
| 可选模块 | 厂商可以选择不提供的 Mainline 模块，NFC Services 文档自称即此 | "The NFC Services module is an optional updatable Mainline module released in Android 16." |
| staged 会话 | 位于 `/metadata/apex/sessions` 的批量安装会话，决定"下一轮激活什么" | apexd.cpp:2221-2222 |

---

## 三、模块归属的六个面（M1–M6）

一次完整的归属至少要说清六件事。本文把它们编号为 M1–M6，后面每一章深挖一个面，最后的归属矩阵（第廿章）把六面拼成可派工的格子。

| 面 | 问的是什么 | 谁来回答 | 设备侧能否只读证明 |
| --- | --- | --- | --- |
| **M1 归属单位** | 这段代码属于哪个模块 | 公告 `Subcomponent` + 模块清单 | 部分：只能读到模块名与版本，读不到"CVE ↔ 版本" |
| **M2 存在形态** | APEX / APK / 扁平化 / 根本没有模块载体 | 构建装配 + `ro.apex.updatable` | 可读到形态线索，但**构建侧口径不在本文**（引向平台代码篇） |
| **M3 更新通道** | Play 系统更新 / 伙伴 OTA / 随整机镜像 | Mainline 定义句 | 可读到"通道是否被厂商关掉"的痕迹（`ro.boot.vendor.apex.*`，第十三章） |
| **M4 生命周期状态** | 已安装 / 已激活 / 被停用 / 会话未完成 | apexd 与 staged 会话 | 部分可读，且**权限差异被 SELinux 硬性规定**（第八章） |
| **M5 契约表面** | 模块能依赖哪些接口、装上来时应用看到什么 | 模块文档与 `@hide` 规则 | 弱：只能读版本与激活标志，读不到契约违反 |
| **M6 可证明性** | 上述五面里哪些能被一次采集固定下来 | SELinux 策略 + 属性类型 | 这就是本文两个脚本的分工边界 |

三句话概括这六个面的关系：

- **M1–M2 是"归属"，M3–M5 是"分发与生效"，M6 是"你敢不敢把结论写进报告"**。三组各有失败模式，把它们混成一个"已修/未修"布尔值时，最常被牺牲的是 M6。
- M4 是被低估最严重的一个面。官方文档给出了明确的时序——安装与激活之间隔着一次重启（"the newly installed version shadows the version already present in the built-in partition. But unlike APKs, the newly installed version of the APEX is only activated after reboot."），而多数清点流程只读"有没有新文件"，于是把 M4 的两态压成一态。
- M3 与 M6 的组合是运维最需要的一条：**通道可以被厂商侧关闭，而关闭这件事在设备上是能读到的**（`ro.boot.vendor.apex.` 与 `persist.vendor.apex.` 前缀，属性类型为 `apexd_select_prop`，`system_public_prop` 即"无读取限制"）。第十三章给策略行，第廿二章脚本把它变成码位。

---

## 四、为什么"模块边界"仍然值得当作归属单位

如果归属链要四次换名字、每一步都可能失败，为什么不干脆回到"分区 + SPL"这套老坐标？答案是：**模块边界是公开文档里唯一一个同时携带"谁修、怎么到、能不能被厂商关掉"三类信息的单位**，而分区只携带第一类。三条来自同一页的定义句支撑这个判断：

> "enables system components to be updated with critical bug fixes and other improvements as needed, without affecting lower-level vendor implementations or higher-level apps and services." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

这句给出**解耦承诺**：模块层的修复不触碰 vendor 实现，也不触碰上层应用。它同时划出了本文与厂商硬件篇、应用弱点篇的边界——同一缺陷如果落点在 vendor 侧或应用侧，就不在模块口径内。

> "The module package installs and rolls back atomically; either all modules that need to be updated are updated or none are updated." — 同页

这句给出**事务单位**：模块批量更新是原子的。它对失败分析的意义很大——一次失败留下的是"全批未更新"，不是"部分模块新部分模块旧"，因此**版本序列上的整齐跳变本身就是批次指纹**（第廿一章用这个性质读漂移样例，第廿三章脚本的 `pair_*` 码位据此设计）。

> "Note: Mainline support for Android 11 and lower is concluded as of Q4 of 2025." — 同页

这句给出**时间边界**：模块通道不是永恒属性，它按 Android 大版本有起止。清点一台旧设备时，"没有模块更新"可能不是治理失败而是通道终止——这是第十二个误区要处理的内容，也是本文要求快照里同时保存 `release` 版本行的原因。

**但模块边界作为归属单位有一个不可回避的代价：它把"归属"从查表变成了推理。**公告侧不提供模块名（第一章实测 0 命中），也不提供模块版本；`Subcomponent` 词表是手写的、跨月会变的（2022-09 那批表头用 `Component | CVE`，2023-05 起改为 `Subcomponent | CVE`，同一含义换了列名）。因此本文的立场不是"模块边界更好所以用它"，而是：**要用它，就必须同时把它的证明成本写进流程**——这正是第四篇（本文）与相邻的补丁治理篇的分工：那篇度量"固件能不能推到设备上"，这篇度量"推到的东西归属到哪、生效了没有、以及我们凭什么说它生效了"。

---
## 五、模块清单的公开口径：Available modules 表能回答什么，不能回答什么

Mainline 页面给出一张 **Available modules** 表，这是公开渠道里唯一成体系的模块名单。本文在核对时把表内出现的包名逐条抄下（名称为原文，不做归一化）：

`com.android.adbd`、`com.google.android.adservices`、`com.google.android.healthfitness`、`com.android.appsearch`、`com.android.art`、`com.google.android.bt`、`com.android.cellbroadcast`、`com.android.configinfrastructure`、`com.android.conscrypt`、`com.android.scheduling`、`com.android.resolv`、`com.android.documentsui`、`com.android.ext.services`、`com.android.ipsec`、`com.android.media`、`com.android.media.swcodec`、`com.android.mediaprovider`、`com.android.modulemetadata`、`com.android.networkstack`、`com.android.neuralnetworks`、`com.google.android.ondevicepersonalization`、`com.google.android.federatedcompute`、`com.android.permissioncontroller`、`com.android.rkpd`、`com.android.sdkext`、`com.android.os.statsd`、`com.android.tethering`、`com.android.tzdata`、`com.android.uwb`、`com.android.wifi`

这张表带有 Type（APEX / APK）与引入版本两类信息，模块子页则逐条给出形态与设备类别，本文只引用**核对到原文**的那几条：

> "The Wi-Fi module (com.android.wifi) is in APEX format and is available for devices running Android 11 or higher." — [Wi-Fi module](https://source.android.com/docs/core/ota/modular-system/wifi)
>
> "The Media module (com.android.media) and Media Codec module (com.android.media.swcodec) are in APEX format and are available for devices running Android 10 or higher." — [Media module](https://source.android.com/docs/core/ota/modular-system/media)
>
> "The NFC Services module is an optional updatable Mainline module released in Android 16." — [NFC Services module](https://source.android.com/docs/core/ota/modular-system/nfc-services)

### 五之一 清单回答的三件事

1. **谁被转换成了模块**——这是 M1 归属单位的候选集合；
2. **它以什么容器形态交付**（APEX 还是 APK）——决定安装与回滚路径，因此决定第 4 章 M2 的读法；
3. **它从哪个 Android 版本开始存在**——决定第 4 章的时间边界（含 Android 11 及以下支持终止那条注记）。

### 五之二 清单不回答的四件事

1. **不回答版本。** 表里没有版本列，也没有"当前发布的模块版本"这件事的任何指针。要拿到版本必须换采集面（第八、九章），而且采集面的可读性受 SELinux 约束（第十三章）。
2. **不回答"这台设备上有没有"。** 清单是**能力清单**，不是**资产清单**。同一份清单在 GMS 设备、AOSP 签名设备、以及厂商关闭了某个模块的设备上，对应的实际集合互不相同——`optional` 一词（NFC Services 的自述）就是这条差异的文档证据。
3. **不回答"是否可独立更新"。** Type=APEX 只说明容器形态；ART 模块页明确出现过 "packaged as non-updateable APEX" 的历史形态，而扁平化 APEX 更是"有 APEX 内容但只能走 OTA"。**形态与可更新性是两个正交属性**，把它们读成一件事是第五个误区。
4. **不回答公告对应关系。** 公告用 `Subcomponent` 词，清单用包名，两套词表之间没有任何公开映射表（第六章给出映射规则的构造方式与其失败模式）。

### 五之三 "缺席"的三种情形必须分开记

清点一台设备时，"某个模块不在"至少有三种成因，而它们在快照里长得一模一样：

| 缺席类型 | 成因 | 公开依据 | 采集侧判别方式 |
| --- | --- | --- | --- |
| **能力缺席** | 该模块不适用于这台设备的 Android 版本，或 Android 11 及以下通道已终止 | "Mainline support for Android 11 and lower is concluded as of Q4 of 2025." | 与 release 版本行一起读（快照里 `release` 与 `apex_updatable` 码位同源） |
| **厂商缺席** | 可选模块未被提供，或被厂商侧属性选择关掉 | "an optional updatable Mainline module"；`ro.boot.vendor.apex.` / `persist.vendor.apex.` 前缀存在 | 读属性前缀（第十三章），不能只读挂载点 |
| **读数缺席** | 模块在设备上，但当前采集面读不到它的名字或版本 | shell 域对 apex 服务的策略限制：`allow shell { service_manager_type -apex_service … }` | 用第二个采集面交叉验证；两面都缺时记为**未采到**而不是"不存在" |

第三行是本文全流程的纪律来源：**清单比对得出的"缺失"必须标为未知，而不是风险**。第廿二章脚本对这一点给了独立码位（`apex_list_unread` 一类），第廿三章的漂移比对也不把"上一次有、这一次没有"直接判成风险等级——理由与第十三章的策略行有关：读不到可以仅仅因为换了一个采集权限面。

---

## 六、名称与签名主体：一台设备上的两个名字，与公告词的映射规则

### 六之一 前缀由签名主体决定，不由模块决定

> "On Google Mobile Services devices, Mainline module packages are signed by Google and their names use the `com.google.android.*` preface. For devices signed with AOSP keys, package names are prefaced with `com.android.*`." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

这条句子的操作性后果比它的篇幅大得多：

1. **匹配规则必须以"后缀"为键。** 任何按完整包名硬编码的清单（例如 `com.android.wifi`）在 GMS 设备上会漏掉 `com.google.android.wifi`（同一模块的另一身份）。本文脚本因此在两处使用后缀匹配：模块名比对与公告词映射。
2. **签名主体决定了谁能推送更新。** 名字前缀与签名者是同一条句子里并列给出的——前缀是"由 Google 签名"这一事实的外显标记。因此**"模块存在"与"这台设备能从官方通道获得它的更新"是两件事**（M3），后者还与厂商是否关闭通道叠加（第十三章）。
3. **清单本身混用两种前缀。** 上表的原文里 `com.google.android.adservices`、`com.google.android.healthfitness`、`com.google.android.bt`、`com.google.android.ondevicepersonalization`、`com.google.android.federatedcompute` 直接以 Google 前缀列出，而其余以 `com.android.` 列出。这说明清单是以"Google 设备视角"书写的名单，**它不是 AOSP 签名设备的清单**。用这张表直接给 AOSP 签名设备做资产核对，会在第一批条目上就产生系统性偏差。

### 六之二 公告的 `Subcomponent` 词表与模块名的映射怎么建

公告侧的书写单位是人造的短语。以核对到的 13 组双表读数（第十九章全表）为样本，可以归纳出**四类映射关系，置信度递减**：

| 类型 | 样本（公告词 → 模块名） | 规则可靠度 | 本文采用的标注 |
| --- | --- | --- | --- |
| **A 后缀直读** | `Conscrypt` → `com.android.conscrypt`；`ART` → `com.android.art`；`AppSearch` → `com.android.appsearch`；`MediaProvider` → `com.android.mediaprovider` | 高：小写去空格即命中清单条目 | `MAP_DIRECT` |
| **B 空格/大小写变形** | `WiFi` → `com.android.wifi`；`Cell Broadcast` → `com.android.cellbroadcast`；`Permission Controller` → `com.android.permissioncontroller`；`Documents UI` → `com.android.documentsui` | 高，但**词形不机械**（`WiFi` 而非 `Wi-Fi`，`Documents UI` 含空格） | `MAP_NORMALIZED` |
| **C 语义换名** | `DNS Resolver` → `com.android.resolv` | 中：需要人工词典；公告词与包名后缀不同源 | `MAP_LEXICON`，并在正文标注该词典是本文构造 |
| **D 一词多候选** | `Media Codecs` → `com.android.media.swcodec` 还是 `com.android.media`？ | **低：公告文本未给包名，两个候选都在清单内** | `MAP_AMBIGUOUS`，禁止在报告里当作已确认归属 |

四类合起来就是"四次换名字"里第二步的真实难度：**公告从不点名 APEX 包名**——本文在 2024-06-01、2025-02-01、2025-09-01 三页检索 `com.android.*`，命中 0 次。因此任何"这个 CVE 属于 `com.android.media.swcodec`"的句子，要么来自厂商自己的公告，要么来自本文这样的推断，**必须显式标注为推断**（第廿一章的判定模型把 `MAP_AMBIGUOUS` 单独降一档）。

### 六之三 映射失败的四种模式（治理上真正要防的东西）

1. **无模块行的月份。** 公告会整月地声明"没有 Mainline 侧问题"：

   > "There are no security issues addressed in Google Play system updates (Project Mainline) this month." — [2024-03-01](https://source.android.com/docs/security/bulletin/2024-03-01)（同句复现于 2024-04-01、2024-08-01、2025-08-01）

   这条句子的正确读法是**"本月的这批修复不在 Mainline 组件里"**，而不是"本月没有修复"。把它读成后者会导致整月漏派工。
2. **同一 CVE 双表出现。** 例如 CVE-2024-43093 同时出现在 `Framework` 表（带 Type/Severity/版本）与 Mainline 表的 `Documents UI` 行——一次修复、两个归属口径。派工时只按其中一张表，就会重复或遗漏。
3. **词表跨月漂移。** 表头在 2022-09 写作 `Component | CVE`，2023-05 起写作 `Subcomponent | CVE`；章节名也有 `Runtime` → `Android Runtime` 的变化。任何把公告词硬编码成枚举值的工具，都会在边界月份静默失配。
4. **公告词覆盖不到的模块。** 本文核对的 17 个月份里，Mainline 表的 Subcomponent 列**未出现** Bluetooth、uwb、NFC、IPsec、Tethering、AdServices、NNAPI 等条目。这不能证明这些模块没有缺陷，只能证明**它们的修复没有以这张表的形式被公开登记**（这是第三章 M6 的一个具体体现：登记口径的边界决定了能说什么）。

---

## 七、APEX 容器的四件套：能证明完整性，证明不了包含哪个修复

### 七之一 容器的公开定义

> "At the top level, an APEX file is a zip file in which files are stored uncompressed and located at 4 KB boundaries." — [APEX file format](https://source.android.com/docs/core/ota/apex)
>
> "The four files in an APEX file are: `apex_manifest.json` `AndroidManifest.xml` `apex_payload.img` `apex_pubkey`" — 同页
>
> "`apex_payload.img` is an ext4 file system image backed by dm-verity. The image is mounted at runtime via a loopback device." — 同页
>
> "the `apex_payload.img` (specifically, the vbmeta descriptor appended to `apex_payload.img`) file is signed with a key. Then, the entire APEX is signed using the APK signature scheme v3." — 同页

四件套各自承载一个属性：manifest 承载**身份**（包名+版本，第八章），`apex_payload.img` 承载**内容与完整性**（dm-verity 哈希树），`apex_pubkey` 承载**签名主体**（对应第六章"前缀由签名者决定"），`AndroidManifest.xml` 承载**目标范围**（SDK 版本上下界）。

构建器一侧对文件系统类型的取值比文档句子更宽：`system/apex/apexer/apexer.py:112-113` 给出 `choices=['ext4', 'f2fs', 'erofs']`，注释为 `'type of filesystem being used for payload image'`。这是一处**文档口径与实现口径的不一致**（文档只写 ext4），对归属分析的实际影响是：不能从"payload 是 ext4"推出设备的构建配置。

### 七之二 哈希树描述符的真实字段名

`apex_payload.img` 后附的是 AVB hashtree 描述符，字段名以 `external/avb/libavb/avb_hashtree_descriptor.h:72-79` 为准：

```
uint64_t image_size;
uint64_t tree_offset;
uint64_t tree_size;
...
uint64_t fec_offset;
uint64_t fec_size;
```

apexd 侧读取它（`system/apex/apexd/apex_file.h:34` — `std::unique_ptr<AvbHashtreeDescriptor> desc;`），并使用 `GetImageOffset()` / `GetImageSize()`、`image_offset_` / `image_size_`（同文件 `:53-56,79-80`）命名映像定位。

> **本节排除的讹传**：`payload_offset` / `payload_size` 不是 APEX 或 AVB 的字段名（`avbtool` 中 0 命中）；AOSP 文档与 `apexer.py` 中**不存在 squashfs** 这一 payload 类型；`cgroups` 也不在 APEX 必备内容清单里。这三项在不少二手资料里被当作 APEX 结构描述，本文不采用。

### 七之三 压缩 APEX 的条目差异

同一页对压缩形态给出条目级说明：

> "`original_apex`: deflated with compression level of 9 … `apex_manifest.pb`: stored only … `AndroidManifest.xml`: stored only … `apex_pubkey`: stored only" — [APEX file format](https://source.android.com/docs/core/ota/apex)

**清单里出现了 `apex_manifest.pb`**——protobuf 形态的 manifest 在压缩容器里是顶层条目，而"四件套"句子只列了 `.json` 形态。这条差异直接对应第十章的运行时读取顺序（`apex_constants.h:76` — `static constexpr const char* kManifestFilenamePb = "apex_manifest.pb";`），也解释了为什么**用"四件套"字面清单去校验设备上的 APEX 文件会得到假阳性缺失告警**。

### 七之四 容器层能证明什么，不能证明什么

容器与签名机制回答的是"**这个包是不是它声称的那个包、由谁签发、内容有没有被改**"。这三问由 AVB 链承担，其机制在[硬件信任根与密钥栈篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)展开，本文不重复。

**对归属而言，容器层的天花板是明确的：完整性证据不能替代补丁证据。**一份签名良好、哈希自洽的 APEX 只说明"它是某个构建的原始产物"，不说明"它包含 CVE-X 的修复"——因为版本号与修复集合之间的关系不写在容器里（第八章说明版本比较规则为何连"同版本号"都不能区分内容）。这就是本文坚持"公告词 ↔ 模块名 ↔ 版本序列"三段都要独立取证的原因。

---

## 八、版本语义：`name` + `version` 是身份，但身份不足以承载补丁结论

### 八之一 身份定义句

> "`apex_manifest.json` file contains the package name and version, which identify an APEX file." — [APEX file format](https://source.android.com/docs/core/ota/apex)
>
> "the APEX can be updated by providing an APEX package with the same package name and a greater than or equal to version code." — 同页

第二句里的 **"greater than or equal"** 是本章的核心：按文档口径，**版本码相等也被描述为一次合法更新**。这条规则对治理的含义是：**版本号不能充当补丁指纹**——两台设备报告同一个 `versionCode`，在"包含哪些修复"这件事上仍然不等价，除非版本号的分配方（Google 或厂商）保证每次内容变化都推进版本。这个保证不在任何公开文档里，因此本文把"同版本号"标为**弱证据**（第廿一章判定模型的第二级）。

apexd 把 `version` 当必填项强制：`system/apex/apexd/apex_manifest.cpp:44-45` — `if (apex_manifest.version() == 0) {` 后跟 `"Missing required field \"version\"..."`。proto 侧的字段以 `system/apex/proto/apex_manifest.proto` 为准：`:28` `string name = 1;`、`:31` `int64 version = 2;`（注意是 int64，不是 int32）、`:41` `string versionName = 5;`、`:85` `bool supportsRebootlessUpdate = 13;`、`:98` `bool bootstrap = 16;`。

**`versionName` 是给人读的字符串，不参与比较**；比较只发生在 `version`（即 `versionCode`）上。任何把 `versionName` 当排序键的清点流程都是脆弱的。

### 八之二 目标范围字段与"装不上去"的另一类原因

> "The `AndroidManifest.xml` file inside an APEX file is minimal, consisting of the package `name`, `versionCode`, and optional `targetSdkVersion`, `minSdkVersion`, and `maxSdkVersion` for fine-grained targeting." — 同页

`minSdkVersion` / `maxSdkVersion` 给出**平台版本范围**：同一个模块版本，对某个平台版本可能根本不适用。它与第五章的"能力缺席"共同决定"设备上找不到该模块"是否正常，也解释了为什么跨机型比较模块版本集合时必须同时携带平台版本行（第廿二章脚本把 `release` 与 `sdk` 作为快照头两行强制记录）。

### 八之三 设备侧可读的身份字段

设备侧真正承载"这台机器上有什么模块、什么版本"的文件是 `/apex/apex-info-list.xml`（`apex_constants.h:78` — `kApexInfoList = "/apex/apex-info-list.xml"`）。它的 schema 是 `system/apex/apexd/ApexInfoList.xsd`，根元素与条目元素的名称在 `:18` 与 `:36`：

```
<xs:element name="apex-info-list">        (xsd:18)
  <xs:element ref="apex-info" minOccurs="1" maxOccurs="unbounded"/>   (xsd:21)
<xs:element name="apex-info">             (xsd:36)
```

`:38-47` 逐字声明十个属性，其中六个是 `use="required"`：

| 属性（xsd 行号） | 类型 / 约束 | 回答的问题 | 归属面 |
| --- | --- | --- | --- |
| `moduleName`（`:38`） | `xs:string`，required | 哪个模块 | M1 |
| `modulePath`（`:39`） | `xs:string`，required | 这个版本的文件在哪 | M2 |
| `preinstalledModulePath`（`:40`） | 可选 | 出厂内置版本在哪 | M2 |
| `versionCode`（`:41`） | `xs:long`，required | 哪个版本（弱，见八之一） | M1 |
| `versionName`（`:42`） | `xs:string`，required | 可读版本串 | M1 |
| `isFactory`（`:43`） | `xs:boolean`，required | 是否出厂内置 | M2 |
| `isActive`（`:44`） | `xs:boolean`，required | 是否已激活 | M4 |
| `lastUpdateMillis`（`:45`） | `xs:long`，可选 | 这个包上次变化时间 | M4 的时间侧 |
| `provideSharedApexLibs`（`:46`） | `xs:boolean`，required | 是否提供共享库 | M5 的旁证 |
| `partition`（`:47`） | 枚举，required | 落在哪个分区 | M2/M3 |

`:26-33` 给 `partition` 定义了**五个字面值**，不是数字：`SYSTEM`、`SYSTEM_EXT`、`PRODUCT`、`VENDOR`、`ODM`。这五个值与 `apex_constants.h:39-43` 的五个内置目录（`/system/apex`、`/system_ext/apex`、`/product/apex`、`/vendor/apex`、`/odm/apex`）一一对应，`apex_constants.h:46-50` 还给出了 `ApexPartition` 到目录的映射表。写解析脚本时要注意：**枚举越界值本身就是读数面异常**，第廿二章脚本为这一格单独留了码位。

两个可选属性的存在与否也是信息：`preinstalledModulePath` 与 `lastUpdateMillis` 在旧世代设备上可能整体缺失，脚本必须显式写 `none` 而不是省略列（第廿一章的输出契约）。

写入侧的字段顺序见 `apexd.cpp:3257-3259`（构造 `ApexInfo` 的实参表）与 `apexd.cpp:3267-3275`（`CollectApexInfoList` 手写根元素、先输出一条序列的 active 集合再输出 inactive 集合）。

`lastUpdateMillis` 值得单独说一句：它是**这份清单里唯一的时间坐标**，来自 `GetBlockApexLastUpdateSeconds` 失败后回退到的 `stat(...).st_mtime`（`apexd.cpp:3246-3254`）。它的语义是"这个包文件的上次修改时间"，**不是**"这次更新到达设备的时间"，也不是"修复被编译进这个版本的时间"。治理上它可以用来给"这台设备上的模块集合何时变过"提供一个粗粒度线索，但不能进任何补丁到达性结论——这也是本文把时间坐标全部交给 SPL 之外的独立通道、而不把 mtime 当成第四类时间锚点的原因（第十二章之三）。

这十个属性合起来正好是 M1/M2/M4 三个面的最小可读集合，也正好是第廿二章脚本 `APEX` 行的八段来源。

### 八之四 本节排除的讹传

`minSupportedVersion` 在 apexd、`apex_manifest.proto` 与 `ApexManager.java` 中均未见强制实现，可引用的只有八之一的文档规则与 `apexd.cpp:3406-3407` 的一句注释——

> "In case of a downgrade install, it differs from the actual, but this is not a supported case."

即**降级是"未受支持的情形"，而不是"被策略拒绝的情形"**，这个区别对第九章的状态分析有意义。`preinstalledApex` / `defaultVersionCode` 也不是 proto 字段名；对应事实的可读表述是 XML 属性 `isFactory` 与 Soong 侧的 `-se version 0 ${default_version}`（`build/soong/apex/builder.go:109`）。

---
## 九、存在形态：内置版本、更新版本与挂载点的四层语义

### 九之一 内置版本是兜底，不是"已修"

> "The APEX manager defaults to using these packages if no updates are available." — [APEX file format](https://source.android.com/docs/core/ota/apex)

这句话给出了设备侧最容易读错的一条默认规则：**没有更新时，模块照样存在、照样挂载、照样提供服务**，只是它的内容停留在出厂分区里。因此"设备上能看到 `/apex/com.android.conscrypt`"这类观察，**对补丁状态零信息量**。本文把它列为第一个误区（第廿五章），并在快照设计里强制要求形态读数与版本读数成对出现。

内置 APEX 文件的搜索位置由 `system/apex/apexd/apex_constants.h` 给出常量名与值（`:30-40,51`）：

```
kApexDataDir             = "/data/apex"
kActiveApexPackagesDataDir = "/data/apex/active"
kApexBackupDir           = "/data/apex/backup"
kApexDecompressedDir     = "/data/apex/decompressed"
kOtaReservedDir          = "/data/apex/ota_reserved"
kApexPackageSystemDir    = "/system/apex"
kApexRoot                = "/apex"
```

同段还声明 `system_ext`、`product`、`vendor`、`odm` 四个分区下的 `apex` 目录。这五个内置位置加一个数据目录，构成 M2 存在形态的全部可枚举路径：**清点"这台设备上有几个 APEX 文件"必须按分区分别计数**，否则 `/vendor/apex` 里的厂商模块会与 `/system/apex` 里的平台模块混成同一个数字。

### 九之二 挂载点：实例目录与"最新版"别名

> "After reboot, the APEX is mounted at the `/apex/<apex_name>@<version>` directory. Multiple versions of the same APEX can be mounted at the same time." — [APEX file format](https://source.android.com/docs/core/ota/apex)
>
> "the one that corresponds to the latest version is bind-mounted at `/apex/<apex_name>`. Clients can use the bind-mounted path to read or execute files from APEX." — 同页

两条合起来给出三层可读信息，而且它们彼此不等价：

| 观察 | 含义 | 归属用途 |
| --- | --- | --- |
| `/apex/<name>@<version>` 实例目录 | 该版本**已挂载** | M4：激活证据 |
| `/apex/<name>` 别名目录 | 指向其中"最新版本" | 只说明选择结果，不说明有多少版本共存 |
| 同一 `<name>` 的多个实例目录 | 新旧共存（内置 + 更新） | M2：可更新形态的证据 |

bind mount 的实现见 `apexd.cpp:1258` 的注释——

> "// Bind mount the latest version to /apex/<package_name>."

——而挂载这个动作本身在策略里被单独授权：`system/sepolicy/private/apexd.te:83-84` — `allow apexd apex_mnt_dir:filesystem { mount unmount };`。这也解释了第二章的第一个混用：**目录存在性是激活结果，不是模块存在性的定义**。

### 九之三 启动顺序也是一种形态：bootstrap 模块

有两个专用进程与两个专用属性把"早启动就需要的模块"与"普通模块"分开：

- `system/apex/apexd/apexd.rc:13` — `service apexd-bootstrap /system/bin/apexd --bootstrap`；`:23` — `service apexd-snapshotde /system/bin/apexd --snapshotde`
- `system/core/rootdir/init.rc:85-92` — "# Run apexd-bootstrap so that APEXes that provide critical libraries" 后跟 `exec_start apexd-bootstrap`
- manifest 侧的对应字段：`apex_manifest.proto:98` — `bool bootstrap = 16;`

对归属分析的直接含义：**如果某个库由 bootstrap 模块提供，那么它在系统其余部分开始使用之前就已经被选定了**，任何"更新后行为差异"的分析都要把这条启动顺序算进去；而 `apexd.rc:1-8` 里 `disabled # does not start with the core class` 与 `reboot_on_failure reboot,apexd-failed` 两句说明 apexd 的失败策略是重启而非降级——**"模块挂载失败"在设备上的表现是启动异常，而不是某个模块静静消失**。这是第十三章把"读不到"与"不存在"分开处理的行为学依据。

### 九之四 扁平化：有内容、无模块通道

> "Note: In Android 15, Flattened APEX is deprecated." — [APEX file format](https://source.android.com/docs/core/ota/apex)
>
> "Flattened APEXes can be updated only via a regular OTA." — 同页
>
> "Mixing flattened and non-flattened APEXes in a device is NOT supported." — 同页

三条合起来是 M2 与 M3 的耦合点：**扁平化形态让"模块"这个归属单位在分发侧消失**——同一段代码仍在，但它只能随整机镜像更新。而 `ro.apex.updatable` 是这条分叉的开关：文档给出 `PRODUCT_PROPERTY_OVERRIDES += ro.apex.updatable=true`，构建侧对应 `build/main/target/product/updatable_apex.mk:20` — `PRODUCT_SYSTEM_PROPERTIES := ro.apex.updatable=true`，策略侧给出它的属性类型 `property_contexts:1505` — `ro.apex.updatable u:object_r:exported_default_prop:s0 exact bool`。**exported 类型意味着这一位是可读的**（第十三章给判定依据），因此"这台设备的 APEX 是否走模块通道"是一个应当进入快照、而且通常能进快照的读数。

---

## 十、生命周期状态：安装不等于激活，会话是原子的

### 十之一 官方给出的时序

> "If the APEX file is verified, the internal database of the APEX manager is updated to reflect that the APEX file gets activated at next boot." — [APEX file format](https://source.android.com/docs/core/ota/apex)
>
> "the newly installed version shadows the version already present in the built-in partition. But unlike APKs, the newly installed version of the APEX is only activated after reboot." — 同页

第二句里有三个对治理要紧的词：**shadows**（遮蔽，新旧并存）、**unlike APKs**（与 APK 模块的关键差异）、**only activated after reboot**（生效需要重启）。于是"已安装 / 已激活"必须分成两态记录，任何把包管理器的"已安装"当成"已生效"的清点都会在同一个设备上得到两种互相矛盾的结论——取决于采集发生在重启前还是重启后。本文把这一对状态做成快照码位 `apex_installed_vs_active_gap`，并允许 `isActive` 与实例目录两个来源交叉验证（第八章给出 XML 的 `isActive` 属性，第九章给出 `@<version>` 实例目录）。

**唯一公开写明的例外**是 manifest 里的无重启更新位：`apex_manifest.proto:85` — `bool supportsRebootlessUpdate = 13;`。它是每个模块自带的布尔声明，意味着"重启后才激活"这条规则按模块而异，而不是按设备而异。

### 十之二 staged 会话：批量变更的原子单位

> "During apex installation, staged sessions located in /metadata/apex/sessions mutate the active set of APEXes." — `system/apex/apexd/apexd.cpp:2221-2222`
>
> "If a module is in both a staged dir and the active dir, the APEX with a higher version will be picked. If the versions are equal, the APEX in staged dir will be picked." — 同文件 `:3400-3403`

会话目录与数据目录在源码里分别是 `apex_constants.h:57` — `kStagedSessionsDir = "/data/app-staging"` 与 `system/core/rootdir/init.rc:622-625` 的 `mkdir /metadata/apex/sessions 0700 root system`。**注意 0700 且属主是 root/system**：这条权限位决定了第廿二章脚本对会话目录的读数在普通 shell 下预期失败，脚本必须把它当作"未采到"而不是"无会话"。

会话状态机的取值逐字来自 `system/apex/proto/session_state.proto:24-32`：

```
UNKNOWN  VERIFIED  STAGED  ACTIVATED  ACTIVATION_FAILED
SUCCESS  REVERT_IN_PROGRESS  REVERTED  REVERT_FAILED
```

九个状态里有三个（`ACTIVATION_FAILED`、`REVERT_IN_PROGRESS`、`REVERT_FAILED`）是**归属分析必须单独归档的**：它们表示"一批变更正在回滚或回滚失败"，此时设备上的模块集合既不是变更前也不是变更后。这与 Mainline 页的原子性声明是同一件事的两面：

> "The module package installs and rolls back atomically; either all modules that need to be updated are updated or none are updated." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

原子性对读数的用处是**批次指纹**：一批模块更新会整体生效或整体不生效，所以版本序列上的整齐跳变是批次边界，而"部分模块新、部分模块旧"这种形态本身就应该触发对会话状态的追问（第廿三章的漂移比对据此设计 `pair_*` 码位）。

`:3400-3403` 那条"等版本时取 staged 侧"的规则还有第二个含义：**内容与内置版本相同（versionCode 相等）的包也会生效**。结合第八章"同版本号可被描述为合法更新"，得到本文对版本证据的降级结论——**versionCode 只能证明"被选择的是哪一个包"，不能证明"这个包里有什么修复"**。

### 十之三 停用与卸载：公开语义只到这一层

文档对 apexd 的职责给出三种动作：

> "The APEX manager (or apexd) is a standalone native process responsible for verifying, installing, and uninstalling APEX files." — [APEX file format](https://source.android.com/docs/core/ota/apex)

源码侧的 `DeactivatePackage()`（`apexd.cpp:1290-1301`）只做卸载挂载点这一动作。本文据此声明两条边界：

1. **"deactivation" 是挂载层面的动作，不是一个持久状态。** 常被引用的 `/data/apex/deactive` 目录在 `apex_constants.h`（本文核对 android12/14/15/16/17-release 五个分支）与 `init.rc` 中**都不存在**，因此任何"读取 deactive 目录判断模块被停用"的脚本在真实设备上必然空转——这是第二章第四个混用的技术根源之一。
2. **降级只是"不受支持"，不是"被拒绝"。** `apexd.cpp:3406-3407` — "In case of a downgrade install, it differs from the actual, but this is not a supported case."；而 `minSupportedVersion` 本文未在 apexd、proto 或 `ApexManager.java` 中找到强制实现。因此"这台设备会不会跑一个更旧的模块版本"在公开证据上**无法回答为否**，只能在快照里记录观测到的版本号并交由漂移比对判断异常。

### 十之四 状态记录的最小完备集

| 记录项 | 来源 | 缺失时的正确读法 |
| --- | --- | --- |
| 模块名 | `apex-info-list.xml` / 实例目录名 | 采集面不可读 → 未采到 |
| `versionCode` | 同上（`xs:long` 必填） | schema 要求必填，因此读不到版本只说明**读数面不完整**，记为缺口 |
| `isFactory` | 同上 | 区分出厂/更新形态 |
| `isActive` | 同上 + 实例目录交叉验证 | 与"已安装"分别记录 |
| `partition` | 同上 | 决定谁有权推送更新 |
| 会话状态 | `/metadata/apex/sessions`（0700 root:system，预期不可读） | 读不到时**不得**推断"无进行中会话" |
| 平台版本 | `getprop ro.build.version.release` / `sdk` | 缺它则第五章"能力缺席"无法判别 |

---

## 十一、构建装配与归属：为什么"能构建出来"不等于"能归上去"

### 十一之一 构建期确实强制了一部分"真可更新"

Soong 的 `apex` 模块类型在 `build/soong/apex/apex.go:125-129` 给出 `Updatable *bool` 及其注释：

> "Whether this APEX is considered updatable or not. When set to true, this will enforce additional rules for making sure that the APEX is truly updatable. To be updatable, min_sdk_version should be set as well. … Default is true."

而这三条规则不是文档愿望，是构建失败的硬检查（`apex.go:2569-2576`）：

```
ctx.PropertyErrorf("updatable", "updatable APEXes should set min_sdk_version as well")
"updatable APEXes should not set min_sdk_version to current…"
"updatable APEXes can't use platform APIs"
```

第三行值得单独注意：**可更新 APEX 在构建期被禁止使用 platform API**。这与第二章引用的模块契约句（"Module updates don't introduce new APIs…"）是同一约束的两个投影——一个写在文档里给读者，一个写在 Soong 里给构建器。对归属分析的含义是：**如果一个缺陷涉及平台内部 API 的使用，那么它的修复载体在构造上就不太可能是可更新模块**（第十八章的反例章节会把这个性质用到内核与微架构缓解上）。

其余被本文当作"归属线索"使用的构建声明，逐字为：`apex.go:48,50,52` 注册 `apex` / `apex_vndk` / `prebuilt_apex` 三种模块类型；`:73-75` 的 `Manifest *string`（注释 "// Json manifest file describing meta info of this APEX bundle... Default: \"apex_manifest.json\""）；`:81-84` 的 `File_contexts *string`（注释 "// For platform APEXes, this should points to a file under /system/sepolicy…"）；`:364` 的 `Apps`（"// List of APKs that are embedded inside this APEX."）；`:376` 的 `Overrides []string`；`:409-411` 的 `Min_sdk_version *string`。

### 十一之二 装配落点与产物身份

装配侧的产品变量为 `build/make/core/product.mk:168,285,374` — `PRODUCT_APEX_SYSTEM_SERVER_JARS`、`PRODUCT_DEFAULT_APEX_PAYLOAD_TYPE`、`PRODUCT_APEX_BOOT_JARS`；签名侧为 `key.go:32` 注册 `apex_key`、`:54,56` 的公钥/私钥注释（"// Path or module to the public key file in avbpubkey format. Installed to the device." / "…private key file in pem format. Used to sign APEXs."），以及 `apex.go:391-398` 的 `Key` / `Certificate` 与注释 "If this is \"foo\", foo.x509.pem and foo.pk8 under PRODUCT_DEFAULT_DEV_CERTIFICATE are used"。

manifest 形态的代际切换也发生在构建侧：`builder.go:388` — "// From R+, protobuf binary format (.pb) is the standard format for apex_manifest"；`:188-189` 说明 zip 内同时放入 `apex_manifest.json` 与 `apex_manifest.pb`；`:376` — "// b/143654022 Q apexd can't understand newly added keys in apex_manifest.json prepare stripped-down version"。

这三条放在一起给出一个对清点很实际的结论：**同一个模块在不同平台代际上的容器内容不完全同形**（Android 10 的读取器需要剥离版 JSON，Android 11+ 以 .pb 为准）。用固定文件名集合去校验跨代际设备的 APEX，会在旧设备上产生假阴性。

### 十一之三 为什么构建口径不能承担归属

`prebuilt_apex` 的存在本身就是答案的一半：`prebuilt.go:99` — "// device (/apex/<apex_name>). If unspecified, follows the name property." 与 `:104` 的 `Source_apex_name` 说明**设备上的 APEX 可以是"直接放进来的现成包"**，而不是本地源码树的产物。于是：

1. **产物身份（包名 + 版本）与源码身份（哪个仓库哪个提交）之间没有公开的可反查字段。** 从设备上读到一个 `com.android.media` 的 versionCode，不能反推到 AOSP 的哪个提交——这个反查在设备侧不存在，在公开文档里也没有给出映射表。
2. **签名主体只到"谁签发"这一层。** `apex_pubkey` 与 `apex_key` 回答签发者，不回答内容集合。
3. **构建侧的完整口径在另一篇。** 分区装配、镜像组装、构建身份（test-keys 与正式签名）的判定，本文一律引向[平台代码组织与构建体系篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)第十一章，不在这里重复；本文只取一个结论：**构建能证明"这个包怎么来的"，不能证明"它修了什么"。**

---

## 十二、更新通道：两种到达方式、厂商侧的选择痕迹，与 SPL 的解释力边界

### 十二之一 通道只有两条，而且第二条由伙伴掌控

> "End-user devices can receive Mainline updates from the Android Google Play system update feature (which runs on the Google Play Store infrastructure) or through a partner-provided over-the-air (OTA) mechanism." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

这句话的治理含义：**"模块可独立更新"是一个能力声明，不是一条到达保证**。到达保证取决于设备属于哪一类（第六章：GMS 与 AOSP 签名的包名前缀与签名者差异）以及伙伴是否使用第二条通道。Conscrypt 模块页是少数把"不依赖 OTA"写到模块级文档里的一处：

> "The Conscrypt module accelerates security improvements and improves device security without relying on OTA updates." — [Conscrypt module](https://source.android.com/docs/core/ota/modular-system/conscrypt)
>
> "Now with Mainline, it's possible to update certificates more frequently using Mainline train updates." — 同页

"train updates"（列车式更新）这个词给出了通道的时间结构：**模块更新按批次发出，不等整机镜像**。它与第十章的原子性合起来，就是"批次"这一归属坐标的来源。

### 十二之二 厂商侧的选择属性：能读到痕迹，不能断言语义

策略侧存在一整类与厂商选择相关的属性：`property_contexts:356-357` —

```
persist.vendor.apex.    u:object_r:apexd_select_prop:s0
ro.boot.vendor.apex.    u:object_r:apexd_select_prop:s0
```

其类型由 `public/property.te:288` 的 `system_public_prop(apexd_select_prop)` 声明，而该宏的注释是"Define a /system-owned property with no restrictions"（`te_macros:954-956`）——**这类键是可读的，且没有任何域限制**。名字（`_select`）、前缀形态（`ro.boot.` 来自命令行/引导参数）与所属分区共同指向"厂商可以按模块选择 APEX 来源"这一族行为。

本文在这里明确停步：**未在公开源码中定位到消费这些键的实现文件**（`apex_selecter.cpp` 在核对的 ref 下返回 404），因此不声称"某属性把某模块通道关掉了"，只把这类键的存在与取值当作**厂商侧意图的可读痕迹**记录进快照（第廿二章的 `apex_vendor_select_*` 码位）。第三章表格里"可读到通道是否被厂商关掉"这一格，在本章收窄为"可读到厂商侧选择属性的存在与取值"。这是本文刻意保留的一条未公开文档化边界，收入第廿八章。

### 十二之三 SPL 解释不了模块状态，原因有三条公开证据

第一条是坐标本身的定义：

> "The component is also a factor in how users get updates. A bug in the framework or kernel requires an over-the-air (OTA) firmware update that each OEM needs to push. A bug in an app or library published in Google Play (for example, Gmail, Google Play Services, or WebView) can be sent to Android users as an update from Google Play." — [Updates and resources](https://source.android.com/docs/security/overview/updates-resources)

注意这句话的分类轴是 framework/kernel 与 Play 上的 app/library，**Mainline 没有被单列为一类**——公告侧那句含糊的承认（"Devices with Android 10 and later may receive security updates as well as Google Play system updates."）是唯一的连接点，而它不落到任何一行数据上。

第二条是 SPL 的覆盖声明本身：

> "Security vulnerabilities that are documented in this security bulletin are required to declare the latest security patch level on Android devices. Additional security vulnerabilities that are documented in the device / partner security bulletins are not required for declaring a security patch level." — [2024-03-01 FAQ](https://source.android.com/docs/security/bulletin/2024-03-01)

即 **SPL 只为本公告页列出的条目背书**，模块侧的到达情况不在这张坐标上。

第三条是源码可见性的时滞：

> "If the security bug is in an AOSP component, the fix is pushed out to AOSP after the OTA is released to users." — [Updates and resources](https://source.android.com/docs/security/overview/updates-resources)
>
> "Android platform security fixes are merged into AOSP 24–48 hours after the quarterly security bulletin is released" — [ASB overview](https://source.android.com/docs/security/bulletin/asb-overview)

于是"用 AOSP 源码核对某模块版本是否含修复"这条审计路径，在时间上先天滞后，在版本上又缺映射（第六章）。

### 十二之四 唯一能把"超出 SPL 的修复"读出来的公开机制

> "The supplemental security patch XML feature lets OEMs report CVEs patched beyond the declared security patch level (SPL) by providing a standardized XML file, so OEMs receive credit for continuous patching." — [Supplemental security patches](https://source.android.com/docs/security/overview/supplemental-security-patches)
>
> "OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product), ensuring that security patches from both the framework and hardware-specific components are accounted for." — 同页

这是设备侧**唯一**一个由平台定义、用来登记 SPL 之外已修 CVE 的读数面，路径为 `/etc/security/supplemental_security_patches.xml`。但它有两个限制使它不能替代模块归属：它是**厂商自愿提供**的（机制目的是让 OEM "receive credit"），并且它登记的单位是**分区与 CVE**，不是模块与版本。因此本文把它列为快照码位之一（有则记录、无则记为缺口），并在第十九章用它解释"为什么一台设备可能已修却读不出来"。

### 十二之五 三条通道 × 模块可见性小结

| 通道 | 单元 | 公告是否记录 | 设备侧证据 | 归属可用性 |
| --- | --- | --- | --- | --- |
| Play 系统更新 | 模块批次（原子） | 仅以 Subcomponent 与 CVE 两列弱记录 | 模块名 + `versionCode` + `isActive` | 可派工，不能验证内容 |
| 伙伴 OTA（模块） | 模块包（随镜像或组件包） | 同月公告 + 厂商公告 | 同上 + `partition` | 依赖厂商公告口径 |
| 随整机镜像 | 分区/组件 | 公告 + SPL 声明 | 版本标识与 SPL | 能验证到达，不能拆分模块 |

---
## 十三、设备侧可读面清单：哪些能读，是被策略写定的

第四章把 M6（可证明性）单列为一个面，本章给出它的实际内容。本文核对到 **10 个候选读数面**，并把每一个的可读性判定挂到具体的 SELinux 策略行上——之所以要挂到策略行，是因为**跨设备比较时，"读不到"最常见的解释不是"这里没有东西"，而是"这台设备的策略不允许你读"**。

### 十三之一 文件面的四行授权

`system/sepolicy/private/shell.te` 里有两组直接关于 APEX 的规则，注释逐字如下：

> "# Allow shell to read /apex/apex-info-list.xml and the vendor apexes" — `:162`
>
> `allow shell apex_info_file:file r_file_perms;` — `:163`
>
> `allow shell vendor_apex_file:file r_file_perms;` / `allow shell vendor_apex_file:dir r_dir_perms;` / `allow shell vendor_apex_metadata_file:dir r_dir_perms;` — `:164-166`
>
> "# Allow shell to read updated APEXes under /data/apex" — `:168`
>
> `allow shell apex_data_file:dir search;` — `:169`

第二组只给了 **`search`**，没有给 `r_dir_perms`。这一字之差在本章很重要：`search` 允许**穿过**目录去访问已知路径，不允许**列出**目录内容。于是"读 `/data/apex` 下已激活模块"这件事的正确读法是：**XML 里已经告诉你有哪些激活模块和它们的路径，你不需要列目录；而试图 `ls /data/apex/active` 在授权良好的设备上也会失败，这是策略，不是设备异常。**

文件标签本身在 `private/file_contexts`：

> `/apex  u:object_r:apex_mnt_dir:s0` — `:55`
>
> `/apex/(\.(bootstrap|default)-)?apex-info-list.xml u:object_r:apex_info_file:s0` — `:62`
>
> `/data/apex(/.*)?  u:object_r:apex_data_file:s0` — `:625`
>
> `/system/bin/apexd  u:object_r:apexd_exec:s0` — `:414`

注意 `:62` 那条正则的形态：`apex-info-list.xml` 在**标签层面**有 **bootstrap / default / 普通** 三种写法。这与第九章的 bootstrap 进程相呼应（`private/file_contexts:56` 还把 `/bootstrap-apex` 单独立成同标签的挂载根）。但要说清一件事：`apex_constants.h:78` 只给出**一个**路径常量 `kApexInfoList = "/apex/apex-info-list.xml"`，而写入点 `EmitApexInfoList()`（`apexd.cpp:1402-1423`）**无论 bootstrap 与否都打开同一个 `kApexInfoList`**（`:1415`）——`is_bootstrap` 只影响是否跳过未激活内置包（条件在 `:1407`，注释逐字为 `:1405-1406` 的 "we skip for non-activated built-in apexes in bootstrap mode / in order to avoid boottime increase"）。因此本文只把"标签有三种变体"当作 SELinux 侧的事实，**不声称设备上能列出三个 XML 文件**；快照脚本对主读数面始终只取 `/apex/apex-info-list.xml` 这一条路径。

这条 bootstrap 跳过逻辑对读数的影响很实际：只有 `IsMountBeforeDataEnabled() || !is_bootstrap` 成立时才计算未激活集合，所以**当 `mount_before_data` 未开启、且这份清单由 bootstrap 阶段写出时，未激活的内置模块整批不出现在清单里**。治理上这给出"同一台设备两次采集条目数不同"的一种**非风险成因**，也是第廿三章把条目数变化单独设码位（`pair_active_count_change`）而不一律升级为风险的原因。

`private/shell.te` 里的 apex 授权可以完整枚举（android17-release 上 `grep -n apex` 的全部命中就是 `:163`、`:164-166`、`:169`、`:172`、`:281`、`:384`、`:569` 这几条），其中**没有** `apex_mnt_dir:dir`，也**没有** `apex_module_data_file`。也就是说 `ls /apex` 与 `ls /data/misc/apexdata` 在标准策略设备上预期失败。本文因此把 `/apex` 目录当**可选交叉面**而不是主读数面，主读数面始终是那份 XML；同时把"这两个目录竟然读到了"做成一个反向码位——读到了说明这台设备的策略不是标准策略，快照的其它结论都要跟着降档。

### 十三之二 属性面的四类可读性

属性可读性由**类型**决定，而类型定义里写死了 neverallow：

| 属性 | 类型（`property_contexts`） | 类型定义（`public/property.te`） | 宏的 neverallow 语义 | 非 coredomain 可读？ |
| --- | --- | --- | --- | --- |
| `ro.apex.updatable` | `exported_default_prop`（`:1505`，exact bool） | `system_vendor_config_prop`（`:155`） | "no restrictions"（读侧）＋只允许 vendor 侧写 | 可读 |
| `apex.all.ready` | `apex_ready_prop`（经 `:364` 的 `apex.` 前缀命中，prefix bool） | `system_restricted_prop`（`:55`） | 只限制 `property_service set` | 可读 |
| `apexd.status` | `apexd_prop`（`:976`，exact enum starting/activated/ready） | `system_internal_prop`（`:6`） | `neverallow { domain -coredomain } $1:file no_rw_file_perms;` | **不可读** |
| `ro.apexd.changed_active_apexes` | `apexd_private_prop`（`:977`） | 同类 private 语义 | 同上 | **不可读** |
| `persist.vendor.apex.` / `ro.boot.vendor.apex.` | `apexd_select_prop`（`:356-357`） | `system_public_prop`（`:288`） | "no restrictions" | 可读 |

`system_internal_prop` 的宏注释与 neverallow 逐字为：

> "# Define a /system-owned property used only in /system" — `public/te_macros:928`
>
> `neverallow { domain -coredomain } $1:file no_rw_file_perms;` — 同宏 `:933-934`

这里有个容易被忽略的实现细节：属性值不是通过 property service 读的，而是通过 `/dev/__properties__` 的**文件映射**读，所以这条 neverallow 管的是 `:file` 权限——**它剥夺的是读的能力，不只是写的能力**。因此 `getprop apexd.status` 从 adb shell（shell 域，非 coredomain）取不到值，是**编译期就被 neverallow 保证的行为**，与厂商是否加白名单无关。这是本章最有治理价值的一条结论：

**"apexd 处于哪个阶段"这个问题，在标准策略下的设备上对 shell 是不可见的；任何声称能稳定读到它的清点流程，要么运行在放宽策略的构建上，要么根本没在验证这件事。**

`apex.` 前缀（`:364`）还顺带说明：`apex.all.ready` 没有独立条目，它的类型是**前缀匹配**得来的；而 `apexd.` 前缀（`:347`）覆盖 `apexd.config.*` 与 `apexd.status`。写解析脚本时不能用"整名查表"，必须按前缀规则处理。

### 十三之三 服务面：两道彼此独立的门

`apexd` 把服务注册成 `apex_service`（`private/apexd.te:6` — `add_service(apexd, apex_service)`；`apexd.rc:2` — `interface aidl apexservice`）。非 coredomain 想调用它，要依次通过两道**互不相干**的门：

1. **SELinux 的 service find 门。** `shell.te:378-384` 的授权写法是排除式的：

   > `allow shell {` / `  service_manager_type` / `  -apex_service` / `  -dnsresolver_service` / … — 同段

   也就是 **shell 域被明确排除在 `apex_service` 之外**。因此 `cmd apex` 与 `dumpsys apexservice` 在 shell 下都不是"数据少"，而是**根本到不了服务**。
2. **服务实现里的调用者校验。** 即使换到允许 find 的域，还要过代码层的门：

   > "Only root and system_server are allowed to call " — `system/apex/service/apexservice.cpp:66`
   >
   > `CheckCallerSystemOrRoot("getAllPackages")` — 同文件 `:176`
   >
   > "Only root, keystore, and system_server are allowed to call " — 同文件约 `:406`

这两道门在故障归因上指向不同的责任方：**前者是策略与设备类别问题（user 构建 / 厂商加白名单），后者是调用者 UID 问题**。第廿二章脚本因此对服务面失败输出两种码位（`*_denied_selinux` 与 `*_denied_caller`），并按输出文本区分——SELinux 拒绝与服务的错误串不同，混淆它们会让运维往错误的方向提工单。

顺带纠正一个流传颇广的读法：`apexservice.cpp` 的 dump 实现本身（`:775` 起，输出 `ACTIVE PACKAGES:` 与 `SESSIONS:`）确实没做调用者校验，但这不足以让 `dumpsys apexservice` 可用——**它在第一道门上就被排除**。"dump 函数没有权限检查"和"shell 能读到 dump"是两回事。

### 十三之四 读数面总表

| # | 读数面 | 采集动作 | 依据 | 预期 |
| --- | --- | --- | --- | --- |
| 1 | 平台版本与 SDK | `getprop ro.build.version.release` / `.sdk` | 通用可读 | 必读，第五章的"能力缺席"判定依赖它 |
| 2 | `ro.apex.updatable` | `getprop` | 十三之二：exported | 决定 M2/M3 分叉 |
| 3 | `apex.all.ready` | `getprop` | restricted，可读 | M4 的就绪位 |
| 4 | `apexd.status` | `getprop` | internal，**预期读不到** | 读不到即正常，不得当作故障 |
| 5 | 厂商选择属性族 | `getprop` 前缀扫描 | public，可读 | 十二之二的"意图痕迹" |
| 6 | `apex-info-list.xml` | `cat` | 明确授权（两处，`:163` 与 `:281`） | **主读数面** |
| 7 | `/vendor/apex` | `ls` | 明确授权 `:164-165` | 厂商自带包，与平台包分开计 |
| 8 | `/apex` 与 `/data/misc/apexdata` 目录 | `ls` | 十三之一：`shell.te` 无 `apex_mnt_dir:dir`、无 `apex_module_data_file` | 可选交叉面，失败为预期；读到即反向异常 |
| 9 | `cmd apex getActivePackages` / `dumpsys apexservice` | `cmd` / `dumpsys` | 十三之三：两道门 | 预期失败，失败原因要分流 |
| 10 | `/etc/security/supplemental_security_patches.xml` | `cat` | 十二之四：机制存在、厂商自愿 | 有则登记，无则记缺口 |

这张表就是第廿二章脚本的骨架：**每一行的预期结果都被写进脚本，包括预期失败的行**。一个把"读不到"当作故障上报的脚本，会在第一批正常设备上被运维废弃——这是本系列反复验证过的工程现实。

---

## 十四、模块身份的三个可见面：字段名并不互相兼容

### 十四之一 AIDL parcelable：程序侧的身份形状

`system/apex/apexd/aidl/android/apex/ApexInfo.aidl:19-54` 逐字给出这个 parcelable 的字段：

```
parcelable ApexInfo {
  @utf8InCpp String moduleName;
  String modulePath;
  String preinstalledModulePath;
  long versionCode;
  @utf8InCpp String versionName;
  boolean isFactory;
  boolean isActive;
  boolean hasClassPathJars;      // 注释：Populated only for getStagedApex() API
  boolean activeApexChanged;
  Partition partition;
```

三个值得单独记住的细节：**`versionCode` 是 `long`**（与第八章 XML 的 `xs:long` 一致，与 `int` 假设不兼容）；**`preinstalledModulePath` 是独立字段**，也就是说"出厂在哪"与"现在在哪"被同时携带——这正是第九章"内置版本是兜底"这条规则在数据结构上的体现；**`hasClassPathJars` 只对暂存（staged）查询填充**，所以"这个包会不会改动 classpath"这个问题在激活态的返回里读不到，属于 M4 与 M5 的交界处。

### 十四之二 包管理侧：只有标志位，没有版本

> "In APEX terminology, this corresponds to packages that are currently active, i.e. mounted and available to other processes of the OS." — `frameworks/base/core/java/android/content/pm/PackageManager.java:1355`
>
> `public static final int MATCH_APEX = 0x40000000;` — 同文件 `:1359`
>
> "// Whether the package is an APEX package." / `public boolean isApex;` — `PackageInfo.java:493-494`

这一面只回答两件事：**是不是 APEX 包**、**是否处于激活（已挂载且可用）状态**。注意那句定义里"active = mounted and available"的措辞与第十章的"activated after reboot"是同一语义，可以互为交叉验证。

框架内部通过 `ApexManager` 取服务侧数据（`frameworks/base/services/core/java/com/android/server/pm/ApexManager.java:22` — `import android.apex.ApexInfo;`）。**本文未在 `main` 分支找到 `PackageManager.getInstalledApexInfo()` 这一公开方法、也没有 `android.content.pm.ApexInfo` 这个类、更没有名为 `APPEX` 的匹配常量**——这三者在二手资料里出现频率很高，一律不采用；公开侧的正确写法是 `MATCH_APEX` 加 `PackageInfo.isApex`。

### 十四之三 shell 侧：动词表与输出标签

`cmd apex` 的帮助串（`apexservice.cpp:831-835`）只列出四个动词：

```
  help - display this help
  getAllPackages - return the list of all packages
  getActivePackages - return the list of active packages
  getStagedSessionInfo [sessionId] - displays information about a given session
```

`dumpsys apexservice` 的段落与字段标签（`:775`、`:791`、`:397`）逐字为 `ACTIVE PACKAGES:`、`SESSIONS:`，以及每条记录里的 `Module: `、`Version: `、`VersionName: `、`Path: `、`IsActive: `、`IsFactory: `、`Partition: `。

### 十四之四 三面字段名不一致的实际后果

| 语义 | XML 面（第 8 章） | AIDL 面（本章之一） | shell 面（本章之三） |
| --- | --- | --- | --- |
| 元素 / 记录名 | `<apex-info>`（xsd `:36`） | `ApexInfo` | 每条以 `Module: ` 起 |
| 模块名 | 属性 `moduleName`（xsd `:38`，required） | `moduleName` | `Module: ` |
| 版本码 | `versionCode`（xs:long，required） | `versionCode`（long） | `Version: ` |
| 出厂标志 | `isFactory` | `isFactory` | `IsFactory: ` |
| 激活标志 | `isActive` | `isActive` | `IsActive: ` |
| 分区 | `partition`（枚举 5 值，xsd `:26-33`） | `partition`（枚举） | `Partition: ` |
| 安装路径 | `modulePath`（required） | `modulePath` | `Path: ` |
| 内置路径 | `preinstalledModulePath`（**可选**） | `preinstalledModulePath` | — |
| 上次变化时间 | `lastUpdateMillis`（**可选**） | — | — |
| 类路径标志 | `provideSharedApexLibs`（required） | `hasClassPathJars` | — |

三点结论：

1. **解析器不能跨面复用正则**——XML 是属性、AIDL 是驼峰字段、dumpsys 是带空格的标签前缀。第廿二章脚本为每个面写独立取值函数，而不是"一个 grep 走天下"。
2. **只有 XML 面在 shell 下稳定可读**，因此它是主面；AIDL 与 shell 面在标准策略下预期失败，它们的价值是"当失败时，能证明前一道门挡在哪"（十三之三）。
3. **`modulePath` 与 `preinstalledModulePath` 的差集是形态证据，而且在主面上直接可得**：两者路径前缀不同（例如一个在 `/data/apex/active/`、一个在 `/system/apex/`）即说明该模块运行在更新版本上，而不是出厂内置版本。三个面里只有 AIDL 侧给了 `hasClassPathJars` 这个名字，XML 侧对应的是 `provideSharedApexLibs`——**两个属性名不同、语义接近，本文不声称它们等价**，只把各自的取值分别记账（第十三章的 M5 旁证、第廿三章的 `pair_classpath_flag_shift`）。

---

## 十五、契约表面：模块被允许依赖什么，装上来时应用看到什么

### 十五之一 一句总纲加两条具体承诺

> "Module updates don't introduce new APIs. They use only the SDK and System APIs guaranteed by the Compatibility Test Suite (CTS), communicate only with each other, and use only stable C API or stable AIDL interfaces." — [Mainline](https://source.android.com/docs/core/ota/modular-system)

模块子页把这条总纲落到具体接口层：

> "Interacts with the framework only via stable @SystemApi (no @hide API usage)." — [Permission Controller module](https://source.android.com/docs/core/ota/modular-system/permissioncontroller)
>
> "Stable AIDL IPCs interface with the system server" — [Tethering module](https://source.android.com/docs/core/ota/modular-system/tethering)
>
> "Interfaces can be declared as stable (backward compatible)." — [Stable AIDL](https://source.android.com/docs/core/architecture/aidl/stable-aidl)

这三句一起把 M5 变成了一个**可检验的集合**：模块对框架的依赖面 = SDK/@SystemApi + 稳定 C API + 稳定 AIDL + 模块之间。第十一章已经看到这条约束的另一面——构建期用 `PropertyErrorf` 拒绝使用 platform API 的可更新 APEX。**同一条规则，文档侧与构建侧各出现一次，这提高了它的可信度**：它不是愿望，是被执行过的检查。

### 十五之二 装上模块之后，应用看到的 API 会变

> "Symbols (classes, methods, fields, etc.) annotated with `@hide` in the Wi-Fi module are not part of its public API surface and cannot be accessed on devices with the module installed." — [Wi-Fi module](https://source.android.com/docs/core/ota/modular-system/wifi)

这是本文与[应用安全弱点分类篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)"目标版本改变默认值"同类的现象：**同一段调用代码，在不同安装形态下行为不同**。治理含义有两层：

1. **兼容性回归会被误读成"模块引入了缺陷"。** 一台设备装了新模块后某应用崩溃，与另一台没装的设备正常，这条差异的正确描述是"契约表面不同"，而不是"某版本有问题"——除非能证明该调用在契约内。
2. **依赖 `@hide` 的应用在模块设备上必然失败。** 这类失败的归属应落在应用侧（弱点篇口径），不应写进模块的缺陷登记。

### 十五之三 classpath：模块能改动的运行时装配

`system/apex/apexd/apex_classpath.cpp:84-86` 逐字给出三条导出注释：

```
// export BOOTCLASSPATH path/to/jar1:/path/to/jar2
// export DEX2OATBOOTCLASSPATH
// export SYSTEMSERVERCLASSPATH path/to/some/jar
```

也就是说，模块可以贡献启动类路径与系统服务器类路径——这是**模块影响全局运行时**的合法通道。它对应两个可读属性：XML 面的 `provideSharedApexLibs`（第八章 schema 列出）与 AIDL 面的 `hasClassPathJars`（第十四章，注明仅对 staged 查询填充）。

治理上这构成 M5 里风险最高的一格：**一个模块改动 classpath 时，它影响的是所有进程的类解析顺序**，而不是它自己进程内的实现。因此本文把"版本跳变 + classpath 相关标志变化"列为漂移比对里需要单独追问的组合（第廿三章），而不是简单地记成一次普通更新。

### 十五之四 SDK extension：另一条版本轴

清单里的 `com.android.sdkext` 是这一轴的载体；构建侧对应 `"framework-sdkextensions"` 模块并导出 `android.os.ext`（`build/soong/apex/apex.go:2771`）。**本文未能在 `main` 分支定位到 `SdkExtensions.java` 的实现文件**（它属于 mainline 仓而非 `frameworks/base`），也未取到 `sdk_extension` 相关属性的源码常量，因此只把这条轴作为"设备可能存在两个独立版本坐标（平台版本 + SDK extension 级别）"的事实记录，不给出该如何读取它的操作步骤。这条缺口收入第廿八章。

---

## 十六、公告登记口径的能与不能

### 十六之一 公告能给出的，是候选而不是答案

本章把散在前面的口径证据合起来。公告侧关于分组的全部自述是一句话加一句列说明：

> "Vulnerabilities are grouped under the component they affect." — [2024-03-01](https://source.android.com/docs/security/bulletin/2024-03-01)
>
> "Issues are described in the tables below and include CVE ID, associated references, type of vulnerability, severity, and updated AOSP versions (where applicable)." — 同页

"where applicable" 这个词很重要：它承认**不是每张表都带版本列**。实测正是如此——本文核对到的 Mainline 表只有两列：

> "Google Play system updates / The following issues are included in Project Mainline components. / Subcomponent | CVE" — [2024-06-01](https://source.android.com/docs/security/bulletin/2024-06-01)

而同一月的 `Framework` / `System` / `Android Runtime` / `Kernel` 表都带 References、Type、Severity、AOSP version 列。**Mainline 那张表是整个公告里字段最少的一张**，这是"公告给候选、不给答案"的直接证据（第十九章用 13 组同月双表读数把这件事量化）。

### 十六之二 分档集合的实测与漂移

本文抓取 2022-09、2023-05、2023-11、2024-01/03/04/06/08/09/10/12、2025-01/02/03/06/08/09/10 共 17 个月份页面，得到的分组标题集合是：`Framework`、`System`、`Android Runtime`（较早月份写作 `Runtime`）、`Kernel`、`Google Play system updates`，再加各厂商自述表。两点值得记录：

1. **没有独立的 `Media` 表**（本系列 Framework 篇按另一口径列过"Media"这一档，本文按实测月份以 `Framework`/`System` 为准，并把这一点作为"分档集合本身会漂移"的证据登记）。媒体类缺陷在 Mainline 表里以 Subcomponent 词出现，如 `Media Codecs`、`MediaProvider`。
2. **列名漂移**：2022-09 写作 `Component | CVE`，2023-05 起写作 `Subcomponent | CVE`。任何硬编码列名的抽取工具都会在这些月份失配（第廿五章的误区之一）。

### 十六之三 三类"落不进模块表"的条目

1. **无 CVE 的低危条目**：

   > "Note: A CVE may not be issued for issues assessed as Low or NSI." — [Updates and resources](https://source.android.com/docs/security/overview/updates-resources)

   登记面先天不完整，"公告里没有"不能读成"没有缺陷"。
2. **星号条目**：

   > "Issues that are not publicly available have an * next to the corresponding reference ID. The update for that issue is generally contained in the latest binary drivers for Pixel devices available from the Google Developer site." — [2024-03-01 FAQ](https://source.android.com/docs/security/bulletin/2024-03-01)

   修复落在二进制驱动，**在模块口径外**，且公开细节缺失。
3. **厂商自述严重度的条目**：

   > "These vulnerabilities affect Qualcomm components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm." — 多个月份同文复现

   AOSP 只转录，不判定。第十八章会把这三类合并成一个命题：**有相当一部分缺陷在构造上就没有模块载体。**

### 十六之四 两条来自兄弟篇的纪律

本文沿用两条既有纪律而不重新论证：

- **"能点名模块"与"不能点名模块"要分开。** [远程代码执行篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)曾拒绝把 WebView 归为 APEX，理由是文档未如此记载；本文对 `Media Codecs` 这种一词双候选（`com.android.media` 与 `com.android.media.swcodec` 都在清单内）同样拒绝合并归类，统一标为 `MAP_AMBIGUOUS`（第六章规则 D）。
- **三态纪律。** [无线与基带篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)对 `com.android.wifi` 采用 mounted / absent / unknown 三态，并强调 absent 是通道未知、不是未加固。本文在第十三章的读数面总表上采用同一三态，只是把主语从"模块在不在"换成"**能不能证明它在**"。

### 十六之五 本章结论

公告口径对模块治理的三个用处与三个无用：

| 有用 | 无用 |
| --- | --- |
| 给出**候选归属**（哪些 CVE 属于 Mainline 组件） | 给不出模块包名（第六章实测 0 命中） |
| 给出**月份批次**与 SPL 声明边界 | 给不出模块版本与到达性 |
| 给出**双表交叉**的可能性（同 CVE 出现在两张表） | 给不出"这台设备是否已修"的判断 |

因此本文第廿一章的判定模型里，公告证据只能把结论推到"待验证"，**不能单独支撑"已修"**；能支撑"已修"的只有设备侧版本读数加上厂商公告或补充声明（十二之四）构成的组合证据。

---
## 十七、模块口径下缺陷归属的十个类型（M-AT1 … M-AT10）

本章给出本文的核心类型学。它的对象不是"缺陷的技术成因"（那是各子系统篇的事），而是**"一条缺陷在模块口径下会被归属成什么形状、以及这个形状为什么值得单独派工"**。每个类型给出：触发条件、公开依据、公告可见性、设备侧证据，以及本文脚本里对应的码位族。

### 十七之一 十个类型的定义

| 编号 | 类型 | 触发条件 | 公开依据（可指认） | 公告可见性 | 设备侧证据 |
| --- | --- | --- | --- | --- | --- |
| **M-AT1** | 模块内实现缺陷 | 缺陷代码位于某模块包内 | 公告 Mainline 表 Subcomponent 行 | 有行，但只有两列 | 模块名 + `versionCode` |
| **M-AT2** | 契约违反型 | 模块依赖了平台内部实现或新增 API | `apex.go:2576` "updatable APEXes can't use platform APIs"；模块页 `@SystemApi` 承诺 | 不可见 | 弱（行为差异，非读数） |
| **M-AT3** | 形态退化型 | 同名代码以扁平化或 non-updateable APEX 存在 | "Flattened APEXes can be updated only via a regular OTA."；"packaged as non-updateable APEX" | 不可见 | `ro.apex.updatable`、`partition` |
| **M-AT4** | 分发落差型 | 包已安装但未激活 | "only activated after reboot." | 不可见 | `isActive` 与实例目录交叉 |
| **M-AT5** | 会话半途型 | 变更处于 `ACTIVATION_FAILED` / `REVERT_*` | `session_state.proto:24-32` | 不可见 | 目录权限 0700，**预期读不到** |
| **M-AT6** | 同码异内容型 | 两个包 `versionCode` 相等但内容不同 | "same package name and a greater than or equal to version code."；"If the versions are equal, the APEX in staged dir will be picked." | 不可见 | 版本号不足以判别 |
| **M-AT7** | 词表失配型 | 公告 Subcomponent 词无法机械映射到包名 | 三页检索 `com.android.*` 命中 0；`Media Codecs` 双候选；列名 `Component`→`Subcomponent` | 可见但不可解 | 需人工词典（第二十章） |
| **M-AT8** | 登记空白型 | 低危/NSI 不发 CVE、星号条目、厂商自述严重度 | "A CVE may not be issued for issues assessed as Low or NSI."；FAQ 星号句；Qualcomm 自述句 | **本应可见却缺席** | 无 |
| **M-AT9** | 通道关闭/缺席型 | 可选模块未提供、旧代支持终止、厂商侧选择 | "an optional updatable Mainline module"；"Mainline support for Android 11 and lower is concluded…"；`apexd_select_prop` 键族 | 不可见 | 属性前缀 + 平台版本行 |
| **M-AT10** | 无载体型 | 缺陷根本不在任何模块包内（内核、微架构、器件、基带） | "A bug in the framework or kernel requires an over-the-air (OTA) firmware update…"；Pixel 表星号行 | 仅出现在非 Mainline 表 | 整机版本与厂商公告 |

### 十七之二 六条要点

1. **M-AT1 是唯一"公告与设备都能各说一半"的类型。** 公告给候选（哪个 Subcomponent、哪个月），设备给状态（哪个版本、激活与否）。两者之间必须靠映射词典缝合，所以 **M-AT1 的正确治理动作是"缝合后派工"，而不是"任一侧单独结案"**。
2. **M-AT2 到 M-AT6 都是"公告看不见"的类型**，它们不改变缺陷是否存在，只改变**能否被登记**。这是本章最重要的观察：**公告口径覆盖的是缺陷的技术落点，不覆盖归属链的断点。**
3. **M-AT6 与 M-AT3 是运营最忌讳的一对。** 前者让"版本号相同"失去含义（第八、十章），后者让"模块名称存在"失去含义（第九、十一章）。合起来意味着：**只看名字和版本号的合规检查，在两个方向上都会给出假阴性结论。**
4. **M-AT7 是纯文档缺陷，却最容易演变成真的治理事故。** 它的样本最直观：同一份修复在 `Framework` 表里带 5 列，在 Mainline 表里只带 2 列，而两张表用的是**不同的词**（`Media Framework` 与 `Media Codecs`、`System` 与 `WiFi`）。
5. **M-AT8 与 M-AT10 是"沉默的大多数"。** 星号条目按 FAQ 的说法落在二进制驱动里，低危条目可以完全不发 CVE；而 M-AT10 在 Pixel 表里以 `Cellular Modem`、`TPU`、`widevine` 这样的 subcomponent 出现，一律带星号、一律无 AOSP 版本列。
6. **CWE 对齐只能粗做，且必须标注为本文构造。** 公告的 Type 列使用权限口径的词（`EoP`、`RCE`、`DoS`、`ID`），本文按如下粗对齐使用：`ID → CWE-200 族（信息暴露）`、`EoP → CWE-269 族（特权管理）`、`RCE → CWE-119/94 族（内存与代码注入）`、`DoS → CWE-400/770 族（资源消耗）`。**这是治理沟通用的粗对齐，不是官方映射，也不是缺陷成因判定**；任何用它推导"该按哪类修复策略处理"的场合都要回到公告原文与厂商公告。

### 十七之三 派工归属的三条判据

把十个类型落到"谁来做"，本文采用三条依次适用的判据：

1. **能否点名模块**（M-AT1/M-AT9 可，M-AT10 否）→ 不能点名的，派工单位退回分区/器件，不得挂到模块负责人名下；
2. **能否点名版本**（M-AT1 可但弱，M-AT6 不可）→ 不能点名版本的，验收标准只能写成"该模块的 versionCode 不低于厂商公告给出的值"，并附**厂商公告链接**作为条件；
3. **设备侧能否只读证明**（第十三章总表 10 行里的 6 行可）→ 不能证明的，验收改为流程证据（更新批次记录、会话日志由具备权限的角色导出），**不得在报告里写成"已验证"**。

这三条判据就是第廿一章四级判定模型的输入。

---

## 十八、反例：有一大类缺陷在构造上就没有模块载体

第十七章的 M-AT10 值得单独一章，因为它决定了整个模块治理体系的**边界**——不知道边界，就会把不属于模块的东西派给模块负责人，或者反过来把属于模块的东西登记在整机版本上。

### 十八之一 三类来源，两条通道

公告侧把修复来源分成三类：

> "Fixes listed in the public bulletin come from various different sources: the Android Open Source Project (AOSP), the upstream Linux kernel, and system-on-chip (SOC) manufacturers." — [ASB overview](https://source.android.com/docs/security/bulletin/asb-overview)
>
> "Fixes from SOC manufacturers are available directly from the manufacturers." — 同页

同一页之外，`Updates and resources` 给出通道侧的分类：

> "A bug in the framework or kernel requires an over-the-air (OTA) firmware update that each OEM needs to push." — [Updates and resources](https://source.android.com/docs/security/overview/updates-resources)

也就是说，**内核与 SoC 两类来源在官方口径里就没有"模块"这一档**：前者被明确归到"必须整机的 OTA"，后者被明确归到"直接从厂商获取"。这两句合起来就是 M-AT10 的公开定义。

### 十八之二 公告表格里的实证形态

本文核对的 17 个月份页面里，Mainline 表的 Subcomponent 列从未出现 Bluetooth（作为 Mainline 表条目）、uwb、NFC、IPsec、Tethering、AdServices、NNAPI 这些词，而**内核缺陷自成 `Kernel` 表**、**器件缺陷进厂商表**。器件行的书写形态尤其能说明问题：

- Pixel 侧（`Pixel Update Bulletins`，列表头为 `CVE | References | Type | Severity | Subcomponent`）：`CVE-2025-36897 A-376027286 * / RCE / High / Cellular Modem`、`CVE-2025-32342 A-407082766 * / EoP / High / TPU`
- 同一月的 AOSP 页厂商表：`CVE-2025-32332 A-319147124 * / High / widevine`

三条共同点：**都带星号（无公开细节）、都无 AOSP 版本列、subcomponent 是器件词而不是包名词**。特别注意 `widevine` 这个词——它出现在厂商侧 subcomponent 列，而**不出现在 Mainline 的 Available modules 清单里**（第五章抄录的样本）。也就是说，同一个技术名词在两个口径下分属两侧：把它读成"一个模块"会直接导致派工错误。

### 十八之三 微架构与缓解项：一个已经论证过的反例

本系列的[微架构侧信道篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)已经给出这条口径：缓解项分布在硅、微码、内核编译期与启动参数上，**其载体在构造上不可能是可独立更新的用户空间模块**，因此只能走"由伙伴提供的 OTA 机制"这一条通道（本文第十二章已引用该句）。本文在此只借用结论，不重复论证，并把它作为 M-AT10 的原型样本：**这类缺陷的设备侧证据面（`/sys/devices/system/cpu/vulnerabilities`）与模块证据面（`apex-info-list.xml`）互不相干，两套快照不能互相替代。**

### 十八之四 内核公告与 Mainline 公告互不相干这件事

`Kernel` 表与 `Google Play system updates` 表在同一个月度公告里并列存在，且列集合不同。这带来一个治理上的直接推论：**"本月这台设备的 Mainline 侧已到位"这条结论，对内核侧的到达性不提供任何信息**，反之亦然。跨表合并成"本月已修"是第十六类常见报告写法里的错误，本文在第廿五章把它列为误区之一。

### 十八之五 本章的操作性结论

归属矩阵（第廿章）必须**显式包含"无模块载体"这一列**，否则会出现两种对称的失败：把器件缺陷派给模块负责人（延误，因为对方无权限也无源码），或把模块缺陷记在整机版本上（掩盖，因为无法验证到达）。识别"无载体"的三条可读线索是：subcomponent 是器件词、条目带星号、没有 AOSP 版本列——这三条同时成立时，归属流程应当**在起点就分流**，而不是走完整条链之后再承认无法闭合。

---
## 十九、十三组公告双表读数：一条修复要读两张表才闭合

本章把前面所有章节的抽象结论落到具体公告行上。样本取自本文核对的 17 个月份页面，选取标准是：**同一个 CVE 同时出现在带完整字段的分组表（`Framework` / `System` / `Android Runtime`）与 `Google Play system updates`（Project Mainline）表里**。这样的行本文核到 13 组。

### 十九之一 读数规则

- **"完整表行原文"逐字抄自公告**，字段顺序为：AOSP 引用号、类型、严重度、受影响版本；方括号内是公告给出的补丁链接编号，本文不展开。
- **"Mainline 表原文"只有两列**（Subcomponent 与 CVE），因此该列里出现的一切映射词都是**本文的推断**，标注规则见第六章与第廿章。
- **映射类型**：`A` 后缀直读、`B` 空格/大小写变形、`C` 语义换名（需人工词典）、`D` 一词双候选（本文拒绝判定）。

### 十九之二 十三组读数

| CVE | 公告页 | 完整表（表名 / 行原文） | Mainline 表原文 | 映射 | 模块名（推断） |
| --- | --- | --- | --- | --- | --- |
| CVE-2024-49723 | [2025-02-01](https://source.android.com/docs/security/bulletin/2025-02-01) | System / `A-357870429 [2]` · ID · High · 15 | `Conscrypt` | A | `com.android.conscrypt` |
| CVE-2024-43093 | [2025-03-01](https://source.android.com/docs/security/bulletin/2025-03-01) | Framework / `A-341680936` · EoP · High · 12, 12L, 13, 14, 15 | `Documents UI` | B | `com.android.documentsui` |
| CVE-2025-48543 | [2025-09-01](https://source.android.com/docs/security/bulletin/2025-09-01) | Android Runtime / `A-421834866` · EoP · High · 13, 14, 15, 16 | `ART` | A | `com.android.art` |
| CVE-2025-48524 | 2025-09-01 | System / `A-399885815` · DoS · High · 13, 14, 15, 16 | `WiFi` | B | `com.android.wifi` |
| CVE-2025-48534 | 2025-09-01 | System / `A-381419370` · DoS · High · 13, 14, 15 | `Cell Broadcast` | B | `com.android.cellbroadcast` |
| CVE-2025-48547 | 2025-09-01 | System / `A-419105158` · EoP · High · 13, 14, 15, 16 | `Permission Controller` | B | `com.android.permissioncontroller` |
| CVE-2025-32323 | 2025-09-01 | System / `A-397216537` · EoP · High · 13, 14, 15, 16 | `Documents UI` | B | `com.android.documentsui` |
| CVE-2025-26456 | [2025-06-01](https://source.android.com/docs/security/bulletin/2025-06-01) | Android Runtime / `A-391895923 [2] [3]` · DoS · High · 14, 15 | `ART` | A | `com.android.art` |
| CVE-2024-40673 | [2024-10-01](https://source.android.com/docs/security/bulletin/2024-10-01) | System / `A-309938635` · RCE · High · 12, 12L, 13, 14 | `ART` | A | `com.android.art` |
| CVE-2024-0018 | [2024-01-01](https://source.android.com/docs/security/bulletin/2024-01-01) | Framework / `A-300476626` · EoP · High · 11, 12, 12L, 13, 14 | `Media Codecs` | **D** | 双候选：`com.android.media` 或 `com.android.media.swcodec` |
| CVE-2023-40100 | [2023-11-01](https://source.android.com/docs/security/bulletin/2023-11-01) | System / `A-278303745` · EoP · High · 11, 12, 12L, 13, 14 | `DNS Resolver` | C | `com.android.resolv`（语义词典） |
| CVE-2023-21114 | 2024-06-01 | System / `A-272106880 [2] [3]` · EoP · High · 13 | `WiFi` | B | `com.android.wifi` |
| CVE-2022-20395 | [2022-09-01](https://source.android.com/docs/security/bulletin/2022-09-01) | System / `A-221855295` · EoP · High · 11, 12, 12L, 13 | `MediaProvider` | A | `com.android.mediaprovider` |

### 十九之三 四条统计事实

1. **完整表行的分布**：`System` 9 条、`Android Runtime` 2 条、`Framework` 2 条。也就是说，**"属于 Mainline"这件事在公告里从来不是由表名表达的**——它只由第二张表表达，而第一张表的表名指向的是另一个分类轴（权限判定的落点层）。
2. **类型分布**：EoP 7 条、DoS 4 条、ID 1 条、RCE 1 条；**严重度 13/13 全为 High**。这两组数并列在一起的意义是：Mainline 表本身不带类型与严重度列，所以**任何"Mainline 侧风险等级"的统计都必须从另一张表借字段**——这就是本文反复说的"一张表不闭合"。
3. **版本轴只存在于完整表行里**：13 条全部携带受影响版本（11 到 16 的各种组合），而 Mainline 表一条都没有。**这条差异正是 M-AT1 的治理成本：公告告诉你"哪些版本受影响"，但不告诉你"哪个模块版本修复了它"。**
4. **在野迹象只出现在完整表所在页的页首注记里**：CVE-2024-43093 与 CVE-2025-48543 两条所在页有同一句提示——

   > "Note: There are indications that the following may be under limited, targeted exploitation." — [2025-03-01](https://source.android.com/docs/security/bulletin/2025-03-01)、[2025-09-01](https://source.android.com/docs/security/bulletin/2025-09-01)

   这条最要紧的信号**与 Mainline 表毫无关联**：它不写在模块行上。做优先级排序时，如果只读 Mainline 表，就会把两条有在野迹象的条目与其余条目同等对待。

### 十九之四 三条逐条评注

1. **CVE-2024-0018（`Media Codecs`，映射类型 D）**：这是本文拒绝判定归属的样本。`Media Codecs` 这个词在清单里有两个候选（`com.android.media` 与 `com.android.media.swcodec`，两者都是 APEX 形态且都在 Android 10 起可用），公告未写包名，模块页也未给出"Media Codecs 指哪一个"的定义句。**治理上的正确处理是把它记为"归属待定"并要求厂商公告或补丁链接来确定，而不是挑一个写进资产表**——挑错候选的后果是把验证工作派给没有对应代码的所有者。
2. **CVE-2023-40100（`DNS Resolver` → `com.android.resolv`，映射类型 C）**：这条是"人工词典不可省"的样本。公告词与包名后缀完全不同源（`resolv` 对 `DNS Resolver`），任何自动化归一化都会失配；但它的语义指向明确，且清单里只有一个候选。**C 类映射必须显式承认词典是本文构造的**（第廿章给出词典全文），否则读者无法判断这条对应关系来自哪里。
3. **CVE-2022-20395（`MediaProvider`）与列名漂移**：这一条所在的 2022-09 页，Mainline 表的列名写作 `Component`，而 2023-05 起的页面写作 `Subcomponent`。**同一含义换了列名，是本文要求映射词典带"生效月份区间"的原因**——不加区间的词典会在跨月批量核对时静默漏读（第廿五章的误区之一）。

### 十九之五 读数之外的事

本表**不包含**任何"这台设备是否已修"的判断，也不包含对补丁内容的推断。它只证明三件事：Mainline 表字段确实少、完整表字段确实多、两套词表确实需要缝合。第十三章的可读面清单负责"设备上能读到什么"，第廿章负责"怎么缝合"，第廿一章负责"合完之后怎么定级"。

---

## 二十、两套词表的差集矩阵，与六面归属矩阵

### 二十之一 三个来源，一张矩阵

差集矩阵的三个输入都是公开可核对的：

1. **公告侧 Subcomponent 词表**：本文从 17 个月份页面的 Mainline 表里读到的全部词（不保证是全集，只保证是实测集合）——`ART`、`AppSearch`、`Cell Broadcast`、`Conscrypt`、`DNS Resolver`、`Documents UI`、`Media Codecs`、`MediaProvider`、`Permission Controller`、`WiFi`，共 **10 个词**。
2. **能力侧模块清单**：第五章抄录的 30 个包名。
3. **设备侧模块名**：第廿二章脚本从 `apex-info-list.xml` 读到的实际集合（每台设备一份）。

矩阵的两个方向含义不同，必须分开记账。

### 二十之二 方向一：清单有、公告未点名（20 个模块）

从 30 个包名里去掉与上述 10 个词能对应的模块，剩下的本文核对月份内**从未**出现在 Mainline 表里。三种解释同时成立，不能只取其一：

| 解释 | 是否可排除 | 处理方式 |
| --- | --- | --- |
| 该模块在这 17 个月里确实没有可登记的修复 | 不可排除 | 中性记录 |
| 该模块的修复以别的口径登记（如 `System` 表但不进 Mainline 表；或厂商公告） | 不可排除 | 交叉核对完整表 |
| 公告的 Subcomponent 词表覆盖不到这类模块（登记口径本身的边界） | **可支持**（第十八章的器件类与 Kernel 表即此理） | 记为"登记口径外" |

**纪律：本方向绝不产出风险结论。**"清单里有、公告里没点名"不等于该模块未修，也不等于它安全。这条纪律与"消失≠已修"是同一条纪律的两个方向。

### 二十之三 方向二：公告点名、清单未必对应

10 个公告词全部能在 30 条清单里找到候选（`Documents UI`、`Media Codecs`、`MediaProvider` 这类是 APK/APEX 混合形态，`WiFi` 与 `Permission Controller` 是 APEX 形态），但**对应关系由本文词典给出，不由公告给出**。词典全文与置信级：

| 公告词 | 模块后缀 | 映射级 | 词典依据 |
| --- | --- | --- | --- |
| `ART` | `art` | A | 后缀直读；模块页与公告词同源 |
| `AppSearch` | `appsearch` | A | 后缀直读 |
| `Conscrypt` | `conscrypt` | A | 后缀直读 |
| `MediaProvider` | `mediaprovider` | A | 后缀直读 |
| `WiFi` | `wifi` | B | 词形变形（`WiFi` 而非 `Wi-Fi`） |
| `Cell Broadcast` | `cellbroadcast` | B | 去空格后直读 |
| `Documents UI` | `documentsui` | B | 去空格后直读 |
| `Permission Controller` | `permissioncontroller` | B | 去空格后直读 |
| `DNS Resolver` | `resolv` | **C** | 语义词典（本文构造，公告未写包名） |
| `Media Codecs` | `media` 或 `media.swcodec` | **D** | 双候选，本文拒绝判定 |

三档处理规则（第廿一章判定模型的输入）：**A/B 级可直接进派工单；C 级必须附"本文词典推断"标注；D 级不得进入派工单，只能进入"待厂商确认"队列。**

### 二十之四 设备侧第三个维度

同一份词典在设备侧还要再过一道：设备上读到的模块名可能是 `com.google.android.*` 前缀（第六章），也可能根本不存在（M-AT9）。因此差集矩阵实际上有**三个集合而不是两个**：公告词集合、能力清单集合、设备实测集合。三者的组合空间里，本文最关心的三格是：

- **公告有 + 清单有 + 设备无** → 该设备不承载此模块（能力/厂商/读数三种缺席之一，第五章的判别表），派工退回资产层；
- **公告有 + 清单有 + 设备有但版本旧** → 真正的处置格，但"旧"只能相对于厂商公告给出的版本号判断，公告本身不提供；
- **公告有 + 清单有 + 设备有且版本等于公告所述** → 仍不能判"已修"，因为 M-AT6（同码异内容）与 M-AT4（未激活）都可能成立。

### 二十之五 六面归属矩阵

把第三章的六个面（M1–M6）与第十七章的十个类型合到一张可派工的矩阵上。行的读法是"**这个类型在这一面上有什么、公开依据是哪条**"：

| 类型 | M1 归属单位 | M2 存在形态 | M3 更新通道 | M4 生命周期 | M5 契约表面 | M6 设备侧可证明性 |
| --- | --- | --- | --- | --- | --- | --- |
| M-AT1 | 可点名（需词典） | APEX 或 APK | Play 或伙伴 OTA | 安装/激活可分读 | 文档承诺 | **部分可**（名字+版本） |
| M-AT2 | 可点名 | 同上 | 同上 | — | **本面即缺陷** | 弱：行为差异 |
| M-AT3 | 名字仍在但通道消失 | 扁平化 / non-updateable | **只剩整机 OTA** | 随镜像 | 不受影响 | `ro.apex.updatable` 可读 |
| M-AT4 | 可点名 | 正常 | 已到 | **核心面** | — | `isActive` + 实例目录 |
| M-AT5 | 可点名 | 正常 | 进行中 | 会话态 | — | **目录 0700，预期不可读** |
| M-AT6 | 可点名 | 正常 | 已到 | 可分读 | — | **版本号不足以判别** |
| M-AT7 | **本面即缺陷** | — | — | — | — | 不影响设备读数 |
| M-AT8 | **本面无登记** | — | — | — | — | 无 |
| M-AT9 | 可点名 | 可选/未提供 | **通道可被关闭** | — | — | 属性前缀可读 |
| M-AT10 | **不存在模块单位** | 分区/器件 | 整机 OTA / 厂商直供 | — | — | 需换整套读数面 |

三点使用说明：

1. **矩阵里"预期不可读"与"未读到"必须区分。** 前者在标准策略设备上出现是正常态（第十三章），后者是采集面问题；把两者合并会让快照的可信度崩塌。
2. **每一格都必须能指回一条公开依据。** 本表里出现的依据（"only activated after reboot."、`neverallow`、`apexd_select_prop`、`session_state.proto`、"Flattened APEXes can be updated only via a regular OTA."）都在前文逐字给出过。矩阵不引入任何新事实。
3. **一列一列地用，不要一行一行地用。** M6 那一列决定验收措辞，M3 那一列决定派工对象，M1 那一列决定能否建台账。把它们混成一个"状态"字段，是本章反对的做法。

---
## 廿一、四级判定模型、台账结论与十项运营指标

前二十章把归属拆成六个面（M1–M6）、十个类型（M-AT1…M-AT10）和三个词表集合。本章把它们换成**可执行的判级规则**：一个读数在设备侧最多能被推到哪一档，哪一档必须靠外部证据补，哪一档永远补不上。

### 廿一之一 四级

沿用本系列的四级，但**本报告里 L3-HIGH 的含义要按本节定义读**：它不表示"这台设备有漏洞"，而表示"**这条归属结论在本机上不成立，或这条通道在本机上不存在——继续按原口径派工，产出的是错误工单**"。

| 等级 | 定义（本报告口径） | 处置要求 | 典型码位（脚本 1 / 脚本 2） |
| --- | --- | --- | --- |
| **L3-HIGH** | 设备侧证据**直接表明**模块口径在本机上塌陷、或快照自身完整性不成立；结论不依赖推断 | **立即处置**：停止使用该读数做清点，改走整机 OTA 口径；修正台账的归属单位后重采 | `apex_updatable_false`、`apex_list_parse_mismatch`、`pair_updatable_true_to_false`、`pair_apex_active_flip_off` |
| **L2-WARN** | 读数成立，但**强度不足以定论**，或指向需要追问的状态分叉（装了未激活、会话进行中、策略被放宽、集合发生迁移） | **限期核查**：按 M4 生命周期面与 M3 通道面各查一次，写明依据来源 | `apex_installed_vs_active_gap`、`apex_dupe_shadow_same_version`、`apexd_status_readable_enforcing`、`apex_name_prefix_unmapped`、`pair_apex_removed`、`pair_spl_shift_no_module_change`、`pair_partial_batch_pattern` |
| **L1-INFO** | 中性事实或改善方向，**不单独构成结论**，作用是把快照变成可比对基线 | **归档**：作为下一次差分的基线，不产工单 | `apex_list_read`、`apex_updatable_true`、`apex_vendor_select_absent`、`apex_name_prefix_mix`、`apex_dir_unread_expected`、`staged_session_unread_expected`、`pair_apex_version_up`、`pair_version_batch_shift`、`pair_no_change_within_readable` |
| **L0-LOW** | **缺口**：本应读到的读数没读到、或该形态在本口径下本就不可判定 | **补齐采集或转外部证据流**，绝不允许记作通过 | `apex_list_unread`、`apex_catalog_absent_list`、`release_unread`、`supp_patch_unread`、`apex_vendor_select_unread`、`pair_list_side_unread` |

**L1 与 L0 的分界是本报告特意做的一个选择，需要说明。** 四个"预期不可读"面（`apexd.status`、`/apex` 目录、`cmd apex` / `dumpsys apexservice`、`/data/misc/apexdata` 与会话目录）在脚本里全部落 **L1-INFO** 而不是 L0，因为读到"它被策略挡住"这一事实**本身就是对该面策略读数的一次确认**，它不欠任何数据；只有"标准策略承诺可读却没读到"（如 `apex_list_unread`、`release_unread`）才是缺口，才落 L0。K6 度量的正是这条分界有没有被正确执行——把预期失败上报成风险，或把真缺口写成"策略正常所以通过"，两个方向都算 K6 不达标。

四条实现约定，它们是这套码位能被第三方复算的前提：

1. **一码一判据。** 同一现象不因来源不同而改用不同码；反过来，`apex_updatable_true` 与 `apex_vendor_select_value_off` **允许在同一份快照里并存**——一条讲平台侧通道档位，一条讲厂商侧意图痕迹，它们说的是 M3 的两个不同侧面。
2. **读不到 ≠ 0，也不 ≠ "没有"。** 缺失一律显式写 `unread` 并出 L0 行。特别是两个方向不能混：**"模块缺席"**（清单里有、设备上没有）走 `apex_catalog_absent_list`，**"读数面失败"**（ XML 读不出来）走 `apex_list_unread`。第五章已经写过：清单比对得出的"缺失"必须标为未知，而不是风险。
3. **不跨码位合成总分。** 本文没有"模块健康分"。任何把等级映射成 0–100 的做法都会自动违反第二十章的"不跨设备比较"。
4. **每个码位都必须被触发过。** 第廿四章给出验收流程：脚本声明的每个码，在合成设备档案全集上至少真实触发一次；**从不触发的分支算报告的缺陷，不算样例的遗漏。**

### 廿一之二 从读数到台账结论：五档结论与各自的合法证据组合

四级判定描述的是**一条读数**，而运维台账上写的是**一个结论**。两者之间需要一张映射表，否则会出现"快照一片 L1，台账写已修"这种最典型的失效。本报告的结论集只有五个：

| 台账结论 | 成立所需证据 | 明确不足的组合 |
| --- | --- | --- |
| **已修（模块口径）** | 设备侧该模块的版本读数（`APEX` 行或 `apex-info-list.xml` 的 `versionCode`）**加上**厂商公告 / 补充声明中同一模块的目标版本或 CVE 列表（第十二章之四的 `supplemental_security_patches.xml` 机制） | 只有 SPL 前进；只有公告存在；只有模块名在设备上可见 |
| **待验证** | 公告侧有对应条目 + 词典映射到 A/B 级后缀 + 设备侧读数存在，但两侧的版本号无法对上 | 这一档是**公告证据能单独到达的最高档**（见下方第二条） |
| **不适用** | 设备不承载该模块，且能落到第五章三种缺席情形之一（能力/厂商/读数），并有对应码位 | 仅仅"这台设备没读到"——那是 L0，不是不适用 |
| **归属未定** | 映射级为 C 或 D（`MAP_INFERRED` / `MAP_AMBIGUOUS`），或该条目根本没有模块载体（M-AT10，第十八章） | 把 C 级当作 A/B 级派工；把 D 级的双候选任选一个填进台账 |
| **无法判定** | 主读数面失败（`apex_list_unread`），或采集权限面与基线不同（K6 未达标） | 用相邻设备的快照替代；用出厂镜像解包结果替代在线读数 |

三条必须写进流程的规则：

1. **`MAP_AMBIGUOUS`（D 级）单独降一档。** 第二十章给出 `Media Codecs` 的两个候选后缀（`media` 与 `media.swcodec`），本文拒绝判定。凡映射级为 D 的条目，其结论上限不是"待验证"而是"**归属未定**"——因为"待验证"隐含"这台设备上有且只有一个对应对象"，而这正是 D 级不成立的前提。
2. **公告证据只支持"待验证"，不支持"已修"。** 第十六章已经给出理由：Mainline 表只有 `Subcomponent` 与 `CVE` 两列，不写机型、不写版本号、也不写"哪个模块版本包含此修复"。因此"公告里有这条 CVE"与"这台设备上的这个模块已含修复"之间缺的正是版本坐标。能补上这个坐标的只有厂商自己的公告文字或补充声明文件（第十二章之四），不是 Android 安全公告本身。
3. **同版本号是弱证据（落在 L2-WARN，不上 L1）。** 第八章逐字引过设备侧规则："the APEX can be updated by providing an APEX package with the same package name and **a greater than or equal to** version code." 相等也被允许，意味着两台报告同一 `versionCode` 的设备在"包含哪些修复"上并不等价——除非版本号的分配方保证每次内容变化都推进版本，而这个保证不在任何公开文档里。所以脚本 2 对"两侧版本号完全相同"**不出改善类码位**，只出 `pair_no_change_within_readable`，并在文字里限定它只覆盖可读项。

### 廿一之三 输出契约

两个脚本共享同一套行前缀，脚本 2 的输入**只能是脚本 1 的输出文本**——它不接触设备、不访问网络、不含任何探针。这一条使得基线可以异地比对、可以进代码评审，也是"消失≠已修"这类纪律能被第三方复查的唯一实现方式。

**脚本 1（`mainline-module-posture-snapshot.sh`）逐行前缀：**

| 前缀 | 形态 | 说明 |
| --- | --- | --- |
| `META` | `META,<k>,<v>` | 身份与采集量；缺值一律写 `none`，绝不省略键 |
| `PROP` | `PROP,<属性全名>,<值\|unread>` | 逐个属性一行，含预期读不到的 `apexd.status` |
| `VSEL` | `VSEL,<键>,<值>` | 厂商选择属性族（`ro.boot.vendor.apex.` / `persist.vendor.apex.`）逐键 |
| `APEX` | `APEX,<name>,<versionCode>,<versionName>,<isFactory>,<isActive>,<partition>,<provideSharedApexLibs>,<lastUpdateMillis>` | **主读数面**：XML 每条一行，九个逗号段齐全，缺字段写 `none` |
| `PATH` | `PATH,<name>,<modulePath>,<preinstalledModulePath>` | 只在两个路径都存在且不相等时输出，它是"安装态与激活态分家"的直接读数 |
| `CATALOG` | `CATALOG,<后缀>,present\|absent` | 与第五章能力清单逐后缀比对，固定 30 行，后缀而不是包名 |
| `DIR` | `DIR,<面>,<条目数\|unread>,<摘要>` | 五个面：`apex_root`、`vendor_apex`、`apexdata`、`staged_sessions`、`supp_patch` |
| `SVC` | `SVC,<cmd_apex\|dumpsys_apexservice>,<ok\|denied_selinux\|denied_caller\|not_found\|unread>,<原文首行>` | 服务面分流，第 4 段保留原始错误文本 |
| `COUNT` | `COUNT,<k>,<v>` | 分组计数（`xml_bytes`、`apex_total`、`apex_active`、`apex_shadow_names`、`catalog_present` 等），供 K1/K4 与 `apex_list_parse_mismatch` 使用 |
| `ROW` | `ROW,<lv>,<code>,<text>` | 判定行 |
| `SUMMARY` | `SUMMARY,…` | 四级计数 + 主读数面与关键档位 |

`SELinux` 档位与 `ro.build.type` 不进 `ROW`，它们作为 `META,selinux` 与 `META,build_type` 存在：它们是**解释其它行的坐标**，不是被解释的对象。

**脚本 2（`mainline-module-posture-drift.sh`）逐行前缀：** `PAIR`（同一键在两侧的取值并排）、`NOTE`（可比性声明与同名记录说明）、`DELTA`（逐模块逐字段差异事实）、`NEW`/`GONE`（单侧出现/消失）、`ROW`、`SUMMARY`。

契约里最要紧的两条：

- **字段缺失写成显式 `none`，不省略列。** 否则差分脚本会把"这一侧没有 partition 属性"读成"两侧不同"。第二十章样例已证明 schema 会随世代变（`ApexInfoList.xsd:41-47` 的六个属性里，`partition` 是较新的一个）。
- **`ROW` 的第 4 段是给人读的句子，不参与比对。** 机器可比对的只有前三种前缀。这条让样例输出可以随措辞修订而不改变验收结论。

### 廿一之四 十项运营指标：把六面归属变成可跟踪的量

| 指标 | 定义 | 数据来源 | 目标 | 常见误用 |
| --- | --- | --- | --- | --- |
| **K1** | 主读数面可得率 | 出现 `COUNT,apex_total`>0 的快照数 ÷ 应采快照数；反向码 `apex_list_unread` | ≥ 98%，且缺口全部进 K8 队列 | 用 `/apex` 目录列出项数替代——本文已证该面对 shell 预期不可读 |
| **K2** | 快照头完备率 | `release`/`sdk`/`security_patch`/`build_type` 四行同时有值的快照比例 | 100%（这是其它指标的分母资格） | 在缺 `release` 的快照上继续算模块集合差 |
| **K3** | 模块名归一率 | 后缀归一（第六章流程）成功、且 `apex_name_prefix_unmapped` 为 0 的设备比例 | ≥ 95%；剩余必须逐台留名 | 用完整包名跨机型做集合比对（GMS 与 AOSP 前缀不重合） |
| **K4** | 安装/激活成对记录率 | 同一快照内同时留存 `isActive` 读数与 `COUNT,apex_shadow_names` 的比例 | 100% | 把包管理器的"已安装"当"已生效"（第十章那条 `only activated after reboot`） |
| **K5** | 通道档位可判率 | `apex_updatable_*` 与 `apex_vendor_select_*` 两类码位都有明确读数的快照比例 | ≥ 90%，读不到必须记 L0 | 把"厂商选择属性存在"读成"通道已关闭"（第十二章之二本文拒绝的推法） |
| **K6** | 预期失败标注正确率 | 四个预期失败面（`apexd.status`、`/apex` 列表、服务面、会话目录）全部按预期码位记录而非笼统"读取失败"的快照比例 | 100% | 把预期不可读上报为风险，或反过来把"读到了"当正常 |
| **K7** | 服务面分流到位率 | 服务面失败落进 `denied_selinux` / `denied_caller` / `not_found` 三码之一的比例（第十三之三的两道门） | 100% | 混记成一条"apex 服务不可用"，导致工单派错方向 |
| **K8** | L0 缺口闭环时长 | 每个 L0 行从出现到被补采或被明确转为外部证据的中位天数 | 中位 ≤ 14 天，无长期挂起 | 缺口静默消失（换采集方式却没登记，等价于漏采） |
| **K9** | 漂移复核闭环率 | 脚本 2 产出的 `pair_apex_removed`、`pair_apex_active_flip_off`、`pair_spl_shift_no_module_change` 三类行中，在期限内写回工单或明确记为可解释的比例 | ≥ 90% | 用"两次快照差异为 0"当作"无需复核"（差异为 0 恰恰可能是主面失败的假象） |
| **K10** | 模块口径盲区占比 | 当月公告条目中落不进 Mainline 表的比例（第十八章的器件类、Kernel 表、厂商自有通道） | **不设目标值**，只设趋势与解释率 | 因为模块口径覆盖不到就放弃模块口径；或反过来把它当 100% 覆盖 |

四条使用说明：

1. **K2 与 K6 是"指标里的指标"。** 它们不描述设备风险，描述的是**这份台账有没有资格参与其它指标的计算**。K2 不达标时，K1/K3/K4 的数字全部作废。
2. **K4 的分子必须是成对。** 只留存 `isActive` 或只留存实例目录的快照不计入——第九章与第十章给过两个独立来源，成对留存才能识别 `apex_installed_vs_active_gap` 与 `apex_dupe_shadow_same_version`。
3. **K10 是唯一一个"数值越高越正常"的指标。** 它衡量的是模块口径的盲区大小，不是工作绩效；把它写成 KPI 会诱导团队把无模块载体的条目硬塞进某个包名，正是第十七章 M-AT10 定义的错误做法。
4. **K8 与 K9 都不允许用"关闭工单"来完成。** 二者的合法终点分别是"补到读数"与"给出可解释依据"；把 L0 直接关掉，等价于把缺口改写成通过，这是本系列在多篇报告里反复指出的同一失效模式。

---
## 廿二、脚本 1：模块姿态快照

本章给出第一个可执行件——`mainline-module-posture-snapshot.sh`。它做一件事：**在设备侧把"模块口径"的六个面（M1–M6）各取一次可读证据，并给每一条证据配上判级。** 它不安装、不卸载、不重启、不降级、不解包任何 `.apex`，也不访问网络；它只读，读得到就读，读不到就把"读不到"本身按预期与否分流记账。

### 廿二之一 这个脚本要证明什么，不证明什么

**它证明的**：这台设备此刻在模块口径下是否可读、读到的集合是什么、哪些面按标准策略应当读不到、以及这次读数有没有资格进入下一次差分。

**它不证明的**（这三条比上面更重要，写在前面是为了防止样例输出被断章取义）：

1. **不证明任何 CVE 已修。** 脚本产出的最强版本证据是 `pair_apex_version_up`（脚本 2），而第廿一章已经限定它的天花板是"待验证"。
2. **不证明模块缺席就是能力缺失。** `apex_catalog_absent_list` 的释义行里三种情形（能力/厂商选择/读数失败）被明确并列，脚本没有信息区分它们。
3. **不证明"预期不可读"的面对别人也不可读。** 那四条授权全部来自 `shell.te` 的枚举；换采集域（root、`adb root`、厂商诊断域）后同一面的含义反转，脚本用 `*_readable_unexpected` 与 `pair_face_availability_change` 记录这种反转，而不是假装它不存在。

### 廿二之二 采集面：八条命令，五个来源

脚本对设备的每一次访问都只可能来自这八条命令，全部经同一个 `raw()` 函数（剥掉 `\r`，Windows 上的 adb 输出常带 CRLF，不剥会让 `case` 与 `grep` 的模式在行尾落空）：

| 命令 | 取什么 | 归属面 | 为什么用它而不是别的 |
| --- | --- | --- | --- |
| `get-state` | 设备可达性 | — | 唯一的前置门；不可达直接退出，不产出半成品快照 |
| `getprop`（一次 bulk） | 全部可见属性 | M1/M2/M3 | **一次 bulk 而不是逐个 `getprop`**：既少 N 次往返，又天然符合"受限属性对 shell 不出现"这一读取语义——逐个查询会把"键不存在"和"键被隐藏"混成同一个空串，bulk 之后本地查表至少保证两侧的解析器相同 |
| `getenforce` | SELinux 档位 | M6 | 整套"预期失败"解释体系的前提，见第廿一章 L1/L0 分界 |
| `cat /apex/apex-info-list.xml` | **主读数面** | M1/M2/M4 | 设备侧唯一同时承载模块名、版本、出厂标记、激活标记与分区的文件（第八章之三） |
| `ls /apex`、`ls /vendor/apex`、`ls /data/misc/apexdata`、`ls /data/app-staging`、`ls /metadata/apex/sessions` | 挂载、厂商包、模块私有数据、会话 | M2/M3/M4 | 交叉面，不是主面：其中三个面对 shell 预期不可读 |
| `cat /etc/security/supplemental_security_patches.xml` | 厂商自愿声明 | M6 的旁证 | 第十二章之四的机制；`cat` 无法区分"不存在"与"无权限"，所以失败一律 L0 |
| `cmd apex getActivePackages`、`dumpsys apexservice` | 服务面 | M6 | 用来检验"两道门"是否被正确分流，而不是用来取数 |
| `ps -A -o NAME` | apexd 进程位 | M4 | `apexd` 是 `oneshot` + `disabled`（`apexd.rc:1-8`），缺席是正常态，脚本必须这样解释它 |

属性名在本地查表前必须过 `esc()`：`ro.apex.updatable` 里的点若不转义，`sed` 会把它当"任意字符"，`roXapexXupdatable` 这样的键也能命中，属于典型的假阳性。这一点在跨会话复用脚本时很容易被忽略，所以直接写进实现而不是留给注释。

### 廿二之三 码位表（脚本 1 全集，83 个）

按采集面分节列出。**"触发档案"列指该码在第廿四章的十四份覆盖用合成设备档案（`mm001`–`mm014`）里由哪个档案真实触发**——这一列是第廿一章第四条实现约定（"每个码位都必须被触发过"）的兑现证据，没有它，这张表就只是注释的目录。

**A. SELinux 档位（3）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `selinux_enforcing` | L1-INFO | `getenforce` 返回 `Enforcing` | mm001 等 12 个档案 |
| `selinux_permissive` | L2-WARN | 返回 `Permissive`：策略只记录不拦截，"读不到即异常"的推论在本机失效 | mm004 |
| `selinux_unread` | L0-LOW | `getenforce` 无输出：后续所有失败只能记"归因未定" | mm014 |

**B. 快照头（13）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `prop_source_unread` | L0-LOW | bulk `getprop` 无输出，属性面全部结论不成立 | mm005 |
| `release_read` / `release_unread` | L1 / L0 | `ro.build.version.release` 有值 / 无值 | mm001 等 12 个档案；`release_unread` mm014 |
| `sdk_read` / `sdk_unread` | L1 / L0 | `ro.build.version.sdk` 有值 / 无值 | mm001 等 11 个档案；`sdk_unread` mm006 mm014 |
| `model_unread` | L0-LOW | 无机型标识：跨设备比对与厂商公告检索失去键 | mm006 mm014 |
| `spl_read` / `spl_absent` | L1 / L0 | 安全补丁级别有值 / 无值（有值也只作时间锚点） | mm001 等 11 个档案；`spl_absent` mm006 mm014 |
| `build_type_user` / `build_type_unread` / `build_type_nonuser` | L1 / L0 / L2 | `user` / 无值 / 其余（`userdebug` 等，预期失败表不适用） | mm001 等 10 个档案；`build_type_unread` mm006 mm014；`build_type_nonuser` mm004 |
| `virtual_device` | L1-INFO | `ro.kernel.qemu` 为 `1`/`true`：模块集合由镜像配置决定 | mm008 |
| `snapshot_header_partial` | L0-LOW | 读到 `release` 但没读到 `sdk`：第五章的缺席判定需要两者同轴 | mm006 |

**C. 设备形态档位与就绪位（7）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_updatable_true` | L1-INFO | `ro.apex.updatable=true`：模块更新可落 `/data/apex/active` 并在重启后激活 | mm001 等 9 个档案 |
| `apex_updatable_false` | **L3-HIGH** | `=false`：本机走内置/扁平化形态，文档写明 `Flattened APEXes can be updated only via a regular OTA.`，模块级快修通道在本机不存在 | mm008 |
| `apex_updatable_unread` | L0-LOW | 无值：该键类型 `exported_default_prop`（`property_contexts:1505`，`exact bool`）标准下可读，"未定义"与"采集问题"在设备侧不可区分 | mm002 mm005 mm014 |
| `apex_updatable_bogus` | L2-WARN | 取值非布尔：厂商改写了属性形态 | mm004 |
| `apex_all_ready_true` | L1-INFO | `apex.all.ready=true`：此刻集合可作下一次差分基线 | mm001 等 10 个档案 |
| `apex_all_ready_false` | L2-WARN | `=false`：采集早于激活完成，集合可能是 bootstrap 阶段的部分集合，不得进 K1 分子 | mm007 |
| `apex_all_ready_unread` | L0-LOW | 无值：其类型为 `system_restricted_prop`，只限制 `set` 不限制读，读不到本身属采集异常 | mm002 mm005 mm014 |

**D. 阶段属性与私有属性（5）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apexd_status_expected_denied` | L1-INFO | `apexd.status` 读不到且 `Enforcing`：与 `system_internal_prop` 的 `neverallow { domain -coredomain } $1:file no_rw_file_perms` 一致，属预期 | mm001 等 11 个档案 |
| `apexd_status_unread_other` | L0-LOW | 读不到但不处于 `Enforcing`：无法归因到策略，按缺口处理 | mm014 |
| `apexd_status_readable_enforcing` | L2-WARN | `Enforcing` 下读到了：本机带厂商追加授权，本报告的预期失败表对它不适用 | mm010 |
| `apexd_status_readable_permissive` | L2-WARN | `Permissive` 下读到：可读性来自放宽而不是机制 | mm004 |
| `apex_private_prop_readable` | L2-WARN | 读到 `apexd_private_prop` 类型的 `ro.apexd.changed_active_apexes`：属性可读性宽于标准，跨设备比较前须先扣除 | mm004 mm010 |

**E. 厂商选择属性族（5）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_vendor_select_absent` | L1-INFO | `ro.boot.vendor.apex.` 与 `persist.vendor.apex.` 前缀下无任何键（两前缀在 `property_contexts:356-357` 为 `apexd_select_prop`，读侧无限制），故"读不到即未定义" | mm001 等 12 个档案 |
| `apex_vendor_select_unread` | L0-LOW | 属性面本身未采到：不判定该族存在与否 | mm005 |
| `apex_vendor_select_value_off` | L2-WARN | 键取值为 `false`/`0`：厂商意图的可读痕迹，但本文未定位到消费方实现，**只登记为需向厂商确认，不断言通道已关闭** | mm004 |
| `apex_vendor_select_value_empty` | L0-LOW | 键存在但值为空：存在性可登记，取值不可用 | mm004 |
| `apex_vendor_select_value_set` | L1-INFO | 键有其它取值：登记为痕迹，不参与判定 | mm004 |

**F. 主读数面记录级判定（6）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_list_unread` | L0-LOW | `/apex/apex-info-list.xml` 未读到。注意 `shell.te:163` 与 `:281` **有两处**明确授予 shell 读 `apex_info_file`，所以失败原因通常不在策略，而在路径不存在或采集通道受限 | mm002 mm005 |
| `apex_list_read` | L1-INFO | 读到，并回报字节数与记录条数 | mm001 等 12 个档案 |
| `apex_list_parse_mismatch` | **L3-HIGH** | 两种情形：非空文件解析出 0 条记录（`ApexInfoList.xsd:21` 的 `minOccurs="1"` 使零条目文件本身就是不完整读数）；或某条记录不含 `moduleName`（`xsd:38` 声明 `use="required"`），该条被丢弃并计数 | mm007 mm012 |
| `apex_name_prefix_unmapped` | L2-WARN | 模块名既不属 `com.android.*` 也不属 `com.google.android.*`：第六章映射覆盖不到 | mm009 |
| `apex_partition_out_of_enum` | L2-WARN | `partition` 取值不在 `xsd:26-33` 五个枚举内（如小写 `system`）：本机 schema 世代与本文依据的 ref 不一致 | mm004 |
| `apex_entry_field_missing` | L2-WARN | `versionCode` 或 `isActive` 为空：两者在 `xsd:41` 与 `xsd:44` 均为 required，缺字段说明记录不完整，**不是**该模块没有版本或没有激活态 | mm004 |

**G. 主读数面集合级判定（10）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_name_prefix_gms` / `apex_name_prefix_aosp` | L1 | 存在 `com.google.android.*` / `com.android.*` 记录 | mm001；`apex_name_prefix_aosp` mm001 等 11 个档案 |
| `apex_name_prefix_mix` | L1-INFO | 两种前缀并存：文档清单本身就同时列有两者，跨机型比对须按后缀名 | mm001 |
| `apex_installed_vs_active_gap` | L2-WARN | 同名多条记录中最高 `versionCode` 未激活：文档 `the newly installed version of the APEX is only activated after reboot.`，属"已安装未激活"（M-AT4） | mm001 |
| `apex_dupe_shadow_same_version` | L2-WARN | 同名多条且激活版本等于最高版本：按 `greater than or equal` 规则相等也合法，故版本号不充当补丁指纹（第二十一条之二第三条规则的落地） | mm008 |
| `apex_dupe_name_over_two` | L2-WARN | 同名记录超过 2 条：出厂/激活/回滚快照并存的公开规则本文未取到，按向厂商追问处理 | mm008 |
| `apex_active_zero` | **L3-HIGH** | 读到清单但无一条 `isActive=true`：M4 面同时不成立 | mm007 |
| `apex_partition_vendor_present` | L1-INFO | 存在 `VENDOR`/`ODM` 分区记录：归属单位不能只写平台侧模块名 | mm004 mm009 |
| `apex_no_shadow_pairs` | L1-INFO | 无任何同名多条记录：本机不呈现"出厂被更新遮蔽"形态 | mm003 等 9 个档案 |
| `apex_shared_libs_present` | L1-INFO | 存在 `provideSharedApexLibs=true`：其版本变化影响面超出自身进程（M5 旁证） | mm001 等 4 个档案 |

**H. 与第五章能力清单的比对（5）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_catalog_not_compared` | L0-LOW | 主读数面失败：缺席与读不到分开记录，不得合并解释 | mm002 mm005 mm012 |
| `apex_catalog_all_present` | L1-INFO | 30 个后缀全部出现：只说明承载形态 | mm011 |
| `apex_catalog_absent_list` | L0-LOW | 列出未出现后缀，并重申三种不可区分情形 | mm001 等 10 个档案 |
| `apex_catalog_extra` | L1-INFO | 本机存在清单之外的后缀：清单不保证是全集，多出项不判异常 | mm003 |
| `apex_catalog_pre_mainline_doc` | L2-WARN | `SDK=29`：清单口径自 Android 11 成型，且文档写明 `Mainline support for Android 11 and lower is concluded as of Q4 of 2025.` | mm013 |

**I. 目录交叉面（11）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `apex_dir_unexpected_readable` | L2-WARN | 能 `ls /apex`：`shell.te` 里没有 `apex_mnt_dir:dir` 授权，读到即策略宽于标准，其余预期失败结论都要降档 | mm003 等 4 个档案 |
| `apex_dir_unread_expected` | L1-INFO | 未能列出：与标准策略一致，不作故障，也不影响主读数面 | mm001 等 10 个档案 |
| `apex_dir_vs_list_mismatch` | L2-WARN | XML 有记录但 `/apex` 下无同名挂载点：优先信 XML，并追问采集时序 | mm003 |
| `apex_dir_list_cross_ok` | L1-INFO | 目录项与 XML 集合一致：两个独立来源互证，M2 读数强度最高 | mm008 mm010 |
| `apex_dir_versioned_mount` | L1-INFO | 目录里出现 `name@version` 条目：对应文档 `Multiple versions of the same APEX can be mounted at the same time.` | mm003 mm008 mm010 |
| `vendor_apex_dir_read` | L1-INFO | 列出 `/vendor/apex`（`shell.te:164-165` 明确授权 `vendor_apex_file`） | mm001 等 6 个档案 |
| `vendor_apex_all_in_list` / `vendor_apex_orphan_file` | L1 / L2 | 包名（去掉前缀与 `.apex` 后）全部/部分在清单中有对应记录；孤儿项属 M-AT9 线索 | mm001 mm010 mm013；`vendor_apex_orphan_file` mm005 mm006 mm009 |
| `vendor_apex_dir_unread` | L0-LOW | 读不到：该面有授权，通常是目录不存在（无厂商自带 APEX），但两者在设备侧不可区分 | mm002 等 8 个档案 |
| `apexdata_dir_unexpected_readable` | L2-WARN | 能列 `/data/misc/apexdata`：无 `apex_module_data_file` 授权；该目录名本身还是第四种命名形态（`file_contexts:680-687` 直接用包名做子目录） | mm005 mm010 |
| `apexdata_dir_unread_expected` | L1-INFO | 未能列出：与标准策略一致 | mm001 等 12 个档案 |

**J. 会话面（2）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `staged_session_readable` | L2-WARN | 读到 `/data/app-staging` 或 `/metadata/apex/sessions`：`init.rc` 分别给 `0751 system system` 与 `0700 root system`，标准 shell 读不到，说明采集发起域不是 shell | mm003 |
| `staged_session_unread_expected` | L1-INFO | 未读到：**不等于**无进行中的会话，M4 的会话面在本快照里是缺口，但缺口不产风险 | mm001 等 13 个档案 |

**K. 服务面分流（10）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `cmd_apex_denied_selinux` | L1-INFO | 输出含 `avc`/`denied`：卡在 `shell.te:378-384` 的 find 排除集，责任在策略与设备类别 | mm001 等 7 个档案 |
| `cmd_apex_denied_caller` | L1-INFO | 输出含 `Only root …`：已过 find 门，卡在 `apexservice.cpp:66` 与 `:176`，责任在采集发起者身份 | mm003 |
| `cmd_apex_service_not_found` | L1-INFO | 报找不到服务：servicemanager 在被拒时对调用方呈现"不存在"，只登记服务面不可用 | mm002 |
| `cmd_apex_readable_unexpected` | L2-WARN | 返回内容：两道门同时可过，说明发起方不是标准 shell 域，须按采集域重采 | mm005 mm010 |
| `cmd_apex_unread` | L0-LOW | 无输出：无法分流，服务面记缺口 | mm006 mm012 mm014 |
| `dumpsys_apexservice_denied_selinux` / `dumpsys_apexservice_denied_caller` / `dumpsys_apexservice_not_found` / `dumpsys_apexservice_readable_unexpected` / `dumpsys_apexservice_unread` | L1×3 / L2 / L0 | 同上分流。**注意** `apexservice.cpp:823` 起的 `shellCommand` 与 `:744` 的 `SHELL_COMMAND_TRANSACTION` 路径自身不做调用者校验，所以"dump 函数没有权限检查"与"shell 能否读到 dump"是两件事 | mm001 等 7 个档案 |

**L–N. 其余（6）**

| 码位 | 等级 | 触发条件 | 触发档案 |
| --- | --- | --- | --- |
| `supp_patch_read` / `supp_patch_unread` | L1 / L0 | 读到 / 未读到 `/etc/security/supplemental_security_patches.xml`；未读到**不是**"厂商未上报" | mm005 mm010；`supp_patch_unread` mm001 等 12 个档案 |
| `apexd_process_present` | L1-INFO | `ps` 中有 `apexd`：`oneshot` 服务出现在进程表说明正处于安装或激活流程 | mm004 等 5 个档案 |
| `apexd_process_absent_normal` | L1-INFO | `ps` 中无 `apexd`：完成激活后正常退出，缺席不判风险 | mm002 等 5 个档案 |
| `ps_unread` | L0-LOW | 未采到进程表：不判定进程位 | mm001 等 4 个档案 |
| `apex_face_pre_apex` | L2-WARN | `SDK<29`：APEX 容器为 Android 10 引入，模块口径整体不适用，归属单位退回分区/整机镜像（M-AT10） | mm002 |

### 廿二之四 三个实现细节，它们决定了输出能不能被复算

1. **XML 解析的切分顺序。** `apex-info-list` 这个根元素名本身就含 `<apex-info` 前缀，所以解析必须**先剥根再切记录**（脚本里是同一条 `sed` 的两个 `-e`）；顺序颠倒会让记录数恒多 1，进而让 `apex_list_parse_mismatch` 在完全正常的文件上误报。同理，属性提取用一个"贪婪前缀 + 属性名 + 等号 + 引号内非引号串"的 `sed`，它取的是**最后一次**出现，这对单记录安全，也是脚本要求"每条记录单独成行"的原因。
2. **`isActive` 与 `versionCode` 的比较不合并。** 同名多条记录的处理是：先按记录取集合，再在该集合内分别求"最高 `versionCode`"与"最高已激活 `versionCode`"。若把两步合成一步（只留最高版本那条），`apex_installed_vs_active_gap` 与 `apex_dupe_shadow_same_version` 将无法区分——前者要求"最高未激活"，后者要求"最高即激活"。
3. **`/apex` 目录比对按完整包名，`/vendor/apex` 按后缀名。** 这两个面的命名对象本来就不同：`/apex` 下是挂载点（就是包名，可能带 `@version`），`/vendor/apex` 下是包文件（`<包名>.apex`）。把它们统一成"后缀名比对"会造出一个假的一致率：`com.google.android.wifi.apex` 与 `com.android.wifi` 在文档口径下是同一个模块的两种签名，统一按后缀才能对齐；而 `/apex/com.android.wifi@350500000` 必须先剥 `@` 才能与 `moduleName="com.android.wifi"` 对上。脚本里对应两处不同实现，不是疏漏。

### 廿二之五 脚本正文

正文如下。它只读取，不写入：全部设备侧动作限于第廿二章之二列出的八类命令，任何一条都不属于安装、卸载、重启、降级或解包。

```bash
#!/usr/bin/env bash
# 脚本 1: mainline-module-posture-snapshot.sh — 模块姿态快照（只读）
# 用法: mainline-module-posture-snapshot.sh <设备序列号>
# 依赖: adb；只调用 get-state / getprop / cat / ls / getenforce / ps / cmd / dumpsys 八类命令，全部只读
# 不写设备、不安装或卸载任何模块、不触发重启、不降级、不访问网络、不解包任何 .apex
set -u
dev="${1:?用法: mainline-module-posture-snapshot.sh <设备序列号>}"
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable,$dev" >&2; exit 1; }

hi=0; warn=0; gap=0; info=0
row() { # row <L3-HIGH|L2-WARN|L1-INFO|L0-LOW> <信号码> <文本>
  local lv="$1" code="$2" txt="$3"
  printf 'ROW,%s,%s,%s\n' "$lv" "$code" "$txt"
  case "$lv" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) warn=$(( warn + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
meta() { printf 'META,%s,%s\n' "$1" "${2:-none}"; }
cnt()  { printf 'COUNT,%s,%s\n' "$1" "$2"; }
raw()  { adb -s "$dev" shell "$@" 2>/dev/null | tr -d '\r'; }

# ---------- 属性面：一次 bulk getprop，本地查表 ----------
# 一次性取全部可见属性，比逐个 getprop 少 N 次往返；也天然符合“受限属性对 shell 不出现”这一读取语义。
propdump="$(raw getprop)"
esc() { printf '%s' "${1//./\\.}"; }        # 属性名里的点必须转义，否则 sed 把它当任意字符
getp() { # getp <属性全名> -> 值（读不到与不存在都返回空串）
  [ -z "$propdump" ] && return
  printf '%s\n' "$propdump" | sed -n "s/^\[$(esc "$1")\]: \[\(.*\)\]$/\1/p" | head -1
}
pref() { # pref <前缀> -> key=value 逐行（按 key 排序）
  [ -z "$propdump" ] && return
  printf '%s\n' "$propdump" | sed -n "s/^\[$(esc "$1")\([^]]*\)\]: \[\(.*\)\]$/\1=\2/p" | sort
}

model="$(getp ro.product.model)"; manuf="$(getp ro.product.vendor.manufacturer)"
board="$(getp ro.product.board)"; rel="$(getp ro.build.version.release)"
sdk="$(getp ro.build.version.sdk)"; spl="$(getp ro.build.version.security_patch)"
btype="$(getp ro.build.type)"; qemu="$(getp ro.kernel.qemu)"
updatable="$(getp ro.apex.updatable)"; allready="$(getp apex.all.ready)"
astatus="$(getp apexd.status)"; changed="$(getp ro.apexd.changed_active_apexes)"
enf="$(raw getenforce | head -1)"

meta device_serial "$dev"; meta model "$model"; meta manufacturer "$manuf"
meta board "$board"; meta release "$rel"; meta sdk "$sdk"
meta security_patch "$spl"; meta build_type "$btype"; meta selinux "$enf"

# ---------- SELinux 模式：整套预期失败解释的前提 ----------
case "$enf" in
  Enforcing) row L1-INFO selinux_enforcing "getenforce=Enforcing：本报告的预期失败表（第十三章）适用于这台设备，读不到多为策略效果而不是采集故障" ;;
  Permissive) row L2-WARN selinux_permissive "getenforce=Permissive：策略只记录不拦截，本快照的“读不到即异常”推论失效，任何可读面都不能与发行设备混算" ;;
  *)          row L0-LOW selinux_unread "未采到 getenforce：无法判定策略档位，后续所有失败都只能记为归因未定" ;;
esac

if [ -z "$propdump" ]; then
  row L0-LOW prop_source_unread "未采到 getprop 输出：属性面的全部结论不成立；读不到不等于这些属性不存在或没有默认值"
else
  cnt prop_lines "$(printf '%s\n' "$propdump" | grep -c '^\[')"
  if [ -n "$rel" ]; then
    row L1-INFO release_read "ro.build.version.release=$rel：Mainline 与 APEX 的适用性都按这一行分档"
  else
    row L0-LOW release_unread "未采到平台版本号：第五章的“清单是否适用”判定与 M-AT3 判定全部降级为缺口"
  fi
  if [ -n "$sdk" ]; then
    row L1-INFO sdk_read "ro.build.version.sdk=$sdk：与模块的 minSdkVersion/maxSdkVersion 同轴比较才有意义"
  else
    row L0-LOW sdk_unread "未采到 SDK 号：无法判断模块的目标平台范围（第八章的 minSdk/maxSdk）"
  fi
  [ -n "$model" ] || row L0-LOW model_unread "未采到机型标识：跨设备比对与厂商公告检索都失去键，本快照只做单机基线"
  if [ -n "$spl" ]; then
    row L1-INFO spl_read "security_patch=$spl：只作时间锚点；SPL 前进不表示任何模块的 versionCode 变过（第十二章之三）"
  else
    row L0-LOW spl_absent "未采到安全补丁级别：本快照不做任何到达性判断"
  fi
  case "$btype" in
    user) row L1-INFO build_type_user "ro.build.type=user：标准发行构建，本快照按标准策略预期解释各面失败" ;;
    '')   row L0-LOW build_type_unread "未采到 ro.build.type：无法把“读不到”归因到策略档位" ;;
    *)    row L2-WARN build_type_nonuser "ro.build.type=$btype：非发行构建，属性与服务面的可读性可能与发行设备不同，不能与本报告的预期失败表混算" ;;
  esac
  case "$qemu" in
    1|true) row L1-INFO virtual_device "ro.kernel.qemu=$qemu：虚拟设备的模块集合由镜像配置决定，清单比对结果不代表任何真实出货形态" ;;
  esac
fi

# ---------- 档位：ro.apex.updatable（决定 M2/M3 是否分叉） ----------
case "$updatable" in
  true)  printf 'PROP,ro.apex.updatable,true\n'
         row L1-INFO apex_updatable_true "ro.apex.updatable=true：设备形态为可更新 APEX，模块更新可以落到 /data/apex/active 并在重启后激活" ;;
  false) printf 'PROP,ro.apex.updatable,false\n'
         row L3-HIGH apex_updatable_false "ro.apex.updatable=false：这台设备上的 APEX 走内置/扁平化形态，公开口径写明 Flattened APEXes can be updated only via a regular OTA.，模块级快修通道在本机上不存在，台账的归属单位必须退回整机 OTA" ;;
  '')    printf 'PROP,ro.apex.updatable,unread\n'
         row L0-LOW apex_updatable_unread "未读到 ro.apex.updatable：该键类型为 exported_default_prop（property_contexts:1505，exact bool），标准策略下可读；读不到可能是该键未定义，也可能是采集面问题，两者在设备侧不可区分" ;;
  *)     printf 'PROP,ro.apex.updatable,%s\n' "$updatable"
         row L2-WARN apex_updatable_bogus "ro.apex.updatable 取值非布尔（$updatable）：property_contexts:1505 声明为 exact bool，取值越界说明厂商改写了属性形态" ;;
esac

# ---------- 就绪位：apex.all.ready（决定本快照的时序有效性） ----------
case "$allready" in
  true)  printf 'PROP,apex.all.ready,true\n'
         row L1-INFO apex_all_ready_true "apex.all.ready=true：所有 APEX 已激活就绪，此刻的模块集合可作为下一次差分的基线" ;;
  false) printf 'PROP,apex.all.ready,false\n'
         row L2-WARN apex_all_ready_false "apex.all.ready=false：采集发生在激活流程完成之前，此刻清单可能仍是 bootstrap 阶段的部分集合，本快照不得进入 K1 达标统计" ;;
  '')    printf 'PROP,apex.all.ready,unread\n'
         row L0-LOW apex_all_ready_unread "未读到 apex.all.ready：其类型为 system_restricted_prop（只限制 property_service set，不限制读），因此读不到本身属采集面异常" ;;
esac

# ---------- 阶段属性：apexd.status（预期读不到） ----------
case "$astatus" in
  '') printf 'PROP,apexd.status,unread\n'
      if [ "$enf" = Enforcing ]; then
        row L1-INFO apexd_status_expected_denied "apexd.status 读不到，且 SELinux 为 Enforcing：与 system_internal_prop 的 neverallow { domain -coredomain } \$1:file no_rw_file_perms 一致，属预期结果，不是故障"
      else
        row L0-LOW apexd_status_unread_other "apexd.status 读不到，但设备不处于 Enforcing（${enf:-none}）：无法把这次失败归因到策略，按缺口处理"
      fi ;;
  *)  printf 'PROP,apexd.status,%s\n' "$astatus"
      if [ "$enf" = Enforcing ]; then
        row L2-WARN apexd_status_readable_enforcing "在 Enforcing 设备上读到了 apexd.status（$astatus）：与 shell 域的 neverallow 不符，说明这台设备带有厂商追加的读取授权或不是标准策略，本快照的预期失败表对它不适用"
      else
        row L2-WARN apexd_status_readable_permissive "在 Permissive 设备读到 apexd.status（$astatus）：非发行策略形态，可读性来自放宽而不是机制"
      fi ;;
esac
case "$changed" in
  '') printf 'PROP,ro.apexd.changed_active_apexes,unread\n' ;;
  *)  printf 'PROP,ro.apexd.changed_active_apexes,%s\n' "$changed"
      row L2-WARN apex_private_prop_readable "在 shell 域读到 apexd_private_prop 类型的属性（ro.apexd.changed_active_apexes）：这台设备的属性可读性宽于标准策略，跨设备比较前必须先扣除这一差异" ;;
esac

# ---------- 厂商选择属性族（只登记痕迹，不推断结论） ----------
vlines="$(printf '%s\n%s\n' "$(pref ro.boot.vendor.apex.)" "$(pref persist.vendor.apex.)" | grep -v '^$' | sort -u)"
vsel_n=0
if [ -z "$vlines" ]; then
  if [ -n "$propdump" ]; then
    row L1-INFO apex_vendor_select_absent "未见 ro.boot.vendor.apex. 与 persist.vendor.apex. 前缀下的任何键：两个前缀在 property_contexts:356-357 声明为 apexd_select_prop（system_public_prop，读侧无限制），读不到即该设备未定义厂商侧选择键"
  else
    row L0-LOW apex_vendor_select_unread "属性面未采到：不判定厂商选择属性族的存在与否"
  fi
else
  while IFS= read -r kv; do
    [ -n "$kv" ] || continue
    k="${kv%%=*}"; v="${kv#*=}"; vsel_n=$(( vsel_n + 1 ))
    printf 'VSEL,%s,%s\n' "$k" "${v:-none}"
    case "$v" in
      false|0|FALSE) row L2-WARN apex_vendor_select_value_off "厂商选择键 $k 取值为 $v：这是厂商侧意图的可读痕迹，本文未定位到消费方实现，因此不断言该模块的更新通道已关闭，只登记为需向厂商确认" ;;
      '')            row L0-LOW apex_vendor_select_value_empty "厂商选择键 $k 存在但值为空：存在性可登记，取值不可用" ;;
      *)             row L1-INFO apex_vendor_select_value_set "厂商选择键 $k 取值为 $v：登记为痕迹，不参与判定" ;;
    esac
  done <<EOF
$vlines
EOF
  cnt vendor_select_keys "$vsel_n"
fi

# ---------- 主读数面：/apex/apex-info-list.xml ----------
XMLF="/apex/apex-info-list.xml"
xml="$(raw cat "$XMLF")"
xbytes="$(printf '%s' "$xml" | wc -c | tr -d ' ')"
xsed() { printf '%s' "$1" | sed -n "s/.*[[:space:]]$2=\"\([^\"]*\)\".*/\1/p" | head -1; }

apex_total=0; act=0; fac=0; gms=0; aosp=0; shared=0; vpar=0
shadow_names=0; samever=0; over_two=0; field_missing=0; enum_bad=0
no_name=0
prefixed=""; names=""; recs=""; tagcount=0
if [ -z "$xml" ]; then
  row L0-LOW apex_list_unread "主读数面 $XMLF 未读到：shell 域对 apex_info_file 有两条明确读授权（shell.te:163 与 :281），因此失败原因通常在路径不存在或采集通道受限；本快照只能记无法判定，不得记无模块"
else
  cnt xml_bytes "$xbytes"
  # 先剥掉 XML 声明与根元素，再按 <apex-info 切记录：根元素名 apex-info-list 含同样的前缀，不剥就会多计一条
  body="$(printf '%s' "$xml" | tr '\n' ' ' | tr -s ' ' \
          | sed -e 's|<?xml[^>]*>||g' -e 's|</*apex-info-list[^>]*>||g')"
  recs="$(printf '%s\n' "$body" | sed -e 's|<apex-info|\n<apex-info|g' | grep '^<apex-info' | sed -e 's|</apex-info>||g')"
  tagcount="$(printf '%s' "$body" | grep -o '<apex-info[[:space:]>/]' | wc -l | tr -d ' ')"
  apex_total="$(printf '%s\n' "$recs" | grep -c '^<apex-info')"
  row L1-INFO apex_list_read "读到 $XMLF（$xbytes 字节，$apex_total 条 apex-info 记录）：本快照的主读数面成立"
  if [ "$apex_total" = "0" ]; then
    row L3-HIGH apex_list_parse_mismatch "XML 非空但解析出 0 条记录（标签计数 $tagcount）：ApexInfoList.xsd:21 声明 minOccurs=1，零条目的文件本身就是不完整读数，任何以它为基础的清点都不可信"
  fi
fi

if [ "$apex_total" != "0" ]; then
  while IFS= read -r rec; do
    [ -n "$rec" ] || continue
    nm="$(xsed "$rec" moduleName)"; vc="$(xsed "$rec" versionCode)"
    vn="$(xsed "$rec" versionName)"; ib="$(xsed "$rec" isFactory)"; ia="$(xsed "$rec" isActive)"
    pt="$(xsed "$rec" partition)"; ps="$(xsed "$rec" provideSharedApexLibs)"
    lu="$(xsed "$rec" lastUpdateMillis)"; mp="$(xsed "$rec" modulePath)"
    pp="$(xsed "$rec" preinstalledModulePath)"
    printf 'APEX,%s,%s,%s,%s,%s,%s,%s,%s\n' "${nm:-none}" "${vc:-none}" "${vn:-none}" "${ib:-none}" "${ia:-none}" "${pt:-none}" "${ps:-none}" "${lu:-none}"
    if [ -z "$nm" ]; then
      no_name=$(( no_name + 1 ))
      row L3-HIGH apex_list_parse_mismatch "清单里有一条记录不含 moduleName（ApexInfoList.xsd:38 声明 use=required）：这条记录无法归属到任何模块，本快照对它的处理是丢弃并计数，不并入任何模块的统计"
      continue
    fi
    names="$names $nm"
    case "$nm" in
      com.google.android.*) gms=$(( gms + 1 )); prefixed="$prefixed ${nm#com.google.android.}" ;;
      com.android.*)        aosp=$(( aosp + 1 )); prefixed="$prefixed ${nm#com.android.}" ;;
      *) row L2-WARN apex_name_prefix_unmapped "模块 $nm 既不属于 com.android.* 也不属于 com.google.android.*：第六章的映射规则覆盖不到它，它属厂商自带或第三方通道，必须先建词典再进台账" ;;
    esac
    [ "$ia" = true ] && act=$(( act + 1 ))
    [ "$ib" = true ] && fac=$(( fac + 1 ))
    [ "$ps" = true ] && shared=$(( shared + 1 ))
    case "$pt" in
      VENDOR|ODM) vpar=$(( vpar + 1 )) ;;
      SYSTEM|SYSTEM_EXT|PRODUCT) : ;;
      *) enum_bad=$(( enum_bad + 1 ))
         row L2-WARN apex_partition_out_of_enum "${nm} 的 partition 取值（${pt:-none}）不在 xsd:26-33 的五个枚举之内：主读数面的 schema 世代与本文依据的 ref 不一致，本条不参与分区统计" ;;
    esac
    if [ -z "$vc" ] || [ -z "$ia" ]; then
      field_missing=$(( field_missing + 1 ))
      row L2-WARN apex_entry_field_missing "${nm} 的 versionCode 或 isActive 为空：schema（xsd:41 与 :44）把两者都标为 required，缺字段说明记录不完整，而不是该模块没有版本或没有激活态"
    fi
    if [ -n "$mp" ] && [ -n "$pp" ] && [ "$mp" != "$pp" ]; then
      printf 'PATH,%s,%s,%s\n' "$nm" "$mp" "$pp"
    fi
  done <<EOF
$recs
EOF
  cnt apex_total "$apex_total"; cnt apex_active "$act"; cnt apex_factory "$fac"
  cnt apex_shared_libs "$shared"; cnt apex_partition_vendor "$vpar"
  cnt apex_field_missing "$field_missing"; cnt apex_partition_enum_bad "$enum_bad"
  cnt apex_record_no_name "$no_name"
  [ "$gms" != 0 ]  && row L1-INFO apex_name_prefix_gms "$gms 条记录使用 com.google.android.* 前缀：与文档口径一致（GMS 设备由 Google 签名）"
  [ "$aosp" != 0 ] && row L1-INFO apex_name_prefix_aosp "$aosp 条记录使用 com.android.* 前缀：同一设备上两种前缀的分布由模块归属而不是设备类别单轴决定"
  if [ "$gms" != 0 ] && [ "$aosp" != 0 ]; then
    row L1-INFO apex_name_prefix_mix "两种前缀同时存在（$gms / $aosp）：第五章清单本身就同时列有 com.google.android.* 与 com.android.*，属正常形态；跨机型比对必须按后缀名而不是完整包名"
  fi
  # 同名多条记录之间的激活关系（M-AT4 的核心读数）；names 可能重复，先去重避免同一模块出两行
  for nm in $(printf '%s\n' $names | sort -u); do
    sub="$(printf '%s\n' "$recs" | grep -F "moduleName=\"$nm\"")"
    nrec="$(printf '%s\n' "$sub" | grep -c '^<apex-info')"
    [ "$nrec" -gt 1 ] || continue
    shadow_names=$(( shadow_names + 1 ))
    cmax="$(printf '%s\n' "$sub" | while IFS= read -r r; do v="$(xsed "$r" versionCode)"; [ -n "$v" ] && printf '%s\n' "$v"; done | sort -n | tail -1)"
    afv="$(printf '%s\n' "$sub" | grep 'isActive="true"' | while IFS= read -r r; do v="$(xsed "$r" versionCode)"; [ -n "$v" ] && printf '%s\n' "$v"; done | sort -n | tail -1)"
    if [ -n "$cmax" ] && [ -n "$afv" ] && [ "$cmax" != "$afv" ]; then
      row L2-WARN apex_installed_vs_active_gap "模块 $nm：同名存在 $nrec 条记录，最高 versionCode=$cmax 未激活，激活的是 $afv。文档口径为 the newly installed version of the APEX is only activated after reboot.，因此这是“已安装未激活”（M-AT4），不能记作已生效"
    else
      samever=$(( samever + 1 ))
      row L2-WARN apex_dupe_shadow_same_version "模块 $nm：同名存在 $nrec 条记录且激活版本等于最高版本。按八之一引用的“greater than or equal”规则，版本号相等也允许是一次合法安装，因此版本号不充当补丁指纹"
    fi
    if [ "$nrec" -gt 2 ]; then
      over_two=$(( over_two + 1 ))
      row L2-WARN apex_dupe_name_over_two "模块 $nm：同名记录超过 2 条（$nrec）。出厂、激活与回滚快照并存的公开规则本文未取到，这一格按向厂商追问处理"
    fi
  done
  cnt apex_shadow_names "$shadow_names"; cnt apex_same_version_shadow "$samever"
  [ "$act" = 0 ] && row L3-HIGH apex_active_zero "读到清单但没有任何 isActive=true 的记录：主读数面与 M4 面同时不成立，这台设备的模块姿态无法由本快照描述"
  [ "$vpar" != 0 ] && row L1-INFO apex_partition_vendor_present "$vpar 条记录的 partition 为 VENDOR/ODM：厂商分区模块存在，归属单位不能只写平台侧模块名（第六章映射 D 类的一个来源）"
  [ "$over_two" = 0 ] && [ "$shadow_names" = 0 ] && row L1-INFO apex_no_shadow_pairs "清单内没有任何同名多条的记录：这台设备当前不呈现“出厂版本被更新版本遮蔽”的形态，M4 面只有一态可读"
  [ "$shared" != 0 ] && row L1-INFO apex_shared_libs_present "$shared 条记录带 provideSharedApexLibs=true：这些模块向其它模块提供共享库，其版本变化的影响面超出自身进程（第十五章的 M5 旁证）"
fi

# ---------- 清单比对：第五章能力清单（按后缀名匹配，不按完整包名） ----------
DOC="adbd adservices appsearch art bt cellbroadcast configinfrastructure conscrypt documentsui ext.services federatedcompute healthfitness ipsec media media.swcodec mediaprovider modulemetadata networkstack neuralnetworks ondevicepersonalization permissioncontroller rkpd resolv scheduling sdkext statsd tethering tzdata uwb wifi"
absent=""; pres=0
for s in $DOC; do
  hit=0
  for n in $prefixed; do [ "$n" = "$s" ] && hit=1; done
  if [ "$hit" = 1 ]; then pres=$(( pres + 1 )); printf 'CATALOG,%s,present\n' "$s"
  else absent="$absent $s"; printf 'CATALOG,%s,absent\n' "$s"; fi
done
cnt catalog_total 30; cnt catalog_present "$pres"
if [ "$apex_total" = "0" ]; then
  row L0-LOW apex_catalog_not_compared "主读数面失败，本次不做清单比对：缺席集合与读不到在两行里分开记录，不得合并解释"
elif [ "$pres" = 30 ]; then
  row L1-INFO apex_catalog_all_present "第五章清单的 30 个后缀在本机全部出现：这只说明承载形态，不说明任何版本或到达性"
else
  row L0-LOW apex_catalog_absent_list "清单内未在本机读到的后缀：${absent# }。按第五章纪律，缺席有三种在设备侧不可区分的情形（能力/厂商/读数），这一行不产出风险结论"
fi
extra=""; xcnt=0
for n in $prefixed; do
  e=1; for s in $DOC; do [ "$n" = "$s" ] && e=0; done
  if [ "$e" = 1 ]; then extra="$extra $n"; xcnt=$(( xcnt + 1 )); fi
done
if [ -n "$extra" ]; then
  cnt catalog_extra "$xcnt"
  row L1-INFO apex_catalog_extra "本机存在清单之外的后缀：${extra# }。清单不保证是全集（第十八章的可选模块与厂商自带模块都落在这一格），多出项不判异常"
fi

# ---------- 目录交叉面 ----------
in_list() { # in_list <后缀名> <空格分隔列表>
  local t; for t in $2; do [ "$t" = "$1" ] && return 0; done; return 1
}
apexls="$(raw ls /apex)"
if [ -n "$apexls" ]; then
  apexlsc="$(printf '%s\n' "$apexls" | grep -c '[^[:space:]]')"
  printf 'DIR,apex_root,%s,%s\n' "$apexlsc" "$(printf '%s\n' "$apexls" | sort | tr '\n' ' ')"
  row L2-WARN apex_dir_unexpected_readable "列出了 /apex（$apexlsc 项）：android17-release 的 shell.te 里没有 apex_mnt_dir:dir 授权，能列出说明这台设备的策略宽于标准；本快照其余“预期失败”的结论都要跟着降档"
  if [ "$apex_total" != "0" ]; then
    inst=""
    # /apex 下的挂载点名就是完整包名（可能带 @version），比对前统一剥掉 @ 后缀并按空格切开
    ls_norm="$(printf '%s\n' "$apexls" | sed -e 's/@[^[:space:]]*//g' | tr '\n' ' ')"
    for n in $(printf '%s\n' $names | sort -u); do
      case " $ls_norm " in *" $n "*) : ;; *) inst="$inst $n" ;; esac
    done
    if [ -n "$inst" ]; then
      row L2-WARN apex_dir_vs_list_mismatch "XML 里有记录但 /apex 下未见同名挂载点：${inst# }。两个面本应互为交叉验证，不一致时优先信 XML（它有明确的 shell 授权），并追问采集时序"
    else
      row L1-INFO apex_dir_list_cross_ok "/apex 目录项与 XML 主读数面的模块集合一致：两个独立来源相互印证，本快照的 M2 形态读数强度最高"
    fi
  fi
  [ "$(printf '%s\n' "$apexls" | grep -c '@')" != "0" ] && row L1-INFO apex_dir_versioned_mount "目录里存在 name@version 形态条目：与文档 Multiple versions of the same APEX can be mounted at the same time. 一致，是版本并存而不是覆盖的直接证据"
else
  printf 'DIR,apex_root,unread,\n'
  row L1-INFO apex_dir_unread_expected "未列出 /apex：shell.te 未给 apex_mnt_dir:dir，标准策略下这是预期结果，不作为故障，也不影响主读数面"
fi

vsapex="$(raw ls /vendor/apex)"
if [ -n "$vsapex" ]; then
  vsapexc="$(printf '%s\n' "$vsapex" | grep -c '[^[:space:]]')"
  printf 'DIR,vendor_apex,%s,%s\n' "$vsapexc" "$(printf '%s\n' "$vsapex" | sort | tr '\n' ' ')"
  row L1-INFO vendor_apex_dir_read "列出 /vendor/apex（$vsapexc 项）：shell.te:164-165 明确授权 vendor_apex_file，这是厂商自带包的直接读数面，与平台包分开记账"
  orp=""; orc=0
  for f in $vsapex; do
    case "$f" in
      # 清单比对统一按后缀名：/vendor/apex 下是完整包名 + .apex，两侧口径要先对齐才能比
      *.apex) b="${f%.apex}"; sfx="${b#com.android.}"; sfx="${sfx#com.google.android.}"
              in_list "$sfx" "$prefixed" || { orp="$orp $b"; orc=$(( orc + 1 )); } ;;
    esac
  done
  if [ -n "$orp" ]; then
    cnt vendor_apex_orphan "$orc"
    row L2-WARN vendor_apex_orphan_file "/vendor/apex 下有包但清单里无对应记录：${orp# }。可能是未被激活的厂商包，也可能是清单世代差异，属 M-AT9 的线索而不是结论"
  else
    row L1-INFO vendor_apex_all_in_list "/vendor/apex 下的包全部在清单中有对应记录：厂商自带包与主读数面自洽"
  fi
else
  printf 'DIR,vendor_apex,unread,\n'
  row L0-LOW vendor_apex_dir_unread "未列出 /vendor/apex：该面有明确 SELinux 授权，读不到通常是设备不存在该目录（无厂商自带 APEX），但两者在设备侧不可区分，记缺口"
fi

apxdata="$(raw ls /data/misc/apexdata)"
if [ -n "$apxdata" ]; then
  printf 'DIR,apexdata,%s,%s\n' "$(printf '%s\n' "$apxdata" | grep -c '[^[:space:]]')" "$(printf '%s\n' "$apxdata" | sort | tr '\n' ' ')"
  row L2-WARN apexdata_dir_unexpected_readable "列出了 /data/misc/apexdata：shell.te 没有 apex_module_data_file 授权，能列出说明策略宽于标准。这个目录名本身还是第四种命名形态（file_contexts:680-687 直接用模块包名做子目录）"
else
  printf 'DIR,apexdata,unread,\n'
  row L1-INFO apexdata_dir_unread_expected "未列出 /data/misc/apexdata：与标准策略一致，不作为故障"
fi

# ---------- 会话面（预期不可读） ----------
staged="$(raw ls /data/app-staging)"; mstage="$(raw ls /metadata/apex/sessions)"
if [ -n "$staged" ] || [ -n "$mstage" ]; then
  sc="$(printf '%s\n' "$staged" | grep -c '[^[:space:]]')"; mc="$(printf '%s\n' "$mstage" | grep -c '[^[:space:]]')"
  printf 'DIR,staged_sessions,%s,%s\n' "$(( sc + mc ))" "$(printf '%s\n%s\n' "$staged" "$mstage" | grep -v '^$' | sort | tr '\n' ' ')"
  row L2-WARN staged_session_readable "读到了 staged 会话目录内容：init.rc 把 /data/app-staging 设为 0751 system system、/metadata/apex/sessions 设为 0700 root system，标准 shell 不应读到；说明采集发起域不是 shell，本快照的会话读数必须注明采集域"
else
  printf 'DIR,staged_sessions,unread,\n'
  row L1-INFO staged_session_unread_expected "未读到 staged 会话目录：与 init.rc:622-625（0700 root system）与 :743-751（0751 system system）的权限位一致。未读到不等于无进行中的会话，M4 的会话面在本快照里是缺口"
fi

# ---------- 服务面：两道门，失败必须分流 ----------
svc_class() { # svc_class <输出文本> -> ok | denied_selinux | denied_caller | not_found | unread
  local t="$1"
  [ -z "$t" ] && { echo unread; return; }
  case "$t" in
    *ACTIVE*PACKAGES:*|*SESSIONS:*|*Module:*) echo ok; return ;;
  esac
  case "$t" in
    *avc:*|*SELinux*|*denied*) echo denied_selinux; return ;;
    *"Only root"*|*"not allowed"*|*"Permission denied"*|*SecurityException*) echo denied_caller; return ;;
    *"Cannot find service"*|*"Failed to find"*|*"Can't find"*|*"not found"*) echo not_found; return ;;
  esac
  echo unread
}
cmdout="$(raw cmd apex getActivePackages)"
dclass="$(svc_class "$cmdout")"
cmdfirst="$(printf '%s\n' "$cmdout" | grep -v '^[[:space:]]*$' | head -1)"
printf 'SVC,cmd_apex,%s,%s\n' "$dclass" "${cmdfirst:-none}"
case "$dclass" in
  ok) row L2-WARN cmd_apex_readable_unexpected "cmd apex getActivePackages 返回了内容：这需要先过 shell.te:384 的 service find 排除，再过 apexservice.cpp:66 与 :176 的 CheckCallerSystemOrRoot，两道门同时可过说明采集发起方不是标准 shell 域；这份快照混用了两个权限面，需按采集域重采" ;;
  denied_selinux) row L1-INFO cmd_apex_denied_selinux "cmd apex 被 SELinux 拒绝（输出含 avc/denied）：与 shell.te:378-384 把 apex_service 列入排除集一致，责任在策略与设备类别，不在服务实现" ;;
  denied_caller)  row L1-INFO cmd_apex_denied_caller "cmd apex 被服务实现拒绝（输出含 Only root / not allowed）：这一条已过 find 门，卡在 apexservice.cpp:66 与 :176，责任在采集发起者身份而不是策略" ;;
  not_found)      row L1-INFO cmd_apex_service_not_found "cmd apex 报找不到服务：与 find 门的观察一致（servicemanager 在被拒时对调用方呈现为不存在），但本文不把它当策略结论，只登记为服务面不可用" ;;
  unread)         row L0-LOW cmd_apex_unread "cmd apex 无输出：无法把这次失败分流到 find 门还是调用者门，服务面记缺口。两种失败的责任方不同，混记会让工单派错方向（第十三之三）" ;;
esac
dout="$(raw dumpsys apexservice)"
pclass="$(svc_class "$dout")"
pfirst="$(printf '%s\n' "$dout" | grep -v '^[[:space:]]*$' | head -1)"
printf 'SVC,dumpsys_apexservice,%s,%s\n' "$pclass" "${pfirst:-none}"
case "$pclass" in
  ok)   row L2-WARN dumpsys_apexservice_readable_unexpected "dumpsys apexservice 返回了内容：dump 实现（apexservice.cpp:775 起）确实不做调用者校验，但 shell 应在 find 门就被挡；读到即说明策略宽于标准，须在台账注明" ;;
  denied_selinux) row L1-INFO dumpsys_apexservice_denied_selinux "dumpsys apexservice 被 SELinux 拒绝：与 shell 域被排除在 apex_service 之外一致。dump 函数没有权限检查与 shell 能读到 dump 是两回事" ;;
  denied_caller)  row L1-INFO dumpsys_apexservice_denied_caller "dumpsys apexservice 被调用者校验拒绝：已过 find 门，责任在身份而不是策略" ;;
  not_found)      row L1-INFO dumpsys_apexservice_not_found "dumpsys 报找不到该服务：登记为服务面不可用，具体卡在哪道门需 avc 日志佐证（本文脚本不读日志）" ;;
  unread)         row L0-LOW dumpsys_apexservice_unread "dumpsys apexservice 无输出：服务面失败原因无法分流，记缺口" ;;
esac

# ---------- 补充声明文件（厂商自愿机制） ----------
supp="$(raw cat /etc/security/supplemental_security_patches.xml)"
if [ -n "$supp" ]; then
  cnt supp_bytes "$(printf '%s' "$supp" | wc -c | tr -d ' ')"
  cnt supp_cve "$(printf '%s' "$supp" | grep -o 'cve' | wc -l | tr -d ' ')"
  row L1-INFO supp_patch_read "读到 supplemental_security_patches.xml：这是第十二章之四的厂商自愿上报机制，平台聚合后暴露给 API；本文只登记存在与条数，不解析其中 CVE 与模块的对应关系"
else
  printf 'DIR,supp_patch,unread,\n'
  row L0-LOW supp_patch_unread "未读到 /etc/security/supplemental_security_patches.xml：机制存在但写入与否由厂商决定，且 cat 无法区分文件不存在与无读权限。这一行是缺口，不是厂商未上报"
fi

# ---------- apexd 进程位（oneshot，非常驻） ----------
psout="$(raw ps -A -o NAME)"
if [ -n "$psout" ]; then
  if printf '%s\n' "$psout" | grep -qx apexd; then
    row L1-INFO apexd_process_present "ps 中有 apexd：apexd 是 oneshot + disabled（apexd.rc:1-8），出现在进程表说明此刻正处于安装或激活流程中，与 staged 会话读数相互印证"
  else
    row L1-INFO apexd_process_absent_normal "ps 中无 apexd：apexd 是 oneshot 服务，完成激活后正常退出；缺席不表示 apexd 未安装或未工作，不得据此判风险"
  fi
else
  row L0-LOW ps_unread "未采到进程表：不判定 apexd 进程位"
fi

# ---------- SDK 档位（清单口径适用性） ----------
case "$sdk" in
  ''|*[!0-9]*) : ;;
  *) if [ "$sdk" -lt 29 ] 2>/dev/null; then
       row L2-WARN apex_face_pre_apex "SDK=$sdk 早于 29：公开口径写明 APEX 容器是 Android 10 引入（introduced in Android 10），本报告的模块口径整体不适用，归属单位必须退回分区或整机镜像（M-AT10）"
     elif [ "$sdk" -lt 30 ] 2>/dev/null; then
       row L2-WARN apex_catalog_pre_mainline_doc "SDK=$sdk 为 Android 10：Mainline 清单口径自 Android 11 起成型，文档还写明 Mainline support for Android 11 and lower is concluded as of Q4 of 2025.，这台设备的清单比对结果只能作历史基线"
     fi ;;
esac
if [ -n "$rel" ] && [ -z "$sdk" ]; then
  row L0-LOW snapshot_header_partial "读到 release 但没读到 sdk：第五章的能力缺席判定需要两者同轴，本快照头部不完备（不进入 K2 分子）"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,apex_total=%s,apex_active=%s,apex_factory=%s,shadow_names=%s,catalog_present=%s,vendor_select=%s,updatable=%s,sdk=%s,model=%s,spl=%s\n' \
  "$hi" "$warn" "$gap" "$info" "$apex_total" "$act" "$fac" "$shadow_names" "${pres:-0}" "$vsel_n" "${updatable:-none}" "${sdk:-none}" "${model:-none}" "${spl:-none}"
```

### 廿二之六 样例输出

以下三份都是脚本对上述合成档案的**实际运行结果**（未删行、未改字段），分别取自第廿四章档案表里的 `mm001`（现代 GMS 发行设备基线）、`mm007`（全部未激活 + 有条目缺 `moduleName`）与 `mm014`（属性面残缺）；三份的构造点见第廿四章之三。合成档案不是任何真实设备，它们的作用是证明每个码位都能被触发。

```text
META,device_serial,mm001
META,model,Pixel 8 Pro
META,manufacturer,Google
META,board,shusbi
META,release,15
META,sdk,35
META,security_patch,2025-08-05
META,build_type,user
META,selinux,Enforcing
ROW,L1-INFO,selinux_enforcing,getenforce=Enforcing：本报告的预期失败表（第十三章）适用于这台设备，读不到多为策略效果而不是采集故障
COUNT,prop_lines,10
ROW,L1-INFO,release_read,ro.build.version.release=15：Mainline 与 APEX 的适用性都按这一行分档
ROW,L1-INFO,sdk_read,ro.build.version.sdk=35：与模块的 minSdkVersion/maxSdkVersion 同轴比较才有意义
ROW,L1-INFO,spl_read,security_patch=2025-08-05：只作时间锚点；SPL 前进不表示任何模块的 versionCode 变过（第十二章之三）
ROW,L1-INFO,build_type_user,ro.build.type=user：标准发行构建，本快照按标准策略预期解释各面失败
PROP,ro.apex.updatable,true
ROW,L1-INFO,apex_updatable_true,ro.apex.updatable=true：设备形态为可更新 APEX，模块更新可以落到 /data/apex/active 并在重启后激活
PROP,apex.all.ready,true
ROW,L1-INFO,apex_all_ready_true,apex.all.ready=true：所有 APEX 已激活就绪，此刻的模块集合可作为下一次差分的基线
PROP,apexd.status,unread
ROW,L1-INFO,apexd_status_expected_denied,apexd.status 读不到，且 SELinux 为 Enforcing：与 system_internal_prop 的 neverallow { domain -coredomain } $1:file no_rw_file_perms 一致，属预期结果，不是故障
PROP,ro.apexd.changed_active_apexes,unread
ROW,L1-INFO,apex_vendor_select_absent,未见 ro.boot.vendor.apex. 与 persist.vendor.apex. 前缀下的任何键：两个前缀在 property_contexts:356-357 声明为 apexd_select_prop（system_public_prop，读侧无限制），读不到即该设备未定义厂商侧选择键
COUNT,xml_bytes,4986
ROW,L1-INFO,apex_list_read,读到 /apex/apex-info-list.xml（4986 字节，15 条 apex-info 记录）：本快照的主读数面成立
APEX,com.android.adbd,351990000,351990000,true,true,SYSTEM,false,1738368000000
PATH,com.android.adbd,/apex/com.android.adbd@351990000,/system/apex/com.android.adbd.apex
APEX,com.android.art,351780000,351780000,true,true,SYSTEM,true,1738368000000
PATH,com.android.art,/apex/com.android.art@351780000,/system/apex/com.android.art.apex
APEX,com.android.conscrypt,351050000,351050000,true,true,SYSTEM,false,1738368000000
PATH,com.android.conscrypt,/apex/com.android.conscrypt@351050000,/system/apex/com.android.conscrypt.apex
APEX,com.android.media,350000000,350000000,true,true,SYSTEM,false,1738368000000
PATH,com.android.media,/apex/com.android.media,/system/apex/com.android.media.apex
APEX,com.android.tzdata,350000000,350000000,true,true,SYSTEM,false,1738368000000
PATH,com.android.tzdata,/apex/com.android.tzdata,/system/apex/com.android.tzdata.apex
APEX,com.android.uwb,350500000,350500000,true,true,SYSTEM,false,1738368000000
PATH,com.android.uwb,/apex/com.android.uwb,/system/apex/com.android.uwb.apex
APEX,com.android.wifi,350500000,350500000,true,true,SYSTEM,false,1738368000000
PATH,com.android.wifi,/apex/com.android.wifi,/system/apex/com.android.wifi.apex
APEX,com.android.resolv,350000000,350000000,true,true,SYSTEM,false,1738368000000
APEX,com.google.android.adservices,350700000,350700000,true,true,SYSTEM,false,1740000000000
PATH,com.google.android.adservices,/apex/com.google.android.adservices@350700000,/system/apex/com.google.android.adservices.apex
APEX,com.google.android.adservices,350900000,350900000,false,false,SYSTEM,false,1742000000000
PATH,com.google.android.adservices,/apex/com.google.android.adservices@350900000,/system/apex/com.google.android.adservices.apex
APEX,com.google.android.healthfitness,350400000,350400000,true,true,SYSTEM,false,1740000000000
PATH,com.google.android.healthfitness,/apex/com.google.android.healthfitness,/system/apex/com.google.android.healthfitness.apex
APEX,com.google.android.ondevicepersonalization,350600000,350600000,true,true,SYSTEM,false,1740000000000
PATH,com.google.android.ondevicepersonalization,/apex/com.google.android.ondevicepersonalization,/system/apex/com.google.android.ondevicepersonalization.apex
APEX,com.google.android.federatedcompute,350600000,350600000,true,true,SYSTEM,false,1740000000000
PATH,com.google.android.federatedcompute,/apex/com.google.android.federatedcompute,/system/apex/com.google.android.federatedcompute.apex
APEX,com.android.permissioncontroller,350010000,350010000,true,true,SYSTEM,false,1738368000000
PATH,com.android.permissioncontroller,/apex/com.android.permissioncontroller,/product/apex/com.android.permissioncontroller.apex
APEX,com.android.rkpd,350000000,350000000,true,true,SYSTEM,false,1738368000000
PATH,com.android.rkpd,/apex/com.android.rkpd,/system/apex/com.android.rkpd.apex
COUNT,apex_total,15
COUNT,apex_active,14
COUNT,apex_factory,14
COUNT,apex_shared_libs,1
COUNT,apex_partition_vendor,0
COUNT,apex_field_missing,0
COUNT,apex_partition_enum_bad,0
COUNT,apex_record_no_name,0
ROW,L1-INFO,apex_name_prefix_gms,5 条记录使用 com.google.android.* 前缀：与文档口径一致（GMS 设备由 Google 签名）
ROW,L1-INFO,apex_name_prefix_aosp,10 条记录使用 com.android.* 前缀：同一设备上两种前缀的分布由模块归属而不是设备类别单轴决定
ROW,L1-INFO,apex_name_prefix_mix,两种前缀同时存在（5 / 10）：第五章清单本身就同时列有 com.google.android.* 与 com.android.*，属正常形态；跨机型比对必须按后缀名而不是完整包名
ROW,L2-WARN,apex_installed_vs_active_gap,模块 com.google.android.adservices：同名存在 2 条记录，最高 versionCode=350900000 未激活，激活的是 350700000。文档口径为 the newly installed version of the APEX is only activated after reboot.，因此这是“已安装未激活”（M-AT4），不能记作已生效
COUNT,apex_shadow_names,1
COUNT,apex_same_version_shadow,0
ROW,L1-INFO,apex_shared_libs_present,1 条记录带 provideSharedApexLibs=true：这些模块向其它模块提供共享库，其版本变化的影响面超出自身进程（第十五章的 M5 旁证）
CATALOG,adbd,present
CATALOG,adservices,present
CATALOG,appsearch,absent
CATALOG,art,present
CATALOG,bt,absent
CATALOG,cellbroadcast,absent
CATALOG,configinfrastructure,absent
CATALOG,conscrypt,present
CATALOG,documentsui,absent
CATALOG,ext.services,absent
CATALOG,federatedcompute,present
CATALOG,healthfitness,present
CATALOG,ipsec,absent
CATALOG,media,present
CATALOG,media.swcodec,absent
CATALOG,mediaprovider,absent
CATALOG,modulemetadata,absent
CATALOG,networkstack,absent
CATALOG,neuralnetworks,absent
CATALOG,ondevicepersonalization,present
CATALOG,permissioncontroller,present
CATALOG,rkpd,present
CATALOG,resolv,present
CATALOG,scheduling,absent
CATALOG,sdkext,absent
CATALOG,statsd,absent
CATALOG,tethering,absent
CATALOG,tzdata,present
CATALOG,uwb,present
CATALOG,wifi,present
COUNT,catalog_total,30
COUNT,catalog_present,14
ROW,L0-LOW,apex_catalog_absent_list,清单内未在本机读到的后缀：appsearch bt cellbroadcast configinfrastructure documentsui ext.services ipsec media.swcodec mediaprovider modulemetadata networkstack neuralnetworks scheduling sdkext statsd tethering。按第五章纪律，缺席有三种在设备侧不可区分的情形（能力/厂商/读数），这一行不产出风险结论
DIR,apex_root,unread,
ROW,L1-INFO,apex_dir_unread_expected,未列出 /apex：shell.te 未给 apex_mnt_dir:dir，标准策略下这是预期结果，不作为故障，也不影响主读数面
DIR,vendor_apex,1,com.android.adbd.apex com.android.art.apex com.android.conscrypt.apex com.android.media.apex com.android.tzdata.apex com.android.wifi.apex com.android.uwb.apex 
ROW,L1-INFO,vendor_apex_dir_read,列出 /vendor/apex（1 项）：shell.te:164-165 明确授权 vendor_apex_file，这是厂商自带包的直接读数面，与平台包分开记账
ROW,L1-INFO,vendor_apex_all_in_list,/vendor/apex 下的包全部在清单中有对应记录：厂商自带包与主读数面自洽
DIR,apexdata,unread,
ROW,L1-INFO,apexdata_dir_unread_expected,未列出 /data/misc/apexdata：与标准策略一致，不作为故障
DIR,staged_sessions,unread,
ROW,L1-INFO,staged_session_unread_expected,未读到 staged 会话目录：与 init.rc:622-625（0700 root system）与 :743-751（0751 system system）的权限位一致。未读到不等于无进行中的会话，M4 的会话面在本快照里是缺口
SVC,cmd_apex,denied_selinux,avc:  denied  { find } for pid=4321 uid=2000 scontext=u:r:shell:s0 tcontext=u:object_r:apex_service:s0 tclass=service_manager permissive=0
ROW,L1-INFO,cmd_apex_denied_selinux,cmd apex 被 SELinux 拒绝（输出含 avc/denied）：与 shell.te:378-384 把 apex_service 列入排除集一致，责任在策略与设备类别，不在服务实现
SVC,dumpsys_apexservice,denied_selinux,avc:  denied  { find } for pid=4321 uid=2000 scontext=u:r:shell:s0 tcontext=u:object_r:apex_service:s0 tclass=service_manager permissive=0
ROW,L1-INFO,dumpsys_apexservice_denied_selinux,dumpsys apexservice 被 SELinux 拒绝：与 shell 域被排除在 apex_service 之外一致。dump 函数没有权限检查与 shell 能读到 dump 是两回事
DIR,supp_patch,unread,
ROW,L0-LOW,supp_patch_unread,未读到 /etc/security/supplemental_security_patches.xml：机制存在但写入与否由厂商决定，且 cat 无法区分文件不存在与无读权限。这一行是缺口，不是厂商未上报
ROW,L0-LOW,ps_unread,未采到进程表：不判定 apexd 进程位
SUMMARY,high=0,warn=1,gap=3,info=21,apex_total=15,apex_active=14,apex_factory=14,shadow_names=1,catalog_present=14,vendor_select=0,updatable=true,sdk=35,model=Pixel 8 Pro,spl=2025-08-05
```

```text
META,device_serial,mm007
META,model,Pixel 7
META,manufacturer,Google
META,board,panther
META,release,13
META,sdk,33
META,security_patch,2023-04-05
META,build_type,user
META,selinux,Enforcing
ROW,L1-INFO,selinux_enforcing,getenforce=Enforcing：本报告的预期失败表（第十三章）适用于这台设备，读不到多为策略效果而不是采集故障
COUNT,prop_lines,9
ROW,L1-INFO,release_read,ro.build.version.release=13：Mainline 与 APEX 的适用性都按这一行分档
ROW,L1-INFO,sdk_read,ro.build.version.sdk=33：与模块的 minSdkVersion/maxSdkVersion 同轴比较才有意义
ROW,L1-INFO,spl_read,security_patch=2023-04-05：只作时间锚点；SPL 前进不表示任何模块的 versionCode 变过（第十二章之三）
ROW,L1-INFO,build_type_user,ro.build.type=user：标准发行构建，本快照按标准策略预期解释各面失败
PROP,ro.apex.updatable,true
ROW,L1-INFO,apex_updatable_true,ro.apex.updatable=true：设备形态为可更新 APEX，模块更新可以落到 /data/apex/active 并在重启后激活
PROP,apex.all.ready,false
ROW,L2-WARN,apex_all_ready_false,apex.all.ready=false：采集发生在激活流程完成之前，此刻清单可能仍是 bootstrap 阶段的部分集合，本快照不得进入 K1 达标统计
PROP,apexd.status,unread
ROW,L1-INFO,apexd_status_expected_denied,apexd.status 读不到，且 SELinux 为 Enforcing：与 system_internal_prop 的 neverallow { domain -coredomain } $1:file no_rw_file_perms 一致，属预期结果，不是故障
PROP,ro.apexd.changed_active_apexes,unread
ROW,L1-INFO,apex_vendor_select_absent,未见 ro.boot.vendor.apex. 与 persist.vendor.apex. 前缀下的任何键：两个前缀在 property_contexts:356-357 声明为 apexd_select_prop（system_public_prop，读侧无限制），读不到即该设备未定义厂商侧选择键
COUNT,xml_bytes,717
ROW,L1-INFO,apex_list_read,读到 /apex/apex-info-list.xml（717 字节，3 条 apex-info 记录）：本快照的主读数面成立
APEX,com.android.wifi,330500000,330500000,true,false,SYSTEM,false,none
PATH,com.android.wifi,/apex/com.android.wifi,/system/apex/com.android.wifi.apex
APEX,com.android.adbd,331000000,331000000,true,false,SYSTEM,false,none
APEX,none,1,1,true,false,SYSTEM,false,none
ROW,L3-HIGH,apex_list_parse_mismatch,清单里有一条记录不含 moduleName（ApexInfoList.xsd:38 声明 use=required）：这条记录无法归属到任何模块，本快照对它的处理是丢弃并计数，不并入任何模块的统计
COUNT,apex_total,3
COUNT,apex_active,0
COUNT,apex_factory,2
COUNT,apex_shared_libs,0
COUNT,apex_partition_vendor,0
COUNT,apex_field_missing,0
COUNT,apex_partition_enum_bad,0
COUNT,apex_record_no_name,1
ROW,L1-INFO,apex_name_prefix_aosp,2 条记录使用 com.android.* 前缀：同一设备上两种前缀的分布由模块归属而不是设备类别单轴决定
COUNT,apex_shadow_names,0
COUNT,apex_same_version_shadow,0
ROW,L3-HIGH,apex_active_zero,读到清单但没有任何 isActive=true 的记录：主读数面与 M4 面同时不成立，这台设备的模块姿态无法由本快照描述
ROW,L1-INFO,apex_no_shadow_pairs,清单内没有任何同名多条的记录：这台设备当前不呈现“出厂版本被更新版本遮蔽”的形态，M4 面只有一态可读
CATALOG,adbd,present
CATALOG,adservices,absent
CATALOG,appsearch,absent
CATALOG,art,absent
CATALOG,bt,absent
CATALOG,cellbroadcast,absent
CATALOG,configinfrastructure,absent
CATALOG,conscrypt,absent
CATALOG,documentsui,absent
CATALOG,ext.services,absent
CATALOG,federatedcompute,absent
CATALOG,healthfitness,absent
CATALOG,ipsec,absent
CATALOG,media,absent
CATALOG,media.swcodec,absent
CATALOG,mediaprovider,absent
CATALOG,modulemetadata,absent
CATALOG,networkstack,absent
CATALOG,neuralnetworks,absent
CATALOG,ondevicepersonalization,absent
CATALOG,permissioncontroller,absent
CATALOG,rkpd,absent
CATALOG,resolv,absent
CATALOG,scheduling,absent
CATALOG,sdkext,absent
CATALOG,statsd,absent
CATALOG,tethering,absent
CATALOG,tzdata,absent
CATALOG,uwb,absent
CATALOG,wifi,present
COUNT,catalog_total,30
COUNT,catalog_present,2
ROW,L0-LOW,apex_catalog_absent_list,清单内未在本机读到的后缀：adservices appsearch art bt cellbroadcast configinfrastructure conscrypt documentsui ext.services federatedcompute healthfitness ipsec media media.swcodec mediaprovider modulemetadata networkstack neuralnetworks ondevicepersonalization permissioncontroller rkpd resolv scheduling sdkext statsd tethering tzdata uwb。按第五章纪律，缺席有三种在设备侧不可区分的情形（能力/厂商/读数），这一行不产出风险结论
DIR,apex_root,unread,
ROW,L1-INFO,apex_dir_unread_expected,未列出 /apex：shell.te 未给 apex_mnt_dir:dir，标准策略下这是预期结果，不作为故障，也不影响主读数面
DIR,vendor_apex,unread,
ROW,L0-LOW,vendor_apex_dir_unread,未列出 /vendor/apex：该面有明确 SELinux 授权，读不到通常是设备不存在该目录（无厂商自带 APEX），但两者在设备侧不可区分，记缺口
DIR,apexdata,unread,
ROW,L1-INFO,apexdata_dir_unread_expected,未列出 /data/misc/apexdata：与标准策略一致，不作为故障
DIR,staged_sessions,unread,
ROW,L1-INFO,staged_session_unread_expected,未读到 staged 会话目录：与 init.rc:622-625（0700 root system）与 :743-751（0751 system system）的权限位一致。未读到不等于无进行中的会话，M4 的会话面在本快照里是缺口
SVC,cmd_apex,denied_selinux,avc:  denied  { find } for pid=4321 uid=2000 scontext=u:r:shell:s0 tcontext=u:object_r:apex_service:s0 tclass=service_manager permissive=0
ROW,L1-INFO,cmd_apex_denied_selinux,cmd apex 被 SELinux 拒绝（输出含 avc/denied）：与 shell.te:378-384 把 apex_service 列入排除集一致，责任在策略与设备类别，不在服务实现
SVC,dumpsys_apexservice,denied_selinux,avc:  denied  { find } for pid=4321 uid=2000 scontext=u:r:shell:s0 tcontext=u:object_r:apex_service:s0 tclass=service_manager permissive=0
ROW,L1-INFO,dumpsys_apexservice_denied_selinux,dumpsys apexservice 被 SELinux 拒绝：与 shell 域被排除在 apex_service 之外一致。dump 函数没有权限检查与 shell 能读到 dump 是两回事
DIR,supp_patch,unread,
ROW,L0-LOW,supp_patch_unread,未读到 /etc/security/supplemental_security_patches.xml：机制存在但写入与否由厂商决定，且 cat 无法区分文件不存在与无读权限。这一行是缺口，不是厂商未上报
ROW,L1-INFO,apexd_process_present,ps 中有 apexd：apexd 是 oneshot + disabled（apexd.rc:1-8），出现在进程表说明此刻正处于安装或激活流程中，与 staged 会话读数相互印证
SUMMARY,high=2,warn=1,gap=3,info=17,apex_total=3,apex_active=0,apex_factory=2,shadow_names=0,catalog_present=2,vendor_select=0,updatable=true,sdk=33,model=Pixel 7,spl=2023-04-05
```

```text
META,device_serial,mm014
META,model,none
META,manufacturer,none
META,board,none
META,release,none
META,sdk,none
META,security_patch,none
META,build_type,none
META,selinux,none
ROW,L0-LOW,selinux_unread,未采到 getenforce：无法判定策略档位，后续所有失败都只能记为归因未定
COUNT,prop_lines,2
ROW,L0-LOW,release_unread,未采到平台版本号：第五章的“清单是否适用”判定与 M-AT3 判定全部降级为缺口
ROW,L0-LOW,sdk_unread,未采到 SDK 号：无法判断模块的目标平台范围（第八章的 minSdk/maxSdk）
ROW,L0-LOW,model_unread,未采到机型标识：跨设备比对与厂商公告检索都失去键，本快照只做单机基线
ROW,L0-LOW,spl_absent,未采到安全补丁级别：本快照不做任何到达性判断
ROW,L0-LOW,build_type_unread,未采到 ro.build.type：无法把“读不到”归因到策略档位
PROP,ro.apex.updatable,unread
ROW,L0-LOW,apex_updatable_unread,未读到 ro.apex.updatable：该键类型为 exported_default_prop（property_contexts:1505，exact bool），标准策略下可读；读不到可能是该键未定义，也可能是采集面问题，两者在设备侧不可区分
PROP,apex.all.ready,unread
ROW,L0-LOW,apex_all_ready_unread,未读到 apex.all.ready：其类型为 system_restricted_prop（只限制 property_service set，不限制读），因此读不到本身属采集面异常
PROP,apexd.status,unread
ROW,L0-LOW,apexd_status_unread_other,apexd.status 读不到，但设备不处于 Enforcing（none）：无法把这次失败归因到策略，按缺口处理
PROP,ro.apexd.changed_active_apexes,unread
ROW,L1-INFO,apex_vendor_select_absent,未见 ro.boot.vendor.apex. 与 persist.vendor.apex. 前缀下的任何键：两个前缀在 property_contexts:356-357 声明为 apexd_select_prop（system_public_prop，读侧无限制），读不到即该设备未定义厂商侧选择键
COUNT,xml_bytes,286
ROW,L1-INFO,apex_list_read,读到 /apex/apex-info-list.xml（286 字节，1 条 apex-info 记录）：本快照的主读数面成立
APEX,com.android.wifi,300000000,300000000,true,true,SYSTEM,false,none
COUNT,apex_total,1
COUNT,apex_active,1
COUNT,apex_factory,1
COUNT,apex_shared_libs,0
COUNT,apex_partition_vendor,0
COUNT,apex_field_missing,0
COUNT,apex_partition_enum_bad,0
COUNT,apex_record_no_name,0
ROW,L1-INFO,apex_name_prefix_aosp,1 条记录使用 com.android.* 前缀：同一设备上两种前缀的分布由模块归属而不是设备类别单轴决定
COUNT,apex_shadow_names,0
COUNT,apex_same_version_shadow,0
ROW,L1-INFO,apex_no_shadow_pairs,清单内没有任何同名多条的记录：这台设备当前不呈现“出厂版本被更新版本遮蔽”的形态，M4 面只有一态可读
CATALOG,adbd,absent
CATALOG,adservices,absent
CATALOG,appsearch,absent
CATALOG,art,absent
CATALOG,bt,absent
CATALOG,cellbroadcast,absent
CATALOG,configinfrastructure,absent
CATALOG,conscrypt,absent
CATALOG,documentsui,absent
CATALOG,ext.services,absent
CATALOG,federatedcompute,absent
CATALOG,healthfitness,absent
CATALOG,ipsec,absent
CATALOG,media,absent
CATALOG,media.swcodec,absent
CATALOG,mediaprovider,absent
CATALOG,modulemetadata,absent
CATALOG,networkstack,absent
CATALOG,neuralnetworks,absent
CATALOG,ondevicepersonalization,absent
CATALOG,permissioncontroller,absent
CATALOG,rkpd,absent
CATALOG,resolv,absent
CATALOG,scheduling,absent
CATALOG,sdkext,absent
CATALOG,statsd,absent
CATALOG,tethering,absent
CATALOG,tzdata,absent
CATALOG,uwb,absent
CATALOG,wifi,present
COUNT,catalog_total,30
COUNT,catalog_present,1
ROW,L0-LOW,apex_catalog_absent_list,清单内未在本机读到的后缀：adbd adservices appsearch art bt cellbroadcast configinfrastructure conscrypt documentsui ext.services federatedcompute healthfitness ipsec media media.swcodec mediaprovider modulemetadata networkstack neuralnetworks ondevicepersonalization permissioncontroller rkpd resolv scheduling sdkext statsd tethering tzdata uwb。按第五章纪律，缺席有三种在设备侧不可区分的情形（能力/厂商/读数），这一行不产出风险结论
DIR,apex_root,unread,
ROW,L1-INFO,apex_dir_unread_expected,未列出 /apex：shell.te 未给 apex_mnt_dir:dir，标准策略下这是预期结果，不作为故障，也不影响主读数面
DIR,vendor_apex,unread,
ROW,L0-LOW,vendor_apex_dir_unread,未列出 /vendor/apex：该面有明确 SELinux 授权，读不到通常是设备不存在该目录（无厂商自带 APEX），但两者在设备侧不可区分，记缺口
DIR,apexdata,unread,
ROW,L1-INFO,apexdata_dir_unread_expected,未列出 /data/misc/apexdata：与标准策略一致，不作为故障
DIR,staged_sessions,unread,
ROW,L1-INFO,staged_session_unread_expected,未读到 staged 会话目录：与 init.rc:622-625（0700 root system）与 :743-751（0751 system system）的权限位一致。未读到不等于无进行中的会话，M4 的会话面在本快照里是缺口
SVC,cmd_apex,unread,none
ROW,L0-LOW,cmd_apex_unread,cmd apex 无输出：无法把这次失败分流到 find 门还是调用者门，服务面记缺口。两种失败的责任方不同，混记会让工单派错方向（第十三之三）
SVC,dumpsys_apexservice,denied_selinux,avc:  denied  { find } for pid=4321 uid=2000 scontext=u:r:shell:s0 tcontext=u:object_r:apex_service:s0 tclass=service_manager permissive=0
ROW,L1-INFO,dumpsys_apexservice_denied_selinux,dumpsys apexservice 被 SELinux 拒绝：与 shell 域被排除在 apex_service 之外一致。dump 函数没有权限检查与 shell 能读到 dump 是两回事
DIR,supp_patch,unread,
ROW,L0-LOW,supp_patch_unread,未读到 /etc/security/supplemental_security_patches.xml：机制存在但写入与否由厂商决定，且 cat 无法区分文件不存在与无读权限。这一行是缺口，不是厂商未上报
ROW,L0-LOW,ps_unread,未采到进程表：不判定 apexd 进程位
SUMMARY,high=0,warn=0,gap=14,info=8,apex_total=1,apex_active=1,apex_factory=1,shadow_names=0,catalog_present=1,vendor_select=0,updatable=none,sdk=none,model=none,spl=none
```

---
## 廿三、脚本 2：模块姿态漂移比对

第二个可执行件是 `mainline-module-posture-drift.sh`。它**不接触设备**：输入是两份脚本 1 的输出文本，输出是这两份快照之间每一次可解释的变化，以及"这次变化能不能算一次变化"的资格判定。第廿一章的五档结论里，只有 M5（时间线）与 M6（口径一致性）需要差分才能成立，这个脚本就是那两面的采集件。

### 廿三之一 它证明什么，不证明什么

**它证明的**：两份快照之间哪些字段确实不同；这两份快照有没有资格被放在一起比较；差异的**形态**（一个模块动、一批模块动、什么都没动但 SPL 动了）更接近模块通道还是整机 OTA。

**它不证明的**：

1. **不证明差异是设备上先后发生的。** 两份快照可以来自两台设备。脚本用 `pair_identity_same_model` / `pair_identity_serial_diff` / `pair_identity_model_diff` 三档把这件事显式区分开——只有第一档才允许写成"变化"，后两档写出来的是"两台设备各自的姿态"。这是第二十章"不跨设备比较"那条纪律里唯一由脚本强制执行的部分。
2. **不证明 versionCode 前进等于补丁到达。** 与脚本 1 一样，最强的版本证据只到 `pair_apex_version_up`，释义行里就把"不能直接写成对应 CVE 已修"写进了文本，防止输出被截断引用后丢掉限定语。
3. **不证明"消失"等于"移除"。** `pair_apex_removed` 落在 L2-WARN 而不是 L3-HIGH，正是第五章那条纪律的实现：设备侧单点读数无法区分移除、未激活与清单世代差异。

还有一条**结构性限制**要在用之前知道：脚本只比对清单里逐字段可见的内容（模块名、versionCode、isFactory、isActive、partition、provideSharedApexLibs）。`modulePath` 与 `lastUpdateMillis` 会原样出现在快照里，但**不参与差分**——前者含 `@<versionCode>` 后缀，跟随版本变化，比对它等于把同一件事数两遍；后者在同一模块上是否刷新本文未获得可引用的实现依据，因此不敢用它做判据。这两列留在输出里供人工核对。

### 廿三之二 输入契约与前置门

```
mainline-module-posture-drift.sh <基线快照.txt> <对照快照.txt>
```

输入必须是脚本 1 的**完整**输出。契约上依赖这些前缀行：

| 前缀 | 用途 | 缺失时 |
| --- | --- | --- |
| `SUMMARY,` | 判定"这是一份脚本 1 的输出" | `pair_input_not_snapshot`（L3-HIGH），差分继续但结论标记为不成立 |
| `META,model / device_serial / build_type / selinux / release / security_patch` | 配对资格与背景字段差分 | 相应差分不执行，`pair_identity_unread` 落 L0 |
| `PROP,ro.apex.updatable`、`PROP,apex.all.ready` | 通道档位与就绪态差分 | 跳过 |
| `APEX,名称,versionCode,versionName,isFactory,isActive,partition,provideSharedApexLibs,lastUpdateMillis` | 模块级差分的唯一来源 | 该侧记 0 条，触发 `pair_list_side_unread` |
| `CATALOG,后缀,present\|absent` | 清单面差分 | 清单差分整体跳过 |
| `DIR,<面>,…`、`SVC,<面>,…` | 观测面可用性差分 | 该面对比跳过 |
| `VSEL,<键>,<值>` | 厂商选择属性族差分 | 视为空集 |

前置门的顺序是**先判身份，再判内容**。理由很直白：如果两侧根本不是同一台设备，后面每一行 `DELTA` 的读法都要变，所以身份行必须先出现在输出里，让人（也让复核脚本）在看差异之前先看到这句限定。

### 廿三之三 三十七个码位

**身份与背景（7 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_identity_unread` | L0-LOW | 至少一侧 `META,model` 为空 |
| `pair_identity_same_model` | L1-INFO | 两侧机型字符串完全相同 |
| `pair_identity_serial_diff` | L2-WARN | 机型相同、采集序列号不同：同型号两台机器，差异里混着个体与批次 |
| `pair_identity_model_diff` | L2-WARN | 机型不同：本次不是时间差分 |
| `pair_build_type_shift` | L2-WARN | `ro.build.type` 变化：两侧的"预期失败表"不可互换 |
| `pair_selinux_shift` | L2-WARN | `getenforce` 档位变化：第十三章的预期失败只在 Enforcing 下成立 |
| `pair_release_shift` | L2-WARN | `ro.build.version.release` 变化：通常意味着整机 OTA 背景 |

**通道档位与就绪（5 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_updatable_true_to_false` | L3-HIGH | `ro.apex.updatable` 由 `true` 变 `false`：模块级快修通道从有到无，先前登记的"模块侧已修"要退回整机 OTA 口径 |
| `pair_updatable_false_to_true` | L1-INFO | 反向：观测面出现，不代表任何缺陷已被更新 |
| `pair_updatable_value_shift` | L2-WARN | 两侧都不是标准布尔：`property_contexts:1505` 声明该键 `exact bool`，非布尔取值说明读数被污染 |
| `pair_readiness_flip_unready` | L2-WARN | `apex.all.ready` 由 `true` 变 `false`：对照侧落在激活流程完成之前，本次"消失"类结论作废 |
| `pair_readiness_flip_ready` | L1-INFO | 反向：对照侧变成完整读数，基线侧仍不得进入 K1 达标统计 |

**输入与读数面完整性（2 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_input_not_snapshot` | L3-HIGH | 任一侧没有 `SUMMARY,` 行 |
| `pair_list_side_unread` | L0-LOW | 任一侧 `APEX,` 行数为 0：模块级差分不成立，只输出属性面、目录面与服务面 |

**模块字段差分（13 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_apex_version_up` | L1-INFO | versionCode 数值前进 |
| `pair_apex_version_down` | L2-WARN | versionCode 数值后退：`apexd.cpp:3406-3407` 写明降级属 `not a supported case`，这一行只能是回滚或读数错位 |
| `pair_apex_version_nonnumeric` | L2-WARN | 至少一侧不是十进制整数：`ApexInfoList.xsd:41` 声明 `xs:long` |
| `pair_apex_version_unreadable` | L2-WARN | 一侧为 `none`：required 字段缺失，方向无法判定，记读数缺口而不是版本事件 |
| `pair_apex_active_flip_off` | L3-HIGH | `isActive` 由 `true` 变 `false`：M4 面上"这台机器在用这个模块"由成立转为不成立 |
| `pair_apex_active_flip_on` | L1-INFO | 反向：与文档 `the newly installed version of the APEX is only activated after reboot.` 的时序一致 |
| `pair_apex_factory_shift` | L2-WARN | `isFactory` 变化：多半是整机镜像换代，归属单位要跟着换 |
| `pair_apex_partition_shift` | L2-WARN | `partition` 变化：改变第六章映射落点 |
| `pair_classpath_flag_shift` | L2-WARN | `provideSharedApexLibs` 变化；释义里明确它与 AIDL `ApexInfo` 的 `hasClassPathJars` 不是同一字段 |
| `pair_apex_newly_present` | L1-INFO | 对照侧有、基线侧无（且未配上前缀迁移）：登记为新增观测，不登记为新增风险 |
| `pair_apex_removed` | L2-WARN | 基线侧有、对照侧无（同上）：需要第二侧证据才能定性 |
| `pair_name_prefix_shift` | L2-WARN | 一对 GONE/NEW 的后缀名相同、前缀不同：按文档口径这只是一次签名前缀迁移，台账登记一条 |
| `pair_active_count_change` | L1-INFO | 激活条目数变了，但没有任何模块逐条翻转 |

**批量形态判定（5 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_version_batch_shift` | L1-INFO | 前进/后退的模块数 ≥ 5：形态与整机 OTA 一致 |
| `pair_partial_batch_pattern` | L2-WARN | 1–4 个模块变化且 SPL 同期变化：可能是"部分前进"，也可能模块更新恰好与 OTA 同窗，这一行本身不区分 |
| `pair_module_level_update_pattern` | L1-INFO | 1–4 个模块变化且 SPL 未变：对应文档 `enables them to be updated outside of the normal Android release cycle` 的可观测痕迹 |
| `pair_spl_shift_no_module_change` | L2-WARN | SPL 前进但零个模块 versionCode 变化：第十二章之三的纪律生效 |
| `pair_no_change_within_readable` | L1-INFO | 两侧主读数面都可读且逐字段无差异：这是一个**有效的**"无变化"记录，可作为下一次差分的基线 |

**清单面、观测面与厂商选择（5 个）**

| 码位 | 等级 | 触发条件 |
| --- | --- | --- |
| `pair_catalog_newly_present` | L1-INFO | 第五章清单项 `absent → present`：只说明承载形态出现，不说明版本或到达性 |
| `pair_catalog_newly_absent` | L2-WARN | `present → absent`：缺席有三种在设备侧不可区分的情形 |
| `pair_face_availability_change` | L2-WARN | 某个目录面两侧可读性不同且一侧为 `unread`：这条描述的是**观测系统**而不是设备 |
| `pair_service_class_shift` | L2-WARN | `cmd apex` / `dumpsys apexservice` 的服务面分类变化：两次"失败"不能当同一种失败统计 |
| `pair_vendor_select_shift` | L2-WARN | `VSEL` 行集合两侧不一致：本文未定位到这些键的消费方实现，只登记痕迹变化 |

### 廿三之四 三个实现细节

**一，前缀迁移配对必须在逐条出行之类之前完成。** GONE 与 NEW 是从同一份 awk 双侧映射里同时落下来的两组记录。如果直接按组出码，一次 `com.android.wifi → com.google.android.wifi` 会产出"一个模块被移除（L2）+ 一个模块新增（L1）"两行，台账上就变成两次模块事件；而按文档口径这两者是同一模块在 GMS 设备与 AOSP 签名设备上的两种命名，只应登记一条前缀迁移。脚本因此先在出行循环之外构造 `pmap`（`旧名=新名` 的配对表，判定条件是剥掉两种前缀后的后缀相等且原名不等），再在循环内用 `in_pmap` 把已经配上对的两侧记录都吞掉。`DELTA` 行里登记的键是后缀名而不是任一完整包名，同样是这个原因。

**二，批量形态是一条优先级链，不是四个独立判断。** 判据顺序是：先数 versionCode 变化的模块数 `nver`，`nver ≥ 5` 直接归为批量；否则看 SPL 是否移动，移动则归为"部分批量"，未移动则归为"模块级通道"；`nver = 0` 时再看 SPL，移动就出 `pair_spl_shift_no_module_change`，不移动且无任何差异才出"有效无变化"。顺序本身就是结论：**批量优先于部分批量**，因为一次整机 OTA 同时携带模块版本前进与 SPL 前进，如果先判 SPL 会把正常的 OTA 误报成"部分前进"这种缺陷形态。阈值 5 不是文档给的，是本文的合成档案标定值——第廿八章的未文档化边界清单把它登记为本文自设阈值。

**三，同名多条记录按"该侧最后一条"参与差分，且这件事会被写进输出。** 清单里允许同名多条（遮蔽关系由脚本 1 的 `apex_installed_vs_active_gap` 与 `apex_dupe_shadow_same_version` 负责），awk 用 `av[name]=…` 赋值，后读入的自然覆盖先读入的。脚本不假装这个选择没有代价：一旦任一侧存在同名重复，它会出一条 `NOTE` 行，列出两侧的重复名，并写明"模块级比对对同名记录取该侧最后一条"。**用一个 NOTE 而不是 ROW，是因为重复本身不是差异，但它会让差异的解释改变**——这正是第廿一章 L1/L0 分界那条选择的延续：把"关于读数质量的事实"和"关于设备的事实"分成两种行前缀。

### 廿三之五 脚本正文

正文如下。它不接触设备：全部输入是两份文本快照，脚本本身没有任何 `adb` 调用，因此读者可以在没有设备的情况下复算第廿三章之六的四段输出。

```bash
#!/usr/bin/env bash
# 脚本 2: mainline-module-posture-drift.sh — 模块姿态漂移比对（只读，比对两份脚本 1 的文本输出）
# 用法: mainline-module-posture-drift.sh <基线快照.txt> <对照快照.txt>
# 输入只能是脚本 1 的输出文本；本脚本不访问设备、不调用 adb
set -u
a="${1:?用法: mainline-module-posture-drift.sh <基线快照.txt> <对照快照.txt>}"
b="${2:?缺少对照快照文件}"
[ -f "$a" ] || { echo "ERR,baseline_missing,$a" >&2; exit 1; }
[ -f "$b" ] || { echo "ERR,current_missing,$b" >&2; exit 1; }

hi=0; warn=0; gap=0; info=0
row() { # row <L3-HIGH|L2-WARN|L1-INFO|L0-LOW> <信号码> <文本>
  local lv="$1" code="$2" txt="$3"
  printf 'ROW,%s,%s,%s\n' "$lv" "$code" "$txt"
  case "$lv" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) warn=$(( warn + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
pair() { printf 'PAIR,%s,%s,%s\n' "$1" "${2:-none}" "${3:-none}"; }
note() { printf 'NOTE,%s\n' "$1"; }

# META/PROP 的"读不到"在脚本 1 里写成 none 或 unread；本脚本把它们一律归一为空串，
# 否则 "user" 与 "none" 会被当成一次 build_type 迁移，而它其实是缺口（廿一之一的 L1/L0 分界）
nz() { awk -F, -v t="$1" -v k="$2" '$1==t && $2==k {if ($3!="none" && $3!="unread") print $3; exit}'; }
mget() { printf '%s\n' "$1" | nz META "$2"; }
pget() { printf '%s\n' "$1" | nz PROP "$2"; }
cget() { printf '%s\n' "$1" | awk -F, -v k="$2" '$1=="COUNT" && $2==k {print $3; exit}'; }
sget() { printf '%s\n' "$1" | awk -F, -v k="$2" '$1=="SVC" && $2==k {print $3; exit}'; }
dget() { printf '%s\n' "$1" | awk -F, -v k="$2" '$1=="DIR" && $2==k {print $3; exit}'; }
sfx()  { local n="$1"; n="${n#com.google.android.}"; printf '%s' "${n#com.android.}"; }

sa="$(cat "$a")"; sb="$(cat "$b")"
for side in "$sa" "$sb"; do
  if ! printf '%s\n' "$side" | grep -q '^SUMMARY,'; then
    row L3-HIGH pair_input_not_snapshot "快照文件里没有 SUMMARY 行：它不是脚本 1 的完整输出（可能被截断，或来自别的工具），本次差分的结论不成立"
  fi
done

# ---------- 配对合法性：先判身份，再判内容 ----------
mod="$(mget "$sa" model)"; mod2="$(mget "$sb" model)"
ser="$(mget "$sa" device_serial)"; ser2="$(mget "$sb" device_serial)"
pair model "$mod" "$mod2"
if [ -z "$mod" ] || [ -z "$mod2" ]; then
  row L0-LOW pair_identity_unread "至少一侧没采到机型标识：本次差分把“未知设备”和“已知设备的变化”混在了一起，结果只能作线索，不进入 K9 闭环统计"
elif [ "$mod" = "$mod2" ]; then
  row L1-INFO pair_identity_same_model "两侧机型一致（$mod）：本次差分可比作同一台设备的两次采集"
  if [ -n "$ser" ] && [ -n "$ser2" ] && [ "$ser" != "$ser2" ]; then
    row L2-WARN pair_identity_serial_diff "机型相同但采集用的设备序列号不同（$ser / $ser2）：同型号两台设备之间的差异里混着个体差异与批次差异，不是时间差"
  fi
else
  row L2-WARN pair_identity_model_diff "机型不同（$mod → $mod2）：本报告禁止跨设备比较，这次差分的每一行都按“两台设备各自的姿态”解释，不得写成“一次变更”"
fi
bt="$(mget "$sa" build_type)"; bt2="$(mget "$sb" build_type)"
pair build_type "$bt" "$bt2"
if [ -n "$bt" ] && [ -n "$bt2" ] && [ "$bt" != "$bt2" ]; then
  row L2-WARN pair_build_type_shift "ro.build.type 由 $bt 变为 $bt2：发行构建与非发行构建的属性可读性、服务面可读性都不同，两侧的“预期失败”表不可互换"
fi
enf="$(mget "$sa" selinux)"; enf2="$(mget "$sb" selinux)"
pair selinux "$enf" "$enf2"
if [ -n "$enf" ] && [ -n "$enf2" ] && [ "$enf" != "$enf2" ]; then
  row L2-WARN pair_selinux_shift "SELinux 档位由 $enf 变为 $enf2：第十三章的预期失败表只在 Enforcing 下成立，换档后同一条“读不到”的含义随之改变"
fi
rel="$(mget "$sa" release)"; rel2="$(mget "$sb" release)"
pair release "$rel" "$rel2"
if [ -n "$rel" ] && [ -n "$rel2" ] && [ "$rel" != "$rel2" ]; then
  row L2-WARN pair_release_shift "平台版本由 $rel 变为 $rel2：这通常是一次整机 OTA，模块 versionCode 批量变化在这种背景下属预期，不能逐条写成“模块已修复”"
fi
spl="$(mget "$sa" security_patch)"; spl2="$(mget "$sb" security_patch)"
pair security_patch "$spl" "$spl2"

# ---------- 通道档位变化（M2 与 M3 的分叉点） ----------
ua="$(pget "$sa" ro.apex.updatable)"; ub="$(pget "$sb" ro.apex.updatable)"
pair ro.apex.updatable "$ua" "$ub"
if [ "$ua" = true ] && [ "$ub" = false ]; then
  row L3-HIGH pair_updatable_true_to_false "ro.apex.updatable 由 true 变为 false：模块级快修通道在这台设备上从有到无，先前登记的“模块侧已修”结论要重做，台账归属单位退回整机 OTA"
elif [ "$ua" = false ] && [ "$ub" = true ]; then
  row L1-INFO pair_updatable_false_to_true "ro.apex.updatable 由 false 变为 true：模块级通道变为存在，但这只打开观测面，不表示任何具体缺陷已被更新"
elif [ -n "$ua" ] && [ -n "$ub" ] && [ "$ua" != "$ub" ]; then
  row L2-WARN pair_updatable_value_shift "ro.apex.updatable 由 $ua 变为 $ub：两侧都不是标准布尔取值，档位判定不成立，按采集面问题处理"
fi
ra="$(pget "$sa" apex.all.ready)"; rb="$(pget "$sb" apex.all.ready)"
pair apex.all.ready "$ra" "$rb"
if [ "$ra" = true ] && [ "$rb" = false ]; then
  row L2-WARN pair_readiness_flip_unready "apex.all.ready 由 true 变为 false：对照侧的采集落在激活流程完成之前，它的模块集合可能是 bootstrap 阶段的部分集合，本次差分的“消失”类结论作废"
elif [ "$ra" = false ] && [ "$rb" = true ]; then
  row L1-INFO pair_readiness_flip_ready "apex.all.ready 由 false 变为 true：对照侧不再是半成品读数，但基线侧仍不得进入 K1 达标统计"
fi

# ---------- 主读数面可用性 ----------
xa="$(printf '%s\n' "$sa" | grep -c '^APEX,')"; xb="$(printf '%s\n' "$sb" | grep -c '^APEX,')"
pair apex_records "$xa" "$xb"
if [ "$xa" = 0 ] || [ "$xb" = 0 ]; then
  row L0-LOW pair_list_side_unread "至少一侧没有 apex-info 记录：模块级差分不成立，本次只输出属性面、目录面与服务面的变化，任何“模块消失”都不得写入台账"
fi
dup_a="$(printf '%s\n' "$sa" | awk -F, '$1=="APEX" && $2!="none"{print $2}' | sort | uniq -d | tr '\n' ' ')"
dup_b="$(printf '%s\n' "$sb" | awk -F, '$1=="APEX" && $2!="none"{print $2}' | sort | uniq -d | tr '\n' ' ')"
if [ -n "$dup_a" ] || [ -n "$dup_b" ]; then
  note "同名多条记录（出厂/激活并存）在基线侧为：${dup_a# }；对照侧为：${dup_b# }。模块级比对对同名记录取该侧最后一条，遮蔽关系本身由脚本 1 的 apex_installed_vs_active_gap 行负责"
fi

# ---------- 模块级差分 ----------
tfile() { printf '%s\n' "$1" | awk -F, '$1=="APEX" && $2!="none" {printf "%s|%s|%s|%s|%s|%s\n", $2,$3,$5,$6,$7,$8}'; }
ndelta=0; nver=0; nverup=0; nverdown=0; nnew=0; ngone=0; nflip=0; npshift=0
diffs=""
if [ "$xa" != 0 ] && [ "$xb" != 0 ]; then
  diffs="$( { printf 'SIDE|A\n'; tfile "$sa"; printf 'SIDE|B\n'; tfile "$sb"; } | awk -F'|' '
    $1=="SIDE" { side=$2; next }
    side=="A"  { av[$1]=$0; next }
    { bv[$1]=$0 }
    END {
      for (k in av) {
        if (!(k in bv)) { print "GONE|" av[k] }
        else {
          split(av[k], x, "|"); split(bv[k], y, "|");
          if (x[2]!=y[2]) print "VER|" x[1] "|" x[2] "|" y[2];
          if (x[3]!=y[3]) print "FACTORY|" x[1] "|" x[3] "|" y[3];
          if (x[4]!=y[4]) print "ACTIVE|" x[1] "|" x[4] "|" y[4];
          if (x[5]!=y[5]) print "PARTITION|" x[1] "|" x[5] "|" y[5];
          if (x[6]!=y[6]) print "SHARED|" x[1] "|" x[6] "|" y[6];
        }
      }
      for (k in bv) if (!(k in av)) print "NEW|" bv[k];
    }' | sort)"
fi
# 先做前缀迁移配对：GONE 与 NEW 只有在后缀名相同、前缀不同时才是"换签名"，不是"移除 + 新增"
gone_l="$(printf '%s\n' "$diffs" | awk -F'|' '$1=="GONE"{print $2}' | sort)"
new_l="$(printf '%s\n' "$diffs" | awk -F'|' '$1=="NEW"{print $2}' | sort)"
pmap=""
for g in $gone_l; do
  gs="$(sfx "$g")"
  for n in $new_l; do
    if [ "$(sfx "$n")" = "$gs" ] && [ "$g" != "$n" ]; then
      pmap="$pmap $g=$n"
      printf 'DELTA,%s,packageName,%s,%s\n' "$gs" "$g" "$n"
      row L2-WARN pair_name_prefix_shift "模块 ${gs} 的包名前缀由 $g 变为 $n：文档口径写明 GMS 设备用 com.google.android.*、AOSP 签名设备用 com.android.*，因此这一对 GONE/NEW 是同义词而不是两次模块事件，台账里只登记一条"
      npshift=$(( npshift + 1 ))
    fi
  done
done
in_pmap() { local t; for t in $pmap; do [ "${t%%=*}" = "$1" ] || [ "${t#*=}" = "$1" ] && return 0; done; return 1; }

while IFS='|' read -r k n1 n2 n3 n4 n5; do
  [ -n "$k" ] || continue
  case "$k" in
    VER)
      ndelta=$(( ndelta + 1 )); nver=$(( nver + 1 ))
      printf 'DELTA,%s,versionCode,%s,%s\n' "$n1" "$n2" "$n3"
      if [ "$n2" = none ] || [ "$n3" = none ]; then
        row L2-WARN pair_apex_version_unreadable "${n1} 的 versionCode 有一侧为 none（$n2 → $n3）：xsd:41 声明 required，字段缺失使升降方向无法判定，这一行只能记成读数缺口而不是版本事件"
      elif printf '%s%s' "$n2" "$n3" | grep -q '[^0-9]'; then
        row L2-WARN pair_apex_version_nonnumeric "${n1} 的 versionCode 至少一侧不是十进制整数（$n2 → $n3）：xsd:41 声明 xs:long，非数值说明读数被污染，本条不参与升降判定"
      elif [ "$n3" -gt "$n2" ] 2>/dev/null; then
        nverup=$(( nverup + 1 ))
        row L1-INFO pair_apex_version_up "${n1} 的 versionCode 由 $n2 升至 $n3：安装或激活确有发生，但按八之一，versionCode 不充当补丁指纹，这一行不能直接写成对应 CVE 已修"
      elif [ "$n3" -lt "$n2" ] 2>/dev/null; then
        nverdown=$(( nverdown + 1 ))
        row L2-WARN pair_apex_version_down "${n1} 的 versionCode 由 $n2 降至 $n3：apexd.cpp:3406-3407 写明降级属 not a supported case，这一行是回滚或读数错位，必须逐条人工确认后再进台账"
      fi ;;
    ACTIVE)
      ndelta=$(( ndelta + 1 )); nflip=$(( nflip + 1 ))
      printf 'DELTA,%s,isActive,%s,%s\n' "$n1" "$n2" "$n3"
      if [ "$n2" = true ] && [ "$n3" = false ]; then
        row L3-HIGH pair_apex_active_flip_off "${n1} 由 isActive=true 变为 false：模块退出激活态，M4 面上“这台机器在用这个模块”的结论由成立转为不成立；先排除对照侧落在激活流程中途，再考虑回滚"
      else
        row L1-INFO pair_apex_active_flip_on "${n1} 由 isActive=false 变为 true：一次安装完成激活，与文档 the newly installed version of the APEX is only activated after reboot. 的时序一致"
      fi ;;
    FACTORY)
      ndelta=$(( ndelta + 1 ))
      printf 'DELTA,%s,isFactory,%s,%s\n' "$n1" "$n2" "$n3"
      row L2-WARN pair_apex_factory_shift "${n1} 的 isFactory 由 $n2 变为 $n3：出厂内置标记变化通常来自整机镜像换代，而不是模块通道更新，归属单位要跟着换" ;;
    PARTITION)
      ndelta=$(( ndelta + 1 ))
      printf 'DELTA,%s,partition,%s,%s\n' "$n1" "$n2" "$n3"
      row L2-WARN pair_apex_partition_shift "${n1} 的 partition 由 $n2 变为 $n3：承载分区变化会改变第六章映射的落点，归属单位需按新分区重述" ;;
    SHARED)
      ndelta=$(( ndelta + 1 ))
      printf 'DELTA,%s,provideSharedApexLibs,%s,%s\n' "$n1" "$n2" "$n3"
      row L2-WARN pair_classpath_flag_shift "${n1} 的 provideSharedApexLibs 由 $n2 变为 $n3：共享库提供标记影响其它模块的解析路径；注意它与 AIDL ApexInfo 的 hasClassPathJars 不是同一字段，两侧读数不可互相替代" ;;
    NEW)
      if in_pmap "$n1"; then continue; fi
      nnew=$(( nnew + 1 )); ndelta=$(( ndelta + 1 ))
      printf 'NEW,%s,%s,%s\n' "$n1" "$n2" "$n4"
      row L1-INFO pair_apex_newly_present "对照侧出现基线里没有的模块 ${n1}（versionCode=${n2}）：可能是清单换代或厂商追加，登记为新增观测，不登记为新增风险" ;;
    GONE)
      if in_pmap "$n1"; then continue; fi
      ngone=$(( ngone + 1 )); ndelta=$(( ndelta + 1 ))
      printf 'GONE,%s,%s,%s\n' "$n1" "$n2" "$n4"
      row L2-WARN pair_apex_removed "基线里的模块 ${n1} 在对照侧消失：按第五章纪律，“消失”不等于“已移除”，也可能是激活状态、采集时机或清单世代差异，需要第二侧证据才能定性" ;;
  esac
done <<EOF
$diffs
EOF

# ---------- 批量形态判定（整机 OTA 与模块通道的可分辨特征） ----------
acnt="$(cget "$sa" apex_active)"; bcnt="$(cget "$sb" apex_active)"
pair apex_active "$acnt" "$bcnt"
if [ -n "$acnt" ] && [ -n "$bcnt" ] && [ "$acnt" != "$bcnt" ] && [ "$nflip" = 0 ]; then
  row L1-INFO pair_active_count_change "激活条目数由 $acnt 变为 $bcnt，但没有任何模块的 isActive 逐条翻转：apexd.cpp:2221-2222 写明 staged session 会改变激活集合，文档又写明激活要等到重启，两侧的采集时机不同就会产出不同计数，条目数变化本身不构成风险信号"
fi
splmoved=0; [ -n "$spl" ] && [ -n "$spl2" ] && [ "$spl" != "$spl2" ] && splmoved=1
if [ "$nver" != 0 ]; then
  if [ "$nver" -ge 5 ]; then
    row L1-INFO pair_version_batch_shift "$nver 个模块的 versionCode 同时变化：批量前进与整机 OTA 的形态一致，按公告口径应查平台侧版本而不是模块通道"
  elif [ "$splmoved" = 1 ]; then
    row L2-WARN pair_partial_batch_pattern "$nver 个模块变化且 SPL 同期由 $spl 变为 $spl2：可能是“部分前进”的不完整批量，也可能是模块更新恰好与 OTA 同窗；这一行本身不能区分两者，需第二侧证据"
  else
    row L1-INFO pair_module_level_update_pattern "$nver 个模块变化且 SPL 未变：与模块级通道的独立更新形态一致，正是文档所述 enables them to be updated outside of the normal Android release cycle 的可观测痕迹"
  fi
elif [ "$splmoved" = 1 ]; then
  row L2-WARN pair_spl_shift_no_module_change "SPL 由 $spl 变为 $spl2 但没有任何模块 versionCode 变化：第十二章之三的纪律生效，SPL 前进不携带模块版本信息，不得据此写“模块侧补丁已到达”"
elif [ "$xa" != 0 ] && [ "$xb" != 0 ] && [ "$ndelta" = 0 ]; then
  row L1-INFO pair_no_change_within_readable "两侧主读数面都可读且逐字段无差异：这是一个有效的“无变化”记录，可作为下一次差分的基线；它不等于“没有待修问题”"
fi

# ---------- 清单面差分（只在两侧主读数面都成立时进行） ----------
if [ "$xa" != 0 ] && [ "$xb" != 0 ]; then
  cat_a="$(printf '%s\n' "$sa" | awk -F, '$1=="CATALOG"{print $2"|"$3}' | sort)"
  cat_b="$(printf '%s\n' "$sb" | awk -F, '$1=="CATALOG"{print $2"|"$3}' | sort)"
  np=0; na=0
  while IFS='|' read -r s fa; do
    [ -n "$s" ] || continue
    fb="$(printf '%s\n' "$cat_b" | awk -F'|' -v k="$s" '$1==k{print $2; exit}')"
    [ -n "$fb" ] || continue
    [ "$fa" = "$fb" ] && continue
    printf 'DELTA,catalog:%s,state,%s,%s\n' "$s" "$fa" "$fb"
    if [ "$fb" = present ]; then
      np=$(( np + 1 ))
      row L1-INFO pair_catalog_newly_present "第五章清单项 $s 由 absent 变为 present：能力面出现，通常来自整机镜像换代；它只说明承载形态存在，不说明版本或到达性"
    else
      na=$(( na + 1 ))
      row L2-WARN pair_catalog_newly_absent "第五章清单项 $s 由 present 变为 absent：按第五章纪律，缺席有设备能力/厂商选择/读数失败三种在设备侧不可区分的情形，需交叉第二侧读数才能定性"
    fi
  done <<EOF
$cat_a
EOF
  [ "$np" = 0 ] && [ "$na" = 0 ] && note "清单面（第五章 30 项）在两侧之间无变化"
fi

# ---------- 观测面可用性差分（观测系统自身的漂移） ----------
for face in apex_root vendor_apex apexdata staged_sessions supp_patch; do
  va="$(dget "$sa" "$face")"; vb="$(dget "$sb" "$face")"
  pair "dir:$face" "$va" "$vb"
  if [ -n "$va" ] && [ -n "$vb" ] && [ "$va" != "$vb" ] && { [ "$va" = unread ] || [ "$vb" = unread ]; }; then
    row L2-WARN pair_face_availability_change "目录面 $face 的可读性由 $va 变为 $vb：一侧为 unread 说明采集域或策略档位变了，这条变化描述的是观测系统而不是设备"
  fi
done
for face in cmd_apex dumpsys_apexservice; do
  va="$(sget "$sa" "$face")"; vb="$(sget "$sb" "$face")"
  pair "svc:$face" "$va" "$vb"
  if [ -n "$va" ] && [ -n "$vb" ] && [ "$va" != "$vb" ]; then
    row L2-WARN pair_service_class_shift "$face 的服务面分类由 $va 变为 $vb：分流到不同责任方（find 门 vs 调用者校验）说明两侧的采集身份或策略世代不同，两次“失败”不能当同一种失败统计"
  fi
done

# ---------- 厂商选择属性族差分 ----------
vs_a="$(printf '%s\n' "$sa" | awk -F, '$1=="VSEL"{print $2"|"$3}' | sort)"
vs_b="$(printf '%s\n' "$sb" | awk -F, '$1=="VSEL"{print $2"|"$3}' | sort)"
if [ "$vs_a" != "$vs_b" ]; then
  vsc="$(comm -3 <(printf '%s\n' "$vs_a") <(printf '%s\n' "$vs_b") | grep -c '[^[:space:]]')"
  if [ "$vsc" != 0 ]; then
    printf 'DELTA,vendor_select,keys,%s,%s\n' "$vsc" "-"
    row L2-WARN pair_vendor_select_shift "厂商选择属性族两侧不一致（$vsc 行差异）：本文未定位到这些键的消费方实现，因此只登记痕迹变化，通道判定要等厂商确认后才成立"
  fi
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,ver_delta=%s,ver_up=%s,ver_down=%s,active_flip=%s,prefix_shift=%s,new=%s,gone=%s,module_delta_total=%s,model_a=%s,model_b=%s\n' \
  "$hi" "$warn" "$gap" "$info" "$nver" "$nverup" "$nverdown" "$nflip" "$npshift" "$nnew" "$ngone" "$ndelta" "$mod" "$mod2"
```

### 廿三之六 四份真实差分输出

下面四段是第廿四章验收环境（合成设备档案 + 替身 `adb`）里脚本 2 的**完整**原始输出，未经删改。四段分别覆盖：同一设备同一次采集的自比对、一份只改 SPL 的对照、一份逐字段类型都变了的对照、以及一份跨设备且策略档位不同的对照。

第一份，`mm001` 与自身比对，用来确认"无变化"是一条**有资格**的记录而不是空输出：

```text
PAIR,model,Pixel 8 Pro,Pixel 8 Pro
ROW,L1-INFO,pair_identity_same_model,两侧机型一致（Pixel 8 Pro）：本次差分可比作同一台设备的两次采集
PAIR,build_type,user,user
PAIR,selinux,Enforcing,Enforcing
PAIR,release,15,15
PAIR,security_patch,2025-08-05,2025-08-05
PAIR,ro.apex.updatable,true,true
PAIR,apex.all.ready,true,true
PAIR,apex_records,15,15
NOTE,同名多条记录（出厂/激活并存）在基线侧为：com.google.android.adservices ；对照侧为：com.google.android.adservices 。模块级比对对同名记录取该侧最后一条，遮蔽关系本身由脚本 1 的 apex_installed_vs_active_gap 行负责
PAIR,apex_active,14,14
ROW,L1-INFO,pair_no_change_within_readable,两侧主读数面都可读且逐字段无差异：这是一个有效的“无变化”记录，可作为下一次差分的基线；它不等于“没有待修问题”
NOTE,清单面（第五章 30 项）在两侧之间无变化
PAIR,dir:apex_root,unread,unread
PAIR,dir:vendor_apex,1,1
PAIR,dir:apexdata,unread,unread
PAIR,dir:staged_sessions,unread,unread
PAIR,dir:supp_patch,unread,unread
PAIR,svc:cmd_apex,denied_selinux,denied_selinux
PAIR,svc:dumpsys_apexservice,denied_selinux,denied_selinux
SUMMARY,high=0,warn=0,gap=0,info=2,ver_delta=0,ver_up=0,ver_down=0,active_flip=0,prefix_shift=0,new=0,gone=0,module_delta_total=0,model_a=Pixel 8 Pro,model_b=Pixel 8 Pro
```

第二份，`mm001 → mm017`（只把 `ro.build.version.security_patch` 从 2025-08-05 推到 2025-09-05），用来读出 `pair_spl_shift_no_module_change` 的完整释义：

```text
PAIR,model,Pixel 8 Pro,Pixel 8 Pro
ROW,L1-INFO,pair_identity_same_model,两侧机型一致（Pixel 8 Pro）：本次差分可比作同一台设备的两次采集
ROW,L2-WARN,pair_identity_serial_diff,机型相同但采集用的设备序列号不同（mm001 / mm017）：同型号两台设备之间的差异里混着个体差异与批次差异，不是时间差
PAIR,build_type,user,user
PAIR,selinux,Enforcing,Enforcing
PAIR,release,15,15
PAIR,security_patch,2025-08-05,2025-09-05
PAIR,ro.apex.updatable,true,true
PAIR,apex.all.ready,true,true
PAIR,apex_records,15,15
NOTE,同名多条记录（出厂/激活并存）在基线侧为：com.google.android.adservices ；对照侧为：com.google.android.adservices 。模块级比对对同名记录取该侧最后一条，遮蔽关系本身由脚本 1 的 apex_installed_vs_active_gap 行负责
PAIR,apex_active,14,14
ROW,L2-WARN,pair_spl_shift_no_module_change,SPL 由 2025-08-05 变为 2025-09-05 但没有任何模块 versionCode 变化：第十二章之三的纪律生效，SPL 前进不携带模块版本信息，不得据此写“模块侧补丁已到达”
NOTE,清单面（第五章 30 项）在两侧之间无变化
PAIR,dir:apex_root,unread,unread
PAIR,dir:vendor_apex,1,1
PAIR,dir:apexdata,unread,unread
PAIR,dir:staged_sessions,unread,unread
PAIR,dir:supp_patch,unread,unread
PAIR,svc:cmd_apex,denied_selinux,denied_selinux
PAIR,svc:dumpsys_apexservice,denied_selinux,denied_selinux
SUMMARY,high=0,warn=2,gap=0,info=1,ver_delta=0,ver_up=0,ver_down=0,active_flip=0,prefix_shift=0,new=0,gone=0,module_delta_total=0,model_a=Pixel 8 Pro,model_b=Pixel 8 Pro
```

第三份，`mm001 → mm015`，一次同时包含前缀迁移、版本改非数值、版本前进、出厂位与激活位翻转、分区位翻转、共享库位翻转、模块删除与模块新增的复合差分。第廿三章之三那张"模块字段差分"组共 13 个码位，这一份样例里同时出现 9 个（`pair_name_prefix_shift`、`pair_apex_version_up`、`pair_apex_version_nonnumeric`、`pair_apex_active_flip_off`、`pair_apex_factory_shift`、`pair_apex_partition_shift`、`pair_classpath_flag_shift`、`pair_apex_newly_present`、`pair_apex_removed`）；余下 4 个由第廿四章矩阵里的反向对与 `mm001:mm004` 补齐，**没有任何一份样例能独自打满这一组**，这正是验收要跑 22 个配对而不是 1 个的原因。该份输出如下：

```text
PAIR,model,Pixel 8 Pro,Pixel 8 Pro
ROW,L1-INFO,pair_identity_same_model,两侧机型一致（Pixel 8 Pro）：本次差分可比作同一台设备的两次采集
ROW,L2-WARN,pair_identity_serial_diff,机型相同但采集用的设备序列号不同（mm001 / mm015）：同型号两台设备之间的差异里混着个体差异与批次差异，不是时间差
PAIR,build_type,user,user
PAIR,selinux,Enforcing,Enforcing
PAIR,release,15,15
PAIR,security_patch,2025-08-05,2025-08-05
PAIR,ro.apex.updatable,true,true
PAIR,apex.all.ready,true,true
PAIR,apex_records,15,15
NOTE,同名多条记录（出厂/激活并存）在基线侧为：com.google.android.adservices ；对照侧为：com.google.android.adservices 。模块级比对对同名记录取该侧最后一条，遮蔽关系本身由脚本 1 的 apex_installed_vs_active_gap 行负责
DELTA,wifi,packageName,com.android.wifi,com.google.android.wifi
ROW,L2-WARN,pair_name_prefix_shift,模块 wifi 的包名前缀由 com.android.wifi 变为 com.google.android.wifi：文档口径写明 GMS 设备用 com.google.android.*、AOSP 签名设备用 com.android.*，因此这一对 GONE/NEW 是同义词而不是两次模块事件，台账里只登记一条
DELTA,com.android.uwb,isActive,true,false
ROW,L3-HIGH,pair_apex_active_flip_off,com.android.uwb 由 isActive=true 变为 false：模块退出激活态，M4 面上“这台机器在用这个模块”的结论由成立转为不成立；先排除对照侧落在激活流程中途，再考虑回滚
DELTA,com.android.conscrypt,isFactory,true,false
ROW,L2-WARN,pair_apex_factory_shift,com.android.conscrypt 的 isFactory 由 true 变为 false：出厂内置标记变化通常来自整机镜像换代，而不是模块通道更新，归属单位要跟着换
GONE,com.android.resolv,350000000,true
ROW,L2-WARN,pair_apex_removed,基线里的模块 com.android.resolv 在对照侧消失：按第五章纪律，“消失”不等于“已移除”，也可能是激活状态、采集时机或清单世代差异，需要第二侧证据才能定性
NEW,com.android.neuralnetworks,351200000,true
ROW,L1-INFO,pair_apex_newly_present,对照侧出现基线里没有的模块 com.android.neuralnetworks（versionCode=351200000）：可能是清单换代或厂商追加，登记为新增观测，不登记为新增风险
DELTA,com.android.rkpd,partition,SYSTEM,VENDOR
ROW,L2-WARN,pair_apex_partition_shift,com.android.rkpd 的 partition 由 SYSTEM 变为 VENDOR：承载分区变化会改变第六章映射的落点，归属单位需按新分区重述
DELTA,com.android.tzdata,provideSharedApexLibs,false,true
ROW,L2-WARN,pair_classpath_flag_shift,com.android.tzdata 的 provideSharedApexLibs 由 false 变为 true：共享库提供标记影响其它模块的解析路径；注意它与 AIDL ApexInfo 的 hasClassPathJars 不是同一字段，两侧读数不可互相替代
DELTA,com.android.conscrypt,versionCode,351050000,351060000
ROW,L1-INFO,pair_apex_version_up,com.android.conscrypt 的 versionCode 由 351050000 升至 351060000：安装或激活确有发生，但按八之一，versionCode 不充当补丁指纹，这一行不能直接写成对应 CVE 已修
DELTA,com.android.media,versionCode,350000000,35xx-beta
ROW,L2-WARN,pair_apex_version_nonnumeric,com.android.media 的 versionCode 至少一侧不是十进制整数（350000000 → 35xx-beta）：xsd:41 声明 xs:long，非数值说明读数被污染，本条不参与升降判定
PAIR,apex_active,14,13
ROW,L1-INFO,pair_module_level_update_pattern,2 个模块变化且 SPL 未变：与模块级通道的独立更新形态一致，正是文档所述 enables them to be updated outside of the normal Android release cycle 的可观测痕迹
DELTA,catalog:neuralnetworks,state,absent,present
ROW,L1-INFO,pair_catalog_newly_present,第五章清单项 neuralnetworks 由 absent 变为 present：能力面出现，通常来自整机镜像换代；它只说明承载形态存在，不说明版本或到达性
DELTA,catalog:resolv,state,present,absent
ROW,L2-WARN,pair_catalog_newly_absent,第五章清单项 resolv 由 present 变为 absent：按第五章纪律，缺席有设备能力/厂商选择/读数失败三种在设备侧不可区分的情形，需交叉第二侧读数才能定性
PAIR,dir:apex_root,unread,unread
PAIR,dir:vendor_apex,1,1
PAIR,dir:apexdata,unread,unread
PAIR,dir:staged_sessions,unread,unread
PAIR,dir:supp_patch,unread,unread
PAIR,svc:cmd_apex,denied_selinux,denied_selinux
PAIR,svc:dumpsys_apexservice,denied_selinux,denied_selinux
SUMMARY,high=1,warn=8,gap=0,info=5,ver_delta=2,ver_up=1,ver_down=0,active_flip=1,prefix_shift=1,new=1,gone=1,module_delta_total=8,model_a=Pixel 8 Pro,model_b=Pixel 8 Pro
```

第四份，`mm003 → mm004`，跨机型、跨 SELinux 档位、跨发行/非发行构建的对照，用来读出"这次比较本身不成立"时脚本会说什么：

```text
PAIR,model,Pixel 4a,DevBoard X
ROW,L2-WARN,pair_identity_model_diff,机型不同（Pixel 4a → DevBoard X）：本报告禁止跨设备比较，这次差分的每一行都按“两台设备各自的姿态”解释，不得写成“一次变更”
PAIR,build_type,user,userdebug
ROW,L2-WARN,pair_build_type_shift,ro.build.type 由 user 变为 userdebug：发行构建与非发行构建的属性可读性、服务面可读性都不同，两侧的“预期失败”表不可互换
PAIR,selinux,Enforcing,Permissive
ROW,L2-WARN,pair_selinux_shift,SELinux 档位由 Enforcing 变为 Permissive：第十三章的预期失败表只在 Enforcing 下成立，换档后同一条“读不到”的含义随之改变
PAIR,release,11,14
ROW,L2-WARN,pair_release_shift,平台版本由 11 变为 14：这通常是一次整机 OTA，模块 versionCode 批量变化在这种背景下属预期，不能逐条写成“模块已修复”
PAIR,security_patch,2021-10-05,2024-05-05
PAIR,ro.apex.updatable,true,yes
ROW,L2-WARN,pair_updatable_value_shift,ro.apex.updatable 由 true 变为 yes：两侧都不是标准布尔取值，档位判定不成立，按采集面问题处理
PAIR,apex.all.ready,true,true
PAIR,apex_records,5,4
GONE,com.android.cellbroadcast,300500000,true
ROW,L2-WARN,pair_apex_removed,基线里的模块 com.android.cellbroadcast 在对照侧消失：按第五章纪律，“消失”不等于“已移除”，也可能是激活状态、采集时机或清单世代差异，需要第二侧证据才能定性
GONE,com.android.conscrypt,300500000,true
ROW,L2-WARN,pair_apex_removed,基线里的模块 com.android.conscrypt 在对照侧消失：按第五章纪律，“消失”不等于“已移除”，也可能是激活状态、采集时机或清单世代差异，需要第二侧证据才能定性
GONE,com.android.oemfeature,100,true
ROW,L2-WARN,pair_apex_removed,基线里的模块 com.android.oemfeature 在对照侧消失：按第五章纪律，“消失”不等于“已移除”，也可能是激活状态、采集时机或清单世代差异，需要第二侧证据才能定性
NEW,com.android.art,none,true
ROW,L1-INFO,pair_apex_newly_present,对照侧出现基线里没有的模块 com.android.art（versionCode=none）：可能是清单换代或厂商追加，登记为新增观测，不登记为新增风险
NEW,com.android.media,340500000,true
ROW,L1-INFO,pair_apex_newly_present,对照侧出现基线里没有的模块 com.android.media（versionCode=340500000）：可能是清单换代或厂商追加，登记为新增观测，不登记为新增风险
DELTA,com.android.adbd,partition,SYSTEM,system
ROW,L2-WARN,pair_apex_partition_shift,com.android.adbd 的 partition 由 SYSTEM 变为 system：承载分区变化会改变第六章映射的落点，归属单位需按新分区重述
DELTA,com.android.wifi,partition,SYSTEM,VENDOR
ROW,L2-WARN,pair_apex_partition_shift,com.android.wifi 的 partition 由 SYSTEM 变为 VENDOR：承载分区变化会改变第六章映射的落点，归属单位需按新分区重述
DELTA,com.android.adbd,versionCode,300500000,340500000
ROW,L1-INFO,pair_apex_version_up,com.android.adbd 的 versionCode 由 300500000 升至 340500000：安装或激活确有发生，但按八之一，versionCode 不充当补丁指纹，这一行不能直接写成对应 CVE 已修
DELTA,com.android.wifi,versionCode,300500000,340500000
ROW,L1-INFO,pair_apex_version_up,com.android.wifi 的 versionCode 由 300500000 升至 340500000：安装或激活确有发生，但按八之一，versionCode 不充当补丁指纹，这一行不能直接写成对应 CVE 已修
PAIR,apex_active,5,4
ROW,L1-INFO,pair_active_count_change,激活条目数由 5 变为 4，但没有任何模块的 isActive 逐条翻转：apexd.cpp:2221-2222 写明 staged session 会改变激活集合，文档又写明激活要等到重启，两侧的采集时机不同就会产出不同计数，条目数变化本身不构成风险信号
ROW,L2-WARN,pair_partial_batch_pattern,2 个模块变化且 SPL 同期由 2021-10-05 变为 2024-05-05：可能是“部分前进”的不完整批量，也可能是模块更新恰好与 OTA 同窗；这一行本身不能区分两者，需第二侧证据
DELTA,catalog:art,state,absent,present
ROW,L1-INFO,pair_catalog_newly_present,第五章清单项 art 由 absent 变为 present：能力面出现，通常来自整机镜像换代；它只说明承载形态存在，不说明版本或到达性
DELTA,catalog:cellbroadcast,state,present,absent
ROW,L2-WARN,pair_catalog_newly_absent,第五章清单项 cellbroadcast 由 present 变为 absent：按第五章纪律，缺席有设备能力/厂商选择/读数失败三种在设备侧不可区分的情形，需交叉第二侧读数才能定性
DELTA,catalog:conscrypt,state,present,absent
ROW,L2-WARN,pair_catalog_newly_absent,第五章清单项 conscrypt 由 present 变为 absent：按第五章纪律，缺席有设备能力/厂商选择/读数失败三种在设备侧不可区分的情形，需交叉第二侧读数才能定性
DELTA,catalog:media,state,absent,present
ROW,L1-INFO,pair_catalog_newly_present,第五章清单项 media 由 absent 变为 present：能力面出现，通常来自整机镜像换代；它只说明承载形态存在，不说明版本或到达性
PAIR,dir:apex_root,5,unread
ROW,L2-WARN,pair_face_availability_change,目录面 apex_root 的可读性由 5 变为 unread：一侧为 unread 说明采集域或策略档位变了，这条变化描述的是观测系统而不是设备
PAIR,dir:vendor_apex,unread,unread
PAIR,dir:apexdata,unread,unread
PAIR,dir:staged_sessions,1,unread
ROW,L2-WARN,pair_face_availability_change,目录面 staged_sessions 的可读性由 1 变为 unread：一侧为 unread 说明采集域或策略档位变了，这条变化描述的是观测系统而不是设备
PAIR,dir:supp_patch,unread,unread
PAIR,svc:cmd_apex,denied_caller,denied_selinux
ROW,L2-WARN,pair_service_class_shift,cmd_apex 的服务面分类由 denied_caller 变为 denied_selinux：分流到不同责任方（find 门 vs 调用者校验）说明两侧的采集身份或策略世代不同，两次“失败”不能当同一种失败统计
PAIR,svc:dumpsys_apexservice,not_found,ok
ROW,L2-WARN,pair_service_class_shift,dumpsys_apexservice 的服务面分类由 not_found 变为 ok：分流到不同责任方（find 门 vs 调用者校验）说明两侧的采集身份或策略世代不同，两次“失败”不能当同一种失败统计
DELTA,vendor_select,keys,3,-
ROW,L2-WARN,pair_vendor_select_shift,厂商选择属性族两侧不一致（3 行差异）：本文未定位到这些键的消费方实现，因此只登记痕迹变化，通道判定要等厂商确认后才成立
SUMMARY,high=0,warn=18,gap=0,info=7,ver_delta=2,ver_up=2,ver_down=0,active_flip=0,prefix_shift=0,new=2,gone=3,module_delta_total=9,model_a=Pixel 4a,model_b=DevBoard X
```

四段输出里第一段的 `SUMMARY` 行 `module_delta_total=0`，与第三段的 `module_delta_total=8`、`prefix_shift=1`、`new=1`、`gone=1`，正是第廿一章之三那张输出契约表里"计数行"的实例：`ROW` 行负责逐条叙述，`SUMMARY` 行负责给台账一个可以直接入库的合计；两者对同一件事的计数必须能对上，否则说明有 `ROW` 没被计入合计。第四段虽然在开头就被 `pair_identity_model_diff` 判为"跨设备，本次差分不成立"，末尾仍然给出 `module_delta_total=9`、`new=2`、`gone=3` 的完整合计——这就是第廿三章之二那条规则的落地方式：**脚本负责把事实数出来并把"能不能用"标在旁边，不负责把数出来的结果藏起来。** 台账若因为资格不足而不入库，那是使用侧的决定，不是脚本替使用侧做的删除。
## 廿四、验收：合成档案、覆盖率与三条核对命令

本章是第廿一章第四条实现约定（"每个码位都必须被触发过"）的兑现记录，也是第廿二、廿三章两张码表的证据来源。它测量的对象是**本文档与这两份脚本的一致性**，不是任何真实设备群——这条边界在廿四之八再讲一次，因为这一章的数字最容易被人当成基准抄走。

### 廿四之一 这套验收证明什么，不证明什么

**证明的四件事：**

| # | 命题 | 判定方式 |
| --- | --- | --- |
| P1 | 码表里每个码位都至少真实触发过一次，正文不是照着注释编的 | 定义集与触发集做双向 `comm`，两个方向都必须为空 |
| P2 | 脚本不会输出码表之外的东西 | 同上，`comm -13` 为空 |
| P3 | 两份脚本对设备的读写意图与第廿二章之二的命令表一致 | 替身 `adb` 只回放八类只读命令；任何其它命令在替身里没有分支，直接返回空，因此一旦脚本偷偷发出新命令，对应读数会立刻变成 `unread` 而不是"看起来还行" |
| P4 | 差分页的每一条"这次变化不成立"的判据都能被构造出来 | 22 个配对里包含自比对、反向比对、跨机型比对与一份截断输入 |

**不证明的四件事：**脚本在真实设备上的输出正确性；真实 ROM 的 APEX 布局；任何漏洞是否可修；任何 CVE 与该模块的对应关系。合成档案里的机型串、SPL、`versionCode`、包名全部是构造值，**它们的作用是让判据走到，不是让世界长这样。**

### 廿四之二 验收环境

三条命令即可复现（`WDATA` 是档案根目录，替身 `adb` 靠它定位每台"设备"）：

- 档案目录：`.wdata16/<序列号>/`，一台设备一个目录，目录名就是传给脚本 1 的序列号参数。
- 档案文件与命令的映射由替身完成，命名规则是"命令名 + 路径的斜杠与冒号换成下划线"，因此一份档案里出现的就是这些文件：`props`（`getprop` 全量）、`prop.<属性名>`（单键读取）、`cat_apex_apex-info-list.xml`（主读数面）、`ls_vendor_apex`（`ls /vendor/apex`）、`cmd_apex_getActivePackages`、`dumpsys_apexservice`、`getenforce`、`ps`。档案里缺哪个文件，就等价于真机上那个读数面读不到。
- 运行脚本 1：把 `.wstub16` 放到 `PATH` 最前，执行 `WDATA=<档案根> bash <脚本 1> mm001 > mm001.out 2> mm001.err`。- 运行脚本 2：`bash <脚本 2> mm001.out mm017.out > pair.txt 2> pair.err`，两份输入都是脚本 1 的真实输出文本。

**替身不是模拟器。** 它不解释命令，只查文件；它不会拒绝任何请求，"读不到"完全靠"档案里没有这个文件"来表达。这正是第廿二章之三把 SELinux 拒绝与文件缺失写成两套不同码位的原因——替身只能制造后者，前者必须由档案里预先录好的 `Permission denied` 文本来表达。

### 廿四之三 脚本 1 的十九份档案

十四份与后五份的分工不同：`mm001`–`mm014` 负责把 83 个码位全部走一遍，`mm015`–`mm019` 只为脚本 2 的差分矩阵存在（它们各自触发的码位都是 `mm001` 已触发过的子集，独有码位为 0）。

| 档案 | 构造 | 触发码位 | 独有码位 |
| --- | --- | --- | --- |
| `mm001` | 现代 GMS 发行设备基线：清单齐、`/apex` 不可读、两个服务面被策略挡住 | 25 | — |
| `mm002` | Android 9：APEX 面整体不存在 | 20 | `apex_face_pre_apex`、`cmd_apex_service_not_found` |
| `mm003` | Android 11：`/apex` 与暂存会话可读，且服务面是调用者门而非文件门 | 24 | `apex_catalog_extra`、`apex_dir_vs_list_mismatch`、`cmd_apex_denied_caller`、`staged_session_readable` |
| `mm004` | `userdebug` + Permissive + 厂商选择键有值 | 28 | `apex_entry_field_missing`、`apex_partition_out_of_enum`、`apex_updatable_bogus`、`apex_vendor_select_value_empty`、`apex_vendor_select_value_off`、`apex_vendor_select_value_set`、`apexd_status_readable_permissive`、`build_type_nonuser`、`selinux_permissive` |
| `mm005` | 属性面整体不可读（受限属性对 shell 不出现） | 17 | `apex_vendor_select_unread`、`prop_source_unread` |
| `mm006` | 清单头部不完备 + 厂商包孤儿 | 24 | `dumpsys_apexservice_denied_caller`、`snapshot_header_partial` |
| `mm007` | 全部模块未激活 + 有条目缺 `moduleName` | 23 | `apex_active_zero`、`apex_all_ready_false` |
| `mm008` | 扁平化布局 + 模拟器 + 同名三条记录 | 26 | `apex_dupe_name_over_two`、`apex_dupe_shadow_same_version`、`apex_updatable_false`、`virtual_device` |
| `mm009` | 前缀未映射（既不是 `com.android.` 也不是 `com.google.android.`） | 24 | `apex_name_prefix_unmapped` |
| `mm010` | 宽策略：服务面可读 + 声明文件存在 | 25 | `apexd_status_readable_enforcing` |
| `mm011` | 第五章那 30 个后缀全部到位（清单比对分母齐全） | 21 | `apex_catalog_all_present` |
| `mm012` | 文件非空但零记录：世代不符 | 20 | `dumpsys_apexservice_unread` |
| `mm013` | Android 10：清单口径尚未成型 | 24 | `apex_catalog_pre_mainline_doc` |
| `mm014` | 属性面残缺：头部多项读不到 | 22 | `apexd_status_unread_other`、`release_unread`、`selinux_unread` |
| `mm015` | 逐字段类型差分：前缀迁移、版本升、版本改非数值、出厂位、激活位、分区位、共享库位、模块删除、模块新增 | 26 | — |
| `mm016` | 六个模块同时前进（批量形态） | 25 | — |
| `mm017` | 只前进 SPL，模块 `versionCode` 全不动 | 25 | — |
| `mm018` | 只前进一个模块，SPL 不动（模块级通道形态） | 25 | — |
| `mm019` | 三个模块前进 + SPL 同期前进（部分批量形态） | 25 | — |

`mm001`–`mm014` 十四份档案的**并集恰好是 83 个码位，缺口为空**——这句话是这一节的核心断言：后五份档案不承担任何覆盖责任，把它们去掉，脚本 1 的覆盖率结论仍然成立。

另外派生出一份不是档案的输入：`.wout16/trunc.out`，即 `mm001.out` 去掉 `SUMMARY` 行。它专门用来喂脚本 2 的"输入根本不是一份快照"这条判据。

### 廿四之四 覆盖率结果

本轮实际运行结果（十九份档案各跑一次脚本 1，二十二个配对各跑一次脚本 2；下表数字全部来自这次运行，不是设计值）：

| 指标 | 脚本 1 | 脚本 2 |
| --- | --- | --- |
| `row` 调用点 | 84 | 37 |
| 唯一码位（定义集） | 83 | 37 |
| 触发集大小 | 83 | 37 |
| `comm -23 定义 触发`（从不触发） | 空 | 空 |
| `comm -13 定义 触发`（码表外输出） | 空 | 空 |
| 非空 stderr 的运次 | 0 / 19 | 0 / 22 |

档位分布与第廿九章口径表一致：脚本 1 是 L0 21 / L1 36 / L2 23 / L3 3，脚本 2 是 L0 2 / L1 11 / L2 21 / L3 3，合并后 L0 23 / L1 47 / L2 44 / L3 6，共 120 个唯一码位、121 个调用点。**84 与 83 的差是 `apex_list_parse_mismatch` 的两条独立判据共用一个码位**，理由与后果在第廿九章已完整说明，这里只补一句验收侧的含义：一个码位被两个分支触发时，覆盖率检查只看码位不看分支，所以这一对判据必须靠**两份不同档案**分别打到——本轮由 `mm007`（条目缺 `moduleName`）与 `mm012`（非空但零记录）分别兑现。

### 廿四之五 脚本 2 的二十二个配对

配对矩阵的设计原则是**每条形同"这次比较不成立"的判据都要有一个配对专门为它存在**。表里的"独有码位"指该码位在其余 21 个配对的输出里都不出现，只有这一对能打到。

| 配对 | 考察点 | 码位 | 独有码位 |
| --- | --- | --- | --- |
| `mm001:mm001` | 自比对：确认"无变化"是有资格的记录而非空输出 | 2 | — |
| `mm001:mm002` | 一侧完全没有 APEX 面（跨世代） | 6 | — |
| `mm001:mm003` | 可读面从不可读变可读、清单与目录互相打补 | 11 | — |
| `mm001:mm004` | 跨机型 + 非发行构建 + Permissive + 属性值非布尔 | 16 | `pair_apex_version_unreadable` |
| `mm001:mm007` | 就绪位由真变假 | 9 | `pair_readiness_flip_unready` |
| `mm001:mm008` | `ro.apex.updatable` 由真变假 | 9 | `pair_updatable_true_to_false` |
| `mm001:mm011` | 清单面从无缺到有齐 | 10 | — |
| `mm001:mm012` | 一侧主读数面零记录 | 6 | — |
| `mm001:mm014` | 一侧身份字段整体读不到 | 8 | `pair_identity_unread` |
| `mm001:mm015` | 一次同时打到模块字段差分 13 个码位中的 9 个，外加前缀迁移 | 14 | — |
| `mm001:mm016` | 批量形态（六模块同时前进） | 4 | — |
| `mm001:mm017` | 只有 SPL 前进 | 3 | — |
| `mm001:mm018` | 只有单个模块前进（模块级通道形态） | 4 | — |
| `mm001:mm019` | 部分批量：三模块前进 + SPL 同期前进 | 4 | — |
| `mm001:trunc` | 输入不是脚本 1 的快照 | 3 | `pair_input_not_snapshot` |
| `mm003:mm004` | 跨机型、跨策略档位、跨发行形态 | 16 | — |
| `mm007:mm001` | 就绪位由假变真（与上面反向） | 9 | `pair_readiness_flip_ready` |
| `mm008:mm001` | `ro.apex.updatable` 由假变真（与上面反向） | 9 | `pair_updatable_false_to_true` |
| `mm011:mm001` | 清单面反向漂移 | 10 | — |
| `mm012:mm001` | 零记录侧作为基线 | 6 | — |
| `mm015:mm001` | 字段差分反向：删除变新增 | 14 | — |
| `mm016:mm001` | 批量形态反向 | 4 | — |

反向配对不是凑数：`_ready` 与 `_unready`、`_true_to_false` 与 `_false_to_true` 这四条判据在同一份文档里承担的是**两种相反的运行含义**（一次升级 vs 一次能力回收），把它们合成一个码位会让台账丢掉方向。四组对调配对的实测结果是**只有方向类码位随对调互换，其余码位原样保留**：`mm001:mm011` 与 `mm011:mm001` 各有 10 个码位、其中 7 个相同，差出的 3 个正好是 `pair_apex_newly_present`/`pair_apex_removed`、`pair_apex_version_up`/`pair_apex_version_down`、`pair_catalog_newly_present`/`pair_catalog_newly_absent` 这三对；`mm001:mm015` 与 `mm015:mm001` 各有 14 个、12 个相同，差出 `active_flip_off`/`active_flip_on` 与 `version_up`/`version_down`；`mm001:mm016` 与 `mm016:mm001` 只差 `version_up`/`version_down` 一对；`mm001:mm012` 与 `mm012:mm001` 完全同集，因为那一侧是零记录、没有任何可换方向的事实。**"对调只换方向码"就是脚本 2 没有把基线侧与对照侧当成有先后之分的角色来处理的证据**，而 `mm012` 那一对的空差集是这条性质的退化情形检查。

### 廿四之六 三条核对命令

第廿九章给出的三条命令是这一章的常驻部分，**改动两份脚本的任何一行判据后必须重跑**：

1. 定义集：`grep -o 'row L[0-9]-[A-Z]* [a-z_0-9]*' <脚本> | awk '{print $3}' | sort -u`
2. 触发集：把所有样例输出串起来 `awk -F, '$1=="ROW"{print $3}' | sort -u`
3. 双向 `comm`：`comm -23 定义 触发` 与 `comm -13 定义 触发` **同时为空**

第 22 个配对跑完后必须再看一眼 stderr：验收里出现过的两次真实缺陷，都是**判据能走到、但走到的那条支路是错的**，`comm` 两个方向都发现不了，只有把输出逐行读一遍才能发现。

### 廿四之七 本轮验收抓出的两个自身缺陷

这两条不是设备的缺陷，是本报告与脚本的缺陷，按同样的口径记在这里：

| # | 症状 | 成因 | 修法 | 副作用检查 |
| --- | --- | --- | --- | --- |
| 1 | 每一档**格式正确**的档案都被判成 `L3-HIGH apex_list_parse_mismatch`，并多出一条 `APEX,none,…` 记录；`COUNT,apex_record_no_name` 恒等于 1 | 剥根标签用的 `sed` 表达式写成 `[^>]*?>`。POSIX BRE 里 `?` 是字面字符，这个模式永远匹配不上 `<apex-info-list>`，于是根标签本身被当成第 16 条记录，而它没有 `moduleName` | 两处剥标签表达式改为 `[^>]*>` | 修完后 `mm001` 稳定输出 15 条 `APEX` 行、`apex_list_parse_mismatch` 只在 `mm007`/`mm012` 触发 |
| 2 | `pair_identity_unread` 从不触发，覆盖率是 37 定义 / 36 触发 | 脚本 1 对读不到的字段写的是字面 `none`/`unread` 而不是空串，脚本 2 用 `[ -z … ]` 判缺失，永远不成立。同一处缺陷还有第二种后果：一个从 `2025-08-05` 变成 `none` 的 SPL 会被当成"真的换了一个值"，即**把缺失读成迁移** | 取值统一走一个归一函数：`none` 与 `unread` 一律折算成"没有该值"，再进入比较 | 覆盖率升到 37 / 37，两个方向仍为空；`mm001:mm014` 成为该码位的唯一触发源 |

第二条值得单独记一句：**"缺失"与"变化"在设备侧文本上无法区分时，差分脚本必须偏向"缺失"。** 把 `none` 当成一个合法的新值，会让台账写出一条根本不存在的迁移；反过来把一次真实迁移当成缺失，只是少记一条——两种误判的代价不对称。这也是第廿三章之二把"身份先于内容"列为输入契约第一条的原因。

### 廿四之八 边界与使用规则

- 这一章所有数字（19 份档案、22 个配对、84/37 个调用点、120 个码位、L3 6 / L2 44 / L1 47 / L0 23）**描述的是本文档的合成集覆盖率**，不得作为任何真实设备群的分布基准引用。第廿六章 Q9 与第廿八章第 14 项从两个方向说过同一件事。
- 合成档案集里，`apex_updatable_false`、`pair_spl_shift_no_module_change` 这类码位各只有 1–2 个触发源；真实分布必须自采。
- 档案的构造方式（从 `mm001` 派生、逐字段 `sed` 扰动）没有随附在本报告里，因为它依赖第廿二章那张"文件↔读数面"映射；**读者要复现覆盖率，需要先按该表自建等价档案**。这一节只承诺结果可复现，不承诺档案可下载。
- 覆盖率检查不校验**文案**。脚本 2 的一处判据说明原本引用了一个未经核验的源码行号，是在读输出样例时发现的，随后换成第廿三章之三那两行的既有锚点。**报告里每一个源码行号与文档句子都必须能在第卅章的取回记录里找到原句**，这是比覆盖率更强的一条验收纪律。

---
## 廿五、十二个常见误区

这一章把前廿四章里**最容易在工单、月报和外部沟通里被写错**的十二种读法列成一张表。左边一句是常见问法，中间一句是它为什么自然，右边一句是本文给的读法和它的出处。

| # | 误区 | 为什么会这么读 | 正确读法（对应章节） |
| --- | --- | --- | --- |
| 1 | **"这台设备有没有 `com.android.media`？"** 当成一个能回答的是非题 | 日常清点只需要"有/没有"两个字 | 设备侧至少有三个来源会给出三个不同答案：清单世代、`/apex` 挂载集合、`/vendor/apex` 目录（第五、九章）。管理单位是"来源 × 世代 × 读数面 × 判据"四元组，脚本 1 把这三个来源的分歧显式写成 `apex_dir_vs_list_mismatch` 与 `apex_dir_list_cross_ok` 两条而不是一条"有/没有"（廿二之三） |
| 2 | 看到 versionCode 前进，就写**"对应 CVE 已修"** | 版本前进是差克里最像"更新发生了"的信号 | `pair_apex_version_up` 的天花板是"待验证"（廿一之二）。`name` + `version` 定义的是模块身份，不是补丁集合（八之一）；容器哈希树证明的是完整性而不是包含哪个修复（七之四）；公告表给出的是候选而不是答案（十六之一） |
| 3 | SPL 前进了一个月，就写**"Mainline 模块侧补丁已到达"** | SPL 是运维手里唯一每个月都会动的数字 | 第十二章之三给了三条公开证据说明 SPL 不携带模块版本信息。脚本 2 专门为此设了 `pair_spl_shift_no_module_change`，并把它的等级放在 L2-WARN 而不是 L1——因为这一行读出来的**是一个需要解释的空缺**，不是好消息 |
| 4 | `com.android.wifi` 消失 + `com.google.android.wifi` 出现 = **两次模块事件** | 两个完整包名在差分输出里就是两行 | 前缀由**签名主体**决定而不是由模块决定（六之一）。脚本 2 因此把它们配成一条 `pair_name_prefix_shift`（L2-WARN，一条），而不是让 L2 的"移除"和 L1 的"新增"各占一行——台账记两条会直接虚增一次变更（廿三之四之一） |
| 5 | `ls /apex` 读不到，所以**这台设备上没装模块** | 命令失败最像"目录是空的" | 标准策略下 `shell.te` 对 `apex_mnt_dir:dir` 没有授权，这一条失败是**设计如此**（十三之一）。脚本给它 L1-INFO 而不是 L0 或 L2，正是为了拦住这种读法；反过来，"清单里承诺可读却没读到"才是缺口（廿一之一的 L1/L0 分界） |
| 6 | 第五章清单里没有某个后缀，就写**该设备不具备这项能力** | 清单读起来像能力矩阵 | 缺席有三种在设备侧**不可区分**的情形：设备能力、厂商选择、读数失败（五之三）。`apex_catalog_absent_list` 的释义行把三种并列列出，就是为了让抄写的人无法只保留其中一种 |
| 7 | 内置版本（`preinstalledModulePath` 指向的那个包）就是**已修基线** | 出厂镜像看起来代表厂商的最终决定 | 内置版本是**兜底**，不是结论（九之一）。`isFactory` 与 `preinstalledModulePath` 描述的是承载位置；一个内置版本可以同时是缺陷的载体，也可以比激活版本更新（十之二与 `apex_installed_vs_active_gap`） |
| 8 | `isActive=false` 就是**模块被卸载了** | 布尔位读起来像存在性 | 文档口径写得很窄：安装与激活是两个动作，且激活要等到重启（十之一）。`isActive=false` 同时兼容"刚安装未激活""会话进行中""被更高版本遮蔽"三种状态，脚本 2 用 `pair_apex_active_flip_off` 的释义行明确要求先排除对照侧落在激活流程中途（廿三之三） |
| 9 | `adb root` 之后读到的，就是**标准口径** | 权限变大了，读数看起来更"真" | 服务面有两道**彼此独立**的门：`find` 门与调用者校验（十三之三）。换采集域之后，同一面失败的含义会反转；脚本 1 用 `apex_dir_unexpected_readable` 与 `cmd_apex_readable_unexpected` 记录反转，脚本 2 用 `pair_face_availability_change` 与 `pair_service_class_shift` 记录"变化的是观测系统而不是设备"。用扩展域采到的快照，不得与标准域快照做差分 |
| 10 | 两台不同机型的同名模块版本不同，就写**其中一台落后** | 版本号可比，是工程师的本能 | 第二十之四给了禁止跨设备比较的理由：机型、构建档位与策略世代都不同，两侧读数不是同一条时间线上的两个点。脚本 2 的身份门在机型不同（`pair_identity_model_diff`）和序列号不同（`pair_identity_serial_diff`）时都出 L2-WARN，意思就是"这次差分不是一条变更"（廿三之二） |
| 11 | 认为 Mainline 公告表**能覆盖当月全部 CVE** | 有模块归属的条目容易被当成总体 | 十八、十六之三给了两类在构造上就没有模块载体的缺陷（器件固件与内核），K10 因此被设计成"数值越高越正常"的指标（廿一之四之三）。模块口径覆盖率应当是被测量的量，而不是被假定的量 |
| 12 | 把 `provideSharedApexLibs` 与 AIDL 的 `hasClassPathJars` **当一个字段两边写** | 名字都指向"这个模块给别人提供代码" | 模块身份在三个面上字段名并不互相兼容（十四章），classpath 是运行时装配面而不是共享库声明面（十五之三）。脚本 2 的 `pair_classpath_flag_shift` 把"两者不是同一字段、两侧读数不可互相替代"直接写进释义行，防止抄录时合并 |

## 廿六、常见问题

**Q1：我怎么知道一个 CVE 是不是该归到某个模块？**
先走公告而不是设备。第一，把公告条目的 `Subcomponent` 词表与第五章清单做映射（六之二），映射失败的四种模式在六之三列表；第二，映射成功只说明"公告认为它有模块载体"，不说明这台设备上的那个载体是什么版本；第三，设备侧只能提供"这台机器此刻的模块姿态"（脚本 1），把两者连起来需要的是一次带时间的采集，而不是一个布尔判断。**K3（模块名归一率）就是为这条链设立的，它衡量的是映射与归一本身，不是修复情况。**

**Q2：能不能只用脚本 1 的快照做资产清点？**
可以，且这是它唯一被设计来做的事。但清点的口径要按快照的读法约束：模块数以 `COUNT,apex_total` 为准而不是挂载点数（第九、十三章），前缀分布要按后缀名归一（六之一），清单覆盖率要分母取第五章的 30 项而不是当月清单（五之二）。快照里的 `SUMMARY` 行已经把这三项做成字段，直接引用它们比自己数 `ROW` 行安全。

**Q3：为什么脚本 2 不直接连两台设备各采一次？**
非目标。其一，脚本 2 的输入是**文本**，这让它可以在没有设备的环境里被第三方复算——本报告的全部样例输出都来自合成档案，读者不需要真机就能验证判级逻辑。其二，一旦脚本自己发起采集，两侧就可能落在不同的 SELinux 档位、不同的用户下、不同的时刻，"配对资格"会被采集过程本身污染；把它做成显式前置门（先判身份，再判内容）才是可控的。**结论：采集要有两次独立记录，比对是另一件事。**

**Q4：`apex.updatable=false` 是不是意味着这台设备完全没有快修？**
不是。`ro.apex.updatable=false` 关掉的是**模块级通道**这一档位（十二之五），但整机 OTA 仍然存在，厂商自有的更新机制（第十二章之四的补充声明、`/vendor/apex` 下的厂商包）也不由这个属性决定。脚本 1 在这一行同时会产出通道面与厂商选择面的独立读数，就是为了防止把"模块通道没有"写成"这台机器不会更新"。台账上正确的写法是：这条缺陷的到达单位退回整机 OTA，且要另查厂商通道。

**Q5：`cmd apex` 和 `dumpsys apexservice` 都失败，算一种失败还是两种？**
两种，而且这是脚本 1 特意做的分流。服务面有两道独立的门（十三之三）：`cmd` 走 servicemanager 的 `find`，`shell.te:384` 把 `apex_service` 从可访问集合里排除；`dumpsys` 走服务自身的 `shellCommand`，其失败原因在调用方校验而不是查找。因此两者的失败**责任方不同**（策略域 vs 服务实现），统计时合并会把两类工单派给同一个团队。脚本 2 的 `pair_service_class_shift` 就是防止两侧的失败分类被混着算。

**Q6：两份快照一模一样，是不是可以写"本月无变化、无需复核"？**
不能这么写。`pair_no_change_within_readable` 只在两侧主读数面都可读、且逐字段无差异时才成立，它是一条**有资格**的"无变化"记录，可以作为下一次差分的基线；但它不等价于"没有待修问题"，因为公告侧的变化根本不走这个面。K9 的反例（用"两次快照差异为 0"当作"无需复核"）就是这一条。要闭合"无变化"，需要同时有 K1 达标（两侧主面都读全）与当月公告映射记录。

**Q7：为什么 `pair_apex_removed` 不是 L3-HIGH？模块消失难道不是最严重的信号？**
因为它**推不出**严重结论。一个模块从清单里消失，可以是它没被激活、可以是采集时机、可以是清单世代换代之后的重命名（那已经被 `pair_name_prefix_shift` 吃掉）。L3-HIGH 在本报告的定义是"这条归属结论在本机上不成立，或这条通道在本机上不存在"（廿一之一），需要的是直接证据而不是推断。"消失"的正确处置是限期核查加第二侧证据，所以落 L2-WARN。反过来说，如果同一次差分里 `apex.all.ready` 还由 `true` 翻到了 `false`，那"消失"这一整类结论会被 `pair_readiness_flip_unready` 直接作废——脚本把这条优先级写进了释义。

**Q8：能不能把脚本 1 的输出喂给别的比对工具？**
不建议。输出契约（廿一之三）只保证前缀与字段顺序稳定，不保证语义对第三方工具友好：例如 `APEX` 行的 `none` 同时兼容"字段缺失"与"解析失败"，比对工具会把两种情形合并统计。要复用请先按本报告的两条不变量自检：每个 `ROW` 的码位必须来自码位表；`SUMMARY` 必须存在。**第二条正是 `pair_input_not_snapshot` 存在的原因。**

**Q9：第廿四章的验收数字（19 份档案、22 个配对、120 个码位）能不能当基准用？**
不能。那一章的数字描述的是**本文合成档案集的覆盖率**，不是任何真实设备群上的分布。特别是 `apex_updatable_false`、`pair_spl_shift_no_module_change` 这类码位，在合成集里各只有 1–2 个触发源；真实设备群上的分布必须自己采。**引用第廿四章时必须带上那句"合成档案，非设备样本"。**

**Q10：这套方法对非 GMS 设备、对厂商自带模块还成立吗？**
部分成立。前缀映射（六之一）与清单比对（第五章）的口径会失效，因为厂商包不在公开清单里；`/vendor/apex` 目录面与厂商选择属性族（十二之二）反而变成主要读数面。脚本 1 对这种情况有专门码位（`apex_name_prefix_unmapped`、`vendor_apex_orphan_file`），它们的等级都停在 L2 而不是 L3，因为本文**没有厂商侧通道的公开实现依据**——第廿八章的边界清单里明确登记了这一点。
## 廿七、趋势与分角色结论

### 廿七之一 六个可以直接从公开文档读出的趋势

这一节的每一条都对应参考资料里的逐字句，不做外推。趋势的意思是"文档口径本身正在朝这个方向移动"，不是"设备会怎样"。

**一，清单是一个移动目标，而且新增可以是"可选的"。** 公开页把 NFC Services 描述为 `an optional updatable Mainline module released in Android 16`。这句话对本报告 method 层面的影响比它看起来大：如果"存在一个模块"本身可以是可选的，那么第五章清单比对得到的"缺席"就更不能读成"能力缺失"——第五之三给的三种情形里，"厂商选择不启用"现在有了文档层面的合法实例。**任何把清单当固定分母的台账，都会在一个大版本内失真。**

**二，模块通道的适用版本范围有明确终点。** 文档原文：`Mainline support for Android 11 and lower is concluded as of Q4 of 2025.` 这意味着对存量 Android 11 及以下设备，模块口径不再是可派的归属单位；本报告第十三章的"预期失败表"（Android 10 才有 APEX 格式，Android 9 及以前根本没有这一面）与脚本 1 的 SDK 门（`release_below10`）正是为这条边界准备的。**台账上应当区分"这台设备不适用模块口径"与"这台设备采集失败"，文档给出了前者的合法写法。**

**三，可更新性不是单调的：有模块被明确退回不可更新。** 文档对 ART 的表述是 `In Android 11, ART and libcore are packaged as non-updateable APEX.` 一个进了 Mainline 名单的组件可以变成非可更新形态。因此 `ro.apex.updatable` 或清单上的 Type 列都不能回答"这个模块现在能不能独立更新"，只有设备侧的 `isFactory` / `preinstalledModulePath` 与激活集合能给当日读数。**归属单位要按"当日读数"而不是"历史身份"写。**

**四，容器形态仍在换代，且旧形态被明确废弃。** `In Android 15, Flattened APEX is deprecated.`、`Flattened APEXes can be updated only via a regular OTA.`、`Mixing flattened and non-flattened APEXes in a device is NOT supported.` 这三句连起来给了一个可操作的推法：**在同一台设备上看到"清单里有模块、但没有对应挂载与激活记录"时，扁平化装配是必须排除的解释之一**，而它恰好完全落在模块口径之外——这类条目应该按第十八章的无模块载体类处理，而不是按第十九章 M-AT4 的"未激活"处理。

**五，文档自己承认模块通道的安全动机，也承认它承载的是证书这类数据。** Media 模块页写 `Media components are a frequent target of attacks (many recent security bugs are media related)`；Conscrypt 页写 `The Conscrypt module accelerates security improvements and improves device security without relying on OTA updates.` 与 `it's possible to update certificates more frequently using Mainline train updates.` 这两句给本报告两条纪律提供了正面依据：**其一**，模块通道存在的首要理由是缩短修复到达时间，所以"模块版本前进"确实是有效信号——但**其二**，同一个通道也承载数据（证书包），所以版本前进同样可能是纯数据刷新。**这正是 `pair_apex_version_up` 只能停在 L1-INFO、结论上限是"待验证"的原因。**

**六，模块边界正在被 CTS 与稳定接口固化，代价是设备侧可见符号集合变成条件量。** 文档口径是模块 `use only the SDK and System APIs guaranteed by the Compatibility Test Suite (CTS)`、只通过稳定 C API 或 Stable AIDL 交互，且 permissioncontroller 页明确 `no @hide API usage`；Wi-Fi 页进一步写明 `Symbols ... annotated with @hide in the Wi-Fi module are not part of its public API surface and cannot be accessed on devices with the module installed.` 也就是说，**"某个类/方法在这台设备上能不能访问"取决于模块装没装**——这是第十五章 contract 面在设备侧的可观测后果，也是第十七章 M-AT6（契约表面缺陷）不能只按应用版本号派工的原因。

### 廿七之二 分角色结论

**平台 / ROM 集成负责人**：把 `ro.apex.updatable` 与厂商选择属性族的读数纳入出厂基线并留档，通道档位一旦变化就是一次需要评审的改动，因为它会让先前所有"模块侧已修"的结论换归属单位（K5）。装配侧变更（第十一章的 `PRODUCT_APEX_SYSTEM_SERVER_JARS` / `PRODUCT_APEX_BOOT_JARS` / `PRODUCT_DEFAULT_APEX_PAYLOAD_TYPE` 与模块增删）要在集成阶段就同时更新公告映射词典（六之二），否则映射失败会在公告发布当天集中出现。

**PSIRT / 补丁治理**：月度派工的输入是**两张公告表 + 一次设备侧姿态快照**，不是任何一张单独来源（第十九章的十三组读数就是这条的实证）。派工时写"模块名 + 公告给出的候选版本 + 快照里的激活版本"三段，缺第三段就写"待验证"，不要写"已修"（廿一之二的五档结论表）。K2 与 K3 要在派工前先达标。

**设备运维 / MDM**：固定采集域、固定命令集、固定脚本版本，然后按差分而不是绝对读数运营（K9）。换采集域（例如 `adb root`）之前先做一次双域对照，并把差异登记为 `pair_face_availability_change`，否则前后两段台账不可比——**运维最容易造成的失真不是漏采，而是悄悄换了采法。**

**应用开发者**：需要判断"某个 API 在装了模块的设备上行为不同"时，按第十五章的 contract 面而不是版本号排查；对 `@hide` 符号，文档的口径是**装了模块的设备上不能访问**，这属于预期行为而不是设备缺陷。SDK extension 是另一条版本轴（十五之四），不要与 APEX 的 versionCode 混在同一个兼容性判断里。

**取证 / 合规**：单台设备的快照只能证明"当日姿态"，不能证明更新历史。要给出时间线证据需要两份带时间戳的快照 + 外部公告记录（M5），并注意 `lastUpdateMillis` 与 `apexd.changed_active_apexes` 这类字段本文没有可引用的实现依据支撑其稳定性（第廿八章）。**任何"这台设备在某日收到了某修复"的断言，在设备侧证据链上目前无法闭合。**

## 廿八、未公开文档化 / 本次未逐字核验的边界

这一节列出报告里**用到但没能逐字核验**、或**公开文档本身没有定义**的东西。它存在的目的不是削弱前廿七章，而是让引用者知道每条结论的天花板在哪里。凡列在这里的项，正文与脚本都不允许把它当已确认事实来写。

| # | 边界项 | 现状 | 本文的处理 |
| --- | --- | --- | --- |
| 1 | `minSupportedVersion` 的强制实现 | 在 `apexd`、`apex_manifest.proto`、`ApexManager` 中未见强制实现路径；能逐字引的只有文档规则 `same package name and a greater than or equal to version code` 与 `apexd.cpp:3406-3407` 的 `downgrade … not a supported case` | 正文不声称"目标版本被运行时校验"；脚本不对降级做任何"被阻止"的断言，只按 `pair_apex_version_down` 记为需要人工确认 |
| 2 | `lastUpdateMillis` 的刷新语义 | 能引的只有 `apexd.cpp:3257-3259` 写字段的顺序（含 `mtime`），没有"何时刷新"的文档句 | 脚本 1 原样输出该列；**脚本 2 明确不比对它**（廿三之一） |
| 3 | `apex.all.ready` 的属性条目 | `property_contexts` 里没有独立条目，靠 `apex.` 前缀命中 `apex_ready_prop`（`:364`）；`apex_all_ready` 这种写法不存在 | 正文只按前缀命中表述，不声称它有独立 SELinux 标签 |
| 4 | 厂商选择属性族的消费方 | `persist.vendor.apex.` / `ro.boot.vendor.apex.` → `apexd_select_prop`（`:356-357`）是标签层面的事实；本文未定位到读取这些键并据此改变行为的实现 | 脚本只记痕迹（`apex_vendor_select_*`、`pair_vendor_select_shift`），等级最高 L2，且释义里写明"通道判定要等厂商确认" |
| 5 | `apex-info-list.xml` 的三个标签变体 | `private/file_contexts:62` 的正则给出 `.bootstrap-` / `.default-` / 无三种，但 `apex_constants.h` 只有一个路径常量 | 正文表述为"标签三变体、常量一个"，不声称设备上一定能列出这三个文件 |
| 6 | 批量差分的阈值 | `nver ≥ 5` 判"批量形态"是**本文自设阈值**，公开文档没有"多少个模块同时变化算一次 OTA"的口径 | 在脚本注释与本节同时登记；换阈值必须重跑第廿四章的配对矩阵 |
| 7 | `Subcomponent` → 模块名的映射词典 | 公告 `Subcomponent` 词表与第五章清单之间没有官方映射表 | 六之二给的是本文建立的映射流程与四种失败模式，映射结果属本文产物，不是文档事实 |
| 8 | STS（Mainline 相关测试框架）的目的散文 | `reference/sts/classes` 只有类索引与 `HostsideMainlineModuleDetector` 类名，**没有**说明 STS 目的的散文页 | 正文不声称"文档规定 STS 用于 X"；只在需要时引类名存在这一事实 |
| 9 | bug-protection / 模块相关披露政策文句 | `/docs/security/bug-protection` 等路径 404；`asb-overview` 无 mainline/module 措辞 | 本文不引用任何此类政策句；第十六章的能/不能只建立在公告表实际列头上 |
| 10 | 两个月度公告页的正文 | `2025-07-01`、`2025-10-01` HTTP 200 但抓取正文 0 条 CVE（客户端渲染）；`sitemap_0..4_of_5.xml` 返回 500，无法全站枚举 | 第十九章的十三组读数明确排除了这两页，并把"抓取失败 ≠ 当月无条目"写进读数规则 |
| 11 | 压缩 APEX 与 `erofs` 的设备侧可分辨性 | 文档给出压缩条目清单与 `erofs_file_backed_mount` 配置项（`apexd.config.…`，`apexd_private_prop`），但 shell 域读不到该配置 | 脚本不推断压缩形态；第七章只在容器结构层描述 |
| 12 | 停用 / 卸载的设备侧终态 | 文档语义只到 `DeactivatePackage()` 的卸载动作与"重启后激活"；本文未找到"已卸载模块在清单中如何表示"的定义 | 第十章与第十七章把"卸载"标为设备侧不可判定，脚本相应只出 `pair_apex_removed`（L2）而不出"已卸载" |
| 13 | `hasClassPathJars` 与 `provideSharedApexLibs` 的关系 | 两个字段在不同面（AIDL `ApexInfo` 与清单 XML）上分别可见，但本文未找到把它们对应起来的公开实现依据 | 脚本 2 的释义行明确写"不是同一字段、两侧读数不可互相替代"；正文不建立等价关系 |
| 14 | 真实设备群上的码位分布 | 第廿四章的触发画像全部来自合成档案；`apex_updatable_false`、`pair_spl_shift_no_module_change` 在合成集里各只有 1–2 个触发源 | 正文与 FAQ 均声明"不得当基准"，并要求真实分布自采 |

三条使用规则：

1. **凡引用本报告的判级或码位，先查本表**：落在 1–14 任一条上的结论必须带上"本文自设"或"设备侧不可判定"的限定语。
2. **本表是活的**：如果后续拿到某条边界的公开实现依据（尤其是第 1、2、4、6、13 条），应当先更新本表与对应码位释义，再改脚本；顺序反了会让脚本与报告出现无法解释的分叉。
3. **沉默不是结论**：本表没有列出的东西，不代表已核验——第廿九章的术语表会把"已逐字核验"与"本文推断"分栏，那张表与本表配套使用。
## 廿九、术语对照与计数口径

### 廿九之一 术语表：本文用法与是否逐字核验

"口径"一列说明这个词在本文里被**限定**成什么；"来源"一列标出它是否逐字核验过（**文档**=source.android.com 页；**源码**=AOSP 文件与行号；**推断**=本文建立、文档没有给的规则）。凡标"推断"的词，引用时必须带这条限定。

| 术语 | 本文口径 | 来源 |
| --- | --- | --- |
| Mainline | `modularizes some Android system components and enables them to be updated outside of the normal Android release cycle` 的那个机制；本文只在"归属单位"意义上使用 | 文档 |
| APEX | Android 10 引入的容器格式；本文所有设备侧字段名都取自它的清单与常量定义 | 文档 / 源码 |
| 模块 | 一次归属判断的单位，由"包名 + 版本 + 承载分区 + 激活态"四元组描述，而不是由代码目录描述 | 推断（本文定义） |
| 主读数面 | `/apex/apex-info-list.xml` 一个文件；本文其它任何面都不足以单独支撑模块级结论 | 源码（`apex_constants.h:78`，唯一路径常量） |
| 激活（active） | `isActive=true`，且文档时序上"要等到重启"；本文不把它等同于"已安装" | 文档 / 源码 |
| 出厂内置（factory） | XML 的 `isFactory` 与 `preinstalledModulePath` 表达的承载位置；本文不把它当作版本优劣或修复状态 | 源码（`ApexInfoList.xsd:40,43`、`apexd.cpp:3257-3259`） |
| 遮蔽（shadow） | 同名多条记录共存时高版本生效；文档句 `the newly installed version shadows the version already present in the built-in partition` | 文档 |
| 前缀迁移 | 剥掉 `com.android.` / `com.google.android.` 之后同名、前缀不同的一对包名；本文判定为同一模块的两种签名形态 | 文档给出前缀规则，"配对"这一步是推断 |
| 清单（第五章） | 公开 Available modules 表逐字出现的 30 个名；本文的分母固定取这 30 项而不是当月页 | 文档 |
| 通道档位 | `ro.apex.updatable` 的布尔值（`exact bool`，exported 因此可读）；本文用它判"模块级通道是否存在"，不判"是否已修" | 文档 / 源码（`property_contexts:1505`） |
| 厂商选择痕迹 | `persist.vendor.apex.` / `ro.boot.vendor.apex.` 一族键的存在与取值；本文只登记痕迹，不断言语义 | 源码（`property_contexts:356-357`）+ 推断（语义未知，见第廿八章边界 4） |
| 补充补丁声明 | `/etc/security/supplemental_security_patches.xml`，OEM 自愿报告超出 SPL 的修复 | 文档 |
| SPL | 安全补丁级别；本文按第廿八章纪律使用：SPL 前进不携带模块版本信息 | 文档 |
| 预期失败 | 标准 `shell` 域下策略本就不授权的读数（`/apex` 列表、`apexd.status`、服务面、`/data/misc/apexdata`、会话目录） | 源码（`shell.te` 逐行枚举 + 无 `apex_mnt_dir:dir` 授权） |
| 缺口 | 策略承诺可读却没读到；与"预期失败"分属不同等级 | 推断（本文分级） |
| 观测面漂移 | 两份快照之间**采集能力**本身的变化；本文把它和设备状态变化分成两种行 | 推断（本文概念） |
| 有效无变化 | 两侧主读数面都可读且逐字段无差异；本文承认这不等价于"没有待修问题" | 推断（本文定义） |
| 批量形态 | versionCode 变化模块数 ≥ 5 的差分形状；阈值是本文自设 | 推断（见第廿八章边界 6） |

### 廿九之二 同一族模块的四套名字

本篇最容易出错的地方不是判级，是**名字**。同样是"这个模块"，四个来源给出的字符串互不完全重合：

| 层 | 形态 | 例子 | 出处 |
| --- | --- | --- | --- |
| 公开清单 | `com.android.X` / `com.google.android.X`，随签名主体变 | `com.android.wifi`、`com.google.android.adservices` | 文档（Available modules 表 + 前缀规则） |
| 设备清单 XML | 与上一层同，但同一文件里可同时存在两条同名记录 | `com.google.android.adservices` ×2 | 源码（`ApexInfoList.xsd:38`） |
| 公告表 | 人读的 `Subcomponent` 词，**从不**出现包名 | `WiFi`、`MediaProvider`、`Permission Controller` | 文档（实测三页 `com.android.*` 命中 0 次） |
| SELinux 标签 | 策略里第三套命名，有四个点名子目录名不在清单 30 项之内 | `com.android.compos`、`com.android.virt`、`com.android.permission` | 源码（`file_contexts:680-687`） |

四条使用规则由此得出，它们也是脚本 1 相关码位的设计依据：

1. 跨机型比对必须按**剥前缀的后缀名**归一，否则 GMS 与 AOSP 签名的两台设备看起来"模块完全不同"（`apex_name_prefix_mix`、K3）。
2. 公告词表到包名的映射是**本文建立的词典**，不是文档给的对照表；映射失败的四种模式在六之三分列，任何一次失败都要留名（第廿八章边界 7）。
3. 策略标签里出现的名字**不能**反推"文档漏写了一个模块"；本文只登记"同一模块族存在四套互不完全重合的名字"这一事实。
4. 未映射前缀的包名（`com.example.oem.radio` 这类）不是"厂商模块"的证明，只是"第六章规则覆盖不到"（`apex_name_prefix_unmapped`，L2 而不是 L3）。

### 廿九之三 码位计数口径

三份数字，各自对应一种提取方式，都对，不能混用：

| 口径 | 脚本 1 | 脚本 2 | 合计 |
| --- | --- | --- | --- |
| `row` 调用点 | 84 | 37 | 121 |
| **唯一码位** | **83** | **37** | **120** |
| 等级分布（按唯一码位） | L0 21 / L1 36 / L2 23 / L3 3 | L0 2 / L1 11 / L2 21 / L3 3 | L0 23 / L1 47 / L2 44 / L3 6 |

**84 与 83 的差是同一个码位、两个触发点。** `apex_list_parse_mismatch` 在脚本 1 里有两条独立判据：一条是"XML 非空但解析出 0 条记录"（schema 的 `minOccurs=1`，`ApexInfoList.xsd:21`），另一条是"某条记录不含 `moduleName`"（`xsd:38` 声明 `use=required`）。两种情形的**后果相同**——主读数面对那条记录失去归属能力——所以本文有意让它们共用一个码，使下游统计不必区分"整份文件坏了"和"某一条坏了"；区分它们的是 `COUNT,apex_total` 与 `COUNT,apex_record_no_name` 两个计数行，而不是等级。两处调用的等级都是 L3-HIGH，因此不存在跨等级现象。

三条核对命令（第廿四章把它们并入验收脚本，改动脚本时必须重跑）：

1. `grep -o 'row L[0-9]-[A-Z]* [a-z_0-9]*' <脚本> | awk '{print $3}' | sort -u` —— 定义集。
2. 全部样例输出里 `awk -F, '$1=="ROW"{print $3}' | sort -u` —— 触发集。
3. `comm -23 定义 触发` 为空 **且** `comm -13 定义 触发` 为空 —— 既没有从不触发的码位，也没有码表之外的输出。

第 3 条两个方向都要查，是因为只查前者会漏掉"输出了码表里没有的码"这种更危险的漂移：它意味着文档与脚本已经分叉，而分叉通常是脚本单方面改了判级。

## 三十、参考资料

以下条目为本次核验实际取得逐字内容的公开来源，编号供正文引用；引文为原文（英文）摘录，破折号后为中文口径说明。**AOSP 源码条目按 `?format=TEXT` 取回后本地 base64 解码逐行核对，行号对应下列 ref；文档条目中把被换行拆开的句子按阅读顺序合并了空白，不改动词序与用词。**

### 一、Mainline 与模块清单

1. source.android.com《Modular system (Mainline)》："Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle."；"Mainline converts selected system components into modules."；"Some Mainline modules use the APEX container format (introduced in Android 10) and some use the APK format." — 本篇三个核心词的官方定义句。第一、二、三章。https://source.android.com/docs/core/ota/modular-system
2. 同页："End-user devices can receive Mainline updates from the Android Google Play system update feature (which runs on the Google Play Store infrastructure) or through a partner-provided over-the-air (OTA) mechanism." — 通道只有两条，且第二条由伙伴掌控。第十二章之一。https://source.android.com/docs/core/ota/modular-system
3. 同页："On Google Mobile Services devices, Mainline module packages are signed by Google and their names use the `com.google.android.*` preface. For devices signed with AOSP keys, package names are prefaced with `com.android.*`." — 前缀由签名主体决定而不是由模块决定，是第廿三章前缀迁移配对的唯一文档依据。第六章之一。https://source.android.com/docs/core/ota/modular-system
4. 同页："The module package installs and rolls back atomically; either all modules that need to be updated are updated or none are updated."；"Module updates don't introduce new APIs. They use only the SDK and System APIs guaranteed by the Compatibility Test Suite (CTS), communicate only with each other, and use only stable C API or stable AIDL interfaces."；"enables system components to be updated with critical bug fixes and other improvements as needed, without affecting lower-level vendor implementations or higher-level apps and services." — 原子性与契约约束；第十五章 contract 面与第十七章 M-AT6 的边界来源。https://source.android.com/docs/core/ota/modular-system
5. 同页："Note: Mainline support for Android 11 and lower is concluded as of Q4 of 2025." — 模块口径的适用版本范围有明确终点。第十三章 SDK 门、第廿七章趋势二。https://source.android.com/docs/core/ota/modular-system
6. 同页 Available modules 表逐字出现的 30 个名（含 Type=APEX/APK 与 Release introduced 列）：`com.android.adbd`、`com.google.android.adservices`、`com.google.android.healthfitness`、`com.android.appsearch`、`com.android.art`、`com.google.android.bt`、`com.android.cellbroadcast`、`com.android.configinfrastructure`、`com.android.conscrypt`、`com.android.scheduling`、`com.android.resolv`、`com.android.documentsui`、`com.android.ext.services`、`com.android.ipsec`、`com.android.media`、`com.android.media.swcodec`、`com.android.mediaprovider`、`com.android.modulemetadata`、`com.android.networkstack`、`com.android.neuralnetworks`、`com.google.android.ondevicepersonalization`、`com.google.android.federatedcompute`、`com.android.permissioncontroller`、`com.android.rkpd`、`com.android.sdkext`、`com.android.os.statsd`、`com.android.tethering`、`com.android.tzdata`、`com.android.uwb`、`com.android.wifi` — 第五章清单比对的分母；脚本 1 的 `DOC` 列表逐项取自此处。https://source.android.com/docs/core/ota/modular-system
7. 《NFC services》："The NFC Services module is an optional updatable Mainline module released in Android 16." — "可选模块"是清单成为移动目标的直接证据。第廿七章趋势一。https://source.android.com/docs/core/ota/modular-system/nfc-services
8. 《ART》："In Android 11, ART and libcore are packaged as non-updateable APEX." — 可更新性不单调。第廿七章趋势三。https://source.android.com/docs/core/ota/modular-system/art
9. 《Media》："The Media module (com.android.media) and Media Codec module (com.android.media.swcodec) are in APEX format and are available for devices running Android 10 or higher."；"Media components are a frequent target of attacks (many recent security bugs are media related), so tightening security by accelerating updates for security bugs and features should reduce the number of security bugs that can be exploited by attacks." — 文档自己给出的安全动机。第四章、第廿七章趋势五。https://source.android.com/docs/core/ota/modular-system/media
10. 《Wi-Fi》："The Wi-Fi module (com.android.wifi) is in APEX format and is available for devices running Android 11 or higher."；"Symbols (classes, methods, fields, etc.) annotated with `@hide` in the Wi-Fi module are not part of its public API surface and cannot be accessed on devices with the module installed." — 模块装载与否改变应用可见符号集，第十五章之二与十七之 M-AT6 的直接依据。第廿七章趋势六。https://source.android.com/docs/core/ota/modular-system/wifi
11. 《Conscrypt》："The Conscrypt module accelerates security improvements and improves device security without relying on OTA updates."；"Now with Mainline, it's possible to update certificates more frequently using Mainline train updates." — 同一通道既承载代码修复也承载证书数据，因此版本前进不能自动等于补丁到达。第八章之四、第廿七章趋势五。https://source.android.com/docs/core/ota/modular-system/conscrypt
12. 《Permission controller》："Interacts with the framework only via stable @SystemApi (no @hide API usage)."；《Tethering》："Stable AIDL IPCs interface with the system server"；《AIDL》："Interfaces can be declared as stable (backward compatible)." — 契约稳定的三处表述。第十五章之一。https://source.android.com/docs/core/ota/modular-system/permissioncontroller https://source.android.com/docs/core/ota/modular-system/tethering https://source.android.com/docs/core/architecture/aidl/stable-aidl

### 二、APEX 容器、挂载与运行时

13. source.android.com《APEX file format》："At the top level, an APEX file is a zip file in which files are stored uncompressed and located at 4 KB boundaries."；"The four files in an APEX file are: `apex_manifest.json` `AndroidManifest.xml` `apex_payload.img` `apex_pubkey`"；"`apex_payload.img` is an ext4 file system image backed by dm-verity. The image is mounted at runtime via a loopback device."；"the hash tree and metadata block are created using the `libavb` library."；"the `apex_payload.img` (specifically, the vbmeta descriptor appended to `apex_payload.img`) file is signed with a key. Then, the entire APEX is signed using the APK signature scheme v3." — 第七章容器四件套与完整性论证的全部依据；本文据此说明"证明完整性、证明不了包含哪个修复"。https://source.android.com/docs/core/ota/apex
14. 同页压缩条目："`original_apex`: deflated with compression level of 9 … `apex_manifest.pb`: stored only … `AndroidManifest.xml`: stored only … `apex_pubkey`: stored only"；"`apex_manifest.json` file contains the package name and version, which identify an APEX file."；"the APEX can be updated by providing an APEX package with the same package name and a greater than or equal to version code." — 身份定义句与更新规则句。第八章之一、第廿八章边界 1。https://source.android.com/docs/core/ota/apex
15. 同页："After reboot, the APEX is mounted at the `/apex/<apex_name>@<version>` directory. Multiple versions of the same APEX can be mounted at the same time."；"the one that corresponds to the latest version is bind-mounted at `/apex/<apex_name>`. Clients can use the bind-mounted path to read or execute files from APEX."；"The APEX manager (or apexd) is a standalone native process responsible for verifying, installing, and uninstalling APEX files." — 四层挂载语义与 apexd 职责定义。第九章之二。https://source.android.com/docs/core/ota/apex
16. 同页："If the APEX file is verified, the internal database of the APEX manager is updated to reflect that the APEX file gets activated at next boot."；"the newly installed version shadows the version already present in the built-in partition. But unlike APKs, the newly installed version of the APEX is only activated after reboot."；"The APEX manager defaults to using these packages if no updates are available." — 安装≠激活、遮蔽与内置兜底三条时序。第十章之一、第九章之一。https://source.android.com/docs/core/ota/apex
17. 同页："Set the following system property to `true` to support APEX file updates. … `PRODUCT_PROPERTY_OVERRIDES += ro.apex.updatable=true`"；"Note: In Android 15, Flattened APEX is deprecated."；"Flattened APEXes can be updated only via a regular OTA."；"Mixing flattened and non-flattened APEXes in a device is NOT supported." — 通道开关与扁平化形态的更新后果。第九章之四、第廿七章趋势四。https://source.android.com/docs/core/ota/apex
18. AOSP `system/apex/docs/README.md:54,434,437` — "The `apex_manifest.json` file contains the package name and version, which identify an APEX file." / "the APEX is mounted at the `/apex/<apex_name>@<version>` directory" / "bind-mounted at `/apex/<apex_name>`"。— 源码侧对同一挂载语义的复述，ref `android17-release`。第九、十四章。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/docs/README.md

### 三、清单 schema、常量与 apexd 实现

19. AOSP `system/apex/apexd/ApexInfoList.xsd` — 根元素 `apex-info-list`（:18）与 `minOccurs="1"`（:21）；`Partition` 枚举（:28-32）；`apex-info`（:36）及其属性 `moduleName`（:38）、`modulePath`（:39）、`preinstalledModulePath`（:40，可选）、`versionCode`（:41，`xs:long` `use=required`）、`versionName`（:42）、`isFactory`（:43）、`isActive`（:44）、`lastUpdateMillis`（:45，可选）、`provideSharedApexLibs`（:46）、`partition`（:47）。— 脚本 1 每条判级的 schema 依据，也是 `apex_entry_field_missing` 与 `apex_partition_out_of_enum` 能落在 L2 的原因。第八、十四章、第廿二章。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/ApexInfoList.xsd
20. AOSP `system/apex/apexd/apex_constants.h:30-40,51,56,57,78,81-84,86-87,94,103` — `kApexDataDir = "/data/apex"`、`kActiveApexPackagesDataDir = "/data/apex/active"`、`kApexBackupDir`、`kApexDecompressedDir`、`kOtaReservedDir`、`kApexPackageSystemDir = "/system/apex"` 与 `"/system_ext/apex"`、`"/product/apex"`、`"/vendor/apex"`、`"/odm/apex"`；`kApexRoot = "/apex"`；`kStagedSessionsDir = "/data/app-staging"`；`kApexInfoList = "/apex/apex-info-list.xml"`；`kApexStatusSysprop = "apexd.status"`（`starting`/`activated`/`ready`）；`kApexdChangedActiveApexesSysprop = "ro.apexd.changed_active_apexes"`；`kApexAllReadyProp = "apex.all.ready"`；`ctl.apex_load` / `ctl.apex_unload`。— 设备侧目录面的唯一权威清单；`/data/apex/deactive` 在此不存在（第廿八章）。第十三章、第廿二章之二。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apex_constants.h
21. AOSP `system/apex/apexd/apexd.cpp:2221-2222` — "During apex installation, staged sessions located in /metadata/apex/sessions mutate the active set of APEXes."；`:3400-3403` — "If a module is in both a staged dir and the active dir, the APEX with a higher version will be picked. If the versions are equal, the APEX in staged dir will be picked."；`:3406-3407` — "In case of a downgrade install, it differs from the actual, but this is not a supported case." — 遮蔽选择规则与"降级不是受支持情形"；`pair_apex_version_down` 与 `apex_dupe_shadow_*` 的判级依据。第十、十一章、第廿三章。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apexd.cpp
22. AOSP `apexd.cpp:1258` — `// Bind mount the latest version to /apex/<package_name>.`；`:1290-1301` — `DeactivatePackage()` 只执行卸载；`:3257-3259` — 写 apex-info-list 的字段顺序 `apex.GetManifest().version(), apex.GetManifest().versionname(), instance.IsPreInstalledApex(apex), is_active, mtime,`。— 清单字段来源顺序，是本文把 `isFactory` 读成"承载位置"而不是"版本优劣"的依据；也是第廿八章边界 2（`lastUpdateMillis` 刷新语义未文档化）的出处。第九、十章。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apexd.cpp
23. AOSP `system/apex/apexd/apex_manifest.cpp:44-45` — `if (apex_manifest.version() == 0) {` 配合 "Missing required field \"version\"..."；`system/apex/proto/apex_manifest.proto:28,31,41,85,98` — `string name = 1;` / `int64 version = 2;` / `string versionName = 5;` / `bool supportsRebootlessUpdate = 13;` / `bool bootstrap = 16;`。— 身份字段的原型；`bootstrap` 位支撑第九章之三。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apex_manifest.cpp
24. AOSP `system/apex/proto/session_state.proto:24-32` — `UNKNOWN / VERIFIED / STAGED / ACTIVATED / ACTIVATION_FAILED / SUCCESS / REVERT_IN_PROGRESS / REVERTED / REVERT_FAILED`。— 第十章"状态记录的最小完备集"就是这十个值；脚本 1 的会话面判级不超出它们。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/proto/session_state.proto
25. AOSP `system/apex/apexd/apex_file.h:34,53-56,79-80` — `std::unique_ptr<AvbHashtreeDescriptor> desc;` / `GetImageOffset()` / `GetImageSize()` / `image_offset_` / `image_size_`；`external/avb/libavb/avb_hashtree_descriptor.h:72-79` — `uint64_t image_size; uint64_t tree_offset; uint64_t tree_size; uint64_t fec_offset; uint64_t fec_size;`。— 哈希树描述符的**真实字段名**；`payload_offset`/`payload_size` 不是 APEX 字段（第廿八章讹传排除）。第七章之二。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apex_file.h
26. AOSP `system/apex/apexer/apexer.py:112-113` — `choices=['ext4', 'f2fs', 'erofs'], help='type of filesystem being used for payload image'`；`system/apex/apexd/apex_classpath.cpp:84-86` — `// export BOOTCLASSPATH path/to/jar1:/path/to/jar2` / `// export DEX2OATBOOTCLASSPATH` / `// export SYSTEMSERVERCLASSPATH path/to/some/jar`。— payload 文件系统只有三种（无 squashfs）；classpath 三个导出口是第十五章之三的设备侧依据。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexer/apexer.py

### 四、策略与属性可读性

27. AOSP `system/sepolicy/private/shell.te`（android17-release 全量 apex 命中，逐行）— `:163` `allow shell apex_info_file:file r_file_perms;`（注释 "# Allow shell to read /apex/apex-info-list.xml and the vendor apexes"）、`:164-165` `vendor_apex_file` 的 file/dir 授权、`:166` `vendor_apex_metadata_file:dir r_dir_perms;`、`:169` `allow shell apex_data_file:dir search;`、`:172` `allow shell apex_dm_device:blk_file r_file_perms;`、`:281` 第二次 `apex_info_file`（注释 "…for CTS."）、`:378-384` `allow shell { service_manager_type -apex_service -dnsresolver_service … }`、`:569` `neverallow shell {dev_type -apex_dm_device}:blk_file ~getattr;`。**没有** `apex_mnt_dir:dir`、**没有** `apex_module_data_file` 授权。— 第十三章整章与全部"预期失败"判级的根据；也是 `ls /apex` 与 `ls /data/misc/apexdata` 预期失败的直接依据。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/android17-release/private/shell.te
28. AOSP `private/file_contexts:55-56,62,414,489,625-628,629,665,680-688` — `/apex u:object_r:apex_mnt_dir:s0` 与 `/bootstrap-apex` 同名标签；`/apex/(\.(bootstrap|default)-)?apex-info-list.xml u:object_r:apex_info_file:s0`；`/system/bin/apexd u:object_r:apexd_exec:s0`；`/(odm|vendor/odm|vendor|system/vendor)/apex(/[^/]+){0,2} u:object_r:vendor_apex_file:s0`；`/data/apex(/.*)? → apex_data_file`、`/data/apex/active/(.*)? → staging_data_file`、`/data/apex/ota_reserved → apex_ota_reserved_file`、`/data/rollback/\d+/[^/]+/.*\.apex → staging_data_file`；`/data/misc/apexdata` 块列 `apex_module_data_file` 通配加八个点名子目录（`art`/`compos`/`virt`/`permission`/`scheduling`/`tethering`/`uwb`/`wifi`）。— 第四套命名层的出处（第廿九章之二）；`com.android.compos` 等三个名不在清单 30 项内，本文只登记"四套名字"这一事实。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/android17-release/private/file_contexts
29. AOSP `private/property_contexts:208,347-349,355-357,361,364,975-977,1505` — `apexd.` → `apexd_prop`；`apexd.config.`；`persist.apexd.`；`persist.vendor.apex.` 与 `ro.boot.vendor.apex.` → `apexd_select_prop`；`apexd.config.runtime.erofs_file_backed_mount` → `apexd_private_prop`；`apex.` → `apex_ready_prop prefix bool`；`apexd.payload_metadata.path`；`apexd.status … exact enum starting activated ready`；`ro.apexd.changed_active_apexes … exact string`；`ro.apex.updatable u:object_r:exported_default_prop:s0 exact bool`；`ctl.stop$apexd → ctl_apexd_prop`。— 属性面四类可读性的出处；`apex.all.ready` **没有独立条目**，靠 `apex.` 前缀命中（第廿八章边界 3）。第十三章之二。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/android17-release/private/property_contexts
30. AOSP `public/property.te:6,55,155,288` 与 `public/te_macros:926-935,943-951,954-956,959-968` — `system_internal_prop(apexd_prop)`、`system_restricted_prop(apex_ready_prop)`、`system_vendor_config_prop(exported_default_prop)`、`system_public_prop(apexd_select_prop)`；以及四类宏各自的 `neverallow`（"Define a /system-owned property used only in /system" / "can't be written outside /system" / "with no restrictions" / "can only be written by vendor processes"）。— 为什么 `ro.apex.updatable` 可读而 `apexd.status` 对 shell 不可读，是类型宏而不是单条规则决定的。第十三章之二、第廿二章码位表 B 组。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/android17-release/public/te_macros
31. AOSP `private/apexd.te:1,3,6,83-84,89,91` — `typeattribute apexd coredomain;`、`init_daemon_domain(apexd)`、`add_service(apexd, apex_service)`、`allow apexd apex_mnt_dir:filesystem { mount unmount };`、`allow apexd apex_mnt_dir:dir mounton;`、`allow apexd apex_info_file:file relabelto;` 与注释 "# apexd needs to update /apex/apex-info-list.xml after non-staged APEX update."；`private/update_engine.te:34-37` — "# Allow to communicate with apexd…" / `allow update_engine apex_service:service_manager find;`。— 谁能 `find` 到 `apex_service`：策略里点名的是 update_engine，不是 shell；服务面两道门的第一道。第十三章之三。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/android17-release/private/apexd.te
32. AOSP `system/apex/apexd/apexd.rc:1-8,13,23` — `service apexd /system/bin/apexd` / `interface aidl apexservice` / `oneshot` / `disabled # does not start with the core class` / `reboot_on_failure reboot,apexd-failed`；`service apexd-bootstrap /system/bin/apexd --bootstrap`；`service apexd-snapshotde /system/bin/apexd --snapshotde`。`system/core/rootdir/init.rc:85-92`（"# Run apexd-bootstrap so that APEXes that provide critical libraries" / `exec_start apexd-bootstrap`）、`:743-751`（`mkdir /data/apex 0755 root system` / `mkdir /data/apex/active 0755 root system` / `mkdir /data/app-staging 0751 system system encryption=DeleteIfNecessary` / `setprop apexd.status ""` / `restart apexd`）、`:622-625`（`mkdir /metadata/apex/sessions 0700 root system`）。— 脚本 1 判定 `apexd` 进程位、`apexd.status` 为空串的含义、以及两个会话目录（`/data/app-staging` 与 `/metadata/apex/sessions`）都要查的理由。第十三、廿二章。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/apexd.rc

### 五、框架侧与更新引擎可见性

33. AOSP `system/apex/apexd/aidl/android/apex/ApexInfo.aidl:19-26,29,40,54` — `parcelable ApexInfo {` / `moduleName` / `modulePath` / `preinstalledModulePath` / `versionCode` / `versionName` / `isFactory` / `isActive` / `hasClassPathJars`（"// Populated only for getStagedApex() API"）/ `activeApexChanged` / `Partition partition;`。— 第三种身份形状；与 XML 属性名和 `PackageManager` 的标志位都不等价，第十四章"三面字段名不一致"的第一栏。https://android.googlesource.com/platform/system/apex/+/refs/heads/android17-release/apexd/aidl/android/apex/ApexInfo.aidl
34. AOSP `frameworks/base/core/java/android/content/pm/PackageManager.java:1355,1359` — "In APEX terminology, this corresponds to packages that are currently active, i.e. mounted and available to other processes of the OS." / `public static final int MATCH_APEX = 0x40000000;`；`PackageInfo.java:493-494` — "// Whether the package is an APEX package." / `public boolean isApex;`；`services/core/java/com/android/server/pm/ApexManager.java:22` — `import android.apex.ApexInfo;`。— 包管理侧只有标志位没有版本，第十四章之二；`android.content.pm.ApexInfo` 与 `getInstalledApexInfo()` 不存在（讹传排除）。https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/content/pm/PackageManager.java
35. AOSP `system/apex/service/apexservice.cpp:66,176,~406,397,744,775,791,823,831-835` — "Only root and system_server are allowed to call " / `CheckCallerSystemOrRoot("getAllPackages")` / "Only root, keystore, and system_server are allowed to call " / ToString 标签 `"Module: " "Version: " "VersionName: " "Path: " "IsActive: " "IsFactory: " "Partition: "` / `case IBinder::SHELL_COMMAND_TRANSACTION:` / `dprintf(fd, "ACTIVE PACKAGES:\n");` / `dprintf(fd, "SESSIONS:\n");` / `status_t ApexService::shellCommand`，help 串只列 `getAllPackages` / `getActivePackages` / `getStagedSessionInfo [sessionId]`。— 服务面第二道门（调用方校验）与 `dumpsys apexservice` 的输出标签集；脚本 1 服务面三分类的实现依据。第十三章之三、第十四章之三。https://android.googlesource.com/platform/system/apex/+/refs/heads/main/service/apexservice.cpp

### 六、构建装配

36. AOSP `build/soong/apex/apex.go:48,50,52,73-75,81-84,125-129,2569-2576,364,376,391-398,409-411` — `RegisterModuleType("apex", BundleFactory)` / `apex_vndk` / `prebuilt_apex`；`Manifest *string`；`File_contexts *string`（"For platform APEXes, this should points to a file under /system/sepolicy…"）；`Updatable *bool`（"Whether this APEX is considered updatable or not. When set to true, this will enforce additional rules for making sure that the APEX is truly updatable. To be updatable, min_sdk_version should be set as well. … Default is true."）；`ctx.PropertyErrorf("updatable", "updatable APEXes should set min_sdk_version as well")` / `"updatable APEXes should not set min_sdk_version to current…"` / `"updatable APEXes can't use platform APIs"`；`Apps` / `Overrides []string` / `Key *string` / `Certificate *string` / `Min_sdk_version *string`。— 第十一章"构建期确实强制了一部分真可更新"的全部依据；同时说明强制发生在构建期而不是设备侧。https://android.googlesource.com/platform/build/soong/+/refs/heads/main/apex/apex.go
37. AOSP `build/soong/apex/key.go:32,54,56`（"// Path or module to the public key file in avbpubkey format. Installed to the device." / "// … private key file in pem format. Used to sign APEXs."）；`builder.go:1059,1070-1071,141,188-189,376,388,109` — `getCertificateAndPrivateKey` / `cert+".x509.pem"` / `cert+".pk8"` / apexer `--key ${key}` / zip 内同时放 `apex_manifest.json` 与 `apex_manifest.pb` / "// b/143654022 Q apexd can't understand newly added keys in apex_manifest.json prepare stripped-down version" / "// From R+, protobuf binary format (.pb) is the standard format for apex_manifest" / `-se version 0 ${default_version}`；`prebuilt.go:99,104` — "// device (/apex/<apex_name>). If unspecified, follows the name property." / `Source_apex_name`。— 装配落点与产物身份；`static_fw_deps` 不在 `apex.go` 中（讹传排除）。第十一章之二。https://android.googlesource.com/platform/build/soong/+/refs/heads/main/apex/builder.go
38. AOSP `build/main/target/product/updatable_apex.mk:20` — `PRODUCT_SYSTEM_PROPERTIES := ro.apex.updatable=true`；`build/make/core/product.mk:168,285,374` — `PRODUCT_APEX_SYSTEM_SERVER_JARS` / `PRODUCT_DEFAULT_APEX_PAYLOAD_TYPE` / `PRODUCT_APEX_BOOT_JARS`；`apex.go:2771` — `"framework-sdkextensions": {` → `"android.os.ext",`。— 通道档位由产品变量设定、classpath 与 SDK extension 各自的装配入口。第十一、十五章之四。https://android.googlesource.com/platform/build/+/refs/heads/main/core/product.mk

### 七、公告与更新口径

39. source.android.com《Android Security Bulletins》(2024-03-01)："Vulnerabilities are grouped under the component they affect."；"Issues are described in the tables below and include CVE ID, associated references, type of vulnerability, severity, and updated AOSP versions (where applicable)."；"Devices with Android 10 and later may receive security updates as well as Google Play system updates."；"There are no security issues addressed in Google Play system updates (Project Mainline) this month."（2024-04-01 / 2024-08-01 / 2025-08-01 三页复现同一句）。— 第十六章"公告能给出候选而不是答案"的口径来源；"当月无模块条目"是一种**合法输出**而不是数据缺失。https://source.android.com/docs/security/bulletin/2024-03-01
40. 同站 (2024-06-01)："Google Play system updates / The following issues are included in Project Mainline components. / Subcomponent | CVE"；(2022-09-01)："The following issues are included in Project Mainline components. / Component | CVE"。— Mainline 表只有两列（无 Type / Severity / 版本），且列名从 `Component` 迁移到 `Subcomponent`；这是第十七章 M-AT1/M-AT2 与第十九章读数规则的依据。https://source.android.com/docs/security/bulletin/2024-06-01
41. 实测章节标题集合（2022-09 / 2023-05 / 2023-11 / 2024-01 / 03 / 06 / 08 / 09 / 10 / 12 / 2025-01…10）：只有 `Framework`、`System`、`Android Runtime`（旧月为 `Runtime`）、`Kernel`、`Google Play system updates` 加厂商表；**无独立 "Media" 表**；三页（2024-06-01 / 2025-02-01 / 2025-09-01）检索 `com.android.*` 命中 **0 次**。— 公告从不点名 APEX 包名，所以映射必须自建（第廿八章边界 7）。第十六、十九章。https://source.android.com/docs/security/bulletin/
42. 《2025-09-01》Mainline 表整段可引：`Subcomponent | CVE` / `ART CVE-2025-48543` / `AppSearch CVE-2025-26464` / `Cell Broadcast CVE-2025-48534` / `Documents UI CVE-2025-32323` / `MediaProvider CVE-2025-32327, CVE-2025-48532, CVE-2025-48544` / `Permission Controller CVE-2025-48547` / `WiFi CVE-2025-48524`。— 第十九章十三组读数之一，也是"同一 CVE 只在模块表出现一次、版本轴要另查"的实证。https://source.android.com/docs/security/bulletin/2025-09-01
43. 已确认的双表条目（严重表 + Mainline 表同时出现）：CVE-2024-49723（2025-02-01，System / Conscrypt）、CVE-2024-43093（2025-03-01，Framework / Documents UI，页首 targeted-exploitation 注）、CVE-2025-48543（2025-09-01，Android Runtime / ART）、CVE-2025-48524（System / WiFi）、CVE-2025-48534（System / Cell Broadcast）、CVE-2025-48547（System / Permission Controller）、CVE-2025-32323（System / Documents UI）、CVE-2025-26456（2025-06-01，ART）、CVE-2024-40673（2024-10-01，System / ART）、CVE-2024-0018（2024-01-01，Framework / Media Codecs）、CVE-2023-40050 族外的 CVE-2023-40100（2023-11-01，System / DNS Resolver）、CVE-2023-21114（2024-06-01，System / WiFi）、CVE-2022-20395（2022-09-01，System / MediaProvider）。— 第十三组样本的完整集；本文只用它证明"一条修复要读两张表才闭合"，不据此判定任何设备状态。第十九章之二。https://source.android.com/docs/security/bulletin/
44. 厂商表措辞："These vulnerabilities affect Qualcomm components and are described in further detail in the appropriate Qualcomm security bulletin or security alert. The severity assessment of these issues is provided directly by Qualcomm."；Pixel Update Bulletins（`bulletin/pixel/2025-09-01`）列头 `CVE | References | Type | Severity | Subcomponent`，行例 `CVE-2025-36897 A-376027286 * / RCE / High / Cellular Modem`、`CVE-2025-32342 A-407082766 * / EoP / High / TPU`、`CVE-2025-32332 A-319147124 * / High / widevine`。— 器件类与厂商通道的归属在设备侧不可闭合（第十八章）。https://source.android.com/docs/security/bulletin/pixel/2025-09-01
45. source.android.com《Security update resources》："The first task in handling a security vulnerability is to identify the severity of the bug and which component of Android is affected. The severity determines how the issue is prioritized, and the component determines who fixes the bug, who is notified, and how the fix gets deployed to users."；"A bug in the framework or kernel requires an over-the-air (OTA) firmware update that each OEM needs to push. A bug in an app or library published in Google Play … can be sent to Android users as an update from Google Play."；"When a security vulnerability in AOSP is fixed in an Android Security Bulletin, we'll notify Android partners of issue details and provide patches."；"If the security bug is in an AOSP component, the fix is pushed out to AOSP after the OTA is released to users."；"Note: A CVE may not be issued for issues assessed as Low or NSI." — 第四章"为什么模块边界值得当归属单位"与第十七章 M-AT9 的官方依据；"组件决定谁来修、通知谁、怎么发到用户"就是本篇归属六面的文档原型。https://source.android.com/docs/security/overview/updates-resources
46. 同站公告 FAQ："Security vulnerabilities that are documented in this security bulletin are required to declare the latest security patch level on Android devices. Additional security vulnerabilities that are documented in the device / partner security bulletins are not required for declaring a security patch level."；"Issues that are not publicly available have an * next to the corresponding reference ID."；"This bulletin has two security patch levels so that Android partners have the flexibility to fix a subset of vulnerabilities that are similar across all Android devices more quickly." — 第十二章之三"SPL 解释力边界"的三条直接出处。https://source.android.com/docs/security/bulletin/2024-03-01
47. 《Android Security Bulletin overview》："Android platform security fixes are merged into AOSP 24–48 hours after the quarterly security bulletin is released"；"Fixes listed in the public bulletin come from various different sources: the Android Open Source Project (AOSP), the upstream Linux kernel, and system-on-chip (SOC) manufacturers."；"Fixes from SOC manufacturers are available directly from the manufacturers." — 修复来源三分法与 AOSP 发布滞后，第十二章之五的通道小结依据。https://source.android.com/docs/security/bulletin/asb-overview
48. 《Supplemental security patches》："The supplemental security patch XML feature lets OEMs report CVEs patched beyond the declared security patch level (SPL) by providing a standardized XML file, so OEMs receive credit for continuous patching."；"OEMs can place the XML file across multiple device partitions (/system, /vendor, or /product), ensuring that security patches from both the framework and hardware-specific components are accounted for."；"The platform aggregates CVE data across these device partitions and exposes them in platform APIs and Jetpack libraries."（文件 `/etc/security/supplemental_security_patches.xml`）— 唯一能把"超出 SPL 的修复"读出来的公开机制；第十二章之四与脚本 1 的 `supp_patch_*` 两条判级。https://source.android.com/docs/security/overview/supplemental-security-patches

### 八、本次核验的取回方式与已知失败

**取回方式。** 源码条目一律经 `https://android.googlesource.com/platform/<repo>/+/refs/heads/<ref>/<path>?format=TEXT` 取回 base64 正文后本地解码，再按行号定位；`platform/system/apex`、`platform/system/sepolicy`、`platform/external/avb` 用 ref `android17-release`，`platform/build`、`platform/build/soong`、`platform/frameworks/base`、`platform/system/core` 用 ref `main`（`platform/build/soong` 无 `android17-release` 分支，请求返回 404，因此该仓库只能以 `main` 为口径，正文引用时不得声称"该文件在 android17-release 上如何"）。文档条目直接取 HTML 正文。

**已知失败与不据此立论的条目。**

1. `/docs/clients/mainline` 与 `/docs/core/architecture/apex` 两个路径返回 404，正文现路径为 `/docs/core/ota/modular-system` 与 `/docs/core/ota/apex`；旧链接不作为引用入口。
2. `2025-07-01` 与 `2025-10-01` 两个月度公告页 HTTP 200 但抓取正文 0 条 CVE（客户端渲染），本文**没有**把"抓不到"写成"当月无条目"，第十九章的十三组读数明确排除这两页。
3. `sitemap_0..4_of_5.xml` 返回 HTTP 500，无法做全站枚举；第五章清单的 30 项因此只声称"该页 Available modules 表逐字出现的名"，不声称"全部 Mainline 模块"。
4. `reference/sts/classes` 只有类索引与 `HostsideMainlineModuleDetector` 类名，没有说明 STS 目的的散文页；本文不声称文档规定了 STS 的用途（第廿八章边界 8）。
5. `docs/security/bug-protection` 等路径 404，`asb-overview` 无 mainline/module 措辞；本文不引用任何"模块相关披露政策"文句（第廿八章边界 9）。
6. 正文中标注"本文推断""本文自设""本次未逐字核验"的句子，其依据一律以第廿八章的 14 条边界为上限；两处冲突时以第廿八章为准。

**范围声明。** 本篇是防御性与归属方法分析：全部内容只涉及公开文档、公开源码行号、公开公告表与本报告的合成档案验证实操，不含任何漏洞利用构造、不含绕过或降级模块安装的操作性步骤。第廿二、廿三章两份脚本对设备只做只读查询，不安装、不卸载、不重启、不降级、不解包任何 `.apex`、不访问网络；读者应在自己的或已获明确授权的设备上运行。第廿四章的全部输入为合成设备档案，不对应任何真实机型，其中的属性值、包名与版本号均为构造值，不得作为设备样本引用。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
