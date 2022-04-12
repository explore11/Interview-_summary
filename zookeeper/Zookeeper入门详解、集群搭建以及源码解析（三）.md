# 10、Paxos 算法
**Paxos算法：一种基于消息传递且具有高度容错特性的一致性算法。
Paxos算法解决的问题：就是如何快速正确的在一个分布式系统中对某个数据值达成一致，并且保证不论发生任何异常（机器宕机、网络异常），都不会破坏整个系统的一致性。**
## 10.1、Paxos算法描述
• 在一个Paxos系统中，首先将所有节点划分为Proposer（提议者），Acceptor（接受者），和Learner（学习者）。
（注意：每个节点都可以身兼数职）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/352b85e6d3884f5ba6c08c0a14f238de.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**一个完整的Paxos算法分为三个阶段：**
-  **Prepare准备阶段**
	 1. Proposer向多个Acceptor发出Propose请求Promise（承诺）
	 2. Acceptor针对收到的Propose请求进行Promise（承诺）
- **Accept接受阶段**
	 1. Proposer收到多数Acceptor承诺的Promise后，向Acceptor发出Propose请求
 	2. Acceptor针对收到的Propose请求进行Accept处理
- **Learn学习阶段**
	1. Proposer将形成的决议发送给所有Learners


## 10.2、Paxos算法流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/564e8b2c3eeb42e783e9cbef7e990181.png)
1. Prepare: Proposer生成全局唯一且递增的Proposal ID，向所有Acceptor发送Propose请求，这里无需携带提案内容，只携带Proposal ID即可。 
2. Promise: Acceptor收到Propose请求后，做出“两个承诺，一个应答”。 
	➢ 不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Propose请求。 
	➢ 不再接受Proposal ID小于（注意：这里是< ）当前请求的Accept请求。 
	➢ 不违背以前做出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。 
3. Propose: Proposer收到多数Acceptor的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果		  所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptor发 送Propose请求。 
4. Accept: Acceptor收到Propose请求后，在不违背自己之前做出的承诺下，接受并持久化当前Proposal ID和提案Value。 
5. Learn: Proposer收到多数Acceptor的Accept后，决议形成，将形成的决议发送给所有Learner。

对上述描述做三种情况的推演举例，为了简化流程，我们这里不设置 Learner。

**情况一：**
1. 有A1, A2, A3, A4, A5 5位议员，就税率问题进行决议。
![在这里插入图片描述](https://img-blog.csdnimg.cn/bbc6e381d6794dc59dafcb50f49005af.png)
2.  A1发起1号Proposal的Propose，等待Promise承诺；
3. A2-A5回应Promise；
4. A1在收到两份回复时就会发起税率10%的Proposal；
5. A2-A5回应Accept；
6. 通过Proposal，税率10%。

**情况二：**
7. 现在我们假设在A1提出提案的同时, A5决定将税率定为20%
![在这里插入图片描述](https://img-blog.csdnimg.cn/91e23ce8c4c24538a4f9286dfe17c07b.png)

8. A1，A5同时发起Propose（序号分别为1，2）
9.  A2承诺A1，A4承诺A5，A3行为成为关键
10. 情况1：A3先收到A1消息，承诺A1。
11. A1发起Proposal（1，10%），A2，A3接受。
12. 之后A3又收到A5消息，回复A1：（1，10%），并承诺A5。
13. A5发起Proposal（2，20%），A3，A4接受。之后A1，A5同时广播决议。

**Paxos 算法缺陷：在网络复杂的情况下，一个应用 Paxos 算法的分布式系统，可能很久无法收敛，甚至陷入活锁的情况。**

**情况3：**
1. 现在我们假设在A1提出提案的同时, A5决定将税率定为20%
![在这里插入图片描述](https://img-blog.csdnimg.cn/528726df347c4558bd6355d04cfd0680.png)

2. A1，A5同时发起Propose（序号分别为1，2） 
3. A2承诺A1，A4承诺A5，A3行为成为关键
4. 情况2：A3先收到A1消息，承诺A1。之后立刻收到A5消息，承诺A5。
5. A1发起Proposal（1，10%），无足够响应，A1重新Propose （序号3），A3再次承诺A1。
6. A5发起Proposal（2，20%），无足够相应。A5重新Propose （序号4），A3再次承诺A5。
7. ....

造成这种情况的原因是系统中有一个以上的 Proposer，多个 Proposers 相互争夺 Acceptor，造成迟迟无法达成一致的情况。针对这种情况，
**一种改进的 Paxos 算法被提出：从系统中选出一个节点作为 Leader，只有 Leader 能够发起提案。** 这样，一次 Paxos 流程中只有一个Proposer，不会出现活锁的情况，此时只会出现例子中第一种情况。

# 11、ZAB 协议
## 11.1、什么是 ZAB 算法
**Zab 借鉴了 Paxos 算法，是特别为 Zookeeper 设计的支持崩溃恢复的原子广播协议**。
	基于该协议，Zookeeper 设计为只有一台客户端（Leader）负责处理外部的写事务请求，然后Leader 客户端将数据同步到其他 Follower 节点。即 Zookeeper 只有一个 Leader 可以发起提案。
## 11.1、Zab 协议内容
**Zab 协议包括两种基本的模式：消息广播、崩溃恢复。**
## 11.1.1、消息广播
![在这里插入图片描述](https://img-blog.csdnimg.cn/a93440af04a54de494d9c719515c88cd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

1. 客户端发起一个写操作请求。
2. Leader服务器将客户端的请求转化为事务Proposal 提案，同时为每个Proposal 分配一个全局的ID，即zxid。
3. Leader服务器为每个Follower服务器分配一个单独的队列，然后将需要广播的 Proposal依次放到队列中去，并且根据FIFO策略进行消息发送。 
4. Follower接收到Proposal后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向Leader反馈一个Ack响应消息。
5. Leader接收到超过半数以上Follower的Ack响应消息后，即认为消息发送成功，可以发送commit消息。 
6. Leader向所有Follower广播commit消息，同时自身也会完成事务提交。Follower 接收到commit消息后，会将上一条事务提交。
7. Zookeeper采用Zab协议的核心，就是只要有一台服务器提交了Proposal，就要确保所有的服务器最终都能正确提交Proposal。


**ZAB协议针对事务请求的处理过程类似于一个两阶段提交过程**
- （1）广播事务阶段
- （2）广播提交操作

**这两阶段提交模型如下，有可能因为Leader宕机带来数据不一致，比如：**
- （1）Leader 发 起 一 个 事 务Proposal1 后 就 宕 机 ， Follower 都 没 有Proposal1 
- （2）Leader收到半数ACK宕 机，没来得及向Follower发送Commit

## 11.1.2、崩溃恢复
一旦Leader服务器出现崩溃或者由于网络原因导致Leader服务器失去了与过半 Follower的联系，那么就会进入崩溃恢复模式。
![在这里插入图片描述](https://img-blog.csdnimg.cn/fd411aeda7494b6eaa13add987dc4c42.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

**1）假设两种服务器异常情况：**
 - （1）假设一个事务在Leader提出之后，Leader挂了。
 - （2）一个事务在Leader上提交了，并且过半的Follower都响应Ack了，但是Leader在Commit消息发出之前挂了。

**2）Zab协议崩溃恢复要求满足以下两个要求：**
 - （1）确保已经被Leader提交的提案Proposal，必须最终被所有的Follower服务器提交。 **（已经产生的提案，Follower必须执行）**
 - （2）确保丢弃已经被Leader提出的，但是没有被提交的Proposal。**（丢弃胎死腹中的提案）**

**崩溃恢复主要包括两部分：Leader选举和数据恢复。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/39ae166592804945a29a864546ef6994.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**Leader选举：**
根据上述要求，Zab协议需要保证选举出来的Leader需要满足以下条件：
- （1）新选举出来的Leader不能包含未提交的Proposal。即新Leader必须都是已经提交了Proposal的Follower服务器节点
- （2）新选举的Leader节点中含有最大的zxid。这样做的好处是可以避免Leader服务器检查Proposal的提交和丢弃工作。

**Zab如何数据同步：** 
- （1）完成Leader选举后，在正式开始工作之前（接收事务请求，然后提出新的Proposal），Leader服务器会首先确认事务日
志中的所有的Proposal 是否已经被集群中过半的服务器Commit。
- （2）Leader服务器需要确保所有的Follower服务器能够接收到每一条事务的Proposal，并且能将所有已经提交的事务Proposal
应用到内存数据中。等到Follower将所有尚未同步的事务Proposal都从Leader服务器上同步过，并且应用到内存数据中以后，
Leader才会把该Follower加入到真正可用的Follower列表中。
# 12、CAP
CAP理论告诉我们，一个分布式系统不可能同时满足以下三种
**CAP理论**
- 一致性（C:Consistency）
-  可用性（A:Available）
- 分区容错性（P:Partition Tolerance）
- 
这三个基本需求，最多只能同时满足其中的两项，因为P是必须的，因此往往选择就在CP或者AP中。
 1）一致性（C:Consistency）
在分布式环境中，一致性是指数据在多个副本之间是否能够保持数据一致的特性。在一致性的需求下，当一个系统在数据一致的状态下执行更新操作后，应该保证系统的数据仍然处于一致的状态。
 2）可用性（A:Available）
可用性是指系统提供的服务必须一直处于可用的状态，对于用户的每一个操作请求总是能够在有限的时间内返回结果。
 3）分区容错性（P:Partition Tolerance）
分布式系统在遇到任何网络分区故障的时候，仍然需要能够保证对外提供满足一致性和可用性的服务，除非是整个网络环境都发生了故障。

**ZooKeeper保证的是CP**
（1）ZooKeeper不能保证每次服务请求的可用性。（注：在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要
重新请求才能获得结果）。所以说，ZooKeeper不能保证服务可用性。 
（2）进行Leader选举时集群都是不可用。
# 13、持久化源码  
Leader 和 Follower 中的数据会在内存和磁盘中各保存一份。所以需要将内存中的数据持久化到磁盘中。
在 org.apache.zookeeper.server.persistence 包下的相关类都是序列化相关的代码。
![在这里插入图片描述](https://img-blog.csdnimg.cn/586a8aa720e54950adc2d292368a28c2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 13.1、快照接口

```java
package org.apache.zookeeper.server.persistence;
import java.io.File;
import java.io.IOException;
import java.util.Map;

import org.apache.zookeeper.server.DataTree;

/**
 * snapshot interface for the persistence layer.
 * implement this interface for implementing 
 * snapshots.
 */
public interface SnapShot {
    
    /**
     * deserialize a data tree from the last valid snapshot and 
     * return the last zxid that was deserialized
     * @param dt the datatree to be deserialized into
     * @param sessions the sessions to be deserialized into
     * @return the last zxid that was deserialized from the snapshot
     * @throws IOException
     */
    long deserialize(DataTree dt, Map<Long, Integer> sessions) 
        throws IOException;
    
    /**
     * persist the datatree and the sessions into a persistence storage
     * @param dt the datatree to be serialized
     * @param sessions 
     * @throws IOException
     */
    void serialize(DataTree dt, Map<Long, Integer> sessions, 
            File name) 
        throws IOException;
    
    /**
     * find the most recent snapshot file
     * @return the most recent snapshot file
     * @throws IOException
     */
    File findMostRecentSnapshot() throws IOException;
    
    /**
     * free resources from this snapshot immediately
     * @throws IOException
     */
    void close() throws IOException;
} 

```

## 13.2、日志接口

```java
package org.apache.zookeeper.server.persistence;

import java.io.IOException;

import org.apache.jute.Record;
import org.apache.zookeeper.server.ServerStats;
import org.apache.zookeeper.txn.TxnHeader;

/**
 * Interface for reading transaction logs.
 *
 */
public interface TxnLog {

    /**
     * Setter for ServerStats to monitor fsync threshold exceed
     * @param serverStats used to update fsyncThresholdExceedCount
     */
    void setServerStats(ServerStats serverStats);
    
    /**
     * roll the current
     * log being appended to
     * @throws IOException 
     */
    void rollLog() throws IOException;
    /**
     * Append a request to the transaction log
     * @param hdr the transaction header
     * @param r the transaction itself
     * returns true iff something appended, otw false 
     * @throws IOException
     */
    boolean append(TxnHeader hdr, Record r) throws IOException;

    /**
     * Start reading the transaction logs
     * from a given zxid
     * @param zxid
     * @return returns an iterator to read the 
     * next transaction in the logs.
     * @throws IOException
     */
    TxnIterator read(long zxid) throws IOException;
    
    /**
     * the last zxid of the logged transactions.
     * @return the last zxid of the logged transactions.
     * @throws IOException
     */
    long getLastLoggedZxid() throws IOException;
    
    /**
     * truncate the log to get in sync with the 
     * leader.
     * @param zxid the zxid to truncate at.
     * @throws IOException 
     */
    boolean truncate(long zxid) throws IOException;
    
    /**
     * the dbid for this transaction log. 
     * @return the dbid for this transaction log.
     * @throws IOException
     */
    long getDbId() throws IOException;
    
    /**
     * commit the transaction and make sure
     * they are persisted
     * @throws IOException
     */
    void commit() throws IOException;

    /**
     *
     * @return transaction log's elapsed sync time in milliseconds
     */
    long getTxnLogSyncElapsedTime();
   
    /** 
     * close the transactions logs
     */
    void close() throws IOException;
    /**
     * an iterating interface for reading 
     * transaction logs. 
     */
    public interface TxnIterator {
        /**
         * return the transaction header.
         * @return return the transaction header.
         */
        TxnHeader getHeader();
        
        /**
         * return the transaction record.
         * @return return the transaction record.
         */
        Record getTxn();
     
        /**
         * go to the next transaction record.
         * @throws IOException
         */
        boolean next() throws IOException;
        
        /**
         * close files and release the 
         * resources
         * @throws IOException
         */
        void close() throws IOException;
        
        /**
         * Get an estimated storage space used to store transaction records
         * that will return by this iterator
         * @throws IOException
         */
        long getStorageSize() throws IOException;
    }
}
```
**处理持久化的核心类**
![在这里插入图片描述](https://img-blog.csdnimg.cn/019b8601cf704f3c8d3bdbb2a6a00011.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
# 14、序列化源码
zookeeper-jute 代码是关于 Zookeeper 序列化相关源码
![在这里插入图片描述](https://img-blog.csdnimg.cn/f7a76edcd0354cd7acf12251b1d66334.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 14.1、 序列化接口

```java
package org.apache.jute;

import org.apache.yetus.audience.InterfaceAudience;

import java.io.IOException;

/**
 * Interface that is implemented by generated classes.
 * 
 */
@InterfaceAudience.Public
public interface Record {
    public void serialize(OutputArchive archive, String tag)
        throws IOException;
    public void deserialize(InputArchive archive, String tag)
        throws IOException;
}

```

## 14.2、迭代

```java
public interface Index {
 // 结束
 public boolean done();
 // 下一个
 public void incr();
}
```

## 14.3、 序列化支持的数据类型

```java
package org.apache.jute;

import java.io.IOException;
import java.util.List;
import java.util.TreeMap;

/**
 * Interface that alll the serializers have to implement.
 *
 */
public interface OutputArchive {
    public void writeByte(byte b, String tag) throws IOException;
    public void writeBool(boolean b, String tag) throws IOException;
    public void writeInt(int i, String tag) throws IOException;
    public void writeLong(long l, String tag) throws IOException;
    public void writeFloat(float f, String tag) throws IOException;
    public void writeDouble(double d, String tag) throws IOException;
    public void writeString(String s, String tag) throws IOException;
    public void writeBuffer(byte buf[], String tag)
        throws IOException;
    public void writeRecord(Record r, String tag) throws IOException;
    public void startRecord(Record r, String tag) throws IOException;
    public void endRecord(Record r, String tag) throws IOException;
    public void startVector(List<?> v, String tag) throws IOException;
    public void endVector(List<?> v, String tag) throws IOException;
    public void startMap(TreeMap<?,?> v, String tag) throws IOException;
    public void endMap(TreeMap<?,?> v, String tag) throws IOException;

}

```

## 14.4、 反序列化支持的数据类型

```java
package org.apache.jute;

import java.io.IOException;

/**
 * Interface that all the Deserializers have to implement.
 *
 */
public interface InputArchive {
    public byte readByte(String tag) throws IOException;
    public boolean readBool(String tag) throws IOException;
    public int readInt(String tag) throws IOException;
    public long readLong(String tag) throws IOException;
    public float readFloat(String tag) throws IOException;
    public double readDouble(String tag) throws IOException;
    public String readString(String tag) throws IOException;
    public byte[] readBuffer(String tag) throws IOException;
    public void readRecord(Record r, String tag) throws IOException;
    public void startRecord(String tag) throws IOException;
    public void endRecord(String tag) throws IOException;
    public Index startVector(String tag) throws IOException;
    public void endVector(String tag) throws IOException;
    public Index startMap(String tag) throws IOException;
    public void endMap(String tag) throws IOException;
}

```
# 15、Zookeeper服务端初始化源码解析
**ZK服务端初始化源码解析流程图**

![在这里插入图片描述](https://img-blog.csdnimg.cn/39085aceb18b48d498647865383d9d0a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 15.1、 ZK 服务端启动脚本分析
1. zkServer.sh start 
```shell
#!/usr/bin/env bash
# use POSTIX interface, symlink is followed automatically
ZOOBIN="${BASH_SOURCE-$0}"
ZOOBIN="$(dirname "${ZOOBIN}")"
ZOOBINDIR="$(cd "${ZOOBIN}"; pwd)"

if [ -e "$ZOOBIN/../libexec/zkEnv.sh" ]; then
  . "$ZOOBINDIR"/../libexec/zkEnv.sh
else
  . "$ZOOBINDIR"/zkEnv.sh  //相当于获取 zkEnv.sh 中的环境变量（ZOOCFG="zoo.cfg"）
fi

# See the following page for extensive details on setting
# up the JVM to accept JMX remote management:
# http://java.sun.com/javase/6/docs/technotes/guides/management/agent.html
# by default we allow local JMX connections
if [ "x$JMXLOCALONLY" = "x" ]
then
    JMXLOCALONLY=false
fi

if [ "x$JMXDISABLE" = "x" ] || [ "$JMXDISABLE" = 'false' ]
then
  echo "ZooKeeper JMX enabled by default" >&2
  if [ "x$JMXPORT" = "x" ]
  then
    # for some reason these two options are necessary on jdk6 on Ubuntu
    #   accord to the docs they are not necessary, but otw jconsole cannot
    #   do a local attach
    ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY org.apache.zookeeper.server.quorum.QuorumPeerMain"
  else
    if [ "x$JMXAUTH" = "x" ]
    then
      JMXAUTH=false
    fi
    if [ "x$JMXSSL" = "x" ]
    then
      JMXSSL=false
    fi
    if [ "x$JMXLOG4J" = "x" ]
    then
      JMXLOG4J=true
    fi
    echo "ZooKeeper remote JMX Port set to $JMXPORT" >&2
    echo "ZooKeeper remote JMX authenticate set to $JMXAUTH" >&2
    echo "ZooKeeper remote JMX ssl set to $JMXSSL" >&2
    echo "ZooKeeper remote JMX log4j set to $JMXLOG4J" >&2
    ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=$JMXPORT -Dcom.sun.management.jmxremote.authenticate=$JMXAUTH -Dcom.sun.management.jmxremote.ssl=$JMXSSL -Dzookeeper.jmx.log4j.disable=$JMXLOG4J org.apache.zookeeper.server.quorum.QuorumPeerMain"
  fi
else
    echo "JMX disabled by user request" >&2
    ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain"
fi

if [ "x$SERVER_JVMFLAGS" != "x" ]
then
    JVMFLAGS="$SERVER_JVMFLAGS $JVMFLAGS"
fi

if [ "x$2" != "x" ]
then
    ZOOCFG="$ZOOCFGDIR/$2"
fi
......
```

2. zkServer.sh start 底层的实际执行内容

```java
nohup "$JAVA"
+ 一堆提交参数
+ $ZOOMAIN（org.apache.zookeeper.server.quorum.QuorumPeerMain）
+ "$ZOOCFG" （zkEnv.sh 文件中 ZOOCFG="zoo.cfg"）
```
## 15.2、ZK 服务端启动入口

QuorumPeerMain类
```java
public static void main(String[] args) {
		// 创建了一个 zk 节点
        QuorumPeerMain main = new QuorumPeerMain();
        try {
        	// 初始化节点并运行， ，args 相当于提交参数中的 zoo.cfg
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
```
initializeAndRun方法

```java
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
    	// 管理 zk 的配置信息
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
       		 // 1 解析参数， ，zoo.cfg 和 myid
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        // 2 启动 定时 任务， 对过期的快照，执行删除 （默认该功能关闭）
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.isDistributed()) {
        	// 3 启动集群
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```
parse方法 

```java
 public void parse(String path) throws ConfigException {
        LOG.info("Reading configuration from: " + path);
       
        try {
        	// 校验文件路径及是否存在
            File configFile = (new VerifyingFileFactory.Builder(LOG)
                .warnForRelativePath()
                .failForNonExistingPath()
                .build()).create(path);
                
            Properties cfg = new Properties();
            FileInputStream in = new FileInputStream(configFile);
            try {
            	// 加载配置文件
                cfg.load(in);
                configFileStr = path;
            } finally {
                in.close();
            }
            // 解析配置文件
            parseProperties(cfg);
        } catch (IOException e) {
```
parseProperties方法

```java
public void parseProperties(Properties zkProp)
    throws IOException, ConfigException {
        int clientPort = 0;
        int secureClientPort = 0;
        String clientPortAddress = null;
        String secureClientPortAddress = null;
        VerifyingFileFactory vff = new VerifyingFileFactory.Builder(LOG).warnForRelativePath().build();
        // 读取 zoo.cfg 文件中的属性值，并赋值给 QuorumPeerConfig 的类对象
        for (Entry<Object, Object> entry : zkProp.entrySet()) {
            String key = entry.getKey().toString().trim();
            String value = entry.getValue().toString().trim();
            if (key.equals("dataDir")) {
                dataDir = vff.create(value);
            } else if (key.equals("dataLogDir")) {
                dataLogDir = vff.create(value);
            } else if (key.equals("clientPort")) {
                clientPort = Integer.parseInt(value);
            } else if (key.equals("localSessionsEnabled")) {
                localSessionsEnabled = Boolean.parseBoolean(value);
            } else if (key.equals("localSessionsUpgradingEnabled")) {
                localSessionsUpgradingEnabled = Boolean.parseBoolean(value);
            } else if (key.equals("clientPortAddress")) {
            ......
    if (dynamicConfigFileStr == null) {
    		// 设置myid
            setupQuorumPeerConfig(zkProp, true);
            if (isDistributed() && isReconfigEnabled()) {
                // we don't backup static config for standalone mode.
                // we also don't backup if reconfig feature is disabled.
                backupOldConfig();
            }
        }
```
setupQuorumPeerConfig方法

```java
    void setupQuorumPeerConfig(Properties prop, boolean configBackwardCompatibilityMode)
            throws IOException, ConfigException {
        quorumVerifier = parseDynamicConfig(prop, electionAlg, true, configBackwardCompatibilityMode);
        setupMyId();
        setupClientPort();
        setupPeerType();
        checkValidity();
    }
```
setupMyId方法
```java
private void setupMyId() throws IOException {
// 读取数据目录下的myid文件
File myIdFile = new File(dataDir, "myid");
// standalone server doesn't need myid file.
if (!myIdFile.isFile()) {
    return;
 }
BufferedReader br = new BufferedReader(new FileReader(myIdFile));
String myIdString;
try {
  myIdString = br.readLine();
  } finally {
  br.close();
}
try {
serverId = Long.parseLong(myIdString);       
 MDC.put("myid", myIdString);
} catch (NumberFormatException e) {
throw new IllegalArgumentException("serverid " + myIdString + " is not a number");
 }
}
```
## 15.3、过期快照删除
initializeAndRun方法
```java
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
    	// 管理 zk 的配置信息
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
       		 // 1 解析参数， ，zoo.cfg 和 myid
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        // 2 启动 定时 任务， 对过期的快照，执行删除 （默认该功能关闭）
		// config.getSnapRetainCount() = 3 最少保留的快照个数
		// config.getPurgeInterval() = 0 默认 0 表示关闭
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.isDistributed()) {
        	// 3 启动集群
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```
 purgeMgr.start();
```java
public void start() {
if (PurgeTaskStatus.STARTED == purgeTaskStatus) {
LOG.warn("Purge task is already running.");
return;
}
// 默认情况 purgeInterval=0 ，该任务关闭，直接返回
// Don't schedule the purge task with zero or negative purge interval.
if (purgeInterval <= 0) {
LOG.info("Purge task is not scheduled.");
return;
}
// 创建一个定时器
timer = new Timer("PurgeTask", true);
// 创建一个清理快照任务
TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
// 如果 purgeInterval 设置的值是 1 ，表示 1 小时检查一次，判断是否有过期快照，有则删除
timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));
purgeTaskStatus = PurgeTaskStatus.STARTED;
}
```
PurgeTask
```java
static class PurgeTask extends TimerTask {
        private File logsDir;
        private File snapsDir;
        private int snapRetainCount;

        public PurgeTask(File dataDir, File snapDir, int count) {
            logsDir = dataDir;
            snapsDir = snapDir;
            snapRetainCount = count;
        }

        @Override
        public void run() {
            LOG.info("Purge task started.");
            try {
            	// 清理过期的数据
                PurgeTxnLog.purge(logsDir, snapsDir, snapRetainCount);
            } catch (Exception e) {
                LOG.error("Error occurred while purging.", e);
            }
            LOG.info("Purge task completed.");
        }
    }
```
purge()方法

```java
public static void purge(File dataDir, File snapDir, int num) throws IOException {
        if (num < 3) {
            throw new IllegalArgumentException(COUNT_ERR_MSG);
        }

        FileTxnSnapLog txnLog = new FileTxnSnapLog(dataDir, snapDir);

        List<File> snaps = txnLog.findNRecentSnapshots(num);
        int numSnaps = snaps.size();
        if (numSnaps > 0) {
        	// 清理老的快照文件
            purgeOlderSnapshots(txnLog, snaps.get(numSnaps - 1));
        }
    }
```
## 15.4、初始化通信组件
initializeAndRun（）方法
```java
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.isDistributed()) {
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```
runFromConfig

```java
public void runFromConfig(QuorumPeerConfig config)
            throws IOException, AdminServerException
    {
      try {
          ManagedUtil.registerLog4jMBeans();
      } catch (JMException e) {
          LOG.warn("Unable to register log4j JMX control", e);
      }

      LOG.info("Starting quorum peer");
      try {
          ServerCnxnFactory cnxnFactory = null;
          ServerCnxnFactory secureCnxnFactory = null;
		  
		  // 通信组件初始化，默认是 NIO 通信
          if (config.getClientPortAddress() != null) {
              cnxnFactory = ServerCnxnFactory.createFactory();
              cnxnFactory.configure(config.getClientPortAddress(),
                      config.getMaxClientCnxns(),
                      false);
          }
			
          if (config.getSecureClientPortAddress() != null) {
              secureCnxnFactory = ServerCnxnFactory.createFactory();
              secureCnxnFactory.configure(config.getSecureClientPortAddress(),
                      config.getMaxClientCnxns(),
                      true);
          }
		 // 把解析的参数赋值给该 zookeeper 节点
          quorumPeer = getQuorumPeer();
          quorumPeer.setTxnFactory(new FileTxnSnapLog(
                      config.getDataLogDir(),
                      config.getDataDir()));
          quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());

          ......
          // 管理 zk 的通信
		  quorumPeer.setCnxnFactory(cnxnFactory);
		  quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
		  quorumPeer.setSslQuorum(config.isSslQuorum());
		  quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
          quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
          quorumPeer.initialize();
          // 启动 zk
          quorumPeer.start();
          quorumPeer.join();
          
```
createFactory()
```java
static public ServerCnxnFactory createFactory() throws IOException {
	String serverCnxnFactoryName = System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
	if (serverCnxnFactoryName == null) {
		serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
	}
	try {
		ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory)
		Class.forName(serverCnxnFactoryName)
		.getDeclaredConstructor().newInstance();
		LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
		return serverCnxnFactory;
	} catch (Exception e) {
		IOException ioe = new IOException("Couldn't instantiate "
		+ serverCnxnFactoryName);
		ioe.initCause(e);
		throw ioe;
	}
}
```
ctrl + alt +B 查找 configure 实现类，NIOServerCnxnFactory.java

```java
public void configure(InetSocketAddress addr, int maxcc, boolean secure) throws IOException
{
	if (secure) {
		throw new UnsupportedOperationException("SSL isn't supported in
		NIOServerCnxn");
	}
	configureSaslLogin();
	maxClientCnxns = maxcc;
	sessionlessCnxnTimeout = Integer.getInteger(
	ZOOKEEPER_NIO_SESSIONLESS_CNXN_TIMEOUT, 10000);
	// We also use the sessionlessCnxnTimeout as expiring interval for
	// cnxnExpiryQueue. These don't need to be the same, but the expiring
	// interval passed into the ExpiryQueue() constructor below should be
	// less than or equal to the timeout.
	cnxnExpiryQueue =
	new ExpiryQueue<NIOServerCnxn>(sessionlessCnxnTimeout);
	expirerThread = new ConnectionExpirerThread();
	int numCores = Runtime.getRuntime().availableProcessors();
	// 32 cores sweet spot seems to be 4 selector threads
	numSelectorThreads = Integer.getInteger(
	ZOOKEEPER_NIO_NUM_SELECTOR_THREADS,
	Math.max((int) Math.sqrt((float) numCores/2), 1));
	if (numSelectorThreads < 1) {
		throw new IOException("numSelectorThreads must be at least 1");
	}
	numWorkerThreads = Integer.getInteger(
	ZOOKEEPER_NIO_NUM_WORKER_THREADS, 2 * numCores);
	workerShutdownTimeoutMS = Long.getLong(
	ZOOKEEPER_NIO_SHUTDOWN_TIMEOUT, 5000);
	... ...
	for(int i=0; i<numSelectorThreads; ++i) {
		selectorThreads.add(new SelectorThread(i));
	}
	// 初始化 NIO 服务端 socket ，绑定 2181 端口，可以接收客户端请求
	this.ss = ServerSocketChannel.open();
	ss.socket().setReuseAddress(true);
	LOG.info("binding to port " + addr);
	// 绑定 2181 端口
	ss.socket().bind(addr);
	ss.configureBlocking(false);
	acceptThread = new AcceptThread(ss, addr, selectorThreads);
}
```

