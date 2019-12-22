---
title: Java CodeBook
date: 2019-12-22 15:14:02
tags:
---

## 推荐使用Builder模式
可用来实现不可变类 可方便的使用`Lombok``@Builder`来使用

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

## 推荐使用try-with-resources来安全的close资源
要使用try-with-resources，资源必须实现AutoCloseable接口，给出一些例子

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
try-with-resources一共有3个步骤
1. 小括号内的代码块 一般为打开资源 
2. 大号内的代码块 一般为使用资源
3. 隐藏的close调用 实现了AutoCloseable的资源会被close
这3个阶段都可能抛出异常  
1. 小括号内的代码块出现异常 => 直接抛出 不再执行下2个阶段 
2. 大号内的代码块出现异常 => 继续执行close阶段 保证资源正常close后抛出异常 
3. 隐藏的close调用出现异常 => 如果2阶段没有异常则抛出异常 否则将自己的异常信息加入到2阶段异常的堆栈中 
给出测试代码 可以分别注释3个阶段的异常抛出代码进行测试
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
给出2、3阶段都出现异常时的堆栈信息，可以明显看到close阶段的异常被`Suppressed`但信息还在，方便排错，可以编程通过`e.getSuppressed()`方法访问到这些信息
```bash
java.lang.Exception: 使用资源出现异常
	at Test.testTryResource(Test.java:69)
	at Test.main(Test.java:53)
	Suppressed: java.lang.Exception: close资源出现异常
		at Test$AutoCloseResource.close(Test.java:63)
		at Test.testTryResource(Test.java:70)
		... 1 more
```








