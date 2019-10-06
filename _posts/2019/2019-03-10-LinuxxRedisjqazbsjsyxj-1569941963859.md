---
layout: post
title:  "Linux下Redis集群安装部署及使用详解"
categories: redis
tags: 集群 安装
author: jiangc
excerpt: Linux下Redis集群安装部署及使用详解
---
* content
{:toc}

# 一、应用场景介绍

　　本文主要是介绍Redis集群在Linux环境下的安装讲解，其中主要包括在联网的Linux环境和脱机的Linux环境下是如何安装的。因为大多数时候，公司的生产环境是在内网环境下，无外网，服务器处于脱机状态（最近公司要上线项目，就是无外网环境的Linux，被离线安装坑惨了，走了很多弯路，说多了都是血泪史啊%\&gt;\_\&lt;%）。这也是笔者写本文的初衷，希望其他人少走弯路，下面就介绍如何在Linux安装部署Redis集群。

# 二、安装环境及工具

　　系统：Red Hat Enterprise Linux Server release 6.6

　　工具：XShell5及Xftp5

　　安装包：GCC-7.1.0

　　　　　　Ruby-2.4.1

　　　　　　Rubygems-2.6.12

　　　　　　Redis-3.2.9(3.x版本才开始支持集群功能)

# 三、安装步骤

　　要搭建一个最简单的Redis集群，我们至少需要6个节点：3个Master和3个Slave。那为什么需要3个Master呢？其实就是一个&quot;铁三角&quot;的关系，当1个Master下线的时候，其他2个Master和对应的Salve立马就能顶替上去，确保集群能够正常使用，如果你之前了解Mongodb/Hadoop/Strom这些的话，你就很容易目标一般分布式的最低要求基数个数节点，这样便于选举（少数服从多数的原则）。本文当中，我们就偷下懒，在一台Linux虚拟机上搭建6个节点的Redis集群(实际真正生产环境，需要3台Linux服务器分布存放3个Master)

**1、安装GCC环境**

安装Redis需要依托GCC环境，先检查Linux是否已经安装了GCC，如果没有安装，则需要进行安装

检查GCC是否安装，可以看看版本号

$ gcc -v

如果已经安装了GCC，则会显示以下信息

![image](/images/2019\03\redis\1569941963864.jpg "image")

如果没有任何信息，则我们可以通过命令yum install gcc-c++进行在线安装

$ yum install gcc-c++

如果没有网络的时候，我们就需要下载GCC的安装包进行手动安装了，具体方法还是比较复杂的，具体离线安装GCC的方法，请参考我的另外一篇文章《[Linux无网离线安装GCC](http://www.cnblogs.com/xuliangxing/p/7132018.html)》

**2、安装Ruby和Rubygems**

如果有网的话，则通过yum命令进行安装，自动将关联的依赖包全部安装

$ yum install ruby
$ yum install rubygems

如果是离线的状态，我们则可以选择下载Ruby和Rubygems，解压手动进行安装，具体的方法请参考我的另外两篇文件《[Linux 离线安装Ruby详解](http://www.cnblogs.com/xuliangxing/p/7132656.html)》和《[Linux 离线安装Rubygems详解](http://www.cnblogs.com/xuliangxing/p/7133544.html)》，这里我们不做多讲解。

# 四、安装Redis

1、到官网(https://redis.io/download)下载Redis，现在最新的版本为：3.2.9 ，将下载好的压缩包上传到服务器当中。如图所示，我是新建了一个Redis临时目录存放，偷懒我就用xftp5手动创建一个目录存放（也可以写命令创建文件夹 $ makdir redis）

![image](/images/2019\03\redis\1569941963886.jpg "image")

2、安装Redis

转到Redis的存放目录，然后通过命令解压Redis压缩包

$ cd /home/cmfchina/redis

$ tar -zxvf redis-3.2.9.tar.gz

![image](/images/2019\03\redis\1569941963894.jpg "image")

通过make命令进行安装Redis(需要root权限)

$ cd /home/cmfchina/redis/redis-3.2.9

$ make &amp;&amp; make install  //make 这里如果不指定PREFIX，默认将安装在/usr/local/bin下，保持默认就好

![image](/images/2019\03\redis\1569941963897.jpg "image")

如果没有root权限是无法安装的，如图所示

![image](/images/2019\03\redis\1569941963901.jpg "image")

我们获取root权限之后再进行安装，看到如下信息，说明Redis安装成功了，也可以到/usr/local/bin目录下看看

![image](/images/2019\03\redis\1569941963907.jpg "image")

![image](/images/2019\03\redis\1569941963911.jpg "image")

如果只是想要单机，不存在集群功能，我们现在就可以将Redis运行起来，我们直接在刚刚解压的Redis目录下运行命令就可以将单机的Redis 运行起来

![image](/images/2019\03\redis\1569941963915.jpg "image")

$ cd /home/cmfchina/redis/redis-3.2.9

$ redis-server redis.conf  //所有相关配置信息都在conf里面，如果不设置，默认端口号为：6379

![image](/images/2019\03\redis\1569941963919.jpg "image")

# 五、配置Redis集群

　　刚刚上面讲到如果只是想运行单机版的Redis(个人研究Redis可以安装单机版)，上面的讲解已经够了，不过现实当中，我们往往是需要使用到集群功能的，进行容错。

　　之前讲到是我们需要6个节点的Redis作为集群，所以我们需要创建6个文件夹，分别存放6个节点的配置信息，6个节点需要对应6个端口号，比如7001~7006，这个端口号我们自行定义，我们通过xftp5可视化创建一下。

![image](/images/2019\03\redis\1569941963922.jpg "image")

**第一步** 、我们也可以通过命令mkdir批量创建，，命令可能会更快点。

![image](/images/2019\03\redis\1569941963926.jpg "image")

下面重点来了，需要6个节点，所有我们需要配置各自的redis.conf配置文件。到我们Redis的安装目录usr/local/bin，将redis-cli、redis-server、redis.conf(没有conf文件，可以从压缩包里拷个出来，或者自己直接新建一个空的conf文件，后面再配置相关信息)，分别复制到刚刚创建的6个文件夹当中。

![image](/images/2019\03\redis\1569941963929.jpg "image")

**第二步** 、接下来，我们需要配置redis.conf文件，如果你是从压缩包拷贝出来，你会发现特别多的备注，这些是都是官网的备注讲解，你可以全部删除，只配置你想配置的信息就行。我们主要配置相对应的端口信息和集群配置信息

![image](/images/2019\03\redis\1569941963947.jpg "image")

还有很多redis.conf配置信息，实际场景我们再自行配置，关于配置redis.conf的相关信息，可以参考笔者另一篇文件《[Redis.conf及其Sentinel.conf配置项详细说明](http://www.cnblogs.com/xuliangxing/p/7149322.html)》。我们分别配置相对应的6个redis.conf信息。

![image](/images/2019\03\redis\1569941963953.jpg "image")

分别将这6个redis服务启动起来（命令redis-server redis.conf），一个一个去启动有点复杂，在redis目录创建一个sh脚本来启动6个实例

1 $cd /home/cmfchina/redis
2 $vim startall.sh 就会打开vim编辑器，创建一个空的文本

![image](/images/2019\03\redis\1569941963955.jpg "image")

:wq!保存脚本，创建成功：

![image](/images/2019\03\redis\1569941963958.jpg "image")

执行./startall.sh 提示permission denied说明权限不足，执行命令chmod 777 startall.sh修改权限获取root用户执行脚本

![image](/images/2019\03\redis\1569941963962.jpg "image")

$ chmod 777 startall.sh 分配权限

$ sh -x startall.sh 执行脚本

![image](/images/2019\03\redis\1569941963964.jpg "image")

**=======补充说明 ：2017-12-14 =========**

**有网友跟博主反应，上面这个批量脚本不能执行，是博主漏加了命令：kill -2**

**因为：每次执行一个redis启动，都会停留在redis的启动界面（上文说到的单机启动redis的那个界面），所以我们需要模拟退出当前界面，执行下一条命令**

**Kill -2 ：功能类似于Ctrl + C 是程序在结束之前，能够保存相关数据，然后再退出。**

**将上文的脚本改造下就行；如图所示**

![image](/images/2019\03\redis\1569941963967.jpg "image")

**Ps:如果出现路径找不到的问题，将上文的路径全部换成绝对路径**

**=================================**

再执行./startall.sh,  然后通过命令netstat -tnulp | grep redis和ps  aux | grep redis查看redis运行情况，可以看到端口7001、7002、7003、7004、7005、7006的redis都起来了。噢耶 ~~~~

![image](/images/2019\03\redis\1569941963970.jpg "image")

**第三步** 、实际上，Redis集群的操作在后文你可以看到是通过Ruby脚本来完成的，因此我们需要安装Ruby相关的RPM包，以及Redis和Ruby的接口包。我们要用到之前安装的Ruby。我们到之前解压的文件redis-3.2.9/src目录下找到文件为：redis-trib.rb，如图所示

![image](/images/2019\03\redis\1569941963978.jpg "image")

将该文件拷贝到与6个文件夹的同级目录下

![image](/images/2019\03\redis\1569941963981.jpg "image")

在redis目录下执行命令：

$ ./redis-trib.rb  create --replicas  1  127.0.0.1:7001  127.0.0.1:7002  127.0.0.1:7003  127.0.0.1:7004  127.0.0.1:7005  127.0.0.1:7006

**===============相关错误汇总解决方案（你以为上面是重点%\&gt;\_\&lt;%，其实下面这才是本文重点（太多坑）！！！）===============**

如果执行上述命令出现Ruby和Rubygems错误的话，那是没有安装Ruby和Rubygems，所有这就是为什么我们文章之前就要提前安装好Ruby和Rubygems。但是有些人说这两个我们已经安装了，为什么还会报如下错误的话

/home/cmfchina/ruby/lib/ruby/site\_ruby/2.4.0/rubygems/core\_ext/kernel\_require.rb:55:in `require&#39;: cannot load such file -- redis (LoadError)
from /home/cmfchina/ruby/lib/ruby/site\_ruby/2.4.0/rubygems/core\_ext/kernel\_require.rb:55:in `require&#39;
from ./redis-trib.rb:25:in `\&lt;main\&gt;&#39;

![image](/images/2019\03\redis\1569941963984.jpg "image")

这错误是提示不能加载redis，那是因为缺少redis和ruby的接口，使用gem 安装，我们这个时候其实还需要安装对应的Redis的Rbuy接口包。我们需要下载对应Redis的gem包安装才行。Rubygems的官网其实提供了Redis的gem包，我们可以直接取下载[https://rubygems.org/gems/redis/](https://rubygems.org/gems/redis/)   下载后上传到服务器当中

![image](/images/2019\03\redis\1569941964001.jpg "image")

执行gem install redis-3.3.0.gem命令安装。

$ gem install redis-3.3.0.gem

但是执行这个又报了错误，如果没有报错的话那就说明人品好啊......真是心塞~~~如图所示，这是因为需要依赖zlib工具。

ERROR:  Loading command: install (LoadError)
    cannot load such file -- zlib
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke\_with\_build\_args&#39; for nil:NilClass

![image](/images/2019\03\redis\1569941964005.jpg "image")



**第四步** 、我们需要再安装zlib才行，下载zlib，上传解压，安装zlib官方网站：[http://www.zlib.net](http://www.zlib.net/) ，最新版1.2.11，安装我们就一笔带过

1 $tar -xvzf zlib-1.2.11.tar.gz
2 $cd zlib-1.2.8.tar.gz
3 $./configure --prefix=/usr/local/zlib  设置安装路径
4 $make
5 $make instal

安装完zlib之后，我们再需要执行以下命令

1 $ cd /home/cmfchina/ruby/ruby-2.4.1/ext/zlib  备注：/home/cmfchina/ruby/ruby-2.4.1这个目录是ruby安装包后解压的目录，就是前面提到的ruby离线安装
2$ ruby extconf.rb
3 $ make &amp;&amp; make install

可是又报错了，真是无力吐槽了~~~错误信息如下

![image](/images/2019\03\redis\1569941964012.jpg "image")

checking for deflateReset() in -lz... no
checking for deflateReset() in -llibz... no
checking for deflateReset() in -lzlib1... no
checking for deflateReset() in -lzlib... no
checking for deflateReset() in -lzdll... no
checking for deflateReset() in -lzlibwapi... no
\*\*\* extconf.rb failed \*\*\*
Could not create Makefile due to some reason, probably lack of necessary
libraries and/or headers.  Check the mkmf.log filefor more details.  You may
need configuration options.

![image](/images/2019\03\redis\1569941964018.jpg "image")

![image](/images/2019\03\redis\1569941964022.jpg "image")

本着不放弃的原则，只能去外国网站查查资料看怎么解决了，发现原来是要将文件安装到本地运行库的里面才行，所有安装的时候需要额外配置信息，但是前提是我们需要安装zlib，才能继续下一步。

安装好zlib，然后我们重新输入命令

![image](/images/2019\03\redis\1569941964028.jpg "image")

1 $ cd /home/cmfchina/ruby/ruby-2.4.1/ext/zlib

备注：/home/cmfchina/ruby/ruby-2.4.1这个目录是ruby安装包后解压的目录，就是前面提到的ruby离线安装

2 $ ruby extconf.rb  --with-zlib-include=/usr/local/zlib/include/ --with-zlib-lib=/usr/local/zlib/lib   //会生成一个Makefile文件

备注：/usr/local/zlib是我的zlib安装目录

3 $ make &amp;&amp; make install

![image](/images/2019\03\redis\1569941964035.jpg "image")

这个时候会自动生成一个Makefile文件，如图所示

![image](/images/2019\03\redis\1569941964038.jpg "image")



接下来我们make &amp;&amp; make install 安装一下，但是当我们make的时候，又出现了错误如下

make: \*\*\* No rule to make target `/include/ruby.h&#39;, needed by `zlib.o&#39;.  Stop

![image](/images/2019\03\redis\1569941964041.jpg "image")

这个时候打开ext/zlib/Makefile文件，找到下面一行把路径进行修改一下。

zlib.o: $(top\_srcdir)/include/ruby.h 改成：zlib.o: ../../include/ruby.h

如图所示

![image](/images/2019\03\redis\1569941964044.jpg "image")

修改完成，然后保存：接着我们再make &amp;&amp; make install，这个时候安装成功了~~所以就放心地接着干吧！（没有看到我也无能为力了）安装完成后如下显示

![image](/images/2019\03\redis\1569941964048.jpg "image")



我们回到redis的gem目录下，继续执行命令：gem install redis-3.3.0.gem

但是......又出现了错误，原来我们还需要安装OpenSSL，因为Redis集群交互是需要OpenSSL

![image](/images/2019\03\redis\1569941964051.jpg "image")

**第五步** 、我们又得安装OpenSSL才行，官网地址：[https://www.openssl.org/source/](https://www.openssl.org/source/) 上次压缩包到服务器，解压，具体不做太细讲解

1 $ tar -xzvf openssl-1.0.2l.tar.gz
2 $ cd openssl-1.0.2l
3 $ ./config -fPIC --prefix=/usr/local/openssl enable-shared
4 $ ./config -t
5 $ make &amp;&amp; make install

安装openssl成功界面如下：

![image](/images/2019\03\redis\1569941964053.jpg "image")

我们又要到到Ruby解压的源码[/home/cmfchina/ruby-2.4.1]目录下的ext/openssl 目录，如图所示

![image](/images/2019\03\redis\1569941964056.jpg "image")

安装和zlib一样的方式安装openssl

![image](/images/2019\03\redis\1569941964059.jpg "image")

1 $ cd /home/cmfchina/ruby-2.4.1/ext/openssl
2备注：/home/cmfchina/ruby/ruby-2.4.1这个目录是ruby安装包后解压的目录，就是前面提到的ruby离线安装

3 $ruby extconf.rb  --with-openssl-include=/usr/local/openssl/include/ --with-openssl-lib=/usr/local/openssl/lib //会生成一个Makefile文件
4备注：/usr/local/openssl是我的openssl安装目录

5 $ make &amp;&amp; make install

![image](/images/2019\03\redis\1569941964062.jpg "image")

但是我们make的时候，又出现了和zlib类似的错误

make: \*\*\* No rule to make target `/include/ruby.h&#39;, needed by `ossl.o&#39;.  Stop

![image](/images/2019\03\redis\1569941964064.jpg "image")

还是按照刚刚zlib操作一样，打开Makefile文件，将$(top\_srcdir)全部改成../..

![image](/images/2019\03\redis\1569941964066.jpg "image")

修改后保存，再执行make &amp;&amp; make install，这一次安装成功了~~

![image](/images/2019\03\redis\1569941964070.jpg "image")

然后我们再回到之前redis目录下执行命令：gem install redis-3.3.0.gem

![image](/images/2019\03\redis\1569941964072.jpg "image")

**第六步** 、启动Redis集群

完成以上步骤之后，我们再回到第三步执行命令

在redis目录下执行命令：

$ ./redis-trib.rb  create --replicas  1  127.0.0.1:7001  127.0.0.1:7002  127.0.0.1:7003  127.0.0.1:7004  127.0.0.1:7005  127.0.0.1:7006

![image](/images/2019\03\redis\1569941964076.jpg "image")

我们选择yes，意思是服从这种主从分配方式，我们也可以通过配置文件自己指定slave

![image](/images/2019\03\redis\1569941964080.jpg "image")

# 第六、Redis集群测试

我们来测试一下Redis集群，通过连接任一redis端口，添加数据

[[email protected] redis7001]# redis-cli -p 7001 -c

[[email protected] redis7001]# redis-cli -c -h 127.0.0.1 -p 7001 shutdown //关闭集群，如果没有-h参数，默认连接127.0.0.1，如果没有-p参数，默认连接6370端口（所有如果用默认的，就没有-h -p）

说明：-h+host –p+端口号 –c 是要连接集群，注意坑，不加会报错的

![image](/images/2019\03\redis\1569941964087.jpg "image")

可以看到连接的是7001的节点，set name的时候计算了存在哪个hash槽上，会跳转到那个槽对应的节点

结束语：至此，Redis的集群配置的前世今生已到此结束，在没有网络的环境下中途碰到了很多坑，现在我们可以尽情享受Redis，纵使虐我千百遍，我待它如初恋

后话：推荐一个Redis的可视化工具：RedisDesktopManager 官网（[https://redisdesktop.com/download](https://redisdesktop.com/download)），具体可以到官网看看，这里我只抛砖引玉一下，下篇文章主要介绍Redis的Sentinel(哨兵)模式
