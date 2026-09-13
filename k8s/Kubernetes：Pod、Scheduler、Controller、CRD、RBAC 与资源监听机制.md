# Kubernetes：Pod、Scheduler、Controller、CRD、RBAC 与资源监听机制

## 1. Kubernetes 集群里到底运行什么？

核心理解：

> **Scheduler、Controller、Agent 本质上都是程序，不是 Kubernetes Resource Kind。**

如果这些程序采用容器化方式运行，通常结构是：

```
程序
 ↓
Container
 ↓
Pod
 ↓
Kubernetes 管理
```

例如：

```
kube-scheduler
my-controller
my-agent
my-planner
```

都可以作为程序运行在 Container 中，而 Container 通常运行在 Pod 中。

# 2. Pod、Deployment 等是什么？

这些才是 Kubernetes 的 Resource Kind。

常见资源：

```
Pod
Deployment
StatefulSet
DaemonSet
Job
CronJob

Service
ConfigMap
Secret
Ingress
Node
Namespace

Role
ClusterRole
RoleBinding
ClusterRoleBinding

CustomResourceDefinition
```

其中用于管理程序运行的常见资源：

```
Deployment
StatefulSet
DaemonSet
Job
CronJob
Pod
```

例如：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:latest
```

实际结构：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod × 2
    ↓
Container
    ↓
nginx
```

# 3. Scheduler 是什么？

Scheduler **不是一种 Kind**。

它本质上就是一个程序：

```
kube-scheduler
```

如果自定义 Scheduler：

```
my-scheduler
```

也是一个程序。

可以把它部署到 Pod：

```
Deployment / Pod
       ↓
     Pod
       ↓
   Container
       ↓
my-scheduler 程序
```

例如：

```
apiVersion: v1
kind: Pod
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  serviceAccountName: my-scheduler

  containers:
    - name: scheduler
      image: my-scheduler:v1
```

这里：

```
Pod
 └── Container
      └── my-scheduler 进程
```

所以：

> **Pod 不是 Scheduler，Pod 只是承载 Scheduler 程序的运行环境。**

# 4. Controller 是什么？

Controller 同样不是 Kubernetes Kind。

它本质上也是一个程序。

例如：

```
Deployment Controller
Node Controller
Job Controller
自定义 Controller
```

自定义 Controller 通常也是：

```
Controller 程序
      ↓
Container
      ↓
Pod
```

Controller 的核心工作：

```
观察 Kubernetes 当前状态
        ↓
判断是否符合预期
        ↓
执行逻辑
        ↓
修改 Kubernetes 资源
        ↓
继续观察
```

可以理解为：

```
Observe
   ↓
Reconcile
   ↓
Action
   ↓
Observe
```

# 5. Scheduler、Controller、Agent 的本质

可以统一理解成：

```
Scheduler
Controller
Agent
Planner
```

都是：

> **运行中的程序。**

它们的区别主要来自**程序内部实现的逻辑**。

例如 Scheduler：

```
获取待调度 Pod
      ↓
获取 Node 信息
      ↓
执行调度算法
      ↓
选择 Node
      ↓
Bind Pod
```

Controller：

```
监听资源
   ↓
发现状态变化
   ↓
执行 Reconcile
   ↓
修改资源
```

Agent：

```
获取集群状态
   ↓
分析 / 推理 / Planner
   ↓
产生 Action
   ↓
调用 Kubernetes API
   ↓
改变集群状态
```

所以：

> **Kubernetes 并不会因为一个程序叫 Scheduler、Controller 或 Agent，就自动赋予它特殊的业务逻辑。真正的功能由程序代码决定。**

# 6. 程序如何获取 Kubernetes 信息？

程序通常通过：

```
Kubernetes API Server
```

获取集群状态。

例如：

```
Controller
   │
   │ List / Get / Watch
   ↓
API Server
   ↓
Node / Pod / CRD / Service ...
```

程序可以根据自己的需求决定：

```
我要监听什么？
什么时候获取？
获取之后做什么？
```

# 7. List → Watch → 本地缓存

Kubernetes Controller、Scheduler 等程序经常使用：

```
List
 ↓
Watch
 ↓
本地 Cache
```

## 第一步：List

先获取当前资源状态。

例如：

```
List Nodes
```

得到：

```
Node-A
Node-B
Node-C
```

## 第二步：Watch

然后建立 Watch：

```
程序
 ↓
Watch Node
 ↓
API Server
```

程序告诉 API Server：

> 我关心 Node 的变化。

## 第三步：资源变化

例如：

```
Node-B 状态发生变化
```

API Server 产生事件：

```
MODIFIED
```

然后通过 Watch 返回给程序。

```
Node-B 变化
    ↓
API Server
    ↓
Watch Event
    ↓
程序
```

# 8. Informer 与本地缓存

实际 Kubernetes 程序通常不会每次都直接请求 API Server。

常见模式：

```
API Server
    ↓
Informer
    ↓
Local Cache
    ↓
Controller / Scheduler
```

完整流程：

```
                 API Server
                      │
               ┌──────┴──────┐
               │             │
             List           Watch
               │             │
               └──────┬──────┘
                      ↓
                  Informer
                      ↓
                 Local Cache
                      ↓
             Controller / Plugin
```

因此程序可以直接从本地缓存获取资源。

这样可以减少：

```
API Server 请求
网络开销
重复 List
```

# 9. 和 NIO 的类比

你可以把 Kubernetes Watch 理解成一种**事件驱动的资源监听机制**。

和 NIO 有一定相似性。

NIO：

```
注册关注的 Channel
        ↓
Selector
        ↓
发生 I/O 事件
        ↓
通知程序
        ↓
处理事件
```

Kubernetes：

```
注册关注的 Resource
        ↓
Watch
        ↓
Resource 发生变化
        ↓
返回 Event
        ↓
处理事件
```

例如：

```
Node
 ↓
MODIFIED
 ↓
Informer
 ↓
Controller
```

所以可以暂时记成：

> **NIO 是面向 I/O 事件的事件驱动机制；Kubernetes Watch/Informer 是面向 Kubernetes Resource 变化的事件驱动机制。**

# 10. CRD 是什么？

CRD：

```
CustomResourceDefinition
```

它不是 Controller，也不是运行中的程序。

CRD 的作用是：

> **给 Kubernetes 增加一种新的资源类型。**

例如定义：

```
SchedulerRule
```

CRD：

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: schedulerrules.example.com
spec:
  group: example.com
  names:
    kind: SchedulerRule
    plural: schedulerrules
  scope: Cluster
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
```

创建：

```
kubectl apply -f crd.yaml
```

之后 Kubernetes 就认识：

```
SchedulerRule
```

# 11. CRD 和 CR

CRD：

```
定义一种资源类型
```

CR：

```
这种资源的具体实例
```

例如：

```
apiVersion: example.com/v1
kind: SchedulerRule

metadata:
  name: gpu-rule

spec:
  requiredGPU: 2
```

这里：

```
CRD
 ↓
定义 SchedulerRule
```

具体：

```
gpu-rule
 ↓
SchedulerRule 的一个实例
```

# 12. CRD 本身不会执行逻辑

这是非常重要的一点：

```
CRD ≠ Controller
CRD ≠ Agent
CRD ≠ Scheduler
```

CRD 更像：

```
数据结构 / 资源类型定义
```

Controller 才是：

```
真正执行逻辑的程序
```

所以：

```
CRD
 ↓
定义数据结构

Controller
 ↓
读取数据
 ↓
执行逻辑
 ↓
修改资源
```

# 13. CRD 能不能动态获取 Node 状态？

**CRD 自己不能。**

例如：

```
Node CPU = 80%
Node Memory = 70%
```

CRD 不会主动检测。

但是 Controller 可以：

```
Node
 ↓
API Server
 ↓
Controller Watch
 ↓
读取 Node 状态
 ↓
执行逻辑
 ↓
更新 CRD
```

例如：

```
Node-A CPU > 80%
        ↓
Controller 发现
        ↓
计算新的调度状态
        ↓
更新 SchedulerRule CR
```

因此：

```
CRD = 状态/数据的载体
Controller = 执行逻辑的程序
```

# 14. CRD + Controller

这两个经常配套：

```
              Kubernetes
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
       CRD                  Node
        ↓                   ↓
   SchedulerRule        Node Status
        │                   │
        └─────────┬─────────┘
                  ↓
              Controller
                  ↓
            执行 Reconcile
                  ↓
           修改 Kubernetes
```

例如：

```
Node CPU 发生变化
      ↓
Controller Watch Node
      ↓
重新计算
      ↓
更新 CR
      ↓
Scheduler Plugin Watch CR
      ↓
调度时使用最新状态
```

# 15. RBAC：权限系统

程序访问 Kubernetes API 时，不是想访问什么就访问什么。

需要经过 RBAC。

例如：

```
Scheduler Pod
      ↓
ServiceAccount
      ↓
ClusterRoleBinding
      ↓
ClusterRole
      ↓
允许的操作
      ↓
API Server
```

# 16. ServiceAccount

ServiceAccount 是一个**身份**。

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler
  namespace: kube-system
```

它表示：

```
创建一个身份：

my-scheduler
```

# 17. ClusterRole

ClusterRole 才是真正定义权限的地方。

例如：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-scheduler-role

rules:
  - apiGroups: [""]
    resources:
      - nodes
      - pods
    verbs:
      - get
      - list
      - watch
```

这里真正定义：

```
Node
 ↓
get / list / watch

Pod
 ↓
get / list / watch
```

# 18. ClusterRoleBinding

ClusterRoleBinding 的作用是：

> **把 ClusterRole 绑定给某个身份。**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: my-scheduler-binding

subjects:
  - kind: ServiceAccount
    name: my-scheduler
    namespace: kube-system

roleRef:
  kind: ClusterRole
  name: my-scheduler-role
  apiGroup: rbac.authorization.k8s.io
```

关系：

```
my-scheduler
     │
     │ ClusterRoleBinding
     ↓
my-scheduler-role
     │
     ↓
get/list/watch
nodes/pods
```

# 19. serviceAccountName

Pod 中：

```
spec:
  serviceAccountName: my-scheduler
```

意思是：

> 这个 Pod 使用 `my-scheduler` 这个 ServiceAccount 身份。

完整关系：

```
Pod
 │
 │ serviceAccountName
 ↓
ServiceAccount
 │
 │ ClusterRoleBinding
 ↓
ClusterRole
 │
 ↓
RBAC Rules
```

所以：

```
ServiceAccount = 我是谁
ClusterRole = 我能干什么
ClusterRoleBinding = 把权限给谁
serviceAccountName = Pod 使用谁的身份
```

# 20. apiVersion 和 apiGroup

## apiVersion

例如：

```
apiVersion: apps/v1
kind: Deployment
```

表示当前 YAML 对象：

```
Group   = apps
Version = v1
Kind    = Deployment
```

## apiGroup

例如：

```
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: my-scheduler-role
```

这里：

```
apiGroup = rbac.authorization.k8s.io
```

是告诉 Kubernetes：

```
引用的 ClusterRole 属于哪个 API Group
```

注意：

> `roleRef` 的 Schema 本身没有 `version` 字段。

因此不能写成：

```
roleRef:
  apiVersion: rbac.authorization.k8s.io/v1
```

# 21. Kubernetes 资源定位

可以先把 Kubernetes API 资源理解成：

```
Group
+
Version
+
Kind
+
Namespace
+
Name
```

例如：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx
  namespace: default
```

对应：

```
Group     = apps
Version   = v1
Kind      = Deployment
Namespace = default
Name      = nginx
```

# 22. 不同资源的定位例子

### Pod

```
apiVersion: v1
kind: Pod
Group   = ""
Version = v1
Kind    = Pod
```

### Deployment

```
apiVersion: apps/v1
kind: Deployment
Group   = apps
Version = v1
Kind    = Deployment
```

### Ingress

```
apiVersion: networking.k8s.io/v1
kind: Ingress
Group   = networking.k8s.io
Version = v1
Kind    = Ingress
```

### CRD

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
Group   = apiextensions.k8s.io
Version = v1
Kind    = CustomResourceDefinition
```

# 23. GVK 和 GVR

程序访问 Kubernetes API 时，还经常看到：

```
GVK
GVR
```

## GVK

```
Group + Version + Kind
```

例如：

```
apps/v1/Deployment
```

## GVR

```
Group + Version + Resource
```

例如：

```
apps/v1/deployments
```

注意：

```
Kind     = Deployment
Resource = deployments
```

CRD 程序访问：

```
gvr := schema.GroupVersionResource{
    Group:    "example.com",
    Version:  "v1",
    Resource: "schedulerrules",
}
```

就是：

```
example.com/v1/schedulerrules
```

# 24. 整体架构

现在把前面的东西全部串起来：

```
                         Kubernetes
                              │
                         API Server
                              │
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
        Pod                  Node                CRD
          │                                       │
          ↓                                       ↓
     Container                              CR Instance
          │                                       │
          ↓                                       │
 Scheduler / Controller / Agent                   │
          │                                       │
          └───────────────┬───────────────────────┘
                          ↓
                   List / Get / Watch
                          │
                          ↓
                       Informer
                          │
                          ↓
                    Local Cache
                          │
                          ↓
                    程序内部逻辑
                          │
                  ┌───────┴────────┐
                  ↓                ↓
                Action          Decision
                  │
                  ↓
              API Server
                  │
                  ↓
             修改集群状态
```

同时所有 API 操作都受到：

```
ServiceAccount
      ↓
ClusterRoleBinding
      ↓
ClusterRole
      ↓
RBAC
```

控制。

# 25. 最终理解

可以把 Kubernetes 理解成：

```
Kubernetes
│
├── Resource Model
│   ├── Pod
│   ├── Deployment
│   ├── Node
│   ├── Service
│   ├── CRD
│   └── ...
│
├── API Server
│   └── 所有程序通过 API 与集群交互
│
├── RBAC
│   └── 控制程序有没有权限执行 API 操作
│
└── 各种程序
    ├── Scheduler
    ├── Controller
    ├── Agent
    └── 其他业务程序
```

其中：

```
Resource = 数据 / 状态
Program  = 逻辑 / 行为
API      = 程序与集群交互的入口
RBAC     = 权限控制
Watch    = 状态变化通知
Informer = Watch + Cache 等机制
```

最核心的一句话：

> **Kubernetes 本身提供资源模型、API Server、事件监听机制和权限体系；Scheduler、Controller、Agent 等本质上都是运行中的程序，它们通过 Kubernetes API 获取/监听资源状态，然后根据自己的内部逻辑执行操作。**