## 1、CAS
### 1.1、没有CAS之前，保证线程安全的方式
- **多线程环境不使用原子类保证线程安全（基本数据类型）**
```java
public class T3
{
    volatile int number = 0;
    //读取
    public int getNumber()
    {
        return number;
    }
    //写入加锁保证原子性
    public synchronized void setNumber()
    {
        number++;
    }
}
 
```

- **多线程环境    使用原子类保证线程安全（基本数据类型）**

```java
public class T3
{
    volatile int number = 0;
    //读取
    public int getNumber()
    {
        return number;
    }
    //写入加锁保证原子性
    public synchronized void setNumber()
    {
        number++;
    }
    //=================================
    AtomicInteger atomicInteger = new AtomicInteger();

    public int getAtomicInteger()
    {
        return atomicInteger.get();
    }

    public void setAtomicInteger()
    {
        atomicInteger.getAndIncrement();
    }


}
 
```
### 1.2、CAS及底层实现，原子操作类是如何保证线程安全的？
```java
public class CASDemo
{
  public static void main(String[] args) {
        AtomicInteger atomicInteger =new AtomicInteger();
        System.out.println(atomicInteger.getAndIncrement());
    }
}
```


拿atomicInteger.getAndIncrement()方法的源码来说吧，该方法的底层实际上调用的是unsafe类的getAndAddInt方法，在AtomicInteger中使用volatile修饰的value来保证可见性和有序性，使用unsafe类的CAS操作+自旋锁（do..while）保证原子性。

1.  atomicInteger.getAndIncrement()的底层调用的是unsafe类的getAndAddInt方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/266b5a231f4a4a71a6bc2e794ba9c31a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
2. 用volatile修饰的value来保证可见性和有序性，使用unsafe类的方法保证原子性。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b67befdbf66447c98f4175fbb22935ae.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
因为Java中的CAS操作的执行依赖于Unsafe类的方法，所以说Unsafe类是CAS操作的核心类，在其内部方法可以通过指针来直接操作内存，由于Java方法无法直接访问底层系统，需要通过Unsafe类的本地（Native）方法来进行访问，通过该类可以直接操作特定的内存数据。

CAS：compare and swap的缩写，比较并交换，是实现并发算法时常用到的一种技术，它包含三个操作数——内存位置、预期原值及更新值，在执行CAS操作的时候，将内存位置的值与预期原值比较：如果相匹配，那么处理器会自动将该位置值更新为新值，如果不匹配，处理器不做任何操作，多个线程同时执行CAS操作只有一个会成功。
![在这里插入图片描述](https://img-blog.csdnimg.cn/023957808b674883ab0367d0a6637d0d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

CAS操作是JDK提供的非阻塞原子性操作，它通过硬件保证了比较-更新的原子性。在CAS操作的底层是依赖于CPU来完成的，调用UnSafe类中的CAS方法，JVM会帮我们实现出CAS汇编指令，它是一条CPU系统原语（原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题），真正实现操作的是CPU的原子指令（cmpxchg指令），在执行cmpxchg指令的时候，会判断当前系统是否为多核系统，如果是就给总线加锁，只有一个线程会对总线加锁成功，加锁成功之后会执行cas操作，也就是说CAS的原子性实际上是CPU实现的， 其实在这一点上还是有排他锁的，只是比起用synchronized， 这里的排他时间要短的多， 所以在多线程情况下性能会比较好。  

Unsafe类中的compareAndSwapInt，是一个本地方法，该方法的实现位于unsafe.cpp中
```java
 UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))  
 // 先想办法拿到变量value在内存中的地址，根据偏移量valueOffset，计算 value 的地址  jint* addr = (jint *) 
 UnsafeWrapper("Unsafe_CompareAndSwapInt");  
 oop p = JNIHandles::resolve(obj);
 index_oop_from_field_offset_long(p, offset);
 return (jint)(Atomic::cmpxchg(x, addr, e)) == e;UNSAFE_END
 // 调用 Atomic 中的函数 cmpxchg来进行比较交换，其中参数x是即将更新的值，参数e是原内存的值 
(Atomic::cmpxchg(x, addr, e)) == e;
 
```

```java


// 调用 Atomic 中的函数 cmpxchg来进行比较交换，其中参数x是即将更新的值，参数e是原内存的值
return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
unsigned Atomic::cmpxchg(unsigned int exchange_value,volatile unsigned int* dest, unsigned int compare_value) 
{    
	assert(sizeof(unsigned int) == sizeof(jint), "more work to do");  
	/** 根据操作系统类型调用不同平台下的重载函数，这个在预编译期间编译器会决定调用哪个平台下的重载函数*/   
	return (unsigned int)Atomic::cmpxchg((jint)exchange_value, 
	(volatile jint*)dest, (jint)compare_value);
}
 
```

```java
 inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) 
 { 
  //判断是否是多核CPU  
  int mp = os::is_MP();  __asm {    
  //三个move指令表示的是将后面的值移动到前面的寄存器上    
  mov edx, dest    
  mov ecx, exchange_value    
  mov eax, compare_value    
  //CPU原语级别，CPU触发    
  LOCK_IF_MP(mp)    
  //比较并交换指令    
  //cmpxchg: 即“比较并交换”指令    
  //dword: 全称是 double word 表示两个字，一共四个字节    
  //ptr: 全称是 pointer，与前面的 dword 连起来使用，表明访问的内存单元是一个双字单元     
  //将 eax 寄存器中的值（compare_value）与 [edx] 双字内存单元中的值进行对比，    
  //如果相同，则将 ecx 寄存器中的值（exchange_value）存入 [edx] 内存单元中   
   cmpxchg dword ptr [edx], ecx  
   }
   }
到这里应该理解了CAS真正实现的机制了，它最终是由操作系统的汇编指令完成的。
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/3f5fe961405842208a417e0fb5c65bdc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 
var5：就是我们从主内存中拷贝到工作内存中的值(每次都要从主内存拿到最新的值到自己的本地内存，然后执行compareAndSwapInt()在再和主内存的值进行比较。因为线程不可以直接越过高速缓存，直接操作主内存，所以执行上述方法需要比较一次，在执行加1操作)

那么操作的时候，需要比较从主内存中拷贝工作内存中的值与当前主内存中的值进行比较

假设执行 compareAndSwapInt返回false，那么就一直执行 while方法，直到期望的值和真实值一样

- val1：AtomicInteger对象本身
- var2：该对象值得引用地址（内存偏移量）
- var4：需要变动的数量
- var5：主内存中拷贝工作内存中的值
  - 用当前对象和内存偏移量找到当前主内存中的值与var5比较
  - 如果相同，更新var5 + var4 并返回true
  - 如果不同，继续取值然后再比较，直到更新完成

这里没有用synchronized，而用CAS，这样提高了并发性，也能够实现一致性，是因为每个线程进来后，进入的do while循环，然后不断的获取内存中的值，判断是否为最新，然后在进行更新操作。


案例：假设线程A和线程B同时执行getAndInt操作（分别跑在不同的CPU上）

1. AtomicInteger里面的value原始值为3，即主内存中AtomicInteger的 value 为3，根据JMM模型，线程A和线程B各自持有一份价值为3的副本，分别存储在各自的工作内存
2. 线程A通过getIntVolatile(var1 , var2) 拿到value值3，这时线程A被挂起（该线程失去CPU执行权）
3. 线程B也通过getIntVolatile(var1, var2)方法获取到value值也是3，此时刚好线程B没有被挂起，并执行了compareAndSwapInt方法，比较内存的值也是3，成功修改内存值为4，线程B执行完毕。
4. 这时线程A恢复，执行CAS方法，比较发现自己手里的数字3和主内存中的数字4不一致，说明该值已经被其它线程抢先一步修改过了，那么A线程本次修改失败，只能够从主内存中重新读取变量值后，再次重新来一遍，也就是在执行do while
5. 线程A重新获取value值，因为变量value被volatile修饰，所以其它线程对它的修改，线程A总能够看到，线程A继续执compareAndSwapInt进行比较并交换，直到成功。



CAS是靠硬件实现的从而在硬件层面提升效率，最底层还是交给硬件来保证原子性和可见性，实现方式是基于硬件平台的汇编指令，在intel的CPU中(X86机器上)，使用的是汇编指令cmpxchg指令。 
核心思想就是：比较要更新变量的值V和预期值E（compare），相等才会将V的值设为新值N（swap）如果不相等自旋再来。

### 1.3、CAS缺点

CAS不加锁，保证一次性，但是需要多次比较

- 循环时间长，开销大（因为执行的是do while，如果比较不成功一直在循环，最差的情况，就是某个线程一直取到的值和预期值都不一样，这样就会无限循环，出现锁饥饿现象）
- 只能保证一个共享变量的原子操作
  1. 当对一个共享变量执行操作时，我们可以通过循环CAS的方式来保证原子操作
  2. 但是对于多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候只能用锁来保证原子性
- 引出来ABA问题？

### 1.4、ABA问题
![在这里插入图片描述](https://img-blog.csdnimg.cn/28c236e1a87c484794ea222eb1de9674.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_10,color_FFFFFF,t_70,g_se,x_16)
假设现在有两个线程，分别是t1 和 t2，去修改共享变量A的值，当线程t1 ，将共享变量A的值拷贝到自己的工作内存中时，线程被挂起，然后线程t2，去修改共享变量A的值，先将A的值修改为2，再将A等于2修改会1，操作完毕之后，线层t1，执行操作，判断主内存中的值和自己期望的值相等，然后就将该值进行了更新操作。

所以说ABA问题就是，在一个线程进行从主内存中获取变量值，并将修改完的内存值写入主内存的时候，其中主内存中的变量值已经被修改了N次，但是最终又改成原来的值了，导致该线程在进行比较并交换的时候，没有发现该变量值被更改过，然后成功更新变量值。

### 1.5、原子引用
原子引用其实和原子包装类是差不多的概念，就是将一个java类，用原子引用类进行包装起来，那么这个类就具备了原子性
```java
@Getter
@ToString
@AllArgsConstructor
class User
{
    String userName;
    int    age;
}

public class AtomicReferenceDemo
{
    public static void main(String[] args)
    {
        User z3 = new User("z3",24);
        User li4 = new User("li4",26);

        AtomicReference<User> atomicReferenceUser = new AtomicReference<>();

        atomicReferenceUser.set(z3);
        System.out.println(atomicReferenceUser.compareAndSet(z3,li4)+"\t"+atomicReferenceUser.get().toString());
        System.out.println(atomicReferenceUser.compareAndSet(z3,li4)+"\t"+atomicReferenceUser.get().toString());
    }
}
```
### 1.6、ABA问题的解决，AtomicStampedReference
新增一种机制，也就是修改版本号，类似于时间戳的概念。时间戳原子引用，来这里应用于版本号的更新，也就是每次更新的时候，需要比较期望值和当前值，以及期望版本号和当前版本号

```java
public class ABADemo
{
    static AtomicInteger atomicInteger = new AtomicInteger(100);
    static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(100,1);

    public static void main(String[] args)
    {
        new Thread(() -> {
            atomicInteger.compareAndSet(100,101);
            atomicInteger.compareAndSet(101,100);
        },"t1").start();

        new Thread(() -> {
            //暂停一会儿线程
            try { Thread.sleep( 500 ); } catch (InterruptedException e) { e.printStackTrace(); };
            System.out.println(atomicInteger.compareAndSet(100, 2019)+"\t"+atomicInteger.get());
        },"t2").start();

        //暂停一会儿线程,main彻底等待上面的ABA出现演示完成。
        try { Thread.sleep( 2000 ); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println("============以下是ABA问题的解决=============================");

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"\t 首次版本号:"+stamp);//1
            //暂停一会儿线程,
            try { Thread.sleep( 1000 ); } catch (InterruptedException e) { e.printStackTrace(); }
            atomicStampedReference.compareAndSet(100,101,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t 2次版本号:"+atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101,100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t 3次版本号:"+atomicStampedReference.getStamp());
        },"t3").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"\t 首次版本号:"+stamp);//1
            //暂停一会儿线程，获得初始值100和初始版本号1，故意暂停3秒钟让t3线程完成一次ABA操作产生问题
            try { Thread.sleep( 3000 ); } catch (InterruptedException e) { e.printStackTrace(); }
            boolean result = atomicStampedReference.compareAndSet(100,2019,stamp,stamp+1);
            System.out.println(Thread.currentThread().getName()+"\t"+result+"\t"+atomicStampedReference.getReference());
        },"t4").start();
    }
}
	
```



## 2、原子操作类之18罗汉增强
### 2.1、原子操作类之分类
- 基本类型原子类
	1. AtomicInteger
	2. AtomicBoolean
	3. AtomicLong
- 数组类型原子类
	1. AtomicIntegerArray
	2. AtomicLongArray
	3. AtomicReferenceArray
- 引用类型原子类
	1. AtomicReference
	2. AtomicStampedReference
	3. AtomicMarkableReference
- 对象的属性修改原子类
	1. AtomicIntegerFieldUpdater
	2. AtomicLongFieldUpdater
	3. AtomicReferenceFieldUpdater
- 原子操作增强类原理深度解析
	1. DoubleAccumulator
	2. DoubleAdder
	3. LongAccumulator
	4. LongAdder
	5. Striped64
	6. Number
	

#### 2.1.1、基本类型原子类
##### 2.1.1.1、常用API简介
- public final int get() //获取当前的值
- public final int getAndSet(int newValue)//获取当前的值，并设置新的值
- public final int getAndIncrement()//获取当前的值，并自增
- public final int getAndDecrement() //获取当前的值，并自减
- public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
- boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）

```java
class MyNumber
{
    @Getter
    private AtomicInteger atomicInteger = new AtomicInteger();
    public void addPlusPlus()
    {
        atomicInteger.incrementAndGet();
    }
}

public class AtomicIntegerDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        MyNumber myNumber = new MyNumber();
        CountDownLatch countDownLatch = new CountDownLatch(100);

        for (int i = 1; i <=100; i++) {
            new Thread(() -> {
                try
                {
                    for (int j = 1; j <=5000; j++)
                    {
                        myNumber.addPlusPlus();
                    }
                }finally {
                    countDownLatch.countDown();
                }
            },String.valueOf(i)).start();
        }

        countDownLatch.await();

        System.out.println(myNumber.getAtomicInteger().get());
    }
}
```
#### 2.1.2、数组类型原子类

##### 2.1.2.1、常用API简介
-  new AtomicIntegerArray(10) 创建给定长度的新 AtomicIntegerArray。
- set():将位置 i 的元素设置为给定值
- length()方法：返回该数组的长度
- addAndGet()方法：以原子方式先对给定下标加上特定的值，再获取相加后的值
- compareAndSet()方法：如果当前值 == 预期值，则以原子方式将位置 i 的元素设置为给定的更新值。
- decrementAndGet()方法：以原子方式先将当前下标的值减1，再获取减1后的结果
- getAndAdd()方法：以原子方式先获取当前下标的值，再将当前下标的值加上给定的值
- getAndDecrement()方法：以原子方式先获取当前下标的值，再对当前下标的值减1
- getAndIncrement()方法：以原子方式先获取当前下标的值，再对当前下标的值加1
- getAndSet()方法：将位置 i 的元素以原子方式设置为给定值，并返回旧值。
- incrementAndGet()方法：以原子方式先对下标加1再获取值

```java
public class AtomicIntegerArrayTest {
    public static void main(String[] args) {
        //1、创建给定长度的新 AtomicIntegerArray。
        AtomicIntegerArray atomicIntegerArray = new AtomicIntegerArray(10);
        //2、将位置 i 的元素设置为给定值,默认值为0
        atomicIntegerArray.set(9, 10);
        System.out.println("Value: " + atomicIntegerArray.get(9) + "默认值：" + atomicIntegerArray.get(0));

        //3、返回该数组的长度
        AtomicIntegerArray atomicIntegerArray1 = new AtomicIntegerArray(10);
        System.out.println("数组长度：" + atomicIntegerArray1.length());

        //4、以原子方式先对给定下标加上特定的值，再获取相加后的值
        AtomicIntegerArray atomicIntegerArray2 = new AtomicIntegerArray(10);
        atomicIntegerArray2.set(5, 10);
        System.out.println("Value: " + atomicIntegerArray2.get(5));
        atomicIntegerArray2.addAndGet(5, 10);
        System.out.println("Value: " + atomicIntegerArray2.get(5));


        //5、如果当前值 == 预期值，则以原子方式将位置 i 的元素设置为给定的更新值。
        AtomicIntegerArray atomicIntegerArray3 = new AtomicIntegerArray(10);
        atomicIntegerArray3.set(5, 10);
        System.out.println("当前值： " + atomicIntegerArray3.get(5));
        Boolean bool = atomicIntegerArray3.compareAndSet(5, 10, 30);
        System.out.println("结果值： " + atomicIntegerArray3.get(5) + " Result: " + bool);

        //6、以原子方式先将当前下标的值减1，再获取减1后的结果
        AtomicIntegerArray atomicIntegerArray4 = new AtomicIntegerArray(10);
        atomicIntegerArray4.set(5, 10);
        System.out.println("下标为5的值为：" + atomicIntegerArray4.get(5));
        Integer result1 = atomicIntegerArray4.decrementAndGet(5);
        System.out.println("result1的值为：" + result1);
        System.out.println("下标为5的值为：" + atomicIntegerArray4.get(5));

        //7、以原子方式先获取当前下标的值，再将当前下标的值加上给定的值
        AtomicIntegerArray atomicIntegerArray5 = new AtomicIntegerArray(10);
        atomicIntegerArray5.set(5, 10);
        Integer result2 = atomicIntegerArray5.getAndAdd(5, 5);
        System.out.println("result2的值为：" + result2);
        System.out.println("下标为5的值为：" + atomicIntegerArray5.get(5));

        //8、 以原子方式先获取当前下标的值，再对当前下标的值减1
        AtomicIntegerArray atomicIntegerArray6 = new AtomicIntegerArray(10);
        atomicIntegerArray6.set(1, 10);
        System.out.println("下标为1的值为：" + atomicIntegerArray6.get(1));
        Integer result3 = atomicIntegerArray6.getAndDecrement(1);
        System.out.println("result3的值为：" + result3);
        System.out.println("下标为1的值为：" + atomicIntegerArray6.get(1));

        //9、 以原子方式先获取当前下标的值，再对当前下标的值加1
        AtomicIntegerArray atomicIntegerArray7 = new AtomicIntegerArray(10);
        atomicIntegerArray7.set(2, 10);
        System.out.println("下标为2的值为：" + atomicIntegerArray7.get(2));
        Integer result4 = atomicIntegerArray7.getAndIncrement(2);
        System.out.println("result4的值为：" + result4);
        System.out.println("下标为2的值为：" + atomicIntegerArray7.get(2));

        //10、将位置 i 的元素以原子方式设置为给定值，并返回旧值。
        AtomicIntegerArray atomicIntegerArray8 = new AtomicIntegerArray(10);
        atomicIntegerArray8.set(3, 10);
        System.out.println("下标为3的值为：" + atomicIntegerArray8.get(3));
        Integer result5 = atomicIntegerArray8.getAndSet(3, 50);
        System.out.println("result5的值为：" + result5);
        System.out.println("下标为3的值为：" + atomicIntegerArray8.get(3));

        //11、 以原子方式先对下标加1再获取值
        AtomicIntegerArray atomicIntegerArray9 = new AtomicIntegerArray(10);
        atomicIntegerArray9.set(4, 10);
        System.out.println("下标为4的值为：" + atomicIntegerArray9.get(4));
        Integer result6 = atomicIntegerArray9.incrementAndGet(4);
        System.out.println("result6的值为：" + result6);
        System.out.println("下标为4的值为：" + atomicIntegerArray9.get(4));

    }
}
```
#### 2.1.3、引用类型原子类
##### 2.1.3.1、AtomicReference
原子引用其实和原子包装类是差不多的概念，就是将一个java类，用原子引用类进行包装起来，那么这个类就具备了原子性
```java
package com.atguigu.Interview.study.thread;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.ToString;

import java.util.concurrent.atomic.AtomicReference;

@Getter
@ToString
@AllArgsConstructor
class User
{
    String userName;
    int    age;
}

public class AtomicReferenceDemo
{
    public static void main(String[] args)
    {
        User z3 = new User("z3",24);
        User li4 = new User("li4",26);
        AtomicReference<User> atomicReferenceUser = new AtomicReference<>();
        atomicReferenceUser.set(z3);
        System.out.println(atomicReferenceUser.compareAndSet(z3,li4)+"\t"+atomicReferenceUser.get().toString());
        System.out.println(atomicReferenceUser.compareAndSet(z3,li4)+"\t"+atomicReferenceUser.get().toString());
    }
}
```
##### 2.1.3.2、AtomicStampedReference
携带版本号的引用类型原子类，可以解决ABA问题，时间戳原子引用，来这里应用于版本号的更新，也就是每次更新的时候，需要比较期望值和当前值，以及期望版本号和当前版本号

```java
public class ABADemo
{
    static AtomicInteger atomicInteger = new AtomicInteger(100);
    static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(100,1);

    public static void main(String[] args)
    {
        abaProblem();
        abaResolve();
    }

    public static void abaResolve()
    {
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println("t3 ----第1次stamp  "+stamp);
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            atomicStampedReference.compareAndSet(100,101,stamp,stamp+1);
            System.out.println("t3 ----第2次stamp  "+atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101,100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp()+1);
            System.out.println("t3 ----第3次stamp  "+atomicStampedReference.getStamp());
        },"t3").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println("t4 ----第1次stamp  "+stamp);
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            boolean result = atomicStampedReference.compareAndSet(100, 20210308, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName()+"\t"+result+"\t"+atomicStampedReference.getReference());
        },"t4").start();
    }

    public static void abaProblem()
    {
        new Thread(() -> {
            atomicInteger.compareAndSet(100,101);
            atomicInteger.compareAndSet(101,100);
        },"t1").start();

        try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            atomicInteger.compareAndSet(100,20210308);
            System.out.println(atomicInteger.get());
        },"t2").start();
    }
}
```

##### 2.1.3.2、AtomicMarkableReference
原子更新带有标记位的引用类型对象，判断是否修改过，它的定义就是将状态戳简化为true|false，状态标记(true/false)原子引用

```java
public class ABADemo
{
    static AtomicInteger atomicInteger = new AtomicInteger(100);
    static AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100,1);
    static AtomicMarkableReference<Integer> markableReference = new AtomicMarkableReference<>(100,false);

    public static void main(String[] args)
    {
        new Thread(() -> {
            atomicInteger.compareAndSet(100,101);
            atomicInteger.compareAndSet(101,100);
            System.out.println(Thread.currentThread().getName()+"\t"+"update ok");
        },"t1").start();

        new Thread(() -> {
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            atomicInteger.compareAndSet(100,2020);
        },"t2").start();

        //暂停几秒钟线程
        try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println(atomicInteger.get());

        System.out.println();
        System.out.println();
        System.out.println();

        System.out.println("============以下是ABA问题的解决,让我们知道引用变量中途被更改了几次=========================");
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName()+"\t 1次版本号"+stampedReference.getStamp());
            //故意暂停200毫秒，让后面的t4线程拿到和t3一样的版本号
            try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }

            stampedReference.compareAndSet(100,101,stampedReference.getStamp(),stampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t 2次版本号"+stampedReference.getStamp());
            stampedReference.compareAndSet(101,100,stampedReference.getStamp(),stampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t 3次版本号"+stampedReference.getStamp());
        },"t3").start();

        new Thread(() -> {
            int stamp = stampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"\t =======1次版本号"+stamp);
            //暂停2秒钟,让t3先完成ABA操作了，看看自己还能否修改
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            boolean b = stampedReference.compareAndSet(100, 2020, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName()+"\t=======2次版本号"+stampedReference.getStamp()+"\t"+stampedReference.getReference());
        },"t4").start();

        System.out.println();
        System.out.println();
        System.out.println();

        System.out.println("============AtomicMarkableReference不关心引用变量更改过几次，只关心是否更改过======================");

        new Thread(() -> {
            boolean marked = markableReference.isMarked();
            System.out.println(Thread.currentThread().getName()+"\t 1次版本号"+marked);
            try { TimeUnit.MILLISECONDS.sleep(100); } catch (InterruptedException e) { e.printStackTrace(); }
            markableReference.compareAndSet(100,101,marked,!marked);
            System.out.println(Thread.currentThread().getName()+"\t 2次版本号"+markableReference.isMarked());
            markableReference.compareAndSet(101,100,markableReference.isMarked(),!markableReference.isMarked());
            System.out.println(Thread.currentThread().getName()+"\t 3次版本号"+markableReference.isMarked());
        },"t5").start();

        new Thread(() -> {
            boolean marked = markableReference.isMarked();
            System.out.println(Thread.currentThread().getName()+"\t 1次版本号"+marked);
            //暂停几秒钟线程
            try { TimeUnit.MILLISECONDS.sleep(100); } catch (InterruptedException e) { e.printStackTrace(); }
            markableReference.compareAndSet(100,2020,marked,!marked);         System.out.println(Thread.currentThread().getName()+"\t"+markableReference.getReference()+"\t"+markableReference.isMarked());
        },"t6").start();
    }
}
```
#### 2.1.4、对象的属性修改原子类
**使用要求：**
- 更新的对象属性必须使用 public volatile 修饰符。
- 因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须，使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。
##### 2.1.4.1、AtomicIntegerFieldUpdater、AtomicILongFieldUpdater
以一种线程安全的方式操作非线程安全对象内的某些字段，原子更新对象中int类型字段的值、原子更新对象中Long类型字段的值

```java
class BankAccount
{
    private String bankName = "CCB";//银行
    public volatile int money = 0;//钱数
    AtomicIntegerFieldUpdater<BankAccount> accountAtomicIntegerFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(BankAccount.class,"money");

    //不加锁+性能高，局部微创
    public void transferMoney(BankAccount bankAccount)
    {
        accountAtomicIntegerFieldUpdater.incrementAndGet(bankAccount);
    }
}

/**
 * 以一种线程安全的方式操作非线程安全对象的某些字段。
 * 需求：
 * 1000个人同时向一个账号转账一元钱，那么累计应该增加1000元，
 * 除了synchronized和CAS,还可以使用AtomicIntegerFieldUpdater来实现。
 */
public class AtomicIntegerFieldUpdaterDemo
{

    public static void main(String[] args)
    {
        BankAccount bankAccount = new BankAccount();

        for (int i = 1; i <=1000; i++) {
            int finalI = i;
            new Thread(() -> {
                bankAccount.transferMoney(bankAccount);
            },String.valueOf(i)).start();
        }

        //暂停毫秒
        try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }

        System.out.println(bankAccount.money);

    }
} 
```

##### 2.1.4.2、AtomicReferenceFieldUpdater
以一种线程安全的方式操作非线程安全对象内的某些字段，原子更新自定义对象中字段的值

```java
class MyVar
{
    public volatile Boolean isInit = Boolean.FALSE;
    AtomicReferenceFieldUpdater<MyVar,Boolean> atomicReferenceFieldUpdater = AtomicReferenceFieldUpdater.newUpdater(MyVar.class,Boolean.class,"isInit");
    public void init(MyVar myVar)
    {
        if(atomicReferenceFieldUpdater.compareAndSet(myVar,Boolean.FALSE,Boolean.TRUE))
        {
            System.out.println(Thread.currentThread().getName()+"\t"+"---init.....");
            //暂停几秒钟线程
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(Thread.currentThread().getName()+"\t"+"---init.....over");
        }else{
            System.out.println(Thread.currentThread().getName()+"\t"+"------其它线程正在初始化");
        }
    }
}

/**
 * 多线程并发调用一个类的初始化方法，如果未被初始化过，将执行初始化工作，要求只能初始化一次
 */
public class AtomicIntegerFieldUpdaterDemo
{
    public static void main(String[] args) throws InterruptedException
    {
        MyVar myVar = new MyVar();
        for (int i = 1; i <=5; i++) {
            new Thread(() -> {
                myVar.init(myVar);
            },String.valueOf(i)).start();
        }
    }
}
```

#### 2.1.5、原子操作增强类以及原理深度解析
**案例demo**
- 1    
- 2    一个很大的List，里面都是int类型，如何实现加加，说说思路
##### 2.1.5.1、DoubleAdder、LongAdder
**常用API:**
- add​(long x)	添加给定值。
- void	decrement()	 相当于 add(-1) 。
- double	doubleValue()	在扩展原始转换后，将 sum()作为 double返回。
- float	floatValue()	在扩展基元转换后，将 sum()作为 float返回。
- void	increment()	相当于 add(1) 。
- int	intValue()	 在缩小基元转换后，将 sum()作为 int返回。
- long	longValue()	相当于 sum() 。
- void	reset()	重置保持总和为零的变量。
- long	sum()	返回当前总和。
- long	sumThenReset()	相当于 sum()，后跟 reset() 。
- String	toString()	返回 sum()的String表示 形式 。

```java
public class Demo{
    public static void main(String[] args) {

        //add​(long x)	添加给定值。
        LongAdder longAdder = new LongAdder();
        longAdder.add(10L);
        System.out.println(longAdder.longValue());
        //void	decrement()	 相当于 add(-1) 。
        longAdder.decrement();
        System.out.println(longAdder.longValue());
        //double	doubleValue()	在扩展原始转换后，将 sum()作为 double返回。
        System.out.println(longAdder.doubleValue());
        //float	floatValue()	在扩展基元转换后，将 sum()作为 float返回。
        System.out.println(longAdder.floatValue());
        //void	increment()	相当于 add(1) 。
        longAdder.increment();
        System.out.println(longAdder.longValue());
        //int	intValue()	 在缩小基元转换后，将 sum()作为 int返回。
        System.out.println(longAdder.intValue());
        //long	longValue()	相当于 sum() 。
        System.out.println(longAdder.longValue());
        //void	reset()	重置保持总和为零的变量。
        longAdder.reset();
        System.out.println(longAdder.intValue());
        //long	sum()	返回当前总和。
        System.out.println(longAdder.sum());
        //long	sumThenReset()	相当于 sum()，后跟 reset() 。
        longAdder.sumThenReset();
        System.out.println(longAdder.intValue());
        //String	toString()	返回 sum()的String表示 形式 。
        System.out.println(longAdder.toString());

    }
}
```

##### 2.1.5.2、DoubleAccumulator、LongAccumulator
**有了LongAdder，为什么还要有LongAccumulator？**
因为LongAdder只能用来计算加法，且从零开始计算，LongAccumulator提供了自定义的函数操作

```java
public class LongAdderAPIDemo
{
    public static void main(String[] args)
    {
        LongAdder longAdder = new LongAdder();
        longAdder.increment();
        longAdder.increment();
        longAdder.increment();
        System.out.println(longAdder.longValue());
        
        LongAccumulator longAccumulator = new LongAccumulator((x,y) -> x * y,2);
        longAccumulator.accumulate(1);
        longAccumulator.accumulate(2);
        longAccumulator.accumulate(3);
        System.out.println(longAccumulator.longValue());
    }
}
```


##### 2.1.5.3、性能对比
- **热点商品点赞计算器，点赞数加加统计，不要求实时精确**

```java
class ClickNumberNet
{
    int number = 0;
    public synchronized void clickBySync()
    {
        number++;
    }

    AtomicLong atomicLong = new AtomicLong(0);
    public void clickByAtomicLong()
    {
        atomicLong.incrementAndGet();
    }

    LongAdder longAdder = new LongAdder();
    public void clickByLongAdder()
    {
        longAdder.increment();
    }

    LongAccumulator longAccumulator = new LongAccumulator((x,y) -> x + y,0);
    public void clickByLongAccumulator()
    {
        longAccumulator.accumulate(1);
    }
}

/**
 * 50个线程，每个线程100W次，总点赞数出来
 */
public class LongAdderDemo2
{
    public static void main(String[] args) throws InterruptedException
    {
        ClickNumberNet clickNumberNet = new ClickNumberNet();

        long startTime;
        long endTime;
        CountDownLatch countDownLatch = new CountDownLatch(50);
        CountDownLatch countDownLatch2 = new CountDownLatch(50);
        CountDownLatch countDownLatch3 = new CountDownLatch(50);
        CountDownLatch countDownLatch4 = new CountDownLatch(50);


        startTime = System.currentTimeMillis();
        for (int i = 1; i <=50; i++) {
            new Thread(() -> {
                try
                {
                    for (int j = 1; j <=100 * 10000; j++) {
                        clickNumberNet.clickBySync();
                    }
                }finally {
                    countDownLatch.countDown();
                }
            },String.valueOf(i)).start();
        }
        countDownLatch.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒"+"\t clickBySync result: "+clickNumberNet.number);

        startTime = System.currentTimeMillis();
        for (int i = 1; i <=50; i++) {
            new Thread(() -> {
                try
                {
                    for (int j = 1; j <=100 * 10000; j++) {
                        clickNumberNet.clickByAtomicLong();
                    }
                }finally {
                    countDownLatch2.countDown();
                }
            },String.valueOf(i)).start();
        }
        countDownLatch2.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒"+"\t clickByAtomicLong result: "+clickNumberNet.atomicLong);

        startTime = System.currentTimeMillis();
        for (int i = 1; i <=50; i++) {
            new Thread(() -> {
                try
                {
                    for (int j = 1; j <=100 * 10000; j++) {
                        clickNumberNet.clickByLongAdder();
                    }
                }finally {
                    countDownLatch3.countDown();
                }
            },String.valueOf(i)).start();
        }
        countDownLatch3.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒"+"\t clickByLongAdder result: "+clickNumberNet.longAdder.sum());

        startTime = System.currentTimeMillis();
        for (int i = 1; i <=50; i++) {
            new Thread(() -> {
                try
                {
                    for (int j = 1; j <=100 * 10000; j++) {
                        clickNumberNet.clickByLongAccumulator();
                    }
                }finally {
                    countDownLatch4.countDown();
                }
            },String.valueOf(i)).start();
        }
        countDownLatch4.await();
        endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒"+"\t clickByLongAccumulator result: "+clickNumberNet.longAccumulator.longValue());
    }
}
```
##### 2.1.5.3、原理(LongAdder为什么这么快)？
LongAdder的基本思路就是分散热点，在LongAdder内部有一个base变量，一个Cell[]数组（增加方式是以2的倍数增加），如果是非竞态条件下，直接累加到base变量上，对同一个base进行操作，当出现竞争关系时则是采用化整为零的做法，以空间换时间，用一个数组cells，将一个value拆分进这个数组cells中，当多个线程需要同时对value进行操作时候，可以对线程id进行hash得到hash值，再根据hash值映射到这个数组cells的某个下标，再对该下标所对应的值进行自增操作，不同线程会命中到数组的不同槽中，各个线程只对自己槽中的那个值进行CAS操作，这样热点就被分散了，冲突的概率就小很多，当多个线程竞争同一个Cell比较激烈时，可能就要对Cell[]扩容，最高扩容到当前电脑CPU的最高核数为止。当所有线程操作完毕，将数组cells的所有值和base变量值都加起来作为最终结果。
依赖的数学表达式为：

**什么时候进行base-->新增cell-->cells扩容？**
1. 最初无竞争时只更新base,如果发生了竞争，竞争成功，还是更新base,竞争失败，则去尝试竞争cells中的值；
2. 当多个线程更新一个base值，如果线程更新base失败后，首次新建一个Cell[]数组
3. 当多个线程竞争同一个Cell比较激烈时，可能就要对Cell[]扩容，最高扩容到当前电脑CPU的最高核数为止。

**为什么最高扩容到当前电脑CPU的最高核数之后，就不继续扩容了？**
因为扩容到当前电脑CPU的最高核数的时候，刚好每一个CPU对应一个线程进行处理，如果再继续扩容的时候，一个CPU需要处理多个线程，线程之间的切换，会消耗CPU的资源，降低效率。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f8a3d79a35ac422f85e80d061b636170.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/e14d9cdd5b384e1fa1386f564076b0d1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/1fa62eec8454461c9c23e29c266e8143.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)



##### 2.1.5.4、源码、原理分析
架构：
![在这里插入图片描述](https://img-blog.csdnimg.cn/aee3fcd4ce2c44e3ab48b71eeaba48e3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)

 **add(1L)方法源码分析**![在这里插入图片描述](https://img-blog.csdnimg.cn/a886f30b2a0644329ce46fc50a832ada.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16 )
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/27132b4ab5d847c7936ed75773840669.png)
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/bedd9da4b04b437eafa41390d236bb1a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/0ca18d2eb9834676b67bd50b3ac3b72c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
**sum源码解析：**
 
sum()会将所有Cell数组中的value和base累加作为返回值。核心的思想就是将之前AtomicLong一个value的更新压力分散到多个value中去，从而降级更新热点。

```java
public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```
sum执行时，并没有限制对base和cells的更新(一句要命的话)。所以LongAdder不是强一致性的，它是最终一致性的。首先，最终返回的sum局部变量，初始被复制为base，而最终返回时，很可能base已经被更新了，而此时局部变量sum不会更新，造成不一致。其次，这里对cell的读取也无法保证是最后一次写入的值。所以，sum方法在没有并发的情况下，可以获得正确的结果。


**longAccumulate源码解析：**
Loading......

![在这里插入图片描述](https://img-blog.csdnimg.cn/0740d5c6e2ac4e99861b0e59463bf52c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/ae5274f514ed44e6b319d83ab5075c68.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/04a1eb5ab1a84b05971ab1d0d1a8128b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/674b2f211fa541d6ba53a744f6fc2347.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAcHJlZmVjdF9zdGFydA==,size_20,color_FFFFFF,t_70,g_se,x_16)
##### 2.1.5.5、使用总结
- AtomicLong
1. 线程安全，可允许一些性能损耗，要求高精度时可使用
2. 保证精度，性能代价
3. AtomicLong是多个线程针对单个热点值value进行原子操作

- LongAdder
1. 当需要在高并发下有较好的性能表现，且对值的精确度要求不高时，可以使用
2. 保证性能，精度代价
3. LongAdder是每个线程拥有自己的槽，各个线程一般只对自己槽中的那个值进行CAS操作



AtomicLong
1. 原理:CAS+自旋
2. 场景:低并发下的全局计算,AtomicLong能保证并发情况下计数的准确性，其内部通过CAS来解决并发安全性的问题。
3. 缺陷:高并发后性能急剧下降,AtomicLong的自旋会成为瓶颈 N个线程CAS操作修改线程的值，每次只有一个成功过，其它N - 1失败，失败的不停的自旋直到成功，这样大量失败自旋的情况，一下子cpu就打高了。
 
LongAdder
1. 原理:CAS+Base+Cell数组分散,空间换时间并分散了热点数据
2. 场景:高并发下的全局计算
3. 缺陷:sum求和后还有计算线程修改结果的话，最后结果不够准确

