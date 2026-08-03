# 🚀 Java 构建 & Maven & 编译问题全景笔记

------

# 一、核心总览（必须先记住）

```text
Java 问题本质就三件事：

1️⃣ 有没有这个类（class）
2️⃣ 有没有这个方法（method）
3️⃣ JVM/编译器能不能找到它（classpath）
```

------

# 二、Maven问题 vs 编译问题

## 1️⃣ Maven 问题（依赖 / classpath）

### 本质

```text
classpath 不正确
```

### 表现

```text
Cannot resolve symbol
Cannot resolve method
package xxx does not exist
```

### 常见原因

```text
- 缺依赖
- 依赖版本冲突
- scope 错误（test / provided）
- Maven 未刷新 / 未编译
```

------

## 2️⃣ 编译问题（Java语法 / 类型）

### 本质

```text
代码写错（语法 / 类型系统错误）
```

### 表现

```text
incompatible types
';' expected
method xxx cannot be applied
```

------

## 🎯 核心区别

| 类型      | 本质         | 是否改代码 |
| --------- | ------------ | ---------- |
| Maven问题 | classpath 错 | ❌ 不一定   |
| 编译问题  | 代码错误     | ✅ 必须     |

------

# 三、Cannot resolve / ClassNotFound 区别

## 1️⃣ Cannot resolve

```text
阶段：编译期
本质：编译器找不到类/方法
原因：classpath（Maven）问题
```

------

## 2️⃣ ClassNotFound

```text
阶段：运行期
本质：JVM 找不到 class
原因：运行环境缺依赖
```

------

## 🎯 对比

| 错误           | 阶段   | 本质               |
| -------------- | ------ | ------------------ |
| Cannot resolve | 编译期 | Maven classpath 错 |
| ClassNotFound  | 运行期 | 运行依赖缺失       |

------

# 四、classpath 本质（核心理解）

```text
classpath = 编译器/JVM 查找类的路径集合
```

------

## 来源

### 编译期

```text
Maven → 依赖 → 提供给编译器
```

### 运行期

```text
jar / 启动命令 → 提供给 JVM
```

------

# 五、IDEA vs 运行环境（高频坑）

## IDEA

```text
✔ 自动加载 ~/.m2 所有依赖
✔ 容错高
✔ 经常“看起来都能跑”
```

------

## 运行环境

```text
❗ 只认打包进去的依赖
❗ 少一个就 ClassNotFound
```

------

## 🎯 结论

```text
IDEA 能跑 ≠ 程序能跑
```

------

# 六、多模块 Maven（重点）

## 结构

```text
parent
 ├── module-a
 └── module-b（依赖 module-a）
```

------

# 七、两种依赖来源（核心）

## 1️⃣ 本地仓库（~/.m2）

```text
来源：mvn install
特点：存储 jar
```

------

## 2️⃣ Reactor（多模块构建）

```text
来源：父项目构建
特点：直接使用源码编译结果
```

------

# 八、为什么父项目可以编译成功？

## 因为使用了 Reactor

```text
mvn install（父项目）
```

Maven 会：

```text
1️⃣ 构建依赖关系图
2️⃣ 自动排序（先 module-a）
3️⃣ 使用 module-a 编译产物
```

------

## 本质

```text
所有模块在同一个构建上下文中
```

------

# 九、为什么单模块编译会失败？

```bash
cd module-b
mvn compile
```

------

## Maven 只会：

```text
去 ~/.m2 找 module-a
```

👉 ❌ 不会：

```text
去 ../module-a 找源码
```

------

# 十、关键区别总结

| 构建方式   | 是否使用源码      |
| ---------- | ----------------- |
| 父项目构建 | ✅ 使用（Reactor） |
| 单模块构建 | ❌ 不使用          |
| install 后 | ❌ 使用 .m2        |

------

# 十一、经典问题场景

## 场景 1️⃣ 修改 module-a 不生效

```text
改了 module-a
↓
没 install
↓
module-b 仍用旧版本（.m2）
```

------

## 场景 2️⃣ IDEA 能跑，Maven 不行

```text
IDEA：用源码 ✔
Maven：只认 .m2 ❌
```

------

## 场景 3️⃣ Cannot resolve

```text
缺依赖 / 版本错 / 未编译
```

------

# 十二、最佳实践（工程推荐）

## 1️⃣ 开发阶段

```bash
mvn -pl module-b -am compile
```

👉 只编译需要的模块 + 依赖模块

------

## 2️⃣ 修改公共模块

```bash
mvn install
```

------

## 3️⃣ 完整构建

```bash
mvn clean install
```

------

# 十三、排查问题 checklist（非常重要）

```text
1️⃣ 看错误类型（Cannot resolve / 编译错误）
2️⃣ 看 import 是否存在
3️⃣ mvn dependency:tree
4️⃣ mvn clean compile
5️⃣ 检查版本冲突
```

------

# 十四、终极总结（必须背）

```text
Maven问题：
→ 依赖没准备好（classpath 错）

编译问题：
→ 代码写错

运行问题：
→ JVM 没加载到类

多模块问题：
→ 是否在 Reactor 中
```

------

# 🎯 最后一条工程真理

```text
Maven 只认“坐标”
不会认“目录结构”
```

------