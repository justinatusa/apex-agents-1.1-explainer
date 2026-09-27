# 99 证据

核对日期：2026-09-27。这一页只记录「这句话从哪来」，不新增结论。

与父门闸核对过的三个远端 HEAD，和本机 `git clone --depth 1` 的结果一致：

| 材料 | revision / commit | 本机核对 |
| --- | --- | --- |
| Hugging Face `mercor/apex-agents-v1.1` | `695dac5bf1388816f97be9eaf882ef01de79d574` | Hub API 的 `sha` 字段 |
| `apex_loop_truncated_tools_agent` | `1ec75805084ca5b8aff1d6411e9e3cef03a46947` | `git rev-parse HEAD` |
| `archipelago` | `1ca0d41db0ad1fe7c8ee491c253b17c9a44d1d11` | `git rev-parse HEAD` |

上一版论文的数字只应出现在 `01-what-is-apex-1.1.md` 的「差异对照」一节。本页末尾保留论文数字，是为了回查那一节，不是第二份叙述。前面各篇若与本页的 SHA 不一致，以本页为准。

## 读到的版本

| 材料 | 标识 | 怎么核对的 |
| --- | --- | --- |
| Hugging Face 数据集 `mercor/apex-agents-v1.1` | revision `695dac5bf1388816f97be9eaf882ef01de79d574` | `GET https://huggingface.co/api/datasets/mercor/apex-agents-v1.1` 的 `sha` 字段。与任务要求的 Hub API SHA 相同 |
| 同一 API | `id = mercor/apex-agents-v1.1`，`private = false`，`gated = auto`，`siblings` 数量 4441 | 同上 |
| 数据集卡正文 | 页面 https://huggingface.co/datasets/mercor/apex-agents-v1.1 | 2026-09-27 抓取的卡片 Markdown。题量、世界、955、5182、Harbor 0.20.0、目录树、Get Started 命令都来自这里 |
| 未登录取文件 | `manifest.json` 与 `environment/images.json` | `GET .../resolve/695dac5.../manifest.json` 返回 HTTP 401，`x-error-code: GatedRepo` |
| 博客 | https://www.mercor.com/blog/introducing-apex-agents-1-1/ | 页面日期 Sep 8, 2026。作者署名为 Austin Bennett、Bertie Vidgen、Arnav Garg、Akul Datta |
| 排行榜页面 | https://www.mercor.com/apex/apex-agents-leaderboard/ | 2026-09-27 抓取。页面自己没有印快照日期 |
| 配套代理 | https://github.com/Mercor-Intelligence/apex_loop_truncated_tools_agent | `git clone --depth 1` 后的 HEAD：`1ec75805084ca5b8aff1d6411e9e3cef03a46947`，时间 2026-09-16 10:41:12 -0700，说明 “Use the exact APEX-Agents 1.1 system prompt from the Studio agent of record” |
| 代理 manifest | 仓库内 `apex_loop_truncated_tools_agent/manifest.json` | `version` 1.0.1；`source_revision` `2bb7e26b3599d9f55e7994198df45f374fe9b600`；`studio_agent_id` `agent_51d240d4a7b04f059f6472925fb83e89`；`system_prompt_sha256` `dcfab3c1f685305d98fc2aa7cf72f275b158494a453f5917f1f0700d21ca69bb` |
| 提示词哈希 | 同上 | 对本机克隆里 `_AGENT_SYSTEM_PROMPTS["loop_truncated_tools_agent"]` 的 Python 字符串做 SHA-256，与 manifest 一致 |
| Archipelago | https://github.com/Mercor-Intelligence/archipelago | `git clone --depth 1` 后的 HEAD：`1ca0d41db0ad1fe7c8ee491c253b17c9a44d1d11`，时间 2026-09-25 08:29:50 +0000，说明 “Sync archipelago from Mercor-io/studio@82ec60ca1a2c5858640f1bcdb9141a4dcb57fdd1” |
| 论文 | https://arxiv.org/abs/2601.14242 ，HTML https://arxiv.org/html/2601.14242 | 2026-09-27 抓取的 HTML。文中旧版数字（480、33、250 步、24.0%、Gemini 3 Flash judge、30,720 条轨迹、附录 D 的 toolbelt 与旧提示词）都来自这篇 |
| BibTeX | 数据集卡与配套 README 末尾 | 1.1 条目作者是 Bennett、Datta、Vidgen，没有博客署名里的 Arnav Garg。两处照抄，没有改 |

克隆使用 `--depth 1`，所以只有上述 tip commit，没有更早的历史。

## 源码路径

1.1 循环与提示词：

- `apex_loop_truncated_tools_agent/apex_loop_truncated_tools_agent.py`：Harbor 插件、系统提示词、`max_steps` 默认 100、`agent_timeout_sec` 默认 10800、网关 URL `http://world:8000/mcp/`、写进 `agent_config.json` 的六个字段、密钥预检、ATIF 转换。
- `apex_loop_truncated_tools_agent/runner_src/runner/agents/loop_truncated_tools_agent/main.py`：单步、截断、停止条件、状态枚举的赋值。
- `apex_loop_truncated_tools_agent/runner_src/runner/utils/llm.py`：`generate_response` 上的 `@with_retry(max_retries=10, base_backoff=5, jitter=5, ...)`。
- `apex_loop_truncated_tools_agent/runner_src/runner/utils/decorators.py`：`for attempt in range(1, max_retries + 1)`，因此 10 是尝试次数。
- `apex_loop_truncated_tools_agent/runner_src/runner/utils/mcp.py`：`SHIELDED_TASK_GRACE_SECONDS = 5.0`，网关用 `streamable-http`。
- `apex_loop_truncated_tools_agent/runner_src/runner/utils/error.py`：`is_system_error`、`is_fatal_mcp_error`。
- `apex_loop_truncated_tools_agent/runner_src/runner_cli.py`：`APEX_MODEL_EXTRA_ARGS`。
- `apex_loop_truncated_tools_agent/runner_src/pyproject.toml`：Python `>=3.13,<3.14`，`litellm==1.97.0`。
- `.env.example`：模型与 grader 变量名。
- `README.md`：Harbor Hub 与 Hugging Face 命令。

Archipelago，用于基础设施词汇，不用于 1.1 题量：

- `README.md`：Environment / Agents / Grading；快速开始指向 `mercor/apex-agents` 的 480 题。
- `environment/Dockerfile`：`EXPOSE 8080`，uvicorn 端口 8080。
- `environment/README.md`：`/health`、`/apps`、`/mcp/`、`/data/populate`、`/data/snapshot`；tar.gz 与 zip 的格式差。
- `agents/runner/agents/registry.py`：`LOOP_TRUNCATED_TOOLS_AGENT` 一段。其中 `max_output_lines` 被注释为对单行 JSON 不可达，并提到 677 条生产结果。这是这份 Archipelago 修订的注释，不是 1.1 插件的行为。
- `agents/runner/agents/models.py`：`LOOP_TRUNCATED_TOOLS_AGENT = "loop_truncated_tools_agent"`。
- `grading/README.md`：helper / verifier / scoring，以及 ungradeable 的三种状态。
- `grading/.env.example`：Reducto、文档缓存、Firecrawl。
- `mcp_servers/*/mcp_servers/**/main.py`：第 04 篇的工具名表，来自 `mcp.tool(名字)` 这种调用。

## 数据集卡上的原句级数字

这些数不在论文里，不要和旧版混用。

- 任务 240，每职业 80。世界 31：投行 8、咨询 11、法务 12。
- 条目 955，均值 3.98，范围 1 到 11。
- 标准答案：206 文字 + 34 文件。
- 文件 5,182，平均每世界 167.2。分职业平均：投行 169.1，法务 161.5，咨询 171.9。
- 带输入文件的任务 63（26.3%）：投行 4（5.0%），法务 28（35.0%），咨询 31（38.8%）。
- 文件输出任务 34（14.2%）：投行 20（25.0%），法务 10（12.5%），咨询 4（5.0%）。
- 分职业平均条目：投行 2.95，法务 4.65，咨询 4.34。
- Runtime：Harbor 0.20.0，三份共享 `linux/amd64` 镜像。
- 许可证栏：CC-BY 4.0。用途限制：仅评测；禁止训练、微调、参数拟合；禁止爬取。
- 代理：100 turns，`max_steps = 100`，`agent_timeout_sec = 10800`。
- 卡片显示的总文件大小：12.9 GB。API 的 sibling 数是 4441。页面抓取文本在大小附近还有一个未标注的 “1,372”，含义不明，正文没有使用它。

## 博客上的原句级数字

- 发布日叙述的榜首：Claude Fable 5.1，Pass@1 68.6%。
- GPT-6 Astra 的 Pass^4：56.3%。
- Gemini 3.8 Flash 在 93.1% 的运行中搜索沟通记录。
- Claude Fable 5.1 的任务级 Pass@1 提升：24.3%。
- Kimi K3 与 GPT-5.6 Sol 搜索沟通记录的运行大约 15%。Kimi K3 任务级提升 29.7%。
- 相对天真 grader，部分较旧模型 Pass@1 罚分超过 20 个百分点。
- Judge：DeepSeek-v4-Flash-0731，temperature 0.1；再做 GEPA。
- 人工标注 1,407 条，来自 337 条轨迹；留出 354 条。假阴性 5.3% 到 8.0%。
- 依赖扫描：26,302 条轨迹，出现至少 25 次的包被纳入环境。
- Scattergunning 的定义句，以及三轮审计的分工。
- 图注中的任务名 `World221_TR_10`。未核对该题是否属于 1.1 的 240。

不在博客正文、而在 Mercor 或作者 LinkedIn 帖子里的数：Pass@1 前五名里除 68.6% 以外的 67.8%、65.8%、65.3%、64.7%；以及「检出 80.2% 的不表态答案」。第 01 篇引用时已标明是帖子而不是博客正文。

## 2026-09-27 排行榜页面抓取到的数

Opus 5.5 Max 73.5% ±4.9%；Opus 5.5 Medium 52.5% ±5.6%；Fable 5.1 Max 68.6% ±4.9%；Gemini 3.7 Flash High 67.8% ±5.1%；Opus 5 Max 65.8% ±5.1%；Grok 4.6 xHigh 65.3% ±5.2%。

职业小节下的百分比见第 01 篇。抓取文本没有在那些百分比旁重复指标名。

页面同时写着：31 个世界、240 道题；Mean Score 是排序主指标；完整题集保持私有、Hugging Face 上有样例；与 Box 和 Harvey 合作开发。

## 论文里用到的旧版数字

全部应当带「旧版」阅读。这里只列说明正文引用过的，便于回查，不是论文摘要的替代。

- 480 题，33 世界（投行 10、咨询 11、法务 12），每职业 160 题。平均每世界 166 个文件。
- 条目平均 4.06，范围 1 到 10。422 道控制台回复，58 道文件。
- 九个应用、63 个工具；两个投行世界额外的数据应用再加 187 个工具。网页搜索关闭。
- 最高 Pass@1：Gemini 3 Flash 24.0%。GPT-5.2 为 23.0%。两者差异在论文的显著性检验里不显著。
- 每代理每题 8 次，30,720 条轨迹。步数上限 250。
- Judge：Gemini 3 Flash，thinking = low。在 747 条标注上准确率 98.5%（249 条 criterion × 3 个模型输出）。论文写明旧 judge 不看轨迹。
- 附录 D：toolbelt、`final_answer` 工具、70% 摘要、保留最后 10 条消息、旧系统提示词。
- 工时：平均估计 1.82 小时。贡献者 256 人，平均经验 12.9 年。这些没有出现在 1.1 数据集卡上。

## 没有读、因此不能当证据的东西

- 门禁包内的题面、rubric 原文、gold output、`grading_config.json`、世界压缩包、`load_images.sh`、`run_task.sh`、`images.json`、镜像 digest。
- 1.1 verifier 镜像里的 judge 提示词，以及它是否包含 Archipelago 评分源码里的 ungradeable 逻辑。
- Harbor 0.20.0 本身的源码。`-y` 的含义 README 没写，本说明没有用 `--help` 补定义。
- 1.1 榜每一题的运行次数。所以没有把任何 Pass@1 百分比换算成「240 道里的几道」。
- 第三方镜像站对 ECR 仓库名和 digest 的转载。没有拿它们覆盖数据集卡。
