## 1、AbstractQueuedSynchronizer之AQS
AbstractQueuedSynchronizer简称为AQS，抽象的队列同步器

AQS：是用来构建锁或者其它同步器组件的重量级基础框架及整个JUC体系的基石，使用一个int类变量表示持有锁的状态，通过内置的FIFO队列来完成资源获取线程的排队工作，将每条要去抢占资源的线程封装成一个Node节点来实现锁的分配，通过CAS完成对State值的修改。

### 1.1、自定义同步组件的设计思路

同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（getState()、setState(int newState)和compareAndSetState(int expect,int update)）来进行操作，因为它们能够保证状态的改变是安全的。

子类推荐被定义为自定义同步组件的静态内部类，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合（组合）同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：
- 锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；
- 同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

同步器的设计是基于模板方法模式的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。
重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。
getState()：获取当前同步状态。
setState(int newState)：设置当前同步状态。
compareAndSetState(int expect,int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。

![在这里插入图片描述](https://img-blog.csdnimg.cn/479865d6941647bc88913510798b4881.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 1.1、AQS为什么是JUC内容中最重要的基石
- ReentrantLock
- ReentrantLock
- ReentrantReadWriteLock
- Semaphore
- ......


**进一步理解锁和同步器的关系：**
- 锁，面向锁的使用者：定义了程序员和锁交互的使用层API，隐藏了实现细节，你调用即可。
- 同步器，面向锁的实现者：比如Java并发大神DougLee，提出统一规范并简化了锁的实现，屏蔽了同步状态管理、阻塞线程排队和通知、唤醒机制等。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/ef41aa19c11043158ce1143819408b18.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 
### 1.2、AQS的作用是什么？
AQS使用一个volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，将每条要去抢占资源的线程封装成一个Node节点来实现锁的分配，通过CAS完成对State值的修改。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4523a88b520a4db08c3be9eb0fd7a079.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

加锁会导致阻塞，有阻塞就需要排队，实现排队必然需要队列，抢到资源的线程直接使用处理业务，抢不到资源的必然涉及一种排队等候机制。抢占资源失败的线程继续去等待，但等候线程仍然保留获取锁的可能且获取锁流程仍在继续，如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配，这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中，这个队列就是AQS的抽象表现，它将请求共享资源的线程封装成队列的结点（Node），通过CAS、自旋以及LockSupport.park()的方式，维护state变量的状态，使并发达到同步的效果。
 

同步器的结构中，包含了一个
- 静态的Node节点
	- waitStatus：等待状态
	- SHARED：共享模式
	- EXCLUSIVE：独占模式
	- prev：前指针
	- next：后指针
	- thread：线程
	- nextWaiter：指向下一个处于CONDITION状态的节点
- head节点
- tail节点
- 锁状态state
![在这里插入图片描述](https://img-blog.csdnimg.cn/38d77f1b60954d4c880c69eba46cf55e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/935e5a2a0e8243a9aa58cb1491e8aa62.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/c2070a4041074fe794e1a153965ca879.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7953f7ac15f4e05a5cbd329d3b4e656.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

对比公平锁和非公平锁的 tryAcquire()方法的实现代码，其实差别就在于非公平锁获取锁时比公平锁中少了一个判断 !hasQueuedPredecessors()
 hasQueuedPredecessors() 中判断了是否需要排队，导致公平锁和非公平锁的差异如下：
- 公平锁：公平锁讲究先来先到，线程在获取锁时，如果这个锁的等待队列中已经有线程在等待，那么当前线程就会进入等待队列中； 
- 非公平锁：不管是否有等待队列，如果可以获取锁，则立刻占有锁对象。也就是说队列的第一个排队线程在unpark()，之后还是需要竞争锁（存在线程竞争的情况下）

### 1.3、源码解读
以Lock的非公平锁进行解读
Lock lock =new ReentrantLock();默认是非公平锁。

执行顺序：
- lock()   ====》 acquire()      ====》  tryAcquire(arg)      ====》 addWaiter(Node.EXCLUSIVE)     ====》 acquireQueued(addWaiter(Node.EXCLUSIVE), arg)    
- unlock()  ====》 sync.release(1);  ====》tryRelease(arg)  ====》unparkSuccessor

整个ReentrantLock 的加锁过程，可以分为三个阶段
- 尝试加锁
- 加锁失败，线程进入队列
- 线程入队列之后，进入阻塞状态

模拟三个线程（A，B，C）去竞争同一把锁
- 线程A是第一个线程，但是耗时严重，长时间占有锁
- 线程B是第二个线程，线程B看到锁被线程A占有，会尝试进行获取锁，获取不到，就进入AQS队列中，等待线程A执行完毕之后，再去抢占锁
- 线程C是第三个线程，线程C看到锁被线程A占有，会尝试进行获取锁，获取不到，就进入AQS队列中，等待线程B执行完毕之后，再去抢占锁

```java
public class AQSDemo
{
    public static void main(String[] args)
    {
        ReentrantLock reentrantLock = new ReentrantLock();//非公平锁

        // A B C三个顾客，去银行办理业务，A先到，此时窗口空无一人，他优先获得办理窗口的机会，办理业务。
        // A 耗时严重，估计长期占有窗口
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in A");
                //暂停50分钟线程
                try { TimeUnit.MILLISECONDS.sleep(50); } catch (InterruptedException e) { e.printStackTrace(); }
            }finally {
                reentrantLock.unlock();
            }
        },"A").start();

        //B是第2个，B一看到受理窗口被A占用，只能去候客区等待，进入AQS队列，等待着A办理完成，尝试去抢占受理窗口。
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in B");
            }finally {
                reentrantLock.unlock();
            }
        },"B").start();


        //C是第3个，C一看到受理窗口被A占用，只能去候客区等待，进入AQS队列，等待着A办理完成，尝试去抢占受理窗口,前面是B顾客，FIFO
        new Thread(() -> {
            reentrantLock.lock();
            try
            {
                System.out.println("----come in C");
            }finally {
                reentrantLock.unlock();
            }
        },"C").start();

    }
}

```




初始的时候，AQS的state状态值为0。

#### 1.3.1、lock.lock()加锁 

1. 线程A第一个进来，判断当前的状态是否为0，如果是，则比较并交换，将当前的state状态值，设置为1，将当前线程设置为独占线程，则线程A获取到锁。
```java
        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```
setExclusiveOwnerThread（）
```java
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
```

2. 线程B第二个进来，因为线程A还处于持有锁状态，并未释放，也会去判断当前的状态（state）是否为0，因为现在锁被线程A所持有（state为1），所以状态state不为0，所以会进入到else，去获取锁。
```java
        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```
tryAcquire(arg)

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

```
3. nonfairTryAcquire(int acquires) 方法，线程B第二个进来，会判断当前的状态值，既不等于0，也不是独占线程，所以会返回false
```java
 /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

4. 接着 会进入 addWaiter(Node.EXCLUSIVE)，然后会将线程B和节点模式构造出一个Node节点（在AQS队列中的存储的元素是Node节点，Node节点中存放的线程），初始状态Tail和Head都是null，接着会将尾节点赋值给一个临时变量，判断当前的尾节点是否为null，尾节点为空，则直接将包含线程B的Node节点插入到AQS队列中去。
```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

5. enq(node)，方法里面是一个自旋锁，插入队列的时候，如果尾节点为null，会先进行一个队列的初始化，向Head中插入一个生成的哨兵节点（Thread为null，state状态值为0），将尾节点的指针，指向哨兵节点。然后继续自旋，进入到另外的一个条件，将线程B的节点的前赴节点执行哨兵节点，并以CAS的方式将线程B节点设置为尾节点，并将哨兵节点的next指针执行线程B节点，然后返回
```java
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

6. 添加线程B的Node节点进入队列后，将线程B的Node节点返回。
```java
 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
7. acquireQueued（node, arg），该方法中也是一个自旋锁，先获取线程B构造的Node节点的前赴节点，判断前赴节点是不是Head，然后再去尝试获取锁，两个条件只满足一个，所以会走向下一个判断。

```java
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

```java
       final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

8. shouldParkAfterFailedAcquire（p, node），p是线程B构成的节点的前赴节点，Node是线程B构成的节点，获取p节点的状态，判断状态值，p节点的等待状态值（waitStatus）为0，根据条件判断到else，以CAS操作的方式将等待状态值（waitStatus）改为-1，

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
9. 然后进入parkAndCheckInterrupt（），将当前线程阻塞。

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/29ff5f494a0e4d058af316de1f12f9c0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

10. 线程C第三个进来，因为线程A还处于持有锁状态，并未释放，也会去尝试获取锁，判断当前的状态（state）是否为0，因为现在锁被线程A所持有（state为1），所以状态state不为0，也不是当前持有锁的线程，返回false。

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

11. 然后执行addWaiter(Node.EXCLUSIVE)方法，先会去构建线程C的节点，获取尾节点的node值，因为当前尾节点是线程B构成的节点，所以该pre节点不为空，然后将线程C节点返回

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
12. acquireQueued（node, arg），该方法中也是一个自旋锁，先获取线程C构造的Node节点的前赴节点，判断前赴节点是不是Head，然后再去尝试获取锁，两个条件只满足一个，所以会走向下一个判断。
```java
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
13. shouldParkAfterFailedAcquire（p, node），p是线程C构成的节点的前赴节点，Node是线程C构成的节点，获取p节点的状态，判断状态值，p节点的等待状态值（waitStatus）为0，根据条件判断到else，以CAS操作的方式将等待状态值（waitStatus）改为-1，

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
15. 然后进行自旋，进入parkAndCheckInterrupt（），将当前线程阻塞，因为线程B也在AQS队列，FIFO排列

```java
   private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/58b26739ec2d48c58278b603eedf18af.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 1.3.2、lock.unlock()解锁 
1. 当线程A获取锁后，执行完业务逻辑之后，开始释放锁，进入tryRelease方法
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

2. 在tryRelease方法中，将AQS的状态减去1，看结果是否为0，如果结果为0，则将当前独占的线程置为null，更改AQS的状态
```java
 protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
3. 回到release方法，将head节点赋值给临时变量，因为head节点是哨兵节点，不为null且waitStatus等于-1
```java
 public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

4. 获取Head节点的waitStatus值，同时获取下一个节点（线程B构成的节点），唤醒线程B的节点
```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
5. 线程B被唤醒后，继续执行程序，返回false

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
6. 返回之后，在acquireQueued方法中继续自旋，线程B的节点获取先驱节点为Head节点，尝试获取锁，线程B获取锁成功，更改AQS的state,接着会将线程B构成的节点设置为Head节点，清空线程B节点的线程和和前驱，并断开原来head的next指针，完成哨兵节点的替换。依次循环......
```java
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
![在这里插入图片描述](https://img-blog.csdnimg.cn/d71f0ef8791c4c84a11b7ba5904bf106.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)




