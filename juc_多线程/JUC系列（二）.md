## 1、聊一聊Java“锁”
### 1.1、乐观锁和悲观锁
- 悲观锁：认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。synchronized关键字和Lock的实现类都是悲观锁。适合写操作多的场景，先加锁可以保证写操作时数据正确，显式的锁定之后再操作同步资源。
- 乐观锁：乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则根据不同的实现方式执行不同的操作，判断数据是否被其他的线程更新，
**乐观锁一般有两种实现方式：**
	1. 版本号机制
	2. CAS（Compare-and-Swap，即比较并替换）算法实现

乐观锁在Java中是通过使用无锁编程来实现，最常采用的是CAS算法，Java原子类中的递增操作就通过CAS自旋实现的。

### 1.2、线程8锁问题
#### 1.2.1、代码

```java
/* *
 * 资源类
 */
class Resource {
    public static synchronized void sendEmail() {
        try {
            TimeUnit.SECONDS.sleep(3);
            System.out.println("*******sendEmail");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public synchronized void sendMs() {
        System.out.println("*******sendMs");
    }

    public void sayHello() {
        System.out.println("*****sayHello");
    }
}
```

测试demo

```java
public class LockDemo {

    public static void main(String[] args) throws Exception {
        Resource resource = new Resource();
        new Thread(() -> resource.sendEmail(), "A").start();
        Thread.sleep(100);
        new Thread(() -> resource.sendMs(), "B").start();
    }
}
```
#### 1.2.2、锁的8问？
```java
 1.  标准访问（两个普通同步方法 sendEmail 和 sendMs），1个资源，两线程中间睡眠 100 毫秒，先打印邮件还是短信？
 2. 标准访问（两个普通同步方法 sendEmail 和 sendMs），1个资源，在 sendEmail() 方法中睡眠 4 秒，先打印邮件还是短信？
 3. 添加普通的 hello() 方法，1个资源，先打印邮件还是 hello？
 4. 2个资源，先打印邮件还是短信？
 5. 2个静态同步方法（sendEmail 和 sendMs），1个资源，先打印邮件还是短信？
 6. 2个静态同步方法（sendEmail 和 sendMs），2个资源，先打印邮件还是短信？
 7. 1个静态同步方法（sendEmail），1个普通同步方法(sendMs)，1个资源，先打印邮件还是短信？
 8. 1个静态同步方法（sendEmail），1个普通同步方法(sendMs)，2个资源，先打印邮件还是短信？
```

#### 1.2.3、答案
```java
  1. 标准访问，先打印邮件
  2. 邮件设置暂停4秒方法，先打印邮件
		对象锁
	       一个对象里面如果有多个synchronized方法，某一个时刻内，只要一个线程去调用其中的一个synchronized方法了，
	       其他的线程都只能等待，换句话说，某一个时刻内，只能有唯一一个线程去访问这些synchronized方法，
	       锁的是当前对象this，被锁定后，其他的线程都不能进入到当前对象的其他的synchronized方法	       
  3. 新增sayHello方法，先打印sayHello
       加个普通方法后发现和同步锁无关
  4. 两个资源，先打印短信
       换成两个对象后，不是同一把锁了，情况立刻变化
  5. 两个静态同步方法，同一个资源，先打印邮件
  6. 两个静态同步方法，同两个资源，先打印邮件，锁的同一个字节码对象
       静态同步方法锁的是Class，是全局锁
       synchronized实现同步的基础：java中的每一个对象都可以作为锁。
       具体表现为一下3中形式。
       对于普通同步方法，锁是当前实例对象，锁的是当前对象this，
       对于静态同步方法，锁是当前类的class对象
       对于同步方法块，锁的是synchronized括号里配置的对象。
  7. 一个静态同步方法，一个普通同步方法，同一个资源，先打印短信
  8. 一个静态同步方法，一个普通同步方法，同二个资源，先打印短信
       静态同步方法锁的是CLass，普通同步方法锁的是对象 加锁的位置不一样，相互无影响
```

#### 1.2.4、总结

```java
1.当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。
也就是说如果一个实例对象的普通同步方法获取锁后，该实例对象的其他普通同步方法必须等待获取锁的方法释放锁后才能获取锁，
但是别的实例对象的普通同步方法因为跟该实例对象的普通同步方法用的是不同的锁，
所以无需等待该实例对象已获取锁的普通同步方法释放锁就可以获取他们自己的锁。
 
2.这两把锁(this/Class)是两个不同的对象，所以静态同步方法与非静态同步方法之间是不会有影响的。
 
3.所有的静态同步方法用的也是同一把锁--类对象本身（Class），
一个静态同步方法获取锁后，其他的静态同步方法都必须等待该方法释放锁后才能获取锁，
不管是同一个实例对象的静态同步方法之间，还是不同的实例对象的静态同步方法之间，只要它们同一个类产生的实例对象
```
#### 1.2.5、8种锁的案例实际体现在3个地方
 - 作用于实例方法，当前实例加锁，进入同步代码前要获得当前实例的锁；
 - 作用于代码块，对括号里配置的对象加锁。
 - 作用于静态方法，当前类加锁，进去同步代码前要获得当前类对象的锁；

### 1.3、从字节码角度分析synchronized实现
字节码反编辑方式：
- javap -c ***.class文件反编译   对代码进行反汇编
- javap -v ***.class文件反编译    -v  -verbose  输出附加信息（包括行号、本地变量表，反汇编等详细信息）

#### 1.3.1、synchronized同步代码块
synchronized同步代码块中的实现，实际使用的是monitorenter和monitorexit指令，正常情况下是由一个monitorenter 和两个monitorexit指令存在，两个monitorexit指令，代表的是正常退出释放锁和异常退出释放锁，但是一个monitorenter 和两个monitorexit指令也不是恒定不变的，当手动去抛出一个异常，就会存在一个monitorenter 和一个monitorexit指令。
```java
public void m1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #5                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1
         8: aload_1
         9: dup
        10: astore_2
        11: monitorenter
        12: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: ldc           #3                  // String 1
        17: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        20: aload_2
        21: monitorexit
        22: goto          30
        25: astore_3
        26: aload_2
        27: monitorexit
        28: aload_3
        29: athrow
        30: return

```

#### 1.3.2、synchronized普通同步方法
当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程（monitor），然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程（monitor）。

```java
 public synchronized void m2();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String 1
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 15: 0
        line 16: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/song/test/Demo;

```

#### 1.3.3、synchronized静态同步方法
ACC_STATIC, ACC_SYNCHRONIZED访问标志区分该方法是否静态同步方法，当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程（monitor），然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程（monitor）。
```java
  public static synchronized void m3();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String 1
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 18: 0
        line 19: 8
```
#### 1.3.4、为什么任何一个对象都可以成为一个锁
因为Java中的每个对象都派生自Object类，在HotSpot虚拟中，monitor（管程）是由ObjectMonitor类来实现的，ObjectMonitor.java类对应C++中的ObjectMonitor.cpp，在ObjectMonitor.hpp中初始化了ObjectMonitor对象，在C++底层ObjectMonitor对象中，具有锁计数器和指向持有该锁的线程的指针的属性，所以说每个对象天生都带着一个对象监视器，自然每一个对象都可以成为一个锁。

- _owner：指向持有ObjectMonitor对象的线程
- _WaitSet：存放处于wait状态的线程队列
- _EntryList：存放处于等待锁block状态的线程队列
- _recursions：锁的重入次数
- _count：用来记录该线程获取锁的次数

所以说每个对象天生都带着一个对象监视器，自然每一个对象都可以成为一个锁。

#### 1.3.5、为什么说synchronized是操作系统级别的重量级锁

在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，挂起线程和恢复线程都需要转入内核态去完成，阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，因为用户态与内核态都有各自专用的内存空间，专用的寄存器等，用户态切换至内核态需要传递给许多变量、参数给内核，内核也需要保护好用户态在切换时的一些寄存器值、变量等，这种状态切换需要耗费处理器时间，如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长”，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。
![在这里插入图片描述](https://img-blog.csdnimg.cn/9861e00ab0744a7bb0f691a475f1aec1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 1.3.6、Synchronized的重入的实现机理
因为Java中的每个对象都派生自Object类，而java的Object类又对应了C++底层ObjectMonitor对象，每个ObjectMonitor对象拥有一个锁计数器和一个指向持有该锁的线程的指针。
- 当执行monitorenter时，如果目标锁对象的计数器为零，那么说明它没有被其他线程所持有，Java虚拟机会将该锁对象的持有线程设置为当前线程，并且将其计数器加1。
- 在目标锁对象的计数器不为零的情况下，如果锁对象的持有线程是当前线程，那么 Java 虚拟机可以将其计数器加1，否则需要等待，直至持有线程释放该锁。
- 当执行monitorexit时，Java虚拟机则需将锁对象的计数器减1。计数器为零代表锁已被释放。

### 1.4、公平锁和非公平锁
#### 1.4.1、从ReentrantLock卖票编码演示公平和非公平现象

```java
class Ticket
{
    private int number = 30;
    ReentrantLock lock = new ReentrantLock();
    public void sale()
    {
        lock.lock();
        try
        {
            if(number > 0)
            {
                System.out.println(Thread.currentThread().getName()+"卖出第：\t"+(number--)+"\t 还剩下:"+number);
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}

public class SaleTicketDemo
{
    public static void main(String[] args)
    {
        Ticket ticket = new Ticket();

        new Thread(() -> { for (int i = 0; i <35; i++)  ticket.sale(); },"a").start();
        new Thread(() -> { for (int i = 0; i <35; i++)  ticket.sale(); },"b").start();
        new Thread(() -> { for (int i = 0; i <35; i++)  ticket.sale(); },"c").start();
    }
}
```
#### 1.4.2、何为公平锁/非公平锁?
⽣活中，排队讲求先来后到视为公平。程序中的公平性也是符合请求锁的绝对时间的，其实就是 FIFO，否则视为不公平

- 公平锁：按序排队公平锁，就是判断同步队列是否还有先驱线程节点的存在(我前面还有人吗?)，如果没有先驱线程节点才能获取锁；
- 非公平锁：先占先得非公平锁，是不管这个事的，只要能抢获到同步状态就可以
#### 1.4.3、为什么会有公平锁/非公平锁的设计，为什么默认非公平？
公平锁和非公平锁的区别主要是抢占锁的时候，是否去判断同步队列中有等待的线程，如果去判断了，则说明是公平锁，如果没有去判断，说明是非公平锁。

- 公平锁：公平锁保证了排队的公平性，根据线程请求锁的时间进行判断，多个线程之间交替获取锁，避免了只有一个线程获取锁，多个线程一直在等待，发生锁饥饿的问题。
- 非公平锁：非公平锁能更充分的利用CPU 的时间片，减少 CPU 空闲状态时间，并且还减少了线程切换的开销。在非公平锁中不去判断同步队列中是否有等待的线程，减少了 CPU 空闲状态时间，更充分的利用CPU 的时间片，当一个刚释放锁的线程，再次获取同步锁的概率是非常大的，减少线程切换，这就避免了在公平锁状态下发生的线程之间切换开销，也节省了CPU的资源。

默认为非公平锁，是因为非公平锁能更充分的利用CPU 的时间片，减少 CPU 空闲状态时间，并且还减少了线程切换的开销。

#### 1.4.4、使⽤公平锁会有什么问题？
公平锁保证了排队的公平性，根据线程请求锁的时间进行判断，多个线程之间交替获取锁，避免了只有一个线程获取锁，多个线程一直在等待，发生锁饥饿的问题，但是在每一次获取锁的时候，需要去判断同步队列中是否有等待的线程，如果有，就切换线程去获取锁，线程的切换，会增减额外的开销。

#### 1.4.5、什么时候用公平？什么时候用非公平？
如果为了更高的吞吐量，很显然非公平锁是比较合适的，因为节省很多线程切换时间，吞吐量自然就上去了，否则那就用公平锁，大家公平使用。

### 1.5、可重入锁(又名递归锁)
#### 1.5.1、对“可重入锁”的解释：
可重入锁（又名递归锁）：是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁(前提，锁对象得是同一个对象)，不会因为之前已经获取过还没释放而阻塞。

- 可：可以。
- 重：再次。
- 入：进入。
- 锁：同步锁。

一个线程中的多个流程可以获取同一把锁，持有这把同步锁可以再次进入。自己可以获取自己的内部锁，在Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。

#### 1.5.2、“可重入锁”的种类
- 隐式锁（即synchronized关键字使用的锁）默认是可重入锁

因为Java中的每个对象都派生自Object类，而java的Object类又对应了C++底层ObjectMonitor对象，每个ObjectMonitor对象拥有一个锁计数器和一个指向持有该锁的线程的指针。当执行monitorenter时，如果目标锁对象的计数器为零，那么说明它没有被其他线程所持有，Java虚拟机会将该锁对象的持有线程设置为当前线程，并且将其计数器加1。在目标锁对象的计数器不为零的情况下，如果锁对象的持有线程是当前线程，那么 Java 虚拟机可以将其计数器加1，否则需要等待，直至持有线程释放该锁。当执行monitorexit时，Java虚拟机则需将锁对象的计数器减1。计数器为零代表锁已被释放。
```java
public class ReEntryLockDemo
{
    public static void main(String[] args)
    {
        final Object objectLockA = new Object();

        new Thread(() -> {
            synchronized (objectLockA)
            {
                System.out.println("-----外层调用");
                synchronized (objectLockA)
                {
                    System.out.println("-----中层调用");
                    synchronized (objectLockA)
                    {
                        System.out.println("-----内层调用");
                    }
                }
            }
        },"a").start();
    }
}
```

```java
/**
 * 在一个Synchronized修饰的方法或代码块的内部调用本类的其他Synchronized修饰的方法或代码块时，是永远可以得到锁的
 */
public class ReEntryLockDemo
{
    public synchronized void m1()
    {
        System.out.println("-----m1");
        m2();
    }
    public synchronized void m2()
    {
        System.out.println("-----m2");
        m3();
    }
    public synchronized void m3()
    {
        System.out.println("-----m3");
    }
    public static void main(String[] args)
    {
        ReEntryLockDemo reEntryLockDemo = new ReEntryLockDemo();
        reEntryLockDemo.m1();
    }
}

```

- 显式锁（即Lock）也有ReentrantLock这样的可重入锁。如果加锁和释放锁的次数不一致，会第二个线程始终无法获取到锁，线程一直处于在等待状态。

```java
public class ReEntryLockDemo
{
    static Lock lock = new ReentrantLock();

    public static void main(String[] args)
    {
        new Thread(() -> {
            lock.lock();
            try
            {
                System.out.println("----外层调用lock");
                lock.lock();
                try
                {
                    System.out.println("----内层调用lock");
                }finally {
                    // 这里故意注释，实现加锁次数和释放次数不一样
                    // 由于加锁次数和释放次数不一样，第二个线程始终无法获取到锁，导致一直在等待。
                    lock.unlock(); // 正常情况，加锁几次就要解锁几次
                }
            }finally {
                lock.unlock();
            }
        },"a").start();

        new Thread(() -> {
            lock.lock();
            try
            {
                System.out.println("b thread----外层调用lock");
            }finally {
                lock.unlock();
            }
        },"b").start();

    }
}
```
### 1.6、死锁及排查
#### 1.6.1、什么是死锁？
死锁是指两个或两个以上的线程在执行过程中,因争夺资源而造成的一种互相等待的现象,若无外力干涉那它们都将继续僵持下去。如果系统资源充足，进程的资源请求都能够得到满足，死锁出现的可能性就很低，否则就会因争夺有限的资源而陷入死锁。

例如：一个线程持有锁对象A，尝试获取锁B，另外一个线程持有锁对象B，尝试获取锁A，造成了相互等待的问题，若无外力干涉那它们都将继续僵持下去。

#### 1.6.2、产生死锁主要原因？
- 系统资源不足
- 资源分配不当
- 进程运行推进的顺序不合适

主要是线程之间顺序不合适导致的。

#### 1.6.3、请写一个死锁代码case

```java
public class DeadLockDemo
{
    public static void main(String[] args)
    {
        final Object objectLockA = new Object();
        final Object objectLockB = new Object();

        new Thread(() -> {
            synchronized (objectLockA)
            {
                System.out.println(Thread.currentThread().getName()+"\t"+"自己持有A，希望获得B");
                //暂停几秒钟线程 保证线程B能够获取锁对象objectLockB 
                try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
                synchronized (objectLockB)
                {
                    System.out.println(Thread.currentThread().getName()+"\t"+"A-------已经获得B");
                }
            }
        },"A").start();

        new Thread(() -> {
            synchronized (objectLockB)
            {
                System.out.println(Thread.currentThread().getName()+"\t"+"自己持有B，希望获得A");
                //暂停几秒钟线程 保证线程A能够获取锁对象objectLockB
                try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
                synchronized (objectLockA)
                {
                    System.out.println(Thread.currentThread().getName()+"\t"+"B-------已经获得A");
                }
            }
        },"B").start();

    }
}

```
#### 1.6.4、如何排查死锁
- 纯命令 ：使用 jps -l  找到进程编号，然后 jstack 进程编号     

```java
D:\demo\target\classes\com\song\test>jps
11760 RemoteMavenServer36
1664 Launcher
18216 DeadLockDemo
19384 Jps
3992
19244

D:\demo\target\classes\com\song\test>jstack 18216
2022-03-29 11:23:41
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.102-b14 mixed mode):

"DestroyJavaVM" #13 prio=5 os_prio=0 tid=0x0000000002abe800 nid=0x48e0 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"B" #12 prio=5 os_prio=0 tid=0x0000000018f18000 nid=0x20f0 waiting for monitor entry [0x000000001994f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.song.test.DeadLockDemo.lambda$main$1(DeadLockDemo.java:33)
        - waiting to lock <0x00000000d64018e0> (a java.lang.Object)
        - locked <0x00000000d64018f0> (a java.lang.Object)
        at com.song.test.DeadLockDemo$$Lambda$2/1324119927.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

"A" #11 prio=5 os_prio=0 tid=0x0000000018f11800 nid=0x2a7c waiting for monitor entry [0x000000001984f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.song.test.DeadLockDemo.lambda$main$0(DeadLockDemo.java:20)
        - waiting to lock <0x00000000d64018f0> (a java.lang.Object)
        - locked <0x00000000d64018e0> (a java.lang.Object)
        at com.song.test.DeadLockDemo$$Lambda$1/295530567.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

"Service Thread" #10 daemon prio=9 os_prio=0 tid=0x0000000018c0b800 nid=0x1f58 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x0000000018c06800 nid=0xef8 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x0000000018b95000 nid=0x363c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x0000000018b9e000 nid=0x287c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x0000000018b98000 nid=0x2de4 runnable [0x000000001924e000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:170)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
        at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
        at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
        at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
        - locked <0x00000000d63878f0> (a java.io.InputStreamReader)
        at java.io.InputStreamReader.read(InputStreamReader.java:184)
        at java.io.BufferedReader.fill(BufferedReader.java:161)
        at java.io.BufferedReader.readLine(BufferedReader.java:324)
        - locked <0x00000000d63878f0> (a java.io.InputStreamReader)
        at java.io.BufferedReader.readLine(BufferedReader.java:389)
        at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:61)

"Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x0000000017810000 nid=0x2e50 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x0000000018b58800 nid=0x294c runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=1 tid=0x00000000177ea800 nid=0x3430 in Object.wait() [0x0000000018b4e000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000d6208e98> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
        - locked <0x00000000d6208e98> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

"Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x00000000177c9000 nid=0x4094 in Object.wait() [0x0000000018a4e000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00000000d6206b40> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:502)
        at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
        - locked <0x00000000d6206b40> (a java.lang.ref.Reference$Lock)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"VM Thread" os_prio=2 tid=0x00000000177c8000 nid=0x4a40 runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000002c67800 nid=0x3f60 runnable

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x0000000002c69000 nid=0x4668 runnable

"GC task thread#2 (ParallelGC)" os_prio=0 tid=0x0000000002c6a800 nid=0x1ab8 runnable

"GC task thread#3 (ParallelGC)" os_prio=0 tid=0x0000000002c6c000 nid=0x4848 runnable

"VM Periodic Task Thread" os_prio=2 tid=0x0000000018c5b000 nid=0x1c6c waiting on condition

JNI global references: 335


Found one Java-level deadlock:
=============================
"B":
  waiting to lock monitor 0x0000000002d47ff8 (object 0x00000000d64018e0, a java.lang.Object),
  which is held by "A"
"A":
  waiting to lock monitor 0x0000000002d4a888 (object 0x00000000d64018f0, a java.lang.Object),
  which is held by "B"

Java stack information for the threads listed above:
===================================================
"B":
        at com.song.test.DeadLockDemo.lambda$main$1(DeadLockDemo.java:33)
        - waiting to lock <0x00000000d64018e0> (a java.lang.Object)
        - locked <0x00000000d64018f0> (a java.lang.Object)
        at com.song.test.DeadLockDemo$$Lambda$2/1324119927.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
"A":
        at com.song.test.DeadLockDemo.lambda$main$0(DeadLockDemo.java:20)
        - waiting to lock <0x00000000d64018f0> (a java.lang.Object)
        - locked <0x00000000d64018e0> (a java.lang.Object)
        at com.song.test.DeadLockDemo$$Lambda$1/295530567.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.
```

- 图形化：使用jconsole工具
![在这里插入图片描述](https://img-blog.csdnimg.cn/7144028f1c2a470daca6e9eb02298b4c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
点击检测死锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/f42dee5817c2440aba5de9256ebbd0a4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
### 1.7、写锁(独占锁)/读锁(共享锁)/读写锁(读共享/写独占)
#### 1.7.1、概要
- 独占锁（写锁）：指该锁一次只能被一个线程所持有，对ReentrantLock 和 Synchronized 而言都是独占锁
- 共享锁：指该锁可被多个线程所持有，对ReentrantReadWriteLock其读锁是共享，其写锁是独占

#### 1.7.2、读写锁
『读写锁ReentrantReadWriteLock』并不是真正意义上的读写分离，它只允许读读共存，而读写和写写依然是互斥的，一个ReentrantReadWriteLock同时只能存在一个写锁但是可以存在多个读锁，但不能同时存在写锁和读锁，也即一个资源可以被多个读操作访问或一个写操作访问，但两者不能同时进行，适合在读多写少情境，读写锁才具有较高的性能体现。

```java
class MyResource
{
    Map<String,String> map = new HashMap<>();
    //=====ReentrantLock 等价于 =====synchronized
    Lock lock = new ReentrantLock();
    //=====ReentrantReadWriteLock 一体两面，读写互斥，读读共享
    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void write(String key,String value)
    {
        rwLock.writeLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"---正在写入");
            map.put(key,value);
            //暂停毫秒
            try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"---完成写入");
        }finally {
            rwLock.writeLock().unlock();
        }
    }
    public void read(String key)
    {
        rwLock.readLock().lock();
        try
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"---正在读取");
            String result = map.get(key);
            //后续开启注释修改为2000，演示一体两面，读写互斥，读读共享，读没有完成时候写锁无法获得
            //try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"---完成读取result："+result);
        }finally {
            rwLock.readLock().unlock();
        }
    }
}

public class ReentrantReadWriteLockDemo
{
    public static void main(String[] args)
    {
        MyResource myResource = new MyResource();

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.write(finalI +"", finalI +"");
            },String.valueOf(i)).start();
        }

        for (int i = 1; i <=10; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.read(finalI +"");
            },String.valueOf(i)).start();
        }

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        //读全部over才可以继续写
        for (int i = 1; i <=3; i++) {
            int finalI = i;
            new Thread(() -> {
                myResource.write(finalI +"", finalI +"");
            },"newWriteThread==="+String.valueOf(i)).start();
        }
    }
}
```
总结：
- 写锁的时候是独占，也就是说，写写不能共存，写读不能共存，写锁释放之后，才能进行读锁操作
- 读锁的时候是共享，也就是说，读读是可以共存的，但是读写的时候不能共存，读锁释放之后，才能进行写锁操作
 
#### 1.7.3、 ReentrantReadWriteLock的锁降级
 从写锁→读锁，ReentrantReadWriteLock可以降级的
 
**ReentrantReadWriteLock锁降级的触发时机：**
- 遵循获取写锁→再获取读锁→再释放写锁的次序，写锁能够降级成为读锁，也就是说如果一个线程占有了写锁，在不释放写锁的情况下，它还能占有读锁，即写锁降级为读锁。

**ReentrantReadWriteLock锁降级的目的以及应用场景：**
- 锁降级是为了让当前线程感知到数据的变化，目的是保证数据可见性，例如说多线程的环境下，一个写线程获取到锁，更改了一个变量数据，希望其他的读线程能够立即感知到数据的变化，并且阻塞其他的写线程进行数据的更改，更改数据的写线程释放锁之后，其他的写线程才能进行数据的更改，其原理是，当一个线程先获取到写锁，在不释放写锁的前提下，再获取读锁，会触发锁降级，将写锁降级为读锁，而在读锁的状态下，读线程可以共享，写线程必须在读线程释放读锁之后，才能进行写操作。


![在这里插入图片描述](https://img-blog.csdnimg.cn/99f4b50d08fd495980c036566c946b42.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
```java
/**
 * 锁降级：遵循获取写锁→再获取读锁→再释放写锁的次序，写锁能够降级成为读锁。
 * 如果一个线程占有了写锁，在不释放写锁的情况下，它还能占有读锁，即写锁降级为读锁。
 */
public class LockDownGradingDemo
{
    public static void main(String[] args)
    {
        ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
        ReentrantReadWriteLock.ReadLock readLock = readWriteLock.readLock();
        ReentrantReadWriteLock.WriteLock writeLock = readWriteLock.writeLock();
        
        writeLock.lock();
        System.out.println("-------正在写入");
        readLock.lock();
        System.out.println("-------正在读取");
        writeLock.unlock();
    }
}
```
线程获取读锁是不能直接升级为写入锁的。在ReentrantReadWriteLock中，当读锁被使用时，如果有线程尝试获取写锁，该写线程会被阻塞，所以，需要释放所有读锁，才可获取写锁，
![在这里插入图片描述](https://img-blog.csdnimg.cn/e1e86b75758444c4998b305dc9dd3b50.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.8、自旋锁SpinLock
自旋锁（spinlock）：是指当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环。
自旋锁其实是基于CAS的思想，它包含三个操作数——内存位置、预期原值及更新值，在执行CAS操作的时候，将内存位置的值与预期原值比较：如果相匹配，那么处理器会自动将该位置值更新为新值，如果不匹配，处理器不做任何操作，多个线程同时执行CAS操作只有一个会成功。[CAS详解](https://editor.csdn.net/md?not_checkout=1&articleId=123938797)

```java
/**
 * 题目：实现一个自旋锁
 * 自旋锁好处：循环比较获取没有类似wait的阻塞。
 *
 * 通过CAS操作完成自旋锁，A线程先进来调用myLock方法自己持有锁5秒钟，B随后进来后发现
 * 当前有线程持有锁，不是null，所以只能通过自旋等待，直到A释放锁后B随后抢到。
 */
public class SpinLockDemo
{
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void myLock()
    {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName()+"\t come in");
        while(!atomicReference.compareAndSet(null,thread))
        {

        }
    }

    public void myUnLock()
    {
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread,null);
        System.out.println(Thread.currentThread().getName()+"\t myUnLock over");
    }

    public static void main(String[] args)
    {
        SpinLockDemo spinLockDemo = new SpinLockDemo();

        new Thread(() -> {
            spinLockDemo.myLock();
            try { TimeUnit.SECONDS.sleep( 5 ); } catch (InterruptedException e) { e.printStackTrace(); }
            spinLockDemo.myUnLock();
        },"A").start();
        //暂停一会儿线程，保证A线程先于B线程启动并完成
        try { TimeUnit.SECONDS.sleep( 1 ); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            spinLockDemo.myLock();
            spinLockDemo.myUnLock();
        },"B").start();

    }
}

t1 come in
.....一秒后.....
t2 come in
.....五秒后.....
t1 invoked myUnlock()
t2 invoked myUnlock() 
```

优点：循环比较获取直到成功为止，没有类似于wait的阻塞，因为自旋锁不会使线程状态发生切换，一直处于用户态，即线程一直都是active的；不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快。

缺点：当不断自旋的线程越来越多的时候，会因为执行while循环不断的消耗CPU资源

### 1.9、无锁→独占锁→读写锁→邮戳锁  
- 无锁情况：读写操作都是共享的，实际读可以共享，写操作不能共享的，会造成数据不一致。
- 独占锁：指该锁一次只能被一个线程锁持有，只允许该线程写或者读，在未释放锁之前，不允许其他线程进行读和写，ReentrantLock和sync而言都是独占锁。
- 读写锁：一个资源能够被多个读线程访问，或者被一个写线程访问，但是不能同时存在读写线程。ReentrantReadWriteLock是读写锁。
- 邮戳锁：为了解决读写锁的锁饥饿的问题而诞生，读的过程中也允许获取写锁介入，读写锁获取读锁之后，其他线程尝试获取写锁的时候会被阻塞，StampedLock采取乐观获取锁后，其他线程尝试获取写锁时不会被阻塞，在获取乐观锁后会返回一个邮戳（类型版本号），执行完业务释放锁的时候，还需要对该邮戳进行校验，如果一致，直接返回，如果不一致，会以读写锁中读锁的方式（获取读锁后，必须要释放读锁之后，才能获取写锁）去获取结果返回。

#### 1.9.1、无锁→独占锁→读写锁→邮戳锁的优缺点
- 无锁：
	1. 优点：读取速度快
	2. 缺点：多线程不能保证数据的读写一致性

- 独占锁：
	1. 优点：多线程环境下能够保证数据的读写一致性
	2. 缺点：读锁和写锁都只能单独获取，读取速度较慢

- 读写锁：
	1. 优点：多线程环境下能够保证数据的读写一致性，读锁可以共享，写锁独占，效率比独占锁速度要快
	2. 优点：①容易出现锁饥饿问题，当读锁过多的时候，写的线程可能一直抢不到锁。②读线程获取锁之后，如果没有释放锁，写线程不可以获得锁，必须读完后，才能有机会写线程才能获取写锁。

- 邮戳锁 ：
	1. 优点：读线程获取锁之后，如果没有释放锁，写线程是可以获得锁，但是邮戳锁在获取读锁的时候会返回一个邮戳（类型版本号），执行完业务释放锁的时候，还需要对该邮戳进行校验，如果一致，直接返回，如果不一致，会以读写锁中读锁的方式（获取读锁后，必须要释放读锁之后，才能获取写锁）去获取结果返回，效率比读写锁要快
	2.  优点：①StampedLock是不可重入的，容易发生死锁问题。②StampedLock的悲观读锁，写锁不支持条件变量③如果线程阻塞在StampedLock的readLock()或writeLock()上时，此时调用该阻塞线程的interrupt()会导致CPU飙升


![在这里插入图片描述](https://img-blog.csdnimg.cn/684078fa96be4631bfe1babb19bb1c6b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.9.2、邮戳锁  
邮戳锁也叫票据锁，StampedLock是JDK1.8中新增的一个读写锁，是对读写锁ReentrantReadWriteLock的优化。
stamp（戳记，long类型）：代表了锁的状态，当stamp返回零时，表示线程获取锁失败，并且，当释放锁或者转换锁的时候，都要传入最初获取的stamp值。

#### 1.9.3、邮戳锁由来以及解决问题的方式
邮戳锁是为了解决由锁饥饿问题引出来的。

##### 1.9.3.1、锁饥饿问题
锁饥饿问题：ReentrantReadWriteLock实现了读写分离，但是一旦读操作比较多的时候，想要获取写锁就变得比较困难了，假如当前1000个线程，999个读，1个写，有可能999个读取线程长时间抢到了锁，那1个写线程就悲剧了，因为当前有可能会一直存在读锁，而无法获得写锁，根本没机会写。

##### 1.9.3.2、如何缓解锁饥饿问题？
- 使用“公平”策略可以一定程度上缓解这个问题，new ReentrantReadWriteLock(true)，但是“公平”策略是以牺牲系统吞吐量为代价的
- 使用StampedLock类的乐观读锁

##### 1.9.3.3、StampedLock的特点：
- 所有获取锁的方法，都返回一个邮戳（Stamp），Stamp为零表示获取失败，其余都表示成功；
- 所有释放锁的方法，都需要一个邮戳（Stamp），这个Stamp必须是和成功获取锁时得到的Stamp一致；
- StampedLock是不可重入的，危险(如果一个线程已经持有了写锁，再去获取写锁的话就会造成死锁)

##### 1.9.3.4、StampedLock有三种访问模式
- Reading（读模式）：功能和ReentrantReadWriteLock的读锁类似。
- Writing（写模式）：功能和ReentrantReadWriteLock的写锁类似。
- Optimistic reading（乐观读模式）：无锁机制，类似于数据库中的乐观锁，支持读写并发，很乐观认为读取时没人修改，假如被修改再实现升级为悲观读模式。

```java
public class StampedLockDemo
{
    static int number = 37;
    static StampedLock stampedLock = new StampedLock();

    public void write()
    {
        long stamp = stampedLock.writeLock();
        System.out.println(Thread.currentThread().getName()+"\t"+"=====写线程准备修改");
        try
        {
            number = number + 13;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            stampedLock.unlockWrite(stamp);
        }
        System.out.println(Thread.currentThread().getName()+"\t"+"=====写线程结束修改");
    }

    //悲观读
    public void read()
    {
        long stamp = stampedLock.readLock();
        System.out.println(Thread.currentThread().getName()+"\t come in readlock block,4 seconds continue...");
        //暂停几秒钟线程
        for (int i = 0; i <4 ; i++) {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t 正在读取中......");
        }
        try
        {
            int result = number;
            System.out.println(Thread.currentThread().getName()+"\t"+" 获得成员变量值result：" + result);
            System.out.println("写线程没有修改值，因为 stampedLock.readLock()读的时候，不可以写，读写互斥");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            stampedLock.unlockRead(stamp);
        }
    }

    //乐观读
    public void tryOptimisticRead()
    {
        long stamp = stampedLock.tryOptimisticRead();
        int result = number;
        //间隔4秒钟，我们很乐观的认为没有其他线程修改过number值，实际靠判断。
        System.out.println("4秒前stampedLock.validate值(true无修改，false有修改)"+"\t"+stampedLock.validate(stamp));
        for (int i = 1; i <=4 ; i++) {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t 正在读取中......"+i+
                    "秒后stampedLock.validate值(true无修改，false有修改)"+"\t"
                    +stampedLock.validate(stamp));
        }
        if(!stampedLock.validate(stamp)) {
            System.out.println("有人动过--------存在写操作！");
            stamp = stampedLock.readLock();
            try {
                System.out.println("从乐观读 升级为 悲观读");
                result = number;
                System.out.println("重新悲观读锁通过获取到的成员变量值result：" + result);
            }catch (Exception e){
                e.printStackTrace();
            }finally {
                stampedLock.unlockRead(stamp);
            }
        }
        System.out.println(Thread.currentThread().getName()+"\t finally value: "+result);
    }

    public static void main(String[] args)
    {
        StampedLockDemo resource = new StampedLockDemo();

        new Thread(() -> {
            resource.read();
            //resource.tryOptimisticRead();
        },"readThread").start();

        // 2秒钟时乐观读失败，6秒钟乐观读取成功resource.tryOptimisticRead();，修改切换演示
        //try { TimeUnit.SECONDS.sleep(6); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            resource.write();
        },"writeThread").start();
    }
}
```

##### 1.9.3.5、StampedLock 的缺点：

（1）StampedLock 不支持重入，没有Re开头。
（2）StampedLock 的悲观读锁和写锁都不支持条件变量（Condition），这个也需要注意。
（3）使用 StampedLock一定不要调用中断操作，即不要调用interrupt() 方法。

##### 1.9.3.6、如果线程阻塞在StampedLock的readLock()或writeLock()上时，此时调用该阻塞线程的interrupt()会导致CPU飙升？

```java
public class StampedLockDemo {
    public static void main(String[] args) throws InterruptedException {
        final StampedLock lock = new StampedLock();
        Thread T1 = new Thread(() -> {
            // 获取写锁
            lock.writeLock();
            // 永远阻塞在此处，不释放写锁
            LockSupport.park();
        });
        T1.start();
        // 保证T1获取写锁
        Thread.sleep(100);
        Thread T2 = new Thread(() ->
                //阻塞在悲观读锁
                lock.readLock()
        );
        T2.start();
        // 保证T2阻塞在读锁
        Thread.sleep(100);
        //中断线程T2
        //会导致线程T2所在CPU飙升
        T2.interrupt();
        T2.join();
    }
}
```
因为中断t2后，StampedLock内太多的自旋引起的，如果没有中断，那么阻塞在readLock()上的线程在经过几次自旋后，会进入park()等待，一旦进入park()等待，就不会占用CPU了。但是park()这个函数有一个特点，就是一旦线程被中断，park()就会立即返回，也不抛点异常，本来在锁准备好的时候，unpark()的线程的，但是现在锁没好，直接中断了，park()也返回了，造成锁没好，所以就又去自旋了，转着转着，又转到了park()函数，但悲催的是，线程的中断标记一直打开着，park()就阻塞不住了，于是乎，下一个自旋又开始了，没完没了的自旋停不下来了，所以CPU就爆满了。

如果需要支持中断功能，一定使用可中断的悲观读锁readLockInterruptibly() 和写锁 writeLockInterruptibly()。






### 1.10、无锁→偏向锁→轻量锁→重量锁
#### 1.10.1、对象的布局以及Mutex Lock 

对象布局分为Java对象布局和数组对象布局，数组对象布局是特殊的ava对象布局，组成结构在对象头的下面增加了数组长度的模块。
对象在堆内存中的存储布局：对象头、实例数据、对齐填充（保证8个字节的倍数）。
对象头分为对象标记（MarkWord）和类元信息(又叫类型指针)，类元信息存储的是指向该对象指向它的类型元数据的指针。
- 对象标记（MarkWord）：默认存储对象的HashCode、分代年龄和锁标志位等信息。这些信息都是与对象自身定义无关的数据，所以MarkWord被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间MarkWord里存储的数据会随着锁标志位的变化而变化。

- 类元信息：类元信息存储的是指向该对象指向它的类型元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。


![在这里插入图片描述](https://img-blog.csdnimg.cn/7edfab242b164a90ba48f84d075e71ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)


![在这里插入图片描述](https://img-blog.csdnimg.cn/6a4007799e05495b9546ced345e8f9e7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/498ef255940a4075aee9d81a1264cb2f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- hash： 保存对象的哈希码
- age： 保存对象的分代年龄
- biased_lock： 偏向锁标识位
- lock： 锁状态标识位
- JavaThread* ：保存持有偏向锁的线程ID
- epoch： 保存偏向时间戳




##### 1.10.1.1、Mutex Lock 
Monitor锁是在jvm底层实现的，底层代码是c++，本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的转换，状态转换需要耗费很多的处理器时间成本非常高。所以synchronized是Java语言中的一个重量级操作。 
 
##### 1.10.1.2、Monitor与java对象以及线程是如何关联 ？
1.如果一个java对象被某个线程锁住，则该java对象的MarkWord字段中LockWord指向monitor的起始地址
2.Monitor的Owner字段会存放拥有相关联对象锁的线程id
 Mutex Lock 的切换需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间。
 
#### 1.10.2、锁升级
synchronized用的锁是存在Java对象头里的Mark Word中，锁升级功能主要依赖MarkWord中锁标志位和释放偏向锁标志位
![在这里插入图片描述](https://img-blog.csdnimg.cn/b948a19b883b41b7af8ce1f4f200c8f8.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/498ef255940a4075aee9d81a1264cb2f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- hash： 保存对象的哈希码
- age： 保存对象的分代年龄
- biased_lock： 偏向锁标识位
- lock： 锁状态标识位
- JavaThread* ：保存持有偏向锁的线程ID
- epoch： 保存偏向时间戳

**多线程访问情况，有3种情况：**

- 只有一个线程来访问，有且唯一Only One
- 有2个线程A、B来交替访问
- 竞争激烈，多个线程来访问




##### 1.10.2.1、无锁
```java
public class MyObject
{
    public static void main(String[] args)
    {
        Object o = new Object();

        System.out.println("10进制hash码："+o.hashCode());
        System.out.println("16进制hash码："+Integer.toHexString(o.hashCode()));
        System.out.println("2进制hash码："+Integer.toBinaryString(o.hashCode()));

        System.out.println( ClassLayout.parseInstance(o).toPrintable());
    }
}
```
value值从后向前看
![在这里插入图片描述](https://img-blog.csdnimg.cn/fadff3a345d74f378345c0e94a075e36.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

##### 1.10.2.2、偏向锁
**什么是偏向锁**：当一段同步代码一直被同一个线程多次访问，由于只有一个线程时，那么该线程在后续访问时便会自动获得锁。多线程的情况下，锁不仅不存在多线程竞争，还存在锁由同一线程多次获得的情况，偏向锁就是在这种情况下出现的，它的出现是为了解决只有在一个线程执行同步时提高性能。

当使用synchronized的时候，锁总是被第一个占用他的线程拥有，这个线程就是锁的偏向线程，锁第一次被拥有的时候，会判断锁的markword中是否存储线程ID，如果不存在，则会以CAS操作替换线程ID，记录下偏向线程ID，这样偏向线程就一直持有着锁(后续这个线程进入和退出这段加了同步锁的代码块时，不需要再次加锁和释放锁，而是直接比较对象头里面是否存储了指向当前线程的偏向锁)。
- 如果相等表示偏向锁是偏向于当前线程的，就不需要再尝试获得锁了，直到竞争发生才释放锁，如果自始至终使用锁的线程只有一个，很明显偏向锁几乎没有额外开销，性能极高。
- 假如不一致意味着发生了竞争，锁已经不是总是偏向于同一个线程了，这时候可能需要升级变为轻量级锁，才能保证线程间公平竞争锁。偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程是不会主动释放偏向锁的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d832825f05dc4778831715532e39a18e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

如果一个synchronized方法被一个线程抢到了锁时，JVM使用CAS操作把线程指针ID记录到Mark Word当中，并修改标偏向状态位，将其所在的MarkWord中将偏向锁状态位0修改状态位1，表示当前线程就获得该锁，锁对象变成偏向锁（通过CAS修改对象头里的锁标志位），同时还会有占用前54位来存储当前线程指针作为标识，若该线程再次访问同一个synchronized方法时，该线程只需去对象头的MarkWord 中去判断一下是否有偏向锁指向本身的ID，无需再进入 Monitor 去竞争对象了。偏向锁的操作不用操作系统介入，不涉及用户到内核转换，如果自始至终使用锁的线程只有一个，偏向锁几乎没有额外开销，性能极高。

偏向锁JVM命令：

```java
java -XX:+PrintFlagsInitial |grep BiasedLock*
 
* 实际上偏向锁在JDK1.6之后是默认开启的，但是启动时间有延迟，
* 所以需要添加参数-XX:BiasedLockingStartupDelay=0，让其在程序启动时立刻启动。
*
* 开启偏向锁：
* -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
*
* 关闭偏向锁：关闭之后程序默认会直接进入------------------------------------------>>>>>>>>   轻量级锁状态。
* -XX:-UseBiasedLocking 
```

 

```java
public class MyObject
{
    public static void main(String[] args)
    {
        Object o = new Object();

        new Thread(() -> {
            synchronized (o){
                System.out.println(ClassLayout.parseInstance(o).toPrintable());
            }
        },"t1").start();
    }
} 

设置VM参数：-XX:BiasedLockingStartupDelay=0
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 f0 7c 1f (00000101 11110000 01111100 00011111) (528281605)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**偏向锁的撤销**

偏向锁使用一种等到竞争出现才释放锁的机制，只有当其他线程竞争锁时，持有偏向锁的原来线程才会被撤销，撤销需要等待全局安全点(该时间点上没有字节码正在执行)，同时检查持有偏向锁的线程是否还在执行： 
 
- 如果第一个线程正在执行synchronized方法(处于同步块)，它还没有执行完，其它线程来抢夺，而正在竞争的线程会尝试以CAS自旋操作去更新锁的对象头，如果更新失败，会等待到全局安全点（此时不会执行任何代码）撤销偏向锁，进行锁升级，升级为轻量级锁，此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，如果还没有结束，两个线程以CAS操作竞争获得该轻量级锁。
- 如果第一个线程执行完成synchronized方法(退出同步块)，刚好另外的一个线程进入竞争锁，则将当前锁的对象头设置成无锁状态，并撤销偏向锁的状态为，清空偏向锁的指针，重新进行偏向。
![在这里插入图片描述](https://img-blog.csdnimg.cn/840e286419da48ffb0e8e4f7e77ae70f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

##### 1.10.2.3、轻量锁

偏向锁使用一种等到竞争出现才释放锁的机制，只有当其他线程竞争锁时，持有偏向锁的原来线程才会被撤销，撤销需要等待全局安全点(该时间点上没有字节码正在执行)，同时检查持有偏向锁的线程是否还在执行： 
 
- 如果第一个线程正在执行synchronized方法(处于同步块)，它还没有执行完，其它线程来抢夺，而正在竞争的线程会尝试以CAS自旋操作去更新锁的对象头，如果更新失败，会等待到全局安全点（此时不会执行任何代码）撤销偏向锁，进行锁升级，升级为轻量级锁，此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，如果还没有结束，两个线程以CAS操作竞争获得该轻量级锁。
- 如果第一个线程执行完成synchronized方法(退出同步块)，刚好另外的一个线程进入竞争锁，则将当前锁的对象头设置成无锁状态，并撤销偏向锁的状态为，清空偏向锁的指针，重新进行偏向。

轻量级锁是为了在线程近乎交替执行同步块时提高性能，其本质就是自旋锁，在没有多线程竞争的前提下，通过CAS减少重量级锁使用操作系统互斥量产生的性能消耗，说白了先自旋再阻塞。

**升级为轻量级锁的时机**：当关闭偏向锁功能（-XX:-UseBiasedLocking），同时会有线程争抢锁，会直接进入轻量锁或多线程竞争偏向锁会导致偏向锁升级为轻量级锁。
 
假如线程A已经拿到锁，这时线程B又来抢该对象的锁，由于该对象的锁已经被线程A拿到，当前该锁已是偏向锁了。而线程B在争抢时发现对象头Mark Word中的线程ID不是线程B自己的线程ID(而是线程A)，那线程B就会进行CAS操作希望能获得锁。
此时线程B操作中有两种情况：
- 如果锁获取成功，直接替换Mark Word中的线程ID为B自己的ID(A → B)，重新偏向于其他线程(即将偏向锁交给其他线程，相当于当前线程"被"释放了锁)，该锁会保持偏向锁状态，A线程Over，B线程上位；
- 如果锁获取失败，则偏向锁升级为轻量级锁，此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，而正在竞争的线程B会进入自旋等待获得该轻量级锁。

**轻量级锁和偏向锁的区别**：
- 轻量级锁争夺锁失败时，会自旋尝试抢占锁，轻量级锁每次退出同步块都需要释放锁。
- 偏向锁是在竞争发生时才释放锁。

##### 1.10.2.4、重量锁
**轻量级锁升级为重量级锁的时机：**轻量级锁线程争抢，当争抢锁的线程自旋达到一定次数和程度，会进行锁升级为重量锁。java6之前，默认启用，默认情况下自旋的次数是10次或者自旋线程数超过cpu核数一半，Java6之后，轻量级锁升级为重量级锁自旋的次数是自适应的，因为自旋的次数不是固定不变的，而是根据同一个锁上一次自旋的时间以及拥有锁线程的状态来决定。比方说如果在同一个锁对象上，自选等待刚刚成功获得过锁，且持有锁的线程正在运行过程中的，那么虚拟机就会认为该线程自选就很可能再次获取到锁，所以会允许该线程自旋的时间长一些，如果对于某一个锁，该线程自旋很少成功获取到锁，那么该线程再次获取锁的时候，可能会直接省略掉自旋过程，避免浪费CPU资源。
##### 1.10.2.5、锁消除
锁消除和锁粗话（锁膨胀）主要是JIT(Just In Time Compiler,即时编译器)对锁做的优化。

锁削除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行削除，比方说定义了一个共享锁对象obj，但是在同步方法中又创建了一个相同的对象，导致每一个线程都会重新生成锁，所以程序运行的时候，JIT会无视它，当做synchronized(对象锁)不存在了，这就叫做锁消除。
```java
/**
 * 锁消除
 * 从JIT角度看相当于无视它，synchronized (o)不存在了,这个锁对象并没有被共用扩散到其它线程使用，
 * 极端的说就是根本没有加这个锁对象的底层机器码，消除了锁的使用
 */
public class LockClearUPDemo
{
    static Object objectLock = new Object();//正常的

    public void m1()
    {
        //锁消除,JIT会无视它，synchronized(对象锁)不存在了。不正常的
        Object o = new Object();

        synchronized (o)
        {
            System.out.println("-----hello LockClearUPDemo"+"\t"+o.hashCode()+"\t"+objectLock.hashCode());
        }
    }

    public static void main(String[] args)
    {
        LockClearUPDemo demo = new LockClearUPDemo();

        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                demo.m1();
            },String.valueOf(i)).start();
        }
    }
}
```

##### 1.10.2.6、锁粗话（锁膨胀）
锁粗化：通常情况下，为了保证多线程间的有效并发，会要求每个线程持有锁的时间尽可能短，但是大某些情况下，一个程序对同一个锁不间断、高频地请求、同步与释放，会消耗掉一定的系统资源，因为锁的请求、同步与释放本身会带来性能损耗，这样高频的锁请求就反而不利于系统性能的优化了，所以JIT编译器就会把前后相邻的这几个synchronized块合并成一个大块，粗加大范围，一次申请锁使用即可，避免次次的申请和释放锁，提升了性能。
```java
/**
 * 锁粗化
 * 假如方法中首尾相接，前后相邻的都是同一个锁对象，那JIT编译器就会把这几个synchronized块合并成一个大块，
 * 加粗加大范围，一次申请锁使用即可，避免次次的申请和释放锁，提升了性能
 */
public class LockBigDemo
{
    static Object objectLock = new Object();
    public static void main(String[] args)
    {
        new Thread(() -> {
            synchronized (objectLock) {
                System.out.println("11111");
            }
            synchronized (objectLock) {
                System.out.println("22222");
            }
            synchronized (objectLock) {
                System.out.println("33333");
            }
        },"a").start();

        new Thread(() -> {
            synchronized (objectLock) {
                System.out.println("44444");
            }
            synchronized (objectLock) {
                System.out.println("55555");
            }
            synchronized (objectLock) {
                System.out.println("66666");
            }
        },"b").start();

    }
}
```

##### 1.10.2.7、各种锁优缺点、synchronized锁升级和实现原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/ad20294265254ce09b08397b12f04afb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 synchronized锁升级过程总结：一句话，就是先自旋，不行再阻塞。实际上是把之前的悲观锁(重量级锁)变成在一定条件下使用偏向锁以及使用轻量级(自旋锁CAS)的形式
 
synchronized在修饰方法和代码块在字节码上实现方式有很大差异，但是内部实现还是基于对象头的MarkWord来实现的。
JDK1.6之前synchronized使用的是重量级锁，JDK1.6之后进行了优化，拥有了无锁->偏向锁->轻量级锁->重量级锁的升级过程，而不是无论什么情况都使用重量级锁。

- 偏向锁:适用于单线程适用的情况，在不存在锁竞争的时候进入同步方法/代码块则使用偏向锁。
- 轻量级锁：适用于竞争较不激烈的情况(这和乐观锁的使用范围类似)， 存在竞争时升级为轻量级锁，轻量级锁采用的是自旋锁，如果同步方法/代码块执行时间很短的话，采用轻量级锁虽然会占用cpu资源但是相对比使用重量级锁还是更高效。
- 重量级锁：适用于竞争激烈的情况，如果同步方法/代码块执行时间很长，那么使用轻量级锁自旋带来的性能消耗就比使用重量级锁更严重，这时候就需要升级为重量级锁。


## 2、LockSupport与线程中断
### 2.1、线程中断
#### 2.1.1、什么是线程中断？
中断只是一种协作机制，Java没有给中断增加任何语法，中断的过程完全需要程序员自己实现。若要中断一个线程，你需要手动调用该线程的interrupt方法，该方法也仅仅是将线程对象的中断标识设成true；接着你需要自己写代码不断地检测当前线程的标识位，如果为true，表示别的线程要求这条线程中断。

**中断的过程程序员如何自己实现？**

每个线程对象中都有一个标识，用于表示线程是否被中断；该标识位为true表示中断，为false表示未中断；通过调用线程对象的interrupt方法将该线程的标识位设为true；可以在别的线程中或者自己的线程中对线程中断状态做判断，执行相对应的业务逻辑。

#### 2.1.2、Thread.stop, Thread.suspend, Thread.resume被弃用的原因？
- Thread.stop：终止线程，调用stop方法无论run()中的逻辑是否执行完，都会释放CPU资源，释放锁资源。这会导致线程不安全。
- Thread.suspend：暂停线程，在调用该方法暂停线程的时候，线程由running状态变成blocked，需要等待resume方法将其重新变成runnable，而线程由running状态变成blocked时，只释放了CPU资源，没有释放锁资源，可能出现死锁。
- Thread.resume：重启线程。调用resume方法将阻塞状态的线程，使其重新变成runnable。

**Thread.stop停止使用的原因？**

因为线程调用stop()会产生线程安全问题，导致数据不一致，当调用stop()方法时会即刻停止run()方法中剩余的全部工作，包括在catch或finally语句中，同时会立即释放该线程所持有的锁，导致数据得不到同步的处理，出现数据不一致的问题。例如当有两个初始变量a,b,初始值都等于0，方法体中有两步操作，a++, sleep(),b++,线程正常执行完毕应该是a=1,b=1,但是在线程睡眠的时候，调用stop方法的时候，会变成a=1,b=0,造成了数据不一致，出现了线程安全的问题。

**Thread.suspend, Thread.resume停止使用的原因？**

在调用suspend()方法暂停线程的时候，线程由running状态变成blocked，需要等待resume方法将其重新变成runnable，而线程由running状态变成blocked时，只释放了CPU资源，没有释放锁资源，可能出现死锁现象。比如：线程A和线程B抢占同一个锁1，线程A拿着锁1被suspend()方法调用，进入了blocked状态，等待线程B需要获取到锁1，然后调用resume()方法，将线程A重启，重新进入runnable状态。但是线程B一直在lock pool中等待锁1,线程B要拿到锁1才能running去执行resumeA，所以就造成了死锁现象。

而且一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。Thread.stop, Thread.suspend, Thread.resume 是过时的，都已经被废弃了。
 
#### 2.1.3、如何停止、中断一个运行中的线程？
- 通过一个volatile变量实现
```java
public class InterruptDemo
{
private static volatile boolean isStop = false;
public static void main(String[] args)
{
    new Thread(() -> {
        while(true)
        {
            if(isStop)
            {
                System.out.println(Thread.currentThread().getName()+"线程------isStop = true,自己退出了");
                break;
            }
            System.out.println("-------hello interrupt");
        }
    },"t1").start();

    //暂停几秒钟线程
    try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
    isStop = true;
	}
}
```
- 通过AtomicBoolean

```java
public class StopThreadDemo
{
    private final static AtomicBoolean atomicBoolean = new AtomicBoolean(true);

    public static void main(String[] args)
    {
        Thread t1 = new Thread(() -> {
            while(atomicBoolean.get())
            {
                try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println("-----hello");
            }
        }, "t1");
        t1.start();

        try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }

        atomicBoolean.set(false);
    }
}
```
- 通过Thread类自带的中断api方法实现

 1. public void interrupt()实例方法，实例方法interrupt()仅仅是设置线程的中断状态为true，不会停止线程。
 2. public static boolean interrupted()静态方法，Thread.interrupted();  判断线程是否被中断，并清除当前中断状态这个方法做了两件事：
	- 返回当前线程的中断状态
	- 将当前线程的中断状态设为false 这个方法有点不好理解，因为连续调用两次的结果可能不一样。
 3. public boolean isInterrupted()实例方法，判断当前线程是否被中断（通过检查中断标志位）

```java
public class InterruptDemo
{
    public static void main(String[] args)
    {
        Thread t1 = new Thread(() -> {
            while(true)
            {
                if(Thread.currentThread().isInterrupted())
                {
                    System.out.println("-----t1 线程被中断了，break，程序结束");
                    break;
                }
                System.out.println("-----hello");
            }
        }, "t1");
        t1.start();

        System.out.println("**************"+t1.isInterrupted());
        //暂停5毫秒
        try { TimeUnit.MILLISECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
        t1.interrupt();
        System.out.println("**************"+t1.isInterrupted());
    }
}
```
#### 2.1.4、interrupt()、 interrupted()、isInterrupted()的区别？
- public void interrupt()实例方法，调用interrupt()方法仅仅是在当前线程中打了一个停止的标记，并不是真正立刻停止线程。
- public boolean isInterrupted()实例方法，获取中断标志位的当前值是什么，判断当前线程是否被中断（通过检查中断标志位），默认是false

```java
public class InterruptDemo2
{
    public static void main(String[] args) throws InterruptedException
    {
        Thread t1 = new Thread(() -> {
            for (int i=0;i<300;i++) {
                System.out.println("-------"+i);
            }
            System.out.println("after t1.interrupt()--第2次---: "+Thread.currentThread().isInterrupted());
        },"t1");
        t1.start();

        System.out.println("before t1.interrupt()----: "+t1.isInterrupted());
        //实例方法interrupt()仅仅是设置线程的中断状态位设置为true，不会停止线程
        t1.interrupt();
        //活动状态,t1线程还在执行中
        try { TimeUnit.MILLISECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("after t1.interrupt()--第1次---: "+t1.isInterrupted());
        //非活动状态,t1线程不在执行中，已经结束执行了。
        try { TimeUnit.MILLISECONDS.sleep(3000); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("after t1.interrupt()--第3次---: "+t1.isInterrupted());
    }
}
```

具体来说，当对一个线程，调用 interrupt() 时：
- 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。所以， interrupt() 并不能真正的中断线程，需要被调用的线程自己进行配合才行。
- 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），在别的线程中调用当前线程对象的interrupt方法，那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常，但是会将中断标志位重置，恢复成false。
 

```java
public class InterruptDemo
{
    public static void main(String[] args)
    {
        Thread t1 = new Thread(() -> {
            while(true)
            {
                if(Thread.currentThread().isInterrupted())
                {
                    System.out.println("-----t1 线程被中断了，break，程序结束");
                    break;
                }
                try {
                    // 线程处于被阻塞状态,调用线程的interrupt方法，会爆出中断异常，并将中断标志位重置，恢复成false。
                    TimeUnit.MILLISECONDS.sleep(500);
                } catch (InterruptedException e) {
                    // 解决方式是在抛出异常的时候，再次将线程打上中断的线程标志位
                    Thread.currentThread().interrupt();
                    e.printStackTrace();
                }
                System.out.println("-----hello");
            }
        }, "t1");
        t1.start();

        System.out.println("**************"+t1.isInterrupted());
        //暂停5毫秒
        try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
        t1.interrupt();
        System.out.println("**************"+t1.isInterrupted());
    }
```

- public static boolean interrupted()静态方法，Thread.interrupted();  判断线程是否被中断，并清除当前中断状态这个方法做了两件事：
1. 返回当前线程的中断状态
2. 将当前线程的中断状态设为false 这个方法有点不好理解，因为连续调用两次的结果可能不一样。

```java
/**
 * 作用是测试当前线程是否被中断（检查中断标志），返回一个boolean并清除中断状态，
 * 第二次再调用时中断状态已经被清除，将返回一个false。
 */
public class InterruptDemo
{

    public static void main(String[] args) throws InterruptedException
    {
        System.out.println(Thread.currentThread().getName()+"---"+Thread.interrupted());
        System.out.println(Thread.currentThread().getName()+"---"+Thread.interrupted());
        System.out.println("111111");
        Thread.currentThread().interrupt();
        System.out.println("222222");
        System.out.println(Thread.currentThread().getName()+"---"+Thread.interrupted());
        System.out.println(Thread.currentThread().getName()+"---"+Thread.interrupted());
    }
}

```

**线程中断相关的方法总结：**
- interrupt()方法是一个实例方法，它通知目标线程中断，也就是设置目标线程的中断标志位为true，中断标志位表示当前线程已经被中断了。
- isInterrupted()方法也是一个实例方法，它判断当前线程是否被中断（通过检查中断标志位）并获取中断标志。
- Thread类的静态方法interrupted()，返回当前线程的中断状态(boolean类型)且将当前线程的中断状态设为false，此方法调用之后会清除当前线程的中断标志位的状态（将中断标志置为false了），返回当前值并清零置false。



### 2.2、LockSupport
#### 2.2.1、LockSupport是什么?
LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。
#### 2.2.2、3种让线程等待和唤醒的方法
- 方式1：使用Object中的wait()方法让线程等待，使用Object中的notify()方法唤醒线程
- 方式2：使用JUC包中Condition的await()方法让线程等待，使用signal()方法唤醒线程
- 方式3：LockSupport类中的park() 和 unpark() 可以阻塞当前线程以及唤醒指定被阻塞的线程
 
**1. Object类中的wait和notify方法实现线程等待和唤醒**

```java
/**
 *
 * 要求：t1线程等待3秒钟，3秒钟后t2线程唤醒t1线程继续工作
 *
 * 1 正常程序演示
 *
 * 以下异常情况：
 * 2 wait方法和notify方法，两个都去掉同步代码块后看运行效果
 *   2.1 异常情况
 *   Exception in thread "t1" java.lang.IllegalMonitorStateException at java.lang.Object.wait(Native Method)
 *   Exception in thread "t2" java.lang.IllegalMonitorStateException at java.lang.Object.notify(Native Method)
 *   2.2 结论
 *   Object类中的wait、notify、notifyAll用于线程等待和唤醒的方法，都必须在synchronized内部执行（必须用到关键字synchronized）。
 *
 * 3 将notify放在wait方法前面
 *   3.1 程序一直无法结束
 *   3.2 结论
 *   先wait后notify、notifyall方法，等待中的线程才会被唤醒，否则无法唤醒
 */
public class LockSupportDemo
{

    public static void main(String[] args)//main方法，主线程一切程序入口
    {
        Object objectLock = new Object(); //同一把锁，类似资源类

        new Thread(() -> {
            synchronized (objectLock) {
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName()+"\t"+"被唤醒了");
        },"t1").start();

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(3L); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            synchronized (objectLock) {
                objectLock.notify();
            }

            //objectLock.notify();

           synchronized (objectLock) {
                try {
                    objectLock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"t2").start();
    }
}
```
**结论：**
 Object类中的wait、notify、notifyAll用于线程等待和唤醒的方法，都必须在synchronized内部执行（必须用到关键字synchronized），wait和notify方法必须要在同步块或者同步方法里面，且成对出现使用，而且必须要先wait后notify才能正常使用，将notify放在wait方法前面
，程序将无法唤醒。

**2. Condition接口中的await后signal方法实现线程的等待和唤醒**

```java
/*
 * 异常：
 * condition.await();和condition.signal();都触发了IllegalMonitorStateException异常
 * 原因：调用condition中线程等待和唤醒的方法的前提是，要在lock和unlock方法中,要有锁才能调用
 * 
 */
public class LockSupportDemo2
{
    public static void main(String[] args)
    {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        new Thread(() -> {
            lock.lock();
            try
            {
                System.out.println(Thread.currentThread().getName()+"\t"+"start");
                condition.await();
                System.out.println(Thread.currentThread().getName()+"\t"+"被唤醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        },"t1").start();

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(3L); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            lock.lock();
            try
            {
                condition.signal();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
            System.out.println(Thread.currentThread().getName()+"\t"+"通知了");
        },"t2").start();

    }
}
```
**总结：**
Condition接口中的await后signal方法实现线程的等待和唤醒，线程先要获得并持有锁，必须在锁块(synchronized或lock)中，必须要先等待await后唤醒signal，线程才能够被唤醒。

**3.LockSupport类中的park等待和unpark唤醒** 
- LockSupport是用来创建锁和其他同步类的基本线程阻塞原语，通过park()和unpark(thread)方法来实现阻塞和唤醒线程的操作。

LockSupport类使用了一种名为Permit（许可）的概念来做到阻塞和唤醒线程的功能， 每个线程都有一个许可(permit)，permit只有两个值1和零，默认是零。可以把许可看成是一种(0,1)信号量（Semaphore），但与 Semaphore 不同的是，许可的累加上限是1。

**主要方法：**

-  阻塞：park() /park(Object blocker) ，阻塞当前线程/阻塞传入的具体线程。

 当调用LockSupport.park()时，permit默认是零，所以一开始调用park()方法，会消耗一个permit，但是因为当前的permit为0，所以当前线程就会阻塞，直到别的线程将当前线程的permit设置为1时，park方法消耗到了permit，线程就会被唤醒，停止阻塞，park方法消耗完permit之后，会将permit再次设置为零并返回。
- 唤醒：unpark(Thread thread) ，唤醒处于阻塞状态的指定线程。

当调用LockSupport.unpark(thread)方法后，就会将thread线程的许可permit设置成1(注意多次调用unpark方法，不会累加，permit值还是1，如果之前该线程正在处于阻塞状态，会自动唤醒thread线程，即之前阻塞中的LockSupport.park()方法会立即返回。

```java
//正常使用+不需要锁块
public class LockSupportDemo3
{
    public static void main(String[] args)
    {
        //正常使用+不需要锁块
	Thread t1 = new Thread(() -> {
	    System.out.println(Thread.currentThread().getName()+" "+"1111111111111");
	    LockSupport.park();
	    System.out.println(Thread.currentThread().getName()+" "+"2222222222222------end被唤醒");
	},"t1");
	t1.start();

	//暂停几秒钟线程
	try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
	
	LockSupport.unpark(t1);
	System.out.println(Thread.currentThread().getName()+"   -----LockSupport.unparrk() invoked over");
	    }
	}
```

```java
//先唤醒后等待，LockSupport照样支持
public class T1
{
    public static void main(String[] args)
    {
        Thread t1 = new Thread(() -> {
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+System.currentTimeMillis());
            LockSupport.park();
            System.out.println(Thread.currentThread().getName()+"\t"+System.currentTimeMillis()+"---被叫醒");
        },"t1");
        t1.start();

        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        LockSupport.unpark(t1);
        System.out.println(Thread.currentThread().getName()+"\t"+System.currentTimeMillis()+"---unpark over");
    }
}
```
**总结：**

LockSupport类中的park等待和unpark唤醒，正常使用+不需要锁块，也支持先唤醒后等待，但是需要成双成对出现，因为许可的累加上限是1，不一致可能会导致线程阻塞。另外线程在park状态，调用interrupt方法也会打断阻塞，将线程进行唤醒，但是一般不这样使用。


