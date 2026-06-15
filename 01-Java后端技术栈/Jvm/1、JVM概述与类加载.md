# 1、JVM 概述与类加载

## 1、JDK、JRE、JVM

| 概念 | 说明 |
| ---- | ---- |
| **JVM** | Java 虚拟机，负责加载 Class、执行字节码，实现跨平台 |
| **JRE** | Java 运行时环境 = JVM + 核心类库 |
| **JDK** | Java 开发工具包 = JRE + 开发工具（javac、javadoc 等） |

> JDK = JRE + 开发工具；JRE = JVM + 标准类库

## 2、Java 代码执行流程

```
.java 源文件
  → javac 编译
  → .class 字节码
  → ClassLoader 加载
  → 运行时数据区（方法区/堆/栈等）
  → Execution Engine 解释 / JIT 编译执行
```

**编译型 + 解释型**：Java 先编译成字节码，再由 JVM **解释执行**或 **JIT 编译**为本地代码后执行。HotSpot 默认采用**混合模式**（解释器 + C1/C2 编译器）。

## 3、JVM 整体架构（逻辑）

```
┌─────────────────────────────────────────────┐
│              Class Loader Subsystem          │
│         加载 / 链接 / 初始化                  │
├─────────────────────────────────────────────┤
│              Runtime Data Areas              │
│  堆 │ 方法区/元空间 │ 虚拟机栈 │ 本地方法栈 │ PC │
├─────────────────────────────────────────────┤
│              Execution Engine                │
│         解释器 │ JIT │ GC │ 本地方法接口        │
└─────────────────────────────────────────────┘
```

HotSpot 中 **Java 字节码基于栈式指令集**（与 x86 等基于寄存器的本地代码不同）。

## 4、类加载子系统

ClassLoader 只负责 **Class 文件的加载**；类能否运行由执行引擎决定。加载后的类元信息存放在**方法区（JDK 8 起为元空间 Metaspace）**，运行时常量池是 Class 文件常量池的运行时映射。

### 4.1 加载（Loading）

1. 通过类的**全限定名**获取二进制字节流。
2. 将字节流代表的静态存储结构转化为方法区内的**运行时数据结构**。
3. 在内存中生成代表该类的 `java.lang.Class` 对象，作为方法区中该类数据的访问入口。

来源可以是：本地文件、Jar、网络、运行时动态生成（动态代理）、数据库等。

### 4.2 链接（Linking）

**验证（Verify）**

确保 Class 文件字节流符合 JVM 规范，防止恶意代码。包括：文件格式、元数据、字节码、符号引用验证。

**准备（Prepare）**

为**类变量（static）**分配内存并设**零值**（0、false、null 等）。

- `static final` 常量若在编译期可确定，在**准备阶段**就赋最终值。
- **实例变量**在此阶段不分配，随对象在堆中分配。

**解析（Resolve）**

将常量池中的**符号引用**替换为**直接引用**（内存地址或偏移量）。部分解析可在初始化前或初始化过程中完成（延迟解析）。

### 4.3 初始化（Initialization）

执行类构造器 `<clinit>()`：按源码顺序执行 static 变量赋值与 static 代码块。

**触发初始化的时机（主动引用）**

- new、getstatic、putstatic、invokestatic 等字节码指令。
- 反射调用类。
- 初始化子类时，若父类未初始化则先初始化父类。
- 含 `main()` 的主类。
- JDK 7 起：动态语言支持相关的 MethodHandle 解析结果对应的类。

**不会触发初始化**

- 通过子类引用父类 static 字段（只初始化父类）。
- 通过数组定义引用类（`new X[10]` 不触发 X 的初始化）。
- 引用常量（编译期常量折叠）。

```java
public class InitDemo {
    static {
        System.out.println("InitDemo 初始化");
    }

    public static void main(String[] args) {
        System.out.println(InitDemo.VALUE); // 只输出 100，不打印 static 块
    }

    public static final int VALUE = 100;
}
```

> 若父类存在，JVM 保证子类 `<clinit>()` 执行前，父类 `<clinit>()` 已完成。

## 5、类加载器

| 类加载器 | 实现 | 加载范围 |
| -------- | ---- | -------- |
| **Bootstrap ClassLoader** | C/C++，JVM 内置 | `JAVA_HOME/lib` 核心库（rt.jar 等），java/javax/sun 等包 |
| **Extension ClassLoader** | Java，`sun.misc.Launcher$ExtClassLoader` | `JAVA_HOME/lib/ext` |
| **Application ClassLoader** | Java，`sun.misc.Launcher$AppClassLoader` | `classpath` / `java.class.path` |
| **Custom ClassLoader** | 用户继承 `ClassLoader` | 自定义路径，隔离、热部署、加密等 |

Bootstrap 不继承 `ClassLoader`，在 Java 代码中表现为 `null`（`getParent()` 对 AppClassLoader 返回 null 通常指 Bootstrap）。

```java
ClassLoader app = ClassLoader.getSystemClassLoader();
System.out.println(app);                    // AppClassLoader
System.out.println(app.getParent());        // ExtClassLoader
System.out.println(app.getParent().getParent()); // null → Bootstrap
```

## 6、双亲委派机制

类加载请求：**先委派给父加载器**，父加载器无法完成时，子加载器才尝试自己加载。

```
AppClassLoader 收到请求
  → 委派 ExtClassLoader
    → 委派 Bootstrap
      → Bootstrap 能加载则返回
      → 否则 Ext 尝试
    → 否则 App 尝试
  → 否则 Custom 尝试
```

**优点**

1. **避免重复加载**：同一类在 JVM 中只加载一次。
2. **保护核心 API**：自定义 `java.lang.String` 无法替换 Bootstrap 已加载的核心类（沙箱安全）。
3. **命名空间隔离**：不同 ClassLoader 加载的同名类视为不同类。

**破坏双亲委派的典型场景**

| 场景 | 原因 |
| ---- | ---- |
| **SPI**（JDBC 等） | 核心类需加载实现类，使用线程上下文类加载器（TCCL） |
| **Tomcat** | 不同 Web 应用类隔离，同一 Web 内优先 WebAppClassLoader |
| **OSGi** | 模块化，类加载网状结构 |
| **热部署 / 插件** | 自定义 ClassLoader 优先加载新版本 |

## 7、JPMS 与类加载（JDK 9+）

- **Bootstrap**：`java.base` 等。
- **Platform ClassLoader**：取代 ExtClassLoader，加载平台模块。
- **App ClassLoader**：模块路径 + classpath。
- **Layer**：多层模块隔离，详见 [11、JVM 进阶专题](./11、JVM进阶专题.md)。

## 8、类的卸载

需同时满足：无实例、ClassLoader 可回收、无 `Class` 引用。自定义 ClassLoader + 短生命周期类才可能；**核心类永不卸载**。

## 9、常见 JVM 实现

HotSpot（默认）、OpenJ9、GraalVM 等——JVM 是**规范**，HotSpot 是**实现**。详见第 11 章。

## 10、常见面试题

**1. 类加载与对象创建的区别？**

类加载把 `.class` 读入方法区并生成 `Class` 对象；`new` 在堆上分配实例，执行实例初始化 `<init>()`。

**2. 两个 ClassLoader 加载同一个 .class 文件，是同一个 Class 吗？**

不是。Class 的唯一性 = **类加载器 + 全限定名**。

**3. 如何自定义 ClassLoader？**

继承 `ClassLoader`，重写 `findClass()`（推荐）或 `loadClass()`，在 `findClass` 中读取字节码并 `defineClass()`。

```java
public class MyClassLoader extends ClassLoader {
    private final String basePath;

    public MyClassLoader(String basePath, ClassLoader parent) {
        super(parent);
        this.basePath = basePath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] data = loadByteCode(name);
        return defineClass(name, data, 0, data.length);
    }

    private byte[] loadByteCode(String name) {
        // 从 basePath 读取 name.replace('.', '/') + ".class"
        throw new UnsupportedOperationException("按路径读取字节码");
    }
}
```
