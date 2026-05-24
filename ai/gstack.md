# gstack 多角色协作

## 一句话

gstack 把需求拆给多个角色（不同 SKILL.md 加载到同一个 LLM，加上偶尔拉入 Codex），它们之间靠**三种通道**沟通：对话历史、subagent 返回值、文件落盘。

## 概念速查

- **Skill** — 一个斜杠命令（`/qa`、`/ship`），本质是一份提示词文件。
- **SKILL.md** — skill 的提示词正文。用户敲 `/qa` 时 Claude Code 把它加载到 context。
- **SKILL.md.tmpl** — 模板，含 `{{XXX}}` 占位符。由 [`gen-skill-docs`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/gen-skill-docs.ts) build 出 SKILL.md。
- **Resolver** — `{{XXX}}` 背后的生成器函数，把占位符在 build 时展开成具体散文。
- **主 agent** — 用户直接对话的 Claude 实例。
- **Subagent** — 主 agent 用 Agent 工具 spawn 出的独立实例，独立 context，跑完返回一条 message。
- **散文（prose）** — 所有控制流都是 .tmpl 里的自然语言，LLM 读了就照做，没有代码 / DSL / 状态机。
- **编排器** — 真正调度多角色沟通的 skill（如 `/autoplan`、`/ship`）。

## 三种沟通通道

| 通道 | 范围 | 介质 | 来源 |
|---|---|---|---|
| 1. 对话历史接力 | 同会话主 agent | Claude Code buffer | 原生 |
| 2. Subagent 返回消息 | 同会话主 agent ↔ subagent | Agent 工具返回值 | 原生 |
| 3. 文件落盘 | 跨会话 / 跨进程 | `~/.gstack/projects/<slug>/` | gstack 独有 |

### 1. 对话历史接力（pipeline）

主 agent 按 .tmpl 里的 Markdown 标题顺序往下读，遇到 "Follow X/SKILL.md" 就 Read 那份子 skill 执行，输出留在 buffer 里，下一步直接读得到。会话结束 buffer 丢。

- 编排器步骤举例：[autoplan Phase 1](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl#L270) / [Phase 2](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl#L395) / [Phase 3](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl#L479)，[ship Step 1-20](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/ship/SKILL.md.tmpl#L74-L919)
- **嵌套调用**：所谓"调用另一个 skill"就是让主 agent 去 Read 它的 SKILL.md。模板里写 `{{INVOKE_SKILL:foo}}`，[`composition.ts`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/composition.ts#L10-L47) build 时展开成 "Read `foo/SKILL.md`，按内容执行，跳过和我重复的 preamble"。
- **Phase gate**：阶段之间用散文软约束防 LLM 跳步。看 [autoplan Phase 1→2 gate](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl#L378-L393) 三件套：显式禁令 + 必须 emit 的过渡摘要 + 下一阶段 checklist。

### 2. Subagent 返回消息（fan-out）

主 agent 在**一条 message 内** emit 多个 Agent 工具调用，runtime 自动并发。每个 subagent 独立 context（看不到主对话历史），跑完返回一条 message 回主 agent。

- [触发措辞](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/review-army.ts#L85-L89)："Launch ALL selected specialists in a single message"
- [`{{REVIEW_ARMY}}`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/review-army.ts#L232) 注入到 [`ship:310`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/ship/SKILL.md.tmpl#L310)
- **独立 context = 独立判断**，不被主对话偏见污染（[review-army.ts:89](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/review-army.ts#L89) "fresh context — no prior review bias"）
- **双声道**是 fan-out 的特例：同时 spawn 一个 Claude subagent + 一个 Codex（Bash 调外部 CLI）。[autoplan Phase 1 CEO 实现](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl#L286-L310) 是典型例子。

### 3. 文件落盘

会话结束 buffer 就丢了，要跨会话协作必须落盘。约定路径 `~/.gstack/projects/<slug>/`（slug = git 仓库归一化标识符，由 [`bin/gstack-slug`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/bin/gstack-slug) 算）。

| 文件 | 写入者 | 读取者 | 写入代码 | 读取代码 |
|---|---|---|---|---|
| `ceo-plans/*.md` | `/plan-*` 系列 | `/ship` | [`plan-ceo-review:320`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/plan-ceo-review/SKILL.md.tmpl#L320) | [`review.ts:1005`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/review.ts#L1005)，插入在 [`ship:268`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/ship/SKILL.md.tmpl#L268) |
| [`CLAUDE.md` "Skill routing"](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/CLAUDE.md#skill-routing) | 用户首次 setup | 所有 skill | [`routing-injection.ts`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/preamble/generate-routing-injection.ts#L18-L36) | Claude Code 启动时自动加载 |

## 编排器（驱动多角色沟通的 skill）

| 类型 | Skill | 用哪些通道 |
|---|---|---|
| **多阶段 pipeline** | [`/autoplan`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/autoplan/SKILL.md.tmpl) | 通道 1 + 2（每 phase 内双声道） |
| | [`/ship`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/ship/SKILL.md.tmpl) | 通道 1 + 2（review army）+ 3（读 ceo-plans） |
| | [`/land-and-deploy`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/land-and-deploy/SKILL.md.tmpl) | 通道 1 |
| **fan-out** | [`/review`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/review/SKILL.md.tmpl) | 通道 2（review army） |
| | [`/plan-*`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/plan-ceo-review/SKILL.md.tmpl)、[`/office-hours`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/office-hours/SKILL.md.tmpl) | 通道 2（双声道） |

其他 30+ skill 都是单角色 worker，不参与多角色协调。

## 代码地图

| 想看什么 | 去哪 |
|---|---|
| 模板编译器 | [`gen-skill-docs.ts:435-453`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/gen-skill-docs.ts#L435-L453) |
| 所有 resolver 注册表 | [`resolvers/index.ts:27-84`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/index.ts#L27-L84) |
| 并行 fan-out | [`review-army.ts`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/review-army.ts) |
| 嵌套调用 | [`composition.ts`](https://github.com/garrytan/gstack/blob/026751ea2012ec7cbedc149ba615929a20026501/scripts/resolvers/composition.ts#L10-L47) |
