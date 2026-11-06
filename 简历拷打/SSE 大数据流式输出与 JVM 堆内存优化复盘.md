# SSE 大数据流式输出与 JVM 堆内存优化复盘

> 主题：Spring MVC `SseEmitter`、Jackson 序列化、Java 21 `String` 内部存储、`byte[]` / `char[]`、Heap Dump 与 JFR  
> 记录依据：2026-10-19 峰值对照、2026-11-05 堆快照对照、2026-11-06 JFR 分配栈对照的实测数据与排查记录。  
> 阅读原则：本文把**实测事实、机制解释**分开写。凡是没有通过实际 Spring Converter、JFR 分配栈或 MAT 引用链确认的内容，明确标注为推测。

---

## 1. 先记住结论

这次优化的核心不是“取消 UTF-8 编码”，也不是“让 Java 不再使用 `byte[]`”，而是：

**原来先把一个完整 SSE 事件对象序列化成完整 JSON `String`，再把这个 `String` 交给 Spring 写入响应；现在直接把事件对象交给 `SseEmitter`，由 Spring/Jackson 在写响应时完成 JSON 序列化和 UTF-8 输出。**

原路径可能存在多层完整数据表示：

```text
业务对象
  |
  v
Jackson 字符缓冲区（可能有 char[]）
  |
  v
完整 JSON String（Java 21 String 内部有 byte[]）
  |
  v
Spring 对 String 做字符集编码（可能再产生 byte[]）
  |
  v
HTTP/SSE 输出流
```

新路径则是：

```text
业务对象
  |
  v
Spring 选择 Jackson 消息转换器
  |
  v
Jackson JSON Generator 序列化并编码
  |
  v
输出缓冲区
  |
  v
HTTP/SSE 输出流
```

两条路径都需要把响应内容编码为网络字节。新路径省掉的是**先构造完整 JSON String 这一中间物化步骤**，以及旧路径可能伴随的完整 String 编码副本和相关临时数组分配。

目前的实验已经证明：

1. 2026-11-05 堆快照：`byte[]` 从 307.5 MB 降至 67.7 MB，`char[]` 从 110.1 MB 降至 56.4 MB，合计减少 293.5 MB（快照时堆用量差 315 MB）。
2. 2026-11-06 JFR 分配栈对照（逐次分配事件，非抽样）：同一请求下，旧路径 `byte[]`+`char[]` 域外分配合计 905.7 MB，新路径 110.2 MB，差额约 795 MB，并且能精确落到三条调用栈上——Spring `StringHttpMessageConverter` 的二次 UTF-8 编码（约 293 MB）、Jackson `TextBuffer.contentsAsString` 拼 String（约 364 MB）、Jackson `TextBuffer` 字符缓冲（约 135 MB）。

结论：省掉的不是一个 String 对象，而是每条大事件发送时的四五次整包数组拷贝。

---

## 2. 实际代码与优化点

### 2.1 现在的主要发送代码

当前 `BoxStreamSseSupport` 的正常发送路径是：

```java
private static void send(SseEmitter emitter, String name, Object payload) {
    try {
        emitter.send(
            SseEmitter.event()
                .name(name)
                .data(payload, SSE_JSON)
        );
    } catch (IOException e) {
        throw new SseClientDisconnectedException(e);
    }
}
```

其中：

```java
private static final MediaType SSE_JSON = MediaType.APPLICATION_JSON;
```

`buildSink()` 会创建不同的事件对象，然后调用 `send()`：

- `BoxStreamLayerEventVO`：`layer` 事件；
- `BoxStreamErrorEventVO`：`error` 事件；
- `BoxStreamDoneEventVO`：`done` 事件。

因此，正常路径传给 `.data(payload, SSE_JSON)` 的是 Java VO 对象，不是提前序列化好的 String。

### 2.2 原来的写法

原来相当于：

```java
String json = objectMapper.writeValueAsString(payload);

emitter.send(
    SseEmitter.event()
        .name(name)
        .data(json, SSE_JSON)
);
```

这里多了一步：先要求 Jackson 返回完整 JSON String，然后再把 String 交给 `SseEmitter` / Spring MVC 发送。

### 2.3 当前代码中仍然存在的一处旧路径

`runStream()` 的异常处理分支目前仍然是：

```java
emitter.send(
    SseEmitter.event()
        .name("error")
        .data(objectMapper.writeValueAsString(evt), SSE_JSON)
);
```

如果希望异常事件也统一走对象直传，可以改为：

```java
emitter.send(
    SseEmitter.event()
        .name("error")
        .data(evt, SSE_JSON)
);
```

这条异常分支通常不是大流量主路径，但统一写法可以避免异常事件仍然走完整 String 物化路径。改完后，如果 `objectMapper` 参数没有其他用途，可以再考虑从 `runStream()` 的参数中移除。

### 2.4 `SSE_JSON` 与整个 HTTP 响应 Content-Type 的区别

需要区分：

- SSE 整体响应通常是 `text/event-stream`；
- `.data(payload, SSE_JSON)` 中的 `APPLICATION_JSON` 是告诉 Spring：这个事件的 data 部分按 JSON 类型选择消息转换器。

SSE 事件仍然会有事件名、`data:` 前缀和事件分隔换行。直接传 VO 不代表取消 SSE 协议封装。

---

## 3. Java 21 的 String 为什么内部是 byte[]

### 3.1 String 变量、String 对象和底层数组不是一回事

例如：

```java
String json = "{\"name\":\"Tom\"}";
```

可以把引用关系简化为：

```text
局部变量 json
    |
    v
String 对象
    |
    +-- value ----> byte[]
    |
    +-- coder
    |
    +-- 其他字段
```

- 局部变量 `json` 保存的是 String 对象引用；
- String 对象表示 Java 字符串，并保存相关元数据；
- Java 21 的 String 内部通过 `value` 字段引用 `byte[]` 保存内容；
- `coder` 用来标识内部采用 Latin-1 还是 UTF-16 表示。

因此，String 的内容并不是“另一个 String”，而是由底层数组保存。

### 3.2 Compact Strings

Java 9 起引入 Compact Strings，Java 21 仍采用这一设计。

大致规则：

| String 内容 | 内部存储方式 | 大致空间 |
|---|---|---:|
| 所有字符都能用 Latin-1 表示 | `byte[]`，Latin-1 coder | 每个 Latin-1 字符约 1 字节 |
| 存在 Latin-1 无法表示的字符 | `byte[]`，UTF-16 coder | 每个 UTF-16 代码单元约 2 字节 |

例如纯 ASCII JSON 通常可以用 Latin-1 紧凑存储；如果 String 中包含中文等 Latin-1 无法表示的字符，整个 String 可能使用 UTF-16 coder。不能简单地假设所有 JSON String 都是“字符数乘 2 字节”，也不能假设所有 JSON String 都是“字符数乘 1 字节”。

### 3.3 String 内部 byte[] 与 UTF-8 输出 byte[] 是两种不同的数组

例如：

```java
String text = "你好";
byte[] utf8 = text.getBytes(StandardCharsets.UTF_8);
```

此时：

- `text` 仍然是 String；
- String 内部有自己的 `value` 数组；
- `utf8` 是另一份 UTF-8 编码结果，是 `byte[]`；
- UTF-8 编码结果不是 String。

因此可能同时出现两份不同用途的 `byte[]`：

1. String 内部用于保存字符串内容的数组；
2. 将 String 编码为 UTF-8、准备写入网络时产生的数组。

两份数组的内容表示、编码方式和生命周期可能不同，不能把它们混为一谈。

---

## 4. Jackson 的 char[]、String 和 byte[] 是怎样关联的

### 4.1 `writeValueAsString()` 为什么会使用字符缓冲区

`ObjectMapper.writeValueAsString(payload)` 的目标是返回一个完整 String。Jackson 需要先把序列化结果收集起来，再构造 String。

Jackson 的相关实现涉及 `SegmentedStringWriter` 和 `TextBuffer`。`TextBuffer` 可以使用字符数组段保存和拼接文本；数据量较大时，不应把它想象成一个永远固定大小、每次 SSE 都重复覆盖的单一 `char[]`。

概念流程：

```text
Java VO / 对象
    |
    v
Jackson 序列化
    |
    v
SegmentedStringWriter / TextBuffer
    |
    +--> char[] 字符缓冲区或数组段
    |
    v
生成完整 String
    |
    +--> Java 21 String.value: byte[]
```

构造 String 时，字符内容需要转换/复制到 String 自己的内部存储。这个过程中，字符缓冲区与新 String 的底层数组可能在一段时间内同时存在。

### 4.2 不能把 Heap Dump 中所有 char[] 都直接认定为 TextBuffer

Histogram 只告诉我们 `char[]` 类实例的总量，不会自动告诉我们每个数组来自哪个类。

`char[]` 可能来自：

- Jackson 的字符缓冲区；
- 其他文本处理代码；
- 仍在使用中的对象；
- 被缓冲区复用机制保留的数组；
- 已不可达但尚未被 GC 回收的数组（如果快照包含不可达对象）。

要证明某一块 `char[]` 属于 Jackson `TextBuffer`，需要查看该数组的具体实例和 Incoming References / 引用链。

**2026-11-06 已用 JFR 分配栈证明**：直接写入路径中约 53.4 MB 的 `char[]` 来自 Kingbase JDBC `UTF8Encoding.decode`（读库解码），旧路径额外多出约 135 MB 的 `char[]` 全部来自 `TextBuffer.carr/expand`（`SegmentedStringWriter.write` 之上，含 `_writeLongString` 与 `_writeString2` 两条入口）。也就是说，`char[]` 的两个来源都被逐栈确认了，不再依赖类名推断。

### 4.3 直接写对象时仍然需要缓冲区

现在把 VO 直接交给 Spring/Jackson，并不意味着不再进行序列化，也不意味着不再分配数组。

Jackson 仍要：

- 遍历对象；
- 序列化字段；
- 对字符串内容进行处理；
- 把 JSON 内容编码为输出字节；
- 使用必要的输出缓冲区。

因此，新路径中仍然有 `byte[]` 和 `char[]` 是正常的。优化目标是避免把整个事件先物化为一个完整 String，而不是让所有数组消失。

---

## 5. Spring MVC / SseEmitter 的消息转换链路

### 5.1 对象直传时的预期路径

当前代码：

```java
.data(payload, MediaType.APPLICATION_JSON)
```

`payload` 是 VO 对象。Spring MVC 会根据对象类型、媒体类型和已注册的 `HttpMessageConverter` 选择转换器。通常预期由 Jackson 的 JSON 消息转换器处理 VO，并将 JSON 写到响应输出流。

概念上：

```text
VO 对象
  |
  v
SseEmitter / ResponseBodyEmitter
  |
  v
Spring MVC HttpMessageConverter
  |
  v
Jackson JSON Generator
  |
  v
UTF-8 字节输出缓冲区
  |
  v
HTTP/SSE OutputStream
```

Jackson 对输出流写 JSON 时仍然需要 UTF-8 编码。它可以在生成 JSON 的过程中逐步编码并写出，而不必先生成一个完整 JSON String。

### 5.2 String 路径可能发生什么

旧代码传入的是 String：

```java
.data(json, MediaType.APPLICATION_JSON)
```

在 Spring MVC 默认配置中，`StringHttpMessageConverter` 通常是可用的，而且常见默认转换器顺序中它位于 Jackson 转换器之前。String 类型可能由它处理。

`StringHttpMessageConverter` 的典型职责是把 String 按选定字符集写成输出字节。实现路径可能调用 String 的字符集编码逻辑，从而创建一份 UTF-8 `byte[]`，再写入输出流。

因此，旧路径可能有：

```text
writeValueAsString()
    |
    +--> 完整 JSON String 的内部 byte[]
    |
    v
StringHttpMessageConverter
    |
    +--> String 编码生成的 UTF-8 byte[]
    |
    v
OutputStream
```

**这一条已经由 JFR 分配栈证实（2026-11-06）**。旧路径录制中最大的一条 `byte[]` 分配栈是：

```text
java/lang/String.getBytes
  <- org/springframework/util/StreamUtils.copy
  <- org/springframework/http/converter/StringHttpMessageConverter.writeInternal
```

该栈在旧路径下累计分配 218.9 MB `byte[]`（另有同一 encode 路径内部的 `Arrays.copyOf` 74.2 MB），而直接写流路径完全没有这条栈。也就是说本项目实际就是由 `StringHttpMessageConverter` 处理 String + `application/json`，并且每次发送都把整个 JSON String 再编码成一份新的 UTF-8 `byte[]`。

差额的其余部分也已被逐栈确认（见第 8.3 节），不再需要打印 converter 列表来间接推断。

### 5.3 如何检查项目实际注册的转换器

可以在项目中临时注入 `RequestMappingHandlerAdapter`，打印转换器及其对 String / VO 的支持情况：

```java
@Autowired
private RequestMappingHandlerAdapter handlerAdapter;

@PostConstruct
public void printMessageConverters() {
    for (HttpMessageConverter<?> converter : handlerAdapter.getMessageConverters()) {
        log.info(
            "converter={}, canWriteString={}, canWriteVO={}",
            converter.getClass().getName(),
            converter.canWrite(String.class, MediaType.APPLICATION_JSON),
            converter.canWrite(BoxStreamLayerEventVO.class, MediaType.APPLICATION_JSON)
        );
    }
}
```

这能帮助确认转换器注册情况和候选支持关系。若要确认某次事件实际选中了哪个转换器，还应结合调试断点、日志或 JFR 调用栈验证。

---

## 6. 为什么 byte[] 总量可以比 char[] 大

不能根据单个元素大小推断整个类的总内存：

- 一个 `byte` 元素通常占 1 字节；
- 一个 `char` 元素通常占 2 字节；
- 但 Histogram 中的总量是所有该类型数组的总和。

总量取决于数组数量和每个数组长度：

```text
byte[] 总量 = 所有 byte[] 实例的实际大小之和
char[] 总量 = 所有 char[] 实例的实际大小之和
```

如果堆里有很多很大的 `byte[]`，即使每个元素只占 1 字节，`byte[]` 总量仍然可能远大于 `char[]`。

本次直接写入快照中：

- `byte[]`：67.7 MB；
- `char[]`：56.4 MB。

这并不矛盾。`char[]` 本身仍然很大；只是当前快照里所有 `byte[]` 的累计大小更大。

另外：

- `byte[]` 不等于“全部是地理数据”；
- `char[]` 不等于“全部是一个固定 SSE 缓冲区”；
- 一个 Histogram 类别可能包含许多不同对象、不同用途和不同生命周期的数组。

---

## 7. GC 为什么会让堆峰值高于当前仍被使用的数据量

假设每次都先创建完整 JSON String：

```java
String json = objectMapper.writeValueAsString(payload);
emitter.send(SseEmitter.event().name("layer").data(json, SSE_JSON));
```

发送完成后，如果没有其他对象引用 `json`，它及其底层数组就可能变成不可达对象。

但要注意：

**对象不可达，不等于内存立即释放。**

GC 何时运行、何时回收该对象，取决于垃圾收集器的触发条件和执行过程。在下一次 GC 真正回收它之前，该数组仍可能计入堆使用量。

连续处理多个大事件时，可能出现：

```text
第 1 条：创建 String1 / byte[]1 -> 发送结束 -> 变成不可达
第 2 条：创建 String2 / byte[]2 -> 发送结束 -> 变成不可达
第 3 条：创建 String3 / byte[]3
...
GC 尚未回收前面的数组
```

这样，多个已经不再需要的数组可能在某个时间窗口内同时占据堆空间，抬高 Heap Used。

这不是内存泄漏的充分证据。内存泄漏通常意味着对象在业务上已经不需要，但仍被某条引用链长期保留。若对象已经不可达，只是等待 GC，则属于正常的垃圾尚未回收。

---

## 8. 实验数据：必须区分 2026-10-19 与 2026-11-05

这两轮测试不能混成同一次实验。

### 8.1 2026-11-05：对象数组构成快照

本次使用同一份代码，仅改变写出方式；在堆用量升高时抓取未先执行全量 GC 的堆快照。两次响应大小一致：

- 响应字节数：`77,963,820` bytes；
- 约 `74.4 MiB`（按 1024² 换算）；
- 整段字符串路径快照时堆用量：约 `510 MB`；
- 直接写入路径快照时堆用量：约 `195 MB`。

Histogram 结果：

| 类 | 整段字符串 | 直接写入 | 差额 |
|---|---:|---:|---:|
| `byte[]` | 307.5 MB | 67.7 MB | **239.8 MB** |
| `char[]` | 110.1 MB | 56.4 MB | **53.7 MB** |
| `String` 对象自身 | 4.8 MB | 5.4 MB | 没有减少 |

两类数组合计：

```text
优化前：307.5 + 110.1 = 417.6 MB
优化后： 67.7 +  56.4 = 124.1 MB
差额：                         293.5 MB
```

堆用量差：

```text
510 - 195 = 315 MB
```

两类数组合计差额与堆用量差额相差约：

```text
315 - 293.5 = 21.5 MB
```

这约 21.5 MB 不能仅凭这张表进一步归因；还可能涉及其他对象类别、采样时刻差异和其他堆对象。

**重要：** `String` 这一行统计的是 String 对象自身的浅堆，不包含它引用的 `byte[]`。因此 String 自身从 4.8 MB 变成 5.4 MB，并不表示 String 内容数组没有变化；底层数组被单独统计在 `byte[]` 类别中。

用户当前记录中的归因是：直接写入后剩余的 67.7 MB `byte[]` 主要来自数据库 `ST_AsGeoJSON` 返回的几何文本 String 底层存储。这个归因符合当前链路理解，但要做到对象级证明，仍应在 MAT 中查看具体数组的 Incoming References。

### 8.2 2026-10-19：堆峰值与容器指标

另一轮历史测试记录：

| 指标 | 整段字符串 | 直接写入 |
|---|---:|---:|
| SSE Heap Used 峰值 | 679 MB | 255 MB |
| SSE Heap Committed | 706 MB | 285 MB |
| 容器驻留峰值 | 1097 MB | 630.3 MB |
| 一次性返回 Heap Used 峰值 | 291 MB | 302 MB |

其中 SSE Heap Used 峰值差：

```text
679 - 255 = 424 MB
```

这 424 MB 是**两次峰值计数器的差**，不是某一类对象被精确证明释放了 424 MB。

2026-10-19 的峰值时刻没有保留对应的对象 Histogram / Heap Dump 构成，因此不能把 2026-11-05 的 `byte[]` 239.8 MB 和 `char[]` 53.7 MB 差额直接当成 2026-10-19 的 424 MB 对象构成。

两轮数据可以共同支持“直接写入显著降低了 SSE 堆压力”这一判断，但对象级归因必须使用各自同一轮的快照证据。

### 8.3 2026-11-06：JFR 分配栈对照（本次归因的关键证据）

同一请求（直连 8108，bbox `[35138.4423, 224812.4682, 64785.6564, 254459.6822]`），响应都是 77,963,820 字节，分别部署旧包（`writeValueAsString`）与新包（直接写流），用 JFR `settings=profile` 外加手动启用 `jdk.ObjectAllocationOutsideTLAB` 录制全程，逐次统计（非抽样）：

| 路径 | `byte[]` 域外分配 | `char[]` 域外分配 | 合计 |
|---|---:|---:|---:|
| 旧方案 `writeValueAsString` | 713.1 MB | 192.6 MB | **905.7 MB** |
| 新方案直接写流 | 56.4 MB | 53.8 MB | **110.2 MB** |

两边共有的部分就是读库本身的固定开销（旧/新几乎一致）：

| 调用栈 | 旧路径 | 新路径 |
|---|---:|---:|
| Kingbase `UTF8Encoding.decode` 的 `char[]` | 53.4 MB | 53.4 MB |
| `KBStream.receiveTupleV3` 读缓冲 `byte[]` | 27.9 MB | 28.2 MB |
| 解码后 `StringUTF16.compress` 的 `byte[]` | 27.4 MB | 27.6 MB |
| **读库小计** | **约 109 MB** | **约 109 MB** |

也就是说新路径的 110.2 MB 几乎全部是读库开销；11-05 快照里直接写入路径剩余的 67.7 MB `byte[]`、56.4 MB `char[]`，对应的就是这三条栈。

旧路径独有的三条栈（差额约 795 MB 的构成）：

| # | 调用栈（截取） | 分配量 | 说明 |
|---|---|---:|---|
| 1 | `String.getBytes ← StreamUtils.copy ← StringHttpMessageConverter.writeInternal` | 218.9 + 74.2 = **293.1 MB** `byte[]` | Spring 把整个 JSON String **再编码一份 UTF-8** 才写网络 |
| 2 | `AbstractStringBuilder.inflate ← StringBuilder.append ← TextBuffer.contentsAsString ← writeValueAsString` | **145.7 MB** `byte[]` | 拼 String 时的 UTF-16 展开 |
| 3 | `Arrays.copyOfRange ← String.<init> ← StringBuilder.toString ← TextBuffer.contentsAsString` | **145.7 MB** `byte[]` | `toString` 的最终拷贝 |
| 4 | `AbstractStringBuilder.<init> ← StringBuilder.<init> ← TextBuffer.contentsAsString` | **72.8 MB** `byte[]` | StringBuilder 本体 |
| 5 | `TextBuffer.carr/expand ← SegmentedStringWriter.write`（`_writeLongString` / `_writeString2`） | 95.9 + 38.5 ≈ **135 MB** `char[]` | Jackson 的 UTF-16 序列化缓冲，翻倍扩容时旧块一并计入 |

三条栈合计约 **790 MB**。以一条 28 MB 的事件为例：`char[]` 缓冲约 2 倍体积、UTF-16 展开一份、`toString` 一份、String 内部 `byte[]` 一份、`getBytes` 再一份，约等于事件体积的五六倍。这些数组全部大于 G1 humongous 阈值（区 1 MB 的一半），Young GC 不回收，请求期间逐条叠加，这正是 10-19 峰值差 424 MB 的来源；11-05 快照里的 293.5 MB 差额是 dump 那一刻仍驻留的子集。

辅证（`jdk.ObjectAllocationSample`，300/s 抽样，只看相对构成）：旧路径 `byte[]` 854.2 MB + `char[]` 152.9 MB，直接路径 225.8 MB + 54.3 MB，最大栈与逐次统计一致。

`HeapByteBuffer` 不是差额来源：抽样里旧路径只有约 2.3 MB（Tomcat `OutputBuffer` 包装输出分片），直接路径更少。MAT 里 3416 个不可达 `HeapByteBuffer` 共享同一份大 `byte[]`，它们是小对象包装，不能把数量乘上去。

注意：OutsideTLAB 事件只覆盖 TLAB 之外的分配（大数组主要在这里），不是全部分配；JFR 录制本身会拖慢请求，本次耗时约 50 s（平时约 50–55 s），不影响结论，但不要用这次的耗时或峰值去更新 10-19 的数字。

### 8.4 MAT 对这两份快照的处理记录

两份 dump 均以 `GC.heap_dump -all=true` 生成（不做全量 GC，不可达对象保留在文件里）。MAT 默认在解析时删除不可达对象：

| dump | MAT 日志 |
|---|---|
| 字符串序列化 | `Removed 263,539 unreachable objects using 403,054,760 bytes` |
| 直接写流 | `Removed 333,147 unreachable objects using 93,850,904 bytes` |

所以 MAT 概览上字符串那份只有 95.3 MB、最大保留对象是仍挂在数据库连接上的 Kingbase `KBStream`（保留 53.4 MB）。被删掉的约 384 MB 才是 `writeValueAsString` 产生的、已不可达但尚未被 GC 的数组。要看它们：`Keep unreachable objects` 勾选后必须 **Delete Index Files 再重新解析**（MAT 会复用旧索引，仅勾选不重解析无效）；保留不可达对象后用 Java Basics → GC Roots → Unreachable Objects 查看。

---

## 9. Histogram、Dominator Tree 和引用链分别解决什么问题

### 9.1 Histogram：先找内存大户

Histogram 按类统计对象数量和浅堆大小。例如：

```text
byte[]       307.5 MB
char[]       110.1 MB
String         4.8 MB
ArrayList      ...
HashMap        ...
```

它适合回答：

> 哪些类占用了最多堆内存？

它不直接回答：

- 这些数组是谁创建的；
- 每个数组属于哪个业务对象；
- 对象是否仍被业务需要；
- 分配发生在哪一行代码；
- 某个时间窗口累计分配了多少内存。

Histogram 是堆快照的分类统计，不是分配火焰图，也不是内存随时间变化的走势图。

### 9.2 Shallow Heap 与 Retained Heap

**Shallow Heap（浅堆）**：对象自身占用的大小，不包含它引用的其他对象。

例如 String 的浅堆只统计 String 对象本身；其 `value` 引用的 `byte[]` 会在 `byte[]` 类别中单独统计。

**Retained Heap（保留堆）**：如果某对象被回收，因它而变得可回收的对象所占内存总量。它用于分析对象是否支配着一批其他对象。

不要把 Shallow Heap 和 Retained Heap 混为一谈。

### 9.3 Dominator Tree：找谁支配了大量内存

Dominator Tree 按对象支配关系展示内存。它适合回答：

> 哪个对象或对象链持有了大量其他对象，使它们不能被回收？

如果发现某个业务对象、缓存、集合或会话对象保留了大量数据，可以进一步沿树展开。

### 9.4 Incoming References / Path to GC Roots：找引用来源

选中一个具体 `byte[]` 或 `char[]`：

- Incoming References：查看哪些对象引用它；
- Path to GC Roots：对仍可达对象，查看从 GC Root 到它的引用路径；
- 若对象在包含不可达对象的快照中已经不可达，可能不存在有效 GC Root 路径。

例如，若看到：

```text
String
  |
  +-- value
        |
        +-- byte[]
```

可以证明这个数组是某个 String 的底层存储。

若看到：

```text
某个 Jackson Generator
  |
  +-- outputBuffer
        |
        +-- byte[]
```

则可进一步确认它属于 Generator 输出缓冲区。

这比仅凭数组类名推断用途可靠得多。

---

## 10. JFR、MAT 和抓包分别该怎么用

### 10.1 MAT：回答“快照里有什么、谁引用它”

建议流程：

1. 打开 Heap Dump；
2. 进入 Histogram；
3. 按 Shallow Heap 降序；
4. 找到 `byte[]`、`char[]`；
5. 对具体数组实例查看 Incoming References；
6. 对仍可达对象查看 Path to GC Roots；
7. 在 Dominator Tree 中检查是否存在大对象持有者；
8. 对比优化前后同类对象的数量和浅堆总量。

注意：用户本次使用了未先做全量 GC 的堆快照。若使用包含不可达对象的 dump 方式，Histogram 也可能统计到尚未回收的垃圾对象。这样适合观察“尚未回收的对象堆积”，但不能把它们全部当成仍然存活、仍被业务引用的对象。

### 10.2 JFR：回答“对象在哪里分配”

JFR 可以记录对象分配事件及调用栈，帮助定位 `byte[]`、`char[]` 是由哪个调用路径分配的。

示例命令：

```powershell
jcmd <PID> JFR.start name=sse-test settings=profile
```

- `jcmd`：向目标 JVM 发送诊断命令；
- `<PID>`：替换为分析服务进程号；
- `JFR.start`：启动 Flight Recording；
- `name=sse-test`：为本次记录命名；
- `settings=profile`：使用 profile 配置。

完成一次测试后：

```powershell
jcmd <PID> JFR.dump name=sse-test filename=D:\Temp\sse-test.jfr
```

这会把记录导出到指定文件。

**JDK 21 的一个坑（本项目实测踩到）**：`settings=profile` 下 `jdk.ObjectAllocationOutsideTLAB` / `jdk.ObjectAllocationInNewTLAB` 默认是**关闭**的（JDK 16 起被 `jdk.ObjectAllocationSample` 取代），录出来是 0 条。要看大数组（humongous、TLAB 外）的逐次分配，必须在启动时追加事件级参数：

```text
jcmd <PID> JFR.start name=sse settings=profile +jdk.ObjectAllocationOutsideTLAB#enabled=true
```

本项目服务跑在公司 Docker 里，`jcmd` 不能从本机直发，实际用的是宿主机 `/tmp/jattach <hostJavaPid> jcmd '...'`，Java 宿主机 PID 取自 `docker top`（不是 `docker inspect State.Pid`，那是 shell 的 PID）；录制文件写在 JVM 可见的容器内 `/tmp`，再 `docker cp` 出来。

`jdk.ObjectAllocationSample` 是采样事件（默认节流 300/s，`weight` 是加权估计），不能把事件大小相加当精确值；`jdk.ObjectAllocationOutsideTLAB` 的 `allocationSize` 是逐次精确值，适合按调用栈聚合。聚合方式：

```powershell
jfr print --json --stack-depth 24 --events jdk.ObjectAllocationOutsideTLAB file.jfr
```

然后按 `objectClass`（JSON 里原始数组类名是 `[B`、`[C`）+ 调用栈前几帧分组求和。注意 JSON 里事件体在 `values` 键下，帧的 `type` 字段是执行模式字符串（如 `Interpreted`），类名在 `method.type.name` 里。

本项目两次录制的归档：`D:\Job\简历\数据` 下的 `sse-allocation-string.jfr`（旧路径）与 `sse-allocation-direct.jfr`（新路径）。

JFR 与 MAT 的区别：

| 工具 | 主要回答 |
|---|---|
| JFR | 对象在哪里分配、哪些调用路径分配频繁 |
| MAT | 快照时对象还在不在、谁引用它、谁支配它 |
| Histogram | 哪些类的对象总量最大 |

### 10.3 抓包：回答“网络上发送了什么”

Wireshark / tcpdump 适合检查：

- 实际传输字节量；
- SSE 帧格式；
- 事件是否完整；
- 是否存在重复发送；
- 网络层重传等问题。

抓包**不能直接告诉你 JVM 内部创建了多少个 String 或 byte[]**。如果使用 HTTPS，抓包默认看到的是 TLS 加密数据，通常无法直接阅读 JSON 内容。

如果只是确认 HTTP 响应体大小，浏览器 DevTools Network 或客户端保存响应后统计字节数通常更简单。

---

## 11. 目前哪些结论已经成立，哪些还需要验证

| 结论 | 当前证据状态 |
|---|---|
| 新代码把 VO 对象直接传给 `.data(payload, APPLICATION_JSON)` | **代码已确认** |
| 旧路径先通过 `writeValueAsString()` 创建完整 String | **旧实现已确认** |
| Java 21 String 内部使用 `byte[]` 保存内容 | **Java 21 实现机制** |
| UTF-8 编码结果是字节，不是 String | **确定** |
| 2026-11-05 两类数组合计减少 293.5 MB | **Histogram 实测** |
| 2026-11-05 快照时堆用量相差 315 MB | **监控读数实测** |
| `char[]` 全部来自 Jackson TextBuffer | **已被 JFR 逐栈否定并修正**：`char[]` 有两个来源——直接路径约 53.4 MB 是 Kingbase 解码，旧路径额外约 135 MB 才是 TextBuffer |
| 剩余 67.7 MB `byte[]` 主要是 GeoJSON String 底层数组 | **已由 JFR 栈定位**：直接路径的 `byte[]` 主要是 Kingbase 读缓冲与解码压缩（约 56 MB），与该归因一致 |
| 旧 String 路径由 `StringHttpMessageConverter` 处理 | **已证实**：JFR 栈 `String.getBytes ← StreamUtils.copy ← StringHttpMessageConverter.writeInternal`，仅旧路径存在 |
| 旧路径一定额外产生一份完整 UTF-8 byte[] | **已证实并量化**：218.9 + 74.2 ≈ 293 MB（逐次事件求和） |
| 239.8 MB 的 byte[] 差额全部是最终 JSON String 的 value | **不能这样下结论**；差额是 String 构造链路 + 二次编码 + 缓冲扩张的叠加，快照只是其中仍驻留的部分 |
| 2026-10-19 的 424 MB 峰值差等于释放的对象大小 | **错误；它只是两次峰值读数之差**。JFR 显示旧路径累计多分配约 790 MB，峰值差 424 MB 是这些 humongous 数组在峰值时刻仍驻留的部分 |
| MAT 概览数字偏小（95.3 MB）是 dump 有问题 | **错误**；是 MAT 默认删除了不可达对象（字符串 dump 删掉约 384 MB），勾选 Keep unreachable + 删索引重解析即可看到 |

---

## 12. 下一步验证计划

前三步已经完成（2026-11-06），保留记录如下：

- **第一步：确认实际消息转换器** —— 已由 JFR 栈直接证实：旧路径实际命中 `StringHttpMessageConverter`（`String.getBytes ← StreamUtils.copy ← writeInternal`，且该栈仅旧路径存在），不需要再打印 converter 列表。
- **第二步：分别对旧路径和新路径录制 JFR** —— 已完成，方法与结果见第 8.3、10.2 节。
- **第三步：对具体数组做 MAT 引用链分析** —— 核心部分已完成：不可达对象的类型构成（`byte[]` 307.5→67.7 MB、`char[]` 110.1→56.4 MB）与 JFR 分配栈互相印证；保留不可达对象后，可达的最大保留者是 Kingbase `KBStream`（53.4 MB），不属于差额来源。
- **第四步：证据归档** —— `D:\Job\简历\数据` 下有两份 hprof.gz、两份 JFR 录制、MAT 索引与 Leak Suspects 报告；输入、响应字节数、耗时见 8.1–8.3。

仍然可做的后续（非必需）：

- 在 MAT 中对直接写流路径的大 `byte[]` 实例逐个跑 Incoming References，把 67.7 MB 全部挂到 Kingbase 解码栈上（当前是 JFR 栈侧证明，对象级引用链未逐个展开）。
- 如果要把 10-19 的 424 MB 峰值差也做成对象构成，需要在峰值时刻（而非请求结束后）用同样的 `-all=true` 方式抓快照，本次未做。

---

## 13. 面试时怎么讲这次优化

### 13.1 推荐的技术表述

> 针对大数据量 SSE 返回过程中先将事件对象整体序列化为 JSON String、再交给 Spring 写出所产生的额外内存分配，将发送方式调整为直接传递事件 VO，由 Spring/Jackson 在响应写出阶段完成 JSON 序列化。用 JFR 分配事件对照验证：同一请求下，旧路径 `byte[]`+`char[]` 的 TLAB 外分配合计 905.7 MB，新路径 110.2 MB；多出来的约 790 MB 落在三条栈上——`StringHttpMessageConverter` 对整个 JSON String 的二次 UTF-8 编码（约 293 MB）、Jackson `TextBuffer.contentsAsString` 构造 String 的 UTF-16 展开与拷贝（约 364 MB）、`TextBuffer` 字符缓冲（约 135 MB）。堆快照同样显示 `byte[]` 从 307.5 MB 降至 67.7 MB、`char[]` 从 110.1 MB 降至 56.4 MB。SSE 堆峰值从 679 MB 降到 255 MB，容器驻留从约 1.1 GB 降到约 630 MB。

这段话的每一处数字都有实测来源：JFR 逐次分配事件、堆快照 Histogram、峰值监控读数。

### 13.2 如果面试官问“为什么 byte[] 减少这么多”

可以回答：

> 原实现先通过 `writeValueAsString()` 构造完整 JSON String，这一步并不是只产生“一个 String 对象”。JFR 分配栈显示构造过程至少经过：Jackson `TextBuffer` 的 UTF-16 字符缓冲（约 135 MB `char[]`）、`contentsAsString` 时 `StringBuilder` 的 UTF-16 展开、`toString` 的最终拷贝和 String 压缩（合计约 364 MB `byte[]`）；然后 `StringHttpMessageConverter` 还要把整个 String 再 `getBytes(UTF-8)` 一份才能写网络（约 293 MB `byte[]`）。这些数组都超过 G1 的 humongous 阈值，Young GC 不回收，连续大事件会在 GC 前叠加。对象直传后 Jackson 边序列化边写出，这四类整包拷贝全部消失，两条路径剩下的都是读库本身约 109 MB 的分配。

### 13.3 如果面试官问“直接写入还需要 UTF-8 编码吗”

回答：

> 需要。网络发送的是字节，JSON 字符内容仍然必须编码。区别是原来先生成完整 String，再由后续写出组件将 String 编码成字节；现在 Jackson 可以在序列化对象时直接生成 UTF-8 字节并写入输出流，省掉完整 String 这一中间表示。

### 13.4 如果面试官问“为什么 byte[] 比 char[] 大”

回答：

> Histogram 统计的是所有该类型数组的累计浅堆大小，不是单个元素大小。虽然 char 元素通常是 2 字节、byte 元素是 1 字节，但 byte[] 的数量和长度总和可能更大。JFR 分配栈也证实了具体来源：byte[] 的主体是 String 的二次 UTF-8 编码和 String 构造链路，char[] 的主体是 Kingbase 读库解码和 Jackson 的 UTF-16 缓冲，两类数组用途不同、不能互相换算。

### 13.5 如果面试官问“一个 74 MB 的响应怎么省出 400 多 MB”

回答：

> 424 MB 是两次堆峰值读数之差，不是某个对象的大小。JFR 逐次分配事件显示旧路径在同一个请求里累计多分配约 790 MB：每条大事件的 JSON 都被物化成 char[] 缓冲、UTF-16 展开、String 内部 byte[]、二次 UTF-8 编码等四五份整包拷贝，单条事件就要分配自身体积五六倍的临时数组；这些数组超过 G1 的 humongous 阈值（约 512 KB），Young GC 不回收，请求期间逐条叠加，峰值时刻仍驻留的部分就是 424 MB。快照对照看到的 293.5 MB 差额是 dump 那一刻仍驻留的子集。

### 13.6 不要这样说

- 不要说“直接写入不需要 UTF-8 编码”；
- 不要说“所有 byte[] 都是地理数据”；
- 不要说“所有 char[] 都是同一个可复用 SSE 缓冲区”；
- 不要说“byte[] 减少 239.8 MB，全部就是 String 内部数组”；
- 不要把 2026-10-19 的 424 MB 峰值差说成精确释放的对象大小；
- 不要把 2026-11-05 的对象快照构成直接套到 2026-10-19 的峰值上；
- 不要说“424 MB 就是省掉的一个 String”，JFR 已证明差额是四五类整包拷贝的叠加。

---

## 14. 最终记忆模型

把整个案例压缩成下面六句话：

1. **String 是对象，String 内部通过 `byte[] value` 保存内容；String 对象浅堆不包含这个数组。**
2. **`char[]` 的来源已被 JFR 逐栈证实：直接路径约 53 MB 来自 Kingbase 解码，旧路径额外约 135 MB 来自 Jackson `TextBuffer`；Histogram 的类名不能替代调用栈。**
3. **UTF-8 编码的结果是 `byte[]`，不是 String；String 内部的 byte[] 与 `StringHttpMessageConverter` 二次编码的 byte[] 是两份不同的数组，后者约 293 MB，是最大单项。**
4. **原来的 `writeValueAsString()` 每条大事件要整包拷贝四五份；对象直传让 Jackson 边序列化边写出，旧路径独有的三条栈全部消失，两条路径都只剩读库约 109 MB 的分配。**
5. **对象不可达不代表立即回收；超过 G1 humongous 阈值的大数组不会被 Young GC 收走，会在请求期间叠加并推高 Heap Used。**
6. **Histogram 找类别，Dominator Tree 找支配者，Incoming References 找引用来源，JFR `ObjectAllocationOutsideTLAB` 找精确分配调用栈（JDK 21 的 profile 默认关它，要手动开）；它们不能互相替代。**

本案的归因链已经闭环：堆快照给出“少了两类数组共 293.5 MB”，JFR 分配栈给出“多分配的 790 MB 分别来自哪三条栈”，两者相互印证。剩余可做的是对象级引用链展开和峰值时刻快照，属于锦上添花。
