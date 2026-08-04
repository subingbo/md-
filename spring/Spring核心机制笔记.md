# Spring核心机制笔记：Starter自动配置与Bean生命周期

## 一、Spring Starter 自动配置机制

### 1. Starter是什么？

Spring Boot Starter本质：

> 一个封装好的功能插件，引入依赖后，可以自动向Spring容器注册Bean。

例如：

```xml
<dependency>
    <groupId>xxx</groupId>
    <artifactId>my-redis-starter</artifactId>
</dependency>
```

引入后：

```java
@Autowired
private RedisClient redisClient;
```

即可使用。

---

## 二、为什么Starter需要自动配置？

Spring默认扫描：

- @Component
- @Service
- @Repository
- @Controller

扫描范围通常是启动类所在包及子包。

第三方Starter：

```
my-starter.jar

com.xxx.redis
    RedisAutoConfiguration
```

不属于用户项目扫描范围，所以不能靠@ComponentScan发现。

Spring不会扫描所有依赖jar，因为依赖数量巨大，会导致启动慢。

所以Starter使用自动配置机制。

---

# 三、META-INF是什么？

META-INF是Java jar中的元信息目录。

Starter：

```
my-starter.jar

com.xxx.RedisAutoConfiguration.class

META-INF
 └── spring
      └── AutoConfiguration.imports
```

文件：

```
com.xxx.RedisAutoConfiguration
```

作用：

告诉Spring：

> 这个jar中有哪些自动配置类需要加载。

流程：

```
Spring启动

↓

@EnableAutoConfiguration

↓

读取META-INF/spring/AutoConfiguration.imports

↓

加载自动配置类

↓

@Bean创建Bean

↓

放入IOC容器
```

---

# 四、自动配置类

Starter不是一堆Bean。

通常包含：

```
RedisClient.class
RedisUtil.class
RedisProperties.class
RedisAutoConfiguration.class
```

普通类：

```java
public class RedisClient {

}
```

不是Bean。

自动配置：

```java
@Configuration
public class RedisAutoConfiguration {

    @Bean
    public RedisClient redisClient(){
        return new RedisClient();
    }

}
```

负责创建Bean。

---

# 五、Bean和Spring容器

## Bean

Bean：

> 被Spring管理的Java对象。

普通对象：

```java
UserService userService = new UserService();
```

不是Bean。

Spring管理：

```java
@Service
public class UserService {

}
```

是Bean。

---

## Spring容器

容器：

> 管理Bean的地方。

例如：

```
ApplicationContext

{
 userService : UserService对象,
 userDao : UserDao对象
}
```

负责：

- 创建Bean
- 保存Bean
- 注入依赖
- 生命周期管理
- AOP代理

---

# 六、Bean生命周期

完整流程：

```
创建IOC容器

↓

扫描Bean

↓

生成BeanDefinition

↓

实例化Bean(new对象)

↓

属性注入@Autowired

↓

Aware接口回调

↓

BeanPostProcessor.before

↓

初始化

↓

BeanPostProcessor.after(AOP代理)

↓

放入IOC容器

↓

使用

↓

销毁
```

---

# 七、初始化三种方式

三种不是三个阶段，而是同一个初始化阶段的三种写法。

执行顺序：

```
@PostConstruct

↓

InitializingBean.afterPropertiesSet()

↓

init-method
```

---

## 1. @PostConstruct

```java
@PostConstruct
public void init(){

}
```

推荐方式。

---

## 2. InitializingBean

```java
public class UserService implements InitializingBean {

    @Override
    public void afterPropertiesSet(){

    }

}
```

缺点：

业务类依赖Spring接口。

---

## 3. init-method

```java
@Bean(initMethod="start")
public RedisClient redisClient(){

    return new RedisClient();

}
```

特点：

非侵入式。

适合第三方类。

不需要修改源码。

---

# 八、BeanPostProcessor

作用：

> Spring提供的Bean统一加工入口。

生命周期：

```
属性注入

↓

BeanPostProcessor.before

↓

初始化

↓

BeanPostProcessor.after

```

## 前置方法

初始化之前处理所有Bean。

用途：

- 修改Bean属性
- 注入额外信息
- 统一日志处理

## 后置方法

初始化之后处理。

典型用途：

AOP代理。

例如：

```
UserService对象

↓

BeanPostProcessor

↓

UserService代理对象
```

---

# 九、三级缓存解决循环依赖

循环依赖：

```
A -> B -> A
```

例如：

```java
class A {

@Autowired
B b;

}

class B {

@Autowired
A a;

}
```

---

# 十、三级缓存

## 一级缓存

singletonObjects

存：

> 完整Bean


## 二级缓存

earlySingletonObjects

存：

> 早期Bean引用


## 三级缓存

singletonFactories

存：

> 创建早期Bean的工厂

---

# 十一、什么是早期对象？

早期对象：

> 已经实例化，但是还没有完成完整生命周期的Bean。

例如：

```java
A a = new A();
```

但是：

```
a.b = null
```

此时就是早期对象。

---

# 十二、循环依赖流程

创建A：

```
new A()

↓

三级缓存：

A -> ObjectFactory
```

A需要B。

创建B。

B需要A。

查找：

一级缓存：

没有。

二级缓存：

没有。

三级缓存：

找到ObjectFactory。

调用：

```
ObjectFactory.getObject()
```

获得A的早期引用。

---

# 十三、为什么需要三级缓存？

如果直接暴露A对象：

后续A可能需要AOP代理。

例如：

```
原始A

↓

A代理对象
```

代理对象和原对象不是同一个对象。

如果B拿到原对象：

```
B -> 原A

容器 -> A代理
```

会导致对象不一致。

---

三级缓存保存工厂：

```
A -> ObjectFactory
```

真正需要A时：

Spring决定返回：

- 原始对象
- 代理对象

保证引用正确。

---

# 十四、为什么B拿到早期A后还能完整？

因为Java保存的是引用。

B保存：

```
B.a -> A对象
```

不是复制。

后面：

```
A.b = B
```

修改的是同一个A对象。

所以最终：

```
A
 |
 B

B
 |
 A
```

形成完整循环引用。

---

# 总结

```
Starter
    |
    ↓
META-INF告诉Spring自动配置入口
    |
    ↓
AutoConfiguration创建Bean
    |
    ↓
IOC容器管理Bean
    |
    ↓
Bean生命周期
创建 → 注入 → 初始化 → 代理 → 使用
    |
    ↓
三级缓存解决循环依赖和AOP代理一致性
```
