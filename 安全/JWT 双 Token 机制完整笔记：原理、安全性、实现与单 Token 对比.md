# JWT 双 Token 机制完整笔记：原理、安全性、实现与单 Token 对比

## 一、核心结论

**双 Token 机制并不是通过增加 Token 数量来提高安全性，而是通过区分短期访问凭证和长期刷新凭证，减少长期凭证的暴露机会，并实现更灵活的会话管理。**

最重要的三个结论：

1. 如果攻击者拿到了有效的 Refresh Token，通常可以刷新出 Access Token，进而冒充用户。因此，Refresh Token 被盗和长期 Access Token 被盗，都属于严重的安全问题。
2. 双 Token 的主要优势是让长期凭证不参与高频业务请求，使日常请求中泄露的 Access Token 只能在较短时间内被利用。
3. **如果两个 Token 的存储方式、暴露范围和安全保护完全相同，那么双 Token 相比单个长期 Token，安全优势确实有限。**

双 Token 是一种安全性、性能和用户体验之间的权衡，不是所有系统都必须使用。

## 二、什么是双 Token？

双 Token（Dual Token）是指身份认证系统同时使用两种凭证：

- Access Token：访问令牌。
- Refresh Token：刷新令牌。

### 2.1 Access Token

Access Token 用于访问需要登录权限的业务接口。

例如：

```
GET /api/user/info
Authorization: Bearer eyJhbGciOi...
```

服务端验证 Access Token 后，确认用户身份和访问权限。

主要特点：

- 有效期较短，例如 15～30 分钟。
- 高频使用，每次需要认证的业务请求通常都携带。
- 可以使用 JWT 格式，实现无需查询数据库的签名验证。
- 一旦被盗，攻击者可以在 Token 有效期内冒充用户。

### 2.2 Refresh Token

Refresh Token 用于在 Access Token 过期后，申请新的 Access Token。

例如：

```
POST /auth/refresh
Cookie: refresh_token=abc123...
```

服务端验证 Refresh Token 有效后，签发新的 Access Token。

主要特点：

- 有效期较长，例如 7 天。
- 使用频率低，通常只有刷新时才使用。
- 不应该用于直接访问普通业务接口。
- 可以配合 Redis 实现撤销、轮换、异常检测。
- 如果被盗且没有其他保护措施，攻击者也可以利用它获得业务访问权限。

### 2.3 对比

| 特征               | Access Token       | Refresh Token        |
| ------------------ | ------------------ | -------------------- |
| 主要作用           | 访问业务接口       | 获取新 Access Token  |
| 示例有效期         | 30 分钟            | 7 天                 |
| 请求频率           | 高                 | 低                   |
| 能否访问业务接口   | 可以               | 不应该直接允许       |
| 常见格式           | JWT                | 随机字符串或 JWT     |
| 是否需要服务器存储 | 无状态 JWT 不需要  | 推荐保存会话状态     |
| 主要安全措施       | 短有效期、签名校验 | 安全存储、撤销、轮换 |

注意：JWT 只是 Token 的一种数据格式，双 Token 是一种认证机制。

双 Token 不要求两个 Token 都是 JWT。

## 三、双 Token 的完整工作流程

假设：

- Access Token 有效期为 30 分钟。
- Refresh Token 有效期为 7 天。

### 3.1 第一步：用户登录

前端发送：

```
POST /auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "123456"
}
```

后端验证账号密码成功后，生成两个 Token。

一种推荐的 Web 实现是：

```
HTTP/1.1 200 OK
Set-Cookie: refresh_token=R1; HttpOnly; Secure; SameSite=Lax; Path=/auth

{
  "accessToken": "A1",
  "expiresIn": 1800
}
```

其中：

- A1：有效期 30 分钟。
- R1：有效期 7 天。
- Access Token 返回给前端，由前端保存在内存。
- Refresh Token 通过 HttpOnly Cookie 保存。

Cookie 的 SameSite、Path 等具体设置需要结合前后端部署域名及 CSRF 防护方案确定。

### 3.2 第二步：正常访问业务接口

前端请求：

```
GET /api/user/info
Authorization: Bearer A1
```

后端处理流程：

```
接收业务请求
    ↓
提取 Access Token
    ↓
验证签名、有效期、令牌类型
    ↓
验证用户权限
    ↓
执行业务
    ↓
返回响应
```

这里的关键是：

**普通业务接口使用的是 A1，而不是长期有效的 R1。**

### 3.3 第三步：Access Token 过期

30 分钟后：

```
GET /api/user/info
Authorization: Bearer A1
```

由于 A1 已经过期，服务端返回：

```
HTTP/1.1 401 Unauthorized
```

前端检测到认证失效后，尝试调用刷新接口。

注意：401 不一定表示 Token 过期，也可能是 Token 无效、登录状态失效等情况。实际实现应区分错误类型，避免无限刷新。

### 3.4 第四步：刷新 Access Token

前端调用：

```
POST /auth/refresh
Cookie: refresh_token=R1
```

如果采用 Cookie 保存 Refresh Token，浏览器会根据 Cookie 策略自动携带它，不需要 JavaScript 读取其值。

后端执行：

```
接收 Refresh Token
    ↓
验证 Token 是否存在、有效
    ↓
查询 Redis 中的会话状态
    ↓
检查是否过期、撤销
    ↓
签发新的 Access Token A2
    ↓
可选：轮换 Refresh Token
    ↓
返回结果
```

获得新的 A2 后，前端重试刚才失败的业务请求：

```
GET /api/user/info
Authorization: Bearer A2
```

用户通常不需要重新输入账号密码。

### 3.5 第五步：Refresh Token 失效

如果 R1 已经过期或者被撤销：

```
Access Token 过期
    ↓
尝试使用 Refresh Token
    ↓
Refresh Token 无效
    ↓
刷新失败
    ↓
前端清除登录状态
    ↓
跳转登录页面
```

这时候用户必须重新登录。

## 四、为什么不用一个有效期 7 天的 Access Token？

这是理解双 Token 的核心问题。

假设有两种设计。

### 4.1 方案 A：单个长期 Access Token

用户登录后获得：

```
Access Token：7 天
```

之后所有业务请求都携带它：

```
请求 1 → Access Token（7 天）
请求 2 → Access Token（7 天）
请求 3 → Access Token（7 天）
...
请求 1000 → Access Token（7 天）
```

如果攻击者窃取了这个 Token，就可以在剩余有效期内直接访问业务接口。

假设刚签发就被盗，理论上可能被利用接近 7 天。

### 4.2 方案 B：双 Token

用户登录后获得：

```
Access Token：30 分钟
Refresh Token：7 天
```

普通业务请求：

```
请求 1 → Access Token（30 分钟）
请求 2 → Access Token（30 分钟）
请求 3 → Access Token（30 分钟）
...
请求 1000 → Access Token（30 分钟）
```

刷新请求：

```
Refresh Token（7 天）
       ↓
POST /auth/refresh
       ↓
新的 Access Token（30 分钟）
```

如果攻击者只窃取了一次普通请求中的 Access Token，就只能使用到这个 Access Token 过期。

关键区别在于：

**单 Token 让长期凭证直接参与日常请求；双 Token 让日常请求只接触短期凭证。**

当然，请求次数多不意味着 Token 一定会泄露。HTTPS 正常工作时，Token 不会因为多发送几次就被网络旁观者直接获取。这里减少的是长期凭证进入不同请求处理环节的机会和暴露范围。

### 4.3 不同攻击场景对比

| 攻击场景               | 单 Token（7 天）            | 双 Token（30 分钟 + 7 天）       |
| ---------------------- | --------------------------- | -------------------------------- |
| 普通请求中的凭证泄露   | 最长可能利用 7 天           | 通常只到 Access Token 过期       |
| 长期凭证被窃取         | 可以直接访问接口            | 可以通过刷新接口获取访问权限     |
| 用户退出登录           | 无状态 JWT 需要额外撤销措施 | 可撤销 Refresh Token             |
| 已签发 JWT 的立即撤销  | 需要额外机制                | 短期 Access Token 也需要额外机制 |
| 凭证被恶意程序全部窃取 | 高风险                      | 同样高风险                       |
| 高频业务接口验证       | 可以仅验证 JWT              | 同样可以仅验证 JWT               |

结论：

**双 Token 缩短的是短期凭证泄露后的攻击窗口，并不能保证长期 Refresh Token 被盗后仍然安全。**

## 五、如果 Refresh Token 被盗，双 Token 还有什么意义？

### 5.1 攻击者确实可以自己刷新

假设攻击者盗取 R1：

```
攻击者获取 R1
    ↓
POST /auth/refresh
    ↓
服务器认为 R1 有效
    ↓
返回 Access Token A2
    ↓
攻击者使用 A2 访问业务接口
```

如果刷新接口只验证 Refresh Token 本身，而不使用其他约束，那么服务器通常无法判断 R1 是原用户还是攻击者提交的。

因此：

**Refresh Token 本质上仍然是一种登录凭证。**

不能认为 Refresh Token 因为不能直接访问业务接口就天然安全。

### 5.2 为什么仍然使用 Refresh Token？

核心原因有三个。

**原因一：减少长期凭证参与的接口范围**

Access Token 需要参与各种业务接口的认证。

Refresh Token 理想情况下只发送到专门的认证刷新接口。

这让长期凭证可以采用更严格的存储、传输和校验规则。

**原因二：让短期 JWT 和长期会话独立管理**

Access Token 可以使用 JWT 本地验证，提高高频请求的处理效率。

Refresh Token 可以采用服务端存储的随机令牌，方便撤销、轮换和会话管理。

**原因三：实现长期登录和短期授权的分离**

Refresh Token 决定用户是否能够持续取得新的访问凭证。

Access Token 决定当前请求是否拥有有效的访问凭证。

这样既不需要用户每 30 分钟登录一次，也不需要把一个 7 天有效的 JWT 交给每个业务接口。

## 六、Refresh Token Rotation：刷新令牌轮换

单纯使用长期 Refresh Token 依然有较大风险，因此实际系统经常配合 Token Rotation。

### 6.1 基本原理

每次刷新时：

1. 验证当前 Refresh Token。
2. 将旧 Refresh Token 标记为失效。
3. 生成新的 Refresh Token。
4. 生成新的 Access Token。
5. 将两个新 Token 返回客户端。

例如：

```
登录：
    Access Token A1
    Refresh Token R1

第一次刷新：
    R1 → A2 + R2
    R1 立即失效

第二次刷新：
    R2 → A3 + R3
    R2 立即失效

第三次刷新：
    R3 → A4 + R4
    R3 立即失效
```

同一个 Refresh Token 原则上只能成功刷新一次。

### 6.2 为什么轮换有用？

假设：

```
用户持有：R1
攻击者偷到：R1
```

用户先刷新：

```
用户：R1 → R2
R1 作废
```

攻击者之后使用 R1：

```
攻击者：R1 → 刷新失败
```

服务器发现一个已经使用过的 Refresh Token 被再次提交，就可能判断存在重放攻击。

如果实施 Token Family（令牌家族）管理，可以进一步撤销整组关联会话，强制重新登录。

### 6.3 Rotation 也有局限

如果攻击者先使用 R1：

```
攻击者：R1 → R2
用户：R1 → 刷新失败
```

攻击者仍然可能先取得新 Token。

因此，Rotation 不能阻止第一次盗用成功，它主要用于限制重复使用并帮助发现异常。

实际实现还需要注意并发：

- 两个刷新请求可能同时到达。
- 旧 Token 失效与新 Token 创建必须保证原子性。
- 可以通过 Redis Lua 脚本或数据库事务实现。
- 需要合理处理正常并发刷新，避免错误地将其视为攻击。

## 七、JWT + Redis 如何实现双 Token？

一种比较实用的方案是：

**Access Token 使用 JWT，Refresh Token 使用随机字符串，Redis 保存 Refresh Token 对应的会话状态。**

### 7.1 Access Token

JWT Payload 示例：

```
{
  "sub": "1001",
  "type": "access",
  "iss": "my-backend",
  "aud": "my-api",
  "iat": 1792000000,
  "exp": 1792001800,
  "jti": "access-uuid-001"
}
```

重要字段：

| 字段 | 含义           |
| ---- | -------------- |
| sub  | 用户 ID        |
| type | Token 类型     |
| iss  | 签发者         |
| aud  | 目标接收方     |
| iat  | 签发时间       |
| exp  | 过期时间       |
| jti  | Token 唯一标识 |

业务接口校验：

```
// 示意代码
Claims claims = jwtService.verify(accessToken);

if (!"access".equals(claims.get("type"))) {
    throw new UnauthorizedException();
}

Long userId = Long.valueOf(claims.getSubject());
```

这里 `verify()` 应完整校验签名、允许的签名算法、签发者、接收方、有效期等约束。

还需要强调：

**Access Token 和 Refresh Token 必须有明确的用途隔离，不能让 Refresh Token 通过普通业务接口的认证。**

### 7.2 Refresh Token

Refresh Token 可以使用安全随机数生成。

例如 Java：

```
SecureRandom random = new SecureRandom();

byte[] bytes = new byte[32];
random.nextBytes(bytes);

String refreshToken = Base64.getUrlEncoder()
        .withoutPadding()
        .encodeToString(bytes);
```

这里生成的是 256 位随机 Token，不是 JWT。

服务端可以对 Token 进行哈希后作为 Redis 查询标识，而不是直接存储原文。

例如：

```
Key:
refresh:token:<SHA256(R1)>

Value:
{
  "userId": 1001,
  "sessionId": "session-001",
  "familyId": "family-001",
  "expiresAt": 1792600000
}

TTL:
7 天
```

Refresh Token 随机性必须足够高，不能使用可预测的用户 ID、时间戳等直接拼接生成。

### 7.3 刷新逻辑

伪代码：

```
public TokenResponse refresh(String refreshToken) {

    String hash = sha256(refreshToken);

    // 1. 查询 Redis 会话
    RefreshSession session = redisService.get(hash);

    if (session == null) {
        throw new UnauthorizedException();
    }

    // 2. 校验会话状态、有效期和轮换约束
    validate(session);

    // 3. 原子地消费旧 Refresh Token，
    //    并创建下一代 Refresh Token
    String newRefreshToken =
            rotateAtomically(session, hash);

    // 4. 签发新的短期 JWT
    String accessToken =
            jwtService.createAccessToken(session.getUserId());

    return new TokenResponse(
            accessToken,
            newRefreshToken
    );
}
```

注意，这是一段结构性伪代码，不是可直接运行的完整代码。

生产实现必须保证轮换操作的原子性，正确处理失败重试和并发请求，并确保刷新接口受到适当的安全保护。

### 7.4 为什么 Redis 不需要每次访问都查？

假设每秒 1000 次需要认证的业务请求。

如果采用：

```
单个 Token + Redis 会话校验
```

每秒可能需要约 1000 次 Redis 会话查询。

如果采用：

```
短期 JWT + Redis Refresh Token
```

普通业务请求只需要验证 JWT，Redis 会话查询主要发生在刷新时。

这能减少认证过程中对 Redis 的依赖。

但存在代价：

**如果 Access Token 不查询服务端撤销状态，那么即使用户已经退出登录，之前签发的 Access Token 也可能继续有效到过期。**

如需立即封禁或撤销 Access Token，可以引入撤销列表、会话版本校验或集中认证查询等机制。

## 八、前端 Vue3 如何管理双 Token？

一种常见的 Web 设计是：

- Access Token 存在 JavaScript 内存中。
- Refresh Token 存在 HttpOnly Cookie 中。
- Axios 请求拦截器自动添加 Access Token。
- Access Token 过期时，通过刷新接口重新获取。
- 刷新失败则清除登录状态。

### 8.1 普通请求

```
api.interceptors.request.use(config => {
    const token = authStore.accessToken;

    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
});
```

### 8.2 Token 过期

前端检测到明确的 Access Token 过期响应：

```
业务请求
    ↓
Access Token 过期
    ↓
请求刷新接口
    ↓
获得新 Access Token
    ↓
更新内存中的 Token
    ↓
重试原业务请求
```

真实实现需要注意：

1. 同一时间有多个请求返回 401 时，应合并为一次刷新操作。
2. 每个原始请求最多执行有限次数的刷新重试，避免死循环。
3. 刷新接口自身不应触发递归刷新。
4. 页面刷新后，可以通过受保护的刷新接口恢复 Access Token。
5. 使用 Cookie 时，需要正确配置 CORS、SameSite、Secure 和 CSRF 防护。

### 8.3 为什么使用 HttpOnly Cookie？

如果 Refresh Token 保存在 localStorage：

```
localStorage.getItem("refreshToken");
```

能够在页面中执行的恶意 JavaScript 也可能读取它。

如果保存在 HttpOnly Cookie 中，JavaScript 不能直接读取 Cookie 的值。

但是：

- HttpOnly 不能彻底防御 XSS。
- 攻击者可能利用 XSS 操作用户当前会话。
- 浏览器会自动携带符合条件的 Cookie，因此还要考虑 CSRF。
- HttpOnly 不能抵御所有浏览器、设备或服务器环境中的凭证窃取。

因此，HttpOnly 是重要防护措施，但不是绝对安全保证。

## 九、为什么不直接使用 Redis + 单 Token？

其实完全可以。

例如：

```
用户登录
    ↓
生成随机 Session Token
    ↓
Redis 保存登录状态
    ↓
返回 Session Token
```

Redis：

```
session:abc123 → userId:1001

TTL = 7 天
```

每次请求：

```
前端携带 Session Token
    ↓
后端查询 Redis
    ↓
找到用户会话
    ↓
执行业务
```

退出登录：

```
删除 Redis 对应 Session
    ↓
Token 立即失效
```

只要所有请求都依赖服务端会话校验，撤销就可以立即生效。

这种方案本身没有问题。

### 三种认证方案对比

| 特征                     | 长期 JWT 单 Token | Redis Session 单 Token | 双 Token               |
| ------------------------ | ----------------- | ---------------------- | ---------------------- |
| 实现复杂度               | 低                | 低～中                 | 中～高                 |
| 普通请求是否需要查 Redis | 不需要            | 通常需要               | 不一定需要             |
| 长期凭证是否参与普通请求 | 是                | 是                     | 否                     |
| 主动撤销长期凭证         | 需额外机制        | 容易                   | 容易（有状态刷新方案） |
| 短期访问凭证可直接撤销   | 需额外机制        | 可以                   | 需额外机制             |
| 自动延续登录             | 需要额外设计      | 可通过 Session 续期    | 通过 Refresh Token     |
| 是否适合中小项目         | 是                | 是                     | 是，但复杂度更高       |

这里所谓的 JWT 单 Token 指的是不查询服务端状态的长期 JWT。

如果为 JWT 增加每次请求的 Redis 校验，那么它实际上也具有了有状态会话的部分特征。

## 十、双 Token 常见误区

### 误区一：双 Token 一定比单 Token 安全

错误。

如果两个 Token 都暴露给不可信代码，且 Refresh Token 可以无限使用，那么双 Token 的实际安全收益会大幅下降。

安全性依赖整个认证系统的设计，而不是 Token 数量。

### 误区二：Refresh Token 不能访问业务接口，所以被盗没关系

错误。

攻击者可以通过刷新接口取得 Access Token。

Refresh Token 实际上相当于一种长期登录凭证。

### 误区三：JWT 天然支持退出登录

错误。

JWT 的签名验证只能证明 Token 由可信方签发且内容没有被篡改等。

仅靠 JWT 签名和过期时间，无法知道用户是否已经退出登录或被封禁。

需要额外的撤销机制，或者等待 Token 自然过期。

### 误区四：Refresh Token Rotation 能完全防止盗用

错误。

轮换主要用于限制 Token 重复利用和检测异常重放。

如果攻击者先于用户完成刷新，仍然可能获得新 Token。

### 误区五：Access Token 和 Refresh Token 必须都是 JWT

错误。

常见设计反而是：

```
Access Token：JWT
Refresh Token：随机字符串 + Redis
```

这样结合了 JWT 本地验证与服务端会话可控的特点。

### 误区六：Token 放进 HTTPS 请求就绝对安全

错误。

HTTPS 保护的是传输过程，无法自动阻止：

- XSS。
- 恶意浏览器扩展。
- 服务端日志泄露。
- 终端设备被入侵。
- 代码错误地记录或转发 Token。

因此，不应把 Token 放入 URL 查询参数，也不应在日志中输出完整 Token。

## 十一、实际项目应该怎么选？

### 场景一：普通后台管理系统

如果并发要求不高，希望登录状态容易撤销，可以采用：

```
随机 Session Token
        +
Redis 会话管理
```

优点是简单、可控、容易退出和封禁。

不一定需要双 Token。

### 场景二：前后端分离的 Web 系统

可以采用：

```
Access Token
    ↓
短期 JWT
    ↓
负责普通业务请求

Refresh Token
    ↓
高强度随机字符串
    ↓
HttpOnly Cookie
    ↓
Redis 管理
    ↓
支持撤销与轮换
```

适合希望兼顾自动续期与高频认证效率的系统。

### 场景三：高安全要求系统

可以在双 Token 基础上进一步加入：

- Refresh Token Rotation。
- Refresh Token 重用检测和令牌家族撤销。
- 风险操作二次认证。
- 异常登录检测。
- 会话过期和重新认证机制。
- 在合适的客户端场景中使用 DPoP 或 mTLS 等持有者证明机制。

不能仅仅依靠双 Token 解决所有认证安全问题。

## 十二、推荐的 Spring Boot 认证架构

技术栈：

```
Spring Boot
Spring Security
JWT
Redis
Vue3
Axios
```

后端接口：

| 接口               | 功能                    |
| ------------------ | ----------------------- |
| POST /auth/login   | 用户登录并签发 Token    |
| POST /auth/refresh | 刷新 Access Token       |
| POST /auth/logout  | 撤销 Refresh Token 会话 |
| GET /api/user/info | 获取当前用户信息        |

认证流程：

```
                用户登录
                   │
                   ▼
             Spring Security
                   │
                   ▼
             验证账号密码
                   │
          ┌────────┴────────┐
          ▼                 ▼
     Access Token      Refresh Token
     JWT / 30min       随机值 / 7d
          │                 │
          ▼                 ▼
      前端内存          HttpOnly Cookie
          │                 │
          ▼                 ▼
      业务接口           刷新接口
          │                 │
          ▼                 ▼
      JWT 校验          Redis 校验
          │                 │
          ▼                 ▼
      执行业务          签发新 Token
```

建议将刷新与业务鉴权分离，业务接口只接受 Access Token。

刷新操作需要原子化，退出登录需要撤销对应刷新会话。若要求立即撤销已签发的 Access Token，还必须补充 Access Token 撤销校验。

具体有效期应根据系统风险和用户体验调整，30 分钟与 7 天只是常见示例，不是统一标准。

## 十三、面试常见问题

**Q1：为什么需要双 Token？**

通过短期 Access Token 访问业务接口，通过长期 Refresh Token 维持登录状态。

主要目的是减少长期凭证在高频业务请求中的暴露，并将访问授权和登录续期分开管理。

**Q2：为什么不把 Access Token 直接设置为 7 天？**

可以，但一旦普通请求中的长期 Token 被盗，攻击者就可能长期访问。

使用短期 Access Token 可以缩短普通访问凭证被盗后的可利用时间。

**Q3：Refresh Token 被盗，不也一样可以登录吗？**

是的。

Refresh Token 也是敏感凭证，必须通过安全存储、服务端撤销、Token Rotation 等措施保护。

双 Token 本身不能阻止长期凭证被盗。

**Q4：为什么 Access Token 用 JWT，Refresh Token 用 Redis？**

Access Token 请求频繁，JWT 可以本地验证，减少认证查询。

Refresh Token 请求较少，但需要支持撤销、轮换和会话管理，因此可以通过 Redis 维护状态。

**Q5：用户退出登录后，Access Token 会立即失效吗？**

不一定。

如果 Access Token 是完全无状态的 JWT，且业务接口只验证签名和有效期，那么退出登录后它仍然可能有效到过期。

如果需要立即失效，需要增加服务端撤销机制。

**Q6：双 Token 和 Redis Session 哪个更好？**

没有绝对答案。

Redis Session 更简单，可控性高，但普通认证请求通常需要查询 Redis。

JWT 双 Token 可以减少普通请求对 Redis 的依赖，但实现复杂度更高，也需要处理短期 JWT 的撤销问题。

## 十四、最终总结

理解双 Token，重点不在于有两个 Token，而在于将不同用途的凭证进行隔离。

**单个长期 Access Token：**

```
一个长期凭证
    ↓
所有业务请求直接使用
    ↓
泄露后可能长期访问
```

**双 Token：**

```
长期 Refresh Token
    │
    │ 低频使用
    ▼
短期 Access Token
    │
    │ 高频使用
    ▼
普通业务接口
```

这种设计的核心价值：

1. **缩短风险窗口**：普通 Access Token 被盗后，攻击者的可利用时间较短。
2. **降低长期凭证暴露范围**：Refresh Token 不参与日常业务请求。
3. **分离会话管理**：Refresh Token 可以通过 Redis 实现撤销和轮换。
4. **平衡性能和安全性**：普通 JWT 请求不必每次查询 Redis。
5. **改善用户体验**：Access Token 过期时可以自动刷新，不必频繁重新登录。

但必须记住：

**如果 Refresh Token 被盗且没有其他安全约束，攻击者仍然可以获取新的 Access Token。**

因此，双 Token 不是从根本上消除长期凭证被盗风险，而是通过凭证隔离、短期授权和受控续期，降低常见泄露场景中的风险。

一句话记忆：

**Access Token 负责短期访问，Refresh Token 负责长期续期；真正的安全性来自两者不同的暴露范围、生命周期和保护机制，而不是 Token 数量。**

补充 Spring Security 代码链路拆分双 Token 实现步骤修正 Cookie 与 CSRF 表述