# Linux top 命令 CPU、内存、Swap 指标详解

## 1. top 命令作用

`top` 用于实时查看 Linux 系统：

- CPU 使用情况
- 内存使用情况
- Swap 使用情况
- 进程资源占用情况
- 系统负载情况

常用于服务器排查：

- CPU 是否过高
- 内存是否不足
- 是否发生 swap 抖动
- 哪个进程占资源

------

# 一、top 整体 CPU 指标

示例：

```text
%Cpu(s):
8.9 us,
33.2 sy,
0.0 ni,
43.4 id,
12.4 wa,
1.8 hi,
0.4 si
```

这些比例加起来约等于：

```text
100%
```

表示整个机器 CPU 的状态分布。

------

## 1. us（user）

用户态 CPU：

```text
us = 8.9%
```

表示：

用户程序消耗 CPU 的比例。

例如：

- Java 应用
- Python 程序
- 数据计算

通常业务代码消耗这里。

------

## 2. sy（system）

内核态 CPU：

```text
sy = 33.2%
```

表示：

Linux 内核消耗 CPU 的比例。

包括：

- 系统调用
- 网络处理
- 文件系统
- 内存管理
- IO 操作

如果 sy 很高：

说明大量时间花在：

- 内核
- IO
- 网络
- 内存管理

例如：

swapoff 过程中：

Linux 需要搬迁内存页，

所以：

```text
sy 会升高
```

------

## 3. id（idle）

CPU 空闲：

```text
id = 43.4%
```

表示：

CPU 当前没有工作。

例如：

8核机器：

总计算能力：

```
800%
```

如果：

```
id = 50%
```

说明：

大约还有一半 CPU 能力空闲。

------

## 4. wa（iowait）

IO 等待：

```text
wa = 12.4%
```

表示：

CPU 在等待 IO。

例如：

- 磁盘读取
- 写文件
- swap 交换

swapoff 时：

```
swap -> RAM
```

需要大量读写，

所以：

```
wa 会升高
```

------

# 二、top CPU 百分比是否包含所有核心？

## top 顶部 CPU

例如：

```text
Cpu(s):
us 10%
sy 20%
id 70%
```

表示：

整个机器平均后的 CPU。

不会超过：

```
100%
```

------

## 进程列表 CPU

例如：

```text
PID       %CPU

java      350%
```

这里可以超过 100%。

原因：

Linux 按 CPU 核心累计计算。

假设：

8核 CPU：

```
CPU0 = 100%
CPU1 = 100%
CPU2 = 100%
...
CPU7 = 100%
```

总能力：

```
800%
```

所以：

```
java 300%
```

表示：

这个 Java 进程占用了约：

```
3个CPU核心
```

不是 CPU 超载。

------

## 查看每个 CPU 核心

top 中按：

```
1
```

显示：

例如：

```
Cpu0 : 80%
Cpu1 : 70%
Cpu2 : 10%
Cpu3 : 0%
```

可以看到哪个核心繁忙。

查看核心数量：

```bash
nproc
```

------

# 三、Load Average 系统负载

示例：

```text
load average:
5.87, 3.66, 5.23
```

分别：

```
1分钟平均
5分钟平均
15分钟平均
```

------

## load 不等于 CPU 使用率

load 表示：

等待运行或者等待 IO 的任务数量。

例如：

4核机器：

```
load = 4
```

表示：

CPU 基本满载。

------

8核机器：

```
load = 4
```

表示：

压力不大。

------

如果：

```
load > CPU核心数
```

说明：

任务排队。

------

查看 CPU 核数：

```bash
nproc
```

------

# 四、Linux 内存指标

示例：

```text
MiB Mem:

15733 total
508 free
14420 used
1824 buff/cache

avail Mem:
2154
```

------

# 1. total

总物理内存：

例如：

```
15733MB
```

表示：

15.7GB RAM。

------

# 2. free

完全空闲内存：

例如：

```
508MB
```

表示：

没有被任何东西使用的内存。

但是：

Linux 不追求 free 大。

------

# 3. used

已经使用：

例如：

```
14420MB
```

包括：

- 应用程序
- JVM
- Docker
- 内核

注意：

Linux 的 used 不代表真正压力。

------

# 4. buff/cache

缓存：

例如：

```
2953MB
```

包括：

## Page Cache

文件缓存：

例如：

读取：

```
xxx.jar
地图数据
配置文件
```

Linux 会缓存。

优点：

再次读取速度更快。

------

## Buffer

块设备缓存。

------

## Slab Cache

内核对象缓存：

例如：

- inode
- dentry

------

# 5. available Mem（最重要）

例如：

```
avail Mem:
2154MB
```

表示：

系统还能安全分配给新程序的内存。

计算：

```
available
≈
free
+
可回收缓存
```

不是：

```
free + buff/cache
```

------

为什么：

例如：

```
buff/cache = 3000MB
```

但是：

```
available = 2000MB
```

原因：

buff/cache 中：

部分可以释放：

```
page cache
```

部分不能完全释放：

```
内核缓存
共享内存
tmpfs
```

所以：

available 小于 buff/cache 很正常。

------

# 五、Swap

示例：

```text
Swap:

8G total
6G used
```

表示：

磁盘上的交换空间。

当 RAM 不够：

Linux 会把：

```
不活跃内存页
```

移动到：

```
swap
```

释放 RAM。

------

# Swap 常见问题

## swap 使用过高

例如：

```
Swap used 6GB
```

说明：

之前内存压力较大。

可能导致：

- Java 响应慢
- IO升高
- 系统卡顿

------

# swapoff 原理

执行：

```bash
swapoff -a
```

作用：

把：

```
swap
```

里面的数据：

搬回：

```
RAM
```

过程：

```
swap
 ↓
内存
```

所以：

会出现：

- available 下降
- sy升高
- wa升高

------

如果内存不足：

swapoff 会慢。

因为：

需要找地方放回 RAM。

------

# 六、排查服务器性能流程

## 1. 看 CPU

```bash
top
```

关注：

```
us
sy
wa
id
```

------

## 2. 看哪个进程占 CPU

```bash
top
```

或者：

```bash
ps aux --sort=-%cpu | head
```

------

## 3. 看内存

```bash
free -h
```

重点：

```
available
```

不是：

```
free
```

------

## 4. 看 swap

```bash
swapon --show
```

------

## 5. 看谁占 swap

```bash
for pid in $(ps -e -o pid=); do
    awk '/VmSwap/{if($2>0) print "'$pid' "$2" KB"}' /proc/$pid/status 2>/dev/null
done | sort -k2 -nr | head
```

------

# 七、实际服务器判断标准

## CPU

正常：

```
id > 20%
```

压力：

```
id < 10%
```

------

## 内存

关注：

```
available
```

不是：

```
used
```

------

## Swap

偶尔使用：

正常。

长期大量使用：

需要排查内存。

------

# 八、核心记忆

一句话：

> Linux 内存不是看 free，而是看 available；CPU 进程占用可以超过100%，因为按核心累计；top 顶部 CPU 是所有核心平均值；swap 使用高说明内存压力，需要关注 IO 和 available。

常用命令：

```bash
top
free -h
nproc
swapon --show
ps aux --sort=-%mem | head
ps aux --sort=-%cpu | head
```