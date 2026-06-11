# hb-automation-design

> 伙伴云零代码平台自动化设计与落地约束 Skill，面向 Claude Code / Claude Agent

`hb-automation-design` 负责自动化落地前后的两段工作：

1. 方案确认前：读取真实表结构，完成业务取舍和节点方案设计。
2. 方案确认后：配合 hac `huoban-automation` 落地，并应用 verified 实现避坑。

真正的命令执行、节点字段、骨架、校验和提交能力由 hac CLI 的 `huoban-automation` 提供。

## 它做什么

只要用户想新建、配置、修改、设计或分析伙伴云自动化（按钮、数据触发、调用触发、自动填写、定时触发、Webhook），都应先从本 skill 起步：

- 选择正确的自动化类型和交互方式
- 基于真实表结构输出逐节点“方案确认”
- 在输出前执行内部方案自检
- 用户确认后按需加载实现避坑，再切到 hac 落地
- 分析解释已有自动化的业务逻辑

## 与 huoban-automation 的分工

| | hb-automation-design | huoban-automation（hac CLI） |
| --- | --- | --- |
| 管什么 | 方案设计、表达规范、经验约束 | 命令、节点字段、骨架、校验、提交 |
| 典型动作 | 判类型、出方案确认、自检、分析 | `skeleton / config / validate / verify / create` |
| 取数 | 规定何时查、查什么 | `docs show <key>`、`table get-table` |

节点字段、ScriptCarrier、condition、变量系统和 validate 错误码统一以 `hac automation docs show <key>` 为准，本 skill 不重复维护。

## 工作流

1. `hac automation docs show workflow/intent-routing` 判类型。
2. 读取 `design-principles.md` 和当前类型的 `type-design-tips.md`。
3. 按 `output-formats.md` 形成完整方案草稿，暂不发送。
4. 对方案草稿强制执行一次独立自检；有缺项则修改后重新自检。
5. 自检通过后才输出“方案确认”。
6. 用户确认后，按需读取 `implementation-pitfalls.md`。
7. 切到 `huoban-automation` 构建、校验、提交并验证编辑器回显。

## 安装

```bash
git clone https://github.com/huoban-skills/hb-automation-design.git ~/.claude/skills/hb-automation-design
```

需要真正落地执行时，再安装 hac 及其内置 skills：

```bash
npm install -g huoban-app-cli@latest --registry http://npm.huoban.com
hac install-skills --output <hb-automation-design 所在目录的父目录>
```

`huoban-automation`、`huoban-table` 等 hac skill 必须与 `hb-automation-design` 同级。

## 认证

写自动化必须使用登录态 `access_token`，不能使用只能查数据的 `api_key`。认证和 hac 版本相关绕行方案见 `references/implementation-pitfalls.md` 的“提交与认证”。

## 目录结构

```text
hb-automation-design/
├── SKILL.md
├── README.md
├── CHANGELOG.md
└── references/
    ├── design-principles.md
    ├── type-design-tips.md
    ├── output-formats.md
    └── implementation-pitfalls.md
```

## 关键约束

- 查真实表结构后才能输出字段和节点方案。
- 必须先让用户确认方案，再构建 JSON。
- 创建主业务单据默认优先交互创建；带明细时子表预填必须有数组来源。
- 方案自检是内部动作，不在用户方案中输出“自检”段落。
- create/update 成功后仍需 `get` 并检查编辑器回显。

## 变更记录

见 [CHANGELOG.md](CHANGELOG.md)。

## License

内部使用。
