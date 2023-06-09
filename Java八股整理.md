<div align="center">
    <h1>
        ☕Java八股整理
    </h1>
</div>


## 强引用和弱引用

### 强引用

强引用就是我们经常使用的引用，其写法如下

```java
StringBuffer buffer = new StringBuffer();
```

上面创建了一个`StringBuffer`对象，并将这个对象的（强）引用存到变量buffer中。是的，就是这个小儿科的操作（请原谅我这样的说 法）。强引用最重要的就是它能够让引用变得强（Strong），这就决定了它和垃圾回收器的交互。具体来说，如果一个对象通过一串强引用链接可到达 (Strongly reachable)，**它是不会被回收的**。如果你不想让你正在使用的对象被回收，这就正是你所需要的。

在 Java 中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，即使该对象以后永远都不会被用到JVM 也不会回收。因此强引用是造成Java 内存泄漏的主要原因之一。



### 弱引用

弱引用需要用`WeakReference`类来实现，它比软引用的生存期更短，对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管 JVM 的内存空间是否足够，总会回收该对象占用的内存。

弱引用简单来说就是将对象留在内存的能力不是那么强的引用。使用`WeakReference`，垃圾回收器会帮你来决定引用的对象何时回收并且将对象从内存移除。创建弱引用如下

```java
WeakReference<Widget> weakWidget = new WeakReference<Widget>(widget);
```

使用`weakWidget.get()`就可以得到真实的Widget对象，因为弱引用不能阻挡垃圾回收器对其回收，你会发现（当没有任何强引用到widget对象时）使用get时突然返回null。

解决上述的widget序列数记录的问题，最简单的办法就是使用Java内置的`WeakHashMap`类。`WeakHashMap`和`HashMap` 几乎一样，唯一的区别就是它的键（不是值!!!）使用`WeakReference`引用。当`WeakHashMap`的键标记为垃圾的时候，这个键对应的条目 就会自动被移除。这就避免了上面不需要的Widget对象手动删除的问题。使用`WeakHashMap`可以很便捷地转为`HashMap`或者`Map`。

虚引用需要`PhantomReference` 类来实现，它不能单独使用，必须和引用队列联合使用。虚引用的主要作用是跟踪对象被垃圾回收的状态。



## `String`, `StringBuffer`与`StringBuilder`

![三者关系](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20200226214858667.png)

### String效率低的原因

在以类似于以下的方式进行字符串拼接时

```java
String str="abc";
System.out.println(str);
str=str+"de";
System.out.println(str);
```

JVM的处理方式为：

首先创建一个String对象str，并把“abc”赋值给str，然后在第三行中，其实JVM又创建了一个新的对象也名为str，然后再把原来的str的值和“de”加起来再赋值给新的str，而原来的str就会被JVM的垃圾回收机制（GC）给回收掉了。

由于拼接过程中的**GC**，导致直接使用String进行字符串拼接的效率比较低。



### StringBuffer线程安全的原因

StringBuffer中很多方法可以带有`synchronized`关键字，所以可以保证线程是安全的

+ StringBuffer-append

  ```java
  @Override
  public synchronized StringBuffer append(String str) {
      toStringCache = null;
      super.append(str);
      return this;
  }
  ```

+ StringBuffer-toString

  ```java
  private transient char[] toStringCache;
  
  @Override
  public synchronized String toString() {
      if (toStringCache == null) {
          // 涉及到字符串长度确认
          toStringCache = Arrays.copyOfRange(value, 0, count);
      }
      return new String(toStringCache, true);
  }
  ```

+ StringBuilder-toString

  ```java
  @Override
  public String toString() {
      // Create a copy, don't share the array
      return new String(value, 0, count);
  }
  ```

  

## `Thread`类与多线程

### 线程的六种状态

java中的线程有六种状态

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230319144606796.png" alt="image-20230319144606796" style="zoom:67%;" />

调用start()后,一个Java线程才由操作系统交由CPU，在CPU上生成一个实际的线程

操作系统中的线程有五种状态

<img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230319154100499.png" alt="image-20230319154100499" style="zoom:67%;" />



### 线程暂停的方法

sleep、yield、join、stop、interrupt

Sleep，会导致**正在执行的线程**进入**阻塞状态TIME_WAITING**

yield：会导致**正在执行的线程**进入**就绪状态RUNNABLE**，**当前正在执行的线程**暂停一次，允许其他线程执行，不阻塞，线程进入就绪状态，如果没有其他等待执行的线程，这个时候当前线程就会马上恢复执行。

Join，会导致**调用它的线程**进入**阻塞状态TIME_WAITING**，而**调用此方法的线程**强制执行，也就是变成**运行状态**

stop：强迫线程停止执行，**已过时**。不推荐使用。

interrupt：停止一个线程执行，但允许该线程料理一下自己的后事。



### sleep和wait的区别

对于sleep()方法，我们首先要知道该方法是属于**Thread类**中的。而wait()方法，则是属于**Object类**中的。

sleep()方法导致了程序暂停执行指定的时间，让出CPU给其他线程，但是他的监控状态依然保持者，当指定的时间到了又会自动恢复运行状态。

在调用sleep()方法的过程中，线程**不会释放对象锁**。

而当调用wait()方法的时候，线程**会放弃对象锁**，进入等待此对象的等待锁定池，只有针对此对象**调用notify()**方法后本线程才进入对象锁定池准备

从使用角度看，sleep是Thread线程类的方法，而wait是Object顶级类的方法。

sleep可以在任何地方使用，而wait只能在同步方法或者同步块中使用。

![Sleep VS Wait](https://cdn.jsdelivr.net/gh/WangMinan/Pics/2019050816141738.png)



### Lock和Synchronized的区别

#### Synchronized关键字

当一个线程访问某个对象的synchronized方法或synchronized代码块时，其他线程仍然可以访问该对象的非同步代码块。但是，当一个线程访问某个对象的synchronized方法或synchronized代码块时，**其他线程将被阻塞**，直到该线程执行完该方法或该代码块 。这是因为synchronized关键字可以保证**在同一个时刻，只有一个线程可以执行某个方法或某个代码块**(主要是对方法或者代码块中存在共享数据的操作)，同时synchronized还可以保证一个线程的变化(主要是共享数据的变化)被其他线程所看到，保证可见性。(Volatile也会干这个)

Sychronized是基于对象实现的。一个对象在JVM中的存储情况如下

```mermaid
graph LR
	new_Object --> 对象头
	new_Object --> 实例数据
	new_Object --> 对象填充
	对象头 --> MarkWord
	对象头 --> ClassPoint
	MarkWord --> 包含四种锁:无锁001,偏向锁101,轻量级锁00,重量级锁10
```





#### Lock与Sychronized的区别

**要求**

* 掌握 lock 与 synchronized 的区别
* 理解 ReentrantLock 的公平、非公平锁
* 理解 ReentrantLock 中的条件变量

**三个层面**

不同点

* 语法层面
  * synchronized 是关键字，源码在 jvm 中，用 c++ 语言实现
  * Lock 是接口，源码由 jdk 提供，用 java 语言实现
  * 使用 synchronized 时，退出同步代码块锁会自动释放，而使用 Lock 时，需要手动调用 unlock 方法释放锁
* 功能层面
  * 二者均属于悲观锁、都具备基本的互斥、同步、锁重入功能
  * Lock 提供了许多 synchronized 不具备的功能，例如获取等待状态、公平锁、可打断、可超时、多条件变量
  * Lock 有适合不同场景的实现，如 ReentrantLock， ReentrantReadWriteLock
* 性能层面
  * 在没有竞争时，synchronized 做了很多优化，如偏向锁、轻量级锁，性能不赖
  * 在竞争激烈时，Lock 的实现通常会提供更好的性能

**公平锁**

* 公平锁的公平体现
  * **已经处在阻塞队列**中的线程（不考虑超时）始终都是公平的，先进先出
  * 公平锁是指**未处于阻塞队列**中的线程来争抢锁，如果队列不为空，则老实到队尾等待
  * 非公平锁是指**未处于阻塞队列**中的线程来争抢锁，与队列头唤醒的线程去竞争，谁抢到算谁的
* 公平锁会降低吞吐量，一般不用

**条件变量**

* ReentrantLock 中的条件变量功能类似于普通 synchronized 的 wait，notify，用在当线程获得锁后，发现条件不满足时，临时等待的链表结构
* 与 synchronized 的等待集合不同之处在于，ReentrantLock 中的条件变量可以有多个，可以实现更精细的等待、唤醒控制

#### 可重入锁

可重入就是说某个线程已经获得某个锁，可以再次获取锁而不会出现死锁。

可重入锁有

- synchronized 无须手动释放
- ReentrantLock 需要手动释放 加锁次数和释放次数要一致 否则死锁

```java
// 演示可重入锁是什么意思，可重入，就是可以重复获取相同的锁，synchronized和ReentrantLock都是可重入的
// 可重入降低了编程复杂性
public class WhatReentrant {
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				synchronized (this) {
					System.out.println("第1次获取锁，这个锁是：" + this);
					int index = 1;
					while (true) {
						synchronized (this) {
							System.out.println("第" + (++index) + "次获取锁，这个锁是：" + this);
						}
						if (index == 10) {
							break;
						}
					}
				}
			}
		}).start();
	}
}
```

实现原理

https://blog.csdn.net/txk396879586/article/details/122428293



### Volatile关键字

"Java语言提供了一种**稍弱的同步机制**，即volatile变量，用来确保将变量的更新操作通知到其他线程"。

这句话说明了两点：①volatile变量**是一种同步机制**；②volatile能够**确保可见性**。->Volatile不能保证线程安全

Java对定义了8种操作内存的方式

lock:(锁定)，unlock(解锁)，read(读取)，load(载入)，use(试用)， assign(赋值)，store(存储)，write(写入)。

volatile 对这八种操作有着两个特殊的限定

+ use动作之前必须要有read和load动作， 这三个动作必须是连续出现的。【表示：每次工作内存要使用volatile变量之前必须去主存中**拿取最新的volatile变量**】
+ assign动作之后必须跟着store和write动作，这三个动作必须是连续出现的。【表示: 每次工作内存改变了volatile变量的值，就必须把该值**写回到主存中**】

**要求**

* 掌握线程安全要考虑的三个问题
* 掌握 volatile 能解决哪些问题

**原子性**

* 起因：多线程下，不同线程的**指令发生了交错**导致的共享变量的读写混乱
* 解决：用悲观锁或乐观锁解决，volatile 并不能解决原子性

**可见性**

* 起因：由于**编译器优化(JIT)、或缓存优化、或 CPU 指令重排序优化**导致的对共享变量所做的修改另外的线程看不到
* 解决：用 volatile 修饰共享变量，能够**防止编译器等优化发生**，让一个线程对共享变量的修改对另一个线程可见

**有序性**

* 起因：由于**编译器优化、或缓存优化、或 CPU 指令重排序优化**导致指令的实际执行顺序与编写顺序不一致

* 解决：用 volatile 修饰共享变量会在读、写共享变量时加入不同的屏障，阻止其他读写操作越过屏障，从而达到阻止重排序的效果

* 注意：
  * **volatile 变量写**加的屏障是阻止上方其它写操作越过屏障排到 **volatile 变量写**之下(写入时volatile变量应当最后写入)
  
  * **volatile 变量读**加的屏障是阻止下方其它读操作越过屏障排到 **volatile 变量读**之上(要读时volatile变量应当最先读取)
  
  * volatile 读写加入的屏障只能防止同一线程内的指令重排
  
    <img src="https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230321152249734.png" alt="image-20230321152249734" style="zoom:67%;" />





### ThreadLocal

ThreadLocal 顾名思义“线程本地变量”，对应到 Java 代码就是线程私有变量，可以把它理解为对于同一个变量，在不同的线程包含不同的副本，并且各个副本之间相互独立

```java
public class UserHolder {
    private static final ThreadLocal<UserDTO> tl = new ThreadLocal<>();

    public static void saveUser(UserDTO user){
        tl.set(user);
    }

    public static UserDTO getUser(){
        return tl.get();
    }

    public static void removeUser(){
        tl.remove();
    }
}
```

ThreadLocal 的使用主要基于 **get()、set() 方法**.这里首先获取当前线程对象，根据线程对象获取 ThreadLocalMap 对象，之后以 this 为 key，value 为值，存入 map。这里 this 指代当前 ThreadLocal 对象

```java
public void set(T value) {
	// 获取调用 set() 方法的线程
    Thread t = Thread.currentThread();
    // 获取一个 map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue()
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

ThreadLocalMap默认的初始capacity为16，factor为2/3.每次扩容capacity翻倍。使用开放寻址解决索引冲突。

 **开放寻址法**：

又称开放定址法，当哈希碰撞发生时，从发生碰撞的那个单元起，按照一定的次序，从哈希表中寻找一个空闲的单元，然后把发生冲突的元素存入到该单元。这个空闲单元又称为开放单元或者空白单元。

查找时，如果探查到空白单元，即表中无待查的关键字，则查找失败。开放寻址法需要的表长度要大于等于所需要存放的元素数量，非常适用于装载因子较小（小于0.5）的散列表。

开放定址法的缺点在于删除元素的时候不能真的删除，否则会引起查找错误，只能做一个特殊标记，直到有下个元素插入才能真正删除该元素。

#### **弱引用 key**

ThreadLocalMap 中的 key 被设计为弱引用，原因如下

* Thread 可能需要长时间运行（如线程池中的线程），如果 key 不再使用，需要在内存不足（GC）时释放其占用的内存

#### **内存释放时机**

* 被动 GC 释放 key
  * 仅是让 key 的内存释放，关联 value 的内存并不会释放
* 懒惰被动释放 value
  * get key 时，发现是 null key，则释放其 value 内存
  * set key 时，会使用启发式扫描，清除临近的 null key 的 value 内存，启发次数与元素个数，是否发现 null key 有关
* 主动 remove 释放 key，value——一般建议手动Remove
  * 会同时释放 key，value 的内存，也会清除临近的 null key 的 value 内存
  * 推荐使用它，因为一般使用 ThreadLocal 时都把它作为静态变量（即强引用），因此无法被动依靠 GC 回收



### 线程池

线程池的优点

+ 降低系统资源消耗，通过**重用已存在的线程**，降低线程创建和销毁造成的消耗；
+ 提高系统响应速度，当有任务到达时，无需等待新线程的创建便能立即执行；
+ 方便线程**并发数的管控**，线程若是无限制的创建，不仅会额外消耗大量系统资源，更是占用过多资源而阻塞系统或oom等状况，从而降低系统的稳定性。线程池能有效管控线程，统一分配、调优，提供资源使用率；
+ 更强大的功能，线程池提供了**定时**、定期以及可控线程数等功能的线程池，使用方便简单。

#### Java中的**四种**线程池

+ **newCachedThreadPool**：
  创建一个**可缓存的无界线程池**，如果线程池长度超过处理需要，可灵活回收空线程，若无可回收，则新建线程。当线程池中的线程空闲时间超过60s，则会自动回收该线程，当任务超过线程池的线程数则创建新的线程，线程池的大小上限为**Integer.MAX_VALUE**,可看作无限大。

  ```java
  /**
   * 可缓存无界线程池测试
   * 当线程池中的线程空闲时间超过60s则会自动回收该线程，核心线程数为0
   * 当任务超过线程池的线程数则创建新线程。线程池的大小上限为Integer.MAX_VALUE，
   * 可看做是无限大。
   */
  @Test
  public void cacheThreadPoolTest() {
      // 创建可缓存的无界线程池，可以指定线程工厂，也可以不指定线程工厂
      ExecutorService executorService = Executors.newCachedThreadPool(new testThreadPoolFactory("cachedThread"));
      for (int i = 0; i < 10; i++) {
          executorService.submit(() -> {
              print("cachedThreadPool");
              System.out.println(Thread.currentThread().getName());
          }
          );
      }
  }
  ```

+ **newFixedThreadPool**：

  创建一个指定大小的线程池，可控制线程的最大并发数，超出的线程会在LinkedBlockingQueue阻塞队列中等待

  一般指定线程池大小为CPU线程整数倍

+ **newScheduledThreadPool：**

  创建一个定长的线程池，可以指定线程池核心线程数，支持定时及周期性任务的执行

  ```java
  /**
   * 创建定时周期执行的线程池测试
   *
   * schedule(Runnable command, long delay, TimeUnit unit)，延迟一定时间后执行Runnable任务；
   * schedule(Callable callable, long delay, TimeUnit unit)，延迟一定时间后执行Callable任务；
   * scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)，延迟一定时间后，以间隔period时间的频率周期性地执行任务；
   * scheduleWithFixedDelay(Runnable command, long initialDelay, long delay,TimeUnit unit)，与scheduleAtFixedRate()方法很类似，
   * 但是不同的是scheduleWithFixedDelay()方法的周期时间间隔是以上一个任务执行结束到下一个任务开始执行的间隔，而scheduleAtFixedRate()方法的周期时间间隔是以上一个任务开始执行到下一个任务开始执行的间隔，
   * 也就是这一些任务系列的触发时间都是可预知的。
   * ScheduledExecutorService功能强大，对于定时执行的任务，建议多采用该方法。
   *
   * 作者：张老梦
   * 链接：https://www.jianshu.com/p/9ce35af9100e
   * 来源：简书
   * 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
   */
  @Test
  public void scheduleThreadPoolTest() {
      // 创建指定核心线程数，但最大线程数是Integer.MAX_VALUE的可定时执行或周期执行任务的线程池
      ScheduledExecutorService executorService = Executors.newScheduledThreadPool(5, new testThreadPoolFactory("scheduledThread"));
  
      // 定时执行一次的任务，延迟1s后执行
      executorService.schedule(new Runnable() {
          @Override
          public void run() {
              print("scheduleThreadPool");
              System.out.println(Thread.currentThread().getName() + ", delay 1s");
          }
      }, 1, TimeUnit.SECONDS);
  
  
      // 周期性地执行任务，延迟2s后，每3s一次地周期性执行任务
      executorService.scheduleAtFixedRate(new Runnable() {
          @Override
          public void run() {
              System.out.println(Thread.currentThread().getName() + ", every 3s");
          }
      }, 2, 3, TimeUnit.SECONDS);
  
  
      executorService.scheduleWithFixedDelay(new Runnable() {
          @Override
          public void run() {
              long start = new Date().getTime();
              System.out.println("scheduleWithFixedDelay 开始执行时间:" +
                      DateFormat.getTimeInstance().format(new Date()));
              try {
                  Thread.sleep(5000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
              long end = new Date().getTime();
              System.out.println("scheduleWithFixedDelay执行花费时间=" + (end - start) / 1000 + "m");
              System.out.println("scheduleWithFixedDelay执行完成时间："
                      + DateFormat.getTimeInstance().format(new Date()));
              System.out.println("======================================");
          }
      }, 1, 2, TimeUnit.SECONDS);
  
  }
  ```

+ **newSingleThreadExecutor**：

  创建一个单线程化的线程池，它只有一个线程，用仅有的一个线程来执行任务，保证所有的任务按照指定顺序（FIFO，LIFO，优先级）执行，所有的任务都保存在队列LinkedBlockingQueue中，等待唯一的单线程来执行任务。

| 工厂方法                | corePoolSize | maximumPoolSize   | keepAliveTime | workQueue           |
| ----------------------- | ------------ | ----------------- | ------------- | ------------------- |
| newCachedThreadPool     | 0            | Integer.MAX_VALUE | 60s           | SynchronousQueue    |
| newFixedThreadPool      | nThreads     | nThreads          | 0             | LinkedBlockingQueue |
| newSingleThreadExecutor | 1            | 1                 | 0             | LinkedBlockingQueue |
| newScheduledThreadPool  | corePoolSize | Integer.MAX_VALUE | 0             | DelayedWorkQueue    |

#### 线程池的7个参数

+ 核心线程：任务执行完毕后得到保留的线程 corePoolSize指定的线程数量为核心线程数
+ 救急线程：任务执行完毕后即被释放的线程 核心线程全忙且任务队列满时使用

![image-20230319161030197](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230319161030197.png)

**7种拒绝策略**

1. corePoolSize 核心线程数目 - 池中会保留的最多线程数
2. maximumPoolSize 最大线程数目 - 核心线程+救急线程的最大数目
3. keepAliveTime 生存时间 - 救急线程的生存时间，生存时间内没有新任务，此线程资源会释放
4. unit 时间单位 - 救急线程的生存时间单位，如秒、毫秒等
5. workQueue - 当没有空闲核心线程时，新来任务会加入到此队列排队，队列满会创建救急线程执行任务
6. threadFactory 线程工厂 - 可以定制线程对象的创建，例如设置线程名字、是否是守护线程等
7. handler 拒绝策略 - 当所有线程都在繁忙，workQueue 也放满时，会触发拒绝策略
   1. 抛异常 java.util.concurrent.ThreadPoolExecutor.AbortPolicy
   2. 由调用者执行任务 java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy
   3. 丢弃任务 java.util.concurrent.ThreadPoolExecutor.DiscardPolicy
   4. 丢弃最早排队任务 java.util.concurrent.ThreadPoolExecutor.DiscardOldestPolicy



## HashMap

### 基本原理

#### 索引的计算

对hash有
$$
hash = x \% 2^n = x \& (2^n-1)
$$
此公式当且仅当capacity为$2^n$时有效

如果追求扩容效率，应当选用$2^n$作为capacity；如果追求分布性，则最好选用质数作为capacity。

选用质数作为capacity不需要经过二次哈希。

#### 二次哈希

+ 不二次哈希的后果

  标准的一次哈希移位运算

  ```java
  static int indexFor(int h, int length) {
      return h & (length-1);
  }
  ```

  **哈希码的高位压根就没有参与运算**，全部被丢弃了。不管哈希码的高位是多少，都不会影响最终Index的计算结果，因为只有低位才参与了运算，这样的哈希函数我们认为是不好的，它会带来更多的冲突，影响HashMap的效率。

+ 二次哈希

  要求高位也参与运算

  二次哈希的过程是这样的：首先，根据key对象的hashCode()方法得到一个32位的整数值（称为原始哈希码），然后对这个整数值进行高低位异或运算（称为二次哈希），最后用这个异或后的结果与HashMap容量减一进行按位与运算，得到最终的数组下标

  ```java
  static final int hash(Object key) {
      int h;
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  ```

  HashMap通过将哈希码的高16位与低16位进行异或运算，得到一个新的哈希码，这样就可以让高位也参与到运算，这个函数也被称作**扰动函数**。

  + 右移16位的原因

    根据哈希码计算下标Index的过程，大家也发现了。实际上，只有数组长度以内的低位才会参与运算。例如数组长度是16，那么只有低4位会参与计算；如果数组长度是256，那么只有低8位会参与计算；**如果数组长度是65536，那么只有低16位会参与计算**。HashMap取16位是一个折中的数字，绝大部分情况下，HashMap数组的长度都不会超过65536。



### HashMap的扩容机制

- capacity 即容量，默认16。
- loadFactor 加载因子，默认是0.75f
  - 底层是因为泊松分布 我实在是 完全不懂 可以看[从泊松分布谈起HashMap为什么默认扩容因子是0.75 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/396019103)
  - 如果加载因子过小，那么扩容门槛低，扩容频繁，这虽然能使元素存储得更稀疏，有效避免了哈希冲突发生，同时操作性能较高，但是会占用更多的空间。
  - 如果加载因子过大，那么扩容门槛高，扩容不频繁，虽然占用的空间降低了，但是这会导致元素存储密集，发生哈希冲突的概率大大提高，从而导致存储元素的数据结构更加复杂（用于解决哈希冲突），最终导致操作性能降低。
  - 还有一个因素是为了提升扩容效率。因为`HashMap`的容量（`size`属性，构造函数中的`initialCapacity`变量）有一个要求：它一定是2的幂。所以加载因子选择了0.75就可以保证它与容量的乘积为整数。
- threshold 阈值。阈值=容量*加载因子。默认12。当元素数量超过阈值时便会触发扩容

![image-20230319141219881](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230319141219881.png)

#### JDK7中的扩容机制

7中的HashMap由数组+链表组成 头插法 **大于等于阈值且没有空位(插入时会发生哈希碰撞)才扩容**

JDK7的扩容机制相对简单，有以下特性：

- 空参数的构造函数：以默认容量、默认负载因子、默认阈值初始化数组。内部数组是**空数组**。
- 有参构造函数：根据参数确定容量、负载因子、阈值等。
- 第一次put时会初始化数组，其容量变为**不小于指定容量的2的幂数**。然后根据负载因子确定阈值。
- 如果不是第一次扩容，则 **新容量=旧容量×2** ， **负载因子新阈值=新容量×负载因子** 。

#### JDK8的扩容机制

JDK8的扩容做了许多调整。

8中的HashMap由数组+链表/红黑树组成 尾插法 **大于阈值就扩容**

HashMap的容量变化通常存在以下几种情况：

1. 空参数的构造函数：实例化的HashMap默认内部数组是null，即没有实例化。第一次调用put方法时，则会开始第一次初始化扩容，长度为16。
2. 有参构造函数：用于指定容量。会根据指定的正整数找到**不小于指定容量的2的幂数**，将这个数设置赋值给**阈值**（threshold）。第一次调用put方法时，会将阈值赋值给容量，然后让 阈值容量负载因子阈值=容量×负载因子 。（因此并不是我们手动指定了容量就一定不会触发扩容，超过阈值后一样会扩容！！）
3. 如果不是第一次扩容，则容量变为原来的2倍，**阈值也变为原来的2倍**。*（容量和阈值都变为原来的2倍时，负载因子还是不变）*

此外还有几个细节需要注意：

- 首次put时，先会触发扩容（算是初始化），然后存入数据，然后判断是否需要扩容；
- 不是首次put，则不再初始化，直接存入数据，然后判断是否需要扩容；

由于数组的容量是以2的幂次方扩容的，那么一个Entity在扩容时，新的位置要么在**原位置**，要么在**原长度+原位置**的位置。

在map**数组长度超过64且链表长度超过8后**，HashMap的实现由数组+链表转为数组+**红黑树**，**8**被称为树化标准长度

低长度下使用链表的原因：链表底层为Node，红黑树底层为TreeNode。红黑树节点比链表更消耗内存。树化是极偶然情况。

+ 红黑树的特点：父节点的一侧都是比父节点小的元素，父节点的另一侧都是比父节点大的元素，搜索时间复杂度为O(log2n)

  ![image-20230318221647352](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230318221647352.png)

![image-20230319104844782](https://cdn.jsdelivr.net/gh/WangMinan/Pics/image-20230319104844782.png)

有关退化情况2，对根节点左孩子、右孩子与左孙子的检查发生在节点移除前。



### HashMap为什么线程不安全

首先HashMap是**线程不安全**的，其主要体现：

1.在jdk1.7中，在多线程环境下，扩容时会造成环形链或数据丢失。

2.在jdk1.8中，在多线程环境下，会发生数据覆盖的情况。

https://cloud.tencent.com/developer/article/1631902

JDK7

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
      while(null != e) {
          	Entry<K,V> next = e.next;
	        if (rehash) {
            	e.hash = null == e.key ? 0 : hash(e.key);
          	}
          	int i = indexFor(e.hash, newCapacity);
         	e.next = newTable[i];
         	newTable[i] = e;
        	e = next;
		}
    }
}
```

在对table进行扩容到newTable后，需要将原来数据转移到newTable中，注意10-12行代码，这里可以看出在转移元素的过程中，使用的是**头插法**，也就是链表的顺序会翻转，这里也是形成死循环的关键点。

当一个线程阻塞在11行而另一个线程完成重hash之后会形成环形链表

JDK8

在jdk1.8中对HashMap进行了优化，在发生hash碰撞，不再采用头插法方式，而是**直接插入链表尾部**，因此不会出现环形链表的情况，但是在多线程的情况下仍然不安全，这里我们看jdk1.8中HashMap的put操作源码：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
     Node<K,V>[] tab; Node<K,V> p; int n, i;
     if ((tab = table) == null || (n = tab.length) == 0)
         n = (tab = resize()).length;
     if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
         tab[i] = newNode(hash, key, value, null);
     else {
         Node<K,V> e; K k;
         if (p.hash == hash &&
             ((k = p.key) == key || (key != null && key.equals(k))))
             e = p;
         else if (p instanceof TreeNode)
             e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
         else {
             for (int binCount = 0; ; ++binCount) {
                 if ((e = p.next) == null) {
                     p.next = newNode(hash, key, value, null);
                     if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                         treeifyBin(tab, hash);
                     break;
                 }
                 if (e.hash == hash &&
                     ((k = e.key) == key || (key != null && key.equals(k))))
                     break;
                 p = e;
             }
         }
         if (e != null) { // existing mapping for key
             V oldValue = e.value;
             if (!onlyIfAbsent || oldValue == null)
                 e.value = value;
             afterNodeAccess(e);
             return oldValue;
         }
     }
     ++modCount;
     if (++size > threshold)
         resize();
     afterNodeInsertion(evict);
     return null;
}
```

这是jdk1.8中HashMap中put操作的主函数， 注意第6行代码，如果没有hash碰撞则会直接插入元素。**如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第6行代码中。**假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给**覆盖**，发生线程不安全。

这里只是简要分析下jdk1.8中HashMap出现的线程不安全问题的体现，后续将会对java的集合框架进行总结，到时再进行具体分析。



### Hashtable vs ConcurrentHashMap

**要求**

* 掌握 Hashtable 与 ConcurrentHashMap 的区别
* 掌握 ConcurrentHashMap 在不同版本的实现区别

> 更形象的演示，见资料中的 hash-demo.jar，运行需要 jdk14 以上环境，进入 jar 包目录，执行下面命令
>
> ```
> java -jar --add-exports java.base/jdk.internal.misc=ALL-UNNAMED hash-demo.jar
> ```

#### **Hashtable 对比 ConcurrentHashMap**

* Hashtable 与 ConcurrentHashMap 都是线程安全的 Map 集合
* Hashtable 并发度低，**整个 Hashtable 对应一把锁**，同一时刻，只能有一个线程操作它
* ConcurrentHashMap 并发度高，**整个 ConcurrentHashMap 对应多把锁**，只要线程访问的是不同锁，那么不会冲突

#### **ConcurrentHashMap 1.7**

* 数据结构：`Segment(大数组) + HashEntry(小数组) + 链表`，每个 Segment 对应一把锁，如果多个线程访问不同的 Segment，则不会冲突
* 并发度：Segment 数组大小即并发度，决定了同一时刻最多能有多少个线程并发访问。Segment 数组不能扩容，意味着并发度在 ConcurrentHashMap 创建时就固定了
* 索引计算
  * 假设大数组长度是 $2^m$，key 在大数组内的索引是 key 的二次 hash 值的高 m 位
  * 假设小数组长度是 $2^n$，key 在小数组内的索引是 key 的二次 hash 值的低 n 位
* 扩容：每个小数组的扩容相对独立，小数组在超过扩容因子时会触发扩容，每次扩容翻倍
* Segment[0] 原型：首次创建其它小数组时，会以此原型为依据，数组长度，扩容因子都会以原型为准

#### **ConcurrentHashMap 1.8**

* 数据结构：`Node 数组 + 链表或红黑树`，数组的每个头节点作为锁，如果多个线程访问的头节点不同，则不会冲突。首次生成头节点时如果发生竞争，利用 cas 而非 syncronized，进一步提升性能
* 并发度：Node 数组有多大，并发度就有多大，与 1.7 不同，Node 数组可以扩容
* 扩容条件：Node 数组满 3/4 时就会扩容
* 扩容单位：以链表为单位从后向前迁移链表，迁移完成的将旧数组头节点替换为 ForwardingNode
* 扩容时并发 get
  * 根据是否为 ForwardingNode 来决定是在新数组查找还是在旧数组查找，不会阻塞
  * 如果链表长度超过 1，则需要对节点进行复制（创建新节点），怕的是节点迁移后 next 指针改变
  * 如果链表最后几个元素扩容后索引不变，则节点无需复制
* 扩容时并发 put
  * 如果 put 的线程与扩容线程操作的链表是同一个，put 线程会阻塞
  * 如果 put 的线程操作的链表还未迁移完成，即头节点不是 ForwardingNode，则可以并发执行
  * 如果 put 的线程操作的链表已经迁移完成，即头结点是 ForwardingNode，则可以协助扩容
* 与 1.7 相比是懒惰初始化(1.7饿汉式，1.8懒汉式)
* capacity 代表预估的元素个数，capacity / factory 来计算出初始数组大小，需要贴近 $2^n$ 
  * capacity和factor只有在第一次初始化map时起作用，之后factor仍按照0.75进行扩容
* loadFactor 只在计算初始数组大小时被使用，之后扩容固定为 3/4
* 超过树化阈值时的扩容问题，如果容量已经是 64，直接树化，否则在原来容量基础上做 3 轮扩容

[ConcurrentHashMap底层结构和原理详解](https://blog.csdn.net/qq_45408390/article/details/122189726)



## List及其子类

### ArrayList

动态数组，使用的时候，只需要操作即可，内部已经实现扩容机制。

- 线程不安全
- 有顺序，会按照添加进去的顺序排好
- 基于数组实现，随机访问速度快，插入和删除较慢一点
- 可以插入null元素，且可以重复

如果需要线程安全，则需要选择其他的类或者使用`Collections.synchronizedList(arrayList)`



#### private常量

如果我们创建的时候不指定大小，那么就会初始化一个默认大小为10,`DEFAULT_CAPACITY`就是默认大小。

```java
private static final int DEFAULT_CAPACITY = 10;
```

扩容时capacity右移一位，表现为`newCapacity=oldCapacity*2`

```java
private void grow(int minCapacity) {
        // 获取旧的容量
        int oldCapacity = elementData.length;
        // 新容量是翻了一倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果新的容量还是小于最小容量，则最新容量更新为最小容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果最新的容量大于最大的容量，则需要调用hugeCapacity函数将容量调小一点
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 底层是调用数组直接复制
        elementData = Arrays.copyOf(elementData, newCapacity);
}
```

里面定义了两个空数组，`EMPTY_ELEMENTDATA`名为空数组,`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`名为默认大小空数组,用来区分是空构造函数还是带参数构造函数构造的`ArrayList`,第一次添加元素的时候使用不同的扩容方式。 之所以是一个空数组，不是null，是因为使用的时候我们需要制定参数的类型，如果是null，那就根本不知道元素类型是什么了。

```java
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

还有一个特殊的成员变量`modCount`，这是快速失败机制(`fail_fast`)所需要的，也就是记录修改操作的次数，主要是迭代遍历的时候，防止元素被修改。如果操作前后的修改次数对不上，那么有些操作就是非法的。`transient`表示这个属性不需要自动序列化。

```java
protected transient int modCount = 0;
```



#### 线程安全问题

```java
public boolean add(E e) {
    //确定添加元素之后，集合的大小是否足够，若不够则会进行扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //插入元素
    elementData[size++] = e;
    return true;
}
```

场景1：多个线程都没进行扩容，但是执行了`elementData[size++] = e;`时，便会出现“数组越界异常”；
场景2：因为size++本身就是非原子性的，多个线程之间访问冲突，这时候两个线程可能对同一个位置赋值，就会出现“size小于期望值的结果”；



### LinkedList

LinkedList 是 Java 集合框架中一个重要的实现，其底层采用的**双向链表结构**。和 ArrayList 一样，LinkedList 也支持空值和重复值。

由于 LinkedList 基于链表实现，存储元素过程中，无需像 ArrayList 那样进行扩容。但有得必有失，LinkedList 存储元素的节点需要**额外的空间**存储前驱和后继的引用。另一方面，LinkedList 在链表**头部和尾部插入效率比较高**，但在**指定位置进行插入时，效率一般**。原因是，在指定位置插入需要定位到该位置处的节点，此操作的时间复杂度为`O(N)`。最后，LinkedList 是非线程安全的集合类，并发环境下，多个线程同时操作 LinkedList，会引发不可预知的错误。

LinkedList 继承自 **AbstractSequentialList**，AbstractSequentialList 又是什么呢？从实现上，AbstractSequentialList 提供了一套基于顺序访问的接口。通过继承此类，子类仅需实现部分代码即可拥有完整的一套访问某种序列表（比如链表）的接口。深入源码，AbstractSequentialList 提供的方法基本上都是通过 ListIterator 实现的，比如：



#### 线程安全问题

现在有3个结点n1、n2、n3。标记指针header的头结点指针first指向n1，尾结点指针last指向n3。

假如线程A、线程B同时要做add操作。

1.线程A找到last结点。

2.线程B找到last结点。（此时A、B找到的last结点是一样的）

3.线程A：新结点n4的previous设为last结点，之前的last结点的next设为新结点n4。size+1=4

4.线程B:新结点n5的previous设为last结点，之前的last结点的next设为新结点n5。size+1=5

此时，会出现这种情况：


![img](https://cdn.jsdelivr.net/gh/WangMinan/Pics/20210729154417515.png)

n3的next指向n5，header的last指向n5，直接跳过n4结点



### Vector

通过观察源码，发现 Vector 类中的大部分方法都是由 synchronized 关键字来修饰的，这也就保证了所有的对外接口都会以 Vector 对象为锁。访问 Vector 的任何方法都必须获得对象的 intrinsic lock (或叫 monitor lock )，所以在Vector内部，所有的方法都不会被多线程访问。
![在这里插入图片描述](https://cdn.jsdelivr.net/gh/WangMinan/Pics/9c1218672787412283a0ff073be2e79d.png)

但单个方法的原子性不代表复合操作也具有原子性。例如在Vector中一边插入一边删除时即存在线程安全问题。

要真正达成线程安全，还需要以 Vector 对象为锁，来进行同步处理。



### 线程安全的一些解决方案

#### Vector和Collections.SynchronizedList的get方法要加锁

Vector和Collections.SynchronizedList的get方法加了synchronized后可以保证顺序性与实时一致性，当一个线程在读取数据时，一定可以看到其他线程解锁前写入的全部数据。

Vector和Collections.SynchronizedList的数组并没有用volatile修饰，如果不加锁，也无法保证可见性。



#### 线程安全的三种List

```java
//方法上使用sync关键字（读写均加锁）
Vector vector = new Vector();
//写操作每一次均copy一个数组，读操作不加锁（写加锁性能低，读不加锁性能极高）
CopyOnWriteArrayList<Integer> r2 = new CopyOnWriteArrayList<>();
//使用sync代码块装饰传入List的读写操作（读写均加锁）
List<String> r3 = Collections.synchronizedList(new ArrayList<>());
```

+ Vector/Collections.synchronizedList：读写均加锁，来实现线程安全；
+ CopyOnWriteArrayList基于写时复制技术实现的，读操作无锁（读取快照），写操作有锁，体现了读写分离的思想，但是无法提供实时一致性
