# 低水位标记

写前日志(write ahead log) 中标记哪部分日志可以丢弃的索引项。

## 问题
写前日志 (write ahead log) 将每一次数据变更都进行了持久化。随着时间的推移，数量会无限增长。分段日志(Segmented Log)允许一次只处理一个较小的日志文件，但是如果不做处理的话，整体占用的磁盘量还是会无限增长。

## 解决方案
创建一种机制，日志系统可以判断哪部分日志可以安全丢弃。该机制提供了一个最低偏移量，或低水位标记，在此之前的日志是可以安全丢弃的。可以使用一个独立的后台线程，持续地检测哪部分日志可以丢弃，并删除对应的日志文件。

```
this.logCleaner = newLogCleaner(config);
this.logCleaner.startup();
```

日志清理器可以使用定时任务来实现：
```
public void startup() {
    scheduleLogCleaning();
}

private void scheduleLogCleaning() {
    singleThreadedExecutor.schedule(() -> {
        cleanLogs();
    }, config.getCleanTaskIntervalMs(), TimeUnit.MILLISECONDS);
}

```

## 基于快照的低水位标记
大多数一致性协议的实现都实现了某种快照机制，如Zookeeper、etcd（通过 RAFT协议）。在此类实现中，存储引擎定期生成快照。随快照一起存储的还包括已经存入快照的日志的索引值。[写前日志](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)模式采用的是简单的键值存储格式，快照可以使用如下方法来获取：
```
public SnapShot takeSnapshot() {
    Long snapShotTakenAtLogIndex = wal.getLastLogEntryId();
    return new SnapShot(serializeState(kv), snapShotTakenAtLogIndex);
}
```

一旦一份快照成功落盘，日志管理器跟进接收到的低水位标志来丢弃旧的日志：
```
List<WALSegment> getSegmentsBefore(Long snapshotIndex) {
    List<WALSegment> markedForDeletion = new ArrayList<>();
    List<WALSegment> sortedSavedSegments = wal.sortedSavedSegments;
    for (WALSegment sortedSavedSegment : sortedSavedSegments) {
        if (sortedSavedSegment.getLastLogEntryId() < snapshotIndex) {
            markedForDeletion.add(sortedSavedSegment);
        }
    }
    return markedForDeletion;
}
```

## 基于时间的低水位标记
在一些系统中，日志并不是用来更新系统状态的，日志可以在给定的时间窗口之后丢弃，而无需等待任何子系统来共享低水位标志。例如，在Kafka中，日志会保存7周。所有的包含超过7周的日志的日志段都会被丢弃。在这种实现方式中，每一个日志条目都会包含一个创建时的时间戳。日志清理器会检查每一个日志段的最后一个条目，丢弃比配置的时间窗口更早的日志段。
```
private List<WALSegment> getSegmentsPast(Long logMaxDurationMs) {
    long now = System.currentTimeMillis();
    List<WALSegment> markedForDeletion = new ArrayList<>();
    List<WALSegment> sortedSavedSegments = wal.sortedSavedSegments;
    for (WALSegment sortedSavedSegment : sortedSavedSegments) {
        if (timeElaspedSince(now, sortedSavedSegment.getLastLogEntryTimestamp()) > logMaxDurationMs) {
            markedForDeletion.add(sortedSavedSegment);
        }
    }
    return markedForDeletion;
}

private long timeElaspedSince(long now, long lastLogEntryTimestamp) {
    return now - lastLogEntryTimestamp;
}

```

## 示例
* 所有一致性算法（如Zookeeper和RAFT）的日志模块的实现采用的是基于快照的日志清理策略
* Kafka 的存储模块采用的是基于时间的日志清理策略
