---
title: Agent Infra入门
date: 2026-09-24 23:13:11
tags:
- Agent Infra
---

写给第一次接触 agent 的人。先把道理讲清楚，再对照本项目的代码走一遍，最后才是操作手册。

<!-- more -->

| 部分 | 讲什么 | 需要看代码吗 |
|---|---|---|
| [第一部分 原理](#第一部分：agent-的基本原理) | agent 到底是什么；本项目每个组件解决什么问题 | 不需要 |
| [第二部分 对照代码](#第二部分：对照代码，看一个任务怎么跑完) | 文件在哪；程序跑在哪；一个任务从提交到完成经过了什么 | 边看边读 |
| [第三部分 开发](#第三部分：怎么开发（改它）) | 想改 agent 的行为，该改哪里、改完怎么验证 | 需要 |
| [第四部分 部署手册](#第四部分：部署层操作手册（查阅用）) | Lima、VM、docker、数据库的基本操作 | 查阅用 |

文中所有例子都来自 2026-09-24 在本机的真实运行，不是编的。

---

# 第一部分：agent 的基本原理

## 1. 起点：LLM 只会"读一段文字，写一段文字"

LLM（本项目用的是 DeepSeek）对外只有一个接口：**你发一串消息给它，它回你一条消息。**

这带来三个限制：

1. **它不能动手。** 它不能执行代码、不能上网、不能读你的文件。它只会产出文字。
2. **它没有记忆。** 每次调用都是全新的。所谓"它记得前面聊过什么"，其实是调用方每次都把之前的消息**重新发一遍**。
3. **它的"心算"不可靠。** 像"算一下这段文字的 SHA-256"这种事，它会一本正经地编一个答案出来。

**agent 就是围绕这三个限制搭起来的一段程序**：让 LLM 负责"想"，让程序负责"做"和"记"。

## 2. 工具调用：LLM 不动手，只"开单子"

调用 LLM 的时候，除了消息，还可以附上一份**工具清单**。下面是本项目的 `run_python` 工具真实发给 DeepSeek 的样子：

```json
{
  "type": "function",
  "function": {
    "name": "run_python",
    "description": "Run a Python 3.10 script in a fresh, isolated Firecracker microVM. Standard library only, no network access. ...",
    "parameters": {
      "type": "object",
      "properties": {
        "code":      {"type": "string"},
        "timeout_s": {"type": "number", "default": 30}
      },
      "required": ["code"]
    }
  }
}
```

LLM 看到这份清单后，可以不直接回答，而是回一张"单子"：**我要调用 `run_python`，参数是这些。**

下面是一次真实任务里 LLM 开出的单子：

```
[ai]  I'll run the script in the sandbox.
      tool_calls: run_python(code="import time\ntime.sleep(8)\nprint(\"ok\")", timeout_s=30)
      id: call_00_d7M8OT7q7Nu4TStMVbXF7482
```

然后是**我们的程序**（不是 LLM）照单执行：启动一台沙箱虚拟机、运行这段代码、拿到结果。结果会作为一条 `tool` 消息接到对话后面：

```
[tool] (回复 call_00_d7M8...)  {"exit_code": 0, "stdout": "ok\n", ..., "boot_ms": 1812.7, ...}
```

接着**再调一次 LLM**，把包括这条结果在内的整段对话发给它。它看完结果后，要么再开一张单子，要么给出最终答案：

```
[ai]  Done. The script ran in the sandbox: it slept 8 seconds, then printed `ok` ...
```

> **这一节最重要的一句话**：LLM 从头到尾都只是在"写文字"，所有真实的动作都是我们的代码做的。所以安全、权限、超时，全都在我们这一侧控制。

## 3. agent 循环：把"开单子 → 执行 → 回报"放进一个循环

把第 2 节的过程写成代码，就是这样（伪代码，不依赖任何框架）：

```python
messages = [system_prompt, user_task]
while True:
    reply = llm(messages, tools=TOOLS)          # 1. 问 LLM
    messages.append(reply)
    if not reply.tool_calls:                    # 2. 没开单子，说明做完了
        return reply.content
    for call in reply.tool_calls:               # 3. 开了单子，就照单执行
        result = TOOLS[call.name](**call.args)
        messages.append(tool_message(call.id, result))
    # 回到 1，带着新结果再问一次
```

**这十几行就是一个 agent。** 本项目其余的所有东西，都是为了让这个循环在真实环境里**可靠、安全、看得见**。

从这段伪代码里，还能直接看出两个事实：

- `messages` 每转一圈就变长一点，所以每次调用 LLM 的 token 数会越来越多。上面那个任务里，两次调用分别是 1116 → 1289 tokens。这也是为什么要给循环设一个"最多几轮"的上限。
- 调用哪个工具、传什么参数，是 LLM 决定的；循环什么时候结束，也是 LLM 决定的（它不再开单子就结束）。**程序只负责把循环的框架搭好。**

## 4. 从"能跑"到"能用"：每个组件解决一个问题

第 3 节那个最小循环，放到真实环境里马上会遇到下面这些问题。本项目的每个组件，都是在解决其中一个：

| 问题 | 最小循环里会怎样 | 本项目的解法 | 代码位置 |
|---|---|---|---|
| **流程不只是一个循环**：想先回忆、再规划、最后总结 | while 里塞满 if/else，越写越乱 | **LangGraph**：把流程画成一张图（节点 + 边），状态放在明面上 | `apps/agent/src/agent/graph.py` |
| **工具要能复用**，最好还能直接用别人写好的 | 工具是写死在程序里的函数 | **MCP**：工具是独立的进程，用标准协议发现和调用 | `sandbox_mcp.py`、`mcp-server-fetch` |
| **LLM 写的代码可能有危险** | 直接在你的电脑上执行 | **Firecracker 微虚拟机**：每次调用启动一台新 VM，没有网络，用完就销毁 | `sandbox/` |
| **程序跑到一半崩了** | 任务直接丢失，或者只能从头再来 | **Temporal**（发现崩溃、把活重新派出去）+ **Redis checkpoint**（记住做到了哪一步） | `workflows.py`、`activities.py` |
| **下次任务想用上这次的结果** | LLM 没有记忆 | **pgvector 长期记忆**：任务结束时存一句总结，新任务开始时按相似度找出来 | `memory.py` |
| **出了问题，不知道 LLM 看到了什么、为什么这么决定** | 只能到处 print | **Langfuse**：每个任务一条 trace，包含每次的 prompt、token 数、工具调用和耗时 | `activities.py` |
| **别人怎么提交任务、查进度** | 只能在终端里跑脚本 | **FastAPI 控制面**：HTTP 接口 | `api.py` |

下面逐个用大白话解释。

### 4.1 编排（LangGraph）：把循环画成一张图

本项目的流程图如下：

```
START → recall → plan → agent ──(LLM 开了单子)──→ tools
                          ↑                          │
                          └──────────────────────────┘
                          │
                          │(LLM 没开单子)
                          ↓
                       remember → END
```

- **节点**就是一个函数：读取当前状态，返回要更新的字段。
- **边**决定下一步去哪。`agent → tools → agent` 这个圈，就是第 3 节里的 while 循环。
- 比起裸写 while 循环，图多出来三样东西：
  - 前后可以挂额外步骤（`recall` 回忆、`plan` 规划、`remember` 总结）；
  - **状态**（任务、计划、消息列表……）集中放在一个明确的结构里（`AgentState`）；
  - **每走完一个节点，状态都会自动存一份快照**，也就是 checkpoint，见 4.4。

### 4.2 工具调用协议（MCP）：工具是另一个程序

最小循环里的 `TOOLS[call.name](...)` 是在同一个程序里直接调函数。MCP 把工具搬到了**另一个进程**里，两边通过标准的 JSON 消息对话：

- `tools/list`：问对方"你有哪些工具"，对方回一份清单（就是第 2 节里的那种 JSON）；
- `tools/call`：请对方执行某个工具。

好处是工具可以用任何语言写、可以复用，也可以直接接入别人写好的工具。本项目接了两个：

- **我们自己写的** `sandbox_mcp.py`：提供 `run_python` 和 `run_shell`，内部把请求转给沙箱；
- **别人写的** `mcp-server-fetch`：提供 `fetch`，用来抓取网页。

agent 这一侧的代码里，一个工具名都没有写死，全靠 `tools/list` 发现。

### 4.3 沙箱（Firecracker）：危险代码关进一次性虚拟机

LLM 写的代码不能直接在你的电脑上跑。本项目的做法是：**每调用一次 `run_python`，就启动一台全新的微虚拟机（microVM）**，在里面执行代码，拿到输出后立刻销毁这台 VM。

- 这台 VM 有独立的内核，**没有网卡**，写进去的文件也随 VM 一起消失。
- 代价是时间：每次启动大约 1.7 秒。第 2 节例子里的 `boot_ms: 1812.7` 就是这一项。

### 4.4 持久执行（Temporal + checkpoint）：崩了能接着做

需要分清两个问题：

- **"这个任务还有没有人在做？"** 由 Temporal 回答。干活的进程（worker）每 3 秒报一次平安（heartbeat）；超过 15 秒没动静，Temporal 就判定它死了，把任务重新派给另一个活着的 worker。
- **"做到哪一步了？"** 由 checkpoint 回答。每走完一个节点，状态都会存进 Redis。接手的 worker 先读取快照，从断掉的那个节点继续，前面的步骤不再重做。

两者缺一不可：没有 Temporal，崩了之后没人接手；没有 checkpoint，接手的人只能从头开始。

### 4.5 记忆：短期的和长期的

- **短期记忆 = 这个任务当前的状态**：消息列表、计划等，也就是 checkpoint。它只属于一个任务，24 小时后过期。
- **长期记忆 = 跨任务的笔记**。任务结束时，让 LLM 用一两句话总结"值得记住什么"，把这段话转成一串数字（向量，embedding）存进 Postgres。新任务开始时，把任务描述也转成向量，找出**意思最接近**的几条笔记，放进 prompt。

"意思接近"是靠向量之间的距离算出来的，所以用中文提问也能找到用英文写的笔记。

### 4.6 可观测性（Langfuse）：把 agent 的每一步摊开看

普通程序出错了看日志就行。agent 出错时，你想知道的是另一类问题：**LLM 当时看到了什么？它为什么选了这个工具？每一步花了多少时间、多少 token？**

Langfuse 会把一个任务记成一棵树（trace），树上每个节点叫一个 span：

```
attempt-1                      25.97s
  mcp-connect                  11.85s   ← 启动两个 MCP 进程
  LangGraph                    14.00s
    recall                      0.15s
    plan                        0.68s   ChatOpenAI tokens=408
    agent                       0.85s   ChatOpenAI tokens=1116
    tools                      10.13s   run_python（microVM boot 1812.7 ms）
    agent                       1.07s   ChatOpenAI tokens=1289
    remember                    0.92s   ChatOpenAI tokens=141
```

在 Langfuse 网页上点开任意一个 span，都能看到它完整的输入和输出。

---

# 第二部分：对照代码，看一个任务怎么跑完

## 5. 项目里哪些文件属于 Phase 1

```
agent-infra/
├── Makefile                     所有操作的入口（make 目标背后是什么，见第 15 节）
├── .env                         LLM 的地址、密钥和模型名（不进 git）
├── PLAN.md  README.md
├── docs/                        本文档和学习验证
│
├── apps/agent/                  ★ agent 本体，跑在 macOS 上（uv 管理的 Python 项目）
│   └── src/agent/
│       ├── config.py            所有地址、端口、常量集中放在这里
│       ├── api.py               控制面：HTTP :8000，负责提交、查询、取消任务
│       ├── worker.py            worker 进程的入口：连上 Temporal，开始领任务
│       ├── workflows.py         Temporal workflow：很薄的一层外壳，只调用一个 activity
│       ├── activities.py        真正干活的地方：启动 MCP、建图、跑图、报平安、写进度、记 trace
│       ├── graph.py             ★ agent 的大脑：LangGraph 流程图和三段 prompt
│       ├── memory.py            长期记忆：embed（转向量）、search（检索）、store（存储）
│       ├── sandbox_mcp.py       我们自己写的 MCP server：run_python / run_shell → 沙箱
│       └── watch.py  trace.py  history.py  recall.py    观察工具（不参与运行）
│
├── sandbox/                     沙箱，全部跑在 Lima 虚拟机里
│   ├── lima/fc-sandbox.yaml     虚拟机的定义：4 核 / 10 GiB / 30 GiB、嵌套虚拟化、挂载仓库
│   ├── executor/executor.py     沙箱执行器：HTTP :8088；一个请求 = 一台 microVM 从生到死
│   ├── executor/fc-executor.service   让执行器开机自动运行（systemd）
│   ├── guest/fc-init            microVM 里的第一个进程：准备好可写的文件系统
│   ├── guest/fc-agent.py        microVM 里负责接收命令、执行、回传结果
│   └── scripts/                 创建 VM、安装、兼容补丁、冒烟测试
│
├── stack/                       后端服务，跑在 Lima 虚拟机的 docker 里
│   ├── docker-compose.yml       Postgres、Redis、Temporal、Langfuse（及其依赖）
│   ├── postgres-init/           第一次启动时建库
│   └── scripts/install-docker.sh
│
└── apps/worker  apps/control-plane  helm/  scripts/  values/  secrets/
                                 Phase 2（k3s）的骨架，Phase 1 用不到，可以先忽略
```

如果只读三个文件，就读 **`graph.py` → `activities.py` → `sandbox_mcp.py`**。合计不到 400 行，agent 的主干全在里面。

## 6. 东西都跑在哪

一共三层"电脑"，一层套一层：

```
┌─ macOS（你的 Mac）───────────────────────────────────────────────────────┐
│  控制面 API     uvicorn agent.api:app            :8000                   │
│  worker         python -m agent.worker                                   │
│    ├─ python -m agent.sandbox_mcp       ← 每个任务启动一次，任务结束就退出 │
│    └─ uvx mcp-server-fetch → python     ← 同上                           │
│  limactl（Lima 的后台进程）：把 VM 里的端口映射到 Mac 的 127.0.0.1         │
│                                                                          │
│  ┌─ Lima 虚拟机 "fc-sandbox"（Ubuntu 24.04）─────────────────────────┐    │
│  │  /mnt/agent-infra     ← Mac 上的仓库，只读挂载进来                  │    │
│  │  /dev/kvm             ← 嵌套虚拟化提供，Firecracker 靠它运行        │    │
│  │  fc-executor（systemd 服务，root）                   :8088          │    │
│  │    └─ firecracker（用户 fc-jail）  ← 每次沙箱调用一台，用完就杀     │    │
│  │         └─ [microVM 里] fc-init → fc-agent.py                      │    │
│  │  docker：postgres :5432   redis :6379   temporal :7233 / UI :8233  │    │
│  │          langfuse-web :3000   langfuse-worker   clickhouse   minio │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

为什么要这样分层：

- **Firecracker 必须在 Linux 上跑**，而且需要 `/dev/kvm`，macOS 没有这个设备。所以要起一台 Linux 虚拟机，并开启嵌套虚拟化。
- **docker 也放进这台 VM**，因为项目不用 Docker Desktop。
- **agent 本体放在 macOS 上**，因为改代码、重启、`kill` 都最方便。
- 各层之间靠**端口映射**连通。Lima 会自动把 VM 里监听的端口映射到 Mac 的 `127.0.0.1`，所以 Mac 上的程序访问 `127.0.0.1:5432`，实际连到的是 VM 里的 Postgres。

## 7. 一个任务的一生

以一次真实任务为例：`task-b6b8b586`，任务内容是"Using the Python sandbox, sleep 8 seconds and then print ok."，总耗时 26 秒。

**① 提交**：`curl -X POST :8000/tasks` → `api.py` 的 `create_task`

- 生成一个任务 id，调用 `start_workflow`，把任务交给 Temporal，**立刻返回**，不等任务做完。
- 本项目没有自己的任务表：Temporal 记录的这份 workflow history 就是任务档案。

**② 排队**：Temporal 把任务放进 `agent-tasks` 队列，等 worker 来领。

- 如果没有 worker 在运行，任务就一直等着，不会报错。

**③ workflow**：`worker.py` 领到任务，执行 `workflows.py` 的 `AgentTaskWorkflow.run`

- 这里只有一行实质内容：`execute_activity("run_agent", ...)`，同时设好规矩：心跳 15 秒超时、最多重试 3 次、最多运行 15 分钟。
- 为什么不在这里直接调 LLM？因为 workflow 代码会被 Temporal**反复重放**，必须每次重放都得到相同的结果。LLM 做不到这一点，所以它只能放进 activity。

**④ activity 开工**：`activities.py` 的 `run_agent`

- 启动一个后台协程，每 3 秒向 Temporal 报一次平安（heartbeat）。
- 在 Langfuse 里打开一个名为 `attempt-1` 的 span。
- **启动两个 MCP server 子进程**，通过 stdin/stdout 对话。先握手，再调用 `tools/list`，拿到 3 个工具：`run_python`、`run_shell`、`fetch`。这一步本次花了 11.85 秒，平时约 0.5 秒，见第 17 节。
- 从 Redis 读取这个任务的 checkpoint：
  - 有，而且没做完，就接着做；
  - 已经做完，就直接返回；
  - 没有，就从头开始。

**⑤ 跑图**：`graph.py` 里的各个节点依次执行

| 节点 | 做什么 | 本次耗时 |
|---|---|---|
| `recall` | 把任务描述转成向量，在 pgvector 里找最接近的 3 条记忆（距离 ≤ 0.6） | 0.15 s |
| `plan` | 调一次 LLM，**不给工具**，只让它写一个最多 5 步的计划 | 0.68 s，408 tokens |
| `agent` | 调 LLM，**给工具**。system prompt 每次都**现场拼接**：规则 + 计划 + 记忆。LLM 回复"开单子：run_python" | 0.85 s，1116 tokens |
| `tools` | 照单执行，经过一长串环节，见下面的表 | 10.13 s |
| `agent` | 再调 LLM，这次消息里多了工具结果。LLM 不再开单子，直接给出答案 | 1.07 s，1289 tokens |
| `remember` | 调一次 LLM，把任务总结成 1–2 句话，转成向量存进 pgvector | 0.92 s，141 tokens |

每个节点跑完，都会发生三件事：状态存进 Redis（checkpoint）、进度写进 Redis Stream（`task-events:<id>`）、trace 推送给 Langfuse。

**一次 `run_python` 调用要穿过的环节**，这是整个项目里最长的一条链路：

| # | 从 → 到 | 怎么传 |
|---|---|---|
| 1 | LangGraph 的 `tools` 节点 → MCP 客户端 | 同一进程内的函数调用（`langchain-mcp-adapters`） |
| 2 | MCP 客户端 → `sandbox_mcp` 进程 | JSON-RPC `tools/call`，走 stdin/stdout |
| 3 | `sandbox_mcp` → `executor.py` | HTTP `POST 127.0.0.1:8088/v1/exec`，经 Lima 映射进入 VM |
| 4 | executor → firecracker | jailer 准备好隔离目录并降低权限，然后启动 microVM（本次 1.81 s） |
| 5 | executor → microVM 里的 `fc-agent` | vsock（虚拟机专用的通道，**不是网络**）：一行 JSON 发进去，一行 JSON 收回来 |
| 6 | `fc-agent` → 你的代码 | 在 VM 里执行 `python3 main.py`，本次 sleep 了 8 秒 |
| ← | 原路返回 | executor 拿到结果后立刻杀掉 VM、删掉隔离目录（10 ms） |

**⑥ 收尾**：`run_agent` 返回结果字典 → activity 完成 → workflow 完成。Temporal 的 history 里一共记了 11 条事件。

**⑦ 查询**：`GET /tasks/{id}` → `api.py` 的 `get_task`

- 向 Temporal 查询状态，从 Redis Stream 读取进度事件，如果任务已完成再附上结果。
- `python -m agent.watch` 就是每 0.5 秒调用一次这个接口，把结果打印出来。

## 8. 组件一览

| 组件 | 跑在哪 | 端口 | 在本项目里负责什么 |
|---|---|---|---|
| 控制面 API（`api.py`） | macOS | 8000 | 提交、查询、取消任务 |
| worker（`worker.py`） | macOS | — | 从队列领任务，执行 workflow 和 activity 的代码 |
| LangGraph | worker 进程内（一个 Python 库） | — | agent 的流程图、状态和 checkpoint |
| LLM（DeepSeek） | 外网 API | — | 规划、决定调用哪个工具、写答案、写总结 |
| `sandbox_mcp` | macOS，worker 的子进程 | stdio | 把沙箱包装成标准工具 |
| `mcp-server-fetch` | macOS，worker 的子进程 | stdio | 抓取网页（第三方工具，有网络权限） |
| fc-executor | Lima VM，systemd | 8088 | 每个请求：启动 VM → 执行 → 销毁 VM |
| Firecracker + jailer | Lima VM | — | 虚拟机管理程序 + 把它关进隔离目录的"看守" |
| fc-agent | microVM 内 | vsock 1024 | 在 VM 里真正执行命令 |
| Temporal | Lima VM，docker | 7233，UI 8233 | 任务档案、队列、心跳检测、重试、取消 |
| Redis | Lima VM，docker | 6379 | checkpoint（短期记忆）、进度流；也给 Langfuse 当队列 |
| Postgres + pgvector | Lima VM，docker | 5432 | `agent` 库的 `memories` 表（长期记忆）；`langfuse` 库 |
| fastembed 模型 | worker 进程内 | — | 在本地把文字转成 384 维向量，不调用外部 API |
| Langfuse（web + worker） | Lima VM，docker | 3000 | 接收并展示 trace |
| ClickHouse、MinIO | Lima VM，docker | 仅 VM 内部 | Langfuse 的存储后端，你不需要直接碰它们 |

---

# 第三部分：怎么开发（改它）

## 9. 改 agent 行为的四个旋钮

按"最常用"到"最少用"排列：

### 旋钮 1：改 prompt（`graph.py`）

agent 的"性格"和规则写在 `graph.py` 的三段 `SystemMessage` 里：

- `plan` 节点："写一个最多 5 步的计划，先不要执行"；
- `agent` 节点："任何计算都必须用沙箱工具，不许心算……"；
- `remember` 节点："写一两句值得记住的话"。

例如，想让它总是用中文回答，就在 `agent` 节点的 prompt 里加一句。**这是成本最低、效果最直接的改法。**

### 旋钮 2：加或改工具（`sandbox_mcp.py`）

加一个工具，就是加一个带 `@mcp.tool()` 装饰器的函数：

```python
@mcp.tool()
async def sandbox_info() -> str:
    """Describe the sandbox itself: OS release, Python version, CPU count, memory."""
    return await _exec({"command": "head -2 /etc/os-release; python3 -V; nproc; free -m | head -2"})
```

- **函数签名决定参数格式**：类型注解会被自动转成第 2 节里那种 JSON 描述。
- **docstring 就是写给 LLM 看的使用说明**，写得好不好，直接影响 LLM 会不会、该不该用它。
- `graph.py` 一行都不用改，worker 也不用重启：下一个任务启动 MCP 进程时，自然会发现这个新工具。这个效果已经实测过。

### 旋钮 3：改流程（`graph.py`）

加一个步骤，只需要三件事。以最简单的 `recall` 节点为模板：

```python
async def recall(state: AgentState):                  # 1. 写一个函数：读 state，返回要更新的字段
    return {"memories": await memory.search(state["task"])}

g.add_node("recall", recall)                          # 2. 注册成节点
g.add_edge(START, "recall")                           # 3. 用边把它接进流程
g.add_edge("recall", "plan")
```

如果新节点要存新的字段，还要在 `AgentState` 里加上这个字段。想做分支，就用 `add_conditional_edges`，写法参考 `route` 函数。

### 旋钮 4：改记忆策略（`memory.py` 和 `remember` 节点）

- 找几条：`k=3`；
- 多像才算相关：`max_distance=0.6`；
- 存什么：`remember` 节点的 prompt。

阈值调松，会召回更多无关的笔记；调紧，又会漏掉有用的笔记。可以用 `python -m agent.recall "..."` 先试一试效果。

## 10. 开发循环：改 → 重启 → 跑 → 看

**改了哪个文件，要做什么：**

| 改了 | 要做什么 |
|---|---|
| `graph.py`、`activities.py`、`memory.py`、`config.py`、`.env` | 重启 worker：在终端 1 按 Ctrl-C，然后 `make agent-worker` |
| `workflows.py` | 重启 worker。注意：如果改动时还有任务在跑，Temporal 重放旧 history 可能报 "non-determinism" 错误，所以最好等任务都跑完再改 |
| `api.py` | 重启 API：终端 2 按 Ctrl-C，然后 `make agent-api` |
| `sandbox_mcp.py` | 什么都不用做，下一个任务自动生效 |
| `sandbox/executor/executor.py` | 在 VM 里重启服务：`limactl shell --workdir / fc-sandbox -- sudo systemctl restart fc-executor`（服务直接从挂载目录读源码，不用拷贝） |
| `sandbox/guest/*`（microVM 里的文件） | `make sandbox-provision`，重新制作 microVM 的系统镜像 |
| `stack/docker-compose.yml` | `make stack-up`（compose 只会重建有变化的服务） |

**跑一个任务，然后从四个角度看它**（命令都在 `apps/agent` 目录下执行）：

| 想知道 | 命令 | 相当于 |
|---|---|---|
| 它走到哪一步了 | `uv run python -m agent.watch "任务内容"` | 进度条 |
| LLM 每一步看到了什么、花了多久、用了多少 token | `uv run python -m agent.trace <task-id>`，或打开 Langfuse http://127.0.0.1:3000 | 录像 |
| 每走完一步，状态是什么样子 | `uv run python -m agent.history <task-id>` | 存档列表 |
| 某句话会召回哪些记忆 | `uv run python -m agent.recall "某句话"` | 搜索框 |
| Temporal 视角：重试、失败原因 | 打开 http://127.0.0.1:8233 | 任务档案 |

**结果不对的时候，按这个顺序排查：**

1. 先看 trace：LLM 收到的 prompt 对不对？工具返回了什么？大部分问题出在 prompt 或工具的说明上。
2. 再看 history：状态是在哪一步变歪的？
3. 如果是沙箱报错，工具结果里会带上 `error` 和 `console_tail`（microVM 控制台输出的最后一段）。更多信息看执行器日志：`make sandbox-logs`。

---

# 第四部分：部署层操作手册（查阅用）

## 11. Lima 是什么

Lima 是在 macOS 上管理 Linux 虚拟机的命令行工具，命令叫 `limactl`，本机版本是 2.2.0。

- 一台 VM 由一个 YAML 文件定义，本项目的是 `sandbox/lima/fc-sandbox.yaml`。
- VM 运行期间，Mac 上有一个 `limactl` 后台进程（hostagent），负责**端口映射**和**目录挂载**。

## 12. Lima 基本操作

| 做什么 | 命令 | 说明 |
|---|---|---|
| 看 VM 状态 | `limactl list` | 能看到 `fc-sandbox  Running  ...  vz  aarch64  4  10GiB  30GiB` |
| 进入 VM 的 shell | `limactl shell --workdir / fc-sandbox` | 输入 `exit` 退出。为什么要加 `--workdir /`，见下文 |
| 在 VM 里执行一条命令 | `limactl shell --workdir / fc-sandbox -- uname -a` | Makefile 里全是这种写法 |
| 以 root 身份执行 | `limactl shell --workdir / fc-sandbox -- sudo <命令>` | VM 里的 sudo 不需要密码 |
| 停止 VM | `limactl stop fc-sandbox` | VM 里的所有东西都会停下（数据库、沙箱）。建议先停掉 worker 和 API |
| 启动 VM | `limactl start fc-sandbox` | 等价于 `make sandbox-vm`（后者在第一次运行时还会创建 VM） |
| 调整 CPU 和内存 | `limactl stop fc-sandbox && limactl edit fc-sandbox --cpus 6 --memory 12 --start` | 必须先停机 |
| 在 Mac 和 VM 之间拷贝文件 | `limactl copy 本地文件 fc-sandbox:/tmp/` | 反方向就把两个参数对调 |
| 用普通 ssh 登录 | `ssh -F ~/.lima/fc-sandbox/ssh.config lima-fc-sandbox` | VS Code Remote-SSH 也能这样连 |
| **删除 VM** | `limactl delete fc-sandbox` | **不可恢复**：数据库、记忆、trace、镜像全部丢失 |

几个需要知道的事实：

- **`--workdir /` 的原因**：`limactl shell` 默认会切换到"你在 Mac 上的当前目录"。但 VM 里只挂载了仓库本身（挂在 `/mnt/agent-infra`），没有 `/Users/...` 这个路径，于是会先报 `cd: /Users/sandszy: No such file or directory`，再退回 VM 里的家目录 `/home/sandszy.guest`。加上 `--workdir /` 就不会报这个错。
- **挂载**：Mac 上的仓库**只读**挂载在 VM 的 `/mnt/agent-infra`（virtiofs）。在 Mac 上改了代码，VM 里立刻就能看到；VM 不能反过来改仓库。
- **端口映射**：Mac 上执行 `lsof -iTCP -sTCP:LISTEN -n -P | grep limactl`，可以看到 `limactl` 在 Mac 上替 VM 监听 3000、5432、6379、7233、8088、8233 这些端口。
- **代理**：VM 里的 `/etc/environment` 被 Lima 自动写入了 `HTTPS_PROXY=http://192.168.5.2:7897`，也就是 Mac 上的代理。但 dockerd 不读这个文件，需要单独配置（`make stack-docker` 已经配好，详见 `AGENTS.md`）。
- **VM 自己的文件**：都在 `~/.lima/fc-sandbox/`：
  - `lima.yaml`：实际生效的配置；
  - `serialv.log`：VM 启动时的控制台输出；
  - `ha.stderr.log`：hostagent 的日志，端口映射出问题时看这里。

## 13. VM 里的两个服务

VM 里有两个 systemd 服务，开机会自动启动。

**fc-executor**（沙箱执行器）：

```bash
limactl shell --workdir / fc-sandbox -- systemctl status fc-executor       # 状态
limactl shell --workdir / fc-sandbox -- sudo journalctl -u fc-executor -f  # 实时日志（= make sandbox-logs）
limactl shell --workdir / fc-sandbox -- sudo systemctl restart fc-executor # 改了 executor.py 之后
curl -s --noproxy '*' http://127.0.0.1:8088/healthz                        # 在 Mac 上检查它是否健康
```

它在 VM 里用到的文件：

| 路径 | 是什么 |
|---|---|
| `/usr/local/bin/firecracker`、`jailer` | v1.10.1 |
| `/var/lib/fc-sandbox/images/vmlinux` | microVM 的内核（6.1.102） |
| `/var/lib/fc-sandbox/images/rootfs.ext4` | microVM 的系统盘：Ubuntu 22.04，已经放入 `fc-init` 和 `fc-agent.py`，只读 |
| `/var/lib/fc-sandbox/runs/` | 出错时保留的 microVM 控制台日志 |
| `/srv/jailer/firecracker/<vm-id>/root/` | 正在运行的 microVM 的隔离目录，VM 销毁后自动删除 |

**docker**：容器都设置了 `restart: unless-stopped`，VM 重启后会自动拉起。

## 14. docker compose 与数据的直接访问

Makefile 里的 `make stack-*` 实际上执行的是：

```bash
limactl shell --workdir / fc-sandbox -- sudo docker compose -f /mnt/agent-infra/stack/docker-compose.yml <子命令>
```

常用的子命令：

- `ps`：看状态；
- `logs -f temporal`：跟踪某个服务的日志；
- `restart langfuse-web`：重启某个服务；
- `up -d --wait`：启动并等待所有服务就绪；
- `down`：停止。数据保留在 volume 里；加上 `-v` 会**连数据一起删**。

**直接查看 agent 的数据**（全部是只读查询，放心执行）：

```bash
# 长期记忆：Postgres
limactl shell --workdir / fc-sandbox -- sudo docker exec agent-infra-postgres-1 \
  psql -U postgres -d agent -c "SELECT task_id, left(content, 80) FROM memories ORDER BY id"

# 短期记忆和进度：Redis
limactl shell --workdir / fc-sandbox -- sudo docker exec agent-infra-redis-1 redis-cli --scan --pattern 'task-events:*'
limactl shell --workdir / fc-sandbox -- sudo docker exec agent-infra-redis-1 redis-cli XRANGE task-events:<task-id> - +

# 任务档案：Temporal 命令行
limactl shell --workdir / fc-sandbox -- sudo docker exec agent-infra-temporal-1 temporal workflow list --limit 5
limactl shell --workdir / fc-sandbox -- sudo docker exec agent-infra-temporal-1 temporal workflow show -w <task-id>
```

最后一条命令会列出 11 条事件：`WorkflowExecutionStarted … ActivityTaskScheduled → ActivityTaskStarted → ActivityTaskCompleted … WorkflowExecutionCompleted`。整个 agent 的运行，在 Temporal 眼里就只是"一个 activity"。

## 15. make 目标背后的真实命令

| make 目标 | 实际执行的操作 |
|---|---|
| `sandbox-vm` | `sandbox/scripts/vm-up.sh`：第一次运行时执行 `limactl create`，之后执行 `limactl start`，最后检查 `/dev/kvm` 是否存在 |
| `sandbox-provision` | 在 VM 里以 root 执行 `sandbox/scripts/provision.sh`：下载 firecracker/jailer 和内核、系统盘；把 `fc-init`、`fc-agent.py` 放进系统盘；安装并重启 `fc-executor` 服务 |
| `sandbox-test` | 在 Mac 上执行 `python3 sandbox/scripts/smoke-test.py`，直接调用 `:8088` 做 8 项检查 |
| `sandbox-logs` | `journalctl -u fc-executor -f` |
| `stack-docker` | 在 VM 里执行 `stack/scripts/install-docker.sh`：安装 docker 和 compose，给 dockerd 配代理 |
| `stack-up` / `stack-down` / `stack-ps` | `docker compose ... up -d --wait` / `down` / `ps` |
| `agent-worker` | `cd apps/agent && uv run python -m agent.worker` |
| `agent-api` | `cd apps/agent && uv run uvicorn agent.api:app --host 127.0.0.1 --port 8000` |

## 16. 开机、关机、从零搭建

**Mac 重启之后**，VM 不会自动启动：

```bash
limactl start fc-sandbox        # docker 容器和 fc-executor 会跟着自动起来
make stack-ps                   # 确认 7 个容器都是 Up；不放心的话再执行一次 make stack-up
make agent-worker               # 终端 1
make agent-api                  # 终端 2
```

**用完关机**：终端 1、2 各按一次 Ctrl-C，然后执行 `limactl stop fc-sandbox`。

**从零搭建**（新机器，或者执行过 `limactl delete` 之后）：

```bash
make sandbox-vm && make sandbox-provision && make sandbox-test
make stack-docker && make stack-up
# 在仓库根目录的 .env 里填好 LLM_BASE_URL / LLM_API_KEY / LLM_MODEL
make agent-worker    # 终端 1
make agent-api       # 终端 2
```

## 17. 排障速查

| 现象 | 先查什么 |
|---|---|
| `curl :8088/healthz` 没有响应 | `limactl list` 看 VM 是否 Running → `systemctl status fc-executor` → `journalctl -u fc-executor -n 50` |
| 工具结果里有 `error` | 结果里的 `console_tail`；VM 里的 `/var/lib/fc-sandbox/runs/<id>.console.log` |
| worker 启动时连不上 Temporal | `make stack-ps`，看 temporal 容器是否在运行 |
| 访问本地服务却走了代理，报 502 之类的错误 | 命令里加 `--noproxy '*'`；Python 代码里用 `trust_env=False`（`config.py` 已经处理了 NO_PROXY） |
| 拉取 docker 镜像失败 | 在 VM 里执行 `systemctl show docker -p Environment`，看 dockerd 有没有配代理 |
| VM 里能访问某个端口，Mac 上访问不了 | `~/.lima/fc-sandbox/ha.stderr.log`（端口映射的日志） |
| `mcp-connect` 偶尔要十几秒 | `uvx mcp-server-fetch` 没有固定版本，每次都要检查 PyPI 上的最新版本。本地的索引缓存过期后，要通过代理重新联网确认，所以会慢 |
| LLM 调用失败 | `.env` 里的 `LLM_*` 配置；外网 API 需要走代理。用 `make agent-worker` 启动时，Makefile 会自动导出 `HTTPS_PROXY=http://127.0.0.1:7897`；如果你手动执行 `uv run python -m agent.worker`，shell 里得自己有这个变量 |

---

## 术语表

| 词 | 在本项目里的意思 |
|---|---|
| agent | 一个"问 LLM → 执行它要的工具 → 把结果告诉它"的循环程序（第 3 节） |
| tool call / 工具调用 | LLM 回复里的一张"单子"：调用哪个工具、传什么参数 |
| prompt / system prompt | 发给 LLM 的文字；system prompt 是放在最前面的规则说明 |
| token | LLM 计量文字的单位，费用和速度都按它算 |
| MCP | 工具的标准协议：工具是一个独立的进程，用 JSON-RPC 通信 |
| stdio | 进程的标准输入和输出。MCP 客户端就是通过子进程的 stdin/stdout 和它对话的 |
| 节点 / 边（LangGraph） | 流程图里的一个步骤 / 步骤之间的连线 |
| state / 状态 | 流程图里所有节点共享的数据：任务、计划、消息列表等 |
| checkpoint | 每走完一个节点存下的一份状态快照（在 Redis 里） |
| workflow / activity（Temporal） | workflow 是必须能重放的"流程外壳"；activity 是可以做任何事的"真正干活的函数" |
| heartbeat / 心跳 | activity 定期报平安；超时不报，就被判定为死亡 |
| worker | 从 Temporal 队列里领任务、执行代码的进程 |
| VM / microVM | 虚拟机 / 精简到极致、启动很快的虚拟机（Firecracker） |
| vsock | 宿主和虚拟机之间的专用通信通道，不经过网络 |
| embedding / 向量 | 把一段文字转成一串数字，意思越接近，数字之间的距离越小 |
| trace / span | 一次任务的完整记录 / 其中的一个步骤 |
