# FissLock / P4 / BMv2 学习笔记（当前阶段总结）

## 1. 整体目标

当前目标是：

- 复现论文《Lock Fission》
- 理解 P4 数据面编程
- 理解 BMv2 软件交换机运行机制
- 最终实现一个基于交换机的数据面锁管理系统（FissLock）

目前已经建立了：

- P4 基础认知
- parser / ingress / egress 流程理解
- metadata（meta）传递机制理解
- BMv2 + simple_switch_cli 基础使用
- FissLock 核心思想理解

------

# 2. P4 到底是什么

P4 不是“交换机软件”。

P4 是：

> 一种“描述数据包如何被交换机处理”的编程语言。

本质上：

- CPU 程序操作“内存”
- P4 程序操作“网络包”

P4 程序定义：

- 如何解析包
- 如何匹配规则
- 如何修改头部
- 如何转发
- 如何维护少量状态

------

# 3. BMv2 是什么

BMv2：

> Behavioral Model v2

它是：

- P4 的软件交换机
- 一个“模拟真实交换机 ASIC”的程序

作用：

- 运行 P4 程序
- 调试数据面逻辑
- 模拟交换机行为

你现在并不在真实交换机上运行。

你是在：

```text
P4 程序
  ↓ 编译
BMv2 软件交换机执行
```

------

# 4. P4 的完整处理流水线

一个包进入交换机后：

```text
网卡
 ↓
Parser
 ↓
Ingress
 ↓
Traffic Manager（队列）
 ↓
Egress
 ↓
Deparser
 ↓
发出
```

------

# 5. Parser 是什么

Parser：

> 把二进制包解析成结构化字段

例如：

```text
Ethernet Header
IP Header
TCP Header
```

Parser 会：

```p4
extract(hdr.ethernet)
extract(hdr.ipv4)
extract(hdr.tcp)
```

于是：

```p4
hdr.ipv4.srcAddr
hdr.tcp.dstPort
```

这些字段后面才能使用。

------

# 6. Header 是什么

Header 是：

> 你定义的数据结构

例如：

```p4
header ipv4_t {
    bit<32> srcAddr;
    bit<32> dstAddr;
}
```

P4 不会自动帮你定义。

你必须：

- 自己定义格式
- 自己解析
- 自己决定处理哪些协议

------

# 7. 为什么必须识别协议

因为：

交换机看到的只是：

```text
010101010101...
```

它根本不知道：

- 哪部分是 MAC
- 哪部分是 IP
- 哪部分是 TCP

必须靠你写 parser。

例如：

```p4
transition select(hdr.ethernet.etherType) {
    0x0800: parse_ipv4;
}
```

这里：

```text
0x0800
```

表示：

> Ethernet 中的 IPv4 类型号

------

# 8. isValid() 是什么

Header 是否存在。

例如：

```p4
if (hdr.tcp.isValid()) {
    ...
}
```

意思：

> 当前包确实解析出了 TCP 头

否则：

- 这个 header 不存在
- 访问会出错

------

# 9. Ingress 是什么

Ingress：

> 包“进入交换机”时的主处理逻辑

这是：

- 最核心的数据面逻辑
- 大多数功能都在这里实现

包括：

- 查表
- 修改 meta
- 修改 header
- 决定转发
- 更新寄存器

------

# 10. Egress 是什么

Egress：

> 包“准备离开交换机”时的处理阶段

通常用于：

- 最后修改
- 队列相关逻辑
- 镜像
- 统计
- 最终封装

------

# 11. Ingress 和 Egress 的区别

Ingress 更像：

```text
“决定怎么处理”
```

Egress 更像：

```text
“真正发出去前再处理”
```

两者：

- 都可以有 table
- 都可以改 meta
- 都可以改 header

但：

看到的数据可能不同。

例如：

- ingress 时还没决定出口端口
- egress 时已经知道输出端口

------

# 12. Metadata（meta）是什么

Metadata：

> 在交换机流水线内部传递的数据

它不是网络包内容。

不会发到网络上。

只是交换机内部使用。

------

# 13. meta 的本质

meta 本质上类似：

```text
全局上下文变量
```

包进入流水线后：

- 创建一份 context
- 一路向后传递
- 最后销毁

里面包括：

- 用户自定义 meta
- 系统 intrinsic meta

------

# 14. meta 是否一直存在

是的。

整个流水线：

```text
parser
→ ingress
→ egress
→ deparser
```

都共享同一个 packet context。

所以实际上：

并不是“传递一部分”。

而是：

> 同一个上下文对象始终存在

------

# 15. ingress / egress 的 meta 可见性

本质上：

```text
所有 meta 都一直存在
```

但：

某些系统字段：

- 在早期阶段尚未生成
- 或尚未允许访问

例如：

```text
egress_spec
egress_port
queue depth
```

不同阶段能看到的不一样。

------

# 16. table 是什么

table：

> 匹配规则表

例如：

```p4
table ipv4_lpm {
    key = {
        hdr.ipv4.dstAddr: lpm;
    }
    actions = {
        ipv4_forward;
    }
}
```

作用：

- 根据 key 查规则
- 命中后执行 action

------

# 17. action 是什么

action：

> 真正执行修改逻辑的代码块

例如：

```p4
action ipv4_forward(macAddr_t dst, egressSpec_t port) {
    standard_metadata.egress_spec = port;
}
```

table 只是“选择”。

真正做事的是 action。

------

# 18. simple_switch_cli 是什么

这是：

> BMv2 的控制面 CLI

作用：

- 给 table 添加规则
- 查看状态
- 修改寄存器

例如：

```bash
table_add ipv4_lpm ipv4_forward 10.0.1.1 => 00:11:22:33:44:55 2
```

意思：

```text
匹配:
10.0.1.1

执行:
ipv4_forward(
    00:11:22:33:44:55,
    2
)
```

后面的参数：

就是传给 action 的。

------

# 19. 为什么会有 Python 环境

因为很多 P4 教程：

- 控制面
- 发包脚本
- 自动化测试

都用 Python 写。

例如：

- Mininet
- P4Runtime
- scapy 发包

但：

```text
simple_switch_cli
```

本身不是 Python。

------

# 20. FissLock 到底是什么

FissLock 核心思想：

> 锁裂解（Lock Fission）

传统分布式锁：

```text
锁状态
锁队列
超时
公平性
客户端信息
```

都放在一个锁服务中。

这会导致：

- 内存大
- CPU 瓶颈
- 网络开销大

------

# 21. FissLock 的核心思想

FissLock：

> 把“锁状态”和“复杂逻辑”拆开

交换机只负责：

```text
锁是否被占用
```

即：

```text
0 / 1 bitmap
```

复杂逻辑：

- 重试
- 排队
- 超时
- 恢复

全部放到客户端。

------

# 22. 为什么叫 Lock Fission

Fission：

```text
裂变 / 裂解
```

意思：

> 将完整锁服务拆散

交换机：

只保留最核心的：

```text
原子锁状态
```

其他：

全部下放。

------

# 23. 为什么适合 P4

因为交换机 ASIC：

非常不适合：

- 大内存
- 复杂链表
- 动态结构
- 阻塞等待

但非常适合：

- 位图
- 哈希
- 原子寄存器
- 超高速匹配

所以：

```text
lock bitmap
```

特别适合放在交换机里。

------

# 24. FissLock 中交换机负责什么

交换机负责：

```text
lock_id → state
```

例如：

```text
0 = free
1 = occupied
```

收到 acquire：

```text
if free:
    set occupied
    return success
else:
    return fail
```

收到 release：

```text
set free
```

仅此而已。

------

# 25. FissLock 中客户端负责什么

客户端负责：

- 重试
- backoff
- 超时
- 重发
- 死锁恢复
- 公平性
- 排队策略

因此：

交换机逻辑极其轻量。

------

# 26. P4 中“状态”怎么存

P4 没有 malloc。

没有对象。

没有动态内存。

状态主要靠：

```p4
register
```

例如：

```p4
register<bit<1>>(1024) lock_table;
```

这本质上：

就是：

```text
1024 位 bitmap
```

------

# 27. 为什么 P4 很像“受限硬件 DSL”

P4 不是通用语言。

没有：

- malloc
- 线程
- while
- 递归
- 动态对象

因为：

它最终要映射到：

```text
交换机 ASIC 流水线
```

必须：

- 固定时延
- 固定资源
- 固定执行路径

------

# 28. V1Switch 是什么

```p4
V1Switch(
    Parser(),
    VerifyChecksum(),
    Ingress(),
    Egress(),
    ComputeChecksum(),
    Deparser()
)
```

这是：

> BMv2 的交换机模型

相当于：

```text
把各阶段函数接起来
```

------

# 29. 能不能自己加流程

不能随便加。

因为：

交换机硬件流水线结构是固定的。

你能做的是：

- 在 ingress 里增加逻辑
- 增加 table
- 增加 action
- 增加 metadata

但不能：

```text
再凭空新增一个硬件阶段
```

------

# 30. 当前你已经理解的核心

目前已经掌握：

## P4 基础

- parser
- header
- metadata
- table
- action
- ingress
- egress

## BMv2

- 软件交换机
- simple_switch_cli
- table_add

## FissLock

- lock bitmap
- lock fission
- 数据面锁状态
- 客户端负责复杂逻辑

------

# 31. 下一阶段应该学习什么

下一步建议：

## 第一阶段

真正手写一个：

```text
acquire/release
```

P4 锁交换机。

包括：

- 自定义 lock packet
- parser
- register bitmap
- acquire/release action

------

## 第二阶段

加入：

- 多 client
- 重试
- 返回 success/fail

------

## 第三阶段

开始接近 FissLock：

- client 逻辑
- backoff
- timeout
- lock partition
- hash mapping

------

# 32. 当前最重要的理解

你目前已经开始真正接触：

> “交换机不是在转发包，而是在执行受限计算”

这是 P4 最核心的思想。

FissLock 本质上：

就是：

```text
把锁服务压缩成交换机能做的最小原子操作
```

然后：

利用交换机的线速能力实现超高吞吐。