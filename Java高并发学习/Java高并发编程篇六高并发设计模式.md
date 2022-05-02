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

> 关于Netty的学习，可以参考本人的博客[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)

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

### 8.1  泡茶案例

华罗庚的课文——《统筹方法》，里面举了一个合理安排工序以便提升效率的泡茶案例。我们就以这个为例使用阻塞模式和异步回调模式分别实现其中的异步泡茶流程。

为了异步执行整个泡茶流程，分别设计三个线程：泡茶线程（MainThread，主线程）、烧水线程（HotWaterThread）和清洗线程（WashThread）。

- 泡茶线程的工作是：启动清洗线程、启动烧水线程，等清洗、烧水的工作完成后，泡茶喝；
- 清洗线程的工作是：洗茶壶、洗茶杯；
- 烧水线程的工作是：洗好水壶，灌上凉水，放在火上，一直等水烧开。

三个线程流程如下图：

![image-20220502211149177](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220502211149177.png)

### 8.2 join 异步阻塞实现

在泡茶的例子中，主线程通过分别调用烧水线程和清洗线程的join()方法，等待烧水线程和清洗线程执行完成，然后执行主线程自己的泡茶操作。流程如下：

![image-20220502211623587](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220502211623587.png)

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

### 8.3 join方法详解



# 九、Java 8的CompletableFuture异步回调
