# @Configuration、CGLIB代理与@Bean方法调用原理


# 1. CGLIB是什么？

CGLIB（Code Generation Library）是一种动态代理实现方式。

动态代理主要有两种：

动态代理
│
├── JDK动态代理
│ └── 基于接口
│
└── CGLIB动态代理
└── 基于继承

```
## JDK动态代理

特点：

- 基于接口实现
- 生成接口实现类
- 目标类必须实现接口


例如：

```java
public interface UserService {

    void save();

}
```

目标类：

```java
public class UserServiceImpl implements UserService {

    public void save(){

    }

}
```

Spring生成代理：

```java
class UserServiceProxy implements UserService {


    private UserService target;


    public void save(){

        //增强逻辑

        target.save();

    }

}
```

结构：

```
调用方

  ↓

代理对象

  ↓

目标对象
```

------

# 2. CGLIB动态代理

CGLIB基于继承生成目标类的子类。

目标类：

```java
public class UserService {


    public void save(){

    }

}
```

生成：

```java
class UserService$$Enhancer extends UserService {


    @Override
    public void save(){

        //增强逻辑

        super.save();

    }

}
```

结构：

```
代理对象

   ↓

继承目标类
```

特点：

- 不需要接口
- 通过继承实现
- 不能代理final类
- 不能代理final方法

------

# 3. @Configuration为什么需要CGLIB？

Spring配置类：

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

目标：

保证：

```
@Bean方法返回的是Spring容器中的同一个Bean
```

------

# 4. 不使用CGLIB会发生什么？

配置：

```java
@Configuration(proxyBeanMethods = false)
```

此时：

@Bean方法就是普通Java方法。

调用：

```java
userService()
```

Java执行：

```java
return new UserService(roleService());
```

内部：

```java
roleService()
```

等价：

```java
this.roleService();
```

普通Java认为：

```
调用当前对象的方法
```

于是：

创建：

```
RoleService对象A
```

之后Spring创建：

```java
@Bean
public RoleService roleService()
```

又创建：

```
RoleService对象B
```

最终：

```
Spring容器:

RoleService Bean(B)


UserService:

    |
    ↓

RoleService A
```

出现两个对象。

------

# 5. @Configuration + CGLIB如何解决？

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

类似：

```java
class MainConfig$$Enhancer extends MainConfig {


    @Override
    public RoleService roleService(){

        return beanFactory.getBean(RoleService.class);

    }

}
```

也就是说：

Spring重写了@Bean方法。

------

# 6. @Configuration内部this为什么有效？

调用：

```java
userService()
```

执行流程：

```
MainConfig增强对象

        ↓

userService()

        ↓

this.roleService()

        ↓

增强后的roleService()

        ↓

Spring容器.getBean()

        ↓

返回已有Bean
```

关键：

这里：

```java
this
```

指向：

```
MainConfig$$Enhancer
```

不是原来的：

```
MainConfig
```

所以：

```java
this.roleService()
```

实际调用：

```
增强后的方法
```

保证：

```
@Bean单例
```

------

# 7. CGLIB不是拦截所有new

错误理解：

> CGLIB会把所有new对象替换成Spring Bean

正确：

> CGLIB只增强@Configuration类中的@Bean方法调用。

例如：

## 会被增强

```java
@Bean
public RoleService roleService(){

    return new RoleService();

}
```

调用：

```
roleService()
```

会进入CGLIB增强逻辑。

------

## 不会被增强

```java
public void test(){

    RoleService roleService = new RoleService();

}
```

这里：

```
new RoleService()
```

Spring不知道。

------

# 8. 为什么@Autowired不用CGLIB？

例如：

```java
@Autowired
private RoleService roleService;
```

流程：

```
创建对象

↓

Spring容器查找Bean

↓

注入Bean
```

本身已经是：

```
从Spring容器获取对象
```

不需要代理方法调用。

------

# 9. @Configuration和@Transactional区别

虽然：

```
@Configuration
```

和：

```
@Transactional
```

都可能使用CGLIB。

但是作用完全不同。

------

# 10. @Configuration代理

目的：

```
保证@Bean方法返回Spring Bean
```

增强对象：

```
@Configuration类本身
```

结构：

```
MainConfig$$Enhancer

        ↓

作为Bean使用
```

所以：

```java
this.bean()
```

可以进入增强方法。

------

# 11. @Transactional代理

目的：

```
给方法添加事务能力
```

Spring AOP结构：

```
调用方

 ↓

代理对象(proxy)

 ↓

事务拦截器

 ↓

目标对象(target)
```

例如：

```java
@Service
public class UserService {


    public void a(){

        this.b();

    }


    @Transactional
    public void b(){

    }

}
```

调用：

```java
userService.a();
```

其中：

```
userService
```

实际上是：

```
proxy
```

执行：

```
proxy.a()

↓

事务拦截器

↓

target.a()

↓

this.b()

↓

target.b()
```

这里：

```
this = target
```

所以：

```
target.b()
```

不会经过：

```
proxy.b()
```

因此：

```
@Transactional失效
```

------

# 12. 为什么外部直接调用事务方法有效？

调用：

```java
userService.b();
```

实际上：

```
proxy.b()
```

流程：

```
proxy.b()

↓

事务拦截器

↓

target.b()
```

因为经过代理：

```
@Transactional生效
```

------

# 13. 内部调用和外部调用区别

## 外部调用

代码：

```java
userService.b();
```

对象：

```
proxy
```

流程：

```
proxy

↓

事务增强

↓

target
```

结果：

```
事务生效
```

------

## 内部调用

代码：

```java
this.b();
```

对象：

```
target
```

流程：

```
target

↓

target.b()
```

结果：

```
绕过代理

事务不生效
```

------

# 14. 核心区别总结

|          | @Configuration   | @Transactional |
| -------- | ---------------- | -------------- |
| 实现方式 | CGLIB增强        | JDK/CGLIB代理  |
| 增强对象 | 配置类本身       | 业务Bean代理   |
| 主要目的 | 保证@Bean单例    | 添加事务       |
| 调用对象 | 增强后的对象     | proxy + target |
| this指向 | 增强对象         | target         |
| 内部调用 | 可以进入增强逻辑 | 绕过代理       |
| 解决问题 | Bean创建控制     | 方法执行增强   |

------

# 15. 最终记忆

## @Configuration

```
@Bean方法调用

↓

@Configuration CGLIB增强

↓

this指向增强对象

↓

调用Spring容器

↓

返回已有Bean

↓

保证单例
```

## @Transactional

```
外部调用

↓

代理对象

↓

事务拦截器

↓

目标对象

↓

业务方法


内部this调用

↓

目标对象直接调用

↓

绕过代理

↓

事务增强失效
```

一句话总结：

> @Configuration通过CGLIB直接增强配置类对象，修改@Bean方法调用逻辑，使内部this调用能够进入Spring容器；@Transactional通过AOP代理包装业务对象，业务方法运行在target对象中，内部this调用不会经过代理，因此无法触发事务增强。