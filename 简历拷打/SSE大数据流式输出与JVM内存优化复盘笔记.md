# Spring MVC SseEmitter 大数据 SSE 输出与 JVM 内存优化复盘

> **主题**：SseEmitter、Spring MVC HttpMessageConverter、Jackson、Java
> 21 String、JFR、MAT、G1\
> **场景**：analysis-service 大框 GeoJSON SSE 输出\
> **核心改动**：从 `writeValueAsString(payload)` 后传 String，改为直接将
> POJO 传给 `SseEmitter`。\
> **说明**：本文区分代码事实、JFR 分配证据、MAT
> 快照证据和机制解释。不同日期、不同指标的实验数据不能混为一谈。

## 1. 核心结论

这次优化**没有更换 SseEmitter，没有删除业务结果 VO，也没有取消 JSON
序列化**。改变的是传给 `SseEmitter.event().data(...)` 的参数类型。Spring
MVC 根据 data 对象类型和 MediaType 选择不同的
`HttpMessageConverter`，因此后续序列化和写出路径发生变化。

旧路径先执行 `ObjectMapper.writeValueAsString(payload)`，要求 Jackson
在堆上构造完整 JSON String；随后 Spring 将 String 交给
`StringHttpMessageConverter`，再编码成 UTF-8 字节写入响应流。

新路径直接传递 POJO，由 Spring 选择 Jackson JSON 消息转换器。Jackson
使用 `JsonGenerator` / `UTF8JsonGenerator`，在生成 JSON 的过程中直接向
HTTP 输出流写出，不再先构造完整 JSON String。

因此，优化减少的不是一个几十字节的 String 对象头，而是**完整 JSON
字符串物化、字符缓冲、StringBuilder 中间存储、String 内部数组，以及
String 后续 UTF-8 编码所形成的整条中间数据分配链路**。

用户提供的 JFR 对照结果显示：相同请求、相同响应字节数下，旧方案
`byte[] + char[]` 累计分配约 905.7 MB，新方案约 110.2 MB，差额约 795.5
MB。这个差额是**累计分配量差异**，不能直接等同于某一时刻的堆存量差异或
Heap Used 峰值差异。

## 2. 代码对比：VO、SseEmitter 都没有变化

### 2.1 旧代码

``` java
String json = objectMapper.writeValueAsString(payload);

emitter.send(
    SseEmitter.event()
        .name(name)
        .data(json, MediaType.APPLICATION_JSON)
);
```

这里有两个对象：`payload` 是业务结果 VO；`json` 是 Jackson 根据
`payload` 生成的完整 JSON String。`writeValueAsString()` 的 API
契约是返回完整 String，所以 Jackson 必须先收集完整序列化文本，再构造
String 返回。

随后 Spring 收到的是 String，通常由 `StringHttpMessageConverter`
处理，并将 String 编码为响应字符集对应的字节，再写入输出流。

### 2.2 新代码

``` java
emitter.send(
    SseEmitter.event()
        .name(name)
        .data(payload, MediaType.APPLICATION_JSON)
);
```

这里传入的仍然是原来的业务 VO。Spring 收到 POJO 后，选择 Jackson JSON
消息转换器，由 JSON Generator 在响应写出阶段生成 JSON 字节。

### 2.3 对照表

  对比项                       旧方案                       新方案
  ---------------------------- ---------------------------- ------------------------
  SseEmitter                   使用                         使用
  SSE 事件封装                 使用                         使用
  业务结果 VO                  存在                         存在
  JSON 序列化                  一次                         一次
  `.data()` 参数               String                       POJO / VO
  典型消息转换器               StringHttpMessageConverter   Jackson JSON Converter
  是否先构造完整 JSON String   是                           否
  是否需要 UTF-8 编码          是                           是
  完整中间 JSON 文本物化       有                           不需要

**注意**：VO 自身可能只有几 KB 的浅堆大小，但它引用的 GeoJSON String
可能很大。不能把"VO 对象自身很小"理解为"整个 VO 对象图只有几 KB"。

## 3. 为什么同一个 SseEmitter 会走不同路径？

SseEmitter 负责 SSE 事件构建与发送，例如事件名、`data:`
内容、事件分隔符和发送过程。它并不意味着不同 Java 类型的 data
内容会使用同一个消息转换器。

Spring MVC 在写出 `ResponseBodyEmitter` 事件时，会根据 data
对象类型、MediaType 和已注册的 `HttpMessageConverter` 选择转换器。

### 3.1 旧路径：String Converter

旧代码传入：

``` java
.data(json, MediaType.APPLICATION_JSON)
```

Spring 收到 String。本次 JFR 证据对应的路径为：

``` text
StringHttpMessageConverter.writeInternal
    -> StreamUtils.copy
        -> String.getBytes(UTF-8)
            -> encode / Arrays.copyOf 等相关分配
```

Jackson 已经把 POJO 序列化成完整 String；Spring 随后又把这个 String
编码成 UTF-8 字节数组，再写入输出流。这不是再次执行 JSON 序列化，而是把
JSON 文本编码为网络字节。网络传输本来就需要字节，但旧路径先完整物化
String，再整体编码，产生了额外的大块中间数组。

### 3.2 新路径：Jackson JSON Converter

新代码传入 POJO：

``` java
.data(payload, MediaType.APPLICATION_JSON)
```

String 转换器不能写 POJO，因此由 Jackson JSON 消息转换器处理。Jackson
使用 `JsonGenerator` / `UTF8JsonGenerator`，在遍历 VO 字段并生成 JSON
的过程中，直接向响应 `OutputStream` 输出字节。

``` text
POJO / VO
    |
    v
Spring Jackson JSON Converter
    |
    v
JsonGenerator / UTF8JsonGenerator
    |
    v
内部输出缓冲区（小块）
    |
    v
HTTP OutputStream
```

新路径仍然执行 JSON 序列化，也仍然进行 UTF-8 编码；区别是无需先构造完整
JSON String，再把它整体编码成另一份完整字节数组。

### 3.3 SSE 协议本身没有改变

-   HTTP 响应整体仍然是 SSE 响应，通常为 `text/event-stream`。
-   `.data(payload, MediaType.APPLICATION_JSON)` 中的 `APPLICATION_JSON`
    用于指定事件 data 内容的媒体类型，帮助 Spring 选择消息转换器。
-   SSE 的事件名、`data:` 前缀和事件分隔仍由 Spring SSE 发送链路处理。
-   本次优化不是手写 Servlet OutputStream，也不是放弃 SseEmitter。

## 4. 旧路径为什么产生多份中间数据？

### 4.1 旧路径概念链路

``` text
业务 VO / payload
    |
    v
ObjectMapper.writeValueAsString(payload)
    |
    v
Jackson TextBuffer / SegmentedStringWriter
    |  char[] 字符缓冲
    v
contentsAsString()
    |
    v
StringBuilder 内部存储 / UTF-16 展开
    |
    v
StringBuilder.toString()
    |
    v
完整 JSON String（Java 21 String 内部 byte[]）
    |
    v
StringHttpMessageConverter
    |
    v
String.getBytes(UTF-8) / encode / Arrays.copyOf
    |
    v
完整或大块 UTF-8 byte[]
    |
    v
HTTP OutputStream
```

### 4.2 新路径概念链路

``` text
业务 VO / payload
    |
    v
SseEmitter.data(payload, APPLICATION_JSON)
    |
    v
Spring Jackson JSON Converter
    |
    v
UTF8JsonGenerator
    |
    v
小块输出缓冲区
    |
    v
HTTP OutputStream
```

新路径仍然遍历 VO、读取原始字段、执行 JSON 转义和 UTF-8
编码，但不再要求先返回完整 JSON String。

### 4.3 Java 21 String 的底层存储

Java 9 起引入 Compact Strings，Java 21 的 String 使用内部 `byte[] value`
保存内容，并通过 coder 标识 Latin-1 或 UTF-16 表示。

-   能用 Latin-1 表示的字符串，通常可以按每字符约 1 字节保存。
-   含有 Latin-1 无法表示字符的字符串，通常使用 UTF-16 表示，约每个
    UTF-16 代码单元 2 字节。
-   String 对象的浅堆不包含它引用的 `byte[]`。
-   String 内部 `byte[]` 与 UTF-8 编码后准备写网络的 `byte[]`
    是两种不同用途的数据。

因此，不能只看 MAT Histogram 中 `String`
类的浅堆大小来判断字符串内容占了多少内存。String 的底层数组会单独统计在
`byte[]` 类别中。

### 4.4 原始 GeoJSON String 在两条路径中都存在

例如：

``` java
class LayerVO {
    private String geoJson;
}
```

这个 `geoJson`
是数据库返回的原始几何文本，属于原始业务数据。新旧路径都需要它，因此它不是这次优化所消除的对象。

``` text
原始数据 String A（geoJson）
    +--> 两条路径都需要

完整 JSON String B（writeValueAsString(payload) 返回值）
    +--> 旧路径需要
    +--> 新路径不需要
```

如果原始 `geoJson` 本身包含 JSON 文本，外层 VO 再序列化时仍要按 JSON
字符串规则处理其中的引号、反斜杠等字符。最终完整 JSON 文本 B 与原始
GeoJSON String A 不是同一个对象。

## 5. JFR 实验：累计分配量的核心证据

### 5.1 为什么普通 profile 记录不够？

本次排查发现，JDK 21 的 `profile`
配置中，`jdk.ObjectAllocationOutsideTLAB` 默认未启用。原先仅依赖常规
`ObjectAllocationSample`
采样，不能得到本次大数组逐次分配的完整事件记录。

随后启用：

``` text
+jdk.ObjectAllocationOutsideTLAB#enabled=true
```

重新录制，获得了 Outside-TLAB 分配事件。

**口径说明**：这能逐次记录该事件覆盖的 TLAB 外分配，不代表 JVM
中所有对象分配（尤其是 TLAB
内的小对象分配）都被完整记录。对本次重点分析的大型数组分配，它提供了关键证据。

### 5.2 对照实验条件

-   同一请求。
-   响应字节数均为 `77,963,820` bytes。
-   比较旧方案 `writeValueAsString` 路径与新方案对象直传路径。
-   录制文件：
    -   `D:\\Temp\\sse-allocation-string.jfr`
    -   `D:\\Temp\\sse-allocation-direct.jfr`

### 5.3 JFR 数组累计分配结果

  路径                            `byte[]`   `char[]`           合计
  ----------------------------- ---------- ---------- --------------
  旧方案 `writeValueAsString`     713.1 MB   192.6 MB   **905.7 MB**
  新方案直接写流                   56.4 MB    53.8 MB   **110.2 MB**
  差额                            656.7 MB   138.8 MB   **795.5 MB**

这里的 MB 数值沿用本次实验记录的统计口径，不额外换算成 MiB。

### 5.4 两条路径共有的读库分配基线

两边共有的部分主要来自读库本身，约 109 MB：

  共有分配来源                                         约分配量
  ------------------------------ ------------------------------
  Kingbase 解码产生的 `char[]`                          53.4 MB
  `receiveTupleV3` 读缓冲                                 28 MB
  解码压缩出的 `byte[]`                                 27.4 MB
  合计                             **约 108.8 MB（约 109 MB）**

这说明新方案没有消除数据库读取和原始 GeoJSON
数据的成本。两边几乎相同的读库分配基线，是分析优化收益时的重要对照。

### 5.5 旧方案独有的三条分配栈

  ---------------------------------------------------------------------------------------------------------------------------------------------------
  旧方案独有分配路径                        约分配量 证据与含义
  --------------------- ---------------------------- ------------------------------------------------------------------------------------------------
  Spring String 再编码               293 MB `byte[]` `StringHttpMessageConverter.writeInternal → StreamUtils.copy → String.getBytes(UTF-8)`，并包含
                                                     encode 内部 `Arrays.copyOf` 等相关分配

  Jackson 完整 String                364 MB `byte[]` `TextBuffer.contentsAsString` 中 StringBuilder UTF-16 展开、`toString` 最终拷贝及 StringBuilder
  构造                                               内部存储

  Jackson TextBuffer                 135 MB `char[]` `TextBuffer.carr/expand`，序列化期间的字符缓冲及扩容相关分配
  字符缓冲                                           

  合计                                 **约 792 MB** 与总差额约 795.5 MB 接近，约有 3.5 MB 统计差异
  ---------------------------------------------------------------------------------------------------------------------------------------------------

Jackson 完整 String 构造这约 364 MB 的细分记录：

  分配项                            约分配量
  --------------------------- --------------
  StringBuilder UTF-16 展开         145.7 MB
  `toString()` 最终拷贝             145.7 MB
  StringBuilder 内部存储             72.8 MB
  合计                          **364.2 MB**

说明：72.8 MB 是 StringBuilder 内部存储相关分配，不是 StringBuilder
对象头本身占用 72.8 MB。

Spring 再编码一项中，实验记录还提到 `String.getBytes(UTF-8)`
的一条直接分配栈约 218.9 MB；连同 encode 内部 `Arrays.copyOf`
等相关分配，按本次归因口径合并约 293
MB。正式复核时应保留各事件筛选条件和汇总口径，避免把单条栈与归并后的总量混为一谈。

### 5.6 如何理解"四五份整包数据"？

旧方案的序列化和发送过程可能出现多层大块数据表示：

-   TextBuffer 的字符缓冲区及扩容产生的数组；
-   StringBuilder 内部存储；
-   `StringBuilder.toString()` 的最终拷贝；
-   完整 String 的内部 `byte[]`；
-   String 后续 UTF-8 编码产生的 `byte[]`，以及编码实现内部的复制。

这可以概括为"多份完整或大块中间数据表示"。但**累计分配了多份数组，不代表这些数组在同一时刻全部存活**。不要把累计分配量直接描述成同一时刻的内存占用。

## 6. JFR、MAT、Heap Used：三个指标不能混为一谈

### 6.1 JFR 累计分配量

JFR
分配事件反映记录时间窗口内对象分配的过程。累计分配量回答：**这次请求执行期间，某类对象总共分配了多少内存？**
它不是某个时刻仍存活对象的大小。

### 6.2 Heap Used 峰值

Heap Used 是某个采样时刻 JVM
已使用堆的量，受到对象分配、对象生命周期、GC
时机和采样时刻等因素影响。它回答：**某个时刻，堆中有多少内存仍处于已使用状态？**

### 6.3 MAT Heap Dump / Histogram

MAT 分析的是 Heap Dump 对应时刻的对象快照：

-   Histogram：按类统计对象数量和浅堆大小。
-   Dominator Tree：分析对象支配关系。
-   Incoming References：查找直接引用者。
-   Path to GC Roots：查看可达对象的 GC Root 路径。

它回答：**快照时有哪些对象？它们是否可达？谁引用它们？谁支配了大量内存？**

Histogram 中的 `byte[]` 总量不能直接说明所有数组都来自 String；`char[]`
也不能直接认定全部来自 Jackson TextBuffer。

## 7. 两轮历史实验：分别保留，不要混算

### 7.1 2026-10-19：Heap Used 峰值与容器指标

  指标                          整段字符串   直接写入
  --------------------------- ------------ ----------
  SSE Heap Used 峰值                679 MB     255 MB
  SSE Heap Committed                706 MB     285 MB
  容器驻留峰值                     1097 MB   630.3 MB
  一次性返回 Heap Used 峰值         291 MB     302 MB

SSE Heap Used 峰值差：

``` text
679 - 255 = 424 MB
```

这 424 MB 是两次峰值计数器之差，不是某一类对象被精确证明释放了 424 MB。

### 7.2 2026-11-05：Heap Dump 快照与 Histogram

相同响应字节数：`77,963,820` bytes，约 74.4 MiB。

  指标                      整段字符串   直接写入
  ----------------------- ------------ ----------
  Heap Used 快照读数            510 MB     195 MB
  `byte[]` Histogram          307.5 MB    67.7 MB
  `char[]` Histogram          110.1 MB    56.4 MB
  `String` 对象自身浅堆         4.8 MB     5.4 MB

两类数组差额：

``` text
旧方案：307.5 + 110.1 = 417.6 MB
新方案： 67.7 +  56.4 = 124.1 MB
差额：                       293.5 MB
```

Heap Used 快照读数差：

``` text
510 - 195 = 315 MB
```

315 MB 与 293.5 MB 相差约 21.5 MB，不能仅凭这张表进一步归因。

### 7.3 为什么 JFR、Histogram 和 Heap Used 数字不同？

-   JFR：请求时间窗口内的累计分配事件。
-   Histogram：Heap Dump 时刻的对象快照。
-   Heap Used：监控采样时刻的堆使用读数。

JFR 中已经分配、但在 Heap Dump 前已经回收的数组，不会出现在该 Heap Dump
的 Histogram 中。反过来，Heap Dump 中尚未回收的不可达对象可能仍然出现在
Histogram 中。

因此，JFR 旧新差额约 795.5 MB、Heap Used 峰值差 424 MB、Histogram
两类数组差额 293.5 MB，可以同时成立，不应强行要求数值相等。

## 8. MAT 排查过程中的观察与边界

### 8.1 大 byte\[\] 显示 Unreachable

之前 MAT 中发现约 83.95 MB 的 `byte[]`，内容开头是 JSON 文本，且显示
`GC root: Unreachable`。对它执行 Incoming References 后没有找到引用者。

可以确认：

-   该数组在当前快照中不可从 GC Roots 到达；
-   它可能是已经完成使用、等待 GC 回收的临时数组；
-   仅凭内容开头是 JSON，不能确定具体由哪个方法分配；
-   不可达对象没有 GC Root 路径是正常的，MAT
    无法凭快照恢复其历史分配调用栈。

### 8.2 HeapByteBuffer 的 Incoming References

另一次追踪中看到大量 `java.nio.HeapByteBuffer` 的 `hb`
字段指向底层数组。`hb` 是 HeapByteBuffer 引用堆内 `byte[]` 的字段。

多个 ByteBuffer 视图可能共享同一个底层数组，例如通过 `duplicate()` 或
`slice()` 创建视图。因此：

-   3416 条引用记录不等于 3416 份独立的大数组；
-   这些 ByteBuffer 显示 Unreachable 时，引用链可能无法继续追溯到 GC
    Roots；
-   后续记录中，用户确认这批对象只是 Tomcat
    输出缓冲对小数组的包装，采样合计约 2.3 MB，并非本次约 795 MB
    分配差额的来源。

### 8.3 MAT 与 JFR 如何配合

-   MAT：确认对象在快照时的存活状态、引用关系和支配关系。
-   JFR：在下一次复现时记录分配事件及其调用栈。
-   不能用 MAT 的快照反向恢复过去对象的分配栈；也不能仅凭 JFR
    累计分配量断言这些对象同时存活。

## 9. G1 Humongous Object 与 GC 解释

本次大 JSON 相关数组远大于普通小对象。实验环境记录的 G1 Region 大小约为
1 MB，超过 Region 一半（约 512 KB）的对象会被视为 Humongous
Object，并使用连续的 Humongous Region 分配。

需要准确理解：

-   Humongous Object 通常不走普通 TLAB 小对象分配路径；
-   它们可能在请求期间形成较大的堆占用；
-   不可达对象不会在变成不可达的瞬间立即释放；
-   G1 对 Humongous Object 有专门的回收处理，存在 Eager Reclaim 等机制；
-   不应绝对表述为"Young GC 永远不会回收 Humongous Object"；
-   本次 JFR 开启 OutsideTLAB
    事件，正是为了捕获这类大数组的逐次分配证据。

对于连续发送大事件，旧路径产生的多份大数组可能在某个时间窗口内同时占用堆，推高
Heap Used 峰值。但累计分配、同时存活和最终峰值之间仍然需要分别测量。

## 10. 复现实验与复核证据

### 10.1 对照代码

旧方案：

``` java
String json = objectMapper.writeValueAsString(payload);

emitter.send(
    SseEmitter.event()
        .name(name)
        .data(json, MediaType.APPLICATION_JSON)
);
```

新方案：

``` java
emitter.send(
    SseEmitter.event()
        .name(name)
        .data(payload, MediaType.APPLICATION_JSON)
);
```

### 10.2 JFR 录制注意事项

JDK 21 `profile` 配置下，`jdk.ObjectAllocationOutsideTLAB`
默认未启用。本次通过增加以下事件配置重新录制：

``` text
+jdk.ObjectAllocationOutsideTLAB#enabled=true
```

记录文件：

``` text
D:\\Temp\\sse-allocation-string.jfr
D:\\Temp\\sse-allocation-direct.jfr
```

在 JDK Mission Control 中打开 JFR 文件，重点检查：

-   `byte[]`、`char[]` 分配；
-   `String` / StringBuilder 相关分配；
-   `StringHttpMessageConverter.writeInternal`；
-   `StreamUtils.copy`；
-   `String.getBytes(UTF-8)` / encode / `Arrays.copyOf`；
-   `TextBuffer.contentsAsString`；
-   `TextBuffer.carr` / `TextBuffer.expand`；
-   `StringBuilder.append` / `toString`；
-   Jackson `UTF8JsonGenerator` 的输出路径；
-   Kingbase `receiveTupleV3` 和解码路径。

OutsideTLAB 事件不是所有 TLAB
内分配的完整记录。对大数组分配链路有价值，但不要把它描述成完整覆盖 JVM
所有对象分配。

### 10.3 保持实验条件一致

对比旧新路径时，尽量保持：

-   Java 版本和 JVM 参数一致；
-   Spring / Jackson / Kingbase 驱动版本一致；
-   输入数据、框选范围一致；
-   响应字节数一致；
-   并发数和请求顺序一致；
-   只改变 `.data()` 参数类型及其必要代码；
-   记录 Heap Used 曲线、Heap Dump、JFR 和响应字节数；
-   避免同一 JVM 同时运行其他大数据业务。

### 10.4 复核 Converter

检查 `RequestMappingHandlerAdapter` 注册的消息转换器及其顺序，并确认：

-   String + `application/json` 的候选 Converter；
-   VO + `application/json` 的候选 Converter；
-   是否有自定义 Converter；
-   实际事件发送时命中了哪个 Converter。

仅检查 `canWrite()`
和注册顺序可以帮助判断候选转换器；要确认某次发送实际命中的
Converter，仍应结合断点、日志或 JFR 调用栈。

## 11. 最终结论与易错点

### 11.1 正确结论

这次优化没有更换 `SseEmitter`，也没有取消业务 VO 或 JSON
序列化。它通过将 `.data()` 参数从 String 改为 POJO，使 Spring MVC 从
String 消息转换路径切换到 Jackson JSON 消息转换路径。

旧路径先通过 `writeValueAsString()` 构造完整 JSON String，再由
`StringHttpMessageConverter` 将 String 编码为 UTF-8 字节。该过程中产生了
Jackson 字符缓冲、StringBuilder 中间存储、完整 String 内部数组，以及
Spring 后续编码数组等多层分配。

新路径将 POJO 交给 Jackson Converter，由 `UTF8JsonGenerator`
在序列化时直接向 HTTP 输出流写出，避免完整 JSON String 的中间物化和
String 后续整体编码所产生的大量数组分配。

### 11.2 不要这样说

-   不要说"新方案不再序列化 JSON"。
-   不要说"新方案不需要 UTF-8 编码"。
-   不要说"新方案删除了业务结果 VO"。
-   不要说"新方案不再使用 SseEmitter"。
-   不要说"只少了一个 String 对象头"。
-   不要把 JFR 累计分配量差额说成同一时刻的堆存量差额。
-   不要把 2026-10-19 的 424 MB 峰值差说成精确释放的对象大小。
-   不要把 Histogram 中所有 `byte[]` 都认定为 String 内部数组。
-   不要把 Histogram 中所有 `char[]` 都认定为 Jackson TextBuffer。
-   不要说 G1 Young GC 绝对不会回收 Humongous Object。
-   不要把 3416 个 HeapByteBuffer 说成 3416 份大数组。

## 12. 面试讲述版本

### 12.1 约 1 分钟版本

我在一个基于 Spring MVC `SseEmitter` 的大数据 SSE 接口中，发现大框
GeoJSON 返回时 JVM 堆内存峰值较高。排查后发现，原实现先通过 Jackson 的
`writeValueAsString()` 把业务 VO 序列化成完整 JSON String，再把 String
传给 `SseEmitter`。Spring 收到 String 后选择
`StringHttpMessageConverter`，又将完整字符串编码为 UTF-8
字节写入响应流。

我将发送方式改为直接传递 VO 对象，保留 `SseEmitter`，让 Spring 选择
Jackson JSON Converter，由 `UTF8JsonGenerator`
在序列化时直接向响应输出流写出。这样避免了完整 JSON String
的中间物化，以及后续 String 整体编码产生的大量数组分配。

为了定位原因，我使用 JFR 记录对象分配调用栈，并结合 MAT
分析堆快照。在相同请求、相同响应字节数（77,963,820 bytes）下，旧方案
`byte[]` 和 `char[]` 累计分配约 905.7 MB，新方案约 110.2 MB，差额约
795.5 MB。分配栈主要集中在 Spring String 再编码、Jackson 完整 String
构造和 TextBuffer 字符缓冲三个环节。

### 12.2 追问：为什么同样是 SseEmitter，内存差这么多？

因为 SseEmitter 的 SSE 发送能力没有变，但 `.data()` 收到的 Java
类型变了。传 String 时由 StringHttpMessageConverter
处理，先前已经生成的完整 JSON String 又被整体编码为 UTF-8 字节；传 POJO
时由 Jackson JSON Converter 处理，Jackson 可以直接使用 JsonGenerator 向
OutputStream 写出。差异在消息转换器和序列化输出路径，不在 SseEmitter
本身。

### 12.3 追问：VO 不是仍然存在吗？

是的。VO 在两条路径中都存在，也都需要被序列化一次。原始 GeoJSON String
及其底层数组同样仍然存在。优化消除的是旧路径额外创建的完整 JSON String
及其字符缓冲、StringBuilder 中间存储和后续 UTF-8
编码数组，不是删除原始业务数据。

### 12.4 追问：直接写流还需要编码吗？

需要。HTTP 网络传输的是字节，JSON 文本仍然要编码。区别是旧方案先把整个
JSON 物化为 String，再整体编码成 UTF-8 字节；新方案由 Jackson 在生成
JSON 时直接编码并写入输出流，避免完整 String 这一中间表示。

### 12.5 追问：怎么证明不是猜测？

我用相同请求和相同响应字节数做了新旧对照，并通过 JFR 的 OutsideTLAB
分配事件检查具体分配调用栈。旧方案的额外分配主要落在
`StringHttpMessageConverter` 的 String 编码路径、Jackson
`TextBuffer.contentsAsString` / StringBuilder 完整字符串构造路径，以及
`TextBuffer` 的字符缓冲路径。MAT
则用于核对快照时对象的数量、大小和可达性。JFR 解释分配来源，MAT
解释快照对象构成，两者互相补充。

## 13. 最终记忆模型

1.  `SseEmitter` 没换，业务 VO 没删，JSON 序列化仍然发生。
2.  `.data(String, APPLICATION_JSON)` 与 `.data(POJO, APPLICATION_JSON)`
    会触发不同的消息转换器路径。
3.  旧路径先完整物化 JSON String，再由 String Converter 编码成 UTF-8
    字节。
4.  新路径由 Jackson JSON Converter 使用 Generator
    直接向响应输出流写出。
5.  原始 GeoJSON String 在两条路径中都存在；被省掉的是完整 JSON String
    及其相关中间数据分配。
6.  JFR 累计分配量、MAT 快照对象总量、Heap Used 峰值是不同指标。
7.  JFR 负责定位分配调用栈，MAT 负责分析快照时对象与引用关系。
8.  本次核心优化不是"少一个 String
    对象"，而是"从完整字符串物化与二次编码，改为序列化时直接写输出流"。

------------------------------------------------------------------------

## 附：本次实验数据速查

  ------------------------------------------------------------------------------------------------------------------
  项目                                数值 / 说明
  ----------------------------------- ------------------------------------------------------------------------------
  响应字节数                          77,963,820 bytes

  JFR 旧方案 `byte[]`                 713.1 MB

  JFR 旧方案 `char[]`                 192.6 MB

  JFR 旧方案合计                      905.7 MB

  JFR 新方案 `byte[]`                 56.4 MB

  JFR 新方案 `char[]`                 53.8 MB

  JFR 新方案合计                      110.2 MB

  JFR 累计分配差额                    约 795.5 MB

  共有读库分配基线                    约 109 MB

  旧方案 Spring 再编码                约 293 MB `byte[]`

  旧方案 Jackson 完整 String 构造     约 364 MB `byte[]`

  旧方案 Jackson TextBuffer           约 135 MB `char[]`

  2026-10-19 Heap Used 峰值           679 MB → 255 MB，差 424 MB

  2026-11-05 Heap Used 快照读数       510 MB → 195 MB，差 315 MB

  2026-11-05 Histogram 数组差额       `byte[]` 239.8 MB + `char[]` 53.7 MB = 293.5 MB

  JFR 文件                            `D:\\Temp\\sse-allocation-string.jfr`、`D:\\Temp\\sse-allocation-direct.jfr`

  部署记录                            服务器已换回原包；MD5 前缀 `0ab70857...`；业务码 200；仅涉及
                                      `analysis-service`
  ------------------------------------------------------------------------------------------------------------------

> 本文中的 JFR 栈归因和数值来自本次实验记录；对尚未由原始 JFR
> 文件或调用栈截图独立复核的细分项，应保留"约""本次归因口径"等限定。不同日期的实验数据分别记录，不应拼接成同一次实验。
