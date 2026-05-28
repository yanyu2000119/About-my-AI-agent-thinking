# Cairn 的 Prompt 设计解读

> 阅读对象：[`oritera/Cairn`](https://github.com/oritera/Cairn)
>
> 阅读日期：2026-05-28
>
> 关注范围：`cairn/src/cairn/dispatcher/prompts/default/` 里的默认 prompt，以及 Dispatcher 如何调用这些 prompt。

## 一句话结论

Cairn 的 prompt 不是一个“万能 Agent 提示词”，而是一套围绕状态空间搜索设计的任务模板。它把 Agent 拆成三种工作模式：

1. 初始推进者：从起点和目标开始，直接尝试推进问题。
2. 图状态规划者：读取现有事实图，判断是否完成目标，或者提出下一批探索方向。
3. 单意图探索者：只围绕一个具体 intent 执行探索，并把新增事实写回图。

所有 Agent 的输出都被要求是结构化 JSON，方便 Cairn Server 把结果写回 `Fact / Intent / Hint` 图。

## Prompt 文件位置

默认 prompt 位于：

```text
cairn/src/cairn/dispatcher/prompts/default/
```

主要文件包括：

```text
bootstrap.md
reason.md
explore.md
bootstrap_conclude.md
explore_conclude.md
```

它们不是直接调用 OpenAI 或 Anthropic API 的 system/user messages，而是由 Dispatcher 渲染成一整段文本，再传给底层 worker，例如 Claude Code、Codex 或 Pi CLI。

## Dispatcher 如何加载 Prompt

Cairn 通过配置项选择 prompt 组：

```yaml
runtime:
  prompt_group: default
```

逻辑大致是：

1. `dispatch.yaml` 或运行时配置里指定 `prompt_group`。
2. Dispatcher 从 `cairn.dispatcher.prompts.<prompt_group>` 读取 Markdown 模板。
3. 读取后做简单占位符替换，例如 `{origin}`、`{goal}`、`{graph_yaml}`、`{hints}`。
4. 渲染后的完整 prompt 被交给 worker 执行。
5. worker 返回 JSON，Dispatcher 解析 JSON，并把结果写入图数据库。

这意味着 Cairn 的 prompt 重点不是“语言风格”，而是“任务边界”和“输出协议”。

## 核心概念

Cairn 的 prompt 反复围绕三个对象组织：

| 概念 | 含义 | 作用 |
| --- | --- | --- |
| `Fact` | 已确认事实 | 表示当前世界状态，例如发现服务、拿到凭据、验证漏洞等 |
| `Intent` | 待探索方向 | 表示下一步值得尝试的行动，例如枚举目录、测试登录、分析源码 |
| `Hint` | 人类提示 | 外部注入的策略建议，不等于事实 |

Agent 不能随意把猜测写成事实。只有已经验证或有明确证据支撑的内容，才应该成为 `Fact`。

## 1. `bootstrap.md`：初始推进 Prompt

### 使用场景

项目刚启动时使用。此时图里还没有足够的事实和 intent，Agent 需要直接从 `origin` 和 `goal` 出发，尝试推进任务。

### 输入内容

模板会注入：

```text
Origin: 起始状态
Goal: 目标状态
Hints: 人类给出的提示
```

### Agent 的任务

Bootstrap Agent 要做的事情可以理解为：

```text
你现在只有起点、目标和提示。
请尝试直接推进任务。
如果目标已经达成，返回 complete。
如果没有达成，返回你确认得到的新事实。
不要把未验证的猜测写成事实。
输出必须是 JSON。
```

### 典型输出

如果目标完成：

```json
{
  "status": "complete",
  "summary": "目标已经完成的简要说明",
  "fact": {
    "content": "最终确认的关键事实"
  }
}
```

如果目标未完成但发现了新事实：

```json
{
  "status": "incomplete",
  "fact": {
    "content": "本轮确认的新事实"
  }
}
```

### 设计意图

`bootstrap.md` 负责“破冰”。它不先做复杂规划，而是让 Agent 在最少上下文下尝试获得第一批有效事实。

## 2. `reason.md`：规划与分解 Prompt

### 使用场景

当系统已经有一些事实后，Dispatcher 会启动 Reason 任务。Reason Agent 不负责执行具体探索，而是负责看完整张图，然后决定下一步应该探索什么。

### 输入内容

模板会注入：

```text
Goal: 最终目标
Graph: 当前 Fact / Intent 图快照
Available Fact IDs: 可引用的事实 ID
Open Intents: 当前仍未完成的探索方向
Max Intents: 本轮最多创建多少个新 intent
Hints: 人类提示
```

### Agent 的任务

Reason Agent 的工作可以理解为：

```text
请读取完整事实图。
先判断 goal 是否已经被事实满足。
如果已经满足，返回 complete。
如果没有满足，请提出少量高价值、互不重复、可以并行探索的 intents。
每个 intent 都必须基于已有 facts，并说明它为什么值得探索。
不要生成泛泛而谈的任务。
不要重复已有 open intents。
输出必须是 JSON。
```

### 典型输出

如果目标已完成：

```json
{
  "status": "complete",
  "summary": "为什么现有事实已经满足目标"
}
```

如果还需要探索：

```json
{
  "status": "incomplete",
  "intents": [
    {
      "content": "下一步要探索的具体方向",
      "reason": "为什么这个方向基于现有事实值得做",
      "based_on": ["fact-id-1", "fact-id-2"]
    }
  ]
}
```

### 设计意图

`reason.md` 是 Cairn 的“搜索策略层”。它避免所有 worker 都盲目乱试，而是让一个专门的规划步骤不断从事实图里派生高质量 intent。

这个设计有两个好处：

1. 可并行：多个 intent 可以分发给不同 worker。
2. 可审计：每个 intent 都能追溯到它依赖的 fact。

## 3. `explore.md`：单 Intent 探索 Prompt

### 使用场景

当 Reason 任务生成 intent 后，Dispatcher 会把某个 intent 分配给 Explore Agent。Explore Agent 只负责这个 intent，不应该扩大范围做别的事情。

### 输入内容

模板会注入：

```text
Goal: 最终目标
Graph: 当前事实图
Intent: 当前被分配的探索方向
Hints: 人类提示
```

### Agent 的任务

Explore Agent 的工作可以理解为：

```text
你只能围绕当前 intent 探索。
请根据完整事实图理解上下文，但不要重复已有事实。
如果探索有结果，返回一个新增 fact。
如果探索失败，也返回已经确认的负面事实或阻塞原因。
不要把猜测、计划或未执行的想法写成 fact。
输出必须是 JSON。
```

### 典型输出

```json
{
  "status": "incomplete",
  "fact": {
    "content": "围绕当前 intent 得到的新事实",
    "based_on": ["intent-id", "fact-id"]
  }
}
```

如果探索直接完成目标，也可以返回：

```json
{
  "status": "complete",
  "summary": "目标完成的证据说明",
  "fact": {
    "content": "完成目标所依赖的关键事实"
  }
}
```

### 设计意图

`explore.md` 是 Cairn 的“执行层”。它把 Agent 的自由度压缩到一个 intent 内，减少跑偏、重复劳动和上下文污染。

## 4. `bootstrap_conclude.md`：Bootstrap 兜底总结 Prompt

### 使用场景

当 Bootstrap Agent 超时、输出格式错误，或者没有正常返回 JSON 时，如果底层 worker 支持续写会话，Dispatcher 会追加一个 conclude prompt。

### Agent 的任务

这个兜底 prompt 的核心意思是：

```text
停止继续操作。
不要再运行命令。
不要继续探索。
只根据当前会话中已经确认的信息，返回符合格式的 JSON。
```

### 设计意图

它解决的是工程稳定性问题。即使 Agent 前面跑偏了或忘记输出 JSON，Dispatcher 仍然尝试把已经确认的信息回收成结构化结果。

## 5. `explore_conclude.md`：Explore 兜底总结 Prompt

### 使用场景

类似 `bootstrap_conclude.md`，但用于 Explore 任务。

### Agent 的任务

它要求 Agent：

```text
不要继续探索当前 intent。
不要新增动作。
只总结已经确认、和当前 intent 有关的信息。
返回一个可解析的 JSON。
```

### 设计意图

Explore 任务是并行执行的，如果某个 worker 没有正常收尾，conclude prompt 可以尽量保留它已经发现的有效事实，避免整轮探索结果丢失。

## Prompt 的总体设计特点

### 1. 任务边界非常清晰

Cairn 不让一个 Agent 同时负责所有事情，而是用不同 prompt 把工作拆开：

```text
Bootstrap: 先获得初始事实
Reason: 基于事实图做规划
Explore: 执行单个探索方向
Conclude: 异常情况下回收结果
```

### 2. 输出强制结构化

所有 prompt 都要求返回 JSON。这让 Dispatcher 可以机械地解析结果，而不是依赖自然语言总结。

### 3. 事实优先，猜测靠后

Prompt 反复强调：只有确认过的信息才能写成 fact。猜测、计划、假设不能污染事实图。

### 4. 通过图实现多 Agent 协作

Worker 之间不直接聊天。它们通过共享图协作：

```text
Reason 读图 -> 生成 intents
Explore 认领 intent -> 写入 facts
Reason 再读图 -> 判断目标或继续生成 intents
```

### 5. Prompt 更像协议，而不是话术

Cairn 的 prompt 不是为了“让模型更聪明”而写长篇角色扮演，而是定义了一个可执行协议：

```text
输入是什么
只能做什么
不能做什么
输出是什么
如何被系统消费
```

## 用中文重写后的简化版 Prompt

下面不是原项目 prompt 的逐字翻译，而是更容易理解的中文等价表达。

### Bootstrap 简化版

```text
你是 Cairn 的初始探索 Agent。

输入：
- 起点 origin
- 目标 goal
- 人类提示 hints

任务：
- 从起点出发尝试推进目标。
- 如果目标已经完成，返回 complete。
- 如果目标没完成，返回你确认的新事实。
- 不要输出猜测。
- 不要输出 Markdown。
- 只输出 JSON。
```

### Reason 简化版

```text
你是 Cairn 的规划 Agent。

输入：
- 当前事实图
- 最终目标
- 已有 open intents
- 可引用的 fact ids
- 本轮最多可生成的 intent 数量

任务：
- 判断当前 facts 是否已经满足目标。
- 如果满足，返回 complete。
- 如果不满足，生成少量具体、可执行、互不重复的探索 intents。
- 每个 intent 都要说明依据哪些 facts。
- 不要重复已有 intents。
- 只输出 JSON。
```

### Explore 简化版

```text
你是 Cairn 的执行 Agent。

输入：
- 当前事实图
- 当前分配给你的 intent
- 最终目标
- 人类提示

任务：
- 只围绕当前 intent 探索。
- 可以参考完整事实图，但不要重复已有事实。
- 如果有发现，返回一个新增 fact。
- 如果没有成功，也返回确认过的失败原因或阻塞点。
- 不要把计划或猜测写成 fact。
- 只输出 JSON。
```

### Conclude 简化版

```text
你现在需要停止继续操作。

任务：
- 不要再执行命令。
- 不要继续探索。
- 只基于已经确认的信息做总结。
- 返回符合要求的 JSON。
```

## 对我们设计 AI Agent 的启发

Cairn 的 prompt 设计最值得借鉴的地方，不是某一句具体提示词，而是下面几个工程原则：

1. 把“思考”和“执行”拆开。
2. 把长期上下文放到结构化状态图里，而不是全塞进对话历史。
3. 让每个 Agent 只负责一个窄任务。
4. 要求 Agent 输出机器可解析结果。
5. 异常情况下用 conclude prompt 回收已确认结果。
6. 对事实、猜测、计划做严格区分。

如果自己做 Agent 系统，可以借鉴这个结构：

```text
State Store: 保存 facts / tasks / hints
Planner Prompt: 读状态，产出下一批 tasks
Worker Prompt: 执行单个 task，产出新 fact
Verifier Prompt: 判断 fact 是否可信
Conclude Prompt: 超时或格式错误时回收结果
```

## 参考链接

- Cairn 仓库：<https://github.com/oritera/Cairn>
- 默认 prompt 目录：<https://github.com/oritera/Cairn/tree/main/cairn/src/cairn/dispatcher/prompts/default>
- Dispatcher prompt 加载逻辑：<https://github.com/oritera/Cairn/blob/main/cairn/src/cairn/dispatcher/prompting.py>
- Dispatcher 配置逻辑：<https://github.com/oritera/Cairn/blob/main/cairn/src/cairn/dispatcher/config.py>
