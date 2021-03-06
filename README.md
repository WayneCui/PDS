

# 分布式系统模式

## 关于



## 问题与解决套路

当数据存储在多台服务器上的时候，很容易出现以下几种问题。

### 进程崩溃
硬件和软件错误都可能导致进程死掉。下面是几种死掉的方式：
* 可能死于系统管理员的日常维护操作；
* 可能死于IO操作，比如硬盘满了，而没有处理异常；
* 在云环境下，一些不相干的事件也有可能导致服务器宕掉。

底线是，如果该进程负责存储数据，那么需要对存储的数据提供持久化保证。
即便是进程突然崩溃，对于已经通知用户成功保存的数据，需要真正能够保存下来。
由于访问模式的不同，不同的存储引擎采用了不同的存储结构，从简单的散列表，到复杂的图结构，不一而足。由于将数据刷到硬盘是非常耗时的，并非每一次插入或更新操作都会刷盘。因此多数数据库应用会包含一个内存存储结构，该结构中的数据定期会刷到硬盘。这样进程突然崩溃时就存在丢失数据的风险。

写前日志通常用于解决此类问题。服务器将每一次状态变更以命令的形式写入一个持久化的只追加(append-only)文件。文件追加通常来说是比较快的操作，因此对性能影响不大。每一次状态更新依次保存到一个日志文件中。服务器在启动的时候，可以重放此日志文件中的操作来构建内存状态。

这样就为持久化提供了保障。服务器突然宕机和重启，数据也不会丢失。但是在服务器重新提供服务之前，客户端是不能读取或存储任何数据的。我们还缺失抵御服务器失败的能力。

一个显然的方案是将数据存储到到台服务器上。我们可以把写前日志（Write-Ahead Log）复制到多台服务器上。

然而，当存在多台服务器的时候，又有很多其他可能失败的场景需要考虑。

### 网络延迟

在 TCP/IP 协议栈中，网络传输的延迟没有上限。网络负载不同，延迟也不一样。例如，1G带宽的网路可能会由于同时触发的需要传输大量数据的任务而造成缓冲区被打满，可能会造成任意长度的消息延迟。


一个典型的数据中心，会将服务器层叠堆在机架上，多个机架通过一个交换机相连。不同部分之间的连接可能会形成一个树状结构的层级关系。在某些情况下，有可能一部分服务器之间可以相互通信，但是与另外一部分服务器是断开的。这种情况称作网络分隔。一组服务器通过网络进行通信，一个根本问题是，如果确定某台服务器宕机了。

这里需要解决两个问题。
* 服务器不会无限等待来确定另外一台服务器是否宕机；
* 不应该出现两组服务器，彼此之间认为对方挂掉了，各自服务于不同的客户端。这种情况称为“脑裂”。

为解决第一个问题，每台服务器需要定时给其他服务器发送[心跳]()信息。如果没有收到心跳信息，则认为（应该）发送方挂掉了。心跳的时间间隔需要足够短，这样不会等很久才发现某台服务器宕掉。后面我们会看到，在最坏的情况下，服务器依然存活，但是集群作为一个整体认为该服务器已经宕机。这样的设计是为了确保为客户端提供的服务不受影响。

要解决的第二个问题是脑裂。如果发生了脑裂，两组服务器分别独自接收数据更新请求，不同的客户端会存取到不同的数据。一旦脑裂问题解决，数据之间的冲突是无法自动解决的。

为了解决脑裂问题，必须保证相互断开的两组服务器之间无法各自独立取得进展。为了确保这一点，服务器执行的每一个操作，只有取得大多数服务器的确认之后才能任务是成功的。如果一组服务器无法组成大多数，那么它们是不能对外提供服务的，一部分客户端会无法得到服务，但是服务器集群会一直处于一致的状态。服务器集群的大多数的数量被称作法定数量(Quorum)。如何决定法定数量(Quorum)呢？它取决于集群能够容忍的失败的服务器的数量。假如集群由5个节点组成，那么它的法定数量(Quorum)就是3。一般来说，如果容忍 f 个节点失败，集群的数量需要设置为 2f + 1。

法定数量(Quorum)能够确保我们拥有足够多的副本来抵御部分服务器宕机。然而用于为客户端提供强一致性的保证还是不够的。举个例子，一个客户端向Quorum 发起了一个写请求，但是该写操作仅在一台机器上执行成功，其余服务器上存储的依然是旧值。当客户端从Quorum中读取数据的时候，有可能获取到最新值，假如拥有最新值的服务器依然存活的话。但是当客户端发起读请求而此机器不能提供服务的时候，大概率会读到过期值。为避免此种情况发生，需要追踪是否整个Quorum对特定的操作达成了一致，仅且仅当Quorum中的所有服务器都存储了该值之后才会通知客户端操作成功。[主从模式]()可以应用于此场景下。一台服务器被选举为领导者，其他服务器则作为追随者。领导者控制和协调向追随者的复制操作。领导者决定哪些变更对客户端可见。[高水位]()用来追踪写前日志中已经成功复制到法定数量的追随者上的条目。高水位以上的条目都是对客户端可见的。领导者还会向追随者传播高水位线。一旦领导者无法提供服务，追随者之一可以被选举为新的领导者，而对客户端来说，不存在不一致的数据。


### 进程暂停
即便有了法定团体、领导者、追随者这些概念，依然存在一个有待解决的难题。那就是领导者进行时可以突然暂停的。暂停的原因有很多。对于具有自动垃圾回收机制的语言来说，会存在垃圾回收时间过长导致系统长时间失去响应。这种情况下，领导者会与其他追随者断开，同时也会在暂停结束后中继续向追随者发送消息。


### 不同步的时钟与有序事件
区分旧leader与新leader的问题可以归结为保持消息有序的问题。第一反应是可以使用系统时钟来对消息进行排序，但是这样做是有问题的。最主要的原因在于不同服务器上的系统时钟不保证是同步的。计算机上的物理时钟是靠石英晶体振荡器的震荡来计时的。

由于晶体的震动会或快或慢，导致不同的机器上的时间不一致，这种机制是很容易出错的。不同机器上的时钟可以通过NTP服务来同步。该服务按照一定的时间间隔来检测一组全局时钟服务器，来校准机器上的时钟。

因为上述操作是基于网络通信的，并且网络延迟会差异很大，时钟同步操作可能会由于网络问题而被延迟。这样会导致不同机器的时钟相互偏离，还有可能在经过NTP服务校准后发生倒流现象。因为物理时钟存在上述问题，通常不会使用物理时钟来对事件排序，而是采用一种称作[Lamport Clock]()的技术。[Generation Clock]()即时其中一例。Lamport Clock本质上就是数字，当且仅当特定事件发生时才会递增。例如在数据库领域，读和写都是事件，但是只有写事件发生时Lamport Clock才会递增。 Lamport Clock 数值还会通过消息体发送到其他进程中。接收进程会从接收到的消息与本地维护的副本之间选择一个较大值。通过上述机制，Lamport Clock 还被用来追踪彼此通信的进程间事件之间的happend before 关系。一个例子是当服务器参与到事务中时。Lamport Clock 可以用来对事件排序，但是与物理时钟没有联系。[Hybid Clock]() 填补了这项空白。Hybid Clock包含系统时间和额外一个保证单调递增的变量，可以当做 Lamport Clock 使用。


Lamport Clock 可以用来确定跨系统事件的顺序。但是对于多个副本并发更新同一个值的情形就无能为力了。这就是就轮到[Version Vector]()上场了，它可以用来检测多个副本之间的冲突。


Lamport Clock 或 Version Vector 与被存储的值密切关联，用来检测值之间的存储顺序，或是否存在冲突。采用这种方式存储的值称为带有版本号的值（[Versioned Value]())。

## 整合 - 模式序列

接下来我们会看到理解了这些模式，如何有助于我们从下往上构建一个完整的系统。以一致性协议的实现为例。分布式一致性协议是分布式系统实现的一个特例，用来提供最强级别的一致性保证。常见的企业级系统包括[Zookeeper](), [etcd](), [Consul]()。它们通过一致性算法（如[zab]()和[Raft]())来保证副本之间的强一致性。除此之外，还有其他一些一致性算法，比如谷歌的[Chubby]()采用的是[Paxos]()算法来提供分布式锁服务、view stamp replication 和  virtual-synchrony。简单来说，一致性协议指的是一组服务器就存储的数据、存储数据的顺序以及数据何时对客户端可见达成一致。

### 实现一致性协议的模式次序
一致性算法使用[状态机复制]()来实现容错。在状态机复制机制中，存储设备如键值存储，会将存储内容复制到集群中的所有机器上，同时客户端输入的指令会以相同的顺序在每台机器上重放。其中用到的关键技术是将[写前日志]()拷贝到所有机器上，因此又称为“复制式写前日志”(Replicate Wal)。

下面我们通过各种模式的组合来实现复制式写前日志。

持久化保证使用的是写前日志模式。写前日志采用[分段日志]()模式来实现。这有助于日志的清理，而后者是基于[低水位]()模式的。容错机制靠的是将写前日志复制到多台服务器上。复制过程则用的是[领导者与追随者模式]()，[法定代表团]()则用来更新[高水位]()来控制哪些值对客户端可见。所有的请求都严格按照一定的次序执行，这里用到了[Singular Update Queue]()。每个请求从领导者发送到追随者，靠的是[Single Socket Channel]模式来保证顺序。在单通道场景下，为了提示吞吐量和降低延迟，需用到[请求管道]()。追随者通过接收领导者的[心跳]()来感知其存在。假如由于网络分隔造成领导者暂时性地与集群断开，可以通过[Generation Clock]来检测到。如果所有的请求都只发往领导者，它可能会过载。在客户端只读，且允许读到过期值的情境下，读请求可以通过追随者服务器来提供，这就是所谓的[追随者读]()模式。

### K8S 与 Kafka 控制面板

K8S或 Kafka 的架构是围绕一个强一致性的元数据库构建的。我们可以通过模式序列来理解它们的架构。[一致性核心]()用来提供强一致性保证。[租约]()用来实现集群节点之间的组员及失败检测。集群节点使用[状态观察者]()模式来获取组员失败或元数据更新等的变更通知。[一致性核心]()通过[幂等接收者]()来忽略重复的请求，这些请求可能是由于节点失败或超时造成的。

### 逻辑时钟的使用

逻辑时钟的
不同的产品用到的组织成员及其他成员的失败检测技术不尽相同。有的采用的[]()，有的采用[]()。存储数据时用到的[Versioned Value]()，用来决定哪些值是最新写入的。如果只有单一一台服务器，或者采用的[领导者与追随者模式]()，此时[兰伯特时钟]()可以用来实现版本号，[Versioned Value]。当时间戳需要从一天时间当中的时间节点，那么需要提供[Hybrid Clock]。如果多台服务器都被允许处理和保存，[Version Vector]()应运而生。

如此，理解问题域与解空间的基本形式，有助于我们理解一个完整系统的各个组成部分。

## 下一步

分布式系统是一个大话题。本章涉及的模式仅仅是其中的一小部分，通过不同的类别来展示模式是如何帮助我们理解和设计分布式系统的。我会持续扩充不同的模式，包括一下在不同分布式系统中已解决的问题类别：
* 组成员关系和失败检测
* 分区
* 复制与一致性
* 存储
* Processing