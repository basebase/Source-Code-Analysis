### ZooKeeper入门系列之安装部署

#### 前期准备
  * JDK (必须, 因为ZooKeeper服务在JVM上运行)
  * 下载ZooKeeper(http://www.apache.org/dist/zookeeper/zookeeper-3.5.6/)
  * 创建一个目录用它来存储与ZooKeeper服务器有关联的一些状态

#### ZooKeeper的三种安装方式
  * 单机模式
  * 伪集群模式
  * 集群模式 (由于我只有一太机器, 所以这个就没法配置了)




#### 单机模式
将下载的ZK解压后, 进入conf/目录下, 可以复制一份zoo_sample.cfg并将文件名称修改为zoo.cfg

```shell
cp zoo_sample.cfg zoo.cfg
```
现在配置我么什么都不去进行修改, 我们进入bin/目录下去运行我们的ZooKeeper服务。
(ps: 需要注意的是, 默认使用2181端口, 看看是否被占用咯)

``` shell
./zkServer.sh start # 启动zk服务
./zkServer.sh status # 查看zk服务状态
```

当我们通过status来查看状态的时候, 会输出如下内容
```text
/usr/bin/java
ZooKeeper JMX enabled by default
-- 这里表示我们使用的配置文件路径
Using config: /zookeeper3.5.6/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
-- 这里表示的是我们使用的是单机模式
Mode: standalone
```

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk02.png?raw=true)

这里服务启动了, 下面我们尝试连接ZK。依旧在bin/目录下。
连接方式有两种, 一种本地连接, 一种远程连接
```shell
./zkCli.sh  # 启用客户端连接(本地连接)
./zkCli.sh -server 127.0.0.1:2181 # 远程连接(ip就是你的远程服务器ip)
```
输入完命令后, 就会输出一大串信息, 接着你就进入了CLI了。

现在看起来还不错, 服务也启动了, 客户端也连接了。现在我们看看如何停止服务把。
```shell
./zkServer.sh stop # 停止服务
```

那么, 到这里zk的单机版就结束了, 很简单的。我们什么配置都没有做过就可以使用了。


#### 伪分布式
虽然我们没有多台服务器, 但是ZooKeeper可以实现伪分布式, 让我们用一台机器模拟出多台机器的服务。

> 注意点: <br />
>> 在一台机器上部署3个server, 我们需要注意端口号不能冲突(clientPort)和dataDir也要不同。另外，还要在dataDir所对应的目录中创建myid文件来指定对应的Zookeeper服务器实例。(也就是前期准备的第三条)

```text
clientPort: 如果在1台机器上部署多个server, 那么每台机器都要不同的clientPort, 例如: server1是2181, server2是2182, server3是2183

dataDir和dataLogDir: dataDir和dataLogDir也需要区分下, 将数据文件和日志文件分开存放, 同时每个server的这两变量所对应的路径都是不同的

server.X和myid: .X对应的是数字, dataDir目录中myid的值。在3个server的myid文件中分别写入了1, 2, 3。那么每个server中的zoo.cfg都要配置server.1, server.2, server.3即可。由于使用的是一台机器每个server的端口号都要配置不同避免端口冲突。
```

上面的注意事项完成之后, 我们创建3份.cfg文件, 可以为zoo1.cfg, zoo2.cfg, zoo3.cfg。


```shell
#### zoo1.cfg

tickTime=2000
initLimit=10
syncLimit=5
# 配置的myid目录
dataDir=/Users/Joker/Desktop/datas/zoo1/server1
# 配置的日志目录
dataLogDir=/Users/Joker/Desktop/datas/zoo1/logs
clientPort=2181
server.1=127.0.0.1:2287:3387
server.2=127.0.0.1:2288:3388
server.3=127.0.0.1:2289:3389



#### zoo2.cfg
### 只展示被修改的地方, 其余地方一样
dataDir=/Users/Joker/Desktop/datas/zoo2/server2
dataLogDir=/Users/Joker/Desktop/datas/zoo2/logs
clientPort=2182


#### zoo3.cfg
### 只展示被修改的地方, 其余地方一样
dataDir=/Users/Joker/Desktop/datas/zoo3/server3
dataLogDir=/Users/Joker/Desktop/datas/zoo3/logs
clientPort=2183
```

在上面的配置中, 我们看到有clientPort不同, 下面的server.X还有端口号?<br />
clientPort: 是我们客户端连接zk服务使用的。<br />
端口2287是zk服务器使用, 用于相互通讯 <br />
端口3387用于Leader选举。

主要我们在一台机器上, 端口不能重复, 所以会有多个端口号。如果是集群的话就不需要这样配置了。

我们在我们配置的dataDir目录下输出对应的数字并写入到myid文件中
```shell
echo 1 > $dataDir1/myid
echo 2 > $dataDir2/myid
echo 3 > $dataDir3/myid
```


以上全部完成后, 下面就可以启动我们的服务了。
```shell
./zkServer.sh start ../conf/zoo1.cfg
./zkServer.sh start ../conf/zoo2.cfg
./zkServer.sh start ../conf/zoo3.cfg
```

可以通过jps -ml来查看是否启动成功
```shell
jps -ml

69397 org.apache.zookeeper.server.quorum.QuorumPeerMain ../conf/zoo1.cfg
69477 org.apache.zookeeper.server.quorum.QuorumPeerMain ../conf/zoo3.cfg
69435 org.apache.zookeeper.server.quorum.QuorumPeerMain ../conf/zoo2.cfg
```

然后在进入我们的dataDir目录下会发现多出很多文件出来, 里面也会记录上面进程的pid

我们再次查看zk的状态看看
```shell
./zkServer.sh status ../conf/zoo1.cfg
./zkServer.sh status ../conf/zoo2.cfg
./zkServer.sh status ../conf/zoo3.cfg
```
会发现Mode不是standalone了, 而是leader和follower了。


![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk03.png?raw=true)

我们再一次连接zk
```shell

### 这里我把所有机器ip都写上了, 而不是一台服务器
./zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183
```

再一次进入客户端, 到这里基本的配置已经完成了。
集群的配置和伪分布式基本上是一致的。
