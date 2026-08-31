# Spring Bean 三级缓存与循环依赖原理

## 一、Spring Bean 创建流程

Spring 创建单例 Bean 大致流程：

```
实例化 Bean
    ↓
依赖注入
    ↓
初始化
    ↓
BeanPostProcessor 后置处理
    ↓
放入一级缓存
    ↓
业务使用
```

其中：

- 构造器注入：发生在实例化阶段
- Setter / 字段注入：发生在属性填充阶段

---

# 二、三级缓存分别存什么？

Spring 单例 Bean 有三级缓存：

```
一级缓存 singletonObjects

二级缓存 earlySingletonObjects

三级缓存 singletonFactories
```

---

## 1. 一级缓存

存放：

> 完全初始化完成的 Bean

例如：

```
singletonObjects

A → 完整A对象
```

业务调用：

```java
applicationContext.getBean(A.class)
```

正常从这里获取。

---

## 2. 二级缓存

存放：

> Bean 的早期引用（early reference）

也就是：

已经实例化，但是还没有完成初始化的引用。

例如：

```
earlySingletonObjects

A → A早期引用
```

这个引用可能：

- 原始对象引用
- 代理对象引用

取决于是否需要 AOP。

---

## 3. 三级缓存

存放：

> 创建早期引用的工厂

不是 Bean 对象。

结构：

```
singletonFactories

A
 |
 ↓
ObjectFactory
```

类似：

```java
ObjectFactory factory = () -> getEarlyBeanReference(bean);
```

它的作用：

> 等真正需要提前暴露 Bean 时，再决定返回什么引用。

---

# 三、为什么三级缓存存 ObjectFactory？

因为 Spring 需要考虑 AOP。

例如：

```java
@Service
@Transactional
class A {

}
```

最终 Spring 暴露的不是：

```
原始A对象
```

而是：

```
A代理对象
```

结构：

```
          代理对象
              |
              ↓
          原始A对象
```

如果三级缓存直接保存原始对象：

```
A → 原始A
```

循环依赖时可能出现：

```
B.a → 原始A

容器中的 A → 代理A
```

导致同一个 Bean 有两个不同引用。

所以三级缓存保存：

```
A → ObjectFactory
```

需要时：

```
ObjectFactory.getObject()
        ↓
getEarlyBeanReference()
        ↓
决定返回:
        |
        ├── 无AOP
        |      ↓
        |   原始A引用
        |
        └── 有AOP
               ↓
            A代理引用
```

---

# 四、三级缓存解决循环依赖流程

例如：

```java
@Component
class A {

    @Autowired
    B b;
}


@Component
class B {

    @Autowired
    A a;
}
```

依赖：

```
A → B
B → A
```

---

## 创建 A

首先实例化：

```java
A a = new A();
```

此时：

```
A对象存在

但是：

A.b = null
```

这是半成品。

然后：

放入三级缓存：

```
三级缓存

A → ObjectFactory
```

---

## 创建 B

B 需要 A：

```
创建B
 ↓
发现需要A
 ↓
查询一级缓存
没有
 ↓
查询二级缓存
没有
 ↓
查询三级缓存
找到ObjectFactory
```

---

调用：

```
ObjectFactory.getObject()
```

得到早期引用：

情况1：无AOP

```
返回原始A对象引用
```

情况2：有AOP

```
创建A代理对象
返回代理对象引用
```

---

然后：

```
放入二级缓存
```

例如：

```
二级缓存

A → A代理对象
```

---

B 注入：

```
B.a = A引用
```

---

继续完成 A：

```
A.b = B
```

初始化完成。

最终：

```
一级缓存

A → 最终Bean
```

---

# 五、二级缓存为什么存在？

假设：

```
A ← B
A ← C
A ← D
```

第一次：

```
B需要A

三级缓存
 ↓
ObjectFactory
 ↓
生成早期引用
 ↓
放入二级缓存
```

之后：

```
C需要A

一级缓存没有
 ↓
二级缓存存在
 ↓
直接拿早期引用
```

不会重复创建代理。

---

# 六、为什么构造器循环依赖无法解决？

例如：

```java
class A {

    A(B b){

    }
}


class B {

    B(A a){

    }
}
```

创建：

```
创建A
 ↓
需要B
 ↓
创建B
 ↓
需要A
```

问题：

A 还没有创建完成。

因为：

```java
new A(B)
```

必须先拿到 B。

此时：

```
A对象不存在
```

三级缓存只能保存：

```
已经实例化的对象引用
```

没有对象：

```
无法提前暴露
```

所以失败。

---

# 七、Setter注入为什么可以解决？

Setter：

```java
class A {

    private B b;

    void setB(B b){

    }
}
```

可以拆开：

第一步：

```
new A()
```

对象已经存在。

第二步：

```
setB(B)
```

注入依赖。

所以：

```
A对象
 ↓
可以提前暴露
 ↓
解决循环依赖
```

---

# 八、构造器注入和Setter注入区别

## 构造器注入

```java
private final UserMapper mapper;

public UserService(UserMapper mapper){
    this.mapper = mapper;
}
```

特点：

- 创建对象时完成依赖注入
- 可以使用 final
- 强制表达依赖关系
- 不容易产生不完整对象


---

## Setter注入

```java
private UserMapper mapper;

@Autowired
public void setMapper(UserMapper mapper){
    this.mapper = mapper;
}
```

特点：

- 实例化后属性填充阶段注入
- 不能使用 final
- 依赖可以被再次修改
- 支持部分循环依赖


---

# 九、关于半成品 Bean

半成品不是：

```
不存在的对象
```

而是：

```
已经执行构造方法

但是：

依赖还没有注入完成
```

例如：

```
A对象

字段:
b = null
```

三级缓存保存的是：

```
如何获得这个对象的早期引用
```

不是：

```
重新创建这个对象的方法
```

---

# 十、核心总结

## 三级缓存关系

```
三级缓存
    |
    | ObjectFactory
    ↓
生成早期引用
    |
    ↓
二级缓存
    |
    ↓
最终初始化完成
    |
    ↓
一级缓存
```

---

## 一句话理解

> Spring 三级缓存不是为了保存 Bean，而是为了在循环依赖发生时，提前暴露一个合适的 Bean 引用；这个引用可能是原始对象，也可能是 AOP 代理对象。

---

## 高频面试回答

问：三级缓存为什么存 ObjectFactory？

答：

> 因为 Spring 需要延迟决定 Bean 的早期引用。如果 Bean 需要 AOP 代理，提前暴露的应该是代理对象而不是原始对象，所以三级缓存保存的是获取早期引用的工厂。

问：为什么构造器循环依赖无法解决？

答：

> 因为构造器注入要求创建 Bean 时就必须拥有全部依赖，Bean 对象还没有实例化完成，无法提前暴露引用给三级缓存。