# Android 安全分析报告集

本仓库收录 Android 平台防御性安全分析文章，内容仅涉及公开披露的机制、漏洞类别、影响分析与加固建议，不提供任何可用的漏洞利用代码或攻击操作步骤。

在线站点（渲染后的正文主版本）：<https://android-security-reports-0e8t10askpz.qoder.website/>
- sitemap：<https://android-security-reports-0e8t10askpz.qoder.website/sitemap.xml>
- GitHub Pages 版本（待启用）：<https://prestonh965118.github.io/Android/>

## 目录

| 文件 | 主题 |
| --- | --- |
| [Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.md](./Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.md) | Android 内核漏洞与提权防御分析 |
| [Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.md](./Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.md) | Android Framework 信任边界与越权漏洞分析 |
| [Android-Linux-Kernel-Structure-and-Subsystems.md](./Android-Linux-Kernel-Structure-and-Subsystems.md) | Android Linux 内核结构与子系统机制分析 |
| [Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.md](./Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.md) | Android 系统架构：分层、进程与信任边界分析 |
| [Android-Root-of-Trust-and-Key-Stack-Mechanisms.md](./Android-Root-of-Trust-and-Key-Stack-Mechanisms.md) | Android 硬件信任根与密钥栈机制分析（AVB/DICE/KeyMint/Gatekeeper/证明/RKP） |
| [Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.md](./Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.md) | Android Binder IPC 协议与运行时机制分析（BC_/BR_ 词表、对象生命周期、缓冲区、SELinux、binderfs、Stable AIDL、RPC） |
| [Android-Attack-Surface-and-Vulnerability-Research-Methodology.md](./Android-Attack-Surface-and-Vulnerability-Research-Methodology.md) | Android 攻击面与漏洞研究方法论分析（九域可达面清点、R0–R4 分级、目标选择评分、静态与模糊测试、补丁到达性验证、检测覆盖矩阵与只读测绘脚本） |
| [Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.md](./Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.md) | Android 远程代码执行分析（公告 RCE 口径、四条投递通道、执行落点权限模型、缺陷类型学、缓解机制与失效面矩阵、七项运营指标与两个只读核验脚本） |
| [Android-Hardware-Security-Components-and-Vendor-Implementations.md](./Android-Hardware-Security-Components-and-Vendor-Implementations.md) | Android 厂商硬件安全组件形态与实现差异分析（独立安全处理器、安全世界、Trusty/OP-TEE、StrongBox/eSE、AVF/pKVM、可外部核验信号与不可核验边界） |
| [Android-Patch-Delivery-Chain-and-OEM-Update-Governance.md](./Android-Patch-Delivery-Chain-and-OEM-Update-Governance.md) | Android 补丁传导链与厂商更新治理分析（SPL 两档与框架/厂商两侧、S1–S5 链条、三条更新通道、更新承诺口径、八项指标与两个只读核验脚本） |
| [Android-Platform-Code-Organization-and-Build-System-Analysis.md](./Android-Platform-Code-Organization-and-Build-System-Analysis.md) | Android 平台代码组织与构建体系分析（repo/manifest 源码供给与分支口径、Soong/kati/Ninja、out 与分区装配、签名与 test-keys 构建身份、AVB 绑定、APEX/Mainline、ACK/GKI 交付口径、CDD/CTS/VTS/STS、十项供应链风险清点与两个只读核验脚本） |
| [Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.md](./Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.md) | Android 应用沙箱与进程隔离防御分析（沙箱四层叠加：UID/DAC 身份、SELinux 域与类别、seccomp 版本下限、进程与数据形态；身份合并与可调试性；S1–S8 逃逸类型学；四级信号分级、十项运营指标与两个只读核验脚本） |
| [Android-App-Security-Weakness-Taxonomy-Defense-Analysis.md](./Android-App-Security-Weakness-Taxonomy-Defense-Analysis.md) | Android 应用安全弱点分类分析（MASVS/MASTG/MASWE/Mobile Top 10/CWE 五套公开口径的分工与换算、MASVS v1 V1–V8 与 v2 八域迁移、CWE-919 移动视图 21 成员、清单层可见的 41 个弱点信号与不可见的四类、目标版本默认值漂移、四级判定模型与十项运营指标、两个离线只读核验脚本） |
| [Android-Wireless-Stack-and-Baseband-Defense-Analysis.md](./Android-Wireless-Stack-and-Baseband-Defense-Analysis.md) | Android 无线协议栈与基带缺陷防御分析（无线侧六层构成、协议栈/配对认证/基带固件三套口径的分离、"邻近即远程"的严重度基线、W1–W10 缺陷类型学与 CWE 对齐、FragAttacks/KRACK/Dragonblood/BIAS/KNB/BlueBorne/BLEEDINGBIT/Exynos 八组公开案例读数、基带可见性边界、层×责任×更新通道×设备侧证据归属矩阵、四级判定模型与十项运营指标、两个只读核验脚本共 104 个信号码） |
| [Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.md](./Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.md) | Android 微架构侧信道与密钥抽取防御分析（六层归属：硅与微码/内核编译期/内核启动参数/平台策略/运行时与虚拟机/密钥业务实现、`/sys/devices/system/cpu/vulnerabilities` 三前缀词表与文件名清单、arm64 与 x86 词表不等价为什么不能跨架构比等级、M1–M10 按观测能力分型与 CWE-1303/200/208 对齐、Spectre/Meltdown/MDS/GDS/Retbleed 与 Arm 专属研究家族的读数方式、Zygote 共享与 DVFS 等 Android 特有放大器、SELinux 对 perf 与 /proc 的收敛、微码到达性这条唯一无法自证的层、四级判定模型与 K1–K10 运营指标、两个只读核验脚本共 141 个信号码与全量分支覆盖验收） |

---

咨询ios系统请咨询 telegram：https://t.me/one00190
