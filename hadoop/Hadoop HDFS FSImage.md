###  Hadoop HDFS FSImage File



### FSImage数据是怎么写入的, 又是如何读取的?



在分析源代码之前，我们先把对应的代码剪出来，然后运行看看究竟是怎么回事。



```java



public class FSImageTest {

    static String FS_IMAGE = "fsimage";
    static String NEW_FS_IMAGE = "fsimage.new";
    static String OLD_FS_IMAGE = "fsimage.old";


    public void saveFSImage(File fullimage, File edits) throws IOException {

        String[] rootDir = new String[7];
        Integer[] blokcs = new Integer[6];

        File curFile = new File(fullimage, FS_IMAGE);
        File newFile = new File(fullimage, NEW_FS_IMAGE);
        File oldFile = new File(fullimage, OLD_FS_IMAGE);

        DataOutputStream out = null;
        try {
            out = new DataOutputStream(new BufferedOutputStream(new FileOutputStream(newFile)));
            out.writeInt(rootDir.length); // HDFS上总文件数


            String fullName = "/a";
            UTF8 utf8 = new UTF8(fullName); // 写入文件名
            utf8.write(out);

            out.writeInt(blokcs.length); // 写入文件对应block数量


            long blkId = -99887711; // block的id
            long len = 99881; // block的大小

            out.writeLong(blkId);
            out.writeLong(len);

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            out.close();
        }
    }


    public void loadFSImage(File currFile) {
        try {

            DataInputStream in = new DataInputStream(new BufferedInputStream((new FileInputStream(currFile))));

            // 总文件数量
            int numFiles = in.readInt();
            
            for (int i = 0; i < numFiles; i++) {
                UTF8 name = new UTF8();
                name.readFields(in);

                // 文件块数量
                int numBlocks = in.readInt();
                // 文件块id
                long blkId = in.readLong();
                // 文件大小
                long len = in.readLong();

                System.out.println("HDFS总文件数: " + numFiles);
                System.out.println("读取的名字是: " + name);
                System.out.println("该文件的block数量: " + numBlocks);
                System.out.println("blockID: " + blkId);
                System.out.println("block大小: " + len);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    public static void main(String[] args) throws IOException {
        File fullimage = new File("/Users/Joker/Desktop");
        File currFile = new File("/Users/Joker/Desktop/" + NEW_FS_IMAGE);
        FSImageTest fsImageTest = new FSImageTest();
        fsImageTest.saveFSImage(fullimage, null);

        fsImageTest.loadFSImage(currFile);
    }
}




```



那么， 通过上面生成出来fsimg长得是什么样子的呢？和HDFS生成的是一个样子的吗？



测试程序生成的文件, 我们可以看到写入的文件/a，然后包裹在其中的就是对应这个文件的数据信息了。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/fsimage_save.png?raw=true)



然后我们在看看读取这个文件的信息内容把。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/fsimage_read.png?raw=true)







现在有了这个基本知识了，我们就开始来看看源码把。



```java

/**
  org.apache.hadoop.dfs.FSDirectory
*/

public FSDirectory(File dir) throws IOException {
        File fullimage = new File(dir, "image");
        // 如果在我们配置namenode的文件夹中不存在image文件夹的话, 就表示没有格式化集群
        // 就需要我们优先进行格式化.
        if (! fullimage.exists()) {
          throw new IOException("NameNode not formatted: " + dir);
        }
        File edits = new File(dir, "edits");
    
        // 重点内容, 其实说实话看一下上面的测试程序也还好.
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

boolean loadFSImage(File fsdir, File edits) throws IOException {
        //
        // Atomic move sequence, to recover from interrupted save
        // 这里说是原子性的保存和移动, 但是有点没看明白, 晚点再细看下.
        // 这里说的就是下面这些文件的移动和删除规则.
        //
        File curFile = new File(fsdir, FS_IMAGE);
        File newFile = new File(fsdir, NEW_FS_IMAGE);
        File oldFile = new File(fsdir, OLD_FS_IMAGE);

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
    
        // 如果存在fsimage文件
        if (curFile.exists()) {
            // 读取fsimage文件
            DataInputStream in = new DataInputStream(new BufferedInputStream(new FileInputStream(curFile)));
            try {
                
                // 读取HDFS总文件数
                int numFiles = in.readInt();
                for (int i = 0; i < numFiles; i++) {
                    UTF8 name = new UTF8();
                    name.readFields(in);
                    // 读取文件block数量
                    int numBlocks = in.readInt();
                    if (numBlocks == 0) {
                        unprotectedAddFile(name, null);
                    } else {
                        Block blocks[] = new Block[numBlocks];
                        for (int j = 0; j < numBlocks; j++) {
                            // 把对应block的id和长度数量读取出来
                            blocks[j] = new Block();
                            blocks[j].readFields(in);
                        }
                        
                        // 把文件挂载到根节点上, 方便写入到fsimage中
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

```



下面，我们将debug以图片的形式截图出来。我们先看看看一个比较的文件图片。



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/data1.jpg?raw=true)







看到了吗，这里/xaa文件被切分成了2个block块了。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/fsimage1.jpg?raw=true)







我们在来看看这个block中相关信息，32MB一个block。

是不是很清楚了，现在剩下一个最关键的点就是unprotectedAddFile(name, blocks)方法了。

让我们点进去看看。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/b1.jpg?raw=true)







```java

/**
  org.apache.hadoop.dfs.FSDirectory
**/

boolean unprotectedAddFile(UTF8 name, Block blocks[]) {
        synchronized (rootDir) {
            if (blocks != null) {
                // Add file->block mapping
                for (int i = 0; i < blocks.length; i++) {
                    activeBlocks.add(blocks[i]);
                }
            }
            return (rootDir.addNode(name.toString(), blocks) != null);
        }
}



INode addNode(String target, Block blks[]) {
    
    		// 这里我没细看, 但是看大概的意思就是判断当前文件是否已经存在了。
            if (getNode(target) != null) {
                return null;
            } else {
                
                // 获取该文件的父节点路径
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
                    // 将节点挂载到父节点下面
                    parentNode.children.put(targetName, newItem);
                    return newItem;
                }
            }
}
```





那么，下面还是以图片的形式展示一下。



看到这个rootDir没，这就是我们在saveFSImage的时候写入的总文件数量。

![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/r1.jpg?raw=true)









![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/a1.jpg?raw=true)



自此，FSImage的基本功能就介绍的差不多了，剩下的就是原子性的操作，这个我还需要点时间研究看看。


我大概明白这个所谓的原子移动文件了，集群在创建的或者保存旧的fsimage文件时候如果遇到其它情况中途失败了，再次重启的时候会进行判断并修复。