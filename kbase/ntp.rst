
NTP服务器客户端配置
===================

目的
----

NTP是网络时间协议，用来提供高精准度的时间校正，可以通过认证加密方式防\
止网络攻击。本文档主要描述NTP服务器/客户端配置。

读者
----

本文档适用于本模块的开发人员，也可供本模块的维护人员和Sdpi项目的其他\
开发人员参考。

名词
----

专有名词
^^^^^^^^

NTP：network time protocol，网络时间协议。

服务器/客户端配置
-----------------

服务器端配置
^^^^^^^^^^^^

1. 通过rpm -qa | grep ntp命令检查ntp相关安装包是否存在，如果不存在需要通过yum install或者rpm安装。


2. NTP服务器的主要配置文件为/etc/ntp/ntp.conf。要求客户端必须进行认证需要如下配置::

    [root@vmware ~]# more /etc/ntp.conf

    # Hosts on local network are less restricted.

    # restrict 192.168.0.0 mask 255.255.255.0  notrust	

其中notrust，表示客户端必须进行认证; 允许192.168.0.0/24子网内的主机同步时间。


3. 设置信任的密钥序号需要如下配置::

    [root@vmware ~]# more /etc/ntp.conf

    # Key file containing the keys and key identifiers used when operating

    # with symmetric key cryptography. 

    keys /etc/ntp/keys

    # Specify the key identifiers which are trusted.

    #trustedkey 4 8 42

    trustedkey 123

其中，keys /etc/ntp/keys存放密钥信息，包括id，type，key; trustedkey 123表示NTP服务器指定的是id为123的密钥。


4. 信任KEY的设置放在/etc/ntp/keys里。如下是/etc/ntp/keys的配置::

    [root@vmware ~]# more /etc/ntp/keys 

    # For more information about this file, see the man page ntp_auth(5).

    #

    # id    type    key

     123   M       aaa

     124   M       bbb

id对应/etc/ntp.conf的trustedkey对应的id; type表示加密类型，M为MD5加密，key是加密密钥。



客户端配置
^^^^^^^^^^

1. 客户端需要安装ntpdate，通过rpm -qa | grep ntpdate命令检查ntpdate\
   相关安装包是否存在，如果不存在可以通过yum install或者rpm安装。


2. 客户端只需要配置信任KEY的配置文件，这个可以随意指定文件位置，\
   但文件格式和服务器的/etc/ntp/keys相同，如::

    [root@vmware ~]# more ntp.keys 

    # For more information about this file, see the man page ntp_auth(5).

    #

    # id    type    key

     123   M       aaa



同步时间
--------

同步时间命令如下::

    [root@vmware ~]#ntpdate -a 123 -k /etc/ntp.keys x.x.x.x

-a 123 表示客户端认证密钥id为123，-k /etc/ntp.keys是密钥信息存放的文件位置，x.x.x.x是NTP服务器IP地址。

