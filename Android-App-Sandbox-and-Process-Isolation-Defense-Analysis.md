---
layout: default
title: "Android 应用沙箱与进程隔离防御分析报告"
description: "应用沙箱的四层构成（Linux UID 身份、SELinux 域与类别、seccomp 系统调用面、进程与数据形态）、身份合并与可调试性等放宽条件、沙箱逃逸类缺陷类型学与缓解矩阵、十项运营指标与两个只读核验脚本（隔离快照与快照漂移比对）"
lang: zh-CN
permalink: /Android-App-Sandbox-and-Process-Isolation-Defense-Analysis/
---
# Android 应用沙箱与进程隔离防御分析报告

**Android Application Sandbox and Process Isolation: A Defensive Analysis of Identity, Labeling, System-Call Surfaces, and Sandbox-Break Indicators**

- **报告日期**：2026-10-26
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动应用与 SDK 安全负责人、设备管理（MDM）与安全基线团队、ROM/OEM 系统工程师、移动平台安全评估机构、检测与取证工程师、应用商店与风控团队
- 视角声明：本文写的是**"一个应用进程到底被哪些机制围住、这些机制在什么条件下会合并或放宽、以及放宽能不能在设备侧被观察出来"**。**逐条 CVE、内核缺陷利用细节、Framework 权限模型的具体越权案例、Binder 协议字段不在本文**，那些内容在姊妹篇里，本文只做交叉引用而不重复展开：
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（六段分层与 AID/SELinux/seccomp 的位置；本文讲这三层**在应用进程上如何叠加、怎样被观察到**）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（权限与身份判定缺陷；本文的沙箱是"权限判定失效之后还剩什么"）
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核侧缺陷与提权取证；本文只讲沙箱层在提权链中的**位置与信号**）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（跨进程调用的 SELinux 判定与凭据传递）
  - [Android Linux 内核结构与子系统分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（设备节点与子系统供给，决定沙箱内进程实际能碰到什么）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面清点与分级；本文的"四层叠加"给出的是其中一域的分解方式）
  - [Android 远程代码执行分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（代码如何进来；本文讲进来之后**落在哪个身份与域里**）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（`user`/`userdebug`/`eng` 变体的来源；本文的可调试性判定依赖这一构建身份）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（平台怎么把应用围住 vs 应用自己在清单里声明了哪些弱点形态；两者是外部约束与自身姿态的两侧，本文的分类只落在应用这一格）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本文讲一个应用进程被 UID/SELinux 域/seccomp 三层围住、以及每层被单独放宽时留下什么信号；该篇讲无线栈进程在这套约束里处于哪个域，并标出这套约束覆盖不到的那一段：基带与蓝牙控制器固件跑在平台 SELinux 策略与 seccomp 之外的处理器上）
- 范围声明：全文只讨论**公开文档化**的沙箱构成、隔离机制、公开披露的缺陷类别与公告口径，以及在这些公开规则之上可安全实施的**只读**清点与检测流程。**不提供任何可用的漏洞利用代码、内存布局偏移、原语构造、策略绕过步骤或可操作的攻击流程。**所有"核验"指读者对**自有或已获授权测试的设备与自有应用**执行采集；两个脚本一个只做设备侧只读采集，一个只做离线文本比对，均不写设备、不装策略、不改配置。

---

## 一、摘要：沙箱不是一个开关，是四层叠加，而每层都可能被单独放宽

Android 对应用进程的约束不是"一个沙箱"这种单点机制，而是**四层彼此独立的机制叠加**：Linux 的按用户隔离（每应用一个 UID，并且运行在自己的进程里）、SELinux 的强制访问控制（进程处于某个域，数据带有类别标签）、seccomp 对系统调用面的裁剪、以及应用私有数据目录的默认不可访问。公开文档把这四层同时称为"Application Sandbox"的组成部分——原生应用与 ART 应用"运行在同一安全环境中，被包含在应用沙箱之内"（参考资料 3）。

这个结构决定了本文的全部判断方式：**"这台设备有没有沙箱"是个错误问题，正确的问题是"这四层此刻各自的强度是多少、以及它们有没有被同一次配置改动同时放宽"。**

三条最重要的结论：

- **最常见的"沙箱变薄"不是漏洞，而是配置。** 共享 Linux UID（`sharedUserId`，参考资料 7）、发布包带 `DEBUGGABLE` 标志、整机 `ro.debuggable=1`、SELinux 处于 Permissive——这四种情况都会让四层中的至少一层实际失效，而且都不需要任何缺陷。它们**可以在设备侧被只读观察到**（脚本 1 的 `FIND_shared_uid`、`FIND_debuggable`、`FIND_selinux`、`FIND_ro_debuggable`）。
- **身份与执行域是两个独立事实，二者不一致时才是高信号。** 应用的 UID 由身份层给出，进程所处的 SELinux 域由标签层给出。公开口径明确"默认情况下应用的类型是 `untrusted_app`，对进程而言其类型也称为域"（参考资料 5）。因此"应用 UID 的进程出现在特权或调试域"不是配置差异，而是边界已被替换的证据，应按事件处置（脚本 1 的 `FIND_priv_ctx`、脚本 2 的 `FIND_identity`）。
- **单次快照只能证明"此刻"，漂移才能证明"变过"。** 判定沙箱是否被改动，可靠做法是留存基线快照并做集合比对：同名进程的 UID 标签或执行域发生变化即为直接证据（脚本 2 以 `<进程名, 用户字段, 上下文>` 三元组为主键）。这也是本文把第二个脚本设计成离线比对而不是设备采集的原因。

本文其余部分依次给出：四层机制的公开口径与可观察信号（第二至第七节）、四种"边界合并或放宽"的形态（第八至第十一节）、沙箱逃逸类缺陷的类型学与缓解矩阵（第十二、十三节），以及十项可运营指标与两个可复制运行的只读脚本（第十四至第十七节）。

---

## 二、四层叠加：一张"谁拦什么"的总表

先固定术语。下表的每一层都独立存在、可独立关闭、并且**在设备侧有不同的观察点**——这是后文所有判定与脚本字段的依据。

| 层 | 公开口径（要点） | 它拦得住什么 | 它拦不住什么 | 设备侧观察点 |
| --- | --- | --- | --- | --- |
| **身份层（UID/DAC）** | 每应用一个唯一 UID 并运行在自己的进程中；平台利用 Linux 的用户级保护来标识和隔离应用资源（参考资料 1、2） | 其他应用对其数据目录/进程的常规 POSIX 访问 | 同一 UID 内的全部代码；内核侧缺陷；被显式合并身份的应用 | `ps` 的用户字段、`dumpsys package` 的 `uid`/`userId` |
| **标签层（SELinux MAC）** | 对**全部进程**强制实施 MAC；应用默认为 `untrusted_app` 类型，进程的类型即其域；Android 12 及以上使用类别隔离其他应用的数据（参考资料 4、5、6） | 域外进程对文件、socket、binder、设备的访问，即使 DAC 允许 | Permissive 模式下不拦截；策略本身过宽的域；同域内的混淆代理 | `getenforce`、`ps -AZ` 的上下文列 |
| **调用层（seccomp）** | 自 Android 8.0 起在 zygote 安装 seccomp 过滤器，zygote 即"所有 Android 应用由此派生"的进程（参考资料 11、12） | 应用进程可用的系统调用集合，从而压缩内核可达面 | 允许集合内的内核缺陷；不被该过滤器覆盖的进程 | `ro.build.version.sdk`（版本下限）；无法直接读取过滤器内容 |
| **数据/进程形态层** | 应用私有区域默认不对其他应用或用户开放（参考资料 16）；存在与系统其余部分隔离且不带自身权限的特殊进程（参考资料 8）；媒体等场景使用"按应用隔离的沙箱进程"（参考资料 15） | 数据被其他应用直接读取；解析不可信内容时的权限面 | 应用自己把数据放到共享位置；进程形态被误用（见第十一节） | `dumpsys package` 的标志、`ps -AZ` 中的 isolated 域进程 |

四层的关键性质是**乘法而不是加法**：任一层被放宽到"不拦截"，其余层的强度就成为实际隔离强度。这也是为什么本文反复要求把四个字段放在一起看，而不是分别报告"SELinux 开着""版本够新"。

---

## 三、身份层：UID 从哪来、区间边界在哪、什么时候会合并

公开文档对沙箱的第一句定义就是身份："Android 为此给每个应用分配一个唯一用户 ID（UID），并让应用运行在自己的进程中"（参考资料 1，原文直译）。这一句里其实包含两个独立承诺：**身份唯一**（UID）与**进程独立**（own process）。二者在实现上由不同的代码给出，因此也可能被不同的配置分别破坏。

UID 的算术边界来自 AOSP 的 `android_filesystem_config.h`，该头文件对区间的定义是逐字可读的：

```
#define AID_APP_START 10000 /* first app user */
#define AID_APP_END 19999
#define AID_USER_OFFSET 100000 /* offset for uid ranges for each user */
```

（参考资料 9。）三条 define 合起来给出本文脚本使用的换算：**每个用户占 `100000` 的偏移区间，应用编号从区间内的 `10000` 起。**于是 `u0_a123` 这类标签对应的 UID 是 `10123`，反向推算即 `appid = UID mod 100000 − 10000`。脚本 1 的 `FIND_uid_label` 就是把这条换算当作**一致性检查**：包声明的 `userId` 与它实际拿到的 SELinux MLS 标签应当吻合，不吻合时不判为漏洞而是提示核对（共享 UID、应用克隆、多用户配置三种常见解释）。

**多用户与隔离进程另有区间。**同一份头文件在 Android 9 的分支上还有 `#define AID_ISOLATED_START 99000 /* start of uids for fully isolated sandboxed processes */`（参考资料 10），说明"完全隔离的沙箱进程"有一整段独立 UID。需要注意：**不同 AOSP 分支上该常量的取值并不一致**（本次核验在较新分支上读到过与 99000 不同的数值），因此任何把隔离 UID 区间硬编码进检测规则的做法都不可靠——本文的脚本只按"用户字段是否形如应用 UID"分类，不判断隔离 UID 区间。

身份层的两个失效形态，都会在设备侧留下清晰痕迹，第八、九节展开。

---

## 四、标签层：域、类别，以及为什么 Permissive 是最高优先级信号

SELinux 在 Android 上的角色是把 MAC 覆盖到**所有**进程，官方表述为"对全部进程强制实施强制访问控制"（参考资料 4，原文在此后仍有内容，本文只引用到该处）。对应用来说，这条机制以两种形式落地：

1. **域（domain）**：官方概念页直接给出默认值——"例如，默认情况下应用具有 `untrusted_app` 类型。对一个进程而言，它的类型也被称为它的域。"（参考资料 5）。因此"`ps -AZ` 里应用进程显示 `u:r:untrusted_app:...`"不是实现细节，而是被文档化的预期状态。
2. **类别（category）**：官方概念页在 Android 12 及以上的语境下列出用途，其中第一条是"将应用数据隔离于其他应用的访问"（参考资料 6，原文为列表且在此后仍有内容）。本文脚本对类别只做**一件事**：检查是否存在"两个不同应用 UID 却使用完全相同的一套类别"（`FIND_category`）。这套类别相同意味着数据隔离在标签层不再区分这两个应用，属于需要核对的异常，但脚本刻意不解释每个 `cXXX` 的含义——**类别分配方案的逐字定义本次未取得**（见第二十二节边界清单）。

**为什么把 `getenforce` 放在第一位**：Permissive 意味着判定结果只记录不拦截，标签层实际上退化为"仅 DAC 一层"。这一状态既可能来自工程构建，也可能来自运行期改动，而在设备侧只是一个单词。脚本 1 把它定为唯一的 **L3-HIGH** 级 MAC 信号，正是因为它使第四节全部机制（域、类别）在同一台设备上同时失效——这是"乘法性质"最典型的体现。

另一个必须避免的推理是反向的：**Enforcing 不等于策略合理。**域的定义由厂商与设备实现参与编写，AOSP 公开的是机制而不是每台设备的策略内容；因此脚本对"应用 UID 的域不在常见应用域词表内"只给 **L2-WARN 并要求逐条核对**，不给"异常"结论——厂商自定义域完全可能是合法的（例如诊断类应用），把它自动判为漏洞会制造大量误报。词表本身是脚本作者维护的启发式集合，这一点在脚本输出里也已经明说。

---

## 五、调用层：seccomp 给出的是一条版本下限，不是一台设备的答案

公开材料的口径是版本化的，而且措辞非常具体。两份 Google 官方材料几乎同句出现：企业安全白皮书写"自 Android 8.0 起，一个 seccomp 过滤器被安装在 zygote 中，而 zygote 是所有 Android 应用由此派生的那个进程"（参考资料 11）；Android 开发者博客在 Android O 发布时写"Android O 包含一个单一 seccomp 过滤器，安装在 zygote 中，该进程是所有 Android 应用派生之源"（参考资料 12）。兼容性定义文档侧另有对 seccomp-BPF 与线程组同步（TSYNC）的要求（参考资料 13）。

这段口径在检测上给出两个直接可用的结论：

- **SDK 级别 ≥ 26 是一层"应当存在"的依据，不是"当前有效"的证明。**脚本 1 因此把 `FIND_seccomp_floor` 设计成三档：≥26 记为符合公开口径区间（L1-INFO）、21–25 记为落在该口径之前的历史区间（L2-WARN）、<21 记为隔离层数更少（L2-WARN），采不到值则明确不判定（L0-LOW）。
- **过滤器内容在只读采集下不可见。**`ps`、`dumpsys`、`getprop` 都不会告诉你某个进程当前的 seccomp 过滤器具体允许哪些调用。因此本文的快照**不把 seccomp 当作可验证项**，只把版本作为下限证据；需要实际验证调用面时，正确做法是在自有应用内做受控实验（这属于应用侧测试，不在设备侧只读采集范围内）。

这也是四层叠加里最容易在报告里被夸大的一层：一句"该设备启用了 seccomp"往往来自 Android 版本的推断，而不是测量。

---

## 六、进程层：派生方式决定了"同一份代码可以有几个不同强度的进程"

沙箱的第二句承诺是"运行在自己的进程中"（参考资料 1）。Android 的应用进程不是由 init 逐个拉起，而是从一个公共父进程派生——官方专门有 zygote 文档页（参考资料 14）。这个结构带来三个对防御有价值的推论：

- **共享的初始状态**：由同一父进程派生的应用进程会带上预加载的类与库，以及**继承自父进程的系统调用过滤策略**——这正是第五节"过滤器装在 zygote"之所以能覆盖全部应用的原因。
- **派生关系可观察**：`ps -AZ` 的父 PID 列会把应用进程指向 zygote（通常 `zygote64`）。本文脚本不判定 ppid（不同 `ps` 实现的列序差异会造成大量假阴性），但样例输出里保留了完整上下文列，供读者自行核对。
- **WebView 有自己的一层**：`webview_zygote` 是独立的派生入口，处理的是渲染不可信网页内容这一最高暴露面场景。官方媒体框架加固文档在描述解析不可信内容的组件时使用的是"按应用隔离的沙箱进程"（app-specific sandboxed processes，参考资料 15）这一表述；这类进程的意义在于**把解析代码的权限面压到具体某个应用身上**，解析缺陷的后果因此被限制在该应用的沙箱内而不是一个共享的系统服务内。

第七节的数据形态与第十节的缓解矩阵会继续用到这条线索：**同一套代码被放进哪种进程形态，决定了缺陷的影响半径。**

---

## 七、数据层：默认不可访问的范围，以及它不管什么

应用私有数据的公开口径同样是一句可引用的话："你的数据是安全的，因为默认情况下该区域不对其他应用或用户开放"（参考资料 16，原文语境是应用专属存储目录）。这句话有两处必须精确理解：

- **"默认"**描述的是文件属主与权限的初始设置，属身份层（第三节）；一旦应用主动把数据放到共享位置、通过内容提供器对外暴露、或使用带 `ALLOW_BACKUP` 之类标志的传输路径，保护范围就随之后移。脚本 1 输出的 `package_flags` 会**原样保留标志集合**而不对单个标志定性——因为标志的语义与产品风险判断绑定，自动判级会产生误导。
- **"不对用户开放"**指的是其他 UID（包括普通调试路径），不等于对拥有特权身份的过程成立。这也是 `DEBUGGABLE` 与 `run-as` 组合（第八节）能形成实际风险的结构性原因：**同一层机制被另一个身份合法借用**。

数据层与标签层的关系在取证上很有用：如果一份应用的私有数据被另一个应用 UID 读到，优先怀疑身份合并（`sharedUserId`）与类别异常（第四节），而不是先怀疑内核。

---

## 八、放宽形态一：`sharedUserId` —— 官方承诺的"每应用一个 UID"在这里被显式取消

`sharedUserId` 的文档定义是逐字的："与其他应用共享的一个 Linux 用户 ID 的名称。默认情况下，Android 为每个应用分配其自己唯一用户 ID。"（参考资料 7）。后半句是第三节沙箱定义的复述，前半句则是对它的**显式豁免**：一旦声明，多个应用包在同一设备上就是同一个 Linux 用户。

后果是可推导的，不需要任何缺陷：

- 身份层在这几个包之间**不存在**——它们的进程 UID 相同，POSIX 权限不再区分彼此；
- 数据层的"默认不可访问"（第七节）同样不再区分这几个包；
- 标签层的类别（第四节）通常按应用身份分配，身份合并时这一层是否仍提供区分，取决于设备策略——这一点本文不下结论，只要求在快照里留下 `package_sharedUserId` 字段。

治理口径建议这样写：**存量应用中的 `sharedUserId` 视为需要逐个说明的例外**，说明内容至少包括"哪几个包合并了身份、合并后共享哪些数据、这几个包的签名与发布流水线是否同源"。判定"该不该存在"很容易变成空谈，但**清点它存在**是完全机械的：脚本 1 在设备侧读取，应用侧则直接读清单文件。输出信号是 `FIND_shared_uid` → L2-WARN。

需要注意脚本的一个设计取舍：`dumpsys package` 中该字段既可能是 `sharedUserId= null`（未声明，L1-INFO），也可能**整行缺失**（未采到，L0-LOW）。这两者必须区分：后者是采集失败，把它读成"没有共享 UID"是典型的静默错误结论。

---

## 九、放宽形态二：可调试性 —— 一个标志就把沙箱变成半透明

三个不同层面的"可调试"经常被混为一谈，本文把它们拆开：

| 层面 | 载体 | 观察点 | 影响 |
| --- | --- | --- | --- |
| 整机构建变体 | `ro.build.type`（`user`/`userdebug`/`eng`）、`ro.debuggable` | 脚本 1 的 `build_type`、`debuggable_prop` | 整机调试通道是否开启；构建变体的公开口径见姊妹篇构建体系报告 |
| 单个发布包 | 包标志中的 `DEBUGGABLE` | `dumpsys package` 的 `flags=[ … ]` | 该包进程可被调试器附加，其私有数据可被调试路径读取 |
| 调试借用能力 | `run-as` | 官方调试文档写"The device supports run-as …"（参考资料 17，原文此后仍有内容） | 在满足条件时把调试能力落到具体包上 |

**判定原则**：出现在**发布**构建中的 `DEBUGGABLE` 包标志，严重度高于整机构建变体问题——前者意味着一个具体业务应用的沙箱对任何具备调试通道的一方都不成立，且往往在灰度/内测阶段引入后忘记移除。脚本 1 因此把 `FIND_debuggable` 定为 L3-HIGH，而把 `ro.debuggable=1` 定为 L2-WARN（后者通常是工程样机或内部机型，属资产管理问题）。

必须避免的过度结论：`run-as` 的**具体前置条件**（与 `ro.debuggable`、包标志之间的确切关系）本次未取得逐字文档，因此本文只说"调试通道的存在使沙箱不再对用户之外的一方成立"，不给出任何"如何用 run-as 读取某应用数据"的步骤；相关核验仅限读者自有应用。

---

## 十、缓解矩阵：把"这一层挡住了什么"写成可核对的表格

把第二节的表倒过来用：对每一类缺陷，逐层问"挡住/挡不住"。下表是防御侧设计依据，不是利用路径描述。

| 缺陷/风险类别 | 身份层 | 标签层（Enforcing） | 调用层 | 数据/进程形态层 | 设备侧可见信号 |
| --- | --- | --- | --- | --- | --- |
| 应用内组件被恶意调用（同包内越权） | 无效（同 UID） | 有限（同域） | 无关 | 无关 | 本文不判定；见 Framework 信任边界篇 |
| 跨应用读取私有数据 | 主挡 | 主挡（类别） | 无关 | 主挡（默认权限） | `FIND_shared_uid`、`FIND_category` |
| 解析不可信内容时的内存缺陷 | 无关 | 限制可访问对象 | 限制可达 syscall | 决定影响半径（共享服务 vs 按应用进程） | 进程形态观察（第六节） |
| 应用进程尝试越域访问文件/socket | 主挡 | 主挡 | 无关 | 无关 | `FIND_priv_ctx`（结果侧信号） |
| 内核本地提权 | 主挡（起点是 app UID） | 提权后失效 | 限制触发面 | 失效 | 提权链信号在内核篇；本文只报告 ID 与域不一致 |
| Permissive/策略放宽 | 无关 | **失效** | 无关 | 无关 | `FIND_selinux` L3 |
| 老版本设备（低于口径下限） | 有效 | 有效 | **少一层** | 有效 | `FIND_seccomp_floor` |
| 发布包可调试 | 有效 | 有效 | 有效 | **被借用** | `FIND_debuggable` L3 |

矩阵的用法是**看行的完整性**：一行里如果只有一层"主挡"，其余都无效或无关，那么这一行的风险只能靠补丁与配置治理来降——因为没有任何一层是设计来挡它的。这也解释了为什么本文把"四层同时放宽"的组合列为最高优先级清点的对象（第十一节）。

---

## 十一、组合失效：四种"单层看正常、合起来看危险"的配置

单独报告每一层的健康度会让风险看起来总是"还行"。下面四种组合是实际评估中反复出现、且只有合起来看才成立的。

1. **Enforcing + 未知域**（样例见脚本 1 的 `SBGAP001` 输出）：MAC 开着，但一个应用 UID 的进程处在不在词表内的厂商自定义域。`FIND_selinux` 给 L1-INFO、`FIND_domain` 给 L2-WARN——两个都"不算严重"，但组合起来需要索取该域的策略说明；**在没有策略内容时，域名字本身不能证明强度**。
2. **同类别 + 不同 UID**：两个应用 UID 使用完全相同的一套类别标签，标签层对这两个应用不再区分数据（`FIND_category` L2-WARN）。这一组合常来自身份合并的遗留配置或类别分配异常。
3. **Permissive + `userdebug` + `ro.debuggable=1`**：三个字段各自在合规清单里是独立条目，同时出现意味着**标签层已关闭且整机存在调试通道**——这才是"这台设备上的应用沙箱不成立"的真实含义（脚本 1 的 `SBBAD001` 样例即属此类）。
4. **UID 一致但域不一致**：包声明的 UID 换算出的标签与实际进程域不匹配。这是全部信号里最少见、也最不该被自动解释成"配置差异"的一种，脚本把它作为事件级判定（L3-HIGH）。

对应的治理动作只有三条，且都可运营化：**（a）把 `getenforce` 与构建变体纳入设备准入并持续清点；（b）对发布包标志做门禁；（c）留存基线快照并做漂移比对。**没有一条需要访问设备上的应用数据。

---

## 十二、沙箱逃逸类缺陷的类型学（S1–S8）

"打破沙箱"是一个影响类别而不是一个根因类别。Google 官方在年度报告里对它的定义是逐字的："Privilege escalation（提权）。一个通过破坏应用沙箱……来损害系统完整性的应用……"（参考资料 18，原文此后仍有内容）。本文按**根因所在层**把公开披露过的缺陷分成八类，只描述类别、可观察信号与检测面——不给利用细节。

| 编号 | 类别 | 根因层 | 典型公开描述形态 | 主要检测面 |
| --- | --- | --- | --- | --- |
| S1 | 内核本地提权后进入特权身份 | 内核 | 提权类公告条目（案例见内核篇） | UID/域不一致；进程出现在特权域 |
| S2 | SELinux 策略过宽或转换规则异常 | 标签层 | 域转换（transition）允许了不该允许的目标域 | 未知域清点；策略内容需厂商提供 |
| S3 | 类别/标签分配异常 | 标签层 | 不同应用共享同一套类别 | `FIND_category` |
| S4 | 共享 UID 造成的边界合并 | 身份层 | `sharedUserId` 声明（第九节） | `FIND_shared_uid` |
| S5 | 特权服务代应用执行动作（混淆代理） | Framework/IPC | 系统服务以自身权限代调用方读写对象 | 见 Framework 信任边界篇与 Binder 篇；风险页入口见参考资料 20 |
| S6 | 可调试发布包 + 调试通道 | 构建/配置 | `DEBUGGABLE` 标志、`run-as` 相关条件 | `FIND_debuggable`、`ro.debuggable` |
| S7 | 共享解析进程中的缺陷外溢 | 进程形态层 | 在共享服务进程内解析不可信内容 | 进程形态清点（是否按应用沙箱进程，第六节） |
| S8 | 系统调用面内的内核可达接口 | 调用层 | 未被过滤器覆盖或允许集合过宽 | 版本下限（`FIND_seccomp_floor`）；实际覆盖需应用侧实验 |

类型学的使用方式是**先定位落在哪一层，再决定谁来修**：S3/S4/S6 属配置与发布治理（应用团队/ROM 团队），S2 属策略编写方（厂商），S1/S8 属内核与补丁链（见补丁传导链篇），S5/S7 属框架与服务设计。

---

## 十三、为什么内存安全缺陷在这条边界上权重更高

沙箱逃逸的影响之所以严重，与它的**根因构成**有关。两份公开材料给出量级：

- Google 官方安全博客在 2024 年 10 月的《Safer with Google: Advancing Memory Safety》中给出趋势数据，Android 上报的内存安全漏洞数量"从 2019 年的 220 多个下降到今年年底预计的 36 个"（参考资料 19，原文表述此后仍有内容）。
- 行业联盟 Prossimo 的文档以归因方式引用 Google 的估计："Google 估计 90% 的 Android 漏洞是内存安全问题"（参考资料 21）。这是一句**转述性估计**，本文按归因引用，不当作官方逐字口径。

对防御侧的含义有两条：

1. **趋势数据支持"减少可达面"这条路线已经被验证过**，而不是仅停留在理论上；同时它也说明剩余的 36 这个量级里，每一条落在应用可达代码上的缺陷都更值得单独评估。
2. **非内存缺陷（S4、S6 这类配置问题）不会被内存安全改造消除**。它们只能靠清点与门禁。把年度指标只看成"内存安全比例"会漏掉本文第八、九节全部条目。

本文刻意**不引用**"Android 16 中内存安全语言占比"一类数字——本次未取得可靠逐字来源（见边界清单）。

---

## 十四、设备侧可见信号的强度分级

脚本输出的判级采用系列统一的四档：**L3-HIGH（按事件处置）/ L2-WARN（按例外核对）/ L1-INFO（符合公开口径，仅记录）/ L0-LOW（数据缺口，禁止读成"正常"）**。本文的关键纪律是最后一条：**采不到 = 缺口，不 = 通过。**样例输出里 `SBGAP001` 的 SUMMARY 行显示 `high=0,warn=1,gap=7`——这不是"基本健康"，而是"7 项未知 + 1 项待核对"，这类设备在报告里应当按未评估处理。

| 信号 | 档位 | 判据（可核对） |
| --- | --- | --- |
| `FIND_selinux` | L3 | Permissive：标签层整体停止拦截 |
| `FIND_priv_ctx` | L3 | 存在应用 UID 的进程位于特权/调试域 |
| `FIND_debuggable` | L3 | 发布包标志含 `DEBUGGABLE` |
| `FIND_pkg_domain` | L3 | 目标包有进程不在应用域词表内 |
| `FIND_shared_uid` | L2 | 声明了 `sharedUserId` |
| `FIND_category` | L2 | 不同 UID 共用同一套类别 |
| `FIND_domain` | L2 | 域不在应用域词表内（可能是厂商自定义域） |
| `FIND_uid_label` | L2 | UID 与 `seInfo` 换算不一致 |
| `FIND_seccomp_floor` | L2 | SDK 低于口径下限区间 |
| `FIND_ro_debuggable` | L2 | 整机 `ro.debuggable=1` |
| 各 `L0-LOW` | — | 字段未采到、`ps` 解析失败、快照无可比对记录 |

---

## 十五、十项可运营指标（沙箱侧）

以下指标全部可由脚本 1/2 的输出聚合，不需要访问应用数据，适合纳入 MDM 或设备资产平台周报表。

| 编号 | 指标 | 计算方式 | 建议初始阈值 |
| --- | --- | --- | --- |
| M1 | Enforcing 覆盖率 | `selinux,Enforcing` 的设备数 / 在线设备数 | 100%；任何偏离为事件 |
| M2 | 应用 UID 处于特权/调试域的台数 | `FIND_priv_ctx` = L3 的设备数 | 0 |
| M3 | 发布包带 `DEBUGGABLE` 的应用数 | `FIND_debuggable` = L3 的条目数 | 0 |
| M4 | `sharedUserId` 声明占比 | L2 命中的应用数 / 已安装第三方应用数 | 逐台清点，按例外清单管理 |
| M5 | 类别碰撞组数 | `category_collisions` 之和 | 0 |
| M6 | 快照身份漂移数 | 脚本 2 的 `identity_changes` 之和 | 0（非 0 即触发核查） |
| M7 | 未知域取值数 | `FIND_domain` L2 命中数 + `new_context_values` | 需有厂商域清单作为基线 |
| M8 | 低于调用层口径下限的设备存量 | `FIND_seccomp_floor` L2 的设备数 | 按退出计划递减 |
| M9 | 采集缺口率 | `L0-LOW` 行数 / 总判定行数 | < 5%；高于则先修采集 |
| M10 | 快照覆盖率与新鲜度 | 有基线且 7 日内有重采的设备占比 | ≥ 90% |

M9 与 M10 优先于其他八项：**没有覆盖率的检测指标会给出虚假的安全感**，这在第十一节的组合失效里尤其致命——组合恰恰需要多字段同时可见才判得出来。

---

## 十六、脚本 1：`sandbox-isolation-snapshot.sh`（设备侧只读快照）

脚本只调用 `getprop`、`getenforce`、`ps -AZ`、`dumpsys package` 四类只读命令，输出一行一条 CSV，供人读也供后续聚合；不写设备、不安装任何东西。用法 `sandbox-isolation-snapshot.sh <设备序列号> <包名>`；`<包名>` 用于把"这个应用实际拿到的身份/域"与声明值对齐。设计要点已在第三、四、八、九、十一节说明，这里给出完整脚本：

```bash
#!/usr/bin/env bash
# 脚本 1: sandbox-isolation-snapshot.sh — 应用沙箱与进程隔离快照（只读）
# 用法: sandbox-isolation-snapshot.sh <设备序列号> <包名>
# 依赖: adb；只调用 getprop / getenforce / ps -AZ / dumpsys package，全部只读
set -u
dev="${1:?用法: sandbox-isolation-snapshot.sh <serial> <package>}"
pkg="${2:?缺少包名}"
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "ERR,device_unreachable" >&2; exit 1; }

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
p() { adb -s "$dev" shell getprop "$1" 2>/dev/null | tr -d '\r'; }

sdk="$(p ro.build.version.sdk)"
btype="$(p ro.build.type)"
dbg="$(p ro.debuggable)"
enforce="$(adb -s "$dev" shell getenforce 2>/dev/null | tr -d '\r')"
psraw="$(adb -s "$dev" shell ps -AZ 2>/dev/null | tr -d '\r')"
pkgraw="$(adb -s "$dev" shell dumpsys package "$pkg" 2>/dev/null | tr -d '\r')"

# 解析 ps -AZ：不假定 SELinux 上下文所在列，按"哪个字段像上下文"定位，再取其后的用户字段
# 输出 PS_ROW,<进程名>,<用户字段>,<上下文>
psrows="$(printf '%s\n' "$psraw" | awk '
  { ctx=""; ci=0; name=""; uid=""
    for (i = 1; i <= NF; i++) {
      if (ctx == "" && $i ~ /^u:[a-z0-9_]+:[a-z0-9_]/) { ctx = $i; ci = i }
    }
    if (ctx != "") {
      if (ci + 1 <= NF && $(ci + 1) !~ /^[0-9]+$/) uid = $(ci + 1)
      name = $NF
      printf "PS_ROW,%s,%s,%s\n", name, uid, ctx
    }
  }')"
ps_total="$(printf '%s\n' "$psraw" | grep -c '[^[:space:]]')"
ps_ok="$(printf '%s\n' "$psrows" | grep -c '^PS_ROW,')"
printf '%s\n' "$psrows" | grep '^PS_ROW,' || true

row sdk "$sdk"; row build_type "$btype"; row debuggable_prop "$dbg"
row selinux "$enforce"; row package "$pkg"
row ps_lines "$ps_total"; row ps_parsed "$ps_ok"

uid_decl="$(printf '%s\n' "$pkgraw" | sed -n 's/.*userId=\([0-9][0-9]*\).*/\1/p' | head -1)"
seinfo="$(printf '%s\n' "$pkgraw" | sed -n 's/.*seInfo=\([^[:space:]]*\).*/\1/p' | head -1)"
shu="$(printf '%s\n' "$pkgraw" | sed -n 's/.*sharedUserId=[[:space:]]*\([^[:space:]]*\).*/\1/p' | head -1)"
flags="$(printf '%s\n' "$pkgraw" | grep -o 'flags=\[[^]]*\]' | head -1)"
row package_userId "$uid_decl"; row package_seInfo "$seinfo"
row package_sharedUserId "$shu"; row package_flags "$flags"

case "$enforce" in
  Enforcing) row FIND_selinux "L1-INFO SELinux Enforcing：MAC 覆盖全部进程" ;;
  Permissive) row FIND_selinux "L3-HIGH SELinux Permissive：域/类别约束不再拦截访问，只剩 UID(DAC) 一层" ;;
  *) row FIND_selinux "L0-LOW 未采到 getenforce：本次不判定 MAC 状态" ;;
esac

case "$sdk" in
  ''|*[!0-9]*) row FIND_seccomp_floor "L0-LOW 未采到 SDK 级别：不判定" ;;
  *) if [ "$sdk" -ge 26 ]; then
       row FIND_seccomp_floor "L1-INFO SDK $sdk：处于公开口径所述 Android 8.0 起在 zygote 安装 seccomp 过滤器的区间"
     elif [ "$sdk" -ge 21 ]; then
       row FIND_seccomp_floor "L2-WARN SDK $sdk：早于该公开口径，按遗留设备处理"
     else
       row FIND_seccomp_floor "L2-WARN SDK $sdk：早于 seccomp-BPF 可用区间，隔离层数更少"
     fi ;;
esac

if [ -n "$uid_decl" ] && [ -n "$seinfo" ]; then
  u=$(( uid_decl / 100000 )); a=$(( uid_decl % 100000 - 10000 ))
  if [ "$a" -lt 0 ]; then
    row FIND_uid_label "L0-LOW UID $uid_decl 不在应用 UID 区间（每用户偏移 100000、应用起始 10000）：不判定"
  else
    expect="u${u}_a${a}"
    if [ "$expect" = "$seinfo" ]; then
      row FIND_uid_label "L1-INFO UID $uid_decl 与标签 $seinfo 一致（appid = UID mod 100000 再减起始 10000）"
    else
      row FIND_uid_label "L2-WARN UID $uid_decl 期望标签 $expect，实际 $seinfo：核对共享 UID、克隆应用或多用户配置"
    fi
  fi
else
  row FIND_uid_label "L0-LOW userId/seInfo 不完整：不判定"
fi

case "$shu" in
  null|NULL) row FIND_shared_uid "L1-INFO sharedUserId=null：应用独占一个 UID" ;;
  '') row FIND_shared_uid "L0-LOW 未采到 sharedUserId 字段：不判定身份边界是否合并" ;;
  *) row FIND_shared_uid "L2-WARN 声明 sharedUserId=$shu：与同签名应用共用一个 Linux UID，身份边界合并" ;;
esac

case "$flags" in
  *DEBUGGABLE*) row FIND_debuggable "L3-HIGH 该包带 DEBUGGABLE 标志：调试通道开启，应用侧隔离对该进程不成立" ;;
  flags=\[*\]) row FIND_debuggable "L1-INFO 标志集合中未出现 DEBUGGABLE" ;;
  *) row FIND_debuggable "L0-LOW 未解析到 flags 集合：不得据此判定可调试性" ;;
esac

if [ "$ps_ok" -eq 0 ]; then
  row FIND_ps "L0-LOW ps 输出无法解析（共 $ps_total 行）：本次隔离观察不成立，不得读成没有异常进程"
else
  app_procs="$(printf '%s\n' "$psrows" | awk -F, '
    { split($4, cc, ":"); t = cc[3]
      if ($3 ~ /^u[0-9]+_a[0-9]+$/ && t ~ /^(untrusted_app(_[0-9]+)?|priv_app|platform_app|ephemeral_app|isolated_app|webview_zygote|app_zygote|sdk_sandbox|shared_isolated_runtime|gpu_service|media_provider|media|mdns|vpn|radio)$/) n++ }
    END { print n + 0 }')"
  odd_procs="$(printf '%s\n' "$psrows" | awk -F, '
    { split($4, cc, ":"); t = cc[3]
      if ($3 ~ /^u[0-9]+_a[0-9]+$/ && t !~ /^(untrusted_app(_[0-9]+)?|priv_app|platform_app|ephemeral_app|isolated_app|webview_zygote|app_zygote|sdk_sandbox|shared_isolated_runtime|gpu_service|media_provider|media|mdns|vpn|radio|su|shell|init|magisk|runas_app|adbd|recovery)$/) n++ }
    END { print n + 0 }')"
  bad_ctx="$(printf '%s\n' "$psrows" | awk -F, '
    { split($4, cc, ":"); t = cc[3]
      if ($3 ~ /^u[0-9]+_a[0-9]+$/ && t ~ /^(su|shell|init|magisk|runas_app|adbd|recovery)$/) n++ }
    END { print n + 0 }')"
  col="$(printf '%s\n' "$psrows" | awk -F, '
    $3 ~ /^u[0-9]+_a[0-9]+$/ { k = $4
      if (!(k in seen)) { seen[k] = $3 }
      else if (seen[k] != $3 && !(k in multi)) { multi[k] = 1; c++ } }
    END { print c + 0 }')"
  iso="$(printf '%s\n' "$psrows" | awk -F, '
    { split($4, cc, ":"); t = cc[3]
      if (t ~ /^isolated_app/) n++ }
    END { print n + 0 }')"
  pkgbad="$(printf '%s\n' "$psrows" | awk -F, -v p="$pkg" '
    { split($4, cc, ":"); t = cc[3]
      if ($2 == p && t !~ /^(untrusted_app(_[0-9]+)?|priv_app|platform_app|ephemeral_app|isolated_app|webview_zygote|app_zygote|sdk_sandbox|shared_isolated_runtime|gpu_service|media_provider|media|mdns|vpn|radio)$/) n++ }
    END { print n + 0 }')"
  pkgrows="$(printf '%s\n' "$psrows" | awk -F, -v p="$pkg" '$2 == p { n++ } END { print n + 0 }')"
  row app_domain_processes "$app_procs"; row unexpected_domain "$odd_procs"
  row privileged_context_apps "$bad_ctx"; row isolated_processes "$iso"
  row package_processes "$pkgrows"; row category_collisions "$col"

  if [ "$bad_ctx" -gt 0 ]; then
    printf '%s\n' "$psrows" | awk -F, '
      { split($4, cc, ":"); t = cc[3] }
      $3 ~ /^u[0-9]+_a[0-9]+$/ && t ~ /^(su|shell|init|magisk|runas_app|adbd|recovery)$/ { print }' | head -5
    row FIND_priv_ctx "L3-HIGH 有 $bad_ctx 个应用 UID 进程运行在特权/调试域：UID 与执行域不一致，按事件处置"
  else
    row FIND_priv_ctx "L1-INFO 未发现应用 UID 进程位于特权/调试域"
  fi

  if [ "$odd_procs" -gt 0 ]; then
    printf '%s\n' "$psrows" | awk -F, '
      { split($4, cc, ":"); t = cc[3] }
      $3 ~ /^u[0-9]+_a[0-9]+$/ && t !~ /^(untrusted_app(_[0-9]+)?|priv_app|platform_app|ephemeral_app|isolated_app|webview_zygote|app_zygote|sdk_sandbox|shared_isolated_runtime|gpu_service|media_provider|media|mdns|vpn|radio|su|shell|init|magisk|runas_app|adbd|recovery)$/ { print }' | head -5
    row FIND_domain "L2-WARN $odd_procs 个应用 UID 进程的域不在本脚本的应用域词表内：可能是厂商自定义域，需逐条核对而不是直接判为漏洞"
  else
    row FIND_domain "L1-INFO 应用 UID 进程的域均在应用域词表内"
  fi

  if [ "$col" -gt 0 ]; then
    row FIND_category "L2-WARN $col 组进程共用同一套类别但 UID 不同：按公开口径类别用于隔离其他应用的数据，需核对为何相同"
  else
    row FIND_category "L1-INFO 未观察到不同 UID 共享同一套 SELinux 类别"
  fi

  if [ "$pkgbad" -gt 0 ]; then
    row FIND_pkg_domain "L3-HIGH 目标包 $pkg 的 $pkgbad 个进程不在应用域词表内：核对是否被以其他身份重新启动"
  elif [ "$pkgrows" -gt 0 ]; then
    row FIND_pkg_domain "L1-INFO 目标包 $pkg 有 $pkgrows 个进程，域均在应用域词表内"
  else
    row FIND_pkg_domain "L0-LOW 目标包 $pkg 当前无进程：不判定，需先确认包已安装并处于运行"
  fi

  if [ "$iso" -gt 0 ]; then
    row FIND_isolated "L1-INFO 观察到 $iso 个 isolated 域进程（公开口径：该进程与系统其余部分隔离且不带自身权限）"
  else
    row FIND_isolated "L0-LOW 未观察到 isolated 域进程：不作为问题判定，取决于设备与在用组件"
  fi

  if [ "$ps_ok" -lt "$ps_total" ]; then
    row FIND_ps_partial "L0-LOW $ps_total 行中有 $(( ps_total - ps_ok )) 行未解析（列序差异或截断）：判定范围仅限已解析行"
  else
    row FIND_ps_partial "L1-INFO ps 行全部解析成功"
  fi
fi

if [ "$dbg" = "1" ]; then
  row FIND_ro_debuggable "L2-WARN ro.debuggable=1：整机为可调试构建，应用之外的边界同样放宽"
elif [ -z "$dbg" ]; then
  row FIND_ro_debuggable "L0-LOW 未采到 ro.debuggable：不判定"
else
  row FIND_ro_debuggable "L1-INFO ro.debuggable=$dbg"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,dev=%s,pkg=%s\n' "$hi" "$md" "$gap" "$info" "$dev" "$pkg"
```

下面三段样例输出来自**合成设备档案**（一组用于脚本验收的固定输入，不是某台真机），运行环境为把 `adb` 替换成读取档案的替身命令，因此读者可以逐字复现。三段分别对应第四节/第十一节描述的三种典型形态：健康、多层放宽、采集缺口。

健康形态（SDK 36、Enforcing、独占 UID、含一个 isolated 域进程）：

```text
PS_ROW,/system/bin/init,root,u:r:init:s0
PS_ROW,zygote64,root,u:r:zygote:s0
PS_ROW,surfaceflinger,system,u:r:surfaceflinger:s0
PS_ROW,system_server,system,u:r:system_server:s0
PS_ROW,com.acme.wallet,u0_a123,u:r:untrusted_app:s0:c123,c256,c512,c768
PS_ROW,com.acme.notes,u0_a124,u:r:untrusted_app:s0:c124,c256,c512,c768
PS_ROW,webview_zygote,u0_a123,u:r:webview_zygote:s0
PS_ROW,com.acme.wallet,u0_a123,u:r:isolated_app:s0:c123,c256
sdk,36
build_type,user
debuggable_prop,0
selinux,Enforcing
package,com.acme.wallet
ps_lines,8
ps_parsed,8
package_userId,10123
package_seInfo,u0_a123
package_sharedUserId,null
package_flags,flags=[ HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP ]
FIND_selinux,L1-INFO SELinux Enforcing：MAC 覆盖全部进程
FIND_seccomp_floor,L1-INFO SDK 36：处于公开口径所述 Android 8.0 起在 zygote 安装 seccomp 过滤器的区间
FIND_uid_label,L1-INFO UID 10123 与标签 u0_a123 一致（appid = UID mod 100000 再减起始 10000）
FIND_shared_uid,L1-INFO sharedUserId=null：应用独占一个 UID
FIND_debuggable,L1-INFO 标志集合中未出现 DEBUGGABLE
app_domain_processes,4
unexpected_domain,0
privileged_context_apps,0
isolated_processes,1
package_processes,2
category_collisions,0
FIND_priv_ctx,L1-INFO 未发现应用 UID 进程位于特权/调试域
FIND_domain,L1-INFO 应用 UID 进程的域均在应用域词表内
FIND_category,L1-INFO 未观察到不同 UID 共享同一套 SELinux 类别
FIND_pkg_domain,L1-INFO 目标包 com.acme.wallet 有 2 个进程，域均在应用域词表内
FIND_isolated,L1-INFO 观察到 1 个 isolated 域进程（公开口径：该进程与系统其余部分隔离且不带自身权限）
FIND_ps_partial,L1-INFO ps 行全部解析成功
FIND_ro_debuggable,L1-INFO ro.debuggable=0
SUMMARY,high=0,warn=0,gap=0,info=12,dev=SBGOOD001,pkg=com.acme.wallet
```

多层放宽形态（Permissive、`userdebug`、`ro.debuggable=1`、发布包带 `DEBUGGABLE`、三个不同 UID 共用同一套类别、且有一个应用 UID 的进程位于 `shell` 域）：

```text
PS_ROW,/system/bin/init,root,u:r:init:s0
PS_ROW,zygote64,root,u:r:zygote:s0
PS_ROW,com.vendor.toolkit,u0_a77,u:r:shell:s0
PS_ROW,com.acme.wallet,u0_a77,u:r:untrusted_app:s0:c77,c256,c512,c768
PS_ROW,com.acme.notes,u0_a88,u:r:untrusted_app:s0:c77,c256,c512,c768
PS_ROW,com.acme.bringer,u0_a99,u:r:untrusted_app:s0:c77,c256,c512,c768
PS_ROW,media.extractor,mediaex,u:r:mediaextractor:s0
sdk,34
build_type,userdebug
debuggable_prop,1
selinux,Permissive
package,com.acme.wallet
ps_lines,7
ps_parsed,7
package_userId,10077
package_seInfo,u0_a77
package_sharedUserId,android.shareduid.acme
package_flags,flags=[ DEBUGGABLE HAS_CODE ALLOW_CLEAR_USER_DATA ]
FIND_selinux,L3-HIGH SELinux Permissive：域/类别约束不再拦截访问，只剩 UID(DAC) 一层
FIND_seccomp_floor,L1-INFO SDK 34：处于公开口径所述 Android 8.0 起在 zygote 安装 seccomp 过滤器的区间
FIND_uid_label,L1-INFO UID 10077 与标签 u0_a77 一致（appid = UID mod 100000 再减起始 10000）
FIND_shared_uid,L2-WARN 声明 sharedUserId=android.shareduid.acme：与同签名应用共用一个 Linux UID，身份边界合并
FIND_debuggable,L3-HIGH 该包带 DEBUGGABLE 标志：调试通道开启，应用侧隔离对该进程不成立
app_domain_processes,3
unexpected_domain,0
privileged_context_apps,1
isolated_processes,0
package_processes,1
category_collisions,1
PS_ROW,com.vendor.toolkit,u0_a77,u:r:shell:s0
FIND_priv_ctx,L3-HIGH 有 1 个应用 UID 进程运行在特权/调试域：UID 与执行域不一致，按事件处置
FIND_domain,L1-INFO 应用 UID 进程的域均在应用域词表内
FIND_category,L2-WARN 1 组进程共用同一套类别但 UID 不同：按公开口径类别用于隔离其他应用的数据，需核对为何相同
FIND_pkg_domain,L1-INFO 目标包 com.acme.wallet 有 1 个进程，域均在应用域词表内
FIND_isolated,L0-LOW 未观察到 isolated 域进程：不作为问题判定，取决于设备与在用组件
FIND_ps_partial,L1-INFO ps 行全部解析成功
FIND_ro_debuggable,L2-WARN ro.debuggable=1：整机为可调试构建，应用之外的边界同样放宽
SUMMARY,high=3,warn=3,gap=1,info=5,dev=SBBAD001,pkg=com.acme.wallet
```

采集缺口形态（`getenforce`、`ro.debuggable`、`dumpsys` 字段全部未采到，`ps` 有 1 行无法解析，且出现一个不在词表内的厂商自定义域）：

```text
PS_ROW,zygote64,root,u:r:zygote:s0
PS_ROW,com.acme.wallet,u0_a123,u:r:untrusted_app:s0:c123,c256,c512,c768
PS_ROW,com.vendor.diag,u0_a40,u:r:vendor_diag_app:s0:c40,c256,c512,c768
sdk,34
build_type,user
debuggable_prop,
selinux,
package,com.acme.wallet
ps_lines,4
ps_parsed,3
package_userId,
package_seInfo,
package_sharedUserId,
package_flags,
FIND_selinux,L0-LOW 未采到 getenforce：本次不判定 MAC 状态
FIND_seccomp_floor,L1-INFO SDK 34：处于公开口径所述 Android 8.0 起在 zygote 安装 seccomp 过滤器的区间
FIND_uid_label,L0-LOW userId/seInfo 不完整：不判定
FIND_shared_uid,L0-LOW 未采到 sharedUserId 字段：不判定身份边界是否合并
FIND_debuggable,L0-LOW 未解析到 flags 集合：不得据此判定可调试性
app_domain_processes,1
unexpected_domain,1
privileged_context_apps,0
isolated_processes,0
package_processes,1
category_collisions,0
FIND_priv_ctx,L1-INFO 未发现应用 UID 进程位于特权/调试域
PS_ROW,com.vendor.diag,u0_a40,u:r:vendor_diag_app:s0:c40,c256,c512,c768
FIND_domain,L2-WARN 1 个应用 UID 进程的域不在本脚本的应用域词表内：可能是厂商自定义域，需逐条核对而不是直接判为漏洞
FIND_category,L1-INFO 未观察到不同 UID 共享同一套 SELinux 类别
FIND_pkg_domain,L1-INFO 目标包 com.acme.wallet 有 1 个进程，域均在应用域词表内
FIND_isolated,L0-LOW 未观察到 isolated 域进程：不作为问题判定，取决于设备与在用组件
FIND_ps_partial,L0-LOW 4 行中有 1 行未解析（列序差异或截断）：判定范围仅限已解析行
FIND_ro_debuggable,L0-LOW 未采到 ro.debuggable：不判定
SUMMARY,high=0,warn=1,gap=7,info=4,dev=SBGAP001,pkg=com.acme.wallet
```

三段输出对照着读，可以看到脚本设计上的几个刻意选择：

- **每个判定行都以 `L?` 前缀开头**，聚合时按前缀计数即可，不需要解析中文；`SUMMARY` 行的 `gap=` 就是"这台设备有多少项其实没看到"。
- **`PS_ROW` 行始终原样输出**（健康样例 8 行全部解析、缺口样例 4 行只解析出 3 行）。缺口的 `FIND_ps_partial` 明确写了"判定范围仅限已解析行"，防止读者把"没报异常"读成"没有异常"。
- **`package_processes,2` 这类计数不是判级**，一个包有两个进程（主进程 + isolated 进程）是完全正常的形态（第六、七节），因此只在 `FIND_pkg_domain` 里对"域是否在应用域词表内"给出判定。
- **命中特权/调试域时会把那几行 `PS_ROW` 直接打印出来**（放宽样例中的 `com.vendor.toolkit` 一行），避免读者为了定位异常再采一次。

---

## 十七、脚本 2：`sandbox-drift-audit.sh`（两次快照的离线漂移比对）

单次快照回答"现在怎样"，漂移比对回答"有没有被改过"。脚本 2 只读两个文本文件（脚本 1 的历史输出），完全不接触设备，因此可以安全地放进任何审计流水线：把基线快照留存下来，周期采集后离线比对。

主键选择是这一版脚本的核心设计：**以 `<进程名, 用户字段, SELinux 上下文>` 三元组为主键**，而不是只按进程名。原因是同一个包名在设备上合法地对应多个进程（主进程与 isolated 进程，第三、六节），只按名字比对会把这种正常形态读成"重复记录"或漏掉身份集合的变化。按三元组做集合差之后：

- 名字在两侧都有、但三元组不同 → `DRIFT_ADD` / `DRIFT_DEL`，即该进程的身份或执行域变过 → `FIND_identity` L3-HIGH；
- 名字只在当前侧出现且形如包名 → `NEW_APP` → L2-WARN；
- 上下文取值集合新增 → `FIND_ctx_vocab` L2-WARN（可能是 OTA 后标签变化，也可能是新出现的域）。

```bash
#!/usr/bin/env bash
# 脚本 2: sandbox-drift-audit.sh — 两次隔离快照的离线差异比对（纯文本，无 adb、无网络、不写设备）
# 用法: sandbox-drift-audit.sh <基线快照> <当前快照>
# 输入: 脚本 1 的输出；只用 PS_ROW,<进程名>,<用户字段>,<SELinux上下文> 行，其余行忽略
set -u
base="${1:?用法: sandbox-drift-audit.sh <baseline-snapshot> <current-snapshot>}"
curr="${2:?缺少当前快照路径}"
for f in "$base" "$curr"; do
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

tb="$(mktemp)"; tc="$(mktemp)"
trap 'rm -f "$tb" "$tc"' EXIT INT TERM
grep '^PS_ROW,' "$base" 2>/dev/null | sed 's/^PS_ROW,//' | sort -u > "$tb"
grep '^PS_ROW,' "$curr" 2>/dev/null | sed 's/^PS_ROW,//' | sort -u > "$tc"

nb="$(grep -c '[^[:space:]]' "$tb")"
nc="$(grep -c '[^[:space:]]' "$tc")"
row baseline_tuples "$nb"; row current_tuples "$nc"
if [ "$nb" -eq 0 ] || [ "$nc" -eq 0 ]; then
  row FIND_input "L0-LOW 任一侧没有可解析的进程记录：比对不成立，不得读成没有变化"
  printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,base=%s,curr=%s\n' "$hi" "$md" "$gap" "$info" "$nb" "$nc"
  exit 0
fi

# 主键 = 三元组 <进程名,用户字段,上下文>；同名而三元组不同即为身份或执行域漂移
d1="$(comm -13 "$tb" "$tc")"   # 当前侧新增的三元组
d2="$(comm -23 "$tb" "$tc")"   # 基线侧消失的三元组
nbf="$(cut -d, -f1 "$tb" | sort -u)"
ncf="$(cut -d, -f1 "$tc" | sort -u)"
rep="$(awk -F, '
  NR == FNR { bn[$1]; next }
  NF == 0 { next }
  { if ($1 in bn) print "DRIFT_ADD," $0
    else if ($1 ~ /\./) print "NEW_APP," $0
    else print "NEW_OTHER," $0 }' <(printf '%s\n' "$nbf") <(printf '%s\n' "$d1")
  awk -F, '
  NR == FNR { cn[$1]; next }
  NF == 0 { next }
  { if ($1 in cn) print "DRIFT_DEL," $0; else print "GONE," $0 }' <(printf '%s\n' "$ncf") <(printf '%s\n' "$d2")
)"

printf '%s\n' "$rep" | grep -E '^(DRIFT_ADD|DRIFT_DEL|NEW_APP|NEW_OTHER),' | head -30 || true
idc="$(printf '%s\n' "$rep" | grep -E '^(DRIFT_ADD|DRIFT_DEL),' | cut -d, -f2 | sort -u | grep -c '[^[:space:]]')"
napp="$(printf '%s\n' "$rep" | grep -c '^NEW_APP,')"
noth="$(printf '%s\n' "$rep" | grep -c '^NEW_OTHER,')"
gone="$(printf '%s\n' "$rep" | grep -c '^GONE,')"
row identity_changes "$idc"; row new_app_names "$napp"
row new_other_names "$noth"; row removed_names "$gone"

# 新出现的安全上下文取值（整串比较，含类别）与新出现的应用 UID 标签
ncx="$(comm -13 <(awk -F, '{print $3}' "$tb" | sort -u) \
               <(awk -F, '{print $3}' "$tc" | sort -u) | grep -c '[^[:space:]]')"
nxu="$(comm -13 <(awk -F, '$2 ~ /^u[0-9]+_a[0-9]+$/ {print $2}' "$tb" | sort -u) \
               <(awk -F, '$2 ~ /^u[0-9]+_a[0-9]+$/ {print $2}' "$tc" | sort -u) | grep -c '[^[:space:]]')"
row new_context_values "$ncx"; row new_app_uids "$nxu"

if [ "$idc" -gt 0 ]; then
  row FIND_identity "L3-HIGH $idc 个同名进程的 UID 标签或执行域发生变化：这是隔离边界被改动的直接证据，按事件核查（尤其是从应用域进入特权或调试域、或多个身份集合被合并）"
else
  row FIND_identity "L1-INFO 同名进程的 UID 标签与执行域未变化"
fi
if [ "$napp" -gt 0 ]; then
  row FIND_newapp "L2-WARN 出现 $napp 个基线中没有的包名形态进程：确认是新增组件、重装还是非预期驻留"
else
  row FIND_newapp "L1-INFO 未见新增包名形态进程"
fi
if [ "$noth" -gt 0 ]; then
  row FIND_newother "L1-INFO $noth 个非包名形态进程为新增：系统组件重启常见，仅记录"
else
  row FIND_newother "L1-INFO 无非包名形态新增进程"
fi
if [ "$ncx" -gt 0 ]; then
  row FIND_ctx_vocab "L2-WARN 出现 $ncx 个基线中未见的上下文取值：核对厂商域定义、OTA 后的标签变化与类别分配"
else
  row FIND_ctx_vocab "L1-INFO 上下文取值集合未新增"
fi
if [ "$nxu" -gt 0 ]; then
  row FIND_newuid "L1-INFO 出现 $nxu 个新的应用 UID 标签：多为新装应用，与身份合并判定一并核对"
fi
if [ "$gone" -gt 0 ]; then
  row FIND_gone "L1-INFO $gone 个三元组在本次快照中不存在：进程正常退出或被杀都会出现，不单独定性"
fi
printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,base=%s,curr=%s\n' "$hi" "$md" "$gap" "$info" "$nb" "$nc"
```

样例同样来自合成档案。**无变化**的一组（基线与当前快照相同，全部为 L1-INFO，`SUMMARY` 里 `high/warn/gap` 均为 0）：

```text
baseline_tuples,8
current_tuples,8
identity_changes,0
new_app_names,0
new_other_names,0
removed_names,0
new_context_values,0
new_app_uids,0
FIND_identity,L1-INFO 同名进程的 UID 标签与执行域未变化
FIND_newapp,L1-INFO 未见新增包名形态进程
FIND_newother,L1-INFO 无非包名形态新增进程
FIND_ctx_vocab,L1-INFO 上下文取值集合未新增
SUMMARY,high=0,warn=0,gap=0,info=4,base=8,curr=8
```

有漂移的一组（一个包从应用域进入 `shell` 域、一个应用的 UID 标签变化、新增一个包名形态进程）：

```text
baseline_tuples,8
current_tuples,10
DRIFT_ADD,com.acme.notes,u0_a125,u:r:untrusted_app:s0:c124,c256,c512,c768
NEW_APP,com.acme.pushservice,u0_a150,u:r:untrusted_app:s0:c150,c256,c512,c768
DRIFT_ADD,com.acme.wallet,u0_a123,u:r:shell:s0
DRIFT_DEL,com.acme.wallet,u0_a123,u:r:untrusted_app:s0:c123,c256,c512,c768
identity_changes,2
new_app_names,1
new_other_names,0
removed_names,0
new_context_values,2
new_app_uids,2
FIND_identity,L3-HIGH 2 个同名进程的 UID 标签或执行域发生变化：这是隔离边界被改动的直接证据，按事件核查（尤其是从应用域进入特权或调试域、或多个身份集合被合并）
FIND_newapp,L2-WARN 出现 1 个基线中没有的包名形态进程：确认是新增组件、重装还是非预期驻留
FIND_newother,L1-INFO 无非包名形态新增进程
FIND_ctx_vocab,L2-WARN 出现 2 个基线中未见的上下文取值：核对厂商域定义、OTA 后的标签变化与类别分配
FIND_newuid,L1-INFO 出现 2 个新的应用 UID 标签：多为新装应用，与身份合并判定一并核对
SUMMARY,high=1,warn=2,gap=0,info=2,base=8,curr=10
```

作为对照，"当前快照里没有可用进程记录"时的输出必须**不是**"全部通过"：

```text
baseline_tuples,8
current_tuples,0
FIND_input,L0-LOW 任一侧没有可解析的进程记录：比对不成立，不得读成没有变化
SUMMARY,high=0,warn=0,gap=1,info=0,base=8,curr=0
```

三段输出的差别全部落在"同名进程"的处理上：无变化组只有计数行；漂移组把发生变化的三元组逐条打印（`DRIFT_ADD` / `DRIFT_DEL`），其中钱包包那一对清楚地展示了"从 `untrusted_app` 进入 `shell` 域"；缺口组则拒绝给出任何"未变化"的结论。脚本 2 的输出可以直接接脚本 1 的历史文件，无需额外转换——这也是把它设计成离线比对的原因：**基线一旦留存，检查成本与设备在线与否无关。**


---

## 十八、检测与取证：从一条 L3 信号开始的研判顺序

### 18.1 事件级信号只有三类

本文所有判定中，只有三类应当进入事件流程（而不是清单）：`FIND_selinux` = Permissive、`FIND_priv_ctx`（应用 UID 的进程位于特权/调试域）、`FIND_identity`（同名进程的身份或域发生漂移）。它们的共同点是**指向"边界本身已经不成立"**，而不是"某一层配置偏松"。

`FIND_debuggable` 的 `DEBUGGABLE` 标志通常属发布质量问题：先在发布流水线里确认这个包是怎么带上的，再决定是否进入事件流程。

### 18.2 一条 L3 信号的取证顺序

1. **先固定证据**：保存触发判定的原始 `ps -AZ` 与 `dumpsys package` 全文（脚本 1 已原样输出 `PS_ROW` 行与被命中的行）。不要只留 `SUMMARY`。
2. **再看是否为采集假象**：确认 `ps_parsed` 与 `ps_lines` 相等；若不等，先修采集再判定（第十一节组合 4 的教训）。
3. **建立时间线**：用脚本 2 与最近一份基线比对，确定漂移发生在哪两次采集之间，把变更窗口与 OTA、MDM 策略下发、内部测试装机对齐。
4. **确定影响面**：以 UID 为单位而不是以包为单位统计——身份合并（第八节）意味着一个 UID 上挂着多个包，数据暴露范围比包名列表更大。
5. **升级或降级**：若 `category_collisions` 与 `FIND_shared_uid` 同时命中，影响面按合并后的身份集合计算；若仅 `FIND_domain` 命中且厂商提供策略说明，可降为例外记录。

### 18.3 误报来源（实际清点中占比最高的四种）

| 现象 | 触发字段 | 为什么不是事件 |
| --- | --- | --- |
| `ps` 换实现/换参数导致列序变化 | `ps_parsed` < `ps_lines` | 采集问题，修解析而不是修设备 |
| 厂商自定义应用域 | `FIND_domain` L2 | 合法存在，需策略说明而非判定 |
| 一个包多个进程（主进程 + isolated） | `package_processes` > 1 | 第六节所述正常形态 |
| OTA 后上下文取值新增 | `new_context_values` > 0 | 与版本发布同源的变化，按变更管理而不是按事件 |

### 18.4 应用侧可自查项（不需要设备）

`sharedUserId`、`android:isolatedProcess`、私有目录使用、`flags` 中的备份/调试相关项，全部可以在构建产物与清单层面清点；这部分与构建体系篇的发布门禁合并执行成本最低（同一套流水线，同一批产物）。

---

## 十九、十二条常见误区

1. **"设备有 SELinux 所以有沙箱。"** 沙箱是四层叠加（第二节），Permissive 时标签层不拦截，此时只剩 DAC 一层——脚本把它列为最高优先级信号正是为此。
2. **"Enforcing 就等于策略严格。"** 域与规则由设备实现编写，本文只把"应用 UID 进程在特权/调试域"当作硬信号，其余（如未知域）要求索取策略说明而不是默认严格。
3. **"SDK ≥ 26 就等于 seccomp 生效且充分。"** 公开口径给的是"过滤器装在 zygote"这一安装位置，不是覆盖清单；只读采集看不到过滤器内容（第五节）。
4. **"每应用一个 UID 是无条件的。"** `sharedUserId` 是对该承诺的显式豁免（第八节），且这是文档里写明的机制，不需要任何缺陷。
5. **"同 UID 的两个应用只是'关系近一点'。"** 身份合并后，数据层的"默认不对其他应用开放"（第七节）在这两个包之间不再成立。
6. **"类别标签只是实现细节。"** 官方口径在 Android 12+ 明确类别用于隔离其他应用的数据（第四节）；不同 UID 共用同一套类别应当核对。
7. **"一个包出现多个进程说明有问题。"** 主进程 + isolated 进程是正常形态（第六节），脚本因此只对"域是否在应用域词表内"给判定。
8. **"没有异常输出说明设备正常。"** 本文的四档里 `L0-LOW` 专门表示"未采到/未解析/无法比对"。缺口样例的 `gap=7` 不是"7 项小问题"，而是"7 项不知道"（第十四节）。
9. **"漂移检测可以只看进程名。"** 必须用 `<名, UID, 上下文>` 三元组，否则同包多进程与身份集合变化都会被漏掉或误报（第十七节）。
10. **"沙箱逃逸 = 内核漏洞。"** 公开分类里的"提权"描述的是**影响**（破坏应用沙箱、损害系统完整性，第十二节引参考资料 18），根因可以是策略、配置、身份合并，也可以不是内存缺陷。
11. **"内存安全改造完成后这条边界就安全了。"** 配置类放宽（S4/S6）不会被语言层面的改造消除；趋势数字（第十三节）说明的是缺陷数量下降，不是边界放宽问题消失。
12. **"评估过一台样机就够了。"** 判定依赖构建变体、策略版本与厂商域定义，跨机型与跨 OTA 会漂移；覆盖率与新鲜度（M10）本身就是指标。

---

## 二十、趋势判断

- **口径会继续向"类别与域的最小化"收紧，而不是新增一层。**官方对增量方向的描述是配置与覆盖面的调整——"更新的 SELinux 配置与更大的 seccomp 覆盖进一步锁住应用沙箱并减少攻击面"（参考资料 22）。因此检测侧的重点应当从"有没有开"转向"开得多严、策略变更有没有被记录"。
- **按应用隔离的进程形态会继续替代共享解析进程。**第六节引用的"按应用沙箱进程"是媒体框架加固的方向：把解析不可信内容的代码放进只与该应用相关的进程里，使缺陷影响半径落在沙箱内。清点时应持续记录**某个解析动作发生在哪种进程形态里**。
- **可调试性会变成一个更容易被自动化引入的问题。**发布包带 `DEBUGGABLE` 的成因通常是构建变体或测试配置泄漏（第九节），而构建变体本身来自流水线；把 `FIND_debuggable` 与构建体系篇的 `test-keys`/指纹检查放在同一道门禁里，成本远低于事后在设备侧发现。
- **多用户与克隆形态会让 UID↔标签换算更容易踩坑。**本文给出的换算式来自逐字可读的三条 define（第三节），但 `u<i>_a<n>` 标签形态的官方逐字定义本次未取得；跨版本比对时应把不一致判为"核对"而不是"违规"。
- **设备侧可见性不会变好。**`getenforce`、`ps -AZ`、`dumpsys package` 这一层能观察到的东西十年间基本稳定；真正提升判定质量的手段是**留存基线 + 提高覆盖率**，这也是本文给两个脚本而不是给一个检查清单的原因。

---

## 二十一、分角色结论与常见问题

**应用/SDK 安全负责人**：三件事立刻可做——清点清单里的 `sharedUserId` 与 `isolatedProcess`（第八、六节）；确认发布构建不携带调试标志（第九节）；确认私有数据没有因为主动共享而后移保护范围（第七节）。

**ROM/OEM 系统工程师**：把 `getenforce` 与构建变体作为出厂与 OTA 后的双重断言；对厂商自定义域准备可交付的策略说明，否则评估方只能把它记成 L2 待核对项（第四节）。

**MDM 与安全基线团队**：按 M1–M10 起步，先保证 M9（缺口率）与 M10（覆盖率）达标，再谈其他八项的阈值；`FIND_priv_ctx` 与 `FIND_identity` 命中直接进事件流程（第十八节）。

**检测与取证工程师**：以 UID 而不是包名为影响面单位；固定证据时保留 `PS_ROW` 原始行；任何"未采到"都不写入"通过"（第十四、十八节）。

**采购与供应商评估**：要求供应商对四个字段给出可核验答案——MAC 状态、构建变体、应用域定义清单、快照基线与比对能力。答不出第四项意味着这台设备上的隔离变化无法被发现。

**Q：能不能只查 `getenforce` 就下结论？** A：不能。Permissive 是最强信号但非唯一：第八、九节的两种放宽在 Enforcing 设备上同样成立，而第十一节的组合失效需要多字段同时可见。

**Q：`ps -AZ` 的列序稳定吗？** A：不保证。脚本因此不取固定列，而是"哪个字段形如 SELinux 上下文就取它"，并统计解析成功率；解析不达标时判定范围被显式缩小。

**Q：能不能在没有基线的情况下用脚本 2？** A：不能，也不应该制造假的"无变化"结论。脚本会输出 `FIND_input,L0-LOW`，把这次比对标为不成立。

**Q：`isolated_app` 进程没出现是不是问题？** A：不是。脚本把它记为 `L0-LOW` 并明确"不作为问题判定"——它取决于设备与在用组件（第六节）。

**Q：这套判定能不能直接用于别人的设备？** A：不能。全部命令为只读采集，但仍要求你对目标设备有测试授权；未授权设备的取证属于另一类问题（见攻击面方法论篇的授权前提）。

**Q：本文为什么不给出具体 CVE 案例？** A：逐条缺陷与在野利用归因在内核篇与 Framework 篇，本文的分工是机制层与检测层；重复列案例只会让两边版本失配。

---

## 二十二、参考资料

以下条目为本次核验实际取得逐字内容的公开来源，编号供正文与脚本引用；引文为原文（英文）摘录，破折号后为中文口径说明。截断处以"（原文截断）"标注。
1. 应用沙箱的身份承诺 — Application sandbox（source.android.com）："To do this, Android assigns a unique user ID (UID) to each Android app and runs it in its own process." — 每应用一个唯一 UID，并运行在自己的进程中。https://source.android.com/docs/security/app-sandbox
2. 安全特性总览（用户级保护）："The Android platform takes advantage of the Linux user-based protection to identify and isolate app resources…"（原文截断） — https://source.android.com/docs/security/features
3. 安全总览（沙箱覆盖范围）："Both ART and native apps run within the same security environment, contained within the Application Sandbox." — https://source.android.com/docs/security/overview
4. SELinux 总览（MAC 覆盖对象）："Android uses Security-Enhanced Linux (SELinux) to enforce mandatory access control (MAC) over all processes, even"（原文截断） — https://source.android.com/docs/security/features/selinux
5. SELinux 概念（类型即域）："For instance, by default, an app has the type `untrusted_app`. For a process, its type is also known as its domain." — https://source.android.com/docs/security/features/selinux/concepts
6. SELinux 概念（类别用途，Android 12 及以上）："In Android 12 and higher, categories are used to: Isolate the app data from access by another app,"（原文截断） — https://source.android.com/docs/security/features/selinux/concepts
7. `<manifest>` 的 `sharedUserId` 定义："The name of a Linux user ID that is shared with other apps. By default, Android assigns each app its own unique user ID." — https://developer.android.com/guide/topics/manifest/manifest-element
8. `<service>` 的 `android:isolatedProcess` 定义："If set to \"true\", this service runs under a special process that is isolated from the rest of the system and has no permissions of its own." — https://developer.android.com/guide/topics/manifest/service-element
9. 应用 UID 区间定义（AOSP 头文件，逐字）：`#define AID_APP_START 10000 /* first app user */`、`#define AID_APP_END 19999`、`#define AID_USER_OFFSET 100000 /* offset for uid ranges for each user */` — https://android.googlesource.com/platform/system/core/+/master/libcutils/include/private/android_filesystem_config.h
10. 隔离进程 UID 区间（同一文件的历史分支）：`#define AID_ISOLATED_START 99000 /* start of uids for fully isolated sandboxed processes */` — https://android.googlesource.com/platform/system/core/+/android-9.0.0_r30/libcutils/include/private/android_filesystem_config.h
11. 系统调用层的安装位置（企业安全白皮书 2018）："As of Android 8.0, a seccomp filter is installed into zygote, the process from which all the Android apps are derived." — https://source.android.com/static/docs/security/overview/reports/Google_Android_Enterprise_Security_Whitepaper_2018.pdf
12. 同一口径的发布时表述（Android Developers Blog, 2017-07-20）："Android O includes a single seccomp filter installed into zygote, the process from which all the Android applications are derived." — https://android-developers.googleblog.com/2017/07/seccomp-filter-in-android-o.html
13. 兼容性侧对 seccomp 的要求（CDD）："…enabling the seccomp-BPF with threadgroup synchronization (TSYNC)…"（原文截断） — https://source.android.com/docs/compatibility/cdd
14. Zygote 机制文档页（本页存在，本次未取得可逐字引用的预加载/派生句，正文不据此立论） — https://source.android.com/docs/core/runtime/zygote
15. 媒体框架加固（进程形态）："app-specific sandboxed processes." — https://source.android.com/docs/core/media/framework-hardening
16. 应用专属数据区（默认不可访问）："Your data is secure, because by default this area is not accessible to other apps or the user." — https://developer.android.com/training/data-storage/sqlite
17. 调试能力与设备条件（Android Studio 调试文档）："The device supports run-as …"（原文截断） — https://developer.android.com/studio/debug
18. "破坏应用沙箱"的公开归类（Android Security 2016 Year in Review）："Privilege escalation. An application that compromises the integrity of the system by breaking the application sandbox, or changing or disabling access to …"（原文截断） — https://source.android.com/static/docs/security/overview/reports/Google_Android_Security_2016_Report_Final.pdf
19. 内存安全趋势数字（Google Online Security Blog, 2024-10）："…dropping from more than 220 in 2019 to a projected 36 by the end…"（原文截断） — https://security.googleblog.com/2024/10/safer-with-google-advancing-memory.html
20. 系统服务代应用执行的风险入口（confused deputy；本次未取得可逐字引用的定义句，正文按机制说明，不引用原文） — https://developer.android.com/privacy-and-security/risks/content-resolver
21. 内存安全占比的归因表述（Prossimo，转述 Google 估计）："Google estimated that 90% of Android vulnerabilities are memory safety issues." — https://www.memorysafety.org/docs/memory-safety/
22. 增量方向的官方表述（Security Enhancements 页）："Updated SELinux configuration and increased seccomp coverage further locks down the Application Sandbox and reduces attack surface." — https://source.android.com/docs/security/enhancements

**未公开文档化 / 本次未逐字核验的边界（明确列出，正文与脚本不依赖以下任何一项）**

- **`u0_aNNN` 这类 SELinux 标签（seInfo）的命名规则定义页**：本次未找到官方逐字页面。第三节的换算只依赖参考资料 9 的三条 `#define`；脚本 1 的 `FIND_uid_label` 因此把不一致判为"核对"（L2-WARN）而不是"违规"。
- **`dumpsys package` 输出字段（`userId`、`seInfo`、`sharedUserId`、`flags`）的官方定义**：未取得逐字文档，脚本把这些当作**观测字段**使用；字段缺失一律记为 `L0-LOW` 数据缺口。
- **SELinux 类别的分配方案**（哪些 `cXXX` 表示应用编号、用户、共享权限等）：参考资料 6 只给出类别用途的第一条，本次未逐字核验分配表；脚本因此**只比较类别是否相同**，不解释其含义。
- **`AID_ISOLATED_START` 在当前分支上的取值**：不同分支读到不同数值（参考资料 10 为历史分支的 99000）；正文与脚本均不硬编码隔离 UID 区间。
- **Zygote 预加载与派生的逐字机制描述**（参考资料 14 页面存在，句子未取得）：第六节的推论建立在"过滤器装在 zygote"（参考资料 11、12）与进程形态文档之上，不依赖未取的句子。
- **`run-as` 与 `ro.debuggable`、包标志之间的确切前置条件**：参考资料 17 只给出一句设备条件；第九节因此不给出任何使用步骤，只说明"调试通道存在时该包的沙箱不再对用户之外的一方成立"。
- **`/proc` 可见性限制（hidepid 类机制）在 Android 上的具体配置**：本次未逐字核验；本文不把"能否看到其他应用进程"当作判定条件。
- **应用侧类加载器/DEX 层的按应用隔离描述**：未取得逐字来源，本文不展开该层。
- **共享媒体服务进程的历史特权表述**：未取得逐字来源，第六节只使用"按应用沙箱进程"这一已核验表述。
- **安全公告的漏洞类型图例逐字页**：本系列已在[远程代码执行分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)与[Framework 信任边界分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)中处理公告口径，本文不重复立论。
- **"Android 16 内存安全语言占比"一类数字**：未取得可靠来源，第十三节不使用。
- **厂商自定义域的策略内容与设备侧实际 seccomp 过滤器内容**：不属公开文档范围，本文一律不给结论，只列为需要向厂商索取的证据（第十一、十四、十八节）。
- **具体 CVE、漏洞与利用细节**：不属本文范围；相关口径见[内核漏洞与提权防御](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)与[攻击面与漏洞研究方法论](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)两篇。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
