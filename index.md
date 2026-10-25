---
layout: default
title: "Android 安全分析报告集"
description: "Android 平台防御性安全分析报告合集：内核漏洞与提权、Framework 信任边界、系统架构、内核结构与子系统、硬件信任根与密钥栈、Binder IPC 协议与运行时"
lang: zh-CN
permalink: /
---

# Android 安全分析报告集

本站点收录 Android 平台的**防御性**安全分析报告。内容只涉及公开文档化的机制、公开披露的漏洞类别、影响分析与加固/检测建议，**不提供任何可用的漏洞利用代码、偏移量或可操作的攻击步骤**。

每篇报告都是一份可独立审阅的长文：机制说明 → 类型学或矩阵 → 检测与取证方法 → 常见误区 → 趋势判断 → FAQ → 逐条可核验的公开参考资料。所有事实性表述均来自 AOSP 源码、内核 uapi 头文件、Google 官方文档与安全公告，并在正文中显式标注"未公开文档化"的边界。

## 报告清单

| 报告 | 覆盖范围 | 章节 | 篇幅 |
| --- | --- | --- | --- |
| [Android 内核漏洞与提权防御分析报告](https://prestonh965118.github.io/Android/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense/) | 内核攻击面分级、缺陷类型学、公开 CVE 案例、缓解矩阵、补丁治理指标与提权取证 | 14 章 | 544 行 |
| [Android Framework 信任边界与越权漏洞分析报告](https://prestonh965118.github.io/Android/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis/) | Framework 层身份与权限模型、越权类缺陷类型学、EoP 公告归因与检测 | 14 章 | 572 行 |
| [Android 系统架构分析报告：分层、进程与信任边界](https://prestonh965118.github.io/Android/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries/) | 六段分层、进程模型、HAL/Treble、SELinux/seccomp/AID 身份与三条更新通道 | 19 章 | 655 行 |
| [Android Linux 内核结构与子系统机制分析报告](https://prestonh965118.github.io/Android/Android-Linux-Kernel-Structure-and-Subsystems/) | GKI 内核镜像形态、模块与子系统、设备节点供给、安全控制落点与能力边界 | 18 章 | 610 行 |
| [Android 硬件信任根与密钥栈机制分析报告](https://prestonh965118.github.io/Android/Android-Root-of-Trust-and-Key-Stack-Mechanisms/) | AVB 验证启动、DICE 与 RKP、KeyMint/StrongBox、Gatekeeper/Weaver、密钥证明与 FBE | 20 章 | 778 行 |
| [Android Binder IPC 协议与运行时机制分析报告](https://prestonh965118.github.io/Android/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms/) | BC_/BR_ 协议词表、对象生命周期与死亡/冻结、事务缓冲区与线程池、SELinux 判定、binderfs、Stable AIDL 与 RPC | 22 章 | 924 行 |

## 建议阅读顺序

1. 先读**系统架构**篇建立分层与信任边界的整体坐标；
2. 再读**内核结构**篇，弄清代码从哪来、打进哪个镜像、由谁加载；
3. 需要评估跨进程调用与权限判定时读 **Binder IPC** 篇（协议与运行时）与 **Framework 信任边界**篇（缺陷类型学）；
4. 关注设备完整性、密钥与证明体系时读**硬件信任根与密钥栈**篇；
5. 关注内核侧风险与补丁治理时读**内核漏洞与提权防御**篇。

各篇之间按"视角"分工并互相交叉引用，重复内容只保留引用不重复展开。

## 代码仓库与镜像

正文以本 Pages 站点为主版本；下列仓库为同源镜像，用于版本管理与审阅提交历史。

- GitHub：<https://github.com/prestonh965118/Android>
- GitLab：<https://gitlab.com/prestonh965118/Android>
- Codeberg：<https://codeberg.org/prestonh965118/Android>
- Bitbucket：<https://bitbucket.org/pengqinglei/android>

## 更新记录

- 2026-10-24：新增《Android Binder IPC 协议与运行时机制分析报告》；站点化（Pages + 目录页 + 站点地图）。
- 2026-10-23：新增《Android 系统架构分析报告》《Android Linux 内核结构与子系统机制分析报告》《Android 硬件信任根与密钥栈机制分析报告》。
- 2026-10-22：新增《Android 内核漏洞与提权防御分析报告》《Android Framework 信任边界与越权漏洞分析报告》。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
