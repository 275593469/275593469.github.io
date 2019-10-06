---
layout: post
title:  "fail-fast总结(通过ArrayList来说明fail-fast的原理、解决办法)"
categories: java
tags: 集合 fail-fast
author: jiangc
excerpt: fail-fast总结(通过ArrayList来说明fail-fast的原理、解决办法)
---
* content
{:toc}

概要

前面，我们已经学习了ArrayList。接下来，我们以ArrayList为例，对Iterator的fail-fast机制进行了解。

**1 fail-fast简介**

**fail-fast 机制是java集合(Collection)中的一种错误机制。**当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。

例如：当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A访问集合时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

在详细介绍fail-fast机制的原理之前，先通过一个示例来认识fail-fast。

**2 fail-fast示例**

示例代码：(FastFailTest.java)

import java.util.\*;
import java.util.concurrent.\*;
_/\*_
 _\* @desc java集合中Fast-Fail的测试程序。_
 _\*_
 _\*   fast-fail事件产生的条件：当多个线程对Collection进行操作时，若其中某一个线程通过iterator去遍历集合时，该集合的内容被其他线程所改变；则会抛出ConcurrentModificationException异常。_
 _\*   fast-fail解决办法：通过util.concurrent集合包下的相应类去处理，则不会产生fast-fail事件。_
 _\*_
 _\*   本例中，分别测试ArrayList和CopyOnWriteArrayList这两种情况。ArrayList会产生fast-fail事件，而CopyOnWriteArrayList不会产生fast-fail事件。_
 _\*   (01) 使用ArrayList时，会产生fast-fail事件，抛出ConcurrentModificationException异常；定义如下：_
 _\*            private static List\&lt;String\&gt; list = new ArrayList\&lt;String\&gt;();_
 _\*   (02) 使用时CopyOnWriteArrayList，不会产生fast-fail事件；定义如下：_
 _\*            private static List\&lt;String\&gt; list = new CopyOnWriteArrayList\&lt;String\&gt;();_
 _\*_
 _\* @author skywang_
 _\*/_
publicclass FastFailTest {
    privatestatic List\&lt;String\&gt; list = new ArrayList\&lt;String\&gt;();
    _//private static List\&lt;String\&gt; list = new CopyOnWriteArrayList\&lt;String\&gt;();_
    publicstaticvoid main(String[] args) {
        _// 同时启动两个线程对list进行操作！_
        new ThreadOne().start();
        new ThreadTwo().start();
    }
    privatestaticvoid printAll() {
        System.out.println(&quot;&quot;);
        String value = null;
        Iterator iter = list.iterator();
        while(iter.hasNext()) {
            value = (String)iter.next();
            System.out.print(value+&quot;, &quot;);
        }
    }
    _/\*\*_
 _    \* 向list中依次添加0,1,2,3,4,5，每添加一个数之后，就通过printAll()遍历整个list_
 _    \*/_
    privatestaticclass ThreadOne extends Thread {
        publicvoid run() {
            int i = 0;
            while (i\&lt;6) {
                list.add(String.valueOf(i));
                printAll();
                i++;
            }
        }
    }
    _/\*\*_
 _    \* 向list中依次添加10,11,12,13,14,15，每添加一个数之后，就通过printAll()遍历整个list_
 _    \*/_
    privatestaticclass ThreadTwo extends Thread {
        publicvoid run() {
            int i = 10;
            while (i\&lt;16) {
                list.add(String.valueOf(i));
                printAll();
                i++;
            }
        }
    }
}

**运行结果：**

运行该代码，抛出异常java.util.ConcurrentModificationException！即，产生fail-fast事件！

**结果说明：**

(01) FastFailTest中通过 new ThreadOne().start() 和 new ThreadTwo().start() 同时启动两个线程去操作list。

    **ThreadOne线程：** 向list中依次添加0,1,2,3,4,5。每添加一个数之后，就通过printAll()遍历整个list。

    **ThreadTwo线程：** 向list中依次添加10,11,12,13,14,15。每添加一个数之后，就通过printAll()遍历整个list。

(02) 当某一个线程遍历list的过程中，list的内容被另外一个线程所改变了；就会抛出ConcurrentModificationException异常，产生fail-fast事件。

**3 fail-fast解决办法**

fail-fast机制，是一种错误检测机制。 **它只能被用来检测错误，因为JDK并不保证fail-fast机制一定会发生。** 若在多线程环境下使用fail-fast机制的集合，建议使用&quot;java.util.concurrent包下的类&quot;去取代&quot;java.util包下的类&quot;。

所以，本例中只需要将ArrayList替换成java.util.concurrent包下对应的类即可。

即，将代码

privatestaticList\&lt;String\&gt; list = new ArrayList\&lt;String\&gt;();

替换为

privatestaticList\&lt;String\&gt; list = new CopyOnWriteArrayList\&lt;String\&gt;();

则可以解决该办法。

**4 fail-fast原理**

产生fail-fast事件，是通过抛出ConcurrentModificationException异常来触发的。

那么，ArrayList是如何抛出ConcurrentModificationException异常的呢?

我们知道，ConcurrentModificationException是在操作Iterator时抛出的异常。我们先看看Iterator的源码。ArrayList的Iterator是在父类AbstractList.java中实现的。代码如下：

publicabstractclassAbstractList\&lt;E\&gt; extendsAbstractCollection\&lt;E\&gt; implementsList\&lt;E\&gt; {
    ...
    _// AbstractList中唯一的属性_
    _// 用来记录List修改的次数：每修改一次(添加/删除等操作)，将modCount+1_
    protectedtransientint modCount = 0;
    _// 返回List对应迭代器。实际上，是返回Itr对象。_
    public Iterator\&lt;E\&gt; iterator() {
        returnnew Itr();
    }
    _// Itr是Iterator(迭代器)的实现类_
    privateclassItrimplementsIterator\&lt;E\&gt; {
        int cursor = 0;
        int lastRet = -1;
        _// 修改数的记录值。_
        _// 每次新建Itr()对象时，都会保存新建该对象时对应的modCount；_
        _// 以后每次遍历List中的元素的时候，都会比较expectedModCount和modCount是否相等；_
        _// 若不相等，则抛出ConcurrentModificationException异常，产生fail-fast事件。_
        int expectedModCount = modCount;
        publicboolean hasNext() {
            return cursor != size();
        }
        public E next() {
            _// 获取下一个元素之前，都会判断&quot;新建Itr对象时保存的modCount&quot;和&quot;当前的modCount&quot;是否相等；_
            _// 若不相等，则抛出ConcurrentModificationException异常，产生fail-fast事件。_
            checkForComodification();
            try {
                E next = get(cursor);
                lastRet = cursor++;
                return next;
            } catch (IndexOutOfBoundsException e) {
                checkForComodification();
                thrownew NoSuchElementException();
            }
        }
        publicvoid remove() {
            if (lastRet == -1)
                thrownew IllegalStateException();
            checkForComodification();
            try {
                AbstractList.this.remove(lastRet);
                if (lastRet \&lt; cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException e) {
                thrownew ConcurrentModificationException();
            }
        }
        finalvoid checkForComodification() {
            if (modCount != expectedModCount)
                thrownew ConcurrentModificationException();
        }
    }
    ...
}

从中，我们可以发现在调用 next() 和 remove()时，都会执行 checkForComodification()。若 &quot; **modCount 不等于 expectedModCount**&quot;，则抛出ConcurrentModificationException异常，产生fail-fast事件。

要搞明白 fail-fast机制，我们就要需要理解什么时候&quot;modCount 不等于 expectedModCount&quot;！

从Itr类中，我们知道 expectedModCount 在创建Itr对象时，被赋值为 modCount。通过Itr，我们知道：expectedModCount不可能被修改为不等于 modCount。所以，需要考证的就是modCount何时会被修改。

接下来，我们查看ArrayList的源码，来看看modCount是如何被修改的。

package java.util;
publicclassArrayList\&lt;E\&gt; extendsAbstractList\&lt;E\&gt;
        implementsList\&lt;E\&gt;, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    _// list中容量变化时，对应的同步函数_
    publicvoid ensureCapacity(int minCapacity) {
        modCount++;
        int oldCapacity = elementData.length;
        if (minCapacity \&gt; oldCapacity) {
            Object oldData[] = elementData;
            int newCapacity = (oldCapacity \* 3)/2 + 1;
            if (newCapacity \&lt; minCapacity)
                newCapacity = minCapacity;
            _// minCapacity is usually close to size, so this is a win:_
            elementData = Arrays.copyOf(elementData, newCapacity);
        }
    }
    _// 添加元素到队列最后_
    publicboolean add(E e) {
        _// 修改modCount_
        ensureCapacity(size + 1);  _// Increments modCount!!_
        elementData[size++] = e;
        returntrue;
    }
    _// 添加元素到指定的位置_
    publicvoid add(int index, E element) {
        if (index \&gt; size || index \&lt; 0)
            thrownew IndexOutOfBoundsException(
            &quot;Index: &quot;+index+&quot;, Size: &quot;+size);
        _// 修改modCount_
        ensureCapacity(size+1);  _// Increments modCount!!_
        System.arraycopy(elementData, index, elementData, index + 1,
             size - index);
        elementData[index] = element;
        size++;
    }
    _// 添加集合_
    publicboolean addAll(Collection\&lt;? extends E\&gt; c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        _// 修改modCount_
        ensureCapacity(size + numNew);  _// Increments modCount_
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
    _// 删除指定位置的元素_
    public E remove(int index) {
        RangeCheck(index);
        _// 修改modCount_
        modCount++;
        E oldValue = (E) elementData[index];
        int numMoved = size - index - 1;
        if (numMoved \&gt; 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null; _// Let gc do its work_
        return oldValue;
    }
    _// 快速删除指定位置的元素_
    privatevoid fastRemove(int index) {
        _// 修改modCount_
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved \&gt; 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; _// Let gc do its work_
    }
    _// 清空集合_
    publicvoid clear() {
        _// 修改modCount_
        modCount++;
        _// Let gc do its work_
        for (int i = 0; i \&lt; size; i++)
            elementData[i] = null;
        size = 0;
    }
    ...
}

从中，我们发现：无论是add()、remove()，还是clear()，只要涉及到修改集合中的元素个数时，都会改变modCount的值。

接下来，我们再系统的梳理一下fail-fast是怎么产生的。步骤如下：

(01) 新建了一个ArrayList，名称为arrayList。

(02) 向arrayList中添加内容。

(03) 新建一个&quot; **线程a**&quot;，并在&quot;线程a&quot;中 **通过Iterator反复的读取arrayList的值** 。

(04) 新建一个&quot; **线程b**&quot;，在&quot;线程b&quot;中 **删除arrayList中的一个&quot;节点A&quot;** 。

(05) 这时，就会产生有趣的事件了。

      在某一时刻，&quot;线程a&quot;创建了arrayList的Iterator。此时&quot;节点A&quot;仍然存在于arrayList中，**创建arrayList时，expectedModCount = modCount(假设它们此时的值为N)**。

      在&quot;线程a&quot;在遍历arrayList过程中的某一时刻，&quot;线程b&quot;执行了，并且&quot;线程b&quot;删除了arrayList中的&quot;节点A&quot;。&quot;线程b&quot;执行remove()进行删除操作时，在remove()中执行了&quot;modCount++&quot;，此时 **modCount变成了N+1** ！

&quot;线程a&quot;接着遍历，当它执行到next()函数时，调用checkForComodification()比较&quot;expectedModCount&quot;和&quot;modCount&quot;的大小；而&quot;expectedModCount=N&quot;，&quot;modCount=N+1&quot;,这样，便抛出ConcurrentModificationException异常，产生fail-fast事件。

至此， **我们就完全了解了fail-fast是如何产生的** ！

即，当多个线程对同一个集合进行操作的时候，某线程访问集合的过程中，该集合的内容被其他线程所改变(即其它线程通过add、remove、clear等方法，改变了modCount的值)；这时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

**5 解决fail-fast的原理**

上面，说明了&quot;解决fail-fast机制的办法&quot;，也知道了&quot;fail-fast产生的根本原因&quot;。接下来，我们再进一步谈谈java.util.concurrent包中是如何解决fail-fast事件的。

还是以和ArrayList对应的CopyOnWriteArrayList进行说明。我们先看看CopyOnWriteArrayList的源码：

package java.util.concurrent;
import java.util.\*;
import java.util.concurrent.locks.\*;
import sun.misc.Unsafe;
publicclassCopyOnWriteArrayList\&lt;E\&gt;
    implementsList\&lt;E\&gt;, RandomAccess, Cloneable, java.io.Serializable{
    ...
    _// 返回集合对应的迭代器_
    public Iterator\&lt;E\&gt; iterator() {
        returnnew COWIterator\&lt;E\&gt;(getArray(), 0);
    }
    ...
    privatestaticclassCOWIterator\&lt;E\&gt; implementsListIterator\&lt;E\&gt; {
        privatefinal Object[] snapshot;
        privateint cursor;
        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            _// 新建COWIterator时，将集合中的元素保存到一个新的拷贝数组中。_
            _// 这样，当原始集合的数据改变，拷贝数据中的值也不会变化。_
            snapshot = elements;
        }
        publicboolean hasNext() {
            return cursor \&lt; snapshot.length;
        }
        publicboolean hasPrevious() {
            return cursor \&gt; 0;
        }
        public E next() {
            if (! hasNext())
                thrownew NoSuchElementException();
            return (E) snapshot[cursor++];
        }
        public E previous() {
            if (! hasPrevious())
                thrownew NoSuchElementException();
            return (E) snapshot[--cursor];
        }
        publicint nextIndex() {
            return cursor;
        }
        publicint previousIndex() {
            return cursor-1;
        }
        publicvoid remove() {
            thrownew UnsupportedOperationException();
        }
        publicvoid set(E e) {
            thrownew UnsupportedOperationException();
        }
        publicvoid add(E e) {
            thrownew UnsupportedOperationException();
        }
    }
    ...
}

从中，我们可以看出:

(01) 和ArrayList继承于AbstractList不同，CopyOnWriteArrayList没有继承于AbstractList，它仅仅只是实现了List接口。

(02) ArrayList的iterator()函数返回的Iterator是在AbstractList中实现的；而CopyOnWriteArrayList是自己实现Iterator。

(03) ArrayList的Iterator实现类中调用next()时，会&quot;调用checkForComodification()比较&#39;expectedModCount&#39;和&#39;modCount&#39;的大小&quot;；但是，CopyOnWriteArrayList的Iterator实现类中，没有所谓的checkForComodification()，更不会抛出ConcurrentModificationException异常！

出处：http://www.cnblogs.com/skywang12345/p/3308762.html
