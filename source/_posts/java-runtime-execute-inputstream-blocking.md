---
title: 项目中java创建子进程造成的I/O阻塞问题定位和解决
cover: 'http://pic.menduo.xyz/20190512114203.gif'
categories: Java
tags: 
    - Java
    - Bug解决
    - 开发
date: 2019-04-27 23:36:26
---

记录一个bug的debug过程，bug起因是`后台调度程序`启动一个异步任务去调用一个docker容器包裹的工具服务（mutation变异测试工具），正常运行的情况下docker容器会正常的退出并自我销毁。而这次的现象是docker容器没有退出而一直处于运行状态。

❤️------------------❤️

<!-- more -->

## 原因分析

### docker下手
分析`docker`未停止的原因大概率是docker所做的事情没有结束。所以，`docker exec`进入容器，用`jps`查看`Java进程`状态，果然发现进程并没有运行结束。

### 观察java进程

1. `jps`观察现象，有两个java进程在运行，`pid=430`就是我们的主进程，`pid=300`是主进程内新启动的一个子进程。
```bash
300 MutationTestMinion
430 Jps
```
2. `jstask [pid]` 查看两个进程的线程栈。 
**查看主进程的线程栈日志如下:**
![主进程](http://pic.menduo.xyz/20190501204936.png)
**查看子进程的线程栈日志如下:**
![子进程](http://pic.menduo.xyz/20190501212950.png)
**现象：**
主线程正在读取inputStream,子线程正在写入outputStream。两个进程均阻塞住，无法继续运行。

### 分析代码

1. 拉代码，本地定位到具体的关键代码如下（比查bug更痛苦的就是查别的的bug了）
```java
String[] commands = {"bash", "-c", commandStr};
Process p = Runtime.getRuntime().exec(commands);
output = IOUtils.toString(p.getInputStream());
``` 

2. 查看`Process`的文档和各种博客：

**收集的有用的信息：**
- 创建`Process`方式有两种： ① `ProcessBuilder`; ② `Runtime`.
- 创建的`Process`没有自己的终端和控制台，他们的`标准I/O（stdin,stdout,stderr）`会被重定向给父进程，为此Process提供了方法`getOutputStream()`,`getInputStream()`,`getErrorStream()`来实现这个功能。
- **重点：** 由于各平台对于标准I/O 设置了有限制的缓冲区大小。导致写入缓冲区可能会发生阻塞或者死锁的现象！！！原文：
``` java
 /*
 * The parent process uses these streams to feed input to and get output
 * from the subprocess.  Because some native platforms only provide
 * limited buffer size for standard input and output streams, failure
 * to promptly write the input stream or read the output stream of
 * the subprocess may cause the subprocess to block, or even deadlock.
 */
```

## Bug详细解释

通过上面的逐步分析，已经可以明显的知道原因是什么了：

`子进程`在执行的过程中会向标准I/O的缓冲区中写入`标准输出`和`错误输出`，而主进程没有对子进程的`错误输出`进行处理，导致子进程发现`错误输出`缓冲区满了的时候自己阻塞住，无法继续运行。而主进程因为子进程无法运行结束，所以`getInputStream()`一直认为还会有数据，会一直等待来自子进程的`标准输出`，所以也阻塞在了`getInputStream()`。 程序中没有任何的超时机制，java进程阻塞住，所以docker也就阻塞住了。

## 解决&引入新的bug 

关键点就是 要及时处理子进程的`标准输出`和`错误输出`,因此我们可以启动两个线程分别去处理两个输出，然后主进程等待子进程运行结束就可以了，关键代码如下：
```java
ExecutorService single = Executors.newCachedThreadPool(); //启动个线程池
Future<String> errorOutput = single.submit(() -> IOUtils.toString(p.getErrorStream()));
Future<String> inputOutput = single.submit(() -> IOUtils.toString(p.getInputStream()));
single.shutdown(); // 这里一定要主动关闭线程池。
p.waitFor(); //主线程挂起，等子进程结束
output = inputOutput.get(); // 获取输出结果
```
程序首先启动了一个线程池，添加了两个任务分别去处理标出输出和错误输出。然后关闭线程池，拒绝新的任务加入进来，主进程等待子进程运行结束后，再去获取线程池中的线程中的运行输出结果。

我在写这个代码的时候引入了一个新的bug,就是没有手动关闭线程池,导致我有一段时间怀疑我之前看的文档和bug分析是不是错的，所以又去复现了一波，发现处理任务的子进程已经运行结束了，主进程park在了`ThreadPoolExecutor.getTask()`, 线程池在确认没有任务加入的时候一定要手动关闭`shutdown()`,否则会导致主进程因为有后台线程在运行的缘故，无法运行结束，造成的结果也是docker阻塞住，其实就是线程池在等待处理新的task而进行的自旋阻塞。幸好因为面试看过线程池源码，确认是线程池阻塞在等待任务的问题。所以多看源码还是对自己很有帮助的，有机会再总结一波线程池相关的源码分析。
![线程池阻塞](http://pic.menduo.xyz/20190502153421.png)


## 后记

1. 关于线程池状态的说明

```java
/*  -1 RUNNING:  Accept new tasks and process queued tasks 可以接受新的任务并处理
*   0 SHUTDOWN: Don't accept new tasks, but process queued tasks 不再接受新的任务，但是会处理完队列里的所有任务
*   1 STOP:     Don't accept new tasks, don't process queued tasks, 
*             and interrupt in-progress tasks 不在接受任务，也不处理队列里的任务，同时中断正在处理任务
*   2 TIDYING:  All tasks have terminated, workerCount is zero,
*             the thread transitioning to state TIDYING
*             will run the terminated() hook method 所有任务终止，worker数为0 线程改变状态为TIDYING 
*   3 TERMINATED: terminated() has completed 终止方法运行完成
*/
```


2. 为什么这里线程池需要手动shutdown，直接看源码发现，线程池会不断的去获取`task`放入`worker`执行的代码是一个for死循环，其中的退出条件:
   - 线程池状态码 >= SHUTDOWN 并且 （状态 >= STOP || 队列里没任务) 这个看上面的状态码就可以理解了，我们手动调用shutdown的原因就是这个
   -  从队列里获取一个task,获取成功则返回

```java
private Runnable getTask() {
        //...略
        for (;;) {
            int c = ctl.get(); 
            int rs = runStateOf(c); //获取线程池状态
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            //...略
            try {
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

1. Java doc
2. [Process.getInputStream()阻塞问题](https://blog.csdn.net/yuanzihui/article/details/51093375)
