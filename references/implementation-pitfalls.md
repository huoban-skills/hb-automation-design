# 伙伴云自动化实现避坑

> 本文件只在**方案已确认、开始构建 JSON 后**读取，集中记录 hac 文档尚未覆盖的 verified 实现经验。方案设计阶段不要加载本文件。

基础结构规则一律以 `huoban-automation` 的 `hac automation docs show <key>` 为权威源：

- 字段类型速查 → `hac automation docs show field/field-type-matrix`
- ScriptCarrier → `hac automation docs show scripting/script-carrier`
- 变量系统 → `hac automation docs show scripting/variables`
- field_params → `hac automation docs show field/field-params`
- condition → `hac automation docs show field/README`

## 通用结构完整性

- 即使没有条件，也要保留完整 `condition` 结构。`filter.and` 为空数组比写 `null` 更稳。
- 节点 JSON 要尽量保持结构完整，不要省略必要壳层。
- 新增字段赋值时，先从最小可用字段集开始，再逐步扩展；子表字段不存在或不可见时不要强行赋值，先只赋 1 到 2 个核心字段，成功后再逐步追加。

## API 调用实现

- 后续节点不依赖返回值时可用“不解析返回值”（已观察到 `resp_body_format: 1`）；需要判断成功失败或提取字段时必须配置解析。
- 联调期若接口返回不稳定，优先补预期返回结构或 `resp_body_mock`，保证后续节点可以先搭起来。
- 返回文本乱码时，优先检查返回编码配置，再决定是否手动拆 `raw_body`。
- 某些 `application/json` 接口仍要求手工拼接 JSON 文本并参与签名，这时保留文本 body；已有真实样本使用 `req_text_body + req_body_format: 6`。
- 按 JSON 解析时常读 `{#N|body.xxx}`；按文本解析时后续节点读取整个 `{#N|body}`，不能继续假定存在 `body.xxx`。
- **GET（及其他无 body 的请求）必须设为“无请求体”**：`hac automation skeleton --with-node api_call` 默认给的是 POST 取向的 `req_body_format: 6` + `req_text_body` + `Content-Type: application/json`。配 GET 时主动去掉请求体和对应的 `Content-Type`。

## 字段归属判断（from_relation_field）

| 场景 | 判断方法 | 结论 |
|---|---|---|
| `from_relation_field` 为空 | 字段直接定义在当前表 | 这是本表原生字段 |
| `from_relation_field.field_id = <某关联字段>` | 字段通过该关联字段从别的表带出 | 这是关联附加字段，不是本表原生字段 |

例：如果 `发货明细.待发货数量` 的 `from_relation_field.field_id = 发货明细.订单明细`，说明这个值本质来自 `订单明细`，不能直接当成“发货明细自身剩余量”使用。

## 脚本变量保序不去重

- `scripts.vars` 数组必须按表达式 `code` 中变量出现的顺序逐项登记，**不能按 `field_id` 去重**。
- 如果同一个字段在 `code` 里出现了 2 次，`vars` 里也必须登记 2 次，位置与 `code` 中的出现顺序一一对应。
- 去重会导致编辑器回显异常或运行时变量绑定错位。verified 样本（`automation_id: 7500000002735670`）确认了“同字段重复登记”的写法。

## 分类字段表达式

- 给 category 字段赋值，或在 `IF` 等函数分支里返回选项常量时，表达式用裸 `{C:表名.字段名.选项名}`，不要默认包 `[]`。
- 不要按 `field_params.is_array` 机械判断是否包 `[]`。即使 `is_array: true`，单选 category 的赋值仍可能是裸 `{C:...}`。
- 只有字段配置上真的多选时，赋值表达式里才可能出现数组字面量；目前没有 verified 正例，不要仅凭 `is_array:true` 反推。
- `condition.filter.query.eq` 对 category 做值比较时，verified 样本使用 `[{C:...}]`。赋值用裸写，比较用方括号，两套规则不要混。

## 数组建模

- `Name/Value` 这类键值对数组，通常更适合先转中间变量，再按固定字段回填。
- `Items` 这类天然明细数组，通常更适合直接 `loop_by_target`。
- 最终目标是逐条创建明细表时，优先直接循环数组并在循环体里 `item_create`。

## 数据触发（item_trigger）

> 异步边界、权限组、涟漪层级等设计判断在 [type-design-tips.md](type-design-tips.md) 的 item_trigger 段。

- 单条精确更新优先 `item_id eq {#N|获取的数据.数据ID}`，不要用业务字段组合重新定位同一条记录。
- `item_find_multi.return_field_id` 有值时返回字段值集合，不是记录集合：下游用 `mode:"input"`，`code:"{#N|获取的数据}"`，`vars.data_type` 写字段真实类型、`table_id` 写 `0`。
- `item_update` 读取当前被修改的目标记录字段用 `{目标数据.字段名}`，`vars` 用 `scope:"target_item" + type:"item"`，不要额外加 `item_find`。
- 更新触发记录某 relation 字段指向的记录时，`condition.filter.target` 用 `field_id:"item_id" + table_id:<目标表ID>`，不带 `belong_info`，值来源传 `{触发的数据.<关系字段>}`。
- `field_params` 直接引用触发数据、前序结果或变量时优先 `mode:"input"`；只有需要函数、拼接、判断或单位转换时才用 `mode:"script"`。
- 旧版 workflow 的 `ITEM(...)` / `FIELDS(...)` / `OBJ_GET(JSON2OBJ(...))` / `JSONPATH(...)` 不要原样迁入 `script`；改为 `item_find` / `item_find_multi`，下游消费节点结果。
- 只是把查询结果整体写入字段时，用 `item_find_multi + field_params`；只有逐条创建或调用接口才用 `loop_by_target`。

## 类型专属实现提醒

### 按钮（button）

- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/button/<button_id>/edit`

### 调用触发（call）

- `params` 和 `output_variables` 的字段类型先对照 `hac automation docs show field/field-type-matrix`。
- 输出变量需要中途改写时，用 `variable_set` 的 `op_type:"update"` 回写。
- 子流程内部读输出变量用 `{输出变量.xxx}`；调用方读返回值用 `{#N|输出变量.xxx}`。
- relation 参数经编辑器保存后 `field_id` 可能被重建，参数映射同时检查下方“跨自动化返回值”。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/call/<automation_id>/edit`

### 数据触发（item_trigger）

- `trigger_action`、`trigger_field_ids`、原值变量（`original:true`）的结构以 `hac automation docs show node/item_trigger` 为准。
- `{数据创建}` / `{数据修改}` / `{数据删除}` 只属于数据触发。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/item_trigger/<automation_id>/edit`

### 自动填写（autofill）

- `backfill_trigger` 是交互节点；分析日志时关注 `interact → continue → end` 链路。
- 单字段编辑 `cell_update` 与完整编辑页 `item_update` 的 payload 语义不同。
- 子表字段触发时，必须结合 `field_id + subtable_field_id` 判断所属子表。
- 接口返回历史兼容字段 `button_id` 时，以 `type:"autofill"` 为准，不要误判为按钮。
- 编辑链接：`https://app.huoban.com/admin/spaces/<space_id>/tables/<table_id>/automations/autofill/<automation_id>/edit`

## 筛选器

- 操作符结构以 `hac automation docs show field/condition` 为准，单条叶子优先用 `hac automation condition build` 生成。
- 能缩成 `item_id eq <记录ID>` 时优先精确匹配；复杂 relation 筛选的编辑器回显风险更高。
- 操作符枚举存疑时交叉核对[官方数据筛选器文档](https://s.apifox.cn/apidoc/docs-site/894643/doc-661258)。

## 跨自动化返回值

- 涉及 `call / on_call` 参数映射时，先创建或更新子自动化，再立即 `get`，以回写后的 `params[].field_id + param_key + name` 为准配置父流程。
- 父流程 `on_call.params[].field_id` 必须与子自动化 `call.params[].field_id` 一一对应；子流程参数变化后父流程同步刷新。
- verified 样本确认 relation 参数的字段 ID 可能被编辑器重建，不能直接沿用来源表字段 ID。

## 编辑器可回显优先

- `create / update` 返回 `code: 0` 不等于完工；还必须 `get` 成功，并在编辑器中确认正常回显。
- 编辑器打不开时，以编辑器认可的真实结构为准，不以上游接口容错为准。
- `call` 额外检查参数字段 ID、`param_key` 和 `{触发参数.xxx}` 是否一致。
- 父流程 `on_call` 额外检查参数字段 ID、当前触发记录的筛选方式，以及 relation 筛选的 `belong_info`。

## 交互创建页预填字段的可回显写法

> `interact_item_create.field_params` 使用 `launch_item` 时，默认生成结构可能被编辑器抹空；条件能回显不代表 `field_params` 也能回显。

- relation 字段引用整条触发记录：`code:"{触发的数据.数据ID}"`，var 用 `data_type:"item_id", field_id:"item_id", scope:"launch_item", type:"launch_item_single", is_array:false`。
- relation 字段引用触发记录的关联附加字段：`code:"{触发的数据.<字段名>}"`，var 用 `data_type:"relation", field_id:<来源字段id>, is_array:true`。
- 子表循环元素的结构以 `hac automation docs show node/sub_tables-loop-shared` 为准；verified 差异是 var 需带 `operative_scene_key:"item_create_element"`，relation 类 `belong_info.table_id` 是字符串。
- 拿不准时先创建壳配置，让用户在编辑器中手选并保存，再 `hac automation get` 读回，按真实结构仿写其余字段。

## 提交与认证（hac CLI）

> 以下是版本相关的 verified 经验；hac 升级后若行为变化，以当前实测为准。

- 管理自动化必须使用登录态 `access_token`；`api_key` 只能查数据，写自动化会报 9600001。
- hac 0.26.1 的 token 模式 create/update 不通时，用 hac 完成 skeleton/config/validate，再用 `curl` 直打 `https://api.huoban.com/paas/automation/...`，带齐：
  - `authorization: Bearer <access_token>`
  - `x-huoban-tenant-id: <租户ID>`
  - `x-huoban-token-company: <加密串>`
- create 用 `POST /paas/automation/<type>`，update 用 `PUT /paas/automation/<automation_id>`；body 先移除 `.mappings` 和 get 带回的顶层残留字段。
- 补齐请求头后仍返回 500 时，读取 `errors.error_message` 定位真实配置错误。
- condition 中不要把 `{C:表.字段.选项}` 当作筛选公式直接提交；优先用计算字段比较，确需选项筛选时用 `hac automation condition build` 生成结构化值。
- 创建页子表预填的 `subtable_field_id` 取写模式 schema 中 `sub_table` 字段自身的 `field_id`；子表表 ID 取 `config.related_table_id`。不要因配置复杂降级为后台静默创建。
