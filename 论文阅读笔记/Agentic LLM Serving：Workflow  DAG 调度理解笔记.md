# Agentic LLM Serving：Workflow / DAG 调度理解笔记

## 1. 核心问题：从「请求调度」变成「工作流调度」

传统 LLM Serving 通常把一次用户请求看成一个独立任务：

```
User Request
     ↓
LLM Request
     ↓
Prefill → Decode
     ↓
Response
```

调度器主要关心：

- 哪个请求先执行
- 哪个请求进入哪个 GPU
- Prefill / Decode 怎么分配
- 如何降低单请求延迟
- 如何提高 GPU 利用率

但 Agent 出现以后，一个用户请求往往不再对应一次 LLM 调用。

例如：

```
用户：
"帮我分析广州某区域的交通情况，并给出建议"

          ↓

      Agent Workflow
          │
          ├── LLM：分析需求
          │
          ├── Tool：查询 GIS 数据
          │
          ├── LLM：分析交通指标
          │
          ├── Tool：计算等时圈
          │
          └── LLM：生成最终报告
```

因此，一个用户请求实际上变成：

```
Workflow = 多个有依赖关系的 LLM / Tool Task
```

也就是一个 **DAG（Directed Acyclic Graph，有向无环图）**。

------

# 2. Agent Workflow 为什么天然适合 DAG

例如：

```
        A
       / \
      B   C
       \ /
        D
```

表示：

```
A → B
A → C
B → D
C → D
```

其中：

- A：分析用户需求
- B：查询交通设施
- C：计算人口/就业
- D：综合分析

D 必须等：

```
B 完成 AND C 完成
```

才能执行。

因此不能简单把整个 Agent 看成一个请求，而应该把它拆成：

```
Workflow
 ├── Task A
 ├── Task B
 ├── Task C
 └── Task D
```

调度问题也从：

```
Request A vs Request B 谁先执行？
```

变成：

```
Workflow 中哪些 Task 应该优先获得资源？
```

------

# 3. 传统 LLM Serving 的局限

传统系统：

- ORCA
- vLLM
- SGLang
- Sarathi-Serve
- Llumnix
- ServerlessLLM

主要围绕 **request-centric scheduling**。

也就是：

```
Request 1
Request 2
Request 3
Request 4
      ↓
Scheduler
      ↓
GPU
```

调度单位通常是：

```
Request / LLM Call
```

但是 Agent 中：

```
Request
   ↓
Workflow
   ↓
Task1 → Task2
   │
   ├→ Task3
   │
   └→ Task4
```

单独看 Task2、Task3、Task4 并不能完全理解它们的重要程度。

例如：

```
        A
       / \
      B   C
       \ /
        D
```

如果 C 是：

```
A → C → D
```

上的任务，那么 C 延迟可能直接导致 D 延迟。

而 B 即使延迟一些，只要：

```
B 在 C 完成之前完成
```

就不会影响 D。

所以：

> **Task 的调度优先级不仅取决于 Task 本身，还取决于它在 Workflow 中的位置。**

------

# 4. Prefill / Decode Disaggregation

LLM 推理通常可以拆成：

```
Prompt
  ↓
Prefill
  ↓
KV Cache
  ↓
Decode
  ↓
Token1
  ↓
Token2
  ↓
...
```

两阶段特点不同。

### Prefill

特点：

- 计算密集
- 输入 Prompt 较大时计算量明显
- 更偏 Compute-bound

### Decode

特点：

- 一次生成一个 Token
- 多次迭代
- KV Cache 占用显存
- 更偏 Memory / bandwidth constrained

因此很多 Serving 系统会进行：

```
          ┌── Prefill GPU Pool
Request ──┤
          └── Decode GPU Pool
```

即：

> **Prefill-Decode Disaggregation（P-D 分离）**

这会进一步增加调度复杂度。

因为 Agent Workflow 中一个 Task 可能产生：

```
Prefill → Decode
```

然后 Workflow 又继续：

```
Task A
 ↓
Task B
 ↓
Task C
```

所以实际上存在：

```
Workflow dependency
+
Prefill/Decode resource dependency
```

两个层面的约束。

------

# 5. Agent Serving 的几个发展方向

论文 Related Work 大致可以分成四层。

## 5.1 Request-level Scheduling

最传统：

```
Request
   ↓
Scheduler
   ↓
GPU
```

目标主要是：

- latency
- throughput
- GPU utilization

------

## 5.2 Call-level Scheduling

进一步把请求拆成 LLM Call：

```
Request
 ├── Call1
 ├── Call2
 └── Call3
```

调度算法开始使用：

- SJF
- SRPT
- Slack
- Attained Service
- Embedding similarity
- LTR

核心思想：

> 不同 LLM Call 的执行时间和价值不同，因此不能简单 FIFO。

但问题是：

**Call-level scheduler 不一定知道这个 Call 在整个 Agent Workflow 中的重要性。**

------

# 6. Workflow-aware / Program-aware Scheduling

进一步：

```
Request
   ↓
Program / Workflow
   ↓
DAG
   ↓
Task
```

例如：

```
       A
      / \
     B   C
      \ /
       D
```

Scheduler 不再只知道：

```
B 要执行 2s
C 要执行 10s
```

而是知道：

```
B → D
C → D
```

因此可以分析：

- Dependency
- Critical Path
- Slack
- Remaining Work
- Workflow Completion Time

这就是 **Workflow-aware Scheduling**。

------

# 7. Critical Path（关键路径）

这是理解论文最重要的概念之一。

假设：

```
        A(5)
       /   \
     B(2)  C(10)
       \   /
        D(2)
```

路径：

```
A → B → D = 5 + 2 + 2 = 9

A → C → D = 5 + 10 + 2 = 17
```

所以：

```
Critical Path = A → C → D
```

长度：

```
17
```

------

# 8. Critical Path 真正表达的是什么？

不要简单理解成：

> “关键路径上的任务都是长任务。”

真正含义是：

> **关键路径上的任务，其延迟更容易传递到整个 Workflow 的最终完成时间。**

例如：

```
        A
       / \
      B   C
       \ /
        D
```

如果：

```
B = 2s
C = 10s
D = 2s
```

那么：

```
D.start = max(B.finish, C.finish)
```

如果 A 在 5s 完成：

```
B finish = 7s
C finish = 15s

D start = max(7,15)
        = 15s
```

所以：

```
B 再晚一点
```

只要：

```
B finish <= 15s
```

都不会影响 D。

但是：

```
C finish 从 15s → 18s
```

那么：

```
D start 从 15s → 18s
```

整个 Workflow 都被拖延。

------

# 9. Slack（松弛时间）

这比简单的 Critical Path 更重要。

Slack 可以理解成：

> 一个任务最多可以延迟多少，而不影响整个 Workflow 的最终完成时间。

继续看：

```
A(5)
├── B(2)
└── C(10)
     ↓
     D(2)
```

最终：

```
A → C → D = 17s
```

B 路径：

```
A → B → D = 9s
```

因此 B 有：

```
Slack = 17 - 9
      = 8s
```

而 C：

```
Slack = 17 - 17
      = 0
```

所以：

| Task | Path | Slack | 延迟影响                     |
| ---- | ---- | ----- | ---------------------------- |
| B    | 9s   | 8s    | 延迟 8s 内通常不影响最终完成 |
| C    | 17s  | 0s    | 延迟会直接影响最终完成       |
| D    | 17s  | 0s    | 延迟会直接影响最终完成       |

因此：

```
Slack = 0
```

意味着：

> 这个任务不能随便拖。

------

# 10. 一个非常重要的误区：Critical Path ≠ 长任务优先

这是前面讨论中最容易混淆的地方。

例如：

```
A
├── B(2)
└── C(10)
     ↓
     D
```

如果 B 和 C：

```
共享同一个 GPU
```

而且必须一个执行完才能执行另一个。

那么：

### B → C

```
2 + 10 = 12s
```

### C → B

```
10 + 2 = 12s
```

最终 D 都要等：

```
12s
```

因此：

> **单纯把 Critical Path 上的 C 放前面，并不会自动降低 Workflow Makespan。**

这是非常重要的。

------

# 11. 那 Critical Path 为什么还有意义？

关键在于：

> **调度通常不是只有一个 Workflow，也不是所有任务都能无限并行执行。**

例如：

```
Workflow 1
A → B → C

Workflow 2
X → Y → Z

Workflow 3
P → Q
```

GPU 只有：

```
2 个执行槽
```

此时 Scheduler 必须决定：

```
现在应该给谁 GPU？
```

如果：

```
Workflow 1 的 B
```

是 Critical Path 上的任务，而：

```
Workflow 2 的 X
```

拥有较大的 Slack。

那么调度器可能选择：

```
优先 B
```

因为：

```
B 晚启动
→ 后面的 C 晚启动
→ Workflow 1 完成时间被直接推迟
```

而 X：

```
晚一点启动
→ 只要在 Slack 范围内完成
→ Workflow 2 最终完成时间不变
```

所以真正的逻辑是：

```
资源有限
     ↓
不能所有 Task 同时执行
     ↓
必须选择哪些 Task 获得资源
     ↓
Critical Task / Slack Task 的延迟影响不同
     ↓
优先保护低 Slack Task
```

------

# 12. 「关键路径先做」真正应该理解成什么

不要记成：

```
Critical Path = 优先执行最长任务
```

更准确的是：

```
Critical Path
      ↓
判断哪些 Task 的延迟会传递到 Workflow 完成时间
      ↓
Slack
      ↓
判断哪些 Task 可以暂时延后
      ↓
资源有限时
      ↓
优先保护 Slack 小 / Critical 的 Task
```

核心：

> **关键路径决定哪些延迟会传递到最终结果；Slack 决定哪些任务可以晚一点。**

------

# 13. 如果所有 Task 都能同时执行呢？

假设：

```
        A
       / \
      B   C
       \ /
        D
```

GPU 足够：

```
B ─────────→
C ───────────────→
```

B 和 C 完全并行。

那么：

```
B finish = 2
C finish = 10

D start = max(2,10)
        = 10
```

此时 Scheduler 先调度 B 还是 C：

```
没有区别
```

因为：

```
资源没有竞争
```

所以：

> **Critical Path 本身不会创造并行能力。**

它只是告诉 Scheduler：

```
如果资源不够，
应该保护哪些任务。
```

------

# 14. 用户最终完成时间 vs 中间 Task 完成时间

这是理解 Agent Scheduling 的另一个关键点。

假设：

```
A
├── B
└── C
     ↓
     D
```

如果：

```
B = 2s
C = 10s
```

那么：

```
B 在 2s 完成
C 在 10s 完成
D 在 10s 开始
```

如果 B 延迟：

```
B = 8s
C = 10s
```

那么：

```
D 还是 10s 开始
```

因此：

```
B 的完成时间变化
```

虽然发生了变化，但是：

```
Workflow 最终完成时间没有变化
```

这说明：

> **并不是每一个中间 Task 的延迟都会影响最终用户感知的 Workflow Latency。**

所以如果论文目标只有：

```
Workflow Completion Time
```

那么单纯优化某个非关键 Task 的完成时间没有价值。

------

# 15. 为什么论文还会关注中间 Task？

因为 Agent Workflow 不一定只有：

```
最终 Join
```

中间 Task 可能继续产生：

```
新的 Task
```

例如：

```
A
 ↓
B
 ↓
C
 ↓
D
```

或者动态生成：

```
A
├── B
├── C
│   └── E
└── D
```

这时候：

```
B 延迟
→ C 延迟
→ E 延迟
→ 后续任务延迟
```

所以 Task 的局部延迟可能逐渐传播。

这也是：

```
Workflow-aware Scheduling
```

比：

```
Request-level Scheduling
```

更有意义的原因之一。

------

# 16. Agent Workflow 与普通 DAG 的区别

普通 DAG：

```
A → B → C
```

通常任务集合和依赖关系提前知道。

Agent Workflow 更复杂：

```
User
 ↓
LLM
 ↓
决定调用 Tool A
 ↓
Tool A
 ↓
LLM
 ↓
决定：
 ├── Tool B
 ├── Tool C
 └── 不调用 Tool
```

因此 DAG 可能是：

```
动态生成的
```

即：

> **Online / Dynamic DAG**

Scheduler 一开始甚至不知道：

```
后面会产生什么 Task
```

只能随着 Agent 执行不断观察。

------

# 17. 这会产生一个新的调度问题

传统 DAG：

```
整个 DAG 已知
```

可以提前计算：

```
Critical Path
Slack
```

Agent：

```
DAG 不一定完整已知
```

可能变成：

```
Task A
 ↓
LLM
 ↓
动态决定
 ↓
Task B / C / D
 ↓
继续动态生成
```

所以：

```
Scheduling
+
Online DAG Revelation
```

结合起来。

这也是 Agent Scheduling 比传统 DAG Scheduling 更复杂的地方。

------

# 18. 论文 Related Work 的整体演进

可以把论文引用的工作理解成：

```
                    Agent Serving
                         │
                         ▼
             ┌─────────────────────┐
             │ Request Scheduling  │
             └─────────────────────┘
                         │
                         ▼
             ┌─────────────────────┐
             │ Call Scheduling     │
             └─────────────────────┘
                         │
                         ▼
             ┌─────────────────────┐
             │ Program-aware       │
             │ Scheduling          │
             └─────────────────────┘
                         │
                         ▼
             ┌─────────────────────┐
             │ Workflow / DAG      │
             │ Scheduling          │
             └─────────────────────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       Critical Path            Slack
              │                     │
              └──────────┬──────────┘
                         ▼
               Resource-aware
                  Scheduling
```

------

# 19. 异构资源进一步增加复杂度

实际 GPU 集群可能不是：

```
GPU 1 = GPU 2 = GPU 3
```

而是：

```
A100
H100
L40S
...
```

它们可能具有不同：

- Compute
- Memory
- Memory Bandwidth
- Interconnect
- KV Cache Capacity
- Prefill 性能
- Decode 性能

因此：

```
Task A → GPU 1
```

和：

```
Task A → GPU 2
```

执行时间可能不同。

于是调度问题变成：

```
哪个 Task
      +
哪个 GPU
      +
哪个时间
      +
Prefill / Decode
      +
Workflow Dependency
```

联合优化。

------

# 20. 最终形成的完整问题

Agentic LLM Serving 可以抽象成：

```
                User Request
                      ↓
                Agent Workflow
                      ↓
                     DAG
                      ↓
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
     Task A        Task B        Task C
        ↓             ↓             ↓
   Prefill/Decode Prefill/Decode Prefill/Decode
        ↓             ↓             ↓
      GPU 1         GPU 2         GPU 3
```

Scheduler 同时面对：

### Workflow 层

```
Dependency
Critical Path
Slack
Remaining Work
Dynamic DAG
```

### LLM Serving 层

```
Prefill
Decode
KV Cache
Batching
```

### Resource 层

```
GPU
Memory
Compute
Network
Heterogeneity
```

------

# 21. 为什么传统 SJF 不够？

SJF：

```
Shortest Job First
```

思想：

```
短任务优先
```

例如：

```
Task A = 2s
Task B = 10s
```

SJF：

```
A → B
```

对于普通独立任务可能有效。

但是 Agent Workflow 中：

```
        A
       / \
      B   C
       \ /
        D
```

B/C 的优先级不能只由：

```
duration
```

决定。

还要考虑：

```
它是否位于 Critical Path？
它还有多少 Slack？
它后面有多少依赖任务？
它是否会阻塞整个 Workflow？
```

所以：

```
Task Duration
```

只是一个维度。

------

# 22. 一个更合理的调度优先级理解

可以抽象成：

```
Priority(Task)
=
f(
    Criticality,
    Slack,
    Remaining Work,
    Deadline,
    Duration,
    Resource Requirement,
    Workflow Importance
)
```

其中：

### Criticality

```
是不是关键路径上的任务
```

### Slack

```
还能延迟多久
```

### Remaining Work

```
这个 Task 后面还有多少工作
```

### Duration

```
预计执行多久
```

### Resource Requirement

```
需要什么 GPU / 显存 / Prefill / Decode
```

------

# 23. 需要特别警惕的论文逻辑

以后读这类论文时，不要看到：

```
Critical Path
```

就直接认为：

> “关键路径优先，所以一定能降低 Workflow Completion Time。”

应该检查以下 5 件事。

### ① 论文到底优化什么？

可能是：

```
Makespan
Average Completion Time
Tail Latency
SLO Violation
Throughput
Weighted Completion Time
Fairness
```

不同目标可能导致完全不同的调度策略。

------

### ② 有没有资源竞争？

如果：

```
GPU 无限
```

那么调度顺序通常没什么意义。

必须存在：

```
有限 GPU
有限显存
有限 Prefill Slot
有限 Decode Slot
```

等资源约束。

------

### ③ Critical Path 是否真的改变了任务启动时间？

关键不是：

```
谁先完成
```

而是：

```
谁因为资源竞争而被迫等待
```

如果 Critical Task 本来就能立即执行：

```
Critical Path Priority
```

可能没有实际收益。

------

### ④ 非关键任务是否真的拥有 Slack？

不能简单：

```
非 Critical = 不重要
```

正确的是：

```
Slack 大
→ 可以延后

Slack 小
→ 不应该轻易延后
```

------

### ⑤ 实验到底证明了什么？

要看：

```
Baseline
vs
Proposed Scheduler
```

比较的是：

```
平均 Workflow Completion Time？
P95？
P99？
Throughput？
SLO？
```

而不是只看：

```
某些 Task 更早完成
```

------

# 24. 最核心的一个例子

记住这个 DAG：

```
             A
            / \
           B   C
           |   |
          2s  10s
            \ /
             D
             2s
```

假设：

```
A = 5s
B = 2s
C = 10s
D = 2s
```

则：

```
A → B → D = 9s
A → C → D = 17s
```

所以：

```
Critical Path = A → C → D
```

并且：

```
Slack(B) = 8s
Slack(C) = 0
```

因此：

```
C 延迟 1s
→ Workflow 延迟 1s

B 延迟 1s
→ Workflow 不一定延迟

B 延迟 8s
→ 仍可能不影响最终结果

B 延迟超过 8s
→ 开始影响 Workflow
```

这就是：

> **Critical Path + Slack 的核心直觉。**

------

# 25. 一句话理解整篇 Related Work

> **传统 LLM Serving 主要调度“请求”，Agent Serving 开始调度“由多个有依赖关系的 LLM/Tool Task 构成的 Workflow”，因此调度器需要从单纯考虑 Task Duration，进一步考虑 DAG Dependency、Critical Path、Slack、动态任务生成、Prefill/Decode 分离以及异构 GPU 资源。**

------

# 26. 当前对论文核心逻辑的判断

目前读到 Related Work 为止，可以得到：

```
传统：
Request → LLM → GPU

Agent：
Request
   ↓
Workflow
   ↓
DAG
   ↓
多个 Task
   ↓
竞争有限 GPU 资源
```

因此论文真正需要证明的不是：

> “Critical Path 很重要。”

而是：

> **在有限资源 + Agent Workflow + 动态 DAG 的情况下，利用 Workflow Dependency / Critical Path / Slack 进行调度，是否能够改善论文定义的目标指标。**

尤其要注意：

```
Critical Path ≠ 自动降低 Makespan
```

只有当：

```
资源存在竞争
+
调度决策能够改变关键任务的启动/完成时间
+
这种变化能够传递到目标指标
```

时，Critical-Path-aware Scheduling 才真正有意义。

**下一步看论文的 Problem Formulation / Objective Function 时，最应该抓这一条：**

```
它到底优化什么？
        ↓
什么资源限制导致必须做选择？
        ↓
Critical Path / Slack 如何改变这个选择？
        ↓
这个选择为什么会改善 Objective？
```

如果这四步中间有断层，那就是论文方法论里真正值得质疑的地方。