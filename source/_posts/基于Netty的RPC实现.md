---
title: 基于Netty的RPC实现
date: 2019-12-22 01:10:20
tags:
- 轮子
- Java
- 未完
---

# 基于Netty实现一个RPCServer
容易想到也容易实现的是一个Json协议的HTTP Server，用起来大概像这个样子：
```bash
curl POST 'localhost:5031/rpc' \
--header 'Content-Type: application/json' \
--data-raw '{
	"interface":"muyi.jnpc.server.api.TestService",
	"method":"sayHi",
	"args":{
		"name": "\"muyi\""
	}
}'
```
那么想象中`RPCClient`也是通过将rpc调用转化为http请求来调用`RPCServer`，**KISS**。

# 基于Netty实现一个RPCClient

# 通过Spring来简化RPC接口的服务端暴露和用户端声明

# 池化RPCServer和RPCClient

