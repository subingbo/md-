# WebFlux / Reactor / Netty / CompletableFuture 完整学习笔记

------

# 一、为什么会有 WebFlux

传统 Spring MVC：

```
一个请求 ≈ 一个线程
```

例如：

```
请求进来
↓
线程处理
↓
等待数据库
↓
等待 HTTP
↓
返回结果
```

问题：

大量时间：

# 线程其实在等待 IO

例如：

- 等数据库
- 等 Redis
- 等 OpenAI
- 等网络

线程被浪费。

------

# 二、WebFlux 的核心思想

# “等待 IO 时不要占线程”

即：

```
发起 IO
↓
线程立刻释放
↓
有结果再通知
```

------

# 三、WebFlux 的核心组成

```
Spring WebFlux
    ↓
Reactor
    ↓
Reactor Netty
    ↓
Netty
    ↓
Selector / epoll
```

------

# 四、各层职责

------

## 1. WebFlux

负责：

- Controller
- HTTP
- SSE
- WebSocket
- 路由
- 参数绑定

例如：

```
@GetMapping("/chat")
public Flux<String> chat() {
    ...
}
```

------

## 2. Reactor

负责：

# 响应式数据流

核心：

- Mono
- Flux
- map
- flatMap
- zip
- Scheduler

------

## 3. Netty

负责：

# 高性能网络通信

核心：

- EventLoop
- Channel
- Pipeline
- Selector
- epoll

------

# 五、Mono 和 Flux

------

## Mono

表示：

# 未来一个结果

类似：

```
CompletableFuture<User>
```

例如：

```
Mono<String>
```

------

## Flux

表示：

# 持续事件流

例如：

```
token1
token2
token3
```

------

# 六、Mono.just() 为什么存在

例如：

```
Mono.just("hello")
```

作用：

# 把普通对象包装成响应式对象

------

## Flux.just()

```
Flux.just(1,2,3)
```

表示：

# 创建数据流

------

# 七、Reactor 是 Lazy 的

例如：

```
Mono.just("hello")
    .map(String::toUpperCase);
```

这里只是：

# 定义数据流

不会执行。

------

## 必须：

```
subscribe()
```

才真正执行。

------

# 八、WebFlux 为什么没写 subscribe

因为：

# Spring 自动 subscribe

例如：

```
@GetMapping("/hello")
public Mono<String> hello() {
    return Mono.just("hello");
}
```

Spring 内部类似：

```
mono.subscribe(v -> response.write(v));
```

------

# 九、为什么不要手动 subscribe

错误：

```
mono.subscribe();
return mono;
```

会：

# 执行两次

------

# 十、什么时候手动 subscribe

真正需要：

- MQ 消费
- 后台任务
- Event Stream
- 定时任务

------

# 十一、WebClient 是什么

WebFlux 的：

# 响应式 HTTP 客户端

------

## MVC 时代

```
RestTemplate
```

------

## WebFlux 时代

```
WebClient
```

------

# 十二、WebClient 示例

```
webClient.get()
    .retrieve()
    .bodyToMono(String.class);
```

特点：

# 非阻塞

------

# 十三、真正非阻塞发生了什么

```
发 HTTP
↓
线程立刻释放
↓
epoll 监听 socket
↓
数据回来通知
```

------

# 十四、Selector / epoll

真正高并发核心。

作用：

# 一个线程监听大量 socket

------

## Netty EventLoop 本质

```
while(true) {

    selector.select();

    处理有事件的 channel
}
```

------

# 十五、Channel 是什么

# TCP连接的高级封装

------

## 一个 TCP连接

对应：

```
一个 Channel
```

------

## Channel 能：

- write
- read
- flush
- close

------

# 十六、EventLoop 是什么

# 处理 Channel 事件的线程

------

## 一个 EventLoop

管理：

```
多个 Channel
```

------

# 十七、为什么高并发

传统：

```
一个请求一个线程
```

------

Netty：

```
一个线程处理大量连接事件
```

------

# 十八、真正对应关系

```
TCP连接
   ↓
Channel
   ↓
HTTP Request
   ↓
Controller
   ↓
Flux/Mono
   ↓
Subscriber
   ↓
Response
   ↓
channel.write()
```

------

# 十九、Controller 和 Channel 的关系

错误理解：

```
Controller ↔ Channel
```

------

正确：

```
Channel
   ↓
HTTP Request
   ↓
URL路由
   ↓
Controller
```

------

# 二十、为什么不会串请求

因为：

每个请求：

都有：

- Request
- Response
- Subscriber

------

## 真正绑定的是：

```
Request ↔ Response
```

不是：

```
Controller ↔ Channel
```

------

# 二十一、SSE 本质

不是：

# 不断发请求

而是：

# 一个长期不断开的 HTTP Response

------

## Flux

持续：

```
onNext(token)
```

------

## Response

持续：

```
write(token)
```

------

# 二十二、Flux 为什么适合 AI

因为：

AI 本质：

# token stream

例如：

```
hello
hello world
hello world !
```

------

## Flux 天然支持：

```
onNext()
onNext()
onNext()
```

------

# 二十三、CompletableFuture vs Mono

------

## CompletableFuture

表示：

# 一个未来结果

------

## Mono

本质：

# 增强版 CompletableFuture

支持：

- Reactive Stream
- cancel
- Scheduler
- backpressure

------

# 二十四、Flux 是 Future 做不到的

Future：

```
只能完成一次
```

------

Flux：

```
持续事件流
```

------

# 二十五、boundedElastic 是什么

很多库：

# 本身是阻塞的

例如：

- JDBC
- 文件 IO
- Thread.sleep
- 老 SDK

------

## 问题

会阻塞：

# Netty EventLoop

------

## 解决

```
.subscribeOn(Schedulers.boundedElastic())
```

作用：

# 把阻塞任务扔到专门线程池

------

# 二十六、为什么需要 boundedElastic

因为：

# WebFlux 不能把阻塞 IO magically 变成非阻塞

只能：

# 隔离阻塞

------

# 二十七、真正理想情况

```
Netty
↓
WebFlux
↓
WebClient
↓
R2DBC
↓
Reactive Redis
```

全链路非阻塞。

------

# 二十八、WebFlux 到底快不快

------

## 不会让：

```
单个 IO 更快
```

例如：

```
OpenAI 5 秒
```

仍然：

# 5 秒

------

## 但：

会提升：

- 吞吐量
- 并发能力
- 资源利用率

------

# 二十九、真正能提升响应速度的

# 并行化

------

## 串行

```
getUser();
getOrder();
getProduct();
```

------

## Reactor 并行

```
Mono.zip(
    getUser(),
    getOrder(),
    getProduct()
)
```

------

# 三十、Reactor 并发不一定靠线程

例如：

两个 WebClient 请求：

```
同时发起
```

可能：

# 一个 EventLoop线程就够

因为：

# 非阻塞 IO

------

# 三十一、WebFlux 可以并行吗

当然可以。

例如：

```
.parallel()
.runOn(Schedulers.parallel())
```

------

# 三十二、为什么说 WebFlux 更适合 IO

因为：

CPU密集：

```
线程不会等待
```

------

WebFlux 最大价值：

# “等待 IO 时不占线程”

------

# 三十三、CPU密集优化重点

真正重要：

- CPU核心数
- 算法
- SIMD
- GPU
- cache locality

------

# 三十四、什么时候适合 WebFlux

特别适合：

- AI Chat
- SSE
- WebSocket
- API Gateway
- 长连接
- 高IO系统

------

# 三十五、什么时候 MVC 足够

例如：

- CRUD
- 管理系统
- ERP
- OA

------

# 三十六、真正性能优化顺序

------

## 1. 减少 IO

------

## 2. SQL优化

------

## 3. Redis缓存

------

## 4. 批量化

------

## 5. 并行化

------

## 6. 线程模型

最后才是。

------

# 三十七、CompletableFuture 和 WebFlux 的关系

不是：

# 二选一

现实：

# 经常混用

------

## Future → Mono

```
Mono.fromFuture(future)
```

------

## Mono → Future

```
mono.toFuture()
```

------

# 三十八、真正核心区别

------

## CompletableFuture

重点：

# “异步任务”

------

## Reactor

重点：

# “异步事件流”

------

# 三十九、真正应该建立的思维

不要再想：

```
函数执行到哪
```

而是：

# “事件流传播到哪”

------

# 四十、最终一句话总结

## WebFlux/Reactor/Netty 的本质：

# “基于 EventLoop + Selector + Reactive Stream 的非阻塞事件驱动系统”

核心目标：

- 少线程
- 高IO并发
- 长连接
- 流式数据
- 高吞吐

而不是：

# “让单个 IO magically 更快”。