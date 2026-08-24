# Kubernetes 调度架构与 Scheduler Plugin 扩展机制笔记

## 1. Kubernetes 核心设计思想

Kubernetes 是一个**声明式系统**。

核心思想：

> 用户声明期望状态，Kubernetes 各组件通过控制循环（Control Loop）不断让实际状态趋近期望状态。

例如：

用户：

```yaml
kind: Deployment

spec:
  replicas: 3
```

表示：

> 我希望运行 3 个 Pod。

Kubernetes：

```
当前状态:
Pod = 1

期望状态:
Pod = 3

Controller:
创建 2 个 Pod
```

------

# 2. Kubernetes 核心组件关系

整体结构：

```
                 etcd
          (集群状态数据库)
                  |
                  |
             API Server
                  |
     --------------------------------
     |              |               |
 scheduler    controller        kubelet
     |              |               |
 调度决策       状态维护        创建容器
```

------

# 3. etcd

## 3.1 作用

etcd 是 Kubernetes 的状态数据库。

保存：

- Pod
- Node
- Deployment
- Service
- ConfigMap
- CRD

例如：

```
Pod:

metadata:
  name: nginx

spec:
  nodeName: worker1
```

表示：

> nginx Pod 应该运行在 worker1。

------

## 3.2 组件不会直接访问 etcd

错误理解：

```
scheduler
    |
    v
  etcd
```

实际：

```
scheduler
    |
    v
API Server
    |
    v
etcd
```

原因：

etcd 只是存储。

API Server 负责：

- 认证
- 授权
- RBAC
- 参数校验
- API转换
- Admission Controller

------

# 4. API Server 的作用

API Server 是 Kubernetes 的统一入口。

所有组件：

- scheduler
- controller
- kubelet
- kubectl

都通过 API Server 通信。

例如：

```
scheduler

    |
    |
    v

API Server

    |
    |
    v

etcd
```

------

# 5. Kubernetes 组件如何通信

不是：

```
API Server
    |
    | 发送命令
    v
kubelet
```

而是：

```
API Server
    |
    |
保存状态
    |
    v

组件监听状态变化

    |
    v

执行动作
```

------

## 5.1 Watch 机制

组件通常：

- List
- Watch

资源。

例如 scheduler：

```
scheduler

   |
   | watch Pod
   |
   v

API Server

   |
   v

etcd
```

发现：

```
Pod:
  nodeName: ""
```

表示：

Pod 未调度。

------

# 6. Kubernetes 控制循环

每个控制组件本质：

```
while(true){

    获取当前状态

    判断:
    当前状态 != 期望状态?

    执行动作

    更新状态

}
```

------

例如：

Controller：

```
Deployment:

期望:
replicas=3

实际:
Pod=1


Controller:

创建2个Pod
```

------

# 7. Scheduler 的作用

Scheduler 不负责创建容器。

它只负责：

> 决定 Pod 应该运行在哪个 Node。

流程：

```
Pod

 |

Scheduler

 |

选择 Node

 |

更新 Pod.nodeName

 |

kubelet 创建容器
```

------

# 8. Pod 调度完整流程

## 第一步：创建 Pod

```
kubectl

 |

API Server

 |

etcd
```

保存：

```
Pod:

nodeName: ""
```

------

## 第二步：Scheduler 发现

Scheduler watch：

```
发现:

Pod 未绑定 Node
```

进入调度流程。

------

## 第三步：调度计算

执行：

```
QueueSort

PreFilter

Filter

Score

Reserve

Permit

Bind
```

------

## 第四步：写回结果

Scheduler：

修改：

```
Pod:

spec:
  nodeName: worker1
```

通过：

```
Scheduler

 |

API Server

 |

etcd
```

保存。

------

## 第五步：kubelet执行

worker 节点：

```
kubelet

 |

发现 Pod 属于自己

 |

container runtime

 |

创建容器
```

------

# 9. Scheduler Framework

## 9.1 为什么需要插件框架

以前：

```
kube-scheduler

所有调度逻辑写死
```

问题：

- 修改困难
- 不同业务需要不同策略
- scheduler越来越复杂

所以 Kubernetes 提供：

```
Scheduler Framework
```

把调度过程拆成固定扩展点。

------

# 10. Scheduler Extension Point

```
Scheduling Cycle


QueueSort

    |

PreFilter

    |

Filter

    |

PostFilter

    |

PreScore

    |

Score

    |

Reserve

    |

Permit



Binding Cycle


PreBind

    |

Bind

    |

PostBind
```

------

# 11. PreFilter / Filter / PostFilter 区别

## PreFilter

作用：

> 调度前准备数据。

不是过滤 Node。

例如：

Pod：

```
resources:
  cpu: 4
  memory: 8Gi
```

PreFilter：

解析：

```
CPU需求=4
Memory需求=8Gi
```

保存到：

```
CycleState
```

供后续使用。

------

## Filter

作用：

> 判断某个 Node 是否能运行这个 Pod。

输入：

```
Pod + Node
```

例如：

```
Pod:

CPU=4


Node1:
剩余CPU=2

失败


Node2:
剩余CPU=8

通过
```

Filter 过滤的是：

```
Node
```

不是 Pod。

------

## PostFilter

作用：

> Filter 全部失败后的处理。

例如：

```
Pod

↓

Filter

↓

所有Node失败

↓

PostFilter

↓

抢占低优先级Pod
```

------

# 12. 一个 Pod 调度执行多少次？

每个需要调度的 Pod：

都会执行一次调度周期。

例如：

创建：

```
Pod A
Pod B
Pod C
```

执行：

```
Pod A:

PreFilter
Filter
Score
Bind


Pod B:

PreFilter
Filter
Score
Bind


Pod C:

PreFilter
Filter
Score
Bind
```

------

# 13. Scheduler Plugin

Scheduler Plugin 不是普通插件包。

不是：

```
plugin.jar

↓

放入 scheduler
```

而是：

```
插件代码

↓

编译进 scheduler

↓

生成新的 scheduler 镜像

↓

部署到 Kubernetes
```

------

例如：

scheduler-plugins：

```
kube-scheduler

+

Coscheduling

+

CapacityScheduling

+

NetworkAware

...
```

编译成：

```
scheduler-plugins-scheduler
```

------

# 14. Scheduler Plugin 部署

集群中：

```
kube-scheduler

        +

scheduler-plugins-scheduler
```

Pod：

```
spec:
  schedulerName: scheduler-plugins-scheduler
```

表示：

使用自定义 scheduler。

流程：

```
Pod

 |

schedulerName

 |

scheduler-plugins

 |

Scheduler Framework

 |

Plugin执行

 |

选择Node
```

------

# 15. Kubernetes 节点角色

## Control Plane 节点

负责：

- API Server
- Scheduler
- Controller Manager
- etcd

结构：

```
Control Plane

 |
 +-- API Server
 |
 +-- Scheduler
 |
 +-- Controller
 |
 +-- etcd
```

------

## Worker 节点

负责：

- kubelet
- container runtime

结构：

```
Worker

 |
 +-- kubelet
 |
 +-- containerd
 |
 +-- Pod
```

------

# 16. 控制组件必须在主节点吗？

不是。

核心要求：

控制组件：

```
能够访问 API Server

+

拥有 RBAC权限
```

即可。

例如：

```
worker节点

    scheduler

        |

        API Server

        |

        etcd
```

一样可以工作。

------

生产环境通常放 Control Plane：

原因：

- 安全隔离
- 网络稳定
- 方便管理
- 高可用设计

不是 Kubernetes 强制要求。

------

# 17. CRD 与 Controller

## CRD

作用：

> 给 Kubernetes 增加新的资源类型。

例如：

定义：

```
NodeMetric
```

之后：

可以：

```
apiVersion: scheduling.example.com/v1
kind: NodeMetric
```

像 Pod 一样管理。

------

## Controller

作用：

> 监听资源变化，让实际状态达到目标状态。

流程：

```
Controller

 |

watch

 |

API Server

 |

etcd
```

------

# 18. 自定义调度数据如何进入 Kubernetes

Kubernetes 默认知道：

- CPU
- Memory
- GPU

但是不知道：

- 能耗
- 温度
- 网络质量
- 任务耗时

需要扩展。

------

## 常见方案

### Node Label

适合静态信息：

```
GPU型号
区域
硬件类型
```

------

### Annotation

适合简单扩展：

```
annotations:

 power: 200W
```

------

### CRD + Controller（生产常用）

结构：

```
节点

 |

采集程序

 |

Controller

 |

NodeMetric CRD

 |

API Server

 |

etcd

 |

Scheduler Plugin读取
```

------

# 19. Pod 自定义变量

Pod 原生：

```
resources:

 cpu

 memory

 gpu
```

不能直接增加：

```
spec:

 time:
 energy:
```

需要扩展。

------

## Annotation

例如：

```
metadata:

 annotations:

   runtime: "7200"
```

------

## Label

例如：

```
labels:

 workload: training
```

------

## CRD

生产更常见：

例如：

```
TrainingJob
```

保存：

```
GPU需求

预计运行时间

优先级

能耗策略
```

Controller生成Pod。

------

# 20. 一个完整的高级调度架构

```
            用户提交任务

                  |

            TrainingJob CRD

                  |

              Controller

                  |

                Pod

                  |

          Scheduler Plugin


        --------------------

        |                  |

   Pod任务信息        Node指标信息

   runtime           power

   GPU需求           temperature


                  |

              Filter

                  |

              Score

                  |

              Bind

                  |

              kubelet

                  |

            创建容器
```

------

# 21. 核心总结

## Kubernetes 控制模型

```
etcd
 |
API Server
 |
控制组件
 |
修改状态
 |
节点执行
```

## Scheduler

负责：

```
选择Node
```

## kubelet

负责：

```
创建容器
```

## Scheduler Plugin

负责：

```
扩展调度算法
```

## CRD + Controller

负责：

```
扩展Kubernetes状态模型
```

最终：

> Kubernetes 的核心不是组件之间直接发命令，而是所有组件围绕 API Server + etcd 维护集群状态。Scheduler 决策，Controller 调谐状态，kubelet 执行实际容器运行。