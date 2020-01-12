---
title: Java CodeBook
date: 2019-12-22 15:14:02
tags:
  - Java
---

## 推荐使用 try-with-resources 来安全的 close 资源

要使用 try-with-resources，资源必须实现 AutoCloseable 接口，给出一些例子

```java
static String readFirstLine(String path) throws IOException{
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
// 同时使用多个资源
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[512];
        int n;
        while ((n = in.read(buf)) >= 0) {
            out.write(buf, 0, n);
        }
    }
}
```

### 关于异常处理

try-with-resources 一共有 3 个步骤

1. 小括号内的代码块 一般为打开资源
2. 大号内的代码块 一般为使用资源
3. 隐藏的 close 调用 实现了 AutoCloseable 的资源会被 close

这 3 个阶段都可能抛出异常

1. 小括号内的代码块出现异常 => 直接抛出 不再执行下 2 个阶段
2. 大括号内的代码块出现异常 => 继续执行 close 阶段 保证资源正常 close 后抛出异常
3. 隐藏的 close 调用出现异常 => 如果 2 阶段没有异常则抛出异常 否则将自己的异常信息加入到 2 阶段异常的堆栈中

给出测试代码 可以分别注释 3 个阶段的异常抛出代码进行测试

```java
static class AutoCloseResource implements AutoCloseable {
    AutoCloseResource() throws Exception {
        throw new Exception("open资源出现异常");
    }

    @Override
    public void close() throws Exception {
        throw new Exception("close资源出现异常");
    }
}
static void testTryResource() {
    try (AutoCloseResource res = new AutoCloseResource()) {
        throw new Exception("使用资源出现异常");
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

给出 2、3 阶段都出现异常时的堆栈信息，可以明显看到 close 阶段的异常被`Suppressed`但信息还在，方便排错，可以编程通过`e.getSuppressed()`方法访问到这些信息

```bash
java.lang.Exception: 使用资源出现异常
	at Test.testTryResource(Test.java:69)
	at Test.main(Test.java:53)
	Suppressed: java.lang.Exception: close资源出现异常
		at Test$AutoCloseResource.close(Test.java:63)
		at Test.testTryResource(Test.java:70)
		... 1 more
```

## 并发-安全的终止线程

### 正确示范

```java
import java.util.concurrent.TimeUnit;

public class Test implements Runnable {
    private volatile boolean stop;

    public void run() {
        while (!stop && !Thread.currentThread().isInterrupted()) {
            // do something
        }
    }

    public void stop() {
        stop = true;
    }

    public static void main(String[] args) throws InterruptedException {
        Test runner = new Test();
        Thread thread = new Thread(runner, "runnerThread");
        thread.start();
        TimeUnit.SECONDS.sleep(1L);
        runner.stop();
        // runner.interrupt(); // 这样线程反应要更快 毕竟interrupt0()是native
    }
}
```

### 错误示范 1

来自《Effective Java》，如书中所说我对下面这段程序的预测也是运行超过 1 秒钟，但本机测试后，这个程序永远没有停止。。。

```java
import java.util.concurrent.TimeUnit;

public class Test implements Runnable {
    private boolean stop;

    public void run() {
        while (!stop) {
        }
    }

    public void stop() {
        stop = true;
    }

    public static void main(String[] args) throws InterruptedException {
        Test runner = new Test();
        Thread thread = new Thread(runner, "runnerThread");
        thread.start();
        TimeUnit.SECONDS.sleep(1L);
        runner.stop();
    }
}
```

书中给了简单的解释是因为 JVM 会将下面的代码：

```java
while (!stop) {
}
```

优化成这样：

```java
if (!stop) {
    while(true) {
    }
}
```

那么是谁出于什么目的依照什么标准在什么阶段完成的上述优化呢？这个问题完全值得深挖一篇新博客了，留坑。这里先做一个简单的回答：

> `JIT编译器`(Just In Time Compiler)出于提高`热点代码`的执行效率的目的按照`JMM`(Java 内存模型)的指导(在这里其实是没做要求)在`运行时`(Runtime)完成的`循环表达式外提`(Loop Expression Hoisting)优化。也就是书中的“提升”。

可以通过参数 **-Xint** 来强制 JVM 运行于`解释模式`(Interpreted Mode)来让 JIT 不工作，此时上面的代码会如想象般运行超过 1s 但最终停止。

## 设计模式 Demo

### 单例模式

1. 提前初始化

```java
class Singleton {
    private static Singleton instance = new Singleton();
    // 构造方法私有
    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```

2. 内部类实现懒汉

```java
class Singleton {
    // 构造方法私有
    private Singleton() {}

    public static Singleton getInstance() {
        return Holder.instance;
    }

    private static class Holder {
        static Singleton instance = new Singleton();
    }
}
```

3. Double-Check

```java
class Singleton {
    // volatile保证未完成初始化的对象不会漏出
    private static volatile Singleton instance;
    // 构造方法私有
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

> 此时还是有几个小问题：

- 可能通过反射调用私有的构造方法
- 序列化问题 readObject()永远返回一个新的实例

4. 最佳实践 枚举实现单例

```java
class Singleton {
    // 构造方法私有
    private Singleton() {}

    public static Singleton getInstance() {
        return HolderEnum.INSTANCE.getInstance();
    }

    static enum HolderEnum {
        INSTANCE;
        private Singleton instance;

        private HolderEnum() {
            this.instance = new Singleton();
        }

        public Singleton getInstance() {
            return this.instance;
        }
    }
}
```

### 工厂模式

接口及实现类

```java
public interface Shape {
   void draw();
}

public class Rectangle implements Shape {
   public void draw() {
      System.out.println("Inside Rectangle::draw() method.");
   }
}

public class Square implements Shape {
   public void draw() {
      System.out.println("Inside Square::draw() method.");
   }
}

public class Circle implements Shape {
   public void draw() {
      System.out.println("Inside Circle::draw() method.");
   }
}
```

工厂

```java
public class ShapeFactory {

   //使用 getShape 方法获取形状类型的对象
   public Shape getShape(String shapeType){
      if(shapeType == null){
         return null;
      }
      if(shapeType.equalsIgnoreCase("CIRCLE")){
         return new Circle();
      } else if(shapeType.equalsIgnoreCase("RECTANGLE")){
         return new Rectangle();
      } else if(shapeType.equalsIgnoreCase("SQUARE")){
         return new Square();
      }
      return null;
   }
}
```

进一步优化:通过配置来实现加载不同的实现类

### 抽象工厂模式

使用工厂来创建工厂

### 建造者模式

使用一个 Builder 来一步步构造最终的对象 可用来实现不可变类 可方便的使用`Lombok` `@Builder`来使用

```java
public class Person {
    private final String name;
    private final int gender;
    private final int age;

    private Person(Builder builder) {
        this.name = builder.name;
        this.gender = builder.gender;
        this.age = builder.age;
    }

    public static class Builder {
        // 必须参数
        private final String name;
        // 可选参数 初值
        private int gender = -1;
        private int age = -1;

        public Builder(String name) {
            this.name = name;
        }

        public Builder gender(int gender) {
            this.gender = gender;
            return this;
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public Person build() {
            return new Person(this);
        }
    }
}
```

使用起来和具名参数一样：

```java
Person person = new Person.Builder("xiaoming").gender(1).age(18).build();
```

### 原型模式

用于创建重复的对象，同时又能保证性能。 例如 Java 中的 Object.clone()

说明：Java 中的 clone 需要保证 1) a.clone() != a 2) a.clone().getClass() == a.getClass() 不强求：a.clone().equals(a)
对于除了基本类型和不可变类以外的成员变量，注意深拷贝
下面的栗子代码虽然没有给出 equals 方法的实现 但显而易见没有保证 person.clone().equals(person)

```java
public class Person implements Cloneable {
    private String name;
    private int age;
    private int gender;

    @Override
    protected Person clone() throws CloneNotSupportedException {
        Object o = super.clone();
        Person clone = (Person) o;
        clone.age = 1;
        return clone;
    }

```

### 适配器模式

接口转换 以双线插头适配三线插座为例

```java
public class TwoWirePlug {
    void fireWire() {
        System.out.println("火线");
    }
    void zeroWire() {
        System.out.println("零线");
    }
}

interface ThreeWirePlug {
    void fireWire();
    void zeroWire();
    void landWire();
}

public class Adapter implements ThreeWirePlug {
    TwoWirePlug twoWirePlug;

    void fireWire() {
        this.twoWirePlug.fireWire();
    }
    void zeroWire() {
        this.twoWirePlug.zeroWire();
    }
    void landWire() {
        System.out.println("地线不接");
    }
}

```

### 桥接模式

桥接 2 个独立变化的维度 偏向组装的概念

### 责任链模式

钉钉机器人的实现

### 观察者模式

当对象间存在一对多关系时，则使用观察者模式（Observer Pattern）。比如，当一个对象被修改时，则会自动通知它的依赖对象。观察者模式属于行为型模式。
关键在于在 Subject 中存放 observer 列表

### 策略模式

大量行为不同的类 代表不同的策略
事件处理方案的实现

### 代理模式

Spring aop
