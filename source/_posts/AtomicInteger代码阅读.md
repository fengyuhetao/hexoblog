---
title: AtomicInteger代码阅读
date: 2019-04-18 12:34:32
tags:
---

# AtomicInteger深入理解

> 高并发的情况下，`i++`无法保证原子性，往往会出现问题，所以引入`AtomicInteger`类。

## 代码测试

```
public class TestAtomicInteger {
    private static final int THREADS_COUNT = 2;

    public static int count = 0;
    public static volatile int countVolatile = 0;
    public static AtomicInteger atomicInteger = new AtomicInteger(0);
    public static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void increase() {
        count++;
        countVolatile++;
        atomicInteger.incrementAndGet();
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS_COUNT];
        for(int i = 0; i< threads.length; i++) {
            threads[i] = new Thread(() -> {
                for(int i1 = 0; i1 < 1000; i1++) {
                    increase();
                }
                countDownLatch.countDown();
            });
            threads[i].start();
        }

        countDownLatch.await();

        System.out.println(count);
        System.out.println(countVolatile);
        System.out.println(atomicInteger.get());
    }
}
```

测试结果如下:

```
1974
1990
2000
```

通过多次测试，我们可以看到只有`AtomicInteger`能够真正保证最终结果永远是2000。关于`volatile`的文章，这里可以推荐一个: [面试官最爱的volatile关键字 ](https://mp.weixin.qq.com/s?__biz=MzI4MDYwMDc3MQ==&mid=2247486266&idx=1&sn=7beaca0358914b3606cde78bfcdc8da3&chksm=ebb74296dcc0cb805a45ca9c0501b7c2c37e8f2586295210896d18e3a0c72b01bea765924ce5&mpshare=1&scene=24&srcid=&key=c8fbfa031bd0c4166acd110fd54b85e9b3568f80a3f4c2d80add2f4add0ced46d1d3a0cf139c0ca64877a98635727a7fc593b850f8082d1fcf77a5ebf067fc1476285146d13d691f80b64b930006a341&ascene=0&uin=MjYwNzAzMzYzNw%3D%3D&devicetype=iMac+MacBookAir6%2C2+OSX+OSX+10.14.2+build(18C54)&version=12020810&nettype=WIFI&lang=zh_CN&fontScale=100&pass_ticket=hbg9AwR77rok2jxxdwyHyTHBDzwwC7lR8aEfF6HfW4KgJwsj0ruOpw8iNsUK%2B5kK)。

##代码阅读

考虑到代码比较长，这里仅仅截取部分代码进行讲解。

### 定义的变量

```
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

// 通过Unsafe计算出value变量在对象中的偏移，该偏移值下边会用到
static {
   try {
       valueOffset = unsafe.objectFieldOffset
          (AtomicInteger.class.getDeclaredField("value"));
   } catch (Exception ex) { throw new Error(ex); }
}

// value保存当前的值
private volatile int value;
```

* unsafe: 一般来说，Java不像c或者c++那样，可以直接操作内存，`Unsafe`可以说是一个后门，可以直接操作内存，或者进行线程调度。（以后会专门写一篇关于Unsafe类的文章），能够使用
* valueOffset: 在类初始化的时候，计算出value变量在对象中的偏移
* value: 保存当前的值

### layzSet方法:

```
/**
 * Eventually sets to the given value.
 */
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}
```

该方法调用了本地方法Unsafe.putOrderedLong，具体实现见[源码](<http://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/9b0ca45cd756/src/share/vm/prims/unsafe.cpp>)。

```
  {CC"putOrderedObject",   CC"("OBJ"J"OBJ")V",         FN_PTR(Unsafe_SetOrderedObject)},
  {CC"putOrderedInt",      CC"("OBJ"JI)V",             FN_PTR(Unsafe_SetOrderedInt)},
  {CC"putOrderedLong",     CC"("OBJ"JJ)V",             FN_PTR(Unsafe_SetOrderedLong)},
```

可以看到该方法会调用`Unsafe_SetOrderedInt`方法。

```
UNSAFE_ENTRY(void, Unsafe_SetOrderedInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint x))
  UnsafeWrapper("Unsafe_SetOrderedInt");
  SET_FIELD_VOLATILE(obj, offset, jint, x);
UNSAFE_END
```

调用了`SET_FIELD_VOLATILE`。

```
#define SET_FIELD_VOLATILE(obj, offset, type_name, x) \
  oop p = JNIHandles::resolve(obj); \
  OrderAccess::release_store_fence((volatile type_name*)index_oop_from_field_offset_long(p, offset), x);
```

继续看: `OrderAccess::release_store_fence`,该方法可见[源码](<http://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/9b0ca45cd756/src/os_cpu/linux_x86/vm/orderAccess_linux_x86.inline.hpp>)。

```
inline void     OrderAccess::release_store_fence(volatile jint*   p, jint   v) {
  __asm__ volatile (  "xchgl (%2),%0"
                    : "=r" (v)
                    : "0" (v), "r" (p)
                    : "memory");
}
```

最终采用`xchgl`指令来实现变量的赋值。

所以: lazySet方法是由依赖于硬件的系统指令(如x86的xchg)实现的。使用lazySet的话，其他线程在之后的一小段时间里还是可以读到旧的值。个人猜测：lazySet方法相比于`set`方法可能性能好一点。

网站上找到的资料:

```
1.首先set()是对volatile变量的一个写操作, 我们知道volatile的write为了保证对其他线程的可见性会追加以下两个Fence(内存屏障)
1)StoreStore // 在intel cpu中, 不存在[写写]重排序, 这个可以直接省略了
2)StoreLoad // 这个是所有内存屏障里最耗性能的
注: 内存屏障相关参考Doug Lea大大的cookbook (http://g.oswego.edu/dl/jmm/cookbook.html)

2.Doug Lea大大又说了, lazySet()省去了StoreLoad屏障, 只留下StoreStore
```

关于为什么会添加`lazySet`方法，可以参考:[https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329](<https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329>)。

简单来说: putOrderedInt 之前的写不会被重排序，之后的写会被重排序。之后会被重排序意味着执行以后对其他线程是没有可见性保证的。

> 设想如下场景: 设置一个 volatile 变量为 null，让这个对象被 GC 掉，volatile write 是消耗比较大（store-load 屏障）的，但是 putOrderedInt 只会加 store-store 屏障，损耗会小一些。

### set方法

由于使用了`volatile`关键字，因此set方法可以保证并发情况下不会出现问题。

### incrementAndGet

```
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}	
```

调用unsafe中的`getAndInt`方法。

```
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 直接从该对象对应位置取出变量的值
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
}
```

* var1: Object
* var2: valueOffset
* var5: 当前该变量在内存中的值
* var5 + var4: 需要写进去的值

采用CAS机制，不断使用`compareAndSwapInt`尝试修改该值，如果失败，重新获取。如果并发量小，问题不大。

* 并发量大的情况下，由于真正更新成功的线程占少数，容易导致循环次数过多，浪费时间。
* 由于需要保证变量真正的共享，缓存行失效，**缓存一致性**开销变大。
* 底层开销可能较大，这个我就不追究了。
* 该函数做的事较多，不仅增加value，同时还给出返回值，返回值换成void就好了。

### getAndUpdate

```
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}
```

该方法需要实现`IntUnaryOperator`接口，然后会调用`applyAsInt`方法对当前值进行处理，将当前值替换为`applyAsInt`方法的返回值。

参考代码:

```
public class TestAtomicInteger {
    private static final int THREADS_COUNT = 2;

    public static AtomicInteger atomicInteger = new AtomicInteger(0);
    public static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void doubleValue() {
        atomicInteger.getAndUpdate(operand -> {
            if(operand % 2 == 0) {
                operand = operand + 11;
            } else {
                operand = operand + 19;
            }
            return operand;
        });
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[THREADS_COUNT];
        for(int i = 0; i< threads.length; i++) {
            threads[i] = new Thread(() -> {
                for(int i1 = 0; i1 < 1000; i1++) {
                    doubleValue();
                }
                countDownLatch.countDown();
            });
            threads[i].start();
        }

        countDownLatch.await();

        System.out.println(atomicInteger.get());
    }
}

```

结果是:30000。

## 操作系统级别CAS实现

**摘自**[CAS操作在ARM和x86下的不同实现](<https://blog.csdn.net/a7980718/article/details/82860505>)。

cmpxchg是X86比较交换指令，这个指令在各大底层系统实现的原子操作和各种同步原语中都有广泛的使用，比如linux内核，JVM,GCC编译器等,cmpxchg就是比较交换指令。

intel P6以及最新系列处理器保证了以下操作是原子的：1.读写一个字节。2.读写16位对齐的字。3.读写32位对齐的双字。4.读写64位对齐的四字。5.读写16位，32位，64位在cache line内的未对齐的字。所以普通的load store指令都是原子的。cache一致性协议保证了不可能有两个cpu同时写一个内存。对于cmpxchg这种比较交换指令肯定不是原子的，intel是CISC复杂指令集架构，在内部流水线执行的时候，肯定会将cmpxchg指令翻译成几条微码执行（对比ARM精简指令集）。所以英特尔对于一些指令提供了LOCK前缀来保证这个指令的原子性。Intel 64和IA-32处理器提供LOCK＃信号，该信号在某些关键存储器操作期间自动置位，以锁定系统总线或等效链路。当该输出信号被断言时，来自其他处理器或总线代理的用于控制总线的请求被阻止。

 为了更清楚理解cmxchg，需要同时看ARM和x86两种架构下的实现一个RISC,一个CISC，linux内核提供了两种架构下的实现。linux内核的原子变量定义如下：

```

//原子变量
typedef struct {
	volatile int counter; //volatile禁止编译器把变量缓冲到寄存器
} atomic_t;
```

先看ARM架构下，ARM架构是精简指令集，没有提供cmpxchg这种复杂指令，和其它所有RISC架构一样提供了LL/SC（链接加载，条件存储）操作，这个操作是很多原子操作的基础。ARMv8指令是LDXR\STXR，属于独占访问，需要有local monitor和global monitor配合使用。这两条指令一般需要成对出现。ldrex是从内存取出数据放到寄存器，然后监视器将此地址标记为独占，strex会先测试是否是当前cpu的独占，如果是则存储成功返回0，如果不是则存储失败返回1。例如cpu0将地址m标记为独占，在strex执行前，线程被调出了，cpu1调用ldrex会清除cpu0的独占，而将自己标记为独占，然后执行strxr，然后cpu0的线程重新被调度，此时执行strex会失败，因为自己的独占位被清除了。这样也会导致后进入ldrex的线程可能比先进入的先执行。标记为独占的地址调用strex后都会清除独占标志。

```
/**
 *  比较ptr->counter和old的值如果相等，则ptr->counter = new,并且返回old，否则ptr->counter不变
 * 返回ptr->counter
 */
static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new)
{
	unsigned long oldval, res;
 
	smp_mb(); //内存屏障，保证cmpxchg不会在屏障前执行
 
	do {
		__asm__ __volatile__("@ atomic_cmpxchg\n"
		"ldrex	%1, [%2]\n" //独占访问，监视器会将此地址标志独占并且将ptr->counter给oldvalue
		"mov	%0, #0\n"   //res = 0
		"teq	%1, %3\n"   //测试oldvalue是否和old相等也就是ptr->counter和old
 
		//独占访问成功并且如果相等则把new赋值给ptr->counter,否则不执行这条指令
		"strexeq %0, %4, [%2]\n" 
		    : "=&r" (res), "=&r" (oldval)
		    : "r" (&ptr->counter), "Ir" (old), "r" (new)
		    : "cc");
	} while (res);  //while res是因为strexeq指令是独占访存指令从，此时可能未标记访存，而res为1
 
	smp_mb();//内存屏障，保证cmpxchg不会在屏障后执行
 
	return oldval;
}
```

X86架构类似:

```

/*
 *  根据size大小比较交换字节，字或者双字，如果返回old则交换成功，否则交换失败
 */
static inline unsigned long __cmpxchg(volatile void *ptr, unsigned long old,
				      unsigned long new, int size)
{
	unsigned long prev;
	switch (size) {
	case 1:
		__asm__ __volatile__(LOCK_PREFIX "cmpxchgb %b1,%2"
				     : "=a"(prev)
				     : "q"(new), "m"(*__xg(ptr)), "0"(old)
				     : "memory");
		return prev;
	case 2:
		__asm__ __volatile__(LOCK_PREFIX "cmpxchgw %w1,%2"
				     : "=a"(prev)
				     : "r"(new), "m"(*__xg(ptr)), "0"(old)
				     : "memory");
		return prev;
//eax = old，比较%2 = ptr->counter和eax是否相等，如果相等则ZF置位，并把%1 = new赋值给ptr->counter,返回old值，否则ZF清除，并且将ptr->counter赋值给eax
	case 4:
		__asm__ __volatile__(LOCK_PREFIX "cmpxchgl %1,%2"
				     : "=a"(prev)
				     : "r"(new), "m"(*__xg(ptr)), "0"(old)  //0表示eax = old
				     : "memory");
		return prev;
	}
	return old;
}
```

代码解释:

以最为常用的4字节交换为例，主要的操作就是汇编指令`cmpxchgl %1,%2`，注意一下其中的%2，也就是后面的`"m"(*__xg(ptr))`。`__xg`是在这个文件中定义的宏：

```
struct __xchg_dummy { unsigned long a[100]; };

#define __xg(x) ((struct __xchg_dummy *)(x))
```

那么%2经过预处理，展开就是`"m"(*((struct __xchg_dummy *)(ptr)))`，这种做法，就可以达到在cmpxchg中的%2是一个地址，就是ptr指向的地址。如果%2是`"m"(ptr)`，那么指针本身的值就出现在cmpxchg指令中。

简单点就是:

```
cmpxchg %ecx, %ebx；如果EAX与EBX相等，则ECX送EBX且ZF置1；否则EBX送ECX，且ZF清0
```

在cmpxchg指令前加了lock前缀，保证在进行操作的时候，不会让其它cpu操作同一个内存。使得整个操作保持原子性。对比来看虽然X86只用了一条指令，但是处理器内部肯定将这条指令转成了类RISC的微码。

## 性能

基于`AtomicLong`可能性能有点差，所以出现了`LongAdder`。接下来通过基准测试，来看看两者的性能差别。

### 基准测试

Mode设为`Throughput`，测试吞吐量。

Mode设为`AverageTime`，测试平均耗时。

```
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@BenchmarkMode(Mode.Throughput)
public class Main {
    private static AtomicLong count = new AtomicLong();

    private static LongAdder longAdder = new LongAdder();

    public static void main(String[] args) throws RunnerException {
        Options options = (Options) new OptionsBuilder().include(Main.class.getName());
        new Runner(options).run();
    }

    @Benchmark
    @Threads(10)
    public void run0() {
        count.getAndIncrement();
    }

    @Benchmark
    @Threads(10)
    public void run1() {
        longAdder.increment();
    }
}
```

#### 吞吐量:

线程为1，结果:

```
Benchmark   Mode  Cnt    Score   Error   Units
Main.run0  thrpt   25  125.268 ± 2.471  ops/us
Main.run1  thrpt   25   74.587 ± 1.484  ops/us
```

线程为10，结果:

```
Benchmark   Mode  Cnt    Score   Error   Units
Main.run0  thrpt   25   24.897 ± 2.888  ops/us
Main.run1  thrpt   25  192.474 ± 2.824  ops/us
```

线程为30，结果:

```
Benchmark   Mode  Cnt    Score   Error   Units
Main.run0  thrpt   25   21.096 ± 1.897  ops/us
Main.run1  thrpt   25  189.732 ± 4.089  ops/us
```

#### 平均耗时:

线程为1，结果：

```
Main.run0  avgt   25  0.008 ±  0.001  us/op
Main.run1  avgt   25  0.013 ±  0.001  us/op
```

线程为10，结果:

```
Benchmark  Mode  Cnt  Score   Error  Units
Main.run0  avgt   25  0.485 ± 0.048  us/op
Main.run1  avgt   25  0.050 ± 0.001  us/op
```

线程为30，结果:

```
Benchmark  Mode  Cnt  Score   Error  Units
Main.run0  avgt   25  1.364 ± 0.033  us/op
Main.run1  avgt   25  0.158 ± 0.003  us/op
```

可以看到除了线程为1的情况下，其他情况下，LogAdder明显比AtomicLong要好的多。

## ABA问题

设想如下场景：

1. 线程1准备用CAS将变量的值由A替换为B。
2. 在此之前，线程2将变量的值由A替换为C，又由C替换为A，
3. 然后线程1执行CAS时发现变量的值仍然为A，所以CAS成功。

但实际上这时的现场已经和最初不同了，尽管CAS成功，但可能存在潜藏的问题。

比如：

* 有一个用单向链表实现的栈，栈顶为A。
* 线程T1获取A.next为B，然后希望用CAS将栈顶替换为B，head.compareAndSet(A,B)。
* 在T1执行上面这条指令之前，线程T2介入，将A、B出栈，再pushD、C、A。此时B.next为null。
* 此时轮到线程T1执行CAS操作，检测发现栈顶仍为A，所以CAS成功，栈顶变为B。
* 但实际上B.next为null，其中堆栈中只有B一个元素，C和D组成的链表不再存在于堆栈中，这样就造成C、D被抛弃的现象。

### 解决方法

各种乐观锁的实现中通常都会用版本戳version来对记录或对象标记，避免并发操作带来的问题，在Java中，AtomicStampedReference<E>也实现了这个作用，它通过包装[E,Integer]的元组来对对象标记版本戳stamp，从而避免ABA问题。

## 参考链接

* http://ifeve.com/java-atomic/
* <http://ifeve.com/juc-atomic-class-lazyset-que/>

* <https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329>

* <http://g.oswego.edu/dl/jmm/cookbook.html>

* <http://footmanff.com/2018/03/27/2018-03-27-unsafe-lazyset/>

* [CAS操作在ARM和x86下的不同实现](<https://blog.csdn.net/a7980718/article/details/82860505>)

* [浅谈缓存一致性原则和Java内存模型（JMM）](<https://blog.csdn.net/sdr_zd/article/details/81323519>)
* [java计数器探秘](https://mp.weixin.qq.com/s/yAvJFZWxfKb38IDMjQd5zg)

* [Linux内核中的cmpxchg函数](https://blog.csdn.net/zdy0_2004/article/details/48013829)
* [Java CAS ABA问题发生的场景分析](https://www.cnblogs.com/senlinyang/p/7875381.html)