# 📘 Java 笔记：（尤其注意第五点toString(),newString()的区别）Group Anagrams + HashMap + String关键点总结

------

# 一、题目核心（字母异位词分组）

## 🎯 目标

将字符串数组中 **字母组成相同（顺序不同）** 的字符串分组

```
eat, tea, ate → 一组
tan, nat → 一组
bat → 一组
```

------

## 🧠 核心思路

> 把“等价的字符串”转成“相同的 key”

### 方法1：排序

```
char[] arr = str.toCharArray();
Arrays.sort(arr);
String key = new String(arr);
```

------

# 二、HashMap 分组核心

```
map.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
```

------

## 📌 等价传统写法

```
List<String> list = map.get(key);
if (list == null) {
    list = new ArrayList<>();
    map.put(key, list);
}
list.add(str);
```

------

## 📌 一句话理解

> 没有就创建分组，有就加入分组

------

# 三、computeIfAbsent 详解

## 方法签名

```
V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)
```

------

## 行为

```
if (!map.containsKey(key)) {
    V value = mappingFunction.apply(key);
    map.put(key, value);
    return value;
}
return map.get(key);
```

------

## 🔑 重点

- 自动 `put`
- lambda 只在 key 不存在时执行
- 返回 value（List）

------

# 四、lambda 表达式关键点

## 写法

```
k -> new ArrayList<>()
```

------

## 为什么没有 return？

👉 单表达式自动返回

等价：

```
k -> {
    return new ArrayList<>();
}
```

------

## 为什么必须写 k？

因为需要匹配：

```
Function<K, V>
```

不能写：

```
() -> new ArrayList<>() ❌
```

------

# 五、最重要：toString vs new String（核心坑）（注意注意注意注意注意）

------

## ❌ 错误写法

```
charArray.toString();
```

输出：

```
[C@1b28cdfa
```

------

## ⚠️ 为什么错？

调用的是：

```
Object.toString()
```

返回：

```
类名@hashCode
```

👉 表示“对象身份”，不是内容

------

## ❗ 本质

| 方法         | 表示       |
| ------------ | ---------- |
| toString()   | 对象是谁   |
| new String() | 内容是什么 |

------

## ❌ 错误结果

```
eat → [C@111
tea → [C@222
```

👉 key 全不同 → 无法分组

------

## ✅ 正确写法

```
String key = new String(charArray);
```

输出：

```
eat → aet
tea → aet
```

------

## ✅ 另一种写法

```
String key = String.valueOf(charArray);
```

------

## ⚠️ 可用但不推荐

```
Arrays.toString(charArray)
```

输出：

```
[a, e, t]
```

问题：

- 有符号
- 不干净
- 性能略差

------

# 六、为什么 String 可以做 key？

因为：

```
String 重写了：
- equals()
- hashCode()
```

------

## 对比

| 类型   | equals | hashCode | 比较方式 |
| ------ | ------ | -------- | -------- |
| String | ✔      | ✔        | 内容     |
| char[] | ❌      | ❌        | 地址     |

------

## 示例

```
new String("aet").equals(new String("aet")) → true
new char[]{'a'} != new char[]{'a'}
```

------

# 七、HashMap.values()

```
hashMap.values()
```

------

## 返回类型

```
Collection<List<String>>
```

👉 不是 List
 👉 不是数组

------

## 转换成 List

```
new ArrayList<>(hashMap.values())
```

------

## 为什么要转？

因为返回值要求：

```
List<List<String>>
```

------

# 八、Map 三大接口

| 方法       | 返回            | 用途          |
| ---------- | --------------- | ------------- |
| keySet()   | Set<K>          | key           |
| values()   | Collection<V>   | value         |
| entrySet() | Set<Entry<K,V>> | ⭐ key + value |

------

## 推荐用 entrySet

```
for (Map.Entry<K, V> e : map.entrySet())
```

------

# 九、HashMap 为什么是 O(1)

## 流程

1. hashCode()
2. 定位数组 index
3. 查桶（链表/红黑树）

------

## 结构

```
数组 + 链表 + 红黑树
```

------

## 优化

- 冲突少 → O(1)
- 链表过长 → 红黑树（O(log n)）

------

# 十、最终完整代码（标准版）

```
public List<List<String>> groupAnagrams(String[] strs) {

    Map<String, List<String>> map = new HashMap<>();

    for (String str : strs) {
        char[] arr = str.toCharArray();
        Arrays.sort(arr);
        String key = new String(arr);

        map.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
    }

    return new ArrayList<>(map.values());
}
```

------

# 🧠 总结（最重要三句话）

1️⃣ **分组核心 = 构造相同 key**
 2️⃣ **数组转字符串必须用 `new String()`**
 3️⃣ **`toString()` 只是对象标识，不是内容**