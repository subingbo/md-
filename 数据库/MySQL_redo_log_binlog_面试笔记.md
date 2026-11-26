# MySQL redo log、undo log 与 binlog 面试笔记

> 适用范围：MySQL 8.0 / 8.4，重点围绕 InnoDB、事务持久性、binlog
> 格式、主从复制和两阶段提交。\
> 学习目标：不仅能背出 redo log 和 binlog
> 的区别，还能解释为什么需要两套日志、为什么 redo log
> 不直接用于常规主从复制，以及两阶段提交解决了什么问题。

## 一、先建立整体认识

MySQL 中有多种日志，它们不是同一份数据的重复备份，而是为不同问题服务：

  -------------------------------------------------------------------------------------------
  日志              所属层级          主要记录内容               核心作用
  ----------------- ----------------- -------------------------- ----------------------------
  redo log          InnoDB 存储引擎层 InnoDB                     崩溃恢复，保证事务持久性
                                      数据页修改相关的重做记录   

  undo log          InnoDB 存储引擎层 数据修改前的旧版本信息     事务回滚、MVCC

  binlog            MySQL Server 层   数据库变更事件，格式可为   主从复制、时间点恢复
                                      SQL 或行事件               

  relay log         从库复制线程      从主库接收的 binlog 事件   在从库暂存待应用的复制事件
  -------------------------------------------------------------------------------------------

最重要的记忆：

-   **redo log：本机 InnoDB 崩溃后，如何恢复数据页。**
-   **undo log：事务需要回滚或一致性读时，如何找到旧版本。**
-   **binlog：主库发生了哪些数据库变更，如何把这些变更传递给从库或用于恢复。**
-   **relay log：从库收到主库 binlog 后，在本地暂存的日志。**

## 二、redo log：重做日志

### 2.1 为什么需要 redo log？

InnoDB 为了提高读写效率，会使用 Buffer Pool 缓存数据页。执行 UPDATE
时，通常先修改 Buffer Pool
中的数据页，并将其标记为脏页，之后再由后台线程择机刷入磁盘。

问题是：如果事务已经提交，但脏页还没有刷入磁盘，MySQL
就突然宕机，那么仅靠数据文件无法保证刚刚提交的修改不会丢失。

因此 InnoDB 引入 redo log。数据页修改时，会产生相应的 redo
记录。事务提交时，在满足持久化配置的前提下，先保证 redo log
持久化；数据页可以稍后再刷盘。数据库异常重启时，InnoDB
可以重放需要应用的 redo 记录。

### 2.2 redo log 记录什么？

redo log 是面向 InnoDB
内部数据页修改的重做日志，通常称为物理日志或偏物理日志。

可以用下面的抽象形式理解：

``` text
某个表空间中的数据页发生了修改
需要对相应的数据页应用某项修改
```

它不是简单地保存一条业务 SQL，例如：

``` sql
UPDATE account SET balance = 900 WHERE id = 1;
```

redo log 关注的是 InnoDB
为完成这次修改而产生的底层重做记录，而不是让从库重新执行这条业务 SQL。

注意：实际 redo log 的记录结构比"页号 + 偏移量 + 新值"更复杂，不能把所有
redo 记录都理解为这种简单格式。

### 2.3 WAL：Write-Ahead Logging

WAL 的核心思想是：

**先写日志，再在合适的时机将数据页写入数据文件。**

简化过程：

1.  读取需要修改的数据页。如果不在 Buffer Pool 中，则从磁盘加载。
2.  修改 Buffer Pool 中的数据页，并标记为脏页。
3.  产生相应 redo log 记录。
4.  事务提交时，按照持久化配置将 redo log 刷入持久存储。
5.  后台线程在合适时机将脏页刷入数据文件。
6.  如果数据库异常退出，重启时通过 redo log 恢复尚未写入数据文件的修改。

``` mermaid
flowchart TD
    A["执行 UPDATE"] --> B["读取数据页到 Buffer Pool"]
    B --> C["修改内存数据页，标记脏页"]
    C --> D["生成 redo log"]
    D --> E["事务提交时按配置持久化 redo log"]
    E --> F["事务获得持久性保障"]
    C --> G["后台线程择机刷脏页到数据文件"]
    F --> H["异常重启时检查 redo log"]
    H --> I["重放需要恢复的修改"]
```

WAL 的好处：

-   不需要每次事务提交都立即随机写入对应的数据页。
-   redo log 通常以追加方式写入，减少直接随机更新数据页的压力。
-   即使脏页尚未刷盘，也可以通过 redo log 恢复已提交事务的修改。

### 2.4 redo log 的写入和空间管理

redo log 先在内存中的 log buffer 等区域形成记录，再按内部机制写入 redo
log 文件并刷盘。

redo log
使用循环利用的空间管理方式。旧日志能否被覆盖，取决于检查点（Checkpoint）等恢复边界是否已经推进到足够靠后的位置。

理解两个概念：

-   **LSN（Log Sequence Number）**：用于标识 redo 日志序列进度。
-   **Checkpoint（检查点）**：表示此前的一部分修改已经落实到数据文件，恢复时不再需要从更早的日志位置重放。

因此 redo log
不是永久保存所有历史事务的归档日志。旧空间满足复用条件后可以被重新使用。

### 2.5 redo log 如何参与崩溃恢复？

数据库异常退出后，InnoDB 启动时会进行崩溃恢复：

-   对需要恢复的数据页应用 redo log，使数据页向前恢复（roll-forward）。
-   对崩溃时尚未提交的事务进行回滚，回滚过程需要 undo log 等信息。

所以不能简单地说"redo log 会把所有事务都恢复成已提交状态"。redo log
负责重做，未提交事务还需要被回滚。

## 三、binlog：二进制日志

### 3.1 binlog 是什么？

binlog 是 MySQL Server 层生成的日志，不属于 InnoDB
独有机制。它记录数据库变更事件，例如：

-   数据库表结构变更，例如 CREATE TABLE、ALTER TABLE。
-   表数据的 INSERT、UPDATE、DELETE 等变更。
-   其他需要写入二进制日志的数据库事件。

普通的 SELECT、SHOW 等不修改数据的查询，不会因为开启 binlog
就作为数据变更事件写入 binlog。若需要记录普通查询语句，可以考虑 general
query log（通用查询日志）。

binlog 的两个核心用途：

1.  **主从复制**：主库把 binlog 事件发送给从库，从库应用这些事件。
2.  **时间点恢复（PITR）**：先恢复一个全量备份，再重放备份之后的
    binlog，将数据恢复到指定时间点。

### 3.2 binlog 记录什么？

binlog 具体记录 SQL 还是行变更，取决于 binlog 格式：

-   STATEMENT：记录 SQL 语句。
-   ROW：记录行级变更事件。
-   MIXED：默认采用 STATEMENT，在需要时切换为 ROW。

因此，不能笼统地说 binlog 永远只记录 SQL，也不能说它永远记录每一行数据。

### 3.3 binlog 的文件组织

binlog 通常采用追加写入的方式：

-   当前文件写满或发生日志轮转条件时，会切换到新的 binlog 文件。
-   旧文件可以在满足保留策略时被清理。
-   binlog 不是无限期永久保存，实际保留时间取决于配置和运维策略。

InnoDB 的 redo log 是循环利用的恢复日志空间；binlog
则更适合保留一段时间内的数据库变更历史。

### 3.4 binlog 与事务提交

对于 InnoDB 这样的事务型表，事务执行期间产生的 binlog
内容会先进入事务相关的 binlog cache。事务提交时，MySQL 再将该事务的
binlog 写入 binlog 文件。

需要区分：

-   写入文件系统缓存（write）；
-   将日志同步到持久存储（sync/fsync）。

是否以及何时真正同步到磁盘，还受 `sync_binlog` 等参数影响。

## 四、redo log 和 binlog 的区别

  ------------------------------------------------------------------------------------
  对比维度                redo log                    binlog
  ----------------------- --------------------------- --------------------------------
  所属层级                InnoDB 存储引擎层           MySQL Server 层

  主要目标                崩溃恢复、事务持久性        主从复制、时间点恢复

  记录内容                InnoDB                      数据库变更事件
                          数据页修改相关的重做记录    

  日志类型                通常称物理日志/偏物理日志   逻辑日志；ROW 是行事件

  适用范围                InnoDB                      支持 binlog 的 MySQL 存储引擎

  空间管理                循环利用，受 Checkpoint     追加写入，文件轮转并按策略清理
                          等影响                      

  能否直接替代对方        不能                        不能
  ------------------------------------------------------------------------------------

一句话理解：

-   redo log 解决的是 **InnoDB 本地数据页如何恢复**。
-   binlog 解决的是 **数据库变更如何记录、传播和重放**。

## 五、为什么不能直接用 redo log 做 MySQL 常规主从复制？

不是说任何数据库都不能使用 WAL 类日志做复制，而是 **MySQL InnoDB 的 redo
log 并不是为常规跨实例主从复制设计的接口**。

### 5.1 redo log 面向主库的 InnoDB 内部结构

redo log 中包含 InnoDB
内部数据页修改相关的信息，例如表空间、页及内部修改记录等。

主库和从库虽然可以拥有相同的业务数据，但它们的底层页布局、页分裂历史、内部物理位置等不一定相同。

如果把主库的底层修改记录直接当作从库的修改指令，就会遇到物理结构和实例状态不匹配的问题。

### 5.2 redo log 的生命周期不适合长期复制历史

redo log 会随着 Checkpoint
推进而循环复用空间。它不是为长期保存完整业务变更历史而设计的。

binlog
则用于记录数据库变更事件，可以通过文件轮转、备份和保留策略支持复制及时间点恢复。

### 5.3 redo log 和 binlog 的职责不同

redo log 的恢复语义是 InnoDB 内部的崩溃恢复；binlog
的事件格式和复制协议则面向数据库变更传播。

从库可以根据 binlog 中的 SQL 或行事件应用变更，不需要依赖主库 InnoDB
的物理页布局。

补充：其他数据库可以采用 WAL 物理流复制等方案，但不能因此认为 MySQL 的
InnoDB redo log 可以直接替代 MySQL binlog 复制机制。

## 六、binlog 的三种格式

### 6.1 STATEMENT：语句格式

STATEMENT 模式记录执行的数据变更 SQL。

例如主库执行：

``` sql
UPDATE account
SET balance = 900
WHERE id = 1;
```

binlog 中会记录相应的 SQL 语句事件。从库读取事件后，再执行对应 SQL。

**优点：**

-   对于批量更新，通常只需记录 SQL，而不必逐行记录全部变更。
-   日志量可能较小。

**缺点：**

-   如果 SQL 在主库和从库执行时可能得到不同结果，就可能导致数据不一致。
-   从库需要重新执行 SQL，复杂语句可能消耗更多执行资源。
-   对执行顺序、表结构、函数行为等存在要求。

### 6.2 ROW：行格式

ROW 模式记录具体哪些行发生了变化，而不是要求从库重新计算原始 WHERE
条件并执行相同 SQL。

例如：

``` sql
UPDATE account
SET balance = 900
WHERE balance = 1000;
```

假设有三行被更新，ROW 格式可以抽象理解为：

``` text
id=1：balance 1000 -> 900
id=2：balance 1000 -> 900
id=3：balance 1000 -> 900
```

从库根据行事件应用这些变更。

**优点：**

-   不需要从库重新计算原 SQL 的筛选结果。
-   对非确定性函数、执行计划差异等问题更稳妥。
-   更准确地反映主库实际发生的行变更。

**缺点：**

-   批量修改大量行时，可能产生更多日志数据。
-   只看 ROW 事件不一定能直接看到原始业务 SQL。

ROW 模式的行事件可能包含修改前镜像和修改后镜像。实际记录哪些列受
`binlog_row_image` 等设置影响，例如 FULL、MINIMAL 等模式。

### 6.3 MIXED：混合格式

MIXED 模式会根据语句是否适合使用 STATEMENT 格式进行判断：

-   默认情况下采用 STATEMENT。
-   当 MySQL 将语句判定为不适合安全地进行语句复制时，切换为 ROW。

这不是主库和从库各执行一次 SQL 后比较结果，而是主库依据 MySQL
的规则判断语句是否安全。

``` mermaid
flowchart TD
    A["主库执行数据变更语句"] --> B["MySQL 判断语句是否适合 STATEMENT 复制"]
    B -->|安全| C["以 STATEMENT 形式记录"]
    B -->|不安全| D["以 ROW 形式记录"]
    C --> E["写入 binlog"]
    D --> E
```

MySQL 官方文档列出的 MIXED 自动切换场景包括：

-   语句中包含 `UUID()`。
-   更新带有 AUTO_INCREMENT 列的表，同时调用触发器或存储函数。
-   视图定义需要 ROW 复制。
-   调用 loadable function（可加载函数）。
-   使用 `FOUND_ROWS()` 或 `ROW_COUNT()`。
-   使用 `USER()`、`CURRENT_USER()` 或 `CURRENT_USER`。
-   涉及 mysql 数据库中的日志表。
-   使用 `LOAD_FILE()`。
-   语句引用某些系统变量（存在文档列出的 session 级变量例外）。

此外，官方安全性规则还列出了 `RAND()`、`SYSDATE()`、`UUID_SHORT()`、带
`LIMIT` 且结果顺序不确定的 UPDATE 等不安全场景。具体记录格式仍要结合
MySQL
版本、语句类型和存储引擎能力判断，不应把某个简化函数清单当作所有版本的完整规则。

### 6.4 三种格式对比

  维度               STATEMENT      ROW                  MIXED
  ------------------ -------------- -------------------- ------------------------
  主要记录           SQL 语句       行变更事件           默认 SQL，必要时行事件
  日志量             通常较小       批量变更时可能较大   视具体语句而定
  从库执行方式       重放 SQL       应用行事件           根据事件格式处理
  非确定性语句风险   需要特别注意   通常更安全           不安全语句切换为 ROW
  主要特点           记录逻辑操作   记录实际行变更       兼顾日志量与复制可靠性

## 七、函数为什么会影响 STATEMENT 复制？

### 7.1 核心问题：主从重新执行可能得到不同结果

假设主库执行：

``` sql
UPDATE user
SET token = UUID()
WHERE id = 1;
```

主库执行时，`UUID()` 可能得到：

``` text
aaa-111
```

如果只记录 SQL，从库重新执行时会再次生成 UUID，可能得到：

``` text
bbb-222
```

这样主从数据就不一致了。

因此，MySQL 会将这类语句视为不安全场景，在 MIXED 模式下使用 ROW
记录实际变更。

### 7.2 NOW() 是一个特殊情况

需要特别记住：**NOW() 虽然与当前时间有关，但 MySQL
并不把它视为不安全的语句复制函数。**

原因是 MySQL 会在 binlog
中包含对应的时间戳信息，使从库重放时可以使用源库语句执行时的时间语义。

例如：

``` sql
INSERT INTO audit_log(created_at)
VALUES (NOW());
```

在语句复制时，binlog
中包含时间戳信息，因此从库不会简单地在另一个时间点重新取当前时间。

所以不能把 `NOW()` 和 `UUID()` 一概而论。

### 7.3 常见函数的安全性

  ----------------------------------------------------------------------------------
  函数                    MySQL 8.4 文档中的分类  说明
  ----------------------- ----------------------- ----------------------------------
  `NOW()`                 不视为不安全            binlog 会包含时间戳

  `CURRENT_TIMESTAMP()`   不视为不安全            与 NOW() 类似

  `CURDATE()`             不视为不安全            按复制规则处理

  `UUID()`                不安全                  主从分别生成可能不同

  `RAND()`                不安全                  结果可能不同

  `SYSDATE()`             不安全                  可在同一语句中返回不同值，且不受
                                                  SET TIMESTAMP 影响

  `USER()`                不安全                  从库执行上下文可能不同

  `FOUND_ROWS()`          不安全场景              依赖前序语句执行状态

  `ROW_COUNT()`           不安全场景              依赖执行上下文
  ----------------------------------------------------------------------------------

注意：这是 MySQL
官方对语句复制安全性的分类，不是简单判断函数是否"会变化"。有些函数虽然具有非确定性，但
MySQL 会通过额外的日志信息或特殊规则保证其复制语义。

### 7.4 UPDATE LIMIT 为什么可能不安全？

例如：

``` sql
UPDATE user
SET status = 1
WHERE status = 0
LIMIT 10;
```

如果没有 ORDER BY，SQL 并没有明确规定必须选择哪 10 条记录。

主库和从库可能由于数据布局、执行计划等差异，选择不同的记录。

因此，MySQL 将这种使用 LIMIT 且更新顺序不确定的语句视为不安全场景。

更明确的写法是根据业务要求指定稳定的排序条件，例如：

``` sql
UPDATE user
SET status = 1
WHERE status = 0
ORDER BY id
LIMIT 10;
```

即使这样，也应结合复制规则、索引和数据一致性要求评估，不能把 ORDER BY
当作所有复制风险的万能解决方案。

## 八、MySQL 主从复制是如何工作的？

### 8.1 整体流程

``` mermaid
flowchart TD
    A["客户端向主库发送写请求"] --> B["主库执行事务"]
    B --> C["生成 binlog 事件"]
    C --> D["主库写入 binlog"]
    D --> E["主库发送 binlog 事件"]
    E --> F["从库接收线程"]
    F --> G["写入从库 relay log"]
    G --> H["从库应用线程读取 relay log"]
    H --> I["从库执行 SQL 或应用行事件"]
    I --> J["从库数据完成同步"]
```

### 8.2 主要角色

**主库（Source）：**

-   执行数据修改。
-   生成 binlog。
-   将 binlog 事件发送给从库。

**从库（Replica）：**

-   接收线程从主库获取 binlog 事件。
-   将接收到的事件写入本地 relay log。
-   应用线程读取 relay log，并执行或应用对应事件。

旧资料中经常使用 Master/Slave、IO Thread/SQL Thread
的称呼；新版本官方文档逐渐使用 Source/Replica、receiver/applier 等术语。

### 8.3 为什么需要 relay log？

主库的 binlog 是主库的日志文件，从库需要先接收并在本地保存待应用的事件。

relay log 就是从库的中继日志。

可以这样记：

-   主库：binlog。
-   从库：relay log。
-   从库应用线程：读取 relay log 并执行复制事件。

### 8.4 主从复制是否一定实时？

不一定。常见 MySQL 主从复制是异步复制：

-   主库提交事务后，不必等待从库执行完成。
-   从库接收和应用日志需要时间。
-   因此主从之间可能存在复制延迟。

半同步复制会在提交流程中等待至少一个从库确认收到相关日志，但它也不等于从库已经完成
SQL 应用，更不等于所有从库都已经同步完成。

## 九、为什么 redo log 和 binlog 需要两阶段提交？

### 9.1 如果只有两个日志各自写入，会发生什么？

假设一笔事务需要同时写 redo log 和 binlog。

如果两个日志完全独立提交，就可能出现：

**情况 A：redo log 成功，binlog 失败**

-   InnoDB 恢复时可能认为事务已经提交。
-   但 binlog 中没有这笔事务。
-   从库无法通过 binlog 得到这笔变更。
-   主从可能不一致。

**情况 B：binlog 成功，redo log 失败**

-   binlog 中存在这笔事务。
-   但 InnoDB 数据可能没有完成相应提交。
-   从库可能重放一笔主库最终没有提交的事务。

因此需要保证两套日志对事务提交的判断一致。

### 9.2 两阶段提交的过程

MySQL 使用内部 XA 协调机制，使 binlog 和 InnoDB redo log
的事务提交保持一致。

简化过程：

1.  **Prepare 阶段**：InnoDB 将 redo log 对应事务标记为
    prepare，并按持久化配置刷盘。
2.  **写入 binlog**：MySQL Server 将事务对应的 binlog
    写入并按配置同步到磁盘。
3.  **Commit 阶段**：MySQL 调用 InnoDB 提交事务，将 redo log
    中事务状态标记为 commit。

``` mermaid
sequenceDiagram
    participant C as Client
    participant S as MySQL Server
    participant I as InnoDB
    participant R as redo log
    participant B as binlog

    C->>S: COMMIT
    S->>I: Prepare
    I->>R: 写入 prepare 状态
    R-->>I: 按配置持久化
    I-->>S: Prepare 完成
    S->>B: 写入事务 binlog
    B-->>S: 按配置同步
    S->>I: Commit
    I->>R: 标记事务为 commit
    I-->>S: 提交完成
    S-->>C: 返回成功
```

### 9.3 崩溃恢复时如何判断？

重点理解 XID（事务标识）。

Prepare 阶段的事务还不能仅凭 prepare 状态就认为已经最终提交。MySQL
会在恢复时结合 binlog 中的事务标识判断。

  --------------------------------------------------------------------------------
  崩溃时机             redo log 状态     binlog 状态            恢复处理
  -------------------- ----------------- ---------------------- ------------------
  Prepare 之前         没有完成 prepare  没有完整提交事务记录   回滚未提交事务

  Prepare 完成，binlog prepare           没有对应的有效 XID     回滚该 prepared
  尚未成功持久化                         事务                   transaction

  binlog               prepare           存在对应 XID           提交该 prepared
  已成功持久化，redo                                            transaction
  log 尚未完成 commit                                           

  Commit 完成          commit            存在对应 XID           事务保持提交状态
  --------------------------------------------------------------------------------

因此，两阶段提交不是简单地"先写 redo，再写 binlog"，而是通过
prepare、binlog 持久化、commit 三个步骤协调两个日志的提交状态。

### 9.4 两阶段提交和持久化参数

两个常见参数：

**`innodb_flush_log_at_trx_commit`**

控制 InnoDB redo log 在事务提交时的写入和刷盘策略：

  -----------------------------------------------------------------------------------------------------------
  值                      含义                                       持久性特点
  ----------------------- ------------------------------------------ ----------------------------------------
  0                       每秒写入并刷盘                             可能丢失最近一段事务日志

  1                       每次事务提交时写入并刷盘                   通常用于要求较高持久性的场景

  2                       每次事务提交时写入文件系统缓存，每秒刷盘   操作系统崩溃或断电时可能丢失未刷盘日志
  -----------------------------------------------------------------------------------------------------------

**`sync_binlog`**

控制 binlog 同步到磁盘的频率：

  值   含义
  ---- ---------------------------
  0    由操作系统决定何时同步
  1    每个事务提交组同步一次
  N    每 N 个事务提交组同步一次

对 InnoDB 事务和 binlog 都要求较高持久性的常见配置：

``` ini
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

注意：这些参数会影响性能与持久性之间的权衡。两阶段提交解决的是两个日志的事务协调问题，不能替代磁盘、控制器和硬件本身的可靠性保障。

## 十、常用查看命令

查看 binlog 是否启用：

``` sql
SHOW VARIABLES LIKE 'log_bin';
```

查看当前 binlog 格式：

``` sql
SHOW VARIABLES LIKE 'binlog_format';
```

查看 binlog 同步策略：

``` sql
SHOW VARIABLES LIKE 'sync_binlog';
```

查看 redo log 提交刷盘策略：

``` sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

查看当前有哪些 binlog 文件：

``` sql
SHOW BINARY LOGS;
```

查看当前 binlog 状态和位置（新版本）：

``` sql
SHOW BINARY LOG STATUS;
```

查看从库复制状态：

``` sql
SHOW REPLICA STATUS\G
```

查看当前语句的复制安全性警告：

``` sql
SHOW WARNINGS;
```

查看 ROW 格式 binlog 的行事件：

``` bash
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000001
```

注意：`mysqlbinlog` 的输出是解析后的日志内容；ROW
格式下可以看到行事件的解码信息，不等于原始 SQL 一定完整保存在 binlog
中。

## 十一、面试高频问题与参考回答

### Q1：binlog 和 redo log 有什么区别？

redo log 是 InnoDB
存储引擎层的重做日志，主要记录数据页修改相关信息，用于崩溃恢复，保证事务持久性。binlog
是 MySQL Server
层的二进制日志，记录数据库变更事件，主要用于主从复制和时间点恢复。redo
log 面向 InnoDB 内部恢复，binlog 面向数据库变更传播，两者不能相互替代。

### Q2：为什么有了 redo log 还需要 binlog？

因为两者解决的问题不同。redo log 只能承担 InnoDB
内部的崩溃恢复职责，不适合作为 MySQL 常规主从复制的变更事件接口。binlog
可以记录 SQL 或行变更事件，支持主从复制和时间点恢复。

### Q3：binlog 是记录 SQL 还是记录行变化？

取决于格式。STATEMENT 记录 SQL；ROW 记录具体行变更事件；MIXED 默认使用
STATEMENT，在不安全场景切换为 ROW。

### Q4：为什么 STATEMENT 可能导致主从不一致？

因为主库和从库重新执行同一 SQL
时，可能由于非确定性函数、执行顺序、数据状态或环境差异而得到不同结果。例如
UUID() 在两边重新生成的值可能不同。

### Q5：MIXED 是怎么选择的？

MySQL 根据语句的复制安全性、存储引擎能力和相关规则判断。安全语句可以使用
STATEMENT；被判定为不安全的语句使用 ROW。不是主从各执行一次再比较结果。

### Q6：NOW() 为什么不一定切换 ROW？

因为 MySQL 对 NOW() 有特殊复制处理，binlog
会包含语句执行时的时间戳信息，从库重放时能够使用源库的时间语义。不能把所有非确定性函数都视为不安全函数。

### Q7：为什么 redo log 不直接用于主从复制？

redo log 是 InnoDB
内部的重做记录，依赖引擎内部的数据页和存储结构；主从复制需要跨实例传递可应用的数据库变更事件。MySQL
使用 binlog 完成这一职责。

### Q8：两阶段提交解决什么问题？

解决 redo log 和 binlog 两套日志的事务提交一致性问题，避免出现 InnoDB
认为事务提交但 binlog 没记录，或者 binlog 记录了事务但 InnoDB
最终回滚的情况。

### Q9：redo log 是不是永久保存所有历史记录？

不是。redo log
采用循环利用空间的方式，旧日志在检查点推进并满足复用条件后可以被覆盖。binlog
则按文件追加写入，并由保留策略决定何时清理。

### Q10：主从复制为什么还需要 relay log？

relay log 是从库本地的中继日志，用于暂存从主库接收到的 binlog
事件，供从库应用线程读取和执行。

## 十二、容易混淆的知识点

1.  **redo log 不等于 binlog。** 一个主要负责 InnoDB
    崩溃恢复，一个主要负责数据库变更传播。
2.  **redo log 不等于 undo log。** redo 是重做，undo
    保存旧版本信息，用于回滚和 MVCC。
3.  **binlog 不一定记录 SQL。** ROW 格式记录行事件。
4.  **ROW 格式不是 redo log。** ROW
    记录行级变更，不是主库数据页的物理偏移修改。
5.  **NOW() 不等于 UUID()。** NOW() 在 MySQL
    语句复制中有特殊时间戳处理；UUID() 是不安全场景。
6.  **MIXED 不是随机选择。** 它依据 MySQL
    对语句安全性的判断规则选择格式。
7.  **redo log 不是永久历史归档。** 它会循环利用；binlog
    也可能按保留策略过期清理。
8.  **两阶段提交不只是两个文件依次写入。** 它通过 prepare、binlog
    持久化和 commit 协调事务状态。
9.  **异步复制不代表主从实时一致。** 从库可能有复制延迟。
10. **提交成功与数据页已经刷盘不是一回事。** 在正常 WAL 机制下，redo log
    持久化可以先于数据页刷盘。

## 十三、资料差异与版本校正

本笔记以本次讨论和上传的《150道MySQL+Redis面试题-小林coding-v2.0》及《大厂后端面试真题-小林coding》相关章节为基础，并用
MySQL 官方文档补充版本和规则细节。

需要注意：

-   部分旧版面试资料将 STATEMENT 写为默认格式。MySQL 8.0 和 8.4
    的官方默认 `binlog_format` 是 `ROW`。
-   部分资料将 NOW() 与 UUID()
    一起作为动态函数风险示例。更准确的说法是：NOW() 被 MySQL
    视为可安全进行语句复制的函数，因为 binlog 会记录时间戳；UUID()
    则是不安全场景。
-   MIXED 的自动切换规则不应只靠记忆几个函数。应理解其原则是：当 MySQL
    不能保证语句复制正确时，使用
    ROW；具体行为还与版本、语句类型和存储引擎能力有关。
-   MySQL 8.0.34 起 `binlog_format` 已被标记为 deprecated；MySQL 8.4
    文档也建议新的复制部署使用 ROW 格式。

## 十四、最后用一条主线串起来

``` text
客户端执行 UPDATE
       |
       v
MySQL Server 处理 SQL
       |
       v
InnoDB 修改 Buffer Pool 中的数据页
       |
       +----> 生成 undo log（旧版本信息）
       |
       +----> 生成 redo log（重做记录）
       |
       v
事务提交
       |
       v
redo log Prepare
       |
       v
binlog 持久化
       |
       v
redo log Commit
       |
       +--------------------+
       |                    |
       v                    v
崩溃恢复              主从复制 / 时间点恢复
使用 redo log          使用 binlog
       |
       v
后台择机刷脏页
```

最终记忆：

-   **undo：回滚、MVCC。**
-   **redo：崩溃恢复、持久性。**
-   **binlog：主从复制、时间点恢复。**
-   **relay log：从库接收 binlog 后的本地中继。**
-   **2PC：保证 redo log 和 binlog 对事务提交的判断一致。**

## 十五、参考资料

### 用户上传的面试资料

-   《150道MySQL+Redis面试题-小林coding-v2.0.md》
-   《大厂后端面试真题-小林coding.md》

### MySQL 官方文档

-   [The Binary Log（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/binary-log.html)
-   [Binary Logging Formats（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/binary-log-formats.html)
-   [Mixed Binary Logging Format（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/binary-log-mixed.html)
-   [Determination of Safe and Unsafe Statements in Binary
    Logging（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/replication-rbr-safe-unsafe.html)
-   [Replication and System Functions（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/replication-features-functions.html)
-   [InnoDB Redo Log（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html)
-   [InnoDB Recovery（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/innodb-recovery.html)
-   [InnoDB Startup Options and System Variables（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
-   [Binary Logging Options and Variables（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/replication-options-binary-log.html)
-   [Using Backups for Recovery（MySQL
    8.4）](https://dev.mysql.com/doc/refman/8.4/en/recovery-from-backups.html)
