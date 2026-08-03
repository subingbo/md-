# RuoYi-Vue-Plus 启动报错排查笔记

## 问题描述

项目启动时报错：

```
ClassFormatError: Duplicate method name "convert"
in OperLogEventToSysOperLogBoMapperImpl
```

随后 Spring 上下文初始化失败：

```
CannotLoadBeanClassException: problem with class file or dependent class
```

------

## 根本原因

`target` 目录里残留了不同 JDK 版本编译的旧 class 文件，导致 MapStruct 自动生成的 `OperLogEventToSysOperLogBoMapperImpl` 字节码中出现了重复的 `convert` 方法，JVM 加载时直接报 `ClassFormatError`。

------

## 完整错误链

```
根本原因：多次切换 JDK，target/ 残留不同版本的 .class 文件
│
├── MapStruct 生成的 Impl 类出现重复 convert 方法
│   ├── ClassFormatError（JVM 加载时）
│   ├── CannotLoadBeanClassException（Spring 初始化时）
│   └── ClassNotFoundException（换 JDK 后 IDEA 缓存失效）
│
└── pom.xml 缺少 <mainClass>（独立问题）
    └── Unable to find main class（mvn install 打包时）
```

------

## 各错误说明

### 1. ClassFormatError: Duplicate method name

MapStruct 是自动生成代码的框架，读取 Mapper 接口后生成对应的 `Impl.class`。
 当 target 目录存在旧的脏文件，新编译的内容叠加上去，导致同一个类里出现两个相同签名的方法，违反 Java 字节码规范。

### 2. CannotLoadBeanClassException

Spring 启动时扫描所有 Bean，加载到问题类时因 `ClassFormatError` 崩溃，整个上下文初始化失败。

### 3. ClassNotFoundException（换 JDK 后）

切换 JDK 版本后，IDEA 的编译缓存是旧 JDK 产物，新 JDK 无法识别，连主类都找不到。
 需要 Clean Project + Rebuild 重建缓存。

### 4. Unable to find main class（mvn install 时）

`ruoyi-admin` 的 `pom.xml` 中 `spring-boot-maven-plugin` 没有配置 `<mainClass>`，打包时插件不知道入口类在哪。
 **注意**：IDEA 直接运行不走打包流程，所以平时不报错，只有执行 `mvn install` 时才暴露。

修复方法，在 `ruoyi-admin/pom.xml` 中添加：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <mainClass>org.dromara.DromaraApplication</mainClass>
    </configuration>
</plugin>
```

------

## 解决方法

单独对出问题的模块执行 clean + compile：

```bash
mvn clean -pl ruoyi-modules/ruoyi-system
mvn compile -pl ruoyi-modules/ruoyi-system -am
```

------

## 注意事项

### IDEA 直接运行 vs mvn install 的区别

| 方式            | 是否需要 `<mainClass>` | 走打包流程 |
| --------------- | ---------------------- | ---------- |
| IDEA 直接点运行 | ❌ 不需要               | ❌ 不走     |
| `mvn install`   | ✅ 必须有               | ✅ 走       |

### JDK 版本统一

RuoYi-Vue-Plus 5.6.0 官方推荐 **JDK 17**，以下三处需要保持一致：

- `File → Project Structure → Project → SDK`
- `Run → Edit Configurations → JRE`
- 终端 Maven 使用的 JDK（`mvn -version` 查看）

------

## 预防措施

每次切换 JDK 或拉取新代码后，执行：

```bash
mvn clean install -DskipTests
```

或在 IDEA 中：`Build → Clean Project` → `Build → Rebuild Project`

保证编译产物始终干净，避免残留旧字节码引发奇怪错误。