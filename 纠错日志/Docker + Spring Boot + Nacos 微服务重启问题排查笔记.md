# Docker + Spring Boot + Nacos 微服务重启问题排查笔记

# 一、问题现象

docker compose 启动后：

```bash
docker compose ps
```

发现部分服务持续重启：

- trans-service
- plan-service
- construct-service
- detail-service
- stat-service
- gzmap-infra-service

表现为：

```text
Up 8 seconds
Up 10 seconds
Up 12 seconds
```

时间不断重新计数。

说明：

- 容器并未正常运行
- restart: always 正在反复拉起容器
- Spring Boot 启动阶段发生异常退出

------

# 二、第一阶段误判

最开始怀疑：

- Docker 内存不足
- Docker Compose 配置错误
- 服务启动顺序问题
- Nacos 无法连接
- 数据库问题
- jar 版本问题

但后续逐步排除。

------

# 三、关键日志定位

执行：

```bash
docker logs --tail 200 plan-service
```

最终定位到核心错误：

```text
APPLICATION FAILED TO START

The bean 'mybatisPlusInterceptor'
could not be registered.

A bean with that name has already been defined
```

具体冲突来源：

## Bean 来源1

```text
cn.iocoder.yudao.framework.mybatis.config.YudaoMybatisAutoConfiguration
```

## Bean 来源2

```text
com.guodi.plan.biz.config.MyBatisPlusConfig
```

------

# 四、真正根因

项目已经使用：

```text
YudaoMybatisAutoConfiguration
```

自动注册：

```java
mybatisPlusInterceptor
```

但业务模块：

```java
com.guodi.plan.biz.config.MyBatisPlusConfig
```

又手动注册了一次：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor()
```

导致：

```text
BeanDefinitionOverrideException
```

最终 Spring Boot 启动失败。

------

# 五、为什么以前没问题

## 原因1：以前 Nacos 实际没连成功

日志：

```text
grpc client connection server:127.0.0.1
Connection refused: /127.0.0.1:9848
```

说明：

服务实际上一直在连接：

```text
127.0.0.1:8848
```

而不是：

```text
nacos:8848
```

Docker 容器内：

```text
127.0.0.1
```

代表容器自己。

不是 Nacos。

------

## 原因2：使用了 optional:nacos

配置：

```yaml
spring:
  config:
    import:
      - optional:nacos:common.yaml
```

意味着：

- 即使 Nacos 完全连接失败
- Spring Boot 也不会退出

所以：

以前很多服务实际上：

```text
没真正连上 Nacos
```

但：

```text
仍然能启动
```

因为本地 application.yml 已经足够。

------

## 原因3：最近修复了 Nacos 地址

后来修改：

```yaml
spring:
  cloud:
    nacos:
      server-addr: nacos:8848
```

之后：

Nacos 真正连接成功。

于是：

```text
common.yaml
```

开始真正生效。

------

## 原因4：远程 common.yaml 中开启了：

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
```

而：

之前由于 Nacos 没连上。

这个配置从未真正生效。

------

## 原因5：修复 Nacos 后

Spring 开始真正加载：

```text
common.yaml
```

于是：

之前隐藏的 Bean 冲突问题暴露出来。

------

# 六、为什么 profile=test 明明生效，却仍然读取错误配置

日志：

```text
The following 1 profile is active: "test"
```

说明：

```text
profile=test
```

确实生效。

但：

## profile 生效 ≠ 配置一定按预期覆盖成功

------

# 七、Spring Boot 配置加载机制核心原理

## 1. application.yml 先加载

例如：

```yaml
spring:
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848
```

------

## 2. Spring 发现：

```yaml
spring.config.import:
  - nacos:common.yaml
```

于是：

开始初始化：

```text
Nacos ConfigDataLoader
```

------

## 3. 初始化 Nacos Client 时

它只能使用：

```text
当前 Environment 已存在配置
```

此时：

只有：

```text
application.yml
```

已完成加载。

------

## 4. application-test.yml 尚未完成 merge

因此：

Nacos Client 已经按：

```text
127.0.0.1
```

初始化完成。

------

## 5. 后续即使：

```yaml
application-test.yml
```

覆盖：

```yaml
server-addr: nacos:8848
```

也已经晚了。

因为：

```text
Nacos Client 已经创建完成
```

不会重新初始化。

------

# 八、为什么去掉 profile 后问题反而减少

原配置链：

```text
application.yml
    ↓
application-test.yml
    ↓
Nacos common.yaml
    ↓
plan-service-test.yaml
```

存在：

- profile
- import
- namespace
- local/test/dev
- Docker env
- JVM 参数

多层叠加。

------

最终：

实际生效配置难以判断。

------

后来：

直接统一：

```yaml
application.yml
```

作为 Docker 唯一配置。

问题明显减少。

因为：

配置链被大幅简化。

------

# 九、最终建议架构

Docker 环境下：

不要让：

- Nacos 地址
- Redis 地址
- 数据库地址

依赖 profile 动态覆盖。

------

建议：

直接在：

```yaml
application.yml
```

固定：

```yaml
spring:
  cloud:
    nacos:
      server-addr: nacos:8848

  datasource:
    url: jdbc:kingbase8://kingbase:54321/test

  data:
    redis:
      host: redis
```

------

# 十、最终修复方案

## 方案1（推荐）

删除业务模块：

```java
MyBatisPlusConfig
```

中的：

```java
@Bean
MybatisPlusInterceptor
```

避免重复定义。

------

## 方案2（临时恢复）

开启：

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
```

允许 Bean 覆盖。

------

# 十一、最终结论

本次问题并不是：

- Docker 重启太多
- 内存不足
- Compose 配置错误

真正根因是：

## 1. Nacos 实际长期未连接成功

## 2. 修复 Nacos 后，远程配置开始真正生效

## 3. 之前隐藏的 MyBatis Bean 冲突被暴露

## 4. Spring Boot 启动失败导致容器不断重启

------

# 十二、核心经验总结

## 1. Docker 内 127.0.0.1 是容器自己

不是宿主机。

------

## 2. optional:nacos 不会阻止服务启动

即使 Nacos 完全不可用。

------

## 3. profile 生效 ≠ 配置一定正确覆盖

尤其涉及：

- ConfigData
- bootstrap
- nacos
- import

时。

------

## 4. 基础设施地址不要依赖 profile 动态覆盖

尤其：

- nacos
- redis
- mysql

在 Docker/K8s 中建议固定。

------

## 5. Spring Boot 配置是“分阶段初始化”

不是：

```text
一次性读取全部配置再统一覆盖
```

很多基础设施配置：

会在 profile merge 前就被消费。

------

# 十三、最终推荐部署方式

Docker 环境：

```text
application.yml = 最终运行配置
```

不要再：

- local/dev/test 多层覆盖
- JVM profile 动态切换
- namespace 多套并存
- import 多层嵌套