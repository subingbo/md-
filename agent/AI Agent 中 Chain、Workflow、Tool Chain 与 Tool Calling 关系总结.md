# AI Agent 中 Chain、Workflow、Tool Chain 与 Tool Calling 关系总结

## 1. 核心概念

在 AI 应用中，经常会看到：

- Chain
- Workflow
- Tool Chain
- Tool Calling
- Agent

这些概念容易混淆。

核心区别：

> **Chain 是流程编排机制，Tool Chain 是使用工具组成的流程，Agent 是让 LLM 动态决定调用什么工具。**

------

# 2. Runnable Chain（流程链）

## 2.1 什么是 Runnable Chain

LangChain 中：

```python
chain = (
    RunnableLambda(step1)
    | RunnableLambda(step2)
    | RunnableLambda(step3)
)
```

表示：

> 将多个步骤按照固定顺序连接起来。

等价于：

```python
state = step1(state)
state = step2(state)
state = step3(state)

return state
```

------

## 2.2 执行流程

例如：

```python
self.chain = (
    RunnableLambda(self._load_session)
    | RunnableLambda(self._retrieve)
    | RunnableLambda(self._generate)
    | RunnableLambda(self._persist)
)
```

执行：

```
用户请求
   |
   v
_load_session
   |
   | 加载用户信息
   | 读取历史消息
   |
   v
_retrieve
   |
   | 查询向量数据库
   | 获取知识
   |
   v
_generate
   |
   | 调用大模型
   | 生成回复
   |
   v
_persist
   |
   | 保存聊天记录
   |
   v
返回结果
```

------

# 3. Chain 在项目中的位置

Chain 不是请求入口。

完整结构：

```
HTTP请求
    |
    v
Controller / API
    |
    v
Workflow / Chain
    |
    +----------------+
    |                |
    v                v
Service           Tool
    |
    v
数据库/API/模型
```

------

例如：

```
POST /agent/chat
        |
        v
SpeakingCoachChain.run()
        |
        +-- session_service
        |
        +-- rag_service
        |
        +-- llm_client
        |
        +-- storage
```

------

# 4. Chain 和 Service 的关系

不要理解成：

```
Chain
 |
Service
```

而应该理解：

```
Chain = 业务流程编排者

Service = 提供具体能力
```

例如：

```python
def run():

    load_user()

    search_knowledge()

    generate_answer()

    save_history()
```

Chain 负责：

- 决定执行顺序
- 传递状态
- 组合能力

Service 负责：

- Redis操作
- 数据库操作
- 文件处理
- API调用

------

类似 Java：

```java
OrderService {

    createOrder(){

        checkUser();

        pay();

        saveOrder();

    }
}
```

其中：

```
createOrder()
```

就是流程编排。

------

# 5. Tool Chain（工具链）

## 5.1 什么是 Tool Chain

如果流程中的节点主要是工具：

例如：

```
用户问题

    |
    v

查询天气工具

    |
    v

穿衣推荐工具

    |
    v

返回结果
```

这就是 Tool Chain。

------

代码：

```python
def weather_chain(city):

    weather = weather_tool(city)

    advice = clothing_tool(weather)

    return advice
```

------

## 5.2 Tool Chain 的特点

工具之间存在依赖：

```
Tool A
 |
 | 输出
 v
Tool B
 |
 | 输出
 v
Tool C
```

例如：

```
音频文件
    |
    v
ASR工具
    |
    v
语法分析工具
    |
    v
评分工具
```

------

# 6. Tool Chain 和普通 Chain 区别

本质都是流程：

```
A → B → C
```

区别在节点类型。

------

## 普通 Chain

节点：

```
函数
Service
Prompt
LLM
Retriever
```

例如：

```
加载用户
 ↓
RAG检索
 ↓
LLM生成
 ↓
保存
```

------

## Tool Chain

节点：

```
Tool
Tool
Tool
```

例如：

```
搜索工具
 ↓
计算工具
 ↓
报告工具
```

------

# 7. Tool Calling（工具调用）

## 7.1 什么是 Tool Calling

Tool Calling 的核心：

> 由 LLM 决定是否调用工具。

例如：

用户：

```
北京天气怎么样？
```

LLM判断：

```
需要天气工具
```

调用：

```json
{
  "tool": "weather_tool",
  "args":{
      "city":"北京"
  }
}
```

工具返回：

```
25℃ 晴
```

LLM继续生成：

```
北京今天25℃，天气晴朗。
```

------

# 8. Tool Chain 和 Tool Calling 区别

关键：

> 谁决定下一步。

------

## 固定 Tool Chain

程序决定：

```
Tool A
 |
Tool B
 |
Tool C
```

代码：

```python
a = tool_a()

b = tool_b(a)

c = tool_c(b)
```

------

## Agent Tool Calling

LLM决定：

```
用户问题

     |
     v

    LLM

     |
     +---- Tool A
     |
     +---- Tool B
     |
     +---- Tool C
```

例如：

用户：

```
我的英语最近有什么问题？
```

LLM：

```
需要：
1. 查询历史评分
2. 分析错误
3. 制定计划
```

动态调用：

```
query_score_tool()

analyze_error_tool()

plan_tool()
```

------

# 9. Agent 和 Workflow 区别

## Workflow Agent

固定流程：

```
用户
 |
读取信息
 |
RAG
 |
LLM
 |
保存
```

特点：

- 稳定
- 可控
- 容易调试

你的：

```
SpeakingCoachChain
```

属于：

> Workflow Agent

------

## Autonomous Agent

动态规划：

```
用户
 |
LLM
 |
思考
 |
选择工具
 |
观察结果
 |
继续思考
 |
完成任务
```

特点：

- 灵活
- 不确定性高
- 更难控制

------

# 10. 为什么 Tool Chain 看起来简单

因为：

固定工具链：

```python
A()
B(A)
C(B)
```

本质就是普通业务代码。

难点不在调用：

而在：

## 1. 工具选择

例如：

```
用户问题
    |
    v
选择哪个工具？
```

------

## 2. 参数生成

用户：

```
最近表现怎么样？
```

需要生成：

```json
{
"user_id":"123",
"time":"last_week"
}
```

------

## 3. 多轮决策

流程：

```
调用工具
    |
得到结果
    |
再次判断
    |
继续调用工具
```

------

## 4. 异常处理

例如：

```
Tool调用失败

怎么办？

重试？
换工具？
继续？
返回错误？
```

------

# 11. AI 应用常见架构

```
              API层
                |
                v
        Agent / Workflow层
                |
     -------------------------
     |          |            |
     v          v            v
  RAG服务    Tool服务     LLM服务
     |          |            |
     v          v            v
 VectorDB    外部API      大模型


                |
                v

            数据存储
```

------

# 12. 你的口语陪练项目属于什么

你的：

```python
SpeakingCoachChain
```

属于：

```
Workflow Chain
        +
RAG
        +
LLM
        +
Memory
```

不是：

```
❌ 完全自主Agent
❌ LLM动态选择工具
```

------

当前流程：

```
POST /agent/chat

        |
        v

SpeakingCoachChain

        |
        +-- 加载会话
        |
        +-- RAG检索
        |
        +-- LLM生成
        |
        +-- 保存记录
```

------

未来升级：

加入：

```
search_tool
speech_tool
grammar_tool
database_tool
```

然后：

```
LLM
 |
决定调用哪些工具
 |
动态组合
```

才变成：

```
Agent + Tool Calling
```

------

# 最终总结

| 概念           | 本质                 | 谁决定流程 |
| -------------- | -------------------- | ---------- |
| Runnable Chain | 固定流水线           | 程序       |
| Workflow       | 业务流程编排         | 程序       |
| Tool Chain     | 工具组成的流水线     | 程序       |
| Tool Calling   | LLM调用工具          | LLM        |
| Agent          | LLM自主规划 + 调工具 | LLM        |

一句话：

> Chain 是“怎么串流程”，Tool Chain 是“串工具流程”，Tool Calling 是“让模型决定调用哪个工具”，Agent 是在 Tool Calling 基础上的智能决策系统。