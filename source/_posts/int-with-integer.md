---
title: 一道笔试题的思考:基本类型和包装类型进行数据比较的问题总结
cover: 'http://pic.menduo.xyz/20190421131526.png'
categories: Java
tags: 
    - 基础知识
    - Java
date: 2019-04-21 12:59:49
---

昨天阅文的一道笔试题，让仔细解释一下 `int`和`Integer`各种不同数据比较的问题，我发现自己能说出来的东西挺少的。阿里一面面试官说我了解知识的深度不够的问题，确实让我想了很多，其实我给自己的`特质`定位应该是可以迅速发现问题并解决、可以很快的举一反三，但是很少会因为这个问题去思考到底是底层是因为什么，也就是说`总结反思`的能力太差了，说白了还是修炼不到位，平时太懒了。

所以以后遇到的每个问题都要认真的记录下来。争取秋招拿offer!

<!-- more -->

对于这个问题，首先有个**装箱**和**拆箱**的概念要首先被明确。也就是基本类型和包装类型可以相互转化。

### 什么是装箱和拆箱？

对于每一个基本类型都会有一个对应的包装类型。例如`int`的包装类型就是`Integer`。而拆箱动作就是将包装类型转化为基本类型，装箱动作就是将基本类型变成包装类型。

#### 对应的API,以Integer为例：
- 基本类型 --> 包装类型 Integer i = Integer.valueOf(int);

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)]; 
        //常量池缓存。[-128,127 or high] 默认127
        //其中缓存池high大小可通过参数『java.lang.Integer.IntegerCache.high』进行调整,
        //最大为Integer.MAX_VALUE - 128 - 1 
    return new Integer(i);
}
```

- 包装类型 --> 基本类型 int ii = i.intValue();

``` java
public int intValue() {
    return value;
}
```
**总结：装箱操作就是调用包装类型的valueOf ;拆箱操作就是调用包装类型的xxValue()**

#### Java可以做到自动拆箱和装箱

> 在java中已经实现了自动拆箱和装箱的操作。

> 时机：当任何一个操作中混用了基本类型和装箱类型，装箱类型就会进行拆箱操作。


### 总结几个混合使用的要点

#### 一些简单的测试用例进行解释整个 int和Integer进行比较的总结：

- int 和 Integer进行混合比较时，Integer会进行拆箱操作变成基本类型。
- Integer xx  = 1 这种会调用valueOf自动装箱。
- Integer进行new操作的对象是放到堆上的。
- Integer 调用valueOf返回的对象，分两种情况：① -128 <= x <= 127 返回的常量池缓冲区的对象 ② x > 128 会在堆上new一个新的对象 (只考虑默认的情况)

#### 具体的几个测试用例
```java
    public static void main(String[] args) {
        int a0 = 1;
        int b0 = 128;
        Integer a1 = 1;
        Integer b1 = 128;
        Integer a2 = new Integer(1);
        Integer a3 = Integer.valueOf(1);
        Integer b3 = Integer.valueOf(128);
        Integer a4 = Integer.valueOf(1);
        Integer b4 = Integer.valueOf(128);
        System.out.println(a0 == a1); //true 首先a1返回的是常量池中的对象，又因为和int进行比较所以进行拆箱操作
        System.out.println(b0 == b1); //true 同上，但是会在堆上new一个新的对象然后自动拆箱
        System.out.println(a1 == a2); //false 首先 a1是缓存在常量池的对象，a2是在堆上new的对象
        System.out.println(a3 == a2); //false a3返回的是放在常量池中的对象，两个对象不是同一个
        System.out.println(a3 == a1); //true 返回都是常量池的中的对象，比较地址相等
        System.out.println(a3 == a4); //true 返回都是常量池进行缓存过的对象，比较地址相等
        System.out.println(b3 == b4); //false 两个堆上新new对象，具体看valueOf源码，在上面
        System.out.println(a0 == a4); //true 拆箱比较
        System.out.println(b0 == b4); //true 拆箱
        System.out.println(b1 == b4); //false 装箱返回new的堆上对象 和 new的堆上对象
    }
```