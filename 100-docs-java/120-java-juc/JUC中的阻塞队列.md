# JUC中的阻塞队列

## Queue接口

队列是一种先进先出（FIFO）的数据结构，java中用`Queue`接口来表示队列。

`Queue`接口中定义了6个方法：

```java
public interface Queue<E> extends Collection<E> {
    /**
     * Inserts the specified element into this queue if it is possible to do so
     * immediately without violating capacity restrictions, returning
     * {@code true} upon success and throwing an {@code IllegalStateException}
     * if no space is currently available.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Collection#add})
     * @throws IllegalStateException if the element cannot be added at this
     *         time due to capacity restrictions
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null and
     *         this queue does not permit null elements
     * @throws IllegalArgumentException if some property of this element
     *         prevents it from being added to this queue
     */
    boolean add(E e);

    /**
     * Inserts the specified element into this queue if it is possible to do
     * so immediately without violating capacity restrictions.
     * When using a capacity-restricted queue, this method is generally
     * preferable to {@link #add}, which can fail to insert an element only
     * by throwing an exception.
     *
     * @param e the element to add
     * @return {@code true} if the element was added to this queue, else
     *         {@code false}
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null and
     *         this queue does not permit null elements
     * @throws IllegalArgumentException if some property of this element
     *         prevents it from being added to this queue
     */
    boolean offer(E e);

    /**
     * Retrieves and removes the head of this queue.  This method differs
     * from {@link #poll poll} only in that it throws an exception if this
     * queue is empty.
     *
     * @return the head of this queue
     * @throws NoSuchElementException if this queue is empty
     */
    E remove();

    /**
     * Retrieves and removes the head of this queue,
     * or returns {@code null} if this queue is empty.
     *
     * @return the head of this queue, or {@code null} if this queue is empty
     */
    E poll();

    /**
     * Retrieves, but does not remove, the head of this queue.  This method
     * differs from {@link #peek peek} only in that it throws an exception
     * if this queue is empty.
     *
     * @return the head of this queue
     * @throws NoSuchElementException if this queue is empty
     */
    E element();

    /**
     * Retrieves, but does not remove, the head of this queue,
     * or returns {@code null} if this queue is empty.
     *
     * @return the head of this queue, or {@code null} if this queue is empty
     */
    E peek();
}
```

每个`Queue`方法都有两种形式：

（1）如果操作失败则抛出异常，

（2）如果操作失败，则返回特殊值（`null`或`false`，具体取决于操作），接口的常规结构如下表所示。

| 操作类型 | 抛出异常    | 返回特殊值 |
| :------- | :---------- | :--------- |
| 插入     | `add(e)`    | `offer(e)` |
| 移除     | `remove()`  | `poll()`   |
| 检查     | `element()` | `peek()`   |

`Queue`从`Collection`继承的`add`方法插入一个元素，除非它违反了队列的容量限制，在这种情况下它会抛出`IllegalStateException`；`offer`方法与`add`不同之处仅在于它通过返回`false`来表示插入元素失败。

`remove`和`poll`方法都移除并返回队列的头部，确切地移除哪个元素是由具体的实现来决定的，仅当队列为空时，`remove`和`poll`方法的行为才有所不同，在这些情况下，`remove`抛出`NoSuchElementException`，而`poll`返回`null`。

`element`和`peek`方法返回队列头部的元素，但不移除，它们之间的差异与`remove`和`poll`的方式完全相同，如果队列为空，则`element`抛出`NoSuchElementException`，而`peek`返回`null`。

队列一般不要插入空元素。



## BlockingQueue接口

`BlockingQueue`位于juc中，熟称阻塞队列， 阻塞队列首先它是一个队列，继承`Queue`接口，是队列就会遵循先进先出（FIFO）的原则，又因为它是阻塞的，故与普通的队列有两点区别：

1. 当一个线程向队列里面添加数据时，如果队列是满的，那么将阻塞该线程，暂停添加数据
2. 当一个线程从队列里面取出数据时，如果队列是空的，那么将阻塞该线程，暂停取出数据

`BlockingQueue`相关方法：

| 操作类型 | 抛出异常    | 返回特殊值 | 一直阻塞 | 超时退出               |
| :------- | :---------- | :--------- | -------- | ---------------------- |
| 插入     | `add(e)`    | `offer(e)` | put(e)   | offer(e,timeuout,unit) |
| 移除     | `remove()`  | `poll()`   | take()   | poll(timeout,unit)     |
| 检查     | `element()` | `peek()`   | 不支持   | 不支持                 |

**重点，再来解释一下，加深印象：**

- 3个可能会有异常的方法，add、remove、element；这3个方法不会阻塞（是说队列满或者空的情况下是否会阻塞）；队列满的情况下，add抛出异常；队列为空情况下，remove、element抛出异常
- offer、poll、peek 也不会阻塞（是说队列满或者空的情况下是否会阻塞）；队列满的情况下，offer返回false；队列为空的情况下，pool、peek返回null
- 队列满的情况下，调用put方法会导致当前线程阻塞
- 队列为空的情况下，调用take方法会导致当前线程阻塞
- `offer(e,timeuout,unit)`，超时之前，插入成功返回true，否者返回false
- `poll(timeout,unit)`，超时之前，获取到头部元素并将其移除，返回true，否者返回false



## BlockingQueue常见的实现类

![LinkedTransferQueue](./assets/LinkedTransferQueue-1731943152754-3.png)

- ArrayBlockingQueue

> 基于数组的阻塞队列实现，其内部维护一个定长的数组，用于存储队列元素。线程阻塞的实现是通过ReentrantLock来完成的，数据的插入与取出共用同一个锁，因此ArrayBlockingQueue并不能实现生产、消费同时进行。而且在创建ArrayBlockingQueue时，我们还可以控制对象的内部锁是否采用公平锁，默认采用非公平锁。

- LinkedBlockingQueue

> 基于单向链表的阻塞队列实现，在初始化LinkedBlockingQueue的时候可以指定大小，也可以不指定，默认类似一个无限大小的容量（Integer.MAX_VALUE），不指队列容量大小也是会有风险的，一旦数据生产速度大于消费速度，系统内存将有可能被消耗殆尽，因此要谨慎操作。另外LinkedBlockingQueue中用于阻塞生产者、消费者的锁是两个（锁分离），因此生产与消费是可以同时进行的。

- PriorityBlockingQueue

> 一个支持优先级排序的无界阻塞队列，进入队列的元素会按照优先级进行排序

- SynchronousQueue

> 同步阻塞队列，SynchronousQueue没有容量，与其他BlockingQueue不同，SynchronousQueue是一个不存储元素的BlockingQueue，每一个put操作必须要等待一个take操作，否则不能继续添加元素，反之亦然

- DelayQueue

> DelayQueue是一个支持延时获取元素的无界阻塞队列，里面的元素全部都是“可延期”的元素，列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行，也就是说只有在延迟期到时才能够从队列中取元素

- LinkedTransferQueue

> LinkedTransferQueue是基于链表的FIFO无界阻塞队列，它出现在JDK7中，Doug Lea 大神说LinkedTransferQueue是一个聪明的队列，它是ConcurrentLinkedQueue、SynchronousQueue(公平模式下)、无界的LinkedBlockingQueues等的超集，`LinkedTransferQueue`包含了`ConcurrentLinkedQueue、SynchronousQueue、LinkedBlockingQueues`三种队列的功能

## ArrayBlockingQueue

有界阻塞队列，内部使用数组存储元素，有2个常用构造方法：

```java
//capacity表示容量大小，默认内部采用非公平锁
public ArrayBlockingQueue(int capacity)//capacity：容量大小，fair：内部是否是使用公平锁
public ArrayBlockingQueue(int capacity, boolean fair)
```

**需求：**业务系统中有很多地方需要推送通知，由于需要推送的数据太多，我们将需要推送的信息先丢到阻塞队列中，然后开一个线程进行处理真实发送，代码如下：

```java
package com.itsoku.chat25;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import sun.text.normalizer.NormalizerBase;
import java.util.Calendar;import java.util.concurrent.*;
/** * 跟着阿里p7学并发，微信公众号：javacode2018 */
public class Demo1 {    //推送队列    
    static ArrayBlockingQueue<String> pushQueue = new ArrayBlockingQueue<String>(10000);    
    static {        
        //启动一个线程做真实推送        
        new Thread(() -> {            
            while (true) {                
                String msg;                
                try {                    
                    long starTime = System.currentTimeMillis();                    
                    //获取一条推送消息，此方法会进行阻塞，直到返回结果                    
                    msg = pushQueue.take();                    
                    long endTime = System.currentTimeMillis();                    
                    //模拟推送耗时                    
                    TimeUnit.MILLISECONDS.sleep(500);                   
                    System.out.println(String.format("[%s,%s,take耗时:%s],%s,发送消息:%s", starTime, endTime, (endTime - starTime), Thread.currentThread().getName(), msg));                
                } catch (InterruptedException e) {                    
                    e.printStackTrace();                
                }            
            }        
        }).start();    
    }    
    //推送消息，需要发送推送消息的调用该方法，会将推送信息先加入推送队列    
    public static void pushMsg(String msg) throws InterruptedException {        
        pushQueue.put(msg);    
    }    
    public static void main(String[] args) throws InterruptedException {        
        for (int i = 1; i <= 5; i++) {            
            String msg = "一起来学java高并发,第" + i + "天";            
            //模拟耗时            
            TimeUnit.SECONDS.sleep(i);            
            Demo1.pushMsg(msg);        
        }    
    }
}
```

输出：

```
[1565595629206,1565595630207,take耗时:1001],Thread-0,发送消息:一起来学java高并发,第1天
[1565595630208,1565595632208,take耗时:2000],Thread-0,发送消息:一起来学java高并发,第2天
[1565595632208,1565595635208,take耗时:3000],Thread-0,发送消息:一起来学java高并发,第3天
[1565595635208,1565595639209,take耗时:4001],Thread-0,发送消息:一起来学java高并发,第4天
[1565595639209,1565595644209,take耗时:5000],Thread-0,发送消息:一起来学java高并发,第5天
```

代码中我们使用了有界队列`ArrayBlockingQueue`，创建`ArrayBlockingQueue`时候需要制定容量大小，调用`pushQueue.put`将推送信息放入队列中，如果队列已满，此方法会阻塞。代码中在静态块中启动了一个线程，调用`pushQueue.take();`从队列中获取待推送的信息进行推送处理。

**注意：**`ArrayBlockingQueue`如果队列容量设置的太小，消费者发送的太快，消费者消费的太慢的情况下，会导致队列空间满，调用put方法会导致发送者线程阻塞，所以注意设置合理的大小，协调好消费者的速度。

## LinkedBlockingQueue

内部使用单向链表实现的阻塞队列，3个构造方法：

```java
//默认构造方法，容量大小为Integer.MAX_VALUE
public LinkedBlockingQueue();
//创建指定容量大小的LinkedBlockingQueue
public LinkedBlockingQueue(int capacity);
//容量为Integer.MAX_VALUE,并将传入的集合丢入队列中
public LinkedBlockingQueue(Collection<? extends E> c);
```

`LinkedBlockingQueue`的用法和`ArrayBlockingQueue`类似，建议使用的时候指定容量，如果不指定容量，插入的太快，移除的太慢，可能会产生OOM。

## PriorityBlockingQueue

**无界的优先级**阻塞队列，内部使用数组存储数据，达到容量时，会自动进行扩容，放入的元素会按照优先级进行排序，4个构造方法：

```java
//默认构造方法，默认初始化容量是11
public PriorityBlockingQueue();
//指定队列的初始化容量
public PriorityBlockingQueue(int initialCapacity);
//指定队列的初始化容量和放入元素的比较器
public PriorityBlockingQueue(int initialCapacity,Comparator<? super E> comparator);
//传入集合放入来初始化队列，传入的集合可以实现SortedSet接口或者PriorityQueue接口进行排序，如果没有实现这2个接口，按正常顺序放入队列
public PriorityBlockingQueue(Collection<? extends E> c);
```

优先级队列放入元素的时候，会进行排序，所以我们需要指定排序规则，有2种方式：

1. 创建`PriorityBlockingQueue`指定比较器`Comparator`
2. 放入的元素需要实现`Comparable`接口

上面2种方式必须选一个，如果2个都有，则走第一个规则排序。

**需求：**还是上面的推送业务，目前推送是按照放入的先后顺序进行发送的，比如有些公告比较紧急，优先级比较高，需要快点发送，怎么搞？此时`PriorityBlockingQueue`就派上用场了，代码如下：

```java
package com.itsoku.chat25;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.TimeUnit;
/**
 * 跟着阿里p7学并发，微信公众号：javacode2018
 */
public class Demo2 {
    //推送信息封装
    static class Msg implements Comparable<Msg> {
        //优先级，越小优先级越高
        private int priority;
        //推送的信息
        private String msg;
        public Msg(int priority, String msg) {
            this.priority = priority;
            this.msg = msg;
        }
        @Override
        public int compareTo(Msg o) {
            return Integer.compare(this.priority, o.priority);
        }
        @Override
        public String toString() {
            return "Msg{" +
                "priority=" + priority +
                ", msg='" + msg + '\'' +
                '}';
        }
    }
    //推送队列
    static PriorityBlockingQueue<Msg> pushQueue = new PriorityBlockingQueue<Msg>();
    static {
        //启动一个线程做真实推送
        new Thread(() -> {
            while (true) {
                Msg msg;
                try {
                    long starTime = System.currentTimeMillis();
                    //获取一条推送消息，此方法会进行阻塞，直到返回结果
                    msg = pushQueue.take();
                    //模拟推送耗时
                    TimeUnit.MILLISECONDS.sleep(100);
                    long endTime = System.currentTimeMillis();
                    System.out.println(String.format("[%s,%s,take耗时:%s],%s,发送消息:%s", starTime, endTime, (endTime - starTime), Thread.currentThread().getName(), msg));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
    //推送消息，需要发送推送消息的调用该方法，会将推送信息先加入推送队列
    public static void pushMsg(int priority, String msg) throws InterruptedException {
        pushQueue.put(new Msg(priority, msg));
    }
    public static void main(String[] args) throws InterruptedException {
        for (int i = 5; i >= 1; i--) {
            String msg = "一起来学java高并发,第" + i + "天";
            Demo2.pushMsg(i, msg);
        }
    }
}
```

输出：

```
[1565598857028,1565598857129,take耗时:101],Thread-0,发送消息:Msg{priority=1, msg='一起来学java高并发,第1天'}[1565598857162,1565598857263,take耗时:101],Thread-0,发送消息:Msg{priority=2, msg='一起来学java高并发,第2天'}[1565598857263,1565598857363,take耗时:100],Thread-0,发送消息:Msg{priority=3, msg='一起来学java高并发,第3天'}[1565598857363,1565598857463,take耗时:100],Thread-0,发送消息:Msg{priority=4, msg='一起来学java高并发,第4天'}[1565598857463,1565598857563,take耗时:100],Thread-0,发送消息:Msg{priority=5, msg='一起来学java高并发,第5天'}
```

main中放入了5条推送信息，i作为消息的优先级按倒叙放入的，最终输出结果中按照优先级由小到大输出。注意Msg实现了`Comparable`接口，具有了比较功能。

## SynchronousQueue

> 同步阻塞队列，SynchronousQueue没有容量，与其他BlockingQueue不同，SynchronousQueue是一个不存储元素的BlockingQueue，每一个put操作必须要等待一个take操作，否则不能继续添加元素，反之亦然。SynchronousQueue 在现实中用的不多，线程池中有用到过，`Executors.newCachedThreadPool()`实现中用到了这个队列，当有任务丢入线程池的时候，如果已创建的工作线程都在忙于处理任务，则会新建一个线程来处理丢入队列的任务。

来个示例代码：

```java
package com.itsoku.chat25;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;
/**
 * 跟着阿里p7学并发，微信公众号：javacode2018
 */
public class Demo3 {
    static SynchronousQueue<String> queue = new SynchronousQueue<>();
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            try {
                long starTime = System.currentTimeMillis();
                queue.put("java高并发系列，路人甲Java!");
                long endTime = System.currentTimeMillis();
                System.out.println(String.format("[%s,%s,take耗时:%s],%s", starTime, endTime, (endTime - starTime), Thread.currentThread().getName()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        //休眠5秒之后，从队列中take一个元素
        TimeUnit.SECONDS.sleep(5);
        System.out.println(System.currentTimeMillis() + "调用take获取并移除元素," + queue.take());
    }
}
```

输出：

```
1565600421645调用take获取并移除元素,java高并发系列，路人甲Java![1565600416645,1565600421645,take耗时:5000],Thread-0
```

main方法中启动了一个线程，调用`queue.put`方法向队列中丢入一条数据，调用的时候产生了阻塞，从输出结果中可以看出，直到take方法被调用时，put方法才从阻塞状态恢复正常。

## DelayQueue

> DelayQueue是一个支持延时获取元素的无界阻塞队列，里面的元素全部都是“可延期”的元素，列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行，也就是说只有在延迟期到时才能够从队列中取元素。

**需求：**还是推送的业务，有时候我们希望早上9点或者其他指定的时间进行推送，如何实现呢？此时`DelayQueue`就派上用场了。

我们先看一下`DelayQueue`类的声明：

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E>
```

元素E需要实现接口`Delayed`，我们看一下这个接口的代码：

```java
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

`Delayed`继承了`Comparable`接口，这个接口是用来做比较用的，`DelayQueue`内部使用`PriorityQueue`来存储数据的，`PriorityQueue`是一个优先级队列，丢入的数据会进行排序，排序方法调用的是`Comparable`接口中的方法。下面主要说一下`Delayed`接口中的`getDelay`方法：此方法在给定的时间单位内返回与此对象关联的剩余延迟时间。

**对推送我们再做一下处理，让其支持定时发送（定时在将来某个时间也可以说是延迟发送），代码如下：**

```java
package com.itsoku.chat25;
import java.util.Calendar;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.TimeUnit;
/**
 * 跟着阿里p7学并发，微信公众号：javacode2018
 */
public class Demo4 {
    //推送信息封装
    static class Msg implements Delayed {
        //优先级，越小优先级越高
        private int priority;
        //推送的信息
        private String msg;
        //定时发送时间，毫秒格式
        private long sendTimeMs;
        public Msg(int priority, String msg, long sendTimeMs) {
            this.priority = priority;
            this.msg = msg;
            this.sendTimeMs = sendTimeMs;
        }
        @Override
        public String toString() {
            return "Msg{" +
                    "priority=" + priority +
                    ", msg='" + msg + '\'' +
                    ", sendTimeMs=" + sendTimeMs +
                    '}';
        }
        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(this.sendTimeMs - Calendar.getInstance().getTimeInMillis(), TimeUnit.MILLISECONDS);
        }
        @Override
        public int compareTo(Delayed o) {
            if (o instanceof Msg) {
                Msg c2 = (Msg) o;
                return Integer.compare(this.priority, c2.priority);
            }
            return 0;
        }
    }
    //推送队列
    static DelayQueue<Msg> pushQueue = new DelayQueue<Msg>();
    static {
        //启动一个线程做真实推送
        new Thread(() -> {
            while (true) {
                Msg msg;
                try {
                    //获取一条推送消息，此方法会进行阻塞，直到返回结果
                    msg = pushQueue.take();
                    //此处可以做真实推送
                    long endTime = System.currentTimeMillis();
                    System.out.println(String.format("定时发送时间：%s,实际发送时间：%s,发送消息:%s", msg.sendTimeMs, endTime, msg));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
    //推送消息，需要发送推送消息的调用该方法，会将推送信息先加入推送队列
    public static void pushMsg(int priority, String msg, long sendTimeMs) throws InterruptedException {
        pushQueue.put(new Msg(priority, msg, sendTimeMs));
    }
    public static void main(String[] args) throws InterruptedException {
        for (int i = 5; i >= 1; i--) {
            String msg = "一起来学java高并发,第" + i + "天";
            Demo4.pushMsg(i, msg, Calendar.getInstance().getTimeInMillis() + i * 2000);
        }
    }
}
```

输出：

```
定时发送时间：1565603357198,实际发送时间：1565603357198,发送消息:Msg{priority=1, msg='一起来学java高并发,第1天', sendTimeMs=1565603357198}
定时发送时间：1565603359198,实际发送时间：1565603359198,发送消息:Msg{priority=2, msg='一起来学java高并发,第2天', sendTimeMs=1565603359198}
定时发送时间：1565603361198,实际发送时间：1565603361199,发送消息:Msg{priority=3, msg='一起来学java高并发,第3天', sendTimeMs=1565603361198}
定时发送时间：1565603363198,实际发送时间：1565603363199,发送消息:Msg{priority=4, msg='一起来学java高并发,第4天', sendTimeMs=1565603363198}
定时发送时间：1565603365182,实际发送时间：1565603365183,发送消息:Msg{priority=5, msg='一起来学java高并发,第5天', sendTimeMs=1565603365182}
```

可以看出时间发送时间，和定时发送时间基本一致，代码中`Msg`需要实现`Delayed接口`，重点在于`getDelay`方法，这个方法返回剩余的延迟时间，代码中使用`this.sendTimeMs`减去当前时间的毫秒格式时间，得到剩余延迟时间。

## LinkedTransferQueue

> LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。

LinkedTransferQueue类继承自AbstractQueue抽象类，并且实现了TransferQueue接口：

```java
public interface TransferQueue<E> extends BlockingQueue<E> {
    // 如果存在一个消费者已经等待接收它，则立即传送指定的元素，否则返回false，并且不进入队列。
    boolean tryTransfer(E e);
    // 如果存在一个消费者已经等待接收它，则立即传送指定的元素，否则等待直到元素被消费者接收。
    void transfer(E e) throws InterruptedException;
    // 在上述方法的基础上设置超时时间
    boolean tryTransfer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
    // 如果至少有一位消费者在等待，则返回true
    boolean hasWaitingConsumer();
    // 获取所有等待获取元素的消费线程数量
    int getWaitingConsumerCount();
}
```

再看一下上面的这些方法，`transfer(E e)`方法和`SynchronousQueue的put方法`类似，都需要等待消费者取走元素，否者一直等待。其他方法和`ArrayBlockingQueue、LinkedBlockingQueue`中的方法类似。

## 总结

1. 重点需要了解`BlockingQueue`中的所有方法，以及他们的区别
2. 重点掌握`ArrayBlockingQueue`、`LinkedBlockingQueue`、`PriorityBlockingQueue`、`DelayQueue`的使用场景
3. 需要处理的任务有优先级的，使用`PriorityBlockingQueue`
4. 处理的任务需要延时处理的，使用`DelayQueue`