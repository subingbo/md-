# Go Interface（接口）实现机制总结

## 一、Go 和 Java 最大的区别

### Java：显式实现接口

```java
interface Animal {
    void eat();
}

class Dog implements Animal {
    public void eat() {}
}
```

Java 必须写：

```java
implements Animal
```

编译器才知道 Dog 实现了 Animal。

---

### Go：隐式实现接口

```go
type Animal interface {
    Eat()
}

type Dog struct{}

func (d *Dog) Eat() {}
```

Go **没有**：

```go
implements Animal
```

只要方法满足接口要求，就认为实现了接口。

---

# 二、什么时候检查是否实现接口？

很多人误以为：

```go
type ScorePlugin interface {
    Score()
}

type EnergyScore struct{}

func (e *EnergyScore) Score() {}
```

编译器会立刻检查：

```
EnergyScore
↓

实现了 ScorePlugin
```

**实际上不会。**

到这里为止，编译器根本不会检查。

---

## 只有"使用接口"时才检查

例如：

```go
var p ScorePlugin = &EnergyScore{}
```

或者

```go
func Run(p ScorePlugin) {}

Run(&EnergyScore{})
```

这时编译器才检查：

```
&EnergyScore

↓

是否满足 ScorePlugin？

↓

需要：

Score()

↓

EnergyScore 是否有 Score()？

↓

✔ 有

↓

允许赋值
```

如果没有：

```go
type EnergyScore struct{}
```

没有：

```go
func (e *EnergyScore) Score() {}
```

那么：

```go
var p ScorePlugin = &EnergyScore{}
```

编译直接报错：

```
*EnergyScore does not implement ScorePlugin
(missing method Score)
```

---

# 三、编译器到底检查什么？

编译器检查的是：

**这个具体类型(Method Set)是否满足接口。**

例如：

```go
type ScorePlugin interface {
    Score()
}
```

EnergyScore：

```go
type EnergyScore struct{}

func (e *EnergyScore) Score() {}
```

编译器检查：

```
EnergyScore

↓

自己的方法：

Score()

↓

接口需要：

Score()

↓

✔ 满足
```

---

# 四、不会去全局搜索同名方法

很多人容易误解：

项目里有多个：

```go
Score()
```

是不是会冲突？

例如：

```go
type EnergyScore struct{}

func (e *EnergyScore) Score() {}
```

还有：

```go
type GPUScore struct{}

func (g *GPUScore) Score() {}
```

还有：

```go
type CPUBalance struct{}

func (c *CPUBalance) Score() {}
```

编译器检查：

```
检查 EnergyScore：

↓

只看：

EnergyScore 自己的方法

↓

不会看：

GPUScore

不会看：

CPUBalance
```

所以：

```
EnergyScore

↓

有 Score()

↓

实现 ScorePlugin
```

```
GPUScore

↓

有 Score()

↓

也实现 ScorePlugin
```

多个结构体都可以实现同一个接口。

互不影响。

---

# 五、什么是 Method Set（方法集）

Method Set（方法集）就是：

> 一个类型拥有的所有方法。

例如：

```go
type EnergyScore struct{}

func (e *EnergyScore) Name() string {}

func (e *EnergyScore) Score() {}

func (e *EnergyScore) NormalizeScore() {}
```

那么：

```
EnergyScore 的 Method Set

↓

Name()

Score()

NormalizeScore()
```

编译器检查接口时，就是查看：

```
Method Set

↓

是否包含接口要求的方法
```

---

# 六、为什么同名方法不会冲突？

因为方法属于类型。

例如：

```go
type Dog struct{}

func (d *Dog) Eat() {}
```

还有：

```go
type Cat struct{}

func (c *Cat) Eat() {}
```

真正的方法分别是：

```
(*Dog).Eat()

(*Cat).Eat()
```

这是两个完全不同的方法。

不是全局只有一个：

```
Eat()
```

---

# 七、Kubernetes Scheduler 为什么能识别插件？

例如：

```go
type ScorePlugin interface {
    Plugin

    Score(...)
}
```

你的插件：

```go
type EnergyScore struct{}
```

实现：

```go
func (e *EnergyScore) Name() string {}

func (e *EnergyScore) Score(...) {}
```

Framework 创建插件：

```go
plugin := &EnergyScore{}
```

然后：

```go
if sp, ok := plugin.(framework.ScorePlugin); ok {
    ...
}
```

Go 检查：

```
EnergyScore

↓

有没有：

Name()

Score()

↓

✔ 有

↓

类型断言成功

↓

加入 scorePlugins
```

如果没有实现：

```
Score()

↓

类型断言失败

↓

不会加入 scorePlugins
```

---

# 八、整个过程

```
写接口

↓

type ScorePlugin interface {
    Score()
}

↓

写插件

↓

type EnergyScore struct{}

↓

实现：

Score()

↓

创建对象：

&EnergyScore{}

↓

把对象当成 ScorePlugin 使用
（赋值、传参、类型断言）

↓

Go 编译器/运行时检查：

EnergyScore 的 Method Set

↓

是否包含：

Score()

↓

✔ 是

↓

EnergyScore 实现了 ScorePlugin
```

---

# 九、核心结论（牢记）

1. Go 没有 `implements` 关键字。
2. 接口实现是**隐式**的。
3. 编译器**不会主动检查**一个类型是否实现某个接口。
4. **只有当一个对象被当作接口类型使用时**（赋值、传参、返回、类型断言等），Go 才会检查是否满足接口。
5. 检查的是**这个具体类型自己的方法集（Method Set）**，不是全局搜索同名方法。
6. 不同结构体可以拥有同名方法，因为方法属于具体类型，不会冲突。
7. Kubernetes Scheduler Framework 正是利用这一机制，通过接口类型和类型断言识别一个插件实现了哪些扩展点（如 `ScorePlugin`、`FilterPlugin` 等）。