# Android 硬件与厂商安全 · 搜索收录关键词方案

站点：`https://android-security-reports-0e8t10askpz.qoder.website/`
生成日期：2026-10-25 · 主题簇：硬件信任根 / 安全组件 / SoC 与基带 / OEM 更新治理
性质：SEO 运营方案，不放入 `Publish-Android/`（该目录内任何 `.md` 都会被 `tools/build-site.mjs` 渲染成站点页面）。
上位文档：`seo-keyword-plan-android.md`（同目录）——分级规则、`meta keywords` 结论、GSC/IndexNow 流程、验收框架均在那份里，本份只补硬件与厂商这一簇，不重复。

---

## 1. 与现有 7 篇的重叠声明（先划清边界）

这一簇的词**不是无主之地**，已有归属：

| 现有页面 | 已占用的硬件/厂商词 |
| --- | --- |
| 信任根报告 ch13《执行环境的三种形态：Trusty、StrongBox、AVF/pKVM》 | Trusty、StrongBox、AVF/pKVM |
| 信任根报告 ch3–ch5、ch10–ch11 | AVB 2.0、KeyMint、密钥证明、RKP、DICE/BCC |
| 内核报告 ch3《内核攻击面盘点》、ch7《加固机制清单》 | 内核攻击面、厂商驱动的暴露路径、缓解机制 |
| 方法论报告 域 4/5/7/8/9 | HAL 与 vendor 进程、内核通用子系统与厂商驱动、连接域与基带、安全域与信任根、分发更新与调试通道 |
| 架构报告 ch7《HAL 与 Treble 的工程细节》、ch12《更新架构》、ch3《启动链与可信根》 | Treble、HAL、VNDK、更新通道、启动链 |

本方案的定位：**把这些词从"机制描述"推进到"跨厂商实现差异 + 治理与采购可核验指标"**，这是现有正文没有覆盖的角度，也是硬件/厂商簇真正的检索缺口。

---

## 2. 覆盖度实测（grep 8 个文件）

| 关键词 | 出现于 | 判定 |
| --- | --- | --- |
| 硬件 / 厂商 / OEM | 6 篇 / 6 篇 / 7 篇 | ✅ 泛词已饱和，不必再争 |
| TrustZone / TEE | 2 篇 / 6 篇 | ✅ 有承接 |
| Trusty | 4 篇 | ✅ |
| 基带 | 3 篇 | ✅ |
| 固件 / 驱动 / bootloader | 各 6 篇 | ✅ |
| 验证启动 | 4 篇 | ✅ |
| Treble / VNDK / GSI | 6 / 2 / 2 篇 | ✅ |
| APEX | 4 篇 | ✅ |
| FRP | 1 篇（信任根） | ⚠️ 单点 |
| 高通 | 2 篇 | ⚠️ 单点，无厂商级专章 |
| Exynos | 1 篇（内核） | ⚠️ 单点 |
| 联发科 | 1 篇（方法论） | ⚠️ 单点 |
| Pixel | 3 篇（多为"测试设备"语境） | ⚠️ 非产品安全语境 |
| **三星 / Knox / Titan / eSE / 安全元件专章 / SPU / OP-TEE / 猎户座 / 骁龙 / fastboot / OEM 锁 / VAB** | **0 篇** | ❌ 零覆盖 |

结论：泛词（硬件、厂商、OEM、固件、驱动）已有内容承接，改标题即可争夺；**产品与厂商专名基本为零**，这类词必须靠新增页面承接，否则就是上一份方案 §3 的同一问题——标题承诺、正文不兑现。

---

## 3. 词表

### T1 簇核心（争首页或新母页）
Android 硬件安全 · Android 厂商安全差异 · 安卓安全芯片 · 硬件信任根 · TEE 可信执行环境 · 基带安全 · OEM 安全更新 · Android 固件安全

### T2 主题词（每条应有一个 H2 承接）
- **安全组件形态**：安全处理器、隔离安全内存、独立安全芯片 vs SoC 内安全区、硬件级密钥存储、防侧信道、抗物理读取
- **厂商实现**：Knox Vault（安全处理器 + 隔离安全内存）、Titan M2 安全芯片、Tensor 安全核、高通移动安全方案与 SPU、联发科安全启动链、厂商 TEE 实现（Trusty / OP-TEE 路线）
- **启动与身份**：Bootloader 锁定状态、OEM 解锁开关、验证启动密钥吊销、设备绑定与恢复保护（FRP）、密钥吊销与熔断（eFuse/RMA）
- **SoC 攻击面**：厂商内核驱动、DSP/HPA 子系统、基带处理器、调制解调器协议栈、多媒体与显示驱动、闭源 blob 与符号缺失
- **更新治理**：SoC 厂商安全公告 → OEM → 设备的补丁传导链、安全补丁级别（SPL）与厂商公告的落差、机型更新承诺（代际/年限）、区域与运营商通道延迟、EOL 设备存量
- **审计与选型**：设备安全评分口径、可核验的更新历史、企业采购视角的 Android 设备安全对比

### T3 长尾（公开来源里的原词，几乎无竞争）
`Knox Vault` · `Titan M2` · `Secure Processing Unit` · `OP-TEE` · `Trusty TEE` · `AVF / pKVM` · `fastrpc` · `DSP Services / HLOS`（高通公告的技术域划分口径） · `CVE-2020-11277` · `CVE-2023-24033` · `CVE-2023-26496/26497/26498`（Exynos 调制解调器） · `BaseMirror` · `BaseComp` · `Internet-to-baseband RCE` · `Doom of Device Drivers` · `Qualcomm security bulletin` · `Samsung security updates scope` · `7 generations OS upgrades` · `Mainline APEX` · `virtual A/B / snapshot merge` · `eFuse` · `anti-rollback` · `bootloader lock state` · `device integrity`

> 已剔除两个我原本打算收录、但**未能在一手来源里核实到拼写**的研究代号（某高通漏洞集群的营销命名、某基带自动发现工具名）。按 §3 的规则，未经证实的专名不进词表——它们大概率根本不存在。

> 型号词（Galaxy S24、Pixel 9、Tensor G4 这类）**不进 `<title>`，不做 H2**，只在正文与 FAQ 出现：一是消费意图流量与开发/审计受众不匹配、拉高跳出；二是型号级事实需要持续维护，过期内容会拖累整站可信度。

---

## 4. 落地路径

**P0 · 零内容成本（只改标题、描述与小节名）**

1. 信任根报告 ch13 的 H2 补一个已覆盖的厂商维度说法，例如"执行环境的三种形态与厂商实现差异：Trusty、StrongBox、AVF/pKVM"。**不要**在这个 H2 里加 `OP-TEE`——正文目前零覆盖，加进去就是标题承诺、正文不兑现；要先在正文点名并给来源，改标题才成立。
2. 方法论报告 域 5 与 域 7 的 H2 已经含"厂商驱动""基带"，无需改动；这两处真正缺的是**具体专名出现在正文首段**——在域 5 首段点名 SoC 厂商闭源模块与高通内核缺陷公开案例，在域 7 首段点名 Exynos 调制解调器与联发科相关公开研究，各一次、附来源。
3. 架构报告 ch12《更新架构》的 H2 与实际检索词之间差的正是 `安全补丁级别` / `SPL` 这一说法，正文已有 ASB 与 SPL 内容，属零成本补词。

**P1 · 需新增内容（这两页才能让 T2 的厂商簇成立）**

- **页面 A《Android 厂商安全组件形态与实现差异》**
  承接：安全芯片/安全处理器/隔离内存/TEE 路线、Knox Vault、Titan M2、Tensor 安全核、高通 SPU、Trusty vs OP-TEE；**必须包含"能核验什么、不能核验什么"**（外部可观测信号、密钥证明字段、bootloader 锁状态），不写成机型评测。
- **页面 B《Android 补丁传导链与厂商更新治理分析》**
  承接：SoC 厂商公告 → OEM → 设备的链条、SPL 与公告口径差、更新承诺代际/年限、区域与运营商延迟、EOL 存量风险；给出一套**只读、可脚本化的设备侧更新核验方法**，与系列既有"可核验指标"风格一致。

两页都要遵守系列规范：开头视角声明（防御性、不含利用代码/偏移量/操作步骤）、逐条可访问核实的公开参考、页脚咨询行。

**文案（供新建页直接用）**
```
页面 A
title: "Android 厂商安全组件形态与实现差异：安全芯片、TEE 与密钥存储 | Android 安全报告集"
description: "对比 Android 设备上硬件信任根的几种落地形态——独立安全处理器与隔离安全内存、SoC 内安全区、Trusty 与 OP-TEE 路线、StrongBox 密钥执法，并给出可外部核验的信号与不可核验的边界。防御性分析，不含利用代码。"

页面 B
title: "Android 补丁传导链与厂商更新治理：从 SoC 公告到设备 SPL | Android 安全报告集"
description: "拆解 SoC 厂商安全公告、OEM 适配与设备分发之间的时间差，说明安全补丁级别 SPL 与厂商公告口径的差距、更新承诺的代际与年限差异，以及可只读核验的设备侧更新证据。面向 Android 设备安全评估。"
```

**P2 · 内网/内链**：新页在首页表格、方法论报告 域 5/7/9 与信任根报告 ch13 各加一条锚文本内链；首页 JSON-LD 的 `about` 补上 T2 厂商词。

---

## 5. 红线（这一簇比软件簇更敏感）

以下词**不进标题、不做 H2、不做内链锚文本、不承诺内容**：

`FRP 绕过` / `激活锁解除` · `bootloader 解锁教程` / `BL 解锁不影响保修` · `Knox 破解` / `Knox 熔断恢复` · `一键 root 工具` · `基带降级` / `IMEI 修复` / `写 IMEI` · `绕过 Play Integrity` / `设备完整性伪造` · `运营商解锁` / `卡贴` · `绕过安全启动`

原因有两层：一是与整站"防御性分析、不含利用代码"的定位直接冲突（页脚与每篇视角声明都是这个承诺）；二是这些意图多数指向**规避防盗、篡改设备标识、绕过运营商合约**，属于会造成实际第三方损害的行为，不是分析性话题。

可以正面写的相邻话题（有意收这批合法意图的检索）：设备二手交易前如何确认解锁历史与锁状态、企业如何核验发放设备的更新状态、用户如何查自己机型还在不在支持范围内、`bootloader 已解锁` 这一状态为何会削弱密钥证明的可信度。**这些都属于取证与核验，不给操作步骤去改变状态。**

---

## 6. 参考来源（本轮逐条命中，落地成页面前仍需按系列规范逐条访问核实）

- [Understanding Samsung Knox Vault — Samsung Knox 官方博客](https://www.samsungknox.com/en/blog/understanding-samsung-knox-vault-protecting-the-data-that-matters-most)（安全处理器 + 隔离安全内存的厂商定义）
- [Samsung Security Updates Scope — Samsung Mobile Security](https://security.samsungmobile.com/workScope.smsb)（更新支持范围与自 2024 年起的延长政策）
- [Samsung Sets the New Standard With Four Generations of OS Upgrades — Samsung Newsroom 2022-02-10](https://news.samsung.com/global/samsung-sets-the-new-standard-with-four-generations-of-os-upgrades-to-ensure-the-most-up-to-date-and-more-secure-galaxy-experience)（4 代 OS + 最多 5 年安全更新的旧口径，用于对照代际政策变化）
- [Qualcomm February 2026 Security Bulletin](https://docs.qualcomm.com/securitybulletin/february-2026-bulletin.html)（"补丁正与 OEM 共享"的供应链传导一手表述）
- [Qualcomm Product Security](https://www.qualcomm.com/company/product-security) · [Qualcomm Mobile Security Solutions](https://www.qualcomm.com/products/features/mobile-security-solutions)
- [Multiple Kernel Vulnerabilities Affecting All Qualcomm Devices — Zimperium zLabs](https://zimperium.com/blog/multiple-kernel-vulnerabilities-affecting-all-qualcomm-devices)
- [Multiple Internet-to-baseband Remote Code Execution Vulnerabilities in Exynos Modems — Project Zero 2023-03](https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html)（本轮复核：文中给出 CVE-2023-24033、CVE-2023-26496、CVE-2023-26497、CVE-2023-26498；**未**出现我原先想收录的那个自动发现工具代号，故已从词表剔除）
- [CVE Record: CVE-2020-11277](https://www.cve.org/CVERecord?id=CVE-2020-11277)（高通 fastrpc 竞态，用作"SoC 闭源模块 = 独立攻击面"的一手锚点）
- [Security Bulletins | Qualcomm Documentation](https://docs.qualcomm.com/product/publicresources/securitybulletin)（公告按 HLOS / DSP Services 等技术域划分，是"T2 更新治理"词表的实际口径来源）
- [BaseMirror: Automatic Reverse Engineering of Baseband Interfaces — arXiv:2409.00475](https://arxiv.org/html/2409.00475) · [BaseComp — USENIX Security '23](https://www.usenix.org/system/files/usenixsecurity23-kim-eunsoo.pdf)
- [Hardening cellular basebands in Android — Google Security Blog 2023-12-12](https://security.googleblog.com/2023/12/hardening-cellular-basebands-in-android.html)
- [An Investigation of the Android Kernel Patch Ecosystem — USENIX Security '21](https://www.usenix.org/system/files/sec21summer_zhang.pdf) · [A Device-Centric Analysis of Android Security Updates — NDSS 2024](https://www.ndss-symposium.org/wp-content/uploads/2024-175-paper.pdf) · [Unveiling the Global Landscape of Android Security Updates — IEEE TQ 2026](https://ieeexplore.ieee.org/document/11370963/)
- [Security: Titan M2 Hardware Root Of Trust Explained](https://kashifmukhtar.com/pixel-10-titan-m2-security-hardware-root-trust/)（二手来源；成稿前需替换为 Google/Pixel 官方一手页，暂不作为正文唯一依据）
