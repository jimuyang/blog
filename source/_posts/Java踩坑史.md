---
title: Java踩坑史
date: 2019-11-24 01:58:08
tags:
---

## list.subList相关
以java.util.ArrayList为例：`java.util.ArrayList#subList`方法返回的结果实际类型为：`java.util.ArrayList.SubList`。它是ArrayList的内部类，持有ArrayList的`this`，当对SubList进行`写`操作时，都会检查自己保存的`modCount`和`ArrayList.this.modCount`是否一致，这要求了在执行了list.subList()之后，不可以对list进行`写`操作（会修改modCount），否则subList的`写`操作就会报ConcurrentModificationException。

### SubList检查modCount源码
```java
    private void checkForComodification() {
        if (ArrayList.this.modCount != this.modCount)
            throw new ConcurrentModificationException();
    }
```
     

## short.equals(1)


## 三元运算符的类型转换
特别简单的2行代码
```java
public class Test{
    public static void main(String[] args) {
        Integer i = null;
        Integer j = true ? i : 0;
    }
};
```
然而第二行代码会报一个NPE。。。
```bash
$ java Test
Exception in thread "main" java.lang.NullPointerException
        at Test.main(Test.java:4)
```
用javap可以明显的看到原因：
```bash
javap -c Test
Compiled from "Test.java"
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: aconst_null
       1: astore_1
       2: aload_1
       3: invokevirtual #2                  // Method java/lang/Integer.intValue:()I
       6: invokestatic  #3                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       9: astore_2
      10: return
}
```
明显的看到执行了``Method java/lang/Integer.intValue:()I``，再测试`Double` `Float` `Boolean`都发现了这个问题。
那么可以得到结论:如果三元运算符`:`两侧为基本类型和包装类型，那么java编译执行时会将``包装类型拆箱处理``，这一点有点反直觉也很坑，使用三元运算符需要注意。

