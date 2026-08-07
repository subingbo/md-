# @Configuration、CGLIB代理与@Bean方法调用原理

## 1. CGLIB是什么？

CGLIB不是动态代理本身，而是**动态代理的一种实现方式**。

动态代理：

```
动态代理
├── JDK动态代理
│      └── 基于接口
│
└── CGLIB动态代理
       └── 基于继承
```

------

## 2. JDK动态代理 vs CGLIB

|          | JDK动态代理    | CGLIB动态代理             |
| -------- | -------------- | ------------------------- |
| 实现方式 | 生成接口实现类 | 生成目标类子类            |
| 要求     | 必须有接口     | 不需要接口                |
| 原理     | implements接口 | extends目标类             |
| 限制     | 接口方法       | 不能代理final类/final方法 |

------

### JDK动态代理

目标：

```java
interface UserService {
    void save();
}
```

Spring生成：

```java
class UserServiceProxy implements UserService {

    public void save(){
        //增强逻辑
        target.save();
    }
}
```

结构：

```
代理对象
    |
    ↓
接口
    |
    ↓
目标对象
```

------

### CGLIB动态代理

目标：

```java
class UserService {

    public void save(){
    }
}
```

Spring生成：

```java
class UserService$$CGLIB extends UserService {

    public void save(){
        //增强逻辑
        super.save();
    }
}
```

结构：

```
代理对象
    |
    ↓
继承目标类
    |
    ↓
目标对象
```

------

# 3. @Configuration为什么需要CGLIB？

## 问题背景

Spring中：

```java
@Configuration
public class MainConfig {

    @Bean
    public UserService userService(){
        return new UserService(roleService());
    }


    @Bean
    public RoleService roleService(){
        return new RoleService();
    }
}
```

看起来：

```java
roleService()
```

应该返回Spring容器里的Bean。

但是实际上：

## Java层面

Java不知道：

```java
@Bean
```

这个注解。

对于Java来说：

```java
userService(){

    return new UserService(roleService());

}
```

就是普通方法调用：

```
userService()
        |
        ↓
this.roleService()
        |
        ↓
new RoleService()
```

------

# 4. 不使用CGLIB会发生什么？

假设关闭：

```java
@Configuration(proxyBeanMethods = false)
```

Spring执行：

## 创建UserService

调用：

```java
userService()
```

内部：

```java
new UserService(roleService());
```

执行：

```java
new RoleService();
```

得到：

```
RoleService对象A
```

然后：

```
UserService
      |
      |
      RoleService A
```

------

## 创建RoleService Bean

Spring发现：

```java
@Bean
public RoleService roleService()
```

再次调用：

```java
roleService()
```

创建：

```
RoleService对象B
```

注册：

```
Spring容器:

UserService
RoleService B
```

------

最终：

```
Spring容器

UserService
      |
      |
      RoleService A
      （普通对象，不是Bean）


RoleService B
      （Spring Bean）
```

出现两个RoleService。

------

# 5. @Configuration的CGLIB代理解决问题

默认：

```java
@Configuration(proxyBeanMethods = true)
```

Spring不会直接使用：

```
MainConfig
```

而是生成：

```
MainConfig$$EnhancerByCGLIB
```

------

调用：

```java
userService()
```

实际上：

```
代理配置类
       |
       ↓
userService()
       |
       ↓
调用roleService()
       |
       ↓
CGLIB拦截
       |
       ↓
Spring容器.getBean(RoleService)
       |
       ↓
返回已有Bean
```

于是：

```
UserService
      |
      |
      RoleService Bean
```

保证单例。

------

# 6. 关键理解：不是所有new都会被代理

错误理解：

> CGLIB会把所有new对象改成从Spring容器获取

错误。

正确：

> CGLIB只拦截@Configuration类内部对@Bean方法的调用。

------

例如：

## 会被处理

```java
@Bean
public RoleService roleService(){
    return new RoleService();
}
```

调用：

```java
roleService();
```

会被代理。

------

## 不会被处理

```java
public void test(){

    RoleService service = new RoleService();

}
```

这里：

```java
new RoleService()
```

Spring完全不知道。

------

# 7. 为什么@Autowired不用CGLIB？

例如：

```java
@Autowired
private RoleService roleService;
```

流程：

```
对象创建
    |
    ↓
Spring容器查找Bean
    |
    ↓
注入对象
```

本来就是从容器获取。

不需要代理。

------

# 8. @Configuration代理的核心作用

一句话：

> @Configuration通过CGLIB代理拦截配置类内部@Bean方法调用，将普通Java方法调用转换为Spring容器Bean查找，从而保证@Bean方法返回的对象是同一个单例Bean。

------

# 9. @Transactional为什么不能解决this调用？

虽然：

```
@Configuration
```

和：

```
@Transactional
```

都使用代理。

但是目的不同。

------

## @Configuration

代理：

```
配置类
```

目的：

```
保证@Bean单例
```

处理：

```
@Bean方法调用
```

------

## @Transactional

代理：

```
业务Bean
```

目的：

```
添加事务增强
```

流程：

```
调用方
 |
 ↓
代理对象
 |
 ↓
事务拦截器
 |
 ↓
目标方法
```

------

但是：

```java
public void test(){

    save();

}
```

内部调用：

```
this.save()
```

绕过代理。

所以：

```
事务失效
```

------

# 10. 最终总结

| 问题                               | 答案                   |
| ---------------------------------- | ---------------------- |
| CGLIB是什么？                      | 动态代理实现方式       |
| CGLIB代理什么？                    | 目标类子类             |
| @Configuration为什么用CGLIB？      | 拦截@Bean方法调用      |
| 为什么需要拦截？                   | 避免内部调用创建新对象 |
| 所有new都会被拦截吗？              | 不会                   |
| 哪些会被拦截？                     | 调用@Bean方法          |
| @Autowired为什么不用？             | 本身就是容器获取       |
| @Transactional为什么this调用失效？ | 内部调用绕过代理       |

核心记忆：

```
@Bean方法
    ↓
普通Java调用
    ↓
@Configuration + CGLIB拦截
    ↓
Spring容器获取Bean
    ↓
保证单例
```