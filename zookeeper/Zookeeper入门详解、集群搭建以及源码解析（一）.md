# 1、Zookeeper 入门 
## 1.1、 概述
Zookeeper 是一个开源的分布式的，为分布式框架提供协调服务的 Apache 项目。
## 1.2、 特点
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/5054603858534325b4ee7b152a0b22ed.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- Zookeeper：一个领导者（Leader），多个跟随者（Follower）组成的集群。
- 集群中只要有半数以上节点存活，Zookeeper集群就能正常服务。所 以Zookeeper适合安装奇数台服务器。
- 全局数据一致：每个Server保存一份相同的数据副本，Client无论连接到哪个Server，数据都是一致的。
- 更新请求顺序执行，来自同一个Client的更新请求按其发送顺序依次执行。
- 数据更新原子性，一次数据更新要么成功，要么失败。
- 实时性，在一定时间范围内，Client能读到最新数据。

## 1.3 、数据结构
ZooKeeper 数据模型的结构与 Unix 文件系统很类似，整体上可以看作是一棵树，每个节点称做一个 ZNode。**每一个 ZNode 默认能够存储 1MB 的数据**，每个 ZNode 都可以通过其路径唯一标识。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f4d1ec42045849e9b97e8f0084eeed01.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 1.4、 应用场景
提供的服务包括：**统一命名服务、统一配置管理、统一集群管理、服务器节点动态上下线、软负载均衡等**。

### 1.4.1、统一命名服务
在分布式环境下，经常需要对应用/服务进行统一命名，便于识别。例如：IP不容易记住，而域名容易记住。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ed7c0c20f9494086b274c6ee69a002bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.4.2、统一配置管理
**1）分布式环境下，配置文件同步非常常见。**
（1）一般要求一个集群中，所有节点的配置信息是一致的，比如 Kafka 集群。
（2）对配置文件修改后，希望能够快速同步到各个节点上。 
**2）配置管理可交由ZooKeeper实现。**
 （1）可将配置信息写入ZooKeeper上的一个Znode
 （2）各个客户端服务器监听这个Znode。 
 （3）一 旦Znode中的数据被修改，ZooKeeper将通知各个客户端服务器。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/12d5053747c64953925fb3c2852c180a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_17,color_FFFFFF,t_70,g_se,x_16)
### 1.4.3、统一集群管理
**1）分布式环境中，实时掌握每个节点的状态是必要的。** 
（1）可根据节点实时状态做出一些调整。
**2）ZooKeeper可以实现实时监控节点状态变化**
（1）可将节点信息写入ZooKeeper上的一个ZNode。
（2）监听这个ZNode可获取它的实时状态变化。
![在这里插入图片描述](https://img-blog.csdnimg.cn/c5e35428e8df41b187726cdf32f848d2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 1.4.4、服务器节点动态上下线
![在这里插入图片描述](https://img-blog.csdnimg.cn/d8a902a2dc834a63ac9b0a03a764fc53.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 1.4.5、软负载均衡
在Zookeeper中记录每台服务器的访问数，让访问数最少的服务器去处理最新的客户端请求
![在这里插入图片描述](https://img-blog.csdnimg.cn/6ff861f023024ec7b34e89401697e032.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 1.4.5、下载地址
[下载地址](https://zookeeper.apache.org/)

# 2、 Zookeeper 本地安装
## 2.1、安装前准备
- 安装 JDK  
![在这里插入图片描述](https://img-blog.csdnimg.cn/4a6048222aa64e28a51f784705ef439e.png)
[linux环境jdk安装](https://blog.csdn.net/prefect_start/article/details/117624615)

- 拷贝 apache-zookeeper-3.5.7-bin.tar.gz 安装包到 Linux 系统下 ![在这里插入图片描述](https://img-blog.csdnimg.cn/911aa79279784f80a20a2dc789332819.png)

- 解压到指定目录 

```xml
[atguigu@hadoop102 software]$ tar -zxvf apache-zookeeper-3.5.7bin.tar.gz -C /opt/module/ 
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/fe4ea74c3ba449a4be29100560694ec8.png)
- 修改名称 
```xml
[atguigu@hadoop102 module]$ mv apache-zookeeper-3.5.7 -bin/ zookeeper-3.5.7 
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/c788af5eaeaf49dab7998a258ab72f39.png)
## 2.2、配置修改
- 	将/opt/module/zookeeper-3.5.7/conf 这个路径下的 zoo_sample.cfg 修改为 zoo.cfg； 

```xml
[atguigu@hadoop102 conf]$ mv zoo_sample.cfg zoo.cfg 
```

- 	打开 zoo.cfg 文件，修改 dataDir 路径： 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ vim zoo.cfg 
```

修改如下内容： 

```xml
dataDir=/opt/module/zookeeper-3.5.7/zkData 
```

- 在/opt/module/zookeeper-3.5.7/这个目录上创建 zkData 文件夹 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ mkdir zkData 
```
## 2.3、操作 Zookeeper 
（1）	启动 Zookeeper 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkServer.sh start 
```

（2）	查看进程是否启动 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ jps 
4020 Jps 
4001 QuorumPeerMain 
```

（3）	查看状态 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkServer.sh status 
ZooKeeper JMX enabled by default 
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg 
Mode: standalone 
```

（4）	启动客户端 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkCli.sh 
```

（5）	退出客户端： 

```xml
[zk: localhost:2181(CONNECTED) 0] quit 
```

（6）	停止 Zookeeper 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkServer.sh stop
```

## 2.4、配置参数解读 
![在这里插入图片描述](https://img-blog.csdnimg.cn/727ca0502e214d8795797af6c6cdc0d9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**Zookeeper中的配置文件zoo.cfg中参数含义解读如下：** 
1. tickTime = 2000：通信心跳时间，Zookeeper服务器与客户端心跳时间，单位毫秒 
![在这里插入图片描述](https://img-blog.csdnimg.cn/687b53d819c84b2ba13c1ff2d375e489.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
2. initLimit = 10：LF初始通信时限,Leader和Follower初始连接时能容忍的最多心跳数（tickTime的数量），也就是说Leader和Follower初始连接的时间不能超过20s。
![在这里插入图片描述](https://img-blog.csdnimg.cn/1555cf4794c345b582b6c336b56cddda.png)

3. syncLimit = 5：LF同步通信时限,Leader和Follower之间通信时间如果超过syncLimit * tickTime，Leader认为Follwer死
掉，从服务器列表中删除Follwer。也就是说Leader和Follower初始连接成功之后，在进行通信的时间不能超过10s。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4da95aefbd504b72bd53d6ce8616aba7.png)
4. dataDir：保存Zookeeper中的数据。注意：默认的tmp目录，容易被Linux系统定期删除，所以一般不用默认的tmp目录。
5. clientPort = 2181：客户端连接端口，通常不做修改。


# 3、Zookeeper 集群安装
## 3.1、集群规划
在 hadoop102、hadoop103 和 hadoop104 三个节点上都部署 Zookeeper。
一般安装奇数台。
- 10 台服务器：3 台 zk；
- 20 台服务器：5 台 zk；
- 100 台服务器：11 台 zk；
- 200 台服务器：11 台 zk

**服务器台数多：好处，提高可靠性；坏处：提高通信延时**
## 3.2、解压安装
（1）在 hadoop102 解压 Zookeeper 安装包到/opt/module/目录下

```xml
[atguigu@hadoop102 software]$ tar -zxvf apache-zookeeper-3.5.7-
bin.tar.gz -C /opt/module/
```

（2）修改 apache-zookeeper-3.5.7-bin 名称为 zookeeper-3.5.7

```xml
[atguigu@hadoop102 module]$ mv apache-zookeeper-3.5.7-bin/ 
zookeeper-3.5.7
```
## 3.3、配置服务器编号
（1）在/opt/module/zookeeper-3.5.7/这个目录下创建 zkData

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ mkdir zkData
```

（2）在/opt/module/zookeeper-3.5.7/zkData 目录下创建一个 myid 的文件

```xml
[atguigu@hadoop102 zkData]$ vi myid
2
```

**在文件中添加与 server 对应的编号（注意：上下不要有空行，左右不要有空格）**

注意：添加 myid 文件，一定要在 Linux 里面创建，在 notepad++里面很可能乱码
（3）拷贝配置好的 zookeeper 到其他机器上

```xml
[atguigu@hadoop102 module ]$ xsync zookeeper-3.5.7
```

并分别在 hadoop103、hadoop104 上修改 myid 文件中内容为 3、4


## 3.4、配置zoo.cfg文件
（1）重命名/opt/module/zookeeper-3.5.7/conf 这个目录下的 zoo_sample.cfg 为 zoo.cfg

```xml
[atguigu@hadoop102 conf]$ mv zoo_sample.cfg zoo.cfg
```

（2）打开 zoo.cfg 文件

```xml
[atguigu@hadoop102 conf]$ vim zoo.cfg
#修改数据存储路径配置
dataDir=/opt/module/zookeeper-3.5.7/zkData
#增加如下配置
#######################cluster##########################
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
server.4=hadoop104:2888:3888
```

（3）配置参数解读
server.A=B:C:D。 A 是一个数字，表示这个是第几号服务器；
集群模式下配置一个文件 myid，这个文件在 dataDir 目录下，这个文件里面有一个数据
就是 A 的值，Zookeeper 启动时读取此文件，拿到里面的数据与 zoo.cfg 里面的配置信息比
较从而判断到底是哪个 server。 B 是这个服务器的地址；
C 是这个服务器 Follower 与集群中的 Leader 服务器交换信息的端口；
D 是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的
Leader，而这个端口就是用来执行选举时服务器相互通信的端口。
（4）同步 zoo.cfg 配置文件

```xml
[atguigu@hadoop102 conf]$ xsync zoo.cfg
```

## 3.5、集群操作 
（1）	分别启动 Zookeeper 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkServer.sh start [atguigu@hadoop103 zookeeper-3.5.7]$ bin/zkServer.sh start 
[atguigu@hadoop104 zookeeper-3.5.7]$ bin/zkServer.sh start 
```

（2）	查看状态 

```xml
[atguigu@hadoop102 zookeeper-3.5.7]# bin/zkServer.sh status 
JMX enabled by default 
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg 
Mode: follower 
[atguigu@hadoop103 zookeeper-3.5.7]# bin/zkServer.sh status 
JMX enabled by default 
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg 
Mode: leader 
[atguigu@hadoop104 zookeeper-3.4.5]# bin/zkServer.sh status 
JMX enabled by default 
Using config: /opt/module/zookeeper-3.5.7/bin/../conf/zoo.cfg Mode: follower 
```

## 3.6、ZK 集群启动停止脚本
1）在 hadoop102 的/home/atguigu/bin 目录下创建脚本

```xml
[atguigu@hadoop102 bin]$ vim zk.sh
```

在脚本中编写如下内容
```shell
#!/bin/bash 
case $1 in "start"){ 
 for i in hadoop102 hadoop103 hadoop104 
 do 
        echo ---------- zookeeper $i 启动 ------------   
		ssh $i "/opt/model/zookeeper-3.5.7/bin/zkServer.sh start" 
 done 
};; 
"stop"){ 
 for i in hadoop102 hadoop103 hadoop104 
 do 
        echo ---------- zookeeper $i 停止 ------------       
		ssh $i "/opt/model/zookeeper-3.5.7/bin/zkServer.sh stop" 
 done 
};;
"status"){ 
 for i in hadoop102 hadoop103 hadoop104 
 do 
        echo ---------- zookeeper $i 状态 ------------       
		ssh $i "/opt/model/zookeeper-3.5.7/bin/zkServer.sh status"  
 done 
};;
esac 

```

2）增加脚本执行权限

```xml
[atguigu@hadoop102 bin]$ chmod 777 zk.sh
```

3）Zookeeper 集群启动脚本

```xml
[atguigu@hadoop102 module]$ zk.sh start
```

4）Zookeeper 集群停止脚本

```xml
[atguigu@hadoop102 module]$ zk.sh stop
```
# 4、客户端命令行操作
## 4.1、 命令行语法
- help 	显示所有操作命令 
- ls path  	使用 ls 命令来查看当前 znode 的子节点 [可监听] 
			-w  监听子节点变化 
			-s   附加次级信息 
- create 	普通创建 
			-s  含有序列 
			-e  临时（重启或者超时消失） 
- get path  	获得节点的值 [可监听] 
			-w  监听节点内容变化 
			-s   附加次级信息 
- set 	设置节点的具体值 
- stat 	查看节点状态 
- delete 	删除节点 
- deleteall 	递归删除节点 

1）	启动客户端 

```xml
[atguigu@hadoop102 hadoop102:2181 	zookeeper-3.5.7]$ bin/zkCli.sh 	-server hadoop102:2181 
```

2）	显示所有操作命令 

```xml
[zk: hadoop102:2181(CONNECTED) 1] help 
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/6137b675f88e442b9d350af2d2d6e86e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 4.2、 znode 节点数据信息
1）	查看当前znode中所包含的内容 

```xml
[zk: hadoop102:2181(CONNECTED) 0] ls / 
[zookeeper] 
```

2）	查看当前节点详细数据 

```xml
[zk: hadoop102:2181(CONNECTED) 5] ls -s / 
[zookeeper]cZxid = 0x0 
ctime = Thu Jan 01 08:00:00 CST 1970 
mZxid = 0x0 
mtime = Thu Jan 01 08:00:00 CST 1970 
pZxid = 0x0 
cversion = -1 
dataVersion = 0 
aclVersion = 0 
ephemeralOwner = 0x0 
dataLength = 0 
numChildren = 1 
```

- czxid：创建节点的事务 zxid 每次修改 ZooKeeper 状态都会产生一个 ZooKeeper 事务 ID。事务 ID 是 ZooKeeper 中所有修改总的次序。每次修改都有唯一的 zxid，如果 zxid1 小于 zxid2，那么 zxid1 在 zxid2 之前发生。 
- ctime：znode 被创建的毫秒数（从 1970 年开始） 
- mzxid：znode 最后更新的事务 zxid 
- mtime：znode 最后修改的毫秒数（从 1970 年开始） 
- pZxid：znode 最后更新的子节点 zxid 
- cversion：znode 子节点变化号，znode 子节点修改次数 
- dataversion：znode 数据变化号 
- aclVersion：znode 访问控制列表的变化号 
- ephemeralOwner：如果是临时节点，这个是 znode 拥有者的 session id。如果不是临时节点则是 0。 
- dataLength：znode 的数据长度 
- numChildren：znode 子节点数量 

## 4.3、节点类型（持久/短暂/有序号/无序号）
- 持久（Persistent）：客户端和服务器端断开连接后，创建的节点不删除
- 短暂（Ephemeral）：客户端和服务器端断开连接后，创建的节点自己删除
- 有序号：创建znode时设置顺序标识，znode名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护
- 无序号：创建znode时不会设置顺序标识

**说明：注意：在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序**

**（1）持久化目录节点**
客户端与Zookeeper断开连接后，该节点依旧存在
**（2）持久化顺序编号目录节点**
客户端与Zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号
**（3）临时目录节点**
客户端与Zookeeper断开连接后，该节点被删除
**（4）临时顺序编号目录节点**
客户端与 Zookeeper 断开连接后，该节点被删除 ，只是Zookeeper给该节点名称进行顺序编号。

![在这里插入图片描述](https://img-blog.csdnimg.cn/9fa10761900b42f5bc32faa5310f5e35.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
1）	分别创建2个普通节点（永久节点 + 不带序号） 

```xml
[zk: localhost:2181(CONNECTED) 3] create /sanguo "diaochan" 
Created /sanguo 
[zk: localhost:2181(CONNECTED) 4] create /sanguo/shuguo  "liubei" 
Created /sanguo/shuguo 
```

 **注意：创建节点时，要赋值** 
2）	获得节点的值 

```xml
[zk: localhost:2181(CONNECTED) 5] get -s /sanguo diaochan 
cZxid = 0x100000003 
ctime = Wed Aug 29 00:03:23 CST 2018 
mZxid = 0x100000003 
mtime = Wed Aug 29 00:03:23 CST 2018 
pZxid = 0x100000004 
cversion = 1 
dataVersion = 0 
aclVersion = 0 
ephemeralOwner = 0x0 
dataLength = 7 
numChildren = 1 

[zk: localhost:2181(CONNECTED) 6] get -s /sanguo/shuguo liubei 
cZxid = 0x100000004 
ctime = Wed Aug 29 00:04:35 CST 2018
mZxid = 0x100000004 
mtime = Wed Aug 29 00:04:35 CST 2018 
pZxid = 0x100000004 
cversion = 0
dataVersion = 0 
aclVersion = 0 
ephemeralOwner = 0x0 
dataLength = 6 
numChildren = 0 
```
3）	创建带序号的节点（永久节点 + 带序号） 
（1）	先创建一个普通的根节点/sanguo/weiguo 

```xml
[zk: localhost:2181(CONNECTED) 1] create /sanguo/weiguo  "caocao" 
Created /sanguo/weiguo 
```

（2）	创建带序号的节点 

```xml
[zk: 	localhost:2181(CONNECTED) 	2] 	create 	-s /sanguo/weiguo/zhangliao "zhangliao" 
Created /sanguo/weiguo/zhangliao0000000000  
[zk: 	localhost:2181(CONNECTED) 	3] 	create 	-s /sanguo/weiguo/zhangliao "zhangliao" 
Created /sanguo/weiguo/zhangliao0000000001  
[zk: 	localhost:2181(CONNECTED) 	4] /sanguo/weiguo/xuchu "xuchu" 
Created /sanguo/weiguo/xuchu0000000002 	create 	-s 
```

**如果原来没有序号节点，序号从 0 开始依次递增。如果原节点下已有 2 个节点，则再排序时从 2 开始，以此类推。** 
4）	创建短暂节点（短暂节点 + 不带序号 or 带序号） 
（1）	创建短暂的不带序号的节点 

```xml
[zk: localhost:2181(CONNECTED) 7] create -e /sanguo/wuguo "zhouyu" 
Created /sanguo/wuguo 
```

（2）	创建短暂的带序号的节点 

```xml
[zk: localhost:2181(CONNECTED) 2] create -e -s /sanguo/wuguo 
"zhouyu" 
Created /sanguo/wuguo0000000001 
```

（3）	在当前客户端是能查看到的 

```xml
[zk: localhost:2181(CONNECTED) 3] ls /sanguo  [wuguo, wuguo0000000001, shuguo] 
```

（4）	退出当前客户端然后再重启客户端 

```xml
[zk: localhost:2181(CONNECTED) 12] quit 
[atguigu@hadoop104 zookeeper-3.5.7]$ bin/zkCli.sh 
```

（5）	再次查看根目录下短暂节点已经删除 

```xml
[zk: localhost:2181(CONNECTED) 0] ls /sanguo 
[shuguo] 
```

5）	修改节点数据值 

```xml
[zk: localhost:2181(CONNECTED) 6] set /sanguo/weiguo "simayi" 
```


## 4.4、监听器原理
客户端注册监听它关心的目录节点，当目录节点发生变化（数据改变、节点删除、子目录节点增加删除）时，ZooKeeper 会通知客户端。监听机制保证 ZooKeeper 保存的任何的数据的任何改变都能快速的响应到监听了该节点的应用程序。
**1、监听原理详解** 
- 1）首先要有一个main()线程
- 2）在main线程中创建Zookeeper客户端，这时就会创建两个线
	程，一个负责网络连接通信（connet），一个负责监听（listener）。
- 3）通过connect线程将注册的监听事件发送给Zookeeper。
- 4）在Zookeeper的注册监听器列表中将注册的监听事件添加到列表中。
- 5）Zookeeper监听到有数据或路径变化，就会将这个消息发送给listener线程。
- 6）listener线程内部调用了process()方法。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2505992e5cbd4f55bfd26d644bdfd231.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

**2、常见的监听**
	1）监听节点数据的变化
```xml
get path [watch]
```
	2）监听子节点增减的变化
```xml
ls path [watch]
```

**3、节点的值变化监听** 
（1）	在 hadoop104 主机上注册监听/sanguo 节点数据变化 

```xml
[zk: localhost:2181(CONNECTED) 26] get -w /sanguo  
```

（2）	在 hadoop103 主机上修改/sanguo 节点的数据 

```xml
[zk: localhost:2181(CONNECTED) 1] set /sanguo "xisi" 
```

（3）	观察 hadoop104 主机收到数据变化的监听 

```xml
WATCHER:: WatchedEvent path:/sanguo 	state:SyncConnected 	type:NodeDataChanged 
```

 注意：在hadoop103再多次修改/sanguo的值，hadoop104上不会再收到监听。因为注册一次，只能监听一次。想再次监听，需要再次注册。 
**4、节点的子节点变化监听（路径变化）** 
（1）	在 hadoop104 主机上注册监听/sanguo 节点的子节点变化 

```xml
[zk: localhost:2181(CONNECTED) 1] ls -w /sanguo  
[shuguo, weiguo] 
```

（2）	在 hadoop103 主机/sanguo 节点上创建子节点 

```xml
[zk: localhost:2181(CONNECTED) 2] create /sanguo/jin "simayi" Created /sanguo/jin 
```

（3）	观察 hadoop104 主机收到子节点变化的监听 

```xml
WATCHER:: 
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/sanguo 
```

注意：节点的路径变化，也是注册一次，生效一次。想多次生效，就需要多次注册。 

## 4.5、节点删除与查看 
1）	删除节点 

```xml
[zk: localhost:2181(CONNECTED) 4] delete /sanguo/jin 
```

2）	递归删除节点 

```xml
[zk: localhost:2181(CONNECTED) 15] deleteall /sanguo/shuguo 
```

3）	查看节点状态 

```xml
[zk: localhost:2181(CONNECTED) 17] stat /sanguo cZxid = 0x100000003 
ctime = Wed Aug 29 00:03:23 CST 2018 
mZxid = 0x100000011 
mtime = Wed Aug 29 00:21:23 CST 2018 
pZxid = 0x100000014 
cversion = 9 
dataVersion = 1
aclVersion = 0 
ephemeralOwner = 0x0 
dataLength = 4 
numChildren = 1 
```
# 5、客户端 API 操作 
前提：保证 hadoop102、hadoop103、hadoop104 服务器上 Zookeeper 集群服务端启动。

## 5.1、IDEA 环境搭建
1）	创建一个工程：zookeeper 
2）	添加pom文件 

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
</dependencies> 
```

3）	拷贝log4j.properties文件到项目根目录需要在项目的 src/main/resources 目录下，新建一个文件，命名为“log4j.properties”，在
文件中填入。 

```xml
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```

## 5.2、创建 ZooKeeper 客户端 

```java
public class zkClient {
    // 注意：逗号前后不能有空格
    private static String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";

    private static int sessionTimeout = 2000;
    private ZooKeeper zkClient = null;

    @Before
    public void init() throws Exception {

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {

            public void process(WatchedEvent watchedEvent) {
            }
        });
    }
 }
```

## 5.3、创建子节点 

```java
public class zkClient {
    // 注意：逗号前后不能有空格
    private static String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";

    private static int sessionTimeout = 2000;
    private ZooKeeper zkClient = null;

    @Before
    public void init() throws Exception {

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {

            public void process(WatchedEvent watchedEvent) {
            }
        });
    }

    // 创建子节点
    @Test
    public void create() throws Exception {

        // 参数1：要创建的节点的路径； 参数2：节点数据 ； 参数3：节点权限 ；参数4：节点的类型
        String nodeCreated = zkClient.create("/song1", "song1".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    }
}
```

## 5.4、获取子节点并监听节点变化 

```java
public class zkClient {
    // 注意：逗号前后不能有空格
    private static String connectString = "hadoop102:2181,hadoop103:2181,hadoop104:2181";

    private static int sessionTimeout = 2000;
    private ZooKeeper zkClient = null;

    @Before
    public void init() throws Exception {

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {

            public void process(WatchedEvent watchedEvent) {
                // 收到事件通知后的回调函数（用户的业务逻辑）
                System.out.println(watchedEvent.getType() + "--" + watchedEvent.getPath());

                // 再次启动监听
                try {
                    List<String> children = zkClient.getChildren("/", true);
                    for (String child : children) {
                        System.out.println(child);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
    }
    // 获取子节点
    @Test
    public void getChildren() throws Exception {
        List<String> children = zkClient.getChildren("/", true);
        for (String child : children) {
            System.out.println(child);
        }
        // 延时阻塞
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

## 5.5、判断 Znode 是否存在

```java
    // 判断znode是否存在
    @Test
    public void exist() throws Exception {
        Stat stat = zkClient.exists("/song", false);
        System.out.println(stat == null ? "not exist" : "exist");
    }
```





