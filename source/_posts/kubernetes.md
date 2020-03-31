---
title: kubernetes
date: 2019-07-29 21:13:14
categories:
- 分布式系统
- kubernetes
tags:
- 秋招
---
## Pod
pod 在启动的时候会创建两种类型的容器，一种用于初始化一些配置，另一种就是Running时的容器，卷是容器中持久化存储数据的空间，pod在启动的过程中只有当卷被挂载到目的地上之后，pod启动的流程才能继续下去，而pod在重新启动的时候，只需要把目录重新挂载就好，并不需要清除存储空间的数据

### 网络
同一个pod中的不同的容器是共享网络的，他们之间通过端口号进行标识，同一个pod上的所有的容器都会链接到同一个网络设备上，由Sandbox中的沙箱容器，就是`pause`容器

#### Pause容器
> In Kubernetes, the pause container serves as the "parent container" for all of the containers in your pod. The pause container has two core responsibilities. First, it serves as the basis of Linux namespace sharing in the pod. And second, with PID (process ID) namespace sharing enabled, it serves as PID 1 for each pod and reaps zombie processes.

从概述上来看，他的意思是Pause容器作为所有容器的父容器，有以下两个作用：
1. 共享Linux的命名空间
2. 用于回收进程，释放进程空间，作为PID为1的进程

功能实现上来说，它首先是创建了一个namespace，然后再把其他的docker加入到了已有的命名空间中，
同时作为一个pid为1的进程，用于回收僵尸进程

### pod的生命周期
从 create 到 probe 到 running  到 shutdown 到 restart，整个流程的说明如下：

1. pod被创建之后，需要检查健康状态，当前Pod 已经能够接受外部的请求时，才会将流量打到新的 Pod 上并继续对外提供服务 
2. Pod 被删除之前都会触发一个`PreStop`的钩子，处理完之后才会删除这个pod，因此可以在这里进行优雅的删除

#### 创建
Pod 的创建都是通过 SyncPod 来实现的，创建的过程大体上可以分为六个步骤：

1. 计算 Pod 中沙盒和容器的变更；
2. 强制停止 Pod 对应的沙盒；
3. 强制停止所有不应该运行的容器；
3. 为 Pod 创建新的沙盒；
4. 创建 Pod 规格中指定的初始化容器；
5. 依次创建 Pod 规格中指定的常规容器；

每一个容器的创建的过程如下所示：

1. 通过镜像拉取器获得当前容器中使用镜像的引用；
2. 调用远程的 runtimeService 创建容器；
3. 调用内部的生命周期方法 PreStartContainer 为当前的容器设置分配的 CPU 等资源；
4. 调用远程的 runtimeService 开始运行镜像；
5. 如果当前的容器包含 PostStart 钩子就会执行该回调；

#### 健康性检查

1. Kubelet使用liveness probe（存活探针）来确定何时重启容器
2. Kubelet使用readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量

检查的方式可以如下所示：

**liveness 命令**

间隔一段时间就会去执行这个命令，判断系统是否还在运行中

```yaml
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
       initialDelaySeconds: 5
       periodSeconds: 5
```

**liveness HTTP请求**

定期会向8080端口发送一个请求，去校验这个服务是否还在运行中

```yaml
livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
          - name: X-Custom-Header
            value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```



**liveness TCP请求**

TCP检查的配置与HTTP检查非常相似。 此示例同时使用了readiness和liveness probe。 容器启动后5秒钟，kubelet将发送第一个readiness probe。 这将尝试连接到端口8080上的goproxy容器。如果探测成功，则该pod将被标记为就绪。Kubelet将每隔10秒钟执行一次该检查。

```yaml
 livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

**Readiness 探针**

Readiness probe的配置跟liveness probe很像