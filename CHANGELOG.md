# Changelog

本文件记录 `hb-automation-design` 的重要变更。

## 2026-06-11

本轮包含 Claude 与 Codex 连续完成的结构优化、规则去重和 hac 0.26 对齐。

### 阶段职责拆分

- 将原先混在 `design-principles.md` 中的“方案设计”和“实现避坑”拆开：
  - `design-principles.md`：只在出方案前读取，负责业务取舍、交互选择、循环、计算字段、API 和失败反馈设计。
  - `implementation-pitfalls.md`：只在方案确认后、构建 JSON 时按需读取，集中 verified 实现经验。
- 将 `type-specific-tips.md` 收敛并更名为 `type-design-tips.md`，只保留各自动化类型的方案设计提醒；payload、字段映射、编辑链接等实现内容移入 `implementation-pitfalls.md`。
- 工作流明确为“判类型 → 读方案设计 → 方案自检并输出确认 → 读实现避坑 → 切到 huoban-automation 落地”。

### 方案自检演进

- 新增出方案前的内部自检机制，要求每个命中项都能指认到方案中的具体节点。
- 自检由最初的输出小节逐步收敛为纯内部动作，方案中不再出现“自检”段落。
- 修复 `SKILL.md`、设计原则和输出格式之间关于“是否输出自检”的残留矛盾。
- 最终取消独立的 `preflight-checklist.md`：检查动作保留在 `SKILL.md`，检查依据直接复用设计原则和当前类型提醒，避免重复加载。
- 将独立清单中仅有的补充判断归回设计原则：先判断是否真的需要自动化；能批量处理时不拆成循环。

### 方案设计强化

- 强制查真实表结构后再输出字段方案、节点方案或 JSON 草稿。
- 强制先输出“方案确认”，待用户确认后再落地配置。
- 强化交互节点优先原则：技术配置难度不能成为降级为后台静默处理的理由。
- 创建带明细的主业务单据时，默认使用“打开数据创建页面”预填主表和子表；子表必须有前序数组来源。
- 节点结构和顺序先以流程图或等价顺序结构让用户确认，再补字段映射和提交配置。

### 去重与 hac 文档对齐

- 删除 `SKILL.md` 与设计原则之间重复的强制规则，保留单一权威入口。
- 删除重复的自动化类型职责边界表；类型选择以 `workflow/intent-routing` 和 `type/<类型>` 为准。
- 删除 hac 已覆盖的节点字段、字段类型、ScriptCarrier、condition、变量系统和 validate 错误码说明，统一指向 `hac automation docs show <key>`。
- 将 item_trigger 的精确更新、`return_field_id`、目标数据引用、relation 定位、旧公式迁移和循环取舍等纯实现写法移入实现避坑。

### Verified 实现经验整理

- 集中整理 `from_relation_field` 字段归属判断、`scripts.vars` 保序、category 赋值与比较差异、筛选器精确匹配等经验。
- 补充 call / on_call 参数字段 ID 被编辑器重建后的回读与同步规则。
- 强制 create/update 后执行 `get` 并验证编辑器正常回显。
- 整理交互创建页使用触发数据预填、子表循环元素和 `subtable_field_id` 的 verified 写法。
- 记录 hac 0.26.1 token 模式写操作的认证与 PaaS `curl` 提交绕行方案。

### 相关提交

`de1cc6b` → `a0286b3` → `b4ac9af` → `d277cd2` → `3009894` → `b6ac432` → `4d03cac` → `1c1a4c8` → `5edb1dc` → `4b14033`

## 2026-06-01

### 初始版本

- 建立伙伴云自动化设计层 skill。
- 提供方案设计原则、类型提醒和“方案确认 / 分析结果”输出格式。
- 明确与 hac `huoban-automation` 的设计层 / 执行层分工。
