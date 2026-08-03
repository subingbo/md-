

# volatile 原理总结

## 1. volatile 作用

volatile 两个核心作用：

1. **保证可见性**
2. **禁止指令重排序（保证有序性）**

注意：

- volatile **不保证原子性**

例如：

```java
volatile int count;

count++;
```

仍然可能线程不安全。

------

# 2. volatile 可见性

## 问题

线程修改变量时，可能只修改自己的工作内存：

```
线程1:
flag=true

线程1工作内存:
true

主内存:
false
```

线程2读取不到最新值。

## volatile 解决

volatile 写：

```
线程修改变量
      ↓
刷新到主内存
```

volatile 读：

```
从主内存读取最新值
```

保证：

> 一个线程修改 volatile 变量后，其他线程能够看到最新值。

------

# 3. 指令重排序

## 为什么发生？

JVM 和 CPU 为提高性能，会调整没有依赖关系的指令执行顺序。

例如：

```java
a = 10;
b = 20;
```

可能执行：

```java
b = 20;
a = 10;
```

单线程结果不变。

但是多线程可能导致通信逻辑错误。

------

# 4. volatile 与重排序

volatile 通过 **内存屏障** 限制指令重排序。

核心理解：

> volatile 不是禁止所有优化，而是禁止影响线程通信的重排序。

例如：

```java
data = 100;

volatile ready = true;
```

不能变成：

```java
ready = true;

data = 100;
```

否则：

线程2：

```java
if(ready){
    use(data);
}
```

可能看到：

```
ready=true

但是data还没有准备好
```

------

# 5. volatile 经典应用：禁止单例重排序

代码：

```java
private volatile static Singleton instance;
```

对象创建：

```
1. 分配内存
2. 初始化对象
3. instance指向对象
```

正常：

```
分配内存
 ↓
初始化对象
 ↓
instance指向对象
```

如果没有 volatile：

可能：

```
分配内存
 ↓
instance指向对象
 ↓
初始化对象
```

其他线程：

```java
if(instance != null){
    return instance;
}
```

可能拿到未初始化对象。

------

# 6. volatile 内存屏障

JMM 中常见四种屏障：

- StoreStore
- StoreLoad
- LoadLoad
- LoadStore

## 6.1 volatile 写

例如：

```java
data = 100;

ready = true; // volatile写
```

### StoreStore

保证：

```
普通写
 ↓
volatile写
```

不能变成：

```
volatile写
 ↓
普通写
```

作用：

> 保证发布数据前，数据写入已经完成。

例如：

错误：

```
ready=true

data=100
```

线程2看到 ready=true 时，data 可能还没准备好。

------

### StoreLoad

保证：

```
volatile写
 ↓
后续读写
```

不能变成：

```
后续读写
 ↓
volatile写
```

作用：

> 保证 volatile 写是一个同步边界，后续操作不会提前。

例如：

正常：

```
数据准备
 ↓
volatile通知
 ↓
后续操作
```

不能变成：

```
数据准备
 ↓
后续操作
 ↓
volatile通知
```

------

## 6.2 volatile 读

例如：

```java
if(ready){

    use(data);
}
```

### LoadLoad

保证：

```
volatile读
 ↓
普通读
```

不能变成：

```
普通读
 ↓
volatile读
```

作用：

> 读取通知后，再读取数据。

------

### LoadStore

保证：

```
volatile读
 ↓
普通写
```

不能变成：

```
普通写
 ↓
volatile读
```

作用：

> 获取状态后，再进行后续写操作。

------

# 7. volatile 读写方向总结

## volatile 写

主要保证：

```
普通写
 ↓
volatile写
```

不能乱序。

以及：

```
volatile写
 ↓
后续读写
```

不能乱序。

目的：

> 发布数据时，保证状态正确。

------

## volatile 读

主要保证：

```
volatile读
 ↓
后续读写
```

不能乱序。

目的：

> 获取状态后，再使用数据。

------

# 8. 为什么不是所有操作都禁止重排序？

因为 JVM 只需要保证：

> 单线程执行结果不变。

普通重排序不会影响单线程。

只有多线程通信时才可能出现问题。

所以 JMM 只禁止会破坏 happens-before 关系的重排序。

------

# 9. 经典线程通信模型

线程1（生产者）：

```java
data = 100;

ready = true;
```

线程2（消费者）：

```java
if(ready){

    System.out.println(data);
}
```

关系：

```
写数据
 ↓
volatile写
 ↓
volatile读
 ↓
读数据
```

volatile 保证：

> 一个线程通过 volatile 发布的数据，另一个线程读取 volatile 后能够看到。

------

# 10. 面试总结

volatile：

> volatile 通过保证变量可见性，并利用内存屏障限制 volatile 读写相关的指令重排序，保证多线程情况下线程通信正确。但是 volatile 不保证原子性。

一句话理解：

> volatile 是一种轻量级同步机制，通过可见性和有序性保证线程之间的数据同步。

```
这版相比之前：
- 去掉了容易误导的“所有操作不能跨越 volatile”
- 补充了 StoreLoad 的作用
- 保留了面试需要的 JMM 细节
- 长度控制在平时复习可快速扫完的范围内
- 和你之前学习 CAS、LongAdder、ThreadLocal 的笔记风格一致
```