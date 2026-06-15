# JVM

Java 虚拟机（JVM）知识点体系，覆盖**类加载、内存模型、对象布局、垃圾回收、字节码与执行引擎、调优排障**。面向 Java 后端面试与线上问题定位。

> 规范参考：[Java Virtual Machine Specification SE 8](https://docs.oracle.com/javase/specs/jvms/se8/html/)

## 目录

| 章节 | 主要内容 |
| ---- | -------- |
| [1、JVM 概述与类加载](./1、JVM概述与类加载.md) | JDK/JRE/JVM、执行流程、类加载过程、类加载器、双亲委派 |
| [2、运行时数据区](./2、运行时数据区.md) | 程序计数器、虚拟机栈、堆、方法区/元空间、直接内存 |
| [3、对象与内存布局](./3、对象与内存布局.md) | 对象创建、内存布局、指针压缩、逃逸分析 |
| [4、垃圾回收](./4、垃圾回收.md) | 可达性、引用类型、GC 算法、分代、收集器、GC 日志 |
| [5、字节码与执行引擎](./5、字节码与执行引擎.md) | Class 文件、字节码指令、解释执行与 JIT |
| [6、JVM 调优与故障排查](./6、JVM调优与故障排查.md) | OOM、常用命令、核心参数、调优思路 |
| [7、Arthas 诊断工具](./7、Arthas诊断工具.md) | 在线诊断、常用命令、典型排查场景 |
| [8、Java 内存模型](./8、Java内存模型.md) | JMM、happens-before、volatile、重排 |
| [9、字符串与常量池](./9、字符串与常量池.md) | 三种常量池、intern、JDK6/7/8 差异 |
| [10、锁与 synchronized 底层](./10、锁与synchronized底层.md) | Mark Word、锁升级、Monitor、偏向锁废弃 |
| [11、JVM 进阶专题](./11、JVM进阶专题.md) | JPMS、Agent、JFR、虚拟线程、GraalVM、Epsilon |

## 知识点覆盖清单

| 大类 | 覆盖章节 |
| ---- | -------- |
| 架构与执行流程 | 1、5 |
| 类加载 / 双亲委派 / JPMS | 1、11 |
| 运行时数据区 / 元空间 / 直接内存 / Code Cache | 2 |
| 对象创建 / 布局 / 逃逸分析 | 3 |
| 字符串 / 常量池 / intern | 9 |
| GC 算法 / 分代 / 全部主流收集器 / 日志 | 4 |
| JMM / volatile / happens-before | 8 |
| synchronized / 锁升级 | 10 |
| 字节码 / JIT / 内联 / invokedynamic | 5 |
| 调优参数 / OOM / jstat/jmap/jstack / MAT | 6 |
| Arthas / JFR / VisualVM | 6、7、11 |
| Agent / 类卸载 / 虚拟线程 / Native Image | 11 |

## 学习路线建议

1. **先建立全局图**：编译 → Class 文件 → 类加载 → 运行时数据区 → 执行引擎。
2. **再抓面试高频**：双亲委派、栈帧结构、分代 GC、G1、OOM 类型、`-Xms/-Xmx`。
3. **最后补实战与进阶**：GC 日志、`jstack`、`jmap`、Arthas、JFR；JMM、字符串池、锁升级、JPMS、虚拟线程。

## 与 Java 基础文档的关系

- [Java 开发环境搭建](../Java/1、Java开发环境搭建.md) 中已介绍 JDK/JRE/JVM 概念。
- [数组内存分析](../Java/4、数组.md)、[面向对象内存解析](../Java/5、面向对象编程.md) 中的堆/栈/方法区简图，是本系列第 2 章的入门铺垫。
