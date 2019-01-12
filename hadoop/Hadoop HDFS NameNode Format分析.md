###  Hadoop HDFS NameNode Format



### HDFS Format 格式化流程

我们知道，当我们在运行并启动我们的HDFS集群的时候，我们是需要进行一个首次的初始化的。

那么，HDFS又是如何初始化的呢？创建了哪些目录文件呢？中间的过程有做了哪些事情呢？



那么我们可以分为下面这几个步骤来说明HDFS NameNode Format的流程。

  1、首先判断是否已经格式化，如果没有格式化则进行格式化，如果已经格式化会询问你是否重新格式化。

  2、当你进行格式化的时候会读取你的dfs.name.dir配置目录，如果没有配置该目录项则使用默认的目录

        即/tmp/hadoop/dfs/name，该目录就是格式化后NameNode存放镜像数据的目录。

  3、在dfs.name.dir配置的目录下创建NameNode的image和edits







在阅读代码之前，我们先把需要的配置进行配置。（注意：我这里只是配置需要的值，HDFS配置请参考官网）

```xm
<property>
  <name>dfs.name.dir</name>
  <value>/documents/source_code/java/hadoop-name</value>
  <description>Determines where on the local filesystem the DFS name node
      should store the name table.</description>
</property>
```





那么通过上面我所说的流程，那我们来看看HDFS具体的实现代码。



``` java
/**
	org.apache.hadoop.dfs.NameNode.java
*/

public static void main(String argv[]) throws Exception {
    	
        // 这里我直接默认就格式化了, 当我们格式化过之后可以将其注释掉或者删除掉。
    	argv = new String[]{"-format"};
    	
        // 读取我们配置文件的类。
        Configuration conf = new Configuration();

        if (argv.length == 1 && argv[0].equals("-format")) {
          // 获取我们dfs.name.dir的目录, 如果不存在则使用默认的目录。
          File dir = getDir(conf);
          // 如果目录已经存在了, 那么询问使用要重新格式化文件系统
          if (dir.exists()) {
            System.err.print("Re-format filesystem in " + dir +" ? (Y or N) ");
            // 通过控制台输入判断, 如果是则进行format()函数, 否则退出。
            if (!(System.in.read() == 'Y')) {
              System.err.println("Format aborted.");
              System.exit(1);
            }
          }
            
          // 格式化NameNode
          format(conf);
          // 输出格式化的目录
          System.err.println("Formatted "+dir);
          System.exit(0);
        }

        NameNode namenode = new NameNode(conf);
        namenode.join();
}
```







```java
/**
  org.apache.hadoop.dfs.NameNode.java
*/

public static void format(Configuration conf) throws IOException {
      // 这里调用的是FSDirectory的一个静态方法, 我们点进去看看。
      FSDirectory.format(getDir(conf), conf);
}

private static File getDir(Configuration conf) {
      return new File(conf.get("dfs.name.dir", "/tmp/hadoop/dfs/name"));
}



```





```java

/***
  org.apache.hadoop.dfs.FSDirectory
*/

public static void format(File dir, Configuration conf)
      throws IOException {
        File image = new File(dir, "image");
        File edits = new File(dir, "edits");

        // 如果你已经存在了image和edits那么会先删除然后新建image文件夹
        // 那么我们来细看一下FileUtil.fullyDelete(image, conf)
        if (!((!image.exists() || FileUtil.fullyDelete(image, conf)) &&
              (!edits.exists() || edits.delete()) &&
              image.mkdirs())) {
          
          throw new IOException("Unable to format: "+dir);
        }
}
```



FileUtil.fullyDelete(image, conf)这一小段代码，涉及的地方还挺多了。我们先来看看它的调用过程。

FileUtil#fullyDelete(File, Configuration) -> FileUtil#fullyDelete(FileSystem, File)->FileSystem#delete(File)

->LocalFileSystem#deleteRaw(File)->LocalFileSystem#makeAbsolute(File)->LocalFileSystem#isAbsolute(File)

->LocalFileSystem#fullyDelete(File)



```java

/**
  org.apache.hadoop.fs.FileUtil.java
*/

public static boolean fullyDelete(File dir, Configuration conf) throws IOException {
        return fullyDelete(new LocalFileSystem(conf), dir);
}


public static boolean fullyDelete(FileSystem fs, File dir) throws IOException {
        return fs.delete(dir);
}





/***
  org.apache.hadoop.fs.FileSystem
*/

public boolean delete(File f) throws IOException {
      if (isDirectory(f)) {
        return deleteRaw(f);
      } else {
        deleteRaw(getChecksumFile(f));            // try to delete checksum
        return deleteRaw(f);
      }
}



/***
  org.apache.hadoop.fs.LocalFileSystem
*/

public boolean deleteRaw(File f) throws IOException {
        // 首先判断文件是否为绝对路径
        f = makeAbsolute(f);
        // 然后判断(image)是否为文件, 如果是文件夹则遍历文件夹下所有数据并删除
        if (f.isFile()) {
            return f.delete();
        } else return fullyDelete(f);
}


private File makeAbsolute(File f) {
      // 判断是否为绝对路径
      if (isAbsolute(f)) {
        return f;
      } else {
        return new File(workingDir, f.toString()).getAbsoluteFile();
      }
}


public boolean isAbsolute(File f) {
      // 通过File方法判断, 或者只要前缀为/或者\的路径都是绝对路径。
      return f.isAbsolute() ||
        f.getPath().startsWith("/") ||
        f.getPath().startsWith("\\");
}


 private boolean fullyDelete(File dir) throws IOException {
        dir = makeAbsolute(dir);
        // 获取image下所有文件
        File contents[] = dir.listFiles();
        // 
        if (contents != null) {
            for (int i = 0; i < contents.length; i++) {
                // 如果是文件则删除
                if (contents[i].isFile()) {
                    if (! contents[i].delete()) {
                        return false;
                    }
                } else {
                    // 如果是文件夹则递归在遍历删除
                    if (! fullyDelete(contents[i])) {
                        return false;
                    }
                }
            }
        }
        return dir.delete();
}


```





上面的执行过程就是Hadoop HDFS NameNode Format的全流程了。

本人水平有限，如有错误希望大家帮忙指出。谢谢！