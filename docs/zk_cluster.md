# Zookeeper 集群

对于要提供可靠的 `Zookeeper` 服务，那么应该以集群模式部署 Zookeeper 服务。只要半数以上的集群节点在线，那么服务就是可用的。因为Zk服务可用需要超过半数以上(n/2 + 1)的节点，所以Zk的集群最好是使用奇数个机器。比如：对于四个机器的Zk集群，只能处理单台机器的故障，如果两台机器故障，则剩下的两台不能占半数以上(3)了。但是，如果有了5个机器，Zk可以处理2个机器的故障。

所以，对于生产环境，3个Zk服务器是推荐的最小的配置，并且还建议它运行在不同的机器上。


> **Note** 通常来说，三台服务器对于生产环境应该足够了，但是为了在维护期间也要获得最大的可靠性，你可以安装五台服务器。使用三台服务器时，如果你拿一台服务器去维护，则在维护期间，剩下两台服务器可能会发生故障导致Zk不可用，如果有5台，拿一台去维护，剩下的再故障一台也能提供Zk服务。

通常，集群有两种角色：`leader` 和 `follower`，但是为了快速拓展集群，还有一个可以启用的角色 : [Observer](#observer)。

## 搭建

下面开始搭建Zk集群。Zk集群最新版3.5需要java7及以上的环境。这里的话演示在一台机器上，根据端口不同的划分搭建三个服务端。

|   服务  | 客户端端口 | 服务通信端口 | leader选举端口|
| :-----:| :--------:| :--------: | :----------:|
| server1|    2181   |    2888    |     3888    |
| server2|    2182   |    2889    |     3889    |
| server3|    2183   |    2890    |     3890    |

上面是三个服务的端口划分。客户端端口是用来客户端连接的；服务通信端口提供各个实例间通信，leader才会监听此端口，follower从此端口同步信息；leader选举端口是为了重新选举leader通信用的。

创建配置文件:

```conf
# zoo_server1.cfg
tickTime=2000
dataDir=/tmp/zookeeper/server1
clientPort=2181
initLimit=5
syncLimit=2
admin.enableServer=false # 禁用AdminServer
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890

# zoo_server2.cfg
tickTime=2000
dataDir=/tmp/zookeeper/server2
clientPort=2182
initLimit=5
syncLimit=2
admin.enableServer=false # 禁用AdminServer
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890

# zoo_server3.cfg
tickTime=2000
dataDir=/tmp/zookeeper/server3
clientPort=2183
initLimit=5
syncLimit=2
admin.enableServer=false # 禁用AdminServer
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

每个服务的配置都需要知道每个服务的地址: `server.x=ip:server-port:leader-election-port`，`x` 是每个服务的id，这个id由 `myid` 文件指定，`myid` 是一个文本文件，每个服务器对应一个文件，存放在 `dataDir` 目录，文件内容只包含这个服务的id，id在集群中是唯一的，其值应该介于1-255之间。**重要信息**: 如果启用ttl节点等拓展功能，由于内部限制，id必须介于1-254之间。

创建myid:

```shell
mkdir -p /tmp/zookeeper/server{1,2,3}

echo 1 > /tmp/zookeeper/server1/myid

echo 2 > /tmp/zookeeper/server2/myid

echo 3 > /tmp/zookeeper/server3/myid
```

接下来一个一个启动，先启动server1：

```shell
zkServer.sh start ./conf/zoo_server1.cfg
```

后面跟的是server1配置的地址，如果没有指定，默认使用的是 `zoo.cfg`。启动server1后使用 `zkServer.sh status ./conf/zoo_server1.cfg` 发现提示可能没有运行:

```shell
zkServer.sh status conf/zoo_server1.cfg
ZooKeeper JMX enabled by default
Using config: conf/zoo_server1.cfg
Client port found: 2181. Client address: localhost.
Error contacting service. It is probably not running.

# 查看日志
Cannot open channel to 3 at election address localhost/127.0.0.1:3890
java.net.ConnectException: 拒绝连接 (Connection refused)

Cannot open channel to 2 at election address localhost/127.0.0.1:3889
java.net.ConnectException: 拒绝连接 (Connection refused)
```
所以不用着急，这是因为以集群模式启动，现在只启动了一个服务，然后我们指定的是3个服务，不到半数以上，集群是不可能提供服务的，并且日志上说和另外两个服务的选举端口连接不上，一直不能发生选举，导致服务不可用。

现在启动server2:

```shell
zkServer.sh start conf/zoo_server2.cfg
```

查看server2的状态，发现server2变成了leader，而server1则是follower：

```shell
zkServer.sh status conf/zoo_server2.cfg

ZooKeeper JMX enabled by default
Using config: conf/zoo_server2.cfg
Client port found: 2182. Client address: localhost.
Mode: leader

zkServer.sh status conf/zoo_server1.cfg
ZooKeeper JMX enabled by default
Using config: conf/zoo_server1.cfg
Client port found: 2181. Client address: localhost.
Mode: follower
```

这里有一个点，为什么server2是leader而不是server1是leader，明明server1比server2更早启动？这里涉及到选举的知识和算法，后面会讲。

按照同样的方法启动server3，发现server3也是follower。

我们可以稍微验证一下集群的可靠性。

我们刚刚看到，server2是leader，我们连接到server2，创建一个节点：

```shell
[zk: localhost:2182(CONNECTED) 1] create /leader-create "server2 is leader"
Created /leader-create
[zk: localhost:2182(CONNECTED) 2] get /leader-create
server2 is leader
```

然后我们停止server2，通过查看状态，发现server3变成了leader，server1还是follower，我们连接server1，查看之前的leader server2创建的节点存不存在:

```shell
[zk: localhost:2181(CONNECTED) 0] ls /
[leader-create, zookeeper]
[zk: localhost:2181(CONNECTED) 1] get /leader-create
server2 is leader
```

显然，很明显的获取到了数据，在一般的zk 客户端，通常我们是使用 `ip:port,ip2:port,ip3:port/chroot`，其实我们使用集群中的一个ip:port就可以自动的服务发现，但是写一个是不推荐的，因为你并不能确定某个故障的服务恰好不是你写的连接，不然就会造成单点故障，所以写多个是必要的，因为会有重连功能。

集群搭建就到了这里，后面会讲一些运维和实践。

上次在讲客户端命令的时候，遗留了两个命令没有讲。(这里依然没有讲清楚，后面会再涉及到)

### config

用于打印当前集群的配置。用法: `config [-c] [-w] [-s]`

```shell
config -s 
server.1=localhost:2888:3888:participant
server.2=localhost:2889:3889:participant
server.3=localhost:2890:3890:participant
version=200000000
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x200000000
mtime = Tue Aug 27 19:43:20 CST 2019
pZxid = 0x0
cversion = 0
dataVersion = -1
aclVersion = -1
ephemeralOwner = 0x0
dataLength = 140
numChildren = 0
```

`-s` 表示输出 `stat` 的统计信息，`-c` 表示只输出版本号和当前配置的连接，`-w` 增加一个监听器。

### reconfig

用法: 

```shell
reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
```

`-s` 打印stat，`-v` 指定版本对了的才能修改成功。

```shell
reconfig -add server.4=localhost:2891:3891:observer;2184

reconfig -members server.1=125.23.63.23:2780:2783:participant;2791,server.2=125.23.63.24:2781:2784:participant;2792,server.3=125.23.63.25:2782:2785:participant;2793

reconfig -remove 3 -add server.5=125.23.63.23:1234:1235;1236

 reconfig -remove 3,4 -add server.5=localhost:2111:2112;2113,6=localhost:2114:2115:observer;2116

 reconfig -file newconfig.cfg ///newconfig.cfg 是动态配置文件
```

## observer

一个集群只能有一个 `leader`，可以有多个 `follower`，但是当我们需要拓展集群的时候，如果我们只能添加 `follower`，那么写入性能会降低，因为写操作需要集群半数以上的节点写成功则写操作成功，因此随着更多选民的增加，投票成本就会相应增加。

Zk 引入了一种名为 `Observer` 的新型 Zk 服务端，它有助于解决这个问题并且进一步的提高 Zk 的可拓展性。观察者是集群中的非投票成员，它只听取投票结果，而不参与投票，除了这个区别，`observer` 和 `follower` 完全相同，观察者可以接收读写请求，并和 `follower` 一样，将写请求转发给 `leader`。由于观察者不参与投票，那么 `leader` 写操作的时候，半数以上的节点并不包括 `observer`，所以可以在不影响投票性能的前提下，可以增加观察者的数量以提高读性能。

观察者还有其他的有点。因为他们不投票，所以他们不是Zk集群的重要组成部分，所以，那么观察者断开连接、宕机也不会影响整个集群的可用性。对用户的好处是，`observer` 可以用比 `follower` 更不可靠的网络连接。实际上，`Observers` 可能由于与另外一个数据中心的Zk服务器通信，对于连接到Observer的客户端来说，读取可能会更快速，写也只会花费少量的网络流量，因为在没有投票的情况下所需要的消息会更少。

把一个服务以 `observer` 运行，需要修改配置文件:

```shell
peerType=observer

server.x=localhost:2181:3181:observer
```

`peerType` 说明以observer的身份运行，`server.x=localhost:2181:3181:observer`则告诉集群中的其它成员，x 是一个观察者，不应该期望它能进行投票。

## 集群怎么处理读写？

现在我们有一个集群，集群中有三个角色 `leader`、`observer`、`follower`。

那么，客户单的读写请求，集群是怎么处理的呢？

我们分几种情况来说明。

### 读 observer

对于客户端发送到 `observer` 的读请求，由 `observer` 直接响应客户端

### 读 follower

对于客户端发送到 `follower` 的读请求，由 `follower` 直接响应客户端

### 读 leader

对于客户端发送到 `leader` 的读请求，由 `leader` 直接响应客户端

### 写 observer

对于客户端发送到 `observer` 的写请求，`observer` 会将请求转发到 `leader`，`leader` 对集群中的 `follower` 广而告之，超过半数的 `follower` 响应 `leader`，`leader` 写成功，响应 `observer`，`observer` 响应客户端。

### 写 follower

对于客户端发送到 `follower` 的写请求，`follower` 会将请求转发到 `leader`，`leader` 对集群中的 `follower` 广而告之，超过半数的 `follower` 响应 `leader`，`leader` 写成功，响应 `follower`，`follower` 响应客户端。

### 写 leader

对于客户端发送到 `leader` 的写请求，`leader` 对集群中的 `follower` 广而告之，超过半数的 `follower` 响应 `leader`，`leader` 写成功，响应客户端。


所以，对于Zk来说，读请求不管哪个服务收到都直接返回，写请求为了一致性都交由 `leader` 来写。那么Zk是怎么保持一致性的呢？我们会在 [Zookeeper的zab协议](../docs/zab.md) 中讲到。