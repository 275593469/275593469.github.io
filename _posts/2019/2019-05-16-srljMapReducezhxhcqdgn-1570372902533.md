---
layout: post
title:  "深入理解MapReduce中环形缓冲区的概念"
categories: 大数据
tags: mapreduce
author: jiangc
excerpt: 深入理解MapReduce中环形缓冲区的概念
---
* content
{:toc}

MapReduce过程图解：

![image](/images/2019\05\dsj\1570372902534.jpg "image")

标题

环形缓冲区图示：

![image](/images/2019\05\dsj\1570372902541.jpg "image")

环形缓冲区

            Map的输出结果是由collector处理的，每个Map任务不断地将键值对输出到在内存中构造的一个环形数据结构中。使用环形数据结构是为了更有效地使用内存空间，在内存中放置尽可能多的数据。

            这个数据结构其实就是个字节数组byte[]，叫Kvbuffer，名如其义，但是这里面不光放置了数据，还放置了一些索引数据，给放置索引数据的区域起了一个Kvmeta的别名，在Kvbuffer的一块区域上穿了一个IntBuffer（字节序采用的是平台自身的字节序）

            的马甲。数据区域和索引数据区域在Kvbuffer中是相邻不重叠的两个区域，用一个分界点来划分两者，分界点不是亘古不变的，而是每次Spill之后都会更新一次。初始的分界点是0，数据的存储方向是向上增长，索引数据的存储方向是向下增长.

            注意：上述的分界点就可以理解图中的赤道信息

            Kvbuffer的存放指针bufindex（即数据的存储方向）是一直闷着头地向上增长，比如bufindex初始值为0，一个Int型的key写完之后，bufindex增长为4，一个Int型的value写完之后，bufindex增长为8。 （int型的数据占有4个字节）

            索引是对在kvbuffer中的键值对的索引，是个四元组，包括：value的起始位置、key的起始位置、partition值、value的长度，占用四个Int长度，Kvmeta的存放指针Kvindex每次都是向下跳四个&quot;格子&quot;，然后再向上一个格子一个格子地填充四元组

            的数据。比如Kvindex初始位置是-4，当第一个键值对写完之后，(Kvindex+0)的位置存放value的起始位置、(Kvindex+1)的位置存放key的起始位置、(Kvindex+2)的位置存放partition的值、(Kvindex+3)的位置存放value的长度，然后Kvindex跳到

            -8位置，等第二个键值对和索引写完之后，Kvindex跳到-12位置。

【注意】[https://blog.csdn.net/HADOOP\_83425744/article/details/49560583 参见该材料，了解环形缓冲区具体内容！！！](https://blog.csdn.net/HADOOP_83425744/article/details/49560583)
