# 线程、锁、CompletableFuture、Netty、Reactor、协程完整深度笔记（新增版）

------

# 一、线程、任务、上下文的真正关系

------

# 1. 什么是线程（Thread）

线程本质：

```text
CPU执行流
```

CPU真正调度的是：

- Linux thread
- Windows thread

CPU：

```text
不认识Future
不认识协程
不认识Task
```

CPU只执行：

```text
线程上下文
```

------

# 2. 什么是任务（Task）

任务本质：

```text
待执行的业务逻辑
```

例如：

- HTTP请求
- Runnable
- Future
- Callback
- Coroutine

任务：

```text
属于用户态/runtime概念
```

不是OS概念。

------

# 3. 什么是上下文（Context）

上下文本质：

```text
恢复执行所需的一切状态
```

包括：

- 当前执行位置
- 局部变量
- continuation
- 返回地址
- stack frame

------

# 二、传统同步模型

------

# 1. 传统同步调用

例如：

```java
void handle() {
    queryDB();
    calc();
    send();
}
```

线程栈：

```text
handle
 └── queryDB
```

CPU：

依赖：

```text
call/ret
```

维护调用链。

------

# 2. 为什么传统任务绑定线程

因为：

```text
执行状态在线程栈
```

所以：

```text
任务执行期间
通常必须由同一线程跑完
```

否则：

- 局部变量丢失
- 返回地址丢失
- 调用栈断裂

------

# 三、锁、自旋锁、CAS

------

# 1. CAS（无锁）

CAS：

```text
Compare And Swap
```

流程：

```text
读取旧值
CAS尝试修改
失败重试
```

特点：

```text
多个线程允许同时尝试修改
```

没有：

```text
锁owner
```

------

# 2. 自旋锁

本质：

```text
CAS + lock状态
```

例如：

```java
while(!CAS(lock,0,1)) {
}
```

含义：

```text
0 = 空闲
1 = 已占用
```

------

# 3. 自旋锁的本质

自旋锁是：

```text
互斥锁
```

即：

```text
同一时刻只有一个线程能进入临界区
```

------

# 四、如果抢到自旋锁的线程没执行完怎么办

------

# 1. 其他线程不能继续执行

例如：

线程A：

```text
获得锁
```

但：

```text
还没执行结束
```

线程B：

```text
只能while自旋等待
```

不能：

```text
接着执行A的任务
```

------

# 2. 为什么不能

因为：

```text
锁保护的是连续同步调用栈
```

例如：

```java
lock();
step1();
step2();
unlock();
```

step1执行到一半：

```text
线程上下文在线程栈
```

不能：

```text
换另一个线程继续step2
```

------

# 3. 所以传统锁本质

传统锁：

```text
天然绑定线程执行流
```

------

# 五、锁属于线程吗

------

# 1. 锁本身不属于线程

锁本质：

```text
共享同步对象
```

例如：

```java
Object lock = new Object();
```

多个线程：

```text
都能竞争同一把锁
```

------

# 2. 但锁的“当前使用权”属于线程

例如：

```java
lock.lock();
```

成功后：

锁内部：

```text
owner = 当前线程
```

因此：

```text
锁当前持有权通常属于某线程
```

------

# 3. 为什么必须记录owner

因为需要支持：

- 可重入
- unlock校验
- 唤醒等待线程

例如：

```java
lock.lock();
foo();

void foo(){
    lock.lock();
}
```

锁必须知道：

```text
当前还是同一个线程
```

------

# 六、自旋锁什么时候释放

------

# 1. 主动释放

例如：

```java
while(!CAS(lock,0,1)){}

count++;

lock = 0;
```

最后：

```text
lock = 0
```

就是：

```text
释放锁
```

------

# 2. 不会自动释放

锁不会：

- 时间到自动释放
- CPU切换自动释放

必须：

```text
owner线程主动unlock
```

------

# 3. 如果线程不释放

例如：

```java
while(!CAS(lock,0,1)){}

while(true){}
```

则：

```text
其他线程永远自旋
```

CPU直接爆炸。

------

# 七、CompletableFuture 本质

------

# 1. 不是暂停线程恢复线程栈

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

# 3. 每阶段结束后调用栈直接销毁

例如：

线程A：

```text
执行queryDB
```

执行完：

```text
线程栈结束
```

不是：

```text
暂停
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

因此：

```text
不是恢复旧线程
```

而是：

```text
新线程执行新任务
```

------

# 八、Netty/Reactor/EventLoop 本质

------

# 1. BIO的问题

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
内核统一监听所有socket
```

------

# 3. EventLoop线程本质

EventLoop：

实际上：

```java
while(true){

    ready = epoll_wait();

    process(ready);
}
```

------

# 4. 真正监听socket的是Linux内核

不是Netty。

而是：

```text
Linux epoll
```

Netty只是：

```text
调用epoll API
```

------

# 5. 数据到达流程

------

## 网卡收到数据

↓

## DMA写入内存

↓

## Linux TCP协议栈处理

↓

## epoll就绪队列加入fd

↓

## epoll_wait返回

↓

## EventLoop线程处理handler

------

# 6. Netty不是恢复线程上下文

Netty：

本质：

```text
事件驱动任务执行
```

不是：

```text
恢复原线程调用栈
```

------

# 九、Java如何调用epoll

------

# 1. Java不能直接执行epoll_wait

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

# 4. Netty native epoll

Netty

支持：

```java
EpollEventLoopGroup
```

直接：

```text
JNI调用epoll
```

性能更高。

------

# 十、为什么引入jar包就能实现这些

因为：

```text
并没有修改OS线程
```

而是：

```text
在用户态实现了新的任务调度模型
```

------

# 十一、CompletableFuture/Netty到底做了什么

------

# 1. CompletableFuture

本质：

```text
把连续调用栈
拆成多个阶段任务
```

然后：

```text
通过回调链重新组织执行流程
```

------

# 2. Netty

本质：

```text
事件来了
执行一小段handler
结束
```

不是：

```text
一个线程长期处理整个连接
```

------

# 3. 两者共同点

共同点：

```text
都不是恢复线程调用栈
```

而是：

```text
任务拆分 + 重新调度
```

------

# 十二、真正能恢复执行流的是谁

------

# 1. 协程/Goroutine/Fiber

这些：

真的会：

```text
保存执行上下文
```

包括：

- stack
- locals
- continuation
- PC

------

# 2. Go runtime

Go

实现：

```text
G-M-P调度模型
```

goroutine：

```text
可暂停恢复
```

------

# 3. Java Loom

Project Loom

virtual thread：

```text
真正支持Continuation
```

------

# 十三、现代并发模型真正变化

------

# 1. 传统模型

```text
线程 = 业务流程
```

线程：

长期绑定任务。

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

等待时：

```text
释放线程
```

------

# 十四、最终核心总结

------

# 1. 自旋锁

本质：

```text
互斥执行连续同步调用栈
```

因此：

```text
线程没执行完
别人不能接着执行
```

------

# 2. CompletableFuture/Netty

本质：

```text
拆分阶段任务
重新调度
```

不是：

```text
恢复线程上下文
```

------

# 3. Reactor/EventLoop

本质：

```text
内核负责等待IO
线程只处理活跃事件
```

------

# 4. 协程/Goroutine

本质：

```text
真正保存执行流
实现暂停恢复
```

------

# 5. 现代并发终极目标

不是：

```text
创建更多线程
```

而是：

```text
减少线程等待
减少共享竞争
提高CPU利用率
```