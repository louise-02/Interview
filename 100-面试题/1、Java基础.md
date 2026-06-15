# Java 基础面试题

本文件可**单独阅读**，每题含完整参考答案。集合与 IO 见 [2、Java集合与IO.md](./2、Java集合与IO.md)；并发见 [3、Java并发.md](./3、Java并发.md)。

深入学习 → [01-Java后端技术栈/Java](../01-Java后端技术栈/Java/0、readme.md)

---

## 一、面向对象

### 1. 面向对象三大特征是什么？

**封装**：把数据和对数据的操作包在类里，用访问修饰符控制可见性（private 字段 + public getter/setter），隐藏实现细节。

**继承**：子类复用父类属性和方法，形成 is-a 关系；Java 单继承，多接口实现。

**多态**：父类引用指向子类对象。编译期看引用类型（能调哪些方法），运行期看实际对象类型（方法重写走子类实现）。**成员变量没有多态**，看引用类型；**方法有多态**。

---

### 2. == 和 equals 的区别？

`==`：基本类型比较**值**；引用类型比较**内存地址**（是否同一对象）。

`equals`：Object 中默认与 `==` 相同。String、Integer 等重写后比较**内容**。重写 equals 必须同时重写 hashCode（HashMap 等依赖：相等对象 hash 必须相同）。

---

### 3. hashCode 和 equals 的契约？

若 `a.equals(b)` 为 true，则 `a.hashCode() == b.hashCode()` 必须成立。反之 hash 相同不一定 equals。不重写 hashCode 会导致 HashSet/HashMap 行为错误（相等对象进不同桶）。

---

### 4. 重载和重写的区别？

| | 重载 Overload | 重写 Override |
| - | ------------- | ------------- |
| 位置 | 同类 | 子类对父类 |
| 方法名 | 相同 | 相同 |
| 参数列表 | 必须不同 | 必须相同 |
| 返回值 | 可不同 | 兼容（子类≤父类） |
| 权限 | 无关 | 不能更严格 |
| 绑定 | 编译期 | 运行期 |

---

### 5. 抽象类和接口的区别？（JDK8+）

| | 抽象类 | 接口 |
| - | ------ | ---- |
| 继承 | 单继承 | 多实现 |
| 构造器 | 有 | 无 |
| 成员 | 任意修饰符字段 | public static final 常量 |
| 方法 | 抽象+普通 | 抽象+default+static |
| 设计意图 | is-a 模板 | can-do 能力 |

---

### 6. 深拷贝和浅拷贝？

**浅拷贝**：复制对象本身，引用字段仍指向原对象内部成员。

**深拷贝**：连同引用指向的对象递归复制。实现：手动复制、序列化、Cloneable 重写 clone 递归、拷贝工具。

---

## 二、String 与基本类型

### 7. Java 四类八种基本类型？

| 整型 | byte(1) short(2) int(4) long(8) |
| 浮点 | float(4) double(8) |
| 字符 | char(2) Unicode |
| 布尔 | boolean（JVM 用 int/fbyte 实现，规范未规定字节数） |

---

### 8. Integer 缓存机制？

`Integer.valueOf(int)` 对 **-128～127** 缓存。`Integer a=127; Integer b=127;` 可能 `a==b` 为 true；128 则为 false。自动装箱用 valueOf。

---

### 9. String 为什么不可变？有什么好处？

JDK9+ 内部 `byte[]` + coder。`final` 类，数组引用不可变（无 setter）。好处：线程安全、常量池复用、作为 HashMap key 安全、配合 StringPool 省内存。

---

### 10. String、StringBuilder、StringBuffer？

| | String | StringBuilder | StringBuffer |
| - | ------ | ------------- | ------------ |
| 可变 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全 | 不安全 | synchronized 安全 |
| 场景 | 常量、共享 | 单线程拼接 | 多线程拼接（少用） |

---

### 11. 字符串常量池和 intern()？

字面量 `"abc"` 进 **String Table（堆）**。`new String("abc")` 在堆上新对象。`intern()` 把引用放入池中并返回池中引用。JDK7+ 池在堆，不在永久代。滥用 intern 大字符串会占堆。

---

## 三、异常

### 20. Error 和 Exception？

**Error**：JVM 级严重问题，如 OutOfMemoryError、StackOverflowError，一般不捕获。

**Exception**：可处理。Checked 必须 try/throws（IOException）；Unchecked 继承 RuntimeException（NPE、IllegalArgument），不强制处理。

---

### 21. finally 一定会执行吗？

一般情况会；**除非** System.exit()、JVM 崩溃、死循环、线程 kill。若 try 有 return，finally 仍先执行再 return（注意 finally 里 return 会覆盖 try 返回值）。

---

### 22. try-with-resources 原理？

Java7+，资源实现 AutoCloseable，编译器生成 finally 自动 close，异常 suppressed 链。比手写 finally 关流更安全。

---

## 四、反射与泛型

### 25. 反射用途与缺点？

用途：框架（Spring 创建 Bean）、动态代理、注解处理、JDBC Class.forName。缺点：破坏封装、性能较低、绕过安全检查。应缓存 Method/Field。

---

### 26. 泛型擦除是什么？

编译后泛型信息擦除为原始类型（List<String> → List），编译器插入强转。运行时无法 `new T()` 或 `instanceof List<String>`。通配符 `? extends T` 只读，`? super T` 只写（PECS）。

---

## 五、常见笔试/坑题

### 27. 接口和抽象类如何选择？

多个 unrelated 类共享行为 → 接口。强 is-a、需要共享字段/非 public 方法 → 抽象类。JDK8 后接口可有 default，但仍不能替代有状态的抽象基类。

---

### 28. 为什么重写 equals 必须重写 hashCode？举例。

两个逻辑相等对象若 hash 不同，放进 HashSet 会出现两个「相等」元素。违反集合契约。

---

### 29. ArrayList 扩容机制？

默认容量 10（首次 add 才创建）。不够时 **1.5 倍** grow，Arrays.copyOf 复制到新数组。

---

### 30. Comparable 和 Comparator？

Comparable：类自身 `compareTo`，自然排序（String、Integer）。

Comparator：外部比较器，可多种排序策略，Collections.sort(list, comparator)。

---

### 31. 什么是 Java 的 valueOf 和 parseInt 区别？

`Integer.valueOf("123")` 返回 Integer 对象，可能走缓存。`Integer.parseInt("123")` 返回基本类型 int。

---

### 32. 静态代码块、构造块、构造器顺序？

静态块（类加载一次）→ 实例块（每次 new）→ 构造器。父类静态 → 子类静态 → 父类块/构造 → 子类块/构造。

---

### 33. 内部类有几种？

成员内部类、静态内部类、局部内部类、匿名内部类。非静态内部类持有外部类 this；静态内部类不持有。

---

### 34. 为什么局部内部类引用局部变量必须是 final 或 effectively final？

内部类对象生命周期可能长于方法栈帧，变量需复制到内部类对象，为保证一致要求 effectively final。

---

### 35. Java 是值传递还是引用传递？

**值传递**。引用类型传的是**引用的副本**（地址值的拷贝），通过副本仍可改对象内容，但不能让调用方变量指向新对象（swap 失败原因）。

---

### 36. 什么是 SPI？和双亲委派？

Service Provider Interface：`META-INF/services` 声明实现。如 JDBC Driver。Bootstrap 加载 Driver 接口，实现类在 classpath，用 **Thread.contextClassLoader** 加载，打破双亲委派。

---

### 37. JDK 与 JRE、JVM 关系？

JVM 运行字节码。JRE = JVM + 核心类库。JDK = JRE + 开发工具（javac 等）。

---

### 38. char 能存中文吗？

可以，char 2 字节 UTF-16 单元；生僻字可能需 surrogate 对，应用层更推荐 String。

---

### 39. 什么是自动装箱拆箱？有什么坑？

基本类型 ↔ 包装类自动转换。null 拆箱 NPE；循环里 Integer 累加产生大量对象；== 比较 128 以外 Integer 用 equals。

---

### 40. 如何理解 Java 跨平台？

源码编译为 **.class 字节码**，由不同平台的 **JVM 解释/JIT** 执行。一次编译，到处运行（前提是装有对应 JVM）。
