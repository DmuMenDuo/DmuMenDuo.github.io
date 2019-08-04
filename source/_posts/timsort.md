---
title: TimSort遇到Comparator抛出异常IllegalArgumentException的那些事儿
cover: 'https://i.loli.net/2019/08/04/LPQotVUkIKSFJn8.png'
categories: Java
tags: 
    - Java
    - TimSort
    - 归并排序
date: 2019-08-04 21:27:13
---

最近组里面在一起学习java开发规范，其中遇到的一点就是Comparator一定要满足对称性和传递性，类似于重写equals方法一样的方式。这是为什么呢？

<!--more-->
下面是Java开发规范中的描述：

> 在 JDK7 版本及以上，Comparator 实现类要满足如下三个条件，不然 Arrays.sort，Collections.sort 会抛 IllegalArgumentException 异常。
> 说明:三个条件如下:
> 1) x，y 的比较结果和 y，x 的比较结果相反。 对称性
> 2) x>y，y>z，则 x>z。传递性
> 3) x=y，则 x，z 比较结果和 y，z 比较结果相同。

```Java
    //反例
    new Comparator<Student>() {
        @Override
        public int compare(Student o1, Student o2) {
            return o1.getId() > o2.getId() ? 1 : -1;
        }
    }
```
​
### 1.先说结论：

1. 归并排序过程会出现`原数组S`中的`部分有序数组A`和`有序数组B`, `A`和`B`会进行merge成一个`整体有序`的`数组S`。

2. 归并排序的优化过程对于`比较函数不满足条件的比较`会出现`悖论`，即比较过程中出现` B[m] < A[n] < A[n+1] < B[m]`，出现这个过程就是`compare方法`错误，排序算法识别出来这个问题，被`throw`出去了,这个异常就是 `java.lang.IllegalArgumentException: Comparison method violates its general contract!` 比较函数不合规。

3. **为什么jdk6以前没出现问题，jdk7之后会出现问题？**jdk6以前是使用传统的归并排序并没有严格的检查，而jdk7以后使用TimSort算法,其中做了多处优化并进行了严格检查。

### 2.参考资料：

结合源码以及自己实际测试，下面两篇文章可以进行参考。

1. https://linsky328.iteye.com/blog/2382272 

2. https://www.coder4.com/archives/4092

### 3.出现原因分析：

`Arrays.sort`使用的排序算法`TimSort`,这个算法是二分插入排序和优化版的归并排序的结合使用的排序算法。时间复杂度会比O(nlogN)要快，很多主流的编程语言的内置排序算法都是`TimSort`

#### 3.1 TimSort的基本原理：

TimSort 是一种基于二分插入排序和归并排序相结合的方式的排序算法。将待排序数组分成一个个的minLen段，对minLen段用二分插入排序排好放入栈中，然后对栈中的排好序的minLen段进行归并排序。

##### 3.1.1 二分插入排序

这里也有个优化，在去切分minLen段的时候，并不是一开始就从起始位置执行二分插入排序，而是先找到最长的升序和降序段，如果长度过长直接放入栈中等待归并，如果太短，才会去执行二分插入。

![3.1.1](https://i.loli.net/2019/08/04/JGoLinWKxzOREVP.png)

##### 3.1.2 归并排序

它用了两个栈`runLen`和`runBase`来维护归并过程：第一个栈保存了每个块的长度(runLen)，第二个栈保存了每个块在原数组中的起始位置(runBase)

**问题：什么时候进行归并?**

整个归并的逻辑代码就是下面这块：

```java
// 栈中元素>=3个的时候，如果第一个小于第三个块
private void mergeCollapse() {
    while (stackSize > 1) {
      int n = stackSize - 2;
      if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1]) {
        if (runLen[n - 1] < runLen[n + 1])
          n--;
        mergeAt(n);
      } else if (runLen[n] <= runLen[n + 1]) {
        mergeAt(n);
      } else {
        break; // Invariant is established
      }
    }
}
```

我一开始没理解，所以画了下面两张图来帮助理解，用一句话概括就是归并的非递归实现也要把过程想象成一颗递归树，同等level的才进行合并。

- 栈中出现三个块的情况首先判断：

![1](https://i.loli.net/2019/08/04/WQrj67UtVmGbJiC.png)

- 栈中只有两个块的情况，分两种：

 1. 第一种`Len(n) <= Len(n+1)` 那么第一个块是没被合并过的，赶紧合并掉。

 2. 第二种就是Len(n)很大，说明之前合并过，和Len(n+1)不属于一个level，暂时不合并，break。

- 具体的merge过程

    1. 块A 和 块B 首先执行第一处优化:找到base2在块A中的位置，前面的均比base2小，后面的均比base2大。去掉块A中比base2小的元素（去头），同理块B去掉比块A尾部都大的元素（去尾），这个过程我们可以得到**第一个条件**`A[base1] > B[base2]`  (**gallopRight应用比较函数**: `compare(B[base2],A[base1]) >= 0 == false）` 得到下面这张图：

    ![](https://i.loli.net/2019/08/04/gRmCPZhYcQELioz.png)

    2. 之后进行正常的归并的merge操作，当发生块A或者块B的比较多次胜利后，会触发优化，直接找到base2在块A中的位置，然后移动cursor1到新的位置，然后把块B全部拷贝到到了A中，触发**第二个条件** `B[base2] > A[base1+1]` （这时在查询应用到的比较函数是：`(compare(A[base1+1],B[base2]) <= 0 == true)）` **gallopLeft函数**

    ![](https://i.loli.net/2019/08/04/yvs61OctYSmQLIh.png)
    
    ![](https://i.loli.net/2019/08/04/IijTdgC34eNKWho.png)
    

 3. 根据最初的条件 `A[base1] < A[base1+1]`,我们可以得出 `B[base2] < A[base1] < A[base1+1] < B[base1]` 。 这个时候的特征就是`cursor2 == -1 `并且`len(tmp) == 0 `所以抛异常

    ```java
    else if (len2 == 0) {
        throw new IllegalArgumentException(
                "Comparison method violates its general contract!");
    }
    ```

    上面直接用的参考资料里的图，我自己也亲自做了测试，确实会发生上面的情况，导致cursor2  = -1 len2 = 0的情况


### 4.最后总结：

其实最根本的问题在于`compare函数`的使用上。在最后merge过程中`compare(A,B)>=0` 和 `compare(B,A) < 0` 这两个是无法等价的。所以产生了悖论，抛异常，告知这个比较方法是错误的