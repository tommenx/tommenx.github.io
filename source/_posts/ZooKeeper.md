---
title: ZooKeeper
date: 2017-11-27 19:24:21
categories:
- 分布式系统
- MIT6.824
tags:
- mapreduce
---
zookeeper是一个分布式的协调系统，目标是封装好关键的服务，提供简易使用的借口，建立更加复杂的协调原语。
并且，zookeeper是高效的，适用于以读为主的服务请求，任意一个server都可以服务于read请求，查找local replica并返回结果，而写操作则需要通过leader，并确保数据的一致性

{% asset_image service.png %}
一组server能够提高性能

## Core feature
在能大概读懂文章的内容之后，有以下的几个特点印象特别深刻：
### Wait-free
zookeeper最大的特点就是高效的，对比其他同步的算法，采用异步的算法能够有效的防止部分machine而阻塞了整个服务进程。
但是采用异步的方式难以保证一致性，但采用**FIFO Client Order**和**Linearizable Writes**保证了系统的协调性
**move away from blocking primitives,such as locks**

### Complicated imformation
Znode并不是被设计来存储数据的，但可以存储复杂的meta-data和configuration
### Znode
znode可以被设置 `sequential flag`，当`emphemral node`被删除后，同时也被删除。
同时采用递增的命名方式，序号小的是先被设置的----可以用于锁的设置
采用类似UNIX的命名方式进行管理，如图所示
{% asset_image zknamespace.jpg %}
### Client API
将复杂的一致性过程用简单的API进行包装，方便使用

## Service overview
下面来介绍一下整体的服务：
- znode作为数据对象，而客户端可以通过提供的API进行操纵
- `wathces`作为一次性的触发器，在操作中绑定，用于数据更新的提醒
- 提供API对`data object`进行操纵


### Hierarchical data object
`data object`具有层次化的组织结构，像UNIX文件的文件组织结构。若要访问数据对象，输入该对象的绝对地址
当建立新的znode，可以设置`sequential flag`，使得znode命名中的数字是单调递增的
数据对象可以分成 **Regular**和**Ephemeral**两种：
#### Regular
客户端可以简单的创建并删除的对象
#### Ephemeral
当客户端连接到ZK后可以创建临时的对象，并让ZK自动的删除（session 结束后）
### Watches callback
`watches`可以让客户端及时的接收某个数据的改变，可以在每次读取等操作时设置，仅仅只更新一次
`getData("/foo",true)`
当`/foo`发生改变之前，会通过`watch`通知client
### Session
- client在连接到zk时确立一个session，当发出请求时获得一个session handle
- session设置了一个定时器，一段时间没有接收到信息，ZK认为client出错了
- watch 状态的变化


### Client API
1. `create(path,data,flags)`创建data object，flags ---> regular, ephemeral, set the sequential flag
2. `delete(path,version)`当版本号相对应后，删除数据对象
3. `exists(path,watch)` 允许设置`watch`，当数据对象不存在之前会notify客户端
4. `getData(path,watch)` 返回相应的data、meta-data，在change之前notify客户端
5. `setData(path,data,version)`将data[]写入相应的path中
6. `getChildren(path,watch)`返回znode的全部的children名字
7. `sync(path)`等待所有的更新操作的结束，使得read到的数据是最新的。


上述API均有同步和异步两个版本，异步调用，zk client保证按照操作被调用的顺序执行回调；
所有更新操作均有版本号，可决定是否做版本检查

## System guarantees
整个系统通过下面两个条件保证整个系统的协调性，下面有一个case以选举的方式具体说明整个过程：
当选举出一个新的leader时，这个leader会改变系统的设置，并通知各个节点，需要保证一下两点：
1. 当leader开始改变某些设置的时候，我们不希望任何process开始使用这些正在改变的设置
2. 当新的leader在设置完全更新完之前宕机后，我们不希望任何的process去使用这不完整的设置

总之，就是希望其他的process在configuration完全设置完后再使用
leader可以指定一个地址作为ready zone，当configuration设置完成后，其他的process去读取，完整自己的更新。
这样一个异步的方式，比起leader去更新每一个znode快的多
### Linearizable writes
所有更新操作都是序列化的。由于ZK强调高性能，因此我们这的linerizability是异步的方式，允许client拥有多个操作，并保证每一个client是以FIFO的方式执行的
### FIFO client order
所有的操作都是按照clent发送的顺序的顺序去执行的。
## Example of primitives
下面是利用ZK的API进行的一些case的设计，帮助更好的实现系统的一致性
### Configuration Management
如上面所说的选举完新的leader后需要对configuration有很多新的操作，因此会指定一个znode，所有的server异步的对这个地址进行读取，完成自身的configuration的更新操作。
利用watch进行监视，以便及时的收取到更新configuration的notification
### Rendezbous（集结地）
当client创建master和client程序时，由于是决策器完成的，client并不知道worker和master的地址和端口，因此无法让worker连接master；
因此,client可以创建一个Zr并将它的地址传递给指定的master和所有的worker。
当worker启动时，read Zr,并设置watch=true，当master写入address和port之后，notify worker
### Group Membership
设置一个`eephemeral Node`Zg代表整个group，并设置`sequential flag`保持唯一性
当创建一个process，则在Zg下创建一个znode，process fail，znode自动移除。
### Simple Lock without Herd Effect
ZK可以创建简单的锁服务，原理是获得锁，就创建一个znode，释放锁就删除相对应的znode。
而znode中的命名的大小是作为请求获得锁的process的顺序，从小到大
为了避免**herd effect**,释放锁后仅仅唤醒一个等待的线程
```
//lock()
n = create(l+"/lock-",EPHEMERAL|SEQUENTIAL)
C = getChildren(l,false)
if n is lowest znode in C,exit
p = znode in C ordered just before n
if exist(p,true) wait for watch event
goto
```
```
//unlock()
delete(n)
```
简单的来说就是，创建在parent l 下创建一个n，如果n是这些锁中最小的一个，则获得锁并退出
如果不是，则在n前面一个锁p设置exsist的watch，当不存在时，通知当前线程，跳转到创建并获得锁的步骤
删除锁则直接将该znode删除即可

## 与其他的系统的对比

### ZK vs Chubby
chubby也有一个类似于文件系统的接口，它也使用一种特定的协议保证副本之间的一致性；however，zk不是一种锁服务，它可以用于实现锁，但是在API中没有锁操作。zk允许客户端连接任意一个zk server，不只是leader，zk client可以使用local replica来服务数据以及管理watch，因为它的一致性模型比chubby更放松；


