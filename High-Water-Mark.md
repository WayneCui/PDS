https://martinfowler.com/articles/patterns-of-distributed-systems/high-watermark.html


# 高水位标记

写前日志(write ahead log) 中标记最新完成同步的日志项的索引。

## 问题

写前日志模式用来在服务器宕机重启后恢复系统状态。但是仅仅有写前日志是不足以保证系统的高可用的。当服务器宕机后恢复前，是无法为客户端提供服务的。为了提升系统的可用性，我们可以把写前日志复制到多台机器上。在主从模式下，领导者会把日志复制给一组法定数目的服务器。这样当领导者宕机后，新的领导者会被选举出来，客户端几乎可以无缝访问集群。但是这里依然有几个容易出错的地方：

主服务器可能在把日志同步给从服务器之前就宕掉了
主服务器宕机之前将日志发送给一部分从服务器，但是没有达到法定数量
在这些错误场景下，某些从服务器的日志中会确实部分条目，某些从服务器的日志条目会比其他的多。从服务器感知到哪一部分日志对外可见是安全的就显得尤为重要。

## 解决方案

高水位标记指代写前日志中已经成功拷贝到满足法定数量的从服务器上的最近条目的索引值。主服务器在同步日志的时候也会把高水位标记同步过去。集群中的所有服务器能且仅能向客户端传输位于低水位以下的数据。

下面是操作流程图。

对于每一条日志条目，主服务器会首先将其拷贝到自身的写前日志中，然后将其发送给所有的从服务器。

从服务器则处理复制请求，然后将其追加到本地日志文件中。在追加成功后，在响应中将它们所拥有的最新的日志条目的索引值返回给主服务器。同时，响应体还包含了本地服务器的Generation clock当前值。

主服务器在收到响应后，主服务器会追踪每台服务器已成功复制的条目索引。
```
class ReplicationModule…

  recordReplicationConfirmedFor(response.getServerId(), response.getReplicatedLogIndex());
  long logIndexAtQuorum = computeHighwaterMark(logIndexesAtAllServers(), config.numberOfServers());
  var currentHighWaterMark = replicationState.getHighWaterMark();
  if (logIndexAtQuorum > currentHighWaterMark) {
      applyLogEntries(currentHighWaterMark, logIndexAtQuorum);
      replicationState.setHighWaterMark(logIndexAtQuorum);
  }
```

高水位标记是可以通过观察从服务器和主服务器存储的日志计算得来的，之后选择一个复制已过半数的计算机的条目。

```
class ReplicationModule…

  Long computeHighwaterMark(List<Long> serverLogIndexes, int noOfServers) {
      serverLogIndexes.sort(Long::compareTo);
      return serverLogIndexes.get(noOfServers / 2);
  }
```


主服务器可以通过附加到常规的[心跳]()消息上或单独的请求两种方式将高水位标记传播给从服务器。从服务器接收到消息之后设置自身的标记。

客户端只能读到高水位以下的日志条目。超过高水位的条目则是不可见的，因为还没有被确认。在主服务器突然宕机，新的机器被选举为主服务器的情况下，会不复存在。

```
class ReplicationModule…

  public WALEntry readEntry(long index) {
      if (index > replicationState.getHighWaterMark()) {
          throw new IllegalArgumentException("Log entry not available");
      }
      return wal.readAt(index);
  }
```


## 日志截断






选主会带来一个微妙的问题。我们必须确保任何服务器在给客服端发送数据之前，集群中的全部服务器的日志都已经是最新的了。

然而在现有主服务器将高水位标记复制给任何从服务器之前失败了的情况下，会存在一个比较微妙的问题。RAFT采用的方法是在成功选主以后向主服务器的日志中追加一条空条目，并且只有在被从服务器确认后才会对外提供服务。在ZAB算法中，新选出来的主服务器会尝试在提供服务之前向所有从服务器推送它所拥有的全部日志条目。