---
name: url-ingest
description: >
  将网页 URL 自动抓取为 Markdown 原文落盘到 raw/articles/,然后执行完整的
  ingest 工作流(创建素材摘要、刷新概念/实体页、维护索引、写日志)。触发条件:
  用户提供 HTTP/HTTPS URL 并要求"ingest"、"摄入"、"剪藏"、"保存这篇文章"、
  "存档"、"存到知识库"。也适用于 `/ingest-url <url>` 斜杠命令。
model: sonnet
tools: Bash, Write, Read, Edit, Glob, Grep, WebFetch
argument-hint: <url>
---

# URL Ingest Skill

## 概述

接受一个网页 URL,自动完成:抓取原文 → 落盘 raw/ → 执行 AGENTS.md 定义的
ingest 工作流。一条命令完成从外部网页到结构化 wiki 知识的全流程。

## 剪藏引擎(三层降级)

本 skill 使用 Defuddle 作为主引擎 — 这是 Obsidian Web Clipper 的底层解析
库,由 Obsidian 作者 Kepano 开源,通过 CLI 调用。它与 Obsidian Web Clipper
使用**相同的解析算法**,输出干净 Markdown。

### 第一层:Defuddle CLI(主引擎,优先)

```bash
npx --yes defuddle parse "<URL>" -m -o /tmp/defuddle_out.md
```

- `npx --yes` 自动安装(无需预装)
- `-m` 输出 Markdown
- `-o` 直接写入文件,避免 stdout 截断
- 返回结构化元数据(title/author/domain/language)通过 stdout 的 JSON

**Defuddle 覆盖范围**:与 Obsidian Web Clipper 相同 — 公开博客、技术论坛、
学术期刊、新闻站点。依赖服务端渲染的 HTML,无法处理纯 SPA(客户端渲染)。

### 第二层:WebFetch(兜底,仅 Defuddle 失败时)

仅当 Defuddle CLI 返回空或报错时使用 WebFetch。这是 LLM 内置工具,直接
解析 HTML → Markdown。**不可作为首选**,因为:

- 无法提取完整 frontmatter 元数据
- 对复杂页面(嵌套表格、数学公式、代码块)解析不稳定
- 可能被大型页面截断

### 第三层:手动 Clipper(最后手段)

Defuddle 和 WebFetch 都失败时(登录墙/验证码/纯 SPA),提示用户:

```
此页面需要手动剪藏。请用 Obsidian Web Clipper 保存到 raw/articles/,
然后告诉我 `ingest raw/articles/<文件名>.md`
```

## 工作流程

### 步骤 1:URL 合法性检查

- 必须是 HTTP/HTTPS 协议
- 排除工具站/导航站/搜索引擎首页
- 若是指向索引页(`docs.xxx.com` 根),提示指定子页面

### 步骤 2:Defuddle 抓取

```bash
npx --yes defuddle parse "<URL>" -m -o /tmp/defuddle_out.md
```

读取 `/tmp/defuddle_out.md` 内容。若文件非空且行数 > 5,跳到步骤 3。

否则,从 stdout 提取 JSON 元数据(title/author/domain/language),然后
执行步骤 2b(WebFetch 兜底)。

### 步骤 2b:WebFetch 兜底(仅 Defuddle 失败)

```
WebFetch(url, prompt="提取完整正文为 Markdown")
```

若 WebFetch 也失败,执行第三层降级(手动 Clipper 提示)。

### 步骤 3:生成 raw/ 文件

根据 AGENTS.md 约定:

- **文件名**:从标题自动派生,格式 `<YYYY-MM>-<简短主题>.md`。简短主题取
  标题前 6-8 个中文字的 slug 化结果
- **保存路径**:`raw/articles/<文件名>.md`
- **frontmatter**:

```yaml
---
url: <原始 URL>
title: <来自 Defuddle/WebFetch>
author: <来自 Defuddle,未知填 unknown>
site: <来源域名>
captured: <当前日期 YYYY-MM-DD>
captured_via: defuddle | web-fetch
language: <zh 或 en>
---
```

将 Markdown 正文(Defuddle/WebFetch 输出)写入 `raw/articles/<文件名>.md`。

**不重命名**:文件名一次确定,落盘后不修改。`captured_via` 如实记录来源。

### 步骤 4:执行 Ingest 工作流

完全按照仓库根 `AGENTS.md` §七执行:

1. 读取 `raw/articles/<文件名>.md`
2. 向用户简短确认关键要点(可跳过)
3. 在 `wiki/素材/` 创建单源摘要(type: 素材摘要)
4. 刷新所有相关 `wiki/概念/` 与 `wiki/实体/` 页面,补全 `[[]]` 链接
5. 维护索引;首次出现的分类创建 `_index.md` 并更新根 `index.md`
6. `wiki/log.md` 顶部插入记录(倒序)
7. 检查对比/合成触发条件
8. **在 `raw/.ingested` 末尾追加**一条记录

### 步骤 5:报告 + commit 提示

输出变更清单,然后提示:

```
> 建议 `git add -A && git commit -m "ingest: <标题>" && git push`
```

## 已知可处理/不可处理的站点

**✅ Defuddle 已验证**:
- 腾讯云(cloud.tencent.com)
- 学术期刊(jos.org.cn)
- linux.do(论坛)
- CSDN、博客园、简书、掘金、阿里云开发者社区(同类型静态渲染)

**❌ Defuddle/WebFetch 均失败,需手动 Clipper**:
- 知乎(zhuanlan.zhihu.com) — anti-bot 拦截
- 微信公众号(mp.weixin.qq.com) — 需微信客户端 Cookie
- 飞书文档(feishu.cn/docx) — 登录墙 + 动态渲染

## Schema 引用

本 skill 遵循仓库 `AGENTS.md` 的所有约定。若仓库无 AGENTS.md,回退到
`references/schema-snapshot.md` 的默认值。

关键约定速查:
- wiki 文件夹中文:概念/ 实体/ 素材/ 对比/ 合成/
- frontmatter type 枚举中文
- tags 默认中文,国际通用术语保留英文
- log.md 倒序
- raw/ 不重命名;frontmatter 不含 `[[wiki-link]]` 双链

## 资源

- `references/schema-snapshot.md` — AGENTS.md 缺失时的默认 schema
