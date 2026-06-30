# 10、锁与 synchronized 底层

`synchronized`、锁升级与对象头 **Mark Word** 紧密相关（见第 3 章对象布局）。

## 1、synchronized 用法与锁对象

| 位置 | 锁对象 |
| ---- | ------ |
| 实例方法 | 当前实例 `this` |
| 静态方法 | 类的 `Class` 对象 |
| 同步块 | 括号内指定对象 |

字节码：`monitorenter` / `monitorexit`（异常路径也会 monitorexit）。

## 2、Mark Word 与锁状态（64 位 HotSpot 概念）

Mark Word 同一块存储复用于：哈希码、GC 年龄、**锁状态**等。

| 锁状态 | 存储内容（概念） | 说明 |
| ------ | ---------------- | ---- |
| **无锁** | 哈希、GC 年龄等 | 未竞争 |
| **偏向锁** | 偏向线程 ID | 同一线程反复进入，几乎无开销 |
| **轻量级锁** | 指向栈中 Lock Record 的指针 | 有竞争但无实际阻塞，CAS |
| **重量级锁** | 指向 Monitor 的指针 | 阻塞、OS 互斥量 |

**锁升级路径（不可逆）**

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

- 偏向锁：另一个线程竞争 → 撤销偏向，升级轻量级。
- 轻量级锁：CAS 自旋失败、多线程激烈竞争 → 膨胀为重量级。
- **不会降级**（除 GC 等特殊情况清理 Mark Word）。

## 3、偏向锁的废弃（JDK 15+）

偏向锁优化「单线程反复加锁」，但：

- 撤销偏向成本高；
- 现代应用多线程、短生命周期锁多，偏向锁收益下降。

**JDK 15 起默认关闭**，**JDK 18 起移除**。面试答法：了解历史四态升级即可，当前 HotSpot 以 **轻量级 + 重量级** 为主。

```bash
# 历史版本可显式关闭偏向锁
-XX:-UseBiasedLocking
```

## 4、轻量级锁

1. 在线程栈创建 **Lock Record**，复制 Mark Word 到 Displaced Header。
2. CAS 尝试把对象 Mark Word 改为指向 Lock Record。
3. 成功则获得锁；失败则自旋或升级重量级。

## 5、重量级锁与 Monitor

对象关联 **ObjectMonitor**（C++），包含：

- `_owner`：持有锁的线程
- `_EntryList`：阻塞等待队列
- `_WaitSet`：调用 `wait()` 的线程

`synchronized` 与 `ReentrantLock` 不同：前者 JVM 内置，后者 JUC、可中断、公平锁、Condition。

## 6、锁优化（JIT）

| 优化 | 说明 |
| ---- | ---- |
| **锁消除** | 逃逸分析证明锁仅单线程使用，去 synchronized |
| **锁粗化** | 连续多次加锁合并为一次 |
| **自适应自旋** | 轻量级锁自旋次数动态调整 |

## 7、AQS 与 synchronized 对比（简表）

| | synchronized | ReentrantLock |
| - | ------------ | ------------- |
| 实现 | JVM Monitor | AQS + CAS |
| 释放 | 自动 | 需 finally unlock |
| 公平 | 非公平 | 可选公平 |
| 条件队列 | 单个 wait/notify | 多个 Condition |

## 8、常见面试题

**1. synchronized 是可重入的吗？**

是。同一线程可多次获取同一把锁，Mark Word / Monitor 记录重入次数。

**2. 两个线程同时锁 A→B 与 B→A 会怎样？**

死锁。与锁升级无关，需固定加锁顺序或超时锁。

**3. 为什么 wait/notify 要在 synchronized 里？**

依赖 Monitor 的 `_WaitSet`；非法状态抛 `IllegalMonitorStateException`。

**4. 对象头里 GC 年龄和锁冲突吗？**

Mark Word 复用位段，不同状态存不同含义；有锁时可能无哈希（未计算 hash 时）。
