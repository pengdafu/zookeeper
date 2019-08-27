# Zookeeper 集群搭建

对于要提供可靠的 `Zookeeper` 服务，那么应该以集群模式部署 Zookeeper 服务。只要半数以上的集群节点在线，那么服务就是可用的。因为Zk服务可用需要超过半数以上(n/2 + 1)的节点，所以Zk的集群最好是使用奇数个机器。比如：对于四个机器的Zk集群，只能处理单台机器的故障，如果两台机器故障，则剩下的两台不能占半数以上(3)了。但是，如果有了5个机器，Zk可以处理2个机器的故障。

所以，对于生产环境，3个Zk服务器是推荐的最小的配置，并且还建议它运行在不同的机器上。


> **Note** 通常来说，三台服务器对于生产环境应该足够了，但是为了在维护期间也要获得最大的可靠性，你可以安装五台服务器。使用三台服务器时，如果你拿一台服务器去维护，则在维护期间，剩下两台服务器可能会发生故障导致Zk不可用，如果有5台，拿一台去维护，剩下的再故障一台也能提供Zk服务。

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