# Java 类加载机制

## 1. 类加载整体流程

Java 类从 `.class` 文件到可以正常使用，大致经历：

```text
.class 文件
    ↓
【加载 Loading】
    ↓
【链接 Linking】
 ┌──────────────┐
 │  验证 Verify  │
 │  准备 Prepare │
 │  解析 Resolve │ ← 可以延迟
 └──────────────┘
    ↓
【初始化 Initialization】
    ↓
使用 Using
    ↓
卸载 Unloading
```

其中严格来说：

> **类加载过程主要包括：加载、链接、初始化。**
>
> 链接又分为：**验证、准备、解析**。

------

# 2. 加载 Loading

## 2.1 做什么

加载阶段主要负责：

1. 根据类的全限定名找到对应的 `.class` 文件
2. 将 `.class` 字节码加载到 JVM
3. 在 JVM 中建立这个类对应的运行时数据结构
4. 创建对应的 `Class` 对象

例如：

```java
Person person = new Person();
```

第一次需要 `Person` 时，如果当前类加载器还没有加载 `Person`，JVM 就需要加载：

```text
Person.class
    ↓
加载
    ↓
JVM 中的 Person 类
```

------

# 3. 验证 Verification

验证属于链接阶段。

## 3.1 什么时候进行

通常在类加载之后进行。

```text
加载
 ↓
验证
```

## 3.2 验证什么

主要检查：

- `.class` 文件格式是否合法
- 字节码是否合法
- 字节码是否符合 JVM 规范
- 类型关系是否合法
- 符号引用是否能够正确解析

目的：

> 防止错误或恶意的字节码破坏 JVM。

------

# 4. 准备 Preparation

准备阶段也是链接的一部分。

## 4.1 主要工作

为类的**静态字段（static）**分配内存，并设置初始值。

例如：

```java
class Test {
    static int a = 10;
    static String name = "hello";
}
```

准备阶段并不会执行：

```java
a = 10;
name = "hello";
```

而是设置默认值：

```text
a    → 0
name → null
```

------

## 4.2 准备阶段 vs 初始化阶段

### 普通 static 变量

```java
static int a = 10;
```

准备：

```text
a = 0
```

初始化：

```text
a = 10
```

------

### 编译期常量

```java
static final int A = 10;
```

这是编译期常量。

准备阶段可以直接设置：

```text
A = 10
```

而不需要等 `<clinit>` 执行。

------

# 5. 解析 Resolution

解析也是链接阶段的一部分。

## 5.1 什么是符号引用

例如：

```java
class A {
    B b;

    void test() {
        B.hello();
    }
}
```

编译后的 `.class` 文件中，并不能简单地直接保存 JVM 运行时内存地址。

常量池中保存的是类似：

```text
B
B.hello
```

这种描述性的引用。

这叫：

> **符号引用（Symbolic Reference）**

------

## 5.2 什么是直接引用

解析就是把：

```text
符号引用
    ↓
直接引用
```

例如：

```text
"B.hello()"
    ↓
JVM 中能够直接定位 B.hello() 的运行时引用
```

------

# 6. 为什么说解析可以延迟？

JVM **不要求所有符号引用必须在链接阶段立即完成解析**。

例如：

```java
class A {
    void test() {
        B.hello();
    }
}
```

加载 A 时，可以先：

```text
加载 A
 ↓
验证 A
 ↓
准备 A
 ↓
暂时不解析 B.hello()
```

等真正执行：

```java
A.test();
```

执行到：

```java
B.hello();
```

JVM 发现需要知道 `B.hello()` 到底在哪里：

```text
需要 B.hello()
    ↓
检查 B 是否已经加载
    ↓
如果没有 → 加载 B
    ↓
链接 B
    ↓
解析 B.hello()
    ↓
调用方法
```

所以：

> **解析可以延迟到真正使用某个符号引用的时候。**

------

# 7. 加载和解析不要混淆

假设：

```java
class A {
    B b;
}
```

A 中存在 B 的引用。

不能简单认为：

```text
加载 A
 ↓
马上加载 B
```

因为 A 中可能只是存在：

```text
B b
```

这样的符号引用。

可以理解成：

```text
A
 └── B 的符号引用
```

只有真正需要 B 的时候，才可能进一步触发 B 的加载、链接、解析。

------

# 8. 初始化 Initialization

初始化是类加载过程中非常重要的一步。

核心工作：

> **执行类的初始化代码。**

主要包括：

- static 变量赋值
- static 代码块

例如：

```java
class Test {

    static int a = 10;

    static {
        System.out.println("Test init");
    }
}
```

编译器会把这些初始化操作组织到：

```text
<clinit>()
```

中。

可以粗略理解成：

```java
static void <clinit>() {
    a = 10;
    System.out.println("Test init");
}
```

------

# 9. 类什么时候初始化？

核心原则：

> **类发生主动使用（Active Use）时，会触发初始化。**

常见主动使用包括：

## 9.1 new 对象

```java
new A();
```

会触发 A 初始化。

------

## 9.2 调用静态方法

```java
A.test();
```

会触发 A 初始化。

------

## 9.3 使用静态字段

例如：

```java
System.out.println(A.value);
```

如果 `value` 不是编译期常量，会触发 A 初始化。

------

## 9.4 反射

例如：

```java
Class.forName("com.example.A");
```

默认会进行类初始化。

------

## 9.5 初始化子类

```java
class Parent {
    static {
        System.out.println("Parent");
    }
}

class Child extends Parent {
    static {
        System.out.println("Child");
    }
}
```

执行：

```java
new Child();
```

初始化顺序：

```text
Parent
 ↓
Child
```

即：

> **初始化子类之前，必须先初始化父类。**

------

# 10. 哪些情况不会触发初始化？

这是面试重点。

## 10.1 `A.class`

```java
System.out.println(A.class);
```

通常不会触发 A 的初始化。

例如：

```java
class A {
    static {
        System.out.println("A init");
    }
}
```

执行：

```java
System.out.println(A.class);
```

不会打印：

```text
A init
```

------

# 11. 编译期常量不会触发初始化

例如：

```java
class A {

    static final int X = 10;

    static {
        System.out.println("A init");
    }
}
```

执行：

```java
System.out.println(A.X);
```

不会触发 A 初始化。

为什么？

因为：

```java
static final int X = 10;
```

是**编译期常量**。

编译器可以在编译阶段就知道：

```text
A.X = 10
```

因此使用方的字节码中可以直接使用：

```text
10
```

而不需要运行：

```text
A.<clinit>()
```

------

# 12. 但不是所有 static final 都是编译期常量

例如：

```java
class A {

    static final int X = Integer.parseInt("10");

    static {
        System.out.println("A init");
    }
}
```

这里：

```java
Integer.parseInt("10")
```

需要运行时执行。

因此：

```java
System.out.println(A.X);
```

会触发 A 初始化。

------

# 13. `static final` 的判断

不要记成：

> static final 就不会初始化。

应该记成：

> **只有编译期常量才不会因为访问该常量而触发类初始化。**

例如：

```java
static final int A = 10;
```

属于编译期常量。

而：

```java
static final int B = Integer.parseInt("10");
```

不是。

------

# 14. 类中的其他类什么时候加载？

假设：

```java
class A {
    B b;

    void test() {
        B.hello();
    }
}
```

A 被加载：

```text
加载 A
```

并不意味着：

```text
B 一定立即加载
```

A 中只是存在：

```text
B 的符号引用
```

真正需要 B 时：

```text
执行 B.hello()
     ↓
需要解析 B.hello()
     ↓
如果 B 尚未加载
     ↓
加载 B
     ↓
链接 B
     ↓
解析 B.hello()
     ↓
执行
```

------

# 15. 父类是特殊情况

例如：

```java
class B {
}

class A extends B {
}
```

A 的父类是 B。

JVM 必须知道 A 的父类，因此 B 的加载和链接不能像普通引用那样完全随意延迟。

而初始化时：

```java
new A();
```

顺序一定是：

```text
B 初始化
    ↓
A 初始化
```

------

# 16. 内部类什么时候加载？

Java 内部类实际上也是独立的 class。

例如：

```java
class Outer {

    static class Inner {
        static {
            System.out.println("Inner init");
        }
    }
}
```

编译后大致是：

```text
Outer.class
Outer$Inner.class
```

加载：

```java
Outer.class
```

并不意味着：

```text
Outer$Inner.class
```

一定同时初始化。

执行：

```java
new Outer.Inner();
```

才会触发 Inner 的主动使用和初始化。

------

# 17. Class.forName 和 ClassLoader.loadClass

这是类加载面试中的经典问题。

### Class.forName

```java
Class.forName("com.example.A");
```

默认：

```text
加载
 ↓
链接
 ↓
初始化
```

------

### ClassLoader.loadClass

```java
classLoader.loadClass("com.example.A");
```

默认主要负责：

```text
加载
 ↓
链接
```

通常**不会立即初始化 A**。

因此：

```text
Class.forName()
    → 默认初始化

ClassLoader.loadClass()
    → 默认不初始化
```

------

# 18. 类加载器

类并不是 JVM 自己直接从磁盘随便读取，而是通过：

> **ClassLoader（类加载器）**

负责加载。

常见类加载器：

```text
Bootstrap ClassLoader
        ↓
Platform ClassLoader
        ↓
Application ClassLoader
```

------

# 19. 双亲委派模型

当 Application ClassLoader 想加载一个类时，并不是自己直接加载。

大致过程：

```text
Application ClassLoader
          ↓
请求父加载器
          ↓
Platform ClassLoader
          ↓
请求父加载器
          ↓
Bootstrap ClassLoader
          ↓
能加载？
   ↓              ↓
  能              不能
  ↓                ↓
返回 Class     子加载器尝试
```

核心思想：

> **优先让父加载器尝试加载，父加载器加载不了，再由子加载器加载。**

------

# 20. 为什么需要双亲委派？

最重要的原因：

## 防止核心类被篡改

例如自己写：

```java
package java.lang;

public class String {
}
```

如果没有双亲委派，自己的类加载器可能加载这个 `String`。

有了双亲委派：

```text
加载 java.lang.String
        ↓
Bootstrap ClassLoader
        ↓
找到 JDK 自带 String
        ↓
直接加载
```

不会使用你自己写的 `String`。

------

# 21. 类的卸载

类加载完成后并不是永远存在。

当一个类满足卸载条件时，JVM 可以卸载它。

通常与：

```text
类
+
Class 对象
+
对应的 ClassLoader
```

之间的可达关系有关。

特别是在：

- Tomcat 热部署
- 插件系统
- OSGi
- 自定义 ClassLoader

中比较重要。

------

# 22. 最容易混淆的几个概念

## 加载 ≠ 初始化

```text
加载
 ↓
类进入 JVM
```

不代表：

```text
<clinit>() 已经执行
```

------

## 解析 ≠ 初始化

解析：

```text
符号引用 → 直接引用
```

初始化：

```text
执行 <clinit>()
```

完全是两件事情。

------

## 加载 B ≠ 初始化 B

例如：

```text
B.class
```

可能已经被加载，但 B 的：

```text
<clinit>()
```

还没有执行。

------

## static final ≠ 一定不会初始化

只有：

```text
编译期常量
```

才有特殊规则。

------

# 23. 一张完整流程图

```text
                    .class 文件
                         │
                         ▼
                    ┌────────┐
                    │  加载  │
                    └────┬───┘
                         │
                         ▼
                    ┌────────┐
                    │  验证  │
                    └────┬───┘
                         │
                         ▼
                    ┌────────┐
                    │  准备  │
                    └────┬───┘
                         │
                         ▼
                    ┌────────┐
                    │  解析  │
                    └────┬───┘
                         │
                  可以延迟解析
                         │
                         ▼
                 ┌──────────────┐
                 │   主动使用？  │
                 └──────┬───────┘
                        是
                         │
                         ▼
                    ┌────────┐
                    │ 初始化 │
                    │<clinit>│
                    └────┬───┘
                         │
                         ▼
                      正常使用
                         │
                         ▼
                       卸载
```

------

# 24. 面试版总结

可以直接记下面这段：

> Java 类从 `.class` 文件到运行主要经历加载、链接、初始化三个阶段。链接又包括验证、准备和解析。加载负责把 `.class` 加入 JVM 并建立类的运行时结构；验证负责检查字节码合法性；准备负责为静态字段分配空间并设置默认值；解析负责将常量池中的符号引用转换为直接引用，而且解析可以延迟到实际使用时；初始化阶段执行类的 `<clinit>()`，完成静态变量赋值和静态代码块执行。类发生主动使用时会触发初始化，例如 `new`、调用静态方法、访问非编译期常量静态字段、反射等，而 `A.class` 和访问编译期常量通常不会触发初始化。

### 最终记忆框架

```text
加载：把 class 搬进 JVM
验证：检查 class
准备：static 分配空间 + 默认值
解析：符号引用 → 直接引用（可延迟）
初始化：执行 <clinit>
使用：真正运行
卸载：类被回收
```

最关键的三组区别：

```text
加载 ≠ 初始化

解析 ≠ 初始化

static final ≠ 编译期常量
```

其中第三组尤其容易出面试题。