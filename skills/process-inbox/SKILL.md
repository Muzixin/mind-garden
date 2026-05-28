---
name: process-inbox
description: >
  扫描 raw/ 下所有未摄入的剪藏文件,自动执行完整 ingest 工作流。
  触发条件:用户说"/process-inbox"、"处理收件箱"、"批量摄入"、"处理未处理文件"、
  "扫描剪藏"、"还有哪些没处理"。
model: sonnet
tools: Bash, Write, Read, Edit, Glob, Grep, WebFetch
---

# Process Inbox Skill

## 概述

扫描 `raw/` 下所有 `.md` 文件,检查 `raw/.ingested` 追踪记录,找出所有
**未被 ingest 的文件**,逐一执行 AGENTS.md §七 Ingest 工作流。适用于
用户手动剪藏了多篇文章后的批量处理。

## 工作流程

### 步骤 1:扫描未处理文件

```bash
cd <repo-root> && grep -v '^#' raw/.ingested | awk -F' *\\| *' '{print $2}' | sed 's/^ *//' | sort > /tmp/ingested.txt && find raw/ -name '*.md' ! -name '.gitkeep' | sort > /tmp/all_raw.txt && comm -13 /tmp/ingested.txt /tmp/all_raw.txt
```

列出未处理文件清单,向用户展示:

```
发现 N 个未处理文件:

1. raw/articles/xxx.md — <标题>
2. raw/inbox/yyy.md — <标题>
...
```

### 步骤 2:用户确认(可跳过)

询问:

```
全部处理(Enter),或指定编号(如"1,3,5"),或跳过(skip)?
```

若用户直接回车或说"全部",处理全部。若指定编号,只处理选中。

### 步骤 3:逐个 Ingest

对每个选中文件,执行完整 AGENTS.md §七 Ingest 工作流:

1. 读取 raw 文件
2. 简短确认关键要点(批量模式下可跳过,仅报告标题)
3. 检查 frontmatter 完整性:若缺 `url`/`title`/`author`/`site`,从正文**推断**
   并补全;缺 `captured_via`,设为 `web-clipper`(手动剪藏默认)
4. 创建 `wiki/素材/<素材摘要>.md`
5. 刷新相关概念/实体页
6. 维护索引
7. `wiki/log.md` 顶部插入记录
8. 检查对比/合成触发
9. **在 `raw/.ingested` 末尾追加记录**

### 步骤 4:批量报告

处理完后输出汇总:

```
处理完成: N/N 成功
新建素材: xxx.md, yyy.md, ...
新建/更新概念页: ...
建议 commit: git add -A && git commit -m "ingest: 批量摄入 N 篇" && git push
```

## 批量处理策略

- 每处理 3 篇**暂停**,询问"继续还是暂停提交?"
- 若一篇 ingest 可能影响已处理篇目的概念页(如同一概念域),标注并优先
  合并写入
- 识别到 frontmatter 缺失时,从正文首段推断标题,从文件名推断日期,
  `captured_via` 填 `web-clipper`

## 与 url-ingest skill 的关系

- `url-ingest`:单篇 URL → 抓取 + ingest(用于在线文章)
- `process-inbox`:扫描 raw/ → 批量 ingest(用于手动剪藏后的清理)

两个 skill 都遵循同一 ingest 工作流,都记录 `raw/.ingested`。
