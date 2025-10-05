---
title: Netty的简单使用
date: 2025-10-06 00:00:00
description: Netty的使用记录
categories: 
- 技术理论
tags:
- Netty
- WebSocket
---

**Netty的主从Reactor多线程模型**：BossGroup处理连接请求，WorkerGroup处理I/O操作

Netty工作一般需要三个线程组：

- **BossGroup**：与客户端建立连接；
- **WorkerGroup**：处理已连接的 I/O 事件（读写、编解码）；
- **自定义业务线程池**：执行耗时的业务逻辑（如数据库操作、复杂计算）；

具体工作流程：

1. **监听端口**：BossGroup 中的线程会绑定服务端端口（如 8080），持续监听客户端的 TCP 连接请求（三次握手）；
2. **接收连接**：当客户端完成 TCP 三次握手后，连接会进入内核的 “已完成连接队列”，BossGroup 的线程调用`accept()`获取该连接，生成代表连接的`Channel`对象；
3. **转交连接**：将`Channel`注册到 WorkerGroup 中某个线程（`EventLoop`）的`Selector`上，此后该连接的所有 I/O 事件都由这个 Worker 线程负责。

## BossGroup

`BossGroup` 的工作非常轻量（仅处理连接建立），因此线程数不需要太多。实际开发中通常直接使用`new NioEventLoopGroup(1)`——1 个线程足够应对大部分场景。

## WorkerGroup

`WorkerGroup`是 Netty 处理 I/O 事件的核心，它的职责是**处理已建立连接的所有网络事件**，包括：

- 读取客户端发送的数据；
- 对数据进行编解码（如 JSON 转对象、协议解析）；
- 将处理后的数据写回客户端。

Q：为什么说 WorkerGroup 是 "传送带"？
A：WorkerGroup 中的每个线程（`EventLoop`）都绑定一个`Selector`（多路复用器），负责监听其管理的所有`Channel`的 I/O 事件。由于 I/O 操作是非阻塞的（基于 NIO 的`select`机制），Worker 线程在等待 I/O 就绪时不会阻塞，可高效切换到其他就绪的`Channel`处理事件 —— 这就像传送带，始终在 "搬运" 数据，不浪费时间等待。

Q：为什么说 WorkerGroup 需要拒绝耗时操作
WorkerGroup 的线程是 "I/O 专用" 的，**绝对不能在其中执行耗时业务逻辑**（如查询数据库、调用远程接口）。原因很简单：如果 Worker 线程被耗时任务占用，会导致其管理的所有`Channel`的 I/O 事件无法及时处理，最终引发连接超时、吞吐量下降。

WorkerGroup 的线程数通常设置为**CPU 核心数 × 2**（Netty 的默认值）。

## 业务线程池

当 WorkerGroup 完成数据的读取和编解码后，就需要处理具体的业务逻辑了（如校验数据、操作数据库、调用 RPC 服务）。这些操作往往耗时较长（毫秒级甚至秒级），如果放在 Worker 线程中执行，会阻塞 I/O 处理 —— 因此需要一个专门的**业务线程池**来承载这些 "重活"。

线程数量分配：
- CPU 密集型：以 “CPU 核心数” 为基准；
- I/O 密集型：以 “CPU 核心数 ×2” 为基准。

# 基于WebSocket的Netty服务器

示例：
```java
@Component
@Slf4j
public class NettyServer {

    @Value("${netty.websocket.port:8080}")
    private int port;

    @Value("${netty.websocket.path:/ws}")
    private String webSocketPath;

    @Value("${netty.websocket.max-frame-size:65536}")
    private int maxFrameSize;

    @Value("${netty.websocket.idle-timeout:60}")
    private int idleTimeout;

    // 自定义的业务线程池
    @Autowired
    private ExecutorService businessExecutor;

    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;
    private Channel serverChannel;

    @Autowired
    private ObjectProvider<WebSocketFrameHandler> webSocketFrameHandlerProvider;

    @PostConstruct
    public void start() {

        bossGroup = new NioEventLoopGroup(1);
        workerGroup = new NioEventLoopGroup(); // 默认CPU核心数*2

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    // 指定服务器通道类型（NIO非阻塞）
                    .channel(NioServerSocketChannel.class)
                    // 服务端TCP参数配置
                    .option(ChannelOption.SO_BACKLOG, 1024) // 半连接队列大小
                    .option(ChannelOption.SO_REUSEADDR, true) // 端口释放后立即重用
                    
                    // 客户端连接参数配置（child开头的选项用于子通道）
                    .childOption(ChannelOption.SO_KEEPALIVE, true) // 开启TCP心跳
                    .childOption(ChannelOption.TCP_NODELAY, true) // 禁用Nagle算法
                    
                    // 配置通道初始化器（每个新连接都会执行）
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            // 获取通道流水线（责任链模式，处理器按顺序执行）
                            ChannelPipeline pipeline = ch.pipeline();

                            // HTTP编解码处理器
                            pipeline.addLast(new HttpServerCodec());

                            // HTTP消息聚合器
                            pipeline.addLast(new HttpObjectAggregator(maxFrameSize));

                            // WebSocket 协议升级
                            pipeline.addLast(new WebSocketServerProtocolHandler(
                                    webSocketPath, // WebSocket连接路径
                                    null, // 子协议（不指定）
                                    true, // 允许扩展
                                    maxFrameSize, // 最大帧大小
                                    false, // 不允许掩码（服务端接收客户端消息时）
                                    true // 自动关闭连接
                            ));

                            // 空闲检测（仅读空闲）
                            pipeline.addLast(new IdleStateHandler(
                                    idleTimeout, 0, 0, TimeUnit.SECONDS
                            ));

                            // 自定义WebSocket消息处理器（注入业务线程池）
                            WebSocketFrameHandler frameHandler = webSocketFrameHandlerProvider.getObject();
                            frameHandler.setBusinessExecutor(businessExecutor);
                            pipeline.addLast(frameHandler);
                        }
                    });

            // 绑定端口并启动服务
            serverChannel = bootstrap.bind(port).sync().channel();
            log.info("Netty WebSocket server started at ws://localhost:{}{}", port, webSocketPath);

        } catch (InterruptedException e) {
            log.error("Netty server start interrupted", e);
            Thread.currentThread().interrupt(); // 恢复中断状态
            stop();
        } catch (Exception e) {
            log.error("Failed to start Netty server", e);
            stop();
        }
    }

    /**
     * 停止Netty服务器（Spring容器销毁时执行）
     */
    @PreDestroy
    public void stop() {
        
        log.info("Shutting down Netty WebSocket server...");
        
        // 关闭服务器通道（解绑端口，停止接收新连接）
        if (serverChannel != null && serverChannel.isActive()) {
            serverChannel.close().addListener(future -> {
                if (future.isSuccess()) {
                    log.info("Server channel closed successfully");
                } else {
                    log.error("Server channel closed failed", future.cause());
                }
            });
        }
        // 关闭Worker线程组（处理完现有任务后关闭）
        if (workerGroup != null) {
            workerGroup.shutdownGracefully(1, 5, TimeUnit.SECONDS)
                    .addListener(future -> log.info("Worker group shutdown completed"));
        }
        // 关闭Boss线程组
        if (bossGroup != null) {
            bossGroup.shutdownGracefully(1, 5, TimeUnit.SECONDS)
                    .addListener(future -> log.info("Boss group shutdown completed"));
        }

        log.info("Netty WebSocket server stopped");
        
    }

    /**
     * 获取当前服务器绑定的端口（用于监控）
     */
    public int getBoundPort() {
        if (serverChannel != null && serverChannel.isActive()) {
            return ((InetSocketAddress) serverChannel.localAddress()).getPort();
        }
        return -1;
    }
}
```

```java
@Component
@Scope("prototype")
@Slf4j
public class WebSocketFrameHandler extends SimpleChannelInboundHandler<WebSocketFrame> {

    // 业务线程池
    private ExecutorService businessExecutor;

    // 提供业务线程池setter方法，供NettyServer注入
    public void setBusinessExecutor(ExecutorService businessExecutor) {
        this.businessExecutor = businessExecutor;
    }

    // ... 其他原有逻辑（channelRead0, 事件处理等）
    // 使用this.businessExecutor提交任务即可
	@Override
	protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame msg) {
	    // 1. 异步处理前，增加引用计数（父类释放时不会导致计数为0）
	    msg.retain(); 

	    businessExecutor.submit(() -> {
	        // 2. 业务线程中处理消息，此时msg未被释放
	        try {
	            if (msg instanceof TextWebSocketFrame) {
	                String text = ((TextWebSocketFrame) msg).text();
	                String result = processBusiness(text);
	                // 写回响应时，切换到I/O线程（Netty要求：写操作必须在I/O线程执行）
	                ctx.executor().execute(() -> {
	                    if (ctx.channel().isActive()) {
	                        ctx.writeAndFlush(new TextWebSocketFrame(result));
	                    }
	                });
	            }
	        } catch (Exception e) {
	            log.error("Business process failed", e);
	        } finally {
	            // 3. 业务处理完后，手动释放引用（平衡之前的retain()）
	            ReferenceCountUtil.release(msg); 
	        }
	    });
	    
	    // 无需手动finally释放——父类会自动处理（此时msg引用计数为1，父类释放后变为0）
	}
}
```

# 基于TCP的Netty服务器

 这个是纯TCP的服务器，之前做测试的时候用的

```java
@Component
public class NettyServer {

    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;
    private Channel serverChannel;
    private final int port = 8080; // Netty 监听端口

    @PostConstruct
    private void start() throws InterruptedException {
        bossGroup = new NioEventLoopGroup(1); // 主线程组（处理连接请求）
        workerGroup = new NioEventLoopGroup(); // 工作线程组（处理 I/O 操作）

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class) // 使用 NIO 传输
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {

                        ch.pipeline()
                                //.addLast(new LoggingHandler(LogLevel.INFO)) // 添加网络层日志
                                .addLast(new StringDecoder())
                                .addLast(new StringEncoder())
                                .addLast(new MyServerHandler()) // 添加自定义处理器（需实现）
                        ;
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128) // 连接队列大小
                .childOption(ChannelOption.SO_KEEPALIVE, true); // 保持 TCP 连接

        // 绑定端口并启动服务器
        serverChannel = bootstrap.bind(port).sync().channel();
        System.out.println("Netty server started on port: " + port);
    }

    @PreDestroy
    private void stop() {
        if (serverChannel != null) {
            serverChannel.close(); // 关闭 Channel
        }
        if (workerGroup != null) {
            workerGroup.shutdownGracefully(); // 优雅关闭工作线程组
        }
        if (bossGroup != null) {
            bossGroup.shutdownGracefully(); // 优雅关闭主线程组
        }
        System.out.println("Netty server stopped");
    }
}
```

```java
public class MyServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("Received: " + msg);
        ctx.writeAndFlush("Echo: " + msg); // 回传消息
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close(); // 发生异常时关闭连接
    }
}
```

测试：
终端使用telnet连上TCP连接：
```bash
$ telnet 127.0.0.1 8080
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
my fork
Echo: my fork
my spoon
Echo: my spoon
```

Netty服务端日志：
```log
Received: my fork
Received: my spoon
```

Netty服务端接收到Msg经过解码编码后进行回应，并记录日志。

