---
layout: default
title: "Android 应用安全弱点分类分析报告"
description: "移动应用弱点分类的五套公开口径（MASVS 要求、MASTG 测试、MASWE 弱点枚举、Mobile Top 10 风险榜单、CWE 缺陷编号）各自回答什么问题、怎样互相换算；MASVS v1 的 V1–V8 与 v2 的八个域、CWE-919 移动视图的二十一个成员、清单层可见与不可见的弱点信号、目标版本引起的默认值漂移、四级判定模型与十项运营指标，附两个只读核验脚本（APK 清单弱点清点与两次清点的漂移比对）"
lang: zh-CN
permalink: /Android-App-Security-Weakness-Taxonomy-Defense-Analysis/
---
# Android 应用安全弱点分类分析报告

**Android Mobile Application Security Weakness Taxonomy: Mapping MASVS, MASTG, MASWE, the Mobile Top 10 and CWE onto Signals You Can Actually Observe in Your Own APK**

- **报告日期**：2026-10-27
- **报告性质**：公开技术文档与防御性安全分析
- **适用对象**：移动应用与 SDK 的安全负责人与架构师、应用安全评估与渗透测试团队、发布门禁与 CI 工程团队、安全基线与移动设备管理（MDM）团队、第三方应用与供应链风控、应用商店审核与安全治理人员
- 视角声明：本文写的是**"一个移动应用的弱点被哪几套公开口径描述、这些口径之间怎样换算、以及其中哪一部分能被你在自己应用的构建产物上静态观察出来"**。**平台侧的强制隔离机制、Framework 权限判定的越权案例、内核缺陷与提权取证、代码投递与执行链不在本文**，那些内容在姊妹篇里，本文只做交叉引用而不重复展开：
  - [Android 应用沙箱与进程隔离防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)（平台怎么把应用围住；本文讲应用自己声明了什么形态，两者是"外部约束"与"自身姿态"的两侧）
  - [Android Framework 信任边界与越权漏洞分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)（权限与身份判定缺陷；本文的清单信号是这类缺陷**进入设备之前的声明形态**）
  - [Android 攻击面与漏洞研究方法论分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)（九域可达面清点与 R0–R4 分级；本文给出的是"这些面上可能站着哪一类弱点"的词表）
  - [Android 远程代码执行分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)（代码怎么进来；本文讲进来之前应用已经声明了哪些**弱点类别**）
  - [Android 系统架构分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-System-Architecture-Layers-Processes-and-Trust-Boundaries.html)（分层与信任边界；本文的分类只落在应用层这一格）
  - [Android Binder IPC 协议与运行时机制分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Binder-IPC-Protocol-and-Runtime-Mechanisms.html)（跨进程调用的机制与身份传递；本文只看清单里那些"谁会接到跨进程调用"的声明）
  - [Android 平台代码组织与构建体系分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)（构建变体与签名身份从哪来；本文的"构建身份类弱点"依赖这一侧的口径）
  - [Android 补丁传导链与厂商更新治理分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)（修没修到；本文讲**这一类弱点有没有被纳入检查**，两者是不同的治理问题）
  - [Android 无线协议栈与基带缺陷防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Wireless-Stack-and-Baseband-Defense-Analysis.html)（该篇按应用自己在清单里声明的弱点形态归类，清点对象是 APK；本篇按系统无线栈的层与责任人归类，清点对象是设备姿态。两者共用"证据从哪来、读不到就记成缺口"这条判定纪律，但换算口径互不适用）
  - [Android 微架构侧信道与密钥抽取防御分析报告](https://android-security-reports-0e8t10askpz.qoder.website/Android-Microarchitectural-Side-Channel-and-Key-Extraction-Defense-Analysis.html)（本文写应用自己声明的弱点形态与五套公开口径的换算；该篇的对象是芯片、内核与运行时，两篇共享"证据从哪来"这条线，但清单层读不到该篇的任何结论，反之该篇不判定应用包内的弱点）
- 范围声明：全文只讨论**公开文档化**的分类体系、公开披露的弱点类别与平台公开行为，以及在这些公开规则之上可安全实施的**只读**清点与检测流程。**不提供任何可用的漏洞利用代码、内存布局偏移、原语构造、策略绕过步骤或可操作的攻击流程。**所有"核验"指读者对**自有或已获授权测试的应用包与设备**执行采集。OWASP 对测试文档本身也给出了同样的边界提醒："Remember, attacking a system without written authorization is illegal in many parts of the world!"（参考资料 5）。本文的两个脚本都不接触设备：一个把你自有 APK 的清单展开文本转成分级清单，一个比对两份这样的清单；两者都不访问网络、不写被检查对象。

---

## 一、摘要：分类不是选一个标准，是让一条断言在五套口径之间可以换算

移动应用的"弱点分类"看起来是一个选型问题——用 OWASP MASVS、用 Mobile Top 10、用 CWE，还是用商店自己的审核条款。但真实工作里遇到的从来不是"该用哪个"，而是**四句来自不同口径的话如何对齐**：产品说"这个包过了 MASVS L1"，测试说"MSTG-PLATFORM-2 有一条未通过"，公告说"修复了一个 CWE-926"，风控说"Top 10 里的 M8 又出现了"。四句话各自成立，合起来却说不清**哪个组件、哪条属性、哪次变更**，也说不清下一次发布该拦谁。

本文的立场是：**分类体系的价值不在覆盖一张清单，而在把弱点陈述变成可复核的断言**。一条合格的断言必须有五个成分——判定条件、证据来源、判定结果、严重性档位、以及**证据缺失时的记法**。前五套公开口径提供的正好是互补的四样东西：MASVS 提供要求编号与适用档位（参考资料 1、2），MASTG 提供"证据从哪来"的测试形态（参考资料 5），MASWE 提供移动侧弱点的命名词表并自称是"要求与测试之间的桥"（参考资料 3、4），Mobile Top 10 提供组织内沟通用的优先级语言（参考资料 6），CWE 提供跨工具、跨公告的公共编号（参考资料 7–11）。缺的是第五样：**证据缺失怎么记**。这部分只能由评估者自己定纪律，本文第十四章给出四级判定模型与四条纪律，核心一条是"未采到 = 缺口，不是通过"。

分类还要处理一个物理约束：**同一个分类条目，在不同层上可观察性差别极大**。以清单（`AndroidManifest.xml`）为例，"组件是否导出、导出时有没有权限门槛、备份通道是否打开、明文流量是否被显式允许、可调试位是否被带上发行包"这一批弱点**在静态文本里就有确定答案**；而"密钥是否存在可导出形态、日志里写了什么、服务端是否真的校验了、深链参数被接到之后怎么用"在同一份文本里**没有任何证据**。把它们放在同一张检查表里打勾，是应用安全治理里最常见的失效模式：表上每一项都有归属，实际每一项的证据来源不同、置信度不同、可自动化程度不同。本文把这条边界画出来（第十一、十二章），并只把**清单层可见**的那部分做成可执行脚本（第十八、十九章）。

还有一个只有 Android 才有的复杂性：**同一份声明文本，会因为目标 SDK 版本不同而含义不同**。`android:exported` 缺省是否隐式导出、明文流量默认允许还是禁止、`setAllowFileAccess()` 的默认值、`PendingIntent` 是否必须显式声明可变性——这些判定都跟着包的 `targetSdkVersion` 走。OWASP 的 V1–V8 要求目录、CWE 的通用缺陷定义都不会替你处理这一层，平台文档会，但它散落在版本行为变更页与各元素参考里。本文第十七章把这条漂移单独列出，因为它直接决定了"同一条属性在两个包里是不是同一类弱点"。

从可执行验证的结果看，一条弱点在分类上被说清楚之后，它的运营形态就很清楚了：本文两个脚本在五个测试输入上产出的信号码共五十六个（脚本 1 四十一、脚本 2 十五），分四个等级，同一条版本线的一次真实漂移被记成"新增 4 个导出组件、新增 4 个 L3/L2 信号码、1 个缺口两次未闭合"（第十九章样例），而一次输入退化被记成"当前副本没有组件记录，本次列出的消失应读作采集失败"（第十九章样例）。这两行输出是本文对"分类落地"的全部要求：**能被差分、能被追责、能在证据不足时拒绝给结论。**

---

## 二、五套口径各自回答什么问题

先做一次正名。五套口径经常被当成"同一件事的不同版本"，实际它们问的是五个不同的问题，判定权也不在同一处。

| 口径 | 它回答的问题 | 条目形态 | 判定权在谁 | 本文怎么用它 |
| --- | --- | --- | --- | --- |
| OWASP MASVS | 这个应用**满足哪些安全要求** | 要求编号（如 `MASVS-V2.2`）+ 适用档位（L1 / L2 / R） | 评估者、采购方、合规 | 作为**域骨架**：本文所有弱点先归到要求域 |
| OWASP MASTG | 这些要求**怎么被测出来** | 测试文档，前言字段同时带 `masvs_v1_id` 与 `masvs_v2_id` | 测试执行者 | 作为**证据形态**参照：每条判定要有可复现的输入 |
| OWASP MASWE | 背后是**哪一类弱点** | 移动侧弱点条目（自称是 MASVS 与 MASTG 之间的桥） | 测试者、开发者 | 作为**命名词表**：整改单上写弱点名而不是现象 |
| OWASP Mobile Top 10 | 先管**哪十类风险** | 榜单条目 `M1`–`M10` | 组织、风险委员会 | 作为**优先级语言**：向管理层沟通时使用，不作为验收基线 |
| CWE / CWE-919 移动视图 | 这是**哪一类软件缺陷** | `CWE-NNN` 编号 + 通用定义句 | 工具、公告、标准 | 作为**交叉索引**：与公告、扫描器、内部缺陷库对齐 |

三处容易被混用的差别值得单独说明。

**要求与弱点不是同一层。** MASVS 说的是"应当满足什么"，MASWE 说的是"没满足时属于哪一类问题"。OWASP 对后者的自我描述是："The OWASP Mobile Application Security Weakness Enumeration (MASWE) is a list of common security and privacy weaknesses in mobile applications. It acts as the bridge between the MASVS and the MASTG."（参考资料 3、4）。一句"要求未满足"不能直接派工，一句"这里是一个未验证 Intent 来源的广播接收器处理路径"才能派工——这就是词表的价值。

**榜单不是清单。** Mobile Top 10 的口径是风险排序，不是覆盖保证；OWASP 自己在文档里就限定了一句："It should be noted that the risks to mobile applications do not stop at the Top 10"（参考资料 6）。用榜单当验收基线，等价于宣布"榜外弱点不需要处理"。

**通用缺陷编号不替你做平台判定。** CWE 的移动视图只是把"常在移动应用中出现"的条目聚合，视图自述是"CWE entries in this view (slice) are often seen in mobile applications."（参考资料 7）。它的成员里只有一部分（如 CWE-925、926、927）的定义句直接以 Android 应用为主语，其余（如 CWE-312、940）是通用软件措辞，把它们当成"Android 专属缺陷"会造成误读——本文在第十六章的逐条对齐里对这类成员明确标注了"通用定义"。

为什么本文以 MASVS 族为骨架而不是另两选：Top 10 是榜单，不能当覆盖基线；CWE 是通用缺陷词表，对"备份通道""导出组件"这类平台特有形态没有专门条目，只有聚合视图；MASVS 提供**带档位的、可直接引用编号的要求目录**，能原样写进合同与门禁语句，而它的配套 MASTG 又把每条要求接到一个可执行的测试形态上。CWE 在本轮里退居交叉索引——本文的信号码不直接引用 CWE 编号，而是在第十三章给出两者的对齐关系，这样脚本输出可以被扫描器与公告检索接住，又不假装自己是 CWE 的实现。

---

## 三、MASVS 的档位语义：两档加一类韧性要求，编号指向一条可判定的控制

MASVS 的公开定义句给出了它的目的，也给出了它的限度："The objective of the MASVS is to establish a level of confidence in the security of mobile apps by providing a set of controls that address the most common mobile application security issues."（参考资料 1）。"level of confidence"这个词值得注意：标准把自己定义为**置信度的生产者**，不是**安全性的担保**。

档位是 MASVS 最常被读错的地方。它只有两档，加上一类独立的韧性要求："The MASVS defines two security verification levels (MASVS-L1 and MASVS-L2), as well as a set of reverse engineering resiliency requirements (MASVS-R)."（参考资料 1），并且这两档的区别是**适用范围**而不是严重程度："MASVS-L1 contains generic security requirements that are recommended for all mobile apps, while MASVS-L2 should be applied to apps handling highly sensitive data."（参考资料 1）。

| 档位 | 它断言的东西 | 常见误读 | 正确的门禁写法 |
| --- | --- | --- | --- |
| MASVS-L1 | 所有应用都建议满足的通用控制 | "L1 = 低危，可以先不做" | 发布前必须满足；缺项需按条目登记风险承担人 |
| MASVS-L2 | 处理高敏感数据的应用应满足的加强控制 | "L2 = 严重漏洞" | 数据分级触发：一旦命中敏感数据类别，L2 变为强制项 |
| MASVS-R | 逆向工程韧性（软件保护类控制） | "上了加固就等价于满足安全要求" | 只能作为**附加层**，不能替代任何 L1/L2 控制 |

第三行不是本文的立场，是标准自己的限定：OWASP 明确写过，MASVS-R 里列出的软件保护控制"can ultimately be bypassed and must never be used as a replacement for security controls."（参考资料 1）。这一句对国内常见的"先问加没加固"的沟通方式很有针对性：加固属于韧性，属于"攻击者要多花多少时间"，与"这个组件是否被无条件导出"是两类完全不同的问题，不能互相折抵。

要求编号的形态决定了它能不能被当成可复核断言。MASVS 给出的样式是：`Requirement: MASVS-Vx.y, e.g. MASVS-V2.2: "No sensitive data is written to application logs."`（参考资料 1）。示例这条的措辞值得抄进内部规范：**它描述的是一个可观察的事实（敏感数据有没有落到应用日志里），而不是一个意图（我们有日志管理策略）**。凡是写成"有策略/有流程/已评审"的条目，都不天然具备可复核性；凡是能写成"某处不存在某物/某处存在某物"的条目，才可以进入门禁。

最后一层限定来自决策归属："implementing a control or not should ultimately be a risk-based decision, which is taken/communicated with the business owners."（参考资料 1）。技术评估输出的是**未满足项与证据**，把"不满足"转成"接受"需要业务负责人签字。本文第十四章的判定纪律把这句话变成脚本口径：凡未采到证据的条目一律记为 `L0-LOW` 缺口而不是通过，正是为了让"接受"这个动作有明确的对象与数量。

---

## 四、从 v1 的 V1–V8 到 v2 的八个域：迁移时该靠什么对齐

MASVS v1.x 的分区是按 `V1`–`V8` 编号的，公开目录给出的八个区标题是（参考资料 1、2）：

| 编号 | 公开标题（原文） | 覆盖范围（本文译述） |
| --- | --- | --- |
| V1 | Architecture, Design and Threat Modeling Requirements | 架构与设计：威胁建模、分层、数据流与信任边界 |
| V2 | Data Storage and Privacy Requirements | 本地数据存储与隐私：静态数据、日志、备份、标识符 |
| V3 | Cryptography Requirements | 密码学使用：算法、密钥管理、随机性、协议选择 |
| V4 | Authentication and Session Management Requirements | 认证与会话管理：凭据、会话生命周期、离线认证 |
| V5 | Network Communication Requirements | 网络通信：传输保护、证书校验、固定与例外 |
| V6 | Platform Interaction Requirements | 与平台环境的交互：组件间通信、WebView、系统能力、输入 |
| V7 | Code Quality and Build Setting Requirements | 代码质量与构建设置：依赖管理、错误处理、构建标志 |
| V8 | Resilience Requirements | 韧性：抵抗逆向与篡改的软件保护控制 |

v2 系列把编号方式换成了域名标识：`MASVS-STORAGE`、`MASVS-CRYPTO`、`MASVS-AUTH`、`MASVS-NETWORK`、`MASVS-PLATFORM`、`MASVS-CODE`、`MASVS-RESILIENCE`、`MASVS-PRIVACY`（参考资料 2、3、4）。OWASP 仓库的发布记录把 v2.1.0 的发布时间标为 2024-01-18（参考资料 2）。从这份标识集合能读出两件事：**数据与隐私被拆开了**（v1 的 V2 同时管存储和隐私，v2 把 `PRIVACY` 单列成一个域），以及**没有一个域专门对应"认证之外的输入输出校验"**——这部分内容分散在 `PLATFORM` 与 `CODE` 里。

需要说明的是：**本文不逐条复述 v2 每个域的官方一句话定义**。八个域标识可以从公开文档确证，八个域各自的标准描述句未全部复核到位，本文把它们列入文末的"未在公开文档中形成可直接引用口径的清单"（第二十三章末）。这不是回避，而是一个方法示范：**在分类迁移期，可靠的对齐手段不是猜标题，而是读条目自带的双编号。**

MASTG 的测试文档正好提供了这个手段。它的 YAML 前言里同时保留两套标识，实测样式是：`masvs_v1_id:` 下挂 `MSTG-STORAGE-1`、`MSTG-STORAGE-2`，`masvs_v2_id:` 下挂 `MASVS-STORAGE-1`（参考资料 5）。也就是说，**"v1 的哪几条测试映射到 v2 的哪一条要求"这件事被维护者逐条写在测试文档头部**，而不是写在某张总表里。任何团队在做口径迁移时，正确做法是逐条解析测试前言的双编号字段生成映射表，而不是按标题相似度手工归类——按标题归类会在 `PRIVACY` 与 `STORAGE` 之间产生系统性重复计数。

这条方法对本系列的直接意义是：**信号码要挂到域，不能挂到具体条目编号。** 第十八章脚本输出的信号码（如 `provider_open`、`backup_unfiltered`、`cleartext_allowed`）在第十三章的对齐表里挂到 v2 域名与 CWE 成员，而不写"违反 MASVS-STORAGE-2"这类断言——因为条目编号本身还在迁移，而"未采到证据的条目"与"证据表明不满足的条目"在整改成本上完全不同。

---

## 五、MASTG：把"要求"变成一次可复现的测试

OWASP 对自身的定位是一句手册式的描述："The OWASP Mobile Application Security Testing Guide (MASTG) is a comprehensive manual for mobile app security testing and reverse engineering."（参考资料 5）。对分类工作来说，MASTG 的贡献不是"教你怎么做"，而是它把**每一次判定拆成了四个可记录成分**：一个测试标识、一组前置条件与输入、一个执行过程、一个期望与实际的证据形态。这一拆分是"可复核"的技术定义——如果一条判定不能被拆成这四项，它在治理上就不可交接。

本文从 MASTG 借三个观念，不借它的流程。

**借"标识可回溯"。** 每个测试有稳定标识（`MSTG-*` / `MASVS-*`），因此报告里引用标识而不是描述，才能跨版本追人。第十八章的信号码承担同样职责：`provider_open` 是一个可引用标识，而不是"某个 provider 有问题"这样的句子。

**借"输入决定结论"。** 静态分析、动态执行、服务端配合得到的结论置信度不同，MASTG 对每个测试写清需要哪一种。本文第十四章的纪律把这一点写成"证据层"字段：同一个弱点，来自清单文本与来自动态验证，在整改单上应该走不同的验收路径。

**借"授权边界"。** 前引的那句"没有书面授权的攻击行为在多国违法"（参考资料 5）不是一句免责套话，而是可执行范围：对第三方应用包做静态清点与对第三方应用做动态调试，是两件在法律与流程上完全不同等级的事。本系列的脚本一律限定为**自有或已获授权测试的产物**的只读处理。

不借的部分同样要说清：MASTG 里包含大量针对具体工具链与运行时环境的操作步骤，本文不复制这些内容；本文只在两个地方给出可直接执行的命令——**对你自己的 APK 用 `aapt2` 展开清单文本**，以及**把文本交给两个只读脚本**。这两步都不接触设备、不修改被检查对象、不依赖动态调试。

---

## 六、MASWE：弱点枚举补上的是"命名"，不是"判定"

OWASP 对 MASWE 的自述有两句关键信息（参考资料 3、4、27）：

> "The OWASP Mobile Application Security Weakness Enumeration (MASWE) is a list of common security and privacy weaknesses in mobile applications. It acts as the bridge between the MASVS and the MASTG."
>
> "The MASWE is intended to be a complementary list to the CWE, focusing specifically on security weaknesses in mobile applications."

第一句说明它的结构位置：MASVS 说"要什么"，MASTG 说"怎么测"，中间缺一个"测出来的问题叫什么"。没有词表时，同一个现象会在不同报告里被写成"备份没关""allowBackup 是 true""数据可被备份导出""M8 配置错误"，四种写法对应不了同一条整改责任。第二句说明它和 CWE 的关系是**互补而非替代**：CWE 提供跨语言的通用缺陷编号，MASWE 提供移动场景下的具体形态。

对治理流程，词表补上的三件事很具体：

**派工单位。** "未验证 Intent 来源的广播接收器处理路径"可以派给一个具体的接收器实现者；"配置类问题"不能。本文信号码里 `receiver_open_custom` 与 `receiver_system_action` 分成两个码位，就是在做这一层区分——同样是导出且无发送方权限的接收器，只监听自定义动作和监听系统标准动作，整改动作并不相同。

**证据归属。** 弱点条目天然带"证据从哪一栏来"的期望：存储类弱点需要文件与日志证据，平台交互类需要组件与调用路径证据，通信类需要链路证据。第十八章脚本每条 `ROW` 只声明清单文本里的证据，绝不越层给结论，这条纪律的词表来源就是这一区分。

**与 CWE 的双向可换算。** 声明"complementary to CWE"意味着一条 MASWE 条目应该能找到若干 CWE 成员作为上位概念。第十六章的表 B 给出的是"从 CWE 侧进入"的逐条对齐；对 MASWE 侧，本文只使用它已公开的定位与自述，**不复述其条目清单与编号格式**——该列表的逐条口径与编号方式未在本系列的复核范围内（见第二十三章末清单）。这一条克制是有意的：一条被写错的弱点编号会在整改单里长期存在，比没有编号更糟。

---

## 七、Mobile Top 10（2024）：一份风险沟通语言，不是验收基线

OWASP Mobile Top 10 的自述口径是榜单："The Mobile Top 10 identifies and lists the top ten vulnerabilities found in mobile applications."（参考资料 6），并且紧接着自己限定了一句："It should be noted that the risks to mobile applications do not stop at the Top 10"（参考资料 6）。2024 版的十条标题（参考资料 6）：

| 编号 | 公开标题（原文） | 本文归入的形态 | 清单层可见性 |
| --- | --- | --- | --- |
| M1 | Improper Credential Usage | W7 凭据与密钥形态 | 部分（清单只能证明"有可导出通道"，不能证明凭据在其中） |
| M2 | Inadequate Supply Chain Security | W8 韧性与供应链形态 | 部分（清单只暴露 SDK 声明的组件与权限，不暴露依赖来源） |
| M3 | Insecure Authentication/Authorization | W5 信任判定形态 | 无（需要代码与服务端证据） |
| M4 | Insufficient Input/Output Validation | W5、W6 | 部分（深链入口的存在可见，参数处理不可见） |
| M5 | Insecure Communication | W3 通道形态 | 是（明文开关与 `networkSecurityConfig` 声明可见） |
| M6 | Inadequate Privacy Controls | W3、W7 | 部分（备份与迁移通道可见，采集范围不可见） |
| M7 | Insufficient Binary Protections | W8 | 无（属韧性层，且不能与安全控制互相折抵） |
| M8 | Security Misconfiguration | W2、W4 | 是（导出、备份、可调试位三项是清单层主要产出） |
| M9 | Insecure Data Storage | W3、W7 | 部分（备份/迁移通道可见，落盘内容不可见） |
| M10 | Insufficient Cryptography | W7 | 无 |

这张表的第三、四列是本文加的分析，**OWASP 并未发布 Top 10 与 MASVS 域或 CWE 的官方一一映射表**。把它挂在这里是为了给沟通提供一个稳定的落点：当管理层说"这是 M8"时，工程侧要能回答"M8 在这一包里具体是哪一条声明、哪个组件、能不能在 CI 里复现"。上表给出的是这种回答的最短路径；十行里有三行是"无"，这三行也正是**不能用静态清单门禁交差**的部分（第十二章）。

榜单口径还有一个必须写进流程的性质：它会随年度调整而变化，而要求目录不会。如果把某次验收写成"覆盖 Top 10 全部十条"，第二年榜单调整时这张验收单就自动失效；写成"覆盖 MASVS 域 × 证据层矩阵中已登记的条目"才能跨年度延续。本系列的建议是：**Top 10 用于优先级排序与外部沟通，MASVS 域用于验收结构，CWE 用于与公告对齐，三者不要互相替换。**

---

## 八、CWE 与 CWE-919 移动视图：聚合语义与二十一个成员

CWE 提供的是缺陷类型编号，跨工具、跨公告、跨语言。它的移动视图（`CWE-919` "Weaknesses in Mobile Applications"）自述只有一句：

> "CWE entries in this view (slice) are often seen in mobile applications."（参考资料 7）

这句话的口径值得强调：视图的成员关系是"**常见于移动应用**"，不是"专属于移动平台"，更不是"移动侧的全部缺陷"。视图成员里只有一部分以 Android 应用为主语（CWE-925、926、927），其余是通用软件措辞（如 CWE-312、319、295 讲的是"以明文存储/传输""证书校验不当"这一类任何平台都会犯的缺陷）。第十六章的逐条对齐表对每一行都标注了"平台特定 / 通用定义"。

按 cwe.mitre.org 的视图页（参考资料 7），成员共 21 条，编号与官方标题如下（本文不做中文改名，只在此处原样列出以便与站内其它章节交叉引用）：

CWE-200 Exposure of Sensitive Information to an Unauthorized Actor；CWE-250 Execution with Unnecessary Privileges；CWE-295 Improper Certificate Validation；CWE-297 Improper Validation of Certificate with Host Mismatch；CWE-312 Cleartext Storage of Sensitive Information；CWE-319 Cleartext Transmission of Sensitive Information；CWE-359 Exposure of Private Personal Information to an Unauthorized Actor；CWE-362 Concurrent Execution using Shared Resource with Improper Synchronization ('Race Condition')；CWE-511 Logic/Time Bomb；CWE-602 Client-Side Enforcement of Server-Side Security；CWE-672 Operation on a Resource after Expiration or Release；CWE-772 Missing Release of Resource after Effective Lifetime；CWE-798 Use of Hard-coded Credentials；CWE-920 Improper Restriction of Power Consumption；CWE-921 Storage of Sensitive Data in a Mechanism without Access Control；CWE-925 Improper Verification of Intent by Broadcast Receiver；CWE-926 Improper Export of Android Application Components；CWE-927 Use of Implicit Intent for Sensitive Communication；CWE-939 Improper Authorization in Handler for Custom URL Scheme；CWE-940 Improper Verification of Source of a Communication Channel；CWE-941 Incorrectly Specified Destination in a Communication Channel。

其中三条的定义句直接落在 Android 组件模型上，本文引用的原句是（参考资料 8–10）：

- CWE-925："The Android application uses a Broadcast Receiver that receives an Intent but does not properly verify that the Intent came from an authorized source."
- CWE-926："The Android application exports a component for use by other applications, but does not properly restrict which applications can launch the component or access the data it contains."
- CWE-927："The Android application uses an implicit intent for transmitting sensitive data to other applications."

CWE-926 的定义句几乎就是第十八章脚本主判据的自然语言版本——"导出了组件"与"没有正确限制哪些应用可以启动它或访问它包含的数据"，正好对应 `exported=true` 与"三类访问权限属性是否齐备"两个观察点。这也解释了为什么脚本输出的 `provider_open` 被定到 `L3-HIGH` 而 `provider_guarded` 只是 `L1-INFO`：CWE 定义里的重点在**后半句**（限制是否到位），不在"导出"这个动作本身。

反过来看，视图里也有三条在移动安全治理里被系统性忽视：CWE-920（功耗限制不当）、CWE-772（资源未释放）与 CWE-672（失效后仍操作资源）。它们不是保密性缺陷，属于"可用性/资源生命周期"类，但在长期驻后台、跨进程共享资源、被系统低电策略干预的场景里，它们的真实业务代价（发热、被系统限制、静默失效）常常高于一条低风险数据面。第十二章会说明这三类**在清单层完全不可见**，只能靠运行时与功耗侧观测，因此它们也是本系列其它报告（沙箱篇、Binder 篇）覆盖的范围。

---

## 九、应用侧弱点的八种形态：分类要落到可重复观察的形态上

五套口径解决了"叫什么"和"归哪类"，但没有解决"在 Android 应用上它长什么样"。本文按**观察形态**而不是按标准条目，把移动应用弱点归成八种形态，记为 W1–W8。这一划分是本文的分析性构造（**不是任何标准的官方分类**），选择它的理由是：一种形态对应一类稳定证据，形态与证据同构，才能被自动化。

| 形态 | 名称 | 典型表现 | 稳定证据源 | 对应要求域 |
| --- | --- | --- | --- | --- |
| W1 | 暴露面形态 | 组件无条件导出、深链入口、可被外部寻址的 `authority` | 清单文本 | PLATFORM（v1 V6） |
| W2 | 默认值漂移形态 | 同一声明随 `targetSdkVersion` 含义变化；未显式书写导致旧默认继续生效 | 清单文本 + 平台行为文档 | PLATFORM / CODE（v1 V6/V7） |
| W3 | 通道形态 | 备份与设备间迁移、明文流量、TLS 信任锚与固定策略 | 清单文本（策略声明层） | STORAGE / NETWORK / PRIVACY |
| W4 | 构建身份形态 | 可调试位进入发行包、权限保护级别与签名不一致、构建变体泄漏 | 清单文本 + 签名与构建产物 | CODE（v1 V7） |
| W5 | 信任判定形态 | 隐式 Intent、未验证来源的接收器、`PendingIntent` 可变性、动态注册接收器 | 代码 + 运行时 | PLATFORM / AUTH（v1 V4/V6） |
| W6 | 跨环境数据形态 | WebView 与宿主文件系统的边界、脚本桥接口暴露、混合内容 | 代码 + 清单（部分） | PLATFORM（v1 V6） |
| W7 | 凭据与静态数据形态 | 硬编码凭据、明文存储、日志写入敏感数据、密钥可导出 | 代码 + 设备数据 + 服务端 | STORAGE / CRYPTO（v1 V2/V3） |
| W8 | 韧性与供应链形态 | 逆向与篡改抵抗（MASVS-R）、第三方 SDK 与依赖治理 | 二进制 + SBOM + 公告 | RESILIENCE / CODE（v1 V8/V7） |

三个使用要点。

**形态之间不做严重程度比较。** W1 与 W5 是"证据来源不同"的两类，不是"危害不同"的两级；一条真实的严重问题往往是 W1 与 W5 的组合（导出的入口 + 入口内部不校验调用者）。第十五章的组合形态给了四个这类例子。

**"清单层可见性"是形态的属性，不是脚本的属性。** W1–W4 大部分可以在清单文本上给出确定性结论，W5–W7 大部分不能，W8 基本不在清单里。第十章按这个事实把本文落点收窄。

**形态命名可以直接写进整改单。** "本包存在 W2（默认值漂移）风险敞口，具体为 targetSdk=27 导致明文流量旧默认继续生效"这一句同时包含形态、机制与可复现判据，比"高危：可能明文传输"更可追责。第十八、十九章脚本的输出结构（`级别 + 信号码 + 说明`）就是按这个句式设计的。

---

## 十、本文落点：只把"可静态复现"的那一格做成脚本

一个弱点检查工具的价值上限，由它能不能在 CI 上稳定复现决定，而不是由它覆盖多少条目决定。按这个标准，本文的选择是把落点放在**离线清单文本**这一格，理由有四：

**输入稳定。** 输入是 `aapt2 dump xmltree --file AndroidManifest.xml <自有 APK>` 展开出的文本，属于构建产物的一部分，不需要设备、不需要授权动态测试，也不受厂商 ROM 差异影响。

**判定确定。** 清单层的属性是存在性的：`android:exported` 要么写了要么没写，`android:permission` 三兄弟要么有一个要么全缺，`android:debuggable` 要么为 true 要么缺省。这些判定不需要概率模型，因此可以把"未采到"单独记成一档而不丢信息。

**可差分。** 弱点分类真正进入治理的时刻，是它能回答"这一版比上一版多了什么"。文本清点天然适合做集合差分——第十九章脚本做的全部事情就是两份清单的集合比较，并且把"差分前提"（两侧同源）也做成一行输出。

**可追责。** 每一条结论都能落到一个具体的组件名或一个具体的属性名，可以直接进变更单。这是第十五章缓解归属矩阵成立的前提。

代价同样要写清楚：这一格只覆盖 W1–W4 的一部分，对 W5–W8 基本给不出证据；它给出的是"声明形态",不是"可利用性"。第十一章列出可见信号的完整定义，第十二章列出这一格看不见的东西，第十四章把"看不见"固定成 `L0-LOW` 缺口而不是空白。这条纪律是整个设计里唯一真正影响结论质量的部分：**一个不会说"我不知道"的分类工具，一定会在某个版本开始说"我确定"。**

---

## 十一、清单层可见的弱点信号：四十一码位的判据与依据

本章把第十八章脚本的每一个信号码写清楚：**判据（清单里的什么条件）+ 等级 + 依据（哪一份公开文档说了这个默认值或这个行为）**。这一章是脚本的可审计说明书，也是把信号码交接给整改人时的解释口径。

### 输入形态与读取

脚本不接受 APK，只接受 `aapt2` 展开的清单文本——`aapt2 dump xmltree --file AndroidManifest.xml <自有 APK>` 的重定向输出。这一选择有三个原因：这是官方构建工具链的只读子命令；展开文本保留了编译后的**实际取值**（清单在打包时会经过合并与资源内联，源文件与产物可能不一致）；文本形式便于版本留存的比对。

编码细节必须写清楚，否则判读会错。布尔属性在这份文本里以资源类型呈现：`0xffffffff` 为 `true`、`0x0` 为 `false`；整型（如 `android:targetSdkVersion`）以 `0xNN` 十六进制呈现，本文脚本会把它换算成十进制再判定；`android:protectionLevel` 是位掩码，脚本只输出**原始值**并明确拒绝把它翻译成权限级别名（第十七章说明为什么）。属性归属按缩进层级解析：一个 `A:` 行归属于最近的、缩进更小的 `E:` 行，因此 `intent-filter` 内部的 `android:name`（动作名）不会污染组件自己的 `android:name`（类名）。

### 组一：身份与前提（决定其余判定的可信度）

| 信号码 | 判据 | 等级 | 依据 |
| --- | --- | --- | --- |
| `sdk_missing` | 未解析到 `android:targetSdkVersion` | L0-LOW | 与 API 级别绑定的默认值全部不可判定（参考资料 12、15） |
| `sdk_below_31` | `targetSdkVersion` < 31 | L2-WARN | Android 12 起的显式 `exported` 声明要求不适用，旧默认继续生效（参考资料 13） |
| `sdk_31_plus` | `targetSdkVersion` ≥ 31 | L1-INFO | 带 `intent-filter` 却未显式声明 `exported` 会直接装不上（参考资料 13） |
| `app_record` | 解析到 `application` 记录 | L1-INFO | 应用级属性（备份、明文、可调试）的读取前提 |
| `app_record_missing` | 没有 `application` 记录 | L0-LOW | 输入很可能不是清单展开文本，整次清点降级 |
| `no_components` | 四大组件与 `provider` 记录数为 0 | L0-LOW | 清点不成立，不得读成"没有暴露面" |

这一组没有一条是"弱点"，但它们是其余三十五个码位能否被信任的前提。第十四章的第一条纪律（未采到 = 缺口）正是为这一组设计的。

### 组二：W1 暴露面（组件导出与访问门槛）

| 信号码 | 判据 | 等级 | 依据 |
| --- | --- | --- | --- |
| `exported_absent` | 目标版本 ≥ 31，组件带 `intent-filter` 但未显式声明 `exported` | L2-WARN | 参考资料 13：此类包在 Android 12+ 设备上无法安装，说明该清单来源或合并结果存疑 |
| `exported_implicit` | 目标版本 < 31，组件带 `intent-filter` 但未显式声明 `exported` | L2-WARN | 参考资料 12、13：旧目标版本下过滤器的存在使隐式默认生效 |
| `exported_absent_no_filter` | 未声明 `exported` 且无过滤器 | L1-INFO | 默认不导出，但应显式书写以免重构引入过滤器 |
| `self_only` | `exported="false"` | L1-INFO | "If \"true\", the activity is accessible to any app, and is launchable by its exact class name."（参考资料 12）——取反即不进入同机跨应用可达面 |
| `provider_open` | 导出的 `provider`，且 `permission`/`readPermission`/`writePermission` 三者皆缺 | **L3-HIGH** | 参考资料 14：`exported="true"` 时"Any application can use the provider's content URI to access it, subject to the permissions specified for the provider"——权限未指定即无门槛 |
| `provider_guarded` | 导出的 `provider` 且至少声明一项访问权限 | L1-INFO | 继续核对权限保护级别与已被授予的应用集合 |
| `provider_grant` | 导出的 `provider` 且 `grantUriPermissions="true"` | L2-WARN | 参考资料 14：该属性允许"temporarily overcoming the restriction imposed by the readPermission, writePermission, permission, and exported attributes"，默认值为 `false` |
| `receiver_open_custom` | 导出的 `receiver`，无 `permission`，过滤器只含自定义动作 | **L3-HIGH** | CWE-925 与 CWE-926 的合取（参考资料 9、10）：入口开放且不校验 Intent 来源 |
| `receiver_system_action` | 同上但监听系统标准动作 | L1-INFO | 动作由系统发出，仍需在接收路径上校验调用者与数据 |
| `receiver_no_action` | 过滤器内未解析到动作 | L1-INFO | 无法判定监听集合，记为待核对 |
| `receiver_registered` | 导出但无过滤器 | L1-INFO | 动态注册或显式调用入口，属代码层证据范围（第十二章） |
| `receiver_guarded` | 导出但声明了发送方权限 | L1-INFO | 核对权限保护级别：仅 `signature` 级等价于同证书自动授予（参考资料 24） |
| `service_filter` | 导出的 `service` 且声明了 `intent-filter` | L2-WARN | 参考资料 17：官方建议"always use an explicit intent when starting a Service and do not declare intent filters for your services"（原文截断） |
| `service_exported` | 导出的 `service`、无过滤器且无 `permission` | L2-WARN | 任意同机应用可按精确类名绑定，服务侧必须自行校验调用者 |
| `activity_open_deeplink` | 导出的 `activity`、带 `VIEW` 过滤器且无权限门槛 | L2-WARN | CWE-939 形态（参考资料 7）：自定义 URL scheme 处理路径的授权缺失 |
| `activity_launcher` | 导出的 `activity`、带 `LAUNCHER` 类过滤器 | L1-INFO | 常规入口形态；进入后接收的数据仍需校验 |
| `activity_explicit_only` | 导出的 `activity`、无过滤器 | L1-INFO | 仅可被精确类名显式启动，仍需校验调用者 |
| `scheme_http` | 过滤器 `data` 的 `android:scheme` 含 `http` | L2-WARN | 深链参数与内容可在链路侧被观察；与 CWE-939/941 相关 |
| `scheme_https` | 同上含 `https` | L1-INFO | 仍需核对域名归属校验（`android:autoVerify`）与参数校验 |

这一组是清单层信息密度最高的部分，也是 CWE-926 那句定义的可执行版本："The Android application exports a component for use by other applications, but does not properly restrict which applications can launch the component or access the data it contains."（参考资料 9）——脚本把"exports"读作 `exported=true` 或隐式默认，把"does not properly restrict"读作三项权限属性齐缺，因此 `L3-HIGH` 的判定必须由**两个条件同时成立**触发，单独一个都不足以定高危。

### 组三：W3 通道（备份、迁移与明文）

| 信号码 | 判据 | 等级 | 依据 |
| --- | --- | --- | --- |
| `backup_unfiltered` | `allowBackup="true"` 且无 `dataExtractionRules`/`fullBackupContent` | L2-WARN | 参考资料 15：`allowBackup` 默认值即为 `"true"`；无排除规则时私有目录整体进入备份与迁移通道 |
| `backup_filtered` | `allowBackup="true"` 但存在排除规则 | L1-INFO | 核对排除项是否覆盖全部敏感路径 |
| `backup_off` | `allowBackup="false"` | L1-INFO | 参考资料 15：`"false"` 时"no backup or restore of the application is ever performed"，但厂商设备间迁移行为存在差异（同页 Note） |
| `backup_absent` | 未声明 `allowBackup` | L0-LOW | 默认值虽为 `true`，但本次未采集到显式声明，记缺口并要求补声明 |
| `cleartext_allowed` | `usesCleartextTraffic="true"` | L2-WARN | 参考资料 15：显式允许明文流量；同页 Note 已宣布该属性对高目标版本将被忽略 |
| `cleartext_denied` | `usesCleartextTraffic="false"` | L1-INFO | 显式关闭 |
| `cleartext_default_off` | 未声明该属性且目标版本 ≥ 28 | L1-INFO | 参考资料 15、16：目标 28+ 默认 `false`；"Starting with Android 9 (API level 28), cleartext support is disabled by default."（参考资料 16） |
| `cleartext_default_on` | 未声明该属性且目标版本 < 28 | L2-WARN | 参考资料 15：目标 27 或更低默认 `true`——**旧目标版本本身就是明文敞口** |
| `cleartext_undecidable` | 属性与目标版本都未解析 | L0-LOW | 不可判定 |
| `nsc_absent` | 未见 `networkSecurityConfig` 声明 | L0-LOW | 证书固定与信任锚策略无法从清单证实存在（参考资料 16） |
| `nsc_present` | 声明了 `networkSecurityConfig` | L1-INFO | 需另行读取该 XML 才能判定固定项与例外域 |

两个易被误读的细节。第一，`backup_absent` 的默认值其实是 `true`，本文仍记 `L0-LOW` 而不是 `L2-WARN`——理由是**缺省依赖不属于应用声明**，把它判成弱点会在整批老包上产出无法行动的噪声，正确动作是要求显式书写；第十四章把这条写成"默认值参与判定，但缺省不代替证据"。第二，`usesCleartextTraffic` 与 `networkSecurityConfig` 同时存在时，前者在 API 24+ 且存在 NSC 时被忽略（参考资料 15），因此脚本对两项分别出码而不合并——`cleartext_denied` + `nsc_present` 这种组合需要人工确认 NSC 里没有反过来允许明文的域配置。

### 组四：W4 构建身份与自定义权限

| 信号码 | 判据 | 等级 | 依据 |
| --- | --- | --- | --- |
| `debuggable` | `application` 的 `android:debuggable="true"` | **L3-HIGH** | 参考资料 15：该属性表示"Whether the application can be debugged, even when running on a device in user mode"，默认 `false`；出现在发行包中即构建流程缺陷 |
| `debuggable_false` | 显式 `false` | L1-INFO | 显式书写，判定可靠 |
| `debuggable_absent` | 清单未出现该属性 | L0-LOW | 发行构建通常为 `false`，但本次采集无法证明，记缺口 |
| `self_permission` | 解析到自定义 `permission` 声明 | L1-INFO | 输出 `protectionLevel` **原始数值**，不翻译级别名（位掩码语义不在本文判定范围，见第十七章） |
| `surface_zero` | 组件记录存在但无一 `exported="true"` | L1-INFO | 同机跨应用可达面在清单层面为空 |

`debuggable` 定为唯一的"构建身份类 L3"是有意的：它不描述某个组件、不描述某条通道，它描述**这个包以什么身份被构建**。平台侧后果（进程进入可调试执行域、私有数据可读性变化）在[沙箱与进程隔离篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)第九节，构建侧来源（`user`/`userdebug`/`eng` 变体与签名身份）在[平台代码组织与构建体系篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Platform-Code-Organization-and-Build-System-Analysis.html)，本文只保留一条纪律：**发行通道出现 `debuggable=true` 即阻断发布**，因为它是零成本、零歧义、零例外的信号。

---

## 十二、清单层看不见的四类：证据在别处

把"看不见"写成一章，比把"看得见"写成清单更有价值。以下四类弱点在分类体系里都有明确归属，但在清单文本上没有任何证据；给它们打勾等于伪造结论。

**第一类：代码逻辑层。** 组件收到了 Intent 之后怎么处理，是 CWE-925（"does not properly verify that the Intent came from an authorized source"，参考资料 8）与 CWE-940（Improper Verification of Source of a Communication Channel，参考资料 7、11）真正的落点。清单只能证明入口是否开放与是否有权限门槛，证明不了回调实现里有没有做来源校验。这一类需要的证据是代码审计与调用路径，或运行时对恶意构造 Intent 的行为验证——后者属授权测试范围，本文不给出流程。WebView 相关判定同理：`setAllowFileAccess()` 的默认值随目标版本变化（API 29 及以下为 `true`，API 30 及以上为 `false`，参考资料 21），但脚本看不见任何 `WebView` 配置调用；桥接接口的暴露后果有两句可直接引用的官方描述——"Using `addJavascriptInterface` lets JavaScript control your Android app"（原文截断）与"The addJavascriptInterface method exposes a Java object to all of a WebView's frames, and any frame can access the object name and call methods on it."（参考资料 22）——它们说明**这类判定必须逐处读代码**，清单级工具在此只能给出"本包使用了 WebView"这一条中性事实。

**第二类：运行时与状态层。** 动态注册的接收器、`PendingIntent` 的实际构造参数、跨进程资源生命周期（CWE-672、772）与功耗类缺陷（CWE-920）都在这里。清单只描述静态注册；`Context.registerReceiver()` 的过滤器与权限是运行期决定的，因此"导出接收器为零"这一条 L1 结论**不能**被读成"不存在开放接收路径"。同理，Android 12 起对 `PendingIntent` 可变性的要求（"If your app targets Android 12, you must specify the mutability of each PendingIntent object…"，参考资料 18）以及 5.0 起对服务隐式绑定的异常（"Beginning with Android 5.0 (API level 21), the system throws an exception if you call `bindService()` with an implicit intent."，参考资料 17）都是**代码层属性**，脚本给不出一行输出。

**第三类：服务端与流程层。** CWE-602（"Client-Side Enforcement of Server-Side Security"）是移动弱点分类里最被低估的一条：把校验放在客户端，等价于把策略交给可被替换的执行体。它的证据在服务端：同一接口对异常输入是否拒绝、令牌是否绑定设备与会话、风控是否独立生效。清单层对它的可见性为零，而 MASVS 的 V1（架构与威胁建模）与 Top 10 的 M3（认证与授权）都指向这里——这一格必须由集成测试与服务端审计覆盖，不能靠发布门禁交差。密钥形态也部分落在这里：平台侧承诺是"the key material remaining non-exportable"（参考资料 20）这一类**可配置属性**，是否真的配置为不可导出、是否被写进共享偏好或日志，需要代码与设备侧证据。

**第四类：隐私与数据内容层。** 清单可以证明备份通道是否开启，证明不了通道里有什么；可以证明存在导出 `provider`，证明不了它查询出的列是否含个人数据。CWE-359（"Exposure of Private Personal Information to an Unauthorized Actor"）与 CWE-312（"Cleartext Storage of Sensitive Information"，两者均属参考资料 7 的移动视图）都要求**数据内容级**证据：哪个文件、哪张表、哪一列、什么生命周期。这一层还需要与平台之外的口径对齐（商店的数据安全申报与业务侧的数据分级），本文只负责在缺口记录里明确指出"通道可见、内容不可见"这一区分，不替代数据分级工作。

平台侧的基线承诺可以作为这些层的参照，但不能当成本包的结论：内部存储的应用专属文件"No other apps can access app-specific files in internal storage."（参考资料 19），文件级加密的意义是"Encrypting at the file level, instead of encrypting the entire storage area as a single unit, better isolates and protects individual users…"（参考资料 19，原文截断），包完整性侧则是"During validation, v2+ scheme treats the APK file as a blob and performs signature checking across the entire file."（参考资料 23）。这四句都是**机制**描述，不是**本包**的证据——把机制描述抄进评估报告，是应用安全文档里最常见的一种自我说服。

---

## 十三、换算表 A：把要求域、CWE、Top 10 与信号码放在同一行

这张表是全文的汇合点：横向读是"这一域在清单层能看到什么"，纵向读是"这个信号码归属于哪些口径"。四个外部列（v1 区、v2 域、CWE 成员、Top 10）来自参考资料 1–7 的公开条目；两列本文自造（清单可见性、信号码），标注为本文分析。

| MASVS v1 区 | MASVS v2 域 | 清单层可见性 | 相关 CWE 成员（参考资料 7） | 对应 Top 10 | 本文信号码（脚本 1） |
| --- | --- | --- | --- | --- | --- |
| V1 架构与威胁建模 | （架构内容在 v2 中按域分散，无独立域标识） | 无 | 602、940 | M3、M4 | 无——需设计与评审证据 |
| V2 数据存储与隐私 | `STORAGE`、`PRIVACY` | 部分（通道可见，内容不可见） | 312、359、921、200 | M6、M9 | `backup_unfiltered`、`backup_filtered`、`backup_off`、`backup_absent` |
| V3 密码学 | `CRYPTO` | 无 | 295、297、798 | M10、M1 | `nsc_absent`、`nsc_present`（仅策略声明侧） |
| V4 认证与会话 | `AUTH` | 无 | 250、602、362 | M1、M3 | 无——需代码与服务端证据 |
| V5 网络通信 | `NETWORK` | 部分（开关与配置引用可见，策略内容不可见） | 319、295、297 | M5 | `cleartext_allowed`、`cleartext_denied`、`cleartext_default_off`、`cleartext_default_on`、`cleartext_undecidable`、`scheme_http`、`scheme_https` |
| V6 与平台环境交互 | `PLATFORM` | **是（本文主要落点）** | 925、926、927、939、941、921、200、672 | M4、M8 | `exported_*` 三条、`self_only`、`provider_*` 三条、`receiver_*` 五条、`service_*` 两条、`activity_*` 三条、`surface_zero` |
| V7 代码质量与构建设置 | `CODE` | 部分（构建标志可见，依赖与错误处理不可见） | 772、511、920 | M2、M8 | `debuggable`、`debuggable_false`、`debuggable_absent`、`self_permission`、`sdk_missing`、`sdk_below_31` |
| V8 韧性 | `RESILIENCE` | 无 | —— | M7 | 无——且不得折抵其它域（参考资料 1） |
| （跨切）目标版本 | 影响以上各域的默认判定 | 是 | 间接影响 312/319/926 的可判定性 | M8 | `sdk_31_plus`、`sdk_below_31`、`sdk_missing`、`app_record_missing`、`no_components` |

三点读法说明。

**"部分"不是"够用"。** `STORAGE` 那一行的"部分"意味着清单能证明通道开没开，不能证明通道里有什么。若把这一行读成"备份已检查"，就漏掉了 CWE-312/359 需要的内容级证据。第十五章的缓解归属矩阵就是按"部分"与"无"把责任分派到不同角色的。

**V1/V4/RESILIENCE 三行的空白是设计结果，不是遗漏。** 它们分别需要评审记录、服务端行为与二进制保护证据，静态清单工具在此的正确输出是"没有证据"，而不是"没问题"。第十八章脚本对这三域**一个码位都不出**——这是它能被信任的原因。

**目标版本是横切项。** 它不属于任何单一要求，却决定多条判定的方向（第十七章展开）。把它单独列成一行，是为了防止分类表被读成静态映射：Android 应用的弱点分类里，`targetSdkVersion` 是一个会改变其它条目语义的变量。

---

## 十四、分级模型：四个等级、四条判定纪律

等级不是为了排序美观，而是为了让"这条结论要求谁在什么时间内做什么"有唯一答案。本文使用四个等级，全部由脚本自动赋值，人工只能**降级并留痕**、不能静默升级。

| 等级 | 含义 | 判定条件 | 要求动作 | 不能读成 |
| --- | --- | --- | --- | --- |
| `L3-HIGH` | 声明形态本身已构成开放 | 由两个以上条件同时成立触发（如"导出 + 三项权限齐缺"） | 阻断发布；变更单必须落到具体组件与属性 | "已经发生泄露" |
| `L2-WARN` | 暴露面扩大或策略缺失，需核对后续实现 | 单条件成立，且后果依赖代码/服务端 | 进入整改队列并登记验收证据来源 | "可以不处理" |
| `L1-INFO` | 已收紧或中性事实 | 存在显式的收紧声明或常规形态 | 留档，作为差分基线 | "无需再看" |
| `L0-LOW` | **证据缺口** | 属性缺失、解析失败、输入退化 | 补采集，不允许写结论 | "低风险"或"通过" |

`L0-LOW` 是这个模型里唯一非直观的一档，也是最关键的一档。它的语义不是"低"，而是"本次不知道"。第十八章的输出里，同一个属性缺省（例如 `android:debuggable` 从未出现在清单中）会产生 `debuggable_absent` 而不是"未开启调试"；输入不是清单文本会产生 `no_components` 而不是"没有组件"。这两条写法在样例输出里可以直接看到（第十八章第三段样例）。

**四条纪律。**

**纪律一：未采到 = 缺口，不是通过。** 任何因解析失败、字段缺失或输入退化而产生的"无异常"，一律记 `L0-LOW` 并计入 `gap` 计数。这条纪律的运营后果是明确的：`SUMMARY` 行的 `gap=` 字段非零时，这次清点**不具备"合规"资格**，只能作为"已知事实清单"使用。第十九章第三段样例演示的就是这条：当前副本解析不出组件记录时，脚本输出 `curr_unparseable` 并要求把"消失"读作采集失败。

**纪律二：清单层只判声明形态的存在性，不判可利用性。** 所有 `L3-HIGH` 都必须写明触发条件的合取式（第十一章逐条给了），不得写"可被任意应用访问数据"这类跨越证据层的结论——正确的说法是"同机任意应用可按该 `authority` 寻址该数据面，且清单未声明任何访问权限门槛"，两者差别在于后者不声称数据可读、不声称内容敏感、不声称已被访问。

**纪律三：默认值参与判定，但缺省不代替证据。** 平台默认值会改变判定的方向（第十七章），因此 `targetSdkVersion` 缺失时必须先记 `sdk_missing` 缺口，再让其余依赖它的判定退化为不可判定，而不是套用某个默认值继续判。同一条逻辑也用于 `backup_absent`：`allowBackup` 的默认值虽然是 `true`，但缺省不是应用的显式声明，因此记缺口并要求补写，而不是直接判成 `backup_unfiltered`。

**纪律四：差分必须先确认同源。** 两份清单副本不携带包名与版本信息，脚本因此**每次**都输出一行 `pair_identity_unverified`（第十九章三段样例里都能看到它），要求操作者自行确认两侧来自同一条版本线。跨包比对产出的 `ADDED`/`REMOVED` 会包含组件改名与产品线差异，只看 `SUMMARY` 计数会得出错误结论——第二十章第四段样例专门演示这种错位读数。

等级与要求域的关系不能反过来用：不是"L3 对应某个 MASVS 域"，而是"某个域在清单层只能产出这两种等级，其余必须去别的证据层"。这是第十三章"部分/无"两列的实际用途，也是本文拒绝把 `L1-INFO` 写成"符合 MASVS-PLATFORM-x"的原因——**要求满足与否是评估结论，不是清单事实。**

---

## 十五、组合形态与缓解归属：单条都合规，合起来是弱点

分类工作真正的产出不是单条信号码，而是**组合**。以下四个组合在真实包里高频出现，每一条的单项都能在检查表上打勾。

**组合一：开放入口 + 内部不做来源校验。** 清单侧可见的是 `activity_open_deeplink` 或 `receiver_open_custom`（第十一章组二），代码侧不可见的是处理路径里有没有校验调用者与参数（CWE-925、940，参考资料 8、11）。清单给出的是"责任落在应用代码里"这一事实，整改单必须同时列出这两层，否则很容易只改清单（把 `exported` 关掉）而破坏外部集成，或只加代码校验而保留不必要的外部可达性。

**组合二：旧目标版本 + 通道默认值。** `sdk_below_31` 与 `cleartext_default_on` 常常同批出现（参考资料 13、15）：目标版本停留在 27 或更低时，明文流量默认是开的，而显式 `exported` 声明要求也不适用。这个组合的可怕之处不在任何单条，而在于**它随时间自动扩大**——设备侧的新约束只作用于高目标版本的应用，包不升级，就长期停在旧默认上。第十七章把这条写成可核对的版本分界表。

**组合三：备份通道 + 本地存储策略。** `backup_unfiltered`（`allowBackup="true"` 且无排除规则，参考资料 15）叠加"敏感文件写在应用私有目录"这一代码事实，构成 CWE-312/359/921 的实际暴露路径（参考资料 7）。清单能证明前半句，后半句需要数据分级。平台文档给出的修复方向也正是前半句可控的部分：把 `allowBackup` 保持为 `true` 并用 `android:dataExtractionRules` 排除敏感键或临时缓存（参考资料 15）。

**组合四：韧性控制折抵安全控制。** 一个包同时具备加固（MASVS-R 类控制）与 `provider_open`（L3），评估结论里却写"已加固，风险可控"。这一组合是标准明确禁止的：MASVS-R 列出的软件保护控制"can ultimately be bypassed and must never be used as a replacement for security controls."（参考资料 1）。韧性只改变攻击者需要投入的时间，不改变声明形态是否开放。

**缓解归属矩阵。** 把"谁能在什么阶段挡住哪一类"写清楚，分类才会进入流程。

| 阶段 | 挡得住的形态 | 挡不住 | 归属角色 | 可自动化程度 |
| --- | --- | --- | --- | --- |
| 设计评审 | W1（是否需要有外部入口）、W5（调用者身份模型）、V1 架构类 | 具体属性书写是否正确 | 架构师 + 安全负责人 | 低（依赖威胁建模记录） |
| 编码规范与 Lint | W2（缺省书写）、W1 单组件、W6（WebView 配置） | 跨组件数据流、服务端信任边界 | 应用开发者 | 中（IDE/Lint 可覆盖一部分） |
| CI 静态清点（本文脚本） | W1–W4 全部清单可见项 | W5–W8（第十二章四类） | 发布工程 + 应用安全 | 高（纯文本、无设备、可门禁） |
| 授权动态评估 | W5、W6 的运行时后果、W7 落盘与日志 | 服务端策略、长期回归 | 评估团队 | 低（人工 + 授权前提） |
| 服务端与接口测试 | W5 的授权判定、CWE-602 类 | 客户端实现细节 | 后端 + 安全 | 中 |
| 发布门禁与变更单 | 回归：新增 L3/L2（脚本 2） | 首次引入时的历史欠账 | 发布委员会 | 高（差分可判定） |
| 设备侧取证与检测 | 实际生效姿态（沙箱篇、Binder 篇） | 意图与流程 | 检测/取证工程师 | 中 |

这张表有一个必须点出的空缺：**没有任何一栏能替"数据内容分级"负责。** W3/W7 的"通道里有什么"落在业务与隐私角色上，工具只能提供通道是否开放的事实。这也是第二十章的指标集里必须有"缺口闭环率"（K9）而不能只有"高危数量"（K2）的原因。

---

## 十六、换算表 B：CWE-919 二十一个成员逐条对齐

视图成员取自 cwe.mitre.org 的 CWE-919 页面（参考资料 7），标题为原文；"定义主语"列区分该条定义句是**以 Android 应用为主语**还是**通用软件（product）措辞**——这一区分决定了它能不能被直接读成"Android 专属缺陷"。

| CWE | 官方标题（原文） | 定义主语 | 本文形态 | 清单可见性 | 需要的证据 |
| --- | --- | --- | --- | --- | --- |
| 200 | Exposure of Sensitive Information to an Unauthorized Actor | 通用 | W3、W7 | 部分 | 数据分级 + 落盘与传输路径 |
| 250 | Execution with Unnecessary Privileges | 通用 | W4、W5 | 部分 | 权限申请集合与运行身份（沙箱篇） |
| 295 | Improper Certificate Validation | 通用 | W3 | 部分 | NSC 内容与服务端证书链（参考资料 16） |
| 297 | Improper Validation of Certificate with Host Mismatch | 通用 | W3 | 无 | 域名校验实现与固定策略 |
| 312 | Cleartext Storage of Sensitive Information | 通用 | W7 | 无 | 文件与数据库内容 |
| 319 | Cleartext Transmission of Sensitive Information | 通用 | W3 | 部分 | 明文开关 + 实际链路（`cleartext_*` 四条） |
| 359 | Exposure of Private Personal Information to an Unauthorized Actor | 通用 | W7 | 无 | 采集范围与隐私声明 |
| 362 | Concurrent Execution using Shared Resource with Improper Synchronization ('Race Condition') | 通用 | W5 | 无 | 代码与并发路径 |
| 511 | Logic/Time Bomb | 通用 | W5 | 无 | 代码审计与行为观察 |
| 602 | Client-Side Enforcement of Server-Side Security | 通用 | W5 | 无 | 服务端对异常请求的独立拒绝能力 |
| 672 | Operation on a Resource after Expiration or Release | 通用 | W5 | 无 | 运行时状态与生命周期 |
| 772 | Missing Release of Resource after Effective Lifetime | 通用 | W5 | 无 | 运行时资源观测 |
| 798 | Use of Hard-coded Credentials | 通用 | W7 | 无 | 代码与产物内容 |
| 920 | Improper Restriction of Power Consumption | 通用 | W5 | 无 | 功耗与后台行为观测 |
| 921 | Storage of Sensitive Data in a Mechanism without Access Control | 通用 | W3、W7 | 部分 | 通道是否开放 + 内容是否敏感 |
| 925 | Improper Verification of Intent by Broadcast Receiver | **Android** | W1、W5 | 部分 | `receiver_open_custom` + 代码校验 |
| 926 | Improper Export of Android Application Components | **Android** | W1 | **是** | `exported_*` 与组件权限三兄弟 |
| 927 | Use of Implicit Intent for Sensitive Communication | **Android** | W5 | 部分 | 过滤器形态 + 实际 Intent 构造 |
| 939 | Improper Authorization in Handler for Custom URL Scheme | **Android** | W1 | 部分 | `activity_open_deeplink`、`scheme_http` + 参数处理 |
| 940 | Improper Verification of Source of a Communication Channel | 通用 | W5 | 无 | 通道建立与调用者身份 |
| 941 | Incorrectly Specified Destination in a Communication Channel | 通用 | W5 | 无 | 目的地绑定与校验 |

三条读法。

**二十一里只有四条以 Android 为主语**（925、926、927、939），它们也正是本文脚本主要覆盖的部分；其余十七条即便在移动视图里，判定仍需其它证据层。**"清单可见性 = 部分"的十四条是本报告最有用的输出**：它们说明"清单能证明一半"，因此任何只看清单的报告必须显式声明缺另一半；第十四章的纪律二就是为这些行写的。**通用措辞的成员不能被写成"Android 缺陷"**，只能在整改单里以"CWE-NNN 在 Android 上的具体形态是……"的句式出现，否则会引发平台间误用。

---

## 十七、目标版本与默认值漂移：同一条属性在两个包里是两类弱点

Android 应用清单里的很多属性**没有"绝对含义"**，它们的默认值与生效方式随包的 `targetSdkVersion` 变化。分类体系不会替你处理这一层，平台文档会，但散落在元素参考与版本行为变更页里。下面这张表是本文脚本判定的依据，全部为公开文档口径。

| 属性/机制 | 目标版本 | 官方默认或行为 | 本文判定变化 | 依据 |
| --- | --- | --- | --- | --- |
| `android:exported` 显式声明 | ≥ 31 | 带 `intent-filter` 的组件必须显式声明，否则无法安装："If an activity, service, or broadcast receiver uses intent filters and doesn't have an explicitly-declared value for `android:exported`, your app can't be installed on a device that runs Android 12 or higher." | 目标 ≥ 31 却缺声明 → `exported_absent`（判来源存疑）；目标 < 31 却缺声明 → `exported_implicit`（判隐式默认） | 参考资料 13 |
| `android:exported` 默认 | 目标 ≥ 17 | `provider` 的该属性自 API 17 引入，低版本设备按 `true` 处理；目标 ≥ 17 时在 API 17+ 设备上默认为 `false` | 本文不因"缺省"直接判导出，而是出 `exported_absent_no_filter`（L1）并要求显式书写 | 参考资料 14 |
| `android:usesCleartextTraffic` | 27 / 28 分界 | "The default value for apps that target API level 27 or lower is `\"true\"`. Apps that target API level 28 or higher default to `\"false\"`." | 未声明时按目标版本分派 `cleartext_default_on`（L2）或 `cleartext_default_off`（L1） | 参考资料 15 |
| 同一属性的未来 | ≥ 38 | 官方 Note：该属性正在弃用，"will be ignored for apps targeting API levels 38 and above" | 高目标版本包里，明文判定的权威来源只能是 NSC；本文仍出 `nsc_absent` 缺口 | 参考资料 15 |
| 明文与 NSC 的优先级 | ≥ 24 | 存在 Android Network Security Config 时该标志被忽略（"This flag is ignored on Android 7.0 (API level 24) and above if an Android Network Security Config is present."） | `cleartext_denied` 与 `nsc_present` 同时出现时必须再读 NSC 内容 | 参考资料 15、16 |
| 明文流量默认（平台侧） | API 28+ | "Starting with Android 9 (API level 28), cleartext support is disabled by default." | 同上，用于解释为什么旧目标版本是敞口 | 参考资料 16 |
| `android:allowBackup` | 全部 | "The default value of this attribute is `\"true\"`."；`"false"` 时"no backup or restore of the application is ever performed" | 缺省记 `backup_absent`（L0）而不是直接判未过滤；`true` 且无排除规则记 `backup_unfiltered`（L2） | 参考资料 15 |
| 设备间迁移 | ≥ 31 | 官方 Note：部分厂商设备上无法禁用应用文件的设备间迁移 | `backup_off` 的说明里保留"厂商差异"，不写成"通道已关闭" | 参考资料 15 |
| `android:debuggable` | 全部 | "The default value is `\"false\"`."（且该属性描述的是设备处于 user 模式时仍可调试） | 出现 `true` 记 L3 并阻断；缺省记缺口 | 参考资料 15 |
| `networkSecurityConfig` / 证书透明度 | API 36 | 平台侧默认关闭证书透明度，应用可按全局或按域选择开启 | 本文只出 `nsc_absent`/`nsc_present`，不推断 CT 状态 | 参考资料 16 |
| `WebView` 文件访问 | 29 / 30 分界 | `setAllowFileAccess()` 默认值在 API 29 及以下为 `true`，API 30 及以上为 `false` | 属代码层配置，脚本不可见；列入第十二章"看不见"证据需求 | 参考资料 21 |
| `PendingIntent` 可变性 | ≥ 31 | 目标 Android 12 及以上必须显式指定每个 `PendingIntent` 的可变性 | 属代码层，脚本不可见 | 参考资料 18 |
| 服务隐式绑定 | ≥ 21 | "Beginning with Android 5.0 (API level 21), the system throws an exception if you call `bindService()` with an implicit intent." | 与 `service_filter`（L2）同一动因：服务不应声明过滤器 | 参考资料 17 |

最后一列之外，还有一条方法性提醒：**`android:protectionLevel` 是位掩码**。脚本把它当原始数值输出（`self_permission` 行写"原始值=0x2"这类形式），不翻译为 `signature` 之类的级别名，因为该字段还包含若干修饰位；翻译需要按位解析与平台版本对照，超出清单文本能给出的确定性范围。权限级别的实际含义（只有同证书应用才自动获得）在参考资料 24，本文只在说明句里引用它一次，不在脚本输出里做等价判断。

这张表也是"**为什么不能用别人的扫描结果**"的答案：同一条 `allowBackup` 缺省，在目标 34 的包里与在目标 22 的包里，风险叙述完全不同；任何一个不携带目标版本上下文的"高危/低危"标签都不可复用。第二十章的第一项指标（K1 清单清点覆盖率）之所以要求"带目标版本上下文的清点数"，就是为这一层服务。

---

## 十八、脚本 1：`apk-manifest-weakness-scan.sh`（自有 APK 清单的离线弱点信号清点）

这是本文两个核验脚本中的第一个。它做一件事：把你**自有或已获授权测试**的应用包的清单展开文本，转成一份带等级、带信号码、可留档可差分的清点结果。

**它的边界先说清楚。** 不接触设备、不访问网络、不安装任何策略、不修改被检查对象、不解包、不反编译；输入是一份文本文件，输出是一份文本文件。它给出的结论全部属于"声明形态"层，不声称任何可利用性——第十一章组二里那条 L3 判据的完整措辞是"已导出且三项权限属性皆缺"，不是"数据已被读取"。它同样不给修复步骤：修哪个属性是读者的设计决定，本文不代替。

### 输入准备

输入来自官方构建工具链的只读子命令（`aapt2` 随 Android SDK Build-Tools 分发，参考资料 28）：

    aapt2 dump xmltree --file AndroidManifest.xml app.apk > demo-1.4.0.xmltree.txt

这条命令读取一个 APK 文件并把编译后的清单以资源树文本形式打印到标准输出。请用**你有权处理的应用包**：自有构建产物、或已获书面授权做静态清点的第三方包。两点实践提醒：其一，务必用**签名后的发行产物**而不是源码里的 `AndroidManifest.xml`，因为清单合并会引入依赖库声明的组件与权限（这类引入在源码文件里看不见）；其二，把这份文本连同包名、版本名与 `targetSdkVersion` 一起留档，它是脚本 2 的输入。

`aapt2` 的展开文本形如 `E: activity(…)` / `A: android:name(0x01010003)="…"` 的缩进树，属性值带资源类型标记。脚本按这个形态解析，因此**输入形态本身就是它的一个前提**：换了别的展开工具、别的输出格式，判读会退化为一堆 `L0-LOW` 缺口——这是设计行为，不是失败。

### 用法

    apk-manifest-weakness-scan.sh <aapt2-xmltree-文本> [副本输出路径]

- 第一个参数是必需输入文件。文件不可读或为空时输出 `ERR,input_unreadable,…` / `ERR,input_empty,…` 到标准错误并以退出码 1 结束，不产生任何"看起来正常"的输出。
- 第二个参数可选：把同一份清点结果**同时**写入该路径，作为脚本 2 的基线副本。写副本不改变标准输出内容，两者逐行一致（本文实测两份内容 `diff` 无差异）。
- 退出码：`0` 表示清点完成（**注意：清点完成不等于没有风险，也不等于通过**）；`1` 表示输入不可用。脚本不会因为出现 `L3-HIGH` 而返回非零，因为它被设计成"事实生产者"，门禁决策留给调用方。若要把它接进 CI，请按 `SUMMARY` 行的 `high=` 字段自行判定并设置阈值。

### 输出行种类

| 前缀 | 含义 | 消费方式 |
| --- | --- | --- |
| `META,<键>,<值>` | 全局事实：`target_sdk`、`components`、`exported_components` | 直接入库，作为分母与上下文 |
| `COMP,<类型>,<名称>,<属性串>` | 每个组件一行的原始属性快照 | 差分与统计的数据源；不参与等级判定 |
| `ROW,<等级>,<信号码>,<说明>` | 一条判定；说明句自带组件名与判据 | 整改单条目；信号码是稳定标识 |
| `SUMMARY,…` | `high/warn/gap/info` 四档计数与组件、导出数 | 门禁阈值与趋势统计 |

`SUMMARY` 是这一章最该被抄进流程的一行。它把"这次采集到底得出了什么"压成六个数字，且 `gap=` 显式告诉你有多少条是"没采到"。任何只引用 `high=`/`warn=` 而不引用 `gap=` 的报告，都在隐藏自己的证据缺口。

### 实现要点（便于自行复核或改造）

**属性归属按缩进解析，不按标签顺序。** 每个 `A:` 行归属于"最近的、缩进比它小的 `E:` 行"。这样 `intent-filter` 里的 `android:name`（动作名）不会被误认成组件的 `android:name`（类名），`data` 子元素的 `android:scheme` 会作为过滤器属性单独收集。脚本内部用一张"缩进 → 标签"的栈表实现，遇到同级或更浅缩进的 `E:` 行就丢弃不浅于它的所有记录。

**布尔值按资源编码判定。** `0xffffffff` 视为 `true`，结尾为 `0x0` 视为 `false`，其余整型原样输出为十六进制（`protectionLevel` 因此只出原始值，理由见第十七章末）。属性缺失时统一返回 `none` 而不是空串——这个区别很重要，因为大量判据是"三项权限属性是否齐缺"，空串与缺失混淆会导致把开放数据面误判成有门槛。

**过滤器存在性单独打标。** 组件记录里维护一个 `IF=` 标记，只要该组件下出现过 `intent-filter` 元素就置位，不依赖过滤器内部有没有解析到动作。原因是"只有 category 没有 action"的过滤器真实存在，如果用动作数推断过滤器，会漏掉隐式 `exported` 默认生效的情形。

**等级与计数同处产生，不事后统计。** 每一条 `ROW` 由同一个函数发出并同时累加计数，避免出现"输出有 4 条 L3、SUMMARY 报 3"的口径分叉——这类不一致在自建工具里极其常见，也是差分结果不可信的常见来源。

### 脚本全文

以下是本次实测使用的脚本原文，逐字未改；本章三段样例输出全部由它直接产生。

```bash
#!/usr/bin/env bash
# 脚本 1: apk-manifest-weakness-scan.sh — 自有应用清单的离线弱点信号清点
# 用法: apk-manifest-weakness-scan.sh <aapt2-xmltree-文本> [副本输出路径]
# 输入: `aapt2 dump xmltree --file AndroidManifest.xml app.apk` 重定向得到的文本
# 纯离线：不读设备、不访问网络、不写回任何被检查对象；副本仅用于前后两次清点对比
set -u
src="${1:?用法: apk-manifest-weakness-scan.sh <aapt2-xmltree-文本> [副本输出路径]}"
[ -r "$src" ] || { echo "ERR,input_unreadable,$src" >&2; exit 1; }
[ -s "$src" ] || { echo "ERR,input_empty,$src" >&2; exit 1; }
out="${2:-}"
if [ -n "$out" ]; then : > "$out" || { echo "ERR,output_unwritable,$out" >&2; exit 1; }; fi

hi=0; md=0; gap=0; info=0
row() {
  case "$1" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) md=$(( md + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
  printf 'ROW,%s,%s,%s\n' "$1" "$2" "$3"
  [ -n "$out" ] && printf 'ROW,%s,%s,%s\n' "$1" "$2" "$3" >> "$out"
  return 0
}
meta() {
  printf 'META,%s,%s\n' "$1" "$2"
  [ -n "$out" ] && printf 'META,%s,%s\n' "$1" "$2" >> "$out"
  return 0
}
comp_line() {
  printf 'COMP,%s,%s,%s\n' "$1" "$2" "$3"
  [ -n "$out" ] && printf 'COMP,%s,%s,%s\n' "$1" "$2" "$3" >> "$out"
  return 0
}

# 归一化：按缩进把每个 A 行挂到最近的、缩进更小的 E 行上，再按组件聚合
records="$(awk '
  function ind(s) { match(s, /^[ \t]*/); return RLENGTH }
  function tagof(s) { sub(/^[ \t]*E:[ \t]*/, "", s); sub(/[ \t]*\(.*$/, "", s); return s }
  function valof(s,   v) {
    sub(/^[^=]*=/, "", s); v = s
    gsub(/^[ \t]+|[ \t]+$/, "", v)
    if (v ~ /\(type 0x12\)0xffffffff/) return "true"
    if (v ~ /\(type 0x12\)0x0$/) return "false"
    if (v ~ /\(type 0x[0-9a-f]+\)0x[0-9a-f]+$/) { sub(/^.*\)0x/, "", v); return "0x" v }
    gsub(/"/, "", v)
    return v
  }
  function keyof(s) { sub(/\(0x[0-9a-f]+\).*/, "", s); gsub(/[ \t]/, "", s); return s }
  BEGIN { cur = ""; curname = ""; curattrs = "" }
  { curind = ind($0) }
  /^[ \t]*E:/ {
    t = tagof($0)
    for (k in st) if (k + 0 >= curind + 0) delete st[k]
    st[curind] = t
    if (t ~ /^(activity|activity-alias|service|receiver|provider|application|uses-sdk|permission)$/) {
      if (cur != "") printf "%s|%s|%s IF=%d\n", cur, curname, curattrs, iff
      cur = t; curname = ""; curattrs = ""; iff = 0
    }
    if (t == "intent-filter" && cur != "" && cur != "application" && cur != "uses-sdk") iff = 1
    next
  }
  /^[ \t]*A:/ {
    if (cur == "") next
    best = -1
    for (k in st) if (k + 0 < curind + 0 && k + 0 > best) best = k + 0
    owner = st[best]
    a = $0; sub(/^[ \t]*A:[ \t]*/, "", a)
    key = keyof(a); v = valof(a)
    if (owner == "intent-filter" || owner == "data" || owner == "action")
      curattrs = curattrs " F:" key "=" v
    else if (owner == "category")
      curattrs = curattrs " CAT:" key "=" v
    else {
      if (key == "android:name" && curname == "") curname = v
      curattrs = curattrs " " key "=" v
    }
    next
  }
  END { if (cur != "") printf "%s|%s|%s IF=%d\n", cur, curname, curattrs, iff }
' "$src")"

attr() { # attr <属性名> <记录> —— 取普通属性值，缺失返回 none
  local v
  v="$(printf '%s\n' "$2" | tr ' ' '\n' | awk -F= -v k="$1" '$1 == k && NF > 1 { print substr($0, index($0, "=") + 1); exit }')"
  [ -n "$v" ] && printf '%s' "$v" || printf 'none'
}
fattr() { # fattr <记录> <属性名> —— 取 intent-filter 子元素属性值
  printf '%s\n' "$2" | tr ' ' '\n' | awk -F= -v k="F:$1" '$1 == k && NF > 1 { print substr($0, index($0, "=") + 1); exit }'
}
apprec="$(printf '%s\n' "$records" | awk -F'|' '$1 == "application" { print $3; exit }')"
sdkrec="$(printf '%s\n' "$records" | awk -F'|' '$1 == "uses-sdk" { print $3; exit }')"
tsdk="$(attr android:targetSdkVersion "$sdkrec")"
tdec=""
case "$tsdk" in 0x*) tdec=$(( tsdk )) ;; ''|none) ;; *) tdec="$tsdk" ;; esac
meta target_sdk "$( [ -n "$tdec" ] && printf '%s' "$tdec" || printf 'unparsed(%s)' "${tsdk:-none}" )"

if [ -z "$tdec" ]; then
  row L0-LOW sdk_missing "清单未解析到 targetSdkVersion：与 API 级别绑定的默认值一律不可判定，不得读成安全"
elif [ "$tdec" -lt 31 ]; then
  row L2-WARN sdk_below_31 "targetSdk=$tdec 低于 31：Android 12 起的导出显式声明要求与相关默认收紧对本包不适用，旧默认继续生效"
else
  row L1-INFO sdk_31_plus "targetSdk=$tdec 不低于 31：导出属性必须显式声明，未声明将导致无法安装"
fi

if printf '%s\n' "$records" | grep -q '^application|'; then
  row L1-INFO app_record "application 记录已解析：可读取应用级属性"
else
  row L0-LOW app_record_missing "没有 application 记录：输入可能不是 AndroidManifest 的 xmltree 展开文本"
fi

total=0; exported=0
while IFS='|' read -r ctype cname cattrs; do
  case "$ctype" in activity|activity-alias|service|receiver|provider) ;; *) continue ;; esac
  [ -n "$ctype" ] || continue
  total=$(( total + 1 ))
  exp="$(attr android:exported "$cattrs")"
  perm="$(attr android:permission "$cattrs")"
  rperm="$(attr android:readPermission "$cattrs")"
  wperm="$(attr android:writePermission "$cattrs")"
  guri="$(attr android:grantUriPermissions "$cattrs")"
  auth="$(attr android:authorities "$cattrs")"
  hasfilter=0
  printf '%s\n' "$cattrs" | grep -q 'IF=1' && hasfilter=1
  schemes="$(printf '%s\n' "$cattrs" | tr ' ' '\n' | awk -F= '$1 == "F:android:scheme" { printf "%s,", substr($0, index($0, "=") + 1) }' | sed 's/,$//')"
  acts="$(printf '%s\n' "$cattrs" | tr ' ' '\n' | awk -F= '$1 == "F:android:name" { printf "%s,", substr($0, index($0, "=") + 1) }' | sed 's/,$//')"
  [ "$exp" = "true" ] && exported=$(( exported + 1 ))
  comp_line "$ctype" "${cname:-unnamed}" "exported=${exp:-none},permission=${perm:-none},readPermission=${rperm:-none},writePermission=${wperm:-none},grantUriPermissions=${guri:-none},authorities=${auth:-none},intentFilter=$hasfilter,scheme=${schemes:-none},action=${acts:-none}"

  if [ "$exp" = "none" ]; then
    if [ "$hasfilter" -eq 1 ]; then
      if [ -n "$tdec" ] && [ "$tdec" -ge 31 ]; then
        row L2-WARN exported_absent "$ctype/${cname:-unnamed} 带 intent-filter 却未显式声明 android:exported：目标版本已要求显式声明，该记录来源存疑"
      else
        row L2-WARN exported_implicit "$ctype/${cname:-unnamed} 带 intent-filter 但未显式声明 android:exported：旧目标版本下过滤器的存在会使隐式默认生效"
      fi
    else
      row L1-INFO exported_absent_no_filter "$ctype/${cname:-unnamed} 未声明 exported 且无 intent-filter：默认不导出，但应显式书写以免重构引入过滤器"
    fi
    continue
  fi
  if [ "$exp" = "false" ]; then
    row L1-INFO self_only "$ctype/${cname:-unnamed} 显式不导出：不进入同机跨应用可达面"
    continue
  fi

  case "$ctype" in
    provider)
      if [ "$perm" = "none" ] && [ "$rperm" = "none" ] && [ "$wperm" = "none" ]; then
        row L3-HIGH provider_open "provider/${cname:-unnamed} 已导出且 permission/readPermission/writePermission 三者皆缺：同机任意应用可按 authority(${auth:-未声明}) 寻址该数据面"
      else
        row L1-INFO provider_guarded "provider/${cname:-unnamed} 导出但声明了访问权限：继续核对该权限的保护级别与已被授予的应用集合"
      fi
      if [ "$guri" = "true" ]; then
        row L2-WARN provider_grant "provider/${cname:-unnamed} grantUriPermissions=true：临时 URI 授权随 Intent 传递，须核对可被授予的域范围与读权限门槛"
      fi
      ;;
    receiver)
      if [ "$perm" = "none" ] && [ "$hasfilter" -eq 1 ]; then
        case "$acts" in
          '') row L1-INFO receiver_no_action "$ctype/${cname:-unnamed} 过滤器内未解析到动作：无法判定监听的动作集合" ;;
          *android.intent.action.*) row L1-INFO receiver_system_action "receiver/${cname:-unnamed} 无发送方权限门槛但监听系统标准动作(${acts})：仍须在接收路径上校验调用者与数据" ;;
          *) row L3-HIGH receiver_open_custom "receiver/${cname:-unnamed} 无 android:permission 且过滤器只含自定义动作(${acts})：同机任意应用都能投递匹配的 Intent" ;;
        esac
      elif [ "$perm" = "none" ] && [ "$hasfilter" -eq 0 ]; then
        row L1-INFO receiver_registered "receiver/${cname:-unnamed} 导出但无 intent-filter：属动态注册或显式调用入口"
      else
        row L1-INFO receiver_guarded "receiver/${cname:-unnamed} 导出但带有发送方权限(${perm})：继续核对该权限的保护级别与可授予范围"
      fi
      ;;
    service)
      if [ "$hasfilter" -eq 1 ]; then
        row L2-WARN service_filter "service/${cname:-unnamed} 导出且声明 intent-filter(${acts:-无动作})：隐式启动服务无法确定响应者，服务侧亦无调用者身份约束"
      elif [ "$perm" = "none" ]; then
        row L2-WARN service_exported "service/${cname:-unnamed} 导出但无 android:permission：任意同机应用可按精确类名绑定该服务，服务侧必须自行校验调用者"
      fi
      ;;
    activity|activity-alias)
      if [ "$hasfilter" -eq 1 ] && [ "$perm" = "none" ]; then
        case "$acts" in
          *android.intent.action.VIEW*) row L2-WARN activity_open_deeplink "$ctype/${cname:-unnamed} 导出、带 VIEW 深链过滤器且无权限门槛：外部应用与外部内容可直接携带 Intent 数据到达该入口(${schemes:-无 scheme})" ;;
          *) row L1-INFO activity_launcher "$ctype/${cname:-unnamed} 导出且带过滤器(${acts:-无动作})：属启动器/常规入口形态，仍须校验进入后接收到的数据" ;;
        esac
      elif [ "$perm" = "none" ] && [ "$hasfilter" -eq 0 ]; then
        row L1-INFO activity_explicit_only "$ctype/${cname:-unnamed} 导出但无 intent-filter：仅可被精确类名显式启动，仍需校验调用者"
      fi
      ;;
  esac

  case ",$schemes," in
    *,http,*) row L2-WARN scheme_http "$ctype/${cname:-unnamed} 的 data scheme 含 http：深链参数与内容可在链路侧被观察" ;;
    *,https,*) row L1-INFO scheme_https "$ctype/${cname:-unnamed} 使用 https 深链：仍需核对域名归属校验(autoVerify)与参数校验" ;;
  esac
done <<EOF
$records
EOF

meta components "$total"
meta exported_components "$exported"
if [ "$total" -eq 0 ]; then
  row L0-LOW no_components "解析后没有任何四大组件记录：清点不成立，不得读成没有暴露面"
elif [ "$exported" -eq 0 ]; then
  row L1-INFO surface_zero "未发现 exported=true 的组件记录：同机跨应用可达面在清单层面为空"
fi

dbg="$(attr android:debuggable "$apprec")"
case "$dbg" in
  true)  row L3-HIGH debuggable "application android:debuggable=true：该应用可被调试附加，其进程同时进入可调试执行域" ;;
  false) row L1-INFO debuggable_false "android:debuggable=false" ;;
  *)     row L0-LOW debuggable_absent "清单未出现 debuggable 属性：发行构建通常为 false，但本次采集无法证明，记为缺口" ;;
esac

ab="$(attr android:allowBackup "$apprec")"
der="$(attr android:dataExtractionRules "$apprec")"
fbc="$(attr android:fullBackupContent "$apprec")"
case "$ab" in
  true)
    if [ "$der" = "none" ] && [ "$fbc" = "none" ]; then
      row L2-WARN backup_unfiltered "allowBackup=true 且无 dataExtractionRules/fullBackupContent：私有目录内容整体进入备份与设备间迁移通道"
    else
      row L1-INFO backup_filtered "allowBackup=true 但存在排除规则(extraction=${der:-none},fullBackupContent=${fbc:-none})：核对排除项是否覆盖全部敏感路径"
    fi
    ;;
  false) row L1-INFO backup_off "allowBackup=false：备份与设备间迁移通道关闭（部分厂商对设备间迁移的处理存在差异）" ;;
  *)     row L0-LOW backup_absent "allowBackup 未显式声明：默认值随平台与目标版本变化，本次不可判定" ;;
esac

ct="$(attr android:usesCleartextTraffic "$apprec")"
nsc="$(attr android:networkSecurityConfig "$apprec")"
case "$ct" in
  true)  row L2-WARN cleartext_allowed "usesCleartextTraffic=true：显式允许明文流量，凭据与会话内容可在链路侧被观察" ;;
  false) row L1-INFO cleartext_denied "usesCleartextTraffic=false：显式关闭明文流量" ;;
  *)
    if [ -n "$tdec" ] && [ "$tdec" -ge 28 ]; then
      row L1-INFO cleartext_default_off "未声明 usesCleartextTraffic 且 targetSdk=$tdec 不低于 28：平台默认禁止明文"
    elif [ -n "$tdec" ]; then
      row L2-WARN cleartext_default_on "未声明 usesCleartextTraffic 且 targetSdk=$tdec 低于 28：平台默认允许明文"
    else
      row L0-LOW cleartext_undecidable "明文默认值取决于 targetSdk，本次两项皆未解析：不可判定"
    fi
    ;;
esac
if [ "$nsc" = "none" ]; then
  row L0-LOW nsc_absent "未见 networkSecurityConfig 声明：证书固定与信任锚策略无法从清单证实存在"
else
  row L1-INFO nsc_present "networkSecurityConfig=${nsc}：需另行读取该 XML 才能判定固定项与例外域"
fi

plcrec="$(printf '%s\n' "$records" | awk -F'|' '$1 == "permission" { print $2 "|" $3; exit }')"
if [ -n "$plcrec" ]; then
  plname="$(printf '%s\n' "$plcrec" | cut -d'|' -f1)"
  plvl="$(attr android:protectionLevel "$(printf '%s\n' "$plcrec" | cut -d'|' -f2)")"
  row L1-INFO self_permission "自定义权限 ${plname:-unnamed} 的 protectionLevel 原始值=${plvl:-none}：仅 signature 级等价于同证书自动授予"
fi

printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,components=%s,exported=%s\n' "$hi" "$md" "$gap" "$info" "$total" "$exported"
[ -n "$out" ] && printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,components=%s,exported=%s\n' "$hi" "$md" "$gap" "$info" "$total" "$exported" >> "$out"
exit 0
```

### 样例 A：一个收紧良好的包（`demo-1.4.0.xmltree.txt`）

下面是这个输入的实际输出全文（未删减；组件名与包名为示例值）：

```text
META,target_sdk,34
ROW,L1-INFO,sdk_31_plus,targetSdk=34 不低于 31：导出属性必须显式声明，未声明将导致无法安装
ROW,L1-INFO,app_record,application 记录已解析：可读取应用级属性
COMP,activity,com.example.demo.MainActivity,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=none,action=android.intent.action.MAIN
ROW,L1-INFO,activity_launcher,activity/com.example.demo.MainActivity 导出且带过滤器(android.intent.action.MAIN)：属启动器/常规入口形态，仍须校验进入后接收到的数据
COMP,activity,com.example.demo.InternalEditor,exported=false,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=0,scheme=none,action=none
ROW,L1-INFO,self_only,activity/com.example.demo.InternalEditor 显式不导出：不进入同机跨应用可达面
COMP,provider,androidx.core.content.FileProvider,exported=false,permission=none,readPermission=none,writePermission=none,grantUriPermissions=true,authorities=com.example.demo.fileprovider,intentFilter=0,scheme=none,action=none
ROW,L1-INFO,self_only,provider/androidx.core.content.FileProvider 显式不导出：不进入同机跨应用可达面
COMP,service,com.example.demo.SyncService,exported=false,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=0,scheme=none,action=none
ROW,L1-INFO,self_only,service/com.example.demo.SyncService 显式不导出：不进入同机跨应用可达面
COMP,receiver,com.example.demo.boot.BootReceiver,exported=true,permission=android.permission.RECEIVE_BOOT_COMPLETED,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=none,action=android.intent.action.BOOT_COMPLETED
ROW,L1-INFO,receiver_guarded,receiver/com.example.demo.boot.BootReceiver 导出但带有发送方权限(android.permission.RECEIVE_BOOT_COMPLETED)：继续核对该权限的保护级别与可授予范围
META,components,5
META,exported_components,2
ROW,L0-LOW,debuggable_absent,清单未出现 debuggable 属性：发行构建通常为 false，但本次采集无法证明，记为缺口
ROW,L1-INFO,backup_off,allowBackup=false：备份与设备间迁移通道关闭（部分厂商对设备间迁移的处理存在差异）
ROW,L1-INFO,cleartext_denied,usesCleartextTraffic=false：显式关闭明文流量
ROW,L1-INFO,nsc_present,networkSecurityConfig=@xml/network_security_config：需另行读取该 XML 才能判定固定项与例外域
SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
```


这份输出值得逐行读，因为它示范了"一个干净的包"在该模型下应该长什么样。

**十一条 `ROW` 里十条是 `L1-INFO`，唯一非 L1 的是 `debuggable_absent`（`L0-LOW`）。** 这不是缺陷，是纪律一的直接体现：这个包确实没有任何开放声明，但清单里从来没有出现过 `debuggable` 属性，脚本不能把"平台默认为 `false`"当作"本包已证明不是可调试构建"——于是记一条缺口，要求构建流程显式书写它。**最干净的包也至少带着一个缺口**，这是本报告反对"零告警即通过"的实证理由。

**`exported=2` 不等于"两个问题"。** 这两个导出组件是启动器 `activity` 与监听 `BOOT_COMPLETED` 的接收器，都是必要的入口，分别被记为 `activity_launcher` 与 `receiver_guarded`。第二十章的 K3 指标（导出面规模）因此必须与信号码联读：一个包可以有 2 个导出组件却全是 L1，也可以有 2 个却出现 `provider_open`。

**不导出的组件不判通道敞口。** `FileProvider` 这一行带着 `grantUriPermissions=true`，但因为 `exported=false`，脚本只输出 `self_only`，不输出 `provider_grant`。这一处理刻意保守：`grantUriPermissions` 的意义是"暂时克服 `readPermission`/`writePermission`/`permission` 和 `exported` 的限制"（参考资料 14），在组件本身不对同机其他应用开放时，它不构成跨应用可达面。**信号码之间存在条件依赖，不叠加告警是减少噪声的主要手段。**

**带发送方权限不等于安全。** `BootReceiver` 声明了 `android.permission.RECEIVE_BOOT_COMPLETED`，脚本记 `L1-INFO` 并在说明里保留一句"继续核对该权限的保护级别与可授予范围"。原因见第十七章末：权限的实际门槛由 `protectionLevel` 决定，只有同签名才自动授予的那一类才等价于"同证书应用可用"（参考资料 24），而清单文本里的 `protectionLevel` 是位掩码，脚本不做翻译。

### 样例 B：一个多点放宽的包（`legacy-1.0.0.xmltree.txt`）

同一个脚本、同一套判据，输入换成一个目标版本停在 27、五个组件全部导出且多数无门槛的包：

```text
META,target_sdk,27
ROW,L2-WARN,sdk_below_31,targetSdk=27 低于 31：Android 12 起的导出显式声明要求与相关默认收紧对本包不适用，旧默认继续生效
ROW,L1-INFO,app_record,application 记录已解析：可读取应用级属性
COMP,activity,com.example.bad.MainActivity,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=none,action=android.intent.action.MAIN
ROW,L1-INFO,activity_launcher,activity/com.example.bad.MainActivity 导出且带过滤器(android.intent.action.MAIN)：属启动器/常规入口形态，仍须校验进入后接收到的数据
COMP,activity,com.example.bad.PayActivity,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=http,action=android.intent.action.VIEW
ROW,L2-WARN,activity_open_deeplink,activity/com.example.bad.PayActivity 导出、带 VIEW 深链过滤器且无权限门槛：外部应用与外部内容可直接携带 Intent 数据到达该入口(http)
ROW,L2-WARN,scheme_http,activity/com.example.bad.PayActivity 的 data scheme 含 http：深链参数与内容可在链路侧被观察
COMP,provider,com.example.bad.db.RowProvider,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=true,authorities=com.example.bad.provider,intentFilter=0,scheme=none,action=none
ROW,L3-HIGH,provider_open,provider/com.example.bad.db.RowProvider 已导出且 permission/readPermission/writePermission 三者皆缺：同机任意应用可按 authority(com.example.bad.provider) 寻址该数据面
ROW,L2-WARN,provider_grant,provider/com.example.bad.db.RowProvider grantUriPermissions=true：临时 URI 授权随 Intent 传递，须核对可被授予的域范围与读权限门槛
COMP,service,com.example.bad.ExportService,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=none,action=com.example.bad.ACTION_DO
ROW,L2-WARN,service_filter,service/com.example.bad.ExportService 导出且声明 intent-filter(com.example.bad.ACTION_DO)：隐式启动服务无法确定响应者，服务侧亦无调用者身份约束
COMP,receiver,com.example.bad.CommandReceiver,exported=true,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=1,scheme=none,action=com.example.bad.CMD
ROW,L3-HIGH,receiver_open_custom,receiver/com.example.bad.CommandReceiver 无 android:permission 且过滤器只含自定义动作(com.example.bad.CMD)：同机任意应用都能投递匹配的 Intent
META,components,5
META,exported_components,5
ROW,L3-HIGH,debuggable,application android:debuggable=true：该应用可被调试附加，其进程同时进入可调试执行域
ROW,L2-WARN,backup_unfiltered,allowBackup=true 且无 dataExtractionRules/fullBackupContent：私有目录内容整体进入备份与设备间迁移通道
ROW,L2-WARN,cleartext_allowed,usesCleartextTraffic=true：显式允许明文流量，凭据与会话内容可在链路侧被观察
ROW,L0-LOW,nsc_absent,未见 networkSecurityConfig 声明：证书固定与信任锚策略无法从清单证实存在
ROW,L1-INFO,self_permission,自定义权限 com.example.bad.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION 的 protectionLevel 原始值=0x2：仅 signature 级等价于同证书自动授予
SUMMARY,high=3,warn=7,gap=1,info=3,components=5,exported=5
```


这份输出是"多点放宽"的标准形态，五个组件全部导出、三条 `L3-HIGH`、七条 `L2-WARN`，且**只有一条 `L0-LOW`**（`nsc_absent`）。最后这一项常被忽略，但它是这份报告质量的关键：这个包的问题不在于"看不见"，而在于"全都写在纸上"。诚实的差配置比沉默的不确定更容易处理，因为它每条都能落到一个组件名上。

**三条 `L3-HIGH` 各自的合取条件。** `debuggable` 只靠一条属性（`application` 的 `android:debuggable="true"`，参考资料 15 明确其默认值是 `false`，因此显式 `true` 必然是构建参数决定）；`provider_open` 需要两条同时成立（导出 + 三项权限属性齐缺）；`receiver_open_custom` 需要三条（导出 + 无 `permission` + 过滤器动作集合里没有系统标准动作）。这三条的触发条件都写进了说明句本身，因此复核时不需要重读脚本。

**旧目标版本带来的双重敞口。** `sdk_below_31`（L2）与 `cleartext_default_on` 是同一根因的两个结果——但这份输出里没有出现 `cleartext_default_on`，因为这个包把 `usesCleartextTraffic` 显式写成了 `true`，于是被判为 `cleartext_allowed`（L2）。第十七章的漂移表在这里可以直接验证：目标 27 时即使不写这一属性也会得到 L2 结论，写了只是让证据更确定。**目标版本决定的是"默认值风险"，显式声明决定的是"证据强度"，两者是两个维度。**

**自定义权限的正确处理是不翻译。** `self_permission` 这一行输出 `protectionLevel` 的原始值（本例为 `0x2`）并保留一句"仅 signature 级等价于同证书自动授予"。脚本没有把 `0x2` 断言成 `signature`：该字段是位掩码，还包含修饰位，且不同平台版本对取值的处理有差异（第十七章末）。这是一条典型的"看起来可以多做一步，但多做的这一步会引入不可复核结论"的位置。

**深链的两条码位不能合并。** `activity_open_deeplink`（入口开放）与 `scheme_http`（链路可观察）说的是两件不同的事，整改责任人常常也不是同一人：前者是组件设计，后者是网络与域名归属策略。第十五章的组合一在这个包上的完整形态是"这两条码位 + 一段未读过的 `onCreate()` 参数处理"。

### 样例 C：一份信息不足的清单（`unknown-1.2.0.xmltree.txt`）

第三段样例最重要：**一个没有任何告警、却几乎没有任何证据的输入**。它没有解析到 `targetSdkVersion`，两个组件都没有声明 `exported`，也没有任何应用级通道属性：

```text
META,target_sdk,unparsed(none)
ROW,L0-LOW,sdk_missing,清单未解析到 targetSdkVersion：与 API 级别绑定的默认值一律不可判定，不得读成安全
ROW,L1-INFO,app_record,application 记录已解析：可读取应用级属性
COMP,activity,com.example.unknown.MainActivity,exported=none,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=none,intentFilter=0,scheme=none,action=none
ROW,L1-INFO,exported_absent_no_filter,activity/com.example.unknown.MainActivity 未声明 exported 且无 intent-filter：默认不导出，但应显式书写以免重构引入过滤器
COMP,provider,com.example.unknown.RowProvider,exported=none,permission=none,readPermission=none,writePermission=none,grantUriPermissions=none,authorities=com.example.unknown.provider,intentFilter=0,scheme=none,action=none
ROW,L1-INFO,exported_absent_no_filter,provider/com.example.unknown.RowProvider 未声明 exported 且无 intent-filter：默认不导出，但应显式书写以免重构引入过滤器
META,components,2
META,exported_components,0
ROW,L1-INFO,surface_zero,未发现 exported=true 的组件记录：同机跨应用可达面在清单层面为空
ROW,L0-LOW,debuggable_absent,清单未出现 debuggable 属性：发行构建通常为 false，但本次采集无法证明，记为缺口
ROW,L0-LOW,backup_absent,allowBackup 未显式声明：默认值随平台与目标版本变化，本次不可判定
ROW,L0-LOW,cleartext_undecidable,明文默认值取决于 targetSdk，本次两项皆未解析：不可判定
ROW,L0-LOW,nsc_absent,未见 networkSecurityConfig 声明：证书固定与信任锚策略无法从清单证实存在
SUMMARY,high=0,warn=0,gap=5,info=4,components=2,exported=0
```


这份输出**没有任何 L3、没有任何 L2**，`SUMMARY` 是 `high=0,warn=0,gap=5,info=4,components=2,exported=0`。如果只看前两列，它比样例 A 还干净；但它恰恰是三类样例里最不可信的一份，因为 `gap=5`。

**`exported=0` 的来源不是"显式不导出"。** 两个组件都写着 `exported=none`（未声明），脚本因此出 `exported_absent_no_filter` 并在说明里保留"默认不导出，但应显式书写以免重构引入过滤器"。这个默认结论依赖目标版本上下文，而 `targetSdkVersion` 本次未解析（`sdk_missing`）——也就是说，支撑这条 L1 结论的前提本身已经记为缺口。**这是本报告里最能说明"计数会骗人"的一处：`exported=0` 与 `exported_components` 分母里的"未声明"是两种完全不同的事实。**

**带 `authorities` 的 `provider` 在这里没有被判成数据面。** `com.example.unknown.RowProvider` 声明了 `authorities`，但没有导出声明。脚本没有把"存在 authority"当成开放——这是对的（`provider_open` 要求导出成立），但也暴露了清单层的固有局限：**authority 名字本身就是可被外部猜到的标识符**，一旦后续某次重构把它改成 `exported="true"`（或把目标版本降到使隐式默认生效的区间），它会立刻变成 L3。这正是第十九章差分脚本要盯的情形，也是第二十章 K8 指标（差分回归发现数）存在的理由。

**四个通道类判定全部退化为缺口。** `debuggable_absent`、`backup_absent`、`cleartext_undecidable`、`nsc_absent` 四条 L0 连成一排，读法只有一种：**这份清单不足以支撑任何"通道已关闭"的结论**。正确动作是补采集（要求构建流程显式书写这几项，或换用签名后的发行产物重新展开），而不是补结论。样例 A 只有 1 条 L0，样例 B 只有 1 条 L0，样例 C 有 5 条——**这三份输出的 `gap` 列，就是本文对"分类落地"最直接的度量**。

### 已知局限（改造前请先读这段）

1. **只看清单，不看资源。** `networkSecurityConfig` 指向的那份 XML、`dataExtractionRules` 指向的规则文件、`file_paths.xml` 暴露的目录集合，都在脚本范围之外；脚本只证明"有没有声明"，不证明"声明里写了什么"。
2. **不读权限请求集合。** `uses-permission` 与它的危险级别、以及运行时授权状态不参与判定；本脚本判的是"谁能进来"，不是"应用要了什么"。
3. **不解组件内部。** 过滤器里的 `data` 只取 `scheme`，`host`/`pathPrefix` 等归属校验细节不判定；`android:autoVerify` 是否设置也未判定（样例里 `scheme_https` 的说明句保留了这个待办）。
4. **不做合并来源归因。** 一个由依赖库引入的导出组件与你自己写的导出组件，在输出里形态完全相同；需要归因时要另外读取合并后的清单与库的原始清单。
5. **不处理 split APK / AAB 的合并视图。** 每个 APK 各跑一次并各存一份副本；跨 split 的组件集合需要你自己合并，否则差分会把"分包变化"读成"暴露面变化"。
6. **差分键是"类型/名称"。** 组件改名在差分里表现为一条消失 + 一条新增，这是第十九章 `pair_identity_unverified` 每次都要输出的原因。
7. **不判定任何动态行为。** 运行时注册的接收器、`PendingIntent` 可变性、WebView 配置、服务端校验，都不在本脚本的证据范围内（第十二章）。

### 接入发布流程的最小写法

脚本被设计成"事实生产者"，门禁阈值留给调用方。最小接线是三行：把清点副本随构建产物入库、按 `SUMMARY` 的 `high=` 与 `gap=` 两个字段设置阻断条件、把 `ROW` 全量转成结构化条目。

    xml=demo-1.5.0.xmltree.txt; copy=snapshots/demo-1.5.0.txt
    apk-manifest-weakness-scan.sh "$xml" "$copy" > /dev/null
    awk -F, '/^SUMMARY/{ if ($2 ~ /high=[1-9]/ || $4 ~ /gap=[3-9]/) exit 1 }' "$copy"

这段是示意，不是建议阈值：`gap=3` 起阻断只对"要求显式声明"的团队有意义，一个仍在合并大量旧依赖的组织会立刻被淹没。**先跑两周只记录不阻断，再用自家分布定阈值**——第二十章的 K1 到 K10 就是为这两周的观察期准备的口径。

第十八、十九章合起来构成一个完整闭环：前者产出**事实**（这个包声明了什么形态），后者产出**变化**（这次发布比上一次多做了什么）。单独使用后者没有意义——没有基线副本就没有差分；单独使用前者则只能回答"现在什么样"，回答不了"是不是这次改坏的"，而后者才是发布流程真正需要的那一句。

---

## 十九、脚本 2：`weakness-surface-diff.sh`（两份清单副本的离线漂移比对）

第二个脚本比第一个更短，但它在治理流程里的位置更重要：**它把"弱点分类"从一次性的报告变成一条回归判定线**。它读两份脚本 1 写出的副本，输出新增与消失的导出组件、新增与消失的信号码、以及两次都未闭合的缺口。

**它不接触设备、不访问网络、不读 APK**，只读两份文本；两份输入都只以只读方式打开，比对结果写到标准输出（需要留档时由调用方自行重定向）。这一点决定了它可以安全地跑在任何构建机上，也不需要测试授权——它处理的是你自己留存的构建产物文本。

### 用法

    weakness-surface-diff.sh <基线清点副本> <当前清点副本>

- 两个参数都是脚本 1 的**副本输出**（不是 `aapt2` 的原始展开文本）。传错文件类型不会崩溃，但会退化为一排 `L0-LOW`，其中最有意义的一条是 `curr_unparseable` / `base_unparseable`（见下文样例 C）。
- 文件不可读或为空时输出 `ERR,…` 到标准错误并以退出码 1 结束。
- 输出行种类与脚本 1 一致（`META` / `ROW` / `SUMMARY`），另加三种明细行：`ADDED,<类型/名称>`、`REMOVED,<类型/名称>`、`CODE_NEW,<信号码>` / `CODE_GONE,<信号码>`。

### 比对的三件事与一件前提

**导出面集合。** 取两侧 `COMP` 行里 `exported=true` 的组件，构成"类型/名称"集合后做集合差。新增项按类型分级：新增导出 `provider` 判 `L3-HIGH`（`surface_provider_added`），新增导出 `receiver`/`service` 判 `L3-HIGH`（`surface_input_added`），其余（`activity`、`activity-alias`）判 `L2-WARN`（`surface_activity_added`）。这个分级不是按"组件类型天然危险"排的，而是按**清单层能证明多少**排的：新增导出 `provider` 的判据里已经包含"数据面可被外部按 authority 寻址"，而新增导出 `activity` 的实际风险取决于入口代码，所以低一档。

**信号码集合。** 只取两侧的 `L3-HIGH` 与 `L2-WARN` 码位做集合差，因为这两档才是需要回归跟踪的；新增高危码位聚合成一条 `high_signal_new`，新增告警码位聚合成一条 `warn_signal_new`，消失的码位一律判 `L2-WARN` 的 `signal_gone`——**"消失"在本模型里不是好消息**，它有两种解释：被修好了，或者这次采不到了（例如目标版本解析失败会让一批默认值判定整体退化为缺口）。这两种解释的整改动作完全不同，因此脚本拒绝替读者选一个。

**缺口集合。** 取两侧 `L0-LOW` 码位的交集，输出 `gap_persistent`：同一个盲区在两次清点里都没有闭合。这一条是第十四章纪律一在差分层的落点，也是第二十章 K9（缺口闭环率）的数据来源。

**一件前提：两侧必须同源。** 副本里没有包名、版本名与构建号，脚本因此每次固定输出一条 `pair_identity_unverified`（L1），要求操作者自己确认这两份副本来自同一条版本线。第四段样例（第二十章）演示了违反这条前提时的读数陷阱。

### 十五个差分信号码

与第十一章对脚本 1 的处理相同，这里把脚本 2 会发出的每一条信号码列全，便于把输出直接映射到整改动作。判据全部是**集合关系**，不是绝对数量，因此同一码位在不同配对里含义相同。

| 信号码 | 等级 | 触发条件（集合语义） | 读法 |
| --- | --- | --- | --- |
| `surface_provider_added` | L3-HIGH | 新增导出面的差集中出现 `provider` | 数据面新进入同机跨应用可达面；立即核对读写权限与 URI 授权范围 |
| `surface_input_added` | L3-HIGH | 新增导出面的差集中出现 `receiver` 或 `service` | 新增可投递/可绑定入口；清单层无法证明调用者校验存在，责任在代码内 |
| `surface_activity_added` | L2-WARN | 新增导出面的差集中只有 `activity`/`activity-alias` | 新增可启动入口；风险取决于入口代码，因此低一档 |
| `high_signal_new` | L3-HIGH | 本次新增的 L3 类或默认放宽类码位（`debuggable`、`provider_open`、`receiver_open_custom`、`exported_implicit`、`exported_absent`）集合非空 | 属回归；按变更单定位引入它的提交 |
| `warn_signal_new` | L2-WARN | 本次新增的暴露面或策略缺失类 L2 码位集合非空 | 进入整改队列，并核对上一轮的验收记录 |
| `signal_gone` | L2-WARN | 基线中的 L3/L2 码位在本次不再出现 | 两种解释（整改生效 / 这次采不到）动作完全不同，脚本不替你选 |
| `gap_persistent` | L0-LOW | 两侧 `L0-LOW` 码位集合的交集非空 | 同一盲区两次未闭合；做成 K9 缺口闭环率的输入，而不是告警 |
| `curr_unparseable` | L0-LOW | 当前副本无组件记录、基线有 | 本次列出的"消失"全部不可信，见样例 C |
| `base_unparseable` | L0-LOW | 基线副本无组件记录、当前有 | 本次列出的"新增"全部不可信，通常是基线留档环节坏了 |
| `input_no_component` | L0-LOW | 两侧都没有组件记录 | 比对不成立，不得读成"没有变化" |
| `surface_both_empty` | L1-INFO | 两侧可解析但导出面集合都为空 | 本次确无新增外部可达面（与 `input_no_component` 的区别是"有组件、只是没导出"） |
| `surface_added_count` | L1-INFO | 差集非空时的计数行 | 给出基线与当前的导出数，供 K3/K8 统计 |
| `surface_no_add` | L1-INFO | 差集为空 | 无新增导出组件；不排斥信号码层面的变化 |
| `surface_removed` | L1-INFO | 反向差集非空 | 确认是主动收紧还是重构遗漏，遗漏会让依赖它的外部集成静默失效 |
| `pair_identity_unverified` | L1-INFO | 每次固定输出 | 同源前提由操作者负责；跨包配对的后果见第二十章第四段样例 |

这张表也说明为什么脚本 2 的 `SUMMARY` 里 `high=` 与 `CODE_NEW` 行数不等：前者按**判定条目**计数（三个新增组件即三条），后者按**码位集合**去重。需要实例数量时回到 `ADDED` 明细行。

### 脚本全文

以下是本次实测使用的第二个脚本原文，逐字未改；本章与第二十章的样例输出全部由它直接产生。

```bash
#!/usr/bin/env bash
# 脚本 2: weakness-surface-diff.sh — 两次清单清点的离线差异比对（纯文本，无 adb、无网络、不写设备）
# 用法: weakness-surface-diff.sh <基线清点副本> <当前清点副本>
# 输入: 脚本 1 写出的副本文件；只使用其中的 COMP 与 ROW 两类行，其余行仅作元信息展示
set -u
export LC_ALL=C

base="${1:?用法: weakness-surface-diff.sh <基线清点副本> <当前清点副本>}"
curr="${2:?缺少当前清点副本路径}"
for f in "$base" "$curr"; do
  [ -r "$f" ] || { echo "ERR,input_unreadable,$f" >&2; exit 1; }
  [ -s "$f" ] || { echo "ERR,input_empty,$f" >&2; exit 1; }
done

hi=0; md=0; gap=0; info=0
row() {
  printf 'ROW,%s,%s,%s\n' "$1" "$2" "$3"
  case "$1" in
    L3-HIGH*) hi=$(( hi + 1 )) ;;
    L2-WARN*) md=$(( md + 1 )) ;;
    L0-LOW*)  gap=$(( gap + 1 )) ;;
    L1-INFO*) info=$(( info + 1 )) ;;
  esac
}
meta() { printf 'META,%s,%s\n' "$1" "$2"; }
cnt() { printf '%s\n' "$1" | grep -c '[^[:space:]]'; }

d="$(mktemp -d)"; trap 'rm -rf "$d"' EXIT INT TERM
tb="$d/base_exported";  tc="$d/curr_exported"
bc="$d/base_codes";     cc="$d/curr_codes"
bg="$d/base_gaps";      cg="$d/curr_gaps"

# 导出面集合：COMP 行中 exported=true 的 <类型/名称>
pick_exported() {
  awk -F, '/^COMP,/ { ex = "no"; for (i = 4; i <= NF; i++) if ($i == "exported=true") ex = "yes"
    if (ex == "yes") printf "%s/%s\n", $2, $3 }' "$1" | sort -u > "$2"
}
# L3/L2 信号码集合（用于回归判定）与 L0 缺口码集合（用于长期未闭合判定）
pick_codes() {
  awk -F, -v lv="$2" '$1 == "ROW" && $2 ~ lv { print $3 }' "$1" | sort -u
}

pick_exported "$base" "$tb"; pick_exported "$curr" "$tc"
pick_codes "$base" '^L[23]-' > "$bc";  pick_codes "$curr" '^L[23]-' > "$cc"
pick_codes "$base" '^L0-'    > "$bg";  pick_codes "$curr" '^L0-'    > "$cg"

nb="$(grep -c '[^[:space:]]' "$tb")"; nc="$(grep -c '[^[:space:]]' "$tc")"
meta baseline_exported "$nb"; meta current_exported "$nc"

bcomp="$(grep -c '^COMP,' "$base")"; ccomp="$(grep -c '^COMP,' "$curr")"
if [ "$bcomp" -eq 0 ] && [ "$ccomp" -eq 0 ]; then
  row L0-LOW input_no_component "两侧都没有可解析的组件记录：比对不成立，不得读成没有变化"
elif [ "$ccomp" -eq 0 ]; then
  row L0-LOW curr_unparseable "当前副本（$curr）没有组件记录，基线有 $bcomp 条：本次列出的“消失”应读作采集或解析失败，不读作暴露面收紧"
elif [ "$bcomp" -eq 0 ]; then
  row L0-LOW base_unparseable "基线副本（$base）没有组件记录，当前有 $ccomp 条：本次列出的“新增”缺少历史参照，只能读作首次清点"
elif [ "$nb" -eq 0 ] && [ "$nc" -eq 0 ]; then
  row L1-INFO surface_both_empty "两侧导出面均为空（基线 $bcomp 条、当前 $ccomp 条组件记录）：本次无新增外部可达面"
fi
row L1-INFO pair_identity_unverified "清点副本不携带包名与版本：需自行确认两侧来自同一包的同一条版本线，否则本比对读作跨包差分没有意义"

if [ -s "$tb" ] || [ -s "$tc" ]; then
  added="$(comm -13 "$tb" "$tc")"; removed="$(comm -23 "$tb" "$tc")"
  na="$(cnt "$added")"; nr="$(cnt "$removed")"
  if [ "$na" -gt 0 ]; then
    printf '%s\n' "$added" > "$d/add"
    while IFS= read -r c; do
      [ -n "$c" ] || continue
      printf 'ADDED,%s\n' "$c"
      case "$c" in
        provider/*)            row L3-HIGH surface_provider_added "$c：新增导出的数据面，同机跨应用可直接寻址，需立即核对读写权限与 URI 授权范围" ;;
        receiver/*|service/*)  row L3-HIGH surface_input_added "$c：新增外部可投递或可绑定的入口，调用者身份校验责任完全落在应用代码内" ;;
        *)                     row L2-WARN surface_activity_added "$c：新增外部可启动入口，核对深链参数校验、跳转目标与返回数据" ;;
      esac
    done < "$d/add"
    row L1-INFO surface_added_count "新增导出组件 $na 个（基线 $nb 个，当前 $nc 个）"
  else
    row L1-INFO surface_no_add "导出面未见新增组件（基线 $nb 个，当前 $nc 个）"
  fi
  if [ "$nr" -gt 0 ]; then
    printf '%s\n' "$removed" > "$d/del"
    while IFS= read -r c; do [ -n "$c" ] && printf 'REMOVED,%s\n' "$c"; done < "$d/del"
    row L1-INFO surface_removed "$nr 个组件退出导出面：确认是主动收紧还是重构遗漏，遗漏会使依赖它的外部集成静默失效"
  fi
fi

newcodes="$(comm -13 "$bc" "$cc")"; gonecodes="$(comm -23 "$bc" "$cc")"
nn="$(cnt "$newcodes")"; ng="$(cnt "$gonecodes")"
meta new_signal_codes "$nn"; meta gone_signal_codes "$ng"
[ "$nn" -gt 0 ] && printf '%s\n' "$newcodes" | while IFS= read -r c; do [ -n "$c" ] && printf 'CODE_NEW,%s\n' "$c"; done
[ "$ng" -gt 0 ] && printf '%s\n' "$gonecodes" | while IFS= read -r c; do [ -n "$c" ] && printf 'CODE_GONE,%s\n' "$c"; done

if [ "$nn" -gt 0 ]; then
  bad_new="$(printf '%s\n' "$newcodes" | grep -c -E '^(debuggable|provider_open|receiver_open_custom|exported_implicit|exported_absent)$')"
  [ "$bad_new" -gt 0 ] && row L3-HIGH high_signal_new "$bad_new 个高危或默认放宽类信号码为本次新增：按变更单定位引入它的提交，并核对是否已在灰度范围内"
  warn_new="$(printf '%s\n' "$newcodes" | grep -c -E '^(activity_open_deeplink|scheme_http|service_filter|service_exported|provider_grant|backup_unfiltered|cleartext_allowed|cleartext_default_on|sdk_below_31)$')"
  [ "$warn_new" -gt 0 ] && row L2-WARN warn_signal_new "$warn_new 个暴露面或策略缺失类信号码为本次新增：进入整改队列，并核对验收记录是否覆盖了这一项"
fi
if [ "$ng" -gt 0 ]; then
  row L2-WARN signal_gone "$ng 个 L3/L2 信号码在本次消失：先确认是整改生效还是采集口径变化导致证据缺失（例如目标版本解析失败会使一批默认值判定退化为缺口）"
fi

pgap="$(comm -12 "$bg" "$cg")"; np="$(cnt "$pgap")"
meta persistent_gap_codes "$np"
[ "$np" -gt 0 ] && row L0-LOW gap_persistent "$np 个缺口码在两次清点中均未闭合（$(printf '%s\n' "$pgap" | paste -sd' ' - 2>/dev/null || printf '%s' "$pgap")）：属于长期观察盲区，需要补采集口径而不是补结论"

meta baseline_summary "$(grep '^SUMMARY,' "$base" | tail -1)"
meta current_summary  "$(grep '^SUMMARY,' "$curr" | tail -1)"
printf 'SUMMARY,high=%s,warn=%s,gap=%s,info=%s,base_exported=%s,curr_exported=%s\n' \
  "$hi" "$md" "$gap" "$info" "$nb" "$nc"
```

### 样例 A：同一条版本线上的真实漂移（`demo-1.4.0` → `demo-1.5.0`）

这是本报告最想让你看到的一种输出：基线是第十八章样例 A 那个"干净包"，当前是同一应用在下一版里的形态。请特别留意 `SUMMARY` 与 `META` 的组合，它把一次发布里发生的四件事全部写清了：

```text
META,baseline_exported,2
META,current_exported,6
ROW,L1-INFO,pair_identity_unverified,清点副本不携带包名与版本：需自行确认两侧来自同一包的同一条版本线，否则本比对读作跨包差分没有意义
ADDED,activity/com.example.demo.InternalEditor
ROW,L2-WARN,surface_activity_added,activity/com.example.demo.InternalEditor：新增外部可启动入口，核对深链参数校验、跳转目标与返回数据
ADDED,provider/androidx.core.content.FileProvider
ROW,L3-HIGH,surface_provider_added,provider/androidx.core.content.FileProvider：新增导出的数据面，同机跨应用可直接寻址，需立即核对读写权限与 URI 授权范围
ADDED,provider/com.example.demo.ShareProvider
ROW,L3-HIGH,surface_provider_added,provider/com.example.demo.ShareProvider：新增导出的数据面，同机跨应用可直接寻址，需立即核对读写权限与 URI 授权范围
ADDED,service/com.example.demo.SyncService
ROW,L3-HIGH,surface_input_added,service/com.example.demo.SyncService：新增外部可投递或可绑定的入口，调用者身份校验责任完全落在应用代码内
ROW,L1-INFO,surface_added_count,新增导出组件 4 个（基线 2 个，当前 6 个）
META,new_signal_codes,4
META,gone_signal_codes,0
CODE_NEW,backup_unfiltered
CODE_NEW,provider_grant
CODE_NEW,provider_open
CODE_NEW,service_exported
ROW,L3-HIGH,high_signal_new,1 个高危或默认放宽类信号码为本次新增：按变更单定位引入它的提交，并核对是否已在灰度范围内
ROW,L2-WARN,warn_signal_new,3 个暴露面或策略缺失类信号码为本次新增：进入整改队列，并核对验收记录是否覆盖了这一项
META,persistent_gap_codes,1
ROW,L0-LOW,gap_persistent,1 个缺口码在两次清点中均未闭合（debuggable_absent）：属于长期观察盲区，需要补采集口径而不是补结论
META,baseline_summary,SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
META,current_summary,SUMMARY,high=2,warn=4,gap=1,info=7,components=6,exported=6
SUMMARY,high=4,warn=2,gap=1,info=2,base_exported=2,curr_exported=6
```


这份输出把一次真实回归的全部要素写在了一起。逐段读：

**变化的实质。** 三个原本 `exported="false"` 的组件（`InternalEditor`、`FileProvider`、`SyncService`）被翻成 `true`，另外新增了一个从未声明过访问权限的 `ShareProvider`，同时 `allowBackup` 从 `false` 翻成 `true` 且仍然没有任何排除规则。这是一次典型的"为了一个外部集成需求，同时改掉了三项通道决定"的变更——在清单层之外，这三项决定分别属于组件设计、数据面授权与备份策略三个不同的责任人。

**为什么 `high=4` 而 `CODE_NEW` 只有四条、`high_signal_new` 只有一条。** 差分脚本的 `high=` 统计的是**判定条目数**：三条来自新增导出面（两个 `provider` + 一个 `service`），一条来自"新增高危码位"的聚合。而 `CODE_NEW` 按**码位集合**去重，`provider_open` 在当前副本里出现了两次（两个不同 provider），这里只列一次。这个口径是有意的——同一次改动导致的同一类问题不应被重复计数；代价是**实例数量丢失**，需要看数量时必须回到 `ADDED` 明细行。这两套计数之间的关系，第二十章 K2 与 K8 的定义里各自写明。

**`META,new_signal_codes,4` 与 `META,gone_signal_codes,0` 的比值才是回归指标。** 一个健康发布线的常见形态是"新增少、消失多"；这份输出是"新增 4、消失 0"，配合导出面从 2 涨到 6，构成了 K8（差分回归发现数）应该触发评审的那一类数据点。

**`gap_persistent` 是唯一一条 L0，也是唯一一条"没人被通知"的结论。** `debuggable_absent` 在第十八章样例 A 里已经出现，这次仍在——这个包的构建流程始终没有显式书写该属性。它不指向任何一位组件负责人，只指向构建流水线；把它做成指标（K9 缺口闭环率）而不是告警，是它唯一可能被处理的方式。

### 样例 B：无变化时的输出形态（同一份副本自检，`demo-1.4.0` → `demo-1.4.0`）

第二段的用途是**口径自检**：把同一份副本同时作为基线与当前传入，检查脚本在"应当什么都不发生"时的输出。请注意它的边界——这**不是**"回归通过"的证据，真正的回归判定必须用不同版本的两份产物（样例 A 那种）。

```text
META,baseline_exported,2
META,current_exported,2
ROW,L1-INFO,pair_identity_unverified,清点副本不携带包名与版本：需自行确认两侧来自同一包的同一条版本线，否则本比对读作跨包差分没有意义
ROW,L1-INFO,surface_no_add,导出面未见新增组件（基线 2 个，当前 2 个）
META,new_signal_codes,0
META,gone_signal_codes,0
META,persistent_gap_codes,1
ROW,L0-LOW,gap_persistent,1 个缺口码在两次清点中均未闭合（debuggable_absent）：属于长期观察盲区，需要补采集口径而不是补结论
META,baseline_summary,SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
META,current_summary,SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
SUMMARY,high=0,warn=0,gap=1,info=2,base_exported=2,curr_exported=2
```


三点读法。

**"没有新增"必须与"没有变化"区分。** 本例里两条 `META` 摘要行逐字段相同（`components=5,exported=2` 两边一致），加上 `new_signal_codes=0`、`gone_signal_codes=0`，才共同支撑"这次没有差分"。只看 `SUMMARY,high=0` 是不够的：一个把 `provider_open` 改成 `service_exported` 的变更，同样能让 `high` 与 `warn` 的聚合计数看起来没有恶化。

**`gap=1` 在"无变化"的输出里依然出现。** 差分脚本不做任何采集，它只是把两份副本里都存在的 L0 码位挑出来。因此"这版没有回归"与"这版仍然带着同一个观察盲区"是同时成立的两句话——第二十章 K9 的定义就是为了让这句话有一个可追踪的数字，而不是每次都靠人记住。

**`pair_identity_unverified` 是唯一一条恒定输出的 L1。** 有读者会想把它降噪掉，本文建议不要：它是这份差分**唯一**的输入合法性声明。副本文件里没有包名与版本名，脚本无法自行确认两侧同源，因此把这件事一直写在结果里。真要降噪，正确做法是在自己的流水线里给副本文件命名时带上包名与版本（如本报告样例用的 `demo-1.4.0-copy.txt`），并在门禁脚本里校验文件名的版本单调性——那是流程层的职责，不是这个比对脚本的职责。

### 样例 C：当前副本采集失败（`demo-1.4.0` → `broken-2.0.0`）

第三段样例是这套流程里最需要在生产环境中被看见的一种输出。**当前副本来自一份不是清单展开文本的输入**（第十八章说明过：换用别的展开工具、拿错文件、或清单合并结果异常时都会走到这条路），脚本 1 对它产出的是一份"只有缺口、没有组件"的清单。现在把它当作"新版本"与旧基线比对：

```text
META,baseline_exported,2
META,current_exported,0
ROW,L0-LOW,curr_unparseable,当前副本（broken-2.0.0-copy.txt）没有组件记录，基线有 5 条：本次列出的“消失”应读作采集或解析失败，不读作暴露面收紧
ROW,L1-INFO,pair_identity_unverified,清点副本不携带包名与版本：需自行确认两侧来自同一包的同一条版本线，否则本比对读作跨包差分没有意义
ROW,L1-INFO,surface_no_add,导出面未见新增组件（基线 2 个，当前 0 个）
REMOVED,activity/com.example.demo.MainActivity
REMOVED,receiver/com.example.demo.boot.BootReceiver
ROW,L1-INFO,surface_removed,2 个组件退出导出面：确认是主动收紧还是重构遗漏，遗漏会使依赖它的外部集成静默失效
META,new_signal_codes,0
META,gone_signal_codes,0
META,persistent_gap_codes,1
ROW,L0-LOW,gap_persistent,1 个缺口码在两次清点中均未闭合（debuggable_absent）：属于长期观察盲区，需要补采集口径而不是补结论
META,baseline_summary,SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
META,current_summary,SUMMARY,high=0,warn=0,gap=7,info=0,components=0,exported=0
SUMMARY,high=0,warn=0,gap=2,info=3,base_exported=2,curr_exported=0
```


**这份输出看起来像"暴露面收紧了"，但它不是。** 三条关键行必须一起读：`curr_unparseable`（L0）说明当前副本一条组件记录都没有；`REMOVED` 两行与 `surface_removed`（L1）列出"退出导出面"的两个组件；`META,current_summary,…,gap=7,info=0,components=0,exported=0` 说明当前那份清单本身就只有缺口。**"当前没有"和"当前采不到"在集合差分里是同一个空集**，区别只能由这条 L0 声明出来。如果流水线只按 `SUMMARY,high=0` 判定放行，这次发布就会以"暴露面下降"的结论通过——这是本系列反复强调的那个失效模式的实证版本，也是纪律一（未采到 = 缺口，不是通过）在差分层的落点。

**`surface_no_add` 与 `REMOVED` 同时出现，是正确行为。** 基线有 2 个导出组件、当前有 0 个，所以"没有新增"是真的，"两个消失"也是真的——但后者**不携带任何原因**。脚本拒绝推断原因（收紧、重构遗漏、还是采集失败），只把三种可能性都留在文字里："确认是主动收紧还是重构遗漏"。把原因判断交回给人，是这类工具唯一稳妥的分工。

**`gap_persistent` 仍然出现。** `debuggable_absent` 这次依旧没闭合——即便当前副本整体采不到，这个缺口在两侧都存在。它的意义是：**采集失败不会掩盖旧缺口，也不会制造新缺口**，两个方向都不会骗人。

### 差分脚本的局限

1. **不识别重命名。** 组件改名在输出里是一条 `REMOVED` 加一条 `ADDED`，需要人工确认是否为同一入口。若要在流水线里自动配对，可以用 `COMP` 行的属性串做相似度比较，但那已经超出"确定性判定"的范围，本文不做。
2. **不追踪修复动作与责任人。** 差分只说"这个码位消失了"，不说"因为谁做了什么"。绑定变更单是流程层的事；K10（回归闭合时长）需要用你们自己的提交/发布记录把两次采集串起来。
3. **不理解语义等价。** `permission="none"` 变成 `readPermission="…"` 与 `permission="…"` 变成 `readPermission="…"`，在码位层面可能都不产生新增，但两者的实际含义差别很大（参考资料 14：`readPermission`/`writePermission`/`grantUriPermissions` 优先于 `permission`）。这类判断需要读原始 `COMP` 行，不能只看码位差分。
4. **不接受跨分包合并视图。** 每个 APK 各一份副本、各一次比对；AAB 的 split 变化会表现为大片 `ADDED`/`REMOVED`（见第二十章第四段样例的错位读数示范）。
5. **不判"两个版本之间还有别的版本"。** 差分是两点关系。若基线距今十个版本，`CODE_NEW` 会包含一堆早已上线的历史变更；正确用法是把基线设为"上一个已发布版本"，把"距上次安全评审的累计变化"交给指标 K8 的趋势线，而不是交给一次比对。

两个脚本合起来给出一条可以长期运行的判定线：**第十八章回答"这个包声明了什么"，第十九章回答"这次发布改了什么"**，两者都以 `L0-LOW` 承认自己不知道的部分。下一章把这两条输出接进检测与取证的研判顺序，并给出十项可以在组织里直接落地的指标口径。

---

## 二十、检测与取证：研判顺序、指标口径与一次错位配对的示范

### 从一条 `L3-HIGH` 开始的六步研判

**第一步，复读合取条件，不要读结论。** 每条 L3 的说明句都写了触发条件（第十一章组二）。`provider_open` 要求"已导出"与"三项权限属性皆缺"同时成立；如果实际清单里存在 `readPermission` 而脚本仍报 L3，那是解析问题，不是风险问题——先怀疑工具，再定性。

**第二步，回到 `COMP` 行看原始属性快照。** 差分与聚合都可能误导，唯一不骗人的是那一行 `COMP,…,exported=…,permission=…,readPermission=…,writePermission=…,grantUriPermissions=…,authorities=…,intentFilter=…`。它是本次采集的原始证据，可以直接贴进整改单。

**第三步，判定证据层归属。** 按第十五章的归属矩阵决定下一份证据从哪来：需要代码（Intent 处理、WebView 配置）、需要资源 XML（NSC、`dataExtractionRules`、`file_paths`）、需要服务端行为，还是需要设备侧实测。**这一步做完之前不要写风险等级**——第十七章那张表里"部分可见"的十四条，全部会在这里被分流。

**第四步，需要时升级到设备侧核验。** 清单层说"应用声明了什么"，设备侧说"这台机器上实际生效什么"（UID、SELinux 域与类别、可调试状态、数据目录权限）。这一层的只读快照方法与脚本在[应用沙箱与进程隔离篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)第十六节，本文不重复；两者的关系是：**清点数出一条待核验的清单，设备快照决定它是否已在生产环境生效。**

**第五步，做变更归因。** 第十九章的 `CODE_NEW`/`ADDED` 是唯一能把 L3 绑到一次发布上的证据。绑定之后，整改责任才从"应用安全团队"转回"引入它的团队"，K10 指标才有意义。

**第六步，登记与复测。** 把这次未闭合的 `L0-LOW` 逐条写成下一次采集的动作（补显式声明、换用签名产物展开、修副本命名）。K9 的定义就是这一层的完成度。

**被外部报告时的取证顺序略有不同：** 先固定物证（APK 摘要值、构建号、清单副本），再取最近 N 次发布的差分时间线，把导出面变化与外部事件时间对齐；**不要在缺少基线副本时制造"无变化"的结论**——第十九章样例 C 演示的正是这种结论如何被自动生成。跨设备/跨版本的可调试性与身份合并判定，属于沙箱篇与内核篇的取证范围，本文只负责清单侧的时间线。

### 十项可运营指标

每条给出分子、分母与"只报分子会导致什么误读"。所有分母都来自脚本输出或你们的发布记录，**任何一条指标都不需要新工具**。

| 指标 | 定义 | 误读风险 |
| --- | --- | --- |
| K1 清单清点覆盖率 | 有可复核清单副本、且 `META,target_sdk` 解析成功的发布包数 ÷ 同期发布包总数 | 覆盖率不到 100% 时，其余指标的分母都是虚构的 |
| K2 L3 存量 | `SUMMARY,high>0` 的包数；并按码位分列实例数（`provider_open` 等三码） | 只报"包数"会掩盖一个包里有多少个开放点 |
| K3 导出面规模 | 每包 `exported_components` 的中位数与 P90，按应用族分列 | 单独看没有意义，必须与 K2 同列（导出多 ≠ 风险高） |
| K4 无门槛数据面比例 | `provider_open` 实例数 ÷ 导出 `provider` 实例数 | 分母为零时该比例无定义，要显式记为"无数据面"而不是 0% |
| K5 默认漂移敞口 | `targetSdk < 31` 或 `sdk_missing` 的在架包占比（含 `cleartext_default_on`） | 这是唯一一条"随时间自动变差"的指标，需要月度趋势而非当期值 |
| K6 备份通道未过滤占比 | `backup_unfiltered` ÷ （`backup_*` 四码总数） | 缺口型 `backup_absent` 被算进分母时，改善可能只是"更多的人写了 false" |
| K7 明文与信任锚敞口 | `cleartext_allowed`+`cleartext_default_on` 的包数；`nsc_absent` 的包数 | 两项要分开报：NSC 缺失是证据缺失，明文允许是策略决定 |
| K8 差分回归发现数 | 每次发布差分的新增 L3/L2 码位数；报 P90 与"连续两次非零的包占比" | 只看单次值会奖励"大版本一次性释放历史欠账" |
| K9 缺口闭环率 | 下一采集周期内闭合的 L0 码位数 ÷ 上期 L0 码位数 | **必须排除"因采不到而消失"的伪闭合**（第十九章样例 C 那种） |
| K10 回归闭合时长 | 从 `CODE_NEW` 出现到该码位在差分中消失且被确认为整改的中位天数 | 未确认原因的消失不计入闭合，否则工具故障会让指标变好看 |

三条总原则：**任何指标都不得只报分子**（没有分母的"零高危"是治理无效的信号，不是治理有效）；**L0 与 L2/L3 不进同一个看板**（它们要求的是完全不同的动作）；**每个季度的阈值重定基**（第十八章末说过，先记录两周再阻断，阈值用自家分布定）。

### 第四段样例：配对错位时应该怎么读（`legacy-1.0.0` → `demo-1.4.0`）

最后一段样例是把**两条不同版本线**（两个不同包）的副本互相配对的结果：基线来自第十八章样例 B 那个宽松包，"当前"来自样例 A 那个干净包。这是第十九章局限第 1、4、5 条的实际后果，也是新流水线最常见的误用方式：

```text
META,baseline_exported,5
META,current_exported,2
ROW,L1-INFO,pair_identity_unverified,清点副本不携带包名与版本：需自行确认两侧来自同一包的同一条版本线，否则本比对读作跨包差分没有意义
ADDED,activity/com.example.demo.MainActivity
ROW,L2-WARN,surface_activity_added,activity/com.example.demo.MainActivity：新增外部可启动入口，核对深链参数校验、跳转目标与返回数据
ADDED,receiver/com.example.demo.boot.BootReceiver
ROW,L3-HIGH,surface_input_added,receiver/com.example.demo.boot.BootReceiver：新增外部可投递或可绑定的入口，调用者身份校验责任完全落在应用代码内
ROW,L1-INFO,surface_added_count,新增导出组件 2 个（基线 5 个，当前 2 个）
REMOVED,activity/com.example.bad.MainActivity
REMOVED,activity/com.example.bad.PayActivity
REMOVED,provider/com.example.bad.db.RowProvider
REMOVED,receiver/com.example.bad.CommandReceiver
REMOVED,service/com.example.bad.ExportService
ROW,L1-INFO,surface_removed,5 个组件退出导出面：确认是主动收紧还是重构遗漏，遗漏会使依赖它的外部集成静默失效
META,new_signal_codes,0
META,gone_signal_codes,10
CODE_GONE,activity_open_deeplink
CODE_GONE,backup_unfiltered
CODE_GONE,cleartext_allowed
CODE_GONE,debuggable
CODE_GONE,provider_grant
CODE_GONE,provider_open
CODE_GONE,receiver_open_custom
CODE_GONE,scheme_http
CODE_GONE,sdk_below_31
CODE_GONE,service_filter
ROW,L2-WARN,signal_gone,10 个 L3/L2 信号码在本次消失：先确认是整改生效还是采集口径变化导致证据缺失（例如目标版本解析失败会使一批默认值判定退化为缺口）
META,persistent_gap_codes,0
META,baseline_summary,SUMMARY,high=3,warn=7,gap=1,info=3,components=5,exported=5
META,current_summary,SUMMARY,high=0,warn=0,gap=1,info=10,components=5,exported=2
SUMMARY,high=1,warn=2,gap=0,info=3,base_exported=5,curr_exported=2
```


这份输出里每一条单看都"合理"，合起来却完全错。它值得逐行对照：

**`CODE_GONE` 十条看起来像十项整改完成。** 实际上没有做过任何整改：`provider_open`、`debuggable`、`cleartext_allowed` 这些码位之所以消失，是因为"当前"那份副本来自另一个本来就没有这些声明的包。**`signal_gone` 被定成 `L2-WARN` 而不是 `L1-INFO`，正是为了让这种消失必须被追问一次。**

**`ADDED` 两行看起来像两个新增入口。** 也不是：`demo` 的启动器 `activity` 与开机接收器一直是导出的，差分只是因为两侧组件名不同而把它们当"新增"。跨包配对必然同时制造假的"新增"与假的"消失"，这就是第十九章局限第 1 条的最坏形态。

**这份错位配对让最干净的包收到了一条 `L3-HIGH`。** `SUMMARY,high=1` 来自 `surface_input_added`（新增导出的接收器）——而被判定的那个接收器，在样例 A 里是带着 `android:permission` 的 `L1-INFO`。**任何按 `high>0` 阻断的门禁，都会被一次参数写反的配对阻断一个正确版本。** 这不是脚本的缺陷（它无法知道两侧来自不同包），而是流程必须补上的一环。

**`persistent_gap_codes=0` 是另一个陷阱。** 基线侧的 L0 是 `nsc_absent`，当前侧是 `debuggable_absent`，两者没有交集，于是"缺口闭合了"。如果把它计入 K9 的分子，缺口闭环率会因为换了一个包而变好——**这就是 K9 定义里那句"必须排除因采不到而消失的伪闭合"的具体来源。**

**因此配对前置校验必须放在流水线里，不是放在脚本里。** 三个做法按成本递增：给副本按"包名+版本名"命名并在门禁里校验两侧包名一致与版本单调（本文样例采用这种）；为每次采集额外写一份 sidecar 元信息文件（包名、版本名、构建号、APK 摘要值），比对前先读它；把差分调用改成由构建系统直接发起，让它天然只会在同一条流水线里取到相邻两个产物。**本文的脚本刻意不做这件事**：副本是纯文本、不读 APK，因此不携带任何包身份，多一个隐式假设不如少一个。

---

## 二十一、十二条常见误区

1. **把 MASVS 的两档读成两级严重程度。** 标准写的是适用范围：L1 是所有应用都建议满足的通用控制，L2 是处理高敏感数据时应加强的控制（参考资料 1）。`L2` 不是"高危"，"这条要求适不适用于本包"才是它的问题。
2. **把 Mobile Top 10 当覆盖清单。** 榜单口径自我限定得很清楚："the risks to mobile applications do not stop at the Top 10"（参考资料 6）。按榜单验收等于宣布榜外条目不需要处理。
3. **把 CWE 移动视图读成"Android 专属缺陷表"。** 视图自述是"CWE entries in this view (slice) are often seen in mobile applications."（参考资料 7）；二十一个成员里只有四条以 Android 应用为定义主语（925、926、927、939）。把 CWE-312、940 之类通用条目写成"Android 漏洞类型"会引发平台间误用。
4. **把 `high=0` 读成可以发布。** 第十九章样例 C 与第二十章第四段样例各演示了一种：前者是采集失败制造了"消失"，后者是配对错位制造了"新增 L3"。`gap=` 与 `pair_identity_unverified` 必须一起看。
5. **把属性缺省读成已经证明。** `android:debuggable` 没出现在清单里，不等于"本包不是可调试构建"；平台默认值参与判定方向，但不代替本包证据（第十四章纪律三）。同理适用于 `allowBackup`——它的默认值恰恰是 `true`（参考资料 15）。
6. **把 `L1-INFO` 当噪声删掉。** `self_only`、`backup_off`、`sdk_31_plus` 这些条目是差分基线；删掉它们，第十九章就没有可比对的 `COMP`/`ROW` 集合，"收紧"将无法被证明。
7. **复用别人的扫描结论。** 同一份声明在不同 `targetSdkVersion` 下含义不同（第十七章整表），不带目标版本上下文的"高危/低危"标签不可跨包移植，更不可跨平台移植。
8. **只改清单，或只改代码。** `exported="false"` 能关掉可达面，但会破坏真实需要它的外部集成；只在代码里加来源校验而保留无条件导出，则声明形态仍然开放。第十五章组合一说明这两半属于不同的责任人，整改单要同时列。
9. **把"清单里没有导出组件"读成"没有跨应用数据面"。** 运行时注册的接收器、其它 split 里的组件、依赖库引入的导出声明都不在这份清单的这一格里；样例 C 里那个只写了 `authorities` 的 `provider` 就是"现在不开放、一次重构即开放"的形态。
10. **用韧性控制折抵安全控制。** MASVS-R 的软件保护控制"can ultimately be bypassed and must never be used as a replacement for security controls."（参考资料 1）。加固改变的是攻击者投入，不改变声明是否开放。
11. **把 `allowBackup="false"` 读成所有迁移通道关闭。** 官方在同一属性里写明：目标 Android 12+ 时，"On devices from some device manufacturers, you can't disable device-to-device migration of your app's files."（参考资料 15）。云备份可控，设备间迁移在不同厂商上不可控——脚本说明句因此保留"部分厂商存在差异"这句。
12. **把差分基线设得越老越好。** 差分是两点关系，基线距今十个版本时 `CODE_NEW` 会混入早已上线的历史欠账，看起来"信息量很大"，实际不可行动。正确做法是基线取上一个已发布版本，累计趋势交给 K8 的时间序列（第十九章局限第 5 条）。

---

## 二十二、趋势判断、分角色结论与常见问题

### 五点趋势判断

**一、要求口径会继续迁移，映射要靠条目自带的双编号而不是标题。** v2 系列已经把 `V1`–`V8` 换成域名标识（`STORAGE`/`CRYPTO`/`AUTH`/`NETWORK`/`PLATFORM`/`CODE`/`RESILIENCE`/`PRIVACY`，参考资料 2），而测试文档同时保留 `masvs_v1_id` 与 `masvs_v2_id` 两个前言字段（参考资料 5）。**流程含义**：任何把检查结果绑定到条目编号的门禁规则，都会在下一次口径迁移时失效；绑定到"证据层 + 信号码"的规则可以原样保留。本文的映射表（第一、十三、十六章）因此只到域级。

**二、属性级开关继续下沉到配置文件，清单层可见性会缩小。** 官方对 `usesCleartextTraffic` 的当前说明是它正在弃用，"will be ignored for apps targeting API levels 38 and above"，并且存在 Network Security Config 时该标志在 API 24+ 就被忽略（参考资料 15）。**流程含义**：只读清单的门禁会逐步失去对明文与信任锚的判定能力，第十八章局限第 1 条（不读资源 XML）将从"可接受的简化"变成"实际盲点"。建议在 K1 口径里把"是否留存 NSC 资源展开文本"纳入采集要求。

**三、平台用安装期强制代替默认值提醒，隐式默认类弱点会自然消亡但存量长期残留。** 目标 31+ 时未显式声明 `exported` 会导致无法安装（参考资料 13），这类约束比文档提醒有效得多。**流程含义**：`exported_implicit` 这一码位在未来新包上会趋零，但在低目标版本存量包上会长期存在——K5（默认漂移敞口）是这段过渡期唯一有意义的指标。

**四、隐私从存储域里独立出来，"内容级"证据需求上升。** v2 出现独立的 `PRIVACY` 域标识（参考资料 2）。**流程含义**：清单层能给的只有通道判定（第十二章第四类），隐私侧的结论必须由数据分级与采集范围证据支撑；把 `backup_off` 写成"隐私合规"是错误的口径。

**五、韧性检测与分发侧判定继续留在远程服务，不进入本地要求。** 标准明确 MASVS-R 不可折抵安全控制（参考资料 1），平台侧的设备完整性判定也已经改为需要与远程服务交互（"The SafetyNet Attestation API is deprecated and has been replaced by the Play Integrity API."，参考资料 26），分发侧则用另一个口径描述风险应用（"Potentially Harmful Applications (PHAs) are apps that could put users, user data, or devices at risk."，参考资料 25）。**流程含义**：这三类结论都不在你们自己的构建产物里，任何把它们写进静态门禁的做法都需要外部依赖，因而不可复现。本文因此完全不给它们出码位。

### 分角色结论

**移动应用安全负责人。** 你需要的不是一份"符合 MASVS"的声明，而是一张能把每条未满足项映射到证据层的矩阵：第十五章那张表可以直接改名后用。选型建议明确一条：MASVS 族做验收骨架、Top 10 做优先级沟通、CWE 做与公告对齐，三者不互换（第二章）。

**移动端架构与组件设计者。** 你负责的那一格是 W1 与 W5——入口是否必要、调用者身份如何确定。第十七章那张表值得贴在评审清单第一页：`android:exported` 与 `intent-filter` 的组合是**安装期**决定，不是运行期提醒，把它交给 Lint 而不是交给文档。

**发布工程与 CI。** 两个脚本可以原样接入，但阈值必须自己定：先跑两周只记录，再用自家分布定阻断线（第十八章末）。三条硬要求：清点数与副本随构建产物入库（K1 的分母）；差分只在同一构建流水线内相邻两个产物之间运行（第二十章第四段样例的教训）；`gap=` 非零时结果标注为"未通过合规判定"而不是"检查完成"。

**检测与取证工程师。** 本文的清单差分能给你一条"声明形态变化时间线"，但生效结论要交给设备侧：身份合并、域与类别、可调试执行域在[沙箱与进程隔离篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-App-Sandbox-and-Process-Isolation-Defense-Analysis.html)，越权与权限判定在[Framework 信任边界篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)。没有基线副本时不要产出"无变化"结论。

**合规与隐私。** 请只引用带上下文的结论：同一属性在不同 `targetSdkVersion` 上是两类弱点（第十七章），`allowBackup="false"` 不等于设备间迁移通道关闭（参考资料 15 的厂商差异说明）。第十四章的四条纪律可以作为你们的证据充分性检查表。

**风控与第三方应用引入。** 静态清单是引入评估里成本最低的一层，也是覆盖面最窄的一层。第十二章那四类"看不见"里，W8（供应链）与 W5（信任判定）才是引入决策的主要风险源，需要用依赖清单与授权测试补齐；平台侧的补丁到达性另见[补丁传导链篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Patch-Delivery-Chain-and-OEM-Update-Governance.html)。

### 常见问题

**Q：能不能只按 Mobile Top 10 建一张检查表？** A：能建，但不能验收。榜单的自我限定是"the risks to mobile applications do not stop at the Top 10"（参考资料 6）；用它排优先级可以，用它定义"覆盖完成"会系统性漏掉不在榜上的要求域。

**Q：MASVS 的 L2 是不是比 L1 更严重的漏洞等级？** A：不是。两档是适用范围（所有应用建议满足 / 处理高敏感数据应加强），不是严重程度（参考资料 1、第三章）。

**Q：脚本能不能直接判定"符合 MASVS-PLATFORM-1"？** A：不能，本文刻意不做这个映射（第十四章末）。清单事实与要求满足之间还差代码、资源与服务端三层证据。

**Q：`gap=5` 的包能不能上线？** A：脚本不会替你回答。诚实的口径是：**这次采集不足以支撑合规结论**；要么补显式声明后重跑，要么按参考资料 1 的风险归属规则由业务负责人签字接受缺口。

**Q：能不能拿别人的扫描结果当自己的结论？** A：不能。同一条声明在不同目标版本上分派到不同码位（第十七章整表），不带目标版本上下文的标签不可移植。

**Q：为什么脚本 2 每次都要输出一条 `pair_identity_unverified`？** A：因为清点副本不携带包名与版本，脚本无法自行确认两侧同源；跨包配对会同时制造假的"新增 L3"和假的"整改完成"（第二十章第四段样例）。降噪的正确做法是在流水线里校验命名与版本单调性。

**Q：`aapt2` 输出的编码变了怎么办？** A：判定会退化成一排 `L0-LOW`——这是设计行为。脚本对"无法解析"一律记缺口，绝不套用默认值继续判定（第十四章纪律一、纪律三）。

**Q：这些脚本能不能用于别人的 APK？** A：静态清点的法律与流程性质确实与动态测试不同，但本文的授权前提不变：**只对自有或已获书面授权处理的产物执行**（第五章末尾与范围声明）。OWASP 对测试文档的同一提醒放在这里更合适："Remember, attacking a system without written authorization is illegal in many parts of the world!"（参考资料 5）。

**Q：为什么不给修复代码或配置片段？** A：本系列的定位是防御性分析与检测口径，配置改写属于你们的设计决定；第十四章纪律二规定脚本只声明"清单里有什么"，不声称"应该怎么改才能避免被利用"——后者一旦写进工具输出，就会变成不可复核的承诺。

**Q：为什么不覆盖 iOS？** A：口径可以复用（MASVS 族与 CWE 移动视图都是跨平台的），但默认值、元素名与强制收紧方式全部不同。第十七章那张漂移表是 Android 专属的；iOS 侧对应内容在本系列的 iOS 报告集里。

---

## 二十三、参考资料

以下条目为本次核验实际取得逐字内容的公开来源，编号供正文与脚本引用；引文为原文（英文）摘录，破折号后为中文口径说明。截断处以"（原文截断）"标注；未取得整句的条目明确写出"本次不据此立论"。

1. OWASP MASVS v1.5.0（使用章节）："The objective of the MASVS is to establish a level of confidence in the security of mobile apps by providing a set of controls that address the most common mobile application security issues."、"The MASVS defines two security verification levels (MASVS-L1 and MASVS-L2), as well as a set of reverse engineering resiliency requirements (MASVS-R)."、"MASVS-L1 contains generic security requirements that are recommended for all mobile apps, while MASVS-L2 should be applied to apps handling highly sensitive data."、"Requirement: MASVS-Vx.y, e.g. MASVS-V2.2: \"No sensitive data is written to application logs.\""、"…implementing a control or not should ultimately be a risk-based decision, which is taken/communicated with the business owners."、"…the software protection controls listed in MASVS-R… can ultimately be bypassed and must never be used as a replacement for security controls."（原文截断） — 标准目标是"建立置信度"而非担保安全；档位只有 L1/L2 两档另加一类逆向韧性要求；要求编号形态是 `MASVS-Vx.y` 且指向一条可观察事实；做不做某控制是与业务负责人共同承担的风险决定；韧性控制不可折抵安全控制。https://raw.githubusercontent.com/OWASP/owasp-masvs/v1.5.0/Document/0x03-Using_the_MASVS.md
2. OWASP MASVS 文档目录与版本记录：v1 分区标题逐字为"V1: Architecture, Design and Threat Modeling Requirements"、"V2: Data Storage and Privacy Requirements"、"V3: Cryptography Requirements"、"V4: Authentication and Session Management Requirements"、"V5: Network Communication Requirements"、"V6: Platform Interaction Requirements"、"V7: Code Quality and Build Setting Requirements"、"V8: Resilience Requirements"；v2 使用域名标识 `MASVS-STORAGE`/`CRYPTO`/`AUTH`/`NETWORK`/`PLATFORM`/`CODE`/`RESILIENCE`/`PRIVACY`；OWASP 仓库发布记录将 v2.1.0 发布时间标为 2024-01-18 — 第四章表格的直接来源；八个 v2 域的官方一句话定义本次未全部逐字取得（见本章末清单）。https://github.com/OWASP/owasp-masvs
3. OWASP MASWE 项目页："The OWASP Mobile Application Security Weakness Enumeration (MASWE) is a list of common security and privacy weaknesses in mobile applications. It acts as the bridge between the MASVS and the MASTG." — 弱点枚举的结构位置：在要求与测试之间。https://mas.owasp.org/MASWE/
4. OWASP MASWE 仓库说明："The MASWE is intended to be a complementary list to the CWE, focusing specifically on security weaknesses in mobile applications." — 与 CWE 是互补关系而非替代；条目编号与总数不在本文复述范围。https://github.com/OWASP/maswe
5. OWASP MASTG："The OWASP Mobile Application Security Testing Guide (MASTG) is a comprehensive manual for mobile app security testing and reverse engineering."；测试文档 YAML 前言同时保留 `masvs_v1_id`（如 `MSTG-STORAGE-1`、`MSTG-STORAGE-2`）与 `masvs_v2_id`（如 `MASVS-STORAGE-1`）；"Remember, attacking a system without written authorization is illegal in many parts of the world!" — 双编号是口径迁移的唯一可靠映射手段；授权前提是所有测试文档自带边界。https://github.com/OWASP/owasp-mastg
6. OWASP Mobile Top 10（2024）："The Mobile Top 10 identifies and lists the top ten vulnerabilities found in mobile applications."、"It should be noted that the risks to mobile applications do not stop at the Top 10"；十条标题为 M1 Improper Credential Usage、M2 Inadequate Supply Chain Security、M3 Insecure Authentication/Authorization、M4 Insufficient Input/Output Validation、M5 Insecure Communication、M6 Inadequate Privacy Controls、M7 Insufficient Binary Protections、M8 Security Misconfiguration、M9 Insecure Data Storage、M10 Insufficient Cryptography — 榜单口径 + 官方自我限定；Top 10 与 MASVS/CWE 之间不存在官方一一映射表。https://owasp.org/www-project-mobile-top-10/
7. CWE-919 移动视图（cwe.mitre.org）："CWE entries in this view (slice) are often seen in mobile applications."；视图成员共 21 条（200、250、295、297、312、319、359、362、511、602、672、772、798、920、921、925、926、927、939、940、941，标题见第八章） — 视图准入语义是"常见于移动应用"，不是"专属于移动平台"。https://cwe.mitre.org/data/definitions/919.html
8. CWE-925："The Android application uses a Broadcast Receiver that receives an Intent but does not properly verify that the Intent came from an authorized source." — 接收器不校验 Intent 来源。https://cwe.mitre.org/data/definitions/925.html
9. CWE-926："The Android application exports a component for use by other applications, but does not properly restrict which applications can launch the component or access the data it contains." — 组件导出且未正确限制可启动/可访问范围；本文 L3 判据的语义来源。https://cwe.mitre.org/data/definitions/926.html
10. CWE-927："The Android application uses an implicit intent for transmitting sensitive data to other applications." — 用隐式 Intent 传递敏感数据。https://cwe.mitre.org/data/definitions/927.html
11. CWE-940（Improper Verification of Source of a Communication Channel）：本页存在，本次未取得可逐字引用的定义整句，正文只使用其标题与"通用软件措辞"定性，不据此立论。 — https://cwe.mitre.org/data/definitions/940.html
12. `<activity>` 元素的 `android:exported`："If \"true\", the activity is accessible to any app, and is launchable by its exact class name." — 取 `true` 即进入同机任意应用可达面。https://developer.android.com/guide/topics/manifest/activity-element
13. Android 12（API 31）行为变更：**Warning** 原文"If an activity, service, or broadcast receiver uses intent filters and doesn't have an explicitly-declared value for `android:exported`, your app can't be installed on a device that runs Android 12 or higher."；"If your app targets Android 12 or higher and contains activities, services, or broadcast receivers that use intent filters, you must explicitly declare the `android:exported` attribute for these app components." — 强制发生在安装期；本文 `exported_absent` 与 `sdk_31_plus` 的判据来源。https://developer.android.com/about/versions/12/behavior-changes-12
14. `<provider>` 元素：`android:exported` 的 `"true"` 释义为"the provider is available to other applications. Any application can use the provider's content URI to access it, subject to the permissions specified for the provider."；`android:grantUriPermissions` 释义为"Whether those who ordinarily don't have permission to access the content provider's data can be granted permission to do so, temporarily overcoming the restriction imposed by the readPermission, writePermission, permission, and exported attributes."，其"The default value is \"false\"."；`android:permission` 释义为"The name of a permission that clients must have to read or write the content provider's data… However, the readPermission, writePermission, and grantUriPermissions attributes take precedence over this one." — 权限三兄弟与 exported 之间的优先级，是 `provider_open`/`provider_grant` 两条码位的语义依据。https://developer.android.com/guide/topics/manifest/provider-element
15. `<application>` 元素：`android:allowBackup`——"Whether to let the application participate in the backup and restore infrastructure. If this attribute is set to \"false\", no backup or restore of the application is ever performed…"、"The default value of this attribute is \"true\"."、Note："On devices from some device manufacturers, you can't disable device-to-device migration of your app's files."；`android:debuggable`——"Whether the application can be debugged, even when running on a device in user mode. It's \"true\" if it can and \"false\" if not. The default value is \"false\"."；`android:usesCleartextTraffic`——"The default value for apps that target API level 27 or lower is \"true\". Apps that target API level 28 or higher default to \"false\"."、Note："This attribute is getting deprecated and will be ignored for apps targeting API levels 38 and above."、"This flag is ignored on Android 7.0 (API level 24) and above if an Android Network Security Config is present."；`android:dataExtractionRules`——"Applications can set this attribute to an XML resource where they specify the rules determining which files and directories can be copied from the device as part of backup or transfer operations." — 备份/调试/明文三项的默认值与相互关系全部来自本页；第十七章漂移表的主依据。https://developer.android.com/guide/topics/manifest/application-element
16. 网络安全配置（Network Security Config）："Starting with Android 9 (API level 28), cleartext support is disabled by default."；平台侧另有"On Android 16 (API level 36), certificate transparency is disabled by default. Apps can opt in to the feature either globally or on a per-domain basis." — 明文默认在 API 28 起关闭；证书透明度在 API 36 起默认关闭、可按域选择开启。https://developer.android.com/privacy-and-security/security-config
17. Intent 与 intent filter 文档：**Caution**"…always use an explicit intent when starting a Service and do not declare intent filters for your services…"（原文截断）；**Caution**"Using an intent filter isn't a secure way to prevent other apps from starting your components…"（原文截断）；另："Beginning with Android 5.0 (API level 21), the system throws an exception if you call `bindService()` with an implicit intent." — 服务不应声明过滤器；声明过滤器不是安全手段。https://developer.android.com/guide/components/intents-filters
18. `PendingIntent` 可变性："If your app targets Android 12, you must specify the mutability of each PendingIntent object…"（原文截断） — 目标 31+ 的强制要求，属代码层证据，清单不可见。https://developer.android.com/about/versions/12/behavior-changes-12#pending-intents
19. 应用专属存储与文件级加密："No other apps can access app-specific files in internal storage."；"Encrypting at the file level, instead of encrypting the entire storage area as a single unit, better isolates and protects individual users…"（原文截断） — 平台机制承诺，不是本包证据（第十二章末的方法提醒）。https://developer.android.com/training/data-storage ；https://source.android.com/docs/security/features/encryption/file-based
20. 密钥库的不可导出形态："…with the key material remaining non-exportable."（原文截断） — 可导出性是密钥类弱点的关键判定，但需代码与密钥参数证据。https://developer.android.com/privacy-and-security/keystore
21. `WebSettings.setAllowFileAccess()`：本页记录该设置的默认值随目标版本变化（API 29 及以下为 `true`，API 30 及以上为 `false`）；本次未取得整句，正文按此口径转述并标注为代码层不可见项，不据此给出包级结论。 — https://developer.android.com/reference/android/webkit/WebSettings#setAllowFileAccess(boolean)
22. WebView 接口暴露："Using `addJavascriptInterface` lets JavaScript control your Android app…"（原文截断）；"The addJavascriptInterface method exposes a Java object to all of a WebView's frames, and any frame can access the object name and call methods on it." — 桥接接口对全部 frame 可见，因此判定必须逐处读代码。https://developer.android.com/privacy-and-security/risks/insecure-bridge-objects ；https://developer.android.com/reference/android/webkit/WebView#addJavascriptInterface(java.lang.Object,%20java.lang.String)
23. APK 签名方案："During validation, v2+ scheme treats the APK file as a blob and performs signature checking across the entire file." — 完整性校验范围；与"清单说了什么"无关，属另一层证据。https://source.android.com/docs/security/features/apksigning
24. 权限保护级别与安全实践：`signature` 级的公开口径为"The permission is granted only if the application receiving the permission is signed by the same certificate as the application that declared the permission."（原文截断）；安全实践另说明仅同签名应用之间的 IPC 才受该级别约束。 — `protectionLevel` 是位掩码，脚本只输出原始值、不翻译级别名（第十七章末）。https://developer.android.com/guide/topics/permissions/overview ；https://developer.android.com/topic/security/tips
25. Google Play 保护机制对风险应用的定义："Potentially Harmful Applications (PHAs) are apps that could put users, user data, or devices at risk." — 分发侧口径与要求侧口径是两套语言，不可互相替代。https://developers.google.com/android/play-protect/potentially-harmful-applications
26. Play Integrity 与 SafetyNet 的状态："The SafetyNet Attestation API is deprecated and has been replaced by the Play Integrity API." — 设备完整性判定位于远程服务，不可在本地构建产物上断言（第二十二章趋势五）。https://developer.android.com/google/play/integrity
27. OWASP 移动应用安全项目主页（三件套关系）：项目页对 MASWE 的描述与参考资料 3 同句："The OWASP Mobile Application Security Weakness Enumeration (MASWE) is a list of common security and privacy weaknesses in mobile applications." — 用于确证 MASVS/MASTG/MASWE 属同一项目族。https://owasp.github.io/www-project-mobile-app-security/
28. `aapt2 dump xmltree`（Android SDK Build-Tools）：本次不引用整句；本文只把该子命令作为**输入形态来源**（打印编译后资源的 XML 树），并依赖实测输出形态解析；输出格式变化时判读退化为 `L0-LOW` 缺口。 — https://developer.android.com/tools/packages/aapt2

**未公开文档化 / 本次未逐字核验的边界（明确列出，正文与脚本不依赖以下任何一项）**

- **MASVS v2 八个域各自的官方一句话定义**：八个域标识可确证（参考资料 2），逐条描述句未全部取得；第四章因此只做编号与标识映射，不复述定义。
- **MASVS v2 的档位表述**：v1.5.0 的"两档 + R"有整句（参考资料 1），v2 是否沿用同一档位口径未取得整句；第三、四章不把 v2 的档位当作既定事实使用。
- **MASWE 的条目编号格式、条目总数与逐条定义句**：本次只取项目自述两句（参考资料 3、4）；第六章明确不复述条目清单。第三方对条目数量的转述（例如"78 条"）仅有非官方来源，正文不采用。
- **MASVS / MASWE / Top 10 / CWE 之间的官方一一映射表**：不存在可引用的公开映射表；第七章、第十三章、第十六章的所有跨口径对应均标注为本文分析。
- **MASTG 的具体测试编号目录与操作步骤**：不属本文范围，本文不复制任何针对运行时或工具链的操作流程。
- **`android:exported` 在 `activity`/`service`/`receiver` 上"有过滤器即默认导出"的整句默认值描述**：参考资料 12 给出可达性语义、13 给出安装期强制要求、14 给出 `provider` 的默认值句；其余元素的默认值整句本次未逐字取得。脚本因此**不依据缺省判"已导出"**，只出 `exported_absent`/`exported_implicit`/`exported_absent_no_filter` 三种"待核对"形态（第十四章纪律三）。
- **`PendingIntent` 可变性常量（`FLAG_IMMUTABLE`/`FLAG_MUTABLE`）的逐字定义**：参考资料 18 只取到要求句；第十二章不把常量语义当作已知前提。
- **`EncryptedSharedPreferences` 的机制整句、`FileProvider` 示例整句、`setJavaScriptEnabled` 默认值整句、APK Signature Scheme v3/v4 的机制整句、`verifyOnlyOverAppData` 与 `MODE_WORLD_READABLE` 的现代措辞**：本次未取得可逐字引用的整句，正文一律不出结论、不写建议。
- **Google Play「数据安全」申报的开发者义务整句**：本次只检索到二手表述，正文与指标不引用；隐私侧只使用清单通道证据。
- **Play Protect 拦截率、年度安全报告里的风险应用比例等数字**：未从官方页面取得，第二十二章趋势五只做口径说明，不给数字。
- **安全公告的漏洞类型图例（RCE/EoP/ID/DoS）**：本系列已在[远程代码执行篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Remote-Code-Execution-Attack-Surface-and-Defense-Analysis.html)与[Framework 信任边界篇](https://android-security-reports-0e8t10askpz.qoder.website/Android-Framework-Trust-Boundary-and-Vulnerability-Analysis.html)处理，本文不重复立论。
- **`aapt2 xmltree` 输出格式规范**：无逐字公开规范文档，本文按实测形态解析并假定其稳定；第十八章末与第二十章样例 C 说明了格式变化时的退化行为。
- **清单合并的来源归因**（哪一条声明由哪个依赖库引入）：不在清单展开文本里，本文不提供判定方法（第十八章局限第 4 条）。
- **具体 CVE、漏洞与利用细节**：不属本文范围；相关口径见[内核漏洞与提权防御](https://android-security-reports-0e8t10askpz.qoder.website/Android-Kernel-Vulnerability-and-Privilege-Escalation-Defense.html)与[攻击面与漏洞研究方法论](https://android-security-reports-0e8t10askpz.qoder.website/Android-Attack-Surface-and-Vulnerability-Research-Methodology.html)两篇。

---

咨询ios系统请咨询 telegram：https://t.me/one00190
