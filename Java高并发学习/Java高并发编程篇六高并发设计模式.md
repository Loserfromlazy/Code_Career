# Java高并发编程篇六高并发设计模式和异步回调

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第六部分高并发设计模式和异步回调篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要对高并发场景常用的几种模式进行学习：线程安全的单例模式、ForkJoin模式、生产者-消费者模式、Master-Worker模式和Future模式。同时对核心模式异步回调模式进行详细的学习。

# 七、高并发设计模式

## 7.1 线程安全的单例模式

关于单例模式可以看我的[设计模式的笔记](https://github.com/Loserfromlazy/Code_Career/blob/master/java/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.md)关于单例模式的部分

> （这篇笔记除了单例模式以外的其他部分可能不是很全，后续会陆续的补全，但是单例模式是全的）

## 7.2 Master-Worker模式

Master-Worker模式是一种常见的高并发模式，它的核心思想是任务的调度和执行分离，调度任务的角色为Master，执行任务的角色为Worker，Master负责接收、分配任务和合并（Merge）任务结果，Worker负责执行任务。Master-Worker模式是一种归并类型的模式。

### 7.2.1 Master-Worker模式的参考实现

j举个例子需要执行N个任务，将这些任务的结果进行累加求和，如果任务太多，就可以采用Master-Worker模式来实现。Master持有workerCount个Worker，并且负责接收任务，然后分发给Worker，最后在回调函数中对Worker的结果进行归并求和。

先创建一个任务类：

```java
@Data
public class Task<R> {
    static AtomicInteger index =new AtomicInteger(1);
    //Consumer表示接受单个输入参数但不返回结果的操作。
    //任务的回调函数
    public Consumer<Task<R>> resultAction;
    //任务的id
    private int id;
    //worker ID
    private int workerId;
    //计算结果
    R result = null;

    public Task() {
        this.id = index.getAndIncrement();
    }

    public void execute(){
        this.result = this.doExecute();
        resultAction.accept(this);
    }

    /**
     * 由子类实现
     * @return R 执行结果
     */
    protected R doExecute(){
        return null;
    }
}
```

然后创建一个Worker类

```java
public class Worker<T extends Task, R> {

    //接收任务的阻塞队列
    private LinkedBlockingQueue<T> taskQueue = new LinkedBlockingQueue<>();
    //Worker的编号
    static AtomicInteger index = new AtomicInteger(1);
    //worker id
    private int workerId;
    //执行任务的线程
    private Thread thread = null;

    public Worker() {
        this.workerId = index.getAndIncrement();
        thread = new Thread(this::run);
        thread.start();
    }

    //轮询执行任务
    public void run() {
        for (; ; ) {
            try {
                T task = this.taskQueue.take();
                task.setWorkerId(workerId);
                task.execute();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    //接收任务到异步队列
    public void submit(T task, Consumer<R> action){
        //设置任务的回调函数
        task.resultAction = action;
        try {
            this.taskQueue.put(task);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

然后在创建一个Master类：

```java
public class Master<T extends Task, R> {
    //所有worker的集合
    private HashMap<String, Worker<T, R>> workers = new HashMap<>();
    //任务的集合
    private LinkedBlockingDeque<T> taskQueue = new LinkedBlockingDeque<>();
    //线程处理结果集合
    protected Map<String, R> resultMap = new ConcurrentHashMap<>();
    //Master的任务调度线程
    private Thread thread = null;
    //保持最终的和
    private AtomicLong sum = new AtomicLong(0);

    public Master(int workerCount) {
        //每个worker对象都需要有queue的引用，用于领任务和提交结果
        for (int i = 0; i < workerCount; i++) {
            Worker<T, R> worker = new Worker<>();
            workers.put("子节点" + i, worker);
        }
        thread = new Thread(() -> this.execute());
        thread.start();
    }

    //提交任务
    public void submit(T task) {
        taskQueue.add(task);
    }

    //获取worker结果处理的回调函数
    private void resultCallBack(Object o) {
        Task<R> task = (Task<R>) o;
        String taskName = "Worker:" + task.getWorkerId() + "- Task:" + task.getId();
        R result = task.getResult();
        resultMap.put(taskName, result);

        sum.getAndAdd(Long.parseLong(result.toString()));
    }

    //启动所有的子任务
    public void execute() {
        for (; ; ) {
            for (Map.Entry<String, Worker<T, R>> entry : workers.entrySet()) {
                T task = null;
                try {
                    task = this.taskQueue.take();
                    Worker<T, R> worker = entry.getValue();
                    worker.submit(task, this::resultCallBack);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public void printResult() {
        System.out.println("---sum is" + sum.get());
        for (Map.Entry<String, R> entry : resultMap.entrySet()) {
            String key = entry.getKey();
            System.out.println(key + " -value:" + entry.getValue());
        }
    }

}
```

最后编写测试类：

```java
public class Test {
    static class SimpleTask extends Task<Integer>{
        @Override
        protected Integer doExecute() {
            System.out.println("task"+getId()+"is done");
            return getId();
        }
    }

    public static void main(String[] args) {
        Master<SimpleTask,Integer> master = new Master(10);
        //这里测试方便，生产环境禁止
        ScheduledExecutorService service = Executors.newScheduledThreadPool(20);
        //定期向master提交任务
        service.scheduleAtFixedRate(()-> master.submit(new SimpleTask()),2,2, TimeUnit.SECONDS);
        //定期从master获取结果
        service.scheduleAtFixedRate(() -> master.printResult(),5,5, TimeUnit.SECONDS);
    }
}
```

### 7.2.2 Netty中Master-Worker模式的实现

Master-Worker模式的核心思想为分而治之，Master角色负责接收和分配任务，Worker角色负责执行任务和结果回填。高性能传输模式Reactor模式就是Master-Worker模式在传输领域的一种应用。基于Java的NIO技术，Netty设计了一套优秀的、高性能Reactor（反应器）模式的具体实现。

Netty是基于Reactor模式的具体实现，体现了Master-Worker模式的思想。Netty的EventLoop（Reactor角色）可以对应到Master-Worker模式的Worker角色，而Netty的EventLoopGroup轮询组则可以对应到Master-Worker模式的Master角色。

> 关于Netty的学习，可以参考本人的博客[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)或者是我的下一篇学习笔记Java高并发编程篇七NIO与Netty

### 7.2.3 Nginx中的Master-Worker模式的实现

大名鼎鼎的Nginx服务器是Master-Worker模式（更准确地说是Reactor模式）在高性能服务器领域的一种应用。Nginx在启动后会以daemon方式在后台运行，它的后台进程有两类：一类称为Master进程（相当于管理进程），另一类称为Worker进程（工作进程）。

Nginx的Master进程主要负责调度Worker进程，比如加载配置、启动工作进程、接收来自外界的信号、向各Worker进程发送信号、监控Worker进程的运行状态等。Master进程负责创建监听套接口，交由Worker进程进行连接监听。Worker进程主要用来处理网络事件，当一个Worker进程在接收一条连接通道之后，就开始读取请求、解析请求、处理请求，处理完成产生的数据后，再返回给客户端，最后断开连接通道。Nginx的架构非常直观地体现了Master-Worker模式的思想。Nginx的Master进程可以对应到Master-Worker模式的Master角色，Nginx的Worker进程可以对应到Master-Worker模式的Worker角色。

## 7.3 ForkJoin模式

“分而治之”是一种思想，所谓“分而治之”，就是把一个复杂的算法问题按一定的“分解”方法分为规模较小的若干部分，然后逐个解决，分别找出各部分的解，最后把各部分的解组成整个问题的解。“分而治之”思想在软件体系结构设计、模块化设计、基础算法中得到了非常广泛的应用。许多基础算法都运用了“分而治之”的思想，比如二分查找、快速排序等。

### 7.3.1 原理

ForkJoin模式先把一个大任务分解成许多个独立的子任务，然后开启多个线程并行去处理这些子任务。有可能子任务还是很大而需要进一步分解，最终得到足够小的任务。ForkJoin模式的任务分解和执行过程大致如图所示（图片来自于原书Java高并发核心编程卷2）：

![image-20220502185823491](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220502185823491.png)

ForkJoin模式借助了现代计算机多核的优势并行处理数据。通常情况下，ForkJoin模式将分解出来的子任务放入双端队列中，然后几个启动线程从双端队列中获取任务并执行。子任务执行的结果放到一个队列中，各个线程从队列中获取数据，然后进行局部结果的合并，得到最终结果。

### 7.3.2 ForkJoin框架

JUC包提供了一套ForkJoin框架的实现，具体以ForkJoinPool线程池的形式提供，并且该线程池在Java 8的Lambda并行流框架中充当着底层框架的角色。

JUC包的ForkJoin框架包含如下组件：

1. ForkJoinPool：执行任务的线程池，继承了AbstractExecutorService类。
2. ForkJoinWorkerThread：执行任务的工作线程（ForkJoinPool线程池中的线程）。每个线程都维护着一个内部队列，用于存放“内部任务”该类继承了Thread类。
3. ForkJoinTask：用于ForkJoinPool的任务抽象类，实现了Future接口。
4. RecursiveTask：带返回结果的递归执行任务，是ForkJoinTask的子类，在子任务带返回结果时使用。
5. RecursiveAction：不返回结果的递归执行任务，是ForkJoinTask的子类，在子任务不带返回结果时使用。

因为ForkJoinTask比较复杂，并且其抽象方法比较多，故在日常使用时一般不会直接继承ForkJoinTask来实现自定义的任务类，而是通过继承ForkJoinTask两个子类RecursiveTask或者RecursiveAction之一去实现自定义任务类，自定义任务类需要实现这些子类的compute()方法，该方法的执行流程一般如下：如果任务足够小就直接返回结果，否则就分割成N个子任务，然后依次调用每个子任务的fork方法执行子任务，依次调用每个子任务的join方法，等待子任务完成，然后合并执行结果。

下面我们通过计算0～100的累加求和这个小需求来熟悉ForkJoin：

首先继承ForkJoinTask的子类，编写可递归执行的异步任务类，代码如下：

```java
public class PlusTask extends RecursiveTask<Integer> {

    //任务的规模阈值
    private static final int THRESHOLD = 2;
    //累加的起始编号
    private int start;
    //累加的结束编号
    private int end;

    public PlusTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        //判断任务的规模，若规模小则直接计算
        boolean canCompute = (end -start) <= THRESHOLD;
        if (canCompute){
            for (int i = start; i <=end ; i++) {
                sum+=i;
            }
            System.out.println("执行任务，计算"+start+"到"+end+"的和。结果是"+sum);
        }else {
            System.out.println("任务过大，切割任务，将"+start+"到"+end+"的和一分为二");
            int middle = (end + start)/2;
            PlusTask plusLeft = new PlusTask(start,middle);
            PlusTask plusRight = new PlusTask(middle,end);
            //依次调用每个子任务的fork方法执行子任务
            plusLeft.fork();
            plusRight.fork();
            //等待子任务完成，依次调用每个子任务的join方法合并执行结果
            Integer leftResult = plusLeft.join();
            Integer rightResult = plusRight.join();
            sum = leftResult+rightResult;
        }
        return sum;
    }
}
```

编写测试类，使用ForkJoinPool调度PlusTask类：

```java
public class TestForkJoinPool {
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        //创建一个累加任务，从1-100
        PlusTask plusTask = new PlusTask(1, 100);
        Future<Integer> submit = forkJoinPool.submit(plusTask);
        Integer integer = submit.get(2, TimeUnit.SECONDS);
        System.out.println("最终结果" + integer);

    }
}
```

运行结果：

~~~
任务过大，切割任务，将1到100的和一分为二
任务过大，切割任务，将1到50的和一分为二
任务过大，切割任务，将26到50的和一分为二
......
执行任务，计算78到79的和。结果是157
执行任务，计算80到82的和。结果是243
最终结果5050
~~~

### 7.3.3 ForkJoin的核心API

ForkJoin框架的核心是ForkJoinPool线程池。该线程池使用一个无锁的栈来管理空闲线程，如果一个工作线程暂时取不到可用的任务，则可能被挂起，而挂起的线程将被压入由ForkJoinPool维护的栈中，待有新任务到来时，再从栈中唤醒这些线程。

在 ForkJoinPool 中，线程池中每个工作线程（ForkJoinWorkerThread）都对应一个任务队列（WorkQueue），工作线程优先处理来自自身队列的任务（LIFO或FIFO顺序，参数 mode 决定），然后以FIFO的顺序随机窃取其他队列中的任务。

我们看一下ForkJoinPool的构造函数：

```java
public ForkJoinPool() {
    this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
         defaultForkJoinWorkerThreadFactory, null, false);
}
public ForkJoinPool(int parallelism,//并行数，默认为CPU数，最小为1
                    ForkJoinWorkerThreadFactory factory,//线程创建工厂
                    UncaughtExceptionHandler handler,//异常处理程序
                    boolean asyncMode) {//是否是异步模式
    this(checkParallelism(parallelism),
         checkFactory(factory),
         handler,
         asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
         "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}
```

下面详细的了解构造函数的参数：

- parallelism：将依据parallelism设定的值决定框架内并行执行的线程数量。并行的每一个任务都会有一个线程进行处理，但parallelism属性并不是ForkJoin框架中最大的线程数量，该属性和ThreadPoolExecutor线程池中的corePoolSize、maximumPoolSize属性有区别，因为ForkJoinPool的结构和工作方式与ThreadPoolExecutor完全不一样。ForkJoin框架中可存在的线程数量和parallelism参数值并不是绝对关联的。

- factory：当ForkJoin框架创建一个新的线程时，同样会用到线程创建工厂。只不过这个线程工厂不再需要实现ThreadFactory接口，而是需要实现ForkJoinWorkerThreadFactory接口。后者是一个函数式接口，只需要实现一个名叫newThread()的方法。在ForkJoin框架中有一个默认的ForkJoinWorkerThreadFactory接口实现DefaultForkJoinWorkerThreadFactory。

- handler：异常捕获处理程序。当执行的任务中出现异常，并从任务中被抛出时，就会被handler捕获。

- asyncMode：asyncMode参数表示任务是否为异步模式，其默认值为false。如果asyncMode为true，就表示子任务的执行遵循FIFO（先进先出）顺序，并且子任务不能被合并；如果asyncMode为false，就表示子任务的执行遵循LIFO（后进先出）顺序，并且子任务可以被合并。

  虽然从字面意思来看asyncMode是指异步模式，它并不是指ForkJoin框架的调度模式采用是同步模式还是异步模式工作，仅仅指任务的调度方式。ForkJoin框架中为每一个独立工作的线程准备了对应的待执行任务队列，这个任务队列是使用数组进行组合的双向队列。asyncMode模式的主要意思指的是待执行任务可以使用FIFO（先进先出）的工作模式，也可以使用LIFO（后进先出）的工作模式，工作模式为FIFO（先进先出）的任务适用于工作线程只负责运行异步事件，不需要合并结果的异步任务。

**ForkJoinPool的common通用池**

很多场景可以直接使用ForkJoinPool定义的common通用池，调用ForkJoinPool.commonPool()方法可以获取该ForkJoin线程池，该线程池通过makeCommonPool()来构造

使用common池的优点是可以通过指定系统属性的方式定义“并行度、线程工厂和异常处理类”，并且common池使用的是同步模式，也就是说可以支持任务合并。

makeCommonPool源代码如下：

```java
/**
 * Creates and returns the common pool, respecting user settings
 * specified via system properties.
 */
private static ForkJoinPool makeCommonPool() {
    int parallelism = -1;
    ForkJoinWorkerThreadFactory factory = null;
    UncaughtExceptionHandler handler = null;
    try {  // ignore exceptions in accessing/parsing properties
        //并行度
        String pp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.parallelism");
        //线程工厂
        String fp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.threadFactory");
        //异常处理
        String hp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
        if (pp != null)
            parallelism = Integer.parseInt(pp);
        if (fp != null)
            factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                       getSystemClassLoader().loadClass(fp).newInstance());
        if (hp != null)
            handler = ((UncaughtExceptionHandler)ClassLoader.
                       getSystemClassLoader().loadClass(hp).newInstance());
    } catch (Exception ignore) {
    }
    if (factory == null) {
        if (System.getSecurityManager() == null)
            factory = defaultForkJoinWorkerThreadFactory;
        else // use security-managed default
            factory = new InnocuousForkJoinWorkerThreadFactory();
    }
    if (parallelism < 0 && // default 1 less than #cores
        (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
        parallelism = 1;
    if (parallelism > MAX_CAP)
        parallelism = MAX_CAP;
    return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                            "ForkJoinPool.commonPool-worker-");
}
```

当然系统属性可以通过以下方式指定，以parallelism参数为例：

~~~java
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism","8");
//或者是通过下面这个Java指令选项的方式指定parallelism值
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
~~~

**提交任务**

可以向ForkJoinPool线程池提交以下两类任务：

1. 外部任务（External/Submissions Task）提交向ForkJoinPool提交外部任务有三种方式：
   - 方式一是调用invoke()方法，该方法提交任务后线程会等待，等到任务计算完毕返回结果；
   - 方式二是调用execute()方法提交一个任务来异步执行，无返回结果；
   - 方式三是调用submit()方法提交一个任务，并且会返回一个ForkJoinTask实例，之后适当的时候可通过ForkJoinTask实例获取执行结果。
2. 子任务（Worker Task）提交向ForkJoinPool提交子任务的方法相对比较简单，由任务实例的fork()方法完成。当任务被分割之后，内部会调用ForkJoinPool.WorkQueue.push()方法直接把任务放到内部队列中等待被执行。

ForkJoinPool 中的任务分为两种：一种是本地提交的任务（如 execute、submit 提交的任务）；另外一种是 fork 出的子任务（Worker task）。两种任务都会存放在 WorkQueue 数组中，但是这两种任务并不会混合在同一个队列里，ForkJoinPool 内部使用了一种随机哈希算法（有点类似 ConcurrentHashMap 的桶随机算法）将工作队列与对应的工作线程关联起来，Submission 任务存放在 WorkQueue 数组的偶数索引位置，Worker 任务存放在奇数索引位。

### 7.3.4 工作窃取算法

ForkJoinPool线程池的任务分为“外部任务”和“内部任务”，两种任务的存放位置不同：

- 外部任务存放在ForkJoinPool的全局队列中。

  ```java
  static final class WorkQueue
  ```

- 子任务会作为“内部任务”放到内部队列中，ForkJoinPool池中的每个线程都维护着一个内部队列，用于存放这些“内部任务”。

由于ForkJoinPool线程池通常有多个工作线程，与之相对应的就会有多个任务队列，这就会出现任务分配不均衡的问题：有的队列任务多，忙得不停，有的队列没有任务，一直空闲。那么有没有一种机制帮忙将任务从繁忙的线程分摊给空闲的线程呢？有就是使用工作窃取算法。

工作窃取算法的核心思想是：工作线程自己的活干完了之后，会去看看别人有没有没干完的活，如果有就拿过来帮忙干。工作窃取算法的主要逻辑：**每个线程拥有一个双端队列（本地队列），用于存放需要执行的任务，当自己的队列没有任务时，可以从其他线程的任务队列中获得一个任务继续执行。**

在实际进行任务窃取操作的时候，操作线程会进行其他线程的任务队列的扫描和任务的出队尝试。（因为有线程安全问题所以是进行尝试）假如在窃取过程中该任务已经开始执行，那么任务的窃取操作就会失败。

如何尽量避免在任务窃取中发生的线程安全问题呢？一种简单的优化方法是：在线程自己的本地队列采取LIFO（后进先出）策略，窃取其他任务队列的任务时采用FIFO（先进先出）策略。简单来说，获取自己队列的任务时从头开始，窃取其他队列的任务时从尾开始。由于窃取的动作十分快速，会大量降低这种冲突，也是一种优化方式。

### 7.3.5 ForkJoin框架的原理

ForkJoin框架的核心原理大致如下：

1. ForkJoin框架的线程池ForkJoinPool的任务分为“外部任务”和“内部任务”。
2. “外部任务”放在ForkJoinPool的全局队列中。
3. ForkJoinPool池中的每个线程都维护着一个任务队列，用于存放“内部任务”，线程切割任务得到的子任务会作为“内部任务”放到内部队列中。
4. 当工作线程想要拿到子任务的计算结果时，先判断子任务有没有完成，如果没有完成，再判断子任务有没有被其他线程“窃取”，如果子任务没有被窃取，就由本线程来完成；一旦子任务被窃取了，就去执行本线程“内部队列”的其他任务，或者扫描其他的任务队列并窃取任务。
5. 当工作线程完成其“内部任务”，处于空闲状态时，就会扫描其他的任务队列窃取任务，尽可能不会阻塞等待。

总之，ForkJoin线程在等待一个任务完成时，要么自己来完成这个任务，要么在其他线程窃取了这个任务的情况下，去执行其他任务，是不会阻塞等待的，从而避免资源浪费，除非所有任务队列都为空。

ForkJoinPool适合需要“分而治之”的场景，特别是分治之后递归调用的函数，例如快速排序、二分搜索、大整数乘法、矩阵乘法、棋盘覆盖、归并排序、线性时间选择、汉诺塔问题等。**ForkJoinPool适合调度的任务为CPU密集型任务**，如果任务存在I/O操作、线程同步操作、sleep()睡眠等较长时间阻塞的情况，最好配合使用ManagedBlocker进行阻塞管理。总体来说，**ForkJoinPool不适合进行IO密集型、混合型的任务调度**。

## 7.4 生产者-消费者模式

生产者-消费者模式是一个经典的多线程设计模式，它为多线程间的协作提供了良好的解决方案，是高并发编程过程中常用的一种设计模式。在实际的软件开发过程中，经常会碰到如下场景：某些模块负责产生数据，另一些模块负责消费数据（此处的模块可以是类、函数、线程、进程等）。产生数据的模块可以形象地称为生产者，而消费数据的模块可以称为消费者。然而，仅仅抽象出来生产者和消费者还不够，该模式还需要有一个数据缓冲区作为生产者和消费者之间的中介：生产者把数据放入缓冲区，而消费者从缓冲区取出数据。

数据缓冲区的作用主要在于能使生产者和消费者解耦。如果没有数据缓冲区，让生产者直接调用消费者的某个方法，那么生产者对于消费者就会产生依赖（也就是耦合）。将来如果消费者的代码发生变化，可能会影响到生产者。而如果两者都依赖于某个缓冲区，两者之间不直接依赖，耦合也就相应降低了。

在生产者-消费者模式中，缓冲区是性能的关键，缓冲区可以基于ArrayList、LinkedList、BlockingQueue、环形队列等各种不同的数据存储组件去设计，所使用的组件不同，生产者-消费者模式实现的性能当然也就不同。

关于生产者-消费者模式的例子这里就不再赘述，可以看我的之前的学习笔记。

## 7.5 异步回调(Future)模式

Future模式是高并发设计与开发过程中常见的设计模式，它的核心思想是异步调用。

对于Future模式来说，它不是立即返回我们所需要的数据，但是它会返回一个契约（或异步任务），将来我们可以凭借这个契约（或异步任务）获取需要的结果。

Future模式的核心思想是异步调用，有点类似于异步的Ajax请求。当调用某个耗时方法时，可以不急于立刻获取结果，而是让被调用者立刻返回一个契约（或异步任务），并且将耗时的方法放到另外的线程中执行，后续凭契约再去获取异步执行的结果。在具体的实现上，Future模式和异步回调模式既有区别，又有联系。Java的Future实现类并没有支持异步回调，仍然需要主动获取耗时任务的结果；而Java 8中的CompletableFuture组件实现了异步回调模式。

# 八、高并发核心模式之异步回调模式

## 8.1  泡茶案例

华罗庚的课文——《统筹方法》，里面举了一个合理安排工序以便提升效率的泡茶案例。我们就以这个为例使用阻塞模式和异步回调模式分别实现其中的异步泡茶流程。

为了异步执行整个泡茶流程，分别设计三个线程：泡茶线程（MainThread，主线程）、烧水线程（HotWaterThread）和清洗线程（WashThread）。

- 泡茶线程的工作是：启动清洗线程、启动烧水线程，等清洗、烧水的工作完成后，泡茶喝；
- 清洗线程的工作是：洗茶壶、洗茶杯；
- 烧水线程的工作是：洗好水壶，灌上凉水，放在火上，一直等水烧开。

三个线程流程如下图：

![image-20220502211149177](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220502211149177.png)

## 8.2 join 异步阻塞实现

在泡茶的例子中，主线程通过分别调用烧水线程和清洗线程的join()方法，等待烧水线程和清洗线程执行完成，然后执行主线程自己的泡茶操作。流程如下：

![image-20220503101031686](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220503101031686.png)

> Java中线程的合并流程是：假设线程A调用线程B的join()方法去合并B线程，那么线程A进入阻塞状态，直到线程B执行完成。

下面我们通过代码来实现，程序中有三个线程：主线程main、烧水线程和清洗线程。main调用了thread.join()实例方法，合并烧水线程和清洗线程。

```java
public class JoinTest {

    public static final int SLEEP_GAP =500;
    public static String getCurrentThreadName(){
        return Thread.currentThread().getName();
    }
    static class HotWaterThread extends Thread{
        public HotWaterThread() {
            super("烧水线程");
        }

        @Override
        public void run() {
            try {
                System.out.println("洗水壶");
                System.out.println("灌凉水");
                System.out.println("放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println("水开了");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("运行结束");
        }
    }
    static class WashThread extends Thread{
        public WashThread() {
            super("清洗线程");
        }

        @Override
        public void run() {
            try {
                System.out.println("洗茶壶");
                System.out.println("洗茶杯");
                Thread.sleep(SLEEP_GAP);
                System.out.println("洗完了");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("运行结束");
        }
    }

    public static void main(String[] args) {
        Thread threadHotWater = new HotWaterThread();
        Thread threadWash = new WashThread();
        threadHotWater.start();
        threadWash.start();
        //在等待烧水和清洗的过程中，可以干点其他事情
        try {
            threadHotWater.join();
            threadWash.join();
            Thread.currentThread().setName("主线程");
            System.out.println("泡茶喝");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(getCurrentThreadName()+"运行结束");
    }
}
```

运行结果略

### 8.2.1 join方法详解

A线程调用B线程的join()方法，等待B线程执行完成，在B线程没有完成前，A线程阻塞。

源码如下：

```java
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

join的实现原理是不停地检查join线程是否存活，如果join线程存活，wait(0)就永远等下去，直至join线程终止后，线程的this.notifyAll()方法会被调用（该方法是在JVM中实现的，JDK中并不会看到源码），join()方法将退出循环，恢复业务逻辑执行。很显然这种循环检查的方式比较低效。

> 调用join()缺少很多灵活性，比如实际项目中很少自己单独创建线程，而是使用Executor，这进一步减少了join()的使用场景，所以join()的使用多数停留在Demo演示上。

## 8.3 FutureTask：异步调用实现

为了获取异步线程的返回结果，Java在1.5版本之后提供了一种新的多线程创建方式——FutureTask方式。通过FutureTask类和Callable接口的联合使用可以创建能获取异步执行结果的线程。

通过join的方式实现的泡茶案例是没办法获取返回结果的，所以我们现在用FutureTask来实现。

下面我们通过FutureTask来实现泡茶案例，总体流程跟join差不多，流程图如下：

![image-20220503101257074](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220503101257074.png)

下面我们来看一下代码：

```java
public class FutureTaskTest {
    public static final int SLEEP_GAP = 500;

    public static String getCurrentThreadName() {
        return Thread.currentThread().getName();
    }

    static class HotWaterThread implements Callable<Boolean> {

        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗水壶");
                System.out.println("灌凉水");
                System.out.println("放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println("水开了");
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }

    static class WashThread implements Callable<Boolean> {

        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗茶壶");
                System.out.println("洗茶杯");
                Thread.sleep(SLEEP_GAP);
                System.out.println("洗完了");
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }
    public static void main(String[] args) {
        Thread.currentThread().setName("主线程");
        FutureTask<Boolean> hotWaterTask = new FutureTask<>(new HotWaterThread());
        Thread threadHotWater = new Thread(hotWaterTask);
        FutureTask<Boolean> washTask = new FutureTask<>(new WashThread());
        Thread threadWash = new Thread(washTask);
        threadHotWater.start();
        threadWash.start();
        //..等待的过程中干其他的事情
        try {
            Boolean hotWaterResult = hotWaterTask.get();
            Boolean washResult = washTask.get();
            if (hotWaterResult&&washResult){
                System.out.println("泡茶喝");
            }else if (!washResult){
                System.out.println("被子没洗成，喝不了茶");
            }else if (!hotWaterResult){
                System.out.println("烧水失败，喝不了茶");
            }
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        System.out.println(getCurrentThreadName()+"运行结束");

    }
}
```

运行结果略。

FutureTask比join线程合并操作更加高明，能取得异步线程的结果。但是，也没有更好，因为通过FutureTask的get()方法获取异步结果时，主线程也会被阻塞。这一点FutureTask和join是一致的，它们都是异步阻塞模式。异步阻塞的效率往往比较低，被阻塞的主线程不能干任何事情，唯一能干的就是傻傻等待。原生Java API除了阻塞模式的获取结果外，并没有实现非阻塞的异步结果获取方法（当然jdk8中提供了Completable，但是出现的比较晚）。

## 8.4 异步回调和主动调用

前面使用join或者thread.get两种方式都属于主动调用。在泡茶喝的例子中，泡茶线程（主线程）是调用线程，烧水（或者清洗）线程是被调用线程，调用线程和被调用线程之间是一种主动关系，而不是被动关系。泡茶线程（主线程）需要主动获取烧水（或者清洗）线程的执行结果。

主动调用是一种阻塞式调用，它是一种单向调用，“调用方”要等待“被调用方”执行完毕才返回。如果“被调用方”的执行时间很长，那么“调用方”线程需要阻塞很长一段时间。

**如果将调用的方向进行反转就是异步回调**。回调是一种反向的调用模式，也就是说，被调用方在执行完成后，会反向执行“调用方”所设置的钩子方法。

Java中回调模式的标准实现类为CompletableFuture，由于该类出现的时间比较晚，因此很多著名的中间件如Guava、Netty等都提供了自己的异步回调模式API供开发者使用。开发者还可以使用RxJava响应式编程组件进行异步回调的开发。

## 8.5 Guava的异步回调

Guava是Google提供的Java扩展包，它提供了一种异步回调的解决方案。Guava中与异步回调相关的源码处于com.google.common.util.concurrent包中。包中的很多类都用于对java.util.concurrent的能力扩展和能力增强。

> 我们这里仅入门学习Guava的使用，如何进行异步回调实现泡茶案例。

### 8.5.1 FutureCallBack

Guava主要增强了Java而不是另起炉灶。为了实现异步回调方式获取异步线程的结果，Guava做了以下增强：

- 引入了一个新的接口ListenableFuture，继承了Java的Future接口，使得Java的Future异步任务在Guava中能被监控和非阻塞获取异步结果。
- 引入了一个新的接口FutureCallback，这是一个独立的新接口。该接口的目的是在异步任务执行完成后，根据异步结果完成不同的回调处理，并且可以处理异步结果。

FutureCallback是一个新增的接口，用来填写异步任务执行完后的监听逻辑。FutureCallback拥有两个回调方法：

1. onSuccess()方法，在异步任务执行成功后被回调。调用时，异步任务的执行结果作为onSuccess方法的参数被传入。
2. onFailure()方法，在异步任务执行过程中抛出异常时被回调。调用时，异步任务所抛出的异常作为onFailure方法的参数被传入。

FutureCallback的源码如下：

```java
public interface FutureCallback<V> {
  /**
   * Invoked with the result of the {@code Future} computation when it is successful.
   */
  void onSuccess(@Nullable V result);

  /**
   * Invoked when a {@code Future} computation fails or is canceled.
   *
   * <p>If the future's {@link Future#get() get} method throws an {@link ExecutionException}, then
   * the cause is passed to this method. Any other thrown object is passed unaltered.
   */
  void onFailure(Throwable t);
}
```

> Guava的FutureCallback与Java的Callable名字相近，实质不同，存在本质的区别：
>
> 1. Java的Callable接口代表的是异步执行的逻辑。
> 2. Guava的FutureCallback接口代表的是Callable异步逻辑执行完成之后，根据成功或者异常两种情形执行不同的方法。
>
> Guava是对Java Future异步回调的增强，使用Guava异步回调也需要用到Java的Callable接口。简单地说，只有在Java的Callable任务执行结果出来后，才可能执行Guava中的FutureCallback结果回调。
>
> Guava为了实现Callable和FutureCallback之间的监视关系引入了一个新接口ListenableFuture，它继承了Java的Future接口，增强了被监控的能力。

### 8.5.2 ListenableFuture

Guava的ListenableFuture接口是对Java的Future接口的扩展，可以理解为异步任务实例，源码如下：

```java
public interface ListenableFuture<V> extends Future<V> {
    //此addListener()方法只在Guava内部调用，在实际编程中，addListener()不会使用到。
  void addListener(Runnable listener, Executor executor);
}
```

ListenableFuture仅仅增加了一个addListener()方法。它的作用就是上面的FutureCallback成功或失败回调逻辑封装成一个内部的Runnable异步回调任务，在Callable异步任务完成后回调FutureCallback成功或失败回调逻辑。

> 在实际编程中可以使用Guava的Futures工具类，它有一个addCallback()静态方法，可以将FutureCallback的回调实例绑定到ListenableFuture异步任务。官方文档上给出了示例：
>
> The main purpose of addListener is to support this chaining. You will rarely use it directly, in part because it does not provide direct access to the Future result. (If you want such access, you may prefer Futures.addCallback.) 
>
> 翻译过来的大意就是addListener很少会直接使用，部分原因是它不提供对Future结果的直接访问。(你可以选择Futures.addCallback。)
>
> 下面是addCallback的官方示例：
>
> ~~~java
> ListenableFuture<QueryResult> future = ...;   
> addCallback(future, new FutureCallback<QueryResult>() {         
>     public void onSuccess(QueryResult result) {           
>         storeInCache(result);         
>     }         
>     public void onFailure(Throwable t) {           
>         reportError(t);         
>     }       
> });
> ~~~

### 8.5.3 ListenableFuture异步任务

如果要获取Guava的ListenableFuture异步任务实例，主要通过向线程池（ThreadPool）提交Callable任务的方式获取。不过，这里所说的线程池不是Java的线程池，而是经过Guava自己定制过的Guava线程池。

Guava线程池是对Java线程池的一种包装。创建Guava线程池的方法如下：

~~~java
public class Test {
    public static void main(String[] args) {
        ExecutorService javaPool = Executors.newFixedThreadPool(10);
        ListeningExecutorService guavaPool = MoreExecutors.listeningDecorator(javaPool);
        ListenableFuture<Integer> futureGuava = guavaPool.submit(() -> {
            Thread.sleep(500);
            System.out.println("task is running");
            return 1;
        });
        Futures.addCallback(futureGuava, new FutureCallback<Integer>() {
            @Override
            public void onSuccess(@Nullable Integer result) {
                System.out.println("success");
            }

            @Override
            public void onFailure(Throwable t) {
                System.out.println("failed");
            }
        });
    }
}
~~~

首先创建Java线程池，然后以其作为Guava线程池的参数再构造一个Guava线程池。有了Guava的线程池之后，就可以通过submit()方法来提交任务了，任务提交之后的返回结果就是我们所要的ListenableFuture异步任务实例。

Guava异步回调的流程如下：

1. 实现Java的Callable接口，创建异步执行逻辑。还有一种情况，如果不需要返回值，异步执行逻辑也可以实现Runnable接口。
2. 创建Guava线程池。
3. 将第一步创建的Callable/Runnable异步执行逻辑的实例提交到Guava线程池，从而获取ListenableFuture异步任务实例。
4. 创建FutureCallback回调实例，通过Futures.addCallback将回调实例绑定到ListenableFuture异步任务上。

### 8.5.4 Guava实现泡茶案例

我们先看一下流程图：

![image-20220503131022335](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220503131022335.png)

然后我们用代码来实现：

```java
public class GuavaTest {
    public static final int SLEEP_GAP = 4000;

    public static String getCurName(){
        return Thread.currentThread().getName();
    }
    static class HotWaterTask implements Callable<Boolean>{

        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println(getCurName()+"洗水壶");
                System.out.println(getCurName()+"灌凉水");
                System.out.println(getCurName()+"放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println(getCurName()+"水开了");
            } catch (InterruptedException e) {
                e.printStackTrace();
                System.out.println(getCurName()+"烧水失败");
                return false;
            }
            System.out.println(getCurName()+"烧水结束");
            return true;
        }
    }
    static class WashTask implements Callable<Boolean>{

        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println(getCurName()+"洗茶壶");
                System.out.println(getCurName()+"洗茶杯");
                Thread.sleep(SLEEP_GAP);
                System.out.println(getCurName()+"洗完了");
            } catch (InterruptedException e) {
                System.out.println(getCurName()+"清洗失败");
                return false;
            }
            System.out.println(getCurName()+"清洗结束");
            return true;
        }
    }

    static class DrinkTask{
        boolean hotWaterResult = false;
        boolean washResult = false;
        public void drinkTea(){
            if (hotWaterResult&&washResult){
                System.out.println(getCurName()+"泡茶喝");
                //喝完茶，没有热水了
                this.hotWaterResult = false;
            }
        }
    }

    public static void main(String[] args) {
        Thread.currentThread().setName("主线程-泡茶线程");
        DrinkTask drinkTask = new DrinkTask();
        //创建Java线程池
        ExecutorService javaPool = Executors.newFixedThreadPool(10);
        //包装Java线程池，构造guava线程池
        ListeningExecutorService guavaPool = MoreExecutors.listeningDecorator(javaPool);
        //启动烧水线程，获取ListenableFuture
        ListenableFuture<Boolean> submitWater = guavaPool.submit(new HotWaterTask());
        //设置回调钩子方法
        Futures.addCallback(submitWater, new FutureCallback<Boolean>() {
            @Override
            public void onSuccess(@Nullable Boolean result) {
                if (result){
                    drinkTask.hotWaterResult =true;
                    drinkTask.drinkTea();
                }
            }

            @Override
            public void onFailure(Throwable t) {
                System.out.println(getCurName()+"烧水失败，无法喝茶");
            }
        });
        //启动清洗线程，获取ListenableFuture
        ListenableFuture<Boolean> submitWash = guavaPool.submit(new WashTask());
        Futures.addCallback(submitWash, new FutureCallback<Boolean>() {
            @Override
            public void onSuccess(@Nullable Boolean result) {
                if (result){
                    drinkTask.washResult =true;
                    drinkTask.drinkTea();
                }
            }

            @Override
            public void onFailure(Throwable t) {
                System.out.println(getCurName()+"没洗杯子，无法喝茶");
            }
        });
        System.out.println(getCurName()+"干点其他事情");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(getCurName()+"主线程结束");
    }
}
```

执行结果：

~~~
pool-1-thread-1洗水壶
pool-1-thread-1灌凉水
pool-1-thread-1放在火上
pool-1-thread-2洗茶壶
pool-1-thread-2洗茶杯
主线程-泡茶线程干点其他事情
主线程-泡茶线程主线程结束
pool-1-thread-1水开了
pool-1-thread-1烧水结束
pool-1-thread-2洗完了
pool-1-thread-2清洗结束
pool-1-thread-2泡茶喝
~~~

可以看到主线程-泡茶线程早就执行结束了，泡茶喝的工作在异步回调方法drinkTea()中执行，执行的线程并不是“泡茶喝”线程，而是烧水线程或清洗线程这种被调用线程。

### 8.5.5 Guava和FutureTask的区别

总结一下Guava异步回调和Java的FutureTask异步调用的区别，具体如下：

1. FutureTask是主动调用的模式，“调用线程”主动获得异步结果，在获取异步结果时处于阻塞状态，并且会一直阻塞，直到拿到异步线程的结果。
2. Guava是异步回调模式，“调用线程”不会主动获得异步结果，而是准备好回调函数，并设置好回调钩子，执行回调函数的并不是“调用线程”自身，回调函数的执行者是“被调用线程”，“调用线程”在执行完自己的业务逻辑后就已经结束了，当回调函数被执行时，“调用线程”可能已经结束很久了。

> 和异步回调模式相比，使用FutureTask获取结果时，调用线程（如泡茶线程）多少存在阻塞；
>
> 使用FutureTask涉及三四个类或接口的使用，与join相比，使用起来比较繁琐

## 8.6 Netty的异步回调

Netty官方文档说明Netty的网络操作都是异步的。Netty源码中大量使用了异步回调处理模式。在Netty的业务开发层面，处于Netty应用的Handler处理程序中的业务处理代码也都是异步执行的。Netty和Guava一样，实现了自己的异步回调体系：Netty继承和扩展了JDKFuture系列异步回调的API，定义了自身的Future系列接口和类，实现了异步任务的监控、异步执行结果的获取。

> 关于Netty的使用可以见我的博客[NIO与Netty学习笔记](https://www.cnblogs.com/yhr520/p/15384520.html)

Netty对Java Future异步任务的扩展如下：

1. 继承Java的Future接口得到了一个新的属于Netty自己的Future异步任务接口，该接口对原有的接口进行了增强，使得Netty异步任务能够非阻塞地处理回调结果。注意，Netty没有修改Future的名称，只是调整了所在的包名，Netty的Future类的包名和Java的Future接口的包不同。
2. 引入了一个新接口——GenericFutureListener，用于表示异步执行完成的监听器。这个接口和Guava的FutureCallback回调接口不同。Netty使用了监听器的模式，异步任务执行完成后的回调逻辑抽象成了Listener监听器接口。可以将Netty的GenericFutureListener监听器接口加入Netty异步任务Future中，实现对异步任务执行状态的事件监听。

> 总体来说，在异步非阻塞回调的设计思路上，Netty和Guava是一致的。对应关系为：
>
> - Netty的Future接口可以对应到Guava的ListenableFuture接口。
> - Netty的GenericFutureListener接口可以对应到Guava的FutureCallback接口。

### 8.6.1 GenericFutureListener接口

和Guava的FutureCallback一样，Netty新增了一个接口，用来封装异步非阻塞回调的逻辑，那就是GenericFutureListener接口。我们看一下GenericFutureListener的源代码：

```java
public interface GenericFutureListener<F extends Future<?>> extends EventListener {

    /**
     * Invoked when the operation associated with the {@link Future} has been completed.
     *
     * @param future  the source {@link Future} which called this callback
     */
    void operationComplete(F future) throws Exception;
}
```

GenericFutureListener拥有一个回调方法operationComplete()，表示异步任务操作完成，在Future异步任务执行完成后回调此方法。

### 8.6.2 Netty的Future接口

Netty也对Java的Future接口进行了扩展，并且名称没有变，还是叫作Future接口，实现在io.netty.util.concurrent包中。类的结构如下（源码太长请自行查看）：

![image-20220503225931410](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220503225931410.png)

Netty的Future接口一般不会直接使用，使用过程中会使用它的子接口。Netty有一系列子接口，代表不同类型的异步任务，如ChannelFuture接口。ChannelFuture子接口表示Channel通道I/O操作的异步任务，如果在Channel的异步I/O操作完成后需要执行回调操作，就需要使用到ChannelFuture接口。

### 8.6.3 ChannelFuture

我们以ChannelFuture为例了解一下netty的异步回调。

在Netty网络编程中，网络连接通道的输入、输出处理都是异步进行的，都会返回一个ChannelFuture接口的实例。通过返回的异步任务实例可以为其增加异步回调的监听器。在异步任务真正完成后，回调执行。

我们看一个netty客户端的例子，代码如下：

```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        ChannelFuture future = new Bootstrap()
                .group(new NioEventLoopGroup())
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel channel) throws Exception {
                        channel.pipeline().addLast(new StringEncoder());
                    }
                })
                .connect(new InetSocketAddress("localhost", 8080));
        //可以进行阻塞时调用
        /*future.sync()
                .channel()
                .writeAndFlush("hello world");*/
        //下面就是异步回调，尾future增加监听器（即回调钩子方法）
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (future.isSuccess()){
                    System.out.println("建立连接成功");
                }else {
                    System.out.println("建立连接失败");
                    future.cause().printStackTrace();
                }
            }
        });
    }
}
```

> GenericFutureListener接口在Netty中是一个基础类型接口。在网络编程的异步回调中，一般使用Netty中提供的某个子接口，如ChannelFutureListener接口。在上面的代码中，使用到的是这个子接口。

# 九、Java 8的CompletableFuture异步回调

## 9.1 CompletableFuture详解

很多语言（如JavaScript）提供了异步回调，一些Java中间件（如Netty、Guava）也提供了异步回调API，为开发者带来了更好的异步编程工具。Java 8提供了一个新的、具备异步回调能力的工具类——CompletableFuture，该类实现了Future接口，还具备函数式编程的能力。CompletableFuture实现了Future和CompletionStage两个接口。该类的实例作为一个异步任务，可以在自己异步执行完成之后触发一些其他的异步任务，从而达到异步回调的效果。

CompletableFuture的结构如下：

![image-20220504124826888](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220504124826888.png)

CompletableFuture继承了两个结构，Future接口不再赘述，他除了Future接口还继承了CompletionStage接口。

### 9.1.1 CompletionStage接口

Stage是阶段的意思。CompletionStage代表某个同步或者异步计算的一个阶段，或者一系列异步任务中的一个子任务（或者阶段性任务）。每个CompletionStage子任务所包装的可以是一个Function、Consumer或者Runnable函数式接口实例。这三个常用的函数式接口的特点如下：

1. Function接口的特点是：有输入、有输出。包装了Function实例的CompletionStage子任务需要一个输入参数，并会产生一个输出结果到下一步。
2. Runnable接口的特点是：无输入、无输出。包装了Runnable实例的CompletionStage子任务既不需要任何输入参数，又不会产生任何输出。
3. Consumer接口的特点是：有输入、无输出。包装了Consumer实例的CompletionStage子任务需要一个输入参数，但不会产生任何输出。

多个CompletionStage构成了一条任务流水线，一个环节执行完成了可以将结果移交给下一个环节（子任务）。多个CompletionStage子任务之间可以使用链式调用，比如下面代码（来自于官方文档）：

~~~java
//stage是一个CompletionStage子任务
//x -> square(x)是一个Function类型的lambda表达式，被thenApply方法包装成一个CompletionStage子任务
stage.thenApply(x -> square(x))
    //x -> System.out.print(x)是一个Consumer的lambda表达式，被thenAccept方法包装成一个CompletionStage子任务
    .thenAccept(x -> System.out.print(x))
    //() -> System.out.println()是一个Runnable的lambda表达式，被tthenRun方法包装成一个CompletionStage子任务
    .thenRun(() -> System.out.println())
~~~

### 9.1.2 创建异步任务

CompletionStage子任务的创建是通过CompletableFuture完成的。CompletableFuture类提供了非常强大的Future的扩展功能来帮助我们简化异步编程的复杂性，提供了函数式编程的能力来帮助我们通过回调的方式处理计算结果，也提供了转换和组合CompletionStage()的方法。

CompletableFuture定义了一组方法用于创建CompletionStage子任务（或者阶段性任务），源码如下：

```java
/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the {@link ForkJoinPool#commonPool()} with
 * the value obtained by calling the given Supplier.
 * 子任务包装一个Supplier实例，并使用默认的线程池来执行
 */
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(asyncPool, supplier);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the given executor with the value obtained
 * by calling the given Supplier.
 * 子任务包装一个Supplier实例，并使用指定的线程池来执行
 */
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                   Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the {@link ForkJoinPool#commonPool()} after
 * it runs the given action.
 * 子任务包装一个Runnable实例，并调用默认线程池来执行
 */
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(asyncPool, runnable);
}

/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the given executor after it runs the given
 * action.
 * 子任务包装一个Runnable实例，并调用指定的线程池来执行
 */
public static CompletableFuture<Void> runAsync(Runnable runnable,
                                               Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```

> 在使用CompletableFuture创建CompletionStage子任务时，如果没有指定Executor线程池，在默认情况下CompletionStage会使用公共的ForkJoinPool线程池（源码如下）。
>
> ```java
> /**
>  * Default executor -- ForkJoinPool.commonPool() unless it cannot
>  * support parallelism.
>  */
> private static final Executor asyncPool = useCommonPool ?
>     ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
> ```

下面举个例子：

```java
//创建一个Runnable无输入无输出异步子任务
@Test
public void testRunAsync() throws ExecutionException, InterruptedException {
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        //模拟执行1s
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("finished");
    });
    //等待异步任务执行完成，主动调用
    future.get();
}

//创建一个Supplier无输入有输出异步子任务
@Test
public void testSupplyAsync() throws ExecutionException, InterruptedException {
    CompletableFuture<Long> future = CompletableFuture.supplyAsync(() -> {
        //模拟执行1s
        long l = System.currentTimeMillis();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("finished");
        return System.currentTimeMillis() - l;
    });
    //等待异步任务执行完成，主动调用
    Long aLong = future.get();
    System.out.println(aLong);
}
```

> 这里只需要调用get等待执行完成即可，因为supplyAsync或runAsync两个方法都会调用传入的线程池或内部公用线程池去执行。（具体可以自行查看源码）

### 9.1.3 设置异步任务回调钩子方法

可以为CompletionStage子任务设置特定的回调钩子，当计算结果完成或者抛出异常的时候，执行这些特定的回调钩子。设置子任务回调钩子的主要函数源码如下：

```java
//设置子任务完成时的回调狗钩子方法
public CompletableFuture<T> whenComplete(
    BiConsumer<? super T, ? super Throwable> action) {
    return uniWhenCompleteStage(null, action);
}
//设置子任务完成时的回调钩子方法，可能不在同一线程执行
public CompletableFuture<T> whenCompleteAsync(
    BiConsumer<? super T, ? super Throwable> action) {
    return uniWhenCompleteStage(asyncPool, action);
}
//设置子任务完成时的回调钩子方法，提交给线程池执行
public CompletableFuture<T> whenCompleteAsync(
    BiConsumer<? super T, ? super Throwable> action, Executor executor) {
    return uniWhenCompleteStage(screenExecutor(executor), action);
}
//设置异常处理的回调钩子
public CompletableFuture<T> exceptionally(
    Function<Throwable, ? extends T> fn) {
    return uniExceptionallyStage(fn);
}
```

我们下面给个例子：

```java
@Test
public void testComplete() throws ExecutionException, InterruptedException {
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("throw Exception");
        throw new RuntimeException("模拟发生异常");
    });
    //任务完成的回调钩子方法
    future.whenComplete(new BiConsumer<Void, Throwable>() {
        @Override
        public void accept(Void unused, Throwable throwable) {
            System.out.println("执行完成");
        }
    });
    //发生异常的回调钩子
    future.exceptionally(new Function<Throwable, Void>() {
        @Override
        public Void apply(Throwable throwable) {
            System.out.println("执行失败，异常信息："+throwable.getMessage());
            return null;
        }
    });
    //获取异步任务结果
    future.get();
}
```

运行结果：

~~~
throw Exception
执行失败，异常信息：java.lang.RuntimeException: 模拟发生异常
执行完成

java.util.concurrent.ExecutionException: java.lang.RuntimeException: 模拟发生异常
......
~~~

### 9.1.4 使用handle方法统一处理异常和结果

除了通过whenComplete、exceptionally设置完成钩子、异常钩子之外，还可以调用handle()方法统一处理结果和异常。

```java
//在执行任务的同一个线程中处理异常和结果
public <U> CompletableFuture<U> handle(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(null, fn);
}
//可能不在执行任务的同一个线程中处理异常和结果
public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(asyncPool, fn);
}
//在指定的线程池中处理异常和结果
public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) {
    return uniHandleStage(screenExecutor(executor), fn);
}
```

给个例子：

```java
@Test
public void testHandle() throws ExecutionException, InterruptedException {
    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("throw Exception");
        throw new RuntimeException("模拟发生异常");
    });
    future.handle(new BiFunction<Void, Throwable, Object>() {
        @Override
        public Object apply(Void unused, Throwable throwable) {
            if (throwable == null){
                System.out.println("没有发生异常");
            }else {
                System.out.println("发生了异常，异常信息："+throwable.getMessage());
            }
            return null;
        }
    });
    future.get();
}
```

运行结果略

### 9.1.5 线程池的使用

默认情况下，通过静态方法runAsync()、supplyAsync()创建的CompletableFuture任务会使用公共的ForkJoinPool线程池，默认的线程数是CPU的核数。可以通过命令指定：

~~~
option: -Djava.util.concurrent.ForkJoinPool.common.parallelism
~~~

> 如果所有CompletableFuture共享一个线程池，那么一旦有任务执行一些很慢的IO操作，就会导致线程池中的所有线程都阻塞在IO操作上，造成线程饥饿，进而影响整个系统的性能。建议不同的业务创建不同的线程池。

例子：

```java
@Test
public void testPool() throws ExecutionException, InterruptedException, TimeoutException {
    //这里仅为测试快捷使用，可以使用之前的自定义混合型线程池代替
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    CompletableFuture<Long> future = CompletableFuture.supplyAsync(() -> {
        //模拟执行1s
        long l = System.currentTimeMillis();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("finished");
        return System.currentTimeMillis() - l;
    },executorService);
    Long aLong = future.get(2, TimeUnit.SECONDS);
    System.out.println("执行时长"+aLong+"ms");
}
```

## 9.2 异步任务的串行执行

如果两个异步任务需要串行（一个任务依赖另一个任务）执行，可以通过CompletionStage接口的thenApply()、thenAccept()、thenRun()和thenCompose()四个方法来实现。

### 9.2.1 thenApply

thenApply有三种重载方法，源码如下：

```java
//后一个任务与前一个任务在同一个线程中执行
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}
//后一个任务与前一个任务不在同一个线程中执行
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}
//后一个任务在执行的线程池中执行
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}
```

thenApply的三个重载版本有一个共同的参数fn，该参数表示要串行执行的第二个异步任务，它的类型为Function。fn的类型声明涉及两个泛型参数，具体如下：

- 泛型参数 T：上一个任务所返回结果的类型。
- 泛型参数 U：当前任务的返回值类型。

我们举个例子，现在要调用thenApply分两步计算（10+10）*5：

```java
//（10+10）*5
@Test
public void testThenApply() throws ExecutionException, InterruptedException {
    CompletableFuture<Long> future = CompletableFuture.supplyAsync(new Supplier<Long>() {
        @Override
        public Long get() {
            long result = 10L + 10L;
            System.out.println("第一步结果"+result);
            return result;
        }
    }).thenApplyAsync(new Function<Long, Long>() {
        @Override
        public Long apply(Long aLong) {
            System.out.println("显示第一步结果"+aLong);
            return aLong*5;
        }
    });
    final Long aLong = future.get();
    System.out.println("result:"+aLong);
}
```

### 9.2.3 thenRun

thenRun()方法与thenApply()方法不同的是，不关心任务的处理结果。只要前一个任务执行完成，就开始执行后一个串行任务。

thenRun的源码如下：

```java
//后一个任务与前一个任务在同一个线程中执行
public CompletableFuture<Void> thenRun(Runnable action) {
    return uniRunStage(null, action);
}
//后一个任务与前一个任务不在同一个线程中执行
public CompletableFuture<Void> thenRunAsync(Runnable action) {
    return uniRunStage(asyncPool, action);
}
//后一个任务在执行的线程池中执行
public CompletableFuture<Void> thenRunAsync(Runnable action,
                                            Executor executor) {
    return uniRunStage(screenExecutor(executor), action);
}
```

thenRun()方法同thenApply()方法类似，不同的是前一个任务处理完成后，thenRun()并不会把计算的结果传给后一个任务，而且后一个任务也没有结果输出。

> thenRun系列方法中的action参数是Runnable类型的，所以thenRun()既不能接收参数又不支持返回值。

### 9.2.2 thenAccept

thenAccept()方法对thenRun()、thenApply()的特点进行了折中，调用此方法时后一个任务可以接收（或消费）前一个任务的处理结果，但是后一个任务没有结果输出。

thenAccept源码如下：

```java
//后一个任务与前一个任务在同一个线程中执行
public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}
//后一个任务与前一个任务不在同一个线程中执行
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) {
    return uniAcceptStage(asyncPool, action);
}
//后一个任务在执行的线程池中执行
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action,
                                               Executor executor) {
    return uniAcceptStage(screenExecutor(executor), action);
}
```

thenAccept系列方法的回调参数为action，它的类型为Consumer<? super T>接口。

> Consumer<T>接口的accept()方法可以接收一个参数，但是不支持返回值

### 9.2.4 thenCompose

thenCompose()方法在功能上与thenApply()、thenAccept()、thenRun()一样，可以对两个任务进行串行的调度操作，第一个任务操作完成时，将它的结果作为参数传递给第二个任务。

```java
public <U> CompletableFuture<U> thenCompose(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}

public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(asyncPool, fn);
}

public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn,
    Executor executor) {
    return uniComposeStage(screenExecutor(executor), fn);
}
```

thenCompose()方法要求第二个任务的返回值是一个CompletionStage异步实例。因此，可以调用CompletableFuture.supplyAsync()方法将第二个任务所要调用的普通异步方法包装成一个CompletionStage异步实例。

我们还用刚才的例子分两步计算（10+10）*5：

```java
@Test
public void testThenCompose() throws ExecutionException, InterruptedException {
    final CompletableFuture<Long> future = CompletableFuture.supplyAsync(new Supplier<Long>() {
        @Override
        public Long get() {
            long result = 10L + 10L;
            System.out.println("第一步结果"+result);
            return result;
        }
    }).thenCompose(new Function<Long, CompletionStage<Long>>() {
        @Override
        public CompletionStage<Long> apply(Long aLong) {
            return CompletableFuture.supplyAsync(new Supplier<Long>() {
                @Override
                public Long get() {
                    final long result = aLong * 2;
                    System.out.println("result:"+result);
                    return result;
                }
            });
        }
    });
    final Long aLong = future.get();
    System.out.println("最后的结果："+aLong);
}
```

这段程序的执行结果与调用thenApply()分两步计算（10+10）*2的结果是一样的。但是，thenCompose()所返回的不是第二个任务所要执行的普通异步方法`Supplier<Long>.get()`的直接计算结果，而是调用CompletableFuture.supplyAsync()方法将普通异步方法`Supplier<Long>.get()`包装成一个CompletionStage异步实例并返回。

### 9.2.5 区别

thenApply()、thenRun()、thenAccept()这三个方法的不同之处主要在于其核心参数fn、action、consumer的类型不同，分别为`Function<T,R>`、`Runnable`、`Consumer<? super T>`类型。但是，thenCompose()方法与thenApply()方法有本质的不同：

1. thenCompose()的返回值是一个新的CompletionStage实例，可以持续用来进行下一轮CompletionStage任务的调度。具体来说，thenCompose()返回的是包装了普通异步方法的CompletionStage任务实例，通过该实例还可以进行下一轮CompletionStage任务的调度和执行，比如可以持续进行CompletionStage链式（或者流式）调用。
2. thenApply()的返回值则简单多了，直接就是第二个任务的普通异步方法的执行结果，它的返回类型与第二步执行的普通异步方法的返回类型相同，通过thenApply()所返回的值不能进行下一轮CompletionStage链式（或者流式）调用。

## 9.3 异步任务的合并执行

如果某个任务同时依赖另外两个异步任务的执行结果，就需要对另外两个异步任务进行合并。

以泡茶喝为例，“泡茶喝”任务需要对“烧水”任务与“清洗”任务进行合并。

对两个异步任务的合并可以通过CompletionStage接口的thenCombine()、runAfterBoth()、thenAcceptBoth()三个方法来实现。这三个方法的不同之处主要在于其核心参数fn、action、consumer的类型不同，分别为`Function<T,R>`、`Runnable`、`Consumer<? super T>`类型。

### 9.3.1 thenCombine

thenCombine()会在两个CompletionStage任务都执行完成后，把两个任务的结果一起交给thenCombine()来处理。

```java
//合并代表第二步任务的CompletionStage实例，返回第三步任务的CompletionStage
public <U,V> CompletableFuture<V> thenCombine(
    CompletionStage<? extends U> other,
    BiFunction<? super T,? super U,? extends V> fn) {
    return biApplyStage(null, other, fn);
}
//不一定在同一个线程中执行第三步任务的CompletionSatge实例
public <U,V> CompletableFuture<V> thenCombineAsync(
    CompletionStage<? extends U> other,
    BiFunction<? super T,? super U,? extends V> fn) {
    return biApplyStage(asyncPool, other, fn);
}
//第三步任务的CompletionSatge实例在指定的线程池中运行
public <U,V> CompletableFuture<V> thenCombineAsync(
    CompletionStage<? extends U> other,
    BiFunction<? super T,? super U,? extends V> fn, Executor executor) {
    return biApplyStage(screenExecutor(executor), other, fn);
}
```

thenCombine()方法的调用者为第一步的CompletionStage实例，该方法的第一个参数为第二步的CompletionStage实例，该方法的返回值为第三步的CompletionStage实例。在逻辑上，**thenCombine()方法的功能是将第一步、第二步的结果合并到第三步上**。

thenCombine系列方法有两个核心参数：

1. other参数：表示待合并的第二步任务的CompletionStage实例。
2. fn参数：表示第一个任务和第二个任务执行完成后，第三步需要执行的逻辑。

fn参数的类型为`BiFunction<? super T,? super U,? extends V>`，该类型的声明涉及三个泛型参数，具体如下：

- 泛型参数 T：表示第一个任务所返回结果的类型。
- 泛型参数 U：表示第二个任务所返回结果的类型。
- 泛型参数 V：表示第三个任务所返回结果的类型。

接下来调用thenCombine分三步计算（50+50）*（50+50）:

```java
@Test
public void testThenCombine() throws ExecutionException, InterruptedException {
    final CompletableFuture<Long> future1 = CompletableFuture.supplyAsync(new Supplier<Long>() {
        @Override
        public Long get() {
            long result = 50L + 50L;
            System.out.println("first step" + result);
            return result;
        }
    });
    final CompletableFuture<Long> future2 = CompletableFuture.supplyAsync(new Supplier<Long>() {
        @Override
        public Long get() {
            long result = 50L + 50L;
            System.out.println("second step" + result);
            return result;
        }
    });
    final CompletableFuture<Long> future = future1.thenCombine(future2, new BiFunction<Long, Long, Long>() {
        @Override
        public Long apply(Long aLong, Long aLong2) {
            final long result = aLong * aLong2;
            System.out.println("second step:"+result);
            return result;
        }
    });
    final Long aLong = future.get();
    System.out.println("最终结果"+aLong);
}
```

### 9.3.2 runAfterBoth

runAfterBoth()方法跟thenCombine()方法不一样的是，runAfterBoth()方法不关心每一步任务的输入参数和处理结果。

源码如下：

```java
//合并代表第二步任务的CompletionStage实例，返回第三步任务的CompletionStage
public CompletableFuture<Void> runAfterBoth(CompletionStage<?> other,
                                            Runnable action) {
    return biRunStage(null, other, action);
}
//不一定在同一个线程中执行第三步任务的CompletionSatge实例
public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other,
                                                 Runnable action) {
    return biRunStage(asyncPool, other, action);
}
//第三步任务的CompletionSatge实例在指定的线程池中运行
public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other,
                                                 Runnable action,
                                                 Executor executor) {
    return biRunStage(screenExecutor(executor), other, action);
}
```

runAfterBoth()方法的调用者为第一步任务的CompletionStage实例，runAfterBoth()方法的第一个参数为第二步任务的CompletionStage实例，runAfterBoth()方法的返回值为第三步的CompletionStage实例。**在逻辑上，第一步任务和第二步任务是并行执行的，thenCombine()方法的功能是将第一步、第二步的结果合并到第三步任务上。**

### 9.3.3 thenAcceptBoth

thenAcceptBoth()方法对runAfterBoth()方法和thenCombine()方法的特点进行了折中，调用该方法，第三个任务可以接收其合并过来的第一个任务、第二个任务的处理结果，但是第三个任务（合并任务）却不能返回结果。

源码如下：

```java
//合并代表第二步任务的CompletionStage实例，返回第三步任务的CompletionStage
public <U> CompletableFuture<Void> thenAcceptBoth(
    CompletionStage<? extends U> other,
    BiConsumer<? super T, ? super U> action) {
    return biAcceptStage(null, other, action);
}
//不一定在同一个线程中执行第三步任务的CompletionSatge实例
public <U> CompletableFuture<Void> thenAcceptBothAsync(
    CompletionStage<? extends U> other,
    BiConsumer<? super T, ? super U> action) {
    return biAcceptStage(asyncPool, other, action);
}
//第三步任务的CompletionSatge实例在指定的线程池中运行
public <U> CompletableFuture<Void> thenAcceptBothAsync(
    CompletionStage<? extends U> other,
    BiConsumer<? super T, ? super U> action, Executor executor) {
    return biAcceptStage(screenExecutor(executor), other, action);
}
```

### 9.3.4 allOf等待所有任务结束

CompletionStage接口的allOf()会等待所有的任务结束，以合并所有的任务。thenCombine()只能合并两个任务，如果需要合并多个异步任务，那么可以调用allOf()。源码如下：

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
    return andTree(cfs, 0, cfs.length - 1);
}
```

举个例子：

```java
@Test
public void testAllOf() throws ExecutionException, InterruptedException {
    final CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> System.out.println("task1"));
    final CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> System.out.println("task2"));
    final CompletableFuture<Void> future3 = CompletableFuture.runAsync(() -> System.out.println("task3"));
    final CompletableFuture<Void> future4 = CompletableFuture.runAsync(() -> System.out.println("task4"));
    final CompletableFuture<Void> future = CompletableFuture.allOf(future1, future2, future3, future4);
    future.get();
}
```

## 9.4 异步任务的选择执行

CompletableFuture对异步任务的选择执行不是按照某种条件进行选择的，而是按照执行速度进行选择的：前面两个并行任务，谁的结果返回速度快，谁的结果将作为第三步任务的输入。

对两个异步任务的选择可以通过CompletionStage接口的applyToEither()、runAfterEither()和acceptEither()三个方法来实现。这三个方法的不同之处在于它的核心参数fn、action、consumer的类型不同，分别为`Function<T,R>`、`Runnable`、`Consumer<? super T>`类型。

### 9.4.1 applyToEither

两个CompletionStage谁返回结果的速度快，applyToEither()方法就用这个最快的CompletionStage的结果进行下一步（第三步）的回调操作。

```java
//和other任务进行速度pk，最快的返回结果，将结果用于执行fn回调函数
public <U> CompletableFuture<U> applyToEither(
    CompletionStage<? extends T> other, Function<? super T, U> fn) {
    return orApplyStage(null, other, fn);
}
//功能相同，不一定在同一个线程中执行fn回调函数
public <U> CompletableFuture<U> applyToEitherAsync(
    CompletionStage<? extends T> other, Function<? super T, U> fn) {
    return orApplyStage(asyncPool, other, fn);
}
//功能相同，在指定线程池中执行fn函数
public <U> CompletableFuture<U> applyToEitherAsync(
    CompletionStage<? extends T> other, Function<? super T, U> fn,
    Executor executor) {
    return orApplyStage(screenExecutor(executor), other, fn);
}
```

举个例子，调用applyToEither随机选择（10+10）和（50+50）：

```java
@Test
public void testApplyToEither() throws ExecutionException, InterruptedException {
    final CompletableFuture<Long> future1 = CompletableFuture.supplyAsync(() -> {
        long result = 10L + 10L;
        System.out.println("first" + result);
        return result;
    });
    final CompletableFuture<Long> future2 = CompletableFuture.supplyAsync(() -> {
        long result = 50L + 50L;
        System.out.println("second" + result);
        return result;
    });
    final CompletableFuture<Long> future = future1.applyToEither(future2, new Function<Long, Long>() {
        @Override
        public Long apply(Long aLong) {
            return aLong;
        }
    });
    final Long aLong = future.get();
    System.out.println(aLong);
}
```

### 9.4.2 runAfterEither

runAfterEither()方法的功能为：前面两个CompletionStage实例，任何一个完成了都会执行第三步回调操作。三个任务的回调函数都是Runnable类型的。源码如下：

```java
//和other任务进行速度pk，最快的返回结果，将结果用于执行fn回调函数
public CompletableFuture<Void> runAfterEither(CompletionStage<?> other,
                                              Runnable action) {
    return orRunStage(null, other, action);
}
//功能相同，不一定在同一个线程中执行fn回调函数
public CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other,
                                                   Runnable action) {
    return orRunStage(asyncPool, other, action);
}
//功能相同，在指定线程池中执行fn函数
public CompletableFuture<Void> runAfterEitherAsync(CompletionStage<?> other,
                                                   Runnable action,
                                                   Executor executor) {
    return orRunStage(screenExecutor(executor), other, action);
}
```

调用runAfterEither()方法，只要前面两个CompletionStage实例其中一个执行完成，就开始执行第三步的CompletionStage实例。

### 9.4.3 acceptEither

acceptEither()方法对applyToEither()方法和runAfterEither()方法的特点进行了折中，两个CompletionStage谁返回结果的速度快，acceptEither()就用那个最快的CompletionStage的结果作为下一步（第三步）的输入，但是第三步没有输出。源码如下：

```java
public CompletableFuture<Void> acceptEither(
    CompletionStage<? extends T> other, Consumer<? super T> action) {
    return orAcceptStage(null, other, action);
}

public CompletableFuture<Void> acceptEitherAsync(
    CompletionStage<? extends T> other, Consumer<? super T> action) {
    return orAcceptStage(asyncPool, other, action);
}

public CompletableFuture<Void> acceptEitherAsync(
    CompletionStage<? extends T> other, Consumer<? super T> action,
    Executor executor) {
    return orAcceptStage(screenExecutor(executor), other, action);
}
```

## 9.5 CompletableFuture的综合案例

### 9.5.1 使用CompletableFuture完成泡茶案例

```java
public class TeaTest {
    public static void main(String[] args) {
        final CompletableFuture<Boolean> hotWaterJob = CompletableFuture.supplyAsync(() -> {
            System.out.println("洗水壶");
            System.out.println("灌凉水");
            System.out.println("烧开水");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("水开了");
            return true;
        });

        final CompletableFuture<Boolean> washJob = CompletableFuture.supplyAsync(() -> {
            System.out.println("洗茶杯");
            System.out.println("洗茶壶");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("洗完了");
            return true;
        });
        final CompletableFuture<String> future = hotWaterJob.thenCombine(washJob, (finishHot, finishWash) -> {
            if (finishHot && finishWash) {
                System.out.println("泡茶喝");
                return "完成喝茶";
            }
            return "没喝成";
        });
        final String join = future.join();
        System.out.println(join);

    }
}
```













