---
title: fastjson升级版本引发bug的思考
cover: 'https://i.loli.net/2019/07/04/5d1ce6dcafb5137144.png'
categories: Java
tags: 
    - Java
    - Bug解决
    - 开发
    - fastjson
date: 2019-07-04 01:12:17
---

因为`fastjson`低版本的安全漏洞问题，所以公司所有服务全部要进行版本升级。

给公司升级`fastjson`版本时候发现了一个问题，在老版本中`key`的类型只能是`String`，而在新版本中可以为任意`Object`类型。但是在获取`keySet()`时候静态检测结果只能允许返回`String`。所以造成了运行时强转类型失败的异常。

<!-- more -->


### 问题起因

`JSON.parseObject()`的过程中老版本不会在乎你的`key`是什么类型，会直接`toString()`。而在新版本中增加了一个`特性开关`的选项：

1. fastjson 1.2.29 中，在进行json解析成object的过程中，会直接把key的类型转成String.
![pic1](https://i.loli.net/2019/07/04/5d1ce2a5d356444371.png)
2. fastjson 1.2.58 中，增加了一个`feature: NonStringKeyAsString`，只有在进行`parse()`时,指定转成`String`的key才会进行转化。
![pic2](https://i.loli.net/2019/07/04/5d1ce2e6f2bb298662.png)
![pic3](https://i.loli.net/2019/07/04/5d1ce39af2f9683542.png)

### 我的问题

下面的代码在`fastjson1.2.58`的条件下，进行编译是可以通过的但是运行就会报出 `java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String` 这个错误。这是为什么？

```java
public static void main(String[] args) {
    JSONObject jsonObject =JSON.parseObject("{64:\"64\"}");
    //keySet()声明的返回类型确实是Set<String> 符合遍历条件
    for (String x:jsonObject.keySet()) { 
        System.out.println(x);
    }
}
```
编译正确也就证明这段代码躲过了`Java的类型检查`。但是在运行过程中，发生了什么导致了**运行过程中得到的数据类型并不是声明的类型**。

### 先说结论

两个因素造成了这个问题： 1. 隐式的数据向上转换 2. 泛型擦除

1. 隐式数据向上转换：
    调用链中数据类型变化：`Map<String,Object> ---> Map ---> Object  最后---> Map<String,Object> `。问题就是,如果我在转换过程中Map存入的`key`是`Integer` 最后为什么会可以转成`Map<String,Object>`？

2. 这个可以用`泛型擦除`进行解释，运行时JVM不感知你的Map类型，所以在向下强转的时候其实转的是`Map<Object,Object>` 。`keySet()`获取的也是`Set<Object>` 。**但是！！** `foreach`进行遍历的时候就要去指定类了，根据静态检测编译的结果，你这个位置一定要声明成`String`的父类才可以，声明成`Integer`肯定会报`编译错误`的。当指定一个实际为`Map<Integer,Object>` 的`key`为`String`时就会报`java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String`异常了。

### 调试代码


1. 简单写了一个出现问题的demo。进行上面思路的图文解释
![小demo](https://i.loli.net/2019/07/04/5d1ce4b8c6c3255842.png)

2. JSONObject 强转成Map类型。注意返回的是Object

![2.1](https://i.loli.net/2019/07/04/5d1ce4d1de45528363.png)

![2.2](https://i.loli.net/2019/07/04/5d1ce4e28421453210.png)

![2.3](https://i.loli.net/2019/07/04/5d1ce4ef26d3540330.png)

3. 断点位置获取的key = Integer 

![3](https://i.loli.net/2019/07/04/5d1ce500dc14460917.png)

4. 存入之前的map。因为泛型类型被强转没了，所以可以存入

![4](https://i.loli.net/2019/07/04/5d1ce535ac5d746962.png)

5. 返回结过 ` object(或者说map)` -->向下强转 `JSONObject` 。由于JVM的泛型擦除，在运行时是没有任何泛型标识的，所以可以强转成功。JSONObject中的map其实是`Map<Object,Object>`
![5](https://i.loli.net/2019/07/04/5d1ce55171b3515352.png)


6. 然后在调用`keySet()`时候也是没有问题的，同样因为泛型擦除变成了`Set<Object>`的原因。 但是foreach时候就不一样了，foreach是要指定类型的，但是因为我们的强转，导致运行时的类型已经不再是我们声明的类型了。所以编译没有保存但是运行报错了。

![6](https://i.loli.net/2019/07/04/5d1ce57d1068952924.png)


### 最后，放一个可以复现Demo

#### 简易版demo 可以躲过静态类型检查，但运行报转换异常

``` java
    public static void main(String[] args) {
        HashMap<String,Object> map = new HashMap();
        Object object = parse(map); // 2️⃣ 类型第二次强转
        HashMap<String,Object> map2 = new HashMap<>();
        if(object instanceof HashMap) {
            map2 = (HashMap)object; // 3️⃣ 类型第三次强转
        }
        map2.keySet(); // 这里并没有报异常
//        for (Object x: map2.keySet()) { // 正确的代码。
//            System.out.println(x);
//        }
        for (String x: map2.keySet()) { // 错误的代码。编译没报错，因为自己声明的时候key的类型就是String。 但是因为我们的向上强转，导致存入的是Integer(Object)
            System.out.println(x);
        }
    }
​
    public static Object parse(final Map map){ // 1️⃣类型第一次强转
        map.put(123,"64");
        map.put("123","64");
        return map;
    }
```
#### 解决方案

1. 遍历时候用Object肯定没问题。

2. key尽量用标准的string。

3. 打开特性开关，让fastjson转化key的类型为string。下面是一个demo:
```java
JSONObject jsonObject =JSON.parseObject("{64:\"64\"}",Feature.NonStringKeyAsString); //开启转换开关。
```
**注：这个开关是在fastjson 1.2.42 时候才出现的，小于这个版本应该都不会出现强转的错误，高于这个版本的需要自己看看key是不是除了String的其他Object类型，是的话就要加上这个开关了。**
![注](https://i.loli.net/2019/07/04/5d1ce5e861cdb40435.png)


