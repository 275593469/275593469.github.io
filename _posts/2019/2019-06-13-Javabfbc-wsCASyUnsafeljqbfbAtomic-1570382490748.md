---
layout: post
title:  "Java并发编程-无锁CAS与Unsafe类及其并发包Atomic"
categories: java
tags: 高并发 CAS
author: jiangc
excerpt: Java并发编程-无锁CAS与Unsafe类及其并发包Atomic
---
* content
{:toc}

>   转载：http://blog.csdn.net/javazejian/article/details/72772470
>
>关联文章：
>
>[深入理解Java类型信息(Class对象)与反射机制](http://blog.csdn.net/javazejian/article/details/70768369)
>
>[深入理解Java枚举类型(enum)](http://blog.csdn.net/javazejian/article/details/71333103)
>
>[深入理解Java注解类型(@Annotation)](http://blog.csdn.net/javazejian/article/details/71860633)
>
>[深入理解Java类加载器(ClassLoader)](http://blog.csdn.net/javazejian/article/details/73413292)
>
>[深入理解Java并发之synchronized实现原理](http://blog.csdn.net/javazejian/article/details/72828483)
>
>[Java并发编程-无锁CAS与Unsafe类及其并发包Atomic](http://blog.csdn.net/javazejian/article/details/72772470)
>
>[深入理解Java内存模型(JMM)及volatile关键字](http://blog.csdn.net/javazejian/article/details/72772461)
>
>[剖析基于并发AQS的重入锁(ReetrantLock)及其Condition实现原理](http://blog.csdn.net/javazejian/article/details/75043422)
>
>[剖析基于并发AQS的共享锁的实现(基于信号量Semaphore)](http://blog.csdn.net/javazejian/article/details/76167357)
>
>[并发之阻塞队列LinkedBlockingQueue与ArrayBlockingQueue](http://blog.csdn.net/javazejian/article/details/77410889)

在前面一篇博文中，我们曾经详谈过有锁并发的典型代表synchronized关键字，通过该关键字可以控制并发执行过程中有且只有一个线程可以访问共享资源，其原理是通过当前线程持有当前对象锁，从而拥有访问权限，而其他没有持有当前对象锁的线程无法拥有访问权限，也就保证了线程安全。但在本篇中，我们将会详聊另外一种反向而行的并发策略，即无锁并发，即不加锁也能保证并发执行的安全性。

本篇的思路是先阐明无锁执行者CAS的核心算法原理然后分析Java执行CAS的实践者Unsafe类，该类中的方法都是native修饰的，因此我们会以说明方法作用为主介绍Unsafe类，最后再介绍并发包中的Atomic系统使用CAS原理实现的并发类，以下是主要内容

1. [无锁的概念](https://blog.csdn.net/javazejian/article/details/72772470#%E6%97%A0%E9%94%81%E7%9A%84%E6%A6%82%E5%BF%B5)
2. [无锁的执行者-CAS](https://blog.csdn.net/javazejian/article/details/72772470#%E6%97%A0%E9%94%81%E7%9A%84%E6%89%A7%E8%A1%8C%E8%80%85-cas)

1.
  1. [CAS](https://blog.csdn.net/javazejian/article/details/72772470#cas)
  2. [CPU指令对CAS的支持](https://blog.csdn.net/javazejian/article/details/72772470#cpu%E6%8C%87%E4%BB%A4%E5%AF%B9cas%E7%9A%84%E6%94%AF%E6%8C%81)

1. [鲜为人知的指针 Unsafe类](https://blog.csdn.net/javazejian/article/details/72772470#%E9%B2%9C%E4%B8%BA%E4%BA%BA%E7%9F%A5%E7%9A%84%E6%8C%87%E9%92%88-unsafe%E7%B1%BB)
2. [并发包中的原子操作类Atomic系列](https://blog.csdn.net/javazejian/article/details/72772470#%E5%B9%B6%E5%8F%91%E5%8C%85%E4%B8%AD%E7%9A%84%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C%E7%B1%BBatomic%E7%B3%BB%E5%88%97)

1.
  1. [原子更新基本类型](https://blog.csdn.net/javazejian/article/details/72772470#%E5%8E%9F%E5%AD%90%E6%9B%B4%E6%96%B0%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B)
  2. [原子更新引用](https://blog.csdn.net/javazejian/article/details/72772470#%E5%8E%9F%E5%AD%90%E6%9B%B4%E6%96%B0%E5%BC%95%E7%94%A8)
  3. [原子更新数组](https://blog.csdn.net/javazejian/article/details/72772470#%E5%8E%9F%E5%AD%90%E6%9B%B4%E6%96%B0%E6%95%B0%E7%BB%84)
  4. [原子更新属性](https://blog.csdn.net/javazejian/article/details/72772470#%E5%8E%9F%E5%AD%90%E6%9B%B4%E6%96%B0%E5%B1%9E%E6%80%A7)
  5. [CAS的ABA问题及其解决方案](https://blog.csdn.net/javazejian/article/details/72772470#cas%E7%9A%84aba%E9%97%AE%E9%A2%98%E5%8F%8A%E5%85%B6%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88)

1. [再谈自旋锁](https://blog.csdn.net/javazejian/article/details/72772470#%E5%86%8D%E8%B0%88%E8%87%AA%E6%97%8B%E9%94%81)

# 无锁的概念

在谈论无锁概念时，总会关联起乐观派与悲观派，对于乐观派而言，他们认为事情总会往好的方向发展，总是认为坏的情况发生的概率特别小，可以无所顾忌地做事，但对于悲观派而已，他们总会认为发展事态如果不及时控制，以后就无法挽回了，即使无法挽回的局面几乎不可能发生。这两种派系映射到并发编程中就如同加锁与无锁的策略，即加锁是一种悲观策略，无锁是一种乐观策略，因为对于加锁的并发程序来说，它们总是认为每次访问共享资源时总会发生冲突，因此必须对每一次数据操作实施加锁策略。而无锁则总是假设对共享资源的访问没有冲突，线程可以不停执行，无需加锁，无需等待，一旦发现冲突，无锁策略则采用一种称为CAS的技术来保证线程执行的安全性，这项CAS技术就是无锁策略实现的关键，下面我们进一步了解CAS技术的奇妙之处。

# 无锁的执行者-CAS

**CAS**

CAS的全称是Compare And Swap 即比较交换，其算法核心思想如下

执行函数：CAS(V,E,N)

其包含3个参数

1. V表示要更新的变量
2. E表示预期值
3. N表示新值

如果V值等于E值，则将V的值设为N。若V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。通俗的理解就是CAS操作需要我们提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，也就是执行CAS操作，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，但可以选择重新读取该变量再尝试再次修改该变量，也可以放弃操作，原理图如下

![image](/images/2019\06\java\1570382490748.jpg "image")

由于CAS操作属于乐观派，它总认为自己可以成功完成操作，当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作，这点从图中也可以看出来。基于这样的原理，CAS操作即使没有锁，同样知道其他线程对共享资源操作影响，并执行相应的处理措施。同时从这点也可以看出，由于无锁操作中没有锁的存在，因此不可能出现死锁的情况，也就是说无锁操作天生免疫死锁。

**CPU指令对CAS的支持**

或许我们可能会有这样的疑问，假设存在多个线程执行CAS操作并且CAS的步骤很多，有没有可能在判断V和E相同后，正要赋值时，切换了线程，更改了值。造成了数据不一致呢？答案是否定的，因为CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。

# 鲜为人知的指针: Unsafe类

Unsafe类存在于sun.misc包中，其内部方法操作可以像C的指针一样直接操作内存，单从名称看来就可以知道该类是非安全的，毕竟Unsafe拥有着类似于C的指针操作，因此总是不应该首先使用Unsafe类，Java官方也不建议直接使用的Unsafe类，据说Oracle正在计划从Java 9中去掉Unsafe类，但我们还是很有必要了解该类，因为Java中CAS操作的执行依赖于Unsafe类的方法，注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务，关于Unsafe类的主要功能点如下：

1. 内存管理，Unsafe类中存在直接操作内存的方法

_//分配内存指定大小的内存_
publicnativelongallocateMemory(long bytes);
_//根据给定的内存地址address设置重新分配指定大小的内存_
publicnativelongreallocateMemory(long address, long bytes);
_//用于释放allocateMemory和reallocateMemory申请的内存_
publicnativevoidfreeMemory(long address);
_//将指定对象的给定offset偏移量内存块中的所有字节设置为固定值_
publicnativevoidsetMemory(Object o, long offset, long bytes, byte value);
_//设置给定内存地址的值_
publicnativevoidputAddress(long address, long x);
_//获取指定内存地址的值_
publicnativelonggetAddress(long address);

_//设置给定内存地址的long值_
publicnativevoidputLong(long address, long x);
_//获取指定内存地址的long值_
publicnativelonggetLong(long address);
_//设置或获取指定内存的byte值_
publicnativebyte  getByte(long address);
publicnativevoid  putByte(long address, byte x);
_//其他基本数据类型(long,char,float,double,short等)的操作与putByte及getByte相同_

_//操作系统的内存页大小_
publicnativeintpageSize();

1. 提供实例对象新途径。

_//传入一个对象的class并创建该实例对象，但不会调用构造方法_
publicnative Object allocateInstance(Class cls) throws InstantiationException;

1. 类和实例对象以及变量的操作，主要方法如下

_//获取字段f在实例对象中的偏移量_
publicnativelongobjectFieldOffset(Field f);
_//静态属性的偏移量，用于在对应的Class对象中读写静态属性_
publicnativelongstaticFieldOffset(Field f);
_//返回值就是f.getDeclaringClass()_
publicnative Object staticFieldBase(Field f);


_//获得给定对象偏移量上的int值，所谓的偏移量可以简单理解为指针指向该变量的内存地址，_
_//通过偏移量便可得到该对象的变量，进行各种操作_
publicnativeintgetInt(Object o, long offset);
_//设置给定对象上偏移量的int值_
publicnativevoidputInt(Object o, long offset, int x);

_//获得给定对象偏移量上的引用类型的值_
publicnative Object getObject(Object o, long offset);
_//设置给定对象偏移量上的引用类型的值_
publicnativevoidputObject(Object o, long offset, Object x);
_//其他基本数据类型(long,char,byte,float,double)的操作与getInthe及putInt相同_

_//设置给定对象的int值，使用volatile语义，即设置后立马更新到内存对其他线程可见_
publicnativevoid  putIntVolatile(Object o, long offset, int x);
_//获得给定对象的指定偏移量offset的int值，使用volatile语义，总能获取到最新的int值。_
publicnativeintgetIntVolatile(Object o, long offset);

_//其他基本数据类型(long,char,byte,float,double)的操作与putIntVolatile及getIntVolatile相同，引用类型putObjectVolatile也一样。_

_//与putIntVolatile一样，但要求被操作字段必须有volatile修饰_
publicnativevoidputOrderedInt(Object o,long offset,int x);

下面通过一个简单的Demo来演示上述的一些方法以便加深对Unsafe类的理解

publicclass UnSafeDemo {

    public  static  voidmain(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        _// 通过反射得到theUnsafe对应的Field对象_
        Field field = Unsafe.class.getDeclaredField(&quot;theUnsafe&quot;);
        _// 设置该Field为可访问_
        field.setAccessible(true);
        _// 通过Field得到该Field对应的具体对象，传入null是因为该Field为static的_
        Unsafe unsafe = (Unsafe) field.get(null);
        System.out.println(unsafe);

        _//通过allocateInstance直接创建对象_
        User user = (User) unsafe.allocateInstance(User.class);

        Class userClass = user.getClass();
        Field name = userClass.getDeclaredField(&quot;name&quot;);
        Field age = userClass.getDeclaredField(&quot;age&quot;);
        Field id = userClass.getDeclaredField(&quot;id&quot;);

        _//获取实例变量name和age在对象内存中的偏移量并设置值_
        unsafe.putInt(user,unsafe.objectFieldOffset(age),18);
        unsafe.putObject(user,unsafe.objectFieldOffset(name),&quot;android TV&quot;);

        _// 这里返回 User.class，_
        Object staticBase = unsafe.staticFieldBase(id);
        System.out.println(&quot;staticBase:&quot;+staticBase);

        _//获取静态变量id的偏移量staticOffset_
        long staticOffset = unsafe.staticFieldOffset(userClass.getDeclaredField(&quot;id&quot;));
        _//获取静态变量的值_
        System.out.println(&quot;设置前的ID:&quot;+unsafe.getObject(staticBase,staticOffset));
        _//设置值_
        unsafe.putObject(staticBase,staticOffset,&quot;SSSSSSSS&quot;);
        _//获取静态变量的值_
        System.out.println(&quot;设置前的ID:&quot;+unsafe.getObject(staticBase,staticOffset));
        _//输出USER_
        System.out.println(&quot;输出USER:&quot;+user.toString());

        long data = 1000;
        byte size = 1;_//单位字节_

        _//调用allocateMemory分配内存,并获取内存地址memoryAddress_
        long memoryAddress = unsafe.allocateMemory(size);
        _//直接往内存写入数据_
        unsafe.putAddress(memoryAddress, data);
        _//获取指定内存地址的数据_
        long addrData=unsafe.getAddress(memoryAddress);
        System.out.println(&quot;addrData:&quot;+addrData);

        _/\*\*
         \* 输出结果:
         [email protected]
         staticBase:class geym.conc.ch4.atomic.User
         设置前的ID:USER\_ID
         设置前的ID:SSSSSSSS
         输出USER:User{name=&#39;android TV&#39;, age=18&#39;, id=SSSSSSSS&#39;}
         addrData:1000
         \*/_

    }
}

class User{
    publicUser(){
        System.out.println(&quot;user 构造方法被调用&quot;);
    }
    private String name;
    privateint age;
    privatestatic String id=&quot;USER\_ID&quot;;

    @Override
    public String toString() {
        return&quot;User{&quot; +
                &quot;name=&#39;&quot; + name + &#39;\&#39;&#39; +
                &quot;, age=&quot; + age +&#39;\&#39;&#39; +
                &quot;, id=&quot; + id +&#39;\&#39;&#39; +
                &#39;}&#39;;
    }
}

虽然在Unsafe类中存在getUnsafe()方法，但该方法只提供给高级的Bootstrap类加载器使用，普通用户调用将抛出异常，所以我们在Demo中使用了反射技术获取了Unsafe实例对象并进行相关操作。

publicstatic Unsafe getUnsafe() {
      Class cc = sun.reflect.Reflection.getCallerClass(2);
      if (cc.getClassLoader() != null)
          thrownew SecurityException(&quot;Unsafe&quot;);
      return theUnsafe;
  }

1. 数组操作

_//获取数组第一个元素的偏移地址_
publicnativeintarrayBaseOffset(Class arrayClass);
_//数组中一个元素占据的内存空间,arrayBaseOffset与arrayIndexScale配合使用，可定位数组中每个元素在内存中的位置_
publicnativeintarrayIndexScale(Class arrayClass);

1. CAS 操作相关

CAS是一些CPU直接支持的指令，也就是我们前面分析的无锁操作，在Java中无锁操作CAS基于以下3个方法实现，在稍后讲解Atomic系列内部方法是基于下述方法的实现的。

_//第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，_
_//expected表示期望值，x表示要设置的值，下面3个方法都通过CAS原子指令执行操作。_
publicfinalnativebooleancompareAndSwapObject(Object o, long offset,Object expected, Object x);

publicfinalnativebooleancompareAndSwapInt(Object o, long offset,int expected,int x);

publicfinalnativebooleancompareAndSwapLong(Object o, long offset,long expected,long x);

这里还需介绍Unsafe类中JDK 1.8新增的几个方法，它们的实现是基于上述的CAS方法，如下

_//1.8新增，给定对象o，根据获取内存偏移量指向的字段，将其增加delta，_
_//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值_
public final intgetAndAddInt(Object o, long offset, int delta) {
     int v;
     do {
         _//获取内存中最新值_
         v = getIntVolatile(o, offset);
       _//通过CAS操作_
     } while (!compareAndSwapInt(o, offset, v, v + delta));
     return v;
 }

_//1.8新增，方法作用同上，只不过这里操作的long类型数据_
public final longgetAndAddLong(Object o, long offset, long delta) {
     long v;
     do {
         v = getLongVolatile(o, offset);
     } while (!compareAndSwapLong(o, offset, v, v + delta));
     return v;
 }

_//1.8新增，给定对象o，根据获取内存偏移量对于字段，将其 设置为新值newValue，_
_//这是一个CAS操作过程，直到设置成功方能退出循环，返回旧值_
public final intgetAndSetInt(Object o, long offset, int newValue) {
     int v;
     do {
         v = getIntVolatile(o, offset);
     } while (!compareAndSwapInt(o, offset, v, newValue));
     return v;
 }

_// 1.8新增，同上，操作的是long类型_
public final longgetAndSetLong(Object o, long offset, long newValue) {
     long v;
     do {
         v = getLongVolatile(o, offset);
     } while (!compareAndSwapLong(o, offset, v, newValue));
     return v;
 }

_//1.8新增，同上，操作的是引用类型数据_
public final Object getAndSetObject(Object o, long offset, Object newValue) {
     Object v;
     do {
         v = getObjectVolatile(o, offset);
     } while (!compareAndSwapObject(o, offset, v, newValue));
     return v;
 }

上述的方法我们在稍后的Atomic系列分析中还会见到它们的身影。

1. 挂起与恢复

将一个线程进行挂起是通过park方法实现的，调用 park后，线程将一直阻塞直到超时或者中断等条件出现。unpark可以终止一个挂起的线程，使其恢复正常。Java对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，其底层实现最终还是使用Unsafe.park()方法和Unsafe.unpark()方法

_//线程调用该方法，线程将一直阻塞直到超时，或者是中断条件出现。  _
publicnativevoidpark(boolean isAbsolute, long time);

_//终止挂起的线程，恢复正常.java.util.concurrent包中挂起操作都是在LockSupport类实现的，其底层正是使用这两个方法，  _
publicnativevoidunpark(Object thread);

1. 内存屏障

这里主要包括了loadFence、storeFence、fullFence等方法，这些方法是在Java 8新引入的，用于定义内存屏障，避免代码重排序，与Java内存模型相关，感兴趣的可以看博主的另一篇博文[全面理解Java内存模型(JMM)及volatile关键字](http://blog.csdn.net/javazejian/article/details/72772461)，这里就不展开了

_//在该方法之前的所有读操作，一定在load屏障之前执行完成_
publicnativevoidloadFence();
_//在该方法之前的所有写操作，一定在store屏障之前执行完成_
publicnativevoidstoreFence();
_//在该方法之前的所有读写操作，一定在full屏障之前执行完成，这个内存屏障相当于上面两个的合体功能_
publicnativevoidfullFence();

1. 其他操作

_//获取持有锁，已不建议使用_
@Deprecated
publicnativevoidmonitorEnter(Object var1);
_//释放锁，已不建议使用_
@Deprecated
publicnativevoidmonitorExit(Object var1);
_//尝试获取锁，已不建议使用_
@Deprecated
publicnativebooleantryMonitorEnter(Object var1);

_//获取本机内存的页数，这个值永远都是2的幂次方  _
publicnativeintpageSize();

_//告诉虚拟机定义了一个没有安全检查的类，默认情况下这个类加载器和保护域来着调用者类  _
publicnative Class defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);

_//加载一个匿名类_
publicnative Class defineAnonymousClass(Class hostClass, byte[] data, Object[] cpPatches);
_//判断是否需要加载一个类_
publicnativebooleanshouldBeInitialized(Class\&lt;?\&gt; c);
_//确保类一定被加载_
publicnative  voidensureClassInitialized(Class\&lt;?\&gt; c)

# 并发包中的原子操作类(Atomic系列)

通过前面的分析我们已基本理解了无锁CAS的原理并对Java中的指针类Unsafe类有了比较全面的认识，下面进一步分析CAS在Java中的应用，即并发包中的原子操作类(Atomic系列)，从JDK 1.5开始提供了java.util.concurrent.atomic包，在该包中提供了许多基于CAS实现的原子操作类，用法方便，性能高效，主要分以下4种类型。

**原子更新基本类型**

原子更新基本类型主要包括3个类：

1. AtomicBoolean：原子更新布尔类型
2. AtomicInteger：原子更新整型
3. AtomicLong：原子更新长整型

这3个类的实现原理和使用方式几乎是一样的，这里我们以AtomicInteger为例进行分析，AtomicInteger主要是针对int类型的数据执行原子操作，它提供了原子自增方法、原子自减方法以及原子赋值方法等，鉴于AtomicInteger的源码不多，我们直接看源码

publicclass AtomicInteger extends Number implements java.io.Serializable {
    privatestaticfinallong serialVersionUID = 6214790243416807050L;

    _// 获取指针类Unsafe_
    privatestaticfinal Unsafe unsafe = Unsafe.getUnsafe();

    _//下述变量value在AtomicInteger实例对象内的内存偏移量_
    privatestaticfinallong valueOffset;

    static {
        try {
           _//通过unsafe类的objectFieldOffset()方法，获取value变量在对象内存中的偏移_
           _//通过该偏移量valueOffset，unsafe类的内部方法可以获取到变量value对其进行取值或赋值操作_
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField(&quot;value&quot;));
        } catch (Exception ex) { thrownew Error(ex); }
    }
   _//当前AtomicInteger封装的int变量value_
    privatevolatileint value;

    publicAtomicInteger(int initialValue) {
        value = initialValue;
    }
    publicAtomicInteger() {
    }
   _//获取当前最新值，_
    publicfinalintget() {
        return value;
    }
    _//设置当前值，具备volatile效果，方法用final修饰是为了更进一步的保证线程安全。_
    publicfinalvoidset(int newValue) {
        value = newValue;
    }
    _//最终会设置成newValue，使用该方法后可能导致其他线程在之后的一小段时间内可以获取到旧值，有点类似于延迟加载_
    publicfinalvoidlazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }
   _//设置新值并获取旧值，底层调用的是CAS操作即unsafe.compareAndSwapInt()方法_
    publicfinalintgetAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
   _//如果当前值为expect，则设置为update(当前值指的是value变量)_
    publicfinalbooleancompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    _//当前值加1返回旧值，底层CAS操作_
    publicfinalintgetAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
    _//当前值减1，返回旧值，底层CAS操作_
    publicfinalintgetAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
   _//当前值增加delta，返回旧值，底层CAS操作_
    publicfinalintgetAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
    _//当前值加1，返回新值，底层CAS操作_
    publicfinalintincrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    _//当前值减1，返回新值，底层CAS操作_
    publicfinalintdecrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }
   _//当前值增加delta，返回新值，底层CAS操作_
    publicfinalintaddAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }
   _//省略一些不常用的方法...._
}

通过上述的分析，可以发现AtomicInteger原子类的内部几乎是基于前面分析过Unsafe类中的CAS相关操作的方法实现的，这也同时证明AtomicInteger是基于无锁实现的，这里重点分析自增操作实现过程，其他方法自增实现原理一样。

_//当前值加1，返回新值，底层CAS操作_
public final intincrementAndGet() {
     returnunsafe.getAndAddInt(this, valueOffset, 1) + 1;
 }

我们发现AtomicInteger类中所有自增或自减的方法都间接调用Unsafe类中的getAndAddInt()方法实现了CAS操作，从而保证了线程安全，关于getAndAddInt其实前面已分析过，它是Unsafe类中1.8新增的方法，源码如下

_//Unsafe类中的getAndAddInt方法_
public final intgetAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }

可看出getAndAddInt通过一个while循环不断的重试更新要设置的值，直到成功为止，调用的是Unsafe类中的compareAndSwapInt方法，是一个CAS操作方法。这里需要注意的是，上述源码分析是基于JDK1.8的，如果是1.8之前的方法，AtomicInteger源码实现有所不同，是基于for死循环的，如下

_//JDK 1.7的源码，由for的死循环实现，并且直接在AtomicInteger实现该方法，_
_//JDK1.8后，该方法实现已移动到Unsafe类中，直接调用getAndAddInt方法即可_
public final intincrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}

ok~,下面简单看个Demo，感受一下AtomicInteger使用方式

publicclass AtomicIntegerDemo {
    _//创建AtomicInteger,用于自增操作_
    static AtomicInteger i=new AtomicInteger();

    publicstaticclass AddThread implements Runnable{
        publicvoidrun(){
           for(int k=0;k\&lt;10000;k++)
               i.incrementAndGet();
        }

    }
    publicstaticvoidmain(String[] args) throws InterruptedException {
        Thread[] ts=new Thread[10];
        _//开启10条线程同时执行i的自增操作_
        for(int k=0;k\&lt;10;k++){
            ts[k]=new Thread(new AddThread());
        }
        _//启动线程_
        for(int k=0;k\&lt;10;k++){ts[k].start();}

        for(int k=0;k\&lt;10;k++){ts[k].join();}

        System.out.println(i);_//输出结果:100000_
    }
}

在Demo中，使用原子类型AtomicInteger替换普通int类型执行自增的原子操作，保证了线程安全。至于AtomicBoolean和AtomicLong的使用方式以及实现原理是一样，大家可以自行查阅源码。

**原子更新引用**

原子更新引用类型可以同时更新引用类型，这里主要分析一下AtomicReference原子类，即原子更新引用类型。先看看其使用方式，如下

publicclass AtomicReferenceDemo2 {

    publicstatic AtomicReference\&lt;User\&gt; atomicUserRef = new AtomicReference\&lt;User\&gt;();

    publicstaticvoidmain(String[] args) {
        User user = new User(&quot;zejian&quot;, 18);
        atomicUserRef.set(user);
        User updateUser = new User(&quot;Shine&quot;, 25);
        atomicUserRef.compareAndSet(user, updateUser);
        _//执行结果:User{name=&#39;Shine&#39;, age=25}_
              System.out.println(atomicUserRef.get().toString());
    }

    static class User {
        public String name;
        privateint age;

        publicUser(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        @Override
        public String toString() {
            return&quot;User{&quot; +
                    &quot;name=&#39;&quot; + name + &#39;\&#39;&#39; +
                    &quot;, age=&quot; + age +
                    &#39;}&#39;;
        }
    }
}

那么AtomicReference原子类内部是如何实现CAS操作的呢？

publicclass AtomicReference\&lt;V\&gt; implements java.io.Serializable {
    privatestatic final Unsafe unsafe = Unsafe.getUnsafe();
    privatestatic final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicReference.class.getDeclaredField(&quot;value&quot;));
        } catch (Exception ex) { thrownew Error(ex); }
    }
    _//内部变量value，Unsafe类通过valueOffset内存偏移量即可获取该变量_
    privatevolatile V value;

_//CAS方法，间接调用unsafe.compareAndSwapObject(),它是一个_
_//实现了CAS操作的native方法_
public final boolean compareAndSet(V expect, V update) {
        returnunsafe.compareAndSwapObject(this, valueOffset, expect, update);
}

_//设置并获取旧值_
public final V getAndSet(V newValue) {
        return (V)unsafe.getAndSetObject(this, valueOffset, newValue);
    }
    _//省略其他代码......_
}

_//Unsafe类中的getAndSetObject方法，实际调用还是CAS操作_
public final Object getAndSetObject(Object o, long offset, Object newValue) {
      Object v;
      do {
          v = getObjectVolatile(o, offset);
      } while (!compareAndSwapObject(o, offset, v, newValue));
      return v;
  }

从源码看来，AtomicReference与AtomicInteger的实现原理基本是一样的，最终执行的还是Unsafe类，关于AtomicReference的其他方法也是一样的，如下

![image](/images/2019\06\java\1570382490754.jpg "image")

红框内的方法是Java8新增的，可以基于Lambda表达式对传递进来的期望值或要更新的值进行其他操作后再进行CAS操作，说白了就是对期望值或要更新的值进行额外修改后再执行CAS更新，在所有的Atomic原子类中几乎都存在这几个方法。

**原子更新数组**

原子更新数组指的是通过原子的方式 **更新数组里的某个元素** ，主要有以下3个类

1. AtomicIntegerArray：原子更新整数数组里的元素
2. AtomicLongArray：原子更新长整数数组里的元素
3. AtomicReferenceArray：原子更新引用类型数组里的元素

这里以AtomicIntegerArray为例进行分析，其余两个使用方式和实现原理基本一样，简单案例如下，

publicclass AtomicIntegerArrayDemo {
    static AtomicIntegerArray arr = new AtomicIntegerArray(10);

    publicstaticclass AddThread implements Runnable{
        publicvoidrun(){
           for(int k=0;k\&lt;10000;k++)
               _//执行数组中元素自增操作,参数为index,即数组下标_
               arr.getAndIncrement(k%arr.length());
        }
    }
    publicstaticvoidmain(String[] args) throws InterruptedException {

        Thread[] ts=new Thread[10];
        _//创建10条线程_
        for(int k=0;k\&lt;10;k++){
            ts[k]=new Thread(new AddThread());
        }
        _//启动10条线程_
        for(int k=0;k\&lt;10;k++){ts[k].start();}
        for(int k=0;k\&lt;10;k++){ts[k].join();}
        _//执行结果_
        _//[10000, 10000, 10000, 10000, 10000, 10000, 10000, 10000, 10000, 10000]_
        System.out.println(arr);
    }
}

启动10条线程对数组中的元素进行自增操作，执行结果符合预期。使用方式比较简单，接着看看AtomicIntegerArray内部是如何实现，先看看部分源码

publicclass AtomicIntegerArray implements java.io.Serializable {
    _//获取unsafe类的实例对象_
    privatestaticfinal Unsafe unsafe = Unsafe.getUnsafe();
    _//获取数组的第一个元素内存起始地址_
    privatestaticfinalint base = unsafe.arrayBaseOffset(int[].class);

    privatestaticfinalint shift;
    _//内部数组_
    privatefinalint[] array;

    static {
        _//获取数组中一个元素占据的内存空间_
        int scale = unsafe.arrayIndexScale(int[].class);
        _//判断是否为2的次幂，一般为2的次幂否则抛异常_
        if ((scale &amp; (scale - 1)) != 0)
            thrownew Error(&quot;data type scale not a power of two&quot;);
        _//_
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    privatelongcheckedByteOffset(int i) {
        if (i \&lt; 0 || i \&gt;= array.length)
            thrownew IndexOutOfBoundsException(&quot;index &quot; + i);

        return byteOffset(i);
    }
    _//计算数组中每个元素的的内存地址_
    privatestaticlongbyteOffset(int i) {
        return ((long) i \&lt;\&lt; shift) + base;
    }
    _//省略其他代码......_
}

通过前面对Unsafe类的分析，我们知道arrayBaseOffset方法可以获取数组的第一个元素起始地址，而arrayIndexScale方法可以获取每个数组元素占用的内存空间，由于这里是Int类型，而Java中一个int类型占用4个字节，也就是scale的值为4，那么如何根据数组下标值计算每个元素的内存地址呢？显然应该是

每个数组元素的内存地址=起始地址+元素下标 \* 每个元素所占用的内存空间

与该方法原理相同

_//计算数组中每个元素的的内存地址_
privatestaticlongbyteOffset(int i) {
     return ((long) i \&lt;\&lt; shift) + base;
 }

这是为什么，首先来计算出shift的值

 shift = 31 - Integer.numberOfLeadingZeros(scale);

其中Integer.numberOfLeadingZeros(scale)是计算出scale的前导零个数(必须是连续的)，scale=4，转成二进制为

00000000 00000000 00000000 00000100

即前导零数为29，也就是shift=2，然后利用shift来定位数组中的内存位置，在数组不越界时，计算出前3个数组元素内存地址

_//第一个数组元素，index=0 ， 其中base为起始地址，4代表int类型占用的字节数_
address = base + 0 \* 4 即address= base + 0 \&lt;\&lt; 2
_//第二个数组元素，index=1_
address = base + 1 \* 4 即address= base + 1 \&lt;\&lt; 2
_//第三个数组元素，index=2_
address = base + 2 \* 4 即address= base + 2 \&lt;\&lt; 2
_//........_

显然shift=2，替换去就是

address= base + i \&lt;\&lt; shift

这就是 byteOffset(int i) 方法的计算原理。因此byteOffset(int)方法可以根据数组下标计算出每个元素的内存地址。至于其他方法就比较简单了，都是间接调用Unsafe类的CAS原子操作方法，如下简单看其中几个常用方法

_//执行自增操作，返回旧值，i是指数组元素下标_
publicfinalintgetAndIncrement(int i) {
      return getAndAdd(i, 1);
}
_//指定下标元素执行自增操作，并返回新值_
publicfinalintincrementAndGet(int i) {
    return getAndAdd(i, 1) + 1;
}

_//指定下标元素执行自减操作，并返回新值_
publicfinalintdecrementAndGet(int i) {
    return getAndAdd(i, -1) - 1;
}
_//间接调用unsafe.getAndAddInt()方法_
publicfinalintgetAndAdd(int i, int delta) {
    return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
}

_//Unsafe类中的getAndAddInt方法，执行CAS操作_
publicfinalintgetAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }

至于AtomicLongArray和AtomicReferenceArray原子类，使用方式和实现原理基本一样。

**原子更新属性**

如果我们只需要某个类里的某个字段，也就是说让普通的变量也享受原子操作，可以使用原子更新字段类，如在某些时候由于项目前期考虑不周全，项目需求又发生变化，使得某个类中的变量需要执行多线程操作，由于该变量多处使用，改动起来比较麻烦，而且原来使用的地方无需使用线程安全，只要求新场景需要使用时，可以借助原子更新器处理这种场景，Atomic并发包提供了以下三个类：

1. AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。
2. AtomicLongFieldUpdater：原子更新长整型字段的更新器。
3. AtomicReferenceFieldUpdater：原子更新引用类型里的字段。

请注意原子更新器的使用存在比较苛刻的条件如下

1. 操作的字段不能是static类型。
2. 操作的字段不能是final类型的，因为final根本没法修改。
3. 字段必须是volatile修饰的，也就是数据本身是读一致的。
4. 属性必须对当前的Updater所在的区域是可见的，如果不是当前类内部进行原子更新器操作不能使用private，protected子类操作父类时修饰符必须是protect权限及以上，如果在同一个package下则必须是default权限及以上，也就是说无论何时都应该保证操作类与被操作类间的可见性。

下面看看AtomicIntegerFieldUpdater和AtomicReferenceFieldUpdater的简单使用方式

publicclass AtomicIntegerFieldUpdaterDemo {
    publicstaticclass Candidate{
        int id;
        volatileint score;
    }

    publicstaticclass Game{
        int id;
        volatile String name;

        publicGame(int id, String name) {
            this.id = id;
            this.name = name;
        }

        @Override
        public String toString() {
            return&quot;Game{&quot; +
                    &quot;id=&quot; + id +
                    &quot;, name=&#39;&quot; + name + &#39;\&#39;&#39; +
                    &#39;}&#39;;
        }
    }

    static AtomicIntegerFieldUpdater\&lt;Candidate\&gt; atIntegerUpdater
        = AtomicIntegerFieldUpdater.newUpdater(Candidate.class, &quot;score&quot;);

    static AtomicReferenceFieldUpdater\&lt;Game,String\&gt; atRefUpdate =
            AtomicReferenceFieldUpdater.newUpdater(Game.class,String.class,&quot;name&quot;);


    _//用于验证分数是否正确_
    publicstatic AtomicInteger allScore=new AtomicInteger(0);


    publicstaticvoidmain(String[] args) throws InterruptedException {
        final Candidate stu=new Candidate();
        Thread[] t=new Thread[10000];
        _//开启10000个线程_
        for(int i = 0 ; i \&lt; 10000 ; i++) {
            t[i]=new Thread() {
                publicvoidrun() {
                    if(Math.random()\&gt;0.4){
                        atIntegerUpdater.incrementAndGet(stu);
                        allScore.incrementAndGet();
                    }
                }
            };
            t[i].start();
        }

        for(int i = 0 ; i \&lt; 10000 ; i++) {  t[i].join();}
        System.out.println(&quot;最终分数score=&quot;+stu.score);
        System.out.println(&quot;校验分数allScore=&quot;+allScore);

        _//AtomicReferenceFieldUpdater 简单的使用_
        Game game = new Game(2,&quot;zh&quot;);
        atRefUpdate.compareAndSet(game,game.name,&quot;JAVA-HHH&quot;);
        System.out.println(game.toString());

        /\*\*
         \* 输出结果:
         \* 最终分数score=5976
           校验分数allScore=5976
           Game{id=2, name=&#39;JAVA-HHH&#39;}
         \*/
    }
}

我们使用AtomicIntegerFieldUpdater更新候选人(Candidate)的分数score，开启了10000条线程投票，当随机值大于0.4时算一票，分数自增一次，其中allScore用于验证分数是否正确(其实用于验证AtomicIntegerFieldUpdater更新的字段是否线程安全)，当allScore与score相同时，则说明投票结果无误，也代表AtomicIntegerFieldUpdater能正确更新字段score的值，是线程安全的。对于AtomicReferenceFieldUpdater，我们在代码中简单演示了其使用方式，注意在AtomicReferenceFieldUpdater注明泛型时需要两个泛型参数，一个是修改的类类型，一个修改字段的类型。至于AtomicLongFieldUpdater则与AtomicIntegerFieldUpdater类似，不再介绍。接着简单了解一下AtomicIntegerFieldUpdater的实现原理，实际就是反射和Unsafe类结合，AtomicIntegerFieldUpdater是个抽象类，实际实现类为AtomicIntegerFieldUpdaterImpl

publicabstractclass AtomicIntegerFieldUpdater\&lt;T\&gt; {

    publicstatic \&lt;U\&gt; AtomicIntegerFieldUpdater\&lt;U\&gt; newUpdater(Class\&lt;U\&gt; tclass,
                                                              String fieldName) {
         _//实际实现类AtomicIntegerFieldUpdaterImpl                                          _
        returnnew AtomicIntegerFieldUpdaterImpl\&lt;U\&gt;
            (tclass, fieldName, Reflection.getCallerClass());
    }
 }

看看AtomicIntegerFieldUpdaterImpl

 privatestaticclass AtomicIntegerFieldUpdaterImpl\&lt;T\&gt;
            extends AtomicIntegerFieldUpdater\&lt;T\&gt; {
        privatestaticfinal Unsafe unsafe = Unsafe.getUnsafe();
        privatefinallong offset;_//内存偏移量_
        privatefinal Class\&lt;T\&gt; tclass;
        privatefinal Class\&lt;?\&gt; cclass;

        AtomicIntegerFieldUpdaterImpl(final Class\&lt;T\&gt; tclass,
                                      final String fieldName,
                                      final Class\&lt;?\&gt; caller) {
            final Field field;_//要修改的字段_
            finalint modifiers;_//字段修饰符_
            try {
                field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction\&lt;Field\&gt;() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);_//反射获取字段对象_
                        }
                    });
                    _//获取字段修饰符_
                modifiers = field.getModifiers();
            _//对字段的访问权限进行检查,不在访问范围内抛异常_
                sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
                ClassLoader cl = tclass.getClassLoader();
                ClassLoader ccl = caller.getClassLoader();
                if ((ccl != null) &amp;&amp; (ccl != cl) &amp;&amp;
                    ((cl == null) || !isAncestor(cl, ccl))) {
              sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
                }
            } catch (PrivilegedActionException pae) {
                thrownew RuntimeException(pae.getException());
            } catch (Exception ex) {
                thrownew RuntimeException(ex);
            }

            Class\&lt;?\&gt; fieldt = field.getType();
            _//判断是否为int类型_
            if (fieldt != int.class)
                thrownew IllegalArgumentException(&quot;Must be integer type&quot;);
            _//判断是否被volatile修饰_
            if (!Modifier.isVolatile(modifiers))
                thrownew IllegalArgumentException(&quot;Must be volatile type&quot;);

            this.cclass = (Modifier.isProtected(modifiers) &amp;&amp;
                           caller != tclass) ? caller : null;
            this.tclass = tclass;
            _//获取该字段的在对象内存的偏移量，通过内存偏移量可以获取或者修改该字段的值_
            offset = unsafe.objectFieldOffset(field);
        }
        }

从AtomicIntegerFieldUpdaterImpl的构造器也可以看出更新器为什么会有这么多限制条件了，当然最终其CAS操作肯定是通过unsafe完成的，简单看一个方法

publicintincrementAndGet(T obj) {
        int prev, next;
        do {
            prev = get(obj);
            next = prev + 1;
            _//CAS操作_
        } while (!compareAndSet(obj, prev, next));
        return next;
}

_//最终调用的还是unsafe.compareAndSwapInt()方法_
public boolean compareAndSet(T obj, int expect, int update) {
            if (obj == null || obj.getClass() != tclass || cclass != null) fullCheck(obj);
            returnunsafe.compareAndSwapInt(obj, offset, expect, update);
        }

**CAS的ABA问题及其解决方案**

假设这样一种场景，当第一个线程执行CAS(V,E,U)操作，在获取到当前变量V，准备修改为新值U前，另外两个线程已连续修改了两次变量V的值，使得该值又恢复为旧值，这样的话，我们就无法正确判断这个变量是否已被修改过，如下图

![image](/images/2019\06\java\1570382490764.jpg "image")

这就是典型的CAS的ABA问题，一般情况这种情况发现的概率比较小，可能发生了也不会造成什么问题，比如说我们对某个做加减法，不关心数字的过程，那么发生ABA问题也没啥关系。但是在某些情况下还是需要防止的，那么该如何解决呢？在Java中解决ABA问题，我们可以使用以下两个原子类

1. AtomicStampedReference

AtomicStampedReference原子类是一个带有时间戳的对象引用，在每次修改后，AtomicStampedReference不仅会设置新值而且还会记录更改的时间。当AtomicStampedReference设置对象值时，对象值以及时间戳都必须满足期望值才能写入成功，这也就解决了反复读写时，无法预知值是否已被修改的窘境，测试demo如下

/\*\*
 \* Created by zejian on 2017/7/2.
 \* Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 \*/
publicclass ABADemo {

    static AtomicInteger atIn = new AtomicInteger(100);

    _//初始化时需要传入一个初始值和初始时间_
    static AtomicStampedReference\&lt;Integer\&gt; atomicStampedR =
            new AtomicStampedReference\&lt;Integer\&gt;(200,0);


    static Thread t1 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            _//更新为200_
            atIn.compareAndSet(100, 200);
            _//更新为100_
            atIn.compareAndSet(200, 100);
        }
    });


    static Thread t2 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean flag=atIn.compareAndSet(100,500);
            System.out.println(&quot;flag:&quot;+flag+&quot;,newValue:&quot;+atIn);
        }
    });


    static Thread t3 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            int time=atomicStampedR.getStamp();
            _//更新为200_
            atomicStampedR.compareAndSet(100, 200,time,time+1);
            _//更新为100_
            int time2=atomicStampedR.getStamp();
            atomicStampedR.compareAndSet(200, 100,time2,time2+1);
        }
    });


    static Thread t4 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            int time = atomicStampedR.getStamp();
            System.out.println(&quot;sleep 前 t4 time:&quot;+time);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean flag=atomicStampedR.compareAndSet(100,500,time,time+1);
            System.out.println(&quot;flag:&quot;+flag+&quot;,newValue:&quot;+atomicStampedR.getReference());
        }
    });

    publicstatic  void  main(String[] args) throws InterruptedException {
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        t3.start();
        t4.start();
        /\*\*
         \* 输出结果:
         flag:true,newValue:500
         sleep 前 t4 time:0
         flag:false,newValue:200
         \*/
    }
}

对比输出结果可知，AtomicStampedReference类确实解决了ABA的问题，下面我们简单看看其内部实现原理

publicclass AtomicStampedReference\&lt;V\&gt; {
    _//通过Pair内部类存储数据和时间戳_
    privatestaticclass Pair\&lt;T\&gt; {
        final T reference;
        final int stamp;
        privatePair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static \&lt;T\&gt; Pair\&lt;T\&gt; of(T reference, int stamp) {
            returnnew Pair\&lt;T\&gt;(reference, stamp);
        }
    }
    _//存储数值和时间的内部类_
    privatevolatile Pair\&lt;V\&gt; pair;

    _//构造器，创建时需传入初始值和时间初始值_
    publicAtomicStampedReference(V initialRef, int initialStamp) {
        pair = Pair.of(initialRef, initialStamp);
    }
}

接着看看其compareAndSet方法的实现：

publicbooleancompareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair\&lt;V\&gt; current = pair;
        return
            expectedReference == current.reference &amp;&amp;
            expectedStamp == current.stamp &amp;&amp;
            ((newReference == current.reference &amp;&amp;
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }

同时对当前数据和当前时间进行比较，只有两者都相等是才会执行casPair()方法，单从该方法的名称就可知是一个CAS方法，最终调用的还是Unsafe类中的compareAndSwapObject方法

privatebooleancasPair(Pair\&lt;V\&gt; cmp, Pair\&lt;V\&gt; val) {
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
    }

到这我们就很清晰AtomicStampedReference的内部实现思想了，通过一个键值对Pair存储数据和时间戳，在更新时对数据和时间戳进行比较，只有两者都符合预期才会调用Unsafe的compareAndSwapObject方法执行数值和时间戳替换，也就避免了ABA的问题。

1. AtomicMarkableReference类

AtomicMarkableReference与AtomicStampedReference不同的是，AtomicMarkableReference维护的是一个boolean值的标识，也就是说至于true和false两种切换状态，经过博主测试，这种方式并不能完全防止ABA问题的发生，只能减少ABA问题发生的概率。

publicclass ABADemo {
    static AtomicMarkableReference\&lt;Integer\&gt; atMarkRef =
              new AtomicMarkableReference\&lt;Integer\&gt;(100,false);

static Thread t5 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            boolean mark=atMarkRef.isMarked();
            System.out.println(&quot;mark:&quot;+mark);
            _//更新为200_
            System.out.println(&quot;t5 result:&quot;+atMarkRef.compareAndSet(atMarkRef.getReference(), 200,mark,!mark));
        }
    });

    static Thread t6 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            boolean mark2=atMarkRef.isMarked();
            System.out.println(&quot;mark2:&quot;+mark2);
            System.out.println(&quot;t6 result:&quot;+atMarkRef.compareAndSet(atMarkRef.getReference(), 100,mark2,!mark2));
        }
    });

    static Thread t7 = new Thread(new Runnable() {
        @Override
        publicvoidrun() {
            boolean mark=atMarkRef.isMarked();
            System.out.println(&quot;sleep 前 t7 mark:&quot;+mark);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean flag=atMarkRef.compareAndSet(100,500,mark,!mark);
            System.out.println(&quot;flag:&quot;+flag+&quot;,newValue:&quot;+atMarkRef.getReference());
        }
    });

    publicstatic  void  main(String[] args) throws InterruptedException {
        t5.start();t5.join();
        t6.start();t6.join();
        t7.start();

        /\*\*
         \* 输出结果:
         mark:false
         t5 result:true
         mark2:true
         t6 result:true
         sleep 前 t5 mark:false
         flag:true,newValue:500 ----\&gt;成功了.....说明还是发生ABA问题
         \*/
    }
}

AtomicMarkableReference的实现原理与AtomicStampedReference类似，这里不再介绍。到此，我们也明白了如果要完全杜绝ABA问题的发生，我们应该使用AtomicStampedReference原子类更新对象，而对于AtomicMarkableReference来说只能减少ABA问题的发生概率，并不能杜绝。

# 再谈自旋锁

自旋锁是一种假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这种方式确实也是可以提升效率的。但问题是当线程越来越多竞争很激烈时，占用CPU的时间变长会导致性能急剧下降，因此Java虚拟机内部一般对于自旋锁有一定的次数限制，可能是50或者100次循环后就放弃，直接挂起线程，让出CPU资源。如下通过AtomicReference可实现简单的自旋锁。

publicclass SpinLock {
  private AtomicReference\&lt;Thread\&gt; sign =new AtomicReference\&lt;\&gt;();

  publicvoidlock(){
    Thread current = Thread.currentThread();
    while(!sign .compareAndSet(null, current)){
    }
  }

  publicvoidunlock (){
    Thread current = Thread.currentThread();
    sign .compareAndSet(current, null);
  }
}

使用CAS原子操作作为底层实现，lock()方法将要更新的值设置为当前线程，并将预期值设置为null。unlock()函数将要更新的值设置为null，并预期值设置为当前线程。然后我们通过lock()和unlock来控制自旋锁的开启与关闭，注意这是一种非公平锁。事实上AtomicInteger(或者AtomicLong)原子类内部的CAS操作也是通过不断的自循环(while循环)实现，不过这种循环的结束条件是线程成功更新对于的值，但也是自旋锁的一种。

ok~，到此关于无锁并发的知识点暂且了解到这，本篇到此告一段落。

主要参考资料

《Java高并发程序设计》

《Java编程思想》
