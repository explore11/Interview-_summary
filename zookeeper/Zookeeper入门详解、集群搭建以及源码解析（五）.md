
# 18、Follower 和 Leader 状态同步源码
当选举结束后，每个节点都需要根据自己的角色更新自己的状态。选举出的 Leader 更新自己状态为 Leader，其他节点更新自己状态为Follower。
Leader 更新状态入口：leader.lead()
Follower 更新状态入口：follower.followerLeader()
注意：
（1）follower 必须要让 leader 知道自己的状态：epoch、zxid、sid 
- 必须要找出谁是 leader；
- 发起请求连接 leader；
- 发送自己的信息给 leader；
- leader 接收到信息，必须要返回对应的信息给 follower。

（2）当leader得知follower的状态了，就确定需要做何种方式的数据同步DIFF、TRUNC、SNAP
（3）执行数据同步
（4）当 leader 接收到超过半数 follower 的 ack 之后，进入正常工作状态，集群启动完成了

**最终数据总结同步的方式：**
- DIFF 咱两一样，不需要做什么
- TRUNC follower 的 zxid 比 leader 的 zxid 大，所以 Follower 要回滚
- COMMIT leader 的 zxid 比 follower 的 zxid 大，发送 Proposal 给 foloower 提交执行
- SNAP 如果 follower 并没有任何数据，直接使用 SNAP 的方式来执行数据同步（直接把数据全部序列到 follower）

## 18.1、Follower和Leader状态同步源码解析图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/cbd30f136db447fb8e73879a3b7e92a4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 18.2、Follower和Leader状态同步源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/2c9319309881484a8a7a6d5846b68123.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 18.2.1、Leader.lead()等待接收 follower 的状态同步申请
Leader.lead()
```java
    void lead() throws IOException, InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("LEADING - LEADER ELECTION TOOK - {} {}", electionTimeTaken,
                QuorumPeer.FLE_TIME_UNIT);
        self.start_fle = 0;
        self.end_fle = 0;

        zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);

        try {
            self.tick.set(0);
            // 恢复数据到内存，启动时，其实已经加载过了
            zk.loadData();

            leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

            // Start thread that waits for connection requests from
            // new followers.
            // 等待其他 follower 节点向 leader 节点发送同步状态
            cnxAcceptor = new LearnerCnxAcceptor();
            cnxAcceptor.start();

            long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());

            zk.setZxid(ZxidUtils.makeZxid(epoch, 0));

            synchronized(this){
                lastProposed = zk.getZxid();
            }
           ......
        } finally {
            zk.unregisterJMX(this);
        }
    }

```
LearnerCnxAcceptor

```java
      public void run() {
            try {
                while (!stop) {
                    Socket s = null;
                    boolean error = false;
                    try {
                   		 // 等待接收 follower 的状态同步申请
                        s = ss.accept();

                        // start with the initLimit, once the ack is processed
                        // in LearnerHandler switch to the syncLimit
                        s.setSoTimeout(self.tickTime * self.initLimit);
                        s.setTcpNoDelay(nodelay);

                        BufferedInputStream is = new BufferedInputStream(
                                s.getInputStream());
                        // 一旦接收到 follower 的请求，就创建 LearnerHandler 对象，处理请求
                        LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                        // 启动线程
                        fh.start();
                    } catch (SocketException e) {
                        ......
            } catch (Exception e) {
                LOG.warn("Exception while accepting follower", e.getMessage());
                handleException(this.getName(), e);
            }
        }
```
其中 ss 的初始化是在创建 Leader 对象时，创建的 socket

```java
Leader(QuorumPeer self,LeaderZooKeeperServer zk) throws IOException {
        this.self = self;
        this.proposalStats = new BufferStats();
        try {
            if (self.shouldUsePortUnification() || self.isSslQuorum()) {
                boolean allowInsecureConnection = self.shouldUsePortUnification();
                if (self.getQuorumListenOnAllIPs()) {
                    ss = new UnifiedServerSocket(self.getX509Util(), allowInsecureConnection, self.getQuorumAddress().getPort());
                } else {
                    ss = new UnifiedServerSocket(self.getX509Util(), allowInsecureConnection);
                }
            } else {
                if (self.getQuorumListenOnAllIPs()) {
                    ss = new ServerSocket(self.getQuorumAddress().getPort());
                } else {
                    ss = new ServerSocket();
                }
            }
            ss.setReuseAddress(true);
            if (!self.getQuorumListenOnAllIPs()) {
                ss.bind(self.getQuorumAddress());
            }
        } catch (BindException e) {
            ......
        }
        this.zk = zk;
        this.learnerSnapshotThrottler = createLearnerSnapshotThrottler(
                maxConcurrentSnapshots, maxConcurrentSnapshotTimeout);
    }
```

### 18.2.2、Follower.followLeader()查找并连接 Leader

```java
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {} {}", electionTimeTaken,
                QuorumPeer.FLE_TIME_UNIT);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
       		 // 查找 leader
            QuorumServer leaderServer = findLeader();            
            try {
            	// 连接 leader
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                // 向 leader 注册
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
                if (self.isReconfigStateChange())
                   throw new Exception("learned about role change");
                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                syncWithLeader(newEpochZxid);                
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }
```

findLeader()

```java
protected QuorumServer findLeader() {
        QuorumServer leaderServer = null;
        // Find the leader by id
        // 选举投票的时候记录的，最后推荐的 leader 的 sid
        Vote current = self.getCurrentVote();
        // 如果这个 sid 在启动的所有服务器范围中
        for (QuorumServer s : self.getView().values()) {
            if (s.id == current.getId()) {
                // Ensure we have the leader's correct IP address before
                // attempting to connect.
                // 尝试连接 leader 的正确 IP 地址
                s.recreateSocketAddresses();
                leaderServer = s;
                break;
            }
        }
        if (leaderServer == null) {
            LOG.warn("Couldn't find the leader with id = "
                    + current.getId());
        }
        return leaderServer;
    }
```
connectToLeader(leaderServer.addr, leaderServer.hostname)

```java
protected void connectToLeader(InetSocketAddress addr, String hostname)
            throws IOException, InterruptedException, X509Exception {
        this.sock = createSocket();

        int initLimitTime = self.tickTime * self.initLimit;
        int remainingInitLimitTime = initLimitTime;
        long startNanoTime = nanoTime();

        for (int tries = 0; tries < 5; tries++) {
            try {
                // recalculate the init limit time because retries sleep for 1000 milliseconds
                remainingInitLimitTime = initLimitTime - (int)((nanoTime() - startNanoTime) / 1000000);
                if (remainingInitLimitTime <= 0) {
                    LOG.error("initLimit exceeded on retries.");
                    throw new IOException("initLimit exceeded on retries.");
                }

                sockConnect(sock, addr, Math.min(self.tickTime * self.syncLimit, remainingInitLimitTime));
                if (self.isSslQuorum())  {
                    ((SSLSocket) sock).startHandshake();
                }
                sock.setTcpNoDelay(nodelay);
                break;
            } catch (IOException e) {
                remainingInitLimitTime = initLimitTime - (int)((nanoTime() - startNanoTime) / 1000000);

                if (remainingInitLimitTime <= 1000) {
                    LOG.error("Unexpected exception, initLimit exceeded. tries=" + tries +
                             ", remaining init limit=" + remainingInitLimitTime +
                             ", connecting to " + addr,e);
                    throw e;
                } else if (tries >= 4) {
                    LOG.error("Unexpected exception, retries exceeded. tries=" + tries +
                             ", remaining init limit=" + remainingInitLimitTime +
                             ", connecting to " + addr,e);
                    throw e;
                } else {
                    LOG.warn("Unexpected exception, tries=" + tries +
                            ", remaining init limit=" + remainingInitLimitTime +
                            ", connecting to " + addr,e);
                    this.sock = createSocket();
                }
            }
            Thread.sleep(1000);
        }

        self.authLearner.authenticate(sock, hostname);

        leaderIs = BinaryInputArchive.getArchive(new BufferedInputStream(
                sock.getInputStream()));
        bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
        leaderOs = BinaryOutputArchive.getArchive(bufferedOutput);
    }
```

### 18.2.3、Leader.lead()创建 LearnerHandler
Leader.lead()
```java
    void lead() throws IOException, InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("LEADING - LEADER ELECTION TOOK - {} {}", electionTimeTaken,
                QuorumPeer.FLE_TIME_UNIT);
        self.start_fle = 0;
        self.end_fle = 0;

        zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);

        try {
            self.tick.set(0);
            // 恢复数据到内存，启动时，其实已经加载过了
            zk.loadData();

            leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

            // Start thread that waits for connection requests from
            // new followers.
            // 等待其他 follower 节点向 leader 节点发送同步状态
            cnxAcceptor = new LearnerCnxAcceptor();
            cnxAcceptor.start();

            long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());

            zk.setZxid(ZxidUtils.makeZxid(epoch, 0));

            synchronized(this){
                lastProposed = zk.getZxid();
            }
           ......
        } finally {
            zk.unregisterJMX(this);
        }
    }

```
LearnerCnxAcceptor

```java
      public void run() {
            try {
                while (!stop) {
                    Socket s = null;
                    boolean error = false;
                    try {
                   		 // 等待接收 follower 的状态同步申请
                        s = ss.accept();

                        // start with the initLimit, once the ack is processed
                        // in LearnerHandler switch to the syncLimit
                        s.setSoTimeout(self.tickTime * self.initLimit);
                        s.setTcpNoDelay(nodelay);

                        BufferedInputStream is = new BufferedInputStream(
                                s.getInputStream());
                        // 一旦接收到 follower 的请求，就创建 LearnerHandler 对象，处理请求
                        LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                        // 启动线程
                        fh.start();
                    } catch (SocketException e) {
                        ......
            } catch (Exception e) {
                LOG.warn("Exception while accepting follower", e.getMessage());
                handleException(this.getName(), e);
            }
        }
```
 LearnerHandler
由于 public class LearnerHandler extends ZooKeeperThread{}，说明 LearnerHandler 是一个线程。所以 fh.start()执行的是 LearnerHandler 中的 run()方法。 

```java
    public void run() {
        try {
            leader.addLearnerHandler(this);
            // 心跳处理
            tickOfNextAckDeadline = leader.self.tick.get() + leader.self.initLimit + leader.self.syncLimit;
            ia = BinaryInputArchive.getArchive(bufferedInput);
            bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
            oa = BinaryOutputArchive.getArchive(bufferedOutput);
			// 从网络中接收消息，并反序列化为 packet
            QuorumPacket qp = new QuorumPacket();
            ia.readRecord(qp, "packet");
            if(qp.getType() != Leader.FOLLOWERINFO && qp.getType() != Leader.OBSERVERINFO){
                LOG.error("First packet " + qp.toString()
                        + " is not FOLLOWERINFO or OBSERVERINFO!");
                return;
            }

            byte learnerInfoData[] = qp.getData();
            // 选举结束后，observer 和 follower 都应该给 leader 发送一个标志信息：FOLLOWERINFO 或者 OBSERVERINFO
            if (learnerInfoData != null) {
                ByteBuffer bbsid = ByteBuffer.wrap(learnerInfoData);
                if (learnerInfoData.length >= 8) {
                    this.sid = bbsid.getLong();
                }
                if (learnerInfoData.length >= 12) {
                    this.version = bbsid.getInt(); // protocolVersion
                }
                if (learnerInfoData.length >= 20) {
                    long configVersion = bbsid.getLong();
                    if (configVersion > leader.self.getQuorumVerifier().getVersion()) {
                        throw new IOException("Follower is ahead of the leader (has a later activated configuration)");
                    }
                }
            } else {
                this.sid = leader.followerCounter.getAndDecrement();
            }

            if (leader.self.getView().containsKey(this.sid)) {
                LOG.info("Follower sid: " + this.sid + " : info : "
                        + leader.self.getView().get(this.sid).toString());
            } else {
                LOG.info("Follower sid: " + this.sid + " not in the current config " + Long.toHexString(leader.self.getQuorumVerifier().getVersion()));
            }
                        
            if (qp.getType() == Leader.OBSERVERINFO) {
                  learnerType = LearnerType.OBSERVER;
            }
			// 读取 Follower 发送过来的 lastAcceptedEpoch
			// 选举过程中，所使用的 epoch，其实还是上一任 leader 的 epoch
            long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());

            long peerLastZxid;
            StateSummary ss = null;
            // 读取 follower 发送过来的 zxid
            long zxid = qp.getZxid();
            // Leader 根据从 Follower 获取 sid 和旧的 epoch，构建新的 epoch
            long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
            long newLeaderZxid = ZxidUtils.makeZxid(newEpoch, 0);

            if (this.getVersion() < 0x10000) {
                // we are going to have to extrapolate the epoch information
                long epoch = ZxidUtils.getEpochFromZxid(zxid);
                ss = new StateSummary(epoch, zxid);
                // fake the message
                leader.waitForEpochAck(this.getSid(), ss);
            } else {
                byte ver[] = new byte[4];
                ByteBuffer.wrap(ver).putInt(0x10000);
                // Leader 向 Follower 发送信息（包含:zxid 和 newEpoch）
                QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, newLeaderZxid, ver, null);
                oa.writeRecord(newEpochPacket, "packet");
                bufferedOutput.flush();
                QuorumPacket ackEpochPacket = new QuorumPacket();
                ia.readRecord(ackEpochPacket, "packet");
                if (ackEpochPacket.getType() != Leader.ACKEPOCH) {
                    LOG.error(ackEpochPacket.toString()
                            + " is not ACKEPOCH");
                    return;
				}
                ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
                ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
                leader.waitForEpochAck(this.getSid(), ss);
            }
            peerLastZxid = ss.getLastZxid();
           
            ......
        } finally {
            LOG.warn("******* GOODBYE "
                    + (sock != null ? sock.getRemoteSocketAddress() : "<null>")
                    + " ********");
            shutdown();
        }
    }
```

### 18.2.4、Follower.lead()创建 registerWithLeader 
```java
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {} {}", electionTimeTaken,
                QuorumPeer.FLE_TIME_UNIT);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
       		 // 查找 leader
            QuorumServer leaderServer = findLeader();            
            try {
            	// 连接 leader
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                // 向 leader 注册
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
                if (self.isReconfigStateChange())
                   throw new Exception("learned about role change");
                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                syncWithLeader(newEpochZxid);                
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }
```
registerWithLeader(Leader.FOLLOWERINFO)

```java
    protected long registerWithLeader(int pktType) throws IOException{
        /*
         * Send follower info, including last zxid and sid
         */
    	long lastLoggedZxid = self.getLastLoggedZxid();
        QuorumPacket qp = new QuorumPacket();                
        qp.setType(pktType);
        qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
        
        /*
         * Add sid to payload
         */
        LearnerInfo li = new LearnerInfo(self.getId(), 0x10000, self.getQuorumVerifier().getVersion());
        ByteArrayOutputStream bsid = new ByteArrayOutputStream();
        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
        boa.writeRecord(li, "LearnerInfo");
        qp.setData(bsid.toByteArray());
        // 发送 FollowerInfo 给 Leader
        writePacket(qp, true);
        // 读取 Leader 返回的结果：LeaderInfo
        readPacket(qp);        
        final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
        // 如果接收到 LeaderInfo
		if (qp.getType() == Leader.LEADERINFO) {
        	// we are connected to a 1.0 server so accept the new epoch and read the next packet
        	leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
        	byte epochBytes[] = new byte[4];
        	final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);
        	// 接收 leader 的 epoch
        	if (newEpoch > self.getAcceptedEpoch()) {
        		// 把自己原来的 epoch 保存在 wrappedEpochBytes 里
        		wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
        		// 把 leader 发送过来的 epoch 保存起来
        		self.setAcceptedEpoch(newEpoch);
        	} else if (newEpoch == self.getAcceptedEpoch()) {
        		// since we have already acked an epoch equal to the leaders, we cannot ack
        		// again, but we still need to send our lastZxid to the leader so that we can
        		// sync with it if it does assume leadership of the epoch.
        		// the -1 indicates that this reply should not count as an ack for the new epoch
                wrappedEpochBytes.putInt(-1);
        	} else {
        		throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
        	}
        	// 发送 ackepoch 给 leader（包含了自己的：epoch 和 zxid）
        	QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
        	writePacket(ackNewEpoch, true);
            return ZxidUtils.makeZxid(newEpoch, 0);
        } else {
        	if (newEpoch > self.getAcceptedEpoch()) {
        		self.setAcceptedEpoch(newEpoch);
        	}
            if (qp.getType() != Leader.NEWLEADER) {
                LOG.error("First packet should have been NEWLEADER");
                throw new IOException("First packet should have been NEWLEADER");
            }
            return qp.getZxid();
        }
    } 
```

### 18.2.5、Leader.lead()接收 Follwer 状态，根据同步方式发送同步消息
 LearnerHandler
由于 public class LearnerHandler extends ZooKeeperThread{}，说明 LearnerHandler 是一个线程。所以 fh.start()执行的是 LearnerHandler 中的 run()方法。 

```java
    public void run() {
        try {
            leader.addLearnerHandler(this);
            // 心跳处理
            tickOfNextAckDeadline = leader.self.tick.get() + leader.self.initLimit + leader.self.syncLimit;
            ia = BinaryInputArchive.getArchive(bufferedInput);
            bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
            oa = BinaryOutputArchive.getArchive(bufferedOutput);
			// 从网络中接收消息，并反序列化为 packet
            QuorumPacket qp = new QuorumPacket();
            ia.readRecord(qp, "packet");
            if(qp.getType() != Leader.FOLLOWERINFO && qp.getType() != Leader.OBSERVERINFO){
                LOG.error("First packet " + qp.toString()
                        + " is not FOLLOWERINFO or OBSERVERINFO!");
                return;
            }

            byte learnerInfoData[] = qp.getData();
            // 选举结束后，observer 和 follower 都应该给 leader 发送一个标志信息：FOLLOWERINFO 或者 OBSERVERINFO
            if (learnerInfoData != null) {
                ByteBuffer bbsid = ByteBuffer.wrap(learnerInfoData);
                if (learnerInfoData.length >= 8) {
                    this.sid = bbsid.getLong();
                }
                if (learnerInfoData.length >= 12) {
                    this.version = bbsid.getInt(); // protocolVersion
                }
                if (learnerInfoData.length >= 20) {
                    long configVersion = bbsid.getLong();
                    if (configVersion > leader.self.getQuorumVerifier().getVersion()) {
                        throw new IOException("Follower is ahead of the leader (has a later activated configuration)");
                    }
                }
            } else {
                this.sid = leader.followerCounter.getAndDecrement();
            }

            if (leader.self.getView().containsKey(this.sid)) {
                LOG.info("Follower sid: " + this.sid + " : info : "
                        + leader.self.getView().get(this.sid).toString());
            } else {
                LOG.info("Follower sid: " + this.sid + " not in the current config " + Long.toHexString(leader.self.getQuorumVerifier().getVersion()));
            }
                        
            if (qp.getType() == Leader.OBSERVERINFO) {
                  learnerType = LearnerType.OBSERVER;
            }
			// 读取 Follower 发送过来的 lastAcceptedEpoch
			// 选举过程中，所使用的 epoch，其实还是上一任 leader 的 epoch
            long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());

            long peerLastZxid;
            StateSummary ss = null;
            // 读取 follower 发送过来的 zxid
            long zxid = qp.getZxid();
            // Leader 根据从 Follower 获取 sid 和旧的 epoch，构建新的 epoch
            long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
            long newLeaderZxid = ZxidUtils.makeZxid(newEpoch, 0);

            if (this.getVersion() < 0x10000) {
                // we are going to have to extrapolate the epoch information
                long epoch = ZxidUtils.getEpochFromZxid(zxid);
                ss = new StateSummary(epoch, zxid);
                // fake the message
                leader.waitForEpochAck(this.getSid(), ss);
            } else {
                byte ver[] = new byte[4];
                ByteBuffer.wrap(ver).putInt(0x10000);
                // Leader 向 Follower 发送信息（包含:zxid 和 newEpoch）
                QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, newLeaderZxid, ver, null);
                oa.writeRecord(newEpochPacket, "packet");
                bufferedOutput.flush();
                // 接收到 Follower 应答的 ackepoch
                QuorumPacket ackEpochPacket = new QuorumPacket();
                ia.readRecord(ackEpochPacket, "packet");
                if (ackEpochPacket.getType() != Leader.ACKEPOCH) {
                    LOG.error(ackEpochPacket.toString()
                            + " is not ACKEPOCH");
                    return;
				}
                ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
                // 保存了对方 follower 或者 observer 的状态：epoch 和 zxid
                ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
                leader.waitForEpochAck(this.getSid(), ss);
            }
            peerLastZxid = ss.getLastZxid();
           	// Take any necessary action if we need to send TRUNC or DIFF
			// startForwarding() will be called in all cases
			// 方法判断 Leader 和 Follower 是否需要同步
			 boolean needSnap = syncFollower(peerLastZxid, leader.zk.getZKDatabase(), 
			leader);
			 
			 /* if we are not truncating or sending a diff just send a snapshot */
			 if (needSnap) {
			 boolean exemptFromThrottle = getLearnerType() != LearnerType.OBSERVER;
			 LearnerSnapshot snapshot = 
			 
			leader.getLearnerSnapshotThrottler().beginSnapshot(exemptFromThrottle);
			 try {
			 long zxidToSend = 
			leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
			 oa.writeRecord(new QuorumPacket(Leader.SNAP, zxidToSend, null, 
			null), "packet");
			 bufferedOutput.flush();
            ......
        } finally {
            LOG.warn("******* GOODBYE "
                    + (sock != null ? sock.getRemoteSocketAddress() : "<null>")
                    + " ********");
            shutdown();
        }
    }
```
 syncFollower(peerLastZxid, leader.zk.getZKDatabase(), leader)
```java
    public boolean syncFollower(long peerLastZxid, ZKDatabase db, Leader leader) {

        boolean isPeerNewEpochZxid = (peerLastZxid & 0xffffffffL) == 0;
        // Keep track of the latest zxid which already queued
        long currentZxid = peerLastZxid;
        boolean needSnap = true;
        boolean txnLogSyncEnabled = db.isTxnLogSyncEnabled();
        ReentrantReadWriteLock lock = db.getLogLock();
        ReadLock rl = lock.readLock();
        try {
            rl.lock();
            long maxCommittedLog = db.getmaxCommittedLog();
            long minCommittedLog = db.getminCommittedLog();
            long lastProcessedZxid = db.getDataTreeLastProcessedZxid();

            LOG.info("Synchronizing with Follower sid: {} maxCommittedLog=0x{}"
                    + " minCommittedLog=0x{} lastProcessedZxid=0x{}"
                    + " peerLastZxid=0x{}", getSid(),
                    Long.toHexString(maxCommittedLog),
                    Long.toHexString(minCommittedLog),
                    Long.toHexString(lastProcessedZxid),
                    Long.toHexString(peerLastZxid));

            if (db.getCommittedLog().isEmpty()) {
              
                minCommittedLog = lastProcessedZxid;
                maxCommittedLog = lastProcessedZxid;
            }
			......

        return needSnap;
    }
```

### 18.2.6、Follower.lead()应答 Leader 同步结果

```java
protected void processPacket(QuorumPacket qp) throws Exception{
        switch (qp.getType()) {
        case Leader.PING:            
            ping(qp);            
            break;
        case Leader.PROPOSAL:           
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            if (hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
            }
            lastQueued = hdr.getZxid();
            
            if (hdr.getType() == OpCode.reconfig){
               SetDataTxn setDataTxn = (SetDataTxn) txn;       
               QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
               self.setLastSeenQuorumVerifier(qv, true);                               
            }
            
            fzk.logRequest(hdr, txn);
            break;
        case Leader.COMMIT:
            fzk.commit(qp.getZxid());
            break;
            
        case Leader.COMMITANDACTIVATE:
           // get the new configuration from the request
           Request request = fzk.pendingTxns.element();
           SetDataTxn setDataTxn = (SetDataTxn) request.getTxn();                                                                                                      
           QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));                                
 
           // get new designated leader from (current) leader's message
           ByteBuffer buffer = ByteBuffer.wrap(qp.getData());    
           long suggestedLeaderId = buffer.getLong();
            boolean majorChange = 
                   self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
           // commit (writes the new config to ZK tree (/zookeeper/config)                     
           fzk.commit(qp.getZxid());
            if (majorChange) {
               throw new Exception("changes proposed in reconfig");
           }
           break;
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Follower started");
            break;
        case Leader.REVALIDATE:
            revalidate(qp);
            break;
        case Leader.SYNC:
            fzk.sync();
            break;
        default:
            LOG.warn("Unknown packet type: {}", LearnerHandler.packetToString(qp));
            break;
        }
    }
```
fzk.commit(qp.getZxid());

```java
public void commit(long zxid) {
        if (pendingTxns.size() == 0) {
            LOG.warn("Committing " + Long.toHexString(zxid)
                    + " without seeing txn");
            return;
        }
        long firstElementZxid = pendingTxns.element().zxid;
        if (firstElementZxid != zxid) {
            LOG.error("Committing zxid 0x" + Long.toHexString(zxid)
                    + " but next pending txn 0x"
                    + Long.toHexString(firstElementZxid));
            System.exit(12);
        }
        Request request = pendingTxns.remove();
        commitProcessor.commit(request);
    }
```

### 18.2.7、Leader.lead()应答 Follower
由于 public class LearnerHandler extends ZooKeeperThread{}，说明 LearnerHandler 是一个线程。所以 fh.start()执行的是 LearnerHandler 中的 run()方法。

```java
	public void run() {
		......
	 //
	 LOG.debug("Sending UPTODATE message to " + sid); 
	 queuedPackets.add(new QuorumPacket(Leader.UPTODATE, -1, null, null));
	 while (true) {
		......
	 } 
}
```
# 19、服务端 Leader 启动
## 19.1、服务端Leader启动源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/beb2b0f141934b848a855892e1a8d9e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
Ctrl + n 全局查找 Leader，然后 ctrl + f 查找 lead
Leader.java

```java
void lead() throws IOException, InterruptedException {
... ...
// 启动 zookeeper 服务
 startZkServer();
... ...
}
```
startZkServer();
```java
 private synchronized void startZkServer() {
        // Update lastCommitted and Db's zxid to a value representing the new epoch
        lastCommitted = zk.getZxid();
        LOG.info("Have quorum of supporters, sids: [ "
                + newLeaderProposal.ackSetsToString()
                + " ]; starting up and setting last processed zxid: 0x{}",
                Long.toHexString(zk.getZxid()));
        
        /*
         * ZOOKEEPER-1324. the leader sends the new config it must complete
         *  to others inside a NEWLEADER message (see LearnerHandler where
         *  the NEWLEADER message is constructed), and once it has enough
         *  acks we must execute the following code so that it applies the
         *  config to itself.
         */
        QuorumVerifier newQV = self.getLastSeenQuorumVerifier();
        
        Long designatedLeader = getDesignatedLeader(newLeaderProposal, zk.getZxid());                                         

        self.processReconfig(newQV, designatedLeader, zk.getZxid(), true);
        if (designatedLeader != self.getId()) {
            allowedToCommit = false;
        }
        // 启动 Zookeeper
        zk.startup();
        ......
    }
```
 zk.startup();
```java
    @Override
    public synchronized void startup() {
        super.startup();
        if (containerManager != null) {
            containerManager.start();
        }
    }
```
 super.startup();
```java
 public synchronized void startup() {
        if (sessionTracker == null) {
            createSessionTracker();
        }
        startSessionTracker();
        setupRequestProcessors();

        registerJMX();

        setState(State.RUNNING);
        notifyAll();
    }
```
setupRequestProcessors();

```java
 protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor syncProcessor = new SyncRequestProcessor(this,
                finalProcessor);
        ((SyncRequestProcessor)syncProcessor).start();
        firstProcessor = new PrepRequestProcessor(this, syncProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
```
点击 PrepRequestProcessor，并查找它的 run 方法

```java
public void run() {
        try {
            while (true) {
                Request request = submittedRequests.take();
                long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
                if (request.type == OpCode.ping) {
                    traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
                }
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
                }
                if (Request.requestOfDeath == request) {
                    break;
                }
                pRequest(request);
            }
        } catch (RequestProcessorException e) {
            if (e.getCause() instanceof XidRolloverException) {
                LOG.info(e.getCause().getMessage());
            }
            handleException(this.getName(), e);
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("PrepRequestProcessor exited loop!");
    }
```
pRequest(request);

```java
 protected void pRequest(Request request) throws RequestProcessorException {
        // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
        // request.type + " id = 0x" + Long.toHexString(request.sessionId));
        request.setHdr(null);
        request.setTxn(null);

        try {
            switch (request.type) {
            case OpCode.createContainer:
            case OpCode.create:
            case OpCode.create2:
                CreateRequest create2Request = new CreateRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true);
                break;
            case OpCode.createTTL:
                CreateTTLRequest createTtlRequest = new CreateTTLRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, createTtlRequest, true);
                break;
            case OpCode.deleteContainer:
            case OpCode.delete:
                DeleteRequest deleteRequest = new DeleteRequest();
                pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
                break;
            case OpCode.setData:
                SetDataRequest setDataRequest = new SetDataRequest();                
                pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
                break;
            case OpCode.reconfig:
                ReconfigRequest reconfigRequest = new ReconfigRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request, reconfigRequest);
                pRequest2Txn(request.type, zks.getNextZxid(), request, reconfigRequest, true);
                break;
            case OpCode.setACL:
                SetACLRequest setAclRequest = new SetACLRequest();                
                pRequest2Txn(request.type, zks.getNextZxid(), request, setAclRequest, true);
                break;
            case OpCode.check:
                CheckVersionRequest checkRequest = new CheckVersionRequest();              
                pRequest2Txn(request.type, zks.getNextZxid(), request, checkRequest, true);
                break;
            case OpCode.multi:
                MultiTransactionRecord multiRequest = new MultiTransactionRecord();
                try {
                    ByteBufferInputStream.byteBuffer2Record(request.request, multiRequest);
                } catch(IOException e) {
                    request.setHdr(new TxnHeader(request.sessionId, request.cxid, zks.getNextZxid(),
                            Time.currentWallTime(), OpCode.multi));
                    throw e;
                }
                List<Txn> txns = new ArrayList<Txn>();
                //Each op in a multi-op must have the same zxid!
                long zxid = zks.getNextZxid();
                KeeperException ke = null;

                //Store off current pending change records in case we need to rollback
                Map<String, ChangeRecord> pendingChanges = getPendingChanges(multiRequest);

                for(Op op: multiRequest) {
                    Record subrequest = op.toRequestRecord();
                    int type;
                    Record txn;

                    /* If we've already failed one of the ops, don't bother
                     * trying the rest as we know it's going to fail and it
                     * would be confusing in the logfiles.
                     */
                    if (ke != null) {
                        type = OpCode.error;
                        txn = new ErrorTxn(Code.RUNTIMEINCONSISTENCY.intValue());
                    }

                    /* Prep the request and convert to a Txn */
                    else {
                        try {
                            pRequest2Txn(op.getType(), zxid, request, subrequest, false);
                            type = request.getHdr().getType();
                            txn = request.getTxn();
                        } catch (KeeperException e) {
                            ke = e;
                            type = OpCode.error;
                            txn = new ErrorTxn(e.code().intValue());

                            if (e.code().intValue() > Code.APIERROR.intValue()) {
                                LOG.info("Got user-level KeeperException when processing {} aborting" +
                                        " remaining multi ops. Error Path:{} Error:{}",
                                        request.toString(), e.getPath(), e.getMessage());
                            }

                            request.setException(e);

                            /* Rollback change records from failed multi-op */
                            rollbackPendingChanges(zxid, pendingChanges);
                        }
                    }

                    //FIXME: I don't want to have to serialize it here and then
                    //       immediately deserialize in next processor. But I'm
                    //       not sure how else to get the txn stored into our list.
                    ByteArrayOutputStream baos = new ByteArrayOutputStream();
                    BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                    txn.serialize(boa, "request") ;
                    ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());

                    txns.add(new Txn(type, bb.array()));
                }

                request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid,
                        Time.currentWallTime(), request.type));
                request.setTxn(new MultiTxn(txns));

                break;

            //create/close session don't require request record
            case OpCode.createSession:
            case OpCode.closeSession:
                if (!request.isLocalSession()) {
                    pRequest2Txn(request.type, zks.getNextZxid(), request,
                                 null, true);
                }
                break;

            //All the rest don't need to create a Txn - just verify session
            case OpCode.sync:
            case OpCode.exists:
            case OpCode.getData:
            case OpCode.getACL:
            case OpCode.getChildren:
            case OpCode.getChildren2:
            case OpCode.ping:
            case OpCode.setWatches:
            case OpCode.checkWatches:
            case OpCode.removeWatches:
                zks.sessionTracker.checkSession(request.sessionId,
                        request.getOwner());
                break;
            default:
                LOG.warn("unknown type " + request.type);
                break;
        ......
        request.zxid = zks.getZxid();
        nextProcessor.processRequest(request);
    }
```

# 20、服务端 Follower 启动
## 20.1、服务端 Follower启动源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/d4120172ad944f42abc4c94d0d5de3a6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
Ctrl + n 全局查找 Follower，然后 ctrl + f 查找 followLeader

```java
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {} {}", electionTimeTaken,
                QuorumPeer.FLE_TIME_UNIT);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
            QuorumServer leaderServer = findLeader();
            try {
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
                if (self.isReconfigStateChange())
                    throw new Exception("learned about role change");
                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                syncWithLeader(newEpochZxid);
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
               ......
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }
```
 readPacket(qp);
```java
void readPacket(QuorumPacket pp) throws IOException {
        synchronized (leaderIs) {
            leaderIs.readRecord(pp, "packet");
        }
        if (LOG.isTraceEnabled()) {
            final long traceMask =
                (pp.getType() == Leader.PING) ? ZooTrace.SERVER_PING_TRACE_MASK
                    : ZooTrace.SERVER_PACKET_TRACE_MASK;

            ZooTrace.logQuorumPacket(LOG, traceMask, 'i', pp);
        }
    }
```

processPacket(qp);

```java
 protected void processPacket(QuorumPacket qp) throws Exception{
        switch (qp.getType()) {
        case Leader.PING:            
            ping(qp);            
            break;
        case Leader.PROPOSAL:           
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            if (hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
            }
            lastQueued = hdr.getZxid();
            
            if (hdr.getType() == OpCode.reconfig){
               SetDataTxn setDataTxn = (SetDataTxn) txn;       
               QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));
               self.setLastSeenQuorumVerifier(qv, true);                               
            }
            
            fzk.logRequest(hdr, txn);
            break;
        case Leader.COMMIT:
            fzk.commit(qp.getZxid());
            break;
            
        case Leader.COMMITANDACTIVATE:
           // get the new configuration from the request
           Request request = fzk.pendingTxns.element();
           SetDataTxn setDataTxn = (SetDataTxn) request.getTxn();                                                                                                      
           QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData()));                                
 
           // get new designated leader from (current) leader's message
           ByteBuffer buffer = ByteBuffer.wrap(qp.getData());    
           long suggestedLeaderId = buffer.getLong();
            boolean majorChange = 
                   self.processReconfig(qv, suggestedLeaderId, qp.getZxid(), true);
           // commit (writes the new config to ZK tree (/zookeeper/config)                     
           fzk.commit(qp.getZxid());
            if (majorChange) {
               throw new Exception("changes proposed in reconfig");
           }
           break;
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Follower started");
            break;
        case Leader.REVALIDATE:
            revalidate(qp);
            break;
        case Leader.SYNC:
            fzk.sync();
            break;
        default:
            LOG.warn("Unknown packet type: {}", LearnerHandler.packetToString(qp));
            break;
        }
    }
```
