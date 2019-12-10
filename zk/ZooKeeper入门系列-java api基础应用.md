### ZooKeeper入门系列之Java基础应用

#### 说在前面
经过前面的学习我们大概知道一些ZK的术语及shell命令的使用, 但是在程序中我们又如何使用呢? 怎么去编写一个简单的ZK Java客户端程序呢?
通过下面的一个例子, 我们来进行学习。

#### 前期准备
  * IDEA
  * Maven

在部署ZK的时候我们肯定已经安装了JDK和ZK服务的, 这里不再说明。


#### 将学习那些API的使用?
这里的话, 基本上我们会学习如何创建一个node, 如何获取node数据, 如何删除数据, 如何获取子节点等
基础API功能。我们会设计一个抽象类来完成上面的功能, 抽象类包含如下方法:
  + create
  + getZNodeData
  + exits
  + update
  + delete
  + getChildren


#### 构建项目

##### 添加pom依赖

第一步: 打开我们的IDEA然后选择创建一个Maven工程, 后面的信息更具自己喜好自己填写就OK了。
![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk06.png?raw=true)

当我们的项目创建完毕之后, 我们在pom.xml中添加zookeeper依赖如下

```xml

<dependencies>
        <!-- 理论上添加zookeeper的jar包即可, 但是我用到了junit进行测试 -->
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.5.6</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

万里长征的第一步我们已经迈出了, 依赖有了, 那么如何连接ZK服务呢?


##### 连接ZK服务

通过zkClient.sh我们可以很轻松的链接上服务, 那么在Java中又是如何链接呢?

创建ZKConnection.java
```java
public class ZKConnection {

    private ZooKeeper zoo ;

    /***
     * 创建链接
     * @param host ZK服务地址
     * @return
     * @throws IOException
     * @throws InterruptedException
     */
    public ZooKeeper connect(String host) throws IOException, InterruptedException {
        /**
         * 在创建ZooKeeper对象时, 需要提供三个参数
         * connectString: ZK主机地址信息
         * sessionTimeout: 会话超时时间
         * watcher: 监听器, 由于zk的创建过程是异步的，这里可以通过注册一个监视器, 监听zk状态
         */

        zoo = new ZooKeeper(host, 3000, (watchedEvent) -> {
            System.out.println("State: " + watchedEvent.getState()); // 事件状态
            System.out.println("Type: " + watchedEvent.getType()); // 事件类型
            System.out.println("Path: " + watchedEvent.getPath()); // 事件发生的路径
            System.out.println("================================");
        });

        return zoo ;
    }

    // 关闭链接
    public void close() throws InterruptedException {
        zoo.close();
    }
}
```
在创建ZooKeeper对象, 我们构建的监听器, 之后会有用, 这里我们先不着急。如何通过Java连接ZK服务我们已经写完了。
可以写@Test方法来进行测试

在运行下面的程序会发现连接成功, 但似乎没有打印输出信息啊, 这里的话请大家思考一下, hhh。
```java
@Test
public void createConn() throws IOException, InterruptedException {
  ZKConnection zkConn = new ZKConnection();
  ZooKeeper connect = zkConn.connect("localhost:2181");
  System.out.println("conn -> " + connect);
}
```


好的, 既然连接成功了, 那么接下来就是一些API的使用了吧。


##### 设计一个ZK接口

```java

public interface ZKManage {
    public void create(String path, byte[] data) throws KeeperException, InterruptedException;
    public Object getZNodeData(String path, boolean watchFlag) throws KeeperException, InterruptedException, UnsupportedEncodingException;
    public Stat exits(String path, boolean watch) throws KeeperException, InterruptedException;
    public void update(String path, byte[] data) throws KeeperException, InterruptedException;
    public void delete(String path) throws KeeperException, InterruptedException;
    public List<String> getChildren(String path) throws KeeperException, InterruptedException;
    public void closeConnection() throws InterruptedException;
}
```

上面的接口方法就是我们需要去实现的了。我们先来构建一个实现接口的子类把。


```java

public class ZKManageImpl implements ZKManage {
  private static final String HOST = "localhost:2181,localhost:2182,localhost:2183";
  private static ZooKeeper zoo ;
  private static ZKConnection zkConnection ;

  public ZKManageImpl() {
    initialize();
  }

  private void initialize() {
    try {
      zkConnection = new ZKConnection() ;
      zoo = zkConnection.connect(HOST) ;
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  @Override
  public void closeConnection() throws InterruptedException {
    zkConnection.close();
  }
}
```

上面的方法不设计ZK的一些使用, 所以先构建出来, 下面所有的方法都是在该类中。我就不把类名都输出来了。



##### 创建节点

```java

@Override
public void create(String path, byte[] data) throws KeeperException, InterruptedException {
  /***
   * zk创建一个节点有4个参数
   * path: 创建的路径
   * data: 节点数据, 字节数组
   * acl: 权限控制, 使用Ids.OPEN_ACL_UNSAFE开发权限即可
   * createMode: 节点类型, 持久化(PERSISTENT)|临时节点(EPHEMRAL)|持久化顺序节点(PERSISTENT_SEQUENTIAL)|临时顺序节点(EPHEMRAL_SEQUENTIAL)
   */
  zoo.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
}
```

测试一下, 是否可以成功创建我们的节点。

```java

@Test
public void createNodes() throws KeeperException, InterruptedException {
  String p1 = "/xiaomoyu3";
  zkManage.create(p1, null);

  String p2 = "/xiaomoyu3/testdb";
  zkManage.create(p2, "yoho".getBytes());
}
```

当前节点下是没有xiaomoyu3节点的

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk07.png?raw=true)


运行我们的程序之后, 就可以看到在zk集群中存在对应的节点数据了。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk08.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk09.png?raw=true)




##### 获取节点数据

```java

@Override
public Object getZNodeData(String path, boolean watchFlag) throws KeeperException, InterruptedException, UnsupportedEncodingException {

  // 判断节点是否存在
  if (exits(path, watchFlag) == null) throw new KeeperException.NoNodeException("节点不存在");

  /***
   * path: 节点路径
   * watch: 是否监听, true|false
   * stat: 一般不用管
   */
    byte[] b = zoo.getData(path, watchFlag, null) ;
    String data = new String(b, "UTF-8") ;
    return data ;
}
```

```java

@Test
public void getNodeData() throws InterruptedException, UnsupportedEncodingException, KeeperException {

        // xiaomoyu3是没有写入数据的, 在获取的时候为null, 抛出异常
//      String p1 = "/xiaomoyu3" ;
//      Object zNodeData = zkManage.getZNodeData(p1, true);
//      System.out.println(zNodeData);

        String p2 = "/xiaomoyu3/testdb" ;
        Object zNodeData2 = zkManage.getZNodeData(p2, true);
        System.out.println(zNodeData2);
}
```


##### 更新数据

```java

@Override
public void update(String path, byte[] data) throws KeeperException, InterruptedException {
  Stat stat = exits(path, true);
  if (stat != null) {
    int version = zoo.exists(path, true).getVersion();
    /***
     * path: 节点路径
     * data: 更新的数据, 字节数组
     * version: 对应的版本, 如果是-1则忽略版本
     */
    zoo.setData(path, data, version);
  }
}
```


```java

@Test
public void setNodeData() throws KeeperException, InterruptedException, UnsupportedEncodingException {
  String p1 = "/xiaomoyu3/testdb" ;

  Object changeBefore = zkManage.getZNodeData(p1, true);
  System.out.println("数据修改之前 -> " + changeBefore);

  zkManage.update(p1, "i love you forever".getBytes());
  Object changeAfter = zkManage.getZNodeData(p1, true);
  System.out.println("数据修改之后 -> " + changeAfter);

  zkManage.update(p1, "爱你直到永远".getBytes());
  changeAfter = zkManage.getZNodeData(p1, true);
  System.out.println("数据修改之后 -> " + changeAfter);
}
```
可以看到, 我们在get上注册了监听, 当我们修改数据的时候, 就获取到了NodeDataChanged, 并且修改的节点
路径为/xiaomoyu3/testdb

```text
State: SyncConnected
Type: None
Path: null
================================
数据修改之前 -> yoho
State: SyncConnected
Type: NodeDataChanged
Path: /xiaomoyu3/testdb
================================
数据修改之后 -> i love you forever
State: SyncConnected
Type: NodeDataChanged
Path: /xiaomoyu3/testdb
================================
数据修改之后 -> 爱你直到永远
```


![image-20190107172729251](https://github.com/basebase/img_server/blob/master/zk/zk10.png?raw=true)


##### 删除节点

```java

@Override
public void delete(String path) throws KeeperException, InterruptedException {
  Stat stat = exits(path, true);
  if (stat != null) {
    int version = stat.getVersion();
    /***
     * path: 节点路径
     * version: 版本
     */
     zoo.delete(path, version);
   }
}
```


```java
@Test
public void deleteNode() throws KeeperException, InterruptedException {
  String p1 = "/xiaomoyu3/testdb" ;
  zkManage.delete(p1);
}
```

```text
删除的时候, 也会发生监听
State: SyncConnected
Type: NodeDeleted
Path: /xiaomoyu3/testdb
```



##### 获取子节点

```java
@Override
public List<String> getChildren(String path) throws KeeperException, InterruptedException {
  Stat stat = exits(path, true);
  if (stat == null) throw new KeeperException.NoNodeException("节点不存在");
  List<String> children = zoo.getChildren(path, true);
  return children;
}
```

```java

@Test
public void childNodes() throws KeeperException, InterruptedException {
  String p1 = "/" ;
  List<String> childs = zkManage.getChildren(p1);
  for (String child : childs) {
    System.out.println("|- " + child);
  }
}
```

```text
|- zookeeper
|- xiaomoyu3
|- xiaomoyu
```

以上就是基本的API的使用了, 该程序我已经上传到github上面去了。喜欢的话大家可以clone下来学习。

```text
https://github.com/basebase/zk-example.git
```



#### 结束语

到这里呢, 基本的ZK Java API的学习就结束了。该系列或许不会很深入但是希望能让大家快速的熟悉ZK的基本使用。
大家有什么好的建议也可以私信发我。欢迎订阅我的公众号。一起进步一起学习。



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/gzh.jpeg?raw=true)
