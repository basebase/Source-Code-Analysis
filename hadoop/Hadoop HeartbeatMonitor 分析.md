### Hadoop HeartbeatMonitor 分析



#### 心跳机制概述

在进行Hadoop心跳监控源码分析之前, 我们先来对心跳机制有个了解, 他是用来做什么的? 心跳机制的出现解决了什么问题？

在网络中的接受和发送数据都是使用操作系统中的socket进行实现。但是如果socket断开, 那么发送数据和接收数据的时候
一定会出现问题。可是如何判断这个socket是否还可以使用呢？这个就需要在系统中创建心跳机制。其实TCP中已经为我们
实现了一个叫做心跳的机制。如果你设置了心跳，那TCP就会在一定的时间（比如你设置的是3秒钟）内发送你设置的次数的心跳（比如2次），
并且此信息不会影响你自己定义的协议。所谓“心跳”就是定时发送一个自定义的结构体（心跳包或心跳帧），让对方知道自己“在线”。
以确保链接的有效性。


到这里, 我们有个基本概念了。如何去设计大家可以自行搜索。本文呢也只是分析如果没有心跳后Hadoop会做什么操作。
下面, 就开始进入主题把，还是老规矩。先上我们的可执行代码。。。。



#### HeartbeatMonitor源码分析


```java



public class HeartbeatMonitorTest {
    public static final Logger LOG = LogFormatter.getLogger("com.hadoop.test.HeartbeatMonitorTest");


    // 心跳运行标记
    boolean fsRunning = true;
    // 心跳轮训的毫秒
    private int heartBeatRecheck = 1000;
    // 心跳超时 
    public static long EXPIRE_INTERVAL = 10 * 60 * 1000;
    // 服务器的总容量和可使用容量(这里我为了方便测试随意些了一些数值)
    long totalCapacity = 0, totalRemaining = 0;
    // 备份数
    private int desiredReplication = 1;
    // datanode节点的map
    TreeMap datanodeMap = new TreeMap();
    // block的map
    TreeMap blocksMap = new TreeMap();
    
    private TreeSet neededReplications = new TreeSet();
    // 备份集合
    TreeMap excessReplicateMap = new TreeMap();
    Configuration conf = new Configuration();
    FSDirectory dir = null;





    //
    // Stores a set of datanode info objects, sorted by heartbeat
    // 这里用最后的发送时间进行比较排序
    TreeSet heartbeats = new TreeSet(new Comparator() {
        public int compare(Object o1, Object o2) {
            DatanodeInfo d1 = (DatanodeInfo) o1;
            DatanodeInfo d2 = (DatanodeInfo) o2;
            long lu1 = d1.lastUpdate();
            long lu2 = d2.lastUpdate();
            if (lu1 < lu2) {
                return -1;
            } else if (lu1 > lu2) {
                return 1;
            } else {
                return d1.getName().compareTo(d2.getName());
            }
        }
    });



    class HeartbeatMonitor implements Runnable {
        /**
         */
        public void run() {
            while (fsRunning) {
                // 心跳检查
                heartbeatCheck();
                try {
                    Thread.sleep(heartBeatRecheck); // 等待的时间
                } catch (InterruptedException ie) {
                }
            }
        }
    }


    /**
     * Check if there are any expired heartbeats, and if so,
     * whether any blocks have to be re-replicated.
     * 检查是否有过期的心跳节点, 如果有是否需要从新复制blocks
     * 
     */
    synchronized void heartbeatCheck() { 
        synchronized (heartbeats) {
            DatanodeInfo nodeInfo = null;
          
            // 如果没有要监控的心跳节点, 如果被心跳监控的Node最后一个更新时间小于当前时间减去指定时间
            // 那么这个时间段还没有超出心跳超时的时间段。
            // 这里为了方便, 我把时间这块注释掉了, 这样好debug
            while ((heartbeats.size() > 0) &&
                    ((nodeInfo = (DatanodeInfo) heartbeats.first()) != null) //&&
                    /*(nodeInfo.lastUpdate() < System.currentTimeMillis() - EXPIRE_INTERVAL)*/) {
              
                // 如果满足了条件, 则当前的节点被认为失去了连接, 执行下面的逻辑
                LOG.info("Lost heartbeat for " + nodeInfo.getName());
                System.out.println("Lost heartbeat for " + nodeInfo.getName());
                // 首先从心跳结构中删除当前失去连接的节点
                heartbeats.remove(nodeInfo);
                synchronized (datanodeMap) {
                    // 删除该节点在内存中的数据
                    datanodeMap.remove(nodeInfo.getName());
                }
              
                // 由于失去了连接, 我们集群总量大小也要减去该节点的总量
                // 以及我们可用的空间也要减去该节点的可用空间
                totalCapacity -= nodeInfo.getCapacity();
                totalRemaining -= nodeInfo.getRemaining();
              
                // 获取到当前节点的block块数据
                Block deadblocks[] = nodeInfo.getBlocks();
                if (deadblocks != null) {
                    for (int i = 0; i < deadblocks.length; i++) {
                        removeStoredBlock(deadblocks[i], nodeInfo);
                    }
                }

                if (heartbeats.size() > 0) {
                    nodeInfo = (DatanodeInfo) heartbeats.first();
                }
            }
        }
    }


    /**
     * Modify (block-->datanode) map.  Possibly generate
     * replication tasks, if the removed block is still valid.
     */
    synchronized void removeStoredBlock(Block block, DatanodeInfo node) {
        // 该block对应的Node节点数据, 一个Node可能包含多个block
        TreeSet containingNodes = (TreeSet) blocksMap.get(block);
        if (containingNodes == null || ! containingNodes.contains(node)) {
            throw new IllegalArgumentException("No machine mapping found for block " + block + ", which should be at node " + node);
        }
      
        // 并删除该node数据
        containingNodes.remove(node);

        // 这里我没有细看, 不过见名知意, 备份block到其他node上。
        // It's possible that the block was removed because of a datanode
        // failure.  If the block is still valid, check if replication is
        // necessary.  In that case, put block on a possibly-will-
        // be-replicated list.
        //
        if (dir.isValidBlock(block) && (containingNodes.size() < this.desiredReplication)) {
            synchronized (neededReplications) {
                neededReplications.add(block);
            }
        }

        //
        // We've removed a block from a node, so it's definitely no longer
        // in "excess" there.
        // 从这个备份的map中获取到失去心跳的节点的blocks
        TreeSet excessBlocks = (TreeSet) excessReplicateMap.get(node.getName());
        if (excessBlocks != null) {
            // 删除该节点上对应的block
            excessBlocks.remove(block);
            // 如果该节点上没有了block了，也需要从内存中清除。
            if (excessBlocks.size() == 0) {
                excessReplicateMap.remove(node.getName());
            }
        }
    }


    public void start() throws IOException {
        this.dir = new FSDirectory(new File(conf.get("dfs.name.dir", "/tmp/hadoop/dfs/name")));

        this.totalCapacity = 100000;
        this.totalRemaining = 50000;


        DatanodeInfo datanode1 = new DatanodeInfo(new UTF8("datanode1"), 10000, 5000);
        DatanodeInfo datanode2 = new DatanodeInfo(new UTF8("datanode2"), 99999, 4444);
        DatanodeInfo datanode3 = new DatanodeInfo(new UTF8("datanode3"), 77777, 3332);
        DatanodeInfo datanode4 = new DatanodeInfo(new UTF8("datanode4"), 11177, 1232);

        Block b1 = new Block(-121212, 232323);
        Block b2 = new Block(-787878, 565656);
        Block b3 = new Block(-343434, 898989);
        Block b4 = new Block(-345634, 778968);

        datanode1.addBlock(b1);
        datanode2.addBlock(b2);
        datanode3.addBlock(b3);
        datanode4.addBlock(b4);

        heartbeats.add(datanode1);
        heartbeats.add(datanode2);
        heartbeats.add(datanode3);
        heartbeats.add(datanode4);

        datanodeMap.put(new UTF8("datanode1"), datanode1);
        datanodeMap.put(new UTF8("datanode2"), datanode2);
        datanodeMap.put(new UTF8("datanode3"), datanode3);
        datanodeMap.put(new UTF8("datanode4"), datanode4);

        TreeSet t1 = new TreeSet();
        TreeSet t2 = new TreeSet();
        TreeSet t3 = new TreeSet();

        TreeSet t4 = new TreeSet();
        TreeSet t5 = new TreeSet();
        TreeSet t6 = new TreeSet();

        t4.add(datanode1);
        t4.add(datanode4);
        t5.add(datanode2);
        t6.add(datanode3);

        t1.add(b1);
        t1.add(b4);

        t2.add(b2);
        t3.add(b3);

        blocksMap.put(b1, t4);
        blocksMap.put(b2, t5);
        blocksMap.put(b3, t6);

        excessReplicateMap.put(datanode1.getName(), t1);
        excessReplicateMap.put(datanode2.getName(), t2);
        excessReplicateMap.put(datanode3.getName(), t3);

        System.out.println("datanode1 = " + datanode1 + " datanode2 = " + datanode2 + " datanode3 = " + datanode3);

        Daemon hbthread = new Daemon(new HeartbeatMonitor());
        hbthread.setName("g1");
        hbthread.start();
    }

    public static void main(String[] args) throws IOException {
        HeartbeatMonitorTest heartbeatMonitorTest = new HeartbeatMonitorTest();
        heartbeatMonitorTest.start();
    }
}



```







好了，到这里基本就差不多结束了。最后我们将运行状态的图截取下来，大家看看吧。2333



最后说个小插曲，就是heartbeatCheck这个方法为什么嵌套了这么多的对象锁！！！

这个是必然的，方法上是this的对象锁，heartbeats和datanodeMap其它地方也会被用到

所以必须也要锁，否则这两部分数据会被影响。。。比如FSNamesystem#gotHeartbeat

方法也使用了，所以这是为什么要加几层锁的原因。





在IDEA对线程的debug呢，在我们设置小红点之后，右键小红点，如下图



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/herart001.png?raw=true)







![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart002.png?raw=true)











![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart003.png?raw=true)





数据被移除了。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart004.png?raw=true)









![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart005.png?raw=true)









![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart006.png?raw=true)







![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart007.png?raw=true)







![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/heart008.png?raw=true)