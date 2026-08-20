# 索引系统详解

> **本文档基于 OKF v0.2 对齐**：`index.md` 遵循 OKF §8，`log.md` 遵循 OKF §9。两个特殊文件使 wiki 无需基于嵌入的 RAG 即可导航。它们用途不同，相互补充。

---

## index.md — 内容索引

按分类组织的所有 wiki 页面目录。LLM 回答问题时先读此文件以定位相关页面。

### 格式要求（OKF §8）

- bundle root 的 `index.md` 可以且应当带 frontmatter 声明版本；子目录 `index.md` 无 frontmatter。
- 正文按分类分节，每个条目为 `* [Title](relative-url) - description`（OKF §8 示例格式：`*` bullet + `-` 分隔符）：
  - `Title` 取被链页面 frontmatter `title`；
- `url` 用**相对路径**（相对 index.md 所在目录，如 `entities/karpathy.md`）；正文互链也使用相对于其所在页面的路径，以兼容普通 Markdown 渲染器；
  - description **取自被链页面 frontmatter `description`**，禁止手写第二套摘要。
- 条目还可附标签、来源数量等元数据。

### 示例

```markdown
---
okf_version: "0.2"
---
# Wiki 索引

## 实体

* [Karpathy](entities/karpathy.md) - AI 研究者，LLM Wiki 方法论提出者 `#person` `#ai`
* [OpenAI](entities/openai.md) - OpenAI 公司概况与关键产品线 `#organization` `#ai`

## 概念

* [LLM Wiki](concepts/llm-wiki.md) - 使用 LLM 增量构建持久化知识库的方法论 `#methodology`
* [RAG](concepts/rag.md) - 检索增强生成，每次查询时从原始文档检索拼凑答案 `#methodology`

## 来源摘要

* [LLM Wiki Gist](sources/karpathy-llm-wiki-gist.md) - Karpathy 的 LLM Wiki 方法论完整文档 (2026-04-03) `#primary-source`
* [Farzapedia 案例分析](sources/farzapedia.md) - Farza 将 2500 条日记编译为 400 篇文章的实践 `#case-study`

## 对比与分析

* [LLM Wiki vs RAG](comparisons/rag-vs-llm-wiki.md) - LLM Wiki 与传统 RAG 方案的对比分析 `#analysis`

## 工作流

* [竞品分析工作流](playbooks/competitive-analysis.md) - 竞品档案的建立与维护流程 `#workflow`
```

### 维护规则

- 每次摄入/反哺操作都必须更新此文件。
- 新页面按分类插入到对应区域。
- 摘要即该页面的 `description`——页面更新后若 `description` 变了，同步刷新条目，保持派生一致。
- 分类标签使用反引号包裹的 hashtag 格式。
- 每个子目录（`entities/`、`concepts/` 等）可维护自己的 `index.md`，形成分层渐进披露（OKF §8）。

---

## log.md — 操作日志

仅追加的时间线记录。记录每次摄入、查询和检查操作，便于回溯 wiki 的演变历史。

### 格式要求（OKF §9）

- 日期头 MUST 用 ISO 8601 `YYYY-MM-DD` 形式，**不带方括号**：`## YYYY-MM-DD`，最新在前。
- 条目为散文或列表；前导加粗词（`**ingest**`、`**query**`、`**lint**`）是约定，非要求。
- 每个条目列出创建/更新的文件路径（相对项目根，如 `wiki/concepts/llm-wiki.md`）。

可用 unix 工具解析：
- `grep "^## " log.md | head -5` — 查看最近 5 条操作（最新在前）
- `grep "ingest" log.md` — 查看所有摄入记录

### 示例

```markdown
# 操作日志

## 2026-04-05
* **ingest**: 摄入 Karpathy LLM Wiki Gist
* **update**: 创建 wiki/sources/karpathy-llm-wiki-gist.md
* **update**: 创建 wiki/entities/karpathy.md
* **update**: 创建 wiki/concepts/llm-wiki.md
* **update**: 更新 wiki/index.md

## 2026-04-03
* **init**: 创建 wiki 目录结构与索引框架
```

### 维护规则

- 每次操作完成后立即追加记录。
- 记录中列出所有创建和更新的文件。
- lint 记录中列出发现的问题和建议。
- 日志只追加，不修改历史条目。

---

## 为什么不需要 RAG

在中等规模下（约 100 个来源，数百个页面），index.md 配合 LLM 的上下文窗口就足以完成导航。原因：

1. **LLM 自己维护索引** — 索引质量随 wiki 成长自动提升
2. **索引是语义级别的** — 不是关键词匹配，而是 LLM 理解后写的摘要
3. **零基础设施** — 不需要向量数据库、嵌入模型或额外服务
4. **交叉引用已内置** — 页面之间的链接本身就是导航网络

当 wiki 规模超出 index.md 的覆盖能力时，可以引入搜索工具（参阅 `getting-started.md` 的扩展性说明）。

---

## 派生索引协议

index.md 是派生缓存，不是真相源。`.md` 文件及其 YAML frontmatter 才是权威数据。

**过时检测**：在读取 index.md 时，比较其条目数与 `wiki/` 中的实际 `.md` 文件数（排除 index.md、log.md 及子目录 index.md 自身）。如果不匹配，说明索引已过时。

**重建流程**：
1. 扫描 `wiki/` 目录中所有 `.md` 文件（排除 index.md/log.md）。
2. 读取每个文件的 frontmatter（`title`、`type`、`description`、`tags`）。
3. 按 `type`/目录分类，用 `description` 作为条目摘要，重新生成 index.md。
4. 在 log.md 中记录索引重建事件。

**好处**：即使 index.md 被意外损坏或删除，也能从文件系统完整重建，不会丢失任何知识。这也意味着手动编辑 index.md 是不必要的——所有变更应通过正常的摄入/查询/检查工作流自动完成。
