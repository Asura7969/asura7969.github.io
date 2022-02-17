---
title: Flink RPC
date: 2021-08-15 01:31:33
tags: flink
categories: rpc
cover: /img/topimg/20210815013700.png
---
### 主要抽象类
**RpcEndpoint**: 具体服务的尸体抽象,线程安全类

**RpcGateway**: 用于远程调用的代理接口,提供对应`RpcEndpoint`的地址方法

**RpcService**: `RpcEndpoint`的运行时环境,并提供异步任务或周期性任务调度

**RpcServer**: `RpcEndpoint`自身的代理对象

**FencedRpcEndpoint** & **FencedRpcGateway**: 在调用`RPC`方法时需提供`token`信息


