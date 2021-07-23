# 在Win10中用Hyper-V虚拟机搭建CDH集群

Version: v0.1

Last Update: July/23/2021

Keywords：CDH，Hadoop，Big Data Platform，Hyper-V，Virtual Machine

Abstract：
因想要学习大数据各种相关组件和技术，而自家有台台式机可以折腾，购买所需配置的云服务器又太贵，故准备自行搭建CDH集群。因有Win10专业版，因此尝鲜玩玩Hyper-V。整体思路如下：规划好整个集群的数量、配置、角色，准备好所有软件包；先配置一台虚拟机A，尽可能将所有公共的配置设置好，再依次复制出其余虚拟机。
文内全部使用经过验证的PowerShell和Shell脚本进行配置，注释丰富便于理解整体操作逻辑（和复制粘贴）。使用VMWare/VirtualBox/KVM等基本流程均相似。

## 0. 背景

- 因想要学习大数据各种相关组件和技术，而自家有台台式机可以折腾，购买所需配置的云服务器又太贵，故准备自行搭建CDH集群。
- 最初准备使用Ubuntu+VKVM，但台式机家人共享，双系统切换也很麻烦，故考虑在Win10一个平台解决所有问题。
- 因有Win10专业版，VMWare Workstation要收费，VirtualBox是Oracle家的不喜欢，因此尝鲜玩玩Hyper-V。
- 以Win10专业版21H1+Hyper-V为例，若使用MacOS/Linux+VMWare/VirtualBox/KVM，则需依据实际情况参考操作。
- Hyper-V需要在BIOS中开启CPU虚拟化，具体操作略。
- 安装过程尽可能参考Cloudera官方文档，其余参考文章也都备注了来源。

整体思路如下：
- 规划好整个集群的数量、配置、角色，准备好所有软件包。
- 因台式机单机资源有限，同时开启多台虚拟机比较费，故先配置一台虚拟机A，尽可能将所有公共的配置设置好，再复制出其余虚拟机。
- 需要有一台虚拟机B，作为CM Server和CDH私有仓库服务器，用于安装其余虚拟机所用的所有服务。其余所有虚拟机C的设置在安装CDH前都是一样的。
- 因此操作流程为：
  - 配置A
  - 由A生成B，配置B为CM Server和CDH私有仓库服务器
  - 由A生成C，配置C为CM Agent
  - 由C生成DEFG等
  - 配置DB
  - 安装CDH

文中操作说明：
- 文中叙述的可选操作或命令，如果是斜体的 *可选：xxx* 或在命令块注释内，是我在自己搭建的实际操作中没有执行的，否则是执行了的。
- 操作涉及到Hosts/Hostname/IP/路径/文件命名/参数名/参数值等，均以集群规划部分的描述为示范。若使用不同的配置项，可将本文下载下来，查找替换成自己的配置再看更舒服。
- 使用命令行操作，不使用界面。其实若PowerShell，鼠标操作可能更简单。
- PowerShell命令行，默认需要**以管理员身份**运行终端，且仅在Terminal/PowerShell中验证过。
- Shell操作，默认需要sudo或以root用户执行。
- 如果使用yum等命令反应迟缓，可以自行配置国内proxy源加速访问。
- 涉及到vi修改文件，含义如下：
  - 修改一行 (modify: from xxx to xxx)
  - 添加一行 (append: xxx)
  - 注释一行 (comment: # xxx)，也可以删除该行

特别感谢参考文章：
- [绿叶悠，CDH6.2安装教程（详细步骤）](https://www.jianshu.com/p/610cce9f9026)
- [小KKKKKKKK，Centos7离线部署CDH6.3.2集群](https://www.jianshu.com/p/6f120a99cbae)

## 1. 集群规划

需要准备的软件和版本。

- CDH：6.3.2
- OS：CentOS 7，参考-[Cloudera Enterprise 6.3.x Supported Operating Systems](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_database_requirements.html)
- DB：MySQL 5.7，参考-[Database Requirements](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_database_requirements.html)
- JDK：Oracle 1.8，参考-[Java Requirements](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html#java_requirements)

### 1.1 硬件与角色规划

参考-[Hardware Requirements](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_hardware_requirements.html#concept_vvv_cxt_gbb)

参考-[CDH Cluster Hosts and Role Assignments](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_host_allocations.html#concept_f43_j4y_dw__section_dn3_ngj_ndb)

依据官方的"20 Worker Hosts with High Availability"角色搭配建议和最低硬件要求，整理图表如下。
其中"4c2g"代表官方建议4核CPU，2GB内存。因磁盘空间一般不是问题，故没有列出。
星号*表示该角色没有计入实际搭建和操作中，只是规划在表格中。
"Final Config"是依据实际最终搭建完成后的运行情况，倒推出的最低配置，仅供参考。
*如果宿主机内存只有16~32GB，可以使用"10 Worker Hosts without High Availability"搭配，且将DataNode放在所有VM上，三台VM即可满足，但实用性可能不足，即无法快乐的跑Job。*

| Service |       Role       |master01|master02|master03|utility01|gateway01|node001|node002|node003| Total |
| :-----: | :--------------: | -----: | -----: | -----: | ------: | ------: | ----: | ----: | ----: | ----: |
|  HDFS   |     NameNode     |   4c2g |        |        |         |         |       |       |       |       |
|         |   SecondaryNN    |        |   4c2g |        |         |         |       |       |       |       |
|         |   *JournalNode   |   1c1g |   1c1g |   1c1g |         |         |       |       |       |       |
|         |      *ZKFC       |      + |      + |        |         |         |       |       |       |       |
|         |     DataNode     |        |        |        |         |         |  4c4g |  4c4g |  4c4g |       |
|  YARN   |  ResourceManager |   1c6g |  *1c6g |        |         |         |       |       |       |       |
|         | JobHistoryServer |   1c1g |        |        |         |         |       |       |       |       |
|         |    NodeManager   |        |        |        |         |         |  8c1g |  8c1g |  8c1g |       |
|   ZK    |    ZooKeeper     |   4c1g |   4c1g |   4c1g |         |         |       |       |       |       |
|  Hive   |   HiveServer2    |        |        |        |         |    4c4g |       |       |       |       |
|         | Metastore Server |        |        |        |    4c4g |         |       |       |       |       |
|   Hue   |    Server&LB     |        |        |        |         |    1c4g |       |       |       |       |
|   CM    |   All Services   |        |        |        |   4c12g |         |       |       |       |       |
|  Oozie  |      Server      |        |        |        |    1c1g |         |       |       |       |       |
|   DB    |      MySQL       |        |        |        |    2c4g |         |       |       |       |       |
| *Kafka  |      *Kafka      |        |        |        |         |         |  2c4g |  2c4g |  2c4g |       |
| *Spark  | *History Server  |        |        |   1c1g |         |         |       |       |       |       |
| *HBase  |     *Master      |        |        |   4c4g |         |         |       |       |       |       |
|         |  *Thrift Server  |        |        |        |         |    2c1g |       |       |       |       |
|         |  *Region Server  |        |        |        |         |         |  4c8g |  4c8g |  4c8g |       |
|         |  Max CPU Cores   |      4 |      4 |      4 |       4 |       4 |     8 |     8 |     8 |    44 |
|         |   Total Memory   |     11 |      4 |      2 |      20 |       8 |     5 |     5 |     5 |    60 |
|         |   Final Config   |  4c16g |   4c8g |   4c4g |   4c16g |    4c8g |  4c8g |  4c8g |  4c8g | 16c64g|

### 1.2 各VM的FQDN与IP规划

|   IP Addr    |            FQDN            | short name |
| :----------- | -------------------------: | :--------: |
| 10.10.64.11  |  master01.cdh.lionxcat.com |     m01    |
| 10.10.64.12  |  master02.cdh.lionxcat.com |     m02    |
| 10.10.64.13  |  master03.cdh.lionxcat.com |     m03    |
| 10.10.64.51  | utility01.cdh.lionxcat.com |    ut01    |
| 10.10.64.71  | gateway01.cdh.lionxcat.com |    gw01    |
| 10.10.64.101 |   node001.cdh.lionxcat.com |    n001    |
| 10.10.64.102 |   node002.cdh.lionxcat.com |    n002    |
| 10.10.64.103 |   node003.cdh.lionxcat.com |    n003    |
| 10.10.64.250 |   centos7.cdh.lionxcat.com |     *c7    |

*c7是模板机，不是CDH集群中的成员，用于复制出其余所有VM。*

## 2. 宿主机环境准备

### 2.1 配置Terminal/PowerShell环境

- 安装Terminal和插件的方法略。
如果对某个PowerShell命令不熟悉，可以使用：

```powershell
Get-Help New-VM -Online
```

- 可选，配置好hosts，方便宿主机访问VM。

```powershell
# 用管理员权限修改 C:\Windows\System32\drivers\etc\hosts
ipconfig /flushdns
```

### 2.2 安装Hyper-V

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
# 或者可以使用DISM安装
# dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V /all /norestart
# 需要重启电脑
Restart-Computer
```

### 2.3 配置VM网络

参考-[Set up a NAT network](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network)

参考-[Hyper-V虚拟机配置内部网络固定IP并连接外网](https://www.cnblogs.com/kasnti/p/11727755.html)

```powershell
# 创建一个名称为cdh-wmswitch的内部虚拟路由器
New-VMSwitch -Name cdh-vmswitch -SwitchType Internal
# 查询出刚创建的cdh-wmswitch的ifIndex为60，依据实际情况更改
Get-NetAdapter
# 设置cdh-wmswitch的IP地址，将上一步查出的InterfaceIndex带入参数
New-NetIPAddress -IPAddress 10.10.64.1 -PrefixLength 24 -InterfaceIndex 60
# 设置cdh-wmswitch使用NAT获得物理机的互联网访问，此操作只能使用命令行
New-NetNat -Name cdh-vmnat -InternalIPInterfaceAddressPrefix 10.10.64.0/24
```

### 2.4 创建CentOS VM

参考-[Create a Virtual Machine with PowerShell](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/create-virtual-machine#create-a-virtual-machine-with-powershell)

下载[CentOS 7](https://www.centos.org/centos-linux/)，可下个Everything版本留着玩。
以下假定文件路径为“$InstallMedia”所示。

```powershell
# Set VM Name, Switch Name, and Installation Media Path.
$VMName = 'c7'
$Switch = 'cdh-vmswitch'
$BasePath = 'D:\VirtualMachines\CDH-Cluster'
$InstallMedia = 'D:\CentOS-7-x86_64-Everything-2009.iso'

# 创建VM，可自行设置启动内存大小
# 硬盘空间可以大些，因为会动态伸缩不会占用过多宿主机资源
# CentOS必须使用1代Hyper-VM
$VM = @{
    Name = $VMName
    MemoryStartupBytes = 4GB
    SwitchName = $Switch
    NewVHDPath = "$BasePath\$VMName\$VMName.vhdx"
    NewVHDSizeBytes = 64GB
    Path = $BasePath
    Generation = 1
}
New-VM @VM
# 设置CPU数量，可选，默认1core
Set-VMProcessor $VMName -Count 2

# 默认情况下如上VM会配备好一个DVD并默认从DVD启动，否则可以执行如下命令创建DVD：
# Add-VMScsiController -VMName $VMName
# Add-VMDvdDrive -VMName $VMName -ControllerNumber 1 -ControllerLocation 0 -Path $InstallMedia

# 加载CentOS7镜像并设置启动顺序
$DVDDrive = Get-VMDvdDrive $VMName
Set-VMDvdDrive $VMName -Path $InstallMedia
# 默认情况下1代VM创建好后，启动顺序即如此，不用修改
# Set-VMBios $VMName -StartupOrder @("CD", "IDE", "LegacyNetworkAdapter", "Floppy")
```

### 2.5 安装CentOS

#### 2.5.1 安装前

开启VM，并安装系统。

```powershell
$VMName = 'c7'
Start-VM $VMName
# 连接上VM进行OS安装
VMConnect localhost $VMName
```

#### 2.5.2 安装中

安装注意事项：
- 可选，在选择"Install CentOS7"前启用强制GPT分区，方便今后挂载大于2TB磁盘和磁盘动态伸缩。
    按TAB键，在最下方参数后面加上"inst.gpt"，即改成："vmlinuz ... quiet inst.gpt"，回车。
- 可选，设置时区为Shanghai。
- 可选，Keyboard layout自行配置。
- 可选，添加简体中文语言支持。
- SOFTWARE SELECTION选取Base Environment：Compute Node（计算节点）。
- 若手动磁盘分区，需添加一个容量为2M的"biosboot"分区放置GPT。
- **如果宿主机内存不够大，最好为CentOS分配够大的Swap分区，例如8GB以上，否则后面服务起不来一切白费。**
- 可选，关闭KDUMP。
- 因为生成VM时配置好了虚拟路由器，可以在安装时直接设置好IP，方便安装完成后ssh上VM。
  - 静态(Manual)IPv4；
  - 添加IP：10.10.64.250；
  - 掩码：255.255.255.0；
  - 网关：10.10.64.1；
  - DNS：223.5.5.5,8.8.8.8；
  - 开机启用（Automatic connect to ...）。
- 可选，设置好Hostname：centos7.cdh.lionxcat.com。
- 可选，创建新用户时勾上"将此用户设置为管理员"，也可今后自行添加到sudoers/wheel组。

#### 2.5.3 安装后

- 使用VMConnect可以不用IP连接VM，但内置的终端超难用。使用自己喜欢的终端ssh登录VM操作比较方便。
    若安装时不配置IP，安装后可在Hyper-V中选中该VM，最下网络Tab里找到Hyper-V自动提供的IP（v4或v6）地址。

- 若安装时配置了IP，则尝试登录VM并判断网络是否连通。以下假定在Win10中配置了hosts，若没配置可用IP。

```bash
ssh root@c7
ifconfig
ping www.xxx.com
```

- 若安装中没有配置IP，则先登录VM并设置IP。

```powershell
# 在宿主机中执行
$VMName = 'c7'
Start-VM $VMName
VMConnect localhost $VMName
```

```bash
# 在c7中执行
ifconfig
vi /etc/sysconfig/network-scripts/ifcfg-eth0 # eth0是具体VM网卡
# modify: BOOTPROTO="static"
# modify: ONBOOT="yes"
# append: IPADDR=10.10.64.250 # 今后需要替换成各VM自己的IP地址
# append: NETMASK=255.255.255.0
# append: GATEWAY=10.10.64.1
# append: DNS1=223.5.5.5
# append: DNS2=8.8.8.8

systemctl restart network
ping www.xxx.com
```

- *可选，如果发现有ssh时登录反应慢的情况，可修改关闭SSH使用DNS查询。*
```bash
vi /etc/ssh/sshd_config
# modify: from "#UseDNS yes" to "UseDNS no"

systemctl restart sshd
```

- 补充说明。
  1. 新装的这台VM（以下简称作c7），将作为基板系统，用于生成所有集群VM。
  2. 后续配置中，可随时为VM做快照（Snapshot/Checkpoint），方便备份、恢复、复制VM。
  3. 高阶玩法，可使用Ansible等批量配置，或制作好CDH安装镜像（参考[Creating Virtual Images of Cluster Hosts](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_create_image.html)和[Creating a CDH Cluster Using a Cloudera Manager Template](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cluster_template.html)）。

## 3. 通用VM的OS环境准备

*以下内容在c7上执行，是所有VM都需要的基础配置。*

### 3.1 关闭防火墙

参考-[Disabling the Firewall](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cdh_disable_iptables.html)

```bash
systemctl stop firewalld
systemctl disable firewalld
# 验证一下
systemctl status firewalld
```

### 3.2 关闭SELinux

参考-[Setting SELinux mode](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cdh_disable_selinux.html)

```bash
vi /etc/selinux/config
# modify: from "SELINUX=enforcing" to "SELINUX=disabled"

setenforce 0
reboot
# 验证一下
getenforce
```

### 3.3 系统配置参数

参考-[nproc Configuration](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_os_requirements.html#nproc_config)

```bash
vi /etc/security/limits.d/20-nproc.conf # 具体文件名中数字可能不是20
# comment: # * soft nproc 4096

vi /etc/security/limits.conf
# append: * soft  nproc  65536
```

参考-[Optimizing Performance in CDH](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cdh_admin_performance.html)

```bash
# Disable the tuned Service
systemctl start tuned
tuned-adm off
# tuned-adm list
systemctl stop tuned
systemctl disable tuned
```

```bash
# Disabling Transparent Hugepages (THP)
vi /etc/rc.d/rc.local
# append: 
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# to ensure that this script will be executed during boot
chmod +x /etc/rc.d/rc.local

# 在启动参数GRUB_CMDLINE_LINUX中添加"transparent_hugepage=never"
vi /etc/default/grub
# modify: GRUB_CMDLINE_LINUX="transparent_hugepage=never ...省略"

grub2-mkconfig -o /boot/grub2/grub.cfg
```

```bash
# Setting the vm.swappiness Linux Kernel Parameter
sysctl -w vm.swappiness=1

# to ensure that this script will be executed during boot
vim /etc/sysctl.conf
# append: vm.swappiness=1
```

最后最好验证是否重启后生效。若有没生效的参数，检查配置文件拼写是否正确。

```bash
reboot
ssh root@c7
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
cat /proc/sys/vm/swappiness
```

### 3.4 配置hostname

```bash
hostnamectl set-hostname centos7.cdh.lionxcat.com
vi /etc/sysconfig/network
# append: 
NETWORKING=YES
HOSTNAME=centos7.cdh.lionxcat.com

systemctl restart network
# 检查一下
uname -a
hostname
# host -v -t A $(hostname)
```

### 3.5 配置hosts

参考-[Configure Network Names](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/configure_network_names.html)

```bash
vi /etc/hosts
# 注意：一定要把FQDN放在紧跟IP后
# append:
10.10.64.11 master01.cdh.lionxcat.com m01
10.10.64.12 master02.cdh.lionxcat.com m02
10.10.64.13 master03.cdh.lionxcat.com m03
10.10.64.51 utility01.cdh.lionxcat.com ut01
10.10.64.71 gateway01.cdh.lionxcat.com gw01
10.10.64.101 node001.cdh.lionxcat.com n001
10.10.64.102 node002.cdh.lionxcat.com n002
10.10.64.103 node003.cdh.lionxcat.com n003
10.10.64.250 centos7.cdh.lionxcat.com c7
```

### 3.6 配置网络时间同步

参考-[Enable an NTP Service](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cdh_enable_ntp.html)

CDH建议删除chrony用ntpd代替，也可以配置CDH使用chrony。以下为ntpd用法，chrony可自行配置。

```bash
yum list installed chrony
# 如果有安装chrony则删除之，换成ntp
systemctl stop chronyd
yum remove -y chrony
yum install -y ntp
# 同步一次
ntpdate ntp.aliyun.com
# 启动服务
systemctl start ntpd
systemctl enable ntpd
# 同步硬件CMOS时钟
hwclock --systohc
```

## 4. CM安装

### 4.1 创建ut01的VM

在通用VM配置完成后，将c7关机并复制一份命名为ut01，用于承载CDH私有仓库（以及其他服务，参考角色规划部分）。

```bash
# c7中执行
poweroff
```

- 采用复制VHD的方式，复制c7的磁盘，使用该磁盘文件生成ut01（Hyper-V会生成新MAC地址）。
```powershell
# 在宿主机中执行
# VM基础配置
$BasePath = 'D:\VirtualMachines\CDH-Cluster'
$Switch = 'cdh-vmswitch'
# 复制源
$SourceVM = 'c7'
$SourceVHD = "$BasePath\$SourceVM\$SourceVM.vhdx"
# 复制目的地
$DestVM = 'ut01'
$DestPath = "$BasePath\$DestVM"
$DestVHD = "$DestPath\$DestVM.vhdx"

# 删除c7所有快照以自动合并磁盘文件
Remove-VMSnapshot $SourceVM

# 将文件拷贝到新目录
New-Item $DestPath -ItemType Directory
Copy-Item $SourceVHD $DestVHD

# 使用复制后的文件创建新VM，注意自行设置启动内存大小
$NewVM = @{
    Name = $DestVM
    MemoryStartupBytes = 16GB
    SwitchName = $Switch
    VHDPath = $DestVHD
    Path = $BasePath
    Generation = 1
}
New-VM @NewVM
# 可选，设置CPU
Set-VMProcessor $DestVM -Count 8
```

- 启动并配置IP和Hostname。

```powershell
# 宿主机上执行
$VMName = 'ut01'
Start-VM $VMName
VMConnect localhost $VMName
```

```bash
# ut01上执行
vi /etc/sysconfig/network-scripts/ifcfg-eth0 # eth0是具VM网卡
# append: IPADDR=10.10.64.51

hostnamectl set-hostname utility01.cdh.lionxcat.com

vi /etc/sysconfig/network
# modify: HOSTNAME=utility01.cdh.lionxcat.com

systemctl restart network
# 外网是否可达
ping www.xxx.com
# 内网c7是否可达，需要先启动c7
ping c7
```

### 4.2 开启NTP时间同步服务

可选：如果非生产环境，所以VM都与公网同步也可以，只不过今后需要关掉CM的烦人的警告。
将ut01作为授时服务器，其余VM均与ut01同步时间。

*以下内容在ut01上操作，配置ut01成为授时服务器。*

```bash
vi /etc/ntp.conf
# 开放本机时间同步服务给指定网段
# append:
# restrict 10.100.64.0 mask 255.255.255.0 nomodify notrap
# 
# 可选：更换时间服务器
# comment:
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst
# append:
# server ntp.aliyun.com prefer
# 
# 本机ntpd与本机CMOS同步
# append:
server 127.127.1.0
fudge 127.127.1.0 stratum 10

systemctl restart ntpd
```

*以下内容在c7上操作，配置c7从ut01获取时间同步。*

```bash
vi /etc/ntp.conf
# comment: 
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst
# append:
# server ut01 prefer

# 立即同步一次
systemctl stop ntpd
ntpdate ut01
systemctl start ntpd
ntpq -p
```

### 4.3 准备本地私有CDH仓库

*以下内容在ut01上执行，配置用于安装CM和CDH的私有Repo。*

参考-[Configuring a Local Package Repository](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_create_local_package_repo.html#internal_package_repo)

- 安装HTTP服务

```bash
# 以下操作在ut01上执行
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# 创建cm6目录
mkdir -p /var/www/html/cloudera-repos/cm6
# 创建cdh6目录
mkdir -p /var/www/html/cloudera-repos/cdh6
```

- 将CM和CDH包拷贝到ut01上

```powershell
# 以下操作在宿主机上执行
# 拷贝CM包
scp "D:\cm6.3.1-redhat7.tar.gz" root@ut01:/var/www/html/cloudera-repos/cm6
# 拷贝CDH包
scp "D:\cdh6\*" root@ut01:/var/www/html/cloudera-repos/cdh6
```

- 解压缩文件

```bash
# 以下操作在ut01上执行
tar xvfz cm6.3.1-redhat7.tar.gz -C /var/www/html/cloudera-repos/cm6 --strip-components=1
# 设置目录权限
chmod -R ugo+rX /var/www/html/cloudera-repos/cm6
chmod -R ugo+rX /var/www/html/cloudera-repos/cdh6
# 在宿主机浏览器中检查文件 http://ut01/cloudera-repos/
```

### 4.4 在VM上配置添加使用私有仓库

*以下内容在ut01上执行，添加本机私有Repo。*

```bash
vi /etc/yum.repos.d/cloudera-repo.repo
# append:
[cloudera-repo]
name=cloudera-repo
baseurl=http://utility01.cdh.lionxcat.com/cloudera-repos/cm6
enabled=1
gpgcheck=0

# 确认配置正确
yum repolist
# 分发到c7
scp /etc/yum.repos.d/cloudera-repo.repo root@c7:/etc/yum.repos.d/
```

### 4.5 安装依赖软件包

*以下内容在c7和ut01上都执行，安装必备的软件包。*

#### 4.5.1 检查Python

参考-[Software Dependencies](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_os_requirements.html#concept_tlf_xnv_22b)

```bash
# 若CentOS选择计算节点安装则默认都满足
# CentOS7原装了python 2.7.*
python --version
# 其他软件包缺失参考文章自行安装
```

#### 4.5.2 安装JDK

参考-[Java Requirements](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html)

推荐使用CM包里的JDK版本安装，或者自行安装Oracle JDK或OpenJDK。

- 安装CM包里的JDK版本

```bash
yum install -y oracle-j2sdk1.8

vi /etc/profile
# append:
JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# 使之生效并测试
source /etc/profile
java -version
```

- *可选：自行安装Oracle JDK*

```bash
# 若centos自带OpenJDK，删除之
# java -version
# rpm -qa | grep java
# rpm -e --nodeps java-1.8.0-openjdk-...
# 下载JDK
tar -zxvf jdk-8u202-linux-x64.tar.gz -C /usr/java/

vi /etc/profile
# append:
JAVA_HOME=/usr/java/jdk1.8.0_202
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# 使之生效并测试
source /etc/profile
java -version
```

#### 4.5.3 安装MySQL-JDBC

参考-[Installing the MySQL JDBC Driver](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_mysql.html#cmig_topic_5_5_3)

*若使用PG、MariaDB等其他DB自行参考文章安装。*

```bash
wget https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-5.1.49.tar.gz
tar zxvf mysql-connector-java-5.1.49.tar.gz
# 最好放置到CM默认的目录下
mkdir -p /usr/share/java
# 最好更改为CM默认的文件名
cp mysql-connector-java-5.1.49/mysql-connector-java-5.1.49-bin.jar /usr/share/java/mysql-connector-java.jar
```

### 4.6 安装CM软件包

*在ut01上执行，安装Cloudera Manager Server，Daemons，Agent。*

```bash
yum install -y cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```

*以下操作在c7上执行，安装CM Daemons & Agent。*

可选：若现在不安装，今后让CMServer自动安装也行。

```bash
yum install -y cloudera-manager-daemons cloudera-manager-agent

# 修改agent配置文件，指向CM Server
vi /etc/cloudera-scm-agent/config.ini
# modify:
# server_host=utility01.cdh.lionxcat.com
```

### 4.7 数据库准备

*以下操作在ut01上执行，若MySQL使用云SaaS服务或规划在其他VM，依据实际情况操作。*

参考-[Install and Configure MySQL for Cloudera Software](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_mysql.html#cmig_topic_5_5)

#### 4.7.1 安装MySQL

参考-[MySQL Community Downloads](https://dev.mysql.com/downloads/)

- 可以直接从MySQL官方下载，300~500MB，需要网速够快

```bash
# 下载官方yum源
wget https://repo.mysql.com/mysql80-community-release-el7-3.noarch.rpm
# 安装yum源
yum localinstall -y mysql80-community-release-el7-3.noarch.rpm
# 更换mysql8.0为mysql5.7
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community
# 查看更换成功否
yum repolist all | grep mysql | grep enabled
# 安装mysql
yum install -y mysql-community-server
# 修改mysql配置文件，参考官方文档配置即可，非生产环境非必须，略
# vi /etc/my.cnf
# 启动mysql
systemctl start mysqld
systemctl enable mysqld
# 找到初始密码
grep 'temporary password' /var/log/mysqld.log
# 执行脚本重置
/usr/bin/mysql_secure_installation
# 设置一个带有大小写字母数字符号的root密码（可能需要输入4遍）然后一路y下去即可
```

- *可选：可提前下载好包，直接安装*

```bash
# 下载包
wget https://cdn.mysql.com/Downloads/MySQL-5.7/mysql-5.7.34-1.el7.x86_64.rpm-bundle.tar
# 解压缩
tar xvf mysql-5.7.34-1.el7.x86_64.rpm-bundle.tar
# 安装
rpm -ivh mysql-community-common-*.rpm mysql-community-libs-*.rpm mysql-community-server-*.rpm mysql-community-client-*.rpm
# 配置与启动等后续步骤同上
```

#### 4.7.2 创建各服务需要的DB

官方文档中有描述所有需要预先创建的DB。utf8编码是必须的。
虽然官方允许配置各服务使用任意DB名和User名，但遵守官方默认规范也不复杂，若图方便可使用相同User和Password。
新建User密码均需要满足密码复杂度要求。也可自行修改配置文件降低要求。

```bash
# 在ut01中执行
mysql -uroot -p
```

```sql
/* 在MySQL CLI中执行 */

CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'scm'@'%' IDENTIFIED BY 'scm_Passw0rd';
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'scm_Passw0rd';

CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'amon'@'%' IDENTIFIED BY 'amon_Passw0rd';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'amon_Passw0rd';

CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'hue'@'%' IDENTIFIED BY 'hue_Passw0rd';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'hue_Passw0rd';

CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive_Passw0rd';
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive_Passw0rd';

CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'oozie'@'%' IDENTIFIED BY 'oozie_Passw0rd';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie_Passw0rd';

FLUSH PRIVILEGES;
```

#### 4.7.3 生成CMServer的DB配置

参考-[Set up the Cloudera Manager Database](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/prepare_cm_database.html)

```bash
# 参数含义：<databaseType> <databaseName> <databaseUser> <password>
# 依据实际情况修改参数
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm scm_Passw0rd
```

### 4.8 启动CMServer

*以下操作在ut01上执行。*

```bash
systemctl start cloudera-scm-server
tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log

systemctl start cloudera-scm-agent
tail -f /var/log/cloudera-scm-agent/cloudera-scm-agent.log

systemctl enable cloudera-scm-server
systemctl enable cloudera-scm-agent
```

*以下操作在c7上执行。*

```bash
systemctl start cloudera-scm-agent
systemctl enable cloudera-scm-agent
```

在宿主机访问http://ut01:7180/，用户名/密码：admin。
如果ut01和c7上的配置均成功，就可以在配置页面看到两台VM。

**注意：**
因后续操作将复制c7生成其余VM，若在c7上启动过cloudera-scm-agent，则**必须在复制c7前删除agent生成的uuid**，agent会在下次启动时重新生成。若不重新生成uuid，**各VM的uuid重复将导致今后server无法识别各agent**。
若没有在c7上启动agent则无此需要。

```bash
systemctl stop cloudera-scm-agent
rm -f /var/lib/cloudera-scm-agent/uuid
# 不能再启动Agent，让新的VM各自启动各自的以生成不同uuid。
```

## 5. 创建CDH集群

操作与创建ut01的VM类似，只不过这次生成的不是ut01，而是其余所有VM。

### 5.1 备份VM

先将两台VM关机。并做些备份操作，以免后面操作出错，或方便配置第二套集群。

```powershell
# 在宿主机上执行
# 删除所有快照以合并磁盘文件
Remove-VMSnapshot c7
Remove-VMSnapshot ut01
# 因为c7本身就是CM-Agent所在VM的备份，因此只需要再备份ut01即可
$BasePath = 'D:\VirtualMachines\CDH-Cluster'
New-Item "$BasePath\backup" -ItemType Directory
Copy-Item "$BasePath\ut01\ut01.vhdx" "$BasePath\backup"
```

### 5.2 生成其余VM

下面采用复制VHD的方式，复制c7的VHD用于创建其余VM。

```powershell
# VM基础配置
$BasePath = 'D:\VirtualMachines\CDH-Cluster'
$Switch = 'cdh-vmswitch'
# 复制源
$SourceVM = 'c7'
$SourceVHD = "$BasePath\$SourceVM\$SourceVM.vhdx"
# 复制目的地VM名集合
$DestVMColl = ('m01', 'm02', 'm03', 'n001', 'n002', 'n003', 'gw01')

# 删除c7所有快照以合并磁盘文件
Remove-VMSnapshot $SourceVM

# 循环创建所有VM
ForEach ($DestVM in $DestVMColl) {
    $DestPath = "$BasePath\$DestVM"
    $DestVHD = "$DestPath\$DestVM.vhdx"
    # 将文件拷贝到新目录
    New-Item $DestPath -ItemType Directory
    Copy-Item $SourceVHD $DestVHD
    # 创建新VM使用复制后的文件
    # 注意自行设定启动内存大小
    $NewVM = @{
        Name = $DestVM
        MemoryStartupBytes = 16GB
        SwitchName = $Switch
        VHDPath = $DestVHD
        Path = $BasePath
        Generation = 1
    }
    New-VM @NewVM
    # 设置CPU
    Set-VMProcessor $DestVM -Count 4
}

# 确认创建无误，可以继续配置

# 可选：创建一个VMGroup把这些VM都加进去
$GroupName = 'lionxcat-cdh'
New-VMGroup -Name $GroupName -GroupType VMCollectionType
$DestVMColl += (,'ut01')
ForEach ($DestVM in $DestVMColl) {
    $VM = Get-VM $DestVM
    Add-VMGroupMember $GroupName $VM
    # 慎重批量启动VM
    # Start-VM @VM
}
```

注意：
- 建议一个一个手工启动VM并执行后续配置，以免宿主机内存不够用最后有些VM启动不起来。
- Hyper-V会在VM启动时为其分配<MemoryStartupBytes>大小的内存，在VM运行3~4分钟后动态内存收缩，此时宿主机内存占用下降可再启新VM。
- 如果最终还是无法启动所有VM，要么调小启动内存（可能导致今后有的组件内存不足无法启动），要么买内存条插上。
- 到此步骤为止，8台VM运行时内存总和约不到17GB。
- c7的作用就到此结束了，可以和ut01的备份VHD一起放置到其他地方备份。

### 5.3 配置其余VM

*以下操作在新生成的VM上执行。*

启动并配置IP和Hostname。启动VM命令略。

```bash
vi /etc/sysconfig/network-scripts/ifcfg-eth0 # eth0是具体VM网卡
# append: IPADDR=10.10.64.xxx

hostnamectl set-hostname xxx.cdh.lionxcat.com

vi /etc/sysconfig/network
# modify: HOSTNAME=xxx.cdh.lionxcat.com

systemctl restart network
# 重启agent
systemctl restart cloudera-scm-agent
# 这里再提醒一遍，若复制VM前启动过scm-agent，需要删除
# /var/lib/cloudera-scm-agent/uuid，并重启agent才能顺利被scm-server识别
```

### 5.4 为NameNode和DataNode的VM创建并挂载磁盘VHD

可选：为NN和DN单独挂载可扩展的虚拟磁盘，方便今后随时增大磁盘空间或挪动磁盘文件到更大的硬盘上。
若不放在可扩展VHD上，今后空间不够用了再扩充比较麻烦。
新挂载的VHD也创建成LVM，方便今后在线扩容。

参考[centos7 xfs磁盘管理(格式化、在线扩容)](https://blog.51cto.com/daisywei/1960761)

创建并向VM添加动态VHD。

```powershell
# 在宿主机上执行
$BasePath = 'D:\VirtualMachines\CDH-Cluster'
$VMColl = ('m01', 'm02', 'm03', 'n001', 'n002', 'n003')
ForEach ($VM in $VMColl) {
    $NewVHD = "$BasePath\$VM\$VM-data.vhdx"
    New-VHD -Path $NewVHD -Dynamic -SizeBytes 500GB
    # 在VM启动状态下可执行，挂载SCSI设备，若SCSI-Controller(0,0)被占用，自行调整位置
    Add-VMHardDiskDrive $VM -Path $NewVHD -ControllerNumber 0 -ControllerLocation 0 -ControllerType SCSI
}
```

在后面安装CDH时，DN和NN的数据目录(dfs.datanode.data.dir和dfs.namenode.name.dir)默认在/dfs下。
若更换其它位置，则下面命令里也应该挂载到对应目录下。

*以下操作在m0[1-3]和n00[1-3]上执行。*

在VM中挂载硬盘到目录/dfs。

```bash
# 查看新挂载的VHD是否成功，正常情况应被挂载到设备sdb
fdisk -l
# 从物理磁盘创建Physical Volume
pvcreate /dev/sdb
pvs
# 从PV创建Volume Group，名称为vg-data
vgcreate vg-data /dev/sdb
vgs
# 从VG创建Logical Volume，名称为lv-data
lvcreate -l 100%FREE -n lv-data vg-data
lvs
# 格式化成xfs，注意路径和名称相对应
mkfs.xfs /dev/vg-data/lv-data
# 挂载到/dfs，若非该目录则需调整
mkdir /dfs
mount /dev/vg-data/lv-data /dfs
mount -l

# 设置开机自动挂载，注意，一定要添加在文件末尾
vi /etc/fstab
# append:
/dev/vg-data/lv-data /dfs                       xfs     defaults        0 0

# 检查一下
vgdisplay vg-data
```

### 5.5 安装CDH

如果上述配置无误，这是最简单的一步，很快就能完成。
在宿主机访问http://ut01:7180/，用户名/密码：admin。

安装注意事项：
1. 若在"Specify Hosts"中没有看到“当前管理的主机”中出现某台VM，请检查该VM的IP/Hostname配置，并查看scm-server的日志排查问题。
2. 在“选择存储库--使用Parcel（建议）--更多选项”中，添加使用自建的私有仓库，地址为：http://utility01.cdh.lionxcat.com/cloudera-repos/cdh6/ 。然后会在“CDH版本”里出现"CDH-6.3.2-xxx"。
3. 在"Inspect Cluster"中，运行两项测试，若无问题，则说明上述配置一切正常。
4. 选择基本安装"Essentials"，以免安装服务太多了内存不足启动不起来。
5. 按照规划选取各Host对应的Service，输入创建好的DB名称、用户、密码，最后执行命令。
6. 若有服务启动失败，查看是否内存不足导致。如果内存不足，可以：
    - 关停一些暂时不需要的服务，如Oozie，HUE，甚至把CM都关了只用Hadoop
    - 修改服务配置中的Java Heap大小
    - 增大CentOS的Swap
    - 去买内存条。

本人情况，将Hive JVM堆栈默认配置为从4G降为2G，客户端从2G降为1G后，所有服务均顺利启动。

启动后，各服务器占用宿主机的内存使用情况如下（因角色放置不同，数据量不同，运行时长不同，下面数据供参考）。 

|       FQDN / Contents      | short name | Mem Used | Total Disk |
| -------------------------: | :--------: | -------: | ---------: |
|  master01.cdh.lionxcat.com |     m01    |    9.2GB |     10.4GB |
|  master02.cdh.lionxcat.com |     m02    |    5.9GB |     10.4GB |
|  master03.cdh.lionxcat.com |     m03    |    3.5GB |      9.4GB |
| utility01.cdh.lionxcat.com |    ut01    |   12.2GB |     16.8GB |
| gateway01.cdh.lionxcat.com |    gw01    |    6.3GB |      9.4GB |
|   node001.cdh.lionxcat.com |    n001    |    5.0GB |     11.8GB |
|   node002.cdh.lionxcat.com |    n002    |    5.0GB |     11.8GB |
|   node003.cdh.lionxcat.com |    n003    |    5.0GB |     11.8GB |
|     c7 & ut01 backup files |      -     |        - |     14.7GB |
|       CDH/CentOS/MySQL/JDK |      -     |        - |     13.5GB |
|                  **TOTAL** | **8 hosts**| **53GB** |  **120GB** |

### 5.6 安装后续

列举一些可以拿这套集群玩点什么的想法。

- 尝试自己写MR，爬点数据放入集群内玩，用eCharts做些BI等等。
- 优化各组件的参数和JVM配置，升高或降低。
- 装Spark，HBase，Hive on Spark等等，但需要**更多内存**。
- 启用HDFS的HA，Yarn和Hive的负载均衡等等，只为理解架构，非生产环境并无实际意义。
- 用HBase做索引，建立自己的照片和媒体库索引，用Hive+Hue做可视化。
- Flink，Hudi ...

## 6. 回顾

整体花费一周时间，中间临时买了两根内存条。
前期规划和查找各类文章花费50%，学习Hyper-V和PowerShell花费30%，安装配置花费15%，整理文档花费15%。
中间重装了两次，只要配置好了c7和ut01两台VM，后面的操作也就是十几分钟搞完。抛开Hyper-V其实很简单。
流程还有可优化的地方，但也没必要浪费时间了。
内存越多越好，个人认为32GB是最低配，但可能需要花费精力折腾并不划算。64GB基本够用，毕竟自己玩不会太多数据文件。我选择了128GB。
后面会将HDP的搭建也整理个文章出来，再往后有精力再尝试搭建Apache原生。

如有勘误、改进意见或自己搭建过程中的疑问，欢迎联系[lionxcat](mailto:7345445@qq.com)或留言。
转载请注明原文[https://github.com/lionxcat/articles-it/blob/main/cdh-setup-guide-using-hyperv-by-yourself.md](https://github.com/lionxcat/articles-it/blob/main/cdh-setup-guide-using-hyperv-by-yourself.md)。