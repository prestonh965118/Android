---
layout: default
title: "Android 厂商硬件安全组件形态与实现差异：安全芯片、TEE 与密钥存储 | Android 安全报告集"
description: "对比 Android 设备上硬件信任根的落地形态——独立安全处理器与安全芯片、SoC 内安全世界、Trusty 与 OP-TEE 路线、StrongBox 与 eSE——并给出可外部核验的信号与不可核验的边界。防御性分析，不含利用代码与绕过步骤。"
lang: zh-CN
permalink: /Android-Hardware-Security-Components-and-Vendor-Implementations/
---
# Android 厂商硬件安全组件形态与实现差异分析报告

**Android Hardware Security Components and Vendor Implementations: Forms, Evidence and Verifiable Signals**

- **报告日期**：2026-10-26
- **报告性质**：公开资料整理与防御性硬件安全分析
- **适用对象**：移动平台安全负责人、OEM/ROM 安全工程、设备采购与企业移动管理（EMM/MDM）、第三方安全评估机构、密钥与凭证服务研发、检测与取证工程师
- 视角声明：本文写的是**同一套 Android 硬件安全要求在各厂商手里落成了哪些不同形态，以及这些形态里哪些能被外部核验、哪些不能**。硬件信任根内部的协议与机制（AVB 度量链、DICE/BCC、KeyMint 语义、Gatekeeper/Weaver、AuthToken、RKP、密钥证明字段逐条解读）在姊妹篇里，本文只做交叉引用而不重复：
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（AVB/DICE/KeyMint/keystore2/Gatekeeper/Weaver/AuthToken/FBE/证明/RKP 的机制与失效模式矩阵）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（六段分层、HAL/Treble、启动链与可信根、身份与隔离模型）
  - [Android Linux 内核结构与子系统分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（镜像与加载、设备节点供给、安全控制在内核中的落点）
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核攻击面盘点、厂商驱动的可达性、加固机制清单）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（HAL 进程间通道与权限判定的细节）
  - [Android 跨域攻击面地图与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域攻击面清点、SoC 厂商驱动与基带域、研究流程）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（固件与补丁怎么推到设备上、SPL 两侧字段与到达性核验——本文负责"器件与可核验信号"那一半）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本文讲器件形态与可外部核验信号；该篇讲无线器件——Wi-Fi/蓝牙组合芯片与基带调制解调器——的固件公告为什么常在厂商登录墙之后甚至已经下线，以及公告不可得如何落进它的四级判定模型）
- 范围声明：全文只讨论**公开文档化**的形态、厂商公开技术文档与认证材料、AOSP 公开代码与 Google 官方页面，以及公开公告/论文中的缺陷类别。**不提供任何提取密钥、绕过验证启动、解锁引导加载器、绕过设备完整性判定或规避防盗保护的操作步骤，也不提供漏洞利用代码、内存布局偏移或原语构造。**文中"核验"类内容一律指在你拥有或获授权的设备上执行只读检查，用于资产清点、采购评估与取证。

---

## 一、摘要

Android 对"硬件安全"的要求其实很薄：AOSP 定义了 KeyMint HAL 的安全级别语义、要求硬件后端密钥与度量根、给出验证启动状态与远程密钥配置的接口。**具体买什么芯片、把密钥放在哪里、隔离做到什么程度，全部留给厂商。**于是同一份兼容性要求，落地成了物理上完全不同的东西：有的设备把密钥放在主板上一颗独立的安全处理器里，有的放在 SoC 内部的一个安全世界，有的只有一份跑在普通内核里的软件实现——而应用侧看到的都是同一个 `KeyStore` API。

这带来三个只有硬件维度才会出现的问题，本文围绕它们展开：

1. **形态不可见。** 应用在绝大多数情况下无法区分"独立安全芯片"与"SoC 内安全世界"，只能读到一个 `SecurityLevel` 枚举值。而这两种形态在面对同一类攻击时的差别是本质性的：Google 对 Pixel 上 Titan M 的公开表述是"因为它是独立芯片，物理隔离……"（"Because Titan M is a separate chip, the physical isolation…"），高通对 SPU 的公开文档表述是"独立子系统，具备自己的引导程序与引导链、独立时钟"，三星对 Knox Vault 的公开表述是"安全处理器独立于运行 Android 的主 CPU，存储与主操作系统隔离"。**这些是架构差别，不是配置开关。**
2. **能力声明与证据混淆。** 抗故障注入、抗侧信道这类属性无法从设备外部核验，能核验的只有**认证记录**：高通 SPU 有 FIPS 140 安全策略文档与 Common Criteria（NSCIB-CC-0569293，SPU270）评估对象，英飞凌 OPTIGA Trust M 公开材料写明基于 CC EAL 6+ 认证的硬件，Google 对 StrongBox 的公开表述是执行在"至少 CC EAL4+ 带 AVA_VAN.5"认证的独立安全元件里，AOSP 硬件最佳实践文档则把 StrongBox 描述为"实现在防篡改硬件中"。**除认证与厂商工程文档外的营销描述，都不构成可引用的证据。**
3. **独立安全域不等于没有可达面。** 安全组件要工作就必须和 Android 侧通信，这条通道本身就是攻击面：高通 fastrpc 的竞态（CVE-2020-11277，公告口径按 HLOS / DSP Services 等技术域划分）、Exynos 调制解调器上一次性公开四个"网络可达→基带 RCE"级别缺陷（Project Zero 2023-03，CVE-2023-24033 / 26496 / 26497 / 26498），以及针对基带接口自动化提取的公开研究（BaseMirror，arXiv:2409.00475）。**"密钥在硬件里"与"硬件的入口有没有被审计"是两个独立命题**，采购评估里通常只问了前者。

对防御方与评估方，本文的交付物是三件：**形态判定表**（第 3–4 章：一台设备上的密钥究竟落在哪个形态上，怎么判断），**外部可核验信号清单**（第 5–7 章：`SecurityLevel`、Root-of-Trust 字段、验证启动状态、Play Integrity 判定、SPL，以及每项信号的适用边界与失效条件），**只读设备清点脚本与台账**（第 10–11 章，可直接用于设备池盘点）。

一句话结论：**硬件安全的差异不在"有没有硬件"，而在"硬件边界画在哪里、这条边界能不能被外部证明"。** 前者决定同一份漏洞在两台设备上的后果差别，后者决定你能不能把它写进评估报告。

---

## 二、把"硬件安全"拆成正交的四个维度

讨论厂商差异时最容易犯的错，是把四个正交的东西揉成一个词（"有安全芯片"）。本文自始至终用这四个维度描述任何一个形态：

| 维度 | 要回答的问题 | 可外部核验性 |
| --- | --- | --- |
| **D1 执行位置** | 密码运算与安全策略检查发生在哪个物理/逻辑单元里？同一 SoC 的安全世界、独立芯片、还是普通内核？ | 部分可（`SecurityLevel`、厂商文档） |
| **D2 隔离机制** | 该单元靠什么与 Android 侧隔开：硬件地址空间分区、独立总线/时钟域、独立引导链，还是仅靠软件权限分离？ | 基本不可（只有认证与工程文档） |
| **D3 密钥与度量存储** | 私钥/对称密钥、防回滚计数器、度量根存在哪里，掉电是否保留，能否被软件擦除或读取 | 部分可（证明扩展字段、eFuse 类属性的公开说明） |
| **D4 证明与信任传递** | 谁能把"我现在跑的是什么、锁着还是解锁"这件事告诉外部，验证方能不能拿到可验证链 | 可（密钥证明、`VerifiedBootState`、完整性判定） |

四个维度里，**只有 D4 是评估方能直接自证的**，D1/D3 只能间接读，D2 几乎只能靠认证材料。这条划分决定了本文后半部分的写法：能核验的给出字段与脚本，不能核验的给出"要问厂商要什么材料"。

顺带一个必要的澄清：**"硬件安全"不等于"硬件里的东西就安全"。** 硬件改变的是攻击成本与攻击类型（从软件利用变成需要设备、探头与故障注入），并不改变缺陷存在与否。所以下文所有形态描述都伴随一个对应的"坏在哪一环能看见什么"。

---

## 三、形态类型学：一台 Android 设备上可能存在的四种落地

### 3.1 F1 独立安全处理器 / 安全芯片（discrete secure processor / SE）

**特征**：D1 上位于主 SoC 之外（或至少是具备独立引导链与独立时钟域的独立子系统）；D2 提供的是物理层面的分离；D3 通常包含防篡改的非易失存储与硬件计数器；D4 通过一组受限命令与 Android 侧交互。

公开可引用的三个实例：

- **Google Titan M**：Android Developers 官方博客《Building a Titan》对 Pixel 3 上 Titan M 的表述是"显著缩小攻击面，因为它是独立芯片，物理隔离带来……"。这是"独立芯片＝架构级属性"最直白的一处公开说法。（Pixel 后续机型上的 Titan M2 属于同一产品线的演进；本文不展开型号级差异，理由见第 12 章误区一节。）
- **Qualcomm SPU（Secure Processing Unit）**：高通文档对 SPU240 的定义是"独立的安全处理单元，内含处理器核、RAM 与集成密码单元"，其面向移动平台的安全白皮书进一步表述为"专用子系统，具备独立引导程序与引导链、独立时钟"。认证侧：SPU 有公开的 FIPS 140 安全策略文档（NIST CSRC 发布），SPU270 有公开的 Common Criteria 评估对象文档（NSCIB-CC-0569293，经 TÜV Nederland 发布），其中描述其处理器提供内存保护以在安全 OS 中隔离互不信任的应用。
- **Samsung Knox Vault**：三星的公开说明把它定义为"专用安全硬件（一颗新的安全处理器与隔离的安全内存）"，并明确"在 TrustZone 提供的保护之上扩展""安全处理器独立于运行 Android 的主 CPU""存于 Knox Vault 的数据与主操作系统隔离"。**注意这条表述本身就在说 D1/D2 的差别：TrustZone 是同一处理器内的分区，Knox Vault 是另一个处理器。**

### 3.2 F2 SoC 内安全世界（ARM TrustZone 型 TEE）

**特征**：D1 与 Android 同处理器，靠处理器的安全/普通世界切换与总线隔离提供分区；D3 依赖 SoC 内部存储或外挂 RPMB；D4 通过 TEE 里的 TA 实现 KeyMint/Gatekeeper 等。

AOSP 自己的默认实现就在这一类：官方文档描述 **Trusty** 是"为 Android 提供 TEE 的安全操作系统"，并指出 Trusty OS **运行在与 Android 相同的处理器上**；配套还有用户态库经内核驱动与可信应用通信。另一条公开路线是 **OP-TEE**：AOSP 有公开的 `tee/optee/ta/keymint` 仓库，自述为"KeyMint reference implementation for OP-TEE"，即基于 OP-TEE 可信应用的 Rust 参考实现（使用 Apache Teaclave 的 TrustZone SDK）。

这一类的差异不在"是不是安全世界"，而在**谁实现了 TA、TA 里有没有把关键检查做完**——这是同一形态内部质量差别最大的一处，第 6 章展开。

### 3.3 F3 强隔离执行环境（pKVM / AVF 型）

D1 仍在 SoC 与主 CPU 上，但隔离由虚拟化层而不是处理器安全世界提供，客户机（protected virtual machine）的内存对宿主不可见。它与 F2 的关系不是替代而是不同用途：安全世界保护的是系统服务与密钥执法，虚拟化隔离保护的是"给某个租户一块宿主看不见的内存"。**逐条机制在信任根姊妹篇的"执行环境的三种形态"一章**，本文不重复，只在形态判定表里保留这一列，因为它会改变"谁的内存可读"这一取证前提。

### 3.4 F4 通用安全元件（eSE / 智能卡型）与 StrongBox 的认证门槛

这一类常被与 F1 混为一谈，区别在于**用途**：eSE 类器件面向凭证/支付/身份应用（APDU 交互、受管应用），F1 面向操作系统自身的信任根。二者可以物理同器件。

可核验的边界由 Google 的公开表述给出：StrongBox 执行在"专用的、物理隔离的安全元件（SE）中，该 SE 至少通过 Common Criteria EAL4+ 带 AVA_VAN.5 认证"；AOSP 硬件最佳实践文档则把"同时支持实现在防篡改硬件中的 StrongBox KeyMint"列为**推荐项而非强制项**。也就是说：

- **有 StrongBox 这个枚举值 ≠ 器件达到该认证门槛**；评估要问的是具体器件与认证号，而不是设备上是否出现了 `STRONGBOX`。
- 反过来，独立器件供应商侧的证据是公开的：例如 OPTIGA Trust M 的公开材料写明其安全控制器基于 CC EAL 6+（high）认证硬件，并提供非易失存储与对称/非对称运算能力——这类"器件级认证 + 平台采用"的配对信息，才是能写进评估报告的材料。

### 3.5 判定表：给定一台设备，怎么知道它属于哪一类

| 观察项 | 取值/来源 | 只能证明什么 | 不能证明什么 |
| --- | --- | --- | --- |
| KeyMint 请求的 `SecurityLevel` | 设备侧属性、`keystore2` 侧公开信息、应用查询结果 | 该实现自报运行在 TRUSTED_ENVIRONMENT / STRONGBOX / SOFTWARE 哪一级 | 该级的物理隔离是否达到厂商宣称的程度 |
| 是否存在 STRONGBOX 后端 | 同上 | 设备暴露了一个更强后端 | 后端是独立 SE 还是同 SoC 的另一分区 |
| 密钥证明中的 Root-of-Trust 字段 | 证明证书扩展（见信任根姊妹篇） | 引导状态与设备是否锁定、度量值 | 度量值由哪颗芯片产生（除非该器件把来源写进证书链） |
| `VerifiedBootState` / 设备管理侧枚举 | 启动属性、Android Management API 的 `VerifiedBootState`（GREEN/YELLOW/ORANGE/RED） | 引导链校验结论与锁定情况 | 校验根落在哪颗芯片、根密钥吊销状态 |
| Play Integrity 判定 | `BASIC` / `DEVICE` / `STRONG` 与 `appIntegrity` 各取值 | Google 侧对设备与应用的综合判断 | 硬件形态、隔离强度（它是判定结果，不是架构描述） |
| 厂商公开文档与认证号 | 器件数据手册、FIPS/CC 证书与评估对象 | D1/D2/D3 的设计意图与认证范围 | 你手上这批设备的固件是否处于认证版本 |

**这张表的用法**：第 1–4 行合起来足以支持"这台设备的密钥不在普通内核里"这一结论；要支持"这台设备的密钥在一颗独立安全处理器里"，只能靠第 6 行，而第 6 行的结论必须写成"依据厂商公开文档，该型号采用 X 器件（认证 Y）"，并附文档名。

---

## 四、外部可核验的四类信号与各自的失效条件

这一章是全文的支点：硬件形态本身看不见，但有四类信号是任何评估方都能采集的。每一类都写清楚"能证明到哪一步"和"什么情况下它会骗你"。

### 4.1 `SecurityLevel`：最省事也最容易被高估的一格

KeyMint HAL 在密钥生成/导入时接受安全级别参数，可枚举值的语义是"这个密钥的执法者在哪一层"：软件实现、受信执行环境（TEE）、StrongBox。**设备侧可以看到某个后端被注册为哪一级**，因此这是判断"这台设备上密钥执法落在 F2 还是 F1/F4"的第一手数据。

- **能证明**：该设备的 KeyMint 实现声明了自己的位置；请求是否被路由到了 StrongBox 后端。
- **失效条件**：①它是实现自报的枚举，不携带物理描述；②AOSP 硬件最佳实践文档把 StrongBox 表述为"推荐（recommended）支持"，因此缺失 StrongBox 只说明厂商没做，不说明其 TEE 弱；③部分设备只在特定密钥类型上提供 StrongBox（容量、运算类型与器件资源限制），"有 StrongBox"不等于"所有密钥都在里面"。
- **写法要求**：评估报告里应写"该设备对 X 类密钥暴露 STRONGBOX 后端"，不应写"该设备使用独立安全芯片"。

### 4.2 验证启动状态：`GREEN / YELLOW / ORANGE / RED`

AOSP 的验证启动流程文档（Boot flow）把状态按类别给出：**YELLOW** 对应"仍为锁定（LOCKED）但使用了自定义信任根"的情形，**ORANGE** 对应"引导加载器已解锁（UNLOCKED）"，**RED** 对应校验失败。Android Management API 侧同一套枚举以 `VerifiedBootState` 形式提供（`GREEN`/`YELLOW`/`ORANGE`/`RED`，逐个给出定义），因此企业可以不经 adb、直接从设备管理接口采集。

- **能证明**：本次启动的校验结论，以及引导加载器锁的状态结论。
- **失效条件**：它是**启动时**的判定，不持续反映运行时完整性；它描述的是"链有没有验过"，不描述"验链的根在哪颗芯片里"。解锁设备（ORANGE）的密钥证明仍然可以合法存在，只是 Root-of-Trust 字段会变， relying 方要处理的是"状态变化"而不是"状态为橙即一切作废"。
- **取证价值**：`ORANGE`/`YELLOW` 是二手设备与发放设备验收中最便宜的一条异常信号，成本近零。

### 4.3 Root-of-Trust 字段与密钥证明：把 4.1 和 4.2 变成可验证链

密钥证明（attestation）证书里携带 Root-of-Trust 扩展：引导状态、设备是否锁定、各分区度量值与校验密钥。它的价值在于**把设备自述变成第三方可以验证的声明**——验证方拿厂商根的证书链核签名。逐字段解读在信任根姊妹篇，本文只强调与硬件形态相关的一条：**证明链里出现的根证书，其签发方是谁，就是你能够客观记录下来的"信任根归属"信息。**如果一台设备的证明链根指向某厂商的 attestation 根，这就是公开可核验材料能支持的最强结论；再往前（"因此该器件抗故障注入"）就没有证据了。

### 4.4 Play Integrity 判定：服务侧的综合结论

Google 的 Play Integrity API 返回分级判定（`BASIC` / `DEVICE` / `STRONG`）以及描述应用与设备状态的字段，其中包括引导加载器锁状态相关的取值；官方文档解释了判定 JSON 的格式与字段含义，2024-12 的官方博客说明了该 API 在速度、抗攻击与隐私上的调整，企业侧公告则记录了判定要求的行为变化（例如要求判定为 `STRONG` 时，应用必须由 Google Play 安装或更新）。

- **能证明**：Google 在该时刻对该设备+该应用组合的判定结论；服务方可以据此拒绝高风险环境。
- **失效条件**：**它是判定，不是架构。**同一个"独立安全芯片"设备可能因为镜像来源、分区状态或应用安装渠道拿不到 `STRONG`；反之，判定通过也不代表密钥一定在 StrongBox 里。把它当硬件能力清单来写进评估报告，是这一领域最常见的一类错误。

---

## 五、同一形态内部的质量差：四项最容易被忽略的审计点

形态分类给出的是"上限"，实际防护由下面四点决定。这四点在跨厂商比较里差别最大，且多数不出现在营销材料里。

### 5.1 TA 实现是否真的做完检查

`SecurityLevel` 为 TRUSTED_ENVIRONMENT 只说明密钥的执法代码运行在安全侧。真正决定强度的是这个 TA 有没有把**用户认证绑定、调用方身份绑定、用途限制、防回滚计数**全部实施。AOSP 把这些语义定义在 KeyMint 接口与硬件后端要求里，因此**审计方法是可执行的**：构造接口层允许/拒绝矩阵，观察 TA 是否拒绝，而不是读它的宣传。这也是本系列反复强调的"策略即契约"在硬件侧的对应物。

### 5.2 时间与计数源

带超时锁定的密钥（如要求用户认证后 N 秒内有效）依赖安全侧的**可信时间**。Google 对 StrongBox 的公开说明中把"具备与安全认证时间对齐的安全时间源"列为条件之一。跨厂商差异在这一项上极常见：有些器件有独立振荡器与电池域，有些依赖宿主写入时间，后者的超时语义在宿主被控制时会退化。**评估问题应该写成："该 StrongBox 的认证超时在系统时间被篡改时如何表现？"**

### 5.3 密钥的可用性与迁移

硬件后端密钥的常见后果是：换 SoC/换器件即不可解密，恢复出厂与换机迁移需要专门设计（密钥栈姊妹篇覆盖 keystore2 命名空间与 LSKF 链）。跨厂商的差别体现在是否支持密钥证明的密钥轮换、是否支持 RKP（不再向工厂预置私钥）——这两项决定了密钥栈在设备生命周期里的运维成本，而它们在设备侧是可采集的。

### 5.4 回滚保护与吊销

防回滚（anti-rollback）与根密钥吊销通常落在器件的一次性可编程存储与引导程序策略里。可核验的做法是：**观察固件降级是否被拒绝、观察降级后依赖硬件后端的密钥是否仍然可用**。这是行为测试，不需要读任何内部实现，也是唯一能把"厂商声称的 eFuse 能力"变成结论的方式。

---

## 六、安全组件自己的可达面：三条通道

独立安全域不改变一个事实：它必须与 Android 侧通信。AOSP 对 Trusty 的公开描述里就包含这条路径——Android 用户态通过一个库经**内核驱动**与可信应用通信。这意味着硬件安全组件的入口同时落在内核、厂商驱动和跨域消息通道上。

| 通道 | 为什么必然存在 | 公开证据 | 防御侧要问的问题 |
| --- | --- | --- | --- |
| **TEE 驱动 / 内核接口** | 普通世界要请求安全侧服务，就要有设备节点与 ioctl 语义 | AOSP Trusty 文档描述了经内核驱动与可信应用通信的用户态库；本系列内核报告与内核结构报告覆盖节点供给与可达性判定 | 该节点对 `untrusted_app` 是否可打开？输入校验与安全侧调用参数检查分别在哪一层？ |
| **DSP / 跨处理器消息通道** | 数字信号处理器与非通用核有自己的服务接口，需与 HLOS 交换请求 | 公开 CVE 记录：CVE-2020-11277 描述 fastrpc 异步会话中的竞态（上下文在异步会话期间被释放）；高通安全公告按 HLOS、DSP Service 等技术域组织条目（公告索引页自述用途是"帮助客户在已发布与将发布的设备中纳入安全更新"） | 该域的缺陷修复进入哪一级补丁？厂商公告里该组件有没有对应到具体机型清单？ |
| **基带 / 调制解调器** | 蜂窝协议栈必须常驻且网络可达 | Project Zero 2023-03 对 Exynos 调制解调器公开四项网络可达→基带代码执行级缺陷（CVE-2023-24033、26496、26497、26498）；Google 安全博客 2023-12 说明了对 Android 蜂窝基带的加固方向；BaseMirror（arXiv:2409.00475）公开了基带接口的自动化提取思路 | 基带固件版本能否独立于系统更新观察？该机型是否已进入基带停止维护名单？ |

一张表里三条通道的共同点是：**它们都属于"硬件安全"话题，却几乎从不出现在硬件安全的宣传材料里。**评估一个设备的安全组件而不审计它的入口，等价于评估一座保险库而不看它连着几条传送带。

---

## 七、生命周期与所有权变化时的语义

硬件安全组件的价值集中在两个时刻：设备被转手时，和设备被判定为不可信时。这里有三条容易被误用的语义，评估与取证都需要它们。

### 7.1 锁定状态改变的不只是"能不能装别的系统"

引导加载器从锁定变为解锁，公开可观察的后果是 `VerifiedBootState` 由绿转橙，且密钥证明携带的 Root-of-Trust 字段随之改变。**依赖硬件密钥的服务方应当按"信任级别变化"处理**，而不是把设备整体拉黑——这也解释了为什么解锁状态是一个可被远程核验的中性事实，而不是一个"成功/失败"的判定。

本文不提供任何改变锁定状态的操作路径。改变锁定状态属于设备所有者的管理动作，但把"如何绕过他人设备的保护"写成步骤会直接落入对第三方的损害风险，超出防御性分析的范围。

### 7.2 二手与发放设备的三条最小核验

在只读前提下，这三条组合已经能覆盖绝大多数"这台设备被动过"的场景：

1. `VerifiedBootState` 与设备锁状态（4.2）；
2. 密钥证明里 Root-of-Trust 的度量值与该厂商公开固件版本是否一致（4.3）；
3. SPL / 补丁字符串与厂商公开公告口径是否对得上（下一章与更新治理姊妹篇交接）。

三条都便宜、都可脚本化（第 10–11 章给出脚本），且都不需要厂商配合。

### 7.3 "厂商自定义信任根"与 ROM 的边界

YELLOW 状态对应的正是"锁定但使用自定义信任根"的情形，这类设备常见于第三方 ROM 与自建设备池。它们在安全上不是"更弱"，而是**证明链不再指向厂商根**，于是所有依赖远程证明的判定都要重新约定。企业设备池里出现这类设备，应在资产台账上显式标注，而不是让它在 MDM 报表里表现为"正常"。

---

## 八、跨厂商差异矩阵：把"宣称"换成"要问的问题"

下面这张矩阵不给结论（结论受第 4 章四类信号约束），给的是**评估时真正能区分设备的六个问题**，以及每个问题的证据来源类型。按这六个问题问，比按品牌问有效得多。

| 维度 | 要问的问题 | 合格证据 | 不合格证据 |
| --- | --- | --- | --- |
| **D1 执行位置** | StrongBox/密钥后端运行在哪一类器件上？该器件的公开型号与数据手册是什么？ | 器件数据手册、厂商工程文档（例：高通对 SPU240 的定义、三星对 Knox Vault 的定义、Google 对 Titan M 的定义） | "军工级""银行级"等形容词 |
| **D2 隔离机制** | 安全侧是否有独立引导链与时钟域？与普通世界共享哪些硬件？ | 白皮书级描述（例：高通公开材料表述 SPU 为"独立引导程序与引导链、独立时钟"的专用子系统） | 产品页 banner |
| **认证范围** | 该器件是否已获 FIPS 140 / Common Criteria 认证？认证的版本与安全策略文档覆盖哪些功能？ | 公开证书与安全策略/评估对象文档（例：SPU 的 FIPS 140 安全策略、SPU270 的 NSCIB-CC-0569293 评估对象、OPTIGA Trust M 公开材料标注 CC EAL 6+ 硬件） | "通过 XX 认证"但没有证书号 |
| **认证门槛一致性** | 该设备的 StrongBox 是否满足 Google 公开表述的最低门槛（SE 且 CC EAL4+ 带 AVA_VAN.5）？ | 认证号 + 固件版本对应关系 | 仅有 `STRONGBOX` 枚举值 |
| **时间源** | 认证超时依赖的安全时间由谁提供？在宿主时间被篡改时行为如何？ | 接口级行为测试（第 9.2）+ 厂商说明 | 无 |
| **入口审计** | 安全侧与普通世界之间的通道由谁审计过？该域缺陷修复走哪条补丁链？ | 公告组件归属（高通公告按 HLOS/DSP Service 等技术域组织）+ 机型清单 | "安全团队自测" |

**关于"认证"的一条必要澄清**：Common Criteria 与 FIPS 140 证明的是**某个版本在某个评估环境下满足某个保护轮廓**，不是"该器件没有漏洞"，也不是"该机型用的就是这个版本"。评估报告里正确的一句是"该器件型号存在证书 X，覆盖版本 Y；本次采集设备固件为 Z，两者对应关系需厂商确认"。

---

## 九、不可核验属性怎么处理：把它变成行为测试

D2（隔离强度）与器件物理特性无法从外部读，但可以**用可观察行为设定下限**。以下三类测试全部只读、全部在自有或授权设备上进行，且不构成对任何组件的攻击步骤。

### 9.1 后端一致性测试（不构造密钥材料，只做接口层观察）

对同一请求分别指定 TRUSTED_ENVIRONMENT 与 STRONGBOX 后端，观察三件事：请求是否被接受、返回的证明证书链根是否变化、密钥句柄的属性回读是否一致。**结论形式**："该设备对 X 算法族提供/不提供 StrongBox 后端"。这是设备池验收里最有价值的一条，因为它直接反驳"整机型通用"的错误结论。

### 9.2 超时与时间源行为测试

生成一个绑定用户认证、带较短有效窗口的密钥，在窗口内与窗口外各使用一次，记录接受/拒绝。若测试设备支持禁用自动时间（`persist.sys.timezone` 与自动时间属性可读出），可把时区/自动时间与拒绝行为做相关记录。**结论形式**："该后端对认证超时的执行不依赖宿主时间"（或反之）。这是第 5.2 节问题的实测答案。

### 9.3 降级拒绝观察

对可正常 OTA 的设备，比较"允许回退"的机型与"不允许回退"的机型在版本回退请求上的响应（这是系统更新机制的行为，不是绕过）。可回退的机型意味着**防回滚计数器没有覆盖该系统分区**，该事实对硬件后端的密钥可用性有直接影响（换固件后密钥是否仍在）。**结论形式**：只写"回退是否被拒绝 + 回退后既有硬件密钥是否仍可用"。

三项测试的共同点：**它们测的是"厂商宣称"与"设备实际"之间的差，而不是器件物理强度。**硬件强度仍然要靠认证，行为下限要靠这三项。

---

## 十、检测与取证之一：硬件安全形态只读采集

脚本 1 用于把第 4 章四类信号一次性采成结构化输出，可直接进设备台账。它只做 `getprop`、`ls`、`settings get` 三类只读操作，不写入设备、不安装任何东西。

```bash
#!/usr/bin/env bash
# 脚本 1: 硬件安全形态可核验信号采集（只读）
# 用法: hw-profile.sh <设备序列号>
# 输出: key,value 形式的行；可直接重定向为台账文件
set -u
dev="${1:?用法: hw-profile.sh <serial>}"

adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable" >&2; exit 1; }

p() { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }
q() { adb -s "$dev" shell "toybox $1" 2>/dev/null | tr -d '\r'; }
row() { printf '%s,%s\n' "$1" "$2"; }

model="$(p ro.product.model)";      row model "$model"
brand="$(p ro.product.brand)";      row brand "$brand"
api="$(p ro.build.version.sdk)";    row api "$api"
first="$(p ro.product.first_api_level)"; row first_api "$first"
spl="$(p ro.build.version.security_patch)"; row spl "$spl"
fwp="$(p ro.build.version_incremental)";    row incremental "$fwp"
fp="$(p ro.build.fingerprint)";             row fingerprint "$fp"

vb="$(p ro.boot.verifiedbootstate)";        row verifiedbootstate "$vb"
dl="$(p ro.boot.vbmeta.device_state)";      row device_state "$dl"
vbmajor="$(p ro.vbmeta.major_version)";     row vbmeta_major "$vbmajor"
vbminor="$(p ro.vbmeta.minor_version)";     row vbmeta_minor "$vbminor"
size="$(q "df -k /vendor 2>/dev/null" | tail -1 | awk '{print $2}')"
row vendor_size_kb "$size"

km="$(p ro.hardware.keystore2)";     row keystore2_hal "$km"
sc="$(p ro.hardware.gatekeeper)";    row gatekeeper_impl "$sc"
wea="$(p ro.hardware.weaver)";       row weaver_impl "$wea"
se="$(p ro.hardware.secure_element)"; row secure_element_impl "$se"
cpuabi="$(p ro.product.cpu.abi)";    row cpu_abi "$cpuabi"
oemunlock="$(adb -s "$dev" shell settings get global oem_device_unlock_allowed 2>/dev/null | tr -d '\r')"
row oem_unlock_setting "${oemunlock:-null}"

tee_nodes="$(q "ls /dev" | grep -i -E 'tee|rpmb' | wc -l | tr -d ' ')"
row tee_node_count "${tee_nodes:-0}"

if [ -n "$vb" ]; then
  case "$vb" in
    green)  row FIND_lock_state "L1-INFO 校验通过且锁定" ;;
    yellow) row FIND_lock_state "L2-WARN 锁定但使用自定义信任根，远程证明链不再指向厂商根" ;;
    orange) row FIND_lock_state "L2-WARN 引导加载器解锁，Root-of-Trust 字段随之变化，依赖硬件密钥的服务应按信任级别变化处理" ;;
    red)    row FIND_lock_state "L3-HIGH 校验失败，设备不应作为可信发放对象" ;;
    *)      row FIND_lock_state "L2-WARN 未知状态值，需按 AOSP 状态类别核对" ;;
  esac
else
  row FIND_lock_state "L0-LOW 未采集到状态，可能非量产固件或属性被裁剪"
fi

if [ -n "$se" ]; then
  row FIND_strongbox "L1-INFO 设备声明了安全元件后端实现，认证范围与证书号需向厂商索取"
else
  row FIND_strongbox "L2-WARN 未声明安全元件后端；该机型 StrongBox 覆盖情况需按算法族实测（第 9.1 节）"
fi

case "$size" in
  ''|*[!0-9]*) row FIND_vendor_freshness "L0-LOW 无法读取 /vendor 尺寸，该机型可能未挂载该分区" ;;
  *)
    if [ "$size" -gt 15728640 ]; then
      row FIND_vendor_freshness "L2-WARN vendor 分区超过 15GB：厂商定制层体量过大，回归与补丁 rebase 成本相应上升"
    else
      row FIND_vendor_freshness "L1-INFO vendor 分区体量在常见范围内"
    fi ;;
esac

if [ "$api" = "$first" ] && [ -n "$first" ]; then
  row FIND_lifecycle "L1-INFO 出厂 API 与当前 API 一致，主版本未升级"
else
  row FIND_lifecycle "L2-WARN 出厂 API($api) 与 first_api_level($first) 不一致，主版本已变化，硬件后端行为需重新核对"
fi
```

**这张输出怎么读**：前三组（型号/SPL/指纹、锁状态/vbmeta、后端实现）是**事实**；后面 `FIND_*` 是**判定**，每条判定都能追溯到上面的事实字段。台账里两者都要保留——只留判定会在复评时无法复核。

---

## 十一、检测与取证之二：批量核验与基线断言

设备池评估的瓶颈不是采集，是**同一型号多台设备的一致性**与**证据缺失时的处理**。脚本 2 把第 4 章四类信号变成断言，缺失项单独报数据质量，不当成"通过"。

```bash
#!/usr/bin/env bash
# 脚本 2: 硬件安全形态批量核验与基线断言（只读）
# 用法: hw-verify.sh <基线文件> <设备序列号>
# 基线文件每行一条 key=value；key 取自脚本 1 的输出字段
set -u
base="${1:?用法: hw-verify.sh <baseline-file> <serial>}"
dev="${2:?缺少设备序列号}"

adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable,$dev" >&2; exit 1; }
[ -f "$base" ] || { echo "ERR,baseline_missing,$base" >&2; exit 1; }

p() { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }
q() { adb -s "$dev" shell "toybox $1" 2>/dev/null | tr -d '\r'; }
get() {
  case "$1" in
    verifiedbootstate) p ro.boot.verifiedbootstate ;;
    device_state)      p ro.boot.vbmeta.device_state ;;
    spl)               p ro.build.version.security_patch ;;
    model)             p ro.product.model ;;
    secure_element)    p ro.hardware.secure_element ;;
    gatekeeper)        p ro.hardware.gatekeeper ;;
    tee_nodes)         q "ls /dev" | grep -c -i -E 'tee|rpmb' ;;
    *)                 printf '' ;;
  esac
}

while IFS='=' read -r key want; do
  case "$key" in ''|'#'*) continue ;; esac
  got="$(get "$key")"
  if [ -z "$got" ]; then
    printf 'LOW,%s,missing,%s,数据缺失，不得记为通过\n' "$dev" "$key"
    continue
  fi
  if [ "$got" != "$want" ]; then
    printf 'HIGH,%s,%s_mismatch,got=%s,want=%s\n' "$dev" "$key" "$got" "$want"
  else
    printf 'INFO,%s,%s,match=%s\n' "$dev" "$key" "$got"
  fi
done < "$base"

vb="$(get verifiedbootstate)"
case "$vb" in
  green) printf 'INFO,%s,lock_state,green\n' "$dev" ;;
  yellow) printf 'MED,%s,lock_state,custom_root_of_trust\n' "$dev" ;;
  orange) printf 'MED,%s,lock_state,bootloader_unlocked\n' "$dev" ;;
  red)   printf 'HIGH,%s,lock_state,verification_failed\n' "$dev" ;;
  *)     printf 'LOW,%s,lock_state,unparsed_value=%s\n' "$dev" "${vb:-empty}" ;;
esac

se="$(get secure_element)"
if [ -z "$se" ]; then
  printf 'MED,%s,strongbox_backend,not_declared\n' "$dev"
else
  printf 'INFO,%s,strongbox_backend,declared=%s\n' "$dev" "$se"
fi

tn="$(get tee_nodes)"
if [ "$tn" -gt 0 ] 2>/dev/null; then
  printf 'INFO,%s,tee_node_count,%s\n' "$dev" "$tn"
else
  printf 'LOW,%s,tee_node_count,zero_or_unreadable\n' "$dev"
fi

if [ -z "$(get gatekeeper)" ]; then
  printf 'LOW,%s,gatekeeper_impl,empty\n' "$dev"
else
  printf 'INFO,%s,gatekeeper_impl,%s\n' "$dev" "$(get gatekeeper)"
fi
```

**为什么断言层与判定层会各输出一条**：上半段是"与基线不符"（事实层，随基线文件变化），下半段是"这条不符意味着什么"（语义层，与设备型号无关）。台账里两层都要留，只留事实层会让复核的人重新解释一遍，只留语义层会让基线变更后无法回溯。

**为什么把"数据缺失"单列成 LOW 而不是跳过**：设备池里最常见的错误结论来自属性被裁剪或机型未挂载分区时，脚本"没报错"被读成"没问题"。缺失必须显式落到台账上。

**`tee_node_count` 的正确用法**：安全侧驱动节点的存在与否由内核与厂商决定，数量本身不是安全属性；这个字段的价值在于**同型号之间应当一致**，异常差异（例如某台设备多了未预期的设备节点）才是需要跟进的信号，具体可达性判定方法在内核报告与内核结构报告里。

---

## 十二、常见误区

1. **把品牌名当形态。** "有 Knox"、"有 Titan"、"有 SPU" 说的是产品线，不是隔离边界。正确的表述必须落到器件与文档：独立安全处理器、SoC 内安全世界、虚拟化隔离环境、eSE 器件，四者对同一攻击的后果不同。
2. **把 TrustZone 与"安全芯片"混用。** AOSP 对 Trusty 的公开描述明确它运行在与 Android 相同的处理器上；三星 Knox Vault 的公开描述则明确它是"独立于主 CPU 的安全处理器"。**同一个词（TEE）覆盖两种物理边界**，写进评估报告时必须分开。
3. **看到 `STRONGBOX` 就写"独立安全芯片"。** `SecurityLevel` 是实现自报的枚举，AOSP 硬件文档把 StrongBox 列为推荐项，Google 公开表述给出的是认证门槛（SE + CC EAL4+/AVA_VAN.5）。要把枚举值和门槛声明区分开，门槛需要认证号支撑。
4. **把 Play Integrity 判定当硬件能力表。** 它是服务侧对"设备+应用"的综合判定，会因安装渠道、分区状态变化；判定通过不等于密钥在 StrongBox，判定失败不等于硬件被拆过。
5. **认为"未解锁＝安全，已解锁＝全废"。** 正确表述是：解锁使 `VerifiedBootState` 变为橙色、Root-of-Trust 字段变化，**依赖远程证明的服务方应按信任级别变化处理**，本地硬件密钥的可用性并不因此归零。
6. **忽略 YELLOW 这一档。** 锁定但使用自定义信任根（常见于第三方 ROM 与自建设备池）意味着证明链不再指向厂商根；这类设备如果在企业台账里显示"正常"，是资产数据错误而不是设备问题。
7. **把认证证书当"无漏洞证明"。** FIPS 140 与 Common Criteria 的结论范围是"某版本、某评估环境、某保护轮廓"。证书存在≠你手上这批固件在认证版本内。
8. **只审器件，不审入口。** 安全组件必须与 Android 侧通信：TEE 走内核驱动与用户态库，跨处理器服务走消息通道（公开 CVE 记录里就有 fastrpc 异步会话竞态 CVE-2020-11277），基带网络可达（Project Zero 2023-03 一次公开四个 Exynos 调制解调器缺陷）。**"密钥在硬件里"与"硬件接口被审计过"是两个独立结论。**
9. **拿型号做安全对比。** 型号级事实（某代机型用哪颗器件、承诺几年更新）需要持续维护，且同一型号不同区域/载体的实现可能不同。本文刻意不做型号对比表；评估要用第 10 章的采集脚本在**手上这台设备**上取数，而不是引用别人的机型结论。
10. **把 `tee_node_count` 当风险分数。** 节点存在与否由内核与厂商决定，数量不是安全属性；它的正确用法是**同型号一致性检查**——异常差异才是需要跟进的信号。

---

## 十三、趋势判断（2024 → 2026）

**已经发生的三件事**

- **独立安全域从"密钥存放"扩展到"凭证签发"。** Google 把 StrongBox 与高保证数字凭证（开放标准下的可验证凭证）放在一起论述，并把 SE 的认证门槛写进公开表述——这意味着硬件后端的选型不再只影响加密库性能，而开始影响设备能否承接需要强设备绑定的凭证业务。
- **虚拟化隔离进入常规形态。** 执行环境从"TEE 里"扩展到"pKVM/AVF 隔离的客户机里"，本系列信任根报告已覆盖机制侧。对硬件形态判定的影响是：以后"这段代码在硬件保护下运行"会有第三种含义，评估时必须追问是哪一种。
- **基带与安全域的公告口径变厚。** Google 公开说明对蜂窝基带的加固方向，芯片厂商公告按技术域（HLOS、DSP Service 等）组织条目并说明其用途是帮助客户在已发布与将发布设备中纳入更新。这类结构化口径是"补丁传导链"能被度量的前提，详细分析见更新治理姊妹篇。

**未来 12–24 个月的判断（概率表述，非承诺）**

- **判定与硬件进一步分离。** 服务方会越来越依赖综合判定（完整性判定、设备风险信号）而不是硬件属性清单；相应地，"我们用的是有安全芯片的机型"这种论证在合规材料里会贬值，能被第三方验证的证明链会升值。
- **认证门槛会成为采购条款。** 一旦"StrongBox 需 SE + CC EAL4+/AVA_VAN.5"这类门槛被公开表述，采购问卷就会开始问证书号；厂商若不公开对应器件与版本，会在企业采购里遇到实质阻力。
- **不可核验属性会被行为测试替代。** 第 9 章那类只读行为测试（后端一致性、超时与时间源、降级拒绝）会成为验收标准动作，因为它比宣传材料便宜且可复现。
- **硬件差异与更新治理会绑定评估。** 一颗安全处理器的实际价值取决于它的固件能不能被推到设备上；厂商若把安全侧固件更新与系统更新解耦，"有硬件、不更新"的设备会大量出现。这正是下一篇（补丁传导链与厂商更新治理）要处理的问题。

---

## 十四、结论

Android 的硬件安全差异不在有没有硬件，而在三件可分别核验的事：**密钥执法的物理位置（D1）、该位置是否有可追溯的认证（D2/认证范围）、以及这个域对外声明的结论能不能被第三方验证（D4）**。三者相互独立，任一成立都不能替另外两个背书。

本文能给出的可执行结论是：

- 形态判定靠四类信号（`SecurityLevel`、验证启动状态、Root-of-Trust 证明字段、完整性判定）加厂商公开文档与认证号，其中只有后两类是"硬件架构"证据，前四类是"设备行为"证据；
- 不可核验的属性用三项只读行为测试设下限（后端一致性、超时与时间源、降级拒绝），不要用形容词；
- 安全组件的入口（TEE 驱动、跨处理器通道、基带）必须与器件一起评估，公开缺陷记录已经证明这些入口会出问题；
- 台账要保留事实字段与判定字段两层，判定必须能回溯到采集值，否则复评无法进行。

对采购与企业设备管理，最短路径是：**先按第 10 章脚本采一遍手上设备，再按第 8 章矩阵的六个问题向厂商要材料。**顺序反过来的评估报告，通常是在复述厂商的营销页。

---

## 十五、常见问题（FAQ）

**Q1：我怎么知道公司发的这批手机里哪些真的有 StrongBox？**
用脚本 1 采集 `ro.hardware.secure_element` 与后端相关属性，再按 9.1 做后端一致性测试（对同一请求分别指定两种安全级别，观察是否被接受、证明链根是否变化）。单一属性或单一型号说明都不足以下结论。

**Q2：TrustZone 是不是就不算"硬件安全"？**
它是硬件提供的隔离机制，但边界在同一颗处理器内部。AOSP 对 Trusty 的公开描述就是运行在与 Android 相同的处理器上，而 Knox Vault / Titan M / SPU 的公开描述都强调独立处理器或独立引导链。**两者都算硬件安全，但抗攻击类型不同，报告里要分开写。**

**Q3：设备解锁了还能继续用它的硬件密钥吗？**
本地可用性通常仍在，但远程可验证性下降：`VerifiedBootState` 变橙、Root-of-Trust 字段改变，依赖证明的服务方会按信任级别变化处理。正确做法是在服务侧按字段判定，而不是简单拒绝所有解锁设备。

**Q4：厂商说他们的安全芯片"通过国际认证"，我应该要什么？**
要三样：器件型号与数据手册、证书编号与认证计划（FIPS 140 还是 Common Criteria）、以及证书覆盖的版本与评估对象范围。只有营销页而无证书号，按第 8 章矩阵属于不合格证据。

**Q5：为什么本文不给机型对比表？**
型号级事实需要持续维护，同型号在不同区域与载体的实现也可能不同，一处过期就会让整页失去可信度。更重要的是，机型结论替代不了对**手上这台设备**的采集（第 10 章）。

**Q6：基带和安全芯片有什么关系？**
两者都是"Android 之外的处理器域"，也都各有对外通道。它们的风险模型不同（安全域被动请求服务、基带网络常驻可达），但评估方法相同：先确定通道，再问该通道谁审计过、缺陷修复走哪条补丁链。

**Q7：这些采集脚本会改动设备吗？**
不会。脚本只用 `getprop`、`toybox ls/df`、`settings get` 三类只读命令，正向断言，不做触发式探测，也不写入任何状态。

**Q8：本文哪些说法是刻意没写的？**
如何改变引导锁状态、如何绕过完整性判定、如何提取或迁移硬件密钥、如何降级固件。这些属于对第三方设备造成损害的操作性内容，超出防御性分析范围；本文只在"锁定状态变化后服务方该如何处理"这一侧给出建议。

---

## 十六、参考资料（公开来源，本文事实均来自以下页面）

**AOSP / Google 官方**

1. Hardware security best practices — https://source.android.com/docs/security/best-practices/hardware （StrongBox KeyMint 为推荐项、"实现在防篡改硬件中"）
2. Hardware-backed Keystore — https://source.android.com/docs/security/features/keystore （KeyMint HAL 与硬件后端密钥）
3. Trusty TEE — https://source.android.com/docs/security/features/trusty （经内核驱动与可信应用通信的用户态库）
4. Android security features — https://source.android.com/docs/security/features （Trusty 是运行在与 Android 相同处理器上的安全 OS）
5. KeyMint reference implementation for OP-TEE — https://android.googlesource.com/tee/optee/ta/keymint/ （AOSP 公开仓库，Rust 参考实现）
6. Boot flow（验证启动状态类别）— https://source.android.com/docs/security/features/verifiedboot/boot-flow （YELLOW=锁定但自定义信任根，ORANGE=解锁，RED=校验失败）
7. VerifiedBootState — https://developers.google.com/android/management/reference/rest/v1/VerifiedBootState （设备管理侧同一套枚举）
8. Returned integrity verdict format — https://developer.android.com/google/play/integrity/verdicts （判定 JSON 与取值）
9. Overview of the Play Integrity API — https://developer.android.com/google/play/integrity/overview
10. Making the Play Integrity API faster, more resilient, and more private — https://android-developers.googleblog.com/2024/12/making-play-integrity-api-faster-resilient-private.html
11. Building a Titan: Better security through a tiny chip — https://android-developers.googleblog.com/2018/10/building-titan-better-security-through.html （Titan M 为独立芯片、物理隔离缩小攻击面）
12. How Android secures high-assurance digital credentials — https://blog.google/security/android-strongbox-and-open-standards-digital-credentials/ （StrongBox 在物理隔离 SE 中，至少 CC EAL4+ 带 AVA_VAN.5）
13. Google Pixel privacy and security features — https://safety.google/intl/en-GB_au/products/pixel/ （Pixel 分层安全与自动安全更新的官方表述）
14. Hardening cellular basebands in Android — https://security.googleblog.com/2023/12/hardening-cellular-basebands-in-android.html

**厂商公开文档与认证材料**

15. Qualcomm secure processing unit — https://docs.qualcomm.com/doc/80-PV086-5P/topic/qualcomm-secure-processing-unit.html （SPU240：独立处理单元，内含核、RAM、集成密码单元）
16. Guard your data with the Qualcomm Snapdragon mobile platform — https://www.qualcomm.com/media/documents/files/guard-your-data-with-the-qualcomm-snapdragon-mobile-platform.pdf （独立引导程序与引导链、独立时钟）
17. Qualcomm® Secure Processing Unit (SPU) Hardware Security Policy（FIPS 140）— https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3549.pdf
18. Qualcomm SPU270 Security Target Lite（NSCIB-CC-0569293）— https://www.tuv-nederland.nl/assets/files/cerfiticaten/2023/02/nscib-cc-0569293-st.pdf
19. Qualcomm Mobile Security Solutions — https://www.qualcomm.com/products/features/mobile-security-solutions
20. Qualcomm Security Bulletins — https://docs.qualcomm.com/product/publicresources/securitybulletin （按 HLOS / DSP Service 等技术域组织；公告用途表述）
21. Infineon OPTIGA Trust M — https://github.com/Infineon/optiga-trust-m-overview/blob/main/README.md 与 https://www.infineon.com/part/OPTIGA-TRUST-M-MTR （CC EAL 6+ 认证硬件、离散安全器件、非易失存储）
22. Understanding Samsung Knox Vault — https://www.samsungmobilepress.com/articles/editorial-understanding-samsung-knox-vault-protecting-the-data-that-matters-most （在 TrustZone 之上扩展；安全处理器独立于主 CPU）
23. Samsung Brings Next-Level Security to Galaxy Devices with Samsung Knox Vault — https://news.samsung.com/global/samsung-brings-next-level-security-to-galaxy-devices-with-samsung-knox-vault （Knox Vault 存储与主操作系统隔离）
24. Real-time Kernel Protection（Samsung Knox 白皮书）— https://docs.samsungknox.com/admin/fundamentals/whitepaper/samsung-knox-mobile-security/system-security/real-time-kernel-protection/
25. Samsung Knox security principles — https://www.samsung.com/africa_en/sustainability/security-and-privacy/security/

**缺陷与研究（公开公告与论文）**

26. CVE-2020-11277（fastrpc 异步会话竞态）— https://www.cve.org/CVERecord?id=CVE-2020-11277
27. Multiple Internet-to-baseband Remote Code Execution Vulnerabilities in Exynos Modems — https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html （CVE-2023-24033、26496、26497、26498）
28. BaseMirror: Automatic Reverse Engineering of Baseband Interfaces to Discover Critical Vulnerabilities — https://arxiv.org/html/2409.00475
29. A Large-Scale Analysis of Secure Key Storage in Android — https://arxiv.org/html/2507.07927v1 （硬件后端密钥的定义性特征：密钥在独立于主操作系统的硬件中存储与使用）

> 检索过程中确认**未公开文档化**、因此本文未采纳为事实的说法：Pixel 上 Titan M/Titan M2 与某供应商器件的对应关系（媒体广泛转述，但本文未在一手来源中核实，故不写）；Titan M2 的官方规格页与逐条能力清单；Knox Vault 与安全元件的器件级对应型号；三星引导锁熔断位（warranty fuse）的官方技术定义；`android.hardware.secure_element` 的供应商实现差异与共享密钥配对的公开文档页；各厂商 StrongBox 密钥配额与安全时间源实现细节。以上均需厂商公开材料或认证文件才能作为结论，本文一律标注为"需向厂商索取"。

---

*本报告基于截至 2026-10-26 的公开资料整理。同一型号在不同区域、载体与固件版本上的硬件后端实现可能不同，任何结论在用于采购、合规或防护决策前都应在目标设备上重新采集核对；本文脚本只做只读采集与正向断言，不包含触发式探测、绕过验证或利用性内容。*

---

咨询ios系统请咨询 telegram：https://t.me/one00190
