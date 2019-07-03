---
title: 线程池中的阻塞队列知识点总结
cover: 'http://pic.menduo.xyz/20190511232556.png'
categories: Java
tags: 
    - Java
    - 线程池
    - 阻塞队列
    - BlockingQueue
date: 2019-05-11 14:16:36
---

本文主要从线程池出发，具体了解一下其使用的各种不同的阻塞队列底层「放入」和「取出」是如何实现的。最后，具体介绍同步阻塞队列`SynchronousQueue`的底层实现和在线程池中的应用。

通过本文：我可以收获到的是----一个线程池参数：阻塞队列的相关知识。多看源码多思考，秋招offer少不了。hhhhh

❤️------------------❤️

<!--more -->

## 线程池中的阻塞队列知识点总结

队列是线程池创建的一个重要参数，java提供了几种不同数据结构实现的阻塞队列，应对不同的线程池需求要选取适合的队列作为参数。
![阻塞队列](http://pic.menduo.xyz/20190511232556.png)


### 几种阻塞队列简要介绍

- `ArrayBlockingQueue`：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。 `数组`
- `LinkedBlockingQueue`：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列 `链表`
- `SynchronousQueue`：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。 `队列和栈两种`
- `PriorityBlockingQueue`：一个具有优先级的无限阻塞队列。`堆`


### 线程池中的阻塞队列的一些思考

1. 线程池中的阻塞队列是做什么的？
2. 队列里放入和取出的操作有很多，线程池用了阻塞队列的哪两个方法？
3. 阻塞队列可以放入Null吗？
4. 无界阻塞阻塞队列长度真的是无大小限制的吗？
5. 阻塞队列的put和take如果自己写应该如何实现？

#### 1. 线程池中的阻塞队列是做什么的？

用于存放实现了`Runnable接口`的可以被`工作线程worker`执行的`任务task`。四种不同的阻塞队列分别可以对应多种不同的实际使用场景。

#### 2. 队列里放入和取出的操作有很多，线程池用了阻塞队列的哪两个方法？

看线程池的源码会发现是`offer()`和`remove()`，对，没错！用了两个非阻塞的方法。。。如果是有界阻塞队列，在队列满了的情况下执行`offer()`将`command`放入阻塞队列，会直接返回`false`。看下面的代码：
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { //① offer
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command)) //② remove
                reject(command);
            else if (workerCountOf(recheck) == 0) 
                addWorker(null, false);
        }
        else if (!addWorker(command, false)) //加入队列失败，在放worker试试，不行再拒绝也不晚。（worker弹性扩容）
            reject(command);
    }
```

`offer()`和`remove()`方法，在成功的时候会返回`true`,放入失败则返回`false`。所以这个是一种非阻塞的模式。为什么是非阻塞的`offer()`和`remove()`而不是使用阻塞的`put`和`take`？

**A：** 我是这样考虑的，如果放不下就阻塞，那么会出现大量提交`task`线程堆积阻塞的情况，而线程池就是为了复用线程，它只是为了用几个一直运行的`worker`线程去调用实现`Runnable接口`的对象的`run()`方法。所以在队列放不下的情况应该直接返回，对于失败的情况应该自己用拒绝策略去处理。这样提交task的线程就不会产生堆积。`run()`是无返回的，不用阻塞，`call()`有自己的处理机制，可以阻塞。

#### 3. 阻塞队列可以放入Null吗？

`null` 被用来作为 `poll` 操作失败的返回值，为了避免这个操作产生二意性（poll一个空值，还是poll失败），所以不允许放入`null`。

```java
 /*
 * <p>A {@code BlockingQueue} does not accept {@code null} elements.
 * Implementations throw {@code NullPointerException} on attempts
 * to {@code add}, {@code put} or {@code offer} a {@code null}.  A
 * {@code null} is used as a sentinel value to indicate failure of
 * {@code poll} operations.
 */
```

#### 4. 无界阻塞队列长度真的是无大小限制的吗？

看了下`LinkedBlockingQueue`的构造函数,不设置初试长度，那么其值为整型的最大值。
```java
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

#### 5. 阻塞队列的put和take如果自己写应该如何实现？

最简单的方式就是使用`互斥锁+条件`的方式，定义锁的两个条件（1. 队列满；2. 队列空）。put方法当出现`队列满`放不下的情况时当前线程阻塞，等待take操作产生的唤醒操作。简单写了下代码如下：

```java
public class BlockedQueue {
    private int size = 3;
    private ReentrantLock lock = new ReentrantLock(); // 锁
    private Condition fullCondition = lock.newCondition(); 
    private Condition empyCondition = lock.newCondition();
    private ArrayList<String> blockQ = new ArrayList<>();

    public void put(String x) throws InterruptedException {
        lock.lock(); //多个put的线程阻塞在这里
        while (blockQ.size() >= size) {  // while 让获得锁的线程 阻塞在添加前，这样避免了死锁，换成if的话，它被唤醒的时候条件不成立导致add会出现超量的情况
            fullCondition.await();
        }
        blockQ.add(x);
        empyCondition.signal();
        lock.unlock();

    }

    public String take() throws InterruptedException {
        lock.lock();
        String result = "";

        while ((blockQ.size() <= 0)) {
            empyCondition.await();
        }
        result = blockQ.remove(0);
        fullCondition.signal();
        lock.unlock();
        return result;

    }
}
```

### 关于SynchronousQueue

写完上面的代码之后，我在想的一个问题是:我在上面用`ReentrantLock`实现的阻塞队列的`put`和`take`操作用了`条件+双锁`的模式。这样阻塞的方式会造成的问题就在于如果同时多个线程去`put`数据，只会有一个线程获得锁，其他线程进入ReentrantLock的等待队列中等待被唤醒，这样会造成线程的频繁切换，消耗大量的CPU资源。

线程池中的几种阻塞队列中，`ArrayBlockingQueue`,`LinkedBlockingQueue`,`PriorityBlockingQueue` 都是使用上面的`双锁`的方式实现的。

而`SynchronousQueue`的实现却和上面几个阻塞队列的实现完全不同，它的实现方式是什么，为什么要这样子实现？

#### 1. 理解SynchronousQueue是什么?

`SynchronousQueue`同步阻塞队列，同步意味着放入和取出操作总是成对出现的，也就是说put一个object,就一定对应着一个take操作，配对成功两个线程才会继续运行。这样子抽象看的话就没有任何一个元素是放到队列里，队列一直处于一种又满又空的状态。这样子可以理解为同步阻塞队列的队列长度一直是0。

所以`SynchronousQueue`的peek()为null,isEmpty()为true等等。

```java
    public E peek() {
        return null;
    }
    public boolean isEmpty() {
        return true;
    }
```


其实理解什么是`SynchronousQueue`之后，我们可以用`ArrayBlockingQueue`设置队列长度为`1`来实现一个乞丐版的`SynchronousQueue`。这样做造成的两个问题：

1. 就是频繁的加锁和解锁操作，涉及了线程频繁切换。
2. 吞吐量只有1，线程池完全串行化了，没有办法进行并行操作。

所以jdk6之后的采用了一种无锁算法来优化上面两个问题。

无锁算法解决上面的问题：

- 这种无锁算法(更多内容见参考资料)定义了两种数据结构。保证公平的FIFO队列，和非公平的LIFO栈。用来存储每一个put操作和take操作。
- 队列和栈中保存的节点存在三种状态，REQUEST(消费者)，DATA（生产者），FULLFILLING(消费者和生产者匹配成功）。
- put操作加入到队列中后，会进行自旋等待，直到有个take操作transfer自己put进来的元素。put线程和take线程才会返回。
- 不用加锁的原因：在多线程下，每个线程可以通过状态来处理自己产生的状态（put or take），同时也可以处理其他线程产生的状态（帮助他移出fullfiling节点或者进行匹配），数据的状态以及CAS操作保证了多线程争用情况下的线程安全。具体看下面的代码。

#### 2. SynchronousQueue的无锁同步算法

用于保证put和take成对出现的核心方法就是transfer()，这个方法的语义：线程A把一个元素E交给线程B，或者线程A从线程B中拿到元素E。

主要进行三种不同状态之间的变换：

1. head节点状态和当前要加入的节点状态相同（都是生产者或者都是消费者），则要把当前线程作为节点加入到栈中，并更新head指向当前节点（考虑竞争情况下，使用原子的CAS操作）
2. 如果head节点是不是fullfilling,则将当前节点标记Fullfiling状态并加入到栈中，更新head节点和匹配节点的match值。然后返回结果就可以了，自己可以不用处理队列状态了。
3. 如果当前节点是fullfiling节点，移出fullfilling以及和它match的成对节点，下个额循环在处理自己这个节点。

> 下面的代码是非公平的栈的实现：

```java
E transfer(E e, boolean timed, long nanos) {
            //栈的模式
            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA;  //REQUEST:请求拿到一个值，DATA请求放一个值

            for (;;) {
                SNode h = head;
                if (h == null || h.mode == mode) {  // empty or same-mode
                    if (timed && nanos <= 0) {      // can't wait
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node
                        else
                            return null;
                    } else if (casHead(h, s = snode(s, e, h, mode))) { //cas操作将当前节点设置到head
                        SNode m = awaitFulfill(s, timed, nanos); //自旋、阻塞等匹配的节点
                        if (m == s) {               // wait was cancelled
                            clean(s);
                            return null;
                        }
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                } else if (!isFulfilling(h.mode)) { // try to fulfill
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }

```
#### 3. 线程池里是如何使用SynchronousQueue?

线程池的工厂类`Executors`创建的`CachedThreadPool`的队列选择就是`SynchronousQueue`。`CachedThreadPool`线程池的特性就是弹性的扩张，当有`task`进来的时候，就会分配一个`worker`来运行这个`task`;如果没有可用`worker`就会new一个新的`worker`。当`worker`空闲一段时间后，还会自动销毁。

##### 3.1. 首先看一下创建`CachedThreadPool`的工厂方法。默认情况下,核心池大小为0（初始化不创建worker）。核心池设置为最大值。`worker`存活时间为60s。阻塞队列为`SynchronousQueue`

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
```

##### 3.2. 然后我一直在想的一个问题是，`SynchronousQueue`队列长度为0，`SynchronousQueue`的`offer`操作如果碰到`队列满`的情况会直接返回`false`。所以在线程池工作最开始的时间是`没有woker`的情况,`offer`操作一定都是返回`false`的。线程池如何处理这个`offer`进来的`task`的?

**A:**看了下源码：在`①`这个位置根据上面`SynchronousQueue`的特性如果明确知道这里的判断为`false`，所以走了`②`，`add`了一个新的`worker`。

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { //①
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false)) //②
            reject(command);
    }
```

##### 3.3. worker线程处理完一个任务之后也会进行offer操作，所以这个时候它会阻塞住吗？(这个问题蠢吗。。我其实想了好久才想明白的)

**A:** 看源码可以发现是会阻塞住，会在`SynchronousQueue`中加入一个`take`节点,他会等待一个`offer`节点match自己。同时`keepAliveTime`woker的存活时间也是在这里发挥的作用。

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            //...略
            try {
                //①
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```



## 参考资料
1. [非阻塞实现](http://www.cs.rochester.edu/research/synchronization/pseudocode/duals.html)
2. [非阻塞算法简介](https://www.ibm.com/developerworks/cn/java/j-jtp04186/)
3. [SynchronousQueue简介](https://www.cnblogs.com/duanxz/p/3252267.html)
4. [源码分析-SynchronousQueue](https://blog.csdn.net/u011518120/article/details/53906484)
