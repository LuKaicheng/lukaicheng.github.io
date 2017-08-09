---
title: HDFS学习总结(二)高可用
date: 2017-08-09 16:35:01
tags:
- Hadoop
- HDFS
---

## 引言

在上一篇[HDFS学习总结(一)基本原理](https://lukaicheng.github.io/2017/08/02/HDFS-Basic-Notes/)的最后提到，由于名称节点容易成为整个HDFS系统的单点故障，这极大限制了hadoop系统在生产环境的使用场景。在hadoop1.x中，虽然有所谓的**Secondary NameNode**，然而对于它的定位仅仅只是**NameNode**的辅助者，严格来讲连冷备都算不上。幸运的是，在hadoop2.x版本里，经过多次迭代和发展，对于HDFS来说终于有了一套较为成熟可靠的高可用方案：**QJM**(*Quorum Journal Manager*)。本文将以此为中心，梳理和总结HDFS高可用的相关知识。

<!-- more -->

## 元数据

众所周知，名称节点管理了整个HDFS系统的元数据信息，在运行期间，元数据信息在内存中不断更新，为了防止数据丢失，这些数据操作记录会被到**EditLog**文件中，辅助节点(*Secondary NameNode*)会帮助名称节点定时将产生的**EditLog**文件合并到**FsImage**文件，其中元数据存储目录结构如下图所示：

![HDFS名称节点元数据存储目录结构](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_NN_Metadata.png)

### EditLog

当客户端提交创建文件或移动文件这种的写操作时，名称节点会首先把这些操作记录在**EditLog**文件中，然后再更新内存中的文件系统镜像。记录在**EditLog**之中的每一个操作称为一个事务，会有一个整型的事务编号，整个**EditLog**会被切割为很多段，每一段称为一个**Segment**。具体表现出来就是目录下存在多个edits前缀的文件，其中文件名形式为`edits_inprogress_${start_txid}`表示是当前正在写入的EditLog Segment，`${start_txid}`表示这个Segment的起始事务id，如果是已经写入完成的EditLog Segment，那么会处于finalized状态，其文件名形式`edits_${start_txid}_${end_txid}`，`${start_txid}`表示这个segment的起始事务id，`${end_txid}`表示这个segment的结束事务id。

通过执行命令(***hdfs oev***)，我将一个edits文件以xml保存，这样可以具体看一下内部的数据样例。可以明显看到，里面每一个**Record**表示一次事务操作，每个事务操作会包含**OPCODE**和**TXID**： 

```xml
<EDITS>
  <EDITS_VERSION>-60</EDITS_VERSION>
  <RECORD>
    <OPCODE>OP_START_LOG_SEGMENT</OPCODE>
    <DATA>
      <TXID>876470</TXID>
    </DATA>
  </RECORD>
  <RECORD>
    <OPCODE>OP_ADD</OPCODE>
    <DATA>
      <TXID>876471</TXID>
      <LENGTH>0</LENGTH>
      <INODEID>156307</INODEID>
      <PATH>/hbase/WALs/hadoop08,60020,1499320794483/hadoop08%2C60020%2C1499320794483.default.1501664647096</PATH>
      <REPLICATION>3</REPLICATION>
      <MTIME>1501664647540</MTIME>
      <ATIME>1501664647540</ATIME>
      <BLOCKSIZE>134217728</BLOCKSIZE>
      <CLIENT_NAME>DFSClient_NONMAPREDUCE_1675402001_1</CLIENT_NAME>
      <CLIENT_MACHINE>172.23.25.141</CLIENT_MACHINE>
      <OVERWRITE>false</OVERWRITE>
      <PERMISSION_STATUS>
        <USERNAME>ceshi</USERNAME>
        <GROUPNAME>supergroup</GROUPNAME>
        <MODE>420</MODE>
      </PERMISSION_STATUS>
      <RPC_CLIENTID>a7425087-ddc0-40af-8974-f62a4ae6aadf</RPC_CLIENTID>
      <RPC_CALLID>115918</RPC_CALLID>
    </DATA>
  </RECORD>
  <RECORD>
    <OPCODE>OP_ALLOCATE_BLOCK_ID</OPCODE>
    <DATA>
      <TXID>876472</TXID>
      <BLOCK_ID>1073823335</BLOCK_ID>
    </DATA>
  </RECORD>
</EDITS>  
```

### FsImage

**FsImage**记录了自最后一次检查点之前HDFS文件系统所有目录和文件的序列化信息，辅助节点会周期性地从名称节点处获取**EditLog**和**FsImage**进行合并，为名称节点生成新的**FsImage**文件，来替代旧文件。从目录结构图可以看到**FsImage**文件名称形式是`fsimage_${end_txid}`，其中`${end_txid}`表示结束事务id。在名称节点启动时，会首先将**FsImage**文件加载到内存中形成内存文件系统镜像，然后再把位于该**FsImage**文件之后(*edits文件名中的起始事务id大于fsimage文件名中的结束事务id*)的**EditLog**回放到该镜像，完成数据恢复。

同样的，通过执行命令(***hdfs oiv***)，我将一个fsimage文件以XML形式保存，下面是省略了大部分内容之后的样例：

```xml
<fsimage> 
  <inode>
    <id>17294</id>
    <type>FILE</type>
    <name>data.txt</name>
    <replication>3</replication>
    <mtime>1492744099454</mtime>
    <atime>1495524378479</atime>
    <perferredBlockSize>134217728</perferredBlockSize>
    <permission>ceshi:supergroup:rwxr-xr-x</permission>
    <blocks>
      <block>
        <id>1073742303</id>
        <genstamp>1479</genstamp>
        <numBytes>437</numBytes>
      </block> 
    </blocks> 
  </inode> 
</fsimage>
```

### Secondary NameNode

从前文知道，名称节点通过将元数据信息保存在**EditLog**和**FsImage**中得以持久化，然而由于生产环境中名称节点很少重启，那么如果没有机制来合并**EditLog**和**FsImage**，那么会导致下一次重启名称节点花费更多时间，而且大量的**EditLog**也不好管理。为了解决这个问题且不加重名称节点的负担，因此引入了所谓的**Secondary NameNode**这个辅助节点。整体处理流程如下图所示：

![HDFS辅助节点备份合并过程](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_Checkpoint.png)

整个过程也被称为一个**Checkpoint**，大致描述：

1. 辅助节点向名称节点请求合并edits文件，名称节点会创建新的edits.new
2. 辅助节点向名称节点请求fsimage文件和edits文件
3. 辅助节点把edits文件和fsimage文件合并，生成新的fsimage文件
4. 名称节点获取新的fsimage文件，并将edits.new替换原来的edits文件

## 高可用架构

目前hadoop2.x中提出的基于QJM的HDFS高可用整体架构示意如下：

![HDFS QJM高可用整体架构](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_QJM_HA.png)

主要分成以下几个部分：

- 名称节点：有Active和Standby之分，前者为主名称节点，对外提供读写服务，后者作为备用。
- 主备切换：通过ZKFailoverController检测名称节点的健康状态，在主名称节点故障时，借助ZooKeeper的主备选举支持，来实现自动的主备切换。
- 共享存储：整个高可用架构的核心部分，通过共享存储实现元数据同步，基于paxos算法。
- 数据节点：需要同时向主名称节点和备用名称节点上报数据块位置。

## 共享存储

实现HDFS高可用最为关键的部分就是共享存储系统，前文提到名称节点的元数据会持久化到**FsImage**和**EditLog**这两个文件，而基于QJM的共享存储系统主要用于保存**EditLog**(*并不保存FsImage*)。其基本思想来自于Paxos算法，采用多个称为**JournalNode**节点组成的集群来存储**EditLog**，主名称节点首先把**EditLog**提交到**JournalNode**集群，然后备用名称节点再从**JournalNode**集群定时同步**EditLog**。

![共享存储整体架构](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_JournalNode_Set.png)

上图是共享存储系统的一个整体视图，下面会针对这两个过程再展开描述。

### 提交EditLog

当主名称节点需要提交**EditLog**时，它不仅需要写入本地磁盘目录，同时还要写入到**JournalNode**集群，后者会通过RPC方式并行的请求每个**JournalNode**进行写入，如果对大多数**JournalNode**都写入成功的话，那么就认为提交**EditLog**成功，否则会视为本次提交失败，这个失败会导致目前的主名称节点停止服务退出进程，留待处于备用状态的名称节点接管进行数据恢复。

![JournalNode写入流程](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_JournalNode_Detail.png)

### 同步EditLog

处于备用状态的名称节点，会启动一个`EditLogTailer`线程，这个线程的作用就是定期调用`doTailEdits`方法从**JournalNode**集群上同步**EditLog**，然后将它回放到内存的文件系统镜像上(*不会写入本地磁盘*)。这里需要注意的是，获取过来的**EditLog**都是处于finalized状态的**Segment**。另外，由于采用的是定时同步方式，所以备用名称节点内存中的文件系统镜像有很大可能性会落后于当前的主名称节点，因此当发生主备切换时，需要先把落后的**EditLog**补上来。

![NameNode补齐EditLog](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_JournalNode_Recovery.png)

上图就描述了当备用名称节点转换成主名称节点之后，进行数据恢复的大致流程：首先为了标识本次主名称节点的生命周期需要确定一个纪元(Epoch)，接着选择出数据恢复的EditLog Segment的id，据此选定同步的基准数据源，让**JournalNode**集群上的各个节点的**EditLog**恢复一致，最终当大多数已经处于一致状态，主名称节点就会从集群上补齐落后的**EditLog**数据。

## 主备切换

共享存储可以说解决了主备节点之间的数据同步问题，那么剩下的就是实现自动的故障转移，即主备切换，这个工作主要由`ZKFailoverController`进程来实现。`ZKFailoverController `会在名称节点上以独立进程启动，启动的时候会创建`HealthMonitor `和`ActiveStandbyElector `，分别负责名称节点的健康检测和自动的主备选举。这其中大致的过程如下图所示：

![主备切换](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/hdfs/HDFS_NN_Switch.png)

1. `HealthMonitor `启动之后会定时对名称节点的健康状态进行检测。
2. `HealthMonitor`如果发现名称节点的健康状态发生变化，将会回调通知`ZKFailoverController `进行处理。
3. `ZKFailoverController`如果判定需要进行主备切换，那么会让`ActiveStandbyElector`进行自动的主备选举。
4. `ActiveStandbyElector`与Zookeeper进行交互完成自动的主备选举。
5. `ActiveStandbyElector`将会通过回调将选举结果通知`ZKFailoverController`，来决定当前名称节点成为主节点或者备节点。
6. 最终`ZKFailoverController`会调用名称节点的RPC接口将其切换成正确的状态。

其中步骤4中主备选举的机制主要是依赖于Zookeeper的写一致性和临时节点机制：如果`HealthMonitor`检测其对应的名称节点正常，那么`ActiveStandbyElector`会发起主备选举，尝试在Zookeeper创建一个临时节点(*/hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock*)，Zookeeper保证只有一个能成功，其余则会失败。成功创建的对应的名称节点会成为主节点，失败的则是备用节点，但是不管成功还是失败，`ActiveStandbyElector`都会对这个临时节点进行监听。一旦主名称节点的`HealthMonitor `检测到其状态异常，`ZKFailoverController `会主动删除该临时节点，这时候监听了此节点的`ActiveStandbyElector `就能收到通知，重新触发主备选举。

## 隔离 

在这个高可用的架构设计中，还有一个比较值得注意的点是任何时间，只允许一个名称节点处于活动状态。因此为了避免可能发生的脑裂(*split-brain*)，有必要进行隔离(*fencing*)。

由前文可知在主备选举的时候会在Zookeeper创建一个临时节点，除此之外还会创建另外一个持久化节点(*/hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb*)，如果主名称节点是正常关闭，那么会同时删除该节点，如果由于异常原因关闭，那么这个持久化节点还会存在，这时候新的选举出来的主名称节点会尝试对旧主节点进行状态转换，如果失败会执行隔离措施(*sshfence或者shellfence*)，只有隔离成功，新主才能开始对外提供服务。

另外就是在新的主名称节点进行数据恢复时，会产生新的纪元(Epoch)，**JournalNode**集群会保存新的纪元，以此为依据来判定应该允许还是拒绝接受到的写操作请求。由于新纪元肯定大于老纪元，所以原先的主名称节点如果尝试写日志操作的话，将会被拒绝。

## 参考

[Hadoop NameNode 高可用 (High Availability) 实现解析](https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/index.html)

