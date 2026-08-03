# 📘 CompletableFuture 进阶笔记（allOf / 执行时机 / 非阻塞模型）

------

# 一、前置认知（简单回顾）

在前面你已经掌握：

```text
complete = 发布结果
thenXXX = 注册回调
```

以及：

```text
CompletableFuture 是“事件驱动模型”，不是顺序执行模型
```

------

# 二、allOf 的本质

## ✅ 定义

```java
CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf3, cf4, cf5);
```

------

## 👉 本质

```text
多个 future → 一个“全部完成信号”
```

------

## 👉 行为

```text
cf3 完成
cf4 完成
cf5 完成
↓
cf6 才完成
```

------

## ⚠️ 注意

```text
allOf 不包含结果，只表示“完成状态”
```

------

# 三、thenApply + allOf 执行时机（核心）

------

## 示例

```java
CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf3, cf4, cf5);

CompletableFuture<String> result = cf6.thenApply(v -> {
    result3 = cf3.join();
    result4 = cf4.join();
    result5 = cf5.join();
    return "result";
});
```

------

## ❓关键问题

> thenApply 会不会提前执行？

------

## ❌ 不会！

```text
thenApply 里的代码绝不会在 cf6 完成之前执行
```

------

## ✔ 正确执行流程

```text
1️⃣ 创建 cf3、cf4、cf5

2️⃣ 创建 cf6 = allOf(...)

3️⃣ 调用 thenApply（注册回调）

4️⃣ 当前线程继续执行后面的代码（不阻塞）

5️⃣ 某个时间点：
   cf3、cf4、cf5 全部完成

6️⃣ cf6 完成

7️⃣ thenApply 被触发执行
```

------

# 四、thenApply 的本质（非常重要）

```text
thenApply = 注册“完成后的动作”
```

------

👉 不是：

```text
立即执行函数
```

------

👉 而是：

```text
等前一个 future 完成 → 再执行
```

------

# 五、非阻塞模型（核心理解）

------

## 示例

```java
System.out.println("1");

cf6.thenApply(v -> {
    System.out.println("3");
    return "ok";
});

System.out.println("2");
```

------

## 输出

```text
1
2
（稍后）
3
```

------

## 👉 说明

```text
thenApply 不会阻塞线程
```

------

## ✔ 对比阻塞写法

```java
cf6.get();   // 阻塞
System.out.println("2");
```

------

## 输出

```text
（等待）
2
```

------

# 六、join 在 allOf 中为什么不会阻塞？

------

## 代码

```java
cf6.thenApply(v -> {
    cf3.join();
    cf4.join();
    cf5.join();
});
```

------

## 原因

```text
thenApply 执行时：
cf3、cf4、cf5 已经全部完成
```

------

👉 所以：

```text
join = 立即返回（不会阻塞）
```

------

## ❌ 错误写法（会阻塞）

```java
cf3.join();
cf4.join();
cf5.join();
```

👉 因为没有等它们完成

------

# 七、allOf 的正确使用模式

------

## 标准写法

```java
CompletableFuture<Void> all =
    CompletableFuture.allOf(cf1, cf2, cf3);

CompletableFuture<Result> result =
    all.thenApply(v -> {
        return combine(
            cf1.join(),
            cf2.join(),
            cf3.join()
        );
    });
```

------

## 👉 本质

```text
allOf = 等全部完成
thenApply = 汇总结果
```

------

# 八、anyOf（补充）

------

## 定义

```java
CompletableFuture<Object> f =
    CompletableFuture.anyOf(cf1, cf2, cf3);
```

------

## 👉 含义

```text
只要有一个完成，就触发
```

------

## 👉 对比

| 方法  | 含义     |
| ----- | -------- |
| allOf | 全部完成 |
| anyOf | 任意完成 |

------

# 九、分支 vs allOf（重要对比）

------

## 分支（1 → N）

```java
f1.thenAccept(A);
f1.thenAccept(B);
一个结果 → 多个消费
```

------

## allOf（N → 1）

```java
allOf(f1, f2)
多个结果 → 一个汇总
```

------

# 十、核心模型总结

------

## 🔁 执行模型

```text
注册 → 触发
```

------

## 🔁 数据流

```text
cf1, cf2, cf3
      ↓
    allOf
      ↓
   thenApply
      ↓
   最终结果
```

------

## 🔁 线程模型

```text
thenApply 不阻塞
代码继续往下执行
```

------

# 🔥 最关键总结（必须记住）

------

```text
1️⃣ allOf = “全部完成信号”
2️⃣ thenApply = “完成后执行的逻辑”
3️⃣ thenApply 不会立即执行
4️⃣ 代码会继续往下走（非阻塞）
5️⃣ join 在 allOf 后使用是安全的（不会阻塞）
```

------

# 🚀 你现在掌握的能力

你已经具备：

- 异步编排（多个 future 协作）
- 非阻塞执行模型理解
- allOf / anyOf 使用
- 执行时机精准控制

------

# 🚀 下一步建议

```text
1️⃣ thenCombine（两个 future 合并）
2️⃣ allOf 收集结果的优雅写法
3️⃣ 异常传播（allOf 中一个失败会怎样）
4️⃣ 线程池控制（thenApplyAsync）
```

------

# 🎯 一句话终极理解

```text
CompletableFuture = 注册逻辑 + 条件触发 + 非阻塞执行
```

------