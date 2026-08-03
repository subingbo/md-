# Maven 多模块与依赖管理面试笔记

## 1. Maven 核心作用

Maven 是 Java 项目构建工具，主要负责：

- 项目依赖管理
- 项目编译
- 测试执行
- 打包发布
- 多模块项目管理

核心配置文件：

```text
pom.xml
```

Maven 根据 POM 文件描述：

- 项目是什么
- 需要哪些依赖
- 如何构建
- 使用哪些插件

------

# 2. Maven 构建流程

执行：

```bash
mvn clean package
```

大致流程：

```
读取 pom.xml
      |
      ↓
解析项目模型
      |
      ↓
解析依赖
      |
      ↓
下载依赖
      |
      ↓
编译源码
      |
      ↓
执行测试
      |
      ↓
打包
```

------

# 3. Maven 依赖下载流程

例如：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.5.9</version>
</dependency>
```

Maven 不会直接下载 jar。

流程：

```
项目声明依赖

        ↓

寻找依赖 POM

        ↓

读取 POM 中的 dependencies

        ↓

解析传递依赖

        ↓

下载所有 jar
```

例如：

```
spring-boot-starter-web

        |
        +---- spring-web
        |
        +---- spring-core
        |
        +---- jackson
        |
        +---- tomcat
```

Maven 会递归解析依赖树。

------

# 4. Maven 仓库机制

Maven 仓库分为：

## 4.1 本地仓库

默认：

```
~/.m2/repository
```

你的配置：

```
localRepository
```

表示下载后的 jar 保存位置。

------

## 4.2 远程仓库

默认：

```
Maven Central
```

地址：

```
https://repo.maven.apache.org/maven2
```

------

## 4.3 镜像仓库

例如阿里云：

```xml
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>central</mirrorOf>
    <url>
        https://maven.aliyun.com/repository/public
    </url>
</mirror>
```

含义：

```
访问 central

        ↓

改为访问阿里云
```

注意：

不是替换所有仓库。

例如：

```
central
    ↓
aliyun


spring-snapshots
    ↓
spring官方仓库
```

------

# 5. Maven 从哪里知道仓库地址？

来源：

优先级：

```
项目 pom.xml

↓

父 pom.xml

↓

settings.xml

↓

默认中央仓库
```

常见配置位置：

## settings.xml

负责：

- 镜像
- 私服
- 代理
- 本地仓库位置

## pom.xml

负责：

- 项目依赖
- 项目构建

------

# 6. Maven parent 继承

子模块：

```xml
<parent>
    <groupId>com.demo</groupId>
    <artifactId>parent</artifactId>
    <version>1.0</version>
</parent>
```

表示：

子模块继承父 POM。

方向：

```
子模块
   |
   | 获取配置
   ↑
父模块
```

类似 Java：

```java
class Child extends Parent
```

------

## 父 POM 可以提供什么？

例如：

```xml
<properties>
    <java.version>21</java.version>
</properties>
```

子模块自动拥有。

可以继承：

- properties
- dependencyManagement
- dependencies
- pluginManagement
- build 配置

------

# 7. Maven 聚合

父模块：

```xml
<modules>

    <module>analysis-service</module>

    <module>community-service</module>

</modules>
```

表示：

父模块管理多个子模块。

方向：

```
父模块
   |
   | 管理
   ↓
子模块
```

作用：

- 一键构建多个模块
- 控制模块构建顺序

例如：

目录：

```
gzmap-parent

├── analysis-service

├── community-service

└── detail-service
```

执行：

```bash
mvn package
```

父模块会按照 modules 构建所有子模块。

------

# 8. 聚合和继承区别

|              | 继承         | 聚合     |
| ------------ | ------------ | -------- |
| 方向         | 子 → 父      | 父 → 子  |
| 配置位置     | parent       | modules  |
| 作用         | 获取配置     | 管理模块 |
| 类似         | Java extends | 项目组织 |
| 是否决定构建 | 否           | 是       |

实际项目通常两个一起使用：

```
              parent继承

analysis-service ------------> gzmap-parent


              modules聚合

gzmap-parent
       |
       ↓
analysis-service
```

------

# 9. dependencies 和 dependencyManagement

这是面试重点。

## 9.1 dependencies

表示：

> 当前模块真正需要的依赖

例如：

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

</dependencies>
```

效果：

- 会下载
- 会加入依赖树
- 子模块继承后也会拥有

------

## 9.2 dependencyManagement

表示：

> 统一管理依赖版本

例如：

```xml
<dependencyManagement>

    <dependencies>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
        </dependency>

    </dependencies>

</dependencyManagement>
```

效果：

不会下载。

它只是告诉 Maven：

如果有人使用：

```xml
<dependency>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

默认使用：

```
8.4.0
```

------

# 10. BOM 是什么？

BOM：

```
Bill Of Materials
```

中文：

依赖版本清单。

本质：

```
一个 packaging=pom 的 Maven 文件
```

典型：

```xml
<dependencyManagement>

    <dependencies>

        <dependency>
            spring-core
            version=6.x
        </dependency>

    </dependencies>

</dependencyManagement>
```

作用：

统一管理一组依赖版本。

例如 Spring Boot：

```
Spring Boot BOM

spring-core 6.x

jackson 2.x

tomcat 10.x
```

保证：

```
spring-core
spring-context
jackson

版本互相兼容
```

------

# 11. import BOM

例如：

```xml
<dependencyManagement>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

    </dependencies>

</dependencyManagement>
```

作用：

只导入：

```
dependencyManagement
```

不会导入：

```
dependencies
```

所以：

BOM 不负责下载业务依赖。

它只提供：

```
版本信息
```

------

# 12. dependencies 和 BOM 区别

## 普通 dependencies

```
我要使用这个依赖
```

会下载。

## BOM dependencyManagement

```
如果有人使用这个依赖
版本按照我的
```

不会下载。

------

# 13. Maven 多模块构建流程

例如：

```
gzmap-parent

(packaging=pom)

        |
        | modules

        ↓

analysis-service

        |
        ↓

analysis-service-biz

(packaging=jar)
```

执行：

```bash
mvn package
```

流程：

```
父模块

↓

子模块

↓

jar模块

↓

解析dependencies

↓

下载依赖

↓

编译

↓

生成jar
```

------

# 14. packaging 区别

## pom

生成：

```
xxx.pom
```

作用：

- 父工程
- 聚合工程
- BOM

不会生成：

```
jar
```

------

## jar

生成：

```
xxx.jar
```

常见：

Spring Boot 服务。

------

## war

生成：

```
xxx.war
```

部署到 Web 容器。

------

# 15. 面试回答模板

面试官：

> Maven 多模块项目是怎么管理的？

回答：

```
Maven 多模块主要通过 parent 和 modules 实现。

parent 用于继承配置，子模块可以继承父 POM 中的 properties、
dependencyManagement 和插件配置。

modules 用于聚合管理，父工程通过 modules 指定有哪些子模块，
执行 Maven 命令时可以一次构建整个项目。

依赖版本通常放在 dependencyManagement 或 BOM 中统一管理，
业务模块通过 dependencies 声明真正使用的依赖。

dependencyManagement 只负责版本管理，不会主动引入依赖，
只有 dependencies 中声明后才会真正参与构建。
```

------

# 16. 一句话总结

```
parent：子模块向上继承配置

modules：父模块向下管理模块

dependencies：真正使用依赖

dependencyManagement：管理依赖版本

BOM：依赖版本清单

Maven下载：
先找依赖POM → 解析依赖树 → 下载jar
```