# 06 官方运行路径

这一篇的命令只来自两处：配套仓库 README，和数据集卡 “Get Started”。两边重复的命令，卡片与 README 一致。没有从 Harbor 的 `--help` 里补旗标，也没有把 Archipelago 的本地命令混进 1.1 路径。

跑之前先知道三件事：

- 数据集卡禁止把这套数据用于训练、微调或参数拟合，也禁止爬取。
- 换代理修订版或换 `max_steps`，结果可能不能和榜比较。可比的默认是 100 步、10800 秒。官方命令没有覆盖这两个数。
- Archipelago 仓库里的 `examples/hugging_face_task` 指向数据集 `mercor/apex-agents`，不是 `mercor/apex-agents-v1.1`。那不是下面任何一条命令。两套题量的差在 [01 的对照节](01-what-is-apex-1.1.md)。

示例里的模型 `anthropic/claude-opus-5` 是 README 印出来的例子。README 说可以换成任何 LiteLLM 兼容的 `provider/model`。

## 两边都要做的准备

配套 README 和数据集卡都从这里开始。

安装 Docker 和 `uv`，再安装指定版本的 Harbor：

```bash
uv tool install harbor==0.20.0
```

克隆参考代理，复制环境变量示例：

```bash
git clone https://github.com/Mercor-Intelligence/apex_loop_truncated_tools_agent.git
cd apex_loop_truncated_tools_agent
cp .env.example .env
```

然后在 `.env` 里填写模型供应商的密钥，以及要用的判分凭据。数据集卡多了一句：`GRADING_MODEL` 留空时，使用任务配置好的 grader。示例文件里有哪些变量，见 [04-dependencies.md](04-dependencies.md)。

`PYTHONPATH="$PWD/apex_loop_truncated_tools_agent"` 出现在后面每一条 `harbor run` 前面。仓库结构是外层目录里再套一层同名 Python 包。Harbor 用 `-a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent` 按「模块:类」加载代理，所以 Python 要能导入这个包。这是从命令形状和仓库布局能直接看出来的，不是另加的隐藏步骤。

## 路径 A：Harbor Hub

不先把数据集克隆到本地。`-d` 指向 Hub 上的数据集引脚，`-i` 是一道题的 id。README 用的引脚是 `mercor/apex-agents-1-1@1.0.1`，和代理 `manifest.json` 的 `version` 1.0.1 相同。

跑一道题：

```bash
PYTHONPATH="$PWD/apex_loop_truncated_tools_agent" \
harbor run \
  --env-file .env \
  -d mercor/apex-agents-1-1@1.0.1 \
  -i 128-jr-1-f7f95d92 \
  -a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent \
  -m anthropic/claude-opus-5
```

跑整个评测：

```bash
PYTHONPATH="$PWD/apex_loop_truncated_tools_agent" \
harbor run \
  --env-file .env \
  -d mercor/apex-agents-1-1@1.0.1 \
  -a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent \
  -m anthropic/claude-opus-5
```

数据集卡写：Hub 上的 world、agent、verifier 镜像按 digest 钉死，第一次运行从公共 ECR 拉取。上面的命令本身没有 `docker pull`。拉取发生在 Harbor 第一次需要这些镜像时。

`128-jr-1-f7f95d92` 是 README 和数据集卡都选用的示例任务 id。本说明没有读取这道题的题面。

这两条 Hub 命令里没有 `-y`。不要从 Hugging Face 那一节把它抄过来。README 没有解释 `-y` 是什么。

## 路径 B：Hugging Face

README 这一节先要求安装 `hf` CLI。数据集是门禁的（Hub API：`gated: auto`）。未登录时，下载会像本说明核对 `manifest.json` 时一样被拒绝。需要在 Hugging Face 上取得访问后再执行。取得访问的点击流程不在 README 里，这里不编造。

下载交付物并装入三份共享镜像：

```bash
hf download mercor/apex-agents-v1.1 \
  --repo-type dataset \
  --local-dir apex-agents-v1.1

bash apex-agents-v1.1/environment/load_images.sh
```

`load_images.sh` 是交付物里面的脚本。数据集卡说 `environment/` 放的是共享镜像归档。脚本正文在门禁包里，本说明没有读，所以不解释它内部调用了哪些 `docker` 子命令。官方步骤就是执行这一行。

跑一道题：

```bash
PYTHONPATH="$PWD/apex_loop_truncated_tools_agent" \
harbor run \
  --env-file .env \
  -p apex-agents-v1.1/tasks/128-jr-1-f7f95d92 \
  -a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent \
  -m anthropic/claude-opus-5 \
  -y
```

跑整个评测：

```bash
PYTHONPATH="$PWD/apex_loop_truncated_tools_agent" \
harbor run \
  --env-file .env \
  -p apex-agents-v1.1/tasks \
  -a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent \
  -m anthropic/claude-opus-5
```

`-p` 指向本机目录：单题是一个任务目录，整榜是 `tasks` 目录。整榜那条没有 `-y`。单题那条有 `-y`。README 没有定义 `-y`。保持官方写法，不要自行去掉或加到整榜命令上。

数据集卡的 “Get Started” 与上面四条 Hugging Face 相关命令（安装 Harbor、下载、`load_images.sh`、两条 `harbor run`）一致。卡没有印 Harbor Hub 的 `-d` / `-i` 变体，那两段以配套 README 为准。

## 交付物里有、但 README 没有叫你执行的文件

数据集卡的目录树里有 `run_task.sh`，注释是 “Validated Harbor task launcher”。配套代理的报错文案也提到 `run_task.sh` 会从 `models.json` 解析模型。两份「怎么开始」都没有把 `bash run_task.sh ...` 印成步骤。本说明不发明这个脚本的参数。

`registry.json` 是任务 slug 到目录的映射。官方示例已经给出一个 slug，不需要先读这个文件才能跑通那一道。

## 命令里的旗标，只解释文档用到的

| 旗标 | 在哪条命令里 | 文档给出的含义 |
| --- | --- | --- |
| `--env-file .env` | 全部 `harbor run` | 把填好的密钥传进去 |
| `-d mercor/apex-agents-1-1@1.0.1` | 只在 Hub 路径 | Hub 数据集和版本引脚 |
| `-i 128-jr-1-f7f95d92` | 只在 Hub 单题 | 任务 id |
| `-p <目录>` | 只在 Hugging Face 路径 | 本机任务目录 |
| `-a apex_loop_truncated_tools_agent:ApexLoopTruncatedToolsAgent` | 全部 | 要加载的代理类 |
| `-m anthropic/claude-opus-5` | 全部 | 模型，可替换 |
| `-y` | 只在 Hugging Face 单题那条 | README 未定义 |

插件在没拿到 `-m` 时会直接报错，文案是：把 `-m <provider/model>` 传给 `harbor run`。没有隐藏的默认模型。

## 跑完以后官方文档没有写的事

配套 README 停在 `harbor run`。它没有写分数文件的路径，也没有写怎样把 Pass@1 聚合成榜。Archipelago 根 README 里的 `cat ./grades.json` 属于那个仓库自己的示例，而且示例指向的不是 `apex-agents-v1.1`。不要假设 1.1 的 Harbor 运行也在当前目录留下同名 `grades.json`。

数据集卡没有给出「怎样复现博客里 68.6%」的聚合脚本。单题命令用来确认链路。整榜命令会跑 `tasks` 下的任务，费用、时间和是否要并行，README 都没有写。

## 明确不要当作 1.1 复现的命令

Archipelago README 的快速开始包括：

```bash
cd examples/hugging_face_task
./run.sh
```

同一 README 写明，这是在跑 `mercor/apex-agents`。它会按那个仓库的方式启动环境、灌快照、跑代理、评分。那不是 1.1 的题、100 步循环和数据集卡上的 Harbor 命令。

Archipelago 环境 README 里的：

```bash
docker-compose --ansi always --env-file .env up --build
```

会在本机 8080 端口拉起一个空的环境网关，供你自己 POST `/apps`。它不加载 1.1 的 31 个世界，也不是榜上的运行。
