---
title: 使用Docker-Java动态的获取Container的log日志
cover: 'http://pic.menduo.xyz/20181119154259784332615.jpg'
categories: docker
tags: 
    - docker-java
    - bug解决
    - 开发
    - 日志
date: 2017-12-13 23:21:16
---

docker-java 是 JAVA 操作Docker Remote API 的一个工具。最近项目需要读取容器的运行日志出来。记录一下自己遇到的问题。更多其他的问题可以参考他们的测试用例。
> github地址: [docker-java](https://github.com/docker-java/docker-java) 目前版本 3.0.13

<!-- more -->

### 准备工作
#### 1. 首先要开启docker的客户端 (我电脑 ubuntu16.04)

```bash
#查看配置文件位于哪里
systemctl show --property=FragmentPath docker 
#编辑配置文件内容，接收所有ip请求
sudo gedit /lib/systemd/system/docker.service  
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://127.0.0.1:8787
# 这样配置的目的是可以通过http的方式进行远程访问，如果不需要的话可以通过自带的sock方式本地访问
#重新加载配置文件，重启docker daemon
sudo systemctl daemon-reload     
sudo systemctl restart docker 

```

#### 2. 建工程 && pom.xml引入依赖

```xml
<dependency>
  <groupId>com.github.docker-java</groupId>
  <artifactId>docker-java</artifactId>
  <version>3.0.13</version>
</dependency>
```

### 获取容器实时运行结果


> 通过查看容器日志的方式,
> 设置一个回调函数，重写onNext(),就可以取到运行的结果了

```java
dockerClient.logContainerCmd(dockerId).withStdOut(true).withStdErr(true)
                    .exec(new LogContainerResultCallback(){
                        @Override
                        public void onNext(Frame item) {
                            super.onNext(item);
                            System.out.println("容器运行的结果："+item.toString())
                        }
                        @Override
                        public void onComplete() {
                            super.onComplete();
                        }
                    }).awaitCompletion();
```

> `awaitComletion()`就是让查日志进程阻塞，直到进程调用onComlete()
> 最好等容器停止后再去读，因为容器在运行中可能还没产生日志，但callback读的时候以为读完了，直接执行onComplete就结束了，因此没有输出。 所以建议查日志前，检查一下容器运行情况，代码如下：之后一直循环判断直到容器停止就可以了

```java
	//获取容器的运行状态
	dockerClient.inspectContainerCmd(dockerId).
                exec().getState().getRunning();
```


