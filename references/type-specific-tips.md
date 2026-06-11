# 各自动化类型的高频提醒与职责边界

落地某一类型的配置前，对照本类型的避坑清单。基础结构、节点字段走 `hac automation docs show <key>`，本文件只放各类型的设计边界与 verified 陷阱。

## 类型职责边界（避免选错入口）

| 类型 | 负责 | 不负责（应切到） |
| --- | --- | --- |
| button | 按钮入口、显示位置、权限、显示规则、按钮里接入 `on_call` | 独立 `call` 本体（→ call）、后台异步处理（→ item_trigger） |
| call | 独立 `call` 本体、`params` / `tips` / `button_name` / `output_variables` 及返回契约 | 按钮显示位置/权限/点击入口（→ button） |
| item_trigger | 后台异步触发、`trigger_action` / `trigger_field_ids` / 权限组 / 并发 / 异步边界 | 需用户确认或页面交互（→ button / autofill） |
| autofill | 填写页即时触发、`input_after` / `trigger_position` / `trigger_fields` / `backfill_trigger` 回填 | 保存后异步（→ item_trigger）、用户主动点击（→ button） |

## 按钮（button）

- 能用按钮显隐条件拦截的前置校验，优先放在按钮入口 `condition` 中：条件不满足时隐藏或灰显按钮。
- 入口 `condition` 已限制的触发条件（状态、数量、权限等），后续流程**不要再重复**写"条件分支 + 提前退出"，避免入口校验和流程校验写两遍。
- 只有点击后才知道的校验结果，才在后续流程中用"条件分支 + 提前退出"；"提前退出"节点前必须有明确的条件分支，不要光秃秃出现在流程开头。
- 如果后台创建主业务单据，必须在方案里说明为什么不用交互节点（见 design-principles 第一区"后台创建准入清单"）。
- `on_call` 是共享节点；被调用的 `call` 本体契约改用 call 类型设计。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/button/<button_id>/edit`

## 调用触发（call）

- 先设计调用契约（触发参数、输出变量、`tips`、`button_name`），再设计节点流程。
- `params` 和 `output_variables` 的字段类型判断，先套 `data_type` 对照（`hac automation docs show field/field-type-matrix`）。
- 输出变量需要中途改写时，用 `variable_set` 以 `op_type: "update"` 回写，不要只定义不更新。
- 子流程内部读输出变量用 `{输出变量.xxx}`；调用方读返回值用 `{#N|输出变量.xxx}`。
- 只要还有无默认值参数未传，执行时就会弹窗补参。
- `call` 是可复用子流程入口，不是按钮节点；被其他自动化调用的 `call`，不要再嵌入 `on_call`。
- 参数映射避坑见 design-principles 第二区"跨自动化返回值提醒"——relation 参数经编辑器保存后 `field_id` 可能被重建。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/call/<automation_id>/edit`

## 数据触发（item_trigger）

- 数据触发是异步后台执行，**不能用任何交互节点**。需求依赖用户确认或补参数时，通常不适合数据触发。
- `trigger_action`（三个布尔位）、`trigger_field_ids`、原值变量（`original: true`）的结构语义已由 `hac automation docs show node/item_trigger` 权威覆盖，以它为准。
- `{数据创建}` / `{数据修改}` / `{数据删除}` 只属于数据触发。
- 要覆盖导入 / OpenAPI / 自动化写入时，优先评估是否应选"全部权限组"。
- 数据触发支持最多 15 层涟漪触发；同表修改链有防循环限制，不能想当然指望无限级联。
- **实现期写法**（单条精确更新 / `return_field_id` 形态 B / `{目标数据}` / relation 字段定位 / `mode:"input"` vs `script` / 旧公式迁移 / `loop_by_target` 取舍）已移到 design-principles 第二区「数据触发（item_trigger）实现避坑」，构建 JSON 时回查那里。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/item_trigger/<automation_id>/edit`

## 自动填写（autofill）

- 自动填写不是保存后后台触发，而是数据填写过程中在页面内即时执行：在"填写结束后、保存提交前"读当前编辑上下文，复用节点体系计算，再经 `backfill_trigger` 回填到当前页面。
- 同一字段被"填写时触发"的自动填写监听数量不能超过 3 个；同一字段多个规则时，只执行第一个满足权限组与触发条件的规则。
- 自动填写产生的字段值变化不会再次触发自动填写。
- `backfill_trigger` 是交互节点；分析日志时关注 `interact → continue → end` 这条链。
- 创建页支持对子表做覆盖式回填；编辑页不支持按同样方式覆盖现有子表。
- 单字段编辑 `cell_update` 与完整编辑页 `item_update` 的 payload 语义不同，分析时不要混用。
- 子表字段触发时，不要只看 `field_id`；必须连同 `subtable_field_id` 一起判断它属于主表上的哪个子表。
- 若接口返回里出现 `button_id` 一类历史兼容字段，不要误判为按钮自动化；以 `type: "autofill"` 为准。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/autofill/<automation_id>/edit`
