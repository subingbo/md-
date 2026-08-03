# Java 并发模型、线程上下文、CompletableFuture、Netty、Reactor、协程完整笔记

------

# 一、线程、任务、上下文到底是什么

------

# 1. 线程（Thread）

线程本质：

```text
CPU执行流
```

操作系统真正调度的是：

- Linux thread
- Windows thread

CPU：

```text
不认识任务
不认识Future
不认识协程
```

CPU只执行：

```text
线程上下文
```

------

# 2. 任务（Task）

任务本质：

```text
待执行的业务逻辑
```

例如：

- HTTP请求
- Runnable
- Future
- Callback
- 协程

任务：

```text
不是OS概念
```

而是：

```text
用户态/runtime概念
```

------

# 3. 执行上下文（Context）

上下文本质：

```text
恢复执行所需的一切状态
```

包括：

- 当前执行位置
- 局部变量
- 返回地址
- continuation
- 状态机state

------

# 二、传统线程模型

------

# 1. 传统同步调用

例如：

```java
void handle() {
    String x = queryDB();
    String y = calc(x);
    send(y);
}
```

执行时：

线程栈：

```text
handle
 └── queryDB
```

CPU依靠：

```text
call/ret
```

维护调用链。

------

# 2. 上下文在线程栈

线程栈保存：

- stack frame
- locals
- return address
- PC(program counter)

因此：

```text
线程 = 执行状态载体
```

------

# 3. 为什么传统任务绑定线程

因为：

```text
执行状态在线程栈
```

所以：

```text
任务执行期间通常不能切线程
```

否则：

- 局部变量没了
- 返回地址没了
- 调用栈断了

------

# 三、锁、CAS、自旋锁

------

# 1. CAS（无锁）

CAS：

```text
Compare And Swap
```

流程：

```text
读取旧值
尝试CAS修改
失败重试
```

特点：

```text
允许多个线程同时尝试修改
```

没有：

```text
锁拥有者
```

------

# 2. 自旋锁

本质：

```text
CAS + lock变量
```

例如：

```java
while(!CAS(lock,0,1)) {
}
```

特点：

```text
必须先获得锁资格
```

只有拿锁线程：

```text
才能进入临界区
```

------

# 3. 阻塞锁

例如：

```java
synchronized
```

高竞争下：

```text
抢不到锁的线程sleep/block
```

而不是一直空转。

------

# 4. 三者区别

| 类型   | 是否互斥 | 等待方式    |
| ------ | -------- | ----------- |
| CAS    | 否       | 重试        |
| 自旋锁 | 是       | 忙等        |
| 阻塞锁 | 是       | sleep/block |

------

# 5. 高竞争问题

------

## CAS

问题：

```text
失败重试风暴
```

导致：

- cache line竞争
- 总线同步
- CPU空转

------

## 自旋锁

问题：

```text
大量线程疯狂while循环
```

CPU浪费更严重。

------

## 阻塞锁

问题：

```text
线程切换开销
```

但：

```text
高竞争更稳定
```

------

# 四、线程和任务是否绑定

------

# 1. 传统模型

通常：

```text
一个任务长期绑定一个线程
```

例如：

```java
pool.submit(() -> {
    doWork();
});
```

任务执行期间：

```text
线程一直占着
```

------

# 2. 现代模型

现代并发：

```text
尽量缩短线程绑定时间
```

线程：

```text
只在真正运行代码时参与
```

等待期间：

```text
释放线程
```

------

# 五、CompletableFuture 本质

------

# 1. 不是恢复线程栈

很多人误解：

```text
CompletableFuture暂停线程后恢复
```

实际上：

```text
根本不会恢复旧调用栈
```

------

# 2. 真正做法：拆任务

例如：

```java
CompletableFuture
    .supplyAsync(this::queryDB)
    .thenApply(this::calc)
    .thenAccept(this::send);
```

本质：

```text
任务1：queryDB
任务2：calc
任务3：send
```

------

# 3. 当前阶段结束

例如：

线程A：

```text
执行queryDB
```

执行完：

```text
线程栈直接结束
```

不是：

```text
暂停栈
```

------

# 4. 下一阶段重新提交

Future完成后：

```text
创建新的Runnable
```

例如：

```text
() -> calc(x)
```

线程池：

```text
重新调度执行
```

所以：

```text
不是恢复线程上下文
```

而是：

```text
重新提交新任务
```

------

# 5. CompletableFuture本质

本质：

```text
回调链(callback chain)
```

或者：

```text
Continuation Passing Style（CPS）
```

------

# 六、Netty/Reactor/EventLoop 本质

------

# 1. BIO问题

传统：

```java
socket.read()
```

如果没数据：

```text
线程阻塞
```

于是：

```text
一个连接一个线程
```

------

# 2. epoll核心思想

不要：

```text
线程自己等socket
```

而是：

```text
让内核统一监听所有socket
```

------

# 3. epoll流程

------

## 注册socket

```c
epoll_ctl(...)
```

告诉内核：

```text
socket有事件时通知我
```

------

## EventLoop线程

一直：

```c
epoll_wait(...)
```

等待：

```text
哪些socket ready
```

------

## 数据到达

内核：

```text
把ready socket放进就绪队列
```

------

## epoll_wait返回

线程：

```text
处理对应handler
```

执行完：

```text
继续等待下一个事件
```

------

# 4. Netty本质

不是：

```text
线程长期处理业务
```

而是：

```text
事件来了
执行一小段handler
结束
```

------

# 5. Reactor模型

本质：

```text
等待IO由内核负责
线程只处理活跃事件
```

------

# 七、Java 如何调用 epoll

------

# 1. Java不能直接调用epoll_wait

因为：

```text
epoll_wait是Linux系统调用
```

Java运行在JVM。

------

# 2. JVM通过JNI调用

流程：

```text
Java代码
↓
JDK NIO
↓
JNI
↓
C代码
↓
epoll_wait
↓
Linux内核
```

------

# 3. selector.select()

Linux下：

底层最终：

```c
epoll_wait(...)
```

------

# 4. JDK内部实现

Linux：

```text
EPollSelectorImpl
```

macOS：

```text
kqueue
```

Windows：

```text
IOCP/select
```

------

# 八、为什么Netty不需要修改OS

因为：

```text
OS本来就支持IO事件通知
```

Netty只是：

```text
利用epoll实现EventLoop
```

不是：

```text
修改线程模型
```

------

# 九、真正保存执行栈的是谁

------

# 1. CompletableFuture/Netty

不会：

```text
保存线程栈
```

它们：

```text
结束旧任务
重新提交新任务
```

------

# 2. Coroutine/Goroutine

才会：

```text
真正保存执行流
```

包括：

- stack
- PC
- locals
- continuation

------

# 3. Go runtime

Go

实现：

```text
G-M-P模型
```

goroutine：

```text
可挂起恢复
```

------

# 4. Java Loom

Project Loom

virtual thread：

```text
真正支持continuation
```

------

# 十、现代并发模型的核心变化

------

# 1. 传统模型

```text
线程 = 业务流程
```

线程：

- 长期绑定任务
- 等待期间也占资源

------

# 2. 现代模型

```text
任务 = 业务流程
线程 = 临时CPU执行器
```

线程：

```text
只在真正运行代码时参与
```

------

# 3. 线程只是CPU执行器

意思：

```text
线程不再拥有业务执行状态
```

而是：

```text
runtime/task对象保存状态
线程只是临时执行代码
```

------

# 十一、为什么现代模型并发高

因为：

传统：

```text
等待IO时线程被浪费
```

现代：

```text
等待IO时线程释放
```

因此：

```text
少量线程
=
大量任务
```

------

# 十二、最终最核心结论

------

# 1. CompletableFuture/Netty

本质：

```text
拆分任务阶段
重新调度
```

不是：

```text
恢复旧线程栈
```

------

# 2. Reactor/EventLoop

本质：

```text
内核负责等待IO
线程只处理活跃事件
```

------

# 3. 协程/Goroutine

本质：

```text
真正保存执行流
实现暂停恢复
```

------

# 4. 现代并发的终极目标

不是：

```text
让线程更多
```

而是：

```text
减少线程等待
减少共享竞争
提高线程利用率
```

------

# 十三、整体架构图

```text
                 ┌──────────────┐
                 │   Task任务    │
                 └──────┬───────┘
                        │
          保存状态/回调/Continuation
                        │
          ┌─────────────▼─────────────┐
          │ Runtime / EventLoop / CF  │
          └─────────────┬─────────────┘
                        │ 调度
                Runnable/Handler
                        │
                 ┌──────▼──────┐
                 │   Thread     │
                 └──────┬──────┘
                        │
                 CPU执行机器码
                        │
                 ┌──────▼──────┐
                 │ Linux Kernel │
                 └─────────────┘
```