# 个人 Wiki Schema

本仓库基于 Karpathy "LLM Wiki" 模式构建。心智模型:**Obsidian 是 IDE,
LLM 是程序员,本仓库是代码库,用户是产品经理 + 代码评审**。

把"摘要、链接、查重、归档"这些枯燥工作交给 LLM,人只做策展和高层判断;
维护成本趋近于零,知识库才能持续增长。

---

## 一、项目结构

```
mind-garden/
├── index.md                  # Obsidian 入口,轻量根目录(< 30 行)
├── raw/                      # 不可变源材料(LLM 只读,永不修改/删除)
│   ├── articles/             # 网页剪藏
│   ├── papers/               # 论文
│   ├── books/                # 书摘
│   ├── transcripts/          # 视频/播客转录
│   ├── images/               # 独立图片
│   ├── assets/               # 剪藏自带的附件(图片优先落此)
│   └── inbox/                # 待分类临时输入
├── wiki/                     # LLM 全权策展层
│   ├── log.md                # 倒序活动日志(仅追加)
│   ├── overview.md           # 写给"人"读的项目说明
│   ├── 概念/<主题>/_index.md  # 各概念分类的详细目录
│   ├── 概念/                 # 概念页(原理、方法、模式)
│   ├── 实体/                 # 实体页(产品、框架、人物、组织)
│   ├── 素材/                 # 单源摘要,一一对应 raw/
│   ├── 对比/                 # 横向对比页(详见 §六)
│   └── 合成/                 # 高价值问答归档(详见 §六)
├── skills/                   # 自定义 Qoder CLI skills
├── AGENTS.md                 # 本文件,即 schema 本体
└── .gitignore
```

**铁律**:`raw/` 永不修改;`wiki/` 由 LLM 全权产出;二者通过 frontmatter
`sources:` 字段连接。

---

## 二、raw/ 存储约定

### 2.1 必须保存原文正文,不接受只存 URL

防止链接腐烂(CSDN/微信迁移、付费墙、域名过期)、保证可重建、可审计、
可离线。

### 2.2 格式优先级

1. **Markdown** — 首选,LLM 友好、Git diff 友好、Obsidian 原生
2. **SingleFile HTML** — 可选备份,保留原始排版作"法律证据"
3. **PDF** — 论文、长报告
4. **图片** — 落到 `raw/assets/`,正文用相对路径引用

### 2.3 文件命名

- 文章: `raw/articles/<剪藏工具默认名>.md`
- 论文: `raw/papers/<年份>-<第一作者>-<简短主题>.{md,pdf}`
- 转录: `raw/transcripts/YYYY-MM-<来源>-<简短主题>.md`
- 同篇 HTML 备份与 Markdown 同名换后缀

### 2.4 文章 frontmatter 必填字段

```yaml
---
url: https://example.com/...
title: 原文标题
author: 作者(未知填 unknown)
site: 来源域名(如 linux.do)
captured: 2026-05-27
captured_via: web-clipper | singlefile | manual | mhtml
archive: raw/articles/xxx.html   # 可选,SingleFile 备份路径
language: zh | en
---
```

### 2.5 剪藏规则

- **图片必须本地化**到 `raw/assets/`,不留外链
- **完整正文**落盘,不接受摘要式剪藏
- **不修改原文**:typo、错字、过时论断都不在 raw 里改,由 wiki 层消化
- **不重命名 raw 文件**:文件名以剪藏当时为准。后期重命名会让所有 wiki 页
  frontmatter 的 `sources:` 引用断裂。剪藏前可调 Web Clipper 模板,落盘
  后即固化
- **frontmatter 不得含 `[[wiki-link]]` 双链**:Web Clipper 默认把
  `author` 包成 `[[张三]]`,剪藏后必须改回纯字符串 `author: 张三`,否则会
  污染图谱(产生与 wiki/ 实体不一致的空白节点),违反 raw/ 不参与互链的
  约定
- 大文件(>10MB)请用 Git LFS 或评估是否值得入库

### 2.6 推荐工具链

- **Obsidian Web Clipper**(首选) — 一键存 Markdown,可配置目标 vault
- **MarkDownload** — 备选,纯 Markdown 浏览器扩展
- **SingleFile** — 整页打包为单 HTML,做原貌存档
- 微信公众号: 用 mp-html-to-md 类脚本,或直接 SingleFile

### 2.7 LLM 行为约束

用户给出 URL 而非本地文件时,LLM **必须先提示用户完成剪藏**到 `raw/`,
再执行 ingest;不得仅根据 URL 联网抓取后直接产出 wiki 摘要。

### 2.8 摄入追踪(`raw/.ingested`)

**目的**:手动剪藏后不知道哪些已处理、哪些待处理。`.ingested` 记录每条
raw 文件的摄入时间与产出,让 LLM 能一次性扫描所有未处理文件。

**格式**(每行一条记录):

```
# 摄入追踪
# 格式: YYYY-MM-DD | raw文件路径 | wiki素材摘要
2026-05-28 | raw/articles/2020-es-pagination.md | 素材/ES分页看这篇就够了.md
```

**LLM 自动维护规则**:

- 每次 ingest 完成后,在 `raw/.ingested` 末尾**追加**一条记录
- 记录至少包含:日期、raw 文件路径、产出的 wiki 素材摘要文件名
- **不得删除或修改已有行**
- 若 raw 文件已被删除(罕见),标记 `[已删除]` 而非删行

**检测未处理文件**:

```
grep -v '^#' raw/.ingested | awk -F'|' '{print $2}' | sort > /tmp/ingested.txt
find raw/ -name '*.md' ! -name '.gitkeep' | sort > /tmp/all_raw.txt
comm -13 /tmp/ingested.txt /tmp/all_raw.txt
```

或用 Python 一行:`[f for f in glob('raw/**/*.md') if not any(f in l for l in open('raw/.ingested'))]`

**用户侧行为**:

- 用户手动剪藏后,无需任何操作;下次执行 `/process-inbox` 时自动发现并处理
- 若用户确认某个 raw 文件不需要 ingest(如工具站、纯导航页),可手动在
  `.ingested` 加 `[跳过]` 标记

**项目结构更新**:`raw/` 下新增 `.ingested` 文件:

```
raw/
├── .ingested                 # 摄入追踪(自动维护)
├── articles/
├── papers/
...
```

---

## 三、wiki 页面规范

### 3.1 frontmatter 必填字段

```yaml
---
title: 页面标题
type: 概念 | 实体 | 素材摘要 | 对比 | 合成 | 目录
entity_kind: 产品 | 框架 | 人物 | 组织   # 仅 type=实体 时必填
sources: [raw/articles/xxx.md]            # 指向 raw 原文,可多条
related: [[其它页]]
created: 2026-05-27
updated: 2026-05-27
confidence: 高 | 中 | 低
tags: [中文标签, RAG, 国际术语保留英文]
---
```

枚举值取中文,便于 Obsidian 节点信息阅读;`captured_via` 与 `language`
等技术标识保留英文。

### 3.2 命名与正文约束

- **wiki 文件夹与文件名一律中文**(例:`概念/检索增强生成/架构与演进.md`);
  仅 `index.md` 与 `_index.md` 作为约定标识保留英文
- **品牌名/产品名**(Dify、Neo4j、LangChain 等)文件名可直接用原名
- **tags 默认中文**;国际通用术语(RAG / LLM / Neo4j / Transformer 等)
  保留英文原形以便跨语言检索
- 文件名允许含空格,但避免特殊字符 `/ \ : * ? " < > |`
- 内部引用一律用 `[[wiki-link]]` 双链
- 单页结构推荐:**摘要 → 要点 → 关联 → 引用**
- 单页超过约 500 行须拆分

---

## 四、索引与导航策略(两层)

为防止 `index.md` 随库膨胀,采用两层导航。

### 4.1 根 `index.md`(仪表盘,目标 < 35 行)

采用 hub-and-spoke 风格:每分类下列出核心概念名称,实体列出名字,
素材按主题分组。细节留给 `_index.md`。模板:

```markdown
# Wiki 主目录

## 概念

### 检索增强生成(RAG)
[[架构与演进]] · [[意图路由]] · [[落地反模式]]
→ 详目:[[wiki/概念/检索增强生成/_index|RAG 目录]]

### 知识图谱
[[可解释推理]] · [[用于检索增强生成]]
→ 详目:[[wiki/概念/知识图谱/_index|知识图谱目录]]

### 对话系统
[[意图识别]] · [[多轮对话]]
→ 详目:[[wiki/概念/对话系统/_index|对话系统目录]]

## 实体
[[Neo4j]] · [[Dify]] · [[LangChain]]
→ 详目:[[wiki/实体/_index|全部实体]]

## 素材 (N 篇)
- RAG 实战:[[Dify 翻车实录]]
- 知识图谱:[[JOS 2022 综述]]
- 对话系统:[[意图识别]] · [[上下文建模]]
→ 详目:[[wiki/素材/_index|全部素材]]

---
导航:[[wiki/overview|项目总览]] · [[wiki/log|活动日志]] · 规则见 `AGENTS.md`
```

> 维护时:新增分类 → 加 `###` 小节;新增核心概念 → 追加到对应行。
> 素材区保持在 4 行内,超出时压缩为"→ 详目"链接。

### 4.2 子目录 `_index.md`(承载详细清单)

每个 `概念/<主题>/`、`实体/`、`素材/`、`对比/`、`合成/` 目录下放一份
`_index.md`,frontmatter `type: 目录`。模板:

```markdown
---
title: <分类名> 目录
type: 目录
updated: 2026-05-27
---

# <分类名>

- [[页面 A]] — 一句话概要 (N 源)
- [[页面 B]] — 一句话概要 (N 源)
```

### 4.3 索引维护规则

- **新建** wiki 页 → 在所属 `_index.md` 追加一行;若分类首次出现,创建
  `_index.md`,并在根 `index.md` 添加分类入口
- **更新** 源数量、摘要措辞 → 只改对应 `_index.md`,**不动根 index**
- **重命名/迁移** → 同步改 `_index.md` 与所有 `[[]]` 引用
- 目标:根 `index.md` 在多数 ingest 中仅需追加 1 行(新核心概念);新增分类或
  新核心概念时自然更新,日常源数/bugfix 不动根 index

---

## 五、Obsidian 图谱兼容性

本 schema 与 Obsidian 图谱视图原生契合:

- `[[wiki-link]]` 双链 → 图谱边
- frontmatter `tags:` → 标签聚类
- 目录结构 `概念/ 实体/ 素材/` → 可配 `colorGroups` 按 path 着色
- `_index.md` 作为分类枢纽,自然成为高入度节点

**关键约定**:`raw/` 下文件**不参与互链**,所有链接发生在 `wiki/` 内。
图谱视图过滤 path `wiki/` 即得干净视图。

---

## 六、`对比/` 与 `合成/` 的用途与触发时机

二者都**不是**通过剪藏新文章产生,而是对已有 wiki 内容的二次加工。

### 6.1 `对比/` — 对象驱动的横向比较

- **定义**:对两个或更多**已成熟的实体/概念**做并列比较
- **何时产生**:当某主题下有 ≥2 个 entity 都积累了一定信息(各自至少 2-3 源)
- **触发方**:通常由 LLM 在 ingest 末尾主动**建议产出**,经用户确认后写入
- **典型例子**:
  - `对比/Neo4j 与 NebulaGraph.md`
  - `对比/Dify、Coze 与 FastGPT.md`
  - `对比/对话系统与 RAG 在意图处理上的差异.md`
- **页面骨架**:被比较项 → 比较维度表 → 选型建议 → 关联引用

### 6.2 `合成/` — 问题驱动的答案归档

- **定义**:针对某个具体问题,综合多个 wiki 页给出的完整答案
- **何时产生**:Query 工作流中,LLM 给出高信息密度回答后**主动询问**用户是否
  归档;用户认可即落盘
- **触发方**:用户提问 + LLM 综合答 + 双方共识"值得复用"
- **典型例子**:
  - `合成/小团队 KG 项目最小可行架构.md`
  - `合成/如何选型企业知识库 RAG 平台.md`
  - `合成/2026 年自托管 LLM 推理栈推荐.md`
- **页面骨架**:原问题 → 结论 → 推导路径 → 引用清单 → 适用边界

---

## 七、Ingest 工作流

> 触发:用户说 `ingest <raw 文件路径>` 或 `摄入 <路径>`

1. 读取 `raw/` 下的源文件;**不得修改源文件**
2. 与用户简短确认关键要点(可跳过)
3. 在 `wiki/素材/` 创建或更新单源摘要
4. 刷新所有相关 `概念/` 与 `实体/` 页面,补全 `[[]]` 链接
5. 维护索引(见 §四):
   - 在对应 `_index.md` 追加/更新条目
   - 若分类首次出现,创建 `_index.md`,并在根 `index.md` 添加入口
6. **在 `wiki/log.md` 顶部插入**一条记录(倒序:最新在最上)
7. **检查是否触发对比/合成的产出建议**:
   - 若新增/刷新使某主题下实体 ≥2 且尚无对比页,主动建议生成 `对比/`
   - 若 ingest 中涌现的内容明显回答了一个常见问题,建议生成 `合成/`
8. **在 `raw/.ingested` 末尾追加**一条记录(格式见 §二.8)
9. 报告本次涉及的所有文件(创建 / 更新 / 引用 / 建议)

---

## 八、Query 工作流

> 触发:用户用自然语言提问

1. 从根 `index.md` 定位分类入口,再钻取到对应 `_index.md` 找具体页
2. 完整阅读相关页面后再作答
3. 回答中用 `[[wiki-link]]` 标注引用来源
4. 若回答信息密度高、综合了多页内容,**主动询问**是否归档到 `wiki/合成/`

---

## 九、Lint 工作流

> 触发:用户说 `lint` 或 `lint wiki`

1. 检测页面间的矛盾陈述
2. 找无入链的孤儿页
3. 列出频繁提及但缺独立页的概念
4. 标记被新材料覆盖的过期断言
5. 检查 `对比/`、`合成/` 触发条件是否已满足但未产出
6. 给出按优先级排序的修复任务清单

---

## 十、log.md 条目格式

> **倒序**:新条目插入文件顶部 `# Activity Log` 标题之下,旧条目依次向下。

```markdown
## [YYYY-MM-DD] <ingest|query|lint|refactor> | <简短标题>
Source: raw/articles/xxx.md
Created:
  wiki/素材/xxx.md
  wiki/概念/<主题>/xxx.md
Updated: index.md, wiki/实体/_index.md
Notes: 关键发现 / 冲突 / 待办 / 触发建议
```

---

## 十一、协作约定与用户偏好(持久化)

- **语言**:wiki 正文与术语**默认中文**,英文专有名词首次出现可括注原文
  (如"检索增强生成(RAG)")
- **wiki 命名**:文件夹与文件名一律中文(`index.md`、`_index.md` 例外;
  品牌名可保留原名)
- **tags 命名**:默认中文,国际通用术语保留英文原形
- **raw 保留剪藏原名**:落盘后不重命名,避免 wiki 引用断裂
- **log 倒序**:新条目顶部插入
- **commit 节奏**:每次 ingest 完成后,提示用户 `git commit`,以便单次回滚
- **schema 共演进**:遇到本文未覆盖的边界情况,LLM 主动建议修改本文件
- **诚实**:不要在 wiki 页中编造未出现于 `raw/` 的事实;不确定时标注
  `confidence: 低`,并在页内"待补充"小节列出
- **可选 Skills**:`skills/` 目录包含三个可选 skill(详见各 SKILL.md):
  - `url-ingest` — URL 自动抓取→ingest
  - `extract-knowledge` — 对话知识提炼
  - `process-inbox` — 扫描 raw/ 批量 ingest
