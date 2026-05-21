# Schema 设计指南

Schema 是将通用 LLM 转变为专业 wiki 维护员的关键配置。以下是经过生产环境验证的设计原则。

---

## 页面命名规范

| 页面类型 | 命名格式 | 示例 |
|---------|---------|------|
| 资料摘要 | `source-{关键词}.md` | `source-karpathy-llm-wiki.md` |
| 实体页 | `{实体名}.md` | `claude-code.md`、`karpathy.md` |
| 概念页 | `{概念名}.md` | `two-layer-automation.md` |
| 对比分析 | `{对象a}-vs-{对象b}.md` | `rag-vs-llm-wiki.md` |
| 工作流 | `{场景}-workflow.md` | `competitive-analysis-workflow.md` |
| 综述 | `overview-{主题}.md` | `overview-knowledge-management.md` |

命名规则：
- 全部小写，单词之间用连字符 `-` 分隔
- 避免过长文件名，保持简洁可辨识
- 实体名用英文（便于跨系统兼容），中文概念取意译

---

## 文章分类体系

编译 wiki 文章时，按以下三种类型分类，每种类型对应不同的写作风格和深度：

| 分类 | 说明 | 写作风格 |
|------|------|---------|
| **概念型**（concept） | 解释一个抽象概念、框架或方法论 | 先定义，再展开原理、证据、争议、应用场景 |
| **主题型**（topic） | 围绕一个具体主题的综合叙述 | 带时间线或叙事结构，综合多个来源讲故事 |
| **参考型**（reference） | 事实性查找信息（工具对比、参数列表、API 参考） | 表格/列表为主，简洁精确，少叙述 |

在 frontmatter 中用 `category` 字段标注：`category: concept | topic | reference`。分类影响模板选择和摘要风格。

---

## 双链格式

为同时兼容 Obsidian 图谱视图和 LLM 文件导航，交叉引用使用双链格式：

```markdown
[[slug|显示名称]] ([显示名称](wiki/category/slug.md))
```

- `[[wikilink]]` 部分供 Obsidian 渲染图谱和反向链接
- `[text](path)` 部分供 LLM 定位和读取文件
- 两者写在同一行，确保无论哪种工具都能正确导航

如果项目不使用 Obsidian，可以只保留标准 Markdown 链接。

---

## 原则一：先分类再提取

不同类型的来源需要不同的处理策略。50 页的报告和 2 页的信件需要不同的方法。

在 schema 中定义：
- 来源类型分类（论文、新闻、笔记、报告、对话记录等）
- 每种类型专属的提取流程和关注点
- 不同类型的摘要详细程度

这能节省大量 token，结果也更好。

---

## 原则二：索引的 Token 预算

使用四级渐进式披露：

| 层级 | Token 预算 | 内容 | 加载时机 |
|------|-----------|------|----------|
| L0 | ~200 | 项目上下文 | 每次会话都加载 |
| L1 | 1-2K | 索引文件 | 会话开始时加载 |
| L2 | 2-5K | 搜索结果/页面摘要 | 查询时按需加载 |
| L3 | 5-20K | 完整页面内容 | 深入阅读时加载 |

**关键纪律**：不读完索引就不读全文。

---

## 原则三：每种实体类型一个模板

为每种页面类型定义不同的字段结构，确保整个 wiki 的结构一致性。

### 人物页模板

```yaml
---
title: "人物姓名"
type: entity
entity_type: person
domain: []
sources: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: []
---
# 人物姓名
## 基本信息
- 角色/职位：
- 隶属组织：
## 关键论断与观点
## 相关实体
## 来源引用
```

### 概念页模板

```yaml
---
title: "概念名称"
type: concept
domain: []
sources: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: []
---
# 概念名称
## 定义
## 相关概念
## 支撑证据
## 未解问题
## 来源引用
```

### 来源摘要页模板

```yaml
---
title: "来源标题"
type: summary
source_file: raw/xxx.md
author: ""
source_date: YYYY-MM-DD
domain: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: []
---
# 来源标题
## 关键要点
## 提及的实体
## 引入的概念
## 与已有知识的关联
```

### 对比分析页模板

```yaml
---
title: "X vs Y 对比"
type: comparison
items_compared: [X, Y]
domain: []
sources: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: []
---
# X vs Y 对比
## 比较维度
## 发现
## 建议
## 来源引用
```

---

## 原则四：每个任务产出双重输出

每个任务应产出两个输出：
1. **直接产出** — 人类要求的回答/产出物
2. **wiki 更新** — 将新生成的知识更新回 wiki

如果不在 schema 中明确要求这一点，有价值的分析就会消失在聊天记录中。在 schema 中加入类似以下的规则：

> 每次完成分析或回答问题后，评估是否有值得持久化的知识。如果有，将其作为新页面或对现有页面的更新写入 wiki。

---

## 原则五：跨域标签

如果知识跨越多个项目、客户或研究领域，从第一天起就在页面 frontmatter 中添加 `domain` 标签。

```yaml
domain: [project-alpha, machine-learning]
```

出现在多个领域的共享实体（人物、组织、概念）会成为知识图谱中最有价值的节点。后期补加这个标签会非常痛苦——需要回溯所有已有页面逐一判断。

---

## 原则六：强制来源引用

要求 wiki 内容中必须附带来源引用。LLM 可以在不标注来源的情况下进行综合，而你不仔细看很难发现这种"无中生有"。

schema 中应包含：
- 每个事实性论断必须链接回具体来源文件
- 综合性结论必须列出所有参与综合的来源
- 引用格式统一（如 `[来源: raw/article-name.md]`）

人类的角色是主编——定期抽查引用的准确性，而非逐字审阅。

---

## Frontmatter 规范

每个 wiki 页面都应包含 YAML frontmatter。按页面类型，最小必填字段如下：

**资料摘要页**：
```yaml
---
tags: [source-summary, 领域标签]
source: "原文标题"
author: "作者名"
date: YYYY-MM-DD
url: "原始链接"
---
```

**实体页**：
```yaml
---
tags: [entity, 子类型标签]  # 子类型如 tool、person、organization
type: entity
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**概念页**：
```yaml
---
tags: [concept, 领域标签]
type: concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**对比/综合页**：
```yaml
---
tags: [comparison, 领域标签]
type: comparison
items_compared: [X, Y]
sources: [source-a.md, source-b.md]
created: YYYY-MM-DD
---
```

可选扩展字段（按需添加）：
- `domain: []` — 跨域标签，知识涉及多个领域时使用
- `sources: []` — 支撑该页面内容的原始资料列表
- `updated: YYYY-MM-DD` — 最近更新日期
- `confidence: high | medium | low` — 置信度评分（见下文）
- `category: concept | topic | reference` — 文章分类（见上文"文章分类体系"）

这使得 Obsidian Dataview 查询成为可能，并保持 wiki 的机器可读性。

---

## 置信度评分

每篇 wiki 文章应在 frontmatter 中标注 `confidence` 字段，表示内容可信程度：

| 等级 | 含义 | 适用场景 |
|------|------|---------|
| **high** | 多个可靠来源相互印证，知识已充分建立 | 教科书级别的确定知识 |
| **medium** | 单一来源，或来源部分同意，或尚未被复现的新发现 | 大多数文章的默认值 |
| **low** | 轶事性证据、单一非同行评审来源，或来源之间有分歧 | 需要更多证据验证 |

**使用规则**：
- 查询回答时，如果引用了低置信度文章，主动告知用户置信度状况
- lint 检查时，标记所有 `confidence: low` 的文章建议补充来源
- 当新来源增强了某篇文章的论据时，可以提升其置信度等级
- 当新来源与某篇文章产生矛盾时，应降低其置信度并标注矛盾
