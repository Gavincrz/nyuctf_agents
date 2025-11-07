# nyuctf_agents 项目流程与多智能体架构解析

## 一、项目定位与整体架构概览
- **项目目标**：`nyuctf_agents` 致力于让大型语言模型在受控 Docker 环境中自动化攻破 NYU CTF 数据集中的挑战，覆盖单智能体与多智能体两种求解模式。`run_dcipher.py` 启动的 Planner-Executor 流水线通过协作式智能体分工来完成复杂任务。【F:run_dcipher.py†L1-L117】
- **多智能体设计动机**：Planner 负责策略规划并选择工具或下游执行者，Executor 负责落地执行指令并汇总结果，AutoPrompter 可选地根据初始探查自动生成首轮提示，从而减少人工提示工程开销。【F:nyuctf_multiagent/agent.py†L182-L365】【F:nyuctf_multiagent/agent.py†L18-L118】

## 二、代码结构总览
- `nyuctf_multiagent/`：多智能体核心实现，包含 Agent 抽象、环境与工具封装、Prompt 管理以及与 LLM 提供商的后端适配层。【F:nyuctf_multiagent/__init__.py†L1-L1】【F:nyuctf_multiagent/agent.py†L1-L471】
- `configs/dcipher/`：按 CTF 分类拆分的多智能体运行配置与 Prompt 模板，控制各角色模型、温度、工具集及提示模板路径。【F:configs/dcipher/crypto_planner_executor.yaml†L1-L34】【F:configs/dcipher/prompts/base_planner_prompt.yaml†L1-L32】
- `nyuctf_multiagent/tools/`：封装 `run_command`、`delegate`、`finish_task` 等工具，提供给智能体在函数调用模式下触发环境行为。【F:nyuctf_multiagent/tools/__init__.py†L1-L15】【F:nyuctf_multiagent/tools/run_command.py†L1-L53】
- `nyuctf_multiagent/backends/`：适配不同模型供应商（OpenAI、Anthropic、Together、Gemini）的调用格式与费用计算，统一提供 `send()` 与工具参数解析能力。【F:nyuctf_multiagent/backends/__init__.py†L1-L8】【F:nyuctf_multiagent/backends/openai_backend.py†L1-L88】
- `nyuctf_multiagent/prompting.py`：`PromptManager` 读取 YAML 模板并注入挑战上下文，提供动态 Prompt 拼装接口。【F:nyuctf_multiagent/prompting.py†L1-L24】
- `run_dcipher.py`：多智能体模式入口脚本，负责解析命令行、加载配置、初始化环境、智能体与后端，再驱动整体流程。【F:run_dcipher.py†L1-L117】

## 三、核心组件与主要 Agent 角色
### 1. AutoPromptAgent
- **定义位置**：`nyuctf_multiagent/agent.py` 中 `AutoPromptAgent`。【F:nyuctf_multiagent/agent.py†L182-L279】
- **职责**：通过 `PromptManager` 绑定的 `autoprompt_prompt.yaml` 模板，引导模型使用 `generate_prompt` 工具探索挑战并生成 Planner 的初始提示。当配置启用自动提示时优先运行，若成功写回 `autoprompt` 字段将覆盖 Planner 初始 User Prompt。【F:nyuctf_multiagent/agent.py†L210-L244】【F:nyuctf_multiagent/agent.py†L307-L325】
- **上下游**：运行于流程最前，输出给 Planner；若返回 `GenAutoPromptTool` 结果，Planner 的首条用户消息即采用该自动化提示。【F:nyuctf_multiagent/agent.py†L307-L325】

### 2. PlannerAgent
- **定义位置**：`nyuctf_multiagent/agent.py` 中 `PlannerAgent`。【F:nyuctf_multiagent/agent.py†L281-L351】
- **职责**：依托 `base_planner_prompt.yaml` 系统提示，负责解析挑战、制定分步计划并调用 `delegate`、`run_command` 等工具。收到 `delegate` 调用后会把 `ToolCall` 暂存于 `self.delegated_task` 以触发 Executor。【F:nyuctf_multiagent/agent.py†L301-L343】
- **上下游**：从 AutoPrompter 或配置模板获得初始用户消息；向环境请求工具执行或调度 Executor；接收 Executor 的 `ToolResult` 作为 Observation 并继续规划。【F:nyuctf_multiagent/agent.py†L324-L365】

### 3. ExecutorAgent
- **定义位置**：`nyuctf_multiagent/agent.py` 中 `ExecutorAgent`。【F:nyuctf_multiagent/agent.py†L353-L471】
- **职责**：根据 Planner 下发的任务描述使用 `run_command`、`disassemble`、`decompile`、`create_file` 等工具完成操作，并最终以 `finish_task` 工具提交总结，或在结束后补发 `finish_summary` 用户提示获取总结文本。【F:nyuctf_multiagent/agent.py†L401-L459】
- **上下游**：由 `PlannerExecutorSystem.run_executor()` 创建与驱动，输出的 summary 以 `ToolResult` 形式返回 Planner；若执行错误则向 Planner 回传错误提示模板。【F:nyuctf_multiagent/agent.py†L418-L469】

### 4. PlannerExecutorSystem
- **定义位置**：`nyuctf_multiagent/agent.py` 末尾。【F:nyuctf_multiagent/agent.py†L473-L565】
- **职责**：多智能体的 orchestrator，管理挑战容器生命周期、成本统计、AutoPrompt 与 Planner/Executor 轮次调度、日志记录。负责在 Planner 发出 `delegate` 后启动新的 Executor 会话，并把执行总结写回 Planner 对话。【F:nyuctf_multiagent/agent.py†L495-L563】

## 四、主流程：从一次任务输入到最终输出的执行路径
### 4.1 入口点
1. 用户通过 `run_dcipher.py` 指定挑战名、模型、配置等，脚本加载数据集并构造 `CTFChallenge` 对象。【F:run_dcipher.py†L1-L63】
2. 创建 `CTFEnvironment`，准备 Docker 容器及可用工具集合；随后加载 YAML 配置生成 `Config`，注入模型与 Prompt 设置。【F:run_dcipher.py†L65-L111】【F:nyuctf_multiagent/environment.py†L1-L68】
3. 根据配置实例化 AutoPrompter、Planner、Executor 三类 Agent，均共享 `PromptManager` 注入的挑战上下文与工具集。【F:run_dcipher.py†L89-L117】【F:nyuctf_multiagent/prompting.py†L1-L24】

### 4.2 任务分解与调度逻辑
- `PlannerExecutorSystem.run()` 是 orchestrator 主循环：先运行 AutoPrompter（若启用），再进入 Planner 回合；Planner 每次调用 `delegate` 时触发 `run_executor()` 启动新的 Executor 对话。系统根据 `max_cost`、`max_rounds` 与环境状态控制流程终止。【F:nyuctf_multiagent/agent.py†L495-L563】
- 成本累计通过 `total_cost()` 汇总 Planner、Executor、AutoPrompter 的 token 花费，超过上限或环境 solved/giveup 时结束。【F:nyuctf_multiagent/agent.py†L517-L533】

### 4.3 多智能体交互时序
1. **初始化 Prompt**：AutoPrompter 与 Planner/Executor 都调用 `PromptManager.get()` 读取 `system` 与 `initial` 模板，通过 `format()` 注入挑战描述、服务器信息、工具说明等变量。【F:nyuctf_multiagent/agent.py†L200-L223】【F:nyuctf_multiagent/prompting.py†L11-L23】【F:configs/dcipher/prompts/base_planner_prompt.yaml†L1-L32】
2. **AutoPrompt 探查**：若启用，`PlannerExecutorSystem.run_autoprompter()` 循环调用 `AutoPromptAgent.run_one_round()`。模型可使用 `run_command` 探索文件或服务，当调用 `generate_prompt` 工具时保存生成的 Planner 初始提示，并在 `finished` 标记后退出。【F:nyuctf_multiagent/agent.py†L515-L540】【F:nyuctf_multiagent/agent.py†L210-L244】【F:nyuctf_multiagent/tools/misc.py†L57-L83】
3. **Planner 规划**：系统输出“PLANNER”标头后，将 AutoPrompter 结果或默认 `initial` Prompt 写入对话。每轮通过后端 `send()` 生成回复，如果产生 `delegate` 工具调用则暂存于 `self.delegated_task`，否则可直接运行命令或提交 Flag。【F:nyuctf_multiagent/agent.py†L541-L563】【F:nyuctf_multiagent/agent.py†L301-L343】【F:configs/dcipher/prompts/base_planner_prompt.yaml†L17-L30】
4. **Executor 执行**：`run_executor()` 新建 `ExecutorAgent`（保持模型与 Prompt 设置），把 Planner 提供的任务描述格式化进 `initial` 模板。Executor 调用工具时，`CTFEnvironment.run_tool()` 将请求路由到具体工具实现（如 `RunCommandTool.call()` 使用 `docker exec` 执行命令），并将输出截断后写入对话的 Observation。【F:nyuctf_multiagent/agent.py†L545-L563】【F:nyuctf_multiagent/environment.py†L58-L84】【F:nyuctf_multiagent/tools/run_command.py†L1-L53】【F:configs/dcipher/prompts/base_executor_prompt.yaml†L20-L37】
5. **任务收尾**：当 Executor 使用 `finish_task` 工具提交总结，或 orchestrator 额外提示 `finish_summary` 获取补充说明后，`run_executor()` 将总结文本封装为 `ToolResult` 返回 Planner。Planner 在下一轮收到 Observation 后继续规划或结束挑战。【F:nyuctf_multiagent/agent.py†L401-L459】【F:nyuctf_multiagent/agent.py†L563-L565】【F:configs/dcipher/prompts/base_executor_prompt.yaml†L38-L45】
6. **终止条件**：若 `SubmitFlagTool` 成功或 `GiveupTool` 被触发，`CTFEnvironment` 标记 `solved`/`giveup`，`PlannerExecutorSystem` 跳出循环并生成日志文件，其中包含每个会话的消息历史与费用统计。【F:nyuctf_multiagent/tools/misc.py†L5-L54】【F:nyuctf_multiagent/agent.py†L507-L513】

## 五、Prompt 体系与动态组装机制详解
### 5.1 Prompt 定义位置
- 所有多智能体模板位于 `configs/dcipher/prompts/`，按角色与挑战类别划分，如 `base_planner_prompt.yaml`、`crypto_executor_prompt.yaml`。模板内提供 `system`、`initial`、`continue`、服务器描述等段落，可用 `.format()` 占位符动态注入上下文。【F:configs/dcipher/prompts/base_planner_prompt.yaml†L1-L32】【F:configs/dcipher/prompts/base_executor_prompt.yaml†L1-L47】
- AutoPrompter 使用 `autoprompt_prompt.yaml`，强调使用 `generate_prompt` 工具输出新的 Planner 初始提示。【F:configs/dcipher/prompts/autoprompt_prompt.yaml†L1-L31】

### 5.2 Prompt 使用方式
- `PromptManager.get(key, **kwargs)` 会读取 YAML 中的键并通过 `format()` 替换 `{challenge.*}`、`{environment.container_home}`、`{task_description}` 等占位符，返回最终字符串供智能体写入 Conversation。【F:nyuctf_multiagent/prompting.py†L11-L23】
- Planner 仅在 `delegate` 前手动注入任务描述；Executor 的 `initial` Prompt 通过 `run_executor()` 时传入 `task_description` 参数动态填充；AutoPrompter 在补发 `finish_autoprompt` 时同样调用模板。【F:nyuctf_multiagent/agent.py†L545-L563】【F:nyuctf_multiagent/agent.py†L329-L341】
- `continue`、`finish_summary`、`finish_autoprompt` 等键作为系统内置追加提示，分别在模型未调用工具、执行总结不足或 AutoPrompter 未生成结果时触发。【F:nyuctf_multiagent/agent.py†L223-L244】【F:nyuctf_multiagent/agent.py†L401-L447】

### 5.3 动态生成与条件分支
- AutoPrompter：若在 `max_rounds` 内未调用 `generate_prompt`，orchestrator 触发 `run_for_autoprompt()`，使用 `finish_autoprompt` 提示迫使模型生成结果；若仍失败则回退至默认 Planner Prompt。【F:nyuctf_multiagent/agent.py†L246-L279】【F:nyuctf_multiagent/agent.py†L307-L325】
- Executor：若未调用 `finish_task` 结束，`run_executor()` 会追加 `finish_summary` 提示；若产生错误，返回 `finish_error` 模板填充错误信息；完全无总结时返回 `finish_empty`。【F:nyuctf_multiagent/agent.py†L434-L469】【F:configs/dcipher/prompts/base_executor_prompt.yaml†L38-L45】
- Planner：若 Planner 未找到可行动作会继续发送 `continue` 提示促使模型给出下一步工具调用或计划。【F:nyuctf_multiagent/agent.py†L301-L343】

## 六、工具调用与环境交互
- 工具由 `CTFEnvironment` 在初始化时批量实例化，保存在 `self.tools` 中。每次模型产生函数调用后经 `Backend.parse_tool_arguments()` 校验参数并转发给 `environment.run_tool()` 执行，返回 `ToolResult` 写入会话。【F:nyuctf_multiagent/environment.py†L10-L57】【F:nyuctf_multiagent/backends/backend.py†L37-L86】
- 典型工具：
  - `run_command`：调用 `docker exec` 在挑战容器内执行命令，返回 stdout/stderr/状态码并截断输出。【F:nyuctf_multiagent/tools/run_command.py†L20-L53】
  - `delegate`：Planner 特有，触发 orchestrator 启动 Executor 并把任务描述转成下游初始 Prompt。【F:nyuctf_multiagent/tools/misc.py†L39-L56】【F:nyuctf_multiagent/agent.py†L545-L563】
  - `finish_task`：Executor 提交任务总结，结束自身会话并把摘要返回 Planner。【F:nyuctf_multiagent/tools/misc.py†L85-L111】【F:nyuctf_multiagent/agent.py†L401-L447】
  - `submit_flag` / `giveup`：改变 `CTFEnvironment` 的 `solved`/`giveup` 状态，驱动整体流程收敛。【F:nyuctf_multiagent/tools/misc.py†L5-L54】

## 七、典型执行 Trace（代码推演）
1. `run_dcipher.py > PlannerExecutorSystem.run()`：解析配置、启用 AutoPrompter，并打印“PLANNER”标题。【F:run_dcipher.py†L89-L117】【F:nyuctf_multiagent/agent.py†L495-L563】
2. `AutoPromptAgent.run_one_round()`：模型执行 `run_command` 探索文件，随后调用 `generate_prompt` 返回新 Prompt；`PlannerExecutorSystem` 用该字符串覆盖 Planner 的初始 User Prompt。【F:nyuctf_multiagent/agent.py†L210-L244】【F:nyuctf_multiagent/tools/misc.py†L57-L83】
3. `PlannerAgent.run_one_round()`：Planner 在第一轮调用 `delegate`，提供详细任务描述；`planner.delegated_task` 存储该调用。【F:nyuctf_multiagent/agent.py†L301-L343】
4. `PlannerExecutorSystem.run_executor()`：读取 `delegated_task.parsed_arguments['task']`，将其插入 Executor `initial` Prompt，创建新的 `ExecutorAgent` 会话。【F:nyuctf_multiagent/agent.py†L545-L563】
5. `ExecutorAgent.run_one_round()`：Executor 执行 `run_command`，Observation 写回 stdout/stderr；执行完毕后调用 `finish_task`，总结中附带可能的 flag 片段。【F:nyuctf_multiagent/agent.py†L401-L447】【F:nyuctf_multiagent/tools/run_command.py†L20-L53】
6. `PlannerAgent` 收到 `ToolResult` 后在下一轮更新计划，若 summary 包含 flag 则调用 `submit_flag`；成功后 `CTFEnvironment.solved=True`，系统退出并记录日志。【F:nyuctf_multiagent/tools/misc.py†L5-L33】【F:nyuctf_multiagent/agent.py†L507-L513】

## 八、扩展与自定义建议
- **新增 Agent**：可继承 `BaseAgent` 并实现 `run_one_round()`，随后在 orchestrator 中实例化并插入交互逻辑，例如在 `PlannerExecutorSystem` 中于 Planner 与 Executor 之间添加验证环节。【F:nyuctf_multiagent/agent.py†L12-L118】
- **自定义 Prompt**：在 `configs/dcipher/prompts/` 添加新的 YAML 模板，更新对应配置文件的 `prompt` 路径，使 `PromptManager` 自动加载。模板占位符可引用 `challenge`、`environment`、或 `PromptManager` 计算出的 `server_description`。【F:nyuctf_multiagent/prompting.py†L11-L23】
- **扩展工具**：在 `nyuctf_multiagent/tools/` 新增工具类并加入 `ALLTOOLS`，配置文件的 `toolset` 列表即可允许模型调用该工具；注意在工具实现中覆写 `print_tool_call` 以保持日志可读性。【F:nyuctf_multiagent/tools/__init__.py†L1-L15】
- **替换模型后端**：实现新的 `Backend` 子类并加入 `BACKENDS` 列表，或直接在配置里选择现有后端提供的模型名称；`MODELS` 字典自动映射模型字符串到后端类型。【F:nyuctf_multiagent/backends/__init__.py†L1-L8】

## 九、附录：关键文件速查表
- `run_dcipher.py`：多智能体入口脚本，加载挑战与配置后启动 orchestrator。【F:run_dcipher.py†L1-L117】
- `nyuctf_multiagent/agent.py`：定义 BaseAgent、AutoPrompter、Planner、Executor 及 `PlannerExecutorSystem` 主流程。【F:nyuctf_multiagent/agent.py†L1-L565】
- `nyuctf_multiagent/prompting.py`：`PromptManager` 实现，负责读取模板并动态格式化 Prompt。【F:nyuctf_multiagent/prompting.py†L1-L24】
- `nyuctf_multiagent/environment.py`：封装 Docker 环境生命周期与工具调度。【F:nyuctf_multiagent/environment.py†L1-L84】
- `nyuctf_multiagent/backends/openai_backend.py`：示例后端实现，演示消息格式转换与费用计算。【F:nyuctf_multiagent/backends/openai_backend.py†L1-L88】
- `configs/dcipher/prompts/*.yaml`：多智能体 Prompt 模板仓库，按角色与题型组织。【F:configs/dcipher/prompts/base_executor_prompt.yaml†L1-L47】
- `configs/dcipher/*_planner_executor.yaml`：多智能体运行配置，声明模型、工具集、Prompt 路径与成本约束。【F:configs/dcipher/crypto_planner_executor.yaml†L1-L34】
- `nyuctf_multiagent/tools/`：模型可调用的函数工具集，实现命令执行、文件写入、任务总结等功能。【F:nyuctf_multiagent/tools/run_command.py†L1-L53】
