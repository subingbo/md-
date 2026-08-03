# Nginx 核心原理与配置笔记

# 一、Nginx 本质是什么

Nginx 本质上是：

```text
HTTP 请求路由器 + 反向代理服务器
```

它负责：

- 接收浏览器请求
- 根据规则匹配请求
- 决定：
  - 返回静态文件
  - 转发给后端
  - 转发 websocket
  - 重定向
  - 负载均衡

---

# 二、一次完整请求流程

用户访问：

```text
https://ice.wqsx.top/api/user?id=1
```

完整流程：

```text
DNS解析
↓
域名 → IP

TCP连接
↓
连接 IP:443

TLS握手(SNI)
↓
告诉服务器域名

HTTP请求
↓
Host: ice.wqsx.top

Nginx server匹配
↓
listen + server_name

Nginx location匹配
↓
path

执行逻辑
↓
proxy_pass / try_files / return
```

---

# 三、URL 结构

完整 URL：

```text
scheme://host:port/path?query#fragment
```

例如：

```text
https://ice.wqsx.top:443/api/user?id=1#top
```

拆分：

| 部分     | 内容         |
| -------- | ------------ |
| scheme   | https        |
| host     | ice.wqsx.top |
| port     | 443          |
| path     | /api/user    |
| query    | ?id=1        |
| fragment | #top         |

注意：

```text
fragment(#top)不会发送给服务器
```

---

# 四、server 与 location 的关系

Nginx 配置层级：

```text
main
 ↓
http
 ↓
server
 ↓
location
```

---

## server

表示：

```text
一个网站
```

负责：

- 域名
- 端口
- HTTPS
- 证书
- 默认配置

---

## location

表示：

```text
网站内部URL路由规则
```

负责：

- URL匹配
- proxy_pass
- try_files
- rewrite
- return

---

# 五、location 必须在 server 内

正确：

```nginx
server {

    location / {

    }
}
```

错误：

```nginx
location / {

}
```

会报错：

```text
"location" directive is not allowed here
```

因为：

```text
location 必须属于某个 server
```

---

# 六、server 匹配规则

Nginx 选择 server：

```text
先匹配 listen
再匹配 server_name
```

---

## 示例

```nginx
server {
    listen 80;
    server_name ice.wqsx.top;
}

server {
    listen 443 ssl;
    server_name ice.wqsx.top;
}
```

---

## 访问：

```text
http://ice.wqsx.top
```

浏览器连接：

```text
IP:80
```

只会匹配：

```nginx
listen 80
```

---

## 访问：

```text
https://ice.wqsx.top
```

浏览器连接：

```text
IP:443
```

只会匹配：

```nginx
listen 443 ssl
```

---

# 七、端口优先级高于域名

即使：

```text
server_name 一样
```

只要：

```text
listen 不同
```

也是不同 server。

---

# 八、server_name 不匹配怎么办

如果：

- listen 匹配
- server_name 不匹配

则：

```text
进入 default_server
```

默认情况下：

```text
同端口第一个 server
```

---

# 九、location 匹配规则

location 匹配：

```text
只匹配 path
```

例如：

```text
/api/user
```

不匹配：

- 域名
- 端口
- query参数

---

# 十、location 匹配优先级

优先级：

```text
1. =
精确匹配

2. ^~
前缀匹配(停止正则)

3. ~ ~*
正则匹配

4. 普通前缀匹配
最长优先

5. /
兜底
```

---

# 十一、/api 和 /api/ 区别

---

## location /api

匹配：

```text
/api
/api/
/apixxx
/api-test
```

本质：

```text
字符串前缀匹配
```

---

## location /api/

匹配：

```text
/api/
/api/user
```

不会匹配：

```text
/apixxx
```

生产环境推荐：

```nginx
location /api/
```

---

# 十二、root 的作用

root：

```text
定义静态文件根目录
```

例如：

```nginx
root C:/www;
```

请求：

```text
/app.js
```

实际查找：

```text
C:/www/app.js
```

---

# 十三、root 本质

公式：

```text
最终文件路径
=
root + URL path
```

---

# 十四、root 不会去掉 location

例如：

```nginx
location /img/ {
    root C:/data;
}
```

请求：

```text
/img/a.png
```

实际路径：

```text
C:/data/img/a.png
```

---

# 十五、alias 与 root 区别

---

## root

保留 location 部分：

```nginx
location /img/ {
    root C:/data;
}
```

结果：

```text
C:/data/img/a.png
```

---

## alias

替换掉 location：

```nginx
location /img/ {
    alias C:/data/;
}
```

结果：

```text
C:/data/a.png
```

---

# 十六、try_files 的作用

```nginx
try_files $uri $uri/ /index.html;
```

作用：

```text
先找真实文件
找不到则返回 index.html
```

用于：

```text
Vue/React SPA 前端路由
```

---

# 十七、为什么 SPA 需要 try_files

例如：

用户访问：

```text
/user/1
```

服务器实际上没有：

```text
/user/1/index.html
```

如果没有：

```nginx
try_files ... /index.html
```

会：

```text
404
```

加了后：

统一返回：

```text
index.html
```

然后：

```text
Vue Router 接管路由
```

---

# 十八、proxy_pass 的作用

```nginx
proxy_pass http://127.0.0.1:8080;
```

表示：

```text
请求转发给后端
```

---

# 十九、proxy_pass URI 替换规则

---

## 有 /

```nginx
proxy_pass http://127.0.0.1:8080/;
```

会：

```text
替换掉 location 匹配部分
```

---

### 示例

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

请求：

```text
/api/user
```

转发：

```text
/user
```

---

## 没有 /

```nginx
proxy_pass http://127.0.0.1:8080;
```

会：

```text
保留完整 URI
```

---

### 请求

```text
/api/user
```

转发：

```text
/api/user
```

---

# 二十、为什么 /api 和前端分离

因为：

```text
location /
```

用于：

```text
前端静态资源
```

而：

```text
location /api/
```

用于：

```text
后端接口
```

实现：

```text
前后端分离
```

---

# 二十一、现代前后端结构

```text
浏览器
    ↓
Nginx
 ┌────────────┐
 ↓            ↓
前端静态资源   后端API
(Vue/React)   (SpringBoot)
```

---

# 二十二、WebSocket 配置

WebSocket 必须：

```nginx
proxy_http_version 1.1;

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

否则：

```text
WebSocket 握手失败
```

---

# 二十三、Nginx 可以路由到不同服务器

例如：

```nginx
location /java/ {
    proxy_pass http://10.0.0.1:8080;
}

location /python/ {
    proxy_pass http://10.0.0.2:5000;
}
```

实现：

```text
不同URL → 不同服务器
```

---

# 二十四、负载均衡

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

Nginx 会：

```text
自动分发请求到多台服务器
```

---

# 二十五、核心理解总结

---

## server

负责：

```text
这个请求属于哪个网站
```

依据：

- listen
- server_name

---

## location

负责：

```text
这个网站里哪个功能
```

依据：

- path

---

## root

负责：

```text
文件去哪找
```

---

## try_files

负责：

```text
文件不存在怎么办
```

---

## proxy_pass

负责：

```text
是否转发给后端
```

---

# 二十六、最终完整模型

```text
URL
 ↓
DNS解析
 ↓
IP:port
 ↓
listen匹配server
 ↓
Host匹配server_name
 ↓
path匹配location
 ↓
proxy_pass / try_files
 ↓
后端 or 文件系统

```

# 二十七、为什么不能写两个相同的 location /

同一个 `server` 内：

```nginx
location / {

}

location / {

}
```

会报错：

```text
duplicate location "/"
```

因为：

```text
location 本质上是 URL 路由规则
```

Nginx 对一次请求：

```text
只会选择一个最匹配的 location
```

不会：

```text
同时执行多个 location
```

---

# 二十八、为什么前后端要分 /api/

现代前后端分离项目通常：

```nginx
location / {
    前端页面
}

location /api/ {
    后端接口
}
```

原因是：

```text
需要区分：
哪些请求返回前端页面
哪些请求转发后端
```

---

# 二十九、location / 是兜底规则

例如：

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

会匹配：

```text
所有普通请求
```

因此：

```text
通常作为前端 SPA 的兜底规则
```

---

# 三十、location 匹配时只会进入一个

例如：

```nginx
location / {
}

location /api/ {
}
```

请求：

```text
/api/user
```

会进入：

```nginx
location /api/
```

因为：

```text
它比 / 更长
```

而：

```nginx
location /
```

不会再执行。

---

# 三十一、现代前后端分离标准结构

```nginx
server {

    location / {
        前端Vue/React页面
    }

    location /api/ {
        SpringBoot接口
    }

    location /websocket {
        websocket服务
    }
}
```

这是现代 Web 项目最经典的部署结构。