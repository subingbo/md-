# Java 中 equals 与 hashCode 契约及 HashMap 行为详解

## 一、Java 官方契约

Java 明确规定了 `equals` 和 `hashCode` 的**黄金契约**：

1. **如果两个对象通过 `equals()` 比较返回 `true`，则它们的 `hashCode()` 必须相等。**
2. 如果两个对象 `hashCode()` 相等，`equals()` 不一定返回 `true`（哈希冲突是允许的）。
3. `hashCode()` 在程序运行期间对同一对象应保持一致（除非参与计算的字段被修改）。

**违反第 1 条 = 违反契约，会导致 HashMap/HashSet 行为完全错乱。**

------

## 二、HashMap 判断 key 是否相同的机制

HashMap 判断两个 key 是否相同，分**两步**：

```
第一步：用 hashCode 定位桶索引（快速筛选）
第二步：在桶内遍历链表/红黑树，先比 hash，再比 equals（精确判断）
```

### 源码关键逻辑（JDK 8 `putVal` 简化版）

```java
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        // 遍历到尾部，没找到相同的，插入新节点
        p.next = newNode(hash, key, value, null);
        break;
    }
    // 关键：短路 && 运算
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k)))) {
        // 找到相同的 key，跳出循环，覆盖旧值
        break;
    }
    p = e; // 继续下一个节点
}
```

**核心要点：**

- 先比较 `e.hash == hash`，不相等则**短路跳过**，`equals` 根本不会被调用
- 只有 hash 相等，才会继续比较 `==` 和 `equals`
- `get` 操作只查 hashCode 算出来的那一个桶，**不会遍历所有桶**

------

## 三、四种重写情况对比

| 重写情况          | 违反契约？ | 桶定位       | 桶内比较   | 结果                            |
| ----------------- | ---------- | ------------ | ---------- | ------------------------------- |
| 都不重写          | 否         | 按内存地址   | 按内存地址 | 一致（默认行为）                |
| **只重写 equals** | **是**     | 大概率不同桶 | 没机会调用 | `get` 返回 null、`Set` 去重失效 |
| 只重写 hashCode   | 否         | 同桶         | 地址不同   | `get` 返回 null、`Set` 去重失效 |
| 两个都重写        | 否         | 同桶         | 内容相同   | 正常工作                        |

------

## 四、只重写 `equals`，不重写 `hashCode`（最严重）

### 问题本质

`equals` 被重写为按内容比较，但 `hashCode` 仍是默认的（基于内存地址）。两个内容相同的对象 `hashCode` 不同，导致：

- 大概率被分配到**不同的桶**，`get` 时去错误的桶查找，返回 `null`
- 小概率哈希冲突到**同一个桶**，但 hash 不同导致 `equals` 被短路跳过，重复存储

### 三种子情况

假设 `p1.equals(p2) == true`，但 `p1.hashCode() != p2.hashCode()`：

#### 情况一：hashCode 恰好相同（概率极低）

```
p1.hashCode() == p2.hashCode()  →  同桶
equals 被调用，返回 true
结果：覆盖旧值，行为正常 （纯属运气）
```

#### 情况二：hashCode 不同但冲突到同桶（较低概率）

```
p1.hashCode() = 12345  →  12345 & 15 = 5  →  桶索引5
p2.hashCode() = 67890  →  67890 & 15 = 5  →  桶索引5（冲突）

桶内比较：
  e.hash(12345) != hash(67890)  →  短路跳过
  equals 不会被调用

结果：p2 被当作新 key 插入链表，重复存储 
```

#### 情况三：hashCode 不同且不同桶（最常见）

```
p1.hashCode() = 12345  →  桶索引5
p2.hashCode() = 67890  →  桶索引10

结果：p2 被放到完全不同的桶，get(p2) 去桶10找，返回 null 
```

### 代码演示

```java
class User {
    String name;
    public User(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(name, user.name);
    }
    // 没重写 hashCode！
}

User u1 = new User("张三");
User u2 = new User("张三");

System.out.println(u1.equals(u2));        // true 
System.out.println(u1.hashCode() == u2.hashCode()); // false 

Map<User, Integer> map = new HashMap<>();
map.put(u1, 100);
System.out.println(map.get(u2));          // null 

Set<User> set = new HashSet<>();
set.add(u1);
set.add(u2);
System.out.println(set.size());           // 2  去重失败
```

------

## 五、只重写 `hashCode`，不重写 `equals`

### 问题本质

`hashCode` 被重写为按内容计算，但 `equals` 仍是默认的 `Object.equals()`（比较内存地址 `==`）。两个内容相同的对象能定位到同一个桶，但桶内比较时 `equals` 返回 `false`。

### 完整流程

```java
class User {
    String name;
    public User(String name) { this.name = name; }

    @Override
    public int hashCode() {
        return name.hashCode(); // 重写了
    }
    // 没重写 equals，默认比较内存地址
}

User u1 = new User("张三");
User u2 = new User("张三");

System.out.println(u1.hashCode() == u2.hashCode()); // true 
System.out.println(u1.equals(u2));                   // false 
```

#### put(u1, 100)

```
u1.hashCode() = "张三".hashCode() = 55555
桶索引：55555 & 15 = 3
存入桶索引3
```

#### get(u2)

```
u2.hashCode() = "张三".hashCode() = 55555
桶索引：55555 & 15 = 3
去桶索引3查找 → 找到 u1

桶内比较：
  e.hash == hash  →  55555 == 55555  →  true 
  (k = e.key) == key  →  u1 == u2  →  false 
  key.equals(k)  →  u2.equals(u1)  →  false （比较内存地址）

结果：整个条件为 false，认为不是同一个 key，返回 null 
```

### 结果

- **`get(u2)`**：返回 `null` 
- **`put(u2, 200)`**：被认为新 key，重复插入 
- **`HashSet` 去重**：`set.add(u1); set.add(u2);` → size 为 2

------

## 六、正确做法：两个一起重写

**参与 `equals` 比较的字段，必须全部参与 `hashCode` 计算。**

```java
class User {
    String name;
    int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age); // 和 equals 用相同的字段
    }
}
```

### 验证

```java
User u1 = new User("张三", 25);
User u2 = new User("张三", 25);

System.out.println(u1.equals(u2));        // true 
System.out.println(u1.hashCode() == u2.hashCode()); // true 

Map<User, Integer> map = new HashMap<>();
map.put(u1, 100);
System.out.println(map.get(u2));          // 100 

Set<User> set = new HashSet<>();
set.add(u1);
set.add(u2);
System.out.println(set.size());           // 1  去重成功
```

------

## 七、实用建议

### 1. 手动编写

使用 `Objects.equals()` 和 `Objects.hash()`，注意 null 安全：

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    User user = (User) o;
    return age == user.age && Objects.equals(name, user.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

### 2. IDE 自动生成

- **IntelliJ IDEA**：`Alt + Insert` → 选择 `equals() and hashCode()`
- **Eclipse**：`Source` → `Generate hashCode() and equals()`

### 3. Lombok

使用 `@EqualsAndHashCode` 注解：

```java
@Data
@EqualsAndHashCode
class User {
    private String name;
    private int age;
}
```

**继承场景**需加 `callSuper = true`：

```java
@EqualsAndHashCode(callSuper = true)
class Admin extends User {
    private String role;
}
```

### 4. Java 16+ record

record 自动生成正确的 `equals` 和 `hashCode`，推荐用于纯数据载体：

```java
record User(String name, int age) {}
// 自动生成正确的 equals 和 hashCode
```

------

## 八、常见面试题

### Q1：为什么 HashMap 要先比 hash 再比 equals？

**答：** 性能优化。`int` 比较只需 1 个时钟周期，而 `equals` 调用可能需要逐个字段比较，耗时几十到几百个时钟周期。先比 hash 可以快速排除绝大多数不相等的对象，避免昂贵的 `equals` 调用。

### Q2：两个对象 `hashCode` 相同，`equals` 一定返回 `true` 吗？

**答：** 不一定。`hashCode` 相同但 `equals` 返回 `false` 就是**哈希冲突**，HashMap 通过链表/红黑树处理这种情况。契约只要求"equals 相等则 hashCode 相等"，反过来不要求。

### Q3：为什么 String、Integer 等类可以直接作为 HashMap 的 key？

**答：** 因为它们已经正确重写了 `equals` 和 `hashCode`，且都是不可变对象（immutable），`hashCode` 在对象生命周期内不会改变，符合 HashMap 对 key 的要求。

### Q4：如果对象作为 HashMap 的 key，需要注意什么？

**答：**

1. 必须正确重写 `equals` 和 `hashCode`
2. 最好使用不可变对象（如 String、Integer），或确保参与 `hashCode` 计算的字段在放入 map 后不被修改
3. 如果字段被修改导致 `hashCode` 改变，`get` 时将找不到该 key

------

## 九、总结

> **`equals` 和 `hashCode` 必须一起重写，且参与 `equals` 比较的字段必须全部参与 `hashCode` 计算。只重写其中一个，HashMap/HashSet 在所有情况下都无法正常工作。**

| 场景            | 结论                                               |
| --------------- | -------------------------------------------------- |
| 都不重写        | 按内存地址比较，符合默认行为，无问题               |
| 只重写 equals   | 违反契约，`get` 找不到、`Set` 去重失效，**最严重** |
| 只重写 hashCode | 不违反契约，但 `equals` 仍比较地址，去重失效       |
| 两个都重写      | 正常工作，**唯一正确做法**                         |

------

笔记整理完了，要不要接着看其他集合类的坑？比如 ArrayList 扩容机制、CopyOnWriteArrayList 的适用场景这些。