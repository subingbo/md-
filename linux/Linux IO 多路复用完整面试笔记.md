# Linux IO 多路复用：select、poll、epoll 完整面试笔记

> 适用方向：Java 后端 / Linux / Redis / Nginx 面试  
> 核心问题：**10 万个 TCP 连接中只有 100 个活跃，服务器怎样高效找到这些就绪连接？**  
> 主要依据：上传的《大厂后端面试真题-小林coding》「select、poll、epoll 区别」「epoll 具体工作流程」「Redis 为什么快」章节。文中的概念辨析、代码、复杂度边界、LT/ET 和 Java NIO 为额外的技术补充与澄清。

---

## 一、先弄懂四个概念：TCP 连接、Socket、fd、线程

### 1. TCP 连接是什么？

客户端和服务端经过 TCP 三次握手后形成的**逻辑通信关系**。在通常的 TCP 服务端模型里，一个已经建立的客户端连接，对应服务端一个**已连接 Socket**。

### 2. Socket 是什么？

Socket 是程序进行网络通信的端点，不是线程。

服务端一般有两类 Socket：

- **监听 Socket（listening socket）**：调用 `bind()`、`listen()`，负责等待新的连接；它一般不会直接承载某个客户端连接的业务数据。
- **已连接 Socket（connected socket）**：`accept()` 成功后获得的 Socket，负责和对应客户端收发数据。

```text
                        服务端
                  监听 Socket（fd=3）
                          │
                     accept() 接入
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
       已连接 Socket A  已连接 Socket B  已连接 Socket C
           fd=4           fd=5           fd=6
            │              │              │
         客户端 A         客户端 B         客户端 C
```

注意：IO 多路复用中的“监听”，指的是**监控 fd 是否就绪**（可读、可写等），不等于 TCP 的 `listen()`。

### 3. fd（文件描述符）是什么？

fd 是 Linux **一个进程内**用来索引已打开资源的**非负整数编号**。Socket、普通文件、管道等都可以通过 fd 访问。

```text
进程的文件描述符表
  fd=0   → 标准输入
  fd=1   → 标准输出
  fd=2   → 标准错误
  fd=3   → 普通文件
  fd=4   → Socket A
  fd=5   → Socket B
  fd=6   → Socket C
```

关键区分：

- **Socket 数量** ≠ **最大 fd 编号**。
- 比如只有 3 个需要监控的 Socket，它们的 fd 可以是 `5、8、10000`（现实中高编号意味着进程的 fd 分配历史或其他 fd 占用，并不是“只有这 3 个资源”）。
- fd 是进程内编号，不是全球唯一的 Socket ID；不同进程可以都有 `fd=5`，但指向不同的对象。
- 一个进程内的多个线程通常共享其文件描述符表。

### 4. 线程是什么？

线程是执行代码、参与 CPU 调度的单位。**线程不是文件，也不是 Socket，更不是 fd。**

Linux 的“一切皆文件”强调许多资源提供统一的文件接口，并不意味着线程也被当作文件描述符来读写。

```text
一个线程 ──操作──> 多个 Socket（通过 fd）
```

传统“一个连接一个线程”只是编程模型，并非 Socket 必须和线程一一对应。

---

## 二、什么是 IO 多路复用？

**同一个线程等待多个 fd 的 IO 就绪事件，只处理已经就绪的 fd。**

例如，服务器监控 10 万个已连接 Socket，但此刻只有 100 个可读：

- 没必要创建 10 万个专门阻塞等待读取的线程。
- 可以由较少线程等待多个 Socket 的就绪事件，然后调用 `read()` / `recv()` 真正读取数据。
- **“就绪”不等于“已经读完数据”**：多路复用负责告知“现在可以尝试 IO”，业务代码仍需要执行读写。

`select`、`poll` 和 `epoll` 都是 IO 多路复用机制；它们**都允许阻塞等待事件**。区别不是“谁会不会等待”，而是**监听集合怎么存、每次调用需要做多少工作、如何获得就绪结果**。

---

## 三、select：位图记录 fd，每次传入集合并扫描

### 1. 底层存什么？

`select` 使用 `fd_set` **位图（Bitmap）**。每个比特位对应一个 fd 编号：1 表示关注，0 表示不关注。位图通常由整数数组实现，所以“位图”和“底层使用数组”不矛盾。

例如只监控 `fd=3、5、7`：

```text
fd 编号： 0 1 2 3 4 5 6 7
位图值： 0 0 0 1 0 1 0 1
               ↑   ↑   ↑
```

### 2. 工作过程

1. 应用程序建立读/写/异常监听位图。
2. 每次调用 `select()` 都将监听位图传给内核。
3. 内核检查关注的 fd；没有就绪事件时可以阻塞等待。
4. 返回时位图被修改，用来表示就绪的 fd。
5. 应用程序扫描位图，找出就绪 fd 并进行读写；下一轮通常要**重新构造监听位图**。

### 3. 性能与限制

- 用户态需要扫描就绪结果；内核处理监听集合的成本也会随监控范围增长，典型用 **O(n)** 概括。
- 常见 Linux 用户态 `fd_set` 的 `FD_SETSIZE` 默认是 **1024**，意味着只能安全表示编号 `0~1023` 的 fd（这主要是**fd 编号范围**限制，不是“只要监听对象不足 1024 个就行”）。
- `select(nfds, ...)` 的 `nfds` 参数通常设为**最大待监控 fd + 1**，并不代表 Socket 总数。

---

## 四、poll：结构体数组直接保存 fd，但仍然全量扫描

### 1. 底层存什么？

`poll` 的用户接口接收 `struct pollfd` **数组**，每个数组元素保存：

- `fd`：要监控的文件描述符；
- `events`：希望关注的事件，例如 `POLLIN`；
- `revents`：`poll()` 返回时实际发生的事件。

```c
struct pollfd fds[3] = {
    {.fd = 3, .events = POLLIN},
    {.fd = 5, .events = POLLIN},
    {.fd = 7, .events = POLLIN}
};

int ready_count = poll(fds, 3, -1);
```

### 2. 为什么 poll 不需要为最大 fd 预留位置？

假设只监控两个 fd：`3` 和 `10000`。

```text
select 位图：下标对应 fd，要能够表示到第 10000 位
  0 1 2 3 4 ......... 10000
        1                 1

poll 数组：数组下标与 fd 编号无关
  fds[0].fd = 3
  fds[1].fd = 10000
```

**这两个数组元素里保存的是 fd 的编号，不是“只有两个连接才会就绪”。**

因此：

- `select`：表示监听集合所需的位图范围，受**最大 fd 编号**影响。
- `poll`：数组元素数等于本次传入的**监控 fd 数量**，不要求 fd 编号连续。

> 例：100 个 Socket 的 fd 为 `1000~1099`。`poll` 需要 100 个数组元素；概念上 `select` 需要至少 1100 个位置才能表示到 fd=1099，但常见默认 `fd_set` 只有 1024 位，不能直接这样用。

### 3. poll 的不足

- 每次调用依然要把整个 `pollfd` 数组交给内核。
- 内核和用户程序仍需检查数组里的监听项，典型复杂度 **O(n)**。
- `poll` **没有 select 那种默认 1024 位图大小的限制**，但仍受进程 fd 限额、内存等系统资源限制。
- 与 `select` 不同，`poll` 的 `events` 和 `revents` 分离，返回结果不会覆盖用户原来的关注事件设置。

**结论：select 与 poll 在核心检测机制上没有本质区别。poll 主要改进数据表示和默认容量限制，而不是解决全量扫描问题。**

> 来源表述辨析：所上传资料将 poll 描述为“动态数组，以链表形式组织”。对于 Linux `poll()` 用户接口，更准确的说法是**传入 `pollfd` 结构体数组**；“链表”不是该接口管理用户监听集合的关键特征。

---

## 五、epoll：一次注册监听项，内核维护就绪列表

### 1. epoll 并没有少保存 Socket

假设监控 10 万个已连接 Socket，100 个可读：

```text
                      内核中的 epoll 实例
            ┌────────────────────────────────┐
            │ 红黑树：注册的全部监听项         │
            │ fd=4, 5, 6, ...（共 10 万个）     │
            │                                │
            │ 就绪链表：已经就绪的监听项       │
            │ fd=5 → fd=91 → ...（约 100 个） │
            └────────────────────────────────┘
                         │
                  epoll_wait()
                         │
                  返回就绪事件
```

- **红黑树**：管理已经注册、需要关注的 fd 对应的 epoll 监听项（不是直接把 Socket 数据存进树里）。
- **就绪链表**：维护就绪监听项及其事件信息的引用（不是把 TCP 数据包或 Socket 本体复制到链表里）。
- 两者都存在，但**职责不同**：红黑树负责“监控谁”，就绪链表负责“谁已就绪”。

### 2. 三个主要 API

```c
int epfd = epoll_create1(0);  // 创建 epoll 实例

struct epoll_event ev = {0};
ev.events = EPOLLIN;           // 关注可读事件
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);  // 注册 fd

struct epoll_event events[128];
int n = epoll_wait(epfd, events, 128, -1); // 阻塞等待就绪事件
```

- `epoll_create1()`：创建 epoll 实例。
- `epoll_ctl()`：添加 `ADD`、修改 `MOD`、删除 `DEL` 监听项；变更注册项通常涉及红黑树等内核结构。
- `epoll_wait()`：等待并取回就绪事件，**不必每次重新传入所有 fd**。

### 3. 内核怎么知道某个 Socket 就绪？

以下是**简化流程**（不必记成“每来一个数据包必调用一次回调”）：

1. 应用通过 `epoll_ctl()` 注册 Socket 和关注事件。
2. Linux 将 epoll 的通知机制接入相关 Socket 的等待队列/唤醒路径。
3. 当 Socket 的收发缓冲区等状态变化，使其满足可读、可写等就绪条件时，内核通过唤醒/回调机制更新 epoll 的就绪信息。
4. 等待中的线程被唤醒，`epoll_wait()` 返回一定数量的就绪事件。
5. 应用根据返回的 fd/event 信息调用 `recv()`、`send()` 等处理 IO。

**不是“内核把请求自动分配给一个对应线程”**；epoll 通知的是等待它的应用线程，具体用单线程还是线程池处理由应用程序决定。

### 4. 为什么大量连接、少量活跃时效率更高？

`select/poll`：每次等待调用都处理完整监听集合，典型扫描成本与 **n（被检查的 fd 数量）**相关。

`epoll`：注册集合长期保存在内核，等待调用主要处理**已就绪的事件**；获取结果的开销通常更接近返回事件数 **k**，不再每次扫描全部 n 个监听项。

当 `n=100000、k=100`，且监控关系相对稳定时，这个优势尤为明显。

**复杂度不能死记成“epoll 全部 O(1)”**：

- 红黑树中注册、查找、删除的典型复杂度约 `O(log n)`。
- `epoll_wait()` 需要把就绪事件交给应用，至少要处理返回的 k 个事件，通常可概括为 `O(k)` 量级（具体内核实现、事件模式、唤醒和并发情况还有其他成本）。
- `epoll` 并不保证任何场景都比 `poll` 快：fd 很少、全都活跃、注册频繁变化时，优势可能缩小。

---

## 六、水平触发 LT 和边缘触发 ET（epoll 常见追问）

假设 fd=5 的接收缓冲区来了 100 字节，应用只读走 30 字节，还剩 70 字节。

| 模式 | 发生什么 | 编程要求 |
|---|---|---|
| LT（Level Triggered，水平触发，默认） | 缓冲区仍可读；后续 `epoll_wait()` 还可能继续报告该 fd 可读 | 比较容易使用 |
| ET（Edge Triggered，边缘触发） | 关注新的就绪变化通知；不能指望每次没读干净都会再次提醒 | 一般配合**非阻塞 fd**，持续读取直到返回 `EAGAIN` / `EWOULDBLOCK` |

```text
LT：还有未读数据 → 后续等待仍可能报告可读
ET：收到就绪通知 → 尽量一次读空，避免留下数据却没有新的通知
```

ET 不是“绝对只通知一次”，而是**不能依赖未读数据自动带来下一次通知**。对于非阻塞 `accept()`，ET 模式下也通常循环接收连接直到 `EAGAIN`。

---

## 七、三者对比表（重点背）

| 对比项 | select | poll | epoll |
|---|---|---|---|
| 监听集合 | `fd_set` 位图 | `pollfd[]` 结构体数组 | 内核红黑树管理注册项 |
| fd 编号与存储位置 | 一位对应一个 fd 编号 | 元素直接存 fd 编号 | 独立注册项引用 fd |
| 数量限制 | 常见 `FD_SETSIZE=1024`（fd 编号范围） | 无固定的 1024 限制 | 无固定的 1024 限制 |
| 每次传入全量集合 | 是 | 是 | 否，首次/变更用 `epoll_ctl()` |
| 查找就绪结果 | 扫描返回位图 | 扫描 `revents` | 读取内核维护的就绪事件 |
| 等待/检查成本 | 常按 O(n) 描述 | 常按 O(n) 描述 | 与返回就绪数 k 更相关；注册操作另计 |
| 典型适用场景 | fd 少，兼容性要求高 | fd 少到中等、接口简单 | 连接数大、活跃比例低、监听关系较稳定 |

**一句话：select 用位图、poll 用结构体数组，二者每次仍扫描；epoll 用红黑树管理全部监听项、就绪链表管理已就绪项。**

---

## 八、Java NIO、Redis、Nginx 与 epoll 的关系

### 1. Java NIO Selector

Java NIO 的 `Selector` 是跨平台的**IO 多路复用抽象**。在常见 Linux JDK 实现中，通常由 epoll 支撑；应用程序主要接触的是 `Selector`、`SelectableChannel`、`SelectionKey`，而不是直接写 `epoll_ctl()`。

```java
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.bind(new InetSocketAddress(8080));

Selector selector = Selector.open();
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // 等待 IO 就绪事件
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove(); // 避免下轮重复处理这个 selected key
        if (key.isAcceptable()) {
            SocketChannel client = server.accept();
            if (client != null) {
                client.configureBlocking(false);
                client.register(selector, SelectionKey.OP_READ);
            }
        } else if (key.isReadable()) {
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(1024);
            int n = client.read(buf);
            if (n == -1) client.close();
            else if (n > 0) {
                buf.flip();
                System.out.println(StandardCharsets.UTF_8.decode(buf));
            }
        }
    }
}
```

上面是**核心流程片段**（需要相应的 `java.nio.*`、`java.net.*`、`java.util.*` 导入和异常处理），用于理解：**注册 → 等待 → 遍历就绪 key → accept/read**。生产环境还要处理半包、粘包、写就绪、背压、取消 key、异常关闭等。

### 2. Redis

Redis 传统的命令执行主路径主要依靠单线程事件循环，并结合 IO 多路复用处理多个客户端连接；这也是它避免为每个连接分配一个命令执行线程的原因之一。不同版本及配置可以使用辅助 IO 线程，但这**不等于每个 Socket 一个线程**。

### 3. Nginx

Nginx 采用事件驱动模型；在 Linux 上可以使用 epoll。一个 worker 进程通常通过事件循环处理很多连接；Nginx 并不是“每个 TCP 连接创建一个线程”。

---

## 九、容易答错的点（本轮讨论重点）

1. **错误：Socket 就是线程。**  正确：Socket 是通信端点，线程是执行代码的单位。
2. **错误：线程和 Socket 都是 Linux 文件。**  正确：Socket 可通过 fd 使用文件接口；线程不是文件。
3. **错误：fd=10000 就说明有 10000 个 Socket。**  正确：fd 是进程内编号，最大 fd 编号不等于 Socket 数量。
4. **错误：poll 只保存就绪 Socket。**  正确：poll 数组保存本次需要监控的**全部 fd**，返回时通过 `revents` 标记就绪项。
5. **错误：select 和 poll 的核心算法完全不同。**  正确：二者都是把监听集合交给内核并遍历，差异主要是位图 vs 结构体数组、集合容量与接口细节。
6. **错误：epoll 只保存活跃 Socket。**  正确：红黑树管理**全部已注册监听项**，就绪链表单独记录**已就绪项**。
7. **错误：epoll 给每个就绪 Socket 分配一个线程。**  正确：`epoll_wait()` 返回就绪事件，后续怎样调度线程是应用自己的决定。
8. **错误：select/poll 会不停地忙轮询。**  正确：调用本身可以阻塞等待；它们的主要成本在于每轮传入并检查整个监听集合。
9. **错误：epoll 所有操作都是 O(1)。**  正确：注册和删除要维护内核数据结构，等待时仍要处理返回的就绪事件。
10. **错误：epoll_wait() 返回代表 TCP 数据已读进应用缓冲区。**  正确：它只给出就绪事件，应用还需要读写。

---

## 十、面试官问：select、poll、epoll 区别？（60 秒版本）

> 三者都是 Linux IO 多路复用机制，一个线程可以等待多个 Socket 的 IO 事件。`select` 使用位图保存关注的文件描述符，存在常见的 `FD_SETSIZE=1024` 默认限制；`poll` 改用 `pollfd` 结构体数组，没有固定的 1024 位图容量限制。但二者每次都要传入监听集合，并扫描集合检查哪些 fd 就绪，成本通常与监控规模成正比。
>
> `epoll` 则通过 `epoll_ctl` 把 fd 注册到内核，由红黑树管理监听项，并通过等待队列通知机制维护就绪链表。调用 `epoll_wait` 时只需获得已经就绪的事件，不用每次全量传入和扫描监听集合。因此在大量长连接、少量活跃连接的场景下，`epoll` 通常更高效。但它并不是所有操作都为 O(1)，并且需要结合 LT/ET 和非阻塞 IO 正确使用。

---

## 十一、面试追问题（自测）

1. 一个进程中有 3 个 Socket，fd 分别是 3、8、10000。为什么不能说有 10001 个 Socket？
2. `select` 的位图为什么与最大 fd 编号有关，而 `poll` 数组只需要 3 个元素？
3. `poll` 数组里到底存全部监听 fd 还是只存就绪 fd？返回结果保存在什么字段？
4. `epoll` 有红黑树，为什么还需要就绪链表？
5. 为什么 `epoll_wait()` 返回后，不能认为应用层已经读到了 TCP 数据？
6. `epoll` 中 LT 和 ET 的区别是什么？为什么 ET 常和非阻塞 IO 一起使用？
7. 10 万连接只有 100 个活跃时，哪种机制更合适？为什么不能简单宣称 epoll 一定是 O(1)？

### 参考要点

1. fd 是进程内文件描述符编号，编号大小不等于 Socket 数量。
2. 位图用**位置表示 fd**；poll 用**数组元素的值表示 fd**。
3. 保存全部被监控 fd；在每个元素的 `revents` 字段记录实际就绪事件。
4. 红黑树回答“监控谁”；就绪链表回答“谁已就绪”。
5. epoll 只报告 IO 就绪，业务程序还需要 `read/recv`。
6. LT 在条件仍成立时会继续报告；ET 不能依赖剩余未读数据反复触发，通常要循环读到 `EAGAIN`。
7. 通常选 epoll；但注册、事件维护、返回事件都有成本，不能将所有操作说成 O(1)。

---

## 十二、最终速记

```text
TCP 连接 → 双端通信关系
Socket   → 程序的网络通信端点
fd       → 进程访问 Socket 等资源的编号
线程     → 执行代码的单位（不是 Socket / fd）

select：位图          → 每轮传集合 + 扫描
poll：  pollfd 数组   → 每轮传集合 + 扫描
 epoll：内核红黑树     → 保存全部注册监听项
        内核就绪链表   → 保存就绪事件
        epoll_wait    → 获取就绪事件，再由应用读写
```

**记忆口诀：select 看位图，poll 看数组，epoll 看就绪；三者都存监听项，只有 epoll 不需要每轮全量扫描。**

### 资料说明

- **来自上传资料的主干**：《大厂后端面试真题-小林coding》，第 9—11 页附近「select、poll、epoll 区别」「epoll 具体工作流程」；同章还讨论了 Redis 使用 IO 多路复用。
- **额外补充/澄清**：fd 与 Socket/线程的关系；`FD_SETSIZE` 是 fd 编号范围限制；`pollfd[]` 接口的精确定义；LT/ET、Java NIO、Nginx、复杂度不能一概视为 O(1) 等。源文“性能损耗呈指数级增长”不应当作精确的复杂度结论：就绪集合线性扫描通常描述为 O(n)。
