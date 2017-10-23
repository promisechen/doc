vtune使用教程
=============

安装部署
---------

Vtune可以重复安装试用，但要先卸载之前的版本。
这是一个license。

.. figure img/Intel_Vtune_TBE.lic

* 下载 

  | 从官网上下载intel-vtune-amplifier-xe的linux版本文件
  | vtune_amplifier_xe_2015_update2.tar.gz
  | (https://software.intel.com/zh-cn/intel-vtune-amplifier-xe/ 似乎翻墙才能连接，下载)

* 解压 

   tar zxvf vtune_amplifier_xe_2015_update2.tar.gz

* 进入目录，执行安装脚本

  |  #cd vtune_amplifier_xe_2015_update2
  |  #./install.sh
        
  figure:: img/vtune_install_01.png

      | 直接回车

    .. figure:: img/vtune_install_02.png

      | 敲入accept

    .. figure:: img/vtune_install_03.png
    .. figure:: img/vtune_install_04.png
    .. figure:: img/vtune_install_05.png
    .. figure:: img/vtune_install_06.png
    .. figure:: img/vtune_install_07.png
    .. figure:: img/vtune_install_08.png

        然后安装自动完成。
    * 最后增加环境变量。运行#source /opt/intel/vtune_amplifier_xe_2015/amplxe-vars.sh



快速使用
---------

首先通过file->new ->project .来创建一个project. 输入Project name和location.
        2.之后会弹出一个配置Project Properties 的对话框，选择需要profile的对象target, 在这里提供了3中Target的类型：
        •   Launch Application, 在下面的Application中输入你要profile的应用程序，于是在后面开始profile的时候，VTune会启动这个应用程序。
        •   Attach to Process, 在下面的ProcessID中输入进程ID即可，主要针对的是已经启动的后台程序，VTune可以attach to Process对某一个时段的操作进行profile.
        •   Profile System，不需要选择target,直接对系统进程进行profile.
        3.在配置完Project Properties之后，就可以选择new Analysis 图标，对这个project选择的target创建新的analysis. 可以针对一个project创建很多次的analysis。
        4.在创建新的analysis时，需要选择analysis的类型，我只试过HotSpots这类型的分析类型，选择类型之后，就可以点击右边的start按钮开始profile工作了。
        5.在完成profile时，点击stop按钮，就会结束profile，接着对profile的结构进行分析整理，以图表的形式展现出每个耗时的hotspot。
        6.于是程序员就可以针对hotspot，进行有针对性的优化。
        图形化结果说明：
        窗口
        含义说明
        Summary
        提供你的执行程序的一般信息。所用的时间包括执行程序线程的活动处理时间+cpu花在并行线程库上面的时间（如TBB和OpenMP）+cpu等待时间（spin time）
        Bottom-up
        查看函数/模块/线程调用的时间耗费
        Caller/Callee
        函数调用的时间
        Top-down tree
        以树形结构展示每个调用所花费的时间及所占比，可以从时间花费最多的地方往下一层一层的展开，找到关键函数，分析其性能
        补充说明，双击可以进入函数。加载后如果看不函数信息，需要添加路径。
        参考：https://software.intel.com/zh-cn/blogs/2011/04/20/vtune-amplifier-xe-2
        命令行分析语句
        参考链接 https://software.intel.com/zh-cn/blogs/2010/11/10/amplxe-cl/





