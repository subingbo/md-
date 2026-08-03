# Spring MVC 参数接收注解总结

## 1. HTTP 参数位置

HTTP 请求中的参数主要有三个常见位置：

| 位置        | 示例           | Spring 注解     |
| ----------- | -------------- | --------------- |
| Query 参数  | `/user?id=100` | `@RequestParam` |
| Path 参数   | `/user/100`    | `@PathVariable` |
| Body 请求体 | JSON数据       | `@RequestBody`  |

---

# 2. @RequestParam

## 作用

接收 URL 查询参数（Query Parameter）。

参数位于：

?

```
后面。

---

## 示例

请求：

```http
GET /user?id=100&name=张三
```

Controller：

```java
@GetMapping("/user")
public User getUser(
        @RequestParam Long id,
        @RequestParam String name
) {

}
```

对应：

```
id = 100
name = 张三
```

------

## 常见使用场景

### 分页

请求：

```
GET /users?page=1&size=10
```

代码：

```java
@GetMapping("/users")
public List<User> list(
        @RequestParam int page,
        @RequestParam int size
){

}
```

------

### 条件查询

请求：

```
GET /users?name=Tom&age=20
```

代码：

```java
@GetMapping("/users")
public List<User> search(
        @RequestParam String name,
        @RequestParam Integer age
){

}
```

------

# 3. @PathVariable

## 作用

接收 URL 路径中的变量。

参数属于 URL 的一部分。

------

## 示例

请求：

```
GET /user/100
```

Controller：

```java
@GetMapping("/user/{id}")
public User getUser(
        @PathVariable Long id
){

}
```

执行：

```
/user/{id}

↓

/user/100

↓

id = 100
```

------

## 常见使用场景

根据资源 ID 查询：

```
GET /users/100
```

根据 ID 删除：

```
DELETE /users/100
```

根据 ID 修改：

```
PUT /users/100
```

------

# 4. @RequestBody

## 作用

接收 HTTP 请求 Body 中的数据。

通常用于 JSON。

------

## 示例

请求：

```http
POST /users
Content-Type: application/json

{
    "name":"张三",
    "age":20
}
```

Controller：

```java
@PostMapping("/users")
public void create(
        @RequestBody User user
){

}
```

Spring 自动把 JSON 转成对象。

------

# 5. 三者区别

| 注解            | 数据位置 | 示例           | 典型场景       |
| --------------- | -------- | -------------- | -------------- |
| `@RequestParam` | URL参数  | `/user?id=1`   | 查询条件、分页 |
| `@PathVariable` | 路径参数 | `/user/1`      | 资源ID         |
| `@RequestBody`  | 请求体   | `{name:"Tom"}` | 新增、修改     |

------

# 6. REST接口常见设计

## 查询列表

```
GET /users?page=1&size=10
```

使用：

```java
@RequestParam
```

------

## 查询详情

```
GET /users/100
```

使用：

```java
@PathVariable
```

------

## 创建数据

```
POST /users

{
    "name":"Tom"
}
```

使用：

```java
@RequestBody
```

------

## 修改数据

```
PUT /users/100

{
    "name":"Jerry"
}
```

通常：

```java
@PathVariable
+
@RequestBody
```

------

## 删除数据

```
DELETE /users/100
```

通常：

```java
@PathVariable
```

------

# 7. 快速记忆

```
@RequestParam
↓
? 后面的参数

@PathVariable
↓
路径里面的参数

@RequestBody
↓
请求体里的数据
```

例如：

```
POST /users/100?force=true

{
    "name":"Tom"
}
```

对应：

```java
@PathVariable Long id
@RequestParam boolean force
@RequestBody User user

```