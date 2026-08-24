# Kubernetes Scheduler-Plugins 与 Kubernetes 调度架构笔记

## 1. Kubernetes 整体架构

Kubernetes 是一个声明式系统，核心思想：

> 用户声明期望状态，Kubernetes 各组件通过控制循环（Control Loop）不断让实际状态趋近期望状态。

整体结构：
		 	            etcd
		      (集群状态数据库)
			              |
 			             |
   		      API Server

​						  |

​			  |          |           |

 scheduler  controller    kubelet
   		   |          |           |
  		    |          |           |
 调度Pod    维护状态    创建容器

---

# 2. etcd、API Server、组件之间的关系

## 2.1 etcd

etcd 是 Kubernetes 的状态存储。

保存：

- Pod
- Node
- Deployment
- Service
- ConfigMap
- CRD

例如：

```yaml
Pod:
  name: nginx
  nodeName: worker1
```

表示：

> nginx Pod 应该运行在 worker1 节点。

------

## 2.2 API Server

API Server 是 Kubernetes 的统一入口。

组件不会直接访问 etcd。

错误：

```
scheduler
    |
    v
  etcd
```

正确：

```
scheduler
    |
    v
API Server
    |
    v
etcd
```

API Server 负责：

- 认证(Authentication)
- 授权(RBAC)
- 参数校验
- API版本转换
- Admission Controller
- 操作 etcd

------

# 3. Kubernetes 组件如何通信

Kubernetes 不是：

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
    | 保存状态
    |
    v
组件监听状态变化
    |
    v
执行动作
```

组件通过：

- List
- Watch

获取资源变化。

------

# 4. Kubernetes 控制循环

每个核心组件都是一个控制循环：

```
while(true){

    获取当前状态

    判断:
        当前状态 != 期望状态?

    执行动作

    更新状态

}
```

例如：

## Controller

期望：

```
Deployment replicas=3
```

实际：

```
Pod=1
```

Controller：

```
创建2个Pod
```

------

## kubelet

期望：

```
Node worker1运行nginx Pod
```

kubelet：

```
调用container runtime

创建nginx容器
```

------

# 5. Pod 创建完整流程

## 第一步：用户创建Pod

```
kubectl
   |
   v
API Server
   |
   v
etcd
```

etcd保存：

```
Pod:
  nodeName:""
```

表示：

Pod存在，但是没有节点。

------

## 第二步：Scheduler发现未调度Pod

Scheduler通过API Server watch：

```
scheduler
    |
    v
API Server
    |
    v
Pod事件
```

发现：

```
nodeName为空
```

开始调度。

------

## 第三步：Scheduler选择节点

执行：

```
Filter
Score
Bind
```

例如：

```
node1:
 CPU剩余2核 ❌

node2:
 CPU剩余8核 ✅
```

选择：

```
node2
```

------

## 第四步：Scheduler写回绑定结果

Scheduler调用API Server：

修改：

```
Pod:
  nodeName: node2
```

保存到etcd。

------

## 第五步：kubelet创建容器

node2上的kubelet watch API Server：

发现：

```
Pod属于自己
```

执行：

```
kubelet
    |
    v
container runtime
    |
    v
创建容器
```

------

# 6. Scheduler Framework

## 6.1 为什么需要Scheduler Framework

以前：

```
kube-scheduler
    |
    |
所有调度逻辑写死
```

问题：

- 修改困难
- 不同业务需要不同算法
- scheduler越来越复杂

所以 Kubernetes 引入：

```
Scheduler Framework
```

将调度流程拆成固定扩展点。

------

# 7. Scheduler Plugin 工作方式

结构：

```
kube-scheduler

    |
    |
Scheduler Framework

    |
    |
Extension Point

    |
    |
Plugin
```

插件只能插入 Kubernetes 定义好的扩展点。

不能自己增加新的调度阶段。

------

# 8. Scheduler Extension Point

完整流程：

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
NormalizeScore
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

# 9. 常见插件位置

## Filter

作用：

过滤不满足条件的节点。

例如：

```
Pod需要8核CPU

node1:
剩余2核 ❌

node2:
剩余16核 ✅
```

------

## Score

作用：

给节点评分。

例如：

```
node1:
资源利用率30%
score=90


node2:
资源利用率80%
score=40
```

选择最高分。

------

## Reserve

作用：

提前预留资源。

例如：

多个Pod竞争GPU：

```
Pod A
 |
Reserve GPU

Pod B
 |
发现资源已经预留
```

------

## Permit

作用：

等待批准。

例如：

Gang Scheduling:

```
AI任务需要3个Pod同时运行

Pod1
Pod2
Pod3

全部满足后一起通过
```

------

## Bind

作用：

最终绑定：

```
Pod
 |
Node
```

------

# 10. Scheduler Plugin 部署方式

Scheduler Plugin不是普通插件包。

不是：

```
plugin.jar
    |
    v
丢进scheduler
```

而是：

```
插件代码

    |
    v

编译进scheduler

    |
    v

新的scheduler镜像

    |
    v

部署到Kubernetes
```

------

例如：

```
scheduler-plugins

    |
    |
    v

scheduler-plugins-scheduler镜像
```

里面包含：

```
kube-scheduler
+
Scheduler Framework
+
各种Plugin
```

------

# 11. scheduler-plugins项目

scheduler-plugins 是 Kubernetes SIG-Scheduling 官方项目。

提供：

- 调度插件实现
- 定制scheduler
- controller
- CRD

例如：

| 插件                 | 作用            |
| -------------------- | --------------- |
| Coscheduling         | Gang Scheduling |
| Capacity Scheduling  | 容量调度        |
| NodeResourceTopology | NUMA拓扑调度    |
| Trimaran             | 负载感知调度    |
| NetworkAware         | 网络感知调度    |

------

# 12. scheduler-plugins架构

```
                 scheduler

                    |
                    |

        Scheduler Framework

                    |

        ----------------------

        |                    |

     Plugin              Controller

        |                    |

  Filter/Score       PodGroup/CRD
```

------

# 13. kind测试环境

开发Scheduler通常不会直接用生产集群。

使用：

```
kind
```

创建本地Kubernetes。

结构：

```
Docker

├── kind-control-plane
│
│   kube-apiserver
│   etcd
│   kube-scheduler
│   controller-manager
│
├── kind-worker
│
│   kubelet
│
└── kind-worker2
    kubelet
```

------

## 13.1 scheduler部署位置

scheduler运行在：

```
control-plane节点
```

例如：

```
kind-control-plane

    |
    |
scheduler-plugins-scheduler
```

worker节点：

```
worker

    |
    |
kubelet
```

------

# 14. 第二调度器模式

scheduler-plugins常作为第二调度器运行。

集群中：

```
kube-scheduler

        +

scheduler-plugins-scheduler
```

Pod指定：

```
spec:
  schedulerName: scheduler-plugins-scheduler
```

表示：

使用自定义scheduler。

流程：

```
Pod

 |
 |
schedulerName

 |
 |
scheduler-plugins-scheduler

 |
 |
MyScorePlugin

 |
 |
选择Node
```

------

# 15. 最核心理解

Kubernetes调度链：

```
用户提交Pod

      |
      v

API Server

      |
      v

etcd保存状态

      |
      v

Scheduler监听Pod

      |
      v

Scheduler Framework

      |
      v

Plugin执行

      |
      v

修改Pod绑定Node

      |
      v

kubelet发现状态

      |
      v

创建容器
```

------

一句话总结：

> Kubernetes Scheduler Plugin 本质是基于 Scheduler Framework 扩展 kube-scheduler 的调度逻辑，插件代码最终编译进 scheduler 镜像并部署到集群中。Scheduler 不直接控制节点，而是通过 API Server 修改 etcd 中的期望状态，由 kubelet 根据状态变化完成实际容器运行。