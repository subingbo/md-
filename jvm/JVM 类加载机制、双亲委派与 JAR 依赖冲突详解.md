# JVM 类加载机制、双亲委派与 JAR 依赖冲突详解

## 一、先记住六个核心结论

1. **类加载器不是直接把** `**.class**` **文件放进堆，而是读取字节码，由 JVM 建立运行时类信息。**
2. `**import**` **不指定类加载器。** 普通字节码类引用通常由引用该类的代码所属类的定义加载器发起解析。
3. **双亲委派的顺序：检查已加载 → 委托父加载器 → 父加载器失败后自己加载。**
4. JVM 判断两个类是否相同，需要同时比较**类的全限定名和定义类加载器**。
5. 默认父加载器优先有利于共享公共类、统一类型身份、保护核心类，但不保证依赖版本正确。
6. **两个 JAR 中存在同名类，不一定立即报错。** 可能只加载其中一个版本，直到运行时才出现兼容性问题。

## 二、类加载到底是什么？

### 2.1 Java 程序的基本运行过程

假设我们定义：

```
package com.example;

public class User {
    private int age = 18;

    public void sayHello() {
        System.out.println("Hello");
    }
}
```

首先，`javac` 将 `User.java` 编译成 `User.class`。

```
User.java
    ↓ javac 编译
User.class
    ↓ 类加载器读取字节码
JVM 建立运行时类信息
    ├── 元空间：类元数据
    └── Java 堆：对应的 Class 对象
```

对于 JDK 8 及以后的 HotSpot JVM：

| 内容                    | 位置             | 作用                    |
| ----------------------- | ---------------- | ----------------------- |
| `.class` 文件           | 磁盘、JAR 等来源 | 保存编译后的字节码      |
| 类元数据                | 主要在元空间     | JVM 执行、管理类时使用  |
| `java.lang.Class` 对象  | Java 堆          | Java 层访问类信息的入口 |
| `new User()` 生成的实例 | 通常在 Java 堆   | 保存实例字段等数据      |

方法区是 JVM 规范定义的逻辑内存区域，而元空间是现代 HotSpot 存储类元数据的主要实现机制。

**Class 对象不是把全部类元数据复制到堆里。** 它关联底层类信息，为反射等 Java API 提供对象化入口。

### 2.2 类加载的生命周期

```
加载（Loading）
    ↓
连接（Linking）
    ├── 验证（Verification）
    ├── 准备（Preparation）
    └── 解析（Resolution）
    ↓
初始化（Initialization）
    ↓
使用（Using）
    ↓
卸载（Unloading）
```

各阶段作用：

| 阶段   | 核心任务                         |
| ------ | -------------------------------- |
| 加载   | 找到字节码并建立运行时类结构     |
| 验证   | 检查字节码是否合法               |
| 准备   | 为静态字段分配存储并赋予初始值   |
| 解析   | 将符号引用解析为直接引用         |
| 初始化 | 执行静态字段显式赋值、静态代码块 |
| 使用   | 创建实例、调用方法等             |
| 卸载   | 满足条件时回收类相关资源         |

其中，解析可以按规范延迟进行，并非所有类都必须在初始化前完成全部解析。

### 2.3 类初始化与实例初始化不同

```
class User {
    static int count = 100;

    int age = 18;

    static {
        System.out.println("类初始化");
    }

    public User() {
        System.out.println("实例初始化");
    }
}
```

执行：

```
new User();
new User();
```

通常输出：

```
类初始化
实例初始化
实例初始化
```

原因：

- `<clinit>`：负责类初始化，成功初始化的类通常只执行一次。
- `<init>`：实例初始化方法，包含构造器相关逻辑，每次正常调用构造器创建对象都会执行相应初始化。
- 多次 `new` 不代表多次加载同一个类。

## 三、Java 有哪些类加载器？

以 JDK 9 及以后的标准加载器体系为例：

```
Bootstrap ClassLoader
        ↑
Platform ClassLoader
        ↑
Application ClassLoader
        ↑
Custom ClassLoader（常见情况）
```

这里箭头表示父加载器委托关系，不是 Java 的类继承关系。

### 3.1 Bootstrap ClassLoader（启动类加载器）

主要负责 Java 核心类和相关模块。

例如：

```
java.lang.String
java.lang.Object
java.lang.Integer
```

HotSpot 中启动类加载器主要由原生代码实现。

```
System.out.println(String.class.getClassLoader());
// null
```

返回 `null` 是这个 API 表示启动类加载器的方式，并非没有加载器。

### 3.2 Platform ClassLoader（平台类加载器）

负责部分 Java 平台模块中的类。

版本区别：

- JDK 8：Extension ClassLoader（扩展类加载器）。
- JDK 9+：Platform ClassLoader（平台类加载器）。

它们不是完全相同的实现。

### 3.3 Application ClassLoader（应用程序类加载器）

负责普通应用类路径中的类。

例如：

```
public class UserService {
}
```

在常规 Java 项目中，`UserService` 通常由 Application ClassLoader 定义。

### 3.4 Custom ClassLoader（自定义类加载器）

开发者可以继承 `ClassLoader` 实现自定义加载逻辑。

常见用途：

- 插件系统类隔离。
- 加载不同版本的依赖。
- 从网络或自定义存储读取字节码。
- 特殊的模块化加载机制。

**注意：自定义类加载器不一定必须以 Application ClassLoader 为父加载器。**

## 四、执行 new 时，JVM 怎么选择类加载器？

这是理解整个类加载机制最重要的一步。

### 4.1 import 不负责选择类加载器

例如：

```
import com.example.User;

public class Main {
    public static void main(String[] args) {
        User user = new User();
    }
}
```

`import` 主要在编译阶段帮助确定类名。

编译后的字节码包含对 `com.example.User` 的符号引用，而不是记录某个固定的类加载器实例。

### 4.2 初始加载器如何确定？

假设 `Main` 已经由 Application ClassLoader 定义。

执行：

```
User user = new User();
```

JVM 在解析 `Main` 字节码中引用的 `User` 时，通常使用定义 `Main` 的类加载器发起加载请求。

过程：

```
Main 由 Application 定义
          ↓
Main 执行 new User
          ↓
需要解析 User 类
          ↓
由 Main 的定义加载器发起请求
          ↓
Application ClassLoader
          ↓
按双亲委派规则寻找 User
```

**这里有两个不同的问题：**

1. 谁发起加载请求？
2. 最后由谁真正定义这个类？

两者可能不是同一个加载器。

例如，Application 发起 `java.lang.String` 的加载请求，最终可能由 Bootstrap 定义。

### 4.3 怎么验证？

```
public class Main {
    public static void main(String[] args) {
        System.out.println(
            Main.class.getClassLoader()
        );

        System.out.println(
            String.class.getClassLoader()
        );
    }
}
```

典型输出：

```
jdk.internal.loader.ClassLoaders$AppClassLoader@...
null
```

反射、SPI、线程上下文类加载器等动态加载机制，可能使用不同的加载器选择方式。

## 五、双亲委派机制的完整流程

### 5.1 什么是双亲委派？

当某个类加载器收到加载请求时：

1. 先检查该类是否已经加载。
2. 如果没有，优先委托父加载器。
3. 父加载器也继续向上委托。
4. 最顶层加载器无法完成时，逐层向下返回。
5. 每一层在父加载失败后尝试自己加载。
6. 某一层加载成功，就将对应的 Class 对象返回。

### 5.2 完整实例

假设 Application 要加载：

```
com.example.User
```

过程：

```
Application 收到请求
      │
      ▼
检查是否已加载
      │ 未加载
      ▼
委托 Platform
      │
      ▼
Platform 委托 Bootstrap
      │
      ▼
Bootstrap 查找 User
      │ 找不到
      ▼
Platform 自己查找
      │ 找不到
      ▼
Application 自己查找
      │ 找到 User.class
      ▼
加载成功，返回 Class 对象
```

注意：

**不是 Bootstrap 加载失败后直接跳回 Application。**

应当先由 Platform 自己尝试，再向下返回 Application。

### 5.3 核心伪代码

Java `ClassLoader.loadClass()` 的默认逻辑可以简化为：

```
protected Class<?> loadClass(String name)
        throws ClassNotFoundException {

    // 1. 检查该加载器是否已经加载过
    Class<?> c = findLoadedClass(name);

    if (c == null) {
        try {
            // 2. 优先委托父加载器
            c = parent.loadClass(name);
        } catch (ClassNotFoundException e) {
            // 3. 父加载器无法加载
            // 当前加载器再自己查找
            c = findClass(name);
        }
    }

    return c;
}
```

这是原理伪代码，省略了 Bootstrap、同步控制、类解析等实现细节。

三个关键方法：

| 方法                | 作用                       |
| ------------------- | -------------------------- |
| `findLoadedClass()` | 查询已有的加载记录         |
| `loadClass()`       | 按加载策略处理请求         |
| `findClass()`       | 当前加载器自己查找并定义类 |

其中 `findClass()` 是开发自定义类加载器时经常需要实现的方法。

**默认双亲委派属于** `**ClassLoader**` **的加载策略，不是 JVM 规定所有类加载器必须遵循的唯一方式。**

## 六、什么叫父加载器找不到类？

假设请求：

```
loadClass("com.example.User");
```

对于基于类路径加载的实现，对应资源一般是：

```
com/example/User.class
```

但这里不是电脑磁盘的绝对路径，而是类资源路径。

例如：

```
D:/project/target/classes/com/example/User.class
```

如果 Application 的 ClassPath 包含：

```
D:/project/target/classes/
```

就可能找到它。

而 Bootstrap 主要负责 Java 核心类，它通常无法从自己的类来源提供 `com.example.User`。

因此：

```
Bootstrap：找不到
Platform：找不到
Application：找得到
```

这里的“找不到”只表示**该加载器无法提供请求的类**，不代表整个 JVM 或磁盘上没有这个文件。

如果父加载器因为字节码无效、链接错误等原因失败，也不一定能够简单地回退给子加载器；默认代码主要针对 `ClassNotFoundException` 进行回退。

## 七、为什么采用父加载器优先？

主要有三个原因。

### 7.1 优先复用公共类

假设：

```
父加载器
└── com.example.User（V1）

子加载器
└── com.example.User（V2）
```

如果默认父优先：

```
子加载器请求 User
        ↓
父加载器找到 V1
        ↓
返回 V1
        ↓
子加载器不再定义 V2
```

这样父子加载器就可以共享父加载器定义的 `User` 类型。

### 7.2 避免本应共享的类出现多种类型身份

JVM 识别运行时类型时，依据：

**类的全限定名 + 定义类加载器。**

例如：

```
(com.example.User, LoaderA)

(com.example.User, LoaderB)
```

如果由 A 和 B 分别定义，即使字节码完全一致，JVM 也认为它们是不同的运行时类型。

可能引发：

```
ClassCastException
```

因此，父优先可以减少公共类型被不同子加载器重复定义的情况。

### 7.3 保护 Java 核心类

假设有人尝试自己定义：

```
package java.lang;

public class String {
}
```

正常应用不允许随便替换 JDK 自带的 `java.lang.String`。

父加载器优先让 Java 核心类优先由受信任的上层加载器提供。

此外，现代 JDK 还有模块封装和 `java.*` 包定义限制，所以核心类保护并不只依赖双亲委派。

### 7.4 父优先不保证版本正确

假设插件编译时使用 V2：

```
Utils.newMethod();
```

但父加载器只能提供 V1，并且 V1 没有 `newMethod()`。

运行时可能发生：

```
java.lang.NoSuchMethodError
```

因此：

**双亲委派主要解决类型共享和加载优先级问题，不负责自动选择业务需要的正确依赖版本。**

## 八、自己定义 String 会怎么样？

### 8.1 包名不同：可以正常加载

你自己定义：

```
package com.example;

public class String {
}
```

JDK 自带：

```
java.lang.String
```

两者完整名称不同。

| 类          | 全限定名             |
| ----------- | -------------------- |
| 你的 String | `com.example.String` |
| JDK String  | `java.lang.String`   |

当 Application 请求 `com.example.String` 时：

```
Application
    ↓ 委托
Platform
    ↓ 委托
Bootstrap
    ↓ 找不到 com.example.String
Platform
    ↓ 找不到
Application
    ↓ 找到
成功加载
```

Bootstrap 虽然能够加载 `java.lang.String`，但它不能将其当作 `com.example.String` 返回。

### 8.2 全限定名相同：存在冲突

如果试图定义：

```
package java.lang;

public class String {
}
```

这与 JDK 核心类全限定名相同。

普通现代 Java 应用会受到模块与核心包限制，无法通过这种方式正常替换内置 String。

### 8.3 结论

**类加载器主要按全限定名查找类，而不是按简单类名，也不是比较源代码是否相同。**

源码文件路径、类资源路径、全限定名、定义加载器需要区分：

| 概念       | 示例                                  |
| ---------- | ------------------------------------- |
| 源码路径   | `src/main/java/com/example/User.java` |
| 全限定名   | `com.example.User`                    |
| 类资源路径 | `com/example/User.class`              |
| 定义加载器 | Application、PluginLoader 等          |

## 九、同一个类加载器会重复加载同一个类吗？

正常情况下不会重新定义。

例如：

```
User u1 = new User();
User u2 = new User();
User u3 = new User();
```

通常：

- `User` 的运行时类定义只有一份。
- 对应的 `Class` 对象只有一个。
- 创建三个不同的 User 实例。
- `<init>` 对应的实例初始化执行三次。

验证：

```
System.out.println(
    u1.getClass() == u2.getClass()
);
// true
```

如果绕过正常已加载检查，让同一个定义加载器再次调用 `defineClass()` 定义同名类，通常会抛出 `LinkageError`。

但不同的定义加载器可以分别定义全限定名相同的类。

### 9.1 加载器不同，结果一定不同吗？

不一定。

假设加载器 A 和 B 都调用：

```
loadClass("com.example.User");
```

若它们最终都委托同一个父加载器定义该类，返回的就是同一个 `Class` 对象。

只有 A、B 各自定义了同名类，才会产生两个不同的运行时类型。

## 十、两个 JAR 包含同名类，会发生什么？

这是实际 Java 后端开发中最重要的应用场景。

假设：

```
tools-v1.jar
└── com/example/Utils.class

tools-v2.jar
└── com/example/Utils.class
```

需要分情况讨论。

### 10.1 情况一：两个 JAR 属于同一个 Application ClassLoader

```
Application ClassLoader
├── tools-v1.jar
│   └── com.example.Utils（V1）
└── tools-v2.jar
    └── com.example.Utils（V2）
```

一般不会因为两个 JAR 存在同名类就立即报错。

典型类路径查找中：

1. Application 请求加载 `com.example.Utils`。
2. 根据类路径查找规则找到一个版本。
3. 例如先找到 V1。
4. 定义 V1。
5. 后续请求复用已加载的 V1。
6. 同一个加载器不会再正常定义 V2。

**注意：这里主要由类路径资源查找顺序决定，并不是双亲委派在两个 JAR 之间选版本。**

### 10.2 情况二：父子加载器各有一个版本

```
Application ClassLoader（父）
└── Utils V1

Plugin ClassLoader（子）
└── Utils V2
```

如果 Plugin 使用默认父优先策略：

```
Plugin 请求 Utils
      ↓
Application 找到 V1
      ↓
返回 V1
      ↓
Plugin 使用 V1
```

V2 不会被定义。

即使插件需要 V2，也不会自动跳过 V1。

### 10.3 情况三：插件必须使用 V2

有两种常见办法。

**办法 A：让父加载器不再提供 V1**

```
Application
└── 不包含 Utils

Plugin
└── Utils V2
```

默认双亲委派下：

```
Plugin 请求 Utils
      ↓
Application 找不到
      ↓
Plugin 加载 V2
```

**办法 B：自定义子加载器优先**

```
Plugin 请求 Utils
      ↓
优先从 tools-v2.jar 查找
      ↓
Plugin 定义 V2
```

这样父加载器的 V1 和子加载器的 V2 可以同时存在。

但是，两者仍是不同的运行时类型，不一定能够相互转换。

### 10.4 三种情况对比

| 场景                             | 结果                           |
| -------------------------------- | ------------------------------ |
| 同一个加载器，两个 JAR 含同名类  | 通常按类路径顺序选择一个       |
| 父子加载器都有同名类，默认父优先 | 子加载器复用父版本             |
| 自定义子优先                     | 子加载器可以单独定义自己的版本 |
| 同一加载器重复强行定义同名类     | 可能发生 LinkageError          |

## 十一、Maven 依赖冲突与双亲委派的区别

这两个概念很容易混淆。

### 11.1 Maven 解决什么问题？

Maven 在构建阶段负责解析依赖。

例如：

```
项目
├── dependency-A
│   └── jackson-databind:2.15
└── dependency-B
    └── jackson-databind:2.17
```

如果发生同一个依赖坐标的版本冲突，Maven 通常依据依赖调解规则选择一个版本。

常见规则：

- 最近路径优先。
- 同一深度时，先声明者优先。
- 依赖管理可以显式指定版本。

### 11.2 Maven 不能自动解决所有同名类问题

例如两个不同的依赖：

```
lib-a.jar
└── com/example/Utils.class

lib-b.jar
└── com/example/Utils.class
```

虽然两个 JAR 的 Maven 坐标不同，但内部都包含同一个全限定名。

Maven 的普通版本调解不一定能够发现并消除这种重复类。

### 11.3 Maven 与双亲委派各管什么？

| 机制           | 发生阶段       | 主要作用                           |
| -------------- | -------------- | ---------------------------------- |
| Maven 依赖调解 | 构建阶段       | 决定使用哪些依赖版本               |
| ClassPath 查找 | 运行时加载阶段 | 确定从哪个可见资源取得字节码       |
| 双亲委派       | 运行时加载阶段 | 决定优先委托谁提供类               |
| JVM 类链接     | 运行时         | 验证和解析类之间的类型、方法等关系 |

因此，**双亲委派不能替代 Maven 依赖管理，Maven 也不能替代类加载器的运行时机制。**

## 十二、常见类加载异常

| 异常                     | 典型原因                                             |
| ------------------------ | ---------------------------------------------------- |
| `ClassNotFoundException` | 显式加载请求找不到指定类，例如 `Class.forName()`     |
| `NoClassDefFoundError`   | JVM 运行时无法取得需要的类定义，或先前类初始化失败等 |
| `ClassCastException`     | 两个运行时类型不兼容，强制转换失败                   |
| `NoSuchMethodError`      | 运行时加载的类缺少编译时代码期望的方法               |
| `NoSuchFieldError`       | 运行时类缺少所需字段                                 |
| `LinkageError`           | 类链接或定义发生冲突，例如同一加载器重复定义同名类   |

### 12.1 ClassNotFoundException

```
Class.forName("com.example.NotExist");
```

如果类无法找到，可能抛出：

```
java.lang.ClassNotFoundException
```

它是受检异常。

### 12.2 NoClassDefFoundError

例如：

项目编译时有 `User.class`，但运行时缺少对应依赖。

代码执行到需要解析该类时，可能抛出：

```
java.lang.NoClassDefFoundError
```

它是 `Error`，不是受检异常。

此外，类初始化失败后再次使用该类，也可能引发这个错误。

### 12.3 NoSuchMethodError

假设 V2 有：

```
public void newMethod() {
}
```

但 V1 没有。

代码编译时使用 V2，运行时却加载 V1：

```
utils.newMethod();
```

就可能抛出：

```
java.lang.NoSuchMethodError
```

这是很典型的依赖版本不兼容问题。

### 12.4 ClassCastException

假设两个加载器分别定义相同的 `com.example.User`。

虽然代码完全相同，但两个运行时类型不同。

把加载器 B 创建的 User 实例强制转换为加载器 A 定义的 User 类型，可能抛出：

```
java.lang.ClassCastException
```

因此排查类转换异常时，也需要关注类加载器，而不只是类名。

## 十三、最小可运行实验：证明相同字节码也能变成两个不同类型

### 13.1 定义 User

文件：

```
src/demo/User.java
package demo;

public class User {
    public int age = 18;
}
```

### 13.2 定义 Main

文件：

```
src/Main.java
import java.net.URL;
import java.net.URLClassLoader;
import java.nio.file.Paths;

public class Main {
    public static void main(String[] args)
            throws Exception {

        URL url = Paths.get("out").toUri().toURL();
        URL[] urls = {url};

        try (
            URLClassLoader loaderA =
                new URLClassLoader(urls, null);

            URLClassLoader loaderB =
                new URLClassLoader(urls, null)
        ) {
            Class<?> classA =
                loaderA.loadClass("demo.User");

            Class<?> classB =
                loaderB.loadClass("demo.User");

            System.out.println(
                classA.getName().equals(classB.getName())
            );

            System.out.println(classA == classB);

            System.out.println(classA.getClassLoader());
            System.out.println(classB.getClassLoader());

            Object objB =
                classB.getDeclaredConstructor().newInstance();

            // 尝试把 B 的对象转换为 A 的类型
            classA.cast(objB);
        }
    }
}
```

这里两个 URLClassLoader 使用相同的字节码目录，但它们是不同的定义加载器。

`parent = null` 表示使用启动类加载器作为父级委托入口，避免它们通过普通应用加载器共享 User。

### 13.3 编译和执行

在项目根目录运行：

```
javac -d out src/demo/User.java src/Main.java
java -cp out Main
```

关键输出类似：

```
true
false
java.net.URLClassLoader@...
java.net.URLClassLoader@...
Exception in thread "main" java.lang.ClassCastException
```

说明：

- 第一个 `true`：两个类全限定名相同。
- 第二个 `false`：不是同一个 Class 对象。
- 不同的加载器对象分别定义了两个 User 类型。
- `classA.cast(objB)` 失败，因为对象 B 不是 A 定义的 User 类型。

**这个实验直接证明：JVM 的类型身份不只取决于代码内容。**

## 十四、实际项目怎么排查类加载与 JAR 冲突？

### 14.1 查看 Maven 依赖树

```
mvn dependency:tree
```

查看某个特定依赖：

```
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

用于判断 Maven 最终解析了哪些版本。

### 14.2 查看某个类到底来自哪个 JAR

```
System.out.println(
    User.class.getProtectionDomain()
        .getCodeSource()
        .getLocation()
);
```

对于普通应用类，通常可以看到对应的 classes 目录或者 JAR 文件路径。

注意：部分核心类或特殊类加载环境中，`getCodeSource()` 可能返回 `null`，因此生产代码应注意空值检查。

### 14.3 查看类由哪个加载器加载

```
System.out.println(
    User.class.getClassLoader()
);
```

普通项目通常显示 AppClassLoader。

### 14.4 查看 JVM 的实际类加载记录

JDK 9+：

```
java -Xlog:class+load=info Main
```

JDK 8 常见方式：

```
java -verbose:class Main
```

这些命令有助于观察实际加载了哪些类及其来源。

### 14.5 检查 JAR 是否含有特定类

PowerShell：

```
jar tf .\tools-v1.jar |
    Select-String "com/example/Utils.class"

jar tf .\tools-v2.jar |
    Select-String "com/example/Utils.class"
```

如果两个 JAR 都存在这个类，就需要进一步确认构建依赖和实际运行时的加载来源。

还可以使用 Maven Enforcer 的扩展规则进行重复类检查，例如额外配置 `BanDuplicateClasses`，而不是仅依赖普通 `dependency:tree`。

## 十五、面试常见追问

### Q1：为什么使用双亲委派？

答：

双亲委派让子加载器优先复用父加载器能够提供的类，减少本应共享的类型被重复定义的情况，并有助于保护 Java 核心类库和保持类型一致性。

### Q2：双亲委派能避免所有重复类吗？

不能。

它主要影响父子加载器之间的加载优先级。同一个加载器的多个 JAR 仍然可能存在同名类，不同隔离加载器也可以各自定义同名类。

### Q3：两个不同类加载器加载同一个类，一定得到不同类型吗？

不一定。

如果两个加载器最终都委托同一个父加载器定义，得到的是同一个类型。

只有分别定义同名类，才会得到不同运行时类型。

### Q4：为什么我没有指定加载器，new 也能工作？

因为 JVM 会根据当前类的定义加载器等运行时信息，解析字节码中的目标类引用。

普通 `new` 不要求开发者手动选择类加载器。

### Q5：子加载器能不能加载父加载器已经存在的类的另一个版本？

默认父优先机制下一般不会。

如果需要，需要调整依赖可见范围，或者采用特定的子加载器优先策略。

### Q6：为什么 Java 不默认子加载器优先？

子优先可以实现依赖隔离，但如果各子加载器都定义一份公共接口或共享类，就可能发生运行时类型不兼容。

Java 默认优先选择公共类型共享。需要隔离的容器或插件系统可以采用专门的加载策略。

### Q7：类加载器是不是只负责读取文件？

不是。

类加载器的职责包括定位类字节码来源、读取字节码以及通过 JVM 机制定义类。字节码不一定来自磁盘文件，也可以来自内存、网络等。

验证、准备、解析和初始化属于 JVM 类加载生命周期中的相关阶段。

## 十六、最终知识链路

```
Java 源码
    ↓
javac 编译
    ↓
.class 字节码
    ↓
类加载器处理加载请求
    ↓
检查是否已经加载
    ↓
按委派策略寻找能够提供类的加载器
    ↓
JVM 定义运行时类
    ├── 类元数据
    └── Class 对象
    ↓
链接、初始化（按 JVM 规则进行）
    ↓
new 对象
    ↓
堆中分配实例（通常）
    ↓
实例初始化 <init>
    ↓
程序使用对象
```

### 三个最容易混淆的问题

**1. 类名相同，不等于运行时类型相同。**

类型身份取决于全限定名和定义类加载器。

**2. 加载请求来自某个加载器，不代表最终由它定义。**

例如 Application 发起 String 的加载请求，最终可能由 Bootstrap 定义。

**3. 双亲委派不是 Maven 依赖冲突处理器。**

前者是运行时类加载优先级机制，后者主要是构建阶段的依赖解析机制。

### 最终面试总结

> Java 类加载器用于查找类字节码，并由 JVM 根据字节码建立运行时类定义。
>
> 默认双亲委派机制首先检查是否已加载，然后优先委托父加载器。父加载器无法完成加载时，才由当前加载器自己尝试。
>
> JVM 使用类的全限定名和定义类加载器共同确定运行时类型身份，因此不同类加载器分别定义的同名类可以共存，但可能无法相互转换。
>
> 双亲委派能够促进公共类共享和核心类型保护，却不能自动解决不同 JAR 的依赖版本冲突。实际项目中需要结合 Maven 依赖树、类加载日志和 JAR 内容进行排查。