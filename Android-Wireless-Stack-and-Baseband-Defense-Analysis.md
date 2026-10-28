---
layout: default
title: "Android 无线协议栈与基带缺陷防御分析报告"
description: "无线侧的六层构成（芯片固件、内核无线子系统、用户态协议栈、HAL、Framework 权限、设备与企业策略）、三类经常被混为一谈的口径（Wi-Fi 协议栈缺陷 / 蓝牙配对与认证缺陷 / 基带固件缺陷）、W1–W10 缺陷类型学与 CWE 对齐、FragAttacks/KRACK/Dragonblood/BIAS/KNB/BlueBorne/BLEEDINGBIT/Exynos 八组公开案例的读数方式、基带为什么单列一档以及它的可见性边界、层×责任×更新通道×设备侧证据的归属矩阵、四级判定模型与十项运营指标、两个只读核验脚本（无线姿态快照与两次快照的漂移比对）"
lang: zh-CN
permalink: /Android-Wireless-Stack-and-Baseband-Defense-Analysis/
---
# Android 无线协议栈与基带缺陷防御分析报告

**Android Wireless Stack and Baseband Defensive Analysis: Layering, Weakness Taxonomy, Attribution of Proximal-Range Defects, and Read-Only Posture Verification**

- **报告日期**：2026-10-27
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动平台与无线系统安全负责人、OEM/ODM 的 Wi-Fi 与蓝牙子系统工程师、基带与调制解调器固件团队、移动安全评估与漏洞治理团队、企业移动管理（EMM/MDM）与安全基线团队、设备风控与取证工程师、应用商店与供应链审核人员
- 视角声明：本文写的是**"无线侧的缺陷落在哪一层、由谁修、通过哪条通道到达设备、以及这些判断中哪一部分能在设备侧被只读观察出来"**。**投递通道的完整清点、代码执行链与内存缺陷利用细节、内核子系统实现、权限判定缺陷的具体案例不在本文**，那些内容在姊妹篇里，本文只做交叉引用而不重复展开：
  - [Android 远程代码执行分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（四条投递通道之一即"近场无线"；那篇讲外部输入怎么到达解析代码，本文讲到达之后**这一层由谁维护、修好了显示在哪个版本号上**）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（SPL 两档语义与 S1–S5 链条；本文只把 SPL 当作**时间锚点**使用，并说明为什么无线侧的到达性它答不了）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（APEX 与 Mainline 的构建与装配口径；本文第十五章把 Wi-Fi/蓝牙模块当作**更新通道**而非构建产物来用）
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（UID/SELinux 域/seccomp 三层叠加；本文讲无线进程在这套约束里**处于哪个域、域之外还有哪些 vendor 侧进程不受它管**）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层与信任边界的整体坐标；本文的六层是其中"无线"这一竖切的放大图）
  - [Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)（内核子系统与设备节点供给；本文只使用 cfg80211/mac80211/nl80211 的公开接口口径，不重复展开内核实现）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（权限与身份判定缺陷；本文第十六章讲的是**权限位应当怎样配置**，不判定判定逻辑本身）
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)（安全元素与密钥存储；本文的 NFC off-host 与 OMAPI 一节依赖这一侧的口径）
  - [Android 厂商硬件安全组件形态与实现差异分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html)（器件形态与可外部核验信号；本文第十三章讲的是**无线器件的固件公告可见性**，两者互补）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面与 R0–R4 分级；本文的类型学回答"这一域上可能站着哪一类缺陷"）
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)（应用侧声明的弱点形态；本文的判定同样落在"证据从哪来"这条线上，但对象是系统无线栈而不是应用包）
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)（内核侧缺陷类型学、缓解矩阵与提权取证；本文的 L2 层只按公开接口口径引用 cfg80211/mac80211，不重复展开内核缺陷本身）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（协议词表、身份与 SELinux 判定、Stable AIDL；本文 L4 层引用的 HAL 接口语言记号按该篇口径读取）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（本文写无线侧的分层与责任、"邻近即远程"的严重度基线；该篇沿用同一口径解释**为什么本地权限的侧信道公告通常仍按高严重度处理**，并把可见性边界从固件公告换成微码与 CPU 厂商公告）
- 范围声明：全文只讨论**公开文档化**的无线分层、公开披露的缺陷类别与公告口径，以及在这些公开规则之上可安全实施的**只读**清点与比对流程。**不提供任何可用的漏洞利用代码、内存布局偏移、原语构造、空口报文构造、策略绕过步骤或可操作的攻击流程；不给出任何针对具体机型或固件的攻防操作。**所有"核验"指读者对**自有或已获授权测试的设备**执行采集。本文两个脚本一个只做设备侧只读采集（`get-state` / `getprop` / `settings get` / `dumpsys` / `pm list features` / `toybox ls /apex` 六类命令），一个只做离线文本比对；两者都不写设备、不安装任何东西、不访问网络、不打印 SSID 与蓝牙设备地址。

---

## 一、摘要：无线侧不是一个攻击面，是三条归属完全不同的通道

"Android 的无线漏洞"这句话在日常讨论里同时指三件不同的事：Wi-Fi 协议实现里的解析缺陷、蓝牙配对与认证流程里的设计缺陷、以及基带固件里的内存缺陷。三件事共享"空口"这个物理入口，但**修复责任人、更新通道、公告出处和设备侧可见性四样属性完全不同**，把它们放进同一张检查表，是无线侧风险治理里最常见的失效模式：表上每一行都有归属，实际每一行的整改路径不同、SLA 不同、能不能自证也不同。

本文的立场是：**无线侧的风险管理单位不是"漏洞"，而是"层 × 责任人 × 更新通道 × 设备侧证据"这个四元组**。四个成分缺一个，一条缺陷就无法被派工，也无法被验证。为此本文做四件事：

1. 把无线侧拆成六层（芯片固件 / 内核无线子系统 / 用户态协议栈 / HAL / Framework 与权限 / 设备与企业策略），逐层给出**公开定义句与代码位置**（第三、四章），并说明每层的进程身份与执行域（第五章）；
2. 把公开口径里的攻击者位置（remote / proximal / local）单独成章（第六章），因为 Android 官方对"邻近即远程"有一句关键定义：**"Proximal attack vectors are considered remote."**（参考资料 17）——这句话决定了无线缺陷在公告里的严重度基线，也解释了为什么"要挨得很近"从来不等于"风险低"；
3. 给出十类缺陷类型学 W1–W10（第七章）、与 CWE 的对齐关系（第八章），并用八组公开案例逐类读数（第九至十一章）；
4. 把"能在设备侧只读观察到什么、观察不到什么"做成两个脚本与一份归属矩阵（第十三、十四、十七、十九至廿四章）。

三条最重要的结论：

- **基带是本文里唯一一条"读不到就无解"的通道。** 平台侧对它的公开表述是防御性口径，Google 的说法很直白："We particularly prioritize hardening the cellular baseband given its unique combination of running in an elevated privilege and parsing untrusted inputs that are remotely delivered into the device."（参考资料 26）。但在设备侧，能拿到的通常只有 `gsm.version.baseband` 一个版本串，公告正文常常在厂商登录墙之后甚至已经下线（第十三章）。所以本文把基带单列一档（第十二章），并在脚本里把"版本读不到"判为 `L0-LOW` 缺口而不是"无风险"。
- **无线侧最常见的暴露不是缺陷，而是配置。** 已保存网络里存在 `keyMgmt=NONE`、存在 WEP/TKIP、蓝牙处于可发现状态、既有绑定使用旧配对方式、NFC 开启而 Secure NFC 关闭、Wi-Fi 关闭时仍允许扫描——这六项都不需要任何漏洞就能被利用面吸收，而且**六项全部可以在设备侧只读观察**（脚本 1 的 `net_open`、`net_wep`、`net_tkip`、`bt_discoverable`、`bt_legacy_pairing_marker`、`nfc_lockscreen_emulation`、`wifi_scan_always_on`）。
- **单次快照证明不了"变好或变坏"，只有留存基线做差分层才证明得了。** 脚本 2 因此被设计成离线比对而不是设备采集，并且把两种"消失"分开处理：采集面未收窄时的消失记 `risk_closed_needs_proof`（仍需一条变更依据），采集面收窄时的消失记 `risk_gone_by_coverage`（`L3-HIGH`，明确写"这一消失应读作采集面收窄，不能读成已修"）。第廿三章样例三给出这条判定的真实输出：同一批判据里缺口数由 5 变为 17，12 个高危与中危信号码全部"消失"，脚本把它们全部读作采集退化而不是整改成果。

从可执行验证的结果看：脚本 1 的 65 个信号码、脚本 2 的 39 个信号码，共 **104 个码位**，在七组合成设备档案与八组快照对上**全部至少触发一次**（第廿四章给出覆盖核对表与验收流程）；本文发布的样例输出中，脚本 1 四段、脚本 2 三段，最长一段 68 行。这是本文对"无线侧风险可治理"的全部要求：**每一层有责任人、每一条通道有版本证据、每一个读不到的字段被记成缺口而不是通过。**

---

## 二、先分清三套口径：协议栈缺陷、配对与认证缺陷、基带固件缺陷

三套口径经常被并列成"无线漏洞的三种"，实际它们的判定对象、证据形态和整改路径都不一样。

| 口径 | 判定对象 | 典型公开出处 | 修复责任人 | 到达设备的通道 | 设备侧可观察性 |
| --- | --- | --- | --- | --- | --- |
| Wi-Fi 协议栈缺陷 | 握手、组加密、帧解析、分片与聚合的实现与配置 | ASB（AOSP 侧条目）、上游内核、认证机构通告 | AOSP 维护者 / 内核上游 / supplicant 维护者 / 芯片厂商（分层负责） | 整机镜像、`com.android.wifi` 模块、固件包 | 中：已保存网络的密钥管理形态、HAL 接口语言记号可读 |
| 蓝牙配对与认证缺陷 | 配对流程、密钥强度、既有绑定的认证假设 | ASB（`system/bt` 相关条目）、漏洞通报机构、厂商通告 | AOSP 蓝牙栈维护者 / 蓝牙控制器固件厂商 | 整机镜像、`com.google.android.bt` 模块（可选）、控制器固件 | 中高：射频状态、可发现性、绑定与配对强度记号可读 |
| 基带固件缺陷 | 调制解调器内部协议栈的解析与状态机 | 芯片厂商安全公告（常带访问限制）、ASB 中的芯片条目 | 基带芯片厂商为主，OEM 负责集成 | 基带固件包，通常随整机镜像 | 低：一般只有 `gsm.version.baseband` 一个版本串 |

三处容易被混用的差别值得单独说明。

**"空口可达"不等于"同一类缺陷"。** Wi-Fi 侧的公开案例多数是**协议设计与实现交织**的问题：KRACK 是密钥重安装导致的 nonce 复用（参考资料 22），Dragonblood 是 SAE 握手在特定条件下的密码学协议问题（参考资料 23），FragAttacks 是分片与聚合的处理问题（参考资料 21）。蓝牙侧的公开案例更多落在**配对与认证的假设**上：BIAS 的公开结论是"It is possible for an unauthenticated, adjacent attacker to impersonate a previously paired/bonded device and successfully authenticate without knowing the link key."（参考资料 24）。基带侧的公开案例则基本是**固件内部解析器的内存安全**问题（参考资料 26、27）。这三类的整改动作分别落在"升级协议实现 / 更换配对假设 / 重编固件并重新签发"，把它们写进同一行工单等于没有派工。

**"归属哪一层"和"归属谁"不是同一件事。** 同一个 Wi-Fi 缺陷可能同时涉及内核（`cfg80211`/`mac80211`）、用户态 supplicant、vendor HAL 与控制器固件四层；内核侧文档对这种分层有责任边界句："cfg80211 is the configuration API for 802.11 devices in Linux. It bridges userspace and drivers..."，并要求"cfg80211 must, directly or indirectly via mac80211, be used by all modern wireless drivers in Linux, so that they offer a consistent API through nl80211."（参考资料 1）——驱动与厂商实现被要求走同一套接口，这正是"一次协议层修复能否覆盖全部器件"的前提。含糊的是**具体某一次设备更新的装配**：这是第十四章归属矩阵要解决的问题。

**公告里的严重度不是你的暴露度。** ASB 对严重度评估有明确前提：severity assessment"is based on the effect that exploiting the vulnerability would possibly have on an affected device, assuming the platform and service mitigations are turned off for development purposes..."（原文截断，参考资料 18）。这句话在无线侧尤其重要，因为平台侧缓解（SELinux 域、seccomp、运行时权限、Mainline 模块）与协议侧缓解（Secure Connections、SAE、PMF）分属不同层，某一层的开启会让另一层的严重度评估显得"过估"，但**任何一层的开启都不是永久承诺**——第十七章的取证流程之所以要求逐层记录状态，就是这个原因。

为什么本文以"层"为主轴而不是以 CVE 编号为主轴：编号是**一次披露**的粒度，而无线侧的治理单位是**一条长期存在的实现通道**。以编号为主轴的报告会在下一批公告出来后失效；以层为主轴的报告可以把新编号挂到既有层上。CWE 与 ASB 编号在本文里退居交叉索引位（第八章、第九至十一章），这样脚本输出既能被公告检索接住，又不假装自己是某套编号体系的实现。

---

## 三、六层分层地图：从芯片固件到企业策略

把无线侧画成六层，是为了让"一条缺陷在哪、由谁修"有唯一答案。每一层给出它的边界条件与观察点，公开定义句在第四章逐条列出。

```
L6  设备与企业策略     用户开关 / EMM 策略 / 运行时权限授予记录
L5  Framework 与权限    WifiManager·BluetoothManager·NfcAdapter 等系统服务与权限判定
L4  HAL                Vendor / Supplicant / Hostapd 三个 Wi-Fi 面 + 蓝牙 HAL（HIDL→AIDL）
L3  用户态协议栈        wpa_supplicant / hostapd / 蓝牙栈（进程 com.android.bluetooth）/ NFC 服务
L2  内核无线子系统      cfg80211 · mac80211 · nl80211 · 驱动与固件加载接口
L1  芯片固件与基带      Wi-Fi/BT 控制器固件 · NFC 控制器（NCI）· 蜂窝调制解调器（独立处理器）
```

**分层之间的三条硬边界值得先说明。**

第一条在 **L1 与 L2 之间**：蜂窝调制解调器不是 Android 里的一个进程，而是一颗独立处理器上的固件。平台侧对它的公开口径是把它当作"运行在高特权、解析远程不可信输入"的对象来加固："Android's defense-in-depth strategy applies not only to the Android OS running on the Application Processor (AP) but also the firmware that runs on devices."（参考资料 26）。这意味着 L1 的缺陷**不经过 L2–L5 的任何一层检查**就能被触发，也意味着本文所有设备侧只读手段对它基本失效。

第二条在 **L3 与 L4 之间**：Bluetooth 一侧"Vendor devices interact with the Bluetooth stack using the HAL interface definition language (HIDL)."（参考资料 2），Wi-Fi 一侧则存在三个 HAL 面："three Wi-Fi HAL surfaces ... Vendor HAL, Supplicant HAL, and Hostapd HAL."（参考资料 5）。vendor 侧进程的构建、更新与策略归属和系统侧不同——这一条差异是第十四章归属矩阵里最容易漏的一列。

第三条在 **L5 与 L6 之间**：应用能不能碰到无线，取决于运行时权限与策略；而设备本身有没有这一层无线能力，取决于 L1 的器件。本文脚本用 `pm list features` 与 `/apex` 清单分别记录这两件事，并在两者读不到时判为缺口而不是判为"没有"（第二十章码表里的 `*_feature_unknown` 与 `*_module_unread`）。

分层地图还有一个副作用：**同一个"关闭"动作在不同层含义不同**。用户把蓝牙关掉（L6）会停止射频，但**不会删除既有绑定**，而公开口径恰恰把"已成功配对的设备"算作本地身份（参考资料 17）——第廿一章样例一里那句"历史绑定不会随关闭射频消失"就是这个意思。关闭 NFC（L6）与"off-host 卡模拟在锁屏下被禁止"（L1+L5 的 Secure NFC）也是两件事，后者"introduced in Android 10"并要求控制器支持 NCI 2.0（参考资料 9）。

---

## 四、各层的公开定义句与代码位置

本章把第三、五、六章用到的每一句平台表述落到具体来源，标注为原文（英文，已按页面渲染归一化空白）与口径说明。凡是本章没有列出的表述，正文一律不作为已知前提。

**L2：内核无线子系统。** cfg80211 的公开定义句整段是："cfg80211 is the configuration API for 802.11 devices in Linux. It bridges userspace and drivers, and offers some utility functionality associated with 802.11. cfg80211 must, directly or indirectly via mac80211, be used by all modern wireless drivers in Linux, so that they offer a consistent API through nl80211."（参考资料 1）。这句话给出三件事：无线驱动的**统一配置入口**、userspace 与 driver 之间的**桥**、以及"所有现代无线驱动必须走这套接口"的强制约定。本文第七与第十四章把"内核侧接口一致性"当作 Wi-Fi 缺陷能否被一次修复覆盖多器件的前提条件使用。

**L3：蓝牙栈与进程形态。** "Android provides a default Bluetooth stack that supports both Classic Bluetooth and Bluetooth Low Energy (BLE)."（参考资料 2）；应用侧"A Bluetooth app communicates with the Bluetooth process through Binder."（参考资料 2）。协议栈进程的 SELinux 身份可以从公开策略文件逐行读到：`user=bluetooth seinfo=bluetooth domain=bluetooth type=bluetooth_data_file`，同文件里还有 `user=nfc … domain=nfc` 与 `user=radio … domain=radio` 的对应条目（参考资料 19）。这比"进程叫什么名字"更有用：**它给出用户标识、seinfo 与执行域三者的一次映射**，也就是本文第五章讨论"身份与域是否一致"时的参照物。

**L3：蓝牙控制器接口与日志。** "The Host Controller Interface (HCI) is used for interacting with a Bluetooth controller."（参考资料 3）；取证侧的公开页面记录了 HCI 抓取的落点与隐私约束——日志存放位置"stored in `data/misc/bluetooth/logs`"，以及"For privacy reasons, always-on, … BTSnoop only logs non-personal information and events."（原文截断，参考资料 4）。SELinux 侧对这条通道有独立标签：`/data/misc/bluetooth/logs(/.*)? u:object_r:bluetooth_logs_data_file:s0`（参考资料 20）。这三句合起来是第十七章的一条独立结论：**无线取证数据本身是被单独打标、被策略管控的一类数据**，"抓 HCI 日志"在受管设备上不是随手可做的动作。

**L3/L4：Wi-Fi 服务与三个 HAL 面。** "The Wi-Fi service communicates with the Wi-Fi HAL over AIDL."；服务侧存在独立进程"a standalone process, **wificond**, located at `system/connectivity/wificond`"；HAL 侧有三个面"three Wi-Fi HAL surfaces ... Vendor HAL, Supplicant HAL, and Hostapd HAL."（参考资料 5）。版本口径："…all three interfaces are available as stable Android Interface Definition Language (AIDL) packages. These interfaces were previously defined using Hardware Interface Definition Language (HIDL)."（参考资料 6）。WPA3/OWE 侧："Android 10 introduces support for the Wi-Fi Alliance's (WFA) Wi-Fi Protected Access version 3 (WPA3) and Wi-Fi Enhanced Open standards."，且"To support WPA3 and Wi-Fi Enhanced Open, implement the Supplicant HAL interface. Starting with Android 13, the interface uses Android Interface Definition Language (AIDL) for the HAL definition."（参考资料 7）。

**L3：密钥管理形态的公开语义。** 这是第七章 W2/W3 类判定和脚本 1 网络清点的语义基础，四句均取自同一页面（参考资料 7）：

- SAE："WPA3-Personal: Uses simultaneous authentication of equals (SAE) instead of pre-shared key (PSK), providing users with stronger security protections against attacks such as offline dictionary attacks, key recovery, and message forging."
- OWE："Wi-Fi Enhanced Open is a new WFA security standard for public networks based on opportunistic wireless encryption (OWE). It provides encryption and privacy on open, non-password-protected networks…"，同一句的落点部分是"Enhanced Open doesn't provide authentication."
- 过渡模式一："WPA2/WPA3 transition mode: The serving access point supports WPA2 and WPA3 standards concurrently. In this mode, Android 10 devices use WPA3 to connect, and devices running Android 9 or lower use WPA2 to connect to the same access point."
- 过渡模式二："OWE transition mode: The serving access point supports both OWE and open standards concurrently. In this mode, Android 10 devices use OWE to connect, and devices running Android 9 or lower connect to the same access point without any encryption."

**这最后两句是本文判级的关键依据**：过渡模式在定义上就允许"同一接入点下，老设备走 PSK / 老设备完全不加密"。因此脚本不把"存在 PSK 网络"直接判成缺陷，而是用 `net_psk_only`（存在 PSK 且本次未见任何 SAE 记录）表达"降级暴露面未被关闭"；把 OWE 记为 `L1-INFO` 而不是强认证，因为官方句子明确"doesn't provide authentication"。

**L3：Passpoint。** "The supplicant provides support for the 802.11u standard, specifically network discovery and selection features such as Generic Advertisement Service (GAS) and access network query protocol (ANQP)."（参考资料 8）——这句话把网络发现放在 supplicant 对 802.11u 的支持上，因此**发现阶段的解析面是平台与固件共同承担的一层**，脚本据此把 `wifi_passpoint_present` 记为"厂商责任面存在"的中性事实而不是加固项。

**L3/L1：NFC。** Secure NFC 的定义句是"Secure NFC is a feature introduced in Android 10 that allows off-host card emulation to be enabled only when the device's screen is unlocked."，前置条件是"the device must have an NFC controller that supports the NCI 2.0 standard..."，机制背景是"Android supports off-host card emulation, which means NFC card emulation with a secure element."（参考资料 9，部分原文截断）。主机侧卡模拟与 off-host 的共存规则："This coexistence is based on a principle called *AID routing*..."，以及一条明确的边界句："Android never starts or binds to a service that is declared as \"off-host\"..."（参考资料 10）。安全元件访问侧："Open Mobile API (OMAPI) is a standard API used to communicate with a device's Secure Element."（参考资料 11）。

**L5/L6：权限与策略。** 蓝牙运行时权限："The `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, and `BLUETOOTH_SCAN` permissions are runtime permissions."（参考资料 14）；扫描结果的隐私过滤有整句："If you include `neverForLocation` in your `android:usesPermissionFlags`, some BLE **beacons** are filtered from the scan results."（参考资料 14）——注意官方措辞是 beacons（早期文档曾写作 sensors，本文采用当前页面的整句）；同一页面还给出选择依据："If your app uses Bluetooth scan results to derive physical location, declare the ACCESS_FINE_LOCATION permission."（参考资料 14）。Wi-Fi 侧扫描结果的约束："An app must hold ACCESS_FINE_LOCATION … and ACCESS_WIFI_STATE permission in order to get valid results."（原文截断，参考资料 16）。管理侧两句取自 `UserManager`：`DISALLOW_BLUETOOTH` 的"Specifies if bluetooth is disallowed on the device…"，以及 `DISALLOW_CONFIG_BLUETOOTH` 的"**This does not restrict the user from turning bluetooth on or off.**"（参考资料 15）——这两句的差别是第十六章策略基线的核心：**能不能用**与**能不能改设置**是两个不同的策略位，而后者不控制射频开关。

**L4/L5：Mainline 更新通道。** "Android 10 introduced Mainline (formerly called modular system components on this site)."，模块表中 Wi-Fi 行以 `com.android.wifi` 形态出现（参考资料 12）。蓝牙侧有专页与明确限定："The Bluetooth module is an optional Mainline module introduced in Android 13. In Android 16 and higher, the module is updatable and contains a fully certified dual-mode Bluetooth…"（原文截断，参考资料 13）。"optional"这个词在脚本判级里被显式使用：模块缺席只判 `L0-LOW`（通道未知），不判风险，也不判"未加固"。

---

## 五、边界与身份：进程、执行域与 vendor 侧

无线侧的隔离讨论必须比应用侧多一个维度：**这条通道上有相当一部分代码根本不运行在 Android 的应用沙箱里**。三类身份值得分开记。

**系统侧进程身份。** 公开策略文件里能逐行读到无线相关身份的映射："user=bluetooth seinfo=bluetooth domain=bluetooth type=bluetooth_data_file"、"user=nfc seinfo=nfc domain=nfc type=nfc_data_file"、"user=radio seinfo=platform domain=radio type=radio_data_file"（参考资料 19）。这类条目的价值不在于名字，而在于它们把**用户标识、seinfo 与执行域**绑成一条可核对的映射：与[应用沙箱篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)一样的方法可以复用——**身份与域是两个独立事实，二者不一致时才是高信号**。无线侧日志数据另有独立标签（`bluetooth_logs_data_file`，参考资料 20），说明取证通道本身在策略里是一等公民。

**vendor 侧进程。** Wi-Fi 的三个 HAL 面与蓝牙 HAL 分属 vendor 侧（参考资料 2、5）。这一侧的构建身份、更新节奏与缺陷归属都跟着芯片厂商，第十四章把"HAL 层"单列一行，正是因为它既不完全归 AOSP、也不完全归整机镜像。**HIDL→AIDL 的迁移**因此不是风格问题：官方口径是"…all three interfaces are available as stable Android Interface Definition Language (AIDL) packages. These interfaces were previously defined using Hardware Interface Definition Language (HIDL)."（参考资料 6）——接口稳定性直接决定"系统侧升级时 vendor 侧要不要一起动"，也就是决定修复能不能拆开走。本文脚本把 `dumpsys wifi` 里的接口语言记号读成 `wifi_hal_iface_aidl` / `wifi_hal_iface_hidl` / `wifi_hal_iface_unknown` 三态（第二十章），并把"读不到"明确判为缺口。

**独立处理器上的固件。** 基带不属于上面任何一类。平台侧对它的公开表述是把它纳入纵深防御的对象（参考资料 26，第四章已引整句）。这句话承认了 Android 侧策略对它的有效范围有限——这也是为什么本文在第七章把 W7（空口解析的内存缺陷）标为"归属主要在 L1"，而它的观察点在设备侧几乎只剩版本字符串。

身份边界在无线侧还有一个反直觉的后果：**权限收紧不等于通道关闭**。运行时权限（参考资料 14）管的是"应用能不能调用蓝牙/Wi-Fi API"，而空口可达性由射频状态与可发现性决定；一个没有任何权限 granted 的设备，只要蓝牙开着且可发现，仍然处在 W6（配对与认证绕过类）的暴露形态里。脚本因此把两类事实分行输出：`FEAT`/`SET` 行记录能力与开关，`ROW` 行记录暴露形态，绝不把"应用没权限"写成"设备没风险"。

---

## 六、攻击者位置口径：邻近即远程

无线侧最容易被误读的是"攻击者要多近"。Android 官方对安全更新资源页里的攻击向量口径给出五句关键定义（参考资料 17）：

> "A remote attack vector indicates that the bug can be exploited without installing an app or without physical access to a device."
> "Proximal attack vectors are considered remote."
> "Local attacks require the attacker to have prior access to the victim."
> "Successfully paired devices (such as Bluetooth companion devices) are considered local."
> "Physical attack vectors are considered local."

这五句合起来给出四条可执行判据，本文的类型学与脚本判级都按它们走。

**判据一：空口可达即远程。** "Proximal attack vectors are considered remote." 意味着 Wi-Fi/蓝牙/NFC 范围内触发、且无需安装应用或物理接触的缺陷，在口径上就是远程。这条判据解释了为什么公开案例里的严重度基线普遍偏高——ASB 里蓝牙条目的官方措辞正是这个组合：一句"could enable a remote attacker using a specially crafted transmission to execute arbitrary code within the context of a privileged process"（参考资料 18）。本文不复制任何构造该"特制传输"的细节；这条措辞在此只用于说明**严重度为什么不应被"距离"说服**。

**判据二：已配对设备算本地身份。** "Successfully paired devices (such as Bluetooth companion devices) are considered local." 这句在治理上有两个方向的用途：一方面它让"必须已配对才能触发"的缺陷在口径上降级；另一方面它把**绑定记录本身变成了需要保护的资产**——一台被窃取并解锁的设备，其上既有绑定在口径上属于"本地凭据"，而 BIAS 类研究的公开结论恰恰是"无需知道链路密钥即可伪装成已配对设备"（参考资料 24）。本文因此把既有绑定的**配对强度记号**（Secure Connections 为真 / 旧配对方式）作为独立信号码输出，而不是只输出"有几个绑定"。

**判据三：物理接触算本地，但无线侧的物理与邻近经常重叠。** "Physical attack vectors are considered local." 与 proximal 的定义之间没有互斥声明，因此对"贴近设备几厘米即可触发"的近场缺陷（NFC 类），单一口径不足以描述风险，必须同时记录**用户在场**与**屏幕状态**——这正是 Secure NFC 存在的理由（参考资料 9），也是脚本把 `nfc_on=1` 单独判为 `L1-INFO`、把"NFC 开启且 Secure NFC 为假"合并判为 `L2-WARN`（`nfc_lockscreen_emulation`）的原因：两个条件合起来才是"无人值守可被触碰"的窗口。

**判据四："研究口径"与"运营口径"不能混用。** 公告的严重度前提是缓解被关闭（参考资料 18），而设备上的实际暴露取决于缓解开启状态。第十七章的取证流程因此要求把"平台侧缓解状态"与"配置暴露状态"分别留存，而不是合成一个分数——一旦合成分数，就无法回答"这条公告对我这台设备是否真的可达"。

需要明确标注的边界：公开矩阵类资料（如攻击技术矩阵）里与无线相邻的技术条目，其平台列表并非 Android。本文在第八章对齐 CWE，但对攻击矩阵只做**措辞层参照**，不宣称任何 Android 专属映射；这一限制在第二十九章边界清单里单列。

---

## 七、缺陷类型学 W1–W10：按"哪一层被绕过"分类

类型学的目的是**给一条新公告找归属**，因此每一类都给出四件事：触发面所在的层、公开证据形态、修复责任人、设备侧可读信号。信号码全部来自脚本 1 或脚本 2（码表见第廿、廿二章），没有信号码的类型学只是词汇表。

| 编号 | 类型 | 主要落层 | 公开证据形态 | 修复责任人 | 设备侧信号（脚本 1） |
| --- | --- | --- | --- | --- | --- |
| W1 | 帧处理与分片/聚合实现缺陷 | L2/L3/L1 | 公告 + 研究页的 CVE 清单 | 内核驱动 / supplicant / 控制器固件 | 无直接信号；靠 SPL 与模块版本间接跟 |
| W2 | 密钥重安装与 nonce 复用 | L3 协议实现 | 密码学研究公开描述 | supplicant / 框架侧 Wi-Fi 模块 | `net_psk_only`、`net_tkip`（配置侧残余） |
| W3 | 握手降级与参数协商 | L3 协议设计 | 官方"过渡模式"定义句 | 网络设备 + 客户端配置策略 | `net_sae_present`（正向）、`net_psk_only`（负向） |
| W4 | 破损或风险加密配置 | L3/L6 配置 | 算法弃用口径 | 设备策略与配置管理 | `net_wep`(L3)、`net_tkip`(L2) |
| W5 | 无保护网络与自动加入 | L6 配置 | 平台网络选择策略 | 使用者与 MDM 策略 | `net_open`(L3)、`net_autojoin_open`(L3) |
| W6 | 配对与认证假设缺陷 | L3 蓝牙栈 + L1 控制器 | 通报机构与厂商公告 | 蓝牙栈与控制器双方 | `bt_legacy_pairing_marker`(L2)、`bt_secure_connections_marker`(L1) |
| W7 | 空口解析的内存安全缺陷 | L1 为主，L2/L3 亦有 | ASB 组件列 + 芯片公告 | 固件厂商 / 上游内核 | 无（基带）；Wi-Fi/BT 侧靠 SPL 跟 |
| W8 | 状态机与资源耗尽 | L1/L3 | 公告里的 DoS 类型条目 | 各层实现者 | `bt_radio_on`/`nfc_on`（暴露面存在性） |
| W9 | 凭据存储与历史绑定 | L3/L6 | 攻击者位置口径（已配对=本地） | 平台与用户双方 | `bt_bonded_present`(L1)、`bt_bond_marker_lines` |
| W10 | 暴露面与权限/策略配置 | L5/L6 | 权限与策略文档 | 应用开发者与 IT 策略 | `bt_discoverable`(L2)、`wifi_scan_always_on`(L2)、`nfc_lockscreen_emulation`(L2) |

四条使用规则。

**规则一：W1–W4 的区分线是"证据在哪一层"，不是"后果多重"。** 同一次公告里既可能改 supplicant 也可能改控制器固件；本文按**可核验的最小落点**归类，也就是"哪一层的产物变了"。这就是脚本 2 为什么把 `hal_iface_*`、`module_channel_*`、`baseband_version_moved` 三类证据分开输出——它们分别指向 HAL 层、框架模块层与固件层，合在一起就失去了归属能力。

**规则二：W5 是本文唯一"零漏洞即可利用"的一类，也是唯一可以完全在设备侧清零的一类。** `keyMgmt=NONE` 的已保存网络意味着同频段任何人都可以提供同名接入点；再加自动加入标记，设备会在无人确认时主动连接。脚本把它判为 `L3-HIGH`，与协议实现缺陷同级——因为从暴露时间尺度看，配置类问题往往存在数年，而实现类缺陷有明确修末日。

**规则三：W6 与 W9 必须成对读。** 官方口径把"已成功配对的设备"算作本地身份（参考资料 17），这既是缓解前提（需已配对），也是资产事实（绑定即凭据）。所以脚本对绑定同时输出两条：`bt_bonded_present`（存在绑定，L1，中性事实）与 `bt_legacy_pairing_marker`（既有绑定使用了旧配对方式，L2）。只报"有绑定"会诱导运维直接全部解除配对，破坏可用性；只报"配对方式"会漏掉历史凭据本身需要被盘点这一事实。

**规则四：W7 在设备侧没有信号，这本身就是结论。** 基带的解析面运行在独立处理器上，Android 侧的只读手段读不到它的内部状态，最多读到一个版本串。脚本因此把版本读不到判为 `L0-LOW`（`baseband_version_absent`），并在第廿一章样例二与样例三的对照里可以看到：同一个字段在一份档案里可读、在另一份里读不到，对应的是两种完全不同的治理动作——前者可以比对公告条目，后者只能向厂商索取证据。

---

## 八、类型学与 CWE 的对齐，以及不对齐的部分

对齐 CWE 的用途只有一个：让脚本输出能被扫描器、缺陷库与公告检索接住。本文的信号码**不直接引用 CWE 编号**，只给出对应关系，这样码位语义由本文自己维护，而 CWE 提供交叉索引。

| 类型 | 常用 CWE 条目（标准命名） | 对齐强度 | 说明 |
| --- | --- | --- | --- |
| W1 帧处理与分片/聚合 | CWE-20（Improper Input Validation）、CWE-130（Improper Handling of Length Parameter Inconsistency） | 强 | 长度字段与分片状态的处理是公开描述里最集中的落点（参考资料 33） |
| W2 密钥重安装与 nonce 复用 | CWE-327（Use of a Broken or Risky Cryptographic Algorithm）语境下常被引用，但更接近"协议状态机的 nonce 管理" | 中 | 本文不把 KRACK 类问题写成算法破损；这是**使用方式**缺陷（参考资料 34） |
| W3/W4 握手降级与破损配置 | CWE-326（Inadequate Encryption Strength）、CWE-327（Use of a Broken or Risky Cryptographic Algorithm） | 强 | WEP/TKIP 在公开口径下属"破损或风险配置"，与本文 `net_wep`(L3)/`net_tkip`(L2) 的分级一致；协商类缺陷本文不硬套单个编号（参考资料 34） |
| W5 无保护网络 | 无对应条目（配置事实，不是软件缺陷） | — | 这正是本文坚持用独立等级而不是硬套 CWE 的位置 |
| W6 配对与认证假设 | CWE-295（Improper Certificate Validation）语系、认证强度类 | 中 | Secure Connections 使用公钥认证路径，旧方式不提供同等保证（参考资料 34） |
| W7 空口解析内存缺陷 | CWE-125（Out-of-bounds Read）、CWE-787（Out-of-bounds Write）、CWE-119 语系 | 强 | 公开基带案例的描述句即属这一类（参考资料 33） |
| W8 状态机与资源耗尽 | CWE-400（Uncontrolled Resource Consumption） | 强 | 与 ASB 的 DoS 类型条目可直接对应（参考资料 35） |
| W9 凭据存储与历史绑定 | CWE-319（Cleartext Exposure of Sensitive Information）语系 | 中 | 绑定密钥的存储与使用路径跨 L3/L6，单一编号不覆盖 |
| W10 暴露面与策略 | CWE-693（Protection Mechanism Failure）作为兜底 | 弱 | 兜底编号在治理上没有派工价值，本文只用它做统计口径 |

三点必须写清的不对齐之处：

**第一，通用编号不替你做平台判定。** CWE-20、CWE-125 这类条目的定义句是通用软件措辞，不含"空口""配对""接入点"这些平台概念。把它们当成"Android 无线专属缺陷"会造成误读，因此表中"对齐强度"列显式区分强/中/弱。

**第二，无线侧最难归类的恰好是配置类。** `keyMgmt=NONE` + 自动加入不是缺陷，而是策略事实。任何以 CWE 为唯一词表的检查表都会在这一格失声——它要么被写成"配置错误"这种不可派工的措辞，要么被跳过。本文用 W5 + 两个 `L3-HIGH` 码位把它固定下来，就是为了在整改单上有东西可写。

**第三，缓解类条目的对齐要小心"保护机制失败"这种兜底编号。** CWE-693 覆盖面极大，用它对齐会制造"什么都是它"的假象。本文只在统计口径里使用（参考资料 35），不作为任何码位的定义依据。

需要说明的边界：本文不对齐 MITRE ATT&CK（含其移动视图）作为判定依据。公开技术矩阵里的无线相邻技术（网络嗅探、中间人、经蓝牙外泄等）其平台列表以桌面系统为主，对 Android 的适用性需要单独论证；本文不提供这一映射（第二十九章边界清单第 9 条）。

---

## 九、公开案例读数（一）：Wi-Fi 侧的三组

本章不复述攻击步骤，只把公开材料"读成"治理输入：每组给出官方描述句、类别归属、以及**你的设备侧能做什么**。

**KRACK（2017）。** 项目自述："Key Reinstallation Attacks: Forcing Nonce Reuse in WPA2"，并给出位置前提："An attacker within range of a victim can exploit these weaknesses using key reinstallation attacks (KRACKs)."（参考资料 22）。归属：W2（协议实现中的 nonce 管理），落层 L3 supplicant 为主。治理读数：这一类的价值不在于"当年修了什么"，而在于它把**"加密已建立"与"密钥使用正确"分开了**——设备侧只读观察能确认网络使用了 SAE/EAP，但**永远无法**确认握手实现是否正确。这条限制在第十七章被写成一条明确分工：配置类归脚本，实现类归公告与模块版本。

**Dragonblood（2019）。** 研究页对 SAE 侧问题的公开描述包含一句关键结论：在特定条件下"an attacker within range of a victim can still recover the password."（参考资料 23）。归属：W3（握手与降级），落层 L3 协议 + 设备侧配置。治理读数：它给出的可执行动作是**把"只用 SAE"当作目标状态**，而不是"支持 SAE"——脚本因此对"存在 PSK 且本次未见 SAE"判 `net_psk_only`(L2)，对"有 SAE 记录"判 `net_sae_present`(L1)，两者一起构成整改方向：减少 PSK 存量，而不是升级即宣告完成。同时要注意第四章引到的官方过渡模式句：接入点同时支持 WPA2/WPA3 时，老设备仍走 PSK——所以**接入点侧的过渡模式配置必须一起治理**，只清点手机端会得出乐观结论。

**FragAttacks（2021）。** 站点自述："This website presents FragAttacks … which is a collection of new security vulnerabilities that affect Wi-Fi devices."，并给出两个量化结论："Every Wi-Fi product is affected by at least one vulnerability and that most products are affected by several vulnerabilities."、"An adversary that is within range of a victim's Wi-Fi network can abuse these vulnerabilities to steal user information or attack devices."（参考资料 21）。归属：W1（帧处理与分片/聚合），落层横跨 L2/L3/L1。治理读数有三条：

1. **跨层缺陷无法用单层证据关闭。** 分片与聚合的处理同时涉及控制器固件与上层实现，因此"某个模块更新了"不等于这一类结束；第十四章的归属矩阵就是为这类跨层条目准备的。
2. **认证机构口径与平台口径是两套语言。** 认证机构在通告里把这一问题描述为帧聚合功能上的漏洞集合，并给出使用侧动作（安装厂商推荐更新）。这句话的价值是：它把"责任在厂商更新"讲得很清楚，但也说明**认证类通告不含到达性证据**——设备侧仍然要用 SPL、模块版本、`baseband_version_read` 自行确认。
3. **本文档站在这一类缺陷上的位置是可观察面的上界。** 内核 cfg80211/nl80211 提供的"接收全部帧"与管理帧收发能力（第四章）是网络诊断与研究的正规接口，也正是这类问题被发现的观察面。对防御侧的含义是：**同一观察面在设备上是否可达，受 SELinux 与权限约束**——这也是本文不在设备上做任何主动空口采集的原因。

---

## 十、公开案例读数（二）：蓝牙侧的四组

**BlueBorne（2017）。** 公开命名口径把它描述为"BlueBorne is an attack vector by which hackers can leverage Bluetooth connections to penetrate and take complete control over targeted devices."（参考资料 29，取自存档快照；原页面当前不可直接抓取），并给出两个位置前提：设备**无需处于已配对状态**、也**无需处于可发现模式**。归属：W7 与 W8 的混合（协议栈实现与状态机），落层 L3 主机栈为主。治理读数：这一组最重要的贡献是"可发现模式不是前提"——所以脚本把 `bt_discoverable` 判为 `L2-WARN`（扩大配对类风险的窗口），但**不把"不可发现"当成"不可达"**：`bt_radio_on` 才是暴露存在性的信号，样例一里两者同时出现就是为了让读者看到这两个事实要一起读。

**BLEEDINGBIT（2019）。** 研究方公开页面的自述两句："Armis has identified two chip-level vulnerabilities impacting access points and potentially other unmanaged devices."，以及"These proximity-based vulnerabilities allow an unauthenticated attacker to break into enterprise networks undetected."（出处登记见参考资料 30，正文两句取自参考资料 31 的存档快照）。归属：W7，落层 **L1 控制器固件**——但**注意受影响对象**：这一组描述的是 BLE 芯片在无线路由器与未纳管设备上的固件问题，不是 Android 主机栈的缺陷，把它读成"手机蓝牙漏洞"属于误用。本文保留它的理由有两层：一是它给出了 L1 控制器档的公开形态（芯片级、邻近可达、未认证、不可检测，与第二章"邻近即远程"的口径完全对齐），二是它演示了第十三章那种可见性——芯片固件是否已修，在被攻击设备所在的 Android 侧**没有任何可只读观察的状态**。这正是脚本 1 把 `baseband_version_absent` / `bootloader_version_absent` 判为缺口而不是通过的原因（同样的逻辑适用于蓝牙控制器固件）。

**BIAS（2020）。** 通报机构描述句："It is possible for an unauthenticated, adjacent attacker to impersonate a previously paired/bonded device and successfully authenticate without knowing the link key."（参考资料 24）。归属：W6（配对与认证假设），落层 L3 协议 + L1 控制器（配对能力协商在控制器侧）。治理读数：这一组直接支撑第九章与 W9 的判断方式——**既有绑定不是可信度的来源，而是需要盘点的资产**。脚本对绑定输出的两条码位（数量 + 配对强度记号）就是按这条案例的公开结论设计的；样例二里 `bt_legacy_pairing_marker` 与 `bt_discoverable` 同时出现，代表的是"历史绑定用旧方式 + 当前可被陌生端发起"这个组合，比任何单条都更值得先处理。

**KNB（Key Negotiation of Bluetooth，2020）。** 公开记录的核心是一条关于密钥协商强度的缺陷编号（CVE-2019-9506；本文未取得可逐字引用的整段描述，参考资料 32 标注为截断）。归属：W6/W4 之间（协商出的密钥强度不足）。治理读数：本文特意保留这一组"未取得整句"的案例，因为它演示了系列统一的纪律——**证据不足时不判级**。脚本层面对应的是 `bt_pairing_strength_unread`：读不到配对强度记号时不判定既有绑定用的是哪种方式，而不是默认它安全。

---

## 十一、公开案例读数（三）：基带侧

**Google 基带加固口径（2023）。** 平台侧对基带的公开表述给了两条本文反复使用的前提（参考资料 26）："Android's defense-in-depth strategy applies not only to the Android OS running on the Application Processor (AP) but also the firmware that runs on devices."，以及"We particularly prioritize hardening the cellular baseband given its unique combination of running in an elevated privilege and parsing untrusted inputs that are remotely delivered into the device."。同页还给出两条方法性表述："They are architecture agnostic, suitable for bare-metal deployment..."，以及关于编译器 sanitizer 只能覆盖部分缺陷类别、长期方向是转向内存安全语言的说明（"Other classes of memory safety vulnerabilities remain unaddressed by these sanitizers. A longer term solution is to begin transitioning today to memory-safe languages such as Rust."）。

读数：**"高特权 + 解析远程不可信输入"这两条同时成立**，是本文把基带单列一档的全部理由；而"architecture agnostic / bare-metal"说明加固手段是**编译期与运行期**的，不是权限模型的，因此 Android 侧的域与标签对它不构成约束。本文不复制加固实现细节，也不宣称任何厂商的具体开关状态。

**Project Zero Exynos 系列（2023）。** 公开表述："In late 2022 and early 2023, Project Zero reported eighteen 0-day vulnerabilities in Exynos Modems produced by Samsung Semiconductor."，并说明其中若干条可造成网络到基带的远程代码执行（参考资料 27）。治理读数三条：

1. **一条从公网到基带的通路，其"远程"属性在口径上与第六章一致**，且不需要在设备上安装任何应用；
2. 这类缺陷的修复以固件包形态发布，设备侧唯一可用信号是版本字符串；
3. 研究者公开发布时间线与厂商补丁节奏之间的差距，正是[补丁传导链篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)处理的问题，本文不重复展开，只在指标 K8（公告核对闭环时长，按层分档）上引用其口径。

**芯片厂商公告形态。** ASB 对芯片侧条目有固定措辞："…affect Qualcomm closed-source components"，并在严重度上注明"The severity assessment of this issue is provided directly by Qualcomm."；对 MediaTek 侧则是"The severity assessment of these issues is provided directly by MediaTek."（参考资料 18）。厂商自身也发布独立公告（参考资料 28）。治理读数：**严重度的判定权在芯片厂商手里，而平台公告只是转述位**——这就是为什么本文不允许用"ASB 里没有某条基带 CVE"推断"该缺陷不适用于本设备"；这类推断的正确形态是缺口记录（`baseband_evidence_unread`），不是通过。

**公告编号形态的实务提醒。** ASB 条目使用多套引用前缀（A-/M-/Q-/N- 等），其中芯片厂商引用号与 CVE 并排出现。设备侧运维在提工单时应同时带上：机型、SPL、`gsm.version.baseband` 值、厂商引用号。前四项本文脚本全部输出（`META,*` 行），第五项需要公告——这构成了第十七章"取证数据包"的最小字段集。

---

## 十二、基带为什么单列一档

前十一章把 Wi-Fi、蓝牙、NFC 与基带放在一起清点，是因为它们共享"空口"这个物理入口；本章开始把它们分开，是因为**四样属性完全不同**：缺陷住在哪里、由谁修、通过哪条通道到达设备、以及这一切在设备侧能被观察到多少。第三章给出的第一条硬边界就在这里：**蜂窝调制解调器不是 Android 里的一个进程**，它不在 ActivityManager 的进程表里，没有 UID，没有 `seapp_contexts` 条目，也不受本文讨论的运行时权限约束。第十一章引到的两句公开口径构成了这一档的全部理由——"Android's defense-in-depth strategy applies not only to the Android OS running on the Application Processor (AP) but also the firmware that runs on devices."，以及把两个条件并置的"We particularly prioritize hardening the cellular baseband given its unique combination of running in an elevated privilege and parsing untrusted inputs that are remotely delivered into the device."（参考资料 26）。

**四条判据把基带与其余三层分开：**

- **判据一：执行位置在另一颗处理器上。** 基带固件运行在调制解调器子系统，不在 AP 上。因此[应用沙箱篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)讨论的 UID/SELinux 域/seccomp 三层叠加**对基带不构成约束**——那三层是给 AP 上的进程准备的。同理，[内核漏洞与提权篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)里的内核缺陷仍在 AP 侧，其后果可以被域与系统调用面限制住；基带缺陷的"后果边界"由固件自身的地址空间决定，Android 侧没有一个可配置的等价物。
- **判据二：触发路径不经过 L2–L5。** Wi-Fi 侧的多数缺陷要经过 supplicant（L3）或 HAL（L4）的解析才落到有缺陷的代码；基带的入口就是空口到调制解调器固件本身。这意味着**平台侧的任何配置收紧都不在触发路径上**：把 SELinux 调严、把权限收回、把 Mainline 模块升到最新，都不会让一条基带缺陷变成不可达。这是本文拒绝"用配置姿态推断基带风险"的根本原因。
- **判据三：修复以固件包形态到达。** 与 `com.android.wifi` 可以走模块更新（参考资料 12）不同，基带固件随厂商固件包分发，公开口径把它的来源限定在整机镜像与专用固件通道上。这正是[补丁传导链篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)处理的传导问题在无线侧最严重的一处：**这一层的修复速度不由平台决定，也不由设备管理员决定**。
- **判据四：严重度的判定权在芯片厂商手里。** ASB 对芯片侧条度的两句措辞在第十一章已经给出（"The severity assessment of this issue is provided directly by Qualcomm." / "…provided directly by MediaTek."，参考资料 18、28）。平台公告是转述位而不是判定位，因此"公告里没有某条基带 CVE"这类推理在方法上不成立。

**由此得出的治理形态差别。** 对 Wi-Fi 与蓝牙主机栈，可以做的是"清点配置存量 + 核对模块版本 + 跟踪公告"，三件事都能在设备上留下证据；对基带，能做的只有后两件，而且都不在设备上：

1. **台账先于检测**：机型、SPL、`gsm.version.baseband`、`ro.boot.bootloader` 四个字段构成基带治理的最小可用记录（本文脚本 1 的 `META,*` 行输出前三项的全部可得值，第四项同理）。没有台账，公告来了也无法回答"哪些设备受影响"。
2. **公告核对按机型而不是按 SPL**：芯片侧条目的适用性由固件版本决定，而 SPL 只反映 Android 侧安全补丁的到达时间，两者不同步；这一点与[平台代码组织与构建体系篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)对构建身份的口径一致。
3. **暴露面收敛是企业侧唯一能立即做的动作**：对不需要蜂窝能力的设备形态（平板、固定终端、车载后装）取消蜂窝能力，属于设备管理策略而不是加固配置；本文只把它记为"策略位存在"这一事实，不展开任何策略下发步骤。
4. **不要把"没有信号"写成"没有风险"**：这是本文脚本对基带侧只做版本采集、不做风险判定的全部原因。第十三章把这条限制写成可见性边界。

**一句话的档位结论**：Wi-Fi、蓝牙、NFC 三层的风险可以"观察—判级—整改—复核"闭环；基带这一层只能"记录—核对—等待—再记录"。把两者放进同一张检查表并要求同一个 SLA，是无线治理里最常见的失效模式（第二十六章误区 1 与之对应）。

---

## 十三、基带可见性与归属困难：三种"看不见"

这一章写的是**限制**，不是方法。它决定第十七章取证流程的产出物形态，也决定第十八章判定模型为什么允许 `L0-LOW` 单独成为一类而不是并入"通过"。

**第一种看不见：源码与符号不可见。** 芯片侧组件在 AOSP 之外，公开公告的措辞本身就是这一点的证据——"…affect Qualcomm closed-source components"（参考资料 18）。读者无法像核对 `cfg80211`（参考资料 1 的接口约定句）或 sepolicy 条目（参考资料 19、20 可逐行读到）那样，从公开代码里确认某段解析逻辑是否存在于自己的设备上。这带来一个具体后果：**本文不允许任何"从代码推断可达性"的结论出现在基带档**，所有基带判断必须落到厂商条目上。

**第二种看不见：到达性不可见。** 设备侧能读到的只有版本串，而版本串与公告条目之间的对应关系由厂商维护。实务中有三种常见的断点：公告页下线或改版、条目被合并进"多个问题"的整段描述、以及同一 CVE 在平台公告与厂商公告里以不同引用号出现。因此本文要求记录**取证证据本身的抓取形态**（URL、抓取时间、页面快照），而不只是记录一个结论；这与第二章"公告里的严重度不是你的暴露度"是同一纪律的两个方向。

**第三种看不见：判定权分离。** 严重度由芯片厂商提供（参考资料 18），补丁由厂商装配，公告由平台转述。三方各自保留裁量，结果是**同一缺陷在不同评分体系里可能相差两档**。本章把文中真实存在的对照写清楚，因为它直接决定工单能不能立：

| 出处 | 对同一族缺陷的口径 | 读法 |
| --- | --- | --- |
| 平台公告（参考资料 18） | 蓝牙侧高危条目使用"could enable a remote attacker using a specially crafted transmission to execute arbitrary code within the context of a privileged process"这一远程代码执行措辞 | 严重度前提是缓解被关闭（第二章判据四） |
| 通报机构（参考资料 24） | "It is possible for an unauthenticated, adjacent attacker to impersonate a previously paired/bonded device and successfully authenticate without knowing the link key." | 关注"未认证 + 邻近"这两个可达条件本身 |
| 通用漏洞库（参考资料 25） | "Legacy pairing and secure-connections pairing authentication in Bluetooth BR/EDR Core Specification v5.2 and earlier may allow an unauthenticated user to complete authentication without pairing credentials via adjacent access."，并给出一个中等级别的评分（该记录的基础严重度字段为 `MEDIUM`） | 归属落在**规范版本**（v5.2 及更早）而不是某个实现 |

**第三行的信息量最大**：它说明这一族缺陷的修复对象是蓝牙核心规范的认证设计，因此"升级 Android 侧的栈"不必然等于"修复"，还要看控制器与规范版本；同一族缺陷在通用库里的评分与平台公告的严重度口径**相差两档**。这两点合起来就是第十四章归属矩阵必须存在的原因：**缺陷编号解决"是什么"，归属矩阵解决"谁的活、走哪条通道、我拿什么证明"**。

**同样的可见性逻辑适用于蓝牙控制器固件。** BLEEDINGBIT 一族落在 BLE 控制器内部（第十一章），Android 侧既没有配置项也没有可只读观察的状态能反映它是否被修——所以本文把它与基带放在同一类处置里：只采版本、不判风险。脚本对这两个字段的缺失判定分别是 `baseband_version_absent` 与 `bootloader_version_absent`，两者都是 `L0-LOW`，且都写明"不判定"而不是"无风险"。

**结论：基带档的产出物是证据包，不是结论。** 一个可交接的最小字段集（第十七章按此设计）：机型、构建形态、SPL、基带版本串、引导版本串、对应公告条目引用号、公告页抓取时间与快照、以及本设备当时四项射频状态位。缺任何一项，这份记录都不能用来结单——也不能用来宣告"安全"。

---

## 十四、归属矩阵：层 × 修复责任人 × 更新通道 × 设备侧证据

本章是全文的调度表。它的唯一用途是：给定一条缺陷或一个信号码，回答"该找谁、走哪条通道、多久能到、我拿什么自证"。表中"设备侧证据"列的每个记号都对应脚本 1 的输出行（记号语义见第二十章）。

| 层 | 典型缺陷族 | 修复责任人 | 更新通道 | 设备侧证据（本文脚本） | 证据不足时的替代证据 |
| --- | --- | --- | --- | --- | --- |
| **L1 芯片固件**（Wi-Fi/BLE 控制器、基带调制解调器、NFC 控制器与安全元件） | W7（控制器实现）、W9（密钥与协商强度）、基带内存安全 | 芯片厂商；安全元件侧另见[硬件安全组件篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Hardware-Security-Components-and-Vendor-Implementations.html) | 厂商固件包，随整机镜像；不进入 APEX | `META,baseband_prop` / `META,bootloader_version_prop`；`baseband_version_absent`、`bootloader_version_absent`（均为缺口） | 厂商公告条目 + 版本对应表；本文不接受任何设备侧推断 |
| **L1 规范层**（蓝牙核心规范、802.11 修正案、WPA3 认证计划） | W6（配对与认证假设）、W3（握手降级） | 标准组织与认证计划；实现方各自跟进 | 规范版本更替 + 实现更新，节奏不由 Android 决定 | 不可直接观察；只能通过 `wifi_networks_*` 计数间接反映配置存量 | 参考资料 25 那种把缺陷归属到具体规范版本的记录 |
| **L2 内核无线子系统**（`cfg80211`/`mac80211`/`nl80211`） | W1（解析与边界检查）、W2（状态与密钥使用） | 上游内核 + AOSP + 厂商驱动三者交界，接口约定见参考资料 1 | 通用内核与厂商模块，随整机镜像；设备侧不暴露模块版本 | 无（本文脚本不进入内核侧采集，避免与[内核结构与子系统篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)重复） | SPL、内核版本串、GKI/厂商模块公告 |
| **L3 用户态协议栈**（Wi-Fi supplicant/hostapd、蓝牙主机栈、NFC 服务） | W2、W5（ fragmented/聚合处理）、W8（状态机） | 平台侧；蓝牙进程与 Wi-Fi 服务身份见参考资料 2、5、19 | **分叉点**：Wi-Fi 侧可走 Mainline 模块 `com.android.wifi`（参考资料 12）；蓝牙模块是 optional 模块（参考资料 13） | `MOD,wifi` / `MOD,bluetooth`；`wifi_module_*`、`bt_module_*`；`dumpsys` 三侧字节数 | 公告条目对框架/模块的归属标注；模块版本（本文脚本不读版本号，只读挂载形态） |
| **L3 HCI 与取证通道** | 不属于缺陷族，属于治理面：抓包数据本身 | 平台 + 设备管理员 | 随平台 | `SET` 与 `META,wifi_dump_bytes` 等采集量记号；数据落点与标签见参考资料 4、20 | 受管设备的日志策略；本文不指导抓取 |
| **L4 三个 HAL 面**（Vendor / Supplicant / Hostapd，参考资料 5）+ 蓝牙 HAL | W4（接口实现不一致）、W10（跨层配置错配） | vendor 实现方，接口定义归平台 | 随 vendor 镜像；接口语言决定能否与系统侧解耦（参考资料 6） | `ROW,wifi_hal_iface_aidl` / `wifi_hal_iface_hidl` / `wifi_hal_iface_unknown` | 接口语言记号读不到时按缺口处理，不推断回退 |
| **L5 Framework 与权限** | 越权调用、扫描结果隐私语义 | 平台 | Framework 更新 + Mainline | 本文脚本不采集应用权限；能力面存在性来自 `FEAT,*`（参考资料 12、14、16） | [Framework 信任边界篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)与[Binder 篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html) |
| **L6 设备与企业策略** | 不属于缺陷族，属于暴露面调节 | 设备所有者与企业设备管理员 | 策略下发，即时生效 | `SET,*` 全部行；`wifi_scan_always_on`、`nfc_secure_disabled`、`nfc_lockscreen_emulation` | 管理台策略记录（脚本 2 的差分可与变更单对齐） |

**三条使用纪律：**

1. **一行现象可能同时命中多行责任。** 一条"Wi-Fi 握手降级"可能同时需要 L3 supplicant 修复、L4 Supplicant HAL 更新与 L1 控制器固件重签；矩阵的正确用法是**列出全部命中行并分别派工**，而不是选一行结案。这也是脚本 1 把 `net_psk_only`（配置存量）与 `wifi_module_*`（更新通道）分行输出的原因——它们在整改单上是两个不同的责任人。
2. **更新通道决定 SLA。** L6 是分钟级，L3 的 Mainline 侧是"取决于模块是否可选与是否挂载"（参考资料 13 明确蓝牙模块为 optional），L1/L4 只能等厂商固件包。同一个高危条目在 L3 与 L1 上的合理 SLA 不是同一个数；第二十五章的 K8 指标（同一公告按 L3 与 L1 分档统计核对闭环时长）按这个差别设计。
3. **设备侧证据弱不等于风险低，也不等于风险高。** 表里最后两列的存在就是为这条服务的：证据不足时的替代证据必须是**外部记录**（公告、版本对应表、管理台策略），而不能是"读不到所以判通过"。脚本 2 的 `risk_gone_by_coverage` 与 `risk_closed_needs_proof` 这对码位是这条纪律在差分层的实现。

**这张表怎么读成结论。** 把它当成一次工单派发的输入：信号码 → 行 → 责任人 → 通道 → 需要补的外部证据。**它不是风险评分表**，不给出优先级数字；优先级由第十七章的取证数据包 + 第十八章的四级判定模型给出，避免把"归属"与"严重度"混成一个不可审计的分数。

---

## 十五、平台侧收敛机制：四条，其中只有两条可观察

这一章回答一个常被误答的问题：**平台到底用什么手段把无线侧的风险收敛住**。答案是四条机制，而本文脚本只能观察到其中两条；剩下的两条只能观察到"配置姿态"，观察不到"机制是否在位"。这个上限必须在报告里写清楚，否则读者会把快照读成加固验收单。

**机制一：Mainline 模块通道（可观察到挂载形态）。** "Android 10 introduced Mainline (formerly called modular system components on this site)."，Wi-Fi 侧以 `com.android.wifi` 形态存在（参考资料 12）；蓝牙侧限定更多："The Bluetooth module is an optional Mainline module introduced in Android 13. In Android 16 and higher, the module is updatable…"（原文截断，参考资料 13）。脚本 1 用 `toybox ls /apex` 判断挂载形态（`MOD,wifi` / `MOD,bluetooth`），并严格区分三态：`mounted` / `absent` / `unknown`。**"absent 只说明通道未知"**——对可选模块尤其是这样，把缺席读成"未加固"是第二十六章误区 5。脚本 2 侧对应 `module_channel_lost`(L2) / `module_channel_added`(L1) / `module_channel_unread`(L0)。这条机制给治理带来的真实变化是：**框架侧修复可以不等整机 OTA**，因此第廿五章的 K3 指标单独统计"有独立更新通道的子系统占比"。

**机制二：接口语言迁移 HIDL→AIDL（可从输出记号观察）。** "…all three interfaces are available as stable Android Interface Definition Language (AIDL) packages. These interfaces were previously defined using Hardware Interface Definition Language (HIDL)."（参考资料 6）。这一句的意义在第十四章已经用过：接口稳定性决定系统侧与 vendor 侧能否解耦更新，因而决定一条修复能不能拆开走。脚本把 `dumpsys wifi` 里的记号读成 `wifi_hal_iface_aidl` / `wifi_hal_iface_hidl` / `wifi_hal_iface_unknown`，把 HIDL 判为 `L2-WARN`（向被公开口径标注为弃用的接口语言回退），把读不到判为 `L0-LOW`。脚本 2 里这条记号是少数能做"方向判定"的字段之一（`hal_iface_regressed` / `hal_iface_modernized` / `hal_iface_lost` / `hal_iface_first_read`），因为它是离散状态而不是数量。

**机制三：SELinux 身份与标签（本文不观察）。** 无线侧身份映射可以逐行读到："user=bluetooth seinfo=bluetooth domain=bluetooth type=bluetooth_data_file"，另有 nfc 与 radio 的对应条目（参考资料 19）；取证数据另有独立标签 `bluetooth_logs_data_file`（参考资料 20）。但**这些不是本文脚本的采集面**：域与标签的核验方法、以及"身份与域不一致才是高信号"这条判据在[应用沙箱篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)里已经完整展开，本文不重复实现一遍采集脚本。这一章保留它的理由是**归属**：L3/L4 进程的策略归属决定了 vendor 侧修复的装配范围，而这一决定不由本文能观察到的任何字段给出。

**机制四：运行时权限与隐私语义（可观察存在性，不可观察强度）。** "The `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, and `BLUETOOTH_SCAN` permissions are runtime permissions."，扫描结果的过滤规则有整句"If you include `neverForLocation` in your `android:usesPermissionFlags`, some BLE **beacons** are filtered from the scan results."（参考资料 14）；Wi-Fi 扫描结果侧另有"An app must hold ACCESS_FINE_LOCATION … and ACCESS_WIFI_STATE permission in order to get valid results."（原文截断，参考资料 16）。**这条机制收敛的是"设备上的应用能做什么"，不是"空口能不能打进来"**——第五章那句"权限收紧不等于通道关闭"就是为它写的。因此本文脚本不采集应用权限：采集它需要包级清点，那是[应用安全弱点分类篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)的范围。

**四条机制合起来给出的判定上限：**

| 机制 | 本文能否只读观察 | 观察到的到底是什么 |
| --- | --- | --- |
| Mainline 通道 | 部分（挂载形态） | 修复**有可能**不等整机 OTA；不是"已修" |
| AIDL/HIDL | 部分（输出记号） | 接口形态；不是缺陷状态 |
| SELinux 域与标签 | 否（本文不采集） | 需要另一份采集，见应用沙箱篇 |
| 运行时权限 | 否（本文不采集） | 观察到的是配置姿态，不是加固强度 |

这张表是第十八章"四条纪律"里"未采到即缺口"这一条存在的理由：判定模型的输入只覆盖四条机制中的两条的一部分，所以模型的输出**必须保留一个专门表达输入不足的等级**，而不是把它折算成低分。

---

## 十六、设备与企业策略加固：能收紧的六处与不能指望的三处

本章面向设备管理员与企业侧，只讨论**公开口径下存在的策略位与它的可观察性**，不给出策略下发步骤，更不讨论任何策略的绕过方式（范围声明已经划界）。

**六处可以收紧、且收紧效果能在快照里看到的策略位：**

- **关网扫描**：`secure/wifi_scan_always_available`。Wi-Fi 关闭时仍允许扫描是公开口径下可被定位侧利用的通道，脚本判 `wifi_scan_always_on`(L2)；翻回 0 时判 `setting_scan_restricted`(L1) 并明确记为改进项。这是六处里唯一"设备侧可直接看到状态位翻转"的策略。
- **可发现性窗口**：脚本读到可发现/可连接记号时判 `bt_discoverable`(L2)，读到字段但为关闭形态时判 `bt_discoverable_off`(L1)，读不到时判 `bt_discoverable_unread`(L0)。**注意第六章那条判据**：不可发现不等于不可达，这一位的收紧只是缩小 W6 类风险的窗口，不是消除暴露存在性（`bt_radio_on` 才是）。
- **Secure NFC**：定义句已在第四章给出——"Secure NFC is a feature introduced in Android 10 that allows off-host card emulation to be enabled only when the device's screen is unlocked."，前置条件要求控制器支持 NCI 2.0（参考资料 9）。脚本把"字段存在且为假"判 `nfc_secure_disabled`(L2)，并在 NFC 同时开启时追加 `nfc_lockscreen_emulation`(L2)，说明句写得很直白：**两个条件合起来才是可被无人值守触碰的窗口，单看任一行都会低估**。
- **蓝牙能力位与设置位分列**：`DISALLOW_BLUETOOTH` 与 `DISALLOW_CONFIG_BLUETOOTH` 是两个不同的策略位，后者一句关键限定是"**This does not restrict the user from turning bluetooth on or off.**"（参考资料 15）。这一区别在治理上很容易踩：只设了"不能改设置"的设备，射频开关仍在用户手里，快照里的 `bluetooth_on` 值与策略名并不匹配。本文因此把 `bluetooth_on` 作为独立事实输出，不用策略名推断它。
- **蜂窝能力是否存在**：`FEAT,android.hardware.telephony`（参考资料 12 的特性面口径）。不需要蜂窝能力的设备形态取消该能力属于设备选型而不是加固；脚本把它记为 `cell_feature_absent`(L1)，是"面不存在"这一类事实，不是风险下降。
- **Passpoint 侧的网络发现**：802.11u 的发现与选择由 supplicant 提供（参考资料 8），企业侧一旦启用，发现阶段的解析面进入厂商责任区（`wifi_passpoint_present` 是中性事实记录）。**收紧它的正确动作是收敛配置存量**，与脚本 `wifi_networks_*` 系列计数配合使用。

**三处不能指望的地方，写在这里是为了避免策略被误读成加固：**

1. **不能指望策略收紧覆盖基带档。** 第十二章判据二：触发路径不经过 L5/L6，任何设备管理策略都不在链路上。
2. **不能指望"关闭射频"清理历史凭据。** 用户关闭蓝牙停止射频但**不删除既有绑定**，而绑定在公开口径里算本地身份（参考资料 17）。第二十一章样例一那句"历史绑定不会随关闭射频消失"就是这一条；`bt_bonded_present` 因此始终配一句"需要资产化清点"的说明。
3. **不能指望单次快照证明收紧生效。** 收紧是"某一位从 1 变 0"这一事件，只有基线差分能表达（脚本 2 的 `SETDIFF` 行 + `setting_scan_restricted` / `setting_radio_flip` 码位）；单次快照只能证明"此刻的值"。

**五组可执行动作（不涉及具体下发方式）：**

1. **绑定与已配置网络按资产登记**，而不是按配置项删除：数量、配对强度记号、密钥管理形态三项都要有留存，`COUNT,*` 行可直接作为登记输入。
2. **快照留存周期化**：至少"入网基线 / 每次大额变更后 / 每季度"三类；脚本 2 的输入是两份纯文本，可以在没有设备的情况下复核，这是它被设计成离线比对的原因。
3. **变更单与差分行对齐**：`DELTA,*`、`SETDIFF,*`、`FEATDIFF,*`、`MODDIFF,*` 四类行的键名可以直接写进变更单，避免"整改完成"只有口头依据——对应 `risk_closed_needs_proof` 那条纪律。
4. **缺口走上报路径而不是豁免**：`L0-LOW` 行不是噪声，第二十五章 K9 把缺口率作为独立指标，K10 把"同一缺口连续两次未闭合"（`gap_persistent`）作为流程告警。
5. **工单最小字段集固定**（第十三章表末）：机型、SPL、基带版本、引导版本、厂商引用号、公告页抓取时间与快照。这六个字段构成基带档唯一可交接的证据。

**边界重申**：本章所有动作都在"清点、留存、对齐、上报"这四个只读动作范围内；本文不提供绕过设备管理策略的方法，不提供关闭某项安全特性以降低风险的操作步骤，也不提供任何空口报文构造。判定与取证是本文的全部交付形态。

---

## 十七、检测与取证：可检出对象、证据包，与"配置归脚本、实现归公告"的分工

无线侧的检测有一个结构性限制：这一面的缺陷大多在**协议实现内部**，而实现内部状态在设备侧不可读。因此本文的检测设计不试图"发现缺陷"，只试图**发现缺陷赖以生效的配置条件**，并把无法从设备侧发现的部分明确交给外部记录。这一章给出这条分工的具体形态。

**分工规则（全文最重要的一条操作边界）：**

| 判断对象 | 归属 | 依据 |
| --- | --- | --- |
| 网络与射频的配置存量（密钥管理形态、自动加入、可发现性、扫描通道、Secure NFC） | **设备侧脚本 1** | 这些状态在 `dumpsys` 与 `settings` 里可读；样例见第廿一章 |
| 更新通道形态（模块是否挂载、接口语言） | **设备侧脚本 1** | `MOD,*` 与 `wifi_hal_iface_*`；只证明通道，不证明已修 |
| 状态是否被改动过 | **脚本 2（离线差分）** | `SETDIFF` / `DELTA` / `MODDIFF` / `FEATDIFF`；单次快照证明不了方向 |
| 协议实现是否正确（握手、nonce、分片与聚合处理、状态机） | **公告 + 版本记录**，不在设备上 | 第九章 KRACK 读数给出的限制："加密已建立"与"密钥使用正确"是两件事 |
| 控制器与基带固件是否已修 | **厂商公告 + 版本对应表** | 第十三章三种"看不见"；设备侧只有版本串 |
| 应用侧权限与代码弱点 | 另一篇的范围 | [应用安全弱点分类篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html) |

**这张表的用途是拒绝越权结论。** 它把"脚本能说的话"限制在配置面与通道面，凡是读表后发现某项不属于自己的行，就不允许在该项上给出结论——第二十六章的误区 3、4、6、7 全部是这张表被违反的具体形态。

**取证数据的三条公开约束（无线侧特有，容易踩）：**

1. **HCI 日志是被单独管控的一类数据。** 平台侧记录其落点为"stored in `data/misc/bluetooth/logs`"（参考资料 4），SELinux 侧有独立对象标签 `/data/misc/bluetooth/logs(/.*)? u:object_r:bluetooth_logs_data_file:s0`（参考资料 20）。同页还有隐私口径"For privacy reasons, always-on, … BTSnoop only logs non-personal information and events."（原文截断，参考资料 4）。**含义**：在受管设备上"抓一份 HCI 日志"不是随手动作，它需要策略允许与标签可达；本文脚本不读取该目录，也不复制任何日志内容——脚本 1 对蓝牙侧只采 `dumpsys bluetooth_manager` 的记号存在性，不采报文。
2. **采集量本身是一个信号。** 脚本输出 `META,wifi_dump_bytes` 等三行采集字节数，用途不是"证明采到了"，而是区分"输出为空"与"输出被截断/被策略限制"。样例三里 `dumpsys wifi` 有内容但一行 `nid=` 都解析不出，就是靠这两个数字与 `wifi_dump_unparsed` 码位组合才读成解析缺口而不是"没有配置网络"。
3. **快照里不落敏感标识。** 本文两个脚本都**不打印 SSID、不打印蓝牙设备地址、不落任何凭据**——脚本 1 的输出只有计数、形态记号与设备版本元数据。这是设计约束而不是取舍：一份要进工单系统、要长期留存的快照，其内容必须可以安全流转。这也决定了脚本 2 的差分主键只能是记号与计数，不能是网络名。

**取证数据包的最小字段集（与第十三章末一致，本章给出交接形态）：**

- 设备侧：机型、制造商、SDK、`ro.build.type`、SPL、基带版本串、引导版本串、四个 `SET` 状态位、三个子系统 `FEAT` 值、`MOD` 挂载形态、脚本 1 全量输出文本（含 `ROW`/`COUNT`/`META` 行）。
- 外部侧：命中的公告条目引用号（含芯片厂商引用号与 CVE）、公告页 URL 与抓取时间、厂商公告页快照（下线风险见第十三章）、模块或固件包的版本对应关系。
- 流程侧：采集人与采集时间、基线快照路径与当前快照路径（脚本 2 的两个输入）、本次未闭合的 `L0-LOW` 项清单。

三组字段缺任一组时的正确写法：**外部侧缺失 → 该项只能记为"待厂商确认"；流程侧缺失 → 差分结论不可用于结单；设备侧缺失 → 整份证据包降级为"部分采集"，不得判"无风险"。**

**时间线纪律。** 无线事件的取证时间线有三个不同时刻，混用会得出错误结论：配置变更时刻（`SETDIFF` 可见）、模块或固件更新时刻（`MOD`/`MODDIFF` 与版本串变化可见）、公告发布时刻（外部记录）。三者的关系是**公告时刻早于或等于修复到达时刻，到达时刻早于或等于配置存量下降时刻**；脚本 2 之所以把 `spl_moved`、`baseband_version_moved`、`module_channel_added` 判成 `L1-INFO` 的"时间锚点"而不是"整改完成"，就是为了让这三个时刻在输出里各归其位。

**三条常见误判防线（第十八章判定模型的输入约束）：**

- **"这次没读到"不能记成"这次没问题"。** 每一个 `none` / 空值都落进 `L0-LOW`，并进入缺口计数；样例三与第廿三章的 `cov_drop` 判定是同一条纪律的两次落地。
- **"这一项消失了"不能记成"这一项修好了"。** 只有采集面未收窄时的消失才降级为 `risk_closed_needs_proof`，且仍要求一条变更依据。
- **"数量变化"不能记成"设备数变化"。** TKIP 与蓝牙绑定两列输出的是**记号行数**（`wifi_tkip_marker_lines`、`bt_bond_marker_lines`），脚本 2 对它们的描述明确写"这是记号行数，不等同于设备台数"——同一台设备在一次 `dumpsys` 里可能有多行 TKIP 记号。

---

## 十八、判定模型与四条纪律

本章把全文的方法收敛成一个可执行的模型：四级 + 四条纪律 + 一张码位到等级的映射规则。**模型的输出是"处置类别"，不是"分数"**——一个数字会诱使读者跨设备、跨批次相加比较，而本文所有判级的合法性都建立在"同一设备、同一套判据、同一采集流程"这个前提上（脚本 2 的 `pair_identity_mismatch` 就是为破坏这个前提准备的）。

**四级定义：**

| 等级 | 名称 | 判定含义 | 处置时限口径 | 典型码位 |
| --- | --- | --- | --- | --- |
| `L3-HIGH` | 高危 | 公开口径下属"破损算法 / 无保护 / 身份与域不一致 / 采集面收窄导致高危消失"这一类**已成立**的暴露事实 | 进整改单，附两份输入快照路径与外部公告引用 | `net_open`、`net_wep`、`net_autojoin_open`、`risk_gone_by_coverage`、`open_surface_up`、`strong_auth_down` |
| `L2-WARN` | 待确认 | 暴露**依赖额外条件**或依赖本设备之外的对象（接入点侧、控制器侧、企业策略侧），或状态位翻转尚无变更依据 | 先分辨"配置变更"还是"策略放宽"，再决定是否升级 | `net_tkip`、`net_psk_only`、`bt_discoverable`、`nfc_secure_disabled`、`wifi_scan_always_on`、`build_type_nonuser`、`wifi_hal_iface_hidl`、`pair_identity_mismatch` |
| `L1-INFO` | 事实记录 | 加固在位、方向改善、或中性事实（能力存在、版本可读、通道存在） | 归档，作为下一次差分的输入 | `net_sae_present`、`bt_secure_connections_marker`、`nfc_secure_enabled`、`spl_read`、`module_channel_added`、`no_drift` |
| `L0-LOW` | 缺口 | **输入不足**，不构成任何方向判定 | 走流程上报，不当作通过；连续两次未闭合升级为流程告警 | `wifi_dump_missing`、`net_kmgmt_unread`、`baseband_version_absent`、`bt_state_unread`、`gap_persistent`、`delta_unread` |

**四条纪律（与系列其余报告一致，此处给出无线侧的具体形态）：**

- **纪律一：未采到 = 缺口，不是通过。** 所有 `*_unread` / `*_absent` / `*_missing` 码位一律 `L0-LOW`。特别地，射频开关读不到（`bt_state_unread`）**不允许**用"`dumpsys bluetooth_manager` 有内容"来代替状态判定——脚本里那一条说明句写的就是这句话。
- **纪律二：消失 ≠ 已修。** 差分里"基线有、当前无"的信号码分两支：当前缺口数上升 → `risk_gone_by_coverage`（`L3-HIGH`，读作采集面收窄）；缺口数未升 → `risk_closed_needs_proof`（`L2-WARN`，仍要求变更依据）。第廿三章样例三给出真实输出：同一批判据缺口由 5 变 17，全部"消失"的高危与中危码位被逐条读作采集退化。
- **纪律三：数量变化必须有方向语义。** 强认证类计数（SAE/OWE/EAP）**减少**判 `strong_auth_down`(L3)；无保护/破损类计数（NONE/WEP/TKIP 记号）**增加**判 `open_surface_up`(L3)；两者都下降时判 `L1-INFO` 并要求"一条变更依据支撑"，不自动记为成果。`unknown_kmgmt` 计数变化只判 `L2-WARN` 并说明"这一批网络在相应那份快照里不可判级，方向本身不构成结论"。
- **纪律四：跨层组合才是窗口。** 单条 `L1`/`L2` 事实不构成完整暴露判断，脚本因此内置了两组组合判定：`nfc_lockscreen_emulation`（NFC 开启 **且** Secure NFC 为假）与 `net_psk_only`（存在 PSK **且**本次未见 SAE）。第六章判据三与第九章过渡模式的官方句子分别是这两组的设计依据。**反面同样成立**：`bt_radio_on` 与 `bt_discoverable` 同时出现时，读者要看到的是"暴露存在性 + 窗口扩大"两个事实，而不是把它们合成一个分数。

**等级不跨设备比较。** 这是模型的一条硬限制：脚本 1 的 `SUMMARY,high=…` 计数只对该设备有意义，因为每台设备适用的判据子集不同（无 NFC 的设备上 NFC 相关码位根本不出现，样例里 `phone-c` 那台子系统数为 0/4）。第廿五章的 K 指标因此全部按"同设备时间序列"或"同基线判据集"设计，不给出跨设备的加权平均。

**判级映射的三条补充规则：**

1. **厂商输出不符标记词契约时判缺口，不判缺陷。** Wi-Fi 网络清点只认 `nid=` 与 `keyMgmt=` 两个记号；一个有几千字节 `dumpsys` 却一行记号都解析不出的设备，得到的是 `wifi_dump_unparsed`(L0)，而不是"该设备无可配置网络"（L1）。这是把"解析失败"与"不存在"分开的唯一办法。
2. **"面不存在"是一类事实，不是一类风险。** `wifi_feature_absent` / `bt_feature_absent` / `nfc_feature_absent` / `cell_feature_absent` 判 `L1-INFO`，而"没有特性"与"没读到特性"（`*_feature_unknown`，`L0-LOW`）必须是两个码位——样例三里 0/4 子系统的设备正是靠这一对区分"平板没有蜂窝"与"采集失败"。
3. **构建形态是前置条件而不是风险项。** `build_type_nonuser`(L2) 的存在理由与[平台代码组织与构建体系篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)一致：非发行构建上，无线侧调试通道的默认状态与发行构建不是同一条基线，因此脚本 2 在机型相同时仍单独比对 `build_type` 并输出 `build_type_changed`(L2)。

**模型给读者的最后一句话**：这套判定不能证明设备安全，只能证明三件事——此刻的配置存量是什么、相对基线变了什么、以及哪些东西我们这次没看到。第二件事最有价值，第三件事最容易被忽略，而它决定前两件能不能用于结单。

---

## 十九、脚本 1 的设计：一次只读采集，七类输出行

**用途与用法。** `wireless-posture-snapshot.sh <设备序列号>` 对**自有或已获授权测试**的设备做一次无线侧姿态采集，把结果以逗号分隔文本写到标准输出。它不写设备、不装任何东西、不改配置、不访问网络，也不打印 SSID 与蓝牙设备地址。

```bash
#!/usr/bin/env bash
# 脚本 1: wireless-posture-snapshot.sh — 设备无线侧姿态快照（只读）
# 用法: wireless-posture-snapshot.sh <设备序列号>
# 依赖: adb；只调用 get-state / getprop / settings get / dumpsys / pm list features / toybox ls，全部只读
# 不写设备、不安装任何东西、不访问网络；不打印 SSID 与蓝牙设备地址
set -u
dev="${1:?用法: wireless-posture-snapshot.sh <设备序列号>}"
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
setv() { # setv <变量名> <表> <键>
  local _v; _v="$(raw settings get "$2" "$3" | head -1)"
  [ -z "$_v" ] && _v=none
  [ "$_v" = "null" ] && _v=none
  printf 'SET,%s/%s,%s\n' "$2" "$3" "$_v"
  printf -v "$1" '%s' "$_v"
}

model="$(p ro.product.model)"; manuf="$(p ro.product.vendor.manufacturer)"
sdk="$(p ro.build.version.sdk)"; spl="$(p ro.build.version.security_patch)"
btype="$(p ro.build.type)"
bb="$(p gsm.version.baseband)"; btl="$(p ro.boot.bootloader)"
meta device_serial "$dev"; meta model "$model"; meta manufacturer "$manuf"
meta sdk "$sdk"; meta security_patch "$spl"; meta build_type "$btype"
meta baseband_prop "$bb"; meta bootloader_prop "$btl"

wifi_dump="$(raw dumpsys wifi)"
bt_dump="$(raw dumpsys bluetooth_manager)"
nfc_dump="$(raw dumpsys nfc)"
feats="$(raw pm list features)"
apex="$(raw toybox ls /apex)"
wbytes="$(printf '%s' "$wifi_dump" | wc -c | tr -d ' ')"
bbytes="$(printf '%s' "$bt_dump"  | wc -c | tr -d ' ')"
nbytes="$(printf '%s' "$nfc_dump" | wc -c | tr -d ' ')"
meta wifi_dump_bytes "$wbytes"; meta bt_dump_bytes "$bbytes"; meta nfc_dump_bytes "$nbytes"

featlist="$(printf '%s\n' "$feats" | sed -e 's/^[[:space:]]*//' -e 's/^feature[:=]//' -e 's/^.*feature=//' -e 's/:.*$//' | grep '[^[:space:]]' || true)"
feat() { # feat <特性全名> -> present | absent | unknown
  if [ -z "$feats" ]; then echo unknown; return; fi
  if printf '%s\n' "$featlist" | grep -qxF "$1"; then echo present; else echo absent; fi
}
f_wifi="$(feat android.hardware.wifi)"; f_p2p="$(feat android.hardware.wifi.direct)"
f_aw="$(feat android.hardware.wifi.aware)"; f_pp="$(feat android.hardware.wifi.passpoint)"
f_bt="$(feat android.hardware.bluetooth)"; f_nfc="$(feat android.hardware.nfc)"
f_cell="$(feat android.hardware.telephony)"
FEAT() { printf 'FEAT,%s,%s\n' "$1" "$2"; }
FEAT android.hardware.wifi "$f_wifi"; FEAT android.hardware.wifi.direct "$f_p2p"
FEAT android.hardware.wifi.aware "$f_aw"; FEAT android.hardware.wifi.passpoint "$f_pp"
FEAT android.hardware.bluetooth "$f_bt"; FEAT android.hardware.nfc "$f_nfc"
FEAT android.hardware.telephony "$f_cell"

apexhas() { # apexhas <模块名> -> mounted | absent | unknown
  if [ -z "$apex" ]; then echo unknown; return; fi
  case "$apex" in *"$1"*) echo mounted ;; *) echo absent ;; esac
}
m_wifi="$(apexhas com.android.wifi)"; m_bt="$(apexhas com.google.android.bt)"
printf 'MOD,wifi,%s\n' "$m_wifi"
printf 'MOD,bluetooth,%s\n' "$m_bt"

setv v_bt global bluetooth_on
setv v_wifi global wifi_on
setv v_nfc secure nfc_on
setv v_scan secure wifi_scan_always_available
setv v_air global airplane_mode_on

# ---------- Wi-Fi：已配置网络的密钥管理形态 ----------
# 标记词契约：只认 nid= 与 keyMgmt=<值> 两个记号；厂商输出不符时按解析缺口处理
netrows="$(printf '%s\n' "$wifi_dump" | awk '
  /nid=/ {
    line = $0; km = "UNKNOWN"
    if (match(line, /keyMgmt=[A-Za-z0-9_-]+/)) km = substr(line, RSTART + 8, RLENGTH - 8)
    aj = "unset"
    if (line ~ /autojoin=true/) aj = "on"
    else if (line ~ /autojoin=false/) aj = "off"
    else if (line ~ /AutojoinEnabled=true/) aj = "on"
    else if (line ~ /AutojoinEnabled=false/) aj = "off"
    printf "NET,%s,%s\n", km, aj
  }')"
net_total="$(printf '%s\n' "$netrows" | grep -c '^NET,' || true)"
n_none=0; n_wep=0; n_psk=0; n_sae=0; n_owe=0; n_eap=0; n_unknown=0; aj_open=0
while IFS=, read -r tag km aj; do
  [ "$tag" = "NET" ] || continue
  case "$km" in
    NONE)    n_none=$(( n_none + 1 )); [ "$aj" = "on" ] && aj_open=$(( aj_open + 1 )) ;;
    WEP)     n_wep=$(( n_wep + 1 ));   [ "$aj" = "on" ] && aj_open=$(( aj_open + 1 )) ;;
    PSK)     n_psk=$(( n_psk + 1 )) ;;
    SAE)     n_sae=$(( n_sae + 1 )) ;;
    OWE)     n_owe=$(( n_owe + 1 )) ;;
    EAP)     n_eap=$(( n_eap + 1 )) ;;
    UNKNOWN) n_unknown=$(( n_unknown + 1 )) ;;
  esac
done <<EOF
$netrows
EOF
n_tkip=0
[ -n "$wifi_dump" ] && n_tkip="$(printf '%s\n' "$wifi_dump" | grep -c 'TKIP' || true)"
cnt wifi_networks_total "$net_total"; cnt wifi_networks_unknown_kmgmt "$n_unknown"
cnt wifi_networks_none "$n_none"; cnt wifi_networks_wep "$n_wep"
cnt wifi_networks_psk "$n_psk"; cnt wifi_networks_sae "$n_sae"
cnt wifi_networks_owe "$n_owe"; cnt wifi_networks_eap "$n_eap"
cnt wifi_tkip_marker_lines "$n_tkip"

if [ -z "$wifi_dump" ]; then
  row L0-LOW wifi_dump_missing "未采到 dumpsys wifi：Wi-Fi 侧全部判定不成立，不得读成没有开放网络"
elif [ "$net_total" -eq 0 ]; then
  row L0-LOW wifi_dump_unparsed "dumpsys wifi 有 $wbytes 字节但没有一行含 nid= 记号：厂商输出与本脚本标记词契约不符，本次网络清点范围为零"
else
  [ "$n_none" -gt 0 ] && row L3-HIGH net_open "$n_none 个已配置网络使用无密钥管理（keyMgmt=NONE）：同频段的任何人都可提供同名接入点"
  [ "$n_wep" -gt 0 ]  && row L3-HIGH net_wep "$n_wep 个已配置网络使用 WEP：公开口径下属破损算法而不是弱参数"
  [ "$aj_open" -gt 0 ] && row L3-HIGH net_autojoin_open "$aj_open 个无保护网络带自动加入标记：设备会在无人确认时主动连到同类名称的网络"
  [ "$n_tkip" -gt 0 ] && row L2-WARN net_tkip "$n_tkip 行出现 TKIP 记号：RC4 系组加密，公开口径下属风险加密配置"
  [ "$n_unknown" -gt 0 ] && row L0-LOW net_kmgmt_unread "$n_unknown 个网络记录带 nid= 却没有 keyMgmt= 字段：这些网络本次不判级"
  [ "$n_sae" -gt 0 ]  && row L1-INFO net_sae_present "$n_sae 个网络使用 SAE：WPA3-Personal 的握手，抗离线字典与抗降级优于 PSK"
  [ "$n_owe" -gt 0 ]  && row L1-INFO net_owe_present "$n_owe 个网络使用 OWE：在开放介质上建立加密会话，但不提供网络身份凭据"
  [ "$n_eap" -gt 0 ]  && row L1-INFO net_eap_present "$n_eap 个网络使用 EAP：企业认证，信任锚落在证书与服务器侧配置"
  if [ "$n_psk" -gt 0 ] && [ "$n_sae" -eq 0 ]; then
    row L2-WARN net_psk_only "$n_psk 个网络停留在 PSK 且本次未见 SAE 记录：降级与离线字典风险的暴露面没有被关闭"
  fi
  if printf '%s\n' "$wifi_dump" | grep -qi 'aidl'; then
    row L1-INFO wifi_hal_iface_aidl "dumpsys wifi 中出现 AIDL 记号：无线 HAL 走稳定 AIDL 接口形态"
  elif printf '%s\n' "$wifi_dump" | grep -qi 'hidl'; then
    row L2-WARN wifi_hal_iface_hidl "dumpsys wifi 中只出现 HIDL 记号：仍在使用被公开口径标注为弃用的接口语言"
  else
    row L0-LOW wifi_hal_iface_unknown "未采到 HAL 接口语言记号：不判定这一层的归属"
  fi
fi
case "$f_wifi" in
  absent)  row L1-INFO wifi_feature_absent "设备未声明 android.hardware.wifi：Wi-Fi 面在本设备不存在" ;;
  unknown) row L0-LOW wifi_feature_unknown "未采到 pm list features：不能区分没有 Wi-Fi 与没读到" ;;
esac
if [ "$f_wifi" = present ] && [ "$f_p2p" = absent ]; then
  row L1-INFO wifi_p2p_absent "未声明 Wi-Fi Direct 特性：近场直连通道在本设备不存在"
fi
if [ "$f_wifi" = present ] && [ "$f_pp" = present ]; then
  row L1-INFO wifi_passpoint_present "声明 Passpoint 特性：802.11u 侧的网络发现依赖固件支持，属厂商责任面"
fi
case "$m_wifi" in
  mounted) row L1-INFO wifi_module_mounted "com.android.wifi 已挂载：框架侧 Wi-Fi 栈存在独立于整机 OTA 的更新通道" ;;
  absent)  row L0-LOW wifi_module_absent "/apex 清单里没有 com.android.wifi：更新通道未知，可能是版本区间或采集不全" ;;
  unknown) row L0-LOW wifi_module_unread "未采到 /apex 清单：不判定 Wi-Fi 框架侧的可更新性" ;;
esac

# ---------- 蓝牙 ----------
case "$v_bt" in
  1)    row L1-INFO bt_radio_on "bluetooth_on=1：蓝牙射频开启，协议栈处于可被空口触达的状态" ;;
  0)    row L1-INFO bt_radio_off "bluetooth_on=0：蓝牙射频关闭；该行可靠性取决于这一 settings 键在本设备可读" ;;
  none) row L0-LOW bt_state_unread "未采到 bluetooth_on：不判定射频状态，不能用 dumpsys 有内容来代替" ;;
esac
if [ -z "$bt_dump" ]; then
  row L0-LOW bt_dump_missing "未采到 dumpsys bluetooth_manager：蓝牙侧判定不成立"
else
  if printf '%s\n' "$bt_dump" | grep -qiE 'discoverable[[:space:]]*[:=][[:space:]]*(on|true)|ScanMode[[:space:]]*[:=][[:space:]]*(ConnectableDiscoverable|CONNECTABLE_DISCOVERABLE)'; then
    row L2-WARN bt_discoverable "采到可发现/可连接记号：设备处于可被陌生端发起配对或连接的窗口"
  elif printf '%s\n' "$bt_dump" | grep -qiE 'discoverab|scanmode'; then
    row L1-INFO bt_discoverable_off "可发现性字段存在但为关闭形态"
  else
    row L0-LOW bt_discoverable_unread "未采到可发现性记号：不能把读不到当成不可发现"
  fi
  bond_n="$(printf '%s\n' "$bt_dump" | grep -icE 'bond state[[:space:]]*[:=][[:space:]]*bonded|Devices bonded' || true)"
  bond_n="${bond_n:-0}"
  cnt bt_bond_marker_lines "$bond_n"
  if [ "$bond_n" -gt 0 ]; then
    row L1-INFO bt_bonded_present "蓝牙侧有 $bond_n 行绑定记号：已配对设备在公开口径下属于本地身份，历史绑定不会随关闭射频消失"
  else
    row L0-LOW bt_bonded_unread "本次未采到绑定记号：不等于没有绑定，需换用可读的输出格式确认"
  fi
  if printf '%s\n' "$bt_dump" | grep -qiE 'legacy[[:space:]]+(pairing|method)|secure[[:space:]]*connections?[[:space:]]*[:=][[:space:]]*false'; then
    row L2-WARN bt_legacy_pairing_marker "绑定记录里出现旧配对方式或非 Secure Connections 记号：认证强度类缺陷的公开落点"
  fi
  if printf '%s\n' "$bt_dump" | grep -qiE 'secure[[:space:]]*connections?[[:space:]]*[:=][[:space:]]*true'; then
    row L1-INFO bt_secure_connections_marker "绑定记录里采到 Secure Connections 为真记号：既有绑定使用了公钥认证配对路径"
  elif ! printf '%s\n' "$bt_dump" | grep -qiE 'legacy[[:space:]]+(pairing|method)|secure[[:space:]]*connections?[[:space:]]*[:=][[:space:]]*false'; then
    row L0-LOW bt_pairing_strength_unread "未采到配对强度记号：不判定既有绑定用的是哪种配对方式"
  fi
fi
case "$m_bt" in
  mounted) row L1-INFO bt_module_mounted "com.google.android.bt 已挂载：蓝牙栈存在独立于整机 OTA 的更新通道" ;;
  absent)  row L0-LOW bt_module_absent "/apex 清单里没有蓝牙模块：公开口径下它是可选模块，缺失只说明通道未知" ;;
  unknown) row L0-LOW bt_module_unread "未采到 /apex 清单：不判定蓝牙栈的可更新性" ;;
esac
case "$f_bt" in
  absent)  row L1-INFO bt_feature_absent "设备未声明 android.hardware.bluetooth：蓝牙面在本设备不存在" ;;
  unknown) row L0-LOW bt_feature_unknown "未采到 pm list features：不能区分没有蓝牙与没读到" ;;
esac

# ---------- NFC ----------
case "$v_nfc" in
  1)    row L1-INFO nfc_on "nfc_on=1：NFC 处于开启状态，近场通道对本设备可达" ;;
  0)    row L1-INFO nfc_off "nfc_on=0：NFC 关闭" ;;
  none) row L0-LOW nfc_state_unread "未采到 nfc_on：不判定 NFC 状态" ;;
esac
if [ -z "$nfc_dump" ]; then
  if [ "$f_nfc" = absent ]; then
    row L1-INFO nfc_feature_absent "设备未声明 android.hardware.nfc：NFC 面在本设备不存在"
  else
    row L0-LOW nfc_dump_missing "未采到 dumpsys nfc：锁屏卡模拟与路由侧判定不成立"
  fi
else
  if printf '%s\n' "$nfc_dump" | grep -qiE 'secure[[:space:]]*nfc[[:space:]]*[:=][[:space:]]*true|SecureNfcEnabled=true'; then
    row L1-INFO nfc_secure_enabled "采到 Secure NFC 为真：off-host 卡模拟被限制在解锁状态下"
  elif printf '%s\n' "$nfc_dump" | grep -qiE 'secure[[:space:]]*nfc|SecureNfcEnabled'; then
    row L2-WARN nfc_secure_disabled "采到 Secure NFC 字段但为假：锁屏或关机状态下仍可能接受 off-host 卡模拟交易"
    if [ "$v_nfc" = 1 ]; then
      row L2-WARN nfc_lockscreen_emulation "NFC 开启且 Secure NFC 为假：两个条件合起来才是可被无人值守触碰的窗口，单看任一行都会低估"
    fi
  else
    row L0-LOW nfc_secure_unread "未采到 Secure NFC 记号：读不到不等于这个限制不存在，需要向厂商确认这一位的可读性"
  fi
fi

# ---------- 蜂窝与基带（设备侧可读性本身就是结论） ----------
case "$f_cell" in
  absent)  row L1-INFO cell_feature_absent "设备未声明 android.hardware.telephony：蜂窝面在本设备不存在" ;;
  unknown) row L0-LOW cell_feature_unknown "未采到 pm list features：不判定蜂窝面存在性" ;;
esac
if [ -n "$bb" ] && [ "$bb" != "none" ]; then
  row L1-INFO baseband_version_read "gsm.version.baseband 可读：这是设备侧唯一能把蜂窝侧变更归因到具体固件版本的字段"
else
  row L0-LOW baseband_version_absent "未采到 gsm.version.baseband：基带侧任何变更在设备上不可归因，向厂商索取证据是唯一路径"
fi
if [ -n "$btl" ] && [ "$btl" != "none" ]; then
  row L1-INFO bootloader_version_read "ro.boot.bootloader 可读：可比对随包分发的引导与调制解调器组件版本"
else
  row L0-LOW bootloader_version_absent "未采到 ro.boot.bootloader：不判定引导与基带包版本证据"
fi

# ---------- 跨子系统的姿态项 ----------
case "$v_scan" in
  1)    row L2-WARN wifi_scan_always_on "wifi_scan_always_available=1：Wi-Fi 关闭时仍允许扫描，公开口径下这是可被定位侧利用的通道" ;;
  0)    row L1-INFO wifi_scan_always_off "wifi_scan_always_available=0：Wi-Fi 关闭时不扫描" ;;
  none) row L0-LOW wifi_scan_state_unread "未采到 wifi_scan_always_available：不判定关网扫描" ;;
esac
case "$v_air" in
  1)    row L1-INFO airplane_mode_on "airplane_mode_on=1：整机无线处于关闭口径，逐项射频状态仍需单独复核" ;;
  0)    row L1-INFO airplane_mode_off "airplane_mode_on=0" ;;
  none) row L0-LOW airplane_mode_unread "未采到 airplane_mode_on：不判定总开关形态" ;;
esac
if [ -z "$spl" ]; then
  row L0-LOW spl_absent "未采到安全补丁级别：本快照不做任何补丁到达性判断"
else
  row L1-INFO spl_read "security_patch=$spl：只作为本快照的时间锚点，不据以判定无线侧缺陷是否已修"
fi
if [ -z "$btype" ]; then
  row L0-LOW build_type_absent "未采到 ro.build.type：不判定构建形态"
elif [ "$btype" = user ]; then
  row L1-INFO build_type_user "ro.build.type=user：发行构建"
else
  row L2-WARN build_type_nonuser "ro.build.type=$btype：非发行构建，无线侧调试通道可能被默认打开"
fi
[ -z "$model" ] && row L0-LOW identity_missing "未采到机型标识：本快照无法与厂商公告对齐"

scope=0
for x in "$f_wifi" "$f_bt" "$f_nfc" "$f_cell"; do [ "$x" = present ] && scope=$(( scope + 1 )); done
printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,subsystems=%s/4,nets=%s,model=%s,spl=%s\n' \
  "$hi" "$warn" "$gap" "$info" "$scope" "$net_total" "${model:-none}" "${spl:-none}"
```

**依赖面就是它的边界。** 脚本只调用六个 adb 子命令，全部只读：`get-state`、`getprop`、`settings get`、`dumpsys wifi|bluetooth_manager|nfc`、`pm list features`、`toybox ls /apex`。这一串就是本文所有设备侧判断的输入上限；第十四章矩阵里凡是"设备侧证据 = 无"的行，在这个脚本里也确实没有任何码位——这不是遗漏，是分章时定下的纪律。设备不可达时输出一行 `ERR,device_unreachable,<序列号>` 并以状态码 1 结束，不做任何降级判定。

**七类输出行（脚本 2 的输入契约建立在这七类之上）：**

| 行首 | 含义 | 示例 | 用途 |
| --- | --- | --- | --- |
| `META,<键>,<值>` | 设备元数据与采集量 | `META,security_patch,2026-10-05` | 归因锚点；缺失值统一写 `none` |
| `SET,<表>/<键>,<值>` | settings 状态位 | `SET,secure/wifi_scan_always_available,0` | 策略位事实；`none` 表示读不到而不是关闭 |
| `FEAT,<特性>,present\|absent\|unknown` | 子系统能力面存在性 | `FEAT,android.hardware.telephony,present` | 区分"没有这个面"与"没读到" |
| `MOD,<子系统>,mounted\|absent\|unknown` | Mainline 模块挂载形态 | `MOD,wifi,mounted` | 更新通道证据，不是修复证据 |
| `COUNT,<键>,<数>` | 计数型事实 | `COUNT,wifi_networks_sae,1` | 差分主键之一；`*_marker_lines` 类明确是行数 |
| `ROW,<等级>,<信号码>,<说明>` | 判定行 | `ROW,L3-HIGH,net_wep,1 个已配置网络使用 WEP：…` | 四级判定的唯一出口，共 65 个码位 |
| `SUMMARY,…` | 汇总计数 | `SUMMARY,high=0,warn=0,gap=0,info=19,…` | 缺口计数是纪律一的可复核证明 |

**六个设计决定：**

1. **标记词契约优先于猜测。** Wi-Fi 网络清点只认 `nid=` 与 `keyMgmt=<值>` 两个记号（`autojoin` / `AutojoinEnabled` 两种拼法都接受），不尝试解析厂商的其他字段。厂商输出不符合契约时判 `wifi_dump_unparsed`(L0)，并把 `dumpsys` 字节数留在 `META` 行里作为对照证据——这是第十八章"补充规则 1"的实现。
2. **三态而不是两态。** 凡是"是否存在/是否挂载/是否声明"的判断都返回 `present / absent / unknown` 三值：特性面、模块挂载、射频开关各自都有一对码位（如 `bt_feature_absent` 与 `bt_feature_unknown`）。**两态会把采集失败伪装成"没有"，这正是纪律一禁止的事**。
3. **组合判定内置在脚本里。** `nfc_lockscreen_emulation` 需要 `nfc_on=1` 且 Secure NFC 为假；`net_psk_only` 需要存在 PSK 且本次未见 SAE 记录。两条都在第十八章纪律四中有对应说明，目的是让快照读者不必自己重做与逻辑。
4. **严重度基线来自公开口径而不是主观权重。** WEP 与 keyMgmt=NONE 判 L3，依据是第四章引到的官方句子（OWE"doesn't provide authentication"、WPA3 过渡模式允许老设备不加密）与第二章"邻近即远程"；TKIP 判 L2 而不是 L3，是因为它是"记号行数"而不是可直接计数的网络数。
5. **不采集权限、不采集域与标签。** 机制三与机制四（第十五章）由其它报告负责，本文不重复实现；这条限制也让脚本在任何受管设备上的可执行性更简单——它需要的只是一条 adb 连接。
6. **输出可以直接当档案。** 全部是单行 CSV，不含换行内嵌与二进制；第廿一章的四份样例就是脚本原样输出，未做任何手工编辑（第廿四章给出逐字节比对方法）。

---

## 二十、脚本 1 的输出契约与 65 个信号码

等级列即 `ROW` 行的第二个字段。**这张表的"触发条件"列是判级的唯一事实来源**；任何一处与第十九章脚本正文不一致，以脚本正文为准。

**A. Wi-Fi 已配置网络与协议形态（14 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `net_open` | L3-HIGH | `keyMgmt=NONE` 的网络数 > 0 | 同频任意对象可提供同名接入点；第二章判据一 |
| `net_autojoin_open` | L3-HIGH | 无保护网络中带自动加入标记的数 > 0 | 无人确认时主动连接同类名称网络 |
| `net_wep` | L3-HIGH | `keyMgmt=WEP` 的网络数 > 0 | 第四章：破损算法而不是弱参数 |
| `net_tkip` | L2-WARN | `dumpsys wifi` 中出现 TKIP 记号的行数 > 0 | RC4 系组加密；计数语义是行数 |
| `net_psk_only` | L2-WARN | 存在 PSK 网络且本次未见任何 SAE 记录 | 第四章过渡模式两句；降级暴露面未关闭 |
| `net_kmgmt_unread` | L0-LOW | 有 `nid=` 但无 `keyMgmt=` 字段的网络数 > 0 | 这批网络本次不判级 |
| `net_sae_present` | L1-INFO | SAE 网络数 > 0 | 第四章 SAE 定义句（抗离线字典与密钥恢复） |
| `net_owe_present` | L1-INFO | OWE 网络数 > 0 | 官方句子明确不提供认证，故只记 INFO |
| `net_eap_present` | L1-INFO | EAP 网络数 > 0 | 信任锚落在证书与服务器侧配置 |
| `wifi_dump_missing` | L0-LOW | `dumpsys wifi` 输出为空 | Wi-Fi 侧全部判定不成立 |
| `wifi_dump_unparsed` | L0-LOW | 有字节数但零行含 `nid=` | 标记词契约不符，清点范围为零 |
| `wifi_hal_iface_aidl` | L1-INFO | 输出中出现 AIDL 记号 | 参考资料 6 的接口语言迁移口径 |
| `wifi_hal_iface_hidl` | L2-WARN | 只出现 HIDL 记号 | 被公开口径标注为弃用的接口形态 |
| `wifi_hal_iface_unknown` | L0-LOW | 两种记号都未出现 | 不判定这一层归属 |

**B. Wi-Fi 能力面与更新通道（6 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `wifi_feature_absent` | L1-INFO | 未声明 `android.hardware.wifi` | "面不存在"是事实不是风险 |
| `wifi_feature_unknown` | L0-LOW | `pm list features` 未采到 | 不能区分没有与没读到 |
| `wifi_p2p_absent` | L1-INFO | 有 Wi-Fi 但未声明 Wi-Fi Direct | 近场直连通道在本设备不存在 |
| `wifi_passpoint_present` | L1-INFO | 声明 Passpoint 特性 | 参考资料 8：802.11u 发现面，厂商责任面 |
| `wifi_module_mounted` | L1-INFO | `/apex` 清单含 `com.android.wifi` | 参考资料 12；框架侧存在独立于整机 OTA 的通道 |
| `wifi_module_absent` / `wifi_module_unread` | L0-LOW | 清单里没有 / 清单未采到 | 更新通道未知，不判"未加固" |

**C. 蓝牙（18 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `bt_radio_on` | L1-INFO | `bluetooth_on=1` | 暴露存在性信号（不等于可达性判定） |
| `bt_radio_off` | L1-INFO | `bluetooth_on=0` | 说明句提醒该行可靠性取决于键可读 |
| `bt_state_unread` | L0-LOW | 键读不到 | 不允许用 `dumpsys` 有内容代替状态 |
| `bt_dump_missing` | L0-LOW | `dumpsys bluetooth_manager` 为空 | 蓝牙侧判定不成立 |
| `bt_discoverable` | L2-WARN | 采到可发现/可连接记号 | 可被陌生端发起配对或连接的窗口 |
| `bt_discoverable_off` | L1-INFO | 字段存在但为关闭形态 | 中性事实 |
| `bt_discoverable_unread` | L0-LOW | 未采到可发现性记号 | 读不到不等于不可发现 |
| `bt_bonded_present` | L1-INFO | 绑定记号行数 > 0 | 第六章判据二：绑定是需盘点的资产 |
| `bt_bonded_unread` | L0-LOW | 未采到绑定记号 | 不等于没有绑定 |
| `bt_legacy_pairing_marker` | L2-WARN | 出现旧配对方式或非 Secure Connections 记号 | 参考资料 24/25 的认证假设落点 |
| `bt_secure_connections_marker` | L1-INFO | 出现 Secure Connections 为真记号 | 既有绑定使用公钥认证配对路径 |
| `bt_pairing_strength_unread` | L0-LOW | 两类配对强度记号都没有 | KNB 一节演示的"证据不足时不判级" |
| `bt_module_mounted` / `bt_module_absent` / `bt_module_unread` | L1/L0/L0 | `/apex` 中蓝牙模块三态 | 参考资料 13：可选模块，缺席只说明通道未知 |
| `bt_feature_absent` / `bt_feature_unknown` | L1/L0 | 特性面两态 | 同 B 组逻辑 |

**D. NFC（9 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `nfc_on` / `nfc_off` | L1-INFO | `secure/nfc_on` 为 1 / 0 | 近场通道可达性事实 |
| `nfc_state_unread` | L0-LOW | 键读不到 | — |
| `nfc_dump_missing` | L0-LOW | `dumpsys nfc` 为空且未确认特性缺席 | 锁屏卡模拟侧判定不成立 |
| `nfc_feature_absent` | L1-INFO | 未声明 NFC 且 dump 为空 | 与上一条互斥，避免把"没装"读成"没采到" |
| `nfc_secure_enabled` | L1-INFO | Secure NFC 为真 | 参考资料 9 的定义句 |
| `nfc_secure_disabled` | L2-WARN | 字段存在且为假 | 锁屏/关机下仍可能接受 off-host 交易 |
| `nfc_lockscreen_emulation` | L2-WARN | NFC 开启 **且** Secure NFC 为假 | 纪律四的组合判定 |
| `nfc_secure_unread` | L0-LOW | 未采到 Secure NFC 记号 | 需向厂商确认这一位的可读性 |

**E. 蜂窝与基带（6 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `cell_feature_absent` | L1-INFO | 未声明 telephony | 设备选型事实 |
| `cell_feature_unknown` | L0-LOW | 特性未采到 | — |
| `baseband_version_read` | L1-INFO | `gsm.version.baseband` 有值 | 第十三章：蜂窝侧唯一可归因字段 |
| `baseband_version_absent` | L0-LOW | 该属性读不到 | 变更在设备上不可归因，向厂商索取证据 |
| `bootloader_version_read` | L1-INFO | `ro.boot.bootloader` 有值 | 可比对随包分发的引导与调制解调器组件 |
| `bootloader_version_absent` | L0-LOW | 读不到 | 不判定引导与基带包版本证据 |

**F. 跨子系统姿态（12 个）**

| 信号码 | 等级 | 触发条件 | 设计依据 |
| --- | --- | --- | --- |
| `wifi_scan_always_on` / `wifi_scan_always_off` / `wifi_scan_state_unread` | L2/L1/L0 | 关网扫描位三态 | 第十六章：可被定位侧利用的通道 |
| `airplane_mode_on` / `airplane_mode_off` / `airplane_mode_unread` | L1/L1/L0 | 总开关三态 | 总开关不替代逐项射频状态 |
| `spl_read` / `spl_absent` | L1/L0 | 安全补丁级别可读性 | 第二章判据四：只作时间锚点 |
| `build_type_user` / `build_type_nonuser` / `build_type_absent` | L1/L2/L0 | `ro.build.type` | 非发行构建不是同一条基线 |
| `identity_missing` | L0-LOW | 未采到机型标识 | 快照无法与厂商公告对齐 |

**计数键（`COUNT` 行，9 个键）**：`wifi_networks_total`、`wifi_networks_unknown_kmgmt`、`wifi_networks_none`、`wifi_networks_wep`、`wifi_networks_psk`、`wifi_networks_sae`、`wifi_networks_owe`、`wifi_networks_eap`、`wifi_tkip_marker_lines`、`bt_bond_marker_lines`。它们不判级，只提供脚本 2 的方向判定输入（第廿三章）；命名带 `_marker_lines` 的键明确是行数语义。

**元数据键（`META` 行，12 个）**：`device_serial`、`model`、`manufacturer`、`sdk`、`security_patch`、`build_type`、`baseband_prop`、`bootloader_prop`、`wifi_dump_bytes`、`bt_dump_bytes`、`nfc_dump_bytes`。缺失值一律 `none`，脚本 2 依赖这一约定实现"读不到不判方向"。

---

## 廿一、脚本 1 样例输出（四份真实采集）

以下四份输出由第十九章脚本正文对本机的**替身 adb 档案**（合成设备画像，用于在无真机条件下复核分支）实际运行得到，未手工编辑；第廿四章给出逐字节复核命令。四份分别代表四种典型形态：健康基线、劣化设备、采集退化、射频全关。

**样例一：健康基线（GoodPhone X1，`nets=4`，四个子系统齐全）**

这份输出的价值在于证明"没有 `L3`/`L2` 不等于没有信息"：19 条 `L1-INFO` 记录的是配置存量、通道形态与证据可得性，它们正是下一次差分的基线。注意三处：`bt_bonded_present` 的说明句写明"历史绑定不会随关闭射频消失"（第十六章不能指望的三处之二）；`net_sae_present`/`net_owe_present`/`net_eap_present` 同时出现，说明这台设备的存量已覆盖强认证形态；`SUMMARY` 行的 `gap=0` 是纪律一可复核的证据。

```text
META,device_serial,WGOOD001
META,model,GoodPhone X1
META,manufacturer,Example Mobile
META,sdk,36
META,security_patch,2026-10-05
META,build_type,user
META,baseband_prop,B2.1.4-20260912
META,bootloader_prop,G3.7.1-20260820
META,wifi_dump_bytes,390
META,bt_dump_bytes,315
META,nfc_dump_bytes,121
FEAT,android.hardware.wifi,present
FEAT,android.hardware.wifi.direct,present
FEAT,android.hardware.wifi.aware,absent
FEAT,android.hardware.wifi.passpoint,present
FEAT,android.hardware.bluetooth,present
FEAT,android.hardware.nfc,present
FEAT,android.hardware.telephony,present
MOD,wifi,mounted
MOD,bluetooth,mounted
SET,global/bluetooth_on,1
SET,global/wifi_on,1
SET,secure/nfc_on,1
SET,secure/wifi_scan_always_available,0
SET,global/airplane_mode_on,0
COUNT,wifi_networks_total,4
COUNT,wifi_networks_unknown_kmgmt,0
COUNT,wifi_networks_none,0
COUNT,wifi_networks_wep,0
COUNT,wifi_networks_psk,1
COUNT,wifi_networks_sae,1
COUNT,wifi_networks_owe,1
COUNT,wifi_networks_eap,1
COUNT,wifi_tkip_marker_lines,0
ROW,L1-INFO,net_sae_present,1 个网络使用 SAE：WPA3-Personal 的握手，抗离线字典与抗降级优于 PSK
ROW,L1-INFO,net_owe_present,1 个网络使用 OWE：在开放介质上建立加密会话，但不提供网络身份凭据
ROW,L1-INFO,net_eap_present,1 个网络使用 EAP：企业认证，信任锚落在证书与服务器侧配置
ROW,L1-INFO,wifi_hal_iface_aidl,dumpsys wifi 中出现 AIDL 记号：无线 HAL 走稳定 AIDL 接口形态
ROW,L1-INFO,wifi_passpoint_present,声明 Passpoint 特性：802.11u 侧的网络发现依赖固件支持，属厂商责任面
ROW,L1-INFO,wifi_module_mounted,com.android.wifi 已挂载：框架侧 Wi-Fi 栈存在独立于整机 OTA 的更新通道
ROW,L1-INFO,bt_radio_on,bluetooth_on=1：蓝牙射频开启，协议栈处于可被空口触达的状态
ROW,L1-INFO,bt_discoverable_off,可发现性字段存在但为关闭形态
COUNT,bt_bond_marker_lines,3
ROW,L1-INFO,bt_bonded_present,蓝牙侧有 3 行绑定记号：已配对设备在公开口径下属于本地身份，历史绑定不会随关闭射频消失
ROW,L1-INFO,bt_secure_connections_marker,绑定记录里采到 Secure Connections 为真记号：既有绑定使用了公钥认证配对路径
ROW,L1-INFO,bt_module_mounted,com.google.android.bt 已挂载：蓝牙栈存在独立于整机 OTA 的更新通道
ROW,L1-INFO,nfc_on,nfc_on=1：NFC 处于开启状态，近场通道对本设备可达
ROW,L1-INFO,nfc_secure_enabled,采到 Secure NFC 为真：off-host 卡模拟被限制在解锁状态下
ROW,L1-INFO,baseband_version_read,gsm.version.baseband 可读：这是设备侧唯一能把蜂窝侧变更归因到具体固件版本的字段
ROW,L1-INFO,bootloader_version_read,ro.boot.bootloader 可读：可比对随包分发的引导与调制解调器组件版本
ROW,L1-INFO,wifi_scan_always_off,wifi_scan_always_available=0：Wi-Fi 关闭时不扫描
ROW,L1-INFO,airplane_mode_off,airplane_mode_on=0
ROW,L1-INFO,spl_read,security_patch=2026-10-05：只作为本快照的时间锚点，不据以判定无线侧缺陷是否已修
ROW,L1-INFO,build_type_user,ro.build.type=user：发行构建
SUMMARY,high=0,warn=0,gap=0,info=19,subsystems=4/4,nets=4,model=GoodPhone X1,spl=2026-10-05
```

**样例二：劣化设备（BudgetPhone B9，SPL 停在 2024-02-05，`high=3,warn=9,gap=5`）**

这一份是"多因素叠加"的样本：`net_open`（无保护存量）、`net_autojoin_open`（无人确认即连）、`net_wep`（破损算法）三条 L3 同时成立；`build_type_nonuser` 与 `wifi_scan_always_on` 说明这台设备的调试与扫描通道默认打开；`net_psk_only` 与 `bt_legacy_pairing_marker` 是两条"依赖额外条件"的 L2。请特别注意 `gap=5`：**同一份快照里既有高危成立又有采集缺口**，这说明"劣化"与"采不全"经常同时出现，读者不能因为 L3 行醒目就忽略缺口列（第二十五章 K9 指标按此设计）。

```text
META,device_serial,WBAD001
META,model,BudgetPhone B9
META,manufacturer,Example Mobile
META,sdk,31
META,security_patch,2024-02-05
META,build_type,userdebug
META,baseband_prop,none
META,bootloader_prop,none
META,wifi_dump_bytes,424
META,bt_dump_bytes,269
META,nfc_dump_bytes,95
FEAT,android.hardware.wifi,present
FEAT,android.hardware.wifi.direct,present
FEAT,android.hardware.wifi.aware,absent
FEAT,android.hardware.wifi.passpoint,absent
FEAT,android.hardware.bluetooth,present
FEAT,android.hardware.nfc,present
FEAT,android.hardware.telephony,present
MOD,wifi,absent
MOD,bluetooth,absent
SET,global/bluetooth_on,1
SET,global/wifi_on,1
SET,secure/nfc_on,1
SET,secure/wifi_scan_always_available,1
SET,global/airplane_mode_on,0
COUNT,wifi_networks_total,5
COUNT,wifi_networks_unknown_kmgmt,1
COUNT,wifi_networks_none,2
COUNT,wifi_networks_wep,1
COUNT,wifi_networks_psk,1
COUNT,wifi_networks_sae,0
COUNT,wifi_networks_owe,0
COUNT,wifi_networks_eap,0
COUNT,wifi_tkip_marker_lines,1
ROW,L3-HIGH,net_open,2 个已配置网络使用无密钥管理（keyMgmt=NONE）：同频段的任何人都可提供同名接入点
ROW,L3-HIGH,net_wep,1 个已配置网络使用 WEP：公开口径下属破损算法而不是弱参数
ROW,L3-HIGH,net_autojoin_open,2 个无保护网络带自动加入标记：设备会在无人确认时主动连到同类名称的网络
ROW,L2-WARN,net_tkip,1 行出现 TKIP 记号：RC4 系组加密，公开口径下属风险加密配置
ROW,L0-LOW,net_kmgmt_unread,1 个网络记录带 nid= 却没有 keyMgmt= 字段：这些网络本次不判级
ROW,L2-WARN,net_psk_only,1 个网络停留在 PSK 且本次未见 SAE 记录：降级与离线字典风险的暴露面没有被关闭
ROW,L2-WARN,wifi_hal_iface_hidl,dumpsys wifi 中只出现 HIDL 记号：仍在使用被公开口径标注为弃用的接口语言
ROW,L0-LOW,wifi_module_absent,/apex 清单里没有 com.android.wifi：更新通道未知，可能是版本区间或采集不全
ROW,L1-INFO,bt_radio_on,bluetooth_on=1：蓝牙射频开启，协议栈处于可被空口触达的状态
ROW,L2-WARN,bt_discoverable,采到可发现/可连接记号：设备处于可被陌生端发起配对或连接的窗口
COUNT,bt_bond_marker_lines,3
ROW,L1-INFO,bt_bonded_present,蓝牙侧有 3 行绑定记号：已配对设备在公开口径下属于本地身份，历史绑定不会随关闭射频消失
ROW,L2-WARN,bt_legacy_pairing_marker,绑定记录里出现旧配对方式或非 Secure Connections 记号：认证强度类缺陷的公开落点
ROW,L0-LOW,bt_module_absent,/apex 清单里没有蓝牙模块：公开口径下它是可选模块，缺失只说明通道未知
ROW,L1-INFO,nfc_on,nfc_on=1：NFC 处于开启状态，近场通道对本设备可达
ROW,L2-WARN,nfc_secure_disabled,采到 Secure NFC 字段但为假：锁屏或关机状态下仍可能接受 off-host 卡模拟交易
ROW,L2-WARN,nfc_lockscreen_emulation,NFC 开启且 Secure NFC 为假：两个条件合起来才是可被无人值守触碰的窗口，单看任一行都会低估
ROW,L0-LOW,baseband_version_absent,未采到 gsm.version.baseband：基带侧任何变更在设备上不可归因，向厂商索取证据是唯一路径
ROW,L0-LOW,bootloader_version_absent,未采到 ro.boot.bootloader：不判定引导与基带包版本证据
ROW,L2-WARN,wifi_scan_always_on,wifi_scan_always_available=1：Wi-Fi 关闭时仍允许扫描，公开口径下这是可被定位侧利用的通道
ROW,L1-INFO,airplane_mode_off,airplane_mode_on=0
ROW,L1-INFO,spl_read,security_patch=2024-02-05：只作为本快照的时间锚点，不据以判定无线侧缺陷是否已修
ROW,L2-WARN,build_type_nonuser,ro.build.type=userdebug：非发行构建，无线侧调试通道可能被默认打开
SUMMARY,high=3,warn=9,gap=5,info=5,subsystems=4/4,nets=5,model=BudgetPhone B9,spl=2024-02-05
```

**样例三：采集退化（`subsystems=0/4`，`gap=17`，`info=0`）**

这一份最容易被误读成"什么都没查到所以没风险"。它实际展示的是第十八章补充规则 2 与纪律一的同时成立：`model=none` 与 `spl=none` 触发 `identity_missing` / `spl_absent`；`pm list features` 读不到，于是 Wi-Fi、蓝牙、蜂窝三个特性面全部落到 `*_feature_unknown`；`dumpsys wifi` **有字节数但没有一行含 `nid=`**，因此得到的是 `wifi_dump_unparsed` 而不是 `wifi_dump_missing`（两个缺口的区分靠 `META,wifi_dump_bytes`）；蓝牙与 NFC 侧输出为空，得到 `bt_dump_missing` / `nfc_dump_missing`。17 行判定全部是 `L0-LOW`，`info=0`——**当一份快照里没有任何 `L1-INFO` 事实时，它描述的是采集状态而不是设备状态**，这一行 `SUMMARY` 就是这份快照唯一可靠的结论。第廿三章样例三会看到它在差分里被读作采集面收窄（`cov_drop=yes`）而不是整改成果。

```text
META,device_serial,WGAP001
META,model,none
META,manufacturer,UnknownPhone
META,sdk,none
META,security_patch,none
META,build_type,none
META,baseband_prop,none
META,bootloader_prop,none
META,wifi_dump_bytes,215
META,bt_dump_bytes,0
META,nfc_dump_bytes,0
FEAT,android.hardware.wifi,unknown
FEAT,android.hardware.wifi.direct,unknown
FEAT,android.hardware.wifi.aware,unknown
FEAT,android.hardware.wifi.passpoint,unknown
FEAT,android.hardware.bluetooth,unknown
FEAT,android.hardware.nfc,unknown
FEAT,android.hardware.telephony,unknown
MOD,wifi,unknown
MOD,bluetooth,unknown
SET,global/bluetooth_on,none
SET,global/wifi_on,none
SET,secure/nfc_on,none
SET,secure/wifi_scan_always_available,none
SET,global/airplane_mode_on,none
COUNT,wifi_networks_total,0
COUNT,wifi_networks_unknown_kmgmt,0
COUNT,wifi_networks_none,0
COUNT,wifi_networks_wep,0
COUNT,wifi_networks_psk,0
COUNT,wifi_networks_sae,0
COUNT,wifi_networks_owe,0
COUNT,wifi_networks_eap,0
COUNT,wifi_tkip_marker_lines,0
ROW,L0-LOW,wifi_dump_unparsed,dumpsys wifi 有 215 字节但没有一行含 nid= 记号：厂商输出与本脚本标记词契约不符，本次网络清点范围为零
ROW,L0-LOW,wifi_feature_unknown,未采到 pm list features：不能区分没有 Wi-Fi 与没读到
ROW,L0-LOW,wifi_module_unread,未采到 /apex 清单：不判定 Wi-Fi 框架侧的可更新性
ROW,L0-LOW,bt_state_unread,未采到 bluetooth_on：不判定射频状态，不能用 dumpsys 有内容来代替
ROW,L0-LOW,bt_dump_missing,未采到 dumpsys bluetooth_manager：蓝牙侧判定不成立
ROW,L0-LOW,bt_module_unread,未采到 /apex 清单：不判定蓝牙栈的可更新性
ROW,L0-LOW,bt_feature_unknown,未采到 pm list features：不能区分没有蓝牙与没读到
ROW,L0-LOW,nfc_state_unread,未采到 nfc_on：不判定 NFC 状态
ROW,L0-LOW,nfc_dump_missing,未采到 dumpsys nfc：锁屏卡模拟与路由侧判定不成立
ROW,L0-LOW,cell_feature_unknown,未采到 pm list features：不判定蜂窝面存在性
ROW,L0-LOW,baseband_version_absent,未采到 gsm.version.baseband：基带侧任何变更在设备上不可归因，向厂商索取证据是唯一路径
ROW,L0-LOW,bootloader_version_absent,未采到 ro.boot.bootloader：不判定引导与基带包版本证据
ROW,L0-LOW,wifi_scan_state_unread,未采到 wifi_scan_always_available：不判定关网扫描
ROW,L0-LOW,airplane_mode_unread,未采到 airplane_mode_on：不判定总开关形态
ROW,L0-LOW,spl_absent,未采到安全补丁级别：本快照不做任何补丁到达性判断
ROW,L0-LOW,build_type_absent,未采到 ro.build.type：不判定构建形态
ROW,L0-LOW,identity_missing,未采到机型标识：本快照无法与厂商公告对齐
SUMMARY,high=0,warn=0,gap=17,info=0,subsystems=0/4,nets=0,model=none,spl=none
```

**样例四：射频全关（GoodPhone X1，`nets=0`，`gap=3`）**

飞行模式 + 全部射频关闭 + 已配置网络被清空的形态。四点值得对照：`airplane_mode_on` 判 L1 并在说明里写"整机无线处于关闭口径，逐项射频状态仍需单独复核"——总开关不替代逐项；`SET` 五行给出 `bluetooth_on=0`、`wifi_on=0`、`nfc_on=0`、`wifi_scan_always_available=0`、`airplane_mode_on=1`，这是本快照最有价值的部分，因为它是一份**完整可读的姿态基线**（`gap=3` 只来自三个 `dumpsys` 为空）；`COUNT,wifi_networks_*` 九个键全部为 0，与样例三"有字节但解析不出"形成对照，这里的 0 是**可读的零**而不是解析失败；蓝牙侧输出为空导致 `bt_dump_missing`，因此绑定清点本次**没有结论**（`bt_bond_marker_lines` 这一计数键不会出现），这正是第十六章"不能指望关闭射频清理历史凭据"在输出层的体现。这份快照同时是第廿三章样例四的基线输入。

```text
META,device_serial,WOFF001
META,model,GoodPhone X1
META,manufacturer,Example Mobile
META,sdk,36
META,security_patch,2026-10-05
META,build_type,user
META,baseband_prop,B2.1.4-20260912
META,bootloader_prop,G3.7.1-20260820
META,wifi_dump_bytes,0
META,bt_dump_bytes,0
META,nfc_dump_bytes,0
FEAT,android.hardware.wifi,present
FEAT,android.hardware.wifi.direct,absent
FEAT,android.hardware.wifi.aware,absent
FEAT,android.hardware.wifi.passpoint,absent
FEAT,android.hardware.bluetooth,present
FEAT,android.hardware.nfc,present
FEAT,android.hardware.telephony,present
MOD,wifi,mounted
MOD,bluetooth,mounted
SET,global/bluetooth_on,0
SET,global/wifi_on,0
SET,secure/nfc_on,0
SET,secure/wifi_scan_always_available,0
SET,global/airplane_mode_on,1
COUNT,wifi_networks_total,0
COUNT,wifi_networks_unknown_kmgmt,0
COUNT,wifi_networks_none,0
COUNT,wifi_networks_wep,0
COUNT,wifi_networks_psk,0
COUNT,wifi_networks_sae,0
COUNT,wifi_networks_owe,0
COUNT,wifi_networks_eap,0
COUNT,wifi_tkip_marker_lines,0
ROW,L0-LOW,wifi_dump_missing,未采到 dumpsys wifi：Wi-Fi 侧全部判定不成立，不得读成没有开放网络
ROW,L1-INFO,wifi_p2p_absent,未声明 Wi-Fi Direct 特性：近场直连通道在本设备不存在
ROW,L1-INFO,wifi_module_mounted,com.android.wifi 已挂载：框架侧 Wi-Fi 栈存在独立于整机 OTA 的更新通道
ROW,L1-INFO,bt_radio_off,bluetooth_on=0：蓝牙射频关闭；该行可靠性取决于这一 settings 键在本设备可读
ROW,L0-LOW,bt_dump_missing,未采到 dumpsys bluetooth_manager：蓝牙侧判定不成立
ROW,L1-INFO,bt_module_mounted,com.google.android.bt 已挂载：蓝牙栈存在独立于整机 OTA 的更新通道
ROW,L1-INFO,nfc_off,nfc_on=0：NFC 关闭
ROW,L0-LOW,nfc_dump_missing,未采到 dumpsys nfc：锁屏卡模拟与路由侧判定不成立
ROW,L1-INFO,baseband_version_read,gsm.version.baseband 可读：这是设备侧唯一能把蜂窝侧变更归因到具体固件版本的字段
ROW,L1-INFO,bootloader_version_read,ro.boot.bootloader 可读：可比对随包分发的引导与调制解调器组件版本
ROW,L1-INFO,wifi_scan_always_off,wifi_scan_always_available=0：Wi-Fi 关闭时不扫描
ROW,L1-INFO,airplane_mode_on,airplane_mode_on=1：整机无线处于关闭口径，逐项射频状态仍需单独复核
ROW,L1-INFO,spl_read,security_patch=2026-10-05：只作为本快照的时间锚点，不据以判定无线侧缺陷是否已修
ROW,L1-INFO,build_type_user,ro.build.type=user：发行构建
SUMMARY,high=0,warn=0,gap=3,info=11,subsystems=4/4,nets=0,model=GoodPhone X1,spl=2026-10-05
```
---

## 廿二、脚本 2 的设计：离线差分，不接触设备

```bash
#!/usr/bin/env bash
# 脚本 2: wireless-snapshot-drift.sh — 两次无线侧姿态快照的离线漂移比对（只读）
# 用法: wireless-snapshot-drift.sh <基线快照> <当前快照>
# 输入是脚本 1 的输出文本；本脚本不接触设备、不访问网络，可用纯文本文件离线复核
set -u
base="${1:?用法: wireless-snapshot-drift.sh <基线快照> <当前快照>}"
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
field()  { awk -F, -v k="$2" '/^SUMMARY,/ { for (i = 2; i <= NF; i++) if ($i ~ "^" k "=") { sub("^" k "=", "", $i); print $i; exit } }' "$1"; }
metav()  { awk -F, -v k="$2" '$1 == "META" && $2 == k { print $3; exit }' "$1"; }
counts() { awk -F, -v k="$2" '$1 == "COUNT" && $2 == k { print $3; exit }' "$1"; }
rowsig() { awk -F, -v p="$2" '$1 == "ROW" && $2 ~ "^" p { print $3 }' "$1" | sort -u; }
gcs()    { awk -F, '$1 == "ROW" && $2 == "L0-LOW" { print $3 }' "$1" | sort -u; }
setlines() { awk -F, '$1 == "SET" { print $2 "=" $3 }' "$1" | sort; }
featlines() { awk -F, '$1 == "FEAT" { print $2 "=" $3 }' "$1" | sort; }
modlines()  { awk -F, '$1 == "MOD"  { print $2 "=" $3 }' "$1" | sort; }
getof()    { printf '%s\n' "$2" | grep "^$1=" | head -1 | cut -d= -f2-; }
nn()       { case "$1" in ''|none) echo none ;; *) echo "$1" ;; esac; }

b_rows="$(grep -c '^ROW,' "$base" || true)"; c_rows="$(grep -c '^ROW,' "$curr" || true)"
if [ "${b_rows:-0}" -eq 0 ] && [ "${c_rows:-0}" -eq 0 ]; then
  echo "ERR,input_no_snapshot_row,$base|$curr" >&2
  exit 1
fi

b_model="$(metav "$base" model)"; c_model="$(metav "$curr" model)"
b_spl="$(metav "$base" security_patch)"; c_spl="$(metav "$curr" security_patch)"
b_bb="$(metav "$base" baseband_prop)"; c_bb="$(metav "$curr" baseband_prop)"
b_type="$(metav "$base" build_type)"; c_type="$(metav "$curr" build_type)"
printf 'PAIR,base,%s,rows=%s,model=%s,spl=%s\n' "$base" "${b_rows:-0}" "${b_model:-none}" "${b_spl:-none}"
printf 'PAIR,curr,%s,rows=%s,model=%s,spl=%s\n' "$curr" "${c_rows:-0}" "${c_model:-none}" "${c_spl:-none}"

# ---------- 同源性 ----------
if [ "$(nn "$b_model")" = none ] || [ "$(nn "$c_model")" = none ]; then
  row L0-LOW pair_identity_unread "至少一份快照没有机型标识：这两份产物无法确认为同一设备线，差分结果只做参考"
elif [ "$b_model" != "$c_model" ]; then
  row L2-WARN pair_identity_mismatch "基线机型 $b_model 与当前机型 $c_model 不同：本差分跨设备，任何差异都不能读成一次升级或收紧"
else
  row L1-INFO pair_identity_same_model "两份快照机型一致（$b_model）：仍要由操作者确认它们来自同一台设备与同一条采集流程"
fi
printf 'NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录\n'
if [ "$b_type" != "$c_type" ]; then
  printf 'DELTA,build_type,%s,%s\n' "${b_type:-none}" "${c_type:-none}"
  row L2-WARN build_type_changed "构建形态由 ${b_type:-未采到} 变为 ${c_type:-未采到}：无线侧调试通道的默认状态随构建形态改变，非发行构建与发行构建不是同一条基线"
fi

# ---------- 信号码集合差分 ----------
b_l3="$(rowsig "$base" 'L3-')"; c_l3="$(rowsig "$curr" 'L3-')"
b_l2="$(rowsig "$base" 'L2-')"; c_l2="$(rowsig "$curr" 'L2-')"
new_l3="$(comm -13 <(printf '%s\n' "$b_l3") <(printf '%s\n' "$c_l3") | grep '[^[:space:]]' || true)"
new_l2="$(comm -13 <(printf '%s\n' "$b_l2") <(printf '%s\n' "$c_l2") | grep '[^[:space:]]' || true)"
gone_l3="$(comm -23 <(printf '%s\n' "$b_l3") <(printf '%s\n' "$c_l3") | grep '[^[:space:]]' || true)"
gone_l2="$(comm -23 <(printf '%s\n' "$b_l2") <(printf '%s\n' "$c_l2") | grep '[^[:space:]]' || true)"
for x in $new_l3; do row L3-HIGH risk_new "当前快照新增 L3 信号码 $x：按整改单处理，附上两份输入快照的路径"; done
for x in $new_l2; do row L2-WARN risk_new_warn "当前快照新增 L2 信号码 $x：先分辨这是配置变更还是策略放宽"; done

# ---------- "消失"能不能读成"已修" ----------
b_gap="$(field "$base" gap)"; c_gap="$(field "$curr" gap)"
cov_drop=no
if [ "${c_gap:-0}" -gt "${b_gap:-0}" ]; then cov_drop=yes; fi
for x in $gone_l3; do
  if [ "$cov_drop" = yes ]; then
    row L3-HIGH risk_gone_by_coverage "基线 L3 信号码 $x 在当前不再出现，但当前快照缺口数由 ${b_gap:-?} 变为 ${c_gap:-?}：这一消失应读作采集面收窄，不能读成已修"
  else
    row L2-WARN risk_closed_needs_proof "基线 L3 信号码 $x 在当前消失且采集面未收窄：仍需一条变更依据（配置单、更新记录或公告条目）才能记为闭合"
  fi
done
for x in $gone_l2; do
  if [ "$cov_drop" = yes ]; then
    row L2-WARN risk_gone_by_coverage "基线 L2 信号码 $x 在当前不再出现，但当前快照缺口数由 ${b_gap:-?} 变为 ${c_gap:-?}：同一批判据里有新缺口，先排除采集退化"
  else
    row L1-INFO warn_resolved "基线 L2 信号码 $x 在当前消失且采集面未收窄：可记为待复核的改进项"
  fi
done
gone_all="$(printf '%s\n%s\n' "$gone_l3" "$gone_l2" | grep '[^[:space:]]' || true)"
printf 'COUNT,gone_codes,%s\n' "$(printf '%s\n' "$gone_all" | grep -c '[^[:space:]]' || true)"

# ---------- 缺口是否长期未闭合 ----------
b_g="$(gcs "$base")"; c_g="$(gcs "$curr")"
same_gap="$(comm -12 <(printf '%s\n' "$b_g") <(printf '%s\n' "$c_g") | grep '[^[:space:]]' || true)"
if [ -n "$same_gap" ]; then
  n="$(printf '%s\n' "$same_gap" | grep -c '[^[:space:]]')"
  row L0-LOW gap_persistent "$n 个采集项在两份快照里都缺（$(printf '%s ' $same_gap)）：连续两次没采到，是流程缺口而不是偶发失败"
fi
new_gap="$(comm -13 <(printf '%s\n' "$b_g") <(printf '%s\n' "$c_g") | grep '[^[:space:]]' || true)"
[ -n "$new_gap" ] && row L2-WARN coverage_drop_new "当前快照新增缺口项：$(printf '%s ' $new_gap)"
gone_gap="$(comm -23 <(printf '%s\n' "$b_g") <(printf '%s\n' "$c_g") | grep '[^[:space:]]' || true)"
[ -n "$gone_gap" ] && row L1-INFO coverage_restored "原先缺失的采集项在本次恢复：$(printf '%s ' $gone_gap)"

# ---------- 数量型事实 ----------
for spec in \
  "wifi_networks_none 无密钥管理网络数" \
  "wifi_networks_wep WEP网络数" \
  "wifi_networks_psk PSK网络数" \
  "wifi_networks_sae SAE网络数" \
  "wifi_networks_owe OWE网络数" \
  "wifi_networks_eap EAP网络数" \
  "wifi_networks_unknown_kmgmt 密钥管理未知网络数" \
  "wifi_tkip_marker_lines TKIP记号行数" \
  "bt_bond_marker_lines 蓝牙绑定记号行数"; do
  set -- $spec
  k="$1"; n="$2"
  bv="$(counts "$base" "$k")"; cv="$(counts "$curr" "$k")"
  [ -z "$bv" ] && bv=na; [ -z "$cv" ] && cv=na
  printf 'DELTA,%s,%s,%s\n' "$k" "$bv" "$cv"
  if [ "$bv" = na ] || [ "$cv" = na ]; then
    row L0-LOW delta_unread "$n（$k）在至少一份快照里没有计数：本次不判方向"
  elif [ "$bv" != "$cv" ]; then
    case "$k" in
      wifi_networks_none|wifi_networks_wep|wifi_tkip_marker_lines)
        if [ "$cv" -gt "$bv" ]; then row L3-HIGH open_surface_up "$n 由 $bv 增至 $cv：无保护或破损加密的配置面扩大"
        else row L1-INFO open_surface_down "$n 由 $bv 降至 $cv：下降需要一条变更依据支撑"; fi ;;
      wifi_networks_sae|wifi_networks_owe|wifi_networks_eap)
        if [ "$cv" -lt "$bv" ]; then row L3-HIGH strong_auth_down "$n 由 $bv 降至 $cv：强认证配置减少，这是降级方向"
        else row L1-INFO strong_auth_up "$n 由 $bv 增至 $cv：强认证配置扩大"; fi ;;
      wifi_networks_unknown_kmgmt)
        row L2-WARN unread_kmgmt_change "密钥管理未知网络数由 $bv 变为 $cv：这一批网络在相应那份快照里不可判级，方向本身不构成结论" ;;
      *)
        row L1-INFO counter_change "$n 由 $bv 变为 $cv：这是记号行数，不等同于设备台数" ;;
    esac
  fi
done

# ---------- 状态位翻转 ----------
b_set="$(setlines "$base")"; c_set="$(setlines "$curr")"
for pair in $(printf '%s\n%s\n' "$b_set" "$c_set" | sed 's/=.*$//' | grep '[^[:space:]]' | sort -u); do
  bv="$(getof "$pair" "$b_set")"; cv="$(getof "$pair" "$c_set")"
  [ -z "$bv" ] && bv=absent; [ -z "$cv" ] && cv=absent
  printf 'SETDIFF,%s,%s,%s\n' "$pair" "$bv" "$cv"
  [ "$bv" = "$cv" ] && continue
  case "$pair" in
    secure/wifi_scan_always_available)
      if [ "$bv" = 0 ] && [ "$cv" = 1 ]; then row L2-WARN setting_scan_enabled "关网扫描位由 0 翻为 1：Wi-Fi 关闭时的扫描通道被打开"
      elif [ "$bv" = 1 ] && [ "$cv" = 0 ]; then row L1-INFO setting_scan_restricted "关网扫描位由 1 翻为 0：通道被关闭，记录为改进项"
      else row L0-LOW setting_state_unread "关网扫描位由 $bv 变为 $cv：至少一份没读到，不判方向"; fi ;;
    global/bluetooth_on|secure/nfc_on)
      if [ "$bv" = none ] || [ "$cv" = none ] || [ "$bv" = absent ] || [ "$cv" = absent ]; then
        row L0-LOW setting_state_unread "$pair 由 $bv 变为 $cv：至少一份没读到该键，不判方向"
      else
        row L1-INFO setting_radio_flip "$pair 由 $bv 变为 $cv：射频开关状态变化，不改变既有绑定与凭据"
      fi ;;
    *)
      if [ "$bv" = none ] || [ "$cv" = none ] || [ "$bv" = absent ] || [ "$cv" = absent ]; then
        row L0-LOW setting_state_unread "$pair 由 $bv 变为 $cv：至少一份没读到该键，不判方向"
      else
        row L2-WARN setting_flip "$pair 由 $bv 变为 $cv：状态位翻转，需与变更单对齐"
      fi ;;
  esac
done

# ---------- 特性面与更新通道 ----------
b_ft="$(featlines "$base")"; c_ft="$(featlines "$curr")"
for k in $(printf '%s\n%s\n' "$b_ft" "$c_ft" | sed 's/=.*$//' | grep '[^[:space:]]' | sort -u); do
  ov="$(getof "$k" "$b_ft")"; cv="$(getof "$k" "$c_ft")"
  [ -z "$ov" ] && ov=absent; [ -z "$cv" ] && cv=absent
  [ "$ov" = "$cv" ] && continue
  printf 'FEATDIFF,%s,%s,%s\n' "$k" "$ov" "$cv"
  case "$cv" in
    unknown) row L0-LOW feat_now_unread "$k 由 $ov 变为读不到：这一项在当次采集失败，不能读成特性被移除" ;;
    absent)  row L2-WARN feat_removed "$k 由 $ov 变为 absent：子系统面消失或不再声明，核对是硬件裁剪还是采集差异" ;;
    present) row L1-INFO feat_added "$k 由 $ov 变为 present：新的无线子系统面进入清点范围，需要重跑一轮判级" ;;
  esac
done
b_md="$(modlines "$base")"; c_md="$(modlines "$curr")"
for k in $(printf '%s\n%s\n' "$b_md" "$c_md" | sed 's/=.*$//' | grep '[^[:space:]]' | sort -u); do
  ov="$(getof "$k" "$b_md")"; cv="$(getof "$k" "$c_md")"
  [ -z "$ov" ] && ov=unknown; [ -z "$cv" ] && cv=unknown
  [ "$ov" = "$cv" ] && continue
  printf 'MODDIFF,%s,%s,%s\n' "$k" "$ov" "$cv"
  if [ "$cv" = mounted ] && [ "$ov" != mounted ]; then
    row L1-INFO module_channel_added "$k 侧更新通道由 $ov 变为已挂载：这一层现在可以不等整机 OTA 单独更新"
  elif [ "$cv" = absent ] && [ "$ov" = mounted ]; then
    row L2-WARN module_channel_lost "$k 侧更新通道由已挂载变为不在清单里：这一层的修复现在只能随整机镜像到达"
  else
    row L0-LOW module_channel_unread "$k 侧通道状态由 $ov 变为 $cv：不判定，只记录形态变化"
  fi
done

# ---------- 归因证据 ----------
if [ "$b_spl" != "$c_spl" ]; then
  if [ "$(nn "$b_spl")" = none ] || [ "$(nn "$c_spl")" = none ]; then
    row L0-LOW spl_change_unread "安全补丁级别由 ${b_spl:-未采到} 变为 ${c_spl:-未采到}：其中一份没读到，不判定方向"
  else
    row L1-INFO spl_moved "安全补丁级别由 $b_spl 变为 $c_spl：这是设备侧最可靠的时间锚点，无线侧缺陷是否已修仍需按公告逐项核对"
  fi
fi
if [ "$b_bb" != "$c_bb" ]; then
  if [ "$(nn "$b_bb")" = none ] || [ "$(nn "$c_bb")" = none ]; then
    row L0-LOW baseband_evidence_unread "基带版本由 ${b_bb:-未采到} 变为 ${c_bb:-未采到}：至少一份读不到，蜂窝侧变更无法在设备上归因"
  else
    row L1-INFO baseband_version_moved "基带版本由 $b_bb 变为 $c_bb：这是本差分里唯一能直接指向固件侧变更的证据，需向厂商索取该版本对应的公告条目"
  fi
fi
b_hal="$(awk -F, '$1 == "ROW" && ($3 == "wifi_hal_iface_aidl" || $3 == "wifi_hal_iface_hidl") { print $3 }' "$base" | head -1)"
c_hal="$(awk -F, '$1 == "ROW" && ($3 == "wifi_hal_iface_aidl" || $3 == "wifi_hal_iface_hidl") { print $3 }' "$curr" | head -1)"
if [ "$b_hal" != "$c_hal" ]; then
  if [ -n "$c_hal" ] && [ -z "$b_hal" ]; then
    row L1-INFO hal_iface_first_read "HAL 接口语言记号首次读到（$c_hal）：基线里这一项是缺口，本次只是补全证据"
  elif [ -z "$c_hal" ] && [ -n "$b_hal" ]; then
    row L0-LOW hal_iface_lost "HAL 接口语言记号由 $b_hal 变为读不到：不判定接口回退，先补采集"
  else
    case "$c_hal" in
      wifi_hal_iface_aidl) row L1-INFO hal_iface_modernized "HAL 接口语言记号由 $b_hal 变为 AIDL：接口形态向稳定 AIDL 迁移" ;;
      *)                   row L2-WARN hal_iface_regressed "HAL 接口语言记号由 $b_hal 变为 HIDL：向被公开口径标注为弃用的接口语言回退" ;;
    esac
  fi
fi

# ---------- 全等判定 ----------
if [ "${b_rows:-0}" -gt 0 ] && [ "$b_rows" = "$c_rows" ] && \
   diff -q <(grep '^ROW,' "$base") <(grep '^ROW,' "$curr") >/dev/null 2>&1 && \
   diff -q <(grep '^COUNT,' "$base") <(grep '^COUNT,' "$curr") >/dev/null 2>&1; then
  row L1-INFO no_drift "两份快照的判定行与计数行完全一致：本次没有可报告的漂移；注意缺口项同样没有变化"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,base_rows=%s,curr_rows=%s,new_l3=%s,new_l2=%s,gone=%s,cov_drop=%s\n' \
  "$hi" "$warn" "$gap" "$info" "${b_rows:-0}" "${c_rows:-0}" \
  "$(printf '%s\n' "$new_l3" | grep -c '[^[:space:]]' || true)" \
  "$(printf '%s\n' "$new_l2" | grep -c '[^[:space:]]' || true)" \
  "$(printf '%s\n' "$gone_all" | grep -c '[^[:space:]]' || true)" \
  "$cov_drop"
```

**为什么第二个脚本不做设备采集。** 判定沙箱或无线姿态"被改动过"的唯一可靠输入是两份留存快照的差分层；设备侧一次采集给不出方向。因此脚本 2 的两个参数都是**文本文件路径**——它可以完全不依赖 adb、在离线机器上复核一份历史快照，这一点对流程审计很重要：复核动作不应该要求当时那台设备还在。

**输入契约（严格由脚本 1 的输出定义）：** 脚本 2 只读四类行——`ROW,`（信号码集合差分的主输入）、`COUNT,`（数量方向判定）、`META,`（同源性与归因锚点）、`SET,` / `FEAT,` / `MOD,`（状态位、特性面、模块通道三条差分线）。它对空文件与不含任何 `ROW,` 行的输入直接以 `ERR,input_no_snapshot_row,…` 退出，不做"空快照等于无风险"的处理。

**输出行种类：**

| 行首 | 含义 |
| --- | --- |
| `PAIR,<base\|curr>,<路径>,rows=,model=,spl=` | 两份输入的身份与规模，第一行就交代差分对象的来历 |
| `NOTE,identity,…` | 一条固定提醒：快照内不含构建号与采集时间戳，同源性由操作者自行记录 |
| `DELTA,<键>,<旧>,<新>` | 计数与属性变化的原始记录（不做判定，供审计对齐变更单） |
| `SETDIFF,<表/键>,<旧>,<新>` / `FEATDIFF` / `MODDIFF` | 三条差分线的原始记录 |
| `ROW,<等级>,<信号码>,<说明>` | 判定行，共 39 个码位 |
| `COUNT,gone_codes,<数>` | 消失码位总数，供人工核对 |
| `SUMMARY,…` | 汇总：`high/warn/gap/info` + `base_rows/curr_rows/new_l3/new_l2/gone/cov_drop` |

**五个设计决定：**

1. **同源性先于一切判定。** 机型不同 → `pair_identity_mismatch`(L2)，并在说明里写"任何差异都不能读成一次升级或收紧"；机型读不到 → `pair_identity_unread`(L0)。同时单独比对 `ro.build.type`，非发行与发行构建不是同一条基线（`build_type_changed`, L2）。
2. **信号码集合差分用集合代数而不是文本 diff。** `comm -13` 取新增、`comm -23` 取消失，主键是信号码字符串，因此同一码位在两份快照里说明文字数量不同不会被误判为变化。
3. **"消失"必须先看采集面。** `cov_drop` 定义为"当前快照的 `gap` 计数大于基线"，它把全部消失码位分成 `risk_gone_by_coverage`(L3) 与 `risk_closed_needs_proof`(L2) 两支——第十八章纪律二的实现，样例三是它的反例演示。
4. **数量方向有语义，不是绝对值。** 无保护/破损类增加与强认证类减少才是 L3；反向变化只给 L1 并要求"一条变更依据支撑"，不自动记成果。`unknown_kmgmt` 的变化只判 L2 并说明"方向本身不构成结论"。
5. **读不到一律不判方向。** 所有 `*_unread` 码位（`delta_unread`、`setting_state_unread`、`feat_now_unread`、`module_channel_unread`、`spl_change_unread`、`baseband_evidence_unread`、`hal_iface_lost`、`pair_identity_unread`）都是 L0，且说明句都写成"不判定"。脚本 2 里出现次数最多的等级是 L0，这是设计目标而不是缺陷。

**它不做的事：** 不连接设备、不访问网络、不修改输入文件、不做任何加权评分、不给跨设备的横向比较（第十八章"等级不跨设备比较"）。`diff -q` 只用于最后的全等判定（`no_drift`）。

---

## 廿三、脚本 2 的码表与样例输出（三份主样例 + 四份补充）

**39 个信号码的分组表：**

| 组 | 码位（等级） |
| --- | --- |
| 同源性与构建形态（4） | `pair_identity_same_model`(L1)、`pair_identity_mismatch`(L2)、`pair_identity_unread`(L0)、`build_type_changed`(L2) |
| 新增风险（2） | `risk_new`(L3)、`risk_new_warn`(L2) |
| 消失的两种读法（3） | `risk_gone_by_coverage`(L3)、`risk_closed_needs_proof`(L2)、`warn_resolved`(L1) |
| 缺口是否长期未闭合（3） | `gap_persistent`(L0)、`coverage_drop_new`(L2)、`coverage_restored`(L1) |
| 数量方向（7） | `open_surface_up`(L3)、`open_surface_down`(L1)、`strong_auth_down`(L3)、`strong_auth_up`(L1)、`unread_kmgmt_change`(L2)、`counter_change`(L1)、`delta_unread`(L0) |
| 状态位翻转（5） | `setting_scan_enabled`(L2)、`setting_scan_restricted`(L1)、`setting_radio_flip`(L1)、`setting_flip`(L2)、`setting_state_unread`(L0) |
| 特性面（3） | `feat_added`(L1)、`feat_removed`(L2)、`feat_now_unread`(L0) |
| 模块更新通道（3） | `module_channel_added`(L1)、`module_channel_lost`(L2)、`module_channel_unread`(L0) |
| 归因证据（4） | `spl_moved`(L1)、`spl_change_unread`(L0)、`baseband_version_moved`(L1)、`baseband_evidence_unread`(L0) |
| HAL 接口语言（4） | `hal_iface_modernized`(L1)、`hal_iface_regressed`(L2)、`hal_iface_first_read`(L1)、`hal_iface_lost`(L0) |
| 全等（1） | `no_drift`(L1) |

**样例一：一次真实的劣化（`phone-a-2026-10` → `phone-a-2026-12`，`high=4,warn=6,cov_drop=no`）**

这是"同一台设备、采集面未收窄、风险确实新增"的正例，也是脚本 2 最有价值的一种输出：2 条 `risk_new`(L3) 与 4 条 `risk_new_warn`(L2) 成对出现，`open_surface_up`(L3) 与 `strong_auth_down`(L3) 从计数方向再次确认同一件事，`setting_scan_enabled`(L2) 给出关网扫描位由 0 翻 1，`feat_removed`(L2) 记录一个特性面不再声明。**关键在于 `cov_drop=no`**：正因为缺口数没有增加，这一批新增与变化才可读作真实变更；`spl_moved` 与 `baseband_version_moved` 两条 L1 提供了归因锚点，但说明句仍写"无线侧缺陷是否已修需按公告逐项核对"。

```text
PAIR,base,phone-a-2026-10.txt,rows=19,model=GoodPhone X1,spl=2026-10-05
PAIR,curr,phone-a-2026-12.txt,rows=21,model=GoodPhone X1,spl=2026-12-05
ROW,L1-INFO,pair_identity_same_model,两份快照机型一致（GoodPhone X1）：仍要由操作者确认它们来自同一台设备与同一条采集流程
NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录
ROW,L3-HIGH,risk_new,当前快照新增 L3 信号码 net_autojoin_open：按整改单处理，附上两份输入快照的路径
ROW,L3-HIGH,risk_new,当前快照新增 L3 信号码 net_open：按整改单处理，附上两份输入快照的路径
ROW,L2-WARN,risk_new_warn,当前快照新增 L2 信号码 bt_discoverable：先分辨这是配置变更还是策略放宽
ROW,L2-WARN,risk_new_warn,当前快照新增 L2 信号码 bt_legacy_pairing_marker：先分辨这是配置变更还是策略放宽
ROW,L2-WARN,risk_new_warn,当前快照新增 L2 信号码 net_psk_only：先分辨这是配置变更还是策略放宽
ROW,L2-WARN,risk_new_warn,当前快照新增 L2 信号码 wifi_scan_always_on：先分辨这是配置变更还是策略放宽
COUNT,gone_codes,0
DELTA,wifi_networks_none,0,1
ROW,L3-HIGH,open_surface_up,无密钥管理网络数 由 0 增至 1：无保护或破损加密的配置面扩大
DELTA,wifi_networks_wep,0,0
DELTA,wifi_networks_psk,1,2
ROW,L1-INFO,counter_change,PSK网络数 由 1 变为 2：这是记号行数，不等同于设备台数
DELTA,wifi_networks_sae,1,0
ROW,L3-HIGH,strong_auth_down,SAE网络数 由 1 降至 0：强认证配置减少，这是降级方向
DELTA,wifi_networks_owe,1,1
DELTA,wifi_networks_eap,1,1
DELTA,wifi_networks_unknown_kmgmt,0,0
DELTA,wifi_tkip_marker_lines,0,0
DELTA,bt_bond_marker_lines,3,4
ROW,L1-INFO,counter_change,蓝牙绑定记号行数 由 3 变为 4：这是记号行数，不等同于设备台数
SETDIFF,global/airplane_mode_on,0,0
SETDIFF,global/bluetooth_on,1,1
SETDIFF,global/wifi_on,1,1
SETDIFF,secure/nfc_on,1,1
SETDIFF,secure/wifi_scan_always_available,0,1
ROW,L2-WARN,setting_scan_enabled,关网扫描位由 0 翻为 1：Wi-Fi 关闭时的扫描通道被打开
FEATDIFF,android.hardware.wifi.passpoint,present,absent
ROW,L2-WARN,feat_removed,android.hardware.wifi.passpoint 由 present 变为 absent：子系统面消失或不再声明，核对是硬件裁剪还是采集差异
ROW,L1-INFO,spl_moved,安全补丁级别由 2026-10-05 变为 2026-12-05：这是设备侧最可靠的时间锚点，无线侧缺陷是否已修仍需按公告逐项核对
ROW,L1-INFO,baseband_version_moved,基带版本由 B2.1.4-20260912 变为 B2.2.0-20261120：这是本差分里唯一能直接指向固件侧变更的证据，需向厂商索取该版本对应的公告条目
SUMMARY,high=4,warn=6,gap=0,info=5,base_rows=19,curr_rows=21,new_l3=2,new_l2=4,gone=0,cov_drop=no
```

**样例二：整改后的复核（`phone-b-2026-02` → `phone-b-2026-10`，`gone=12,cov_drop=no`）**

这一份演示"消失"的正确读法。基线的 3 条 L3 与 9 条 L2 在当前全部消失，缺口数没有增加（`gap` 由 5 变 0，`coverage_restored` 记录恢复），因此三条 L3 走的是 `risk_closed_needs_proof`(L2)——**脚本明确拒绝直接记为"已修"**，要求补一条变更依据（配置单、更新记录或公告条目）。其余 9 条走 `warn_resolved`(L1)。方向类判定同时给出证据：`open_surface_down` ×3、`strong_auth_up` ×3、`hal_iface_modernized`（HIDL→AIDL）、`module_channel_added` ×2（两条更新通道出现）、`setting_scan_restricted`。`build_type_changed`(L2) 提醒这次整改同时换了构建形态，严格说这是两条不同基线。另有一处值得注意：`baseband_evidence_unread`(L0)——基带版本在整改后读不到了，蜂窝侧变更因此不可归因，这一条不能被前面一堆改进行"平均"掉。

```text
PAIR,base,phone-b-2026-02.txt,rows=22,model=BudgetPhone B9,spl=2024-02-05
PAIR,curr,phone-b-2026-10.txt,rows=18,model=BudgetPhone B9,spl=2026-10-05
ROW,L1-INFO,pair_identity_same_model,两份快照机型一致（BudgetPhone B9）：仍要由操作者确认它们来自同一台设备与同一条采集流程
NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录
DELTA,build_type,userdebug,user
ROW,L2-WARN,build_type_changed,构建形态由 userdebug 变为 user：无线侧调试通道的默认状态随构建形态改变，非发行构建与发行构建不是同一条基线
ROW,L2-WARN,risk_closed_needs_proof,基线 L3 信号码 net_autojoin_open 在当前消失且采集面未收窄：仍需一条变更依据（配置单、更新记录或公告条目）才能记为闭合
ROW,L2-WARN,risk_closed_needs_proof,基线 L3 信号码 net_open 在当前消失且采集面未收窄：仍需一条变更依据（配置单、更新记录或公告条目）才能记为闭合
ROW,L2-WARN,risk_closed_needs_proof,基线 L3 信号码 net_wep 在当前消失且采集面未收窄：仍需一条变更依据（配置单、更新记录或公告条目）才能记为闭合
ROW,L1-INFO,warn_resolved,基线 L2 信号码 bt_discoverable 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 bt_legacy_pairing_marker 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 build_type_nonuser 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 net_psk_only 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 net_tkip 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 nfc_lockscreen_emulation 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 nfc_secure_disabled 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 wifi_hal_iface_hidl 在当前消失且采集面未收窄：可记为待复核的改进项
ROW,L1-INFO,warn_resolved,基线 L2 信号码 wifi_scan_always_on 在当前消失且采集面未收窄：可记为待复核的改进项
COUNT,gone_codes,12
ROW,L1-INFO,coverage_restored,原先缺失的采集项在本次恢复：baseband_version_absent bootloader_version_absent bt_module_absent net_kmgmt_unread wifi_module_absent 
DELTA,wifi_networks_none,2,0
ROW,L1-INFO,open_surface_down,无密钥管理网络数 由 2 降至 0：下降需要一条变更依据支撑
DELTA,wifi_networks_wep,1,0
ROW,L1-INFO,open_surface_down,WEP网络数 由 1 降至 0：下降需要一条变更依据支撑
DELTA,wifi_networks_psk,1,1
DELTA,wifi_networks_sae,0,1
ROW,L1-INFO,strong_auth_up,SAE网络数 由 0 增至 1：强认证配置扩大
DELTA,wifi_networks_owe,0,1
ROW,L1-INFO,strong_auth_up,OWE网络数 由 0 增至 1：强认证配置扩大
DELTA,wifi_networks_eap,0,1
ROW,L1-INFO,strong_auth_up,EAP网络数 由 0 增至 1：强认证配置扩大
DELTA,wifi_networks_unknown_kmgmt,1,0
ROW,L2-WARN,unread_kmgmt_change,密钥管理未知网络数由 1 变为 0：这一批网络在相应那份快照里不可判级，方向本身不构成结论
DELTA,wifi_tkip_marker_lines,1,0
ROW,L1-INFO,open_surface_down,TKIP记号行数 由 1 降至 0：下降需要一条变更依据支撑
DELTA,bt_bond_marker_lines,3,3
SETDIFF,global/airplane_mode_on,0,0
SETDIFF,global/bluetooth_on,1,1
SETDIFF,global/wifi_on,1,1
SETDIFF,secure/nfc_on,1,0
ROW,L1-INFO,setting_radio_flip,secure/nfc_on 由 1 变为 0：射频开关状态变化，不改变既有绑定与凭据
SETDIFF,secure/wifi_scan_always_available,1,0
ROW,L1-INFO,setting_scan_restricted,关网扫描位由 1 翻为 0：通道被关闭，记录为改进项
MODDIFF,bluetooth,absent,mounted
ROW,L1-INFO,module_channel_added,bluetooth 侧更新通道由 absent 变为已挂载：这一层现在可以不等整机 OTA 单独更新
MODDIFF,wifi,absent,mounted
ROW,L1-INFO,module_channel_added,wifi 侧更新通道由 absent 变为已挂载：这一层现在可以不等整机 OTA 单独更新
ROW,L1-INFO,spl_moved,安全补丁级别由 2024-02-05 变为 2026-10-05：这是设备侧最可靠的时间锚点，无线侧缺陷是否已修仍需按公告逐项核对
ROW,L0-LOW,baseband_evidence_unread,基带版本由 none 变为 B9.0.7-20260925：至少一份读不到，蜂窝侧变更无法在设备上归因
ROW,L1-INFO,hal_iface_modernized,HAL 接口语言记号由 wifi_hal_iface_hidl 变为 AIDL：接口形态向稳定 AIDL 迁移
SUMMARY,high=0,warn=5,gap=1,info=23,base_rows=22,curr_rows=18,new_l3=0,new_l2=0,gone=12,cov_drop=no
```

**样例三：采集面收窄伪装成整改（`phone-b-2026-02` → `phone-c-2026-10`，`cov_drop=yes`）**

先说明一处容易读错的字段：脚本 2 的 `SUMMARY` 里 `gap=19` 计的是**本差分自身**产生的 `L0-LOW` 行数，而 `cov_drop` 的比较对象是**两份输入快照**的缺口字段（本例为基线 `gap=5` 对当前 `gap=17`）。两者含义不同，判定时以后者为准——这也是脚本把 `cov_drop` 单独放进 `SUMMARY` 的原因。

这是全文最重要的一份输出。当前快照是第廿一章样例三那份"17 个缺口、0 条事实"的采集退化产物，脚本 2 的处理是把基线里每一条消失的高危码位判成 `risk_gone_by_coverage`(L3)——**等级不降反升**，说明句直接写"这一消失应读作采集面收窄，不能读成已修"。同时 `pair_identity_unread`(L0) 表明机型读不到、同源性无法确认；`delta_unread`、`setting_state_unread` ×5、`feat_now_unread` ×7、`module_channel_unread` ×2、`spl_change_unread`、`hal_iface_lost` 全部为 L0；`coverage_drop_new`(L2) 与 `gap_persistent`(L0) 分别指出新增缺口与长期未闭合缺口。**如果这份输出被误读成"高危清零"，一次真实的采集故障就成了整改成果**——第廿五章的 K9（缺口率上升则本批比率不可比）与 K10（连续两次未闭合）就是为这条防线设计的。

```text
PAIR,base,phone-b-2026-02.txt,rows=22,model=BudgetPhone B9,spl=2024-02-05
PAIR,curr,phone-c-2026-10.txt,rows=17,model=none,spl=none
ROW,L0-LOW,pair_identity_unread,至少一份快照没有机型标识：这两份产物无法确认为同一设备线，差分结果只做参考
NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录
DELTA,build_type,userdebug,none
ROW,L2-WARN,build_type_changed,构建形态由 userdebug 变为 none：无线侧调试通道的默认状态随构建形态改变，非发行构建与发行构建不是同一条基线
ROW,L3-HIGH,risk_gone_by_coverage,基线 L3 信号码 net_autojoin_open 在当前不再出现，但当前快照缺口数由 5 变为 17：这一消失应读作采集面收窄，不能读成已修
ROW,L3-HIGH,risk_gone_by_coverage,基线 L3 信号码 net_open 在当前不再出现，但当前快照缺口数由 5 变为 17：这一消失应读作采集面收窄，不能读成已修
ROW,L3-HIGH,risk_gone_by_coverage,基线 L3 信号码 net_wep 在当前不再出现，但当前快照缺口数由 5 变为 17：这一消失应读作采集面收窄，不能读成已修
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 bt_discoverable 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 bt_legacy_pairing_marker 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 build_type_nonuser 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 net_psk_only 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 net_tkip 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 nfc_lockscreen_emulation 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 nfc_secure_disabled 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 wifi_hal_iface_hidl 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
ROW,L2-WARN,risk_gone_by_coverage,基线 L2 信号码 wifi_scan_always_on 在当前不再出现，但当前快照缺口数由 5 变为 17：同一批判据里有新缺口，先排除采集退化
COUNT,gone_codes,12
ROW,L0-LOW,gap_persistent,2 个采集项在两份快照里都缺（baseband_version_absent bootloader_version_absent ）：连续两次没采到，是流程缺口而不是偶发失败
ROW,L2-WARN,coverage_drop_new,当前快照新增缺口项：airplane_mode_unread bt_dump_missing bt_feature_unknown bt_module_unread bt_state_unread build_type_absent cell_feature_unknown identity_missing nfc_dump_missing nfc_state_unread spl_absent wifi_dump_unparsed wifi_feature_unknown wifi_module_unread wifi_scan_state_unread 
ROW,L1-INFO,coverage_restored,原先缺失的采集项在本次恢复：bt_module_absent net_kmgmt_unread wifi_module_absent 
DELTA,wifi_networks_none,2,0
ROW,L1-INFO,open_surface_down,无密钥管理网络数 由 2 降至 0：下降需要一条变更依据支撑
DELTA,wifi_networks_wep,1,0
ROW,L1-INFO,open_surface_down,WEP网络数 由 1 降至 0：下降需要一条变更依据支撑
DELTA,wifi_networks_psk,1,0
ROW,L1-INFO,counter_change,PSK网络数 由 1 变为 0：这是记号行数，不等同于设备台数
DELTA,wifi_networks_sae,0,0
DELTA,wifi_networks_owe,0,0
DELTA,wifi_networks_eap,0,0
DELTA,wifi_networks_unknown_kmgmt,1,0
ROW,L2-WARN,unread_kmgmt_change,密钥管理未知网络数由 1 变为 0：这一批网络在相应那份快照里不可判级，方向本身不构成结论
DELTA,wifi_tkip_marker_lines,1,0
ROW,L1-INFO,open_surface_down,TKIP记号行数 由 1 降至 0：下降需要一条变更依据支撑
DELTA,bt_bond_marker_lines,3,na
ROW,L0-LOW,delta_unread,蓝牙绑定记号行数（bt_bond_marker_lines）在至少一份快照里没有计数：本次不判方向
SETDIFF,global/airplane_mode_on,0,none
ROW,L0-LOW,setting_state_unread,global/airplane_mode_on 由 0 变为 none：至少一份没读到该键，不判方向
SETDIFF,global/bluetooth_on,1,none
ROW,L0-LOW,setting_state_unread,global/bluetooth_on 由 1 变为 none：至少一份没读到该键，不判方向
SETDIFF,global/wifi_on,1,none
ROW,L0-LOW,setting_state_unread,global/wifi_on 由 1 变为 none：至少一份没读到该键，不判方向
SETDIFF,secure/nfc_on,1,none
ROW,L0-LOW,setting_state_unread,secure/nfc_on 由 1 变为 none：至少一份没读到该键，不判方向
SETDIFF,secure/wifi_scan_always_available,1,none
ROW,L0-LOW,setting_state_unread,关网扫描位由 1 变为 none：至少一份没读到，不判方向
FEATDIFF,android.hardware.bluetooth,present,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.bluetooth 由 present 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.nfc,present,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.nfc 由 present 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.telephony,present,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.telephony 由 present 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.wifi,present,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi 由 present 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.wifi.aware,absent,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi.aware 由 absent 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.wifi.direct,present,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi.direct 由 present 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.wifi.passpoint,absent,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi.passpoint 由 absent 变为读不到：这一项在当次采集失败，不能读成特性被移除
MODDIFF,bluetooth,absent,unknown
ROW,L0-LOW,module_channel_unread,bluetooth 侧通道状态由 absent 变为 unknown：不判定，只记录形态变化
MODDIFF,wifi,absent,unknown
ROW,L0-LOW,module_channel_unread,wifi 侧通道状态由 absent 变为 unknown：不判定，只记录形态变化
ROW,L0-LOW,spl_change_unread,安全补丁级别由 2024-02-05 变为 none：其中一份没读到，不判定方向
ROW,L0-LOW,hal_iface_lost,HAL 接口语言记号由 wifi_hal_iface_hidl 变为读不到：不判定接口回退，先补采集
SUMMARY,high=3,warn=12,gap=19,info=5,base_rows=22,curr_rows=17,new_l3=0,new_l2=0,gone=12,cov_drop=yes
```

**样例四：跨设备差分（同版本快照，`phone-a` vs `phone-a2`）**

两份快照除机型外完全一致，脚本给出 `pair_identity_mismatch`(L2) + `no_drift`(L1) 这一对组合：**内容相同不代表可以差分**。它的实用价值是在误传基线文件时立刻暴露（说明句写"本差分跨设备，任何差异都不能读成一次升级或收紧"），而不是一堆看起来合理的 `DELTA` 行。

```text
PAIR,base,phone-a-2026-10.txt,rows=19,model=GoodPhone X1,spl=2026-10-05
PAIR,curr,phone-a2-2026-10.txt,rows=19,model=GoodPhone X2,spl=2026-10-05
ROW,L2-WARN,pair_identity_mismatch,基线机型 GoodPhone X1 与当前机型 GoodPhone X2 不同：本差分跨设备，任何差异都不能读成一次升级或收紧
NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录
COUNT,gone_codes,0
DELTA,wifi_networks_none,0,0
DELTA,wifi_networks_wep,0,0
DELTA,wifi_networks_psk,1,1
DELTA,wifi_networks_sae,1,1
DELTA,wifi_networks_owe,1,1
DELTA,wifi_networks_eap,1,1
DELTA,wifi_networks_unknown_kmgmt,0,0
DELTA,wifi_tkip_marker_lines,0,0
DELTA,bt_bond_marker_lines,3,3
SETDIFF,global/airplane_mode_on,0,0
SETDIFF,global/bluetooth_on,1,1
SETDIFF,global/wifi_on,1,1
SETDIFF,secure/nfc_on,1,1
SETDIFF,secure/wifi_scan_always_available,0,0
ROW,L1-INFO,no_drift,两份快照的判定行与计数行完全一致：本次没有可报告的漂移；注意缺口项同样没有变化
SUMMARY,high=0,warn=1,gap=0,info=1,base_rows=19,curr_rows=19,new_l3=0,new_l2=0,gone=0,cov_drop=no
```

**样例五：接口语言回退 + 补丁倒退（`phone-a-2026-10` vs `phone-a-2026-10-hidl`）**

`hal_iface_regressed`(L2) 与 `strong_auth_down`(L3) 同时出现，`spl_moved` 的方向是 2026-10-05 → 2024-08-05。这里必须说明一处**设计上的诚实取舍**：`spl_moved` 是"时间锚点"语义而不是"改善/恶化"语义，所以补丁**倒退**时它仍判 L1 并只陈述变化。这是本文刻意的限制——脚本不猜测倒退的原因（重装旧镜像、降级、还是另一台设备的档案），原因判断属于操作者的变更单核对。读者若把这一行读成"SPL 变好"就是误用（第二十六章误区 8）。

```text
PAIR,base,phone-a-2026-10.txt,rows=19,model=GoodPhone X1,spl=2026-10-05
PAIR,curr,phone-a-2026-10-hidl.txt,rows=18,model=GoodPhone X1,spl=2024-08-05
ROW,L1-INFO,pair_identity_same_model,两份快照机型一致（GoodPhone X1）：仍要由操作者确认它们来自同一台设备与同一条采集流程
NOTE,identity,快照内不含构建号与采集时间戳，同源性由操作者自行记录
ROW,L2-WARN,risk_new_warn,当前快照新增 L2 信号码 wifi_hal_iface_hidl：先分辨这是配置变更还是策略放宽
COUNT,gone_codes,0
DELTA,wifi_networks_none,0,0
DELTA,wifi_networks_wep,0,0
DELTA,wifi_networks_psk,1,1
DELTA,wifi_networks_sae,1,1
DELTA,wifi_networks_owe,1,1
DELTA,wifi_networks_eap,1,0
ROW,L3-HIGH,strong_auth_down,EAP网络数 由 1 降至 0：强认证配置减少，这是降级方向
DELTA,wifi_networks_unknown_kmgmt,0,0
DELTA,wifi_tkip_marker_lines,0,0
DELTA,bt_bond_marker_lines,3,3
SETDIFF,global/airplane_mode_on,0,0
SETDIFF,global/bluetooth_on,1,1
SETDIFF,global/wifi_on,1,1
SETDIFF,secure/nfc_on,1,1
SETDIFF,secure/wifi_scan_always_available,0,0
ROW,L1-INFO,spl_moved,安全补丁级别由 2026-10-05 变为 2024-08-05：这是设备侧最可靠的时间锚点，无线侧缺陷是否已修仍需按公告逐项核对
ROW,L1-INFO,baseband_version_moved,基带版本由 B2.1.4-20260912 变为 Legacy-Old：这是本差分里唯一能直接指向固件侧变更的证据，需向厂商索取该版本对应的公告条目
ROW,L2-WARN,hal_iface_regressed,HAL 接口语言记号由 wifi_hal_iface_aidl 变为 HIDL：向被公开口径标注为弃用的接口语言回退
SUMMARY,high=1,warn=2,gap=0,info=3,base_rows=19,curr_rows=18,new_l3=0,new_l2=1,gone=0,cov_drop=no
```

**样例六：模块通道消失（`phone-d-2026-10` → `phone-e-2026-10`，节选判定行）**

这一对里当前快照采集退化，脚本 2 却仍给出 `module_channel_lost`(L2) ×2——因为 `/apex` 清单可读而模块不在清单里，与"清单读不到"（`module_channel_unread`, L0）是两件事。它演示了同一份退化快照里"可以判定的部分"和"不可以判定的部分"如何并存：

```text
FEATDIFF,android.hardware.wifi.direct,absent,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi.direct 由 absent 变为读不到：这一项在当次采集失败，不能读成特性被移除
FEATDIFF,android.hardware.wifi.passpoint,absent,unknown
ROW,L0-LOW,feat_now_unread,android.hardware.wifi.passpoint 由 absent 变为读不到：这一项在当次采集失败，不能读成特性被移除
MODDIFF,bluetooth,mounted,absent
ROW,L2-WARN,module_channel_lost,bluetooth 侧更新通道由已挂载变为不在清单里：这一层的修复现在只能随整机镜像到达
MODDIFF,wifi,mounted,absent
ROW,L2-WARN,module_channel_lost,wifi 侧更新通道由已挂载变为不在清单里：这一层的修复现在只能随整机镜像到达
ROW,L0-LOW,spl_change_unread,安全补丁级别由 2026-10-05 变为 none：其中一份没读到，不判定方向
ROW,L0-LOW,baseband_evidence_unread,基带版本由 B2.1.4-20260912 变为 none：至少一份读不到，蜂窝侧变更无法在设备上归因
SUMMARY,high=0,warn=8,gap=14,info=2,base_rows=14,curr_rows=21,new_l3=0,new_l2=1,gone=0,cov_drop=yes
```

**样例七：能力面新增与 HAL 记号首次读到（`phone-d-2026-10` → `phone-d-2026-12`，节选）**

`feat_added`(L1) ×2 与 `hal_iface_first_read`(L1) 都出现在"基线里是缺口、当前补全了证据"的情形——注意 `hal_iface_first_read` 明确写"本次只是补全证据"，不判方向。这一组码位存在的原因是：第一次读到某项证据时，任何"改善/回退"的判定都缺基线。

```text
ROW,L1-INFO,setting_radio_flip,secure/nfc_on 由 0 变为 1：射频开关状态变化，不改变既有绑定与凭据
SETDIFF,secure/wifi_scan_always_available,0,0
FEATDIFF,android.hardware.wifi.direct,absent,present
ROW,L1-INFO,feat_added,android.hardware.wifi.direct 由 absent 变为 present：新的无线子系统面进入清点范围，需要重跑一轮判级
FEATDIFF,android.hardware.wifi.passpoint,absent,present
ROW,L1-INFO,feat_added,android.hardware.wifi.passpoint 由 absent 变为 present：新的无线子系统面进入清点范围，需要重跑一轮判级
ROW,L1-INFO,hal_iface_first_read,HAL 接口语言记号首次读到（wifi_hal_iface_aidl）：基线里这一项是缺口，本次只是补全证据
SUMMARY,high=0,warn=2,gap=1,info=11,base_rows=14,curr_rows=19,new_l3=0,new_l2=0,gone=0,cov_drop=no
```

---

## 廿四、分支覆盖与脚本验收：104 个码位全部被触发过

内嵌脚本是报告的一部分，因此它按系列统一的验收标准处理：**每一条分支都必须被真实触发一次，触发不了就改脚本或加档案，而不是把没走过的分支留给读者**。本节给出可复现的验收过程。

**验收三件事：**

1. **语法**：`bash -n` 通过；无 `set -u` 冲突；两个脚本都只用 POSIX 兼容的 `awk` / `sed` / `grep` / `comm` / `sort` 组合，在 git-bash 与常见 Linux shell 下行为一致。
2. **抽取一致性**：发布文档里两段脚本按代码围栏行号重新抽取后与原脚本逐字节相同（`cmp`），四份脚本 1 样例与七份脚本 2 输出同样与运行产物逐字节相同——**样例不是手写的**。
3. **分支覆盖**：脚本 1 的 65 个码位、脚本 2 的 39 个码位，全部在合成档案上至少触发一次，合计 104 个。

**覆盖方法（可复现）**：从两个脚本正文里取出全部 `row` 调用得到码位全集，从各份输出里取出全部实际出现的码位，用集合代数比对未覆盖项，然后针对缺口构造新的合成设备档案与快照对。核心命令形态：

```text
# 采集：替身 adb 只回放 ${WDATA}/<序列号>/ 下的档案文本，全部命令只读
$ export PATH="$PWD/.wstub:$PATH" WDATA=.wdata
$ for d in WGOOD001 WBAD001 WGAP001 WOFF001 WBRANCH001 WNOFACE001 WREG001; do
      bash wireless-posture-snapshot.sh "$d" > "wsnap/$(printf '%s' "$d" | tr 'A-Z' 'a-z').txt"; done

# 差分：八对快照，输出为 d1..d8（输入即第廿一章的四份样例再加四份）
$ bash wireless-snapshot-drift.sh snap14/phone-a-2026-10.txt  snap14/phone-a-2026-12.txt   > snap14/d1.txt
$ bash wireless-snapshot-drift.sh snap14/phone-b-2026-02.txt  snap14/phone-b-2026-10.txt   > snap14/d2.txt
$ bash wireless-snapshot-drift.sh snap14/phone-b-2026-02.txt  snap14/phone-c-2026-10.txt   > snap14/d3.txt
$ bash wireless-snapshot-drift.sh snap14/phone-a-2026-10.txt  snap14/phone-a-2026-10.txt    > snap14/d4.txt
$ bash wireless-snapshot-drift.sh snap14/phone-a-2026-10.txt  snap14/phone-a2-2026-10.txt   > snap14/d5.txt
$ bash wireless-snapshot-drift.sh snap14/phone-d-2026-10.txt  snap14/phone-e-2026-10.txt    > snap14/d6.txt
$ bash wireless-snapshot-drift.sh snap14/phone-a-2026-10.txt  snap14/phone-a-2026-10-hidl.txt > snap14/d7.txt
$ bash wireless-snapshot-drift.sh snap14/phone-d-2026-10.txt  snap14/phone-d-2026-12.txt    > snap14/d8.txt

# 1) 码位全集：从两个脚本正文提取
$ grep -ohE 'row (L[0-9]-[A-Z]+) [a-z0-9_]+' wireless-posture-snapshot.sh wireless-snapshot-drift.sh \
    | awk '{print $3}' | sort -u > all-codes.txt
$ wc -l < all-codes.txt
104

# 2) 实际触发集：从七份画像输出与八对差分输出提取（ROW 行第三个字段才是信号码）
$ cat wsnap/*.txt snap14/phone-*.txt snap14/d*.txt | grep '^ROW,' | awk -F, '{print $3}' | sort -u > seen-codes.txt
$ wc -l < seen-codes.txt
104

# 3) 两个方向的差集都必须为空
$ echo "写了却没触发: $(comm -23 all-codes.txt seen-codes.txt | wc -l)"
写了却没触发: 0
$ echo "触发了但正文未定义: $(comm -13 all-codes.txt seen-codes.txt | wc -l)"
触发了但正文未定义: 0
```

**七份设备画像与八对快照的分工（替身 adb 只回放档案目录里的文本，不含任何设备交互）：**

| 画像 / 快照对 | 为哪些码位而存在 |
| --- | --- |
| 健康基线（GoodPhone X1） | 19 条 L1 事实、强认证三态、Secure NFC 为真、绑定与配对强度记号齐备 |
| 劣化设备（BudgetPhone B9） | `net_open` / `net_wep` / `net_autojoin_open` / `net_tkip` / `net_psk_only` / `nfc_lockscreen_emulation` / `bt_legacy_pairing_marker` / `build_type_nonuser` |
| 采集退化（`subsystems=0/4`） | `wifi_dump_unparsed` 与 `*_feature_unknown` / `*_dump_missing` / `*_module_unread` 全套缺口码位 |
| 射频全关（WOFF001） | `airplane_mode_on`、`*_radio_off` / `nfc_off`、`wifi_dump_missing`（与 unparsed 区分）、`wifi_p2p_absent` |
| 无特性面设备（FeaturelessPhone W1） | `wifi_feature_absent` / `bt_feature_absent` / `nfc_feature_absent` / `cell_feature_absent` 四个"面不存在"分支 |
| 接口回退画像（HIDL + 旧 SPL + 旧基带） | `wifi_hal_iface_hidl`、`baseband_version_read` 的旧值分支 |
| 记号缺失画像（无 prop、无 features、有无记号的 wifi 输出） | 19 条缺口的组合形态，含 `bt_discoverable_unread`、`bt_pairing_strength_unread`、`net_kmgmt_unread` |
| 快照对（劣化 vs 同设备两月后） | `risk_new` / `risk_new_warn` / `open_surface_up` / `strong_auth_down` / `setting_scan_enabled` / `feat_removed` |
| 快照对（整改前 vs 整改后） | `risk_closed_needs_proof` / `warn_resolved` / `open_surface_down` / `strong_auth_up` / `hal_iface_modernized` / `module_channel_added` / `coverage_restored` |
| 快照对（整改前 vs 采集退化） | `risk_gone_by_coverage` / `cov_drop=yes` / `gap_persistent` / `coverage_drop_new` / 全部 `*_unread` |
| 快照对（同内容跨设备） | `pair_identity_mismatch` + `no_drift` |
| 快照对（关闭射频 vs 补全证据） | `feat_added` / `hal_iface_first_read` / `setting_radio_flip` / `coverage_restored` |
| 快照对（可读清单 vs 模块缺席） | `module_channel_lost` 与 `module_channel_unread` 的区分 |

**未覆盖清单：无。** 覆盖过程中曾经有两处一次跑不出来，处理方式值得记录，因为它们正是"分支即缺陷"的实例：

- `airplane_mode_on` 最初没有触发——替身画像里 `settings_global_airplane_mode_on` 是 `0`，走的是 `airplane_mode_off` 分支。**这是档案问题不是脚本问题**，改正画像后触发。
- `feat_added` 与 `hal_iface_first_read` 最初没有触发——所有已构造的快照对里，特性面与 HAL 记号的变化都发生在"当前快照读不到"的方向上，只能走 `feat_now_unread` / `hal_iface_lost`。**这条分支要的是"基线是缺口、当前补全"这一形态**，于是新增一对"射频全关基线 vs 补全后当前"的快照（样例七）。如果没有这轮覆盖检查，这两条码位会成为报告里写了却没验证过的句子。

**红线复核。** 两个脚本的调用面（第十九章列出）全部只读；正文不含任何报文构造、偏移、原语装配或策略绕过内容；公开案例章节只引用公告与研究者自述的结论句，不复现利用过程；样例输出不含 SSID、蓝牙地址与任何凭据。第廿五章指标定义里凡涉及"是否已修"的判定，全部要求外部公告依据，不允许由快照单独给出。

---

## 廿五、十项运营指标：定义、输入行、以及每项不能用来做什么

指标的作用是把本文的判定变成可重复度量的流程量。**每条指标都只从脚本输出行取数**，取不到的项必须记为缺口而不是 0。"禁止用途"一列与前几章的纪律一一对应，它们是同一件事的度量表达。

| 编号 | 指标 | 定义与输入 | 目标口径 | 禁止用途 |
| --- | --- | --- | --- | --- |
| **K1** | 快照覆盖率 | 已留存脚本 1 快照的设备数 ÷ 应清点设备数（台账来自第十六章动作 1） | 覆盖率优先于任何比率结论；未覆盖设备单独列示 | 不得用"已覆盖设备的均值"代表整体 |
| **K2** | 高危成立率 | `SUMMARY` 的 `high>0` 设备占比 | 用于排期，不用于评价整改质量 | 不得跨机型横向比较（第十八章"等级不跨设备比较"） |
| **K3** | 独立更新通道在位率 | `MOD,wifi,mounted` / `MOD,bluetooth,mounted` 的设备占比 | 反映"可不等整机 OTA 修复"的能力面 | **不得读成"已修"**；蓝牙模块本身是可选模块（参考资料 13） |
| **K4** | 强认证存量占比 | `COUNT,wifi_networks_sae + owe + eap` ÷ `wifi_networks_total` | 追踪方向；OWE 不提供认证，单独统计 | 不得用占比替代逐网整改单（第十四章矩阵） |
| **K5** | 无保护与破损存量 | `wifi_networks_none + wep` 的绝对值，以及 `net_autojoin_open` 是否触发 | 绝对值必须归零，不是百分比下降 | 不得把"占比下降"当作成果（存量绝对值才是暴露面） |
| **K6** | TKIP 记号行数 | `COUNT,wifi_tkip_marker_lines` | 趋势用；行数下降记为待复核改进项 | **不得当作设备台数**（第十八章补充规则） |
| **K7** | 绑定与配对强度存量 | `COUNT,bt_bond_marker_lines` + `bt_legacy_pairing_marker` 是否触发 | 绑定按资产登记；旧方式记号要求逐台确认 | 不得由 `bluetooth_on=0` 推断绑定已清理 |
| **K8** | 公告核对闭环时长 | 从公告发布到证据包内含厂商引用号（第十三章最小字段集）的日数 | 分档统计：L3 主机栈类 / L4 vendor 类 / L1 固件类 | 不得对基带档与 Wi-Fi 档用同一个 SLA（第十二章判据三） |
| **K9** | 缺口率 | `SUMMARY` 的 `gap ÷ (high+warn+gap+info)`，按设备与按批次各出一版 | 缺口率上升时，本批所有比率指标标记为不可比 | 缺口不得计为通过；缺口率高的批次不出结论 |
| **K10** | 长期未闭合缺口 | 脚本 2 的 `gap_persistent` 命中的码位数（同一缺口连续两次未闭合） | 走流程告警，而不是继续等设备 | 不得用"消失"（`risk_gone_by_coverage`）替代闭合证明 |

**三条度量纪律：**

- **指标只描述输入，不代替判断。** K3 与 K4 是能力与存量，K2 与 K9 才是处置依据；把 K3 当成加固评分是第十五章"机制可观察性"表格禁止的读法。
- **分母不可比时不出趋势。** 判据子集会随设备能力面变化（无 NFC 的设备不产生 NFC 码位），跨批次比较前必须先比 `subsystems=N/4` 与 `COUNT,wifi_networks_total`。
- **每个指标都要能回溯到行。** 报表里任何一格都应能指出它来自哪份快照的哪一行；这就是本文把脚本输出设计成单行 CSV 的原因（第十九章设计决定 6）。

---

## 廿六、常见误区十二条

1. **把"无线"当成一层。** 六层里任何一层的"存在"都不代表另一层的强度（第三章）。典型症状：一份检查表只有一行"Wi-Fi 安全 = 是"。
2. **把"距离近"读成"风险低"。** 官方口径是"Proximal attack vectors are considered remote."（参考资料 17，第二章判据一）。症状：以"需要靠近"为由降低处置优先级。
3. **把"连得上、用了 WPA3"读成"实现正确"。** KRACK 与 FragAttacks 的公开结论分别证明"密钥建立"与"加密使用"、"帧处理"是可分离的两件事（第九章）。配置面归脚本，实现面归公告与版本记录——这条分工在第十七章是硬边界。
4. **把 SPL 当作基带修复证据。** `spl_read` 只给时间锚点；芯片侧修复的到达由固件包决定，严重度判定权在芯片厂商（第十一、十二、十三章）。症状：以"SPL 已最新"结掉一条基带工单。
5. **把模块缺席读成"未加固"。** 蓝牙 Mainline 模块是可选模块（参考资料 13），`bt_module_absent` 只说明通道未知；而 `/apex` 读不到时更不能用缺席代替（`bt_module_unread`）。
6. **把"读不到"当"没问题"，或把全 `L0` 的快照当安全。** 第廿一章样例三里 `info=0` 的快照描述的是采集状态，不是设备状态（纪律一）。
7. **把"消失"当"已修"。** 必须先比采集面：缺口数上升时的消失判 `risk_gone_by_coverage`(L3)；即便采集面未收窄，也只到 `risk_closed_needs_proof`(L2)，仍要一条变更依据（纪律二）。
8. **把 `spl_moved` 读成"变好了"。** 它是方向无关的锚点语义：第廿三章样例五里 SPL 从 2026-10-05 变为 2024-08-05，该行仍判 L1。倒退的原因（降级、重装、拿错档案）属于变更单核对，不属于脚本判定。
9. **把关闭射频当清理凭据。** `bluetooth_on=0` 不删除绑定，而已配对设备在口径上属于本地身份（参考资料 17；第十六章"不能指望"清单）。
10. **把策略名当效果。** `DISALLOW_CONFIG_BLUETOOTH` 的官方限定句是"This does not restrict the user from turning bluetooth on or off."（参考资料 15）——只设这一位不等于禁用蓝牙。
11. **把行数当设备数、把等级相加当总分。** `*_marker_lines` 是记号行数（K6），等级不可跨设备相加（第十八章末）。
12. **把 HCI 抓包当随手可做的取证动作。** 日志落点与对象标签都在策略管控内，且平台侧自带隐私口径（参考资料 4、20；第十七章）。在受管设备上，这一步需要授权与标签可达，本文不提供抓取方法。

---

## 廿七、趋势判断与分角色结论

**五条趋势（每条都限定在本文口径内）：**

- **趋势一：接口语言迁移会把"能不能拆开修"变成主要差异。** AIDL 三面稳定化（参考资料 6）意味着系统侧升级不再必然要求 vendor 侧同步；反过来说，仍走 HIDL 的设备在第十四章矩阵上属于"通道耦合"的一行。`wifi_hal_iface_*` 三态与脚本 2 的方向判定就是为观察这一迁移而保留的最小探针。
- **趋势二：Mainline 覆盖面扩大，但"可选"仍是关键限定词。** `com.android.wifi` 与蓝牙模块的更新通道语义不同（参考资料 12、13）。**通道在位率的上升不等于风险下降**（K3 禁止用途），它改变的是整改 SLA 的可达形态。
- **趋势三：平台侧对基带的公开姿态从"事后补丁"转向"编译期与运行期加固"。** 参考资料 26 里"architecture agnostic, suitable for bare-metal deployment"与内存安全语言的长期方向两句，说明这一层的收敛手段不依赖 Android 的权限模型。对治理的含义是：**基带档的证据要求会长期保持"外部依据 + 版本串"的形态**，不会因设备侧工具演进而改变。
- **趋势四：协议设计类假设被公开记录为缺陷归属对象。** 通用漏洞库把 BIAS 族归属到"Core Specification v5.2 and earlier"（参考资料 25）而不是某个实现——这意味着未来的同类问题更可能以"规范版本 + 控制器能力"形态出现，而这类问题在设备上不可观察。第十七章"实现类归公告"的分工因此要长期保留。
- **趋势五：抓包与日志数据的治理属性上升。** 落点与独立 SELinux 标签、以及"只记录非个人信息"的官方隐私口径（参考资料 4、20）说明：**取证通道本身正在被当作一类受控数据治理**。对企业侧的含义是把"是否允许 HCI 日志"纳入基线而不是临时决定（第十六章动作 1）。

**分角色结论：**

- **移动应用与 SDK 负责人**：无线侧不是你的层，但你是"配置存量的引入者"。本文对你只有一条硬结论：**权限收紧不等于通道关闭**（第五章、第十五章机制四）。不要在任何对外承诺里写"已启用安全蓝牙/Wi-Fi"来暗示协议实现正确。
- **MDM 与安全基线团队**：以第十六章六个策略位为基线候选，以脚本 1 输出为留存格式，以脚本 2 差分为验收。三件必须做的事：分档 SLA（K8）、缺口走上报（K10）、策略位与状态位分别记录（误区 10）。
- **ROM/OEM 系统工程师**：第十四章矩阵里 L2/L4 两行归你。接口语言状态与三个 HAL 面的归属应在构建说明里明确，因为**设备侧只能读到记号，读不到装配关系**；这也决定客户能否把一次修复拆开下发。
- **检测与取证工程师**：你的产出物是第十七章的证据包，不是评分。三条硬限制：不越界判实现类缺陷、不把采集退化报成整改、不在未授权设备上抓取受标签管控的数据。
- **应用商店与风控 / 平台评估机构**：本文的四级判定不可跨设备横向比较（第十八章末、K2 禁止用途），因此不适合用于评级排序，适合用于**同设备时间序列的整改验证**。

**一句收束**：无线侧风险治理的难点从来不是"发现某个缺陷"，而是**在没有缺陷证据的情况下，仍然能把"我不知道什么"记录下来并让它可复核**。本文两个脚本给出的全部价值都在这里。

---

## 廿八、常见问题

**问：这份快照能不能告诉我设备"安全"？**
答：不能。第十八章的模型只输出三件事：此刻的配置存量、相对基线的变化、以及本次没看到的东西。第三件最容易被忽略，也最影响前两件能不能用于结单。

**问：为什么高危项只出现在网络配置上，而不是"某个 CVE"？**
答：因为设备侧只读采集能可靠判定的就是配置面。CVE 归属来自公告与版本记录（第十七章分工表）；把两者混在一张表里，会让"没有触发高危"被读成"没有 CVE"——这是误区 3 与 4 的同一形态。

**问：`gap` 那一列能不能忽略？**
答：不能，它决定其它列的可信度。脚本 2 的 `cov_drop` 直接以它作为"消失是否可读成已修"的开关；第廿三章样例三是这条机制存在的理由。

**问：为什么脚本不采集 SELinux 域、不采集应用权限？**
答：故意的边界（第十五章机制三、机制四）。前者在[应用沙箱篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)已有完整方法与脚本，后者属于应用侧清点。重复实现只会让两份报告给出可能不一致的判定。

**问：能不能拿 A 设备的基线和 B 设备的当前快照做差分？**
答：不能，脚本会告诉你（`pair_identity_mismatch`）。同一机型也不行——第廿三章样例四显示内容完全相同的两份快照仍因机型不同而只给"不可差分"的结论。同源性由操作者记录（`NOTE,identity` 行）。

**问：Wi-Fi 关闭、蓝牙关闭是不是就不用采了？**
答：第廿一章样例四演示了反面：关闭状态下仍有可采事实（策略位、模块通道、能力面），且 `dumpsys` 为空会带来三个新缺口。**关闭不减少采集需求，它改变判据子集。**

**问：为什么 OWE 只判 `L1-INFO`，不算强认证？**
答：官方句子写明"Enhanced Open doesn't provide authentication."（参考资料 7）。OWE 在开放介质上建立加密会话，但不提供网络身份凭据，因此与 SAE/EAP 分档（第四章）。

**问：脚本能不能在没有真机的情况下验证？**
答：本报告的全部样例就是在合成设备画像 + 替身 adb 上跑出来的，第廿四章给出可复现的覆盖方法。真机的价值是验证标记词契约是否符合厂商输出——这件事只能你自己做一次（第十九章设计决定 1）。

**问：能不能直接给这些信号码打分、算一个总分？**
答：不能，且这是设计选择（第十八章）。总分会掩盖两件事：等级不可跨设备比较，以及 `L0-LOW` 是输入不足而不是低分。

**问：这些案例（KRACK/BIAS/Exynos）现在还有什么可执行的意义？**
答：三件：它们分别定义了"配置面能判到哪"（第九章）、"绑定是资产不是状态"（第十章）、"哪一层必须靠外部证据"（第十一章）。它们的利用细节不在本文范围，正文只引用公开结论句与位置前提。

---

## 廿九、参考资料

以下条目为本次核验实际取得逐字内容的公开来源，编号供正文引用；引文为原文（英文）摘录，破折号后为中文口径说明。截断处以"（原文截断）"标注；未取得整句的条目明确写出"本次不据此立论"。

1. 内核文档 `cfg80211` 接口页："cfg80211 is the configuration API for 802.11 devices in Linux. It bridges userspace and drivers, and offers some utility functionality associated with 802.11. cfg80211 must, directly or indirectly via mac80211, be used by all modern wireless drivers in Linux, so that they offer a consistent API through nl80211." — 无线驱动的统一配置入口、userspace 与驱动之间的桥、以及"所有现代驱动必须走这套接口"的强制约定；第三章 L2 与第十四章"一次修复能否覆盖多器件"的依据。https://www.kernel.org/doc/html/latest/driver-api/80211/cfg80211.html
2. AOSP 蓝牙文档："Android provides a default Bluetooth stack that supports both Classic Bluetooth and Bluetooth Low Energy (BLE)."、"A Bluetooth app communicates with the Bluetooth process through Binder."、"Vendor devices interact with the Bluetooth stack using the HAL interface definition language (HIDL)." — 默认栈同时覆盖经典与 BLE；应用经 Binder 与蓝牙进程通信；vendor 侧接口口径。https://source.android.com/docs/core/connect/bluetooth
3. AOSP 蓝牙 HCI 要求页："The Host Controller Interface (HCI) is used for interacting with a Bluetooth controller." — 主机与控制器接口定义；第三章 L3/L1 分界与第十七章取证通道的术语来源。https://source.android.com/docs/core/connect/bluetooth/hci_requirements
4. AOSP 蓝牙调试与验证页：日志落点"stored in `data/misc/bluetooth/logs`"；隐私口径"For privacy reasons, always-on, … BTSnoop only logs non-personal information and events."（原文截断）— HCI 日志位置与"只记录非个人信息"的官方限定；第十七章限制 1 的依据。https://source.android.com/docs/core/connect/bluetooth/verifying_debugging
5. AOSP Wi-Fi 概览："The Wi-Fi service communicates with the Wi-Fi HAL over AIDL."、"a standalone process, **wificond**, located at `system/connectivity/wificond`"、"three Wi-Fi HAL surfaces ... Vendor HAL, Supplicant HAL, and Hostapd HAL." — 服务与 HAL 的接口语言、wificond 独立进程位置、三个 HAL 面。https://source.android.com/docs/core/connect/wifi-overview
6. AOSP Wi-Fi HAL："…all three interfaces are available as stable Android Interface Definition Language (AIDL) packages. These interfaces were previously defined using Hardware Interface Definition Language (HIDL)." — 三面已迁至稳定 AIDL，此前为 HIDL；第十五章机制二与脚本 HAL 记号判级的依据。https://source.android.com/docs/core/connect/wifi-hal
7. AOSP WPA3/OWE："Android 10 introduces support for the Wi-Fi Alliance's (WFA) Wi-Fi Protected Access version 3 (WPA3) and Wi-Fi Enhanced Open standards."、"WPA3-Personal: Uses simultaneous authentication of equals (SAE) instead of pre-shared key (PSK), providing users with stronger security protections against attacks such as offline dictionary attacks, key recovery, and message forging."、"Wi-Fi Enhanced Open is a new WFA security standard for public networks based on opportunistic wireless encryption (OWE). It provides encryption and privacy on open, non-password-protected networks…"、"Enhanced Open doesn't provide authentication."、"To support WPA3 and Wi-Fi Enhanced Open, implement the Supplicant HAL interface. Starting with Android 13, the interface uses Android Interface Definition Language (AIDL) for the HAL definition."、"WPA2/WPA3 transition mode: The serving access point supports WPA2 and WPA3 standards concurrently…"、"OWE transition mode: The serving access point supports both OWE and open standards concurrently…" — SAE 抗离线字典与密钥恢复；OWE 不提供认证；Supplicant HAL 是 WPA3 前置；两种过渡模式在定义上允许老设备走 PSK 或不加密（第四章、`net_psk_only` 判级依据）。https://source.android.com/docs/core/connect/wifi-wpa3-owe
8. AOSP Passpoint："The supplicant provides support for the 802.11u standard, specifically network discovery and selection features such as Generic Advertisement Service (GAS) and access network query protocol (ANQP)." — 网络发现落在 supplicant 的 802.11u 支持；`wifi_passpoint_present` 记为厂商责任面存在的依据。https://source.android.com/docs/core/connect/wifi-passpoint
9. AOSP Secure NFC："Secure NFC is a feature introduced in Android 10 that allows off-host card emulation to be enabled only when the device's screen is unlocked."、"the device must have an NFC controller that supports the NCI 2.0 standard…"（原文截断）、"Android supports off-host card emulation, which means NFC card emulation with a secure element." — Secure NFC 定义、NCI 2.0 前置条件、off-host 含义；第六章判据三与 `nfc_secure_*` 码位依据。https://source.android.com/docs/core/connect/secure-nfc
10. Android 开发者 NFC 主机卡模拟："This coexistence is based on a principle called *AID routing*…"、"Android never starts or binds to a service that is declared as \"off-host\"…" — AID 路由与共存规则；主机侧与 off-host 的边界句。https://developer.android.com/develop/connectivity/nfc/hce
11. AOSP Open Mobile API："Open Mobile API (OMAPI) is a standard API used to communicate with a device's Secure Element." — 安全元件访问接口定义；本文只用其界定"SE 访问是独立接口面"。https://source.android.com/docs/security/features/open-mobile-api
12. AOSP Mainline（modular system components）："Android 10 introduced Mainline (formerly called modular system components on this site)." — 模块化的起点与站点旧称；第十五章机制一与 `MOD,*` 行的依据。https://source.android.com/docs/core/ota/modular-system
13. AOSP 蓝牙模块页："The Bluetooth module is an optional Mainline module introduced in Android 13. In Android 16 and higher, the module is updatable and contains a fully certified dual-mode Bluetooth…"（原文截断）— 蓝牙 Mainline 模块为**可选**模块；"optional"是 `bt_module_absent` 只判缺口的直接依据。https://source.android.com/docs/core/ota/modular-system/bluetooth
14. Android 蓝牙运行时权限："The `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, and `BLUETOOTH_SCAN` permissions are runtime permissions."、"If you include `neverForLocation` in your `android:usesPermissionFlags`, some BLE **beacons** are filtered from the scan results."、"If your app uses Bluetooth scan results to derive physical location, declare the ACCESS_FINE_LOCATION permission." — 三个运行时权限；`neverForLocation` 过滤的是 beacons（官方当前措辞）；位置推导的权限选择。https://developer.android.com/guide/topics/connectivity/bluetooth/permissions
15. `UserManager` 参考文档：`DISALLOW_BLUETOOTH` 的"Specifies if bluetooth is disallowed on the device…"（原文截断）；`DISALLOW_CONFIG_BLUETOOTH` 的"This does not restrict the user from turning bluetooth on or off." — "能不能用"与"能不能改设置"是两个策略位，后者不控制射频开关；第十六章与误区 10 的依据。https://developer.android.com/reference/android/os/UserManager
16. `WifiManager` 参考文档："An app must hold ACCESS_FINE_LOCATION … and ACCESS_WIFI_STATE permission in order to get valid results."（原文截断）— 扫描结果有效性的权限前提。https://developer.android.com/reference/android/net/wifi/WifiManager
17. AOSP 安全公告口径（攻击向路与严重度评估）："Proximal attack vectors are considered remote."、"A remote attack vector indicates that the bug can be exploited without installing an app or without physical access to a device."、"Local attacks require the attacker to have prior access to the victim."、"Successfully paired devices (such as Bluetooth companion devices) are considered local."、"Physical attack vectors are considered local." — 第二章四条判据的全部来源；也是"邻近不降级"与"已配对算本地身份"的官方口径。https://source.android.com/docs/security/overview/updates-resources
18. Android 安全公告 2021-06-01：严重度前提"…severity assessment is based on the effect that exploiting the vulnerability would possibly have on an affected device, assuming the platform and service mitigations are turned off for development purposes…"（原文截断）；芯片侧措辞"…affect Qualcomm closed-source components"、"The severity assessment of this issue is provided directly by Qualcomm."、"The severity assessment of these issues is provided directly by MediaTek."；蓝牙条目形态"could enable a remote attacker using a specially crafted transmission to execute arbitrary code within the context of a privileged process" — 严重度前提是缓解被关闭、判定权可属芯片厂商、以及"邻近即远程"在条目里的实际措辞；本文只引用这些口径句，不引用任何构造细节。https://source.android.com/docs/security/bulletin/2021-06-01
19. AOSP sepolicy `private/seapp_contexts`（main 分支）："user=bluetooth seinfo=bluetooth domain=bluetooth type=bluetooth_data_file"、"user=nfc seinfo=nfc domain=nfc type=nfc_data_file"、"user=radio seinfo=platform domain=radio type=radio_data_file" — 无线相关进程的用户标识、seinfo 与执行域映射可逐行读取；第五章身份与域判定的参照物。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/seapp_contexts
20. AOSP sepolicy `private/file_contexts`（main 分支）："/data/misc/bluetooth/logs(/.*)? u:object_r:bluetooth_logs_data_file:s0" — HCI 日志目录有独立对象标签；第十七章"取证数据是被单独管控的一类数据"的依据。https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts
21. FragAttacks 项目页："This website presents FragAttacks … which is a collection of new security vulnerabilities that affect Wi-Fi devices."、"Every Wi-Fi product is affected by at least one vulnerability and that most products are affected by several vulnerabilities."、"An adversary that is within range of a victim's Wi-Fi network can abuse these vulnerabilities to steal user information or attack devices." — 分片与聚合类缺陷的存在性、覆盖面与邻近前提；第九章 W1 类型的公开依据，本文不引用任何利用细节。https://fragattacks.com
22. KRACK 项目页："Key Reinstallation Attacks: Forcing Nonce Reuse in WPA2"、"An attacker within range of a victim can exploit these weaknesses using key reinstallation attacks (KRACKs)." — 密钥重安装与 nonce 复用的公开命名口径；"加密已建立 ≠ 密钥使用正确"的来源。https://krackattacks.com
23. Dragonblood 项目页（WPA3 SAE 研究）：在特定条件下"an attacker within range of a victim can still recover the password." — SAE 侧密码学协议问题与"只用 SAE 才是目标状态"的整改方向依据。https://wpa3.mathyvanhoef.com
24. CERT/CC 通报 VU#647177："It is possible for an unauthenticated, adjacent attacker to impersonate a previously paired/bonded device and successfully authenticate without knowing the link key." — 未认证 + 邻近 + 无需链路密钥三个可达条件；W6 类型与绑定资产化的依据。https://kb.cert.org/vuls/id/647177/
25. 通用漏洞库条目 CVE-2020-10135（经 NVD API 取得；网页端对本机返回 403）："Legacy pairing and secure-connections pairing authentication in Bluetooth BR/EDR Core Specification v5.2 and earlier may allow an unauthenticated user to complete authentication without pairing credentials via adjacent access."，其基础严重度字段为 `MEDIUM`（评分 5.4）— 同一族缺陷归属到**规范版本**而非实现，且与公告口径存在两档差异；第十三章对照表的第三行。https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2020-10135
26. Google 安全博客（基带加固）："Android's defense-in-depth strategy applies not only to the Android OS running on the Application Processor (AP) but also the firmware that runs on devices."、"We particularly prioritize hardening the cellular baseband given its unique combination of running in an elevated privilege and parsing untrusted inputs that are remotely delivered into the device."、"They are architecture agnostic, suitable for bare-metal deployment…"（原文截断）、"Other classes of memory safety vulnerabilities remain unaddressed by these sanitizers. A longer term solution is to begin transitioning today to memory-safe languages such as Rust." — 基带单列一档的全部公开依据；加固手段属编译期与运行期而非权限模型。https://security.googleblog.com/2023/12/hardening-cellular-basebands-in-android.html
27. Project Zero（2023）："In late 2022 and early 2023, Project Zero reported eighteen 0-day vulnerabilities in Exynos Modems produced by Samsung Semiconductor." — 网络到基带的远程缺陷族存在性；本文只引用其公开结论与时间线口径。https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html
28. 芯片厂商产品安全公告入口（MediaTek PSGB）：本次抓取返回 302 且未取得正文，**不据此立论**；第十一章"厂商自身也发布独立公告"的依据实际取自参考资料 18 中对 MediaTek 条目的转述措辞。https://corp.mediatek.com/product-security-bulletin
29. BlueBorne 项目页（存档快照；原页面对本机返回 403）："BlueBorne is an attack vector by which hackers can leverage Bluetooth connections to penetrate and take complete control over targeted devices." — 蓝牙无需配对与无需可发现即可被攻击的公开命名口径。http://web.archive.org/web/20201228171231/https://www.armis.com/blueborne/
30. BLEEDINGBIT 项目页（原始出处登记）：原站地址 `https://www.armis.com/bleedingbit/` 对本机返回 403，正文取自下一条的存档快照；本条仅登记出处，不单独引用。https://www.armis.com/bleedingbit/
31. BLEEDINGBIT 项目页（存档快照）："Armis has identified two chip-level vulnerabilities impacting access points and potentially other unmanaged devices."、"These proximity-based vulnerabilities allow an unauthenticated attacker to break into enterprise networks undetected." — 受影响对象是接入点与未纳管设备（不是 Android 主机栈）；L1 控制器档"邻近 + 未认证 + 不可检测"的公开形态。http://web.archive.org/web/20200120062729/https://www.armis.com/bleedingbit/
32. 通用漏洞库条目 CVE-2019-9506（Key Negotiation of Bluetooth）：本文未取得可逐字引用的整段描述（网页端对本机返回 403），第廿八章与第十章的 KNB 段落仅使用该编号的存在性，**不据此立论**。https://nvd.nist.gov/vuln/detail/CVE-2019-9506
33. CWE 编号定义页：CWE-20（Improper Input Validation）、CWE-125（Out-of-bounds Read）、CWE-130（Improper Handling of Length Parameter Inconsistency）、CWE-787（Out-of-bounds Write）— 第八章 W1/W7 对齐所用的四个编号与标题；本文不使用其"常见落点"扩展描述。https://cwe.mitre.org/data/definitions/20.html 、 https://cwe.mitre.org/data/definitions/125.html 、 https://cwe.mitre.org/data/definitions/130.html 、 https://cwe.mitre.org/data/definitions/787.html
34. CWE 编号定义页：CWE-326（Inadequate Encryption Strength）、CWE-327（Use of a Broken or Risky Cryptographic Algorithm）、CWE-319（Cleartext Transmission of Sensitive Information）、CWE-295（Improper Certificate Validation）— W3/W4 握手降级与破损配置一行的对齐来源；协商类缺陷本文不硬套单个编号。https://cwe.mitre.org/data/definitions/326.html 、 https://cwe.mitre.org/data/definitions/327.html 、 https://cwe.mitre.org/data/definitions/319.html 、 https://cwe.mitre.org/data/definitions/295.html
35. CWE 编号定义页：CWE-400（Uncontrolled Resource Consumption）、CWE-693（Protection Mechanism Failure）— 第八章 W8 行与"缓解类兜底编号不作为定义依据"这一自我限制的来源。https://cwe.mitre.org/data/definitions/400.html 、 https://cwe.mitre.org/data/definitions/693.html

**未公开文档化 / 本次未逐字核验的边界（正文一律不据此立论）：**

- **Wi-Fi Direct 侧关于 WEP 与 ad-hoc 网络的历史措辞**：本次未取得可逐字引用的整句（相关 guide 页面返回 404），第三章与脚本判级不使用这一表述；WEP 的 L3 判定只依据参考资料 21/23 所对应的"破损算法"这一通用口径与参考资料 17 的攻击向路定义。
- **`nl80211` 中 monitor 接口与注入命令的枚举文本**（`NL80211_IFTYPE_MONITOR`、`NL80211_CMD_FRAME` 一类常量的官方逐字定义）：未取得，第二章与第十二章不解释这两个常量的语义，只使用"内核侧统一配置接口"这一层表述（参考资料 1）。
- **`AID_BLUETOOTH` 的具体数值**：未取得逐字来源，本文不使用任何数值身份，只用 UID/域映射行（参考资料 19）。
- **RIL 重构与调制解调器 wakelock 的公开机制整句**：未取得，第十二章只用"基带不经过 L2–L5"这一层推理，不涉及 RIL 具体实现。
- **`com.android.wifi` 在模块表里"引入版本"一行的逐字措辞**：参考资料 12 只给到 Mainline 起点句，本文不写"Wi-Fi 模块在 Android 11 引入"这类未逐字核验的版本行。
- **蓝牙"Android 11 及以下通过位置权限承载扫描结果"的历史整句**：未取得，第十四章与第十五章只使用参考资料 14 当前页面的三句口径。
- **Gabeldorsche（Google 自研蓝牙栈）的公开文档**：本次在所有尝试路径均返回 404，正文不提及该栈的形态、可用范围或默认状态。
- **Wi-Fi 联盟关于 WPA3 强制性的某份年度声明原文**：未定位到可引用的页面地址，第九章与第十六章不引用"强制时间点"。
- **芯片厂商公告页正文（参考资料 28、30）**：本机抓取不可达，正文对这两处只登记出处；任何具体条目、版本对应表与严重度都必须由读者按第十三章方法自行抓取留存。
- **厂商 `dumpsys wifi` / `dumpsys bluetooth_manager` 的输出格式规范**：不存在公开格式文档，脚本按标记词契约解析，并在解析不出时判 `wifi_dump_unparsed`（第十九章设计决定 1）；厂商字段增减属预期事件，不构成缺陷判定。
- **具体 CVE 的利用链、内存布局、报文构造与策略绕过步骤**：不属本文范围。相关口径见[内核漏洞与提权防御](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)、[远程代码执行](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)与[攻击面与漏洞研究方法论](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)三篇。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
