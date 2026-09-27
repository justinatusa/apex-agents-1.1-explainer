# 03 1.1 的代理循环

1.1 榜上用的参考代理叫 `apex_loop_truncated_tools_agent`，类名 `ApexLoopTruncatedToolsAgent`。作者在 README 里的自述是：一个很小的 LiteLLM 工具调用循环，用的是世界里的 MCP 工具；工具输出截到 200 行或 32K 字符；默认 100 步（他们也叫 turns），超时 10800 秒，与榜上的运行一致。

下面描述的是配套仓库 commit `1ec75805084ca5b8aff1d6411e9e3cef03a46947` 里的 `runner_src`。Harbor 插件把这份 runner 放进 main 容器再执行。公开 Archipelago 同名代理在 `1ca0d41` 上多了上下文裁剪等旋钮，1.1 插件不会把那些键写进配置，本篇末尾单独说明。上一版另一种循环（toolbelt、`final_answer` 工具）只在 [01 的对照节](01-what-is-apex-1.1.md)，不要把那里的步数套到这份源码上。

这份源码的结束条件是：模型这一次回复里没有任何工具调用。日志类型 `final_answer` 只是把这段文字记下来，不是一个要调用的工具。

## 预算这几个数

| 配置键 | 1.1 默认 | 是什么 | 不是什么 |
| --- | --- | --- | --- |
| `max_steps` | 100 | 循环最多叫多少次模型。数据集卡称 100 turns。一步 = 一次模型回复，加上这次带出的工具结果 | 不是工具调用总次数。一次回复里可以有多个 tool call，仍只占一步 |
| `timeout` | 10800 秒 | 整次 `run()` 的墙钟，3 小时。装载工具和全部步骤都算在内 | 不是单次模型等待，也不是 Archipelago 示例 `.env` 里的 22.5 小时 |
| `llm_response_timeout` | 600 秒 | 单次模型请求的等待上限。内部最多 10 次尝试 | 不是 100 步各自再加 600 秒的预算；外层 10800 秒先到就整次结束 |
| `tool_call_timeout` | 60 秒 | 单个 MCP 工具调用的等待上限 | 不是一步的上限。同一步后面的工具继续用各自的 60 秒 |
| `max_output_lines` | 200 | 工具输出按行截断的上限，和字符上限谁先到用谁 | 不是对话总行数 |
| `max_output_chars` | 32768 | 工具输出的字符上限 | 不是 token 数 |

插件可用环境变量覆盖这六个数：`MAX_STEPS`、`AGENT_TIMEOUT_SEC`、`MAX_OUTPUT_CHARS`、`MAX_OUTPUT_LINES`、`TOOL_CALL_TIMEOUT`、`LLM_RESPONSE_TIMEOUT`。官方 README 的命令没有设置它们。数据集卡写：换步数预算，结果可能不能和榜比较。

Harbor 执行 runner 的外层超时是 `timeout + 900` 秒。默认即 11700 秒。这 900 秒是写轨迹和收尾的余量，不是模型多出来的思考时间。

## 一次 MCP 工具调用

工具不在代理进程里实现。代理只拿着网关地址。1.1 插件默认是 `http://world:8000/mcp/`，传输是 `streamable-http`。有 actor id 或 token 时才加 `Authorization` 头；`runner_cli` 把 token 传成 `None`，所以这条官方入口不自己签 Bearer。

调用分成「开跑前一次」和「每一步里若干次」：

1. **开跑。** `_initialize_tools` 用 FastMCP 客户端连上网关，`load_mcp_tools(..., format="openai")` 把当前列出的工具收成 OpenAI 风格的函数定义。日志会打出每个工具名。这一步失败（网关还没起来、URL 错了）会在进入 100 步之前就结束。没有网关 URL 时，构造函数直接抛 `MCP gateway URL is required`。
2. **模型决定。** `generate_response` 把迄今的消息和这份工具表一起送给 LiteLLM。模型可以在同一次回复里给出多个 `tool_calls`。
3. **逐个执行。** 源码是 `for tool_call in tool_calls`，一个接一个，不是并行。每个调用走 `call_openai_tool(client.session, tool_call)`，外面套 `asyncio.wait_for(..., timeout=tool_call_timeout)`。
4. **结果变成消息。** `content_blocks_to_messages` 把 MCP 的文本块和图片块收成恰好一条 `role: tool` 的消息，带上 `tool_call_id`。模型名以 `anthropic/` 开头时，图片可以留在这条工具消息里。其他模型的工具消息放不下图片，图片改成这一步全部工具都写完之后再附上的用户消息。
5. **截断。** `_truncate_tool_message` 只改 `role == tool` 的内容。先按 200 行，再按 32768 字符。截断后的文本才进入后续步骤。用法统计记的也是截断后的文本，源码注释写明：这是模型实际被计费看到的那一份。
6. **这一步没有工具调用。** 循环把 `_finalized` 设为真，用 `logger.bind(message_type="final_answer")` 记下助手文字。文字为空就记 `No content`。

工具侧的失败不会都变成整次崩溃：

- 超时：写入 `Tool call timed out`，继续下一个工具。底层调用先被 shield 住，再给 5 秒（`SHIELDED_TASK_GRACE_SECONDS`），还没结束就取消，避免占着同一条 MCP 会话。
- 普通错误：把 `Error calling tool: ...` 交回模型，循环继续。
- 致命错误：MCP 错误码 32600 或 -32600、报文字样 `Session terminated`，或 FastMCP 的 `Client is not connected`。这条写入对话后立刻结束整次运行。
- 空的工具结果：写入「结果无效」，继续。不把它当成已经做完。

## 一步里面发生什么

一次运行先有两条消息：系统提示词，然后是这道题的用户说明。接着进入循环。循环的上限是 `max_steps`，默认 100。每一步是一次模型调用，加上这次调用带出来的工具结果。一步里可以有多次工具调用，源码里是按顺序逐个执行，不是并行。

一步的顺序：

1. 把到目前为止的消息和全部工具定义发给模型。单次等待上限是 `llm_response_timeout`，默认 600 秒。这一层内部最多尝试 10 次（`with_retry` 的 `max_retries=10`，`range(1, max_retries + 1)`）。可重试的错误包括限流、超时、部分 400、服务不可用、连接错误、500、502。上下文超长，以及提示词里被判定为「再试也一样」的请求错误，不重试，直接抛出。退避是 `5 * 2^(第几次 - 1)` 秒，再加 0 到 5 秒的随机抖动。
2. 若这 10 次都因为超时失败，这一步记一条错误日志，然后进入下一步，不往对话里追加消息。
3. 若模型返回空的 choices，追加一条用户消息，内容是 `continue`，这一步结束。
4. 把助手回复放进对话。如果其中有工具调用，按顺序执行。每个工具调用的上限是 `tool_call_timeout`，默认 60 秒。
5. 工具成功则把结果写回，并按 200 行或 32768 字符截断，先碰到哪个用哪个。截断后的文本才是后面步骤会再送给模型的内容。
6. 工具超时：写入一条内容为 `Tool call timed out` 的工具消息，继续下一个工具。被屏蔽的底层调用再给 5 秒（`SHIELDED_TASK_GRACE_SECONDS`），还没结束就取消，避免占着 MCP 会话。
7. 工具抛出致命 MCP 错误：会话已死、客户端已断开，或 JSON-RPC 错误码 32600 / -32600。这条错误写入对话后，整次运行结束。
8. 其他工具错误：把错误文本交回给模型，循环继续。模型还有机会换一种叫法。
9. 若这次助手回复里没有工具调用，循环把 `_finalized` 设为真。助手的文字被记成 `final_answer` 日志。没有文字就记 `No content`。

`final_answer` 在这份代码里是日志的 `message_type`，不是一个工具。模型不需要、也没有被要求去调用名叫 `final_answer` 的函数。它只要停止调用工具并写出文字（或什么都不写）。

图片有一个分叉：模型名以 `anthropic/` 开头时，图片可以留在工具结果里；其他模型的工具结果放不下图片，代码会把图片改成后续的用户消息，等这一步所有工具结果都写完再附上。

## 整次运行何时停

外层还有一个墙钟，`asyncio.timeout`，默认 10800 秒，也就是 3 小时。它包住装载工具和全部步骤。单步的 600 秒和 60 秒都算在这 3 小时里面。Harbor 执行这条命令时再多给 900 秒，避免代理刚写完轨迹、外层先被杀掉。

| 发生了什么 | 轨迹状态 | 这意味着什么 |
| --- | --- | --- |
| 某一步的回复里没有工具调用 | `completed` | 代理自己停了。文字进入 `final_answer` 日志 |
| 走满 `max_steps` 仍没有这种回复 | `failed` | 源码打错误日志：「100 步后仍未结束」。进程仍会把轨迹写到磁盘，`runner_cli` 不因为这个状态改退出码 |
| 墙钟超过 `timeout` | `error` | 3 小时到了 |
| 运行被取消 | `cancelled` | 外部取消 |
| 上下文超长等被 `is_system_error` 判成模型侧问题 | `failed` | 再跑一次同样会失败的那类 |
| 限流、连接失败、5xx 等被判成系统问题 | `error` | 源码把它和模型答错分开 |
| MCP 会话死亡 | 先抛致命错误，再按上面的系统/模型分类落入 `error` 或 `failed` | 工具通道断了 |
| runner 进程非 0 退出 | Harbor 插件抛 `RuntimeError` | 插件注释：应当让试验报错，而不是把空作业送去打分 |
| 容器里根本没有模型密钥 | 插件在开跑前抛 `ValueError` | 预检，避免空转到 3 小时 |

数据集卡写：`max_steps = 100` 且 `agent_timeout_sec = 10800` 才和榜上运行一致。换一个修订版或换步数预算，行为会变，结果可能不能和公布的榜比较。

插件允许用环境变量覆盖这六个数：`MAX_STEPS`、`AGENT_TIMEOUT_SEC`、`MAX_OUTPUT_CHARS`、`MAX_OUTPUT_LINES`、`TOOL_CALL_TIMEOUT`、`LLM_RESPONSE_TIMEOUT`。非数字会被警告并忽略。这些变量是源码能力。官方 README 的运行命令没有设置它们。要和榜比较，就保持默认。

还有一个源码里的可选变量 `APEX_MODEL_EXTRA_ARGS`。它应当是 JSON，会作为 LiteLLM 的额外参数传给模型。1.1 的 `.env.example` 没有这一行。温度、推理力度之类，官方复现命令也没有写死。旧论文附录 E 把每个旧模型的 temperature 和 max tokens 列了表，那张表是旧版八个模型的配置，不是 1.1 的。

`runner_cli` 不读取 Archipelago 设置类里那个 22.5 小时的 `AGENT_TIMEOUT_SECONDS`。1.1 实际生效的墙钟是插件放进 `agent_config.json` 的 `timeout`，默认 10800。看到 81000 或「22 小时 30 分」时，那是 Archipelago agents 示例 `.env` 里的另一个旋钮。

## 系统提示词

提示词写在 `apex_loop_truncated_tools_agent.py` 的 `_AGENT_SYSTEM_PROMPTS["loop_truncated_tools_agent"]`。`manifest.json` 记录的 `system_prompt_sha256` 是 `dcfab3c1f685305d98fc2aa7cf72f275b158494a453f5917f1f0700d21ca69bb`。对源码里的这条字符串做 SHA-256，结果相同。

下面是这条字符串按空行分段后的正文。源码在 “clarification.” 后面有一个空格，再接换行。哈希针对的是源码字符串，不是下面的排版。

```text
You are an agent that completes tasks independently. Use the tools and files provided to you to complete the task to the best of your ability. You should use the code_exec tool when needed, such as when calculating values. When calculating numbers, unless specified otherwise, use the exact values without rounding them.

You must attempt to execute the task. You cannot ask for help or further clarification.

You should not scattergun your answers. Scattergunning is when you provide alternate answers based on information that is not explicitly requested in the task prompt. If you do this, it will be marked wrong. Please note, providing answers to legitimately different cases that are explicitly requested by the prompt is NOT scattergunning. Please respond to all parts of the task, but commit to answers instead of hedging.

You do not have access to the internet. Do not try to look up the answer through requests, beautifulsoup, or any other packages.

For every tool except the code_exec tool, you may assume that all relevant files are located under the root path /. For the code_exec tool, however, you must explicitly use /filesystem/ as the root path to locate all relevant files.
```

逐段它在约束什么：

- 独立做完。算数用 `code_exec`。没说要四舍五入时，用精确值。
- 必须动手。不能回头问人要澄清。
- 不要 scattergun。定义写在提示词里：任务说明没有明确要求备选答案时，却给出备选答案。提示词说这样做会被判错。任务真的要求分别回答几种情形，不算 scattergun。要回答所有部分，但每个部分要选定答案，不要留退路（hedging）。
- 没有互联网。不要用 `requests`、BeautifulSoup 或其他包去查答案。
- 路径：普通工具用 `/`，`code_exec` 用 `/filesystem/`。

博客写过一句：模型在系统提示词里被告知，上下文信息可以在世界文件里找到，包括沟通记录。配套仓库里这条已经对上哈希的提示词，没有这句话的原文。它只说 “use the tools and files provided to you”。博客的那句更像对行为的转述。以哈希对得上的字符串为准，不把博客的转述补进提示词。

反 hedging 就在第三段，不在别的隐藏提示里。模型被要求：任务说明没有明确要求备选答案时，不要给备选；说明真的要求分情形作答，那不算 scattergunning；每个部分都要答，但要选定答案。博客说这样做是为了让「同时抛出好几个数」不再靠「正确信息只要出现过」就过条目。Judge 侧把这种条目打成 0 的规则在 [01](01-what-is-apex-1.1.md)。上一版提示词没有这段，差异只列在 01 的对照节。

## 截断在截什么

函数 `_truncate_output` 先按行，再按字符。默认 200 行、32768 字符。截断后缀类似 `... (truncated N lines) ...` 或 `... (truncated N chars) ...`。只处理 `role` 为 `tool` 的消息。

32768 是字符，不是 token。Archipelago 后来的注册表注释估计，这类 JSON 大约 2.5 个字符对应 1 个计费 token，因此 32768 字符大约是一万多个 token 量级。那是后来注释里的估算，不是 1.1 卡上的数字。本说明不把它当成精确换算。

同一份后来的注册表还说：`max_output_lines` 这个旋钮对「单行 JSON 工具结果」实际上切不到，因为行数是 1，作者在 677 条生产结果里看到 0 次按行截断，于是把这个控件从界面上退休，代码默认值仍是 200。那是 Archipelago commit `1ca0d41` 的注释，统计对象是他们说的生产结果，不是本说明复跑 1.1 得到的。1.1 插件仍然会把 `max_output_lines: 200` 写进配置，截断函数也仍然先检查行数。两件事同时为真。

## 这份循环里没有的东西

启动时 `load_mcp_tools` 把网关当前工具一次装上。源码里没有「先只给元工具、用时再加入」的步骤，也没有上下文用到某个比例就摘要的逻辑。对话按步追加，靠截断工具输出控制长度。上下文超长按模型错误结束（`failed`），不会自动摘要后再试。

Archipelago 仓库里还登记着别的代理实现。1.1 的 `manifest.json` 和插件都把 id 写成 `loop_truncated_tools_agent`。官方命令不会跑到那些实现。和上一版循环的差别见 [01 的对照节](01-what-is-apex-1.1.md)。

## 更晚的 Archipelago 旋钮，不是 1.1 插件的默认

`1ca0d41` 的注册表给同名代理加了这些字段，默认大多是关的：`context_trim_utilization`（默认 0）、`context_trim_floor`（默认 0.6）、`max_step_tool_result_tokens`（默认 0）、`spill_oversized_results`（默认关）。1.1 插件生成的 `agent_config.json` 不含这些键。vendored 的 `LoopTruncatedToolsAgent.__init__` 也不读取它们。

把「后来的 Archipelago 已经会在 85% 上下文时裁掉旧的工具交换」说成 1.1 榜的行为，是错的。1.1 榜对应的是插件写死的那六个数字，加上上面的提示词。
