### 1、为什么多线程极其重要？
​	多线程变得极其重要的原因从软硬件两个方面来说，首先硬件方面：主要是摩尔定律失效，它是由英特尔创始人之一Gordon Moore(戈登·摩尔)提出来的,其内容是当价格不变时，将每隔18个月，性能也将提升一倍（当价格不变时，集成电路上可容纳的元器件的数目约每隔18-24个月便会增加一倍，性能也将提升一倍，换言之，每一美元所能买到的电脑性能，将每隔18-24个月翻一倍以上，这一定律揭示了信息技术进步的速度），但是可是从2003年开始CPU主频已经不再翻倍，而是采用多核而不是更快的主频，在主频不再提高且核数在不断增加的情况下，要想让程序更快就要用到并行或并发编程。在软件方面，在现在的高并发系统中需要处理更多的异步和回调等生产需求。
### 2、进程与线程的关系
进程：是程序的⼀次执⾏，是系统进行资源分配和调度的一个独立单位，每⼀个进程都有它⾃⼰的内存空间和系统资源，比方说一个应用程序。
线程：线程是进程中的一个实体，作为系统调度和分派的基本单位，在同⼀个进程内⼜可以执⾏多个任务，⽽这每⼀个任务我们就可以看做是⼀个线程。
管程：其实就是值Monitor对象。Monitor其实是一种同步机制，他的义务是保证（同一时间）只有一个线程可以访问被保护的数据和代码，也就是常说的“锁”，JVM中同步是基于进入和退出监视器对象(Monitor,管程对象)来实现的，每个对象实例都会有一个Monitor对象，Monitor对象会和Java对象一同创建并销毁，它底层是由C++语言来实现的。

```java
 Object o = new Object();
new Thread(() -> {
 //o 就是管程对象（Monitor）
    synchronized (o)
    {
	...
    }
},"t1").start();
```

**在周志明老师的JVM第三版是这样描述的：**
	Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程(Monitor，更常见的是直接将它称为“锁”）来实现的。
- 方法级的同步是隐式的，无须通过字节码指令来控制，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池中的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否被声明为同步方法。
- 当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。
- 在方法执行期间，执行线程持有了管程，其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法边界之外时自动释放。
- 同步一段指令集序列通常是由Java语言中的synchronized语句块来表示的，Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义，正确实现synchronized关键字需要Javac编译器与Java虚拟机两者共同协作支持，



**进程和线程的区别？**
 - 一个线程只能属于一个进程，但是一个进程可以有多个线程，而且至少有一个线程。
 - 同一进程的所有线程共享该进程的所有资源。
 - 每一个线程都有自己独立的栈空间，共享一个堆空间。
 
 
**用户线程和守护线程？**
Java线程分为用户线程和守护线程，线程的daemon属性为true表示是守护线程，false表示是用户线程
- 用户线程：是系统的工作线程，它会完成这个程序需要完成的业务操作。
- 守护线程：是一种特殊的线程，在后台默默地完成一些系统性的服务，比如垃圾回收线程
当程序中所有用户线程执行完毕之后，意味着程序需要完成的业务操作已经结束了，当系统只剩下守护进程的时候，不管守护线程是否结java虚拟机会自动退出。

### 3、多线程的实现方式
 - 继承 Thread类，重写run()方法
 - 实现 Runable接口，重写run()方法，然后使用Thread类来包装
 - 实现 Callable接口，重写call()方法，然后包装成FutureTask, 再然后包装成Thread
 - 线程池创建

三种方式比较：

- Thread: 继承方式, 不建议使用, 因为Java是单继承的，继承了Thread就没办法继承其它类了，不够灵活
- Runnable: 实现接口，比Thread类更加灵活，没有单继承的限制
- Callable: Thread和Runnable都是重写的run()方法并且没有返回值，Callable是重写的call()方法并且有返回值并可以借助FutureTask类来判断线程是否已经执行完毕或者取消线程执行

总结：当线程不需要返回值时使用Runnable，需要返回值时就使用Callable，一般情况下不直接把线程体代码放到Thread类中，一般通过Thread类来启动线程，Thread类是实现Runnable接口，Callable封装成FutureTask，FutureTask实现RunnableFuture，RunnableFuture继承Runnable，所以Callable也算是一种Runnable，所以三种实现方式本质上都是Runnable实现。


### 4、线程的状态
- 创建（NEW）状态: 准备好了一个多线程的对象，即执行了new Thread(); 创建完成后就需要为线程分配内存。
- 就绪/可运行（READY）状态: 线程调用了start()方法, 此时该状态位于可运行线程池中，等待被线程调度选中，获取cpu 的时间片。
- 运行（RUNNING）状态:	可运行状态的线程获取到cpu的时间片，执行run()方法。
- 阻塞（BLOCKED）状态：表示线程阻塞于锁，线程阻塞在进入synchronized关键字修饰的方法或代码块(获取锁)时的状态。
- 等待（WAITING）状态：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断），处于这种状态的线程不会被分配CPU执行时间，它们要等待被显式地唤醒，否则会处于无限期等待的状态。
- 超时等待（TIMED_WAITING）：该状态不同于WAITING，处于这种状态的线程不会被分配CPU执行时间，但是它可以在指定的时间后自行返回。
- 终止（TERMINATED）状态: 线程销毁(正常执行完毕、发生异常或者被打断interrupt()都会导致线程终止)，死亡的线程不可再次复生。
	
![在这里插入图片描述](https://img-blog.csdnimg.cn/5c511e98c79348fdb7f4cc87141e7fbb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**总结：**
	当创建一个新的线程时，调用了start()方法后,会进入就绪状态，该状态下的当前线程会进入可运行线程池中，等待被线程调度选中，获取cpu 的时间片，获取cpu 的时间片的当前线程会进入运行状态，执行run方法，run方法执行完毕或者异常中断会进入终止状态，在运行状态调用不带延时的wait方法或者join方法会进入等待状态，等待notify、notifyAll唤醒再次进入就绪状态，在运行状态调用带延时的wait方法或者join方法会进入延时等待状态，等待睡眠时间已满，再次进入就绪状态，在运行状态调用同步方法、同步代码块阻塞或者IO阻塞时进入阻塞状态，等待同步方法、同步代码块释放或者IO释放会进入就绪状态等待，在运行状态调用yield方法，会放弃CPU的时间片，进入就绪状态，但不会释放锁。
#### 4.1、Thread的方法介绍以及方法对比
##### 4.1.1. start() 与 run()
start(): 启动一个线程，线程之间是没有顺序的，是按CPU分配的时间片来回切换的。
run(): 调用线程的run方法，就是普通的方法调用。

##### 4.1.2. sleep() 与 interrupt()
sleep(long millis): 睡眠指定时间，程序暂停运行，睡眠期间会让出CPU的执行权，去执行其它线程，同时CPU也会监视睡眠的时间，一旦睡眠时间到就会立刻执行(因为睡眠过程中仍然保留着锁，有锁只要睡眠时间到就进入就绪状态，等到获取到CPU时间片后，可以立刻执行)。

interrupt(): 唤醒正在睡眠的程序，调用interrupt()方法，会使得sleep()方法抛出InterruptedException异常，当sleep()方法抛出异常就中断了sleep的方法，从而让程序继续运行下去。

**使用 interrupt()唤醒线程而不是stop()的原因?**
因为线程调用stop()会产生线程安全问题，导致数据不一致，当调用stop()方法时会即刻停止run()方法中剩余的全部工作，包括在catch或finally语句中，同时会立即释放该线程所持有的所有的锁，导致数据得不到同步的处理，出现数据不一致的问题。例如当有两个初始变量a,b,初始值都等于0，方法体中有两步操作，a++, sleep(),b++,线程正常执行完毕应该是a=1,b=1,但是在线程睡眠的时候，调用stop方法的时候，会变成a=1,b=0,造成了数据不一致，出现了线程安全的问题。

##### 4.1.3. wait() 与 notify()
wait、notify和notifyAll方法是Object类的final native方法。所以这些方法不能被子类重写，Object类是所有类的超类，因此在程序中可以通过this或者super来调用this.wait(), super.wait()。

- wait(): 导致线程进入等待阻塞状态，当前线程释放对象锁，进入等待队列，会一直等待直到它被其他线程通过notify()或者notifyAll唤醒。该方法只能在同步方法中调用。如果当前线程不是锁的持有者，该方法抛出一个IllegalMonitorStateException异常。wait(long timeout): 时间到了自动执行，类似于sleep(long millis)，当调用interrupt()方法后，线程必须先获取到锁后，然后才抛出异常InterruptedException 。注意： 在获取锁之前是不会抛出异常的，只有在获取锁之后才会抛异常。
- notify(): 该方法只能在同步方法或同步块内部调用， 随机选择一个(注意：只会通知一个)在该对象上调用wait方法的线程，解除其阻塞状态，唤醒的是等待队列中的线程。
- notifyAll(): 唤醒是等待队列中所有的wait对象。
![在这里插入图片描述](https://img-blog.csdnimg.cn/c3a8ebedb7e6410eb345d462bcd6b0b7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**流程：**
- 线程1获取对象A的锁，正在使用对象A。
- 线程1调用对象A的wait()方法。
- 线程1释放对象A的锁，并马上进入等待队列。
- 锁池里面的对象争抢对象A的锁。
- 线程5获得对象A的锁，进入synchronized块，使用对象A。
- 线程5调用对象A的notifyAll()方法，唤醒所有线程，所有线程进入同步队列。若线程5调用对象A的notify()方法，则唤醒一个线程，不知道会唤醒谁，被唤醒的那个线程进入同步队列。
- notifyAll()方法所在synchronized结束，线程5释放对象A的锁。
- 同步队列的线程争抢对象锁，但线程1什么时候能抢到就不知道了

**同步队列状态：同步队列状态又叫做锁池状态**
- 当前线程想调用对象A的同步方法时，发现对象A的锁被别的线程占有，此时当前线程进入锁池状态。简言之，锁池里面放的都是想争夺对象锁的线程。
- 当一个线程1被另外一个线程2唤醒时，1线程进入锁池状态，去争夺对象锁。
- 锁池是在同步的环境下才有的概念，一个对象对应一个锁池。


##### 4.1.4. sleep() 与 wait()
- sleep在Thread类中，wait在Object类中
- sleep不会释放锁，wait会释放锁
- sleep使用interrupt()来唤醒，wait需要notify或者notifyAll来通知

##### 4.1.5.  interrupt()
当调用interrupt方法时，实际上interrupt方法只是改变了线程的“中断状态”而已，所谓中断状态是一个boolean值，表示线程是否被中断的状态并不是调用对象的线程就会InterruptedException异常。

- isInterrupted() 检查中断状态
若指定线程处于中断状态则返回true,若指定线程为非中断状态，则反回false, isInterrupted() 只是获取中断状态的值，并不会改变中断状态的值。

- interrupted()
检查中断状态并清除当前线程的中断状态。如当前线程处于中断状态返回true，若当前线程处于非中断状态则返回false, 并清除中断状态(将中断状态设置为false), 只有这个方法才可以清除中断状态，Thread.interrupted的操作对象是当前线程，所以该方法并不能用于清除其它线程的中断状态

**interrupt()与interrupted()的区别？**

- interrupt()：打断线程，将中断状态修改为true
- interrupted(): 不打断线程，获取线程的中断状态，并将中断状态设置为false

##### 4.1.6.  join()
当前线程里调用另外一个线程thread1的join方法，当前线程进入WAITING/TIMED_WAITING状态，当前线程不会释放已经持有的对象锁。线程thread1执行完毕或者millis时间到，当前线程一般情况下进入RUNNABLE状态，也有可能进入BLOCKED状态（因为join是基于wait实现的，所以说会让出锁）。

##### 4.1.7.  yield()
会交出CPU的执行时间，不会释放锁，让线程进入就绪状态，等待重新获取CPU执行时间，yield就像一个好人似的，当CPU轮到它了，它却说我先不急，先给其他线程执行吧, 此方法很少被使用到。
##### 4.1.8.  setDaemon(boolean on)
- 用户线程：如果主线程main停止掉，不会影响用户线程，用户线程可以继续运行。
- 守护线程：如果主线程死亡，守护线程如果没有执行完毕也要跟着一块死，GC垃圾回收线程就是守护线程。


### 5、当一个线程调用了start方法
当一个线程调用start方法时，在start方法内实际上调用的是本地方法start0()，start0()的具体实现是用C++来编写的，Thread.java中本地方法start0其实就是对应了openJdk中Thread.c的JVM_StartThread方法，在JVM_StartThread中创建本地线程，然后由操作系统去开启线程。
**thread.c**
```cpp
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};
```

查看源代码可以看到在jvm.h中找到了声明，jvm.cpp中有实现，
**jvm.cpp**
```cpp
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
JVMWrapper("JVM_StartThread");
JavaThread *native_thread = NULL;
......
//创建线程
native_thread = new JavaThread(&thread_entry, sz);
......
// 启动线程
Thread::start(native_thread);
```

**thread.cpp**
```cpp
void Thread::start(Thread* thread) {
  trace("start", thread);
  // Start is different from resume in that its safety is guaranteed by context or
  // being called from a Java method synchronized on the Thread object.
  if (!DisableStartThread) {
    if (thread->is_Java_thread()) {
      // Initialize the thread state to RUNNABLE before starting this thread.
      // Can not set it after the thread started because we do not know the
      // exact thread state at that time. It could be in MONITOR_WAIT or
      // in SLEEPING or some other state.
      java_lang_Thread::set_thread_status(((JavaThread*)thread)->threadObj(),
                                          java_lang_Thread::RUNNABLE);
    }
    //操作系统开启线程
    os::start_thread(thread);
  }
}
```
### 6、CompletableFuture
#### 6.1、Future和Callable接口
Future接口定义了操作异步任务执行一些方法，如获取异步任务的执行结果、取消任务的执行、判断任务是否被取消、判断任务执行是否完毕等。
Callable接口中定义了需要有返回的任务需要实现的方法。比如主线程让一个子线程去执行任务，子线程可能比较耗时，启动子线程开始执行任务后，主线程就去做其他事情了，过了一会才去获取子任务的执行结果。

```java
public class CompletableFutureDemo
{
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException
    {
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            System.out.println("-----come in FutureTask");
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            return ""+ThreadLocalRandom.current().nextInt(100);
        });

        Thread t1 = new Thread(futureTask,"t1");
        t1.start();

        //3秒钟后才出来结果，还没有计算你提前来拿(只要一调用get方法，对于结果就是不见不散，会导致阻塞)
        //System.out.println(Thread.currentThread().getName()+"\t"+futureTask.get());

        //3秒钟后才出来结果，我只想等待1秒钟，过时不候
        System.out.println(Thread.currentThread().getName()+"\t"+futureTask.get(1L,TimeUnit.SECONDS));

        System.out.println(Thread.currentThread().getName()+"\t"+" run... here");

    }
}
```
FutureTask 的缺点在于get()会进行阻塞，一旦调用get()方法，不管是否计算完成都会导致阻塞。如果想要获取异步线程返回的结果，通常使用轮询的方式替换阻塞。但是这种样式也会造成阻塞，所以就出现了completableFuture.

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            System.out.println("-----come in FutureTask");
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            return ""+ThreadLocalRandom.current().nextInt(100);
        });

        new Thread(futureTask,"t1").start();

        System.out.println(Thread.currentThread().getName()+"\t"+"线程完成任务");

        /**
         * 用于阻塞式获取结果,如果想要异步获取结果,通常都会以轮询的方式去获取结果
         */
        while(true)
        {
            if (futureTask.isDone())
            {
                System.out.println(futureTask.get());
                break;
            }
        }

    }
}
```
#### 6.2、CompletableFuture对Future的改进
CompletableFuture分别实现了Future接口和CompletionStage接口。
CompletionStage：代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另外一个阶段，有些类似Linux系统的管道分隔符传参数。

CompletableFuture的四个静态方法：
- runAsync 无返回值 
	1. public static CompletableFuture<Void> runAsync(Runnable runnable)
	2. public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor)
- supplyAsync 有返回值
	1. public static <U> CompletableFuture<U>  supplyAsync(Supplier<U> supplier)
	2. public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor)

没有指定Executor的方法，直接使用默认的ForkJoinPool.commonPool() 作为它的线程池执行异步代码
如果指定线程池，则使用我们自定义的或者特别指定的线程池执行异步代码

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
    	//自定义线程池
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 20, 2L,
                TimeUnit.SECONDS, new LinkedBlockingDeque<>(50),
                Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
                
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName()+"\t"+"-----come in");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println("-----task is over");
        });
        System.out.println(future.get());
        
        CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName()+"\t"+"-----come in");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println("-----task is over");
        },poolExecutor);
        System.out.println(future2.get());
    }
}
 
```

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
    	//自定义线程池
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 20, 2L,
                TimeUnit.SECONDS, new LinkedBlockingDeque<>(50),
                Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
                
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "-----come in");
            //暂停几秒钟线程
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return ThreadLocalRandom.current().nextInt(100);
        });
        System.out.println(completableFuture.get());
        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "-----come in");
            //暂停几秒钟线程
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return ThreadLocalRandom.current().nextInt(100);
        },poolExecutor);
        System.out.println(completableFuture2.get());
    }
}
```
#### 6.3、案例精讲-从电商网站的比价需求

```java
package com.song.test;
import lombok.Getter;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

/**
 * @auther swq
 * @create 2022-03-08 15:28
 * <p>
 * 案例说明：电商比价需求
 * 1 同一款产品，同时搜索出同款产品在各大电商的售价;
 * 2 同一款产品，同时搜索出本产品在某一个电商平台下，各个入驻门店的售价是多少
 * <p>
 * 出来结果希望是同款产品的在不同地方的价格清单列表，返回一个List<String>
 * 《mysql》 in jd price is 88.05
 * 《mysql》 in pdd price is 86.11
 * 《mysql》 in taobao price is 90.43
 * <p>
 * 3 要求深刻理解
 * 3.1 函数式编程
 * 3.2 链式编程
 * 3.3 Stream流式计算
 */
public class CompletableFutureNetMallDemo {
    static List<NetMall> list = Arrays.asList(
            new NetMall("jd"),
            new NetMall("pdd"),
            new NetMall("taobao"),
            new NetMall("dangdangwang"),
            new NetMall("tmall")
    );

    //同步 ，step by step
    /**
     * List<NetMall>  ---->   List<String>
     * @param list
     * @param productName
     * @return
     */
    public static List<String> getPriceByStep(List<NetMall> list, String productName) {
        return list
                .stream().
                 map(netMall -> String.format(productName + " in %s price is %.2f", netMall.getMallName(), 		         netMall.calcPrice(productName)))
                .collect(Collectors.toList());
    }
    //异步 ，多箭齐发
    /**
     * List<NetMall>  ---->List<CompletableFuture<String>> --->   List<String>
     * @param list
     * @param productName
     * @return
     */
    public static List<String> getPriceByASync(List<NetMall> list, String productName) {
        return list.stream()
                .map(netMall -> CompletableFuture.supplyAsync(() -> String.format(productName + " is %s price is %.2f", netMall.getMallName(), netMall.calcPrice(productName))))
                .collect(Collectors.toList())
                .stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
    }
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();
        List<String> list1 = getPriceByStep(list, "mysql");
        for (String element : list1) {
            System.out.println(element);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime - startTime) + " 毫秒");

        System.out.println();

        long startTime2 = System.currentTimeMillis();
        List<String> list2 = getPriceByASync(list, "mysql");
        for (String element : list2) {
            System.out.println(element);
        }
        long endTime2 = System.currentTimeMillis();
        System.out.println("----costTime: " + (endTime2 - startTime2) + " 毫秒");
    }
}
class NetMall {
    @Getter
    private String mallName;
    public NetMall(String mallName) {
        this.mallName = mallName;
    }
    public double calcPrice(String productName) {
        //检索需要1秒钟
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return ThreadLocalRandom.current().nextDouble() * 2 + productName.charAt(0);
    }
}
```
#### 6.4、CompletableFuture常用方法
##### 6.4.1、获得结果和触发计算
- 获得结果
	1. public T  get() ：发生阻塞，
	2. public T  get(long timeout, TimeUnit unit)：发生阻塞，超时会抛出异常
	3. public T  join()：发生阻塞，不会抛出异常
	4. public T  getNow(T valueIfAbsent)：立即获取结果不阻塞，如果已经计算完，返回计算完成后的结果，如果没算完，返回设定的valueIfAbsent值
- 主动触发计算
	1. public boolean complete(T value) ：是否成功打断，如果调用方法时，计算已经完成，则打断为false，如果计算未完成，此时调用方法为打断成功。打断成功，get方法立即返回括号值，打断失败，返回计算返回的值。

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            return 533;
        });

        //注释掉暂停线程，get还没有算完只能返回complete方法设置的444；暂停2秒钟线程，异步线程能够计算完成返回get
        try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }

        //当调用CompletableFuture.get()被阻塞的时候,complete方法就是结束阻塞并get()获取设置的complete里面的值.
        System.out.println(completableFuture.complete(444)+"\t"+completableFuture.get());


    }
}
```

##### 6.4.2、对计算结果进行处理
- thenApply：计算结果存在依赖关系，这两个线程串行化，由于存在依赖关系(当前步错，不走下一步)，当前步骤有异常的话就叫停。
	

```java
public class CompletableFutureDemo2
{
public static void main(String[] args) throws ExecutionException, InterruptedException
{
    //当一个线程依赖另一个线程时用 thenApply 方法来把这两个线程串行化,
    CompletableFuture.supplyAsync(() -> {
        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("111");
        return 1024;
    }).thenApply(f -> {
        System.out.println("222");
        return f + 1;
    }).thenApply(f -> {
        //int age = 10/0; // 异常情况：那步出错就停在那步。
        System.out.println("333");
        return f + 1;
    }).whenCompleteAsync((v,e) -> {
        System.out.println("*****v: "+v);
    }).exceptionally(e -> {
        e.printStackTrace();
        return null;
    });
    System.out.println("-----主线程结束，END");
    // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
    try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
}
}

```
- handle：有异常也可以往下一步走，根据带的异常参数可以进一步处理

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        //当一个线程依赖另一个线程时用 handle 方法来把这两个线程串行化,
        // 异常情况：有异常也可以往下一步走，根据带的异常参数可以进一步处理
        CompletableFuture.supplyAsync(() -> {
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println("111");
            return 1024;
        }).handle((f,e) -> {
            int age = 10/0;
            System.out.println("222");
            return f + 1;
        }).handle((f,e) -> {
            System.out.println("333");
            return f + 1;
        }).whenCompleteAsync((v,e) -> {
            System.out.println("*****v: "+v);
        }).exceptionally(e -> {
            e.printStackTrace();
            return null;
        });
        System.out.println("-----主线程结束，END");
        // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
        try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
    }
}
```
##### 6.4.3、对计算结果进行消费
- thenAccept：接收任务的处理结果，并消费处理，无返回结果

```java
public static void main(String[] args) throws ExecutionException, InterruptedException
{
    CompletableFuture.supplyAsync(() -> {
        return 1;
    }).thenApply(f -> {
        return f + 2;
    }).thenApply(f -> {
        return f + 3;
    }).thenApply(f -> {
        return f + 4;
    }).thenAccept(r -> System.out.println(r));
}
```

**thenRun、thenAccept、thenApply的区别？**
- thenRun(Runnable runnable)：任务 A 执行完执行 B，并且 B 不需要 A 的结果
- thenAccept(Consumer action)：任务 A 执行完执行 B，B 需要 A 的结果，但是任务 B 无返回值
- thenApply(Function fn)：任务 A 执行完执行 B，B 需要 A 的结果，同时任务 B 有返回值

```java
System.out.println(CompletableFuture.supplyAsync(() -> "resultA").thenRun(() -> {}).join());
System.out.println(CompletableFuture.supplyAsync(() -> "resultA").thenAccept(resultA -> {}).join());
System.out.println(CompletableFuture.supplyAsync(() -> "resultA").thenApply(resultA -> resultA + " resultB").join());
```

##### 6.4.4、对计算速度进行选用
- applyToEither：谁快用谁

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            return 10;
        });

        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            return 20;
        });

        CompletableFuture<Integer> thenCombineResult = completableFuture1.applyToEither(completableFuture2,f -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            return f + 1;
        });

        System.out.println(Thread.currentThread().getName() + "\t" + thenCombineResult.get());
    }
}
 
```

##### 6.4.5、对计算结果进行合并
- thenCombine：两个CompletionStage任务都完成后，最终能把两个任务的结果一起交给thenCombine 来处理，先完成的先等着，等待其它分支任务

```java
public class CompletableFutureDemo2
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        CompletableFuture<Integer> completableFuture1 = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            return 10;
        });

        CompletableFuture<Integer> completableFuture2 = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            return 20;
        });

        CompletableFuture<Integer> thenCombineResult = completableFuture1.thenCombine(completableFuture2, (x, y) -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in ");
            return x + y;
        });
        
        System.out.println(thenCombineResult.get());
    }
}
 
 //套娃
public class CompletableFutureDemo3
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        CompletableFuture<Integer> thenCombineResult = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in 1");
            return 10;
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in 2");
            return 20;
        }), (x,y) -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in 3");
            return x + y;
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in 4");
            return 30;
        }),(a,b) -> {
            System.out.println(Thread.currentThread().getName() + "\t" + "---come in 5");
            return a + b;
        });
        System.out.println("-----主线程结束，END");
        System.out.println(thenCombineResult.get());

        // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
        try { TimeUnit.SECONDS.sleep(10); } catch (InterruptedException e) { e.printStackTrace(); }
    }
}

```

- thenCompose：允许将两个异步操作进行流水线，第一个操作完成时,将其结果作为参数传递给第二个操作。

```java
public static void main(String[] args) {
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            return 12;
        });
        System.out.println(future1.thenCompose(integer -> {
            return CompletableFuture.supplyAsync(() -> {
                return integer + 100;
            });
        }).join());
        
        // 主线程不要立刻结束，否则CompletableFuture默认使用的线程池会立刻关闭:
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

### 7.线程池
#### 7.1、什么是线程池?

线程池就是创建若干个可执行的线程放入一个池（容器）中，有任务需要处理时，会提交到线程池中的任务队列，处理完之后线程并不会被销毁，而是仍然在线程池中等待下一个任务

#### 7.2、为什么要使用线程池？

1. 降低资源消耗。通过重复利用己创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
3. 提高线程的可管理性。使用线程池可以进行统一的分配，调优和监控。

#### 7.3、线程池的创建方式

1. 使用Executor创建线程池

   ​	**newFixedThreadPool**：使用`LinkedBlockingQueue`实现，定长线程池。

   ```java
   public static ExecutorService newFixedThreadPool(int nThreads) {
       return new ThreadPoolExecutor(nThreads, nThreads,
                                     0L, TimeUnit.MILLISECONDS,
                                     new LinkedBlockingQueue<Runnable>());
   }
   ```

   

   ​	**newSingleThreadExecutor**：使用`LinkedBlockingQueue`实现，一池只有一个线程。

   ```java
   public static ExecutorService newSingleThreadExecutor() {
       return new FinalizableDelegatedExecutorService(new ThreadPoolExecutor(1, 1,
                                       0L, TimeUnit.MILLISECONDS,
                                       new LinkedBlockingQueue<Runnable>()));
   }
   ```

   

   ​	**newCachedThreadPool**：使用`SynchronousQueue`实现，变长线程池。

   ```java
   public static ExecutorService newCachedThreadPool() {
       return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
   }
   ```

   

2. 自定义线程池

   ```java
   package com.song.gulimall.product.config;
   
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   
   import java.util.concurrent.*;
   
   /* *
    * @program: gulimall
    * @description
    * @author: swq
    * @create: 2021-03-24 23:15
    **/
   @Configuration
   public class MyThreadPoolConfig {
   
       @Bean
       public ThreadPoolExecutor threadPoolExecutor(ThreadPoolConfigProperties pool) {
           return new ThreadPoolExecutor(pool.getCoreSize(),
                   pool.getMaxSize(),
                   pool.getKeepAliveTime(),
                   TimeUnit.SECONDS,
                   new LinkedBlockingDeque<>(100000),
                   Executors.defaultThreadFactory(),
                   new ThreadPoolExecutor.AbortPolicy());
       }
   }
   
   
   
   package com.song.gulimall.product.config;
   
   import lombok.Data;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.stereotype.Component;
   
   /* *
    * @program: gulimall
    * @description
    * @author: swq
    * @create: 2021-03-24 23:20
    **/
   @ConfigurationProperties(prefix = "spring.thread")
   @Component
   @Data
   public class ThreadPoolConfigProperties {
       private Integer coreSize;
       private Integer maxSize;
       private Integer keepAliveTime;
   
   }
   
   
   ```

**总结：**

1. 项目必须使用自定义的线程池，
2. FixedThreadPool和SingleThreadPool：允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
3. CachedThreadPool和ScheduledThreadPool：允许的创建线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。


#### 7.4、ThreadPoolExecutor

**创建线程池的7个参数？**

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```



- corePoolSize：线程池中的常驻核心线程数,在创建了线程池后，当有请求任务来之后，就会安排池中的线程去执行请求任务，近似理解为今日当值线程，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中
- maximumPoolSize：线程池能够容纳同时执行的最大线程数，此数值必须大于等于1
- keepAliveTime：多余的空闲线程的存活时间。当前线程池数量超过corePoolSize时，当空闲时间达到keepAliveTime值时，多余空闲线程会被销毁直到只剩下corePoolSize个线程为止
- unit：keepAliveTime的单位
- workQueue：任务队列，被提交但尚未被执行的任务（阻塞队列）
- threadFactory：表示生成线程池中工作线程的线程工厂，用于创建线程一般用默认的即可
- handler：拒绝策略，表示当队列满了并且工作线程大于等于线程池的最大线程数时，如何来拒绝



#### 7.5、线程池的拒绝策略

**什么是线程池的拒绝策略？**

等待队列也已经排满了，再也塞不下新任务了，同时线程池中的max线程也达到了，无法继续为新任务服务，阻止新任务进入的一种方式。

- AbortPolicy（默认）：直接抛出RejectedExecutionException异常阻止系统正常运行
- CallerRunsPolicy：调用者运行，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退给调用者，从而降低新任务的流量
- DiscardOldestPolicy：抛弃队列中等待最久的任务，然后把当前任务加入队列中尝试再次提交当前任务
- DiscardPolicy：直接丢弃任务，不予任何处理也不抛出异常。如果允许任务丢失，这是最好的一种方案

#### 7.6、线程池的底层原理
**底层架构**
![在这里插入图片描述](https://img-blog.csdnimg.cn/5acbf2590ff94ebab6b4ce0b79b4b583.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_19,color_FFFFFF,t_70,g_se,x_16)

**工作流程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/16eae95854b74a72ae9db0b34cf668f6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**解释：**

1. 在创建了线程池后，等待提交过来的任务请求。当调用execute()方法添加一个请求任务时，线程池会做如下判断：
2. 如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个任务；
3. 如果正在运行的线程数量大于或等于corePoolSize，那么将这个任务放入队列；
4. 如果这时候队列满了且正在运行的线程数量还小于maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
5. 如果队列满了且正在运行的线程数量大于或等于maximumPoolSize，那么线程池会启动饱和拒绝策略来执行。
6. 当一个线程完成任务时，它会从队列中取下一个任务来执行。
7. 当一个线程无事可做超过一定的时间（keepAliveTime）时，线程池会判断：
8. 如果当前运行的线程数大于corePoolSize，那么这个线程就被停掉，所以线程池的所有任务完成后它最终会收缩到corePoolSize的大小

**总结：**

​	当向线程池提交一个任务后，如果当前的线程数小于核心线程数（corePoolSize），那么就会立即创建线程执行任务，如果当前线程数大于核心线程数，而且阻塞队列容量未满的情况下，会放入到阻塞队列中，如果阻塞队列也满了，但是当前的线程运行数小于最大线程数，那么就会进行扩容，创建非核心线程来立即执行这个任务，如果阻塞队列满了，当前的运行线程数也达到了最大值，那么线程池就会启动拒绝策略。如果有线程的闲置时间超过线程的最大存活时间，那么就会关闭这个线程。

#### 7.7、线程池的参数（最大线程数）

**如何查看机器的逻辑处理器个数**

```java
System.out.println(Runtime.getRuntime().availableProcessors());
```

**CPU 密集型**

- CPU密集的意思是该任务需要大量的运算，而没有阻塞，CPU一直全速运行。
- CPU密集型任务配置尽可能少的线程数量，一般公式：CPU核数+1个线程的线程池

**IO 密集型**

- 由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如CPU核数*2。
- IO密集型，即该任务需要大量的IO，即大量的阻塞。在单线程上运行IO密集型的任务会导致浪费大量的CPU运算能力浪费在等待。
  所以在IO密集型任务中使用多线程可以大大的加速程序运行，即使在单核CPU上，这种加速主要就是利用了被浪费掉的阻塞时间。
- IO密集型时，大部分线程都阻塞，故需要多配置线程数：参考公式：CPU核数/(1-阻塞系数)，阻塞系数在0.8~0.9之间，比如8核CPU：8/(1-0.9)=80个线程数
