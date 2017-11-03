
suricata4.0.1源码分析
=======================

约定
--------------

* 假定按照这样进行配置编译
    
  ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-nfqueue --enable-lua


当前进度：开始分析AppLayerSetup->AppLayerParserRegisterProtocolParsers->RegisterHTPParsers

概述及全局观览
----------------

全局变量
***********

==================  ============================================================================================================================= 
 定义                                   说明                                                                                        
==================  ============================================================================================================================= 
host_mode             包括router和sniffer-only、auto三种模式，当使用auto模式时，在ips状态下设置为router,在ids下为sniffer-only
alpd_ctx              协议识别的全局变量，存放了各种协议识别使用的数据：如字符串，状态机等
==================  ============================================================================================================================= 

alpd_ctx介绍及内存布局
........................




main
---------

1. 主线    


.. graphviz::    

    digraph G {
                SCInstanceInit:初始化实例
                ->SC_ATOMIC_INIT 
                ->SCLogInitLogModule_创建日志子系统实例
                ->SCSetThreadName_设置线程名称
                ->ParseSizeInit_应该是解析用的_后续研究
                ->RunModeRegisterRunModes
                ->ConfInit_创建一个root节点_存储配置信息
                ->ParseCommandLine_解析命令信息_存放到SCInstance结构suri局部变量中
                ->FinalizeRunMode_检查和设定运行模式
                ->StartInternalRunMode_处理内部模式，比如查看当前支持的协议列表、支持的模式等
                ->GlobalsInitPreConfig_初始化全局配置_如队列、快速匹配链表等
                ->LoadYamlConfig_读取yaml格式配置文件
                ->PostConfLoadedSetup 
                
                }

   SCInstanceInit(初始化实例)->SC_ATOMIC_INIT->SCLogInitLogModule(创建日志子系统实例)->SCSetThreadName（设置线程名称）-> ParseSizeInit(应该是解析用的，后续研究)->RunModeRegisterRunModes
   ->ConfInit(创建一个root节点,存储配置信息)->ParseCommandLine(解析命令信息，存放到SCInstance suri局部变量中)->FinalizeRunMode(检查和设定运行模式)->StartInternalRunMode(处理内部模式，比如查看当前支持的协议列表、支持的模式等)
   ->GlobalsInitPreConfig(1.2 初始化全局配置 如队列、快速匹配链表等)－>LoadYamlConfig(1.3  读取yaml格式配置文件)
   ->PostConfLoadedSetup 

RunModeRegisterRunModes
*************************

   调用RunModeIdsXXXXRegister将各种模式注册好(todo:以pcap的模式作为模板进行研究)
   他们都调用RunModeRegisterNewRunMode紧张注册，

GlobalsInitPreConfig
***************************

    初始化trans_q 和data_queues（todo:深入分析两个变量） CreateLowercaseTable（字母转换数组初始化） 
    TimeInit SupportFastPatternForSigMatchTypes三个函数逐个调用。     
    SupportFastPatternForSigMatchTypes将DETECT_SM_LIST_PMATCH加入sm_fp_support_smlist_list链表，优先级是3 

1.3 todo: 

PostConfLoadedSetup
*********************

.. graphviz::    

    digraph G {

            label="PostConfLoadedSetup处理流程"

            PostConfLoadedSetup  [label="PostConfLoadedSetup"] ;
            MpmTableSetup [label="MpmTableSetup"] ;
            SpmTableSetup [label="SpmTableSetup"] ;
            AppLayerSetup [label="AppLayerSetup"] ;
            AppLayerProtoDetectSetup [label="AppLayerProtoDetectSetup"] ;
            AppLayerParserSetup [label="AppLayerParserSetup"] ;
            AppLayerParserRegisterProtocolParsers [label="AppLayerParserRegisterProtocolParsers \n注册协议识别字符串特征或端口特征；注册协议解析函数回调"] ;
            RegisterHTPParsers [label="RegisterHTPParsers \nhttp协议识别和解析初始化"] ;
            AppLayerProtoDetectConfProtoDetectionEnabled [label="AppLayerProtoDetectConfProtoDetectionEnabled"] ;
            AppLayerProtoDetectRegisterProtocol [label="AppLayerProtoDetectRegisterProtocol"] ;
            HTPRegisterPatternsForProtocolDetection [label="HTPRegisterPatternsForProtocolDetection\n将字符串、端口特征添加到状态机"] ;
            AppLayerParserRegisterXXXXX [label="HTPRegisterPatternsForProtocolDetection\n添加解析相关函数集"] ;
            RegisterSSLParsers [label="RegisterSSLParsers"] ; 
            RegisterFTPParsers [label="RegisterFTPParsers"] ; 
            AppLayerProtoDetectPrepareState [label="AppLayerProtoDetectPrepareState"] ;
            SCHInfoLoadFromConfig [label="SCHInfoLoadFromConfig"] ;
            AppLayerProtoDetectPMMapSignatures [label="AppLayerProtoDetectPMMapSignatures "] ; 
            AppLayerProtoDetectPMPrepareMpm [label="AppLayerProtoDetectPrepareState"] ; 
            dengdeng [label="......"] ;
            PostConfLoadedSetup->SpmTableSetup
            PostConfLoadedSetup->MpmTableSetup
            PostConfLoadedSetup->AppLayerSetup
                AppLayerSetup->AppLayerParserSetup
                AppLayerSetup->AppLayerProtoDetectSetup
                AppLayerSetup->AppLayerParserRegisterProtocolParsers
                    AppLayerParserRegisterProtocolParsers->RegisterHTPParsers
                        RegisterHTPParsers->AppLayerProtoDetectConfProtoDetectionEnabled
                        RegisterHTPParsers->AppLayerProtoDetectRegisterProtocol
                        RegisterHTPParsers->HTPRegisterPatternsForProtocolDetection
                        RegisterHTPParsers->AppLayerParserRegisterXXXXX
                    AppLayerParserRegisterProtocolParsers->RegisterFTPParsers
                    AppLayerParserRegisterProtocolParsers->dengdeng
                    AppLayerParserRegisterProtocolParsers->RegisterSSLParsers
            PostConfLoadedSetup->AppLayerProtoDetectPrepareState
                AppLayerProtoDetectPrepareState->AppLayerProtoDetectPMMapSignatures
                AppLayerProtoDetectPrepareState->AppLayerProtoDetectPMPrepareMpm
            PostConfLoadedSetup->SCHInfoLoadFromConfig


    }

    MpmTableSetup(注册多模式匹配算法)->SpmTableSetup(注册单模式匹配算法)->网卡offloading、checksum等配置读取->AppLayerSetup


* MpmTableSetup

注册各种多模匹配算法，将ac ac-cuda ac_bs ac_tile hyperscan 这几种多模式匹配算法，注册到mpm_table(结构为MpmTableElmt)

全局变量中 mpm_default_matcher作为默认配置

* SpmTableSetup

注册各种单模匹配算法，将bm hyperscan这两种单模式匹配算法，注册到spm_table(结构为SpmTableElmt)的全局变量中

* AppLayerSetup 

* AppLayerProtoDetectSetup
           
             主要是对alpd_ctxl4层协议(tcp,udp,icmp,sctp)层面的多模和单模的注册和初始化，
             主要是给alpd_ctx.spm_global_thread_ctx和MpmInitCtx调用进行赋值(todo:多模匹配算法插件接口)

             alpd_ctx是协议识别的全局变量，存放了各种协议识别使用的数据：如字符串，状态机等

* AppLayerParserSetup

* AppLayerParserRegisterProtocolParsers
    
        注册协议识别字符串特征或端口特征；注册协议解析函数回调

        * RegisterHTPParsers
           
            http协议识别字符串注册，解析函数注册 
           
            * AppLayerProtoDetectConfProtoDetectionEnabled(判断该协议是否启动)
            * AppLayerProtoDetectRegisterProtocol(注册http协议识别)
            * HTPRegisterPatternsForProtocolDetection:(将该协议识别的特征串放入alpd_ctx相应的状态机中)

              这里将调用AppLayerProtoDetectPMRegisterPatternCI/CS注册字符串特征，
              如果有端口特征则通过AppLayerProtoDetectPPRegister注册（如RegisterDNSUDPParsers）,该函数有2个参数ProbingParserFPtr，
              当命中端口后，还会运行该函数做进一步判断。

            * AppLayerParserRegisterXXXXX(该系列函数是注册协议解析的相关插件,todo:研究解析过程)
         
* AppLayerProtoDetectPrepareState
          
            (todo:详细分析协议维度字符串添加过程、内存布局)：添加特征到状态机并编译
           
            * AppLayerProtoDetectPMMapSignatures :添加到状态机
            
            * AppLayerProtoDetectPMPrepareMpm :编译

* SCHInfoLoadFromConfig

           将配置文件中的host-os-policy的配置加入到一棵radix树上，在匹配是使用。(todo:识别或重组时使用？？)

      
                              

开源引擎借鉴
-------------

   支持协议维度识别和解析
   协议识别、解析插件化
   特征区分服务端和客户端
