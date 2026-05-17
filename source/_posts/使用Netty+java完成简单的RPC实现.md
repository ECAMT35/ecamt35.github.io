---
title: 使用Netty+java完成简单的RPC实现
date: 2025-11-25 00:00:00
description: 使用Netty+java完成简单的RPC实现
categories: 
- 技术理论
---
RPC（Remote Procedure Call，远程过程调用）是一种允许客户端像调用本地方法一样调用远程服务的技术。RPC 屏蔽了网络通信、序列化、服务发现等复杂细节，让分布式系统的调用更像本地调用。

| 组件              | 作用                                     |
| --------------- | -------------------------------------- |
| **RPC Client**  | 调用远程服务的客户端，通过代理对象封装 RPC 请求并发送给服务器      |
| **RPC Server**  | 提供服务实现的服务器，接收客户端请求并返回调用结果              |
| **公共服务接口（API）** | 定义客户端与服务端共享的服务接口契约，客户端通过代理调用，服务端提供具体实现 |

注：RPC 框架的核心目标是**让客户端像调用本地方法一样调用远程服务**，而不是让客户端直接操作网络或序列化。

# RPC Client

RPC Client 的主要作用是完成 RPC 请求封装。
客户端调用远程方法时，需要构建一个请求对象，包含关键信息：

```java
public class RpcRequest {
    private String interfaceName;      // 服务接口全限定名
    private String methodName;         // 调用方法名
    private Class<?>[] parameterTypes; // 参数类型
    private Object[] parameters;       // 参数值
}
```

示例：

```java
RpcRequest request = new RpcRequest(
    "com.api.HelloService",
    "sayHello",
    new Class[]{String.class},
    new Object[]{"张三"}
);
```

请求通常需要序列化（JSON、Protobuf、Hessian 等），然后通过网络发送到服务器：

```java
String jsonRequest = new ObjectMapper().writeValueAsString(request);
endpoint.sendMessage(jsonRequest);
```

以上是 RPC Client 完成的基本工作。

当然，生产环境通常通过**代理对象封装 RPC 调用**，客户端代码不直接处理 `RpcRequest`：

```java
HelloService helloService = RpcProxy.create(HelloService.class);
String result = helloService.sayHello("张三");  // 实际是发 RPC 请求
System.out.println(result);
```

# RPC Server

主要工作：
- 对抽象接口进行具体的实现；
- 完成服务注册功能（此处仅是基于内存的简单实现）；
- 使用 Netty 完成 WebSocket 服务器的构建；

工作流程：
1. 完成服务注册
2. 启动网络服务器（如 Netty、WebSocket、HTTP 等）
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

## 方法调用

核心是通过反射获取类的方法，然后执行：

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

以上是一个 rpc 的简单实现思路，具体实现肯定是比较复杂的。如：
- 使用 Zookeeper、Consul、Etcd 做服务注册、发现；
- 高性能序列化；
- 负载均衡；
- 重试机制；
- 限流/降级/熔断；

# 总结

一个简单的 RPC 实现流程：

1. 客户端调用代理方法 → 构建 `RpcRequest`
2. 序列化请求 → 发送到服务端
3. 服务端接收请求 → 从注册中心获取实现类 → 反射调用方法
4. 将结果序列化 → 返回给客户端
5. 客户端接收响应 → 返回调用结果


