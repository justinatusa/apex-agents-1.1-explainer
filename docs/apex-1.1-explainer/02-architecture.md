# 02 一次运行的结构

先看 1.1 官方要你怎么跑，再看 Archipelago 这三个词是什么。两套文档说的是同一类系统，入口不一样。

1.1 的数据集卡把运行时写成 Harbor 0.20.0，加上三份共享的 `linux/amd64` 镜像：world、agent、verifier。Harbor 在这里是任务运行器：它按一个任务包拉起容器、把代理接进去、再把判分接上。配套代理的源码把一次试验写成两个容器加一次判分，代理不在你的笔记本进程里直接调工具。

## 从按下命令到打分

你在自己的机器上安装 Docker 和 Harbor，准备好模型的 API 密钥。Harbor 启动一次试验（配套代码里叫 trial）。试验内部大致是这样：

```text
你的机器
  harbor run
    │
    ├─ main 容器（agent 镜像）
    │    插件 ApexLoopTruncatedToolsAgent
    │    把 vendored runner 解压到 /agent_runner
    │    uv 装依赖，然后跑 runner_cli
    │    模型调用走出容器，打到 Anthropic / OpenAI / Google 等 API
    │
    ├─ world 容器（world 镜像，compose 里的 sidecar）
    │    启动时灌入这个世界的种子
    │    可以再叠这道题自己的文件
    │    里面跑着 MCP 网关和各个办公应用
    │    插件默认去找 http://world:8000/mcp/
    │
    └─ verifier（verifier 镜像）
         数据集卡：用任务说明、代理输出、相关产物或世界变化
         按 rubric 逐条打分
         判分模型的提示词在门禁任务包里，本说明没有读
```

配套插件文件 `apex_loop_truncated_tools_agent.py` 顶部的说明写得很具体，下面这些是源码里的话，不是推断：

- 同一个代理类服务这批交付物里的所有任务。世界的 slug 和密钥通过任务的 compose 文件进 world 容器。
- 代理跑在 main 容器里，经试验网络访问 world 网关，默认 URL 是 `http://world:8000/mcp/`。
- 执行发生在沙箱里的 `env.exec`，不把端口映射到宿主机。
- `setup()` 只做一件事：把这份交付物自带的 runner 放进 main 容器。世界容器在 compose 里作为旁边的容器自己启动。插件注释写 “POST-healthcheck-gated”：世界容器先通过健康检查，试验才继续把代理接上去。

数据集卡对磁盘上的交付物是这样分的：

```text
environment/    共享的 world、agent、verifier 镜像归档
tasks/          240 个 Harbor 任务目录
worlds/         31 个只读世界种子
manifest.json   交付物和镜像清单
registry.json   任务 slug 到任务目录的映射
run_task.sh     卡上称为已经验证过的 Harbor 启动脚本
```

每个任务目录里有说明、元数据、代理配置、环境覆盖层、solution 和 grader。每个世界里有 `filesystem.tar.gz`、`apps_data.tar.gz`、MCP 配置、snapshot hooks 和世界元数据。这些文件名来自数据集卡。文件本体在门禁后面，本说明没有打开，所以不能在这里画出某一个世界的目录树，也不能引用某一道题的 grader JSON。

卡还区分两个分发通道，内容是同一批 240 道题：

| 通道 | 卡上的说法 |
| --- | --- |
| Harbor Hub | 240 个任务包。world、agent、verifier 镜像按 digest 钉死，第一次运行从公共 ECR 拉取 |
| Hugging Face | 同样的 240 个可构建 Harbor 任务，三份镜像归档和 31 个世界种子在磁盘上 |

digest 是镜像内容的哈希。卡说它们被钉死，但没有在卡片正文里印出哈希值。未登录读取 `environment/images.json` 得到 401。本说明不编造哈希。

## 世界、种子、快照

这三个词经常叠在一起。按时间顺序拆开就清楚了。

**世界**是专家搭的一个项目现场：哪家虚构或半虚构的客户、已经有哪些文件、邮件和日历事件、代理可以使用哪些应用。1.1 有 31 个这样的现场。

**世界种子**是这个现场的开局存档。卡把它叫做 read-only world seed。磁盘上每个世界至少有两包：`filesystem.tar.gz` 和 `apps_data.tar.gz`。只读说的是这份发布物不要被你改掉当作新的标准世界。运行中的容器不是只读的。代理会改表格、写文档、发邮件。那些改动留在容器里，最后被打成快照。

**快照**是某一时刻环境状态的打包。Archipelago 的 `POST /data/snapshot` 会先跑 pre-snapshot hooks，再把 `filesystem` 和 `.apps_data` 两个子系统打成 `tar.gz` 流式返回。钩子是打包前执行的命令。源码注释里的例子是导出数据库，让快照里不只有散落文件，还有应用自己的状态。数据集卡说每个 1.1 世界带有 snapshot hooks。钩子脚本在门禁的世界种子里，本说明没有打开，不能列出 1.1 实际跑了哪些命令。

评分要看运行前和运行后各一份。环境 README 提醒：环境吐出 `.tar.gz`，Archipelago 本地评分要的是 `.zip`。那是基础设施仓库自己的格式差。1.1 的 Harbor verifier 是否在镜像里做了转换，任务包未打开，不确定。

**快照不是种子。** 种子是开局存档。快照是某一时刻的打包，包括代理改完之后的那一份。只读的是发布出来的种子文件，不是运行中的容器。

一道题还可以在种子之上放一层任务自己的文件。卡的用词是 task-specific file overlays。种子是全世界大厅里的公共档案，覆盖层是这道题桌上多放的那几份。

## MCP 网关和九个应用

MCP 是一种让模型调用外部工具的协议。网关是环境里的一个 HTTP 端点，后面挂着多个 MCP 服务器。代理只连网关，不分别去记九个服务的地址。

1.1 数据集卡说，每个世界都暴露同一组核心应用：

| 卡上的名字 | Archipelago 里的目录 | 这个应用在现场里扮演的角色 |
| --- | --- | --- |
| Calendar | `mcp_servers/calendar` | 日程 |
| Chat | `mcp_servers/chat` | 内部聊天 |
| Code execution | `mcp_servers/code` | 跑代码，算数必须走这里 |
| Documents | `mcp_servers/documents` | 文本文档 |
| File system | `mcp_servers/filesystem` | 列目录、读文本和图片 |
| Mail | `mcp_servers/mail` | 邮件。博客里补参数的例子就放在邮件里 |
| PDFs | `mcp_servers/pdfs` | 读或生成 PDF |
| Spreadsheets | `mcp_servers/spreadsheets` | 表格 |
| Presentations | `mcp_servers/presentations` | 幻灯片 |

这九个名字以数据集卡为准。右边的目录来自公开的 Archipelago 仓库（commit `1ca0d41`），用来告诉你实现放在哪，不是 1.1 镜像的文件清单。

Archipelago 里还有 `edgar_sec` 和 `fmp`。1.1 的数据集卡没有把这两个应用列进「每个世界都暴露」的那张表。不能根据仓库里有这两个目录，就说 1.1 的 31 个世界都带 SEC 或行情工具。上一版论文里多出来的工具数写在 [01 的对照节](01-what-is-apex-1.1.md)。

公开源码里，文件系统服务器注册的工具名包括 `list_files`、`read_text_file`、`read_image_file`、`search_files`、`get_file_metadata`、`get_directory_tree`。代码执行服务器注册的是 `code_exec`。邮件、表格、文档的名字更多，放在 [04-dependencies.md](04-dependencies.md)。1.1 某一次运行实际能看见的工具列表，是世界容器启动后网关 `list_tools` 的结果。Archipelago 网关代码里还有按允许名单过滤工具的中间件。任务包可以收窄工具，本说明没有读到 1.1 的 MCP 配置，所以不宣称 240 道题的工具集合完全相同，只复述卡上的话：每个世界暴露同一组核心应用。

系统提示词对路径有一条硬规则，循环篇会引用原文：除了 `code_exec`，文件都在 `/` 下面找；`code_exec` 必须用 `/filesystem/` 作为根。Archipelago 环境的默认子系统名也是 `filesystem` 和 `.apps_data`，和这条规则对得上。

## 端口：8000 和 8080 不是笔误

1.1 插件把网关默认写成 `http://world:8000/mcp/`。这是容器网络里的主机名 `world`，不是你笔记本上的 localhost。

Archipelago 环境的 Dockerfile 和 docker-compose 监听的是 **8080**，健康检查是 `http://localhost:8080/health`，配置 MCP 的接口是 `POST /apps`，网关挂在 `/mcp/`。那是基础设施仓库本地开发的入口。

两套数字都是真的，它们描述的入口不同。复现 1.1 榜时不要把 Archipelago README 里的 `localhost:8080` 填进 Harbor 命令。Harbor 任务的 compose 在门禁包里，本说明没有核对 world 容器进程究竟 bind 了 8000 还是由 compose 把 8000 转到容器内的 8080。能确定的是：1.1 代理默认去连 `world:8000`。

## Environment、Agents、Grading

Archipelago README 把基础设施分成三块，三块都在 Docker 里。1.1 数据集卡的三份镜像和这三块对应。1.1 的编排者是 Harbor 0.20.0。

| 1.1 的镜像 | Archipelago 的组件 | 在一次试验里做什么 |
| --- | --- | --- |
| world | Environment | 无界面沙箱。灌入两份 tar.gz，挂上 MCP 网关，结束时打快照 |
| agent（main 容器） | Agents | 跑 `loop_truncated_tools_agent`。只连 `/mcp/`，不自己实现邮件或表格 |
| verifier | Grading（以前叫 Verifier） | 按 rubric 逐条打分。材料见数据集卡：任务说明、代理输出、产物或世界变化 |

**Environment。** 世界容器里的管理进程。Archipelago 本地开发时，`GET /health` 做健康检查，`POST /apps` 热更新工具配置，网关在 `/mcp/`，`POST /data/populate` 上传 tar.gz（查询参数 `subsystem` 取 `filesystem` 或 `.apps_data`），`POST /data/snapshot` 打包。可以接 S3。1.1 的官方命令没有要求你填 S3 密钥。1.1 插件不走笔记本上的这些 URL，它在试验网络里访问 `http://world:8000/mcp/`。

**Agents。** 一份注册表，同一种 runner 可以登记多种循环。1.1 的 `manifest.json` 把 `agent_config_id` 写成 `loop_truncated_tools_agent`。同一仓库里还有 `loop_agent`、`react_toolbelt_agent`、`singleshot_agent`、`stirrup_agent` 等。官方命令加载的类是 `ApexLoopTruncatedToolsAgent`，不会走到其他注册项。步数和停止条件在下一篇。

**Grading。** 评分 README 把流水线分成三段。Helper 先算一次共用数据，例如前后快照的文件 diff。Verifier 是一条一条的检查；1.1 由 judge 按 criterion 做二元判断。Scoring method 把多条结果收成一个分数。1.1 卡没有公布 scoring method 的 id。能确定的是条目独立打分，博客的 Pass@1 要全部达到。Judge 和 GEPA 见 [01](01-what-is-apex-1.1.md)。

Archipelago 的执行顺序是：拉起环境并等健康检查，灌入世界和任务数据，配置 MCP，跑代理，做快照。1.1 的 Harbor 插件把「拉起世界」交给 compose，自己只往 main 容器里装 runner。

评分源码里还有 `ungradeable`：快照读不出来、判分模型不可用、judge 拒绝下结论时，不应当记成模型得了 0 分。这是 commit `1ca0d41` 的规则。1.1 verifier 镜像是否包含这段逻辑，镜像未打开，不能确定。见 [05](05-unnamed-concepts.md)。

## 代理镜像里实际跑起来的进程

`setup()` 把 `runner_src` 打成 tar，上传到 main 容器的 `/agent_runner`，解压。日志目录是 `/logs/agent`。

`run()` 做这些事：

1. 要求调用方传入模型名，形如 `anthropic/claude-opus-5`。没有默认模型。
2. 在容器环境变量里预检密钥。缺密钥就立刻失败。注释说，否则重试会把整个代理超时耗光。
3. 写下两条消息：一条 system，一条 user。user 的内容是 Harbor 传进来的任务说明。
4. 写下 `agent_config.json`。1.1 插件写进去的字段只有六个：`timeout`、`max_steps`、`max_output_chars`、`max_output_lines`、`tool_call_timeout`、`llm_response_timeout`。
5. 在容器里 `uv sync --frozen`，再执行 `python -m runner_cli`，网关 URL 用上面的默认值。
6. Harbor 这一层的 `environment.exec` 超时是代理超时再加 900 秒。默认即 10800 + 900。
7. 把 runner 写出的 native 轨迹转成 ATIF（Harbor 的轨迹格式），路径是 `/logs/agent/trajectory.json`。若 native 文件缺失或无法解析，就写一份没有步骤的 ATIF。若进程退出码不是 0，插件抛错。源码注释的意图是：runner 崩溃应当让试验报错，而不是被当成一份空作业去打分。

`manifest.json` 里的版本是 1.0.1，和 Harbor Hub 数据集引脚 `mercor/apex-agents-1-1@1.0.1` 对齐。`studio_agent_id` 是 `agent_51d240d4a7b04f059f6472925fb83e89`。`source_revision` 是 `2bb7e26b3599d9f55e7994198df45f374fe9b600`，这是 vendored runner 声明的上游修订，不是 GitHub 仓库的 commit。仓库 commit 是 `1ec75805084ca5b8aff1d6411e9e3cef03a46947`，提交说明是采用 Studio 记录在案的那条 1.1 系统提示词。

## 不要走错的那条「看起来也能跑」的路

Archipelago README 的快速开始推荐 `examples/hugging_face_task/run.sh`，并写明它跑的是 `mercor/apex-agents`。那不是 `mercor/apex-agents-v1.1`。题量差见 [01 的对照节](01-what-is-apex-1.1.md)。官方 1.1 命令在 [06-runbook.md](06-runbook.md)。
