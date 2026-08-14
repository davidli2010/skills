# 快速上手与扩展指南

> **触发条件**：当用户说"创建知识库"、"初始化 wiki"、"搭建 wiki"、"从零开始"时，执行以下流程。
>
> **OKF 对齐**：初始化产出的 wiki 遵循 OKF v0.2，`wiki/` 即 OKF bundle root。

## 快速上手清单

当从零开始搭建一个 LLM Wiki 时，按以下步骤执行：

### 第一步：准备工具
- 安装 Obsidian + Obsidian Web Clipper 浏览器插件
- 在 Obsidian 设置 > 文件与链接 中，将附件目录指向 `raw/assets/`
- 在 Obsidian 设置 > 快捷键 中，搜索 "Download"，为"下载当前文件的附件"绑定快捷键（如 `Ctrl+Shift+D`）

### 第二步：创建目录结构
```
mkdir -p raw/assets wiki/{entities,concepts,sources,comparisons,playbooks} output
touch wiki/index.md wiki/log.md
```

### 第三步：初始化索引和日志
- 在 `wiki/index.md` 顶部写入 `okf_version: "0.2"` frontmatter，正文写入分类标题框架（实体、概念、来源摘要、对比与分析、工作流）。格式详见 `indexing.md`。
- 在 `wiki/log.md` 中写入创建记录（`## YYYY-MM-DD` + `**init**`）。

### 第四步：起草 Schema
创建 Schema 配置文件，包含：
- 用户的领域和研究方向
- 适合其场景的页面类型（`type` 值，如 `Entity`/`Concept`/`Source Summary`/`Comparison`/`Playbook`）
- 文件命名约定与目录（详见 `schema-design.md`）
- 偏好的工作流程（全程参与 vs. 批量处理）
- 页面模板定义（详见 `schema-design.md`）
- 操作规则（详见 `schema-design.md` 的"Schema 必须包含的操作规则"章节）

> **文件命名**：如果当前使用的 agent 平台是 Claude 的产品（Claude Code、Claude.ai 等），Schema 文件命名为 `CLAUDE.md`；如果是其它 agent 平台（如 OpenCode、Cursor、Windsurf 等），命名为 `AGENTS.md`。两者内容和作用完全相同。

### 第五步：校准期（前 5-10 个来源）
- 与 LLM 一起逐个摄入来源，全程参与
- 阅读 LLM 写的摘要，检查更新质量
- 在人工确认的页面上补 `verified: { by: "human:<id>", at: ... }`（校准期是天然的信任建立时机）
- 根据实际效果调整 schema
- 确定检查频率（每周一次或每 N 次摄入后一次）

校准期之后，工作流趋于稳定，wiki 开始自行累积增长。

---

## 扩展性说明

| 规模 | 来源数量 | 导航方案 | 说明 |
|------|---------|----------|------|
| 小 | < 100 | `index.md` | 无需额外基础设施 |
| 中 | 100-500 | `index.md` + 搜索工具 | 推荐 [qmd](https://github.com/tobi/qmd)，支持 BM25/向量混合搜索，提供 CLI 和 MCP server |
| 大 | 500+ | 分层索引 + 搜索工具 | 每个分类一个子索引，更激进的摘要策略 |

Wiki 本质上就是一个 Markdown 文件的 git 仓库，天然具备版本历史、分支和协作能力。作为 OKF bundle，它可被任何遵循 OKF 的消费者读取和交换。

---

## 推荐工具

| 工具 | 用途 |
|------|------|
| **Obsidian** | Wiki 浏览器和编辑器，Graph View 可视化页面关系 |
| **Obsidian Web Clipper** | 浏览器插件，将网页一键转为 Markdown |
| **Marp** | Markdown 幻灯片格式，Obsidian 有对应插件 |
| **Dataview** | Obsidian 插件，基于 frontmatter 字段生成动态表格和列表 |
| **qmd** | 本地 Markdown 搜索引擎，支持 CLI 和 MCP server |

---

## 适用场景

- **个人成长** — 日记、健康记录、自我提升笔记的结构化管理
- **研究深耕** — 论文、报告、文章的长期积累，形成带演进论点的知识体系
- **读书伴侣** — 按章节摄入，为角色、主题、情节线建页，最终形成个人版 fan wiki
- **团队知识库** — 会议纪要、Slack 讨论、客户反馈、项目文档作为 raw source，LLM 自动维护结构化内部 wiki
- **竞品分析、尽职调查、旅行规划、课程笔记、爱好研究** — 任何需要长期积累并组织知识的场景

---

## 注意事项

- **对模型能力有要求** — 方法论默认使用顶级模型（Claude、GPT-4 级别），小模型效果可能打折扣。
- **冷启动需耐心** — 前 5-10 篇资料需要深度参与，不是"丢进去就能用"的东西。
- **验证成本不可忽视** — LLM 会在不引用来源的情况下做综合。用于严肃研究或商业决策时，抽查验证的时间要算进去。用 OKF 的 `verified` 记录人工确认，形成信任痕迹。
- **规模上限待探索** — 当前验证规模约 100 篇文章、40 万字。更大规模的表现尚未充分验证。
