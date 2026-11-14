# Java 中 `equals` 与 `hashCode` 契约及 HashMap 行为详解（完善版）

> 适用：Java 8+，重点以 **JDK 8 `HashMap`** 的实现为例。  
> 核心记忆：**`hashCode` 用来定位和筛选，`==` / `equals` 用来确认 Key，找到 `Node` 后读取 `value`。**

## 一、先建立完整模型：HashMap 到底存什么？

`HashMap<K, V>` 的底层主体是 `Node<K, V>[] table`，数组的每个位置称为一个**桶（bucket）**。桶中存的是节点引用，不是单独的 Key。

```java
// JDK 8 的 Node 结构（省略接口及其他代码）
static class Node<K, V> {
    final int hash;       // 经 HashMap 扰动后的 hash
    final K key;          // 键
    V value;              // 值
    Node<K, V> next;      // 链表中的下一个节点
}
```

```text
HashMap.table（Node 数组）
    [0] -> null
    [1] -> Node(hash1, K1, V1) -> Node(hash2, K2, V2) -> null
    [2] -> Node(hash3, K3, V3) -> null
    [3] -> null
    ...
```

- **`put(key, value)`**：先找桶；桶内找到相同 Key 就更新 Value，否则插入新节点。
- **`get(key)`**：先找桶；桶内找到相同 Key 就返回对应 `Node.value`。
- **`remove(key)` / `containsKey(key)`**：同样先计算哈希、定位桶，再识别 Key。
- **哈希冲突**：不同 Key 落在同一桶。JDK 8 中，桶内可以是链表；达到树化条件时可以转换为红黑树。

### 三个经常混淆的概念

| 概念 | 表达式或来源 | 作用 |
| --- | --- | --- |
| 原始哈希值 | `key.hashCode()` | Key 提供的 `int` 哈希值 |
| 扰动后的 hash | `h ^ (h >>> 16)` | 将高位信息混入低位，保存到 `Node.hash` |
| 桶下标 index | `(table.length - 1) & hash` | 决定访问 `table[index]` |

**注意：`hashCode` 相同必然得到相同的扰动后 hash（在同一容量下也必然同桶）；但同桶不意味着 hash 相同，更不意味着 Key 相等。**

---

## 二、Java 官方契约：`equals()` 与 `hashCode()` 的关系

### 1. `equals()` 契约

在合理的对象相等关系中，`equals` 应满足：

1. **自反性**：`x.equals(x)` 为 `true`。
2. **对称性**：`x.equals(y)` 和 `y.equals(x)` 结果相同。
3. **传递性**：若 `x.equals(y)`、`y.equals(z)` 都为 `true`，则 `x.equals(z)` 也为 `true`。
4. **一致性**：参与比较的信息没变时，重复比较应得到相同结果。
5. **非空性**：`x.equals(null)` 为 `false`（`x` 非 null）。

### 2. `hashCode()` 契约（重点）

**相等的对象必须有相等的哈希值：**

```text
x.equals(y) == true  =>  x.hashCode() == y.hashCode()
```

**反过来不成立：**

```text
x.hashCode() == y.hashCode()  不能推出  x.equals(y) == true
```

此外，对同一对象，在参与 `equals` 比较的信息没有改变时，重复调用 `hashCode()` 应返回相同的整数。

> `hashCode()` **不是唯一 ID**，也**不保证等于内存地址**。`Object` 默认的 `equals()` 采用对象身份比较（`==`）；其默认 `hashCode()` 的具体生成机制由 JVM 实现决定。

### 3. 两个公式记住就够了

```text
equals 为 true  →  hashCode 必须相同
hashCode 相同   →  equals 可能 true，也可能 false
```

因此，**只比较 hash 不能识别真正相同的 Key；只比较 equals 虽然能做精确判断，但会失去便宜的哈希预筛选。**

---

## 三、从 `get(key)` 看到返回 Value 的完整流程

假设：

```java
Map<String, Integer> map = new HashMap<>();
map.put("name", 100);
Integer value = map.get("name");
```

### 第一步：计算扰动后的 hash

JDK 8 简化源码：

```java
static final int hash(Object key) {
    int h;
    return key == null ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里 `^` 是**按位异或**，`>>>` 是**无符号右移**。

### 第二步：根据 hash 找桶

```java
int index = (table.length - 1) & hash;
Node<K, V> first = table[index];
```

例如，数组容量为 16、扰动后 hash 为 19：

```text
容量 n = 16        = 0001 0000
n - 1 = 15        = 0000 1111
hash  = 19        = 0001 0011
                       &
                         ---------
index                = 0000 0011 = 3
```

所以访问的是 `table[3]`。容量为 2 的幂时，`(n - 1) & hash` 等价于对相应的非负下标范围取模映射，比一般除法取余运算更适合这里的实现。

### 第三步：在桶中确认 Key

以下是**链表场景下的教学简化版**，省略红黑树分支、初始化及异常细节：

```java
V getSimplified(Object key) {
    int h = hash(key);                         // 扰动后的 hash
    int index = (table.length - 1) & h;      // 计算桶下标
    Node<K, V> node = table[index];

    while (node != null) {
        if (node.hash == h &&
            (node.key == key ||
             (key != null && key.equals(node.key)))) {
            return node.value;               // 找到 Node，取出 Value
        }
        node = node.next;                     // 本节点不匹配，检查下一个
    }
    return null;                              // 没找到
}
```

**判断分三层：**

1. `node.hash == h`：先比较**保存的扰动后 hash**，快速筛掉不匹配节点。
2. `node.key == key`：如果就是同一个对象，直接命中。
3. `key.equals(node.key)`：不是同一个对象，再判断**逻辑相等**。

**重点：**`node.hash` 已在插入时计算并保存。查询时不需要对每个旧节点重新调用 `node.key.hashCode()`。

> `get()` 找不到 Key 返回 `null`，但 Value 本身也可能是 `null`。若需要区分“没有这个 Key”和“Key 存在但 Value 为 null”，可以用 `containsKey(key)`。

### 第四步：`put` 为什么也要判断这些？

插入时如果找到了逻辑相同的 Key，就**更新 Value，不增加新的映射数量**；否则在该桶插入新节点。

```java
map.put("name", 100);
map.put(new String("name"), 200);

System.out.println(map.size());        // 1
System.out.println(map.get("name"));   // 200
```

两个 `String` 可以是不同对象，但内容相同，`equals()` 为 `true`，因此视为同一个 Key。

---

## 四、为什么既不能只比较 hash，也没必要只比较 equals？

### 1. 只比较 hash：**不正确**

Java 存在经典的**不同字符串、相同 hashCode**：

```java
String a = "Aa";
String b = "BB";

System.out.println(a.hashCode());     // 2112
System.out.println(b.hashCode());     // 2112
System.out.println(a.equals(b));      // false
```

```java
map.put("Aa", 100);
map.put("BB", 200);
```

假如 HashMap 只比较 hash，就会误把 `"Aa"` 和 `"BB"` 当成同一个 Key，错误地覆盖 100。**所以即使 hash 相同，也必须通过 `==` / `equals` 进一步确认。**

### 2. 桶内只比较 equals：**逻辑上可行，但通常不够高效**

在 `equals` / `hashCode` 契约被正确遵守的前提下，**已经定位到桶之后**，可以逐个执行 `equals` 找目标，但可能浪费调用成本。

```text
table[3]
  -> Node(hash=19, key=A)
  -> Node(hash=35, key=B)
  -> Node(hash=51, key=C)
```

查找 hash=51 的 C：

| 方案 | A | B | C |
| --- | --- | --- | --- |
| 只调用 equals | equals | equals | equals |
| 先比较 hash | 19 != 51，跳过 | 35 != 51，跳过 | hash 相同，再 equals |

比较两个 `int` 通常比执行复杂对象的 `equals()` 更便宜。但**不应机械地声称一次 int 比较必定只需一个时钟周期，或 equals 固定耗时多少周期**；实际取决于实现与运行环境。

### 3. 三类“冲突”要分清

| 情形 | 是否同桶 | 是否调用 equals | 能否认定为同一个 Key |
| --- | --- | --- | --- |
| 扰动后 hash 不同、桶下标不同 | 否 | 不会 | 否 |
| 扰动后 hash 不同、桶下标相同 | 是 | 节点 hash 不同，通常跳过 | 否 |
| 扰动后 hash 相同 | 是 | 不同引用时需要比较 | 取决于 equals |

---

## 五、四种重写情况：结果为什么不同？

**前提：业务希望“两个 `User` 的 name 相同，就认为是同一个 Key”。**

| 实现方式 | 是否遵守契约 | 业务上能否按 name 识别相同 Key | 典型结果 |
| --- | --- | --- | --- |
| 两个都不重写 | **遵守**默认身份相等契约 | 不能 | 只有同一实例作为 Key 才匹配 |
| 只重写 `equals` | **可能违反**；当相等对象 hashCode 不同时就违反 | **不可靠** | 可能无法用等价实例 `get`，去重也可能失效 |
| 只重写 `hashCode` | 通常仍遵守默认身份相等契约 | 不能 | hash 相同，但不同实例的 equals 为 false |
| 按一致语义正确重写两者 | 遵守 | 可以 | 可用等价实例查找、覆盖、去重 |

> **不是“所有类必须重写两个方法”。**如果就是要按对象身份区分 Key，两个都不重写完全合法。改变了 `equals` 的语义后，才需要特别确保 `hashCode` 保持一致。

### 情况 A：两个都不重写

`Object.equals()` 默认等价于 `==`，`Object.hashCode()` 与对象身份相容：

```java
User u1 = new User("张三");
User u2 = new User("张三");
map.put(u1, 100);

map.get(u1); // 100：同一个对象
map.get(u2); // null：另一实例不是相同 Key（假设未重写 equals）
```

这里是**正常的身份语义**，不是 HashMap 损坏。

### 情况 B：只重写 equals，不重写 hashCode

例如 `equals` 用 `name` 比较，但继承了 `Object.hashCode()`：

```text
u1.equals(u2) == true
u1.hashCode() 和 u2.hashCode() 却没有被保证相等
```

如果两个原始 hashCode 不同，那么扰动后的 hash 也不同：

- **不同桶**：查询根本不会访问到 `u1` 所在的桶。
- **同一个桶**：节点的完整 hash 不相同，比较在 `equals` 之前就失败。

例如以下数值是为讲解而**假设**的，并非 JVM 对 `User` 的真实固定输出：

```text
数组容量 = 16
u1 的 hash = 19  -> 19 & 15 = 3
u2 的 hash = 35  -> 35 & 15 = 3

虽然同桶，但 19 != 35，equals 根本不会被调用。
```

**边界说明：**若默认哈希值恰好相同，单次查找有可能成功；所以不应把示例中的 `get(u2) == null` 或 `set.size() == 2` 说成在任何 JVM 上都绝对成立。问题在于**行为没有受到契约保障**。

### 情况 C：只重写 hashCode，不重写 equals

```java
@Override
public int hashCode() {
    return name.hashCode();
}
// equals 没有重写
```

此时两个 name 相同的对象必定 hash 相同，但 `Object.equals()` 仍然按身份比较：

```text
u1.hashCode() == u2.hashCode()   -> true
u1 == u2                       -> false
u1.equals(u2)                  -> false
```

因此两者仍是不同的 Key。**这不必然违反 Java 契约，但违背了“按 name 相等”的业务预期。**

### 情况 D：两个一起正确重写

```java
import java.util.Objects;

class User {
    final String name;
    final int age;

    User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User other = (User) o;
        return age == other.age && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

```java
User u1 = new User("张三", 25);
User u2 = new User("张三", 25);

Map<User, Integer> map = new HashMap<>();
map.put(u1, 100);
map.put(u2, 200);

System.out.println(map.size());      // 1
System.out.println(map.get(u1));     // 200
System.out.println(map.get(u2));     // 200
```

**规则的准确表述是：`equals` 认定相等的任意两个对象，`hashCode` 必须计算出相同结果。**常见、安全的做法是让两者使用相同的一组业务字段；但“所有参与 equals 的字段必须逐一参与 hashCode”**不是数学上的硬性要求**（例如所有对象返回固定 hash 也符合契约，但性能可能很差）。

---

## 六、为什么 HashMap 容量是 2 的幂、扩容为什么乘 2？

JDK 8 常见默认设置：

| 属性 | 值 |
| --- | --- |
| 默认初始容量（实际首次分配时） | 16 |
| 默认负载因子 | 0.75 |
| 初始扩容阈值 | `16 × 0.75 = 12` |
| 第 13 个新映射插入后 | 通常扩容至 32 |
| 扩容策略 | 容量通常翻倍（有最大容量等特殊边界） |

### 1. 容量为 2 的幂，方便求下标

```java
index = (n - 1) & hash;
```

例如 `n = 16`，则 `n - 1 = 15 = 0000 1111`，相当于取 hash 的低 4 位作为桶索引。

### 2. 翻倍扩容，原元素只有两个可能的新位置

原容量 16，扩容为 32：

```text
hash = 19：原下标 19 & 15 = 3；新下标 19 & 31 = 19（3 + 16）
hash = 35：原下标 35 & 15 = 3；新下标 35 & 31 = 3 （位置不变）
```

JDK 8 重新分配元素时，可以利用：

```java
if ((node.hash & oldCap) == 0) {
    newIndex = oldIndex;
} else {
    newIndex = oldIndex + oldCap;
}
```

**因为容量乘 2 时，下标仅多考虑一位二进制位。**扩容仍需遍历并迁移相关节点，不能说扩容本身是 O(1)。乘 2 还有减少未来扩容次数的好处，但容量增长过大也会浪费空间，属于时间与空间的折中。

---

## 七、作为 Key 的对象还有哪些坑？

### 1. Key 放进去后，别修改参与相等/哈希计算的字段

```java
class MutableKey {
    String name;
    MutableKey(String name) { this.name = name; }

    @Override public boolean equals(Object o) {
        return o instanceof MutableKey other
            && Objects.equals(name, other.name);
    }
    @Override public int hashCode() { return Objects.hashCode(name); }
}
```

```java
MutableKey key = new MutableKey("A");
map.put(key, 100);
key.name = "B";
map.get(key); // 可能返回 null，因当前 hash 对应的查找位置已改变
```

**更好的做法：**使用 `final` 字段的不可变 Key，或保证放入 HashMap 后参与 `equals/hashCode` 的字段保持不变。

> 上面 `instanceof MutableKey other` 是 Java 16+ 语法；JDK 8 应改成先 `instanceof` 再显式强制类型转换。

### 2. `null` Key 可以用吗？

可以。`HashMap` 允许**一个** `null` Key；它的 hash 视为 0，位于下标 0 对应的桶。多个 `put(null, ...)` 会更新同一映射。

### 3. `HashSet` 与之是什么关系？

`HashSet` 内部使用 `HashMap` 保存元素：Set 元素作为 Map 的 Key，Value 使用占位对象。因此 `HashSet` 去重遵循同样的 `hashCode + equals` 规则。

### 4. JDK 8 红黑树化怎么理解？

桶中链表过长时，达到树化阈值会考虑转成红黑树；但如果表容量较小，优先扩容而不是立即树化。常见阈值：链表树化阈值 8、最小树化容量 64。**无论链表还是树，最终仍需要正确识别 Key。**

---

## 八、IDE / Lombok / record 的实际使用

### IntelliJ IDEA

`Alt + Insert` → `equals() and hashCode()` → 选定参与相等判断的字段 → 生成并检查。

### Lombok

```java
@EqualsAndHashCode
class User {
    private String name;
    private int age;
}
```

- `@Data` **已经包含**生成 `equals/hashCode` 的功能，通常无需再重复写同样的注解。
- 继承场景的 `callSuper = true` **不是无条件必须**：当父类的相等字段确实应参与子类相等判断时才考虑，并需要审查整个继承层次的对称性与一致性。

### Java record（Java 16+）

```java
record User(String name, int age) {}
```

编译器会依据组件生成 `equals()` 和 `hashCode()`，很适合这类数据键。注意 record 是**浅层不可变**：若组件本身引用可变对象，仍要避免它被改动后影响 Key 的相等与哈希语义。

---

## 九、常见面试问答

**Q1：为什么桶内先比 hash，再比 equals？**  
A：先比较整数 hash 可以低成本排除大多数节点，减少可能更贵的 `equals()` 调用；但 hash 相同不代表 Key 相等，所以不能省掉最终的相等判断。

**Q2：为什么不能只比 hashCode？**  
A：允许哈希冲突。例如 `"Aa"` 与 `"BB"` 的 `hashCode()` 都是 2112，却不是相同的 String。

**Q3：为什么不能完全只用 equals？**  
A：**理论上可以**在定位桶后逐个用 equals 判断（前提是契约正确）；但效率通常不如先用节点保存的 hash 筛选。此外，定位桶本身仍需要 hash。

**Q4：只重写 equals 一定 `get()` 失败吗？**  
A：不能说一定。若两个 equals 相同的对象的 hash 不同，就会失败；如果哈希值碰巧相同，可能成功。真正的问题是行为不能保证，且可能违反契约。

**Q5：只重写 hashCode 是不是违反 Java 契约？**  
A：不一定。保留 `Object.equals` 的对象身份语义时，可以遵守契约；但无法按内容判定相等。

**Q6：为什么容量扩为两倍？**  
A：保证容量仍是 2 的幂，便于按位与定位；扩容后的节点要么在原下标，要么在 `原下标 + 旧容量`，且增长适度，避免过频繁扩容。

**Q7：HashMap 的 Key 一定要重写这两个方法吗？**  
A：不一定。若需要按内容相等，就应正确实现；若使用身份语义，继承 Object 默认实现也合法。

---

## 十、一段可以直接运行的验证代码（JDK 8+）

把下面内容保存为 `HashMapContractDemo.java`，执行 `javac HashMapContractDemo.java`，然后执行 `java HashMapContractDemo`：

```java
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

public class HashMapContractDemo {
    static final class User {
        private final String name;

        User(String name) {
            this.name = name;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof User)) return false;
            User other = (User) o;
            return Objects.equals(name, other.name);
        }

        @Override
        public int hashCode() {
            return Objects.hashCode(name);
        }
    }

    public static void main(String[] args) {
        // 相同 hashCode 不等于相同 Key
        System.out.println("Aa".hashCode()); // 2112
        System.out.println("BB".hashCode()); // 2112
        Map<String, Integer> collisions = new HashMap<>();
        collisions.put("Aa", 100);
        collisions.put("BB", 200);
        System.out.println(collisions.size()); // 2

        // 正确重写两个方法后，可用逻辑相等的另一实例查找
        User u1 = new User("张三");
        User u2 = new User("张三");
        Map<User, Integer> map = new HashMap<>();
        map.put(u1, 100);
        map.put(u2, 200);
        System.out.println(map.size());     // 1
        System.out.println(map.get(u1));    // 200
        System.out.println(map.get(u2));    // 200

        Set<User> set = new HashSet<>();
        set.add(u1);
        set.add(u2);
        System.out.println(set.size());      // 1
    }
}
```

---

## 十一、速记总结（考前 30 秒）

```text
get(key)
  |
  v
key.hashCode()                 // 原始哈希
  |
  v
h ^ (h >>> 16)                 // 扰动
  |
  v
(n - 1) & hash                // 找桶 table[index]
  |
  v
node.hash == hash ?            // 先过滤
  |
  +-- 否 -> 下一节点
  |
  +-- 是 -> node.key == key ?   // 同一对象？
                |              
                +-- 是 -> node.value
                |
                +-- 否 -> equals() ?
                             |
                             +-- 是 -> node.value
                             +-- 否 -> 下一节点
```

1. **底层是 Node 数组，Node 才存 `hash、key、value、next`。**
2. **`hashCode` 用来定位与过滤；`equals` 判断逻辑相等；hash 相同不代表 Key 相同。**
3. **两个对象 `equals` 为 true，`hashCode` 必须相同；反过来不成立。**
4. **不同 hash 也可能落入同桶；因此桶内仍要比较完整 hash。**
5. **两个方法都不重写没有错，只是使用对象身份语义；按内容相等时应让两个方法一致。**
6. **扩容 2 倍是为了位运算寻址、简化节点新位置判断，并降低扩容频率。**

### 原笔记中特别需要修正的表述

| 原表述 | 更严谨的表述 |
| --- | --- |
| “默认 hashCode 基于内存地址” | Java 不保证默认 hashCode 等于内存地址 |
| “只重写 equals，get 一定返回 null” | 当相等对象的 hash 不同时匹配会失败；碰巧 hash 相同时可能成功 |
| “参与 equals 的字段必须全部参与 hashCode” | 必须保证 equals 为 true 时 hashCode 相同；字段一致是推荐策略而非唯一方案 |
| “两个都重写是唯一正确做法” | 默认对象身份语义也正确；业务希望按内容相等时才需要相应实现 |
| “只重写其中一个，所有情况下都无法正常工作” | 说法过于绝对；要区分 Java 契约、业务语义和具体对象比较场景 |
| “`@Data` 再写 `@EqualsAndHashCode`、继承时必须 callSuper=true” | `@Data` 本身提供该功能；`callSuper` 应按父类是否参与相等关系决定 |
| “int 比较固定 1 周期、equals 固定几十至几百周期” | 执行成本依赖 JIT、对象字段与 JVM，不能写死 |
| “12345 & 15 = 5；67890 & 15 = 5” | 原例算错；可换成 `19 & 15 = 3` 与 `35 & 15 = 3` |
