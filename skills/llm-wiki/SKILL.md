---
name: llm-wiki
version: 0.2.0
description: >
  基于 Andrej Karpathy 的 LLM Knowledge Bases 模式，指导构建和维护个人知识库的方法论。生成的 wiki 遵循 Open Knowledge Format (OKF) v0.2 规范，使知识语料具备可溯源（provenance）、可信任（trust）、可判新（freshness）、可管理生命周期（lifecycle）的能力。当用户想要创建、维护或操作 LLM 驱动的 wiki/知识库时使用此技能。适用场景包括：构建 wiki、知识库管理、摄入文章/论文/笔记、组织研究资料、创建个人百科、LLM 辅助的结构化笔记，以及任何涉及将文档增量编译为互相链接的 Markdown 文件的场景。当用户询问 LLM Wiki 的摄入(ingest)、查询(query)、检查(lint)或 schema 设计等工作流时，也应使用此技能。
---

# LLM Wiki 技能（OKF v0.2 对齐）

一套使用 LLM 构建持久化、可累积的知识库的方法论。与 RAG（每次查询都从原始文档重新检索拼凑答案）不同，LLM Wiki 将知识增量编译为结构化、互相链接的 Markdown wiki，每添加一个来源、每提出一个问题，wiki 都会变得更丰富。

**核心心智模型**：人类负责策划来源、引导分析方向、提出好问题；LLM 负责所有繁重的记账工作——摘要、交叉引用、归档和维护。

**OKF 对齐**：wiki 产出遵循 [Open Knowledge Format v0.2](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) 规范。`wiki/` 目录即一个 OKF **knowledge bundle**，其 frontmatter 五大家族（provenance、trust、lifecycle、computation、其他扩展）使 agent 维护的知识语料自描述、可溯源、可验证。细则见 `references/schema-design.md`。

---

## 架构：三层结构

| 层次 | 所有者 | 职责 |
|------|--------|------|
| **原始资料** `raw/` | 人类（不可变） | 文章、论文、图片、笔记等源文档，LLM 只读不写；位于 bundle 外，通过 `sources[].resource` 引用 |
| **Wiki** `wiki/`（= OKF bundle root） | LLM（持久化） | 实体页、概念页、来源摘要页、对比分析页、工作流页、索引和日志 |
| **Schema** 配置文件 | 人类与 LLM 协同 | 结构规则、命名约定、页面模板、工作流定义 |
| **产出物** `output/` | LLM 生成 | 报告、幻灯片、图表等交付物，位于 bundle 外 |

```
project/
  raw/                    # 原始资料（人类拥有，不可变）
    assets/               # 下载的图片和附件
  wiki/                   # Wiki（LLM 拥有）= OKF bundle root
    index.md              # 内容目录（含 okf_version 声明）
    log.md                # 操作日志
    entities/             # 实体页（type: Entity）
      index.md
    concepts/             # 概念页（type: Concept）
      index.md
    sources/              # 来源摘要页（type: Source Summary）
      index.md
    comparisons/          # 对比分析页（type: Comparison）
      index.md
    playbooks/            # 工作流/操作指南页（type: Playbook）
      index.md
  output/                 # 产出物（报告、幻灯片等交付物）
  CLAUDE.md / AGENTS.md   # Schema 配置文件（见下方说明）
```

> **Schema 文件命名约定**：Schema 文件的名称取决于使用的 agent 平台。仅当 agent 平台是 Claude 的产品（Claude Code、Claude.ai 等）时，使用 `CLAUDE.md`；在其它 agent 平台（如 OpenCode、Cursor、Windsurf 等）上，使用 `AGENTS.md`。文件内容和作用完全相同，只是文件名不同。

---

## 页面类型（`type` 字段）

`type` 是 frontmatter 中唯一必填字段（OKF §4.1），用于路由、过滤和呈现。本技能约定的值：

| type 值 | 存放目录 | 说明 |
|---------|---------|------|
| `Entity` | `entities/` | 人物、组织、工具等实体，可用扩展键 `entity_type: person \| tool \| organization` 细分 |
| `Concept` | `concepts/` | 抽象概念、框架、方法论 |
| `Source Summary` | `sources/` | 某一原始资料的摘要页 |
| `Comparison` | `comparisons/` | 对比分析（可用扩展键 `items_compared: [X, Y]`） |
| `Playbook` | `playbooks/` | 工作流、操作指南、综述 |

type 值不集中注册，可扩展。消费者遇到未知 type 时应优雅降级为通用概念，不得拒绝（OKF §4.1、§11）。

---

## Frontmatter 五大家族（OKF §4–5、§10）

| 家族 | 字段 | 作用 |
|------|------|------|
| 必填 | `type` | 概念类型 |
| 推荐 | `title`、`description`、`resource`、`tags` | 显示名、单句摘要（索引由此派生）、底层资产 URI、标签 |
| **Provenance** | `sources[]`（含 `id`/`resource`/`title`/`author`/`usage_count`/`last_modified`）、`usage_window` | 内容从何而来、来源可信信号 |
| **Trust** | `generated: { by, at }`、`verified: [{ by, at }]` | 谁写的、谁确认的；信任层级由 `human:` 前缀推导 |
| **Lifecycle** | `status: draft \| stable \| deprecated`、`stale_after`（绝对日期） | 生命周期与时效性 |

信任层级（由 `verified` 推导，OKF §5.3）：无 `verified` ⇒ unverified；仅非 `human:` 验证 ⇒ machine-confirmed；有 `human:<id>` ⇒ human-reviewed。**记录信号，不记录评分**——不使用 `confidence` 字段。

---

## Actor 约定（OKF §7）

记录身份的字段（`generated.by`、`verified[].by`）使用统一约定：

- `<producer>/<version>` — agent 或工具，如 `llm-wiki/claude-sonnet-4`
- `human:<id>` — 人，如 `human:david`
- `process:<id>` — 自动进程，如 `process:nightly`

信任层级判定键控在 `human:` 前缀上，因此人工撰写或确认的内容必须使用 `human:<id>`。

---

## 三大核心操作

- **摄入（Ingest）** — 当用户说"导入"、"处理这份资料"、"添加到知识库"、"ingest"时执行。新资料进入 `raw/`，LLM 读取、写来源摘要页（入 `sources/`）、更新索引、扫描并增量更新所有相关页面。
- **查询（Query）** — 当用户对知识库提问或要求分析时执行。先读 `index.md` 定位相关页面，深入阅读后综合回答，并标注引用页面的信任层级。好的回答应反哺 wiki 成为新页面。
- **检查（Lint）** — 当用户说"检查知识库"、"维护一下"、"lint"时执行。审计 wiki 健康状态：找矛盾、过时内容（`stale_after` 过期）、孤立页面、缺失页面、数据缺口，并建议下一步研究方向。
- **初始化** — 当用户说"创建知识库"、"初始化 wiki"、"搭建 wiki"时执行。参阅 `references/getting-started.md`。

> 详细的操作工作流和步骤，参阅 `references/operations.md`。
> 实战经验和踩坑总结，参阅 `references/practical-tips.md`。

---

## 索引系统

两个特殊文件使 wiki 无需 RAG 即可导航（OKF §3.1 保留文件名）：
- **`index.md`**（面向内容）— 按分类组织的页面目录，每条含链接、一句话摘要（取自页面 frontmatter `description`）、标签。LLM 回答问题时先读此文件。**索引是派生缓存**：`.md` 文件及其 frontmatter 才是真相源，index.md 是其派生视图，发现过时时应从文件系统重建。
- **`log.md`**（面向时间线）— 只增不改的操作记录，日期头为 ISO `YYYY-MM-DD`，**最新在前**（OKF §9 结构要求），可用 grep 快速回溯。

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
- **记录信号，不记录结论。** 只记录客观事实（来源、时间、actor），信任与可信度由消费者推导，不写评分。

---

## 环境行为

当此技能在非显式 wiki 命令时被激活（例如用户在包含 wiki 的项目中提问）：

1. 检查当前项目是否存在 `wiki/index.md`
2. 如果存在，读取 `index.md` 评估 wiki 是否能回答用户问题
3. 如果有相关内容 → 读取相关文章，附带引用回答，并注明引用页面的信任层级（unverified / machine-confirmed / human-reviewed，推导见 `references/schema-design.md`）
4. 如果没有相关内容 → 正常回答，并建议"这个内容可以通过摄入相关资料添加到 wiki"
5. 引用 low 信任（unverified）或可能过期（`stale_after` 已过）的页面时，主动告知状况

---

## 参考文档目录

根据需要按需阅读以下参考文档：

| 文档 | 内容 | 何时阅读 |
|------|------|----------|
| `references/operations.md` | 摄入、查询、检查的详细工作流和步骤 | 执行具体操作时 |
| `references/schema-design.md` | OKF frontmatter 规范、页面模板、命名与链接约定、生成页面前后必查的「OKF 合规自检清单」、最小完整示例 | 创建或调整 schema 时，或生成页面前后 |
| `references/indexing.md` | index.md 和 log.md 的详细格式与示例 | 初始化或维护索引时 |
| `references/getting-started.md` | 快速上手清单、扩展性说明、工具推荐 | 从零开始搭建 wiki 时 |
| `references/practical-tips.md` | 资料获取技巧、交叉引用原则、导入规模预期 | 日常操作中遇到问题时 |
