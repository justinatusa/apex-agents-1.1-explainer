# 04 依赖、网络和外部服务

跑 1.1 的官方路径时，你的机器要有 Docker、`uv`，以及 Harbor 0.20.0。模型密钥和判分密钥放在配套仓库的 `.env` 里，由 `harbor run --env-file .env` 传进去。数据集卡写：这些密钥在运行时通过环境变量提供，不包含在数据集里。`GRADING_MODEL` 留空时，用任务自己配置的 grader。

下面先列 1.1 复现真正点名的东西，再列 Archipelago 源码里存在、但 1.1 卡片没有要求你配置的东西。混在一张表里会让人以为必须申请 S3 和 Datadog 才能跑榜。

## 1.1 路径点名的依赖

| 依赖 | 出处 | 它是什么 |
| --- | --- | --- |
| Docker | 配套 README、数据集卡 | 跑三份镜像的容器引擎 |
| `uv` | 同上 | Python 工具安装器。命令是 `uv tool install harbor==0.20.0`。runner 也用 `uv sync` |
| Harbor `0.20.0` | 数据集卡的 Runtime 行；README 把版本钉在安装命令里 | 任务运行器。其他版本是否兼容，两份文档都没说 |
| `hf` CLI | 配套 README 的 Hugging Face 一节 | 只在你走 Hugging Face 下载时需要。走 Harbor Hub 的命令不调用它 |
| Python `>=3.13,<3.14` | runner 的 `pyproject.toml` | 这是容器里的 runner。宿主机用 `uv tool install` 装 Harbor 即可，README 没有要求宿主机是 3.13 |
| LiteLLM `1.97.0` | 同一份 `pyproject.toml` | 用 `provider/model` 这种字符串调不同公司的模型 |
| fastmcp `>=3.2.0` | 同上 | 连 MCP 网关的客户端 |
| 三份 `linux/amd64` 镜像 | 数据集卡 | world、agent、verifier。Hub 通道从公共 ECR 按 digest 拉取；Hugging Face 通道用 `environment/load_images.sh` 从本地归档装入 Docker |

模型名字的格式，README 的原话是：换成任何 LiteLLM 兼容的 `provider/model`。文档里的例子是 `anthropic/claude-opus-5`。例子不是榜首，也不是唯一允许的模型。

## 1.1 `.env.example` 里的变量

文件是配套仓库根目录的 `.env.example`。README 只说把模型供应商的密钥和判分凭据填进去。下表是这个示例文件里出现的名字。空着的含义是「你不用这个供应商就留空」，不是「必须全部填满」。

| 变量 | 分组 | 源码里能看到的用途 |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | 模型 | 模型前缀是 `anthropic/` 时，预检要求容器里有它 |
| `ANTHROPIC_BASE_URL` | 模型 | 示例文件有此行。预检只检查密钥是否存在，不解释这个 URL |
| `OPENAI_API_KEY` | 模型 | 前缀 `openai/` 时需要 |
| `OPENAI_BASE_URL` | 模型 | 示例文件有此行 |
| `GOOGLE_API_KEY` | 模型 | 前缀 `gemini/` 或 `google/` 时，它和 `GEMINI_API_KEY` 有一个即可 |
| `GEMINI_API_KEY` | 模型 | 同上 |
| `FIREWORKS_AI_API_KEY` | 模型 | 前缀 `fireworks_ai/` 时，它和下一行有一个即可 |
| `FIREWORKS_API_KEY` | 模型 | 同上 |
| `GRADING_MODEL` | 判分 | 数据集卡：留空则用任务配置的 grader |
| `GOOGLE_APPLICATION_CREDENTIALS` | 判分 | 示例文件把它放在 Grader 下面。卡没有解释何时必填 |
| `VERTEXAI_CREDENTIALS` | 判分 | 同上 |
| `VERTEXAI_LOCATION` | 判分 | 同上 |
| `VERTEXAI_PROJECT` | 判分 | 同上 |

插件预检还有两条不在这份 `.env.example` 里、但源码会认的路：

- 前缀 `litellm_proxy/` 需要 `LITELLM_PROXY_API_KEY`。
- 只要 `LITELLM_PROXY_API_BASE` 和 `LITELLM_PROXY_API_KEY` 同时在容器环境里，预检就不再要求上面的供应商密钥。`runner/utils/llm.py` 会把补全请求转到这个代理。

未知的供应商前缀不会被预检挡住。缺密钥时会在真正调用模型时失败。

判分密钥最终被 verifier 容器还是 main 容器读取，取决于 Harbor 任务包怎么传 `--env-file`。任务包是门禁的，本说明只确定到「官方要求你写进 `.env` 并传给 `harbor run`」。

## Judge 和 GEPA

博客：基线 judge 是 DeepSeek-v4-Flash-0731，temperature = 0.1。作者说它的基线准确率最好。然后用 Genetic-Pareto（GEPA）优化系统提示词。优化的对象是判分提示词的文字，不是去训练一个新的判分模型权重。

标注规模写在博客里：Mercor 研究人员手工标了 1,407 条 rubric 条目，来自 337 条轨迹。留出 354 条做对照。相对天真 grader，假阴性率从 5.3% 升到 8.0%。博客认为 scattergunning 的检出上升了，而且准确率好于三名独立评分人的平均，所以接受了更高的假阴性。博客没有附混淆矩阵。这里的假阴性按常见定义理解：人认为这条已经达到，judge 却判成没达到。如果他们把「检出 scattergunning」当作正类，这个比率会是另一种意思；正文把假阴性和「检出率上升」分开说，所以更像判分误差，而不是检出率本身。

博客的英文是：“a judge that could accurately score scattergunned rubric items, but not full answers, as a zero.” 按这句话的结构，被打成 0 的是发生了 scattergunning 的那一条评分条目，不是整份回答。其他条目仍然单独打分。博客没有附上判分提示词，所以「整份回答」在实现里具体指哪一段输出，以门禁包里的 grader 为准。

Austin Bennett 在发布日的 LinkedIn 帖子里写，这个 GEPA judge 能检出 80.2% 的不表态答案。博客正文没有印 80.2% 这个数。图 3 的说明只比较天真 judge、能识别 scattergun 的 judge，以及人类评分人的平均准确率，抓取到的正文没有给出图中的百分比。80.2% 只作为作者帖子里的数引用。

完整的 LiteLLM 模型字符串（例如是否带 `fireworks_ai/accounts/...` 前缀）写在每道题的 grading 配置里。那份 JSON 在门禁任务包中，本说明没有打开。第三方镜像仓库的 README 声称见过这个字符串。那不是 Mercor 的文档，这里不转载成官方钉死值。

## 九个应用在公开源码里注册的工具名

下表来自 Archipelago commit `1ca0d41` 各服务器 `main.py` 里的 `mcp.tool(名字)`。带 `_schema` 的名字也照录，因为它们出现在同一种注册调用里。本说明没有把服务器跑起来，不能证明每一个名字都会在 1.1 网关的 `list_tools` 里出现。1.1 卡保证的是应用种类，不是这张名字表的每一个拼写。

| 应用 | 注册名 |
| --- | --- |
| Calendar | `list_events`，`read_event`，`create_event`，`update_event`，`delete_event`，`calendar`，`calendar_schema` |
| Chat | `list_channels`，`get_channel_history`，`get_thread_replies`，`get_user_profile`，`get_users`，`post_message`，`reply_to_thread`，`add_reaction`，`delete_post`，`chat`，`chat_schema` |
| Code execution | `code_exec` |
| Documents | `create_document`，`delete_document`，`get_document_overview`，`read_document_content`，`read_image`，`add_content_text`，`edit_content_text`，`delete_content_text`，`add_image`，`modify_image`，`apply_formatting`，`header_footer`，`page_margins`，`page_orientation`，`comments`，`docs`，`docs_schema` |
| File system | `list_files`，`read_image_file`，`read_text_file`，`search_files`，`get_file_metadata`，`get_directory_tree` |
| Mail | `list_mails`，`read_mail`，`search_mail`，`send_mail`，`reply_mail`，`reply_all_mail`，`forward_mail`，`mail`，`mail_schema` |
| PDFs | `create_pdf`，`read_pdf_pages`，`read_image`，`read_page_as_image`，`search_pdf`，`pdf`，`pdf_schema` |
| Presentations | `create_deck`，`delete_deck`，`add_slide`，`edit_slides`，`add_image`，`modify_image`，`insert_chart`，`insert_table`，`add_shape`，`read_slides`，`read_completedeck`，`read_individualslide`，`read_image`，`slides`，`slides_schema` |
| Spreadsheets | `create_spreadsheet`，`delete_spreadsheet`，`read_tab`，`read_csv`，`list_tabs_in_spreadsheet`，`add_tab`，`delete_tab`，`edit_spreadsheet`，`add_content_text`，`delete_content_cell`，`create_chart`，`filter_tab`，`sheets`，`sheets_schema` |

`edgar_sec` 和 `fmp` 在同一仓库里各有一批工具（公司申报、行情、财务报表等）。1.1 卡片的核心应用列表没有它们。上一版有没有在部分世界里打开过，见 [01 的对照节](01-what-is-apex-1.1.md)。1.1 的截断循环也不会提供「把工具加入 toolbelt」这种元工具。

## 代码执行里的库

博客的规则：扫描 26,302 条轨迹，依赖若总共出现至少 25 次，就放进环境。这是 1.1 环境的收录标准。

公开的 `mcp_servers/code/pyproject.toml` 在主依赖里写了 pandas、numpy、scipy、matplotlib、seaborn、statsmodels、scikit-learn、xgboost、duckdb、openpyxl、python-pptx、python-docx、reportlab、pypdf、pdfplumber、beautifulsoup4 等，并另有 `finance`、`data-science`、`scicomp`、`medicine` 等可选组（里面有 yfinance、sympy、biopython 等）。这份文件是 Archipelago 源码的声明。它使用的 LiteLLM 版本是 `1.83.10`，和 1.1 runner 钉的 `1.97.0` 不是同一个组件，版本不同不奇怪。

不能把这份 `pyproject.toml` 当成 1.1 world 镜像的 `pip freeze`。镜像归档在门禁数据里。能确定的是提示词禁止代理用这些库去访问互联网，即使某个库（例如 `requests` 或 BeautifulSoup）出现在源码依赖里。

## 网络

1.1 要出站的地方，官方文档写清楚的是模型 API：代理在 main 容器里通过 LiteLLM 调用你配置的供应商。判分同样要调用 judge 模型。数据集没有附带密钥。

世界里面，数据集卡写 web search 没有配置。提示词禁止用代码去查网上的答案。本说明没有在已读文档里找到「容器网卡丢弃全部出站流量」这样一条防火墙规则。所以「不能上网」在已读材料里是工具配置和提示词，不是一条已经核对过的网络策略。

Archipelago 环境还可以从 S3 灌数据和上传快照。那是基础设施的能力。1.1 的 Hugging Face 交付物把种子放在磁盘上的 `worlds/`，Harbor Hub 则拉取钉死的镜像。两条官方 1.1 命令都没有要求你填写 `S3_ACCESS_KEY_ID`。

## Archipelago 里有、1.1 复现命令没要求的服务

这些出现在 Archipelago 的 README 或 `.env.example`。跑官方 1.1 命令时不要先去申请它们。

| 名字 | 出现在哪 | 和 1.1 的关系 |
| --- | --- | --- |
| S3（`S3_SNAPSHOTS_BUCKET` 等） | 环境 `.env.example`，默认可不填 | 本地 Archipelago 把快照放进对象存储时用。1.1 Harbor 命令未提及 |
| Reducto（`REDUCTO_API_KEY`） | 评分 `.env.example`；旧论文脚注 | 旧 judge 用它抽取文档。评分示例把 Mercor 文档缓存放在前面，Reducto 作本地回退。1.1 卡没有要求调用者自备 Reducto 密钥 |
| `MERCOR_DOCUMENT_API` | 评分 `.env.example` | Studio 内部文档缓存。不是 1.1 README 的步骤 |
| Firecrawl | 评分 `.env.example` | 抓网页。1.1 关闭了 web search |
| Redis、Datadog、PostgreSQL | Agents README 的日志后端 | 可观测性。1.1 `.env.example` 没有 |
| `RL_STUDIO_API`、LLM Gateway、`SAVE_WEBHOOK_URL` | runner 的 `settings.py` | Studio 生产环境的上报和排队。本地 Harbor 路径不依赖它们。`ENV` 默认是 `local`，gateway 路由在 local 下直接关闭 |
| 公共 ECR | 数据集卡 | 只在 Harbor Hub 首次拉镜像时由 Harbor 访问。卡没有印账号或仓库 URL |

镜像从哪一个 ECR 仓库名拉取，卡片只说 “public ECR”。具体仓库字符串在门禁的 `images.json` 或任务包里。未登录读不到。不要把第三方转载的仓库名写成 Mercor 卡片上的原文。
