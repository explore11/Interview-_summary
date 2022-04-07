## 1、Java内存模型
### 1.1、什么是Java内存模型JMM？
JMM(Java内存模型Java Memory Model，简称JMM)本身是一种抽象的概念并不真实存在，它仅仅描述的是一组约定或规范，通过这组规范定义了程序中(尤其是多线程)各个变量（包括实例字段，静态字段和构成数组对象的元素）的读写访问方式。
**JMM关于同步的规定：**
- 线程解锁前，必须把共享变量的值刷新回主内存
- 线程加锁前，必须读取主内存的最新值，到自己的工作内存
- 加锁和解锁是同一把锁

**JMM规范有三大特性：**
- 可见性：可见性是指当一个线程修改了某一个共享变量的值，其他线程是否能够立即知道该变更，这种机制就是JMM中的可见性。。
- 原子性：是指一个操作是不可中断的，要保证完整性，操作不能被其他线程干扰。也就是说某个线程在做具体的业务的时，它的中间过程不可以被分割，需要整体完整，要么全部成功，要么全部失败。
- 有序性：程序执行的顺序按照代码的先后顺序执行。



### 1.2、要有JMM的原因及作用？
#### 1.2.1、为什么要有JMM，它为什么出现？
因为java是跨平台的，要想在不同的平台上（window、linux、mac）进行正常访问，需要屏蔽各个硬件平台和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。
#### 1.2.1、JMM的作用和功能？
- 通过JMM来实现线程和主内存之间的抽象关系。
- 屏蔽各个硬件平台和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。
### 1.3、可见性、有序性
 - **可见性**：可见性是指当一个线程修改了某一个共享变量的值，其他线程是否能够立即知道该变更。
![在这里插入图片描述](https://img-blog.csdnimg.cn/1fc3b2aa26f04947adb88aa8a32624e3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/0996b05122e3441893e0456c9190b5dd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

Java中普通的共享变量不保证可见性，因为数据修改被写入内存的时机是不确定的，多线程并发下很可能出现"脏读"，所以每个线程都有自己的工作内存，线程自己的工作内存中保存了该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作（读取，赋值等 ）都必需在线程自己的工作内存中进行，而不能够直接读写主内存中的变量。不同线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成。
 
**线程脏读**：如果没有可见性保证主内存中有变量 x，初始值为 0线程 A 要将 x 加 1，先将 x=0 拷贝到自己的私有内存中，然后更新 x 的值线程 A 将更新后的 x 值回刷到主内存的时间是不固定的刚好在线程 A 没有回刷 x 到主内存时，线程 B 同样从主内存中读取 x，此时为 0，和线程 A 一样的操作，最后期盼的 x=2 就会变成 x=1
 
  **有序性：** 对于一个线程的执行代码而言，我们总是习惯性认为代码的执行总是从上到下，有序执行。
但为了提供性能，编译器和处理器通常会对指令序列进行重新排序。
指令重排可以保证串行语义一致，但没有义务保证多线程间的语义也一致，即可能产生"脏读"，简单说，
两行以上不相干的代码在执行的时候有可能先执行的不是第一条，不见得是从上到下顺序执行，执行顺序会被优化。
 
单线程环境里面确保程序最终执行结果和代码顺序执行的结果一致。处理器在进行重排序时必须要考虑指令之间的数据依赖性,多线程环境中线程交替执行,由于编译器优化重排的存在，两个线程中使用的变量能否保证一致性是无法确定的,结果无法预测。
 
### 1.4、 JMM规范下，多线程对变量的读写过程？

#### 1.4.1、读写过程：
由于JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(有些地方称为栈空间)，工作内存是每个线程的私有数据区域，而Java内存模型中规定所有变量都存储在主内存，主内存是共享内存区域，所有线程都可以访问，但线程对变量的操作(读取赋值等)必须在工作内存中进行，首先要将变量从主内存拷贝到的线程自己的工作内存空间，然后对变量进行操作，操作完成后再将变量写回主内存，不能直接操作主内存中的变量，各个线程中的工作内存中存储着主内存中的变量副本拷贝，因此不同的线程间无法访问对方的工作内存，线程间的通信(传值)必须通过主内存来完成，其简要访问过程如下图:
![在这里插入图片描述](https://img-blog.csdnimg.cn/fb811cb2972c4942a76c3243d0d7dafa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.4.2、缓存一致性
当多个处理器运算任务都涉及到同一块主内存区域的时候，将可能导致各自的缓存数据不一，为了解决缓存一致性的问题，需要各个处理器访问缓存时都遵循一些协议，在读写时要根据协议进行操作，这类协议主要有MSI、MESI等等。

**解决方式：**
- 通过在总线加LOCK锁的方式：因为CPU和其他部件进行通信都是通过总线来进行的，如果对总线加LOCK锁的话，也就是说明阻塞了其他CPU对其它部件访问（如内存），从而使得只能有一个CPU能使用这个变量的内存，导致效率低下。
- 通过缓存一致性协议：当CPU向内存写入数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存是无效的，那么它就会从内存重新读取。
#### 1.4.3、总线嗅探技术
- **那么是如何发现缓存中的数据是否失效呢？**
这里是用到了总线嗅探技术，本质上就是把所有的读写请求都通过总线（Bus）广播给所有的CPU核心，就是每个处理器通过嗅探在总线上传播的数据来检查自己缓存值是否过期了，当处理器发现自己的缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置为无效状态，当处理器对这个数据进行修改操作的时候，会重新从内存中把数据读取到处理器缓存中。
- **什么时候去触发“嗅探”这个操作？**
猜测是在volatile修饰的共享变量的写操作会触发“嗅探”，让处理器本地缓存中的volatile变量失效；

#### 1.4.4、总线风暴
**缓存一致性流量：** 在多核处理器架构上，所有的处理器是共用一条总线的，都是靠此总线来和主内存进行数据交互。当主内存的数据同时存在于多个处理的高速缓存中时，某一处理器更新了此共享数据后。会通过总线触发嗅探机制来通知其他处理器将自己高速缓存内的共享数据置为无效，在下次使用时重新从主内存加载最新数据。而这种通过总线来进行通信则称之为”缓存一致性流量“。

**总线风暴：** 在java中使用unsafe实现cas,而其底层由cpp调用汇编指令实现的，如果是多核cpu是使用lock cmpxchg指令（锁总线），单核cpu 使用compxch指令。如果在短时间内产生大量的cas操作在加上 volatile的嗅探机制则会不断地占用总线带宽，导致缓存一致性流量激增，就会产生总线风暴。 总之，就是因为volatile 和CAS 的操作导致BUS总线缓存一致性流量激增所造成的影响。

#### 1.4.5、JMM定义了线程和主内存之间的抽象关系
1 线程之间的共享变量存储在主内存中(从硬件角度来说就是内存条)
2 每个线程都有一个私有的本地工作内存，本地工作内存中存储了该线程用来读/写共享变量的副本(从硬件角度来说就是CPU的缓存，比如寄存器、L1、L2、L3缓存等)

#### 1.4.6、总结：
每个线程都有自己独立的工作内存，里面保存该线程使用到的变量的副本(主内存中该变量的一份拷贝)，线程对共享变量所有的操作都必须先在线程自己的工作内存中进行后写回主内存，不能直接从主内存中读写(不能越级)，不同线程之间也无法直接访问其他线程的工作内存中的变量，线程间变量值的传递需要通过主内存来进行(同级不能相互访问)。

### 1.5、多线程先行发生原则之happens-before
在JMM中，如果一个操作执行的结果需要对另一个操作可见或者代码重排序，那么这两个操作之间必须存在happens-before关系。
 
```java
x = 5 线程A执行
y = x 线程B执行
上述称之为：写后读

问题?
y是否等于5呢？ 
如果线程A的操作（x= 5）happens-before(先行发生)线程B的操作（y = x）,那么可以确定线程B执行后y = 5 一定成立; 
如果他们不存在happens-before原则，那么y = 5 不一定成立。 
这就是happens-before原则的威力。-------------------》包含可见性和有序性的约束
```

如果Java内存模型中所有的有序性都仅靠volatile和synchronized来完成，那么有很多操作都将会变得非常啰嗦，但是我们在编写Java并发代码的时候并没有察觉到这一点。我们没有时时、处处、次次，添加volatile和synchronized来完成程序？

这是因为Java语言中JMM原则下有一个“先行发生”(Happens-Before)的原则限制和规矩。这个原则非常重要： 它是判断数据是否存在竞争，线程是否安全的非常有用的手段。依赖这个原则，我们可以通过几条简单规则一揽子解决并发环境下两个操作之间是否可能存在冲突的所有问题，而不需要陷入Java内存模型苦涩难懂的底层编译原理之中。
#### 1.5.1、happens-before总原则
- 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
- 两个操作之间存在happens-before关系，并不意味着一定要按照happens-before原则制定的顺序来执行。如果重排序之后的执行结果与按照happens-before关系来执行的结果一致，那么这种重排序并不非法。例如：1+2+3 = 3+2+1

#### 1.5.2、happens-before中的8条规则
- **次序规则：** 一个线程内，按照代码顺序，写在前面的操作先行发生于写在后面的操作；前一个操作的结果可以被后续的操作获取。讲白点就是前面一个操作把变量X赋值为1，那后面一个操作肯定能知道X已经变成了1。
- **锁定规则：** 一个unLock操作先行发生于后面((这里的“后面”是指时间上的先后))对同一个锁的lock操作，对于同一把锁objectLock，threadA一定先unlock同一把锁后，B才能获得该锁，A 先行发生于B。
- **volatile变量规则：** 对一个volatile变量的写操作先行发生于后面对这个变量的读操作，前面的写对后面的读是可见的，这里的“后面”同样是指时间上的先后。
- **传递规则：** 如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
- **线程启动规则(Thread Start Rule)：** Thread对象的start()方法先行发生于此线程的每一个动作
- **线程中断规则(Thread Interruption Rule)：** 对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()检测到是否发生中断
- **线程终止规则(Thread Termination Rule)：** 线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread::join()方法是否结束、Thread::isAlive()的返回值等手段检测线程是否已经终止执行。
- **对象终结规则(Finalizer Rule)：** 一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始，即对象没有完成初始化之前，是不能调用finalized()方法的。
#### 1.5.3、案例说明
![在这里插入图片描述](https://img-blog.csdnimg.cn/507e33be63c9432fb062be3bc1cd9015.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_12,color_FFFFFF,t_70,g_se,x_16)
假设存在线程A和B， 线程A先（时间上的先后）调用了setValue(1)， 然后线程B调用了同一个对象的getValue()，那么线程B收到的返回值是什么？

 我们就这段简单的代码一次分析happens-before的规则（规则5、6、7、8 可以忽略，因为他们和这段代码毫无关系）：
 1.  由于两个方法是由不同的线程调用，不在同一个线程中，所以肯定不满足程序次序规则；
 2.  两个方法都没有使用锁，所以不满足锁定规则；
 3.  变量不是用volatile修饰的，所以volatile变量规则不满足；
 4.  传递规则肯定不满足,一共就两个线程。

 所以我们无法通过happens-before原则推导出线程A happens-before线程B，虽然可以确认在时间上线程A优先于线程B指定，但就是无法确认线程B获得的结果是什么，所以这段代码不是线程安全的。
 
 **那么怎么修复这段代码呢？**
 - 把getter/setter方法都定义为synchronized方法
 - 把value定义为volatile变量，由于setter方法对value的修改不依赖value的原值，满足volatile关键字使用场景


### 1.6、volatile与Java内存模型
#### 1.6.1、被volatile修改的变量有2大特性，可见性、有序性，但不能保证原子性
**volatile的内存语义：**
- 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值立即刷新回主内存中。
- 当读一个volatile变量时，JMM会把该线程对应的本地内存设置为无效，直接从主内存中读取共享变量。
- 所以volatile的写内存语义是直接刷新到主内存中，读的内存语义是直接从主内存中读取。

#### 1.6.2、volatile凭什么可以保证可见性和有序性？
内存屏障 (Memory Barriers / Fences)
#### 1.6.3、什么是内存屏障？
内存屏障 (Memory Barriers / Fences)：内存屏障其实就是一种JVM指令，是一类同步屏障指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作，避免代码重排序，Java内存模型的重排规则会要求Java编译器在生成JVM指令时插入特定的内存屏障指令，通过这些内存屏障指令，volatile实现了Java内存模型中的可见性和有序性，但volatile无法保证原子性。
 
内存屏障之前的所有写操作都要回写到主内存，内存屏障之后的所有读操作都能获得内存屏障之前的所有写操作的最新结果(实现了可见性)。因此重排序时，不允许把内存屏障之后的指令重排序到内存屏障之前。

一句话：对一个 volatile 域的写, happens-before 于任意后续对这个 volatile 域的读，也叫写后读。
#### 1.6.4、内存屏障的四类内存屏障指令？
内存屏障的底层实际就是四大内存屏障指令
![在这里插入图片描述](https://img-blog.csdnimg.cn/5627f7c8f9f94768beb34ed1144d83fb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)


#### 1.6.5、happens-before 之 volatile 变量规则
![在这里插入图片描述](https://img-blog.csdnimg.cn/d5a70fc5cc2b4b0a84547f666a2955a6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 当第一个操作为volatile读时，不论第二个操作是什么，都不能重排序。这个操作保证了volatile读之后的操作不会被重排到volatile读之前。
- 当第二个操作为volatile写时，不论第一个操作是什么，都不能重排序。这个操作保证了volatile写之前的操作不会被重排到volatile写之后。
- 第一个操作为volatile写时，第二个操作为volatile读时，不能重排。
 
#### 1.6.6、JMM 就将内存屏障插⼊策略分为 4 种
![在这里插入图片描述](https://img-blog.csdnimg.cn/5627f7c8f9f94768beb34ed1144d83fb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 写：
	1. 在每个 volatile 写操作的前⾯插⼊⼀个 StoreStore 屏障
	2. 在每个 volatile 写操作的后⾯插⼊⼀个 StoreLoad 屏障
![在这里插入图片描述](https://img-blog.csdnimg.cn/6efad2a5a4c84641af798abbe292bf1f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2cc84807a44947af83b5560bb2c5c423.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 读：
	3.  在每个 volatile 读操作的后⾯插⼊⼀个 LoadLoad 屏障
	4.  在每个 volatile 读操作的后⾯插⼊⼀个 LoadStore 屏障
![在这里插入图片描述](https://img-blog.csdnimg.cn/1f0125f987e246d78ae9ab2ddfee75da.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/78982c7f885d4053a789b93e0e3002de.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)



#### 1.6.5、volatile特性
##### 1.6.5.1、保证可见性
保证不同线程对这个变量进行操作时的可见性，即变量一旦改变所有线程立即可见

```java
public class VolatileSeeDemo
{
    static          boolean flag = true;       //不加volatile，没有可见性
    //static volatile boolean flag = true;       //加了volatile，保证可见性
    public static void main(String[] args)
    {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName()+"\t come in");
            while (flag)
            {
            }
            System.out.println(Thread.currentThread().getName()+"\t flag被修改为false,退出.....");
        },"t1").start();、
        //暂停2秒钟后让main线程修改flag值
        try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
        flag = false;
        System.out.println("main线程修改完成");
    }
}
```
**在flag 不加volatile的情况下，线程t1中为何看不到被主线程main修改为false的flag的值？**
 
**可能出现的问题:**
1. 主线程修改了flag之后没有将其刷新到主内存，所以t1线程看不到。
2. 主线程将flag刷新到了主内存，但是t1一直读取的是自己工作内存中flag的值，没有去主内存中更新获取flag最新的值。
 
**解决：使用volatile修饰共享变量，就可以达到上面的效果，被volatile修改的变量有以下特点：**
3. 线程中读取的时候，每次读取都会去主内存中读取共享变量最新的值，然后将其复制到工作内存
4. 线程中修改了工作内存中变量的副本，修改之后会立即刷新到主内存



 
 **volatile变量的读写流程**

Java内存模型中定义的8种工作内存与主内存之间的原子操作，read(读取)→load(加载)→use(使用)→assign(赋值)→store(存储)→write(写入)→lock(锁定)→unlock(解锁)![在这里插入图片描述](https://img-blog.csdnimg.cn/c5bdde08ed594bf29db494fefe4842b8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**volatile变量的读写流程描述：**

 一个volatile变量读写的时候，先会在主内存中进行read操作，将变量的值从主内存传输到工作内存，接着会在工作内存中进行load操作，将从主内存传输的变量值放入工作内存变量副本中，然后在工作内存中进行use操作，将工作内存变量副本的值传递给执行引擎（CPU），由cpu进行计算，cpu计算完毕之后，会进行assign操作，将计算完毕接收到的值赋值给工作内存变量，然后会进行store操作，将赋值完毕的工作变量的值从工作内存写回到主内存，接着在主内存中会进行write操作，将从工作内存中传过来的值赋值给主内存中的变量，在write的时候，会先对变量进行lock操作，对变量加锁（加锁之后会通过总线嗅探将其他线程的工作内存中变量的值，置为失效，当其他的线程再次使用的时候，只能去主内存中再次加载），变量的值修改完毕之后，会在进行unlock操作，对变量释放锁。
 
 **8大原子操作解析：read(读取)→load(加载)→use(使用)→assign(赋值)→store(存储)→write(写入)→lock(锁定)→unlock(解锁)**
- read: 作用于主内存，将变量的值从主内存传输到工作内存，主内存到工作内存
- load: 作用于工作内存，将read从主内存传输的变量值放入工作内存变量副本中，即数据加载
- use: 作用于工作内存，将工作内存变量副本的值传递给执行引擎，每当JVM遇到需要该变量的字节码指令时会执行该操作
- assign: 作用于工作内存，将从执行引擎接收到的值赋值给工作内存变量，每当JVM遇到一个给变量赋值字节码指令时会执行该操作
- store: 作用于工作内存，将赋值完毕的工作变量的值写回给主内存
- write: 作用于主内存，将store传输过来的变量值赋值给主内存中的变量
- lock: 作用于主内存，将一个变量标记为一个线程独占的状态，只是写时候加锁，就只是锁了写变量的过程。
- unlock: 作用于主内存，把一个处于锁定状态的变量释放，然后才能被其他线程占用
##### 1.6.5.2、没有原子性
对于含有数据依赖的复合操作就不具有原子性，例如i++

```java
class MyNumber
{
    volatile int number = 0;
    public void addPlusPlus()
    {
        number++;
    }
}

public class VolatileNoAtomicDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        MyNumber myNumber = new MyNumber();
        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                for (int j = 1; j <= 1000; j++) {
                    myNumber.addPlusPlus();
                }
            },String.valueOf(i)).start();
        }
        
        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println(Thread.currentThread().getName() + "\t" + myNumber.number);
    }
}
```
原子性指的是一个操作是不可中断的，即使是在多线程环境下，一个操作一旦开始就不会被其他线程影响。

```java
public void add()
{
		//不具备原子性，该操作是先读取值，然后写回一个新值，相当于原来的值加上1，
		//从字节码上看，整个操作是分成3个步骤的，分3步完成
        i++; 
 }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/37d7aed34c8f4507ab6b8e8b82542747.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
字节码
```java
 0 aload_0
 1 dup
 2 getfield #2 <com/song/test/MyNumber.number>   //获取
 5 iconst_1
 6 iadd     //计算
 7 putfield #2 <com/song/test/MyNumber.number>    //写入
10 return
```
**代码解析：**
如果第二个线程在第一个线程读取旧值和写回新值期间读取i的域值，那么第二个线程就会与第一个线程一起看到同一个值，
并执行相同值的加1操作，这也就造成了线程安全失败，因此对于add方法必须使用synchronized修饰，以便保证线程安全.

![在这里插入图片描述](https://img-blog.csdnimg.cn/b535c27757e8495fb9bceb5247f8b96d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

在多线程环境下，"数据计算"和"数据赋值"操作可能多次出现，即操作非原子。若数据在加载之后，若主内存count变量发生修改之后，由于线程工作内存中的值在此前已经加载，从而不会对变更操作做出相应变化，即私有内存和公共内存中变量不同步，进而导致数据不一致
对于volatile变量，JVM只是保证从主内存加载到线程工作内存的值是最新的，也就是数据加载时是最新的。由此可见volatile解决的是变量读时的可见性问题，但无法保证原子性，对于多线程修改共享变量的场景必须使用加锁同步。

**读取赋值一个普通变量的情况**
当线程1对主内存对象发起read操作到write操作第一套流程的时间里，线程2随时都有可能对这个主内存对象发起第二套操作。
![在这里插入图片描述](https://img-blog.csdnimg.cn/7c9c30eb3c5d406d920ec87b4b2f08e2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_14,color_FFFFFF,t_70,g_se,x_16)
每一个步骤都是原子性操作的，没有加volatile的情况下，线程1对主内存对象发起read操作到write操作时，其他线程随时都有可能再次发起read操作到write操作，做成线程不安全的问题。



**读取赋值一个volatile变量的情况**
![在这里插入图片描述](https://img-blog.csdnimg.cn/b45a7837de6c40e9954371bace9a11d8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_13,color_FFFFFF,t_70,g_se,x_16)
加了volatile之后，volatile主要是对其中部分指令做了处理，
- 在进行volatile读的时候，要求要 **use(使用)一个变量的时候必需load(载入），要载入的时候必需从主内存read(读取）** 这样就解决了读的可见性。
- 在进行volatile读的时候，要求要 **write(写入)一个变量的时候必需store(存储)，要存储的时候，必须要先从工作内存中assign（赋值）**，也就是做到了给一个变量赋值的时候一串关联指令直接把变量值写到主内存。就这样通过用的时候直接从主内存取，在赋值到直接写回主内存保证了写操作的内存可见性。
- 简单一句就是：**加了volatile关键字，就是将read-load-use指令关联成一个原子操作 以及assign-store-write也关联成一个原子操作** 。

**volatile既然一修改就是可见，为什么还不能保证原子性？**

这是因为在use和assign之间依然有极小的一段真空期，有可能变量会被其他线程读取，导致写丢失一次，是无论在哪一个时间点主内存的变量和任一工作内存的变量的值都是相等的。这个特性就导致了volatile变量不适合参与到依赖当前值的运算，如i = i + 1; i++;之类的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/d35aaa8dd37b46ca85b10d873cb80036.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/9ba6645cda4841388062101e0f1deb55.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
所以当有volatile变量依赖当前值的运算的过程如下图所示，在use和assign之间依然有极小的一段真空期，有可能变量会被其他线程读取，导致线程不安全，所以volatile不能保证原子性。
![在这里插入图片描述](https://img-blog.csdnimg.cn/14f032e483ba4400a41f4740fa08ea1e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

##### 1.6.5.3、指令禁重排
重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段，有时候会改变程序语句的先后顺序。

指令重排序它需要满足以下两个条件：
- 不改变程序运行的结果；
- 不存在数据依赖关系

不存在数据依赖关系，可以重排序；存在数据依赖关系，禁止重排序，重排后的指令绝对不能改变原有的串行语义！
 
**重排序的分类：**

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3种类型。
- 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
- 指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-LevelParallelism，ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
- 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行

**重排序的执行流程：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/3be79eac25be493988b4a7ab3b024314.png)
 
**数据依赖性：若两个操作访问同一变量，且这两个操作中有一个为写操作，此时两操作间就存在数据依赖性**。

 **案例 ：**![在这里插入图片描述](https://img-blog.csdnimg.cn/b682305243bb4a1dae48382c1bf7c048.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
存在数据依赖关系，禁止重排序===> 重排序发生，会导致程序运行结果不同。
编译器和处理器在重排序时，会遵守数据依赖性，不会改变存在依赖关系的两个操作的执行,但不同处理器和不同线程之间的数据性不会被编译器和处理器考虑，其只会作用于单处理器和单线程环境，下面三种情况，只要重排序两个操作的执行顺序，程序的执行结果就会被改变。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ce9cf0cd467e470a9f2c1757084a990d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**volatile有关的禁止指令重排的行为：**
- 当第一个操作为volatile读时，不论第二个操作是什么，都不能重排序。这个操作保证了volatile读之后的操作不会被重排到volatile读之前。
- 当第二个操作为volatile写时，不论第一个操作是什么，都不能重排序。这个操作保证了volatile写之前的操作不会被重排到volatile写之后。
- 当第一个操作为volatile写时，第二个操作为volatile读时，不能重排。

**四大屏障的插入情况：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/315f2ce5051341f4832cc2c099dd6c45.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 在每一个volatile写操作前面插入一个StoreStore屏障，StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作都已经刷新到主内存中。
- 在每一个volatile写操作后面插入一个StoreLoad屏障，StoreLoad屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序
![在这里插入图片描述](https://img-blog.csdnimg.cn/3d8919b68788445cadae29c4cb0d9f23.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

- 在每一个volatile读操作后面插入一个LoadLoad屏障，LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。
- 在每一个volatile读操作后面插入一个LoadStore屏障，LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4eb6ae23f799497ba09ea70a4117947e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.7、如何正确使用volatile
1. 单一赋值可以，but含复合运算赋值不可以(i++之类)使用volatile，不能保证原子性。
2. 状态标志，判断业务是否结束
```java
/**
 * 使用：作为一个布尔状态标志，用于指示发生了一个重要的一次性事件，例如完成初始化或任务结束
 * 理由：状态标志并不依赖于程序内任何其他状态，且通常只有一种状态转换
 * 例子：判断业务是否结束
 */
public class UseVolatileDemo
{
    private volatile static boolean flag = true;
	volatile int a = 10
    public static void main(String[] args)
    {
        new Thread(() -> {
            while(flag) {
                //do something......
            }
        },"t1").start();

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(2L); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            flag = false;
        },"t2").start();
    }
}
```

3. 开销较低的读，写锁策略
```java
 
public class UseVolatileDemo
{
    /**
     * 使用：当读远多于写，结合使用内部锁和 volatile 变量来减少同步的开销
     * 理由：利用volatile保证读取操作的可见性；利用synchronized保证复合操作的原子性
     */
    public class Counter
    {
        private volatile int value;

        public int getValue()
        {
            return value;   //利用volatile保证读取操作的可见性
              }
        public synchronized int increment()
        {
            return value++; //利用synchronized保证复合操作的原子性
               }
    }
}
 
```

#### 1.7.1、DCL双端锁的发布
**DCL双端锁单例模式**
```java
public class SafeDoubleCheckSingleton
{
    private static SafeDoubleCheckSingleton singleton;
    //私有化构造方法
    private SafeDoubleCheckSingleton(){
    }
    //双重锁设计
    public static SafeDoubleCheckSingleton getInstance(){
        if (singleton == null){
            //1.多线程并发创建对象时，会通过加锁保证只有一个线程能创建对象
            synchronized (SafeDoubleCheckSingleton.class){
                if (singleton == null){
                    //隐患：多线程环境下，由于重排序，该对象可能还未完成初始化就被其他线程读取，实际上返回null
                    singleton = new SafeDoubleCheckSingleton();
                }
            }
        }
        //2.对象创建完毕，执行getInstance()将不需要获取锁，直接返回创建对象
        return singleton;
    }
}
```
对象的创建过程
![在这里插入图片描述](https://img-blog.csdnimg.cn/e1ccbc91a325407da935b5d764d935f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

**存在的问题：**多线程环境下，在"问题代码处"，会执行如下操作，由于重排序导致2,3乱序，后果就是其他线程得到的是null而不是完成初始化的对象。
right
![在这里插入图片描述](https://img-blog.csdnimg.cn/ac2ae01377e349d283998d44ec15de54.png)
 problem
![在这里插入图片描述](https://img-blog.csdnimg.cn/fbee4f8aa0bf41b8aeee531c5d35f542.png)
**解决方式：**
1. 加volatile修饰

```java
public class SafeDoubleCheckSingleton
{
    //通过volatile声明，实现线程安全的延迟初始化。
    private volatile static SafeDoubleCheckSingleton singleton;
    //私有化构造方法
    private SafeDoubleCheckSingleton(){
    }
    //双重锁设计
    public static SafeDoubleCheckSingleton getInstance(){
        if (singleton == null){
            //1.多线程并发创建对象时，会通过加锁保证只有一个线程能创建对象
            synchronized (SafeDoubleCheckSingleton.class){
                if (singleton == null){
                    //隐患：多线程环境下，由于重排序，该对象可能还未完成初始化就被其他线程读取
                                      //原理:利用volatile，禁止 "初始化对象"(2) 和 "设置singleton指向内存空间"(3) 的重排序
                    singleton = new SafeDoubleCheckSingleton();
                }
            }
        }
        //2.对象创建完毕，执行getInstance()将不需要获取锁，直接返回创建对象
        return singleton;
    }
}
```

2. 采用静态内部类的方式实现

```java
//现在比较好的做法就是采用静态内部内的方式实现
public class SingletonDemo
{
    private SingletonDemo() { }
    private static class SingletonDemoHandler
    {
        private static SingletonDemo instance = new SingletonDemo();
    }
    public static SingletonDemo getInstance()
    {
        return SingletonDemoHandler.instance;
    }
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/8fb5b952ad2b4cfb81758bc3bf08f1b0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.8、最后的小总结
#### 1.8.1、内存屏障是什么
内存屏障 (Memory Barriers / Fences)：内存屏障其实就是一种JVM指令，是一类同步屏障指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作，避免代码重排序，Java内存模型的重排规则会要求Java编译器在生成JVM指令时插入特定的内存屏障指令，通过这些内存屏障指令，volatile实现了Java内存模型中的可见性和有序性，但volatile无法保证原子性。
#### 1.8.2、内存屏障能干嘛
- 阻止屏障两边的指令重排序
- 写数据时加入屏障，强制将线程私有工作内存的数据刷回主物理内存
- 读数据时加入屏障，线程私有工作内存的数据失效，重新到主物理内存中获取最新数据

#### 1.8.3、内存屏障四大指令
![在这里插入图片描述](https://img-blog.csdnimg.cn/5627f7c8f9f94768beb34ed1144d83fb.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

- 在每一个volatile写操作前面插入一个StoreStore屏障
- 在每一个volatile写操作后面插入一个StoreLoad屏障
- 在每一个volatile读操作后面插入一个LoadLoad屏障
- 在每一个volatile读操作后面插入一个LoadStore屏障
#### 1.8.4、凭什么我们java写了一个volatile关键字,系统底层加入内存屏障？两者关系怎么勾搭上的?
![在这里插入图片描述](https://img-blog.csdnimg.cn/64ebd46f78b54a5cab15143d79c72434.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
当某个字段加上volatile时，字节码中对应的Field的flags就会添加一个ACC_VOLATILE，当JVM将字节码生成为机器码的时候，发现操作是volatile的变量的话，就会根据JMM要求，在相应的位置去插入内存屏障指令


#### 1.8.5、volatile可见性
![在这里插入图片描述](https://img-blog.csdnimg.cn/63628997506c4634bed836d3231a1ac6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 1.8.6、volatile禁重排
![在这里插入图片描述](https://img-blog.csdnimg.cn/315f2ce5051341f4832cc2c099dd6c45.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- 在每一个volatile写操作前面插入一个StoreStore屏障，StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作都已经刷新到主内存中。
- 在每一个volatile写操作后面插入一个StoreLoad屏障，StoreLoad屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序
![在这里插入图片描述](https://img-blog.csdnimg.cn/3d8919b68788445cadae29c4cb0d9f23.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

- 在每一个volatile读操作后面插入一个LoadLoad屏障，LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。
- 在每一个volatile读操作后面插入一个LoadStore屏障，LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4eb6ae23f799497ba09ea70a4117947e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 1.8.7、对比java.util.concurrent.locks.Lock来理解
![在这里插入图片描述](https://img-blog.csdnimg.cn/9cb708b498e84f0c9de3752c2d9b50e8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
