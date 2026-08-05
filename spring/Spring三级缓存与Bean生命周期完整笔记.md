# Spring三级缓存与Bean生命周期完整笔记

## 一、Starter自动配置

Starter不是Bean集合，而是提供自动配置能力。

引入Starter后：

```
Spring启动
 ↓
@EnableAutoConfiguration
 ↓
读取 META-INF/spring/AutoConfiguration.imports
 ↓
找到自动配置类
 ↓
@Configuration + @Bean创建Bean
 ↓
放入IOC容器
```

META-INF作用：

告诉Spring这个jar里面有哪些自动配置类。

---

## 二、Bean与容器

Bean：

被Spring管理的对象。

IOC容器：

管理Bean生命周期的地方。

负责：

- 创建Bean
- 依赖注入
- 初始化
- AOP代理
- 销毁

---

## 三、Bean生命周期

```
实例化(new)

↓

属性注入

↓

Aware回调

↓

BeanPostProcessor.beforeInitialization

↓

初始化

@PostConstruct
InitializingBean
init-method

↓

BeanPostProcessor.afterInitialization

↓

一级缓存
```

---

## 四、初始化方式

执行顺序：

```
@PostConstruct

↓

InitializingBean.afterPropertiesSet()

↓

init-method
```

init-method适合第三方类：

```java
@Bean(initMethod="start")
public RedisClient redisClient(){
    return new RedisClient();
}
```

---

## 五、BeanPostProcessor

普通BeanPostProcessor：

```
beforeInitialization

afterInitialization
```

AOP代理通常在：

```
postProcessAfterInitialization()
```

完成。

---

## 六、SmartInstantiationAwareBeanPostProcessor

继承关系：

```
BeanPostProcessor

↓

InstantiationAwareBeanPostProcessor

↓

SmartInstantiationAwareBeanPostProcessor
```

它提供更早阶段的Bean干预。

核心方法：

```java
getEarlyBeanReference()
```

用于循环依赖时提前暴露引用。

---

## 七、三级缓存

### 一级缓存 singletonObjects

保存：

完整Bean。

应用运行期间@Autowired获取的就是这里。

---

### 二级缓存 earlySingletonObjects

保存：

早期引用。

Bean已经实例化，但没有完成完整初始化。

可能保存：

- 原对象
- 代理对象

---

### 三级缓存 singletonFactories

保存：

ObjectFactory。

作用：

创建早期引用。

不是保存Bean。

---

## 八、循环依赖流程

A依赖B：

```
A -> B -> A
```

创建A：

```
new A()

↓

三级缓存:

A -> ObjectFactory
```

创建B。

B需要A：

```
一级缓存 没有

二级缓存 没有

三级缓存 找到工厂
```

调用：

```
ObjectFactory.getObject()
```

进入：

```
getEarlyBeanReference()
```

---

## 九、AOP代理提前创建

调用：

```
SmartInstantiationAwareBeanPostProcessor
```

其中：

```
AbstractAutoProxyCreator
```

判断是否需要代理。

如果需要：

```
A原对象

↓

A代理对象
```

返回代理对象。

放入二级缓存。

---

## 十、为什么三级缓存存工厂

因为需要延迟决定：

返回：

- 原对象
- 代理对象

三级缓存：

```
A -> ObjectFactory
```

真正需要A：

```
调用工厂

↓

判断代理

↓

返回正确引用
```

---

## 十一、单例保证

单例不是三级缓存保证。

真正依靠：

```
singletonObjects 一级缓存
```

创建完成后：

```
A -> 最终Bean
```

所有依赖A的地方拿同一个引用。

三级缓存里的工厂：

不是：

```
new A()
```

而是：

```
返回当前正在创建的A的早期引用
```

---

## 十二、缓存锁

Spring使用：

```java
synchronized(singletonObjects)
```

singletonObjects只是锁对象。

保护：

- 一级缓存
- 二级缓存
- 三级缓存
- Bean创建状态

之间的一致性。

---

## 总结

```
Starter
 ↓
META-INF找到自动配置
 ↓
@Configuration创建Bean
 ↓
Bean生命周期
 ↓
SmartInstantiationAwareBeanPostProcessor
 ↓
getEarlyBeanReference
 ↓
三级缓存解决循环依赖
 ↓
一级缓存保存最终单例Bean
```
