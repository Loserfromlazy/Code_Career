# 限流算法和原理

在通信领域，限流被用来控制网络接口收发通信数据的速率，实现通信时的优化性能、较少延迟和提高带宽的功能。

互联网领域借鉴了此概念，用于控制高并发、大流量到来时对服务接口的请求的速率，比如618、双十一、抢票等活动。比如某个接口能抗住10000的QPS（每秒的响应请求数，也即是最大吞吐能力，QPS = 并发量 / 平均响应时间），但是突然来了15000个请求，那么这时候我们可以对请求进行限制，比如先放进来10000个请求，让其他请求阻塞一段时间，而不是直接返回404，让客户端等一会在重试，起到了流量削峰的作用。接口限流的算法主要有三种，分别是计数器限流、漏桶算法和令牌桶算法。

## 一、限流算法和简单实现

### 1.1 计数器限流

计数器限流原理十分简单，即在一个时间窗口内，所请求的最大数量是有限制的，对超出限制的部分请求不做处理。

计数器限流分为固定和滑动窗口两种：

1. 固定窗口就是每一分钟或者每一秒中的请求不能超过阈值，在下一秒钟计数器清零，重新进行计算

   ![image-20220701102338752](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220701102338752.png)

   但是固定窗口的计数器限流可能会存在流量突刺，比如在一分钟的第59秒有100个请求进来了，这时刚好没超过阈值，可以通过。但是第二分钟的第一秒又来了100个请求，因为count计数器清0了，所以这时又能通过，这样短时间内实际上通过了200个请求。

2. 滑动窗口就是细分后的计数器，他将每个时间窗口细分成若干个时间片段，每过一个时间段，整个时间窗口就会向右移动一格。

   ![image-20220701102839189](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220701102839189.png)

   如上图，时间窗口划分的越细比如5s、10s，限流效果就会越精细，滑动效果就会越平滑。

计数器限流简单实现：

```java
@Slf4j
public class CounterLimiting {

    //开始时间
    private static long startTime = System.currentTimeMillis();
    //限流时间，即固定时间窗口
    long maxTime = 1000;
    //允许通过的最大通过数
    long maxPass = 2;
    //当前通过的次数
    AtomicLong atomicLong = new AtomicLong(0);

    public long tryPass(long id, int turn) {
        long nowTime = System.currentTimeMillis();
        if (nowTime - startTime < maxTime ) {
            //没超过时间窗口，计数判断是否超过最大可通过次数
            long nowPass = atomicLong.incrementAndGet();
            if (nowPass < maxPass) {
                return nowPass;
            } else {
                //log.info("线程{},第{}轮，被拒绝", id, turn);
                return -nowPass;
            }
        } else {
            //超过时间窗口，计数器清零，默认通过
            synchronized (CounterLimiting.class) {
                //log.info("线程{},第{}轮，超过了时区，现在进行重置", id, turn);
                if (nowTime -startTime > maxTime) {
                    //加锁，进行二次判断，防止多次修改
                    startTime = nowTime;
                    atomicLong.set(0);
                }
            }
            return 0L;
        }
    }
    //线程数
    final int threadNum = 2;
    private final ExecutorService executors = Executors.newFixedThreadPool(threadNum);

    //测试用例
    @Test
    public void testLimit() throws InterruptedException {
        long start = System.currentTimeMillis();
        AtomicInteger integer = new AtomicInteger(0);

        CountDownLatch latch = new CountDownLatch(threadNum);
        for (int i = 0; i < threadNum; i++) {
            executors.submit(() -> {
                for (int j = 0; j < 20; j++) {
                    long l = tryPass(Thread.currentThread().getId(), j);
                    if (l < 0) {
                        integer.getAndIncrement();
                    }
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                latch.countDown();
            });
        }
        latch.await();
        final float time = (System.currentTimeMillis() - start) / 1000F;
        log.info("一共被限制了" + integer.get() + "次");
        log.info("比例为" + (float) integer.get() / (threadNum * 20));
        log.info("用时" + time + "秒");

    }
}
```

运行结果：

![image-20220704151528607](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220704151528607.png)

> 运行结果可能是32次或更少，如果电脑稍微卡一点可能会比32次少，因为加锁等操作也会耗费一点时间，但整体来说比例会在0.8左右。

### 1.2 漏桶限流

漏桶大小固定、处理速率固定，但请求进入的速度不固定。如果出现突发情况，可能会丢弃过多请求。

这个算法其实很形象，它就类似一个限制出水的水桶，通过一个固定大小的FIFO队列+定时队列元素的方式实现，请求进入队列后会被匀速的取出来，当队列被占满后新的请求就会直接拒绝，就像水从水桶中漏出来一样。这个算法优缺点也很明显，优点是请求匀速发给后端，不会造成流量突刺，缺点是请求会在队列中排队，响应时间会很长。

下面是简单的漏桶限流算法的实现（代码未完成）：

```java
@Slf4j
public class LeakyBucketLimiting {


    //开始时间
    private static long startTime = System.currentTimeMillis();
    //流动速率,每秒两滴水
    private int rate = 2;
    //漏桶剩余水量
    private long water;

    /**
     * @return 是否通过
     */
    public boolean tryPass(long id, int turn) {
        long nowTime = System.currentTimeMillis();
        //当前流出水量等于经过时间乘速率
        final long outWater = (nowTime - startTime) * rate / 1000;
        water = water - outWater;
        if (water < 0) {
            water = 0;
        }
        if (water <= 1) {
            startTime = nowTime;
            water++;
            return true;
        } else {
            return false;
        }
    }

    //线程数
    final int threadNum = 2;
    private final ExecutorService executors = Executors.newFixedThreadPool(threadNum);

    @Test
    public void testLimit() throws InterruptedException {
        long start = System.currentTimeMillis();
        AtomicInteger integer = new AtomicInteger(0);

        CountDownLatch latch = new CountDownLatch(threadNum);
        for (int i = 0; i < threadNum; i++) {
            executors.submit(() -> {
                for (int j = 0; j < 20; j++) {
                    boolean l = tryPass(Thread.currentThread().getId(), j);
                    if (l) {
                        integer.getAndIncrement();
                    }
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                latch.countDown();
            });
        }
        latch.await();
        final float time = (System.currentTimeMillis() - start) / 1000F;
        log.info("一共被限制了" + integer.get() + "次");
        log.info("比例为" + (float) integer.get() / (threadNum * 20));
        log.info("用时" + time + "秒");

    }
}
```

### 1.3 令牌桶限流

令牌桶的大小固定，令牌的产生速度固定，但是消耗令牌（请求）的速度不固定（可以应对某些时间请求过多的情况）。每个请求都会从令牌桶中取出令牌，如果没有令牌， 就丢弃这次请求。

令牌桶的优点是可以应对突发流量，当桶里有令牌时请求可以快速的响应，也不会产生漏桶队列中的等待时间, 缺点就是相对漏桶一定程度上减小了对下游服务的保护。

![image-20220701103914036](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220701103914036.png)

## 二、分布式计数器限流



## 三、Nginx漏桶限流



## 四、分布式令牌桶限流

