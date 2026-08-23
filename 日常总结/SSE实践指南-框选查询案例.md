# SSE 实践指南 —— 以「框选查询」为例

> 写给想学后端的学生。本文用 `analysis-service` 框选查询的真实代码，讲清楚：SSE 是什么、为什么要用它、项目里每一行关键代码在做什么、实战中踩过的坑，最后是我的学习建议。建议配合源码一起读：`gzmap-parent/analysis-service/analysis-service-biz/src/main/java/com/guodi/analysis/biz/`。

> ⚠️ **版本说明**：本文基于 2026-08 初版「逐图层推送 layer 事件」实现编写。现实现已演进为：**边算边推**——每算完一个图层立即推 该图层产出的 `indicator` 事件（有指标图层，图层内按 指标与模块对应关系.md 顺序）+ 无指标图层 `layer` 事件，全部图层算完后追加派生预警指标（4-2）并推 `done`。SSE 传输层、Sink 回调模式、done/error 语义均不变，仅回调粒度从「逐图层」变为「逐指标条目」；下文的 `sink.onLayer` 对应新版 `sink.onIndicator` / `sink.onLayer(无指标图层)`。协议细节以 [框选查询接口文档.md](./框选查询接口文档.md) 与 [框选查询-前端对接指南.md](./框选查询-前端对接指南.md) 为准。

---

## 1. 从业务问题开始：为什么要学 SSE

假设你负责一个地图系统。用户在地图上画一个框，后端要查 21 个图层的数据（道路、公交站、生态红线……），每个图层查数据库要花几秒，全部算完要 40 秒。

**第一个版本**：前端发一个请求，后端全算完，一次性返回。

前端用户看着 40 秒白屏，不知道是不是挂了。这是很差的体验。

**需求**：边算边回，算完一层推一层，前端能看到进度（"第 3/21 层"）。算完给个完成标记。

这个需求就是 SSE 的典型场景。先记住结论，后面细讲：

| 技术 | 特点 | 适合 |
| --- | --- | --- |
| **轮询** | 前端定时发普通请求 | 简单但浪费请求，延迟高 |
| **长轮询** | 请求挂住，有数据才返回 | 能省请求，但实现复杂 |
| **SSE** | **HTTP 单向长连接，服务端持续推送** | **服务端 → 客户端的单向数据流（本场景）** |
| **WebSocket** | 双向长连接 | 双向通信、实时游戏、聊天 |

本项目需要的是"服务端算一层推一层"的**单向推送**，SSE 就是为此设计的——基于普通 HTTP（好穿透网关/代理），不需要额外协议。这行逻辑："用最简单的协议解决你的问题"是后端的重要思维方式。

---

## 2. SSE 协议长什么样（先看懂格式）

SSE 本质是：HTTP 响应不关闭，服务端按固定格式持续往连接里写文本。协议格式：

```
event: layer
data: {"layerName":"道路线网","count":69028}

event: done
data: {"completed":true,"totalFeatures":95525}
```

- 每帧：`event:` 行（事件名）+ `data:` 行（载荷，通常是 JSON）+ **空行**结束。
- 帧之间用空行分隔；客户端按空行切分帧。
- `Content-Type` 必须是 `text/event-stream`，服务端不关闭连接，持续写。

用 `curl -N` 能看到原始流（`-N` 禁用缓冲，逐事件输出）：

```bash
curl -N http://localhost:8108/api/analysis/box-selection/query/stream \
  -H "Content-Type: application/json" -d '{"bbox":[...]}'
```

---

## 3. 本项目代码走读（Spring MVC SseEmitter）

### 3.1 完整链路

```
浏览器 POST 请求
  → BoxSelectionController.queryStream()         ① 校验 + 创建 SseEmitter
  → sseExecutor.execute(runStream)               ② 丢给专用线程池（不阻塞 Tomcat 线程）
  → BoxSelectionServiceImpl.queryByBoxStream()   ③ 逐图层查库算指标
  → sink.onLayer() 回调                         ④ 翻译成 SSE 帧
  → emitter.send(...)                            ⑤ 写进连接，前端立刻收到
```

### 3.2 Controller 层（入口）

```java
@PostMapping(value = "/query/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter queryStream(@RequestBody BoxSelectionQueryDTO query) {
    boxSelectionService.validateQuery(query);            // ① 校验必须在建连接之前！

    SseEmitter emitter = new SseEmitter(300_000L);       // ② 超时：5 分钟
    BoxSelectionStreamSink sink = buildSink(emitter);    // ③ 回调翻译层
    sseExecutor.execute(() -> runStream(query, emitter, sink));  // ④ 异步跑

    emitter.onTimeout(() -> emitter.complete());         // ⑤ 超时兜底
    return emitter;
}
```

五个要点：

1. **`produces = TEXT_EVENT_STREAM_VALUE`**：告诉 Spring 这个接口返回 SSE 流。
2. **返回类型必须显式写 `SseEmitter`**：Spring 按方法声明类型找对应的返回值处理器（`SseEmitterReturnValueHandler`），写 `Object` 或 `R<SseEmitter>` 都会失效。
3. **校验放行**：参数错了要返回普通 JSON 错误（422），而不是建一条流再推错误——校验放最前面，失败直接抛异常走 `@ExceptionHandler`。
4. **异步执行**：`sseExecutor.execute(...)` 让耗时的计算不占 Tomcat 线程。Tomcat 线程是稀缺资源（默认 200 个），占满后整个服务别的接口都卡死。这是 Web 应用的基本功：**慢活一定要丢线程池**。
5. **`300_000L` 超时**：SseEmitter 必须显式给超时。给 `0L` 永不超时会泄漏线程和连接；不给用默认 30 秒，计算没完就断了。**超时是绝对计时，不因 send 重置**——所以 300s 是"连接最长存活 5 分钟"，不是"5 分钟没数据就断"。

### 3.3 线程池（专门为流式请求配的）

```java
@Bean(destroyMethod = "shutdown")
public ExecutorService sseExecutor() {
    return new ThreadPoolExecutor(
            4, 8,                          // 核心 4 线程，最多 8
            60L, TimeUnit.SECONDS,         // 空闲回收时间
            new LinkedBlockingQueue<>(1000),
            new ThreadFactoryBuilder().setNamePrefix("box-sse-").build(),
            new ThreadPoolExecutor.CallerRunsPolicy());
}
```

- 每个流式请求占 1 个线程串行推图层，所以核心 4 线程 ≈ 能同时服务 4 个流。
- 线程名 `box-sse-` 前缀很重要：出问题看日志时能分清是哪个池的线程。这是生产意识。
- `CallerRunsPolicy`：队列满 1000 时，新任务在**请求线程**里同步执行——不拒绝请求，只是降级占 Tomcat 线程。宁可降级也不丢请求，这是可用性设计。

> 💡 学习点：`ThreadPoolExecutor` 的 4 个参数（核心数、最大数、空闲回收、队列）+ 拒绝策略，是面试必考也是生产必用的。试着回答：为什么核心 4 最大 8？为什么用有界队列？`CallerRunsPolicy` 和 `AbortPolicy` 的区别？

### 3.4 回调接口（Sink 模式）—— 本项目最值得学的设计

Service 层不直接碰 SseEmitter，而是依赖一个回调接口：

```java
public interface BoxSelectionStreamSink {
    void onLayer(BoxLayerResultVO layer, int layerIndex, int totalLayers);
    void onError(String layerName, String message);
    void onDone(int totalLayers, long totalFeatures, long elapsedMs, List<FailedLayer> failedLayers);
}
```

Service 只负责算：每算完一层调 `sink.onLayer(...)`，全算完调 `sink.onDone(...)`。至于回调是写 SSE 帧、写日志、还是塞进内存列表——Service 完全不关心。

**好处**：
- **可测试**：测试里用一个收集器实现 Sink，就能在不启动 Web 容器的情况下单测整个流式逻辑（本项目就是这么做的，`StreamCollector`）。
- **解耦**：以后想换成 WebSocket 或 WebFlux，Service 一行不改，只换 Sink 的实现。
- **责任清晰**：Service 管"算"，Controller 管"传输"。

这是设计模式里的"回调/观察者"思想。大项目里到处是这个模式（Spring 的事件机制、MyBatis 的拦截器都是）。

### 3.5 事件翻译层（Controller 里的匿名 Sink 实现）

```java
public void onLayer(BoxLayerResultVO layer, int layerIndex, int totalLayers) {
    BoxStreamLayerEventVO evt = new BoxStreamLayerEventVO();
    BeanUtils.copyProperties(layer, evt);   // 同名字段自动拷贝
    evt.setLayerIndex(layerIndex);
    evt.setTotalLayers(totalLayers);
    evt.setCompleted(false);
    send(emitter, "layer", evt);
}

private void send(SseEmitter emitter, String name, Object payload) {
    try {
        emitter.send(SseEmitter.event().name(name)
                .data(objectMapper.writeValueAsString(payload), MediaType.APPLICATION_JSON));
    } catch (IOException e) {
        throw new SseClientDisconnectedException(e);   // 客户端断了 → 自定义异常标记
    }
}
```

注意两件事：

1. **手动序列化 JSON**：先 `objectMapper.writeValueAsString` 再按 `APPLICATION_JSON` 发送。直接传对象会触发 Spring 的 converter 竞争（String 和 Jackson 都想处理），结果不稳定。**"自己控住序列化"是 SSE 实战第一条经验**。
2. **`IOException` 就是"客户端断开"信号**：浏览器关页面/断网后，连接对服务端变成半开状态，写进去就抛异常。这里包成 `SseClientDisconnectedException`，让顶层能区分"客户端跑了"（正常停止，记日志）和"服务端真出错了"（推 error 事件）。

### 3.6 Service 层：失败隔离 + 断连停止

```java
for (int i = 0; i < totalLayers; i++) {
    try {
        vo = computeOneLayer(def, ...);      // 只包"计算"
    } catch (Exception e) {
        failedLayers.add(...);
        sink.onError(def.layerName(), e.getMessage());  // 单个图层失败，不中断
        continue;
    }
    sink.onLayer(vo, i, totalLayers);        // 注意：在 try 外！
}
sink.onDone(totalLayers, totalFeatures, elapsed, failedLayers);
```

两个细节是踩坑换来的：

- **失败隔离**：第 5 层 SQL 挂了，不能整个查询挂掉——推 `error` 事件，继续算第 6 层，最后 `done` 里带失败清单。前端展示"部分成功"。
- **`sink.onLayer` 在 try 外**：客户端断开时 `onLayer` 抛异常，直接冲出循环，**剩余图层不再计算**。如果放 try 里，会被当成"图层失败"继续把 21 层全算完——白算，还一直往断开的连接写。这一行位置的差异，实战中很关键。

### 3.7 完整事件协议（前端视角）

| 事件 | 时机 | 关键字段 |
| --- | --- | --- |
| `layer` | 每算完一层 | `layerName`/`count`/`features`/`layerIndex`/`totalLayers`/`completed:false` |
| `done` | 全部处理完毕（**必达**） | `completed:true`/`totalFeatures`/`elapsedMs`/`failedLayers` |
| `error` | 单层失败（不中断） | `completed:false`/`layerName`/`message` |

`done` 事件就是需求里的"全部计算完成的标记字段"，前端收到它就知道流结束了。

---

## 4. 实战踩坑记录（这些就是经验值）

### 坑 1：EventSource 用不了 —— SSE 前端没你想的那么简单

浏览器原生 `EventSource` **不支持 POST**，也不能带自定义 Header。本项目要 POST + JWT 认证，所以前端必须用 `fetch` + `ReadableStream` 手动解析 SSE 帧。代码见《框选查询-前端对接指南.md》的 `sseParser`。

> 💡 学习点：EventSource 很"玩具"，生产里经常要自己解析。读一读对接指南里的 sseParser 实现——它兼容 `\r\n`、处理 chunk 切分到一半的 JSON、多行 `data:`，这些是流式解析的经典问题。

### 坑 2：中文乱码

SSE 流输出中文乱码，原因是 Spring Boot 3 默认 `server.servlet.encoding` 配置不生效于流式。修复：`application.yml` 显式配 UTF-8：

```yaml
server:
  servlet:
    encoding:
      enabled: true
      charset: UTF-8
      force: true
```

### 坑 3：异步 dispatch 时拦截器重执行（框架级 Bug，本项目真实遇到）

SSE 接口在非 prod 环境 500 崩溃，排查到 `yudao-framework` 的 `ApiAccessLogInterceptor`：

- **现象**：DispatcherServlet 在 ASYNC dispatch（第二次分发，取流结果）时会**重新执行**拦截器链的 `preHandle`；
- **根因**：访问日志拦截器 `preHandle` 里读请求体（`getInputStream`），但 body 已被 `@RequestBody` 消费过 → `IllegalStateException`；
- **修复**：`if (request.getDispatcherType() == DispatcherType.ASYNC) return true;` 异步分发直接放行。

> 💡 学习点：这个 bug 值得反复琢磨——它让你看到 Spring MVC 请求的完整生命周期：`preHandle → controller（提交异步任务）→ 返回 → ASYNC dispatch（真正执行异步任务）→ postHandle → afterCompletion`。拦截器在两次 dispatch 中都会跑。理解了这条链路，很多"诡异的二次执行"问题都能定位。

### 坑 4：客户端断开后的线程清理

断连后 worker 线程如果还在算，就是白耗 CPU。本项目用"onLayer 抛异常冲出循环"快速停止。测试里专门有单测验证：`verify(mapper, times(3))`——第 3 层断开后，第 4 层起不再查询。

### 坑 5：超时语义

见 §3.2 要点 5。记住：`0L` 泄漏、默认 30s 断流、绝对计时不重置。

---

## 5. 本次任务你用到的其他新技术盘点

| 技术 | 在项目里怎么用的 | 建议掌握程度 |
| --- | --- | --- |
| **SseEmitter** | 流式接口核心 | ⭐⭐⭐ 会写会用 |
| **回调接口（Sink 模式）** | Service/Controller 解耦，可测试 | ⭐⭐⭐ 思想比语法重要 |
| **ThreadPoolExecutor** | 专用线程池 + 拒绝策略 | ⭐⭐⭐ 面试必考 |
| **BeanUtils.copyProperties** | 老 VO → 流式 VO 同名字段拷贝 | ⭐ 会用即可 |
| **Java record**（Java 16+） | `LayerDef`/`FailedLayer` 不可变数据载体 | ⭐⭐ 高频新语法 |
| **@PostConstruct + 注册表** | 21 个图层配置式注册 | ⭐⭐ 配置化思想 |
| **@DynamicPropertySource** | 测试里覆盖 Nacos 远端配置 | ⭐⭐⭐ 测试实战神器 |
| **MockMvc 异步测试** | `getAsyncResult` + `asyncDispatch` | ⭐⭐ 面试加分项 |
| **函数式接口 + lambda** | `LayerQuery`、注册表查询闭包 | ⭐⭐⭐ 现代 Java 基础 |
| **Docker Compose 部署** | 公司服务器容器化 | ⭐⭐ 后端必备运维基础 |

---

## 6. 给大三学生的建议（怎么从这份代码里学到东西）

### 6.1 本次任务的学习路径（按推荐顺序）

1. **先看懂协议**：用 curl 亲手请求一次 `/query/stream`，盯着原始输出看 `event:`/`data:`/空行。协议是理解一切的基础。
2. **读懂链路**：按 §3.1 的链路图，从 Controller 跟到 Service 再到 Sink，每一层问"这层在做什么、为什么需要这层"。
3. **重点啃两个设计**：Sink 回调模式（解耦 + 可测试）和"onLayer 在 try 外"（错误边界设计）。这两个是笔面试能讲出来的东西。
4. **动手改造**：试试给项目加一个图层（开发文档 §9 有标准流程）——亲手跑通全链路比读十遍有效。
5. **写测试**：把 `StreamCollector` 看明白，理解"回调接口让 Service 层可以脱离 Web 容器单测"。这是你实习/校招时和多数人拉开差距的能力。

### 6.2 大二升大三这个阶段，我建议你重点做这些事

**Java 本身**：
- 把《Java 核心技术》或《Effective Java》读掉前 1/3（对象、异常、泛型、并发基础）。本项目里的 lambda、record、函数式接口就是"现代 Java"的日常。
- 并发必须掌握到：`synchronized` vs `Lock`、线程池参数与拒绝策略、`CompletableFuture`。本项目线程池就是活的教材。

**框架（Spring Boot）**：
- 不要只看 CRUD 教程。跟着本项目学三件事：**请求生命周期**（拦截器/过滤器/异常处理器顺序）、**依赖注入**（`@Resource` vs `@Autowired`、`@PostConstruct`）、**配置体系**（`application.yml` + Nacos + `@DynamicPropertySource`）。
- 强烈建议读 Spring 官方文档的《Web on Servlet Stack》一章——SseEmitter、异步请求、拦截器都讲得比任何博客清楚。

**工程能力（多数大三学生的短板）**：
- **写测试**：本项目测试覆盖"条数、时间、成败"三类断言，这是工业级测试的思路（不是只测"能不能跑"）。试着给别的功能补测试。
- **写文档**：你现在看的这几份文档（接口文档/开发文档/前端指南）就是任务产物的三种视角。学会把自己做的功能用文档讲清楚。
- **看日志定位问题**：拦截器那个 bug 的排查过程（从 500 日志 → 看异常栈 → 找到读 body 的拦截器 → 理解 ASYNC dispatch 语义）就是标准排查流程。多遇到、多拆。

**简历与面试**：
- 把这个项目写进简历时，别写"实现了一个查询接口"，要写**"基于 SSE 实现逐图层流式推送，通过 Sink 回调解耦 + 失败隔离保证单图层故障不中断，实测 21 层 40s 降至首层 5s 内可见"**——结果导向 + 技术名词 + 数据。
- 面试被问"SSE 和 WebSocket 区别"：讲清楚"单向 vs 双向、基于 HTTP vs 独立协议、自动重连 vs 手动实现"，再补一句"本项目因为只需服务端推数据、又要穿网关，所以选 SSE"——有项目支撑的回答和背八股的完全不同。

### 6.3 接下来半年可以做的练手项目（三选一）

1. **把本项目的流式实现用 WebFlux 重写一遍**——对比阻塞/响应式两种模型，面试话题拉满。
2. **做一个 WebSocket 在线编辑器/聊天室**——体会双向通信的场景差异，和 SSE 形成对照。
3. **给本项目加一个"并发计算图层"的优化**——理解串行→并行的并发控制难点（顺序、对账、限流），这是真实系统优化题。

---

## 7. 参考资料

- Spring 官方文档：Web on Servlet Stack（SseEmitter / 异步请求章节）
- MDN：Server-Sent Events（协议规范 + EventSource API）
- 《Effective Java》第 3 版（第 5 章泛型、第 11 章并发）
- 本项目文档：接口文档（协议细节）、前端对接指南（前端消费 + sseParser）、开发文档（架构与踩坑）
