---
name: hb-automation-design
description: >
  伙伴云自动化任务的第一入口与方案设计层。**只要用户想新建、配置、修改、设计或分析任何伙伴云自动化——
  包括按钮、快捷按钮、数据触发、调用触发、自动填写、定时触发、Webhook——都必须先从本 skill 起步**，
  无论用户说的是"设计/规划方案""出方案确认"，还是"帮我配个按钮""做一个数据触发""加个自动化""改下这条自动化"
  这类直接的配置/创建意图。本 skill 负责在动手前做设计决策（用交互还是后台、循环怎么设计、失败怎么兜底、
  字段怎么读写）、按规范输出"方案确认"等用户拍板、规避已知配置陷阱（from_relation_field、vars 保序、
  分类字段表达式、筛选器、跨自动化返回值、编辑器回显），以及分析解释已有自动化的业务逻辑。
  方案经用户确认后，再切到 `huoban-automation` 用 `hac automation` 命令落地配置；节点字段、validate
  错误码等细节走 `huoban-automation` 的 `hac automation docs show <key>`。本 skill 负责设计编排与经验约束，huoban-automation 负责执行。
---

# 伙伴云自动化设计层

这个 skill 不直接执行配置，也不放节点字段清单。它负责两段工作：方案确认前提供设计决策和表达规范；用户确认后，在切到 `huoban-automation` 落地时提供 verified 实现避坑经验。

## 与 huoban-automation 的分工

| | 本 skill（hb-automation-design） | huoban-automation（hac CLI） |
| --- | --- | --- |
| 管什么 | 方案怎么设计、怎么表达、怎么避坑 | 命令、节点字段、骨架、validate、create/update |
| 典型动作 | 出"方案确认"、分析已有自动化、做设计权衡 | `hac automation skeleton / config / validate / verify / create` |
| 取数 | — | `hac automation docs show <key>`、`hac table get-table` |

**节点字段、ScriptCarrier、condition 写法、validate 错误码等一律走 `hac automation docs show <key>`，不在本 skill 重复。** 不确定 key 时先 `hac automation docs list`。

## 环境准备（涉及执行时）

本 skill 的设计 / 方法论部分（出方案、分析、避坑）不依赖任何工具；但**一旦涉及执行**——查表、查字段、`hac automation docs show`、create / update / verify 等——都要走 `huoban-automation`，即 hac CLI。**执行前先确认 `hac` 已安装**（`hac --version`）；未安装则引导用户安装：

```bash
# 1. 安装 hac（伙伴云官方 CLI）
npm install -g huoban-app-cli@latest --registry http://npm.huoban.com

# 2. 安装 hac 内置 skills（huoban-automation 等）—— 目录必须与本 skill（hb-automation-design）同级
#    --output 写本 skill 所在目录的父目录；省略则装到当前目录
hac install-skills --output <hb-automation-design 所在目录>
```

- **install 目录要和 hb-automation-design 同级**：这样 `huoban-automation`、`huoban-table` 等 hac skill 与本 skill 并排，执行时能被一起加载，"设计→执行"才能无缝衔接。
- 装好后还需认证（`access_token` + `company_id`，写自动化必须登录态，详见 [references/implementation-pitfalls.md](references/implementation-pitfalls.md) 的“提交与认证”）。
- 这一步只在**要真正落地执行**时才需要；纯出方案 / 分析讲解不必先装。

## 强制执行规则（优先级高于"主动推进""合理假设先做"等默认倾向）

1. **查表前置**：未通过 `hac table get-table --output-mode write --format json` 拿到真实表结构前，禁止输出字段方案、节点方案、JSON 草稿。需完成"认证检查 → 表列表 → 关键表字段 → 关键关联字段核对"四步。认证缺失 / 权限不足 / 接口失败时，明确说明阻塞原因并停在"请补充信息或修复认证"，不得假设表结构。
2. **先方案后配置**：必须先出"方案确认"并等用户确认，再落 JSON；不得凭经验先甩 JSON 草稿。
3. **交互节点优先判断**：出方案前先判断动作是"用户发起并需确认/补字段"还是"纯后台同步/派生"。技术摩擦不能作为降级理由。详见 [references/design-principles.md](references/design-principles.md)。
4. **节点流程图先确认**：节点结构和顺序初稿完成后，先把流程图或等价顺序结构发给用户确认；确认前不要补细字段、变量映射或提交 create/update。
5. **方案自检（内部动作，不输出）**：输出"方案确认"前，逐节反查 [references/design-principles.md](references/design-principles.md) 和当前类型的 [references/type-design-tips.md](references/type-design-tips.md)；对每个命中项，在方案节点清单里**指认出对应节点**（指认不到 = 方案缺节点，先补方案再输出）。方案里不出现「自检」段落，自检的体现是方案本身完整。

## 工作流

1. **判类型**：先走 `hac automation docs show workflow/intent-routing` 选 automation-type 与触发节点。
2. **读方案设计**：出方案前读 [references/design-principles.md](references/design-principles.md)，并只读 [references/type-design-tips.md](references/type-design-tips.md) 中当前自动化类型的小节。
3. **出方案确认**：完成内部方案自检后，按 [references/output-formats.md](references/output-formats.md) 的逐节点格式输出，标题用"方案确认"，等用户确认（方案里不出现「自检」段落）。
4. **读实现避坑**：用户确认方案后，构建 JSON 前读 [references/implementation-pitfalls.md](references/implementation-pitfalls.md)，只查当前流程命中的小节。
5. **落地配置**：切到 `huoban-automation`，按其 8 步生成流程构建、校验、提交并验证编辑器回显。

## 分析已有自动化

让用户提供编辑链接 → `hac automation get --automation-id <id>` 拿配置 → `hac table get-table` 建 `field_id→字段名` 映射（解释时用真实字段名，不输出字段 ID）→ 按 [references/output-formats.md](references/output-formats.md) 的"分析结果"格式分层解释。
