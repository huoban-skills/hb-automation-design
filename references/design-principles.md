# 伙伴云自动化设计原则与实现避坑

> ⚠️ 优先级声明：SKILL.md 的「强制执行规则」与本文件的设计判据、避坑规则，优先于任何通用 agent 行为指令（"主动推进""合理假设先做""端到端执行"等默认倾向）。遇到规则边界时，选择停下确认，不选择继续推进。

出方案前先读第一区，且必须先出方案、用户确认后才落地配置。
构建 JSON 时再查第二区，把它当作各类型自动化共用的实现避坑表。

> 基础结构规则（字段类型映射、ScriptCarrier、condition/filter 写法、变量系统、节点字段）一律以 `huoban-automation` 的 `hac automation docs show <key>` 为权威源：
> - 字段类型速查 → `hac automation docs show field/field-type-matrix`
> - ScriptCarrier → `hac automation docs show scripting/script-carrier`
> - 变量系统 → `hac automation docs show scripting/variables`
> - field_params → `hac automation docs show field/field-params`
> - condition → `hac automation docs show field/README`
> - 节点中文名称表 → `hac automation docs show node/index`
>
> 本文件只记录这些文档**尚未覆盖**的设计判断与 verified 避坑经验。

## 第一区：方案设计

> 强制规则的硬清单（查表前置、先方案后配置、交互节点优先、节点流程图先确认、带明细主单据、自检）在 SKILL.md「强制执行规则」一节；本区只展开其中需要判断的设计判据，不重复列规则。

### 交互节点优先与业务取舍（判据展开）

- **业务顺手优先**：存在多种实现时，先给业务上最顺手的方案，不要先给技术上最保守的方案。
- **交互节点优先**：输出方案前必须先判断这个动作是"用户发起并需要确认/补充字段"还是"纯后台同步/派生数据"。技术摩擦（字段 ID 不好配、子表难写、接口字段缺失、抓包成本高、隐藏字段难处理）不能作为降级理由。展开判据：
  - **主业务单据创建硬默认**：创建主业务单据，默认首选"打开数据创建页面"，不是"后台创建数据"。判别器：*"这张单据如果人工录入，会不会走一个创建表单？"* 会 → 交互创建；不会（纯派生/同步/状态机写回）→ 后台创建。
  - **关联明细表默认可预填子表**：只要明细表通过关系字段关联主表，就默认视为"打开数据创建页面"能预填该子表，不得因为字段列表里没看到显式"子表字段"条目就降级——要否定这个默认，必须先通过 `get-table` 实际核对无果。
  - **反误判**：已存在"主单提交后派生下游单据"的链路（例如发货单 → 出库单 → 出库明细），不是"上游主单应后台静默创建"的证据；恰恰相反，这通常说明主单是业务确认入口，更应优先交互创建。
  - **后台创建准入清单**：主业务单据走"后台创建数据"必须同时满足：① 用户明确要求纯后台一键处理 ② 字段完全确定无需人工确认 ③ 明细无需用户调整 ④ 不依赖预填子表能力。少一个条件就回到交互创建。方案里必须写明为什么不能用"打开数据创建页面"。

### 方案表达

- 输出"方案确认"时必须使用逐节点格式，不得只写业务步骤。每个步骤至少包含：
  - `节点：<中文节点名称>（<节点简述>）`
  - `作用：<这个节点具体做什么>`
  - `配置：<这个节点怎么配置>`
- 如果流程涉及子流程或执行调用触发，主流程和子流程必须分开列节点清单。
- 如果因为循环套循环、接口限制或字段权限导致推荐拆成子流程，必须在方案阶段写明"为什么拆"和"主流程节点 / 子流程节点"。
- 出方案时，所有节点名称和术语一律使用中文，不要用英文。节点中文名称取自 `hac automation docs show node/index` 的"节点名称"列，不要自行翻译或直接写 `type`。
- 节点简述不是节点本身的通用功能，而是用一句话概括该节点在当前业务流程里具体做的事情（例如"按订单号查询待发货明细""判断库存是否充足""循环回写每行发货数量"），和后续 `作用` 字段互补：简述贴在名称后便于一眼扫读，`作用` 再展开细节。
- 打开数据创建页的主/子表预填均属于一个节点，不要拆成两个节点来描述。

### 交互设计

- 打开数据创建页可以获取前序数据，预填主表和子表数据，在一个节点内即可完成主子表的预填。
- **子表预填必须有前置数组来源**：预填子表行时，"打开数据创建页面"依赖前序节点提供的数组；没有数组来源就无法预填子表。数组可以来自「获取多条数据」、关联字段带出的多值集合或其他能产出数组的节点——出方案时必须明确这个数组从哪个前置节点来，不能省略。

### 循环设计

- 需要逐条处理集合时，再考虑循环。
- 优先选择最贴合目标的循环模式：每条数据只被执行一次，用数组循环；如果每条数据被多次执行，用条件循环。
- 同一个自动化中，循环不能套循环；如果要循环套循环，可以考虑执行调用触发。
- 循环中禁止使用交互节点。

### 计算字段优先

- 当流程需要用数值做判断、筛选、分支、循环退出或数量分配时，必须先查目标表及其关联附加字段里是否已有可用计算字段。
- 如果已有计算字段能表达业务含义，优先直接使用计算字段，不要先用"数据统计"或"设置变量"重算。
- 计算字段既包括本表原生计算字段，也包括通过关联字段带出的附加计算字段；使用前必须核对 `from_relation_field`，确认它的真实来源和业务口径。
- 只有在没有合适计算字段、计算字段口径不匹配、或需要跨节点动态累加/递减的运行时状态时，才考虑"数据统计"或"设置变量"。

### 字段、条件与结构完整性

- 先看字段真实来源，再决定如何读写。
- 方案阶段只判断字段属于本表原生字段、关联附加字段、系统字段中的哪类；不要在未查字段配置时假设 `data_type`。
- 构建 JSON 时回到第二区，用 `data_type` 对照表和 `from_relation_field` 判断表核对具体写法。
- 即使没有条件，也要保留完整 `condition` 结构。`filter.and` 为空数组比写 `null` 更稳。
- 节点 JSON 要尽量保持结构完整，不要省略必要壳层。
- 新增字段赋值时，先从最小可用字段集开始，再逐步扩展；子表字段不存在或不可见时不要强行赋值，先只赋 1 到 2 个核心字段，成功后再逐步追加。

### API 调用设计

- `api_call` 是共享节点，不属于某个单独 automation。
- 只要流程依赖第三方接口结果，就要在方案阶段明确"失败怎么办""返回结构是否稳定""后续节点读什么字段"。
- 如果后续节点根本不依赖返回值，可以明确设计成"不解析返回值"（已观察到 `resp_body_format: 1`）；但只要后续还要判断成功失败、提取消息或字段，就不要选这一档。
- 能直接用结构化 JSON 解析就不要先走字符串拆分；只有返回不稳定或不是标准 JSON 时，再考虑读 `raw_body`。
- 需要后续节点反复使用的响应值，尽量尽早落成变量，而不是在多个节点里重复写长路径。
- 联调期若接口返回不稳定，优先补预期返回结构或 `resp_body_mock`，保证后续节点可以先搭起来。
- 返回文本乱码时，优先检查返回编码配置，再决定是否手动拆 `raw_body`。
- 涉及签名、token、nonce、timestamp 的接口，方案里必须同时说明请求头、请求体和签名串之间的关系；`url / query / headers / body` 的真实发送内容必须与签名串完全一致。
- 某些 `application/json` 接口仍会要求手工拼接 JSON 文本并参与签名，这时优先保留文本 body，不要机械改成对象 body；已有真实样本使用 `req_text_body + req_body_format: 6` 手工拼 JSON。
- 按 JSON 解析时常读 `{#N|body.xxx}`；如果节点配置为"按文本解析"，方案里就要明确后续节点是读取整个 `{#N|body}`，而不是假定还能继续走 `body.xxx`。
- **GET（及其他无 body 的请求）必须设为"无请求体"**：`hac automation skeleton --with-node api_call` 默认给的是 POST 取向的 `req_body_format: 6`(json) + `req_text_body` + `Content-Type: application/json`。配 GET 时要主动清掉请求体（去掉 `req_text_body` 与对应 `Content-Type` 头、把请求体设为"无"），不要沿用骨架的 json body——否则会留一坨多余配置，用户得手动改。落地阶段就处理，别留给用户。

### 结束与失败反馈设计

- 流程正常完成时，优先使用 `end` 做收尾。
- 流程因校验失败、外部接口失败、关键数据缺失而必须中断时，优先使用 `exit`。
- 如果用户需要明确看到失败原因，优先设计成 `exit + dialog`，而不是悄悄停掉。`exit` 常见组合是 `is_notice: true + notice_type: "dialog" + notice_icon: "error"`。
- 失败分支里的提示文案尽量直接引用真实错误字段，例如 `api_call` 的 `body.message`；不要先手拆整段 `raw_body`。
- 不要把成功和失败都塞进同一个结束节点；用 `branch + exit` 分开表达。
- 一个自动化必须有且只能有一个 `end`；分支内需要提前结束的路径统一用 `exit`，不要加第二个 `end`。

### 出方案前检查清单

已集中到 [preflight-checklist.md](preflight-checklist.md)：输出"方案确认"前逐区走查（内部动作，结论不输出到方案）。新增必检项一律加到该文件，不要回填到本节。

## 第二区：实现避坑（verified 经验，基础结构规则走 docs show）

### 字段归属判断（from_relation_field）

| 场景 | 判断方法 | 结论 |
|---|---|---|
| `from_relation_field` 为空 | 字段直接定义在当前表 | 这是本表原生字段 |
| `from_relation_field.field_id = <某关联字段>` | 字段通过该关联字段从别的表带出 | 这是关联附加字段，不是本表原生字段 |

例：如果 `发货明细.待发货数量` 的 `from_relation_field.field_id = 发货明细.订单明细`，说明这个值本质来自 `订单明细`，不能直接当成"发货明细自身剩余量"使用。

### 脚本变量保序不去重

- `scripts.vars` 数组必须按表达式 `code` 中变量出现的顺序逐项登记，**不能按 `field_id` 去重**。
- 如果同一个字段在 `code` 里出现了 2 次（例如 `IF({目标数据.待完工数量}>0,...,IF({目标数据.待完工数量}<=0,...)`），`vars` 里也必须登记 2 次，位置与 `code` 中的出现顺序一一对应。
- 去重会导致编辑器回显异常或运行时变量绑定错位。verified 样本（`automation_id: 7500000002735670`）确认了"同字段重复登记"的写法是编辑器正常回显的前提。

### 分类字段表达式提醒

- 给 category 字段赋值，或在 `IF` 等函数分支里返回选项常量时，**表达式一律用裸 `{C:表名.字段名.选项名}`**，不要默认包 `[]`。
- **不要按 `field_params.is_array` 机械判断是否包 `[]`**。已有真实样本确认：即使 `field_params.is_array: true`，单选 category 的 `code` 规范写法仍是裸 `{C:...}`，不是 `[{C:...}]`。
- 典型正例：
  - 直接赋值：`{C:工序任务.工序状态.未开始}`
  - 函数分支：`IF({目标数据.待下发生产总数}>0, {C:生产计划.状态.待下发生产}, {C:生产计划.状态.已下发生产})`
- 只有字段配置上**真的多选**（可同时勾中多个选项）的 category，赋值表达式里才可能出现数组字面量写法；这种场景目前没有 verified 正例，不要仅凭 `is_array:true` 反推成 `[{C:...}]`。
- `condition.filter.query.eq` 里对 category 做**值比较**时，写法和赋值不同：verified 样本确认比较值要用 `[{C:...}]`（带方括号）。典型正例（`automation_id: 7500000002735670`）：`"code": "[{C:委外订单明细.委外类型.工序委外}]"`。赋值用裸写，比较用方括号——两套规则不要混。

### 数组建模提醒

- `Name/Value` 这类键值对数组，通常更适合先转中间变量，再按固定字段回填。
- `Items` 这类天然明细数组，通常更适合直接 `loop_by_target`。
- 如果最终目标就是逐条创建明细表，优先直接循环数组并在循环体里 `item_create`，不要为了保留主表明细文本把流程绕复杂。

### 数据触发（item_trigger）实现避坑

> 设计判断（异步边界、权限组、涟漪层级、类型职责）在 type-specific-tips 的 item_trigger 段；本节只放构建 JSON 时的写法。

- 单条精确更新优先 `item_id eq {#N|获取的数据.数据ID}`，不要用业务字段组合重新定位同一条记录。
- `item_find_multi` 的 `return_field_id` 一旦有值，返回的是**字段值集合**不是记录集合：下游字段用 `mode:"input"`，脚本 `code: "{#N|获取的数据}"`（裸，不接字段名），`vars` 里 `data_type` 写字段真实类型、`table_id` 写 `0`（形态 B，见 `hac automation docs show node/item_find_multi`）。
- `item_update` 读"当前被修改的目标记录"字段用 `{目标数据.字段名}`，`vars` 里 `scope:"target_item" + type:"item"`，不要再加 `item_find` 先查一遍。
- 更新"触发记录某 relation 字段指向的那条记录"时，`item_update.condition.filter.target` 直接写 `field_id:"item_id" + table_id:<目标表ID>`（**不带 `belong_info`**），值来源传 `{触发的数据.<关系字段>}`。
- `field_params` 直接引用触发数据/前序结果/变量时**优先 `mode:"input"`**，脚本只填裸引用；只有需要函数、拼接、判断或单位转换时才切 `mode:"script"`。
- 旧版 workflow 的 `ITEM(...)` / `FIELDS(...)` / `OBJ_GET(JSON2OBJ(...))` / `JSONPATH(...)` 读别表数据，迁数据触发时**不要原样塞进 `script`**；改为加 `item_find` / `item_find_multi` 节点，下游用 `{#N|获取的数据.字段名}` 消费。
- `loop_by_target` 按需用：只是"查一组数据整体写进某字段"用 `item_find_multi` + `field_params` 即可；只有真要"对每条结果单独建下游记录或调接口"才走循环。

### 筛选器补充提醒

- `em` / `spec_boolean` / 各类型操作符的结构化写法已由 `hac automation docs show field/condition` 权威覆盖，单条叶子优先用 `hac automation condition build` 生成，本节不再重复。
- 能把目标筛选条件缩成 `item_id eq <某个具体记录ID>` 的场景，优先缩成 `item_id` 精确匹配。复杂关系字段筛选的编辑器回显风险显著高于 `item_id` 精确匹配。
- 官方数据筛选器文档：[https://s.apifox.cn/apidoc/docs-site/894643/doc-661258](https://s.apifox.cn/apidoc/docs-site/894643/doc-661258)。遇到操作符枚举等存疑问题，可回官方文档交叉核对。

### 跨自动化返回值提醒

- 涉及 `call / on_call` 的参数映射时，**先创建或更新子自动化，再立刻 `get` 一次，以回写后的 `params[].field_id + param_key + name` 为准，再去配置父流程**。不要假设 `call.params[].field_id` 等于来源表原字段 ID，尤其 relation 参数经编辑器保存后可能被重建为新的参数字段 ID。
- 父流程 `on_call.params[].field_id` 必须与子自动化 `call.params[].field_id` 一一对应；如果页面端改过参数名、是否必填、默认值或 relation 参数配置，父流程也要同步刷新。
- verified 样本（按钮 `7500000002768373` 调子流程 `7500000002768372`）确认：子流程 relation 参数的真实字段 ID 是编辑器回写后的 `2200000601408825`，不是来源表字段 `2200000601328153`。

### 编辑器可回显优先

- `create` / `update` 接口返回 `code: 0` 不等于自动化已完工。创建或更新一条自动化后，必须再做两步验证——`get` 成功 + 在编辑器里打开这条自动化确认能正常回显。
- 只要编辑器打不开，就以编辑器回显的真实结构为准，不以接口容错为准；遇到"接口收得下但编辑器打不开"时，应回头比对字段壳层，而不是上线这个版本。
- 如果是 `call` 自动化，额外检查：① `params[].field_id` 是否被编辑器重建 ② `params[].param_key` 是否变化 ③ `code` 里的 `{触发参数.xxx}` 是否与最新参数名一致。
- 如果是父流程 `on_call`，额外检查：① `params[].field_id` 是否已对齐子流程最新参数字段 ID ② `item_find_multi` 针对当前触发单据的筛选是否优先使用 `item_id` ③ `filter.target.field_id` 为 relation 字段时是否补齐 `belong_info`。

### 交互创建页用触发数据预填字段的可回显写法（verified）

> `interact_item_create`（打开数据创建页）的 `field_params` 用 `launch_item` 预填字段时，`script build field-assign` 默认生成的 `launch_item` 引用**编辑器渲染不出来**，用户在编辑器一保存就被抹成空（`code:""`、`vars:[]`），表现为"预填值没填进去"。`item_find_multi` 的 `condition` 里同样的 `launch_item` 引用反而能存活（后端会转成 `{触发的数据.数据ID}` 这种编辑器认得的形式）——所以**条件能回显不代表 field_params 也能回显**，两处要分别确认。编辑器认可的写法：

- **relation 字段 ← 整条触发记录**（如 发货单.销售订单 ← 当前订单）：`code:"{触发的数据.数据ID}"`，var 用 `data_type:"item_id", field_id:"item_id", scope:"launch_item", type:"launch_item_single", is_array:false`。**不要用 `data_type:"item"` 不带 field_id 的整条 item 写法**——那个版本会被编辑器抹空。
- **relation 字段 ← 触发记录的某关联附加字段**（如 发货单.客户 ← 订单.客户）：`code:"{触发的数据.<字段名>}"`，var 用 `data_type:"relation", field_id:<来源字段id>, is_array:true`。relation 字段值要 `is_array:true`，即便目标字段 `is_multi=0`。
- **子表行预填引用循环元素**：`sub_tables` 结构、`loop_subtable` 变量绑定与类型推断已由 `hac automation docs show node/sub_tables-loop-shared` 权威覆盖，以它为准。本文只补两个 verified 差异：var 需带 `operative_scene_key:"item_create_element"`；子表 relation 类 `field_params` 的 `belong_info.table_id` 是**字符串**。
- **拿不准回显写法时的兜底（推荐）**：先 create 一个只配了壳的版本，让用户在编辑器里手选一次该字段（relation 选"触发的数据"/子表选好循环来源）→ 保存 → `hac automation get` 读回，以编辑器存下的 `code`/`vars` 结构为样板**原样仿写**其余字段，再 update。比盲配 `launch_item`/`loop_element` 可靠得多。

### 提交与认证实战避坑（hac CLI，verified）

> 用 hac 落地自动化时实测踩到的坑，与运行环境/版本相关；hac 后续版本可能修复，遇到先按此排查。

- **管理自动化必须用登录态 `access_token`，`api_key` 不行**：`api_key`（`HUOBAN_API_KEY`，`Open-Authorization` 头）只能查表 / 查数据 / 查工作区（走 OpenAPI / AI 网关）；创建·修改自动化走 PaaS 网关（`/paas/automation/...`），按**用户登录态**判权限。用 api_key 调会报 `非工作区或表格管理员，无权管理触发器`（code 9600001）——这不是"账号不是管理员"，而是 api_key 这种应用身份在 PaaS 接口下不被当作可管理触发器的用户。改用 token 模式：config 里设 `HUOBAN_ACCESS_TOKEN` + `HUOBAN_COMPANY_ID`（去掉 api_key）。access_token 从浏览器登录态抓：F12 → 任意请求头 `authorization: Bearer <token>`。

- **hac token 模式 create / update / verify 不通 → 改 `curl` 直打 PaaS（verified @ hac 0.26.1）**：hac ≥0.26 把自动化写操作改发 AI 网关 `https://api.huoban.com/ai/v1/paas/automation/...`，token 模式只带 `Authorization: Bearer <access_token>`，网关回 `unknown method`（code 501）；旧版直打 PaaS 缺 `x-huoban-token-company` 头则是裸 500——**两代报错不同，结论一致：hac 的 create/update 在 token 模式下都不通**。只读的 `get` 能用。**绕过办法**——照常用 hac `skeleton` / `config` / `validate` 把 config 做对（这些本地/只读步骤 token 模式都能用），**create/update 这一步改用 `curl` 直打 PaaS 接口**，带齐登录态三件套头：
  - `authorization: Bearer <access_token>`
  - `x-huoban-tenant-id: <租户ID>`（浏览器请求头 `x-huoban-tenant-id`）
  - `x-huoban-token-company: <加密串>`（浏览器请求头 `x-huoban-token-company`）
  - 端点：create → `POST https://api.huoban.com/paas/automation/<type>`（**不带 `/ai/v1` 前缀**）；update → `PUT https://api.huoban.com/paas/automation/<automation_id>`。body 结构可用 `hac automation create ... --show-request` 反查（注意把它显示的 `/ai/v1/paas/...` 换成 `/paas/...`）。
  - body 用 hac validate 通过的 config 文件（先 `del(.mappings)` 及 get 带回的顶层残留字段如 `button_id/status/tips/...`）。
  - **补齐头后若仍 500，读 `errors.error_message`**——那才是真正的配置错误（裸 500 时被掩盖）。

- **condition 里 category 不要用 `{C:表.字段.选项}` 公式**：后端写入时报 `script_error：变量无效或不存在`（`verify` 可能拦不住，到 create/update 才暴露）。优先改用等价的**计算字段数值比较**（例：按钮"订单状态∈未发货/部分发货才显示" → 改判"待发货总数 > 0"，正好命中计算字段优先原则，且避开 category 公式坑）；确需按选项筛选时用 `hac automation condition build` 配选项的结构化值，不要塞 `{C:}` 公式字面量。

- **创建页子表预填的 `subtable_field_id` 直接取自写模式 schema（verified，hac ≥0.26）**：`hac table get-table --output-mode write` 返回的主表字段里含 `sub_table` 类型字段，该字段自身的 `field_id` 就是 `subtable_field_id`，子表表 ID 取其 `config.related_table_id`，同步开关看 `config.sub_table_sync`（结构见 `hac automation docs show data-model/table-relations`）。`loop.id` 也无需从编辑器读回，按 `lp`+6 位随机串自行生成（规则见 `node/sub_tables-loop-shared`）。**不要因为子表配置麻烦就降级成后台 `loop_by_target + item_create`**——那是把"打开创建页让用户确认主子表"错误降级成后台静默创建。
