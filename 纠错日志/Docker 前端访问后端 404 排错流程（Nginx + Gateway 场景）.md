# Docker 前端访问后端 404 排错流程（Nginx + Gateway 场景）

## 一、问题现象

典型场景：

浏览器
|
↓
前端容器 Nginx
|
↓
Spring Cloud Gateway
|
↓
后端服务

```
出现问题：

- 本地前端访问正常
- Apifox 调用接口正常
- Docker 部署后的前端访问接口返回 404
- Gateway 没有收到请求

例如：
```

GET /gzmap/api/monitor/district/para

返回：
HTTP/1.1 404 Not Found
Server: nginx
Content-Type: text/html

```
---

# 二、排错思路

遇到这种问题，不要直接修改配置。

按照请求链路逐层排查：
```

前端浏览器
↓
Nginx
↓
Docker网络
↓
Gateway
↓
后端服务

```
原则：

> 从入口开始，一层一层确认请求到底走到了哪里。

---

# 三、第一步：确认请求是否到达前端 Nginx

## 1. 查看浏览器请求

打开：
```

F12 → Network

```
查看：

- Request URL
- Status Code
- Response Header


例如：
```

Request URL:

http://192.168.1.240:8080/gzmap/api/monitor/district/para

```
重点关注：
```

Server: nginx
Content-Type: text/html

```
说明：

返回的是 Nginx 的错误页面。


如果 Gateway 返回，一般类似：

```json
{
  "code":404,
  "msg":"No static resource"
}
```

两者区别：

| 返回             | 说明             |
| ---------------- | ---------------- |
| nginx HTML 404   | 请求停留在 nginx |
| Gateway JSON 404 | 请求已经到网关   |

------

# 四、第二步：检查 Nginx 转发配置

进入前端容器：

```bash
docker exec -it frontend bash
```

查看配置：

```bash
cat /etc/nginx/conf.d/default.conf
```

例如：

```nginx
location /gzmap/api/ {

    proxy_pass http://gjy-gateway-server:48080;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

}
```

确认：

- location 是否匹配请求路径
- proxy_pass 地址是否正确
- gateway 服务名是否正确

------

# 五、第三步：确认 Nginx 使用的是当前配置

不要只看配置文件。

执行：

```bash
nginx -T
```

例如：

```bash
nginx -T | grep -A 20 gzmap/api
```

确认实际加载：

```nginx
location /gzmap/api/ {

    proxy_pass http://gjy-gateway-server:48080;

}
```

原因：

配置文件正确 ≠ nginx 进程已经加载。

修改配置后需要：

```bash
nginx -t
```

检查：

```
syntax is ok
test is successful
```

然后：

```bash
nginx -s reload
```

------

# 六、第四步：检查 Docker 网络

如果 Nginx 转发不到 Gateway，需要检查容器网络。

## 1. 查看网络

```bash
docker network ls
```

例如：

```
gzmap-net
```

## 2. 查看网络详情

```bash
docker network inspect gzmap-net
```

确认：

```
frontend

gateway-server
```

是否在同一个网络。

例如：

```
gzmap-net

    frontend
       |
       |
    gateway-server
```

------

# 七、第五步：测试 Docker 内部 DNS

进入前端容器：

```bash
docker exec -it frontend bash
```

测试：

```bash
ping gjy-gateway-server
```

成功：

```
172.20.0.8
```

说明：

Docker DNS 正常。

注意：

ping 只能说明网络通。

不能证明 HTTP 正常。

------

# 八、第六步：直接访问 Gateway

在前端容器执行：

```bash
curl -v http://gjy-gateway-server:48080
```

正常：

```
HTTP/1.1 200 OK

{
 "code":404,
 "msg":"No static resource"
}
```

说明：

```
frontend
   |
   |
gateway
```

网络正常。

如果这里失败：

检查：

- Gateway 是否启动
- 端口是否暴露
- Docker network

------

# 九、第七步：绕过浏览器测试 Nginx 转发

重点测试：

不是访问 gateway：

```bash
curl http://gjy-gateway-server:48080
```

而是访问 nginx：

```bash
curl -v http://localhost/gzmap/api/monitor/district/para
```

链路：

```
curl
 |
nginx
 |
gateway
```

结果分析：

## 情况1：

返回 Gateway JSON

说明：

Nginx 转发正常。

问题可能：

- 前端代码 baseURL
- 浏览器访问地址
- 跨域

------

## 情况2：

返回 nginx HTML 404

说明：

请求没有进入 proxy。

检查：

- location 匹配
- nginx 配置
- 是否访问了正确 nginx

------

# 十、第八步：查看 Nginx 日志

查看访问日志：

```bash
tail -f /var/log/nginx/access.log
```

请求接口。

如果看到：

```
GET /gzmap/api/monitor/district/para
```

说明：

请求到了 nginx。

继续看：

有没有 gateway 日志。

------

# 十一、常见问题总结

## 1. proxy_pass 路径问题

### 保留原路径：

```nginx
location /api/ {

    proxy_pass http://gateway:48080;

}
```

请求：

```
/api/user
```

转发：

```
/api/user
```

### 去掉前缀：

```nginx
location /api/ {

    proxy_pass http://gateway:48080/;

}
```

请求：

```
/api/user
```

转发：

```
/user
```

区别：

最后有没有 `/`。

------

## 2. Docker 服务名错误

错误：

```nginx
proxy_pass http://localhost:48080;
```

原因：

容器中的 localhost 是自己。

正确：

```nginx
proxy_pass http://gateway-server:48080;
```

利用 Docker DNS。

------

## 3. 修改配置后没有 reload

错误流程：

```
修改配置文件

↓


直接测试

↓


发现不生效
```

正确：

```bash
nginx -t

nginx -s reload
```

或者：

```bash
docker restart frontend
```

------

## 4. 请求到了错误的 Nginx

例如：

```
浏览器
 |
宿主机 nginx
 |
前端容器 nginx
```

检查：

```bash
docker ps
```

查看：

```
8080->80
```

到底映射到了哪个容器。

------

# 十二、面试回答模板

面试官：

> Docker 部署后前端访问后端 404，你怎么排查？

回答：

> 我会按照请求链路逐层排查。首先通过浏览器 Network 查看请求地址和响应来源，判断 404 是 Nginx 返回还是 Gateway 返回。如果是 Nginx 返回，我会检查 Nginx location 匹配和 proxy_pass 配置。然后进入前端容器，通过 nginx -T 确认实际加载的配置。接着检查 Docker 网络，确认前端和 Gateway 是否在同一个 network，并通过容器内 curl 直接访问 Gateway，验证网络和服务是否正常。最后通过 nginx access.log 和 Gateway 日志确认请求是否真正经过网关。这样可以快速定位问题是在前端、Nginx、Docker 网络还是后端服务。

------

# 十三、核心排错口诀

```
先看请求到哪里

再看 nginx 转不转

然后看 docker 通不通

最后看 gateway 收没收
浏览器
 ↓
Nginx
 ↓
Docker DNS
 ↓
Gateway
 ↓
服务
```

按照链路排查，不要盲目修改配置。

