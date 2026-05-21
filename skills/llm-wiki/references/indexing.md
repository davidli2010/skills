# 索引系统详解

两个特殊文件使 wiki 无需基于嵌入的 RAG 即可导航。它们用途不同，相互补充。

---

## index.md — 内容索引

按分类组织的所有 wiki 页面目录。LLM 回答问题时先读此文件以定位相关页面。

### 格式要求

每个条目包含：
- 指向页面的 wiki 链接
- 一句话摘要
- 分类标签
- 可选元数据（添加日期、来源数量）

### 示例

```markdown
# Wiki 索引

## 实体

- [[entity-karpathy]] — Andrej Karpathy，AI 研究者，LLM Wiki 方法论提出者 `#person` `#ai`
- [[entity-openai]] — OpenAI 公司概况与关键产品线 `#organization` `#ai`

## 概念

- [[concept-llm-wiki]] — 使用 LLM 增量构建持久化知识库的方法论 `#methodology`
- [[concept-rag]] — 检索增强生成，每次查询时从原始文档检索拼凑答案 `#methodology`

## 来源摘要

- [[summary-karpathy-gist]] — Karpathy 的 LLM Wiki 方法论完整文档 (2026-04-03) `#primary-source`
- [[summary-farzapedia]] — Farza 将 2500 条日记编译为 400 篇文章的实践 `#case-study`

## 对比与分析

- [[comparison-llm-wiki-vs-rag]] — LLM Wiki 与传统 RAG 方案的对比分析 `#analysis`
```

### 维护规则

- 每次摄入操作都必须更新此文件
- 新页面按分类插入到对应区域
- 摘要保持一句话，不超过 30 字
- 分类标签使用反引号包裹的 hashtag 格式

---

## log.md — 操作日志

仅追加的时间线记录。记录每次摄入、查询和检查操作，便于回溯 wiki 的演变历史。

### 格式要求

每条记录以统一前缀开头：`## [YYYY-MM-DD] 操作类型 | 标题`

这种格式的好处是可以用简单的 unix 工具解析：
- `grep "^## \[" log.md | tail -5` — 查看最近 5 条操作
- `grep "ingest" log.md` — 查看所有摄入记录

### 示例

```markdown
# 操作日志

## [2026-04-02] ingest | Karpathy LLM Wiki Gist
- 创建: wiki/summary-karpathy-gist.md
- 创建: wiki/entity-karpathy.md
- 创建: wiki/concept-llm-wiki.md
- 更新: wiki/index.md

## [2026-04-02] ingest | Farzapedia 案例分析
- 创建: wiki/summary-farzapedia.md
- 创建: wiki/entity-farza.md
- 更新: wiki/concept-llm-wiki.md（新增实践案例）
- 更新: wiki/index.md

## [2026-04-03] query | "LLM Wiki 和 RAG 的核心区别？"
- 创建: wiki/comparison-llm-wiki-vs-rag.md
- 更新: wiki/index.md

## [2026-04-05] lint | 每周健康检查
- 发现: concept-memex 被多处提及但无独立页面
- 发现: entity-openai 无入链（孤立页面）
- 建议: 创建 concept-memex.md
- 建议: 在 summary-karpathy-gist.md 中添加指向 entity-openai 的链接
```

### 维护规则

- 每次操作完成后立即追加记录
- 记录中列出所有创建和更新的文件
- lint 记录中列出发现的问题和建议
- 日志只追加，不修改历史条目

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

**过时检测**：在读取 index.md 时，比较其条目数与 `wiki/` 目录中的实际 `.md` 文件数（排除 index.md 和 log.md 自身）。如果不匹配，说明索引已过时。

**重建流程**：
1. 扫描 `wiki/` 目录中所有 `.md` 文件
2. 读取每个文件的 frontmatter（title、type、tags、summary）
3. 按分类重新生成 index.md
4. 在 log.md 中记录索引重建事件

**好处**：即使 index.md 被意外损坏或删除，也能从文件系统完整重建，不会丢失任何知识。这也意味着手动编辑 index.md 是不必要的——所有变更应通过正常的摄入/查询/检查工作流自动完成。
