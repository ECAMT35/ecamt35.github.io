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

JDK 21 正式发布虚拟线程（Virtual Thread，JEP 444）。这是 Java 并发模型的一项重要演进，目标是在保持现有线程编程模型基本不变的前提下，以更低的线程成本支持更高的并发规模。它尤其适用于 IO 密集型应用，可在大量请求处于等待状态时提高系统整体吞吐，并改善 CPU 资源利用率，同时保持与现有 `Thread` API 的兼容性。

# 传统并发模型的瓶颈

在虚拟线程出现之前，Java 线程主要对应平台线程（Platform Thread），其本质是对操作系统内核线程的一对一封装，生命周期和调度都由操作系统内核负责。

这种模型在高并发、长时间阻塞的场景中存在几个明显限制：

- **资源成本高**：每个线程都需要分配独立的栈内存（通常 MB 级别），创建数千个线程就会消耗大量内存，也容易接近操作系统允许的线程数量上限。
- **调度开销大**：平台线程的创建、销毁和上下文切换都涉及系统调用以及用户态/内核态切换，开销较大。
- **并发能力受限**：为控制资源消耗，服务端应用通常采用线程池。例如 Tomcat 默认工作线程数为 200。若这些线程大部分都阻塞在数据库、网络或其他外部 IO 上，即使 CPU 仍有空闲，新的请求也无法被处理，形成瓶颈。

# 虚拟线程的核心设计

虚拟线程是 **JVM 实现的轻量级用户态线程**。它不再与 OS 线程绑定，而是由 JVM 在运行时将其调度到少量**载体线程**（Carrier Thread）上执行。

其核心特性如下：

- **轻量开销低**：栈内存可动态伸缩，初始占用极小，因此可以在单个 JVM 中创建远多于平台线程数量的线程实例。
- **高效调度**：当虚拟线程执行可识别的阻塞操作（如 IO、`Thread.sleep`）时，JVM 会将其**挂起**，并释放其占用的载体线程，该载体线程可立即去执行其他就绪的虚拟线程。这一切都在**用户态**完成，规避了内核切换的开销。
- **无缝兼容**：`java.lang.Thread` API 保持不变，现有代码几乎无需修改即可使用虚拟线程。

这种设计的重点不在于让单个任务执行更快，而在于让大量“经常阻塞”的任务能够同时存在，并尽量减少线程本身带来的资源负担。

# 调度原理
## 非抢占式（协作式）用户态调度

平台线程由操作系统进行基于时间片的**抢占式调度**。虚拟线程则不同：JVM 对其采用的是**非抢占式（协作式）用户态调度**机制，虚拟线程通常只有在到达特定阻塞点时才会让出执行权。

这意味着：

- 遇到阻塞点时，虚拟线程会被挂起并卸载；
- 不包含阻塞点的纯 CPU 计算代码不会主动让出载体线程；
- 因而虚拟线程并不适合替代 CPU 密集型任务的执行模型。

下面的示例中，先创建与 CPU 核心数相同的虚拟线程，每个线程都持续循环，但循环内部包含 `Thread.sleep(1)` 阻塞点：

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

这段代码中，前面的虚拟线程虽然在无限循环，但由于循环中包含 `sleep`，它们会周期性地进入阻塞状态并释放载体线程，因此后创建的虚拟线程仍然有机会被调度执行，输出 `"New virtual thread"`。

然而，将 `sleep` 移除后，使其变为纯 CPU 循环，情况会发生变化：

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

这时 `"New virtual thread"` 永远不会输出。这是因为纯 CPU 循环没有阻塞点，虚拟线程永远不会主动让出执行权，从而独占载体线程。

这就是非抢占式调度：虚拟线程之间需要 " 协商 " 何时释放资源，而协商的契机就是阻塞操作。

这种调度模式的**好处**是不需要 JVM 定期中断操作，避免了额外的性能开销。**坏处**是如果有 11 个持续运行且不阻塞的计算任务（即 11 个虚拟线程）要运行在一个 10 线程的 CPU 上，其中一个任务就会持续无法被调度运行。

## 载体线程池与工作窃取算法

虚拟线程默认由 JVM 全局调度器负责调度，底层使用 `ForkJoinPool` 作为载体线程池。

- **并行度**：池中活跃载体线程数默认等于 CPU 核心数，可通过 `-Djdk.virtualThreadScheduler.parallelism=<N>` 调整。
- **工作窃取**：每个载体线程拥有一个本地任务队列。当自身队列为空时，它会从其他线程的队列尾部 ”窃取“ 任务，以此实现高效的负载均衡。
- **挂载与卸载**：就绪的虚拟线程被挂载到空闲载体线程上运行；遇到阻塞点时，虚拟线程被卸载，载体线程被释放回池。该虚拟线程重新进入调度队列，等待再次运行。

**重要边界**：虚拟线程的“海量并发”主要体现在“可同时存在大量阻塞中的任务”。如果同时有大量虚拟线程都处于持续运行的非阻塞状态，那么它们仍然会受载体线程数量限制。默认载体线程池最大规模为 `256`，可通过以下参数调整：

```bash
-Djdk.virtualThreadScheduler.maxPoolSize=<N>
```

这意味着，如果同时处于活跃运行状态的非阻塞虚拟线程数量超过载体线程池容量，超出的部分只能排队等待。

# 适用场景与使用边界

虚拟线程是 **IO 密集型任务**的利器，而非所有并发问题的万能解药。

| 推荐场景                         | 优势                          |
| :--------------------------- | :-------------------------- |
| Web 服务器 (Spring Boot/Tomcat) | 支持海量并发连接，无需复杂线程池调优          |
| 微服务 RPC 调用链                  | 每个调用可分配独立虚拟线程，避免线程池耗尽导致级联故障 |
| 数据库、缓存、HTTP 客户端              | 阻塞等待期间线程可卸载，提升整体吞吐          |
| 消息队列消费者                      | 可用较低线程成本实现高并发消费模型           |

**需谨慎或避免的场景**：

- **CPU 密集型任务**：这类任务通常很少阻塞，虚拟线程无法发挥“阻塞即卸载”的优势。此时更适合使用固定线程池、`ForkJoinPool` 或针对计算任务设计的并行框架。
- **虚拟线程池化**：虚拟线程创建成本已经很低，通常不需要再维护专门的线程池来复用它们。池化反而会引入额外复杂度。
- **在 `synchronized` 块内阻塞**：可能导致 **Pinning**（钉住），使虚拟线程在阻塞时无法从载体线程卸载，降低并发能力。**应优先使用 `ReentrantLock`**。（注：JDK 24 已优化此问题，但未完全根除）。
- **阻塞型 Native / JNI 调用**：若本地方法未适配虚拟线程，其阻塞同样可能导致载体线程被持续占用。

## 注意事项
### Pinning 问题

什么是 Pinning？
Pinning 指的是：虚拟线程在阻塞时无法从载体线程卸载，导致该载体线程在阻塞期间一直被占用，不能调度其他虚拟线程。
虚拟线程的伸缩性依赖于“阻塞时释放载体线程”。一旦发生 Pinning，这一机制就会失效，系统吞吐量可能显著下降。

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

为什么 synchronized 可能导致 Pinning？
`synchronized` 的底层基于对象监视器（monitor）实现，这套机制诞生较早，其语义和实现长期围绕平台线程建立。在虚拟线程场景下，问题主要来自以下几个方面：

1. **监视器与平台线程绑定**：Java 的监视器机制与平台线程深度耦合。每个对象都有一个关联的监视器，一次只能有一个线程持有对象的监视器。在 JVM 内部，监视器所有权是**在平台线程（内核线程）层面跟踪的**，而不是在虚拟线程层面。当虚拟线程进入 synchronized 块时，其**载体线程**（平台线程）被记录为监视器的所有者，而不是虚拟线程本身。
    
2. **卸载会破坏监视器语义的一致性**：假如虚拟线程在 synchronized 块内被卸载，其载体线程会被释放回线程池，可能被分配给其他虚拟线程使用。但 JVM 仍然记录该载体线程持有监视器，这会导致：
    
    - 新的虚拟线程（使用同一载体线程）被错误地认为持有监视器
    - 其他试图获取同一监视器的虚拟线程可能被不当阻塞
    - 这一假设破坏了 synchronized 的互斥语义，因此 JVM 必须阻止虚拟线程在持有监视器时卸载
    
3. **wait/notify 机制**：`Object.wait()` 要求在同一线程上释放和重新获取监视器。如果虚拟线程在执行 synchronized 方法时调用了 `Object.wait()`，并且虚拟线程在等待期间卸载，当被 `notify()` 唤醒时，它可能被调度到不同的载体线程上，无法满足 “同一线程重新获取监视器”的要求，因此也会被固定。

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
    
3. **监控与诊断**。可以通过以下方式观察 Pinning：
	
    - 使用 JFR 监控 `jdk.VirtualThreadPinned` 事件
    - 通过 `-Djdk.tracePinnedThreads=full` 参数追踪 Pinning
    
4. **JDK 24+ 的改进**  
    JDK 24 通过 JEP 491 大幅减少了 synchronized 导致的 Pinning，但在递归锁等复杂场景下仍可能存在。

### ThreadLocal 的使用问题

虚拟线程与普通线程一样，支持 `ThreadLocal`，会维护一个自己的 ThreadLocalMap，**每访问某个 `ThreadLocal` 的线程，都会持有自己独立的一份线程本地值**。但在虚拟线程场景下，`ThreadLocal` 的使用方式需要重新评估。

在传统平台线程数量有限的情况下，这种“每线程一份”的模型通常仍然可控；但虚拟线程的设计目标是支持远高于平台线程数量的并发规模。如果系统中存在大量虚拟线程，并且每个线程都通过 `ThreadLocal` 保存用户上下文、认证信息、数据库会话或其他较重对象，那么这些对象会随着线程数增加而线性累积，从而显著放大整体内存占用，带来较大的内存膨胀问题。

因此，在虚拟线程环境下，`ThreadLocal` 的主要风险并不只是传统意义上的“泄漏”，更常见的问题是**大量线程本地状态带来的整体内存膨胀**。

对于只需要在受控调用范围内传递上下文、且数据本身应保持不可变的场景，更适合使用 `ScopedValue`（JDK 21+ 预览，JDK 25 稳定）。其设计目标不是为线程附着一个可变本地状态，而是提供一种**按作用域绑定的上下文传递机制**。它可以理解为“隐式方法参数”：在某个调用作用域内，将一个值绑定到当前线程；当该作用域结束时，这个绑定自动失效（如果没有其他强引用，则**具备 GC 条件**），而不会继续附着在线程整个生命周期上。

```java
private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();  
  
ScopedValue.where(USER_ID, "user123").run(() -> {  
    // 在此作用域内，USER_ID.get() 返回 "user123"  
});
```

上例中，`USER_ID` 是一个静态共享的键，但绑定的值并不是全局唯一的。不同线程、不同作用域可以同时绑定不同的值，因此它适合表示“当前请求的用户 ID”“当前租户”“traceId”这类请求级上下文。`ScopedValue.where(...).run(...)` 本身是同步执行的：它会在当前线程中执行作用域内的代码，待代码执行结束后自动解除绑定。

二者的区别在于生命周期上：`ThreadLocal` 的值默认随线程存活，而 `ScopedValue` 的绑定默认随作用域存活。前者更接近“线程本地状态”，后者更接近“调用链上的上下文”。这并不意味着 `ScopedValue` 不占用内存，而是意味着它更不容易因为线程数量巨大或状态长期滞留而放大资源成本。OpenJDK 对它的定位也是：相比 `ThreadLocal`，`ScopedValue` 更容易推理，且在与虚拟线程配合时通常具有更低的空间和时间成本。

因此，在虚拟线程环境下可以采用一个更清晰的原则：如果需要的是**线程私有、可变、且确实应与线程生命周期绑定的状态**，可以考虑使用 `ThreadLocal`；如果需要的是**沿调用链传递的只读上下文**，则应优先考虑 `ScopedValue`。对于高并发、短生命周期的虚拟线程应用，后者通常更符合资源模型和代码语义。

# 最佳实践

## 创建方式

**简单启动**：

```java
Thread.startVirtualThread(() -> {
});
```

**构建器模式**（支持命名、配置）：

```java
Thread vt = Thread.ofVirtual().name("my-vt").unstarted(task);
vt.start();
```

**执行器服务**（推荐，便于管理）：

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
Future<String> future = executor.submit(() -> "Hello");

// 这种方式更便于任务管理、结果收集、关闭资源以及与现有并发框架对接。
//其语义不是“维护一个可复用线程池”，而是“每个任务分配一个新的虚拟线程”。
```

## 结构化并发

当一个业务操作需要派生多个并发子任务，并在结束时统一收集结果或统一处理失败时，结构化并发（Structured Concurrency）会比手工管理多个 `Future` 更清晰。

`StructuredTaskScope` 在 JDK 21 中为预览特性，在 JDK 23 中转为稳定特性。示例：

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> fetchUser());
    Future<Integer> order = scope.fork(() -> fetchOrder());

    scope.join(); // 等待所有子任务
    scope.throwIfFailed(); // 统一处理异常

    return new Result(user.resultNow(), order.resultNow());
}
```

这种写法的核心价值在于：

- 子任务生命周期被限定在父作用域内；
- 子任务失败可统一传播；
- 父任务取消时，子任务也能协同处理；
- 并发逻辑与顺序代码结构更接近，便于维护。

虚拟线程与结构化并发通常是配套使用的：前者降低并发执行成本，后者规范并发组织方式。

## 监控与调优

在虚拟线程环境下，调优的重点通常不是线程池大小本身，而是：

- 是否出现 Pinning；
- 是否有大量任务长期不阻塞、持续占用载体线程；
- 是否存在不适配虚拟线程的库或本地调用；
- 载体线程并行度是否与工作负载匹配。

常见参数如下：

```bash
-Djdk.tracePinnedThreads=full \
-Djdk.virtualThreadScheduler.parallelism=32 \
-Djdk.virtualThreadScheduler.maxPoolSize=512
```

含义分别是：

- `tracePinnedThreads`：打印虚拟线程被钉住的位置；
- `parallelism`：设置调度器目标并行度；
- `maxPoolSize`：限制载体线程池允许增长到的最大规模。

是否需要调高这些参数，应依据应用负载特点与监控数据判断，而不应机械套用。

# 总结

1. **虚拟线程本质**：是 JVM 管理的轻量级用户态线程，它通过“阻塞时卸载、恢复时重新调度”的机制，让少量载体线程支持大量并发任务。
2. **性能收益的来源**：采用**非抢占式调度**，依赖阻塞点让出执行权，收益主要来自 IO 阻塞期间不再长期占用平台线程，因此特别适合高并发、长等待、短计算的任务模型。
3. **调度上的关键限制**：虚拟线程不是对 CPU 密集型任务的通用加速方案。没有阻塞点的任务不会主动让出载体线程，因此无法体现其主要优势。
4. **使用时需要关注的问题**：需要避免在 `synchronized` 块内执行阻塞操作，关注 JNI/Native 阻塞问题，警惕 `ThreadLocal` 在超大线程规模下的内存成本，并通过 JFR 或诊断参数监控 Pinning。
5. **推荐实践**：在 IO 密集型系统中，可以优先考虑使用 `Executors.newVirtualThreadPerTaskExecutor()` 以及 `StructuredTaskScope` 来组织并发逻辑；对于锁场景，优先选择更适合虚拟线程的显式锁实现；对于上下文传递，可评估 `ScopedValue`。

虚拟线程并没有改变 Java 对线程式编程模型的基本抽象，而是在兼容原有生态的基础上，显著降低了“线程作为并发单元”的成本。它最适合用于以阻塞式代码表达 IO 并发的应用场景，在这些场景下，能够以更低复杂度获得更高的并发能力。

