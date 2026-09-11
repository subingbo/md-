# K8s Dynamic Scheduling Agent：Evaluation 与 Benchmark 设计

## 1. 研究任务更新

研究对象是一个面向 Kubernetes 动态调度的 Agent，核心闭环为：

```text
K8s Environment
      ↓
Feedback
      ↓
State Update
      ↓
External Planner
      ↓
Dynamic Replanning
      ↓
新的调度 Action
```

Agent 的目标不是只生成一段调度计划，而是在真实或仿真的 Kubernetes 环境中执行 Action，根据执行结果和环境变化更新状态，并在必要时重新调用外部规划器。

本研究不强行加入 Reflexion。失败后的纠错可以由“反馈 → 状态/约束更新 → External Planner 重新求解”完成。只有当后续实验需要研究显式错误总结或可复用经验时，才把 Reflexion 作为额外变量。

## 2. ALFWorld 是什么

ALFWorld 是用于评估 Agent 规划与交互能力的文字环境。例如，Agent 需要完成“把加热后的苹果放到桌上”这类多步任务：

```text
观察环境 → 执行 Action → 获得 Observation → 再决定下一步
```

它的常见评价方式是任务成功率或累计 Reward。ALFWorld 对 K8s 研究的启发不是必须直接使用这个数据集，而是采用相同的实验思想：

- 给 Agent 一个明确任务；
- 让 Agent 与环境交互，而不是只输出计划文本；
- 让环境返回成功、失败或状态变化；
- 观察 Agent 是否能继续执行、重新规划并完成任务。

对应关系如下：

| ALFWorld | K8s Agent |
|---|---|
| Text-based environment | Kubernetes Cluster 或仿真集群 |
| PICKUP / OPEN / PUT | schedule / migrate / scale / evict |
| Observation | 调度结果、事件、资源状态、故障信息 |
| Task success | Pod/Workload 达到目标状态并满足约束 |
| Reward / Success Rate | 调度成功率、SLA、资源利用率等 |

## 3. Evaluation 与 Benchmark 设计

### 3.1 评估对象

将每个调度任务表示为：

```text
Initial K8s State + User Goal + Constraints
                    ↓
                 Agent
                    ↓
                 Actions
                    ↓
              K8s Feedback
                    ↓
              Final Outcome
```

任务可以覆盖以下场景：

1. **静态初始调度**：在给定节点、Pod 和资源约束下完成部署。
2. **资源动态变化**：CPU/Memory 负载变化，导致原方案不再合适。
3. **Workload 动态变化**：Pod 到达、删除、扩缩容。
4. **节点或执行故障**：节点不可用、taint/affinity 不匹配、调度执行失败。
5. **SLA 驱动调度**：在资源约束下满足延迟、可用性或副本数目标。

每个场景应固定初始状态、任务目标、约束、扰动类型和随机种子，并重复多次运行，以便公平比较。

### 3.2 核心机制

```text
读取 K8s State
      ↓
LLM 理解任务与约束
      ↓
External Planner 生成可行调度方案
      ↓
执行调度
      ↓
收集 K8s Feedback
      ↓
更新 State / Constraint
      ↓
判断是否需要 Dynamic Replanning
```

这里的 Replanning 触发条件可以包括：执行失败、关键资源变化、节点故障、SLA 违约风险或原方案不可行。

## 4. Baseline、Metric、Result、Ablation

### 4.1 Baseline：和谁比较

Baseline 是用于参照的已有方法或删减版系统，不是某个指标的数值。建议至少包含：

| 方法 | 作用 |
|---|---|
| Kubernetes Native Scheduler | 工程基线，代表默认调度能力 |
| Heuristic Scheduler | 如 First-Fit、Bin Packing 或 Least-Requested |
| LLM Direct Agent | LLM 直接决定调度，不使用 External Planner |
| LLM + External Planner | 有外部规划器，但失败后不重新规划 |
| **Ours** | LLM + External Planner + Feedback-driven Dynamic Replanning |

是否加入 ReAct 或 Reflexion，应根据研究问题决定，不必为了复现论文术语而强行加入。若加入，应清楚说明它们是额外对照，而不是你的核心架构。

### 4.2 Metric：比较什么

建议分成五类：

| 类别 | 指标 | 含义 |
|---|---|---|
| 任务完成 | Scheduling Success Rate、Task Completion Rate | 是否完成目标调度任务 |
| 动态适应 | Recovery Rate、Recovery Time、Replanning Success Rate | 扰动或失败后能否恢复 |
| 服务质量 | SLA/SLO Violation Rate、Latency、Availability | 是否满足业务约束 |
| 资源与操作代价 | CPU/Memory Utilization、Migration Count、Rescheduling Count | 调度方案的资源效率与扰动成本 |
| Agent 成本 | Planning Latency、LLM Calls、Token Cost、Planner Runtime | Agent 带来的时间和计算开销 |

核心指标可以先收敛为：

```text
Success Rate
Recovery Rate
SLA Violation Rate
Resource Utilization
Recovery Time
Planning Cost
```

### 4.3 Result：实际测得多少

Result 是实验运行后得到的具体数值。Baseline 和 Metric 组合后才形成结果表，例如：

| 方法 | Success Rate | Recovery Rate | SLA Violation | Recovery Time | Planning Cost |
|---|---:|---:|---:|---:|---:|
| K8s Native Scheduler | 待测 | 待测 | 待测 | 待测 | 待测 |
| Heuristic Scheduler | 待测 | 待测 | 待测 | 待测 | 待测 |
| LLM Direct Agent | 待测 | 待测 | 待测 | 待测 | 待测 |
| LLM + External Planner | 待测 | 待测 | 待测 | 待测 | 待测 |
| **Ours** | **待测** | **待测** | **待测** | **待测** | **待测** |

不要预先把示例中的 80%、90% 等数字写成实验结论；正式论文中应报告多次运行的均值、标准差或置信区间。

### 4.4 Ablation：哪个模块起作用

Ablation 不是和外部方法比较，而是从自己的系统中逐个移除组件：

| 版本 | 移除内容 | 要验证的问题 |
|---|---|---|
| Ours | 无 | 完整系统表现 |
| Ours − Dynamic Replanning | 失败后不重新求解 | Replanning 是否提升动态恢复能力 |
| Ours − Feedback-driven State Update | 不使用最新反馈更新状态 | 状态更新是否是关键 |
| Ours − External Planner | LLM 直接输出调度 | 外部规划器是否提升可行性与稳定性 |
| Ours − Constraint Update | 只刷新资源，不更新失败约束 | 约束更新是否减少重复失败 |

推荐的最小消融实验是：

```text
完整系统
vs.
去掉 Dynamic Replanning
vs.
去掉 External Planner
vs.
去掉 Feedback-driven State Update
```

Reflexion 不作为必选消融项。只有在系统确实实现了“显式分析失败原因并形成可复用经验”后，才可以增加：

```text
Ours + Explicit Reflexion
```

用来研究显式反思相对于状态/约束更新是否还有额外收益。

## 5. 推荐实验矩阵

### 实验一：静态调度能力

- 固定集群状态和工作负载。
- 比较所有 Baseline 与 Ours。
- 主要看 Success Rate、SLA、资源利用率和调度延迟。

### 实验二：动态变化适应能力

- 在执行过程中注入 Pod 到达、负载变化或节点故障。
- 比较“无 Replanning”和“有 Dynamic Replanning”。
- 主要看 Recovery Rate、Recovery Time、SLA Violation Rate 和 Replanning 次数。

### 实验三：组件消融

- 固定任务集合和扰动集合。
- 逐个移除 External Planner、Feedback-driven State Update、Dynamic Replanning。
- 说明每个模块对性能的独立贡献。

### 实验四：成本—收益分析

- 记录 LLM 调用次数、Token、Planner 求解时间和总调度延迟。
- 与成功率、恢复率、SLA 改善一起报告。
- 回答“性能提升是否值得额外 Agent 成本”这一问题。

## 6. 论文中可以采用的核心表述

> We evaluate the K8s Agent as an interactive planning system rather than a one-shot plan generator. The Kubernetes environment returns execution feedback and dynamic state changes, which are used to update the planning state and trigger the external planner for dynamic replanning. Our evaluation compares the proposed method with native, heuristic, direct-LLM, and non-replanning baselines, and uses ablation studies to quantify the contribution of feedback-driven state updates, external planning, and dynamic replanning. Reflexion is not assumed as a required component; explicit reflection is treated as an optional extension rather than the core mechanism.

## 7. 一句话总结

```text
Baseline：和谁比？
Metric：比较什么？
Result：实际是多少？
Ablation：我的哪个模块有效？
```

对当前 K8s Agent，最重要的实验问题是：

> 当 Kubernetes 环境发生变化或调度失败时，Feedback-driven State Update + External Planner + Dynamic Replanning 是否能以可接受的成本提高任务完成率和恢复能力？
