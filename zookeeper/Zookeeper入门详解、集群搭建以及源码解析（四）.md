# 16、Zookeeper服务端加载数据源码解析 
![在这里插入图片描述](https://img-blog.csdnimg.cn/cc7890ef15a64245b5b0cb69ccbd2a83.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
1. zk 中的数据模型，是一棵树，DataTree，每个节点，叫做 DataNode
2. zk 集群中的 DataTree 时刻保持状态同步
3. Zookeeper 集群中每个 zk 节点中，数据在内存和磁盘中都有一份完整的数据。
	- 内存数据：DataTree
	- 磁盘数据：快照文件 + 编辑日志

## 16.1、Zookeeper服务端加载数据源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/dc49ea3f7f76427f9db6b3270b543ef5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 16.2、冷启动数据恢复快照数据
1）  启动集群
QuorumPeerMain类
```java
public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (DatadirException e) {
            LOG.error("Unable to access datadir, exiting abnormally", e);
            System.err.println("Unable to access datadir, exiting abnormally");
            System.exit(3);
        } catch (AdminServerException e) {
            LOG.error("Unable to start AdminServer, exiting abnormally", e);
            System.err.println("Unable to start AdminServer, exiting abnormally");
            System.exit(4);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
```
initializeAndRun

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
        	// 集群启动
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```
runFromConfig方法

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
		......
		quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
        quorumPeer.initialize();
   		quorumPeer.start();
        quorumPeer.join();
```


2）  冷启动恢复数据

start() 
```java
@Override
    public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
         }
         // 冷启动数据恢复
        loadDataBase();
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        startLeaderElection();
        super.start();
    }
```

```java
private void loadDataBase() {
try {
	// 加载磁盘数据到内存，恢复 DataTree
	// zk 的操作分两种：事务操作和非事务操作
	// 事务操作：zk.cteate()；都会被分配一个全局唯一的 zxid，zxid 组成：64 位：
	（前 32 位：epoch 每个 leader 任期的代号；后 32 位：txid 为事务 id）
	// 非事务操作：zk.getData()
	// 数据恢复过程：
	// （1）从快照文件中恢复大部分数据，并得到一个 lastProcessZXid
	// （2）再从编辑日志中执行 replay，执行到最后一条日志并更新 lastProcessZXid
	// （3）最终得到，datatree 和 lastProcessZXid，表示数据恢复完成
	zkDb.loadDataBase();
	// load the epochs
	long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
	long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
	try {
		currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
	......
```
loadDataBase()
```java
    public long loadDataBase() throws IOException {
        long zxid = snapLog.restore(dataTree, sessionsWithTimeouts, commitProposalPlaybackListener);
        initialized = true;
        return zxid;
    }
```
restore()

```java
public long restore(DataTree dt, Map<Long, Integer> sessions,
	PlayBackListener listener) throws IOException {
	// 恢复 快照文件到 数据到 DataTree
	long deserializeResult = snapLog.deserialize(dt, sessions);
	FileTxnLog txnLog = new FileTxnLog(dataDir);
	RestoreFinalizer finalizer = () -> {
	// 恢复 编辑日志到 数据到 DataTree
	long highestZxid = fastForwardFromEdits(dt, sessions, listener);
	return highestZxid;
if (-1L == deserializeResult) {
 /* this means that we couldn't find any snapshot, so we need to
             * initialize an empty database (reported in ZOOKEEPER-2325) */
            if (txnLog.getLastLoggedZxid() != -1) {
                // ZOOKEEPER-3056: provides an escape hatch for users upgrading
                // from old versions of zookeeper (3.4.x, pre 3.5.3).
                if (!trustEmptySnapshot) {
                    throw new IOException(EMPTY_SNAPSHOT_WARNING + "Something is broken!");
                } else {
                    LOG.warn("{}This should only be allowed during upgrading.", EMPTY_SNAPSHOT_WARNING);
                    return finalizer.run();
                }
            }
            /* TODO: (br33d) we should either put a ConcurrentHashMap on restore()
             *       or use Map on save() */
            save(dt, (ConcurrentHashMap<Long, Integer>)sessions);
            /* return a zxid of zero, since we the database is empty */
            return 0;
}	
```
deserialize() 
ctrl + alt +B 查找 deserialize 实现类 FileSnap.java
```java
public long deserialize(DataTree dt, Map<Long, Integer> sessions)
            throws IOException {
        // we run through 100 snapshots (not all of them)
        // if we cannot get it running within 100 snapshots
        // we should  give up
        List<File> snapList = findNValidSnapshots(100);
        if (snapList.size() == 0) {
            return -1L;
        }
        File snap = null;
        boolean foundValid = false;
        // 依次遍历每一个快照的数据
        for (int i = 0, snapListSize = snapList.size(); i < snapListSize; i++) {
            snap = snapList.get(i);
            LOG.info("Reading snapshot " + snap);
            try (
            // 反序列化 环境准备
            InputStream snapIS = new BufferedInputStream(new FileInputStream(snap));
                CheckedInputStream crcIn = new CheckedInputStream(snapIS, new Adler32())) {
                InputArchive ia = BinaryInputArchive.getArchive(crcIn);
                // 反序列到 化，恢复数据到 DataTree
                deserialize(dt, sessions, ia);
                long checkSum = crcIn.getChecksum().getValue();
                long val = ia.readLong("val");
                if (val != checkSum) {
                    throw new IOException("CRC corruption in snapshot :  " + snap);
                }
                foundValid = true;
                break;
            } catch (IOException e) {
                LOG.warn("problem reading snap file " + snap, e);
            }
        }
        if (!foundValid) {
            throw new IOException("Not able to find valid snapshots in " + snapDir);
        }
        dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
        return dt.lastProcessedZxid;
    }
```
deserialize（）
```java
 public void deserialize(DataTree dt, Map<Long, Integer> sessions,
            InputArchive ia) throws IOException {
        FileHeader header = new FileHeader();
        header.deserialize(ia, "fileheader");
        if (header.getMagic() != SNAP_MAGIC) {
            throw new IOException("mismatching magic headers "
                    + header.getMagic() +
                    " !=  " + FileSnap.SNAP_MAGIC);
        }
        // 恢复快照数据到 DataTree
        SerializeUtils.deserializeSnapshot(dt,ia,sessions);
    }
```

deserializeSnapshot（）
```java
public static void deserializeSnapshot(DataTree dt,InputArchive ia,
            Map<Long, Integer> sessions) throws IOException {
        int count = ia.readInt("count");
        while (count > 0) {
            long id = ia.readLong("id");
            int to = ia.readInt("timeout");
            sessions.put(id, to);
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG, ZooTrace.SESSION_TRACE_MASK,
                        "loadData --- session in archive: " + id
                        + " with timeout: " + to);
            }
            count--;
        }
         // 恢复快照数据到 DataTree
        dt.deserialize(ia, "tree");
    }
```

```java
public void deserialize(InputArchive ia, String tag) throws IOException {
	aclCache.deserialize(ia);
	nodes.clear();
	pTrie.clear();
	String path = ia.readString("path");
	// 从快照中恢复每一个 datanode 节点数据到 DataTree
	while (!"/".equals(path)) {
		// 每次循环创建一个节点对象
		DataNode node = new DataNode();
		ia.readRecord(node, "node");
		// 将 将 DataNode 恢复到 DataTree
		nodes.put(path, node);
		synchronized (node) {
			aclCache.addUsage(node.acl);
		}
		int lastSlash = path.lastIndexOf('/');
		if (lastSlash == -1) {
		root = node;
	} else {
		// 处理父节点
		String parentPath = path.substring(0, lastSlash);
		DataNode parent = nodes.get(parentPath);
	if (parent == null) {
		throw new IOException("Invalid Datatree, unable to find parent " + parentPath + " of path " + path);
	}
		// 处理子节点
		parent.addChild(path.substring(lastSlash + 1));
		// 处理临时节点和永久节点
		long eowner = node.stat.getEphemeralOwner();
		EphemeralType ephemeralType = EphemeralType.get(eowner);
	if (ephemeralType == EphemeralType.CONTAINER) {
		containers.add(path);
	} else if (ephemeralType == EphemeralType.TTL) {
		ttls.add(path);
	} else if (eowner != 0) {
		HashSet<String> list = ephemerals.get(eowner);
		if (list == null) {
		list = new HashSet<String>();
		ephemerals.put(eowner, list);
	}
		list.add(path);
	}
	}
	path = ia.readString("path");
	}
	nodes.put("/", root);
	// we are done with deserializing the
	// the datatree
	// update the quotas - create path trie
	// and also update the stat nodes
	setupQuota();
	aclCache.purgeUnused();
}
```

## 16.3、冷启动数据恢复编辑日志
restore()
```java
public long restore(DataTree dt, Map<Long, Integer> sessions,
	PlayBackListener listener) throws IOException {
		// 恢复快照文件数据到 DataTree
		long deserializeResult = snapLog.deserialize(dt, sessions);
		FileTxnLog txnLog = new FileTxnLog(dataDir);
		RestoreFinalizer finalizer = () -> {
		// 恢复 编辑日志到 数据到 DataTree
		long highestZxid = fastForwardFromEdits(dt, sessions, listener);
		return highestZxid;
	};
	......
	return finalizer.run();
}
```
fastForwardFromEdits()
```java
public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                     PlayBackListener listener) throws IOException {
		// 在此之前，已经从快照文件中恢复了大部分数据，接下来只需从快照的 zxid + 1位置开始恢复
        TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
        // 快照中最大的 zxid ，在执行编辑日志时，这个值会不断更新，直到所有操作执行完 
        long highestZxid = dt.lastProcessedZxid;
        TxnHeader hdr;
        try {
        	// 从 lastProcessedZxid 事务编号器开始，不断的从编辑日志中恢复剩下的还没有恢复的数据
            while (true) {
                // iterator points to
                // the first valid txn when initialized
                // 获取事务头信息（有 zxid ）
                hdr = itr.getHeader();
                if (hdr == null) {
                    //empty logs
                    return dt.lastProcessedZxid;
                }
                if (hdr.getZxid() < highestZxid && highestZxid != 0) {
                    LOG.error("{}(highestZxid) > {}(next log) for type {}",
                            highestZxid, hdr.getZxid(), hdr.getType());
                } else {
                    highestZxid = hdr.getZxid();
                }
                try {
                // 根据编辑日志恢复数据到 DataTree， ， 每务 执行一次，对应的事务 id，highestZxid + 1
                    processTransaction(hdr,dt,sessions, itr.getTxn());
                } catch(KeeperException.NoNodeException e) {
                   throw new IOException("Failed to process transaction type: " +
                         hdr.getType() + " error: " + e.getMessage(), e);
                }
                listener.onTxnLoaded(hdr, itr.getTxn());
                if (!itr.next())
                    break;
            }
        } finally {
            if (itr != null) {
                itr.close();
            }
        }
        return highestZxid;
    }
```
processTransaction()

```java
public void processTransaction(TxnHeader hdr,DataTree dt, Map<Long, Integer> sessions, Record txn) throws  KeeperException.NoNodeException {
        ProcessTxnResult rc;
        switch (hdr.getType()) {
        case OpCode.createSession:
            sessions.put(hdr.getClientId(),
                    ((CreateSessionTxn) txn).getTimeOut());
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,ZooTrace.SESSION_TRACE_MASK,"playLog --- create session in log: 0x"
                                + Long.toHexString(hdr.getClientId())
                                + " with timeout: "
                                + ((CreateSessionTxn) txn).getTimeOut());
            }
            // give dataTree a chance to sync its lastProcessedZxid
            // 创建节点、删除节点和其他的各种事务操作等
            rc = dt.processTxn(hdr, txn);
            break;
        case OpCode.closeSession:
            sessions.remove(hdr.getClientId());
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,ZooTrace.SESSION_TRACE_MASK,"playLog --- close session in log: 0x"
                                + Long.toHexString(hdr.getClientId()));
            }
            rc = dt.processTxn(hdr, txn);
            break;
        default:
            rc = dt.processTxn(hdr, txn);
        }

        /**
         * Snapshots are lazily created. So when a snapshot is in progress,
         * there is a chance for later transactions to make into the
         * snapshot. Then when the snapshot is restored, NONODE/NODEEXISTS
         * errors could occur. It should be safe to ignore these.
         */
        if (rc.err != Code.OK.intValue()) {
            LOG.debug("Ignoring processTxn failure hdr: {}, error: {}, path: {}",hdr.getType(), rc.err, rc.path);
        }
    }
```
processTxn（）

```java
public ProcessTxnResult processTxn(TxnHeader header, Record txn, boolean isSubTxn)
    {
        ProcessTxnResult rc = new ProcessTxnResult();

        try {
            rc.clientId = header.getClientId();
            rc.cxid = header.getCxid();
            rc.zxid = header.getZxid();
            rc.type = header.getType();
            rc.err = 0;
            rc.multiResult = null;
            switch (header.getType()) {
                case OpCode.create:
                    CreateTxn createTxn = (CreateTxn) txn;
                    rc.path = createTxn.getPath();
                    createNode(
                            createTxn.getPath(),
                            createTxn.getData(),
                            createTxn.getAcl(),
                            createTxn.getEphemeral() ? header.getClientId() : 0,
                            createTxn.getParentCVersion(),
                            header.getZxid(), header.getTime(), null);
                    break;
                case OpCode.create2:
                    CreateTxn create2Txn = (CreateTxn) txn;
                    rc.path = create2Txn.getPath();
                    Stat stat = new Stat();
                    createNode(
                            create2Txn.getPath(),
                            create2Txn.getData(),
                            create2Txn.getAcl(),
                            create2Txn.getEphemeral() ? header.getClientId() : 0,
                            create2Txn.getParentCVersion(),
                            header.getZxid(), header.getTime(), stat);
                    rc.stat = stat;
                    break;
                case OpCode.createTTL:
                    CreateTTLTxn createTtlTxn = (CreateTTLTxn) txn;
                    rc.path = createTtlTxn.getPath();
                    stat = new Stat();
                    createNode(
                            createTtlTxn.getPath(),
                            createTtlTxn.getData(),
                            createTtlTxn.getAcl(),
                            EphemeralType.TTL.toEphemeralOwner(createTtlTxn.getTtl()),
                            createTtlTxn.getParentCVersion(),
                            header.getZxid(), header.getTime(), stat);
                    rc.stat = stat;
                    ......
```
# 17、Zookeeper选举源码解析
## 17.1、Zookeeper选举工作机制
### 17.1.1、第一次启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/aa27daaf66d54df1838dd0fe0827c6fe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 17.1.2、非第一次启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/e165ea14e5834ec38a7cc4997cc3444c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 17.2、Zookeeper选举源码解析图解
选举源码解析组件图
![在这里插入图片描述](https://img-blog.csdnimg.cn/27536939f65a4575a4c6d2ebe00a6b90.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
选举源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/984c88ed672742e7a576e942a9819c47.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
QuorumPeerMain类
```java
public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (DatadirException e) {
            LOG.error("Unable to access datadir, exiting abnormally", e);
            System.err.println("Unable to access datadir, exiting abnormally");
            System.exit(3);
        } catch (AdminServerException e) {
            LOG.error("Unable to start AdminServer, exiting abnormally", e);
            System.err.println("Unable to start AdminServer, exiting abnormally");
            System.exit(4);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
```
initializeAndRun方法

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

          quorumPeer = getQuorumPeer();
          quorumPeer.setTxnFactory(new FileTxnSnapLog(
                      config.getDataLogDir(),
                      config.getDataDir()));
          quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
          ......
          quorumPeer.initialize();
          quorumPeer.start();
          quorumPeer.join();
```
start方法

```java
 @Override
    public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
         }
        loadDataBase();
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        // 开始领导选举
        startLeaderElection();
        super.start();
    }
```
startLeaderElection()

```java
 synchronized public void startLeaderElection() {
       try {
           if (getPeerState() == ServerState.LOOKING) {
           		// 创建选票
				// （1 ）选票组件：epoch （leader 的任期代号）、zxid （某个 leader 当选期间执行的事务编号）、myid （serverid ）
				// （2 ）开始选票时，都是先投自己
               currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
           }
       } catch(IOException e) {
           RuntimeException re = new RuntimeException(e.getMessage());
           re.setStackTrace(e.getStackTrace());
           throw re;
       }
	      // if (!getView().containsKey(myid)) {
	      //      throw new RuntimeException("My id " + myid + " not in the peer list");
	      // }
        if (electionType == 0) {
            try {
                udpSocket = new DatagramSocket(getQuorumAddress().getPort());
                responder = new ResponderThread();
                responder.start();
            } catch (SocketException e) {
                throw new RuntimeException(e);
            }
        }
        // 创建选举算法实例
        this.electionAlg = createElectionAlgorithm(electionType);
    }
```
createElectionAlgorithm()

```java
protected Election createElectionAlgorithm(int electionAlgorithm){
        Election le=null;

        //TODO: use a factory rather than a switch
        switch (electionAlgorithm) {
        case 0:
            le = new LeaderElection(this);
            break;
        case 1:
            le = new AuthFastLeaderElection(this);
            break;
        case 2:
            le = new AuthFastLeaderElection(this, true);
            break;
        case 3:
       	 // 1 创建 QuorumCnxnManager，负责选举过程中的所有网络通信
            QuorumCnxManager qcm = createCnxnManager();
            QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
            if (oldQcm != null) {
                LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
                oldQcm.halt();
            }
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
           		 // 2 启动监听线程
                listener.start();
                // 3 准备开始选举
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
                fle.start();
                le = fle;
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
        }
        return le;
    }
```
createCnxnManager()

```java
public QuorumCnxManager(QuorumPeer self,
                            final long mySid,
                            Map<Long,QuorumPeer.QuorumServer> view,
                            QuorumAuthServer authServer,
                            QuorumAuthLearner authLearner,
                            int socketTimeout,
                            boolean listenOnAllIPs,
                            int quorumCnxnThreadsSize,
                            boolean quorumSaslAuthEnabled) {
        // 创建各种队列
        this.recvQueue = new ArrayBlockingQueue<Message>(RECV_CAPACITY);
        this.queueSendMap = new ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>>();
        this.senderWorkerMap = new ConcurrentHashMap<Long, SendWorker>();
        this.lastMessageSent = new ConcurrentHashMap<Long, ByteBuffer>();

        String cnxToValue = System.getProperty("zookeeper.cnxTimeout");
        if(cnxToValue != null){
            this.cnxTO = Integer.parseInt(cnxToValue);
        }

        this.self = self;

        this.mySid = mySid;
        this.socketTimeout = socketTimeout;
        this.view = view;
        this.listenOnAllIPs = listenOnAllIPs;

        initializeAuth(mySid, authServer, authLearner, quorumCnxnThreadsSize,
                quorumSaslAuthEnabled);

        // Starts listener thread that waits for connection requests
        listener = new Listener();
        listener.setName("QuorumPeerListener");
    }
```
listener.start()，点击 QuorumCnxManager.Listener，找到对应的 run 方法
```java
public void run() {
 int numRetries = 0;
 InetSocketAddress addr;
 Socket client = null;
 Exception exitException = null;
 ......
 LOG.info("My election bind port: " + addr.toString());
 setName(addr.toString());
// 绑定服务器地址
 ss.bind(addr);
// 死循环
 while (!shutdown) {
   try {
     client = ss.accept();
     setSockOpts(client);
    .....
   }
 }
```
FastLeaderElection(this, qcm)

```java
 private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;
		// 初始化队列和信息
        sendqueue = new LinkedBlockingQueue<ToSend>();
        recvqueue = new LinkedBlockingQueue<Notification>();
        this.messenger = new Messenger(manager);
    }
```


## 17.3、zookeeper选举执行源码
![在这里插入图片描述](https://img-blog.csdnimg.cn/d7b32e9704724c3f89d1404bd4e92b7a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
QuorumPeer.java
```java
public synchronized void start() {
	 if (!getView().containsKey(myid)) {
	 throw new RuntimeException("My id " + myid + " not in the peer list");
	 }
	// 冷启动数据恢复
	 loadDataBase();
	 startServerCnxnFactory();
	 try {
	// 启动通信工厂实例对象
	 adminServer.start();
	 } catch (AdminServerException e) {
	 LOG.warn("Problem starting AdminServer", e);
	 System.out.println(e);
	 }
	// 准备选举环境
	 startLeaderElection();
	// 执行选举
	 super.start();
}
```

### 17.3.1、执行 super.start()
就相当于执行 QuorumPeer.java 类中的 run()方法当 Zookeeper 启动后，首先都是 Looking 状态，通过选举，让其中一台服务器成为 Leader，其他的服务器成为 Follower。

```java
{
        updateThreadName();

        LOG.debug("Starting quorum peer");
        try {
            jmxQuorumBean = new QuorumBean(this);
            MBeanRegistry.getInstance().register(jmxQuorumBean, null);
            for(QuorumServer s: getView().values()){
                ZKMBeanInfo p;
                if (getId() == s.id) {
                    p = jmxLocalPeerBean = new LocalPeerBean(this);
                    try {
                        MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                        jmxLocalPeerBean = null;
                    }
                } else {
                    RemotePeerBean rBean = new RemotePeerBean(this, s);
                    try {
                        MBeanRegistry.getInstance().register(rBean, jmxQuorumBean);
                        jmxRemotePeerBean.put(s.id, rBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                    }
                }
            }
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            jmxQuorumBean = null;
        }

        try {
            /*
             * Main loop
             */
            while (running) {
                switch (getPeerState()) {
                case LOOKING:
                    LOG.info("LOOKING");
                    if (Boolean.getBoolean("readonlymode.enabled")) {
                        LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                        // Create read-only server but don't start it immediately
                        final ReadOnlyZooKeeperServer roZk =
                            new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);
    
                        // Instead of starting roZk immediately, wait some grace
                        // period before we decide we're partitioned.
                        //
                        // Thread is used here because otherwise it would require
                        // changes in each of election strategy classes which is
                        // unnecessary code coupling.
                        Thread roZkMgr = new Thread() {
                            public void run() {
                                try {
                                    // lower-bound grace period to 2 secs
                                    sleep(Math.max(2000, tickTime));
                                    if (ServerState.LOOKING.equals(getPeerState())) {
                                        roZk.startup();
                                    }
                                } catch (InterruptedException e) {
                                    LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                                } catch (Exception e) {
                                    LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                                }
                            }
                        };
                        try {
                            roZkMgr.start();
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                            // 进行选举，选举结束，返回最终成为 Leader 胜选的那张选票
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        } finally {
                            // If the thread is in the the grace period, interrupt
                            // to come out of waiting.
                            roZkMgr.interrupt();
                            roZk.shutdown();
                        }
                    } else {
                        try {
                           reconfigFlagClear();
                            if (shuttingDownLE) {
                               shuttingDownLE = false;
                               startLeaderElection();
                               }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                            setPeerState(ServerState.LOOKING);
                        }                        
                    }
                    break;
                case OBSERVING:
                    try {
                        LOG.info("OBSERVING");
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e );
                    } finally {
                        observer.shutdown();
                        setObserver(null);  
                       updateServerState();
                    }
                    break;
                case FOLLOWING:
                    try {
                       LOG.info("FOLLOWING");
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) {
                       LOG.warn("Unexpected exception",e);
                    } finally {
                       follower.shutdown();
                       setFollower(null);
                       updateServerState();
                    }
                    break;
                case LEADING:
                    LOG.info("LEADING");
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        if (leader != null) {
                            leader.shutdown("Forcing shutdown");
                            setLeader(null);
                        }
                        updateServerState();
                    }
                    break;
                }
                start_fle = Time.currentElapsedTime();
            }
        } finally {
            LOG.warn("QuorumPeer main thread exited");
            MBeanRegistry instance = MBeanRegistry.getInstance();
            instance.unregister(jmxQuorumBean);
            instance.unregister(jmxLocalPeerBean);

            for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {
                instance.unregister(remotePeerBean);
            }

            jmxQuorumBean = null;
            jmxLocalPeerBean = null;
            jmxRemotePeerBean = null;
        }
    }
```
### 17.3.2、setCurrentVote(makeLEStrategy().lookForLeader());
ctrl+alt+b 点击 lookForLeader()的实现类 FastLeaderElection.java
 
 

```java
public Vote lookForLeader() throws InterruptedException {
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
           self.start_fle = Time.currentElapsedTime();
        }
        try {
        	// 正常启动中，所有其他服务器，都会给我发送一个投票
			// 保存每一个服务器的最新合法有效的投票
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();
            // 存储合法选举之外的投票结果
            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
            // 一次选举的最大等待时间，默认值是 0.2s
            int notTimeout = finalizeWait;
            
            // 每发起一轮选举，logicalclock++
			// 在没有合法的 epoch 数据之前，都使用逻辑时钟代替
			// 选举 leader 的规则：依次比较 epoch（任期） zxid（事务 id） serverid （myid） 谁大谁当选 leader
            synchronized(this){
			// 更新逻辑时钟，每进行一次选举，都需要更新逻辑时钟
                logicalclock.incrementAndGet();
                // 更新选票（serverid， zxid, epoch），
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            // 广播选票，把自己的选票发给其他服务器
            sendNotifications();

            /*
             * Loop in which we exchange notifications until we find a leader
             */
			// 一轮一轮的选举直到选举成功
            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                ......
            return null;
        } finally {
            ......
        }
    }
```
### 17.3.3、 sendNotifications();
广播选票，把自己的选票发给其他服务器
```java
private void sendNotifications() {
// 遍历投票参与者，给每台服务器发送选票
        for (long sid : self.getCurrentAndNextConfigVoters()) {
            QuorumVerifier qv = self.getQuorumVerifier();
            // 创建发送选票
            ToSend notmsg = new ToSend(ToSend.mType.notification,
                    proposedLeader,
                    proposedZxid,
                    logicalclock.get(),
                    QuorumPeer.ServerState.LOOKING,
                    sid,
                    proposedEpoch, qv.toString().getBytes());
            if(LOG.isDebugEnabled()){
                LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                      Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock.get())  +
                      " (n.round), " + sid + " (recipient), " + self.getId() +
                      " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
            }
            // 把发送选票放入发送队列
            sendqueue.offer(notmsg);
        }
    }
```
### 17.3.4、在 FastLeaderElection.java 类中查找 WorkerSender 线程

```java
{
            volatile boolean stop;
            QuorumCnxManager manager;

            WorkerSender(QuorumCnxManager manager){
                super("WorkerSender");
                this.stop = false;
                this.manager = manager;
            }

            public void run() {
                while (!stop) {
                    try {
                   		 // 队列阻塞，时刻准备接收要发送的选票
                        ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                        if(m == null) continue;
						// 处理要发送的选票
                        process(m);
                    } catch (InterruptedException e) {
                        break;
                    }
                }
                LOG.info("WorkerSender is down");
            }

            /**
             * Called by run() once there is a new message to send.
             *
             * @param m     message to send
             */
            void process(ToSend m) {
                ByteBuffer requestBuffer = buildMsg(m.state.ordinal(),
                                                    m.leader,
                                                    m.zxid,
                                                    m.electionEpoch,
                                                    m.peerEpoch,
                                                    m.configData);
				// 发送选票
                manager.toSend(m.sid, requestBuffer);

            }
        }
```
toSend(m.sid, requestBuffer)
```java
 public void toSend(Long sid, ByteBuffer b) {
        /*
         * If sending message to myself, then simply enqueue it (loopback).
         */
         // 判断如果是发给自己的消息，直接进入自己的 RecvQueue
        if (this.mySid == sid) {
             b.position(0);
             addToRecvQueue(new Message(b.duplicate(), sid));
            /*
             * Otherwise send to the corresponding thread to send.
             */
        } else {
             /*
              * Start a new connection if doesn't have one already.
              */
               // 如果是发给其他服务器，创建对应的发送队列或者获取已经存在的发送队列
			   // ，并把要发送的消息放入该队列
             ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(
                SEND_CAPACITY);
             ArrayBlockingQueue<ByteBuffer> oldq = queueSendMap.putIfAbsent(sid, bq);
             if (oldq != null) {
                 addToSendQueue(oldq, b);
             } else {
                 addToSendQueue(bq, b);
             }
             // 将选票发送出去
             connectOne(sid);
        }
    }
```
### 17.3.5、addToRecvQueue(new Message(b.duplicate(), sid));
如果数据是发送给自己的，添加到自己的接收队列

```java
public void addToRecvQueue(Message msg) {
        synchronized(recvQLock) {
            if (recvQueue.remainingCapacity() == 0) {
                try {
                    recvQueue.remove();
                } catch (NoSuchElementException ne) {
                    // element could be removed by poll()
                     LOG.debug("Trying to remove from an empty " +
                         "recvQueue. Ignoring exception " + ne);
                }
            }
            try {
            // 将发送给自己的选票添加到 recvQueue 队列
                recvQueue.add(msg);
            } catch (IllegalStateException ie) {
                // This should never happen
                LOG.error("Unable to insert element in the recvQueue " + ie);
            }
        }
    }
```
###  17.3.6、addToSendQueue(oldq, b);
数据添加到发送队列

```java
private void addToSendQueue(ArrayBlockingQueue<ByteBuffer> queue,
          ByteBuffer buffer) {
        if (queue.remainingCapacity() == 0) {
            try {
                queue.remove();
            } catch (NoSuchElementException ne) {
                // element could be removed by poll()
                LOG.debug("Trying to remove from an empty " +
                        "Queue. Ignoring exception " + ne);
            }
        }
        try {
        // 将要发送的消息添加到发送队列
            queue.add(buffer);
        } catch (IllegalStateException ie) {
            // This should never happen
            LOG.error("Unable to insert an element in the queue " + ie);
        }
    }
```
###  17.3.7、 connectOne(long sid)
与要发送的服务器节点建立通信连接
```java
 synchronized void connectOne(long sid){
        if (senderWorkerMap.get(sid) != null) {
            LOG.debug("There is a connection already for server " + sid);
            return;
        }
        synchronized (self.QV_LOCK) {
            boolean knownId = false;
            // Resolve hostname for the remote server before attempting to
            // connect in case the underlying ip address has changed.
            self.recreateSocketAddresses(sid);
            Map<Long, QuorumPeer.QuorumServer> lastCommittedView = self.getView();
            QuorumVerifier lastSeenQV = self.getLastSeenQuorumVerifier();
            Map<Long, QuorumPeer.QuorumServer> lastProposedView = lastSeenQV.getAllMembers();
            if (lastCommittedView.containsKey(sid)) {
                knownId = true;
                if (connectOne(sid, lastCommittedView.get(sid).electionAddr))
                    return;
            }
            if (lastSeenQV != null && lastProposedView.containsKey(sid)
                    && (!knownId || (lastProposedView.get(sid).electionAddr !=
                    lastCommittedView.get(sid).electionAddr))) {
                knownId = true;
                if (connectOne(sid, lastProposedView.get(sid).electionAddr))
                    return;
            }
            if (!knownId) {
                LOG.warn("Invalid server id: " + sid);
                return;
            }
        }
    }
```
### 17.3.8、startConnection(sock, sid);
创建并启动发送器线程和接收器线程

```java
private boolean startConnection(Socket sock, Long sid)
            throws IOException {
        DataOutputStream dout = null;
        DataInputStream din = null;
        try {
            // Use BufferedOutputStream to reduce the number of IP packets. This is
            // important for x-DC scenarios.
            // 通过输出流，向服务器发送数据
            BufferedOutputStream buf = new BufferedOutputStream(sock.getOutputStream());
            dout = new DataOutputStream(buf);

            // Sending id and challenge
            // represents protocol version (in other words - message type)
            dout.writeLong(PROTOCOL_VERSION);
            dout.writeLong(self.getId());
            String addr = formatInetAddr(self.getElectionAddress());
            byte[] addr_bytes = addr.getBytes();
            dout.writeInt(addr_bytes.length);
            dout.write(addr_bytes);
            dout.flush();
			// 通过输入流读取对方发送过来的选票
            din = new DataInputStream(
                    new BufferedInputStream(sock.getInputStream()));
        } catch (IOException e) {
            LOG.warn("Ignoring exception reading or writing challenge: ", e);
            closeSocket(sock);
            return false;
        }

        // authenticate learner
        QuorumPeer.QuorumServer qps = self.getVotingView().get(sid);
        if (qps != null) {
            // TODO - investigate why reconfig makes qps null.
            authLearner.authenticate(sock, qps.hostname);
        }

        // If lost the challenge, then drop the new connection
        // 如果对方的 id 比我的大，我是没有资格给对方发送连接请求的，直接关闭自己的客户端
        if (sid > self.getId()) {
            LOG.info("Have smaller server identifier, so dropping the " +
                    "connection: (" + sid + ", " + self.getId() + ")");
            closeSocket(sock);
            // Otherwise proceed with the connection
        } else {
       		 // 初始化，发送器 和 接收器
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);

            if(vsw != null)
                vsw.finish();

            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(
                    SEND_CAPACITY));
			// 启动发送器线程和接收器线程
            sw.start();
            rw.start();

            return true;

        }
        return false;
    }
```

### 17.3.9、sw.start();
点击 SendWorker，并查找该类下的 run 方法

```java
 public void run() {
            threadCnt.incrementAndGet();
            try {
                ArrayBlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
                if (bq == null || isSendQueueEmpty(bq)) {
                   ByteBuffer b = lastMessageSent.get(sid);
                   if (b != null) {
                       LOG.debug("Attempting to send lastMessage to sid=" + sid);
                       send(b);
                   }
                }
            } catch (IOException e) {
                LOG.error("Failed to send last message. Shutting down thread.", e);
                this.finish();
            }

            try {
            	// 只要连接没有断开
                while (running && !shutdown && sock != null) {

                    ByteBuffer b = null;
                    try {
                        ArrayBlockingQueue<ByteBuffer> bq = queueSendMap
                                .get(sid);
                        if (bq != null) {
                        // 不断从发送队列 SendQueue 中，获取发送消息，并执行发送
                            b = pollSendQueue(bq, 1000, TimeUnit.MILLISECONDS);
                        } else {
                            LOG.error("No queue of incoming messages for " +
                                      "server " + sid);
                            break;
                        }

                        if(b != null){
                        // 更新对于 sid 这台服务器的最近一条消息
                            lastMessageSent.put(sid, b);
                            // 执行发送
                            send(b);
                        }
                    } catch (InterruptedException e) {
                        LOG.warn("Interrupted while waiting for message on queue",
                                e);
                    }
                }
            } catch (Exception e) {
                LOG.warn("Exception when using channel: for id " + sid
                         + " my id = " + QuorumCnxManager.this.mySid
                         + " error = " + e);
            }
            this.finish();
            LOG.warn("Send worker leaving thread " + " id " + sid + " my id = " + self.getId());
        }
    }
```
 send(b)
 

```java
synchronized void send(ByteBuffer b) throws IOException {
            byte[] msgBytes = new byte[b.capacity()];
            try {
                b.position(0);
                b.get(msgBytes);
            } catch (BufferUnderflowException be) {
                LOG.error("BufferUnderflowException ", be);
                return;
            }
            // 输出流向外发送
            dout.writeInt(b.capacity());
            dout.write(b.array());
            dout.flush();
        }
```

### 17.3.10、 rw.start();
点击 RecvWorker，并查找该类下的 run 方法

```java
public void run() {
            threadCnt.incrementAndGet();
            try {
            // 只要连接没有断开
                while (running && !shutdown && sock != null) {
                    /**
                     * Reads the first int to determine the length of the
                     * message
                     */
                    int length = din.readInt();
                    if (length <= 0 || length > PACKETMAXSIZE) {
                        throw new IOException(
                                "Received packet with invalid packet: "
                                        + length);
                    }
                    /**
                     * Allocates a new ByteBuffer to receive the message
                     */
                    byte[] msgArray = new byte[length];
                    // 输入流接收消息
                    din.readFully(msgArray, 0, length);
                    ByteBuffer message = ByteBuffer.wrap(msgArray);
                    // 接收对方发送过来的选票
                    addToRecvQueue(new Message(message.duplicate(), sid));
                }
            } catch (Exception e) {
                LOG.warn("Connection broken for id " + sid + ", my id = "
                         + QuorumCnxManager.this.mySid + ", error = " , e);
            } finally {
                LOG.warn("Interrupting SendWorker");
                sw.finish();
                closeSocket(sock);
            }
        }
```
addToRecvQueue(Message msg) 
```java
public void addToRecvQueue(Message msg) {
        synchronized(recvQLock) {
            if (recvQueue.remainingCapacity() == 0) {
                try {
                    recvQueue.remove();
                } catch (NoSuchElementException ne) {
                    // element could be removed by poll()
                     LOG.debug("Trying to remove from an empty " +
                         "recvQueue. Ignoring exception " + ne);
                }
            }
            try {
            	// 将接收到的消息，放入接收消息队列
                recvQueue.add(msg);
            } catch (IllegalStateException ie) {
                // This should never happen
                LOG.error("Unable to insert element in the recvQueue " + ie);
            }
        }
    }
```
