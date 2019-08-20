# Zookeeper 开发指南

这篇文件，会讲解Zk独立模式、复制模式、概念，目录：

[Zk 单节点搭建](#Zk-单节点搭建)

[Zookeeper 概念](#Zookeeper-概念)

## Zk 单节点搭建

### 下载

[点击前往下载地址](https://www.apache.org/dyn/closer.cgi/zookeeper/), Zk的启动依赖Java环境，并且必须是Java1.7及以上([点击去下载jdk](https://www.oracle.com/technetwork/java/javase/downloads/index.html))。

下载好Zk的压缩包之后，进行解压:

```shell
$ sudo tar -zxvf path/zk.tar.gz -C /usr/local/zookeeper
```

设置环境变量:

```shell
$ sudo vim /etc/profile

export ZKHOME=/usr/local/zookeeper
export PATH=${PATH}:${ZKHOME}/bin

$ sudo source /etc/profile
```

测试一下:

```shell
zkServer.sh status

ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.
```

如果是出现的类似上面的输出，则可以继续往下看。(切换到root)

### 单节点运行

一个节点的Zk适合开发使用，对于生产，推荐使用的是奇数个节点，并且最少3个。

启动Zk，我们需要一份配置文件，运行：

```shell
vim /usr/local/zookeeper/config/zoo.cfg

tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```

文件名是可以叫任何名字的，但是在这里先不讨论，请命名为: `zoo.cfg`。我们来解释一下这几个参数的意思:

- tickTime: Zk的时间单位为毫秒(ms), 用于心跳检测，最小的会话超时时间是两倍心跳时间。
- dataDir: 存储内存数据库快照的位置，除非有别的说明，否则为更新数据库的事务日志。
- clientPort: Zk监听客户端连接的端口。

创建完配置文件之后，就可以启动单节点的Zk服务了:

```shell
zkServer.sh start

ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

查看状态:

```shell
zkServer.sh status

ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: standalon
```

可以看到，现在zk是处于 单节点的模式(Mode:standalon)。当然，独立模式适合绝大部分开发测试使用，但是如果要运行在生产环境，必须要在外部去管理Zk的存储(`dataDir`和`logs`)。

### 连接到Zookeeper

连接Zk:

```shell
zkCli.sh -server localhost:2181
```

如果连接成功，那么你应该进入Zk的命令行界面。

在shell中，输入 `help` 可以获取客户端可执行的命令列表:

```shell
help 

ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port

```

我们进行一些简单的尝试来感觉这个命令界面，首先从list开始，查看当前Zk的存储节点：

```shell
ls /

[zookeeper]
```
这说明在根节点上存在一个子节点，子节点的名字叫 `zookeeper`；接下来我们创建一个叫 `zk_test`的节点，并且携带数据`my_data`:

```shell
create /zk_test my_data

Created /zk_test

# 注意，所有的节点必须是以根(/)开始，如果创建znode的时候，没有携带/，则报错，比如

create zk_test my_data

Command failed: java.lang.IllegalArgumentException: Path must start with / character
```

再发出一个 `ls /` 命令看看发生了什么：

```shell
ls /

[zookeeper, zk_test]
```

请注意，现在已经创建了 `zk_test` 节点。

接下来，我们使用 `get` 命令去验证刚刚创建znode时携带的数据是否和znode关联:

```shell
get /zk_test

my_data
cZxid = 0x4
ctime = Tue Aug 20 15:48:02 CST 2019
mZxid = 0x4
mtime = Tue Aug 20 15:48:02 CST 2019
pZxid = 0x4
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 7
numChildren = 0
```

我们还可以使用 `set` 更改这个znode的数据：

```shell
set /zk_test my_data_change

cZxid = 0x4
ctime = Tue Aug 20 15:48:02 CST 2019
mZxid = 0x5
mtime = Tue Aug 20 16:02:16 CST 2019
pZxid = 0x4
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 14
numChildren = 0

```
(经过get再次验证，发现set后数据是被更改了的)。

那么，经过一些简单的感受，我们是成功的启动一个单实例Zk。但是我们先不急着讲复制模式。我们先来了解Zk的一些概念。

## Zookeeper 概念

[Zookeeper的数据模型](#Zookeeper的数据模型)

[Znodes](#Znodes)

[Zookeeper中的时间](#Zookeeper中的时间)

[ZooKeeper Stat Structure](#ZooKeeper-Stat-Structure)

### Zookeeper的数据模型

Zk是有一个有层级的命名空间，就比如分布式文件系统。唯一有区别的就是命名空间的每一个节点都可以包含与之关联的数据和子节点。这就像拥有一个文件也是目录的文件系统。节点的路径始终是绝对路径，并且以斜杠(/)分隔，相关路径和引用Zk将无法识别。除了以下的约束，任何Unicode字符都可以用在路径中:

1. `null`字符(\u0000)不能成为路径名字的一部分(会导致C的绑定出现问题)
2. 下面几个字符不能使用，因为他们有时候会以一种令人很困惑的方式显示: \u0001 - \u001F and \u007F
3. \u009F
4. 下面的字符不允许:  \ud800 - uF8FF, \uFFF0 - uFFFF
5. 字符"."可以做为路径名字组成的一部分，但是不能单独使用"."或者".."表示路径的节点，比如/a/b/./c是无效的，/a/b/.c是有效的
6. `zookeeper` 是保留字，/zookeeper默认已经存在了。

### Znodes

Zookeeper树中的每个节点都叫Znode。Znodes维护一个stat结构，这个stat结构包含了数据更改的版本号、ACL的变化、时间戳。版本号和时间戳共同作用使Zk可以验证缓存和协调更新，每一次节点数据变化，版本号都会加1，比如，任何时候一个客户端从Zk取数据，同时也会拿到这个数据的版本，当客户端想要对这个节点执行更新数据或者删除的时候，客户端必须提供它正在改变的znode的数据的版本，如果提供的版本和数据的实际版本不一致，则更新会失败（当然，我们也可以覆盖版本）。

**Note**

> 在分布式系统中，'node' 可以值通用主机、服务器、集合成员、客户端进程等等，但是在Zk中，znode指的是数据节点，Servers才是构成Zk服务的主机，client指的是使用Zk服务的任何主机或者进程。

znode是开发人员访问的最主要的功能，它有几个值得一提的特征:

#### Watches

客户端可以在znodes上设置监听，改变znode会触发监听机制并移除该监听，当监听触发时，Zk会给客户端发送一个通知。更多信息在[Zk Watches](#Zk-Watcher)。

#### 数据访问(Data Access)

在一个命名空间的每个znode的数据存储不管是写入还是读取都是原子的。读取的时候，获取znode的所有数据，写入的时候，替换znode的所有数据。每个节点都有一个准入控制列表(ACL: Access Control List)控制谁可以做什么。

ZK并非设计为通用数据库或者大对象存储服务。相反，它管理协调数据，数据可以是配置、状态信息、集合点等形式。各种各样的数据有一个共同的属性就是他们都很小：以千字节为标准(1M,强制性)。Zookeeper客户端和服务端实现了健康检查确保znode数据小于1M，但是数据应该远低于平均数据。操作较大的数据将导致一些操作花费更多的时间，并且会影响一些操作的延迟，因为在网络和存储媒介中移动更多的数据将需要额外的时间。如果需要存储大数据，通常的处理是把数据存储在一个大容量存储系统中，并把存储位置的指针存储到ZooKeeper上。

#### 临时节点(Ephemeral Nodes)

ZooKeeper也临时节点的概念。这些znodes存活的时间和创建这个节点的会话有效期是一样的。当会话结束，节点被删除。因为这种临时节点的特性，临时节点不允许有子节点

#### 顺序节点-命名唯一(Sequence Nodes -- Unique Naming)

当创建一个节点的时候，也可以请求ZooKeeper在路径后面增加一个自增的计数器。对父节点来说，这个计数器是 唯一的。计数器是%010d的格式——是一个十位数，比如：<path>0000000001。

注意：这个计数器用来存储下一个序列号的是一个四字节的数，当增加到2147483647后，计数器将会溢出(结果是"-2147483648")。

#### 容器节点(Container Nodes)

**Added in 3.5.3**

容器节点是为leader、lock等操作存在的，当容器节点的最后一个孩子节点被删除之后，容器节点将被标注并在一段时间后删除。 
由于容器节点的该特性，当你在容器节点下创建一个孩子节点时，应该始终准备捕获KeeperException.NoNodeException，如果捕获了该异常，则需要重新创建容器节点。

#### TTL 节点

**Added in 3.5.3**

当创建永久节点或者永久顺序节点的时候，可以为znode设置ttl(单位:ms)，在ttl时间内如果没有更改znode并且该znode没有子节点，则服务端在未来某个时间将删除这个znode。

### Zookeeper中的时间

Zk有多种方式跟踪时间:

#### Zxid

Zookeeper每次改变状态都会接受一个`zxid`(Zookeeper Transaction Id)形式的标记，他暴露了Zk的所有变化的顺序，每次变更都会生成唯一的zxid，如果zxid1小于zxid2则说明zxid1先于zxid2发生

#### Version numbers

节点的每次变化都会引起这个节点版本号之一的一次增加。这三个版本号是：version（一个节点的数据变化次数），cversion（一个节点的子节点变化次数），aversion（一个节点的ACL 变化次数）。

#### Tricks

当使用多个ZooKeeper服务，服务器使用ticks来确定事件的时间，比如说状态上传、会话超时、连接超时等。这个tick时间仅仅通过最小会话超时时间间接的暴露出来；如果一个客户端请求会话的超时时间小于最小超时时间，服务器将会告诉客户端实际的会话超时时间是最小超时时间。

#### real time

ZooKeeper不使用实时、时钟时间。除了把时间戳放在stat结构中。

### ZooKeeper Stat Structure

Zk中每个znode的Stat结构由以下字段组成:

- **czxid**: 该节点被创建时候的事务ID
- **mzxid**: 该节点最后被修改时候的事务ID
- **pzxid**: 该节点的子节点最后被修改时候的事务ID
- **ctime**: 该节点被创建的时间
- **mtime**: 该节点最后被修改的时间
- **version**: znode数据的版本号,即变化次数(dataVersion)
- **cversion**: 该节点的子节点变化次数
- **aversion**: 该节点ACL改变次数(aclVersion)
- **ephemeralOwner**: 如果该节点是临时节点，则表示创建该节点的会话id，否则是0
- **dataLength**: 这个节点的数据长度
- **numChildren**: 该节点的子节点个数