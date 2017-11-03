..  Copyright (C), 2014-2016, HAOHAN DATA Technology Co., Ltd.
    All rights reserved.

    @author liuyu@haohandata.com.cn
    @date 2017.06.19


redis性能测试
==============

1 测试说明
^^^^^^^^^^^^
1.1 测试环境
----------------
1. 使用工控机10.10.22.114，  Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
#. Redis版本3.2.3
#. 使用默认配置做测试
#. 使用redis自带的测试工具，以及自己写的小程序进行测试。
#. 测试用例层层深入，请按顺序阅读。

1.2 测试用例
-------------
1.2.1 验证redis支持的key长度，保证以URL、HOST为key可行
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

【测试说明】

1. 在http协议中并没有对url长度作出限制，往往url的最大长度和用户浏览器和Web服务器有关。以IE浏览器为例，URL的最大限制为2083个字符，如果超过这个数字，提交按钮没有任何反应。
#. 未找到文档里对key名的长度的限制说明，只说明了key值的大小最大能存储512MB。这里不对key值做验证，只对key名的长度做验证。

【测试结果】

    使用redis-cli命令行进行测试，由于该工具的输入限制，提供输入的最长命令的字符为4095，除去命令的其它关键字，这里使用4086个字符作为key名，测试结果可以正常设置、读取。
    
.. note:: 使用C语言接口是否能设置、读取更长的key名未做测试，因为4086已经够用了。该值超过常用URL的长度，因此认为Key名长度能够满足需求。

1.2.2 内存占用测试
>>>>>>>>>>>>>>>>>>>
【测试说明】

    首先使用自己写的小程序插入数据，再用redis-cli查看内存占用情况。这里使用“集合”数据结构，因为在审计功能中比较可能用到它（一个key对应多个无序value，也就是一个URL对应多个关键字或策略ID）。

    测试所用数据：key个数200w，每个key名长度长度200字节左右，每个键下挂载20个value，每个value占用64个字节。

【测试结果】

* 插入数据前：793.20K
    
* 插入数据后：901.78M

1.2.3 使用redis自带测试程序测试性能
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

【测试说明】

1. 使用redis-benchmark命令进行性能测试，测试参数包括：请求数、并发数、数据大小、使用TCP或Unix socket、管道技术等。
#. 以下为redis-benchmark测试用到的参数说明

-n  指定请求数
-d  以字节的形式指定 SET/GET 值的数据大小
-k  0表示每次请求重新连接，1表示保持持续连接
-r  SET/GET/INCR 使用随机 key, SADD 使用随机值
-c  指定并发连接数
-s  指定服务器 socket
-t  仅运行以逗号分隔的测试命令列表
-P  通过管道传输 <numreq> 请求

【测试结果】

1. 使用TCP或Unix socket模式，测试所用数据：key个数200w，value大小20，并发20，持久连接（连接一次发送所有数据），测试所有支持数据类型的常用操作。

.. note:: 通过查看源码，这里的并发并不是多线程或多进程并行操作，实际上是通过在一个进程里创建多个socket异步发送、读取数据。

测试命令：

* TCP：redis-benchmark -n 2000000 -d 20 -c 20 -k 1
* Unix socket：redis-benchmark -n 2000000 -d 20 -c 20 -k 1 -s /tmp/redis.sock

通过下表对比，在单机上使用unix socket的读写速度明显高于TCP。其中比较关心的“集合”数据结构的操作：SADD（插入）在unix socket，并发20的情况下达到了24.4万每秒，SPOP（返回一个数据并删除）达到了24.1万每秒。

.. note:: 该命令未提供“SMEMBER（返回集合所有数据）”的测试数据，但是在下文会提供c代码的测试情况。

=============== =============================  ==============================
指令            TCP                            Unix socket  
=============== =============================  ==============================
PING_INLINE     completed in 13.64 seconds     completed in 10.63 seconds
                146616.81 requests per second  188182.16 requests per second
PING_BULK       completed in 13.82 seconds     completed in 10.81 seconds
                144728.27 requests per second  184928.34 requests per second
SET             completed in 13.69 seconds     completed in 10.81 seconds
                146081.38 requests per second  184928.34 requests per second
GET             completed in 13.77 seconds     completed in 10.77 seconds
                145264.39 requests per second  185787.27 requests per second
INCR	        completed in 13.68 seconds     completed in 10.65 seconds
                146230.89 requests per second  187775.80 requests per second
LPUSH	        completed in 13.51 seconds     completed in 10.49 seconds
                148093.30 requests per second  190712.30 requests per second
RPUSH	        completed in 13.50 seconds     completed in 10.44 seconds
                148192.05 requests per second  191497.50 requests per second
LPOP	        completed in 13.58 seconds     completed in 8.08 seconds
                147253.72 requests per second  247524.75 requests per second
RPOP	        completed in 13.87 seconds     completed in 8.15 seconds
                144216.91 requests per second  245549.41 requests per second
SADD	        completed in 13.63 seconds     completed in 8.18 seconds
                146713.61 requests per second  244439.00 requests per second
SPOP	        completed in 13.57 seconds     completed in 8.29 seconds
                147427.39 requests per second  241283.64 requests per second
LPUSH	        completed in 13.62 seconds     completed in 7.93 seconds
                146853.66 requests per second  252175.00 requests per second
LRANGE_100      completed in 37.52 seconds     completed in 31.94 seconds
                53303.48 requests per second   62611.52 requests per second
LRANGE_300      completed in 110.12 seconds    completed in 104.21 seconds
                18162.17 requests per second   19191.83 requests per second
LRANGE_500      completed in 171.87 seconds    completed in 165.77 seconds
                11637.04 requests per second   12064.76 requests per second
LRANGE_600      completed in 226.59 seconds    completed in 221.31 seconds
                8826.67 requests per second    9037.06 requests per second
MSET (10 keys)  completed in 17.73 seconds     completed in 221.31 seconds
                112784.08 requests per second  9037.06 requests per second
=============== =============================  ==============================


2. 同样是200w数据，但是将请求并发数降为1的情况下再进行测试。（这里只测试SADD和SPOP操作）

测试命令：

* TCP：redis-benchmark -n 2000000 -d 20 -c 1 -k 1 -t SADD,SPOP
* Unix socket：redis-benchmark -n 2000000 -d 20 -c 1 -k 1 -s /tmp/redis.sock -t SADD,SPOP

做这个测试是因为看源码所谓的并发不是非多线程/多进程操作，只是多连接异步读写。因此想看下在单连接的情况下的读写性能。通过下表看出，单连接的读写速度明显降低。其中Unix socket性能下降了68%。

=============== =============================  ==============================
指令            Unix socket（20并发）          Unix socket（1并发）  
=============== =============================  ==============================
SADD	        completed in 8.18 seconds      completed in 26.03 seconds
                244439.00 requests per second  76846.23 requests per second	
SPOP	        completed in 8.29 seconds      completed in 25.32 seconds
                241283.64 requests per second  78985.83 requests per second
=============== =============================  ==============================

3. 同样是200w数据，还是20并发，但是使用管道技术，即每次请求处理多条命令。这里只测试SADD和SPOP操作，每次请求处理10条命令，通过下表发现，使用管道技术后SADD达到了103.8万每秒，性能提升约425%，SPOP达到了134.1万每秒，提升了555%。

测试命令：

* redis-benchmark -n 2000000 -d 20 -c 20 -k 1 -s /tmp/redis.sock -t sadd,spop -P 10

======== =============================  ===============================
指令     Unix socket（普通20并发）	    Unix socket（20并发+管道10命令）
======== =============================  ===============================
SADD	 completed in 8.18 seconds      completed in 1.92 seconds
         244439.00 requests per second  1038961.06 requests per second
SPOP	 completed in 8.29 seconds      completed in 1.49 seconds
         241283.64 requests per second  1341381.62 requests per second
======== =============================  ===============================

1.2.4 写测试小程序测试性能
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
【测试说明】

1. 这里只测试Unix socket。
#. 使用o3编译，测试前先清空redis数据。
#. 使用阻塞/非阻塞（Non-Block socket）连接、调用原生API/改进后的API、并发1/并发20的情况进行对比。

.. note:: redis自带测试程序使用的是非阻塞连接，异步处理。

【测试结果】

1. 并发1，阻塞连接/非阻塞、调用原生API。
为了与redis自带测试程序进行对比，这里测试所用数据相同，200w个key，每个key对应1个value，数据大小20，持久连接。对于阻塞和非阻塞的区别，主要在于非阻塞连接在执行完命令后立刻返回，不等待结果命令的返回结果。通过以下测试数据可以看出：

首先，对于set/sadd设置类命令，非阻塞速度提升了约210%

其次，小程序未对get、smembers、spop这类需要返回结果的命令做异步读取处理，表中XXX部分只计算了执行完命令的时间，因此不做参考。

========== ============================================  =============================================
指令       阻塞连接	                                     非阻塞
========== ============================================  =============================================
SET        35.38 seconds, 56529.11 requests per second	 11.39 seconds, 175654.31 requests per second
GET        30.71 seconds, 65116.88 requests per second	 XXX
SADD       35.67 seconds, 56071.10 requests per second	 11.12 seconds, 179920.83 requests per second
SMEMBERS   35.30 seconds, 56662.04 requests per second	 XXX
SPOP       34.32 seconds, 58281.85 requests per second	 XXX
========== ============================================  =============================================

2. 并发1，阻塞连接，原生API/改进的API。
上一个测试例中已经说明了，阻塞与非阻塞主要区别在于是否等待返回，对于“读取”类的API在应用中是需要等待返回的，而“设置”类API可以不需要等待返回。因此对原生API做了些改进，通过对.h文件里提供的其他API重新组合封装成“改进API”，使得“设置”类的API即使在建立阻塞连接时，也仅发送不等待返回。
在建立阻塞连接时，测试结果如下：

=============== ============================================== ===========================================================
指令            原生API	                                        改进API
=============== ============================================== ===========================================================
SET             35.38 seconds, 56529.11 requests per second     11.41 seconds, 175346.31 requests per second
SADD            35.67 seconds, 56071.10 requests per second     11.02 seconds, 181504.67 requests per second
=============== ============================================== ===========================================================

3. 接下来测试20个线程并发的情况，每个线程绑定一个core，建立阻塞连接，“读取”类使用原生API，“写入”类使用改进API。测试结果如下：

============== ========================================================================
指令           测试结果
============== ========================================================================
SET	           completed 2000000 times in 6.01 seconds，332612.67 requests per second
GET	           completed 2000000 times in 10.50 seconds，190421.78 requests per second
SADD	       completed 2000000 times in 6.71 seconds，298195.91 requests per second
SMEMBERS	   completed 2000000 times in 14.22 seconds，140607.42 requests per second
SPOP	       completed 2000000 times in 14.38 seconds，139082.06 requests per second
============== ========================================================================

4. 在以上测试基础上，使用管道（不确定在审计里是否能用得到），缓存10条命令再发送。
通过下表看出，性能有所提升，约10%。与redis自带测试程序的555%提升相差甚远。原因暂时不明，因为redis自带测试程序是异步处理？还是因为我调用方式不对？后续若有用到管道技术，可以再补充测试一下。

=============== =========================================  ==========================================
指令            一次发一条命令                             一次发10条命令
=============== =========================================  ==========================================
SET             completed 2000000 times in 6.01 seconds    completed 2000000 times in 5.44 seconds
                332612.67 requests per second	           367917.59 requests
SADD	        completed 2000000 times in 6.71 seconds    completed 2000000 times in 6.14 seconds,
                298195.91 requests per second	           325892.13 requests per second
=============== =========================================  ==========================================

1.3.	测试总结
-----------------
1. key长度能够满足需求

2. 内存使用情况：在“key个数200w，每个key名长度长度200字节左右，每个键下挂载20个value，每个value占用64个字节”的测试条件下，占用约900M内存。

3. 使用redis自带测试程序测试性能：

    * Unix socket比TCP快，并发（多client异步通信，非多线程/多进程）比单client快。
    * 在20个client，unix socket，“集合数据结构”命令SADD（插入）达到 24.4万每秒 ，SPOP（取出并删除一条数据）达到24.1万每秒。
    * 在使用管道（一次处理10条命令），SADD达到了 103万每秒  SPOP 134万每秒。
    
4. 自己写小程序验证，在unix scoket、20线程绑定不同core下的并发：

    * 使用改进API ，SADD （插入）达到了29.8万每秒，SMEMBERS（读取key所有集合成员）达到了14万每秒，SPOP（读取并删除一条数据）13.9万。
    * 在使用管道（每次发送10条数据），性能无显提升（提升10%），与redis自带测试程序的555%提升相差甚远，原因暂时不明。可能与自带测试程序是异步处理有关，也有可能是因为我调用方式的不对。若有用到管道技术再补充测试。


