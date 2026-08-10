# Spring Cache @Cacheable 原理与使用总结

## 1. @Cacheable 本质

`@Cacheable` 是 Spring 提供的缓存注解。

核心作用：

> 增强一个方法的调用流程，将方法返回值缓存起来。

它不是缓存 SQL，也不是缓存某个查询语句。

例如：

```java
@Cacheable(
    cacheNames = "article",
    key = "#id"
)
public Article getArticle(Long id){

    return articleMapper.selectById(id);
}
```

实际效果类似：

```java
public Article getArticle(Long id){

    // 1. 先查缓存
    Article article = cache.get(id);

    if(article != null){
        return article;
    }


    // 2. 缓存不存在，执行原方法
    article = articleMapper.selectById(id);


    // 3. 保存返回值
    cache.put(id, article);


    return article;
}
```

------

# 2. Controller 本质

Controller 方法本质上还是一个普通 Java 方法。

例如：

```java
@GetMapping("/{id}")
public Article detail(
        @PathVariable Long id
){

    return articleService.getArticle(id);
}
```

从 Java 角度：

```java
public Article detail(Long id){

}
```

只是 Spring MVC 额外做了：

```
HTTP请求
    |
    v
DispatcherServlet
    |
    v
找到Controller方法
    |
    v
参数绑定
    |
    v
执行Java方法
    |
    v
返回结果
    |
    v
转换JSON
```

------

# 3. @Cacheable 加在哪里，就增强哪里

例如：

```java
@RestController
public class ArticleController {


    @Cacheable(
        cacheNames="article",
        key="#id"
    )
    @GetMapping("/{id}")
    public Article detail(Long id){

        return articleMapper.selectById(id);
    }

}
```

Spring 会代理 Controller 的 `detail()` 方法。

执行流程：

```
请求
 |
 v
Controller代理对象
 |
 v
CacheInterceptor
 |
 +---- 查询缓存
 |
 +---- 缓存存在
 |        |
 |        v
 |      返回缓存
 |
 +---- 缓存不存在
          |
          v
       执行detail()
          |
          v
       保存返回值
```

------

# 4. 缓存的是返回值，不是查询过程

例如：

```java
@Cacheable(
    cacheNames="article",
    key="#id"
)
public ArticleDetailVO detail(Long id){

    Article article =
        articleMapper.selectById(id);


    User user =
        userMapper.selectById(article.getUserId());


    List<Comment> comments =
        commentMapper.list(id);


    return new ArticleDetailVO(
        article,
        user,
        comments
    );
}
```

缓存内容：

```
article::100

        |
        v

ArticleDetailVO
{
    article,
    user,
    comments
}
```

不是：

```
article表缓存
user表缓存
comment表缓存
```

Spring Cache 不知道方法里面执行了什么。

它只关注：

```
输入参数
    |
    v
方法执行
    |
    v
返回结果
```

------

# 5. Controller 参数方式不影响缓存

Spring Cache 不关心参数来自哪里。

## 方式1：路径参数

```java
@GetMapping("/{id}")
@Cacheable(
    cacheNames="article",
    key="#id"
)
public Article detail(
        @PathVariable Long id
){

}
```

请求：

```
GET /article/100
```

方法：

```java
detail(100)
```

缓存：

```
article::100
```

------

## 方式2：查询参数

```java
@GetMapping("")
@Cacheable(
    cacheNames="article",
    key="#id"
)
public Article detail(
        @RequestParam Long id
){

}
```

请求：

```
GET /article?id=100
```

方法：

```java
detail(100)
```

缓存：

```
article::100
```

------

## 方式3：请求体

```java
@PostMapping("")
@Cacheable(
    cacheNames="article",
    key="#query.id"
)
public Article detail(
        @RequestBody ArticleQuery query
){

}
```

请求：

```json
{
    "id":100
}
```

key：

```
query.id = 100
```

缓存：

```
article::100
```

------

# 6. 多个 Controller 方法不会共享缓存

例如：

方法A：

```java
@Cacheable(
    cacheNames="article",
    key="#id"
)
public Article detail(Long id){

}
```

方法B：

```java
public Article info(Long id){

}
```

执行：

```
detail(100)
```

产生：

```
article::100
```

然后：

```
info(100)
```

不会自动查询：

```
article::100
```

原因：

`@Cacheable` 增强的是 detail 方法。

不是：

```
cacheNames="article"
```

以后所有 Article 查询自动共享。

------

# 7. 为什么一般不把缓存放 Controller

Controller 通常负责：

```
接收请求
参数转换
返回响应
```

一个 Controller 方法可能包含：

```
查询文章

查询作者

查询评论

组装VO
```

如果缓存 Controller：

缓存的是整个接口结果：

```
articleDetail::100

{
 article,
 user,
 comments
}
```

粒度比较粗。

------

推荐：

```
Controller

    |
    v

Service

    |
    +------ ArticleService
    |              |
    |          @Cacheable
    |
    +------ UserService
    |              |
    |          @Cacheable
    |
    +------ CommentService
                   |
               @Cacheable
```

每个业务数据自己管理缓存。

------

# 8. Service 层缓存示例

```java
@Service
public class ArticleService {


    @Cacheable(
        cacheNames="article",
        key="'detail:'+#id",
        unless="#result==null"
    )
    public Article getArticle(Long id){

        return articleMapper.selectById(id);

    }

}
```

缓存：

```
article::detail:100
        |
        v
Article对象
```

Controller：

```java
@GetMapping("/{id}")
public Article detail(Long id){

    return articleService.getArticle(id);

}
```

------

# 9. @Cacheable 参数

## cacheNames

缓存空间。

例如：

```java
cacheNames="article"
```

类似 Redis 前缀：

```
article::100
```

------

## key

缓存唯一标识。

例如：

```java
key="#id"
```

调用：

```java
getArticle(100)
```

生成：

```
article::100
```

------

## 常见 key 写法

### 参数

```java
key="#id"
```

表示方法参数 id。

------

### 多参数

```java
key="#userId+'-'+#articleId"
```

结果：

```
1-100
```

------

### 对象属性

```java
key="#query.id"
```

------

### 添加业务前缀

推荐：

```java
key="'detail:'+#id"
```

结果：

```
detail:100
```

避免不同业务 key 冲突。

------

# 10. condition 和 unless

## condition

执行前判断：

```java
@Cacheable(
    key="#id",
    condition="#id>0"
)
```

只有：

```
id > 0
```

才缓存。

------

## unless

执行后判断：

```java
@Cacheable(
    key="#id",
    unless="#result==null"
)
```

如果返回：

```
null
```

不缓存。

常用于防止缓存穿透。

------

# 11. Spring Cache 注意事项

## 1. 类内部调用不会触发缓存

例如：

```java
public void test(){

    this.getArticle(100);

}
```

不会经过 Spring 代理。

原因：

```
this调用
 |
绕过代理
 |
没有CacheInterceptor
```

------

## 2. 参数名称问题

推荐：

```java
key="#id"
```

需要保留方法参数名。

如果参数名不存在，可以：

```java
key="#p0"
```

表示第一个参数。

------

# 12. 核心总结

1. `@Cacheable` 增强的是方法调用。
2. 缓存的是方法返回值。
3. Controller 本质是普通 Java 方法。
4. Controller 使用不同参数来源（PathVariable、RequestParam、RequestBody）不会影响缓存。
5. 只有调用被 `@Cacheable` 修饰的方法，才会经过缓存逻辑。
6. 多个方法不会因为 `cacheNames` 相同自动共享缓存。
7. 实际项目一般将缓存放在 Service 层。
8. Service 缓存粒度更合理，可以被多个入口复用。

```

```