# Zookeeper学习笔记

> 目前仅是初步整理
>
> 参考资料：
>
> - Java高并发核心编程卷1
> - https://blog.csdn.net/qq_37960603/article/details/121835169
> - https://github.com/apache/curator/blob/master/curator-examples/src/main/java/cache/CuratorCacheExample.java

# 一、概述

这里以windwos平台为例

下载

创建data和log目录

![image-20221118142007523](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221118142007523.png)

在data和log目录中建三个文件夹

![image-20221118142122565](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221118142122565.png)

在data目录的每一个文件夹中创建一个myid的文件，然后文件内容为id

![image-20221118142222915](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221118142222915.png)

然后复制zoo_sample.cfg文件，复制三份，并修改配置文件内容，：

~~~properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=D:\zookeeper\apache-zookeeper-3.6.3-bin\data\zoo-1
dataLogDir=D:\zookeeper\apache-zookeeper-3.6.3-bin\log\zoo-1
# the port at which the clients will connect
clientPort=2181
#在3.6.x以上版本zookeeper有一个管理后台，默认占用8080端口，所以需要手动修改
admin.serverPort=2666
server.1=127.0.0.1:2888:3888
~~~

进入bin目录，复制zkServer.cmd，复制三分，并进行修改：

~~~bash
@echo off
setlocal
call "%~dp0zkEnv.cmd"

set ZOOCFG=D:\zookeeper\apache-zookeeper-3.6.3-bin\conf\zoo-1.cfg

set ZOOMAIN=org.apache.zookeeper.server.quorum.QuorumPeerMain
set ZOO_LOG_FILE=zookeeper-%USERNAME%-server-%COMPUTERNAME%.log

echo on
call %JAVA% "-Dzookeeper.admin.enableServer=false -Dzookeeper.log.dir=%ZOO_LOG_DIR%" "-Dzookeeper.root.logger=%ZOO_LOG4J_PROP%" "-Dzookeeper.log.file=%ZOO_LOG_FILE%" "-XX:+HeapDumpOnOutOfMemoryError" "-XX:OnOutOfMemoryError=cmd /c taskkill /pid %%%%p /t /f" -cp "%CLASSPATH%" %ZOOMAIN% "%ZOOCFG%" %*

endlocal
~~~

然后分别启动三个cmd即可

# 二、客户端的使用

```xml
<!--curator版本需要与zookeeper版本匹配,这里-->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.2.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-client</artifactId>
    <version>5.2.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.2.0</version>
</dependency>
```

```java
public class ClientFactory {

    /**
     * 创建简单的CuratorFramework实例
     *
     * @param connectionStr 连接地址
     * @author Yuhaoran
     * @date 2022/11/18 10:38
     */
    public static CuratorFramework createSimple(String connectionStr) {
        //重试策略，第一次重试等待1s，第二次重试等待2s，第三次4s。。。
        //baseSleepTimeMs:等待时间的基础单位，下面设定了1000ms
        //maxRetries:重试次数
        ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);
        //使用工厂类的静态方法
        //参数1：zk的连接地址；参数2：重试策略
        return CuratorFrameworkFactory.newClient(connectionStr, retry);
    }

    /**
     * 创建带参数的CuratorFramework实例
     *
     * @param connStr 连接地址
     * @param retryPolicy 重试策略
     * @param connTimeoutMs 连接超时时长
     * @param sessionTimeoutMs 会话超时时长
     * @author Yuhaoran
     * @date 2022/11/18 10:39
     */
    public static CuratorFramework createWithOptions(String connStr, RetryPolicy retryPolicy, int connTimeoutMs, int sessionTimeoutMs) {
        //通过builder构造者方法构造
        return CuratorFrameworkFactory.builder()
                .connectString(connStr)
                .retryPolicy(retryPolicy)
                .connectionTimeoutMs(connTimeoutMs)
                .sessionTimeoutMs(sessionTimeoutMs)
                .build();
    }
}
```

```java
@Slf4j
public class TestCurator {
    private static final String ZK_ADDRESS="127.0.0.1:2881";


    @Test
    public void createNode(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            //启动客户端，连接服务器
            client.start();
            //创建一个节点，节点数据为 payload
            String data = "Hello,ZK!";
            byte[] bytes = data.getBytes(StandardCharsets.UTF_8);
            String zkPath = "/test/CURD/node-1";
            //create方法可以创建Znode节点，但该方法不会立即创建节点，而是返回一个CreateBuilder构造者实例，并设置一些参数
            //最终通过forPath方法完成真正的节点创建工作
            client.create()
                    .creatingParentsIfNeeded()
                    //节点类型主要有四种
                    //PERSISTENT 持久化节点：持久节点，是指在节点创建后，就一直存在，直到有删除操作来主动清除这个节点。持久节点的生命周期是永久有效，不会因为创建该节点的客户端会话失效而消失
                    //PERSISTENT_SEQUENTIAL 持久化顺序节点：持久化顺序节点的每个父节点会为他的第一级子节点维护一份次序，会记录每个子节点创建的先后顺序
                    //EPHEMERAL临时节点：临时节点的生命周期和客户端会话绑定
                    //EPHEMERAL_SEQUENTIAL临时顺序节点：带有顺序编号的临时节点
                    .withMode(CreateMode.PERSISTENT)
                    .forPath(zkPath,bytes);

        }catch (Exception e){
            log.error(e.getMessage());
        }finally {
            client.close();
        }
    }
    @Test
    public void readNode(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            client.start();
            String zkPath = "/test/CURD/node-1";
            //无论是checkExists、getData、getChildren方法，都有一个共同的特点,就是方法法返回的是构造者实例，不会立即执行，最终需要调用forPath进行调用
            Stat stat = client.checkExists().forPath(zkPath);
            if (stat!=null){
                byte[] bytes = client.getData().forPath(zkPath);
                String data = new String(bytes);
                log.info("data={}",data);
                String parentPath = "/test";
                List<String> list = client.getChildren().forPath(parentPath);
                for (String child : list) {
                    log.info("child={}",child);
                }
            }
        }catch (Exception e){
            log.error(e.getMessage());
        }finally {
            client.close();
        }
    }

    //节点的更新操作，可以分为同步更新与异步更新,以下是同步更新
    @Test
    public void updateNode(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            client.start();
            String zkPath = "/test/CURD/node-1";
            String data = "Hello,Update_ZK";
            byte[] bytes = data.getBytes(StandardCharsets.UTF_8);
            //使用setData()方法可以进行同步更新
            client.setData().forPath(zkPath,bytes);
        }catch (Exception e){
            log.error(e.getMessage());
        }finally {
            client.close();
        }
    }

    //节点的更新操作，可以分为同步更新与异步更新,以下是异步更新
    @Test
    public void updateNodeAsync(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            AsyncCallback callback = new AsyncCallback.StringCallback() {
                @Override
                public void processResult(int i, String s, Object o, String s1) {
                    log.info("i={};\ns={};\no={};\ns1={}",i,s,o,s1);
                }
            };
            client.start();
            String zkPath = "/test/CURD/node-1";
            String data = "Hello,Update_Async_ZK";
            byte[] bytes = data.getBytes(StandardCharsets.UTF_8);
            //使用setData()方法可以进行同步更新
            client.setData()
                    .inBackground(callback)//设置回调实例
                    .forPath(zkPath,bytes);
        }catch (Exception e){
            log.error(e.getMessage());
        }finally {
            client.close();
        }
    }

    @Test
    public void deleteNode(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            client.start();
            String zkPath = "/test/CURD/node-1";
            //删除也可以异步，用法与更新类似
            client.delete().forPath(zkPath);
            String parentPath = "/test";
            List<String> list = client.getChildren().forPath(parentPath);
            for (String child : list) {
                log.info("child={}",child);
            }
        }catch (Exception e){
            log.error(e.getMessage());
        }finally {
            client.close();
        }
    }

}
```

示例：

```java
public class IDMaker {
    private static final String ZK_ADDRESS = "127.0.0.1:2181";

    private String createID(String prefix) {
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        try {
            client.start();
            String forPath = client
                    .create()
                    .creatingParentsIfNeeded()
                    .withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                    .forPath(prefix);
            return forPath;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    public String getID(String nodeName) {
        //返回创建的节点的路径名称
        String result = createID(nodeName);
        if (result == null) {
            return null;
        }
        int index = result.lastIndexOf(nodeName);
        if (index >= 0) {
            index += nodeName.length();
            return index < result.length() ? result.substring(index) : "";
        }
        return result;
    }
}
```

```java
@Test
public void testIDMaker(){
    IDMaker idMaker = new IDMaker();
    for (int i = 0; i < 10; i++) {
        System.out.println("####"+idMaker.getID("/test/IDMake-"));
    }
}
```

# 三、分布式事件监听

针对ZooKeeper服务端的节点操作事件监听，是客户端操作服务器的一项重点工作。在Curator的API中，事件监听有两种方式：一是标准的观察者模式，二是缓存监听模式。标准的观察者模式，是通过Watcher监听器实现，缓存监听模式则是引入了本地缓存Cache机制实现。简单地理解，Cache在客户端缓存了Znode的各种状态，当感知到Zookeeper集群的Znode变化，会触发event事件，注册到这些事件上的监听器会处理这些事件。虽然Cache是缓存机制，但是可以借助Cache实现事件的监听，并且Watcher只能监听一次，Cache可以多次监听。

下面是Watcher的用法：

```java
@Slf4j
public class TestZkWatcher {

    private static final String ZK_ADDRESS="127.0.0.1:2181";
    private final String  workerPath = "/test/listener/remoteNode";
    private final String subWorkerPath = "/test/listener/remoteNode/id-";



    @Test
    public void testWatcher(){
        CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
        client.start();
        try {
            Stat stat = client.checkExists().forPath(workerPath);
            if (stat==null){
                client.create().creatingParentsIfNeeded().forPath(workerPath);
            }
            //构建Watcher实例
            Watcher watcher = new Watcher() {
                //process是回调方法；Watcher只能监听一次
                @Override
                public void process(WatchedEvent watchedEvent) {
                    //输出结果为：
                    //####监听的变化 watchEvent=WatchedEvent state:SyncConnected type:NodeDataChanged path:/test/listener/remoteNode
                    System.out.println("####监听的变化 watchEvent="+watchedEvent);
                }
            };
            //一般来说可以通过GetDataBuilder、GetChildrenBuilder、ExistsBuidler等实现了Watchable接口的构造者实例，通过其usingWatcher方法使用Watcher监听器
            //比如下面getData()返回的就是一个GetDataBuilder实例
            byte[] bytes = client.getData().usingWatcher(watcher).forPath(workerPath);
            log.info("监听节点内容={}",new String(bytes));
            client.setData().forPath(workerPath,"第一次变动".getBytes(StandardCharsets.UTF_8));
            client.setData().forPath(workerPath,"第二次变动".getBytes(StandardCharsets.UTF_8));
            Thread.sleep(Integer.MAX_VALUE);
        }catch (Exception e){
            e.printStackTrace();
        }


    }
}
```

在WatchedEvent中一共有三个属性：keeperState通知状态、eventType事件类型、path节点路径，三个属性。关于通知状态和事件类型如下图：（图片来自高并发核心编程卷一）

![image-20221129100029635](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221129100029635.png)

Curator引入的Cache缓存实现，Cache缓存拥有一个系列的类型，包括了Node Cache 、Path Cache、Tree Cache三组类：

- Node Cache节点缓存可以用于ZNode节点的监听；
- Path Cache子节点缓存用于ZNode的子节点的监听；
- Tree Cache树缓存是Path Cache的增强，不仅仅能监听子节点，也能监听ZNode节点自身。

下面是Node Cache的例子：

```java
@Test
public void testNodeCache(){
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    client.start();
    try {
        NodeCache nodeCache = new NodeCache(client,workerPath,false);
        NodeCacheListener listener = new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                ChildData currentData = nodeCache.getCurrentData();
                log.info("####节点变化,path={}",currentData.getPath());
                log.info("####节点变化,data={}",new String(currentData.getData(),StandardCharsets.UTF_8));
                log.info("####节点变化,stat={}",currentData.getStat());
            }
        };
        nodeCache.getListenable().addListener(listener);
        nodeCache.start();
        client.setData().forPath(workerPath,"第111111次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(1000);
        client.setData().forPath(workerPath,"第222222次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(1000);
        client.setData().forPath(workerPath,"第333333次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(Integer.MAX_VALUE);
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

下面是Path Cache的例子：



下面是Tree Cache的例子：



对于Zookeeper3.6.x之前版本，Curator提供的Cache分为Path Cache、NodeCache、Tree Cache，三者隶属于三个独立的类。对于之后的版本，统一使用一个工具类CuratorCache实现。就是说，Zookeeper3.6.x之后版本，Path Cache、Node Cache、Tree Cache均标记为废弃，建议使用Curator Cache。例子如下：

```java
@Test
public void testCuratorCache() {
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    client.start();
    try {
        /*默认已指定节点为根的整个子树都被缓存；如果使用SINGLE_NODE_CACHE则只会缓存指定的节点
            * CuratorCache.Options除了单节点还有两个选项：
            * COMPRESSED_DATA： 通过org.apache.curator.framework.api.GetDataBuilder.decompressed()解压数据
            * DO_NOT_CLEAR_ON_CLOSE：当通过close()关闭缓存时，会通过CuratorCacheStorage.clear()清除storage，此选项可以防止storage被清除
             * */
        CuratorCache curatorCache = CuratorCache.build(client, workerPath, CuratorCache.Options.SINGLE_NODE_CACHE);
        //通过lambda构建listener
        CuratorCacheListener listener = CuratorCacheListener.builder()
            //添加缓存中的数据时调用
            .forCreates(node -> System.out.println(String.format("####Node created: [%s]", node)))
            //修改缓存中的数据时调用
            .forChanges((oldNode, node) -> System.out.println(String.format("####Node changed. Old: [%s] New: [%s]", oldNode, node)))
            //删除缓存中的数据时调用
            .forDeletes(oldNode -> System.out.println(String.format("####Node deleted. Old value: [%s]", oldNode)))
            //初始化完成时调用
            .forInitialized(() -> System.out.println("####Cache initialized"))
            .build();
        curatorCache.listenable().addListener(listener);
        //匿名创建listener
        //            curatorCache.listenable().addListener(new CuratorCacheListener() {
        //                @Override
        //                public void event(Type type, ChildData oldData, ChildData data) {
        //                    switch (type.name()) {
        //                        case "NODE_CREATED":
        //                            System.out.println("####NODE_CREATED: " + oldData + " : " + data);
        //                            break;
        //                        case "NODE_CHANGED":
        //                            System.out.println("####NODE_CHANGED: " + oldData + " : " + data);
        //                            break;
        //                        case "NODE_DELETED":
        //                            System.out.println("####NODE_DELETED: " + oldData + " : " + data);
        //                            break;
        //                        default:
        //                            break;
        //                    }
        //                }
        //            });
        curatorCache.start();
        client.setData().forPath(workerPath, "第111111次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(1000);
        client.setData().forPath(workerPath, "第222222次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(1000);
        client.setData().forPath(workerPath, "第333333次变动".getBytes(StandardCharsets.UTF_8));
        Thread.sleep(Integer.MAX_VALUE);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

# 四、分布式锁

