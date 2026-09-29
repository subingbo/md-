# Speculate with Memory：实现思路与 PDDL 实现报告

> 对应论文：[arXiv:2607.12236](https://arxiv.org/abs/2607.12236)  
> 本地复现仓库：`D:\code\speculate_with_memory`  
> 撰写目的：说明论文方法如何落地，以及 PDDL（Type 2）在论文中与本仓库中的实现路径

---

## 1. 论文要解决什么

LLM Agent 每一步都要串行等待两段延迟：

1. **Actor LLM 推理延迟** \(\ell_{LLM}\)
2. **环境反馈延迟** \(\ell_{env}\)（API、页面加载、机器人执行、用户回复等）

**投机执行（Speculative Execution）**用一个更小、更快的 Speculator，在“空闲窗口”里提前预测下一步，并预启动可安全并行的工作。预测对了就省延迟；错了就丢弃预启动结果。关键约束是 **lossless**：Actor 的轨迹必须与非投机执行完全一致，不能因为猜错而改写环境或 Actor 上下文。

论文指出：现有投机器大多是 **stateless**——每条任务独立，处理过 500 次客服交互的系统与刚上线的系统预测能力相同。然而 Agent 任务有强规律（查用户 → 查订单；走到架子 → 拿起杯子）。因此他们给 Speculator 加上 **三套在线记忆**，让命中率随经验上升。

---

## 2. 总体架构（实现视角）

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent 主循环（Actor）                      │
│  state s_t ──LLM──► action a_t ──env──► observation o_t      │
└───────────────┬───────────────────────────┬─────────────────┘
                │ 空闲窗口并行                │ 任务结束后在线更新
                ▼                             ▼
┌──────────────────────────┐     ┌────────────────────────────┐
│ Speculator（小模型）       │     │ Memory Systems              │
│ + 记忆注入的 prompt        │◄────│ 1. Contrastive Transition   │
│ 预测 â / ô / (ô, â)       │     │ 2. Episodic (+ Miss)        │
│ 预启动只读工作             │     │ 3. Confusion Tracker        │
└──────────────────────────┘     └────────────────────────────┘
```

本仓库采用论文主实验同款的 **Replay 协议**：

1. 用 Actor **只采集一次**轨迹并落盘；
2. 七档记忆消融都对着 **同一批轨迹** 回放；
3. 投机结果 **不写回** Actor 上下文；
4. 记忆只在 **整条任务结束后** 提交，避免同一任务内泄漏。

对应代码：

| 模块 | 路径 | 职责 |
|------|------|------|
| 类型与七档开关 | `src/swm/types.py` | `Trajectory` / `Action` / `MemorySetting` |
| 转移表 | `src/swm/memory/transition.py` | Eq.1–2 统计 |
| 情景记忆 | `src/swm/memory/episodic.py` | 检索、miss、arg_sources |
| 混淆器 | `src/swm/memory/confusion.py` | ≥3 次硬约束 |
| 记忆组装 | `src/swm/memory/stack.py` | 任务缓冲 + commit |
| 提示词 | `src/swm/prompts.py` | Appendix C 序列化 |
| 回放评测 | `src/swm/replay.py` | step0 排除、聚合准确率 |
| 基准适配 | `src/swm/benchmarks/*` | ALFWorld / HotpotQA / PDDL |

---

## 3. 三种投机类型（决定“猜什么、省什么”）

论文把延迟落在 Agent–环境环路的不同位置，归纳为三类：

### Type 1：动作预测（Action Prediction）

- **时机**：与 Actor 并行，预测下一个动作 \(\hat{a}_{t+1}\)
- **命中收益**：动作已预启动，最多省 \(\ell_{env}\)；实际约 \(\min(\ell_{env}, \ell_{LLM}-\ell_{spec})\)
- **安全**：只预启动只读动作（查询、导航、读库）；写操作进白名单外则禁止
- **指标**：只读动作 **type + 全部参数** 精确匹配
- **论文基准**：WebArena、VisualWebArena（本复现 **不做**）

### Type 2：观测预测（Observation Prediction）

- **时机**：环境处理已提交动作期间，预测 \(\hat{o}_t\)
- **命中收益**：可带着预测观测预启动下一次 Actor 调用，最多省 \(\ell_{LLM}\)
- **安全**：预启动的是 Actor LLM 调用，不改环境，天然安全
- **指标**：**观测精确匹配**（空白归一化后比较）
- **论文基准**：ALFWorld、**PDDL**
- **特点**：动作空间结构化、序列高度重复时，记忆增益最大（ALFWorld Full ≈ Stateless × 2.5）

### Type 3：链式预测（Chained Prediction）

- **时机**：环境处理期间，先预测观测再预测下一动作
- **命中收益**：最多 \(\ell_{LLM}+\ell_{env}\)
- **主指标**：动作全匹配（决定能否预启动）
- **论文基准**：τ²-bench、HotpotQA（本复现做 HotpotQA）

实现上，`SpeculationType` 驱动：

- 提示词模板（`build_speculator_messages`）
- 命中判定（`is_hit`）
- 转移表是“按当前动作裁剪”还是 `get_full_context()` 全表（Type 2 用全表）

---

## 4. 三套记忆系统（实现细节）

### 4.1 Contrastive Transition Table（对比转移表）

**捕捉什么**：无条件的 \(P(\text{next\_action\_type} \mid \text{current\_action\_type})\)，以及成功/失败对比信号。

**更新规则（论文 Eq.1）**：任务 \(\tau\) 完成后，对相邻动作对 \((a_i, a_{i+1})\)：

- 成功任务：\(n^+_{ij} += 1\)
- 失败任务：\(n^-_{ij} += 1\)
- `ArgSig(a_i→a_j)` 累计后继动作的参数名集合（如 `order_id,reason`）

**查询（Eq.2）**：

\[
\mathrm{Confidence}(a_i\to a_j)=\frac{n^+_{ij}+n^-_{ij}}{\sum_k(n^+_{ik}+n^-_{ik})}
\quad
\mathrm{SuccessRate}=\frac{n^+_{ij}}{n^+_{ij}+n^-_{ij}}
\]

**注入 prompt（Appendix C）**：按当前动作列出“最可能下一工具 + 成功率 + typical args”，并标出高失败率转移。

**Type 2 特例**：ALFWorld / PDDL 这类 turn-level 观测投机，论文用 `get_full_context()` 输出 **整张表**，不按单个当前动作裁剪。本仓库在 `MemoryStack.build_context` 里对 `OBSERVATION` 调用 `format_transition_full`。

**局限**：表是无条件的，不知道对话上下文和具体参数值 → 需要情景记忆补足。

### 4.2 Episodic Memory（情景记忆）

**捕捉什么**：上下文相关的完整片段，用作 in-context examples。

每条 episode（论文 Eq.3）：

| 字段 | 含义 | 本仓库实现 |
|------|------|-----------|
| `context` | 当前情境自然语言摘要 | `state_summary` / 对话摘要 |
| `actions` | 带完整参数的动作列表 | `Action(type, args)` |
| `observation` | 环境响应 | 参与 **embedding 索引**，**不**写入 prompt |
| `arg_sources` | 参数值来源 | 对先前观测做 **子串匹配** |
| `outcome` | 任务成败 | `Trajectory.success` |
| `lesson` | 经验教训 | **规则模板**生成，不另调 LLM |

**检索**：`text-embedding-3-small`（1536d）+ 余弦相似度 top-k。本仓库默认 `SWM_EPISODIC_TOP_K=3`；无 API 时用确定性 `HashEmbedder` 跑通协议。

**Miss episodes**：投机失败时额外存“预测 vs 实际”，再检索注入。论文在 ALFWorld / PDDL 上显示 **miss 贡献最大单档增益**（例如 Episodic→Episodic+Miss：ALFWorld +14.8pp，PDDL +14.0pp）。

**关键工程约束**：字段全部启发式构造，记忆更新 **零额外 LLM 延迟**；检索时唯一 API 是 embedding（&lt;50ms），可藏在空闲窗口里。

### 4.3 Confusion Tracker（混淆追踪器）

**捕捉什么**：反复出现的“猜成 A、实际是 B”。

- 每次验证后记 `(predicted_type, actual_type)`
- 同一模式出现 **≥ 3 次** → 注入硬约束：`Do NOT predict A in this context`
- 轻量字典，无需 embedding

**本仓库 Type 2 适配**：论文混淆示例是动作类型；ALFWorld/PDDL 的预测目标是观测，因此用 **观测首句归一化模板**（数字塌缩为 `<N>`）作为 type，避免稀疏。

---

## 5. 七档消融如何组装

论文设定：除 Stateless 外，后续档都 **含 Confusion 作为底座**。

| Setting | confusion | table | episodic | miss |
|---------|-----------|-------|----------|------|
| Stateless | | | | |
| Confusion | ✓ | | | |
| Table | ✓ | ✓ | | |
| Episodic | ✓ | | ✓ | |
| Table+Episodic | ✓ | ✓ | ✓ | |
| Episodic+Miss | ✓ | | ✓ | ✓ |
| Full | ✓ | ✓ | ✓ | ✓ |

对应 `SETTING_FLAGS`（`types.py`）与 `MemoryStack`（`stack.py`）。

**提交时序（防泄漏）**：

```
begin_task(traj)
  for each step:
      build_context()   # 只读已提交记忆
      speculate()
      record_verdict()  # 先缓冲，不立刻写入全局表/库
  commit_task()         # 整任务结束后才更新三套记忆
```

---

## 6. 回放评测协议（本仓库核心）

1. **Actor 轨迹一次性采集** → `data/trajectories/*.jsonl`
2. **逐步回放**：Speculator 只看到当前步之前的对话/状态 + 记忆块
3. **step_idx == 0 不计入分母**（论文：投机发生在每任务第一步之后；Figure 8 的 depth 0 是“第一个被投机的步”）
4. **精确匹配**：观测空白归一；动作 type+args 全等
5. **聚合准确率**：全任务全部计入步数上，正确步 / 总步
6. **best-of-k**：同一 prompt 抽 k 次，任一命中即算命中（论文主表报 k=1）

脚本：

```bash
python scripts/collect_actor.py --benchmark <name> --limit N [--synthetic]
python scripts/replay_eval.py --trajectories ... [--settings all] [--k 1]
```

---

## 7. 论文中的 PDDL：定位与结果

### 7.1 在实验矩阵中的位置

| 项 | 内容 |
|----|------|
| Speculation Type | **Type 2：Observation Prediction** |
| 来源 | IPC / AgentBoard 风格网格与结构化谜题 |
| 规模 | **60 tasks**（论文 Table 1） |
| Win rate（Actor） | 83.3% |
| Action space | Grid commands / 规划算子 |
| Pattern type | **State transitions**（状态转移规律强） |
| Metric | **Obs. match**（观测精确匹配） |
| 域分组（Figure 7） | Barman、Blockworld、Gripper、**Tyreworld** |

### 7.2 为什么 PDDL 适合记忆增强投机

1. 环境是 **离散、确定性、可文本化** 的状态转移：动作 → 观测几乎是模板化的。
2. 跨任务共享算子（`move` / `pick` / `stack` …），转移表能积累 \(a_i\to a_j\)。
3. 规划错误往往 **系统性**：Speculator 反复在同类状态猜错 → miss episodes / confusion 收益大。
4. 论文 Table 2（k=1，GPT-4.1-mini）：

| Setting | PDDL Acc | ∆ vs Stateless |
|---------|----------|----------------|
| Stateless | 17.6% | — |
| Confusion | 19.0% | +1.4 |
| Table | 21.1% | +3.5 |
| Episodic | 19.0% | +1.4 |
| Table+Episodic | 20.0% | +2.4 |
| **Episodic+Miss** | **33.0%** | **+15.4** |
| **Full** | **33.8%** | **+16.2** |

解读：

- PDDL 上 **失败学习（miss）是主导因子**（Episodic→Epi+Miss +14.0pp，全基准最大单档增益之一）。
- Full 略高于 Epi+Miss（33.8 vs 33.0），转移表有小幅正向贡献。
- 相对提升约 **1.9×** Stateless，仅次于 ALFWorld 的 2.5×。

按域：Tyreworld 记忆增益极大（+33.7pp），Blockworld 更温和（+6.4pp），因状态转移更可变。

### 7.3 PDDL 上的 Type 2 时间线

```
Actor 选出 action a_t
        │
        ▼
环境执行 a_t（仿真器 / 机器人）──── 空闲窗口 ────► Speculator 预测 ô_t
        │                                              │
        ▼                                              ▼
真实观测 o_t                                      用 ô_t 预启动下一次 Actor LLM
        │
        ├─ ô_t == o_t  → 命中，直接用预计算结果
        └─ 否则        → 丢弃，按真实 o_t 正常调用 Actor
```

预启动对象是 **下一次 Actor 调用**（用预测观测当输入），不改 PDDL 世界状态 → lossless。

---

## 8. 本仓库的 PDDL 实现现状

文件：[`src/swm/benchmarks/pddl_bench.py`](src/swm/benchmarks/pddl_bench.py)

### 8.1 已实现

1. **`PDDLAdapter`**：实现统一 `BenchmarkAdapter` 接口  
   - `name = "pddl"`  
   - `DOMAINS = (barman, blockworld, gripper, tyreworld)`（对齐论文 Figure 7）  
   - `list_task_ids()`：占位 ID（每域 15 个，共 60，对齐论文规模）  
   - `collect_trajectory()`：**显式 `NotImplementedError`**，提示需要 AgentBoard

2. **`build_synthetic_pddl_trajectories(n)`**  
   - 合成 Gripper / Blockworld 风格 `(command, observation)` 序列  
   - `speculation_type = OBSERVATION`  
   - 可直接走完整回放 + 七档记忆协议（与 ALFWorld 同路径）

3. **与记忆栈的衔接**（无需再改记忆代码）  
   - Type 2 → 全表转移上下文  
   - 观测精确匹配  
   - 混淆用观测模板 type  
   - step0 排除、任务级 commit

验证：

```bash
python scripts/collect_actor.py --benchmark pddl --limit 4 --synthetic
python scripts/replay_eval.py --trajectories data/trajectories/pddl_limit4.jsonl --fake-speculator
```

### 8.2 尚未实现（Live AgentBoard）

论文用的是真实规划环境轨迹（Actor=GPT-5.4）。本仓库 **故意** 停在骨架，原因：

- AgentBoard / IPC PDDL 依赖本地下载与域文件布局，环境差异大；
- 计划约定：装不上环境则骨架不阻塞 ALFWorld/HotpotQA。

要补齐 live 采集，建议按下列步骤接进 `PDDLAdapter.collect_trajectory`。

---

## 9. PDDL Live 接入设计（推荐实现蓝图）

### 9.1 数据与环境

优先对接 **AgentBoard** 的 PDDL 子集（与论文引用 Ma et al. 2024 一致）：

| Domain | 典型算子 | 观测形态 |
|--------|----------|----------|
| Barman | grasp, fill, pour, … | 手持物、杯子状态 |
| Blockworld | pickup, putdown, stack, unstack | 积木 clear / holding |
| Gripper | move, pick, drop | 位置、持球 |
| Tyreworld | open, fetch, remove, … | 轮胎/工具状态文本 |

任务规模目标：**约 60**（可按域均匀抽样或使用 AgentBoard 默认划分）。

### 9.2 轨迹 schema（必须与现有 `Trajectory` 对齐）

每一步：

```json
{
  "step_idx": 0,
  "state_summary": "Domain: gripper\nGoal: ...\nState: ...",
  "action": { "type": "move", "args": { "command": "move a b" } },
  "observation": "Agent moved from a to b. Holding nothing.",
  "conversation_snippet": "..."
}
```

约定：

- `action.type` = 算子名（转移表键）
- `action.args.command` = 完整 grounded 命令字符串
- `observation` = 环境返回的 **完整文本**（评测精确匹配用）
- `success` = 是否达到 PDDL goal
- `speculation_type` = `"observation"`

### 9.3 Actor 采集伪代码

```python
env = AgentBoardPDDLEnv(domain, task_id)
obs = env.reset()
history = [obs]
steps = []
for t in range(max_steps):
    state_summary = render_state(history, goal)
    raw = actor.complete(react_prompt(state_summary, admissible))
    cmd = parse_pddl_action(raw, admissible)   # 必须落在合法算子上
    next_obs, reward, done, info = env.step(cmd)
    steps.append(Step(
        step_idx=t,
        state_summary=state_summary,
        action=Action(type=cmd.split()[0], args={"command": cmd}),
        observation=next_obs,
        conversation_snippet="\n".join(history[-8:]),
    ))
    history += [f"> {cmd}", next_obs]
    if done: break
return Trajectory(..., success=info["goal"], steps=steps, ...)
```

### 9.4 Speculator 在 PDDL 上的输入输出

**输入（空闲窗口）**：

- 当前 `state_summary`
- 正在执行的 `action.format_call()`
- 记忆块：Confusion + **Full transition table** + Episodic ± Miss

**输出**：下一条观测纯文本（与环境字符串一致，含标点与实体名）。

**命中**：`normalize_observation(pred) == normalize_observation(gold)`。

### 9.5 记忆在 PDDL 上为何有效（实现直觉）

| 组件 | PDDL 上的作用 |
|------|----------------|
| Transition table | 学会 `pick→move`、`unstack→putdown` 等高频算子链；Full 表帮助在未知当前算子时仍有全局先验 |
| Episodic | 检索相似 goal / 部分状态，提供 **具体物体名与观测模板** |
| Miss | 纠正“在 clear 积木上却预测 holding …”这类系统性观测错误 |
| Confusion | 当同一观测模板反复被猜错时，硬禁止错误模板 |

这与论文分析一致：规划任务产生高度系统化的预测错误，故 miss 主导。

### 9.6 评测清单（对齐 Table 2 / Figure 3 / 7）

1. 采集 60 条（或子集冒烟）Actor 轨迹，固定种子与任务顺序。  
2. 七档 × k=1 回放，报聚合观测准确率。  
3. 画 cumulative accuracy（经验积累曲线）。  
4. 按域拆分 Barman / Blockworld / Gripper / Tyreworld。  
5. （可选）k=3 best-of-k；打乱顺序做 ordering robustness（论文 Appendix A.3）。

---

## 10. 与论文的已知差距（诚实清单）

| 项目 | 论文 | 本仓库 |
|------|------|--------|
| 官方代码/轨迹 | 未公开 | 独立重实现 |
| Speculator 完整 system prompt | 未完整给出 | 按 §2 + Appendix C 重建 |
| temperature / episodic k | 未明示 | 默认 0 / top_k=3 |
| PDDL 环境 | AgentBoard 真环境 | 骨架 + 合成轨迹 |
| ALFWorld Actor | GPT-5.4 正式轨迹 | 可接 alfworld；默认可合成 |
| HotpotQA 检索 | 未细述后端 | 题内 supporting 段 BM25（可复现，非在线维基） |
| 数值目标 | Table 2 绝对值 | **方向对齐**，非逐点复现 |

---

## 11. 结论

1. **实现主线**是：三类投机目标 × 三套在线记忆 × 七档消融 × 无泄漏回放评测。  
2. **工程关键点**是任务级记忆提交、精确匹配、Type 2 全表上下文、miss/confusion 的在线闭环。  
3. **PDDL** 在论文中是 Type 2 的代表性基准，记忆尤其是 **miss episodes** 带来接近翻倍的观测命中率。  
4. 本仓库已把 PDDL **接到同一套协议**（合成轨迹可跑通）；**Live AgentBoard 采集**按第 9 节蓝图补 `collect_trajectory` 即可，无需改动记忆与回放核心。

---

## 附录 A：关键代码入口

```text
src/swm/memory/transition.py   # 对比转移表
src/swm/memory/episodic.py     # 情景 + miss + arg_sources
src/swm/memory/confusion.py    # 阈值 3
src/swm/memory/stack.py        # 七档 + begin/commit
src/swm/prompts.py             # Appendix C
src/swm/replay.py              # 回放与指标
src/swm/benchmarks/pddl_bench.py
scripts/collect_actor.py
scripts/replay_eval.py
```

## 附录 B：论文 PDDL 主结果摘录（k=1）

Stateless 17.6% → Full 33.8%（+16.2pp，约 1.9×）。  
最强单步：加入 miss（19.0% → 33.0%）。
