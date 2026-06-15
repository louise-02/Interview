# 8、Java 内存模型（JMM）

JMM（Java Memory Model）规定多线程下 **共享变量的可见性、有序性、原子性**，与第 2 章「运行时数据区」是不同层次：

- **运行时数据区**：JVM 内存**怎么划分**（堆、栈、元空间）。
- **JMM**：线程如何**读写共享变量**（主内存与工作内存的交互规则）。

## 1、主内存与工作内存

每个线程有自己的 **工作内存**（逻辑概念，对应栈、CPU 缓存等），保存用到的共享变量副本。

```
        主内存（Main Memory）= 堆上的共享变量
              ↑ read / write
    ┌─────────┼─────────┐
  线程1工作内存  线程2工作内存  线程3工作内存
```

线程对变量操作必须在工作内存进行，不能直接读写主内存（逻辑模型；实际由 JVM + CPU 实现）。

## 2、三大特性

| 特性 | 含义 | 典型手段 |
| ---- | ---- | -------- |
| **原子性** | 操作不可被中断 | `synchronized`、`Lock`、`Atomic*` |
| **可见性** | 一个线程修改，其他线程能立刻看到 | `volatile`、`synchronized`、`final` |
| **有序性** | 禁止/限制指令重排 | `volatile`、`synchronized`、happens-before |

**注意**：`volatile` 保证可见性与有序性，**不保证**复合运算的原子性（如 `i++`）。

## 3、happens-before 规则

若 A happens-before B，则 A 的结果对 B **可见**，且 A 在 B **之前**执行（逻辑序）。

JMM 无需程序员记所有重排，只要遵守以下规则：

| 规则 | 说明 |
| ---- | ---- |
| **程序次序** | 同一线程中，前面的操作 hb 后面的操作 |
| **监视器锁** | 解锁 hb 后续对同一锁的加锁 |
| **volatile** | 写 volatile hb 后续对该变量的读 |
| **线程启动** | `Thread.start()` hb 该线程内任意操作 |
| **线程终止** | 线程内操作 hb 其他线程检测到该线程已终止 |
| **传递性** | A hb B，B hb C → A hb C |
| **中断** | `interrupt()` hb 检测到中断 |
| **对象终结** | 构造器结束 hb `finalize()` 开始（已 deprecated） |

## 4、volatile

**实现原理（HotSpot）**

- 写 volatile：强制刷回主内存，并使其他 CPU 缓存行失效（MESI 等）。
- 读 volatile：从主内存或最新缓存读。
- 通过 **内存屏障**（LoadLoad、StoreStore、LoadStore、StoreLoad）限制重排。

**适用场景**

- 状态标志：`volatile boolean stop`
- 双重检查锁（DCL）单例中的实例引用
- 一写多读的无锁配置

**不适用**

- 依赖当前值写回：`count++` 需 `AtomicInteger` 或锁。

```java
// 错误：volatile 不能保证 i++ 原子
volatile int i;
void inc() { i++; }

// 正确
AtomicInteger count = new AtomicInteger();
void inc() { count.incrementAndGet(); }
```

## 5、指令重排

编译器、CPU 为优化性能可能**重排**指令，单线程结果不变，多线程可能看到「半初始化」对象。

**经典 DCL 单例（需 volatile）**

```java
public class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 非原子：分配→初始化→赋值，可能重排
                }
            }
        }
        return instance;
    }
}
```

`new` 对象步骤可能被重排，其他线程可能看到未初始化完的对象；`volatile` 禁止重排。

## 6、synchronized 的 JMM 语义

- **加锁**：清空工作内存，从主内存加载最新值。
- **解锁**：把工作内存刷新到主内存。

因此 synchronized 同时保证 **互斥、可见性、有序性**（块内 as-if-serial）。

## 7、final 的可见性

final 字段在构造器完成前对其他线程不可见；构造器结束后，final 字段对其他线程可见（无需额外同步），前提是构造期间 `this` 未逸出。

## 8、与运行时数据区的关系

| 问题 | 用哪个概念 |
| ---- | ---------- |
| 对象在堆还是栈 | 运行时数据区 + 逃逸分析 |
| 线程为何看不到别的线程改的变量 | JMM / happens-before |
| ThreadLocal 数据在哪 | 线程私有，堆上 Entry 对象 |
| 锁竞争、锁升级 | 对象头 Mark Word（见第 10 章） |

## 9、常见面试题

**1. JMM 和 JVM 内存结构是一回事吗？**

不是。JMM 是并发语义；JVM 内存结构是物理/逻辑划分。

**2. volatile 和 synchronized 区别？**

volatile：不互斥，轻量，一写多读；synchronized：互斥 + 可见 + 可复合操作。

**3. 什么是 as-if-serial？**

单线程内重排不影响最终结果；多线程需 happens-before 约束。
