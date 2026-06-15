# Java 并发面试题

本文件可**单独阅读**，每题含完整参考答案。

---

## 一、基础概念

### 1. 进程和线程区别？

**进程**：资源分配单位，独立内存空间，有 PID。

**线程**：CPU 调度单位，共享进程堆、方法区，有独立栈、PC。线程切换开销小于进程。Java 线程映射 OS 线程（平台线程），JDK21+ 可有虚拟线程。

---

### 2. 并行和并发？

**并发 Concurrency**：单核/multiple 任务交替执行，逻辑上同时。

**并行 Parallelism**：多核同时执行，物理上同时。多线程在多核上可并行。

---

### 3. 创建线程的方式？

继承 Thread（不推荐，单继承）；实现 Runnable/Callable；**线程池**（推荐）；`Executors` 或 `ThreadPoolExecutor`；JDK21 `Thread.startVirtualThread()`。

---

### 4. 线程有哪些状态？

NEW → RUNNABLE（就绪/运行）→ BLOCKED（等 monitor 锁）→ WAITING（wait、join）→ TIMED_WAITING（sleep、带超时 wait）→ TERMINATED。

---

### 5. sleep 和 wait 区别？

| | sleep | wait |
| - | ----- | ---- |
| 类 | Thread | Object |
| 锁 | 不释放 | 释放 |
| 唤醒 | 时间到 | notify/notifyAll |
| 使用 | 任意 | 必须在 synchronized 内 |

---

### 6. start 和 run？

`start()` 启动新线程，JVM 调用 run()。直接 `run()` 只是普通方法调用，在当前线程执行。

---

## 二、锁与 synchronized

### 7. synchronized 原理？

字节码 `monitorenter/monitorexit`。对象头 Mark Word 关联 **Monitor**，互斥进入。可重入（计数）。锁升级：偏向→轻量 CAS→重量（阻塞）。

---

### 8. synchronized 和 ReentrantLock 对比？

| | synchronized | ReentrantLock |
| - | ------------ | ------------- |
| 实现 | JVM Monitor | AQS |
| 释放 | 自动 | finally unlock |
| 公平 | 非公平 | 可选公平 |
| 功能 | 基础 | tryLock、可中断、多 Condition |

---

### 9. 什么是可重入锁？

同一线程可多次获取同一把锁，Monitor/ AQS state 计数，减到 0 释放。避免死锁自己。

---

### 10. 死锁四个必要条件？如何排查？

互斥、占有且等待、不可剥夺、循环等待。破坏任一即可避免。排查：`jstack` 输出 `Found one Java-level deadlock`。预防：固定加锁顺序、tryLock 超时、银行家算法（少用）。

---

### 11. volatile 作用？能保证原子吗？

保证**可见性**（写刷主存、读最新）和**禁止特定重排**。不保证复合运算原子性，`i++` 需 AtomicInteger 或锁。

---

### 12. 什么是 happens-before？

JMM 规则：若 A happens-before B，则 A 对 B 可见。包括：程序次序、unlock happens-before lock、volatile 写 happens-before 读、线程 start/join 等。

---

### 13. DCL 单例为什么要 volatile？

`new` 对象非原子：分配→初始化→赋值可能重排，其他线程可能看到未初始化完的对象。volatile 禁止重排。

---

## 三、ThreadLocal

### 14. ThreadLocal 原理？

每个 Thread 有 ThreadLocalMap，key 为 ThreadLocal（弱引用），value 强引用。get/set 当前线程 map。实现线程隔离（如 SimpleDateFormat 旧用法、用户上下文）。

---

### 15. ThreadLocal 内存泄漏原因？

key 弱引用被 GC 后 entry 成 null key，但 **value 仍强引用**直到 remove。线程池线程长期存活导致泄漏。**用完必须 remove()**，尤其线程池场景。

---

## 四、线程池

### 16. 线程池核心参数？

corePoolSize、maximumPoolSize、keepAliveTime、unit、**workQueue**、threadFactory、**RejectedExecutionHandler**。

---

### 17. 线程池执行流程？

任务来 → 核心线程未满则创建 → 否则入队 → 队满则扩到 max → 仍满则拒绝策略。

---

### 18. 四种拒绝策略？

AbortPolicy 抛异常；CallerRunsPolicy 调用者线程跑；DiscardPolicy 丢弃；DiscardOldestPolicy 丢最老任务。

---

### 19. 为什么不用 Executors 创建线程池？

`FixedSingleThreadCached` 可能 **Integer.MAX_VALUE** 队列或线程数，OOM 风险。用 **ThreadPoolExecutor** 显式指定有界队列和 max。

---

### 20. 核心线程会回收吗？

默认不会。`allowCoreThreadTimeOut(true)` + keepAlive 可回收核心线程。

---

## 五、JUC

### 21. CAS 是什么？ABA 问题？

Compare And Swap  CPU 原子指令。ABA：值从 A 变 B 又变 A，CAS 认为未变。解决：`AtomicStampedReference` 带版本号。

---

### 22. AQS 原理？

AbstractQueuedSynchronizer：state + CLH 队列。ReentrantLock、Semaphore、CountDownLatch 基于 AQS。tryAcquire/release 模板方法。

---

### 23. CountDownLatch 和 CyclicBarrier？

CountDownLatch：倒数到 0 唤醒等待线程，**一次性**（如等多任务完成）。

CyclicBarrier：多线程到齐再一起走，**可重用**（如分阶段计算）。

---

### 24. Semaphore 用途？

信号量，控制同时访问资源的线程数，如限流、连接池逻辑。

---

### 25. ConcurrentHashMap JDK7 和 8？

7：Segment 分段锁。8：Node 数组 + CAS 插桶头 + synchronized 锁头节点，链表/树。size 用 baseCount + CounterCell。

---

### 26. CopyOnWriteArrayList 适用场景？

读多写少，写时复制新数组，读无锁弱一致。写开销大，如监听器列表。

---

## 六、虚拟线程与其它

### 27. 虚拟线程是什么？

JDK21 Project Loom，轻量线程由 JVM 调度，阻塞 I/O 时释放载体平台线程。适合高并发 I/O，**不要池化**虚拟线程。

---

### 28. 虚拟线程 pinning？

在 synchronized 或 Native 代码中阻塞可能 **pin** 到平台线程，失去伸缩性。高并发 I/O 优先 ReentrantLock 或重构 synchronized。

---

### 29. 如何保证线程安全？

不可变对象、线程封闭、锁、synchronized、JUC 原子类、并发容器、ThreadLocal（注意 remove）。

---

### 30. 乐观锁和悲观锁？

悲观：先加锁再操作（synchronized、数据库 for update）。

乐观：CAS/version 号，提交时检查冲突（Atomic、MyBatis 乐观锁字段）。

---

### 31. 线程池队列如何选择？

CPU 密集：小队列+线程≈核数。IO 密集：大队列+线程可多于核数。有界队列防 OOM，ArrayBlockingQueue、LinkedBlockingQueue（指定容量）。

---

### 32. Future 和 CompletableFuture？

Future get 阻塞取结果。CompletableFuture 支持链式 thenApply、thenCompose、allOf 组合异步，非阻塞编排。

---

### 33. 什么是 happens-before 与指令重排？

编译器和 CPU 为优化可重排指令，单线程 as-if-serial。多线程需 volatile、synchronized、并发工具保证可见性。

---

### 34. 读写锁 ReadWriteLock？

读读不互斥，读写、写写互斥。ReentrantReadWriteLock；读多写少提升吞吐。注意锁降级、写饥饿。

---

### 35. 如何设计一个线程安全的单例？

枚举单例（Effective Java 推荐）；静态内部类；DCL + volatile。避免双重检查不用 volatile 的写法。

---

### 36. park 和 unpark？

LockSupport，线程阻塞/唤醒，不要求先 park 再 unpark 顺序（与 wait/notify 不同），unpark 可先于 park。

---

### 37. 公平锁和非公平锁？

非公平：新来的可能插队，吞吐高。公平：FIFO，ReentrantLock(true)。synchronized 非公平。

---

### 38. 为什么 notify 要在 synchronized 里？

wait 释放锁并进入 WaitSet，notify 操作 Monitor 的 WaitSet，非法 monitor 状态抛 IllegalMonitorStateException。

---

### 39. 三个线程交替打印 ABC？

用 Lock + Condition 三个、或 Semaphore、或 BlockingQueue 传递令牌。考察 wait/notify 或 JUC 协调。

---

### 40. 如何停止线程？

旧 API stop/suspend 废弃。协作式：**volatile boolean flag**、interrupt() + 检查 `Thread.interrupted()` 或 `isInterrupted()`，阻塞方法抛 InterruptedException 时处理中断状态。
