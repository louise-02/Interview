# 11、JVM 进阶专题

## 1、常见 JVM 实现对比

| 实现 | 说明 |
| ---- | ---- |
| **HotSpot** | OpenJDK / Oracle JDK 默认，C1/C2、G1/ZGC |
| **OpenJ9**（Eclipse） | 低内存占用，适合容器；IBM  heritage |
| **GraalVM** | 高性能 JIT、Native Image、多语言 |
| **Azul Zing / Prime** | C4 收集器、商业低延迟 |
| **Dragonwell / Tencent Kona 等** | 基于 OpenJDK 的发行版优化 |

面试一般默认 **HotSpot**；说清「JVM 是规范，HotSpot 是主流实现之一」即可。

## 2、JPMS 模块化（JDK 9+）与类加载

模块系统（`module-info.java`）在类路径之上增加 **模块路径**。

```
模块 A exports / opens 包
  → 模块 B requires A
  → 类加载仍基于 ClassLoader，但可读性由模块边界约束
```

**与类加载相关**

- **Bootstrap** 加载 `java.base` 等核心模块。
- **Platform ClassLoader**（JDK 9+ 取代 ExtClassLoader）加载平台模块。
- **App ClassLoader** 加载应用模块与 classpath。
- **Layer**：模块可在不同 Layer 中同名，类加载器隔离（类似 OSGi 思想）。

**破坏双亲委派的现代场景**：SPI + **ServiceLoader** + TCCL；Spring Boot **LaunchedURLClassLoader**；OSGi / JPMS Layer。

## 3、线程上下文类加载器（TCCL）

```java
Thread.currentThread().getContextClassLoader();
```

SPI 中：接口由 Bootstrap 加载，实现类在 classpath，需 TCCL（常为 AppClassLoader）加载实现类。

## 4、类的卸载

类被卸载条件（同时满足，极难发生）：

1. 该类的所有实例已回收。
2. 加载该类的 ClassLoader 已回收。
3. 该类对应的 `Class` 对象无任何引用。

**典型**：自定义 ClassLoader 加载临时类，Loader 置 null，Full GC 后可能卸载。**应用类加载器加载的核心类几乎不会卸载**。

## 5、Java Agent 与 Instrumentation

**启动时 Agent**：`-javaagent:agent.jar`

**Attach Agent**：运行时加载（Arthas、部分 APM）。

```java
public class MyAgent {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer(new MyClassFileTransformer());
    }
}
```

用途：字节码增强（APM、热部署、Mock）、`getObjectSize()`、redefine/retransform 类。

**限制**：不能改已加载类的 schema  arbitrarily；retransform 有约束。

## 6、JFR（Java Flight Recorder）

JDK 内置**低开销** profiling，JDK 11+ 默认部分可用，Oracle JDK 需商业许可历史已调整（OpenJDK 11+ 生产可用）。

```bash
# 启动时录制
-XX:StartFlightRecording=duration=60s,filename=rec.jfr

# 运行中
jcmd <pid> JFR.start name=rec settings=profile duration=60s filename=rec.jfr
jcmd <pid> JFR.dump name=rec filename=rec.jfr
```

**JDK Mission Control（JMC）** 分析：GC、锁、I/O、热点方法、分配。

与 Arthas 互补：JFR 适合**事后分析**与**长时间采样**。

## 7、VisualVM / jconsole

| 工具 | 能力 |
| ---- | ---- |
| **jconsole** | JMX：堆、线程、MBean |
| **VisualVM** | 堆 dump、CPU 采样、插件（Visual GC 等） |
| **jcmd** | 脚本化诊断，替代部分 jmap/jstack |

JDK 9+：`jhsdb`（CLHSDB）分析 core dump。

## 8、虚拟线程（Virtual Threads，JDK 21+）

**平台线程**（1:1 OS 线程）占内存大；**虚拟线程**由 JVM 调度，大量阻塞 I/O 场景可百万级并发。

```java
Thread.startVirtualThread(() -> { /* ... */ });
Executors.newVirtualThreadPerTaskExecutor();
```

**注意**

- 虚拟线程**不要**池化（与平台线程池思想不同）。
- **pinning**：在 synchronized 或 Native 中阻塞可能钉住载体线程，影响伸缩；高并发 I/O 优先用 `ReentrantLock` 或重构 synchronized。
- 堆、GC 压力仍在，只是线程栈成本大降。

## 9、GraalVM 与 Native Image

**Graal JIT**：替代 C2 的深度优化编译器（`-XX:+UseJVMCICompiler` 等，视发行版）。

**Native Image**：AOT 编译为本地可执行文件，启动快、内存小；**反射 / 动态代理 / classpath 扫描**需配置 `reflect-config.json` 等；不适合所有 Spring 应用。

## 10、Epsilon GC（无操作收集器）

`-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`

**不回收**堆（分配直至 OOM）。用于：**性能基准**、极短生命周期进程、配合外部内存管理。

## 11、Safepoint 与 STW 再深入

- 线程运行到 **Safepoint** 才能 STW（方法调用、循环回边、异常跳转等）。
- **长时间 counted 循环**可能长时间不到 Safepoint → GC 停顿延迟（**TTSP 问题**）。
- **安全区域**：线程处于 Sleep / Blocked 且无法响应 Safepoint 时，先标记进入安全区域，GC 时可单独处理。

**偏向 Safepoint 的优化**：部分 GC 使用 **Colored Pointer / Load Barrier**（ZGC、G1）减少全堆 STW。

## 12、常见面试题

**1. 为什么 Tomcat 要自定义 ClassLoader？**

Web 应用隔离、热部署；同一容器多应用同名类不冲突。

**2. Agent 与 Arthas redefine 区别？**

都是 Instrumentation；Arthas 封装交互与诊断命令。

**3. 虚拟线程会替代线程池吗？**

I/O 密集型可大量用虚拟线程；CPU 密集型仍用有限平台线程 + 队列。

**4. Native Image 和 C 编译程序比？**

启动与内存更优；峰值吞吐、动态特性、兼容性需权衡。
