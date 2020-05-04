# 华为交换机常用命令

## 视图分类

### 用户视图

登陆设备后，直接进入用户模式，只能执行少量查看配置的命令

```
Info: The max number of VTY users is 10, and the number
      of current VTY users on line is 1.
      The current login time is 2020-04-10 12:15:00+00:00.
<ZD_agg_5720>
```

### 系统视图

用户模式下，输入system-view命令进入视图模式，可执行设备全局配置的命令

```
<ZD_agg_5720>system-view 
Enter system view, return user view with Ctrl+Z.
[ZD_agg_5720]
```

### 局部视图

系统视图模式下，输入局部配置命令，进入局部对像的配置视图。如interface GE 1/0/0，进入GE1/0/0端口配置模式

```
<ZD_agg_5720>system-view 
Enter system view, return user view with Ctrl+Z.
[ZD_agg_5720]interface GigabitEthernet 1/0/1
[ZD_agg_5720-GigabitEthernet1/0/1]
```

## 常用命令

### 命令简介

| 命令        | 缩写 | 解释                   |
| ----------- | ---- | ---------------------- |
| display     | dis  | 查看相应对象信息       |
| undo        |      | 撤消或反向操作对应命令 |
| system-view | sy   | 进入系统视图           |
| sysname     |      | 设置交换机名称         |
| quit        |      | 退出当前视图           |
| reboot      |      | 交换机重启             |
| reset       |      |                        |
| restart     |      | 重新启动当前接口       |
| shutdown    |      | 关闭当前接口           |

### 常用操作

#### 交换机信息查看

```
display version  查看交换机软件版本
display clock    查看交换机时钟
```

#### 配置查看

```
display saved-configuration    显示系统保存配置
display current-configuration  显示系统当前配置
```

#### 当前对象信息查看

```
display this include-default    显示当前接口视图下的接口信息，包括默认值。
display this interface          显示当前接口视图下的接口信息。
```

#### 查看接口

```
display interface               查看接口当前运行状态和接口统计信息
display interface brief         查看接口状态和配置的简要信息。
display interface description   查看指定接口的描述信息
display interface vlanif        查看VLANIF接口的状态信息、配置信息和统计信息。
```

#### 查看IP相关

```
display ip interface             查看接口与IP相关的配置和统计信息，包括接口接收和发送的报文数、字节数和组播报文数，以及接口接收、发送、转发和丢弃的广播报文数。
display ip interface brief       看接口与IP相关的简要信息，包括IP地址、子网掩码、物理链路和协议的Up/Down状态以及处于不同状态的接口数目。
display ip interface description 查看接口与IP相关的简要信息，包括IP地址、子网掩码、物理层状态、链路层协议状态，及接口描述信息和处于不同状态的接口数目。
```

```
display ip routing-table  显示路由信息
display ip pool                                                          显示所有ip pool
display ip pool name {pool name} {all|conflict|expired|used}             显示ip pool详细信息
display ip host                查看静态DNS表项
display ip socket              查看已创建的IPv4 Socket信息。
display ip statistics          显示IP流量统计信息。
```

#### 网络及流量

```
display network status { all|tcp|udp|port port-number }   显示IP流量统计信息 
display tcp statistics   查看TCP流量统计信息
display udp statistics   查看UDP流量统计信息
```

#### VLAN查看

```
display vlan                              显示VLAN信息
display vlan {pvid} verbose               查看vlan的详细信息
display port vlan                         查看VLAN中包含的接口信息
display sub-vlan                          查看Sub-VLAN类型的VLAN表项信息
display super-vlan                        查看Super-VLAN类型的VLAN表项信息
display mac-vlan mac-address all          查看所有MAC地址划分VLAN的配置信息
display mac-vlan vlan 2                   查看vlan 2 MAC地址划分VLAN的配置信息
```

#### 端口管理

```
port                                               配置接口的缺省VLAN并加入该VLAN
port description                                   配置接口的描述信息，描述与接口相连的设备类型。
port gigabitethernet 0/0/1 to 0/0/4
port default vlan                                  配置接口的缺省VLAN并同时加入这个VLAN。
port link-type {access | hybird | trunk}           配置接口的链路类型
```

#### 端口配置

```
speed  {10|100|auto}                             配置端口工作速率
duplex {half|full|auto}                          配置端口工作状态
```

#### 端口组操作

```
display port-group  {all}                                     查看端口组
port-group {id}                                               创建端口组
group-member gigabitethernet 0/0/2 to gigabitethernet 0/0/10  将2到10端口加入端口组
```

#### VLAN管理

```
vlan {id}                      创建VLAN并进入VLAN视图，如果VLAN已存在，直接进入该VLAN的视图。
vlan batch 10 to 20            批量创建VLAN
vlan range 10 to 20            创建临时VLAN组，并进入VLAN-Range视图
vlan statistics                配置VLAN的流量统计模式，即配置按包或按字节进行VLAN流量统计。
vlan statistics interval       配置VLAN的流量统计的时间间隔
ip address                     用来配置接口的IP地址。 
```

#### 接口管理

```
interface gigabitethernet 0/0/1     进入指定的接口或子接口视图，进入0/0/1的接口
```

#### DNS管理

```shell
#查看
display dns dynamic-host       查看动态DNS表项
display dns domain             查看域名后缀的相关信息
display dns server             查看DNS服务器的相关信息
#设置
dns domain domain-name    命令用来配置域名后缀，如 dns domain com.cn。
dns resolve               命令用来使能动态域名解析功能
dns server {ip}           命令用来配置DNS服务器的IP地址
ip host {domain} {ip}     命令用来配置静态DNS表项 ip host www.huawei.com 10.10.10.4。
```

#### 用户管理

```shell
#查看本地用户
display local-user
#查看用户接口
display user-interface
#设置用户vty0为4个并发
user-interface vty 0 4
#进入用户console0接口
user-interface console 0
#用户管理
local-user {username} password cipher {password}
local-user {username} level 15
local-user {username} service-type telnet terminal ssh
```

#### 绑定IP与MAC

```
user-bind ip-address 10.0.0.2 mac-address 0001-0203-0405
user privilege level 3
```

LEVEL 0(访问级)：可以执行用于网络诊断等功能的命令。包括ping、tracert、telnet等命令，执行该级别命令的结果不能被保存到配置文件中。
LEVEL 1(监控级)：可以执行用于系统维护、业务故障诊断等功能的命令。包括debugging、terminal等命令，执行该级别命令的结果不能被保存到配置文件中。
LEVEL 2(系统级)：可以执行用于业务配置的命令，主要包括路由等网络层次的命令，用于向用户提供网络服务。
LEVEL 3(管理级)：最高级，可以运行所有命令：关系到系统的基本运行、系统支撑模块功能的命令，这些命令对业务提供支撑作用。包括文件系统、FTP、TFTP、XModem下载、用户管理命令、级别设置命令等。

#### 日志与统计

```
#打开统计
display counters               查看接口的流量统计计数
statistic enable
trace mac enable
trace mac aa99-6600-5600 vlan 2
```

#### 其它命令

```
display stp                   显示生成树信息
display mac-address           显示MAC地址表
display bridge mac-address    查看当前桥接设备mac地址
display arp                   显示ARP信息表
display acl all               查看所有acl规则
display voice-vlan oui                          查看Voice VLAN的OUI及其相关属性。
display voice-vlan status                       查看当前Voice VLAN的相关信息

mac-vlan mac-address
dhcp enable                          命令用来开启DHCP功能。 
```



### 操作实战

#### VLAN操作

创建vlan，设置vlan的ip，并将端口加入vlan中。

```shell
#进入全局配置视图
<Huawei> system-view
#新建vlan2
[Huawei] vlan 2
#进入vlan2的接口视图
[Huawei] interface vlan 2
#设置vlan2的三层网关路由
[Huawei-Vlanif2] ip address 10.0.0.1 255.255.255.0
#进入0/0/1接口，将该接口加入vlan2中
[Huawei] interface GigabitEthernet 0/0/1
[Huawei-GigabitEthernet0/0/1] port link-type access
[Huawei-GigabitEthernet0/0/1] port default vlan 2
[Huawei-GigabitEthernet0/0/1] dis this
#进入vlan2，将0/0/2到0/0/5端口加入到vlan2中（port link-type需要是access类型）
[Huawei] vlan 2
[Huawei-vlan2] port GigabitEthernet 0/0/2 to 0/0/5 
```

#### 端口组操作

需要对多个端口进行相同操作时，可以创建端口组进行批量操作。

```shell
#创建端口组1，进行批量端口操作
[Huawei] port-group 1
#将6到10端口加入端口组
[Huawei-port-group-1] group-member gigabitethernet 0/0/6 to gigabitethernet 0/0/10
#批量将2~10端口修改为access模式
[Huawei-port-group-1] port link-type access
#批量将6~10端口加入到vlan 2中
[Huawei-port-group-1] port default vlan 2
```

#### IP POOL操作

```shell
<Huawei> system-view
#进入ip pool中
[Huawei] ip pool vlan10
#在ip pool中添加dns server
[Huawei-ip-pool-vlan10] dns-list 8.8.8.8
#设置网关
[Huawei-ip-pool-vlan10] gateway-list 10.0.0.1
#设置ip pool的ip段和池，需要与vlanif接口配置匹配，否则ip分配不了
[Huawei-ip-pool-vlan10] network 10.0.0.0 mask 255.255.255.0
#过期时间
[Huawei-ip-pool-vlan10] lease day 10
```

#### DHCP操作

```shell
<Huawei> system-view
#全局打开DHCP服务
[Huawei] dhcp enable
#进入vlan的接口视图
[Huawei] interface vlanif 2
#设置vlan2的三层网关路由
[Huawei-Vlanif2] ip address 10.0.0.1 255.255.255.0
#从全局配置中获取dhcp配置
[Huawei-Vlanif2] dhcp select global
```

#### 添加路由

```shell
<Huawei> system-view
<Huawei> display ip routing-table
#ip route-static 目的ip  目标地址掩码  下一跳ip
[Huawei] ip route-static 10.0.1.0 255.255.255.0 10.0.0.1
#删除路由
[Huawei] undo ip route-static 10.0.1.0 255.255.255.0 10.0.0.1
```

#### 用户管理 

```shell
#查看本地用户
<Huawei> display local-user
<Huawei> system-view
#进入aaa配置模式
[Huawei] aaa
#配置本地用户账号密码
[Huawei-aaa] loca-user user1 password cipher 123456
#用户服务类型为telnet，使用telnet登陆
[Huawei-aaa] local-user user1 service-type telnet
#配置用户特权等级15
[Huawei-aaa] local-user user1 privilege level 15
```



