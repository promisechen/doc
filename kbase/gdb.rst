..  Copyright (C), 2014-2016, HAOHAN DATA Technology Co., Ltd.
    All rights reserved.

    @author clx@haohandata.com.cn
    @date 2016.06.27


gdb速查手册
============

常用命令
---------

常见汇编指令
^^^^^^^^^^^^

* ``set disassembly-flavor intel/att``  将汇编安装intel或者att格式显示

* ``set disassemble-next-line on``  按照汇编显示


调试技巧
---------

图形界面使用
^^^^^^^^^^^^^

* ctrl+x+a 调出图形界面
* layout asm/src/split/reg 分别调出相关界面，详细help layout
* focus asm/src/split/cmd 分别将焦点定位到命令行


内存断点
^^^^^^^^^^^^^

``watch *(unsigned long*)0x7fffffffe3f0 if *(unsigned long*)0x7fffffffe3f0 > 0xffffffff``

当0x7fffffffe3f0 地址按照unsigned long取值，被写入的数据大于0xffffffff时，被断下.

经常遇到因为内存而导致段错误，但是根本原因可能不在当前现场。命案现场已经被破坏了，\
那么我们怎么能获取第一现场呢？ 所以我们经常需要用到内存断点，并且是需要下有条件的内存断点。

启动自加载断点
^^^^^^^^^^^^^^^

GDB使用中比较麻烦的事情，就是每次启动，还要手动敲一把命令，特别是断点比较多的情况，这个特便影响，工作效率。

查了一下gdb info，gdb支持自动读取一个启动脚本文件.gdbinit，所以经常\
输入的启动命令， 就都可以写在gdb启动目录的.gdbinit里面。比如

.gdbinit::

    file myapp 
    handle SIGPIPE nostop 
    break ss.c:100 
    break ss.c:200 
    run


GDB和bash类似，也支持source这个命令，执行另外一个脚本文件。所以可以修改一下.

.gdbinit::
 
    file myapp 
    handle SIGPIPE nostop 
    source gdb.break 
    run 
    
    gdb.break: 
    break ss.c:100 
    break ss.c:200

这样修改的断点配置，只需要编辑gdb.break就可以了。

偶而还是需要单独启动GDB，不想执行自动脚本，于是又改进了一下。首先把.gdbinit命名为gdb.init，\
然后定义一个shell alias: $ alias .gdb=”gdb -x gdb.init” 这样如果需要使用自动脚本，就用.gdb命令，\
否则用gdb进入交互状态的gdb。这样配置以后可以一个简单命令就开始调试，整个效率就能提高不少。

通过dmesg查看段错误位置  
^^^^^^^^^^^^^^^^^^^^^^^^

经常遇到的段错误，但是没有挂载gdb，如何定位问题？？同dmesg可以看到更多信息。

调用dmesg看到如下信息::

   [350082.160832] xdr_formate_0[23239]: segfault at 7ff1b67f5f28 ip 00007ff19ae204f3 sp 00007ff191a53b70 error 4 in general.so[7ff19ae1e000+3000]
   
有几个重要信息可以观测：  

* error 4 这个4代表用户态读内存越界    

    error number是由三个字位组成的，从高到底分别为bit2 bit1和bit0,所以它的取值范围是0~7. 

    - bit2: 值为1表示是用户态程序内存访问越界，值为0表示是内核态程序内存访问越界 
    - bit1: 值为1表示是写操作导致内存访问越界，值为0表示是读操作导致内存访问越界 
    - bit0: 值为1表示没有足够的权限访问非法地址的内容，值为0表示访问的非法地址根本没有对应的页面，也就是无效地址 
   
定位到c代码
====================

* 通过dmesg信息定位文件偏移  
    * bin
        ``[2721889.027883] rubicon[25303]: segfault at 0 ip 00000000004077b2 sp 00007fff60580a20 error 6 in rubicon[400000+52000]``

        文件偏移file_offset即为ip，即0x4077b2 
    * so
        ``[2722279.369451] rubicon[26548]: segfault at 0 ip 00007f6c673316cb sp 00007fff40343330 error 6 in libstream.so[7f6c6732c000+e000]``

        文件偏移file_offset=ip-动态库基址，即 0x7f6c673316cb - 0x7f6c6732c000

* 由文件偏移(file_offset)定位到c代码或汇编

    通过上述方法定位到文件偏移地址，在用下面的方法对应到c代码

    1. addr2line -e so/bin file_offset        
    2. gdb bin/so 进入gdb可视化界面，调用 disass file_offset
    3. objdump -S so/bin >tmp.dump 搜索file_offset即可。
        .. warning:: file_offset是十六进制

 * 示例 

    1. dmesg|grep rub  
            ``[2722279.369451] rubicon[26548]: segfault at 0 ip 00007f6c673316cb sp 00007fff40343330 error 6 in libstream.so[7f6c6732c000+e000]``

    2. file_offset为
           ``0x7f6c6732c000-0x7f6c6732c000=``
   
    3. 定位代码行 

        * 方法一
          source code below :: 

           [root@vmware promise]# addr2line -e /hs/lib64/libstream.so 0x56cb
           /root/work/xxx/SpotStream.cpp:66
        * 方法二
          source code below ::

           gdb /hs/lib64/libstream.so 
           disass 0x56cb
                63     int InitModuleOK()                                                          
                64     {
                65         int *p = NULL;                                                          
                66         *p = 0;                                                                 
                67         printf("bad\n");                                                        
                68         FiniStreamCmd();                                                        
                69         return 0;                                                               
                70     }   

        * 方法三
          source code below ::

            objdump -S /hs/lib64/libstream.so > /tmp/stream.dump 
            vim /tmp/stream.dump 搜索0x56cb  

* 实验过程 

  source code below :: 

    段错误代码如下 
    484     int *p = NULL;                                                              
    485     *p = 0;                                                                     
    486     printf("bad\n");

  1. rubicon   
       rubicon的main中添加了段错误代码
  2. rubicon调用的动态库       
       在spotStream.cpp添加段错误代码
  3. rubicon调用动态库，动态库中又已-l指定  
       在sniper中的eng.c添加段错误代码
       rubicon会dlopen SpotDpi.so ;
       在dpi的Makefile中又通过-L引用libsniper_x86_64.so        
  4. rubicon调用动态库，动态库又调用动态库  
       rubicon会dlopen SpotXdr.so;xdr中又dlopen http.so 
       在xdr的http.c中添加段错误代码  
