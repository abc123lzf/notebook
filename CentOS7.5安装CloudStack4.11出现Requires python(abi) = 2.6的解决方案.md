---
title: CentOS7.5安装CloudStack4.11出现Requires python(abi) = 2.6的解决方案
date: 2018-11-14 14:55:48
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/84066571]( https://blog.csdn.net/abc123lzf/article/details/84066571)   
  #### []()错误描述

 
```
[root@manage ~]# yum install cloudstack-management
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: repos.lax.quadranet.com
 * extras: mirror.linuxfix.com
 * updates: mirror.linuxfix.com
base                                                                                                                              | 3.6 kB  00:00:00     
cloudstack                                                                                                                        | 3.2 kB  00:00:00     
extras                                                                                                                            | 3.4 kB  00:00:00     
mysql-connectors-community                                                                                                        | 2.5 kB  00:00:00     
mysql-tools-community                                                                                                             | 2.5 kB  00:00:00     
mysql57-community                                                                                                                 | 2.5 kB  00:00:00     
updates                                                                                                                           | 3.4 kB  00:00:00     
cloudstack/primary_db                                                                                                             | 6.8 kB  00:00:00     
正在解决依赖关系
--> 正在检查事务
---> 软件包 cloudstack-management.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 cloudstack-common = 4.11.1.0，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 java-1.8.0-openjdk，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 nfs-utils，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 mysql-connector-python，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 jakarta-commons-daemon，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 ipmitool，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 python-paramiko，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 mkisofs，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 jsvc，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 unzip，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 jakarta-commons-daemon-jsvc，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 mysql-connector-java，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 bzip2，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 /sbin/mount.nfs，它被软件包 cloudstack-management-4.11.1.0-1.el6.x86_64 需要
--> 正在检查事务
---> 软件包 apache-commons-daemon.x86_64.0.1.0.13-7.el7 将被 安装
--> 正在处理依赖关系 jpackage-utils，它被软件包 apache-commons-daemon-1.0.13-7.el7.x86_64 需要
---> 软件包 apache-commons-daemon-jsvc.x86_64.0.1.0.13-7.el7 将被 安装
---> 软件包 bzip2.x86_64.0.1.0.6-13.el7 将被 安装
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 python-netaddr，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 libuuid.so.1，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
--> 正在处理依赖关系 libc.so.6(GLIBC_2.3)，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 genisoimage.x86_64.0.1.1.11-23.el7 将被 安装
--> 正在处理依赖关系 libusal = 1.1.11-23.el7，它被软件包 genisoimage-1.1.11-23.el7.x86_64 需要
--> 正在处理依赖关系 libusal.so.0()(64bit)，它被软件包 genisoimage-1.1.11-23.el7.x86_64 需要
--> 正在处理依赖关系 librols.so.0()(64bit)，它被软件包 genisoimage-1.1.11-23.el7.x86_64 需要
---> 软件包 ipmitool.x86_64.0.1.8.18-7.el7 将被 安装
--> 正在处理依赖关系 OpenIPMI-modalias，它被软件包 ipmitool-1.8.18-7.el7.x86_64 需要
---> 软件包 java-1.8.0-openjdk.x86_64.1.1.8.0.191.b12-0.el7_5 将被 安装
--> 正在处理依赖关系 java-1.8.0-openjdk-headless(x86-64) = 1:1.8.0.191.b12-0.el7_5，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 xorg-x11-fonts-Type1，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libpng15.so.15(PNG15_0)(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjvm.so(SUNWprivate_1.1)(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjpeg.so.62(LIBJPEG_6.2)(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjava.so(SUNWprivate_1.1)(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 fontconfig(x86-64)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libpng15.so.15()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjvm.so()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjpeg.so.62()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libjava.so()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libgif.so.4()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libXtst.so.6()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libXrender.so.1()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libXi.so.6()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libXext.so.6()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libXcomposite.so.1()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 libX11.so.6()(64bit)，它被软件包 1:java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64 需要
---> 软件包 mysql-connector-java.noarch.1.5.1.25-3.el7 将被 安装
--> 正在处理依赖关系 jta >= 1.0，它被软件包 1:mysql-connector-java-5.1.25-3.el7.noarch 需要
--> 正在处理依赖关系 slf4j，它被软件包 1:mysql-connector-java-5.1.25-3.el7.noarch 需要
---> 软件包 mysql-connector-python.x86_64.0.8.0.13-1.el7 将被 安装
---> 软件包 nfs-utils.x86_64.1.1.3.0-0.54.el7 将被 安装
--> 正在处理依赖关系 libtirpc >= 0.2.4-0.7，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 gssproxy >= 0.7.0-3，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 rpcbind，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 quota，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 libnfsidmap，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 libevent，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 keyutils，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 libtirpc.so.1()(64bit)，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 libnfsidmap.so.0()(64bit)，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
--> 正在处理依赖关系 libevent-2.0.so.5()(64bit)，它被软件包 1:nfs-utils-1.3.0-0.54.el7.x86_64 需要
---> 软件包 python-paramiko.noarch.0.2.1.1-4.el7 将被 安装
--> 正在处理依赖关系 python2-pyasn1，它被软件包 python-paramiko-2.1.1-4.el7.noarch 需要
--> 正在处理依赖关系 python-cryptography，它被软件包 python-paramiko-2.1.1-4.el7.noarch 需要
---> 软件包 unzip.x86_64.0.6.0-19.el7 将被 安装
--> 正在检查事务
---> 软件包 OpenIPMI-modalias.x86_64.0.2.0.23-2.el7 将被 安装
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 fontconfig.x86_64.0.2.10.95-11.el7 将被 安装
--> 正在处理依赖关系 fontpackages-filesystem，它被软件包 fontconfig-2.10.95-11.el7.x86_64 需要
--> 正在处理依赖关系 font(:lang=en)，它被软件包 fontconfig-2.10.95-11.el7.x86_64 需要
---> 软件包 geronimo-jta.noarch.0.1.1.1-17.el7 将被 安装
---> 软件包 giflib.x86_64.0.4.1.6-9.el7 将被 安装
--> 正在处理依赖关系 libSM.so.6()(64bit)，它被软件包 giflib-4.1.6-9.el7.x86_64 需要
--> 正在处理依赖关系 libICE.so.6()(64bit)，它被软件包 giflib-4.1.6-9.el7.x86_64 需要
---> 软件包 glibc.i686.0.2.17-222.el7 将被 安装
--> 正在处理依赖关系 libfreebl3.so(NSSRAWHASH_3.12.3)，它被软件包 glibc-2.17-222.el7.i686 需要
--> 正在处理依赖关系 libfreebl3.so，它被软件包 glibc-2.17-222.el7.i686 需要
---> 软件包 gssproxy.x86_64.0.0.7.0-17.el7 将被 安装
--> 正在处理依赖关系 libini_config >= 1.3.1-28，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libverto-module-base，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libref_array.so.1(REF_ARRAY_0.1.1)(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libini_config.so.3(INI_CONFIG_1.2.0)(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libini_config.so.3(INI_CONFIG_1.1.0)(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libref_array.so.1()(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libini_config.so.3()(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libcollection.so.2()(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
--> 正在处理依赖关系 libbasicobjects.so.0()(64bit)，它被软件包 gssproxy-0.7.0-17.el7.x86_64 需要
---> 软件包 java-1.8.0-openjdk-headless.x86_64.1.1.8.0.191.b12-0.el7_5 将被 安装
--> 正在处理依赖关系 tzdata-java >= 2015d，它被软件包 1:java-1.8.0-openjdk-headless-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 nss-softokn(x86-64) >= 3.36.0，它被软件包 1:java-1.8.0-openjdk-headless-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 nss(x86-64) >= 3.36.0，它被软件包 1:java-1.8.0-openjdk-headless-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 copy-jdk-configs >= 2.2，它被软件包 1:java-1.8.0-openjdk-headless-1.8.0.191.b12-0.el7_5.x86_64 需要
--> 正在处理依赖关系 lksctp-tools(x86-64)，它被软件包 1:java-1.8.0-openjdk-headless-1.8.0.191.b12-0.el7_5.x86_64 需要
---> 软件包 javapackages-tools.noarch.0.3.4.1-11.el7 将被 安装
--> 正在处理依赖关系 python-javapackages = 3.4.1-11.el7，它被软件包 javapackages-tools-3.4.1-11.el7.noarch 需要
---> 软件包 keyutils.x86_64.0.1.5.8-3.el7 将被 安装
---> 软件包 libX11.x86_64.0.1.6.5-1.el7 将被 安装
--> 正在处理依赖关系 libX11-common >= 1.6.5-1.el7，它被软件包 libX11-1.6.5-1.el7.x86_64 需要
--> 正在处理依赖关系 libxcb.so.1()(64bit)，它被软件包 libX11-1.6.5-1.el7.x86_64 需要
---> 软件包 libXcomposite.x86_64.0.0.4.4-4.1.el7 将被 安装
---> 软件包 libXext.x86_64.0.1.3.3-3.el7 将被 安装
---> 软件包 libXi.x86_64.0.1.7.9-1.el7 将被 安装
---> 软件包 libXrender.x86_64.0.0.9.10-1.el7 将被 安装
---> 软件包 libXtst.x86_64.0.1.2.3-1.el7 将被 安装
---> 软件包 libevent.x86_64.0.2.0.21-4.el7 将被 安装
---> 软件包 libjpeg-turbo.x86_64.0.1.2.90-5.el7 将被 安装
---> 软件包 libnfsidmap.x86_64.0.0.25-19.el7 将被 安装
---> 软件包 libpng.x86_64.2.1.5.13-7.el7_2 将被 安装
---> 软件包 libtirpc.x86_64.0.0.2.4-0.10.el7 将被 安装
---> 软件包 libusal.x86_64.0.1.1.11-23.el7 将被 安装
---> 软件包 libuuid.x86_64.0.2.23.2-52.el7 将被 升级
--> 正在处理依赖关系 libuuid = 2.23.2-52.el7，它被软件包 libblkid-2.23.2-52.el7.x86_64 需要
--> 正在处理依赖关系 libuuid = 2.23.2-52.el7，它被软件包 util-linux-2.23.2-52.el7.x86_64 需要
--> 正在处理依赖关系 libuuid = 2.23.2-52.el7，它被软件包 libmount-2.23.2-52.el7.x86_64 需要
---> 软件包 libuuid.i686.0.2.23.2-52.el7_5.1 将被 安装
---> 软件包 libuuid.x86_64.0.2.23.2-52.el7_5.1 将被 更新
---> 软件包 python-netaddr.noarch.0.0.7.5-9.el7 将被 安装
---> 软件包 python2-cryptography.x86_64.0.1.7.2-2.el7 将被 安装
--> 正在处理依赖关系 python-six >= 1.4.1，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
--> 正在处理依赖关系 python-idna >= 2.0，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
--> 正在处理依赖关系 python-cffi >= 1.4.1，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
--> 正在处理依赖关系 python-setuptools，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
--> 正在处理依赖关系 python-ipaddress，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
--> 正在处理依赖关系 python-enum34，它被软件包 python2-cryptography-1.7.2-2.el7.x86_64 需要
---> 软件包 python2-pyasn1.noarch.0.0.1.9-7.el7 将被 安装
---> 软件包 quota.x86_64.1.4.01-17.el7 将被 安装
--> 正在处理依赖关系 quota-nls = 1:4.01-17.el7，它被软件包 1:quota-4.01-17.el7.x86_64 需要
--> 正在处理依赖关系 tcp_wrappers，它被软件包 1:quota-4.01-17.el7.x86_64 需要
---> 软件包 rpcbind.x86_64.0.0.2.0-44.el7 将被 安装
---> 软件包 slf4j.noarch.0.1.7.4-4.el7_4 将被 安装
--> 正在处理依赖关系 mvn(log4j:log4j)，它被软件包 slf4j-1.7.4-4.el7_4.noarch 需要
--> 正在处理依赖关系 mvn(javassist:javassist)，它被软件包 slf4j-1.7.4-4.el7_4.noarch 需要
--> 正在处理依赖关系 mvn(commons-logging:commons-logging)，它被软件包 slf4j-1.7.4-4.el7_4.noarch 需要
--> 正在处理依赖关系 mvn(commons-lang:commons-lang)，它被软件包 slf4j-1.7.4-4.el7_4.noarch 需要
--> 正在处理依赖关系 mvn(ch.qos.cal10n:cal10n-api)，它被软件包 slf4j-1.7.4-4.el7_4.noarch 需要
---> 软件包 xorg-x11-fonts-Type1.noarch.0.7.5-9.el7 将被 安装
--> 正在处理依赖关系 ttmkfdir，它被软件包 xorg-x11-fonts-Type1-7.5-9.el7.noarch 需要
--> 正在处理依赖关系 ttmkfdir，它被软件包 xorg-x11-fonts-Type1-7.5-9.el7.noarch 需要
--> 正在处理依赖关系 mkfontdir，它被软件包 xorg-x11-fonts-Type1-7.5-9.el7.noarch 需要
--> 正在处理依赖关系 mkfontdir，它被软件包 xorg-x11-fonts-Type1-7.5-9.el7.noarch 需要
--> 正在检查事务
---> 软件包 apache-commons-lang.noarch.0.2.6-15.el7 将被 安装
---> 软件包 apache-commons-logging.noarch.0.1.1.2-7.el7 将被 安装
--> 正在处理依赖关系 mvn(logkit:logkit)，它被软件包 apache-commons-logging-1.1.2-7.el7.noarch 需要
--> 正在处理依赖关系 mvn(avalon-framework:avalon-framework-api)，它被软件包 apache-commons-logging-1.1.2-7.el7.noarch 需要
---> 软件包 cal10n.noarch.0.0.7.7-4.el7 将被 安装
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 copy-jdk-configs.noarch.0.3.3-10.el7_5 将被 安装
---> 软件包 fontpackages-filesystem.noarch.0.1.44-8.el7 将被 安装
---> 软件包 javassist.noarch.0.3.16.1-10.el7 将被 安装
---> 软件包 libICE.x86_64.0.1.0.9-9.el7 将被 安装
---> 软件包 libSM.x86_64.0.1.2.2-2.el7 将被 安装
---> 软件包 libX11-common.noarch.0.1.6.5-1.el7 将被 安装
---> 软件包 libbasicobjects.x86_64.0.0.1.1-29.el7 将被 安装
---> 软件包 libblkid.x86_64.0.2.23.2-52.el7 将被 升级
---> 软件包 libblkid.x86_64.0.2.23.2-52.el7_5.1 将被 更新
---> 软件包 libcollection.x86_64.0.0.7.0-29.el7 将被 安装
---> 软件包 libini_config.x86_64.0.1.3.1-29.el7 将被 安装
--> 正在处理依赖关系 libpath_utils.so.1(PATH_UTILS_0.2.1)(64bit)，它被软件包 libini_config-1.3.1-29.el7.x86_64 需要
--> 正在处理依赖关系 libpath_utils.so.1()(64bit)，它被软件包 libini_config-1.3.1-29.el7.x86_64 需要
---> 软件包 libmount.x86_64.0.2.23.2-52.el7 将被 升级
---> 软件包 libmount.x86_64.0.2.23.2-52.el7_5.1 将被 更新
---> 软件包 libref_array.x86_64.0.0.1.5-29.el7 将被 安装
---> 软件包 libverto-libevent.x86_64.0.0.2.5-4.el7 将被 安装
---> 软件包 libxcb.x86_64.0.1.12-1.el7 将被 安装
--> 正在处理依赖关系 libXau.so.6()(64bit)，它被软件包 libxcb-1.12-1.el7.x86_64 需要
---> 软件包 lksctp-tools.x86_64.0.1.0.17-2.el7 将被 安装
---> 软件包 log4j.noarch.0.1.2.17-16.el7_4 将被 安装
--> 正在处理依赖关系 mvn(org.apache.geronimo.specs:geronimo-jms_1.1_spec)，它被软件包 log4j-1.2.17-16.el7_4.noarch 需要
--> 正在处理依赖关系 mvn(javax.mail:mail)，它被软件包 log4j-1.2.17-16.el7_4.noarch 需要
---> 软件包 nss.x86_64.0.3.34.0-4.el7 将被 升级
--> 正在处理依赖关系 nss = 3.34.0-4.el7，它被软件包 nss-sysinit-3.34.0-4.el7.x86_64 需要
--> 正在处理依赖关系 nss(x86-64) = 3.34.0-4.el7，它被软件包 nss-tools-3.34.0-4.el7.x86_64 需要
---> 软件包 nss.x86_64.0.3.36.0-7.el7_5 将被 更新
--> 正在处理依赖关系 nss-util >= 3.36.0-1，它被软件包 nss-3.36.0-7.el7_5.x86_64 需要
--> 正在处理依赖关系 nspr >= 4.19.0，它被软件包 nss-3.36.0-7.el7_5.x86_64 需要
---> 软件包 nss-softokn.x86_64.0.3.34.0-2.el7 将被 升级
---> 软件包 nss-softokn.x86_64.0.3.36.0-5.el7_5 将被 更新
---> 软件包 nss-softokn-freebl.x86_64.0.3.34.0-2.el7 将被 升级
---> 软件包 nss-softokn-freebl.i686.0.3.36.0-5.el7_5 将被 安装
---> 软件包 nss-softokn-freebl.x86_64.0.3.36.0-5.el7_5 将被 更新
---> 软件包 python-cffi.x86_64.0.1.6.0-5.el7 将被 安装
--> 正在处理依赖关系 python-pycparser，它被软件包 python-cffi-1.6.0-5.el7.x86_64 需要
---> 软件包 python-enum34.noarch.0.1.0.4-1.el7 将被 安装
---> 软件包 python-idna.noarch.0.2.4-1.el7 将被 安装
---> 软件包 python-ipaddress.noarch.0.1.0.16-2.el7 将被 安装
---> 软件包 python-javapackages.noarch.0.3.4.1-11.el7 将被 安装
--> 正在处理依赖关系 python-lxml，它被软件包 python-javapackages-3.4.1-11.el7.noarch 需要
---> 软件包 python-setuptools.noarch.0.0.9.8-7.el7 将被 安装
--> 正在处理依赖关系 python-backports-ssl_match_hostname，它被软件包 python-setuptools-0.9.8-7.el7.noarch 需要
---> 软件包 python-six.noarch.0.1.9.0-2.el7 将被 安装
---> 软件包 quota-nls.noarch.1.4.01-17.el7 将被 安装
---> 软件包 stix-fonts.noarch.0.1.1.0-5.el7 将被 安装
---> 软件包 tcp_wrappers.x86_64.0.7.6-77.el7 将被 安装
---> 软件包 ttmkfdir.x86_64.0.3.0.9-42.el7 将被 安装
---> 软件包 tzdata-java.noarch.0.2018f-2.el7 将被 安装
---> 软件包 util-linux.x86_64.0.2.23.2-52.el7 将被 升级
---> 软件包 util-linux.x86_64.0.2.23.2-52.el7_5.1 将被 更新
---> 软件包 xorg-x11-font-utils.x86_64.1.7.5-20.el7 将被 安装
--> 正在处理依赖关系 libfontenc.so.1()(64bit)，它被软件包 1:xorg-x11-font-utils-7.5-20.el7.x86_64 需要
--> 正在处理依赖关系 libXfont.so.1()(64bit)，它被软件包 1:xorg-x11-font-utils-7.5-20.el7.x86_64 需要
--> 正在检查事务
---> 软件包 avalon-framework.noarch.0.4.3-10.el7 将被 安装
--> 正在处理依赖关系 xalan-j2，它被软件包 avalon-framework-4.3-10.el7.noarch 需要
---> 软件包 avalon-logkit.noarch.0.2.1-14.el7 将被 安装
--> 正在处理依赖关系 tomcat-servlet-3.0-api，它被软件包 avalon-logkit-2.1-14.el7.noarch 需要
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 geronimo-jms.noarch.0.1.1.1-19.el7 将被 安装
---> 软件包 javamail.noarch.0.1.4.6-8.el7 将被 安装
---> 软件包 libXau.x86_64.0.1.0.8-2.1.el7 将被 安装
---> 软件包 libXfont.x86_64.0.1.5.2-1.el7 将被 安装
---> 软件包 libfontenc.x86_64.0.1.1.3-3.el7 将被 安装
---> 软件包 libpath_utils.x86_64.0.0.2.1-29.el7 将被 安装
---> 软件包 nspr.x86_64.0.4.17.0-1.el7 将被 升级
---> 软件包 nspr.x86_64.0.4.19.0-1.el7_5 将被 更新
---> 软件包 nss-sysinit.x86_64.0.3.34.0-4.el7 将被 升级
---> 软件包 nss-sysinit.x86_64.0.3.36.0-7.el7_5 将被 更新
---> 软件包 nss-tools.x86_64.0.3.34.0-4.el7 将被 升级
---> 软件包 nss-tools.x86_64.0.3.36.0-7.el7_5 将被 更新
---> 软件包 nss-util.x86_64.0.3.34.0-2.el7 将被 升级
---> 软件包 nss-util.x86_64.0.3.36.0-1.el7_5 将被 更新
---> 软件包 python-backports-ssl_match_hostname.noarch.0.3.5.0.1-1.el7 将被 安装
--> 正在处理依赖关系 python-backports，它被软件包 python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch 需要
---> 软件包 python-lxml.x86_64.0.3.2.1-4.el7 将被 安装
---> 软件包 python-pycparser.noarch.0.2.14-1.el7 将被 安装
--> 正在处理依赖关系 python-ply，它被软件包 python-pycparser-2.14-1.el7.noarch 需要
--> 正在检查事务
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 python-backports.x86_64.0.1.0-8.el7 将被 安装
---> 软件包 python-ply.noarch.0.3.4-11.el7 将被 安装
---> 软件包 tomcat-servlet-3.0-api.noarch.0.7.0.76-8.el7_5 将被 安装
---> 软件包 xalan-j2.noarch.0.2.7.1-23.el7 将被 安装
--> 正在处理依赖关系 xerces-j2，它被软件包 xalan-j2-2.7.1-23.el7.noarch 需要
--> 正在处理依赖关系 osgi(org.apache.xerces)，它被软件包 xalan-j2-2.7.1-23.el7.noarch 需要
--> 正在检查事务
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 xerces-j2.noarch.0.2.11.0-17.el7_0 将被 安装
--> 正在处理依赖关系 xml-commons-resolver >= 1.2，它被软件包 xerces-j2-2.11.0-17.el7_0.noarch 需要
--> 正在处理依赖关系 xml-commons-apis >= 1.4.01，它被软件包 xerces-j2-2.11.0-17.el7_0.noarch 需要
--> 正在处理依赖关系 osgi(org.apache.xml.resolver)，它被软件包 xerces-j2-2.11.0-17.el7_0.noarch 需要
--> 正在处理依赖关系 osgi(javax.xml)，它被软件包 xerces-j2-2.11.0-17.el7_0.noarch 需要
--> 正在检查事务
---> 软件包 cloudstack-common.x86_64.0.4.11.1.0-1.el6 将被 安装
--> 正在处理依赖关系 python(abi) = 2.6，它被软件包 cloudstack-common-4.11.1.0-1.el6.x86_64 需要
---> 软件包 xml-commons-apis.noarch.0.1.4.01-16.el7 将被 安装
---> 软件包 xml-commons-resolver.noarch.0.1.2-15.el7 将被 安装
--> 解决依赖关系完成
错误：软件包：cloudstack-common-4.11.1.0-1.el6.x86_64 (cloudstack)
          需要：python(abi) = 2.6
          已安装: python-2.7.5-68.el7.x86_64 (@anaconda)
              python(abi) = 2.7
              python(abi) = 2.7
          可用: python-2.7.5-69.el7_5.x86_64 (updates)
              python(abi) = 2.7
              python(abi) = 2.7
 您可以尝试添加 --skip-broken 选项来解决该问题
 您可以尝试执行：rpm -Va --nofiles --nodigest

```
 提示需要依赖Python2.6的依赖，而CentOS7本身持有Python2.7，结果装好了Python2.6之后还是这种情况。于是谷歌，终于在一个论坛中找到了解决方案：

 **1、修改CloudStack软件源：**

 
```
vi /etc/yum.repos.d/cloudstack.repo

```
 如果没有就创建一个

 **2、添加一下内容：**

 
```
[cloudstack]
name=cloudstack
baseurl=http://cloudstack.apt-get.eu/rhel/7/4.11/
enabled=1
gpgcheck=0

```
 重新执行命令yum install cloudstack-management，问题解决。  
 估计是很多朋友定义CloudStack软件源时将baseurl写成了baseurl=http://cloudstack.apt-get.eu/rhel/**6**/4.11/，导致了上述这个问题

   
  