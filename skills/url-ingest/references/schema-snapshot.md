# AGENTS.md Schema Snapshot(仅供 url-ingest skill 回退使用)

当仓库根目录无 `AGENTS.md` 时,以下约定作为默认 schema。
若仓库存在 `AGENTS.md`,则以仓库文件为准,本文件仅作参考。

---

## wiki 目录结构

```
wiki/
├── index.md / log.md / overview.md
├── 概念/<主题>/_index.md
└── 概念/ / 实体/ / 素材/ / 对比/ / 合成/
```

## wiki 页面 frontmatter 必填字段

```yaml
---
title: 页面标题
type: 概念 | 实体 | 素材摘要 | 对比 | 合成 | 目录
entity_kind: 产品 | 框架 | 人物 | 组织   # 仅 type=实体
sources: [raw/articles/xxx.md]
related: [[other-page]]
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: 高 | 中 | 低
tags: [中文标签, 国际术语保留英文]
---
```

## 命名约束

- wiki 文件夹与文件名一律中文(index.md/_index.md 保留英文)
- 品牌名/产品名(Dify/Neo4j 等)文件名可保留原名
- tags 默认中文,国际通用术语保留英文原形
- 内部引用一律用 `[[wiki-link]]` 双链

## Ingest 工作流

1. 读取 raw/ 源文件;不修改
2. 与用户简短确认关键要点(可跳过)
3. 在 wiki/素材/ 创建单源摘要(type: 素材摘要)
4. 刷新相关概念/实体页,补全 `[[]]` 链接
5. 维护 _index.md;首次分类创建,修改根 index.md
6. wiki/log.md 顶部插入记录(倒序)
7. 检查是否触发对比/合成建议

## raw/ 约定

- 保存原文正文,不接受只存 URL
- 不重命名剪藏后的文件
- frontmatter 不得含 `[[wiki-link]]` 双链

## 对比/ 与 合成/ 触发时机

- 对比/:某主题下 ≥2 个实体有足够积累(各 ≥2-3 源) → 建议生成对比页
- 合成/:Query 后信息密度高 → 询问用户是否归档
