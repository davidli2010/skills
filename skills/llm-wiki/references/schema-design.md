# Schema 设计指南

> **本文档基于 OKF v0.2 规范对齐**。frontmatter 五大家族（provenance、trust、lifecycle）遵循 [OKF §4–§5](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)。Schema 是将通用 LLM 转变为专业 wiki 维护员的关键配置。以下是经过生产环境验证的设计原则。

---

## 目录与页面命名规范

`wiki/` 是 OKF bundle root。页面按类型放入子目录，**去掉前缀命名**，前缀含义移入 frontmatter `type` 字段：

| type 值 | 目录 | 命名格式 | 示例 |
|---------|------|---------|------|
| `Entity` | `entities/` | `{实体名}.md` | `karpathy.md`、`openai.md` |
| `Concept` | `concepts/` | `{概念名}.md` | `llm-wiki.md`、`two-layer-automation.md` |
| `Source Summary` | `sources/` | `{来源关键词}.md` | `karpathy-llm-wiki-gist.md` |
| `Comparison` | `comparisons/` | `{obj-a}-vs-{obj-b}.md` | `rag-vs-llm-wiki.md` |
| `Playbook` | `playbooks/` | `{场景}.md` | `incident-response.md` |

命名规则：
- 全部小写，单词之间用连字符 `-` 分隔
- 避免过长文件名，保持简洁可辨识
- 实体名用英文（便于跨系统兼容），中文概念取意译

概念 ID = bundle 内文件路径去掉 `.md` 后缀（如 `concepts/llm-wiki.md` 的概念 ID 是 `concepts/llm-wiki`）。**位置即身份**，因此移动文件会改变 ID，应同步更新所有入链。

每个子目录可带自己的 `index.md`，构成分层渐进披露（OKF §8 允许任意层级 index）。

> **保留文件名（OKF §3.1）**：`index.md` 与 `log.md` 在任何层级都有定义含义，**MUST NOT** 用作概念文档。所有其它 `.md` 文件才允许是概念页。

---

## 文章分类风格

编译 wiki 文章时，可参考以下三种写作风格（写作风格是页内约定，不写入 frontmatter）：

| 风格 | 适用 type | 写作风格 |
|------|---------|---------|
| **解释型** | `Concept` | 先定义，再展开原理、证据、争议、应用场景 |
| **叙事型** | `Source Summary`、`Playbook` | 带时间线或叙事结构，综合多个来源讲故事 |
| **查询型** | `Comparison`、`Entity` | 表格/列表为主，简洁精确，少叙述 |

---

## 链接与路径（OKF §6）

`wiki/` 内的交叉引用使用 **标准 Markdown 相对链接**，路径相对于当前页面所在目录。不要以 `/` 开头：普通 Markdown 渲染器会将其解析为站点或文件系统根，而不是 `wiki/` 根。移动页面时必须同步更新其入链和出链：

```markdown
# 概念页正文
参见 [RAG](../concepts/rag.md) 了解对比。

# 来源摘要页引用图片（跨 bundle，相对路径）
![架构图](../raw/assets/article-fig1.png)
```

链接规则：
- wiki 内互链：使用相对于当前页面的路径，如来源摘要页链接概念页时写作 `[显示名称](../concepts/文件名.md)`；同目录页面写作 `[显示名称](文件名.md)`。
- 引用 bundle 外文件（`raw/`、`output/`）：相对路径，如 `../raw/xxx.md`（OKF §6.2 允许）。
- **断链不是错误**（OKF §6.1）：链接目标不存在可能只是"尚未写出的知识"，不得因断链拒绝消费。lint 中区分"指向已删页面的断链"（应修复）与"指向未建成概念的断链"（应建议建页）。
- 链接是**未类型化的有向边**：关系种类（父子、引用、相关）由正文措辞传达，不在链接上编码。

**Path 型字段**（`resource`、`sources[].resource`）统一接受三种形式（OKF §6.2）：
- 绝对 URL：`https://...`
- bundle-relative 路径：`/concepts/xxx.md`（OKF 允许；仅用于机器消费或已配置 bundle 根解析的工具）
- 相对路径：`../computations/revenue.md`

`sources[].resource` 有一个例外：可以是**作用域描述符**（描述一个群体/范围而非具体文件，如"某项目 BQ 中的全部查询"），它不是路径。

**`references/` 约定（可选，OKF §6.3）**：可在 bundle 内建 `references/` 子目录，镜像外部材料、运行指令或代码为一等概念，供 `resource`/`sources[].resource` 指向。本技能不强制，主要是 `raw/` 已承担该角色。

---

## 正文结构（OKF §4.2）

正文是标准 markdown，优先使用结构化内容（标题、列表、表格、围栏代码块）而非自由散文。无强制章节。按 OKF §4.2，以下标题有**约定意义**，适用时建议使用：

| 标题 | 用途 |
|------|------|
| `# Schema` | 资产的结构化描述（列/字段定义），描述数据表/接口时用 |
| `# Examples` | 具体用法示例，多用围栏代码块 |
| `# Computation` | 仅用于 `type: Attested Computation`（见 §10） |

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

## 原则三：每种页面类型一个模板

以下模板的 frontmatter 遵循 OKF v0.2。**`type` 是唯一必填字段**；其余按需填写。所有标注均可选，缺省也能消费（OKF §11）。

### 统一 frontmatter 骨架

```yaml
---
type: Concept                 # REQUIRED — Entity | Concept | Source Summary | Comparison | Playbook
title: "显示名称"              # 可选，缺省由文件名推导
description: "一句话摘要"       # 可选；index.md 的摘要由此派生，务必维护
resource: "https://..."       # 可选；底层资产 URI（实体页/摘要页建议有）
tags: [tag1, tag2]            # 可选
domain: []                    # 可选扩展键；跨域标签
# --- provenance (OKF §5.1) ---
sources:
  - id: source-id             # 可选；正文 footnote 归因用，建议有
    resource: ../raw/xxx.md   # REQUIRED（条目内）；URL / bundle 路径 / 相对路径 / 作用域描述
    title: "来源标题"          # 可选
    author: "human:xxx"       # 可选；credibility signal，用 actor 约定
    usage_count: 150          # 可选；粗略活性信号（多少读者/调用），由 usage_window 框定
    last_modified: YYYY-MM-DD # 可选；来源自身最近变更
usage_window: { from: YYYY-MM-DD, to: YYYY-MM-DD }   # 可选；框定所有 usage_count，单条目可覆盖
# --- trust (OKF §5.2) ---
generated: { by: "llm-wiki/<model>", at: 2026-04-05T10:00:00Z }   # at 可选
verified:
  - { by: "human:david", at: 2026-04-08T09:00:00Z }    # 可选；裸 mapping 视为单元素列表
# --- lifecycle (OKF §5.4-5.5) ---
status: stable                # draft | stable（缺省）| deprecated
stale_after: 2027-01-01       # 可选；绝对日期，今天 >= 此日即视为 stale
---
```

> 注意：`generated.at`、`verified[].at` 是 ISO 8601 datetime，默认用完整形式 `YYYY-MM-DDThh:mm:ssZ`（如 `2026-04-05T10:00:00Z`）；仅在确不知具体时间时才可退化为日期 `YYYY-MM-DD`。`stale_after` 与 `usage_window` 用 `YYYY-MM-DD` 日期即可。actor 遵循约定（见 SKILL.md "Actor 约定"）。

### 实体页模板

```markdown
---
type: Entity
title: "人物姓名"
description: "一句话摘要"
resource: https://...
entity_type: person           # person | tool | organization
domain: []
tags: []
sources:
  - { id: Source Summary 的 id, resource: ../sources/karpathy-llm-wiki-gist.md, title: "LLM Wiki Gist" }
generated: { by: "llm-wiki/<model>", at: ... }
status: stable
---
# 人物姓名
## 基本信息
- 角色/职位：
- 隶属组织：
## 关键论断与观点
## 相关实体
```

### 概念页模板

```markdown
---
type: Concept
title: "概念名称"
description: "一句话摘要"
tags: []
sources:
  - { id: ..., resource: ../sources/xxx.md, title: "..." }
generated: { by: "llm-wiki/<model>", at: ... }
status: stable
---
# 概念名称
## 定义
## 相关概念
## 支撑证据
## 未解问题
```

### 来源摘要页模板

```markdown
---
type: Source Summary
title: "来源标题"
description: "一句话摘要"
resource: https://原始链接
tags: []
sources:
  - id: self                  # 该摘要页自身的稳定 id，供其它页面 footnote 归因
    resource: ../raw/xxx.md   # 相对路径指向不可变原始资料
    title: "来源标题"
    author: "human:原作者或 curator"
    last_modified: YYYY-MM-DD
generated: { by: "llm-wiki/<model>", at: ... }
status: stable
---
# 来源标题
## 关键要点
## 提及的实体
## 引入的概念
## 与已有知识的关联
```

### 对比分析页模板

```markdown
---
type: Comparison
title: "X vs Y 对比"
description: "一句话摘要"
items_compared: [X, Y]
tags: []
sources:
  - { id: ..., resource: ../sources/xxx.md, title: "..." }
generated: { by: "llm-wiki/<model>", at: ... }
status: stable
---
# X vs Y 对比
## 比较维度
## 发现
## 建议
```

### 工作流/操作指南页模板

```markdown
---
type: Playbook
title: "场景名称"
description: "一句话摘要"
tags: []
sources:
  - { id: ..., resource: ../sources/xxx.md, title: "..." }
generated: { by: "llm-wiki/<model>", at: ... }
status: draft
---
# 场景名称
## 触发条件
## 步骤
## 注意事项
```

---

## 原则四：每个任务产出双重输出

每个任务应产出两个输出：
1. **直接产出** — 人类要求的回答/产出物（进 `output/`）
2. **wiki 更新** — 将新生成的知识更新回 wiki

如果不在 schema 中明确要求这一点，有价值的分析就会消失在聊天记录中。在 schema 中加入类似以下的规则：

> 每次完成分析或回答问题后，评估是否有值得持久化的知识。如果有，将其作为新页面或对现有页面的更新写入 wiki（并同步更新 `index.md` 与 `log.md`）。

---

## 原则五：跨域标签

如果知识跨越多个项目、客户或研究领域，从第一天起就在页面 frontmatter 中添加 `domain` 标签（producer 扩展键，OKF 允许）：

```yaml
domain: [project-alpha, machine-learning]
```

出现在多个领域的共享实体（人物、组织、概念）会成为知识图谱中最有价值的节点。后期补加这个标签会非常痛苦——需要回溯所有已有页面逐一判断。

---

## 原则六：强制来源引用（per-claim attribution）

要求 wiki 内容中必须附带来源引用。LLM 可以在不标注来源的情况下进行综合，而你不仔细看很难发现这种"无中生有"。引用采用 **footnote keyed to `sources[].id`**（OKF §5.1）：

```markdown
Karpathy 提出 wiki 的核心是"编译"而非"检索"。[^karpathy-gist]

[^karpathy-gist]: LLM Wiki Gist
```

规则：
- 每个事实性论断用 footnote 指回 `sources[].id`；footnote label **必须等于** 某个 `sources[].id`，不能自造。
- 综合性结论必须在 `sources` 中列出所有参与综合的来源。
- **为什么不用位置索引**：agent 会不断重写这些文档，列表重排后 `sources[0]` 会静默错配；稳定的 `id` 在重排后依然可靠（OKF §5.1）。
- 人类是主编——定期抽查引用的准确性，而非逐字审阅。

---

## Frontmatter 规范（OKF 五大家族）

### 必填

- **`type`**：概念类型，非空字符串。只有它必填（OKF §11）。

### 推荐（每页尽量维护）

- **`title`**：可读显示名；缺省可由文件名推导。
- **`description`**：单句摘要，**index.md 的条目描述由此派生**——维护页面时同步更新它，索引就不会失真。
- **`resource`**：底层资产的规范 URI；描述抽象概念时可省略。
- **`tags`**：跨维度分类。

### Provenance（§5.1）

- `sources[]`：每个条目内 `resource` 必填，可指向 URL、bundle 内路径（如 `/concepts/xxx.md`）、相对路径或作用域描述符；本地资源优先使用相对路径以兼容通用工具。`id` 为稳定键，供 footnote 归因（正文引用时建议有）；`author`、`usage_count`、`last_modified` 是**来源可信信号**（记录信号不记录评分，可信度由消费者推断）。
- **来源分层/递归**：当 `sources[].resource` 指向 bundle 内另一个概念时，该概念自身也有 `sources`——派生边已存在于链接图中，消费者可递归读取、让可信度传播。外部叶子来源只携带其自身信号。
- **`usage_count` 的解读**（OKF §5.1）：它是粗糙信号，只适合"存活 vs 消亡"、数量级比较、以及该来源自身随时间的变化——不能当作跨类型精确排名（定时任务的执行次数与人工 Dashboard 浏览权值不同）。写入时可参考来源的实际使用情况。
- `usage_window`：与 `sources` 平级，框定 `usage_count` 的时间范围；单个条目可携带自己的 `usage_window` 覆盖共享值。

### Trust（§5.2–5.3）

- `generated: { by, at }`：记录内容如何产生；`by` 出现时必填，actor 遵循约定。`generated.at` = 内容最近一次有意义修改的 ISO 8601 时间戳（可含时间）。
- `verified`: 验证事件列表（人类签字 + 进程校验可各记一条）；裸 mapping 视为单元素列表。信任层级由它推导：无 ⇒ unverified；仅非 `human:` ⇒ machine-confirmed；含 `human:<id>` ⇒ human-reviewed。
- `generated.at` 与 `verified[].at` **相互独立**：内容可未经重新确认而变更，事实也可在无重新生成时被再次确认。
- **不使用 `confidence` 字段**——OKF 记录信号而非评分，可信度由消费者从信号推导。

### Lifecycle（§5.4–5.5）

- `status`: `draft`（未评审）/ `stable`（缺省）/ `deprecated`（保留链接和历史）。
- `stale_after`: 绝对日期（`YYYY-MM-DD`），内容在该日当天及之后视为 stale。用绝对日期而非相对 TTL，使过期判断不依赖读取时刻。

### 扩展

OKF 允许 producer 自定义任意附加键（如 `domain`、`entity_type`、`items_compared`）。消费者不得拒绝未知键，round-trip 时应保留。

---

## 计算与验证（OKF §10，可选）

知识库中若出现"必须按既定方式计算某个值"的概念（如指标、数据产出），遵循 OKF §10 建模为独立的 `type: Attested Computation` 概念，其它页面用普通链接引用它，不内联计算：

```markdown
---
type: Attested Computation
title: "Revenue for fiscal year"
description: "Recognized revenue for a fiscal year, per Finance's definition."
status: stable
runtime: bigquery                                   # REQUIRED — 决定 parameters 的绑定语义
parameters:
  - { name: year, type: integer, required: true }
computation: references/computations/revenue.sql    # 可选；缺省用正文 # Computation 围栏块
executor:
  resource: references/skills/run-on-bq.md
  receipt: [job_id, executed_sql, result]           # 运行必须返回的证据字段
attester:
  resource: references/attesters/revenue.py         # 确定性代码，吃 receipt、出 verdict
generated: { by: "llm-wiki/<model>", at: ... }
verified: { by: "human:david", at: ... }
stale_after: 2027-01-01
sources:
  - { id: rev-policy, resource: https://wiki.acme/finance/revenue-recognition, title: "Revenue recognition policy" }
---

# Computation
...计算代码...
```

要点：
- `runtime` 决定 `parameters` 是 SQL 绑定变量、dbt var 还是 Python 参数。
- 计算的内容放在正文 `# Computation` 围栏块，或由 `computation` 指向文件；agent 只能为声明的 `parameters` 提供**值**，不得改写计算本身。
- `verified`（定义是否符合策略、文档级、慢）与 attestation（单次运行是否符合、调用级、不存 bundle）是两回事，二者并存。
- 知识库场景下这类概念通常不出现，给出此节仅为完整覆盖 OKF 规范。

---

## OKF 合规自检清单（Conformance，OKF §11）

生成或修改任何 wiki 页面后，用此清单做最终自检。**MUST** 项违反即不合规；**SHOULD** 项是规范建议的软性约束；最后的容忍清单是 lint 与查询时必须遵守的——不得据此误报。

**MUST（缺一即不合规）**：
1. 每个非保留 `.md` 概念文件包含可解析的 YAML frontmatter 块（文件开头 `---` 与结尾 `---` 各自独占一行）。
2. 每个 frontmatter 含**非空** `type` 字段。
3. 保留文件名只用其约定用途：`index.md` 仅作文档列举（§8）、`log.md` 仅作更新历史（§9），二者 MUST NOT 用作概念文档（§3.1）。子目录 `index.md` 不得带 frontmatter；仅 bundle root `index.md` 可带 `okf_version`（§12）。
4. 若使用五大家族：`sources[].resource` 在条目内必填（§5.1）；`generated` 出现时必含 `by`（§5.2）；`type: Attested Computation` 必含 `runtime`（§10.2）。
5. 身份字段（`sources[].author`、`generated.by`、`verified[].by`）遵循 actor 约定；人类撰写或确认的内容**必须**用 `human:<id>`（§7）。
6. 正文 footnote 的 label 必须等于某个 `sources[].id`，不能自造（§5.1 per-claim attribution）。

**SHOULD（建议遵守）**：
7. `verified` 裸 mapping 视为单元素列表，无需改写成列表（§5.2）。
8. `status` 缺省即 `stable`（§5.4）；`stale_after` 用绝对日期 `YYYY-MM-DD`，使过期判断是纯日期比较（§5.5）。
9. 优先维护 `title` / `description` / `resource` / `tags`；空 `description` 会让 index 条目无摘要可用（§4.1、§8）。
10. 站内 Markdown 交叉链接使用相对于当前页面的路径，且不得以 `/` 开头；移动页面后同步更新链接。
11. 信任层级与 stale 判断只从规范定义的字段推导，不使用 `confidence` 等评分字段（§5、§5.3）。

**不得拒收 / 不得误报**（§11 后半段）：
- 缺少可选 frontmatter 字段的页面
- 未知 `type` 值（降级为通用概念处理）
- 未知附加 frontmatter 键（round-trip 时保留）
- 断链（可能是"尚未写出的知识"）
- 缺少 `index.md`

---

## Schema 必须包含的操作规则

以下规则必须写入项目的 Schema 配置文件中，否则 LLM 在日常操作中不会遵守。Schema 是 LLM 操作 wiki 时的唯一行为依据——skill 只在初始化时起作用，之后全靠 Schema 驱动。

### 规则一：查询前必须先读索引

在 Schema 中明确写入以下规则（可根据项目实际措辞调整）：

> **查询规则**：回答任何关于知识库的问题时，第一步必须读取 `wiki/index.md`，根据索引中的摘要和标签定位相关页面，然后再深入阅读具体页面。禁止跳过索引直接搜索文件内容。

**为什么需要这条规则**：LLM 的默认行为是用搜索工具（grep/glob）直接查找文件内容，这样虽然快但会绕过 wiki 精心维护的分类和摘要体系，导致索引形同虚设。强制先读索引可以确保 LLM 利用已编译的知识结构，而不是每次都退化为原始文件搜索。

### 规则二：产出物路由——wiki/ vs output/

在 Schema 中明确写入以下规则：

> **产出物路由**：
> - `wiki/` 只存放持久化知识页面（实体、概念、来源摘要、对比分析、工作流）——这些是会被反复引用和增量更新的内容。
> - `output/` 存放一次性交付物（报告、幻灯片、图表、导出文件）——这些是为特定请求生成的最终产品，不参与 wiki 的交叉引用和增量更新。
> - 判断标准：如果一份内容会被未来的查询引用或在未来摄入时更新，放 `wiki/`；如果是回答某个具体请求的最终产出（如"帮我生成一份 PPT"），放 `output/`。

**为什么需要这条规则**：没有这条规则时，LLM 要么把所有产出都放进 `wiki/`（导致 wiki 中混入大量一次性报告），要么直接输出在对话中（导致交付物丢失）。明确的路由规则让目录职责清晰。

### Schema 中的写法示例

```markdown
## 操作规则

1. 查询时，第一步读取 `wiki/index.md` 定位相关页面，禁止跳过索引直接搜索。
2. 持久化知识（实体、概念、来源摘要、对比分析、工作流）写入 `wiki/`，frontmatter 遵循 OKF 五大家族，正文引用用 footnote keyed to `sources[].id`。
3. 一次性交付物（报告、幻灯片、导出文件）写入 `output/`。
4. 每次查询产出交付物后，评估是否有值得持久化的知识，如有则同步更新 wiki（页面 + index.md + log.md）。
5. 记录身份使用 actor 约定：agent 用 `llm-wiki/<model>`，人用 `human:<id>`，进程用 `process:<id>`。
6. 信任层级由 `verified` 推导（无=unverified / 仅非 human:=machine-confirmed / 含 human:=human-reviewed），不使用 confidence。
```

---

## 附录：最小完整示例（一个合规 bundle 的诞生）

从一份原始资料到一组 OKF 合规产出的完整流程。`wiki/` = bundle root，示例内容为演示占位。

### 1. 原始资料（不可变，位于 bundle 外 `raw/`）

`raw/karpathy-llm-wiki-gist.md` 是来源，摘要页通过 `sources[].resource` 以相对路径引用它。

### 2. 来源摘要页 `wiki/sources/karpathy-llm-wiki-gist.md`

五大家族全部落位，footnote label 命中 `sources[].id`：

```markdown
---
type: Source Summary
title: "LLM Wiki Gist"
description: "Karpathy 提出的 LLM 驱动 wiki 方法论：将知识增量编译为互相链接的 Markdown 文档，而非逐次检索。"
resource: https://gist.github.com/xxx/llm-wiki
tags: [llm, wiki, methodology]
sources:
  - id: self                    # 该摘要页的稳定 id，供其它页面 footnote 归因
    resource: ../raw/karpathy-llm-wiki-gist.md
    title: "LLM Wiki Gist"
    author: "human:akarpathy"
    last_modified: 2026-04-03
  - id: live
    resource: https://gist.github.com/xxx/llm-wiki
    title: "线上 gist"
    usage_count: 12000
    last_modified: 2026-04-03
usage_window: { from: 2026-01-01, to: 2026-06-30 }
generated: { by: "llm-wiki/claude-sonnet-4", at: 2026-04-05T10:00:00Z }
verified: { by: "human:david", at: 2026-04-08T12:00:00Z }   # 裸 mapping = 单元素列表
status: stable
stale_after: 2027-01-01
---

# LLM Wiki Gist

## 关键要点
- Wiki 的核心是"编译"而非"检索"。[^self]
- 每类来源应有专属的提取流程与摘要详细程度。[^self]
- 每个任务产出双重输出：直接交付物 + 可持久化的 wiki 知识。[^self]

## 引入的概念
参见 [LLM Wiki](../concepts/llm-wiki.md)。

## 与已有知识的关联
与 [RAG](../concepts/rag.md) 形成对比，见 [LLM Wiki vs RAG](../comparisons/rag-vs-llm-wiki.md)。

[^self]: LLM Wiki Gist
```

### 3. 实体页 `wiki/entities/karpathy.md`（链接摘要页，footnote 归因）

```markdown
---
type: Entity
title: "Andrej Karpathy"
description: "AI 研究者，LLM Wiki 方法论提出者。"
entity_type: person
resource: https://karpathy.ai
tags: [ai, research]
sources:
  - { id: gist-summary, resource: ../sources/karpathy-llm-wiki-gist.md, title: "LLM Wiki Gist" }
generated: { by: "llm-wiki/claude-sonnet-4", at: 2026-04-05T10:00:00Z }
status: stable
---

# Andrej Karpathy

## 基本信息
- 角色：AI 研究者

## 关键论断与观点
- 主张 wiki 的核心是增量编译知识。[^gist-summary]

[^gist-summary]: LLM Wiki Gist
```

### 4. 索引与日志（保留文件名，各扮演 §8 / §9 的角色）

`wiki/index.md`（bundle root，唯一允许带 frontmatter 的 index；条目用相对 URL，摘要取自页面 `description`）：

```markdown
---
okf_version: "0.2"
---
# Wiki 索引

## 实体
* [Karpathy](entities/karpathy.md) - AI 研究者，LLM Wiki 方法论提出者 `#person`

## 来源摘要
* [LLM Wiki Gist](sources/karpathy-llm-wiki-gist.md) - Karpathy 的 LLM Wiki 方法论 `#primary-source`
```

`wiki/log.md`（日期头 ISO `YYYY-MM-DD`，最新在前）：

```markdown
# 操作日志

## 2026-04-05
* **ingest**: 摄入 Karpathy LLM Wiki Gist
* **update**: 创建 wiki/sources/karpathy-llm-wiki-gist.md、wiki/entities/karpathy.md、wiki/index.md
```

> **检查点**：摘要页 `description` 被 index 条目原样引用；实体页与摘要页的 `verified` / `stale_after` 各自独立，信任层级与时效逐页推导；所有事实性论断都有 footnote 且 label 命中 `sources[].id`。以此示例为基准核对实际产出，即可确保满足 OKF v0.2 合规要求。
