---
title: CentOS 7 安装KVM虚拟机详解
date: 2018-11-12 12:01:56
tags: CSDN迁移
---
  **转载自：[https://github.com/jaywcjlove/handbook/blob/master/CentOS/CentOS7安装KVM虚拟机详解.md](https://github.com/jaywcjlove/handbook/blob/master/CentOS/CentOS7%E5%AE%89%E8%A3%85KVM%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%AF%A6%E8%A7%A3.md)**

 ![](img/kvmbanner-logo3.png)

 基于 CentOS Linux release 7.2.1511 (Core) 的环境下命令行的方式安装KVM的详细过程。

 
# []()目录

   
  * [检测是否支持KVM](#%E6%A3%80%E6%B5%8B%E6%98%AF%E5%90%A6%E6%94%AF%E6%8C%81kvm) 
  * [安装 KVM 环境](#%E5%AE%89%E8%A3%85-kvm-%E7%8E%AF%E5%A2%83) 
  * [安装虚拟机](#%E5%AE%89%E8%A3%85%E8%99%9A%E6%8B%9F%E6%9C%BA)  
      * [命令行配置系统](#%E5%91%BD%E4%BB%A4%E8%A1%8C%E9%85%8D%E7%BD%AE%E7%B3%BB%E7%BB%9F) 
      * [连接虚拟机](#%E8%BF%9E%E6%8E%A5%E8%99%9A%E6%8B%9F%E6%9C%BA) 
      * [虚拟机其它管理](#%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%85%B6%E5%AE%83%E7%AE%A1%E7%90%86)   
  * [配置物理机网络](#%E9%85%8D%E7%BD%AE%E7%89%A9%E7%90%86%E6%9C%BA%E7%BD%91%E7%BB%9C) 
  * [端口转发](#%E7%AB%AF%E5%8F%A3%E8%BD%AC%E5%8F%91) 
  * [公网访问虚拟机](#%E5%85%AC%E7%BD%91%E8%AE%BF%E9%97%AE%E8%99%9A%E6%8B%9F%E6%9C%BA) 
  * [配置宿主机网络](#%E9%85%8D%E7%BD%AE%E5%AE%BF%E4%B8%BB%E6%9C%BA%E7%BD%91%E7%BB%9C)  
      * [Bridge模式配置](#bridge%E6%A8%A1%E5%BC%8F%E9%85%8D%E7%BD%AE) 
      * [NAT模式](#nat%E6%A8%A1%E5%BC%8F) 
      * [自定义NAT网络](#%E8%87%AA%E5%AE%9A%E4%B9%89nat%E7%BD%91%E7%BB%9C) 
      * [退出虚拟机](#%E9%80%80%E5%87%BA%E8%99%9A%E6%8B%9F%E6%9C%BA)   
  * [修改虚拟机配置信息](#%E4%BF%AE%E6%94%B9%E8%99%9A%E6%8B%9F%E6%9C%BA%E9%85%8D%E7%BD%AE%E4%BF%A1%E6%81%AF) 
  * [克隆虚拟机](#%E5%85%8B%E9%9A%86%E8%99%9A%E6%8B%9F%E6%9C%BA) 
  * [通过镜像创建虚拟机](#%E9%80%9A%E8%BF%87%E9%95%9C%E5%83%8F%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA) 
  * [动态更改cpu数量和内存大小](#%E5%8A%A8%E6%80%81%E6%9B%B4%E6%94%B9cpu%E6%95%B0%E9%87%8F%E5%92%8C%E5%86%85%E5%AD%98%E5%A4%A7%E5%B0%8F) 
  * [挂载磁盘](#%E6%8C%82%E8%BD%BD%E7%A3%81%E7%9B%98)  
      * [创建磁盘](#%E5%88%9B%E5%BB%BA%E7%A3%81%E7%9B%98)   
  * [常用命令说明](#%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4%E8%AF%B4%E6%98%8E)  
      * [virt-install](#virt-install) 
      * [virsh](#virsh)   
  * [错误解决](#%E9%94%99%E8%AF%AF%E8%A7%A3%E5%86%B3) 
  * [参考文章](#%E5%8F%82%E8%80%83%E6%96%87%E7%AB%A0)   
## []()检测是否支持KVM

 KVM 是基于 x86 虚拟化扩展(Intel VT 或者 AMD-V) 技术的虚拟机软件，所以查看 CPU 是否支持 VT 技术，就可以判断是否支持KVM。有返回结果，如果结果中有vmx（Intel）或svm(AMD)字样，就说明CPU的支持的。

 
```
cat /proc/cpuinfo | egrep 'vmx|svm'

flags   : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm arat epb pln pts dtherm tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc

```
 关闭SELinux，将 /etc/sysconfig/selinux 中的  `SELinux=enforcing`  修改为  `SELinux=disabled` 

 
```
vi /etc/sysconfig/selinux

```
 
## []()安装 KVM 环境

 通过 [yum](https://jaywcjlove.github.io/linux-command/c/yum.html) 安装 kvm 基础包和管理工具

 kvm相关安装包及其作用:

  
  * `qemu-kvm`  主要的KVM程序包 
  * `python-virtinst`  创建虚拟机所需要的命令行工具和程序库 
  * `virt-manager`  GUI虚拟机管理工具 
  * `virt-top`  虚拟机统计命令 
  * `virt-viewer`  GUI连接程序，连接到已配置好的虚拟机 
  * `libvirt`  C语言工具包，提供libvirt服务 
  * `libvirt-client`  为虚拟客户机提供的C语言工具包 
  * `virt-install`  基于libvirt服务的虚拟机创建命令 
  * `bridge-utils`  创建和管理桥接设备的工具  
```
# 安装 kvm 
# ------------------------
# yum -y install qemu-kvm python-virtinst libvirt libvirt-python virt-manager libguestfs-tools bridge-utils virt-install

yum -y install qemu-kvm libvirt virt-install bridge-utils 

# 重启宿主机，以便加载 kvm 模块
# ------------------------
reboot

# 查看KVM模块是否被正确加载
# ------------------------
lsmod | grep kvm

kvm_intel             162153  0
kvm                   525259  1 kvm_intel


```
 开启kvm服务，并且设置其开机自动启动

 
```
systemctl start libvirtd
systemctl enable libvirtd

```
 查看状态操作结果，如 `Active: active (running)` ，说明运行情况良好

 
```
systemctl status libvirtd
systemctl is-enabled libvirtd

● libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
   Active: active (running) since 二 2001-01-02 11:29:53 CST; 1h 41min ago
     Docs: man:libvirtd(8)
           http://libvirt.org

```
 
## []()安装虚拟机

 安装前要设置环境语言为英文 `LANG="en_US.UTF-8"` ，如果是中文的话某些版本可能会报错。 `CentOS 7`  在这里修改  `/etc/locale.conf` 。

 kvm创建虚拟机，特别注意 `.iso` 镜像文件一定放到 `/home`  或者根目录重新创建目录，不然会因为权限报错，无法创建虚拟机。

 
```
virt-install \
--virt-type=kvm \
--name=centos78 \
--vcpus=2 \
--memory=4096 \
--location=/tmp/CentOS-7-x86_64-Minimal-1511.iso \
--disk path=/home/vms/centos78.qcow2,size=40,format=qcow2 \
--network bridge=br0 \
--graphics none \
--extra-args='console=ttyS0' \
--force
# ------------------------
virt-install --virt-type=kvm --name=centos88 --vcpus=2 --memory=4096 --location=/tmp/CentOS-7-x86_64-Minimal-1511.iso --disk path=/home/vms/centos88.qcow2,size=40,format=qcow2 --network bridge=br0 --graphics none --extra-args='console=ttyS0' --force

```
 
### []()命令行配置系统

 上面创建虚拟机命令最终需要你配置系统基础设置，带  `[!]`  基本都是要配置的，按照顺序往下配置，按对用的数字以此进行设置。

 
```

Installation

 1) [x] Language settings                 2) [!] Timezone settings
        (English (United States))                (Timezone is not set.)
 3) [!] Installation source               4) [!] Software selection
        (Processing...)                          (Processing...)
 5) [!] Installation Destination          6) [x] Kdump
        (No disks selected)                      (Kdump is enabled)
 7) [ ] Network configuration             8) [!] Root password
        (Not connected)                          (Password is not set.)
 9) [!] User creation
        (No user will be created)
  Please make your choice from above ['q' to quit | 'b' to begin installation |
  'r' to refresh]:

```
  
  2. Timezone settings 时区设置选择  `5) Asia亚洲` ，再选择城市  `62) Shanghai上海`   
```
Available regions
 1)  Africa                 6)  Atlantic              10)  Pacific
 2)  America                7)  Australia             11)  US
 3)  Antarctica             8)  Europe                12)  Etc
 4)  Arctic                 9)  Indian
 5)  Asia
Please select the timezone.
Use numbers or type names directly [b to region list, q to quit]: 5
--------------------

 8)  Baghdad               35)  Kathmandu             61)  Seoul
 9)  Bahrain               36)  Khandyga              62)  Shanghai
10)  Baku                  37)  Kolkata               63)  Singapore
26)  Hong_Kong             53)  Pontianak
27)  Hovd
Please select the timezone.
Use numbers or type names directly [b to region list, q to quit]: 62


```
  
  2. Installation source 安装源输入数字 `2`   
```
Choose an installation source type.
 1)  CD/DVD
 2)  local ISO file
 3)  Network
  Please make your choice from above ['q' to quit | 'c' to continue |
  'r' to refresh]: 2

```
  
  2. Software selection 软件选择  
```
Base environment
Software selection

Base environment

 1)  [x] Minimal Install
  Please make your choice from above ['q' to quit | 'c' to continue |
  'r' to refresh]:

```
  
  2. Installation Destination 安装目的地  
```
Installation Destination

[x] 1) : 40 GiB (vda)

1 disk selected; 40 GiB capacity; 40 GiB free ...

  Please make your choice from above ['q' to quit | 'c' to continue |
  'r' to refresh]: c


Autopartitioning Options 自动分区选项

[ ] 1) Replace Existing Linux system(s) 替换现有的Linux系统

[x] 2) Use All Space 使用所有空间

[ ] 3) Use Free Space 使用可用空间

================================================================================
Partition Scheme Options 分区方案选项

[ ] 1) Standard Partition 标准分区

[ ] 2) Btrfs Btrfs

[x] 3) LVM LVM(逻辑卷管理)

[ ] 4) LVM Thin Provisioning 精简配置

Select a partition scheme configuration.

  Please make your choice from above ['q' to quit | 'c' to continue |
  'r' to refresh]: c


```
 此处也可以只设置  `Root 密码` 和 `Installation Destination 安装目的地` 其它进入系统设置比如时区设置如下：

 
```
echo "TZ='Asia/Shanghai'; export TZ" >> /etc/profile

```
 
### []()连接虚拟机

 通过  `virsh console <虚拟机名称>`  命令来连接虚拟机

 
```
# 查看虚拟机
virsh list              # 查看在运行的虚拟机
virsh list --all         # 查看所有虚拟机

 Id    Name                           State
----------------------------------------------------
 7     centos72                       running

```
 连接虚拟机

 
```
virsh console centos72

```
 配置虚拟机网络，编辑 `vi /etc/sysconfig/network-scripts/ifcfg-eth0` 

 
```
TYPE=Ethernet
BOOTPROTO=static
IPADDR=192.168.120.200
PREFIX=24
GATEWAY=192.168.120.1
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eth0
UUID=adfa3b7d-bf60-47e6-8482-871dee686fb5
DEVICE=eth0
ONBOOT=yes

```
 添加DNS配置，也可以放到 `ifcfg-eth0` 中，DNS不是随便设置的，你可以通过[host](https://jaywcjlove.github.io/linux-command/c/host.html)、[dig](https://jaywcjlove.github.io/linux-command/c/dig.html)、[nslookup](https://jaywcjlove.github.io/linux-command/c/nslookup.html)命令查询DNS，如果这些工具不存在可以通过 `yum install bind-utils -y` 来安装一下。

 
```
# 如果没有在网络配置添加DNS可以这种方式添加DNS
echo "nameserver 192.168.188.1" > /etc/resolv.conf

```
 激活网卡

 
```
ifup eth0 # 激活网卡

```
 
### []()虚拟机其它管理

 
```
virsh start centos72     # 虚拟机开启（启动）：
virsh reboot centos72    # 虚拟机重新启动
virsh shutdown centos72  # 虚拟机关机
virsh destroy centos72   # 强制关机（强制断电）
virsh suspend centos72   # 暂停（挂起）KVM 虚拟机
virsh resume centos72    # 恢复被挂起的 KVM 虚拟机
virsh undefine centos72  # 该方法只删除配置文件，磁盘文件未删除
virsh autostart centos72 # 随物理机启动而启动（开机启动）
virsh autostart --disable centos72 # 取消标记为自动开始（取消开机启动）

```
 
## []()配置物理机网络

 目前我只有一个固定IP，通过配置 `eno2` ，网桥当做路由器，虚拟机共享物理机进出网络。物理机网络配置，网络进出走 `eno2`  编辑 `vi /etc/sysconfig/network-scripts/ifcfg-eno2` 

 
```
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eno2
UUID=f66c303e-994a-43cf-bd91-bb897dc2088d
DEVICE=eno2
ONBOOT=yes

IPADDR=<这里固定IP配置的地方>  # 设置IP地址
PREFIX=24                   # 设置子网掩码
GATEWAY=<这里设置网关>        # 设置网关
DNS1=<这里设置DNS>           # DNS

```
 `ifcfg-br0`  桥接网卡配置在同一个目录中。

 
```
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.120.1
PREFIX=24

```
 `ifcfg-eno1`  物理网卡指定桥接网卡 `BRIDGE="br0"` 

 
```
TYPE=Ethernet
BOOTPROTO=none
NAME=eno1
DEVICE=eno1
ONBOOT=yes
BRIDGE="br0"

```
 配置路由转发 `vi /etc/sysctl.conf` 

 
```
# Controls IP packet forwarding
net.ipv4.ip_forward = 0
修改为
# Controls IP packet forwarding
net.ipv4.ip_forward = 1    允许内置路由

```
 再执行  `sysctl -p`  使其生效

 
## []()端口转发

 现在我们还以上述VM为例，目前该KVM的公网IP为 `211.11.61.7` ，VM的IP为 `192.168.188.115` ，现在我要求通过访问KVM的2222端口访问VM的22端口。

 编辑 `vi /etc/rc.d/rc.local`  添加下面命令，达到开机重启配置网络转发规则。

 
```
# 启动网络转发规则
iptables -t nat -A : -s 192.168.188.0/24 -j SNAT --to-source 211.11.61.7

iptables -t nat -A POSTROUTING -s 192.168.188.0/24 -j SNAT --to-source 211.11.61.7
iptables -t nat -A PREROUTING -d  211.11.61.7 -p tcp --dport 2222  -j DNAT --to-dest 192.168.188.115:22
iptables -t nat -A PREROUTING -d  211.11.61.7 -p tcp --dport 2221  -j DNAT --to-dest 192.168.188.115:21

# 实际效果可以通过外网连接虚拟机
ssh -p 2222 root@211.11.61.7

```
 通过[iptables](https://jaywcjlove.github.io/linux-command/c/iptables.html)命令来设置转发规则，源SNAT规则，源网络地址转换，SNAT就是重写包的源IP地址。

 
```
# 数据包进行 源NAT(SNAT)，系统先路由——>再过滤（FORWARD)——>最后才进行POSTROUTING SNAT地址翻译
# -t<表>：指定要操纵的表；
# -A：向规则链中添加条目；
# -s：指定要匹配的数据包源ip地址；
# -j<目标>：指定要跳转的目标；
# -j SNAT：源网络地址转换，SNAT就是重写包的源IP地址
# --to-source ipaddr[-ipaddr][:port-port] 
#   它可以指定单个新的源IP地址，IP地址的包含范围，以及可选的端口范围（仅当规则还指定-p tcp或-p udp时才有效）。 
#   如果没有指定端口范围，则低于512的源端口将映射到512以下的其他端口：512和1023之间的端口将映射到低于1024的端口，
#   其他端口将被映射到1024或更高。 在可能的情况下，不会发生港口更改。
#   在内核高达2.6.10，您可以添加几个 - 源选项。 
#   对于这些内核，如果通过地址范围或多个源选项指定多个源地址，则会在这些地址之间进行简单的循环（循环中循环）。 
#   后来的内核（> = 2.6.11-rc1）不再具有NAT到多个范围的能力。
iptables -t nat -A POSTROUTING -s 192.168.120.0/24 -j SNAT --to-source <固定IP>
# cat /etc/sysconfig/iptables

```
 
## []()公网访问虚拟机

 通过公网ip  `192.168.188.222` 端口 `2280` ，转发到虚拟机 `192.168.111.133:80` 上面

 
```
iptables -t nat -A PREROUTING -d 192.168.188.222 -p tcp --dport 2280 -j DNAT --to-dest 192.168.111.133:80

```
 重启并保存  `iptables`  配置。

 
```
# 保存 
service iptables save
# 重启
service iptables restart

```
 
## []()配置宿主机网络

  
  2. KVM 虚拟机是基于 NAT 的网络配置； 
  4. 只有同一宿主机的虚拟键之间可以互相访问，跨宿主机是不能访问； 
  6. 虚拟机需要和宿主机配置成桥接模式，以便虚拟机可以在局域网内可见；  
### []()Bridge模式配置

 Bridge方式即虚拟网桥的网络连接方式，是客户机和子网里面的机器能够互相通信。可以使虚拟机成为网络中具有独立IP的主机。**桥接网络**（也叫 **物理设备共享**）被用作把一个物理设备复制到一台虚拟机。网桥多用作高级设置，特别是主机多个网络接口的情况。

 
```
┌─────────────────────────┐      ┌─────────────────┐
│          HOST           │      │Virtual Machine 1│
│ ┌──────┐      ┌───────┐ │      │    ┌──────┐     │
│ │ br0  │──┬───│ vnet0 │─│─ ─ ─ │    │ br0  │     │
│ └──────┘  │   └───────┘ │      │    └──────┘     │
│     │     │             │      └─────────────────┘
│     │     │   ┌───────┐ │      ┌─────────────────┐
│ ┌──────┐  └───│ vnet1 │─│─     │Virtual Machine 2│
│ │ eno0 │      └───────┘ │ │    │    ┌──────┐     │
│ └──────┘                │  ─ ─ │    │ br0  │     │
│ ┌──────┐                │      │    └──────┘     │
│ │ eno1 │                │      └─────────────────┘
│ └──────┘                │
└─────────────────────────┘

```
 通过[ip](https://jaywcjlove.github.io/linux-command/c/ip.html) 命令查看宿主机配置文件的名字

 
```
ip addr

6: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 38:63:bb:44:cf:6c brd ff:ff:ff:ff:ff:ff
    inet 192.168.188.132/24 brd 192.168.188.255 scope global dynamic eno1
       valid_lft 2822sec preferred_lft 2822sec
    inet6 fe80::3a63:bbff:fe44:cf6c/64 scope link
       valid_lft forever preferred_lft forever

```
 可以看到上面 `eno1` 是有获取到ip地址的，相对应的文件在 `/etc/sysconfig/network-scripts/` 目录中， `ifcfg-eno1`  宿主机的物理网卡配置文件

 
```
# cat ifcfg-eno1
TYPE=Ethernet
BOOTPROTO=static
NAME=eno1
DEVICE=eno1
UUID=242b3d4d-37a5-4f46-b072-55554c185ecf
ONBOOT=yes

BRIDGE="br0" # 指定桥接网卡的名称

```
 `ifcfg-br0`  桥接网卡配置在同一个目录中。

 
```
# cat ifcfg-br0
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=br0
UUID=242b3d4d-37a5-4f46-b072-55554c185ecf
DEVICE=br0
ONBOOT=yes
TYPE=bridge  # 将制定为桥接类型
IPADDR=192.168.188.133  # 设置IP地址
PREFIX=24               # 设置子网掩码
GATEWAY=192.168.188.1   # 设置网关

```
 配置好之后，通过[systemctl](https://jaywcjlove.github.io/linux-command/c/systemctl.html) 命令重启网卡。

 
```
ifup eno1 # 激活网卡
ifup br0 # 激活桥接网卡
# 两种重启网络的方法
systemctl restart network.service
service network restart

# 校验桥接接口
brctl show

bridge name bridge id   STP enabled interfaces
br0   8000.3863bb44cf6c no    eno1
              vnet0
virbr0    8000.525400193f0f yes   virbr0-nic

```
 
### []()NAT模式

 NAT(Network Address Translation网络地址翻译)，NAT方式是kvm安装后的默认方式。它支持主机与虚拟机的互访，同时也支持虚拟机访问互联网，但不支持外界访问虚拟机。

 
```
virsh net-edit default # 如果要创建或者修改NAT网络，要先编辑default.xml：
virsh net-list --all

 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     no            no

```
 default是宿主机安装虚拟机支持模块的时候自动安装的。

 
```
ip a l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens1f0: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 2c:44:fd:8c:43:44 brd ff:ff:ff:ff:ff:ff
3: ens1f1: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 2c:44:fd:8c:43:45 brd ff:ff:ff:ff:ff:ff
4: ens1f2: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 2c:44:fd:8c:43:46 brd ff:ff:ff:ff:ff:ff
5: ens1f3: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 2c:44:fd:8c:43:47 brd ff:ff:ff:ff:ff:ff
6: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP qlen 1000
    link/ether 38:63:bb:44:cf:6c brd ff:ff:ff:ff:ff:ff
7: eno2: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 38:63:bb:44:cf:6d brd ff:ff:ff:ff:ff:ff
8: eno3: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 38:63:bb:44:cf:6e brd ff:ff:ff:ff:ff:ff
9: eno4: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether 38:63:bb:44:cf:6f brd ff:ff:ff:ff:ff:ff
10: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 52:54:00:19:3f:0f brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
11: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 500
    link/ether 52:54:00:19:3f:0f brd ff:ff:ff:ff:ff:ff
12: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 38:63:bb:44:cf:6c brd ff:ff:ff:ff:ff:ff
    inet 192.168.188.132/24 brd 192.168.188.255 scope global dynamic br0
       valid_lft 3397sec preferred_lft 3397sec
    inet 192.168.188.133/24 brd 192.168.188.255 scope global secondary br0
       valid_lft forever preferred_lft forever
    inet6 fe80::3a63:bbff:fe44:cf6c/64 scope link
       valid_lft forever preferred_lft forever
19: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UNKNOWN qlen 500
    link/ether fe:54:00:72:12:a8 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe72:12a8/64 scope link
       valid_lft forever preferred_lft forever

```
 其中virbr0是由宿主机虚拟机支持模块安装时产生的虚拟网络接口，也是一个switch和bridge，负责把内容分发到各虚拟机。几个虚拟机管理模块产生的接口关系如下图:

 
```
┌───────────────────────┐                      
│         HOST          │                      
│ ┌──────┐              │   ┌─────────────────┐
│ │ br0  │─┬──────┐     │   │Virtual Machine 1│
│ └──────┘ │      │     │   │   ┌──────┐      │
│     │    │  ┌───────┐ │ ─ │   │ br0  │      │
│     │    │  │ vnet0 │─│┘  │   └──────┘      │
│ ┌──────┐ │  └───────┘ │   └─────────────────┘
│ │virbr0│ │  ┌───────┐ │   ┌─────────────────┐
│ │ -nic │ └──│ vnet1 │─│┐  │Virtual Machine 2│
│ └──────┘    └───────┘ │   │                 │
│ ┌──────┐              │└ ─│   ┌──────┐      │
│ │ eno0 │              │   │   │ br0  │      │
│ └──────┘              │   │   └──────┘      │
│ ┌──────┐              │   └─────────────────┘
│ │ eno1 │              │
│ └──────┘              │
└───────────────────────┘

```
 从图上可以看出，虚拟接口和物理接口之间没有连接关系，所以虚拟机只能在通过虚拟的网络访问外部世界，无法从网络上定位和访问虚拟主机。

 virbr0是一个桥接器，接收所有到网络192.168.122.*的内容。从下面命令可以验证：

 
```
brctl show
# 输出结果
# ---------------------
# bridge name bridge id   STP enabled interfaces
# br0   8000.3863bb44cf6c no    eno1
#               vnet0
# virbr0    8000.525400193f0f yes   virbr0-nic

ip route
# default via 192.168.188.1 dev br0
# 169.254.0.0/16 dev br0  scope link  metric 1012
# 192.168.122.0/24 dev virbr0  proto kernel  scope link  src 192.168.122.1
# 192.168.188.0/24 dev br0  proto kernel  scope link  src 192.168.188.132

```
 同时，虚拟机支持模块会修改iptables规则，通过命令可以查看：

 
```
iptables -t nat -L -nv
iptables -t filter -L -nv

```
 如果没有default的话，或者需要扩展自己的虚拟网络，可以使用命令重新安装NAT。

 
```
virsh net-define /usr/share/libvirt/networks/default.xml

```
 此命令定义一个虚拟网络，default.xml的内容：

 
```
<network>
  <name>default</name>
  <bridge name="virbr0" />
  <forward/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254" />
    </dhcp>
  </ip>
</network>

```
 也可以修改xml，创建自己的虚拟网络。

 重新加载和激活配置：

 
```
virsh  net-define /etc/libvirt/qemu/networks/default.xml

```
 标记为自动启动：

 
```
virsh net-autostart default
# Network default marked as autostarted

virsh net-start default

```
 启动网络：

 
```
virsh net-start default
# Network default started

```
 网络启动后可以用命令brctl show 查看和验证。

 修改 `vi /etc/sysctl.conf` 中参数，允许ip转发，CentOS7是在 `vi /usr/lib/sysctl.d/00-system.conf`  这里面修改

 
```
net.ipv4.ip_forward=1

```
 通过  `sysctl -p`  查看修改结果

 
### []()自定义NAT网络

 创建名为 `management` 的NAT网络， `vi /usr/share/libvirt/networks/management.xml` 

 
```
<network>
  <name>management</name>
  <bridge name="virbr1"/>
  <forward/>
  <ip address="192.168.123.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.123.2" end="192.168.123.254"/>
    </dhcp>
  </ip>
</network>

```
 启用新建的NAT网络

 
```
virsh net-define /usr/share/libvirt/networks/management.xml
virsh net-start management
virsh net-autostart management

```
 验证

 
```
brctl show
# bridge name bridge id   STP enabled interfaces
# br0   8000.3863bb44cf6c no    eno1
#               vnet0
# virbr0    8000.525400193f0f yes   virbr0-nic
# virbr1    8000.52540027f0ba yes   virbr1-nic

virsh net-list --all
#  Name                 State      Autostart     Persistent
# ----------------------------------------------------------
#  default              active     no            no
#  management           active     yes           yes

```
 
### []()退出虚拟机

 
```
exit # 退出系统到登录界面

Ctrl+5 # 从虚拟机登录页面，退出到宿主机命令行页面
Ctrl+] # 或者下面

```
 
## []()修改虚拟机配置信息

 直接通过vim命令修改

 
```
vim  /etc/libvirt/qemu/centos72.xml

```
 通过virsh命令修改

 
```
virsh edit centos72

```
 
## []()克隆虚拟机

 
```
# 暂停原始虚拟机
virsh shutdown centos72
virt-clone -o centos72 -n centos.112 -f /home/vms/centos.112.qcow2 -m 00:00:00:00:00:01
virt-clone -o centos88 -n centos.112 --file /home/vms/centos.112.qcow2 --nonsparse

```
 `virt-clone`  参数介绍

  
  * `--version`  查看版本。 
  * `-h，--help`  查看帮助信息。 
  * `--connect=URI`  连接到虚拟机管理程序 libvirt 的URI。 
  * `-o 原始虚拟机名称`  原始虚拟机名称，必须为关闭或者暂停状态。 
  * `-n 新虚拟机名称`  --name 新虚拟机名称。 
  * `--auto-clone`  从原来的虚拟机配置自动生成克隆名称和存储路径。 
  * `-u NEW_UUID, --uuid=NEW_UUID`  克隆虚拟机的新的UUID，默认值是一个随机生成的UUID。 
  * `-m NEW_MAC, --mac=NEW_MAC`  设置一个新的mac地址，默认为随机生成 MAC。 
  * `-f NEW_DISKFILE, --file=NEW_DISKFILE`  为新客户机使用新的磁盘镜像文件地址。 
  * `--force-copy=TARGET`  强制复制设备。 
  * `--nonsparse`  不使用稀疏文件复制磁盘映像。  
## []()通过镜像创建虚拟机

 创建虚拟机镜像文件

 
```
# 复制第一次安装的干净系统镜像，作为基础镜像文件，
# 后面创建虚拟机使用这个基础镜像
cp /home/vms/centos.88.qcow2 /home/vms/centos7.base.qcow2

# 使用基础镜像文件，创建新的虚拟机镜像
cp /home/vms/centos7.base.qcow2 /home/vms/centos7.113.qcow2

```
 创建虚拟机配置文件

 
```
# 复制第一次安装的干净系统镜像，作为基础配置文件。
virsh dumpxml centos.88 > /home/vms/centos7.base.xml

# 使用基础虚拟机镜像配置文件，创建新的虚拟机配置文件
cp /home/vms/centos7.base.xml /home/vms/centos7.113.xml

# 编辑新虚拟机配置文件
vi /home/vms/centos7.113.xml

```
 主要是修改虚拟机文件名，UUID，镜像地址和网卡地址，其中 UUID 在 Linux 下可以使用  `uuidgen`  命令生成

 
```
<domain type='kvm'>
  <name>centos7.113</name>
  <uuid>1e86167a-33a9-4ce8-929e-58013fbf9122</uuid>
  <devices>
    <disk type='file' device='disk'>
      <source file='/home/vms/centos7.113.img'/>
    </disk>
    <interface type='bridge'>
      <mac address='00:00:00:00:00:04'/>
    </interface>    
    </devices>
</domain>

```
 
```
virsh define /home/vms/centos7.113.xml
# Domain centos.113 defined from /home/vms/centos7.113.xml

```
 
## []()动态更改cpu数量和内存大小

 动态调整，如果超过给虚拟机分配的最大内存，需要重启虚拟机。

 
```
virsh list --all
#  Id    名称                         状态
# ----------------------------------------------------
#  2     working112                     running

# 更改CPU
virsh setvcpus working112 --maximum 4 --config
# 更改内存
virsh setmaxmem working112 1048576 --config
# 查看信息
virsh dominfo working112

```
 
## []()挂载磁盘

 
### []()创建磁盘

 
```
mkdir /home/vms

```
 查看镜像信息

 
```
virt-filesystems --long --parts --blkdevs -h -a working112.qcow2

# Name       Type       Size  Parent
# /dev/sda1  partition  200M  /dev/sda
# /dev/sda2  partition  9.8G  /dev/sda
# /dev/sda   device     10G   -

qemu-img info working112.qcow2

# image: working112.qcow2
# file format: qcow2
# virtual size: 140G (150323855360 bytes)
# disk size: 33G
# cluster_size: 65536
# Format specific information:
#     compat: 1.1
#     lazy refcounts: true

```
 给虚拟机镜像添加 `200G` 大小，注意需要停止 `working112` 虚拟机

 
```
qemu-img resize working112.qcow2 +200G
# Image resized.

```
 首先，我们制作如下所示的磁盘的备份副本。

 
```
cp working112.qcow2 working112-orig.qcow2

```
 然后我们运行下面的命令来增加  `/dev/sda` 

 
```
virt-resize --expand /dev/sda1 working112-orig.qcow2 working112.qcow2

```
 查看镜像信息

 
```
qemu-img info working112.qcow2
# image: working112.qcow2
# file format: qcow2
# virtual size: 140G (150323855360 bytes)
# disk size: 33G
# cluster_size: 65536
# Format specific information:
#     compat: 1.1
#     lazy refcounts: true

```
 进入虚拟机 `virsh console working112`  查看信息：

 
```
vgdisplay # 显示卷组大小
lvdisplay # 显示逻辑卷大小

```
 卷组大小已增加，下面需要分配容量给逻辑卷

 
```
lvextend -L +60G /dev/centos/root

```
 还有最后一步，分配好了需要做系统调整

 
```
# ext 系统格式使用：
resize2fs /dev/centos/root
# xfs 系统格式使用下面命令
xfs_growfs /dev/centos/root

```
 
## []()常用命令说明

 
### []()virt-install

 常用参数说明

 
```
–name指定虚拟机名称
–memory分配内存大小。
–vcpus分配CPU核心数，最大与实体机CPU核心数相同
–disk指定虚拟机镜像，size指定分配大小单位为G。
–network网络类型，此处用的是默认，一般用的应该是bridge桥接。
–accelerate加速
–cdrom指定安装镜像iso
–vnc启用VNC远程管理，一般安装系统都要启用。
–vncport指定VNC监控端口，默认端口为5900，端口不能重复。
–vnclisten指定VNC绑定IP，默认绑定127.0.0.1，这里改为0.0.0.0。
–os-type=linux,windows
–os-variant=rhel6

--name      指定虚拟机名称
--ram       虚拟机内存大小，以 MB 为单位
--vcpus     分配CPU核心数，最大与实体机CPU核心数相同
–-vnc       启用VNC远程管理，一般安装系统都要启用。
–-vncport   指定VNC监控端口，默认端口为5900，端口不能重复。
–-vnclisten  指定VNC绑定IP，默认绑定127.0.0.1，这里改为0.0.0.0。
--network   虚拟机网络配置
  # 其中子选项，bridge=br0 指定桥接网卡的名称。

–os-type=linux,windows
–os-variant=rhel7.2

--disk 指定虚拟机的磁盘存储位置
  # size，初始磁盘大小，以 GB 为单位。

--location 指定安装介质路径，如光盘镜像的文件路径。
--graphics 图形化显示配置
  # 全新安装虚拟机过程中可能会有很多交互操作，比如设置语言，初始化 root 密码等等。
  # graphics 选项的作用就是配置图形化的交互方式，可以使用 vnc（一种远程桌面软件）进行链接。
  # 我们这列使用命令行的方式安装，所以这里要设置为 none，但要通过 --extra-args 选项指定终端信息，
  # 这样才能将安装过程中的交互信息输出到当前控制台。
--extra-args 根据不同的安装方式设置不同的额外选项

```
 
### []()virsh

 基础命令

 
```
virsh list --all           # 查看所有运行和没有运行的虚拟机
virsh list                 # 查看在运行的虚拟机
virsh dumpxml vm-name      # 查看kvm虚拟机配置文件
virsh start vm-name        # 启动kvm虚拟机
virsh shutdown vm-name     # 正常关机

virsh destroy vm-name      # 非正常关机，强制关闭虚拟机（相当于物理机直接拔掉电源）
virsh undefine vm-name     # 删除vm的配置文件

ls /etc/libvirt/qemu
# 查看删除结果，Centos-6.6的配置文件被删除，但磁盘文件不会被删除

virsh define file-name.xml # 根据配置文件定义虚拟机
virsh suspend vm-name      # 挂起，终止
virsh resumed vm-name      # 恢复被挂起的虚拟机
virsh autostart vm-name    # 开机自启动vm
virsh console <虚拟机名称>   # 连接虚拟机

```
 
## []()错误解决

 
```
console test
Connected to domain test
Escape character is ^]

```
 如果出现上面字符串使用 CTRL+Shift+5 CTRL+Shift+]

  
  2. ERROR Format cannot be specified for unmanaged storage.  
      virt-manager 没有找到存储池，创建储存池即可
     
       
  4. KVM VNC客户端连接闪退  
      使用real vnc或者其它vnc客户端连接kvm闪退，把客户端设置中的ColourLevel值设置为rgb222或full即可
     
       
  6. virsh shutdown 无法关闭虚拟机  
      使用该命令关闭虚拟机时，KVM是向虚拟机发送一个ACPI的指令，需要虚拟机安装acpid服务：
     
       
  8. operation failed: Active console session exists for this domain
     
        
```
# 方案1
$ ps aux | grep console
$ kill -9 <进程号>
# 方案2
$ /etc/init.d/libvirt-bin restart
# 方案3
$ ps aux | grep kvm
$ kill 对应的虚拟机进程

```
 
## []()参考文章

  
  * [KVM官方网站](https://www.linux-kvm.org/page/Main_Page) 
  * [KVM虚拟机Linux系统增加硬盘](http://www.cnblogs.com/ilanni/p/3878151.html) 
  * [virt-install 命令参数详解](https://www.ibm.com/support/knowledgecenter/zh/linuxonibm/liaat/liaatvirtinstalloptions.htm) 
  * [使用virt-install安装虚拟机，发行版安装代码直接复制运行](https://raymii.org/s/articles/virt-install_introduction_and_copy_paste_distro_install_commands.html) 
  * [KVM Linux - Expanding a Guest LVM File System Using Virt-resize](http://blog.oneiroi.co.uk/linux/kvm/virt-resize/RHEL/LVM/kvm-linux-expanding-a-lvm-guest-file-system-using-virt-resize/)    
  