
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


当前进度：开始分析PostConfLoadedSetup->MayDaemonize


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
tmqh_table            提供了4种类型队列:simple,flow,packetpool,nfq,其中nfq不同于其他三种队列，他是内核中的Netfilter Queue。
                      simple是简单的先入先出的普通队列,packetpool
tmm_modules           线程模块的全局数组，有pcap/pfring/netmap等收包类线程，有flow管理类
==================  ============================================================================================================================= 

alpd_ctx介绍及内存布局
........................


sigmatch_table介绍(todo)
..........................


tmqh_table对应的Tmqh结构体说明
.................................
todo:tm_queuehandlers.h中

 :: 

    enum {
    
        TMQH_SIMPLE,
        TMQH_NFQ,
        TMQH_PACKETPOOL,
        TMQH_FLOW,
        
        TMQH_SIZE,
    }; 
    typedef struct Tmqh_ {
    
        const char *name;                          
        Packet *(*InHandler)(ThreadVars *);           /*< 是针对ThreadVars的输入，而非入队的意思，即从相应队列取出Packet。by clx 20171107 */
        void (*InShutdownHandler)(ThreadVars *);      /*< 貌似是把队列中的报文都拿出来的意思??。by clx 20171107 */
        void (*OutHandler)(ThreadVars *, Packet *);   /*< 将报文压入队列中。by clx 20171107*/
        void *(*OutHandlerCtxSetup)(const char *);    /*< 初始化线程上下文。by clx 20171107*/
        void (*OutHandlerCtxFree)(void *);            /*< 释放线程上下文。by clx 20171107*/
        void (*RegisterTests)(void);
    } Tmqh; 
    Tmqh tmqh_table[TMQH_SIZE];


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
            size="1024,1024";
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
            SigTableSetup [label="SigTableSetup\n注册关键字回调函数 "] ; 
            DetectSidRegister [label="DetectSidRegister"] ;
            DetectContentRegister [label="DetectContentRegister"] ; 
            DetectUricontentRegister [label="DetectUricontentRegister"] ; 
            DetectBufferTypeFinalizeRegistration [label="DetectBufferTypeFinalizeRegistration"] ;
            TmqhSetup [label="TmqhSetup\n注册队列接口"] ;
            TmqhSimpleRegister [label="TmqhSimpleRegister\n普通队列"] ; 
            TmqhNfqRegister [label="TmqhNfqRegister\n内核Netfilter 队列"] ;
            TmqhPacketpoolRegister [label="TmqhPacketpoolRegister\n类似mbuf"] ;
            TmqhFlowRegister [label="TmqhFlowRegister\n根据五元组hash的队列"]
            SigParsePrepare [label="SigParsePrepare\n初始化sig解析正则库"]
            SCProtoNameInit [label="SCProtoNameInit\n从/etc/protocols获取协议名称"]
            xxxTagInit [label="TagInitCtx/PacketAlertTagInit/ThresholdInit\nHostBitInitCtx/IPPairBitInitCtx"]
            RegisterAllModules [label="RegisterAllModules\n注册各线程模块回调"]
            TmModuleFlowManagerRegister [label="TmModuleFlowManagerRegister\n流表管理"]
            TmModuleReceiveAFPRegister [label="TmModuleReceiveAFPRegister\nafp接收线程"]
            TmModuleDecodeAFPRegister [label="TmModuleDecodeAFPRegister\nafp解码线程"]
            AppLayerHtpNeedFileInspection [label="AppLayerHtpNeedFileInspection\n设置htp库部分配置"]
            StorageFinalize [label="StorageFinalize\n初始化storage_map"]
            TmModuleRunInit [label="TmModuleRunInit\n调用tm的init"]
            HostInitConfig [label="HostInitConfig\n主机内存初始化"]      

            PreRunInit [label="PreRunInit\n流表，ip分片重组，tcp分片重组分配内存并进行初始化"]      
            DefragInit [label="DefragInit\nIP分片内存初始化"]      
            FlowInitConfig [label="FlowInitConfig\nFLOW内存初始化"]      
            IPPairInitConfig [label="IPPairInitConfig\nIP声望内存初始化"]      
            StreamTcpInitConfig [label="StreamTcpInitConfig\n流重组内存初始化"] 

            dengdeng [label="......"] ;
            RegisterXXXParsers [label="......"] ;
            DetectXXXXRegister [label="......"] ; 
            TmModuleReceiveXXXRegister [label="......"] ;
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
                    AppLayerParserRegisterProtocolParsers->RegisterXXXParsers
                    AppLayerParserRegisterProtocolParsers->RegisterSSLParsers
            PostConfLoadedSetup->AppLayerProtoDetectPrepareState
                AppLayerProtoDetectPrepareState->AppLayerProtoDetectPMMapSignatures
                AppLayerProtoDetectPrepareState->AppLayerProtoDetectPMPrepareMpm
            PostConfLoadedSetup->SCHInfoLoadFromConfig
            PostConfLoadedSetup->SigTableSetup
                SigTableSetup->DetectSidRegister
                SigTableSetup->DetectContentRegister
                SigTableSetup->DetectXXXXRegister
                SigTableSetup->DetectUricontentRegister
                SigTableSetup->DetectBufferTypeFinalizeRegistration
            PostConfLoadedSetup->TmqhSetup
                TmqhSetup->TmqhSimpleRegister
                TmqhSetup->TmqhNfqRegister
                TmqhSetup->TmqhPacketpoolRegister
                TmqhSetup->TmqhFlowRegister 
            PostConfLoadedSetup->SigParsePrepare
            PostConfLoadedSetup->SCProtoNameInit

            PostConfLoadedSetup->xxxTagInit
            PostConfLoadedSetup->RegisterAllModules
                RegisterAllModules->TmModuleFlowManagerRegister
                RegisterAllModules->TmModuleReceiveXXXRegister
                RegisterAllModules->TmModuleReceiveAFPRegister 
                RegisterAllModules->TmModuleDecodeAFPRegister 
            PostConfLoadedSetup->AppLayerHtpNeedFileInspection
            PostConfLoadedSetup->StorageFinalize
            PostConfLoadedSetup->TmModuleRunInit

            PostConfLoadedSetup->HostInitConfig  
            PostConfLoadedSetup->PreRunInit
                PreRunInit->DefragInit
                PreRunInit->FlowInitConfig
                PreRunInit->IPPairInitConfig
                PreRunInit->StreamTcpInitConfig
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

* TmqhSetup

       注册4中类型队列，后续各线程交互时使用  

    * TmqhSimpleRegister 

            简单的普通的入队出队队列，主要注册了TmqhInputSimple和TmqhOutputSimple，TmqhInputSimple
            输入回调，即从相应队列中获取报文，这里的input是针对ThreadVars来说的。

    * TmqhNfqRegister

            内核层面的队列，即 Netfilter Queue队列，与其他三种队列不同，他只需要注册OutHandler

    * TmqhPacketpoolRegister

            这个更像是一个dpdk中的mbuf，内核中的skb_mbuf之类的ringbuffer. 这个其实更像说是内存池，这种队列应该是用在
            收包这一层层面。

    * TmqhFlowRegister 
            根据flow进行分发的队列,出队列与Simple是一样的，入队会根据flow的hash进行除余得到相应的队列。
            根据配置的不同，将选择不同的分发算法:TmqhOutputFlowHash TmqhOutputFlowIPPair 

        TmqhOutputFlowIPPair的部分代码 :: 
        
             void TmqhOutputFlowIPPair(ThreadVars *tv, Packet *p)
             {

                 int16_t qid = 0;
                 uint32_t addr_hash = 0;
                 int i;

                 TmqhFlowCtx *ctx = (TmqhFlowCtx *)tv->outctx;

                 if (p->src.family == AF_INET6) {

                 for (i = 0; i < 4; i++) {

                     addr_hash += p->src.addr_data32[i] + p->dst.addr_data32[i];
                 }
                 } else {

                     addr_hash = p->src.addr_data32[0] + p->dst.addr_data32[0];
                 }

                 /* we don't have to worry about possible overflow, since
                 * ctx->size will be lesser than 2 ** 31 for sure */
                   qid = addr_hash % ctx->size;

                 PacketQueue *q = ctx->queues[qid].q;
                 SCMutexLock(&q->mutex_q);
                 PacketEnqueue(q, p);
                 SCCondSignal(&q->cond_q);
                 SCMutexUnlock(&q->mutex_q);

                 return;
             }

* SigParsePrepare 

   初始化config_pcre、config_pcre_extra、option_pcre三个全局变量，后面解析使用 
    ::

        opts |= PCRE_UNGREEDY;
        config_pcre = pcre_compile(regexstr, opts, &eb, &eo, NULL);
        if(config_pcre == NULL)
        {
        
            SCLogError(SC_ERR_PCRE_COMPILE, "pcre compile of \"%s\" failed at offset %" PRId32 ": %s", regexstr, eo, eb);
            exit(1);
        }
        
        config_pcre_extra = pcre_study(config_pcre, 0, &eb);
        if(eb != NULL)
        {
        
            SCLogError(SC_ERR_PCRE_STUDY, "pcre study failed: %s", eb);
            exit(1);
        }
        
        regexstr = OPTION_PCRE;
        opts |= PCRE_UNGREEDY;
        
        option_pcre = pcre_compile(regexstr, opts, &eb, &eo, NULL);
        if(option_pcre == NULL)
        {
        
            SCLogError(SC_ERR_PCRE_COMPILE, "pcre compile of \"%s\" failed at offset %" PRId32 ": %s", regexstr, eo, eb);
            exit(1);
        }
        
* xxxTagInit 
    存储结构的初始化，有三种存储方式STORAGE_HOST\STORAGE_FLOW\STORAGE_IPPAIR分别用于不同类型的存储。
    这里共初始化了host_tag_id、flow_tag_id、threshold_id、host_bit_id、ippair_bit_id5个储存实体对象。
    应该与与规则中的tag、threshould关键字的实现相关;

  :: 
    
    static StorageList *storage_list = NULL; /**< by clx 20171109 储存链表*/
    static int storage_max_id[STORAGE_MAX];  /**< by clx 20171109 三种储存方式的id编号*/
    static int storage_registraton_closed = 0; /**< by clx 20171109 关闭标记，当设置为1时，不在注册*/
    static StorageMapping **storage_map = NULL;/**< by clx 20171109 将储存链表上所有storage实体做映射成二维数组，
    通过储存类型和在该类型的储存方式对应的id进行读取。如storage_map[STORAGE_HOST][host_tag_id]读取host_tag_id的存储注册函数*/


* DetectAddressTestConfVars、DetectPortTestConfVars
    检查配置文件中vars.address-groups和vars.port-groups的合法性。

* RegisterAllModules
    注册线程模式:流表管理相关、报文接收方式(pcap/pfring/netmap等)
    线程类型共下面几类:其中文接收方式使用的是RECEIVE_TM和DECODE_TM，
    其中五元组表有MANAGEMENT_TM|TM_FLAG_STREAM_TM|TM_FLAG_DETECT_TM三个专用类型,以af-packet为例

  ::

      #define TM_FLAG_RECEIVE_TM      0x01
      #define TM_FLAG_DECODE_TM       0x02
      #define TM_FLAG_STREAM_TM       0x04
      #define TM_FLAG_DETECT_TM       0x08
      #define TM_FLAG_LOGAPI_TM       0x10 /**< TM is run by Log API */
      #define TM_FLAG_MANAGEMENT_TM   0x20
      #define TM_FLAG_COMMAND_TM      0x40

  以af-packet为例:TmModuleDecodeAFPRegister和TmModuleReceiveAFPRegister分别定义了收包和解码的回调。
    ::  

        void TmModuleReceiveAFPRegister (void)
        {
            tmm_modules[TMM_RECEIVEAFP].name = "ReceiveAFP";
            tmm_modules[TMM_RECEIVEAFP].ThreadInit = NoAFPSupportExit;
            tmm_modules[TMM_RECEIVEAFP].Func = NULL;
            tmm_modules[TMM_RECEIVEAFP].ThreadExitPrintStats = NULL;
            tmm_modules[TMM_RECEIVEAFP].ThreadDeinit = NULL;
            tmm_modules[TMM_RECEIVEAFP].RegisterTests = NULL;
            tmm_modules[TMM_RECEIVEAFP].cap_flags = 0;
            tmm_modules[TMM_RECEIVEAFP].flags = TM_FLAG_RECEIVE_TM;
        }
        
        /**
        * \brief Registration Function for DecodeAFP.
        * \todo Unit tests are needed for this module.
        */
        void TmModuleDecodeAFPRegister (void)
        {
            tmm_modules[TMM_DECODEAFP].name = "DecodeAFP";
            tmm_modules[TMM_DECODEAFP].ThreadInit = NoAFPSupportExit;
            tmm_modules[TMM_DECODEAFP].Func = NULL;
            tmm_modules[TMM_DECODEAFP].ThreadExitPrintStats = NULL;
            tmm_modules[TMM_DECODEAFP].ThreadDeinit = NULL;
            tmm_modules[TMM_DECODEAFP].RegisterTests = NULL;
            tmm_modules[TMM_DECODEAFP].cap_flags = 0;
            tmm_modules[TMM_DECODEAFP].flags = TM_FLAG_DECODE_TM;
        }
* AppLayerHtpNeedFileInspection    

     为htp库设置一些标记，如解析响应、解析请求的标记
     
* StorageFinalize    
 
      将xxxTagInit注册的实体，将储存链表上所有storage实体做映射成二维数组，
      通过储存类型和在该类型的储存方式对应的id进行读取。如storage_map[STORAGE_HOST][host_tag_id]读取host_tag_id的存储注册函数

* TmModuleRunInit

       调用tmm_modules[i]->Init进行模块初始化
* MayDaemonize 

    | 后台运行,检查是否进入Daemon模式。若需要进入Daemon模式，则会检测pidfile是
    | 否已经存在（daemon下只能有一个实例运行），然后进行Daemonize，最后创建一个
    | pidfile。Daemonize的主要思路是：fork->子进程调用setsid创建一个新的session
    | ，关闭stdin、stdout、stderr，并告诉父进程 –> 父进程等待子进程通知，然后退
    | 出 –> 子进程继续执行.

* InitSignalHandler 
    
    | 注册各种信号,初始化信号handler。首先为SIGINT（ctrl-c触发）和SIGTERM（不带
    | 参数kill时触发）这两个常规退出信号分别注册handler，对SIGINT的处理是设置程
    | 序的状态标志为STOP，即让程序优雅地退出；而对SIGTERM是设置为KILL，即强杀。
    | 接着，程序会忽略SIGPIPE（这个信号通常是在Socket通信时向已关闭的连接另一端
    | 发送数据时收到）和SIGSYS（当进程尝试执行一个不存在的系统调用时收到）信号，
    | 以加强程序的容错性和健壮性。

* HostInitConfig 
    与FlowInitConfig类似: host_hash,host_config
* SCAsn1LoadConfig 
    读取asn1-max-frames设置到全局变量asn1_max_frames_config
* CoredumpLoadConfig
    coredump设置 
* DecodeGlobalConfig 
    设置是否解析g_teredo_enabled
* PreRunInit 
   为流表，ip分片重组，tcp分片重组分配内存并进行初始化    
    * StatsInit 统计初始化
    * DefragInit 分片重组,暂不关心
    * FlowInitConfig 
        初始化flow的配置，flow_config是flow的全局配置,flow_hash是hash表,flow_spare_q flow节点
        的队列，调用FlowAlloc分配内存之后，将把分配的flow放到flow_spare_q队列中。

        ::

          /* global flow config */
          typedef struct FlowCnf_
          {

            uint32_t hash_rand;
            uint32_t hash_size;  /**<by clx 20171110 hash筒的大小*/
            uint64_t memcap;     /**<by clx 20171110 flows最大占的内存限制*/
            uint32_t max_flows; 
            uint32_t prealloc;   /**<by clx 20171110 最大并发数*/

            uint32_t timeout_new;
            uint32_t timeout_est;

            uint32_t emerg_timeout_new;
            uint32_t emerg_timeout_est;
            uint32_t emergency_recovery;

          } FlowConfig;
    * IPPairInitConfig 
         类似FlowInitConfig初始化ip的相关内存 
    * LogFilestoreInitConfig 文件存储的配置
    * StreamTcpInitConfig 流重组的初始化,todo:暂不看tcp流重组的细节
    * AppLayerParserPostStreamSetup todo:暂不看tcp流重组的细节
    * AppLayerRegisterGlobalCounters todo:设置一些计数配置，后面研究下咋用的。

参考文献
--------------
      http://blog.csdn.net/vevenlcf/article/details/50600324

开源引擎借鉴
-------------

  | 支持协议维度识别和解析
  | 协议识别、解析插件化
  | 特征区分服务端和客户端
