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

### 7.2.3 Nginx中的Master-Worker模式的实现

## 7.3 ForkJoin模式

“分而治之”是一种思想，所谓“分而治之”，就是把一个复杂的算法问题按一定的“分解”方法分为规模较小的若干部分，然后逐个解决，分别找出各部分的解，最后把各部分的解组成整个问题的解。“分而治之”思想在软件体系结构设计、模块化设计、基础算法中得到了非常广泛的应用。许多基础算法都运用了“分而治之”的思想，比如二分查找、快速排序等。













