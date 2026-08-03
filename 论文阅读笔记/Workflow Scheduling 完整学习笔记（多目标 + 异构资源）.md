~~~markdown
# 🧠 Workflow Scheduling 完整学习笔记（多目标 + 异构资源）

---

# 📌 一、问题定义

## 1. Workflow（工作流）

```text
Workflow = DAG（有向无环图）
G = (A, E)

A：任务集合（tasks）
E：依赖关系（edges）
~~~

------

## 2. 调度问题本质

```text
输入：
  DAG + 资源 + 参数

输出：
  每个任务：
    - 分配到哪个机器
    - 什么时候执行

目标：
  优化某些指标
```

------

# 📊 二、核心指标（必须掌握）

## 1. Makespan（总完成时间）

```text
Makespan = max(task_finish_time)
```

👉 最后一个任务完成的时间

------

## 2. Cost（成本）

```text
Cost = Σ (execution_time × price)
```

------

## 3. Energy（能耗）

```text
E_total = E_compute + E_data

E_compute = power × time
E_data = data_size × energy_per_byte
```

------

## 4. Turnaround Time

```text
Turnaround = completion_time - submission_time
```

------

## 5. Throughput

```text
Throughput = |tasks| / makespan
```

------

# 🧩 三、系统建模（非常关键）

------

## 1. 任务参数

```python
comp_cost[t][m]   # 任务 t 在机器 m 上的执行时间
```

------

## 2. 通信参数

```python
data[i][j]                # 数据量
bandwidth[m1][m2]         # 带宽
communication_time = data / bandwidth
```

------

## 3. 资源参数（异构）

```python
price[m]
power[m]
```

------

## 4. 调度状态

```python
machine_available_time[m]
task_finish_time[t]
```

------

# ⚙️ 四、HEFT 算法（核心基础）

------

## 1. Rank 计算（优先级）

```text
rank_u(t) = avg_comp_cost(t) +
            max(successor rank + avg_comm)
```

------

## 2. EST（最早开始时间）

```text
EST(t, m) =
    max(
        machine_available_time[m],
        max(parent_finish + communication)
    )
```

------

## 3. FT（完成时间）

```text
FT = EST + execution_time
```

------

## 4. 调度流程

```python
for task in sorted_by_rank:
    for machine in machines:
        est = compute_EST(task, machine)
        ft = est + exec_time

    select machine with min FT

    assign task
    update machine_available_time
```

------

# 💰 五、Cost & Energy 模型

------

## 1. Cost

```text
C(task, m) = exec_time × price[m]
```

------

## 2. Energy

```text
E_compute = power[m] × exec_time

E_data = data[i][j] × energy_per_byte
```

------

# 🎯 六、多目标调度（核心提升）

------

## 1. 加权模型（推荐）

```text
Score = α·Time + β·Cost + γ·Energy
```

------

## ❗ 归一化（必须）

```text
norm(x) = (x - min) / (max - min)
```

------

## 2. 多目标决策流程

```python
for machine in machines:
    compute FT
    compute cost
    compute energy

    normalize

    score = weighted sum

select min score
```

------

# 🚀 七、优化策略（重点）

------

## 1. 关键路径优化

```text
Critical Path → 优先分配最快机器
```

------

## 2. 通信优化

```text
尽量让依赖任务在同一机器
```

------

## 3. 异构资源调度

```text
不同机器性能不同：

CPU强 → 计算任务
IO强 → 数据任务
```

------

## 4. 局部优化（Local Search）

```text
尝试：
  迁移任务
  交换任务
```

------

## 5. 动态权重（进阶）

```text
前期重时间
后期重成本/能耗
```

------

# 📊 八、实验设计（必须做）

------

## 1. 对比算法

```text
HEFT（baseline）
vs
你的算法
```

------

## 2. 指标

```text
Makespan
Cost
Energy
```

------

## 3. 实验变量

```text
任务数量
机器数量
数据量
带宽
```

------

## 4. 数据集

```text
随机生成 DAG
```

------

# 🧪 九、示例代码框架

------

```python
for task in tasks:

    best_machine = None
    best_score = inf

    for m in machines:

        est = compute_EST(task, m)
        ft = est + exec_time

        cost = ft * price[m]
        energy = power[m] * exec_time

        score = α*ft + β*cost + γ*energy

        if score < best_score:
            best_score = score
            best_machine = m

    assign(task, best_machine)
```

------

# 🧠 十、进阶方向

------

## 1. Pareto 优化

```text
不只一个最优解
```

------

## 2. 生物算法

```text
GA / PSO / ACO
```

------

## 3. 动态调度

```text
任务动态到达
```

------

## 4. 强化学习

```text
用 RL 做调度
```

------

# 🧱 十一、项目结构建议

```text
scheduler/
 ├── dag.py
 ├── heft.py
 ├── cost.py
 ├── energy.py
 ├── scheduler.py
 ├── optimizer.py
 └── experiment.py
```

------

# 📌 十二、学习路线总结

```text
Step1：DAG + 拓扑排序
Step2：HEFT
Step3：Cost / Energy
Step4：多目标调度
Step5：优化策略
Step6：实验验证
```

------

# 🧾 总结一句话

> Workflow 调度的核心是：在 DAG 约束下，将任务映射到异构资源上，并在时间、成本、能耗等多目标之间做权衡优化。