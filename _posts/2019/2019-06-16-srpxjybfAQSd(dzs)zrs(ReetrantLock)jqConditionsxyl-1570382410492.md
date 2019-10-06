---
layout: post
title:  "深入剖析基于并发AQS的(独占锁)重入锁(ReetrantLock)及其Condition实现原理"
categories: java
tags: 高并发 AQS
author: jiangc
excerpt: 深入剖析基于并发AQS的(独占锁)重入锁(ReetrantLock)及其Condition实现原理
---
* content
{:toc}

> 转载：http://blog.csdn.net/javazejian/article/details/75043422
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

在阅读本篇博文前，建议有CAS知识储备，因为关于CAS的操作在ReetrantLock的实现原理中可是随处可见，如没有了解过CAS可以先看博主的另一篇博文【[Java并发编程-无锁CAS与Unsafe类及其并发包Atomic](http://blog.csdn.net/javazejian/article/details/72772470)】，以下是本篇的主要内容

1. [Lock接口](https://blog.csdn.net/javazejian/article/details/75043422#lock%E6%8E%A5%E5%8F%A3)
2. [重入锁ReetrantLock](https://blog.csdn.net/javazejian/article/details/75043422#%E9%87%8D%E5%85%A5%E9%94%81reetrantlock)
3. [并发基础组件AQS与ReetrantLock](https://blog.csdn.net/javazejian/article/details/75043422#%E5%B9%B6%E5%8F%91%E5%9F%BA%E7%A1%80%E7%BB%84%E4%BB%B6aqs%E4%B8%8Ereetrantlock)

1.
  1. [AQS工作原理概要](https://blog.csdn.net/javazejian/article/details/75043422#aqs%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E6%A6%82%E8%A6%81)
  2. [基于ReetrantLock分析AQS独占模式实现过程](https://blog.csdn.net/javazejian/article/details/75043422#%E5%9F%BA%E4%BA%8Ereetrantlock%E5%88%86%E6%9E%90aqs%E7%8B%AC%E5%8D%A0%E6%A8%A1%E5%BC%8F%E5%AE%9E%E7%8E%B0%E8%BF%87%E7%A8%8B)

1.
  1.
    1. [ReetrantLock中非公平锁](https://blog.csdn.net/javazejian/article/details/75043422#reetrantlock%E4%B8%AD%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81)
    2. [ReetrantLock中公平锁](https://blog.csdn.net/javazejian/article/details/75043422#reetrantlock%E4%B8%AD%E5%85%AC%E5%B9%B3%E9%94%81)

1.
  1. [关于synchronized 与ReentrantLock](https://blog.csdn.net/javazejian/article/details/75043422#%E5%85%B3%E4%BA%8Esynchronized-%E4%B8%8Ereentrantlock)

1. [神奇的Condition](https://blog.csdn.net/javazejian/article/details/75043422#%E7%A5%9E%E5%A5%87%E7%9A%84condition)

1.
  1. [关于Condition接口](https://blog.csdn.net/javazejian/article/details/75043422#%E5%85%B3%E4%BA%8Econdition%E6%8E%A5%E5%8F%A3)
  2. [Condition的使用案例-生产者消费者模式](https://blog.csdn.net/javazejian/article/details/75043422#condition%E7%9A%84%E4%BD%BF%E7%94%A8%E6%A1%88%E4%BE%8B-%E7%94%9F%E4%BA%A7%E8%80%85%E6%B6%88%E8%B4%B9%E8%80%85%E6%A8%A1%E5%BC%8F)
  3. [Condition的实现原理](https://blog.csdn.net/javazejian/article/details/75043422#condition%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

# Lock接口

前面我们详谈过解决多线程同步问题的关键字synchronized，synchronized属于隐式锁，即锁的持有与释放都是隐式的，我们无需干预，而本篇我们要讲解的是显式锁，即锁的持有和释放都必须由我们手动编写。在Java 1.5中，官方在concurrent并发包中加入了Lock接口，该接口中提供了lock()方法和unLock()方法对显式加锁和显式释放锁操作进行支持，简单了解一下代码编写，如下：

Lock lock = new ReentrantLock();
lock.lock();
try{
    _//临界区......_
}finally{
    lock.unlock();
}

正如代码所显示(ReentrantLock是Lock的实现类，稍后分析)，当前线程使用lock()方法与unlock()对临界区进行包围，其他线程由于无法持有锁将无法进入临界区直到当前线程释放锁，注意unlock()操作必须在finally代码块中，这样可以确保即使临界区执行抛出异常，线程最终也能正常释放锁，Lock接口还提供了锁以下相关方法

publicinterface Lock {
    _//加锁_
    void lock();

    _//解锁_
    void unlock();

    _//可中断获取锁，与lock()不同之处在于可响应中断操作，即在获_
    _//取锁的过程中可中断，注意synchronized在获取锁时是不可中断的_
    void lockInterruptibly() throws InterruptedException;

    _//尝试非阻塞获取锁，调用该方法后立即返回结果，如果能够获取则返回true，否则返回false_
    boolean tryLock();

    _//根据传入的时间段获取锁，在指定时间内没有获取锁则返回false，如果在指定时间内当前线程未被中并断获取到锁则返回true_
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    _//获取等待通知组件，该组件与当前锁绑定，当前线程只有获得了锁_
    _//才能调用该组件的wait()方法，而调用后，当前线程将释放锁。_
    Condition newCondition();

可见Lock对象锁还提供了synchronized所不具备的其他同步特性，如可中断锁的获取(synchronized在等待获取锁时是不可中的)，超时中断锁的获取，等待唤醒机制的多条件变量Condition等，这也使得Lock锁在使用上具有更大的灵活性。下面进一步分析Lock的实现类重入锁ReetrantLock。

# 重入锁ReetrantLock

重入锁ReetrantLock，JDK 1.5新增的类，实现了Lock接口，作用与synchronized关键字相当，但比synchronized更加灵活。ReetrantLock本身也是一种支持重进入的锁，即该锁可以支持一个线程对资源重复加锁，同时也支持公平锁与非公平锁。所谓的公平与非公平指的是在请求先后顺序上，先对锁进行请求的就一定先获取到锁，那么这就是公平锁，反之，如果对于锁的获取并没有时间上的先后顺序，如后请求的线程可能先获取到锁，这就是非公平锁，一般而言非，非公平锁机制的效率往往会胜过公平锁的机制，但在某些场景下，可能更注重时间先后顺序，那么公平锁自然是很好的选择。需要注意的是ReetrantLock支持对同一线程重加锁，但是加锁多少次，就必须解锁多少次，这样才可以成功释放锁。下面看看ReetrantLock的简单使用案例：

import java.util.concurrent.locks.ReentrantLock;

publicclass ReenterLock implements Runnable{
    publicstatic ReentrantLock lock=new ReentrantLock();
    publicstaticint i=0;
    @Override
    publicvoidrun() {
        for(int j=0;j\&lt;10000000;j++){
            lock.lock();
            _//支持重入锁_
            lock.lock();
            try{
                i++;
            }finally{
                _//执行两次解锁_
                lock.unlock();
                lock.unlock();
            }
        }
    }
    publicstaticvoidmain(String[] args) throws InterruptedException {
        ReenterLock tl=new ReenterLock();
        Thread t1=new Thread(tl);
        Thread t2=new Thread(tl);
        t1.start();t2.start();
        t1.join();t2.join();
        _//输出结果：20000000_
        System.out.println(i);
    }
}

代码非常简单，我们使用两个线程同时操作临界资源i，执行自增操作，使用ReenterLock进行加锁，解决线程安全问题，这里进行了两次重复加锁，由于ReenterLock支持重入，因此这样是没有问题的，需要注意的是在finally代码块中，需执行两次解锁操作才能真正成功地让当前执行线程释放锁，从这里看ReenterLock的用法还是非常简单的，除了实现Lock接口的方法，ReenterLock其他方法说明如下

_//查询当前线程保持此锁的次数。_
int getHoldCount()

_//返回目前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null。      _
protected  Thread   getOwner();

_//返回一个 collection，它包含可能正等待获取此锁的线程，其内部维持一个队列，这点稍后会分析。      _
protected  Collection\&lt;Thread\&gt;   getQueuedThreads();

_//返回正等待获取此锁的线程估计数。  _
int getQueueLength();

_// 返回一个 collection，它包含可能正在等待与此锁相关给定条件的那些线程。_
protected  Collection\&lt;Thread\&gt;   getWaitingThreads(Condition condition);

_//返回等待与此锁相关的给定条件的线程估计数。      _
int getWaitQueueLength(Condition condition);

_// 查询给定线程是否正在等待获取此锁。    _
boolean hasQueuedThread(Thread thread);

_//查询是否有些线程正在等待获取此锁。    _
boolean hasQueuedThreads();

_//查询是否有些线程正在等待与此锁有关的给定条件。    _
boolean hasWaiters(Condition condition);

_//如果此锁的公平设置为 true，则返回 true。    _
boolean isFair()

_//查询当前线程是否保持此锁。      _
boolean isHeldByCurrentThread()

_//查询此锁是否由任意线程保持。        _
boolean isLocked()

由于ReetrantLock锁在使用上还是比较简单的，也就暂且打住，下面着重分析一下ReetrantLock的内部实现原理，这才是本篇博文的重点。实际上ReetrantLock是基于AQS并发框架实现的,我们先深入了解AQS，然后一步步揭开ReetrantLock的内部实现原理。

# 并发基础组件AQS与ReetrantLock

**AQS工作原理概要**

AbstractQueuedSynchronizer又称为队列同步器(后面简称AQS)，它是用来构建锁或其他同步组件的基础框架，内部通过一个int类型的成员变量state来控制同步状态,当state=0时，则说明没有任何线程占有共享资源的锁，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待，AQS内部通过内部类Node构成FIFO的同步队列来完成线程获取锁的排队工作，同时利用内部类ConditionObject构建等待队列，当Condition调用wait()方法后，线程将会加入等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移动同步队列中进行锁竞争。注意这里涉及到两种队列，一种的同步队列，当线程请求锁而等待的后将加入同步队列等待，而另一种则是等待队列(可有多个)，通过Condition调用await()方法释放锁后，将加入等待队列。关于Condition的等待队列我们后面再分析，这里我们先来看看AQS中的同步队列模型，如下

/\*\*
 \* AQS抽象类
 \*/
publicabstractclass AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer{
_//指向同步队列队头_
privatetransientvolatile Node head;

_//指向同步的队尾_
privatetransientvolatile Node tail;

_//同步状态，0代表锁未被占用，1代表锁已被占用_
privatevolatileint state;

_//省略其他代码......_
}

![image](/images/2019\06\java\1570382410493.jpg "image")

head和tail分别是AQS中的变量，其中head指向同步队列的头部，注意head为空结点，不存储信息。而tail则是同步队列的队尾，同步队列采用的是双向链表的结构这样可方便队列进行结点增删操作。state变量则是代表同步状态，执行当线程调用lock方法进行加锁后，如果此时state的值为0，则说明当前线程可以获取到锁(在本篇文章中，锁和同步状态代表同一个意思)，同时将state设置为1，表示获取成功。如果state已为1，也就是当前锁已被其他线程持有，那么当前执行线程将被封装为Node结点加入同步队列等待。其中Node结点是对每一个访问同步代码的线程的封装，从图中的Node的数据结构也可看出，其包含了需要同步的线程本身以及线程的状态，如是否被阻塞，是否等待唤醒，是否已经被取消等。每个Node结点内部关联其前继结点prev和后继结点next，这样可以方便线程释放锁后快速唤醒下一个在等待的线程，Node是AQS的内部类，其数据结构如下：

staticfinal class Node {
    _//共享模式_
    staticfinal Node SHARED = new Node();
    _//独占模式_
    staticfinal Node EXCLUSIVE = null;

    _//标识线程已处于结束状态_
    staticfinalint CANCELLED =  1;
    _//等待被唤醒状态_
    staticfinalint SIGNAL    = -1;
    _//条件状态，_
    staticfinalint CONDITION = -2;
    _//在共享模式中使用表示获得的同步状态会被传播_
    staticfinalint PROPAGATE = -3;

    _//等待状态,存在CANCELLED、SIGNAL、CONDITION、PROPAGATE 4种_
    volatileint waitStatus;

    _//同步队列中前驱结点_
    volatile Node prev;

    _//同步队列中后继结点_
    volatile Node next;

    _//请求锁的线程_
    volatile Thread thread;

    _//等待队列中的后继结点，这个与Condition有关，稍后会分析_
    Node nextWaiter;

    _//判断是否为共享模式_
    finalboolean isShared() {
        return nextWaiter == SHARED;
    }

    _//获取前驱结点_
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            thrownew NullPointerException();
        else
            return p;
    }

    _//....._
}

其中SHARED和EXCLUSIVE常量分别代表共享模式和独占模式，所谓共享模式是一个锁允许多条线程同时操作，如信号量Semaphore采用的就是基于AQS的共享模式实现的，而独占模式则是同一个时间段只能有一个线程对共享资源进行操作，多余的请求线程需要排队等待，如ReentranLock。变量waitStatus则表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。

1. CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，其结点的waitStatus为CANCELLED，即结束状态，进入该状态后的结点将不会再变化。
2. SIGNAL：值为-1，被标识为该等待唤醒状态的后继结点，当其前继结点的线程释放了同步锁或被取消，将会通知该后继结点的线程执行。说白了，就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
3. CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
4. PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
5. 0状态：值为0，代表初始化状态。

pre和next，分别指向当前Node结点的前驱结点和后继结点，thread变量存储的请求锁的线程。nextWaiter，与Condition相关，代表等待队列中的后继结点，关于这点这里暂不深入，后续会有更详细的分析，嗯，到此我们对Node结点的数据结构也就比较清晰了。总之呢，AQS作为基础组件，对于锁的实现存在两种不同的模式，即共享模式(如Semaphore)和独占模式(如ReetrantLock)，无论是共享模式还是独占模式的实现类，其内部都是基于AQS实现的，也都维持着一个虚拟的同步队列，当请求锁的线程超过现有模式的限制时，会将线程包装成Node结点并将线程当前必要的信息存储到node结点中，然后加入同步队列等会获取锁，而这系列操作都有AQS协助我们完成，这也是作为基础组件的原因，无论是Semaphore还是ReetrantLock，其内部绝大多数方法都是间接调用AQS完成的，下面是AQS整体类图结构

![image](/images/2019\06\java\1570382410497.jpg "image")

这里以ReentrantLock为例，简单讲解ReentrantLock与AQS的关系

![image](/images/2019\06\java\1570382410500.jpg "image")

1. AbstractOwnableSynchronizer：抽象类，定义了存储独占当前锁的线程和获取的方法
2. AbstractQueuedSynchronizer：抽象类，AQS框架核心类，其内部以虚拟队列的方式管理线程的锁获取与锁释放，其中获取锁(tryAcquire方法)和释放锁(tryRelease方法)并没有提供默认实现，需要子类重写这两个方法实现具体逻辑，目的是使开发人员可以自由定义获取锁以及释放锁的方式。
3. Node：AbstractQueuedSynchronizer 的内部类，用于构建虚拟队列(链表双向链表)，管理需要获取锁的线程。
4. Sync：抽象类，是ReentrantLock的内部类，继承自AbstractQueuedSynchronizer，实现了释放锁的操作(tryRelease()方法)，并提供了lock抽象方法，由其子类实现。
5. NonfairSync：是ReentrantLock的内部类，继承自Sync，非公平锁的实现类。
6. FairSync：是ReentrantLock的内部类，继承自Sync，公平锁的实现类。
7. ReentrantLock：实现了Lock接口的，其内部类有Sync、NonfairSync、FairSync，在创建时可以根据fair参数决定创建NonfairSync(默认非公平锁)还是FairSync。

ReentrantLock内部存在3个实现类，分别是Sync、NonfairSync、FairSync，其中Sync继承自AQS实现了解锁tryRelease()方法，而NonfairSync(非公平锁)、 FairSync(公平锁)则继承自Sync，实现了获取锁的tryAcquire()方法，ReentrantLock的所有方法调用都通过间接调用AQS和Sync类及其子类来完成的。从上述类图可以看出AQS是一个抽象类，但请注意其源码中并没一个抽象的方法，这是因为AQS只是作为一个基础组件，并不希望直接作为直接操作类对外输出，而更倾向于作为基础组件，为真正的实现类提供基础设施，如构建同步队列，控制同步状态等，事实上，从设计模式角度来看，AQS采用的模板模式的方式构建的，其内部除了提供并发操作核心方法以及同步队列操作外，还提供了一些模板方法让子类自己实现，如加锁操作以及解锁操作，为什么这么做？这是因为AQS作为基础组件，封装的是核心并发操作，但是实现上分为两种模式，即共享模式与独占模式，而这两种模式的加锁与解锁实现方式是不一样的，但AQS只关注内部公共方法实现并不关心外部不同模式的实现，所以提供了模板方法给子类使用，也就是说实现独占锁，如ReentrantLock需要自己实现tryAcquire()方法和tryRelease()方法，而实现共享模式的Semaphore，则需要实现tryAcquireShared()方法和tryReleaseShared()方法，这样做的好处是显而易见的，无论是共享模式还是独占模式，其基础的实现都是同一套组件(AQS)，只不过是加锁解锁的逻辑不同罢了，更重要的是如果我们需要自定义锁的话，也变得非常简单，只需要选择不同的模式实现不同的加锁和解锁的模板方法即可，AQS提供给独占模式和共享模式的模板方法如下

_//AQS中提供的主要模板方法，由子类实现。_
publicabstractclass AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer{

    _//独占模式下获取锁的方法_
    protectedbooleantryAcquire(int arg) {
        thrownew UnsupportedOperationException();
    }

    _//独占模式下解锁的方法_
    protectedbooleantryRelease(int arg) {
        thrownew UnsupportedOperationException();
    }

    _//共享模式下获取锁的方法_
    protectedinttryAcquireShared(int arg) {
        thrownew UnsupportedOperationException();
    }

    _//共享模式下解锁的方法_
    protectedbooleantryReleaseShared(int arg) {
        thrownew UnsupportedOperationException();
    }
    _//判断是否为持有独占锁_
    protectedbooleanisHeldExclusively() {
        thrownew UnsupportedOperationException();
    }

}

在了解AQS的原理概要后，下面我们就基于ReetrantLock进一步分析AQS的实现过程，这也是ReetrantLock的内部实现原理。

**基于ReetrantLock分析AQS独占模式实现过程**

**ReetrantLock中非公平锁**

AQS同步器的实现依赖于内部的同步队列(FIFO的双向链表对列)完成对同步状态(state)的管理，当前线程获取锁(同步状态)失败时，AQS会将该线程以及相关等待信息包装成一个节点(Node)并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会将头结点head中的线程唤醒，让其尝试获取同步状态。关于同步队列和Node结点，前面我们已进行了较为详细的分析，这里重点分析一下获取同步状态和释放同步状态以及如何加入队列的具体操作，这里从ReetrantLock入手分析AQS的具体实现，我们先以非公平锁为例进行分析。

_//默认构造，创建非公平锁NonfairSync_
publicReentrantLock() {
    sync = new NonfairSync();
}
_//根据传入参数创建锁类型_
publicReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}

_//加锁操作_
publicvoidlock() {
     sync.lock();
}

前面说过sync是个抽象类，存在两个不同的实现子类，这里从非公平锁入手，看看其实现

/\*\*
 \* 非公平锁实现
 \*/
staticfinal class NonfairSync extends Sync {
    _//加锁_
    finalvoid lock() {
        _//执行CAS操作，获取同步状态_
        if (compareAndSetState(0, 1))
       _//成功则将独占锁线程设置为当前线程  _
          setExclusiveOwnerThread(Thread.currentThread());
        else
            _//否则再次请求同步状态_
            acquire(1);
    }
}

这里获取锁时，首先对同步状态执行CAS操作，尝试把state的状态从0设置为1，如果返回true则代表获取同步状态成功，也就是当前线程获取锁成，可操作临界资源，如果返回false，则表示已有线程持有该同步状态(其值为1)，获取锁失败，注意这里存在并发的情景，也就是可能同时存在多个线程设置state变量，因此是CAS操作保证了state变量操作的原子性。返回false后，执行 acquire(1)方法，该方法是AQS中的方法，它对中断不敏感，即使线程获取同步状态失败，进入同步队列，后续对该线程执行中断操作也不会从同步队列中移出，方法如下

publicfinalvoidacquire(int arg) {
    _//再次尝试获取同步状态_
    if (!tryAcquire(arg) &amp;&amp;
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

这里传入参数arg表示要获取同步状态后设置的值(即要设置state的值)，因为要获取锁，而status为0时是释放锁，1则是获取锁，所以这里一般传递参数为1，进入方法后首先会执行tryAcquire(arg)方法，在前面分析过该方法在AQS中并没有具体实现，而是交由子类实现，因此该方法是由ReetrantLock类内部实现的

_//NonfairSync类_
staticfinalclass NonfairSync extends Sync {

    protectedfinalboolean tryAcquire(int acquires) {
         return nonfairTryAcquire(acquires);
     }
 }

_//Sync类_
abstractstaticclass Sync extends AbstractQueuedSynchronizer {

  _//nonfairTryAcquire方法_
  finalboolean nonfairTryAcquire(int acquires) {
      final Thread current = Thread.currentThread();
      int c = getState();
      _//判断同步状态是否为0，并尝试再次获取同步状态_
      if (c == 0) {
          _//执行CAS操作_
          if (compareAndSetState(0, acquires)) {
              setExclusiveOwnerThread(current);
              returntrue;
          }
      }
      _//如果当前线程已获取锁，属于重入锁，再次获取锁后将status值加1_
      elseif (current == getExclusiveOwnerThread()) {
          int nextc = c + acquires;
          if (nextc \&lt; 0) _// overflow_
              thrownew Error(&quot;Maximum lock count exceeded&quot;);
          _//设置当前同步状态，当前只有一个线程持有锁，因为不会发生线程安全问题，可以直接执行 setState(nextc);_
          setState(nextc);
          returntrue;
      }
      returnfalse;
  }
  _//省略其他代码_
}

从代码执行流程可以看出，这里做了两件事，一是尝试再次获取同步状态，如果获取成功则将当前线程设置为OwnerThread，否则失败，二是判断当前线程current是否为OwnerThread，如果是则属于重入锁，state自增1，并获取锁成功，返回true，反之失败，返回false，也就是tryAcquire(arg)执行失败，返回false。需要注意的是nonfairTryAcquire(int acquires)内部使用的是CAS原子性操作设置state值，可以保证state的更改是线程安全的，因此只要任意一个线程调用nonfairTryAcquire(int acquires)方法并设置成功即可获取锁，不管该线程是新到来的还是已在同步队列的线程，毕竟这是非公平锁，并不保证同步队列中的线程一定比新到来线程请求(可能是head结点刚释放同步状态然后新到来的线程恰好获取到同步状态)先获取到锁，这点跟后面还会讲到的公平锁不同。ok~，接着看之前的方法acquire(int arg)

publicfinalvoidacquire(int arg) {
    _//再次尝试获取同步状态_
    if (!tryAcquire(arg) &amp;&amp;
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

如果tryAcquire(arg)返回true，acquireQueued自然不会执行，这是最理想的，因为毕竟当前线程已获取到锁，如果tryAcquire(arg)返回false，则会执行addWaiter(Node.EXCLUSIVE)进行入队操作,由于ReentrantLock属于独占锁，因此结点类型为Node.EXCLUSIVE，下面看看addWaiter方法具体实现

private Node addWaiter(Node mode) {
    _//将请求同步状态失败的线程封装成结点_
    Node node = new Node(Thread.currentThread(), mode);

    Node pred = tail;
    _//如果是第一个结点加入肯定为空，跳过。_
    _//如果非第一个结点则直接执行CAS入队操作，尝试在尾部快速添加_
    if (pred != null) {
        node.prev = pred;
        _//使用CAS执行尾部结点替换，尝试在尾部快速添加_
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    _//如果第一次加入或者CAS操作没有成功执行enq入队操作_
    enq(node);
    return node;
}

创建了一个Node.EXCLUSIVE类型Node结点用于封装线程及其相关信息，其中tail是AQS的成员变量，指向队尾(这点前面的我们分析过AQS维持的是一个双向的链表结构同步队列)，如果是第一个结点，则为tail肯定为空，那么将执行enq(node)操作，如果非第一个结点即tail指向不为null，直接尝试执行CAS操作加入队尾，如果CAS操作失败还是会执行enq(node)，继续看enq(node)：

private Node enq(final Node node) {
    _//死循环_
    for (;;) {
         Node t = tail;
         _//如果队列为null，即没有头结点_
         if (t == null) { _// Must initialize_
             _//创建并使用CAS设置头结点_
             if (compareAndSetHead(new Node()))
                 tail = head;
         } else {_//队尾添加新结点_
             node.prev = t;
             if (compareAndSetTail(t, node)) {
                 t.next = node;
                 return t;
             }
         }
     }
    }

这个方法使用一个死循环进行CAS操作，可以解决多线程并发问题。这里做了两件事，一是如果还没有初始同步队列则创建新结点并使用compareAndSetHead设置头结点，tail也指向head，二是队列已存在，则将新结点node添加到队尾。注意这两个步骤都存在同一时间多个线程操作的可能，如果有一个线程修改head和tail成功，那么其他线程将继续循环，直到修改成功，这里使用CAS原子操作进行头结点设置和尾结点tail替换可以保证线程安全，从这里也可以看出head结点本身不存在任何数据，它只是作为一个牵头结点，而tail永远指向尾部结点(前提是队列不为null)。

![image](/images/2019\06\java\1570382410505.jpg "image")

添加到同步队列后，结点就会进入一个自旋过程，即每个结点都在观察时机待条件满足获取同步状态，然后从同步队列退出并结束自旋，回到之前的acquire()方法，自旋过程是在acquireQueued(addWaiter(Node.EXCLUSIVE), arg))方法中执行的，代码如下

finalboolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        _//自旋，死循环_
        for (;;) {
            _//获取前驱结点_
            final Node p = node.predecessor();
            当且仅当p为头结点才尝试获取同步状态
            if (p == head &amp;&amp; tryAcquire(arg)) {
                _//将node设置为头结点_
                setHead(node);
                _//清空原来头结点的引用便于GC_
                p.next = null; _// help GC_
                failed = false;
                return interrupted;
            }
            _//如果前驱结点不是head，判断是否挂起线程_
            if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            _//最终都没能获取同步状态，结束该线程的请求_
            cancelAcquire(node);
    }
}

当前线程在自旋(死循环)中获取同步状态，当且仅当前驱结点为头结点才尝试获取同步状态，这符合FIFO的规则，即先进先出，其次head是当前获取同步状态的线程结点，只有当head释放同步状态唤醒后继结点，后继结点才有可能获取到同步状态，因此后继结点在其前继结点为head时，才进行尝试获取同步状态，其他时刻将被挂起。进入if语句后调用setHead(node)方法，将当前线程结点设置为head

_//设置为头结点_
privatevoidsetHead(Node node) {
        head = node;
        _//清空结点数据_
        node.thread = null;
        node.prev = null;
}

设置为node结点被设置为head后，其thread信息和前驱结点将被清空，因为该线程已获取到同步状态(锁)，正在执行了，也就没有必要存储相关信息了，head只有保存指向后继结点的指针即可，便于head结点释放同步状态后唤醒后继结点，执行结果如下图

![image](/images/2019\06\java\1570382410508.jpg "image")

从图可知更新head结点的指向，将后继结点的线程唤醒并获取同步状态，调用setHead(node)将其替换为head结点，清除相关无用数据。当然如果前驱结点不是head，那么执行如下

_//如果前驱结点不是head，判断是否挂起线程_
if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;parkAndCheckInterrupt())

      interrupted = true;
}

privatestaticbooleanshouldParkAfterFailedAcquire(Node pred, Node node) {
        _//获取当前结点的等待状态_
        int ws = pred.waitStatus;
        _//如果为等待唤醒（SIGNAL）状态则返回true_
        if (ws == Node.SIGNAL)
            returntrue;
        _//如果ws\&gt;0 则说明是结束状态，_
        _//遍历前驱结点直到找到没有结束状态的结点_
        if (ws \&gt; 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus \&gt; 0);
            pred.next = node;
        } else {
            _//如果ws小于0又不是SIGNAL状态，_
            _//则将其设置为SIGNAL状态，代表该结点的线程正在等待唤醒。_
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        returnfalse;
    }

privatefinalbooleanparkAndCheckInterrupt() {
        _//将当前线程挂起_
        LockSupport.park(this);
        _//获取线程中断状态,interrupted()是判断当前中断状态，_
        _//并非中断线程，因此可能true也可能false,并返回_
        return Thread.interrupted();
}

shouldParkAfterFailedAcquire()方法的作用是判断当前结点的前驱结点是否为SIGNAL状态(即等待唤醒状态)，如果是则返回true。如果结点的ws为CANCELLED状态(值为1\&gt;0),即结束状态，则说明该前驱结点已没有用应该从同步队列移除，执行while循环，直到寻找到非CANCELLED状态的结点。倘若前驱结点的ws值不为CANCELLED，也不为SIGNAL(当从Condition的条件等待队列转移到同步队列时，结点状态为CONDITION因此需要转换为SIGNAL)，那么将其转换为SIGNAL状态，等待被唤醒。

若shouldParkAfterFailedAcquire()方法返回true，即前驱结点为SIGNAL状态同时又不是head结点，那么使用parkAndCheckInterrupt()方法挂起当前线程，称为WAITING状态，需要等待一个unpark()操作来唤醒它，到此ReetrantLock内部间接通过AQS的FIFO的同步队列就完成了lock()操作，这里我们总结成逻辑流程图

![image](/images/2019\06\java\1570382410513.jpg "image")

关于获取锁的操作，这里看看另外一种可中断的获取方式，即调用ReentrantLock类的lockInterruptibly()或者tryLock()方法，最终它们都间接调用到doAcquireInterruptibly()

 privatevoiddoAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head &amp;&amp; tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; _// help GC_
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                    parkAndCheckInterrupt())
                    _//直接抛异常，中断线程的同步状态请求_
                    thrownew InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

最大的不同是

if (shouldParkAfterFailedAcquire(p, node) &amp;&amp;
                    parkAndCheckInterrupt())
     _//直接抛异常，中断线程的同步状态请求_
       thrownew InterruptedException();

检测到线程的中断操作后，直接抛出异常，从而中断线程的同步状态请求，移除同步队列，ok~,加锁流程到此。下面接着看unlock()操作

_//ReentrantLock类的unlock_
publicvoidunlock() {
    sync.release(1);
}

_//AQS类的release()方法_
publicfinalbooleanrelease(int arg) {
    _//尝试释放锁_
    if (tryRelease(arg)) {

        Node h = head;
        if (h != null &amp;&amp; h.waitStatus != 0)
            _//唤醒后继结点的线程_
            unparkSuccessor(h);
        returntrue;
    }
    returnfalse;
}

_//ReentrantLock类中的内部类Sync实现的tryRelease(int releases)_
protectedfinalbooleantryRelease(int releases) {

      int c = getState() - releases;
      if (Thread.currentThread() != getExclusiveOwnerThread())
          thrownew IllegalMonitorStateException();
      boolean free = false;
      _//判断状态是否为0，如果是则说明已释放同步状态_
      if (c == 0) {
          free = true;
          _//设置Owner为null_
          setExclusiveOwnerThread(null);
      }
      _//设置更新同步状态_
      setState(c);
      return free;
  }

释放同步状态的操作相对简单些，tryRelease(int releases)方法是ReentrantLock类中内部类自己实现的，因为AQS对于释放锁并没有提供具体实现，必须由子类自己实现。释放同步状态后会使用unparkSuccessor(h)唤醒后继结点的线程，这里看看unparkSuccessor(h)

privatevoidunparkSuccessor(Node node) {
    _//这里，node一般为当前线程所在的结点。_
    int ws = node.waitStatus;
    if (ws \&lt; 0)_//置零当前线程所在的结点状态，允许失败。_
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;_//找到下一个需要唤醒的结点s_
    if (s == null || s.waitStatus \&gt; 0) {_//如果为空或已取消_
        s = null;
        for (Node t = tail; t != null &amp;&amp; t != node; t = t.prev)
            if (t.waitStatus \&lt;= 0)_//从这里可以看出，\&lt;=0的结点，都是还有效的结点。_
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);_//唤醒_
}

从代码执行操作来看，这里主要作用是用unpark()唤醒同步队列中最前边未放弃线程(也就是状态为CANCELLED的线程结点s)。此时，回忆前面分析进入自旋的函数acquireQueued()，s结点的线程被唤醒后，会进入acquireQueued()函数的if (p == head &amp;&amp; tryAcquire(arg))的判断，如果p!=head也不会有影响，因为它会执行shouldParkAfterFailedAcquire()，由于s通过unparkSuccessor()操作后已是同步队列中最前边未放弃的线程结点，那么通过shouldParkAfterFailedAcquire()内部对结点状态的调整，s也必然会成为head的next结点，因此再次自旋时p==head就成立了，然后s把自己设置成head结点，表示自己已经获取到资源了，最终acquire()也返回了，这就是独占锁释放的过程。

ok~，关于独占模式的加锁和释放锁的过程到这就分析完，总之呢，在AQS同步器中维护着一个同步队列，当线程获取同步状态失败后，将会被封装成Node结点，加入到同步队列中并进行自旋操作，当当前线程结点的前驱结点为head时，将尝试获取同步状态，获取成功将自己设置为head结点。在释放同步状态时，则通过调用子类(ReetrantLock中的Sync内部类)的tryRelease(int releases)方法释放同步状态，释放成功则唤醒后继结点的线程。

**ReetrantLock中公平锁**

了解完ReetrantLock中非公平锁的实现后，我们再来看看公平锁。与非公平锁不同的是，在获取锁的时，公平锁的获取顺序是完全遵循时间上的FIFO规则，也就是说先请求的线程一定会先获取锁，后来的线程肯定需要排队，这点与前面我们分析非公平锁的nonfairTryAcquire(int acquires)方法实现有锁不同，下面是公平锁中tryAcquire()方法的实现

_//公平锁FairSync类中的实现_
protectedfinalbooleantryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            _//注意！！这里先判断同步队列是否存在结点_
                if (!hasQueuedPredecessors() &amp;&amp;
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    returntrue;
                }
            }
            elseif (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc \&lt; 0)
                    thrownew Error(&quot;Maximum lock count exceeded&quot;);
                setState(nextc);
                returntrue;
            }
            returnfalse;
        }

该方法与nonfairTryAcquire(int acquires)方法唯一的不同是在使用CAS设置尝试设置state值前，调用了hasQueuedPredecessors()判断同步队列是否存在结点，如果存在必须先执行完同步队列中结点的线程，当前线程进入等待状态。这就是非公平锁与公平锁最大的区别，即公平锁在线程请求到来时先会判断同步队列是否存在结点，如果存在先执行同步队列中的结点线程，当前线程将封装成node加入同步队列等待。而非公平锁呢，当线程请求到来时，不管同步队列是否存在线程结点，直接尝试获取同步状态，获取成功直接访问共享资源，但请注意在绝大多数情况下，非公平锁才是我们理想的选择，毕竟从效率上来说非公平锁总是胜于公平锁。

    以上便是ReentrantLock的内部实现原理，这里我们简单进行小结，重入锁ReentrantLock，是一个基于AQS并发框架的并发控制类，其内部实现了3个类，分别是Sync、NoFairSync以及FairSync类，其中Sync继承自AQS，实现了释放锁的模板方法tryRelease(int)，而NoFairSync和FairSync都继承自Sync，实现各种获取锁的方法tryAcquire(int)。ReentrantLock的所有方法实现几乎都间接调用了这3个类，因此当我们在使用ReentrantLock时，大部分使用都是在间接调用AQS同步器中的方法，这就是ReentrantLock的内部实现原理,最后给出张类图结构

![image](/images/2019\06\java\1570382410518.jpg "image")

**关于synchronized 与ReentrantLock**

在JDK 1.6之后，虚拟机对于synchronized关键字进行整体优化后，在性能上synchronized与ReentrantLock已没有明显差距，因此在使用选择上，需要根据场景而定，大部分情况下我们依然建议是synchronized关键字，原因之一是使用方便语义清晰，二是性能上虚拟机已为我们自动优化。而ReentrantLock提供了多样化的同步特性，如超时获取锁、可以被中断获取锁（synchronized的同步是不能中断的）、等待唤醒机制的多个条件变量(Condition)等，因此当我们确实需要使用到这些功能是，可以选择ReentrantLock

# 神奇的Condition

**关于Condition接口**

在并发编程中，每个Java对象都存在一组监视器方法，如wait()、notify()以及notifyAll()方法，通过这些方法，我们可以实现线程间通信与协作（也称为等待唤醒机制），如生产者-消费者模式，而且这些方法必须配合着synchronized关键字使用，关于这点，如果想有更深入的理解，可观看博主另外一篇博文【[ 深入理解Java并发之synchronized实现原理](http://blog.csdn.net/javazejian/article/details/72828483)】，与synchronized的等待唤醒机制相比Condition具有更多的灵活性以及精确性，这是因为notify()在唤醒线程时是随机(同一个锁)，而Condition则可通过多个Condition实例对象建立更加精细的线程控制，也就带来了更多灵活性了，我们可以简单理解为以下两点

1. 通过Condition能够精细的控制多线程的休眠与唤醒。
2. 对于一个锁，我们可以为多个线程间建立不同的Condition。

Condition是一个接口类，其主要方法如下：

publicinterface Condition {

/\*\*
  \* 使当前线程进入等待状态直到被通知(signal)或中断
  \* 当其他线程调用singal()或singalAll()方法时，该线程将被唤醒
  \* 当其他线程调用interrupt()方法中断当前线程
  \* await()相当于synchronized等待唤醒机制中的wait()方法
  \*/
void await() throws InterruptedException;

_//当前线程进入等待状态，直到被唤醒，该方法不响应中断要求_
void awaitUninterruptibly();

_//调用该方法，当前线程进入等待状态，直到被唤醒或被中断或超时_
_//其中nanosTimeout指的等待超时时间，单位纳秒_
long awaitNanos(long nanosTimeout) throws InterruptedException;

  _//同awaitNanos，但可以指明时间单位_
  boolean await(long time, TimeUnit unit) throws InterruptedException;

_//调用该方法当前线程进入等待状态，直到被唤醒、中断或到达某个时_
_//间期限(deadline),如果没到指定时间就被唤醒，返回true，其他情况返回false_
  boolean awaitUntil(Date deadline) throws InterruptedException;

_//唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须_
_//获取与Condition相关联的锁，功能与notify()相同_
  void signal();

_//唤醒所有等待在Condition上的线程，该线程从等待方法返回前必须_
_//获取与Condition相关联的锁，功能与notifyAll()相同_
  void signalAll();
}

关于Condition的实现类是AQS的内部类ConditionObject，关于这点我们稍后分析，这里先来看一个Condition的使用案例，即经典消费者生产者模式

**Condition的使用案例-生产者消费者模式**

这里我们通过一个卖烤鸭的案例来演示多生产多消费者的案例，该场景中存在两条生产线程t1和t2，用于生产烤鸭，也存在两条消费线程t3，t4用于消费烤鸭，4条线程同时执行，需要保证只有在生产线程产生烤鸭后，消费线程才能消费，否则只能等待，直到生产线程产生烤鸭后唤醒消费线程，注意烤鸭不能重复消费。ResourceByCondition类中定义product()和consume()两个方法，分别用于生产烤鸭和消费烤鸭，并且定义ReentrantLock锁，用于控制product()和consume()的并发，由于必须在烤鸭生成完成后消费线程才能消费烤鸭，否则只能等待，因此这里定义两组Condition对象，分别是producer\_con和consumer\_con，前者拥有控制生产线程，后者拥有控制消费线程，这里我们使用一个标志flag来控制是否有烤鸭，当flag为true时，代表烤鸭生成完毕，生产线程必须进入等待状态同时唤醒消费线程进行消费，消费线程消费完毕后将flag设置为false，代表烤鸭消费完成，进入等待状态，同时唤醒生产线程生产烤鸭，具体代码如下

package com.zejian.concurrencys;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/\*\*
 \* Created by zejian on 2017/7/22.
 \* Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 \*/
publicclass ResourceByCondition {
    private String name;
    privateint count = 1;
    privateboolean flag = false;

    _//创建一个锁对象。_
    Lock lock = new ReentrantLock();

    _//通过已有的锁获取两组监视器，一组监视生产者，一组监视消费者。_
    Condition producer\_con = lock.newCondition();
    Condition consumer\_con = lock.newCondition();

    /\*\*
     \* 生产
     \* @param name
     \*/
    public  voidproduct(String name)
    {
        lock.lock();
        try
        {
            while(flag){
                try{producer\_con.await();}catch(InterruptedException e){}
            }
            this.name = name + count;
            count++;
            System.out.println(Thread.currentThread().getName()+&quot;...生产者5.0...&quot;+this.name);
            flag = true;
            consumer\_con.signal();_//直接唤醒消费线程_
        }
        finally
        {
            lock.unlock();
        }
    }

    /\*\*
     \* 消费
     \*/
    public  voidconsume()
    {
        lock.lock();
        try
        {
            while(!flag){
                try{consumer\_con.await();}catch(InterruptedException e){}
            }
            System.out.println(Thread.currentThread().getName()+&quot;...消费者.5.0.......&quot;+this.name);_//消费烤鸭1_
            flag = false;
            producer\_con.signal();_//直接唤醒生产线程_
        }
        finally
        {
            lock.unlock();
        }
    }
}

执行代码

package com.zejian.concurrencys;
/\*\*
 \* Created by zejian on 2017/7/22.
 \* Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 \*/
publicclass Mutil\_Producer\_ConsumerByCondition {

    publicstaticvoidmain(String[] args) {
        ResourceByCondition r = new ResourceByCondition();
        Mutil\_Producer pro = new Mutil\_Producer(r);
        Mutil\_Consumer con = new Mutil\_Consumer(r);
        _//生产者线程_
        Thread t0 = new Thread(pro);
        Thread t1 = new Thread(pro);
        _//消费者线程_
        Thread t2 = new Thread(con);
        Thread t3 = new Thread(con);
        _//启动线程_
        t0.start();
        t1.start();
        t2.start();
        t3.start();
    }
}

/\*\*
 \* @decrition 生产者线程
 \*/
class Mutil\_Producer implements Runnable {
    private ResourceByCondition r;

    Mutil\_Producer(ResourceByCondition r) {
        this.r = r;
    }

    publicvoidrun() {
        while (true) {
            r.product(&quot;北京烤鸭&quot;);
        }
    }
}

/\*\*
 \* @decrition 消费者线程
 \*/
class Mutil\_Consumer implements Runnable {
    private ResourceByCondition r;

    Mutil\_Consumer(ResourceByCondition r) {
        this.r = r;
    }

    publicvoidrun() {
        while (true) {
            r.consume();
        }
    }
}

正如代码所示，我们通过两者Condition对象单独控制消费线程与生产消费，这样可以避免消费线程在唤醒线程时唤醒的还是消费线程，如果是通过synchronized的等待唤醒机制实现的话，就可能无法避免这种情况，毕竟同一个锁，对于synchronized关键字来说只能有一组等待唤醒队列，而不能像Condition一样，同一个锁拥有多个等待队列。synchronized的实现方案如下，

publicclass KaoYaResource {

    private String name;
    privateint count = 1;_//烤鸭的初始数量_
    privateboolean flag = false;_//判断是否有需要线程等待的标志_
    /\*\*
     \* 生产烤鸭
     \*/
    publicsynchronizedvoidproduct(String name){
        while(flag){
            _//此时有烤鸭，等待_
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        this.name=name+count;_//设置烤鸭的名称_
        count++;
        System.out.println(Thread.currentThread().getName()+&quot;...生产者...&quot;+this.name);
        flag=true;_//有烤鸭后改变标志_
        notifyAll();_//通知消费线程可以消费了_
    }

    /\*\*
     \* 消费烤鸭
     \*/
    publicsynchronizedvoidconsume(){
        while(!flag){_//如果没有烤鸭就等待_
            try{this.wait();}catch(InterruptedException e){}
        }
        System.out.println(Thread.currentThread().getName()+&quot;...消费者........&quot;+this.name);_//消费烤鸭1_
        flag = false;
        notifyAll();_//通知生产者生产烤鸭_
    }
}

如上代码，在调用notify()或者 notifyAll()方法时，由于等待队列中同时存在生产者线程和消费者线程，所以我们并不能保证被唤醒的到底是消费者线程还是生产者线程，而Codition则可以避免这种情况。嗯，了解完Condition的使用方式后，下面我们将进一步探讨Condition背后的实现机制

**Condition的实现原理**

Condition的具体实现类是AQS的内部类ConditionObject，前面我们分析过AQS中存在两种队列，一种是同步队列，一种是等待队列，而等待队列就相对于Condition而言的。注意在使用Condition前必须获得锁，同时在Condition的等待队列上的结点与前面同步队列的结点是同一个类即Node，其结点的waitStatus的值为CONDITION。在实现类ConditionObject中有两个结点分别是firstWaiter和lastWaiter，firstWaiter代表等待队列第一个等待结点，lastWaiter代表等待队列最后一个等待结点，如下

 publicclass ConditionObject implements Condition, java.io.Serializable {
    _//等待队列第一个等待结点_
    privatetransient Node firstWaiter;
    _//等待队列最后一个等待结点_
    privatetransient Node lastWaiter;
    _//省略其他代码......._
}

每个Condition都对应着一个等待队列，也就是说如果一个锁上创建了多个Condition对象，那么也就存在多个等待队列。等待队列是一个FIFO的队列，在队列中每一个节点都包含了一个线程的引用，而该线程就是Condition对象上等待的线程。当一个线程调用了await()相关的方法，那么该线程将会释放锁，并构建一个Node节点封装当前线程的相关信息加入到等待队列中进行等待，直到被唤醒、中断、超时才从队列中移出。Condition中的等待队列模型如下

![image](/images/2019\06\java\1570382410523.jpg "image")

正如图所示，Node节点的数据结构，在等待队列中使用的变量与同步队列是不同的，Condtion中等待队列的结点只有直接指向的后继结点并没有指明前驱结点，而且使用的变量是nextWaiter而不是next，这点我们在前面分析结点Node的数据结构时讲过。firstWaiter指向等待队列的头结点，lastWaiter指向等待队列的尾结点，等待队列中结点的状态只有两种即CANCELLED和CONDITION，前者表示线程已结束需要从等待队列中移除，后者表示条件结点等待被唤醒。再次强调每个Codition对象对于一个等待队列，也就是说AQS中只能存在一个同步队列，但可拥有多个等待队列。下面从代码层面看看被调用await()方法(其他await()实现原理类似)的线程是如何加入等待队列的，而又是如何从等待队列中被唤醒的

publicfinalvoidawait() throws InterruptedException {
      _//判断线程是否被中断_
      if (Thread.interrupted())
          thrownew InterruptedException();
      _//创建新结点加入等待队列并返回_
      Node node = addConditionWaiter();
      _//释放当前线程锁即释放同步状态_
      int savedState = fullyRelease(node);
      int interruptMode = 0;
      _//判断结点是否同步队列(SyncQueue)中,即是否被唤醒_
      while (!isOnSyncQueue(node)) {
          _//挂起线程_
          LockSupport.park(this);
          _//判断是否被中断唤醒，如果是退出循环。_
          if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
              break;
      }
      _//被唤醒后执行自旋操作争取获得锁，同时判断线程是否被中断_
      if (acquireQueued(node, savedState) &amp;&amp; interruptMode != THROW\_IE)
          interruptMode = REINTERRUPT;
       _// clean up if cancelled_
      if (node.nextWaiter != null)
          _//清理等待队列中不为CONDITION状态的结点_
          unlinkCancelledWaiters();
      if (interruptMode != 0)
          reportInterruptAfterWait(interruptMode);
  }

执行addConditionWaiter()添加到等待队列。

 private Node addConditionWaiter() {
    Node t = lastWaiter;
      _// 判断是否为结束状态的结点并移除_
      if (t != null &amp;&amp; t.waitStatus != Node.CONDITION) {
          unlinkCancelledWaiters();
          t = lastWaiter;
      }
      _//创建新结点状态为CONDITION_
      Node node = new Node(Thread.currentThread(), Node.CONDITION);
      _//加入等待队列_
      if (t == null)
          firstWaiter = node;
      else
          t.nextWaiter = node;
      lastWaiter = node;
      return node;
        }

await()方法主要做了3件事，一是调用addConditionWaiter()方法将当前线程封装成node结点加入等待队列，二是调用fullyRelease(node)方法释放同步状态并唤醒后继结点的线程。三是调用isOnSyncQueue(node)方法判断结点是否在同步队列中，注意是个while循环，如果同步队列中没有该结点就直接挂起该线程，需要明白的是如果线程被唤醒后就调用acquireQueued(node, savedState)执行自旋操作争取锁，即当前线程结点从等待队列转移到同步队列并开始努力获取锁。

接着看看唤醒操作singal()方法

 publicfinalvoidsignal() {
     _//判断是否持有独占锁，如果不是抛出异常_
   if (!isHeldExclusively())
          thrownew IllegalMonitorStateException();
      Node first = firstWaiter;
      _//唤醒等待队列第一个结点的线程_
      if (first != null)
          doSignal(first);
 }

这里signal()方法做了两件事，一是判断当前线程是否持有独占锁，没有就抛出异常，从这点也可以看出只有独占模式先采用等待队列，而共享模式下是没有等待队列的，也就没法使用Condition。二是唤醒等待队列的第一个结点，即执行doSignal(first)

 privatevoiddoSignal(Node first) {
     do {
             _//移除条件等待队列中的第一个结点，_
             _//如果后继结点为null，那么说没有其他结点将尾结点也设置为null_
            if ( (firstWaiter = first.nextWaiter) == null)
                 lastWaiter = null;
             first.nextWaiter = null;
          _//如果被通知节点没有进入到同步队列并且条件等待队列还有不为空的节点，则继续循环通知后续结点_
         } while (!transferForSignal(first) &amp;&amp;
                  (first = firstWaiter) != null);
        }

_//transferForSignal方法_
finalboolean transferForSignal(Node node) {
    _//尝试设置唤醒结点的waitStatus为0，即初始化状态_
    _//如果设置失败，说明当期结点node的waitStatus已不为_
    _//CONDITION状态，那么只能是结束状态了，因此返回false_
    _//返回doSignal()方法中继续唤醒其他结点的线程，注意这里并_
    _//不涉及并发问题，所以CAS操作失败只可能是预期值不为CONDITION，_
    _//而不是多线程设置导致预期值变化，毕竟操作该方法的线程是持有锁的。_
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
         returnfalse;

        _//加入同步队列并返回前驱结点p_
        Node p = enq(node);
        int ws = p.waitStatus;
        _//判断前驱结点是否为结束结点(CANCELLED=1)或者在设置_
        _//前驱节点状态为Node.SIGNAL状态失败时，唤醒被通知节点代表的线程_
        if (ws \&gt; 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            _//唤醒node结点的线程_
            LockSupport.unpark(node.thread);
        returntrue;
    }

注释说得很明白了，这里我们简单整体说明一下，doSignal(first)方法中做了两件事，从条件等待队列移除被唤醒的节点，然后重新维护条件等待队列的firstWaiter和lastWaiter的指向。二是将从等待队列移除的结点加入同步队列(在transferForSignal()方法中完成的)，如果进入到同步队列失败并且条件等待队列还有不为空的节点，则继续循环唤醒后续其他结点的线程。到此整个signal()的唤醒过程就很清晰了，即signal()被调用后，先判断当前线程是否持有独占锁，如果有，那么唤醒当前Condition对象中等待队列的第一个结点的线程，并从等待队列中移除该结点，移动到同步队列中，如果加入同步队列失败，那么继续循环唤醒等待队列中的其他结点的线程，如果成功加入同步队列，那么如果其前驱结点是否已结束或者设置前驱节点状态为Node.SIGNAL状态失败，则通过LockSupport.unpark()唤醒被通知节点代表的线程，到此signal()任务完成，注意被唤醒后的线程，将从前面的await()方法中的while循环中退出，因为此时该线程的结点已在同步队列中，那么while (!isOnSyncQueue(node))将不在符合循环条件，进而调用AQS的acquireQueued()方法加入获取同步状态的竞争中，这就是等待唤醒机制的整个流程实现原理，流程如下图所示（注意无论是同步队列还是等待队列使用的Node数据结构都是同一个，不过是使用的内部变量不同罢了）

![image](/images/2019\06\java\1570382410526.jpg "image")

ok~，本篇先到这，关于AQS中的另一种模式即共享模式，下篇再详聊，欢迎继续关注。

主要参考资料

《Java并发编程的艺术》

《Java 编程思想》
