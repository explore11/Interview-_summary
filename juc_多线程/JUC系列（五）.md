## 1、ThreadLocal
### 1.1、什么是ThreadLocal
线程局部变量。 
### 1.2、ThreadLocal的作用以及可以为什么保证线程安全？
多线程访问同一个共享变量的时候容易出现并发问题，特别是多个线程对一个变量进行写入的时候，为了保证线程安全，一般使用者在访问共享变量的时候需要进行额外的同步措施才能保证线程安全性。
ThreadLocal是除了加锁这种同步方式之外的一种规避多线程访问出现线程不安全的方法，当我们在创建一个变量后，每一个线程在访问ThreadLocal实例的时候，都有自己的、独立初始化的变量副本，且该副本只由当前线程自己使用，其它 Thread 不可访问，每个线程对其进行访问的时候，访问的都是线程自己的变量，从而避免了线程安全问题。

```java
class MovieTicket
{
    int number = 50;
    public synchronized void saleTicket()
    {
        if(number > 0)
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"号售票员卖出第： "+(number--));
        }else{
            System.out.println("--------卖完了");
        }
    }
}

class House
{
    ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

    public void saleHouse()
    {
        Integer value = threadLocal.get();
        value++;
        threadLocal.set(value);
    }
}

/**
 * 1  三个售票员卖完50张票务，总量完成即可，吃大锅饭，售票员每个月固定月薪
 * 2  分灶吃饭，各个销售自己动手，丰衣足食
 */
public class ThreadLocalDemo
{
    public static void main(String[] args)
    {
        /*MovieTicket movieTicket = new MovieTicket();

        for (int i = 1; i <=3; i++) {
            new Thread(() -> {
                for (int j = 0; j <20; j++) {
                    movieTicket.saleTicket();
                    try { TimeUnit.MILLISECONDS.sleep(10); } catch (InterruptedException e) { e.printStackTrace(); }
                }
            },String.valueOf(i)).start();
        }*/

        //===========================================
        House house = new House();

        new Thread(() -> {
            try {
                for (int i = 1; i <=3; i++) {
                    house.saleHouse();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"---"+house.threadLocal.get());
            }finally {
                house.threadLocal.remove();//如果不清理自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题
            }
        },"t1").start();

        new Thread(() -> {
            try {
                for (int i = 1; i <=2; i++) {
                    house.saleHouse();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"---"+house.threadLocal.get());
            }finally {
                house.threadLocal.remove();
            }
        },"t2").start();

        new Thread(() -> {
            try {
                for (int i = 1; i <=5; i++) {
                    house.saleHouse();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"---"+house.threadLocal.get());
            }finally {
                house.threadLocal.remove();
            }
        },"t3").start();
        System.out.println(Thread.currentThread().getName()+"\t"+"---"+house.threadLocal.get());
    }
}
```
#### 1.2.1、为什么线程执行完毕，要调用ThreadLocal的remove()方法
因为在线程池场景下，线程经常会被复用，如果不清理自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题。

### 1.3、非线程安全的SimpleDateFormat
#### 1.3.1、代码演示：
```java
public class DateUtils
{
    public static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    /**
     * 模拟并发环境下使用SimpleDateFormat的parse方法将字符串转换成Date对象
     * @param stringDate
     * @return
     * @throws Exception
     */
    public static Date parseDate(String stringDate)throws Exception
    {
        return sdf.parse(stringDate);
    }
    
    public static void main(String[] args) throws Exception
    {
        for (int i = 1; i <=30; i++) {
            new Thread(() -> {
                try {
                    System.out.println(DateUtils.parseDate("2020-11-11 11:11:11"));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            },String.valueOf(i)).start();
        }
    }
}
```
#### 1.3.2、出现的问题：

因为在项目上很多的地方，都需要用到时间转换的方法，所以我就时间转换抽出来了一个工具类，把SimpleDateFormat定义为一个静态属性，并提供了一个时间转换的方法，本地测试也是通过的。但是在真正使用的时候，有的时候会报出来莫名其妙的数据错误或者抛出异常的现象（for input String ""，empty String）。 
![在这里插入图片描述](https://img-blog.csdnimg.cn/38dda1953e7f4fd2b27d5c4eb66eb1fc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 1.3.3、原因：
SimpleDateFormat类内部有一个Calendar对象引用,它用来储存和这个SimpleDateFormat相关的日期信息，如果你的SimpleDateFormat是个static的, 那么多个thread 之间就会共享这个SimpleDateFormat，同时也是共享这个Calendar引用，但是在一个线程在解析完日期后，会清楚这个Calendar引用，所以也就导致了下一个线程进来进行日期转换的时候，报出没有格式信息的异常。
#### 1.3.4、解决方式
- 转换的方法加锁
```java
public class DateUtils {
    public static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    /* *
     * 模拟并发环境下使用SimpleDateFormat的parse方法将字符串转换成Date对象
     * @param stringDate
     * @return
     * @throws Exception
     */
    public static synchronized Date parseDate(String stringDate) throws Exception {
        return sdf.parse(stringDate);
    }

    public static void main(String[] args) throws Exception {
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                try {
                    System.out.println(DateUtils.parseDate("2020-11-11 11:11:11"));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```
- 将SimpleDateFormat定义成局部变量。

```java
public class DateUtils
{
    public static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    /**
     * 模拟并发环境下使用SimpleDateFormat的parse方法将字符串转换成Date对象
     * @param stringDate
     * @return
     * @throws Exception
     */
    public static Date parseDate(String stringDate)throws Exception
    {
        return sdf.parse(stringDate);
    }

    public static void main(String[] args) throws Exception
    {
        for (int i = 1; i <=30; i++) {
            new Thread(() -> {
                try {
                    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                    System.out.println(sdf.parse("2020-11-11 11:11:11"));
                    sdf = null;
                } catch (Exception e) {
                    e.printStackTrace();
                }
            },String.valueOf(i)).start();
        }
    }
}
```
- ThreadLocal，也叫做线程本地变量或者线程本地存储

```java
public class DateUtils {
    //2   ThreadLocal可以确保每个线程都可以得到各自单独的一个SimpleDateFormat的对象，那么自然也就不存在竞争问题了。
    public static final ThreadLocal<SimpleDateFormat> SIMPLE_DATE_FORMAT_THREAD_LOCAL = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
    public static String format(Date date) {
        return SIMPLE_DATE_FORMAT_THREAD_LOCAL.get().format(date);
    }
    public static Date parseDate(String datetime) throws ParseException {
        return SIMPLE_DATE_FORMAT_THREAD_LOCAL.get().parse(datetime);
    }
    public static void remove(){
        SIMPLE_DATE_FORMAT_THREAD_LOCAL.remove();
    }
    public static void main(String[] args) throws Exception {
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                try {
                    System.out.println(DateUtils.parseDate("2020-11-11 11:11:11"));
                } catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    DateUtils.remove();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```
- 使用DateTimeFormatter 代替 SimpleDateFormat

```java
 	//3 DateTimeFormatter 代替 SimpleDateFormat
    public static final DateTimeFormatter DATE_TIME_FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    public static String format(LocalDateTime localDateTime) {
        return DATE_TIME_FORMAT.format(localDateTime);
    }
    public static LocalDateTime parse(String dateString) {
        return LocalDateTime.parse(dateString, DATE_TIME_FORMAT);
    }
    public static void main(String[] args) throws Exception {
        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                try {
                    System.out.println(DateUtils.parse("2020-11-11 11:11:11"));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
    }
```


```java
package com.atguigu.juc.senior.utils;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * @auther zzyy
 * @create 2020-05-03 10:14
 */
public class DateUtils
{
    /*
    1   SimpleDateFormat如果多线程共用是线程不安全的类
    public static final SimpleDateFormat SIMPLE_DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static String format(Date date)
    {
        return SIMPLE_DATE_FORMAT.format(date);
    }

    public static Date parse(String datetime) throws ParseException
    {
        return SIMPLE_DATE_FORMAT.parse(datetime);
    }*/

    //2   ThreadLocal可以确保每个线程都可以得到各自单独的一个SimpleDateFormat的对象，那么自然也就不存在竞争问题了。
    public static final ThreadLocal<SimpleDateFormat> SIMPLE_DATE_FORMAT_THREAD_LOCAL = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String format(Date date)
    {
        return SIMPLE_DATE_FORMAT_THREAD_LOCAL.get().format(date);
    }

    public static Date parse(String datetime) throws ParseException
    {
        return SIMPLE_DATE_FORMAT_THREAD_LOCAL.get().parse(datetime);
    }


    //3 DateTimeFormatter 代替 SimpleDateFormat
    /*public static final DateTimeFormatter DATE_TIME_FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    public static String format(LocalDateTime localDateTime)
    {
        return DATE_TIME_FORMAT.format(localDateTime);
    }

    public static LocalDateTime parse(String dateString)
    {

        return LocalDateTime.parse(dateString,DATE_TIME_FORMAT);
    }*/
}
```

### 1.4、Thread，ThreadLocal，ThreadLocalMap关系

Thread里面有一个ThreadLocalMap类型的变量threadLocals，
ThreadLocal里面有一个静态内存类叫做ThreadLocalMap，ThreadLocalMap内部也有一个静态内部类叫做Entry，Entry是有K,V键值对构成
K是ThreadLocal(也就是当前线程)，V是Object类型的初始值

![在这里插入图片描述](https://img-blog.csdnimg.cn/cd74ab7be7544b7686256055ca661e28.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/5abca1742a174bb28aa15e287a4a324b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

### 1.5、ThreadLocal的key是弱引用，这是为什么？
#### 1.5.1、强引用
强引用是我们最常见的普通对象引用，只要还有强引用指向一个对象，就能表明对象还“活着”，垃圾收集器不会碰这种对象,就算是出现了OOM也不会对该对象进行回收。
在 Java 中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，即使该对象以后永远都不会被用到JVM也不会回收。因此强引用是造成Java内存泄漏的主要原因之一。
 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null，一般认为就是可以被垃圾收集的了(当然具体回收时机还是要看垃圾收集策略)。
 
```java
public static void strongReference()
{
    MyObject myObject = new MyObject();
    System.out.println("-----gc before: "+myObject);
    myObject = null;
    System.gc();
    try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
    System.out.println("-----gc after: "+myObject);
}
```
#### 1.5.2、软引用
软引用是一种相对强引用弱化了一些的引用，需要用java.lang.ref.SoftReference类来实现，可以让对象豁免一些垃圾收集。对于只有软引用的对象来说，当系统内存充足时它不会被回收，当系统内存不足时它会被回收。软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，**内存够用的时候就保留，不够用就回收！**

```java
class MyObject
{
    //一般开发中不用调用这个方法
    @Override
    protected void finalize() throws Throwable
    {
        System.out.println(Thread.currentThread().getName()+"\t"+"---finalize method invoked....");
    }
}

public class ReferenceDemo
{
    public static void main(String[] args)
    {
        //当我们内存不够用的时候，soft会被回收的情况，设置我们的内存大小：-Xms10m -Xmx10m
        SoftReference<MyObject> softReference = new SoftReference<>(new MyObject());

        System.gc();
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("-----gc after内存够用: "+softReference.get());

        try
        {
            byte[] bytes = new byte[9 * 1024 * 1024];
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            System.out.println("-----gc after内存不够: "+softReference.get());
        }
    }
}
```
#### 1.5.3、弱引用
 
弱引用需要用java.lang.ref.WeakReference类来实现，它比软引用的生存期更短，对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管JVM的内存空间是否足够，都会回收该对象占用的内存。 

```java
class MyObject
{
    //一般开发中不用调用这个方法
    @Override
    protected void finalize() throws Throwable
    {
        System.out.println(Thread.currentThread().getName()+"\t"+"---finalize method invoked....");
    }
}

public class ReferenceDemo
{
    public static void main(String[] args)
    {
        WeakReference<MyObject> weakReference = new WeakReference<>(new MyObject());
        System.out.println("-----gc before内存够用: "+weakReference.get());

        System.gc();
        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println("-----gc after内存够用: "+weakReference.get());
    }
}
```
软引用和弱引用的适用场景:
假如有一个应用需要读取大量的本地图片:

     如果每次读取图片都从硬盘读取则会严重影响性能,
     如果一次性全部加载到内存中又可能造成内存溢出。
 
此时使用软引用可以解决这个问题。
设计思路是：用一个HashMap来保存图片的路径和相应图片对象关联的软引用之间的映射关系，在内存不足时，JVM会自动回收这些缓存图片对象所占用的空间，从而有效地避免了OOM的问题。
 Map<String, SoftReference<Bitmap>> imageCache = new HashMap<String, SoftReference<Bitmap>>();


#### 1.5.4、虚引用
 
 虚引用需要java.lang.ref.PhantomReference类来实现。顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收，它不能单独使用也不能通过它访问对象，**虚引用必须和引用队列 (ReferenceQueue)联合使用。**
 
虚引用的主要作用是跟踪对象被垃圾回收的状态。 仅仅是提供了一种确保对象被 finalize以后，做某些事情的机制。 PhantomReference的get方法总是返回null，因此无法访问对应的引用对象。其意义在于：说明一个对象已经进入finalization阶段，可以被gc回收，用来实现比finalization机制更灵活的回收操作。换句话说，设置虚引用关联的唯一目的，就是在这个对象被收集器回收的时候收到一个系统通知或者后续添加进一步的处理。

```java
class MyObject
{
    //一般开发中不用调用这个方法
    @Override
    protected void finalize() throws Throwable
    {
        System.out.println(Thread.currentThread().getName()+"\t"+"---finalize method invoked....");
    }
}
public class ReferenceDemo
{
    public static void main(String[] args)
    {
        ReferenceQueue<MyObject> referenceQueue = new ReferenceQueue();
        PhantomReference<MyObject> phantomReference = new PhantomReference<>(new MyObject(),referenceQueue);
        //System.out.println(phantomReference.get());

        List<byte[]> list = new ArrayList<>();

        new Thread(() -> {
            while (true)
            {
                list.add(new byte[1 * 1024 * 1024]);
                try { TimeUnit.MILLISECONDS.sleep(600); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println(phantomReference.get());
            }
        },"t1").start();

        new Thread(() -> {
            while (true)
            {
                Reference<? extends MyObject> reference = referenceQueue.poll();
                if (reference != null) {
                    System.out.println("***********有虚对象加入队列了");
                }
            }
        },"t2").start();

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
    }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/a2edfaaeebe44dceb2055cf17d1076af.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

 

![在这里插入图片描述](https://img-blog.csdnimg.cn/0e3b4b7042ec42c2b0bd9dff34cc9cfd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/a060904b8cca4e29a7700a18fa993aa1.png)
每个Thread对象维护着一个ThreadLocalMap的引用，ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储
调用ThreadLocal的set()方法时，实际上就是往ThreadLocalMap设置值，key是ThreadLocal对象，值Value是传递进来的对象
调用ThreadLocal的get()方法时，实际上就是往ThreadLocalMap获取值，key是ThreadLocal对象
ThreadLocal本身并不存储值，它只是自己作为一个key来让线程从ThreadLocalMap获取value，正因为这个原理，所以ThreadLocal能够实现“数据隔离”，获取当前线程的局部变量值，不受其他线程影响～


#### 1.5.5、为什么ThreadLocalMap中的Entry的key源代码用弱引用?

当方法执行完毕后，栈帧销毁，ThreadLocal对象就应该被回收了，但此时线程的ThreadLocalMap里某个entry的key引用还指向这个对象，
若这个key引用是强引用，就会导致key指向的ThreadLocal对象及value指向的对象不能被gc回收，对象已经无用，但是变量占用的内存不能被回收，造成内存泄漏或者获取到上个线程遗留下来的value值，造成bug；
若这个key引用是弱引用就大概率会减少内存泄漏的问题，因为当threadLocal外部强引用被置为null,那么系统 GC 的时候，entry的key引用是弱引用，key被回收，根据可达性分析，这个threadLocal实例就没有任何一条链路能够引用到它，这个ThreadLocal势必会被回收。


![在这里插入图片描述](https://img-blog.csdnimg.cn/9dc6adfcd3b146f291569cbd7bbf6380.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

**但是弱引用不能100%保证内存不泄露**

因为如果是线程池的情况下，线程复用，ThreadLocalMap里某个entry的key为虚引用，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，ThreaLocalMap中的 Entry 没有被引用 value值不能回收，也会造成内存泄漏。
在ThreadLocal的get、set，remove方法都有清理entry中key为null的数据，所以说为了保证内存不泄露，在方法执行完毕之后，最好手动调用remove方法，去清除entry中key为null的数据。


![在这里插入图片描述](https://img-blog.csdnimg.cn/c956ccfb6f0b4baea9e23879e7b3e22e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 
**小总结：**
- ThreadLocal 并不解决线程间共享数据的问题
- ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景，
- ThreadLocal 通过隐式的在不同线程内创建独立实例副本避免了实例线程安全的问题，每个线程持有一个只属于自己的专属Map并维护了 ThreadLocal对象与具体实例的映射，该Map由于只被持有它的线程访问，故不存在线程安全以及锁的问题
- ThreadLocalMap的Entry对ThreadLocal的引用为弱引用，避免了ThreadLocal对象无法被回收的问题
- 都会通过expungeStaleEntry，cleanSomeSlots,replaceStaleEntry这三个方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏，属于安全加固的方法
 


## 2、Java对象内存布局和对象头
### 2.1、Object object = new Object()谈谈你对这句话的理解以及new一个对象占多少内存空间?
**markWord 8个字节   + 类型指针4个字节（默认情况下，JVM使用了压缩，显示的是4个字节）+无示例数据 + 对齐填充4个字节 =16 个字节
或者
markWord 8个字节   + 类型指针8个字节（无压缩）+无示例数据 +对齐填充0个字节=16 个字节**

对象布局分为Java对象布局和数组对象布局，数组对象布局是特殊的ava对象布局，组成结构在对象头的下面增加了数组长度的模块。
对象在堆内存中的存储布局：对象头、实例数据、对齐填充（保证8个字节的倍数）。
对象头分为对象标记（MarkWord）和类元信息(又叫类型指针)，类元信息存储的是指向该对象指向它的类型元数据的指针。
- 对象标记（MarkWord）：默认存储对象的HashCode、分代年龄和锁标志位等信息。这些信息都是与对象自身定义无关的数据，所以MarkWord被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间MarkWord里存储的数据会随着锁标志位的变化而变化。

- 类元信息：类元信息存储的是指向该对象指向它的类型元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。


![在这里插入图片描述](https://img-blog.csdnimg.cn/7edfab242b164a90ba48f84d075e71ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)


![在这里插入图片描述](https://img-blog.csdnimg.cn/6a4007799e05495b9546ced345e8f9e7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/01200b2bc915441ea8ea91348432dcfc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)


![在这里插入图片描述](https://img-blog.csdnimg.cn/a9bfb20dfee04ecba4ed5e8a993bd754.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

GC年龄采用4位bit存储，最大为15（1111），例如MaxTenuringThreshold参数默认值就是15.
![在这里插入图片描述](https://img-blog.csdnimg.cn/498ef255940a4075aee9d81a1264cb2f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
- hash： 保存对象的哈希码
- age： 保存对象的分代年龄
- biased_lock： 偏向锁标识位
- lock： 锁状态标识位
- JavaThread* ：保存持有偏向锁的线程ID
- epoch： 保存偏向时间戳
 



**对象头多大**：在64位系统中，Mark Word占了8个字节，类型指针占了8个字节（默认情况下，JVM使用了压缩，显示的是4个字节，-XX:+UseCompressedClassPointers），一共是16或者12个字节。
**实例数据**：存放类的属性(Field)数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
**对齐填充**：虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这部分内存按8字节补充对齐。

```xml
<!--
官网：http://openjdk.java.net/projects/code-tools/jol/ 定位：分析对象在JVM的大小和分布
-->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>
```

```java
public class MyObject
{
    public static void main(String[] args) {
     	//VM的细节详细情况
        System.out.println(VM.current().details());
        //所有的对象分配的字节都是8的整数倍。
        System.out.println(VM.current().objectAlignment());
        
        Object o = new Object();
        System.out.println( ClassLayout.parseInstance(o).toPrintable());
    }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/ce8fb61817024300b6de0d18b6da6e51.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/a6ea4e1ec0b3401cb2946434aa0cbafa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
















