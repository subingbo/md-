# FissLock 实验笔记：架构、进程与联调手册

本文是 **p4dev VM 上 BMv2 + Java Sidecar** 的总览笔记，便于复习「谁是谁、谁在跑、为什么这样设计」。  
细节命令见各专题文档。

---

## 1. 仓库里几块代码各干什么

| 目录 | 是否本次实验需要 | 作用 |
|------|------------------|------|
| `switch/p4_bmv2/` | **要** | BMv2 版 P4 数据面 + 脚本 + `test_paths.py` |
| `switch/p4/` + `switch/control/` | 否（要 Tofino） | 论文真机：Tofino P4 + BFRT 控制面 |
| `lib/` | 否（阶段 0.2+） | C++ 完整锁管理：DPDK、agent、等待队列 |
| `java/fisslock-java/` | **要** | Java API + gRPC Sidecar（阶段 0.1） |
| `baseline/` | 否 | NetLock / ParLock 对比实验，不是 FissLock 本体 |
| `experiments/` | 否 | 8 机压测、论文图表复现 |
| `tests/`（根目录） | 否 | 接 `lib` 的 C++ microbench，非 Java |

**结论**：当前路径 = **P4（BMv2）+ Python 验证 + Java Sidecar 演示**；不是完整论文栈。

---

## 2. Sidecar 是什么？

**Sidecar（边车）**：在业务进程 **旁边** 跑一个辅助服务，干业务不方便做的事。

在本项目中：

```text
┌─────────────────────┐     gRPC (:50051)     ┌─────────────────────────┐
│ Java 业务            │ ◄──────────────────► │ Lock Sidecar             │
│ FissLockClient       │                      │ LockSidecarMain (sudo)   │
│ 普通用户、无 DPDK     │                      │ Pcap 发包 + 读 pcap GRANT │
└─────────────────────┘                       └───────────┬─────────────┘
                                                          │
                                                          ▼
                                              veth-inject → simple_switch
```

| 角色 | 类比 |
|------|------|
| Java `FissLock` | 像 **Redisson** 的 `RLock` API |
| Sidecar | 像 **连 Redis 的客户端代理**，但后端是 **交换机锁协议** 不是 Redis |
| `simple_switch` | 像 **可编程交换机芯片**（软件仿真） |

Sidecar **不是**交换机，**不是** C++ `lib`，是 **Java 与交换机之间的适配层**。

---

## 3. 联调时后台在跑什么？

典型需要 **3 个角色**（2～3 个 SSH 终端）：

| # | 进程 | 启动命令 | 前台/后台 | 作用 |
|---|------|----------|-----------|------|
| ① | **simple_switch** | `RESTART=1 bash scripts/start_switch.sh` | **后台**（nohup） | 加载 `fisslock_bmv2.json`，执行 P4 锁逻辑，写 `build/pcap/` |
| ② | **Lock Sidecar** | `bash scripts/vm-run-sidecar.sh` | **前台**（占终端） | gRPC 服务；发 ACQUIRE、读 GRANT |
| ③ | **BasicExample** | `bash scripts/vm-run-example.sh 1` | 前台（跑完退出） | 演示 `tryLock` / `unlock` |

**注意**：只能有 **1 个** `simple_switch` 绑 `fisslock_bmv2.json`。多个实例会导致行为混乱（`vm-check.sh` 会警告）。

### 3.1 虚拟网卡

| 网卡 | 对接 |
|------|------|
| `veth-inject` ↔ `veth-switch` | 交换机 **port 0**；Sidecar / Scapy **从这里注入** |
| `veth-h1` ↔ `veth-host1` | port 1，常作 host_id=1 |
| `veth-h2` ↔ `veth-host2` | port 2，常作 host_id=2 |

### 3.2 重要目录

| 路径 | 作用 |
|------|------|
| `switch/p4_bmv2/build/pcap/` | 交换机抓包；Sidecar **从 `*_out.pcap` 读 GRANT** |
| `switch/p4_bmv2/build/switch.log` | 交换机日志 |
| `java/fisslock-java/target/*.jar` | Sidecar 可执行 JAR |

---

## 4. 一次 `tryLock` 谁做了什么？

1. **BasicExample**：`FissLockClient.connectLocal(1)` → `getLock("order:1001").tryLock()`
2. **gRPC**：`TryAcquireRequest` → Sidecar `:50051`
3. **Sidecar**：`order:1001` 映射为 **`lock_id=1`**（演示用，与 Python 状态机一致）
4. **Sidecar**：在 `veth-inject` 发 **ACQUIRE**（17 字节 UDP 锁包，对齐 `lib/post.h`）
5. **simple_switch**：P4 处理 → 更新 `lock_free` / agent 等 → 产生 **GRANT**
6. **Sidecar**：发现 `build/pcap` 增大，解析 `veth-switch_out.pcap` 等中的 **GRANT_W_AGENT**
7. **gRPC 返回** `granted=true` → Java 进入临界区 → `unlock` → Sidecar 发 **FREE**

**为何从 pcap 读 GRANT？**  
Python `test_paths.py` 已证明交换机正常；在线 `live sniff` 在部分环境收不到包，故 Sidecar 与测试脚本 **同一数据源**（`build/pcap`）。

---

## 5. 三条数据面路径（论文核心）

| 路径 | P4 在做什么 | Python 怎么测 | Java 阶段 0.1 |
|------|-------------|---------------|----------------|
| **状态机** | 空闲锁首次 ACQUIRE → GRANT_W_AGENT | `test_paths --only state` | `order:1001` → lock_id=1，独占 tryLock |
| **组播** | shared 二次 ACQUIRE → mgrp 299 两副本 | `--only mcast` | 未完整覆盖 |
| **counter** | TRANSFER 时 ncnt 与交换机计数一致 | `--only counter` | 未完整覆盖 |

BMv2 三条路径 **已全部 PASS**（Python）见 [BMV2_UBUNTU_VALIDATION_zh.md](./BMV2_UBUNTU_VALIDATION_zh.md)。

---

## 6. Python 测试 vs Java 联调

| | `test_paths.py` | Java + Sidecar |
|--|-----------------|----------------|
| **目的** | 验证 **交换机 P4** | 验证 **Java → Sidecar → 交换机** |
| **入口** | Scapy 直接发包 | gRPC |
| **权限** | 通常要 sudo | 业务无需 sudo；Sidecar 要 sudo |
| **是否算「业务接 FissLock」** | 否 | 是（阶段 0.1） |

两者可共用同一台 VM、同一 `simple_switch`，但 **不要**在 Sidecar 处理请求时重复灌组播 CLI。

---

## 7. 和「完整锁管理」差什么？

| 能力 | 交换机 P4（已验证） | C++ `lib/`（未跑） | Java 0.1 |
|------|-------------------|-------------------|----------|
| 快速 grant / agent 指定 | ✅ | ✅ | ✅（经 Sidecar） |
| 等待队列、agent_daemon | ❌（在主机） | ✅ | ❌ |
| lock_register / RPC | ❌ | ✅ | ❌（演示用 hash→lock_id） |
| Redisson 式租约/看门狗 | ❌ | ❌ | ❌（需 Java 自实现） |

演进路线见 [JAVA_INTEGRATION_zh.md](./JAVA_INTEGRATION_zh.md)（0.2 接 `lib`，1.0 接 Tofino）。

---

## 8. 标准联调顺序（ checklist ）

```bash
# === 终端 1：交换机 ===
cd ~/fisslock/switch/p4_bmv2
export P4C="$HOME/tutorials/p4c-stable/build/p4c"
export PATH="$HOME/tutorials/p4c-stable/build:/usr/local/bin:$PATH"
bash scripts/stop_switch.sh && bash scripts/stop_switch.sh
sudo rm -f build/pcap/*.pcap
RESTART=1 bash scripts/start_switch.sh
echo mc_dump | sudo simple_switch_CLI | grep -A3 'mgrp(299)'

# === 终端 2：Sidecar（保持运行）===
cd ~/fisslock/java/fisslock-java
bash scripts/vm-run-sidecar.sh
# 期望: grant-from-pcap=.../build/pcap

# === 终端 3：Java 示例 ===
cd ~/fisslock/java/fisslock-java
bash scripts/vm-run-example.sh 1
# 期望: held, critical section / released
```

编译 JAR（代码更新后）：

```bash
cd ~/fisslock/java/fisslock-java
mvn -q package -DskipTests
```

---

## 9. 常见问题速查

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `gRPC UNKNOWN` | Sidecar 内未捕获错误（如 pcap 读失败） | 看 **Sidecar 终端堆栈**；`sudo ldconfig`；重编 JAR |
| `tryLock` 卡很久 | 旧版无 gRPC 超时 | 更新代码、`mvn package` |
| `not granted` / timeout | 未收到 GRANT | 确认 1 个 switch；Python state 是否 PASS |
| `ss` 有 50051 但无 Acquire 日志 | 连错进程 / Sidecar 未起 | `sudo ss -tlnp \| grep 50051` |
| 组播 CLI `INVALID_L1_HANDLE` | `mc_node_associate` 用了 rid 而非 handle | 用仓库最新 `multicast.txt`；勿重复灌表 |
| `scp` 失败 `No such file` | 远程目录不存在 | `ssh host "mkdir -p ~/fisslock/java"` |
| 多个 simple_switch | 多次 start 未 stop | `stop_switch.sh` 两次 + `pgrep` 确认 |

---

## 10. 文档索引

| 文档 | 何时看 |
|------|--------|
| [OPERATIONS_MANUAL_zh.md](./OPERATIONS_MANUAL_zh.md) | **按步骤联调**（三终端、CRLF、成功输出） |
| **本文** | 搞清架构、进程、Sidecar 含义 |
| [BMV2_UBUNTU_VALIDATION_zh.md](./BMV2_UBUNTU_VALIDATION_zh.md) | P4 三条路径验证、组播/thrift/pcap 排错 |
| [JAVA_INTEGRATION_zh.md](./JAVA_INTEGRATION_zh.md) | Java 阶段规划、与 Redisson 差异 |
| [JAVA_VM_DEPLOY_zh.md](./JAVA_VM_DEPLOY_zh.md) | scp 上传、VM 三终端步骤 |
| [../switch/LEARNING_zh.md](../switch/LEARNING_zh.md) | P4 / 锁包 / 锁裂变概念 |
| [../java/fisslock-java/README.md](../java/fisslock-java/README.md) | Maven 构建与脚本说明 |

---

## 11. 复现成功标准（当前阶段）

**交换机（Python）**

```text
[状态机] PASS  [组播] PASS  [counter] PASS  全部通过。
```

**Java（阶段 0.1）**

```text
Sidecar: [backend] sent ACQUIRE ...  [backend] GRANT from pcap ...
Example:   held, critical section  →  released
```

达到以上即：**BMv2 数据面 + Java 经 Sidecar 接锁** 联调成功；不等于 OSDI 全文实验复现。