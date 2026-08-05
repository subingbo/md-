# 消息队列 MQ 原理总结

（内容已整理为完整 Markdown 笔记）

## 一、MQ核心模型

### RabbitMQ

```text
Producer
    |
 Exchange
    |
 Queue
    |
Consumer
```

- Exchange负责路由
- Queue保存消息
- ACK确认消费
- 同一个Queue里的消费者竞争消费


### Kafka

```text
Topic
 |
 +-- Partition0
 +-- Partition1
 +-- Partition2
```

- Partition是日志分片
- 消息追加写入
- 消费者通过offset记录消费位置
- 同Group分摊消费
- 不同Group可以重复消费


### RocketMQ

```text
Topic
 |
 +-- MessageQueue0
 +-- MessageQueue1
```

- 类似Kafka
- Consumer Group维护消费进度
- 支持事务消息、顺序消息

---

## 二、Kafka Partition

Partition不是复制消息，而是切分数据：

```text
Topic

Partition0 -> Broker1
Partition1 -> Broker2
Partition2 -> Broker3
```

作用：

1. 提升吞吐
2. 支持多机器并行
3. 提高消费者并发

---

## 三、Consumer Group

同一个Group：

```text
Consumer1
Consumer2
Consumer3
```

多个消费者分配Partition。

不同Group：

```text
Group A
Group B
Group C
```

可以消费同一份消息。

原因：

每个Group维护自己的offset。

---

## 四、Offset

Kafka/RocketMQ不会像RabbitMQ一样消费后删除消息。

而是：

```text
消息一直保存

offset不断移动
```

offset表示：

消费者消费到的位置。

---

## 五、消息可靠性

三个阶段：

Producer -> Broker -> Consumer


Producer：

- ACK确认
- 失败重试


Broker：

- 磁盘持久化
- 副本机制


Consumer：

- 业务成功后ACK
- 避免消息丢失


---

## 六、重复消费

原因：

### 生产者

发送成功，但是ACK丢失：

```text
Producer重试
```

导致重复。


### 消费者

业务成功，但是ACK/offset提交失败：

导致重新消费。


---

## 七、消费者幂等

解决重复消费：

### 1. 数据库唯一索引

例如：

payment_id UNIQUE


### 2. Redis保存messageId

消费前检查是否处理过。


### 3. 状态机

例如：

```text
待支付 -> 已支付
```

重复执行不会改变结果。

---

## 八、Kafka零拷贝

普通IO：

```text
磁盘
 ↓
Page Cache
 ↓
用户空间
 ↓
Socket
 ↓
网卡
```


零拷贝：

```text
磁盘
 ↓
Page Cache
 ↓
Socket
 ↓
网卡
```

Kafka通过sendfile减少CPU复制和上下文切换。


---

## 九、Kafka生产者幂等

开启：

```properties
enable.idempotence=true
```

Kafka给Producer：

- Producer ID
- Sequence Number


Broker记录：

```text
PID -> 最大Sequence
```

重复消息直接丢弃。


---

## 十、RocketMQ事务消息

解决：

数据库和MQ一致性问题。


流程：

### 1. Half Message

消息进入MQ，但是消费者不可见。


### 2. 执行本地事务

例如：

创建订单。


### 3. Commit / Rollback

成功：

消息可消费。

失败：

删除消息。


### 4. 事务回查

如果RocketMQ不知道事务结果：

主动询问生产者。


---

## 十一、三个MQ区别总结

| |RabbitMQ|Kafka|RocketMQ|
|-|-|-|-|
|模型|Queue|Topic+Partition|Topic+MessageQueue|
|消费进度|ACK|Offset|Offset|
|消息消费后|删除|保留|保留|
|消费组|无|有|有|
|广播|多个Queue|不同Group|不同Group|
|优势|业务异步|高吞吐日志|业务消息|

---

## 最终记忆

RabbitMQ：

> 消息是任务，消费后删除。


Kafka：

> 消息是日志，消费只是移动offset。


RocketMQ：

> 消息是业务事件日志，支持更多业务能力。


同Group：

> 负载均衡。


不同Group：

> 广播消费。
