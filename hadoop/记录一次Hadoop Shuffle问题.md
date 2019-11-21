### Hadoop Shuffle问题记录(流水账记录)

#### 发生了什么?

平常我们计算任务通常在10点钟基本全部结束, 然而今天却执行了寥寥数个任务, 所以任务全部卡主, <br />
平常20分钟的任务, 竟然运行了长达7个小时之久...
而我们重要的数据报表完全没有可用的资源导致任务被kill。


#### 我的分析结论

* 怀疑是否死锁
* 验证是否真的为死锁
* 简单查看源码得出自己认为的结论

说简单点, 就是没动脑子, 一开始就是认为简单的资源问题, 没有多想, 毕竟线上运行很长时间了。<br />
但是, 觉得又不太对劲, 由于完全不知道是怎么引起的。（目前为止还是不知道什么操作引起的, 我能做到只有下面的记录）


我首先打开了hadoop任务运行日志, 发现很多reduce任务都被kill了, 第一步就是看看有没有人相关的分析文章。
还别说, 这哥们写的挺不错： http://www.chongblog.xyz/hadoop/reduce_kill%E5%88%86%E6%9E%90/

说实话, 他的问题几乎和我是一样的, 我也就怀疑是否为"死锁问题", 但我往下看的时候, 发现其实他的log日志和我的log并不一样。
但也已经足够了, 有了头绪之后，我再次观察我的log日志信息内容。

我想验证的是, 我碰到的是否为"死锁"
```Text
[EventFetcher for fetching Map Completion Events] org.apache.hadoop.mapreduce.task.reduce.EventFetcher attempt_1570524358804_357940_r_000000_0 Thread started: EventFetcher for fetching Map Completion Events
2019-11-19 02:28:47,957 INFO [fetcher#20] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: Assigning m22p185:13562 with 1 to fetcher#20
2019-11-19 02:28:47,957 INFO [fetcher#20] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: assigned 1 of 1 to m22p185:13562 to fetcher#20
2019-11-19 02:28:47,958 INFO [fetcher#1] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: Assigning m22p186:13562 with 1 to fetcher#1
2019-11-19 02:28:47,958 INFO [fetcher#1] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: assigned 1 of 1 to m22p186:13562 to fetcher#1
2019-11-19 02:28:47,960 INFO [fetcher#19] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: Assigning m30p162:13562 with 1 to fetcher#19
2019-11-19 02:28:47,960 INFO [fetcher#19] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: assigned 1 of 1 to m30p162:13562 to fetcher#19
2019-11-19 02:28:47,961 INFO [fetcher#2] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: Assigning m20p182:13562 with 1 to fetcher#2
2019-11-19 02:28:47,961 INFO [fetcher#2] org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl: assigned 1 of 1 to m20p182:13562 to fetcher#2
....
```

我搜索了一下：EventFetcher for fetching Map Completion Events] org.apache.hadoop.mapreduce.task.reduce.EventFetcher
发现了一些issue <br />

```Text
https://issues.apache.org/jira/browse/MAPREDUCE-5423 (下面两个链接是从这个衍生出来的)
https://issues.apache.org/jira/browse/MAPREDUCE-4842
https://issues.apache.org/jira/browse/MAPREDUCE-3721
```

当我看到下图的评论说：
![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle001.png?raw=true)

我甚至以为找到了真理, 或许就是资源死锁的问题。


然而, 到这里并没有结束, 我感觉我还是有点问题, 我找了找成功的日志信息看了看, 发现了一个问题:
```Text

2019-11-13 13:53:34,331 INFO [main] org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl: finalMerge called with 292 in-memory map-outputs and 0 on-disk map-outputs
2019-11-13 13:53:34,356 INFO [main] org.apache.hadoop.mapred.Merger: Merging 292 sorted segments
2019-11-13 13:53:34,357 INFO [main] org.apache.hadoop.mapred.Merger: Down to the last merge-pass, with 0 segments left of total size: 0 bytes
2019-11-13 13:53:34,365 INFO [main] org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl: Merged 292 segments, 584 bytes to disk to satisfy reduce memory limit
2019-11-13 13:53:34,366 INFO [main] org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl: Merging 1 files, 22 bytes from disk
2019-11-13 13:53:34,366 INFO [main] org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl: Merging 0 segments, 0 bytes from memory into reduce
2019-11-13 13:53:34,366 INFO [main] org.apache.hadoop.mapred.Merger: Merging 1 sorted segments
2019-11-13 13:53:34,369 INFO [main] org.apache.hadoop.mapred.Merger: Down to the last merge-pass, with 0 segments left of total size: 0 bytes
```

这是一个正常的log信息, 当完成了最后的sort就开始了reduce阶段了, 毕竟shuffle帮助reduce准备好了数据, 而我被kill的日志没有出现过finalMerge。
这也就导致我想一探究竟了。

```Text
http://bigdatadecode.club/MapReduce%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90--Reduce%E8%A7%A3%E6%9E%90.html (感觉不错, 一篇reduce分析文章)
```

说真的, 虽然里面有些内容我也不是能看懂, 其实我并不关心, 我只关心我的问题是怎么样产生的？
我自己很早之前下载过hadoop源码, hhh.(真心感谢上面那篇文章, 不然我可能又会迷失在代码的海洋之中。)

打开了我的idea, 我找到finalMerge在哪里被调用的代码片段。


![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle002.png?raw=true)

依次查找, 谁调用了close()


![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle003.png?raw=true)
找到这里, 我犯难了, 我不知道中间还有没有其它的弯弯绕绕。但我一想, 我的长时间运行的任务似乎从来没有给过输出过内容, 我继续往上查代码

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle004.png?raw=true)

我在这里判断, 是否shuffle还未完成, 导致资源没有被有效的释放呢?
自此, 我最终的结论只到了这里。


#### 2019-11-21新增(上下求索?)
![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle005.gif?raw=true)

这张图, 是我在debug源码进行测试的, 当我去修改里面代码来进行模拟测试, 可以的是后面很多代码没有log输出, 只有出错后才会存在。<br />
[动图不好看的话, 我把原图片一起上传。] <br /><br /><br /><br />

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle006.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle007.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle008.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle009.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle0010.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle0011.png?raw=true)

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/shuffle0012.png?raw=true)





#### 写在最后
虽然看似一长串的查找, 但其实我并不确定我的分析是否正确, 我只能按照我现有的逻辑分析排查, 文中有纰漏的地方还请多指教。
