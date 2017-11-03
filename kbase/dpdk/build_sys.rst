
编译系统
========


概述
----

本文主要介绍dpdk的源码组织及编译系统，这些内容对于了解dpdk源码、
根据需要裁剪dpdk功能、编译dpdk本身或基于dpdk的应用程序等都有帮助。

文后给出了对dpdk编译系统进行修改以达成某些特定需求的实例。


源码组织
---------

dpdk源码包可以在 `dpdk.org <http://www.dpdk.org/download>`_ 下载。\
2.2版本后，版本号直接变为16.04。

git方式获取源码（只读）::

    git clone git://dpdk.org/dpdk , 或
    git clone http://dpdk.org/git/dpdk

RTE_SDK是编译dpdk之前需要设置的一个环境变量，其值是dpdk源码包解压\
后的主目录。以下就以<RTE_SDK>表示dpdk源码主目录::

    <RTE_SDK>
    ├── app           # 一些测试程序
    ├── buildtools    # XXX
    ├── config        # 配置文件
    ├── doc           # rst文档
    ├── drivers       # 设备驱动程序(poll-mode)
    ├── examples      # 示例程序
    ├── GNUmakefile   # Head Makefile for compiling rte SDK
    ├── lib           # dpdk库
    ├── LICENSE.GPL   
    ├── LICENSE.LGPL  
    ├── MAINTAINERS   # dpdk维护者信息列表
    ├── Makefile      # 无用
    ├── mk            # lib和app的Makefiles
    ├── pkg           # 软件包相关（？） 
    ├── README        # 简要说明
    ├── scripts       # 脚本
    └── tools         # 辅助工具/脚本


config
.......

这里的配置文件用于对不同的编译目标（target）进行配置，包括是否编译\
某些库，调试选项是否开启等。其中 ``common_base`` 文件非常重要，\
其中的内容需要非常熟悉。

mk
....

这里的Makefile文件（后缀.mk）对应多个target、源码种类或编译选项。\
例如rte.lib.mk用于编译<RTE_SDK>/lib，rte.combinedlib.mk用于将dpddk\
编译为单个组合库，rte.app.mk用于编译基于dpdk的应用程序。

drivers
........

poll模式的设备驱动源码，例如10GbE，40GbE，PCAP驱动等。

lib
....

dpdk库源码，每个库一个目录。例如EAL, mbuf, mempool, ring, hash等。

tools
......

一些有用的工具:

* **cpu_layout.py** 显示JCPU socket/lcore布局信息
* **dpdk-devbind.py** dpdk设备驱动绑定
* **dpdk-pmdinfo.py** dump a PMDs hardware support info
* **dpdk-setup.sh** dpdk编译/配置脚本

app
....

主要是一些测试程序源码，可用于测试dpdk编译和配置有没有问题。

examples
.........

示例程序，对学习如何使用dpdk非常有帮助。见 
`文档 <http://dpdk.org/doc/guides/sample_app_ug/index.html>`_ 。


编译系统
----------

本节描述dpdk所采用的编译系统，以及一些约束条件和机制。

涉及dpdk的编译工作主要有两类：

* 编译dpdk库和示例程序，编译系统生成头文件，二进制库文件和示例程序
* 编译外部程序或库，它们使用已安装的dpdk

编译dpdk
.........

在编译dpdk前，需要设置RTE_SDK环境变量，以及"arch-machine-execenv-\
toolchain"组成的编译目标（target）。比如，要使用gcc为x86_64架构\
native处理器，linux环境编译dpdk，编译目录同target::
    
    export RTE_TARGET=x86_64-native-linuxapp-gcc
    cd $RTE_SDK
    make config T=$RTE_TARGET O=$RTE_TARGET

.. note:: RTE_TARGET实际上是dpdk编译系统的一个环境变量，应设置成
    arch-machine-execenv-toolchain形式，输出目录也应该设置此值，
    因为输出目录$(RTE_OUTPUT)默认是build，而编译外部程序时，链接
    dpdk的路径是RTE_SDK_BIN=$RTE_SDK/$RTE_TARGET, 如果RTE_OUTPUT
    和RTE_TARGET不一致，会发生找不到库的错误. 更多环境变量见X.X节

完成后，将在<RTE_SDK>生成mybuild子目录，其中包含对应此target的配置\
文件Makefile等。接着可以进行编译::

    cd $RTE_TARGET
    make

或者::

    make O=$RTE_TARGET

编译完成后，mybuild目录内容为::

    x86_64-native-linuxapp-gcc
    ├── .config  # 针对此target生成的配置文件
    ├── app      # 生成的自带应用程序 
    ├── build    # 编译期间的临时文件
    ├── include  # 头文件（链接）
    ├── kmod     # 生成的内核驱动
    ├── lib      # 生成的dpdk库文件
    └── Makefile # 生成的Makefile


编译外部程序
..............

假设外部程序源码在/home/zzq/helloworld目录。
在此目录下的Makefile中，需要包含dpdk的一些.mk文件，如\
$RTE_SDK/mk/rte.vas.mk，$RTE_SDK/mk/rte.app.mk等。具体见
`Building Your Own Application 
<http://dpdk.org/doc/guides/prog_guide/build_app.html>`_

首先确保设置了RTE_SDK和RTE_TARGET环境变量::

    export RTE_SDK=/home/zzq/dpdk-stable-16.07.2
    export RTE_TARGET=x86_64-native-linuxapp-gcc
    cd /home/zzq/helloworld
    make

编译完成后，生成的文件位于/home/zzq/helloworld/build目录。Makefile\
写法之类可以参考$RTE_SDK/examples目录中的各示例。


dpdk Makefile介绍
------------------

一般规则
..........

dpdk Makefile一般都是遵循以下方案：

#. 在开头include $(RTE_SDK)/mk/rte.vars.mk
#. 定义某些特定变量
#. 根据需求，include $(RTE_SDK)mk/rte.XYZ.mk，XYZ可能是app,lib,\
   extlib,obj等等，取决于要编译的目标文件类型
#. 包含用户自定义的规则和变量

以下是一个极简示例（取自examples/helloworld）::

    ifeq ($(RTE_SDK),)
    $(error "Please define RTE_SDK environment variable")
    endif

    # Default target, can be overriden by command line or environment
    RTE_TARGET ?= x86_64-native-linuxapp-gcc

    include $(RTE_SDK)/mk/rte.vars.mk

    # binary name
    APP = helloworld

    # all source are stored in SRCS-y
    SRCS-y := main.c

    CFLAGS += -O3
    CFLAGS += $(WERROR_FLAGS)

    include $(RTE_SDK)/mk/rte.extapp.mk

类型
.....

根据所包含的.mk文件的不同，Makefile有不同的角色。不可能用同一个\
Makefile编译库和应用程序，而必须得使用2个Makefile。

用户的Makefile都应包含rte.vars.mk。

应用程序
*********

以下Makefile用于生成二进制程序：

* rte.app.mk: dpdk包含的应用程序
* rte.extapp.mk: 外部应用程序
* rte.hostapp.mk: 编译dpdk所需的工具

库
***

* rte.lib.mk: dpdk库
* rte.extlib.mk: 外部库
* rte.hostlib.mk: dpdk中的host库

install
********

* rte.install.mk: 不编译任何东西，只用于创建链接或将文件拷贝到安装\
  目录

内核模块
*********

* rte.module.mk: 编译dpdk中的内核模块

objects
********

* rte.obj.mk: 将编译dpdk生成的多个.o文件合并成一个 
* rte.extobj.mk: 将编译外部代码生成的多个.o文件合并成一个 

杂项
*****

* rte.doc.mk: 编译dpdk文档
* rte.gnuconfigure.mk: 编译基于configure的应用程序
* rte.subdir.mk: 编译dpdk的多个子目录

常用变量
.........

* **RTE_SDK** dpdk源码主目录的绝对路径
* **RTE_SRCDIR** 源码根目录，编译dpdk时等于$RTE_SDK，编译外部程序\
  时指向外部程序源码的根目录
* **RTE_OUTPUT** 输出文件路径，默认是$(RTE_SDK)/build，但可以通过\
  命令行中的 ``O=`` 选项来指定
* **RTE_TARGET** 用于识别当前编译目标的字符串，格式是 ``arch-machine\
  -execenv-toolchain``
* **RTE_SDK_BIN** 引用$(RTE_SDK)/$(RTE_TARGET)。注意，编译基于dpdk\
  的外部程序时会通过$RTE_SDK_BIN来引用dpdk库，因此编译dpdk时，最好\
  通过O=$(RTE_TARGET)指定输出目录到$(RTE_TARGET)，而不是"build","mybuild"\
  等目录
* **RTE_ARCH** 定义CPU架构(i686,x86_64等)，它与CONFIG_RTE_ARCH值相同，\
  但没有两边的"号
* **RTE_MACHINE** 定义machine(native, armv8a等)，它与CONFIG_RTE_MACHINE\
  值相同，但没有两边的"号
* **RTE_TOOLCHAIN** 定义编译工具链(gcc,icc,clang等)，它与CONFIG_RTE_TOOLCHAIN\
  值相同，但没有两边的"号
* **RTE_EXEC_ENV** 定义执行环境(linuxapp,bsdapp等)，它与CONFIG_RTE_EXEC_ENV\
  值相同，但没有两边的"号
* **RTE_KERNELDIR** 包含用于编译内核模块的内核源码的绝对路径，默认\
  是/lib/modules/$(shell uname -r)/build，当目标机器也是编译机器时\
  是没问题的，否则要在目标机器上重新编译
* **RTE_DEVEL_BUILD** Stricter options (stop on warning). It \
  defaults to y in a git tree. 

仅可以在Makefile中设置的变量
.............................

* **VPATH** 源码搜索路径列表，RTE_SRCDIR默认会包含在其中
* **CFLAGS** C编译选项
* **LDFLAGS** 链接选项
* **ASFLAGS** 汇编选项
* **CPPFLAGS** C预处理器选项（仅用于汇编.S文件）
* **LDLIBS** 要链接的库列表
* **SRC-y** 源文件列表（.c,.S或.o），这些文件必须在VPATH中
* **INSTALL-y-$(INSTPATH)** 要安装到$(INSTPATH)中的文件列表，这些\
  文件必须在VPATH中，且会被拷贝到$(RTE_OUTPUT)/$(INSTPATH)
* **SYMLINK-y-$(INSTPATH)** 要安装到$(INSTPATH)中的文件列表，这些\
  文件必须在VPATH中，且会被符号链接到$(RTE_OUTPUT)/$(INSTPATH)
* **PREBUILD** 编译之前需要执行的动作
* **POSTBUILD** 主要编译之后需要执行的动作
* **PREINSTALL** 安装之前需要执行的动作 
* **POSTINSTALL**  安装之后需要执行的动作
* **PRECLEAN** 在清理之前需要执行的动作
* **POSTCLEAN** 在清理之后需要执行的动作
* **DEPDIR-y** 仅用于确定当前目录的编译是否依赖其他目录的编译，并行\
  编译需要此变量


仅可以在命令行中设置的变量
.............................

见 :ref:`root_makefile_help` 和 `External Application/Library Makefile Help \
<http://dpdk.org/doc/guides/prog_guide/ext_app_lib_make_help.html>`_ 。

* **WERROR_CFLAGS** 默认设置为依赖于编译器的某特定值，推荐用法如::
    
    CFLAGS += $(WERROR_CFLAGS)

可以在Makefile或命令行中设置的变量
...................................

* **CFLAGS_my_file.o** 额外的my_file.c的C编译选项
* **LDFLAGS_my_app** 额外的my_app的链接选项
* **EXTRA_CFLAGS** 编译时添加到CFLAGS之后
* **EXTRA_LDFLAGS** 链接时添加到LDFLAGS之后
* **EXTRA_LDLIBS** 链接时添加到LDLIBS之后
* **EXTRA_ASFLAGS** 汇编时添加到ASFLAGS之后
* **EXTRA_CPPFLAGS** 添加到CPPFLAGS之后

.. _root_makefile_help:

根Makefile介绍
---------------

dpdk编译系统提供了一个包含多个target的根Makefile，这些target包括\
config, build, test, install及其他。此Makefile应该是<RTE_SDK>/\
mk/rte.sdkroot.mk。

config target
..............

编译此target必须通过 ``T=`` 指定target名称，所有可用的target名称\
见<RTE_SDK>/config/defconfig_XXX。还可以通过 ``O=`` 指定输出目录，\
默认目录是build。示例::

    export RTE_TARGET=x86_64-native-linuxapp-gcc
    make config T=$RTE_TARGET O=$RTE_TARGET
    
完成后，将在输出目录创建配置文件和Makefile等。

build target
.............

build target也支持输出目录的可选项，即 ``O=`` ，默认目录是build。

* all, 或仅make 在make config创建的输出目录中编译dpdk
* clean 清理
* %_sub 仅编译某个子目录。例： ``make lib/librte_eal_sub O=$RTE_TARGET``  
* %_clean 仅清理某个子目录。例： ``make lib/librte_eal_clean O=$RTE_TARGET``

以上target都可以带 ``O=`` 选项。

install target
...............

安装编译后的文件。例： ``make install DESTDIR=myinstall prefix=/usr``

注意需要指定DESTDIR。

test target
.............

启动自动测试（需要在编译前将config/common_base中的 CONFIG_RTE_APP_TEST\
改为y)。例： ``make test O=$RTE_TARGET``

文档target
...........

* doc 生成API和指南文档
* doc-api-html 生成html格式的API文档(基于Doxygen)
* doc-guides-html 生成html格式的指南文档
* doc-guides-pdf 生成pdf格式的指南文档 

deps target
............

* depdirs 此target由make config隐式调用，用户一般不需要显式调用，\
  除非Makefile中的DEPDIRS-y变量发生了变化。执行后会生成$RTE_OUTPUT\
  /.depdirs文件。
* depgraph 此命令生成依赖项的dot graph，可用于调试循环引用问题。

    示例::
        
        make depgraph O=$RTE_TARGET > /tmp/graph.dot && dotty /tmp/graph.dot

有关dot和dotty可参考： http://hoagland.org/Dot.html

杂项target
............

* help 显示编译帮助

其他命令行变量
...............

* **V=** verbose模式（V=y），显示完整的编译命令，包括一些中间命令
* **D=** 依赖性调试
* EXTRA_CFLAGS=, EXTRA_LDFLAGS=, EXTRA_LDLIBS=, EXTRA_ASFLAGS=, EXTRA_CPPFLAGS=
* **CROSS=** Specify a cross toolchain header that will prefix all \
  gcc/binutils applications. This only works when using gcc.

编译Debug版本
..............

修改EXTRA_CFLAGS环境变量即可::

    export EXTRA_CFLAGS='-O0 -g'


ABI管理
----------

什么是ABI
..........

XXX

DPDK ABI策略
.............

XXX 见 `DPDK ABI policy <http://dpdk.org/doc/guides/contributing/versioning.html#the-dpdk-abi-policy>`_

ABI宏及示例
.............

库中的符号（symbol）不仅提供了API，还提供了函数名、返回值和参数\
这些调用约定。有时候在新版本中需要修改这些函数，此时，应在一段时间\
内与旧版本保持向后兼容。

头文件 ``lib/librte_compat/rte_compat.h`` 提供了向后兼容所需要的\
宏，这些宏与文件 ``rte_<library>_version.map`` 结合使用，以支持在\
同一个动态库中提供同一符号的多个版本，这样一来，依赖此动态库的旧\
版本的程序就不需要重新编译。

这些宏包括：

* ``VERSION_SYMBOL(b, e, n)`` 创建一个符号版本表项，绑定版本化符号\
  ``b@DPDK_n`` 到内部函数 ``b_e``
* ``BIND_DEFAULT_SYMBOL(b, e, n)`` 创建一个符号版本项，指导链接器\
  将符号 ``b`` 绑定到内部函数 ``b_e``
* ``MAP_STATIC_SYMBOL(f, p)`` 声明原型 ``f`` ，并将它映射到完整函数
  ``p`` 。这样的话，若某符号版本化（有多个版本），它仍可被映射回\
  公共符号名

下面是一些实例。

更新API
*********

假设老版本函数为

.. code-block:: c

    /*
     * Create an acl context object for apps to manipulate
     */
    struct rte_acl_ctx *
    rte_acl_create(const struct rte_acl_param *param)
    {
        ...
    }

现在需要修改此API，给acl context设置debug标志：

.. code-block:: c

    /*
     * Create an acl context object for apps to manipulate
     */
    struct rte_acl_ctx *
    rte_acl_create(const struct rte_acl_param *param, int debug)
    {
        ...
    }

如果不使用ABI宏直接编译，那么链接此动态库的旧程序将不得不修改代码。\
而使用ABI宏可以让多个函数映射到同一符号，让新版动态库保持向后兼容，\
旧程序不需要修改代码。

首先需要修改此库对应的.map文件，原本的文件内容如下（rte_acl_version.map）::

    DPDK_2.0 {
        global:

        rte_acl_add_rules;
        rte_acl_build;
        rte_acl_classify;
        rte_acl_classify_alg;
        rte_acl_classify_scalar;
        rte_acl_create;
        rte_acl_dump;
        rte_acl_find_existing;
        rte_acl_free;
        rte_acl_ipv4vlan_add_rules;
        rte_acl_ipv4vlan_build;
        rte_acl_list_dump;
        rte_acl_reset;
        rte_acl_reset_rules;
        rte_acl_set_ctx_classify;

        local: *;
    };

需要修改为::

    DPDK_2.0 {
        global:

        rte_acl_add_rules;
        rte_acl_build;
        rte_acl_classify;
        rte_acl_classify_alg;
        rte_acl_classify_scalar;
        rte_acl_create;
        rte_acl_dump;
        rte_acl_find_existing;
        rte_acl_free;
        rte_acl_ipv4vlan_add_rules;
        rte_acl_ipv4vlan_build;
        rte_acl_list_dump;
        rte_acl_reset;
        rte_acl_reset_rules;
        rte_acl_set_ctx_classify;

        local: *;
    };

    DPDK_2.1 {
        global:
        rte_acl_create;

    } DPDK_2.0;

增加的部分告诉链接器新增了一个版本节点（version node）DPDK_2.1，\
它包含符号rte_acl_create，并继承节点DPDK_2.0的符号。当DPDK被编译\
为动态库时，此列表会直接翻译成导出符号表。

接下来，需要在代码中指定哪个版本映射哪个rte_acl_create。首先，需\
要改变原函数的定义，使其命名唯一且不和公用符号名冲突::

    struct rte_acl_ctx *
    -rte_acl_create(const struct rte_acl_param *param)
    +rte_acl_create_v20(const struct rte_acl_param *param)
    {
        size_t sz;
        struct rte_acl_ctx *ctx;
        ...

注意符号的基本名称（指的应该是rte_acl_create_v20中的rte_acl_create\
前缀）保持不变，这对用于符号版本化的宏有好处。接着，把新符号名映射\
到版本节点2.0中的原始符号名。紧接着此函数的定义，添加一行代码::

    VERSION_SYMBOL(rte_acl_create, _v20, 2.0);

记得把rte_compat.h头文件包含到.c源码文件中。上面的宏指导链接器创建\
一个新符号 ``rte_acl_create@DPDK_2.0`` ，它匹配老版本中的符号，\
但指向上一步我们新命名的函数。到此为止，已经把原始的rte_acl_create\
符号映射到原始函数，只不过改了一个新名字。

然后，需要创建符号的2.1版本。实现的新版函数使用一个新的函数名，\
带不同的后缀::

    struct rte_acl_ctx *
    rte_acl_create_v21(const struct rte_acl_param *param, int debug);
    {
        struct rte_acl_ctx *ctx = rte_acl_create_v20(param);

        ctx->debug = debug;

        return ctx;
    }

同样地，需要把此函数映射到符号 ``rte_acl_create@DPDK_2.1`` 。修改\
此API原型所在的头文件，添加版本宏，通知调用此函数的程序在重新链接\
时，默认的rte_acl_create符号应指向新版本函数::

    struct rte_acl_ctx *
    -rte_acl_create(const struct rte_acl_param *param);
    +rte_acl_create(const struct rte_acl_param *param, int debug);
    +BIND_DEFAULT_SYMBOL(rte_acl_create, _v21, 2.1);

BIND_DEFAULT_SYMBOL宏显式地告诉包含此头文件的程序链接函数rte_acl_create_v21\
并对其应用版本节点DPDK_2.1。这种方法比重新实现原符号名更显式和灵活。

最后，以上步骤只适用于动态链接，静态链接并没有版本化的概念，以上\
步骤会导致静态编译时因为找不到 ``rte_acl_create`` 这个符号而失败。

要解决此问题，可以用MAP_STATIC_SYMBOL宏将指定函数映射到公用符号。\
一般来说要映射的特定函数就是最新版本的函数。因此，回到函数定义所在\
的.c文件，在 ``rte_acl_create_v21`` 定义之后，紧接着添加宏代码::

    struct rte_acl_ctx *
    rte_acl_create_v21(const struct rte_acl_param *param, int debug)
    {
         ...
    }
    MAP_STATIC_SYMBOL(struct rte_acl_ctx *rte_acl_create(const struct \
    rte_acl_param *param, int debug), rte_acl_create_v21);

这告诉编译器在编译静态库时，把对符号 ``rte_acl_create`` 的调用链接\
到 ``rte_acl_create_v21`` 。

完成后，编译之后的新版本动态库，将含有2个版本的rte_acl_create，\
其中DPDK_2.0旧版本用于之前编译的用户程序，DPDK_2.1新版本用于未来\
编译的用户程序。

验证ABI
........

<RTE_SDK>/scripts目录中包含一个名为 ``validate-abi.sh`` 的脚本工具，
可用于验证DPDK ABI（基于Linux ABI Compliance Checker）。

详见：
http://dpdk.org/doc/guides/contributing/versioning.html#running-the-abi-validator


实例：将v16.07.2编译为单个动态库
---------------------------------

dpdk从某个版本（可能是16.04）起，编译系统取消了对“编译成一个动态库”\
的支持。如果还需要将dpdk（除内核驱动以外）编译成单个的.so文件，\
需要对编译系统的某些文件做一些修改。

.. note:: 以下描述只针对编译单个动态库的需求，如果有其他编译需求,
    请参考相关文档

config/common_base
....................

修改::

    CONFIG_RTE_BUILD_SHARED_LIB=y

添加::

    CONFIG_RTE_BUILD_COMBINE_LIBS=y
    CONFIG_RTE_LIBNAME=intel_dpdk

mk/rte.lib.mk
...............

在 ``-include .$(LIB).cmd`` 之前，添加::

    ifeq ($(CONFIG_RTE_BUILD_SHARED_LIB),n)
    O_TO_C = $(AR) crus $(LIB_ONE) $(OBJS-y)
    O_TO_C_STR = $(subst ','\'',$(O_TO_C)) #'# fix syntax highlight
    O_TO_C_DISP = $(if $(V),"$(O_TO_C_STR)","  AR_C $(@)")
    O_TO_C_DO = @set -e; \
        $(lib_dir) \
        $(copy_obj)
    else
    O_TO_C = $(LD) -shared $(OBJS-y) -o $(LIB_ONE)
    O_TO_C_STR = $(subst ','\'',$(O_TO_C)) #'# fix syntax highlight
    O_TO_C_DISP = $(if $(V),"$(O_TO_C_STR)","  LD_C $(@)")
    O_TO_C_DO = @set -e; \
        $(lib_dir) \
        $(copy_obj)
    endif

    copy_obj = cp -f $(OBJS-y) $(RTE_OUTPUT)/build/lib;
    lib_dir = [ -d $(RTE_OUTPUT)/lib ] || mkdir -p $(RTE_OUTPUT)/lib;

$(copy_obj)把编译成的.o文件拷贝到$(RTE_OUTPUT)/build/lib目录。

在 ``ifeq ($(CONFIG_RTE_BUILD_SHARED_LIB), y)`` 判断语句块的第一个\
else前，添加::

    ifeq ($(CONFIG_RTE_BUILD_COMBINE_LIBS),y)
        $(if $(or \
            $(file_missing),\
            $(call cmdline_changed,$(O_TO_C_STR)),\
            $(depfile_missing),\
            $(depfile_newer)),\
            $(O_TO_C_DO))
    endif
   
在这语句块结束的最后一个 ``endif`` 前，添加::

    ifeq ($(CONFIG_RTE_BUILD_COMBINE_LIBS),y)
        $(if $(or \
            $(file_missing),\
            $(call cmdline_changed,$(O_TO_C_STR)),\
            $(depfile_missing),\
            $(depfile_newer)),\
            $(O_TO_C_DO))
    endif

mk/rte.combinedlib.mk
......................

首先要包含rte.build-pre.mk::

    include $(RTE_SDK)/mk/internal/rte.build-pre.mk

修改输出库文件名，删除::

    RTE_LIBNAME := dpdk

因为库名称在common_base文件中配置了。修改::
    
    COMBINEDLIB := lib$(CONFIG_RTE_LIBNAME)$(EXT)
    
在LIBS后面添加::

    OBJS = $(wildcard $(RTE_OUTPUT)/build/lib/*.o)
    CPU_LDFLAGS += --version-script=$(SRCDIR)/lib/libdpdk.map

之后需要链接OBJS，libdpdk.map是上文说到的版本控制脚本。

在 ``all: FORCE`` 前添加::

    ifeq ($(LINK_USING_CC),1)
    # Override the definition of LD here, since we're linking with CC
    LD := $(CC) $(CPU_CFLAGS)
    O_TO_S = $(LD) $(call linkerprefix,$(CPU_LDFLAGS)) \
        -shared $(OBJS) -o $(RTE_OUTPUT)/lib/$(COMBINEDLIB)
    else
    O_TO_S = $(LD) $(CPU_LDFLAGS) \
        -shared $(OBJS) -o $(RTE_OUTPUT)/lib/$(COMBINEDLIB)
    endif

    O_TO_S_STR = $(subst ','\'',$(O_TO_S)) #'# fix syntax highlight
    O_TO_S_DISP = $(if $(V),"$(O_TO_S_STR)","  LD $(@)")
    O_TO_S_CMD = "cmd_$@ = $(O_TO_S_STR)"
    O_TO_S_DO = @set -e; \
        echo $(O_TO_S_DISP); \
        $(O_TO_S)
    
修改all目标( ``all: FORCE`` 下一行）::

    all: FORCE
        $(O_TO_S_DO)


mk/rte.app.mk
..............

此文件改动较大。首先在 ``_LDLIBS-y += -L$(RTE_SDK_BIN)/lib`` 之后\
添加::

    _LDLIBS-$(CONFIG_RTE_BUILD_COMBINE_LIBS)    += -l$(CONFIG_RTE_LIBNAME)
    _LDLIBS-y += -lm
    _LDLIBS-y += -lpcap

之后，在 ``_LDLIBS-y += --whole-archive`` 之前，修改为::


    ifeq ($(CONFIG_RTE_BUILD_COMBINE_LIBS),n)

    _LDLIBS-$(CONFIG_RTE_LIBRTE_DISTRIBUTOR)    += -lrte_distributor
    _LDLIBS-$(CONFIG_RTE_LIBRTE_REORDER)        += -lrte_reorder
    ifeq ($(CONFIG_RTE_EXEC_ENV_LINUXAPP),y)
    _LDLIBS-$(CONFIG_RTE_LIBRTE_KNI)            += -lrte_kni
    _LDLIBS-$(CONFIG_RTE_LIBRTE_IVSHMEM)        += -lrte_ivshmem
    endif

    _LDLIBS-$(CONFIG_RTE_LIBRTE_PIPELINE)       += -lrte_pipeline
    _LDLIBS-$(CONFIG_RTE_LIBRTE_TABLE)          += -lrte_table
    _LDLIBS-$(CONFIG_RTE_LIBRTE_PORT)           += -lrte_port
    _LDLIBS-$(CONFIG_RTE_LIBRTE_TIMER)          += -lrte_timer
    _LDLIBS-$(CONFIG_RTE_LIBRTE_HASH)           += -lrte_hash
    _LDLIBS-$(CONFIG_RTE_LIBRTE_JOBSTATS)       += -lrte_jobstats
    _LDLIBS-$(CONFIG_RTE_LIBRTE_LPM)            += -lrte_lpm
    _LDLIBS-$(CONFIG_RTE_LIBRTE_POWER)          += -lrte_power
    # librte_acl needs --whole-archive because of weak functions
    _LDLIBS-$(CONFIG_RTE_LIBRTE_ACL)            += --whole-archive
    _LDLIBS-$(CONFIG_RTE_LIBRTE_ACL)            += -lrte_acl
    _LDLIBS-$(CONFIG_RTE_LIBRTE_ACL)            += --no-whole-archive
    _LDLIBS-$(CONFIG_RTE_LIBRTE_PDUMP)          += -lrte_pdump
    _LDLIBS-$(CONFIG_RTE_LIBRTE_IP_FRAG)        += -lrte_ip_frag
    _LDLIBS-$(CONFIG_RTE_LIBRTE_METER)          += -lrte_meter
    _LDLIBS-$(CONFIG_RTE_LIBRTE_SCHED)          += -lrte_sched
    _LDLIBS-$(CONFIG_RTE_LIBRTE_VHOST)          += -lrte_vhost

    endif # ! CONFIG_RTE_BUILD_COMBINE_LIBS

之后，在 ``_LDLIBS-$(CONFIG_RTE_LIBRTE_KVARGS) += -lrte_kvargs`` 
之前，添加::

    ifeq ($(CONFIG_RTE_BUILD_SHARED_LIB),n)
    # The static libraries do not know their dependencies.
    # So linking with static library requires explicit dependencies.
    _LDLIBS-$(CONFIG_RTE_LIBRTE_EAL)            += -lrt
    _LDLIBS-$(CONFIG_RTE_LIBRTE_SCHED)          += -lm
    _LDLIBS-$(CONFIG_RTE_LIBRTE_SCHED)          += -lrt
    _LDLIBS-$(CONFIG_RTE_LIBRTE_METER)          += -lm
    ifeq ($(CONFIG_RTE_LIBRTE_VHOST_NUMA),y)
    _LDLIBS-$(CONFIG_RTE_LIBRTE_VHOST)          += -lnuma
    endif
    ifeq ($(CONFIG_RTE_LIBRTE_VHOST_USER),n)
    _LDLIBS-$(CONFIG_RTE_LIBRTE_VHOST)          += -lfuse
    endif
    _LDLIBS-$(CONFIG_RTE_PORT_PCAP)             += -lpcap
    endif # !CONFIG_RTE_BUILD_SHARED_LIB


    _LDLIBS-y += --start-group
    ifeq ($(CONFIG_RTE_BUILD_COMBINE_LIBS),n)

之后，在 ``endif # CONFIG_RTE_LIBRTE_CRYPTODEV`` 之后，
``_LDLIBS-y += --no-whole-archive`` 之前，添加::

    endif # !CONFIG_RTE_BUILD_SHARED_LIB
    endif # ! CONFIG_RTE_BUILD_COMBINE_LIBS

    _LDLIBS-y += $(EXECENV_LDLIBS)
    _LDLIBS-y += --end-group

删除下面的::

    _LDLIBS-y += $(EXECENV_LDLIBS)

最后，将 ``exe2cmd = $(strip $(call dotfile,$(patsubst %,%.cmd,$(1))))``
之后，``O_TO_EXE_STR = $(subst ','\'',$(O_TO_EXE)) #'# fix syntax highlight``
之前的内容，修改为::

    ifeq ($(LINK_USING_CC),1)
    override EXTRA_LDFLAGS := $(call linkerprefix,$(EXTRA_LDFLAGS))
    O_TO_EXE = $(CC) $(CFLAGS) $(LDFLAGS_$(@)) \
        -Wl,-Map=$(@).map,--cref -o $@ $(OBJS-y) $(call linkerprefix,$(LDFLAGS)) \
        $(EXTRA_LDFLAGS) $(call linkerprefix,$(LDLIBS))
    else
    O_TO_EXE = $(LD) $(LDFLAGS) $(LDFLAGS_$(@)) $(EXTRA_LDFLAGS) \
        -Map=$(@).map --cref -o $@ $(OBJS-y) $(LDLIBS)
    endif

libdpdk.map
............

最后，需要提供一个版本控制脚本libdpdk.map，将它放在$RTE_SDK/lib目录中，
该文件内容为::

    DPDK_2.0 {

    };

    DPDK_2.1 {

    } DPDK_2.0;

    DPDK_2.2 {

    } DPDK_2.1;

    DPDK_16.04 {

    } DPDK_2.2;

    DPDK_16.07 {

    } DPDK_16.04;


附： patch文件 ``<promise_git>/pkg/dpdk16.07_patch``


参考
----

.. [dpdk_prog_guide_source_org] `DPDK Programmer's Guide - Source Organization \
    <http://dpdk.org/doc/guides/prog_guide/source_org.html>`_

.. [dpdk_prog_guide_build_system] `DPDK Programmer's Guide - Development Kit Build System \
    <http://dpdk.org/doc/guides/prog_guide/dev_kit_build_system.html>`_

.. [dpdk_prog_guide_makefile_help] `DPDK Programmer's Guide - Development Kit Root Makefile Help \
    <http://dpdk.org/doc/guides/prog_guide/dev_kit_root_make_help.html>`_

.. [dpdk_contr_abi] `DPDK Contributor's Guidelines - Managing ABI updates \
    <http://dpdk.org/doc/guides/contributing/versioning.html>`_

