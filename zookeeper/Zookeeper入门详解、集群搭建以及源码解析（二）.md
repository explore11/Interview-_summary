# 6、服务器动态上下线监听案例 
## 6.1、需求
某分布式系统中，主节点可以有多台，可以动态上下线，任意一台客户端都能实时感知到主节点服务器的上下线。
## 6.2、 需求分析
![在这里插入图片描述](https://img-blog.csdnimg.cn/f31b870f1ad74daa8dee339394655b58.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 6.3、具体实现
（1）	先在集群上创建/servers 节点 

```java
[zk: localhost:2181(CONNECTED) 10] create /servers "servers" Created /servers 
```

（2）	在 Idea 中创建包名：com.atguigu.zkcase 
（3）	服务器端向 Zookeeper 注册代码 

```java
package com.song.zkcase;

import java.io.IOException;

import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.ZooDefs.Ids;

public class DistributeServer {

    private static String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";
    private static int sessionTimeout = 2000;
    private ZooKeeper zk = null;
    private String parentNode = "/servers";

    // 创建到zk的客户端连接
    public void getConnect() throws IOException {

        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            public void process(WatchedEvent event) {
            }
        });
    }

    // 注册服务器
    public void registerServer(String hostname) throws Exception {

        String create = zk.create(parentNode + "/server", hostname.getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println(hostname + " is online " + create);
    }

    // 业务功能
    public void business(String hostname) throws Exception {
        System.out.println(hostname + " is working ...");
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws Exception {
        // 1获取zk连接
        DistributeServer server = new DistributeServer();
        server.getConnect();
        // 2 利用zk连接注册服务器信息
        server.registerServer(args[0]);
        // 3 启动业务功能
        server.business(args[0]);
    }
}

```

（4）	客户端代码

```java
package com.song.zkcase;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

public class DistributeClient {

    private static String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";
    private static int sessionTimeout = 2000;
    private ZooKeeper zk = null;
    private String parentNode = "/servers";

    // 创建到zk的客户端连接
    public void getConnect() throws IOException {
        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            public void process(WatchedEvent event) {
                // 再次启动监听
                try {
                    getServerList();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }

    // 获取服务器列表信息
    public void getServerList() throws Exception {
        // 1获取服务器子节点信息，并且对父节点进行监听
        List<String> children = zk.getChildren(parentNode, true);
        // 2存储服务器信息列表
        ArrayList<String> servers = new ArrayList<String>();
        // 3遍历所有节点，获取节点中的主机名称信息
        for (String child : children) {
            byte[] data = zk.getData(parentNode + "/" + child, false, null);
            servers.add(new String(data));
        }
        // 4打印服务器列表信息
        System.out.println(servers);
    }

    // 业务功能
    public void business() throws Exception {
        System.out.println("client is working ...");
        Thread.sleep(Long.MAX_VALUE);
    }

    public static void main(String[] args) throws Exception {
        // 1获取zk连接
        DistributeClient client = new DistributeClient();
        client.getConnect();
        // 2获取servers的子节点信息，从中获取服务器信息列表
        client.getServerList();
        // 3业务进程启动
        client.business();
    }
}

```
（5）	测试
1）	在 Linux 命令行上操作增加减少服务器 
（1）	启动 DistributeClient 客户端 
（2）	在 hadoop102 上 zk 的客户端/servers 目录上创建临时带序号节点 

```java
[zk: localhost:2181(CONNECTED) 1] create 	-e 	-s  /servers/hadoop102 "hadoop102" 
[zk: localhost:2181(CONNECTED) 2] create 	-e 	-s  /servers/hadoop103 "hadoop103" 
```

（3）	观察 Idea 控制台变化 

```java
[hadoop102, hadoop103] 
```

（4）	执行删除操作 

```java
[zk: 	localhost:2181(CONNECTED)  8] 	delete  /servers/hadoop1020000000000 	
```

（5）	观察 Idea 控制台变化 

```java
[hadoop103] 
```
2）	在 Idea 上操作增加减少服务器 
（1）	启动 DistributeClient 客户端（如果已经启动过，不需要重启） 
（2）	启动 DistributeServer 服务 
![在这里插入图片描述](https://img-blog.csdnimg.cn/78720c8054904bb3b40963a7300d6849.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 回到 DistributeServer 的 main 方法，右键，在弹出的窗口中点击 Run “DistributeServer.main()”
- 观察 DistributeServer 控制台，提示 hadoop102 is working 
- 观察 DistributeClient 控制台，提示 hadoop102 已经上线

# 7、Zookeeper 实现分布式锁
**什么叫做分布式锁呢？**
比如说"进程 1"在使用该资源的时候，会先去获得锁，"进程 1"获得锁以后会对该资源保持独占，这样其他进程就无法访问该资源，"进程 1"用完该资源以后就将锁释放掉，让其他进程来获得锁，那么通过这个锁机制，我们就能保证了分布式系统中多个进程能够有序的访问该临界资源。那么我们把这个分布式环境下的这个锁叫作分布式锁。 

**Zookeeper 实现分布式锁原理**
![在这里插入图片描述](https://img-blog.csdnimg.cn/f9ae00c7468144e48f6d6b3479270ffe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 - **ZooKeeper的临时顺序节点，都是一个天然的顺序发号器。**

	在每一个节点下面创建临时顺序节点（EPHEMERAL_SEQUENTIAL）类型，新的子节点后面，会加上一个次序编号，而这个生成的次序编号，是上一个生成的次序编号加一。例如，有一个用于发号的节点“/locks”为父亲节点，可以在这个父节点下面创建相同前缀的临时顺序子节点，假定相同的前缀为“/locks/seq-”。第一个创建的子节点基本上应该为/locks/seq-0000000000，下一个节点则为/locks/seq-0000000001，依次类推，

- **ZooKeeper节点的递增有序性，可以确保锁的公平**
一个ZooKeeper分布式锁，首先需要创建一个父节点，尽量是持久节点（PERSISTENT类型），然后每个要获得锁的线程，都在这个节点下创建个临时顺序节点。由于ZK节点，是按照创建的次序，依次递增的。为了确保公平，可以简单的规定：**编号最小的那个节点，表示获得了锁**。所以，每个线程在尝试占用锁之前，首先判断自己是排号是不是当前最小，如果是，则获取锁。

- **ZooKeeper的节点监听机制，可以保障占有锁的传递有序而且高效**
	每个线程抢占锁之前，先尝试创建自己的ZNode。同样，释放锁的时候，就需要删除创建的Znode。创建成功后，如果不是排号最小的节点，就处于等待通知的状态。等谁的通知呢？不需要其他人，只需要等前一个Znode的通知就可以了。前一个Znode删除的时候，会触发Znode事件，当前节点能监听到删除事件，就是轮到了自己占有锁的时候。第一个通知第二个、第二个通知第三个，依次向后。ZooKeeper的节点监听机制，能够非常完美地实现这种击鼓传花似的信息传递。
	
	具体的方法是，每一个等通知的Znode节点，只需要监听（linsten）或者监视（watch）排号在自己前面那个，而且紧挨在自己前面的那个节点，就能收到其删除事件了。只要上一个节点被删除了，就进行再一次判断，看看自己是不是序号最小的那个节点，如果是，自己就获得锁。
	另外，ZooKeeper能保证由于网络异常或者其他原因，集群中占用锁的客户端失联时，锁能够被有效释放。一旦占用Znode锁的客户端与ZooKeeper集群服务器失去联系，这个临时Znode也将自动删除。排在它后面的那个节点，也能收到删除事件，从而获得锁。正是由于这个原因，在创建取号节点的时候，尽量创建临时znode节点，

- **ZooKeeper的节点监听机制，能避免羊群效应**
	ZooKeeper这种首尾相接，后面监听前面的方式，可以避免羊群效应。所谓羊群效应就是一个节点挂掉，所有节点都去监听，然后做出反应，这样会给服务器带来巨大压力，所以有了临时顺序节点，当一个节点挂掉，只有它后面的那一个节点才做出反应。

## 7.1 原生 Zookeeper 实现分布式锁案例 
**分布式锁**

```java
package com.song.lock;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class DistributedLock {

    // zookeeper server列表 
    private String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";
    // 超时时间
    private int sessionTimeout = 20000;
    private ZooKeeper zk;
    private String rootNode = "locks";
    private String subNode = "seq-";
    // 当前client等待的子节点 
    private String waitPath;
    //ZooKeeper连接 
    private CountDownLatch connectLatch = new CountDownLatch(1);
    //ZooKeeper节点等待
    private CountDownLatch waitLatch = new CountDownLatch(1);
    // 当前client创建的子节点 
    private String currentNode;

    // 和zk服务建立连接，并创建根节点
    public DistributedLock() throws IOException, InterruptedException, KeeperException {
        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {

            public void process(WatchedEvent event) {
                // 连接建立时, 打开latch, 唤醒wait在该latch上的线程 
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    connectLatch.countDown();
                }

                // 发生了waitPath的删除事件 
                if (event.getType() == Event.EventType.NodeDeleted && event.getPath().equals(waitPath)) {
                    waitLatch.countDown();
                }
            }
        });

        // 等待连接建立
        connectLatch.await();
        //获取根节点状态
        Stat stat = zk.exists("/" + rootNode, false);
        //如果根节点不存在，则创建根节点，根节点类型为永久节点
        if (stat == null) {
            System.out.println("根节点不存在");
            zk.create("/" + rootNode, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

    }

    // 加锁方法
    public void zkLock() {
        try {
            //在根节点下创建临时顺序节点，返回值为创建的节点路径
            currentNode = zk.create("/" + rootNode + "/" + subNode, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            // wait一小会, 让结果更清晰一些
            Thread.sleep(10);
            // 注意, 没有必要监听"/locks"的子节点的变化情况
            List<String> childrenNodes = zk.getChildren("/" + rootNode, false);
            // 列表中只有一个子节点, 那肯定就是 currentNode , 说明 client获得锁
            if (childrenNodes.size() == 1) {
                return;
            } else {
                //对根节点下的所有临时顺序节点进行从小到大排序
                Collections.sort(childrenNodes);
                //当前节点名称
                String thisNode = currentNode.substring(("/" + rootNode + "/").length());
                //获取当前节点的位置
                int index = childrenNodes.indexOf(thisNode);
                if (index == -1) {
                    System.out.println("数据异常");
                } else if (index == 0) {
                    // index == 0, 说明 thisNode 在列表中最小, 当前 client获得锁
                    return;
                } else {
                    // 获得排名比currentNode 前1位的节点
                    this.waitPath = "/" + rootNode + "/" + childrenNodes.get(index - 1);
                    // 在 waitPath 上注册监听器, 当 waitPath 被删除时, zookeeper会回调监听器的process方法
                    zk.getData(waitPath, true, new Stat());
                    //进入等待锁状态
                    waitLatch.await();
                    return;
                }
            }
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 解锁方法
    public void zkUnlock() {
        try {
            zk.delete(this.currentNode, -1);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

**测试**

```java
package com.song.lock;

import org.apache.zookeeper.KeeperException;

import java.io.IOException;

public class DistributedLockTest {

    public static void main(String[] args) throws InterruptedException, IOException, KeeperException {

        // 创建分布式锁1
        final DistributedLock lock1 = new DistributedLock();
        // 创建分布式锁2
        final DistributedLock lock2 = new DistributedLock();
        //线程1
        new Thread(new Runnable() {

            public void run() {
                // 获取锁对象
                try {
                    lock1.zkLock();
                    System.out.println("线程1获取锁");
                    Thread.sleep(5 * 1000);

                    lock1.zkUnlock();
                    System.out.println("线程1释放锁");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        },"线程1").start();

        //线程2
        new Thread(new Runnable() {
            public void run() {
                // 获取锁对象
                try {
                    lock2.zkLock();
                    System.out.println("线程2获取锁");
                    Thread.sleep(5 * 1000);

                    lock2.zkUnlock();
                    System.out.println("线程2释放锁");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        },"线程2").

                start();
    }
}

```
## 7.2、Curator 框架实现分布式锁案例
### 7.2.1、原生的 Java API 开发存在的问题
（1）	会话连接是异步的，需要自己去处理。比如使用 CountDownLatch 
（2）	Watch 需要重复注册，不然就不能生效 
（3）	开发的复杂性还是比较高的 
（4）	不支持多节点删除和创建。需要自己去递归 
### 7.2.2、Curator 案例实操 
Curator是一个专门解决分布式锁的框架，解决了原生 Java API开发分布式遇到的问题。

**依赖**

```xml
<dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.8.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.5.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.3.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.3.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.3.0</version>
        </dependency>
    </dependencies>
```

```java
package com.song.lock;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessLock;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class CuratorLockTest {

    private String rootNode = "/locks";
    // zookeeper server列表 
    private String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";
    // connection超时时间
    private int connectionTimeout = 2000;
    // session超时时间
    private int sessionTimeout = 2000;

    public static void main(String[] args) {
        new CuratorLockTest().test();
    }

    // 测试
    private void test() {
        // 创建分布式锁1
        final InterProcessLock lock1 = new InterProcessMutex(getCuratorFramework(), rootNode);
        // 创建分布式锁2
        final InterProcessLock lock2 = new InterProcessMutex(getCuratorFramework(), rootNode);

        new Thread(new Runnable() {
            public void run() {
                // 获取锁对象
                try {
                    lock1.acquire();
                    System.out.println("线程1获取锁");
                    // 测试锁重入
                    lock1.acquire();
                    System.out.println("线程1再次获取锁");
                    Thread.sleep(5 * 1000);
                    lock1.release();
                    System.out.println("线程1释放锁");
                    lock1.release();
                    System.out.println("线程1再次释放锁");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(new Runnable() {
            public void run() {
                // 获取锁对象
                try {
                    lock2.acquire();
                    System.out.println("线程2获取锁");
                    // 测试锁重入
                    lock2.acquire();
                    System.out.println("线程2再次获取锁");
                    Thread.sleep(5 * 1000);
                    lock2.release();
                    System.out.println("线程2释放锁");
                    lock2.release();
                    System.out.println("线程2再次释放锁");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

    }

    // 分布式锁初始化
    public CuratorFramework getCuratorFramework() {
        //重试策略，初试时间3秒，重试3次 
        RetryPolicy policy = new ExponentialBackoffRetry(3000, 3);
        //通过工厂创建Curator 
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connectString)
                .connectionTimeoutMs(connectionTimeout)
                .sessionTimeoutMs(sessionTimeout)
                .retryPolicy(policy).build();

        //开启连接
        client.start();
        System.out.println("zookeeper 初始化完成...");
        return client;
    }

}

```
# 8、选举机制（面试重点） 
## 8.1、第一次启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/0a2ddd54ac8c4f8fb26e292b0288119c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 8.2、非一次启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/8b59e92343044083a22f627a5988a157.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

# 9、客户端向服务端写数据流程 
## 9.1、写流程的写入请求直接发给Leader节点![在这里插入图片描述](https://img-blog.csdnimg.cn/6f1f79f9576049949222434f3aa4b0c0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 当客户端发出的写请求，直接请求到Leader节点时，会自己先写入数据
- 然后与Follwer节点进行通信，写入数据
- Follwer节点写入数据成功之后，会给Leader节点返回ack
- Leader节点收到ack后，会判断ack的个数是否超过半数，如果超过半数，会给客户端返回写入成功
- 接着再让其他的Follwer节点继续写入数据，返回ack.

## 9.2、写流程的写入请求直接发给Follower节点
 
 
![在这里插入图片描述](https://img-blog.csdnimg.cn/58c57fd9819146288561b192322aa8ae.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

- 当客户端发出的写请求，直接请求到Follwer节点时，Follwer节点会与Leader节点进行通信，发送写请求
- Leader节写入数据完毕后，会和Follwer节点通信，让Follwer节点进行写数据，写入成功返回ack
- Leader节点收到ack后，会判断ack的个数是否超过半数，如果超过半数，会给请求的Follwer节点返回ack,
- 接着Follwer节点再给客户端返回ack.
