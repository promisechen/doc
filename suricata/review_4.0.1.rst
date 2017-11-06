
suricata4.0.1源码分析
=======================

约定
--------------

* 代码对应位置

  | 在做review的时候，会在这个链接中写相关注释，尽量不在本文档中贴代码。
  | https://github.com/promisechen/suricata/tree/master/suricata-4.0.1

* 交流方式

  | 可以通过微信promisechen或issue(推荐)进行沟通
  | 可在这里建立issue https://github.com/promisechen/suricata/issues 

* 假定按照这样进行配置编译
    
  | ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-nfqueue --enable-lua


当前进度：开始分析PostConfLoadedSetup->TmqhSetup


概述及全局观览
----------------

全局变量
***********

==================  ============================================================================================================================= 
 定义                                   说明                                                                                        
==================  ============================================================================================================================= 
host_mode             包括router和sniffer-only、auto三种模式，当使用auto模式时，在ips状态下设置为router,在ids下为sniffer-only
alpd_ctx              协议识别的全局变量，存放了各种协议识别使用的数据：如字符串，状态机等
sigmatch_table        特征关键字匹配表，sid、priority、msg、within、distance等等。该变量主要应用于应用的识别和规则的检测。
==================  ============================================================================================================================= 

alpd_ctx介绍及内存布局
........................


sigmatch_table介绍(todo)
..........................

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
            SigTableSetup [label="SigTableSetup\n注册关键字回调函数"] ; 
            DetectSidRegister [label="DetectSidRegister"] ;
            DetectContentRegister [label="DetectContentRegister"] ; 
            DetectUricontentRegister [label="DetectUricontentRegister"] ; 
            DetectBufferTypeFinalizeRegistration [label="DetectBufferTypeFinalizeRegistration"] ;
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
            PostConfLoadedSetup->SigTableSetup
                SigTableSetup->DetectSidRegister
                SigTableSetup->DetectContentRegister
                SigTableSetup->dengdeng
                SigTableSetup->DetectUricontentRegister
                SigTableSetup-DetectBufferTypeFinalizeRegistration

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

* SigTableSetup 
    注册关键字的各种回调,比如注册sid,content等相关回调，在读取加载规则库、应用识别的时候将调用相关回调函数.
    目前看到这些函数应该是被SigInit调用.这里注册的关键非常的多，可以慢慢分析自己感兴趣的,其中发现很多关键字没有注册
    Match这个回调。http相关的注册项有很多,http的一些注册还会初始化一些资源,后面以DetectHttpUriRegister为例。
    * DetectSidRegister
        注册了重要的函数DetectSidSetup，该函数将在加载规则库的时候，被调用。

        DetectSidSetup将会把规则库中的sidstr赋给s->id

         :: 

            static int DetectSidSetup (DetectEngineCtx *de_ctx, Signature *s, const char *sidstr)
            {
            
                unsigned long id = 0;
                char *endptr = NULL;
                id = strtoul(sidstr, &endptr, 10);
                if (endptr == NULL || *endptr != '\0') {
            
                SCLogError(SC_ERR_INVALID_SIGNATURE, "invalid character as arg "
                "to sid keyword");
                goto error;
                }
                if (id >= UINT_MAX) {
            
                SCLogError(SC_ERR_INVALID_NUMERIC_VALUE, "sid value to high, max %u", UINT_MAX);
                goto error;
                }
                if (id == 0) {
            
                SCLogError(SC_ERR_INVALID_NUMERIC_VALUE, "sid value 0 is invalid");
                goto error;
                }
                if (s->id > 0) {
            
                SCLogError(SC_ERR_INVALID_RULE_ARGUMENT, "duplicated 'sid' keyword detected");
                goto error;
                }
            
                s->id = (uint32_t)id;
                return 0;
            
                error:
                return -1;
            }

    * DetectPriorityRegister

      注册了重要的函数DetectPrioritySetup，该函数将在加载规则库的时候，被调用。
      DetectPrioritySetup将把规则库中的rawstr赋值给s->prio,但是相对DetectSidSetup多了一些pcre_exec、pcre_copy_substring相关函数调用,做什么用的呢？？
      他们主要是判断关键字是否合法，并提取相关字段，注意regex、regex_study是static类型的,这2个全局变量在很多文件中都存在。
    
    * DetectHttpUriRegister 
      也注册了Setup回调。注册回调之后，重点注册了DetectAppLayerMpmRegister和DetectAppLayerInspectEngineRegister(todo:检查相关注册)


开源引擎借鉴
-------------

  | 支持协议维度识别和解析
  | 协议识别、解析插件化
  | 特征区分服务端和客户端
