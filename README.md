# 实战大数据

标签（空格分隔）： 大数据 ZooKeeper Storm Kafka Hadoop Spark

[TOC]

---

## ZooKeeper

### 简介
ZooKeeper是一个**分布式的**、用来管理大量主机的**协调服务**。
在分布式环境中协调和管理一个服务是很复杂的工作，而ZooKeeper用简单的架构和API解决了这个问题，它屏蔽了分布式环境中的复杂性，让开发人员可以专注于核心应用功能的开发。

### 分布式应用
分布式应用运行在其上的一组系统被称为『集群』(cluster)，运行在集群中的每个机器被称为『节点』(node)。

#### 优点

- 可靠性 单机或少量机器的故障不会导致整个系统不可用。
- 可扩展性 不用停机只需要做很少的配置就可以根据需求通过增加机器来提升系统的性能。
- 透明性 隐藏了系统的复杂性，对外值暴露单一的入口/应用。

#### 难点

- 竞争条件 两个或多个机器都尝试去执行同一个任务，而该任务在任意时刻都应该只被一台机器执行。比如，共享资源在某一时刻只应该被一台机器修改。
- 死锁 两个或多个操作无限期的相互等待对方完成。
- 不一致性 数据的部分错误。

### ZooKeeper提供的服务

- 名字服务 在一个集群内根据name找到主机，类似DNS服务
- 配置管理
- 集群管理 
- 主节点选举
- 加锁和同步服务
- 存放高可用数据

fail-safe同步机制解决了竞争和死锁的问题。
atomicity解决了数据的一致性问题。

### ZooKeeper的优点

- 简单的分布式协调过程
- 同步 服务器进程间的互斥和协作
- 有序的消息
- 序列化 用指定的规则编码数据。保证你的应用一致的运行。这种方式可以用在MapReduce中来协调对来执行线程
- 可靠性
- 原子性 数据传输要么成功要么失败，不存在中间状态

### ZooKeeper的架构

![ZooKeeper架构图](http://ww3.sinaimg.cn/large/006y8mN6jw1f7alv0grqej30fw04zdgg.jpg)

- Client Client定时向Server发送消息通知Server该Client是Alive，同时Server会返回Response给Client，如果Client发送Message后没有收到Response，则会自动重定向到其他Server。
- Server 套装中的一个节点，提供给Clients所有的服务。
- Ensemble 套装，一组，最少3个
- Leader 节点故障时执行自动恢复的节点，启动时选举出的。
- Follower 根据Leader的指示执行任务

### 层级的命名空间

![ZooKeeper的层级的命名空间](http://ww4.sinaimg.cn/large/006y8mN6jw1f7ayltmo57j30go0ce0t8.jpg)

层级结构中的每个节点叫做znode, 每个znode维护一个`stat`结构。这个`stat`仅仅提供一个znode的元信息，其中包括版本号(Version number)、行为控制列表(Action Control List, ACL)、时间戳(Timestamp)、数据长度(Data Lenght)。

### znode的类型
znode分为三种类型：

- 永久型 永久型节点当客户端断开连接之后仍然存在，默认情况下创建的节点都是永久型节点
- 临时型 只有client保持alive时才存在的节点叫临时节点，当client从ZooKeeper集群断开时，节点被自动删除。所以临时节点不允许有子节点。如果一个临时节点被删除了，下一个合适的节点会填充它的位置。临时节点在Leader的选取中起到重要作用。
- 顺序型 序列型节点可以是永久的也可以是临时的。当一个znode被创建为顺序型时，ZooKeeper在它原来的name后面加上十位的十进制数字。如果两个顺序型节点是并发创建的，ZooKeeper会保证两个节点的name不同。顺序型节点在锁和同步中起到重要作用。

### 会话（Sessions）
一个会话中的请求是按照FIFO的顺序执行的。当一个client连接上一个server，一个会话就创建成功了并且会生成一个session id给client。  
client会按一个时间间隔给server发送heartbeat来保证session的有效性。如果在一个session的生命周期内没有收到client的heartbeat，它就会认为这个client已经死掉了。  
Session超时通常用ms表示。当一个session不管由于什么原因结束时，在session中创建的临时节点都会被删除掉。


### 手表（看门人？）Watches
Watches是用来保证client能在znode上的数据发生变化时收到通知的一种简单机制。client在读取znode的数据时可以设置一个watches给一个特定的znode，当这个znode上的数据或者它的子节点发生变化时都会触发watches给client发送通知。  
watches只会被触发一次，如果client还需要通知，那就需要另外一次的读取操作了。当一个client和server之间的会话过期时，它们之间的连接就断开了，同时watches也会被移除。  

### 工作流程

- client读取数据 client发送一个**读取请求**给ZooKeeper的一个节点，该节点根据请求中的path信息读取**自己数据库中的数据**返回znode的信息给client。所以读取操作在ZooKeeper集群中是很快的。
- client写数据 如果收到请求的是Follower，它会先把请求转发给Leader，由Leader再发送写请求给Followers。只有**大多数节点**正确响应时，写请求才会成功并且返回正确的返回码给client。否则写请求就会失败。这个严格的**大多数节点**被称为**Quorum（法定人数)**。

### ZooKeeper中的节点

- 当只有一个节点时，没有大多数
- 只有两个节点，一个出故障时，也没有大多数
- 当有三个节点，有一个出了故障，那2个就是大多数
- 当有四个节点，2个出故障了，那也是没有大多数的

所以，ZooKeeper集群的中节点的数量不要太多，不然写的性能会有下降。同时节点的数量是3/5/7这种基数，而不要是偶数。

### 小结
看了上面的这么一大套理论，可能还是对ZooKeeper做的事情云里雾里，因为它做的事情太抽象了，好像实际它什么都没做，但又发现好像每个组件比如Kafka/Storm都要和ZooKeeper配合才能用。到底为什么呢？

上面讲过，ZooKeeper其实是一个配置分发服务，也就是具体的应用如Kafka和Storm都是**无状态**的，它本身为了保持**容错**的特性，而容错很重要的一项特性就是应用Down掉之后重启还要能从之前结束时的地方继续。既然是无状态，其实是**自己不存储状态**，那要实现的这个特性肯定是**需要知道**应用Down掉之前的状态的，那么好，我就把状态存在ZooKeeper里。

举个生动的例子，假设有一个很长的（水）槽，Kafka会每秒把一个玻璃球放在槽里，这样的结果就是最先放进去的玻璃球在最前面。而Storm就是**计数工**，（注意**不是搬运工**，因为它数了之后并不会真实的改变玻璃球的位置）极端一点这个游戏是在一个战场上，Storm随时会死掉，它怎么保证它的后来者来到之后马上知道它之前数到哪个位置了？自己当然是不可靠的，因为它死掉之后这个信息就丢失了，所以它**每数一个**就朝ZooKeeper大喊一声（发送写请求）告诉它数到哪个位置了，而Storm又是个健忘症（无状态），刚数完的自己就忘了，更不用说后面来的人了，那么当它把位置信息告诉ZooKeeper之后其实它和自己的后来者就没有区别了，因为不管是谁，在计数之前都需要先去ZooKeeper读取一下前面数到的位置。这样的好处就是每个Storm随时都可以死掉，只要能有新的应用随时可以起来即可。那么存到了ZooKeeper就万无一失了么？考虑前面ZooKeeper处理写请求的特点，它是把相同的信息在集群中所有的机器上都写了一份，即使其中的一台或几台宕掉了，除非在这几台重启之前仅剩的一台也宕掉了，服务是不受影响的。如果全宕掉了，那真的没办法了，你把整个机房的电源拔掉，肯定会丢数据的。

也可以理解为把状态和应用做的解耦。

那么问题来了，为什么是在战场上？为什么好好的一个应用会无缘无故的Down掉呢？这就要从分布式应用的特点说起了。我们知道以前的所谓大型机、小型机都是很大很昂贵的特殊机器，是区别于普通的硬件的，包括CPU、内存、硬盘都是特制的，所以一台机器上百万甚至千万的都很常见，这种机器宕机的几率很小，但如果宕机的话影响也会相当严重。所以可以说这些机器的费用里其实也包含了保险费，因为机器宕机导致的损失，供应商是要负责任的。

但现在不同了，现在是用大量普通（廉价）的机器组成集群来替代之前特殊的机器，既然是普通，那出错的几率当然就更高了，这也就是为什么Storm在设计之初就表示**无状态**了。

### 安装和配置

#### 安装
大数据的这套东西安装起来都是很简单，因为都是编译好的包，直接解压之后就可以以默认配置执行了。不过ZooKeeper有点特殊，因为它需要读取的配置文件是`conf/zoo.cfg`，而默认的发行包里面是有个`conf/zoo_sample.cfg`，不过好在只需要重命名一下即可。

```
[root@localhost zookeeper-3.4.8]# cp conf/zoo_sample.cfg conf/zoo.cfg
[root@localhost zookeeper-3.4.8]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /root/packages/zookeeper-3.4.8/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
[root@localhost zookeeper-3.4.8]# bin/zkServer.sh status
Mode: standalone
```

> 注意这里的Mode，表示单点模式，区别于集群模式


#### 配置
前面只讲了基础配置，这样的配置是没法跑集群环境的，下面先从默认配置出发，一步一步搭建一个集群环境。
先贴默认配置：

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
#maxClientCnxns=60
#autopurge.snapRetainCount=3
#autopurge.purgeInterval=1
```

1. `tickTime`： ZooKeeper服务器或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime就会发送一条心跳
2. `dataDir` 顾名思义就是ZooKeeper保存数据的目录
3. `clientPort` ZooKeeper对外提供服务的端口，即客户端通过该端口与ZooKeeper通信
4. `initLimit` ZooKeeper集群中的Leader忍受Follower多少个心跳间隔不发送心跳。从这里的默认配置推算，10个心跳间隔，每个心跳间隔2秒钟，也就是当Leader经过2*10秒还收不到Follower的信条时就认为这个Follower已经挂了
5. `syncLimit` Leader和Follower之间发送消息时，请求和应答的时间长度，默认5，及10秒

从上面的描述就可以看到，从第4条开始就是集群需要的配置了，然而仅仅在每个机器上这样配置并不能变成一个集群，还需要一个重要的配置，形如

```
server.1=c1:2888:3888
server.2=c2:2888:3888
server.3=c3:2888:3888
```

其中的`server.n`中的n表示节点的编号，那么问题来了，编号从哪里定义呢？我觉得这个设计其实不太好，当然我也想不到更好的方式来解决这个问题了。我们还需要在`dataDir`中写入一个名为`myid`的文件，其中填写当前机器的编号，操作如下

```
cd /path/to/dataDir
echo 1 > myid
```

c1的位置是节点机器的hostname或者IP地址，这样写当然还是不行的，因为它们并不知道c1是什么鬼，所以还需要修改`/etc/hosts`，并没有什么黑科技。以我当前的本地集群为例，在该文件中添加

```
192.168.1.111 c1
192.168.1.110 c2
192.168.1.112 c3
```

2888是默认的Follower与Leader交换信息的端口，3888是用于选举Leader的端口，当Leader挂了，当然需要选举一个新的Leader来继续它未竟的事业了。

#### 启动集群

这时可以在三台机器上同时执行`bin/zkServer.sh start`了。如果看到和前面一样的结果（注意把刚才已经启动的服务先关掉，执行`bin/zkServer.sh stop`），恭喜你成功了一半了。
这时再执行`bin/zkServer.sh status`你会惊奇的发现其中一台会显示

```
[root@localhost zookeeper-3.4.8]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /root/packages/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: leader
```

另外两台显示

```
[root@localhost zookeeper-3.4.8]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /root/packages/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: follower
```

为了证明Leader节点是自动选举的，可以把Leader手动关掉，再分别看看另外两台的`status`。是不是一个变成了Leader了？

> 几天前写这篇文章时没有问题，但今天继续编写Kafka部分时，因为之前重启过这几台虚拟机，IP变了，重启所有虚拟机之后发现虽然已经在本地启动了ZooKeeper服务，但执行`zkServer.sh status`时总是提示没有运行，而如果你再执行`start`指令，它又会提示说进程已经在运行了。根据[StackOverFlow的答案](http://stackoverflow.com/questions/29909191/zookeeper-it-is-probably-not-running)，把`/root/packages/zookeeper-3.4.8/bin`添加至`$PATH`中即可。
>
> ```bash
> $ vim /etc/profile.d/zookeeper.sh
> export ZK_HOME=/root/packages/zookeeper-3.4.8
> export PATH=$PATH:$ZK_HOME/bin
> ```
>
> 之后执行`source /etc/profile.d/zookeeper.sh`即可直接在系统的任何地方执行`zkServer.sh start`了。

通过动态的选举Leader节点，就解决了**主从系统的单点故障问题**。

### API
（略)
### Java Binding
（略)
### Scala Binding
（略)

本文并未覆盖ZooKeeper的API，因为作为写应用程序的我们来说，这对我们并不重要，我们并不直接和ZooKeeper的API打交道。

## Kafka
### 简介
在大数据的处理中，有两个重要的问题需要解决：如何收集还海量数据和如何分析收集到的数据。为了解决这些问题，就需要一个**消息系统**。Kafka就是为高吞吐量的分布式系统设计的。相比于其他的消息系统，Kafka有几个优点，更高的吞吐量，内置的分片和赋值还有与生俱来的容错能力，这让它很适合作大规模的消息处理应用。

### 消息系统
消息系统负责从一个应用向别的应用传递数据，所以应用可以专注于数据，而不用关心如何共享它们。分布式的消息是基于可信消息队列的概念。消息在客户端应用和消息系统之间异步的排队。有两种消息模式，一种是点对点模式，一种是发布-订阅模式。大多数的消息类型是发布-订阅模式。

#### 点对点模式
在一个点对点的系统中，消息持久化在一个队列中。一个或多个消费者可以来消费队列里的消息，但**同一个消息最多只能被消费一次**。当一个消费者从队列中读取了一条消息，这条消息就从队列中消失了。典型的应用是订单处理系统。

#### 发布订阅模式
在发布订阅模式中，消息是持久化在一个**主题(topic)**中。不像点对点的系统，消费者可以订阅一个或多个主题，并且消费主题中所有的消息。在发布订阅系统中，消息生产者被称为**发布者(publisher)**，消息消费者被称为**订阅者(subscriber)**。典型的应用是电视。

### Kafka是什么？
Kafka是一个分布式的、发布-订阅模式的消息系统，同时也是一个可以处理海量数据的健壮的队列，它可以把消息从一个终端传送到另一个终端。Kafka适合做离线或在线的消息消费。Kafka的消息是持久化在磁盘上的，并且在集群内保存副本以保证数据不会丢失。它是建立在ZooKeeper同步服务至上的，它可以和Apache Storm和Spark很好的结合起来做实时的流数据分析。

#### 优点
- 可靠性
- 可扩展性
- 持久性
- 高性能

### 架构图
![Kafka架构图](http://ww2.sinaimg.cn/large/65e4f1e6jw1f7j3p4oxf3j20go08fjsd.jpg)

如上图所示，一个topic被分为3个分片，其中分片1有2个偏移因子0和1，分片2有4个偏移因子0, 1, 2, 3，分片3有1个偏移因子0。副本的id和它所在服务器的ID一致。
假设该topic的复制因子为3，Kafka就会为该主题创建3份相同的副本并把它们保存在集群中用于它的所有操作。为了平衡集群的负载，每个broker存储一个或多个分片。多个producers和多个consumers可以同时发布和获取消息。

### 概念解释
1.  topics
        属于一个分类的消息流称为一个topic，数据存在topics。
        topics被分成分片。对于每个topic，Kafka会保证每个分片的最小化。每个这种分片包含一个不可变序的序列。一个分片的实现就是一个具有相同大小的片段的集合。
2.  partitions
          topics可以有多个分片，所以它可以处理任意数量的数据。
3.  分片偏移量
          每个分片的ID有一个唯一的id叫做偏移量
4.  分片的副本
          副本就是分片的备份。副本既不是用来读的也不是用来写的，而仅仅是为了保证数据不丢失。
5.  brokers
    - brokers是维护已经发布的数据的简单系统。每个broker可以有一个topic的0个或多个分片。假设一个topic中有N个分片，有N个broker，每个broker就会有1个分片
    - 假设N个分片，并且有N+m个broker，这N个分片会分布在前N个broker中，而后面m个broker不会有该topic的任何分片
    - 假设N个分片，并且有N-m个broker，每个broker可能有1一个或多个分片。不推荐这种使用方式，因为broker的负载会失衡
6.  Kafka集群
          有多个broker的Kafka就叫Kafka集群。一个Kafka集群可以无需停机的扩展。这些集群可以用来管理消息数据的持久化和复制。
7.  producers
          producers是向Kafka topic中发布消息的发布者。producers向Kafka的broker发送数据。每当一个producer发送一条消息到broker，broker就简单的把它追加到最后一个片段后面。实际上，消息就是被追加到分片的后面。producer也可以指定要发送到的分片
8.  consumers
          consumers从broker读取数据。consumers订阅到一个或多个topics并且从brokers中拉取数据
9.  Leader
          Leader是负责给定的分片的所有读写的节点。每个分片都有一个服务器作为Leader。
10.  Follower
     服从Leader的指令的节点叫做Follower。如果Leader出故障，会自动的从followers中选举出一个leader。follower像普通的consumer一样，拉取消息并在自己的数据仓库中更新数据。


### 集群架构

![Kafka集群架构](http://ww3.sinaimg.cn/large/72f96cbajw1f7k9hy3hv6j20go09cmy2.jpg)

上图描述了Kafka集群的架构，从图中可以看出：

1. producer从ZooKeeper中获取可用的broker id, 然后把消息push到broker中
2. consumer从ZooKeeper更新offset，然后从broker中拉取消息



下面是图中几个组件的职责：

1. Broker

   一个典型的Kafka集群会由多个broker组成以保持负载均衡。Kafka broker是**无状态的**，所以它们使用ZooKeeper来维护集群状态。一个Kafka broker实例每秒可以处理数十万个读写请求并且每个broker可以在对性能没有影响的情况下处理上TB的消息。Kafka broker Leader的选举是由ZooKeeper完成的

2. ZooKeeper

   ZooKeeper用来管理和协调Kafka broker。ZooKeeper服务主要用来在集群中新加入broker或有broker故障时通知producer和consumer。当ZooKeeper收到一个表示broker新加入或者故障的消息时，就会决定在当前可用的broker中分配任务。

3. Producer

   producer可以push数据给broker。当新的broker启动时，所有producer都会发现它并且自动发送消息给新的broker。Kafka producer并不会等待broker的响应，而是以broker可以处理的最快速度把消息发送过去。

4. Consumer

   因为Kafka broker是无状态的，这意味着consumer就需要通过分片的偏移量来维持已经消费了多少消息。如果consumer告知一个特定的消息偏移量，就表示它已经消费了这个偏移量之前的所有消息。Consumer会向broker发起一个异步的拉取请求来创建一些字节的缓冲区来准备消费。Consumer可以重置offset或简单的通过提供一个偏移值来调到分片上的任何一个位置。Consumer的偏移量是由ZooKeeper来通知的。



### Kafka工作流

Kafka就是**被分成一个或多个分片（partition）的topics的集合**。一个Kafka partition线性的有序消息序列，每条消息都可以用它的索引来标识（称为offset）。所有存在Kafka集群上的数据是不连贯的partition的集合。新来的消息被写在一个分片的最后，consumer按照顺序读取消息。持久性是通过把消息的副本存放在不同的broker上实现的。



#### 生产-订阅模式工作流

1. Producers每隔一定时间发送一条消息到topic
2. Kafka broker把该topic的所有消息存储在配置好的分片中。它保证所有消息是均等的分布在所有的partitions中的。如果producer发送了2条消息而存在2个partitions， Kafka会把第一条消息存放在第一个partition，第二条message存放在第二个partition。
3. Consumer订阅一个特定的topic
4. 当一个consumer订阅了一个topic，Kafka会该topic当前的offset提供给consumer，并且把这个offset存放在zookeeper集群中
5. Consumer会按一定时间间隔（例如100ms）向Kafka请求新的message
6. 当Kafka从producers接收到新的消息，它就会把它们转发给consumers
7. Consumer接收消息并处理它
8. 当一条Message处理完成之后，consumer会发送一条确认消息（ack）给Kafka broker
9. 当Kafka收到确认消息，它会修改offset并且把它更新到ZooKeeper。由于offset是在ZooKeeper中维护的，所以即使当Kafka服务出问题时也能正确的读取message
10. 上面的流程会一直持续到consumer不再向Kafka发送请求
11. Consumer可以随时倒回或跳转到topic的某一个特定的offset和读取所有消息



####  队列消息 / 消费者组(consumer group)

在一个消息队列系统中存在多于一个的consumer，它们持有相同的group id订阅了同一个topic。简单地说，就是持有相同group id并且消费同一个topic的所有consumers被看作一个组（group）并且所有的message在它们之间是共享的。

1. Producers每隔一定时间发送一条消息到topic

2. Kafka broker把该topic的所有消息存储在配置好的分片中。它保证所有消息是均等的分布在所有的partitions中的。如果producer发送了2条消息而存在2个partitions， Kafka会把第一条消息存放在第一个partition，第二条message存放在第二个partition。

3. 单个consumer订阅了一个指定的topic，例如topic是"Topic-01", group id 是 “Group-1”

4. Kafka就像普通的生产-订阅模式一样和consumer进行交互，直到新的consumer订阅了相同的topic

5. 当一个新的consumer加入，Kafka就会切换到『共享模式』并且在两个consumer之间共享数据。这种共享会一直持续到consumer的数量达到该topic的partitions的数量。

6. 当consumer的数量达到了topic的partitions的数量，这时新加入的consumer在当前有consumer取消订阅之前是收不到任何message的。发生这种情况的原因是Kafka中的每个consumer至少会被分配一个partition，当所有partitions都已经分配给当前订阅的consumer时，新加入的consumer只能等待。

   ​

> Kafka会以同样的方式保证这两种方式以一种非常简单而高效的方式运行。



#### ZooKeeper的职责

Kafka的一个非常关键的依赖因素是ZooKeeper，关于后者的相关信息可以在本文的第一部分看到。

ZooKeeper作为Kafka broker和consumer的中介接口来提供服务。Kafka服务器通过ZooKeeper集群来共享信息。Kafka把诸如topic、brokers、consumer offset等元信息存放在ZooKeeper中。

因为所有的关键信息都是存放在ZooKeeper中并且它们自然就会在ZooKeeper集群中保存副本，所以Kafka broker或者ZooKeeper服务的故障都不会影响Kafka 集群的工作。当ZooKeeper服务一旦重启，Kafka马上就能恢复之前的状态。这给了Kafka『零停机时间』的特性。kafka集群中broker的leader的选举也是通过ZooKeeper来实现的。



### 安装

Kafka提供的安装包可以让我们很容易的使用命令行来体验Kafka工作的全过程。前面说过，Kafka的broker运行是需要依赖ZooKeeper的，而Kafka的二进制包贴心的提供了一个简易的（也可能是完整的）ZooKeeper包，但我还是计划使用独立的ZooKeeper。

首先像前面那样，先把Kafka的可执行文件加入到`$PATH`中，避免不必要的麻烦。

```shell
$ vim /etc/profile.d/kafka.sh
export KAFKA_HOME=/root/packages/kafka_2.11-0.10.0.0
export PATH=$PATH:$KAFKA_HOME/bin
```

之后执行`source /etc/profile.d/kafka.sh`即可。

#### Local

1. 先看一下默认安装的Kafka和ZooKeeper给我们带来了什么见面礼

   ```shell
   [root@localhost ~]# kafka-topics.sh --zookeeper localhost:2181 --list
   Hello-Kafka
   topic-name
   ```

   这里可以写`--zookeeper localhost:2181`是因为每台机器上都运行着ZooKeeper，无非是leader还是follower的问题。

   好，看来我们已经有一些topic了，那么看看这些topic的配置信息吧

   ```powershell
   [root@localhost ~]# kafka-topics.sh --describe --zookeeper localhost:2181
   Topic:Hello-Kafka      	PartitionCount:1       	ReplicationFactor:1    	Configs:
          	Topic: Hello-Kafka     	Partition: 0   	Leader: 0      	Replicas: 0    	Isr: 0
   Topic:topic-name       	PartitionCount:1       	ReplicationFactor:1    	Configs:
          	Topic: topic-name      	Partition: 0   	Leader: 0      	Replicas: 0    	Isr: 0
   [root@localhost ~]# kafka-topics.sh --describe --zookeeper localhost:2181 --topic topic-name
   Topic:topic-name       	PartitionCount:1       	ReplicationFactor:1    	Configs:
          	Topic: topic-name      	Partition: 0   	Leader: 0      	Replicas: 0    	Isr: 0
   ```

   可以看到，如果不加`--topic`则列出所有的topic。而且需要注意到`kafka-topics.sh`系列命令在执行时都需要加上`--zookeeper zookeeperhost:port`，这么说是因为还有另外两个重要的命令需要类似的信息。

   对照前面的描述，我们来分析一下这些数字的意义

   - Topic: 当然是topic的名字了
   - PartitionCount: topic的分片的总数
   - ReplicationFactor: 复制因子，也就是副本的份数
   - 下面那行就是每个Partition的情况了，因为这里只有一个Partition所以也就只有一行。

2. 还是需要看一下topic中的message的情况

   先把broker启动起来

   ```shell
   kafka-server-start.sh -daemon packages/kafka_2.11-0.10.0.0/config/server.properties
   ```

   > 在这里启动的时候可能会出现问题，比如我们要把上面配置的三台机器上的Kafka broker(server)都启动起来，第一次可能会直接全部执行上面这条命令，但实际上因为它们是在一个集群上，所以互相之间是需要进行区分的。默认的`server.properties`里面有这样一行`broker.id=0`，其中的注释里写的很清楚，说每个broker需要有**唯一的id**，所以在启动之前需要给它们配置好不同的id。
   >
   > 如果你已经启动过一次了，然后发现有冲突，凭直觉就改了`server.properties`中的`broker.id=101`这样，会发现还是无法启动。这是因为Kafka broker启动之后会在`log.dirs`（在`server.properties`中定义）中产生`meta.properties`文件，其中包含了Kafka的版本号和当前的broker的id，因为第一次启动时的id（在这里是0）和你后面修改的(101)不一致，所以broker会拒绝启动。因为这是实验环境，所以把`/tmp/kafka-logs`中的所有内容都删了其实也无所谓。当然正确的方法是修改`/tmp/kafka-logs/meta.properties`中的`broker.id`项，使之与`server.properties`中的一致。

   通过kafka提供的console-consumer来查看指定topic中的message

   ​

#### Cluster

## Storm

storm jar ....jar com.ksyun.poc.KafkaSample mystrom
storm list 查看正在运行的topology
storm kill mystorm 杀掉正在执行的集群
### Kafka-Storm

## Hadoop
### HDFS

storm写的日志 日志路径 /mnt/log/storm，到其中查找相应的日志，可以查关键字Exception
`grep -i 'exception' *`，里面记录了很详细的函数调用堆栈。
默认情况下，storm应用写到HDFS里面的文件的根目录是`/user/storm`，注意这里是`/user`而不是`/usr`，放心吧，我没有写错。
这个目录？可能是不会自动创建的，因此需要事先手动创建

```
hadoop fs -mkdir /user/storm
hadoop fs -chown -R storm:storm /user/storm
```

集群中把hdfs-site.xml、core-site.xml打包到jar里面

[1] https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/
