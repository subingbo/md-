# Spring 依赖注入方式总结

## 1. 依赖注入（DI）

### 核心思想

对象不自己创建依赖对象，而是由 IoC 容器创建并注入。

传统方式：

```java
class UserService {

    private UserRepository repository;

    public UserService(){
        this.repository = new MySQLUserRepository();
    }
}
```

问题：

- Service 和具体实现强耦合
- 更换实现需要修改业务代码

依赖注入：

```java
class UserService {

    private UserRepository repository;

    public UserService(UserRepository repository){
        this.repository = repository;
    }
}
```

Service 只依赖接口：

```
UserService
      |
      ↓
UserRepository(抽象)
      ↑
      |
MySQLUserRepository
```

------

# 2. 三种注入方式

## ① 构造器注入（推荐）

```java
@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository){
        this.repository = repository;
    }
}
```

Spring 创建 Bean：

```
找到 UserRepository Bean
        ↓
调用构造方法
        ↓
创建 UserService
        ↓
放入 IoC 容器
```

类似：

```java
new UserService(repository);
```

### 优点

- 依赖是创建对象的必要条件
- 对象创建后一定完整
- 可以使用 final
- 方便单元测试
- 依赖关系明确

------

## ② Setter 注入

```java
@Service
public class UserService {

    private UserRepository repository;

    @Autowired
    public void setRepository(UserRepository repository){
        this.repository = repository;
    }
}
```

Spring 流程：

```
创建 UserService
        ↓
调用 setter
        ↓
注入依赖
```

类似：

```java
UserService service = new UserService();

service.setRepository(repository);
```

### 优点

- 可以注入可选依赖
- 可以后续修改依赖
- 可以在 setter 中增加校验逻辑

### 缺点

- 对象创建时可能处于不完整状态

------

## ③ 字段注入

```java
@Service
public class UserService {

    @Autowired
    private UserRepository repository;

}
```

Spring 类似：

```
创建对象
    ↓
反射修改字段
    ↓
完成注入
```

### 缺点

- 依赖隐藏在字段中
- 破坏封装
- 不方便脱离 Spring 测试
- 无法使用 final
- 对象可能先创建后补依赖

------

# 3. 构造器注入为什么更推荐？

## 不是因为不用 new

Spring 创建 Bean 时仍然需要实例化对象。

区别：

字段注入：

```
创建对象
    ↓
注入依赖
```

构造器注入：

```
准备依赖
    ↓
创建对象
```

------

## 核心优势：保证对象完整

字段注入：

```java
UserService service = new UserService();
```

对象可以存在：

```
UserService
      |
      ↓
repository = null
```

后续调用：

```java
service.save();
```

才发现问题。

------

构造器注入：

```java
new UserService(repository);
```

没有依赖：

```java
new UserService();
```

直接无法创建。

保证：

```
UserService存在
        =>
一定拥有Repository
```

------

# 4. "依赖显式"是什么意思？

不是说字段注入看不到依赖。

字段注入：

```java
@Autowired
private UserRepository repository;
```

人当然能看到。

真正含义：

## 字段注入

依赖声明在对象内部：

```
UserService
    |
    ↓
@Autowired字段
```

创建入口：

```java
new UserService()
```

不知道创建时需要什么。

------

## 构造器注入

依赖声明在创建入口：

```java
public UserService(UserRepository repository)
```

创建对象必须提供：

```java
new UserService(repository)
```

依赖成为对象创建条件。

------

# 5. 和 IoC 的关系

## 依赖倒置

解决：

> 应该依赖什么？

答案：

```
依赖抽象接口
```

例如：

```
UserService
      |
      ↓
UserRepository
      ↑
      |
MySQLRepository
```

------

## IoC

解决：

> 对象由谁创建？

答案：

```
Spring 容器创建并管理
```

Spring：

```
创建 MySQLRepository Bean

        ↓

注入 UserService
```

------

关系：

```
依赖倒置
    ↓
业务代码依赖接口

IoC
    ↓
Spring负责创建和注入实现
```

------

# 6. 三种注入方式比较

| 方式       | 特点                         | 推荐  |
| ---------- | ---------------------------- | ----- |
| 构造器注入 | 创建时注入，依赖必须存在     | ⭐⭐⭐⭐⭐ |
| Setter注入 | 创建后注入，可修改           | ⭐⭐⭐   |
| 字段注入   | 反射修改字段，简单但隐藏依赖 | ⭐⭐    |

------

# 一句话总结

> 构造器注入不是为了避免 new，而是让依赖成为对象创建的必要条件；Spring 负责创建具体实现并注入，业务类只依赖抽象接口，从而降低耦合。

```

```