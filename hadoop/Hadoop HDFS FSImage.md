###  Hadoop HDFS FSImage File



### FSImage数据是怎么写入的, 又是如何读取的?



在分析源代码之前，我们先把对应的代码剪出来，然后运行看看究竟是怎么回事。



```java



public class FSImageTest {

    public void saveFSImage() throws IOException {
        String fsImage = "/Users/bairong/fsimg";
        DataOutputStream out = 
            new DataOutputStream(new BufferedOutputStream(new FileOutputStream(fsImage)));
        String[] files = new String[7];
        try {
            out.writeInt(files.length - 1);
            String hdfsFileName = "/a";
            UTF8 utf8 = new UTF8(hdfsFileName);
            utf8.write(out);

            Integer[] blocks = new Integer[7];
            out.writeInt(blocks.length);

            Long blockId = -99887766L;
            Long blockLength = 520L;

            out.writeLong(blockId);
            out.writeLong(blockLength);
        } finally {
            out.close();
        }
    }

    public void readFSImage() throws IOException {
        String fsImage = "/Users/bairong/fsimg";
        DataInputStream in = new DataInputStream(new BufferedInputStream(new FileInputStream(fsImage)));

        // 获取HDFS上有多少个文件
        int numFiles = in.readInt();

        UTF8 name = new UTF8();
        // 获取HDFS文件名字
        name.readFields(in);

        // 获取文件block数量
        int numBlocks = in.readInt();

        // blockID信息
        long blockId = in.readLong();
        // block大小
        long len = in.readLong();

        System.out.println("HDFS总文件数: " + numFiles);
        System.out.println("读取的名字是: " + name);
        System.out.println("该文件的block数量: " + numBlocks);
        System.out.println("blockID: " + blockId);
        System.out.println("block大小: " + len);

    }


    public static void main(String[] args) throws IOException {
        FSImageTest fsImageTest = new FSImageTest();
        fsImageTest.saveFSImage();
        fsImageTest.readFSImage();
    }
}



```



那么， 通过上面生成出来fsimg长得是什么样子的呢？和HDFS生成的是一个样子的吗？



测试程序生成的文件, 我们可以看到写入的文件/a，然后包裹在其中的就是对应这个文件的数据信息了。

![image-20190107172729251](/var/folders/0m/wf03pfg10tb55bmmyfb3g7pc0000gn/T/abnerworks.Typora/image-20190107172729251.png)



现在有了这个基本知识了，我们就开始来看看源码把。



```java


/**
  org.apache.hadoop.dfs.FSDirectory.java
**/

public FSDirectory(File dir) throws IOException {
        // 获取已经存在的image文件夹
        // 该dir就是我们配置NameNode路径的数据
        File fullimage = new File(dir, "image");
        // 如果image不存在则需要对NameNode进行格式化
        if (! fullimage.exists()) {
          throw new IOException("NameNode not formatted: " + dir);
        }
    
        // 获取edits文件
        File edits = new File(dir, "edits");
    
        /**
        
            loadFSImage(核心):
              
        **/
        if (loadFSImage(fullimage, edits)) {
            saveFSImage(fullimage, edits);
        }

        synchronized (this) {
            this.ready = true;
            this.notifyAll();
            this.editlog = new DataOutputStream(new FileOutputStream(edits));
        }
}



```



```java


INode rootDir = new INode("", null, null);
TreeSet activeBlocks = new TreeSet();


boolean loadFSImage(File fsdir, File edits) throws IOException {
        //
        // Atomic move sequence, to recover from interrupted save
        //
        File curFile = new File(fsdir, FS_IMAGE);  // fsimage
        File newFile = new File(fsdir, NEW_FS_IMAGE); // fsimage.new
        File oldFile = new File(fsdir, OLD_FS_IMAGE); // fsimage.old

        // Maybe we were interrupted between 2 and 4
        if (oldFile.exists() && curFile.exists()) {
            oldFile.delete();
            if (edits.exists()) {
                edits.delete();
            }
        } else if (oldFile.exists() && newFile.exists()) {
            // Or maybe between 1 and 2
            newFile.renameTo(curFile);
            oldFile.delete();
        } else if (curFile.exists() && newFile.exists()) {
            // Or else before stage 1, in which case we lose the edits
            newFile.delete();
        }

        //
        // Load in bits
        // 如果存在fsimage元数据则先读取对应数据
        if (curFile.exists()) {
            DataInputStream in = 
                new DataInputStream(new BufferedInputStream(newFileInputStream(curFile)));
            try {
                // 获取有多少个文件
                // 例如我上传了 a|b|c|xxa|等等文件
                // 那么这里优先会读取出对应文件的/.file.crc文件
                // 即/a.crc|/b.crc|/.xxa.crc, 然后在读取/a|/b|/c|/xxa|在HDFS对应的文件数据
                int numFiles = in.readInt();
                for (int i = 0; i < numFiles; i++) {
                    UTF8 name = new UTF8();
                    // 读取对应的文件
                    name.readFields(in);
                    // 该文件有多少个block
                    int numBlocks = in.readInt();
                    if (numBlocks == 0) {
                        unprotectedAddFile(name, null);
                    } else {
                        Block blocks[] = new Block[numBlocks];
                        // 获取该文件的所有block数据块
                        for (int j = 0; j < numBlocks; j++) {
                            blocks[j] = new Block();
                            blocks[j].readFields(in);
                        }
                        unprotectedAddFile(name, blocks);
                    }
                }
            } finally {
                in.close();
            }
        }

        if (edits.exists() && loadFSEdits(edits) > 0) {
            return true;
        } else {
            return false;
        }
}







boolean unprotectedAddFile(UTF8 name, Block blocks[]) {
        synchronized (rootDir) {
            if (blocks != null) {
                // Add file->block mapping
                for (int i = 0; i < blocks.length; i++) {
                    // 记录还存活的block数
                    activeBlocks.add(blocks[i]);
                }
            }
            
            // 将当前block记录在对应目录下
            return (rootDir.addNode(name.toString(), blocks) != null);
        }
}

```





```java

class INode {
    
    public String name;
    public INode parent;
    public TreeMap children = new TreeMap();
    public Block blocks[];

    
    INode(String name, INode parent, Block blocks[]) {
        this.name = name;
        this.parent = parent;
        this.blocks = blocks;
    }
    
    INode addNode(String target, Block blks[]) {
            if (getNode(target) != null) {
                return null;
            } else {
                String parentName = DFSFile.getDFSParent(target);
                if (parentName == null) {
                    return null;
                }

                INode parentNode = getNode(parentName);
                if (parentNode == null) {
                    return null;
                } else {
                    String targetName = new File(target).getName();
                    INode newItem = new INode(targetName, parentNode, blks);
                    parentNode.children.put(targetName, newItem);
                    return newItem;
                }
            }
    }


```







![image-20190107172617427](/var/folders/0m/wf03pfg10tb55bmmyfb3g7pc0000gn/T/abnerworks.Typora/image-20190107172617427.png)