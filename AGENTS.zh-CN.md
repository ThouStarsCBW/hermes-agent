# Hermes Agent - 开发指南

面向在 hermes-agent 代码库上工作的 AI 编码助手与开发者的说明。

**永远不要放弃正确的解决方案。**

## Hermes 是什么

Hermes 是一个个人 AI 智能体，它在 CLI、消息网关（Telegram、Discord、Slack 以及约 20 个其他平台）、TUI 以及一个 Electron 桌面应用中运行同一个智能体核心。它跨会话学习（记忆 + 技能），委派给子智能体，运行定时任务，并驱动真实的终端与浏览器。它的扩展主要通过 **插件与技能** 实现，而不是通过膨胀核心。

有两个特性塑造了几乎每一个设计决策，也是审视任何改动的视角：

- **每轮对话的 prompt 缓存是神圣的。** 一个长生命周期的对话每一轮都会复用缓存的前缀。任何改变过往上下文、切换工具集、或在对话中途重建系统提示词的行为，都会让该缓存失效并成倍抬高用户的成本。我们不这样做（唯一的例外是上下文压缩）。
- **核心是窄腰；能力位于边缘。** 我们添加的每一个模型工具都会随每一次 API 调用一起下发，因此新增一个*核心*工具的门槛很高。大多数新能力应当作为 CLI 命令 + 技能、服务门控工具或插件来落地——而不是作为核心表面。

## 贡献准则——我们想要什么 / 我们不想要什么

这是项目的意图层。以两种方式使用它：

1. **给人看，也给你的工作看**——什么会被合入，什么会被拒绝，这样一份贡献就能瞄准目标。
2. **给自动化审查（triage sweeper，分诊清扫器）看**——指导在什么情况下一个 PR 可以安全地以三种允许的理由关闭（`implemented_on_main`、`cannot_reproduce`、`incoherent`），同样重要的是，**在什么时候不应关闭**一个 PR。基于品味的“我们不想要这个 / 超出范围”的关闭不是自动化决策——那些留给人类维护者。清扫器在这里的工作是识别设计意图，*避免错误地关闭一份合法贡献*，而不是由它自己来做出“不实现”的判定。

把握好平衡：Hermes 发布的东西**很多**——大多数合入的是针对真实报告行为的 bug 修复，而产品表面（平台、渠道、provider、模型、桌面/TUI 功能）是有意地、激进地扩张的。下面的克制精准地指向 **核心智能体 + 模型工具 schema**，即唯一一个每多一项都会被每一次 API 调用付费的地方。“最小足迹”约束的是*一项能力如何接入核心*，而**不是**该产品是否允许增长。我们在边缘上扩张，在腰部上保守。

### 我们想要的

- **把真实 bug 修好。** 落地的大部分是针对一个实际报告症状的 `fix(...)`。一个好的修复能在当前 `main` 上复现该症状，指出它具体出现在哪一行，并修复整个 bug 类——包括兄弟调用路径——而不只是报告者撞到的那一个站点。
- **在边缘上扩展覆盖面。** 新的平台适配器、渠道、provider、模型，以及桌面/TUI/仪表盘功能都受欢迎，并且会例行合入，包括大型的（一个新的消息渠道、一个 session-cap 功能、一个 Windows PTY 桥）。产品广度是一个目标，而非足迹顾虑——只要它接入现有的 setup/config UX（`hermes tools`、`hermes setup`、自动安装），而不是硬塞一个裸环境变量。
- **把 god-file 重构为干净的模块。** 把 `cli.py` / `run_agent.py` / `gateway/run.py` 中数千行的代码块抽取成一个聚焦的 mixin 或模块，是受欢迎的工作，即便 diff 巨大且机械（大型的 `+N/-N` 重构会例行合入）。“每一行都能追溯到请求”的测试只适用于*功能*类 PR；一个声明过的重构，其请求本身就是这次抽取。
- **保持核心窄。** 新的*模型工具*是昂贵的例外——每一个工具都会随每一次调用下发。优先顺序为：扩展已有代码 → CLI 命令 + 技能 → 服务门控工具（`check_fn`）→ 插件 → catalog 中的 MCP server → 新的核心工具（最后手段）。参见“足迹阶梯”。
- **扩展，而非重复。** 在新增一个模块/管理器/hook 之前，先检查已有的基础设施是否已经覆盖了该用例。当多个 PR 集成的是同一个*类别*的东西时，设计一个共享接口，而不是把它们一个一个地合入（参见足迹阶梯下的 ABC + orchestrator 说明）。
- **行为契约优先于快照。** 测试应当断言两段数据之间必须如何关联（不变关系），而不是冻结一个当前值（模型列表、配置版本字面量、枚举计数）。参见“不要写变更检测器式测试”。
- **E2E 验证，而非仅让单元测试 mock 变绿。** 对于任何触及解析链、配置传播、安全边界、远程后端或文件/网络 I/O 的东西，要用真实导入、在一个临时的 `HERMES_HOME` 上跑真实路径。Mock 会掩盖集成 bug。
- **对缓存、交替与不变关系安全。** 保留 prompt 缓存、严格的消息角色交替（绝不让两个同角色消息连续出现；绝不在循环中途注入一个合成的用户消息），以及一个在整段对话生命周期内字节稳定的系统提示词。
- **保留贡献者署名。** 通过 cherry-pick（rebase-merge）来抢救外部工作，使作者身份保留在 git 历史中；当你可以在其上构建时，不要推倒重来。

### 我们不想要的（即便写得好也会被拒）

- **投机性基础设施。** 没有具体消费者的 hook、回调或扩展点。加一个 hook 很容易；但在插件依赖上它之后要移除它却很难。如果贡献者有一个真实的、声明过的用例，那这个 hook 就*不是*投机的——即使消费者是分开发布的。
- **为非密钥配置新增 `HERMES_*` 环境变量。** `.env` 只用于密钥（API key、令牌、密码）。所有行为设置——超时、阈值、功能开关、显示偏好——都放进 `config.yaml`。如果该机制确实需要某个环境变量，从 `config.yaml` 桥接过去，但面向用户的文档要指向 `config.yaml`。拒绝那些让用户“在 .env 里设置 X”的 PR，除非 X 是凭据。
- **当 terminal + file 已经能胜任、或用一个技能就能做到时，还新增一个核心工具。** 如果唯一的障碍是某个远程后端上的文件可见性，那就去修挂载，而不是去改工具集。
- **在教学类工具上加懒读取逃逸口。** 在那些需要智能体完整读取内容的工具（技能、提示词、playbook）上，不要加 `offset`/`limit` 分页。模型会读第 1 页然后跳过其余部分。
- **会摧毁其所保护功能的“修复”。** 一个把功能目的都干掉的缓解措施，本身就是错误的缓解。在限制行为之前，先阅读原始提交的意图（`git log -p -S`）；找一个能保留该功能的修复。
- **未经 opt-in 门控的出站遥测 / 用量归因。** 在存在一个通用的、面向用户的 opt-in（配置门 + 安装引导提示 + `hermes tools` 开关）之前，不要新增任何分析、第三方标识符打标或归因标签。先挂在某个标签后面，不要合入。
- **变更检测器式测试、对话中途破坏缓存、没有 E2E 证明就接进来的死代码，以及触碰核心文件的插件。** 插件活在它们自己的目录里，在我们提供的 ABC/hook 之内工作；如果一个插件需要更多能力，那就拓宽通用的插件表面，而不要在核心里为它开特例。
- **把第三方产品 / 别人的项目集成进核心树。** 可观测性后端、厂商 SaaS 集成、分析看板，以及类似的“别人家产品”的插件，不会落在本仓库的 `plugins/` 下。它们会给我们带来持续的维护负担，因为我们要让它们在快速演进的核心之上保持工作，而我们并不拥有那个后端。把它们作为**独立插件仓库**发布，让用户安装到 `~/.hermes/plugins/`（或通过 pip entry point），并在 Nous Research Discord（`#plugins-skills-and-skins`）推广。这是一个耦合与维护决策，而非质量门槛——插件可以很优秀却仍然被拒。往树里加这种目录的 PR 会被关闭，并附带一个指向“把它作为自己的仓库发布”的指引。

### 在你把它称为 bug 之前——先验证前提（以及在什么情况下不应关闭）

一份写得很好的 PR 被关闭，最常见的原因不是代码质量——而是这次改动建立在**错误的前提**之上，或者它把**刻意的某个设计**当成了缺口。这些模式具有双向作用：它们告诉人类审查者该细看什么，也告诉自动化清扫器一个 PR 在什么时候*不*应被以 `implemented_on_main` / `cannot_reproduce` 关闭（拿不准时，留给人类保持开放）。这些是提炼自真实关闭案例的。

- **“刻意的设计，而非缺口。”** 一个看起来像疏漏的限制，往往是有意为之。在“修复”一个缺失的链接或限制之前，先问一问这个隔离*本身*是不是设计。例子：profile 是彼此独立的孤岛，这是有意为之——一个让实时配置从默认 profile 继承的 PR 被关闭了，因为把 profile 耦合在一起恰恰是该设计所阻止的（在创建时复制的 `--clone` 路径已经覆盖了合法的“从我的默认配置起步”的情形）。在假设某物“未完成”之前，先阅读原始提交的意图（`git log -p -S "<symbol>"`）。
- **“该前提在 X 的真实运作方式面前站不住脚。”** 一个 PR 的论证理由常常建立在对某个已有机制的错误心智模型之上。在接受其理由之前，先追踪真实的代码/运行时。两个真实关闭案例：一个限流“在冷却期内重新探测”的 PR（断路器只在*已确认空*的账户桶上跳闸，所以重新探测只是对我们已经证明为空了的桶一顿猛锤）；一个用量累积的修复，其新分支在运行时**永远不会执行**，因为更早的一道保护已经把它所依赖的状态弹出了。如果你无法指出 bug 具体出现在哪一行，*并且*证明修复改变了那一行的行为，那你就没有验证过前提。
- **“这个修复是错的——那个缺失/遗漏是有意的。”** 加上那个看起来显然缺失的部件，可能会破坏该遗漏所保护的东西。例子：恢复“缺失的” `__init__.py` 文件，让一个测试树变得可被作为带点的包导入，从而遮蔽了真实的插件，在导入时删掉了它的 `register()`。那个“缺失”其实是有承载重量的。
- **“越界 / 复活了我们早已放弃的方向。”** 取代了既定基础的范畴蔓延，或者复活了维护者刻意关闭的方向，即便代码能跑也会被拒。把改动保持在真正达成一致的那个窄小片段上；其余部分作为聚焦的后续提议。

贯穿其中的主线：**在写修复或合入修复之前，先在代码库中验证主张*与*意图。** 一份在当前 `main` 上确认的复现，加上一段“修复作用于哪一行”的行级说明，永远胜过听起来有理的理由。拿不准意图时，问一句比发出一个与设计对抗的修复要便宜得多。

### 足迹阶梯（新能力决策）

每一级都比它上面那级增加了更多永久表面。选择能够正确解决问题的最高（最小足迹）那一级：

1. **扩展已有代码**——该能力是某种已存在事物的变体。零新增表面。
2. **CLI 命令 + 技能**——管理那些可以表示为 shell 命令的配置/状态/基础设施。智能体在技能的引导下运行 `hermes <subcommand>`。零模型工具足迹。订阅、定时任务、服务设置的默认选择。例子：`hermes webhook`、`hermes cron`、`hermes tools`。
3. **服务门控工具（`check_fn`）**——需要结构化的参数/返回值*并且*只在某个前置条件被配置时才出现。否则零足迹。例子：Home Assistant 工具（以令牌为门控）、memory-provider 工具。
4. **插件**——不属于核心发布的第三方/小众/用户特定能力。活在 `~/.hermes/plugins/` 或 pip 包中，在运行时被发现。
5. **MCP server（在 catalog 中）**——如果该能力确实需要是一个工具（智能体调用的结构化 I/O），但又不是核心基础的，那么比起膨胀核心工具集，优先把它构建成一个 MCP server 并加入 MCP catalog。智能体通过内置的 MCP 客户端连接它；零永久核心 schema 足迹，且可被任何 MCP host 复用。
6. **新的核心工具**——仅当该能力是基础性的、对几乎所有用户都广泛有用、且无法经由 terminal + file（或一个 MCP server）触达时。正确核心工具的例子：terminal、read_file、web_search、browser_navigate。

当 3+ 个开放的 PR 试图集成同一个*类别*的东西（memory 后端、provider、通知器）时，不要一个一个地合入——设计一个 ABC + orchestrator，把已有的内置实现作为第一个 provider 包起来，然后把竞争的 PR 改成对着那个接口的插件。

## 开发环境

```bash
# 优先用 .venv；如果你的 checkout 里是 venv，就退回 venv。
source .venv/bin/activate   # 或：source venv/bin/activate
```

`scripts/run_tests.sh` 会先探测 `.venv`，然后是 `venv`，然后是 `$HOME/.hermes/hermes-agent/venv`（针对与主线 checkout 共享 venv 的 worktree）。

## 项目结构

文件数量不断变化——不要把下面的树当成穷尽的。
规范的来源是文件系统。注释标出了你实际会去编辑的承载重量的入口点。

```
hermes-agent/
├── run_agent.py          # AIAgent 类——核心对话循环（约 12k 行）
├── model_tools.py        # 工具编排、discover_builtin_tools()、handle_function_call()
├── toolsets.py           # 工具集定义、_HERMES_CORE_TOOLS 列表
├── cli.py                # HermesCLI 类——交互式 CLI 编排器（约 11k 行）
├── hermes_state.py       # SessionDB——SQLite 会话存储（FTS5 搜索）
├── hermes_constants.py   # get_hermes_home()、display_hermes_home()——感知 profile 的路径
├── hermes_logging.py     # setup_logging()——agent.log / errors.log / gateway.log（感知 profile）
├── batch_runner.py       # 并行批处理
├── agent/                # 智能体内部（provider 适配器、记忆、缓存、压缩等）
├── hermes_cli/           # CLI 子命令、安装引导向导、插件加载器、皮肤引擎
├── tools/                # 工具实现——经由 tools/registry.py 自动发现
│   └── environments/     # 终端后端（local、docker、ssh、modal、daytona、singularity）
├── gateway/              # 消息网关——run.py + session.py + platforms/
│   ├── platforms/        # 每个平台的适配器（telegram、discord、slack、whatsapp、
│   │                     #   homeassistant、signal、matrix、mattermost、email、sms、
│   │                     #   dingtalk、wecom、weixin、feishu、qqbot、bluebubbles、
│   │                     #   yuanbao、webhook、api_server、...）。参见 ADDING_A_PLATFORM.md。
│   └── builtin_hooks/    # 始终注册的网关 hook 的扩展点（未发布任何内容）
├── plugins/              # 插件系统（见下文“插件”一节）
│   ├── memory/           # 记忆提供方插件（honcho、mem0、supermemory、...）
│   ├── context_engine/   # 上下文引擎插件
│   ├── model-providers/  # 推理后端插件（openrouter、anthropic、gmi、...）
│   ├── kanban/           # 多智能体看板调度器 + worker 插件
│   ├── hermes-achievements/  # 游戏化成就追踪
│   ├── observability/    # 指标 / trace / 日志插件
│   ├── image_gen/        # 图像生成 provider
│   └── <others>/         # disk-cleanup、google_meet、platforms、spotify、
│                         #   strike-freedom-cockpit、...
├── optional-skills/      # 随仓库发布但默认不激活的较重/小众技能
├── skills/               # 与仓库一起打包的内置技能
├── ui-tui/               # Ink（React）终端 UI——`hermes --tui`
│   └── src/              # entry.tsx、app.tsx、gatewayClient.ts + app/components/hooks/lib
├── tui_gateway/          # TUI 的 Python JSON-RPC 后端
├── acp_adapter/          # ACP server（VS Code / Zed / JetBrains 集成）
├── cron/                 # 调度器——jobs.py、scheduler.py
├── scripts/              # run_tests.sh、release.py、辅助脚本
├── website/              # Docusaurus 文档站点
└── tests/                # Pytest 套件（截至 2026 年 5 月，约 17k 测试，分布在约 900 个文件）
```

**用户配置：** `~/.hermes/config.yaml`（设置）、`~/.hermes/.env`（仅 API key）。
**日志：** `~/.hermes/logs/`——`agent.log`（INFO 及以上）、`errors.log`（WARNING 及以上）、运行网关时的 `gateway.log`。经由 `get_hermes_home()` 感知 profile。
用 `hermes logs [--follow] [--level ...] [--session ...]` 浏览。

## TypeScript 风格

适用于 Hermes 中的 TypeScript：桌面、TUI、网站以及未来的 TS 包。

- 当状态被共享、被复用、或被远距离的 UI 读取时，优先使用小型 nanostore，而非组件状态。
- 让每个特性拥有它自己的 atom。聊天状态靠近聊天，shell 状态靠近 shell，共享状态放在 `src/store`。
- 从 atom 渲染的组件应使用 `useStore`。非渲染的动作应用 `$atom.get()` 读取。
- 当一个叶子组件可以订阅该 atom 时，不要把状态穿过三个组件层层传递。
- 把持久化放在拥有它的那个 atom 旁边。
- 保持路由根组件轻薄。它们组合路由与 shell；不应变成控制器。
- 不要写庞大的 hook。一个 hook 应当只负责一件窄小的事。
- 优先使用就近放置的 action 模块，而非隐藏的 god hook。
- 如果一个回调是纯副作用，使用精简的 void 形式：`onState={st => void setGatewayState(st)}`。
- 异步 UI 处理器应让意图显式化：`onClick={() => void save()}`。
- 公共 props 与共享对象形状优先使用 interface。避免对对象 props 使用 `type X = { ... }`。
- 用 React 原语扩展 props：`React.ComponentProps<'button'>`、`React.ComponentProps<typeof Dialog>`、`Omit<...>`、`Pick<...>`。
- 当映射 id、路由或视图时，表驱动优于条件阶梯。
- `src/app` 拥有路由、页面与页面特定组件。
- `src/store` 拥有共享 atom。
- `src/lib` 拥有共享的纯函数 helper。

## 文件依赖链

```
tools/registry.py  （无依赖——被所有工具文件导入）
       ↑
tools/*.py  （每个文件在导入时调用 registry.register()）
       ↑
model_tools.py  （导入 tools/registry + 触发工具发现）
       ↑
run_agent.py、cli.py、batch_runner.py、environments/
```

---

## AIAgent 类（run_agent.py）

真实的 `AIAgent.__init__` 接收约 60 个参数（凭据、路由、回调、会话上下文、预算、凭据池等）。下面的签名是你通常会接触的最小子集——完整列表请阅读 `run_agent.py`。

```python
class AIAgent:
    def __init__(self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,              # "chat_completions" | "codex_responses" | ...
        model: str = "",                   # 空 → 之后从 config/provider 解析
        max_iterations: int = 90,          # 工具调用迭代次数（与子智能体共享）
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,              # "cli"、"telegram" 等
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        credential_pool=None,
        # ... 还有回调、thread/user/chat ID、iteration_budget、fallback_model、
        # checkpoints 配置、prefill_messages、service_tier、reasoning_config 等。
    ): ...

    def chat(self, message: str) -> str:
        """简单接口——返回最终的响应字符串。"""

    def run_conversation(self, user_message: str, system_message: str = None,
                         conversation_history: list = None, task_id: str = None) -> dict:
        """完整接口——返回带 final_response + messages 的 dict。"""
```

### 智能体循环

核心循环位于 `run_conversation()` 内部——完全同步，带有中断检查、预算追踪，以及一次转弯的宽限调用：

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) \
        or self._budget_grace_call:
    if self._interrupt_requested: break
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

消息遵循 OpenAI 格式：`{"role": "system/user/assistant/tool", ...}`。
推理内容存储在 `assistant_msg["reasoning"]`。

---

## CLI 架构（cli.py）

- **Rich** 用于 banner/面板，**prompt_toolkit** 用于带自动补全的输入
- **KawaiiSpinner**（`agent/display.py`）——API 调用期间的动画表情，`┊` 用于工具结果的动态信息流
- `cli.py` 中的 `load_cli_config()` 合并硬编码默认值 + 用户 config YAML
- **皮肤引擎**（`hermes_cli/skin_engine.py`）——数据驱动的 CLI 主题化；在启动时从 `display.skin` 配置键初始化；皮肤自定义 banner 颜色、spinner 表情/动词/翅膀、工具前缀、响应框、品牌文字
- `process_command()` 是 `HermesCLI` 上的一个方法——经由中央注册表的 `resolve_command()` 按规范命令名分发
- 技能斜杠命令：`agent/skill_commands.py` 扫描 `~/.hermes/skills/`，作为**用户消息**注入（而非系统提示词），以保留 prompt 缓存

### 斜杠命令注册表（`hermes_cli/commands.py`）

所有斜杠命令都定义在一个中央的 `COMMAND_REGISTRY` 列表里，元素是 `CommandDef` 对象。每一个下游消费者都自动从这个注册表派生：

- **CLI**——`process_command()` 经由 `resolve_command()` 解析别名，按规范名分发
- **网关**——`GATEWAY_KNOWN_COMMANDS` frozenset 用于 hook 发射，`resolve_command()` 用于分发
- **网关 help**——`gateway_help_lines()` 生成 `/help` 输出
- **Telegram**——`telegram_bot_commands()` 生成 BotCommand 菜单
- **Slack**——`slack_subcommand_map()` 生成 `/hermes` 子命令路由
- **自动补全**——扁平的 `COMMANDS` dict 喂给 `SlashCommandCompleter`
- **CLI help**——`COMMANDS_BY_CATEGORY` dict 喂给 `show_help()`

### 新增一个斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中加一个 `CommandDef` 条目：
```python
CommandDef("mycommand", "它是做什么的描述", "Session",
           aliases=("mc",), args_hint="[arg]"),
```
2. 在 `cli.py` 的 `HermesCLI.process_command()` 中加处理器：
```python
elif canonical == "mycommand":
    self._handle_mycommand(cmd_original)
```
3. 如果该命令在网关中可用，在 `gateway/run.py` 中加处理器：
```python
if canonical == "mycommand":
    return await self._handle_mycommand(event)
```
4. 对于持久化设置，使用 `cli.py` 中的 `save_config_value()`

**CommandDef 字段：**
- `name`——不带斜杠的规范名（例如 `"background"`）
- `description`——人类可读的描述
- `category`——`"Session"`、`"Configuration"`、`"Tools & Skills"`、`"Info"`、`"Exit"` 之一
- `aliases`——替代名字的元组（例如 `("bg",)`）
- `args_hint`——help 中显示的参数占位（例如 `"<prompt>"`、`"[name]"`）
- `cli_only`——仅在交互式 CLI 中可用
- `gateway_only`——仅在消息平台中可用
- `gateway_config_gate`——配置点路径（例如 `"display.tool_progress_command"`）；当设置在一个 `cli_only` 命令上时，如果该配置值为真，则该命令在网关中变为可用。`GATEWAY_KNOWN_COMMANDS` 总是包含配置门控命令，以便网关能分发它们；help/菜单只在门打开时才显示它们。

**新增别名**只需要在已有 `CommandDef` 的 `aliases` 元组里加上即可。无需改动其他文件——分发、help 文字、Telegram 菜单、Slack 映射和自动补全都会自动更新。

---

## TUI 架构（ui-tui + tui_gateway）

TUI 是经典（prompt_toolkit）CLI 的完整替代品，经由 `hermes --tui` 或 `HERMES_TUI=1` 激活。

### 进程模型

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ 渲染转录、composer、提示词、动态信息流
```

TypeScript 拥有屏幕。Python 拥有会话、工具、模型调用和斜杠命令逻辑。

### 传输

基于 stdio 的换行分隔 JSON-RPC。请求来自 Ink，事件来自 Python。完整的方法/事件目录见 `tui_gateway/server.py`。

### 关键表面

| 表面 | Ink 组件 | 网关方法 |
|---------|---------------|----------------|
| 聊天流式 | `app.tsx` + `messageLine.tsx` | `prompt.submit` → `message.delta/complete` |
| 工具活动 | `thinking.tsx` | `tool.start/progress/complete` |
| 审批 | `prompts.tsx` | `approval.respond` ← `approval.request` |
| Clarify/sudo/secret | `prompts.tsx`、`maskedPrompt.tsx` | `clarify/sudo/secret.respond` |
| 会话选择器 | `sessionPicker.tsx` | `session.list/resume` |
| 斜杠命令 | 本地处理器 + fallthrough | `slash.exec` → `_SlashWorker`、`command.dispatch` |
| 补全 | `useCompletion` hook | `complete.slash`、`complete.path` |
| 主题化 | `theme.ts` + `branding.tsx` | 带皮肤数据的 `gateway.ready` |

### 斜杠命令流程

1. 内置的客户端命令（`/help`、`/quit`、`/clear`、`/resume`、`/copy`、`/paste` 等）在 `app.tsx` 本地处理
2. 其余一切 → `slash.exec`（在持久的 `_SlashWorker` 子进程中运行）→ `command.dispatch` 兜底

### 开发命令

```bash
cd ui-tui
npm install       # 首次
npm run dev       # watch 模式（重建 hermes-ink + tsx --watch）
npm start         # 生产环境
npm run build     # 完整构建（hermes-ink + tsc）
npm run typecheck # 仅类型检查（tsc --noEmit）
npm run lint      # eslint
npm run fmt       # prettier
npm test          # vitest
```

### 仪表盘中的 TUI（`hermes dashboard` → `/chat`）

仪表盘嵌入的是真实的 `hermes --tui`——**而不是**重写。参见 `hermes_cli/pty_bridge.py` + `hermes_cli/web_server.py` 中的 `@app.websocket("/api/pty")` 端点。

- 浏览器加载 `web/src/pages/ChatPage.tsx`，它挂载 xterm.js 的 `Terminal`（带 WebGL 渲染器）、`@xterm/addon-fit` 用于容器驱动的缩放，以及 `@xterm/addon-unicode11` 用于现代宽字符宽度。
- `/api/pty?token=…` 升级为 WebSocket；鉴权使用与 REST 相同的临时 `_SESSION_TOKEN`，经由 query 参数（浏览器无法在 WS 升级上设置 `Authorization`）。
- 服务端生成 `hermes --tui` 本应生成的任何东西，通过 `ptyprocess`（POSIX PTY——WSL 可用，原生 Windows 不行）。
- 帧：每一方向都是原始 PTY 字节；缩放经由服务器拦截的 `\x1b[RESIZE:<cols>;<rows>` 并应用 `TIOCSWINSZ`。

**不要在 React 中重新实现主聊天体验。** 主转录、composer/输入流（包括斜杠命令行为）以及 PTY 支撑的终端属于被嵌入的 `hermes --tui`——你对 Ink 新增的任何东西都会自动出现在仪表盘中。如果你发现自己为了仪表盘而重建转录或 composer，停下来，去扩展 Ink。

**在 TUI 周围构建结构化的 React UI 是允许的，前提是它不是第二个聊天表面。** 侧边栏小部件、检视器、摘要、状态面板以及类似的辅助视图（例如 `ChatSidebar`、`ModelPickerDialog`、`ToolCall`）是可以的，只要它们补充被嵌入的 TUI，而不是取代转录 / composer / 终端。让它们的状态独立于 PTY 子进程的会话，并以非破坏性的方式呈现它们的失败，以便终端面板能不受损地持续工作。

### Electron 桌面聊天应用（`apps/desktop/`）

一个与经典 CLI 以及仪表盘嵌入的 TUI 都**不同的**聊天表面。它是一个 Electron + React + nanostore 渲染器（`@assistant-ui/react`），通过 JSON-RPC（`requestGateway(method, params)`）与一个 `tui_gateway` 后端对话。WebSocket/JSON-RPC 传输位于框架无关的 `apps/shared` 包（`@hermes/shared`——`JsonRpcGatewayClient` + WS URL helper）中，Web 仪表盘（`web/`）也消费它；**桌面对仪表盘前端没有构建/运行时依赖**——它生成一个 headless 的 `hermes serve` 后端服务器（与 `dashboard` 服务的是同一个网关，只是完全去掉了浏览器 UI：`serve` 设置 `headless_backend=True`，因此 `cmd_dashboard` 跳过 `_build_web_ui` *并且* 导出 `HERMES_SERVE_HEADLESS=1`，这样即便存在一个游离的 `web_dist/`，`mount_spa()` 也会禁用 SPA——只有 JSON-RPC/WS/API 表面可达）。`dashboard` 和 `serve` 共享 `cmd_dashboard`/`start_server`，但是独立的表面——两者都不会启动另一个。唯一的例外是一个向后兼容的*兜底*：`serve` 较新，因此桌面生成（`electron/backend-command.ts` + `electron/main.ts` 中的 `backendSupportsServe()`）会检测解析出的运行时是否注册了 `serve`，并且仅当没有注册时（一个较旧的管理式安装 / 应用尚未更新的 PATH 中 `hermes`），才把 argv 改写为旧的 `dashboard --no-open`。如果没有它，一个新应用面对一个未升级的运行时会在未知子命令上崩溃，并让每个升级中途的用户都卡死。它**不**嵌入 `hermes --tui`——它有自己的 composer、转录和斜杠命令管线。关于范围限定的桌面架构、状态、解析器、传输和测试规则，请阅读 `apps/desktop/AGENTS.md`。

**桌面应用中的斜杠命令先在客户端精选，再分发到后端。** 管线如下：

- **后端已经提供了一切。** `tui_gateway/server.py` 的 `commands.catalog`（空查询列表）和 `complete.slash`（带输入查询的补全）都包含内置命令、用户的 `quick_commands`，*以及* 技能派生的命令（`scan_skill_commands()` / `get_skill_commands()`）。桌面应用不需要为看到技能而新增一个 RPC。
- **渲染器通过 `apps/desktop/src/lib/desktop-slash-commands.ts` 精选。** 这是承载重量的文件。它持有 `DESKTOP_COMMAND_SPECS`（内置命令及其桌面表面）以及 `NO_DESKTOP_SURFACE` 屏蔽列表，用于那些不应污染桌面弹出层的、终端专用 / 消息专用 / 选择器拥有 / 设置拥有 / 高级命令。
  - `isDesktopSlashCommand(name)`——门控**执行**。对内置*以及* 任何非内置（技能 / 快捷命令）都返回 true，因此带输入扩展的命令可以运行。
  - `isDesktopSlashSuggestion(name)`——门控**发现/补全**。被 `app/chat/composer/hooks/use-slash-completions.ts` 中的两条补全路径（空查询 catalog 过滤器 + 带输入查询的 `complete.slash` 过滤器）*以及* `filterDesktopCommandsCatalog` 使用。
  - `isDesktopSlashExtensionCommand(name)`——当该命令不是一个已知的 Hermes 内置命令（即一个技能或用户快捷命令）时为 true。补全路径和 catalog 过滤路径都放行扩展，以便技能命令出现在面板中。（在修复“桌面斜杠面板中缺失技能命令”时加入——那个精选过的允许列表曾静默地把每个技能/快捷命令从补全中丢弃，尽管它们被输入时运行得很好。）
- **分发**位于 `app/session/hooks/use-prompt-actions/slash.ts`（`runSlash`）：桌面拥有的内置命令（`/skin`、`/help`、`/new`、…）在本地或通过 `commands.catalog` 处理；其余一切走 `slash.exec`，兜底到 `command.dispatch`（网关将其解析为技能 / 别名 / exec 指令）。一个技能命令被解析为 `{type: "skill", message}` 并作为普通提示词提交。

**规则：** 桌面斜杠面板的精选是为了隐藏噪音（终端专用 / 消息专用的内置命令），而*不是*为了隐藏用户激活的扩展。技能命令和 `quick_commands` 是后端呈现的扩展——它们属于补全。如果你收紧 `desktop-slash-commands.ts`，保持 `isDesktopSlashExtensionCommand` 流入补全路径和 catalog 过滤路径两者。测试：从 `apps/desktop` 出发，运行 `npx vitest run src/lib/desktop-slash-commands.test.ts`（工作区依赖安装在仓库根目录）。

---

## 新增工具

在新增任何工具之前，先解决足迹问题（见贡献准则中的“足迹阶梯”）：大多数能力*不应*是核心工具。对于自定义或仅本地使用的工具，请**不要**修改 Hermes 核心。改用插件路线：创建 `~/.hermes/plugins/<name>/plugin.yaml` 和 `~/.hermes/plugins/<name>/__init__.py`，然后用 `ctx.register_tool(...)` 注册工具。插件工具集会被自动发现，且无需触碰 `tools/` 或 `toolsets.py` 即可启用或禁用。

仅当用户明确贡献一个应当在基础系统中发布的新*核心* Hermes 工具时，才使用下面的内置路线。

内置/核心工具需要改动 **2 个文件**：

**1. 创建 `tools/your_tool.py`：**
```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**2. 加入 `toolsets.py`**——要么 `_HERMES_CORE_TOOLS`（所有平台），要么一个新的工具集。**这一步是必需的：** 自动发现会导入该工具并注册其 schema，但只有当其名字出现在一个工具集中时，该工具才会*暴露给智能体*。`_HERMES_CORE_TOOLS` 不是死代码——它是每个平台基础工具集所继承的默认包。

自动发现：任何带有顶层 `registry.register()` 调用的 `tools/*.py` 文件都会被自动导入——无需维护手动的导入列表。接入某个工具集仍然是一个刻意的、手动的步骤。

注册表负责 schema 收集、分发、可用性检查与错误包装。所有 handler 必须返回一个 JSON 字符串。

**工具 schema 中的路径引用**：如果 schema 描述提到了文件路径（例如默认输出目录），使用 `display_hermes_home()` 使其感知 profile。该 schema 在导入时生成，那是在 `_apply_profile_override()` 设置 `HERMES_HOME` 之后。

**状态文件**：如果一个工具存储了持久状态（缓存、日志、检查点），使用 `get_hermes_home()` 作为基础目录——绝不要用 `Path.home() / ".hermes"`。这确保每个 profile 都有自己的状态。

**智能体级工具**（todo、memory）：在 `handle_function_call()` 之前被 `run_agent.py` 拦截。模式见 `tools/todo_tool.py`。

---

## 依赖固定策略

所有依赖都必须有上限，以限制供应链攻击面。该策略在 litellm 被攻陷（PR #2796、#2810）之后确立，并在 Mini Shai-Hulud 蠕虫行动（2026 年 5 月）之后强化。

| 来源类型 | 处理方式 | 例子 |
|---|---|---|
| PyPI 包 | `>=floor,<next_major` | `"httpx>=0.28.1,<1"` |
| Git URL | Commit SHA | `git+https://...@<40-char-sha>` |
| GitHub Actions | Commit SHA + 注释 | `uses: actions/checkout@<sha>  # v4` |
| 仅 CI 的 pip | `==exact` | `pyyaml==6.0.2` |

**往 `pyproject.toml` 新增依赖时：**
1. 对于 1.0 之后的版本，固定为 `>=current_version,<next_major`（例如 `>=1.5.0,<2`）。
2. 对于 1.0 之前的包，使用 `<0.(current_minor + 2)`（例如 `>=0.29,<0.32`）。
3. 绝不能提交一个没有上限的裸 `>=X.Y.Z`——CI 和审查者都会拒绝它。
4. 运行 `uv lock` 以重新生成带哈希的 `uv.lock`。

参考：#2810（上限修正）、#9801（SHA 固定 + 审计 CI）。

---

## 新增配置

### config.yaml 选项：
1. 在 `hermes_cli/config.py` 的 `DEFAULT_CONFIG` 中加入。
2. 仅当你需要主动迁移/转换既有用户配置（重命名键、改变结构）时，才提升 `_config_version`（查看 `DEFAULT_CONFIG` 顶部的当前值）。在一个既有 section 里新增一个键，会被深合并自动处理，*不需要*提升版本号。

### 顶层 `config.yaml` section（非穷尽）：

`model`、`agent`、`terminal`、`compression`、`display`、`stt`、`tts`、`memory`、`security`、`delegation`、`smart_model_routing`、`checkpoints`、`auxiliary`、`curator`、`skills`、`gateway`、`logging`、`cron`、`profiles`、`plugins`、`honcho`。

`auxiliary` 持有针对 side-LLM 工作的逐任务覆盖（curator、vision、embedding、标题生成、session_search 等）——每个任务可以钉住自己的 provider/model/base_url/max_tokens/reasoning_effort。解析顺序见 `agent/auxiliary_client.py::_resolve_auto`。

`curator` 持有后台技能维护配置——`enabled`、`interval_hours`、`min_idle_hours`、`stale_after_days`、`archive_after_days`、`backup`（嵌套）。

### .env 变量（仅限密钥——API key、令牌、密码）：
1. 在 `hermes_cli/config.py` 的 `OPTIONAL_ENV_VARS` 中加入元数据：
```python
"NEW_API_KEY": {
    "description": "它是用来做什么的",
    "prompt": "显示名称",
    "url": "https://...",
    "password": True,
    "category": "tool",  # provider、tool、messaging、setting
},
```

非密钥设置（超时、阈值、功能开关、路径、显示偏好）属于 `config.yaml`，而非 `.env`。如果内部代码需要某个环境变量镜像以向后兼容，就在代码里从 `config.yaml` 桥接到那个环境变量（见 `gateway_timeout`、`terminal.cwd` → `TERMINAL_CWD`）。

### 配置加载器（三条路径——清楚你身在何处）：

| 加载器 | 使用者 | 位置 |
|--------|---------|----------|
| `load_cli_config()` | CLI 模式 | `cli.py`——合并 CLI 特定默认值 + 用户 YAML |
| `load_config()` | `hermes tools`、`hermes setup`、大多数 CLI 子命令 | `hermes_cli/config.py`——合并 `DEFAULT_CONFIG` + 用户 YAML |
| 直接 YAML 加载 | 网关运行时 | `gateway/run.py` + `gateway/config.py`——原始读取用户 YAML |

如果你新增了一个键，CLI 看到了但网关没看到（或反之），那你用错了加载器。检查 `DEFAULT_CONFIG` 的覆盖范围。

### 工作目录：
- **CLI**——使用进程的当前目录（`os.getcwd()`）。
- **消息**——使用 `config.yaml` 中的 `terminal.cwd`。网关把它桥接到子工具用的 `TERMINAL_CWD` 环境变量。**`MESSAGING_CWD` 已被移除**——如果它设置在 `.env` 里，配置加载器会打印一条弃用警告。`.env` 中的 `TERMINAL_CWD` 同理；规范设置是 `config.yaml` 中的 `terminal.cwd`。

---

## 皮肤/主题系统

皮肤引擎（`hermes_cli/skin_engine.py`）提供数据驱动的 CLI 视觉定制。皮肤是**纯数据**——新增一个皮肤无需改动代码。

### 架构

```
hermes_cli/skin_engine.py    # SkinConfig dataclass、内置皮肤、YAML 加载器
~/.hermes/skins/*.yaml       # 用户安装的自定义皮肤（即插即用）
```

- `init_skin_from_config()`——在 CLI 启动时调用，读取 config 中的 `display.skin`
- `get_active_skin()`——返回当前皮肤的缓存 `SkinConfig`
- `set_active_skin(name)`——在运行时切换皮肤（被 `/skin` 命令使用）
- `load_skin(name)`——先加载用户皮肤，再加载内置皮肤，最后回退到默认
- 缺失的皮肤值会自动从 `default` 皮肤继承

### 皮肤定制的内容

| 元素 | 皮肤键 | 使用者 |
|---------|----------|---------|
| Banner 面板边框 | `colors.banner_border` | `banner.py` |
| Banner 面板标题 | `colors.banner_title` | `banner.py` |
| Banner section 头部 | `colors.banner_accent` | `banner.py` |
| Banner 暗色文字 | `colors.banner_dim` | `banner.py` |
| Banner 正文文字 | `colors.banner_text` | `banner.py` |
| 响应框边框 | `colors.response_border` | `cli.py` |
| Spinner 表情（等待中） | `spinner.waiting_faces` | `display.py` |
| Spinner 表情（思考中） | `spinner.thinking_faces` | `display.py` |
| Spinner 动词 | `spinner.thinking_verbs` | `display.py` |
| Spinner 翅膀（可选） | `spinner.wings` | `display.py` |
| 工具输出前缀 | `tool_prefix` | `display.py` |
| 逐工具 emoji | `tool_emojis` | `display.py` → `get_tool_emoji()` |
| 智能体名称 | `branding.agent_name` | `banner.py`、`cli.py` |
| 欢迎消息 | `branding.welcome` | `cli.py` |
| 响应框标签 | `branding.response_label` | `cli.py` |
| 提示符 | `branding.prompt_symbol` | `cli.py` |

### 内置皮肤

- `default`——经典 Hermes 金/可爱风格（当前外观）
- `ares`——深红/青铜战神主题，带自定义 spinner 翅膀
- `mono`——干净的灰阶单色
- `slate`——冷蓝、面向开发者的主题

### 新增一个内置皮肤

在 `hermes_cli/skin_engine.py` 的 `_BUILTIN_SKINS` dict 中加入：

```python
"mytheme": {
    "name": "mytheme",
    "description": "简短描述",
    "colors": { ... },
    "spinner": { ... },
    "branding": { ... },
    "tool_prefix": "┊",
},
```

### 用户皮肤（YAML）

用户创建 `~/.hermes/skins/<name>.yaml`：

```yaml
name: cyberpunk
description: Neon-soaked terminal theme

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  banner_accent: "#FF1493"

spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "

tool_prefix: "▏"
```

用 `/skin cyberpunk` 或在 config.yaml 中设置 `display.skin: cyberpunk` 激活。

---

## 插件

Hermes 有两个插件表面。两者都活在仓库的 `plugins/` 下，这样随仓库发布的插件能和用户安装于 `~/.hermes/plugins/` 的插件、以及 pip 安装的 entry point 一起被发现。

### 通用插件（`hermes_cli/plugins.py` + `plugins/<name>/`）

`PluginManager` 从 `~/.hermes/plugins/`、`./.hermes/plugins/` 和 pip entry point 发现插件。每个插件暴露一个 `register(ctx)` 函数，它可以：

- 注册 Python 回调生命周期 hook：
  `pre_tool_call`、`post_tool_call`、`pre_llm_call`、`post_llm_call`、`on_session_start`、`on_session_end`
- 经由 `ctx.register_tool(...)` 注册新工具
- 经由 `ctx.register_cli_command(...)` 注册 CLI 子命令——插件的 argparse 树在启动时接入 `hermes`，于是 `hermes <pluginname> <subcmd>` 无需改动 `main.py` 即可工作

Hook 从 `model_tools.py`（pre/post tool）和 `run_agent.py`（生命周期）调用。**发现时机陷阱：** `discover_plugins()` 只作为导入 `model_tools.py` 的副作用运行。那些在导入 `model_tools.py` 之前就读插件状态的代码路径，必须显式调用 `discover_plugins()`（它是幂等的）。

### 记忆提供方插件（`plugins/memory/<name>/`）

用于可插拔记忆后端的独立发现系统。当前内置 provider 包括 **honcho、mem0、supermemory、byterover、hindsight、holographic、openviking、retaindb**。

每个 provider 实现 `MemoryProvider` ABC（见 `agent/memory_provider.py`），并被 `agent/memory_manager.py` 编排。生命周期 hook 包括 `sync_turn(turn_messages)`、`prefetch(query)`、`shutdown()`，以及可选的 `post_setup(hermes_home, config)`（用于安装引导集成）。

**经由 `plugins/memory/<name>/cli.py` 的 CLI 命令：** 如果一个记忆插件定义了 `register_cli(subparser)`，`discover_plugin_cli_commands()` 会在 argparse 设置时找到它，并接入 `hermes <plugin>`。该框架只为**当前激活的**记忆 provider（从 config.yaml 的 `memory.provider` 读取）暴露 CLI 命令，因此被禁用的 provider 不会污染 `hermes --help`。

**规则（Teknium，2026 年 5 月）：** 插件**不得**修改核心文件（`run_agent.py`、`cli.py`、`gateway/run.py`、`hermes_cli/main.py` 等）。如果一个插件需要框架未暴露的能力，那就拓宽通用的插件表面（新的 hook、新的 ctx 方法）——绝不要把插件特定的逻辑硬编码进核心。PR #5295 正是因此从 `main.py` 移除了 95 行硬编码的 honcho argparse。

**不再新增树内记忆 provider（策略，2026 年 5 月）：** `plugins/memory/` 下的内置记忆 provider 集合已封闭。新的记忆后端必须作为**独立插件仓库**发布，让用户安装到 `~/.hermes/plugins/`（或通过 pip entry point）——它们实现同一个 `MemoryProvider` ABC，经由同一个发现路径注册，并通过 `hermes memory setup` / `post_setup()` 集成，而不落地于本树。往 `plugins/memory/` 下加新目录的 PR 会被关闭，并附带一个指向“把 provider 作为自己的仓库发布”的指引。既有的树内 provider 保留；对它们的 bug 修复受欢迎。

**不再新增树内第三方产品插件（策略，2026 年 6 月）：** 同样的规则超出记忆 provider 的范围。集成别人家产品或项目的插件——可观测性/指标后端、厂商 SaaS 连接器、分析看板、付费服务绑定——必须作为**独立插件仓库**发布，让用户安装到 `~/.hermes/plugins/`（或通过 pip entry point）。它们经由已有的插件发现路径注册，并使用我们暴露的 ABC/hook/ctx 表面；核心里不需要任何特别的东西。原因是维护负担：我们吸收进树里的每一个产品，都会成为我们要在一个我们不拥有的后端上、对抗快速演进的核心去保持它工作的负担。在 Nous Research Discord（`#plugins-skills-and-skins`）推广独立插件。往 `plugins/` 下加这种目录的 PR 会被关闭，并附带一个指向“把它作为自己的仓库发布”的指引——这是一个耦合决策，而非质量判定。（已经存在于树里的 `observability/`、`kanban/`、`disk-cleanup/` 等目录是既有先例，而不是一个“在它们旁边再加更多第三方产品插件”的邀请。）

### 模型 provider 插件（`plugins/model-providers/<name>/`）

每一个推理后端（openrouter、anthropic、gmi、deepseek、nvidia、…）都作为此处的插件发布。每个插件的 `__init__.py` 在模块加载时调用 `providers.register_provider(ProviderProfile(...))`。`providers/__init__.py._discover_providers()` 是一个**惰性的、独立的**发现系统——在第一次 `get_provider_profile()` 或 `list_providers()` 调用时扫描，*而非* 由通用 PluginManager 扫描。

扫描顺序：
1. 打包的：`<repo>/plugins/model-providers/<name>/`
2. 用户的：`$HERMES_HOME/plugins/model-providers/<name>/`
3. 遗留的：`<repo>/providers/<name>.py`（向后兼容）

同名用户插件覆盖打包插件——`register_provider()` 是最后写入者获胜。这让第三方无需仓库补丁即可替换任何内置 profile。

通用 PluginManager 记录 `kind: model-provider` 清单，但**不会**导入它们（那样会双重实例化 `ProviderProfile`）。没有显式 `kind:` 的插件会通过源文本启发式（在 `__init__.py` 中的 `register_provider` + `ProviderProfile`）被自动强制转换。

完整编写指南：`website/docs/developer-guide/model-provider-plugin.md`。

### 仪表盘 / 上下文引擎 / 图像生成插件目录

`plugins/context_engine/`、`plugins/image_gen/` 等遵循相同模式（ABC + orchestrator + 每插件目录）。上下文引擎接入 `agent/context_engine.py`；图像生成 provider 接入 `agent/image_gen_provider.py`。参考 / 文档配套插件（`example-dashboard`、`strike-freedom-cockpit`、`plugin-llm-example`、`plugin-llm-async-example`）活在
[`hermes-example-plugins`](https://github.com/NousResearch/hermes-example-plugins)
配套仓库中，而非本树。

---

## 技能

两个平行的表面：

- **`skills/`**——随仓库发布、默认可加载的内置技能。按类别目录组织（例如 `skills/github/`、`skills/mlops/`）。
- **`optional-skills/`**——随仓库发布但默认*不*激活的较重或小众技能。经由 `hermes skills install official/<category>/<skill>` 显式安装。适配器位于 `tools/skills_hub.py`（`OptionalSkillSource`）。类别包括 `autonomous-ai-agents`、`blockchain`、`communication`、`creative`、`devops`、`email`、`health`、`mcp`、`migration`、`mlops`、`productivity`、`research`、`security`、`web-development`。

审查技能 PR 时，检查它们针对的是哪个目录——重度依赖或小众的技能属于 `optional-skills/`。

### SKILL.md frontmatter

标准字段：`name`、`description`、`version`、`author`、`license`、`platforms`（OS 门控列表：`[macos]`、`[linux, macos]`、...）、`metadata.hermes.tags`、`metadata.hermes.category`、`metadata.hermes.related_skills`、`metadata.hermes.config`（技能需要的 config.yaml 设置——存储在 `skills.config.<key>` 下，在安装时提示，在加载时注入）。

顶层的 `tags:` 和 `category:` 也被接受，并由加载器从 `metadata.hermes.*` 镜像过来。

### 技能编写标准（硬性底线）

每一个新增或现代化过的技能——无论是打包的、可选的还是贡献的——在合入前都必须满足这些标准。违反它们的 PR 会被审查者拒绝。

1. **`description` ≤ 60 字符，一句话，以句号结尾。** 过长的描述会膨胀技能列表，并在许多技能被加载时稀释模型的注意力。陈述能力，而非实现。不要有营销词（“powerful”、“comprehensive”、“seamless”、“advanced”）。不要重复技能名。用以下方式验证：
   ```python
   import re, pathlib
   m = re.search(r'^description: (.*)$',
                 pathlib.Path('skills/<cat>/<name>/SKILL.md').read_text(),
                 re.MULTILINE)
   assert len(m.group(1)) <= 60, len(m.group(1))
   ```

2. **SKILL.md 正文中引用的工具必须是原生 Hermes 工具或该技能明确期望的 MCP server。** 当技能需要一项能力时，用反引号按名字指向正确的工具（`` `terminal` ``、`` `web_extract` ``、`` `read_file` ``、`` `patch` ``、`` `search_files` ``、`` `vision_analyze` ``、`` `browser_navigate` ``、`` `delegate_task` `` 等）。**不要**给已经被智能体包装过的 shell 工具起名字——`grep` → `search_files`、`cat`/`head`/`tail` → `read_file`、`sed`/`awk` → `patch`、`find`/`ls` → `search_files target='files'`。如果技能依赖某个 MCP server，请写出该 MCP server 的名字，并在 `## Prerequisites` 中记录期望的安装。其他任何东西（第三方 CLI、shell 管道等）在脚本文件里是合法的，但不应作为正文里的头条交互表面。

3. **`platforms:` 门控要对照真实的脚本导入进行审计。** 使用 POSIX 专有原语（`fcntl`、`termios`、`os.setsid`、`os.kill(pid, 0)` 用于存活检测、`/proc`、硬编码的 `/tmp`、`signal.SIGKILL`、bash here-doc、`osascript`、`apt`、`systemctl`）的技能必须声明它们支持的平台。默认姿态：先尝试跨平台修复它——`tempfile.gettempdir`、`pathlib.Path`、`psutil.pid_exists`、Python 级别的过滤而非 `grep`。仅当该依赖确实是平台绑定的，才收窄到更小的集合。

4. **`author` 先署人类贡献者的名。** 对于外部贡献，贡献者的真实姓名 + GitHub handle 放在最前；“Hermes Agent”是次要协作者。如果贡献者的提交显示作者是“Hermes Agent”（因为他们用 Hermes 起草了技能），把它替换成他们的真实姓名——署人类的名，而非工具的名。

5. **SKILL.md 正文使用现代 section 顺序。** `# <Skill> Skill` 标题、2-3 句介绍说明它做什么和不做什么、`## When to Use`、`## Prerequisites`、`## How to Run`、`## Quick Reference`、`## Procedure`、`## Pitfalls`、`## Verification`。复杂技能目标约 200 行，简单技能约 100 行。删掉冗余的开场废话、营销式散文，以及对 `## Prerequisites` 中已有 env 变量的重复解释。

6. **脚本放在 `scripts/`，参考放在 `references/`，模板放在 `templates/`。** 不要指望模型每次调用都内联写出解析器、XML 遍历器或非平凡逻辑——发布一个 helper 脚本。在 SKILL.md 中按相对于技能目录的路径引用它。

7. **测试位于 `tests/skills/test_<skill>_skill.py`**，且只使用 stdlib + pytest + `unittest.mock`。不要有真实网络调用。经由 `scripts/run_tests.sh tests/skills/test_<skill>_skill.py -q` 运行。

8. **`.env.example` 的增补要隔离在一个清晰界定边界的块内。** 不要动周围的文件——贡献者提供的 `.env.example` 版本通常已过时，在抢救时，该技能自身块之外的编辑必须丢弃。

外部技能 PR 的完整抢救 / 现代化清单位于 `hermes-agent-dev` 技能中的 `references/new-skill-pr-salvage.md`——在打磨贡献者技能 PR 之前先加载它。

---

## 工具集

所有工具集都定义在 `toolsets.py` 中，作为一个单一的 `TOOLSETS` dict。每个平台的适配器挑选一个基础工具集（例如 Telegram 用 `"messaging"`）；`_HERMES_CORE_TOOLS` 是大多数平台继承的默认包。

当前工具集键：`browser`、`clarify`、`code_execution`、`cronjob`、`debugging`、`delegation`、`discord`、`discord_admin`、`feishu_doc`、`feishu_drive`、`file`、`homeassistant`、`image_gen`、`kanban`、`memory`、`messaging`、`moa`、`rl`、`safe`、`search`、`session_search`、`skills`、`spotify`、`terminal`、`todo`、`tts`、`video`、`vision`、`web`、`yuanbao`。

逐平台启用/禁用：经由 `hermes tools`（curses UI）或 `config.yaml` 中的 `tools.<platform>.enabled` / `tools.<platform>.disabled` 列表。

---

## 委派（`delegate_task`）

`tools/delegate_tool.py` 生成一个带有隔离上下文 + 终端会话的子智能体。默认情况下，父进程等待子进程返回摘要，再继续它自己的循环。配合 `background=true`，Hermes 立即返回一个委派 id，结果稍后经由异步委派完成队列重新进入对话。

两种形态：

- **单发：** 传入 `goal`（+ 可选的 `context`、`toolsets`）。
- **批处理（并行）：** 传入 `tasks: [...]`——每个都获得自己的子智能体并发运行。并发度由 `delegation.max_concurrent_children`（默认 3）封顶。

角色：

- `role="leaf"`（默认）——专注的 worker。不能调用 `delegate_task`、`clarify`、`memory`、`send_message`、`execute_code`。
- `role="orchestrator"`——保留 `delegate_task`，以便它能生成自己的 worker。由 `delegation.orchestrator_enabled`（默认 true）门控，并由 `delegation.max_spawn_depth`（默认 2）约束。

关键配置旋钮（在 `config.yaml` 的 `delegation:` 下）：`max_concurrent_children`、`max_spawn_depth`、`child_timeout_seconds`、`orchestrator_enabled`、`subagent_auto_approve`、`inherit_mcp_toolsets`、`max_iterations`。

持久性规则：后台 `delegate_task` 与当前轮次解耦，但仍是进程局部的。对于必须跨越进程重启存活的工作，改用 `cronjob` 或 `terminal(background=True, notify_on_complete=True)`。

---

## Curator（技能生命周期）

后台技能维护系统，追踪智能体自建技能的使用情况，并自动归档陈旧的那些。用户永远不会丢失技能；归档会进入 `~/.hermes/skills/.archive/` 且可恢复。

- **核心：** `agent/curator.py`（审查循环、自动转换、LLM 审查提示词）+ `agent/curator_backup.py`（运行前 tar.gz 快照）。
- **CLI：** `hermes_cli/curator.py` 接入 `hermes curator <verb>`，动词包括：`status`、`run`、`pause`、`resume`、`pin`、`unpin`、`archive`、`restore`、`prune`、`backup`、`rollback`。
- **遥测：** `tools/skill_usage.py` 拥有侧车文件 `~/.hermes/skills/.usage.json`——逐技能的 `use_count`、`view_count`、`patch_count`、`last_activity_at`、`state`（active / stale / archived）、`pinned`。

不变关系：
- Curator 只触碰 `created_by: "agent"` 来源的技能——打包的 + hub 安装的技能免碰。
- 从不删除；最具破坏性的动作是归档。
- 钉住的技能豁免于每一次自动转换，也豁免于 LLM 审查环节。
- `skill_manage(action="delete")` 拒绝钉住的技能；patch/edit/write_file/remove_file 则放行，以便智能体能持续改进钉住的技能。

配置 section（`config.yaml` 中的 `curator:`）：`enabled`、`interval_hours`、`min_idle_hours`、`stale_after_days`、`archive_after_days`、`backup.*`。

完整面向用户的文档：`website/docs/user-guide/features/curator.md`。

---

## Cron（定时任务）

`cron/jobs.py`（任务存储）+ `cron/scheduler.py`（tick 循环）。智能体经由 `cronjob` 工具调度任务；用户经由 `hermes cron <verb>`（`list`、`add`、`edit`、`pause`、`resume`、`run`、`remove`）或 `/cron` 斜杠命令。

支持的 schedule 格式：
- 时长：`"30m"`、`"2h"`、`"1d"`
- “every”短语：`"every 2h"`、`"every monday 9am"`
- 5 段 cron 表达式：`"0 9 * * *"`
- ISO 时间戳（一次性）：`"2026-06-01T09:00:00Z"`

逐任务字段包括 `skills`（加载特定技能）、`model` / `provider` 覆盖、`script`（运行前数据收集脚本，其 stdout 被注入提示词；`no_agent=True` 会把脚本变成整个任务）、`context_from`（把任务 A 的最后输出链式接入任务 B 的提示词）、`workdir`（在一个加载了它自己的 `AGENTS.md`/`CLAUDE.md` 的特定目录中运行），以及多平台投递。

加固不变关系：
- Cron 会话上的 **3 分钟硬中断**——失控的智能体循环无法垄断调度器。
- 追补窗口：任务周期的一半，夹在 120s–2h 之间。
- 宽限窗口：针对错过触发时间的一次性任务，120s。
- `~/.hermes/cron/.tick.lock` 的文件锁防止跨进程重复 tick。
- Cron 会话默认传入 `skip_memory=True`；记忆 provider 在 cron 期间有意不运行。

Cron 投递**不**镜像进目标网关会话——它们落在自己的 cron 会话中，带有 header/footer 框，以便主对话的消息角色交替保持完整。

---

## Kanban（多智能体工作队列）

一个持久的、SQLite 支撑的看板，让多个 profile / worker 在共享任务上协作。用户经由 `hermes kanban <verb>` 驱动它；由调度器生成的 worker 经由专用的 `kanban_*` 工具集驱动它，这样当它们不在看板任务内部时，schema 足迹为零。

- **CLI：** `hermes_cli/kanban.py` 接入 `hermes kanban`，动词包括 `init`、`create`、`list`（别名 `ls`）、`show`、`assign`、`link`、`unlink`、`comment`、`attach`、`attachments`、`attach-rm`、`complete`、`block`、`unblock`、`archive`、`tail`，以及较少使用的 `watch`、`stats`、`runs`、`log`、`assignees`、`heartbeat`、`notify-*`、`dispatch`、`daemon`、`gc`。
- **Worker/orchestrator 工具集：** `tools/kanban_tools.py` 暴露 `kanban_show`、`kanban_complete`、`kanban_block`、`kanban_heartbeat`、`kanban_comment`、`kanban_create`、`kanban_link`、`kanban_attach`、`kanban_attach_url`、`kanban_attachments`；在调度器生成的任务之外显式启用 `kanban` 工具集的 profile 还会获得 `kanban_list` 和 `kanban_unblock` 用于看板路由。
- **调度器：** 一个长生命周期的循环，（默认每 60s）回收陈旧声明、提升就绪任务、原子地认领，并生成被分派的 profile。默认**在网关内**运行，经由 `kanban.dispatch_in_gateway: true`。
- **插件资产：** `plugins/kanban/dashboard/`（Web UI）+ `plugins/kanban/systemd/`（`hermes-kanban-dispatcher.service` 用于独立调度器部署）。

隔离模型：
- **看板**是硬边界——worker 被生成时，其 env 中钉死了 `HERMES_KANBAN_BOARD`，使它们看不到其他看板。
- **租户**是看板*内部*的一个软命名空间——一个专家舰队可以用工作区路径 + 记忆键隔离，服务多个业务。
- 在同一个任务上连续 `kanban.failure_limit` 次（默认：2）非成功尝试之后，调度器自动封锁它以防自旋循环。

完整面向用户的文档：`website/docs/user-guide/features/kanban.md`。

---

## 重要策略

### Prompt 缓存绝不可破坏

Hermes-Agent 确保缓存在整段对话期间保持有效。**不要实现会导致以下后果的任何改动：**
- 在对话中途改变过往上下文
- 在对话中途改变工具集
- 在对话中途重新加载记忆或重建系统提示词

破坏缓存会强制大幅抬高成本。我们*唯一*改变上下文的时机是在上下文压缩期间。

会改变系统提示词状态的斜杠命令（技能、工具、记忆等）必须**感知缓存**：默认采用延迟失效（改动在下一会话生效），并提供一个 opt-in 的 `--now` 标志用于立即失效。参见 `/skills install --now` 的规范模式。

### 后台进程通知（网关）

当使用 `terminal(background=true, notify_on_complete=true)` 时，网关运行一个 watcher，检测进程完成并触发一个新的智能体轮次。用 `config.yaml` 中的 `display.background_process_notifications`（或 `HERMES_BACKGROUND_NOTIFICATIONS` 环境变量）控制后台进程消息的啰嗦程度：

- `all`——运行输出更新 + 最终消息（默认）
- `result`——仅最终完成消息
- `error`——仅当退出码 != 0 时的最终消息
- `off`——完全没有 watcher 消息

---

## 配置：多实例支持（Profile）

Hermes 支持 **profile**——多个完全隔离的实例，每个实例拥有自己的 `HERMES_HOME` 目录（配置、API key、记忆、会话、技能、网关等）。

核心机制：`hermes_cli/main.py` 中的 `_apply_profile_override()` 在任何模块导入之前设置 `HERMES_HOME`。所有 `get_hermes_home()` 引用都会自动限定到当前激活的 profile。

### 面向 profile 安全代码的规则

1. **所有 HERMES_HOME 路径都使用 `get_hermes_home()`。** 从 `hermes_constants` 导入。绝不要在读写状态的代码里硬编码 `~/.hermes` 或 `Path.home() / ".hermes"`。
   ```python
   # 好
   from hermes_constants import get_hermes_home
   config_path = get_hermes_home() / "config.yaml"

   # 坏——破坏 profile
   config_path = Path.home() / ".hermes" / "config.yaml"
   ```

2. **面向用户的消息使用 `display_hermes_home()`。** 从 `hermes_constants` 导入。它对默认返回 `~/.hermes`，对 profile 返回 `~/.hermes/profiles/<name>`。
   ```python
   # 好
   from hermes_constants import display_hermes_home
   print(f"Config saved to {display_hermes_home()}/config.yaml")

   # 坏——对 profile 显示错误路径
   print("Config saved to ~/.hermes/config.yaml")
   ```

3. **模块级常量是没问题的**——它们在导入时缓存 `get_hermes_home()`，那是在 `_apply_profile_override()` 设置环境变量*之后*。只要用 `get_hermes_home()`，而不要用 `Path.home() / ".hermes"`。

4. **mock 了 `Path.home()` 的测试也必须设置 `HERMES_HOME`**——因为代码现在用的是 `get_hermes_home()`（读取环境变量），而非 `Path.home() / ".hermes"`：
   ```python
   with patch.object(Path, "home", return_value=tmp_path), \
        patch.dict(os.environ, {"HERMES_HOME": str(tmp_path / ".hermes")}):
       ...
   ```

5. **网关平台适配器应使用 token 锁**——如果该适配器用一个唯一凭据（bot token、API key）连接，就在 `connect()`/`start()` 方法中调用 `gateway.status` 里的 `acquire_scoped_lock()`，在 `disconnect()`/`stop()` 中调用 `release_scoped_lock()`。这防止两个 profile 使用同一凭据。参见 `plugins/platforms/irc/adapter.py` 的规范模式。

6. **Profile 操作是 HOME 锚定的，而非 HERMES_HOME 锚定的**——`_get_profiles_root()` 返回 `Path.home() / ".hermes" / "profiles"`，*而不是* `get_hermes_home() / "profiles"`。这是有意为之——它让 `hermes -p coder profile list` 能看到所有 profile，无论当前激活的是哪一个。

## 已知陷阱

### 不要硬编码 `~/.hermes` 路径

代码路径使用 `hermes_constants` 中的 `get_hermes_home()`。面向用户的打印/日志消息使用 `display_hermes_home()`。硬编码 `~/.hermes` 会破坏 profile——每个 profile 都有自己独立的 `HERMES_HOME` 目录。这是 PR #3575 中修复的 5 个 bug 的根源。

### 不要引入新的 `simple_term_menu` 用法

`hermes_cli/main.py` 中既有的调用点仅为遗留兜底保留；首选 UI 是 curses（标准库），因为 `simple_term_menu` 在 tmux/iTerm2 中配合方向键有重复渲染的幽灵 bug。新的交互式菜单必须使用 `hermes_cli/curses_ui.py`——参见 `hermes_cli/tools_config.py` 的规范模式。

### 不要在 spinner/display 代码中使用 `\033[K`（ANSI 擦到行尾）

在 `prompt_toolkit` 的 `patch_stdout` 之下，它会泄漏成字面量 `?[K` 文本。使用空格填充：`f"\r{line}{' ' * pad}"`。

### `_last_resolved_tool_names` 是 `model_tools.py` 中的进程全局变量

`delegate_tool.py` 中的 `_run_single_child()` 在子智能体执行前后保存并恢复这个全局变量。如果你新增读取这个全局变量的代码，要意识到在子智能体运行期间它可能会临时失效。

### 不要在 schema 描述中硬编码跨工具引用

工具 schema 描述不得按名字提及来自其他工具集的工具（例如 `browser_navigate` 说“优先用 web_search”）。那些工具可能不可用（缺失 API key、被禁用的工具集），导致模型产生对不存在工具的幻觉调用。如果需要跨引用，请在 `model_tools.py` 的 `get_tool_definitions()` 中动态添加——参见 `browser_navigate` / `execute_code` 的后处理块的模式。

### 网关有两道消息守卫——两者都必须对审批/控制命令放行

当智能体在运行时，消息会穿过两道顺序守卫：(1) **基础适配器**（`gateway/platforms/base.py`）在 `session_key in self._active_sessions` 时把消息排队进 `_pending_messages`，以及 (2) **网关运行器**（`gateway/run.py`）在消息抵达 `running_agent.interrupt()` 之前拦截 `/stop`、`/new`、`/queue`、`/status`、`/approve`、`/deny`。任何必须在智能体被阻塞时抵达运行器的命令（例如审批提示）必须绕过*两道*守卫、以内联方式分发，而非经由 `_process_message_background()`（那会与会话生命周期竞争）。

### 来自陈旧分支的 squash 合并会静默回退最近的修复

在 squash 合并一个 PR 之前，确保该分支与 `main` 保持同步（在 worktree 中 `git fetch origin main && git reset --hard origin/main`，然后重新应用该 PR 的提交）。一个陈旧分支中某个不相关文件的版本，在被 squash 时会静默覆盖 `main` 上最近的修复。合并后用 `git diff HEAD~1..HEAD` 验证——意外的删除是红旗。

### 不要在没有 E2E 验证的情况下接进死代码

从未发布的未使用代码之所以是死代码是有原因的。在把一个未使用的模块接入活动代码路径之前，要用真实导入（而非 mock）针对一个临时的 `HERMES_HOME` 做 E2E 测试真实的解析链。

### 测试不得写入 `~/.hermes/`

`tests/conftest.py` 中的 `_isolate_hermes_home` 自动 fixture 会把 `HERMES_HOME` 重定向到一个临时目录。绝不要在测试中硬编码 `~/.hermes/` 路径。

**Profile 测试**：在测试 profile 功能时，也要 mock `Path.home()`，以便 `_get_profiles_root()` 和 `_get_default_hermes_home()` 能解析到临时目录内。使用 `tests/hermes_cli/test_profiles.py` 中的模式：
```python
@pytest.fixture
def profile_env(tmp_path, monkeypatch):
    home = tmp_path / ".hermes"
    home.mkdir()
    monkeypatch.setattr(Path, "home", lambda: tmp_path)
    monkeypatch.setenv("HERMES_HOME", str(home))
    return home
```

---

## 测试

### Python

**始终使用 `scripts/run_tests.sh`**——不要直接调用 `pytest`。该脚本强制与 CI 等价的封闭环境（清除凭据变量、TZ=UTC、LANG=C.UTF-8、`-n auto` xdist worker、树内子进程隔离插件）。在一个设置了 API key 的 16+ 核开发者机器上直接 `pytest`，会以多种曾造成多次“本地能跑、CI 挂掉”（以及反之）事故的方式偏离 CI。

```bash
scripts/run_tests.sh                                  # 完整套件，CI 等价
scripts/run_tests.sh tests/gateway/                   # 单个目录
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # 单个测试
scripts/run_tests.sh -v --tb=long                     # 透传 pytest 标志
```

**Flake 策略：** 运行器会自动在一个全新子进程中重试一次失败的测试*文件*（`--file-retries`，默认 1；`HERMES_TEST_FILE_RETRIES=0` 可禁用）。重试通过算作绿，但会打印在一个 `⚠ FLAKY` 摘要段中，包含两次尝试的输出。一份 FLAKY 报告是一个需要修复的 bug，而非可以忽略的噪音——对时序敏感的测试不得假设一个安静的运行器（宽松的挂钟边界 ≥ 2s、基于事件的同步、不要 `assert not _wait_until(...)` 这种负向时序竞态）。

#### 每测试文件的子进程隔离

每个测试文件都经由 `run_tests_parallel.py` 在一个新生成的 Python 子进程中运行。这意味着一个测试文件里的模块级 dict/set 和 ContextVar 无法泄漏到下一个。

#### 为什么需要这个包装器

|                     | 没有包装器                             | 有包装器                              |
| ------------------- | ------------------------------------------- | ----------------------------------------- |
| Provider API key   | 你环境里的任何东西（自动探测池） | 除特定几个外所有环境变量都被清除。 |
| HOME / `~/.hermes/` | 你真实的 config+auth.json                  | 每个测试一个临时目录                         |
| 时区            | 本地时区（PDT 等）                         | UTC                                       |
| 区域              | 任何被设置的                             | C.UTF-8                                   |

### 测试该放在哪里

CI 变更分类器（`scripts/ci/classify_changes.py`）根据改动的文件运行特定的作业。一个断言 `package.json`、`package-lock.json`、`.ts`/`.tsx` 源或任何其他 JS 侧产物的 Python 测试，不会在一个只触碰这些文件的 PR 上运行。这意味着一个回归可能在 PR 上变绿，却在 `main` 上变红（分类器在那里失败开放，会运行一切）。

任何读取或断言 `package.json`、`package-lock.json`、`tsconfig.json`、`.ts`/`.tsx`/`.js`/`.mjs`/`.cjs` 源文件配置的测试，都应属于 JS（vitest）测试套件，而非 `tests/*.py`。

### 不要写变更检测器式测试

一个测试如果每当*预期会变化*的数据被更新时就失败——模型目录、配置版本号、枚举计数、硬编码的 provider 模型列表——那它就是**变更检测器**。这些测试不提供任何行为覆盖；它们只是保证例行源码更新会破坏 CI，并耗费工程时间去“修复”。

**不要写：**

```python
# 目录快照——每次模型发布都会破
assert "gemini-2.5-pro" in _PROVIDER_MODELS["gemini"]
assert "MiniMax-M2.7" in models

# 配置版本字面量——每次 schema 提升都会破
assert DEFAULT_CONFIG["_config_version"] == 21

# 枚举计数——每次新增一个技能/provider 就会破
assert len(_PROVIDER_MODELS["huggingface"]) == 8
```

**要写：**

```python
# 行为：目录的管道机制到底工不工作？
assert "gemini" in _PROVIDER_MODELS
assert len(_PROVIDER_MODELS["gemini"]) >= 1

# 行为：迁移是否把用户的版本提升到当前最新？
assert raw["_config_version"] == DEFAULT_CONFIG["_config_version"]

# 不变关系：没有任何仅计划的模型泄漏进遗留列表
assert not (set(moonshot_models) & coding_plan_only_models)

# 不变关系：目录中的每个模型都有上下文长度条目
for m in _PROVIDER_MODELS["huggingface"]:
    assert m.lower() in DEFAULT_CONTEXT_LENGTHS_LOWER
```

规则：如果一个测试读起来像当前数据的快照，删掉它。如果它读起来像关于两段数据必须如何关联的契约，保留它。当一个 PR 新增一个 provider/模型且你想要一个测试时，让该测试断言那个关系（例如“目录条目都有上下文长度”），而非具体的名字。

审查者应拒绝新的变更检测器式测试；作者应在重新请求审查之前把它们转换成不变关系。

### 绝不要在测试中读取源代码

读取源文件文本的测试，测试的是*源代码的形状*，而非其行为。这是一个硬性反模式，被彻底禁止。任何读取 `.py`、`.ts`、`.tsx` 等文件的测试都可疑。

**为什么它是有害的，而非仅仅是虚弱的：**

- 当实现被微妙地破坏时它反而通过（正则匹配到一个存在但接线错误的调用点），而当一次正确的重构改变了格式、变量名或控制流（运行时行为完全相同）时它反而失败。两个方向的失败都是错的。
- 它无法针对一个构建后/打包后/压缩后的产物运行，因此一旦代码移动、被重命名、或某个依赖重新格式化了它，它就会静默地停止测试任何东西。
- 它积极地阻碍重构：审查者看到“保持某个模式完好”的测试在一次纯粹结构性清理（没有任何行为变化）时失败，要么对失败睁一只眼闭一只眼（危险），要么浪费时间去更新那些毫无增益的正则。
- 它带来虚假的自信。一个满是源正则测试的绿色套件看起来像覆盖，却从未执行过它声称要守护的代码路径一次。

**不要写：**

```ts
const source = fs.readFileSync(path.join(__dirname, 'main.ts'), 'utf8')

test('backend spawn hides the Windows console', () => {
  assert.match(source, /spawn\(\s*backend\.command,\s*backend\.args[\s\S]{0,300}hiddenWindowsChildOptions/)
})
```

**要写——把逻辑抽取成一个小的纯函数 / 可 DI 测试的纯函数并真实调用它：**

```ts
// backend-spawn.ts
export function hiddenWindowsChildOptions(options: SpawnOptionsLike = {}, isWindows = process.platform === 'win32') {
  if (!isWindows || 'windowsHide' in options) return options
  return { ...options, windowsHide: true }
}

// backend-spawn.test.ts
test('windowsHide defaults to true on Windows, is left alone elsewhere', () => {
  assert.equal(hiddenWindowsChildOptions({}, true).windowsHide, true)
  assert.equal(hiddenWindowsChildOptions({}, false).windowsHide, undefined)
  assert.equal(hiddenWindowsChildOptions({ windowsHide: false }, true).windowsHide, false)
})
```

如果逻辑内联在一个 god-file（`main.ts`、`cli.py`、`gateway/run.py`）里，抽取它看起来有破坏性：那恰恰是一个“该去抽取它”的真实信号，而不是绕着它写正则。
