# 7、Arthas 诊断工具

[Arthas](https://arthas.aliyun.com/) 是 Alibaba 开源的 **Java 在线诊断**工具：无需重启，attach 到目标 JVM，排查 CPU、内存、线程、类加载、方法耗时等问题。

## 1、安装与启动

```bash
# 下载并启动（会列出本机 Java 进程）
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 选择进程序号 attach
# 远程隧道、Docker 内 attach 等见官方文档
```

**注意**：生产 attach 会触发 **Safepoint**，短暂 STW；大堆 `heapdump` 影响更大，低峰操作。

## 2、常用命令速查

| 命令 | 作用 |
| ---- | ---- |
| `dashboard` | 线程、内存、GC 实时面板 |
| `thread` | 线程列表；`thread -n 3` 最忙 3 线程 |
| `jvm` | JVM 信息、类加载、GC 收集器 |
| `memory` | 堆各区域使用 |
| `heapdump /tmp/dump.hprof` | 导出堆（类似 jmap） |
| `sc` / `sm` | 查类 / 查方法 |
| `watch` | 观察方法入参、返回值、异常 |
| `trace` | 方法内部调用链耗时 |
| `stack` | 方法被谁调用 |
| `tt` | 时间隧道，记录调用，支持 replay |
| `ognl` | 执行 OGNL 表达式（改静态字段等，慎用） |
| `redefine` / `mc`+`redefine` | 热更新类（临时排查） |

## 3、典型场景

### 3.1 CPU 飙高

```bash
dashboard
thread -n 5
# 记下忙线程 id
thread <id>
# 看栈顶是否在业务循环、锁、GC
```

对比 `jstack`：Arthas 可**重复执行**、过滤 BUSY 线程，无需记 PID 手工换算 nid。

### 3.2 接口慢、定位慢方法

```bash
# 统计 com.example.service.OrderService createOrder 每层耗时
trace com.example.service.OrderService createOrder

# 只看耗时 > 100ms
trace com.example.service.OrderService createOrder '#cost > 100'
```

配合 `watch` 看参数与返回值：

```bash
watch com.example.service.OrderService createOrder '{params,returnObj,throwExp}' -x 2
```

### 3.3 确认方法是否被调用、调用路径

```bash
stack com.example.dao.OrderMapper selectById
```

### 3.4 类是否加载、Jar 来源

```bash
sc -d com.example.util.JsonUtil
# 显示 classLoaderHash、codeSource（哪个 jar）
```

排查 **NoClassDefFoundError**、**类冲突**（同一类多个版本）时有用。

### 3.5 内存与 OOM 前分析

```bash
memory
jvm
heapdump /tmp/app.hprof
```

下载 dump 后用 MAT 分析；在线还可：

```bash
vmtool --action getInstances --className java.util.HashMap --limit 10
```

（高级用法，需熟悉对象结构。）

### 3.6 动态修改日志级别（Logback）

```bash
ognl '@org.slf4j.LoggerFactory@getLogger("com.example").setLevel(@ch.qos.logback.classic.Level@DEBUG)'
```

临时打开 DEBUG，**重启后失效**。

## 4、trace 与 watch 区别

| 命令 | 侧重 |
| ---- | ---- |
| **trace** | 方法内部**调用树**与各节点耗时 |
| **watch** | 单方法**入参、出参、异常**观测 |
| **stack** | 谁**调用**了该方法 |

## 5、与 JDK 工具对比

| 能力 | jstack / jmap | Arthas |
| ---- | ------------- | ------ |
| 安装 | JDK 自带 | 独立 jar |
| 交互 | 单次快照 | REPL，可反复执行 |
| 方法级 | 无 | watch / trace / tt |
| 学习成本 | 低 | 中 |

建议：**Arthas 线上定位** + **MAT 离线分析 dump** + **GC 日志长期监控** 组合使用。

## 6、安全与生产规范

1. attach 权限：仅运维/开发有权限的服务器。
2. `ognl`、`redefine` 可能改运行状态，生产慎用。
3. `heapdump` 含堆数据，注意敏感信息与磁盘空间。
4. 排查完毕 `stop` 或 `quit` 断开，避免长时间 attach。

## 7、参考

- 官方文档：https://arthas.aliyun.com/doc/
- 命令列表：https://arthas.aliyun.com/doc/commands.html
