# Android 报告站搜索收录关键词方案

站点：`https://android-security-reports-0e8t10askpz.qoder.website/`
生成日期：2026-10-25 · 适用页面：8 页（首页 + 7 篇报告）
性质：SEO 运营方案，**不作为报告页发布**（放在 `Publish-Android/` 会被 `tools/build-site.mjs` 渲染成第 9 个页面）。

---

## 1. 现状诊断（实测，非推测）

| 项目 | 现状 | 判定 |
| --- | --- | --- |
| `<title>` | 每页由 front-matter `title` 输出，全部存在 | ✅ 可用 |
| `<meta name="description">` | 每页存在，来自 front-matter `description` | ✅ 可用 |
| `<meta name="robots" content="index,follow">` | 全页存在 | ✅ |
| `rel=canonical` | 全页存在，指向 `.html` 绝对地址 | ✅ |
| `og:type/title/description/locale/url` | 全页存在 | ✅ |
| `article:modified_time` | 由 git 提交日期推导，全页为 2026-10-25 | ✅ |
| JSON-LD | 输出 `WebPage` + `Organization` | ⚠️ 缺 `Article`/`TechArticle` 与 `BreadcrumbList` |
| `<meta name="keywords">` | **不存在**（builder 未输出） | ⚠️ 见 §3：Google 明确忽略此标签，补它不带来排名 |
| `sitemap.xml` | 8 条 `loc` + `lastmod`，可访问 | ✅ |
| `robots.txt`（dist 生成版） | 指向本站 `sitemap.xml` | ✅ |
| `Publish-Android/robots.txt`（Git 源文件） | Sitemap 仍指向 `https://prestonh965118.github.io/Android/sitemap.xml` | ❌ 该 Pages 站点未开通，指向不存在的主机，需改 |
| `.md` 原文可匿名访问 | `Content-Type: application/octet-stream`（实测） | ✅ 不会被当网页索引，无重复内容风险 |
| `twitter:card` | 缺失 | ⚠️ 影响转发展示，不影响收录 |
| 页面间内链 | 首页表格 + 各页「视角声明」里的同级链接 | ✅ 有爬取路径 |
| 收录入口 | Google Search Console / Bing Webmaster 均未验证 | ❌ 阻塞项 |

结论：技术侧基础项已具备，**当前的瓶颈不是关键词标签，而是（a）站点未在任何搜索控制台完成所有权验证，（b）标题与描述偏"报告内部术语"，没有覆盖真实检索说法。**

---

## 2. 关键词分层体系

### 2.1 分级规则

- **T1 核心词**：检索意图与整站主题一致，首页争夺；竞争激烈、需要长期积累。
- **T2 主题词**：对应单篇报告的主线，逐页争夺；这是本站最现实的一层。
- **T3 长尾词**：具体机制名/工具名/文档名，几乎无竞争，且是"内部黑话"检索者（安全工程师、固件开发、审计人员）真实的进入方式。系列报告的价值主要在这层。

> 说明：本方案没有关键词工具的流量数据（无法访问 Google Keyword Planner / Ahrefs / Semrush），T1/T2/T3 的划分依据是**意图宽度 × 本站正文实际覆盖度**，不是搜索量估算值。落地后以 GSC 的实际查询曝光校正。

### 2.2 中文词表

**T1 核心（首页）**
- Android 安全分析 / Android 安全研究报告
- Android 系统安全机制
- Android 漏洞分析
- 安卓底层安全
- 防御性安全研究

**T2 主题（按页归属）**
- 内核：Android 内核漏洞、内核提权、Linux 内核子系统、GKI 内核、内核攻击面、权限提升防御
- Framework：Android 权限模型、越权漏洞、Framework 信任边界、PendingIntent 风险、身份与 UID/AID
- 架构：Android 系统架构、系统分层、进程模型、Treble、HAL、SELinux 策略、seccomp
- 信任根：验证启动 AVB、硬件信任根、KeyMint、Keystore、StrongBox、远程密钥配置 RKP、DICE、文件级加密 FBE
- Binder：Binder IPC、AIDL、Stable AIDL、binderfs、Binder 事务缓冲区、IPC 权限判定
- 方法论：攻击面清点、漏洞研究方法论、模糊测试、漏洞分诊归因、补丁验证、检测覆盖

**T3 长尾（正文已实际出现的说法）**
`neverallow` · `SELinux coredomain` · `untrusted_app_all` · `seccomp allowlist` · `kCFI` · `ShadowCallStack` · `MTE` · `ASB 补丁级别` · `安全补丁级别 SPL` · `CTS SELinuxHostTest` · `syzkaller / syzbot` · `libFuzzer` · `ClusterFuzz` · `cc_fuzz` · `AIDL 模糊测试` · `Doom of Device Drivers` · `BaseMirror` · `LORIS` · `Bad Binder` · `基带 RCE` · `bughunters 奖励规则` · `90 天披露`

### 2.3 英文词表（用作同一页面的术语锚点，不单独建英文站）

AOSP security · Android kernel exploit mitigation · privilege escalation defense · Android attack surface · vulnerability research methodology · Binder IPC internals · SELinux policy · seccomp · verified boot AVB · KeyMint / StrongBox · RKP / DICE · GKI · fuzzing syzkaller ClusterFuzz · CVE · security bulletin

理由：中文检索者查这类底层主题时会直接输入英文术语（`KeyMint`、`binderfs`、`kCFI`）。这些词正文里已经是原词，只需保证**首次出现在标题或 H2 中**，不需要翻译。

---

## 3. 「逆向工程」关键词：覆盖度实测与处理策略

原始需求是"Android 底层与逆向工程关键词"。实测 grep 7 篇正文的结果：

| 关键词 | 正文覆盖 | 判定 |
| --- | --- | --- |
| 逆向 / 逆向工程 | 仅方法论报告 1 篇 | ⚠️ 单点 |
| 静态分析 / 动态分析 | 仅方法论报告 | ⚠️ 单点 |
| 模糊测试 / fuzz / syzkaller | 方法论、Framework、内核 | ✅ 3 篇 |
| 符号执行 | 0 篇 | ❌ 无内容 |
| Frida / Ghidra / smali / 脱壳 | 0 篇 | ❌ 无内容 |
| IDA | 1 处（信任根报告内，非工具语境） | ❌ 不算覆盖 |
| 混淆 | 3 篇（语境是控制流混淆，非"脱壳加固"） | ⚠️ 词同义不同 |
| PoC | 0 篇（正文刻意不使用 PoC 代码） | ❌ 有意为之 |

**处理策略（按性价比排序）：**

1. **不要在标题里塞未覆盖的逆向词。** Google 对"标题承诺—正文内容"不匹配的处理是相关性下降与高跳出，不是排名提升；`Frida 教程` 这类词与本站定位也不符。
2. **优先做 T3 术语自然吸收。** 把"静态分析、动态分析、模糊测试、分诊归因、补丁验证"这些**已有正文支撑**的方法论说法提进方法论报告的 H2 与描述——这是本轮唯一零风险的动作。
3. **逆向类关键词要成立，必须补内容，且只能补防御视角的两页：**
   - 《Android 应用逆向与加固的可观测面对比》——dex/so 形态、混淆与加壳的**检测特征**、完整性校验信号；不做脱壳教程。
   - 《Android 固件提取与静态审计流程（授权场景）》——super.img 分区结构、AOSP 符号与源码对齐、策略文件解析；限定自有设备/授权审计。
   这两页能把 `逆向`、`静态分析`、`固件审计`、`加固`、`so 库`、`smali` 从"零覆盖"变成"有正文承接"，之后才谈得上争夺这些词。
4. **红线**：`破解`、`绕过`、`脱壳工具`、`root 工具推荐`、`PoC 下载`、`免杀` 一类词**不承诺内容、不进标题、不做内链锚文本**。这与整站"防御性分析、不含利用代码"的定位（页脚声明）冲突，会带来无关流量与投诉。
5. **`meta keywords` 的定位**：Google 官方自 2009 年起不使用该标签做排名，Search Central 明确写它对索引和排名"完全没有作用"。因此本方案**不投入**改写 keywords 标签来提升 Google 收录；如仅为兼容其他引擎可作无害元数据补齐。关键词的实际控制面是 `<title>`、`description`、H 级标题、正文首段与内链锚文本。

---

## 4. 逐页改写文案（可直接替换 front-matter）

规则：`title` ≤ 30 个汉字（超出会被截断），末尾统一 ` | Android 安全报告集` 做品牌；`description` 70–110 字，前 40 字必须含 T2 词，并写出"不含利用代码"以匹配站方定位。

### 首页 `index.md`
```
title: "Android 系统安全分析报告集：内核、Binder、信任根与攻击面 | Android 安全报告集"
description: "7 篇 Android 平台防御性安全研究报告：内核漏洞与提权、Framework 越权与权限模型、系统架构与信任边界、GKI 内核子系统、硬件信任根与 KeyMint、Binder IPC 机制、跨域攻击面与漏洞研究方法论。全部基于 AOSP 与公开公告，不含漏洞利用代码、偏移量或攻击操作步骤。"
```

### 内核漏洞报告
```
title: "Android 内核漏洞类型学与提权防御：攻击面分级、CVE 案例与缓解矩阵 | Android 安全报告集"
description: "解析 Android 内核攻击面的分级方式、缺陷类型学、公开 CVE 的归因规律，以及 kCFI、ShadowCallStack、MTE、seccomp、SELinux 等缓解机制与补丁治理指标，并给出提权行为的取证与检测信号。防御性分析，不含利用代码与偏移量。"
```

### Framework 报告
```
title: "Android Framework 信任边界与越权漏洞：权限模型与 EoP 归因 | Android 安全报告集"
description: "梳理 Framework 层的身份、UID/AID 与权限模型，PendingIntent 与权限误用类缺陷的类型学，以及 Android 安全公告中 EoP 条目的归因方法、检测信号与常见误判。防御性分析，不提供绕过权限校验的做法。"
```

### 系统架构报告
```
title: "Android 系统架构分析：分层、进程模型、Treble 与信任边界 | Android 安全报告集"
description: "从应用、Framework、HAL、内核到安全域，拆解 Android 六段分层与进程模型，说明 SELinux 策略、seccomp 过滤、AID 身份机制和三条更新通道如何构成实际的信任边界。适合安卓底层安全与系统机制研究。"
```

### 内核结构报告
```
title: "Android Linux 内核结构与子系统：GKI 镜像、模块与安全控制落点 | Android 安全报告集"
description: "分析 GKI 内核的镜像形态、模块化边界、常见子系统与设备节点的供给方式，并定位各安全控制（能力、命名空间、策略、过滤）在内核中的实际落点与能力边界。面向 Android 底层与内核安全研究者。"
```

### 信任根报告
```
title: "Android 硬件信任根与密钥栈：AVB 验证启动、DICE、RKP 与 KeyMint | Android 安全报告集"
description: "串联验证启动 AVB、DICE 度量链、远程密钥配置 RKP、KeyMint 与 StrongBox、Gatekeeper/Weaver、密钥证明与文件级加密 FBE，说明每一环可核验的审计证据与失效条件。设备完整性与密钥管理方向的防御性分析。"
```

### Binder 报告
```
title: "Android Binder IPC 协议与运行时：BC_/BR_ 词表、AIDL 与权限判定 | Android 安全报告集"
description: "拆解 Binder 的 BC_/BR_ 控制协议、对象生命周期、事务缓冲区与线程池模型，以及 binderfs、SELinux 判定、Stable AIDL 与 RPC 机制，给出 IPC 攻击面的清点方法和可观测检测点。"
```

### 方法论报告（本轮重点，承接"逆向/分析"类词）
```
title: "Android 跨域攻击面地图与漏洞研究方法论：静态、模糊测试与归因 | Android 安全报告集"
description: "按应用、Framework、Binder、HAL、内核、媒体栈、基带、安全域与分发通道清点 Android 攻击面，并给出漏洞研究方法论：目标选择、静态与动态分析、fuzzing（syzkaller、libFuzzer、ClusterFuzz）、分诊归因、补丁验证与检测覆盖。含可在真机执行的只读核对脚本，不含利用代码。"
```

> 以上只改 `<title>`/`<description>`，不改正文与文件名；文件名保持英文即 URL slug，天然覆盖英文术语检索。**URL 不做重命名**，避免刚建立的收录信号作废与 404。

---

## 5. 技术侧改动清单（`tools/build-site.mjs`）

按投入产出排序，前三项做完即可提交收录。

| 优先级 | 改动 | 理由 |
| --- | --- | --- |
| P0 | JSON-LD 从 `WebPage` 扩为 `@type: TechArticle`，补 `headline`、`dateModified`、`inLanguage: zh-CN`、`mainEntityOfPage`、`isPartOf`（系列 `CollectionPage`）、`about`（数组，填 T2/T3 术语） | 给机器可读的主题锚点；`about` 是本站最接近"关键词声明"且不违反规范的位置 |
| P0 | 输出 `BreadcrumbList`（首页 → 报告页） | 搜索结果面包屑展示 + 强化层级语义 |
| P0 | 修 `Publish-Android/robots.txt` 的 Sitemap 行，改为本站域名 | 当前指向未开通的 GitHub Pages 主机，是无效引用 |
| P1 | 按 §4 落地 8 页 `title`/`description`，同步 `README.md` 首段 | `<title>` 是权重最高的页面级信号 |
| P1 | 每篇正文首个 H2 前保证一句"本文回答什么问题"含目标词 | 正文首段是相关性判定主区域，目前部分页面直接进术语表 |
| P1 | 页底加"同系列其他报告"内链块，锚文本用目标页 T2 词 | 锚文本是主题传递主要途径，比纯标题卡片更自然 |
| P2 | `twitter:card: summary_large_image` | 改善转发预览；不影响收录 |
| P2 | `sitemap.xml` 保持真实 `lastmod`，首页 priority 1.0、报告 0.8 | 伪造 `lastmod` 会降低抓取信任 |
| P3 | `meta keywords` | 仅为兼容部分引擎时补，明确标注无害元数据，**不作为收录手段** |

不建议做：为 7 篇报告各建关键词堆砌页、批量同义改写页、购买外链、页脚挂无关热词。

---

## 6. 收录提交渠道与节奏

1. **Google Search Console（当前阻塞点）**：`*.qoder.website` 是共享域，无法做 DNS 验证。可选路径：
   - URL 前缀属性 + HTML 标签/HTML 文件验证。文件验证需要站点根能返回 `google<hash>.html`，需在 builder 里加"额外静态文件复制"入口。
   - 若 Qoder Sites 平台自身提供 Search Console 对接，优先用平台入口，不手工猜验证方式。
   - 验证后：提交 `sitemap.xml` → 用「网址检查 → 请求索引入库」逐条提交 8 个 URL。
2. **Bing Webmaster Tools**：可从 GSC 一键导入验证，支持 IndexNow；用它推送 8 条 URL，对 Bing/Yandex/Naver/Seznam 生效。**Google 不支持 IndexNow**，不要用它的返回码判断 Google 收录。
3. **Yandex Webmaster**：若面向俄语安全社区可加，收录反馈快，用作早期相关性信号源。
4. **不做的渠道**：向搜索引擎"提交关键词列表"（不存在该接口）、付费收录服务、批量 ping。
5. **节奏**：每次重发站点只做一次 sitemap 重新提交；新增页面单独请求索引；不要每天重复提交同一 URL。

---

## 7. 验收指标

| 时间 | 指标 | 判定方式 |
| --- | --- | --- |
| 第 1 周 | GSC 完成所有权验证，sitemap 读取无错误，8 URL 已请求索引 | GSC 站点概览 + sitemap 状态 |
| 第 2–4 周 | 覆盖页数 ≥ 6/8（有曝光记录） | 效果报告按页拆分，而非只看总量 |
| 第 4–8 周 | T3 术语词（`binderfs`、`KeyMint`、`kCFI`、`Stable AIDL`、`neverallow`）出现非品牌查询曝光 | 查询列表过滤品牌词后人工核对 |
| 第 8–12 周 | 至少 1 个 T2 组合词进入前 2 页 | `site:` + 无痕核对，不做程序化排名抓取 |
| 全程 | 描述改写后首页 CTR 相对变化 | 同页面前后对比，不用总站 CTR |

若 8 周后 T3 词曝光仍为零，优先怀疑的是**站点未被授予索引**（共享域名信任度、抓取频次），而不是关键词选错——该分支要先解决收录，再谈词表。

---

## 8. 执行顺序

1. 你完成 GSC 验证（上一轮已交接；我无法操作你的搜索控制台账号，也不索取任何凭据）。
2. 我改 `tools/build-site.mjs`：TechArticle + BreadcrumbList + `about` 术语 + twitter card + 静态文件复制入口。
3. 我按 §4 替换 8 个 front-matter 的 `title`/`description`，并在方法论报告加含目标词的 H2。
4. 重建站点 → 复核渲染后的 `<head>` → `publish_site`（新 actionId）→ 匿名 curl 抽查。
5. 提交 sitemap、IndexNow 推送、逐 URL 请求索引。
6. 决定是否补 §3 的两篇逆向类内容——这是让"逆向工程"关键词成立的前提，工作量约等于两篇既有报告。

## 9. 参考来源

- [Google does not use the keywords meta tag in web ranking](https://developers.google.com/search/blog/2009/09/google-does-not-use-keywords-meta-tag)
- [Meta tags and attributes that Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)
- [IndexNow 协议与支持引擎](https://www.indexnow.org/)
- [IndexNow — Wikipedia（引擎支持列表）](https://en.wikipedia.org/wiki/IndexNow)
