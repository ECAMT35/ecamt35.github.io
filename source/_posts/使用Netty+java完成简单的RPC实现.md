---
title: 使用Netty+java完成简单的RPC实现
date: 2025-11-25 00:00:00
description: 使用Netty+java完成简单的RPC实现
categories: 
- 技术理论
---

RPC又称为远程调用，它让客户端像调用本地方法一样调用远程服务，屏蔽了底层的网络通信细节。

一个基本的RPC框架需要这几个部分：
- **rpc-client**：提供服务接口和实现
- **rpc-server**：调用远程服务的客户端
- **公共服务接口-api**：客户端用来调用远程方法的代理对象，由服务端完成具体的实现

# rpc-client

主要是构建 rpc 请求，一般有几个重要的参数：
- 抽象接口类名
- 抽象方法名（版本、组等，可选）
- 参数类型、参数列表

```java
RpcRequest request = new RpcRequest("com.api.HelloService", "sayHello", new Class[]{String.class}, new Object[]{"张三"});
// 此处使用 Jackson 序列化
String jsonRequest = new ObjectMapper().writeValueAsString(request);
// 发送 rpc 请求报文
endpoint.sendMessage(jsonRequest);
```

# rpc-server

主要工作：
- 对抽象接口进行具体的实现；
- 完成服务注册功能（此处仅是基于内存的简单实现）；
- 使用 Netty 完成 WebSocket 服务器的构建；

工作流程：
1. 完成服务注册
2. 启动 WebSocket 服务器
3. 收到 rpc 请求报文，根据抽象接口类名，在服务注册中心找到对应的实现类
4. 再根据方法名、参数列表，通过反射的方式获取到调用的方法
5. 调用方法，构建响应报文给返回客户端

## 服务注册

此处仅是基于内存的简单实现，本质是一个 `Map<String, Object> serviceMap` 完成的一个映射表。

- key=抽象接口类名（版本、组等为可选）
- value=服务实现类实例

需要在启动 WebSocket 服务器前完成注册：
```java
public final Map<String, Object> serviceMap = new HashMap<>();

/**
 * 注册服务：将接口与实现类绑定
 * @param serviceInterface 服务接口（如 api 模块下的 HelloService.class）
 * @param serviceImpl 服务实现类实例（如 new HelloServiceImpl()）
*/
public void registerService(Class<?> serviceInterface, Object serviceImpl) {
    // 存储接口的全限定名（客户端通过该名称找到服务）
    serviceMap.put(serviceInterface.getName(), serviceImpl);
}
```

## 调用本地接口

核心是反射获取类的方法，然后执行：
```java
// 根据请求找到服务实现类
Object serviceImpl = serviceMap.get(request.getInterfaceName());
if (serviceImpl == null) {
    throw new RuntimeException("服务未注册：" + request.getInterfaceName());
}

// 反射得到类的信息
Class<?> serviceClass = serviceImpl.getClass();
// 再根据方法名、参数列表获取要调用的具体方法
Method method = serviceClass.getMethod(
	request.getMethodName(),
	request.getParameterTypes() // 注意：参数类型必须完全匹配
);
// 执行方法，result用于返回rpc响应
Object result = method.invoke(serviceImpl, request.getParameters());
```

以上是一个rpc的简单实现思路，具体实现肯定是比较复杂的。
如可使用 zookeeper 做注册中心和配置中心；高性能序列化；负载均衡；重试机制；限流/降级/熔断；

