# Redis 单线程模型与网络 I/O 多路复用

## 1. Redis 为什么需要网络 I/O？

Redis 的核心数据存储确实是在内存中。

例如：

```
Redis内存

HashMap

user:100  --->  Tom
user:200  --->  Jack
```

执行：

```
GET user:100
```

真正查数据：

```
Redis线程
    |
    ↓
内存Hash表
    |
    ↓
Tom
```

这个过程是内存访问，非常快。

但是客户端请求 Redis 的过程不是直接调用 Redis 内部函数，而是通过网络：

```
客户端
  |
  | TCP网络
  ↓
Redis进程
  |
  ↓
执行命令
  |
  ↓
返回结果
```

所以 Redis 存在两个阶段：

```
网络IO:
    请求进入Redis
    响应离开Redis


内存访问:
    查询Redis数据结构
```

------

# 2. 网络 I/O 到底是什么？

IO = Input / Output。

网络 IO 指：

## 输入

客户端发送请求：

```
GET user:100
```

进入 Redis：

```
客户端
 ↓
网卡
 ↓
Linux TCP协议栈
 ↓
socket接收缓冲区
 ↓
Redis read()
 ↓
Redis进程
```

------

## 输出

Redis返回：

```
Tom
```

流程：

```
Redis
 ↓
write(socket)
 ↓
socket发送缓冲区
 ↓
TCP协议栈
 ↓
网卡
 ↓
客户端
```

所以：

网络 IO 操作的是：

- 命令
- 参数
- 返回结果

不是 Redis 存储的数据。

------

# 3. socket是什么？

Linux中：

> socket 是网络连接的抽象。

每个 socket 有一个编号：

## fd（file descriptor）

文件描述符。

Linux把很多资源都抽象成文件：

- 普通文件
- socket
- 管道

例如：

```
Redis进程

fd表:

0  标准输入
1  标准输出
2  标准错误

10 客户端A socket
11 客户端B socket
12 客户端C socket
```

所以：

```
fd=10
```

代表：

> Redis进程中的某一个客户端TCP连接。

------

# 4. 一个请求是不是一个socket？

不是。

关系：

```
一个socket
    |
    +-- 请求1
    |
    +-- 请求2
    |
    +-- 请求3
```

例如：

客户端连接 Redis：

```
客户端A
    |
    TCP连接
    |
socket fd=10
```

然后发送：

```
SET name Tom

GET name

INCR count
```

这些请求都通过：

```
fd=10
```

发送。

------

# 5. Redis为什么需要epoll？

假设 Redis 有10万个客户端：

```
fd1
fd2
fd3
...
fd100000
```

Redis需要知道：

- 哪个客户端发送请求？
- 哪个socket可以发送数据？

## 传统方式：轮询

不停检查：

```
for(fd in all_socket){

    check(fd)

}
```

问题：

大量连接没有请求。

CPU浪费。

------

# 6. epoll是什么？

epoll 是 Linux 提供的：

> I/O 多路复用机制。

作用：

让一个线程同时监听大量 socket。

核心思想：

不要让 Redis 主动检查所有连接。

而是：

让 Linux 内核帮 Redis 监听。

------

# 7. epoll工作流程

## 7.1 注册socket

Redis启动：

创建监听：

```
6379端口
```

得到：

```
listen socket fd=6
```

注册：

```
epoll_ctl()
```

告诉Linux：

```
关注:

fd=6
```

客户端连接后：

```
fd=10
fd=11
fd=12
```

继续加入epoll。

------

## 7.2 Redis等待事件

Redis线程：

```
epoll_wait()
```

进入等待。

此时：

```
Redis线程
      |
      |
      睡眠
      |
等待socket事件
```

------

## 7.3 数据到达

客户端：

```
GET user:100
```

数据：

```
客户端
 ↓
网卡
 ↓
TCP协议栈
 ↓
socket接收缓冲区
```

Linux发现：

```
fd=10

状态:

不可读
 ↓
可读
```

然后：

把：

```
fd=10
```

加入：

```
epoll就绪队列
```

------

## 7.4 Redis被唤醒

Redis：

```
epoll_wait()
```

返回：

```
fd=10 readable
```

注意：

返回的是：

```
socket编号
```

不是数据。

------

Redis：

```
read(fd=10)
```

才真正读取：

```
GET user:100
```

然后：

解析命令。

------

# 8. epoll就绪队列是什么？

不是：

```
请求队列
```

而是：

```
socket事件队列
```

里面保存：

```
哪些fd已经准备好
```

例如：

```
epoll ready list:

fd=10  可读
fd=20  可写
fd=30  可读
```

------

# 9. Redis 6之前的单线程模型

很多人误解：

> Redis单线程是不是只执行命令？

不是。

Redis主线程负责全部：

```
Redis主线程

        |
        ↓

epoll_wait()

        |
        ↓

发现socket事件

        |
        ↓

read请求

        |
        ↓

解析命令

        |
        ↓

执行Redis命令

        |
        ↓

write响应

        |
        ↓

继续epoll_wait()
```

------

所以：

Redis单线程不是：

```
一直执行GET
```

而是：

```
事件循环
```

------

# 10. 写响应过程

Redis执行：

```
GET user:100
```

得到：

```
Tom
```

尝试：

```
write(fd,"Tom")
```

------

## 情况1：socket可写

直接：

```
Redis
 |
write()
 |
socket发送缓冲区
 |
客户端
```

------

## 情况2：socket发送缓冲区满

例如：

客户端网络慢。

write：

```
EAGAIN
```

表示：

当前不能写。

Redis保存：

```
Redis客户端输出缓冲区

client.reply_buffer

Tom
xxx
xxx
```

等待：

```
fd可写事件
```

------

之后：

epoll通知：

```
fd=10 writable
```

Redis继续：

```
write(fd)
```

------

# 11. 两层缓冲区

## Redis自己的缓冲区

用户空间：

```
Redis进程内存

client reply buffer
```

保存：

还没有发送的数据。

------

## Linux socket缓冲区

内核空间：

```
socket send buffer
```

保存：

已经write，但是还没发送到网络的数据。

------

整体：

```
Redis buffer

    |
    | write()

    ↓

Linux socket buffer

    |
    ↓

TCP

    |
    ↓

客户端
```

------

# 12. Redis 6为什么增加IO多线程？

Redis瓶颈变化：

以前：

```
网络慢
```

后来：

```
网卡越来越快

Redis命令执行越来越快

瓶颈变成:

socket读写
```

------

## Redis 6之前

一个线程：

```
读取请求
执行命令
发送响应
```

全部串行。

------

## Redis 6之后

拆分：

```
             主线程

          执行Redis命令


              |
              |


       IO线程池

       read socket
       write socket
```

注意：

多线程只负责：

- 网络读
- 网络写

不是执行Redis命令。

------

# 13. I/O多路复用 vs 多线程IO

区别：

| 技术         | 解决问题                 |
| ------------ | ------------------------ |
| epoll        | 如何知道哪些socket有事件 |
| Redis IO线程 | 如何更快处理socket读写   |

关系：

```
Redis 6以前:

epoll
 +
单线程处理IO


Redis 6以后:

epoll
 +
多线程处理IO
```

------

# 14. 整体流程总结

完整Redis请求：

```
客户端

  |
  | TCP

  ↓

socket接收缓冲区

  |
  | socket变为可读

  ↓

epoll就绪队列

  |
  | epoll_wait返回fd

  ↓

Redis线程

  |
  | read(fd)

  ↓

解析命令

  |
  ↓

Redis内存数据结构

  |
  ↓

生成结果

  |
  ↓

write(fd)

  |
  ↓

socket发送缓冲区

  |
  ↓

客户端
```

------

# 核心记忆

一句话：

> Redis快在数据存储于内存，但客户端访问Redis仍需要经过网络。epoll负责告诉Redis哪些连接有数据，Redis再通过fd读取请求、访问内存、生成结果并发送响应。Redis 6优化的是网络读写，不是内存查询。

关键概念：

```
socket:
    一个TCP连接


fd:
    socket在Linux中的编号


epoll:
    Linux提供的事件通知机制


ready队列:
    已经准备好的socket列表，不是请求数据


网络IO:
    命令参数和响应结果的搬运


Redis内存:
    真正存储key-value数据
```