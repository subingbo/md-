# Spring Boot WebSocket 使用总结

# 1. WebSocket是什么

WebSocket 是一种**全双工通信协议**。

传统 HTTP：

客户端 ----请求----> 服务端

客户端 <---响应---- 服务端

```
特点：

- 一次请求一次响应
- 服务端不能主动推送消息
- 每次通信需要重新发送 HTTP 请求


WebSocket：
```

客户端
|
| 建立长连接
|
服务端

```
连接建立后：

- 客户端可以主动发送消息
- 服务端可以主动推送消息
- 双方可以持续通信


适合场景：

- 在线聊天
- 实时通知
- 股票行情
- 游戏状态同步
- AI 流式输出


---

# 2. Spring Boot WebSocket 两种使用方式

Spring Boot 常见两种：

## 2.1 原生 WebSocket

直接操作：
```

WebSocketSession

```
自己负责：

- 保存用户连接
- 用户和连接绑定
- 消息发送
- 广播


结构：
```

客户端

|
WebSocket

|

SpringBoot

|

WebSocketSession

```
特点：

简单直接，但是业务逻辑需要自己实现。


---

## 2.2 WebSocket + STOMP

STOMP：

> Simple Text Oriented Messaging Protocol

是一种消息协议。

它运行在 WebSocket 之上。


提供：

- 消息路由
- 发布订阅
- 广播
- 用户消息


结构：
```

客户端

|

WebSocket

|

STOMP

|

Message Broker

|

订阅用户

```
---

# 3. 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

核心类：

```java
org.springframework.web.socket.WebSocketSession
```

------

# 4. 原生 WebSocket 工作流程

## 4.1 Endpoint

配置：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig 
        implements WebSocketConfigurer {


    @Override
    public void registerWebSocketHandlers(
            WebSocketHandlerRegistry registry) {


        registry.addHandler(
                new ChatHandler(),
                "/ws/chat"
        );

    }
}
```

这里：

```
/ws/chat
```

叫：

> WebSocket Endpoint

客户端连接：

```
ws://localhost:8080/ws/chat
```

------

# 5. Endpoint、Handler、Session关系

三者区别：

```
Endpoint

连接入口


Handler

处理连接和消息的代码


WebSocketSession

某一个具体用户的连接
```

完整流程：

```
客户端

   |
   | ws://xxx/ws/chat
   |

Endpoint

   |

ChatHandler

   |

WebSocketSession
```

------

## Endpoint

作用：

负责建立连接。

例如：

```
/ws/chat
```

它不是一个 Java 类。

Spring 内部维护：

```
/ws/chat

    |

ChatHandler
```

------

## Handler

负责：

- 连接成功
- 收到消息
- 连接关闭

例如：

```java
@Component
public class ChatHandler 
        extends TextWebSocketHandler {


    @Override
    public void afterConnectionEstablished(
            WebSocketSession session){

        System.out.println("连接成功");

    }



    @Override
    protected void handleTextMessage(
            WebSocketSession session,
            TextMessage message){

        System.out.println(
            message.getPayload()
        );

    }



    @Override
    public void afterConnectionClosed(
            WebSocketSession session,
            CloseStatus status){

        System.out.println("关闭");

    }

}
```

------

## WebSocketSession

表示：

> 一条已经建立的 WebSocket 长连接

例如：

```
用户A

   |

sessionA


用户B

   |

sessionB
```

通常保存：

```
用户ID -> WebSocketSession
```

例如：

```java
Map<Long, WebSocketSession> sessions;
```

数据：

```
10001 -> session1

10002 -> session2
```

------

# 6. 原生 WebSocket 消息发送

## 6.1 客户端发送

浏览器：

```javascript
ws.send("hello");
```

服务器：

```
客户端消息

      |

      ↓

handleTextMessage()
```

------

## 6.2 服务端主动发送

代码：

```java
session.sendMessage(
    new TextMessage("hello")
);
```

流程：

```
业务代码

    |

WebSocketSession

    |

客户端
```

注意：

真正发送消息的是：

```
WebSocketSession
```

不是 Endpoint。

------

# 7. 多用户通信

服务器维护：

```
用户ID

  |

WebSocketSession
```

例如：

```
10001 -> sessionA

10002 -> sessionB

10003 -> sessionC
```

A发送：

```
A

 |

hello

 ↓

服务器
```

服务器找到：

```java
sessionB.sendMessage(
    "hello"
);
```

B收到：

```
hello
```

------

# 8. 服务端主动推送

例如：

订单完成：

```
订单服务

创建订单成功

       |

       ↓

WebSocket服务

       |

       ↓

用户浏览器
```

代码：

```java
webSocketManager.sendMessage(
    userId,
    "订单成功"
);
```

------

# 9. 多 Endpoint 的意义

多个 Endpoint：

```java
/ws/chat

/ws/notice

/ws/game
```

默认：

仍然连接同一个 Spring Boot。

结构：

```
SpringBoot

    |

 ------------------

 |                |

/ws/chat      /ws/notice

 |                |

ChatHandler   NoticeHandler
```

意义：

## 1. 不同业务隔离

聊天：

```
/ws/chat
```

通知：

```
/ws/notice
```

## 2. 不同权限

例如：

普通用户：

```
/ws/chat
```

管理员：

```
/ws/admin
```

## 3. 微服务中对应不同服务

例如：

```
Gateway

    |

 -----------------

 |               |

chat-service notice-service

    |               |

 /ws/chat     /ws/notice
```

------

# 10. WebSocket 集群问题

单机：

```
用户

 |

SpringBoot

 |

WebSocketSession
```

多节点：

```
              Nginx

        /      |      \

      服务1   服务2   服务3


用户A连接服务1

用户B连接服务3
```

问题：

服务1不知道服务3上的用户连接。

解决：

```
业务服务

   |

Redis / MQ

   |

WebSocket服务

   |

用户
```

------

# 11. STOMP模式

STOMP 是 WebSocket 上的消息协议。

解决：

- 消息路由
- 订阅
- 广播

------

# 12. STOMP配置

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig
implements WebSocketMessageBrokerConfigurer {


    @Override
    public void registerStompEndpoints(
            StompEndpointRegistry registry) {


        registry.addEndpoint("/ws");

    }



    @Override
    public void configureMessageBroker(
            MessageBrokerRegistry registry) {


        registry.enableSimpleBroker(
            "/topic"
        );


        registry.setApplicationDestinationPrefixes(
            "/app"
        );

    }

}
```

------

# 13. STOMP三个核心地址

## 13.1 Endpoint

连接入口：

```
ws://localhost:8080/ws
```

作用：

建立 WebSocket 连接。

------

## 13.2 /app

客户端发送消息给服务器。

客户端：

```javascript
stompClient.send(
    "/app/chat",
    {},
    "hello"
);
```

进入：

```java
@MessageMapping("/chat")
public void chat(String msg){

}
```

关系：

```
/app/chat

      ↓

@MessageMapping("/chat"
```

------

## 13.3 /topic

服务器广播地址。

客户端订阅：

```javascript
stompClient.subscribe(
    "/topic/room1",
    callback
);
```

服务器：

```java
template.convertAndSend(
    "/topic/room1",
    "hello"
);
```

所有订阅者收到：

```
/topic/room1


    |

-----------------

A       B       C


收到消息
```

------

# 14. @MessageMapping 和 MVC注解区别

Spring MVC：

```java
@GetMapping

@PostMapping

@RequestMapping
```

处理：

```
HTTP请求
```

流程：

```
HTTP

↓

DispatcherServlet

↓

Controller
```

WebSocket STOMP：

```java
@MessageMapping
```

处理：

```
STOMP消息
```

流程：

```
WebSocket

↓

MessageBroker

↓

@MessageMapping
```

------

# 15. MVC 和 WebSocket 可以共存

同一个项目：

```java
@Controller
public class ChatController {


    @GetMapping("/history")
    public List history(){

    }



    @MessageMapping("/chat")
    public void chat(){

    }

}
```

但是：

```
@GetMapping

属于HTTP


@MessageMapping

属于WebSocket
```

两套处理链路。

------

# 16. 类比理解

## WebSocket

类似：

```
电话线路
```

建立：

```
拨电话
```

之后：

```
持续通话
```

------

## STOMP

类似：

```
电话交换机
```

负责：

- 转接
- 分组
- 广播

------

# 17. 最终架构理解

简单项目：

```
用户

 |

WebSocket

 |

SpringBoot

 |

WebSocketSession
```

企业项目：

```
             Gateway

                |

        WebSocket服务

                |

          Redis / MQ

                |

           业务服务
```

------

# 核心记忆

```
Endpoint

负责建立连接


Handler

负责处理连接和消息


WebSocketSession

代表一个用户连接


/ws

建立连接


/app/chat

客户端发送消息给服务器


@MessageMapping("/chat")

处理消息


/topic/xxx

服务器广播
```

最终理解：

> WebSocket负责建立实时连接；WebSocketSession代表具体连接；Handler负责处理连接生命周期；STOMP负责连接后的消息路由和订阅。Spring MVC处理HTTP请求，WebSocket(STOMP)处理实时消息，两者是两套机制。

