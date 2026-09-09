# LLM-Agent Planning：核心方法与 Kubernetes 调度映射

## 1. 总览

LLM-agent planning 的常见增强方向，主要是在解决不同问题：

- **External Planner-Aided Planning**：提高计划的可行性、搜索能力和求解效率。
- **Reflection and Refinement**：检查并修正 LLM 初次生成的计划。
- **Memory-augmented Planning**：利用常识、历史经验和领域知识。
- **Task Decomposition**：将复杂任务拆成更容易处理的子任务。

这些方向不是互斥选项，而是可以组合使用。例如：先检索历史经验，再进行任务分解；LLM 生成初始计划后，用 External Planner 验证，并通过 Reflection 继续修正。

> 论文提到的五个研究方向是相互关联的，实际系统通常会同时采用多种技术。本笔记重点整理与当前研究问题最相关的内容。

---

## 2. External Planner-Aided Planning

### 核心思想

LLM 不负责从头到尾求解计划，而主要负责理解自然语言任务并将其转换成形式化表示；真正的计划由外部规划器计算。

$$
h = formalize(E, g; \Theta, P)
$$

$$
p = plan(E, g, h; \Phi)
$$

### 符号含义

| 符号 | 含义 |
|---|---|
| $E$ | Environment，当前环境或状态 |
| $g$ | Goal，当前目标 |
| $\Theta$ | LLM 的模型参数，即具体 LLM 的参数化模型 |
| $P$ | Prompt，用于指导 LLM 的提示信息 |
| $h$ | Formalized information，形式化后的任务信息 |
| $\Phi$ | External planner module，外部规划器模块本身，不是普通意义上的参数 |
| $p$ | 最终生成的计划 |

### 流程

```text
自然语言任务 + 环境 E + 目标 g
                ↓
        LLM(Θ, Prompt P)
                ↓
     形式化任务信息 h
                ↓
      External Planner Φ
                ↓
          最终计划 p
```

外部规划器可以是 PDDL Planner、搜索算法、CP-SAT、MILP 或其他求解模块。它擅长进行约束检查、状态搜索和可行性判断。

### 优势与局限

- **优势**：计划更容易满足明确约束，搜索和求解能力更强；适合规则多、约束强的任务。
- **局限**：LLM 生成的形式化信息 $h$ 如果错误，外部规划器可能是在错误的问题上求解；还需要额外的规划器和接口设计。

---

## 3. Reflection and Refinement

### 核心思想

LLM 先生成一个初始计划，再反思计划中的问题，并根据反思结果进行修改。

$$
p_0 = plan(E, g; \Theta, P)
$$

$$
r_i = reflect(E, g, p_i; \Theta, P)
$$

$$
p_{i+1} = refine(E, g, p_i, r_i; \Theta, P)
$$

### 三个公式分别表示

- $p_0$：第 0 版，即初始计划。
- $r_i$：对第 $i$ 版计划 $p_i$ 的反思结果，例如发现违反约束、步骤不可执行或遗漏了任务。
- $p_{i+1}$：根据 $r_i$ 修改后的下一版计划。

### 流程

```text
p0：生成初始计划
 ↓
ri：检查当前计划并找出问题
 ↓
pi+1：根据问题修改计划
 ↓
继续反思和修改，直到满足停止条件
```

### 优势与局限

- **优势**：实现简单，不必额外引入规划器；适合通过多轮检查逐步改善结果。
- **局限**：反思者和计划生成者通常都是 LLM，可能出现“错误计划 → 错误反思 → 继续错误”的情况，缺少严格的可行性保证。

---

## 4. Memory-augmented Planning

### 核心思想

LLM 规划前，先从记忆模块中检索与当前环境和目标相关的信息，用作辅助信号。

$$
m = retrieve(E, g; M)
$$

$$
p = plan(E, g, m; \Theta, P)
$$

### `M` 和 `m` 的区别

- **$M$**：完整的 Memory module，即整个记忆库，可能包含常识、历史任务、用户偏好、领域知识和成功/失败经验。
- **$m$**：本次任务从 $M$ 中检索出来的相关信息。

```text
M：整个记忆库
    ↓ retrieve(E, g; M)
m：本次任务真正使用的相关记忆
    ↓
LLM 生成计划 p
```

### 优势与局限

- **优势**：不必每次从零开始；可以复用相似任务经验，减少重复错误。
- **局限**：检索质量会直接影响规划质量；过时、错误或不相关的记忆可能误导 LLM。

---

## 5. Task Decomposition

复杂任务可以先拆成多个子任务，再分别进行子任务规划。常见的两种方式是 **Decomposition-First** 和 **Interleaved**。

### 5.1 Decomposition-First

```text
Goal
 ↓
一次性拆出 Sub-goal 1、2、3、...
 ↓
分别规划每个子任务
```

也就是：

> **先把任务结构基本确定，再规划各个子任务。**

适合子任务之间相对独立、环境稳定、依赖关系清晰的情况。优点是结构清楚、容易并行；缺点是早期信息不足时，可能把后续任务拆错，之后需要大量重规划。

### 5.2 Interleaved

```text
拆 Sub-task 1
 ↓
规划 Sub-task 1
 ↓
根据已有信息决定如何拆 Sub-task 2
 ↓
规划 Sub-task 2
 ↓
继续决定后续任务
```

更准确地说，Interleaved 是：

> **任务分解和子任务规划交替进行，而不是先把所有任务拆完再统一规划。**

这里的“交替”不一定意味着同时执行，也不等于“边规划边执行”。在 Task Decomposition 语境中，它主要强调 **Decomposition 与 Sub-planning 的交错**。

### 为什么前面的结果会影响后面？

前面子任务的规划结果，可能暴露新的约束、资源限制或依赖关系，从而改变：

1. 后续任务如何执行；
2. 后续任务如何规划；
3. 后续任务应该如何进一步分解。

例如，规划“确定场地”后发现容量有限，后续的嘉宾筛选、住宿安排和设备准备都需要改变。此时后续子任务并不是一开始就完全确定的。

### 一句话区分

- **Decomposition-First**：先把路线图全部画出来，再走。
- **Interleaved**：先决定下一小段怎么走，获得新信息后，再决定后面怎么走。

---

## 6. 三种 planning 增强方法对比

| 方法 | 主要解决的问题 | 核心机制 | 优势 | 局限 |
|---|---|---|---|---|
| External Planner | 可行性、约束满足、搜索效率 | LLM 形式化，外部模块求解 | 约束处理严格，搜索能力强 | 形式化可能出错，需要额外模块 |
| Reflection & Refinement | 初始计划质量和自我纠错 | 生成 → 反思 → 修改 | 简单、灵活、易接入 Agent | LLM 可能错误地检查自己 |
| Memory-augmented Planning | 经验复用和知识利用 | 检索记忆 → 辅助规划 | 减少重复推理和错误 | 依赖记忆库与检索质量 |

可以把它们记成三个问题：

```text
External Planner：这个计划是否可行？
Reflection：我刚才的计划哪里有问题？
Memory：以前遇到类似问题时是怎么做的？
```

---

## 7. 与 LLM + Kubernetes 动态调度的关系

### 7.1 三种架构的区别

| 架构 | LLM 的角色 | 谁负责最终 Pod placement |
|---|---|---|
| LLM 直接调度 | 直接输出“Pod → Node”，再调用 Kubernetes API | LLM/自定义 Agent |
| LLM + Scheduler Plugin | 根据集群状态调整策略或权重，影响 Filter/Score | K8s Scheduler |
| LLM + Custom Scheduler | LLM 参与或指导自己实现的调度器 | Custom Scheduler |

#### LLM 直接调度

```text
Pod / Cluster State
        ↓
       LLM
        ↓
“Pod A → Node B”
        ↓
 Kubernetes API
```

优点是决策自由度高；但需要自己处理资源约束、并发竞争、拓扑、亲和性、抢占、速度和可靠性，实际上接近于重新实现一个 Scheduler。

#### LLM + Scheduler Plugin

```text
集群状态与指标
        ↓
       LLM
        ↓
调整调度策略/权重
        ↓
K8s Scheduler Plugin
        ↓
Filter / Score / Bind
        ↓
具体 Pod → Node
```

LLM 可以输出类似：

```text
CPU balance       = 0.2
Memory balance    = 0.1
Network locality  = 0.7
```

然后由 K8s Scheduler 结合真实资源和约束，计算具体节点。

#### LLM + Custom Scheduler

不修改或少依赖默认 `kube-scheduler`，而是自己实现调度器。它可以使用规则、优化算法、LLM 或其他模块，但需要自行承担更多 Kubernetes 调度基础设施的责任。

### 7.2 当前更合理的研究思路

当前关注的不是让 LLM 为每一个 Pod 直接选择 Node，而是：

> **LLM 作为高层策略控制器，根据动态集群状态调整调度策略；Kubernetes Scheduler 负责具体的 Pod placement。**

整体可以表示为：

```text
动态集群状态
        ↓
       LLM
        ↓
高层策略/权重调整
        ↓
K8s Scheduler
        ↓
具体 Pod placement
```

这是一种分层决策架构：

$$
S_{cluster} \rightarrow \pi_{scheduling}
$$

LLM 根据集群状态 $S_{cluster}$ 选择或调整调度策略 $\pi_{scheduling}$；随后由 Scheduler 完成：

$$
\pi_{scheduling} + Pod + Nodes \rightarrow Node
$$

### 7.3 这种设计的意义

- LLM 只处理“当前应该更重视什么”，不必搜索所有 Pod-Node 组合。
- 调度频率可以较低，例如每几十秒或每分钟调整一次策略，而不是每个 Pod 都调用 LLM。
- K8s Scheduler 继续负责确定性的约束检查和节点选择，降低 LLM 输出错误的影响范围。
- 模型可以更小，研究重点从“LLM 能否替代 Scheduler”转为“动态策略是否比固定策略更有效”。

### 7.4 是否需要 External Planner？

在“LLM 调整策略 + K8s Scheduler 负责 placement”的方案中，**不一定需要 External Planner**。K8s Scheduler 本身已经承担了大量确定性的过滤、打分和绑定工作。

只有当研究目标变成：在更长时间窗口内对多个工作负载、资源和约束进行全局组合优化时，才可以额外引入 CP-SAT、MILP 或其他 Planner/Solver。

---

## 8. 最终记忆点

1. **External Planner**：LLM 负责形式化，外部 Planner 负责求解。
2. **Reflection**：先生成，再检查，再修改。
3. **Memory**：$M$ 是整个记忆库，$m$ 是本次检索结果。
4. **Decomposition-First**：先拆完，再规划。
5. **Interleaved**：边分解边规划，不等于边规划边执行。
6. 前面子任务得到的新信息，可能改变后续任务的执行、规划甚至分解方式。
7. 当前 Kubernetes 方向适合采用：**LLM 做高层策略控制，K8s Scheduler 做底层 Pod placement**。
