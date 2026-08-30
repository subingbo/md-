# Git Restore、Revert 完整笔记

## 一、核心结论

Git 中最容易混淆的几个命令：

```text
restore
reset
revert
```

核心区别：

| 命令      | 核心作用                     | 主要针对        | 是否产生新 Commit | 是否可能修改历史 |
| --------- | ---------------------------- | --------------- | ----------------- | ---------------- |
| `restore` | 恢复文件                     | 工作区 / 暂存区 | 否                | 否               |
| `reset`   | 移动 HEAD、调整暂存区/工作区 | Commit / HEAD   | 否                | 是，可能         |
| `revert`  | 撤销某个 Commit              | Commit          | 是                | 否               |

最重要的记忆：

```text
restore = 恢复文件

restore --staged = 取消 git add

revert = 用一个新的 Commit 撤销旧 Commit

reset = 把 HEAD 移回去，可能改变历史
```

------

# 二、理解 Git 的三个区域

学习 `restore` 前必须先理解 Git 的三个区域。

```text
工作区 Working Tree
        │
        │ git add
        ↓
暂存区 Staging Area
        │
        │ git commit
        ↓
本地仓库 Repository
```

例如：

```text
工作区
    UserService.java
        ↓
    当前正在修改的代码

暂存区
    UserService.java
        ↓
    git add 后准备提交的代码

本地仓库
    UserService.java
        ↓
    最近一次 commit 保存的代码
```

因此同一个文件实际上可能存在三个不同版本：

```text
HEAD
 ↓
暂存区
 ↓
工作区
```

------

# 三、git restore

## 3.1 基本作用

```bash
git restore <file>
```

作用：

> 将工作区中的文件恢复成暂存区中的版本。

例如：

```bash
vim UserService.java
```

修改了代码：

```text
HEAD
 ↓
暂存区：旧版本
 ↓
工作区：新版本
```

执行：

```bash
git restore UserService.java
```

变成：

```text
HEAD
 ↓
暂存区：旧版本
 ↓
工作区：旧版本
```

也就是说：

```text
丢弃工作区修改
```

### 注意

这个操作可能导致**尚未提交/暂存的代码修改丢失**。

------

# 四、git restore .

恢复当前目录下所有工作区修改：

```bash
git restore .
```

例如：

```text
UserService.java       修改
OrderService.java      修改
UserController.java    修改
```

执行：

```bash
git restore .
```

这些工作区修改都会被丢弃。

------

# 五、git restore --staged

这是非常重要的用法。

```bash
git restore --staged <file>
```

作用：

> 将文件从暂存区撤回到工作区。

例如：

```bash
git add UserService.java
```

现在：

```text
工作区
   ↓ git add
暂存区
```

如果后悔执行 `git add`：

```bash
git restore --staged UserService.java
```

变成：

```text
暂存区
   ↓
工作区
```

### 重点

它：

```text
不会删除代码修改
```

只是：

```text
取消暂存
```

所以：

```bash
git restore --staged UserService.java
```

≈

```text
取消 git add
```

------

# 六、git restore --staged .

取消所有文件的暂存：

```bash
git restore --staged .
```

例如：

```bash
git status
```

显示：

```text
Changes to be committed:

    UserService.java
    OrderService.java
    UserController.java
```

执行：

```bash
git restore --staged .
```

变成：

```text
Changes not staged for commit:

    UserService.java
    OrderService.java
    UserController.java
```

代码仍然存在，只是取消了 `git add`。

------

# 七、git restore --source

可以从指定 Commit 恢复文件。

语法：

```bash
git restore --source <commit> <file>
```

例如：

```bash
git log --oneline
```

得到：

```text
a1b2c3d fix user bug
e4f5g6h add user service
1234567 initial commit
```

想把 `UserService.java` 恢复成 `e4f5g6h` 中的版本：

```bash
git restore --source e4f5g6h UserService.java
```

结果：

```text
e4f5g6h 中的 UserService.java
             ↓
         当前工作区
```

### 注意

不会自动创建 Commit。

如果想提交：

```bash
git add UserService.java
git commit -m "restore UserService"
```

------

# 八、git restore --source HEAD

```bash
git restore --source HEAD UserService.java
```

作用：

> 将工作区文件恢复成当前 HEAD Commit 中的版本。

------

## 8.1 `git restore <file>` 和 `git restore --source HEAD <file>` 的区别

默认情况下：

```bash
git restore UserService.java
```

主要是：

```text
暂存区 → 工作区
```

而：

```bash
git restore --source HEAD UserService.java
```

是：

```text
HEAD → 工作区
```

如果暂存区和 HEAD 一样，那么两者效果看起来相同。

但是如果暂存区已经发生变化，就不同。

------

# 九、git restore --source HEAD --staged

```bash
git restore --source HEAD --staged UserService.java
```

作用：

> 从 HEAD 恢复文件，并更新暂存区。

也就是：

```text
HEAD
 ↓
暂存区
```

但是默认不一定修改工作区。

------

# 十、彻底恢复文件

如果希望：

```text
工作区 = HEAD
暂存区 = HEAD
```

可以使用：

```bash
git restore --source HEAD --staged --worktree UserService.java
```

效果：

```text
             HEAD
              │
       ┌──────┴──────┐
       ↓             ↓
    暂存区          工作区
```

最终两个区域都恢复成 HEAD 的版本。

这可以理解为：

> **彻底丢弃该文件当前未提交的修改。**

------

# 十一、restore 参数总结

| 命令                                                 | 作用                         |
| ---------------------------------------------------- | ---------------------------- |
| `git restore file`                                   | 暂存区 → 工作区              |
| `git restore .`                                      | 恢复所有工作区修改           |
| `git restore --staged file`                          | 暂存区 → 工作区，取消暂存    |
| `git restore --staged .`                             | 取消所有文件暂存             |
| `git restore --source C file`                        | 从 Commit C 恢复文件到工作区 |
| `git restore --source HEAD file`                     | 从 HEAD 恢复文件到工作区     |
| `git restore --source HEAD --staged file`            | 从 HEAD 恢复暂存区           |
| `git restore --source HEAD --staged --worktree file` | 工作区、暂存区都恢复到 HEAD  |

------

# 十二、git revert

## 12.1 基本概念

```bash
git revert <commit>
```

作用：

> 创建一个新的 Commit，用来抵消指定 Commit 的修改。

例如：

```text
A ---- B ---- C
```

其中：

```text
C = 错误代码
```

执行：

```bash
git revert C
```

变成：

```text
A ---- B ---- C ---- D
                  ↑
              revert C
```

其中：

```text
D
```

是一个新的 Commit。

------

# 十三、revert 不会删除原来的 Commit

这是最重要的概念之一。

执行：

```bash
git revert C
```

并不是：

```text
删除 C
```

而是：

```text
C 仍然存在

D = 抵消 C 的修改
```

所以：

```text
revert ≠ 删除 Commit
```

而是：

```text
revert = 创建一个反向 Commit
```

------

# 十四、为什么公共分支推荐 revert？

假设：

```text
A ---- B ---- C ---- D
```

已经：

```bash
git push
```

到了 GitHub。

其他开发者可能已经基于 D 开发。

如果此时：

```bash
git reset
git push --force
```

就可能修改公共历史。

例如：

```text
原来：

A ---- B ---- C ---- D
```

强制修改之后：

```text
A ---- B
```

其他人的本地历史可能和远程历史发生冲突。

而：

```bash
git revert C
```

则变成：

```text
A ---- B ---- C ---- D ---- E
```

历史仍然完整。

因此：

> **已经推送到公共分支的错误提交，通常优先使用 `git revert`。**

------

# 十五、git revert HEAD

撤销最近一次 Commit：

```bash
git revert HEAD
```

例如：

```text
A ---- B ---- C
             ↑
            HEAD
```

执行：

```bash
git revert HEAD
```

结果：

```text
A ---- B ---- C ---- D
                  ↑
              revert C
```

------

# 十六、HEAD、HEAD~1、HEAD~2

假设：

```text
A ---- B ---- C ---- D
                  ↑
                 HEAD
```

那么：

```text
HEAD
 ↓
D

HEAD~1
 ↓
C

HEAD~2
 ↓
B

HEAD~3
 ↓
A
```

所以：

```bash
git revert HEAD
```

撤销 D。

```bash
git revert HEAD~1
```

撤销 C。

```bash
git revert HEAD~2
```

撤销 B。

------

# 十七、git revert

先查看历史：

```bash
git log --oneline
```

例如：

```text
9f8e7d6 fix order bug
3a2b1c0 add order service
7f6e5d4 initial
```

撤销：

```text
3a2b1c0
```

执行：

```bash
git revert 3a2b1c0
```

Git 创建一个新的 Commit：

```text
7f6e5d4
    ↓
3a2b1c0
    ↓
9f8e7d6
    ↓
新的 revert commit
```

------

# 十八、git revert --no-edit

默认：

```bash
git revert HEAD
```

可能会打开编辑器，让你确认 Commit Message。

默认类似：

```text
Revert "fix user bug"
```

如果不想打开编辑器：

```bash
git revert --no-edit HEAD
```

------

# 十九、revert 冲突

`revert` 并不保证一定成功。

例如：

```text
A ---- B ---- C ---- D
```

C 修改：

```java
int age = 18;
```

D 又修改成：

```java
int age = 20;
```

现在：

```bash
git revert C
```

Git 想反向执行 C：

```text
int age = 18
```

但是 D 又修改了同一个位置。

Git 可能无法判断应该怎么处理，于是产生：

```text
CONFLICT
```

------

# 二十、解决 revert 冲突

首先：

```bash
git status
```

查看冲突文件。

手动解决冲突后：

```bash
git add .
```

然后：

```bash
git revert --continue
```

继续完成 revert。

------

# 二十一、取消 revert

如果发生冲突后发现：

> 我不想继续了。

执行：

```bash
git revert --abort
```

可以取消当前的 revert 操作。

------

# 二十二、一次 revert 多个 Commit

可以：

```bash
git revert A B C
```

但是要注意 Commit 之间可能存在依赖关系。

例如：

```text
A ---- B ---- C
```

如果：

```text
B 依赖 A
C 依赖 B
```

那么同时撤销它们可能产生冲突。

因此实际开发中需要根据提交关系处理。

------

# 二十三、连续 revert 一段 Commit

例如：

```text
A ---- B ---- C ---- D ---- E ---- F
```

想撤销：

```text
C
D
E
```

可以：

```bash
git revert C^..E
```

其中：

```text
C^
```

表示：

```text
C 的父 Commit
```

也就是：

```text
B
```

所以：

```text
C^..E
```

表示：

```text
C → E
```

即：

```text
C
D
E
```

------

# 二十四、restore 和 revert 的根本区别

## restore

关注：

```text
文件
```

例如：

> 我刚才把 UserService.java 改坏了，恢复一下。

```bash
git restore UserService.java
```

------

## revert

关注：

```text
Commit
```

例如：

> 我刚才提交的这个功能有问题，撤销这个 Commit。

```bash
git revert <commit>
```

------

# 二十五、restore vs revert 对比

| 对比项                        | restore      | revert                   |
| ----------------------------- | ------------ | ------------------------ |
| 主要对象                      | 文件         | Commit                   |
| 主要解决                      | 文件恢复     | 撤销提交                 |
| 是否创建 Commit               | 否           | 是                       |
| 是否修改历史                  | 否           | 否                       |
| 是否适合撤销已 push 的 Commit | 不是主要用途 | 是                       |
| 是否可能丢工作区修改          | 是           | 通常不会直接丢工作区修改 |
| 常见使用场景                  | 改错代码     | 提交错误                 |
| 是否操作 HEAD                 | 通常不操作   | 不移动 HEAD              |

------

# 二十六、restore、reset、revert 三者区别

这是 Git 中非常重要的一组知识。

```text
                Git 撤销操作
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
      restore       reset       revert
        │            │            │
      文件         HEAD/索引      Commit
        │            │            │
      恢复文件     移动 HEAD      新建 Commit
        │            │            │
    不修改历史     可能改历史      不改历史
```

------

# 二十七、reset 简单理解

假设：

```text
A ---- B ---- C ---- D
                  ↑
                 HEAD
```

执行：

```bash
git reset B
```

HEAD 移动：

```text
A ---- B
      ↑
     HEAD
```

C、D 不再是当前分支历史的一部分。

所以：

```text
reset
    ↓
移动 HEAD
    ↓
可能修改历史
```

------

# 二十八、reset 的三种模式

## `--soft`

```bash
git reset --soft B
```

结果：

```text
HEAD → B
```

但是：

```text
暂存区
工作区
```

基本保留原来的修改。

适合：

> 我想撤销 Commit，但是保留代码并继续修改/重新提交。

------

## `--mixed`

默认：

```bash
git reset B
```

等价于：

```bash
git reset --mixed B
```

效果：

```text
HEAD → B

暂存区 → 根据 B 重置
工作区 → 保留修改
```

常见用途：

> 撤销 Commit，同时取消暂存，但保留代码修改。

------

## `--hard`

```bash
git reset --hard B
```

会让：

```text
HEAD = B
暂存区 = B
工作区 = B
```

结果：

```text
A ---- B
      ↑
     HEAD
```

C、D 的内容可能直接从当前工作区消失。

因此：

> `reset --hard` 是高风险操作，使用前确认不需要这些修改。

------

# 二十九、什么时候使用哪个？

## 场景 1：代码改了一半，还没 Commit

```text
我要放弃修改
```

使用：

```bash
git restore .
```

------

## 场景 2：代码已经 `git add`

```text
我只是想取消 git add
```

使用：

```bash
git restore --staged .
```

------

## 场景 3：Commit 已经提交，但是还没有推送

可以根据目的选择：

```text
只想撤销 Commit，但保留代码
        ↓
reset --soft

想撤销 Commit，并取消暂存
        ↓
reset --mixed

连代码都不要
        ↓
reset --hard
```

------

## 场景 4：Commit 已经 push 到公共分支

通常：

```bash
git revert <commit>
```

而不是：

```bash
git reset --hard
git push --force
```

原因：

```text
revert
    ↓
保留历史
    ↓
创建新的撤销 Commit
```

------

# 三十、一个完整开发场景

假设当前：

```text
A ---- B ---- C
             ↑
            HEAD
```

现在修改：

```text
UserService.java
OrderService.java
```

------

## 情况一：UserService 改错了

还没有 `git add`：

```bash
git restore UserService.java
```

------

## 情况二：已经 git add，但后悔了

```bash
git restore --staged UserService.java
```

代码仍然保留。

------

## 情况三：连修改也不要了

```bash
git restore --source HEAD --staged --worktree UserService.java
```

最终：

```text
工作区 = HEAD
暂存区 = HEAD
```

------

## 情况四：已经 Commit

```text
A ---- B ---- C
```

C 有问题：

```bash
git revert C
```

结果：

```text
A ---- B ---- C ---- D
                  ↑
              revert C
```

------

## 情况五：已经 Push

仍然：

```bash
git revert C
git push
```

最终：

```text
A ---- B ---- C ---- D
                  ↑
              revert C
```

不会删除 C。

------

# 三十一、常见误区

## 误区 1：`restore` 是撤销 Commit

错误。

```text
restore
```

主要针对：

```text
工作区 / 暂存区中的文件
```

而：

```text
revert
```

才是：

```text
撤销 Commit
```

------

## 误区 2：`revert` 会删除 Commit

错误。

```text
revert C
```

结果：

```text
A ---- B ---- C ---- D
```

C 还存在。

D 只是抵消 C 的修改。

------

## 误区 3：`restore --staged` 会删除代码

错误。

```bash
git restore --staged UserService.java
```

只是：

```text
取消暂存
```

代码仍然在工作区。

------

## 误区 4：`git restore .` 很安全

不完全正确。

如果工作区存在没有保存到 Git 的修改：

```bash
git restore .
```

可能直接丢失这些修改。

------

## 误区 5：公共分支直接 reset

通常不推荐。

公共分支：

```text
已经 push
↓
其他人可能基于这个历史开发
```

优先：

```bash
git revert
```

------

# 三十二、面试回答模板

如果面试官问：

> `git restore` 和 `git revert` 有什么区别？

可以直接回答：

> `git restore` 主要用于恢复文件内容，可以操作工作区和暂存区，本身不会产生新的 Commit。例如 `git restore file` 可以丢弃工作区修改，`git restore --staged file` 可以取消 `git add`。
>
> `git revert` 主要用于撤销已经提交的 Commit，它不会删除原来的 Commit，而是创建一个新的 Commit，通过反向修改抵消之前 Commit 的内容。因此对于已经推送到公共分支的错误提交，通常使用 `revert`，这样不会改写公共历史。

如果继续问：

> 那 `reset` 呢？

回答：

> `reset` 主要是移动 HEAD，并根据 `--soft`、`--mixed`、`--hard` 不同模式决定是否重置暂存区和工作区。它可能改写提交历史，因此公共分支一般不推荐随意使用。

------

# 三十三、最终记忆图

```text
                     Git
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       restore       reset      revert
          │           │           │
          ↓           ↓           ↓
        文件         HEAD        Commit
          │           │           │
          ↓           ↓           ↓
       恢复文件      移动HEAD     新建Commit
          │           │           │
          ↓           ↓           ↓
      不改历史      可能改历史     不改历史
```

## 最终口诀

```text
改错了但没提交
    ↓
restore

git add 错了
    ↓
restore --staged

Commit 提交错了
    ↓
revert

想让 HEAD 回到以前
    ↓
reset

已经 push 到公共分支
    ↓
优先 revert
```

## 最常用命令

```bash
# 丢弃工作区修改
git restore <file>

# 丢弃所有工作区修改
git restore .

# 取消 git add
git restore --staged <file>

# 取消所有 git add
git restore --staged .

# 从指定 Commit 恢复文件
git restore --source <commit> <file>

# 恢复文件到 HEAD，同时修改暂存区和工作区
git restore --source HEAD --staged --worktree <file>

# 撤销最近一次 Commit
git revert HEAD

# 撤销指定 Commit
git revert <commit>

# revert 时不打开编辑器
git revert --no-edit <commit>

# revert 冲突后继续
git revert --continue

# 放弃当前 revert
git revert --abort
```

> **核心只记四句话：**
>
> ```text
> restore：恢复文件
> restore --staged：取消暂存
> revert：创建新 Commit 撤销旧 Commit
> reset：移动 HEAD，可能改写历史
> ```