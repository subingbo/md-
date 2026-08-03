# 🚀 Java 依赖冲突 & 类加载机制（进阶笔记）

------

# 一、Maven 依赖冲突（核心问题）

## 1️⃣ 什么是依赖冲突？

```text
多个依赖引入了“同一个库的不同版本”
```

------

## 示例

```text
A → B(2.0)
C → B(1.0)
当前项目 → A + C
```

👉 问题：

```text
到底用 B 1.0 还是 2.0？
```

------

## Maven 规则（必须记）

```text
就近原则（Nearest Definition）
```

👉 距离当前项目最近的依赖优先

------

## 冲突结果

```text
你以为用的是 B 2.0
实际用的是 B 1.0
```

👉 导致：

```text
Cannot resolve method（编译期）
NoSuchMethodError（运行期）
```

------

# 二、如何分析依赖冲突

## 命令

```bash
mvn dependency:tree
```

------

## 你要看什么？

```text
同一个 artifactId 是否出现多个版本
```

------

## 示例输出

```text
[INFO] +- A:1.0
[INFO] |  \- B:2.0
[INFO] \- C:1.0
[INFO]    \- B:1.0   ← 被选中（更近）
```

------

# 三、解决依赖冲突的方法

------

## 1️⃣ 指定版本（推荐）

```xml
<dependency>
    <groupId>xxx</groupId>
    <artifactId>B</artifactId>
    <version>2.0</version>
</dependency>
```

------

## 2️⃣ 使用 dependencyManagement（企业级推荐）

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>xxx</groupId>
            <artifactId>B</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

------

## 3️⃣ 排除依赖（exclusion）

```xml
<dependency>
    <groupId>A</groupId>
    <artifactId>A</artifactId>
    <exclusions>
        <exclusion>
            <groupId>xxx</groupId>
            <artifactId>B</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

------

# 四、为什么会出现 NoSuchMethodError（重点）

```text
编译时：用的是 B 2.0 ✔
运行时：加载的是 B 1.0 ❌
```

👉 JVM 找不到方法 → 报错

------

# 五、类加载机制（ClassLoader）

------

## 1️⃣ JVM 类加载流程

```text
加载 → 验证 → 准备 → 解析 → 初始化
```

------

## 2️⃣ 类加载器层级（必须掌握）

```text
Bootstrap ClassLoader（启动类加载器）
↓
Extension ClassLoader（扩展类加载器）
↓
Application ClassLoader（应用类加载器）
```

------

# 六、双亲委派模型（核心）

## 规则

```text
加载类时：
先让父加载器加载
父加载器不行 → 自己加载
```

------

## 流程

```text
AppClassLoader
   ↓
ExtClassLoader
   ↓
Bootstrap
```

------

## 作用

```text
1️⃣ 防止重复加载
2️⃣ 保证核心类安全（如 java.lang.String）
```

------

# 七、为什么要有双亲委派？

## 安全问题

```text
如果没有双亲委派：
你可以写一个 java.lang.String
替换掉 JDK 的 String ❌
```

------

## 有了双亲委派

```text
Bootstrap 优先加载
→ 不可能被替换 ✔
```

------

# 八、什么时候会“打破双亲委派”？

## 常见场景

```text
SPI（Service Provider Interface）
Tomcat
Spring Boot
```

------

## 原因

```text
需要“下层类加载器加载上层接口的实现”
```

------

# 九、SPI 机制（重点）

## Java 提供：

```text
ServiceLoader
```

------

## 使用方式

```text
META-INF/services/xxx.Interface
```

------

## 本质

```text
接口：JDK 提供
实现：应用提供
```

👉 需要反向加载（打破双亲委派）

------

# 十、Spring Boot 如何处理类加载

（结合你前面学的）

------

## 特点

```text
fat jar（jar 包中嵌套 jar）
```

------

## 问题

```text
JVM 默认不支持 jar-in-jar
```

------

## 解决方案

```text
自定义 ClassLoader
```

------

## 核心类

```text
LaunchedURLClassLoader
JarLauncher
```

------

## 作用

```text
从 BOOT-INF/lib 加载依赖
```

------

# 十一、类加载 + Maven 的关系（关键理解）

```text
Maven：决定有哪些 jar（依赖）
ClassLoader：决定加载哪个 jar
```

------

👉 两者共同决定：

```text
最终用的是哪个类
```

------

# 十二、终极问题链（你现在应该能理解）

------

## ❓ 为什么会：

### Cannot resolve

```text
编译期 classpath 错
```

------

### ClassNotFound

```text
运行期 classpath 缺失
```

------

### NoSuchMethodError

```text
编译期版本 ≠ 运行期版本
```

------

# 🎯 最终总结（必须掌握）

```text
依赖冲突
→ Maven 决定用哪个 jar

类加载
→ JVM 决定加载哪个类

问题本质
→ 两者不一致
```

------

# 🧠 一句话升维理解

```text
Maven 决定“你拥有什么”
ClassLoader 决定“你实际用哪个”
```

------