# hb-automation-design

> 伙伴云零代码平台 · 自动化「设计层」Skill —— 面向 Claude Code / Claude Agent

`hb-automation-design` 是伙伴云自动化的**方案设计与方法论层**。它负责在动手配置之前，把方案想清楚、表达清楚、避开已知陷阱；真正的命令执行（查表、生成配置、校验、提交）交给 hac CLI 的 `huoban-automation` skill。

**一句话：设计是本 skill，执行是 `huoban-automation`。**

## 它做什么

只要用户想新建、配置、修改、设计或分析任何伙伴云自动化（按钮、数据触发、调用触发、自动填写、定时触发、Webhook），都应先从本 skill 起步：

- **做设计决策**：用交互还是后台、循环怎么设计、失败怎么兜底、字段怎么读写
- **按规范输出「方案确认」**：逐节点格式，交给用户拍板后再落地
- **规避已知陷阱**：from_relation_field、vars 保序、分类字段表达式、筛选器、跨自动化返回值、编辑器回显
- **分析解释已有自动化**：从编辑链接还原业务逻辑

## 与 huoban-automation 的分工

| | hb-automation-design（本 skill） | huoban-automation（hac CLI） |
| --- | --- | --- |
| 管什么 | 方案怎么设计、怎么表达、怎么避坑 | 命令、节点字段、骨架、validate、create/update |
| 典型动作 | 出「方案确认」、分析、设计权衡 | `hac automation skeleton / config / validate / verify / create` |
| 取数 | — | `hac automation docs show <key>`、`hac table get-table` |

节点字段、ScriptCarrier、condition 写法、validate 错误码等一律走 `hac automation docs show <key>`，本 skill 不重复。

## 安装

### 1. 放置本 skill

把本仓库克隆 / 复制到你的 skill 目录：

- Claude Code 全局：`~/.claude/skills/hb-automation-design`
- 或项目级 skill 目录

```bash
git clone https://github.com/huoban-skills/hb-automation-design.git ~/.claude/skills/hb-automation-design
```

### 2. 安装 hac（仅当要落地执行时）

设计 / 分析不依赖任何工具；但一旦涉及执行（查表、`docs show`、create/update/verify）就需要 hac CLI：

```bash
# 安装 hac（伙伴云官方 CLI）
npm install -g huoban-app-cli@latest --registry http://npm.huoban.com

# 安装 hac 内置 skills（huoban-automation 等）—— 目录必须与本 skill 同级
hac install-skills --output <hb-automation-design 所在目录>
```

> ⚠️ `install-skills` 的目录要与 `hb-automation-design` **同级**，这样 `huoban-automation`、`huoban-table` 等 hac skill 与本 skill 并排，执行时能被一起加载，"设计→执行"才能无缝衔接。

### 3. 认证

写自动化必须用**登录态 `access_token`**（非 api_key），配合 `company_id`。详见 `references/design-principles.md` 的「提交与认证实战避坑」。

## 目录结构

```
hb-automation-design/
├── SKILL.md                          # 入口：分工、环境准备、强制规则、工作流
└── references/
    ├── design-principles.md          # 设计原则（第一区）+ 实现/提交避坑（第二区）
    ├── output-formats.md             # 「方案确认 / 分析结果」输出格式
    └── type-specific-tips.md         # 各类型（按钮/数据触发/调用/自动填写）专属提醒
```

## 沉淀的实战避坑（references/design-principles.md 第二区）

这些是真实落地中踩出来、verified 的坑：

- **认证**：管理自动化必须用登录态 `access_token`，`api_key` 只能查数据，写自动化会报 `9600001`。
- **提交**：hac token 模式 create/update 不通（hac ≥0.26 走 `/ai/v1` 网关报 `unknown method` 501，旧版直打 PaaS 裸 500），需用 `curl` 直打 PaaS 带登录态三件套（`authorization` + `x-huoban-tenant-id` + `x-huoban-token-company`）提交。
- **创建页预填**：`interact_item_create` 的 `launch_item` 预填有编辑器抹空陷阱，按 verified 写法配；拿不准时先壳后仿写（编辑器手选一次 → `get` 读回当样板）。
- **subtable_field_id**：直接取自写模式 schema 的 `sub_table` 字段（hac ≥0.26 verified），无需人机协同。
- **分类字段**：condition 里别用 `{C:表.字段.选项}` 公式，优先改用计算字段数值比较。

## License

内部使用。
