### Hadoop HDFS Edits文件分析

#### HDFS Edit文件如何加载和写入

在我们贴出Edits加载和写入的部分源码之前，我想先向大家问一下，如果是由你来设计&开发，你会怎么做？

我们的设计方案和HDFS的方案有什么不同？区别和优缺点？

那么在后面的文章中，我希望大家保持这个状态来学习，包括我自己，也会写出自己的一些看法。

虽然可能会很菜。



下面就开始吧。



按照老规矩，我们先把edits文件加载和写入的代码单独摘出来，单独运行，这样方便理解。



``` java

public class LogEditTest {
    
    // 这里的几个常量, 代表着我们的操作。
    private static final byte OP_ADD = 0;
    private static final byte OP_RENAME = 1;
    private static final byte OP_DELETE = 2;
    private static final byte OP_MKDIR = 3;

    DataOutputStream editlog = null;
    File edits = null;

    public LogEditTest() throws IOException {
        // edits文件路径
        this.edits = new File("/Users/Joker/Desktop/", "edits_bk");
//        this.editlog = new DataOutputStream(new FileOutputStream(edits));
        loadFSEdits(this.edits);
    }




    public int loadFSEdits(File edits) throws IOException {
        int numEdits = 0;
        if (edits.exists()) { // 首先判断文件是否存在
            DataInputStream in = new DataInputStream(new BufferedInputStream(new FileInputStream(edits)));
            try {
                while (in.available() > 0) {
                    byte opcode = in.readByte();
                    numEdits++; // edits文件中又多次操作
                    switch (opcode) {
                        // 也就是说edits里面有hdfs文件名, hdfs block信息
                        case OP_ADD: { // 如果是添加
                            UTF8 name = new UTF8(); 
                            name.readFields(in); // 获取hdfs文件名
                            ArrayWritable aw = new ArrayWritable(Block.class);
                            aw.readFields(in);
                            Writable writables[] = (Writable[]) aw.get();
                            Block blocks[] = new Block[writables.length];
                            // 把writables的数据复制到Block数组中
                            System.arraycopy(writables, 0, blocks, 0, blocks.length);
//                            unprotectedAddFile(name, blocks);
                            break;

                        }
                        case OP_RENAME: { // 如果是重命名
                            UTF8 src = new UTF8();
                            UTF8 dst = new UTF8();
                            src.readFields(in);
                            dst.readFields(in);
//                            unprotectedRenameTo(src, dst);
                            break;
                        }
                        case OP_DELETE: {
                            UTF8 src = new UTF8();
                            src.readFields(in);
//                            unprotectedDelete(src);
                            break;
                        }
                        case OP_MKDIR: { // 如果是新建目录
                            UTF8 src = new UTF8();
                            src.readFields(in);
//                            unprotectedMkdir(src.toString());
                            break;
                        }
                        default: {
                            throw new IOException("Never seen opcode " + opcode);
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                in.close();
            }
        }

        return numEdits;
    }

    /**
      写入信息到edits文件
    */
    public void logEdit(byte op, Writable w1, Writable w2) {
        // 那这里用锁呢, 其实就是避免到时候被其它线程写入, 导致元数据不统一。
        synchronized (editlog) {
            try {
                // 写入对应事件0, 1, 2, 3
                editlog.write(op);
                if (w1 != null) { // 文件名
                    w1.write(editlog);
                }

                if (w2 != null) { // block信息
                    w2.write(editlog);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws IOException {
        LogEditTest logEditTest = new LogEditTest();
        UTF8 src = new UTF8("aa");
        // 也可以这样写 src.set("aa")来设置名字
        Block b1 = new Block(1122L, 10);
        Block b2 = new Block(-2211L, 20);
        Block blocks[] = {b1, b2};
//        logEditTest.logEdit(OP_ADD, src, new ArrayWritable(Block.class, blocks));
    }
}

```







来看看程序在加载的时候，具体是什么样子把。



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/edits_001.png?raw=true)









![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/edits_002.png?raw=true)

这里通过程序我们更能直观的看到数据的变化。



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/edits_003.png?raw=true)



然后numEdits++一次。完成一次操作。



好了，加载过程完成后，我们在看看写入的操作把!





![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/edits_004.png?raw=true)







![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/edits_005.png?raw=true)



可以看到，写的话其实就是把需要对应的事件code以及文件名字和block信息写进去就完事了。



