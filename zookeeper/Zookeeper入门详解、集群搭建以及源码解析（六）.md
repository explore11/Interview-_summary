# 21、客户端启动
## 21.1、客户端启动源码解析流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/1428454811ae49659226787ecd15d48d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 21.1.1、创建 ZookeeperMain 
在 ZkCli.sh 启动 Zookeeper 时，会调用 ZooKeeperMain.java
Ctrl + n 查找 ZooKeeperMain，找到程序的入口 main()方法 

```java
public static void main(String args[]) throws CliException, IOException, InterruptedException
{ 
    ZooKeeperMain main = new ZooKeeperMain(args);    
     main.run(); 
 } 
```
ZooKeeperMain
```java
public ZooKeeperMain(String args[]) throws IOException, InterruptedException {
    cl.parseOptions(args);
    System.out.println("Connecting to " + cl.getOption("server"));
    connectToZK(cl.getOption("server"));
}
```
 connectToZK(cl.getOption("server"));

```java
protected void connectToZK(String newHost) throws InterruptedException, IOException {
        if (zk != null && zk.getState().isAlive()) {
            zk.close();
        }

        host = newHost;
        boolean readOnly = cl.getOption("readonly") != null;
        if (cl.getOption("secure") != null) {
            System.setProperty(ZKClientConfig.SECURE_CLIENT, "true");
            System.out.println("Secure connection is enabled");
        }
        zk = new ZooKeeperAdmin(host, Integer.parseInt(cl.getOption("timeout")), new MyWatcher(), readOnly);
    }
```
ZooKeeperAdmin

```java
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly) throws IOException {
        this(connectString, sessionTimeout, watcher, canBeReadOnly,
                createDefaultHostProvider(connectString));
    }
```
this(connectString, sessionTimeout, watcher, canBeReadOnly, createDefaultHostProvider(connectString))

```java
   public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly, HostProvider aHostProvider)
            throws IOException {
        this(connectString, sessionTimeout, watcher, canBeReadOnly,aHostProvider, null);
    }
```

### 21.1.2、初始化监听器

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly, HostProvider aHostProvider,
            ZKClientConfig clientConfig) throws IOException {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

        if (clientConfig == null) {
            clientConfig = new ZKClientConfig();
        }
        this.clientConfig = clientConfig;
        watchManager = defaultWatchManager();
        // 赋值 watcher 给默认的 defaultWatcher
        watchManager.defaultWatcher = watcher;
        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        hostProvider = aHostProvider;
		// 客户端与服务器端通信的终端
        cnxn = createConnection(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        cnxn.start();
    }
```

### 21.1.3、解析连接地址 
 ConnectStringParser
 

```java
 public ConnectStringParser(String connectString) {
        // parse out chroot, if any
        int off = connectString.indexOf('/');
        if (off >= 0) {
            String chrootPath = connectString.substring(off);
            // ignore "/" chroot spec, same as null
            if (chrootPath.length() == 1) {
                this.chrootPath = null;
            } else {
                PathUtils.validatePath(chrootPath);
                this.chrootPath = chrootPath;
            }
            connectString = connectString.substring(0, off);
        } else {
            this.chrootPath = null;
        }
		// "hadoop102:2181,hadoop103:2181,hadoop104:2181"用逗号切割
        List<String> hostsList = split(connectString,",");
        for (String host : hostsList) {
            int port = DEFAULT_PORT;
            int pidx = host.lastIndexOf(':');
            if (pidx >= 0) {
                // otherwise : is at the end of the string, ignore
                if (pidx < host.length() - 1) {
                    port = Integer.parseInt(host.substring(pidx + 1));
                }
                host = host.substring(0, pidx);
            }
            serverAddresses.add(InetSocketAddress.createUnresolved(host, port));
        }
    }
```
InetSocketAddress.createUnresolved(host, port)

```java
  private static class InetSocketAddressHolder {
        // The hostname of the Socket Address
        private String hostname;
        // The IP address of the Socket Address
        private InetAddress addr;
        // The port number of the Socket Address
        private int port;

        private InetSocketAddressHolder(String hostname, InetAddress addr, int port) {
            this.hostname = hostname;
            this.addr = addr;
            this.port = port;
        }
	......
```

### 21.1.4、创建通信
```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly, HostProvider aHostProvider,
            ZKClientConfig clientConfig) throws IOException {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

        if (clientConfig == null) {
            clientConfig = new ZKClientConfig();
        }
        this.clientConfig = clientConfig;
        watchManager = defaultWatchManager();
        // 赋值 watcher 给默认的 defaultWatcher
        watchManager.defaultWatcher = watcher;
        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        hostProvider = aHostProvider;
		// 客户端与服务器端通信的终端
        cnxn = createConnection(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        cnxn.start();
    }
```
getClientCnxnSocket()

```java
 private ClientCnxnSocket getClientCnxnSocket() throws IOException {
        String clientCnxnSocketName = getClientConfig().getProperty(
                ZKClientConfig.ZOOKEEPER_CLIENT_CNXN_SOCKET);
        if (clientCnxnSocketName == null) {
            clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
        }
        try {
        // 通过反射获取 clientCxnSocket 对象
            Constructor<?> clientCxnConstructor = Class.forName(clientCnxnSocketName).getDeclaredConstructor(ZKClientConfig.class);
            ClientCnxnSocket clientCxnSocket = (ClientCnxnSocket) clientCxnConstructor.newInstance(getClientConfig());
            return clientCxnSocket;
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + clientCnxnSocketName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```
createConnection

```java
  protected ClientCnxn createConnection(String chrootPath,
            HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
            ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
            boolean canBeReadOnly) throws IOException {
        return new ClientCnxn(chrootPath, hostProvider, sessionTimeout, this,
                watchManager, clientCnxnSocket, canBeReadOnly);
    }
```
new ClientCnxn(chrootPath, hostProvider, sessionTimeout, this, watchManager, clientCnxnSocket, canBeReadOnly);
```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
            ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
            long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
        this.zooKeeper = zooKeeper;
        this.watcher = watcher;
        this.sessionId = sessionId;
        this.sessionPasswd = sessionPasswd;
        this.sessionTimeout = sessionTimeout;
        this.hostProvider = hostProvider;
        this.chrootPath = chrootPath;

        connectTimeout = sessionTimeout / hostProvider.size();
        readTimeout = sessionTimeout * 2 / 3;
        readOnly = canBeReadOnly;

        sendThread = new SendThread(clientCnxnSocket);
        eventThread = new EventThread();
        this.clientConfig=zooKeeper.getClientConfig();
        initRequestTimeout();
    }
```
 SendThread(clientCnxnSocket);
```java
SendThread(ClientCnxnSocket clientCnxnSocket) {
 super(makeThreadName("-SendThread()"));
 state = States.CONNECTING;
 this.clientCnxnSocket = clientCnxnSocket;
 setDaemon(true);
}
```
super(makeThreadName("-SendThread()"));
```java
public ZooKeeperThread(String threadName) {
 super(threadName);
 setUncaughtExceptionHandler(uncaughtExceptionalHandler);
}
```
// ZooKeeperThread 是一个线程，执行它的 run()方法

```java
      public void run() {
            clientCnxnSocket.introduce(this, sessionId, outgoingQueue);
            clientCnxnSocket.updateNow();
            clientCnxnSocket.updateLastSendAndHeard();
            int to;
            long lastPingRwServer = Time.currentElapsedTime();
            final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
            InetSocketAddress serverAddress = null;
            // 在循环里面，循环发送，循环接收
            while (state.isAlive()) {
                try {
                    if (!clientCnxnSocket.isConnected()) {
                        // don't re-establish connection if we are closing
                        if (closing) {
                            break;
                        }
                        if (rwServerAddress != null) {
                            serverAddress = rwServerAddress;
                            rwServerAddress = null;
                        } else {
                            serverAddress = hostProvider.next(1000);
                        }
                        // 启动连接服务端
                        startConnect(serverAddress);
                        clientCnxnSocket.updateLastSendAndHeard();
                    }

                    if (state.isConnected()) {
                        ......
                        to = readTimeout - clientCnxnSocket.getIdleRecv();
                    } else {
                        to = connectTimeout - clientCnxnSocket.getIdleRecv();
                    }
                    
                    ......

                    // If we are in read-only mode, seek for read/write server
                    if (state == States.CONNECTEDREADONLY) {
                        long now = Time.currentElapsedTime();
                        int idlePingRwServer = (int) (now - lastPingRwServer);
                        if (idlePingRwServer >= pingRwTimeout) {
                            lastPingRwServer = now;
                            idlePingRwServer = 0;
                            pingRwTimeout =
                                Math.min(2*pingRwTimeout, maxPingRwTimeout);
                            pingRwServer();
                        }
                        to = Math.min(to, pingRwTimeout - idlePingRwServer);
                    }
					// 接收服务端响应，并处理
                    clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this);
                } catch (Throwable e) {
                   ......
                }
            }
            synchronized (state) {
                // When it comes to this point, it guarantees that later queued
                // packet to outgoingQueue will be notified of death.
                cleanup();
            }
            clientCnxnSocket.close();
            if (state.isAlive()) {
                eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                        Event.KeeperState.Disconnected, null));
            }
            eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                        Event.KeeperState.Closed, null));
            ZooTrace.logTraceMessage(LOG, ZooTrace.getTextTraceLevel(),
                    "SendThread exited loop for session: 0x"
                           + Long.toHexString(getSessionId()));
        }

```
 startConnect(serverAddress);
 

```java
        private void startConnect(InetSocketAddress addr) throws IOException {
            // initializing it for new connection
            saslLoginFailed = false;
            if(!isFirstConnect){
                try {
                    Thread.sleep(r.nextInt(1000));
                } catch (InterruptedException e) {
                    LOG.warn("Unexpected exception", e);
                }
            }
            state = States.CONNECTING;

            String hostPort = addr.getHostString() + ":" + addr.getPort();
            MDC.put("myid", hostPort);
            setName(getName().replaceAll("\\(.*\\)", "(" + hostPort + ")"));
            ......
            logStartConnect(addr);
			// 建立连接
            clientCnxnSocket.connect(addr);
        }
```
connect(addr);ctrl + alt +B 查找 connect 实现类，ClientCnxnSocketNIO.java

```java
 void connect(InetSocketAddress addr) throws IOException {
        SocketChannel sock = createSock();
        try {
           registerAndConnect(sock, addr);
      } catch (IOException e) {
            LOG.error("Unable to open socket to " + addr);
            sock.close();
            throw e;
        }
        initialized = false;

        /*
         * Reset incomingBuffer
         */
        lenBuffer.clear();
        incomingBuffer = lenBuffer;
    }
```
registerAndConnect

```java
void registerAndConnect(SocketChannel sock, InetSocketAddress addr) 
    throws IOException {
        sockKey = sock.register(selector, SelectionKey.OP_CONNECT);
        boolean immediateConnect = sock.connect(addr);
        if (immediateConnect) {
            sendThread.primeConnection();
        }
    }
```
 sendThread.primeConnection();
```java
void primeConnection() throws IOException {
 LOG.info("Socket connection established, initiating session, client: {}, server: {}",
 clientCnxnSocket.getLocalSocketAddress(),
 clientCnxnSocket.getRemoteSocketAddress());
// 标记不是第一次连接
 isFirstConnect = false;
 ... ...
}
```
ctrl + alt +B 查找 doTransport 实现类，ClientCnxnSocketNIO.java

```java
void doTransport(int waitTimeOut, List<Packet> pendingQueue, ClientCnxn cnxn)
            throws IOException, InterruptedException {
        selector.select(waitTimeOut);
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();
        }
        // Everything below and until we get back to the select is
        // non blocking, so time is effectively a constant. That is
        // Why we just have to do this once, here
        updateNow();
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    updateSocketAddresses();
                    sendThread.primeConnection();
                }
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                doIO(pendingQueue, cnxn);
            }
        }
        if (sendThread.getZkState().isConnected()) {
            if (findSendablePacket(outgoingQueue,
                    sendThread.tunnelAuthInProgress()) != null) {
                enableWrite();
            }
        }
        selected.clear();
    }
```
 doIO(pendingQueue, cnxn);
 

```java
   void doIO(List<Packet> pendingQueue, ClientCnxn cnxn)
      throws InterruptedException, IOException {
        SocketChannel sock = (SocketChannel) sockKey.channel();
        if (sock == null) {
            throw new IOException("Socket is null!");
        }
        ......
    }
```
### 21.1.5、执行 run() 

```java
    public static void main(String args[]) throws CliException, IOException, InterruptedException
    {
        ZooKeeperMain main = new ZooKeeperMain(args);
        main.run();
    }
```
  main.run();
  

```java
void run() throws CliException, IOException, InterruptedException {
        if (cl.getCommand() == null) {
            System.out.println("Welcome to ZooKeeper!");

            boolean jlinemissing = false;
            // only use jline if it's in the classpath
            try {
                Class<?> consoleC = Class.forName("jline.console.ConsoleReader");
                Class<?> completorC =
                    Class.forName("org.apache.zookeeper.JLineZNodeCompleter");

                System.out.println("JLine support is enabled");

                Object console =
                    consoleC.getConstructor().newInstance();

                Object completor =
                    completorC.getConstructor(ZooKeeper.class).newInstance(zk);
                Method addCompletor = consoleC.getMethod("addCompleter",
                        Class.forName("jline.console.completer.Completer"));
                addCompletor.invoke(console, completor);

                String line;
                Method readLine = consoleC.getMethod("readLine", String.class);
                while ((line = (String)readLine.invoke(console, getPrompt())) != null) {
                    executeLine(line);
                }
            } catch (ClassNotFoundException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (NoSuchMethodException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (InvocationTargetException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (IllegalAccessException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (InstantiationException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            }

            if (jlinemissing) {
                System.out.println("JLine support is disabled");
                BufferedReader br =
                    new BufferedReader(new InputStreamReader(System.in));

                String line;
                while ((line = br.readLine()) != null) {
                // 一行一行读取命令
                    executeLine(line);
                }
            }
        } else {
            // Command line args non-null.  Run what was passed.
            processCmd(cl);
        }
        System.exit(exitCode);
    }
```
  executeLine(line);
  

```java
   public void executeLine(String line) throws CliException, InterruptedException, IOException {
      if (!line.equals("")) {
        cl.parseCommand(line);
        addToHistory(commandCount,line);
        // 处理客户端命令
        processCmd(cl);
        commandCount++;
      }
    }
```
processCmd(cl);

```java
    protected boolean processCmd(MyCommandOptions co) throws CliException, IOException, InterruptedException {
        boolean watch = false;
        try {
        // 解析命令
            watch = processZKCmd(co);
            exitCode = 0;
        } catch (CliException ex) {
            exitCode = ex.getExitCode();
            System.err.println(ex.getMessage());
        }
        return watch;
    }
```
processZKCmd(co);

```java
protected boolean processZKCmd(MyCommandOptions co) throws CliException, IOException, InterruptedException {
        String[] args = co.getArgArray();
        String cmd = co.getCommand();
        if (args.length < 1) {
            usage();
            throw new MalformedCommandException("No command entered");
        }

        if (!commandMap.containsKey(cmd)) {
            usage();
            throw new CommandNotFoundException("Command not found " + cmd);
        }
        
        boolean watch = false;
        LOG.debug("Processing " + cmd);


        if (cmd.equals("quit")) {
            zk.close();
            System.exit(exitCode);
        } else if (cmd.equals("redo") && args.length >= 2) {
            Integer i = Integer.decode(args[1]);
            if (commandCount <= i || i < 0) { // don't allow redoing this redo
                throw new MalformedCommandException("Command index out of range");
            }
            cl.parseCommand(history.get(i));
            if (cl.getCommand().equals("redo")) {
                throw new MalformedCommandException("No redoing redos");
            }
            history.put(commandCount, history.get(i));
            processCmd(cl);
        } else if (cmd.equals("history")) {
            for (int i = commandCount - 10; i <= commandCount; ++i) {
                if (i < 0) continue;
                System.out.println(i + " - " + history.get(i));
            }
        } else if (cmd.equals("printwatches")) {
            if (args.length == 1) {
                System.out.println("printwatches is " + (printWatches ? "on" : "off"));
            } else {
                printWatches = args[1].equals("on");
            }
        } else if (cmd.equals("connect")) {
            if (args.length >= 2) {
                connectToZK(args[1]);
            } else {
                connectToZK(host);
            }
        }
        
        // Below commands all need a live connection
        if (zk == null || !zk.getState().isAlive()) {
            System.out.println("Not connected");
            return false;
        }
        
        // execute from commandMap
        CliCommand cliCmd = commandMapCli.get(cmd);
        if(cliCmd != null) {
            cliCmd.setZk(zk);
            watch = cliCmd.parse(args).exec();
        } else if (!commandMap.containsKey(cmd)) {
             usage();
        }
        return watch;
    }

```

