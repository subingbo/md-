# Kubernetes Scheduler Framework 与多调度器机制

## 1. Kubernetes Scheduler 是什么

`kube-scheduler` 是 Kubernetes Control Plane 中负责 **为未绑定 Node 的 Pod 选择 Node** 的组件。

基本流程：

```
Pod
 ↓
API Server
 ↓
Scheduler
 ↓
选择 Node
 ↓
Bind
 ↓
Pod 运行在 Node
```

# 2. Scheduler Framework 是什么

Scheduler Framework 是 `kube-scheduler` 内部的一套**插件框架**。

它不是一个独立 Scheduler，也不是 Kubernetes 集群中的一个组件。

```
kube-scheduler
│
└── Scheduler Framework
      │
      ├── Extension Points
      │
      ├── Profiles
      │
      └── Plugins
```

作用：

> 允许开发者在 Scheduler 的特定调度阶段插入自己的调度逻辑。

# 3. Plugin 是什么

Plugin 是开发者自己实现的调度逻辑。

例如：

```
MyPlugin
│
├── Filter()
├── Score()
└── Reserve()
```

一个 Plugin **可以实现多个 Extension Point**。

例如：

```
MyPlugin
   │
   ├── Filter
   ├── Score
   └── Reserve
```

# 4. Extension Point 是什么

Extension Point 是 Scheduler Framework 提供的**调度流程插入点**。

例如：

```
Pod
 ↓
PreFilter
 ↓
Filter
 ↓
PreScore
 ↓
Score
 ↓
Reserve
 ↓
Permit
 ↓
Bind
```

不同 Kubernetes 版本的 Extension Point 可能有所变化，因此具体实验时应以对应版本文档和源码为准。

# 5. Profile 是什么

Profile 属于 **kube-scheduler 的配置层**。

它不是：

- 独立 Scheduler
- Kubernetes 资源
- 独立进程

一个 Profile 描述一套 Scheduler 配置，例如：

```
profiles:
  - schedulerName: scheduler-A
    plugins:
      score:
        enabled:
          - name: MyPlugin
```

可以理解为：

```
Profile
│
├── schedulerName
│
└── Plugin 配置
      │
      └── Extension Point
```

# 6. schedulerName 是什么

Pod 中：

```
spec:
  schedulerName: scheduler-A
```

表示：

> 这个 Pod 指定由名为 `scheduler-A` 的 Scheduler 处理。

在 Scheduler Framework 的多 Profile 模式中：

```
profiles:
  - schedulerName: scheduler-A
```

这个 `schedulerName` 是连接 **Pod 和 Profile** 的关键名称。

关系：

```
Pod
│
│ schedulerName = scheduler-A
▼
对应的 Profile
│
│
▼
Plugin 配置
```

注意：

> `schedulerName` 不是 Plugin 的名字。

Plugin 的名字来自：

```
func (p *MyPlugin) Name() string {
    return "MyPlugin"
}
```

配置中的：

```
- name: MyPlugin
```

对应的是 Plugin 名称。

# 7. 一个 Scheduler 可以有多个 Profile

例如：

```
kube-scheduler
│
├── Profile A
│     ├── schedulerName: scheduler-A
│     └── Plugin A
│
└── Profile B
      ├── schedulerName: scheduler-B
      └── Plugin B
```

Pod：

```
spec:
  schedulerName: scheduler-A
```

使用 Profile A。

另一个 Pod：

```
spec:
  schedulerName: scheduler-B
```

使用 Profile B。

因此：

```
一个 kube-scheduler 进程
        │
        ├── Profile A
        │
        └── Profile B
```

**不是两个 Scheduler。**

而是：

> 一个 Scheduler，提供多套调度配置。

# 8. 多个独立 Scheduler

Kubernetes 也可以存在多个独立 Scheduler。

例如：

```
Kubernetes Cluster
│
├── kube-scheduler
│     └── schedulerName = default-scheduler
│
├── scheduler-A
│     └── schedulerName = scheduler-A
│
└── scheduler-B
      └── schedulerName = scheduler-B
```

这里才是真正的：

> 多个独立 Scheduler 实例并存。

Pod：

```
spec:
  schedulerName: scheduler-A
```

由 Scheduler A 处理。

另一个：

```
spec:
  schedulerName: scheduler-B
```

由 Scheduler B 处理。

# 9. Scheduler Framework + 多 Scheduler

如果每个 Scheduler 都使用 Scheduler Framework，可以形成：

```
Kubernetes Cluster
│
├── Scheduler A
│     │
│     └── Scheduler Framework
│           │
│           └── Profile
│                 ├── schedulerName: scheduler-A
│                 └── Plugin A
│
└── Scheduler B
      │
      └── Scheduler Framework
            │
            └── Profile
                  ├── schedulerName: scheduler-B
                  └── Plugin B
```

因此：

```
Scheduler
  ↓
Framework
  ↓
Profile
  ↓
Plugin
  ↓
Extension Point
```

这是一个非常重要的层级关系。

# 10. Plugin 如何进入 Scheduler

Plugin 不是运行时随便放进 Kubernetes 集群的。

基本流程：

```
编写 Plugin
     ↓
注册 Plugin
     ↓
编译进 kube-scheduler
     ↓
生成自定义 Scheduler 镜像
     ↓
部署 Scheduler
```

例如：

```
MyPlugin
   ↓
Plugin Registry
   ↓
自定义 kube-scheduler
   ↓
my-kube-scheduler:v1
```

因此：

> Plugin 是编译进 Scheduler 的，而不是向 Kubernetes 集群“注册一个插件”。

# 11. Scheduler 如何部署进 Kubernetes

Scheduler 本质上是一个程序。

例如已经有：

```
my-kube-scheduler:v1
```

可以将它作为 Kubernetes 工作负载运行。

部署时需要考虑：

```
Scheduler
│
├── Scheduler 镜像
├── Scheduler 配置
└── API Server 访问权限
```

在正式 Kubernetes Control Plane 中，默认 `kube-scheduler` 常见的是 Static Pod；自定义独立 Scheduler 也可以根据部署环境采用 Deployment 等方式运行。

重点是：

> Kubernetes 并不存在一个 `kind: Scheduler` 的内置资源。

例如：

```
kind: Deployment
```

这里只表示 Kubernetes 如何运行这个程序。

**Deployment 本身不会让 Kubernetes 认为它是 Scheduler。**

# 12. Kubernetes 如何“知道”这个程序是 Scheduler

实际上：

> Kubernetes 不需要通过一个 Scheduler Registry 去识别它。

不存在：

```
kubectl register scheduler
```

这样的必要步骤。

真正的机制是：

```
my-scheduler
     │
     │ Watch / List
     ▼
API Server
     │
     ▼
获取 Pod
```

Scheduler 主动从 API Server 获取集群状态。

例如：

```
spec:
  schedulerName: my-scheduler
```

Scheduler 看到这个 Pod 后，就知道：

> 这个 Pod 是我负责处理的。

然后：

```
my-scheduler
     ↓
执行调度算法
     ↓
选择 Node
     ↓
调用 API Server
     ↓
Bind Pod → Node
```

# 13. 为什么多个 Scheduler 不会全部抢同一个 Pod

假设：

```
Pod A
schedulerName: scheduler-A

Pod B
schedulerName: scheduler-B
```

那么：

```
Scheduler A
    ↓
处理 scheduler-A 的 Pod

Scheduler B
    ↓
处理 scheduler-B 的 Pod
```

所以：

```
Pod.schedulerName
        ↓
决定哪个 Scheduler 负责
```

# 14. 为什么采用 Scheduler 主动 Watch

可以有两种设计。

### 方案 A：Scheduler 主动 Watch

Kubernetes 当前采用这种控制器风格：

```
              API Server
             /    |    \
            /     |     \
           ↓      ↓      ↓
     Scheduler  Controller  Controller
        │
       Watch
```

Scheduler 自己从 API Server 获取状态。

### 方案 B：集群主动推送

理论上也完全可以：

```
API Server
    ↓
调度任务分发器
    ↓
┌───┴────┐
↓        ↓
Scheduler A
Scheduler B
```

这同样可以实现多 Scheduler。

所以：

> Scheduler 主动 Watch 并不是“多 Scheduler 必须这样实现”的原因。

它只是 Kubernetes 选择的架构。

# 15. 为什么 Kubernetes 使用 Watch

核心原因是 **解耦 + 状态驱动**。

Kubernetes 整体大量采用：

```
API Server
    ↑
    │ Watch
    │
Controller / Scheduler
```

这样 API Server 不需要维护：

```
有哪些 Scheduler？
Scheduler 在哪里？
Scheduler 是否在线？
如何向 Scheduler 建立连接？
消息如何重试？
```

Scheduler 自己负责：

```
我是谁？
 ↓
我负责哪个 schedulerName？
 ↓
Watch API Server
 ↓
获取属于我的 Pod
```

因此 API Server 和 Scheduler 可以保持较弱的耦合。

# 16. 两种“多调度”方式总结

## 方式一：一个 Scheduler + 多 Profile

```
kube-scheduler
│
├── Profile A
│     └── scheduler-A
│
└── Profile B
      └── scheduler-B
```

特点：

- 一个 Scheduler 进程
- 多套调度配置
- 不同 Pod 可以选择不同 Profile
- 适合不同调度策略共存

## 方式二：多个独立 Scheduler

```
Scheduler A
│
└── Profile
      └── scheduler-A

Scheduler B
│
└── Profile
      └── scheduler-B
```

特点：

- 多个 Scheduler 进程
- 可以使用完全不同的调度算法
- 可以使用不同的 Scheduler 镜像
- 可以独立升级和运行

# 17. 如果研究调度算法，推荐的结构

如果研究重点是：

> **提出一个新的调度算法，并验证它是否优于其他算法。**

推荐：

```
Kubernetes
      │
      ▼
Scheduler Framework
      │
      ▼
Custom Plugin
      │
      ▼
你的调度算法
      │
      ▼
Node
```

没有必要自己重新实现整个 Scheduler。

如果需要同时比较多个算法，可以：

```
Kubernetes Cluster
│
├── Scheduler A
│     └── Algorithm A
│
├── Scheduler B
│     └── Algorithm B
│
└── Scheduler C
      └── Algorithm C
```

然后让不同测试 Pod：

```
schedulerName: scheduler-A
```

或者：

```
schedulerName: scheduler-B
```

这样可以在同一个 Kubernetes 集群中进行不同调度算法的实验。

# 18. 最核心的关系

最终记住这一张：

```
                    Kubernetes Cluster
                           │
                    ┌──────┴──────┐
                    │             │
               Scheduler A    Scheduler B
                    │             │
              Framework      Framework
                    │             │
                 Profile       Profile
                    │             │
                 Plugins       Plugins
                    │             │
              Extension      Extension
                Points          Points
                    │             │
                 算法 A          算法 B
```

而 Pod：

```
Pod
 │
 └── schedulerName
          │
          ▼
    指定负责它的 Scheduler
```

最重要的三个概念：

```
Scheduler
    = 真正运行的调度器程序

Profile
    = Scheduler 内部的一套配置

Plugin
    = 插入 Scheduler 调度流程中的自定义逻辑
```

以及：

```
schedulerName
    = Pod 与 Scheduler/Profile 之间的选择标识
```

最后：

> **Kubernetes 不通过“注册中心”发现 Scheduler；Scheduler 自己通过 API Server 获取状态，Pod 的** `**schedulerName**` **决定它应该由哪个 Scheduler 处理。**