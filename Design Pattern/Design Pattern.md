# 0、前言

# 1、单例模式

## 1.1、单例模式介绍

所谓类的单例设计模式，就是采取一定的方法保证在整个的软件系统中，对某个类**只能存在一个对象实例**，并且该类只提供一个取得其对象实例的方法(静态方法)。

## 1.2、注意事项和细节

单例模式保证了系统内存中该类只存在一个对象，节省了系统资源，对于一些需要频繁创建销毁的对象，使用单例模式可以提高系统性能。

当想实例化一个单例类的时候，必须要记住使用相应的获取对象的方法，而不是使用 new。

单例模式使用的场景：需要频繁的进行创建和销毁的对象、创建对象时耗时过多或耗费资源过多（即：重量级对象），但又经常用到的对象、工具类对象、频繁访问数据库或文件的对象（比如数据源、session工厂等）。

## 1.3、代码实现

### 饿汉式(静态常量)

优点：这种写法比较简单，就是在类装载的时候就完成实例化。避免了线程同步问题。

缺点：在类装载的时候就完成实例化，没有达到 Lazy Loading 的效果。如果从始至终从未使用过这个实例，则会造成内存的浪费。

```java
public class Singleton_01 {
    public static final Singleton_01 SINGLETON = new Singleton_01();

    private Singleton_01() {
    }

    public static Singleton_01 getInstance() {
        return SINGLETON;
    }
}
```

### 饿汉式(静态代码块)

```java
public class Singleton_02 {
    public static final Singleton_02 SINGLETON;

    static {
        SINGLETON = new Singleton_02();
    }

    private Singleton_02() {
    }

    public static Singleton_02 getInstance() {
        return SINGLETON;
    }
}
```

### 懒汉式(线程不安全)

起到了 Lazy Loading 的效果，但是只能在单线程下使用。

```java
public class Singleton_03 {
    public static Singleton_03 SINGLETON;

    private Singleton_03() {
    }

    public static Singleton_03 getInstance() {
        if (SINGLETON == null) {
            SINGLETON = new Singleton_03();
        }
        return SINGLETON;
    }
}
```

### 懒汉式(同步方法)

解决了线程不安全问题。

效率太低了，每个线程在想获得类的实例时候，执行 getInstance() 方法都要进行同步。而其实这个方法只执行一次实例化代码就够了，后面的想获得该类实例，直接 return 就行了。方法进行同步效率太低。

在实际开发中，**不推荐使用这种方式**。

```java
public class Singleton_04 {
    public static Singleton_04 SINGLETON;

    private Singleton_04() {
    }

    public synchronized static Singleton_04 getInstance() {
        if (SINGLETON == null) {
            SINGLETON = new Singleton_04();
        }
        return SINGLETON;
    }
}
```

### 懒汉式(同步代码块)

这种同步并不能起到线程同步的作用，如果两个线程同时进入 if 就会导致出现多个实例。

在实际开发中，**不能使用这种方式**。

```java
public class Singleton_05 {
    public static Singleton_05 SINGLETON;

    private Singleton_05() {
    }

    public static Singleton_05 getInstance() {
        if (SINGLETON == null) {
            synchronized (Singleton_05.class) {
                SINGLETON = new Singleton_05();
            }
        }
        return SINGLETON;
    }
}
```

### 双检锁

Double-Check 概念是多线程开发中常使用到的，通过两次 if 判断检查，这样就可以保证线程安全。

优点：线程安全，延迟加载，效率较高。

在实际开发中，**推荐使用这种单例设计模式**。

```java
public class Singleton_06 {
    public static Singleton_06 SINGLETON;

    private Singleton_06() {
    }

    public static Singleton_06 getInstance() {
        if (SINGLETON == null) {
            synchronized (Singleton_06.class) {
                if (SINGLETON == null) {
                    SINGLETON = new Singleton_06();
                }
            }
        }
        return SINGLETON;
    }
}
```

### 静态内部类

这种方式采用了类装载的机制来保证初始化实例时只有一个线程。

静态内部类方式在 Singleton 类被装载时并不会立即实例化，而是在需要实例化时，调用 getInstance 方法，才会装载 SingletonHolder 类，从而完成 Singleton 的实例化。

类的静态属性只会在第一次加载类的时候初始化，所以在这里，JVM 帮助我们保证了线程的安全性，在类进行初始化时，别的线程是无法进入的。

优点：避免了线程不安全，利用静态内部类特点实现延迟加载，效率高。

在实际开发中，**推荐使用这种单例设计模式**。

```java
public class Singleton_07 {

    private Singleton_07() {
    }

    private static final class SingletonHolder {
        static final Singleton_07 SINGLETON = new Singleton_07();
    }

    public static Singleton_07 getInstance() {
        return SingletonHolder.SINGLETON;
    }
}
```

### 枚举

这借助 JDK1.5 中添加的枚举来实现单例模式。不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象。

在实际开发中，**推荐使用这种单例设计模式**。

```java
public enum Singleton_08 {
    INSTANCE;
}
```







# 2、工厂模式

## 2.1、需求引入

一个披萨的项目：要便于披萨种类的扩展，要便于维护。

披萨的种类很多(比如 GreekPizz、CheesePizz 等)。

披萨的制作方法有 prepare，bake，cut，box。

完成披萨店订购功能。

## 2.2、







# 3、原型模式

## 3.1、需求引入

现在有一只羊，姓名：tom, 年龄：1，颜色：白色，请编写程序创建和 tom 羊属性完全相同的10只羊。

## 3.2、原始方法

sheep

```java
public class Sheep {
    String name;
    Integer age;
    String color;

    public Sheep(String name, Integer age, String color) {
        this.name = name;
        this.age = age;
        this.color = color;
    }
}
```

client

```java
public class Test {
    public static void main(String[] args) {
        Sheep sheep = new Sheep("tom", 1, "白色");
        
        Sheep sheep1 = new Sheep(sheep.name, sheep.age, sheep.color);
        Sheep sheep2 = new Sheep(sheep.name, sheep.age, sheep.color);
    }
}
```

**优缺点**

优点是比较好理解，简单易操作。

在创建新的对象时，总是需要重新获取原始对象的属性，如果创建的对象比较复杂时，效率较低。

总是需要重新初始化对象，而不是动态地获得对象运行时的状态, 不够灵活。

## 3.3、改进思路

Java 中 Object 类是所有类的根类，Object 类提供了一个 clone() 方法，该方法可以将一个 Java 对象复制一份，但是需要实现 clone 的 Java 类必须要实现一个接口 Cloneable，该接口表示该类能够复制且具有复制的能力。

## 3.4、原型模式介绍

原型模式（Prototype 模式）是指：用原型实例指定创建对象的种类，并且通过拷贝这些原型，创建新的对象。

原型模式是一种创建型设计模式，允许一个对象再创建另外一个可定制的对象，无需知道如何创建的细节。

工作原理是：通过将一个原型对象传给那个要发动创建的对象，这个要发动创建的对象通过请求原型对象拷贝它们自己来实施创建，即 对象.clone()。

## 3.5、代码实现

sheep

```java
public class Sheep implements Cloneable {
    String name;
    Integer age;
    String color;
    Sheep friend;

    public Sheep(String name, Integer age, String color) {
        this.name = name;
        this.age = age;
        this.color = color;
    }

    @Override
    public Sheep clone() {
        try {
            return (Sheep) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

client

```java
public class Test {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Sheep sheep = new Sheep("tom", 1, "白色");
        sheep.friend = new Sheep("rose", 2, "黑色");

        Sheep clone = sheep.clone();
    }
}
```

## 3.6、浅拷贝和深拷贝

### 3.6.1、浅拷贝

对于基本数据类型的成员变量，浅拷贝会直接进行值传递，也就是将该属性值复制一份给新的对象。

对于引用数据类型的成员变量，比如说成员变量是某个数组、某个类的对象等，那么浅拷贝会进行引用传递，也就是只是将该成员变量的引用值（内存地址）复制一份给新的对象。因为实际上两个对象的该成员变量都指向同一个实例。在这种情况下，在一个对象中修改该成员变量会影响到另一个对象的该成员变量值。

浅拷贝是使用默认的 clone() 方法来实现。

### 3.6.2、深拷贝

复制对象的所有基本数据类型的成员变量值。

为所有引用数据类型的成员变量申请存储空间，并复制每个引用数据类型成员变量所引用的对象，直到该对象可达的所有对象。也就是说，对象进行深拷贝要对整个对象进行拷贝。

深拷贝实现方式1：重写clone方法来实现深拷贝。

深拷贝实现方式2：通过对象序列化实现深拷贝（推荐）。

### 3.6.3、序列化深拷贝

sheep

```java
import java.io.Serializable;

public class Sheep implements Serializable {
    String name;
    Integer age;
    String color;
    Sheep friend;

    public Sheep(String name, Integer age, String color) {
        this.name = name;
        this.age = age;
        this.color = color;
    }
}
```

client

```java
public class Test {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Sheep sheep = new Sheep("tom", 1, "白色");
        sheep.friend = new Sheep("rose", 2, "黑色");

        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
        objectOutputStream.writeObject(sheep);

        ObjectInputStream objectInputStream = new ObjectInputStream(new ByteArrayInputStream(outputStream.toByteArray()));
        Sheep sheep1 = (Sheep) objectInputStream.readObject();
    }
}
```

