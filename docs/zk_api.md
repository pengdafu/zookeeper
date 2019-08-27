# zookeeper 客户端命令

本篇的重点是讲解使用 `zkCli.sh -server localhost:2181` 连接到Zk之后，在交互式shell中，ZkCli的命令。

首先我们连接到服务端，查看一下当前有哪些可执行命令:

```shell
ZooKeeper -server host:port cmd args
	addauth scheme auth
	close 
	config [-c] [-w] [-s]
	connect host:port
	create [-s] [-e] [-c] [-t ttl] path [data] [acl]
	delete [-v version] path
	deleteall path
	delquota [-n|-b] path
	get [-s] [-w] path
	getAcl [-s] path
	history 
	listquota path
	ls [-s] [-w] [-R] path
	ls2 path [watch]
	printwatches on|off
	quit 
	reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
	redo cmdno
	removewatches path [-c|-d|-a] [-l]
	rmr path
	set [-s] [-v version] path data
	setAcl [-s] [-v version] [-R] path acl
	setquota -n|-b val path
	stat [-w] path
	sync path

```

接下来我们一个一个讲一下这些命令。

### addauth scheme auth

添加认证用户，在znode中，使用了 `setAcl` 设置 ACL，则需要使用 `addauth scheme auth` 来给当前的连接授权。比如:

```shell
[zk: localhost:2181(CONNECTED) 5] addauth digest pengdafu:password
```

这里要注意的是，这里的password用的是明文密码，而不是经过 `sha1` 和 `base64` 的编码密码。

### close

关闭当前连接，使连接变为 `close` 的状态：

```shell
[zk: localhost:2181(CONNECTED) 6] close

2019-08-23 16:45:16,931 [myid:] - INFO  [main:ZooKeeper@1422] - Session: 0x10000185e930005 closed
[zk: localhost:2181(CLOSED) 7] 2019-08-23 16:45:16,931 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@524] - EventThread shut down for session: 0x10000185e930005


[zk: localhost:2181(CLOSED) 7]
```

### config

`config` 命令将会在[集群](./docs/zk_cluster.md#config)中讨论。

### reconfig

`reconfig` 命令将会在[集群](./docs/zk_cluster.md#reconfig)中讨论。

### connect host:port

这个命令可以用来在交互式时候，连接到别的Zk服务。这个会断开当前连接。

### create

创建节点的命令，用法为:

```shell
create [-s] [-e] [-c] [-t ttl] path [data] [acl]
```

znode的数据是可选的，可以不关联数据，只创建节点:

```shell
[zk: localhost:2181(CONNECTED) 2] create /app02
Created /app02
```

`-s` 参数是创建有序节点，是 `sequence` 的意思:

```shell
[zk: localhost:2181(CONNECTED) 3] create -s /app03
Created /app030000000004

[zk: localhost:2181(CONNECTED) 5] create -s /app03
Created /app030000000005
```

`-e` 参数表示创建临时节点(ephemeral node)，临时节点在会话结束的时候会被删除。

```shell
[zk: localhost:2181(CONNECTED) 6] create -e /ephemeral-node
Created /ephemeral-node

[zk: localhost:2181(CONNECTED) 7] ls /
[app01, app02, app030000000004, app030000000005, ephemeral-node, zookeeper]

[zk: localhost:2181(CONNECTED) 8] connect 127.0.0.1:2181
[zk: 127.0.0.1:2181(CONNECTED) 9] ls /
[app01, app02, app030000000004, app030000000005, zookeeper]

# 临时节点在重新连接的时候被删除
```

`-c` 表示创建容器节点。

`-t ttl` 为创建的永久节点或者永久有序节点设置一个ttl，当ttl时间内，该节点没有发生改变，则会在某个时间被删除。使用该功能，必须要设置系统属性为true才能使用，因为默认是禁止的，如果没有启用，则会报错。下面是例子: 

```shell
# 未启用时使用报错
[zk: localhost:2181(CONNECTED) 0] create -t 10 /ttl
KeeperErrorCode = Unimplemented for /ttl

# 启用时设置后过了ttl时间节点被删除
[zk: localhost:2181(CONNECTED) 0] create -t 10 /ttl ttl
Created /ttl
[zk: localhost:2181(CONNECTED) 1] ls /
[app01, app02, app030000000004, app030000000005, coption, ttl, zookeeper]
[zk: localhost:2181(CONNECTED) 2] ls /
[app01, app02, app030000000004, app030000000005, coption, zookeeper]

```
> 开启ttl节点，在zoo.cfg文件添加配置 `extendedTypesEnabled=true`，然后重启就可以。

`acl` 将在 [setAcl](#setacl) 中讲解。

### delete

用法: `delete [-v version] path`。用于删除一个节点，`-v` 参数很有用，我们知道，在每次设置znode的数据的时候，数据版本会加1，那么如果没有-v指定删除的版本，就很有可能在一个客户端获取这个节点的时候，版本是v1，但是在执行删除之前，别的客户端又对这个节点做了修改，那么删除就可能导致删除了别的客户端想要的数据。如果删除的时候加上获取的版本号，当版本号不一致时，将无法删除成功:

```shell
[zk: localhost:2181(CONNECTED) 12] delete -v 0 /app
version No is not valid : /app
```

### deleteall

用法: `deleteall path`。用于删除节点以及它的子节点， `delete` 只能用于删除单个节点，对于存在子节点的节点应该使用 `deleteall` 命令。

### delquota

用法: `delquota [-n|-b] path`。删除配额，我们可以使用 [setquota](#setquota) 设置配额，也可以使用 [listquota](#listquota) 来查看当前节点的配额。注意：删除配额只要在配置了配额的时候才能删除，否则报错。加上 `-n | -b` 选项表示只能删除对应的节点数限制或者大小限制，举个例子，设置的时候使用 `-n` 指定节点数，则删除的时候就不能接 `-b` 选项，否则报错。加了选项删除，不会删除该节点的配额信息，还可用 `listquota` 查看配额信息，如果直接使用 `delquota path`，那么使用 `listquota` 提示 `quota for /path does not exist.`


### get

获取节点信息，用法:

```shell
get [-s] [-w] path
```
看一些实例:

```shell
[zk: localhost:2181(CONNECTED) 18] create /get 
Created /get
[zk: localhost:2181(CONNECTED) 19] get /get
null
# 由于创建的时候没有关联数据，所以获取的时候是null
```

注意，新版本的 `get` 默认已经不会返回该节点的 `stat` 结构了，只会返回该节点的数据，`-s` 选项是要求返回 `stat` 结构:

```shell
[zk: localhost:2181(CONNECTED) 20] get -s /get
null
cZxid = 0x10
ctime = Sat Aug 24 10:51:26 CST 2019
mZxid = 0x10
mtime = Sat Aug 24 10:51:26 CST 2019
pZxid = 0x10
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 0

```

`-w` 选项是为该节点增加一个监听器，当该节点发生以下事件时，都会向客户端发送监听事件:

- **NodeDataChanged**: 节点数据被改变，比如节点被 `set` 了。
- **NodeDeleted**: 节点被删除。
- **DataWatchRemoved**: 监听器被移除时触发

> **Note**: 监听器只会被触发一次。

### getAcl

用法: `getAcl [-s] path`。获取该节点的ACL，`-s` 选项是返回该节点的 `stat` 结构:

```shell
[zk: localhost:2181(CONNECTED) 35] getAcl -s /zookeeper
'world,'anyone
: cdrwa
cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -2
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 2
```

### history && redo

这是两个命令，放在一块讲，是因为一般都是两个命令联合使用的。

`history` 命令用于获取最近 10 条执行命令的获取记录:

```shell
# 第一列叫 cmdno, 第三列是执行的命令
[zk: localhost:2181(CONNECTED) 39] history
29 - get -w /get
30 - delete /get/test
31 - help
32 - getAcl /get
33 - getAcl -s /get
34 - getAcl /zookeeper
35 - getAcl -s /zookeeper
36 - help
37 - history
38 - get -w /get
39 - history
```

`redo` 的用法是 `redo cmdno`，可以重新执行 cmdno对应的那条命令。
> 注意：`redo` 并不是只能执行 `history` 显示的那10条。

### listquota

用法: `listquota path`。非常简单的一个命令，用于获取该节点的配额信息(节点数限制):

```shell
[zk: localhost:2181(CONNECTED) 49] listquota /test
absolute path is /zookeeper/quota/test/zookeeper_limits
Output quota for /test count=10,bytes=-1
Output stat for /test count=1,bytes=0
# 第一行是配额信息存放的位置
#第二行表示节点数最大是10个，节点大小无限制
#第三行表示该当前节点数量，当前节点大小
```

### setquota

设置节点的配额。配额有两种，只能选其一:

- **-n**：节点个数配额，节点个数是包括该节点和子节点的个数，比如3表示该节点最多拥有两个子节点
- **-b**：节点大小配额，节点大小是指一个节点关联数据的大小，这个配额包含该节点和所有子节点的大小

`setquota path` 将会在 `/zookeeper/quota/path` 创建两个子节点 `zookeeper_limits` 和 `zookeeper_stats`：

- zookeeper_stats：存放实际path节点及其子节点个数和节点大小
- zookeeper_limits：存放path节点的配额信息

所以如果要防止别人修改配额信息，则应该对这两个节点设置ACL。

用法: `setquota -n|-b /path`。

> **Note!** 尽管你配置了配额，但是实际是物理限制并不会生效，你依旧可以超过配额限制的节点数或者节点大小，Zk并不会抛出异常，只会在日志中打印一个警告:
> 
> ```shell
> WARN  [SyncThread:0:DataTree@340] - Quota exceeded: /test count=4 limit=1
> ```


### ls

用法为:

```shell
ls [-s] [-w] [-R] /path

# 例子
[zk: localhost:2181(CONNECTED) 10] ls /test
[01, 02, 03, 04, 05]

```

`-s` 参数用于返回查看的节点的 `stat` 结构:

```shell
[01, 02, 03, 04, 05]cZxid = 0x2
ctime = Mon Aug 26 11:45:37 CST 2019
mZxid = 0x2
mtime = Mon Aug 26 11:45:37 CST 2019
pZxid = 0x1c
cversion = 5
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 5
```

`-R` 参数用于返回该节点以及递归式返回该节点下的所有子节点：

```shell
[zk: localhost:2181(CONNECTED) 25] ls -R /level1
/level1
/level1/level2

```

`-w` 为该节点添加监听器，当该节点发生以下事件时，都会向客户端发送监听事件:

- **NodeChildrenChanged**: 创建子节点和删除子节点的时候会触发该事件
- **NodeDeleted**: 节点被删除时触发
- **ChildWatchRemoved**: 监听器被移除时触发


### ls2

该命令已过时，使用 `ls -s /path` 代替。

### printwatches

用法: `printwatches on|off`，用于shell客户端是否打印监听器返回的事件。默认是 `on`，即收到监听时打印。

### quit

用于关闭连接并退出交互式客户端。

### rmr

已经过时，使用 `deleteall`代替。

### set

用法：`set [-s] [-v version] /path data`。

`-s` 参数用于返回该节点的 `stat结构`。

`-v version` 用于限定你修改的是你自己想要的结果。比如当你获取数据的版本是1，那么如果你要修改它的数据，但是你又怕在此之前被别的客户端修改过，那么可以使用 `set -v 2 /path data`, 如果数据的版本是1才能修改成功，否则修改失败。如果不加 `-v` 你的修改始终是最新的，也就可能覆盖了别人的修改。

### stat

用法: `stat [-w] /path`。用于返回该节点的 `stat` 结构。

`-w` 将会为该节点添加监听器，当该节点发生以下事件时，都会向客户端发送监听事件:

- **NodeDeleted**: 当前节点被删除时触发
- **NodeDataChanged**: 当前节点数据被修改时
- **DataWatchRemoved**: 当前节点监听器被移除时

### sync

`sync path`，用于同步数据，建议在get操作之前，先调用 `sync`。

### removewatches

用法: `removewatches path [-c|-d|-a] [-l]`。用于移除监听器，并且会根据不同的监听器触发不同的事件，主要有:

- DataWatchRemoved
- ChildWatchRemoved

### setAcl

用法: `setAcl [-s] [-v version] [-R] path acl`。

`-s` 参数用于返回 `stat` 结构。

`-v version` 指的是 `aclVersion`，在指定版本的时候，只有当前节点的acl版本和指定的版本一致，才能加上ACL，否则添加失败。每设置一次ACL，`aclVersion` 将会加1。

`-R` 参数表示递归的设置ACL，将会对该节点以及该节点下的所有子节点设置ACL。

`ACL` 有五种内置方案: [world 方案](#world-方案)、[ip 方案](#ip-方案)、[auth 方案](#auth-方案)、[digest 方案](#digest-方案)、[x509 方案](#x509-方案)。

> **Note!** \<acl\> 分别表示:
> - **a**: ADMIN权限，拥有设置ACL的权限
> - **d**：DELETE权限，拥有删除子节点的权限(仅仅儿子节点，不能孙子及以下的节点)
> - **c**：CREATE权限，拥有创建子节点的权限
> - **w**：WRITE权限，可以设置关联数据的权限
> - **r**：READ权限，可以读取该节点数据和子节点

#### world 方案

设置方式: `setAcl /path world:anyone:<acl>`。一个节点被创建的时候，默认的ACL是 `world:anyone:cdrwa`。`world` 方案的id就是 `anyone`，设置的时候使用非 `anyone`则会报错。

#### ip 方案

`ip` 方案使用ip进行认证，设置方式: `setAcl /path ip:<ip>:<acl>`。

`<ip>`：可以是具体IP也可以是IP/bit格式，即IP转换为二进制，匹配前bit位，如 `192.168.0.0/16` 匹配 `192.168.*.*`

#### auth 方案

`auth` 方案使用当前添加的认证用户来快捷设置acl，在指南中提到，`auth` 方案也需要满足 `scheme:expression:perms` 形式。设置方式:

```shell
addauth digest <user>:<password> 
setAcl /path auth:<user>:<acl>
```
例子:

```shell
# 创建测试节点
[zk: localhost:2181(CONNECTED) 5] create /auth

# 未认证用户设置Acl将失败
[zk: localhost:2181(CONNECTED) 7] setAcl /auth auth:pdf:awrdc
Acl is not valid : /auth

# 认证
[zk: localhost:2181(CONNECTED) 8] addauth digest pdf:pdf

# 设置Acl
[zk: localhost:2181(CONNECTED) 9] setAcl /auth auth:pdf:awrdc
[zk: localhost:2181(CONNECTED) 10] getAcl /auth
'digest,'pdf:3HxRJ+7BQcbwChnEI9ujXDOgruE=
: cdrwa

```

认证的时候使用的是明文密码，添加Acl的时候也是用的明文密码，`a` 和 `r` 都能获取Acl，但是 `a` 可以看到密码，`r` 不行。

#### digest 方案

设置方式: `setAcl /path digest:<user>:<password>:<acl>`。

这里的密码是经过 SHA1 和 BASE64 编码处理的密文，在 SHELL 中通过下面的命令计算:

```shell
echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64

# 比如
echo -n pengdafu:pengdafupassword | openssl dgst -binary -sha1 | openssl base64
ZycDzt849Q77V9bt2nX/MI9jKuA=
```

添加Acl和验证:

```shell
# 创建节点
[zk: localhost:2181(CONNECTED) 0] create /digest
Created /digest

# 设置权限，注意，密码使用 sha1 和 base64 编码后的密码
[zk: localhost:2181(CONNECTED) 1] setAcl /digest digest:pengdafu:ZycDzt849Q77V9bt2nX/MI9jKuA=:ra

# 未认证时无法访问
[zk: localhost:2181(CONNECTED) 2] ls /digest
Authentication is not valid : /digest

# 添加认证，认证使用明文
[zk: localhost:2181(CONNECTED) 3] addauth digest pengdafu:pengdafupassword

# 可以访问了
[zk: localhost:2181(CONNECTED) 4] ls /digest
[]
```

#### x509 方案
	太复杂，不讨论。

# Zookeeper 四字命令

Zk 提供四字命令用于运维，我们可以借助 `nc` 或者 `telnet` 等工具向Zk发送四字命令。在Zk3.5以上版本，提供 `AdminServer` 为四字命令提供http调用接口，http服务使用jetty服务器，端口8080。

## 四字命令

### conf

打印有关服务端的详细信息:

```shell
echo conf | nc localhost 2181

secureClientPort=-1
dataDir=/tmp/zookeeper/version-2
dataDirSize=424
dataLogDir=/tmp/zookeeper/version-2
dataLogSize=424
tickTime=2000
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
serverId=0
```

### cons

列出所有连接到该服务端的客户端的连接/会话的详细信息，包括发送/接收包的数量、会话Id、操作延迟、最后执行的操作等。。

```shell
echo cons | nc localhost 2181

 /127.0.0.1:52736[0](queued=0,recved=1,sent=0)
 /127.0.0.1:52710[1](queued=0,recved=1,sent=1,sid=0x100002021320000,lop=SESS,est=1566871910529,to=30000,lcxid=0x0,lzxid=0x1,lresp=2502727,llat=23,minlat=0,avglat=23,maxlat=23)
```

### crst

重置所有连接/会话的统计信息。

```shell
echo crst | nc localhost 2181
Connection stats reset.
```

### dump

列出未完成的会话和临时节点，这个命令只能 `leader` 使用。

### envi

列出服务环境的详细信息：

```shell
echo envi | nc localhost 2181
Environment:
zookeeper.version=3.5.5-390fe37ea45dee01bf87dc1c042b5e3dcce88653, built on 05/03/2019 12:07 GMT
host.name=pg
java.version=11.0.1
java.vendor=Oracle Corporation
java.home=/usr/local/java
java.class.path=/usr/local/zookeeper/bin/../zookeeper-server/target/classes:/usr/local/zookeeper/bin/../build/classes:/usr/local/zookeeper/bin/../zookeeper-server/target/lib/*.jar:/usr/local/zookeeper/bin/../build/lib/*.jar:/usr/local/zookeeper/bin/../lib/zookeeper-jute-3.5.5.jar:/usr/local/zookeeper/bin/../lib/zookeeper-3.5.5.jar:/usr/local/zookeeper/bin/../lib/slf4j-log4j12-1.7.25.jar:/usr/local/zookeeper/bin/../lib/slf4j-api-1.7.25.jar:/usr/local/zookeeper/bin/../lib/netty-all-4.1.29.Final.jar:/usr/local/zookeeper/bin/../lib/log4j-1.2.17.jar:/usr/local/zookeeper/bin/../lib/json-simple-1.1.1.jar:/usr/local/zookeeper/bin/../lib/jline-2.11.jar:/usr/local/zookeeper/bin/../lib/jetty-util-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/jetty-servlet-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/jetty-server-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/jetty-security-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/jetty-io-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/jetty-http-9.4.17.v20190418.jar:/usr/local/zookeeper/bin/../lib/javax.servlet-api-3.1.0.jar:/usr/local/zookeeper/bin/../lib/jackson-databind-2.9.8.jar:/usr/local/zookeeper/bin/../lib/jackson-core-2.9.8.jar:/usr/local/zookeeper/bin/../lib/jackson-annotations-2.9.0.jar:/usr/local/zookeeper/bin/../lib/commons-cli-1.2.jar:/usr/local/zookeeper/bin/../lib/audience-annotations-0.5.0.jar:/usr/local/zookeeper/bin/../zookeeper-*.jar:/usr/local/zookeeper/bin/../zookeeper-server/src/main/resources/lib/*.jar:/usr/local/zookeeper/bin/../conf:.:/usr/local/java/lib/dt.jar:/usr/local/java/lib/tools.jar
java.library.path=/usr/java/packages/lib:/usr/lib64:/lib64:/lib:/usr/lib
java.io.tmpdir=/tmp
java.compiler=<NA>
os.name=Linux
os.arch=amd64
os.version=4.15.0-29deepin-generic
user.name=root
user.home=/root
user.dir=/root
os.memory.free=117MB
os.memory.max=1000MB
os.memory.total=124MB
```

### ruok

这个命令非常重要，用于检测服务是否没有错误的运行。如果服务在没有报错的情况下运行，则会响应一个 `imok` 的字符串，否则它不会响应任何东西。"imok" 响应并不意味着该服务器已经加入ZK法定成员中，而仅代表服务器处于活动状态且与某个特定的客户端口建立了绑定关系。你可以使用 `stat` 命令查看该服务器是否加入法定投票成员及客户端连接等详细状态信息。

```shell
echo ruok | nc localhost 2181
imok
```

### srst

重置服务的统计信息

### srvr

列出服务器的完整详细信息。

```shell
root@pg:~# echo srvr | nc localhost 2181

Zookeeper version: 3.5.5-390fe37ea45dee01bf87dc1c042b5e3dcce88653, built on 05/03/2019 12:07 GMT
Latency min/avg/max: 0/0/23
Received: 197
Sent: 196
Connections: 2
Outstanding: 0
Zxid: 0x2
Mode: standalone
Node count: 6
```

### stat

列出服务端和连接客户端的简要信息:

```shell
echo stat | nc localhost 2181

Zookeeper version: 3.5.5-390fe37ea45dee01bf87dc1c042b5e3dcce88653, built on 05/03/2019 12:07 GMT
Clients:
 /127.0.0.1:52710[1](queued=0,recved=172,sent=172)
 /127.0.0.1:58484[0](queued=0,recved=1,sent=0)

Latency min/avg/max: 0/0/23
Received: 212
Sent: 211
Connections: 2
Outstanding: 0
Zxid: 0x2
Mode: standalone
Node count: 6
```

### wchs

列出服务的监听器的简要信息:

```shell
echo wchs | nc localhost 2181

1 connections watching 1 paths
Total watches:1
```

### wchc

从session视角列出watch该服务器的客户端的详细信息。该命令将输出session(connection)所watch的路径的相关信息。**!!** 有一点要注意下，执行这个命令的性能损耗与watch的数量有关，请谨慎使用。

```shell
echo wchc | nc localhost 2181

0x100002021320000   # sessionId
	/ep             # 监听的path
```

### dirs

**Added in 3.5.1**

以字节为单位显示快照和日志文件的总大小。

```shell
echo dirs | nc localhost 2181

datadir_size: 67109304
logdir_size: 67109304
```

### wchp

从path视角列出watch该path的相关客户端session。 **!!** 有一点要注意下，执行这个命令的性能损耗与watch的数量有关，请谨慎使用。

```shell
echo wchp | nc localhost 2181
/ep
	0x100002021320000
```

### mntr

输出可用于监控集群运行状况的列表

```shell
echo mntr | nc localhost 2180
	zk_version  3.4.0
	zk_avg_latency  0
	zk_max_latency  0
	zk_min_latency  0
	zk_packets_received 70
	zk_packets_sent 69
	zk_num_alive_connections 1
	zk_outstanding_requests 0
	zk_server_state leader
	zk_znode_count   4
	zk_watch_count  0
	zk_ephemerals_count 0
	zk_approximate_data_size    27
	zk_followers    4                   - only exposed by the Leader
	zk_synced_followers 4               - only exposed by the Leader
	zk_pending_syncs    0               - only exposed by the Leader
	zk_open_file_descriptor_count 23    - only available on Unix platforms
	zk_max_file_descriptor_count 1024   - only available on Unix platforms
	zk_last_proposal_size 23
	zk_min_proposal_size 23
	zk_max_proposal_size 64
```

### isro

测试服务器是否以只读模式运行，如果处于只读模式，将会响应 `ro` ，否则响应 `rw`。

```shell
echo isro | nc localhost 2181
rw
```

### gtmk

以十进制64位有符号长整形获取当前跟踪掩码。

### stmk

设置当前的跟踪掩码。跟踪掩码是64位，其中每个位启用或禁用服务器上特定类别的跟踪日志记录。必须将Log4J的启用级别配置为TRACE首才能查看跟踪日志记录消息

## AdminServer

`AdminServer` 是一个嵌入式的Jetty服务器，它为四字命令提供http接口。默认情况下，服务器在端口8080上启动，并通过路径 `/commands/[command name]` 发出请求，比如 : `localhost:8080/commands/stat` 。响应是json格式。与原始协议不同，命令不再局限于四个字母，命令可以有多个名称，要查看所有可用命令列表，请求: `http://localhsot:8080/commands`。

AdminServer 默认启用，如果你在启动Zk的时候，发现启动不了，并且查看日志提示地址已被使用，则表示 8080 端口已被其他应用占用。你可以更改 AdminServer 的默认端口或者直接禁用 AdminServer。

- **admin.enableServer** - 设置为 `false` 以禁用 AdminServer。默认已启用。
- **admin.serverAddress** - 嵌入式服务器Jetty的监听地址，默认是 0.0.0.0。
- **admin.serverPort** - AdminServer 的监听端口
- **admin.idleTimeout** - 设置连接在发送或者接收数据之前可以等待的最长空闲时间（以毫秒为单位），默认是 30000 毫秒。
- **admin.commandURL** - 用于列出和发出相对于根url的命令的url。默认是 `commands`。