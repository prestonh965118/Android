---
layout: default
title: "Android Binder IPC 协议与运行时机制分析报告"
description: "Binder 的 BC_/BR_ 协议词表、对象生命周期、事务缓冲区与线程池、SELinux 判定、binderfs、Stable AIDL 与 RPC 机制分析"
lang: zh-CN
permalink: /Android-Binder-IPC-Protocol-and-Runtime-Mechanisms/
---
# Android Binder IPC 协议与运行时机制分析报告

## Binder IPC on Android: Transaction Protocol, Object Lifecycle, Thread Pools, SELinux Checks and Observability Boundaries

- 版本：v1.0（2026-10-24）
- 适用对象：设备/固件安全审计、系统与应用性能治理、跨进程接口设计（AIDL/HIDL/RPC）、移动安全管理（MDM/MAS）、取证与可观测性工程
- 视角声明：本文写的是 Binder 的**协议与运行时机制**——一次调用在内核里经历了哪些命令、对象与引用怎么存活、缓冲区和线程池怎么成为容量边界、身份与权限在哪一步被检查、以及这些事实里哪些在生产设备上真的可观测。为避免与已有五篇重复，下列内容只做交叉引用：
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)：三个 binder 域的形状、Parcel「数据即指令」、进程与 Zygote 模型
  - [Android Linux 内核结构与子系统机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Linux-Kernel-Structure-and-Subsystems.html)：`/dev/binder` 等设备节点由谁创建、驱动在内核里的位置、debugfs 与遥测的总体约束
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)：Framework 侧的攻击面清单与缺陷类型学
  - [Android 硬件信任根与密钥栈机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Root-of-Trust-and-Key-Stack-Mechanisms.html)：`@VintfStability` 与密钥 HAL 的关系、证明与启动状态
  - [Android 内核漏洞与提权防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)：缺陷分类与缓解矩阵；本文只引用 binder 相关的公告条目，不重复案例复盘
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)：应用进程侧的 UID/域/seccomp 叠加与身份合并（`sharedUserId`、隔离进程）；本文讲 binder 在哪一步检查这套身份，该篇讲身份本身怎么形成、怎么被观察
  - [Android 应用安全弱点分类分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Security-Weakness-Taxonomy-Defense-Analysis.html)：清单里那些"谁会接到跨进程调用"的导出与权限声明属应用侧姿态（`sharedUserId`、导出组件、URI 授权），本文讲 binder 在哪一步检查这套身份与权限
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（本文讲 binder 在哪一步检查身份与权限；该篇讲承载 Wi-Fi/蓝牙/NFC 请求的那几类协议栈进程处于哪个执行域，以及 HAL 接口语言从 HIDL 转 AIDL 的记号为什么只能当旁证而不是修复证明）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（本文写协议词表、身份传递与 SELinux 判定；该篇不涉及 IPC 接口缺陷，只在平台策略层借用本文的域标识口径说明采样与观测权限归谁）
  - [Android Project Mainline 与 APEX 模块缺陷归属分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Project-Mainline-and-APEX-Module-Defect-Attribution-Analysis.html)（本文写协议词表、身份传递与 SELinux 判定；该篇不涉及接口缺陷，只把服务面的"查找门"与"调用者门"当成观测面来分流，用来说明一次失败该记策略缺口还是记身份缺口）
- 范围声明：全文仅描述**公开文档化与源码可核验的机制**。不提供任何利用步骤、偏移量或绕过流程；涉及 binder 驱动缺陷类别时，只转述 Google 安全公告给出的定性字段（类型、严重级、受影响组件），不描述成因细节。

---

## 一、摘要

1. **Binder 的全部每调用流量只经过一个 ioctl**：`BINDER_WRITE_READ`（`_IOWR('b', 1, struct binder_write_read)`）。其余 ioctl 都是配置/查询：`BINDER_SET_MAX_THREADS`、`BINDER_SET_CONTEXT_MGR`、`BINDER_VERSION`、`BINDER_FREEZE`、`BINDER_ENABLE_ONEWAY_SPAM_DETECTION`、`BINDER_GET_EXTENDED_ERROR` 等。审计任何"Binder 行为异常"，可读性最好的证据面就是这次读写里来回的 BC_/BR_ 命令序列。
2. **协议词表是可枚举的**：出向 `BC_*` 22 条、入向 `BR_*` 24 条（见第三章清单，逐字取自 uapi 头文件枚举）。网传的一些名字并不存在——`BC_ATTEMPT_FREEZE`、`BR_INCREASE_PRIORITY`、`BR_TRY_ACQUIRE_RESULT` 在当前 uapi 头文件中**没有**；冻结相关命令实名是 `BC_REQUEST_FREEZE_NOTIFICATION` / `BC_CLEAR_FREEZE_NOTIFICATION` / `BC_FREEZE_NOTIFICATION_DONE`；`BC_ACQUIRE_RESULT`、`BC_ATTEMPT_ACQUIRE`、`BR_ACQUIRE_RESULT` 在头文件注释里直接标注 **"not currently supported"**。
3. **对象模型是 node / ref / handle 三层**：调用方拿到的 `handle` 是"指向另一个进程 node 的 ref"的编号；`BC_INCREFS/BC_ACQUIRE/BC_RELEASE/BC_DECREFS` 分别推动弱/强计数的内外两侧，驱动以 `BR_INCREFS/BR_ACQUIRE/BR_RELEASE/BR_DECREFS` 回投给持有者。`uapi` 结构体里**没有** `extra_size` 字段（驱动内部参数叫 `extra_buffers_size`），**也没有** `sender.sid`（安全上下文只在 `BR_TRANSACTION_SEC_CTX` + `struct binder_transaction_data_secctx` 里出现）。
4. **死亡与冻结是两套独立通知**：死亡走 `BC_REQUEST_DEATH_NOTIFICATION` → `BR_DEAD_BINDER` → 用户态回 `BC_DEAD_BINDER_DONE`；同步调用直接失败则收到 `BR_DEAD_REPLY`。冻结走 `BINDER_FREEZE(pid, enable, timeout_ms)` + `BC_REQUEST_FREEZE_NOTIFICATION`，回复侧区分 `BR_FROZEN_REPLY`（同步目标被冻结）与 `BR_TRANSACTION_PENDING_FROZEN`（异步目标被冻结）。
5. **缓冲区与异步空间是真实的容量边界，不是抽象建议**：每个进程的映射区由 `BINDER_VM_SIZE`（libbinder 中为 `1MB - 2 * 页大小`）决定；驱动为每个进程维护 `free_async_space`，专用于异步（oneway）事务；耗尽表现为 `BR_FAILED_REPLY`，libbinder 映射为 `FAILED_TRANSACTION`。oneway 洪泛有专门的检测通道：`BINDER_ENABLE_ONEWAY_SPAM_DETECTION`（默认在 libbinder 侧为 `DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION 1`）→ `BR_ONEWAY_SPAM_SUSPECT`，libbinder 打印 "Process seems to be sending too many oneway calls." 并用 `CallStack::logStack("oneway spamming")` 落栈。
6. **线程池只能长大、不能缩小**：`DEFAULT_MAX_BINDER_THREADS` 为 **15**；`setThreadPoolMaxThreadCount` 的注释明确"线程池大小永远不能减小"，`startThreadPool` 是"1 条 + setThreadPoolMaxThreadCount 请求的 N 条 + 手动 join 的 M 条"（源码示例 3+1+1=5）。驱动在没有线程待命时发 `BR_SPAWN_LOOPER`。对审计而言，**"目标进程池已满"是一类与"目标崩溃"完全不同的失效**，症状是延迟而不是 `DEAD_OBJECT`。
7. **身份的传递方式决定了权限校验能不能被信任**：驱动只填 `sender_euid` 与 `sender_pid`；安全上下文经 `BR_TRANSACTION_SEC_CTX` 单独送达。libbinder 的 `getCallingSid()` 只有在服务端调用过 `Binder::setRequestingSid` 时才非空，且 `clearCallingIdentity()` 注释写明"for legacy reasons 忽略 mCallingSid"、`restoreCallingIdentity()` 会把 `mCallingSid` 置空（"没有足够数据恢复"）。`getCallingPid()` 的官方警告：**oneway 事务收不到 PID**，行为不良的客户端可以把它变成 0。Java 公开 API 里**没有** `Binder.getCallingSid()`。
8. **SELinux 对 binder 只定义了四个权限**：`impersonate`、`call`、`set_context_mgr`、`transfer`（`system/sepolicy/private/access_vectors`）。常见引用里的 `binder { transaction acquire set_rlimit }` 并不存在。`call` 检查发生在"调用方域 → 目标域"，`transfer` 发生在句柄/fd 中转，`impersonate` 只在 `current_sid() != fromsid` 时检查；`set_context_mgr` 被 neverallow 限定给 `servicemanager`/`hwservicemanager`/`vndservicemanager`。ioctl 侧还有 `allowxperm domain binder_device:chr_file ioctl { unpriv_binder_ioctls }` 白名单，**`BINDER_FREEZE` 被刻意排除在外**。
9. **节点供给已经 binderfs 化**：Kconfig 为 `ANDROID_BINDER_IPC`（依赖 MMU）、`ANDROID_BINDERFS`、`ANDROID_BINDER_DEVICES`（默认 `"binder,hwbinder,vndbinder"`）；`init.rc` 执行 `mount binder binder /dev/binderfs stats=global`、`chmod 0666 /dev/binderfs/hwbinder`，并把 `/dev/binder` 做成指向 binderfs 的符号链接。feature 实名只有四个：`oneway_spam_detection`、`extended_error`、`freeze_notification`、`transaction_report`。
10. **接口稳定性是一层独立的工程约束**：`Stability.h` 区分"接口稳定性"（libbinder 跟踪）与"线格式稳定性"（普通 binder 调用**不**跟踪）；`@VintfStability` 与 `stability: "vintf"` 声明，`aidl_api/<name>/<V>` 冻结版本，方法只能追加。Stable AIDL 出现在 Android 10，AIDL 成为 HAL 接口在 Android 11，HIDL 自 Android 10 起被标注弃用；合规由 `vts_treble_vintf_vendor_test` / `vts_treble_vintf_framework_test` 把关。
11. **Binder 也能跑在 socket 上**：`RpcServer` 支持 `setupUnixDomainServer` / `setupVsockServer` / `setupInetServer` / `setupRawSocketServer` 与 bootstrap 会话；线协议版本常量为 `RPC_WIRE_PROTOCOL_VERSION = 1`、`_NEXT = 2`、`_EXPERIMENTAL = 0xF0000000`；fd 传输模式要显式协商（`setSupportedFileDescriptorTransportMode`）。这为无法访问 binder 节点的场景提供了第二条 IPC 路径，也带来"实验协议不得在 Android 之外使用"之类的工程护栏。
12. **服务可见性与按需启动是被 SELinux 精确约束的**：`service_manager` 权限只有 `add`、`find`、`list`；servicemanager 自己解析目标上下文（`selabel_lookup(..., SELABEL_CTX_ANDROID_SERVICE)`）再 `selinux_check_access`，调用方 sid 取 `getCallingSid()`，取不到才回落到 `getPidcon(pid)`。找不到服务时它会 `SetProperty("ctl.interface_start", "aidl/" + name)` 触发 lazy AIDL 服务启动，这一步同样需要策略授权（`set_prop(servicemanager, ctl_interface_start_prop)`）。
13. **生产设备的可观测边界比看起来窄**：binder 状态与统计节点（`stats`、`state`、`transactions`、`transaction_log` 等，模式 0444）挂在 debugfs 上，而官方要求 Android 12 起 user 版本**不挂载 debugfs**、GTS 会检查；binderfs 的 `binder_logs` 镜像只有在挂载带 `stats=global` 时出现，且 `binderfs_logs_transactions` / `_transaction_history` / `_stats` 被 neverallow 限定给极少数域，理由注释是"**Binderfs logs contain sensitive information about other processes**"。结论：审计应以"策略与状态自洽性"为主证据，不要指望在生产 user 版本上读到事务日志。
14. **驱动缺陷类别的历史是公开且有界的**：安全公告中 binder 驱动条目呈"EoP / High / Upstream kernel 或 Binder"形态（本文第十七章列出 7 条公告行）。Google 的定性表述是"最严重的一类可在**无需额外执行权限**的情况下导致本地提权"。与此同时，内核侧正在把 binder 用 Rust 重写（`drivers/android/binder/` 下有 `thread.rs`、`deferred_close.rs`、`stats.rs`、`netlink.rs` 等，并以模块参数 `impl`/`active_impl` 选择实现），这是内存安全意义上的缓解方向，不是"缺陷已消失"的证明。
15. 本文不提供利用步骤；所有接口名、命令名、常量值、属性与策略规则均取自本会话逐条访问的 uapi 头文件、AOSP 源码与官方文档（第二十二章），无法核实者显式标注"未公开文档化"。

---

## 二、范围与分层定位

### 2.1 本文覆盖与不覆盖

| 主题 | 本文 | 去处 |
| --- | --- | --- |
| BC_/BR_ 命令集、`BINDER_WRITE_READ` 词表 | ✅ 第三章 | — |
| node/ref/handle、引用计数、死亡与冻结通知 | ✅ 第四章 | — |
| 事务缓冲区大小、`free_async_space`、oneway 洪泛 | ✅ 第五章 | — |
| 线程池上限、循环控制、调度与 RT 继承 | ✅ 第六章 | — |
| 调用方身份如何进入目标进程 | ✅ 第七章 | Framework 篇的越权缺陷类型学 |
| binder 的 SELinux 权限与服务可见性 | ✅ 第八章 | 架构篇的域/类型总论 |
| binderfs 与设备节点供给 | ✅ 第九章 | 内核篇的设备节点与 ueventd |
| Stable AIDL / `@VintfStability` / 冻结与版本 | ✅ 第十章 | 密钥栈篇的 HAL 稳定性 |
| libbinder RPC（socket 上的 binder） | ✅ 第十一章 | — |
| 服务发现、lazy AIDL 按需启动 | ✅ 第十二章 | 架构篇的 HAL 装载 |
| Java/NDK 语义与异常码 | ✅ 第十三章 | — |
| 可观测性边界与取证可行性 | ✅ 第十四、十六章 | 内核篇的 debugfs/eBPF |
| 三个 binder 域的形状、Parcel「数据即指令」 | ❌ 只交叉引用 | 架构篇第六章 |
| binder 缺陷的 CVE 案例复盘 | ❌ 只引用公告字段 | 内核漏洞与提权篇 |

### 2.2 版本适用性说明

- 内核侧结论以 GKI `android-mainline`（6.1/6.6/6.12，对应 Android 14–16）与 `android16-6.12` 的 uapi 头为准；较老内核（5.4/5.10）上的命令集是其子集，`BINDER_GET_EXTENDED_ERROR`、冻结通知等新命令不一定存在。
- 用户态结论取自 `frameworks/native` 的 `main` 与 `android16-release` 分支；属性与默认值可能随 OEM 定制变化。
- 策略结论取自 `system/sepolicy` 的 `main`。

---

## 三、一次调用的解剖：ioctl 与命令集

### 3.1 唯一的每调用 ioctl

```text
ioctl(fd, BINDER_WRITE_READ, &binder_write_read)
        │
        ├── 写侧：把一串 BC_* 命令（含 read_transaction 的数据缓冲指针）交给驱动
        └── 读侧：取回一串 BR_* 命令（含事务数据 / 回复 / 状态）
```

配置与查询类 ioctl（取自 uapi 头文件，编号逐字）：

| ioctl | 定义 | 作用 |
| --- | --- | --- |
| `BINDER_WRITE_READ` | `_IOWR('b', 1, struct binder_write_read)` | 唯一的每调用入口 |
| `BINDER_SET_IDLE_TIMEOUT` | `_IOW('b', 3, __s64)` | 空闲超时 |
| `BINDER_SET_MAX_THREADS` | `_IOW('b', 5, __u32)` | 允许驱动派生的最大线程数 |
| `BINDER_SET_CONTEXT_MGR` | `_IOW('b', 7, __s32)` | 成为该节点的服务管理器 |
| `BINDER_THREAD_EXIT` | `_IOW('b', 8, __s32)` | 线程退出循环 |
| `BINDER_VERSION` | `_IOWR('b', 9, struct binder_version)` | 协议版本；注释"driver protocol version — 有不兼容变更时递增" |
| `BINDER_GET_NODE_DEBUG_INFO` | `_IOWR('b', 11, …)` | 遍历节点调试信息 |
| `BINDER_GET_NODE_INFO_FOR_REF` | `_IOWR('b', 12, …)` | 查询 ref 对应节点信息 |
| `BINDER_SET_CONTEXT_MGR_EXT` | `_IOW('b', 13, struct flat_binder_object)` | 带对象信息的上下文管理器注册 |
| `BINDER_FREEZE` | `_IOW('b', 14, struct binder_freeze_info)` | 冻结目标进程（`pid`/`timeout_ms`/`marker`） |
| `BINDER_GET_FROZEN_INFO` | `_IOWR('b', 15, struct binder_frozen_status_info)` | 查询冻结状态 |
| `BINDER_ENABLE_ONEWAY_SPAM_DETECTION` | `_IOW('b', 16, __u32)` | 开启本进程的 oneway 洪泛检测 |
| `BINDER_GET_EXTENDED_ERROR` | `_IOWR('b', 17, struct binder_extended_error)` | 取回扩展错误（`id`/`command`/`param`） |

需要点名的两个"常见误解"：`BINDER_CLEAR_ACACHE` 在 AOSP `android-mainline` 与 bionic master 头文件里**已不存在**（移除时间未公开文档化）；`BINDER_REQUEST_BINDER_DEVICE` **从未存在**，真正的"造节点"机制是 binderfs 的 `BINDER_CTL_ADD`。

### 3.2 出向命令（BC_*）

驱动接收的命令名（逐字，来自 uapi 头文件）：

```text
BC_TRANSACTION  BC_REPLY  BC_ACQUIRE_RESULT  BC_FREE_BUFFER
BC_INCREFS  BC_ACQUIRE  BC_RELEASE  BC_DECREFS
BC_INCREFS_DONE  BC_ACQUIRE_DONE  BC_ATTEMPT_ACQUIRE
BC_REGISTER_LOOPER  BC_ENTER_LOOPER  BC_EXIT_LOOPER
BC_REQUEST_DEATH_NOTIFICATION  BC_CLEAR_DEATH_NOTIFICATION  BC_DEAD_BINDER_DONE
BC_TRANSACTION_SG  BC_REPLY_SG
BC_REQUEST_FREEZE_NOTIFICATION  BC_CLEAR_FREEZE_NOTIFICATION  BC_FREEZE_NOTIFICATION_DONE
```

头文件注释给出的语义要点：`BC_FREE_BUFFER` 的参数是"读到的事务数据指针"（把缓冲归还驱动）；`BC_INCREFS`/`BC_ACQUIRE`/`BC_RELEASE`/`BC_DECREFS` 参数是"descriptor"（即 handle）；`BC_REQUEST_DEATH_NOTIFICATION` 参数是"handle + cookie"；`BC_ACQUIRE_RESULT`/`BC_ATTEMPT_ACQUIRE`/`BR_ACQUIRE_RESULT` 明确标注 **"not currently supported"**。

### 3.3 入向命令（BR_*）

```text
BR_ERROR  BR_OK  BR_TRANSACTION_SEC_CTX  BR_TRANSACTION  BR_REPLY
BR_ACQUIRE_RESULT  BR_DEAD_REPLY  BR_TRANSACTION_COMPLETE
BR_INCREFS  BR_ACQUIRE  BR_RELEASE  BR_DECREFS  BR_ATTEMPT_ACQUIRE
BR_NOOP  BR_SPAWN_LOOPER  BR_FINISHED
BR_DEAD_BINDER  BR_CLEAR_DEATH_NOTIFICATION_DONE
BR_FAILED_REPLY  BR_FROZEN_REPLY  BR_ONEWAY_SPAM_SUSPECT
BR_TRANSACTION_PENDING_FROZEN  BR_FROZEN_BINDER  BR_CLEAR_FREEZE_NOTIFICATION_DONE
```

语义摘读（头文件注释）：`BR_DEAD_BINDER`——"目标已经不在了"；`BR_FAILED_REPLY`——"上一次事务失败（例如内存不足）"；`BR_ONEWAY_SPAM_SUSPECT`——"本进程向同一目标发了太多 oneway，异步事务将使异步缓冲超出检测阈值"；`BR_FROZEN_REPLY`——"最后一次同步事务的目标被冻结"；`BR_TRANSACTION_PENDING_FROZEN`——"最后一次异步事务的目标被冻结"。

> 审计读法：`BR_DEAD_REPLY` 与 `BR_FAILED_REPLY` 的处置方式完全不同——前者是对象生命周期问题（对端进程消失），后者是容量/资源问题（缓冲区或内存）。把两者混成"调用失败"会错过一类可预防的稳定性与压力问题。

### 3.4 事务数据结构

`struct binder_transaction_data` 的成员（逐字）：`target`（`union { __u32 handle; binder_uintptr_t ptr; }`）、`cookie`、`code`、`flags`、`sender_pid`、`sender_euid`、`data_size`、`offsets_size`、`data`（`union { struct { binder_uintptr_t buffer; binder_uintptr_t offsets; } ptr; __u8 buf[8]; }`）。

- **没有 `extra_size` 成员**；驱动内部的对应参数名为 `extra_buffers_size`（`binder_alloc_new_buf(…, size_t extra_buffers_size, …)`）。
- `offsets` 的含义（头文件原文）："随 binder 事务一起提供的 `offsets` 指向数据中出现这些结构的位置"——即 Parcel 里内嵌对象的位置表。
- 内嵌对象类型：`BINDER_TYPE_BINDER`、`BINDER_TYPE_WEAK_BINDER`、`BINDER_TYPE_HANDLE`、`BINDER_TYPE_WEAK_HANDLE`、`BINDER_TYPE_FD`、`BINDER_TYPE_FDA`、`BINDER_TYPE_PTR`。
- 相关结构：`struct binder_object_header { __u32 type; }`、`struct flat_binder_object { hdr; flags; union { binder; handle }; cookie; }`、`struct binder_ptr_cookie`、`struct binder_fd_array_object { hdr; pad; num_fds; parent; parent_offset; }`、`struct binder_transaction_data_secctx { transaction_data; secctx; }`。
- `struct binder_object` / `binder_big_object`：本次未在 uapi 头文件中检索到，**不作引用**。
- 事务标志：`TF_ONE_WAY`、`TF_ROOT_OBJECT`、`TF_STATUS_CODE`、`TF_ACCEPT_FDS`、`TF_CLEAR_BUF`、`TF_UPDATE_TXN`。
- "Binder 是零拷贝/一次拷贝"这类表述：AOSP 与内核文档均未给出该断言（**未公开文档化**），本文不写。可核实的只有：数据被复制进事务缓冲区，且映射区大小受 `BINDER_VM_SIZE` 限制。

---

## 四、对象与生命周期：node / ref / handle、死亡、冻结

### 4.1 引用计数的四个动作

| 命令 | 语义 | 回投 |
| --- | --- | --- |
| `BC_INCREFS` | 增加弱引用（外部） | 目标侧收 `BR_INCREFS` |
| `BC_ACQUIRE` | 弱转强（外部） | `BR_ACQUIRE` |
| `BC_RELEASE` | 强转弱（外部） | `BR_RELEASE` |
| `BC_DECREFS` | 减少弱引用（外部） | `BR_DECREFS` |

驱动内部用 `internal_strong_refs`、`local_weak_refs` 等字段镜像这些计数（`binder_internal.h`）。用户态的对偶动作是 `BC_INCREFS_DONE` / `BC_ACQUIRE_DONE`——通知驱动"我已经处理完这次增/ acquisitions 的通知"。

工程含义：**跨进程持有的不是一次地址，而是一条 ref**。`BC_RELEASE`/`BC_DECREFS` 没有正确配对，表现为目标进程的对象滞留（内存增长）而不是崩溃；这是"句柄泄漏"类问题在 Binder 里的真实形态。

### 4.2 死亡通知的三步与两种失败

```text
注册：BC_REQUEST_DEATH_NOTIFICATION(handle, cookie)
取消：BC_CLEAR_DEATH_NOTIFICATION(handle, cookie) → BR_CLEAR_DEATH_NOTIFICATION_DONE
触发：BR_DEAD_BINDER(cookie)  ──用户态处理完──> BC_DEAD_BINDER_DONE(cookie)
```

另一条路径：同步调用打到已死目标时直接收 `BR_DEAD_REPLY`（libbinder 映射为 `DEAD_OBJECT`）。

libbinder 侧的显式限制（`IBinder.h` 注释）："**只有远程 binder 才会收到死亡通知**；对本地 binder 使用会得到 `INVALID_OPERATION`"；`unlinkToDeath` 在对端已死时返回 `DEAD_OBJECT`。NDK 对偶接口是 `AIBinder_linkToDeath` / `AIBinder_unlinkToDeath` / `AIBinder_DeathRecipient_setOnUnlinked`。服务可用性回调走 `IServiceManager::registerForNotifications` / `unregisterForNotifications`，而不是自行轮询。

延迟释放：驱动在 fd close 路径上做 `binder_deferred_release`；Rust 侧把它形式化为 `deferred_close.rs`（"Logic for closing files in a deferred manner"，`DeferredFdCloser::close_fd`）——**在 binder 上下文中直接 close fd 会阻塞正在持锁的事务**，因此延后。审计看到"close 出现在 binder 回调线程上"的实现，属于可解释的性能风险。

### 4.3 冻结：同步与异步的不同失败形态

| 机制 | 用途 |
| --- | --- |
| `BINDER_FREEZE(pid, enable, timeout_ms)` | 请求冻结目标进程，等待其进入可冻结边界 |
| `BC_REQUEST_FREEZE_NOTIFICATION` / `BC_CLEAR_FREEZE_NOTIFICATION` / `BC_FREEZE_NOTIFICATION_DONE` | 冻结状态变更通知的注册与确认 |
| `BR_FROZEN_REPLY` | 同步事务目标被冻结 → libbinder 转为 `FAILED_TRANSACTION` |
| `BR_TRANSACTION_PENDING_FROZEN` | 异步事务目标被冻结 → libbinder 记录 "Sending oneway calls to frozen process" |
| `BR_FROZEN_BINDER` | 冻结通知的携带 |
| `BINDER_GET_FROZEN_INFO` | 查询冻结信息；libbinder 有 `getProcessFreezeInfo` |
| NDK 回调 | `AIBinder_FrozenStateChangeCallback_new`（仅远程 binder、弱引用语义） |

策略约束（可核验）：`BINDER_FREEZE` **不在** `unpriv_binder_ioctls` 白名单里（`system/sepolicy/public/ioctl_macros`），即普通域无权对 binder 设备发起冻结 ioctl；这既是能力限制也是防滥用设计。

> 结论形态：现代 Android 上"对端进程被冻结"是一类独立的调用失败原因，应与"对端崩溃"分开统计。缓存被冻结进程的错误码并退避，是治理层面可执行的动作。

---

## 五、事务缓冲区、异步空间与 oneway 背压

### 5.1 三个大小限制

| 名称 | 取值/来源 | 约束对象 |
| --- | --- | --- |
| `BINDER_VM_SIZE` | libbinder：`((1*1024*1024) - sysconf(_SC_PAGE_SIZE)*2)` | 每进程 binder 映射区（≈1 MB 减 2 页） |
| `data_size` / `offsets_size` | `binder_transaction_data` | 单事务的数据与对象位置表 |
| `free_async_space` | `struct binder_alloc`（驱动内部） | 每进程**异步事务**可用预算 |

异步预算的归还方式（上游补丁正文）：`alloc->free_async_space += buffer_size + sizeof(struct binder_buffer);`。也就是说，缓冲区被释放（`BC_FREE_BUFFER`）才恢复额度。

**未公开文档化**：`AS_NORMAL`/`AS_GLOBAL`/`AS_ONEWAY` 之类的枚举名、`binder_alloc_free_size` 作为可调参数、`debug.binder.*` 属性、以及 binder 相关的 `task_profiles` 条目（AOSP `task_profiles.json` 中没有）——本文不写成事实。

### 5.2 耗尽与洪泛

- 耗尽 → `BR_FAILED_REPLY`（"例如内存不足"），libbinder 映射 `FAILED_TRANSACTION`。
- 洪泛检测：`BINDER_ENABLE_ONEWAY_SPAM_DETECTION`（libbinder 默认 `DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION 1`，`ProcessState::enableOnewaySpamDetection()`，通过 `/dev/binderfs/features/oneway_spam_detection` 探测能力）→ 超出阈值时 `BR_ONEWAY_SPAM_SUSPECT` → libbinder 打印 `Process seems to be sending too many oneway calls.` 并 `CallStack::logStack("oneway spamming")`。
- binderfs feature 文件（实名）：`oneway_spam_detection`、`extended_error`、`freeze_notification`、`transaction_report`。

治理含义：**oneway 不是"免费"的调用**。它占用与同步事务同一映射区中的异步预算；发送方与接收方之间没有回复通道，因此背压只能靠检测阈值与日志暴露。任何"高频埋点用 oneway 打给系统服务"的设计都应把 `BR_ONEWAY_SPAM_SUSPECT` 当作预期事件来监控。

### 5.3 敏感数据与 fd

- `TF_CLEAR_BUF`（libbinder 常量 `FLAG_CLEAR_BUF`）：事务缓冲在调用后清零，用于密钥材料等敏感数据。
- `TF_ACCEPT_FDS`：接收方拒绝接收 fd 的开关。
- fd 数组通过 `BINDER_TYPE_FDA` 表达（"描述需要修复的一组文件描述符，内核需区别处理"），驱动按事务记录 `fd_fixups`；tracepoint 为 `binder_transaction_fd_send` / `binder_transaction_fd_recv`。
- 跨域 fd 传递的具体数值限制与规则：**未公开文档化**。
- Parcel 侧的 fd 相关接口是 `writeFileDescriptor(int fd, bool takeOwnership)` / `readFileDescriptor()` / `writeNativeHandle()`；ashmem 归属通过 `BINDER_TYPE_FD` 对象计数（`ashmem_valid`、`ashmem_get_size_region`、`getOpenAshmemSize()`）。"通过 Parcel 传大块共享内存"在可观测面上表现为 ashmem/匿名共享内存的额度增长，而不是单次调用变慢。

---

## 六、线程池、循环与调度

### 6.1 谁在等待事务

| 机制 | 事实 |
| --- | --- |
| 默认上限 | `DEFAULT_MAX_BINDER_THREADS = 15`（`ProcessState.cpp`） |
| 池只能增 | `setThreadPoolMaxThreadCount` 注释："线程池大小永远不能减小"；库代码不应调用 |
| 组合语义 | `startThreadPool()` = 1 条 + `setThreadPoolMaxThreadCount` 请求的 N 条 + 手动 join 的 M 条（源码示例 3+1+1=5） |
| 驱动派生 | 无待命线程时驱动发 `BR_SPAWN_LOOPER`，由 libbinder 派生 `ThreadPoolThread` |
| 应用线程 | `BC_REGISTER_LOOPER` / `BC_ENTER_LOOPER` / `BC_EXIT_LOOPER`（注释："随应用级线程进入/退出 binder 循环发送"），退出用 `BINDER_THREAD_EXIT` |
| 背压 | `IPCThreadState::blockUntilThreadAvailable()`："阻塞直到执行中的 binder 线程数小于本进程允许的最大值" |
| 遗留 API | `spawnPooledThread(bool)` 标注 "TODO: deprecate" |

### 6.2 对象与线程的对应关系

- `ProcessState` 是**每进程**的（且仅用于内核 binder）；`IPCThreadState` 是**每线程**的（源码注释"Kernel binder process/thread state，按进程/线程分配"）。
- `BBinder` 是服务端 node，`BpBinder` 是客户端代理。
- 服务端要拿到调用方安全上下文需 `Binder::setRequestingSid(true)`；`isRequestingSid()` 可读回。`setInheritRt` 控制是否继承实时调度属性。
- `IPCThreadState::disableBackgroundScheduling(bool)` 的注释说明它是为 Android system 进程准备的——"不希望在后台时于其服务里持锁"。

### 6.3 优先级传播的真实状态

可核实的只有一条：`/dev/binder` 域上的实时优先级继承被官方文档标注为关闭（理由是 Android 性能团队认为其有非预期副作用）。至于"异步事务优先级提升"的具体机制名、`BR_INCREASE_PRIORITY` 之类的命令，**均不存在或未公开文档化**。

> 因此，"通过 Binder 把低优先级调用方提升到系统服务实时优先级"这类模型不成立。跨进程调度的现实约束是：目标进程池容量（第六章 6.1）+ cgroup 调度（见架构篇资源层）。

---

## 七、身份如何穿过内核

### 7.1 内核给了什么

```text
驱动填充（binder.c）：
  t->sender_euid = current_euid()
  trd->sender_euid / trd->sender_pid
  tr.secctx = t->security_ctx          → 经 BR_TRANSACTION_SEC_CTX 投递
```

即：`struct binder_transaction_data` 只有 `sender_pid` 与 `sender_euid`；**没有 gid 字段，也没有 `sender.sid`**。安全上下文出现在 `struct binder_transaction_data_secctx` 的 `secctx` 成员上，需要服务端声明才会拿到（见 7.2）。

### 7.2 libbinder 暴露与陷阱

| API | 事实与陷阱（源码注释） |
| --- | --- |
| `getCallingUid()` | 不在 binder 调用上下文中时返回 0 |
| `getCallingPid()` | 不在调用中时返回 `getpid()`；**oneway 事务收不到 PID**，行为不良客户端以异步方式发送可导致 PID 为 0 |
| `getCallingSid()` | 除非服务端调用过 `Binder::setRequestingSid`，否则返回 `nullptr`；一旦清除就无法恢复 |
| `clearCallingIdentity()` | 注释：出于 legacy 原因**忽略 `mCallingSid`** |
| `restoreCallingIdentity()` | 恢复 PID/UID（**不恢复 SID**），把 `mCallingSid` 置 `nullptr`（"没有足够数据恢复"） |
| `SpGuard` | 让依赖 `getCalling*` 的错误用法变成 abort |

Java 侧（AOSP javadoc）：PID/UID "可用于判断其身份并检查权限"；同样警告 oneway 拿不到 PID；公开 API 为 `getCallingPid/Uid/UidOrThrow/UserHandle/WorkSourceUid`，`getCallingUidOrThrow()` 在事务外抛 `IllegalStateException`；**`Binder.getCallingSid()` 不在公开 API 中**。NDK 侧为 `AIBinder_getCallingPid()` / `AIBinder_getCallingUid()`。

### 7.3 审计要点

1. **权限校验用 UID，不用 PID**：PID 可被 oneway 变成 0，也可被复用；UID 由内核填充。
2. **`clearCallingIdentity()` 之后的代码路径不得再依赖调用方身份**：SID 不可恢复，UID/PID 变成自己的。任何"clear 之后再判权限"的实现都属于可解释的越权风险（缺陷类型学见 Framework 篇）。
3. **需要按 SELinux 域做二次判定时**，服务端必须显式 `setRequestingSid`，否则连读取的入口都没有；这应作为接口设计评审的一条检查项。

---

## 八、SELinux 对 Binder 的四项权限与服务可见性

### 8.1 权限词表（`private/access_vectors`）

`binder` 类的权限**恰好**是：

```
impersonate   call   set_context_mgr   transfer
```

被反复误引的 `transaction`、`acquire`、`set_rlimit`、`set_context_manager` 在该文件中不存在。

内核钩子到权限的映射（`security/selinux/hooks.c`，`android16-6.12`）：

| 钩子 | 检查 |
| --- | --- |
| `selinux_binder_set_context_mgr` | `BINDER__SET_CONTEXT_MGR` |
| `selinux_binder_transaction` | 仅当 `current_sid() != fromsid` 时检查 `BINDER__IMPERSONATE`；随后 `BINDER__CALL`（调用方 sid → 目标 sid） |
| `selinux_binder_transfer_binder` | `BINDER__TRANSFER` |

驱动调用点位于 `drivers/android/binder.c`（`binder_transaction` 主路径与 fd/binder 对象修复处）。

### 8.2 宏与策略写法（`public/te_macros`）

| 宏 | 展开要点 |
| --- | --- |
| `binder_use` / `hwbinder_use` / `vndbinder_use` | 授予对应设备结点的访问与服务管理器能力（文件内注释：对 `/dev/binder` 的读写在 `domain.te` 里授予所有域） |
| `binder_call(client, server)` | `allow client server:binder { call transfer }; allow server client:binder transfer; allow client server:fd use;` |
| `binder_service` | 已弃用，改名为 `binderservicedomain` |
| `add_service` | `allow $1 $2:service_manager { add find };` + `neverallow { domain -… } $2:service_manager add` |

注意：不存在名为 `binding` 的宏（常见误引）。

上下文管理器保护与 ioctl 面：

```
neverallow { domain -servicemanager -hwservicemanager -vndservicemanager } *:binder set_context_mgr;
allowxperm domain binder_device:chr_file ioctl { unpriv_binder_ioctls };
```

`unpriv_binder_ioctls` 白名单在 `public/ioctl_macros` 中，`BINDER_FREEZE` 被刻意排除。

### 8.3 服务查找的权限判定在哪发生

`servicemanager` 自行完成上下文解析与判定（`cmds/servicemanager/Access.cpp`）：

1. 以 `SELABEL_CTX_ANDROID_SERVICE` 查 `selabel_lookup()` 得到目标服务的类型上下文；
2. 取调用方 sid：优先 `IPCThreadState::getCallingSid()`，取不到回落 `getPidcon(pid)`；
3. 调 `selinux_check_access(caller_sid, target_ctx, "service_manager", perm)`，`perm` 属于 `add`/`find`/`list`；
4. 被拒时以 `Status::fromExceptionCode(Status::EX_SECURITY, "SELinux denied.")` 返回。

`service_manager` 类权限只有 `add`、`find`、`list`。服务类型需带 `service_manager_type` 属性（`private/service.te`）；vendor 服务必须使用 `vndservice_manager_type`（官方 binder 使用文档）。

### 8.4 三个节点与跨域规则

| 事实 | 出处与版本 |
| --- | --- |
| 三域配置：`CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"` | Kconfig 默认值 |
| 设备侧选择节点：`ProcessState::initWithDriver("/dev/vndbinder")`；编译期由 `#ifdef __ANDROID_VNDK__` 决定 `kDefaultDriver`，节点不可读时回落 `/dev/binder`；`isVndservicemanagerEnabled()` 用 `access("/vendor/bin/vndservicemanager")` 判断 | `ProcessState.cpp` |
| `vndservicemanager` 以 `/vendor/bin/vndservicemanager /dev/vndbinder` 启动 | `cmds/servicemanager/vndservicemanager.rc` |
| Android 8：`/dev/binder` 归框架进程独占，vendor 进程不再可访问 | 官方 binder 使用文档 |
| Android 10 起同一文档补充：Stable AIDL 使所有进程都可以使用 `/dev/binder` | 同上 |
| `full_treble_only` 下 neverallow 阻止 `coredomain`（`shell`、userdebug 上的 `su` 除外）访问 `vndbinder_device:chr_file`、`vndservice_manager_type:service_manager *`、`vndservicemanager:binder *` | `private/domain.te` |

**未公开文档化**：任何通过环境变量选择节点的机制（`SERVICE_MANAGER_SHIM`、`DEV_PFS` 等在源码里不存在，`getenv` 相关计数为 0）。

---

## 九、binderfs：节点从哪里来

### 9.1 内核侧

| 项 | 事实 |
| --- | --- |
| Kconfig | `ANDROID_BINDER_IPC`（bool，依赖 `MMU`）、`ANDROID_BINDERFS`（bool）、`ANDROID_BINDER_DEVICES`（string，默认 `"binder,hwbinder,vndbinder"`） |
| 挂载 | `mount -t binder binder /dev/binderfs`（内核管理文档） |
| 造节点 | "创建 binder 设备结点的唯一途径是通过 binder-control 设备"；`BINDER_CTL_ADD = _IOWR('b', 1, struct binderfs_device)`，结构含 `char name[BINDERFS_MAX_NAME+1]`、`__u32 major`、`__u32 minor` |
| 命名空间 | `CONFIG_IPC_NS` 影响 binderfs 实例可见性（`binderfs.c`） |
| 挂载选项 | `max=`（最大设备数）、`stats=global`（把日志/统计镜像进 binderfs） |
| feature 文件 | `oneway_spam_detection`、`extended_error`、`freeze_notification`、`transaction_report` |

> 常见误引的 `FEATURE_VERSION`、`ONewaySpamDetect`、`EXT_RING_BUFFER`、`SPAM_RETRY` 在文档与源码中都不存在，不应写入报告。

### 9.2 Android 侧接线

`system/core/rootdir/init.rc` 的可见事实：

```
mount binder binder /dev/binderfs stats=global
chmod 0666 /dev/binderfs/hwbinder
symlink /dev/binderfs/binder /dev/binder
```

三条结论：

1. **`/dev/binder` 现在是符号链接**，真实结点在 `/dev/binderfs/`。审计脚本若只 `ls /dev/binder`，读到的是链接目标是否存在，而不是驱动能力。
2. **`hwbinder` 被显式 chmod 0666**：这是既有设计，不构成缺陷；跨域访问的真正闸门是第八章的 SELinux 规则。
3. `stats=global` 意味着 binder 统计/日志节点也在 binderfs 下（第十四章），可见性由策略而非文件系统位置决定。

哪个 Android 版本"强制要求" binderfs：本次未能在官方文档中核实到明确截止表述（**未公开文档化**）。

---

## 十、接口稳定性层：Stable AIDL 与 `@VintfStability`

### 10.1 两种稳定性

`libs/binder/include/binder/Stability.h` 的原文区分：

- **接口稳定性（interface stability）**——由 libbinder 跟踪；
- **线格式稳定性（wire format stability）**——普通 binder 调用**不**跟踪。

层级与降级 API：`forceDowngradeToLocalStability`、`forceDowngradeToVendorStability`、`forceDowngradeToSystemStability`；注释规定只有 `AIBinder_forceDowngradeToLocalStability()` 应调用后两者，因为 `getLocalLevel()` 在 `libbinder_ndk`（即使位于 vendor 分区）里返回 `Level::SYSTEM`。

其他关键成员：`markCompilationUnit`、`markVintf`（"保证多年稳定，即 Stable AIDL"）、`markVndk`（在未定义 `__ANDROID_VNDK__` 时调用属未定义行为，"很可能在 GSI 测试时让设备损坏"）、`requiresVintfDeclaration`。

### 10.2 版本、冻结与生成物

| 项 | 事实 |
| --- | --- |
| Stable AIDL 出现 | Android 10 增加支持 |
| AIDL 作为 HAL 接口 | Android 11 起 |
| HIDL 状态 | "As of Android 10, HIDL is deprecated" |
| 声明方式 | 每个类型加 `@VintfStability`，`aidl_interface` 模块设 `stability: "vintf"` |
| 冻结产物 | `aidl_api/<name>/<V>` 目录保存各冻结版本 |
| 兼容性规则 | 方法与枚举元素**只能追加**，不可删除/改序/改语义 |
| 生成类 | Java `Stub`；C++ `BnFoo`/`BpFoo`；NDK `BnFoo`；描述符经 `getInterfaceDescriptor()` |
| 合规验证 | `atest vts_treble_vintf_vendor_test`（已发布版本冻结 + VINTF 声明可用）；`vts_treble_vintf_framework_test`（抓未冻结的扩展接口） |

**未公开文档化**：`interface_hash`、`--stable` 命令行、`.aidl` 的 `added`/`interrupted` 属性——本文不写。

> 审计视角：稳定性层决定的是"这个接口能否跨分区、跨 OTA 被依赖"。一个未 `@VintfStability` 声明却被供应商实现引用的接口，是"看起来能工作、下次更新即断"的结构性风险，而不是当下的功能缺陷。

---

## 十一、libbinder RPC：socket 上的 Binder

### 11.1 能力面（`RpcServer.h` / `RpcSession.h`）

| 接口 | 事实 |
| --- | --- |
| 定位 | "一个接口的服务端，可被任意数量客户端通过 socket 连接" |
| 传输 | `setupUnixDomainServer(path)`、`setupVsockServer(bindCid, port, assignedPort)`、`setupInetServer(address, port, …)`（IPv6 为 TODO）、`setupRawSocketServer(fd)` |
| 预连接 | `setupUnixDomainSocketBootstrapServer(fd)` 与 `RpcSession::setupUnixDomainSocketBootstrapClient(fd)` 配对 |
| 线协议 | `RPC_WIRE_PROTOCOL_VERSION = 1`、`_NEXT = 2`、`_EXPERIMENTAL = 0xF0000000`、`…_EXPLICIT_PARCEL_SIZE`；`setProtocolVersion()` |
| fd 传输 | `setSupportedFileDescriptorTransportMode(...)`（枚举 `FileDescriptorTransportMode`）；两端必须协商 |
| 容量 | 可配置最大入站线程数与最大出站连接数；`ExclusiveConnection` / `ConnectionUse` |
| 安全传输 | `RpcTransportTls`、`RpcTransportCtxTls`、`RpcTransportTipcAndroid`（Trusty）、`RpcTransportCtxFactory`；证书/密钥格式头 `RpcCertificateFormat.h`、`RpcKeyFormat.h` |
| 平台内 shim | `libs/binder/ndk/include_platform/android/binder_rpc.h`（平台专用，非稳定 ABI），符号如 `ARpcServer_newInet/newVsock/newBoundSocket/start/join/shutdown/free`，以及 `VsockRpcClient`、`UnixDomainRpcClient`、`RpcPreconnectedClient`；导出清单在 `libbinder_rpc_unstable.map.txt` |
| 线程下限 | NDK `AIBinder_setMinRpcThreads` |
| Rust | `libbinder_rs` 导出 `BinderAsyncPool`、`IBinder`、`Strong/Weak`、`DeathRecipient`、`SpIBinder`、`ProcessState`、`ThreadState`、`StatusCode`、`FLAG_CLEAR_BUF`、`FLAG_PRIVATE_LOCAL`、`Accessor`/`ConnectionInfo`（`system_only`） |
| 护栏 | 日志文本"Cannot use experimental RPC binder protocol outside of Android." |

**未公开文档化 / 不存在**：`RpcAddressedTransport`、`RpcTlsTransport`、`RpcServer::AllowedService`（在 `main` 与 `android14-release` 中均检索不到对应头/方法）；RPC binder 的引入版本号。

### 11.2 为什么审计要单独看这条路径

1. **RPC binder 不经过 binder 设备节点**，因此第八章里"节点 + `binder` 类权限"的推理链在 RPC 会话上不成立；服务端可达性由 socket 的命名空间/权限决定（Unix socket 路径、vsock CID/port、inet 地址）。
2. **fd 传递需要显式协商传输模式**：默认与允许的 `FileDescriptorTransportMode` 决定"能不能把 fd 传过去"，与内核 binder 上的 `TF_ACCEPT_FDS` 语义不同。
3. **入站并发是服务端配置项**（最大入站线程/最大出站连接），因此 RPC 服务端存在与第六章同构的"池饱和"失效，但可观测面不同。
4. 平台内 C shim 名字里带 `unstable`——把它当作可依赖的对外接口本身是风险。

---

## 十二、服务发现与按需启动

### 12.1 客户端 API 的正确用法

`IServiceManager.h` 的注释给出使用规则（逐字要点）：

| API | 注释要点 |
| --- | --- |
| `getService` | **已弃用**："它会轮询 5 秒，优先用 `waitForService` 或 `checkService`"；并警告"nullptr 不表示服务不存在" |
| `checkService` | 非阻塞 |
| `waitForService` | 只在权限问题或致命错误时返回 nullptr |
| `isDeclared` / `getDeclaredInstances` / `updatableViaApex` | 声明面查询，与运行时是否存在是两件事 |
| `defaultServiceManager()` | 域内默认入口 |
| `addService(name, service, allowIsolated, dumpPriority)` | `allowIsolated` 决定隔离 UID 能否取得；`dumpPriority & FLAG_IS_LAZY_SERVICE` 标记 lazy 服务 |
| `listServices(dumpPriority)` / `tryUnregisterService` / `registerClientCallback` | 存在性枚举与生命周期回调 |

隔离门禁（`ServiceManager.cpp`）：`!service->allowIsolated && is_multiuser_uid_isolated(ctx.uid)` 时拒绝——这是 isolated 进程（如渲染沙箱）能拿到哪些服务的唯一开关。

### 12.2 lazy AIDL 服务的启动链

```text
客户端 waitForService(name)
   → servicemanager 未命中
   → tryStartService()：ALOG "…could not be found trying to start it as a lazy AIDL service"
   → SetProperty("ctl.interface_start", "aidl/" + name)
   → init 按 VINTF `interface aidl` 声明拉起（init 脚本需 disabled + oneshot）
   → 服务 AServiceManager_addService / LazyServiceRegistrar 注册
   → 引用计数归零后可退出
```

策略前提：`allow servicemanager servicemanager:set_prop ctl_interface_start_prop`（`private/servicemanager.te`）。C++ 侧 `LazyServiceRegistrar` 的注释条件很具体：注册经该对象的全部服务引用为 0 时退出；要求所有服务都注册进来、init 脚本 `oneshot` + `disabled`；`forcePersist` 用于避免竞态。

### 12.3 vndbinder 侧

`vndservicemanager` 与 `servicemanager` 同源，差别在驱动节点（`/dev/vndbinder`）与类型（`vndservice_manager_type`）；`ProcessState::isVndservicemanagerEnabled()` 通过 `access("/vendor/bin/vndservicemanager")` 判断。框架进程被 neverallow 阻止与之通信（8.4）。

> 审计要点：`service list`（见 16.1）只覆盖当前节点上的可见服务，**声明了但未启动**的 lazy 服务不会以"存在"的形式出现。把"清单里没有"直接写成"服务缺失/被禁用"是常见误判；应先用 `isDeclared` 类接口区分声明面与运行面。

---

## 十三、Java / NDK 层语义与异常码

### 13.1 `oneway` 的确切语义

| 来源 | 原文要点 |
| --- | --- |
| AIDL 语言规范 | "Oneway methods are dispatched asynchronously and cannot return a result."；"One-way methods from the same thread to the same binder also execute serially." |
| 应用开发文档 | oneway "远程调用不阻塞……发送事务数据后立即返回"；**本地调用使用 `oneway` 没有影响，仍是同步**；服务端"作为来自 Binder 线程池的普通调用收到"；因此"AIDL 接口的实现必须完全线程安全" |
| NDK | "All oneway calls are guaranteed to be ordered if they are sent on the same AIBinder object" |
| Java | `IBinder.FLAG_ONEWAY = 0x00000001`；对本地 binder "没有 oneway 语义" |

`FIRST_CALL_TRANSACTION = 0x1`，`LAST_CALL_TRANSACTION = 0x00ffffff`；保留码包括 `PING_TRANSACTION`、`DUMP_TRANSACTION`、`INTERFACE_TRANSACTION`、`SYSPROPS_TRANSACTION`、`EXTENSION_TRANSACTION`、`DEBUG_PID_TRANSACTION`、`SET_RPC_CLIENT_TRANSACTION`。libbinder 侧 `FLAG_ONEWAY` 注释："对应 `TF_ONE_WAY`——异步调用"；`FLAG_PRIVATE_VENDOR = 0x10000000` 为私有用途。

### 13.2 失败与异常码

| 常量 | 值/来源 | 语义 |
| --- | --- | --- |
| `STATUS_DEAD_OBJECT` | `-EPIPE` | 对端对象已死 |
| `STATUS_TIMED_OUT` | `-ETIMEDOUT` | 超时 |
| `STATUS_FAILED_TRANSACTION` | 见 NDK/`Status.h` | 底层事务失败（缓冲/内存/冻结） |
| `EX_TRANSACTION_FAILED` | `-129` | "特殊，表示本地 binder 代理在底层就已经失败" |
| `EX_NONE`…`EX_PARCELABLE` | `0, -1, -2, -3, -4, -5, -6, -7, -8, -9` | `EX_SECURITY=-1`、`EX_BAD_PARCELABLE=-2`、`EX_ILLEGAL_ARGUMENT=-3`、`EX_NULL_POINTER=-4`、`EX_ILLEGAL_STATE=-5`、`EX_NETWORK_MAIN_THREAD=-6`、`EX_UNSUPPORTED_OPERATION=-7`、`EX_SERVICE_SPECIFIC=-8`、`EX_PARCELABLE=-9` |
| `EX_HAS_REPLY_HEADER` | `-128` | 携带回复头 |
| libbinder 映射 | — | `BR_DEAD_REPLY→DEAD_OBJECT`；`BR_FAILED_REPLY→FAILED_TRANSACTION`；`BR_FROZEN_REPLY→FAILED_TRANSACTION`；状态码回复从 `tr.data.ptr.buffer` 直接取 |

`IServiceManager.h` 备注 `-7` 之类的异常在 `utils/Errors.h` 中没有对偶错误码。**"sticky 与非 sticky binder 状态码"的官方区分本次未核实**（`TransactionStats`/`IBinder::getTransactionStats`/`getTimeSpentInCallback` 在 `main`、`android13-release`、`android14-release` 头文件与 `developer.android.com` 参考页均检索不到，属**不存在或未公开文档化**；可核实的是驱动侧 `BINDER_GET_EXTENDED_ERROR` + `struct binder_extended_error{id, command, param}` 与 binderfs `extended_error` feature，以及 `RecordedTransaction.h` + `START_RECORDING_TRANSACTION`/`STOP_RECORDING_TRANSACTION`）。

### 13.3 服务端自我保护

| 机制 | 事实 |
| --- | --- |
| `CallRestriction::ERROR_IF_NOT_ONEWAY` / `FATAL_IF_NOT_ONEWAY` | 限制调用形态；`setCallRestriction` 注释要求"在任何线程派生前调用"（servicemanager 自己就这么用） |
| `Binder::setRequestingSid` | 决定是否接收调用方 SELinux 上下文 |
| `SpGuard` | 阻止在非调用上下文误用 `getCalling*` |
| `Parcel` StrictMode | 头文件注释：策略掩码会随事务传播；同时"native 实现不做任何 StrictMode 检查" |
| `FLAG_CLEAR_BUF` | 敏感缓冲区调用后清零 |
| 未验证 | `ALLOW_FD`/`DO_NOT_GAIN_RIGHTS`、`setParcelDataLimit`、`unpadded`、`ReplyHashes`、`flatten_binder`/`unflatten_binder`（在 `main`/`android13` 的 Parcel 头/源中检索不到；`libs/binder/` 下无 README）——本文不写 |

### 13.4 Java/NDK 对应关系

- Java：`android.os.Binder`、`BinderProxy.transactNative`/`linkToDeathNative`、`Binder.clearCallingIdentity()`/`restoreCallingIdentity(long)`（`public static final native`）、`ServiceManager.getService/checkService/addService/getServiceOrThrow`、AIDL `Stub.asInterface()`、`Context.bindService(...)`。
- NDK：`AIBinder_Class_define`、`AIBinder_linkToDeath`/`unlinkToDeath`、`AIBinder_DeathRecipient_setOnUnlinked`、`AIBinder_FrozenStateChangeCallback_new`、`AIBinder_prepareTransaction`、`AIBinder_toJavaBinder`/`fromJavaBinder`、`AServiceManager_addService`、`AParcel_readStrongBinder`/`readStatusHeader`/`readParcelFileDescriptor`。
- 未核实：`bindAidl`/`bindUtf8`、`PingListener`、`isCallingUidPrivileged`（在 `main` 的相关头文件中检索不到）。

---

## 十四、可观测性与取证的边界

### 14.1 内核侧文件

debugfs 节点（`binder.c` 中注册，模式 0444）：`state`、`state_hashed`、`stats`、`transactions`、`transactions_hashed`、`transaction_log`（另有按进程的 `proc/`）。binderfs 在带 `stats=global` 挂载时把同一批内容镜像到 `/dev/binderfs/binder_logs/`。

上游动机（LKML 补丁说明）："目前访问 binder 状态与统计的唯一途径是 debugfs。我们需要一种在未挂载 debugfs 时也能访问同样信息的方式。"

### 14.2 生产环境的三重限制

| 限制 | 事实 |
| --- | --- |
| debugfs 本身 | Android 12 起官方要求 **user 版本不挂载 debugfs**；GTS 检查；内核文件系统文档写明 Android 11 起默认不挂载、仅在 bug report 期间临时挂载 |
| binder 日志类型 | `binderfs_logs_transactions` 的 neverallow 只放行 `init`/`vendor_init`（`userdebug_or_eng` 外加 `dumpstate`/`system_server`）；`binderfs_logs_transaction_history` 限 `dumpstate`/`init`/`vendor_init`/`system_server`；`binderfs_logs_stats` 类似。理由注释："**Binderfs logs contain sensitive information about other processes**" |
| 类型引入时间 | `binderfs_logs_transactions`、`binderfs_logs_transaction_history` 仅存在于 `starting_at_board_api(202504)`（Board API 202504，Android 16 QPR） |

tracepoint 名单（`binder_trace.h` 逐字）：`binder_ioctl`、`binder_ioctl_done`、`binder_write_done`、`binder_read_done`、`binder_wait_for_work`、`binder_transaction`、`binder_transaction_received`、`binder_transaction_node_to_ref`、`binder_transaction_ref_to_node`、`binder_transaction_ref_to_ref`、`binder_transaction_fd_send`、`binder_transaction_fd_recv`、`binder_transaction_alloc_buf`（注意实名不是 `binder_transaction_alloc`）、`binder_transaction_buffer_release`、`binder_transaction_failed_buffer_release`、`binder_transaction_update_buffer_release`、`binder_update_page_range`、`binder_alloc_lru_start/end`、`binder_free_lru_start/end`、`binder_alloc_page_start/end`、`binder_unmap_user_start/end`、`binder_unmap_kernel_start/end`、`binder_command`、`binder_return`、`binder_txn_latency_free`、`binder_netlink_report`。

atrace/systrace 的类别名是 **`binder_driver`**（不是 `binder`）。

Rust 实现的观测面：`drivers/android/binder/` 下含 `stats.rs`、`debug.rs`、`trace.rs`、`netlink.rs`、`rust_binder_events.c/h`；C 侧也有 `binder_netlink.c/h`。binderfs feature `transaction_report` 与之对应。

**未公开文档化**：官方的 eBPF/内核遥测页面（`statsd` 通道）**未列出任何 binder/IPC 指标**，因此"生产 binder 指标可由内核遥测框架直接替代"不能写入结论。

### 14.3 可执行取证结论

1. user 版本上，**不要承诺能拿到事务级记录**：debugfs 不挂载、binderfs 日志被 neverallow。
2. 可稳定获得的信号是**状态与策略自洽性**：`getenforce`、SELinux 拒绝日志（enforcing 下"既记录又强制执行"，permissive 只记录）、`/dev/binderfs` 结构与 `features/`、libbinder 自己打印的三类事件（oneway 洪泛、发往冻结进程、lazy 服务拉起）、以及版本/补丁水位。
3. 事务级分析应当在 userdebug/eng 构建或有权限的采样设备上做，并把"生产不可见"写进报告的可行性章节，而不是把生产环境取不到数据写成"设备无异常"。

---

## 十五、失效模式矩阵

| 层 | 失效 | 可观测证据 | 定级建议 |
| --- | --- | --- | --- |
| 协议 | 把 `BR_FAILED_REPLY` 当"对端已死"处理 | 日志/错误码归因错误；无 `BR_DEAD_REPLY` | MED（治理与重试策略错误） |
| 缓冲区 | 异步预算耗尽（oneway 洪泛/大包） | `BR_ONEWAY_SPAM_SUSPECT`、"oneway spamming" 栈日志 | MED→HIGH（视影响面） |
| 缓冲区 | 单事务超过映射区 | 调用返回 `FAILED_TRANSACTION`，对端未退出 | MED |
| 对象 | 强/弱计数不配对 | 目标进程对象滞留、内存增长 | MED |
| 对象 | 未注册死亡通知（对远程 binder） | 上游长期持有陈旧代理、调用持续 `DEAD_OBJECT` | MED |
| 线程 | 目标进程池饱和（上限 15，且不可缩小） | 表现为延迟与 ANR，而非 `DEAD_OBJECT` | HIGH（服务侧容量问题） |
| 调度 | 期待 Binder 做 RT 优先级传播 | `/dev/binder` 域 RT 继承关闭 | LOW（认知错误，非设备缺陷） |
| 身份 | `clearCallingIdentity()` 后再判权限 | 代码评审信号；可用 Framework 篇缺陷类型学归因 | HIGH |
| 身份 | 依赖 `getCallingPid()` 做鉴权 | oneway 下 PID 为 0 | HIGH |
| 身份 | 需要 SID 却未 `setRequestingSid` | `getCallingSid()` 恒为 nullptr | MED |
| 权限 | `binder_call` 规则缺失 | 拒绝日志形如 `avc: denied { call } … tclass=binder`；`EX_SECURITY "SELinux denied."` | MED（功能被拒即暴露） |
| 权限 | 服务 `find` 未授权 | 拒绝日志形如 `avc: denied { find } … tclass=service_manager` | MED |
| 权限 | 允许非管理器 `set_context_mgr` | 与 `neverallow` 冲突（策略应拒绝）；出现即为构建异常 | HIGH |
| 隔离 | `allowIsolated` 误开 | isolated UID 能取得敏感服务 | HIGH |
| 节点 | 只检查 `/dev/binder` 存在性 | 它是 binderfs 符号链接；能力应看 `features/` 与驱动版本 | LOW（方法错误） |
| 稳定性 | 未冻结却被跨分区依赖 | `vts_treble_vintf_vendor_test` 失败或 OTA 后断裂 | HIGH（结构性） |
| 稳定性 | `markVndk` 在非 VNDK 编译单元误用 | GSI 构建期设备损坏征兆 | HIGH |
| RPC | 未协商 fd 传输模式 | fd 传递失败而调用本身可用 | MED |
| RPC | 把 `libbinder_rpc_unstable` 当稳定 ABI 依赖 | 符号名与导出清单自证 | MED |
| 发现 | 把"清单里没有"写成"服务缺失" | lazy 服务未启动时不出现；需 `isDeclared` 区分 | LOW（结论方法错误） |
| 冻结 | 未处理 `BR_FROZEN_REPLY`/`PENDING_FROZEN` | "Sending oneway calls to frozen process" 日志 | MED |
| 可观测 | user 版本期待事务日志 | debugfs 未挂载、neverallow 限定 binderfs 日志 | LOW（预期管理问题） |

---

## 十六、检测与取证采集

### 16.1 采集脚本（`binder-ipc-probe.sh`）

只读采集。设备侧可用面在生产 user 版本上受第十四章的限制，脚本会显式区分"取到"与"取不到"。

```bash
#!/usr/bin/env bash
# Binder IPC 运行时证据采集（只读）
set -u
dev="${1:-}"
[ -n "$dev" ] || { echo "用法: $0 <adb-serial>"; exit 2; }
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "设备不可达: $dev"; exit 2; }

sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
sp() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r'; }   # 多行
p()  { sh "getprop $1"; }

echo "== 版本与构建 =="
echo "release=$(p ro.build.version.release) sdk=$(p ro.build.version.sdk)"
echo "spl=$(p ro.build.version.security_patch) build_type=$(p ro.build.type)"
echo "kernel=$(sh 'uname -r')"

echo "== 节点供给 =="
echo "dev_binder_link=$(sh 'ls -l /dev/binder 2>/dev/null')"
echo "node_binder=$(sh 'ls /dev/binder 2>/dev/null') node_hw=$(sh 'ls /dev/hwbinder 2>/dev/null') node_vnd=$(sh 'ls /dev/vndbinder 2>/dev/null')"
echo "binderfs_entries=$(sp 'ls /dev/binderfs 2>/dev/null' | tr '\n' ' ')"
echo "binderfs_features=$(sp 'ls /dev/binderfs/features 2>/dev/null' | tr '\n' ' ')  # 取不到多为权限限制"

echo "== 服务可见性 =="
echo "service_list_count=$(sp 'service list' | grep -c ':')"
echo "hw_or_vnd_note=注意：以上仅覆盖当前节点可见的服务，未启动的 lazy 服务不会出现在清单中"

echo "== 日志中的运行时事件（受限：user 版本可能不可读）=="
echo "oneway_spam=$(sp 'logcat -d -t 4000' | grep -c 'oneway spamming')"
echo "oneway_spam_msg=$(sp 'logcat -d -t 4000' | grep -c 'too many oneway calls')"
echo "frozen_oneway=$(sp 'logcat -d -t 4000' | grep -c 'Sending oneway calls to frozen process')"
echo "lazy_start=$(sp 'logcat -d -t 4000' | grep -c 'trying to start it as a lazy AIDL service')"
echo "selinux_denied_binder=$(sp 'logcat -d -t 4000' | grep -Eic 'avc: +denied.*tclass=(binder|service_manager)')"

echo "== 可观测文件可见性 =="
echo "debugfs_binder=$(sp 'ls /sys/kernel/debug/binder 2>/dev/null' | tr '\n' ' ')  # 生产 user 版本预期为空"
echo "binderfs_logs=$(sp 'ls /dev/binderfs/binder_logs 2>/dev/null' | tr '\n' ' ')"

echo "== SELinux =="
echo "getenforce=$(sh 'getenforce')"
```

### 16.2 判定脚本（`binder-ipc-assert.sh`）

只输出有官方依据的判定；对无法从设备侧权威确定的项给 INFO 并指向正确判定路径。

```bash
#!/usr/bin/env bash
# 依据采集结果给出 Binder IPC 运行时发现（只读）
set -u
dev="${1:-}"
[ -n "$dev" ] || { echo "用法: $0 <adb-serial>"; exit 2; }
sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
sp() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r'; }
p()  { sh "getprop $1"; }
trim() { printf '%s' "$1" | tr -s '[:space:]' ' ' | sed 's/^ //;s/ $//'; }  # adb 输出常含空白，避免误判为非空

sdk=$(p ro.build.version.sdk)
type=$(p ro.build.type)
enf=$(sh 'getenforce')
logs=$(sp 'logcat -d -t 4000')

[ "$enf" = "Permissive" ] && echo "HIGH  SELinux=Permissive：binder 的 call/transfer/service_manager 判定不再强制执行，仅剩日志"

spam=$(printf '%s' "$logs" | grep -c 'oneway spamming')
if [ "$spam" -gt 0 ]; then
  echo "MED   近期日志出现 oneway 洪泛 ${spam} 次：存在异步缓冲压力（BR_ONEWAY_SPAM_SUSPECT 路径）"
fi
frz=$(printf '%s' "$logs" | grep -c 'Sending oneway calls to frozen process')
[ "$frz" -gt 0 ] && echo "INFO  出现发往冻结进程的 oneway ${frz} 次：需区分对端崩溃与对端冻结"

lz=$(printf '%s' "$logs" | grep -c 'trying to start it as a lazy AIDL service')
[ "$lz" -gt 0 ] && echo "INFO  lazy AIDL 按需启动发生 ${lz} 次：声明面与运行面分离，勿据此判定服务缺失"

den=$(printf '%s' "$logs" | grep -Eic 'avc: +denied.*tclass=(binder|service_manager)')
if [ "$den" -gt 0 ]; then
  echo "MED   存在 binder/服务可见性相关的 SELinux 拒绝 ${den} 条：核对 binder_call 与 service_manager 规则"
fi

dbg=$(trim "$(sp 'ls /sys/kernel/debug/binder 2>/dev/null')")
if [ "$type" = "user" ] && [ -n "$dbg" ]; then
  echo "HIGH  user 版本可见 debugfs binder 节点（${dbg}）：与官方要求及 GTS 检查不一致"
elif [ "$type" = "user" ]; then
  echo "OK    user 版本未暴露 debugfs binder 节点（符合预期）"
fi

bfs=$(trim "$(sp 'ls /dev/binderfs 2>/dev/null')")
if [ -z "$bfs" ]; then
  echo "INFO  未读到 /dev/binderfs：可能是权限限制或旧形态；能力判定应改用 binderfs features 或驱动版本"
else
  echo "OK    binderfs 可见：${bfs}"
fi

feat=$(trim "$(sp 'ls /dev/binderfs/features 2>/dev/null')")
case "$feat" in
  *extended_error*) echo "OK    驱动具备 extended_error：可用 BINDER_GET_EXTENDED_ERROR 取结构化失败原因";;
  "") : ;;
  *) echo "INFO  features 未包含 extended_error：失败归因需依赖日志与状态码映射";;
esac

link=$(sh 'ls -l /dev/binder 2>/dev/null')
case "$link" in
  *"-> /dev/binderfs"*|*"-> binderfs"*) echo "INFO  /dev/binder 为 binderfs 符号链接（正常形态）" ;;
  "") echo "INFO  未读到 /dev/binder：需结合 init 挂载顺序与权限复判" ;;
  *)  echo "INFO  /dev/binder 非 binderfs 符号链接：${link}" ;;
esac

v=$(sh 'cat /dev/binderfs/binder_logs/stats 2>/dev/null | head -1')
[ -z "$v" ] && echo "INFO  事务级日志在生产环境受策略限定（binderfs_logs_* neverallow）；无数据不等于无异常"

spl=$(p ro.build.version.security_patch)
if [ -n "$spl" ]; then
  age=$(( ( $(date +%s) - $(date -d "$spl" +%s) ) / 86400 ))
  if   [ "$age" -gt 180 ]; then echo "HIGH  SPL 落后 ${age} 天：binder 驱动历史 EoP 类修补丁未必落地"
  elif [ "$age" -gt 90 ];  then echo "MED   SPL 落后 ${age} 天"
  else echo "OK    SPL 落后 ${age} 天"; fi
fi

case "$sdk" in
  2[6-9]|1[0-9]|2[0-5]) echo "INFO  SDK ${sdk}：早于 Android 8，三域/vndservicemanager 形态不同，本文规则不直接适用";;
esac
```

### 16.3 使用约束

- `logcat`、`ls /dev/binderfs/features`、`cat …/binder_logs/*` 在 user 版本上可能因权限为空；脚本把空值一律输出为 INFO，不产出"无异常"结论。
- 未包含 `service call`：该命令可写状态，属变更操作，不在只读采集范围内。
- 跨设备对比时，`service list` 的数量差异不等于安全差异；声明面/运行面见 12.3。

---

## 十七、公开公告中的 binder 缺陷类别（仅定性转述）

| CVE | 公告 | 官方字段 |
| --- | --- | --- |
| CVE-2017-13162 | 2017-12-01 | EoP / High / Upstream kernel（binder） |
| CVE-2019-2025 | 2019-03-01 | EoP / High / Binder |
| CVE-2019-2213 | 2019-11-01 | EoP / High / Binder |
| CVE-2019-2214 | 2019-11-01 | EoP / High / Binder |
| CVE-2023-21255 | 2023-07-01 | EoP / High / Binder |
| CVE-2024-26926 | 2024-06-01 | EoP / High / Binder |
| CVE-2024-56556 | 2025-04-01 | EoP / High / Binder |

Google 在公告中的定性与建议（原文要点）："最严重的一类可在**无需额外执行 privileges** 的情况下导致本地提权"；"较新 Android 版本的平台增强使许多问题的利用更加困难，鼓励所有用户更新"；补丁以 SPL 表达（`-01` 与 `-05` 两档月度水位）。

两条方法论说明：

1. 现代公告只给 CVE/漏洞 ID、类型、严重级、组件，**不逐条给出缺陷类别词**（如"use-after-free"）。因此本文不把具体成因写成事实；成因级别的归类请参照内核漏洞与提权篇的类型学方法。
2. 同一时期存在的工程性线索可作缓解方向证据：模块参数 `stop_on_user_error`（用户错误即停）与 `debug_mask`，以及 Rust/C 实现选择参数 `impl`/`active_impl`；Rust 版驱动目录（`drivers/android/binder/`）含 `deferred_close.rs` 等模块，属内存安全重写路径。这些是**机制事实**，不构成"该类缺陷已消除"的判断。

---

## 十八、常见误区

1. **"Binder 是零拷贝"**：官方文档没有这一断言；可核实的是数据被复制进受 `BINDER_VM_SIZE` 约束的事务缓冲区。
2. **`binder { transaction acquire set_rlimit }`**：`binder` 类权限只有 `impersonate, call, set_context_mgr, transfer`。写策略时引用不存在的权限名会直接编译失败或变成无效规则。
3. **`BC_ATTEMPT_FREEZE` / `BR_INCREASE_PRIORITY`**：不存在。冻结命令实名见 4.3；优先级传播在 `/dev/binder` 域上是关闭的。
4. **`extra_size`、`sender.sid` 是 uapi 字段**：都不是。`extra_buffers_size` 是驱动内部参数；安全上下文走 `binder_transaction_data_secctx`。
5. **"oneway 是免费的异步"**：oneway 消耗异步缓冲预算，可能触发 `BR_ONEWAY_SPAM_SUSPECT`；且服务端"作为来自 Binder 线程池的普通调用"处理，实现必须线程安全。
6. **"同一目标上的 oneway 会并发执行"**：规范原文是"同一线程发往同一 binder 的一对一方法也串行执行"。
7. **"用 `getCallingPid()` 判身份"**：oneway 事务拿不到 PID，可为 0；鉴权只用 UID（或服务端显式索取的 SID）。
8. **"SID 一直可读"**：需要 `Binder::setRequestingSid`；`clearCallingIdentity()` 后不可恢复。
9. **"线程池能打满再收缩"**：池大小**永远不能减小**，`setMaxThreads`/`setThreadPoolMaxThreadCount` 是上限而非弹性。
10. **"服务清单里没有就是缺失"**：lazy 服务在首次等待时由 servicemanager 拉起，声明面与运行面要分开判定。
11. **"`getService` 返回 null 表示服务不存在"**：源码注释警告"nullptr 不表示服务不存在"，且该 API 已因轮询 5 秒被弃用。
12. **"生产设备可读 binder 事务日志"**：debugfs 不挂载、binderfs 日志被 neverallow；官方理由正是"包含关于其他进程的敏感信息"。
13. **"atrace 抓 binder 用 `binder` 类别"**：类别名是 `binder_driver`；tracepoint 实名含 `binder_transaction_alloc_buf`（不是 `binder_transaction_alloc`）。
14. **"Stable AIDL 只要加注解就稳定"**：线格式稳定性对普通 binder 调用不被 libbinder 跟踪；且必须冻结 `aidl_api` 并通过 VINTF 声明，否则 `vts_treble_vintf_*` 会失败。
15. **"RPC binder 只是换端口"**：它不经过 binder 节点，SELinux 的 `binder` 类判定链不适用；fd 传递要协商 `FileDescriptorTransportMode`；平台内实现的名字本身就写着 `unstable`。
16. **"binder 提权靠新版本已彻底解决"**：公告层面 Google 说的是"较新版本使利用更困难"，并伴随 Rust 重写等缓解方向，而非缺陷消除的声明。

---

## 十九、趋势判断（2024 → 2026）

1. **可观测性从 debugfs 迁往受策略约束的 binderfs 通道**：`stats=global` 镜像 + `binderfs_logs_*` 类型（Board API 202504 引入）+ neverallow 限定，说明方向是"提供结构化可观测性，同时明确它敏感"。审计报告要相应把"取不到事务日志"当作设计预期。
2. **失败归因正在结构化**：`BINDER_GET_EXTENDED_ERROR` + `struct binder_extended_error{id,command,param}` + feature `extended_error`，把"BR_FAILED_REPLY 为什么失败"变成可查询事实。以日志字符串为主的判定方法会被这条通道替代。
3. **冻结语义成为一等公民**：`BINDER_FREEZE`/`BR_FROZEN_REPLY`/`BR_TRANSACTION_PENDING_FROZEN`/`transaction_report` 与 NDK 冻结回调一起构成"目标被挂起"这一类独立失败。后台管控（phantom process、cached 进程）越严，这一类的占比越高。
4. **驱动实现正在 Rust 化**：`drivers/android/binder/`（`thread.rs`、`deferred_close.rs`、`stats.rs`、`netlink.rs`、`rust_binder_events`）与 `impl`/`active_impl` 参数意味着同一内核里可选实现。审计上需要记录"当前生效实现"，因为它决定内存安全假设与 debugfs/netlink 形态。
5. **接口稳定性成为跨团队工程约束**：Stable AIDL + 冻结 API + VINTF 验证已经能机械化拒绝"隐式跨分区依赖"；未来越来越多的风险会以"未冻结/未声明"的形式提前暴露，而不是运行期崩溃。
6. **RPC 通道扩张**：vsock/Unix socket/TLS 传输与 `libbinder_rs` 异步池（`BinderAsyncPool`）说明"binder 语义、非节点传输"在被用于更多隔离场景。审计清单需要把"是否存在非节点 IPC 路径"单列一项。

---

## 二十、结论

Binder 的审计问题可以收敛成五个：

1. **协议层**：调用失败的原因被正确区分了吗——`BR_DEAD_REPLY`（生命周期）、`BR_FAILED_REPLY`（容量/资源）、`BR_FROZEN_REPLY`/`PENDING_FROZEN`（冻结）、`BR_ONEWAY_SPAM_SUSPECT`（洪泛）？
2. **容量层**：映射区（`BINDER_VM_SIZE` ≈ 1MB−2 页）、异步预算（`free_async_space`）、线程池上限（默认 15，不可缩小）是否被当作可预测的边界来管理？
3. **身份层**：鉴权只用内核填充的 UID；SID 只在 `setRequestingSid` 时可得且不可恢复；PID 在 oneway 下不可用。
4. **策略层**：`binder` 的四项权限、`service_manager` 的三项权限、`set_context_mgr` 的 neverallow、`unpriv_binder_ioctls` 白名单（排除 `BINDER_FREEZE`）、Treble 的 vndbinder 隔离是否与实际接口声明一致？
5. **可观测层**：报告中每一条 IPC 结论是否落在生产 user 版本**实际可读**的面上（策略自洽性、状态结构、libbinder 日志三类事件），而不是落在只在 userdebug 存在的事务日志上？

这五问全部可答的报告，才配得上"Binder 机制审计"这个标题；只统计 `service list` 数量的，是清单，不是审计。

---

## 二十一、常见问题（FAQ）

**Q1：Binder 与 socket/共享内存相比，安全上的差别在哪？**
差别不在"更快"，而在**内核知道每一次调用的发起者身份**（`sender_euid`）并且 SELinux 能在事务发生的瞬间做"调用方域→目标域"的 `call` 判定。socket 上没有这层内建身份。

**Q2：我们偶发 `FAILED`，怎么判断是不是 Binder 容量问题？**
看是否伴随 `oneway spamming` 日志、是否有大批 oneway 目标同一服务；再用 `BINDER_GET_EXTENDED_ERROR`（feature 存在时）取结构化原因。对端进程未退出而调用失败，是容量类问题的典型形态（区别于 `DEAD_OBJECT`）。

**Q3：能否通过 Binder 让服务端以更高实时优先级处理我的请求？**
不能。`/dev/binder` 域的实时优先级继承被官方文档标注为关闭；服务端并发受其线程池上限约束。

**Q4：`service list` 看不到某服务，能判定它被禁用吗？**
不能。lazy AIDL 服务首次等待时才拉起；应先看 `isDeclared`/VINTF 声明面，再看 `addService(..., dumpPriority & FLAG_IS_LAZY_SERVICE)` 的运行面。

**Q5：审计时能拿到 Binder 事务记录吗？**
生产 user 版本预期拿不到：debugfs 不挂载（GTS 检查），`binderfs_logs_*` 被 neverallow 限定给 init/dumpstate/system_server 等少数域且带敏感理由注释。事务级分析请在 userdebug/eng 或有权限的采样设备上做。

**Q6：`oneway` 方法能否返回数据？**
不能，AIDL 规范原文即"cannot return a result"。同一线程到同一 binder 的一对一调用串行执行；服务端把它当线程池里的普通调用处理，因此实现必须线程安全。

**Q7：跨分区接口需要注意什么？**
`@VintfStability` + `stability: "vintf"` + 冻结 `aidl_api/<name>/<V>`，方法只追加；合规由 `vts_treble_vintf_vendor_test` 把关。注意 `Stability.h` 的区分：普通 binder 调用不跟踪线格式稳定性。

**Q8：vendor 进程能连 `/dev/binder` 吗？**
Android 8 起 `/dev/binder` 归框架；但同一文档在 Android 10 补充 Stable AIDL 允许所有进程使用 `/dev/binder`。实际判定看 `full_treble_only` 的 neverallow 与 `vndbinder_use`/`vndservice_manager_type` 规则，不看机型宣传。

**Q9：为什么本文不引用 `TransactionStats`？**
本次在 `main`、`android13-release`、`android14-release` 的 libbinder 头文件与 `developer.android.com` 参考页均检索不到；能核实的替代物是驱动的 `BINDER_GET_EXTENDED_ERROR` 与 `RecordedTransaction.h`/`START_RECORDING_TRANSACTION`。宁缺不假。

**Q10：Binder 相关的提权洞还值得单独关注吗？**
值得，但方式不是找利用：公告显示它是持续产出 EoP/High 的组件。审计价值在于（1）SPL 是否覆盖历史 binder 修补丁；（2）策略是否把可被任何 shell/app 触达的服务面收窄；（3）目标服务的线程池与异步预算是否被设计为可被压满。

---

## 二十二、参考资料（公开来源，本会话逐条访问核实）

内核 uapi 与驱动：

1. binder uapi 头（AOSP GKI `android-mainline`） — https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/include/uapi/linux/android/binder.h
2. binder uapi 头（bionic 用户态视角） — https://android.googlesource.com/platform/bionic/+/refs/heads/master/libc/kernel/uapi/linux/android/binder.h
3. binder uapi 头（Linux mainline） — https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/include/uapi/linux/android/binder.h
4. `drivers/android/binder.c`（`android16-6.12`） — https://android.googlesource.com/kernel/common/+/refs/heads/android16-6.12/drivers/android/binder.c
5. `drivers/android/binder_alloc.c` / `binder_alloc.h` — https://android.googlesource.com/kernel/common/+/refs/heads/android16-6.12/drivers/android/binder_alloc.c ； https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/android/binder_alloc.h
6. `binder_internal.h`（`free_async_space`、`fd_fixups`、`oneway_spam_detection_enabled`） — https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder_internal.h
7. `binder_trace.h`（tracepoint 名单） — https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/android/binder_trace.h
8. `binderfs.c`（feature 文件、`stats=global`） — https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/android/binderfs.c
9. `binderfs.h`（`struct binderfs_device`、`BINDER_CTL_ADD`） — https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/include/uapi/linux/android/binderfs.h
10. `drivers/android/Kconfig` — https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/Kconfig
11. Rust 版 binder（`thread.rs`/`deferred_close.rs`/`stats.rs`/`netlink.rs`） — https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder/
12. binderfs 管理指南 — https://docs.kernel.org/admin-guide/binderfs.html
13. LKML：把 binder 统计/日志暴露到 binderfs 的动机 — https://lkml.org/lkml/2019/9/4/1128
14. 异步空间记账修复补丁正文（`free_async_space` 归还式） — https://patchew.org/linux/20211220190150.2107077-1-tkjos@google.com/
15. SELinux binder 钩子（`hooks.c`） — https://android.googlesource.com/kernel/common/+/refs/heads/android16-6.12/security/selinux/hooks.c

用户态 libbinder / servicemanager：

16. Binder 概览（BBinder/BpBinder） — https://source.android.com/docs/core/architecture/ipc/binder-overview
17. 使用 Binder IPC（域、节点、RT 继承说明） — https://source.android.com/docs/core/architecture/hidl/binder-ipc
18. `ProcessState.h` / `ProcessState.cpp` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/android16-release/libs/binder/ProcessState.cpp
19. `IPCThreadState.h` / `IPCThreadState.cpp` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/IPCThreadState.cpp
20. `IBinder.h`（死亡通知与事务码） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/IBinder.h
21. `Binder.h`（`setRequestingSid`/`setInheritRt`） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/Binder.h
22. `Parcel.h` / `Parcel.cpp` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/Parcel.h
23. `Stability.h` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/Stability.h
24. `Status.h`（`EX_*` 异常码） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/Status.h
25. `IServiceManager.h`（`getService`/`checkService`/`waitForService`/`isDeclared`） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/IServiceManager.h
26. `LazyServiceRegistrar.h` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/LazyServiceRegistrar.h
27. `RpcServer.h` / `RpcSession.h` / `RpcTransport.h` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/include/binder/RpcServer.h
28. 平台内 RPC shim 头与导出清单 — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/ndk/include_platform/android/binder_rpc.h ； https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/libbinder_rpc_unstable.map.txt
29. `libbinder_rs` 导出面 — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/binder/rust/src/lib.rs
30. `servicemanager/main.cpp`（默认 `/dev/binder`） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/servicemanager/main.cpp
31. `ServiceManager.cpp`（`addService`、隔离门禁、`tryStartService`） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/servicemanager/ServiceManager.cpp
32. `Access.cpp`（`selinux_check_access`、`SELABEL_CTX_ANDROID_SERVICE`） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/servicemanager/Access.cpp
33. `vndservicemanager.rc` — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/servicemanager/vndservicemanager.rc
34. `cmds/service/service.cpp`（`service list/check/call` 用法） — https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/service/service.cpp
35. `system/core/rootdir/init.rc`（binderfs 挂载与符号链接） — https://chromium.googlesource.com/aosp/platform/system/core/+/master/rootdir/init.rc
36. `frameworks/base` Java：`Binder.java` / `IBinder.java` / `BinderProxy.java` / `ServiceManager.java` — https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/

策略与兼容性：

37. `system/sepolicy/private/access_vectors`（`binder`、`service_manager` 权限） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/access_vectors
38. `public/te_macros`（`binder_use`/`binder_call`/`add_service`） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros
39. `public/ioctl_macros`（`unpriv_binder_ioctls`） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/ioctl_macros
40. `private/domain.te`（`set_context_mgr` neverallow、Treble vndbinder、binderfs 日志 neverallow） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/domain.te
41. `public/file.te`（`binderfs_logs_*` 与 `starting_at_board_api(202504)`） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/file.te
42. `private/servicemanager.te`（`ctl_interface_start_prop`） — https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/servicemanager.te
43. Stable AIDL — https://source.android.com/docs/core/architecture/aidl/stable-aidl
44. AIDL 作为 HAL 接口与 `vts_treble_vintf_vendor_test` — https://source.android.com/docs/core/architecture/aidl/aidl-hals
45. 动态 AIDL（lazy 服务） — https://source.android.com/docs/core/architecture/aidl/dynamic-aidl
46. AIDL 语言规范（oneway） — https://source.android.com/docs/core/architecture/aidl/aidl-language
47. AIDL 后端与生成类 — https://source.android.com/docs/core/architecture/aidl/aidl-backends
48. HIDL 弃用状态 — https://source.android.com/docs/core/architecture/hidl
49. 在生产版本使用 DebugFS — https://source.android.com/docs/core/architecture/kernel/using-debugfs-12
50. 内核文件系统支持（debugfs 默认不挂载） — https://source.android.com/docs/core/architecture/android-kernel-file-system-support
51. 用 eBPF 采集内核遥测 — https://source.android.com/docs/core/architecture/kernel/kernel-telemetry
52. SELinux 概念 / 设备策略 — https://source.android.com/docs/security/features/selinux/concepts ； https://source.android.com/docs/security/features/selinux/device-policy
53. systrace/atrace 类别 — https://source.android.com/docs/core/tests/debug/systrace
54. 特权应用权限允许列表（`ro.control_privapp_permissions`） — https://source.android.com/docs/core/permissions/perms-allowlist
55. 隐私指示器（app-op 通道，非 binder 级） — https://source.android.com/docs/core/permissions/privacy-indicators
56. 数据访问审计（AppOps 归因） — https://developer.android.com/privacy-and-security/audit-data-access

开发者文档：

57. NDK Binder 参考（`AIBinder_*`、`AServiceManager_*`、`AParcel_*`、`STATUS_*`） — https://developer.android.com/ndk/reference/group/ndk-binder
58. AIDL 服务开发指南 — https://developer.android.com/develop/background-work/services/aidl
59. `android.os.Binder` 参考 — https://developer.android.com/reference/android/os/Binder

安全公告（仅字段转述）：

60. 2017-12-01 / 2019-03-01 / 2019-11-01 — https://source.android.com/docs/security/bulletin/2017-12-01 ； https://source.android.com/docs/security/bulletin/2019-03-01 ； https://source.android.com/docs/security/bulletin/2019-11-01
61. 2023-07-01 / 2024-06-01 / 2025-04-01 — https://source.android.com/docs/security/bulletin/2023-07-01 ； https://source.android.com/docs/security/bulletin/2024-06-01 ； https://source.android.com/docs/security/bulletin/2025-04-01
62. Android 安全公告概览（SPL `-01`/`-05`） — https://source.android.com/docs/security/bulletin/asb-overview

> 本会话确认**不存在或未公开文档化**、因此未采纳为事实的说法：`BC_ATTEMPT_FREEZE`、`BR_INCREASE_PRIORITY`、`BR_TRY_ACQUIRE_RESULT`；uapi 中的 `extra_size`、`sender.sid`、`struct binder_object`/`binder_big_object`；`BINDER_REQUEST_BINDER_DEVICE`；binderfs feature 的 `FEATURE_VERSION`/`ONewaySpamDetect`/`EXT_RING_BUFFER`/`SPAM_RETRY`；`AS_NORMAL`/`AS_GLOBAL`/`AS_ONEWAY` 与 `binder_alloc_free_size`；`debug.binder.*` 属性与 binder 的 `task_profiles` 条目；`Binder.getCallingSid()`（公开 API）与 `isCallingUidPrivileged`；`TransactionStats`/`getTransactionStats`/`getTimeSpentInCallback`/`getLastTransactionBinderDebugId`；`ALLOW_FD`/`DO_NOT_GAIN_RIGHTS`/`setParcelDataLimit`/`unpadded`/`ReplyHashes`/`flatten_binder`；`interface_hash`/`--stable`/`added`/`interrupted`；`RpcAddressedTransport`/`RpcTlsTransport`/`RpcServer::AllowedService`；`PingListener`；`bindAidl`/`bindUtf8`；`IServicemanagerLazyService`/`force_lazy`；选择 binder 节点的环境变量；"零拷贝/一次拷贝"作为文档化结论；binder 的 eBPF 遥测指标；CDD/强制 binderfs 的版本截止表述。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
