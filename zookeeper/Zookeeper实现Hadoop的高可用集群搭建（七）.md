# 1、Hadoop HA 高可用 
## 1.1、HDFS-HA 核心问题
### 1.1.1、 怎么保证三台 namenode 的数据一致 
   - Fsimage:让一台 nn 生成数据,让其他机器 nn 同步 
 - Edits:需要引进新的模块 JournalNode 来保证 edtis 的文件的数据一致性 
### 1.1.2、 怎么让同时只有一台 nn 是 active，其他所有是 standby 的 
  - 手动分配 
- .自动分配 
### 1.1.3、 2nn 在 ha 架构中并不存在，定期合并 fsimage 和 edtis 的活谁来干 
 -   由 standby 的 nn 来干 
### 1.1.4、 如果 nn 真的发生了问题，怎么让其他的 nn 上位干活 
 - 手动故障转移 
 - 自动故障转移 

## 1.2、HDFS-HA 集群搭建
HA 的主要目的是消除 namenode 的单点故障,需要将 hdfs 集群规划成以下模样 
![在这里插入图片描述](https://img-blog.csdnimg.cn/1a695ff20ad94fba99136515dd9f7781.png)
### 1.2.1、HDFS-HA 手动模式
#### 1.2.1.1、环境准备 
（1）	修改 IP 
（2）	修改主机名及主机名和 IP 地址的映射 
（3）	关闭防火墙 
（4）	ssh 免密登录 
（5）	安装 JDK，配置环境变量等 

[查看详情查看之前的文章](https://blog.csdn.net/prefect_start/article/details/117624615)

#### 1.2.1.2、配置 HDFS-HA 集群
1）	官方地址：http://hadoop.apache.org/ 
2）	在 opt 目录下创建一个 ha 文件夹 

```java
[song@hadoop102 ~]$ cd /opt 
[song@hadoop102 opt]$ sudo mkdir ha 
[song@hadoop102 opt]$ sudo chown song:song /opt/ha 
```
3）	将/opt/module/下的 hadoop-3.1.3 拷贝到/opt/ha 目录下（记得删除 data 和 log 目录） 

```java
[song@hadoop102 opt]$ cp -r /opt/module/hadoop-3.1.3 /opt/ha/ 
```
4）配置 core-site.xml 

```xml
<configuration> 
<!-- 把多个NameNode的地址组装成一个集群mycluster --> 
  <property> 
    <name>fs.defaultFS</name> 
    <value>hdfs://mycluster</value> 
  </property> 
<!-- 指定hadoop运行时产生文件的存储目录 --> 
  <property> 
    <name>hadoop.tmp.dir</name> 
    <value>/opt/ha/hadoop-3.1.3/data</value> 
  </property>
</configuration> 
```
5）配置 hdfs-site.xml 

```xml
<configuration> 
<!-- NameNode数据存储目录 --> 
  <property> 
    <name>dfs.namenode.name.dir</name> 
    <value>file://${hadoop.tmp.dir}/name</value> 
  </property> 
<!-- DataNode数据存储目录 --> 
  <property> 
    <name>dfs.datanode.data.dir</name> 
    <value>file://${hadoop.tmp.dir}/data</value> 
  </property> 
<!-- JournalNode数据存储目录 --> 
  <property> 
    <name>dfs.journalnode.edits.dir</name> 
    <value>${hadoop.tmp.dir}/jn</value> 
  </property> 
<!-- 完全分布式集群名称 --> 
  <property> 
    <name>dfs.nameservices</name> 
    <value>mycluster</value> 
  </property> 
<!-- 集群中NameNode节点都有哪些 --> 
  <property> 
    <name>dfs.ha.namenodes.mycluster</name> 
    <value>nn1,nn2,nn3</value>   </property> 
<!-- NameNode的RPC通信地址 --> 
  <property> 
    <name>dfs.namenode.rpc-address.mycluster.nn1</name> 
    <value>hadoop102:8020</value> 
  </property> 
  <property> 
    <name>dfs.namenode.rpc-address.mycluster.nn2</name> 
    <value>hadoop103:8020</value> 
  </property> 
  <property> 
    <name>dfs.namenode.rpc-address.mycluster.nn3</name> 
    <value>hadoop104:8020</value> 
  </property> 
<!-- NameNode的http通信地址 --> 
  <property> 
    <name>dfs.namenode.http-address.mycluster.nn1</name> 
    <value>hadoop102:9870</value> 
  </property> 
  <property> 
    <name>dfs.namenode.http-address.mycluster.nn2</name> 
    <value>hadoop103:9870</value> 
  </property> 
  <property> 
    <name>dfs.namenode.http-address.mycluster.nn3</name> 
    <value>hadoop104:9870</value> 
  </property> 
<!-- 指定NameNode元数据在JournalNode上的存放位置 --> 
  <property> 
<name>dfs.namenode.shared.edits.dir</name> 
<value>qjournal://hadoop102:8485;hadoop103:8485;hadoop104:8485/mycluster</value> 
  </property> 
<!-- 访问代理类：client用于确定哪个NameNode为Active --> 
  <property> 
    <name>dfs.client.failover.proxy.provider.mycluster</name>     
	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value> 
  </property> 
<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 --> 
  <property> 
    <name>dfs.ha.fencing.methods</name> 
    <value>sshfence</value> 
  </property> 
<!-- 使用隔离机制时需要ssh秘钥登录--> 
  <property> 
    <name>dfs.ha.fencing.ssh.private-key-files</name> 
    <value>/home/atguigu/.ssh/id_rsa</value> 
  </property> 
</configuration> 
```
6）分发配置好的 hadoop 环境到其他节点

#### 1.2.1.3、启动 HDFS-HA 集群
1）将 HADOOP_HOME 环境变量更改到 HA 目录(三台机器) 

```xml
[song@hadoop102 ~]$ sudo vim /etc/profile.d/my_env.sh 
```

将 HADOOP_HOME 部分改为如下 
```shell
#HADOOP_HOME 
export HADOOP_HOME=/opt/ha/hadoop-3.1.3 
export PATH=$PATH:$HADOOP_HOME/bin 
export PATH=$PATH:$HADOOP_HOME/sbin 
```
去三台机器上 source 环境变量 
```java
[song@hadoop102 ~]$source /etc/profile 
[song@hadoop103 ~]$source /etc/profile 
[song@hadoop104 ~]$source /etc/profile 
```

2）在各个 JournalNode 节点上，输入以下命令启动 journalnode 服务 

```xml
[song@hadoop102 ~]$ hdfs --daemon start journalnode 
[song@hadoop103 ~]$ hdfs --daemon start journalnode 
[song@hadoop104 ~]$ hdfs --daemon start journalnode 
```

3）在[nn1]上，对其进行格式化，并启动 

```xml
[song@hadoop102 ~]$ hdfs namenode -format 
[song@hadoop102 ~]$ hdfs --daemon start namenode 
```

4）在[nn2]和[nn3]上，同步 nn1 的元数据信息 

```xml
[song@hadoop103 ~]$ hdfs namenode -bootstrapStandby 
song@hadoop104 ~]$ hdfs namenode -bootstrapStandby 
```

5）启动[nn2]和[nn3] 

```xml
[song@hadoop103 ~]$ hdfs --daemon start namenode
[song@hadoop104 ~]$ hdfs --daemon start namenode 
```
6）查看web页面

![在这里插入图片描述](https://img-blog.csdnimg.cn/4eae1863ae264af39e3332af5816525c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/9d6e2c4616e24131b78c8d309bad61fa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/562f2a0a8c884150bc72892485158670.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
7）在所有节点上，启动 datanode

```xml
[song@hadoop102 ~]$ hdfs --daemon start datanode
[song@hadoop103 ~]$ hdfs --daemon start datanode
[song@hadoop104 ~]$ hdfs --daemon start datanode
```

8）将[nn1]切换为 Active

```xml
[song@hadoop102 ~]$ hdfs haadmin -transitionToActive nn1
```

9）查看是否 Active

```xml
[song@hadoop102 ~]$ hdfs haadmin -getServiceState nn1
```

### 1.2.2、HDFS-HA 自动模式
#### 1.2.2.1、HDFS-HA 自动故障转移工作机制
自动故障转移为 HDFS 部署增加了两个新组件：ZooKeeper 和 ZKFailoverController（ZKFC）进程，如图所示。ZooKeeper 是维护少量协调数据，通知客户端这些数据的改变和监视客户端故障的高可用服务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/e61d7044902c47b38dea0837409577a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.2.2.2、HDFS-HA 自动故障转移的集群规划
![在这里插入图片描述](https://img-blog.csdnimg.cn/22b4f2b47edf4a5ea60e7ae49d159a4e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.2.2.3、配置 HDFS-HA 自动故障转移
**1）	具体配置** 

（1）	在 hdfs-site.xml 中增加 

```xml
<!-- 启用nn故障自动转移 --> 
<property> 
 <name>dfs.ha.automatic-failover.enabled</name> 
 <value>true</value> 
</property> 
```

（2）	在 core-site.xml 文件中增加 

```xml
<!-- 指定zkfc要连接的zkServer地址 --> 
<property> 
 <name>ha.zookeeper.quorum</name> 
 <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value> 
</property> 
```

（3）	修改后分发配置文件 

```xml
[song@hadoop102 etc]$ pwd 
/opt/ha/hadoop-3.1.3/etc 
[song@hadoop102 etc]$ xsync hadoop/ 
```
**2）	启动** 
（1）	关闭所有 HDFS 服务： 

```xml
[song@hadoop102 ~]$ stop-dfs.sh 
```

（2）	启动 Zookeeper 集群： 

```xml
[song@hadoop102 ~]$ zkServer.sh start
[song@hadoop103 ~]$ zkServer.sh start 
[song@hadoop104 ~]$ zkServer.sh start 
```

（3）	启动 Zookeeper 以后，然后再初始化 HA 在 Zookeeper 中状态： 

```xml
[song@hadoop102 ~]$ hdfs zkfc -formatZK 
```

（4）	启动 HDFS 服务： 

```xml
[song@hadoop102 ~]$ start-dfs.sh 
```

（5）	可以去 zkCli.sh 客户端查看 Namenode 选举锁节点内容： 

```xml
[zk: localhost:2181(CONNECTED) 7] get -s 
/hadoop-ha/mycluster/ActiveStandbyElectorLock 
  myclusternn2 hadoop103 >(> 
cZxid = 0x10000000b 
ctime = Tue Jul 14 17:00:13 CST 2020 mZxid = 0x10000000b 
mtime = Tue Jul 14 17:00:13 CST 2020 pZxid = 0x10000000b cversion = 0 dataVersion = 0 aclVersion = 0 
ephemeralOwner = 0x40000da2eb70000 dataLength = 33 numChildren = 0 
```

**3）	验证** 
（1）	将 Active NameNode 进程 kill，查看网页端三台 Namenode 的状态变化 

```xml
[song@hadoop102 ~]$ kill -9 namenode的进程id 
```

## 1.3、YARN-HA 配置
### 1.3.1、YARN-HA 工作机制
![在这里插入图片描述](https://img-blog.csdnimg.cn/568392d51a644192917461f5b69d7d21.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.3.2、配置 YARN-HA 集群
#### 1.3.2.1、环境准备
（1）修改 IP
（2）修改主机名及主机名和 IP 地址的映射
（3）关闭防火墙
（4）ssh 免密登录
（5）安装 JDK，配置环境变量等
（6）配置 Zookeeper 集群

#### 1.3.2.2、规划集群
![在这里插入图片描述](https://img-blog.csdnimg.cn/d756c02a076349a1af0b6a749fa9e34f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.3.2.3、核心问题
- 如果当前 active rm 挂了，其他 rm 怎么将其他 standby rm 上位核心原理跟 hdfs 一样，利用了 zk 的临时节点
- 当前 rm 上有很多的计算程序在等待运行,其他的 rm 怎么将这些程序接手过来接着跑 rm 会将当前的所有计算程序的状态存储在 zk 中,其他 rm 上位后会去读取，然后接着跑

#### 1.3.2.4、具体配置
（1）	yarn-site.xml 

```xml
<configuration> 
 
    <property> 
        <name>yarn.nodemanager.aux-services</name> 
        <value>mapreduce_shuffle</value> 
    </property> 
 
    <!-- 启用resourcemanager ha --> 
    <property> 
        <name>yarn.resourcemanager.ha.enabled</name> 
        <value>true</value> 
    </property> 
  
    <!-- 声明两台resourcemanager的地址 --> 
    <property> 
        <name>yarn.resourcemanager.cluster-id</name> 
        <value>cluster-yarn1</value> 
    </property> 
    <!--指定resourcemanager的逻辑列表--> 
    <property> 
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2,rm3</value> 
</property> 
<!-- ========== rm1的配置 ========== --> 
<!-- 指定rm1的主机名 --> 
    <property> 
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hadoop102</value> 
</property> 
<!-- 指定rm1的web端地址 --> 
<property> 
     <name>yarn.resourcemanager.webapp.address.rm1</name> 
     <value>hadoop102:8088</value> 
</property> 
<!-- 指定rm1的内部通信地址 --> 
<property> 
     <name>yarn.resourcemanager.address.rm1</name> 
     <value>hadoop102:8032</value> 
</property> 
<!-- 指定AM向rm1申请资源的地址 --> 
<property> 

     <name>yarn.resourcemanager.scheduler.address.rm1</name>   
     <value>hadoop102:8030</value> 
</property> 
<!-- 指定供NM连接的地址 -->   
<property> 
     <name>yarn.resourcemanager.resource-tracker.address.rm1</name> 
     <value>hadoop102:8031</value> 
</property> 
<!-- ========== rm2的配置 ========== --> 
    <!-- 指定rm2的主机名 --> 
    <property> 
        <name>yarn.resourcemanager.hostname.rm2</name> 
        <value>hadoop103</value> 
</property> 
<property> 
     <name>yarn.resourcemanager.webapp.address.rm2</name> 
     <value>hadoop103:8088</value> 
</property> 
<property> 
     <name>yarn.resourcemanager.address.rm2</name> 
     <value>hadoop103:8032</value> 
</property> 
<property> 
     <name>yarn.resourcemanager.scheduler.address.rm2</name> 
     <value>hadoop103:8030</value> 
</property> 
<property> 
     <name>yarn.resourcemanager.resource-tracker.address.rm2</name> 
     <value>hadoop103:8031</value> 
</property> 
 <!-- ========== rm3的配置 ========== --> 
<!-- 指定rm1的主机名 --> 
    <property> 
        <name>yarn.resourcemanager.hostname.rm3</name>         <value>hadoop104</value> 
</property> 
<!-- 指定rm1的web端地址 --> 
<property> 
     <name>yarn.resourcemanager.webapp.address.rm3</name> 
     <value>hadoop104:8088</value> 
</property> 
<!-- 指定rm1的内部通信地址 --> 
<property> 
     <name>yarn.resourcemanager.address.rm3</name> 
     <value>hadoop104:8032</value> 
</property> 
<!-- 指定AM向rm1申请资源的地址 --> 
<property> 
     <name>yarn.resourcemanager.scheduler.address.rm3</name>   
     <value>hadoop104:8030</value> 
</property> 
<!-- 指定供NM连接的地址 -->   
<property> 
     <name>yarn.resourcemanager.resource-tracker.address.rm3</name> 
     <value>hadoop104:8031</value> 
</property> 
    <!-- 指定zookeeper集群的地址 -->  
    <property> 
        <name>yarn.resourcemanager.zk-address</name> 
        <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value> 
    </property> 
 
    <!-- 启用自动恢复 -->  
    <property> 
        <name>yarn.resourcemanager.recovery.enabled</name> 
        <value>true</value> 
    </property> 
  
    <!-- 指定resourcemanager的状态信息存储在zookeeper集群 -->  
    <property> 
        <name>yarn.resourcemanager.store.class</name>     
		<value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
 </property> 
<!-- 环境变量的继承 --> 
 <property> 
        <name>yarn.nodemanager.env-whitelist</name>         
		<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE
,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME
		</value> 
</property> 
 
</configuration> 

```
（2）	同步更新其他节点的配置信息，分发配置文件 

```xml
[song@hadoop102 etc]$ xsync hadoop/ 
```

#### 1.3.2.5、启动 YARN  
（1）	在 hadoop102 或者 hadoop103 中执行： 

```xml
[song@hadoop102 ~]$ start-yarn.sh 
```

（2）	查看服务状态 

```xml
[song@hadoop102 ~]$ yarn rmadmin -getServiceState rm1 
```

（3）	可以去 zkCli.sh 客户端查看 ResourceManager 选举锁节点内容： 

```xml
[song@hadoop102 ~]$ zkCli.sh 
[zk: localhost:2181(CONNECTED) 16] get -s 
/yarn-leader-election/cluster-yarn1/ActiveStandbyElectorLock 
 
cluster-yarn1rm1 cZxid = 0x100000022 
ctime = Tue Jul 14 17:06:44 CST 2020 mZxid = 0x100000022 
mtime = Tue Jul 14 17:06:44 CST 2020 pZxid = 0x100000022 cversion = 0 dataVersion = 0 aclVersion = 0 
ephemeralOwner = 0x30000da33080005 dataLength = 20 numChildren = 0 
```
（4）	web 端查看 hadoop102:8088 和 hadoop103:8088 的 YARN 的状态
![在这里插入图片描述](https://img-blog.csdnimg.cn/88dce4c7bd064ef79d5a0cd7920faafa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 1.4、HADOOP HA 的最终规划
![在这里插入图片描述](https://img-blog.csdnimg.cn/26e8cbda2053480e846fe456a59f6c3f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/67ccaf0cf2c849398e6e03ecc9300985.png)


