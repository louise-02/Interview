# 6、JVM 调优与故障排查

## 1、OOM 类型与原因

| 异常 | 区域 | 常见原因 |
| ---- | ---- | -------- |
| `Java heap space` | 堆 | 内存泄漏、堆过小、大对象/缓存无上限 |
| `GC overhead limit exceeded` | 堆 | GC 时间占比过高，几乎无有效回收 |
| `Metaspace` / `PermGen space` | 元空间/永久代 | 类加载过多、CGLib 动态类、反射 |
| `Unable to create new native thread` | 线程 | 线程数超限、`-Xss` 过大导致可创建线程数减少 |
| `Direct buffer memory` | 直接内存 | DirectBuffer 未释放、Netty 堆外泄漏 |
| `StackOverflowError` | 栈 | 无限递归、栈帧过深 |

**排查思路**

1. 保留现场：GC 日志、`-XX:+HeapDumpOnOutOfMemoryError` 堆 dump。
2. 区分泄漏 vs 不足：dump 中大量重复对象 vs 业务正常大缓存。
3. 定位引用链：MAT / VisualVM / JProfiler 看 GC Roots 到泄漏对象的路径。

## 2、JDK 自带命令行工具

| 工具 | 作用 |
| ---- | ---- |
| `jps -l` | 列出 Java 进程 PID 与 main 类 |
| `jstat -gcutil <pid> 1000` | 每秒打印 GC 与堆各区占用 |
| `jinfo -flags <pid>` | 查看/修改部分 VM 参数 |
| `jmap -heap <pid>` | 堆配置与摘要 |
| `jmap -dump:live,format=b,file=heap.hprof <pid>` | 导出堆 dump |
| `jstack <pid>` | 线程栈，查死锁、阻塞 |
| `jcmd <pid> help` | JDK 7+ 统一诊断 |

**jstack 死锁示例输出**

```
Found one Java-level deadlock:
...
Java stack information for the threads listed above:
```

**jstat 列含义（gcutil）**

- S0/S1/E/O/M：Survivor、Eden、Old、Metaspace 使用率
- YGC/YGCT、FGC/FGCT：Young/Full GC 次数与总耗时

## 3、核心 JVM 参数

### 3.1 堆与元空间

```bash
# 堆：生产建议 Xms = Xmx，避免扩缩带来的抖动
-Xms4g -Xmx4g

# 年轻代（可选，G1 下通常只设堆与 MaxGCPauseMillis）
-Xmn1g
-XX:NewRatio=2          # 老年代:年轻代 = 2:1
-XX:SurvivorRatio=8     # Eden:单个Survivor = 8:1

# 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### 3.2 垃圾收集器

```bash
# JDK 8 吞吐量
-XX:+UseParallelGC

# JDK 8 G1
-XX:+UseG1GC -XX:MaxGCPauseMillis=200

# JDK 11+ 低延迟（按发行版与支持情况选用）
-XX:+UseZGC
# 或 -XX:+UseShenandoahGC
```

### 3.3 GC 日志与 OOM

```bash
# JDK 8
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/var/log/app/gc.log
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M

# JDK 9+
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# OOM 自动 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdump.hprof
```

### 3.4 线程与直接内存

```bash
-Xss512k                    # 线程栈，过小 StackOverflow，过大可创建线程数变少
-XX:MaxDirectMemorySize=1g
```

### 3.5 常用诊断开关

```bash
-XX:+DisableExplicitGC     # 禁止 System.gc() 触发 Full GC（慎用，某些框架依赖）
-XX:+PrintCommandLineFlags # 启动时打印实际生效参数
-XX:+UseCompressedOops     # 压缩普通对象指针（堆 < 32G 默认开）
-XX:+UseCompressedClassPointers
-XX:StringTableSize=1009   # 字符串常量池桶数（JDK 8+）
```

### 3.6 G1 / ZGC 补充参数

```bash
# G1
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1ReservePercent=10

# ZGC（JDK 21 分代）
-XX:+UseZGC -XX:+ZGenerational
-XX:ZCollectionInterval=5
```

## 4、JFR 与可视化工具

**JFR（Java Flight Recorder）**

```bash
-XX:StartFlightRecording=duration=120s,filename=app.jfr
jcmd <pid> JFR.start duration=60s filename=rec.jfr
jcmd <pid> JFR.dump filename=rec.jfr
```

用 **JDK Mission Control（JMC）** 打开 `.jfr`：GC、锁、热点、异常、I/O。

**VisualVM / jconsole**

- 连接本地或 JMX 远程进程，查看堆、线程、CPU 采样。
- VisualVM 插件 **Visual GC** 观察各区变化。
- JDK 9+ 部分功能由 **jhsdb**、**jcmd** 补充。

**jcmd 常用**

```bash
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print    # 等同 jstack
jcmd <pid> GC.class_histogram
```

详见 [11、JVM 进阶专题](./11、JVM进阶专题.md)。

## 5、生产调优思路

### 5.1 内存占比（与容器配合）

| 场景 | 建议 |
| ---- | ---- |
| 裸机 / 固定内存 | 堆约 **物理内存 50%~70%**，留 OS、Metaspace、Direct、线程栈 |
| K8s 容器 | `-Xmx` ≤ **容器 memory limit 的 70%~75%**，避免 OOMKilled |
| 4G 容器示例 | `-Xms2g -Xmx2g`，Metaspace 256~512m |

### 5.2 选择收集器

| 目标 | 建议 |
| ---- | ---- |
| 批处理、吞吐 | Parallel / G1 默认停顿目标放宽 |
| Web、API 低延迟 | G1，`MaxGCPauseMillis=100~200` |
| 大堆、极低停顿 | ZGC / Shenandoah |

### 5.3 调优步骤

1. **基线**：GC 日志、吞吐、P99 延迟、Full GC 频率。
2. **定位瓶颈**：频繁 Full GC → 堆或泄漏；长 STW → 收集器或 Region 设置；Metaspace 涨 → 类加载。
3. **小步调整**：一次改少量参数，压测对比。
4. **避免**：盲目增大堆掩盖泄漏；生产随意 `jmap -dump` 大堆（STW，用 `live` 或 Arthas heapdump）。

## 6、MAT 分析堆 dump（要点）

1. **Histogram**：按类统计实例数与占用。
2. **Dominator Tree**：谁占内存最多。
3. **Leak Suspects**：自动可疑报告。
4. **Path to GC Roots**：为何无法回收（exclude 弱/软引用看强引用链）。

典型泄漏：静态集合无限增长、ThreadLocal 未 remove、连接/流未关闭、缓存无淘汰。

## 7、CPU 100% 排查

```bash
# 1. 找 Java 进程
top -Hp <pid>

# 2. 高 CPU 线程转 16 进制
printf "%x\n" <tid>

# 3. jstack 中搜 nid
jstack <pid> | grep -A 30 <hex_tid>
```

常见：死循环、正则回溯、频繁 GC 线程、锁竞争。

## 8、常见面试题

**1. `-Xms` 和 `-Xmx` 为什么要设成一样？**

避免运行期扩容/收缩堆带来的性能抖动与 Full GC 风险。

**2. 如何排查内存泄漏？**

重复 Full GC 后老年代仍涨 → dump → MAT 看 Dominator 与 GC Roots 引用链 → 查代码（集合、ThreadLocal、监听器未注销）。

**3. `System.gc()` 会怎样？**

建议 Full GC；生产常 `-XX:+DisableExplicitGC`，但 NIO DirectBuffer 清理等场景可能依赖，需评估。

**4. 容器里 JVM 如何感知内存限制？**

JDK 8u191+ / 10+ 识别 cgroup；使用 `-XX:+UseContainerSupport`（新版默认），**不要**只设 `-Xmx` 等于容器 limit。

## 9、相关文档

- 线上诊断：[7、Arthas 诊断工具](./7、Arthas诊断工具.md)
- 部署 JDK 参数示例：[JDK 配置与运维](../../../05-运维和部署/Deploy/2、jdk.md)
- JMM / 锁：[8、Java 内存模型](./8、Java内存模型.md)、[10、锁与 synchronized 底层](./10、锁与synchronized底层.md)
