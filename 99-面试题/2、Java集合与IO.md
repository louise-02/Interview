# Java 集合与 IO 面试题

本文件可**单独阅读**，每题含完整参考答案。OOP、String、异常等见 [1、Java基础.md](./1、Java基础.md)。

深入学习 → [01-Java后端技术栈/Java](../01-Java后端技术栈/1、Java/0、readme.md)

---

## 一、集合框架

### 12. Collection 体系结构？

`Collection` → `List`（有序可重复）、`Set`（不重复）、`Queue`（队列）。`Map` 单独接口。常用：ArrayList、LinkedList、HashMap、HashSet、TreeMap、PriorityQueue、ConcurrentHashMap。

---

### 13. ArrayList 和 LinkedList 区别与复杂度？

**ArrayList**：动态数组。随机访问 O(1)，尾插 amortized O(1)，中间插删 O(n) 需移动元素。内存连续，缓存友好。

**LinkedList**：双向链表。头尾插删 O(1)，按索引访问 O(n)。每个节点额外指针开销。一般业务 List 优先 ArrayList。

---

### 14. HashMap 底层原理（JDK8）？

数组 + 链表 + 红黑树。put：算 hash → 桶下标 → 链表/树插入，key 相同则覆盖。负载因子 **0.75**，容量达阈值 **扩容 2 倍** rehash。链表长度 **≥8 且数组≥64** 转红黑树；≤6 退化为链表。非线程安全。

---

### 15. HashMap 的 hash 扰动做什么？

`(h = key.hashCode()) ^ (h >>> 16)`，让高 16 位参与下标计算 `(n-1)&h`，减少低位冲突。

---

### 16. HashMap 和 Hashtable、ConcurrentHashMap？

**Hashtable**：synchronized 全表锁，旧 API，不用。

**ConcurrentHashMap**：JDK7 分段锁 Segment；JDK8 **CAS + synchronized 锁桶头节点**，粒度更细。读大多无锁。size 用 baseCount + CounterCell。

---

### 17. HashSet 如何保证不重复？

内部 **HashMap**，元素作 key，固定 PRESENT 对象作 value。依赖 hashCode + equals。

---

### 18. TreeMap 和 HashMap？

TreeMap **红黑树**，key **有序**（Comparable 或 Comparator），O(log n)。HashMap 无序，O(1) 均摊。需要排序用 TreeMap。

---

### 19. fail-fast 和 fail-safe？

**fail-fast**：迭代时结构被改抛 ConcurrentModificationException（ArrayList 迭代器 check modCount）。

**fail-safe**：迭代副本（CopyOnWriteArrayList），弱一致，不抛异常。

---

## 二、IO 与 NIO

### 23. BIO、NIO、AIO 区别？

**BIO**：Blocking IO，一连接一线程，阻塞 read/write。

**NIO**：Channel + Buffer + Selector，多路复用，单线程管多连接，非阻塞模式。

**AIO**：异步 IO，操作系统回调；Linux 上 Java AIO 支持有限，Netty 常用 NIO。

---

### 24. 序列化和 transient？

实现 Serializable，ObjectOutputStream 写对象。`serialVersionUID` 版本不一致抛 InvalidClassException。`transient` 字段不序列化。静态字段不属于对象实例不序列化。
