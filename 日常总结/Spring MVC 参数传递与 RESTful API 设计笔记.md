# Spring MVC 参数传递与 RESTful API 设计笔记

---

# 一、为什么现在还有 Path 参数？

很多人刚学 Spring Boot 时会疑惑：

为什么不直接全部使用 `@RequestBody`？

例如：

```http
POST /user/get

{
    "id":123
}
```

不是也能拿到参数吗？

实际上：

```text
Path
Query
Body
```

在 HTTP 中职责完全不同。

Body 并不是 Path 和 Query 的替代品。

---

# 二、HTTP 请求结构

一个完整 HTTP 请求：

```http
PUT /users/123?notify=true HTTP/1.1
Host: api.xxx.com
Content-Type: application/json

{
    "name":"Tom",
    "age":20
}
```

拆开看：

```text
请求行
↓
PUT /users/123?notify=true HTTP/1.1

请求头
↓
Content-Type: application/json

请求体（Body）
↓
{
    "name":"Tom",
    "age":20
}
```

---

# 三、Path、Query、Body 的位置

## Path

```text
/users/123
```

用于：

```text
定位资源
```

表示：

```text
用户ID=123
```

---

## Query

```text
?notify=true
```

用于：

```text
查询条件
附加选项
分页
排序
```

表示：

```text
notify=true
```

---

## Body

```json
{
    "name":"Tom",
    "age":20
}
```

用于：

```text
提交数据内容
```

表示：

```text
修改后的用户信息
```

---

# 四、三种参数对应的 Spring 注解

| 参数类型 | Spring注解    | 用途     |
| -------- | ------------- | -------- |
| Path     | @PathVariable | 资源定位 |
| Query    | @RequestParam | 查询条件 |
| Body     | @RequestBody  | 数据内容 |

---

# 五、Path 参数

## 作用

标识要操作的资源。

例如：

```http
GET /users/123
```

表示：

```text
查询用户123
```

---

```http
DELETE /users/123
```

表示：

```text
删除用户123
```

---

## Spring写法

```java
@GetMapping("/users/{id}")
public User getUser(
        @PathVariable Long id) {
    ...
}
```

访问：

```http
GET /users/123
```

得到：

```java
id = 123
```

---

## Path特点

适用于：

```text
用户ID
订单ID
商品ID
文章ID
```

例如：

```text
/users/123
/orders/456
/books/789
```

URL本身就能表达资源。

---

# 六、Query 参数

## 作用

表示查询条件。

---

## 分页

```http
GET /users?page=1&size=10
```

对应：

```java
@RequestParam Integer page
@RequestParam Integer size
```

---

## 搜索

```http
GET /users?name=Tom
```

对应：

```java
@RequestParam String name
```

---

## 排序

```http
GET /users?sort=createTime
```

---

## Query特点

适用于：

```text
分页
筛选
搜索
排序
附加选项
```

---

# 七、Body 参数

## 作用

用于提交数据内容。

一般用于：

```text
新增
修改
复杂对象传输
```

---

## Spring写法

```java
@PostMapping("/users")
public void create(
        @RequestBody UserCreateReqVO reqVO) {
}
```

请求：

```http
POST /users

{
    "name":"Tom",
    "age":20
}
```

---

## 修改

```java
@PutMapping("/users/{id}")
public void update(
        @PathVariable Long id,
        @RequestBody UserUpdateReqVO reqVO) {
}
```

请求：

```http
PUT /users/123

{
    "name":"Jerry"
}
```

得到：

```java
id = 123

reqVO.name = "Jerry"
```

---

# 八、为什么 Query 不放 Body？

很多人误以为：

```text
Query 就是 Body 的一部分
```

实际上不是。

例如：

```http
GET /users?page=1&size=10
```

这里：

```text
?page=1&size=10
```

属于：

```text
URL
```

而不是：

```text
Body
```

---

## Query的位置

```http
GET /users?page=1&size=10
```

URL：

```text
/users?page=1&size=10
```

---

## Body的位置

```http
POST /users

{
    "name":"Tom"
}
```

Body：

```json
{
    "name":"Tom"
}
```

两者在 HTTP 协议中属于完全不同的位置。

---

# 九、为什么不全部使用 Body？

例如：

```http
POST /user/search

{
    "page":1,
    "size":10,
    "name":"Tom"
}
```

技术上完全可以。

很多企业项目就是这么做的。

但会失去很多 HTTP 的优势。

---

## 原因1：URL失去语义

REST风格：

```http
GET /users/123
```

一眼知道：

```text
查询用户123
```

---

RPC风格：

```http
POST /user/get

{
    "id":123
}
```

必须打开Body才能知道。

---

## 原因2：浏览器天然支持Query

直接访问：

```text
https://xxx.com/users?page=1
```

浏览器就能发请求。

---

但是：

```json
{
    "page":1
}
```

不能写到浏览器地址栏。

---

## 原因3：缓存支持更好

例如：

```http
GET /users?page=1
```

浏览器：

```text
可缓存
```

CDN：

```text
可缓存
```

Nginx：

```text
可缓存
```

---

因为：

```text
/users?page=1
/users?page=2
```

是两个不同URL。

---

如果放Body：

```http
POST /user/search

{
    "page":1
}
```

很多缓存系统根本不会缓存。

---

## 原因4：符合HTTP设计

HTTP本身设计：

```text
GET
获取资源

POST
提交数据

PUT
修改资源

DELETE
删除资源
```

因此：

```text
资源定位 → Path
查询条件 → Query
数据内容 → Body
```

职责明确。

---

# 十、一个完整请求分析

请求：

```http
PUT /users/123?notify=true HTTP/1.1
Host: api.xxx.com
Content-Type: application/json

{
    "name":"Tom",
    "age":20
}
```

---

## Path

```text
/users/123
```

表示：

```text
修改哪个用户
```

得到：

```java
id = 123
```

---

## Query

```text
notify=true
```

表示：

```text
修改后是否发送通知
```

得到：

```java
notify = true
```

---

## Body

```json
{
    "name":"Tom",
    "age":20
}
```

表示：

```text
修改成什么内容
```

得到：

```java
reqVO.name = "Tom"
reqVO.age = 20
```

---

最终Spring接收到：

```java
id = 123

notify = true

reqVO = {
    name = "Tom",
    age = 20
}
```

---

# 十一、REST 风格 VS RPC 风格

## REST风格（推荐）

查询：

```http
GET /users/123
```

新增：

```http
POST /users
```

修改：

```http
PUT /users/123
```

删除：

```http
DELETE /users/123
```

特点：

```text
URL表示资源
HTTP方法表示动作
```

---

## RPC风格（国内项目很多）

查询：

```http
POST /user/get
{
    "id":123
}
```

修改：

```http
POST /user/update
{
    "id":123,
    "name":"Tom"
}
```

删除：

```http
POST /user/delete
{
    "id":123
}
```

特点：

```text
开发方便
统一POST
不严格遵循REST
```

---

# 十二、Spring中最常见写法

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User get(
            @PathVariable Long id) {
    }

    @GetMapping
    public PageResult<User> page(
            @RequestParam Integer pageNo,
            @RequestParam Integer pageSize) {
    }

    @PostMapping
    public void create(
            @RequestBody UserCreateReqVO reqVO) {
    }

    @PutMapping("/{id}")
    public void update(
            @PathVariable Long id,
            @RequestBody UserUpdateReqVO reqVO) {
    }

    @DeleteMapping("/{id}")
    public void delete(
            @PathVariable Long id) {
    }
}
```

---

# 十三、面试总结

## Path、Query、Body 的区别

### Path

```java
@PathVariable
```

作用：

```text
定位资源
```

示例：

```http
/users/123
```

---

### Query

```java
@RequestParam
```

作用：

```text
查询条件
分页
筛选
排序
```

示例：

```http
/users?page=1&size=10
```

---

### Body

```java
@RequestBody
```

作用：

```text
提交数据内容
新增
修改
复杂对象
```

示例：

```http
POST /users

{
    "name":"Tom"
}
```

---

# 一句话记忆

资源在哪里
→ Path

怎么查
→ Query

数据是什么
→ Body

即：

Path = 操作谁

Query = 怎么操作

Body = 改成什么