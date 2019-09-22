---
title: 批量保存的一些思考
cover: 'https://i.loli.net/2019/07/22/5d35b8eda101f46703.png'
categories: 
    - Java
tags: 
    - Java
    - Lambda
    - 优化
date: 2019-07-22 21:08:22
---

最近实习一直在不停的接各种小需求做，这次就是导师给的一个任务，解决目前系统批量查询接口慢而且重复代码过多的问题，如何设计一个简洁又通用的方案。

<!-- more -->

### 1.问题

查询接口目前有两种： 1. 单个数据查询 2. 批量数据查询。 但他们都或多或少存在一些问题。

1. 单个数据查询接口如果进行批量的数据查询就只能在外面包一层`for`循环。这样执行sql会造成大量的磁盘的IO操作，性能很差。for循环查mysql，公司是明令禁止的，一定要写批量操作查询。

2. 如果改用批量数据查询，那么对于大批量数据的查询，在`Mysql`用到的`in语句`就会有大量的参数，`Mysql`执行语句的长度有大小限制，并且我们知道`in`在包含大量参数的执行上，效率不是很高 (见参考文献)。

**为什么直接批量全部数据不好？** 数据库的连接是要通过连接池的，每一个`db操作`都会占用一个连接，如果这个批量数据的接口并发量很高的话，会对同时需要访问数据库的其他`db操作`造成干扰。如果用`for循环`的方式一个一个查询，又会造成自己的查询过慢的情况。所以选择一种折中的方案

**如何统一分散到各个方法中的分片逻辑？** 目前项目中的批量查询都是`传入List传出List的方法函数`，想到了写一个分片的util，传入一个方法函数返回这个函数的返回结果。

### 2.方案

所以有了一种折中的解决方案: 对查询进行分片，分批进行查询，最后进行结果聚集。这样可以**减少IO次数**并且可以**限制in中参数的个数**。

第一个版本批量查询接口的主要功能就是先进行分片，之后对分片数据执行传入的函数得到分片结果，最后对分片的结果进行聚集成最后的结果并返回。

![2](https://i.loli.net/2019/07/22/5d35b5facb85f75788.png)

我在想这个方法的优化时候，首先想到的就是这个过程特别像`mapreduce`的过程。先`partition`之后`map`分批运算最后进行`reduce`。所以这个过程我觉得是可以进行`并行`的。第二点就是传入的参数是`List<Integer>`，不能支持批量查询Long类型的参数，例如我将要写的批量接口就是根据bigint类型的查询， 所以我改成下面这种形式：(返回Steam会不会灵活性更好一点？可以转List,可以转Set)

![2.1](https://i.loli.net/2019/07/22/5d35b62d4fbbc80535.png)

### 3.测试

那么改成这种方式，效率真的提高了吗？

做个测试，假设每一次请求大致会花费100ms。这个过程可以用线程阻塞来模拟。所以用了sleep。

![3](https://i.loli.net/2019/07/22/5d35b6404e6de59146.png)

串行查询代码如下，串行批量查询接口：`104094ms`

```java
    //执行时间104094
    public static void main(String[] args) {
        List<Integer> list = Lists.newArrayList();
        for (int i = 0; i < 1000; i++) {
            list.add(i);
        }
        long start = System.currentTimeMillis();
        List<Integer> list2 = findByIds(list,10, LoopUtils::add);
        System.err.println(System.currentTimeMillis() - start);
    }
```

并行查询代码如下，并行批量查询接口：`14664ms`

```java
    //执行时间14664
    public static void main(String[] args) {
        List<Integer> list = Lists.newArrayList();
        for (int i = 0; i < 1000; i++) {
            list.add(i);
        }
        long start2 = System.currentTimeMillis();
        List<Integer> list1 = findByIdsParallel(list, 10,ids ->LoopUtils.add(ids).stream());
        System.err.println(System.currentTimeMillis() - start2);
    }
```
并行是和CPU核数相关的，和想象中的差不多，我的MBP是8C的 ，并行比串行大致快了7倍左右。

**上面的测试其实没有说明什么本质上的问题：**

1. 单个接口单个测试，没有模拟并发的场景。
2. 用Thread.sleep()代替db操作

### 4.新的问题

1. **结果是乱序的**。如果对查询结果有顺序要求，那么就要对结果进行排序。例如很多查询结果get(0)的操作就不适合用parallel了

2. **CPU爆满效率可能不如串行**。真实的场景中，8核理论上都是在被使用的，如果全部爆满的情况下，因为多了更多的线程切换操作，可能运行效率还不如串行执行。

### 5.参考资料

1. https://blog.csdn.net/shooke/article/details/56673523 一个博主关于in中传入不同参数数量的查询时间的测试。大意就是in中个数越多，查询越慢？但是为什么？

2. 看到以一种嵌套查询的解释，https://www.cnblogs.com/wxw16/p/6105624.html时间复杂度是O(N + N*M)，但是对于直接传入List.size() = M,那么查询效率就是O(M)，其实对于几百个参数的查询还是可以接受的

3. https://www.jianshu.com/p/bd825cb89e00 parallelStream背后的男人ForkJoinPool。

4. http://gee.cs.oswego.edu/dl/html/StreamParallelGuidance.html when to use parallelStream

