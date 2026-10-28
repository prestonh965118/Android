---
layout: default
title: "Android 微架构侧信道与密钥抽取防御分析报告"
description: "微架构侧信道的六层归属（硅与微码、内核编译期、内核启动参数、平台策略、运行时与虚拟机、密钥业务实现）、设备侧唯一权威读数面 /sys/devices/system/cpu/vulnerabilities 的三前缀词表与十九个文件名、arm64 与 x86 词表不等价为什么不能跨架构比等级、M1–M10 缺陷类型学与 CWE-1303/200/208 对齐、Spectre/Meltdown/MDS/GDS/Retbleed 与 Arm 专属研究家族的读数方式、Android 特有的放大器（Zygote 共享、ART 与 JIT、DVFS）与收敛器（SELinux 对 perf 与 /proc、init.rc 的三个 sysctl、Profiling 模块）、层×责任×更新通道×设备侧证据的归属矩阵、四级判定模型与十项运营指标、两个只读核验脚本（侧信道姿态快照与两次快照的漂移比对）"
lang: zh-CN
permalink: /Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis/
---
# Android 微架构侧信道与密钥抽取防御分析报告

**Android Microarchitectural Side-Channel and Key-Extraction Defensive Analysis: Observation Channels, Mitigation Attribution, Vendor Vocabulary, and Read-Only Posture Verification**

- **报告日期**：2026-10-28
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动平台安全负责人、OEM/ODM 的内核与 SoC 集成工程师、密钥与密码实现团队（KeyMint/StrongBox/TEE 侧）、应用沙箱与 SELinux 策略维护者、设备风控与取证工程师、漏洞治理与补丁管理负责人、对 SDK 与运行时做性能/安全权衡取舍的框架维护者
- 视角声明：本文写的是**"一台 Android 设备上，微架构层面的信息泄露通道由哪些观察者构成、每一层缓解由谁负责、以及这些责任中哪一部分能在设备侧被只读证明"**。**具体泄露通道的构造方法、探针序列、信道容量测算与任何绕过步骤一律不在本文**；与相邻篇目的分工如下，本文只做交叉引用而不重复展开：
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核内存安全缺陷与提权取证；本文只谈内核作为**缓解的实现者与暴露面的宿主**，不谈内核缺陷本身）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（KeyMint/StrongBox/DICE 的机制与密钥存储；本文第十七章只讲"安全元件是否在侧信道口径内免疫"这一条边界，密钥栈的构造细节全部引向该篇）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（UID/SELinux 域/seccomp 三层；本文第四章只取其中与**观察通道**相关的那几行策略，第六、十一章把它当作收敛器来读）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层与信任边界整体坐标；本文的六层是其中"硅与微架构"这一竖切的放大图）
  - [Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（内核子系统与设备节点供给；本文只用 `/sys/devices/system/cpu/` 与 `/proc/sys/kernel/` 这两处公开接口，不重复展开内核实现）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（SPL 两档语义与固件到达性；本文把 SPL 严格当**时间锚点**使用，并说明为什么微架构缓解项的到达性它答不了）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（器件形态与可外部核验信号；本文第十三章讲的是**同一型号 SoC 在不同核心上的缓解差异**，两者互补）
  - [Android 无线协议栈与基带防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（邻近即远程的公告口径；本文第六章沿用同一口径解释"为什么本地权限的侧信道公告通常仍按高严重度处理"）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面与 R0–R4 分级；本文的类型学回答"这一域上可能站着哪一类泄露缺陷"）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（应用侧声明的弱点形态；本文第十五章的密钥业务层与它共享"证据从哪来"这条线，但对象是芯片与运行时而不是应用包）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（权限与身份判定缺陷；本文第十一章的"谁被允许读观测通道"按该篇的身份模型读取）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（协议词表与 Stable AIDL；本文不涉及 IPC 接口缺陷，只在第十一章用它的域标识口径解释采样权限的归属）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（`CONFIG_*` 与编译选项的装配口径；本文第八章的编译期缓解层依赖该篇的构建可复现性表述）
  - [Android 远程代码执行攻击面与防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（远程入口与执行链；本文第十八章说明**为什么缓解项不能作为可独立更新的模块单元下发**，远程代码执行链的构造细节全部引向该篇）
- 范围声明：全文只讨论**公开文档化**的微架构缺陷类别、厂商公告口径、内核与平台的公开缓解机制，以及在这些公开规则之上可安全实施的**只读**清点与比对流程。**不提供任何可用的漏洞利用代码、探针序列、内存布局偏移、原语构造、信道容量测算方法、缓存集数设定步骤或策略绕过流程；不给出任何针对具体机型或微码版本的攻防操作。**本文两个脚本一个只做设备侧只读采集（`get-state` / `getprop` / `uname` / `cat` / `ls` / `getenforce` / `mount` / `ps` 八类命令，全部指向公开节点），一个只做离线文本比对；两者都不写设备、不安装任何东西、不访问网络、不运行任何计时或缓存探针、不读取任何进程的非公开 `/proc` 条目。所有"核验"指读者对**自有或已明确获授权的测试设备**执行采集。

---

## 一、摘要：侧信道不是一个漏洞编号，是"谁能观察谁"

"这台手机有没有 Spectre 漏洞"这个问题在日常讨论里问不出答案，因为它同时混着三件不同的事：某一代硅的推测执行设计是否会被 transient execution 用到、这一台设备的内核有没有把对应缓解编译并启用、以及这台设备上**是否已经站着一个能够做观测的进程**。三件事共享"同一颗芯片"这个物理前提，但**责任人、更新通道、设备侧可证明性、以及"缓解生效"到底在数值上意味着什么**四样属性完全不同。把它们写进同一张检查表，是这一类风险治理里最常见的失效模式：表上每一行都有一个"是否已修"的格子，实际上有的格子问的是硬件设计、有的问的是内核配置、有的问的是启动参数、有的问的是"这台设备上到底有没有对手"。

本文的立场是：**微架构侧信道风险的管理单位不是"漏洞编号"，而是"观测通道 × 层 × 责任人 × 设备侧证据"这个四元组**。四个成分缺一个，一条缓解既没法派工，也没法验证——尤其没法验证，因为在设备侧可自证的只有其中一层。为此本文做四件事：

1. 把侧信道拆成六层（硅与微码 / 内核编译期 / 内核启动参数 / 平台策略 / 运行时与虚拟机 / 密钥业务实现），逐层给出**公开定义句与代码或文档位置**（第三、四章）；
2. 把设备侧唯一权威的读数面 `/sys/devices/system/cpu/vulnerabilities/` 的**取值词表与文件名单**做成一整章（第六章），因为内核 ABI 文档对这一面的定义极其克制：**"The files are named after the code names of CPU vulnerabilities. The output of those files reflects the state of the CPUs in the system."**，并且只给三种合法取值（参考资料 1）。这一句话决定了本文所有判定的边界：**这个面说的是"这一台设备的这些 CPU 的状态"，它不说芯片型号、不说公告、不说风险**;
3. 单列一章说明 **arm64 与 x86 的词表不等价**（第七章），因为绝大多数误读来自"拿两台设备的同名文件比等级"。最典型的一例：`csv2` 与 `csv3` 在 arm64 上**不会出现在 `/proc/cpuinfo` 的 Features 行里**（本文实测的上游 `cpuinfo.c` 打印名单含 `ssbs`、`sb`、`wfxt`、`afp`、`gcs`、`aes`，其中没有 `csv2`/`csv3`），而 Spectre-BHB 在 arm64 上**根本没有独立读数文件**，它的状态被拼进 `spectre_v2` 那句说明文字里（参考资料 2、3）。看不到 `csv2` 就判"这台设备没有 CSV2"，是把打印格式读成了硬件能力；
4. 给出十类缺陷类型学 M1–M10（第九章）、与 CWE 的对齐与一处必须纠正的常见错标（第十章），并把"能在设备侧只读观察到什么、观察不到什么"做成两个脚本与一份归属矩阵（第六、十一、十二、十三、十八、二十至廿五章）。

四条最重要的结论：

- **设备侧能自证的只有"内核认为现在的状态"这一层，其余三层都不能。** `vulnerabilities` 文件读回来的是内核在启动时算出的结论，不是芯片是否受影响的物理事实，也不是厂商公告。上游对 `old_microcode` 这一项把话说得很清楚：**"Systems that had microcode older than this at boot will say \"Vulnerable\"."** 以及 **"Just like all the other hardware vulnerabilities, exposure is determined at boot. Runtime microcode updates do not change the status of this vulnerability."**（参考资料 4）。这两句合起来是一个对本篇全部判定都成立的纪律：**运行时改了微码，读数不会变；所以"读数没变"也不能读成"微码没到"**。要证明后者，只能看 x86 上 `/proc/cpuinfo` 的 `microcode` 行，或者向厂商索取——arm64 侧连这个锚点都没有（第十三章）。
- **Android 平台把"谁能观察"这件事交给了 SELinux，而不是交给了一个 sysctl。** 平台侧的做法是：内核如果有 LSM 钩子，`init.rc` 就把系统级 `perf_event_paranoid` 写成 **-1（不设限）**，把访问控制整个交给策略；只有在没有钩子的老内核上才回落到 paranoid 值。原文写得很直白：**"In this case, the system-wide perf_event_paranoid sysctl is set to -1 (unrestricted use), and the SELinux policy is used for controlling access."**（参考资料 5）。这一条对判级的影响是决定性的：**在 Android 上读到 `perf_event_paranoid=-1` 不等于采样面开放**，因为真正管着它的是策略域；而策略侧确实做了这件事——`domain.te` 里的宏只对可调试域放行 perf 能力：**"Allow profiling using perf_event_open by traced_perf."**（参考资料 6）。脚本 1 因此把 paranoid 值与 SELinux 状态**绑在一起判**，并在第十一章明确把"低档位"限定为"该读取域内"。
- **"本地访问"在 Android 的公告口径里不是低风险同义词。** 侧信道类缺陷在厂商公告里几乎都写成"an authenticated user ... via a side channel with **local access**"（参考资料 7、8），而 Android 官方的位置口径是：**"Local attacks require the attacker to have prior access to the victim."**（参考资料 9）。这两句之间的落差正是本篇第十三章与第十五章要处理的东西：**在手机上，"事先获得访问"的最低门槛就是装进一个应用**，而一个普通应用与系统里所有其他应用共享的硅侧资源，比它共享的任何内核对象都多。这也是为什么本文把"同机邻居"当作一类需要单独清点的对象（第十二章），而不是当作攻击者能力的一部分。
- **单次快照不能证明"变好或变坏"，只有留存基线做差分层才能证明。** 脚本 2 因此被设计成**离线比对而不是设备采集**，并把两种"消失"分开处理：采集面未收窄时的消失记 `risk_gone_needs_proof`（仍需一条变更依据），采集面收窄时的消失记 `risk_gone_by_coverage`。第廿四章的一份真实差分样例给出这条判定的输出形态：同一批判据里，中高危信号码大面积"消失"，而缺口行数同时大幅上升，脚本把它们全部读作采集退化而不是整改成果。

从可执行验证的结果看：脚本 1 的 **73** 个信号码与脚本 2 的 **68** 个信号码，共 **141 个码位**，在 **15 组合成设备档案**与 **225 组快照对**（含 15 组自比对）上**全部至少触发一次**（第廿五章给出覆盖核对表与验收流程）。本文对"这一类风险可治理"的全部要求就是一句话：**每一层有责任人、每一条通道有可核验的到达证据、每一个读不到的字段被记成缺口而不是通过。**

---

## 二、先分清三套口径：推测执行类、缓存与内存访问类、密钥实现类

这一领域里"侧信道"一词至少压着三类属性完全不同的缺陷。不先分开，后面所有分级与归属都会串味。

| 口径 | 公开定义句（原文） | 泄露发生在哪 | 缓解归属层 | 设备侧是否有读数 |
| --- | --- | --- | --- | --- |
| **A. 推测执行 / transient execution 类** | **"Systems with microprocessors utilizing speculative execution and indirect branch prediction may allow unauthorized disclosure of information to an attacker with local user access via a side-channel analysis."**（参考资料 7）；**"Spectre breaks the isolation between different applications. It allows an attacker to trick error-free programs, which follow best practices, into leaking their secrets."**（参考资料 10） | 猜测执行把不该被读到的数据带进缓存/预测器状态，再经旁路观测出来 | 硅与微码 + 内核编译期 + 内核启动参数，三层同时 | **有**：`vulnerabilities/` 下与该项同名的文件（第六章） |
| **B. 缓存、TLB 与缓冲区类（不依赖推测执行）** | **"Microarchitectural Store Buffer Data Sampling (MSBDS): Store buffers on some microprocessors utilizing speculative execution may allow an authenticated user to potentially enable information disclosure via a side channel with local access."**（参考资料 8）；**"Unlike existing attacks, our attacks can leak arbitrary in-flight data from CPU-internal buffers (Line Fill Buffers, Load Ports, Store Buffers), including data never stored in CPU caches."**（参考资料 11） | 数据在 CPU 内部缓冲里"顺路"被后任读者取到，即使它从不落进缓存 | 硅与微码为主，内核只能冲刷或禁用特性 | **部分**：`mds`/`mmio_stale_data`/`srbds`/`gather_data_sampling`/`reg_file_data_sampling` 有文件（第六章名单） |
| **C. 密钥与协议实现的计时/访问模式类** | **"Two separate operations in a product require different amounts of time to complete, in a way that is observable to an actor and reveals security-relevant information about the state of the product, such as whether a particular operation was successful or not."**（参考资料 12，CWE-208） | 与芯片设计无关：实现本身在不同密钥位上走了不同时长或不同内存访问模式 | **完全在实现层**：库、TEE 固件、应用代码 | **无**：任何 `vulnerabilities` 读数都不覆盖这一类 |

三条分开之后立刻能得到的判断：

1. **A 与 B 在设备侧共用一个读数面，但责任人不共用。** A 类的内核侧缓解是编译期与启动期决定的，B 类的核心缓解是微码与"要不要冲刷缓冲"决定的。上游对 MDS 的默认策略把这一点写成了条件句：**"If a CPU is affected and the microcode is available, then the kernel enables the mitigation by default."**，并且 **"The processor is vulnerable but microcode is not updated. The mitigation is enabled on a best effort basis."**（参考资料 13）——**"best effort"** 是内核自己的措辞，不是本文的推测。所以一台设备上 `mds` 读到 `Vulnerable: Clear CPU buffers attempted, no microcode`，含义是"框架在位、硅侧缺"，这一格在第十三章归到微码与固件责任，不在内核责任。
2. **C 类与芯片型号基本无关，且它是 Android 上唯一"每台设备都可能自带"的一类。** 一张 A 类/B 类缺陷的清单是芯片厂商与内核决定的，读者只能等；C 类清单取决于这台设备上跑的是什么密码实现、什么解锁判定、什么时限逻辑。本文第十五章把它单独处理，并且**不给任何可操作的探测方法**——只给"这一类缺陷的证据必须来自实现侧而不是设备侧"这一条归属结论。
3. **把 C 类算进"这台设备有没有 Spectre"，是这一领域最常见的错误结论。** 一台 `vulnerabilities` 全部读到 `Mitigation` 的设备，仍然可能因为一个把密钥位编进分支的实现而泄露密钥；反过来一台读到 `Vulnerable` 的设备，如果没有进程能够安排观测，实际暴露也不成立。这就是第一章说"管理单位是四元组而不是编号"的具体含义。

## 三、六层分层地图：从硅到密钥实现

把"缓解"当成一件事看会漏掉大部分内容；把它按**决定它的东西在哪**切开，得到六层。每层后面的两列是本文真正要用的：**这层的负责人是谁、这层在设备上能不能被只读证明**。

| # | 层 | 这一层决定什么 | 公开落点 | 责任人 | 设备侧证据 |
| --- | --- | --- | --- | --- | --- |
| **H1** | 硅设计与微码 | 是否存在该通道、有没有供内核使用的抑制/冲刷指令或 MSR、补丁级别是否达标 | 芯片厂商公告（Intel-SA-*、AMD-SB-*）；x86 上 `/proc/cpuinfo` 的 `microcode` 行 | 芯片设计方 + 整机固件包维护方 | **x86：有（版本号）；arm64：无**（第十三章） |
| **H2** | 内核编译期 | 缓解代码是否被编进内核：`CONFIG_MITIGATION_*`、rethunk、`__user` 指针清理、PAN/PTI 之类 | 内核 config；上游文档明确说档位选择依赖编译选项（参考资料 14） | SoC BSP 内核维护方 / AOSP 公共内核 | **间接**：`vulnerabilities` 的说明文字里会写出用了哪种实现（第七章） |
| **H3** | 内核启动参数 | 是否把已编译的缓解整体或逐项关掉：`mitigations=`、`nospectre_v2`、`ssbd=`、`spec_store_bypass_disable=` 等 | `Documentation/admin-guide/kernel-parameters.txt`（参考资料 15）；`/proc/cmdline` | 整机厂商（出厂 cmdline）+ 任何能改 boot image 的人 | **有**：`/proc/cmdline` 可读时逐串可判（第八章） |
| **H4** | 平台策略与内核 sysctl | 谁被允许使用观测通道：SELinux 域、`perf_event_paranoid`、`kptr_restrict`、`dmesg_restrict`、`/proc` 挂载选项 | AOSP `sepolicy` 与 `system/core/rootdir/init.rc`（参考资料 5、6） | 平台与厂商 sepolicy 维护者 | **有**：`getenforce` + 三个 sysctl + `mount`（第十一章） |
| **H5** | 运行时与虚拟机 | 同一颗芯片上有哪些邻居、进程是否共享母进程与代码缓存、guest 与 host 是否同核 | Android Zygote/ART/JIT 形态；`vm.dirty_*` 之外的虚拟化边界由内核 `vmscape`/`its` 类读数体现（参考资料 1、2） | 平台运行时 + 虚拟化实现 | **部分**：`ps` 能清点邻居，测不出缓存共享程度（第十二章） |
| **H6** | 密钥与协议实现 | 密钥值是否被编进控制流或访存模式、是否使用查表实现 | 库与实现自身；CWE-200/208/1303 口径（参考资料 12、16、17） | 库维护者、TEE 固件方、应用开发者 | **无**：设备侧没有任何只读读数覆盖这一层（第十五章） |

三条结构性结论：

- **可证明性与危险度不成比例。** 设备侧可自证的只有 H3、H4 两层，加上 x86 的 H1 版本号锚点；H2 只能从说明文字里间接推断，H5 只能数进程，H6 **完全不能**。而实际风险最高、整改成本最低、也最常被忽略的恰是 H6。本文因此在第十九章把"设备侧读数全部良好"明确定义为**不构成结论**的一种情形。
- **H1/H2/H3 是"能不能"，H4/H5 是"有没有人"。** 前三层决定一台设备是否具备泄露的物理条件；后两层决定这台设备上有没有一个能安排观测的进程。一条只收紧 H4 的整改（例如关掉非调试域的 perf 能力）会让设备侧**读数完全不变**——`vulnerabilities` 九个文件一个字都不会改，只有脚本 1 的 `POL,selinux` 与 `SYSCTL,*` 三段会变。这解释了为什么脚本 1 要把两组读数放在同一次采集里：**分开采会把"策略收紧"和"缓解生效"混成同一个"看起来更安全了"。**
- **H3 是唯一一层"改了不需要重编译也不需要升级"的层，因此也是唯一一层最容易在无人注意时变化的层。** 上一轮 OTA 不会动 cmdline，一次性能调优、一次开发者解锁流程、一次为跑分做的定制内核就会。脚本 2 因此把 cmdline 当作**漂移比对的主对象之一**（第廿三章的 `cmdline_disable_token_added`、`cmdline_mitigations_value_changed` 都只为这一层存在）。

## 四、各层的公开定义句与代码位置

本章把上面六层的公开依据逐层落到**可点开的文件与可逐字引用的句子**，供后面的判定模型引用。**注意：本章只列取得逐字内容的来源；未取得逐字内容的部分在第三十章的边界清单里单列。**

**H1 硅与微码。** Intel 对 Spectre/Meltdown 这一族的原始定义句（参考资料 7）与后续各族公告使用同一句式模板：**"TSX Asynchronous Abort condition on some CPUs utilizing speculative execution may allow an authenticated user to potentially enable information disclosure via a side channel with local access."**（参考资料 18）；对 2025 年的间接分支预测器公告，同一模板是：**"Exposure of sensitive information caused by shared microarchitectural predictor state that influences transient execution in the indirect branch predictors for some Intel® Processors may allow an authenticated user to potentially enable information disclosure via local access."**（参考资料 19）。AMD 的表述方式不同，它更强调与软件缺陷的组合条件：**"AMD is aware of research from the VUsec group at Vrije Universiteit Amsterdam and believes that these issues are only exploitable in conjunction with software vulnerabilities related to incorrect speculation of specific sequences as described below."**，并给出实现侧的整改手段：**"Potential software vulnerabilities can be mitigated by inserting a LFENCE between the overwriting of the instructions and the execution of the overwritten instructions."**（参考资料 20）。**AMD 侧关于 Spectre v2 是否需要微码的那句常被引用的表述，本次未取得可逐字引用的整段（原公告路径返回 404），本文不据此立论**（第三十章边界 3）。

**H2 内核编译期。** 上游对"选哪个缓解实现"这件事的公开描述把三个变量并列：**"Selecting 'on' will, and 'auto' may, choose a mitigation method at run time according to the CPU, the available microcode, the setting of the CONFIG_MITIGATION_RETPOLINE configuration option, and the compiler with which the kernel was built."**（参考资料 14）。这句话是本文第八章"档位与实现不是一回事"的依据：**`auto` 的结论同时依赖微码可用性与编译器**，所以同一个内核版本在两台设备上报出不同的说明文字属预期事件，不是配置漂移。

**H3 内核启动参数。** `mitigations=` 条目本身：**"Control optional mitigations for CPU vulnerabilities. This is a set of curated, arch-independent options, each of which is an aggregation of existing arch-specific options."**，以及一个常被忽略的前提：**"Note, \"mitigations\" is supported if and only if the kernel was built with CPU_MITIGATIONS=y."**（参考资料 15）。`off` 档的后果与它的聚合范围由同一条目逐项列出，其中包括 **`nospectre_bhb [ARM64]`**、**`ssbd=force-off [ARM64]`**、**`nopti [X86,PPC]`**、**`spec_store_bypass_disable=off [X86,PPC]`**、**`gather_data_sampling=off [X86]`** 等（参考资料 15，完整清单见第八章）。arm64 的 BHB 专用开关单独成条：**"nospectre_bhb [ARM64,EARLY] Disable all mitigations for Spectre-BHB (branch history injection) vulnerability. System may allow data leaks with this option."**（参考资料 15）。

**H4 平台策略与 sysctl。** 平台侧的全部关键行为集中在 `init.rc` 的一段注释与四条 `write`（参考资料 5，完整原文见第十一章）。策略侧的三条直接相关规则：**"# profiling needs to read /proc/sys/kernel/perf_event_max_sample_rate" / `allow domain proc_perf:file r_file_perms;`** 与 **"Allow profiling using perf_event_open by traced_perf." / `can_profile_perf({ dumpable_domain -app_zygote -hal_configstore_server -webview_zygote -zygote })`**（参考资料 6）。跨 UID 可见性侧，策略用的是 neverallow 而不是挂载选项：**"# A different form of hidepid=2 like protections" / `neverallow appdomain { domain -appdomain }:file no_w_file_perms;`**，以及 `neverallow { appdomain -shell } { domain -appdomain }:file no_rw_file_perms;`（参考资料 6）。

**H5 运行时与邻居。** 这一层没有一句"官方定义说邻居不能共享缓存"的公开文档，本文因此**不使用任何此类表述**；可逐字引用的只有工具侧的可用范围：**"However, security restrictions mean that only apps with android::debuggable set to true can be profiled."** 与 **"If you are on a rooted device, you can profile any app."**（参考资料 21）。Android 16 引入的 Profiling 模块则给出了应用侧可请求的观测类型与结果范围：**"Apps can request profiling using Java heap dump, heap profile, stack sampling, or system trace, and set select configurations for each."** 以及 **"Results are redacted to contain information relating to the requesting process only and delivered to the app's storage directory for further processing by the app."**（参考资料 22）。后一句是本文第十五章"平台把可观测范围收缩到自己进程之内"这条判断的直接依据。

**H6 密钥实现。** 这一层没有任何 Android 侧的公开定义句可用（不存在"Android 规定密码实现必须常数时间"这类文档），本文只用 CWE 的通用口径（参考资料 12、16、17）与一句关于共享微架构资源的中性定义作为分类依据，并把所有"某库/某实现是否常数时间"的判断**留给读者按第十五章的方法自查**。
## 五、缺陷家族与各自的公开口径：把"公告名"落到"文件名"上

第一章说管理单位是"观测通道 × 层 × 责任人 × 设备侧证据"。要填这张表，第一件事是把研究界与厂商公告里的**家族名**和内核对外的**文件名**一一对上，因为设备侧只认后者。本章按"泄露发生在 CPU 里的哪一段状态"把家族分五组，每行给出**上游实现里逐字写出的读数词表**（取自参考资料 23、24 的本地源码，全部逐字核对）。

### 五之一 分支预测与间接跳转类

| 家族（公告/研究名） | 设备侧文件名 | 上游读数词表（逐字，除公共后缀） |
| --- | --- | --- |
| Spectre v1（Bounds check bypass, CVE-2017-5753） | `spectre_v1` | x86：**`Vulnerable: __user pointer sanitization and usercopy barriers only; no swapgs barriers`** / **`Mitigation: usercopy/swapgs barriers and __user pointer sanitization`**；arm64 只输出一条常量（第七章） |
| Spectre v2（Branch target injection, CVE-2017-5715） | `spectre_v2` | **`Vulnerable`** / **`Mitigation: Retpolines`** / **`Vulnerable: LFENCE`** / **`Mitigation: Enhanced / Automatic IBRS`** / **`Mitigation: Enhanced / Automatic IBRS + LFENCE`** / **`Mitigation: Enhanced / Automatic IBRS + Retpolines`** / **`Mitigation: IBRS`**，其后按条件拼接 `; IBPB: always-on|conditional|disabled`、`; IBRS_FW`、`; STIBP: disabled|forced|always-on|conditional`、`; RSB filling`、`; PBRSB-eIBRS: SW sequence|Vulnerable|Not affected`、`; BHI: Not affected|BHI_DIS_S|SW loop, KVM: SW loop|Retpoline|Vulnerable`，最后追加 `User space: …` 一串（参考资料 23） |
| Retbleed（同分支预测器族，2022） | `retbleed` | **`Vulnerable`** / **`Mitigation: untrained return thunk`** / **`Mitigation: IBPB`** / **`Mitigation: IBRS`** / **`Mitigation: Enhanced IBRS`** / **`Mitigation: Stuffing`** |
| SRSO（Speculative RAS Overflow） | `spec_rstack_overflow` | **`Vulnerable`** / **`Vulnerable: No microcode`** / **`Vulnerable: Safe RET, no microcode`** / **`Vulnerable: Microcode, no safe RET`** / **`Mitigation: SMT disabled`** / **`Mitigation: Safe RET`** |
| ITS（Indirect Target Selection） | `indirect_target_selection` | **`Vulnerable`** / **`Mitigation: Vulnerable, KVM: Not affected`** / **`Mitigation: Aligned branch/return thunks`** / **`Mitigation: Retpolines, Stuffing RSB`** |
| TSA（Transient Scheduler Attacks） | `tsa` | **`Vulnerable`** / **`Vulnerable: No microcode`** / **`Mitigation: Clear CPU buffers: user/kernel boundary`** / **`Mitigation: Clear CPU buffers: VM`** / **`Mitigation: Clear CPU buffers`** |
| Spectre-BHB / Branch history injection（arm64 专属） | **无独立文件** | 状态拼在 `spectre_v2` 的文字里：`", BHB"` 或 `", but not BHB"`（第七章、参考资料 2） |

三条读法上的硬约束：

1. **`Mitigation:` 开头的读数里可以合法地写着 `Vulnerable`。** `indirect_target_selection` 的 `Mitigation: Vulnerable, KVM: Not affected` 是上游逐字写出的字符串（参考资料 23），它的意思是"内核侧已处理、KVM 侧不受影响"。任何按"字符串里有没有 Vulnerable"来判级的实现都会在这里翻车——这正是脚本 1 把分类器改成**前缀锚定**的原因（第廿一章）。
2. **`Vulnerable: LFENCE` 不是"没缓解"，而是"用了被内核认为不充分的缓解"。** AMD 平台上把 LFENCE 当作分支隔离手段时，内核仍旧在开头写 `Vulnerable`。这一格在归属矩阵（第十六章）里落在 H2 编译期 + H1 硅，不落在运维侧。
3. **`spectre_v2` 是八个片段拼出来的长句，跨设备比对必须按片段拆。** 两台设备前半句相同、`; STIBP:` 片段不同，是常见现象；把它当成"整串不等→配置漂移"会制造大量噪声。脚本 2 因此对 `VULN` 行同时保留**状态前缀**与**整串文字**两个字段，前缀变了才升级，整串变了只记 `INFO`。

### 五之二 缓冲区数据采样类（不依赖错误预测，依赖"缓冲未清零"）

| 家族 | 设备侧文件名 | 上游读数词表（逐字） |
| --- | --- | --- |
| MDS / MSBDS / MFBDS / MLPDS（2019） | `mds` | **`Vulnerable`** / **`Mitigation: Clear CPU buffers`** / **`Vulnerable: Clear CPU buffers attempted, no microcode`**，后缀 `; SMT vulnerable|mitigated|disabled` 或 `; SMT Host state unknown` |
| TAA（TSX Asynchronous Abort） | `tsx_async_abort` | **`Vulnerable`** / **`Mitigation: Clear CPU buffers`** / **`Vulnerable: Clear CPU buffers attempted, no microcode`** / **`Mitigation: TSX disabled`**，后缀同上 |
| SRBDS（RDRAND/RDSEED/PRS 特殊寄存器缓冲） | `srbds` | **`Vulnerable`** / **`Vulnerable: No microcode`** / **`Mitigation: Microcode`** / **`Mitigation: TSX disabled`** / **`Unknown: Dependent on hypervisor status`** |
| GDS / Downfall 同族（2022–2023 收集类） | `gather_data_sampling` | **`Vulnerable`** / **`Vulnerable: No microcode`** / **`Mitigation: AVX disabled, no microcode`** / **`Mitigation: Microcode`** / **`Mitigation: Microcode (locked)`** / **`Unknown: Dependent on hypervisor status`** |
| RFDS（Register File Data Sampling） | `reg_file_data_sampling` | **`Vulnerable`** / **`Mitigation: Clear Register File`** / **`Vulnerable: No microcode`** |
| MMIO stale data | `mmio_stale_data` | **`Vulnerable`** / **`Mitigation: Clear CPU buffers`** / **`Vulnerable: Clear CPU buffers attempted, no microcode`**，后缀同上 |

这一组里最值得记住的是 **`Unknown: Dependent on hypervisor status`**：SRBDS 与 GDS 在虚拟机里会直接给出这句话，而它既不是 `Vulnerable` 也不是 `Mitigation`。设备侧读到它，含义是"这一格在这台机器上不可判定"，脚本按 `L2-WARN` 记 `vuln_state_unknown_prefix` 而不是当成缓解生效（第廿一章）。**虚拟化形态（Android 上常见于把基带/安全子系统跑在客户机里的实现）会把这一族的证据整格抹掉**，这是第十八章"安全元件边界"的物理来源。

另一条是 **`; SMT Host state unknown`**：当 `X86_FEATURE_HYPERVISOR` 置位时，内核不再报 SMT 状态而是报这句（参考资料 23）。也就是说**在客户机里，连"关掉超线程能收敛这一族"这个判断都无法自证**。运维侧若要拿 SMT 状态做派工，必须先确认自己读的是宿主而不是客户机。

### 五之三 页表、地址翻译与缓存映射类

| 家族 | 设备侧文件名 | 上游读数词表（逐字） |
| --- | --- | --- |
| Meltdown / variant 3（2018） | `meltdown` | x86 与 arm64 都用 **`Mitigation: PTI`**，另有 `Vulnerable` / `Not affected`（arm64 分支见参考资料 2） |
| L1TF（Foreshadow, 2018） | `l1tf` | 默认句 + 条件后缀 `; VMX: auto|vulnerable|conditional cache flushes|cache flushes|EPT disabled|flush not necessary` 与 `; SMT …` |
| itlb_multihit（LDS/MMIO，2019–2020） | `itlb_multihit` | **`Processor vulnerable`**（这是上游唯一一个**三种前缀都不符合**的固定读数） |
| VMSCAPE（2024，虚拟化侧 EPT 指示位） | `vmscape` | 名单内文件；本文未取得其公开定义句（第三十章边界 9） |

`itlb_multihit` 的 `Processor vulnerable` 值得单独说一句：**内核 ABI 文档规定的三种前缀，上游自己的实现就没有全部遵守。** 所以任何"按前缀白名单校验读数格式"的做法都不能反过来用作"这个设备状态正常"的证据——前缀不认识的读数不是"格式异常"，而是**必须原样保留、单独立一码**的证据。脚本 1 用 `vuln_other_state` / `vuln_state_unknown_prefix` 两码分别承接"空串/无法归类"与"非三前缀开头"，就是为这一条准备的。

### 五之四 存储转发类

`spec_store_bypass`（Spectre v4 / SSB / CVE-2018-3639）的词表最干净：**`Vulnerable`** / **`Mitigation: Speculative Store Bypass disabled`** / **`Mitigation: Speculative Store Bypass disabled via prctl`** / **`Mitigation: Speculative Store Bypass disabled via prctl and seccomp`**。

注意后两条：**"via prctl" 意思是内核没有全局关掉它，而是把开关交给每个进程自己**。这一格在 Android 上的实际含义要谨慎读：策略与运行时决定了一个进程会不会真的去调 prctl，设备侧的读数只说"机制在位"，不说"这台设备上的应用都被 SSB 关闭保护"。这是第七章把"缓解形态"与"生效范围"分开登记的原因，也是第十五章把 SSBS/SB 特性位单独计数、而不是并入"已缓解"的原因。

### 五之五 `old_microcode`：它不是缺陷家族，是一条"到达性元读数"

`vulnerabilities/old_microcode` 与其余十八个文件性质完全不同：**它报告的不是某条微架构通道，而是"本机启动时加载的微码是否早于内核要求的最低版本"**。上游文档的两句话决定了本篇对它的用法：**"Systems that had microcode older than this at boot will say \"Vulnerable\"."** 与 **"Just like all the other hardware vulnerabilities, exposure is determined at boot. Runtime microcode updates do not change the status of this vulnerability."**（参考资料 4）。它的实现还多一条：**虚拟机里直接读 `Unknown: running under hypervisor`**（参考资料 23）。

三条派生纪律（第十八章展开）：

- 这一项读 `Vulnerable` 时，责任人是**固件包/引导链**，不是内核维护者；
- 这一项读 `Not affected` 时，**不能**推成"微码是新的"，只能推成"启动时不低于该内核的最低要求线"——而这条线本身随内核版本变；
- 这一项读 `Unknown: …` 时，本设备上没有任何硅侧到达性证据，只能向厂商索取带型号的更新记录。

## 六、设备侧唯一权威读数面：三前缀词表、十九个文件与它们的成立条件

上一章按家族看，本章按接口看。`/sys/devices/system/cpu/vulnerabilities/` 是 Android 设备上**唯一**由内核对外承诺的推测执行缓解状态读数面；本文其余章节凡是给出"能自证/不能自证"的判断，都以这一面的实现为界。

### 六之一 ABI 只承诺两件事

上游 ABI 文档对这一面的全部规范性表述是：**"The files are named after the code names of CPU vulnerabilities. The output of those files reflects the state of the CPUs in the system."**，取值只定义三种形态：`Not affected`、`Vulnerable`、`Mitigation: $M`（参考资料 1）。

这句话里有三个容易被跳过的限定：

1. **"the state of the CPUs in the system"——说的是这台机器上这批 CPU 的状态。** 它不是型号查询、不是公告索引、不是风险评级。用两台设备的同名文件互推"同型号所以同风险"，或者用一台设备的读数为另一个型号背书，都不在这句话的授权范围内。
2. **"reflects the state"——是内核算出来的结论，不是硬件事实。** 结论由三样东西决定：编译期选项、启动早期检测（`boot_cpu_has_bug`）、以及 cmdline 档位（参考资料 23）。第三章的 H1–H3 三层各自都可能改变这个字符串，而设备侧无法从这个字符串里区分是哪一层导致的。
3. **文档不承诺文件集合。** 名单是随内核版本增长的：`vmscape`、`tsa`、`indirect_target_selection`、`ghostwrite` 都是近几个版本才加入。**所以"这个文件不存在"永远有两种成因，必须靠 `uname -r` 与厂商内核配置去区分，设备侧自己分不出来**（脚本 1 的 `vuln_core_file_absent` 因此固定为 `L0-LOW` 缺口，不升级）。

### 六之二 上游当前的十九个文件名（完整名单）

以下名单逐字取自 `drivers/base/cpu.c` 的 `cpu_root_vulnerabilities_attrs[]`（参考资料 24）——**这是本文所有"这个文件名是不是上游的"判断的基准**：

```
meltdown  spectre_v1  spectre_v2  spec_store_bypass  l1tf  mds
tsx_async_abort  itlb_multihit  srbds  mmio_stale_data  retbleed
spec_rstack_overflow  gather_data_sampling  reg_file_data_sampling
ghostwrite  old_microcode  indirect_target_selection  tsa  vmscape
```

四个立刻可用的判读结论：

- **名单里没有 `downfall`、没有 `ssb`、没有 `spectre_bhb`、没有 `csv2`。** 设备清单里出现这些名字，说明该内核带厂商补丁或补丁集，其命名与取值口径不在本文可核验范围内。脚本 1 对此不是忽略而是**单独出码并计数**：每一处非名单名出 `vuln_name_outside_abi`（`L1-INFO`），并累加进 `COUNT,vuln_outside_abi`。**注意 `vuln_files_total`、`vuln_mitigated` 等前缀分组计数仍然包含这些项**——脚本没有把它们从统计里扣除，所以任何消费这份快照的看板在算"缓解到位率"时必须自行减去 `vuln_outside_abi`，否则厂商自造词会按前缀被算进分子或分母。第二十六章把这一条写进 K4 的分子定义。
- **`vulnerabilities` 这一组只有在上一个条件成立时才存在**：整个注册被 `#ifdef CONFIG_GENERIC_CPU_VULNERABILITIES` 包住（参考资料 24）。这意味着**目录不存在 ≠ 内核不管这些缺陷**，很可能只是该内核没开这个配置项、或该架构走的是另一条上报路径。脚本 1 把这种情况记 `vuln_dir_unread` / `kernel_pre_vulnfs` 并把整套推测执行判定降级为缺口。
- **同一名单在所有架构上注册。** 这一点很多人不知道：**某个架构没有为某个文件实现 show 函数时，该文件仍然存在并读回 `Not affected`**，因为上游为每一项都准备了弱别名回退：**`CPU_SHOW_VULN_FALLBACK(...)` → 指向 `cpu_show_not_affected()`，后者的实现就是一行 `return sysfs_emit(buf, "Not affected\n");`**（参考资料 24）。
- **ABI 文档页与 `cpu.c` 的名单并不同步。** 本地核对同一份上游树：ABI 页的 `What:` 行只列 **17** 个具体文件（缺 `spec_rstack_overflow`、`ghostwrite`），而 `cpu.c` 的注册数组是 **19** 个。所以本文一切"这个名字是不是上游的"判断都以**注册数组**为准，不以文档页为准（参考资料 1、24）——脚本 1 的 `VULN_ABI_NAMES` 照数组写。这条差异本身也是"文档不承诺文件集合"（六之一第 3 点）的一个具体例证：**看文档页做白名单，会把两个真实存在的上游名字判成厂商自造词。**

最后一条的运维后果很重：**"十九个文件全部读到 `Not affected`"不是一次好消息，而是一次信息量为零的读数。** 它有两种无法在设备侧区分的成因：这批 CPU 真的都不受影响；或者这台内核根本没为这些项实现上报，全部走回退。脚本 1 因此把 `COUNT,vuln_not_affected` 与 `COUNT,vuln_total` 同时输出，并在第十九章规定：**两者相等时，本快照的推测执行段一律记"不构成结论"，不得进入 K1/K2 类达标统计。**

### 六之三 读数的四种形态与各自的判定

| 形态 | 例子（逐字来源） | 脚本判定 | 归属层 |
| --- | --- | --- | --- |
| `Not affected` | 回退实现与 arm64 `cpu_show_meltdown` 分支 | `vuln_not_affected`（`L1-INFO`），并与总数联判 | 不可判层（见六之二） |
| `Vulnerable…` | `Vulnerable`、`Vulnerable: LFENCE`、`Vulnerable: No microcode` | `vuln_vulnerable`（`L3-HIGH`），带 `no microcode` 时再叠 `vuln_note_microcode` | H1 为主 |
| `Mitigation: …` | `Mitigation: Clear CPU buffers`、`Mitigation: Safe RET`、`Mitigation: PTI` | `vuln_mitigated`（`L1-INFO`），带 `; SMT vulnerable` 时叠 `vuln_note_smt` | H2/H3 |
| 前缀外形态 | `Processor vulnerable`（itlb）、`Unknown: running under hypervisor`、`Unknown: Dependent on hypervisor status`、`Mitigation: Vulnerable, KVM: Not affected` | 以 `Unknown` 起头 → `vuln_state_unknown_prefix`（`L2-WARN`）；其余不属于三前缀 → `vuln_other_state`（`L1-INFO`）；空串/仅换行 → `vuln_file_empty`（`L0-LOW`） | 缺口层 |

两个后缀形态单独登记，因为它们改变"缓解够不够"的结论而不改变前缀：

- **`; SMT …`**：`vuln_note_smt`。MDS/TAA/MMIO 三族在 SMT 打开时即使有 `Mitigation:` 前缀也仍报 `; SMT vulnerable`（参考资料 23）。**关掉 SMT 是这一族唯一在设备侧可自证的额外收敛**，但设备侧只能读到结论（`sched_smt_active()` 的呈现），不能读到"能不能关"。
- **`eBPF` / `Unprivileged eBPF enabled`**：`vuln_note_ebpf`（`L2-WARN`）。x86 有 `Vulnerable: eIBRS with unprivileged eBPF` 与 `Vulnerable: eIBRS+LFENCE with unprivileged eBPF and SMT` 两条专门读数，arm64 有 `Vulnerable: Unprivileged eBPF enabled`（参考资料 2、23）。**这三条的共同含义是：本来判为已缓解的分支，因为非特权 eBPF 可用而被内核重新判为 Vulnerable。** 这是一条真正的跨层组合——H1 已缓解、H4 未收紧，合起来才出现敞口，也正是第一章说"单次快照要分层看"的具体一处。脚本 1 对 arm64 的 `spectre_v2` 走专门的 `v2_arm_ebpf_reopens`，避免与非 arm64 的同名片段重复计数。

### 六之四 这一面不覆盖什么

明确写下来，免得后面各章被过度解读：

- **不覆盖缓存侧信道本身**（Prime+Probe、Flush+Reload、Prefetch 探测、TLB 探测）。这些通道的存在不由 `vulnerabilities` 报告，设备侧也没有任何只读接口说明"这台机器上缓存可被谁观测"。**本文不提供任何测量方法**，只在第十四章把"可观测面由谁能运行什么决定"作为策略问题处理。
- **不覆盖密钥实现层（H6）**。`mds` 读 `Mitigation` 与"这台机器上的密码库是否常数时间"没有任何关系。
- **不覆盖微码到达性**（除 `old_microcode` 那一格，且它只说"启动时不低于最低要求"）。
- **不覆盖具体核。** 该面注册在 `/sys/devices/system/cpu/` 根上，读的是** boot CPU 的结论**（`boot_cpu_has_bug`、`boot_cpu_has`，参考资料 23）。**big.LITTLE / 多簇 SoC 上，这一个字符串不代表每个核**（第十三章）。这是 Android 上最容易被误用的一条：**一台八核设备只有一个 `spectre_v2` 文件。**
- **不覆盖已加载但运行时更新的微码。** 上游对 `old_microcode` 已明说运行时更新不改变读数（参考资料 4）。

## 七、arm64 与 x86 的词表不等价：为什么不能跨架构比等级

Android 设备绝大多数是 arm64。把 x86 世界积累的判断习惯直接搬过来，会产生一批系统性误判。本章全部依据本地核对过的 `arch/arm64/kernel/errors.c`（参考资料 2）与 `arch/arm64/kernel/cpuinfo.c`（参考资料 3）。

### 七之一 四条实测到的词表差异

1. **`spectre_v1` 在 arm64 上是一句常量。** 实现只有一条返回路径，输出 **`Mitigation: __user pointer sanitization`**（参考资料 2）。也就是说：**两台 arm64 设备的 `spectre_v1` 一定相同，这个文件在这台设备上不提供任何区分度。** 把它纳入"缓解到位率"统计会让所有 arm64 设备都得满分。
2. **BHB 没有独立文件，状态被拼进 `spectre_v2`。** arm64 的实现先取 `arm64_get_spectre_bhb_state()`，再用一个三值映射函数把结果换成后缀串：**`SPECTRE_UNAFFECTED → ""`、`SPECTRE_VULNERABLE → ", but not BHB"、SPECTRE_MITIGATED → ", BHB"`**（参考资料 2）。因此：
   - 读到 **`Mitigation: CSV2, BHB`** 与读到 **`Mitigation: CSV2, but not BHB`** 是两种不同的暴露，而**前缀完全一样**；
   - 读到 **`Mitigation: Branch predictor hardening, but not BHB`** 时，主句已缓解、BHB 未缓解——**这类"两半不一致"的读数必须按片段判**，脚本 1 为此有 `vuln_bhb_covered_in_v2`（`L1-INFO`）与 `vuln_note_bhb_partial`（`L2-WARN`）两码；
   - 设备清单里若出现 `spectre_bhb` 文件，那不是上游 ABI，而是厂商自行导出，脚本 1 记 `vuln_name_outside_abi` 并排除出分组统计。
3. **arm64 的 `spectre_v2` 有一条"主句不受影响也不能报不受影响"的规则。** 实现里，当主状态为 `SPECTRE_UNAFFECTED` 而 BHB 状态不是不受影响时，代码**强制把主串换成 `CSV2` 并继续走 Mitigated 分支**，注释原话是 **"Platforms affected by Spectre-BHB can't report \"Not affected\" for Spectre-v2."**（参考资料 2）。**这是一条架构级的"读数下限"**：反过来说，如果在 arm64 上看到裸的 `Vulnerable` 或带 BHB 后缀的 `Mitigation:`，都不能直接和 x86 的同名读数对齐。
4. **非特权 eBPF 在 arm64 上能把已缓解重新判为敞口。** 实现在 `SPECTRE_MITIGATED` 分支里加了一条判断：若 BHB 已缓解且 `_unprivileged_ebpf_enabled()` 为真，则输出 **`Vulnerable: Unprivileged eBPF enabled`**（参考资料 2）。这与 x86 的 eIBRS 两条读数同构，是本文在第一章写"跨层组合才是窗口"的最直接证据——**同一个内核、同一颗 CPU、同一份微码，仅仅因为 `kernel.unprivileged_bpf_disabled` 的差别，sysfs 读数就从 `Mitigation:` 变成 `Vulnerable:`。**

### 七之二 特性位不能当能力清单读

一个高频错误是拿 `/proc/cpuinfo` 的 Features 行做"能力核验"。**上游 arm64 的 cpuinfo 打印函数只遍历一张固定的能力名表 `hwcap_str[]`**，本地核对其中的名字按序为 `fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid … ssbs sb paca pacg gcs … bti mte ecv afp rpres … wfxt`（参考资料 3）。这张表按 `KERNEL_HWCAP_*` 下标取名字，打印什么完全由这张表决定，与内核内部还登记了哪些能力无关。这张表里：

- **没有 `csv2`，也没有 `csv3`。** CSV2/CSV3 是 ARM 架构的能力标识，内核用它来选缓解实现，但**不会把它打印到 Features 行**——本地核对的两处证据是同一件事的两面：`hwcap_str[]` 里没有这两项的名字，而 `ID_AA64PFR0_EL1` 的 `CSV2`、`CSV3` 字段在 `arch/arm64/kernel/cpufeature.c` 的能力登记表里被标为 **`FTR_HIDDEN`**（参考资料 27）。因此在 Android 设备上看不到 `csv2` **不构成任何判断依据**——无论正向还是反向。
- **`ssbs` 与 `sb` 会被打印**，所以脚本 1 的 arm64 能力清单只查 `ssbs sb wfxt afp gcs` 五项（x86 侧查 `ssbd ibrs ibpb stibp md_clear flush_l1d`），并把"读不到"与"读到但没有"分码：`cpu_cap_present` / `cpu_cap_absent` / `cpu_cap_unread`。其中 `cpu_ssbs_absent`、`cpu_sb_absent` 定为 `L2-WARN`：**缺 `ssbs`/`sb` 意味着内核少了两条便宜的抑制手段可用**，这与"是否已缓解"不是同一件事，但对派工有用（该设备只能依赖其他缓解路径）。
- 反过来的错误同样常见：**Features 行里出现 `mte`/`bti` 不等于任何缓解已启用**，它们只说明硬件实现了该指令。本文所有能力位结论只用于"内核可用的手段集合"，不用于"风险状态"。

### 七之三 微码锚点只存在于 x86

| 项 | x86 | arm64 |
| --- | --- | --- |
| 硅侧版本读数 | `/proc/cpuinfo` 有 `microcode` 行（脚本 1 记 `cpu_microcode_line` / `POL,microcode`） | **没有对应字段**（脚本 1 固定记 `arm_microcode_face_absent`，`L0-LOW`） |
| 判定依据 | 需要厂商的"内核最低要求 vs 微码版本"表才能定新旧 | 只能向厂商索取；设备侧无可自证字段 |
| `old_microcode` 文件 | 存在且语义明确（参考资料 4） | 由同一名单注册，架构无实现时走 `Not affected` 回退（六之二） |

这张表是三件事的来源：**为什么 arm64 设备的侧信道整改特别难验收**、**为什么第十三章把"厂商证据"设成一条独立的派工项**，以及**为什么脚本 2 对 x86 的 `microcode_changed` 只报方向不定新旧**（`L2-WARN`，措辞明确写"本脚本不猜测新旧"）。

### 七之四 由此得出的比对纪律

- **等级只在同架构、同内核代际、同采集面内比较。** 脚本 2 一进入就先比 `STATE,arch`，两侧架构不同直接记 `arch_changed`（`L2-WARN`）并在结论里声明"逐项差异不能按同族升级解读"。
- **缺席项的解释依赖架构。** 六之一说"文档不承诺文件集合"，七之三说"arm64 无微码面"。所以 `vuln_meltdown_absent_x86`、`vuln_mds_absent_x86` 只在 x86 分支输出，`arm_microcode_face_absent` 只在 arm64 分支输出。
- **判不出架构时，全部缺席项解释作废。** 脚本 1 在 `arch` 判不出时只输出一条 `vuln_arch_unknown` 并把这一节留作缺口，绝不输出任何"该平台不导出该项"的句子。

## 八、内核启动参数层：`mitigations=` 的聚合语义、逐项开关与"参数存在但无效"

H3 是六层里唯一"改一行 cmdline 就能整体翻转"的层，也是本文两个脚本对 `/proc/cmdline` 逐串判定的原因。

### 八之一 聚合档位的官方措辞

`mitigations=` 条目原文：**"Control optional mitigations for CPU vulnerabilities. This is a set of curated, arch-independent options, each of which is an aggregation of existing arch-specific options."**，同一条目还有一个常被忽略的前提：**"Note, \"mitigations\" is supported if and only if the kernel was built with CPU_MITIGATIONS=y."**（参考资料 15）。

由此得三条：

1. **`mitigations=off` 不是"一个开关"，而是一串开关的宏。** 它展开后包含（同条目逐项列出）**`nopti`、`nospectre_v1`、`nospectre_v2`、`nospectre_bhb [ARM64]`、`ssbd=force-off [ARM64]`、`spec_store_bypass_disable=off`、`gather_data_sampling=off [X86]`** 等（参考资料 15）。所以脚本 1 看到 `mitigations=off` 时直接记 `cmdline_mitigations_off`（`L3-HIGH`），**不再逐项等待其它串**——因为在语义上它已经把那些串设为关闭了。
2. **`mitigations=auto,nosmt` 是另一个档，不是"off 的变体"。** 它保留自动缓解、只额外关 SMT。脚本 1 对它单独记 `cmdline_mitigations_nosmt`（`L1-INFO`），因为第五章里 MDS/TAA/MMIO 三族的 `; SMT vulnerable` 后缀正是被这一档影响的。
3. **`CPU_MITIGATIONS=n` 编译出来的内核里，cmdline 上写什么都无效。** 这条把 H3 与 H2 绑死：**cmdline 只证明"意图"，不证明"生效"。** 因此脚本 1 对 cmdline 的判定全部输出 `CMD,<token>,present|absent`，并**从 cmdline 单独推不出任何缓解状态结论**；缓解状态只从 `VULN` 行读。第廿三章的漂移比对里，cmdline 变化被当作**"需要重读并留证的事件"**，而不是"风险已变"本身。

### 八之二 逐项开关：合法写法只有这些

脚本 1 的 `CMDTOKS` 串**只收录在 `kernel-parameters.txt` 里确实存在的写法**（参考资料 15）：

```
mitigations=  nopti  kpti=0  nokaslr  nospectre_v1  nospectre_v2
nospectre_bhb  ssbd=  spec_store_bypass_disable=  spectre_v2_user=
spectre_bhi=  init_on_free=1
```

配套的两条：

- **`nospectre_bhb` 的原文自带后果说明：**"nospectre_bhb [ARM64,EARLY] Disable all mitigations for Spectre-BHB (branch history injection) vulnerability. System may allow data leaks with this option."（参考资料 15）。**"System may allow data leaks" 是内核文档自己写下的后果**，所以这一项在 arm64 上一出现就是 `L3-HIGH` 级别的处置对象（脚本 1 通过通用 `cmdline_disable_token` 出 `L2-WARN`，第廿一章解释为何不升到 L3：单项关闭的实害仍取决于该核是否受 BHB 影响，而这一点在设备侧读不出）。
- **`csv2_disable` 这一类"看起来像参数"的字符串不存在。** 上一轮排查里，社区流传的 `csv2_disable`、`no_mitigations`、`spectre_bhb=`、`mitigations=auto,nosecbug` 都**没有**上游条目。脚本 1 不会因为它们出现在 cmdline 上就给任何结论——但**也不会忽略它们**：任何不在 `CMDTOKS` 里的串都只是 `cmdline_bytes` 的一部分，不影响判定。这一条防的是"把厂商私有 boot 参数当上游开关解读"。

### 八之三 最阴的一条：参数会被静默忽略

本地核对的上游解析函数里，`spectre_v2=` 的取值分支覆盖 `off|on|retpoline|retpoline,amd|retpoline,lfence|retpoline,generic|eibrs|eibrs,lfence|eibrs,retpoline|auto|ibrs`，**落在末尾 `else` 时只做一件事：`pr_err("Ignoring unknown spectre_v2 option (%s).", str);` 然后返回 0**（参考资料 23）。同时 `nospectre_v2` 由 `early_param` 注册并直接把 `spectre_v2_cmd` 设为 `SPECTRE_V2_CMD_NONE`，且解析函数开头有一条 **"if (nospectre_v2)"** 紧跟 **"return 0;"**（源码里这两句分两行写，这里为排版并置）——**`nospectre_v2` 在时，写在它后面的 `spectre_v2=on` 完全不生效。**

这两段代码合起来是一条重要的运维结论：**"cmdline 上有某个开关"与"该开关被内核采纳"是两件事，而且后者在被忽略时不会让启动失败。** 本文因此在两处设防：

1. 脚本 1 对 `mitigations=` **额外输出第 4 字段（cmdline 上的字面取值）**，脚本 2 用它做档位漂移比对，使 `mitigations=auto` → `mitigations=off` 这类"同为 present、语义相反"的变化不会漏报（`cmdline_mitigations_value_changed`，`L2-WARN`）。
2. 第廿二章的样例输出里，凡 cmdline 出现关闭串而 `VULN` 行仍读 `Mitigation:` 的组合，都被明确标注为**"两者需要一次人工对齐"**，脚本不擅自判定谁对。因为如上所述，可能是内核编译期没开 `CPU_MITIGATIONS`、可能是该串被静默忽略、也可能该架构的这条通道本就不受影响——**三种成因在设备侧不可区分，只能记成待核查事件。**

### 八之四 `kpti=0`、`nopti`、`nokaslr`、`init_on_free=1` 各自的口径

- **`nopti [X86,PPC]` 与 `kpti=0`** 是同一件事的两种写法（参考资料 15）。它们直接影响第五章里 `meltdown` 文件能否读成 `Mitigation: PTI`。**Android 上 `kpti=0` 更常见于厂商性能调优分支，属于必须逐串点名的一类。**
- **`nokaslr`** 关的是地址随机化。它不改变 `vulnerabilities` 任何读数，因此**不属于缓解档位**，但属于侧信道治理：它让"固定偏移"变得可行。本文把它单列为 `cmdline_disable_token` 里的一条，并在第十四章解释为何它与 `dmesg_restrict`/`kptr_restrict` 要一起看——**KASLR 的价值依赖地址不泄露，而后两者是地址泄露的策略闸门。**
- **`init_on_free=1`** 是上游提供的**内存清零**选项，脚本 1 给 `L1-INFO`（`cmdline_init_on_free`）：**它降低的是"残留数据被后继读者取到"这一族的前提**，与第五章的缓冲区采样类同源，但它不是任何一项的官方缓解，因此绝不计入"已缓解"统计。

### 八之五 为什么 cmdline 必须与 `security_patch` 一起读

厂商在一次 OTA 里同时改 SPL 和 cmdline 是完全可能的，也是最常见的"看起来是升级、实际是放松"的组合。脚本 2 因此把 `spl_moved_forward`（`L1-INFO`）与 `cmdline_disable_token_added`（`L3-HIGH`）放在同一份差分里输出，并在结论层规定：**只要同一次差分里出现后者的痕迹，前者的"已前进"就不得写成"整改到位"。** 这条纪律属于第十二章 KPI 分母的定义，具体输出形态在第廿四章的样例里可见。
## 九、M1–M10 缺陷类型学：按"需要什么观测能力"分，而不是按公告名分

第五章按文件名整理了家族。这一章换一个维度：**同一颗芯片上，十种泄露各自要求观察者具备什么能力、由哪一层收敛、设备侧留下什么证据。** 这个维度才可直接用于派工，因为它回答的是"我要治的是哪一段"，而公告名只回答"这件事叫什么"。

| 型 | 名称 | 观测方需要什么能力 | 收敛主要在哪层 | 设备侧证据（信号码） | 最常见误判 |
| --- | --- | --- | --- | --- | --- |
| **M1** | 边界检查绕过型推测读取（Spectre v1 族） | 能在同机内安排"猜测窗口内的读取 + 事后观测" | H2（`__user` 指针清理、usercopy 屏障） | `VULN,spectre_v1,…`；arm64 为常量句（第七章）→ `vuln_mitigated` | 在 arm64 上把它当"已确认缓解"，实际那句只证明内核用了这条实现 |
| **M2** | 间接分支预测器注入型（Spectre v2 / Retbleed / SRSO / ITS / TSA） | 能训练预测器；预测器在同机上下文间共享 | H1 微码 + H2（retpoline/IBRS/eBPF 路径）+ H3 | `VULN,spectre_v2,…` 及其片段；`; SMT vulnerable` → `vuln_note_smt` | 把 `Mitigation: Vulnerable, KVM: Not affected` 读成"已缓解"或"未缓解"，两种都错 |
| **M3** | 内核地址映射泄露型（Meltdown / L1TF / itlb / VMSCAPE） | 能触发内核态投机访问并观测其缓存后果 | H2（PTI）+ H3（`nopti`/`kpti=0`）+ H1 | `meltdown`/`l1tf`/`itlb_multihit`/`vmscape` 四行；`Processor vulnerable` → `vuln_state_unknown_prefix` | 读到 `Mitigation: PTI` 就认为 cmdline 不可能有 `kpti=0`——两者共存时必有一次人工对齐（八之三） |
| **M4** | 缓冲区数据采样型（MDS / TAA / SRBDS / GDS / RFDS / MMIO stale） | 与泄漏方共享同一物理核的内部缓冲（同 SMT 线程尤甚） | **H1 为主**，内核只能冲刷或禁用特性 | `mds`/`tsx_async_abort`/`srbds`/`gather_data_sampling`/`reg_file_data_sampling`/`mmio_stale_data`；`no microcode` → `vuln_note_microcode`；hypervisor → `vuln_state_unknown_prefix` | 看到 `Mitigation: Clear CPU buffers` 就结束；这句与 `; SMT vulnerable` 是同一行的两半 |
| **M5** | 存储转发/抢占型（SSB / SSBS / SB） | 在同核上排布存在依赖的访存序列 | H2 + H3（`spec_store_bypass_disable=`）+ 每进程 prctl | `VULN,spec_store_bypass,…`；能力位 `CAP,ssbs`/`CAP,sb` → `cpu_ssbs_absent`/`cpu_sb_absent` | 把"via prctl"读成"全部进程已关闭"；prctl 是**交给进程**，不是替进程做完 |
| **M6** | 分支历史注入型（Spectre-BHB / STLB，arm64 专属） | 能填满共享的分支历史缓冲 | H1（架构 CSV2 等级）+ H2（BHB 清理）+ H3（`nospectre_bhb`） | 只在 `spectre_v2` 后缀里可见 → `vuln_bhb_covered_in_v2` / `vuln_note_bhb_partial`；无独立文件 | 在清单里找不到 `spectre_bhb` 就判"该平台不关心 BHB"；出现该文件反而是不在上游 ABI 内 |
| **M7** | 地址翻译与预取探测型（缓存/TLB/预取通路） | 能占用或驱逐共享结构并测量其后果 | **没有官方缓解项**；只能靠 H4 限制谁能运行、H5 减少共享 | **无对应读数**（六之四明确写出这一面不覆盖缓存侧信道） | 因为读不到就断言"不存在该通道"；本文一律记为"证据缺失"，不记为"通过" |
| **M8** | 计时与执行路径观测型（CWE-208 族） | 能反复触发同一操作并测量时长 | **完全在实现层（H6）** | **无**任何设备侧读数覆盖；仅由第十五章的实现自查承担 | 把它并入"这台设备有没有侧信道"，用芯片结论给库实现背书 |
| **M9** | 共享资源占用/噪声型（邻居、DVFS、大小核调度） | 只需在同一台机器上运行 | H4 + H5 | `SYSCTL,*` 三行、`POL,selinux`、`POL,hidepid`、`ps` 派生的 `COUNT,zygote_processes` 与 `neighbor_app_domain_gap`、`cpufreq_governor` | 把"读到低档位 paranoid"直接判成"采样面开放"；在 Android 上策略才是闸门（第十一章） |
| **M10** | 隔离执行环境内的密钥路径型（安全元件、TEE 固件、协处理器） | 需要在该环境内部或其边界上安排观测 | 器件与固件厂商；平台侧无法代为缓解 | **无**；`Unknown: Dependent on hypervisor status` 之类只能说明"读不到" | 用"设备主 CPU 全部 Mitigation"推成"密钥路径安全"；第十七章专门处理这条边界 |

四条来自这张表的结构判断：

1. **M1–M6 有读数，M7–M10 没有。** 这不是"后四类不重要"，而是**后四类的证据不在设备侧**。本文把这条写成硬纪律：K 类运营指标里凡涉及 M7–M10 的，一律用"是否留存了实现侧/厂商侧证据"来度量，绝不用 sysfs 读数替代。
2. **同一个缺陷家族可能跨型。** `spec_store_bypass` 同时是 M5（转发机制）与 M2 的观测后果载体；MDS 同时是 M4 与 M9（SMT 邻居决定它是否成立）。所以**类型学用于派工，不用于唯一归档**——一张检查表里允许同一行出现两个型号，这正是第三章反对"一格子一问"的原因。
3. **每往下一型，"缓解到位"的含义就更弱一分。** M1–M3 的 `Mitigation:` 至少能说清内核做了什么；M4 的 `Mitigation: Clear CPU buffers` 需要微码配合（参考资料 23 里 `*UCODE_NEEDED*` 全部映射到 `Vulnerable:` 开头）；M5 的 "via prctl" 把责任推给进程；M6 只在后缀里；M7–M10 完全说不出。**因此判级要按型而不是按字符串**——这是第二十章判定模型把 M1–M3 与 M4–M6 分档的原因。
4. **M9 是唯一"无需漏洞即可成立"的一型。** CWE-1303 的官方表述把它说得很清楚：**"Contention for shared resources between different contexts opens covert channels that allow malicious programs executing in one context to recover information from another context."**（参考资料 17）。也就是说，**这一型的存在与否取决于这台设备上运行了什么、以及谁被允许看，而不取决于硅是否"有 bug"**。第十二章因此不是把邻居当背景，而是当独立派工对象。

## 十、CWE 对齐与一处必须纠正的常见错标

### 十之一 三条可用的 CWE

- **CWE-1303 Non-Transparent Sharing of Microarchitectural Resources**（参考资料 17）。这是本领域**最贴切的类目**，而且它的扩展描述本身就是本文的立场：**"Modern processors use techniques such as out-of-order execution, speculation, prefetching, data forwarding, and caching to increase performance. Details about the implementation of these techniques are hidden from the programmer's view. This is problematic when the hardware implementation of these techniques results in resources being shared across supposedly isolated contexts."**，并且对"回滚不彻底"给出了机制级解释：**"Some examples of shared micro-architectural resources that have been used to leak information between contexts are caches, branch prediction logic, and load or store buffers."** 以及 **"Even though the processor rolls back the architectural effects of the mis-predicted indirect branch, the memory accesses alter data cache state, which is not rolled back after the indirect branch is resolved."**
  最后这句值得抄进任何一份工单：**架构状态回滚了，缓存状态没回滚。** 它解释了为什么"处理器已经撤销了错误预测"不构成安全论据，也解释了第五章 M2/M4 为什么必须分开看。
- **CWE-208 Observable Discrepancy**（参考资料 12）：管 M8。定义句已在第二章引用。它的价值是**把"计时差"从"侧信道"这个大词里单独拎出来**，让密钥实现类缺陷可以进常规缺陷流程而不必挂到微码治理下。
- **CWE-200 Exposure of Sensitive Information to an Unauthorized Actor**（参考资料 16）：作为上位类使用，仅在没有更贴切类目时。**注意 CWE-1303 的存在使绝大多数微架构泄露不应该停在 CWE-200。**

### 十之二 必须纠正的一条错标

在检索本领域资料时会反复看到把 **CWE-1063** 写成"通过推测数据访问泄露敏感信息"。**这是错的。** MITRE 页面上 CWE-1063 的名称是 **"Creation of Class Instance within a Static Code Block"**，并且其映射标注为 **"Vulnerability Mapping : PROHIBITED / This CWE ID must not be used to map to real-world vulnerabilities"**（参考资料 25）——它是一个 Java/静态代码块相关的废弃类，与本主题毫无关系。

正确条目是 **CWE-1303**。这条纠正本身不重要，重要的是它暴露的产生方式：**一个不存在的编号，因为反复被复制而在文档里变得像真的。** 所以本文对每一个编号都要求"打开来源、读到标题"，并在第十五章把这条纪律落到密钥实现的自查表里：**引用一个 CWE 编号前先核对标题，引用一条缓解前先读它的定义句，引用一个 cmdline 参数前先确认它在上游参数表里存在（八之二）。**

### 十之三 本系列其它篇目的对齐

侧信道类缺陷常与另外两类同因不同治：

| 现象 | 本系列位置 | 治理口径差异 |
| --- | --- | --- |
| 密钥在解密过程中走了不同分支/不同时长 | 第十五章（M8，CWE-208） | 改实现，与微码无关 |
| 通过 `/proc` 或日志观察到内核地址与计时 | 第十一章（M9，策略层） | 改策略与 sysctl，与硅无关 |
| 由内存破坏缺陷泄露指针，再用于构造推测读取 | 内核篇与提权篇（缺陷本身） | 修缺陷；本文只承担"泄露之后是否有观测通道"这一段 |

第三行是**唯一真正需要跨篇联合**的情形：**内存安全缺陷提供信息，微架构通道提供读取手段，两者都不是对方的必要条件，但合起来才构成完整泄露。** 上游对 SPECTRE_V2_USER 的实现里就有同构的表述——内核在特定条件下把已缓解的分支改写为 `Vulnerable: eIBRS with unprivileged eBPF`（参考资料 23）：机制上"已缓解"，被一个策略选择重新打开。本文不在此展开任何组合方法，只保留这条归属结论：**侧信道风险的评审必须和缺陷评审在同一张表上，否则会分别被两个团队各自判为"不归我"。**

## 十一、平台策略层：Android 用 SELinux 而不是 sysctl 来管"谁能看"

H4 是本文唯一一层"整改不需要动内核也不需要等厂商"的层，因此也是治理收益最快的层。这一章的依据全部逐字取自本地核对过的 AOSP 文件（参考资料 5、6）。

### 十一之一 `perf_event_paranoid` 在 Android 上有三种取值，含义与 Linux 直觉相反

`system/core/rootdir/init.rc` 的相关段落原文（含上游那个拼写错误 `pofiling`，本文原样保留）：

```
# perf_event_open syscall security:
# Newer kernels have the ability to control the use of the syscall via SELinux
# hooks. init tests for this, and sets sys_init.perf_lsm_hooks to 1 if the
# kernel has the hooks. In this case, the system-wide perf_event_paranoid
# sysctl is set to -1 (unrestricted use), and the SELinux policy is used for
# controlling access. On older kernels, the paranoid value is the only means of
# controlling access. It is normally 3 (allow only root), but the shell user
# can lower it to 1 (allowing thread-scoped pofiling) via security.perf_harden.

on load-bpf-programs && property:sys.init.perf_lsm_hooks=1
    write /proc/sys/kernel/perf_event_paranoid -1
on property:security.perf_harden=0 && property:sys.init.perf_lsm_hooks=""
    write /proc/sys/kernel/perf_event_paranoid 1
on property:security.perf_harden=1 && property:sys.init.perf_lsm_hooks=""
    write /proc/sys/kernel/perf_event_paranoid 3
```

三条派生判读：

1. **`-1` 是这条链上"最安全"的取值，而不是最开放的取值。** 它出现的前提是 `sys.init.perf_lsm_hooks=1`，即内核有 LSM 钩子，访问控制整个交给了策略。所以**任何"paranoid 低=危险"的直接映射在 Android 上都是错的**。脚本 1 因此把 `SYSCTL,kernel/perf_event_paranoid` 与 `POL,selinux` 绑在同一份快照里，并且 `sysctl_paranoid_low`（`L2-WARN`）的措辞里明确限定"在该读取域内"，不写"采样面开放"。
2. **`1` 的出现是有条件的：`security.perf_harden=0` 且没有 LSM 钩子。** 这是一个**可被 shell 侧属性触发的放松路径**（原注释就写着 "the shell user can lower it to 1"）。因此它是脚本 2 差分里少数真正值得关注的漂移之一：它意味着这台设备上"唯一的闸门"是那个整数，而它被改了。
3. **`3` 只出现在老内核上。** 于是 **`paranoid=3` 不能读成"这台设备更严"**——按第三章 H4 的定义，它读成"这台设备的内核没有 LSM 钩子，平台退回到只有 sysctl 的状态"。脚本 1 用 `sysctl_paranoid_high`（`L1-INFO`）记录，不升级也不降级，并在第十九章说明它为什么不构成结论。

顺带一条同样重要的：**写 `-1` 的触发器是 `on load-bpf-programs`**。也就是说这一条与 BPF 加载事件相关；而第五章里 x86 与 arm64 **都有把非特权 eBPF 判回 `Vulnerable` 的读数**。这两段代码在同一主题上相遇：**平台的 perf 放松与内核的 eBPF 收紧判断，都挂在这个属性族上。** 本文只报告这一处关联的存在，并把它列为第十七章的一条待核查项；**不做任何利用性推论**。

### 十一之二 策略侧的真实闸门：一条宏与一条域集合

`private/domain.te`（参考资料 6）：

```
# Allow profiling using perf_event_open by traced_perf.
can_profile_perf({
  dumpable_domain
  -app_zygote
  -hal_configstore_server
  -webview_zygote
  -zygote
})
```

三点读法：

- **能力被授予给 `dumpable_domain` 这个集合，不是被授予给"root"或"系统"。** 治理动作因此有明确落点：把某个域移出可调试集合，就移出采样能力，不需要动内核。
- **四个 `-` 排除项里有两个是 Zygote。** 平台明确把母进程排除在可被 perf 剖析之外——这正好是第十二章"Zygote 共享"的治理侧对应物：**Android 承认这些进程共享状态，并在策略上禁止直接观测它们。**
- 同文件里还有一条容易忽略的：**`# profiling needs to read /proc/sys/kernel/perf_event_max_sample_rate` / `allow domain proc_perf:file r_file_perms;`**（参考资料 6）。**这是给 `domain`（即所有域）开的读权限。** 所以 `perf_event_max_sample_rate` 这个节点在任何 Android 设备上都是全局可读的，本文把它当作**已知暴露面**记录，而不是当配置项讨论。

### 十一之三 跨 UID 可见性：策略用 neverallow 复现了 `hidepid=2` 的效果

`private/app.te`（参考资料 6）：

```
# Read or write access to /proc/pid entries for any non-app domain.
# A different form of hidepid=2 like protections
neverallow appdomain { domain -appdomain }:file no_w_file_perms;
neverallow { appdomain -shell } { domain -appdomain }:file no_rw_file_perms;
```

**"A different form of hidepid=2 like protections"** 是策略文件自己的措辞。它的意义在于：**Android 上不依赖 `/proc` 的 `hidepid=` 挂载选项来实现跨 UID 隐藏。** 于是：

- 脚本 1 检查 `mount` 输出里的 `hidepid=` 是**有信息价值的，但不是决定性的**：读到 `proc_hidepid_off`（`L1-INFO`）只说明挂载选项没设，不说明应用能读别人的 `/proc/pid`——后者由上面两条 neverallow 拦。这一码因此永不升级。
- 反过来，`POL,selinux` 读到 `Permissive` 或 `Disabled` 才是实质变化：`neverallow` 在 permissive 域里不产生拦截。这正是 `selinux_permissive` / `selinux_unexpected` 两条定 `L2-WARN` 的理由，也是**为什么脚本 1 要求 `getenforce` 与三个 sysctl 同批采集**。
- `untrusted_app_all.te` 还有一条与观测面直接相关：**"# These have been disallowed since Android O. / For P, we assume that apps are safely handling the denial." 后跟 `dontaudit untrusted_app_all { proc_stat proc_uptime proc_vmstat proc_zoneinfo }:file read;`**（参考资料 6）。**这四个节点是低精度系统状态观测的主要来源**，平台从 Android O 起就不再允许普通应用读，并从 P 起连拒绝日志都抑制。**"抑制日志"这件事本身要在治理上留意：它不是允许，但它让"应用在尝试读"变得不可见。** 因此本文的运行时取证侧不能依赖 avc 日志的完整性（该口径引向沙箱篇）。

### 十一之四 `kptr_restrict` 与 `dmesg_restrict`：地址外泄的策略闸门

`init.rc` 只用属性驱动 `kptr_restrict`，原文与注释如下（参考资料 5）：

```
# This property can be set only on userdebug/eng. See neverallow rule in
# /system/sepolicy/private/property.te .
on property:security.lower_kptr_restrict=1
    write /proc/sys/kernel/kptr_restrict 0

on property:security.lower_kptr_restrict=0
    write /proc/sys/kernel/kptr_restrict 2
```

`dmesg_restrict` 不在这一段落里，它由厂商 defconfig 或 init 片段设定，**因此它在两台同名设备上的差异更可能来自厂商分支而不是平台**——这正是本文把它单列一条 `SYSCTL` 而不并入属性链的原因。三条与本文相关的判读：

- **`kptr_restrict` 管的是指针是否打印成 `0` 还是真地址。** 它与八之四的 `nokaslr` 是**一对**：KASLR 的价值依赖地址不外泄，而这里是外泄闸门。两者同时放松时，本文把它们当作**同一格待核查项**输出，而不是两条独立提示。
- **`dmesg_restrict` 管的是能否读内核日志。** 内核日志里有缓解选择的结论行——例如 `pr_info("Spectre v2 / PBRSB-eIBRS: Retire a single CALL on VMEXIT\n")` 与 `pr_err("Ignoring unknown spectre_v2 option (%s).", str);`（参考资料 23）。**`dmesg_restrict=0` 意味着这些"内核到底采纳了哪个档位"的证据可以从设备侧读到，这正是八之三那条"参数被静默忽略"的唯一可核验渠道。** 所以本文把 `sysctl_dmesg_open` 定 `L2-WARN` 的同时，在运维口径里把它当**双刃**：收紧它会让诊断变难，这是第十七章 KPI 表里要显式记录的取舍。
- 属性侧有一条限制值得注意，它就是上面那段注释的第一句：**"This property can be set only on userdebug/eng. See neverallow rule in /system/sepolicy/private/property.te ."**（参考资料 5）。这与脚本 1 的 `build_type_nonuser`（`L2-WARN`）构成同一条判断链：**在 `eng`/`userdebug` 上读到的宽松值，其含义与在 `user` 上读到的不同——前者是平台允许的调试路径，后者意味着有人改了什么。**

### 十一之五 这一层为什么值得先做

六层里 H4 是唯一满足全部四条的：**改它不需要重编译（对比 H2）、不需要厂商发固件（对比 H1）、不需要重签 boot image（对比 H3）、且改动会立刻反映在脚本 1 的 `POL/SYSCTL` 段（可验证）**。第三章说"可证明性与危险度不成比例"，这里给出一个例外：**H4 是既最容易做、又最容易在设备侧证明做到的一层。** 本文所有 KPI 中，只有 H4 类的指标可以直接拿快照差分当验收证据；其余层的验收必须搭配一条厂商或变更单依据（第二十六章）。

## 十二、同机邻居与 Android 特有的放大器

M9 型风险的实体不是"攻击者"，而是"这台机器上同时运行着什么、以及谁能看见谁"。Android 的三个结构特征让这一层的密度明显高于桌面 Linux。

### 十二之一 Zygote：一台设备上的绝大多数进程有同一个母进程

Android 的应用进程由 Zygote fork 而来，**这意味着它们共享同一份已预加载的类与应用资源镜像**。对本文的意义只有一条：**共享状态是默认值，不是异常值。** CWE-1303 那句"resources being shared across supposedly isolated contexts"（参考资料 17）在这里几乎是逐字成立的——被隔离的是 UID 与 SELinux 域，共享的是硅侧资源。

平台侧的可核验对应物是两点（第十一章）：`can_profile_perf` 显式排除 `zygote` 与 `app_zygote`；`app.te` 的 neverallow 让一个应用无法读另一个应用的 `/proc/pid` 条目。**所以"Zygote 共享"不是"没人管"，而是"管的方式不是靠隔离缓存，而是靠限制谁能观测"。** 脚本 1 因此把 Zygote 只当作**邻居计数的锚点**：`COUNT,zygote_processes`、`zygote_shared`（`L1-INFO`）与 `zygote_absent`（`L0-LOW`，采不到即缺口），绝不从"有 Zygote"推出任何风险结论。

### 十二之二 ART、JIT 与可执行代码缓存

应用侧的 JIT 代码在设备上是可执行且跨进程复用形态的；对本文的意义不在 JIT 本身，而在**运行时决定了很多访问模式由平台代码而非应用代码产生**。这带来一条治理要求：**密钥相关代码不应该运行在与大量共享代码同一进程内的地方。** 这是一条纯归属结论，落到第十五章与第十七章；本文不讨论任何关于代码缓存的具体机制或方法。

### 十二之三 DVFS 与调度：性能治理会改变可观测性

脚本 1 采集 `/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`，并在读到 `performance` 时给出提示。这条读数**不属于任何缓解**，它属于 M9：

- **固定高频（`performance`）会压平时长差异**，而动态调频会把频率变化本身变成可观测信号。两者都不是"更安全"，只是改变了哪一段信息更明显。
- **`walt`、`schedutil`、`interactive`、`conservative` 这些厂商调频器同时改变"大小核如何被选中"**。第五章提到 `vulnerabilities` 只反映 boot CPU（六之四），而调频/调度决定一个任务实际落在哪个核上——**这两件事合起来意味着：在多簇 SoC 上，"这台设备缓解状态如何"这个问题没有单一答案，只有"这个任务落在哪个核上时缓解状态如何"。**
- 因此 `cpufreq_governor` 的信号码定为 `L1-INFO`（`cpufreq_governor`）与 `L0-LOW`（`cpufreq_unread`）：**永远不升级，永远要求读者把它当作"证据适用范围"的限定条件而不是风险项。**

### 十二之四 邻居清点能看到什么、看不到什么

脚本 1 用 `ps` 输出做一件很小的事：确认这台设备上有没有应用域进程在跑，以及采集是否被截断（`neighbor_app_domain_gap`，`L0-LOW`）。**明确不做的事**：不推断任何进程的身份、不推断它们之间是否可互相观测、不给出任何"哪些进程构成邻居对"的结论。理由是第三章 H5 那一行的定义：**设备侧没有可自证的缓存共享度量**，任何此类结论都需要厂商的微架构文档，而不是设备读数。

这带来一条治理口径：**Android 上"邻居集合"的治理单位是"可安装应用的准入"，不是"进程隔离参数"。** 第一章引用官方位置口径 **"Local attacks require the attacker to have prior access to the victim."**（参考资料 9）与厂商公告里的 "an authenticated user … via a side channel with **local access**"（参考资料 7、8）之后，落地的派工对象就是**准入与应用签名侧**（该口径分别属于沙箱篇与应用弱点篇）。本文在这里只补一条本篇特有的结论：**准入侧收紧对 M7–M9 有效，对 M1–M4 完全无效**，因为后者的成立不需要一个"新的邻居"，只需要**任何一个**同核邻居存在。
## 十三、微码与固件到达性：唯一一条"设备侧几乎无法自证"的层

H1 是本领域责任最重、可证明性最低的一层。本章把"到达性"拆开：能读到什么、读到之后能说明什么、说明不了的时候向谁要证据。

### 十三之一 x86 上有一个锚点，但它只到"版本号"为止

脚本 1 在 x86 设备上读 `/proc/cpuinfo` 的 `microcode` 行（参考资料 23），输出 `POL,microcode,<值>` 与 `cpu_microcode_line`/`cpu_microcode_unread`。**它能自证的只有一件事：这台机器当前报告的这个数值。** 这个数值本身不含任何判断信息（该行由 x86 微码子系统打印，本文未取得其打印代码的逐字来源，因此只按公开格式使用，见第三十章边界 11）：

- **"这个版本够不够"需要厂商的版本对照表**，不是设备能算出来的。上游对 `old_microcode` 的定义句写的正是这件事的另一半：**"Systems that had microcode older than this at boot will say \"Vulnerable\"."**（参考资料 4）——**"this" 指的是当前这份内核要求的最低线**，它随内核版本变。所以同一份微码在两个内核下可以得到不同的 `old_microcode` 读数。
- 因此脚本 2 对 `POL,microcode` 变化的处理是**只报方向不定新旧**：`microcode_changed` 定为 `L2-WARN`，措辞明确写"这是设备侧唯一能读到的硅侧缓解证据，方向要按厂商微码版本表判定，本脚本不猜测新旧"。
- 还有一条更容易被忽略的：**运行时更新不改变读数**——**"Runtime microcode updates do not change the status of this vulnerability."**（参考资料 4）。所以"我刚才 reload 了微码，`vulnerabilities` 没变"是完全预期的结果，**不能**读成"更新失败"。要证明一次固件更新真的到了设备，唯一可自证的路径是重启后 `POL,microcode` 值本身变了（脚本 2 的 `microcode_changed`），且仍需厂商侧一条变更记录来解释它。

### 十三之二 arm64 上没有这个锚点，且不是"没采到"

Android 绝大多数设备是 arm64。脚本 1 在 arm64 分支固定输出一条 `arm_microcode_face_absent`（`L0-LOW`）：**"arm64 侧没有 x86 那样的微码读数面：CPU 侧修复随厂商固件到达，设备上没有可自证的字段，只能向厂商索取证据。"**

三条纪律：

1. **不要因为读不到就把这一层判成"未整改"或"已整改"。** 它是**不可判定**，落在 L0-LOW 这一档的定义正是"缺口"（第二十章）。
2. **arm64 的硅侧修复到达性只能由厂商可核验信号承担**：更新说明里是否点到 SoC 型号与缺陷族、`ro.build.version.security_patch` 只能说明 Android 侧补丁级别（第九章 SPL 语义引向补丁传导链篇），以及是否有厂商自己导出的非上游读数文件（这类文件出现时脚本 1 记 `vuln_name_outside_abi` 并排除出统计）。
3. **`/proc/cpuinfo` 上的 arm64 能力行不能替代微码锚点**（七之二：`csv2`/`csv3` 根本不被打印）。这三条合起来是本文对"arm64 侧信道整改特别难验收"这一印象的正式表述。

### 十三之三 `no microcode`：内核自己在读数里承认这一层缺失

第五章词表里，**所有 `*UCODE_NEEDED*` 枚举都被映射到以 `Vulnerable` 开头的串**（参考资料 23）：`Vulnerable: No microcode`（SRBDS/GDS/TSA/SRSO/RFDS）、`Vulnerable: Clear CPU buffers attempted, no microcode`（MDS/TAA/MMIO）、`Vulnerable: Safe RET, no microcode`（SRSO）。另有 `Mitigation: AVX disabled, no microcode`（GDS：没有微码时退化为禁用 AVX 这种功能代价明显的缓解）。

这给运维一条**极其便宜但常被漏掉的判据**：只要 `VULN` 行的文字里出现 `no microcode`／`Microcode not updated` 这类片段，**格子的责任人立刻从内核侧改到固件侧**，而内核侧无事可做。脚本 1 专门有 `vuln_note_microcode`（`L2-WARN`）承接这一片段，并且它**不与 `vuln_vulnerable` 互斥**——同一行会同时出两条，一条给风险等级、一条给派工方向。这是本文两个信息（等级、归属）分离的典型实现。

### 十三之四 一颗"SoC"不是一个 CPU

三个已经在前文埋好的事实，合起来构成 Android 上最难处理的一条：

1. `vulnerabilities` 的读数由 `boot_cpu_has_bug()` / `boot_cpu_has()` 决定，且注册在 `/sys/devices/system/cpu/` 根上（六之四、参考资料 23、24）；
2. 移动 SoC 普遍是多簇（大小核、甚至三簇），不同簇可以是**不同的微架构**；
3. 第十二章说明调度器决定任务落在哪一簇。

于是："这台设备的 `spectre_v2` 是 `Mitigation: …`" 这句话**在这台机器上是不可扩展的**——它说的是 boot CPU。**本文的处置办法是把它降级为"采集范围声明"：脚本 1 的 `META,cpuinfo_bytes` 与 `COUNT,vuln_files_total` 只说明采集量，不说明核数与簇拓扑；这份快照的每一条 `VULN` 读数都按"boot CPU 结论"读取。** 需要按核判定的场景（例如给密钥运算绑定专用核）只能回到厂商文档；这一条进入第三十章边界 6。

### 十三之五 虚拟化形态会把这一层整格变成"不可知"

第五章已列出上游会输出的三句虚拟化工术语：**`Unknown: running under hypervisor`**（old_microcode）、**`Unknown: Dependent on hypervisor status`**（SRBDS/GDS）、**`; SMT Host state unknown`**（MDS/TAA/MMIO）。共同含义：**客户机内的观测者无法判断宿主的硅侧状态。**

Android 上出现这一族读数的典型场景有两类：把某个子系统跑在客户机里的实现，以及**把 Android 跑在虚拟化环境中的形态（云端手机、容器化测试农场）。** 对后者，本文给一条明确治理口径：**云测/农场设备上的侧信道快照不能与真机快照混在同一张达标表里**，因为它们的 M4/M3 段在定义上不可判定；脚本 2 遇到这类读数时输出的 `vuln_state_unknown_prefix` 组合应当作**环境标记**使用，而不是当作设备状态。

## 十四、设备侧采集面清单：八个命令覆盖什么、遗漏什么

本章是脚本 1 的**接口契约**，也是本文对"证据从哪来"的正式回答。**只读、无副作用、不访问网络、不含任何探针。**

### 十四之一 采集面与它支撑的判据

| 采集命令 | 取到什么 | 支撑哪几章的判据 | 采不到时的信号码 |
| --- | --- | --- | --- |
| `adb get-state` | 设备是否可达 | 全部（前置校验） | 直接终止，不出快照 |
| `getprop`（型号、厂商、SDK、SPL、build type、board、hardware、abilist、bootloader、`ro.kernel.qemu`） | 身份、构建形态、架构线索、补丁时间锚点 | 第三、十二、十三章；脚本 2 的同机判定 | `identity_missing`、`spl_absent`、`build_type_unread`、`bootloader_absent` |
| `uname` | 内核版本与 `aarch64`/`armv8`/`x86_64` 串 | 第七章（词表选择）、六之一（名单随版本增长） | `kernel_release_unread`；版本过旧另有 `kernel_pre_vulnfs` |
| `cat /proc/cmdline` | H3 层的全部档位串 | 第八章 | `cmdline_unread` |
| `ls` + `cat /sys/devices/system/cpu/vulnerabilities/*` | H1/H2/H3 的结论面（唯一权威面） | 第五、六、七章 | `vuln_dir_unread`、`vuln_file_empty` |
| `cat /proc/cpuinfo` | 厂商、`flags`/`Features` 能力位、`microcode` | 第七章、第十三章 | `cpuinfo_unread`、`cpu_cap_unread`、`cpu_microcode_unread` |
| `getenforce` | SELinux 运行模式 | 第十一章 | `selinux_unread` |
| `mount` | `/proc` 挂载选项（`hidepid=`） | 第十一章之三 | `proc_mount_unread` |
| `ps` | 邻居是否存在、采集是否被截断 | 第十二章之四 | `ps_unread`、`zygote_absent`、`neighbor_app_domain_gap` |
| `cat /proc/sys/kernel/{perf_event_paranoid,kptr_restrict,dmesg_restrict}` | H4 的三个闸门 | 第十一章 | `sysctl_paranoid_unread`、`sysctl_unread` |
| `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor` | 调频形态（M9 限定条件） | 第十二章之三 | `cpufreq_unread` |

**这张表里没有任何一项需要 root，也没有任何一项会改变设备状态。** 全部是已文档化的公开节点；对读不到的节点，脚本一律显式写 `unread`/`none` 而**不是省略字段**——这是"未采到=缺口"这条纪律在数据格式层的实现。

### 十四之二 这套采集面系统性地覆盖不到什么

必须写清楚，否则读者会把"没有告警"读成"没有问题"：

- **不含任何缓存、计时、TLB、预取通道的测量。**（六之四）本文不提供方法，也不声称这类测量安全无副作用；**M7 因此永远没有设备侧结论。**
- **不含进程级能力检查。** 脚本不读任何 `/proc/<pid>/` 条目——一方面 `app.te` 的 neverallow 让这类读取本身在生产设备上是被禁的（第十一章之三），另一方面**用某个进程的视角采集会引入"谁在观测"的混淆**。脚本固定从 shell 视角采集，并在报告头部声明这一点：**这份快照证明的是"该读取域内可读"，不证明"应用域内可读"。**
- **不含内核日志解析。** `dmesg_restrict=0` 时理论上可以从日志读到内核选择的缓解实现（八之三、十一之四），但**读日志需要更高权限且内容随厂商改动**，本文因此不把它纳入契约。
- **不含 CPU 拓扑与簇信息。** 十三之四的"boot CPU 结论不代表每个核"在本采集面上**无法被量化**；这是清单里的已知缺陷，不是可以靠加一条命令补救的缺口。
- **不含微码版本对照。** 十三之一说明那需要厂商表格。

### 十四之三 与相邻篇目共享但不重复的采集

`/proc` 可读性、seccomp 状态、域标识等属于[应用沙箱与进程隔离](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)的口径；SPL 与固件到达性属于[补丁传导链](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)的口径。本文只取其中**与观测通道直接相关**的字段，两篇的完整采集清单不在此重复。

## 十五、H6：密钥实现层的归属与自查方法

M8 型缺陷是这一领域里唯一"每台设备都可能自带、且完全不在读数面上"的一类。本章给方法，不给探测手段。

### 十五之一 为什么它不属于芯片结论

CWE-208 的定义句（参考资料 12）里没有一个字涉及微架构：**它说的是"两个操作耗时不同且可被观察到"。** 于是：

- **一台 `vulnerabilities` 全部读 `Mitigation:` 的设备，仍然可能因为一个把密钥位编进分支的实现而逐位泄露；反过来也一样。**（第二章第三条）
- **它的整改在设备上不可自证**，只能由代码与测试证明。因此它必须走的是**代码评审与库版本治理**这条流程，不是固件/内核这条流程。把两者放进同一张"侧信道整改表"，结果一定是实现类缺陷被"等厂商"这条路径永久挂起。

### 十五之二 自查清单（面向实现团队，不含任何测量方法）

| 检查项 | 判据 | 证据形态 |
| --- | --- | --- |
| 密钥比较是否走短路分支 | 是否存在"第 i 位不同就提前返回"的形态 | 代码评审记录；不允许以"通过测试"替代 |
| 是否使用了查表型分组实现 | 表格访问模式随密钥变化 | 库版本与实现说明，或厂商/上游对该实现的安全声明 |
| 密码运算是否运行在共享进程内 | 与第十二章的 Zygote/JIT 共享面对照 | 进程与线程模型说明 |
| 是否有跨语言边界的时长回传 | 例如把校验结果与耗时一起返回给调用方 | 接口契约评审 |
| 依赖库是否有侧信道类修复记录 | 上游 changelog/安全公告中的相关条目 | 版本对照表 |
| 引用的 CWE 编号是否核对过标题 | 见十之二那条纪律 | 评审 checklist 一行即可 |

**这张表里所有证据都是"实现侧"的。设备侧一条都不需要——这不是疏忽，而是本章的结论。**

### 十五之三 与 KeyMint / StrongBox 的关系

密钥的存储与使用边界属于[硬件信任根与密钥栈机制](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)。本文只在两处交叉：

1. **密钥是否"离开"安全元件决定了 M8 型的适用范围。** 全链路在安全元件内完成的运算，其计时/访存观测面在器件侧，不在主 Android 侧；本文第十七章按"口径边界"处理这一条。
2. **Android 16 的 Profiling 模块给出了一条平台侧收敛表述：** **"Results are redacted to contain information relating to the requesting process only and delivered to the app's storage directory for further processing by the app."**（参考资料 22）。**"只包含请求进程自身的信息"是平台在应用可请求的观测结果上做的收缩**，这与第十一章的 `dontaudit`、`can_profile_perf` 是同一条治理线的三处落点。但它**不覆盖内核/策略层的实际能力**，也不构成对任何 M 型的缓解。

## 十六、归属矩阵：层 × 责任 × 更新通道 × 设备侧证据

把第三、五、六、十一、十三、十四章的内容压成一张可派工的表。**"更新通道"一列决定这件事要等多久，"设备侧证据"一列决定这件事能不能验收——两者都必须写死，否则工单无法关闭。**

| 型（章九） | 主要层（章三） | 责任人 | 更新通道 | 设备侧证据 | 可验收？ |
| --- | --- | --- | --- | --- | --- |
| M1 | H2 + H1 | BSP 内核维护方 | 内核镜像（OTA） | `VULN,spectre_v1`（arm64 无区分度） | **部分**：能验收"读数变了"，不能验收"缓解充分" |
| M2 | H1 + H2 + H3 | 芯片厂商 + 内核 + 整机 | 固件包 + 内核 + cmdline | `VULN,spectre_v2` 全部片段 | **部分**：片段级可验收，跨核不可 |
| M3 | H2 + H3 + H1 | 内核 + 整机 | 同上 | `meltdown`/`l1tf`/`itlb_multihit`/`vmscape` + `CMD` | **是**（本类的 cmdline 证据强） |
| M4 | **H1 为主** | 芯片厂商 + 固件包维护方 | 微码/固件 | `VULN,*` 里的 `no microcode` 片段；x86 的 `POL,microcode`；arm64 无 | **仅 x86 部分可**：arm64 需厂商证据 |
| M5 | H2 + H3 + 进程 | 内核 + 应用运行时 | 内核 + 应用自身 | `VULN,spec_store_bypass` + `CAP,ssbs/sb` | **部分**："via prctl" 只证明机制在位 |
| M6 | H1 + H2 + H3 | 芯片厂商 + 内核 | 固件 + 内核 | 只在 `spectre_v2` 后缀里（`, BHB` / `, but not BHB`） | **弱**：依赖厂商 CSV2 等级证据 |
| M7 | 无官方缓解 | 平台（准入） + 策略 | 系统与准入 | **无** | **不可**：只能用准入与运行集合治理 |
| M8 | **H6** | 库/TEE 固件/应用团队 | 应用发布、库升级 | **无** | **不可（在设备侧）**：由第十五章的实现侧证据验收 |
| M9 | H4 + H5 | 平台与厂商 sepolicy 维护者 | **sepolicy / init 片段（可独立于内核）** | `POL,selinux`、`POL,hidepid`、`SYSCTL,*`、`COUNT,*`、`cpufreq_governor` | **是**：唯一可拿快照差分直接验收的层 |
| M10 | 器件与固件 | 器件厂商 | 器件固件 | **无**；虚拟化时仅得 `Unknown:` 句 | **不可**：第十七章 |

三条来自矩阵整体的结论：

- **"设备侧证据"这一列只有三行是"有"级别的：M3、M9，以及 M4 的 x86 半边。** 所以任何把侧信道治理做成"每月扫一遍 `vulnerabilities` 全绿即通过"的做法，**实际上只覆盖了十型里的两型半**。这是第十九章的立论基础。
- **可验收性与整改速度不成比例。** M9 最快也最可验收；M4/M6 最慢且（arm64）不可验收。治理节奏上正确的顺序是**先 M9，再 M3/M5，M4/M6 走厂商证据流，M7/M8/M10 走另两条完全不同的流程（准入流与实现流）**——把它们混在同一个看板上是这一领域最常见的组织性失败。
- **同一个型号设备上的差异主要来自后两列而不是前两列。** 因为 M 型与层由硅决定，而通道与证据由整机与策略决定。这就是第一章"等级不跨设备比较"的具体展开，也是第六章把 ABI 那句话（**"reflects the state of the CPUs in the system"**）当作全篇边界的原因。
## 十七、安全元件是否在侧信道口径内免疫：把"免疫"拆成三个问题

移动端最常见的过度声明是"密钥在安全元件里，所以侧信道不适用"。本章把这句话拆成三个可以分别回答的问题，并且只回答设备侧能回答的那一部分。

### 十七之一 三个问题分开问

| 问题 | 能否在设备侧只读回答 | 本文的处理 |
| --- | --- | --- |
| **Q1 密钥是否曾经离开安全元件？** | **部分**：可读到的只有 KeyMint/StrongBox 的形态线索与构建属性，读不到密钥路径 | 归口到[硬件信任根与密钥栈](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)；本文不代答 |
| **Q2 安全元件自身是否受微架构侧信道影响？** | **否**：设备侧没有任何读数面覆盖另一个处理器；虚拟化时连宿主状态都变成 `Unknown:`（十三之五） | 只能取器件厂商公告；进入本章的派工口径 |
| **Q3 与元件交互的 Android 侧代码是否有 M8 型缺陷？** | **否**（H6 无读数），但**可由实现侧证据回答** | 第十五章的自查清单 |

**"密钥在安全元件里"这句话只回答了 Q1 的前半，对 Q2 与 Q3 没有任何作用。** 而真正在 Android 设备上高发的恰是 Q3：**元件内部做得再好，只要应用侧把"验证成功/失败"的时长或异常路径差异暴露出去，泄露就发生在 Android 侧，与器件无关。** 这就是 CWE-208 的定义句（参考资料 12）在本篇被单列为 M8 的原因。

### 十七之二 Q2 的可核验路径只有一条：向厂商要，并留档

本文对 Q2 唯一的操作性建议是**把它列为一条必须有外部证据的派工项**，并要求证据满足三条最低形式要求：

1. 指明**器件型号与固件版本**（不是平台名，不是产品名）；
2. 指明**结论覆盖的缺陷族**（微架构类缺陷是按机制分的，一次性"本器件不受侧信道影响"这种表述不可用）；
3. 指明**证据日期**，因为它会随新家族公布而失效。

这三条也是第二十六章 KPI 里"厂商证据完备率"一项的评分口径。

### 十七之三 一条来自上游的反面提醒

第五章列过：SRBDS 与 GDS 在 `X86_FEATURE_HYPERVISOR` 置位时直接读成 **`Unknown: Dependent on hypervisor status`**（参考资料 23）。**这条读数的存在说明：即便是内核自己，也无法在隔离层之下给出结论。** 把它推广到移动端的多处理器形态是合理的方法论而不是类比：**当你不在同一个可观测域里时，你只能声明不可判定。** 本文因此规定：**任何把 Q2 写成"免疫"的报告都必须同时给出上面三条证据，否则把该格改为"不可判定"。**

## 十八、为什么缓解项不能作为可独立更新的模块下发

Project Mainline 的公开表述是：**"Android 10 introduced Mainline (formerly called modular system components on this site)."**、**"Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle."**，以及投递渠道 **"End-user devices can receive Mainline updates from the Android Google Play system update feature (which runs on the Google Play Store infrastructure) or through a partner-provided over-the-air (OTA) mechanism."**（参考资料 26）。

### 十八之一 按层看，六层里没有一层能被模块单元送达

| 层 | 决定它的东西 | 能否作为独立可更新单元下发 | 实际通道 |
| --- | --- | --- | --- |
| H1 硅/微码 | 芯片与引导固件 | **否** | 厂商 OTA 的固件分区，且 `old_microcode` 只在启动时判定（参考资料 4） |
| H2 内核编译期 | `vmlinux`/`Image` 与 `CONFIG_*` | **否** | boot 镜像，属整机 OTA |
| H3 启动参数 | boot image 内的 cmdline/bootconfig | **否** | 同上；**且改它不需要重编译，因此它是"随一次无关改动顺被改掉"的那一层**（第三章） |
| H4 平台策略与 sysctl | sepolicy 与 init 脚本 | **有限**：这些文件位于系统分区镜像内 | 整机 OTA；本文的 `init.rc`/`sepolicy` 证据全部来自源码树而非模块（参考资料 5、6） |
| H5 运行时 | ART、框架与工具 | **部分是**（框架组件确实被模块化） | Mainline / OTA；与侧信道直接相关的只有"能请求到什么观测结果"这一段（参考资料 22） |
| H6 密钥实现 | 库与应用代码 | **部分是**（应用与库可独立发布） | 应用商店/SDK 发布——**这才是这一层真正的快速通道** |

结论一句话：**六层里唯一"改起来快、发出去也快"的两层，恰好都不在 H1/H2/H3；而 M4/M6 这类最严重的缺陷恰恰只能靠最慢的 H1 通道。** 这就是本篇反复强调"验收要按层而不是按编号"的现实原因。

### 十八之二 一个真实存在、但方向相反的模块化例子

Android 16 把性能剖析能力做成 Mainline 模块，其公开描述是：**"Apps can request profiling using Java heap dump, heap profile, stack sampling, or system trace, and set select configurations for each."** 与 **"Results are redacted to contain information relating to the requesting process only and delivered to the app's storage directory for further processing by the app."**（参考资料 22）。

这一例值得单独记住，因为它是**极少数"模块化确实改变了观测面"的情形**，而且方向是**收缩**：应用可请求的剖析结果被限定到"只含请求进程自身的信息"。三条推论：

1. **可观测范围的收缩可以随模块发布**——这给 H5 层一条真实的快通道，运维侧应当把它纳入变更来源，而不是只盯着内核 OTA；
2. 但它**收缩的是应用自己拿到的产物，不是内核实际的能力**：第十一章说明真正管住 `perf_event_open` 的是策略与 `sys.init.perf_lsm_hooks`，那是镜像内的东西，模块改不到；
3. 因此在设备侧，**这类变化的可见证据不是 `VULN` 行，而是 `POL/SYSCTL` 段（以及构建属性）是否变**。脚本 2 把 `POL,selinux` 与 `SYSCTL,*` 的变化单独出码，正是为了让这类"非内核整改"也能被差分捕捉到。

## 十九、"读数全部良好"为什么不构成结论

这一章把前十八章的边界条款汇成一条可执行的判定纪律。**它不是免责声明，而是本文对达标口径的定义。**

### 十九之一 四种"全绿"，每种都不是好消息

| 表象 | 为什么不能当结论 | 对应信号码/规则 |
| --- | --- | --- |
| **A. 所有文件都读 `Not affected`** | 上游为每个未实现 show 函数的项准备了回退，其实现就是 `return sysfs_emit(buf, "Not affected\n");`（六之二，参考资料 24）。**全 `Not affected` 至少有两种不可区分的成因** | `COUNT,vuln_files_total` 与 `COUNT,vuln_not_affected` 相等时，推测执行段整段记"不构成结论" |
| **B. 所有文件都读 `Mitigation:`** | 说明文字只证明内核选了哪条实现，不证明充分：`Mitigation: Speculative Store Bypass disabled via prctl` 把开关交给进程；`Mitigation: AVX disabled, no microcode` 说明硅侧仍缺；`Mitigation: Vulnerable, KVM: Not affected` 甚至同时含两种判定 | `vuln_mitigated` 与 `vuln_note_microcode`/`vuln_note_smt`/`vuln_note_ebpf` **可并存**，不互斥 |
| **C. cmdline 干净、sysctl 严格** | 这只覆盖 H3/H4，且**八之三说明 cmdline 上的串可能被静默忽略**（`pr_err("Ignoring unknown spectre_v2 option (%s).", str);`）；`perf_event_paranoid=-1` 在 Android 上是策略接管的标志而非放松 | `cmdline_mitigations_default`、`sysctl_paranoid_low` 的措辞都限定"该读取域内" |
| **D. 没有任何 `L3-HIGH` 行** | M7/M8/M10 三型**在定义上不产生任何行**（第十五、十六、十七章）。"没有高危"与"高危不在观测面上"是两件事 | `SUMMARY` 之外必须交叉第二十六章的 K8/K9 |

### 十九之二 三条不可跨越的比较规则

1. **不跨架构比。** x86 与 arm64 词表不同、能力位不同、`spectre_v1` 一句是常量、BHB 无独立文件、微码锚点只在一侧（第七章）。脚本 2 进门先比 `STATE,arch`，不同就出 `arch_changed` 并声明"逐项差异不能按同族升级解读"。
2. **不跨采集面比。** 一次采集少了 `ls`、`cpuinfo`、`cmdline` 任一源，"这一项消失了"就有第二种成因。脚本 2 因此**先算采集面是否收窄**（`snapshot_thinner`、`vuln_files_fewer`、`sysctl_became_unread`、`proc_mount_became_unread` 等码），再解释风险项变化；收窄时消失的一律记 `risk_gone_by_coverage`。
3. **不跨设备比。** 十六之一说明同一型号可能因 boot CPU、调度落核、厂商 cmdline 而不同。**一台设备的读数为另一台设备背书，在本篇的口径里一律不成立。**

### 十九之三 消失≠已修

这是全系列共用、在本篇被用得最狠的一条纪律。两种"消失"必须分开：

- **`risk_gone_needs_proof`**：采集面没变、某条高危读数为 `Mitigation:`/`Not affected` 了。**仍然要一条变更依据**（哪次 OTA、哪条 cmdline 改动、哪个固件版本），否则只是"读数变了"而不是"整改完成"。特别提醒 `old_microcode`：**运行时更新微码不会改变任何读数**（参考资料 4），所以"读数没变"也不能反证"没更新"。
- **`risk_gone_by_coverage`**：采集面收窄导致的消失。**直接读成缺口**，且必须同时输出采集退化那条行本身，避免读者只看中高危计数下降就宣布收敛。

第廿四章的差分样例专门演示这一种：同一批判据里中高危行大面积减少、缺口行大面积增加，脚本的结论是"不可比"，而不是"变好"。

## 二十、四级判定模型与信号码约定

### 二十之一 四级

| 等级 | 定义 | 处置要求 | 典型码（脚本 1） |
| --- | --- | --- | --- |
| **L3-HIGH** | 设备侧证据**直接表明**缓解处于关闭或明确失效状态，且该结论不依赖推断 | **立即处置**：定位改动来源、回滚或提交变更单，重采并留存 | `vuln_vulnerable`、`cmdline_mitigations_off` |
| **L2-WARN** | 证据表明**收敛度低于同层可达状态**，但后果取决于其它层或不可在设备侧确认 | **限期核查**：确认改动依据；跨层组合时联合评估 | `cmdline_disable_token`、`vuln_state_unknown_prefix`、`vuln_note_ebpf`、`selinux_permissive`、`sysctl_kptr_open`、`microcode_changed` |
| **L1-INFO** | 中性事实或改善方向，**不单独构成结论**，用于建立基线与可比性 | **归档**：作为下一次差分的基线 | `vuln_mitigated`、`selinux_enforcing`、`cpu_cap_present`、`vuln_bhb_covered_in_v2` |
| **L0-LOW** | **缺口**：读不到、判不出、或该平台形态在本口径下不可判定 | **补齐采集或转外部证据流**，绝不允许当通过 | `vuln_dir_unread`、`cpuinfo_unread`、`sysctl_unread`、`arm_microcode_face_absent`、`vuln_core_file_absent` |

四条实现约定，它们是这套码位能被交叉复算的前提：

1. **一码一判据。** 同一现象不因来源不同而用不同码；反过来，`vuln_vulnerable` 与 `vuln_note_microcode` **可以同一行并存**，因为一条讲等级、一条讲归属。
2. **读不到 ≠ 0。** 缺失一律显式写 `none`/`unread` 并出 L0 行（十四之一）。脚本 2 因此能区分"这项没这项"与"这项读不到"。
3. **不跨码位合成总分。** 本报告没有"侧信道分数"。任何把等级映射成 0–100 分的做法都会自动违反十九之二（跨设备比较）。
4. **每个码位都必须可复现。** 第廿五章给出验收流程：脚本里声明的每个码，在合成档案与快照对全集上至少真实触发一次；**从不触发的分支视为报告的缺陷而非样例的缺失。**

### 二十之二 输出契约（供二次开发与自动比对使用）

**脚本 1**（快照）逐行前缀：

| 前缀 | 形态 | 说明 |
| --- | --- | --- |
| `META` | `META,<k>,<v>` | 身份与采集量；缺值写 `none`，绝不省略键 |
| `STATE` | `STATE,arch,<arm64\|arm\|x86\|x86_64\|unknown>` | 架构判定，决定后续哪些缺席项有意义 |
| `VULN` | `VULN,<name>,<state>,<value…>` | `state` ∈ `not_affected\|mitigated\|vulnerable\|empty\|other`；第 4 段起为**原样读数**（含分号后缀） |
| `CMD` | `CMD,<token>,present\|absent\|unread[,<字面值>]` | 第 4 段仅 `mitigations=` 携带，用于档位漂移比对（八之三） |
| `CAP` | `CAP,<cap>,present\|absent\|unknown\|unread` | 能力位；`unknown` 表示 cpuinfo 有内容但该项不在此列，`unread` 表示整源不可读 |
| `POL` | `POL,selinux\|hidepid\|microcode,<v>` | 策略/挂载/微码三项 |
| `SYSCTL` | `SYSCTL,<key>,<v\|unread>` | 三个闸门，**始终三行齐全** |
| `COUNT` | `COUNT,<k>,<v>` | 分组计数，供十九之一 A 情形判"不构成结论" |
| `ROW` | `ROW,<lv>,<code>,<text>` | 判定行 |
| `SUMMARY` | `SUMMARY,…` | 四级计数与采集完整度 |

**脚本 2**（差分）逐行前缀：`PAIR`（两侧输入与行数）、`NOTE`（可比性声明）、`DELTA`（逐项差异事实）、`NEW`/`GONE`（单侧新增/消失）、`ROW`、`SUMMARY`。

**契约里最要紧的一条**：脚本 2 的输入**只能是脚本 1 的输出文本**。它不接触设备、不访问网络、不含任何探针。这使得留存基线可以异地比对、可以进代码评审，也可以在第廿四章里以文本形式直接放进报告——**这是"消失≠已修"这条纪律能够被第三方复查的唯一实现方式。**
## 廿一、脚本 1 的设计：一次只读采集，八类输出行，七十三的信号码

**文件名：`side-channel-posture-snapshot.sh`。用法：`side-channel-posture-snapshot.sh <设备序列号>`。依赖：`adb`。**

### 廿一之一 设计目标与非目标

目标是**在一条命令内产出一份可留档、可比对、可被第三方复算的设备侧姿态快照**。四类非目标同样重要：

| 不做 | 原因 |
| --- | --- |
| 不运行任何计时/缓存探针 | 本文范围声明；而且这类测量本身有副作用与权限需求（第十四章） |
| 不读任何 `/proc/<pid>/` 条目 | 采集固定从 shell 域发起，避免把"谁在观测"混进读数（十四之二） |
| 不解析内核日志 | 需要更高权限且内容随厂商改动（十四之二） |
| 不做跨设备评分 | 十九之二的三条比较规则不允许 |

### 廿一之二 采集层：八个命令、一层封装

所有设备访问经三个小函数收口：`raw`（执行 `adb shell` 并剥掉 `\r`）、`p`（`getprop` 取首行）、`f1`（`cat` 取首行）。**这样做的唯一目的是让"采集失败"只有一种表现：字符串为空。** 后续每一节都只需要区分"空/非空"，不需要再判断退出码，也就不会出现"命令报错但被当成读到 0"的情形。

采到的原始数据存进十一个变量（型号、厂商、SDK、SPL、build type、qemu、hardware、board、abilist、bootloader、内核版本串）与五个整段文本（`vullist` 目录清单、`cmdline`、`cpuinfo`、`enforce`、`mounts`、`pslist`）。两个长度值 `cpuinfo_bytes`/`cmdline_bytes` 单独入 `META`：**它们让"读到但内容很短"与"没读到"在数值上可区分**，第十四章的采集完整度指标以此为基础。

### 廿一之三 架构判定是第一个分叉点

判定顺序是 `cpuinfo` → `uname` → `abilist`（三级回退）：`CPU implementer`/`AArch64` 判 arm64，`model name`/`vendor_id`/`microcode` 判 x86；内核串里的 `aarch64`/`arm64` 与 `x86_64`/`i686`/`smp` 次之；最后才用 `ro.product.cpu.abilist`。**顺序不能反**——`abilist` 表示"这台设备能跑什么 ABI"，不表示"内核打印了什么"。第七章的所有词表选择、缺席项解释、能力位清单都挂在这个判定上；判不出时不猜，直接出 `vuln_arch_unknown`（`L0-LOW`）并把整节解释作废。

一个已知折衷：`smp` 也出现在 arm64 内核串里，所以第三级之前的 `uname` 判定可能把 arm64 误判成 x86。**这是本文接受的风险，理由是 `cpuinfo` 通常先给出结论**；一旦两侧矛盾，第廿五章的验收流程会把这类档案作为异常样本单独复核。

### 廿一之四 分类器：前缀锚定，不是子串匹配

这是本篇唯一一处"读错就会把风险读反"的地方，所以单独说明。实现是：

```bash
    stt=other
    case "$val" in
      '')                  stt=empty ;;
      "Not affected"*)     stt=not_affected ;;
      "Mitigation"*)       stt=mitigated ;;
      "Vulnerable"*)       stt=vulnerable ;;
    esac
```

**四条全部锚在字符串开头。** 原因在第五章已经给出：x86 的 `spectre_v2` 是八个片段拼出来的，里面会出现 `; PBRSB-eIBRS: Not affected`、`; BHI: Not affected`；`indirect_target_selection` 甚至有一个上游取值 `Mitigation: Vulnerable, KVM: Not affected`。**按子串匹配的实现会把这条读成"不受影响"**，把等级从 L3 直接降到 L1——这不是理论风险，是上一轮核验时在这个脚本里真实存在过的缺陷，本文把它写出来正是为了说明：**这类判定必须把上游词表读通再实现，不能靠"字符串里包含什么"。**

分类之后还有五条**可并存的附加记号**（`Unknown` 前缀、eBPF、SMT、no microcode、BHB 两半），它们不改前缀分类，只叠行。**并存是刻意的**：`vuln_vulnerable` 与 `vuln_note_microcode` 同时出现时，前者给等级、后者给归属方向（十三之三）。

### 廿一之五 八段判定与码位分布

七十三码按脚本内的自然分段如下（这张表也是第廿五章覆盖核对的分组依据，四个等级列由脚本源码里的 `row` 调用逐条统计得到）：

| 段 | 覆盖内容 | 码位数 | L3 | L2 | L1 | L0 |
| --- | --- | --- | --- | --- | --- | --- |
| vulnerabilities 逐文件 | 三前缀分类 + 五种附加记号 + 非 ABI 名单 + arm64 的 BHB 文字锚点 | 14 | 1 | 6 | 6 | 1 |
| 读数面缺席与架构 | 目录整体读不到、三个核心文件缺席、x86 专属缺席、arm 无微码面、架构判不出 | 6 | 0 | 0 | 0 | 6 |
| cmdline | 12 个记号 + `mitigations=` 三档语义 + 关闭串 + `off` 取值 + `init_on_free` | 8 | 1 | 2 | 4 | 1 |
| CPU 特性位与微码 | arm64 五项 / x86 六项 + AES + 微码行 + cpuinfo 整体缺失 | 11 | 0 | 3 | 5 | 3 |
| SELinux、三个 sysctl 与 hidepid | 强制模式四态 + 三闸门 + `/proc` 挂载行 | 15 | 0 | 5 | 6 | 4 |
| 邻居与运行时 | 进程表、zygote、采集域声明 | 4 | 0 | 0 | 1 | 3 |
| 身份与构建形态 | 型号、bootloader、qemu、build type、SPL | 10 | 0 | 2 | 4 | 4 |
| 内核版本与调频 | 版本可判性、读数接口档位、governor | 5 | 0 | 1 | 2 | 2 |
| **合计** | | **73** | **2** | **19** | **28** | **24** |


**L3 只有两码（`vuln_vulnerable`、`cmdline_mitigations_off`）是有意为之。** 本篇不制造高危噪声；其余一律交给等级较低的 L2 与缺口 L0，让"高危"这个词在报告里保持可执行的含义：**看到 L3 就是要么读数自述无缓解、要么启动参数把缓解关了，没有第三种。**

### 廿一之六 输出契约与 SUMMARY

行前缀集合与字段定义在二十之二给出，此处补三条实现约定：

1. **`SYSCTL` 永远三行齐全**（读不到写 `unread`）。这是脚本 2 能区分"节点被删除"与"采集权限退化"的形式前提——虽然两者在本实现里仍不等价（脚本 2 对此明确出 `sysctl_became_unread` 并声明不等价）。
2. **`CMD,mitigations=` 带第 4 字段**（cmdline 上的字面取值，如 `mitigations=auto`）。没有它，脚本 2 无法实现"两侧都 present 但档位从 `auto` 变 `off`"这一类最危险的漂移（八之三）。
3. **`SUMMARY` 是最后一行**，含四级计数、`vuln` 总数、`vuln_bad`、架构、型号、SPL。**它是报表侧唯一的聚合入口**，其余全部要按行解析。

`set -u` 全程开启，输入设备不可达时**只往 stderr 写一行 `ERR,device_unreachable,<serial>` 并以 1 退出，不产出半份快照**。**宁可不产出，也不产出可被误读的残缺产物**——这条与"未采到=缺口"是同一个纪律的两面：残缺必须是显式的。

### 廿一之七 完整清单

下面就是脚本 1 的全文，与本报告内嵌的其余片段一致，未做任何省略：

```bash
#!/usr/bin/env bash
# 脚本 1: side-channel-posture-snapshot.sh — 微架构侧信道姿态快照（只读）
# 用法: side-channel-posture-snapshot.sh <设备序列号>
# 依赖: adb；只调用 get-state / getprop / uname / cat / ls / getenforce / mount / ps -A，全部只读
# 不写设备、不安装任何东西、不访问网络、不采集任何密钥材料、不运行任何计时或缓存探针
set -u
dev="${1:?用法: side-channel-posture-snapshot.sh <设备序列号>}"
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable,$dev" >&2; exit 1; }

hi=0; warn=0; gap=0; info=0
row() { # row <L3-HIGH|L2-WARN|L1-INFO|L0-LOW> <信号码> <格式串> [参数…]
  local lv="$1" code="$2" fmt="$3"; shift 3
  local txt
  if [ "$#" -gt 0 ]; then txt="$(printf "$fmt" "$@")"; else txt="$fmt"; fi
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
p()    { raw getprop "$1" | head -1; }
f1()   { raw cat "$1" | head -1; }

model="$(p ro.product.model)"; manuf="$(p ro.product.vendor.manufacturer)"
sdk="$(p ro.build.version.sdk)"; spl="$(p ro.build.version.security_patch)"
btype="$(p ro.build.type)"; qemu="$(p ro.kernel.qemu)"
hw="$(p ro.boot.hardware)"; board="$(p ro.product.board)"
abilist="$(p ro.product.cpu.abilist)"; btl="$(p ro.boot.bootloader)"
krel="$(raw uname | head -1)"
VDIR="/sys/devices/system/cpu/vulnerabilities"
vullist="$(raw ls "$VDIR")"
cmdline="$(f1 /proc/cmdline)"
cpuinfo="$(raw cat /proc/cpuinfo)"
enforce="$(raw getenforce | head -1)"
mounts="$(raw mount)"
pslist="$(raw ps -A -o NAME)"
cbytes="$(printf '%s' "$cpuinfo" | wc -c | tr -d ' ')"
kbytes="$(printf '%s' "$cmdline" | wc -c | tr -d ' ')"

meta device_serial "$dev"; meta model "$model"; meta manufacturer "$manuf"
meta board "$board"; meta hardware "$hw"; meta sdk "$sdk"
meta security_patch "$spl"; meta build_type "$btype"
meta kernel_release "$krel"; meta cpu_abilist "$abilist"
meta bootloader "$btl"; meta cpuinfo_bytes "$cbytes"; meta cmdline_bytes "$kbytes"

# ---------- 架构判定：决定后面读哪一组特性位、哪些文件本就不该存在 ----------
arch=unknown
if [ -n "$cpuinfo" ]; then
  case "$cpuinfo" in
    *"CPU implementer"*|*"AArch64"*) arch=arm64 ;;
    *"model name"*|*"vendor_id"*|*"microcode"*) arch=x86 ;;
  esac
fi
if [ "$arch" = unknown ] && [ -n "$krel" ]; then
  case "$krel" in *aarch64*|*arm64*) arch=arm64 ;; *x86_64*|*i686*|*smp*) arch=x86 ;; esac
fi
if [ "$arch" = unknown ] && [ -n "$abilist" ]; then
  case "$abilist" in *arm64*) arch=arm64 ;; *armeabi*) arch=arm ;; *x86_64*|*x86*) arch=x86 ;; esac
fi
meta arch "$arch"
printf 'STATE,arch,%s\n' "$arch"

cmdtok() { # cmdtok <记号> -> present | absent | unknown
  [ -z "$cmdline" ] && { echo unknown; return; }
  case " $cmdline " in *" $1"*) echo present ;; *) echo absent ;; esac
}
cpuf() { # cpuf <特性词> -> present | absent | unknown
  [ -z "$cpuinfo" ] && { echo unknown; return; }
  if printf '%s\n' "$cpuinfo" | grep -qw -- "$1"; then echo present; else echo absent; fi
}
hasv() { case " $vnames " in *" $1 "*) return 0 ;; *) return 1 ;; esac; }

# ---------- vulnerabilities 逐文件读数 ----------
# 上游 Documentation/ABI/testing/sysfs-devices-system-cpu 列出的文件名（2026-10 快照）
ABI_NAMES="gather_data_sampling indirect_target_selection itlb_multihit l1tf mds meltdown mmio_stale_data old_microcode reg_file_data_sampling retbleed spec_rstack_overflow spec_store_bypass spectre_v1 spectre_v2 srbds tsa tsx_async_abort ghostwrite vmscape"
vn=0; vbad=0; vmit=0; vna=0; vother=0; vempty=0; vabi=0
vnames=""
inabi() { local a; for a in $ABI_NAMES; do [ "$a" = "$1" ] && return 0; done; return 1; }
if [ -z "$vullist" ]; then
  row L0-LOW vuln_dir_unread "未采到 $VDIR 目录清单：本快照对全部推测执行类缓解状态不做判定，不得读成没有暴露面"
else
  while IFS= read -r entry; do
    [ -n "$entry" ] || continue
    name="${entry##*/}"; name="${name%\*}"
    case "$name" in ''|.|..|.*) continue ;; esac
    val="$(f1 "$VDIR/$name")"
    # 上游 ABI 把取值定义为三种前缀之一；x86 的复合读数里会内嵌 "PBRSB-eIBRS: Not affected"
    # 这类片段，所以判定必须锚在开头，不能按子串匹配，否则整条会被误判成"不受影响"。
    stt=other
    case "$val" in
      '')                  stt=empty ;;
      "Not affected"*)     stt=not_affected ;;
      "Mitigation"*)       stt=mitigated ;;
      "Vulnerable"*)       stt=vulnerable ;;
    esac
    vn=$(( vn + 1 )); vnames="$vnames $name"
    printf 'VULN,%s,%s,%s\n' "$name" "$stt" "$val"
    case "$stt" in
      vulnerable)   vbad=$(( vbad + 1 ))
        row L3-HIGH vuln_vulnerable "$name 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层" ;;
      mitigated)    vmit=$(( vmit + 1 ))
        row L1-INFO vuln_mitigated "$name：$val" ;;
      not_affected) vna=$(( vna + 1 ))
        row L1-INFO vuln_not_affected "$name 自述不受影响：这是本芯片与本内核组合的读数，不外推到同族其它型号" ;;
      empty)        vempty=$(( vempty + 1 ))
        row L0-LOW vuln_file_empty "$name 出现在清单里但内容为空：读不到不等于没有暴露面，也不等于有" ;;
      *)            vother=$(( vother + 1 ))
        row L1-INFO vuln_other_state "$name 的取值不属于 Vulnerable / Mitigation / Not affected 三个前缀口径：$val" ;;
    esac
    if ! inabi "$name"; then
      vabi=$(( vabi + 1 ))
      row L1-INFO vuln_name_outside_abi "$name 不在上游 sysfs 漏洞 ABI 名单内：这是厂商内核或补丁集自行导出的读数，其命名与取值口径不在这份报告的可核验范围内；下面的按前缀分组计数仍包含它，消费该统计时要减去 COUNT,vuln_outside_abi"
    fi
    case "$val" in Unknown*)
      row L2-WARN vuln_state_unknown_prefix "$name 的取值以 Unknown 起头（$val）：这是内核词表里的降级档位，既不算已缓解也不算敞口已确认" ;;
    esac
    # 非特权 eBPF 记号：arm64 的 spectre_v2 有专门读数（见下），这里只处理其它情形
    if [ "$name" = spectre_v2 ] && [ "$arch" = arm64 ]; then :; else
      case "$val" in *eBPF*)
        row L2-WARN vuln_note_ebpf "$name 的缓解说明里带非特权 eBPF 记号（$val）：公开口径把这一记号列为缓解可被削弱的条件之一" ;;
      esac
    fi
    case "$val" in *SMT*)
      row L2-WARN vuln_note_smt "$name 的缓解说明里带同核多线程记号（$val）：同一物理核上的另一个线程是本机最近的观测者" ;;
    esac
    case "$val" in *"no microcode"*|*"No microcode"*)
      row L2-WARN vuln_note_microcode "$name 的缓解说明里带微码未到记号（$val）：内核侧框架在位，落到硅上的那一半没到" ;;
    esac
    case "$val" in *", but not BHB"*)
      row L2-WARN vuln_note_bhb_partial "$name 的缓解说明里带 CSV2 有而 BHB 无的组合记号（$val）：分支历史类面的兜底要靠异常入口的循环重试缓解，其覆盖率与代价由内核版本与具体核心决定" ;;
    esac
    case "$val" in *", BHB"*)
      row L1-INFO vuln_bhb_covered_in_v2 "$name 的缓解说明里带 BHB 已覆盖记号（$val）：arm64 上分支历史类面没有独立读数文件，这一串文字就是设备侧唯一的直接证据" ;;
    esac
    if [ "$name" = spectre_v2 ] && [ "$arch" = arm64 ] && [ -n "$val" ]; then
      case "$val" in *"Branch predictor hardening"*)
        row L1-INFO v2_arm_bp_hardening "arm64 spectre_v2 报 Branch predictor hardening：该核走上下文切换时冲刷预测器的路线，跨上下文猜测面由软件缓解关闭" ;;
      esac
      case "$val" in *"Vulnerable: Unprivileged eBPF enabled"*)
        row L2-WARN v2_arm_ebpf_reopens "arm64 spectre_v2 报 Unprivileged eBPF enabled：硬件与内核缓解在位，但非特权 eBPF 让代码在内核同域执行，公开口径明确把这条件列为缓解被削弱的入口" ;;
      esac
    fi
  done <<EOF
$vullist
EOF
fi
cnt vuln_files_total "$vn"; cnt vuln_vulnerable "$vbad"; cnt vuln_mitigated "$vmit"
cnt vuln_not_affected "$vna"; cnt vuln_empty "$vempty"; cnt vuln_other "$vother"
cnt vuln_outside_abi "$vabi"

# ---------- 关键项缺席：区分"这个平台不导出"与"没采到" ----------
if [ -n "$vullist" ]; then
  for need in spectre_v1 spectre_v2 spec_store_bypass; do
    hasv "$need" || row L0-LOW vuln_core_file_absent "$need 未出现在 $VDIR 清单中：本快照不推断它是平台不提供还是采集被截断"
  done
  case "$arch" in
    arm64|arm) # 上游 arm64 不导出独立的 spectre_bhb 文件，BHB 状态拼在 spectre_v2 的文字里（见上一节的 outside_abi 读数）
               row L0-LOW arm_microcode_face_absent "arm64 侧没有 x86 那样的微码读数面：CPU 侧修复随厂商固件到达，设备上没有可自证的字段，只能向厂商索取证据" ;;
    x86)   hasv meltdown || row L0-LOW vuln_meltdown_absent_x86 "x86 未导出 meltdown：可能是内核判定不影响、也可能是接口未提供，本快照不区分"
           hasv mds      || row L0-LOW vuln_mds_absent_x86 "x86 未导出 mds：同上，需要向厂商或内核配置确认这一项的可读性" ;;
    *) row L0-LOW vuln_arch_unknown "架构判定失败：缺席项的解释依赖架构（哪些文件本就不该存在），这一节只留缺口" ;;
  esac
fi

# ---------- 内核 cmdline 上的缓解开关（只列上游 kernel-parameters.txt 里存在的写法） ----------
CMDTOKS="mitigations= nopti kpti=0 nokaslr nospectre_v1 nospectre_v2 nospectre_bhb ssbd= spec_store_bypass_disable= spectre_v2_user= spectre_bhi= init_on_free=1"
OFFVALS="ssbd=force-off spec_store_bypass_disable=off spectre_v2_user=off spectre_bhi=off retbleed=off mds=off"
if [ -z "$cmdline" ]; then
  for tok in $CMDTOKS; do printf 'CMD,%s,unread\n' "$tok"; done
  row L0-LOW cmdline_unread "未采到 /proc/cmdline：启动参数层的全部判定不成立，读不到不等于按默认值运行"
else
  cnt cmdline_bytes "$kbytes"
  # mitigations= 额外带第 4 字段：cmdline 上这一项的字面取值，供脚本 2 做档位漂移比对
  mval="$(printf '%s\n' $cmdline | grep -m1 '^mitigations=' || true)"
  for tok in $CMDTOKS; do
    case "$tok" in
      mitigations=) printf 'CMD,mitigations=,%s,%s\n' "$(cmdtok mitigations=)" "${mval:-none}" ;;
      *)            printf 'CMD,%s,%s\n' "$tok" "$(cmdtok "$tok")" ;;
    esac
  done
  case "$(cmdtok mitigations=)" in
    present)
      case "$cmdline" in
        *"mitigations=off"*) row L3-HIGH cmdline_mitigations_off "cmdline 含 mitigations=off：上游文档写明这一档等价于把各架构的关闭开关一并注入（arm64 上含 nospectre_bhb 与 ssbd=force-off，x86 上含 nopti 与 spec_store_bypass_disable=off），vulnerabilities 读数会随之集体变为 Vulnerable" ;;
        *"mitigations=auto,nosmt"*) row L1-INFO cmdline_mitigations_nosmt "cmdline 含 mitigations=auto,nosmt：这是上游口径里最完整的档位，代价是在受影响核上关闭同核多线程——同一物理核上的另一个线程本是最近的观测者" ;;
        *) row L1-INFO cmdline_mitigations_set "cmdline 显式固定了 mitigations= 档位：需与厂商出厂启动参数比对，出厂档位与手工档位含义不同" ;;
      esac ;;
    absent) row L1-INFO cmdline_mitigations_default "cmdline 未出现 mitigations=：按内核默认档位运行；默认档位在两次快照之间可能因内核升级而改变" ;;
  esac
  for tok in nopti kpti=0 nokaslr nospectre_v1 nospectre_v2 nospectre_bhb; do
    [ "$(cmdtok "$tok")" = present ] && row L2-WARN cmdline_disable_token "cmdline 含 $tok：上游把这一项定义为显式关闭某一类缓解或地址空间隔离的开关，带它启动时系统允许数据泄露"
  done
  for ov in $OFFVALS; do
    case " $cmdline " in *" $ov"*) row L2-WARN cmdline_off_value "cmdline 含 $ov：这一档把对应缓解取值为关闭，属于性能优先配置，与出厂默认档不同" ;; esac
  done
  [ "$(cmdtok init_on_free=1)" = present ] && row L1-INFO cmdline_init_on_free "cmdline 含 init_on_free=1：释放内存即清零，公开口径下这会压缩残留数据被再次读到的窗口"
fi

# ---------- CPU 特性位：缓解是否有硬件入口 ----------
# 只用内核确实会打印的名字：arm64 侧 CSV2/CSV3 是 FTR_HIDDEN，不出现在 Features 行，
# 其状态要从 spectre_v2 的说明文字里读（见下一节），不能按特性位判定。
ARM_CAPS="ssbs sb wfxt afp gcs"
X86_CAPS="ssbd ibrs ibpb stibp md_clear flush_l1d"
case "$arch" in
  arm64|arm)
    for cap in $ARM_CAPS; do
      r="$(cpuf "$cap")"
      printf 'CAP,%s,%s\n' "$cap" "$r"
      case "$r" in
        present) row L1-INFO cpu_cap_present "$cap 位在 /proc/cpuinfo Features 中可见：这一类缓解有硬件入口可用" ;;
        absent)
          case "$cap" in
            ssbs) row L2-WARN cpu_ssbs_absent "arm64 未声明 ssbs：存储绕过抑制要靠内核逐次插入指令或由固件代做，热路径代价与覆盖率都由内核与固件决定" ;;
            sb)   row L2-WARN cpu_sb_absent "arm64 未声明 sb（Speculation Barrier）：清空分支历史的推荐序列要退回 DSB NSH + ISB 形式，异常入口的兜底代价更高" ;;
            *)    row L1-INFO cpu_cap_absent "$cap 位不可见：本机不要求这一项，不作为风险读数" ;;
          esac ;;
        unknown) row L0-LOW cpu_cap_unread "未采到 /proc/cpuinfo：不判定 $cap 一类硬件前提" ;;
      esac
    done
    r="$(cpuf aes)"; printf 'CAP,aes,%s\n' "$r"
    case "$r" in
      present) row L1-INFO cpu_aes_cap "arm64 声明 aes 硬件加速：AES/SHA 计时面的归属落在 SoC 厂商，不在应用层" ;;
      absent)  row L1-INFO cpu_aes_absent_cap "未声明 aes 位：本机走软件实现路线，查表类密码实现的缓存计时面归属回到库与内核" ;;
      unknown) : ;;
    esac ;;
  x86)
    for cap in $X86_CAPS; do
      r="$(cpuf "$cap")"
      printf 'CAP,%s,%s\n' "$cap" "$r"
      case "$r" in
        present) row L1-INFO cpu_cap_present "$cap 位在 /proc/cpuinfo flags 中可见" ;;
        absent)  row L2-WARN cpu_x86_cap_absent "x86 flags 未声明 $cap：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性" ;;
        unknown) row L0-LOW cpu_cap_unread "未采到 /proc/cpuinfo：不判定 x86 缓解特性位" ;;
      esac
    done
    if [ -n "$cpuinfo" ]; then
      mc="$(printf '%s\n' "$cpuinfo" | grep -i -m1 '^microcode' | head -1)"
      printf 'POL,microcode,%s\n' "${mc:-none}"
      if [ -n "$mc" ]; then
        row L1-INFO cpu_microcode_line "$mc：微码版本是设备侧唯一能读到的硅侧缓解证据，比对要用厂商的微码版本表"
      else
        row L0-LOW cpu_microcode_unread "未采到 microcode 行：无法在设备侧证明微码缓解已到达"
      fi
    else
      printf 'POL,microcode,%s\n' unread
    fi ;;
  *) [ -z "$cpuinfo" ] && row L0-LOW cpuinfo_unread "未采到 /proc/cpuinfo：特性位判定全部留为缺口" ;;
esac

# ---------- 观察通道由谁收敛：SELinux 与三个 sysctl ----------
printf 'POL,selinux,%s\n' "${enforce:-unread}"
case "$enforce" in
  '')         row L0-LOW selinux_unread "未采到 getenforce：不判定强制模式，读不到不等于未启用" ;;
  Enforcing)  row L1-INFO selinux_enforcing "SELinux=Enforcing：域级限制生效，perf 与 /proc 侧观察通道的收敛由策略域决定" ;;
  Permissive) row L2-WARN selinux_permissive "SELinux=Permissive：域级限制只记录不拦截，本快照里所有依赖策略的收敛判定都要降级重读" ;;
  *)          row L2-WARN selinux_unexpected "getenforce 返回 $enforce：非 Enforcing/Permissive 口径，需确认采集域与构建形态" ;;
esac
for k in kernel.perf_event_paranoid kernel.kptr_restrict kernel.dmesg_restrict; do
  v="$(f1 "/proc/sys/$(printf '%s' "$k" | tr '.' '/')")"
  if [ -z "$v" ]; then
    printf 'SYSCTL,%s,%s\n' "$k" unread
    case "$k" in
      kernel.perf_event_paranoid) row L0-LOW sysctl_paranoid_unread "$k 未采到：发行构建上该节点常不导出，读不到时不能按上游默认值判定" ;;
      *)                          row L0-LOW sysctl_unread "$k 未采到：这一项在本设备不可读，不能按默认值判定" ;;
    esac
    continue
  fi
  printf 'SYSCTL,%s,%s\n' "$k" "$v"
  case "$k" in
    kernel.perf_event_paranoid)
      if [ "$v" -ge 2 ] 2>/dev/null; then
        row L1-INFO sysctl_paranoid_high "$k=$v：高档位收敛 PMU 采样与用户态回溯面"
      else
        row L2-WARN sysctl_paranoid_low "$k=$v：低档位意味着采样面在该读取域内开放；shell 域可读不等于应用域可读"
      fi ;;
    kernel.kptr_restrict)
      if [ "$v" = 0 ]; then
        row L2-WARN sysctl_kptr_open "$k=0：内核指针不哈希，符号化信息对可读 /proc 的域暴露"
      else
        row L1-INFO sysctl_kptr_hash "$k=$v：内核地址被遮蔽或哈希"
      fi ;;
    kernel.dmesg_restrict)
      if [ "$v" = 0 ]; then
        row L2-WARN sysctl_dmesg_open "$k=0：内核日志对非特权域开放，日志里的时序与地址信息属于可被动收集的旁证"
      else
        row L1-INFO sysctl_dmesg_restricted "$k=$v：内核日志读取被限制"
      fi ;;
  esac
done
hp=unknown
if [ -n "$mounts" ]; then
  if printf '%s\n' "$mounts" | grep -qE ' /proc .*hidepid=(invisible|1|2|restricted)'; then hp=on
  elif printf '%s\n' "$mounts" | grep -qE ' /proc '; then hp=off
  fi
fi
printf 'POL,hidepid,%s\n' "$hp"
case "$hp" in
  on)      row L1-INFO proc_hidepid_on "/proc 挂载带 hidepid：跨 UID 读取他进程 /proc 条目被挂载选项收敛；这一档在 Android 上属厂商自选，不是平台默认" ;;
  off)     row L1-INFO proc_hidepid_off "/proc 未带 hidepid：Android 上跨 UID 可见性主要由 SELinux 与 proc 权限决定，缺这一项不判成敞口" ;;
  unknown) row L0-LOW proc_mount_unread "未采到 /proc 挂载行：不判定 hidepid" ;;
esac

# ---------- 谁可能与你在同一颗芯片上 ----------
if [ -z "$pslist" ]; then
  row L0-LOW ps_unread "未采到进程表：同机邻居清点不成立"
else
  zy="$(printf '%s\n' "$pslist" | grep -cE '^zygote64$|^zygote$' || true)"
  cnt zygote_processes "${zy:-0}"
  if [ "${zy:-0}" -gt 0 ]; then
    row L1-INFO zygote_shared "采到 ${zy} 个 zygote 进程：应用进程由共享母进程孵化，预加载类与代码缓存的初始缓存状态在同一台机器上是公共的"
    row L0-LOW neighbor_app_domain_gap "本次采集在 shell 域：普通应用域能否观察到同样这些通道，本快照不覆盖，需按应用侧实验另立证据"
  else
    row L0-LOW zygote_absent "进程表里没有 zygote：要么采集被截断，要么该设备形态不是标准 Android 运行时"
  fi
fi

# ---------- 构建与设备形态噪声 ----------
[ -n "$model" ] || row L0-LOW identity_missing "未采到机型标识：本快照无法与厂商公告、微码版本表对齐"
case "$btl" in
  '') row L0-LOW bootloader_absent "未采到 ro.boot.bootloader：CPU 侧修复随厂商固件包到达时，这是设备侧唯一的固件版本锚点，缺失即不可归因" ;;
  *)  row L1-INFO bootloader_read "ro.boot.bootloader=$btl：可把厂商固件版本与公告对齐；它不证明微码级别，只证明出厂镜像标识" ;;
esac
case "$qemu" in
  1) row L2-WARN emulator_build "ro.kernel.qemu=1：模拟器形态，其微架构读数不代表任何真实 SoC，本快照只标形态不下风险结论" ;;
  ''|0) : ;;
  *)  row L1-INFO qemu_prop_odd "ro.kernel.qemu=$qemu：取值不属于 0/1 口径，按采集噪声处理" ;;
esac
case "$btype" in
  '')   row L0-LOW build_type_unread "未采到 ro.build.type：不判定构建形态" ;;
  user) row L1-INFO build_type_user "ro.build.type=user：发行构建，调试类观察通道按平台默认收敛" ;;
  *)    row L2-WARN build_type_nonuser "ro.build.type=$btype：非发行构建，perf、trace 与调试通道的默认收敛可能比发行构建宽松" ;;
esac
case "$spl" in
  '') row L0-LOW spl_absent "未采到安全补丁级别：本快照不做任何缓解代码到达性判断" ;;
  *)  row L1-INFO spl_read "security_patch=$spl：只作时间锚点；SPL 不表示微码或内核缓解项是否生效，这两件事各有独立证据" ;;
esac
kpair="$(printf '%s' "$krel" | sed -e 's/^\([0-9][0-9]*\)\.\([0-9][0-9]*\).*/\1.\2/')"
case "$kpair" in
  ''|*[!0-9.]*) row L0-LOW kernel_release_unread "未解析到内核版本：不做读数接口可用性档位判断" ;;
  *)
    kmaj="${kpair%%.*}"; kmin="${kpair#*.}"
    if [ "$kmaj" -lt 4 ] || { [ "$kmaj" -eq 4 ] && [ "$kmin" -lt 16 ]; }; then
      row L2-WARN kernel_pre_vulnfs "内核 $krel 早于 vulnerabilities 读数接口普及的版本区间：即便存在缓解代码，设备侧也没有统一读数面"
    else
      row L1-INFO kernel_release_read "内核 $krel：vulnerabilities 读数面在该版本区间内应当存在"
    fi ;;
esac
gov="$(f1 /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor)"
case "$gov" in
  '') row L0-LOW cpufreq_unread "未采到 cpufreq governor：DVFS 状态未知，计时侧信道的信噪比条件无法归一化" ;;
  *)  row L1-INFO cpufreq_governor "scaling_governor=$gov：频率档位形态决定计时噪声底与频率量化程度，跨设备比较等级前必须先看这一行" ;;
esac

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,vuln=%s,vuln_bad=%s,arch=%s,model=%s,spl=%s\n' \
  "$hi" "$warn" "$gap" "$info" "$vn" "$vbad" "$arch" "${model:-none}" "${spl:-none}"
```
## 廿二、脚本 1 的三份真实输出

本章给出三份**脚本原样产出、未经编辑**的快照。选取它们不是为了展示三种设备，而是为了让读者在同一批文字里看到三种完全不同的判读处境：

| 样例 | 档案 | 形态 | SUMMARY | 这份样例要教的事 |
| --- | --- | --- | --- | --- |
| A | `sc001` | arm64、`user` 构建、采集完整 | `high=0,warn=0,gap=2,info=26` | **"全绿"读法**：为什么 `high=0` 仍然不构成结论，以及两个 L0 缺口各自挡住哪一条结论 |
| B | `sc002` | x86、`userdebug` 构建、采集完整 | `high=6,warn=16,gap=3,info=9` | **"跨层组合"读法**：六条 L3 与十六条 L2 分属 H1/H3/H4 三层，同时放宽才是窗口 |
| C | `sc010` | arm64、`user` 构建、三处采集失败 | `high=1,warn=0,gap=11,info=13` | **"退化采集"读法**：一条高危 + 十一处缺口时，这份快照能说什么、不能说什么 |

采集对象是本报告配套的**合成设备档案**（`WDATA` 目录下的文本文件，由 `.mk15.sh` 生成），不是任何真实在网设备。用合成档案有两个原因：一是真实设备的 `/proc/cmdline` 与 `/proc/cpuinfo` 全文属于该设备的配置信息，不应当在公开报告里扩散；二是**只有合成档案能构造出"微码未到 + `mitigations=off` + SELinux Permissive"这类在同一台真实设备上未必存在、但判读规则必须被验证的组合**。本章的三份输出与第十九章的判读规则之间没有任何人为加工——把这三份文本喂回脚本 2，得到的差分行就是第廿四章的样例。

复现命令（把 `scNNN` 换成 `sc001` / `sc002` / `sc010`）：

```
export PATH="$PWD/.wstub15:$PATH"; WDATA="$PWD/.wdata15" bash .sb15a.sh scNNN
```

`.wstub15/adb` 是只读桩：它把 `adb -s <serial> shell <cmd>` 映射到 `WDATA/<serial>/` 下的对应文本文件，任何写操作（`push` / `install` / `rm`）都不实现。用桩而不是真实设备，是为了让**读者可以在自己的机器上逐字节复现本章的全部输出**，而不需要借到这三台设备。

### 廿二之一 样例 A：arm64 全绿，但两个缺口各自挡住一条结论

```text
META,device_serial,sc001
META,model,Pixel 7
META,manufacturer,Google
META,board,cheetah
META,hardware,tensor
META,sdk,33
META,security_patch,2026-10-05
META,build_type,user
META,kernel_release,5.10.209-android13-8-g0e57eb5cd3a1 #1 SMP PREEMPT
META,cpu_abilist,arm64-v8a,armeabi-v7a,armeabi
META,bootloader,cheetah-1.4.2-11791055
META,cpuinfo_bytes,158
META,cmdline_bytes,88
META,arch,arm64
STATE,arch,arm64
VULN,spectre_v1,mitigated,Mitigation: __user pointer sanitization
ROW,L1-INFO,vuln_mitigated,spectre_v1：Mitigation: __user pointer sanitization
VULN,spectre_v2,mitigated,Mitigation: CSV2, BHB
ROW,L1-INFO,vuln_mitigated,spectre_v2：Mitigation: CSV2, BHB
ROW,L1-INFO,vuln_bhb_covered_in_v2,spectre_v2 的缓解说明里带 BHB 已覆盖记号（Mitigation: CSV2, BHB）：arm64 上分支历史类面没有独立读数文件，这一串文字就是设备侧唯一的直接证据
VULN,spec_store_bypass,mitigated,Mitigation: Speculative Store Bypass disabled via prctl
ROW,L1-INFO,vuln_mitigated,spec_store_bypass：Mitigation: Speculative Store Bypass disabled via prctl
VULN,spectre_bhb,mitigated,Mitigation: Loop
ROW,L1-INFO,vuln_mitigated,spectre_bhb：Mitigation: Loop
ROW,L1-INFO,vuln_name_outside_abi,spectre_bhb 不在上游 sysfs 漏洞 ABI 名单内：这是厂商内核或补丁集自行导出的读数，其命名与取值口径不在这份报告的可核验范围内；下面的按前缀分组计数仍包含它，消费该统计时要减去 COUNT,vuln_outside_abi
VULN,ssb,not_affected,Not affected
ROW,L1-INFO,vuln_not_affected,ssb 自述不受影响：这是本芯片与本内核组合的读数，不外推到同族其它型号
ROW,L1-INFO,vuln_name_outside_abi,ssb 不在上游 sysfs 漏洞 ABI 名单内：这是厂商内核或补丁集自行导出的读数，其命名与取值口径不在这份报告的可核验范围内；下面的按前缀分组计数仍包含它，消费该统计时要减去 COUNT,vuln_outside_abi
COUNT,vuln_files_total,5
COUNT,vuln_vulnerable,0
COUNT,vuln_mitigated,4
COUNT,vuln_not_affected,1
COUNT,vuln_empty,0
COUNT,vuln_other,0
COUNT,vuln_outside_abi,2
ROW,L0-LOW,arm_microcode_face_absent,arm64 侧没有 x86 那样的微码读数面：CPU 侧修复随厂商固件到达，设备上没有可自证的字段，只能向厂商索取证据
COUNT,cmdline_bytes,88
CMD,mitigations=,absent,none
CMD,nopti,absent
CMD,kpti=0,absent
CMD,nokaslr,absent
CMD,nospectre_v1,absent
CMD,nospectre_v2,absent
CMD,nospectre_bhb,absent
CMD,ssbd=,absent
CMD,spec_store_bypass_disable=,absent
CMD,spectre_v2_user=,absent
CMD,spectre_bhi=,absent
CMD,init_on_free=1,absent
ROW,L1-INFO,cmdline_mitigations_default,cmdline 未出现 mitigations=：按内核默认档位运行；默认档位在两次快照之间可能因内核升级而改变
CAP,ssbs,present
ROW,L1-INFO,cpu_cap_present,ssbs 位在 /proc/cpuinfo Features 中可见：这一类缓解有硬件入口可用
CAP,sb,present
ROW,L1-INFO,cpu_cap_present,sb 位在 /proc/cpuinfo Features 中可见：这一类缓解有硬件入口可用
CAP,wfxt,present
ROW,L1-INFO,cpu_cap_present,wfxt 位在 /proc/cpuinfo Features 中可见：这一类缓解有硬件入口可用
CAP,afp,present
ROW,L1-INFO,cpu_cap_present,afp 位在 /proc/cpuinfo Features 中可见：这一类缓解有硬件入口可用
CAP,gcs,absent
ROW,L1-INFO,cpu_cap_absent,gcs 位不可见：本机不要求这一项，不作为风险读数
CAP,aes,present
ROW,L1-INFO,cpu_aes_cap,arm64 声明 aes 硬件加速：AES/SHA 计时面的归属落在 SoC 厂商，不在应用层
POL,selinux,Enforcing
ROW,L1-INFO,selinux_enforcing,SELinux=Enforcing：域级限制生效，perf 与 /proc 侧观察通道的收敛由策略域决定
SYSCTL,kernel.perf_event_paranoid,2
ROW,L1-INFO,sysctl_paranoid_high,kernel.perf_event_paranoid=2：高档位收敛 PMU 采样与用户态回溯面
SYSCTL,kernel.kptr_restrict,1
ROW,L1-INFO,sysctl_kptr_hash,kernel.kptr_restrict=1：内核地址被遮蔽或哈希
SYSCTL,kernel.dmesg_restrict,1
ROW,L1-INFO,sysctl_dmesg_restricted,kernel.dmesg_restrict=1：内核日志读取被限制
POL,hidepid,on
ROW,L1-INFO,proc_hidepid_on,/proc 挂载带 hidepid：跨 UID 读取他进程 /proc 条目被挂载选项收敛；这一档在 Android 上属厂商自选，不是平台默认
COUNT,zygote_processes,1
ROW,L1-INFO,zygote_shared,采到 1 个 zygote 进程：应用进程由共享母进程孵化，预加载类与代码缓存的初始缓存状态在同一台机器上是公共的
ROW,L0-LOW,neighbor_app_domain_gap,本次采集在 shell 域：普通应用域能否观察到同样这些通道，本快照不覆盖，需按应用侧实验另立证据
ROW,L1-INFO,bootloader_read,ro.boot.bootloader=cheetah-1.4.2-11791055：可把厂商固件版本与公告对齐；它不证明微码级别，只证明出厂镜像标识
ROW,L1-INFO,build_type_user,ro.build.type=user：发行构建，调试类观察通道按平台默认收敛
ROW,L1-INFO,spl_read,security_patch=2026-10-05：只作时间锚点；SPL 不表示微码或内核缓解项是否生效，这两件事各有独立证据
ROW,L1-INFO,kernel_release_read,内核 5.10.209-android13-8-g0e57eb5cd3a1 #1 SMP PREEMPT：vulnerabilities 读数面在该版本区间内应当存在
ROW,L1-INFO,cpufreq_governor,scaling_governor=schedutil：频率档位形态决定计时噪声底与频率量化程度，跨设备比较等级前必须先看这一行
SUMMARY,high=0,warn=0,gap=2,info=26,vuln=5,vuln_bad=0,arch=arm64,model=Pixel 7,spl=2026-10-05
```

**逐段读法。**

1. **`META` 段（第 1–14 行）先确定"这份快照说的是什么机器"。** `arch=arm64` 由 `/proc/cpuinfo` 里出现 `CPU implementer` 判出（第 14 行与 `STATE,arch,arm64` 是同一个值的两处记录，前者给人看，后者给脚本 2 读）。`cpuinfo_bytes=158`、`cmdline_bytes=88` 是**采集厚度锚点**：这两条不是装饰——脚本 2 无法判断"这一份是不是同一台设备的同一条采集流程"，但两次快照的字节数差出一个量级时，采集面收窄几乎是唯一解释。
2. **`VULN` 段（第 16–35 行）里有一条容易被略过的重要记录。** 五个读数文件中，`spectre_bhb`（第 23 行）与 `ssb`（第 26 行）**不在上游 sysfs 漏洞 ABI 的 19 个文件名之内**，于是各触发一条 `vuln_name_outside_abi`（第 25、28 行）。这两行不是噪声，它们是这条结论的落点：**`COUNT,vuln_files_total,5` 里有两个名字是厂商内核自行导出的，`COUNT,vuln_mitigated,4` 也把它们算在内**。要把这份快照写进任何统计口径，先减去 `COUNT,vuln_outside_abi`（这里是 2），得到"上游可核验口径下的读数：3 个文件中 2 个 mitigated、1 个 not_affected"。这正是 K4 指标要求的减法动作，也是第六章"这个面说的是状态，不是名字的可信度"的具体执行。
3. **第 20 行的 `vuln_bhb_covered_in_v2` 是这份快照里信息量最大的一行。** arm64 上分支历史注入**没有独立读数文件**，设备侧唯一的直接证据是 `spectre_v2` 说明文字末尾那个 `, BHB` 后缀。这一行因此不是"补充说明"，而是"这一类面在本机上到底缓解没缓解"的**唯一**答案来源——如果没有这一行，本快照对 BHB 就只能记缺口。
4. **`CMD` 段（第 37–50 行）十二个记号全部 `absent`。** 读法要克制：**这只说明启动参数层没有显式关闭缓解，不说明缓解生效**。第一章已经给出理由——`spectre_v2_parse_cmdline()` 对无法识别的取值只打印一行 `pr_err("Ignoring unknown spectre_v2 option (%s).")` 就返回 0，参数写了但被忽略的情形，设备侧读数与"没写"完全相同。
5. **`CAP` 段（第 51–62 行）四个位可见、一个不可见。** `ssbs`/`sb`/`wfxt`/`afp` 都在 `/proc/cpuinfo` 的 Features 行里，`gcs` 不在。按第七章的口径，`gcs` 属"本机不要求这一项"，**不参与任何判定**；真正有价值的是 `ssbs` 与 `sb`：前者意味着存储绕过抑制不必由内核逐次插指令，后者意味着异常入口的分支历史兜底有推荐序列可用。**注意这一段的名单里没有 `csv2` 与 `csv3`**——这两个是 `FTR_HIDDEN`，不会打印（第七章），所以这份快照里没有 `CAP,csv2,...` 行是**预期事件**，把它读成"这台机器没有 CSV2"就是把打印格式读成了硬件能力。
6. **`POL`/`SYSCTL` 段（第 63–72 行）四条收敛证据。** `Enforcing` + `paranoid=2` + `kptr_restrict=1` + `dmesg_restrict=1` + `hidepid=on`。这里要按第十一章的纪律读：**`hidepid=on` 是厂商自选，不是 Android 平台默认**，所以它出现是一项加分，不出现也不是敞口（脚本对 `off` 只给 `proc_hidepid_off`，L1，不判风险）。
7. **两条 L0 缺口各自挡住一条结论（第 36、75 行）。** `arm_microcode_face_absent` 挡住的是"这台设备的 CPU 侧修复已经到位"——arm64 上没有微码读数面，第 13 章因此只能给出"向厂商索取证据"的派工动作；`neighbor_app_domain_gap` 挡住的是"普通应用看不到这些通道"——本次采集在 shell 域，第 12 章的邻居清点因此只覆盖 shell 域可见的进程表，应用域要另立证据。**这两条缺口如果哪一天变成 L1，说明采集流程改了，不代表设备改了。**

**为什么 `high=0,warn=0` 仍然不是结论。** 把上面七条合起来看：这份快照在 H3、H4 两层没有任何放宽记号，H1 层在 arm64 上**结构性不可证**，H2 层只能从说明文字反推，H6 层**完全没有被这份快照覆盖**。按第十九章的定义，这是一份"读数良好 + 覆盖面有边界"的产物，能支撑的结论止于"在这台设备上、在这个采集域内、在这批可读字段里，内核侧没有自述敞口"。它不能支撑"这台设备不受侧信道影响"，也不能支撑"这台设备上的密钥实现是安全的"。

### 廿二之二 样例 B：六条 L3 分属三层，十六条 L2 同时放宽

```text
META,device_serial,sc002
META,model,Tablet X2
META,manufacturer,Intel
META,board,broxton
META,hardware,grouper
META,sdk,30
META,security_patch,2023-11-05
META,build_type,userdebug
META,kernel_release,4.19.81-00001-g81c7d33 #1 SMP PREEMPT
META,cpu_abilist,x86_64,x86
META,bootloader,none
META,cpuinfo_bytes,240
META,cmdline_bytes,65
META,arch,x86
STATE,arch,x86
VULN,spectre_v1,mitigated,Mitigation: usercopy/swapgs barriers and __user pointer sanitization
ROW,L1-INFO,vuln_mitigated,spectre_v1：Mitigation: usercopy/swapgs barriers and __user pointer sanitization
VULN,spectre_v2,vulnerable,Vulnerable: eIBRS with unprivileged eBPF
ROW,L3-HIGH,vuln_vulnerable,spectre_v2 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
ROW,L2-WARN,vuln_note_ebpf,spectre_v2 的缓解说明里带非特权 eBPF 记号（Vulnerable: eIBRS with unprivileged eBPF）：公开口径把这一记号列为缓解可被削弱的条件之一
VULN,spec_store_bypass,vulnerable,Vulnerable
ROW,L3-HIGH,vuln_vulnerable,spec_store_bypass 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
VULN,mds,vulnerable,Vulnerable: Clear CPU buffers attempted, no microcode
ROW,L3-HIGH,vuln_vulnerable,mds 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
ROW,L2-WARN,vuln_note_microcode,mds 的缓解说明里带微码未到记号（Vulnerable: Clear CPU buffers attempted, no microcode）：内核侧框架在位，落到硅上的那一半没到
VULN,l1tf,not_affected,Not affected
ROW,L1-INFO,vuln_not_affected,l1tf 自述不受影响：这是本芯片与本内核组合的读数，不外推到同族其它型号
VULN,mmio_stale_data,vulnerable,Vulnerable: Clear CPU buffers attempted, no microcode; SMT vulnerable
ROW,L3-HIGH,vuln_vulnerable,mmio_stale_data 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
ROW,L2-WARN,vuln_note_smt,mmio_stale_data 的缓解说明里带同核多线程记号（Vulnerable: Clear CPU buffers attempted, no microcode; SMT vulnerable）：同一物理核上的另一个线程是本机最近的观测者
ROW,L2-WARN,vuln_note_microcode,mmio_stale_data 的缓解说明里带微码未到记号（Vulnerable: Clear CPU buffers attempted, no microcode; SMT vulnerable）：内核侧框架在位，落到硅上的那一半没到
VULN,retbleed,vulnerable,Vulnerable
ROW,L3-HIGH,vuln_vulnerable,retbleed 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
VULN,srbds,mitigated,Mitigation: TSX disabled
ROW,L1-INFO,vuln_mitigated,srbds：Mitigation: TSX disabled
COUNT,vuln_files_total,8
COUNT,vuln_vulnerable,5
COUNT,vuln_mitigated,2
COUNT,vuln_not_affected,1
COUNT,vuln_empty,0
COUNT,vuln_other,0
COUNT,vuln_outside_abi,0
ROW,L0-LOW,vuln_meltdown_absent_x86,x86 未导出 meltdown：可能是内核判定不影响、也可能是接口未提供，本快照不区分
COUNT,cmdline_bytes,65
CMD,mitigations=,present,mitigations=off
CMD,nopti,absent
CMD,kpti=0,absent
CMD,nokaslr,present
CMD,nospectre_v1,absent
CMD,nospectre_v2,absent
CMD,nospectre_bhb,absent
CMD,ssbd=,absent
CMD,spec_store_bypass_disable=,absent
CMD,spectre_v2_user=,absent
CMD,spectre_bhi=,absent
CMD,init_on_free=1,absent
ROW,L3-HIGH,cmdline_mitigations_off,cmdline 含 mitigations=off：上游文档写明这一档等价于把各架构的关闭开关一并注入（arm64 上含 nospectre_bhb 与 ssbd=force-off，x86 上含 nopti 与 spec_store_bypass_disable=off），vulnerabilities 读数会随之集体变为 Vulnerable
ROW,L2-WARN,cmdline_disable_token,cmdline 含 nokaslr：上游把这一项定义为显式关闭某一类缓解或地址空间隔离的开关，带它启动时系统允许数据泄露
CAP,ssbd,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 ssbd：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
CAP,ibrs,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 ibrs：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
CAP,ibpb,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 ibpb：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
CAP,stibp,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 stibp：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
CAP,md_clear,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 md_clear：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
CAP,flush_l1d,absent
ROW,L2-WARN,cpu_x86_cap_absent,x86 flags 未声明 flush_l1d：对应缓解缺少硬件入口，只能靠内核替代实现或关闭特性
POL,microcode,microcode : 0x2ef
ROW,L1-INFO,cpu_microcode_line,microcode : 0x2ef：微码版本是设备侧唯一能读到的硅侧缓解证据，比对要用厂商的微码版本表
POL,selinux,Permissive
ROW,L2-WARN,selinux_permissive,SELinux=Permissive：域级限制只记录不拦截，本快照里所有依赖策略的收敛判定都要降级重读
SYSCTL,kernel.perf_event_paranoid,1
ROW,L2-WARN,sysctl_paranoid_low,kernel.perf_event_paranoid=1：低档位意味着采样面在该读取域内开放；shell 域可读不等于应用域可读
SYSCTL,kernel.kptr_restrict,0
ROW,L2-WARN,sysctl_kptr_open,kernel.kptr_restrict=0：内核指针不哈希，符号化信息对可读 /proc 的域暴露
SYSCTL,kernel.dmesg_restrict,0
ROW,L2-WARN,sysctl_dmesg_open,kernel.dmesg_restrict=0：内核日志对非特权域开放，日志里的时序与地址信息属于可被动收集的旁证
POL,hidepid,off
ROW,L1-INFO,proc_hidepid_off,/proc 未带 hidepid：Android 上跨 UID 可见性主要由 SELinux 与 proc 权限决定，缺这一项不判成敞口
COUNT,zygote_processes,2
ROW,L1-INFO,zygote_shared,采到 2 个 zygote 进程：应用进程由共享母进程孵化，预加载类与代码缓存的初始缓存状态在同一台机器上是公共的
ROW,L0-LOW,neighbor_app_domain_gap,本次采集在 shell 域：普通应用域能否观察到同样这些通道，本快照不覆盖，需按应用侧实验另立证据
ROW,L0-LOW,bootloader_absent,未采到 ro.boot.bootloader：CPU 侧修复随厂商固件包到达时，这是设备侧唯一的固件版本锚点，缺失即不可归因
ROW,L2-WARN,build_type_nonuser,ro.build.type=userdebug：非发行构建，perf、trace 与调试通道的默认收敛可能比发行构建宽松
ROW,L1-INFO,spl_read,security_patch=2023-11-05：只作时间锚点；SPL 不表示微码或内核缓解项是否生效，这两件事各有独立证据
ROW,L1-INFO,kernel_release_read,内核 4.19.81-00001-g81c7d33 #1 SMP PREEMPT：vulnerabilities 读数面在该版本区间内应当存在
ROW,L1-INFO,cpufreq_governor,scaling_governor=performance：频率档位形态决定计时噪声底与频率量化程度，跨设备比较等级前必须先看这一行
SUMMARY,high=6,warn=16,gap=3,info=9,vuln=8,vuln_bad=5,arch=x86,model=Tablet X2,spl=2023-11-05
```

**这份快照是本报告判读规则的反面标本，它的价值在于三层同时出问题。**

1. **六条 L3 的来源必须先拆开看。** 五条来自 `VULN` 段的 `vuln_vulnerable`（第 19、22、24、29、33 行：`spectre_v2`、`spec_store_bypass`、`mds`、`mmio_stale_data`、`retbleed`），第六条来自 `CMD` 段的 `cmdline_mitigations_off`（第 57 行）。**这两类 L3 不是同一件事**：前者是内核自述"这一类通道本机没有生效中的缓解"，后者是启动参数层"有人显式把缓解整体关掉"。脚本 1 只有这两个码位是 L3（第 21 章），正是因为它们分别对应"结果"与"人为动作"，只有这两种值得打断读者。
2. **`mitigations=off` 与五条 Vulnerable 之间的因果是"可能"，不是"已证"。** 第 45 行的 `CMD,mitigations=,present,mitigations=off` 带出了字面取值（这是 `CMD` 行唯一的四字段项，专为脚本 2 的档位漂移比对保留）。按上游文档，这一档是"按架构聚合的关闭开关"，x86 上会注入 `nopti` 与 `spec_store_bypass_disable=off`。**但这份快照里 `nopti` 与 `spec_store_bypass_disable=` 两个记号仍是 `absent`（第 46、53 行）**——因为 `mitigations=` 的聚合发生在内核内部，cmdline 上看不到被聚合出来的那些串。这是一条容易被误读的线：**"聚合后的开关不在 cmdline 上"不等于"聚合没生效"**；反过来，第 21、22 行 `spec_store_bypass` 自述 Vulnerable 才是聚合生效的旁证。
3. **第 71–72 行的微码锚点与第 25、31 行的"微码未到"记号互相印证，这正是 x86 相对 arm64 唯一的优势。** `POL,microcode,microcode : 0x2ef` 给出设备侧能读到的硅侧版本号；`mds` 与 `mmio_stale_data` 的说明文字里都带 `no microcode`（触发 `vuln_note_microcode`）。合起来能写出的一条结论是：**"内核侧冲刷框架在位、硅侧补丁未到"**——这一格按第十六章的归属矩阵记在 H1（固件包维护方），不记在内核责任。arm64 的样例 A 写不出这一条，因为它连版本号锚点都没有。**注意这条结论仍然没有说出 `0x2ef` 是新是旧**——判定要拿厂商微码版本表比对，本脚本不猜。
4. **第 30 行的 SMT 记号是这台机器上"最近的观测者"的位置。** `mmio_stale_data` 的读数带 `; SMT vulnerable`，触发 `vuln_note_smt`。按第六章的读法，这一串是内核写在同一个文件里的第二个维度：**同一个物理核上的另一个逻辑核处于可泄露状态**。它和 H4 层的四条放宽记号放在一起才构成完整判断——如果这台设备是单线程或者 `nosmt` 档，这条记号的含义会立刻缩小。
5. **十六条 L2 的分布是这份快照真正的信息。** 按层归类：H3（启动参数）一条 `cmdline_disable_token`（第 58 行，`nokaslr`）；H1（硬件前提）六条 `cpu_x86_cap_absent`（第 60–70 行：`ssbd`/`ibrs`/`ibpb`/`stibp`/`md_clear`/`flush_l1d` 全部缺席）；H4（策略与 sysctl）五条（第 74、76、78、80 行的 `selinux_permissive`、`sysctl_paranoid_low`、`sysctl_kptr_open`、`sysctl_dmesg_open`，加上第 87 行 `build_type_nonuser`）；VULN 段的说明文字记号四条（第 20、25、26、31 行）。**其中六条 `cpu_x86_cap_absent` 与五条 Vulnerable 是同一件事的两种写法**（这颗芯没有抑制入口，所以内核只能报 Vulnerable），把它们当成十一条独立风险会高估敞口数量——这是第十六章"归属矩阵按层去重"要防的读法错误。
6. **真正的危险来自后一组的组合。** `Permissive`（只记录不拦截）+ `paranoid=1`（采样面在该读取域内开放）+ `kptr_restrict=0`（内核指针不哈希）+ `dmesg_restrict=0`（内核日志开放）+ `userdebug`（调试类通道默认宽松）——**这五条同时成立时，一个非特权 shell 域进程既能采样、又能拿到符号地址、又能读日志里的时序旁证，而且策略不会拦它**。单看任何一条都可以解释成"这台机器为了开发方便"，五条并列才构成第十三章与第十九章说的"跨层组合窗口"。这也是本报告不给出"单项整改清单"而给出归属矩阵的原因。
7. **`META,bootloader,none` 与第 86 行的 `bootloader_absent` 是同一处的两处记录。** 这份档案没有 `ro.boot.bootloader`，于是 H1 层"修复随固件包到达"这条链在设备侧失去归因锚点——注意这与第 5 点里的微码版本号是**两个不同的锚点**：微码行说明"这颗芯片现在跑的是哪版微码"，bootloader 串说明"这台整机装的是哪个固件镜像"，前者能对微码版本表，后者能对厂商公告。样例 B 只有前者。

### 廿二之三 样例 C：一条高危 + 十一处缺口时，这份快照能说什么

```text
META,device_serial,sc010
META,model,Table T3
META,manufacturer,none
META,board,none
META,hardware,none
META,sdk,34
META,security_patch,2026-06-05
META,build_type,user
META,kernel_release,5.15.123-ab14752355 #1 SMP PREEMPT Thu Aug 21 17:05:00 UTC 2025 aarch64
META,cpu_abilist,none
META,bootloader,none
META,cpuinfo_bytes,0
META,cmdline_bytes,0
META,arch,arm64
STATE,arch,arm64
VULN,spectre_v1,mitigated,Mitigation: __user pointer sanitization
ROW,L1-INFO,vuln_mitigated,spectre_v1：Mitigation: __user pointer sanitization
VULN,spectre_v2,mitigated,Mitigation: CSV2, BHB
ROW,L1-INFO,vuln_mitigated,spectre_v2：Mitigation: CSV2, BHB
ROW,L1-INFO,vuln_bhb_covered_in_v2,spectre_v2 的缓解说明里带 BHB 已覆盖记号（Mitigation: CSV2, BHB）：arm64 上分支历史类面没有独立读数文件，这一串文字就是设备侧唯一的直接证据
VULN,spec_store_bypass,vulnerable,Vulnerable
ROW,L3-HIGH,vuln_vulnerable,spec_store_bypass 自述 Vulnerable：内核认为这一类泄露通道在本机没有生效中的缓解，剩余防线只在策略与应用层
VULN,spectre_bhb,not_affected,Not affected
ROW,L1-INFO,vuln_not_affected,spectre_bhb 自述不受影响：这是本芯片与本内核组合的读数，不外推到同族其它型号
ROW,L1-INFO,vuln_name_outside_abi,spectre_bhb 不在上游 sysfs 漏洞 ABI 名单内：这是厂商内核或补丁集自行导出的读数，其命名与取值口径不在这份报告的可核验范围内；下面的按前缀分组计数仍包含它，消费该统计时要减去 COUNT,vuln_outside_abi
COUNT,vuln_files_total,4
COUNT,vuln_vulnerable,1
COUNT,vuln_mitigated,2
COUNT,vuln_not_affected,1
COUNT,vuln_empty,0
COUNT,vuln_other,0
COUNT,vuln_outside_abi,1
ROW,L0-LOW,arm_microcode_face_absent,arm64 侧没有 x86 那样的微码读数面：CPU 侧修复随厂商固件到达，设备上没有可自证的字段，只能向厂商索取证据
CMD,mitigations=,unread
CMD,nopti,unread
CMD,kpti=0,unread
CMD,nokaslr,unread
CMD,nospectre_v1,unread
CMD,nospectre_v2,unread
CMD,nospectre_bhb,unread
CMD,ssbd=,unread
CMD,spec_store_bypass_disable=,unread
CMD,spectre_v2_user=,unread
CMD,spectre_bhi=,unread
CMD,init_on_free=1,unread
ROW,L0-LOW,cmdline_unread,未采到 /proc/cmdline：启动参数层的全部判定不成立，读不到不等于按默认值运行
CAP,ssbs,unknown
ROW,L0-LOW,cpu_cap_unread,未采到 /proc/cpuinfo：不判定 ssbs 一类硬件前提
CAP,sb,unknown
ROW,L0-LOW,cpu_cap_unread,未采到 /proc/cpuinfo：不判定 sb 一类硬件前提
CAP,wfxt,unknown
ROW,L0-LOW,cpu_cap_unread,未采到 /proc/cpuinfo：不判定 wfxt 一类硬件前提
CAP,afp,unknown
ROW,L0-LOW,cpu_cap_unread,未采到 /proc/cpuinfo：不判定 afp 一类硬件前提
CAP,gcs,unknown
ROW,L0-LOW,cpu_cap_unread,未采到 /proc/cpuinfo：不判定 gcs 一类硬件前提
CAP,aes,unknown
POL,selinux,Enforcing
ROW,L1-INFO,selinux_enforcing,SELinux=Enforcing：域级限制生效，perf 与 /proc 侧观察通道的收敛由策略域决定
SYSCTL,kernel.perf_event_paranoid,unread
ROW,L0-LOW,sysctl_paranoid_unread,kernel.perf_event_paranoid 未采到：发行构建上该节点常不导出，读不到时不能按上游默认值判定
SYSCTL,kernel.kptr_restrict,1
ROW,L1-INFO,sysctl_kptr_hash,kernel.kptr_restrict=1：内核地址被遮蔽或哈希
SYSCTL,kernel.dmesg_restrict,1
ROW,L1-INFO,sysctl_dmesg_restricted,kernel.dmesg_restrict=1：内核日志读取被限制
POL,hidepid,off
ROW,L1-INFO,proc_hidepid_off,/proc 未带 hidepid：Android 上跨 UID 可见性主要由 SELinux 与 proc 权限决定，缺这一项不判成敞口
COUNT,zygote_processes,1
ROW,L1-INFO,zygote_shared,采到 1 个 zygote 进程：应用进程由共享母进程孵化，预加载类与代码缓存的初始缓存状态在同一台机器上是公共的
ROW,L0-LOW,neighbor_app_domain_gap,本次采集在 shell 域：普通应用域能否观察到同样这些通道，本快照不覆盖，需按应用侧实验另立证据
ROW,L0-LOW,bootloader_absent,未采到 ro.boot.bootloader：CPU 侧修复随厂商固件包到达时，这是设备侧唯一的固件版本锚点，缺失即不可归因
ROW,L1-INFO,build_type_user,ro.build.type=user：发行构建，调试类观察通道按平台默认收敛
ROW,L1-INFO,spl_read,security_patch=2026-06-05：只作时间锚点；SPL 不表示微码或内核缓解项是否生效，这两件事各有独立证据
ROW,L1-INFO,kernel_release_read,内核 5.15.123-ab14752355 #1 SMP PREEMPT Thu Aug 21 17:05:00 UTC 2025 aarch64：vulnerabilities 读数面在该版本区间内应当存在
ROW,L0-LOW,cpufreq_unread,未采到 cpufreq governor：DVFS 状态未知，计时侧信道的信噪比条件无法归一化
SUMMARY,high=1,warn=0,gap=11,info=13,vuln=4,vuln_bad=1,arch=arm64,model=Table T3,spl=2026-06-05
```

**这份样例专门用来练"缺口读法"，因为它的 `gap=11` 比 `high=1` 更能说明问题。**

1. **先看架构是怎么判出来的。** `cpuinfo_bytes=0`（第 12 行）说明 `/proc/cpuinfo` 没读到，但 `META,arch,arm64` 仍然成立——它来自第 9 行的内核版本串结尾那个 `aarch64`（第 21 章的三级判定顺序：cpuinfo → uname → abilist；这里第一级失败、第二级命中）。**这一条很重要：架构判定成立，不代表架构相关的所有读数面都可读。** 第 47–57 行的六个 `CAP` 全部是 `unknown` 而不是 `absent`——按第十一章的口径，`unknown` 是"没读到所以不判定"，`absent` 才是"读到了且这个位不在"。把这份快照读成"这台设备没有 ssbs"是一个纯粹的采集退化误读，而脚本用两个不同的值把这两种情形隔开了。
2. **十一处缺口的构成（逐项点名）。** `arm_microcode_face_absent`（第 33 行，结构性，不可补）、`cmdline_unread`（第 46 行，一条覆盖整个 `CMD` 段的十二行 `unread`）、`cpu_cap_unread` ×5（第 48–56 行，逐位记录）、`sysctl_paranoid_unread`（第 61 行）、`bootloader_absent`（第 71 行）、`cpufreq_unread`（第 75 行）、`neighbor_app_domain_gap`（第 70 行，域边界，不是采集失败）。**这里有一个必须维持的区分：`cmdline_unread` 是采集失败，`CAP=unknown` 是判定不确定**，两者都记缺口，但补的方式不同——前者重跑采集即可，后者要先解决读取权限。
3. **唯一的 L3 落在一个只有四个文件的可读面上。** 第 21–22 行：`spec_store_bypass` 自述 `Vulnerable`。这份快照的可读读数面只剩 4 个文件（`COUNT,vuln_files_total,4`，样例 A 是 5、样例 B 是 8），而第 18 行的 `spectre_v2` 报 `Mitigation: CSV2, BHB`。**所以这份快照能支持的结论是"存储绕过抑制这一项在本机自述没有生效中的缓解，且这一条与 `spectre_v2` 的缓解状态不矛盾"**——两件事各有独立实现，不是同一个开关。它**不能**支持的结论是"这台设备的其余推测执行类面也敞着"，因为那些文件在本快照里根本没有读数。
4. **`spectre_bhb` 在这里读 `Not affected`（第 23–25 行），而样例 A 的同一个名字读 `Mitigation: Loop`。** 这两份档案架构相同、名字相同、取值口径不同，正是第七章"厂商自行导出的文件名与取值不在可核验范围内"的实例：`vuln_name_outside_abi`（第 25 行）在这里再次触发，提醒读者**不要拿这两个 `spectre_bhb` 的读数做一次"状态变好了"的比对**。按第九章 M2 与第七章的纪律，这类名字的比较只在同一台设备、同一份采集流程内有效。
5. **第 75 行的 `cpufreq_unread` 看起来与侧信道无关，实际是本报告特意保留的一条归一化锚点。** 样例 B 的 `scaling_governor=performance` 意味着频率固定在高档，计时噪声底与频率量化程度都跟样例 A 的 `schedutil` 不同。**如果哪天有人拿这三份快照去做"计时侧信道难度"的横向比较，这一行是唯一能让比较作废或成立的信息。** 第十二章把它列为必读项，脚本把它放在最后一行，位置本身就是一种提示：读数之前先看噪声条件。
6. **这份快照该配什么动作。** 按第二十章的契约，`gap=11` 且唯一 L3 来自一个 4 文件的窄面，正确动作不是"派工修 `spec_store_bypass`"，而是**先重采**：cmdline、cpuinfo、paranoid 三处失败是同一类权限/截断问题的三个症状，补齐之后这份快照才有资格进入差分基线。脚本 2 在第廿四章会把这种"采集面收窄"单独记成 `snapshot_thinner` 与 `risk_gone_by_coverage`，而不是让它伪装成整改成果。

### 廿二之四 三份样例的横向核对

| 维度 | A（sc001） | B（sc002） | C（sc010） | 读法约束 |
| --- | --- | --- | --- | --- |
| 架构判定来源 | cpuinfo | cpuinfo | uname（cpuinfo 未读到） | 判定成立 ≠ 读数面可读 |
| `VULN` 文件数 | 5（2 个非 ABI 名） | 8（0 个非 ABI 名） | 4（1 个非 ABI 名） | 跨份比较先做 ABI 减法 |
| L3 条数 | 0 | 6（5 结果 + 1 动作） | 1 | 结果类与动作类不可合并计数 |
| L2 条数 | 0 | 16 | 0 | 六条特性位与五条 Vulnerable 同源，需去重 |
| L0 缺口 | 2（结构性 1 + 域边界 1） | 3（结构性 1 + 采集 1 + 域边界 1） | 11（9 采集 + 结构性 1 + 域边界 1） | 采集缺口可补，结构性缺口只能换证据源 |
| 微码锚点 | 无（arm64 不导出） | 有（`microcode : 0x2ef`） | 无（arm64 + cpuinfo 未读到） | 只有 x86 能在设备侧自证 H1 |
| `hidepid` | on | off | off | Android 上属厂商自选，不作达标项 |
| governor | schedutil | performance | 未采到 | 跨份做计时类比较前必须先看这一行 |

最后一行不是凑数：**这三份快照的 `scaling_governor` 各不相同或不可读，因此它们之间任何"计时侧信道难度"的横向比较都缺乏归一化条件**。这也是本报告的三个样例只用来演示**判读规则**、不用来演示"哪台设备更安全"的根本原因——第二十章的契约里"等级不跨设备、不跨架构、不跨采集面比较"这一条，在这三份产物上同时成立。
## 廿三、脚本 2 的设计：把"没有变化"限定在剩下的可读项里

脚本 2 的输入是**两份脚本 1 的输出文本**，不是设备。它不执行 `adb`、不访问网络、不读任何 `/proc` 或 `/sys` 节点、不运行任何计时或缓存探针——**它是一次纯离线的文本比对**，可以在采集之后的任何时间、任何机器上重跑。这个设计选择是这份报告里最重要的一条纪律：**漂移判定必须与采集解耦**，否则每次重判都要重新接触设备，而重新接触设备本身就改变了"这两份产物是否可比"这个前提。

### 廿三之一 目标与非目标

| 目标 | 非目标 |
| --- | --- |
| 把两份快照的**同一字段**并列，输出 `DELTA,<段>,<键>,<基线值>,<当前值>` | 判断"这台设备安不安全"——单份快照都不判，两份更不会 |
| 区分"值变了""文件新增""文件消失"三种形态，各给不同码位 | 把"文件消失"读成"已修复"（这是本章的核心禁区） |
| 区分**档位漂移**与**可读性漂移**：`2 → 1` 是收紧或放宽，`2 → unread` 是缺口 | 用数值大小替代方向判定：`kptr_restrict` 的 0/1/2 不是单调刻度，脚本对它只按"是否 0"分级 |
| 对**信号码集合**做差集，找出新增与消失的判定项 | 把差集当成整改台账——差集里每一行都要另找变更依据 |
| 在跨设备、跨架构、跨构建形态时**显式降低整份差分的解释力** | 阻止比对：脚本仍然跑完，只是把结论口径改掉 |

### 廿三之二 十一个取值与分组辅助函数

脚本 2 的全部解析能力集中在开头十行辅助函数里。它们的存在决定了后面每个分支的语义边界：

| 函数 | 作用 | 为什么不能省 |
| --- | --- | --- |
| `mf <文件> <键>` | 取 `META,<键>,<值>` 的第 3 字段 | `META,cpu_abilist` 本身含逗号，只能取第 3 字段而不能取"行尾" |
| `lf3 <文件> <段> <键>` | 取 `<段>,<键>,<值>` 的第 3 字段 | 读数面统一的定位方式；`CMD,mitigations=` 的第 4 字段不会被误读进来 |
| `lf4 <文件> <段> <键>` | 取第 4 字段**及其后所有逗号** | 内核版本串与 `Mitigation: CSV2, BHB` 这类含逗号的值必须整体取回，否则比对会截断 |
| `nn <值>` | 把 `''`/`none`/`unread`/`unknown` 归一为空 | 让"没采到"在条件判断里变成可判定的空串，同时不把它和真实值混淆 |
| `nnc <值>` | 只归一 `''`/`none`，保留 `unread`/`unknown` | 架构字段的 `unknown` 是**判定失败的结论**，不是缺采，两者要分开处理 |
| `readable <值>` | `''`/`none`/`unread`/`unknown` 返回 1，其余 0 | 所有"由可读变为不可读"的分支都靠它做前置门，避免把 `absent` 当成不可读 |
| `cntv <文件> <键>` | 取 `COUNT,<键>` 数值 | 缺席时返回空串，从而与"计数为 0"区分开 |
| `vnames` / `ckey` / `pkey` / `skey` | 取某一段的键名集合（去重排序） | 逐项比对必须遍历**两侧键的并集**，否则新增与消失都看不见 |
| `rowsig <文件> <前缀>` | 取某一等级前缀下出现过的信号码集合 | 这是"信号码集合差分"的输入，也就是本章第六节两种"消失"判定的根 |
| `tight_rank <键> <值>` | 把 sysctl 值映射成"收敛程度"刻度 | `perf_event_paranoid` 用原值（越大越收敛），其余三项压成 0/1 二档——**因为它们不是单调刻度** |

`tight_rank` 这一行值得单独说明。`kernel.kptr_restrict` 有三个合法值（0 不哈希、1 哈希、2 全遮蔽），1 和 2 都属"遮蔽"，把它们排成 1 < 2 会让 `2 → 1` 被读成一次放宽——而这条放宽在观察通道口径下没有实际意义（哈希与全遮蔽对非特权域都不可读）。脚本因此对这三项只做**是否等于 0** 的二档判定，只有 `perf_event_paranoid` 保留数值刻度。这是"档位漂移"必须按**语义**而不是按**数值**排序的具体落地，也是第二十章契约里"方向判定要有公开依据"的执行方式。

### 廿三之三 判定顺序：先确立可比性，再谈差异

脚本 2 的段落顺序是刻意安排的，而且第一段就可以让后面所有段落失去解释力：

1. **`PAIR` 行先记录两份产物各自有多少条判定行**（`rows=`），这是后面 `snapshot_thinner` / `snapshot_richer` 的基线，也让人一眼看到输入是不是空文件。若两侧都没有 `ROW,` 行，脚本直接以 `ERR,input_no_snapshot_row` 退出——**不给一份无法确认是快照的文本做任何判定**。
2. **同源性（机型、构建形态、内核版本、SPL）**。机型不同 → `pair_identity_mismatch`（L2），并在文字里写明"任何差异都不能读成一次收紧或劣化"；机型相同 → `pair_identity_same_model`（L1），提醒仍需操作者确认同一台设备、同一条采集流程、同一采集域。`NOTE,identity` 那一行是这一节的诚实声明：**快照里没有采集时间戳与构建号，同源性无法由脚本证明，只能由操作者的记录证明。**
3. **构建形态变化**（`build_type_changed`，L2）。非发行构建与发行构建不是同一条基线——这一条不是修辞：`userdebug` 上 `paranoid` 的默认值、trace 通道的开放范围都由构建形态决定（第十一章），跨构建形态比 `paranoid` 数值是在比两台机器。
4. **架构判定漂移**。四种迁移各有码位：判不出 → 判得出（`arch_gained`，L1）、判得出 → 判不出（`arch_lost`，L0）、同族变化（不输出）、跨族变化（`arch_changed`，L2）。放在 VULN 逐项之前是必要的：**x86 与 arm64 的 vulnerabilities 文件集与缓解形态不等价（第七章），架构变了还按"同名文件逐项比"会造出一批假的敞口。**
5. **SPL 只按"前进/后退"两档读**（`spl_moved_forward` L1 / `spl_moved_backward` L2），且两侧都必须能解析成 `YYYYMMDD` 形式（`${spl//-/}` 比较）；任一为空则退到 `spl_unread_both`（L0）。脚本在这里保持第一章的纪律：**SPL 变化不说明微架构缓解项是否生效**，所以前进那一档的文字仍然是"仍要逐项看 VULN 读数"。

### 廿三之四 VULN 段：三种迁移方向，两类不可分类

逐项遍历两侧键的并集，先分**存在性**再分**取值**：

- 基线无、当前有 → `NEW,VULN` + `vuln_file_new`（L1）。文字明确拒绝读成改善：**"无法判断是内核新增了读数还是基线采集被截断"**。
- 基线有、当前无 → `GONE,VULN` + `vuln_file_gone`（L0）。这是本章最重要的一行：**"文件不导出不等于该面被修复"**。
- 两侧都有 → 先无条件打印 `DELTA,VULN,<name>,<基线状态>,<当前状态>`（即使两侧相同也打印，让"逐项可比且无变化"这件事本身留下证据），再按状态迁移分流：
  - 状态同、说明文字变 → `vuln_mitigation_text_changed`（L1）。**缓解实现细节常常就写在这段文字里**，状态字段不变不代表什么都没动。
  - `vulnerable → mitigated|not_affected`、`not_affected → mitigated` → `vuln_state_better_needs_proof`（L1）：方向上是变好了，但要证明是补丁到位而不是启动参数被改动或读数口径变了，需与脚本一的 cmdline 一节和厂商公告对齐。
  - 任何状态 → `vulnerable` → `vuln_state_worse`（**L3**）：先查 cmdline 是否被加了关闭开关，再查内核是否换档。
  - 任何状态 → 空值 → `vuln_state_emptied`（L2）：**这既不是敞口也不是缓解，按缺口处理**。
  - 空值 → 任何状态 → `vuln_state_now_readable`（L1）：两侧证据强度不等价，基线的空值同样是缺口。
  - 其余迁移 → `vuln_state_unclassified_move`（L1）：**脚本拒绝给方向**，把两个原始取值交人工复核。这一档不是偷懒——它防的正是"把所有变化都读成变坏"或"都读成变好"。样例里 `mitigated → not_affected` 就落在这里（第廿四章样例一会逐条讨论这个选择的后果）。

之后是两条**聚合计数**漂移：`vuln_files`（文件个数）与 `vuln_vulnerable`（自述敞口个数）。前者减少记 `vuln_files_fewer`（L2，"收窄本身不是改善"），后者增加记 `vuln_open_count_up`（**L3**），减少记 `vuln_open_count_down_needs_proof`（L1）。**计数行与逐项行故意分开输出**：一份差分如果只有计数行变化而逐项行没有，说明变化发生在可比面之外，这类"总量变了、明细没变"的情形只能重采。

### 廿三之五 CMD / CAP / POL / SYSCTL：档位漂移与可读性漂移分开

这四段共用同一个模板（遍历并集 → 比较 → 分流），但每段的方向语义不同：

- **`CMD`**：先判 cmdline 整段的可读性迁移（`cmdline_now_readable` L1 / `cmdline_became_unread` L0），可读时才逐记号比。**`mitigations=` 的比较键不是 `present/absent` 标志，而是 cmdline 上的字面取值**（`CMD` 行的第 4 字段）——两侧都写了 `mitigations=` 但一个是 `auto` 一个是 `off`，是这次差分里最严重的一类变化，只看标志位会把它漏掉。三种情形各有码位：新增（`cmdline_mitigations_off_appeared`，**L3**）、移除（`cmdline_mitigations_off_removed`，L1）、档位取值变化（`cmdline_mitigations_value_changed`，L2，并且文字里声明"本脚本不解释具体档位"）。其余关闭类记号只按"新增/移除"两档给码（`cmdline_disable_token_added` L2 / `cmdline_disable_token_removed` L1），`init_on_free=1` 反向（新增 L1、移除 L2）。
- **`CAP`**：先判可读性（`cpu_cap_became_unread` L0 / `cpu_cap_now_readable` L1），两侧可读时才比 `present/absent`。**这里有一个不讨喜但必要的判断**：`present → absent` 记 L2（`cpu_cap_lost`）而不是 L3，理由是"同一颗芯片的特性位不会因升级消失"——它更可能来自采集域或内核打印格式变化；若确为不同芯片，则整份差分已由第二节的同源性判定降级。**硬件能力位的漂移在本报告里首先是一个采集问题。**
- **`POL,selinux`**：六种迁移六种码，且只有一档不给"方向"（`selinux_state_changed`，L2，取值不属于 Enforcing/Permissive 口径）。两条 L3 都在"收敛被撤掉"的方向上：`Enforcing → Permissive`（`selinux_loosened`）与任何值 → `Disabled`（`selinux_disabled_now`）。后者附带一句**"这是本差分里优先级最高的一行"**，因为策略一关，本快照里所有依赖策略域的收敛判定同时失效——这与脚本 1 里同样的码位配对使用。
- **`POL,hidepid`**：`on → off` 记 L2（`proc_hidepid_lost`），但文字同时写明"Android 上这一项属厂商自选"，所以它是"要问改动来源"而不是"要整改"。`off → on` 记 L1 并附"不能作为基线要求的达标项"。
- **`SYSCTL`**：按 `tight_rank` 的方向给 `sysctl_loosened`（L2）/ `sysctl_tightened`（L1），不可读时优先落到 `sysctl_became_unread`（L0）。**文字里那句"shell 域可读不等于应用域可读"是从第十一章搬过来的**，它限制了这一档能被写成什么结论。
- **`POL,microcode`**（x86）：只在两侧都可读时给 `microcode_changed`（L2）；变成不可读记 `microcode_evidence_lost`（L0）并直说"设备侧失去证明微码缓解到达的唯一锚点，只能向厂商索取证据"。方向一律不判：**新旧要按厂商微码版本表，脚本没有这张表。**

### 廿三之六 信号码集合差分：两种"消失"，一种"新增"

这是脚本 2 与"简单 diff"真正分开的地方。前面各段比的是**字段值**；这一节比的是**判定本身**：把两侧 `ROW,` 行的信号码按等级取集合，做差集。

- **新增 L3 / L2 码** → `drift_new_high` / `drift_new_warn`（同等级转发，L3 与 L2 各一条码位）。这是"相对基线新出现了某类信号"的机器可读记号，配合上面的 `DELTA,` 行逐项定位。
- **消失的 L3 / L2 码**：脚本先查**产生这个码的采集面在当前快照里是否还可读**（用一段按码名前缀选的探针：`vuln_*` 查 `VULN,` 行数、`cmdline_*` 查 CMD 段可读、`cpu_*` 查 `ssbs`/`ibrs` 位、`sysctl_*` 查 SYSCTL 键集合、其余按可读处理），然后二选一：
  - 采集面同时不可读 → `risk_gone_by_coverage`（**L1，不是好消息**）：**"这是覆盖收窄造成的消失，不能记成风险已闭合"**。
  - 采集面仍可读 → `risk_gone_needs_proof`（L2）：**"消失有两种解释——问题确实被处理，或判定条件不再触发"**，需要公告、版本号或重采这类独立证据才能记成已修。
- **新增缺口（L0 码集合差）** → 合并成一条 `drift_new_gaps`（L0），把这些码名列出来当作"下一轮优先补"的清单；**缺口消失** → `drift_gap_closed`（L1），文字是"采集面被补上，这一轮的结论可比范围扩大"——注意这条也只是可比范围变大，不是风险变小。

这一节的输出形态决定了它只能作为**运营指标**使用（第二十六章 K6/K7 的分子分母），不能作为工单来源：一条 `risk_gone_needs_proof` 的正确后续动作是"去找一条变更依据"，而不是"关闭一张旧工单"。

### 廿三之七 快照厚度与 SUMMARY

最后一段把两份快照的 `ROW,` 行数直接比较：变少记 `snapshot_thinner`（L2），文字是**"本差分的'没有变化'只覆盖剩下的这些项"**；变多记 `snapshot_richer`（L1），提醒新增项多为首次可读、没有基线可比，下一轮才能形成趋势。这一行看着平淡，实际是整份差分的适用范围声明——**没有它，"SUMMARY 里 high=0" 很容易被读成"这一轮没有新增高危"，而真实含义只是"剩下的可比项里没有新增高危"。**

`SUMMARY,high=,warn=,gap=,info=` 统计的是**脚本 2 自己输出的判定行**，与两份快照各自的 SUMMARY 数字没有可加关系（同一条敞口在两份快照里各计一次，在差分里可能一次也不计）。这一条常被误用，第二十章的输出契约里单独写了它。

### 廿三之八 错误路径与输出契约

| 行前缀 | 字段 | 用途 |
| --- | --- | --- |
| `PAIR,base\|curr,<路径>,rows=<n>` | 3 | 输入识别与判定行数 |
| `NOTE,identity,<文字>` | 2 | 不可由脚本证明的前提，显式交回操作者 |
| `DELTA,<段>,<键>,<基线>[,<当前>]` | ≥4 | 字段级并列，含"无变化"项 |
| `NEW,<段\|等级>,<键\|码>` / `GONE,...` | 3 | 存在性变化；`GONE` 恒为不可改善读的一方 |
| `ROW,<等级>,<码>,<文字>` | 4 | 判定行，与脚本 1 同构，`rowsig` 据此取集合 |
| `SUMMARY,high=,warn=,gap=,info=` | 5 | 本轮差分自身的分布 |

`set -u` 全程开启。三条错误路径都只写 stderr 并以 1 退出，**不产出半份差分**：`ERR,input_unreadable,<路径>`、`ERR,input_empty,<路径>`、`ERR,input_no_snapshot_row,<基线>|<当前>`。与脚本 1 相同的纪律：残缺必须是显式的，不能以"看起来完整的空差分"形态出现。

### 廿三之九 完整清单

```bash
#!/usr/bin/env bash
# 脚本 2: side-channel-snapshot-drift.sh — 两次微架构侧信道姿态快照的离线漂移比对（只读）
# 用法: side-channel-snapshot-drift.sh <基线快照> <当前快照>
# 输入是脚本 1 的输出文本；本脚本不接触设备、不访问网络、不运行任何计时或缓存探针
set -u
base="${1:?用法: side-channel-snapshot-drift.sh <基线快照> <当前快照>}"
curr="${2:?缺少当前快照路径}"
[ -r "$base" ] || { echo "ERR,input_unreadable,$base" >&2; exit 1; }
[ -r "$curr" ] || { echo "ERR,input_unreadable,$curr" >&2; exit 1; }
[ -s "$base" ] || { echo "ERR,input_empty,$base" >&2; exit 1; }
[ -s "$curr" ] || { echo "ERR,input_empty,$curr" >&2; exit 1; }

hi=0; warn=0; gap=0; info=0
row() { # row <等级> <信号码> <格式串> [参数…]
  local lv="$1" code="$2" fmt="$3"; shift 3
  local txt
  if [ "$#" -gt 0 ]; then txt="$(printf "$fmt" "$@")"; else txt="$fmt"; fi
  printf 'ROW,%s,%s,%s\n' "$lv" "$code" "$txt"
  case "$lv" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) warn=$(( warn + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
mf()   { awk -F, -v k="$2" '$1 == "META" && $2 == k { print $3; exit }' "$1"; }
nn()   { case "$1" in ''|none|unread|unknown) printf '' ;; *) printf '%s\n' "$1" ;; esac; }
lf3()  { awk -F, -v t="$2" -v k="$3" '$1 == t && $2 == k { print $3; exit }' "$1"; }
lf4()  { awk -F, -v t="$2" -v k="$3" '$1 == t && $2 == k { out = ""; for (i = 4; i <= NF; i++) out = out (i > 4 ? "," : "") $i; print out; exit }' "$1"; }
cntv() { awk -F, -v k="$2" '$1 == "COUNT" && $2 == k { print $3; exit }' "$1"; }
vnames() { awk -F, '$1 == "VULN" { print $2 }' "$1" | sort -u; }
ckey()   { awk -F, '$1 == "CMD" { print $2 }' "$1" | sort -u; }
pkey()   { awk -F, '$1 == "CAP" { print $2 }' "$1" | sort -u; }
skey()   { awk -F, '$1 == "SYSCTL" { print $2 }' "$1" | sort -u; }
rowsig() { awk -F, -v p="$2" '$1 == "ROW" && $2 ~ "^" p { print $3 }' "$1" | sort -u; }
nnc()    { case "$1" in ''|none) echo none ;; *) echo "$1" ;; esac; }
readable() { case "$1" in ''|none|unread|unknown) return 1 ;; *) return 0 ;; esac; }

b_rows="$(grep -c '^ROW,' "$base" || true)"; c_rows="$(grep -c '^ROW,' "$curr" || true)"
if [ "${b_rows:-0}" -eq 0 ] && [ "${c_rows:-0}" -eq 0 ]; then
  echo "ERR,input_no_snapshot_row,$base|$curr" >&2
  exit 1
fi
printf 'PAIR,base,%s,rows=%s\n' "$base" "${b_rows:-0}"
printf 'PAIR,curr,%s,rows=%s\n' "$curr" "${c_rows:-0}"

b_model="$(nn "$(mf "$base" model)")"; c_model="$(nn "$(mf "$curr" model)")"
b_spl="$(nn "$(mf "$base" security_patch)")"; c_spl="$(nn "$(mf "$curr" security_patch)")"
b_krel="$(nn "$(mf "$base" kernel_release)")"; c_krel="$(nn "$(mf "$curr" kernel_release)")"
b_type="$(nn "$(mf "$base" build_type)")"; c_type="$(nn "$(mf "$curr" build_type)")"
b_arch="$(nnc "$(lf3 "$base" STATE arch)")"; c_arch="$(nnc "$(lf3 "$curr" STATE arch)")"

# ---------- 同源性：两份产物是不是同一台设备 ----------
if [ -z "$b_model" ] || [ -z "$c_model" ]; then
  row L0-LOW pair_identity_unread "至少一份快照没有机型标识：无法确认这两份产物来自同一台设备，差分结果只做参考"
elif [ "$b_model" != "$c_model" ]; then
  row L2-WARN pair_identity_mismatch "基线机型 $b_model 与当前机型 $c_model 不同：本差分跨设备，任何差异都不能读成一次收紧或劣化"
else
  row L1-INFO pair_identity_same_model "两份快照机型一致（$b_model）：仍要由操作者确认它们来自同一台设备、同一条采集流程与同一采集域"
fi
printf 'NOTE,identity,快照内不含采集时间戳与构建号，同源性由操作者自行记录\n'
if [ "$b_type" != "$c_type" ]; then
  printf 'DELTA,build_type,%s,%s\n' "${b_type:-none}" "${c_type:-none}"
  row L2-WARN build_type_changed "构建形态由 ${b_type:-未采到} 变为 ${c_type:-未采到}：观察通道的默认收敛随构建形态改变，非发行构建与发行构建不是同一条基线"
fi
if [ -n "$b_krel" ] && [ -n "$c_krel" ] && [ "$b_krel" != "$c_krel" ]; then
  printf 'DELTA,kernel_release,%s,%s\n' "$b_krel" "$c_krel"
  row L1-INFO kernel_changed "内核版本串变化（$b_krel → $c_krel）：读数词表、缓解形态与默认档位都可能随之整体换档，逐项差异要放在这个前提下读"
fi
if [ -z "$b_spl" ] && [ -z "$c_spl" ]; then
  row L0-LOW spl_unread_both "两份快照都没有安全补丁级别：本差分不做任何补丁到达性推断"
elif [ -n "$b_spl" ] && [ -n "$c_spl" ] && [ "$b_spl" != "$c_spl" ]; then
  case "$b_spl:$c_spl" in
    *:*) if [ "${c_spl//-/}" -gt "${b_spl//-/}" ] 2>/dev/null; then
           row L1-INFO spl_moved_forward "security_patch 由 $b_spl 前进到 $c_spl：SPL 变化不说明微架构缓解项是否生效，仍要逐项看 VULN 读数"
         else
           row L2-WARN spl_moved_backward "security_patch 由 $b_spl 退到 $c_spl：补丁级别回退，本差分的其余读数都按回退后的镜像重读"
         fi ;;
  esac
fi

# ---------- 架构判定漂移：决定后面哪些缺席项有意义 ----------
case "$(nnc "$b_arch"):$(nnc "$c_arch")" in
  unknown:unknown) : ;;
  unknown:*) row L1-INFO arch_gained "架构判定由判不出变为 $c_arch：基线里那些'某文件不存在'的缺口现在才有了解释口径，需重跑基线才能对比缺席项" ;;
  *:unknown) row L0-LOW arch_lost "架构判定由 $b_arch 变为判不出：当前快照无法区分'这个平台不导出该项'与'采集被截断'，缺席项一节全部留为缺口" ;;
  *) [ "$b_arch" != "$c_arch" ] && row L2-WARN arch_changed "架构由 $b_arch 变为 $c_arch：两份快照的读数词表本就不同（x86 与 arm64 的 vulnerabilities 文件集与缓解形态不等价），逐项差异不能按同族升级解读" ;;
esac

# ---------- VULN 逐项状态迁移 ----------
b_vn="$(vnames "$base")"; c_vn="$(vnames "$curr")"
if [ -z "$b_vn" ] && [ -z "$c_vn" ]; then
  row L0-LOW vuln_pair_dir_gap "两份快照都没有 VULN 读数行：推测执行类缓解状态在本差分里完全不可见，不得读成两侧一致"
else
  for name in $(printf '%s\n' "$b_vn" "$c_vn" | grep '[^[:space:]]' | sort -u); do
    bs="$(lf3 "$base" VULN "$name")"; cs="$(lf3 "$curr" VULN "$name")"
    bt="$(lf4 "$base" VULN "$name")"; ct="$(lf4 "$curr" VULN "$name")"
    if [ -z "$bs" ]; then
      printf 'NEW,VULN,%s,%s\n' "$name" "$cs"
      row L1-INFO vuln_file_new "$name 在当前快照新增（$cs）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断"
      continue
    fi
    if [ -z "$cs" ]; then
      printf 'GONE,VULN,%s,%s\n' "$name" "$bs"
      row L0-LOW vuln_file_gone "$name 在当前快照消失（基线为 $bs）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了"
      continue
    fi
    printf 'DELTA,VULN,%s,%s,%s\n' "$name" "$bs" "$cs"
    [ "$bs" = "$cs" ] && [ "$bt" = "$ct" ] && continue
    if [ "$bs" = "$cs" ]; then
      row L1-INFO vuln_mitigation_text_changed "$name 的档位未变但说明文字变了：$bt → $ct；缓解实现细节的变化常常就写在这段文字里，需要按内核版本核对含义"
      continue
    fi
    case "$bs:$cs" in
      vulnerable:mitigated|vulnerable:not_affected|not_affected:mitigated)
        row L1-INFO vuln_state_better_needs_proof "$name 由 $bs 变为 $cs：方向上是变好了，但要证明是补丁到位而不是启动参数被改动或读数口径变了，需与脚本一的 cmdline 一节和厂商公告对齐" ;;
      *:vulnerable)
        row L3-HIGH vuln_state_worse "$name 由 $bs 变为 Vulnerable：这一类泄露通道在本机自述没有生效中的缓解；先查 cmdline 是否被加了关闭开关，再查内核是否换档" ;;
      *:empty)
        row L2-WARN vuln_state_emptied "$name 由 $bs 变为空值：读数面存在但读不到内容，这既不是敞口也不是缓解，按缺口处理" ;;
      empty:*)
        row L1-INFO vuln_state_now_readable "$name 由空值变为 $cs：当前可读了；基线的空值同样是缺口，两侧不是同一证据强度" ;;
      *)
        row L1-INFO vuln_state_unclassified_move "$name 由 $bs 变为 $cs：这一迁移不属于本脚本枚举的三类方向，按原始读数交人工复核（$bt → $ct）" ;;
    esac
  done
fi

# ---------- VULN 文件集与敞口计数 ----------
b_files="$(printf '%s\n' "$b_vn" | grep -c '[^[:space:]]' || true)"
c_files="$(printf '%s\n' "$c_vn" | grep -c '[^[:space:]]' || true)"
b_bad="$(cntv "$base" vuln_vulnerable)"; c_bad="$(cntv "$curr" vuln_vulnerable)"
if [ -n "$b_files" ] && [ -n "$c_files" ] && [ "$b_files" != "$c_files" ]; then
  printf 'DELTA,vuln_files,%s,%s\n' "$b_files" "$c_files"
  if [ "$c_files" -lt "$b_files" ]; then
    row L2-WARN vuln_files_fewer "vulnerabilities 读数文件由 ${b_files} 个减少到 ${c_files} 个：可读面收窄，本次差分能确认的项随之减少，收窄本身不是改善"
  else
    row L1-INFO vuln_files_more "vulnerabilities 读数文件由 ${b_files} 个增加到 ${c_files} 个：新增项多为内核新版本导出的新读数，首次出现时没有基线可比"
  fi
fi
if [ -n "$b_bad" ] && [ -n "$c_bad" ] && [ "$b_bad" != "$c_bad" ]; then
  printf 'DELTA,vuln_vulnerable,%s,%s\n' "$b_bad" "$c_bad"
  if [ "$c_bad" -gt "$b_bad" ]; then
    row L3-HIGH vuln_open_count_up "自述 Vulnerable 的读数项由 $b_bad 增至 $c_bad：敞口项数量上升，配合逐项 DELTA 行定位是哪一类通道新开放"
  else
    row L1-INFO vuln_open_count_down_needs_proof "自述 Vulnerable 的读数项由 $b_bad 降至 $c_bad：数量下降可能来自补丁、来自 cmdline 改动，也可能来自读数口径变化，本脚本不区分"
  fi
fi

# ---------- CMD：启动参数层的缓解开关 ----------
b_cmd="$(ckey "$base")"; c_cmd="$(ckey "$curr")"
b_cmdany="$(lf3 "$base" CMD 'mitigations=')"; c_cmdany="$(lf3 "$curr" CMD 'mitigations=')"
if [ "$b_cmdany" = unread ] && [ "$c_cmdany" != unread ]; then
  row L1-INFO cmdline_now_readable "/proc/cmdline 由读不到变为可读：新增的启动参数证据，需重跑基线才有可比性"
elif [ "$b_cmdany" != unread ] && [ "$c_cmdany" = unread ]; then
  row L0-LOW cmdline_became_unread "/proc/cmdline 由可读变为读不到：启动参数层的漂移全部不可判定，这一档缺口要在下一轮采集补上"
elif [ "$b_cmdany" != unread ] && [ "$c_cmdany" != unread ]; then
  for tok in $(printf '%s\n' "$b_cmd" "$c_cmd" | grep '[^[:space:]]' | sort -u); do
    bv="$(lf3 "$base" CMD "$tok")"; cv="$(lf3 "$curr" CMD "$tok")"
    # mitigations= 两侧都 present 不代表档位相同：这一项的比较键要换成 cmdline 上的字面取值
    bk="$bv"; ck="$cv"
    if [ "$tok" = "mitigations=" ]; then
      bmv="$(lf4 "$base" CMD mitigations=)"; cmv="$(lf4 "$curr" CMD mitigations=)"
      bk="$bv:$bmv"; ck="$cv:$cmv"
    fi
    [ "$bk" = "$ck" ] && continue
    printf 'DELTA,CMD,%s,%s,%s\n' "$tok" "${bv:-none}" "${cv:-none}"
    case "$tok" in
      mitigations=)
        case "$bv:$cv" in
          absent:present)
            row L3-HIGH cmdline_mitigations_off_appeared "cmdline 新增 ${cmv:-mitigations=}：脚本一已把它判为整体关闭或性能优先档；上游文档写明 mitigations=off 是一组按架构聚合的关闭开关，这一处改动会覆盖内核侧全部缓解" ;;
          present:absent) row L1-INFO cmdline_mitigations_off_removed "cmdline 移除了 ${bmv:-mitigations=} 档位：回到内核默认档；仍需逐项看 VULN 读数确认，参数消失不等于缓解生效" ;;
          *) row L2-WARN cmdline_mitigations_value_changed "mitigations= 档位取值由 ${bmv:-未采到} 变为 ${cmv:-未采到}：上游为每个取值定义了它聚合哪些架构专项开关，方向要按内核文档逐个取值判定，本脚本不解释具体档位" ;;
        esac ;;
      init_on_free=1)
        case "$bv:$cv" in
          absent:present) row L1-INFO cmdline_init_on_free_gained "cmdline 新增 init_on_free=1：释放内存即清零，公开口径下这会压缩残留数据被再次读到的窗口" ;;
          present:absent) row L2-WARN cmdline_init_on_free_lost "cmdline 移除了 init_on_free=1：释放页残留数据可被后任读者看到，这一项常被性能调优顺手关掉" ;;
          *) : ;;
        esac ;;
      *)
        case "$bv:$cv" in
          absent:present) row L2-WARN cmdline_disable_token_added "cmdline 新增 $tok：这是一个显式关闭某一类缓解或地址空间隔离的启动参数，要查是谁在什么时候改的" ;;
          present:absent) row L1-INFO cmdline_disable_token_removed "cmdline 移除了 $tok：对应缓解不再被启动参数显式关闭，但是否真正生效仍看 VULN 读数" ;;
          *) : ;;
        esac ;;
    esac
  done
fi

# ---------- CAP：缓解是否有硬件入口 ----------
b_cp="$(pkey "$base")"; c_cp="$(pkey "$curr")"
for cap in $(printf '%s\n' "$b_cp" "$c_cp" | grep '[^[:space:]]' | sort -u); do
  bv="$(lf3 "$base" CAP "$cap")"; cv="$(lf3 "$curr" CAP "$cap")"
  [ "$bv" = "$cv" ] && continue
  printf 'DELTA,CAP,%s,%s,%s\n' "$cap" "${bv:-none}" "${cv:-none}"
  case "$(readable "$bv"; echo $?):$(readable "$cv"; echo $?)" in
    0:1) row L0-LOW cpu_cap_became_unread "$cap 位由 $bv 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失" ;;
    1:0) row L1-INFO cpu_cap_now_readable "$cap 位由读不到变为 $cv：新增硬件前提证据，基线侧没有可比读数" ;;
    0:0)
      case "$bv:$cv" in
        present:absent) row L2-WARN cpu_cap_lost "$cap 位由可见变为不可见：同一颗芯片的特性位不会因升级消失，这一变化更可能来自采集域或内核打印格式变化，需复核；若确为不同芯片则本差分跨设备" ;;
        absent:present) row L1-INFO cpu_cap_gained "$cap 位由不可见变为可见：同上，先排除读数格式变化，再谈"这一类缓解有了硬件入口"" ;;
        *) : ;;
      esac ;;
    *) : ;;
  esac
done

# ---------- POL / SYSCTL：观察通道由谁收敛 ----------
b_en="$(lf3 "$base" POL selinux)"; c_en="$(lf3 "$curr" POL selinux)"
if [ "$b_en" != "$c_en" ]; then
  printf 'DELTA,POL,selinux,%s,%s\n' "${b_en:-none}" "${c_en:-none}"
  case "$b_en:$c_en" in
    Enforcing:Permissive) row L3-HIGH selinux_loosened "SELinux 由 Enforcing 变为 Permissive：域级限制只记录不拦截，本快照里所有依赖策略的观察通道收敛判定同时失效" ;;
    Permissive:Enforcing) row L1-INFO selinux_tightened "SELinux 由 Permissive 变为 Enforcing：域级限制恢复拦截，基线里的 perf 与 /proc 侧结论要按新策略重跑" ;;
    Disabled:*) row L2-WARN selinux_disabled_now "SELinux 基线为 Disabled 变为 $c_en：以 Disabled 为基线的快照不能用于安全收敛判定，整份基线作废重采" ;;
    *:Disabled) row L3-HIGH selinux_disabled_now "SELinux 当前为 Disabled：强制模式关闭，观察通道不再由策略域收敛，这是本差分里优先级最高的一行" ;;
    unread:*) row L1-INFO selinux_now_readable "getenforce 由读不到变为 $c_en：新增证据，基线侧不可比" ;;
    *:unread) row L0-LOW selinux_became_unread "getenforce 由 $b_en 变为读不到：强制模式不可判定，本脚本按缺口处理而不是按未启用处理" ;;
    *) row L2-WARN selinux_state_changed "SELinux 取值由 $b_en 变为 $c_en：不属于 Enforcing/Permissive 的常规口径，需确认采集域与构建形态" ;;
  esac
fi
b_hp="$(lf3 "$base" POL hidepid)"; c_hp="$(lf3 "$curr" POL hidepid)"
if [ "$b_hp" != "$c_hp" ]; then
  printf 'DELTA,POL,hidepid,%s,%s\n' "${b_hp:-none}" "${c_hp:-none}"
  case "$b_hp:$c_hp" in
    on:off) row L2-WARN proc_hidepid_lost "/proc 挂载选项由带 hidepid 变为不带：跨 UID 读取他进程 /proc 条目的那层收敛被撤掉；Android 上这一项属厂商自选，撤掉要问改动来源" ;;
    off:on) row L1-INFO proc_hidepid_gained "/proc 挂载新增 hidepid：跨 UID 可见性多了一层挂载级收敛；注意它不是 Android 平台默认，不能作为基线要求的达标项" ;;
    *:unknown) row L0-LOW proc_mount_became_unread "/proc 挂载行由 $b_hp 变为采不到：hidepid 判定留为缺口" ;;
    unknown:*) row L1-INFO proc_mount_now_readable "/proc 挂载行由采不到变为 $c_hp：新增证据" ;;
  esac
fi
tight_rank() { # tight_rank <key> <value> -> 越大越收敛；读不到返回空
  local k="$1" v="$2"
  case "$v" in ''|none|unread) return 0 ;; esac
  case "$k" in
    kernel.perf_event_paranoid) printf '%s\n' "$v" ;;
    *) case "$v" in 0) printf '0\n' ;; *) printf '1\n' ;; esac ;;
  esac
}
for k in $(printf '%s\n' "$(skey "$base")" "$(skey "$curr")" | grep '[^[:space:]]' | sort -u); do
  bv="$(lf3 "$base" SYSCTL "$k")"; cv="$(lf3 "$curr" SYSCTL "$k")"
  [ "$bv" = "$cv" ] && continue
  printf 'DELTA,SYSCTL,%s,%s,%s\n' "$k" "${bv:-none}" "${cv:-none}"
  br="$(tight_rank "$k" "$bv")"; cr="$(tight_rank "$k" "$cv")"
  if [ -z "$cr" ]; then
    row L0-LOW sysctl_became_unread "$k 由 $bv 变为读不到：观察通道收敛状态不可判定，节点被删除与采集权限退化在本脚本里不等价"
  elif [ -z "$br" ]; then
    row L1-INFO sysctl_now_readable "$k 由读不到变为 $cv：新增证据，基线侧不可比"
  elif [ "$cr" -lt "$br" ] 2>/dev/null; then
    row L2-WARN sysctl_loosened "$k 由 $bv 变为 $cv：档位向更开放的方向移动，采样与符号化信息的可读面在该采集域内扩大；shell 域可读不等于应用域可读"
  else
    row L1-INFO sysctl_tightened "$k 由 $bv 变为 $cv：档位向更收敛的方向移动"
  fi
done
b_mc="$(lf3 "$base" POL microcode)"; c_mc="$(lf3 "$curr" POL microcode)"
if [ "$b_mc" != "$c_mc" ]; then
  printf 'DELTA,POL,microcode,%s,%s\n' "${b_mc:-none}" "${c_mc:-none}"
  case "$(readable "$b_mc"; echo $?):$(readable "$c_mc"; echo $?)" in
    0:0) row L2-WARN microcode_changed "x86 微码版本行变化（$b_mc → $c_mc）：这是设备侧唯一能读到的硅侧缓解证据，方向要按厂商微码版本表判定，本脚本不猜测新旧" ;;
    0:1) row L0-LOW microcode_evidence_lost "微码版本行由可读变为读不到：设备侧失去证明微码缓解到达的唯一锚点，只能向厂商索取证据" ;;
    1:0) row L1-INFO microcode_evidence_gained "微码版本行由读不到变为可读（$c_mc）：新增锚点，与厂商版本表对齐后才构成结论" ;;
    *) : ;;
  esac
fi

# ---------- 邻居与固件锚点 ----------
b_zy="$(cntv "$base" zygote_processes)"; c_zy="$(cntv "$curr" zygote_processes)"
zy_xor=0
if [ -z "$b_zy" ] && [ -n "$c_zy" ]; then zy_xor=1; fi
if [ -n "$b_zy" ] && [ -z "$c_zy" ]; then zy_xor=1; fi
if [ "$zy_xor" = 1 ]; then
  printf 'DELTA,COUNT,zygote_processes,%s,%s\n' "${b_zy:-none}" "${c_zy:-none}"
  row L0-LOW neighbour_list_degraded "进程表只在其中一份快照里采到：同机邻居一节的漂移不可判定，缺侧按采集退化处理"
elif [ -n "$b_zy" ] && [ -n "$c_zy" ] && [ "$b_zy" != "$c_zy" ]; then
  printf 'DELTA,COUNT,zygote_processes,%s,%s\n' "$b_zy" "$c_zy"
  if [ "$c_zy" = 0 ]; then
    row L0-LOW zygote_absent_now "zygote 进程计数由 $b_zy 变为 0：进程表采集被截断或该设备形态不是标准 Android 运行时，同机邻居清点不成立"
  else
    row L1-INFO zygote_count_changed "zygote 进程计数由 $b_zy 变为 $c_zy：孵化母进程数量变化会影响共享缓存初始态的清点口径"
  fi
fi
b_btl="$(mf "$base" bootloader)"; c_btl="$(mf "$curr" bootloader)"
if [ "$b_btl" != "$c_btl" ]; then
  printf 'DELTA,META,bootloader,%s,%s\n' "${b_btl:-none}" "${c_btl:-none}"
  case "$(readable "$b_btl"; echo $?):$(readable "$c_btl"; echo $?)" in
    0:1) row L0-LOW bootloader_lost "ro.boot.bootloader 由可读变为读不到：厂商固件版本锚点丢失，CPU 侧修复是否随固件到达这件事重新变成不可归因" ;;
    1:0) row L1-INFO bootloader_gained "ro.boot.bootloader 由读不到变为 $c_btl：新增固件锚点，可开始与厂商公告对齐" ;;
    *) row L1-INFO bootloader_changed "ro.boot.bootloader 由 $b_btl 变为 $c_btl：固件镜像标识变化，可与厂商公告比对；它不直接证明微码级别" ;;
  esac
fi

# ---------- 信号码集合差分：消失不等于已修 ----------
for lv in L3 L2; do
  newc="$(comm -13 <(rowsig "$base" "$lv") <(rowsig "$curr" "$lv") | grep '[^[:space:]]' || true)"
  gonec="$(comm -23 <(rowsig "$base" "$lv") <(rowsig "$curr" "$lv") | grep '[^[:space:]]' || true)"
  for code in $newc; do
    printf 'NEW,%s,%s\n' "$lv" "$code"
    if [ "$lv" = L3 ]; then
      row L3-HIGH drift_new_high "当前快照新增高等级信号 $code：这是相对基线新出现的敞口或状态迁移，逐项定位见上方 DELTA 行"
    else
      row L2-WARN drift_new_warn "当前快照新增中等级信号 $code：相对基线新出现的需复核项"
    fi
  done
  for code in $gonec; do
    printf 'GONE,%s,%s\n' "$lv" "$code"
    case "$code" in
      vuln_*|arm_microcode_face_absent) probe="$(grep -c '^VULN,' "$curr" || true)" ;;
      cmdline_*)                       probe="$(lf3 "$curr" CMD 'mitigations=')"; case "$probe" in unread) probe=0 ;; *) probe=1 ;; esac ;;
      cpu_*)  probe="$(lf3 "$curr" CAP 'ssbs')"; case "$probe" in ''|unknown|unread) probe="$(lf3 "$curr" CAP 'ibrs')" ;; esac; case "$probe" in ''|unknown|unread) probe=0 ;; *) probe=1 ;; esac ;;
      sysctl_*)                        probe="$(skey "$curr" | grep -c '[^[:space:]]' || true)"; [ "${probe:-0}" -eq 0 ] && probe=0 ;;
      proc_mount_*|selinux_*|neighbour_*|ps_*) probe=1 ;;
      *) probe=1 ;;
    esac
    if [ "$probe" = 0 ]; then
      row L1-INFO risk_gone_by_coverage "$lv 级信号 $code 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合"
    else
      row L2-WARN risk_gone_needs_proof "$lv 级信号 $code 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修"
    fi
  done
done
newgap="$(comm -13 <(rowsig "$base" 'L0-') <(rowsig "$curr" 'L0-') | grep '[^[:space:]]' || true)"
[ -n "$newgap" ] && printf 'NEW,L0-LOW,%s\n' "$(printf '%s\n' "$newgap" | tr '\n' ';')" && \
  row L0-LOW drift_new_gaps "当前快照新增缺口信号 $(printf '%s\n' "$newgap" | tr '\n' ' ')：这些是采集面收窄的记号，下一轮优先补"
gonegap="$(comm -23 <(rowsig "$base" 'L0-') <(rowsig "$curr" 'L0-') | grep '[^[:space:]]' || true)"
[ -n "$gonegap" ] && printf 'GONE,L0-LOW,%s\n' "$(printf '%s\n' "$gonegap" | tr '\n' ';')" && \
  row L1-INFO drift_gap_closed "基线缺口信号 $(printf '%s\n' "$gonegap" | tr '\n' ' ') 在当前快照不再出现：采集面被补上，这一轮的结论可比范围扩大"

# ---------- 快照厚度 ----------
b_n="$(grep -c '^ROW,' "$base" || true)"; c_n="$(grep -c '^ROW,' "$curr" || true)"
if [ "${c_n:-0}" -lt "${b_n:-0}" ]; then
  row L2-WARN snapshot_thinner "当前快照判定行由 ${b_n:-0} 条减到 ${c_n:-0} 条：可比项减少，本差分的"没有变化"只覆盖剩下的这些项"
elif [ "${c_n:-0}" -gt "${b_n:-0}" ]; then
  row L1-INFO snapshot_richer "当前快照判定行由 ${b_n:-0} 条增到 ${c_n:-0} 条：新增项多为首次可读的读数，没有基线可比，下一轮才能形成趋势"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s\n' "$hi" "$warn" "$gap" "$info"
```
## 廿四、脚本 2 的四份真实差分输出

本章的四段输出都由第廿三章的脚本对第廿二章同源的快照文件实际运行产生，**逐字节取自运行产物，没有手写、没有节选、没有改行序**。文件路径写在输出首两行的 `PAIR,base` / `PAIR,curr` 里，读者可以按第廿五章的命令重跑。

四份样例是按**四种对子关系**选的，而不是按"报警多少"选的。这是脚本 2 的实际用法：**先确认这两份快照之间是什么关系，再决定这份输出能读成什么。**

| 样例 | 对子关系 | 为什么值得单独看 |
| --- | --- | --- |
| 廿四之一 `p014-015` | 同型号、同 SPL、同内核的两轮 | 差分的**默认形态**：没有一条 L3，绝大多数行是"新增读数首次可比" |
| 廿四之二 `p001-010` | 跨设备（Pixel 7 → Table T3） | 演示 `pair_identity_mismatch` 如何把**整份输出的解释口径**从"变化"改成"跨设备" |
| 廿四之三 `p003-001` | 塌陷基线 → 可读当前 | 演示"缺口被补上"这一方向：可读项大增，但**没有一条能算趋势** |
| 廿四之四 `p002-003` | 可读基线 → 塌陷当前 | 演示最危险的一种误读：**高等级行大面积消失，而结论是"不可比"而不是"变好"** |

四份输出合起来覆盖第廿三章的全部判定分流；第廿五章给出它们与其余 221 份输出一起构成的分支覆盖账。

### 廿四之一 样例一：同一台设备的两轮（`p014-015`）

两份快照来自同一份合成档案族的相邻两版：`Kit K1`、`user` 构建、SPL 同为 2026-09-05、内核同为 `6.1.75-ab99999`。基线（sc014）**没有 vulnerabilities 目录**，当前（sc015）有 `spectre_v1`、`spectre_v2` 两个文件；基线的 `/proc/cpuinfo` Features 含 `sb`，当前的 Features 串里没有。

```text
PAIR,base,.re15/sc014.txt,rows=20
PAIR,curr,.re15/sc015.txt,rows=24
ROW,L1-INFO,pair_identity_same_model,两份快照机型一致（Kit K1）：仍要由操作者确认它们来自同一台设备、同一条采集流程与同一采集域
NOTE,identity,快照内不含采集时间戳与构建号，同源性由操作者自行记录
NEW,VULN,spectre_v1,mitigated
ROW,L1-INFO,vuln_file_new,spectre_v1 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
NEW,VULN,spectre_v2,mitigated
ROW,L1-INFO,vuln_file_new,spectre_v2 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
DELTA,vuln_files,0,2
ROW,L1-INFO,vuln_files_more,vulnerabilities 读数文件由 0 个增加到 2 个：新增项多为内核新版本导出的新读数，首次出现时没有基线可比
DELTA,CAP,sb,present,absent
ROW,L2-WARN,cpu_cap_lost,sb 位由可见变为不可见：同一颗芯片的特性位不会因升级消失，这一变化更可能来自采集域或内核打印格式变化，需复核；若确为不同芯片则本差分跨设备
DELTA,POL,selinux,Enforcing,EnforcingPermissive
ROW,L2-WARN,selinux_state_changed,SELinux 取值由 Enforcing 变为 EnforcingPermissive：不属于 Enforcing/Permissive 的常规口径，需确认采集域与构建形态
NEW,L2,cpu_sb_absent
ROW,L2-WARN,drift_new_warn,当前快照新增中等级信号 cpu_sb_absent：相对基线新出现的需复核项
NEW,L2,selinux_unexpected
ROW,L2-WARN,drift_new_warn,当前快照新增中等级信号 selinux_unexpected：相对基线新出现的需复核项
NEW,L0-LOW,arm_microcode_face_absent;vuln_core_file_absent;
ROW,L0-LOW,drift_new_gaps,当前快照新增缺口信号 arm_microcode_face_absent vuln_core_file_absent ：这些是采集面收窄的记号，下一轮优先补
GONE,L0-LOW,vuln_dir_unread;
ROW,L1-INFO,drift_gap_closed,基线缺口信号 vuln_dir_unread  在当前快照不再出现：采集面被补上，这一轮的结论可比范围扩大
ROW,L1-INFO,snapshot_richer,当前快照判定行由 20 条增到 24 条：新增项多为首次可读的读数，没有基线可比，下一轮才能形成趋势
SUMMARY,high=0,warn=4,gap=1,info=6
```

九点读法：

1. **第 3 行是这份输出的准入条件。** `pair_identity_same_model` 只说"型号字符串相同"，句子后半句"仍要由操作者确认它们来自同一台设备、同一条采集流程与同一采集域"是**脚本无法替读者完成的部分**。快照里没有采集时间戳与构建号（第 4 行 `NOTE,identity` 明说这件事），所以同源性在这套契约里是**操作者的记录义务**，不是脚本的证明能力。
2. **这份对子里没有任何一条 L3、没有任何 `vuln_state_worse`。** 这是最重要的一课：**同一台设备的两轮差分，绝大多数时候就是这个样子**——变化集中在"新增读数"和"采集面变化"上。把它当成"什么都没发生"是错的，把它当成"风险在下降"也是错的。
3. **第 5–10 行是"基线里没有这一项"这一档。** 三条 `vuln_file_new` + 一条 `vuln_files_more`：`spectre_v1`、`spectre_v2` 首次出现。**脚本明确拒绝把"新增"读成"新增敞口"或"新增缓解"**，句子写的是"无法判断是内核新增了读数还是基线采集被截断"。这两件事在设备侧不可区分，因为快照里没有采集面完整性的自证字段（第十四章之二列出的系统性遗漏之一）。
4. **第 11–12 行是本样例唯一一条真需要复核的变化。** `sb` 位由 present 变 absent。`cpu_cap_lost` 的句子给出了正确读法：**同一颗芯片的特性位不会因为升级而消失**，所以先怀疑采集域或内核打印格式（第七章 七之二 说明 Features 行来自一张固定的名字表），再怀疑"确实换了芯片"——而如果确实换了芯片，那这对快照就是跨设备对子，`pair_identity_same_model` 反而成了假信号。
5. **第 13–14 行是一个刻意构造的分支演示。** sc015 档案里的 `getenforce` 文本被写成 `EnforcingPermissive`——这不是任何真实设备的取值（合法取值只有 Enforcing / Permissive / Disabled），它存在的唯一目的是让 `selinux_unexpected` 与 `selinux_state_changed` 这两条兜底分支**可达并被真实触发**。这一条在第廿五章的覆盖账里登记，第廿五章同时说明：**为了触发分支而构造的画像，不能同时被当作真实设备的读数来引用。**
6. **第 15–18 行是等级归属的补充记号。** 两条 `drift_new_warn`：`cpu_sb_absent`、`selinux_unexpected` 是**当前快照新出现的判定行**，由两份快照的 ROW 码位集合做差集得出（廿三之六），而不是从 DELTA 推出来的。这一点决定了它们不会被"逐项变化"漏掉：只要当前有、基线没有，就进 `NEW,L2`。
7. **第 19–22 行说明缺口也是双向的。** `drift_new_gaps` 登记当前新增的两条缺口（`arm_microcode_face_absent`、`vuln_core_file_absent`），`drift_gap_closed` 登记基线缺口 `vuln_dir_unread` 在当前不再出现。**两者都是"采集面形状变了"，都不是"安全状态变了"**——但 `drift_gap_closed` 有一个真实用途：它告诉操作者这一轮的可比范围扩大了，从下一轮开始这一节才有历史可比。
8. **第 23 行的"厚"不等于"好"。** `snapshot_richer`（20 → 24 行）句子写的是"新增项多为首次可读的读数，没有基线可比，下一轮才能形成趋势"。**这份输出的正确结论是一句流程话**：vulnerabilities 面已被采到，下一轮起把 `spectre_v1`/`spectre_v2` 与 Features 行的 `sb` 纳入逐项跟踪。
9. **`SUMMARY,high=0,warn=4,gap=1,info=6`。** 四个 warn 里有两个来自同一条变化（第 12 行 `cpu_cap_lost` 与第 16 行 `drift_new_warn(cpu_sb_absent)`）——**SUMMARY 统计的是判定行数，不是事件数**。拿它算"问题数"会重复计数，这就是第廿六章 K9 的口径必须写明"按行统计、不按事件统计"的原因。

### 廿四之二 样例二：跨设备 + 采集面收窄（`p001-010`）

基线是第廿二章样例 A（`Pixel 7`、arm64、SPL 2026-10-05、bootloader 可读、五个读数文件），当前是第廿二章样例 C（`Table T3`、arm64、SPL 2026-06-05、cpuinfo 与 cmdline 都采不到、四个读数文件）。

```text
PAIR,base,.re15/sc001.txt,rows=28
PAIR,curr,.re15/sc010.txt,rows=25
ROW,L2-WARN,pair_identity_mismatch,基线机型 Pixel 7 与当前机型 Table T3 不同：本差分跨设备，任何差异都不能读成一次收紧或劣化
NOTE,identity,快照内不含采集时间戳与构建号，同源性由操作者自行记录
DELTA,kernel_release,5.10.209-android13-8-g0e57eb5cd3a1 #1 SMP PREEMPT,5.15.123-ab14752355 #1 SMP PREEMPT Thu Aug 21 17:05:00 UTC 2025 aarch64
ROW,L1-INFO,kernel_changed,内核版本串变化（5.10.209-android13-8-g0e57eb5cd3a1 #1 SMP PREEMPT → 5.15.123-ab14752355 #1 SMP PREEMPT Thu Aug 21 17:05:00 UTC 2025 aarch64）：读数词表、缓解形态与默认档位都可能随之整体换档，逐项差异要放在这个前提下读
ROW,L2-WARN,spl_moved_backward,security_patch 由 2026-10-05 退到 2026-06-05：补丁级别回退，本差分的其余读数都按回退后的镜像重读
DELTA,VULN,spec_store_bypass,mitigated,vulnerable
ROW,L3-HIGH,vuln_state_worse,spec_store_bypass 由 mitigated 变为 Vulnerable：这一类泄露通道在本机自述没有生效中的缓解；先查 cmdline 是否被加了关闭开关，再查内核是否换档
DELTA,VULN,spectre_bhb,mitigated,not_affected
ROW,L1-INFO,vuln_state_unclassified_move,spectre_bhb 由 mitigated 变为 not_affected：这一迁移不属于本脚本枚举的三类方向，按原始读数交人工复核（Mitigation: Loop → Not affected）
DELTA,VULN,spectre_v1,mitigated,mitigated
DELTA,VULN,spectre_v2,mitigated,mitigated
GONE,VULN,ssb,not_affected
ROW,L0-LOW,vuln_file_gone,ssb 在当前快照消失（基线为 not_affected）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
DELTA,vuln_files,5,4
ROW,L2-WARN,vuln_files_fewer,vulnerabilities 读数文件由 5 个减少到 4 个：可读面收窄，本次差分能确认的项随之减少，收窄本身不是改善
DELTA,vuln_vulnerable,0,1
ROW,L3-HIGH,vuln_open_count_up,自述 Vulnerable 的读数项由 0 增至 1：敞口项数量上升，配合逐项 DELTA 行定位是哪一类通道新开放
ROW,L0-LOW,cmdline_became_unread,/proc/cmdline 由可读变为读不到：启动参数层的漂移全部不可判定，这一档缺口要在下一轮采集补上
DELTA,CAP,aes,present,unknown
ROW,L0-LOW,cpu_cap_became_unread,aes 位由 present 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,afp,present,unknown
ROW,L0-LOW,cpu_cap_became_unread,afp 位由 present 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,gcs,absent,unknown
ROW,L0-LOW,cpu_cap_became_unread,gcs 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,sb,present,unknown
ROW,L0-LOW,cpu_cap_became_unread,sb 位由 present 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,ssbs,present,unknown
ROW,L0-LOW,cpu_cap_became_unread,ssbs 位由 present 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,wfxt,present,unknown
ROW,L0-LOW,cpu_cap_became_unread,wfxt 位由 present 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,POL,hidepid,on,off
ROW,L2-WARN,proc_hidepid_lost,/proc 挂载选项由带 hidepid 变为不带：跨 UID 读取他进程 /proc 条目的那层收敛被撤掉；Android 上这一项属厂商自选，撤掉要问改动来源
DELTA,SYSCTL,kernel.perf_event_paranoid,2,unread
ROW,L0-LOW,sysctl_became_unread,kernel.perf_event_paranoid 由 2 变为读不到：观察通道收敛状态不可判定，节点被删除与采集权限退化在本脚本里不等价
DELTA,META,bootloader,cheetah-1.4.2-11791055,none
ROW,L0-LOW,bootloader_lost,ro.boot.bootloader 由可读变为读不到：厂商固件版本锚点丢失，CPU 侧修复是否随固件到达这件事重新变成不可归因
NEW,L3,vuln_vulnerable
ROW,L3-HIGH,drift_new_high,当前快照新增高等级信号 vuln_vulnerable：这是相对基线新出现的敞口或状态迁移，逐项定位见上方 DELTA 行
NEW,L0-LOW,bootloader_absent;cmdline_unread;cpu_cap_unread;cpufreq_unread;sysctl_paranoid_unread;
ROW,L0-LOW,drift_new_gaps,当前快照新增缺口信号 bootloader_absent cmdline_unread cpu_cap_unread cpufreq_unread sysctl_paranoid_unread ：这些是采集面收窄的记号，下一轮优先补
ROW,L2-WARN,snapshot_thinner,当前快照判定行由 28 条减到 25 条：可比项减少，本差分的没有变化只覆盖剩下的这些项
SUMMARY,high=3,warn=5,gap=11,info=2
```

十一点读法：

1. **第 3 行覆盖第 4 行以下的全部句子。** `pair_identity_mismatch` 把整份输出的性质从"这台机器的变化"改成"两台机器的差别"。句子写得很硬："任何差异都不能读成一次收紧或劣化"。**注意脚本并没有因此停止判定**——它继续逐项出 DELTA，因为跨设备比对本身有正当用途（比较两个机型的出厂形态），但用途不同，见第 10 点。
2. **第 5–6 行是这个用途的示例：内核换档。** 5.10.209-android13 → 5.15.123，`kernel_changed`（L1）句子里给出前提："读数词表、缓解形态与默认档位都可能随之整体换档，逐项差异要放在这个前提下读"。**这条 L1 不是装饰**：第 7 行 `spl_moved_backward`（L2）之所以只报方向而不报"退步"，正是因为两台机器的 SPL 本就不可比。
3. **第 7 行是这套脚本里最容易读错的一行。** `spl_moved_backward` 的句子把结论限定成"本差分的其余读数都按回退后的镜像重读"，而不是"这台设备变差了"。**SPL 是时间锚点，不是缓解状态**（第八章 八之五），跨设备时它连锚点作用都要打折。
4. **第 8–9 行是全篇唯一的 L3 方向性判定，而且它站得住。** `spec_store_bypass` 由 mitigated 变 vulnerable → `vuln_state_worse`。之所以在跨设备场景下仍值得记 L3：**两侧这一项都是可读的具名读数，方向明确，不存在"基线是缺口"的问题**；句子同时给出下一步动作顺序（先查 cmdline 是否被加了关闭开关，再查内核是否换档）。第廿二章样例 C 已经说明当前侧 cmdline 采不到——所以这条 L3 的处置动作**必然是重采**，而不是派工修 CPU。
5. **第 10–11 行示范"脚本拒绝给方向"。** `spectre_bhb` 由 mitigated 变 not_affected：这一迁移不在枚举的三类方向里（改善/恶化/空值），于是走 `vuln_state_unclassified_move` 并把两个原始说明文字（`Mitigation: Loop` → `Not affected`）交人工复核。**这不是保守，是正确性**：`not_affected` 可能是厂商判定，也可能是通用回退实现（第六章 六之二），两者等级完全不同，脚本没有信息区分它们。
6. **第 12–13 行是这套契约里唯一一处"值没变也打印"的地方。** `spectre_v1` 与 `spectre_v2` 都是 `mitigated,mitigated`，后面**不跟任何 ROW 行**。第廿三章 廿三之四 把这条设计写成契约：VULN 段的 `DELTA` 行无条件打印，让"逐项可比且无变化"这件事本身留下证据；其他段（CMD/CAP/POL/SYSCTL）只在值不同才打印。**消费这份输出时，判断"是否真的变化"要看有没有 ROW 行，而不是看有没有 DELTA 行。**
7. **第 14–15 行是"文件消失"的标准处理。** `ssb` 在当前不再出现 → `vuln_file_gone`（L0-LOW）+ 句子"文件不导出不等于该面被修复"。**注意它没有被记成改善**，而第 16–17 行（`DELTA,vuln_files,5,4` 与 `vuln_files_fewer`，L2）把这件事升级成一条需要处理的采集面事件：**可读面收窄本身是 L2，因为它削减的是下一轮的可判定范围。**
8. **第 18–19 行是全篇第二处 L3 的来源。** `vuln_open_count_up`（自述 Vulnerable 的项数 0 → 1）。**逐项 DELTA 告诉你哪一项，计数行告诉你规模**——两者都要有，只有前者容易被"这只是个例"淡化，只有后者无法派工。
9. **第 20–38 行是采集退化的整块形态。** cmdline 变不可读（20）、六个特性位变 unknown（21–32）、paranoid 变读不到（35–36）、bootloader 锚点丢失（37–38）。**每一条都是 L0-LOW，每一条句子都明确写"不判成能力丧失"**。这批行合起来的含义是：**这台机器的这一轮快照，可判定面比基线小得多**——所以第 43 行 `snapshot_thinner` 是 L2 而不是 L1：可比项从 28 减到 25，"没有变化"这句话的覆盖范围随之缩小。
10. **第 33–34 行是本样例里唯一一条"策略面真的松了"的候选。** `proc_hidepid_lost`（on → off）判 L2-WARN，句子点出 Android 特殊性：**这一项属厂商自选**，缺它不等于敞口（第十一章 十一之三 给出策略侧如何以 neverallow 复现同类效果），但**变化来源必须问**——它不该在一次常规 OTA 里静默消失。
11. **第 39–42 行给出这份输出的正确出口。** `NEW,L3,vuln_vulnerable` + `drift_new_high` 是码位集合差集的结果（当前有、基线没有的 L3 码位），`drift_new_gaps` 列出五条新缺口。**跨设备对子的用途在这里**：它不是"这台机器的趋势"，而是**机型间的差异清单**——把 `vuln_vulnerable`、`cmdline_unread`、`cpu_cap_unread` 这三组记成"Table T3 相对 Pixel 7 的可见差异"，然后据此决定这个机型要不要单独建档、单独采证。**同一份输出，读成趋势是错的，读成机型差异清单是对的。**
12. **`SUMMARY,high=3,warn=5,gap=11,info=2`（第 44 行）。** gap=11 与 warn=5 一起才是这份输出的重心：**一条 L3 加上十一格采集缺口**，处置顺序是先补缺口再谈缓解（第二十章的信号码语义把这一点写成"未采到 = 操作未到位"）。

### 廿四之三 样例三：塌陷基线 → 可读当前（`p003-001`）

基线 sc003 是采集面几乎全塌的一份（无机型、无 SPL、无内核版本、无 vulnerabilities 目录，`getenforce` 与三个 sysctl 都读不到，16 行）；当前 sc001 是第廿二章样例 A（`Pixel 7`、28 行）。这是**留存基线做错了第一轮**的典型后果。

```text
PAIR,base,.re15/sc003.txt,rows=16
PAIR,curr,.re15/sc001.txt,rows=28
ROW,L0-LOW,pair_identity_unread,至少一份快照没有机型标识：无法确认这两份产物来自同一台设备，差分结果只做参考
NOTE,identity,快照内不含采集时间戳与构建号，同源性由操作者自行记录
DELTA,build_type,none,user
ROW,L2-WARN,build_type_changed,构建形态由 未采到 变为 user：观察通道的默认收敛随构建形态改变，非发行构建与发行构建不是同一条基线
ROW,L1-INFO,arch_gained,架构判定由判不出变为 arm64：基线里那些'某文件不存在'的缺口现在才有了解释口径，需重跑基线才能对比缺席项
NEW,VULN,spec_store_bypass,mitigated
ROW,L1-INFO,vuln_file_new,spec_store_bypass 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
NEW,VULN,spectre_bhb,mitigated
ROW,L1-INFO,vuln_file_new,spectre_bhb 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
NEW,VULN,spectre_v1,mitigated
ROW,L1-INFO,vuln_file_new,spectre_v1 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
NEW,VULN,spectre_v2,mitigated
ROW,L1-INFO,vuln_file_new,spectre_v2 在当前快照新增（mitigated）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
NEW,VULN,ssb,not_affected
ROW,L1-INFO,vuln_file_new,ssb 在当前快照新增（not_affected）：基线里没有这一项，无法判断是内核新增了读数还是基线采集被截断
DELTA,vuln_files,0,5
ROW,L1-INFO,vuln_files_more,vulnerabilities 读数文件由 0 个增加到 5 个：新增项多为内核新版本导出的新读数，首次出现时没有基线可比
ROW,L1-INFO,cmdline_now_readable,/proc/cmdline 由读不到变为可读：新增的启动参数证据，需重跑基线才有可比性
DELTA,CAP,aes,none,present
ROW,L1-INFO,cpu_cap_now_readable,aes 位由读不到变为 present：新增硬件前提证据，基线侧没有可比读数
DELTA,CAP,afp,none,present
ROW,L1-INFO,cpu_cap_now_readable,afp 位由读不到变为 present：新增硬件前提证据，基线侧没有可比读数
DELTA,CAP,gcs,none,absent
ROW,L1-INFO,cpu_cap_now_readable,gcs 位由读不到变为 absent：新增硬件前提证据，基线侧没有可比读数
DELTA,CAP,sb,none,present
ROW,L1-INFO,cpu_cap_now_readable,sb 位由读不到变为 present：新增硬件前提证据，基线侧没有可比读数
DELTA,CAP,ssbs,none,present
ROW,L1-INFO,cpu_cap_now_readable,ssbs 位由读不到变为 present：新增硬件前提证据，基线侧没有可比读数
DELTA,CAP,wfxt,none,present
ROW,L1-INFO,cpu_cap_now_readable,wfxt 位由读不到变为 present：新增硬件前提证据，基线侧没有可比读数
DELTA,POL,selinux,unread,Enforcing
ROW,L1-INFO,selinux_now_readable,getenforce 由读不到变为 Enforcing：新增证据，基线侧不可比
DELTA,POL,hidepid,unknown,on
ROW,L1-INFO,proc_mount_now_readable,/proc 挂载行由采不到变为 on：新增证据
DELTA,SYSCTL,kernel.dmesg_restrict,unread,1
ROW,L1-INFO,sysctl_now_readable,kernel.dmesg_restrict 由读不到变为 1：新增证据，基线侧不可比
DELTA,SYSCTL,kernel.kptr_restrict,unread,1
ROW,L1-INFO,sysctl_now_readable,kernel.kptr_restrict 由读不到变为 1：新增证据，基线侧不可比
DELTA,SYSCTL,kernel.perf_event_paranoid,unread,2
ROW,L1-INFO,sysctl_now_readable,kernel.perf_event_paranoid 由读不到变为 2：新增证据，基线侧不可比
DELTA,COUNT,zygote_processes,none,1
ROW,L0-LOW,neighbour_list_degraded,进程表只在其中一份快照里采到：同机邻居一节的漂移不可判定，缺侧按采集退化处理
DELTA,META,bootloader,none,cheetah-1.4.2-11791055
ROW,L1-INFO,bootloader_gained,ro.boot.bootloader 由读不到变为 cheetah-1.4.2-11791055：新增固件锚点，可开始与厂商公告对齐
GONE,L2,emulator_build
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 emulator_build 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
NEW,L0-LOW,arm_microcode_face_absent;neighbor_app_domain_gap;
ROW,L0-LOW,drift_new_gaps,当前快照新增缺口信号 arm_microcode_face_absent neighbor_app_domain_gap ：这些是采集面收窄的记号，下一轮优先补
GONE,L0-LOW,bootloader_absent;build_type_unread;cmdline_unread;cpufreq_unread;cpuinfo_unread;identity_missing;kernel_release_unread;proc_mount_unread;ps_unread;selinux_unread;spl_absent;sysctl_paranoid_unread;sysctl_unread;vuln_dir_unread;
ROW,L1-INFO,drift_gap_closed,基线缺口信号 bootloader_absent build_type_unread cmdline_unread cpufreq_unread cpuinfo_unread identity_missing kernel_release_unread proc_mount_unread ps_unread selinux_unread spl_absent sysctl_paranoid_unread sysctl_unread vuln_dir_unread  在当前快照不再出现：采集面被补上，这一轮的结论可比范围扩大
ROW,L1-INFO,snapshot_richer,当前快照判定行由 16 条增到 28 条：新增项多为首次可读的读数，没有基线可比，下一轮才能形成趋势
SUMMARY,high=0,warn=2,gap=3,info=22
```

八点读法：

1. **第 3 行的 `pair_identity_unread` 与样例一的 `pair_identity_same_model` 是两种准入失败。** 后者是"机型相同但仍需操作者确认同源"，前者是"至少一份没有机型字符串，**连是不是同一台都无法判断**"，所以句子写成"差分结果只做参考"。**参考不是没用**：它仍标出了当前侧的缺口与新增项，只是不能进任何趋势账。
2. **第 5–6 行 `build_type_changed`（none → user）。** 这是这一族样例里反复出现的一类：基线侧是"未采到"而不是"另一个值"。`tight_rank` 无法给这种迁移定方向，所以脚本只陈述事实并提醒"非发行构建与发行构建不是同一条基线"。
3. **第 7 行 `arch_gained` 解释了一件事：基线里那些"某文件不存在"的缺口现在才有了解读口径。** 在 sc003 里，`vuln_dir_unread` 只是"没读到"；在 sc001 里架构已知是 arm64，缺席项才可以用第七章 七之四 的解释纪律去读。**架构判定的补上是后续所有缺席项判定的前提**，这条 L1 的实际权重比它看起来大。
4. **第 8–19 行：五条 `vuln_file_new` 与 `vuln_files_more`。** 基线一个读数都没有，当前有五个，其中四项 mitigated、`ssb` 是 not_affected。**一条都不能记成"这五个面被修好了"**，理由写在句子里：基线里没有这一项。这份输出的正确用法是**把 sc001 当作新的起点基线重跑一轮**。
5. **第 20–46 行是"新增可读证据"的整块。** cmdline（20）、六个特性位（21–32）、SELinux（33–34）、`/proc` 挂载行（35–36）、三个 sysctl（37–42）、bootloader 锚点（45–46）——全部是 `*_now_readable` 家族（L1），句子结构统一为"新增证据，基线侧不可比"。**这一族是本脚本里唯一成批出现而不该成批消费的记录**：它们说明采集被补好了，然后逐项交给下一轮。
6. **第 43–44 行是这一族的一个变体。** `neighbour_list_degraded`（L0）：进程表只在一侧采到，所以 zygote 邻居一节**整体不可判定**。它区别于"zygote 数变了"（`zygote_count_changed`），后者是 `DELTA,COUNT` 加一条方向性 ROW；前者是"这一节没有可比数据"。第廿三章 廿三之六 把这两条分成不同分支正是为此。
7. **第 47–48 行是全篇第一条 L2-WARN，也是这一章标题级的一课。** `GONE,L2,emulator_build` → `risk_gone_needs_proof`。基线被判成模拟器构建，当前不再触发这个判定。**"消失"有两种解释：问题确实被处理，或判定条件不再触发。** 这一条尤其危险，因为它是全篇唯一的好消息形状。处置动作在句子里：需要独立证据（公告、版本号、重采）才能记成已修。**注意 sc001 的机型是 Pixel 7、有真实 bootloader 串**，所以更可能的解释是判定条件不再触发（`ro.build` 里的 qemu 记号采到了不同值），而不是"模拟器被修好了"——但这是**读者的背景知识在补脚本的缺口**，脚本本身不能给出这个结论。
8. **第 49–54 行与 `SUMMARY,high=0,warn=2,gap=3,info=22`。** 22 条 info 是这份输出的形状：一次塌陷基线被补上之后，**几乎所有行都在说"这里以前没数据"**。`drift_gap_closed`（51–52）列出的十四条基线缺口在当前不再出现，是这次差分的真实收获；`drift_new_gaps`（49–50）提醒新的两条（`arm_microcode_face_absent`、`neighbor_app_domain_gap`）从这一轮起要补。**gap 从四份样例的 25 → 11 → 3 → 1 的变化轨迹，就是这套脚本被正确使用的过程**：先补采集面，再看等级。

### 廿四之四 样例四：高等级行大面积消失（`p002-003`）

基线 sc002 是全篇信息量最大的一份（第廿二章样例 B：`Tablet X2`、x86、`userdebug`、SPL 2023-11-05、八个读数文件其中五个自述 Vulnerable、`mitigations=off`、SELinux Permissive、三个 sysctl 全松、微码行可读）；当前 sc003 是采集面塌陷的一份。这份对子是第十九章之三承诺的专门演示：**同一批判据里中高危行大面积减少、缺口行大面积增加，脚本的结论必须是"不可比"，不能是"变好"。**

```text
PAIR,base,.re15/sc002.txt,rows=34
PAIR,curr,.re15/sc003.txt,rows=16
ROW,L0-LOW,pair_identity_unread,至少一份快照没有机型标识：无法确认这两份产物来自同一台设备，差分结果只做参考
NOTE,identity,快照内不含采集时间戳与构建号，同源性由操作者自行记录
DELTA,build_type,userdebug,none
ROW,L2-WARN,build_type_changed,构建形态由 userdebug 变为 未采到：观察通道的默认收敛随构建形态改变，非发行构建与发行构建不是同一条基线
ROW,L0-LOW,arch_lost,架构判定由 x86 变为判不出：当前快照无法区分'这个平台不导出该项'与'采集被截断'，缺席项一节全部留为缺口
GONE,VULN,l1tf,not_affected
ROW,L0-LOW,vuln_file_gone,l1tf 在当前快照消失（基线为 not_affected）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,mds,vulnerable
ROW,L0-LOW,vuln_file_gone,mds 在当前快照消失（基线为 vulnerable）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,mmio_stale_data,vulnerable
ROW,L0-LOW,vuln_file_gone,mmio_stale_data 在当前快照消失（基线为 vulnerable）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,retbleed,vulnerable
ROW,L0-LOW,vuln_file_gone,retbleed 在当前快照消失（基线为 vulnerable）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,spec_store_bypass,vulnerable
ROW,L0-LOW,vuln_file_gone,spec_store_bypass 在当前快照消失（基线为 vulnerable）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,spectre_v1,mitigated
ROW,L0-LOW,vuln_file_gone,spectre_v1 在当前快照消失（基线为 mitigated）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,spectre_v2,vulnerable
ROW,L0-LOW,vuln_file_gone,spectre_v2 在当前快照消失（基线为 vulnerable）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
GONE,VULN,srbds,mitigated
ROW,L0-LOW,vuln_file_gone,srbds 在当前快照消失（基线为 mitigated）：文件不导出不等于该面被修复，也可能只是换了内核配置或采集域可读性变了
DELTA,vuln_files,8,0
ROW,L2-WARN,vuln_files_fewer,vulnerabilities 读数文件由 8 个减少到 0 个：可读面收窄，本次差分能确认的项随之减少，收窄本身不是改善
DELTA,vuln_vulnerable,5,0
ROW,L1-INFO,vuln_open_count_down_needs_proof,自述 Vulnerable 的读数项由 5 降至 0：数量下降可能来自补丁、来自 cmdline 改动，也可能来自读数口径变化，本脚本不区分
ROW,L0-LOW,cmdline_became_unread,/proc/cmdline 由可读变为读不到：启动参数层的漂移全部不可判定，这一档缺口要在下一轮采集补上
DELTA,CAP,flush_l1d,absent,none
ROW,L0-LOW,cpu_cap_became_unread,flush_l1d 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,ibpb,absent,none
ROW,L0-LOW,cpu_cap_became_unread,ibpb 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,ibrs,absent,none
ROW,L0-LOW,cpu_cap_became_unread,ibrs 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,md_clear,absent,none
ROW,L0-LOW,cpu_cap_became_unread,md_clear 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,ssbd,absent,none
ROW,L0-LOW,cpu_cap_became_unread,ssbd 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,CAP,stibp,absent,none
ROW,L0-LOW,cpu_cap_became_unread,stibp 位由 absent 变为读不到：硬件前提层不可判定，通常是 /proc/cpuinfo 采集退化，不判成能力丧失
DELTA,POL,selinux,Permissive,unread
ROW,L0-LOW,selinux_became_unread,getenforce 由 Permissive 变为读不到：强制模式不可判定，本脚本按缺口处理而不是按未启用处理
DELTA,POL,hidepid,off,unknown
ROW,L0-LOW,proc_mount_became_unread,/proc 挂载行由 off 变为采不到：hidepid 判定留为缺口
DELTA,SYSCTL,kernel.dmesg_restrict,0,unread
ROW,L0-LOW,sysctl_became_unread,kernel.dmesg_restrict 由 0 变为读不到：观察通道收敛状态不可判定，节点被删除与采集权限退化在本脚本里不等价
DELTA,SYSCTL,kernel.kptr_restrict,0,unread
ROW,L0-LOW,sysctl_became_unread,kernel.kptr_restrict 由 0 变为读不到：观察通道收敛状态不可判定，节点被删除与采集权限退化在本脚本里不等价
DELTA,SYSCTL,kernel.perf_event_paranoid,1,unread
ROW,L0-LOW,sysctl_became_unread,kernel.perf_event_paranoid 由 1 变为读不到：观察通道收敛状态不可判定，节点被删除与采集权限退化在本脚本里不等价
DELTA,POL,microcode,microcode : 0x2ef,none
ROW,L0-LOW,microcode_evidence_lost,微码版本行由可读变为读不到：设备侧失去证明微码缓解到达的唯一锚点，只能向厂商索取证据
DELTA,COUNT,zygote_processes,2,none
ROW,L0-LOW,neighbour_list_degraded,进程表只在其中一份快照里采到：同机邻居一节的漂移不可判定，缺侧按采集退化处理
GONE,L3,cmdline_mitigations_off
ROW,L1-INFO,risk_gone_by_coverage,L3 级信号 cmdline_mitigations_off 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
GONE,L3,vuln_vulnerable
ROW,L1-INFO,risk_gone_by_coverage,L3 级信号 vuln_vulnerable 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
NEW,L2,emulator_build
ROW,L2-WARN,drift_new_warn,当前快照新增中等级信号 emulator_build：相对基线新出现的需复核项
GONE,L2,build_type_nonuser
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 build_type_nonuser 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
GONE,L2,cmdline_disable_token
ROW,L1-INFO,risk_gone_by_coverage,L2 级信号 cmdline_disable_token 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
GONE,L2,cpu_x86_cap_absent
ROW,L1-INFO,risk_gone_by_coverage,L2 级信号 cpu_x86_cap_absent 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
GONE,L2,selinux_permissive
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 selinux_permissive 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
GONE,L2,sysctl_dmesg_open
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 sysctl_dmesg_open 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
GONE,L2,sysctl_kptr_open
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 sysctl_kptr_open 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
GONE,L2,sysctl_paranoid_low
ROW,L2-WARN,risk_gone_needs_proof,L2 级信号 sysctl_paranoid_low 在当前快照不再出现，但采集面仍可读：消失有两种解释——问题确实被处理，或判定条件不再触发。需要独立证据（公告、版本号、重采）才能记成已修
GONE,L2,vuln_note_ebpf
ROW,L1-INFO,risk_gone_by_coverage,L2 级信号 vuln_note_ebpf 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
GONE,L2,vuln_note_microcode
ROW,L1-INFO,risk_gone_by_coverage,L2 级信号 vuln_note_microcode 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
GONE,L2,vuln_note_smt
ROW,L1-INFO,risk_gone_by_coverage,L2 级信号 vuln_note_smt 在当前快照消失，且产生它的采集面同时不可读：这是覆盖收窄造成的消失，不能记成风险已闭合
NEW,L0-LOW,build_type_unread;cmdline_unread;cpufreq_unread;cpuinfo_unread;identity_missing;kernel_release_unread;proc_mount_unread;ps_unread;selinux_unread;spl_absent;sysctl_paranoid_unread;sysctl_unread;vuln_dir_unread;
ROW,L0-LOW,drift_new_gaps,当前快照新增缺口信号 build_type_unread cmdline_unread cpufreq_unread cpuinfo_unread identity_missing kernel_release_unread proc_mount_unread ps_unread selinux_unread spl_absent sysctl_paranoid_unread sysctl_unread vuln_dir_unread ：这些是采集面收窄的记号，下一轮优先补
GONE,L0-LOW,neighbor_app_domain_gap;vuln_meltdown_absent_x86;
ROW,L1-INFO,drift_gap_closed,基线缺口信号 neighbor_app_domain_gap vuln_meltdown_absent_x86  在当前快照不再出现：采集面被补上，这一轮的结论可比范围扩大
ROW,L2-WARN,snapshot_thinner,当前快照判定行由 34 条减到 16 条：可比项减少，本差分的没有变化只覆盖剩下的这些项
SUMMARY,high=0,warn=9,gap=25,info=9
```

十二点读法：

1. **先看第 86 行的 SUMMARY：`high=0,warn=9,gap=25,info=9`。** 基线有六条 L3，当前一份的差分结果是 **high=0**。如果这份输出被读成"这台设备的高危项清零"，就是第十九章之三与第十二章列出的最严重误读。**这份输出的正确一句话结论是：当前快照不可判定，因为采集面塌了。**
2. **第 3 行先给准入：`pair_identity_unread`。** 基线有型号、当前没有，连"是不是同一台"都无法确认——这已经先于任何等级判断把整份输出降级成参考。
3. **第 7 行 `arch_lost`（L0）是本篇最关键的一行。** 架构由 x86 变判不出，句子写明后果："当前快照无法区分'这个平台不导出该项'与'采集被截断'，**缺席项一节全部留为缺口**"。第七章 七之四 的解释纪律在这一轮整张作废。
4. **第 8–23 行是八条 `vuln_file_gone`。** 八个读数文件（含五个 Vulnerable）全部消失，**每一条都只配 L0-LOW**。这里的等级安排是刻意的：**"读数不见了"永远不比"读数说 Vulnerable"更值得报警**，因为它同时兼容"被修复"与"采不到"两种解释，而后者在这一轮明显更可能（第 24 行 `DELTA,vuln_files,8,0` 与第 25 行 `vuln_files_fewer` 就是"采不到"的直接证据）。
5. **第 26–27 行给出一条必须拒绝的诱惑。** `vuln_open_count_down_needs_proof`：**自述 Vulnerable 的项由 5 降至 0**，句子明说"数量下降可能来自补丁、来自 cmdline 改动，也可能来自读数口径变化，**本脚本不区分**"。这是 `vuln_open_count_up` 的镜像条目，两者成对存在（第廿六章 K6/K7 就是按这对计数定义的）。
6. **第 28–52 行是采集退化的整块，比样例二更彻底。** cmdline（28）、六个 x86 特性位（29–40）、SELinux（41–42）、`/proc` 挂载行（43–44）、三个 sysctl（45–50）、**微码版本行**（51–52）全部转不可读。其中 `microcode_evidence_lost` 的代价最大：x86 侧唯一能自证"CPU 修复随固件到达"的锚点没了，第十三章的派工项在这一轮**无法闭环**。
7. **第 55–58 行是本样例存在的理由。** 两条 `GONE,L3` → 两条 `risk_gone_by_coverage`（L1-INFO）。脚本在这里做了两件不同的事：**先如实记录"L3 信号消失"这一事实，再给出不许记成已修的理由**——"产生它的采集面同时不可读"。这是第廿三章 廿三之六 那条分流的实际形态：`risk_gone_by_coverage` 与 `risk_gone_needs_proof` 的区别**不在消失的信号等级，而在采集面状态**（前者 L1、后者 L2，等级安排的理由见第廿三章）。
8. **第 61–80 行是同一条规则的对照组：十三条 L2 消失，分成两档。** `cmdline_disable_token`（63–64）、`cpu_x86_cap_absent`（65–66）、`vuln_note_ebpf`（75–76）、`vuln_note_microcode`（77–78）、`vuln_note_smt`（79–80）走 `risk_gone_by_coverage`（L1，因为它们的采集面也塌了）；`build_type_nonuser`（61–62）、`selinux_permissive`（67–68）、`sysctl_dmesg_open`（69–70）、`sysctl_kptr_open`（71–72）、`sysctl_paranoid_low`（73–74）走 `risk_gone_needs_proof`（L2，因为**它们对应的采集面在这一轮仍然可读**——构建类型读到了、SELinux 读到了、sysctl 读到了，只是值不再触发判定）。**这一组对照是本脚本最实用的功能：它把"哪些消失需要独立证据"从"哪些消失只是采不到"里分了出来，而且给后者的等级更高。**
9. **第 62 行那句"消失有两种解释——问题确实被处理，或判定条件不再触发"是本章要读者带走的一句话。** 注意它的处置动作是"需要独立证据（公告、版本号、重采）才能记成已修"，**没有一条消失可以被差分本身记成已修**。
10. **第 59–60 行给出一个反方向：`NEW,L2,emulator_build`。** 当前新出现中等级信号——塌陷采集的那一份反而被判定为模拟器构建。**这类"变差"其实可能是"变模糊"的副产品**（读不到真实型号时，判定退化成弱信号），第廿三章把它记 L2 而不是 L3 正是为此。
11. **第 81–84 行是缺口集合的双向差集。** 十三条新缺口 vs 两条被关闭的基线缺口。**新缺口的处置优先级由第 82 行的句子给出**（"下一轮优先补"），而不是由等级给出——这正是本报告反复强调的：**L0-LOW 不等于低优先级，它等于"现在无法判定"。**
12. **第 85 行 `snapshot_thinner` 判 L2-WARN，句子是本篇的收口：**"可比项减少，本差分的没有变化只覆盖剩下的这些项"。**把这一行放在 SUMMARY 之前是刻意的**：任何消费这份输出的看板都应当把 `snapshot_thinner` 当作一条需要响应的采集事件，而不是一行统计注释。

### 廿四之五 四份样例的横向核对

| 核对项 | 廿四之一 | 廿四之二 | 廿四之三 | 廿四之四 |
| --- | --- | --- | --- | --- |
| 对子关系 | 同型号两轮 | 跨设备 | 塌陷基线 → 可读 | 可读 → 塌陷 |
| 准入行 | `pair_identity_same_model` | `pair_identity_mismatch` | `pair_identity_unread` | `pair_identity_unread` |
| L3 判定行 | 0 | 3 | 0 | 0 |
| L2 判定行 | 4 | 5 | 2 | 9 |
| L1 判定行 | 6 | 2 | 22 | 9 |
| L0 缺口行 | 1 | 11 | 3 | 25 |
| `vuln_state_worse` | — | 触发（第 9 行） | — | — |
| `vuln_state_unclassified_move` | — | 触发（第 11 行） | — | — |
| `risk_gone_by_coverage` | — | — | — | 触发 7 次 |
| `risk_gone_needs_proof` | — | — | 触发 1 次（唯一 L2 消失） | 触发 5 次 |
| `vuln_file_gone` | — | 触发 1 次 | — | 触发 8 次 |
| `snapshot_richer` / `thinner` | richer（20→24） | thinner（28→25） | richer（16→28） | thinner（34→16） |
| SUMMARY | high=0 warn=4 gap=1 info=6 | high=3 warn=5 gap=11 info=2 | high=0 warn=2 gap=3 info=22 | high=0 warn=9 gap=25 info=9 |

**这张表的最后两行是本章唯一允许被单独引用的部分。** 前三类信息（准入行、四个等级的判定行数、SUMMARY）合起来才决定一份差分能不能进趋势账：**准入行不成立时，等级计数没有意义；`gap` 上升时，`high` 下降不构成改善。** 第廿六章的 K6/K7/K9 就是按这三项的组合定义的。
## 廿五、分支覆盖验收：141 个码位全部被触发过

前三章给出的是"脚本在样例上输出什么"。本章回答一个方向相反的问题：**脚本里定义的每一个码位，是否真的在某个样例上输出过。**

这个问题必须单独成章，因为一份从未被触发过的分支和一份正确但罕见的分支，在成品文本上完全长得一样。前者更危险：写反的比较符号、拼错的字段名、被上游改名的读数，都会以"这条规则安静地待在脚本里"的形式留到最后，而读者在十二个章节里被反复教导"没有输出行等于没有异常"——如果脚本自己就带着一段不可能被走到的分支，这条教导会直接把读者推进坑里。所以本章不是附录，**它是前廿四章里那两份脚本可以被引用的前提条件**。

### 廿五之一 验收用的四条命令

验收是两个集合的比对：**静态定义集合**（从脚本源码里提出"这个脚本声明了哪些码位"）和**动态触发集合**（从全部输出里提出"哪些码位真的印出来过"）。提取式如下，都在读者自己的工作树里跑，只读文本、不碰设备：

```bash
# 1. 静态侧：脚本源码里出现过的 row 调用，去掉重复，按码位排序
grep -ohE '\brow L[0-9]-[A-Z]+ [a-z0-9_]+' 采集脚本.sh \
  | sed 's/row //' | awk '{print $2}' | sort -u > 定义.txt

# 2. 动态侧：全部输出里出现过的 ROW 行的码位列
grep -hoE '^ROW,L[0-9]-[A-Z]+,[a-z0-9_]+,' 输出目录/*.txt \
  | cut -d, -f3 | sort -u > 触发.txt

# 3. 死分支：定义了但从来没印出来
comm -23 定义.txt 触发.txt

# 4. 反向污染：印出来了但源码里没有定义（通常意味着提取式或输出格式对不上）
comm -13 定义.txt 触发.txt
```

第 3、4 两条命令**都输出空**才算通过。只跑第 3 条是不够的：如果第 4 条非空，说明动态侧提取到了源码里不存在的名字，此时"死分支为 0"这个结论同样不可信——两个集合可能是靠一个错误的提取式勉强对上的。

### 廿五之二 本篇的输入：十五份设备画像与它们的分工

验收需要输入。本篇的全部输入是 15 份**构造出来的设备画像**——它们不是真机，而是第廿二章那套只读桩（`.wstub15/adb` 把 `adb -s <serial> shell <cmd>` 映射到 `WDATA/<serial>/` 下的文本文件，写操作一律不实现）背后的一组假数据。每一份画像负责满足一批分支的进入条件。**下面这张表是本章所有数字的分母来源，也是"这些数字不能当基准"这句话的证据来源。**

| 画像 | 机型标识 | 架构 | 构建形态 | SPL | `vulnerabilities` 项 / 自述敞口 | L3 / L2 / L0 行数 | 只有这一份能触发的码位 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `sc001` | Pixel 7 | arm64 | user | 2026-10-05 | 5 / 0 | 0 / 0 / 2 | `proc_hidepid_on` |
| `sc002` | Tablet X2 | x86 | userdebug | 2023-11-05 | 8 / 5 | 6 / 16 / 3 | `cmdline_mitigations_off`、`selinux_permissive`、`vuln_note_ebpf` |
| `sc003` | 无 | 判不出 | 未采到 | 无 | 0 / 0 | 0 / 1 / 15 | `emulator_build`、`build_type_unread`、`ps_unread`、`selinux_unread` |
| `sc004` | Box A53 | arm64 | user | 2022-06-05 | 3 / 0 | 0 / 4 / 6 | `kernel_pre_vulnfs`、`vuln_file_empty`、`cpu_aes_absent_cap` |
| `sc005` | 无 | x86 | userdebug | 无 | 4 / 0 | 0 / 9 / 9 | `qemu_prop_odd` |
| `sc006` | Devkit D1 | arm64 | eng | 2026-09-05 | 4 / 2 | 2 / 8 / 4 | `zygote_absent` |
| `sc007` | Phone P9 | arm64 | user | 2026-10-05 | 4 / 1 | 1 / 3 / 5 | — |
| `sc008` | Whitebox Z1 | 判不出 | user | 2024-02-05 | 3 / 2 | 2 / 1 / 8 | `vuln_arch_unknown` |
| `sc009` | Kit K1 | x86 | user | 2026-08-05 | 4 / 0 | 0 / 6 / 5 | — |
| `sc010` | Table T3 | arm64 | user | 2026-06-05 | 4 / 1 | 1 / 0 / 11 | `cpu_cap_unread` |
| `sc011` | Phone B8 | arm64 | user | 2026-10-05 | 3 / 0 | 0 / 3 / 3 | `cmdline_mitigations_nosmt`、`v2_arm_bp_hardening` |
| `sc012` | Pad M2 | arm64 | userdebug | 2026-05-05 | 3 / 1 | 1 / 4 / 3 | `v2_arm_ebpf_reopens` |
| `sc013` | Box I5 | x86 | user | 2026-07-05 | 8 / 3 | 3 / 6 / 2 | `cmdline_off_value`、`vuln_other_state`、`vuln_state_unknown_prefix` |
| `sc014` | Kit K1 | arm64 | user | 2026-09-05 | 0 / 0 | 0 / 0 / 4 | — |
| `sc015` | Kit K1 | arm64 | user | 2026-09-05 | 2 / 0 | 0 / 2 / 5 | — |

（最后一列的"—"表示这一份画像不独占任何码位，但仍然是多个对子的合法输入。L3/L2/L0 三列是脚本 1 在该画像上印出的判定行数，不含 L1。）

这张表里三件事值得在验收之前先说明，否则后面的表会被读错：

- **`sc003` 存在的唯一理由是"采集失败"。**它一份就印出 15 行 L0，其中 4 个码位只有它能触发。这类码位（`*_unread`）判定的是采集动作本身，因此**必须**有一份画像把每个采集面各打断一次；否则脚本里那些"读不到"的分支就是死代码，而读者无法知道。
- **`sc001` 与 `sc014` 是两份"看起来干净"的画像。**`sc001` 是 0 条 L3、0 条 L2，`sc014` 同样是 0/0，但它有 0 个 `vulnerabilities` 读数项。**这两份的差别正是第十九章"读数全部良好为什么不构成结论"的两种情形**：一份是读了但都良好，一份是根本没有可读的读数面。把它们混为"这台设备没问题"，是本章所有数字里最容易被误用的一处。
- **三份 `Kit K1` 画像（`sc009`/`sc014`/`sc015`）是差分脚本同源判定的唯一来源。**它们提供 6 个跨画像的同机型对子；加上 13 份有机型标识画像各自的自对子，就是 19 个 `pair_identity_same_model`。`sc003` 与 `sc005` 机型标识为空，因此凡有一侧是它们的对子都落进 `pair_identity_unread`，共 56 个——这个数字等于 225 减去"两侧都在其余 13 份之内"的 169 个。第廿四章样例一用的 `p014-015` 正是这三份里的一个。

### 廿五之三 本篇的实际验收结果

| 脚本 | 码位数 | 源码调用点数 | 触发数 | 定义有-触发无 | 触发有-定义无 | 触发所用的样例 |
| --- | --- | --- | --- | --- | --- | --- |
| 脚本 1（单份采集判定） | **73** | 75 | 73 | **0** | **0** | 15 份设备画像 |
| 脚本 2（两份快照差分） | **68** | 69 | 68 | **0** | **0** | 225 个对子（15×15，含自对子） |
| 合计 | **141** | 144 | 141 | 0 | 0 | — |

码位数与调用点数不相等，这是提取口径的正常结果，不是笔误，三处差异各自有明确原因，全部在廿五之六逐条列出。样例规模则决定了为什么脚本 2 需要用 225 个对子而不是 15 份样例来验收：**差分脚本的分支不是由"读到什么"触发的，而是由"两侧读到不同的什么"触发的。**同一份画像集合，两两配对才构成它的输入空间。

### 廿五之四 脚本 1 的 73 个码位与触发画像

下表按等级、码位名字排序。`触发画像`列写的是本章廿五之二列出的 15 份设备画像编号（`sc0NN` 去掉前缀）；只要出现过一次就算触发。

| 码位 | 等级 | 触发画像 |
| --- | --- | --- |
| `arm_microcode_face_absent` | L0 | 001、004、006、007、010、011、012、015 |
| `bootloader_absent` | L0 | 002、003、004、005、007、008、009、010、011、012、013、014、015 |
| `build_type_unread` | L0 | 003 |
| `cmdline_unread` | L0 | 003、010 |
| `cpu_cap_unread` | L0 | 010 |
| `cpu_microcode_unread` | L0 | 005、009 |
| `cpufreq_unread` | L0 | 003、005、008、010、014、015 |
| `cpuinfo_unread` | L0 | 003、008 |
| `identity_missing` | L0 | 003、005 |
| `kernel_release_unread` | L0 | 003、008 |
| `neighbor_app_domain_gap` | L0 | 001、002、004、005、007、008、009、010、011、012、013、014、015 |
| `proc_mount_unread` | L0 | 003、006、007 |
| `ps_unread` | L0 | 003 |
| `selinux_unread` | L0 | 003 |
| `spl_absent` | L0 | 003、005 |
| `sysctl_paranoid_unread` | L0 | 003、007、008、010 |
| `sysctl_unread` | L0 | 003、004、005 |
| `vuln_arch_unknown` | L0 | 008 |
| `vuln_core_file_absent` | L0 | 004、005、006、008、015 |
| `vuln_dir_unread` | L0 | 003、014 |
| `vuln_file_empty` | L0 | 004 |
| `vuln_mds_absent_x86` | L0 | 005、009 |
| `vuln_meltdown_absent_x86` | L0 | 002、009 |
| `zygote_absent` | L0 | 006 |
| `bootloader_read` | L1 | 001、006 |
| `build_type_user` | L1 | 001、004、007、008、009、010、011、013、014、015 |
| `cmdline_init_on_free` | L1 | 005、006 |
| `cmdline_mitigations_default` | L1 | 001、006、007、008、009、012、013、014、015 |
| `cmdline_mitigations_nosmt` | L1 | 011 |
| `cmdline_mitigations_set` | L1 | 004、005 |
| `cpu_aes_absent_cap` | L1 | 004 |
| `cpu_aes_cap` | L1 | 001、006、007、011、012、014、015 |
| `cpu_cap_absent` | L1 | 001、004、006、007、011、012、014、015 |
| `cpu_cap_present` | L1 | 001、011、012、013、014、015 |
| `cpu_microcode_line` | L1 | 002、013 |
| `cpufreq_governor` | L1 | 001、002、004、006、007、009、011、012、013 |
| `kernel_release_read` | L1 | 001、002、005、006、007、009、010、011、012、013、014、015 |
| `proc_hidepid_off` | L1 | 002、004、005、008、009、010、011、012、013、014、015 |
| `proc_hidepid_on` | L1 | 001 |
| `qemu_prop_odd` | L1 | 005 |
| `selinux_enforcing` | L1 | 001、004、005、007、008、009、010、011、012、013、014 |
| `spl_read` | L1 | 001、002、004、006、007、008、009、010、011、012、013、014、015 |
| `sysctl_dmesg_restricted` | L1 | 001、004、007、008、009、010、011、012、013、014、015 |
| `sysctl_kptr_hash` | L1 | 001、005、007、008、009、010、011、013、014、015 |
| `sysctl_paranoid_high` | L1 | 001、004、006、009、011、013、014、015 |
| `v2_arm_bp_hardening` | L1 | 011 |
| `vuln_bhb_covered_in_v2` | L1 | 001、009、010、015 |
| `vuln_mitigated` | L1 | 001、002、004、005、006、007、008、009、010、011、012、013、015 |
| `vuln_name_outside_abi` | L1 | 001、004、006、007、009、010 |
| `vuln_not_affected` | L1 | 001、002、005、006、007、009、010、011、013 |
| `vuln_other_state` | L1 | 013 |
| `zygote_shared` | L1 | 001、002、004、005、007、008、009、010、011、012、013、014、015 |
| `build_type_nonuser` | L2 | 002、005、006、012 |
| `cmdline_disable_token` | L2 | 002、004、005、006 |
| `cmdline_off_value` | L2 | 013 |
| `cpu_sb_absent` | L2 | 004、006、007、011、015 |
| `cpu_ssbs_absent` | L2 | 004、006、007、011 |
| `cpu_x86_cap_absent` | L2 | 002、005、009 |
| `emulator_build` | L2 | 003 |
| `kernel_pre_vulnfs` | L2 | 004 |
| `selinux_permissive` | L2 | 002 |
| `selinux_unexpected` | L2 | 006、015 |
| `sysctl_dmesg_open` | L2 | 002、006 |
| `sysctl_kptr_open` | L2 | 002、006、012 |
| `sysctl_paranoid_low` | L2 | 002、005、012 |
| `v2_arm_ebpf_reopens` | L2 | 012 |
| `vuln_note_bhb_partial` | L2 | 007、011 |
| `vuln_note_ebpf` | L2 | 002 |
| `vuln_note_microcode` | L2 | 002、008、013 |
| `vuln_note_smt` | L2 | 002、013 |
| `vuln_state_unknown_prefix` | L2 | 013 |
| `cmdline_mitigations_off` | L3 | 002 |
| `vuln_vulnerable` | L3 | 002、006、007、008、010、012、013 |

一眼能看出的三件事：

1. **21 个码位只由一份画像触发**，分布为 `003` 4 个、`013` 3 个、`004` 3 个、`002` 3 个、`011` 2 个，其余 6 个各 1 个。这个分布不是随机的：`003` 是"采集大面积塌陷"的画像，它一份就印出 15 行 L0 缺口判定，其中 4 个码位只有它能触发；`013` 专门用来触发 `vulnerabilities` 里的特判分支（`cmdline_off_value`、`vuln_other_state`、`vuln_state_unknown_prefix` 三个都只有它一个触发源）；`002` 是跨层组合最严重的那份画像（六条 L3），本篇唯一的启动参数级 L3 码位也只由它触发。
2. **`cmdline_mitigations_off` 是 2 个 L3 码位里只有一个触发源的那个**（`002`）。另一个 `vuln_vulnerable` 有 7 个触发源（`002 006 007 008 010 012 013`）。这组对比正好是两份脚本分工的体现：**内核自述敞口是相对常见的读数，而"启动参数被显式改成关闭缓解"是一个需要有人在 cmdline 上动过手才会出现的读数**——样例集里只造了一份这样的设备。**真实设备群上这一档的分布必须自己采，不能引用本节任何数字。**
3. **缺口类码位压倒性地集中在 L0**：24 个 L0 码位里有 22 个是 `*_unread`、`*_absent`、`*_gap`、`*_missing` 命名，只有 `vuln_arch_unknown` 与 `vuln_file_empty` 两个例外，而这两条例外描述的同样是"读到了但读不出可判定的内容"。它们描述的是**采集动作的边界**，不是设备的状态。这一点在第廿章已经立为口径，本节给的是它在数字上的对应关系。

### 廿五之五 脚本 2 的 68 个码位与触发对子数

差分脚本的每个码位都列出**触发它的对子数**（225 分之几）与**四个样例对子编号**。样例对子的完整读法在第廿四章。

| 码位 | 等级 | 触发对子数 | 代表对子 |
| --- | --- | --- | --- |
| `arch_lost` | L0 | 26 | 001-003 001-008 002-003 002-008 |
| `bootloader_lost` | L0 | 26 | 001-002 001-003 001-004 001-005 |
| `cmdline_became_unread` | L0 | 26 | 001-003 001-010 002-003 002-010 |
| `cpu_cap_became_unread` | L0 | 100 | 001-002 001-003 001-005 001-008 |
| `drift_new_gaps` | L0 | 182 | 001-002 001-003 001-004 001-005 |
| `microcode_evidence_lost` | L0 | 26 | 002-001 002-003 002-004 002-005 |
| `neighbour_list_degraded` | L0 | 28 | 001-003 002-003 003-001 003-002 |
| `pair_identity_unread` | L0 | 56 | 001-003 001-005 002-003 002-005 |
| `proc_mount_became_unread` | L0 | 36 | 001-003 001-006 001-007 002-003 |
| `selinux_became_unread` | L0 | 13 | 001-003 002-003 004-003 005-003 |
| `spl_unread_both` | L0 | 4 | 003-003 003-005 005-003 005-005 |
| `sysctl_became_unread` | L0 | 73 | 001-003 001-004 001-005 001-007 |
| `vuln_file_gone` | L0 | 144 | 001-002 001-003 001-004 001-005 |
| `vuln_pair_dir_gap` | L0 | 4 | 003-003 003-014 014-003 014-014 |
| `zygote_absent_now` | L0 | 13 | 001-006 002-006 004-006 005-006 |
| `arch_gained` | L1 | 26 | 003-001 003-002 003-004 003-005 |
| `bootloader_changed` | L1 | 2 | 001-006 006-001 |
| `bootloader_gained` | L1 | 26 | 002-001 002-006 003-001 003-006 |
| `cmdline_disable_token_removed` | L1 | 57 | 002-001 002-004 002-006 002-007 |
| `cmdline_init_on_free_gained` | L1 | 22 | 001-005 001-006 002-005 002-006 |
| `cmdline_mitigations_off_removed` | L1 | 36 | 002-001 002-006 002-007 002-008 |
| `cmdline_now_readable` | L1 | 26 | 003-001 003-002 003-004 003-005 |
| `cpu_cap_gained` | L1 | 33 | 002-013 004-001 004-006 004-007 |
| `cpu_cap_now_readable` | L1 | 100 | 001-002 001-005 001-009 001-013 |
| `drift_gap_closed` | L1 | 182 | 001-002 001-003 001-005 001-006 |
| `kernel_changed` | L1 | 154 | 001-002 001-004 001-005 001-006 |
| `microcode_evidence_gained` | L1 | 26 | 001-002 001-013 003-002 003-013 |
| `pair_identity_same_model` | L1 | 19 | 001-001 002-002 004-004 006-006 |
| `proc_hidepid_gained` | L1 | 11 | 002-001 004-001 005-001 008-001 |
| `proc_mount_now_readable` | L1 | 36 | 003-001 003-002 003-004 003-005 |
| `risk_gone_by_coverage` | L1 | 37 | 002-003 002-008 002-010 002-014 |
| `selinux_now_readable` | L1 | 13 | 003-001 003-002 003-004 003-005 |
| `selinux_tightened` | L1 | 11 | 002-001 002-004 002-005 002-007 |
| `snapshot_richer` | L1 | 99 | 001-002 001-005 001-006 001-013 |
| `spl_moved_forward` | L1 | 72 | 002-001 002-006 002-007 002-008 |
| `sysctl_now_readable` | L1 | 73 | 003-001 003-002 003-004 003-005 |
| `sysctl_tightened` | L1 | 72 | 001-005 001-006 001-011 002-001 |
| `vuln_file_new` | L1 | 144 | 001-002 001-004 001-005 001-008 |
| `vuln_files_more` | L1 | 87 | 001-002 001-013 003-001 003-002 |
| `vuln_mitigation_text_changed` | L1 | 84 | 001-002 001-005 001-007 001-011 |
| `vuln_open_count_down_needs_proof` | L1 | 73 | 002-001 002-003 002-004 002-005 |
| `vuln_state_better_needs_proof` | L1 | 52 | 002-001 002-005 002-007 002-009 |
| `vuln_state_now_readable` | L1 | 8 | 004-001 004-005 004-007 004-009 |
| `vuln_state_unclassified_move` | L1 | 9 | 001-006 001-007 001-009 001-010 |
| `zygote_count_changed` | L1 | 37 | 001-002 002-001 002-004 002-005 |
| `arch_changed` | L2 | 72 | 001-002 001-005 001-009 001-013 |
| `build_type_changed` | L2 | 114 | 001-002 001-003 001-005 001-006 |
| `cmdline_disable_token_added` | L2 | 57 | 001-002 001-004 001-005 001-006 |
| `cmdline_init_on_free_lost` | L2 | 22 | 005-001 005-002 005-004 005-007 |
| `cmdline_mitigations_value_changed` | L2 | 12 | 002-004 002-005 002-011 004-002 |
| `cpu_cap_lost` | L2 | 33 | 001-004 001-006 001-007 001-011 |
| `drift_new_warn` | L2 | 160 | 001-002 001-003 001-004 001-005 |
| `microcode_changed` | L2 | 2 | 002-013 013-002 |
| `pair_identity_mismatch` | L2 | 150 | 001-002 001-004 001-006 001-007 |
| `proc_hidepid_lost` | L2 | 11 | 001-002 001-004 001-005 001-008 |
| `risk_gone_needs_proof` | L2 | 159 | 002-001 002-003 002-004 002-005 |
| `selinux_disabled_now` | L2 | 28 | 001-006 002-006 003-006 004-006 |
| `selinux_state_changed` | L2 | 24 | 001-015 002-015 004-015 005-015 |
| `snapshot_thinner` | L2 | 99 | 001-003 001-004 001-007 001-008 |
| `spl_moved_backward` | L2 | 72 | 001-002 001-004 001-006 001-008 |
| `sysctl_loosened` | L2 | 60 | 001-002 001-005 001-006 001-012 |
| `vuln_files_fewer` | L2 | 87 | 001-003 001-004 001-005 001-006 |
| `vuln_state_emptied` | L2 | 12 | 001-004 002-004 005-004 006-004 |
| `cmdline_mitigations_off_appeared` | L3 | 36 | 001-002 001-004 001-005 001-011 |
| `drift_new_high` | L3 | 62 | 001-002 001-006 001-007 001-008 |
| `selinux_loosened` | L3 | 11 | 001-002 004-002 005-002 007-002 |
| `vuln_open_count_up` | L3 | 73 | 001-002 001-006 001-007 001-008 |
| `vuln_state_worse` | L3 | 51 | 001-002 001-006 001-007 001-008 |

这张表里最有信息量的是**两端**：

- **最高档不是风险码位。**`drift_new_gaps`（182 对）与 `drift_gap_closed`（182 对）并列第一，它们是缺口的进与出；`drift_new_warn` 160 对、`pair_identity_mismatch` 150 对、`risk_gone_needs_proof` 159 对紧随其后。也就是说，在 15 份画像构成的输入空间里，差分脚本最常说的话是"这两份不能直接比"和"这一项相对基线是新增/消失的"。**这正是第廿三章设计意图得到实现的证据**：一个把大多数结论都写成"需要你补证据"的脚本，才是这个题目下正确的脚本。
- **最低档是 `microcode_changed`，只有 2 对**（`002-013`、`013-002`）。微码版本是一个几乎不动的量，15 份画像里只有两份持有可读且互不相同的微码行，所以它的正反两个方向各出现一次。这个数字要连着一句提醒读：**升级过内核并重启之后，微码读数完全可能一个字符都没变，此时"这一项没有输出行"是正确结果，不是漏判。**同理，`bootloader_changed`（2 对）与 `spl_unread_both`（4 对）、`vuln_pair_dir_gap`（4 对）都属于"应当罕见"的分支，它们的低触发数是特性而不是缺陷。

### 廿五之六 三处计数口径必须写清楚

如果只把上面的表抄走，会在三个地方算不出 141。逐条写在这里：

1. **`selinux_disabled_now` 是同一个码位、两个等级。**脚本 2 的 SELinux 迁移分支里，从 `Disabled` 基线出发是 `L2-WARN`（脚本 2 源码里 `Disabled:*)` 那一行），读到当前为 `Disabled` 是 `L3-HIGH`（紧随其后的 `*:Disabled)` 那一行）。两个方向的严重性不同，因为"基线本来就关"让整份差分作废，而"当前突然关了"是一次真实的状态放宽。**所以脚本 2 是 68 个码位、69 个等级标签**：等级分布 `L0 15 / L1 30 / L2 18 / L3 6` 里，L3 的那 6 个之一同时也被计入 L2 的 18 个。按 `sort -u` 提取码位名会得到 68，按 `row` 调用点提取会得到 69，两个都对。
2. **脚本 1 的 75 个调用点里有两对重复。**`cpu_cap_present`（`L1-INFO`）与 `cpu_cap_unread`（`L0-LOW`）各出现两次——arm64 分支与 x86 分支各写一次。这两个架构的能力位词表不同（`ssbs sb wfxt afp gcs` 对 `ssbd ibrs ibpb stibp md_clear flush_l1d`，见第七章 七之二），但输出码位有意保持同名，**目的是让下游统计不必区分架构**。重复调用点的等级完全一致，所以不存在第 1 条那种跨等级现象。
3. **141 = 73 + 68，取的是唯一码位数。**如果哪天有人把两份脚本合并成一个文件，本节的 69 与 75 会先变，141 不会变；反过来，如果 141 变了而两张表没变，那一定是在表外面新增了码位——那必须回到廿五之一的四条命令重跑。

### 廿五之七 自对子不变量

225 个对子里有 15 个是"同一份快照与自己比"。这 15 个对子的输出是本章唯一一条**可以被读者当自检代码用的断言**：

```bash
for i in 001 002 003 004 005 006 007 008 009 010 011 012 013 014 015; do
  grep -h '^ROW,' 差分目录/p$i-$i.txt
done | cut -d, -f3 | sort | uniq -c
```

本篇样例集上这条命令的输出只有四行，全部码位都属于"同源声明"与"这份快照本身读不到什么"两类：`pair_identity_same_model` 13 次（13 份有机型标识的画像各自一次）、`pair_identity_unread` 2 次（`003`、`005` 无机型标识）、`spl_unread_both` 2 次、`vuln_pair_dir_gap` 2 次（都来自 `003`、`005`、`014` 之间那几份读不到补丁级与安全状态的画像）。

**由此得到两条判据：**

- **自对子的 `L3` 行数必须是 0。**15 个自对子里没有一行 `L3`，也没有任何一条变化类或风险类判定行。如果你跑出自对子里出现 `vuln_state_worse` 或 `selinux_loosened`，那不是脚本严格，那是脚本把自己的输出当成了一次变化。
- **自对子不是"没有输出"。**它一定会留下准入行与缺口行。这是第廿三章之四那条设计的直接推论：`DELTA,VULN` 即使两侧相同也无条件打印，"逐项可比且无变化"这件事本身要留下证据。一个在自对子上返回全空的差分脚本，无法区分"完全一致"和"根本没比"，这在运维上是不可接受的。

顺便给一个用来确认提取式没写错的交叉点：`vuln_pair_dir_gap` 的 4 个对子是 `003-003`、`003-014`、`014-003`、`014-014`——恰好是"两侧 `VULN` 行数都为 0"的全部组合（`003` 与 `014` 的 `vuln=0`，见本章廿五之二表格的读数项列）。如果哪天这个集合里多出别的对子，说明判定条件从"两侧都空"漂到了"任一侧空"，那是脚本改动，不是设备变化。

### 廿五之八 全量输出的行级分布（不要把这些数字当基准）

| 来源 | 判定行总数 | L0 | L1 | L2 | L3 |
| --- | --- | --- | --- | --- | --- |
| 脚本 1 × 15 份画像 | 389 | 85 | 225 | 63 | 16 |
| 脚本 2 × 225 个对子 | 6319 | 1504 | 2565 | 1988 | 262 |

另有 **111 / 225 个对子至少含一行 `L3`**。这两行数字的作用是说明样例集的性质：**它们是被"覆盖分支"这个目标构造出来的，不是被"还原真实设备分布"构造出来的。**所以：

- 不要把 16/389 或 262/6319 当成"你们机房里高危行的预期占比"。真实设备群里这个比例取决于你们是否有人在启动参数上动过手脚，与脚本无关。
- 不要把 `microcode_changed` 只触发 2 次当成"这一项不重要"。它触发少是因为它变动少。
- 上面两张表里所有"代表对子""触发画像"列出的编号，只能用来证明分支可达，**不能用来证明任何一种设备的读数长这样**。

### 廿五之九 两份为构造而存在的画像，引用时必须带这句

样例集里有两份画像的角色是"把分支走到"，不是"描述一台设备"：

- **`sc015`**：它的 `getenforce` 返回 `EnforcingPermissive` ——一个不属于 Enforcing/Permissive 常规口径的值，专门用来触发 `selinux_unexpected`（脚本 1）与 `selinux_state_changed`（脚本 2，见第廿四章样例一）。**真实设备上如果读到这个串，第一解释是采集域串了（比如同时挂了两个策略上下文，或者读到的是别的工具的合并输出），第二解释才是策略真的处于某种非常规状态。**引用这一份的读数当设备证据是不允许的。
- **`sc003`**：采集大面积塌陷的画像，机型、内核版本、`cmdline`、进程表、sysctl、SELinux、微码全部读不到。它的价值是让 15 个 `*_unread` 类码位各得到一次触发机会。**它同时也是本篇里唯一一份 `emulator_build` 画像**（第廿四章样例四里，脚本 2 把它作为"当前快照"时新增了一个 `L2` 的 `emulator_build` 信号，那一次触发是这一份画像在差分侧的用途）。把 `sc003` 的"全部读不到"当成"这台设备什么都查不到所以安全"是第廿七章误区表里的原话，这里只补一句：它同样是**样例**，不是设备。

### 廿五之十 改动这两份脚本时必须重跑的清单

第廿一章与第廿三章的脚本是文本，任何读者都可能为了适配自己的采集环境去改它。改了就必须重跑，否则你改出来的那份不再享有本章的验收结论：
1. `bash -n` 语法检查——这条只保证能解释，不保证任何分支会被走到。
2. 用你手上的全部画像跑脚本 1，得到全部单份输出；再把它们两两配对（含自对子）跑脚本 2。**自对子必须包含在内**，否则覆盖验收会漏掉所有"两侧相同"的分支。
3. 廿五之一的四条命令，双向 `comm` 都为空。
4. 廿五之七的自对子断言：只允许出现同源类与缺口类码位，`L3` 为 0。
5. 全部输出重定向 stderr 到独立文件，逐份确认为空——本篇 15 份单份输出与 225 份差分输出的 `.err` 文件全部是 0 字节。非空 stderr 意味着某个 `$(...)` 里的命令在你的环境上报错，而脚本会**把这种失败读成"读不到"**，从而产出一条看起来正常、实际由错误造成的缺口行。

第 5 条是这一节最容易被跳过、也最容易在真实运维里埋人的一条：**脚本把"命令失败"和"该项不存在"合并成同一个码位，是有意的设计**（第廿、廿三章反复说明设备侧无法区分二者），但正因为合并了，验收时必须单独盯 stderr，否则合并会掩盖的不是信息缺失，而是你自己的采集写错了。

这五条在本篇的最后一次脚本改动上是这样走的：那次改动只动了脚本 2 的 `VULN` 段落（见第廿三章之四，改成无条件打印 `DELTA,VULN` 行），改完把 225 个对子的输出全量重跑一遍，与改动前逐字节比对——**差异 0 个，全部 `.err` 仍是 0 字节**。这就是第 5 条要求在实际操作中长什么样：改动被证明是"输出中性"的，本章的 68 个码位与 225 份差分样例才仍然对它有效。

### 廿五之十一 本章不证明什么

最后把边界钉住，防止本章被读成"脚本已经过完整测试"：

- **本章证明的是可达性，不是正确性。**一个分支被触发过，只说明它能被走到、走到的那次输出的格式与等级和源码里写的一致。它不说明该分支对**真实内核读数**的判断是对的。第廿五之四表里那 21 个单画像码位尤其如此——它们的触发条件各由一份画像满足一次。
- **本篇的桩只模拟"文件里有什么"，不模拟"真实内核为什么是那个值"。**第廿二、廿三、廿四章里所有跨层结论，仍然要靠厂商公告与变更记录闭合，脚本一行都不替你做这件事。
- **红线没有因为覆盖面扩大而移动。**两份脚本全程只读：不 `push`、不 `install`、不删除设备上的任何东西；不打印内存布局、偏移、探针周期数或任何构造步骤；推测执行部分只落在内核自己导出的 `vulnerabilities` 读数、`/proc/cmdline` 与 `/proc/cpuinfo` 能力位上。**这一章给出的 141 这个数字，衡量的是"本报告能对设备说的每句话都有对应的、被走到过的代码"，不是"我们能查出多少漏洞"。**后者从来不由采集脚本决定。
## 廿六、十项运营指标：把六层归属变成可跟踪的量

前面二十五章都在回答"怎么读"。本章把它换成"怎么按月跟踪"。十项指标全部由两个脚本的**现有输出行**定义，不需要新采集动作，也不需要任何探针——**指标的定义式就是脚本输出的行名与字段**，这样每一次评审都能把数字追回到底层文本。

| # | 指标 | 定义式（直接取自脚本输出） | 建议频率 | 达标口径 | 这一项防的失效 |
| --- | --- | --- | --- | --- | --- |
| **K1** | 读数面可得率 | 出现 `COUNT,vuln_files_total` 且值 > 0 的快照数 ÷ 应采快照数；反向信号是 `vuln_dir_unread` | 每次采集 | ≥ 95%，且缺口必须进入 K8 队列 | "这份快照什么都没读到，所以没有风险" |
| **K2** | 自述敞口项数 | `SUMMARY,vuln_bad=` 的机器均值与最大值；逐台看 `ROW,L3-HIGH,vuln_vulnerable` | 每次采集 | **不设目标值**，只设变化阈值 | 把"敞口数下降"当改善（见 K7） |
| **K3** | 启动参数显式关闭项数 | `cmdline_disable_token` + `cmdline_mitigations_off` + `cmdline_off_value` 三类 L2/L3 行的出现台数 | 每次采集 | 目标 0；非 0 必须能写出改动来源 | 性能调优顺手关缓解，且无人记录 |
| **K4** | 非 ABI 读数占比 | `COUNT,vuln_outside_abi` ÷ `COUNT,vuln_files_total` | 每次采集 | 只作口径注记，不作评级 | 把厂商自行导出的文件名当成可核验读数（第七章） |
| **K5** | 观察通道四项齐备率 | 同一快照内同时出现 `selinux_enforcing`、`sysctl_paranoid_high`、`sysctl_kptr_hash`、`sysctl_dmesg_restricted` 的台数 ÷ 可读台数 | 每次采集 | 趋势向上；单台下降即触发脚本 2 复核 | 只看 `vulnerabilities` 而忽略 H4 层真正管着观察者 |
| **K6** | 消失信号的证据化率 | `risk_gone_needs_proof` 行数中，能在 30 天内附上一条变更依据（公告号、版本号、cmdline 变更记录）的比例 | 每月 | ≥ 90%，未附依据的一律继续挂账 | **"这一项不再报警 = 已修"** |
| **K7** | 覆盖收窄发生率 | `snapshot_thinner`、`vuln_files_fewer`、`risk_gone_by_coverage` 任一出现的差分对数 ÷ 差分总数 | 每月 | 目标 0；出现即视为采集问题而非安全问题 | 把采集退化读成整改成果（第廿四章样例二） |
| **K8** | 缺口补采周期 | `drift_new_gaps` 列出的每个码位，到对应 `drift_gap_closed` 出现之间的天数中位数 | 每月 | 中位数 ≤ 2 个采集周期 | 缺口长期挂着，最后被当成"该项不适用" |
| **K9** | 可比基线留存率 | 每台在册设备至少持有两份"机型、构建形态、架构三项一致"的快照的比例 | 每季 | ≥ 90% | 只有单份快照却做"有没有变好"的判断 |
| **K10** | 不可证层的证据通道覆盖率 | 对 H1（arm64 全部机型与 x86 无微码行机型）、H2、H6 三层，是否已建立厂商证据通道（公告订阅、BSP 说明、库审计）的比例 | 每季 | 三层各自 ≥ 80%，且写明证据来源人 | 设备侧读不到就默认不存在（第十九章第一种"全绿"） |

四点使用说明：

1. **K2 不设目标值。** "敞口项数为 0"不是可达成的运营目标——它取决于芯片与内核，不取决于安全团队。给它设阈值只会诱导把读数改好（改 cmdline 就能做到）而不是把风险降低。**这一条是整个指标体系里最容易被写错的一格。**
2. **K6 与 K7 是一对。** 前者管"消失了的是不是真被修了"，后者管"消失是因为采集面窄了"。两式共用脚本 2 的输出行，但结论方向相反：K7 升高时，K6 的分母会虚假下降。**只看 K6 会把采集退化误判成整改效率提升。**
3. **K5 是 Android 特有的指标。** 在服务器口径下 `perf_event_paranoid` 是主闸门；在 Android 上平台把这件事交给了策略域（第十一章），所以"策略在强制模式 + 三个 sysctl 在收敛档"这个组合才有意义，单看任何一个都会误判。
4. **K10 是唯一一个"设备侧完全没有数据"的指标。** 它只能靠流程与台账统计，这正是它必须存在的理由：**本报告能自证的只有 H3、H4 两层，其余四层如果不去建立证据通道，就永远没有任何数字描述它们。**

## 廿七、十二个常见误区

| # | 误区 | 为什么会这么读 | 正确读法（对应章节） |
| --- | --- | --- | --- |
| 1 | **"这台手机有没有 Spectre？"** 当成一个能回答的是非题 | 日常语言把芯片设计、内核实现、启动配置、有没有观察者四件事压进一个词 | 管理单位是"观测通道 × 层 × 责任人 × 设备侧证据"四元组（第一章） |
| 2 | 把 `vulnerabilities` 的读数读成**芯片是否受影响** | 文件名就叫 CPU 漏洞名，直觉上像在说芯片 | 它是内核在启动时算出的结论；上游对 `old_microcode` 明写运行时微码更新不改变状态（第六、十三章） |
| 3 | 读到一片 `Not affected` 就判"这台机器免疫" | 这是最像好消息的字样 | 通用回退实现（`CPU_SHOW_VULN_FALLBACK` 弱别名到 `cpu_show_not_affected()`）本身返回 `"Not affected\n"`——文件存在且读"Not affected"可能只是回退，不是厂商判定（六之三） |
| 4 | `/proc/cpuinfo` 里没有 `csv2` ⇒ 这台 arm64 没有 CSV2 | 特性位看起来是硬件能力的清单 | `CSV2/CSV3` 是 `FTR_HIDDEN`，不打印；状态要读 `spectre_v2` 的说明文字（第七之二） |
| 5 | arm64 没有 `spectre_bhb` 文件 ⇒ BHB 面不存在 | 拿 x86 的文件清单当全集 | arm64 把 BHB 拼在 `spectre_v2` 的后缀里（`, BHB` / `, but not BHB`），没有独立文件（第七之三） |
| 6 | `perf_event_paranoid=-1` ⇒ 采样面开放 | 这是 Linux 通用口径 | Android 平台在**有 LSM 钩子时**故意写 -1，把准入交给 SELinux 域；判级必须与 `getenforce` 绑在一起（第十一、二十一章） |
| 7 | 两份快照逐项对比，**某项不再报警就算修好** | diff 的自然读法 | 脚本 2 分两种消失：`risk_gone_by_coverage`（采集面窄了）与 `risk_gone_needs_proof`（要独立证据）；两者都不是"已修"（第廿三、廿四章） |
| 8 | cmdline 里没有 `nospectre_v2` ⇒ 缓解没被关 | "没写就是没做"的直觉 | `spectre_v2_parse_cmdline()` 对无法识别的取值只 `pr_err` 一行就返回 0；写了但被忽略，与没写在读数上不可区分（八之三） |
| 9 | 拿两台不同架构设备的同名文件"比等级" | 同名容易让人以为同口径 | x86 与 arm64 的文件集、取值词表、缓解形态都不等价；脚本 2 在架构变化时先给 `arch_changed` 再谈逐项（第七、廿三之三） |
| 10 | 把 `vuln_files_total` 当"这台设备的漏洞数" | 计数行看起来像清单总数 | 里面含厂商自行导出的名字（样例 A 是 5 个里有 2 个），要减 `vuln_outside_abi` 才是可核验口径（六之三、K4） |
| 11 | 认为"缓解项可以像模块一样单独下发更新" | Mainline 的常识外推 | 缓解在编译期/启动期/微码三处落地，没有可独立更新的模块载体；第十八章给出逐层更新通道 |
| 12 | 认为**安全元件天然免疫侧信道** | "密钥在 TEE 里"被读成"密钥不可测" | 要拆三问：硅侧是否同核、内核读数是否覆盖、实现层是否常数时间；只有第二问答案是"不覆盖"，第一、三问都要单独取证（第十七章） |
## 廿八、趋势与分角色结论

### 廿八之一 六个可以从公开文档直接读出的趋势

1. **读数面的文件清单仍在增长，而不是收敛。** 上游 `drivers/base/cpu.c` 的注册数组在本文快照里列出 19 个文件名（第六章），其中 `indirect_target_selection`、`reg_file_data_sampling`、`vmscape` 是按较新的缺陷类别补进去的条目；同一份树里 ABI 文档页的 `What:` 行只列到 17 个，落后于数组（六之二第 4 点）——**两处口径不相等本身就是清单还在动的证据**。**趋势不是"漏洞被发现完就稳定"，而是"每新增一类通道就新增一个读数文件"**。对设备侧的直接后果是：**同一份检查表在内核升级后覆盖面会变大，而变大本身会让"差分结果"看起来像劣化**（脚本 2 用 `vuln_files_more` + `NEW,VULN` + `vuln_file_new` 三行专门标注这种新增，不给风险等级）。
2. **"文件存在"与"厂商判定过"这两件事正在进一步分离。** 通用注册路径里的弱别名回退（`CPU_SHOW_VULN_FALLBACK` → `cpu_show_not_affected()`）意味着**一个没有架构专属实现的文件名照样能读出 `Not affected`**（六之三）。随着注册的文件越来越多、架构专属实现跟不上，这种"合法但无信息"的读数的比例只会上升。**设备侧要防的不是读到坏消息，而是读到一条看起来像好消息的回退值。**
3. **Android 侧把"谁能观测"继续留在策略域，而不是 sysctl。** 平台的行为已经有明确的公开写法：有 LSM 钩子时把 `perf_event_paranoid` 写成 -1、由 SELinux 决定准入（第五章引用的 `init.rc` 注释）；策略侧用 `can_profile_perf({...})` 的域集合而不是数值闸门（第十一章）。**这意味着任何照搬服务器 Linux 的"paranoid ≥ 3 才算收紧"的 Android 检查表都会误判**——本报告因此把 K5 定义成"策略在强制模式 + 三个 sysctl 在收敛档"的组合指标。
4. **可观测范围正被平台主动收缩到"请求者自己的进程"。** Android 16 的 Profiling 模块对结果的处理写得很直接：**"Results are redacted to contain information relating to the requesting process only and delivered to the app's storage directory for further processing by the app."**（参考资料 22）。这与第十七章"设备侧唯一能读到的邻居信息只剩进程名"是同一方向的两个证据：**平台在把跨进程观测能力收回各自的域内**，而这恰好是 H4 层能做的、且不需要重编译的那一类整改。
5. **公告口径从"某个漏洞"转向"共享的微架构状态"。** 2025 年那批间接分支预测器公告的措辞已经不再是"某某指令序列会被误测"，而是**"Exposure of sensitive information caused by shared microarchitectural predictor state that influences transient execution in the indirect branch predictors..."**（参考资料 19）。CWE-1303 的定义句同样落在"共享资源的争用开出隐蔽信道"上（参考资料 17）。**趋势是责任描述从"补丁修某个洞"移到"某类状态被谁共享"**——这正好是本文第四元组（观测通道）而非漏洞编号的口径。
6. **缓解项不会获得可独立下发的更新载体。** 第十八章按层列出的更新通道（微码随固件包、内核编译期随 boot image、启动参数随 boot image 或 recovery、策略随 sepolicy 与 OTA）里没有一个是模块级的。**趋势是把这类风险的整改周期牢牢绑定在整机 OTA 与 BSP 更新上**，而这两条通道的时延由厂商治理决定（参见补丁传导链那一篇）。**这条对读者的实际含义是：把"微架构缓解"放进季度迭代计划是无效的，它只能进固件与 OTA 的节奏表。**

### 廿八之二 分角色结论

| 角色 | 现在就该做的事 | 不该再做的事 |
| --- | --- | --- |
| **SoC / BSP 内核集成工程师** | 在发版说明里逐文件写出 `vulnerabilities` 读数的**取值与实现名**，并给出该版本对应的 cmdline 默认档；`CONFIG_GENERIC_CPU_VULNERABILITIES` 与各架构专属实现是否编入要说清 | 只发一句"已合并上游缓解"；不发微码版本对照表 |
| **整机厂商固件与更新治理** | 为 x86 机型维护 `microcode` 行版本表；为 arm64 机型**建立"CPU 侧修复随哪个固件包到达"的可查记录**，因为设备侧没有自证锚点 | 让 SPL 充当缓解到达性证据（第一章与第十三章都否掉了这种用法） |
| **密钥与密码实现团队（KeyMint / StrongBox / TEE 侧）** | 按第十五章的六项自查清单过一遍自有实现：分支、查表、时限、缓存驻留、共享工作区、常数时间来源 | 用"密钥在安全元件里"作为侧信道免疫结论（第十七章的三问要分开答） |
| **应用沙箱与 sepolicy 维护者** | 把 `can_profile_perf` 的域集合、`appdomain` 对他进程文件的 neverallow、`untrusted_app_all` 的 dontaudit 三处当作**侧信道缓解的一部分**来评审，而不是当作权限卫生 | 以为收紧 `perf_event_paranoid` 数值就等价于收紧 Android 上的采样面 |
| **漏洞治理与补丁管理负责人** | 采用第二十六章 K1–K10；把 K6/K7 成对上报，把 K10 作为"设备侧无法自证"的补偿指标 | 用"敞口项数为 0"作目标；用单份快照做趋势 |
| **设备风控与取证工程师** | 留存脚本 1 的完整文本输出作为时间点的姿态证据；判定只用脚本 2 的差分，并把 `risk_gone_needs_proof` 挂账到证据到位 | 把"这次没报警"写进结案；在两份不同机型的快照之间做收紧/劣化结论 |

## 廿九、常见问题

**Q1：一个普通应用能不能靠侧信道读出另一个应用的密钥？**
本文不回答"怎么做"，只回答"哪些条件必须同时成立"。按第三章的六层：硅侧要有该通道（H1）、内核侧要没有生效缓解或被参数关掉（H2/H3）、该应用要能安排观测（H4：能不能采样、能不能读符号与日志）、要能与目标共享足够近的微架构资源（H5）、而目标实现本身要把密钥位编进可观测的时长或访存模式（H6）。**这五件事分属五个责任人，任何单层收紧都能把这条链断掉；这也是为什么本文的结论都是归属矩阵而不是检测规则。**

**Q2：脚本 1 全绿，能不能写进合规报告说"无风险"？**
不能。第一，它自证的范围只有 H3、H4 与（x86 的）H1 版本号；第二，`Not affected` 可能来自弱别名回退；第三，H6 完全不在采集面内。第十九章把这四种"全绿"情形分列表说明，K10 就是为此存在的补偿指标。

**Q3：为什么脚本不直接读 `/proc/<pid>/` 或跑一次计时测量来"证明"信道存在？**
非目标。其一，本文范围声明排除任何构造观测的操作性内容；其二，跨进程读 `/proc` 会引入权限与隐私问题，采集固定从 shell 域发起且不触碰任何进程的非公开条目；其三，一旦把"能不能测到"作为判定，脚本结论会随噪声条件变化（第十二章的 governor 行就是这个风险的显式记录）。**本报告的判定对象是"缓解由谁负责、设备侧能证明到哪一层"，不是"这台机器能不能被观测"。**

**Q4：`vulnerabilities` 里为什么有我不认识的名字？**
两种可能，脚本已经替你分开记：厂商内核或补丁集自行导出的名字（`vuln_name_outside_abi`，L1）与本就不该出现在这个架构上的名字（缺席项一节按架构给不同码位，L0）。**处理动作是减法而不是分类**：统计口径里减掉 `COUNT,vuln_outside_abi`（样例 A 是 5 个里 2 个）。

**Q5：两份快照之间少了一个文件，是不是说明这个问题被修了？**
不是。脚本 2 用两条互斥的码位拒绝这个读法：采集面同时不可读记 `risk_gone_by_coverage`，采集面仍可读记 `risk_gone_needs_proof`。后者也**只要求一条独立证据**（公告号、版本号、重采），不判"已修"。

**Q6：`mitigations=off` 和高一级的 `Vulnerable` 读数，哪个更该先处理？**
`mitigations=off` 先处理。它是**人为动作**（能改 boot image 的人都能引入），且按上游文档是"按架构聚合的关闭开关"，一次改动覆盖全部内核侧缓解；`Vulnerable` 读数里有一部分是芯片本身没有抑制入口，责任在 H1，动作是索取固件与公告而不是改配置。脚本 1 的两个 L3 码位正是按"动作类 / 结果类"分开的。

**Q7：SELinux 是 Permissive，但 `vulnerabilities` 全绿，要不要紧？**
要紧，而且是两个不同方向的要紧：Permissive 使 H4 层的所有"收敛由策略决定"的判定同时失效（脚本 1 的 `selinux_permissive` L2、脚本 2 的 `selinux_loosened` L3），而全绿只说明 H3 层没有敞口。**观察通道放开 + 内核侧缓解到位，仍然不等于密钥实现层安全（H6 不受任何读数覆盖）。**

**Q8：能不能用这两个脚本做量产线检查？**
可以，但要注意三点：其一，脚本只读，适合在装机验收时跑；其二，判定必须有**留存基线**（K9），单份快照只做记录不做结论；其三，非发行构建（`build_type_nonuser`，L2）与模拟器形态（`emulator_build`，L2）在产线上会大量出现，**这两类快照不能作为"达标/不达标"的依据**，只能标形态。

**Q9：arm64 完全没有微码锚点，那 CPU 侧修复到底怎么确认？**
目前只有一条路：向厂商索取，并把它记进 K10 的证据通道。第十三章给出的理由是结构性的——arm64 侧没有 x86 那样的 `microcode` 行，`vulnerabilities` 也不区分微码是否到位，而**运行时微码更新不改变读数**这一条在上游文档里是明确的（参考资料 4）。`ro.boot.bootloader` 只能证明出厂镜像标识，不证明微码级别（脚本 1 的 `bootloader_read` 文字就是这么写的）。

**Q10：为什么脚本 2 不做成"连上设备自动比对"？**
刻意解耦。漂移判定的前提（同源性、可比性）必须由操作者的记录提供，脚本无法证明；把采集与判定合并会让每次重判都接触设备，从而污染"这两份产物是否同一条件下采集"这个前提。**离线文本比对还带来第二个好处：任何一份历史快照都可以被重新判定，而设备早已不在你手上。**

**Q11：这套东西和 [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html) 是什么关系？**
那一篇回答"谁能读谁的文件描述符、`/proc` 条目和内存"；本篇只取其中与**观测通道**相关的策略行（第十一章引用了三处具体规则），并把它们当作侧信道缓解的一部分来读。两份报告的采集面不重叠：那一篇看沙箱边界，这一篇看内核自述的缓解状态与硅侧责任归属。

**Q12：本报告为什么不给任何"信道容量"或"需要多少次采样"的数字？**
这类数字本身就是操作性知识的一半，而且它们依赖探针构造。更重要的是，本文的核心结论不依赖它们：**每一层有责任人、每一条通道有可核验的到达证据、每一个读不到的字段被记成缺口而不是通过。**
## 三十、参考资料

以下条目为本次核验实际取得逐字内容的公开来源，编号供正文引用；引文为原文（英文）摘录，破折号后为中文口径说明。截断处以"（原文截断）"标注；未取得整句的条目明确写出"本次不据此立论"。**内核源码与内核文档条目中的长句在原文里是被换行与缩进拆开的，本文按阅读顺序合并了空白，不改动词序与用词。**

1. Linux kernel，`Documentation/ABI/testing/sysfs-devices-system-cpu`："The files are named after the code names of CPU vulnerabilities. The output of those files reflects the state of the CPUs in the system." — 设备侧 `vulnerabilities` 目录的 ABI 定义句：文件名是漏洞代号，**内容是内核给出的状态**，不是芯片型号。第六章、第七章（对应第一章"三套口径"的落点）。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-devices-system-cpu
2. `arch/arm64/kernel/proton-pack.c`："We try to ensure that the mitigation state can never change as the result of onlining a late CPU." — arm64 侧缓解状态的成立条件被写在代码注释里：状态在启动时锁定，热插晚到的 CPU 不应改变它。这是第八章"缓解档位不是运行时量"的直接依据。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/proton-pack.c
3. `arch/arm64/kernel/cpuinfo.c`：`static const char *const hwcap_str[]` 的表项，本地按序核对到的一段为 `fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 sm3 sm4 asimddp sha512 sve asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg gcs ls64 dcpodp … bti mte ecv afp rpres mte3 sme … wfxt`（表在 `KERNEL_HWCAP_*` 下标上继续延伸，此处按省略号截断）。— `/proc/cpuinfo` 的 Features 行只打印这张表里的名字，因此"读不到某能力位"与"内核没实现该能力位"两件事在设备侧不可分。第七章 七之二。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/cpuinfo.c
4. `Documentation/admin-guide/hw-vuln/old_microcode.rst`："Just like all the other hardware vulnerabilities, exposure is determined at boot. Runtime microcode updates do not change the status of this vulnerability." — 上游把"运行时更新微码不会改变读数"写成了明文。第十三章整章的成立前提。https://docs.kernel.org/admin-guide/hw-vuln/old_microcode.html
5. AOSP `system/core/rootdir/init.rc`："In this case, the system-wide perf_event_paranoid sysctl is set to -1 (unrestricted use), and the SELinux policy is used for controlling access."；以及 "This property can be set only on userdebug/eng. See neverallow rule in /system/sepolicy/private/property.te ." — 平台把 `perf_event_paranoid` 的三档写入条件、以及 `kptr_restrict` 只允许在 `userdebug/eng` 上放开，都写在同一份 `init.rc` 里。第十一章、第十四章之二。https://android.googlesource.com/platform/system/core/+/refs/heads/main/rootdir/init.rc
6. AOSP `system/sepolicy/private/`：``# profiling needs to read /proc/sys/kernel/perf_event_max_sample_rate` 与 `allow domain proc_perf:file r_file_perms;`（`domain.te`）、`# Allow profiling using perf_event_open by traced_perf.` 与 `can_profile_perf({ dumpable_domain -app_zygote -hal_configstore_server -webview_zygote -zygote })`（`domain.te`）、`# A different form of hidepid=2 like protections` 与 `neverallow appdomain { domain -appdomain }:file no_w_file_perms;` / `neverallow { appdomain -shell } { domain -appdomain }:file no_rw_file_perms;`（`app.te`）。— Android 上"谁看得见谁"主要由策略域而不是挂载选项决定，这三条是第十一章的核心证据。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/
7. Intel® SA-00088 公告要点（Spectre 变体 2 / 侧信道分析）。— 本次未能逐字重取，见本章末"边界 1"。第二、三章用它标注公告归属，结论均以设备侧读数独立成立。https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-technical-advisories.html
8. Intel® SA-00233 公告要点（L1 Terminal Fault / L1TF）。— 同上，本次未能逐字重取，见边界 1。第五章家族表按公告名落位。https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-technical-advisories.html
9. source.android.com《Security Update Information》："Local attacks require the attacker to have prior access to the victim." — Android 自己的攻击面分类口径把"本地"定义为"已经拿到设备"，这正是第一章把侧信道从"远程漏洞"里分出来的依据。第一章、第九章。https://source.android.com/docs/security/overview/updates-resources
10. SpectreAttack 项目主页（The Spectre and Meltdown attacks 白皮书页）："Spectre breaks the isolation between different applications. It allows an attacker to trick error-free programs, which follow best practices, into leaking their secrets." — 原始定义句；本文只在"分类与命名"意义上引用，不引用其攻击构造部分。第一章。https://spectreattack.com/
11. MDS Attacks 项目主页："Unlike existing attacks, our attacks can leak arbitrary in-flight data from CPU-internal buffers (Line Fill Buffers, Load Ports, Store Buffers), including data never stored in CPU caches." — "从未进入缓存的在途数据也会被读出"是 MDS 族与前几族的关键差别，也是第十五章把密钥实现层单列的理由。第五章、第十五章。https://mdsattacks.com/
12. CWE-208：「Observable Timing Discrepancy」定义句 "Two separate operations in a product require different amounts of time to complete, in a way that is observable to an actor and reveals security-relevant information about the state of the product, such as whether a particular operation was successful or not." — 时序侧信道（密钥实现类）的对齐口径，第十章。https://cwe.mitre.org/data/definitions/208.html
13. `Documentation/admin-guide/hw-vuln/mds.rst`："If a CPU is affected and the microcode is available, then the kernel enables the mitigation by default."；以及 "The processor is vulnerable but microcode is not updated. The mitigation is enabled on a best effort basis." — MDS 默认策略是条件句，且微码缺失时是 best effort。第一章、第十三章。https://docs.kernel.org/admin-guide/hw-vuln/mds.html
14. `Documentation/admin-guide/kernel-parameters.txt`，`spectre_v2=` 条目："Selecting 'on' will, and 'auto' may, choose a mitigation method at run time according to the CPU, the available microcode, the setting of the CONFIG_MITIGATION_RETPOLINE configuration option, and the compiler with which the kernel was built." — "选了 on 不代表用了哪套缓解"，本句是第八章与第十四章之二那条判读规则的直接出处。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/kernel-parameters.txt
15. 同文件的 `mitigations=` 条目："… is supported if and only if the kernel was built with CPU_MITIGATIONS=y."；以及 `nospectre_bhb [ARM64,EARLY] Disable all mitigations for Spectre-BHB (branch history injection) vulnerability. System may allow data leaks with this option." — 启动参数层有两个必须检查的前提（编译期开关、arm64 的 BHB 专用开关），第廿五章样例里 `cmdline_mitigations_off` 的判读依赖这两句。第一章、第八章、第十一章。URL 同参考资料 14，条目不同。
16. CWE-200「Exposure of Privileged Information to Unauthorized Actor」。— 本文只在"信息暴露给不该看到它的观察者"这个通用意义上使用，用于第十章的类别对齐；该条目的映射标注为 `DISCOURAGED`，因此本文不把它当作缺陷定级依据。https://cwe.mitre.org/data/definitions/200.html
17. CWE-1303：「Non-Transparent Sharing of Microarchitectural Resources」定义句 "Contention for shared resources between different contexts opens covert channels that allow malicious programs executing in one context to recover information from another context." — 这是本篇"观测通道 × 层 × 责任人 × 设备侧证据"四元组里"共享微架构资源"一格的正式口径。第一章、第十章、第二十八章。https://cwe.mitre.org/data/definitions/1303.html
18. Intel® SA-00270 公告要点（TAA / TSX Asynchronous Abort）。— 本次未能逐字重取，见边界 1。第五章家族表按公告名落位。https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-technical-advisories.html
19. Intel® SA-01247 公告要点（共享分支历史预测器 / GDS 族）。— 本次未能逐字重取，见边界 1；其"共享预测器状态导致信息泄露"的表述与参考资料 17 的 CWE 定义同构，本文以 CWE 口径立论。第五章、第二十八章。https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-technical-advisories.html
20. AMD-SB-1003（`Speculative Code Store Bypass and Floating-Point Value Injection`，2025-02-06 版）："Potential software vulnerabilities can be mitigated by inserting a LFENCE between the overwriting of the instructions and the execution of the overwritten instructions." — AMD 把缓解手段写成"软件侧插入 LFENCE"，即该族的收敛点在编译器/内核代码而不是启动参数；第十三章的"微码不是唯一到达路径"由此得到厂商侧证据。https://www.amd.com/en/resources/product-security/bulletin/amd-sb-1003.html
21. AOSP `simpleperf/doc/android_application_profiling.md`："However, security restrictions mean that only apps with android::debuggable set to true can be profiled."；"If you are on a rooted device, you can profile any app." — 观察者侧可用范围的官方表述：非 debuggable 应用默认不可被剖析，除非设备已 root。第十五章 H5 层。https://android.googlesource.com/platform/system/extras/+/refs/heads/main/simpleperf/doc/android_application_profiling.md
22. source.android.com《Modular system Profiling》："Apps can request profiling using Java heap dump, heap profile, stack sampling, or system trace, and set select configurations for each."；"Results are redacted to contain information relating to the requesting process only and delivered to the app's storage directory for further processing by the app." — 平台把可观测范围收缩到请求者自己进程之内，是本篇"平台侧收敛观察者"判断的直接依据。第一章、第十五章、第十七章、第二十八章。https://source.android.com/docs/core/ota/modular-system/profiling
23. `arch/x86/kernel/cpu/bugs.c`：`pr_err("Ignoring unknown spectre_v2 option (%s).", str);` 与 `nospectre_v2_parse_cmdline()` 里 `nospectre_v2 = true; spectre_v2_cmd = SPECTRE_V2_CMD_NONE;`。— 无法识别的启动参数只打印一行错误并回落到默认，不报错退出；这是第七章"读不到不等于关掉了"与第十四章之二的代码依据。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/cpu/bugs.c
24. `drivers/base/cpu.c`：`#ifdef CONFIG_GENERIC_CPU_VULNERABILITIES`、`return sysfs_emit(buf, "Not affected\n");`、`__attribute__((weak, alias("cpu_show_not_affected")))`，以及 `cpu_root_vulnerabilities_attrs[]` 登记的 19 个读数文件名。— 内核配置没打开时，这些文件会以"弱别名 → Not affected"的形式存在或被整体省略；第六章据此区分"文件不导出"与"该面已缓解"。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/cpu.c
25. CWE-1063：映射标注原文 "Vulnerability Mapping : PROHIBITED / This CWE ID must not be used to map to real-world vulnerabilities." — 该 ID 只能用于创建类 CWE 映射，不能直接标注真实缺陷；第十章的"一处必须纠正的常见错标"即指此类误用。https://cwe.mitre.org/data/definitions/1063.html
26. source.android.com《Modular system (Mainline)》："Android 10 introduced Mainline (formerly called modular system components on this site)."；"Mainline modularizes some Android system components and enables them to be updated outside of the normal Android release cycle."；"End-user devices can receive Mainline updates from the Android Google Play system update feature (which runs on the Google Play Store infrastructure) or through a partner-provided over-the-air (OTA) mechanism." — 第十八章"缓解项为什么不能走 Mainline"的边界由这三句界定：可独立更新的单元与其分发通道都在这里写明。https://source.android.com/docs/core/ota/modular-system
27. `arch/arm64/kernel/cpufeature.c`：`ftr_id_aa64pfr0[]` 中 `ARM64_FTR_BITS(FTR_HIDDEN, FTR_NONSTRICT, FTR_LOWER_SAFE, ID_AA64PFR0_EL1_CSV3_SHIFT, 4, 0)` 与同名 CSV2 条目。— **CSV2/CSV3 被登记为 `FTR_HIDDEN`，因此不会出现在 `/proc/cpuinfo` 的 Features 行**。第七章 七之二"看不到 csv2 不构成任何判断依据"的代码依据。https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/cpufeature.c

### 未公开文档化 / 本次未逐字核验的边界

下面十四条是本文**不立论**的部分。编号是固定的，正文（第五、六、九、十一、十三、十四、十七章）按这些编号回指，改动编号会断链。

1. **Intel 公告正文本轮无法逐字重取。**参考资料 7、8、18、19 四条在核验期间以 HTTP 直接访问一律返回 403，浏览器访问被重定向到开发者站点首页（正文已不存在于旧路径）。这四条的引文来自先前核验时的记录，**本轮不据其措辞立论**：正文凡涉及这四条之处，结论均改由设备侧读数（参考资料 1、13、23、24）、CWE 口径（参考资料 12、16、17、25）或 AMD/内核文档（参考资料 4、20）独立支撑。
2. **`docs.kernel.org` 不渲染 ABI 目录。**参考资料 1 的 `sysfs-devices-system-cpu` 在文档站上返回 404，只能取 git 树或 kernel.org 的裸文档副本。读者复核时若先搜文档站会以为该文件不存在。
3. **AMD 对 Spectre v2 的微码表述没有可逐字引用的公开句。**AMD 侧给出的可引用表述是参考资料 20 的软件侧 LFENCE 缓解；"AMD 平台上 Spectre v2 由微码还是由软件序列收敛"这一问，本次没有找到厂商侧的明文，本文按"不可判定"处理，并在第十三章把它列为需要向厂商索取证据的层。
4. **Arm 没有公开文字说明 CSV2/CSV3 是否应当对外可见。**参考资料 27 只证明内核把它们登记为隐藏；"隐藏"是内核的实现选择，不是架构的定义。第七章据此只给出"看不到不构成依据"这一条消极结论。
5. **Android 安全公告没有为"侧信道/微架构"设立独立漏洞类型。**公告按 CVE 与组件分型，不按 CWE-1303 这类资源争用维度分型。第九章的类型学与第十七章的归属表因此是本文的组织方式，不是平台的分类；引用它们时不能写成"Android 官方分类"。
6. **没有公开的、按核/簇拓扑判定侧信道敞口的口径。**`vulnerabilities` 的读数是系统级聚合，不区分核、簇、插槽；同一台机器上的异构核（如大小核）在读数上不可分。本文因此不给出"某几个核受影响"的判读规则，只要求把拓扑作为采集注记保留。
7. **`mitigations=` 与各架构专用开关之间的聚合关系没有独立表格。**能引用的只有参考资料 15 里 `off` 档逐项列出的那串 arch-specific 选项；"某个子开关被 `mitigations=auto,nosmt` 之类组合怎样覆盖"没有公开穷举，本文一律交回给启动参数原文与重采。
8. **厂商 BSP 是否改写了参考资料 5 那四条 `write` 不可从设备侧证伪。**设备侧能读到的是当前 `perf_event_paranoid`、`kptr_restrict` 的**值**，读不到"这段 `init.rc` 是 AOSP 原版还是被厂商改过"。本文的 sysctl 结论只描述观察通道的实际收敛状态，不推断改动责任人。
9. **VMSCAPE 没有可引用的公开定义句。**厂商与上游对这一项的公开表述是"受影响处理器型号的缓解通过微码提供"这类公告级描述，没有像参考资料 4、13 那样的条件句可供落到读数上。第五章的家族表里这一行只登记公告归属与读数文件名，不登记判读规则。
10. **参考资料 21 的两句在本轮曾指向错误的页面。**它们不在 `developer.android.com/ndk/guides/simpleperf`（该页本轮抓取成功但不含这两句），而在 AOSP 的 `simpleperf/doc/android_application_profiling.md`。本文已把编号改指后者。这是一条**口径变化提醒**：同一句话在不同版本文档站点上的归属会漂移，引用前必须按句子而不是按页面核对。
11. **x86 的 `/proc/cpuinfo` 是否打印 `microcode` 行没有可引用的文档句。**参考资料 3 只覆盖 arm64 的打印表；x86 侧那一行由架构代码写入，本次没有取到对应的文档或表定义句。第十三章与第十四章之二因此把"微码版本行读不到"分成两种可能（该平台不导出 / 采集被截断）并都记为缺口，不记为"没有微码"。
12. **没有机器可读的"某机型命中哪些 CVE"官方清单。**Android 安全公告按月发布、按机型分档，但不提供可按读数键入的映射表。第二十六章 K10 与第二十九章 FAQ 里的"证据通道"因此只能写成流程要求，不能写成查询接口。
13. **CWE 条目的示例段含微架构实现细节，本文只取定义句。**参考资料 17 的原文在定义句之后还列举共享资源类别；本文不转述其中可被用作实现指引的部分，也不把 CWE 的映射标注当作缺陷等级（参考资料 16、25 的标注本身就说明这一点）。
14. **第廿五章的 15 份画像与 225 个对子不对应任何真实设备。**它们是为本篇分支覆盖构造的输入，未获任何厂商确认。引用第廿四、廿五章的数字时，只能引用"分支可达"这一层含义，不能引用分布、比例或机型读数。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
