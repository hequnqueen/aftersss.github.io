---
layout: post
title:  "java里final一定是不可变的吗"
date:   2017-08-08 15:12:12 +0800
categories: Java
author: 小米粒
keywords: java, final
description: 本文探讨java中final变量是否一定不可变，以及final常量的一些编译器优化细节
---

> 本文探讨java中final变量是否一定不可变，以及final常量的一些编译器优化细节

出于防御式编程的需要，我们在写一些SDK代码或者不希望被别人破坏的代码的时候会用到final来表示不可变的引用，比如如下代码：

### 代码1：final字段
```java:n
public class User implements Serializable {
    private final String finalField;
    public User(){
        this.finalField = "original";
    }
    public String getFinalField() {
        return finalField;
    }
}
```
然后构造User user = new User()来使用，你可能认为这样就高枕无忧了，其他人调用user.getFinalField()拿到的一定是字符串original。

然而别忘了反射，有些人可能会想，final的变量一旦初始化后应该就不能改变了，按道理反射也不能打破这个条框啊，实践出真知，让我们写代码来验证下：
### 代码2：使用反射改变final字段
```java:n
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
    User user = new User();
    Field finalField = User.class.getDeclaredField("finalField");
    finalField.setAccessible(true);
    finalField.set(user, "changed");
    System.out.println(user.getFinalField());
}
```
你应该能看到了，输出的结果是changed，反射打破了final的语义。

让我们再来深入下，如果finalField不是在构造函数里初始化，而是声明字段时就初始化会怎么样呢？
### 代码3：使用反射改变final常量
```java:n
public class User implements Serializable {
    private final String finalField = "original";

    public String getFinalField() {
        return finalField;
    }

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        User user = new User();
        Field finalField = User.class.getDeclaredField("finalField");
        finalField.setAccessible(true);
        finalField.set(user, "changed");
        System.out.println(user.getFinalField());
    }
}
```
运行代码3，输出为original，你可能会很奇怪，命名变化不大，为什么行为差别这么大？

这是因为编译期间final类型的数据被自动优化了，即：所有用到该变量的地方都被替换成了常量。所以 get方法在编译后自动优化成了return "original"； 而不是 return this.finalField；

如果你不希望编译器自动做这样的优化，按如下代码4改动一下就可以了(这里的代码只是为了让你深入一些细节，实际上你没有理由这样做)：
### 代码4：绕过编译器自动优化
```java:n
private final String finalField = (null != null ? "notUsed" : "original");
```
实际上代码3确实执行成功了而且有效果，如果你调用finalField.get()会发现拿到的是改变后的值，很神奇吧。

那么java为什么允许通过反射改变final的引用？我也不知道，虽然存在弊端，但是这样做也带来了一些好处，比如一个SDK内部提供的类字段是final的，一般而言，SDK定义为final肯定是有道理的，我们应该遵守他的玩法，但是如果出于某些原因（比如SDK内部有bug，而你又来不及等待官方修复）我们不得不修改这个字段的引用时，使用反射来改变不失为一个可选的办法(当然如果SDK使用代码3的方式来声明常量，那么此路不通)。

