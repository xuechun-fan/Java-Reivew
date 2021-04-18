# ReentrantLock源码分析

### 1、首先上图，看一下类和接口的关系

![image-20210417191922212](E:\Github\Java-Reivew\typro资源\ReentrantLock类图.png)

### 2、分析类结构

![image-20210417200927291](E:\Github\Java-Reivew\typro资源\ReentrantLock类结构.png)

这张图可以看到，ReentrantLock类中定义了AbstractQueuedSynchronizer类的子类作为静态内部类。这是JDK推荐的使用AbstractQueuedSynchronizer的方式，即在锁类的内部以继承的方式实现AQS框架。（补充：AQS是一个用来构建锁和同步器的框架，一般实现锁的类都需要内部定义AQS的子类，它为锁对象提供了锁的的获取、释放、阻塞和唤醒等一系列的机制）

![image-20210417201055699](E:\Github\Java-Reivew\typro资源\image-20210417201055699.png)

另外还有：

- 非公平锁
  ![image-20210417201229113](E:\Github\Java-Reivew\typro资源\image-20210417201229113.png)

- 公平锁
  ![image-20210417201245232](C:\Users\FXC\AppData\Roaming\Typora\typora-user-images\image-20210417201245232.png)

这两张图说明ReentrantLock可以实现公平锁和非公平锁两种锁机制。具体的做法时通过构造方法传参：
![image-20210417201515151](E:\Github\Java-Reivew\typro资源\image-20210417201515151.png)
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



### 3、分析锁的使用过程

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

首先会来到![image-20210417204641158](E:\Github\Java-Reivew\typro资源\image-20210417204641158.png)sync就是ReentrantLock类中的一个AQS的实例对象，也就是说，我们锁的其实是AQS的实例对象。
ReentrantLock实例对象lock持有sync实例变量。![image-20210417204940233](E:\Github\Java-Reivew\typro资源\image-20210417204940233.png)

接下来看sync.lock()方法的执行：

- 公平锁：![image-20210417205405231](E:\Github\Java-Reivew\typro资源\image-20210417205405231.png)

这里调用的是sync实例的方法，来到了ReentrantLock类中的内部类FairSync中定义的方法。

- 非公平锁：![image-20210417205710462](E:\Github\Java-Reivew\typro资源\image-20210417205710462.png)

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
       //	该方法见Code1源码解析
        if (!tryAcquire(arg) &&	 //如果获取到了锁，tryAcquire()方法返回true，则该行代码取反结果为false，&&短路作用，acquire()方法直接返回，一直向上返回到lock.lock()的调用处，也就是意味着锁是空闲的，当前线程现在已经成功获取到了锁，可以接着执行需要lock.lock()方法调用后面的代码块,
            //如果没有获取到锁，即当前锁已被占有，那么上一行代码就是true，转而会接着该行代码，首先分析addWaiter(Node.EXCLUSIVE) 
            // Node.EXCLUSIVE for exclusive, Node.SHARED for shared
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	//acquireQueued方法见下方Code2源码解析
            selfInterrupt();   |	//	如果。。。，中断当前线程
    }                          |
                               |
/**                           \|/
 * Creates and enqueues node for current thread and given mode.
 * 为当前线程和给定模式创建节点并将其排队。
 * @param mode Node.EXCLUSIVE for 独占式, Node.SHARED for 共享式
 * @return the new node	返回的是当前线程封装的Node节点
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);//调用如下的Node构造方法将当前线程封装成Node节点，Node在AQS中定义的内部类
/**
    Node(Thread thread, Node mode) {     // Used by addWaiter
		this.nextWaiter = mode;
		this.thread = thread;
	}
*/
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;	//将pred执行尾节点
    if (pred != null) {//如果尾节点不为空
        node.prev = pred;//令当前线程节点的前驱指针指向尾节点
        if (compareAndSetTail(pred, node)) {//通过CAS操作尝试将当前节点设置为尾节点，底层调用的式Unsafe类的方法
			//private final boolean compareAndSetTail(AbstractQueuedSynchronizer.Node expect, 
			//														AbstractQueuedSynchronizer.Node update)
            pred.next = node;//将原来的尾节点的next指针指向新的尾节点，即当前线程的节点node
            return node;
        }
    }
    enq(node);//如果尾节点为空即当前阻塞队列还是空的，则将当前线程节点插入到队列中，如果有必要就进行初始化，该方法定义在AQS中，源代码分析见下面			     |
    return node;	   //      |		//返回的是当前线程封装的Node节点
}                      //      |
					 //     \|/
	//       		enq(Node node)源码解析
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert 传入参数为需要插入的节点，即当前线程封装成的节点
     * @return node's predecessor	返回传入节点的前驱节点
     */
    private Node enq(final Node node) {
        for (;;) {//死循环
            Node t = tail;//临时Node引用变量指向尾节点
            if (t == null) { // Must initialize：如果尾节点为空，说明当前还没有形成队列，则必须进行初始化
                if (compareAndSetHead(new Node()))//创建一个“空”节点，也可以理解为dummy节点，并通过CAS操作，让head节点指向该哑节点
                    tail = head;//如果head成功通过CAS操作指向了哑节点，则令尾节点也指向该节点，见下图
                    //					|------------|
                    //		head ------> | dummy哑节点 |  <------tail    
                    //					|------------|
            } else {//如果尾节点不为空，即当前已经初始化完成
                node.prev = t;//将当前节点的前驱指针指向尾节点
                if (compareAndSetTail(t, node)) {//通过CAS操作将当前节点设置为尾节点
                    t.next = node;//将原来的尾节点的后继指针指向当前线程节点，如果是刚初始化完则原来尾节点就是dummy节点
                    return t;//返回当前线程节点的前驱节点
                }
            }
        }
    }
/**	分析到这里，我们就可以看出，addWaiter(Node mode)方法总的作用就是将需要排队的线程封装成Node节点，并加入到队尾，这里涉及两种情况：case1：如果当前队列还没有初始化，则需要调用enq(Node node)方法进行队列初始化，并将当前节点加入到队尾；case2：如果当前队列已经初始化过了，则直接将当前线程节点加入队尾即可。
Note：这里说的初始化，在源码上体现的就是在队列中是否已经创建了dummy哑节点。
**/
```

public final void acquire(int arg) 方法定义在AQS抽象类中，并使用了final关键字修饰，表明该方法不可被重写，因此任何子类调用该方法均是跳转到AQS类中的定义来执行。在该方法中if判断条件，首先是调用了tryAcquire(arg)，该方法再AQS中定义了模板，在ReentrantLock类中的子类FairSync以及UnFairSync中都有实现，我们点进去看一下该方法源码，并加上注释

#### Code1、trytryAcquire(int acquires)源码解析

```java
/**
* Fair version of tryAcquire.  Don't grant access unless
* recursive call or no waiters or is first.
*/
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();//获取当前线程
    int c = getState();//获取当前锁的状态
    if (c == 0) {//如果当前锁状态为0，说明当前锁资源空闲
        if (!hasQueuedPredecessors() &&//检查当前队列里是否已经有等待的线程在排队
            compareAndSetState(0, acquires)) {//如果没有，则CAS尝试获取锁，如果CAS成功，转而执行下一句，将当前线程设置为有效线程，并且返回true
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//如果当前锁非空闲，则判断当前申请锁的线程和当前持有锁的线程是否是同一个线程，简单的来说，这里是实现了可重入锁的机制
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //	如果当前锁非空闲，且申请锁的线程与持有锁的线程并非同一个线程，则获取锁失败，返回false
    return false;
}

//	AQS中定义的方法：查询是否有线程等待获取的时间长于当前线程，即查看当前AQS中的阻塞队列是否已经形成
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

#### Code2、acquireQueued(final Node node, int arg)源码解析

```java
	/**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node ：
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

