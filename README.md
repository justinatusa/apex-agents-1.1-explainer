# APEX-Agents 1.1 说明书

这是一份给零背景读者的中文说明。它解释 Mercor 的评测 **APEX-Agents 1.1**：让一个 AI 代理在模拟的项目现场里，跨邮件、表格、文档和日历做长任务，再按专家写的评分条目打分。

本仓库只有说明，没有题目、没有模型、也不能代替官方运行。题目在 Hugging Face 数据集 [`mercor/apex-agents-v1.1`](https://huggingface.co/datasets/mercor/apex-agents-v1.1)（接口显示为门禁数据集）。1.1 的参考代理在 [`Mercor-Intelligence/apex_loop_truncated_tools_agent`](https://github.com/Mercor-Intelligence/apex_loop_truncated_tools_agent)。跑任务的基础设施概念来自 [`Mercor-Intelligence/archipelago`](https://github.com/Mercor-Intelligence/archipelago)，但 1.1 公开复现路径是 Harbor，不是 Archipelago 仓库里那个指向旧数据集的示例。

核对日期：2026-09-27。数字的出处、commit 和没有读到的文件在 [`docs/apex-1.1-explainer/99-evidence.md`](docs/apex-1.1-explainer/99-evidence.md)。

## 先读哪一页

按这个顺序就够。某一页里的词不认识，再翻第 05 篇，不必先背术语。

| 顺序 | 文件 | 读完你能回答 |
| --- | --- | --- |
| 0 | 本页 | 这套评测在测什么，240 这个数是什么 |
| 1 | [`00-readme.md`](docs/apex-1.1-explainer/00-readme.md) | 五份材料谁说了算，互相打架时听谁的 |
| 2 | [`01-what-is-apex-1.1.md`](docs/apex-1.1-explainer/01-what-is-apex-1.1.md) | 题量、三个职业、和旧版 480 题差在哪 |
| 3 | [`02-architecture.md`](docs/apex-1.1-explainer/02-architecture.md) | 一次运行里容器、世界、快照、工具网关怎么拼 |
| 4 | [`03-agent-loop.md`](docs/apex-1.1-explainer/03-agent-loop.md) | 代理每一步做什么、何时停、系统提示词原文 |
| 5 | [`04-dependencies.md`](docs/apex-1.1-explainer/04-dependencies.md) | 要装什么、要哪些密钥、要连哪些外部服务 |
| 6 | [`05-unnamed-concepts.md`](docs/apex-1.1-explainer/05-unnamed-concepts.md) | world seed、rubric、scattergunning、GEPA 这些词 |
| 7 | [`06-runbook.md`](docs/apex-1.1-explainer/06-runbook.md) | 官方 README 里的命令，逐条是干什么的 |
| 8 | [`99-evidence.md`](docs/apex-1.1-explainer/99-evidence.md) | 每条硬事实钉在哪个 SHA、哪个文件 |

## 一页纸

APEX 是 AI Productivity Index。APEX-Agents 看的是代理能不能做职业服务里的长任务，不是一道问答题对不对。1.1 把职业收成三个：投资银行、管理咨询、公司法务。出题的人是这些职业的从业者。代理拿到的是一句任务说明，然后自己去项目文件和办公应用里把信息找齐，交出文字，或者交出一个文件。

**一次任务长什么样。** 现场叫 world（世界）：一批项目文件，加上日历、聊天、代码执行、文档、文件系统、邮件、PDF、表格、演示文稿。这些应用通过一个叫 MCP 的工具接口暴露出来。MCP（Model Context Protocol）可以先理解成：模型不直接敲这些软件的内部 API，而是对着一个网关说「我要调哪个工具、参数是什么」。1.1 这一版没有配置网页搜索，官方的理由是让每次运行更可重复。

**240 是什么。** 数据集卡写的是 240 道任务，每个职业 80 道，所以 240 = 80 × 3。旁边还有 31 个世界：投资银行 8 个、管理咨询 11 个、法务 12 个。一个世界里可以有多道任务。240 不是「80 个世界乘以 3」，也不是旧论文那 480 道题的机械一半。旧论文是 33 个世界、每个职业 160 道。1.1 的博客说，他们从每个职业里选出 80 道，并由领域专家审了三轮，还改过世界里的文件。本说明没有做世界名单的逐条对照，不能指出少掉的是哪几个世界。

**怎么算过关。** 每道题有一份 rubric（评分细则），里面是若干条只能判「达到」或「没达到」的标准。数据集卡：一共 955 条，每题 1 到 11 条，平均 3.98 条。一个 judge（判分模型）按条目单独打分，看到的是任务说明、代理输出，以及相关产物或世界里的变化。每道题都有专家做好的 gold output（标准答案）：206 道是文字，34 道是文件。榜上常说的 Pass@1，在旧论文里的定义是：均匀抽一道题、跑一次，全部条目都达到的概率。1.1 的发布博客用 Pass@1 排名，没有把公式重写一遍。

**1.1 为什么要改。** 2026 年 9 月 8 日的博客说，模型会 scattergunning：环境里已经有一个明确答案，模型却同时抛出好几个答案。旧的打分只检查「正确信息有没有出现」，所以多猜几个更占便宜。1.1 用三件事压这个行为：专家把题目收成「有唯一可辩护的答案」、新的 judge 把这种条目判成没达到、系统提示词明文禁止。提示词原文在第 03 篇。

**和榜上分数有关的两句话。** 博客发布当天，Claude Fable 5.1 的 Pass@1 是 68.6%，博客称它当时排第一。2026-09-27 打开的排行榜页面上，Opus 5.5 Max 显示 73.5% ±4.9%，Fable 5.1 Max 仍是 68.6% ±4.9%。页面还写 Mean Score（条目通过比例的平均）是排序主指标。博客排序用的是 Pass@1。两个说法都保留在第 01 篇，不把它们捏成一个数。

**旧论文里的数字不要拿来当 1.1。** arXiv 2601.14242 写的是上一版：480 道题、最多 250 步、当时最高的 Pass@1 是 Gemini 3 Flash 的 24.0%、judge 是 Gemini 3 Flash。那些数字在后文一律标成「旧版」。1.1 配套代理的步数上限是 100，整次运行的超时是 10800 秒（3 小时）。

## 官方入口

- 数据集卡：https://huggingface.co/datasets/mercor/apex-agents-v1.1
- 发布博客：https://www.mercor.com/blog/introducing-apex-agents-1-1/
- 参考代理：https://github.com/Mercor-Intelligence/apex_loop_truncated_tools_agent
- 基础设施：https://github.com/Mercor-Intelligence/archipelago
- 旧版论文：https://arxiv.org/abs/2601.14242
- 排行榜：https://www.mercor.com/apex/apex-agents-leaderboard/
- 联系：apex@mercor.com

数据集卡写明：这套数据只用于模型评测。用来训练、微调或拟合参数是被禁止的。爬取也是被禁止的。本说明不收录题面。
