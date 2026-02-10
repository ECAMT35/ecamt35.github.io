---
title: Java虚拟线程
date: 2026-02-10 00:00:00
description: Java虚拟线程
categories: 
- 技术理论
tags:
- Java虚拟线程
---

参考：
[https://www.nasuiyile.cn/794.html](https://www.nasuiyile.cn/794.html)  
[https://www.jdon.com/76350-synchronized-xunixiancheng.html](https://www.jdon.com/76350-synchronized-xunixiancheng.html)

JDK 21 正式推出的虚拟线程（Virtual Thread，JEP 444）是 Java 并发模型的一次重大演进。它旨在以近乎零成本的资源消耗应对海量并发连接，特别适合 IO 密集型应用，可以大大提高 CPU 的使用率，同时保持了与现有线程 API 的完全兼容。

# 传统并发模型的瓶颈

在虚拟线程之前，Java 的并发构建于**平台线程**之上，即对操作系统内核线程的一对一包装。其生命周期与调度完全由操作系统内核管理。

这种模型在高并发 IO 场景下暴露出固有缺陷：

- **资源成本高**：每个线程都需要分配独立的栈内存（通常 MB 级别），创建数千个线程就会消耗大量内存，并触及 OS 的线程数上限。
- **调度开销大**：线程的创建、销毁以及上下文切换都涉及昂贵的系统调用（用户态与内核态切换）。
- **并发能力受限**：为平衡资源开销与调度成本，应用通常采用线程池。例如，Tomcat 默认工作线程池大小为 200。当所有线程都在等待数据库、网络等 IO 响应时，即便 CPU 空闲，新的请求也无法被处理，形成瓶颈。

# 虚拟线程的核心设计

虚拟线程是 **JVM 实现的轻量级用户态线程**。它不再与 OS 线程绑定，而是由 JVM 负责调度到少量的**载体线程**上执行。

其核心特性如下：

- **极致轻量**：栈内存可动态伸缩，初始占用极小，使得创建数百万个虚拟线程成为可能。
- **高效调度**：当虚拟线程执行阻塞操作（如 IO、`Thread.sleep`）时，JVM 会将其**挂起**，并释放其占用的载体线程，该载体线程可立即去执行其他就绪的虚拟线程。这一切都在**用户态**完成，规避了内核切换的开销。
- **无缝兼容**：`java.lang.Thread` API 保持不变，现有代码几乎无需修改即可使用虚拟线程。

# 调度原理

理解虚拟线程的调度机制是正确使用的关键。

## 非抢占式（协作式）用户态调度

与操作系统内核基于时间片**抢占式**调度平台线程不同，JVM 对虚拟线程采用**非抢占式**调度。虚拟线程只会在遇到**阻塞点**（如 IO 操作、`LockSupport.park` 等）时主动让出执行权。如果一个虚拟线程永不阻塞，它将独占其载体线程。

以下代码示例展示了这一特性。首先，我们创建 CPU 核心数个虚拟线程，每个线程都包含一个 `sleep(1)` 阻塞点：

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        int busyThreads = Runtime.getRuntime().availableProcessors();
        int newThreadIndex = 0;
        List<Thread> threads = new ArrayList<>();

        // 创建并启动"死循环虚拟线程"
        for (int i = 0; i < busyThreads; i++) {
            int id = i;
            Thread virtualThread = Thread.ofVirtual()
                    .name("busy-virtual-thread-" + id)
                    .start(() -> {
                        System.out.println("Busy virtual thread " + id + " started");
                        while (true) {
                            try {
                                Thread.sleep(1);
                            } catch (InterruptedException e) {
                                throw new RuntimeException(e);
                            }
                        }
                    });
            threads.add(virtualThread);
        }

        // 等待一会儿，让死循环线程占满平台线程
        Thread.sleep(1000);

        // 创建并启动一个新虚拟线程尝试执行
        Thread newVirtualThread = Thread.ofVirtual()
                .name("new-virtual-thread-" + newThreadIndex)
                .start(() -> {
                    System.out.println("New virtual thread " + newThreadIndex + " started");
                });
        threads.add(newVirtualThread);

        // 等待一会儿查看结果
        Thread.sleep(10000);
    }
}
```

因为其他线程的循环中的 `sleep` 函数将会使这些虚拟线程将当前执行权让给新来的线程，所以这段代码能够成功输出 "New virtual thread"。

然而，如果我们稍微修改一下代码，移除 `sleep` 调用，使其变为纯 CPU 循环：

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        int busyThreads = Runtime.getRuntime().availableProcessors();
        int newThreadIndex = 0;
        List<Thread> threads = new ArrayList<>();

        // 创建并启动"死循环虚拟线程"
        for (int i = 0; i < busyThreads; i++) {
            int id = i;
            Thread virtualThread = Thread.ofVirtual()
                    .name("busy-virtual-thread-" + id)
                    .start(() -> {
                        System.out.println("Busy virtual thread " + id + " started");
                        while (true) {
                            // 纯 CPU 循环，不阻塞
                        }
                    });
            threads.add(virtualThread);
        }

        // 等待一会儿，让死循环线程占满平台线程
        Thread.sleep(1000);

        // 创建并启动一个新虚拟线程尝试执行
        Thread newVirtualThread = Thread.ofVirtual()
                .name("new-virtual-thread-" + newThreadIndex)
                .start(() -> {
                    System.out.println("New virtual thread " + newThreadIndex + " started");
                });
        threads.add(newVirtualThread);

        // 等待一会儿查看结果
        Thread.sleep(10000);
    }
}
```

这时 "New virtual thread" 永远不会输出。这是因为纯 CPU 循环没有阻塞点，虚拟线程永远不会主动让出执行权，从而独占载体线程。这就是非抢占式调度：虚拟线程之间需要 " 协商 " 何时释放资源，而协商的契机就是阻塞操作。

这种调度模式的**好处**是不需要 JVM 定期中断操作，避免了额外的性能开销。**坏处**是如果有 11 个完全不休息的任务（即 11 个虚拟线程）要运行在一个 10 线程的 CPU 上，其中一个任务就会持续无法被调度运行。

## 载体线程池与工作窃取算法

虚拟线程默认由 JVM 全局的 `ForkJoinPool`（作为载体线程池）进行调度。

- **并行度**：池中活跃载体线程数默认等于 CPU 核心数，可通过 `-Djdk.virtualThreadScheduler.parallelism` 调整。
- **工作窃取**：每个载体线程拥有一个本地任务队列。当自身队列为空时，它会从其他线程的队列尾部 " 窃取 " 任务，以此实现高效的负载均衡。
- **挂载与卸载**：就绪的虚拟线程被挂载到空闲载体线程上运行；遇到阻塞点时，虚拟线程被卸载，载体线程被释放回池。阻塞结束后，虚拟线程被重新排队等待调度。

**重要边界**：虚拟线程的总并发能力受限于载体线程池的规模。默认最大载体线程数为 `256`（可通过 `-Djdk.virtualThreadScheduler.maxPoolSize` 调整）。这意味着，如果同时有超过 256 个虚拟线程处于**非阻塞**运行状态，超出的部分将会等待。

# 适用场景与使用边界

虚拟线程是 **IO 密集型任务**的利器，而非万能解药。

| 推荐场景 | 优势 |
| :--- | :--- |
| Web 服务器 (Spring Boot/Tomcat) | 支持海量并发连接，无需复杂线程池调优 |
| 微服务 RPC 调用链 | 每个调用可分配独立虚拟线程，避免线程池耗尽导致级联故障 |
| 数据库/缓存客户端 | IO 等待期间自动释放资源，提升系统整体吞吐量 |
| 消息队列消费者 | 轻松实现高并发消费 |

**需谨慎或避免的场景**：

- **CPU 密集型计算**：无阻塞点，无法发挥调度优势，应用 `ForkJoinPool` 或固定线程池。
- **虚拟线程池化**：创建成本极低，池化反增复杂度。
- **在 `synchronized` 块内阻塞**：可能导致 **Pinning**（钉住），即虚拟线程无法从载体线程卸载，严重降低吞吐量。**应优先使用 `ReentrantLock`**。（注：JDK 24 已优化此问题，但未完全根除）。
- **未适配的 Native 方法阻塞**：同样可能引发 Pinning。

## 注意事项
### Pinning 问题

什么是 Pinning？
**Pinning** 是指虚拟线程在阻塞时无法从载体线程卸载，导致载体线程被 " 钉住 " 无法执行其他虚拟线程的现象。这会严重降低系统并发能力和吞吐量。

Pinning 的触发场景：

1. **synchronized 同步块内阻塞**

```java
public class PinningExample {
    private final Object lock = new Object();
    
    public void blockingMethod() {
        synchronized (lock) {  // 虚拟线程进入synchronized块
            try {
                // 模拟阻塞操作（如数据库查询、网络请求）
                Thread.sleep(1000);  // 虚拟线程无法卸载！
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

2. **JNI/Native 方法阻塞**

```java
public class NativePinning {
    // 加载未适配虚拟线程的本地库
    static {
        System.loadLibrary("unadapted");
    }
    
    // 本地方法声明
    public native void blockingNativeMethod();
    
    public void callNative() {
        // 调用可能阻塞的本地方法
        blockingNativeMethod();  // 可能无法卸载
    }
}
```

为什么 synchronized 会导致 Pinning？
Java 中的 `synchronized` 关键字基于监视器（monitors）实现，其设计存在历史局限性：

1. **监视器绑定平台线程**：Java 的监视器机制与平台线程深度耦合。每个对象都有一个关联的监视器，一次只能有一个线程持有对象的监视器。在 JVM 内部，监视器所有权是**在平台线程（内核线程）层面跟踪的**，而不是在虚拟线程层面。当虚拟线程进入 synchronized 块时，其**载体线程**（平台线程）被记录为监视器的所有者，而不是虚拟线程本身。
    
2. **卸载破坏互斥性**：如果虚拟线程在 synchronized 块内被卸载，其载体线程会被释放回线程池，可能被分配给其他虚拟线程使用。但 JVM 仍然记录该载体线程持有监视器，这会导致：
    
    - 新的虚拟线程（使用同一载体线程）被错误地认为持有监视器
    - 其他试图获取同一监视器的虚拟线程可能被不当阻塞
    - 这破坏了 synchronized 的互斥语义，因此 JVM 必须阻止虚拟线程在持有监视器时卸载
    
3. **wait/notify 机制**：`Object.wait()` 要求在同一线程上释放和重新获取监视器。如果虚拟线程在执行 synchronized 方法时调用了 `Object.wait()`，并且虚拟线程在等待期间卸载，当被 `notify()` 唤醒时，它可能被调度到不同的载体线程上，无法满足 " 同一线程重新获取监视器 " 的要求，因此也会被固定。

解决方案：

1. **使用 ReentrantLock 替代 synchronized**：ReentrantLock 的锁状态通过 `AbstractQueuedSynchronizer` (AQS) 管理，不依赖线程标识，使用原子变量（`AtomicInteger`）跟踪锁状态，而不是线程引用。所以 JVM 并不会阻止虚拟线程的卸载。

```java
private final ReentrantLock lock = new ReentrantLock();
    
public void doWork() {
    lock.lock();
    try {
        // 阻塞操作可以正常卸载
        Thread.sleep(1000);
    } finally {
        lock.unlock();
    }
}
```

2. **升级第三方库**。确保使用的库已适配虚拟线程（支持可中断的阻塞操作）。
    
3. **监控与诊断**
	
    - 使用 JFR 监控 `jdk.VirtualThreadPinned` 事件
    - 通过 `-Djdk.tracePinnedThreads=full` 参数追踪 Pinning
    
4. **JDK 24+ 的改进**  
    JDK 24 通过 JEP 491 大幅减少了 synchronized 导致的 Pinning，但在递归锁等复杂场景下仍可能存在。

### ThreadLocal 内存泄漏

虚拟线程支持创建百万级实例，滥用 `ThreadLocal` 可能导致严重的内存泄漏。考虑使用 `ScopedValue`（JDK 21+ 预览，JDK 25 稳定）进行不可变数据传递。

```java
private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();

ScopedValue.where(USER_ID, "user123").run(() -> {
    // 在此作用域内，USER_ID.get() 返回 "user123"
});
```

# 最佳实践

## 创建方式

**简单启动**：`Thread.startVirtualThread(Runnable task)`

**构建器模式**（支持命名、配置）：

```java
Thread vt = Thread.ofVirtual().name("my-vt").unstarted(task);
vt.start();
```

**执行器服务**（推荐，便于管理）：

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
Future<String> future = executor.submit(() -> "Hello");
```

## 结构化并发

对于有多个并发子任务的场景，使用 `StructuredTaskScope`（JDK 21+ 预览，JDK 23 稳定）可以极大地简化生命周期和错误处理。

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> fetchUser());
    Future<Integer> order = scope.fork(() -> fetchOrder());

    scope.join(); // 等待所有子任务
    scope.throwIfFailed(); // 统一处理异常

    return new Result(user.resultNow(), order.resultNow());
}
```

## 监控与调优

- 关注 **Pinning 事件**，可通过 JFR（`jdk.VirtualThreadPinned`）监控。
- 根据应用负载特性，考虑调整载体线程池的并行度（`parallelism`）和最大大小（`maxPoolSize`）。
- 使用以下 JVM 参数进行诊断：

```bash
-Djdk.tracePinnedThreads=full \
-Djdk.virtualThreadScheduler.parallelism=32 \
-Djdk.virtualThreadScheduler.maxPoolSize=512
```

# 总结

1. **虚拟线程本质**：是 JVM 管理的轻量级用户态线程，通过 " 阻塞即卸载 " 的机制，用少量载体线程支撑海量并发。
2. **调度核心**：采用**非抢占式调度**，依赖阻塞点让出执行权，因此**只适用于包含阻塞操作的任务**。
3. **正确使用**：将其用于 IO 密集型服务；避免在 CPU 密集型任务和关键路径中使用 `synchronized`；优先采用 `Executors.newVirtualThreadPerTaskExecutor()` 和 `StructuredTaskScope` 进行管理。
4. **性能关键**：警惕 Pinning 问题，优先使用 `ReentrantLock` 替代 `synchronized`，并做好监控。

虚拟线程的引入不是对现有并发模型的颠覆，而是在特定场景下的极大增强。它让 Java 在高并发 IO 领域拥有了与 Go 等语言的协程相匹敌的能力，同时保持了对现有生态的最大兼容性。
