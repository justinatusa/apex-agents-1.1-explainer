# 00 怎么读这套说明

读者不需要先知道什么是 agent、MCP、容器或榜单指标。每一篇都先讲一件事的经过，再给表。专有名词保持英文，正文是中文。词的白话解释集中在 [05-unnamed-concepts.md](05-unnamed-concepts.md)，第一次出现时也会顺手解释一次。

本说明钉死的版本：

| 材料 | revision / commit |
| --- | --- |
| `mercor/apex-agents-v1.1` | `695dac5bf1388816f97be9eaf882ef01de79d574` |
| `apex_loop_truncated_tools_agent` | `1ec75805084ca5b8aff1d6411e9e3cef03a46947` |
| `archipelago` | `1ca0d41db0ad1fe7c8ee491c253b17c9a44d1d11` |

上一版论文的题量、步数、旧 judge、旧循环，只出现在 [01 的差异对照](01-what-is-apex-1.1.md)。其他篇不重复那些数字。

## 这套说明回答什么

APEX-Agents 1.1 是 Mercor 在 2026 年 9 月发布的一版代理评测。一个模型被放进模拟的银行、咨询或法务项目里，自己调用工具，完成一件需要来回查文件的工作，然后被逐条打分。

说明想让一个没参与过这个项目的人能回答四件事：

1. 它在测什么。1.1 的题量以数据集卡为准。和上一版的差，只看 01 最后一节。
2. 一次运行在机器上分成哪几块。
3. 1.1 指定的那个代理循环怎么走、怎么停、系统提示词写了什么。
4. 若要按官方文档复现，命令从哪一段复制，哪些命令看起来像、其实是旧版。

## 材料的优先顺序

后面每一条硬事实都应该能回到下面某一份材料。冲突时不取平均。

1. **数据集卡**，Hugging Face `mercor/apex-agents-v1.1`，Hub API 返回的 revision 是 `695dac5bf1388816f97be9eaf882ef01de79d574`。题量、世界数、文件数、目录布局、许可证、Harbor 版本、官方的 Hugging Face 运行命令，以这张卡为准。
2. **发布博客**，2026-09-08，[Introducing APEX-Agents 1.1](https://www.mercor.com/blog/introducing-apex-agents-1-1/)。为什么要出 1.1、scattergunning 的定义、三轮审计、judge 怎么改、当天的榜上叙述，以博客为准。
3. **配套 agent 仓库**，`Mercor-Intelligence/apex_loop_truncated_tools_agent`，本说明读到的 commit 是 `1ec75805084ca5b8aff1d6411e9e3cef03a46947`。循环、截断、超时、系统提示词、Harbor Hub 与 Hugging Face 两套命令，以这个仓库的 README 和源码为准。
4. **Archipelago**，`Mercor-Intelligence/archipelago`，本说明读到的 commit 是 `1ca0d41db0ad1fe7c8ee491c253b17c9a44d1d11`。Environment、Agents、Grading、MCP 网关、快照这些基础设施概念以它为准。它的快速开始指向数据集 `mercor/apex-agents`，不是 `mercor/apex-agents-v1.1`。
5. **论文** arXiv 2601.14242。只在 01 的差异对照里使用。不要用论文数字覆盖数据集卡上的 240、31、955。

博客署名是 Austin Bennett、Bertie Vidgen、Arnav Garg、Akul Datta。数据集卡上的 BibTeX 作者是 Bennett、Datta、Vidgen，没有 Garg。两处都照录，不替官方合并名单。

## 已经能看到的不一致

这些不是笔误待修，而是不同文档在讲不同时间点。正文里会再落到具体篇。

上一版和 1.1 在题量、步数、结束条件、工具装载方式、judge 上的差别，集中在 01 的对照表，这里不重列。

下面是 1.1 材料之间仍然对不上的地方：

| 题目 | A 怎么说 | B 怎么说 |
| --- | --- | --- |
| 榜首 | 博客 2026-09-08：Fable 5.1，Pass@1 68.6% | 2026-09-27 的排行榜页面：Opus 5.5 Max 73.5% ±4.9% |
| 主指标 | 博客按 Pass@1 讲榜 | 排行榜页面的问答写 Mean Score 是排序主指标 |
| 网关端口 | 1.1 插件默认 `http://world:8000/mcp/` | Archipelago 本地 Docker 监听 8080 |
| 数据是否公开下载 | 数据集卡：Harbor Hub 与 Hugging Face 都交付这 240 道 | Hub API：`gated: auto`；未登录拉取 `manifest.json` 返回 401。排行榜问答又写完整题集保持私有、Hugging Face 上有样例 |

## 本说明故意不写的东西

- 不收录任何 1.1 题面、评分条目原文、标准答案。数据集是门禁的，卡片也禁止爬取。
- 不跑评测，不训练，不猜镜像 digest。三份镜像在门禁数据里，本说明没有解开它们。
- 不把 Archipelago 在 1.1 发布之后才出现的新旋钮，说成 1.1 榜的默认配置。配套仓库里真正写进 `agent_config.json` 的字段，以插件源码为准。
- 命令只复制官方 README 或数据集卡里已经印出来的那几行。Harbor 某个旗标如果 README 没解释，这里就写「README 没有定义」。

## 文件一览

- [01-what-is-apex-1.1.md](01-what-is-apex-1.1.md)
- [02-architecture.md](02-architecture.md)
- [03-agent-loop.md](03-agent-loop.md)
- [04-dependencies.md](04-dependencies.md)
- [05-unnamed-concepts.md](05-unnamed-concepts.md)
- [06-runbook.md](06-runbook.md)
- [99-evidence.md](99-evidence.md) 放在最后，核对出处时可以随时翻。
