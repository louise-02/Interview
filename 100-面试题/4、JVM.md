# JVM 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

## 一、内存结构

### 1. 说说 JVM 运行时数据区？

**线程私有**：程序计数器（当前字节码行号）、虚拟机栈（方法栈帧、局部变量、操作数栈）、本地方法栈（Native 方法）。

**线程共享**：堆（对象、数组）、方法区/元空间（类元信息、常量、静态变量）、运行时常量池。

**直接内存**：堆外，NIO DirectBuffer，受 MaxDirectMemorySize 限制，不属于规范定义区域但常一起调优。

---

### 2. 堆和栈分别存什么？异常有哪些？

**栈**：局部变量表（含引用变量）、操作数栈、方法返回地址。`StackOverflowError` 递归过深；线程过多 `OutOfMemoryError` 无法扩展栈。

**堆**：几乎所有 new 出来的对象和数组。`OutOfMemoryError: Java heap space`。

成员变量：实例字段在堆（随对象）；static 在方法区/元空间（JDK7+ 静态变量对象在堆，类元在元空间）。

---

### 3. JDK7/8 永久代和元空间区别？

JDK7：字符串常量池、静态变量移到**堆**；类元仍在永久代 PermGen，固定大小易 OOM。

JDK8+：**Metaspace 元空间**用**本地内存**存类元数据，默认仅受物理内存限制，用 `-XX:MaxMetaspaceSize` 限制。字符串池在堆 String Table。

---

### 4. 程序计数器有什么特点？

线程私有，记录当前线程执行的字节码地址（Native 方法时为 undefined）。线程切换时恢复执行位置。**唯一不会 OOM 的区域**。

---

### 5. 什么是 TLAB？

Thread Local Allocation Buffer：Eden 中每个线程私有分配缓冲区，**bump-the-pointer** 无锁分配小对象，减少竞争。用尽后在 Eden 同步分配或触发 Minor GC。

---

## 二、类加载

### 6. 类加载过程？

**加载**：读 class 字节流，生成 Class 对象。

**链接**：验证（格式、字节码、符号引用）→ 准备（static 零值，final 常量赋终值）→ 解析（符号引用改直接引用，可延迟）。

**初始化**：执行 `<clinit>()`，static 赋值与 static 块按源码顺序。

---

### 7. 哪些情况会初始化类？

new、getstatic、putstatic、invokestatic；反射；初始化子类时先初始化父类；main 所在主类；MethodHandle 相关。

**不会**：子类引用父类 static 字段；`new X[10]`；编译期常量 `static final` 直接引用。

---

### 8. 双亲委派模型？好处？如何打破？

类加载请求先委派父加载器，父无法加载子才加载。

好处：防重复加载；保护核心类（自定义 java.lang.String 无效）；沙箱安全。

打破：SPI（JDBC）、Tomcat WebAppClassLoader、OSGi、Spring Boot LaunchedURLClassLoader、线程上下文类加载器 TCCL。

---

### 9. 两个 ClassLoader 加载同一 class 文件，是同一个 Class 吗？

不是。**Class 唯一性 = 类加载器 + 全限定名**。Tomcat 不同 Web 应用同名类互不干扰。

---

## 三、对象与内存

### 10. 对象创建过程？

类加载检查 → 堆上分配（指针碰撞/空闲列表，TLAB）→ 零值初始化 → 设置对象头（Mark Word、Klass 指针）→ `<init>` 构造。

---

### 11. 对象内存布局？

**对象头**：Mark Word（hash、GC 年龄、锁状态）、类型指针（压缩指针 Klass 4 字节）、数组长度（数组才有）。

**实例数据**：字段，对齐策略影响顺序。

**对齐填充**：8 字节倍数。空 Object 约 16 字节（64 位压缩指针）。

---

### 12. 对象一定在堆上吗？

规范上在堆。JIT **逃逸分析**后可能标量替换、栈上分配，对程序员不可见，不能依赖。

---

### 13. 什么是逃逸分析？

分析对象是否逃出方法（被其他线程或外部引用）。未逃逸可 **锁消除**、**标量替换**（不 new，拆成局部变量），减轻 GC。

---

## 四、垃圾回收

### 14. 如何判断对象可回收？

**可达性分析**：从 GC Roots 不可达则可回收。不用引用计数（循环引用问题）。

GC Roots：栈中引用、static、常量、JNI、锁持有对象、JVM 内部引用等。

---

### 15. 四种引用类型？

**强引用**：不回收。`Object o=new Object()`。

**软引用 SoftReference**：内存不足前回收，适合缓存。

**弱引用 WeakReference**：下次 GC 即回收，ThreadLocalMap key、WeakHashMap。

**虚引用 PhantomReference**：无法 get 对象，跟踪回收，Cleaner、堆外内存。

---

### 16. 常见 GC 算法？

**标记-清除**：碎片。

**标记-复制**：新生代 Eden/Survivor，无碎片，浪费空间。

**标记-整理**：老年代，移动存活对象。

---

### 17. Minor GC、Major GC、Full GC？

**Minor/Young GC**：年轻代，频繁，较快。

**Major**：老年代。

**Full GC**：整堆 + 常含元空间等，STW 长，要减少。触发：老年代满、Metaspace 不足、System.gc、担保失败、CMS 失败等。

---

### 18. 分代假说？

绝大多数对象朝生夕灭 → 新生代复制；熬过多次 GC 的对象难死 → 老年代标记整理/清除。

---

### 19. 说说 G1 收集器？

整堆划 **Region**（1~32MB），Eden/Survivor/Old/Humongous。可预测停顿 `-XX:MaxGCPauseMillis`。Young GC 复制；并发标记后 **Mixed GC** 回收部分 Old。Remembered Set + SATB 写屏障。JDK9+ 默认。

---

### 20. CMS 有什么问题？为什么被移除？

并发标记清除，低延迟但：碎片、CPU 敏感、浮动垃圾、Concurrent Mode Failure 退 Full GC。JDK14 移除。

---

### 21. ZGC 特点？

超低延迟，着色指针，并发整理，适合 TB 级堆。JDK15+ 生产可用，JDK21 可分代 ZGC。

---

### 22. 对象晋升老年代条件？

年龄达 MaxTenuringThreshold（默认15）；大对象直接进 Old/Humongous；动态年龄判定 Survivor 同年龄超一半则批量晋升；Survivor 放不下担保失败。

---

## 五、JMM 与锁（JVM 视角）

### 23. JMM 和 JVM 内存结构区别？

JMM 是**并发语义**（主内存、工作内存、happens-before、volatile）。JVM 内存结构是**物理/逻辑划分**（堆栈元空间）。多线程可见性问题用 JMM 解释。

---

### 24. volatile 原理？

写刷主内存，读从主内存，内存屏障禁止特定重排。保证可见性和有序性，**不保证** i++ 原子性。

---

### 25. synchronized 锁升级（概念）？

无锁 → 偏向锁（JDK15+ 默认关）→ 轻量级锁 CAS → 重量级锁 Monitor。升级不可逆。Mark Word 存锁状态。

---

## 六、调优与排查

### 26. 常见 OOM 类型？

heap space、Metaspace、Direct buffer memory、unable to create native thread、GC overhead limit exceeded、StackOverflowError。

---

### 27. -Xms 和 -Xmx 为什么要设成一样？

避免运行期堆扩缩带来的性能抖动和额外 Full GC。生产常见 Xms=Xmx。

---

### 28. 如何排查内存泄漏？

现象：Full GC 后老年代仍涨、最终 OOM。步骤：`-XX:+HeapDumpOnOutOfMemoryError` 或 jmap dump → MAT Histogram/Dominator → Path to GC Roots 查谁持有引用。常见：静态集合、ThreadLocal 未 remove、监听器未注销、连接未关。

---

### 29. jstack、jmap、jstat 用途？

**jstack**：线程栈，死锁、阻塞。

**jmap -heap**：堆概况；**-dump**：堆 dump。

**jstat -gcutil**：GC 与各区占用实时统计。

---

### 30. CPU 100% 怎么查 Java 进程？

top -Hp pid 找高 CPU 线程 → tid 转 16 进制 → jstack 搜 nid=0x... → 看栈顶业务代码或 GC 线程。

---

### 31. 什么是 Safepoint？

线程可安全暂停执行 GC/ dump 的点（方法调用、循环回边等）。长时间 counted 循环可能导致 GC 停顿延迟。

---

### 32. 强引用、软引用在项目里怎么用？

强引用默认。软引用做**有内存压力的缓存**（图片缓存），OOM 前回收。弱引用做**自动清理的关联**（WeakHashMap 缓存 key）。

---

### 33. finalize 还能用吗？

不推荐，JDK9 deprecated。回收前执行一次，可能复活对象，顺序不确定。用 try-with-resources、Cleaner。

---

### 34. 什么是跨代引用？如何解决？

老年代对象引用年轻代对象。Minor GC 不能扫全堆 Old，用 **卡表 Card Table** + **写屏障** 记录 dirty card，只扫相关 Old 区域。

---

### 35. Java 是解释执行还是编译执行？

两者兼有：字节码 + 解释器 + JIT（C1/C2）编译热点为本地代码，混合模式为默认。

---

### 36. 什么是 JIT 内联？

热点方法把小 callee **展开进 caller**，减少调用开销，配合逃逸分析等优化。

---

### 37. 容器里 JVM 内存怎么设？

JDK8u191+/10+ 识别 cgroup。`-Xmx` 约为容器 memory limit 的 **70%~75%**，留 Metaspace、Direct、栈、系统余量，避免 OOMKilled。

---

### 38. System.gc() 会怎样？

建议 Full GC；生产常 `-XX:+DisableExplicitGC`，但 DirectByteBuffer 清理等可能依赖，需评估。

---

### 39. 字符串常量池在哪？intern 注意什么？

JDK7+ 在**堆** String Table。intern 重复短字符串省内存；大字符串 intern 占堆；JDK6 在永久代已过时。

---

### 40. 说说类卸载条件？

类所有实例已回收；加载该类的 ClassLoader 已回收；Class 对象无引用。自定义 ClassLoader 短生命周期类才可能；AppClassLoader 加载的核心类几乎不卸载。
