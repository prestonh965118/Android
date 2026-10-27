---
layout: default
title: "Android 补丁传导链与厂商更新治理：从 SoC 公告到设备 SPL | Android 安全报告集"
description: "拆解 SoC 厂商安全公告、OEM 适配与设备分发之间的时间差，说明安全补丁级别 SPL 与厂商公告口径的差距、更新承诺的代际与年限差异，以及可只读核验的设备侧更新证据。面向 Android 设备安全评估与补丁治理。"
lang: zh-CN
permalink: /Android-Patch-Delivery-Chain-and-OEM-Update-Governance/
---
# Android 补丁传导链与厂商更新治理分析报告

**Android Patch Delivery Chain and OEM Update Governance: From SoC Advisories to On-Device SPL**

- **报告日期**：2026-10-26
- **报告性质**：公开资料整理与防御性补丁治理分析
- **适用对象**：漏洞管理与补丁治理负责人、企业移动管理（EMM/MDM）与安全运营、OEM/ROM 发布工程、移动平台安全负责人、第三方评估与审计机构、取证与设备资产管理
- 视角声明：本文写的是**补丁如何从公开公告走到你的设备上，以及怎么用可核验的数据判断它到了没有**。各子系统内部的缺陷类型学、CVE 逐条复盘与缓解机制在姊妹篇里，本文只做交叉引用：
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核缺陷类型学、公开案例、加固机制清单、作为治理指标的 SPL 一节）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（EoP 类缺陷与公告归因）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层、HAL/Treble、第 12 章更新架构——本文的通道划分与其对齐）
  - [Android Linux 内核结构与子系统分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（ACK/GKI 内核形态与模块边界）
  - [Android 跨域攻击面地图与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（第 12 域分发/更新/调试通道、分诊归因与补丁验证流程）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（版本绑定、度量与证明——本文第 2.3 节用到它的语义）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（器件与认证侧证据；本文负责"固件能不能推到设备上"这一半）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（本文讲修没修到，该篇讲这一类弱点有没有被纳入检查：更新治理解决已定义缺陷的分发，分类学解决未声明形态的发现）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本文的 S1–S5 链条解释补丁为什么到不了设备；该篇把无线侧当作最难归因的一条通道来使用这些结论：基带固件与蓝牙控制器固件通常落在 SPL 语义难以直接覆盖的位置，设备侧能读到的往往只有 `gsm.version.baseband`，所以"读不到"被记成缺口而不是通过）
- 范围声明：全文只讨论**公开文档化**的公告机制、更新通道、厂商公开政策与公开研究数据，并给出防御侧的度量、核验与治理方法。**不提供任何绕过更新校验、伪造补丁级别、降级固件或规避厂商限制的操作步骤，也不含漏洞利用代码。**所有"核验"均指在你拥有或获授权的设备上执行只读采集。

---

## 一、摘要

一条 Android 漏洞从公开到"你手上这台设备已修复"，中间隔着一条至少五段的链条：**上游/内核树 → AOSP 或 GKI 发布 → SoC 厂商适配 → OEM 集成与验证 → OTA 推到具体机型/区域/载体**。每一段都有自己的公开产出物，也就都有自己的可测量时间差。安全社区反复测量的结论是稳定的：决定设备真实风险的从来不是"公告有没有发"，而是**这条链在你这台机器上走完了没有**。

本文回答四个治理层面的问题，它们和子系统层面的分析（姊妹篇负责）不同：

1. **SPL 这个日期到底承诺了什么。** 它是"该构建已包含的修复的日期标记"，且 Android 公告体系同时存在**两个补丁级别**（同一期公告给出 `YYYY-MM-01` 与 `YYYY-MM-05` 两档，让合作伙伴可以先修复其中一组相似且严重度不高于某一水平的漏洞）；框架侧与厂商侧还各自记录补丁级别（`ro.build.version.security_patch` 与 vendor 侧的 `ro.vendor.build.version.security_patch`，后者被 AOSP 版本绑定文档描述为存放在只读属性中、并可由引导程序提供校验哈希）。**一个 SPL 字符串不等于整机已修复**——第 2 章给出拆法。
2. **五段链条上每一段的公开证据是什么。** 上游优先（AOSP 内核文档要求开发者先用 LKML 提交，再进 ACK 的 `android-mainline`）、GKI **按季度节奏在 KMI 冻结后发布**、SoC 厂商按自己的公告体系发布（高通公告自述用途是"帮助客户在已发布与将发布的设备中纳入安全更新"，并说明补丁正与 OEM 共享；联发科公告"包含影响某些联发科芯片组的安全漏洞详情"）、OEM 有自己的更新范围页（三星公开其安全更新支持范围，并说明 OS 升级可能导致计划内安全更新的延迟）。**每段都有可引用的一手页面，链条断裂处也就有据可查。**
3. **时间差能不能度量。** 能，而且有两类互补数据：学术侧的设备级测量（USENIX Security 2021 对 Android 内核补丁生态的调查指出"多数 OEM 跟随 Google 的月度节奏更新补丁级别，因此 OEM 补丁会被不必要地延迟"；NDSS 2024 的设备中心分析指出延迟原因不止碎片化，还包含多个厂商特有问题；2026 年 IEEE 的更新景观研究测量出"不同 OEM、设备类型与区域之间的更新时间差异显著"）；运营侧的设备采集（第 10–11 章脚本，直接从设备读 SPL、vendor SPL、构建日期、指纹与 APEX 目录）。
4. **怎么把它变成能运营的指标。** 第 9 章给出八项指标（含定义、数据来源、建议阈值与各自会被钻空子的地方），第 12 章给出更新异常的取证信号（SPL 与构建日期矛盾、指纹与 incremental 不一致、vendor SPL 长期落后等）。

三条从数据里直接得出的判断：

- **"有更新承诺"与"有更新"是两件事。** 承诺是政策文档（Google 对 Pixel 8 及后续机型公开"自该机型的美国 Google Store 首发日起七年更新"；三星从 2024 年 1 月起扩大支持范围，此前公开口径是四代 OS 升级与最长五年安全更新），实际到达要靠设备上采到的 SPL 与构建日期。政策还会**双向修改**：Google 在 2024 年 12 月调整了信息页，把 Pixel 6/7 系列的支持期延长——这说明"以购买时的承诺评估风险"是不可靠的，要以当前公告页与设备实测为准。
- **厂商侧补丁是链条上最长、也最不可见的一段。** AOSP 侧每月公告、有源码、有强制测试；厂商驱动、闭源 HAL、基带固件的更新往往只在芯片公告里以组件名出现，机型映射不公开。这与硬件报告得出的结论一致：**入口审计与固件到达性，是硬件安全价值的上限。**
- **更新通道已经不止一条。** 系统 OTA、Mainline 模块（可"由 Google 通过 Google Play 系统更新推送，也可由 OEM 推送"）、GKI 内核与模块各有节奏；一条漏洞可能已经通过 APEX 修好而未反映在 SPL 上，也可能 SPL 很漂亮而厂商分区停在半年前。**度量必须按通道拆分，不能只看一个日期。**

对防御方，本文的交付物是三件：**五段链条的责任与证据表**（第 3–4 章）、**八项可运营指标**（第 9 章）、**两个只读脚本与台账格式**（第 10–11 章，用于设备池核对与验收）。

---

## 二、SPL 的语义：一个日期标记能说什么、不能说什么

### 2.1 SPL 是"包含关系"的标记，不是"修复完成"的证明

安全补丁级别（Security Patch Level）的作用是让设备声明"本构建已包含截至该日期的修复"。Android 公告体系本身就按**补丁字符串**组织：同一期公告可以同时给出两档 SPL，公告原文对此的解释是让合作伙伴有灵活性去先修复其中一组（相似且严重度不高于某个水平的）漏洞。因此：

- 设备报 `YYYY-MM-05`，说的是它包含该月**较完整**的一组修复；报 `YYYY-MM-01`，说的是它只承诺第一组。
- 两档机制的存在意味着：**"这个月的公告发过了"不等于"所有设备都能升到同一个日期"**，比较设备时必须带上是哪一档。

一个可从第三方发布说明中观察到的现实例子：某定制系统公开说明自己"把声明的 Android 安全补丁级别提高到 2026-01-05"，同时说明该级别自上一季度起已在提供。这说明 SPL 在不同构建体系里是一个**声明值**，其可信度依赖该构建体系的更新流程与可核验产物，而不是这个数字本身。

### 2.2 框架 SPL 与厂商 SPL 是两个字段

Android 的 Treble 结构把系统侧与厂商侧分开演进，补丁级别也分开记录：

| 字段 | 属性 | 覆盖范围 |
| --- | --- | --- |
| 框架补丁级别 | `ro.build.version.security_patch` | 平台/框架侧（AOSP 代码） |
| 厂商补丁级别 | `ro.vendor.build.version.security_patch`（AOSP 版本绑定文档描述的只读属性；部分公开报道使用 `ro.vendor.build.security_patch` 的写法） | vendor 侧（厂商实现、HAL、厂商驱动） |

**治理上的后果很直接**：只看前者会把"框架每月更新、厂商停在半年前"的设备读成健康设备。第 10 章脚本把两个字段都采出来并计算差值；第 9 章把"厂商 SPL 落后月数"列为独立指标。

### 2.3 版本绑定：SPL 与硬件密钥之间有直接关系

AOSP 的密钥版本绑定机制把密钥与"补丁级别/OS 版本"这类状态关联，并描述为把厂商补丁级别存放在只读属性中、由引导程序提供所校验数据的哈希。**对治理的意义**：当补丁级别成为密钥可用性的输入时，SPL 就不再只是一个信息字段——它会影响业务侧的密钥能否继续用。这也是"厂商长期不更新 vendor 侧"的真正代价之一：不只是暴露面还在，而是依赖版本绑定的硬件密钥会随之出现可用性变化。逐字段语义在密钥栈姊妹篇的 AuthToken 与版本绑定相关章节。

### 2.4 SPL 的四个"不能"

1. **不能证明补丁真的在你的机型上生效**——它证明的是构建声明。验证要靠第 10 章的构建日期、指纹与厂商公告对应关系交叉核对。
2. **不能覆盖基带与安全侧固件**——蜂窝调制解调器、安全处理器等域的固件更新通常不在 SPL 语义内，需要厂商文档与设备侧固件版本字段另行采集（硬件报告第 6 章的三条通道正是这类）。
3. **不能反映 Mainline 模块的实际版本**——模块可以独立推送，其版本要用 APEX 侧数据观察，而非 SPL。
4. **不能跨机型比较安全性**——同一 SPL 在两台设备上意味着不同的暴露面，因为厂商实现不同；比较要在"同一机型不同版本"或"同一版本不同区域"之间做。

---

## 三、传导链的五段结构：每段的产出物与公开证据

把链条拆开，是为了让"延迟发生在哪一段"变成可查证的问题，而不是"OEM 不给力"这种无法行动的结论。

| 段 | 谁负责 | 公开产出物 | 可用于核验的证据 | 该段的典型延迟来源 |
| --- | --- | --- | --- | --- |
| **S1 上游与内核树** | 内核社区 / AOSP 内核工程 | 上游提交、Android Common Kernels（ACK）分支 | AOSP 内核开发文档要求**先向上游（LKML）提交**，再提交到 ACK 的 `android-mainline` | 上游合入排队、backport 判断 |
| **S2 平台与 GKI 发布** | Google | AOSP 源码、月度安全公告、GKI 发布 | GKI **在 KMI 冻结后按季度发布**；内核 5.10 及以上即称 GKI 内核 | 季度节奏与月度公告之间的天然错位 |
| **S3 SoC 厂商适配** | 芯片厂商 | 芯片侧安全公告 + SDK/补丁包 | 高通公告索引自述用途是"帮助 QTI 客户在已发布与将发布的设备中纳入安全更新"，且说明补丁正与 OEM 共享；联发科公告"包含影响某些联发科芯片组的安全漏洞详情" | 芯片公告按**技术域**（HLOS、DSP Service 等）组织，不给出机型清单 |
| **S4 OEM 集成与验证** | 设备厂商 | 机型更新说明、支持范围页 | 三星公开安全更新支持范围页，并说明 **OS 升级可能导致计划内安全更新延迟** | 兼容性验证、载体审核、区域认证 |
| **S5 分发到设备** | OEM / Google（Mainline）| OTA 批次、Play 系统更新 | Mainline 文档说明更新模块"可由 Google 通过 Google Play 系统更新推送，也可由 OEM 推送" | 分批灰度、载体审批、用户未重启/未接受 |

三个结构性事实值得单独强调：

- **S2 与 S3 的节奏天生不同步。** 公告是月度、GKI 是季度，厂商 SDK 又跟着自己的排期。这解释了为什么"厂商 SPL 落后框架 SPL"是常态而非例外，也是第 9 章把两者差值单独设指标的原因。
- **S3 是链条上证据最薄的一段。** 芯片公告通常只到"组件/技术域"层级，机型映射由 OEM 掌握。要评估一条芯片侧缺陷是否影响你 fleet 里的设备，必须拿到厂商的机型适配说明，或退而用设备侧 vendor SPL 与固件字段做代理指标。
- **S5 有第二条腿。** Mainline 让一部分平台组件可以不经整机 OTA 更新——这是过去几年 Android 更新治理真正改变性质的变化，也是"整机 SPL 落后但组件已修"这一现象的成因。

---

## 四、时间差怎么测：两类互补的数据

### 4.1 公开测量研究给出的量级

跨厂商的更新时间差是可以测量的，公开研究至少给出三个稳定结论：

- **OEM 跟随月度节奏会造成不必要延迟。** USENIX Security 2021 关于 Android 内核补丁生态的调查明确指出："多数 OEM 厂商跟随 Google 的月度节奏更新安全补丁级别，因此 OEM 补丁会被不必要地延迟。"这条结论的治理含义是：**以公告日为起点评估设备风险会低估真实暴露窗口**。
- **延迟原因不止碎片化。** NDSS 2024 的设备中心分析指出，安全更新推出的延迟原因不仅限于碎片化，还包含多个厂商特有的因素。这直接否定"小厂商慢是因为技术能力"的单一解释，也说明**同一厂商不同机型/区域的差异可能比不同厂商之间还大**。
- **差异在 OEM、设备类型与区域三个维度上同时存在。** 2026 年发表于 IEEE 的更新景观研究测量出"不同 OEM、设备类型与区域之间在更新时间上存在显著差异"，并把延迟与碎片化及厂商特有挑战关联。含义：**任何以"品牌"为单位的更新治理结论都太粗**，要下沉到机型与区域。

### 4.2 设备侧的自证数据

在 fleet 层面，你不依赖上面的研究也能拿到自己的数据。可采集的字段是：

| 字段 | 来源 | 用途 |
| --- | --- | --- |
| 框架 SPL | `ro.build.version.security_patch` | 平台侧声明 |
| 厂商 SPL | `ro.vendor.build.version.security_patch` | 厂商侧声明，与上者求差 |
| 构建日期 | `ro.build.date.utc` / `ro.build.date` | 与 SPL 交叉验证（SPL 晚于构建日期即异常） |
| 增量版本 | `ro.build.version.incremental` | 同机型同 SPL 但 incremental 不同 = 批次差异 |
| 指纹 | `ro.build.fingerprint` | 机型/区域/载体识别的锚点 |
| 首次 API 级别 | `ro.product.first_api_level` | 生命周期位置（决定支持期起算） |
| APEX 目录 | `/apex` 下的挂载项数量 | Mainline 组件到达性的粗粒度代理 |

**这七个字段全部只读可得**，第 10 章把它们采成结构化输出，第 11 章做批量断言。它们不能证明"补丁存在"（那要靠源码与公告核对），但足以在 fleet 尺度上把"哪台设备的链条没走完"排出来。

---

## 五、三条并行更新通道：分别怎么观察、分别怎么度量

架构报告的更新架构一章给出通道划分，本文给出**治理视角的观察方法**。三条通道的时间差互不相关，混在一个指标里会得出错误结论。

### 5.1 整机 OTA（SPL 所属通道）

- **观察**：框架 SPL、厂商 SPL、构建日期、指纹、增量版本。
- **度量**：距公告日的天数、同机型 fleet 内的分布（不是平均值，是尾部：最旧的那批设备决定风险）。
- **常见误读**：把"厂商发布了新固件"当成"我的设备收到了"。S5 的分批灰度与载体审核会使两者相差数周到数月，NDSS 2024 的结论正是这类差异不应归因于单一因素。

### 5.2 Mainline 模块 / Google Play 系统更新

- **观察**：`/apex` 目录下的活动模块（数量与名称可作为粗粒度到达性代理）；组件级版本要由具备权限的包管理服务读取，采集范围依设备授权状态而定。
- **公开依据**：AOSP Mainline 文档说明更新后的模块"可由 Google 通过 Google Play 系统更新推送，也可由 OEM 推送"；Google 系统服务更新说明页描述其用途（安全与可靠性更新）；Binary Transparency 的 Mainline 模块透明度页承诺列出以该形式提供的每个 Android OS 模块。
- **度量**：模块是否与厂商自定义版本冲突、厂商是否禁用了 Play 系统更新通道（企业设备池里常见）。
- **为什么必须单独度量**：一条平台组件缺陷可能已通过模块修好，而整机 SPL 仍显示旧值；反之 SPL 漂亮不代表某个厂商替换过的模块是最新版。**只看 SPL 会在两个方向上都错。**

### 5.3 内核与 GKI

- **观察**：内核版本字符串与模块（机制细节见内核结构报告）。
- **公开依据**：GKI 在 KMI 冻结后按季度发布；5.10 及以上版本即称 GKI 内核；内核开发文档要求优先上游提交再进 ACK。
- **度量**：内核版本相对当期 GKI 发布的落后程度；厂商模块（DLKM）与 GKI 核心的更新是否分离。
- **结构性风险**：GKI 的季度节奏与月度公告之间存在天然错位。这意味着**内核侧修复的到达时间以 GKI 发布为节拍**，用月度公告日评估内核风险会系统性乐观。

---

## 六、更新承诺：政策口径的读法与三处坑

承诺文件是采购与合规里最常引用的证据，也是最容易被误读的证据。公开可引用的例子：

- **Google Pixel**：官方支持页表述为"Pixel 8 及之后的机型将获得七年更新，自该机型首次在美国 Google Store 上市之日起计算"。同一页面还承载过政策调整——2024 年 12 月 Google 更新了软件更新信息页，把 Pixel 6 与 Pixel 7 全系的支持期延长。
- **Samsung Galaxy**：官方安全更新范围页说明自 2024 年 1 月起扩展 Galaxy 设备的安全更新支持，并在同页提示 **OS 升级可能导致计划内安全更新出现延迟**；更早的官方新闻稿口径是"四代 OS 升级 + 最长五年安全更新"。
- **社区维护的生命周期数据**：`endoflife.date` 这类聚合页把机型的支持期与到期日结构化，适合做 fleet 的初筛，但**不能替代厂商页面**（它自己也以指向官方页为来源）。

### 6.1 坑一：起算点

Pixel 的七年从"该机型在美国 Google Store 首次上市"起算，而不是从你购买、也不是从你所在区域的上市时间起算。采购合同里如果只写"七年支持"，同一型号在不同批次的实际剩余支持期可能差一年以上。

### 6.2 坑二：OS 升级与安全更新的相互挤占

三星的页面直接把"OS 升级可能导致计划内安全更新延迟"写成条款。这是**结构性而非例外**的现象：大版本升级占用同一批工程与验证资源。评估某机型某季度的更新可靠性时，要问"这个季度有没有大版本升级"，而不是只看历史月数。

### 6.3 坑三：承诺会被修改，而且双向

延长（Pixel 6/7）与终止（各机型到期）都会发生。因此**合规材料必须写"截至某日依据某 URL 的政策"**，并配第 10 章的设备实测值。只有政策链接没有实测值的评估，是一次快照而非治理。

---

## 七、长周期设备的官方指引：把"支持"变成可测试项

Google 对长生命周期设备（车载为代表）发布了厂商指南，公开表述是"描述 Google 推荐的应用安全补丁最佳实践，这些实践由 Android 兼容性测试套件（CTS）评估"。这段话在治理上很有用，因为它把两件事连起来了：

1. **补丁应用流程本身是可评估的对象**，不是厂商内部事务；
2. **兼容性测试是"厂商说支持"与"设备真的按支持流程走"之间的桥梁**。本系列的架构与方法论报告都覆盖 CTS 侧（含 SELinux 主机测试这类可核验项），本文只补一条治理动作：**采购问卷里要问"该机型是否通过相应版本的 CTS/VTS，测试报告能否提供"**，而不是只问"支持几年"。

对自有 ROM/设备的团队，这份指南同样可直接用作流程蓝本：补丁挑选、验证节奏、回归范围——这些决定了你在链条 S4 段的耗时，而 S4 恰好是 fleet 层面最常被观测到延迟的一段。

---

## 八、缺陷归属：公告口径的三种粒度与核验难度

| 公告来源 | 粒度 | 公开可核验度 | 治理动作 |
| --- | --- | --- | --- |
| **AOSP 平台公告** | 按组件（Framework / System / Library）+ CVE + 严重度 + 受影响版本 | 高：可回到源码提交核对 | 用 SPL 判断是否覆盖；第 10 章脚本 |
| **厂商（OEM）公告** | 按机型与固件版本 | 中：需要厂商页面与批次信息 | 用指纹/增量版本与公告对应；企业池按机型台账 |
| **SoC 厂商公告** | 按**技术域**（如 HLOS、DSP Service）与组件名，不一定给机型清单 | 低：高通公告索引自述用途是帮助客户在已发布与将发布设备中纳入更新；补丁"正与 OEM 共享" | 只能问 OEM：该芯片域缺陷是否已在我机型修复；或用 vendor SPL 做代理指标 |

**这条粒度差是整份文档最重要的操作性事实。** 平台侧缺陷你可以自己核对到源码；芯片侧缺陷你连"影响哪些机型"都拿不到公开答案，只能要求 OEM 提供映射。基带类是极端情形：Project Zero 2023-03 一次性公开 Exynos 调制解调器四个网络可达级别的缺陷（CVE-2023-24033、26496、26497、26498），Google 随后公开说明了基带加固方向；但设备侧要确认"我的基带固件版本是否已含对应修复"，通常只能依赖厂商的固件说明页——第 12 章把这类"采不到"的情况列为数据质量缺口，而不是当作"没问题"。

---

## 九、八项可运营指标

每项给出：定义、数据来源、建议阈值与**这一项会被怎么钻空子**。阈值按企业池经验给出，需要按业务基线校准。

| # | 指标 | 数据来源 | 建议阈值 | 会被钻空子的地方 |
| --- | --- | --- | --- | --- |
| 1 | **框架 SPL 落后天数** | `ro.build.version.security_patch` vs 当期公告日 | P1 设备 ≤45 天；一般设备 ≤90 天 | 用 `-01` 档冒充 `-05` 档；比较时必须带档位 |
| 2 | **厂商 SPL 落后月数** | `ro.vendor.build.version.security_patch` | ≤2 个月 | 属性缺失时不能记 0，缺失要单列（脚本 2 的处理方式） |
| 3 | **框架/厂商 SPL 差值** | 两者相减 | ≤1 个月 | 只报框架值会掩盖厂商侧长期停更 |
| 4 | **SPL 与构建日期一致性** | SPL vs `ro.build.date.utc` | SPL 不得晚于构建日期 | 声明式 SPL 可被改写，第三方构建需用产物核对 |
| 5 | **fleet 内最旧设备占比** | 台账分组统计（尾部而非均值） | 最旧 5% 单独处置 | 用平均 SPL 汇报会藏住长尾 |
| 6 | **同机型批次离散度** | 同机型不同设备的 SPL/incremental 分布 | 离散度突增需查 | 灰度发布期正常，长期离散=有设备掉队 |
| 7 | **Mainline 通道可用性** | `/apex` 观察 + 厂商是否禁用 Play 系统更新 | 禁用需有例外审批 | 只看 SPL 会忽略这条通道带来的实际修复 |
| 8 | **EOL/临期设备数量** | 机型支持政策页 + `ro.product.first_api_level` | 距支持终点 ≤6 个月的设备要有替换计划 | 政策起算点误读（第 6.1 节）会让计划提前或延后一年 |

**指标之间的取舍要显式写进政策**：第 1、2、3 项是"链条走完没有"；第 4、5、6 项是"数据可信不可信"；第 7、8 项是"通道与生命周期"。只汇报第 1 项的更新治理，等同于把 S3/S4 段的延迟从报表里删掉。

---

## 十、设备侧核验之一：补丁到达性采集

脚本 1 采出第 4.2 节字段表里的可采项，并对第 9 章指标 1–4 直接给出判定。**基准日期由调用者给定**，不依赖本机时钟——审计与取证场景里这一点很重要，否则同一设备在不同机器上会得到不同结论。全程只读。

```bash
#!/usr/bin/env bash
# 脚本 1: 补丁到达性采集与判定（只读）
# 用法: patch-arrival.sh <设备序列号> <基准日期 YYYY-MM-DD>
# 输出: key,value 与 FIND_*,判定可回溯到上面的采集值
set -u
dev="${1:?用法: patch-arrival.sh <serial> <YYYY-MM-DD>}"
ref="${2:?缺少基准日期，格式 YYYY-MM-DD}"

adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable" >&2; exit 1; }

p() { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }
q() { adb -s "$dev" shell "toybox $1" 2>/dev/null | tr -d '\r'; }
row() { printf '%s,%s\n' "$1" "$2"; }

# 月序号：把 YYYY-MM-DD 变成可相减的整数（十进制强制，避免八进制歧义）
mi() {
  case "$1" in
    [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]) printf '%s' $(( 10#${1:0:4} * 12 + 10#${1:5:2} )) ;;
    *) printf '' ;;
  esac
}

spl="$(p ro.build.version.security_patch)"
vspl="$(p ro.vendor.build.version.security_patch)"
bd="$(p ro.build.date.utc)"
inc="$(p ro.build.version.incremental)"
fp="$(p ro.build.fingerprint)"
model="$(p ro.product.model)"
sdk="$(p ro.build.version.sdk)"
first="$(p ro.product.first_api_level)"
apex="$(q "ls /apex" | grep -c .)"
apex="${apex:-0}"

row model "$model"; row spl "$spl"; row vendor_spl "$vspl"
row build_date_utc "$bd"; row incremental "$inc"; row fingerprint "$fp"
row sdk "$sdk"; row first_api "$first"; row apex_count "$apex"
row reference_date "$ref"

ref_m="$(mi "$ref")"
spl_m="$(mi "$spl")"
if [ -z "$ref_m" ]; then echo "ERR,bad_reference_date" >&2; exit 2; fi

if [ -z "$spl_m" ]; then
  row FIND_spl "L0-LOW 未采集到框架 SPL，属性缺失或被裁剪，不得记为已修复"
else
  gap=$(( ref_m - spl_m ))
  if [ "$gap" -gt 3 ]; then
    row FIND_spl "L3-HIGH 框架 SPL 落后基准约 ${gap} 个月，超出任何合理的推送窗口"
  elif [ "$gap" -gt 1 ]; then
    row FIND_spl "L2-WARN 框架 SPL 落后约 ${gap} 个月，需确认是推送未完成还是该机型已停止更新"
  else
    row FIND_spl "L1-INFO 框架 SPL 与基准同月或仅差 1 个月"
  fi
fi

if [ -z "$vspl" ]; then
  row FIND_vendor_spl "L0-LOW 未采集到厂商 SPL：该机型未暴露此属性，厂商侧到达性无法从设备判定，需向厂商索取机型说明"
else
  v_m="$(mi "$vspl")"
  if [ -z "$v_m" ]; then
    row FIND_vendor_spl "L0-LOW 厂商 SPL 值无法解析（${vspl}）"
  elif [ -z "$spl_m" ]; then
    row FIND_vendor_spl "L0-LOW 框架 SPL 缺失，无法计算两侧差值"
  else
    vgap=$(( spl_m - v_m ))
    if [ "$vgap" -gt 1 ]; then
      row FIND_vendor_spl "L3-HIGH 厂商侧落后框架侧 ${vgap} 个月：vendor 分区（HAL/厂商驱动）为独立更新面，此差值即链条 S3/S4 段的实测长度"
    elif [ "$vgap" -lt 0 ]; then
      row FIND_vendor_spl "L2-WARN 厂商 SPL 晚于框架 SPL，两侧声明不同步，需核对构建来源"
    else
      row FIND_vendor_spl "L1-INFO 厂商 SPL 与框架 SPL 同步"
    fi
  fi
fi

if [ -z "$bd" ]; then
  row FIND_build_date "L0-LOW 无构建日期，无法与 SPL 交叉验证"
else
  bm="$(date -u -d "@$bd" +%Y-%m 2>/dev/null || date -u -r "$bd" +%Y-%m 2>/dev/null || printf '')"
  b_m="$(mi "${bm}-01")"
  if [ -z "$b_m" ]; then
    row FIND_build_date "L0-LOW 本机 date 无法换算构建时间，请手动核对（utc=${bd}）"
  elif [ "$spl_m" -gt "$b_m" ]; then
    row FIND_build_date "L2-WARN 声明的 SPL 晚于构建日期，属于不一致状态：可能是第三方构建改写声明或采集到混合分区"
  else
    row FIND_build_date "L1-INFO 构建日期不早于声明的 SPL，两者一致"
  fi
fi

if [ "$apex" -gt 0 ]; then
  row FIND_apex "L1-INFO 观察到 ${apex} 个活动 APEX 挂载项；Mainline 通道存在，模块实际版本需按权限进一步采集"
else
  row FIND_apex "L0-LOW 未观察到 APEX 挂载项，可能是授权受限或非量产系统；不得据此判定 Mainline 已更新或未更新"
fi

if [ -z "$first" ] || [ -z "$sdk" ]; then
  row FIND_lifecycle "L0-LOW 出厂或当前 API 级别缺失，无法判断是否发生主版本升级"
elif [ "$first" != "$sdk" ]; then
  row FIND_lifecycle "L2-WARN 出厂 API(${first}) 与当前 API(${sdk}) 不同，已发生主版本升级；按第 6.2 节，主版本升级季度可能挤占安全更新节奏"
else
  row FIND_lifecycle "L1-INFO 未观察到主版本升级，支持期起算点按机型政策页核对"
fi
```

**读法**：`FIND_spl` 是"链条有没有走完"，`FIND_vendor_spl` 是"S3/S4 段有多长"，`FIND_build_date` 是"这条声明自不自洽"。三者必须一起看：只看第一条会把"框架每月更新、厂商停在半年前"的设备读成健康设备。

**判定粒度是月**（`mi()` 只取年月），所以 `FIND_spl` 的 `L3-HIGH` 对应"落后 >3 个月"，比第 9 章指标 1 的 45/90 天阈值更粗。要按天考核，请在台账里用 `spl` 与当期公告日直接做日期差；本脚本的职责是把单台设备的判定做成可复现、可回溯，而不是替代 fleet 报表。

---

## 十一、设备侧核验之二：fleet 批量基线断言

脚本 2 用一份基线文件对多台设备做同一组断言。基线文件按机型写明**最低可接受的框架 SPL 与厂商 SPL**（ISO 日期可直接做字符串比较），因此基线变更不需要改脚本。

```bash
#!/usr/bin/env bash
# 脚本 2: fleet 补丁基线断言（只读）
# 用法: patch-ledger.sh <基线文件> <设备序列号> [设备序列号 ...]
# 基线文件每行: 型号,最低框架SPL,最低厂商SPL   以 # 开头为注释
set -u
base="${1:?用法: patch-ledger.sh <baseline-file> <serial> ...}"
shift
[ -f "$base" ] || { echo "ERR,baseline_missing,$base" >&2; exit 1; }
[ "$#" -gt 0 ] || { echo "ERR,no_device" >&2; exit 1; }

p() { adb -s "$1" shell getprop "$2" 2>/dev/null | tr -d '\r'; }

sum_high=0; sum_warn=0; sum_low=0; sum_ok=0

for dev in "$@"; do
  if ! adb -s "$dev" get-state >/dev/null 2>&1; then
    printf 'LOW,%s,unreachable,设备不可达\n' "$dev"; sum_low=$((sum_low + 1)); continue
  fi
  model="$(p "$dev" ro.product.model)"
  spl="$(p "$dev" ro.build.version.security_patch)"
  vspl="$(p "$dev" ro.vendor.build.version.security_patch)"
  if [ -z "$model" ]; then
    printf 'LOW,%s,model_missing,型号属性为空，无法安全匹配基线行\n' "$dev"
    sum_low=$((sum_low + 1)); continue
  fi

  line="$(grep -m1 -F "$model," "$base" || true)"
  if [ -z "$line" ]; then
    printf 'LOW,%s,no_baseline,型号=%s,基线文件未覆盖该型号\n' "$dev" "$model"
    sum_low=$((sum_low + 1)); continue
  fi
  want="$(printf '%s' "$line" | cut -d, -f2)"
  vwant="$(printf '%s' "$line" | cut -d, -f3)"

  if [ -z "$spl" ]; then
    printf 'LOW,%s,spl_missing,型号=%s,不得记为通过\n' "$dev" "$model"; sum_low=$((sum_low + 1))
  elif [[ "$spl" < "$want" ]]; then
    printf 'HIGH,%s,spl_below_baseline,got=%s,min=%s\n' "$dev" "$spl" "$want"; sum_high=$((sum_high + 1))
  else
    printf 'INFO,%s,spl_ok,got=%s\n' "$dev" "$spl"; sum_ok=$((sum_ok + 1))
  fi

  if [ -z "$vwant" ] || [ "$vwant" = "-" ]; then
    printf 'LOW,%s,vendor_baseline_absent,厂商侧无法断言\n' "$dev"; sum_low=$((sum_low + 1))
  elif [ -z "$vspl" ]; then
    printf 'MED,%s,vendor_spl_missing,型号=%s,厂商侧到达性未验证\n' "$dev" "$model"; sum_warn=$((sum_warn + 1))
  elif [[ "$vspl" < "$vwant" ]]; then
    printf 'HIGH,%s,vendor_spl_below_baseline,got=%s,min=%s\n' "$dev" "$vspl" "$vwant"; sum_high=$((sum_high + 1))
  else
    printf 'INFO,%s,vendor_spl_ok,got=%s\n' "$dev" "$vspl"; sum_ok=$((sum_ok + 1))
  fi
done

printf 'SUMMARY,high=%s,warn=%s,low=%s,ok=%s,devices=%s\n' "$sum_high" "$sum_warn" "$sum_low" "$sum_ok" "$#"
```

**基线文件里写 `-` 表示"该机型不承诺厂商 SPL"**，脚本会把它报成数据缺口而不是通过——这类"无法断言"必须显式存在，否则 fleet 报表里会出现沉默的偏差。

**汇总行按断言条数而非设备数统计**：一台设备可能同时贡献 `HIGH` 与 `LOW`。要看设备数请回到设备字段，别把 `SUMMARY` 当设备计数。

---

## 十二、更新异常的取证信号

以下信号都不需要厂商配合，且都在只读范围内。它们的价值在于把"更新没到"与"数据采不到"区分开。

| 信号 | 观察方式 | 意味着什么 | 不能意味着什么 |
| --- | --- | --- | --- |
| SPL 晚于构建日期 | 脚本 1 `FIND_build_date` | 声明与构建产物不一致（第三方构建、混合分区、属性被改写） | 设备被攻击；也可能只是采集到非量产镜像 |
| 框架 SPL 新、厂商 SPL 旧 | `FIND_vendor_spl` 差值 | S3/S4 段未完成，厂商侧仍在旧基线 | 平台侧不安全；两侧本就是不同分区 |
| 同机型多设备 `incremental` 离散 | 台账分组 | 灰度批次差异；长期离散=有设备掉队 | 设备被篡改 |
| 厂商 SPL 属性缺失 | `L0-LOW` 记录 | 该机型不暴露该字段，或授权受限 | 厂商侧未更新（**必须回落到厂商公告核对**） |
| `/apex` 为空或极少 | `FIND_apex` | 采集受限，或该设备 Mainline 通道被厂商改动 | 组件必然过期 |
| 指纹中的区域/载体与资产台账不符 | 指纹字段核对 | 设备来源与登记不一致（转手、跨区采购） | 安全事件 |
| 首次 API 与当前 API 差 ≥2 | `FIND_lifecycle` | 该机型仍在大版本升级窗口内，安全更新可能被挤占（第 6.2 节） | 更新停滞 |

**取证顺序建议**：先分"数据可得性"（哪些字段采不到），再看"落后程度"（采到的字段差多少），最后才谈"风险"。把不可得当成安全，是设备池报告里最常见的一类错误结论。

---

## 十三、常见误区

1. **把 SPL 当成"已修复"的证明。** 它是构建的日期声明，覆盖范围还分框架/厂商两侧，且不覆盖基带与安全侧固件。
2. **比较 SPL 时不看档位。** 同一期公告有 `-01` 与 `-05` 两档，混在一起比会得出"两台设备同月即同安全水位"的错误结论。
3. **只看框架 SPL。** 厂商侧停更半年在这类报表上完全隐形——这也是第 9 章指标 2、3 单独存在的原因。
4. **用公告日当起点算暴露窗口。** USENIX 2021 的结论是 OEM 跟随月度节奏会造成不必要延迟；真实窗口要从设备上采。
5. **按品牌下更新结论。** NDSS 2024 与 2026 年 IEEE 的测量都指向"厂商内部、机型之间、区域之间"同样存在显著差异，品牌粒度不足以支撑处置决策。
6. **把更新承诺当购买合同的固定值。** 政策会延长也会终止，起算点是"该机型在美国 Google Store 首发"这类定义，不是购买日。
7. **忽视 Mainline 通道。** 组件可能已修而 SPL 未变，反之亦然；禁用 Play 系统更新的企业池要单独度量。
8. **拿芯片公告当机型结论。** 高通公告按技术域组织且自述用途是帮助客户纳入更新，联发科公告按"某些芯片组"描述——机型映射不公开是常态，需要向 OEM 索取，或用厂商 SPL 做代理指标并明确标注这是代理。
9. **把"采不到"记成"没问题"。** 属性缺失、授权受限、`/apex` 不可见都属于数据缺口，脚本 2 把它们单列为 LOW/MED 正是为此。
10. **用平均 SPL 汇报。** 风险由最旧的那批设备决定，尾部统计与平均数会得出相反结论。
11. **把内核更新与平台更新混为一谈。** GKI 按季度在 KMI 冻结后发布，内核侧到达节拍与月度公告不同，混在一起评估会系统性乐观。

---

## 十四、趋势判断（2024 → 2026）

- **通道继续增多，单一 SPL 的表述力继续下降。** 整机 OTA、Mainline 模块、GKI 三条节拍并存已是既成事实，治理指标必须按通道拆分，否则报表与真实暴露面脱钩。
- **支持期成为竞争维度且政策仍会变动。** Pixel 已把七年写成公开政策并回头延长过旧机型支持；三星从"四代 OS + 最长五年"走到 2024 年 1 月起的扩展支持。政策仍会被修改，任何以"承诺"为单位的长期风险模型都需要定期重采政策页。
- **大版本升级与安全更新的资源竞争是结构性的。** 厂商支持范围页把"OS 升级可能导致计划内安全更新延迟"写进条款；随着支持期变长，同一批设备要经历的升级次数增加，这类挤占会更频繁地出现在指标 6、7 里。
- **长周期设备的补丁流程会被标准化要求。** Google 的车载厂商指南明确其最佳实践"由 CTS 评估"，这个模式（把补丁应用流程变成兼容性可测项）向其他长生命周期品类扩散是可预期方向。
- **厂商侧不透明度不会自行改善。** 芯片公告按域组织、不给机型清单的现状短期难变；fleet 层面能做的只有两件事——要求 OEM 提供机型映射，以及用设备采集建立自己的代理指标。
- **硬件更新与硬件价值继续绑定。** 安全组件的实际防护取决于它的固件能不能推到设备上；硬件报告与本文是同一枚硬币的两面：**器件决定上限，更新决定实际值。**

---

## 十五、结论

补丁治理的问题从来不是"有没有公告"，而是**五段链条上每一段的可观测性与责任归属**。AOSP 侧每月公告、可回源码；GKI 季度节拍；芯片厂商公告按技术域组织且不提供机型清单；OEM 有自己的支持范围页与延迟条款；分发还有 Mainline 这条独立通道。把这五段画在同一张表上（第 3 章），绝大多数"这台设备安全吗"的问题就有了可执行的答案形式：**去采七个字段，再和对应公告核对。**

最小可执行三件事：

1. 按第 10 章脚本采集，指标 1–4 建基线（框架 SPL、厂商 SPL、两者差值、构建日期一致性）；
2. 按第 11 章脚本做 fleet 断言，把"无法断言"单列，绝不当作通过；
3. 按第 6 章读法登记机型支持政策，含政策 URL、起算点口径与采集日期。

其余一切（供应商评分、采购条款、合规说明）都应能从这三件事的数据里推出，而不是从宣传页里推出。

---

## 十六、常见问题（FAQ）

**Q1：设备显示本月 SPL，是不是就说明它安全了？**
不是。它说明构建声明包含该日期的平台侧修复；厂商侧要看 `vendor` 的 SPL，基带与安全侧固件不在 SPL 语义内，Mainline 组件版本要另行观察。

**Q2：为什么厂商 SPL 比框架 SPL 旧，不算异常？**
不算异常但算风险信号。两侧属于不同分区、不同上游（GKI 与厂商 SDK 各有节拍），差距是链条 S3/S4 段长度的直接读数，指标 3 就是为它设的。

**Q3：能不能用 SPL 判断一条具体 CVE 是否影响我的设备？**
只能做初筛。SPL 是"包含关系"标记；要确认单条缺陷，需要公告的组件归属 + 该机型固件说明，芯片侧缺陷通常只能向 OEM 索取映射。

**Q4：第三方 ROM 的 SPL 可信吗？**
把它当"声明值"处理，用脚本 1 的构建日期一致性与指纹字段交叉验证。第 2.1 节里第三方发布说明自述调整声明级别的例子，正说明这个数字在不同体系里的含义需要用产物佐证。

**Q5：企业禁用 Play 系统更新会带来什么度量问题？**
会让 Mainline 这条通道失效，但 SPL 可能仍显示较新，形成"看着很新、组件不动"的组合。指标 7 存在就是为了暴露这种情况。

**Q6：多久采一次？**
公告节奏是月度，采集至少月度；机型大版本升级窗口（第 6.2 节）期间加采一次，因为那段时间的延迟是结构性的。

**Q7：这些脚本会改动设备吗？**
不会。只使用 `getprop`、`toybox ls`、`get-state` 与本机 `date` 换算；正向断言，不含任何触发式探测、降级或写操作。

**Q8：本文哪些内容刻意没写？**
如何改写 SPL 属性、如何跳过更新校验、如何回退到旧固件、如何获取他人设备的更新控制权。这些属于对第三方设备造成损害的操作性内容；本文只覆盖"在你拥有或授权的设备上读取与核对"。

---

## 十七、参考资料（公开来源，本文事实均来自以下页面）

**公告与补丁体系（AOSP / Google）**

1. Android Security and Update Bulletins — https://source.android.com/docs/security/bulletin
2. Android Security Bulletin—December 2025 — https://source.android.com/docs/security/bulletin/2025-12-01 （两个补丁级别与其用意）
3. Android Security Bulletin—October 2025 — https://source.android.com/docs/security/bulletin/2025-10-01
4. Android Security Bulletin—July 2025 — https://source.android.com/docs/security/bulletin/2025-07-01 （"Security patch levels of 2025-07-05 or later address…"）
5. Android Security Bulletin—March 2026 — https://source.android.com/docs/security/bulletin/2026/2026-03-01 （"patch levels of 2026-03-05…"）
6. Android Security Bulletin—May 2026 — https://source.android.com/docs/security/bulletin/2026/2026-05-01
7. Version binding — https://source.android.com/docs/security/features/keystore/version-binding （厂商补丁级别存于只读属性、引导程序可提供校验数据哈希）
8. Manufacturer guide for long-term Android security — https://source.android.com/docs/automotive/security/mfg_guide （推荐最佳实践，由 CTS 评估）
9. Google System Services Release Notes — https://support.google.com/product-documentation/answer/14343500

**内核与模块化更新**

10. Kernel overview — https://source.android.com/docs/core/architecture/kernel （5.10 及以上称 GKI 内核）
11. Generic Kernel Image (GKI) release process — https://source.android.com/docs/core/architecture/kernel/gki-releases （KMI 冻结后按季度发布）
12. Android common kernels — https://source.android.com/docs/core/architecture/kernel/android-common
13. Develop kernel code for GKI — https://source.android.com/docs/core/architecture/kernel/kernel-code （先向 LKML 提交，再进 ACK `android-mainline`）
14. kernel/common — https://android.googlesource.com/kernel/common/ （上游优先的提交指引）
15. Mainline — https://source.android.com/docs/core/ota/modular-system （模块可由 Google 经 Play 系统更新推送，或由 OEM 推送）
16. Android Mainline Modules Transparency — https://developers.google.com/android/binary_transparency/mainline_modules/overview

**芯片厂商公告**

17. Security Bulletins | Qualcomm Documentation — https://docs.qualcomm.com/product/publicresources/securitybulletin （公告用途表述；按技术域组织条目）
18. Qualcomm February 2026 Security Bulletin — https://docs.qualcomm.com/securitybulletin/february-2026-bulletin.html （补丁与 OEM 共享的表述）
19. Qualcomm Product Security — https://www.qualcomm.com/company/product-security
20. MediaTek Product Security Bulletin — https://www.mediatek.com/product-security-bulletin （"影响某些联发科芯片组的安全漏洞详情"）
21. May 2026 MediaTek Product Security Bulletin — https://www.mediatek.com/product-security-bulletin/may-2026
22. December 2025 MediaTek Product Security Bulletin — https://www.mediatek.com/product-security-bulletin/december-2025

**厂商更新政策**

23. Samsung Mobile Security — Security Updates Scope — https://security.samsungmobile.com/workScope.smsb （自 2024 年 1 月扩展支持；OS 升级可能导致安全更新延迟）
24. Samsung Sets the New Standard With Four Generations of OS Upgrades — https://news.samsung.com/global/samsung-sets-the-new-standard-with-four-generations-of-os-upgrades-to-ensure-the-most-up-to-date-and-more-secure-galaxy-experience （四代 OS + 最长五年安全更新的旧口径）
25. Learn when you'll get software updates on Google Pixel phones — https://support.google.com/pixelphone/answer/4457705 （Pixel 8 及之后七年，自美国 Google Store 首发起算）
26. Google Pixel（生命周期聚合页，二手来源，仅作初筛）— https://endoflife.date/pixel （2024 年 12 月 Google 延长 Pixel 6/7 支持期的记录）

**缺陷与研究**

27. Multiple Internet-to-baseband Remote Code Execution Vulnerabilities in Exynos Modems — https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html （CVE-2023-24033、26496、26497、26498）
28. Hardening cellular basebands in Android — https://security.googleblog.com/2023/12/hardening-cellular-basebands-in-android.html
29. An Investigation of the Android Kernel Patch Ecosystem（USENIX Security 2021）— https://www.usenix.org/system/files/sec21summer_zhang.pdf
30. A Device-Centric Analysis of Android Security Updates（NDSS 2024）— https://www.ndss-symposium.org/wp-content/uploads/2024-175-paper.pdf
31. Unveiling the Global Landscape of Android Security Updates（IEEE Transactions on Quality, 2026）— https://ieeexplore.ieee.org/document/11370963/
32. GrapheneOS Releases（第三方构建的 SPL 声明实例）— https://grapheneos.org/releases

> 检索过程中确认**未公开文档化**、因此本文未采纳为事实的说法：厂商 SPL 属性在不同机型上的统一名称（AOSP 版本绑定文档给出 `ro.vendor.build.version.security_patch`，公开报道亦出现 `ro.vendor.build.security_patch` 写法，本文以 AOSP 文档为准并要求按机型实测）；高通/联发科公告中"芯片缺陷 → 具体机型"的公开映射；各 OEM 的 OTA 批次灰度比例与完成时间线；基带与安全处理器固件版本的统一可读字段；"季度维护版"作为独立发布物的现行官方页面（本文只采用公告中两个补丁级别的原始表述）。

---

*本报告基于截至 2026-10-26 的公开资料整理。公告口径、机型支持政策与厂商属性名称都会变化，任何结论在用于处置决策前都应在目标机型与目标固件上重新采集；本文脚本只做只读采集与正向断言，不含触发式探测、降级操作或利用性内容。*

---

咨询ios系统请咨询 telegram：https://t.me/one00190
