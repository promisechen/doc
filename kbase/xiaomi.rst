..  Copyright (C), 2014-2016
    All rights reserved.

    @author promisechen
    @date 2017.11.06

小米路由器上开发网络监控程序
==============================


小米路由器mini刷机  
---------------------

  如需切换ROM版本，请前往miwifi.com `点击链接 <http://www1.miwifi.com/miwifi_download.html>`_ 
  下载页->点击ROM，下载更新日志 刷机教程 下载小米路由器mini ROM for Mini 开发版

  首先，请先准备一个U盘，并确保这个U盘的格式为FAT或FAT32.  
  接下来，就是具体的操作流程了。  

 |  1. 在miwifi.com官网下载路由器对应的ROM包，并将其放在U盘的根目录下，命名为miwifi.bin  
 |  2. 断开小米路由器mini的电源，将U盘插入路由器的USB接口  
 |  3. 按下reset按钮后重新接入电源，待指示灯变为黄色闪烁状态后松开reset键  
 |  4. 等待5~8分钟，刷机完成之后系统会自动重启并进入正常的启动状态（指示灯由
 |    黄灯常亮变为蓝灯常亮），此时，说明刷机成功完成！  

 | 如果出现异常/失败/U盘无法读取状态，会进入红灯状态，建议重试或更换U盘再试。    
 | 注意：如果你的电脑通过网线方式上网，无线方式连接到小米路由器，同时小米路由器无法联网。  
 | 但是, 如果你很想很想联网，你需要将无线的dns修改成其他的dns地址.如果你想管理路由器，可以直接输入192.168.31.1进行管理。    

开启ssh  
------------

注意：先删除掉上一步骤遗留的文件：miwifi.bin 否则会刷rom而不刷ssh  
  参照 ` "百度经验的分享" <http://jingyan.baidu.com/article/624e7459ae65e834e8ba5afd.html>`_ 


开发环境搭建  
----------------

* 下载开发包  

   访问 `这里 <http://www1.miwifi.com/miwifi_open.html>`_ 下载
   如果是14年左右的小米路由器，可以从`这里直接<https://pan.baidu.com/s/1kVaK5Gr>`_下载
* 部署开发环境  

   * 首先弄一个centos fedora 之类的linux系统，我用的是centos7.0 64位  
   * 压缩包解压，sdk_package_r1c/toolchain/是开发工具链  
   * 在/etc/profile中添加  
         export PATH=$PATH:/root/git/miwifi/lib/sdk_package_r1c/toolchain/bin  
   * 执行soure /etc/profile  

* hello world  

   | 编写标准c代码，使用mipsel-openwrt-linux-gcc进行编译
   |     如： mipsel-openwrt-linux-gcc test.c 
   | 然后通过scp上传到小米路由器上，执行即可

.. note:: 如果出现缺少库之类的问题，使用 yum whatprovides libxxx 然后根据whatprovides   
          返回的信息，使用yum install xxx 进行安装.

   
交叉编译libpcap 
--------------------------  
   
* 创建交叉编译目录 
  ::

     mkdir /usr/local/mips 
     ./configure --host=mipsel-openwrt-linux  --with-pcap=linux --prefix=/usr/local/mips  //host是目标机器 prefix  
     make && make install  

* 交叉编译pcap小程序  
  ::

     mipsel-openwrt-linux-gcc pcaptest.c -lpcap -L/usr/local/mips/lib -I/usr/local/include -o pcaptest  

* 上传真实机器运行  
  
  :: 

    将编译好的pcaptest和libpcap.so上传到/extdisks/sda1/pcap/   (extdisks/sda1/pcap可以自己随意指定)
    export  LD_LIBRARY_PATH=/extdisks/sda1/pcap/  
    然后执行./pcaptest  

远程抓包
-------------

* rpcapd交叉编译

 
    1. `下载Wpcap <http://www.winpcap.org/install/bin/WpcapSrc_4_1_3.zip>`_
    
    2. 解压  
    
        unzip WpcapSrc_4_1_3.zip  
    3. 修改源代码

       cd winpcap/wpcap/libpcap/ 修改confiugre文件，把下面两段注释掉 

     :: 

        #if test -z "$with_pcap" && test "$cross_compiling" = yes; then
        # {
            {
                echo "$as_me:$LINENO: error: pcap type not determined when cross-compiling; use --with-pcap=..." >&5
        #echo "$as_me: error: pcap type not determined when cross-compiling; use --with-pcap=..." >&2;}
        #   {
            (exit 1); exit 1; }; }
        #fi
            .......
        #  if test $ac_cv_linux_vers = unknown ; then
        #   {
            {
                echo "$as_me:$LINENO: error: cannot determine linux version when cross-compiling" >&5
        #echo "$as_me: error: cannot determine linux version when cross-compiling" >&2;}
        #   {
            (exit 1); exit 1; }; }

    4. 编译libpcap 

      ::
      
        ./configure --host=mipsel-openwrt-linux
        make
        
    5. 编译rpcapd
    
      :: 
         
        修改Makefile
        cd rpcapd;
        CC =mipsel-openwrt-linux-gcc
        LIB +=-ldl
        make
        
    6. 上传真实机器运行

        ./rpcapd -4 -n -p 333

    7. 在装有wireshark的机器上进行抓包

        打开抓包选项卡，在接口框中输入格式：rpcap://[ip]:port/if-name
        rpcap://[192.168.1.104]:335/br-lan


gdb调试
-----------

`下载后交叉编译 <https://pan.baidu.com/s/1hr4wEhe>`_

