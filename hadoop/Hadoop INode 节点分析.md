

### Hadoop INode 节点分析



#### INode结构分析

在对源码分析之前，我们先来看看INode对象有些什么。



```java
public String name;  // 这个是我们当前路径名称
public INode parent; // 我们父节点对象
public TreeMap children = new TreeMap(); // 子节点对象
public Block blocks[]; // 对应的block块数据

// 在创建对象的时候就将这些基本信息配置好。
INode(String name, INode parent, Block blocks[]) {
  this.name = name;
  this.parent = parent;
  this.blocks = blocks;
}


```





额,,, 本来想围绕着这里为什么使用TreeMap来展开分析讨论的, 后来发现水平有限, 之后等会补好知识后我在回来补上。

好吧，下面就来看看INode对象吧，我这里是稍微修改过的程序，否则运行会有问题。。。

```java

public class INodeTest {

    public String name;
    public INodeTest parent;
    public TreeMap children = new TreeMap();
    public Block blocks[];

    /**
     */
    INodeTest(String name, INodeTest parent, Block blocks[]) {
        this.name = name;
        this.parent = parent;
        this.blocks = blocks;
    }

    /**
     * Check whether it's a directory
     * @return
     */
    synchronized public boolean isDir() {
        return (blocks == null);
    }

    /**
     * This is the external interface
     */
    INodeTest getNode(String target) {
        // 如果输入的路径最前面没有"/"或者没有输入路径, 则会返回null，表示路径有问题, 需要检查下
        if (! target.startsWith("/") || target.length() == 0) {
            return null;
        } else if (parent == null && "/".equals(target)) {
            // 如果我们要查询的节点没有父节点, 且路径为"/"则返回当前对象，就是根节点对象
            return this;
        } else {
            Vector components = new Vector();
            int start = 0;
            int slashid = 0;
            boolean flag = true; // 这里是我添加的一个标志
            // 首先找到"/"标志, 然后获取到对应位置进行截取。
            while (start < target.length() && (slashid = target.indexOf('/', start)) >= 0) {
                if (flag) // 如果是第一次进入, 需要在slashil+1, 否者获取不到"/", 下面会有问题
                    components.add(target.substring(start, slashid + 1));
                else
                    // 之后的话也需要加上"/"表示, 例如: /test/aa/bb/cc
                    // 会被解析成"/", "/aa", "/bb", "/cc" 这样的格式
                    components.add(target.substring(start - 1, slashid));
                flag = false;
                start = slashid + 1;
            }
          
            // 到最后找不到"/"了把剩下的数据也全部截取出来
            if (start < target.length()) {
                components.add(target.substring(start - 1));
            }
            return getNode(components, 0);
        }
    }

    /**
     */
    INodeTest getNode(Vector components, int index) {
        // 首先判断当前对象和被分段好的队列数据进行判断, 如果不匹配则找不到
        if (! name.equals((String) components.elementAt(index))) {
            return null;
        }
        if (index == components.size()-1) {
            return this;
        }

        // Check with children
        // 获取到对应的子节点对象
        INodeTest child = (INodeTest) children.get(components.elementAt(index+1));
        if (child == null) {
            return null;
        } else {
            // 递归调用
            return child.getNode(components, index+1);
        }
    }

    /**
     */
    INodeTest addNode(String target, Block blks[]) {
        // 如果节点已经存在, 则返回null
        if (getNode(target) != null) {
            return null;
        } else {
            String parentName = DFSFile.getDFSParent(target);
            // 如果没有父节点的名称...则返回null
            if (parentName == null) {
                return null;
            }

            // 获取到父节点, 除了root节点, 其余节点都存在父节点,
            // 当然不能同时存在多个root节点。
            INodeTest parentNode = getNode(parentName);
            if (parentNode == null) {
                return null;
            } else {
                // 剩下的就是创建新节点, 并设置基本信息，并把节点挂载到对应的父节点下。
                String targetName = new File(target).getName();
                INodeTest newItem = new INodeTest(targetName, parentNode, blks);
                parentNode.children.put(targetName, newItem);
                return newItem;
            }
        }
    }

    /**
     */
    boolean removeNode() {
        if (parent == null) {
            return false;
        } else {
            parent.children.remove(name);
            return true;
        }
    }

    /**
     * Collect all the blocks at this INode and all its children.
     * This operation is performed after a node is removed from the tree,
     * and we want to GC all the blocks at this node and below.
     */
    void collectSubtreeBlocks(Vector v) {
        if (blocks != null) {
            for (int i = 0; i < blocks.length; i++) {
                v.add(blocks[i]);
            }
        }
        for (Iterator it = children.values().iterator(); it.hasNext(); ) {
            INodeTest child = (INodeTest) it.next();
            child.collectSubtreeBlocks(v);
        }
    }


    public static void main(String[] args) {
        Block b1 = new Block(66778899L, 10000);
        Block b2 = new Block(-1122334455L, 500000);
        Block[] blocks = {b1, b2};
        INodeTest iNodeTest = new INodeTest("/", null, blocks);

        INodeTest iNodeTest1 = new INodeTest("/test", null, blocks);
        INodeTest iNodeTest2 = new INodeTest("/db", null, blocks);
        INodeTest iNodeTest3 = new INodeTest("/cc", null, blocks);

        iNodeTest.children.put("/test", iNodeTest1) ;
        iNodeTest1.children.put("/db", iNodeTest2);
        iNodeTest2.children.put("/cc", iNodeTest3);

        System.out.println("iNodeTest1 = " + iNodeTest1);
        System.out.println("iNodeTest2 = " + iNodeTest2);
        System.out.println("iNodeTest3 = " + iNodeTest3);



//        INodeTest node = iNodeTest.getNode("/test/db/cc");
        INodeTest node = iNodeTest.getNode("/test/db/cc" +
                "");
        System.out.println(node);

    }

}

```


这里, 我们只分析getNode和addNode这两个方法, 其余的这里就不会过多的分析。



下面先来看看getNode运行时候的一些状态把，这样能更好的理解哦。



![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/inode001.png?raw=true)



这里就把目标数据解析成红框中的样子了。







![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/inode002.png?raw=true)

s



这里的name就是当前调用对象中的name数据信息。我们是root对象调用的，所以是"/"





![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/inode003.png?raw=true)





最后，我们也可以看到root对象他的children对象就是"/test"，我们解析也获取到了，所以我们有对应的子节点对象，child不为空所以我们通过递归在调用。





![image-20190107172729251](https://github.com/basebase/img_server/blob/master/hadoop%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%9B%BE%E7%89%87%E9%9B%86%E5%90%88/inode004.png?raw=true)



可以看到，又是重复执行上面的操作。



那么, 其实addNode就很简单了, 判断有没有对应的节点, 如果存在对应的节点则返回null，然后获取到对应的父节点。
除了根节点都会存在父节点的，当然不可以出现多个根节点。之后就是创建新对象挂载到对应的父节点对象下。


剩下的一些内容，我也没怎么细看，主要是看了获取和添加两块。如果大家有不同意见欢迎和我交流。














