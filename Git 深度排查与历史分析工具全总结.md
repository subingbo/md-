# Git 深度排查与历史分析工具全总结

> 覆盖：`git grep` / `git blame` / `git log` / `git bisect`
> 适用于：代码搜索、问题定位、历史追踪、Bug 回溯

------

# 一、git grep —— 高效代码搜索

## 1.1 是什么

`git grep` 是 Git 内置的代码搜索工具，用于在 **Git 管理的文件中搜索内容**。

👉 默认搜索范围：

- 当前分支的 **工作区 + 暂存区**
- ❌ 不会搜索历史提交

------

## 1.2 常用命令

### 基础搜索

```bash
git grep "keyword"
```

------

### 指定文件类型

```bash
git grep "keyword" -- "*.java"
```

------

### 显示行号

```bash
git grep -n "keyword"
```

------

### 显示上下文

```bash
git grep -C 3 "keyword"
```

------

### 搜索某个提交

```bash
git grep "keyword" HEAD~3
```

------

### 搜索所有分支

```bash
git grep "keyword" $(git rev-list --all)
```

------

## 1.3 和 grep 区别

| 对比项   | git grep              | grep     |
| -------- | --------------------- | -------- |
| 搜索范围 | Git 管理的文件        | 文件系统 |
| 速度     | 更快（利用 Git 索引） | 较慢     |
| 支持历史 | ✔（指定 commit）      | ❌        |

------

## 1.4 使用场景

- 快速定位代码
- 查找函数/变量定义
- 分析某段逻辑在哪些地方被使用

------

# 二、git blame —— 谁改了这行代码

## 2.1 是什么

用于查看 **某一文件每一行最后一次修改是谁做的**。

------

## 2.2 基础用法

```bash
git blame file.java
```

输出示例：

```
a1b2c3d (Alice 2024-01-01) int x = 10;
```

------

## 2.3 常用参数

### 查看指定行

```bash
git blame -L 10,20 file.java
```

------

### 忽略空白变更

```bash
git blame -w file.java
```

------

### 显示提交时间

```bash
git blame --date=short file.java
```

------

## 2.4 核心理解

👉 `git blame` 只看：

- **最后一次修改该行的提交**

👉 不会展示：

- 完整历史（需要配合 `git log`）

------

## 2.5 典型使用流程

```bash
git blame file.java
→ 找到可疑行的 commit id

git log -p <commit>
→ 查看修改内容
```

------

## 2.6 使用场景

- 查 Bug 责任来源
- 理解代码演进
- 找到相关开发者

------

# 三、git log —— 提交历史分析

## 3.1 是什么

用于查看 Git 的提交历史。

------

## 3.2 基础用法

```bash
git log
```

------

## 3.3 常用参数

### 简洁模式

```bash
git log --oneline
```

------

### 图形化分支

```bash
git log --graph --oneline --all
```

------

### 查看文件历史

```bash
git log file.java
```

------

### 查看修改内容

```bash
git log -p
```

------

### 限制数量

```bash
git log -n 5
```

------

### 按作者过滤

```bash
git log --author="Alice"
```

------

### 按时间过滤

```bash
git log --since="2024-01-01"
```

------

### 搜索提交信息

```bash
git log --grep="fix bug"
```

------

### 搜索代码变更内容

```bash
git log -S "keyword"
```

------

## 3.4 关键能力

| 能力     | 说明            |
| -------- | --------------- |
| 历史追踪 | 查看所有提交    |
| 代码变更 | `-p`            |
| 搜索     | `--grep` / `-S` |
| 可视化   | `--graph`       |

------

## 3.5 高级技巧

### 查看某行历史

```bash
git log -L 10,20:file.java
```

------

### 查看某个 commit 详情

```bash
git show <commit>
```

------

## 3.6 使用场景

- 分析代码演进
- 查找引入 Bug 的提交
- 回溯功能修改

------

# 四、git bisect —— 二分查找 Bug

## 4.1 是什么

用于在提交历史中 **快速定位引入 Bug 的 commit**。

👉 核心思想：二分查找

------

## 4.2 使用流程

### 1️⃣ 开始

```bash
git bisect start
```

------

### 2️⃣ 标记当前版本是坏的

```bash
git bisect bad
```

------

### 3️⃣ 标记一个正常版本

```bash
git bisect good <commit>
```

------

### 4️⃣ Git 自动切换到中间版本

你需要判断：

```bash
git bisect good   # 没问题
git bisect bad    # 有问题
```

------

### 5️⃣ 重复直到找到

最终输出：

```
<commit> is the first bad commit
```

------

### 6️⃣ 结束

```bash
git bisect reset
```

------

## 4.3 自动化测试

```bash
git bisect run ./test.sh
```

👉 返回值：

- 0 → good
- 非0 → bad

------

## 4.4 时间复杂度

👉 O(log n)

| 提交数 | 步数   |
| ------ | ------ |
| 1000   | ~10 次 |
| 10000  | ~14 次 |

------

## 4.5 使用场景

- Bug 回溯
- 性能问题定位
- 回归问题分析

------

# 五、四者对比总结

| 工具       | 作用     | 核心能力     |
| ---------- | -------- | ------------ |
| git grep   | 搜索代码 | 快速定位文本 |
| git blame  | 行级责任 | 谁改了这一行 |
| git log    | 历史分析 | 提交记录     |
| git bisect | Bug定位  | 二分查找     |

------

# 六、组合使用（非常重要🔥）

## 场景：定位一个 Bug

### Step 1：搜索代码

```bash
git grep "bugKeyword"
```

------

### Step 2：查看是谁改的

```bash
git blame file.java
```

------

### Step 3：查看历史

```bash
git log -p file.java
```

------

### Step 4：精确定位

```bash
git bisect
```

------

# 七、核心理解总结

## 1️⃣ git grep

👉 **查“现在”代码在哪**

------

## 2️⃣ git blame

👉 **查“这行是谁最后改的”**

------

## 3️⃣ git log

👉 **查“历史是怎么变的”**

------

## 4️⃣ git bisect

👉 **查“哪次提交引入问题”**

------

# 八、一句话记忆

```
grep   → 找代码
blame  → 找人
log    → 看历史
bisect → 找 Bug
```

------

# 九、最佳实践建议

- 日常开发：
  - `git grep` + IDE 搜索结合使用
- 查问题：
  - `blame → log → bisect`
- 大项目：
  - 优先用 Git 自带工具（比系统 grep 更快）

------

# 十、进阶建议

如果你想再往上走一层，可以深入：

- Git 内部存储模型（commit / tree / blob）
- reflog（找回误操作）
- cherry-pick / rebase（历史修改）

------

如果你想，我可以帮你再补一份：
👉「**Git 排查 Bug 实战流程（真实案例版）**」，把这四个命令串成完整实战。