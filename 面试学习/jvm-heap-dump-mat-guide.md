# Java JVM 堆快照（Heap Dump）与 Eclipse MAT 内存排查完整笔记

> 适用：Java 17/21、Spring Boot、Windows PowerShell、Linux、Docker / Docker Compose。  
> 目标：从“发现内存异常”到“拿到 `.hprof` → 分析对象引用图 → 定位源码 → 修复验证”。  
> 文中的 PID、容器名称、目录和内存数字均为示例，操作时应替换为真实值。

## 目录

1. [核心概念：源码、堆快照和 MAT 的关系](#1-核心概念源码堆快照和-mat-的关系)
2. [堆快照能看见什么，不能看见什么](#2-堆快照能看见什么不能看见什么)
3. [重要术语：GC Roots、引用链、Shallow/Retained Heap](#3-重要术语gc-roots引用链shallowretained-heap)
4. [排查前先判断是不是 Java 堆问题](#4-排查前先判断是不是-java-堆问题)
5. [Windows 本地 Spring Boot：完整命令](#5-windows-本地-spring-boot完整命令)
6. [Linux 服务器：完整命令](#6-linux-服务器完整命令)
7. [Docker：生成并拷贝堆快照](#7-docker生成并拷贝堆快照)
8. [提前配置 OOM 自动生成堆快照](#8-提前配置-oom-自动生成堆快照)
9. [Eclipse MAT：图形界面实操](#9-eclipse-mat图形界面实操)
10. [MAT 命令行批量报告](#10-mat-命令行批量报告)
11. [完整案例：排查不断增长的缓存](#11-完整案例排查不断增长的缓存)
12. [让 AI 结合分析报告与源码定位问题](#12-让-ai-结合分析报告与源码定位问题)
13. [常见错误与生产环境注意事项](#13-常见错误与生产环境注意事项)
14. [命令速查表与标准排查流程](#14-命令速查表与标准排查流程)
15. [官方文档](#15-官方文档)

---

## 1. 核心概念：源码、堆快照和 MAT 的关系

### 1.1 为什么不只分析源码？

- **源码分析**：找出可能造成内存占用或泄漏的代码逻辑，如无界集合、静态缓存、ThreadLocal 未清理、监听器未注销等。
- **堆快照（Heap Dump）**：保留 JVM 某个时刻的堆对象信息，包括类型、实例、字段及引用关系。典型文件后缀是 `.hprof`。
- **MAT（Eclipse Memory Analyzer Tool）**：解析堆快照，构建对象引用图，统计对象数量、内存占用、支配关系、GC Roots 路径，并生成疑似泄漏报告。

关系：

```text
Java 源码 ────────────────→ 解释为什么出现问题、修改逻辑
   ▲
   │ 对照类名、字段和引用链
   │
MAT / IDE / VisualVM 等 ──→ 解析、查询、可视化
   ▲
   │ 读取
   │
heap.hprof ──────────────→ 某一时刻的运行时堆数据
   ▲
   │ JVM 导出
   │
正在运行的 Java 进程
```

**结论：可以不使用 MAT，但不能仅靠文本编辑器直接有效阅读二进制 HPROF；需要某种解析工具。**

### 1.2 什么情况值得抓快照？

| 情况 | 优先动作 |
|---|---|
| 一眼能看出无界 `List` 不断追加 | 先修源码并复现验证 |
| 老年代长期增长，GC 后仍高 | 监控 GC，再抓快照 |
| 偶发 `OutOfMemoryError: Java heap space` | 提前开启自动 HPROF，保留现场 |
| 某接口突然变慢、GC 频繁 | 先看 GC、分配速率与负载，必要时抓快照 |
| 进程 RSS 很大，但 Java 堆使用不高 | 排查直接内存、线程栈、Metaspace、native memory，不要只看堆快照 |
| 非堆类型 OOM（如 `Direct buffer memory`） | 使用对应诊断工具；HPROF 可能只能提供部分线索 |

---

## 2. 堆快照能看见什么，不能看见什么

### 能看见

1. `String`、`HashMap`、`ArrayList`、`byte[]` 等对象的**实例数量**与大小。
2. 对象的字段值、数组元素，以及**谁引用谁**。
3. 深层路径，例如 `Bean → Map → Entry → List → byte[]`。
4. 哪些对象能从 **GC Roots** 到达（因此不能按普通不可达对象回收）。
5. 一个对象“支配”了多大内存，以及多条引用路径的关系。

### 不能直接得出

1. 不能自动、可靠地定位**哪一行源码执行了对象分配**；需要 JFR、分配采样或运行时 profiler。
2. 单份快照**不能证明“持续增长”**；最好结合一段时间的 GC/堆曲线，必要时对比两份快照。
3. 对象大不等于泄漏；可能是正常业务缓存、批处理或某个请求仍在执行。
4. HPROF 主要针对 **Java 堆**；不等同于进程所有内存的完整快照。
5. 快照不是执行录屏，不包含完整历史操作链、所有调用堆栈或每次对象创建时刻。

---

## 3. 重要术语：GC Roots、引用链、Shallow/Retained Heap

### 3.1 GC Roots（垃圾回收根）

GC 需要从一组根出发做可达性分析。常见根相关来源包括：

- 线程栈中仍被使用的局部变量引用；
- 类的静态字段所关联的可达对象；
- JNI 和其他 JVM 内部维护的根引用。

示例：

```text
GC Root（与静态字段相关）
    ↓
UserCache.cache : HashMap
    ↓
HashMap.Entry
    ↓
UserData
    ↓
history : ArrayList
    ↓
byte[]
```

**重点：** 堆快照可以追溯很深的引用链，并非只能看一层或两层。但某个对象可能存在多条引用路径，解除其中一条引用也不一定能释放对象。

### 3.2 Shallow Heap 与 Retained Heap

- **Shallow Heap**：对象自身占用的堆空间，不算它引用的其他对象。
- **Retained Heap**：如果这个对象不再作为可达路径的一部分，理论上随之变得不可达的那一组对象所占堆空间（MAT 根据支配关系计算）。

假设：

```text
UserCache 本体：             48 B  （Shallow Heap）
UserCache 支配的缓存数据：  800 MB （Retained Heap）
```

应重点检查 `Retained Heap` 很大的对象及其所有者，而不是只看实例自身大小。

**注意：** Retained Heap 不是“该对象直接引用的所有对象大小之和”。如果另一个独立 GC Root 仍能到达某个子对象，它就不一定属于当前对象的 retained set。

### 3.3 Dominator Tree（支配树）是什么？

如果**从根到对象 B 的每条路径都经过对象 A**，就说 A 支配 B。MAT 将对象引用图转换成支配树，方便找到“大量内存究竟由哪个入口保留”。

- **引用图**：真实的对象引用边，可能一对多、多对一、成环。
- **支配树**：由支配关系计算出来的树，树上的边**不一定是一条直接的 Java 字段引用**。

### 3.4 Incoming / Outgoing / Path to GC Roots

| 功能 | 回答的问题 |
|---|---|
| Incoming References | **谁引用了我？** |
| Outgoing References | **我引用了谁？** |
| Path to GC Roots | **对象为什么还活着？** |
| Dominator Tree | **哪个入口支配了最大的一批内存？** |
| Histogram | **哪些类的对象数量多/总占用大？** |
| Leak Suspects | **有哪些值得先看的疑似保留热点？** |

`Leak Suspects` 是启发式报告，不代表它已经证明存在真正泄漏。

---

## 4. 排查前先判断是不是 Java 堆问题

### 4.1 看 JVM 与进程

```bash
jcmd -l                         # 列出可连接的 Java 进程
jcmd <PID> VM.command_line      # JVM 启动命令和参数
jcmd <PID> VM.flags             # 运行时 JVM 标志
jcmd <PID> GC.heap_info         # 堆信息
jstat -gcutil <PID> 1000 10     # 每秒采样一次，共 10 次
```

`jstat -gcutil` 重点关注：

- `O`：老年代使用比例（具体列及含义与 JDK/GC 有关）；
- `YGC` / `FGC`：年轻代 / Full GC 次数；
- `GCT`：累计 GC 耗时。

**不能只因为 O 值高就认定内存泄漏。** 要结合负载、垃圾回收行为、吞吐与时间趋势。

### 4.2 区分不同类型内存异常

| 现象/错误 | 可能方向 |
|---|---|
| `OutOfMemoryError: Java heap space` | Java 堆不足、对象保留过多、数据批量过大 |
| `GC overhead limit exceeded` | GC 花费过高，释放空间有限；与堆压力相关 |
| `OutOfMemoryError: Metaspace` | 类元数据、类加载器等 |
| `OutOfMemoryError: Direct buffer memory` | NIO 直接内存、缓冲区生命周期 |
| 容器直接被 `OOMKilled` / exit 137 | 容器内存限制触发，可能来不及生成 JVM 堆快照 |
| RSS 远大于 Java Heap Used | 线程栈、直接内存、JIT code cache、native allocations 等 |

如果是本地内存异常且启动时已经启用 NMT，可以进一步使用：

```bash
jcmd <PID> VM.native_memory summary
```

> NMT 通常需要在 JVM 启动时设置 `-XX:NativeMemoryTracking=summary` 或 `detail`；它不是默认随时可用的。

---

## 5. Windows 本地 Spring Boot：完整命令

下面均在 PowerShell 执行。

### 第 1 步：确认 JDK 工具可用

```powershell
java -version
jcmd -l
```

如果 `jcmd` 找不到，请检查 `JAVA_HOME` 和 PATH，并使用安装的 JDK（而非仅有 JRE 的运行环境）：

```powershell
& "$env:JAVA_HOME\bin\jcmd.exe" -l
```

### 第 2 步：找到项目 PID

```powershell
jcmd -l
```

示例输出：

```text
12345 com.example.Application
18236 org.jetbrains.idea.Main
```

假设 Spring Boot 的 PID 是 `12345`。

### 第 3 步：观察堆与 GC

```powershell
jcmd 12345 VM.command_line
jcmd 12345 GC.heap_info
jstat -gcutil 12345 1000 10
```

### 第 4 步：准备目录并生成堆快照

```powershell
New-Item -ItemType Directory -Path D:\heap -Force
jcmd 12345 GC.heap_dump D:\heap\app.hprof
```

**重要：** 一般情况下，`GC.heap_dump` 在生成快照前会请求 Full GC，可能让应用暂停；请在适当时机执行。

如果想保留包括不可达对象在内的所有对象，可在支持该选项的 JDK 上用：

```powershell
jcmd 12345 GC.heap_dump -all D:\heap\all-objects.hprof
```

`-all` 通常用于特定诊断目的；**常规泄漏排查优先使用默认命令**，便于聚焦 GC 后保留的对象。

### 第 5 步：确认文件生成

```powershell
Get-Item D:\heap\app.hprof | Select-Object FullName,Length,LastWriteTime
```

文件通常不小，取决于堆对象的大小及生成方式。

---

## 6. Linux 服务器：完整命令

```bash
# 1. Java 进程
jcmd -l

# 2. 观察（将 12345 替换为真实 PID）
jcmd 12345 GC.heap_info
jstat -gcutil 12345 1000 10

# 3. 准备一个 Java 进程用户可写、容量足够的目录
mkdir -p /tmp/jvm-dumps

# 4. 导出快照
jcmd 12345 GC.heap_dump /tmp/jvm-dumps/app.hprof

# 5. 检查
ls -lh /tmp/jvm-dumps/app.hprof
```

如果报 `Permission denied` 或 attach 失败，确认运行命令的账号与 Java 进程账号、操作系统 attach 权限、容器隔离等条件。

远程拷贝到本地的常见方式：

```bash
# 在本机终端执行；主机名与路径按实际替换
scp user@server:/tmp/jvm-dumps/app.hprof ./app.hprof
```

---

## 7. Docker：生成并拷贝堆快照

### 方案 A：容器内已经有 `jcmd`（最简单）

Windows PowerShell：

```powershell
# 1. 找容器
docker ps

# 2. 查看 Java 进程
docker exec java-service jcmd -l

# 3. 假设 Java PID 是 1：在容器内部生成文件
docker exec java-service jcmd 1 GC.heap_dump /tmp/app.hprof

# 4. 拷贝到 Windows
New-Item -ItemType Directory -Path D:\heap -Force
docker cp java-service:/tmp/app.hprof D:\heap\app.hprof

# 5. 检查
Get-Item D:\heap\app.hprof
```

这里 `1` **只是示例**，必须使用容器内实际的 Java PID。

### 方案 B：容器中没有 `jcmd`

很多生产镜像只包含 JRE 或裁剪版运行时，执行命令可能得到：

```text
exec: "jcmd": executable file not found in $PATH
```

可选办法：

1. **推荐提前配置 OOM 自动生成**，并把输出目录挂载到宿主机（见下一章）。
2. 在开发/排障环境改用包含 JDK 诊断工具的镜像重新部署。
3. 经评估后使用具备兼容工具、权限及相应命名空间的诊断容器；不能假设随便起一个 JDK 容器就能 attach 到目标 JVM。

### 方案 C：Docker Compose 提前挂载 dump 目录

```yaml
services:
  java-service:
    image: your-java-image:latest
    environment:
      JAVA_TOOL_OPTIONS: >-
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/app/dumps
    volumes:
      - ./dumps:/app/dumps
    # 若容器内存受限，还需合理配置 JVM 堆与容器内存限额
```

**解释：** Java 堆快照先生成在容器路径 `/app/dumps`，宿主机的 `./dumps` 会接收到文件。挂载目录必须存在/可写，且有足够空间。

### 生产环境额外注意

- `docker exec` 进入的容器可能没有 `bash`，但有 `sh`；也可能连 JDK 工具都没有。
- `/tmp` 不一定有足够空间，尤其是临时文件系统或受限容器。
- 容器进程如果被内核 OOM killer 直接杀死，JVM 可能**没有机会**写出自动 heap dump。
- Java 堆之外也要留容器内存：Metaspace、Direct Memory、线程栈、JIT 和 JVM 自身开销。

---

## 8. 提前配置 OOM 自动生成堆快照

### 8.1 启动命令（Linux 示例）

```bash
java \
  -Xms512m \
  -Xmx2g \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/dumps \
  -jar app.jar
```

参数：

| 参数 | 意义 |
|---|---|
| `-Xms512m` | 初始 Java 堆容量 |
| `-Xmx2g` | 最大 Java 堆容量 |
| `-XX:+HeapDumpOnOutOfMemoryError` | 在支持的 JVM OOM 场景下尝试导出 HPROF |
| `-XX:HeapDumpPath=/app/dumps` | dump 目标目录或文件路径 |

### 8.2 Windows IDEA 中配置

在 **Run/Debug Configurations → VM options** 加入：

```text
-Xms512m -Xmx2g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=D:\heap
```

提前创建目录并确保进程对其有写权限。

### 8.3 自动快照的局限

- 自动 HPROF 不代表所有 OOM 场景都能成功落盘；尤其是容器强制杀进程、磁盘满、目录权限错误。
- 堆快照可能包含真实业务数据和敏感信息，按生产数据级别保护。
- 堆快照写出会产生磁盘 I/O 与额外负载；不要把 dumps 文件夹放在容量很小的系统分区。

---

## 9. Eclipse MAT：图形界面实操

下载：<https://eclipse.dev/mat/>

### 9.1 打开 HPROF

1. 启动 MAT。
2. 点击 **File → Open Heap Dump**，选择 `D:\heap\app.hprof`。
3. 等待索引构建；大型 HPROF 的第一次导入可能非常耗时，并占用较多磁盘和 MAT 自身内存。
4. 如果出现报告选择界面，可以先运行 **Leak Suspects Report**。

### 9.2 第一步：Overview / Leak Suspects

- 先看堆总量、Top Consumers、疑似热点。
- 优先排查某个业务类、缓存集合、ClassLoader 或线程相关对象是否异常地保留大量堆。
- **不要直接认定报告中的“suspect”就是 bug**；还要验证业务场景。

### 9.3 第二步：Histogram（按类统计）

1. 打开 **Histogram**。
2. 搜索 `byte[]`、`char[]`、`java.lang.String`、`java.util.HashMap` 或自己的业务类。
3. 查看实例数、Shallow Heap；必要时计算/显示 Retained Heap。
4. 如果某类对象多，右键查看实例及引用关系，进一步查谁持有它们。

**注意：** `byte[]` 很大并不等于数组泄漏；真正应检查的是数组所属缓存、业务对象、线程或请求。

### 9.4 第三步：Dominator Tree（按保留内存定位）

1. 打开 **Dominator Tree**。
2. 按 **Retained Heap** 降序查看。
3. 展开异常条目：`Map → table → Node/Entry → value → List → byte[]`。
4. 找到第一个具有业务含义的对象/字段，如 `UserService.cache`。

不要误把 Dominator Tree 中每一级都理解为**直接字段引用链**；它表达的是支配关系。需要确认直接引用时，再使用 Incoming/Outgoing References。

### 9.5 第四步：Path to GC Roots（为什么没被回收）

1. 选中可疑对象实例，右键。
2. 选择 **Path To GC Roots**。
3. 常规排查可先选 **exclude all phantom/weak/soft etc. references** 一类过滤选项，聚焦强引用。
4. 沿路径向上查找业务侧的 `static` 字段、Spring 单例 Bean、线程、ThreadLocal、消息队列消费者等。
5. 如果怀疑过滤掉了重要路径，再切换不同的引用过滤条件查看。

示例：

```text
GC Root
  └─ Spring ApplicationContext / 单例 Bean
       └─ userService
            └─ cache : HashMap
                 └─ value : UserData
                      └─ history : ArrayList
                           └─ byte[]
```

此时应去代码中寻找 `cache.put(...)`、过期清理、`remove(...)`、容量限制和生命周期管理。

### 9.6 第五步：Incoming / Outgoing References

- **Incoming References**：当前对象被谁引用。
- **Outgoing References**：当前对象的字段/数组元素引用了什么。

对于复杂场景，先用 `Dominator Tree` 发现热点，再用 `Incoming References + Path to GC Roots` 追溯实际对象持有者。

### 9.7 MAT 自身内存不足怎么办？

MAT 解析大型堆快照时也需要大量内存。可在安装目录的 `MemoryAnalyzer.ini` 中，按机器内存适当调大 MAT 的 `-Xmx`，例如：

```text
-vmargs
-Xmx4g
```

> 保留原有配置，只修改已有 `-Xmx` 项；别盲目把 `-Xmx` 设得超过可用内存。不同版本安装结构略有区别。

---

## 10. MAT 命令行批量报告

适合不想每次打开 GUI，或者希望把报告交给 AI 的场景。

### Windows PowerShell

假设 MAT 安装在 `D:\tools\mat`（实际路径需确认 `ParseHeapDump.bat` 存在）：

```powershell
& "D:\tools\mat\ParseHeapDump.bat" `
  "D:\heap\app.hprof" `
  org.eclipse.mat.api:suspects
```

生成堆概览报告：

```powershell
& "D:\tools\mat\ParseHeapDump.bat" `
  "D:\heap\app.hprof" `
  org.eclipse.mat.api:overview
```

### Linux

```bash
./ParseHeapDump.sh /data/heap/app.hprof org.eclipse.mat.api:suspects
./ParseHeapDump.sh /data/heap/app.hprof org.eclipse.mat.api:overview
```

常见报告标识：

| 报告 ID | 内容 |
|---|---|
| `org.eclipse.mat.api:suspects` | 内存泄漏嫌疑报告 |
| `org.eclipse.mat.api:overview` | 堆概况、线程、Top Consumers、Histogram 等 |
| `org.eclipse.mat.api:top_components` | 大型内存组件分析 |

**输出**：通常为 HTML 报告的 ZIP 包，文件名和具体位置以 MAT 的控制台输出及版本为准。先解压并打开其中的 HTML 文件。

如果希望进一步分析实例级引用链，GUI 通常比仅有 HTML 概览更适合。

官方批处理文档：<https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/tasks/batch.html>

---

## 11. 完整案例：排查不断增长的缓存

### 11.1 可疑代码

```java
@Service
public class UserService {
    private final Map<Long, List<byte[]>> cache = new ConcurrentHashMap<>();

    public void addChunk(long userId, byte[] chunk) {
        cache.computeIfAbsent(userId, id -> new ArrayList<>()).add(chunk);
    }
}
```

这段代码只是用于演示潜在的无界保留：

- `UserService` 是 Spring 单例 Bean，生命周期通常很长。
- `cache` 可能随着用户数和上传块数量持续增长。
- 没有容量上限、过期策略或删除逻辑。
- 多线程同时修改同一个 `ArrayList` 还有并发安全风险，但这与堆快照排查是不同问题。

### 11.2 运行时排查步骤

1. 压测/执行一段持续添加数据的工作负载。
2. 使用 `jstat -gcutil` 观察 GC 后堆是否仍持续增加。
3. 运行 `jcmd <PID> GC.heap_dump /path/app.hprof`。
4. 在 MAT 的 Histogram 中发现大量 `byte[]`。
5. 在 Dominator Tree 中定位保留它们的集合或业务对象。
6. 用 Path to GC Roots 查到 `UserService.cache`。
7. 回到源码分析数据生命周期和清理策略。

**示意（非真实结果）：**

```text
ApplicationContext
    └── UserService
         └── cache (ConcurrentHashMap)
              └── List<byte[]>
                   └── 大量 byte[]
```

### 11.3 修复思路

按业务需求选择：

- 缓存条目设置最大容量、TTL、过期策略，或直接使用成熟缓存方案（如 Caffeine）。
- 上传结束后及时释放会话中的块引用，不无限保存历史内容。
- 大文件尽量采用流式处理，避免整个文件长期留在 Java 堆中。
- 并发写入同一用户对应的集合时，使用安全的同步/并发模型。

### 11.4 如何证明修好了？

不要只看“修改后没再出现 OOM”。更可靠的验证：

1. 固定 JDK、JVM 参数、负载模式、测试时长，做修复前/后对照。
2. 比较 GC 后堆基线，以及稳定负载时堆是否趋于稳定。
3. 如有必要，在相近的运行阶段抓两份 HPROF 对比实例数量和 retained memory。
4. 验证正常业务请求结果、缓存命中率和性能是否受影响。

注意：`jcmd GC.heap_dump` 默认会请求 Full GC，前后快照的 GC 状态与取样时机应尽量一致。

---

## 12. 让 AI 结合分析报告与源码定位问题

### 12.1 推荐链路

```text
JVM / Docker
  ↓ jcmd / OOM 自动生成
heap.hprof
  ↓ MAT GUI 或批处理
Leak Suspects + Histogram + Dominator Tree + GC Roots 证据
  ↓ 提供给 AI
AI 对照类名、字段、源码路径、缓存/线程/对象生命周期
  ↓
输出可疑逻辑 → 复现步骤 → 最小修复方案 → 验证指标
```

### 12.2 可复制的分析提示词

```text
请作为 Java/JVM 内存排查工程师，根据我提供的 MAT 报告、对象引用链和项目源码进行分析。

要求：
1. 先区分“已证实的运行时事实”和“推测的代码问题”。
2. 按 Retained Heap、实例数量和 GC Roots 路径排序嫌疑对象。
3. 逐条找出关联 Java 类、字段、对象生命周期以及可能的 put/add/remove/clear 路径。
4. 区分真正泄漏、预期缓存、瞬时峰值、堆外内存问题。
5. 给出最小修改建议、回归测试和修复前后可观测指标。
6. 缺少足够证据时，告诉我下一步该在 MAT 中查看什么，不要臆断源代码行号。
```

**隐私与安全：** HPROF 可能含 Token、密码、用户信息、请求正文等敏感数据。不要直接上传到不受信任的外部服务；尽量先在本地分析，仅分享脱敏后的报表、类名、引用链和必要的代码片段。

---

## 13. 常见错误与生产环境注意事项

| 问题 | 可能原因 | 处理 |
|---|---|---|
| `jcmd` 找不到 | JRE/精简运行时、PATH 不对 | 使用 JDK，或设置 `JAVA_HOME/bin` |
| `Unable to attach` | 进程用户/权限、命名空间隔离、JVM attach 受限 | 检查权限与运行环境，尽量用同一用户执行 |
| `Permission denied` | dump 目录无写权限 | 改成进程用户可写的路径 |
| `No such file or directory` | 容器内部路径不存在、文件尚未生成 | 在实际 JVM 所在文件系统内确认路径 |
| dump 特别大 | 对象多、堆占用高 | 预留空间，选择合适时机；不要随意压缩/移动正在写的文件 |
| MAT 打开时 OOM | MAT 解析内存不足 | 调大 MAT 自身 `-Xmx`，关闭不必要程序 |
| MAT 显示一个大 `byte[]` | 仅是对象数据类型 | 找其实际引用者和业务生命周期 |
| 自动 dump 没生成 | 被容器强杀、没有权限/空间、OOM 类型不适用 | 同时检查容器状态、JVM 日志和宿主机限制 |

### 必须记住的风险

- 生成堆快照是**高影响诊断操作**，可能引发停顿、I/O 峰值、磁盘压力。
- 对生产服务先评估流量窗口、磁盘空间、恢复预案，必要时在副本上操作。
- 快照生成过程本身可能改变对象状态或触发 Full GC；抓取时机很重要。
- 静态对象“持有数据多”、或者 MAT 报告提示“suspect”，**均不自动等于泄漏**。
- JVM 堆参数 `-Xmx` 不等于容器总内存，不能把容器内存全分给 Java 堆。

---

## 14. 命令速查表与标准排查流程

### Windows 本地最短链路

```powershell
jcmd -l
jcmd 12345 GC.heap_info
jstat -gcutil 12345 1000 10
New-Item -ItemType Directory -Path D:\heap -Force
jcmd 12345 GC.heap_dump D:\heap\app.hprof
```

### Docker 最短链路

```powershell
docker ps
docker exec java-service jcmd -l
docker exec java-service jcmd 1 GC.heap_dump /tmp/app.hprof
New-Item -ItemType Directory -Path D:\heap -Force
docker cp java-service:/tmp/app.hprof D:\heap\app.hprof
```

> PID `12345` 和 `1`、容器 `java-service` 均要替换；容器里必须有可用的 JDK 诊断工具。

### MAT 最短链路

```text
File → Open Heap Dump
  ↓
Leak Suspects / Overview：先找大内存块
  ↓
Histogram：按类看实例数量
  ↓
Dominator Tree：按 Retained Heap 找支配者
  ↓
Path to GC Roots：找为何无法回收
  ↓
Incoming/Outgoing References：核对真实字段引用
  ↓
定位 Java 源码 → 修复 → 同负载复测
```

### 标准排查判断树

```text
发现 JVM/容器内存异常
       │
       ├─ 先确认错误类型与 JVM/容器内存指标
       │
       ├─ 确认主要是 Java 堆？
       │       ├─ 否 → NMT / JFR / 容器与系统内存工具
       │       └─ 是
       │            ├─ 源码能直接定位并复现？
       │            │       ├─ 是 → 修改 + 验证
       │            │       └─ 否 → 导出 HPROF
       │            └─ MAT：Histogram → Dominator Tree → GC Roots
       │                           ↓
       │                      结合源码修复
       └───────────────────────────↓
                            复测 GC、堆基线、性能
```

### 面试可用的精简回答

> Java 内存异常我会先通过 JVM 参数、GC 日志以及 `jstat` 或监控判断是否属于 Java 堆问题。对于持续增长且难以定位的情况，使用 `jcmd <PID> GC.heap_dump` 或 OOM 自动生成 HPROF，通过 MAT 的 Histogram 找异常类型、Dominator Tree 找主要 retained memory，再用 Path to GC Roots 追踪对象为什么仍可达，最后结合源码定位缓存、线程或集合等对象生命周期问题，并通过一致负载的前后对照验证修复效果。

---

## 15. 官方文档

- [Eclipse MAT 官网](https://eclipse.dev/mat/)
- [MAT：Batch mode 批量分析](https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/tasks/batch.html)
- [MAT：Path to GC Roots](https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/reference/inspections/path_to_gc_roots.html)
- [MAT：Dominator Tree](https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/concepts/dominatortree.html)
- [MAT：Leak Suspects Report](https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/tasks/runningleaksuspectreport.html)
- [Oracle JDK 21：jcmd 手册](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)

> 说明：不同 JDK/MAT 版本的选项名称、启动脚本目录、菜单细节可能有细微差异，执行前以当前版本帮助信息为准。特别是 `jcmd <PID> help GC.heap_dump` 可查看本机支持的选项。
