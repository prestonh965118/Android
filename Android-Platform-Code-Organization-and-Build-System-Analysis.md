---
layout: default
title: "Android 平台代码组织与构建体系分析报告"
description: "AOSP 源码供给（repo/manifest/分支口径）、构建流水线（Soong/kati/Ninja）、产物装配（分区/out 目录）、签名与身份（test-keys/指纹/AVB）、内核与模块化构建（ACK/GKI/APEX）以及兼容性约束（VINTF/FCM/CDD/CTS/VTS/STS）的防御性分析与构建身份核验流程"
lang: zh-CN
permalink: /Android-Platform-Code-Organization-and-Build-System-Analysis/
---
# Android 平台代码组织与构建体系分析报告

**Android Platform Source Organization and Build System: A Defensive Analysis of Code Supply, Assembly, and Build Identity**

- **报告日期**：2026-10-26
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：ROM/OEM 系统与构建工程师、供应链与发布治理负责人、移动平台安全评估机构、设备资产管理与合规团队、检测与取证工程师、采购与安全基线制定者
- 视角声明：本文写的是**从"上游代码"到"设备上运行的镜像"这段供给链的组织方式与身份形成规则**——谁提供了哪些项目、以什么分支提供、构建如何把这些装配成分区镜像、身份与可信度在哪一步被写入、又在哪一步丢失。**运行时机制、协议词表、逐条 CVE 与设备侧攻击面不在本文**，那些内容在姊妹篇里，本文只做交叉引用而不重复展开：
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（六段分层、进程与 IPC、SELinux/seccomp/AID 身份；本文只讲这些分层**是怎样被构建出来的**）
  - [Android Linux 内核结构与子系统分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（ACK/GKI 内核形态与子系统落点；本文只讲内核的**分支命名与构建/交付口径**）
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核缺陷类型学与提权取证）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（AVB/DICE/KeyMint 与失效模式；本文只讲 AVB 作为**构建产物绑定动作**的那一段）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（SPL 两侧字段与五段传导链；本文讲这两侧字段**为什么来自两次不同的构建**）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（可达面清点与研究流程；本文的"构建身份"是它的一个横切属性）
  - [Android 远程代码执行攻击面与防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（投递通道与缓解矩阵）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（本文讲 `user`/`userdebug`/`eng` 构建身份从哪来、被谁签发，该篇讲这一身份在设备上落到可调试性与 `ro.debuggable` 之后，应用沙箱的执行域与 UID 会一起放宽到什么程度）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（本文讲构建身份从哪来、被谁签发，该篇讲同一套身份在应用侧变成哪些可在 APK 清单上静态观察的弱点信号，以及弱点分类按哪五套公开口径换算）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本文讲 APEX 与 Mainline 模块怎样被构建、装配与签名；该篇把 `com.android.wifi` 与可选的 `com.google.android.bt` 当**更新通道**而不是构建产物来使用，说明模块清单能证明什么、不能证明什么）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（本文写 `CONFIG_*` 与编译选项怎样被装配进产物、构建能否归因；该篇的"内核编译期缓解层"直接依赖本文的构建可复现性口径，反之不重复展开缓解语义）
  - [Android Project Mainline 与 APEX 模块缺陷归属分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Project-Mainline-and-APEX-Module-Defect-Attribution-Analysis.html)（本文写编译选项与产物怎样被装配、构建能否归因；该篇只在装配这一处引用本文口径，重点反过来：设备侧读数与构建产物之间**为什么不能互相归因**）
- 范围声明：全文只讨论**公开文档化**的仓库组织、构建工具链、镜像装配、签名与兼容性检查规则，以及在这些公开规则之上可以安全实施的清点与核验流程。**不提供任何伪造签名、绕过验证启动、篡改构建产物或复刻他人固件的操作步骤。**文中所有"如何核验"类内容均指自有代码库/自有设备的自查、供应商评估与检测覆盖建设；脚本全部为只读采集或离线文本对照，不写入设备、不修改构建、不安装策略。

---

## 一、摘要：这条链决定的是"能不能归因"，不是"能不能防住"

Android 的运行时防护（SELinux、seccomp、沙箱、CFI）解决的是"漏洞被触发之后能走多远"。构建体系解决的是另一个问题，而且是一个经常被安全团队忽略的问题：**当设备上跑着一份固件时，谁能说出它是从哪些源码、在什么时候、由谁、用哪把钥匙装配出来的。**

这两个问题的答案互相独立。一个正确启用了全部运行时缓解、但用默认 test-keys 签名、框架侧与厂商侧来自两次不同装配、发布流水线没有留存构建描述的固件，在事故发生时的可归因程度可能接近于零；反过来，一个缓解项偏旧但身份链完整的固件，至少能被判定"该修哪一份、推给谁"。

本文给出的是这条链的六段结构与每段的**可外部核验信号**：

1. **源码供给**（repo + manifest + 分支口径）——决定了"这次构建吃了哪些上游项目"是可清点的还是黑箱；
2. **构建流水线**（Soong/kati/Ninja）——决定了模块定义是否可静态枚举；
3. **产物装配**（out 目录 → 分区镜像）——决定了某个库或可执行文件属于哪个镜像、由谁加载；
4. **签名与身份**（test-keys、target files 签名、指纹）——决定了设备侧读到的构建身份是否有发布流水线背书；
5. **启动绑定**（AVB/VBMeta）——决定了构建身份能不能在设备上被硬件根验证链条续上；
6. **兼容性与模块化**（VINTF/FCM、APEX/Mainline、CTS/VTS/STS、CDD）——决定了这份构建产物能以什么粒度被单独更新或单独验证。

三条最重要的结论：

- **AOSP 侧的公开口径是"可核验"，不是"可复现"。** 官方为 Pixel 提供的是 Binary Transparency 日志（帮助使用者验证设备上运行的系统或固件对应哪份已记录的二进制），公开页面中**没有**对 AOSP 或 Pixel 镜像作出"重新构建可得到同一份产物"的可复现构建承诺。把"开源"读成"可复现"是这条链上最普遍的一个误判（详见第二、十九节）。
- **框架侧与厂商侧是两个构建身份，不是一份。** SPL 之所以有两个字段、指纹之所以有多个来源，是因为 system 侧与 vendor 侧在装配阶段本就是不同的产物；任何只看单个字段的合规判定都会漏掉"两侧来自不同批次"这种情况（脚本 1 的 `FIND_spl_split` 与 `FIND_fp_split` 就是为此存在）。
- **构建期就能消除的风险，成本远低于运行期检测。** manifest 项目清单的漂移、test-keys 泄漏进发布构建、内核以二进制形态交付造成的审计缺口——这三类都能在发布前用文本级对照发现，而事后在设备侧只能观察到结果而不是原因。

---

## 二、先分清三件事：可归因、可核验、可复现

安全团队讨论构建体系时最常用的三个词其实是三个不同强度的性质。把它们分清，后面的判断才不会走偏。

| 性质 | 它在回答什么 | 这条链上由什么提供 | 缺失时是什么表现 |
| --- | --- | --- | --- |
| **可归因（attributable）** | 这份产物是谁、在什么时候、从哪些源码做出来的 | manifest 项目清单与分支、构建描述与指纹、签名密钥归属、CI 留存 | 出事后只能对设备固件做二进制比对，无法指认某个源码提交 |
| **可核验（verifiable）** | 设备上跑的东西能否被证明属于某个已记录的合法集合 | AVB/VBMeta 启动度量、厂商或 Google 的透明度日志 | 设备侧只能读到 `verifiedbootstate` 一档粗粒度结论，无法回指具体二进制 |
| **可复现（reproducible）** | 第三方用同样的源码重新构建，能否得到逐字节相同的产物 | 需要构建环境本身被完整描述并去随机化 | 无法独立重构建比对；只能信任发布方留存的那一份 |

**关键区分**：公开文档给出的是前两者的支撑，第三者在这条链上**不是 AOSP 的公开承诺**。GKI 的表述甚至是反向的——内核"以二进制形态构建并交付"（原文见参考资料 20），即厂商侧不重新编译内核主体。这在构建可核验性上不是缺陷（KMI 稳定与 hermetic manifest 描述了构建环境），但在"任何人都能重算一遍"这个意义上确实留下了缺口。

对防御侧的实际含义：

- 若目标是**事故定位与补丁归因**，只需要"可归因 + 可核验"，这两项是任何厂商都能做且应当做的；
- 若目标是**独立验证某份发布固件未被替换**，在没有可复现构建承诺的情况下，可行路径是透明度日志 + 二进制比对 + 签名链，而不是"我自己重编一份对一下"；
- 采购条款如果把三项混写成"要求可复现构建"，供应商几乎必然给出一个无意义的答案。

---

## 三、源码供给层：repo、manifest 与"一棵树其实是几百个项目"

AOSP 的源码供给不是"一个巨大的仓库"，而是**清单驱动的聚合**。官方对 repo 的定位一句话讲清了机制："Repo uses manifest files to aggregate Git projects into the Android superproject."（manifest 文件把 Git 项目聚合成 Android 超级项目）——这意味着"这次构建吃了哪些代码"这件事，被一个清单文件完整描述。

从防御角度看，这个设计给出三个可清点的单元：

| 清点单元 | 它是什么 | 为什么是安全单元而不是工程单元 |
| --- | --- | --- |
| **项目集合**（manifest 列出的项目） | 参与本次构建的仓库全集 | 项目数与项目名的变化 = 上游供给面变化；新增一个项目就是新增一个可引入未审查代码的入口 |
| **每项目的分支/revision** | 该项目从哪个分支取 | 同一项目切到不同分支，等于换了一条上游；这比"源码有没有改"更早地解释差异 |
| **manifest 自身的来源与版本** | 客户端初始化时使用的 manifest（`.repo/` 下留存） | 如果 manifest 来源不可信或不可追溯，前两项全部失去意义 |

初始化 repo 客户端会建立一个 `.repo/` 目录，官方描述为："This command creates a .repo/ directory with Git repositories for the Repo source code and the standard Android manifest files."——这个目录是**构建可归因性的物理载体**。一个常见的评估失误是只检查源码树而忽略 `.repo/`：项目清单和分支信息恰恰在那里，而不在源码目录里。

三点结构性判断：

1. **"源码组织"的可枚举性来自 manifest，不来自目录树。** 因此对厂商的评估问题应该是"能否提供本次发布的 manifest 与 `.repo/` 快照"，而不是"能否给源码目录结构图"。本次核验中没有找到一份 AOSP 官方的"源码树目录结构"页面（见第二十二节边界清单），这也侧面说明目录布局本身不是被文档固定下来的契约。
2. **AOSP 主服务器上的清单是聚合点。** 该服务器首页对其 manifest 项目的描述是："Manifest that contains all the projects that are hosted on the AOSP server."——这份"全部项目"的清单，是任何人做上游漂移对照的基准输入。
3. **构建体系的"读一次、用多次"特性让 manifest 变成高风险对象。** manifest 是构建的第一份输入；它一旦被替换或改写，下游所有阶段都会忠实地装配错误的内容。这就是脚本 2 存在的理由：把 manifest 当作**配置项**做版本对照，而不是当作构建细节忽略掉。

---

## 四、分支口径：android-latest-release、aosp-main 与"只读镜像"

2025 年 3 月是 AOSP 分支口径的一个分界点。官方变更记录的表述是："Starting March 27, 2025, the latest release branch will always be referenced by the new android-latest-release manifest, which can be used directly with Repo."同一份措辞也出现在 platform/manifest 项目里被检出的默认清单文件中，也就是说它同时是文档说明与构建输入的一部分。

这条变更对安全侧有三个直接后果：

| 事实（公开口径） | 对"源码供给可归因性"的影响 |
| --- | --- |
| 最新分支由 `android-latest-release` 清单指代；FAQ 表述为"The most recent branch is considered the current stable branch version, which the android-latest-release manifest points to." | 归因时必须记录"当时 `android-latest-release` 解析到了哪个分支"，只写"取最新代码"无法复现 |
| FAQ 明确："No, the code won't be merged to aosp-main, which is a read-only branch as of March 27, 2025." | `aosp-main` 与发布分支之间不存在自动合并；把两者混为一个"主干"会造成"我以为这个修复在里面"的判断错误 |
| 分支随发布滚动 | 一个长期不重新解析分支的内部基线，等价于在跟踪一条已不再被指代的历史分支——这是 SPL 落后之外另一种"落后" |

**"read-only"这个词在 AOSP 语境里的含义值得单独强调**：它描述的是镜像的合并方向（不往该分支合并），而不是产物可信度或受保护程度。把"只读分支"读成"经过额外审核的分支"是常见误读（第十九节误区 2）。

对评估流程的实际要求：任何"我们基于 AOSP 最新代码"的表述，都应被替换为三项可落库的记录——**manifest 来源、解析到的分支名、解析时间**。这三项组成一条供给记录，缺一项就不能用于事后归因。

---

## 五、构建流水线：Soong、kati、Ninja 的分工

AOSP 的构建执行分三段，官方一句话给出结构："AOSP uses the Soong build system to build Android. Soong uses the kati GNU Make clone tool and Ninja build system component to speed up builds of Android."

```text
模块声明                转换/生成                    执行
Android.bp ──┐
             ├─► Soong（Blueprint 层） ─► 生成 .ninja ─► Ninja 并发执行 ─► 产物
Android.mk ──┘        ▲
                      └─ kati：把遗留 Make 文件纳入同一张依赖图
```

Soong 的声明文件在文档中有明确称谓："Soong build files are called blueprint files"；迁移工作也被当作工具能力公开描述："Soong includes a tool perform a first pass at converting Android.mk files to Android.bp files"（原文如此，见参考资料 34）。

这段流水线给防御侧的不是运行时属性，而是**三份可静态枚举的清单**：

| 清单 | 从哪里取 | 用途 |
| --- | --- | --- |
| 模块全集 | `Android.bp` / `Android.mk` 的模块声明 | 回答"某个库/可执行文件是不是本次构建的合法产物"；也是"设备上多出来的二进制"判定的基线 |
| 依赖图 | Soong 生成阶段的输出 | 回答"某个第三方 blob 被哪些模块依赖"，用于收缩供给面 |
| 动作序列 | 生成的 Ninja 文件 | 是"这次构建做了什么"的最细粒度记录，比 CI 日志更接近事实 |

两个判断：

1. **"模块可枚举"是这条链上性价比最高的加固点。** 一旦某次发布的模块全集被留存，设备侧任何无法回指到模块全集的可执行文件都成为高价值异常信号——不需要任何运行时防护配合。
2. **遗留 Make 路径的收敛不会自动改善安全。** kati 的存在是为了兼容与速度；把 Android.mk 转成 Android.bp 的收益是构建可分析性提升，不是缺陷减少。以"完成 Soong 迁移"作为安全 KPI 会误导资源投向（第十九节误区 6）。

---

## 六、构建目标三元组与变体语义：一个字符串里装着两条策略

AOSP 的构建目标由 `envsetup.sh` 提供的环境函数选择。文档对顺序有强制表述："You must run source envsetup.sh once per shell and before the lunch command because lunch is defined by envsetup.sh"——`lunch` 不是可执行文件而是 shell 函数，这也是它必须先 source 的原因。

目标字符串是三段式（产品—发布配置—变体）。**变体那一段直接决定设备的调试面**，是三处最容易出问题的地方：

| 变体 | 文档定位 | 安全侧读法 |
| --- | --- | --- |
| `user` | "This build variant provides limited security access and is suited for production." | 唯一可作为发布形态接受的值 |
| `userdebug` | 面向设备开发者的调试构建 | 出现在发布固件里即为构建身份异常，无论功能是否被人为关闭 |
| `eng` | 更快的构建、面向工程迭代 | 同上，且通常伴随更宽的调试属性 |

三段式的第三段还与另一个设备侧字段绑定：`ro.build.tags` 记录该构建使用何种密钥类型（发布密钥或 AOSP 测试密钥）。官方文档确认树内含测试密钥："The Android tree includes test-keys under build/target/product/security ."这两段合起来（变体 + 密钥类型）构成**设备侧最容易读到、也最难伪装的构建意图声明**：脚本 1 的核心判定全部建立在这两项加指纹一致性上。

关于 `m`：官方说明是"The m command builds from the top of the tree, so you can run m from within subdirectories."——工程便利背后有一个可归因性影响：**在子目录里发起的构建不改变产物归属，但经常导致 CI 记录的发起上下文不完整**。发布构建应当固定从顶层发起，并把工作目录、目标三元组与工具链版本一起入日志。

---

## 七、源码只读原则与 out 目录：为什么"第一现场"不在源码树里

构建阶段的一条硬规则值得原样引用："During the build, the AOSP source tree is read-only."产物落在别处："The output of your build appears in $OUT_DIR ."

这条规则对安全工程的实际意义有三层：

1. **任何"为了修安全问题直接改源码树内产物"的临时做法都不成立。** 变更必须体现在模块声明或构建参数上——这是好事，因为它让安全修复天然可 diff、可回溯。反过来说，若一个组织内部确实存在"手工替换 out 里的某个二进制再打包"的做法，那它就是这条链上最隐蔽的断点：源码与产物不再由同一次构建绑定。
2. **审计的第一现场是 out 目录与 target files，不是源码树。** 源码树只读意味着所有装配结果（分区镜像、模块清单、生成的依赖信息）都留在 out 侧；评估一个厂商时，要求其提供 target files 层级的产物清单，比要求源码访问权限更能说明问题。
3. **产物与源码的可归因性依赖留存。** 源码只读不等于"产物可回指源码"。只有当构建描述（含指纹与增量号）、manifest 快照与 out 产物被一同归档时，才能事后指认"这个镜像来自这批提交"。这三份留存是第十五节风险清点里的 R3 项。

---

## 八、分区装配：super、动态分区与核心厂商分区

构建的最后阶段是"把产物装配成设备可刷写的分区"。两条公开口径定义了装配边界：

- 厂商侧核心分区："The vendor , vendor_boot , and odm partitions are core vendor partitions."
- 动态分区的组织方式："The super partition contains metadata listing the names and block ranges of each dynamic partition within super ."

这段机制对安全侧的价值是**归属问题**：某个库、某个 HAL 实现、某个守护进程属于哪个分区，直接决定它由哪条更新通道维护、由谁的签名背书、以及是否能在不动其他分区的情况下被替换。这与[补丁传导链报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)中"两侧 SPL 来自不同维护主体"的结论是同一条链的两端：分区归属是因，SPL 两档是果。

三个装配期的观察：

| 装配期事实 | 安全侧推论 |
| --- | --- |
| `super` 用元数据描述内部各动态分区的名称与块范围 | 分区表元数据本身是构建产物的一部分；对它的变更（尺寸、增删分区）应纳入发布差异审查 |
| 核心厂商分区只有 vendor/vendor_boot/odm 三项 | 任何超出这三项的"厂商专属分区"都处在标准兼容性约束之外，是评估时需要单独问清的对象 |
| 动态分区使"只更新其中一段"成为可能 | 收益是修复可以更小步到达；代价是"整包一致性"不再是天然保证，需靠启动链度量维持（第十节） |

**一个容易被忽略的判定**：如果框架侧与厂商侧来自不同批次构建，那么 `super` 里两者的组合就是一个**非标准装配**。它可能完全正常（分阶段构建的现代流程本就如此），但在事故发生时，"这份组合是否被发布流水线正式产出并归档"是必须能回答的问题。脚本 1 对这件事只给 WARN 而不给 HIGH，正是因为它需要人工确认而非直接定性。

---

## 九、签名与构建身份：target files 签名、指纹与 test-keys

签名环节把"一次构建"变成"一次发布"。公开的机制口径是："The signing script sign_target_files_apks works on the target files generated for a build."——签名动作的对象是 target files（构建产物集合），而不是零散的镜像文件。这一句给出两个安全侧结论：

1. **签名密钥的归属就是发布权的归属。** 谁持有发布密钥，谁就能产出被设备接受的"合法下一版"（前提是启动链的信任锚也在其手中）。密钥治理因此属于本文这条链，而不是属于密钥栈那篇——后者讲的是设备侧的密钥服务与证明，此处讲的是**发布侧的产物签名**。相关设备侧机制见[硬件信任根与密钥栈报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)。
2. **构建身份字段是在打包与签名阶段确定的，因此它可以与源码状态不完全同步。** 这意味着 `ro.build.fingerprint` 类字段是可"配置"的：流水线可以记录真实值，也可以记录一个便于管理但不真实的值（例如固定字符串）。任何把指纹字段直接当作"版本真相"的合规判定都必须先确认厂商的写入规则——这也是本文所有指纹相关判定都只给 INFO/WARN 而不给 HIGH 的原因。

AOSP 自带的 test-keys 是这条链上最容易被忽视的失误形态。它本身不是漏洞，只是开发默认值；但当它出现在**对外发布的构建**里，后果是三项同时失效：

- 无法归因（无法证明该产物来自某个受控流水线）；
- 无法追责（密钥是公开的，任何人都能产出同签名身份的下一版）；
- 无法核验（启动链的信任锚不成立，具体见第十节）。

设备侧对这件事的读数成本极低——`ro.build.tags` 一项即可——所以它必须是**发布门禁**而不是事后审计项（脚本 1 的 `FIND_tags` 把 test-keys 定为 L3-HIGH）。

---

## 十、启动绑定：AVB 把构建产物接到可信根上

构建身份要变成设备上的实际约束，靠的是验证启动的元数据。AVB 2.0 的公开结构描述里可以看到这一点："Following the hashtree is a VBMeta struct which contains the hashtree descriptor with hashtree metadata (root hash, salt, offset, etc.)"——即每个受验证分区以 hashtree 形式被描述，其根哈希等元数据写进 VBMeta 结构。

从构建体系的视角看，这给出三个可操作事实：

| 事实 | 构建侧动作 | 设备侧可观测 |
| --- | --- | --- |
| 分区的根哈希在构建期确定 | 若构建后手工替换分区内容，VBMeta 与内容不再匹配 | `verifiedbootstate` 与锁状态是粗粒度读数（脚本 1） |
| VBMeta 需要被签名链锚定 | 锚定用的密钥归属决定"谁能签发合法下一版" | 设备上无法读到密钥归属本身 |
| 元数据是"结构"而非"策略" | 结构可核验；策略（失败时是否阻断）由厂商实现决定 | 需要厂商文档或实测才能知道阻断行为 |

**这条链的断点在这里最清楚**：构建产物有根哈希 ≠ 有可归因性。如果签发 VBMeta 的密钥就是 AOSP test-keys，那么"验证通过"只证明内容未被随机损坏，不证明内容来自某个负责任的发布方——因为任何人都能用同一把公开密钥签发。第九节与第十节合起来才是完整的构建身份：**签名给出归属，AVB 给出不可篡改，两者缺一。**

关于设备侧究竟"验证到什么程度"，属[系统架构](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)与[信任根](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)两篇的范围，本文不展开。

---

## 十一、APEX 与 Mainline：构建产物如何变成可独立更新的单元

模块化把"一次装配"拆成了"多个可独立交付的单元"。两条公开口径界定了机制与目的：

- 容器与签名：官方 APEX 文档给出了文件级签名的属性入口——"To sign an APEX at the file-level, set the certificate property in one of these three ways:"，即 APEX 有独立的文件级签名属性，而不是仅依赖所在分区的内容完整性。
- 更新意图："Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle."

对这条链的影响是**签名主体与更新通道随模块而分裂**：

| 单元 | 谁签名 | 走哪条更新通道 | 构建侧的可归因性要求 |
| --- | --- | --- | --- |
| 分区内普通模块 | 随整包签名 | 整机固件更新 | 一次构建、一份归档即可 |
| APEX 模块（设备内置） | 文件级签名属性 | 随整机或模块化通道 | 需要保留每个 APEX 的版本与其文件级签名归属 |
| Mainline 模块（商店通道下发） | 由平台签名体系决定 | 独立于整机发布周期 | 整机固件版本号**不能**说明模块版本；两者必须分开记录 |

第三条就是很多治理动作失效的原因：一个把"固件版本"当作唯一资产字段的企业资产库，无法表达"同一固件上模块版本不同"这件事，因此也无法回答"某个模块级修复是否到达"。这与[远程代码执行报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)中把模块化通道作为独立投递/修复腿的处理方式对应。

设备侧的可观测边界（本文再次强调）：无特权采集能看到的是 `/apex` 下挂载项的**存在性**，不是模块版本。因此脚本 1 对模块只统计条目数并标注为"记录，不判定"；任何把"没看到某个挂载项"读成"该模块未更新"的做法都不成立。

---

## 十二、内核侧构建口径：ACK 分支、上游优先、KMI 与二进制交付

内核是这条链上唯一一个"公开口径为二进制交付"的部分，所以单独成节。

| 口径 | 公开表述 | 安全侧读法 |
| --- | --- | --- |
| 分支即版本 | "Branch names correspond to kernel versions, such as android15-6.6 ." | 内核归因需要**发行版 + 分支 + 子级别修订**三层，只记 Android 版本无法定位缺陷影响范围 |
| 上游优先 | "Upstream development is strongly encouraged, and after development is accepted there, it can be backported to the specific ACK branch as needed." | 一段修复的"上游已合入"与"已进入某条 ACK 分支"是两个时间点；用前者声称后者是常见话术 |
| KMI 稳定性 | "The GKI kernel is built and shipped in binary form and vendor-loadable"（原文截断） | 内核主体由平台侧构建并交付二进制；厂商侧主要是模块。审计焦点随之从"内核源码"转向"模块与配置" |
| hermetic 构建 | "The hermetic build process ensures a stable KMI by having repo manifests in kernel/manifest completely describe the build environment." | 内核这条子链的**可归因性靠 manifest 完整描述构建环境**——与第三节同一机制，作用在内核上 |
| 构建工具演进 | "Android 13 introduced building kernels with Bazel." | 内核构建入口与平台构建工具链并不总是同一套，脚本化的构建审计要分开处理 |
| 发布清单 | "This document provides a monthly listing of GKI release builds for android17-6.18." | GKI 构建号可回指公开清单，是内核侧最省力的外部核验点 |

三点判断：

1. **"内核以二进制交付"既是审计成本也是审计工具。** 它意味着第三方无法用重编方式独立验证，但反过来给了一个清晰的核对点：某个 GKI 构建号应当对应公开清单里的一条记录。
2. **`kernel/manifest` 的存在说明"构建环境被清单描述"是可实现的。** 把这个要求外推到整机（要求厂商可提供完整 manifest 与构建环境描述）是采购条款里最具体的一条，而不是空洞的"要求开源合规"。
3. **模块侧是厂商责任，内核主体侧是平台责任。** 这条分工决定了事故时该找谁：涉及内核主体的缺陷归因走平台清单，涉及厂商模块与设备节点的归因走厂商构建（后者与[内核结构与子系统报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)的设备节点供给讨论相接）。

---

## 十三、VINTF/FCM 与 VNDK 退役：构建期的兼容性约束

兼容性约束中有一部分是在**构建期**生效的，这也是本文与架构篇分工的地方。

- 方向性定义："The framework compatibility matrix describes what the Android framework expects from a given device."——框架兼容性矩阵描述框架对设备的要求。
- 多目标版本："An Android framework release has multiple Framework Compatibility Matrixes (FCMs), one for each upgradable Target FCM Version"（原文截断）——一次框架发布对应多个 FCM，逐个对应可升级的目标版本。

这两句给出的构建期事实是：设备的 vendor 侧产物必须在一组**版本化的接口承诺**下装配，而不是"能跑就行"。对安全侧的推论是"接口承诺的版本号"本身是一个可用于评估维护状态的信号：一个长期停留在低 Target FCM Version 的 vendor 产物，通常也意味着其上游供给与测试覆盖处于历史状态。

**VNDK 的状态必须单独说明**，因为大量历史资料仍在描述它：官方表述为"Based on these issues, we decided to deprecate VNDK starting with Android 15."——自 Android 15 起 VNDK 被弃用。

| 观察 | 正确读法 | 错误读法 |
| --- | --- | --- |
| 设备上仍能读到 VNDK 相关属性 | 存量设备/存量 vendor 产物的正常残留 | "该平台仍要求 VNDK 合规" |
| 新平台构建不再有 VNDK 快照概念 | 依赖形态变了，共享库的归属跟着变 | "所以库审计不需要做了" |
| 历史文档大量描述 VNDK 规则 | 需要按平台版本分档阅读 | 直接当作现行要求执行 |

对审计的实际影响：把"VNDK 列表"作为跨版本统一基线的检查表会同时产生误报与漏报——在 Android 15 及以后它不再适用，在存量设备上它仍是相关约束。基线必须按平台版本切分（脚本 1 对 VNDK 属性一律"记录，不判定"）。

---

## 十四、兼容性义务：CDD / CTS / VTS / STS 各自证明什么

四套东西经常被合称"合规要求"，但它们在构建链上证明的对象完全不同。混淆它们会让采购条款写成一句无法验证的话。

| 名称 | 它证明什么 | 公开口径 |
| --- | --- | --- |
| **CDD**（兼容性定义文档） | 需求本身：软件与硬件要求的集合 | "The CDD enumerates the software and hardware requirements for an Android-compatible device." |
| **CTS**（兼容性测试套件） | 上述要求在当前产物上的通过证据 | 由兼容性体系发布的测试套件（本文未取得可直接引用的逐字定义句，见第二十二节边界） |
| **VTS**（厂商测试套件） | 内核与 HAL 层的接口行为 | "The Android Vendor Test Suite (VTS) provides extensive testing on the following: Kernel; Hardware abstraction layer (HAL)." |
| **STS**（安全测试套件） | 安全级别修复相关的回归验证 | 官方以 Security Test Suite 名称发布，并给出 STS 参考文档入口 |

四者与构建体系的关系：

1. **CTS 是产物级的**，因此一次构建失败对应一组可回指的模块变更——这让 CTS 结果成为构建门禁的天然候选。
2. **VTS 是接口级的**，跨越框架与 vendor 的边界，正好覆盖第八节"两侧来自不同批次"这种非标准装配最容易引入问题的位置。
3. **STS 是补丁级的**，它与[补丁传导链报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)同侧：SPL 声称修复，STS 提供回归证据。
4. **CDD 是文本义务**，它不产生自动化证据；任何声称"满足 CDD"的表述都必须附带测试执行记录。

---

## 十五、构建身份与供应链风险清点（十项）

把前面各节收敛成可运营的清点单元。每项给出"证据在哪里 / 缺失时表现 / 责任位置"。**这张表是本文对外的主要产出**：它可以直接被转成发布门禁清单或供应商评估问卷。

| 编号 | 风险单元 | 需要的证据 | 缺失时的表现 | 责任位置 |
| --- | --- | --- | --- | --- |
| R1 | **test-keys 进入发布构建** | `ro.build.tags`、签名密钥归属记录 | 产物无归属、任何人都能产出同身份下一版 | 发布流水线（可 100% 阻断） |
| R2 | **非发布变体出厂** | 变体为 `user` 的构建记录与设备侧读数一致 | 设备侧读到 `userdebug`/`eng` | 发布门禁 |
| R3 | **构建产物未归档** | manifest 快照 + 构建描述 + out/target files 三者留存 | 事后无法把镜像回指到提交集合 | 构建平台 |
| R4 | **手工替换产物** | 源码只读原则未被绕过（第七节） | 产物与构建描述不一致，差异无法解释 | 系统集成/OTA 打包 |
| R5 | **manifest 项目漂移未审查** | 项目集合与 revision 集合的版本对照（脚本 2） | 新增上游项目/分支切换无人知晓 | 供给治理 |
| R6 | **两侧构建批次混用未记录** | 框架侧与厂商侧指纹/增量号对照表 | `super` 内组合从未被流水线正式产出 | 发布治理 |
| R7 | **模块版本与整机版本未分别记录** | APEX/模块级版本台账 | 无法回答"该模块修复是否到达" | 资产与补丁治理 |
| R8 | **内核交付形态的审计缺口** | GKI 构建号可回指公开清单；厂商模块清单留存 | 只能对二进制做比对，无法源码归因 | 平台与厂商分工 |
| R9 | **签名/发布密钥治理不明** | 密钥归属、轮换、使用范围记录 | 无法界定发布权归属（第九节） | 安全与运维共同 |
| R10 | **把"开源"当"可复现"** | 明确区分第二节的三档性质 | 采购条款失效、事后验证路径选错 | 采购与法务 |

**七项可运营指标**（每项都能从设备侧读数或流水线留存里算出来，不依赖厂商口头承诺）：

1. `test-keys` 出现率（发布批次中 `ro.build.tags` 非 release-keys 的占比）——目标恒为 0；
2. 非 `user` 变体出厂数量——目标恒为 0；
3. 构建归档完整率：同时具备 manifest 快照 + 构建描述 + target files 的发布批次占比；
4. 两侧批次一致率：框架侧与厂商侧构建标识能被同一发布记录解释的比例；
5. manifest 漂移审查覆盖率：项目集合发生变化的构建中，做过对照并留档的比例；
6. 模块版本台账覆盖率：资产库中记录到 APEX/模块级版本而不是只记整机版本的比例；
7. 供给记录新鲜度：`android-latest-release` 解析记录与内部基线之间的时间差（第四节）。

---

## 十六、脚本 1：设备侧构建身份快照（只读）

**目的**：在一台自有或已授权的设备上，只读采集与构建身份相关的属性，输出可回指采集值的判定。**不做**任何写入、安装、触发式探测。

**采集面**：`getprop`（构建类型、密钥标签、可调试标志、框架侧与厂商侧 SPL/指纹/增量号、VNDK 版本、平台、启动验证状态与锁状态）、`getenforce`、`toybox ls /apex`。

**判定分层**：`L3-HIGH` 构建身份不成立（不可归因）；`L2-WARN` 需要人工核对；`L1-INFO` 正常记录；**`L0-LOW` 数据缺口——采不到不等于没问题**。

```bash
#!/usr/bin/env bash
# 脚本 1: build-identity-snapshot.sh — 设备侧构建身份快照（只读）
# 用法: build-identity-snapshot.sh <设备序列号> <基准日期 YYYY-MM-DD>
# 依赖: adb；全部为 getprop / getenforce / toybox ls 只读调用
set -u
dev="${1:?用法: build-identity-snapshot.sh <serial> <YYYY-MM-DD>}"
ref="${2:?缺少基准日期，格式 YYYY-MM-DD}"
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable" >&2; exit 1; }
case "$ref" in
  [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]) ref_m=$(( 10#${ref:0:4} * 12 + 10#${ref:5:2} )) ;;
  *) echo "ERR,bad_reference_date" >&2; exit 2 ;;
esac

hi=0; md=0; gap=0; info=0
p() { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }
row() {
  printf '%s,%s\n' "$1" "$2"
  case "$2" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) md=$(( md + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
month_of() { case "$1" in
  [0-9][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]) printf '%s' $(( 10#${1:0:4} * 12 + 10#${1:5:2} )) ;;
  *) printf '' ;; esac; }
fp_triple() { printf '%s' "$1" | cut -d: -f1; }
fp_build()  { printf '%s' "$1" | cut -d: -f2 | cut -d/ -f1-2; }

btype="$(p ro.build.type)";   tags="$(p ro.build.tags)"
dbg="$(p ro.debuggable)";     spl="$(p ro.build.version.security_patch)"
vspl="$(p ro.vendor.build.version.security_patch)"
fp="$(p ro.build.fingerprint)";  vfp="$(p ro.vendor.build.fingerprint)"
inc="$(p ro.build.version.incremental)"; vinc="$(p ro.vendor.build.version.incremental)"
vndk="$(p ro.vndk.version)";  board="$(p ro.board.platform)"
vb="$(p ro.boot.verifiedbootstate)"; lock="$(p ro.boot.flash.locked)"
enforce="$(adb -s "$dev" shell getenforce 2>/dev/null | tr -d '\r')"
apex_list="$(adb -s "$dev" shell "toybox ls /apex" 2>/dev/null | tr -d '\r')"

row build_type "$btype";      row build_tags "$tags"
row debuggable "$dbg";        row framework_spl "$spl"
row vendor_spl "$vspl";       row fingerprint "$fp"
row vendor_fingerprint "$vfp"; row framework_incremental "$inc"
row vendor_incremental "$vinc"; row vndk_version "$vndk"
row board_platform "$board";  row verifiedbootstate "$vb"
row flash_locked "$lock";     row selinux "$enforce"
apex_n="$(printf '%s\n' "$apex_list" | grep -c '[^[:space:]]')"
row apex_entries "$apex_n"

case "$tags" in
  test-keys)     row FIND_tags "L3-HIGH 发布构建使用 AOSP test-keys：产物无法归因，同身份的下一版可被任何人产出" ;;
  release-keys)  row FIND_tags "L1-INFO release-keys（密钥归属仍需发布流水线留存佐证，本项不能单独证明发布权）" ;;
  "")            row FIND_tags "L0-LOW 未采到 ro.build.tags：不得据此判定密钥类型" ;;
  *)             row FIND_tags "L2-WARN ro.build.tags 为未知形态（$tags），需与构建配置核对" ;;
esac

case "$btype" in
  user)          row FIND_build_type "L1-INFO 发布变体" ;;
  userdebug|eng) row FIND_build_type "L2-WARN 设备以非发布变体运行（$btype），与构建归档记录核对" ;;
  "")            row FIND_build_type "L0-LOW 未采到 ro.build.type" ;;
  *)             row FIND_build_type "L0-LOW 未预期的变体值（$btype）" ;;
esac

case "$dbg" in
  1) row FIND_debuggable "L2-WARN ro.debuggable=1：调试面放宽，与变体判定互相印证" ;;
  0) row FIND_debuggable "L1-INFO ro.debuggable=0" ;;
  *) row FIND_debuggable "L0-LOW 未采到 ro.debuggable" ;;
esac

f_m="$(month_of "$spl")"; v_m="$(month_of "$vspl")"
if [ -z "$f_m" ]; then
  row FIND_spl "L0-LOW 未采到框架侧 SPL：不判定"
else
  lag=$(( ref_m - f_m ))
  if [ "$lag" -gt 6 ]; then
    row FIND_spl "L3-HIGH 框架侧 SPL 落后 ${lag} 个月（基准 ${ref}）"
  elif [ "$lag" -lt 0 ]; then
    row FIND_spl "L0-LOW 框架侧 SPL 晚于基准日期（${spl} > ${ref}），核对基准输入"
  else
    row FIND_spl "L1-INFO 框架侧 SPL 落后 ${lag} 个月"
  fi
fi
if [ -n "$f_m" ] && [ -n "$v_m" ]; then
  d=$(( f_m - v_m )); [ "$d" -lt 0 ] && d=$(( 0 - d ))
  if [ "$d" -gt 1 ]; then
    row FIND_spl_split "L2-WARN 框架侧与厂商侧 SPL 相差 ${d} 个月：两侧来自不同批次，需分别归因"
  else
    row FIND_spl_split "L1-INFO 两侧 SPL 相差 ${d} 个月"
  fi
else
  row FIND_spl_split "L0-LOW 两侧 SPL 不完整：不判定差值"
fi

t1="$(fp_triple "$fp")"; t2="$(fp_triple "$vfp")"
b1="$(fp_build "$fp")";  b2="$(fp_build "$vfp")"
if [ -z "$t1" ] || [ -z "$t2" ]; then
  row FIND_fp_split "L0-LOW 指纹不完整（框架/厂商至少一项未采到）：不判定装配关系"
elif [ "$t1" != "$t2" ]; then
  row FIND_fp_split "L2-WARN 框架侧与厂商侧设备三元组不一致（$t1 | $t2）：非标准装配，核对发布记录"
else
  row FIND_fp_split "L1-INFO 设备三元组一致（$t1）"
fi
if [ -n "$b1" ] && [ -n "$b2" ]; then
  if [ "$b1" != "$b2" ]; then
    row FIND_fp_build "L1-INFO 两侧构建标识不同（$b1 | $b2）：分阶段构建下属正常，仅作记录"
  else
    row FIND_fp_build "L1-INFO 两侧构建标识相同（$b1）"
  fi
else
  row FIND_fp_build "L0-LOW 构建标识可比对信息不足"
fi

if [ -n "$inc" ] && [ -n "$vinc" ]; then
  row FIND_incremental "L1-INFO 增量号 框架=$inc 厂商=$vinc（记录，不判定）"
else
  row FIND_incremental "L0-LOW 增量号缺失：不得据此判定归档完整性"
fi

if [ -z "$vndk" ]; then
  row FIND_vndk "L0-LOW 未读到 VNDK 版本；Android 15 起 VNDK 已弃用，缺失不构成异常"
else
  row FIND_vndk "L1-INFO VNDK 版本 $vndk（存量平台属性，仅记录不判定）"
fi
if [ -z "$board" ]; then row FIND_board "L0-LOW 未采到 ro.board.platform"
else row FIND_board "L1-INFO $board"; fi

case "$vb" in
  green)             row FIND_avb "L1-INFO verifiedbootstate=green" ;;
  yellow|orange|red) row FIND_avb "L2-WARN verifiedbootstate=$vb：构建身份无法由启动链完整背书，结合锁状态与厂商实现判断" ;;
  "")                row FIND_avb "L0-LOW 未采到 verifiedbootstate" ;;
  *)                 row FIND_avb "L0-LOW 未预期的 verifiedbootstate 值（$vb）" ;;
esac
case "$lock" in
  1) row FIND_lock "L1-INFO bootloader 锁定" ;;
  0) row FIND_lock "L2-WARN bootloader 未锁定" ;;
  *) row FIND_lock "L0-LOW 未采到 ro.boot.flash.locked" ;;
esac
case "$enforce" in
  Enforcing)  row FIND_selinux "L1-INFO SELinux Enforcing（运行时策略，此处作为构建侧装配结果记录）" ;;
  Permissive) row FIND_selinux "L2-WARN SELinux Permissive" ;;
  *)          row FIND_selinux "L0-LOW 未采到 getenforce" ;;
esac
if [ "$apex_n" -gt 0 ]; then
  row FIND_apex "L1-INFO /apex 条目数 $apex_n（存在性观察，不含模块版本）"
else
  row FIND_apex "L0-LOW 未采到 /apex 清单；不得据此判定任何模块的更新状态"
fi
printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,dev=%s\n' "$hi" "$md" "$gap" "$info" "$dev"
```

**设计说明**

- **只用一个采集点做两类判定**：`ro.build.tags` 与 `ro.build.type` 是构建期写入的常量，因此它们的异常值不依赖运行时状态，可以直接做发布门禁；其余字段（SPL、锁状态、SELinux）作为交叉印证而不是单独定性。
- **指纹比对只比"结构"不比"版本"**：`fp_triple` 取冒号前的品牌/产品/设备三段，`fp_build` 取冒号后的构建标识两段。比较这两个结构性片段可以识别"两个不同产品的产物被装进同一台设备"，而不需要假设指纹字符串的完整格式在所有厂商上相同。
- **VNDK 缺失被显式声明为正常**：这是第十三节结论的落地——不把已经退役的属性当成异常。
- **不可归因不等于被攻击**：`L3-HIGH` 在这里的含义是"这份产物无法追溯到发布流水线"，它是一条治理结论，不是恶意证据。

**样例输出（构造的三份只读采集画像，与脚本逐字对应）**

一台发布形态正常的设备：

```text
FIND_tags,L1-INFO release-keys（密钥归属仍需发布流水线留存佐证，本项不能单独证明发布权）
FIND_build_type,L1-INFO 发布变体
FIND_spl_split,L1-INFO 两侧 SPL 相差 1 个月
FIND_fp_split,L1-INFO 设备三元组一致（google/oriole/oriole）
SUMMARY,high=0,warn=0,gap=0,info=14,dev=BGOOD001
```

一台由工程迭代产物直接出厂的设备：

```text
FIND_tags,L3-HIGH 发布构建使用 AOSP test-keys：产物无法归因，同身份的下一版可被任何人产出
FIND_build_type,L2-WARN 设备以非发布变体运行（eng），与构建归档记录核对
FIND_spl,L3-HIGH 框架侧 SPL 落后 18 个月（基准 2026-10-26）
FIND_spl_split,L2-WARN 框架侧与厂商侧 SPL 相差 5 个月：两侧来自不同批次，需分别归因
FIND_fp_split,L2-WARN 框架侧与厂商侧设备三元组不一致（vendor/aim09/aim09 | vendor/kx500/kx500）：非标准装配，核对发布记录
FIND_incremental,L0-LOW 增量号缺失：不得据此判定归档完整性
FIND_avb,L2-WARN verifiedbootstate=orange：构建身份无法由启动链完整背书，结合锁状态与厂商实现判断
SUMMARY,high=2,warn=7,gap=2,info=3,dev=BBAD001
```

一台采集严重受限的设备（只读到两个属性）：

```text
FIND_tags,L0-LOW 未采到 ro.build.tags：不得据此判定密钥类型
FIND_fp_split,L0-LOW 指纹不完整（框架/厂商至少一项未采到）：不判定装配关系
FIND_apex,L0-LOW 未采到 /apex 清单；不得据此判定任何模块的更新状态
SUMMARY,high=0,warn=0,gap=11,info=3,dev=BGAP001
```

第三种画像在真实环境里最常见（无调试权限、部分属性被限制读取）。它的正确读法是**"这台设备的构建身份无法评估"**，而不是"这台设备没有构建身份问题"。

---

## 十七、脚本 2：离线 manifest 漂移对照

**目的**：把第三节与第十五节 R5 变成一条可执行动作——对两份 repo manifest 做纯文本对照，回答"上游供给面变了没有"。**完全离线**：不联网、不写入、不调用 repo 或 git，只读两个文件。

```bash
#!/usr/bin/env bash
# 脚本 2: manifest-drift-audit.sh — AOSP repo manifest 漂移离线对照（纯文本，无网络无写入）
# 用法: manifest-drift-audit.sh <基线 manifest.xml> <待审 manifest.xml>
# 解析为文本级：按 <project / <remote / <default / <include 起始标签抽取属性；解析失败记为数据缺口而不是结论
set -u
base="${1:?用法: manifest-drift-audit.sh <baseline.xml> <candidate.xml>}"
cand="${2:?缺少待审 manifest 路径}"
for f in "$base" "$cand"; do
  [ -r "$f" ] || { echo "ERR,input_unreadable,$f" >&2; exit 1; }
  [ -s "$f" ] || { echo "ERR,input_empty,$f" >&2; exit 1; }
done

hi=0; md=0; gap=0; info=0
row() {
  printf '%s,%s\n' "$1" "$2"
  case "$2" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) md=$(( md + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
attr() { # attr <文件> <标签> <属性名> → 逐行取值（同一行内匹配，跨行书写的属性可能漏取）
  grep -o "<$2[^>]*$3=\"[^\"]*\"" "$1" 2>/dev/null | sed -n "s/.*$3=\"\([^\"]*\)\".*/\1/p" | sort -u
}

tb="$(mktemp)"; tc="$(mktemp)"; rb="$(mktemp)"; rc2="$(mktemp)"; rmb="$(mktemp)"; rmc="$(mktemp)"
trap 'rm -f "$tb" "$tc" "$rb" "$rc2" "$rmb" "$rmc"' EXIT

attr "$base" project name > "$tb"
attr "$cand" project name > "$tc"
attr "$base" remote name   > "$rb"
attr "$cand" remote name   > "$rc2"
attr "$base" project revision > "$rmb"
attr "$cand" project revision > "$rmc"

nb="$(grep -c '[^[:space:]]' "$tb")"; nc="$(grep -c '[^[:space:]]' "$tc")"
row baseline_path "$base"; row candidate_path "$cand"
row baseline_projects "$nb"; row candidate_projects "$nc"

if [ "$nb" -eq 0 ] || [ "$nc" -eq 0 ]; then
  row FIND_parse "L0-LOW 项目行解析结果为空：本次对照不成立（可能是跨行书写的 XML），不得读成无差异"
elif [ "$nb" -lt 50 ] || [ "$nc" -lt 50 ]; then
  row FIND_parse "L2-WARN 项目数远低于 AOSP 量级（$nb | $nc）：确认输入是否为完整 manifest 或 partial clone 子集"
else
  row FIND_parse "L1-INFO 解析成功（$nb | $nc）"
fi

added="$(comm -13 "$tb" "$tc" | grep '[^[:space:]]')"
removed="$(comm -23 "$tb" "$tc" | grep '[^[:space:]]')"
na="$(printf '%s\n' "$added" | grep -c '[^[:space:]]')"
nr="$(printf '%s\n' "$removed" | grep -c '[^[:space:]]')"
row projects_added "$na"; row projects_removed "$nr"
for n in $added; do printf 'ADD,%s\n' "$n"; done
for n in $removed; do printf 'DEL,%s\n' "$n"; done
if [ "$na" -gt 0 ]; then
  row FIND_added "L2-WARN 新增 ${na} 个项目：上游供给面扩大，逐项确认来源、分支与必要性"
else
  row FIND_added "L1-INFO 无新增项目"
fi
if [ "$nr" -gt 0 ]; then
  row FIND_removed "L2-WARN 移除 ${nr} 个项目：确认是有意裁剪还是清单缺失（裁剪会改变可归因的源码全集）"
else
  row FIND_removed "L1-INFO 无移除项目"
fi

vb="$(mktemp)"; vc="$(mktemp)"; rvb="$(mktemp)"; rvc="$(mktemp)"
trap 'rm -f "$tb" "$tc" "$rb" "$rc2" "$rmb" "$rmc" "$vb" "$vc" "$rvb" "$rvc"' EXIT
comm -13 "$rmb" "$rmc" > "$vb"   # 仅在待审清单中出现的 revision 值
comm -23 "$rmb" "$rmc" > "$vc"   # 仅在基线中出现的 revision 值
nrv="$(grep -c '[^[:space:]]' "$vb")"; nrv_old="$(grep -c '[^[:space:]]' "$vc")"
row revisions_added "$nrv"; row revisions_dropped "$nrv_old"
for n in $(cat "$vb"); do printf 'REV_NEW,%s\n' "$n"; done
for n in $(cat "$vc"); do printf 'REV_GONE,%s\n' "$n"; done
if [ "$nrv" -gt 0 ] || [ "$nrv_old" -gt 0 ]; then
  row FIND_revision "L2-WARN revision 集合发生变化（新引入 $nrv 个 / 消失 $nrv_old 个）：分支切换比代码差异更早解释产物不同"
else
  row FIND_revision "L1-INFO revision 集合未变化"
fi

new_remote="$(comm -13 "$rb" "$rc2" | grep '[^[:space:]]')"
gone_remote="$(comm -23 "$rb" "$rc2" | grep '[^[:space:]]')"
if [ -n "$new_remote" ] || [ -n "$gone_remote" ]; then
  for n in $new_remote;  do printf 'REMOTE_NEW,%s\n' "$n"; done
  for n in $gone_remote; do printf 'REMOTE_GONE,%s\n' "$n"; done
  row FIND_remote "L3-HIGH remote 名称集合变化：抓取来源被改写，属于供给链变更，需授权与审计记录"
else
  row FIND_remote "L1-INFO remote 集合未变化"
fi

db_rev="$(grep -o '<default[^>]*revision="[^"]*"' "$base" | sed 's/.*revision="\([^"]*\)".*/\1/' | head -1)"
dc_rev="$(grep -o '<default[^>]*revision="[^"]*"' "$cand" | sed 's/.*revision="\([^"]*\)".*/\1/' | head -1)"
if [ -z "$db_rev" ] && [ -z "$dc_rev" ]; then
  row FIND_default "L0-LOW 两份清单均未读到 <default revision>：不判定"
elif [ "$db_rev" != "$dc_rev" ]; then
  row FIND_default "L2-WARN 默认分支由 ${db_rev:-空} 变为 ${dc_rev:-空}：记录解析时间与分支名（AOSP 分支口径变更见正文第四节）"
else
  row FIND_default "L1-INFO 默认分支一致（$dc_rev）"
fi
ib="$(grep -c '<include' "$base")"; ic="$(grep -c '<include' "$cand")"
if [ "$ib" != "$ic" ]; then
  row FIND_include "L2-WARN manifest include 数量变化（$ib → $ic）：项目全集可能由多文件拼接，对照需覆盖被包含文件"
else
  row FIND_include "L1-INFO include 数量一致（$ic）"
fi

hb="$(grep -c '<hook' "$base")"; hc="$(grep -c '<hook' "$cand")"
if [ "$hc" -gt 0 ]; then
  if [ "$hc" -gt "$hb" ]; then
    row FIND_hook "L2-WARN 待审清单出现 ${hc} 处 hook 类条目（基线 ${hb}）：manifest 携带构建期执行内容，须逐条人工核对与审批"
  else
    row FIND_hook "L1-INFO hook 类条目数量未增加（$hb → $hc），仍建议纳入配置审查"
  fi
else
  row FIND_hook "L1-INFO 未出现 hook 类条目"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,base=%s,cand=%s\n' "$hi" "$md" "$gap" "$info" "$nb" "$nc"
```

**设计说明**

- **文本级解析，并承认解析的失败模式。** manifest 是 XML，属性可以跨行书写；`attr()` 只在同一行内匹配标签与属性。脚本因此把"解析结果为空"处理成 `L0-LOW 对照不成立`，绝不把解析失败读成"没有差异"——这是这类对照工具最容易造成的假安心。
- **项目数量级检查**：低于 50 个项目时给 WARN 并提示可能是局部清单。AOSP 的项目规模远大于此，数量级本身就是"输入是否完整"的第一道校验。
- **remote 集合变化定为 L3-HIGH**：项目增减是"供给变多"，而抓取来源（remote）的变化是"供给换了地方"，后者的供应链含义更直接。
- **hook 类条目**：本文未取得该元素官方定义的逐字描述（见第二十二节边界），因此脚本只报"出现数量增加即人工核对"，不作定性。它的价值在于把注意力引向"清单里是否携带构建期执行内容"这一类问题。

**样例输入（构造的最小清单片段，真实清单有数百个项目行；完整 fixture 未随本文发布）**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="aosp" fetch="https://android.googlesource.com/" />
  <default revision="android16-qpr2-release" remote="aosp" sync-j="4" />
  <project name="platform/manifest" path="manifest" revision="android16-qpr2-release" />
  <project name="platform/build/soong" path="build/soong" revision="android16-qpr2-release" />
  <!-- … 其余项目行 … -->
</manifest>
```

**样例输出（对上述 fixture 的两次真实运行）**

无漂移对照（同一份清单）：

```text
FIND_parse,L1-INFO 解析成功（52 | 52）
FIND_added,L1-INFO 无新增项目
FIND_revision,L1-INFO revision 集合未变化
FIND_remote,L1-INFO remote 集合未变化
FIND_default,L1-INFO 默认分支一致（android16-qpr2-release）
FIND_hook,L1-INFO 未出现 hook 类条目
SUMMARY,high=0,warn=0,gap=0,info=8,base=52,cand=52
```

构造一次真实世界中最常见的三种漂移（新增两个项目、默认分支切换、多一处 hook 条目）：

```text
FIND_parse,L1-INFO 解析成功（52 | 54）
projects_added,2
projects_removed,0
ADD,platform/external/newlib-fork
ADD,platform/vendor/acme/powerhal
FIND_added,L2-WARN 新增 2 个项目：上游供给面扩大，逐项确认来源、分支与必要性
revisions_added,1
revisions_dropped,1
REV_NEW,android17-main
REV_GONE,android16-qpr2-release
FIND_revision,L2-WARN revision 集合发生变化（新引入 1 个 / 消失 1 个）：分支切换比代码差异更早解释产物不同
FIND_default,L2-WARN 默认分支由 android16-qpr2-release 变为 android17-main：记录解析时间与分支名（AOSP 分支口径变更见正文第四节）
FIND_hook,L2-WARN 待审清单出现 1 处 hook 类条目（基线 0）：manifest 携带构建期执行内容，须逐条人工核对与审批
SUMMARY,high=0,warn=4,gap=0,info=4,base=52,cand=54
```

再构造一次只改 remote 的漂移（内部镜像替换官方来源）：

```text
REMOTE_NEW,mirror-eng
FIND_remote,L3-HIGH remote 名称集合变化：抓取来源被改写，属于供给链变更，需授权与审计记录
SUMMARY,high=1,warn=0,gap=0,info=7,base=52,cand=52
```

这份输出对应的是**项目集合完全没变、只有抓取来源变了**的情况——它也是"只看项目清单"的对照会漏掉的唯一一类高危变化。

---

## 十八、检测与取证：构建侧异常在设备与其他地方留下什么痕迹

构建体系本身不产生运行时告警，它的证据散在三处不同的地方，需要分别建立采集。

| 证据位置 | 信号 | 强度与限制 |
| --- | --- | --- |
| **设备侧属性** | `ro.build.tags` 非 release-keys；`ro.build.type` 非 user；`ro.debuggable`=1 | 强（构建期写入、批量可比对）；但只说明"意图"，不证明产物未被事后替换 |
| **设备侧指纹族** | 框架/厂商/引导三类指纹的设备三元组或构建标识互不一致 | 中等：分阶段构建下属正常，必须与厂商发布记录联合判断 |
| **启动链读数** | `verifiedbootstate`、`flash.locked` | 粗粒度：给"能不能信任这份产物"的方向，不给来源 |
| **模块清单** | `/apex` 条目集合与上一版本快照的差异 | 只到存在性；差异本身值得核对，不能直接读成"更新缺失" |
| **发布侧留存** | manifest 快照、target files、构建描述、签名记录归档 | 最强，也是唯一能支撑归因的一层；设备侧看不到它 |
| **外部公开记录** | GKI 发布清单条目、安全公告、厂商更新日志 | 用于把内部构建号回指到公开可核验的记录（第十二节） |

三条**可被安全团队直接采用**的取证动作：

1. **建立"构建身份基线"**：对每一个在用机型记录一次脚本 1 的完整输出，作为后续比对的锚。之后任何一台同型号设备读出的 `tags/type/三元组` 与锚不同，即为高价值异常——这是不需要 root、不需要日志转发的批量检测。
2. **事故时先问留存再问设备**：如果发布侧留存（manifest + 构建描述 + target files）不存在，那么对设备做再多分析也无法回答"这段代码是什么时候进入产品的"。R3 的缺失会把事故调查从"定位一次提交"降级为"比对一份二进制"。
3. **把两侧批次关系写进调查模板**：厂商侧修复与框架侧修复落在不同批次上时，"这台设备是否有该修复"是两个问题。补丁链报告的取证口径与此处一致，本文只补一条：**批次关系属于构建侧事实，不取决于设备当前状态。**

---

## 十九、常见误区（12 条）

1. **"源码是公开的，所以固件是可复现的。"** 公开口径支撑的是可归因与可核验；AOSP 与 Pixel 的公开页面没有作出可复现构建承诺，Pixel 提供的是透明度日志（参考资料 32、33）。
2. **把 `aosp-main` 当作"最新可信主干"。** 自 2025-03-27 起它是不再被合入的只读分支（参考资料 5），而最新分支由 `android-latest-release` 清单指代（参考资料 4）。
3. **只看一个 SPL 字段就判定合规。** 框架侧与厂商侧本就是从不同装配与不同维护主体来的两个值；第八、九节给出结构性原因，字段级别的处理在补丁链报告。
4. **"设备上读到 VNDK 版本，说明这套机制仍然有效。"** VNDK 自 Android 15 起弃用（参考资料 17），存量属性值只是残留，不是现行约束。
5. **"test-keys 只影响启动警告，不影响安全。"** 它同时消掉归因、追责与核验三件事（第九节），是治理级别的高危项而不是显示级别的小问题。
6. **"把 Android.mk 全部转成 Android.bp 是一项安全改进。"** 转换的收益是可分析性与速度（参考资料 7、9）；缺陷数量与它没有直接关系。
7. **"`/apex` 里没有某个模块 = 该模块没更新。"** 无特权采集只能看到存在性，版本需要另外的证据；脚本 1 把它记为"存在性观察"。
8. **"内核有公开源码，所以厂商内核可第三方独立重编验证。"** GKI 的公开口径是二进制形态构建与交付（参考资料 20）；可行的核验点是构建号能否回指公开发布清单。
9. **"满足 CDD 就等于通过测试。"** CDD 是要求清单（参考资料 29），不产生自动化证据；证据在 CTS/VTS/STS 的执行记录里。
10. **"设备三元组一致就说明两侧来自同一次构建。"** 一致只是必要条件之一；构建标识与增量号也可能因分阶段构建而不同（脚本 1 对此只给 INFO）。
11. **"改一次 manifest 只是工程内部的事。"** manifest 是构建的第一输入，remote 与项目集合的变化属于供给链变更（第十七节脚本 2 的 L3/L2 判定）。
12. **"构建体系审计需要源码权限。"** 大部分结论可以从**文本级输入**（manifest、模块声明、发布留存）与**设备侧只读属性**得出，不需要访问厂商源码——这也是本文两个脚本都只做这两类动作的原因。

---

## 二十、趋势判断

1. **清单化描述会持续扩展。** GKI 已经在用 `kernel/manifest` 完整描述构建环境（参考资料 21）；这个模式向整机延伸（更完整的构建环境描述与留存）在技术上没有障碍，剩下的只是厂商是否愿意公开。对企业侧的实际影响是：**采购时要求"构建环境可描述"比要求"开源合规声明"更容易验收。**
2. **交付单元会继续变小，构建身份会继续变多。** APEX/Mainline（参考资料 24、25）已经把"一份固件"拆成"整机 + 若干独立签名、独立通道的模块"。构建身份因此不再是单值，而是每台设备上一组需要分别记录的字段——资产模型不跟着改，所有覆盖率指标都会失真。
3. **接口承诺的版本号会成为维护状态的代理指标。** VINTF/FCM 的多目标版本机制（参考资料 15、16）意味着"停留在哪一档"是可以直接读出来的历史信号；评估供应商时它比自述的更新承诺更早给出答案。
4. **构建期门禁的收益会高于运行期检测的追加。** 本文第十节列的 R1、R2、R5 三项，都可以用文本级或属性级检查在发布前 100% 阻断；相比之下，任何运行期检测都无法把已出厂的 test-keys 构建变成可归因产物。
5. **2025-03-27 的分支口径变更会在未来两年持续制造误判。** 长期存在的历史文档、内部脚本与培训材料里仍写着旧的分支用法；供给记录里"解析到的分支名 + 解析时间"两个字段缺失时，这类误判无法被发现（第四节）。

---

## 二十一、分角色结论与 FAQ

**给系统与构建工程师**：把三件事变成流水线里的硬门禁——`ro.build.tags` 与 `ro.build.type` 的期望值检查、构建归档三元组（manifest 快照 + 构建描述 + target files）、manifest 差异对照（脚本 2 可直接接入 CI，它只读文件）。这三项加起来的工作量远小于一次事故调查。

**给 ROM/OEM 安全负责人**：本文唯一需要你新建立的资产是**构建身份基线**（每个在用机型一份脚本 1 输出）。有了它，"某型号是否存在非发布变体出厂"这类问题可以从提问变成查询。

**给采购与供应商评估**：把"可复现构建"从条款里删掉，换成三条可验收的表述——能否提供本次发布的 manifest 与构建环境描述；能否提供框架侧与厂商侧构建标识的对应关系；能否提供 CTS/VTS/STS 的执行记录。这三条分别覆盖 R3、R6、第十四节。

**给检测与取证**：优先采集构建期写入的常量（tags/type/指纹族），它们批量可比对、不依赖运行时状态；`/apex` 与启动链读数只作为方向性证据（第十八节）。

**FAQ**

**Q1：脚本 1 报 `L3-HIGH` 是不是意味着设备被入侵了？**
不是。它意味着这份产物的构建身份无法归因，属治理结论。恶意替换与流水线偷懒在只读属性上看起来是一样的，需要发布侧留存才能区分。

**Q2：为什么不判定模块版本？**
无特权采集看不到模块版本（第十一节）。脚本 1 因此只统计 `/apex` 条目数并显式声明"存在性观察"。

**Q3：脚本 2 需要能访问厂商源码库吗？**
不需要。它只对照两份 manifest 文件，可以是内部两份历史快照，也可以是"公开基准清单 vs 厂商提供清单"。

**Q4：为什么 remote 变化定 L3-HIGH，项目增减只定 L2-WARN？**
项目增减改变供给面大小；remote 变化改变供给来源。后者意味着代码从别处来，属于需要在授权与审计下解释的变更。

**Q5：设备上没有 VNDK 属性，是不是构建有问题？**
不是。Android 15 起 VNDK 弃用（第十三节），脚本 1 对该项的缺失明确写为"不构成异常"。

**Q6：`source envsetup.sh` 与直接调用 `lunch` 的区别是什么？**
`lunch` 是由 `envsetup.sh` 定义的 shell 函数，因此必须 source 后才存在（参考资料 9）。把文档里的命令顺序照抄成脚本时漏掉 source，是构建审计最常见的执行错误。

**Q7：本文能给"如何验证某份固件来源"的答案吗？**
只能给方向。在可复现构建没有公开承诺的前提下，可行组合是透明度日志 + 签名链 + 构建号回指公开清单（第二、十、十二节）；本文不提供替换或伪造任何一项的内容（范围声明）。

---

## 二十二、参考资料

以下条目全部为本次核验实际取得逐字内容的公开来源，编号供正文与脚本引用；引文为原文（英文）摘录，破折号后为中文口径说明。截断处以"（原文截断）"标注。

1. Repo 与 manifest 的聚合机制 — Download the Android source / Source control tools（source.android.com）："Repo uses manifest files to aggregate Git projects into the Android superproject." — manifest 文件把 Git 项目聚合成 Android 超级项目。https://source.android.com/docs/setup/download/source-control-tools
2. 同一文档站的下载入口（含 "Initialize the Repo client" 小节）："Set up your client to access the Android source repository:" — 客户端初始化入口。https://source.android.com/docs/setup/download
3. `repo init` 的落地物："This command creates a .repo/ directory with Git repositories for the Repo source code and the standard Android manifest files." — `.repo/` 是构建可归因性的物理载体。https://source.android.com/docs/setup/reference/repo
4. 分支口径变更（2025-03-27）："Starting March 27, 2025, the latest release branch will always be referenced by the new android-latest-release manifest, which can be used directly with Repo." — 最新分支由 `android-latest-release` 清单指代。https://source.android.com/docs/whatsnew/site-updates
5. AOSP FAQ 关于 `aosp-main`："No, the code won't be merged to aosp-main, which is a read-only branch as of March 27, 2025." — 只读描述的是合并方向，不是审核强度。https://source.android.com/docs/setup/about/faqs
6. 同页关于稳定分支："The most recent branch is considered the current stable branch version, which the android-latest-release manifest points to." https://source.android.com/docs/setup/about/faqs
7. 构建系统三段结构："AOSP uses the Soong build system to build Android. Soong uses the kati GNU Make clone tool and Ninja build system component to speed up builds of Android." — https://source.android.com/docs/setup/build
8. 声明文件称谓："Soong build files are called blueprint files" — Soong 的构建文件即 blueprint 文件。https://source.android.com/docs/setup/build
9. `lunch` 的依赖关系："You must run source envsetup.sh once per shell and before the lunch command because lunch is defined by envsetup.sh" — https://source.android.com/docs/setup/build/building
10. 源码只读原则："During the build, the AOSP source tree is read-only." — https://source.android.com/docs/setup/build/building
11. 产物落点："The output of your build appears in $OUT_DIR ." — https://source.android.com/docs/setup/build/building
12. 顶层构建语义："The m command builds from the top of the tree, so you can run m from within subdirectories." — https://source.android.com/docs/setup/build/building
13. 构建变体 `user`："This build variant provides limited security access and is suited for production." — https://source.android.com/docs/setup/build/building
14. 迁移工具（platform/build/soong 项目说明，原文语法如此）："Soong includes a tool perform a first pass at converting Android.mk files to Android.bp files" — https://android.googlesource.com/platform/build/soong/
15. 框架兼容性矩阵方向性："The framework compatibility matrix describes what the Android framework expects from a given device." — https://source.android.com/docs/core/architecture/vintf
16. FCM 多目标版本："An Android framework release has multiple Framework Compatibility Matrixes (FCMs), one for each upgradable Target FCM Version"（原文截断） — https://source.android.com/docs/core/architecture/vintf/fcm
17. VNDK 退役："Based on these issues, we decided to deprecate VNDK starting with Android 15." — https://source.android.com/docs/core/architecture/vndk
18. 核心厂商分区："The vendor , vendor_boot , and odm partitions are core vendor partitions."（原文空格如此） — https://source.android.com/docs/core/architecture/partitions
19. super 的元数据组织："The super partition contains metadata listing the names and block ranges of each dynamic partition within super ." — https://source.android.com/docs/core/ota/dynamic_partitions/implement
20. GKI 交付形态："The GKI kernel is built and shipped in binary form and vendor-loadable"（原文截断） — https://source.android.com/docs/core/architecture/kernel/stable-kmi
21. hermetic 构建与 `kernel/manifest`："The hermetic build process ensures a stable KMI by having repo manifests in kernel/manifest completely describe the build environment." — https://source.android.com/docs/core/architecture/kernel/stable-kmi
22. ACK 分支命名："Branch names correspond to kernel versions, such as android15-6.6 ." — https://source.android.com/docs/core/architecture/kernel
23. 上游优先："Upstream development is strongly encouraged, and after development is accepted there, it can be backported to the specific ACK branch as needed." — https://source.android.com/docs/core/architecture/kernel/android-common
24. 内核构建工具："Android 13 introduced building kernels with Bazel." — https://source.android.com/docs/setup/build/building-kernels
25. GKI 发布清单（以 android17-6.18 为例）："This document provides a monthly listing of GKI release builds for android17-6.18." — https://source.android.com/docs/core/architecture/kernel/gki-android17-6_18-release-builds
26. APEX 文件级签名属性："To sign an APEX at the file-level, set the certificate property in one of these three ways:" — https://source.android.com/docs/core/ota/apex
27. Mainline 的更新意图："Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle." — https://source.android.com/docs/core/ota/modular-system
28. target files 签名入口："The signing script sign_target_files_apks works on the target files generated for a build." — https://source.android.com/docs/core/ota/sign_builds
29. 测试密钥位置："The Android tree includes test-keys under build/target/product/security ." — https://source.android.com/docs/core/ota/sign_builds
30. AVB 元数据结构："Following the hashtree is a VBMeta struct which contains the hashtree descriptor with hashtree metadata (root hash, salt, offset, etc.)" — https://android.googlesource.com/platform/external/avb/+/master/README.md
31. CDD 的义务性质："The CDD enumerates the software and hardware requirements for an Android-compatible device." — https://source.android.com/docs/compatibility/overview
32. VTS 覆盖对象："The Android Vendor Test Suite (VTS) provides extensive testing on the following: Kernel; Hardware abstraction layer (HAL)." — https://source.android.com/docs/core/tests/vts
33. STS 的官方名称与文档入口（Security Test Suite 参考文档由该页链接）："For details, see the Security Test Suite reference docs and the Tradefed reference docs." — https://source.android.com/docs/security/test/autorepro ；参考文档 https://source.android.com/reference/sts/packages
34. Pixel Binary Transparency（可核验，不是可复现）："We've enabled OEMs to create their own transparency log to help their users verify the OS or firmware running on their devices." — https://developers.google.com/android/binary_transparency/pixel_overview
35. 同一主题的官方安全博客表述："For example, Android Verified Boot strives to ensure all executed code comes from a trusted source, rather than from an attacker or corruption." — https://security.googleblog.com/2023/08/pixel-binary-transparency-verifiable.html
36. AOSP 主站 manifest 项目定位："Manifest that contains all the projects that are hosted on the AOSP server." — https://android.googlesource.com/
37. 同一措辞亦出现在被检出的清单文件中（default.xml）："Starting March 27, 2025, the latest release branch will always be referenced by the new android-latest-release manifest" — https://android.googlesource.com/platform/manifest/+/master/default.xml
38. CTS 文档入口（本页未取得可直接引用的定义句，仅作为检索入口列出，正文不据此立论） — https://source.android.com/docs/compatibility/cts

**未公开文档化 / 本次未逐字核验的边界（明确列出，正文与脚本不依赖以下任何一项）**

- **AOSP 源码树的整体目录结构**：本次检索未在 source.android.com 找到一份官方"目录结构"页面；正文第三节的结论只依赖 manifest 聚合机制，不依赖任何固定的目录布局。
- **可复现构建**：本次检索未在 AOSP/Pixel 官方文档中找到任何"重新构建可得同一产物"的承诺（记录为 `NO REPRODUCIBLE-BUILD CLAIM FOUND`）。第二节的区分据此成立；若未来出现该承诺，本文第二节需要重写。
- **`lunch` 三段式参数形式的逐字片段**：官方页面确认 `lunch` 由 `envsetup.sh` 定义（参考资料 9），但本次未取得三段式形式字符串的英文逐字片段；正文按语义描述该三元组，不引用为原文。
- **`$OUT_DIR` 之下的具体子路径布局**（如按产品/设备分目录）：未取得逐字来源，正文只使用"产物出现在 `$OUT_DIR`"这一已核验表述。
- **设备侧 `ro.build.*` 各属性的官方定义页**：本次未逐字核验属性定义文档；脚本 1 使用这些属性作为**观测字段**，判定强度受限于此（缺失一律记为数据缺口）。
- **`target_files` 包的内部结构与文件清单**、`ota_from_target_files` 的完整参数：未取得逐字来源，正文只使用"签名动作作用于 target files"这一已核验表述。
- **manifest XML 的元素与属性完整词表**（含 `revision`、`remote`、`include`、构建期执行类条目）：未取得逐字定义，脚本 2 因此只做文本级计数与集合对照，并对解析失败给出显式的"对照不成立"。
- **supported dynamic partitions 的逐项枚举**：已取得 super 的元数据描述（参考资料 19），但"支持列表"的逐字句子未取得；正文第八节不逐项列举。
- **VNDK 在各 Android 版本间的具体迁移细节**：已取得弃用宣告（参考资料 17），未逐字核验迁移步骤；第十三节只讨论读法与误读。
- **厂商实现（非 AOSP 公开部分）的分区、密钥与验证策略**：不属公开文档范围，本文一律不给结论，仅列为需要向厂商索取的证据（第十五、十八节）。
- **具体 CVE、漏洞与利用细节**：不属本文范围。相关口径见[内核漏洞与提权防御](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)、[Framework 信任边界与越权漏洞](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)与[远程代码执行](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)三篇。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
