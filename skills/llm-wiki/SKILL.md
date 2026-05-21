---
name: llm-wiki
description: >
  基于 Andrej Karpathy 的 LLM Knowledge Bases 模式，指导构建和维护个人知识库的方法论。当用户想要创建、维护或操作 LLM 驱动的 wiki/知识库时使用此技能。适用场景包括：构建 wiki、知识库管理、摄入文章/论文/笔记、组织研究资料、创建个人百科、LLM 辅助的结构化笔记，以及任何涉及将文档增量编译为互相链接的 Markdown 文件的场景。当用户询问 LLM Wiki 的摄入(ingest)、查询(query)、检查(lint)或 schema 设计等工作流时，也应使用此技能。
---

# LLM Wiki 技能

一套使用 LLM 构建持久化、可累积的知识库的方法论。与 RAG（每次查询都从原始文档重新检索拼凑答案）不同，LLM Wiki 将知识增量编译为结构化、互相链接的 Markdown wiki，每添加一个来源、每提出一个问题，wiki 都会变得更丰富。

**核心心智模型**：人类负责策划来源、引导分析方向、提出好问题；LLM 负责所有繁重的记账工作——摘要、交叉引用、归档和维护。

---

## 架构：三层结构

| 层次 | 所有者 | 职责 |
|------|--------|------|
| **原始资料** `raw/` | 人类（不可变） | 文章、论文、图片、笔记等源文档，LLM 只读不写 |
| **Wiki** `wiki/` | LLM（持久化） | 摘要页、实体页、概念页、对比分析页、索引和日志 |
| **Schema** 配置文件 | 人类与 LLM 协同 | 结构规则、命名约定、页面模板、工作流定义 |
| **产出物** `output/` | LLM 生成 | 报告、幻灯片、图表等交付物，与 wiki 知识页面分离 |

```
project/
  raw/                    # 原始资料（人类拥有）
    assets/               # 下载的图片和附件
  wiki/                   # Wiki（LLM 拥有）
    index.md              # 内容目录
    log.md                # 操作日志
  output/                 # 产出物（报告、幻灯片等交付物）
  CLAUDE.md / AGENTS.md   # Schema 配置文件（见下方说明）
```

> **Schema 文件命名约定**：Schema 文件的名称取决于使用的 agent 平台。仅当 agent 平台是 Claude 的产品（Claude Code、Claude.ai 等）时，使用 `CLAUDE.md`；在其它 agent 平台（如 OpenCode、Cursor、Windsurf 等）上，使用 `AGENTS.md`。文件内容和作用完全相同，只是文件名不同。

---

## 三大核心操作

- **摄入（Ingest）** — 当用户说"导入"、"处理这份资料"、"添加到知识库"、"ingest"时执行。新资料进入 `raw/`，LLM 读取、写摘要、更新索引、扫描并增量更新所有相关页面。
- **查询（Query）** — 当用户对知识库提问或要求分析时执行。先读 `index.md` 定位相关页面，深入阅读后综合回答。好的回答应反哺 wiki 成为新页面。
- **检查（Lint）** — 当用户说"检查知识库"、"维护一下"、"lint"时执行。审计 wiki 健康状态：找矛盾、过时内容、孤立页面、缺失页面、数据缺口，并建议下一步研究方向。
- **初始化** — 当用户说"创建知识库"、"初始化 wiki"、"搭建 wiki"时执行。参阅 `references/getting-started.md`。

> 详细的操作工作流和步骤，参阅 `references/operations.md`。
> 实战经验和踩坑总结，参阅 `references/practical-tips.md`。

---

## 索引系统

两个特殊文件使 wiki 无需 RAG 即可导航：
- **`index.md`**（面向内容）— 按分类组织的页面目录，每条含链接、一句话摘要、标签。LLM 回答问题时先读此文件。**索引是派生缓存**：`.md` 文件及其 frontmatter 才是真相源，index.md 是其派生视图，发现过时时应从文件系统重建。
- **`log.md`**（面向时间线）— 仅追加的操作记录，格式如 `## [YYYY-MM-DD] ingest | 标题`，可用 grep 快速回溯。

> 索引和日志的详细格式与示例，参阅 `references/indexing.md`。

---

## 核心原则

- **Wiki 是持久化的、可累积的知识产物。** 知识编译一次后持续更新，而非每次查询都重新推导。
- **LLM 绝不修改原始资料。** 原始资料不可变；wiki 是 LLM 的工作空间。
- **每次交互都可以丰富 wiki。** 查询、分析和对比都是潜在的 wiki 页面。
- **人类是主编，LLM 是作者。** 策划来源、引导分析、提出问题、验证内容——但把记账工作交给 LLM。
- **诚实承认知识缺口。** wiki 没有答案时明确说"不知道"，绝不编造。建议摄入什么资料来填补缺口。
- **增量优先，按需全量。** 默认只处理新增来源，完整重编译是昂贵操作，需要用户显式请求。
- **从小处开始，快速迭代。** 一个 Markdown 文件目录加一个 LLM agent 就够了。

---

## 环境行为

当此技能在非显式 wiki 命令时被激活（例如用户在包含 wiki 的项目中提问）：

1. 检查当前项目是否存在 `wiki/index.md`
2. 如果存在，读取 `index.md` 评估 wiki 是否能回答用户问题
3. 如果有相关内容 → 读取相关文章，附带引用回答
4. 如果没有相关内容 → 正常回答，并建议"这个内容可以通过摄入相关资料添加到 wiki"
5. 回答时注明引用文章的置信度等级（参阅 `references/schema-design.md` 的置信度评分）

---

## 参考文档目录

根据需要按需阅读以下参考文档：

| 文档 | 内容 | 何时阅读 |
|------|------|----------|
| `references/operations.md` | 摄入、查询、检查的详细工作流和步骤 | 执行具体操作时 |
| `references/schema-design.md` | Schema 设计六大原则、页面模板、Frontmatter 规范 | 创建或调整 schema 时 |
| `references/indexing.md` | index.md 和 log.md 的详细格式与示例 | 初始化或维护索引时 |
| `references/getting-started.md` | 快速上手清单、扩展性说明、工具推荐 | 从零开始搭建 wiki 时 |
| `references/practical-tips.md` | 资料获取技巧、交叉引用原则、导入规模预期 | 日常操作中遇到问题时 |
