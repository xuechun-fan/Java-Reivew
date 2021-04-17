# ReentrantLock源码分析

### 1、首先上图，看一下类和接口的关系

![image-20210417191922212](E:\Github\java基础复习\typro资源\ReentrantLock类图.png)

### 2、分析类结构

![image-20210417200927291](E:\Github\java基础复习\typro资源\ReentrantLock类结构.png)

这张图可以看到，ReentrantLock类中定义了AbstractQueuedSynchronizer类的子类作为静态内部类。这是JDK推荐的使用AbstractQueuedSynchronizer的方式，即在锁类的内部以继承的方式实现AQS框架。（补充：AQS是一个用来构建锁和同步器的框架，一般实现锁的类都需要内部定义AQS的子类，它为锁对象提供了锁的的获取、释放、阻塞和唤醒等一系列的机制）

![image-20210417201055699](E:\Github\java基础复习\typro资源\image-20210417201055699.png)

另外还有：

- 非公平锁
  ![image-20210417201229113](E:\Github\java基础复习\typro资源\image-20210417201229113.png)

- 公平锁
  ![image-20210417201245232](C:\Users\FXC\AppData\Roaming\Typora\typora-user-images\image-20210417201245232.png)

这两张图说明ReentrantLock可以实现公平锁和非公平锁两种锁机制。具体的做法时通过构造方法传参：
![image-20210417201515151](E:\Github\java基础复习\typro资源\image-20210417201515151.png)
源码如下：

```java
//	无参构造方法
public ReentrantLock() {
    sync = new NonfairSync();
}
//	有参构造方法
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();	//	
}

/**
*	通过这两个构造方法可以得知，默认的ReentrantLock是非公平锁，如果调用有参构造方法，并传入true，则会创建一个公平锁
*/	
```



## 3、分析锁的使用过程

首先来看一下锁的使用的代码模板：

```java
private Lock lock = new ReentrantLock();
public void testLock(){
    lock.lock();
    try{
        //	业务代码
    }finally{
        lock.unlock();
    }
}
```

```java
lock.lock();
```

这行代码会开始获取锁。点进去看调用流程：

首先会来到![image-20210417204641158](E:\Github\java基础复习\typro资源\image-20210417204641158.png)sync就是ReentrantLock类中的一个AQS的实例对象，也就是说，我们锁的其实是AQS的实例对象。
ReentrantLock实例对象lock持有sync实例变量。![image-20210417204940233](E:\Github\java基础复习\typro资源\image-20210417204940233.png)

接下来看sync.lock()方法的执行：

- 公平锁：![image-20210417205405231](E:\Github\java基础复习\typro资源\image-20210417205405231.png)

这里调用的是sync实例的方法，来到了ReentrantLock类中的内部类FairSync中定义的方法。

- 非公平锁：![image-20210417205710462](E:\Github\java基础复习\typro资源\image-20210417205710462.png)

这里有一个有趣的地方，也引出了一个知识点：**公平锁和非公平锁的区别?**

分析：

​		通过源码可以看出，公平锁直接调用了acquire（1）方法，而非公平锁则是先通过compareAndSetState(0, 1)，（Note：这是一个CAS操作，它并不是任何锁相关的类的方法，是Unsafe类中的方法，关于Unsafe类以后会详细研究一下，在这里我们只需要知道它能够保证我们完成对锁状态变量进行CAS更新。)这个方法的作用就是它会首先尝试通过CAS的方式去抢夺锁，如果当前锁对象是空闲的状态则抢夺成功返回true，否则返回false。**case1：**抢夺成功，则会执行setExclusiveOwnerThread(Thread.currentThread()); 语句直接设置当前拥有独占访问权限，即占有锁的有效线程。**case2：**抢夺失败，则会转而执行acquire(1)方法调用。接下来看一下公平锁和非公平锁都有的代码：acquire(1)方法调用。

这里给出该方法源代码以及注释的机翻：

```java
/**
	以独占模式获取，忽略中断。通过调用至少一次tryAcquire()方法来实现，并在成功时返回。否则线程会排队，可能会反复阻塞和取消阻塞，调用tryAcquire()方法，直到成功。此方法可用于实现方法Lock中的lock()方法。

	@param arg获取参数。这个值被传递给{@link#tryAcquire}。
*/
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

```

public final void acquire(int arg) 方法定义在AQS抽象类中，并使用了final关键字修饰，表明该方法不可被重写，因此任何子类调用该方法均是跳转到AQS类中的定义来执行。