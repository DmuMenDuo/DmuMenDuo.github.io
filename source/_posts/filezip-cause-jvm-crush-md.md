---
title: 'FileZip导致的JVM崩溃问题分析'
cover: 'http://pic.menduo.xyz/20190310003457.png'
categories: JVM
tags: 
    - Java
    - Bug解决
    - 开发
    - 日志
date: 2019-03-10 00:33:47
---

3.10 爬起床查日志，改bug.T_T.
<!-- more -->

## ZipFile 导致JVM奔溃现象分析

### 现象：
1. 日志无任何异常产生，但是服务进程已经死掉。定位到日志最后一行线程调用了`OnlineJudgeControler`，因此怀疑是慕码社区的代码（也就是我的锅T_T）导致的崩溃
2. 查看JVM崩溃日志`hs_err_pid**.log`，发现down机时，线程发出了SIGBUS信号，线程停在了ZipFile对代码包解压上。搜索了一下网上关于这方面的解释。

### 原因
具体原因就是：我的一个接口会去下载一个压缩包到一个特定的目录，然后在解开这个压缩包（这个文件被映射到了系统内存），但是如果此时另外一个线程也执行这个接口就会将这个压缩包重写（改写了硬盘文件）。导致第一个解压过程出现异常（因此此时会发现自己已经被另外一个线程改变了），从而导致了JVM直接崩溃。

**这里解释一下mmap内存机制：**

我们知道直接对磁盘上的内容会比从内存中读去慢很多，mmap内存文件映射其实就是将磁盘上的一个文件映射到内存中的作用。进程通过对内存的修改来达到对文件的修改，因此mmap系统调用常常用于进程间共享内存方式的系统调用。

**官方给出的关于mmap导致JVM崩溃的解释：**

Disabling mmap Usage (on Solaris or Linux) This release includes a new system property, sun.zip.disableMemoryMapping, which allows the user to disable the mmap usage in Sun's java.util.zip.Zipfile implementation (on Solaris and Linux platforms). Solaris or Linux applications that use java.util.zip.ZipFile may experience a SIGBUS VM crash if the application accidentally overwrites any zip or jar files that are still being used by the same Java runtime. Although this is a programming error of the offending application, this system property provides a solution to avoid the VM crash. With the property set to true (-Dsun.zip.disableMemoryMapping=true, or simply -Dsun.zip.disableMemoryMapping) the Sun JDK/JRE runtime disables the mmap usage and the VM crash that might otherwise occur by overwriting the jar or zip file can be avoided.


### 解决方法

1. 官方说明的那样对zip禁用mmap
> 在启动参数中加入-Dsun.zip.disableMemoryMapping=true 
2. 对代码进行修改，防止多个进程访问同一个文件，可以考虑加锁，或者对该文件进行备份，每个线程访问不同的文件


### 参考资料

> 参考资料1 https://blog.csdn.net/zqixiao_09/article/details/51088478
> 参考资料2 https://my.oschina.net/shipley/blog/979231