# Flask 后端端口占用导致接口超时排查笔记

## 故障现象

前端请求接口一直转圈，无法返回结果。

使用 curl 测试：

```bash
curl -v http://127.0.0.1:6000/
```

结果：

```text
Connected to 127.0.0.1 port 6000
...
Operation timed out after 5000 milliseconds
```

说明：

- TCP 三次握手成功
- 已成功连接到 6000 端口
- 服务端没有返回任何 HTTP 响应

------

## 初步怀疑

### 1. 内存不足

查看内存：

```cmd
wmic OS get TotalVisibleMemorySize,FreePhysicalMemory /value
```

结果：

```text
FreePhysicalMemory=454388
TotalVisibleMemorySize=2013484
```

换算：

- 总内存约 1.92 GB
- 剩余内存约 444 MB
- 内存使用率约 77%

结论：

```text
不是内存不足导致的问题
```

------

### 2. 后端未启动

检查端口：

```cmd
netstat -ano | findstr :6000
```

发现 6000 端口处于 LISTENING 状态。

说明：

```text
后端进程存在
```

------

## 关键现象

查看 Python 进程：

```cmd
tasklist | findstr python
```

发现存在两个 Python 进程。

实际原因：

```text
之前启动的后端实例没有关闭
后来又启动了一个新的实例
```

导致系统中同时存在多个相同后端进程。

------

## 为什么会出现超时

### 情况一：旧实例正常

```text
旧实例（正常） -> 6000
新实例启动失败
```

结果：

```text
前端仍然可以正常访问
```

只是访问的仍然是旧版本代码。

------

### 情况二：旧实例异常或卡死

```text
旧实例（卡死） -> 6000
新实例启动失败
```

结果：

```text
curl Connected
curl Timeout
前端一直转圈
后端无日志
```

原因：

```text
请求实际上进入了旧实例
但旧实例已经无法处理请求
因此没有任何响应返回
```

这与本次故障现象完全一致。

------

## 解决方法

关闭所有 Python 后端进程：

```cmd
taskkill /IM python.exe /F
```

确认端口释放：

```cmd
netstat -ano | findstr :6000
```

重新启动后端：

```cmd
python app.py
```

服务恢复正常。

------

## 排查经验总结

当出现：

```text
前端一直转圈
curl 超时
后端无日志
```

时，优先检查：

### 查看端口

```cmd
netstat -ano | findstr :6000
```

### 查看 Python 进程

```cmd
tasklist | findstr python
```

### 测试接口

```cmd
curl -v --max-time 5 http://127.0.0.1:6000/
```

若出现：

```text
Connected
Operation timed out
```

则说明：

```text
TCP正常
应用层异常
大概率是旧后端进程卡死或残留进程导致
```

------

## 开发习惯建议

每次启动项目前先检查：

```cmd
tasklist | findstr python
```

或：

```cmd
netstat -ano | findstr :6000
```

确保不存在残留后端进程。

如有异常：

```cmd
taskkill /IM python.exe /F
```

清理后再重新启动项目。