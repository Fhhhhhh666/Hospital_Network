**步骤一：配置【内网核心A】与【内网核心B】的聚合与向下 Trunk**



**内网核心A:**

enable

configure terminal

hostname Core-A\_Inner



! 1. 全局创建 VLAN 数据库

vlan 10

exit

vlan 20

exit

vlan 30

exit

vlan 40

exit

vlan 100

exit

do show vlan brief

! 2. 配置与核心B之间的链路聚合 (Fa0/1 和 Fa0/2)

interface range FastEthernet 0/1 - 2

&#x20;channel-group 1 mode active

&#x20;no shutdown

&#x20;exit

do show etherchannel summary

! 3. 配置聚合逻辑口 (Port-channel 1) 为 Trunk

interface Port-channel 1

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit

do show interfaces trunk

! 4. 配置向下连接汇聚交换机 (Switch0, Switch2) 以及 WLC 的接口为 Trunk

! 端口为 Fa0/3, Fa0/6, Fa0/7

interface range FastEthernet 0/3  , FastEthernet 0/6-7

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit

do show interfaces trunk



**内网核心B:**

enable

configure terminal

hostname Core-B\_Inner



! 1. 全局创建 VLAN 数据库

vlan 10

exit

vlan 20

exit

vlan 30

exit

vlan 40

exit

vlan 100

exit

do show vlan brief

! 2. 配置与核心A之间的链路聚合

interface range FastEthernet 0/1 - 2

&#x20;channel-group 1 mode active

&#x20;no shutdown

&#x20;exit

do show etherchannel summary

! 3. 配置聚合逻辑口为 Trunk

interface Port-channel 1

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit

do show interfaces trunk

! 4. 配置向下连接汇聚交换机的接口为 Trunk (Fa0/3, Fa0/4)

interface range FastEthernet 0/3 - 4,FastEthernet 0/7

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit

do show interfaces trunk





**步骤二：配置【中间汇聚层】的 Trunk 穿透**

**Multilayer Switch0:**

enable

configure terminal

hostname Dist\_SW0\_Left



! 1. 同步创建 VLAN 数据库

vlan 10

exit

vlan 20

exit

vlan 30

exit

vlan 40

exit

vlan 100

exit

vlan 99

&#x20;name AP\_Management

&#x20;exit

! 2. 向上和向下的接口统统配置为 Trunk

interface range FastEthernet 0/1 - 4

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit



**Multilayer Switch1:**

enable

configure terminal

hostname Dist\_SW2\_Right

! 1. 同步创建 VLAN 数据库

vlan 10

exit

vlan 20

exit

vlan 30

exit

vlan 40

exit

vlan 100

exit

vlan 99

&#x20;name AP\_Management

&#x20;exit

! 2. 向上和向下的接口统统配置为 Trunk

interface range FastEthernet 0/1 - 4

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit



**三层网关与 HSRP 双机热备:**

**内网核心A:**

enable

configure terminal

ip routing



! 1. 配置医生站 VLAN 10 网关

interface Vlan 10

&#x20;ip address 10.10.10.252 255.255.254.0

&#x20;standby 10 ip 10.10.10.254

&#x20;standby 10 priority 110

&#x20;standby 10 preempt

&#x20;no shutdown

&#x20;exit

do show standby brief

! 2. 配置护士站 VLAN 20 网关

interface Vlan 20

ip address 10.10.20.252 255.255.255.0

&#x20;standby 20 ip 10.10.20.254

&#x20;standby 20 priority 110

&#x20;standby 20 preempt

&#x20;no shutdown

&#x20;exit

! 3. 配置医疗设备 VLAN 40 网关

interface Vlan 40

ip address 172.16.40.252 255.255.255.0

&#x20;standby 40 ip 172.16.40.254

&#x20;standby 40 priority 110

&#x20;standby 40 preempt

&#x20;no shutdown

&#x20;exit



**内网核心B:**

enable

configure terminal

ip routing



! 1. 配置医生站 VLAN 10 备用网关

interface Vlan 10

&#x20;ip address 10.10.10.253 255.255.254.0

&#x20;standby 10 ip 10.10.10.254

&#x20;standby 10 preempt

&#x20;no shutdown

&#x20;exit



! 2. 配置护士站 VLAN 20 备用网关

interface Vlan 20

&#x20;ip address 10.10.20.253 255.255.255.0

&#x20;standby 20 ip 10.10.20.254

&#x20;standby 20 preempt

&#x20;no shutdown

&#x20;exit



! 3. 配置医疗设备 VLAN 40 备用网关

interface Vlan 40

&#x20;ip address 172.16.40.253 255.255.255.0

&#x20;standby 40 ip 172.16.40.254

&#x20;standby 40 preempt

&#x20;no shutdown

&#x20;exit

&#x20;**STP :**

**内网核心 A:**



enable

configure terminal

! 强行指定核心A为所有VLAN的绝对老大 (主根桥)

spanning-tree vlan 10,20,30,40,100 root primary

exit





**内网核心 B:**



enable

configure terminal



! 指定核心B为备用老大，如果核心A死了，核心B不仅接管网关，也接管STP根桥

spanning-tree vlan 10,20,30,40,100 root secondary

exit





**配置核心交换机连接防火墙的互联 IP**

**内网核心 A:**

enable

configure terminal



! 1. 配置连接出口 防火墙 ASA1 的接口 (假设为 Fa0/5)

interface FastEthernet 0/5

&#x20;! 关键指令：将二层交换口变成三层路由口

&#x20;no switchport

&#x20;! 配置 /30 的互联 IP

&#x20;ip address 10.254.254.1 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接数据中心 防火墙 ASA2 的接口 (假设为 Fa0/8)

interface FastEthernet 0/4

&#x20;no switchport

&#x20;ip address 10.254.254.5 255.255.255.252

&#x20;no shutdown

&#x20;exit



**内网核心 B:**

enable

configure terminal



! 1. 配置连接出口 防火墙 ASA1 的接口

interface FastEthernet 0/6

&#x20;no switchport

&#x20;ip address 10.254.254.9 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接数据中心 防火墙 ASA2 的接口

interface FastEthernet 0/5

&#x20;no switchport

&#x20;ip address 10.254.254.13 255.255.255.252

&#x20;no shutdown

&#x20;exit



**配置防火墙**

**ASA1:**

enable

configure terminal

hostname ASA1\_Exit



! 1. 配置连接【内网核心 A】的接口 (Gig1/1)

interface GigabitEthernet 1/1

&#x20;! 命名为 inside\_coreA (内部区域)

&#x20;nameif inside\_coreA

&#x20;! 设置最高信任级别 100 (代表这是绝对安全的内网)

&#x20;security-level 100

&#x20;! 配置 /30 互联 IP，对应核心 A 的 .1

&#x20;ip address 10.254.254.2 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接【内网核心 B】的接口 (Gig1/2)

interface GigabitEthernet 1/2

&#x20;nameif inside\_coreB

&#x20;security-level 100

&#x20;! 对应核心 B 的 .9

&#x20;ip address 10.254.254.10 255.255.255.252

&#x20;no shutdown

&#x20;exit



**ASA2:**

enable

configure terminal

hostname ASA2\_DC



! 1. 配置连接【内网核心 A】的接口

interface GigabitEthernet 1/1

&#x20;nameif inside\_coreA

&#x20;security-level 100

&#x20;! 对应核心 A 的 .5

&#x20;ip address 10.254.254.6 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接【内网核心 B】的接口

interface GigabitEthernet 1/2

&#x20;nameif inside\_coreB

&#x20;security-level 100

&#x20;! 对应核心 B 的 .13

&#x20;ip address 10.254.254.14 255.255.255.252

&#x20;no shutdown

&#x20;exit



**启动OSPF动态路由协议**

**内网核心 A:**

enable

configure terminal



! 启动 OSPF 进程 1

router ospf 1

&#x20;! 1. 宣告向下的业务网段 (使用反掩码)

&#x20;network 10.10.10.0 0.0.1.255 area 0

&#x20;network 10.10.20.0 0.0.0.255 area 0

&#x20;network 172.16.40.0 0.0.0.255 area 0

&#x20;network 10.10.100.0 0.0.0.255 area 0

&#x20;

&#x20;! 2. 宣告向上的防火墙互联链路 (反掩码为 0.0.0.3)

&#x20;network 10.254.254.0 0.0.0.3 area 0

&#x20;network 10.254.254.4 0.0.0.3 area 0

&#x20;exit



**内网核心 B:**

enable

configure terminal



router ospf 1

&#x20;! 1. 宣告向下的业务网段 (与核心 A 完全一致，因为 HSRP 是共享的)

&#x20;network 10.10.10.0 0.0.1.255 area 0

&#x20;network 10.10.20.0 0.0.0.255 area 0

&#x20;network 172.16.40.0 0.0.0.255 area 0

&#x20;network 10.10.100.0 0.0.0.255 area 0

&#x20;

&#x20;! 2. 宣告向上的防火墙互联链路 (对应核心 B 自己的 /30 网段)

&#x20;network 10.254.254.8 0.0.0.3 area 0

&#x20;network 10.254.254.12 0.0.0.3 area 0

&#x20;exit



**ASA1:**

enable

configure terminal



router ospf 1

&#x20;! 注意：这里用的是正经的子网掩码 255.255.255.252！

&#x20;network 10.254.254.0 255.255.255.252 area 0

&#x20;network 10.254.254.8 255.255.255.252 area 0

&#x20;exit



**ASA2:**

enable

configure terminal



router ospf 1

&#x20;network 10.254.254.4 255.255.255.252 area 0

&#x20;network 10.254.254.12 255.255.255.252 area 0

network 10.10.100.0 255.255.255.0 area 0

&#x20;exit





**配置防火墙 ASA2 与数据中心的连接:**

**ASA2:**

enable

configure terminal



! 1. 进入连接服务器交换机的接口

interface GigabitEthernet 1/3

&#x20;! 命名为 server\_zone (或 inside)

&#x20;nameif server\_zone

&#x20;! 设置最高安全级别 100

&#x20;security-level 100

&#x20;! 配置数据中心网段的网关 IP

&#x20;ip address 10.10.100.254 255.255.255.0

&#x20;no shutdown

&#x20;exit



**配置防火墙 ASA1 与卫健委的连接:**

**ASA1:**

enable

configure terminal



! 1. 配置连接路由器的物理接口

interface GigabitEthernet 1/3

&#x20;! 命名为政务外网区域

&#x20;nameif outside\_gov

&#x20;! 安全级别 50 (内网是100，互联网是0，政务网居中)

&#x20;security-level 50

&#x20;! 配置 /30 互联 IP

&#x20;ip address 10.254.254.17 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 编写静态路由，告诉防火墙医保局的服务器在哪里

! 假设医保局/卫健委的网段全都在 10.100.x.x 这个大网段里

route outside\_gov 10.100.0.0 255.255.0.0 10.254.254.18



**配置边界路由器 Router0**

enable

configure terminal

hostname Router0\_GovLine



! 1. 配置连接防火墙 ASA1 的接口

interface GigabitEthernet 0/0

&#x20;ip address 10.254.254.18 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接卫健委/医保局专线光纤的接口

interface GigabitEthernet 0/1

&#x20;ip address 10.100.1.1 255.255.255.0

&#x20;no shutdown

&#x20;exit



! 3. 编写回程路由：告诉路由器，医院的内网怎么走

! 将所有去往 10.x.x.x (内网医生、HIS等) 的流量，全扔给防火墙的 .17

ip route 10.0.0.0 255.0.0.0 10.254.254.17



! 4. 编写默认路由：告诉路由器，凡是去卫健委的，全扔给远端政务网关

ip route 0.0.0.0 0.0.0.0 10.100.1.254





**在核心交换机配置 DHCP 与 AP 管理网段：**

**内网核心 A：**

enable

configure terminal



! 1. 创建专门用于 AP 管理的 VLAN 99

vlan 99

&#x20;name AP\_Management

&#x20;exit



! 2. 配置 VLAN 99 的网关

interface Vlan 99

&#x20;ip address 10.10.99.252 255.255.255.0

&#x20;standby 99 ip 10.10.99.254

&#x20;standby 99 preempt

&#x20;no shutdown

&#x20;exit



interface FastEthernet 0/3

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;switchport trunk native vlan 99

&#x20;exit





! 3. 将 VLAN 99 宣告进 OSPF (让全网可达)

router ospf 1

&#x20;network 10.10.99.0 0.0.0.255 area 0

&#x20;exit



! 4. 核心加分项：开启 DHCP 服务并配置地址池

! 为 AP 配置地址池

ip dhcp pool AP\_POOL

&#x20;network 10.10.99.0 255.255.255.0

&#x20;default-router 10.10.99.254

&#x20;! 【高阶指令】Option 43：告诉 AP，控制器的 IP 是 10.10.99.250

&#x20;option 43 ip 10.10.99.250

&#x20;exit

ip dhcp excluded-address 10.10.99.250 10.10.99.254

ip dhcp excluded-address 172.16.40.250 172.16.40.254







! 为护士的 PDA (VLAN 40) 配置地址池

ip dhcp pool PDA\_POOL

&#x20;network 172.16.40.0 255.255.255.0

&#x20;default-router 172.16.40.254

&#x20;exit



! 为临床医生站 (VLAN 10) 配置 DHCP

ip dhcp pool Doctor\_POOL

&#x20;network 10.10.10.0 255.255.254.0

&#x20;default-router 10.10.10.254

&#x20;dns-server 114.114.114.114

&#x20;exit

ip dhcp excluded-address 10.10.10.250 10.10.10.254



! 为护理与分诊 (VLAN 20) 配置 DHCP

ip dhcp pool Nurse\_POOL

&#x20;network 10.10.20.0 255.255.255.0

&#x20;default-router 10.10.20.254

&#x20;dns-server 114.114.114.114

&#x20;exit

ip dhcp excluded-address 10.10.20.250 10.10.20.254





**内网核心 B：**



enable

configure terminal



! 1. 创建 AP 管理 VLAN



! 2. 配置 VLAN 99 的备用网关

interface Vlan 99

&#x20;ip address 10.10.99.253 255.255.255.0

&#x20;standby 99 ip 10.10.99.254

&#x20;standby 99 preempt

&#x20;no shutdown

&#x20;exit



! 3. 将 VLAN 99 宣告进 OSPF 动态路由

router ospf 1

&#x20;network 10.10.99.0 0.0.0.255 area 0

&#x20;exit



interface FastEthernet 0/7  ! 假设这是连接WLC的接口

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;switchport trunk native vlan 99  ! 关键：让管理/AP的CAPWAP报文无标签通过

&#x20;exit



**门诊内网：**

enable

configure terminal



! 1. 在底层交换机上创建 VLAN 99

vlan 99

&#x20;name AP\_Management

&#x20;exit



! 2. 将连接 AP 的接口划入 VLAN 99，并开启边缘端口快速上线

interface FastEthernet 0/4

&#x20;switchport mode access

&#x20;switchport access vlan 99

&#x20;spanning-tree portfast

&#x20;exit



enable

configure terminal



! 确保交换机里有 VLAN 10

vlan 10

&#x20;name Doctor\_WS

&#x20;exit



! 将连接医生电脑的接口 (假设是 Fa0/2) 划入 VLAN 10

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 10

&#x20;spanning-tree portfast

&#x20;exit



vlan 20

&#x20;name Inpatient\_Dept

&#x20;exit

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 20

&#x20;spanning-tree portfast

&#x20;exit









**防火墙 NAT 与默认路由（打通去往卫健委和互联网的路）**

**ASA1:**



enable

configure terminal



! 1. 编写去往外网的默认路由（假设下一跳是 Router0 的 10.254.254.18）

route outside\_gov 0.0.0.0 0.0.0.0 10.254.254.18



! 2. 配置动态 NAT，让内网医生能访问卫健委系统

object network obj\_any

&#x20;subnet 0.0.0.0 0.0.0.0

&#x20;nat (inside\_coreA,outside\_gov) dynamic interface



! 3. 在 OSPF 中向内网核心下发默认路由

router ospf 1

&#x20;default-information originate

&#x20;exit



ACL 直接放行响应包（简单粗暴必通法）

access-list OUTSIDE\_IN extended permit icmp any any echo-reply

access-group OUTSIDE\_IN in interface outside\_gov





**绝对禁止访客网络（VLAN 50）访问 HIS/PACS 服务器，仅允许医生工作站（VLAN 10）访问**

enable

configure terminal



! 创建访问控制列表，允许医生网段，拒绝访客网段

access-list DC\_ACCESS extended permit ip 10.10.10.0 255.255.254.0 10.10.100.0 255.255.255.0

access-list DC\_ACCESS extended permit ip 10.10.99.0 255.255.255.0 10.10.100.0 255.255.255.0

access-list DC\_ACCESS extended deny ip 192.168.50.0 255.255.255.0 10.10.100.0 255.255.255.0

access-list DC\_ACCESS extended permit ip any any



! 将 ACL 应用到连接核心交换机的接口上

access-group DC\_ACCESS in interface inside\_coreA

access-group DC\_ACCESS in interface inside\_coreB





**住院部内网：**

enable

configure terminal



! 1. 在底层交换机上创建 VLAN 99

vlan 99

&#x20;name AP\_Management

&#x20;exit





interface FastEthernet 0/4

&#x20;switchport mode access

&#x20;switchport access vlan 99

&#x20;spanning-tree portfast

&#x20;exit



enable

configure terminal





vlan 10

&#x20;name Doctor\_WS

&#x20;exit





interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 10

&#x20;spanning-tree portfast

&#x20;exit



vlan 20

&#x20;name Inpatient\_Dept

&#x20;exit

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 20

&#x20;spanning-tree portfast

&#x20;exit



**医技与设备：**

enable

configure terminal



interface FastEthernet 0/1

&#x20;switchport mode trunk

&#x20;exit



! 1. 在底层交换机上创建 VLAN 99

vlan 99

&#x20;name AP\_Management

&#x20;exit



! 2. 将连接 AP 的接口划入 VLAN 99，并开启边缘端口快速上线

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 99

&#x20;spanning-tree portfast

&#x20;exit



enable

configure terminal



! 确保交换机里有 VLAN 10

vlan 10

&#x20;name Doctor\_WS

&#x20;exit



! 将连接医生电脑的接口 (假设是 Fa0/2) 划入 VLAN 10

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 10

&#x20;! 开启边缘端口，让电脑插上网线秒通

&#x20;spanning-tree portfast

&#x20;exit





**急诊与手术中心：**

enable

configure terminal



! 1. 在底层交换机上创建 VLAN 99

vlan 99

&#x20;name AP\_Management

&#x20;exit



! 2. 将连接 AP 的接口划入 VLAN 99，并开启边缘端口快速上线

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 99

&#x20;spanning-tree portfast

&#x20;exit



enable

configure terminal



! 确保交换机里有 VLAN 10

vlan 10

&#x20;name Doctor\_WS

&#x20;exit



! 将连接医生电脑的接口 (假设是 Fa0/2) 划入 VLAN 10

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 10

&#x20;spanning-tree portfast

&#x20;exit



interface FastEthernet 0/1

&#x20;switchport mode trunk

&#x20;exit





**外网核心A:**

enable

configure terminal

ip routing               ! 开启三层交换机的路由功能

vlan 30

&#x20;name Admin\_Office

vlan 50

&#x20;name Guest\_WLAN

vlan 60

&#x20;name Finance\_Net

exit



interface range FastEthernet 0/1 - 2

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;channel-group 1 mode active    ! 使用 LACP 协议将两个接口捆绑为聚合口 1

&#x20;exit



interface Port-channel 1

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit



interface range FastEthernet 0/3  , FastEthernet 0/5

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit

do show interfaces trunk





! 行政办公 (VLAN 30)

interface Vlan 30

&#x20;ip address 192.168.30.252 255.255.255.0

&#x20;standby 30 ip 192.168.30.254

&#x20;standby 30 priority 120

&#x20;standby 30 preempt

&#x20;no shutdown

&#x20;exit



! 访客无线 (VLAN 50)

interface Vlan 50

&#x20;ip address 192.168.50.252 255.255.255.0

&#x20;standby 50 ip 192.168.50.254

&#x20;standby 50 priority 120

&#x20;standby 50 preempt

&#x20;no shutdown

&#x20;exit



! 财务专网 (VLAN 60)

interface Vlan 60

&#x20;ip address 192.168.60.252 255.255.255.0

&#x20;standby 60 ip 192.168.60.254

&#x20;standby 60 priority 120

&#x20;standby 60 preempt

&#x20;no shutdown

&#x20;exit



spanning-tree vlan 30,50,60 root primary



! 排除网关和核心交换机的物理地址，防止冲突

ip dhcp excluded-address 192.168.30.250 192.168.30.254

ip dhcp excluded-address 192.168.50.250 192.168.50.254

ip dhcp excluded-address 192.168.60.250 192.168.60.254



ip dhcp pool VLAN30\_Admin

&#x20;network 192.168.30.0 255.255.255.0

&#x20;default-router 192.168.30.254



ip dhcp pool VLAN50\_Guest

&#x20;network 192.168.50.0 255.255.255.0

&#x20;default-router 192.168.50.254



ip dhcp pool VLAN60\_Finance

&#x20;network 192.168.60.0 255.255.255.0

&#x20;default-router 192.168.60.254

exit



**外网核心B:**



enable

configure terminal

ip routing

vlan 30

vlan 50

vlan 60

exit



interface range FastEthernet 0/1 - 2

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;channel-group 1 mode active

&#x20;exit



interface Port-channel 1

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit



! 行政办公 (VLAN 30)

interface Vlan 30

&#x20;ip address 192.168.30.253 255.255.255.0

&#x20;standby 30 ip 192.168.30.254

&#x20;standby 30 preempt

&#x20;no shutdown

&#x20;exit



! 访客无线 (VLAN 50)

interface Vlan 50

&#x20;ip address 192.168.50.253 255.255.255.0

&#x20;standby 50 ip 192.168.50.254

&#x20;standby 50 preempt

&#x20;no shutdown

&#x20;exit



! 财务专网 (VLAN 60)

interface Vlan 60

&#x20;ip address 192.168.60.253 255.255.255.0

&#x20;standby 60 ip 192.168.60.254

&#x20;standby 60 preempt

&#x20;no shutdown

&#x20;exit



enable

configure terminal

! 补齐连接底下汇聚层交换机的 Trunk

interface range FastEthernet 0/3  , FastEthernet 0/5

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;no shutdown

&#x20;exit



write memory



spanning-tree vlan 30,50,60 root secondary



**MultilayerSwitch3**



enable

configure terminal

! 1. 创建所有外网需要的 VLAN

vlan 30

&#x20;name Admin

vlan 50

&#x20;name Guest\_Wireless

vlan 60

&#x20;name Finance

exit



! 2. 批量配置接口为 Trunk (假设 Fa0/1 到 Fa0/4 都是连接其他交换机的)

interface range FastEthernet 0/1 - 4

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit





**MultilayerSwitch4**



enable

configure terminal

! 1. 创建所有外网需要的 VLAN

vlan 30

&#x20;name Admin

vlan 50

&#x20;name Guest\_Wireless

vlan 60

&#x20;name Finance

exit



! 2. 批量配置接口为 Trunk (假设 Fa0/1 到 Fa0/4 都是连接其他交换机的)

interface range FastEthernet 0/1 - 4

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;exit



**行政大楼:**



enable

configure terminal

vlan 30

vlan 50

exit





! 配置上联汇聚的接口 (假设是 Fa0/1)

interface FastEthernet 0/1

&#x20;switchport mode trunk

&#x20;exit



! 配置连接行政PC的接口 (假设是 Fa0/2)

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 30

&#x20;spanning-tree portfast

&#x20;exit



! 配置连接 AP4 的接口 (假设是 Fa0/3 或 Gig0/1)

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 50

&#x20;spanning-tree portfast

&#x20;exit



**财务与后勤交换机:**



enable

configure terminal

vlan 60

vlan 50

exit



! 配置上联口 (假设 Fa0/1)

interface FastEthernet 0/1

&#x20;switchport mode trunk

&#x20;exit



! 配置财务PC (假设 Fa0/2)

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 60

&#x20;spanning-tree portfast

&#x20;exit



! 配置连 AP5 的接口 (假设 Fa0/3)

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 50

&#x20;spanning-tree portfast

&#x20;exit



**门诊外网查询 \& 住院部外网查询:**



enable

configure terminal

vlan 50

exit



! 配置上联汇聚的接口 (假设 Fa0/1)

interface FastEthernet 0/1

&#x20;switchport mode trunk

&#x20;exit



! 配置连接 查询PC 的接口 (假设 Fa0/2)

interface FastEthernet 0/2

&#x20;switchport mode access

&#x20;switchport access vlan 50

&#x20;spanning-tree portfast

&#x20;exit



! 配置连接 AP (AP6/AP7) 的接口 (假设 Fa0/3)

interface FastEthernet 0/3

&#x20;switchport mode access

&#x20;switchport access vlan 50

&#x20;spanning-tree portfast

&#x20;exit



**外网核心A 和 外网核心B：**





enable

configure terminal

interface FastEthernet 0/4

&#x20;switchport trunk encapsulation dot1q

&#x20;switchport mode trunk

&#x20;switchport trunk native vlan 50    ! 关键：让外网AP的管理数据无标签通过

&#x20;exit







**外网核心A:**

enable

configure terminal

! 将连向防火墙的 Fa0/6 升级为三层路由口

interface FastEthernet 0/6

&#x20;no switchport

&#x20;ip address 192.168.254.2 255.255.255.252

&#x20;no shutdown

&#x20;exit



enable

configure terminal

router ospf 1

&#x20;! 宣告底层的业务网段

&#x20;network 192.168.30.0 0.0.0.255 area 0

&#x20;network 192.168.50.0 0.0.0.255 area 0

&#x20;network 192.168.60.0 0.0.0.255 area 0

&#x20;! 宣告连向防火墙的主互联链路 (192.168.254.0/30)

&#x20;network 192.168.254.0 0.0.0.3 area 0

&#x20;exit

write memory



**外网核心B:**

enable

configure terminal

! 将连向防火墙的 Fa0/6 升级为三层路由口

interface FastEthernet 0/6

&#x20;no switchport

&#x20;ip address 192.168.253.2 255.255.255.252

&#x20;no shutdown

&#x20;exit



enable

configure terminal

router ospf 1

&#x20;! 宣告底层的业务网段 (因为核心A和B通过汇聚层二层互通，所以网段一样)

&#x20;network 192.168.30.0 0.0.0.255 area 0

&#x20;network 192.168.50.0 0.0.0.255 area 0

&#x20;network 192.168.60.0 0.0.0.255 area 0

&#x20;! 宣告连向防火墙的备用互联链路 (192.168.253.0/30)

&#x20;network 192.168.253.0 0.0.0.3 area 0

&#x20;exit

write memory



**ASA3:**



enable

! (密码为空，直接回车)

configure terminal



! 1. 配置连接核心 A 的接口 (Gig1/2) - 主力通道

interface GigabitEthernet1/1

&#x20;nameif inside\_a

&#x20;security-level 100

&#x20;ip address 192.168.254.1 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 2. 配置连接核心 B 的接口 (Gig1/1) - 备用通道

interface GigabitEthernet1/2

&#x20;nameif inside\_b

&#x20;security-level 100

&#x20;ip address 192.168.253.1 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 3. 配置连接 Router1 的接口 (Gig1/3) - 互联网出口

interface GigabitEthernet1/3

&#x20;nameif outside

&#x20;security-level 0

&#x20;ip address 202.101.1.1 255.255.255.252

&#x20;no shutdown

&#x20;exit







! 去往互联网的默认路由，扔给外面的 Router1 (202.101.1.2)

route outside 0.0.0.0 0.0.0.0 202.101.1.2



enable

configure terminal

router ospf 1

&#x20;! 宣告连向核心A和核心B的两个接口网段

&#x20;network 192.168.254.0 255.255.255.252 area 0

&#x20;network 192.168.253.0 255.255.255.252 area 0

network 202.101.1.0 255.255.255.252 area 0

&#x20;! 关键指令：将防火墙的默认路由自动下发给核心交换机！

&#x20;default-information originate

&#x20;exit

write memory



! 为核心 A 上来的流量做 NAT

object network OBJ\_INSIDE\_A

&#x20;subnet 192.168.0.0 255.255.0.0

&#x20;nat (inside\_a,outside) dynamic interface

&#x20;exit



! 为核心 B 上来的流量做 NAT

object network OBJ\_INSIDE\_B

&#x20;subnet 192.168.0.0 255.255.0.0

&#x20;nat (inside\_b,outside) dynamic interface

&#x20;exit



! 开启 ICMP 检查，否则你的电脑 Ping 不通公网

enable

configure terminal



! 1. 先手动创建默认检测类 (这是你漏掉的)

class-map inspection\_default

&#x20;match default-inspection-traffic

&#x20;exit



! 2. 然后再把它绑进策略里放行 Ping 包

policy-map global\_policy

&#x20;class inspection\_default

&#x20; inspect icmp

&#x20; exit

&#x20;exit



! 3. 全局应用该策略 (极其关键)

service-policy global\_policy global

exit

write memory



! 1. 创建一张名为 OUTSIDE\_IN 的门禁表，明确允许外部回复的 Ping 包 (echo-reply) 进入

access-list OUTSIDE\_IN extended permit icmp any any echo-reply

access-group OUTSIDE\_IN in interface outside

exit





**Router1:**



enable

configure terminal

! 配置连接防火墙的接口

interface GigabitEthernet0/0

&#x20;ip address 202.101.1.2 255.255.255.252

&#x20;no shutdown

&#x20;exit



! 进入连接 Cloud-PT 的接口

interface GigabitEthernet0/1

&#x20;ip address dhcp            ! 自动从“云端”获取 IP 地址

&#x20;no shutdown                ! 开启接口（敲完这句，红灯就会变绿！）

&#x20;exit





enable

configure terminal



! 1. 标记内外网接口

interface GigabitEthernet0/0

&#x20;ip nat inside

&#x20;exit



interface GigabitEthernet0/1

&#x20;ip nat outside

&#x20;exit



! 2. 抓取允许上网的流量

access-list 1 permit any



! 3. 配置端口复用转换 (PAT)

ip nat inside source list 1 interface GigabitEthernet0/1 overload



enable

configure terminal

router ospf 1

&#x20;! Router1 宣告互联的网段（使用反掩码）

&#x20;network 202.101.1.0 0.0.0.3 area 0

&#x20;exit

write memory

