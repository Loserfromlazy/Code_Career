# Zookeeper学习笔记

> 目前仅是初步整理
>
> 参考资料：
>
> - Java高并发核心编程卷1
> - https://cdn.modb.pro/db/237605

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
            Watcher watcher = new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    System.out.println("####监听的变化 watchEvent="+watchedEvent);
                }
            };
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

- Node Cache节点缓存可以用于ZNode节点的监听；
- Path Cache子节点缓存用于ZNode的子节点的监听；
- Tree Cache树缓存是Path Cache的增强，不仅仅能监听子节点，也能监听ZNode节点自身。

下面是NodeCache的例子：

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

对于Zookeeper3.6.x之前版本，Curator提供的Cache分为Path Cache、NodeCache、Tree Cache，三者隶属于三个独立的类。对于之后的版本，统一使用一个工具类CuratorCache实现。就是说，Zookeeper3.6.x之后版本，Path Cache、Node Cache、Tree Cache均标记为废弃，建议使用Curator Cache。例子如下：

```java
@Test
public void testCuratorCache() {
    CuratorFramework client = ClientFactory.createSimple(ZK_ADDRESS);
    client.start();
    try {
        CuratorCache curatorCache = CuratorCache.build(client, workerPath, CuratorCache.Options.SINGLE_NODE_CACHE);

        curatorCache.listenable().addListener(new CuratorCacheListener() {
            @Override
            public void event(Type type, ChildData oldData, ChildData data) {
                switch (type.name()) {
                    case "NODE_CREATED":
                        System.out.println("####NODE_CREATED: " + oldData + " : " + data);
                        break;
                    case "NODE_CHANGED":
                        System.out.println("####NODE_CHANGED: " + oldData + " : " + data);
                        break;
                    case "NODE_DELETED":
                        System.out.println("####NODE_DELETED: " + oldData + " : " + data);
                        break;
                    default:
                        break;
                }
            }
        });
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

