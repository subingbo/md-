# Spring Cloud Gateway / Reactor Netty 半死状态问题笔记

## 一、问题现象

系统表现：

- `docker ps` 显示容器正常
- `gateway-server` 容器没有退出
- 部分接口正常
- 部分接口卡死
- `plan-service` 完全没有请求日志
- 重启 gateway 后恢复正常

典型日志：

```text
RejectedExecutionException: event executor terminated
Event loop shut down?
Force-closing a channel whose registration task was not accepted
```



------

# 二、核心原因

本质：

```text
Gateway 的 Reactor Netty EventLoop / ConnectionPool 出现异常
```

不是：

```text
业务 Controller 异常
```

也不是：

```text
Docker 容器退出
```

而是：

```text
Gateway 内部 IO Runtime 半死
```

------

# 三、Gateway 底层架构

Spring Cloud Gateway：

```text
Gateway
  ↓
Spring WebFlux
  ↓
Reactor Netty
  ↓
Netty EventLoop
```

核心：

```text
EventLoop = IO事件循环线程
```

负责：

- TCP连接
- HTTP请求
- 请求转发
- 响应写回
- socket事件

------

# 四、什么是 EventLoop

传统 Tomcat：

```text
一个请求一个线程
```

WebFlux / Netty：

```text
少量 EventLoop 线程处理大量连接
```

类似：

```java
while(true){
    accept();
    read();
    write();
}
```

因此：

```text
EventLoop 非常重要
```

------

# 五、什么是 event executor terminated

日志：

```text
RejectedExecutionException: event executor terminated
```

意思：

```text
Netty 的 EventLoop线程池已经关闭
```

类似：

```java
threadPool.shutdown()
```

但：

```text
仍然有新请求进来
```

于是：

```text
新连接无法注册
```

所以：

```text
RejectedExecutionException
```

------

# 六、为什么部分接口正常，部分接口异常

因为：

```text
Gateway 是多 EventLoop + 多连接池
```

不是：

```text
单线程
```

例如：

```text
EventLoop-1 正常
EventLoop-2 正常
EventLoop-3 已terminated
```

于是：

| 请求 | 结果 |
| ---- | ---- |
| A    | 正常 |
| B    | 卡死 |
| C    | 正常 |

------

还可能：

```text
只有某个 downstream 连接池坏了
```

例如：

```text
plan-service连接池异常
stat-service正常
```

于是：

| 服务           | 状态 |
| -------------- | ---- |
| `/api/stat/**` | 正常 |
| `/api/plan/**` | 异常 |

------

# 七、为什么 plan-service 没日志

因为：

```text
请求根本没到达 plan-service
```

请求链：

```text
TCP连接
↓
Netty EventLoop
↓
Gateway Route
↓
plan-service
```

但现在：

```text
死在 EventLoop
```

所以：

- Gateway 业务日志没有
- plan-service 没日志
- Controller 没执行

------

# 八、为什么 Docker 看起来正常

Docker 只关心：

```text
PID1 是否退出
```

但：

```text
JVM 还活着
```

只是：

```text
Netty EventLoop 已损坏
```

因此：

| 层级             | 状态     |
| ---------------- | -------- |
| Docker容器       | 正常     |
| JVM进程          | 正常     |
| Spring Context   | 正常     |
| Netty IO Runtime | 部分损坏 |

这叫：

```text
Partial Failure（部分失效）
```

或者：

```text
半死状态
```

------

# 九、为什么重启后恢复

因为损坏的是：

```text
运行时状态
```

包括：

- EventLoop
- ConnectionPool
- selector
- direct memory
- Reactor state

重启后：

```text
全部重新初始化
```

所以恢复。

------

# 十、为什么会出现

## 1. 下游连接池缓存坏连接（最常见）

流程：

```text
gateway
  ↓
连接池
  ↓
plan-service
```

如果：

- plan-service 重启
- Docker 网络变化
- idle timeout
- connection reset

则：

```text
连接已失效
```

但：

```text
gateway 仍复用这个连接
```

导致：

```text
连接池污染
```

------

## 2. WebFlux 阻塞 EventLoop

危险操作：

```java
Thread.sleep()
同步数据库
同步Redis
同步HTTP
```

如果运行在：

```text
EventLoop线程
```

则：

```text
整个 IO 循环被阻塞
```

------

## 3. Direct Memory 耗尽

Netty 大量使用：

```text
DirectByteBuffer
```

不是 JVM Heap。

所以：

```text
heap正常
≠
Netty正常
```

------

## 4. Linux epoll/socket问题

例如：

- fd过多
- TIME_WAIT过多
- epoll异常

------

## 5. Reactor Netty Bug

在：

- JDK21
- Spring Boot 3.x
- Reactor Netty

组合下偶尔存在。

------

# 十一、为什么 WebFlux 不能阻塞

传统 MVC：

```text
一个请求一个线程
```

阻塞：

```text
只影响当前请求
```

------

WebFlux：

```text
一个 EventLoop 负责大量连接
```

如果：

```java
Thread.sleep(10s)
```

则：

```text
整个 EventLoop 卡10秒
```

影响：

```text
该线程负责的所有连接
```

------

错误示例：

```java
@GetMapping
public String test() throws Exception{
    Thread.sleep(10000);
    return "ok";
}
```

------

正确：

```java
@GetMapping
public Mono<String> test(){
    return Mono.delay(Duration.ofSeconds(10))
               .thenReturn("ok");
}
```

------

# 十二、同步数据库为什么危险

错误：

```java
jdbcTemplate.query(...)
```

因为：

```text
会阻塞 EventLoop
```

------

正确：

```java
Mono.fromCallable(() -> jdbc.query(...))
    .subscribeOn(Schedulers.boundedElastic())
```

意思：

```text
切换到专门阻塞线程池
```

避免：

```text
阻塞 EventLoop
```

------

# 十三、为什么会出现大量奇怪日志

因为：

```text
Netty 是异步事件驱动
```

当 EventLoop 崩后：

每个新请求都会：

```text
accept TCP
↓
尝试注册channel
↓
发现 EventLoop 已关闭
↓
抛异常
```

于是：

```text
不断刷底层异常日志
```

这些日志来自：

- Netty
- Reactor
- epoll
- DefaultPromise

不是业务代码。

------

# 十四、为什么异常处理器接不到

因为：

```text
请求还没进入 Spring
```

死在：

```text
Netty IO层
```

因此：

- `@ControllerAdvice`
- `GlobalExceptionHandler`
- Filter

可能都不会执行。

------

# 十五、如何测试复现

## 1. 测试坏连接池

持续请求：

```bash
while true
do
  curl http://gateway/xxx
  sleep 0.2
done
```

然后：

```bash
docker restart plan-service
```

观察 gateway。

------

## 2. 压测

使用：

wrk

```bash
wrk -t8 -c500 -d60s http://gateway/xxx
```

------

## 3. 模拟阻塞

```java
Thread.sleep(60000)
```

观察：

- timeout
- event loop
- gateway状态

------

## 4. 查看 DirectMemory

```bash
jcmd PID VM.native_memory summary
```

------

## 5. 打开 Reactor DEBUG

```yaml
logging:
  level:
    reactor.netty: DEBUG
```

------

# 十六、如何排查

## 查看 OOM

```bash
dmesg -T | grep -Ei 'oom|killed process'
```

------

## 查看 Docker 是否 OOMKilled

```bash
docker inspect gateway-server | grep -i oom
```

------

## 查看 JVM 参数

重点：

```text
-Xmx
-XX:MaxDirectMemorySize
```

------

## 查看连接情况

```bash
ss -s
```

------

## 查看 fd

```bash
lsof -p PID | wc -l
```

------

# 十七、如何避免

## 1. 限制连接池生命周期

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          max-idle-time: 20s
          max-life-time: 60s
```

------

## 2. 配置 timeout

```yaml
connect-timeout: 3000
response-timeout: 5s
```

------

## 3. 禁止阻塞 EventLoop

严禁：

```java
Thread.sleep()
同步IO
同步HTTP
同步数据库
```

------

## 4. 使用 boundedElastic

```java
.subscribeOn(Schedulers.boundedElastic())
```

------

## 5. 健康检查

Docker：

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
```

------

## 6. 自动重启

```yaml
restart: unless-stopped
```

------

# 十八、最终总结

本次问题本质：

```text
Gateway Reactor Netty Runtime 部分损坏
```

特点：

- Docker 正常
- JVM 正常
- 部分接口正常
- 部分接口失效
- 请求无法进入 Controller
- 重启恢复

核心原因：

```text
EventLoop / ConnectionPool / Reactor Runtime 异常
```

而不是：

```text
普通业务代码异常
```