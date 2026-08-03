# Git 文件恢复与提交操作笔记

## 一、Git 基础流程

完整流程：

```bash
git init

git add .

git commit -m "提交说明"

git branch -M main

git remote add origin 仓库地址

git push -u origin main
```

------

# 二、git add

## 1. 添加单个文件

例如项目：

```
my-project/
├── User.java
├── Order.java
└── src/
    └── Main.java
```

添加 User.java：

```bash
git add User.java
```

或者：

```bash
git add ./User.java
```

两者等价。

------

## 2. 添加子目录文件

当前目录：

```
my-project/
└── src/
    └── Main.java
```

执行：

```bash
git add src/Main.java
```

或者：

```bash
git add ./src/Main.java
```

------

## 3. 添加全部修改

```bash
git add .
```

或者：

```bash
git add ./
```

含义：

添加当前目录以及所有子目录中的修改。

例如：

```
my-project/

├── User.java
├── pom.xml
└── src/
    └── Main.java
```

都会加入暂存区。

------

## 4. 路径规则

Git 使用的是：

> 相对于当前命令执行目录的路径

例如：

当前：

```
D:/project/my-project
```

结构：

```
my-project/
└── src/
    └── Main.java
```

执行：

```bash
git add src/Main.java
```

正确。

如果进入：

```
D:/project/my-project/src
```

再执行：

```bash
git add src/Main.java
```

错误。

因为 Git 会寻找：

```
src/src/Main.java
```

此时应该：

```bash
git add Main.java
```

------

# 三、git commit

commit 是提交到本地 Git 仓库。

例如：

```bash
git commit -m "添加用户模块"
```

注意：

commit ≠ 上传

区别：

```
commit
  ↓
保存到本地 Git 历史

push
  ↓
上传到远程仓库
```

------

# 四、git branch -M main

命令：

```bash
git branch -M main
```

作用：

把当前分支强制改名为 main。

以前 Git 默认：

```
master
```

现在 GitHub 常用：

```
main
```

例如：

执行前：

```
master
 |
 commit
```

执行后：

```
main
 |
 commit
```

------

# 五、restore 和 revert 区别

## 1. git restore

作用：

恢复文件内容。

特点：

- 针对文件
- 不产生新的 commit

例如：

```bash
git restore User.java
```

恢复：

```
当前修改的 User.java

↓

HEAD 中的 User.java
```

------

## 2. git restore --source

格式：

```bash
git restore --source 版本 文件
```

例如：

```bash
git restore --source HEAD^ User.java
```

含义：

从上一次提交中取 User.java，覆盖当前文件。

------

# 六、HEAD 和 HEAD^

## HEAD

表示当前提交。

例如：

```
commit A
    |
commit B
    |
commit C  ← HEAD
```

HEAD：

```
commit C
```

------

## HEAD^

表示上一个提交。

例如：

```
commit A
    |
commit B
    |
commit C ← HEAD
```

那么：

```
HEAD^ = commit B
```

------

## HEAD^^

表示上上个提交：

```
HEAD^^ = HEAD的父提交的父提交
```

------

# 七、恢复某一个已提交文件

场景：

当前：

```
commit B
 |
 ├── User.java  新版本
 └── Order.java
```

想恢复 User.java 到上一次提交：

执行：

```bash
git restore --source HEAD^ User.java
```

效果：

```
commit B

User.java
  ↓
恢复成 commit A 的版本


Order.java
  ↓
不变
```

然后重新提交：

```bash
git add User.java

git commit -m "恢复User.java"
```

------

# 八、git revert

作用：

撤销一个 commit。

特点：

- 针对提交
- 会产生新的 commit

例如：

原历史：

```
A ---- B ---- C
```

撤销 B：

```bash
git revert B
```

结果：

```
A ---- B ---- C ---- D
                  ↑
              撤销B
```

B 仍然存在，只是 D 抵消了 B 的修改。

------

# 九、restore 和 revert 对比

| 命令        | 作用对象   | 是否产生commit | 用途     |
| ----------- | ---------- | -------------- | -------- |
| git restore | 文件       | 否             | 恢复文件 |
| git revert  | commit     | 是             | 撤销提交 |
| git reset   | commit指针 | 否             | 修改历史 |

------

# 十、已经 commit 后恢复部分文件

场景：

一次提交：

```
commit B

├── User.java
├── Order.java
└── Product.java
```

只恢复 User.java：

```bash
git restore --source HEAD^ User.java
```

然后：

```bash
git add User.java

git commit -m "恢复User文件"
```

不会影响：

```
Order.java
Product.java
```

------

# 十一、GitHub Desktop 对应关系

| Git命令     | GitHub Desktop           |
| ----------- | ------------------------ |
| git add     | Changes 页面勾选文件     |
| git commit  | Commit to main           |
| git push    | Push origin              |
| git restore | Discard changes          |
| git revert  | History 中 Revert commit |

------

# 十二、常用命令总结

查看状态：

```bash
git status
```

查看提交历史：

```bash
git log --oneline
```

添加文件：

```bash
git add 文件路径
```

提交：

```bash
git commit -m "说明"
```

上传：

```bash
git push
```

恢复文件：

```bash
git restore 文件
```

从历史恢复文件：

```bash
git restore --source HEAD^ 文件
```

撤销提交：

```bash
git revert commitId

```