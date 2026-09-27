# 05 那些不好翻译的词

这一篇按「先遇到先查」来写。每条都是白话定义，后面跟着出处。出处不足以确定时，定义里会写不确定。

## world（世界）

一套已经发生到一半的项目。里面有客户背景、文件、邮件、聊天和日程，以及代理可以调用的应用。1.1 有 31 个世界。一道任务是世界里的一个案例，所以任务数（240）大于世界数。

出处：数据集卡 “Each case is a task inside a world”。

## world seed（世界种子）

世界开局状态的发布件。卡说 31 个种子是只读的，每个世界包含 `filesystem.tar.gz`、`apps_data.tar.gz`、MCP 配置、snapshot hooks 和世界元数据，31 个种子合计 5,182 个文件。

只读指的是这份存档。容器跑起来以后，代理可以改里面的文件。改过的状态靠快照留下，不靠把种子仓库改掉。

出处：数据集卡 “31 read-only world seeds” 和 “World context” 一段。

## task overlay（任务覆盖层）

卡的用词是 task-specific file overlays：在世界种子之上，某道题再加自己的文件。63 道题（26.3%）带输入文件。这列数字和 5,182 不是同一批文件。

出处：数据集卡总表 “Tasks with input files”，以及 “read-only world seeds plus any task-specific file overlays”。

## snapshot（快照）

某一时刻环境的打包。评分用运行前和运行后的两份，看代理创建或改动了什么。Archipelago 环境接口吐出 `tar.gz`。同一仓库的环境 README 提醒：评分端要的是 zip，两边格式不同，本地手工跑评分时要转换。1.1 的 Harbor verifier 是否在镜像里做了转换，没有读到镜像，不确定。

出处：Archipelago 根 README 的 Environment 与 Grading；环境 README 的 Snapshot 一节。数据集卡只点名每个世界带有 snapshot hooks，没有描述钩子的脚本内容。

## MCP 与 gateway（网关）

MCP（Model Context Protocol）是模型调用工具的协议。网关是一个 HTTP 地址，后面代理着多个应用服务器。代理只连这一个地址。1.1 插件的默认地址是 `http://world:8000/mcp/`。Archipelago 本地开发文档写的是宿主机 `localhost:8080`。两个端口对应两条入口，见架构篇。

出处：配套代理 `apex_loop_truncated_tools_agent.py`；Archipelago 环境 Dockerfile。

## toolbelt（工具腰带）

上一版论文给当时代理起的结构名：先只拿元工具，用到具体应用时再把工具加进腰带，结束要调用 `final_answer`。1.1 的参考代理不是这套。公开 Archipelago 里 `react_toolbelt_agent` 仍在注册表中，官方 1.1 命令选中的是 `loop_truncated_tools_agent`。上一版的步数和摘要规则只在 [01 的对照节](01-what-is-apex-1.1.md)。

出处：01 对照节所引的论文附录；配套仓库 `manifest.json` 的 `agent_config_id`。

## truncated loop（截断循环）

1.1 这份参考实现的做法：启动时装上网关列出的工具；工具输出超过 200 行或 32768 字符就截断；默认 100 步、10800 秒。模型哪一步不再调用工具，哪一步就当作做完。`final_answer` 只是日志类型。

出处：配套 README；`loop_truncated_tools_agent/main.py`；数据集卡 “Agent implementation” 一节。

## rubric 与 criterion

Rubric 是一道题的评分细则。Criterion 是其中一条，只能判达到或没达到。1.1 共 955 条，每题 1 到 11 条，平均 3.98。Judge 按条单独打分。955 是条目总数，不是通过的题数。

出处：数据集卡 “Evaluation”。打分顺序见 01。

## gold output（标准答案）

专家按题目要求的格式做出的答案。1.1 每题都有：206 个文字，34 个文件。Judge 打分时，文字题会有文字参照，文件题会有专家文件。卡是这样写的。它没有写 judge 会把标准答案原文整段贴进提示词，也没有写不会。那是 grader 配置里的事，配置未读。

出处：数据集卡 “Gold outputs” 与 “Evaluation”。

## grading target

上一版论文用这个字段标明一条 criterion 要看控制台文字还是某份文件。1.1 的数据集卡没有使用这个英文名。卡只说 judge 使用任务说明、代理输出、相关产物或世界变化。Archipelago 评分示例里有 `criteria` 和 `artifacts_to_reference`，那是基础设施配置的形状，不是从 1.1 任务包抄来的。1.1 是否仍用 grading target 这个字段名，不确定。

出处：数据集卡 Evaluation；01 对照节里的上一版描述。

## scattergunning

博客给的定义：环境里已经有一个被交代清楚的答案，模型却给出多个答案。中文可以把它想成散弹枪式作答。博客把 hedging（不表态、留退路）和它放在同一组行为里讲。

为什么旧打分会被钻空子：条目只检查正确信息是否出现。多个互相矛盾的数里只要有一个是对的，条目就可能通过。博客举的职业对照是：银行分析师不会在一份 LBO 模型里同时交上「用中位数」和「用平均数」两套数，他们会选定一个。

提示词里的定义更窄，而且写进了给模型的原文：提供任务说明没有明确要求的备选答案。说明真的要求分情形作答，提示词说那不是 scattergunning。

博客图 1 的例子题名叫 `World221_TR_10`。博客没有说这道题是否留在 1.1 的 240 道里。这里只把它当作博客图注里的名字。

出处：博客 “Scattergunning” 一节；配套系统提示词（哈希见证据篇）。

## naive grader（天真判分）

博客用来对照的旧判法：不专门把 scattergun 判成失败。1.1 的新 judge 相对它，假阴性从 5.3% 升到 8.0%，检出更好。一些旧模型在新 judge 下 Pass@1 掉了 20 个百分点以上。

出处：博客 “Judge model changes” 和 “Performance”。

## GEPA

博客写作 Genetic-Pareto，用来改 judge 的系统提示词。它不是把 DeepSeek-v4-Flash-0731 的权重再训练一遍。数据集卡禁止用这套题做训练或参数拟合。GEPA 发生在 Mercor 准备 judge 的过程里，不是复现者跑榜时的一个步骤。官方复现命令没有 GEPA。

优化用的标注是 1,407 条人工条目（337 条轨迹），留出 354 条。博客说优化后的 judge 把 scattergun 的条目打成 0，不是把整份回答打成 0。

出处：博客 “Judge model changes”。GEPA 这个缩写在公开文献里也指一种提示词优化算法。博客没有给出他们使用的代码仓库或超参数。除了 temperature 0.1 和模型名，本说明不补充未写出的优化细节。

## Pass@1、Pass@k、Pass^k、mean score

1.1 博客和排行榜页面用这些词，没有把公式重写一遍：

- Pass@1：一次运行里，条目全部达到才算这道题通过。博客用它排名。
- Pass@4：博客用在「至少做对过一次」。
- Pass^4：四次都做对。博客给出的最高值是 GPT-6 Astra 的 56.3%，那不是 Pass@1。
- Mean score：条目通过百分比的平均。做对一半条目时，它可以高于 0，而这道题的 Pass@1 仍是没通过。排行榜问答把它写成排序主指标。

不要把 68.6% 乘以 240 说成「过了多少道」。博客没写每题跑了几次。上一版论文里的 Pass@8 算法只在 [01 的对照节](01-what-is-apex-1.1.md)。

出处：博客 “Performance”；排行榜页面问答；01 的打分一节。

## 三轮审计

博客：每个职业 80 道，经 Mercor 领域专家三轮审计。前两轮确认有唯一且写清楚的答案。第三轮检查放进世界文件的信息公不公平。澄清参数写在邮件、表格和文档里，而不是全部写进提示词。

出处：博客 “Task updates”。

## doom loop

上一版论文用这个词描述代理在工具调用里打转、直到步数或时间耗尽。1.1 的博客和配套 README 没有沿用这个词。1.1 的步数帽是 100，到顶记为轨迹状态 `failed`。上一版的超时比例见 [01 的对照节](01-what-is-apex-1.1.md)，不要套到 1.1。

出处：01 对照节；1.1 循环源码里 `max_steps` 用尽时的 `FAILED` 状态。

## ATIF

配套插件把 runner 的原始轨迹转成 Harbor 用的轨迹格式，`schema_version` 写成 `ATIF-v1.5`，文件是 `trajectory.json`。原始文件是 `trajectory.native.json`。转换失败时写一份空步骤的 ATIF，避免判分端因为缺文件而报另一种错。插件仍会在 runner 非 0 退出时抛异常。

出处：`apex_loop_truncated_tools_agent.py` 的 `convert_trajectory`。

## ungradeable（无法判分）

Archipelago 评分源码里的约定：快照损坏、磁盘满了、judge 没有给出结论、judge 拒绝回答，这些情况不应当写成「模型得 0 分」。0 分要留给「代理交了作业但作业是错的」或者「代理什么都没交」。无法判分应当让这次评分作废。

这是基础设施 commit `1ca0d41` 的评分 README 和 `ungradeable.py` 的设计。1.1 verifier 镜像有没有这段代码，不确定。读榜时仍然值得把「基础设施失败」和「模型答错」分开，因为博客讨论的假阴性是判分误差，不是容器崩溃。

出处：`archipelago/grading/README.md`，“Reporting a criterion you could not judge”。

## Harbor trial 与 main / world 容器

Harbor 的一次试验里，代理进程在 main 容器，办公应用在 world 容器。插件用试验网络里的主机名 `world` 找网关，不映射到宿主机端口。Verifier 是第三份镜像，数据集卡把它和 world、agent 并列。

出处：数据集卡 Runtime 与 Distribution；插件文件顶部注释。

## 公共 ECR 与 digest

数据集卡：Harbor Hub 上的三份镜像按 digest 钉死，首次运行从公共 ECR 拉取。Digest 是内容哈希，用来保证拉到的镜像和发布时一致。哈希值本身在门禁的清单文件里。本说明不编造。

出处：数据集卡 Distribution 表。
