# Redis 配置指南（Spring Boot 3.x + Nacos）

本文档说明本项目 test / K8s 环境下 Redis 的正确配置方式，以及 `spring.data.redis.ssl` 导致服务启动失败的常见原因与修复方法。

适用版本：

- Spring Boot **3.5.x**（日志中可见 `Spring Boot Version: 3.5.9`）
- Spring Cloud Alibaba Nacos Config
- 客户端：**Redisson**（通过 `spring.data.redis.*` 自动配置）

---

## 1. 问题现象

多个微服务（如 `gjy-infra-server`、`gjy-analysis-service`）启动失败，日志中出现：

```text
APPLICATION FAILED TO START

Description:
Failed to bind properties under 'spring.data.redis.ssl' to org.springframework.boot.autoconfigure.data.redis.RedisProperties$Ssl:
    Reason: org.springframework.core.convert.ConverterNotFoundException: No converter found capable of converting from type [java.lang.Boolean] to type [org.springframework.boot.autoconfigure.data.redis.RedisProperties$Ssl]

Action:
Update your application's configuration
```

伴随出现的次要错误（**可忽略，属于连锁反应**）：

- `Error creating bean with name 'redisController': Injection of resource dependencies failed`
- `Error creating bean with name 'testDemoController': Injection of resource dependencies failed`
- `BeanCreationNotAllowedException: Error creating bean with name 'taskScheduler'`

这些是因为 Redis 相关 Bean 未能创建，Spring 容器回滚时的副作用，**不是根因**。

---

## 2. 根因说明

### 2.1 Spring Boot 3.x 配置格式变更

从 **Spring Boot 3.0** 起，`spring.data.redis.ssl` 不再是布尔值，而是一个嵌套对象 `RedisProperties.Ssl`，主要字段：

| 属性 | 类型 | 说明 |
|------|------|------|
| `spring.data.redis.ssl.enabled` | boolean | 是否启用 SSL |
| `spring.data.redis.ssl.bundle` | string | 可选，引用 `spring.ssl.bundle` 中定义的证书包 |

因此以下写法在 Boot 3.x 中**一律非法**，无论值是 `true` 还是 `false`：

```yaml
spring:
  data:
    redis:
      ssl: true    # ❌
      ssl: false   # ❌ 同样会报错
```

### 2.2 本次实际错误配置

在 Nacos `common.yaml` 中配置了：

```yaml
host: cspon-redis
port: 6379
password: admin@123
database: 0
ssl: false        # ❌ 触发启动失败
```

应用**尚未尝试连接 Redis**，在**配置绑定阶段**就已失败。

### 2.3 是否需要改代码？

**不需要。** 这是纯配置问题，修改 Nacos（或对应 yaml）即可，无需改动 Java 代码或 Redisson 相关类。

---

## 3. 配置加载路径

本项目各服务通过 Nacos 加载公共 Redis 配置，典型结构如下。

### 3.1 芋道模块（如 infra-server）

`yudao-module-infra-server/src/main/resources/application.yaml`：

```yaml
spring:
  config:
    import:
      - optional:classpath:application-${spring.profiles.active}.yaml
      - optional:nacos:common.yaml?group=DEFAULT_GROUP
      - optional:nacos:${spring.application.name}-${spring.profiles.active}.yaml
```

### 3.2 gzmap 业务服务（如 analysis-service）

`gzmap-parent/*/src/main/resources/application.yml`：

```yaml
spring:
  config:
    import:
      - optional:classpath:application-gzmap-main-system-gateway.yaml
      - optional:nacos:common.yaml?group=DEFAULT_GROUP
      - optional:nacos:${spring.application.name}-${spring.profiles.active}.yaml
```

### 3.3 配置优先级（简化理解）

1. 本地 `application.yml` / `application-test.yaml` 等
2. Nacos `common.yaml`（**Redis 通常配在这里**）
3. Nacos `{服务名}-{profile}.yaml`（如 `gjy-infra-server-test.yaml`）

test 环境日志示例：

```text
[Nacos Config] Load config[dataId=common.yaml, group=DEFAULT_GROUP] success
The following 1 profile is active: "test"
```

**结论：test 环境改 Redis，应优先改 Nacos 控制台中的 `common.yaml`，改本地文件不一定生效。**

---

## 4. 推荐配置模板

### 4.1 内网 Redis，不使用 SSL（当前环境推荐）

适用于：`cspon-redis:6379`、K8s 集群内 Service、无 TLS 的普通 Redis。

在 Nacos `common.yaml` 中配置：

```yaml
spring:
  data:
    redis:
      host: cspon-redis
      port: 6379
      password: 'admin@123'   # 含 @ 等特殊字符务必加引号
      database: 0
      timeout: 10s
      # 不要写 ssl: false，直接省略 ssl 节点即可
```

**要点：**

- 删除 `ssl: false` 或 `ssl: true`
- 不启用 SSL 时，**不写 `ssl` 段**即为默认关闭
- 密码含 `@`、`:`、`#` 等字符时，使用单引号包裹，避免 YAML 解析异常

### 4.2 需要 SSL 的云 Redis

适用于：阿里云 / 腾讯云等托管 Redis 的 SSL 端口（常见 6380）。

```yaml
spring:
  data:
    redis:
      host: your-redis.example.com
      port: 6380
      password: 'your-password'
      database: 0
      ssl:
        enabled: true
```

### 4.3 需要自定义 CA 证书（自签名 / 私有 CA）

```yaml
spring:
  ssl:
    bundle:
      pem:
        redis-bundle:
          truststore:
            certificate: classpath:redis-ca.crt   # 或 file:/path/to/ca.crt
  data:
    redis:
      host: your-redis.example.com
      port: 6380
      password: 'your-password'
      database: 0
      ssl:
        enabled: true
        bundle: redis-bundle
```

证书文件需放入各服务可访问的路径（如 jar 内 `classpath:` 或挂载卷 `file:`）。

---

## 5. 完整 Nacos common.yaml 示例（可直接参考）

以下为 test 环境常见片段，按实际中间件地址调整：

```yaml
spring:
  datasource:
  # ... 数据库配置 ...

  data:
    redis:
      host: cspon-redis
      port: 6379
      password: 'admin@123'
      database: 0
      timeout: 10s
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0

# 若使用 Redisson 高级参数，一般无需单独配置；
# 本项目通过 redisson-spring-boot-starter 读取 spring.data.redis.*
```

**不要出现：**

```yaml
ssl: false
ssl: true
```

---

## 6. 配置写法对照表

| 场景 | Spring Boot 2.x（旧） | Spring Boot 3.x（本项目） |
|------|----------------------|---------------------------|
| 关闭 SSL | `ssl: false` 或省略 | **省略 `ssl` 节点** |
| 开启 SSL | `ssl: true` | `ssl: { enabled: true }` |
| 指定证书 | 需自定义代码或 Redisson 配置 | `ssl.enabled` + `ssl.bundle` + `spring.ssl.bundle` |

---

## 7. 排错步骤

按顺序执行：

### 步骤 1：修改 Nacos

1. 登录 Nacos 控制台
2. 配置管理 → 配置列表 → `DEFAULT_GROUP` → `common.yaml`
3. 找到 `spring.data.redis` 段
4. **删除** `ssl: false` / `ssl: true`
5. 确认 `password` 已加引号（若含特殊字符）
6. 发布配置

### 步骤 2：重启受影响服务

至少包括日志中报错的服务，例如：

- `gjy-infra-server`
- `gjy-analysis-service`
- 其他引用 `common.yaml` 的微服务

K8s 环境示例：

```bash
kubectl rollout restart deployment/gjy-infra-server -n <namespace>
kubectl rollout restart deployment/gjy-analysis-service -n <namespace>
```

### 步骤 3：验证启动日志

成功标志：

- 不再出现 `Failed to bind properties under 'spring.data.redis.ssl'`
- 出现 Tomcat started / Application started 类日志

### 步骤 4：若仍失败，区分错误类型

| 错误关键词 | 含义 | 处理方向 |
|-----------|------|----------|
| `Failed to bind properties ... ssl` | 配置格式仍错误 | 检查是否还有 `ssl: true/false` |
| `Unable to connect to Redis` | 已解析配置，但连不上 | 检查 host/port、网络、密码、防火墙 |
| `SSL` / `certificate` / `handshake` | SSL 握手失败 | 确认端口是否为 SSL 端口，是否需 CA 证书 |
| `NOAUTH` | 密码错误 | 核对 Redis 密码 |
| `Connection refused` | 地址或端口不可达 | 检查 K8s Service 名、DNS、Redis Pod 状态 |

---

## 8. 常见问题 FAQ

### Q1：我配了 `ssl: false`，为什么还不行？

因为 Boot 3.x 不接受布尔值。`false` 和 `true` 都会触发同样的绑定异常。**删掉该行即可。**

### Q2：内网 `cspon-redis:6379` 需要开 SSL 吗？

一般**不需要**。集群内普通 Redis 通常走明文 6379。只有云厂商明确要求 TLS 或端口为 6380 时才启用 `ssl.enabled: true`。

### Q3：改了本地 `application-test.yaml` 为什么没效果？

test 环境主要从 Nacos `common.yaml` 加载 Redis。请改 Nacos，或确认本地文件未被 Nacos 覆盖。

### Q4：需要改 Redisson 或 Java 配置类吗？

通常**不需要**。`yudao-spring-boot-starter-redis` 使用 Redisson，自动读取 `spring.data.redis.*`。格式正确后即可工作。

### Q5：`gjy-infra-server-test.yaml` 显示 empty 有影响吗？

日志中：

```text
[Nacos Config] config[dataId=gjy-infra-server-test.yaml, group=DEFAULT_GROUP] is empty
```

表示该服务专属配置为空，**不影响**，公共配置已从 `common.yaml` 加载成功。

### Q6：密码 `admin@123` 为什么要加引号？

YAML 中 `@` 等字符可能被误解析。建议始终对 Redis 密码使用引号：

```yaml
password: 'admin@123'
```

---

## 9. 与本项目相关的文件索引

| 文件 | 说明 |
|------|------|
| Nacos `common.yaml` | **Redis 主配置位置（test 环境）** |
| `yudao-module-infra-server/.../application.yaml` | 声明从 Nacos 导入 common |
| `gzmap-parent/*/.../application.yml` | 业务服务 Nacos 导入声明 |
| `docs/报错/logs` | 本次启动失败原始日志 |
| `yudao-framework/yudao-spring-boot-starter-redis/` | Redis / Redisson 自动配置（一般无需改） |

---

## 10. 快速修复清单（Checklist）

- [ ] Nacos `common.yaml` 中删除 `ssl: false` / `ssl: true`
- [ ] 确认 `host: cspon-redis`、`port: 6379`
- [ ] 密码使用引号：`password: 'admin@123'`
- [ ] 发布 Nacos 配置
- [ ] 重启相关 Deployment / Pod
- [ ] 检查启动日志无 `RedisProperties$Ssl` 绑定错误
- [ ] 若仍有连接错误，再排查网络与 Redis 服务本身

---

## 11. 参考链接

- [Spring Boot 3.x — Working with NoSQL (Redis SSL)](https://docs.spring.io/spring-boot/reference/data/nosql.html#data.nosql.redis)
- [Spring Boot 3.0 Migration Guide — Redis properties](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide#redis-properties)
- [RedisProperties.Ssl API](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/data/redis/RedisProperties.Ssl.html)

---

**文档版本：** 2026-06-12  
**适用环境：** test / K8s（`cspon-redis`、`cspon-config`）  
**维护说明：** 若升级 Spring Boot 大版本，请重新核对 `spring.data.redis.ssl` 相关属性是否有变更。