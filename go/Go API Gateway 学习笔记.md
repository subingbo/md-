# Go API Gateway 学习笔记

> 项目：go-gateway-practice
> 技术栈：Go、net/http、Redis、JWT、ReverseProxy、Context、Goroutine、sync.WaitGroup
> 学习重点：HTTP 请求处理、负载均衡、健康检查、控制平面、认证授权、并发控制、优雅停机。

# 一、Go HTTP 服务的基本结构

## 1.1 HTTP Handler 是什么

Go 的 HTTP 服务主要围绕 `http.Handler` 接口组织。

它只有一个方法：

```
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

任何实现了 `ServeHTTP` 方法的类型，都可以作为 HTTP Handler 处理请求。

例如：

```
type MyHandler struct{}

func (h *MyHandler) ServeHTTP(
    w http.ResponseWriter,
    r *http.Request,
) {
    w.Write([]byte("Hello Gateway"))
}
```

注册路由：

```
mux := http.NewServeMux()

mux.Handle("/hello", &MyHandler{})
```

这里的 `MyHandler` 实现了 `http.Handler`，因此可以注册到路由器中。

## 1.2 HandlerFunc 是什么

Go 还提供了一个函数类型：

```
type HandlerFunc func(ResponseWriter, *Request)
```

它实现了 `http.Handler` 接口：

```
func (f HandlerFunc) ServeHTTP(
    w ResponseWriter,
    r *Request,
) {
    f(w, r)
}
```

因此，普通函数也可以直接作为 Handler。

例如：

```
func helloHandler(
    w http.ResponseWriter,
    r *http.Request,
) {
    w.Write([]byte("Hello"))
}
```

注册：

```
mux.HandleFunc("/hello", helloHandler)
```

两者的区别：

- `http.Handler`：接口，要求实现 `ServeHTTP`。
- `http.HandlerFunc`：函数类型，可以把普通函数转换成 Handler。

## 1.3 为什么有些 Handler 返回 HandlerFunc

项目中的代码：

```
func listUpstreamsHandler(
    balancers map[string]*RoundRobinBalancer,
) http.HandlerFunc {

    return func(
        w http.ResponseWriter,
        r *http.Request,
    ) {
        // 处理请求
    }
}
```

这实际上是一个 Handler 工厂函数。

它接收 `balancers`，然后返回一个闭包。

闭包可以访问外部函数传入的 `balancers`，因此不需要每次处理 HTTP 请求时重新传递负载均衡器。

注册：

```
apiMux.HandleFunc(
    "GET /api/upstreams",
    listUpstreamsHandler(balancers),
)
```

执行过程：

```
main()
  |
  v
listUpstreamsHandler(balancers)
  |
  v
返回 HandlerFunc
  |
  v
注册到 ServeMux
  |
  v
HTTP 请求到达
  |
  v
执行返回的匿名函数
```

# 二、HTTP 请求进入 Gateway 后的完整链路

这是整个项目最重要的基础。

## 2.1 Gateway 主路由

项目中：

```
apiHandler := authMiddleware(
    apiMux,
    jwtSecret,
)

apiHandler = timeoutMiddleware(
    apiHandler,
    apiTimeout,
)

mux.Handle("/api/", apiHandler)
```

外层日志和指标中间件：

```
handler := metricsMiddleware(
    loggingMiddleware(mux),
    gatewayMetrics,
)
```

HTTP Server：

```
server := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
}
```

## 2.2 实际请求路径

以：

```
GET /api/orders/slow
Authorization: Bearer <JWT>
```

为例。

请求执行路径：

```
客户端
  |
  v
HTTP Server :8080
  |
  v
metricsMiddleware
  |
  v
loggingMiddleware
  |
  v
主路由 mux
  |
  v
/api/ 路由
  |
  v
timeoutMiddleware
  |
  v
authMiddleware
  |
  v
apiMux
  |
  v
具体业务 Handler
  |
  v
返回响应
```

注意：中间件的包装顺序决定了请求进入和返回的执行顺序。

例如：

```
apiHandler := authMiddleware(apiMux, jwtSecret)

apiHandler = timeoutMiddleware(
    apiHandler,
    apiTimeout,
)
```

等价于：

```
timeoutMiddleware(
    authMiddleware(apiMux, jwtSecret),
    apiTimeout,
)
```

因此超时中间件在外层，认证中间件在内层。

# 三、Context 上下文机制

## 3.1 Context 是什么

`context.Context` 用于在多个函数和 Goroutine 之间传递：

- 取消信号。
- 超时信号。
- 截止时间。
- 请求范围内的少量数据。

它本身不是用来强制终止 Goroutine 的。

## 3.2 创建带超时的 Context

项目中的超时中间件：

```
func timeoutMiddleware(
    next http.Handler,
    timeout time.Duration,
) http.Handler {

    return http.HandlerFunc(func(
        w http.ResponseWriter,
        r *http.Request,
    ) {

        ctx, cancel := context.WithTimeout(
            r.Context(),
            timeout,
        )

        defer cancel()

        newRequest := r.WithContext(ctx)

        next.ServeHTTP(w, newRequest)
    })
}
```

重点理解：

```
ctx, cancel := context.WithTimeout(
    r.Context(),
    timeout,
)
```

这里创建了一个子 Context。

它有两种取消方式：

1. 到达 timeout 指定的时间，自动取消。
2. 主动调用 cancel()，立即取消。

## 3.3 defer cancel() 是什么意思

```
defer cancel()
```

不是创建完 Context 就立即执行 cancel。

`defer` 的含义是：

**当前函数返回时，执行注册的函数。**

例如：

```
func test() {
    defer fmt.Println("B")

    fmt.Println("A")
}
```

输出：

```
A
B
```

回到超时中间件：

```
ctx, cancel := context.WithTimeout(
    r.Context(),
    timeout,
)

defer cancel()

next.ServeHTTP(w, r.WithContext(ctx))
```

执行顺序：

```
创建 Context
    |
    v
注册 defer cancel()
    |
    v
调用下一个 Handler
    |
    v
等待下一个 Handler 返回
    |
    v
当前中间件函数返回
    |
    v
执行 cancel()
```

如果超时时间先到，Context 会自动取消，不需要等函数返回。

## 3.4 Context 的父子取消关系

例如：

```
parent := context.Background()

child, cancel := context.WithCancel(parent)
```

父 Context 取消时，子 Context 也会取消。

```
parent Context
      |
      v
child Context
      |
      v
grandchild Context
```

取消传播：

```
parent 取消
    |
    v
child 自动取消
    |
    v
grandchild 自动取消
```

不需要手动逐层取消。

但是反过来，取消 child 不会取消 parent。

## 3.5 如何监听取消信号

Context 内部提供了一个 Channel：

```
ctx.Done()
```

它是一个只读 Channel。

当 Context 被取消或者超时时，这个 Channel 会关闭。

例如：

```
go func() {
    for {
        select {
        case <-ctx.Done():
            log.Println("Goroutine stopped")
            return

        case <-ticker.C:
            doSomething()
        }
    }
}()
```

这里：

```
case <-ctx.Done():
```

就是监听取消信号。

注意：

**Context 不会主动杀死 Goroutine，而是通知 Goroutine 应该退出。**

Goroutine 必须主动检查信号，或者调用能够响应 Context 的函数。

# 四、负载均衡与反向代理

## 4.1 LoadBalancer 接口

项目定义：

```
type LoadBalancer interface {
    Next() *url.URL
}
```

接口只规定一件事：

选择下一个可用的后端实例。

轮询负载均衡器：

```
type RoundRobinBalancer struct {
    mu                  sync.RWMutex
    targets             []*upstream
    counter             atomic.Uint64
    healthChecksStarted atomic.Bool

    healthClient *http.Client
}
```

其中：

- `mu`：保护后端实例切片。
- `targets`：保存后端实例。
- `counter`：原子计数器，用于轮询。
- `healthChecksStarted`：保证健康检查只启动一次。
- `healthClient`：健康检查专用 HTTP Client。

## 4.2 Next() 如何选择实例

核心代码：

```
func (b *RoundRobinBalancer) Next() *url.URL {

    targets := b.snapshotTargets()

    if len(targets) == 0 {
        return nil
    }

    start := (b.counter.Add(1) - 1) %
        uint64(len(targets))

    for offset := 0; offset < len(targets); offset++ {

        index := (int(start) + offset) % len(targets)

        candidate := targets[index]

        if !candidate.healthy.Load() {
            continue
        }

        target := *candidate.target

        return &target
    }

    return nil
}
```

假设有三个实例：

```
A：健康
B：不健康
C：健康
```

轮询：

```
第一次：A
第二次：C
第三次：A
第四次：C
```

如果没有健康实例：

```
return nil
```

反向代理就会返回 HTTP 503。

## 4.3 为什么使用 snapshotTargets()

代码：

```
func (b *RoundRobinBalancer) snapshotTargets() []*upstream {

    b.mu.RLock()
    defer b.mu.RUnlock()

    return append(
        []*upstream(nil),
        b.targets...,
    )
}
```

这里：

```
b.targets...
```

是 Go 的切片展开语法。

例如：

```
numbers := []int{1, 2, 3}

result := append([]int(nil), numbers...)
```

会把 numbers 中的元素追加到一个新的切片。

为什么需要快照？

因为负载均衡器可能同时发生：

- 添加实例。
- 删除实例。
- 选择实例。
- 查询实例。

如果直接遍历原始切片，同时其他 Goroutine 修改它，就可能出现并发安全问题。

快照让 Next() 遍历独立的切片结构。

注意：这里复制的是指针，不是每个 upstream 对象本身。实例的健康状态仍通过 atomic.Bool 读取。

## 4.4 反向代理如何工作

项目定义：

```
type loadBalancedReverseProxy struct {
    balancer LoadBalancer
    proxy    *httputil.ReverseProxy
}
```

它同时持有：

- 负载均衡器。
- 反向代理。

实现：

```
func (p *loadBalancedReverseProxy) ServeHTTP(
    w http.ResponseWriter,
    r *http.Request,
) {

    target := p.balancer.Next()

    if target == nil {
        http.Error(
            w,
            "Service Unavailable",
            http.StatusServiceUnavailable,
        )
        return
    }

    ctx := withSelectedUpstream(
        r.Context(),
        target,
    )

    p.proxy.ServeHTTP(
        w,
        r.WithContext(ctx),
    )
}
```

执行逻辑：

```
客户端请求
    |
    v
Gateway
    |
    v
balancer.Next()
    |
    v
选择健康实例
    |
    v
把目标地址放进 Context
    |
    v
ReverseProxy
    |
    v
发送请求到后端
    |
    v
后端返回响应
    |
    v
Gateway 返回客户端
```

## 4.5 Rewrite 如何设置目标地址

```
Rewrite: func(pr *httputil.ProxyRequest) {

    targetURL, ok := selectedUpstreamFromContext(
        pr.In.Context(),
    )

    if !ok {
        return
    }

    pr.SetURL(targetURL)

    pr.Out.URL.Path = strings.TrimPrefix(
        pr.Out.URL.Path,
        prefix,
    )

    pr.Out.URL.RawPath = ""

    pr.SetXForwarded()
},
```

几个关键方法：

- `pr.In`：客户端原始请求。
- `pr.Out`：准备发送给后端的请求。
- `pr.SetURL()`：设置后端目标地址。
- `strings.TrimPrefix()`：删除路径前缀。
- `pr.SetXForwarded()`：设置转发相关请求头。

# 五、健康检查机制

## 5.1 健康检查检查什么

项目中：

```
if err := balancer.StartHealthChecks(
    healthCtx,
    5*time.Second,
    1*time.Second,
); err != nil {
    log.Fatal(err)
}
```

含义：

- 每隔 5 秒检查一次。
- 单次 HTTP 检查最多等待 1 秒。

## 5.2 健康检查 URL 从哪里来

关键代码：

```
healthURL := target.JoinPath("health")
```

例如：

```
target:
http://127.0.0.1:9090

healthURL:
http://127.0.0.1:9090/health
```

因此 URL 不需要额外传入。

健康检查函数自己根据 target 构造。

## 5.3 HTTP Client 的作用

```
healthClient: &http.Client{
    CheckRedirect: func(
        req *http.Request,
        via []*http.Request,
    ) error {
        return http.ErrUseLastResponse
    },
},
```

这个 Client 专门用于健康检查。

不跟随 HTTP 重定向。

因为我们希望：

```
200 OK：健康
302 Found：不健康
500 Internal Server Error：不健康
连接失败：不健康
```

健康检查请求：

```
req, err := http.NewRequestWithContext(
    ctx,
    http.MethodGet,
    healthURL.String(),
    nil,
)
```

发送：

```
resp, err := client.Do(req)
```

判断：

```
return resp.StatusCode >= 200 &&
    resp.StatusCode < 300
```

只有 HTTP 2xx 才认为实例健康。

## 5.4 健康检查与实际业务请求的区别

两者不是同一个 HTTP 请求。

健康检查：

```
Gateway
   |
   | GET /health
   v
后端实例
```

实际代理请求：

```
客户端
   |
   | GET /api/orders/slow
   v
Gateway
   |
   | 转发业务请求
   v
后端实例
```

健康检查只负责判断实例是否可以接收流量。

# 六、控制平面与数据平面

这是 Gateway 架构中的一个重要概念。

## 6.1 数据平面 Data Plane

负责实际业务流量的转发。

例如：

```
客户端请求
    |
    v
Gateway
    |
    v
负载均衡
    |
    v
选择后端
    |
    v
反向代理
    |
    v
后端服务
```

对应代码：

```
balancer.Next()
```

以及：

```
p.proxy.ServeHTTP(w, r)
```

## 6.2 控制平面 Control Plane

负责管理 Gateway 的运行状态和配置。

例如：

- 查询后端实例。
- 添加后端实例。
- 删除后端实例。
- 查看服务路由。
- 查看运行指标。

项目中对应：

```
GET /api/upstreams?service=/api/users/
```

查询后端实例。

```
POST /api/upstreams
```

添加后端实例。

请求体：

```
{
  "service": "/api/users/",
  "target": "http://127.0.0.1:9999"
}
```

删除：

```
DELETE /api/upstreams?service=/api/users/&target=http://127.0.0.1:9999
```

## 6.3 为什么还需要 HTTP Handler

负载均衡器本身已经有方法：

```
balancer.AddTarget(target)
balancer.RemoveTarget(target)
balancer.ListTargets()
```

这些是内部 Go 方法，只能在程序内部调用。

HTTP Handler 则把这些能力暴露成 HTTP API。

例如：

```
func addUpstreamHandler(
    balancers map[string]*RoundRobinBalancer,
) http.HandlerFunc {

    return func(w http.ResponseWriter, r *http.Request) {

        // 解析 JSON

        // 获取 service

        // 查找对应 balancer

        // 调用 balancer.AddTarget()

        // 返回 JSON
    }
}
```

两者区别：

| 维度     | Balancer 方法    | HTTP Handler     |
| -------- | ---------------- | ---------------- |
| 作用     | 操作内存中的实例 | 对外提供管理接口 |
| 调用方式 | Go 函数调用      | HTTP 请求        |
| 参数     | Go 参数          | URL、JSON        |
| 返回     | Go 类型          | HTTP 响应        |
| 使用场景 | 程序内部         | 管理端、控制台   |

可以理解为：

**Balancer 是实际执行管理操作的组件，HTTP Handler 是对外暴露的管理入口。**

## 6.4 为什么添加和删除要管理员权限

项目中：

```
apiMux.Handle(
    "POST /api/upstreams",
    adminOnlyMiddleware(
        addUpstreamHandler(balancers),
    ),
)
```

删除：

```
apiMux.Handle(
    "DELETE /api/upstreams",
    adminOnlyMiddleware(
        removeUpstreamHandler(balancers),
    ),
)
```

查询：

```
apiMux.HandleFunc(
    "GET /api/upstreams",
    listUpstreamsHandler(balancers),
)
```

查询允许普通用户和管理员使用。

添加、删除仅允许管理员使用。

这是 RBAC（基于角色的访问控制）。

# 七、JWT 认证与 RBAC 授权

## 7.1 登录流程

项目使用：

- 用户名和密码。
- bcrypt 密码哈希。
- JWT Access Token。
- Redis Refresh Token。

登录：

```
POST /auth/login
```

请求：

```
{
  "username": "admin",
  "password": "DevPassword123!"
}
```

执行：

```
客户端
   |
   v
loginHandler
   |
   v
users.Authenticate()
   |
   v
bcrypt 验证密码
   |
   v
generateJWT()
   |
   v
RefreshTokenManager.Issue()
   |
   v
Redis 保存 Refresh Token
   |
   v
返回 Access Token
```

## 7.2 JWT Claims

项目自定义：

```
type GatewayClaims struct {
    UserID string `json:"user_id"`
    jwt.RegisteredClaims
}
```

其中包含：

- UserID：用户 ID。
- Issuer：签发者。
- Subject：用户主体。
- Audience：接收方。
- IssuedAt：签发时间。
- NotBefore：生效时间。
- ExpiresAt：过期时间。
- ID：JWT 唯一标识。

项目使用 HS256 对称签名。

## 7.3 认证中间件

```
authorization := r.Header.Get("Authorization")
```

检查 Bearer：

```
if !strings.HasPrefix(authorization, "Bearer ") {
    http.Error(w, "Unauthorized", 401)
    return
}
```

提取 Token：

```
tokenString := strings.TrimPrefix(
    authorization,
    "Bearer ",
)
```

验证：

```
claims, err := verifyJWT(
    tokenString,
    secret,
)
```

将用户 ID 放入 Context：

```
ctx := context.WithValue(
    r.Context(),
    userIDContextKey{},
    claims.UserID,
)
```

这样下游 Handler 就能获取当前用户身份。

## 7.4 认证与授权的区别

认证 Authentication：

回答：

```
你是谁？
```

例如验证 JWT，确定用户是 `dev-user-001`。

授权 Authorization：

回答：

```
你能做什么？
```

例如：

```
admin：允许添加、删除实例
reader：允许查询实例
```

认证通过不代表授权通过。

# 八、Goroutine 与并发管理

## 8.1 Goroutine 是什么

Goroutine 是 Go 的轻量级并发执行单元。

启动：

```
go func() {
    fmt.Println("Running")
}()
```

主 Goroutine 不会自动等待它完成。

例如：

```
func main() {
    go func() {
        time.Sleep(5 * time.Second)
        fmt.Println("Done")
    }()

    fmt.Println("Main exit")
}
```

程序可能在后台任务执行完成之前就退出。

因为：

**Go 程序的 main 函数返回后，整个进程就会退出，不会自动等待其他 Goroutine。**

## 8.2 Context 与 WaitGroup 的分工

这是非常重要的知识点。

Context：

```
通知 Goroutine 停止
```

WaitGroup：

```
等待 Goroutine 真正结束
```

它们不能互相替代。

## 8.3 sync.WaitGroup

项目中用于等待健康检查 Goroutine。

结构体增加：

```
type RoundRobinBalancer struct {
    // 其他字段

    healthCheckWG sync.WaitGroup
}
```

启动 Goroutine：

```
b.healthCheckWG.Add(1)

go func() {

    defer b.healthCheckWG.Done()

    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {

        case <-ctx.Done():
            log.Println("Upstream health checks stopped")
            return

        case <-ticker.C:
            b.checkAll(ctx, timeout)
        }
    }
}()
```

等待：

```
func (b *RoundRobinBalancer) WaitHealthChecks() {
    if b == nil {
        return
    }

    b.healthCheckWG.Wait()
}
```

三个方法：

| 方法   | 作用               |
| ------ | ------------------ |
| Add(1) | 任务计数加一       |
| Done() | 任务完成，计数减一 |
| Wait() | 等待计数归零       |

注意：`WaitGroup` 不会停止 Goroutine，只会等待。

## 8.4 与 Java JUC 的区别

Go：

```
wg.Add(1)

go func() {
    defer wg.Done()
    doSomething()
}()

wg.Wait()
```

Java：

```
CountDownLatch latch = new CountDownLatch(1);

executor.submit(() -> {
    try {
        doSomething();
    } finally {
        latch.countDown();
    }
});

latch.await();
```

两者用途非常相似。

| 对比     | WaitGroup                        | CountDownLatch |
| -------- | -------------------------------- | -------------- |
| 任务计数 | Add                              | 初始化计数     |
| 任务完成 | Done                             | countDown      |
| 等待     | Wait                             | await          |
| 重复使用 | 可以，但需遵守 Add/Wait 时序规则 | 不可以重置     |
| 主要用途 | 等待一批 Goroutine               | 等待一批任务   |

`WaitGroup` 和 `CountDownLatch` 是近似对应关系，但不是完全相同的实现。

### CyclicBarrier

CyclicBarrier 的作用不同。

它用于多个线程在同一个同步屏障处相互等待。

例如：

```
A 完成第一阶段 ─┐
B 完成第一阶段 ─┼─ 全部到齐后一起进入第二阶段
C 完成第一阶段 ─┘
```

区别：

- CountDownLatch：等待任务完成。
- WaitGroup：等待 Goroutine 完成。
- CyclicBarrier：等待多个参与者到齐，再一起继续。

# 九、HTTP Server 的生命周期

## 9.1 创建 HTTP Server

```
server := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
}
```

这里仅仅创建 Server 对象。

并没有开始监听端口。

真正启动：

```
server.ListenAndServe()
```

它会：

1. 监听指定端口。
2. 接受 TCP 连接。
3. 读取 HTTP 请求。
4. 调用 Handler 处理请求。
5. 返回 HTTP 响应。

## 9.2 ListenAndServe 为什么放进 Goroutine

项目：

```
serverErr := make(chan error, 1)

go func() {
    serverErr <- server.ListenAndServe()
}()
```

因为 `ListenAndServe()` 是阻塞调用。

如果直接在 main 中执行：

```
server.ListenAndServe()
```

main 就会一直停留在这里，无法继续执行后面的信号监听逻辑。

因此放入独立 Goroutine。

## 9.3 serverErr 是什么

```
serverErr := make(chan error, 1)
```

它是一个用于传递错误的 Channel。

HTTP Server 退出时：

```
serverErr <- server.ListenAndServe()
```

将退出结果发送给 main。

例如：

正常关闭：

```
http.ErrServerClosed
```

端口被占用：

```
listen tcp :8080: bind: address already in use
```

`serverErr` 接收的是 HTTP Server 的 `ListenAndServe()` 返回值，不是某个 HTTP 请求的业务错误。

## 9.4 为什么还需要 select

```
select {

case err := <-serverErr:

    serverExited = true

    if err != nil &&
        !errors.Is(err, http.ErrServerClosed) {
        log.Printf("Gateway server failed: %v", err)
    }

case <-rootCtx.Done():

    log.Printf("Shutdown signal received")
}
```

这里等待两种情况：

1. HTTP Server 自己退出。
2. 收到操作系统的退出信号。

如果 Server 异常退出，main 就能及时发现。

如果收到 Ctrl+C，则开始执行优雅停机。

# 十、优雅停机 Graceful Shutdown

这是当前项目的重要阶段。

## 10.1 为什么需要优雅停机

直接退出：

```
Ctrl+C
  |
  v
进程直接终止
  |
  v
正在执行的 HTTP 请求可能中断
  |
  v
资源清理不保证执行
```

优雅停机：

```
Ctrl+C
  |
  v
接收退出信号
  |
  v
停止接收新请求
  |
  v
等待正在执行的请求
  |
  v
关闭后台 Goroutine
  |
  v
关闭 Redis
  |
  v
main 返回
  |
  v
进程退出
```

## 10.2 signal.NotifyContext

项目：

```
rootCtx, stopRoot := signal.NotifyContext(
    context.Background(),
    os.Interrupt,
    syscall.SIGTERM,
)

defer stopRoot()
```

作用：

监听操作系统信号。

常见信号：

- `os.Interrupt`：通常对应 Ctrl+C。
- `syscall.SIGTERM`：请求进程正常终止。

当收到信号：

```
<-rootCtx.Done()
```

会被触发。

## 10.3 健康检查 Context 为什么继承 rootCtx

```
healthCtx, stopHealthChecks := context.WithCancel(rootCtx)
```

健康检查 Context 是 rootCtx 的子 Context。

因此：

```
操作系统信号
    |
    v
rootCtx 取消
    |
    v
healthCtx 自动取消
    |
    v
健康检查 Goroutine 收到 Done()
    |
    v
退出
```

也可以主动调用：

```
stopHealthChecks()
```

取消健康检查。

## 10.4 Server.Shutdown()

核心代码：

```
shutdownCtx, cancel := context.WithTimeout(
    context.Background(),
    10*time.Second,
)

defer cancel()

if err := server.Shutdown(shutdownCtx); err != nil {

    log.Printf("Graceful shutdown failed: %v", err)

    if closeErr := server.Close(); closeErr != nil {
        log.Printf("Force close failed: %v", closeErr)
    }
}
```

`Shutdown()` 的主要工作：

1. 关闭监听器，不再接受新的连接。
2. 关闭空闲连接。
3. 等待正在处理请求的连接变为空闲。
4. 所有请求处理完成后返回。

注意：

**Shutdown 不会立即终止正在处理的 HTTP 请求。**

它会等待这些请求结束。

## 10.5 为什么要设置 10 秒

假设某个订单请求需要 5 秒。

你在请求执行 2 秒时按下 Ctrl+C。

```
请求执行 2 秒
    |
    v
收到退出信号
    |
    v
Shutdown 开始
    |
    v
关闭监听端口
    |
    v
等待剩余 3 秒
    |
    v
请求完成
    |
    v
Shutdown 返回
```

如果请求超过 10 秒仍然没有结束：

```
server.Shutdown(shutdownCtx)
```

会因为 shutdownCtx 超时而返回错误。

随后：

```
server.Close()
```

强制关闭连接。

所以 10 秒是优雅关闭的等待上限，不是固定等待时间。

## 10.6 Shutdown 与 Close 的区别

| 对比     | Shutdown | Close    |
| -------- | -------- | -------- |
| 监听器   | 关闭     | 关闭     |
| 空闲连接 | 关闭     | 关闭     |
| 活跃连接 | 等待完成 | 强制关闭 |
| 是否等待 | 是       | 不等待   |
| 主要用途 | 优雅停机 | 强制停机 |

## 10.7 什么是正在处理的 HTTP 请求

例如：

```
GET /api/orders/slow
```

这个请求进入 Gateway 后，会被 HTTP Server 分配给对应的 Handler 执行。

Server 会跟踪它对应的连接状态。

因此：

```
server.Shutdown(shutdownCtx)
```

可以知道还有多少连接处于活跃状态。

并不需要自己维护一个 HTTP 请求列表。

这也是为什么 Shutdown 能够等待正在执行的订单请求。

# 十一、优雅停机中的完整代码流程

项目中的核心逻辑可以归纳为：

```
// 1. 启动 HTTP Server
serverErr := make(chan error, 1)

go func() {
    serverErr <- server.ListenAndServe()
}()

serverExited := false

// 2. 等待退出信号或者 Server 异常退出
select {

case err := <-serverErr:

    serverExited = true

    if err != nil &&
        !errors.Is(err, http.ErrServerClosed) {
        log.Printf("Gateway server failed: %v", err)
    }

case <-rootCtx.Done():

    log.Printf("Shutdown signal received")
}

// 3. 停止健康检查
stopHealthChecks()

// 4. 等待 HTTP Server 优雅关闭
shutdownCtx, cancel := context.WithTimeout(
    context.Background(),
    10*time.Second,
)

if err := server.Shutdown(shutdownCtx); err != nil {

    log.Printf("Graceful shutdown failed: %v", err)

    if closeErr := server.Close(); closeErr != nil {
        log.Printf("Force close failed: %v", closeErr)
    }
}

cancel()

// 5. 确认 HTTP Server Goroutine 退出
if !serverExited {
    err := <-serverErr

    if err != nil &&
        !errors.Is(err, http.ErrServerClosed) {
        log.Printf("Gateway server exit error: %v", err)
    }
}

// 6. 等待所有健康检查 Goroutine 退出
for _, balancer := range balancers {
    balancer.WaitHealthChecks()
}

// 7. main 返回，执行 defer 中的资源清理
```

## 11.1 为什么还要接收一次 serverErr

前面的 select 可能走两种分支。

情况 A：HTTP Server 异常退出。

```
select 收到 serverErr
    |
    v
serverExited = true
    |
    v
不需要再次接收
```

情况 B：手动退出。

```
select 收到 rootCtx.Done()
    |
    v
执行 Shutdown
    |
    v
等待 HTTP Server 退出
    |
    v
接收 serverErr
```

因此：

```
if !serverExited {
    err := <-serverErr
}
```

是为了确保 HTTP Server 启动 Goroutine 已经返回。

## 11.2 为什么不能只依赖 main 返回

Go 程序的 main 函数返回后，整个进程就会退出。

即使其他 Goroutine 还没有执行完成，也不会继续等待它们。

因此需要：

```
balancer.WaitHealthChecks()
```

在 main 返回之前，明确等待健康检查 Goroutine 退出。

# 十二、WaitGroup 在 Gateway 中的完整作用

假设 Gateway 有三个后端服务，每个负载均衡器都有一个健康检查 Goroutine。

```
Gateway
  |
  +-- RoundRobinBalancer A
  |       |
  |       +-- 健康检查 Goroutine A
  |
  +-- RoundRobinBalancer B
  |       |
  |       +-- 健康检查 Goroutine B
  |
  +-- RoundRobinBalancer C
          |
          +-- 健康检查 Goroutine C
```

收到退出信号：

```
stopHealthChecks()
       |
       v
healthCtx 取消
       |
       +---- A 收到 Done()
       |
       +---- B 收到 Done()
       |
       +---- C 收到 Done()
       |
       v
三个 Goroutine 退出
       |
       v
各自执行 Done()
       |
       v
WaitHealthChecks()
       |
       v
全部完成
```

这就构成了完整的 Goroutine 生命周期管理。

需要注意：当前项目的 `StartHealthChecks()` 在启动后台循环前还会同步执行一次初始 `checkAll()`。因此初始检查本身由调用它的启动流程执行，而 WaitGroup 跟踪的是后台健康检查循环 Goroutine。

# 十三、go run 与 gateway.exe 的区别

## 13.1 go run

执行：

```
go run ./gateway
```

Go 工具会编译目标包并运行生成的可执行文件。

从概念上看：

```
powershell.exe
    |
    v
go.exe
    |
    v
编译并启动 Gateway 可执行程序
```

因此，使用 go run 时会涉及 Go 命令工具和实际运行的 Gateway 程序。

## 13.2 go build

执行：

```
go build -o .\gateway.exe .\gateway
```

生成可执行文件。

然后：

```
.\gateway.exe
```

直接运行 Gateway。

这样可以减少 go run 这一层对问题排查的干扰。

## 13.3 Windows Ctrl+C 的排查

Windows 控制台中的 Ctrl+C 会产生控制台中断事件，`signal.NotifyContext` 可以接管 Go 程序收到的中断信号。

但不能只根据 PowerShell 是否重新显示提示符，就判断 Gateway 是否已经退出。

需要检查实际进程。

查看 Gateway 进程：

```
Get-Process gateway -ErrorAction SilentlyContinue
```

查看监听 8080 的 TCP 连接：

```
Get-NetTCPConnection -LocalPort 8080 -State Listen -ErrorAction SilentlyContinue
```

查看 Go 和 Gateway 的进程关系：

```
Get-CimInstance Win32_Process |
    Where-Object {
        $_.Name -match '^(go|gateway)\.exe$'
    } |
    Select-Object ProcessId, ParentProcessId, Name, CommandLine
```

判断：

| 进程   | 8080 监听 | 说明                       |
| ------ | --------- | -------------------------- |
| 存在   | 存在      | Gateway 仍在监听           |
| 存在   | 不存在    | 可能正在停机或执行其他清理 |
| 不存在 | 不存在    | Gateway 已退出             |

日志：

```
Request Completed
Gateway shutdown sequence completed
Redis Close finished
```

说明停机流程已经执行到资源清理阶段。但要严格确认进程已经退出，仍然应该检查进程本身。

另外，项目中如果在 main 返回前打印：

```
log.Println("Gateway shutdown sequence completed")
```

而 Redis 关闭是通过 `defer` 执行的，那么这个日志会先于 Redis Close 日志出现。这是 Go 的正常 defer 执行顺序。

# 十四、项目中几个重要的并发安全知识

## 14.1 sync.Mutex 与 sync.RWMutex

`sync.Mutex`：互斥锁。

```
b.mu.Lock()
defer b.mu.Unlock()
```

同一时间只允许一个 Goroutine 持有锁。

`sync.RWMutex`：读写锁。

```
b.mu.RLock()
defer b.mu.RUnlock()
```

允许多个读操作并发执行，但写操作需要独占锁。

在负载均衡器中：

- `Next()` 获取快照时使用读锁。
- `ListTargets()` 查询实例时使用读锁。
- `AddTarget()` 添加实例时使用写锁。
- `RemoveTarget()` 删除实例时使用写锁。

## 14.2 atomic.Bool

```
healthy atomic.Bool
```

用于原子地读写健康状态。

读取：

```
candidate.healthy.Load()
```

写入：

```
item.healthy.Swap(healthy)
```

避免多个 Goroutine 同时访问健康状态时发生数据竞争。

## 14.3 atomic.Uint64

```
counter atomic.Uint64
```

用于原子计数。

```
counter.Add(1)
```

多个请求同时选择后端时，不需要额外使用 Mutex 保护计数器。

# 十五、常用命令与测试

## 15.1 格式化

```
gofmt -w .\gateway\main.go .\gateway\balancer.go
```

格式化指定文件。

## 15.2 运行 Gateway

```
go run ./gateway
```

或者：

```
go build -o .\gateway.exe .\gateway
.\gateway.exe
```

## 15.3 执行全部测试

```
go test ./...
```

## 15.4 竞争检测

```
go test -race ./gateway
```

用于检测并发数据竞争。

## 15.5 查询后端实例

```
Invoke-RestMethod `
    -Uri "http://127.0.0.1:8080/api/upstreams?service=/api/users/" `
    -Headers @{Authorization = "Bearer $($admin.access_token)"}
```

## 15.6 查询端口监听

```
Get-NetTCPConnection -LocalPort 8080 -State Listen
```

## 15.7 查询进程

```
Get-Process gateway -ErrorAction SilentlyContinue
```

# 十六、核心知识总结

| 知识点               | 核心作用                                |
| -------------------- | --------------------------------------- |
| http.Handler         | HTTP 处理器接口                         |
| http.HandlerFunc     | 将普通函数适配为 Handler                |
| ServeMux             | HTTP 路由分发                           |
| Middleware           | 在请求进入业务 Handler 前后执行通用逻辑 |
| ReverseProxy         | 将请求转发给后端                        |
| LoadBalancer         | 选择后端实例                            |
| atomic               | 原子操作，避免并发读写竞争              |
| Mutex                | 互斥锁                                  |
| RWMutex              | 读写锁                                  |
| Context              | 传递取消、超时和请求范围数据            |
| ctx.Done()           | 监听 Context 取消                       |
| defer cancel()       | 当前函数返回时释放 Context 相关资源     |
| Goroutine            | Go 的轻量级并发执行单元                 |
| WaitGroup            | 等待一批 Goroutine 完成                 |
| Channel              | Goroutine 之间通信和同步                |
| signal.NotifyContext | 将操作系统信号转换为 Context 取消       |
| ListenAndServe       | 启动 HTTP 服务                          |
| Shutdown             | 优雅关闭 HTTP Server                    |
| Close                | 强制关闭 HTTP Server                    |
| Redis                | 保存 Refresh Token 等数据               |
| JWT                  | 认证用户身份                            |
| RBAC                 | 根据角色控制操作权限                    |
| Control Plane        | 管理 Gateway 配置和实例                 |
| Data Plane           | 转发实际业务流量                        |

## 最重要的五组概念

**第一组：Context 与 WaitGroup**

Context 负责通知停止，WaitGroup 负责等待结束。

**第二组：Shutdown 与 Close**

Shutdown 等待正在处理的请求，Close 强制关闭连接。

**第三组：认证与授权**

认证确定用户身份，授权决定用户能执行什么操作。

**第四组：控制平面与数据平面**

控制平面管理实例，数据平面处理实际流量。

**第五组：CountDownLatch 与 CyclicBarrier**

CountDownLatch 等待任务完成，CyclicBarrier 等待参与者到齐。

# 十七、建议掌握的完整知识链路

复习时建议按照以下顺序：

1. HTTP Handler、HandlerFunc、ServeMux。
2. Middleware 的包装机制与请求执行顺序。
3. Context 的父子关系、取消传播和超时。
4. Goroutine、Channel、WaitGroup。
5. HTTP Server 的启动和生命周期。
6. Shutdown、Close 与优雅停机。
7. 负载均衡、反向代理和健康检查。
8. 控制平面与数据平面。
9. JWT、Redis Refresh Token 和 RBAC。
10. Mutex、RWMutex、Atomic 和并发安全。

**最终需要建立的整体认识：**

Go API Gateway 不只是一个 HTTP 转发程序，它同时涉及网络请求处理、并发调度、资源管理、身份认证、服务治理和生命周期管理。

理解这些组件之间的调用关系，比单独记住某个 Go API 更重要。