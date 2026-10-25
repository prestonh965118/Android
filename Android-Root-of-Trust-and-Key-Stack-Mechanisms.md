---
layout: default
title: "Android 硬件信任根与密钥栈机制分析报告"
description: "AVB 验证启动、DICE/RKP、KeyMint 与 StrongBox、Gatekeeper/Weaver、密钥证明与 FBE 的机制与审计证据"
lang: zh-CN
permalink: /Android-Root-of-Trust-and-Key-Stack-Mechanisms/
---
# Android 硬件信任根与密钥栈机制分析报告

## The Android Hardware Root of Trust and Key Stack: Verified Boot, DICE, KeyMint, Gatekeeper, Attestation and Remote Key Provisioning

- 版本：v1.0（2026-10-24）
- 适用对象：设备/固件合规审计、移动安全管理（MDM/MAS）、密钥与证明依赖的服务端风控设计、应用安全评估人员
- 视角声明：本文写的是**信任与密钥这条链怎么搭起来、每一环由谁执法、失效时外部能看到什么证据**。为避免与已有文章重复，以下内容只做交叉引用而不展开：
  - [Android 系统架构分析报告](https://prestonh965118.github.io/Android/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries/)：六段分层、进程与 IPC、SELinux/seccomp/AID 身份模型、分区与更新通道
  - [Android Linux 内核结构与子系统机制分析报告](https://prestonh965118.github.io/Android/Android-Linux-Kernel-Structure-and-Subsystems/)：内核镜像形态、dm-verity/fs-verity 在内核里的落点、debugfs 与遥测
  - [Android Framework 信任边界与越权漏洞分析报告](https://prestonh965118.github.io/Android/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis/)：Framework 的攻击面与缺陷类型学
  - [Android 内核漏洞与提权防御分析报告](https://prestonh965118.github.io/Android/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense/)：缺陷分类、公开 CVE 案例与缓解矩阵
  - [Android Binder IPC 协议与运行时机制分析报告](https://prestonh965118.github.io/Android/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms/)：BC_/BR_ 词表、身份与 SELinux 判定、binderfs、Stable AIDL 与 RPC
- 范围声明：全文仅描述**公开文档化的机制与其设计意图**，用于加固、选型与取证。不提供任何密钥提取、签名校验绕过、锁屏口令恢复或可操作的攻击流程；涉及已知失效类别时，只引用官方通告与文档给出的定性结论。

---

## 一、摘要

1. **Android 的"信任根"不是一件东西，而是四层互相咬合的结构**：安全硬件里不可变的东西（ROM 与该 SoC 的唯一秘密）→ 启动过程中的**度量**（AVB 2.0 的签名校验与状态、DICE/BCC 派生出的固件身份）→ 密钥的**执法者**（KeyMint HAL 跑在 TEE 或 StrongBox 里）→ 密钥的**保管者与命名空间**（keystore2 守护进程 + UID/SELinux 标签）。审计提问必须落到具体一层，"这台机器有没有硬件信任根"这种问法拿不到可执行结论。
2. **完整性与机密性是两套独立机制，靠一个接口咬合**：AVB 负责镜像完整性，官方文档明确 **AVB 不验证 userdata（`/data`）**；`/data` 的机密性由文件级加密（FBE）与元数据加密承担，而元数据加密的密钥"由 KeyMint 保护，KeyMint 自身又受 verified boot 保护"——这句话是整条静态数据链的闭环点。
3. **KeyMint 自 Android 12 起替代 Keymaster**。`SecurityLevel` 的四个取值（`SOFTWARE=0`、`TRUSTED_ENVIRONMENT=1`、`STRONGBOX=2`、`KEYSTORE=100`）与"每个密钥标签由**最弱的那个执法环境**记录"的归属规则，是判断"这把密钥到底靠什么"的第一手依据，不是靠机型宣传。
4. **锁屏口令的抗暴破能力只取决于一件事：Gatekeeper/Weaver 是否运行在 SE/TEE 里**。官方限速表（Android 16 QPR2 起收紧到 5 次失败 1 分钟、20 次以上永久封锁）、"失败计数器先写入再校验"的顺序、以及用**仅安全组件知晓的每启动共享 HMAC 密钥**签发的授权令牌，三者都只在安全环境内成立；富 OS 实现等于三项全无。
5. **证明链是可远程核验的，但边界由官方文档写死**：扩展 OID `1.3.6.1.4.1.11129.2.1.17`、`RootOfTrust` 四字段、版本绑定标签（`OS_VERSION`/`OS_PATCHLEVEL` 自 Android 7.1，`VENDOR_PATCHLEVEL`/`BOOT_PATCHLEVEL` 自 Android 9）、以及 Google 的吊销列表页。同时官方限定语必须原样进入结论：密钥证明**不证明设备未被 root、不证明 OS 未修改、不证明无恶意软件**。
6. **远程密钥配置（RKP）把"工厂 keybox 泄露"从平台级事件降级成单设备事件**：Android 12 起进入 AOSP，Android 14 拆成 Mainline 模块 `com.android.rkpd`，官方文档写明**以 Android 16 启动的设备仅支持 RKP**，工厂私钥不再预置、撤销可精确到单台、证书短期有效。
7. **Play Integrity 与密钥证明不是一回事**。`MEETS_DEVICE_INTEGRITY` 在 Android 13+ 依赖"硬件证明 bootloader 已锁且加载的是厂商认证镜像"，`MEETS_BASIC_INTEGRITY` 明确允许"不确定"，`MEETS_STRONG_INTEGRITY` 额外要求近一年内的补丁水位；Google 文档同时要求**不要把它当成唯一的反滥用手段**，并指出缓存 verdict 可被代理转发。
8. **分发侧有自己的完整性链**：v1（JAR）只覆盖部分内容、ZIP 中央目录不在保护范围且已弃用；v2 自 Android 7.0 覆盖整个文件；v3 自 **Android 9**（常被误记为 11）加入密钥轮换证明与支持的 SDK 版本信息；v4 自 Android 11 起把签名放到独立的 `<apk>.apk.idsig`，并用与 fs-verity 同构的 Merkle 树做安装期校验。
9. **可信执行环境正在从"必须有一块 TEE"变成"必须由 hypervisor 保护的实例"**：AVF 的官方表述是"即使宿主 Android 或其他 pVM 被完全妥协，hypervisor 仍维持所执行代码的完整性与 pVM 资产的机密性"；DICE 证书链与 CDI "只能由该特定实例派生"；Microdroid 在 `boot.img`、`super.img`、`vbmeta.img`、`vbmeta_system.img` 任一无法验证时不启动。
10. **审计上的正确问序**是：这颗 SoC 提供了什么安全硬件 → 口令由谁校验 → 密钥的层级与标签归属 → 证明链能否锚定到 Google 根或被 OEM 根（黄色状态）→ 限速与防回滚是否在硬件里生效 → 分发完整性方案与远程判定策略。属性存在性只能当作线索，不能当作结论。
11. 本文不提供任何利用步骤；所有版本号、接口名、属性名、字段名均取自本仓库可复现的公开文档与 AOSP 源码页（见第二十章），无法核实的说法已在正文显式标注为"未公开文档化"。

---

## 二、分层全景：一条链上的四层与各自的失败模式

### 2.1 全景图

```text
┌─ 层 0：安全硬件与不可变秘密 ────────────────────────────────┐
│ SoC ROM / 一次性可编程熔丝；该设备的唯一秘密（DICE 的 UDS 起点、│
│ Gatekeeper 的 hardware-backed secret key、Weaver 槽内的密钥值）  │
│ 失败模式：物理/侧信道攻击、强防护实验室级故障（对应 StrongBox） │
├─ 层 1：启动度量与固件身份 ───────────────────────────────────┤
│ AVB 2.0：bootloader 用嵌入的公钥校验 vbmeta 签名 → 描述符逐一体 │
│ 校验（hash / hashtree / chain partition / cmdline / property）；│
│ 输出 verified boot 状态与回滚索引；DICE/BCC 逐级派生固件身份链  │
│ 失败模式：解锁（橙色）、自签密钥（黄色）、镜像不可验证（红色）  │
├─ 层 2：密钥与授权的执法者 ───────────────────────────────────┤
│ KeyMint HAL（TRUSTED_ENVIRONMENT 或 STRONGBOX）：keyblob 加解密、│
│ 标签强制、AuthToken 校验、证明证书签发、回滚抗性、ID 证明        │
│ Gatekeeper / Weaver：口令校验与限速，产出 AuthToken 所需的 SID   │
│ 失败模式：SOFTWARE 层实现、AuthToken 可被富 OS 伪造、限速失效    │
├─ 层 3：保管、命名与消费 ─────────────────────────────────────┤
│ keystore2 守护进程：IKeystoreService、密钥元数据库、UID/AID 命名 │
│ 空间 + sepolicy keystore2_key 标签；上层是应用、vold、MediaDrm、│
│ Wi-Fi/EAP、RKP 客户端（com.android.rkpd Mainline 模块）          │
│ 失败模式：保管者被妥协只泄露密文与元数据；执法者被妥协才是密钥泄露│
└──────────────────────────────────────────────────────────────┘
```

### 2.2 每一环"防什么、不防什么"

| 机制 | 防的是 | 不防 | 外部可见证据 |
| --- | --- | --- | --- |
| AVB 2.0 | 镜像被替换/篡改、版本降级 | `/data` 内容（官方明确不验证 userdata） | `ro.boot.verifiedbootstate`、`ro.boot.veritymode`、`ro.boot.vbmeta.*` |
| dm-verity（由 AVB 描述符驱动） | 运行期块设备读取被改动 | 写入路径、挂载点之外的数据 | `veritymode` 取值；`RESTART_AND_INVALIDATE` 会失效当前 slot 后重启 |
| DICE / BCC | 固件层被静默替换后仍能远程识别状态 | 运行后的动态妥协 | RKP 证书请求里携带的链；pVM 的 CDI 只该实例可派生 |
| KeyMint | 密钥明文出现在富 OS 内存、标签被绕过 | 恶意应用**在授权条件下**滥用密钥 | 应用侧 `attestationSecurityLevel`、`hardwareEnforced`/`softwareEnforced` 归属 |
| keystore2 | 跨应用使用他人密钥（UID + SELinux 标签） | 拥有 root/内核妥协的本机攻击 | `/data/misc/keystore` 存在、`dumpsys -l` 中的 keystore 服务 |
| Gatekeeper/Weaver | 离线暴力猜测锁屏口令、令牌伪造 | 用户主动交出口令 | `IGatekeeper.verify` 的 `timeoutMs`、`IWeaver` 的 `STATUS_THROTTLE` |
| FBE + 元数据加密 | 设备失窃后的静态数据读取 | 已解锁运行中的设备 | 存储加密状态与 `DE/CE` 分类（见第九章） |
| Key Attestation | 服务端无法知道密钥在哪一层、设备处于什么启动状态 | OS 是否被改、是否有恶意软件（官方限定语） | 证书链 + 扩展 OID 内容 + 吊销列表 |
| Play Integrity | 常见伪造设备/非认证环境 | 全部作弊手段（官方要求不可作为唯一手段） | verdict 三档字段 |

### 2.3 名词对照（本文统一用法）

| 缩写/名称 | 全称 | 一句话 |
| --- | --- | --- |
| AVB | Android Verified Boot | 启动镜像完整性方案，核心数据结构是 VBMeta |
| vbmeta | Verified Boot Metadata | 承载签名与描述符的分区/结构 |
| DICE | Device Identifier Composition Engine | 逐层派生的固件身份；RKP 的信任根 |
| BCC | Boot Certificate Chain | DICE 派生出的证书链 |
| UDS | Unique Device Secret | 设备唯一秘密，DICE 链的起点 |
| LSKF | Lock Screen Knowledge Factor | 锁屏口令/图案，人知道的因子 |
| SID | Secure User ID | 安全环境内生成的用户身份标识，密钥绑定它 |
| AuthToken | Hardware Auth Token | 安全组件用每启动共享 HMAC 密钥签发的授权凭证 |
| RKP | Remote Key Provisioning | 不预置工厂私钥、按设备签发证明密钥的方案 |
| FBE / DE / CE | File-Based Encryption / Device / Credential Encrypted | 静态加密；DE 开机可用、CE 需解锁 |
| CDD | Compatibility Definition Document | 本文仅引用被 AOSP 文档转述的条款编号（9.11.1 / 9.11.2） |

---

## 三、层 1 之一：AVB 2.0 —— 镜像完整性怎么落地

### 3.1 信任的锚点与算法形态

- AVB 的核心数据结构是 **VBMeta struct**；描述符用于镜像哈希、镜像 hashtree 元数据与"链式分区"（chained partitions）。
- 锚点在 bootloader：**"Bootloader checks signature using embedded public key"**——签名算法形如 `SHA256_RSA2048`、`SHA256_RSA4096`，描述符里记录 `hash_algorithm`（README 列出 `sha1`/`sha256`）。
- 官方文档把"哪些密钥签什么"分成两种责任：Google 与 OEM 的分工靠**链式描述符**——"设备可被配置为创建额外的 vbmeta 分区作为链式分区，以便在不改动顶层 vbmeta 的情况下更新一部分分区"；"vendor/OEM 通过链式描述符签名自己的分区"。
- GKI/动态分区形态下常见 `vbmeta` + `vbmeta_system` + `vbmeta_vendor`（A/B 设备再带 `_a`/`_b` 后缀）。

> 审计意义：链式签名意味着**一颗顶层密钥并不等于全机镜像都由同一把密钥背书**。当合规要求写"启动链必须锚定到厂商认证密钥"时，必须逐分区确认签名者，而不是只看 `vbmeta` 一条。

### 3.2 描述符与运行期行为

| 描述符 | 作用 | 运行期后果 |
| --- | --- | --- |
| hash | 整镜像哈希（如 boot） | 载入前一次性比对 |
| hashtree | 块设备的 Merkle 树（如 system/vendor） | 转成 dm-verity 表，读块时逐块校验 |
| chain partition | 委托另一分区/另一密钥 | 把校验责任交给下层 |
| kernel cmdline | 向内核传递经校验的参数 | `androidboot.veritymode`、`androidboot.vbmeta.device_state` 等由此而来 |
| property | 键值属性（含 OS 版本信息） | `com.android.build.system.os_version`、`vendor.os_version`、`boot.os_version`；Android 13 起允许自定义 OS 版本值 |

错误模式是审计最容易忽略的一项：AVB README 明确 **`AVB_HASHTREE_ERROR_MODE_RESTART_AND_INVALIDATE` 意味着 HLOS 会失效当前 slot 并重启**，而 **`LOGGING` 模式"只应用于诊断与调试"**。生产镜像带着 logging 模式发布，等价于"完整性度量仍在做，但读取被篡改的块不再被拒绝"。

### 3.3 四种状态：绿色不是"安全"，黄色不是"被破解"

| 状态 | 文档含义 | 审计结论写法 |
| --- | --- | --- |
| green | 锁定 + 厂商验证 | 满足"设备完整性"类要求 |
| yellow | 锁定但使用自定义/自签密钥 | 信任锚是 OEM 自己而非 Google 链；跨 OEM 不可比 |
| orange | 引导加载程序解锁 | 无完整性保证；解锁会擦除用户数据（官方陈述） |
| red | 无法验证 / 无 OS | 设备实际无法启动到正常系统，取证时应视为不可信样本 |

对应属性：`ro.boot.verifiedbootstate`、`ro.boot.vbmeta.deviceState`、`ro.boot.flash.locked`、`ro.boot.veritymode`、`ro.boot.vbmeta.hash_alg`、`ro.boot.vbmeta.digest`。bootloader 通过内核 cmdline 或（Android 12+）bootconfig 传入 `androidboot.verifiedbootstate` 与 `androidboot.veritymode`。

密钥证明侧的枚举与之**不同名**：`Verified` / `SelfSigned` / `Unverified` / `Failed`，其中"Unverified 表示 bootloader 已解锁"。把两边字段对照起来写结论，比只写"是否 green"更可核验。

### 3.4 防回滚：`rollback_index[n]` 与防篡改存储

- 定义（AVB README）："回滚保护指设备除非 `rollback_index[n] >= stored_rollback_index[n]` 对所有 n 成立，否则拒绝该镜像；并随时间增大 `stored_rollback_index[n]`"；"设备把最后见到的回滚索引存放在**防篡改（tamper-evident）存储**里"。
- 动机（Verified Boot 文档）："回滚保护有助于防止某个可能的漏洞利用变成持久化，办法是确保设备只更新到更新的版本。"
- KeyMint 侧对应两件事：`Tag::ROLLBACK_RESISTANCE`（303）与实现者文档中"RPMB 提供回滚抗性"的表述；一旦密钥被销毁，"安全硬件确保它永远不再可用"。
- Android 8.0 的增强条目记录了"支持回滚保护的 Verified Boot 代码库进入 boot loaders"。

> 未公开文档化：CDD 对回滚保护的逐条强制原文、以及"设备必须使用 RPMB（而非其他防篡改存储）"的排他表述，本次未能从官方页面核实到原文，因此本文不写成硬性要求。

---

## 四、层 1 之二：DICE/BCC —— 把"我现在跑的是什么固件"变成可验证身份

- 定义：DICE "**为启动链中加载的每一段固件创建一组强、不可变的密码学身份**"，其不可绕过性只在"ROM 被妥协"时失效。
- 用途（官方 `applications-of-dice` 页）：远程验证设备**当前固件状态**，从而"**在 TEE 漏洞之后恢复信任**"——这与"把所有密钥全部撤销"是两条截然不同的响应路径。
- 与 RKP 的关系：安全配置文档写明"安全配置（RKP）的信任根是 **DICE 链**"。
- 与虚拟化的关系：AVF 文档写明给 pVM 实例的"DICE 证书链与复合设备标识（CDI）**只能由该特定实例派生**"。
- 与度量日志的区别：DICE 是**密钥派生**机制而非"可追加的度量日志"。审计时不要用"度量日志里有几条记录"这种表述去套 Android——官方文档没有这个模型（未公开文档化）。

一句话概括这一层的价值：**AVB 决定"这台设备肯不肯启动"，DICE 决定"别人能不能远程知道它现在启动的是什么"**。前者是本地安全属性，后者是可运营的安全属性。

---

## 五、层 2 之一：KeyMint —— 密钥的执法者

### 5.1 代际与接口

| 代 | 进入版本 | 文档记录的能力增量 |
| --- | --- | --- |
| Keymaster 1.0 | Android 6 | 基础密钥管理 |
| Keymaster 2.0 | Android 7 | 密钥证明、版本绑定 |
| Keymaster 3.0 | Android 8 | HIDL 化 |
| Keymaster 4.0 / 4.1 | Android 9 / 10 | StrongBox / 安全单元 |
| KeyMint v1 / v2 | Android 12 / 13 | v1 替代 Keymaster并加入 ECDH；v2 加入 Curve25519 签名与密钥协商 |

- AIDL 包名 `android.hardware.security.keymint`，接口目录含 `IKeyMintDevice`、`IKeyMintOperation`、`ISecureRandomNumberGenerator`、`SecurityLevel`、`Tag`、`KeyOrigin`、`KeyPurpose`、`HardwareAuthToken`、`KeyMintHardwareInfo`、`BeginResult`、`KeyCreationResult` 等；`IKeyMintDevice` 标注 `@VintfStability @SensitiveData`。
- `IKeyMintDevice` 方法（源码清单）：`getHardwareInfo`、`addRngEntropy`、`generateKey`、`importKey`、`importWrappedKey`、`upgradeKey`、`deleteKey`、`deleteAllKeys`、`destroyAttestationIds`、`begin`、`deviceLocked`、`earlyBootEnded`、`convertStorageKeyToEphemeral`、`getKeyCharacteristics`、`getRootOfTrustChallenge`、`getRootOfTrust`、`sendRootOfTrust`、`setAdditionalAttestationInfo`。
- 同一 `security/` HAL 目录下的兄弟接口构成一套：`authgraph`、`keymint`、`rkp`、`secretkeeper`、`secureclock`、`see`、`sharedsecret`。其中 SharedSecret 是"N 方每启动 HMAC 密钥协商"，SecureClock 是"基于挑战的时间戳协议，在没有安全时间源时可选"——**这两件正是 AuthToken 可信的时间与密钥基础**（第八章）。
- 关于"出厂设备是否还允许只提供 Keymaster HIDL"：本次未核实到明确排他表述，本文按"KeyMint 是 Android 12 起的替代方案"陈述，不写成截止线。

### 5.2 `SecurityLevel`：四个值与"最弱执法者"归属规则

| 值 | 语义（AIDL 原文表述） | 审计含义 |
| --- | --- | --- |
| `SOFTWARE = 0` | 运行在 Android 进程内；被进程妥协、root 或内核妥协击败 | 硬件后端声明不成立 |
| `TRUSTED_ENVIRONMENT = 1` | 满足 CDD 9.11.1 要求的隔离环境；**能在全 Android + 内核妥协下存活** | 主流"硬件支持"的最低达标线 |
| `STRONGBOX = 2` | 满足 CDD 9.11.2 的硬件；对物理/侧信道攻击有额外抗性、经实验室验证 | 高价值场景（支付、密钥托管）才可要求 |
| `KEYSTORE = 100` | 仅由框架强制的标签；**不会**由 `getHardwareInfo` 返回 | 标签被降级为"框架说了算"，需在报告中点出 |

归属规则是关键：**标签记录的是"执行该标签的最弱环境的安全层级"**；`SOFTWARE` 与 `KEYSTORE` 归入证明里的 `softwareEnforced` 列表，TEE/StrongBox 执行的才进 `hardwareEnforced`。这就是为什么"设备有 TEE"并不等于"这把密钥的要求有 TEE 保证"——必须逐标签看归属。

工程注记：AOSP 提供默认 Rust KeyMint 实现，注册 `IKeyMintDevice`、`IRemotelyProvisionedComponent`、`ISecureClock`、`ISharedSecret`，文档明确其"**仅为允许测试与策略合规而存在**"。量产机型若停留在默认实现，是可直接写入合规发现的差异。

### 5.3 keyblob 模型：谁拿到什么

- KeyMint 把密钥材料"**以 keyblob 形式返回，即加密且认证过的形态**"；keystore 删除该 blob 后密钥即消失。
- 应用侧的官方承诺："**密钥材料从不进入应用进程**"、密钥"不可导出"、硬件支持密钥"从不暴露到安全硬件之外"。
- **硬件包裹密钥（hardware-wrapped keys）**：Android 11 引入，目的是让**原始存储密钥从不出现在系统内存中**，即使 OS 被妥协也读不到——这是给 keystore2/MediaDrm 这类需要在富 OS 侧"拿着密钥用"的消费方的补救通道（消费方细节本文按未公开文档化处理，只陈述该页给出的目的）。

由此得到一条清晰的取证判断：**拿到设备镜像与 `/data/misc/keystore` 的密文与元数据 ≠ 拿到密钥**；差距就在执法层级上。反之，若执法层级是 `SOFTWARE`，密文与运行环境在同一台被妥协的机器上，二者差别消失。

### 5.4 可审计的密钥特征（`Tag`）

| 标签 | 编号 | 用途 |
| --- | --- | --- |
| `PURPOSE` / `ALGORITHM` | 1 / 2 | 密钥只能做被声明的事 |
| `ROLLBACK_RESISTANCE` | 303 | 依赖防回滚存储，销毁不可复活 |
| `ACTIVE_DATETIME` / `ORIGINATION_EXPIRE_DATETIME` / `USAGE_EXPIRE_DATETIME` | 400/401/402 | 时间窗（时间与信任由 SecureClock 类机制补强） |
| `USER_SECURE_ID` | 502 | 绑定 64 位 SID，密钥只在特定安全用户认证状态下可用 |
| `NO_AUTH_REQUIRED` | 503 | 明确"不需要用户认证"，是常见的策略缺陷点 |
| `USER_AUTH_TYPE` / `AUTH_TIMEOUT` | 504 / 505 | 认证类型与"认证必须多新鲜"（秒） |
| `ALLOW_WHILE_ON_BODY` | 506 | 佩戴状态类宽松条件（历史标签，评估时应保守看待） |
| `TRUSTED_CONFIRMATION_REQUIRED` | — | 未经用户确认则该密钥不可用（第八章 8.3） |
| `ROOT_OF_TRUST` | 704 | 证明里携带启动状态结构 |
| `ATTESTATION_CHALLENGE` | 708 | 服务端子挑战 |
| `OS_VERSION` / `OS_PATCHLEVEL` | 705 / 706 | 版本绑定 |
| `VENDOR_PATCHLEVEL` / `BOOT_PATCHLEVEL` | 718 / 719 | Android 9 起分层的补丁水位 |
| `ATTESTATION_ID_*`（BRAND/DEVICE/PRODUCT/SERIAL/IMEI/MEID/MANUFACTURER/MODEL/SECOND_IMEI） | 710–717, 723 | 设备身份证明 |
| `RESET_SINCE_ID_ROTATION` | 1004 | ID 轮换后是否重置过 |
| `ORIGIN`（`KeyOrigin`） | — | 区分"安全生成"与"导入"，是密钥来源的最直接证据 |

版本绑定的编码方式：**OS 版本形如 `MMmmss`，补丁水位形如 `YYYYMM`**；`OS_VERSION`/`OS_PATCHLEVEL` 自 Android 7.1、`BOOT_PATCHLEVEL`/`VENDOR_PATCHLEVEL` 自 Android 9。语义上密钥"**只向前棘轮（ratchet forward）**"，回滚会使密钥失效——这是"降级攻击对密钥无效"的机制来源。

---

## 六、层 3：keystore2 —— 保管者、命名空间与它真正防的东西

### 6.1 位置与职责

- 官方描述：**keystore2 是系统守护进程，通过 Binder API 暴露全部 Keystore 功能（`IKeystoreService.aidl`），并存储 KeyMint 的 keyblob**。
- 落盘：源码 `keystore2/src/globals.rs` 定义 `DB_PATH = "/data/misc/keystore"`。
- 命名空间：Keystore **始终**使用 UID/AID 命名空间；Keystore 2 **额外**强制 sepolicy 命名空间，标签由 `keystore2_key_contexts` 赋予（SELinux 实现文档）。
- 访问默认：密钥访问"**通常限制在创建该密钥的应用**"。

### 6.2 分工的正确理解

```text
keystore2  → 保管：谁能碰这条记录（UID/AID + SELinux 标签）、元数据、生命周期
KeyMint    → 执法：密钥能不能用、在什么条件下用（标签、AuthToken、层级）
安全硬件   → 兜底：即使上面两层都被妥协，明文仍不出现
```

审计中两个常见误判由此产生：

1. **"keystore 里有密钥记录"被当成"密钥泄露风险"** —— 记录是密文 keyblob 加元数据，风险取决于执法层级（第五章）而非记录本身。
2. **"应用能读到自己的密钥别名"被当成"跨应用可用"** —— 别名到密钥的映射在 UID 命名空间内，跨 UID 受 SELinux `keystore2_key` 标签约束。

关于备份/迁移时密钥是否可导出、以及别名到 `KeyMetadata` 的具体映射细节，本次未从官方页面核实到原文，本文不做断言（未公开文档化）。

### 6.3 设备 ID 证明（ID attestation）

- Android 8.0 / Keymaster 3 起：ID 在**工厂被复制到安全存储**，用于在证明证书里带上品牌/型号/序列号/IMEI 等；若这些 ID 未被安全配置，则**证明会失败**。
- `destroyAttestationIds()` **永久清除**这些 ID；`RESET_SINCE_ID_ROTATION` 表达 ID 轮换后的重置状态。
- 工程含义：ID 证明是**可选强度**——很多设备不带 ID，这本身不是缺陷；但服务端如果依赖序列号做绑定，就必须把"不带 ID 的合法设备"作为一等公民处理。

---

## 七、层 2 之二：Gatekeeper 与 Weaver —— 口令在硬件里的样子

### 7.1 Gatekeeper

- 定义（官方）："**Gatekeeper 在 TEE 中执行设备图案/密码认证。它使用硬件支持的密钥来注册（enroll）与校验（verify）口令**"；由三个组件构成：`gatekeeperd` 守护进程、Gatekeeper HAL 服务、TEE 内的 Gatekeeper 可信应用（TA）。
- 接口：AIDL `android.hardware.gatekeeper.IGatekeeper`，`enroll(uid, currentPasswordHandle, desiredPassword)` 与 `verify(uid, challenge, providedPassword)`；响应 `GatekeeperVerifyResponse` 含 `statusCode`、`timeoutMs`、`hardwareAuthToken`；`STATUS_REENROLL = 1`。HIDL 旧版（`gatekeeper/1.0`）在 `ERROR_RETRY_TIMEOUT` 时只给出 `response.timeout`。
- 密钥要求：硬件支持的秘密密钥"**必须永不改变，且应在每次启动时可重新导出**"。
- 计数与顺序：限速是**强制要求**；失败计数器**在校验之前写入**、成功时清除；在没有 FBE 的设备上计数写入安全存储或 RPMB。
- 参考实现两条线：Trusty 提供经认可的 Gatekeeper 实现（安全侧 TA）；AOSP 另有普通世界参考库 `platform/system/gatekeeper`，其 handle 携带 `salt, user_id, flags, HANDLE_VERSION`。

> "双重 salt 以兼容哈希方案变更"这一说法在 AOSP 与 Gatekeeper 页面均未直接文档化，本文按未公开文档化处理，不作为事实陈述（handle 中确有 `salt` 字段，用途不引申）。

### 7.2 Weaver

- 定义：Weaver（Android 8.1 引入，`IWeaver`）**为"只有在出示相应密钥后才能被读出"的秘密值提供安全持久存储**；结构是**槽数组，每槽一个键值对**，槽号范围 `[0, getConfig().slots)`。
- 方法：`getConfig()`、`read(slotId, key)`、`write(slotId, key, value)`；**没有 delete**，`write` 原子覆盖。状态：`STATUS_FAILED=1`、`STATUS_INCORRECT_KEY=2`、`STATUS_THROTTLE=3`；文档要求"**必须使用限速来限制失败读取的频率**"。
- 配置：`WeaverConfig` 含 `slots`、`keySize`、`valueSize`。**具体槽数是每设备值**，由 `getConfig()` 读出；官方文档不规定数值（"常见 8 槽"这类说法本次未核实，不写）。
- 部署约束：支持 StrongBox 的设备必须使用专用安全硬件；没有 SE 的设备可在 TEE（如 Trusty）里跑 Weaver；Android 17+ 即使没有 SE 也**强烈建议**实现 Weaver。
- 与登录的关系（Weaver 文档）：`LockSettingsService` **为一个用户管理一个槽**；`weaverKey` 是用户 LSKF 的哈希（无锁屏时用默认串），`weaverSecret` 是只能由正确 key 在 TA 限速策略下恢复的高熵随机秘密；**两者一起用于加密"合成口令（synthetic password）"**。
- 参考实现：`system/weaver`（通用安全环境）与 `external/libese`（ISO/IEC 7816-4 安全单元）；验证手段是 `VtsHalWeaverTargetTest`，且文档指出 **VTS 无法测试很长的超时**。
- 属性 `ro.hardware.weaver` 出现在文档中用于指示实现可用性；`ro.hardware.<class>` 是 libhardware 的通用 HAL 载入机制。**单一来源，脚本里只当线索。**

### 7.3 官方限速表（Android 16 QPR2+，SE/TEE 内实现）

| 连续失败次数 | 需等待 |
| --- | --- |
| 1–4 | 0 |
| 5 | 1 分钟 |
| 6 | 5 分钟 |
| 7 | 15 分钟 |
| 8 | 30 分钟 |
| 9 | 90 分钟 |
| 10 | 4 小时 |
| 11 | 12 小时 |
| 12 | 36 小时 |
| 13 | 4 天 |
| 14 | 13 天 |
| 15 | 41 天 |
| 16 | 123 天 |
| 17 | 1 年 |
| 18 | 3 年 |
| 19 | 9 年 |
| > 20 | **永久封锁** |

恢复出厂会使当前 LSKF 失效并把限速计数**归零**。这一条经常被误读为"限速可被出厂重置击败从而口令可猜"——正确的读法是：**出厂重置同时销毁了密钥**，被保护的目标（失窃设备上的用户数据）不因此变得可达。

### 7.4 SID：让"改密码"不破坏密钥绑定，也不给攻击者复活机会

- SID 在首次注册（此前无口令）时**以密码学方式生成**，在安全环境内创建。
- **可信的重新注册（trusted re-enroll）会迁移 SID**，从而让绑定的密钥继续可用；
- **不可信的重新注册会改变 SID，使绑定的密钥作废**——官方给出的理由原文是"**即使攻击者控制 Android 系统，也无法销毁敏感密钥**"。

审计结论模板：若某设备的密钥绑定策略允许"口令变更后密钥仍可用"，需确认走的是可信重注册路径；若反过来发现"口令一改密钥全废"，通常是 `USER_SECURE_ID` 绑定生效的正常表现，而不是缺陷。

---

## 八、AuthToken：跨安全组件的授权凭证

### 8.1 结构与信任来源

官方描述：授权令牌"**由仅安全组件知晓的每启动共享 HMAC 密钥签名**"，并经 Android 传递给 KeyMint。`HardwareAuthToken` 字段（源码逐字）：

| 字段 | 文档语义 |
| --- | --- |
| `challenge` | "用于让授权令牌授权特定事件" |
| `userId` | "安全"用户 ID，"在安全环境的 Gatekeeper 应用里创建"（即 SID） |
| `authenticatorId` | 来自认证应用的"安全" ID（如指纹） |
| `authenticatorType` | 认证类型 |
| `timestamp` | 自共享安全环境起点起的毫秒数，用于对照 `Tag::AUTH_TIMEOUT` |
| `mac` | 向后兼容的 MAC 算法，"由 Keymaster 3.0、Gatekeeper 1.0 与 Fingerprint 1.0 使用" |

时间从哪来、密钥从哪来，分别由 `hardware/interfaces/security/` 下的 `secureclock`（挑战式时间戳协议）与 `sharedsecret`（N 方每启动 HMAC 密钥协商）提供——**这也是为什么"没有安全时间源时 SecureClock 可选"会直接影响 `AUTH_TIMEOUT` 的可信度**：超时判断的时间基线本身需要安全组件背书。

### 8.2 应用侧的两种绑定形态

- 需要用户认证：`USER_SECURE_ID` + `USER_AUTH_TYPE` + `AUTH_TIMEOUT`（"认证后密钥可用的秒数"）。
- 不需要认证：`NO_AUTH_REQUIRED`——在敏感场景（如离线解密主密钥）里这是应当写进发现的策略缺陷。

### 8.3 确认（confirmation）与生物识别层级

- **受保护确认**：Android 9（API 28）引入，使用 `TRUSTED_CONFIRMATION_REQUIRED` 与确认令牌，依赖硬件保护的 UI + 安全处理器 + secure viewport 显示；密钥"在用户未确认时不得可用"。
- 生物识别（Framework 侧文档）：Class 3 = STRONG、Class 2 = WEAK、Class 1 = Convenience；"**原始生物数据必须用仅 TEE 知晓的设备专属密钥加密**"。应用侧 `BIOMETRIC_STRONG`/`BIOMETRIC_WEAK`/`DEVICE_CREDENTIAL`，其中 `STRONG|DEVICE_CREDENTIAL` 组合在 API 29 及以下不受支持。
- 密钥有效窗口：`setUserAuthenticationParameters(durationSeconds, authenticators)` 给出"认证后一段窗口"；**不传时长则每次操作都要重新认证**。`setInvalidatedByBiometricEnrollment(true)`（API 24 起）让新录入指纹使既有密钥失效；`setConfirmationRequired(true)` 强制显式确认。
- 指纹 HAL 代际：`android.hardware.biometrics.fingerprint@2.1`/`@2.2`、`face@1.0`（HIDL）。"BpBinder/AIDL 指纹接口"与"Biometrics 命名规范细节"本次未核实到原文，不写。

---

## 九、静态数据解锁链：LSKF → 合成口令 → FBE / 元数据加密

```text
用户 LSKF（PIN/图案/密码）
  → LockSettingsService 用 scrypt 拉伸（文档给出约 25 ms、约 2 MiB）
  → 若设备实现 Weaver：把拉伸后的 LSKF 映射到 SE/TEE 槽内的高熵随机秘密
  → 与 weaverSecret 共同加密"合成口令（synthetic password）"
  → 合成口令派生/解封 FBE 密钥（CE 每用户、DE 每设备）
  → FBE 密钥由 KeyMint 保护；元数据加密密钥同样由 KeyMint 保护
  → KeyMint 自身受 verified boot 保护（官方原文）
  → 校验侧的口令认证由 Gatekeeper（TEE）完成，产出 AuthToken 所需 SID
```

要点与结论：

1. **DE 与 CE 的差别是"何时可用"，不是"是否安全"**：DE（设备加密）在成功完成验证启动后即可用，官方 Direct Boot 文档表述为"设备加密存储包含**只有在设备成功完成一次验证启动后才可用**的密钥加密的数据"；CE（凭据加密）默认必须等用户解锁。
2. **例外要写进清单**：官方 FBE 文档把 `/data/unencrypted` 列为未加密例外；`/data/system/de_locked*` 一类 DE 内容也属于解锁前可见面（细节参见架构报告存储章）。
3. **元数据加密是必选项**：官方表述"**Android 11 及以后必须**在内部存储上启用元数据加密"，且元数据密钥"由 KeyMint（原 Keymaster）保护，其自身又受 verified boot 保护"。
4. **改密与重置的密钥后果**：LSKF 变更时"LockSettingsService 删除与旧 LSKF 绑定合成口令相关的全部信息"；恢复出厂移除密钥。
5. **为什么口令校验必须在 TEE**：FBE 文档写明 KeyMint/Keymaster **与** Gatekeeper "必须在 TEE 中实现，这样未授权的 OS 不能简单地向其索取 DE 密钥"。
6. 关于 `sid_create`/`sid_reseed`/`sid_set_boundary`、"escrow token"/"统一二级口令"、SPAKE2 与多用户解锁模型：本次均**未公开文档化**（在官方页面与源码注释中未检索到），本文不写成机制事实。
7. 关于"Android 15/16 的 `android.hardware.secure_locking` 接口""增强保护/盗窃保护/FRP 的技术底座"：本次未能在官方文档核实到对应页面或源码目录（`hardware/interfaces/secure_locking/` 返回 404），仅存在企业 API 层的 `FactoryResetProtectionPolicy` 参考页。因此本文只在"应关注但证据不足"的意义上提及，不作结论。

---

## 十、密钥证明（Key Attestation）：能证明什么、不能证明什么

### 10.1 结构

- 扩展 **OID `1.3.6.1.4.1.11129.2.1.17`**；链中第 0 项是被证明密钥自身的证书，`RootOfTrust` 是一个 ASN.1 SEQUENCE（CBOR 编码于扩展内），字段为 **`verifiedBootKey`、`deviceLocked`、`verifiedBootState`、`verifiedBootHash`**（tag `ROOT_OF_TRUST = 704`）。`verifiedBootState` 枚举为 `Verified / SelfSigned / Unverified / Failed`。
- 关于 `verifiedBootHashCornerstone`/"cornerstone" 字段：本次**未公开文档化**，不写。
- 每个标签都带 `attestationSecurityLevel`/`keymasterSecurityLevel` 归属（第五章的"最弱执法者"规则在此体现）。

### 10.2 服务端核验的正确姿势

1. 证书链必须锚定到 Google 的证明根证书列表；只信任链中**首次出现**的证明扩展（重复扩展是攻击面）。
2. 检查吊销状态：`https://android.googleapis.com/attestation/status`，文档列出的原因含 `KEY_COMPROMISE`、`CA_COMPROMISE`、`SUPERSEDED`、`SOFTWARE_FLAW`。
3. 只有 `attestationSecurityLevel` 为 `TrustedEnvironment` 或 `StrongBox` 的字段才可按硬件保证对待；`softwareEnforced` 字段在设备被妥协时可被伪造。
4. 版本绑定字段（705/706/718/719）让服务端能表达"至少 X 补丁水位"的策略，但要注意 `BOOT_PATCHLEVEL` 与 `VENDOR_PATCHLEVEL` 自 Android 9 才有。

### 10.3 官方限定语（必须原样进入结论）

Google 文档明确：证明**不证明设备已被 root 与否、不证明 OS 未被修改、不证明设备上无恶意软件**。把"拿到 attestation = 设备可信"写进风控策略，是对该机制最常见的误用。

---

## 十一、RKP：不再往工厂里灌私钥

- 现状：RKP"**自 Android 12 起就是 AOSP 的一部分**"；Android 14 把它移入 Mainline 模块 `com.android.rkpd`（此前是 `RemoteProvisioner` 应用 + Keystore 2.0）；模块仍依赖 `IRemotelyProvisionedComponent` HAL。
- 政策：**"以 Android 16 启动的设备，系统只支持 RKP。该政策逐步淘汰工厂密钥。"** 官方给出的收益：密钥不在工厂写入以"**防止密钥泄露**"；吊销可"**针对单台设备**"；RKP 证书的短有效期"**旨在允许对安全事件更快响应**"。
- 接口与流程：`IRemotelyProvisionedComponent` 的方法为 `getHardwareInfo`、`generateEcdsaP256KeyPair`、`generateCertificateRequest`、`generateCertificateRequestV2`；身份根是 DICE 链，BCC 以加密形式承载于 `ProtectedData`；`UDS_pub` 由设备唯一秘密派生。
- HAL 与版本映射（源码 CHANGELOG）：Android 12 = v1；Android 13 = v2（新增 `uniqueId`）；Android 14 = v3（移除 EEK、简化 CSR、新增 `CertificateType` 与 `supportedNumKeysInCsr`）。
- 配置阶段（README）：Phase 1「Degenerate DICE」单一自签证书；Phase 2「DICE」使用硬件信任根密钥对并遵循 Android Profile for DICE；Phase 3「SoC vendor certified DICE」收紧供应链。
- 为什么这是运营层面的变化：DICE 允许"在 TEE 漏洞之后恢复信任"，因此一次实现漏洞不再自动等价于"撤销全部出厂密钥、全机型停机"。
- 未公开文档化：本文不陈述"health stripes / 具体轮换周期 / backward distance"等未在官方页面出现的实现术语。

**审计对照表**

| 观察 | 提示的配置 | 结论写法 |
| --- | --- | --- |
| 证明链锚定 Google 根 + 全生命周期为 RKP 短期证书 | 已启用 RKP 且配置到可信阶段 | 泄露影响可精确到单设备 |
| 链中出现出厂预置的长期 keybox 证书 | 工厂密钥路径（Android 16 前形态） | 存在跨设备共享私钥的批量风险，应要求迁移计划 |
| `IRemotelyProvisionedComponent` 层级为 `SOFTWARE` 或默认 Rust 实现 | 无硬件根 | RKP 声称的设备唯一性不成立 |
| 证明扩展里 `deviceLocked=false` 或 `verifiedBootState=Unverified` | 引导解锁 | 任何"硬件证明设备完整性"的依赖都应拒绝 |

---

## 十二、分发与运行时的完整性判定：APK 签名与 Play Integrity

### 12.1 APK 签名四代（官方表格）

| 方案 | 起点 | 覆盖范围 | 备注 |
| --- | --- | --- | --- |
| v1（JAR） | 一直存在 | **仅部分内容**，ZIP 元数据不验证 | 已弃用；只靠 v1 的包应视为弱完整性 |
| v2 | Android 7.0 | 整个文件 | 篡改即签名失效 |
| v3 | **Android 9** | 与 v2 同级覆盖 + **签名密钥轮换证明**、支持的 SDK 版本信息 | 常被误记为 Android 11 |
| v4 | Android 11+ | 独立文件 `<apk>.apk.idsig`；与 fs-verity 同构的 Merkle 树用于 `adb install` 期间校验 | 减少安装期整包重哈希成本 |

失败行为：内容被修改会"**使 APK 签名失效**"，安装被拒；签名证书不一致会被当作**新包**处理从而拒绝更新。

### 12.2 Play Integrity 三档（官方定义）

| Verdict | 文档语义 | 使用建议 |
| --- | --- | --- |
| `MEETS_DEVICE_INTEGRITY` | 真实、已认证的 Android 设备；Android 13+ 有**硬件证明 bootloader 已锁且加载厂商认证 OS 镜像** | 可作为"设备真实性"门槛 |
| `MEETS_BASIC_INTEGRITY` | bootloader 可能锁可能不锁、启动状态可能验证可能未验证、设备可能未认证，"**Google 无法提供保证**"；Android 13+ 仍要求 Google 提供的信任根 | 只作宽松信号，不可当作硬件保证 |
| `MEETS_STRONG_INTEGRITY` | 真实且认证的设备，并有**近期安全更新**；Android 13+ 要求 OS 与厂商补丁在一年内 | 高价值动作的附加门槛 |

官方同时写明两点边界：Integrity API"**抽象掉了信号的复杂性**"，以及**不要把它当作唯一的反滥用机制**；对 verdict 做缓存会被代理转发。服务端应当把 Integrity verdict 与密钥证明（第十章）**交叉核对**：前者是 Google 侧的综合裁决，后者是可独立验链的硬件陈述，两者不一致时按更保守的一侧处理。

### 12.3 企业侧与 CDD 关联

"Verified Access（为企业提供的设备证书以表达 OS 状态）"与"CDD 中把验证启动状态绑定到证明的条款原文"，本次**未在 source.android.com / CDD 检索到对应页面**，因此本文只列为待核实项，不作机制陈述。

---

## 十三、执行环境的三种形态：Trusty、StrongBox、AVF/pKVM

### 13.1 Trusty

"**Trusty 是一个为 Android 提供 TEE 的安全操作系统（OS）**"；官方文档列出其在其中实现：**KeyMint、Gatekeeper、Weaver、配置（provisioning）、确认 UI、存储**。审计语境里"TEE 实现来自 Trusty 还是厂商私有安全 OS"通常不影响结论，但影响补丁归属与响应路径（安全公告里 Trusty 相关问题有独立通道）。

### 13.2 StrongBox

- 所需环境（硬件最佳实践文档）：**独立 CPU、安全存储、高质量真随机数、防篡改封装、侧信道抗性**（完整要求属 CDD 9.11.2，自 Android 9 起）。
- 算法/尺寸：RSA-2048、AES-128/256、ECDSA+ECDH P-256、HMAC-SHA256（8–64 B）、Triple-DES；文档直白说明它"**更慢、资源更受限、支持的并发操作更少**"。
- 应用侧探测：`FEATURE_STRONGBOX_KEYSTORE` 或 `setIsStrongBoxBacked(true)`。具体槽位/配额数字本次未核实，不写。
- 没有 StrongBox 不是缺陷：它是**可选**层级。合规要求里若出现"必须 StrongBox"，应先确认该要求是否真的对应业务风险（大量场景 TEE 已足够），否则会把实现推向"降低密钥可用性"的副作用。

### 13.3 AVF / pKVM：把"可信执行"从专属硬件改成 hypervisor 保护的实例

- 保障表述："**hypervisor 维持所执行代码的完整性与 pVM 资产的机密性，即使宿主 Android 或其他 pVM 被妥协**"。
- 身份表述：给 pVM 实例的"DICE 证书链与 CDI **只能由该特定实例派生**"。
- 启动约束："**Microdroid 在 `boot.img`、`super.img`、`vbmeta.img` 或 `vbmeta_system.img` 无法验证时不会启动**"；文档同时说明解锁后设备所有者可"重刷通常受 verified boot 保护的分区"。
- AVF 在 Android 13 的增强条目中出现。Android 15/16 关于"无 TEE/厂商自管 TEE"的硬件要求表述本次未核实，不写。

审计含义：当供应商声称"我们的可信执行跑在 pVM 里"，需要三件证据——hypervisor 本身如何被验证、pVM 的 DICE 身份如何被远程核验、宿主被妥协时的降级行为是什么。这三件都能在上述官方表述里找到判定基线。

---

## 十四、失效模式矩阵：坏在哪一环，外部能看到什么

| 环节 | 典型失效 | 外部可见证据 | 影响定级（建议） |
| --- | --- | --- | --- |
| 硬件信任根 | 默认 Rust KeyMint 进入量产 | `getHardwareInfo` 层级为 `SOFTWARE`；证明字段大量落入 `softwareEnforced` | HIGH（"硬件支持密钥"宣称全部不成立） |
| 启动度量 | 生产镜像使用 `LOGGING` verity 模式 | `ro.boot.veritymode=logging` | HIGH（篡改块不再被拒绝） |
| 启动度量 | bootloader 解锁后继续用于企业场景 | `ro.boot.verifiedbootstate=orange`、`ro.boot.vbmeta.deviceState`、证明里 `deviceLocked=false` | HIGH |
| 启动度量 | 自签密钥链 | 状态 yellow；证明 `verifiedBootState=SelfSigned` | MEDIUM（信任锚为 OEM，不等同被破解） |
| 防回滚 | 允许刷回旧镜像 | 旧 `OS_PATCHLEVEL`/`BOOT_PATCHLEVEL` 的证明仍可签发；无法用防篡改存储推进索引 | HIGH |
| 口令层 | Gatekeeper 为富 OS 实现 | 无法从属性可靠判定；应用侧 `attestationSecurityLevel` 与令牌来源异常 | HIGH（限速与令牌不可伪造性同时失去） |
| 口令层 | 未实现 Weaver | 无槽可读，LSKF 拉伸结果直接依赖 KeyMint/Gatekeeper 路径 | MEDIUM（Android 17+ 建议线之前不判违规） |
| 授权层 | 密钥用 `NO_AUTH_REQUIRED` | 密钥特征（`getKeyCharacteristics`/证明内容） | MEDIUM→HIGH 视场景 |
| 授权层 | `AUTH_TIMEOUT` 过长 | 密钥特征 + 业务风险 | LOW→MEDIUM |
| 保管层 | keystore 元数据可被同 UID 复用 | `/data/misc/keystore`、SELinux `keystore2_key` 标签 | LOW（密文与元数据，不含明文） |
| 加密链 | 元数据加密未启用（Android 11+ 设备） | 分区映射与挂载形态（见架构报告） | HIGH（违反官方"必须"表述） |
| 证明链 | 仍使用出厂 keybox、证书长期有效 | 链结构与有效期、无 RKP 请求痕迹 | MEDIUM（Android 16 起为 HIGH） |
| 证明依赖 | 服务端只看 attestation 存在 | 服务端策略本身 | MEDIUM（忽略官方限定语） |
| 分发 | 仅 v1 签名 | 包签名块 | MEDIUM |
| 运行时判定 | 把 BASIC 档当硬件保证 | 策略文档/代码 | MEDIUM |
| 虚拟化 | 声称 pVM 但无可验证身份 | 无 DICE/CDI 远程核验路径 | MEDIUM |

---

## 十五、检测与取证采集

### 15.1 采集脚本（`root-of-trust-probe.sh`）

只读采集，不修改设备状态。输出交给判定脚本或人工阅读。

```bash
#!/usr/bin/env bash
# 采集 Android 设备侧"信任根与密钥栈"证据（只读）
set -u
dev="${1:-}"
[ -n "$dev" ] || { echo "用法: $0 <adb-serial>"; exit 2; }
adb -s "$dev" get-state >/dev/null 2>&1 || { echo "设备不可达: $dev"; exit 2; }

sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

echo "== 设备与版本 =="
echo "model=$(p ro.product.model)"
echo "release=$(p ro.build.version.release) sdk=$(p ro.build.version.sdk)"
echo "first_api=$(p ro.product.first_api_level) spl=$(p ro.build.version.security_patch)"
echo "build_type=$(p ro.build.type) debuggable=$(p ro.debuggable)"

echo "== 启动度量（层 1）=="
for pr in ro.boot.verifiedbootstate ro.boot.vbmeta.deviceState ro.boot.flash.locked \
          ro.boot.veritymode ro.boot.vbmeta.hash_alg ro.boot.vbmeta.digest; do
  printf '%-34s %s\n' "$pr" "$(p "$pr")"
done

echo "== 口令执法者（层 2）=="
# 说明：这些属性名是 OEM 相关的加载线索，不是能力的权威声明；
# 未命中不等于缺失，命中也不等于工作在安全环境。权威判定见本文 15.3 节。
for pr in ro.hardware.gatekeeper ro.hardware.weaver; do
  printf '%-24s %s\n' "$pr" "$(p "$pr")"
done

echo "== HAL 可见性 =="
echo "keystore_svc_lines=$(sh 'dumpsys -l' | tr ' ' '\n' | grep -ci keystore)"
# lshal 面向 HIDL；AIDL HAL 需通过 dumpsys/servicemanager 侧观察
echo "hidl_hint=$(sh 'lshal 2>/dev/null | head -400' | tr ' ' '\n' \
  | grep -Eic 'gatekeeper|weaver|keymaster|keymint')"

echo "== 密钥存储与解锁状态 =="
echo "keystore_dir=$(sh 'ls /data/misc/keystore 2>/dev/null | wc -l')  # 需 root；无权限时为 0"
echo "unencrypted_marker=$(sh 'ls /data/unencrypted 2>/dev/null | wc -l')  # 需 root"
echo "metadata_enc=$(sh 'grep -c dm-default-key /proc/misc 2>/dev/null')  # 需 root，仅作线索"

echo "== 内核侧验证模式 =="
echo "cmdline_veritymode=$(sh 'cat /proc/cmdline' | tr ' ' '\n' | grep -i veritymode)"
echo "bootconfig=$(sh 'cat /proc/bootconfig 2>/dev/null' | tr '\n' ' ' | cut -c1-160)"
```

### 15.2 判定脚本（`root-of-trust-assert.sh`）

只做**有官方依据**的判定；对无法从设备侧权威确定的项输出 INFO 并指向正确判定路径。

```bash
#!/usr/bin/env bash
# 依据采集结果给出信任根/密钥栈发现（只读，不写设备）
set -u
dev="${1:-}"
[ -n "$dev" ] || { echo "用法: $0 <adb-serial>"; exit 2; }
sh() { adb -s "$dev" shell "$1" 2>/dev/null | tr -d '\r\n'; }
p()  { sh "getprop $1"; }

vb=$(p ro.boot.verifiedbootstate)
vm=$(p ro.boot.veritymode)
ds=$(p ro.boot.vbmeta.deviceState)
fl=$(p ro.boot.flash.locked)
digest=$(p ro.boot.vbmeta.digest)
rel=$(p ro.build.version.release)
spl=$(p ro.build.version.security_patch)

case "$vb" in
  green)  echo "OK    verifiedbootstate=green（锁定且厂商验证）" ;;
  yellow) echo "MED   verifiedbootstate=yellow：使用自定义/自签密钥，信任锚为 OEM" ;;
  orange) echo "HIGH  verifiedbootstate=orange：引导加载程序已解锁，无完整性保证" ;;
  red)    echo "HIGH  verifiedbootstate=red：镜像不可验证，样本不可信" ;;
  *)      echo "INFO  未取到 verifiedbootstate（旧设备或 OEM 未透传），改以 attestation 的 verifiedBootState 判定" ;;
esac

case "$vm" in
  logging) echo "HIGH  veritymode=logging：hashtree 错误模式用于诊断，篡改块不再被拒绝" ;;
  enforcing|"") : ;;
  *) echo "INFO  veritymode=${vm}：非 enforcing，按机型文档核对预期" ;;
esac

# 锁定声明与状态一致性
if [ -n "$ds" ] && [ -n "$vb" ]; then
  case "$ds$vb" in
    lockedgreen|unlockedorange) : ;;
    lockedorange|unlockedgreen|lockedyellow) echo "HIGH  deviceState=${ds} 与 verifiedbootstate=${vb} 不自洽" ;;
    *) echo "INFO  deviceState=${ds} / verifiedbootstate=${vb}：字段组合需人工核对" ;;
  esac
fi
[ "$fl" = "1" ] && [ "$vb" = "orange" ] && echo "HIGH  flash.locked=1 但状态为 orange：上报与实际不符"

[ -n "$digest" ] && echo "OK    vbmeta.digest 已上报（可与镜像侧重算值比对）" \
  || echo "INFO  vbmeta.digest 为空：不判定，常见于 OEM 未透传"

if [ -n "$spl" ]; then
  age=$(( ( $(date +%s) - $(date -d "$spl" +%s) ) / 86400 ))
  if   [ "$age" -gt 180 ]; then echo "HIGH  SPL 落后 ${age} 天：证明里携带的补丁水位同样陈旧"
  elif [ "$age" -gt 90 ];  then echo "MED   SPL 落后 ${age} 天：STRONG 档判定（要求一年内）可能失败"
  else echo "OK    SPL 落后 ${age} 天"; fi
fi

# 口令执法者：不依据属性存在性下结论
gk=$(p ro.hardware.gatekeeper); wv=$(p ro.hardware.weaver)
if [ -z "$gk" ]; then
  echo "INFO  未从属性读到 gatekeeper 实现名：请在应用侧读 KeyMint getHardwareInfo 层级，" \
       "并以密钥证明的 attestationSecurityLevel 判定 Gatekeeper/AuthToken 是否可信"
else
  echo "INFO  gatekeeper=${gk}（属性为加载线索，不代表运行于 SE/TEE）"
fi
if [ -z "$wv" ]; then
  echo "INFO  未从属性读到 weaver 实现名：Weaver 缺失时 LSKF 抗暴破完全依赖 Gatekeeper 层级"
else
  echo "INFO  weaver=${wv}：需以 IWeaver.getConfig() 的 slots/keySize/valueSize 判定实际强度"
fi

# 依赖密钥证明的服务端交叉核对提示
echo "INFO  权威判定路径：验链至 Google 证明根 → 核对 RootOfTrust(verifiedBootKey/deviceLocked/"
echo "      verifiedBootState/verifiedBootHash) → 核对 705/706/718/719 版本绑定 → 查 CRL"

case "$rel" in
  8) echo "INFO  Android ${rel}：无 BOOT/VENDOR_PATCHLEVEL（Android 9 起才有），无 RKP（Android 12 起）：" \
     "证明只能给出单一的 OS 补丁水位";;
  9|10|11) echo "INFO  Android ${rel}：RKP 尚未进入 AOSP（Android 12 起）：证明密钥大概率仍是工厂预置形态，" \
     "泄露影响按存量机型批量计";;
esac
```

### 15.3 什么才是"权威判定"

设备侧 adb 只能看到**上报**，看不到**执法**。要判断密钥到底在哪一层，可用路径只有三条，按可信度排序：

1. **验密钥证明链**：读 `attestationSecurityLevel` / `keymasterSecurityLevel`，看关键标签是进 `hardwareEnforced` 还是 `softwareEnforced`。这是可远程核验、不可由本机伪造（前提是链锚定正确且未在 CRL 上）的唯一路径。
2. **应用侧读密钥特征**：`getKeyCharacteristics` 与 Android Keystore 的 `KeyInfo`；适合自检与回归。
3. **行为验证**：在受控条件下验证"锁定/解锁、改密、恢复出厂、降级尝试"对密钥可用性的影响（这是策略验证，不是绕过测试；具体步骤属项目内部规程，本文不给出）。

本章脚本负责快速圈定可疑设备与留证据，不用于出具"该设备有硬件信任根"这类结论性判定。

---

## 十六、常见误区

1. **"设备有 TEE 所以密钥有硬件保护"**：标签由**最弱执法环境**记录；`SOFTWARE`/`KEYSTORE` 归入 `softwareEnforced`，必须逐标签看归属。
2. **"keystore2 里有密钥文件 = 密钥泄露"**：里面是加密且认证过的 keyblob 加元数据；风险取决于执法层级与口令层。
3. **"attestation 通过 = 设备没被 root / OS 没被改"**：官方明确不证明这三件事；它证明的是密钥层级与启动状态声明。
4. **"Play Integrity 的 BASIC 档等于设备真实"**：文档写明 bootloader 可能锁可能不锁、启动状态可能未验证、"Google 无法提供保证"。
5. **"黄色（yellow）状态等于被破解"**：黄色是**锁定 + 自定义密钥**，是 OEM 自签形态；它改变信任锚归属，不等同完整性失效。
6. **"AVB 保护了 `/data`"**：官方写明 AVB 不验证 userdata；`/data` 靠 FBE + 元数据加密，两者接口在"元数据密钥由 KeyMint 保护、KeyMint 受 verified boot 保护"。
7. **"恢复出厂能重置限速，所以口令可被暴力猜"**：重置同时销毁密钥；被保护目标并未因此可达。
8. **"APK v3 签名从 Android 11 开始"**：官方表格是 **Android 9**；Android 11 开始的是 v4。
9. **"签名方案够用就行，v1 无所谓"**：v1 不覆盖 ZIP 元数据且已弃用；只靠 v1 应视为弱完整性。
10. **"必须有 StrongBox 才算安全"**：StrongBox 是可选层级，代价是更慢、并发更少、算法集更窄；把它列为普适要求通常带来可用性损伤而非安全增益。
11. **"Weaver 有固定槽数（比如 8）"**：槽数由 `IWeaver.getConfig().slots` 逐设备返回，文档不规定数值。
12. **"属性 `ro.hardware.gatekeeper` 为空说明没有 Gatekeeper"**：属性名是 OEM 相关的加载线索；权威判定走证明链（15.3）。
13. **"RKP 只是换了个证书签发流程，安全等价"**：差别在**吊销粒度（单设备）**与**工厂不再持有私钥**，以及可用 DICE 在 TEE 漏洞后恢复信任。
14. **"降级只影响用户体验"**：回滚索引在防篡改存储里推进，密钥"只向前棘轮"，旧补丁水位的证明与密钥可用性本身就是防御目标。
15. **"虚拟化 pVM 只是把东西关进虚拟机"**：官方主张是宿主或其他 pVM 被妥协时仍维持 pVM 代码完整性与资产机密性；这是需要独立核验的能力宣称。

---

## 十七、趋势判断（2024 → 2026）

1. **工厂密钥退场已进入执行期**：Android 16 起仅 RKP 的表述把"keybox 泄露"从批量事件改写成个体事件。2026 年的合规检查点应从"有没有 keybox"转为"这台设备是否在 RKP 阶段 2/3 的配置路径上"。
2. **限速表成为可审计常量**：Android 16 QPR2+ 把 Gatekeeper/Weaver 的退避表写成公开文档（含 >20 次永久封锁）。这意味着"实现是否符合官方表"第一次成为可对照的验收标准，也意味着老设备的宽松退避会被更容易地标红。
3. **补丁水位的分层继续加深**：`BOOT_PATCHLEVEL` / `VENDOR_PATCHLEVEL`（Android 9 起）+ AVB property 描述符里的 `os_version` 三兄弟 + Android 13 起允许自定义 OS 版本值，共同把"一个 SPL 数字"拆成"引导、系统、厂商三条可分别陈述的水位"。服务端策略应相应拆维。
4. **可信执行从专属硬件走向 hypervisor 实例**：AVF/pKVM + DICE/CDI + Microdroid 的验证启动前置，提供了"没有专用安全 OS 也能有可验证隔离"的第二条路径。近期（本次检索）关于 Android 15/16 硬件 TEE 要求松动的说法**未官方文档化**，应作为观察项而非结论。
5. **"完整性 + 机密性 + 可远程陈述"三件套正在合流**：AVB 决定肯不肯启动，DICE/RKP 决定别人能不能知道启动的是什么，attestation/Integrity 决定业务能不能据此做门控。审计上，只测其中一件的报告在 2026 年已不完整。
6. **判定难度上升的方向是可验证性而非保密性**：Google 把字段名、OID、CRL、状态枚举、限速表都写成公开文档。安全评估的护城河从"知道内部细节"转移到"能否正确解读公开结构并交叉核对"。

---

## 十八、结论

Android 的信任模型可以压缩成三句话：

1. **完整性靠度量，机密性靠密钥，可运营性靠可远程验证的身份**——AVB 与 dm-verity 做第一件，FBE/元数据加密/KeyMint 做第二件，DICE/BCC + RKP + attestation 做第三件。三者接口处（"元数据密钥由 KeyMint 保护，KeyMint 受 verified boot 保护"）就是整机的安全收口点。
2. **"有没有硬件信任根"这个问题问错了，正确的问法有三个**：密钥的标签由哪一层执法（`SecurityLevel` 与 `softwareEnforced` 归属）、口令由谁校验并有谁来限速（Gatekeeper/Weaver 是否在 SE/TEE）、设备状态能否被远程核验且泄露影响是否可精确到单台（attestation 链与 RKP 阶段）。
3. **判定必须走可核验链，不走设备侧上报**：`verifiedBootState`、`deviceLocked`、`OS/VENDOR/BOOT_PATCHLEVEL`、`attestationSecurityLevel`、CRL 状态是同一批可交叉核对的事实；属性、宣传语和"机型档次"不是。

对审计交付的最低要求：每台被评估设备都应能回答"它在哪一环上是 `SOFTWARE`"。回答不了，报告里所有"硬件支持"的表述都是转述，而不是结论。

---

## 十九、常见问题（FAQ）

**Q1：同一台设备上，锁屏口令很强但数据仍不安全，可能吗？**
可能。若 Gatekeeper/Weaver 不在 SE/TEE 里，限速与令牌不可伪造性都不成立，口令强度提供的只是富 OS 侧的拉伸（官方给出的 scrypt 参数约 25 ms / 2 MiB，量级上远不足以独立抵抗离线猜测）。

**Q2：为什么我的密钥要求（如"必须用户认证"）出现在 `softwareEnforced` 里？**
因为该标签由**最弱执法环境**记录。常见原因是密钥规格要求了实现不支持的认证类型/确认能力，或整条 KeyMint 实现层级为 `SOFTWARE`。

**Q3：设备已解锁（橙色）但 attestation 仍能签出，是不是伪造？**
不一定。`deviceLocked=false` + `verifiedBootState=Unverified` 是**如实上报**；问题是你的服务端策略是否接受它。拒绝它才算机制生效。

**Q4：Android 12 以上的设备一定用 KeyMint 吗？**
KeyMint 是 Android 12 起对 Keymaster 的替代方案。本次未核实到"出厂设备禁止继续只提供 Keymaster HIDL"的排他表述，因此不要把"没有 KeyMint"直接当作不合规判定，应先确认机型代际。

**Q5：恢复出厂能重置限速，是否算缺陷？**
不算。恢复出厂会使当前 LSKF 失效并移除密钥，被保护的静态数据不因此可达（官方对"解锁擦除用户数据"与"恢复出厂移除密钥"均有陈述）。

**Q6：没有 StrongBox 需要整改吗？**
StrongBox 是可选层级（CDD 9.11.2 形态）。除非业务真的要求抗物理/侧信道，否则普适要求会带来更慢、并发更少、算法集更窄的实际代价。

**Q7：Play Integrity 够用吗，还要自己验证明链？**
官方要求不要把它当唯一反滥用手段，且缓存 verdict 可被代理转发。需要"哪个密钥/哪次签发"级别的绑定时，密钥证明链不可替代。

**Q8：元数据加密没开会怎样？**
Android 11+ 的官方表述是"必须启用"。缺失时暴露的是文件系统结构与块级模式信息，且违反文档化的必选项，可直接写为发现。

**Q9：RKP 之后还需要关注 keybox 吗？**
需要，但关注点变了：Android 16 前形态的设备仍可能携带工厂密钥，批量泄露风险按存量设备计；对这些设备应看 CRL 命中与迁移计划。

**Q10：本文为什么完全不给提权或绕过内容？**
定位是机制与审计证据。缺陷类型、CVE 案例与提权取证在姊妹篇里（见开头视角声明），本文若重复会让两份报告都变差。

---

## 二十、参考资料（公开来源，本次逐条访问核实）

Android 官方安全文档（source.android.com）：

1. Keystore / KeyMint 概览与代际 — https://source.android.com/docs/security/features/keystore
2. KeyMint 实现者参考 — https://source.android.com/docs/security/features/keystore/implementer-ref
3. Keystore 特性与安全层级 — https://source.android.com/docs/security/features/keystore/features
4. 版本绑定 — https://source.android.com/docs/security/features/keystore/version-binding
5. 密钥证明结构与 RootOfTrust — https://source.android.com/docs/security/features/keystore/attestation
6. 硬件包裹密钥 — https://source.android.com/docs/security/features/encryption/hw-wrapped-keys
7. 元数据加密 — https://source.android.com/docs/security/features/encryption/metadata
8. 文件级加密（FBE）与合成口令 — https://source.android.com/docs/security/features/encryption/file-based
9. SELinux 实现与 `keystore2_key_contexts` — https://source.android.com/docs/security/features/selinux/implement
10. Gatekeeper — https://source.android.com/docs/security/features/authentication/gatekeeper
11. Weaver — https://source.android.com/docs/security/features/authentication/weaver
12. 认证限速表 — https://source.android.com/docs/security/features/authentication/rate-limiting
13. 受保护确认 — https://source.android.com/docs/security/features/protected-confirmation
14. 生物识别 — https://source.android.com/docs/security/features/biometric
15. Verified Boot 总览与属性 — https://source.android.com/docs/security/features/verifiedboot
16. AVB 与链式分区 — https://source.android.com/docs/security/features/verifiedboot/avb
17. 验证启动流程与状态 — https://source.android.com/docs/security/features/verifiedboot/boot-flow
18. 设备状态（锁定/解锁） — https://source.android.com/docs/security/features/verifiedboot/device-state
19. DICE — https://source.android.com/docs/security/features/dice
20. DICE 的应用（TEE 漏洞后恢复信任） — https://source.android.com/docs/security/features/dice/applications-of-dice
21. Trusty — https://source.android.com/docs/security/features/trusty
22. APK 签名方案 — https://source.android.com/docs/security/features/apksigning
23. APK Signature Scheme v3 — https://source.android.com/docs/security/features/apksigning/v3
24. APK Signature Scheme v4 — https://source.android.com/docs/security/features/apksigning/v4
25. 硬件安全最佳实践（StrongBox 要求） — https://source.android.com/docs/security/best-practices/hardware
26. 远程密钥配置（Mainline 模块 `com.android.rkpd`） — https://source.android.com/docs/core/ota/modular-system/remote-key-provisioning
27. Android Virtualization Framework — https://source.android.com/docs/core/virtualization
28. AVF 安全模型（pKVM、CDI、Microdroid） — https://source.android.com/docs/core/virtualization/security
29. AIDL HAL 与 HIDL 弃用 — https://source.android.com/docs/core/architecture/aidl/aidl-hals
30. 引导加载程序锁定/解锁 — https://source.android.com/docs/core/architecture/bootloader/locking_unlocking
31. 通过 AVB 传递版本信息 — https://source.android.com/docs/core/architecture/bootloader/version-info-avb
32. 动态分区与 `vbmeta_*` — https://source.android.com/docs/core/ota/dynamic_partitions/implement
33. 各版本安全增强条目 — https://source.android.com/docs/security/enhancements

开发者文档（developer.android.com）：

34. Android Keystore 系统 — https://developer.android.com/privacy-and-security/keystore
35. Android 密钥证明与吊销列表 — https://developer.android.com/privacy-and-security/security-key-attestation
36. Direct Boot 与 DE 存储 — https://developer.android.com/privacy-and-security/direct-boot
37. Play Integrity verdicts — https://developer.android.com/google/play/integrity/verdicts
38. Play Integrity 概览 — https://developer.android.com/google/play/integrity/overview
39. 生物识别认证与密钥有效窗口 — https://developer.android.com/identity/sign-in/biometric-auth
40. 应用签名与 Play App Signing — https://developer.android.com/studio/publish/app-signing
41. `FactoryResetProtectionPolicy`（企业 API） — https://developer.android.com/reference/android/app/admin/FactoryResetProtectionPolicy.Builder

AOSP 源码（android.googlesource.com）：

42. `hardware/interfaces/security/` 目录（keymint/rkp/secureclock/sharedsecret/authgraph/see/secretkeeper） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/
43. `IKeyMintDevice.aidl` 方法清单 — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/IKeyMintDevice.aidl
44. `SecurityLevel.aidl`（层级定义与归属规则） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/SecurityLevel.aidl
45. `Tag.aidl`（标签编号表） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/Tag.aidl
46. `HardwareAuthToken.aidl`（字段逐字） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/HardwareAuthToken.aidl
47. KeyMint AIDL 接口目录与默认 Rust 实现 — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/README.md ； https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/default/main.rs
48. RKP HAL README（DICE 阶段、BCC、UDS_pub） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/rkp/README.md
49. `IRemotelyProvisionedComponent.aidl` 与 CHANGELOG（v1/v2/v3 映射） — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/rkp/aidl/android/hardware/security/keymint/IRemotelyProvisionedComponent.aidl ； https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/rkp/CHANGELOG.md
50. `IGatekeeper.aidl` / `GatekeeperVerifyResponse.aidl` / 旧 HIDL — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/gatekeeper/aidl/android/hardware/gatekeeper/IGatekeeper.aidl
51. `IWeaver.aidl` 与 `WeaverConfig.aidl` — https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/weaver/aidl/android/hardware/weaver/IWeaver.aidl
52. Gatekeeper 普通世界参考库 — https://android.googlesource.com/platform/system/gatekeeper/+/refs/heads/main/gatekeeper.cpp
53. keystore2 `DB_PATH` — https://android.googlesource.com/platform/system/security/+/refs/heads/main/keystore2/src/globals.rs
54. AVB README（VBMeta、描述符、错误模式、回滚索引、签名算法） — https://android.googlesource.com/platform/external/avb/+/refs/heads/master/README.md
55. `lshal`（HIDL 调试工具说明） — https://android.googlesource.com/platform/frameworks/native/+/master/cmds/lshal/Lshal.cpp

其他：

56. Google 工厂/开发者镜像与解锁数据擦除 — https://developers.google.com/android/images

> 检索过程中确认**未公开文档化**、因此本文未采纳为事实的说法：`android.hardware.secure_locking` 接口；`sid_create`/`sid_reseed`/`sid_set_boundary`；SPAKE2 二级口令解锁模型；Weaver 具体槽数；StrongBox 具体密钥配额；`verifiedBootHashCornerstone`；RKP 的"health stripes"/"backward distance"；CDD 把验证启动状态绑定到证明的条款原文；Verified Access 的 AOSP 文档页；Keymaster HIDL 的排他截止线。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
