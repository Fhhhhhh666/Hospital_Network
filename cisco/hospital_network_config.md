# 医院园区网络配置命令

---

## 一、内网核心层

### 1.1 内网核心A（Core-A_Inner）

```
enable
configure terminal
hostname Core-A_Inner

! ========== VLAN 数据库 ==========
vlan 10
 exit
vlan 20
 exit
vlan 99
 name AP_Management
 exit
vlan 100
 exit
do show vlan brief

! ========== 与核心B的链路聚合 (Fa0/1-2) ==========
interface range FastEthernet 0/1 - 2
 channel-group 1 mode active
 no shutdown
 exit
do show etherchannel summary

interface Port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit
do show interfaces trunk

! ========== 向下连接汇聚层 + WLC 的 Trunk ==========
interface range FastEthernet 0/3, FastEthernet 0/6 - 7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

! Fa0/3 连接 Dist_SW0，native VLAN 99 给 AP 管理
interface FastEthernet 0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 exit

do show interfaces trunk

! ========== 启用路由 ==========
ip routing

! ========== SVI 网关 + HSRP（主） ==========
interface Vlan 10
 ip address 10.10.10.252 255.255.254.0
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt
 no shutdown
 exit

interface Vlan 20
 ip address 10.10.20.252 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt
 no shutdown
 exit

interface Vlan 99
 ip address 10.10.99.252 255.255.255.0
 standby 99 ip 10.10.99.254
 standby 99 priority 110
 standby 99 preempt
 no shutdown
 exit

! VLAN 100 数据中心不在核心交换机上配置 SVI
! 数据中心网关统一由 ASA2 的 server_zone 接口 10.10.100.254 提供

do show standby brief

! ========== STP 主根桥 ==========
spanning-tree vlan 10,20,99,100 root primary

! ========== 连接防火墙的路由口 ==========
! ASA1 (出口防火墙)
interface FastEthernet 0/5
 no switchport
 ip address 10.254.254.1 255.255.255.252
 no shutdown
 exit

! ASA2 (数据中心防火墙)
interface FastEthernet 0/4
 no switchport
 ip address 10.254.254.5 255.255.255.252
 no shutdown
 exit

! ========== OSPF ==========
router ospf 1
 network 10.10.10.0 0.0.1.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 network 10.254.254.0 0.0.0.3 area 0
 network 10.254.254.4 0.0.0.3 area 0
 exit

! ========== DHCP 服务 ==========
! VLAN 10 — 核心A 负责前半段 .1-.127，排除后半段+网关
ip dhcp excluded-address 10.10.10.128 10.10.10.254
! VLAN 20 — 核心A 负责前半段
ip dhcp excluded-address 10.10.20.128 10.10.20.254
! VLAN 99 — 核心A 负责前半段
ip dhcp excluded-address 10.10.99.128 10.10.99.254

ip dhcp pool Doctor_POOL
 network 10.10.10.0 255.255.254.0
 default-router 10.10.10.254
 dns-server 114.114.114.114
 exit

ip dhcp pool Nurse_POOL
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.254
 dns-server 114.114.114.114
 exit

ip dhcp pool AP_POOL
 network 10.10.99.0 255.255.255.0
 default-router 10.10.99.254
 option 43 ip 10.10.99.250
 exit

! ========== 保存 ==========
write memory
```

### 1.2 内网核心B（Core-B_Inner）

```
enable
configure terminal
hostname Core-B_Inner

! ========== VLAN 数据库 ==========
vlan 10
 exit
vlan 20
 exit
vlan 99
 name AP_Management
 exit
vlan 100
 exit
do show vlan brief

! ========== 与核心A的链路聚合 (Fa0/1-2) ==========
interface range FastEthernet 0/1 - 2
 channel-group 1 mode active
 no shutdown
 exit
do show etherchannel summary

interface Port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit
do show interfaces trunk

! ========== 向下连接汇聚层 + WLC 的 Trunk ==========
interface range FastEthernet 0/3 - 4, FastEthernet 0/7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

! Fa0/7 连接 WLC，native VLAN 99
interface FastEthernet 0/7
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 exit

do show interfaces trunk

! ========== 启用路由 ==========
ip routing

! ========== SVI 网关 + HSRP（备） ==========
interface Vlan 10
 ip address 10.10.10.253 255.255.254.0
 standby 10 ip 10.10.10.254
 standby 10 preempt
 no shutdown
 exit

interface Vlan 20
 ip address 10.10.20.253 255.255.255.0
 standby 20 ip 10.10.20.254
 standby 20 preempt
 no shutdown
 exit

interface Vlan 99
 ip address 10.10.99.253 255.255.255.0
 standby 99 ip 10.10.99.254
 standby 99 preempt
 no shutdown
 exit

! VLAN 100 数据中心不在核心交换机上配置 SVI
! 数据中心网关统一由 ASA2 的 server_zone 接口 10.10.100.254 提供

do show standby brief

! ========== STP 备份根桥 ==========
spanning-tree vlan 10,20,99,100 root secondary

! ========== 连接防火墙的路由口 ==========
! ASA1 (出口防火墙)
interface FastEthernet 0/6
 no switchport
 ip address 10.254.254.9 255.255.255.252
 no shutdown
 exit

! ASA2 (数据中心防火墙)
interface FastEthernet 0/5
 no switchport
 ip address 10.254.254.13 255.255.255.252
 no shutdown
 exit

! ========== OSPF ==========
router ospf 1
 network 10.10.10.0 0.0.1.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 network 10.254.254.8 0.0.0.3 area 0
 network 10.254.254.12 0.0.0.3 area 0
 exit

! ========== DHCP 服务（冗余） ==========
! VLAN 10 — 核心B 负责后半段 .128-.249，排除前半段+网关
ip dhcp excluded-address 10.10.10.1 10.10.10.127
ip dhcp excluded-address 10.10.10.250 10.10.10.254
! VLAN 20 — 核心B 负责后半段
ip dhcp excluded-address 10.10.20.1 10.10.20.127
ip dhcp excluded-address 10.10.20.250 10.10.20.254
! VLAN 99 — 核心B 负责后半段
ip dhcp excluded-address 10.10.99.1 10.10.99.127
ip dhcp excluded-address 10.10.99.250 10.10.99.254

ip dhcp pool Doctor_POOL
 network 10.10.10.0 255.255.254.0
 default-router 10.10.10.254
 dns-server 114.114.114.114
 exit

ip dhcp pool Nurse_POOL
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.254
 dns-server 114.114.114.114
 exit

ip dhcp pool AP_POOL
 network 10.10.99.0 255.255.255.0
 default-router 10.10.99.254
 option 43 ip 10.10.99.250
 exit

! ========== 保存 ==========
write memory
```

---

## 二、内网汇聚层

### 2.1 汇聚交换机0 — 左侧（Dist_SW0_Left）

```
enable
configure terminal
hostname Dist_SW0_Left

vlan 10
 exit
vlan 20
 exit
vlan 99
 name AP_Management
 exit
vlan 100
 exit

! 全部接口 Trunk 透传
interface range FastEthernet 0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

write memory
```

### 2.2 汇聚交换机2 — 右侧（Dist_SW2_Right）

```
enable
configure terminal
hostname Dist_SW2_Right

vlan 10
 exit
vlan 20
 exit
vlan 99
 name AP_Management
 exit
vlan 100
 exit

! 全部接口 Trunk 透传
interface range FastEthernet 0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

write memory
```

---

## 三、内网接入层

### 3.1 门诊内网

```
enable
configure terminal

vlan 10
 name Doctor_WS
 exit
vlan 20
 name Inpatient_Dept
 exit
vlan 99
 name AP_Management
 exit

! 上联 Trunk
interface FastEthernet 0/1
 switchport mode trunk
 exit

! 医生电脑 → VLAN 10
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit

! 护士站 → VLAN 20
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 exit

! AP → VLAN 99
interface FastEthernet 0/4
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 exit

write memory
```

### 3.2 住院部内网

```
enable
configure terminal

vlan 10
 name Doctor_WS
 exit
vlan 20
 name Inpatient_Dept
 exit
vlan 99
 name AP_Management
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit

interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 exit

interface FastEthernet 0/4
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 exit

write memory
```

### 3.3 医技与设备

```
enable
configure terminal

vlan 10
 name Doctor_WS
 exit
vlan 99
 name AP_Management
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

! 医生电脑 → VLAN 10
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit

! 医疗设备 & AP → VLAN 99
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 exit

write memory
```

### 3.4 急诊与手术中心

```
enable
configure terminal

vlan 10
 name Doctor_WS
 exit
vlan 99
 name AP_Management
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

! 医生电脑 → VLAN 10
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 exit

! 设备 & AP → VLAN 99
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 exit

write memory
```

---

## 四、内网防火墙

### 4.1 ASA1 — 出口防火墙（连接卫健委）

```
enable
configure terminal
hostname ASA1_Exit

! ========== 连接内网核心 ==========
interface GigabitEthernet 1/1
 nameif inside_coreA
 security-level 100
 ip address 10.254.254.2 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet 1/2
 nameif inside_coreB
 security-level 100
 ip address 10.254.254.10 255.255.255.252
 no shutdown
 exit

! ========== 连接政务外网路由器 ==========
interface GigabitEthernet 1/3
 nameif outside_gov
 security-level 50
 ip address 10.254.254.17 255.255.255.252
 no shutdown
 exit

! ========== OSPF ==========
router ospf 1
 network 10.254.254.0 255.255.255.252 area 0
 network 10.254.254.8 255.255.255.252 area 0
 default-information originate
 exit

! ========== 路由 ==========
! 去往卫健委/医保局的静态路由 (下一跳 Router0)
route outside_gov 10.100.0.0 255.255.0.0 10.254.254.18
! 默认路由指向 Router0
route outside_gov 0.0.0.0 0.0.0.0 10.254.254.18

! ========== NAT ==========
! 内网访问卫健委/医保局的业务全部通过 IPsec VPN 加密传输，不在 ASA1 上做 NAT/PAT。
! 保留内网原始地址便于卫健委侧做访问控制、日志审计和故障定位。

! ========== ACL ==========
access-list OUTSIDE_IN extended permit udp host 10.254.254.18 host 10.254.254.17 eq isakmp
access-list OUTSIDE_IN extended permit esp host 10.254.254.18 host 10.254.254.17
access-list OUTSIDE_IN extended permit icmp any any echo-reply
access-group OUTSIDE_IN in interface outside_gov

write memory
```

### 4.2 ASA2 — 数据中心防火墙

```
enable
configure terminal
hostname ASA2_DC

! 允许 ASA2 的同安全级别接口之间转发流量
same-security-traffic permit inter-interface

! ========== 连接内网核心 ==========
interface GigabitEthernet 1/1
 nameif inside_coreA
 security-level 100
 ip address 10.254.254.6 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet 1/2
 nameif inside_coreB
 security-level 100
 ip address 10.254.254.14 255.255.255.252
 no shutdown
 exit

! ========== 连接数据中心交换机 ==========
interface GigabitEthernet 1/3
 nameif server_zone
 security-level 100
 ip address 10.10.100.254 255.255.255.0
 no shutdown
 exit

! ========== OSPF ==========
router ospf 1
 network 10.254.254.4 255.255.255.252 area 0
 network 10.254.254.12 255.255.255.252 area 0
 network 10.10.100.0 255.255.255.0 area 0
 exit

! ========== 数据中心访问控制 ==========
! VLAN 10 医生工作站允许访问 HIS/LIS 与 PACS
access-list DC_ACCESS extended permit ip 10.10.10.0 255.255.254.0 10.10.100.0 255.255.255.0
! VLAN 99 医疗设备/AP 管理网允许访问数据中心
access-list DC_ACCESS extended permit ip 10.10.99.0 255.255.255.0 10.10.100.0 255.255.255.0
! VLAN 20 护士站禁止访问 PACS 影像存储服务器 10.10.100.2
access-list DC_ACCESS extended deny ip 10.10.20.0 255.255.255.0 host 10.10.100.2
! 其余业务流量放行，例如护士站访问 HIS/LIS 服务器 10.10.100.1
access-list DC_ACCESS extended permit ip any any
access-group DC_ACCESS in interface inside_coreA
access-group DC_ACCESS in interface inside_coreB

write memory
```

---

## 五、政务外网出口

### 5.1 边界路由器 Router0（Router0_GovLine）— Cisco 1941

> **设备替换说明：** 原路由器已替换为 Cisco 1941（或 2911），接口为 GigabitEthernet。
> 若后续需要配置 IPsec VPN，需先激活 securityk9 许可证：
> ```
> license boot module c1900 technology-package securityk9
> reload
> ```

```
enable
configure terminal
hostname Router0_GovLine

! 连接 ASA1
interface GigabitEthernet 0/0
 ip address 10.254.254.18 255.255.255.252
 no shutdown
 exit

! 连接卫健委/医保局专线
interface GigabitEthernet 0/1
 ip address 10.100.1.1 255.255.255.0
 no shutdown
 exit

! 静态路由
ip route 10.0.0.0 255.0.0.0 10.254.254.17
ip route 0.0.0.0 0.0.0.0 10.100.1.254

write memory
```

---

## 六、外网核心层

### 6.1 外网核心A

```
enable
configure terminal
ip routing

vlan 30
 name Admin_Office
 exit
vlan 50
 name Guest_WLAN
 exit
vlan 60
 name Finance_Net
 exit

! ========== 与核心B的链路聚合 ==========
interface range FastEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 exit

interface Port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

! ========== 向下连接汇聚层 ==========
interface range FastEthernet 0/3, FastEthernet 0/5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

! 连接外网 WLC（native VLAN 50）
interface FastEthernet 0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 50
 exit

! ========== SVI 网关 + HSRP（主） ==========
interface Vlan 30
 ip address 192.168.30.252 255.255.255.0
 standby 30 ip 192.168.30.254
 standby 30 priority 120
 standby 30 preempt
 no shutdown
 exit

interface Vlan 50
 ip address 192.168.50.252 255.255.255.0
 standby 50 ip 192.168.50.254
 standby 50 priority 120
 standby 50 preempt
 no shutdown
 exit

interface Vlan 60
 ip address 192.168.60.252 255.255.255.0
 standby 60 ip 192.168.60.254
 standby 60 priority 120
 standby 60 preempt
 no shutdown
 exit

! ========== STP 主根桥 ==========
spanning-tree vlan 30,50,60 root primary

! ========== DHCP ==========
! VLAN 30 — 核心A 负责前半段 .1-.127
ip dhcp excluded-address 192.168.30.128 192.168.30.254
! VLAN 50 — 核心A 负责前半段
ip dhcp excluded-address 192.168.50.128 192.168.50.254
! VLAN 60 — 核心A 负责前半段
ip dhcp excluded-address 192.168.60.128 192.168.60.254

ip dhcp pool VLAN30_Admin
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.254
 exit

ip dhcp pool VLAN50_Guest
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.254
 exit

ip dhcp pool VLAN60_Finance
 network 192.168.60.0 255.255.255.0
 default-router 192.168.60.254
 exit

! ========== 连接防火墙的路由口 ==========
interface FastEthernet 0/6
 no switchport
 ip address 192.168.254.2 255.255.255.252
 no shutdown
 exit

! ========== OSPF ==========
router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.254.0 0.0.0.3 area 0
 exit

write memory
```

### 6.2 外网核心B

```
enable
configure terminal
ip routing

vlan 30
 name Admin_Office
 exit
vlan 50
 name Guest_WLAN
 exit
vlan 60
 name Finance_Net
 exit

! ========== 与核心A的链路聚合 ==========
interface range FastEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 exit

interface Port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

! ========== 向下连接汇聚层 ==========
interface range FastEthernet 0/3, FastEthernet 0/5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
 exit

interface FastEthernet 0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 50
 exit

! ========== SVI 网关 + HSRP（备） ==========
interface Vlan 30
 ip address 192.168.30.253 255.255.255.0
 standby 30 ip 192.168.30.254
 standby 30 preempt
 no shutdown
 exit

interface Vlan 50
 ip address 192.168.50.253 255.255.255.0
 standby 50 ip 192.168.50.254
 standby 50 preempt
 no shutdown
 exit

interface Vlan 60
 ip address 192.168.60.253 255.255.255.0
 standby 60 ip 192.168.60.254
 standby 60 preempt
 no shutdown
 exit

! ========== STP 备份根桥 ==========
spanning-tree vlan 30,50,60 root secondary

! ========== 连接防火墙的路由口 ==========
interface FastEthernet 0/6
 no switchport
 ip address 192.168.253.2 255.255.255.252
 no shutdown
 exit

! ========== DHCP 服务（冗余） ==========
ip dhcp excluded-address 192.168.30.1 192.168.30.127
ip dhcp excluded-address 192.168.30.250 192.168.30.254
ip dhcp excluded-address 192.168.50.1 192.168.50.127
ip dhcp excluded-address 192.168.50.250 192.168.50.254
ip dhcp excluded-address 192.168.60.1 192.168.60.127
ip dhcp excluded-address 192.168.60.250 192.168.60.254

ip dhcp pool VLAN30_Admin
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.254
 exit

ip dhcp pool VLAN50_Guest
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.254
 exit

ip dhcp pool VLAN60_Finance
 network 192.168.60.0 255.255.255.0
 default-router 192.168.60.254
 exit

! ========== OSPF ==========
router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.253.0 0.0.0.3 area 0
 exit

write memory
```

---

## 七、外网汇聚层

### 7.1 汇聚交换机3（MultilayerSwitch3）

```
enable
configure terminal

vlan 30
 name Admin
 exit
vlan 50
 name Guest_Wireless
 exit
vlan 60
 name Finance
 exit

interface range FastEthernet 0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

write memory
```

### 7.2 汇聚交换机4（MultilayerSwitch4）

```
enable
configure terminal

vlan 30
 name Admin
 exit
vlan 50
 name Guest_Wireless
 exit
vlan 60
 name Finance
 exit

interface range FastEthernet 0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 exit

write memory
```

---

## 八、外网接入层

### 8.1 行政大楼

```
enable
configure terminal

vlan 30
 exit
vlan 50
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

! 行政PC → VLAN 30
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 exit

! AP → VLAN 50
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 exit

write memory
```

### 8.2 财务与后勤

```
enable
configure terminal

vlan 60
 exit
vlan 50
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

! 财务PC → VLAN 60
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
 exit

! AP → VLAN 50
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 exit

write memory
```

### 8.3 门诊外网查询 & 住院部外网查询

```
enable
configure terminal

vlan 50
 exit

interface FastEthernet 0/1
 switchport mode trunk
 exit

! 查询PC → VLAN 50
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 exit

! AP → VLAN 50
interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 exit

write memory
```

---

## 九、外网防火墙与互联网出口

### 9.1 ASA3 — 互联网出口防火墙

```
enable
configure terminal

! ========== 接口配置 ==========
! 连接外网核心A
interface GigabitEthernet1/1
 nameif inside_a
 security-level 100
 ip address 192.168.254.1 255.255.255.252
 no shutdown
 exit

! 连接外网核心B
interface GigabitEthernet1/2
 nameif inside_b
 security-level 100
 ip address 192.168.253.1 255.255.255.252
 no shutdown
 exit

! 连接 Router1（互联网）
interface GigabitEthernet1/3
 nameif outside
 security-level 0
 ip address 202.101.1.1 255.255.255.252
 no shutdown
 exit

! ========== 路由 ==========
route outside 0.0.0.0 0.0.0.0 202.101.1.2

! ========== OSPF ==========
router ospf 1
 network 192.168.254.0 255.255.255.252 area 0
 network 192.168.253.0 255.255.255.252 area 0
 network 202.101.1.0 255.255.255.252 area 0
 default-information originate
 exit

! ========== NAT ==========
object network OBJ_INSIDE_A
 subnet 192.168.0.0 255.255.0.0
 nat (inside_a,outside) dynamic interface
 exit

object network OBJ_INSIDE_B
 subnet 192.168.0.0 255.255.0.0
 nat (inside_b,outside) dynamic interface
 exit

! ========== ICMP 放行 ==========
class-map inspection_default
 match default-inspection-traffic
 exit

policy-map global_policy
 class inspection_default
  inspect icmp
  exit
 exit

service-policy global_policy global

access-list OUTSIDE_IN extended permit icmp any any echo-reply
access-group OUTSIDE_IN in interface outside

write memory
```

### 9.2 Router1 — 互联网边界路由器

```
enable
configure terminal

! 连接 ASA3
interface GigabitEthernet0/0
 ip nat inside
 ip address 202.101.1.2 255.255.255.252
 no shutdown
 exit

! 连接 Cloud-PT（DHCP 获取公网地址）
interface GigabitEthernet0/1
 ip nat outside
 ip address dhcp
 no shutdown
 exit

! PAT
access-list 1 permit any
ip nat inside source list 1 interface GigabitEthernet0/1 overload

! OSPF
router ospf 1
 network 202.101.1.0 0.0.0.3 area 0
 exit

write memory
```

---

## 十、IPsec VPN 站点到站点加密

> ASA1 与 Router0 之间建立 IPsec Site-to-Site VPN，加密内网（医生站、数据中心）与卫健委之间的数据传输。

### 10.1 ASA1 追加 IPsec 配置

> 以下命令在现有 ASA1 基础配置（第四章）之上追加。

```
! ========== 1. 开启 ISAKMP/IKE 协议 ==========
crypto isakmp enable outside_gov

! ========== 2. 配置 Phase 1 (ISAKMP/IKE) 策略 ==========
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 5
 lifetime 86400
 exit

! ========== 3. 配置预共享密钥 (Tunnel Group) ==========
tunnel-group 10.254.254.18 type ipsec-l2l
tunnel-group 10.254.254.18 ipsec-attributes
 pre-shared-key cisco123
 exit

! ========== 4. 配置 Phase 2 (IPsec) 转换集 ==========
crypto ipsec transform-set TRANS_AES_SHA esp-aes esp-sha-hmac

! ========== 5. 配置感兴趣流 (ACL) ==========
access-list VPN_TRAFFIC extended permit ip 10.10.10.0 255.255.254.0 10.100.0.0 255.255.0.0
access-list VPN_TRAFFIC extended permit ip 10.10.20.0 255.255.255.0 10.100.0.0 255.255.0.0
access-list VPN_TRAFFIC extended permit ip 10.10.99.0 255.255.255.0 10.100.0.0 255.255.0.0
access-list VPN_TRAFFIC extended permit ip 10.10.100.0 255.255.255.0 10.100.0.0 255.255.0.0

! ========== 6. 配置 Crypto Map ==========
crypto map GOV_VPN_MAP 10 match address VPN_TRAFFIC
crypto map GOV_VPN_MAP 10 set peer 10.254.254.18
crypto map GOV_VPN_MAP 10 set transform-set TRANS_AES_SHA

! ========== 7. 应用到接口 ==========
crypto map GOV_VPN_MAP interface outside_gov

! ========== 8. NAT 豁免（VPN 流量不做 NAT） ==========
! ASA1 不配置内网到卫健委方向的 NAT/PAT。
! VLAN10、VLAN20、VLAN99、VLAN100 保持原始地址，统一匹配 VPN_TRAFFIC 并触发 IPsec。

write memory
```

### 10.2 Router0 追加 IPsec 配置

> 以下命令在现有 Router0 基础配置（第五章）之上追加。需先确认 `securityk9` 许可证已激活。

```
! ========== 0. 确保已激活安全许可（若已激活请忽略） ==========
! license boot module c1900 technology-package securityk9
! (激活后需要保存并 reload 重启)

! ========== 1. 配置 Phase 1 (ISAKMP) 策略 ==========
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
 exit

! ========== 2. 配置预共享密钥 ==========
crypto isakmp key cisco123 address 10.254.254.17

! ========== 3. 配置 Phase 2 (IPsec) 转换集 ==========
crypto ipsec transform-set TRANS_AES_SHA esp-aes esp-sha-hmac
 mode tunnel
 exit

! ========== 4. 配置感兴趣流（镜像反转） ==========
access-list 100 permit ip 10.100.0.0 0.0.255.255 10.10.10.0 0.0.1.255
access-list 100 permit ip 10.100.0.0 0.0.255.255 10.10.20.0 0.0.0.255
access-list 100 permit ip 10.100.0.0 0.0.255.255 10.10.99.0 0.0.0.255
access-list 100 permit ip 10.100.0.0 0.0.255.255 10.10.100.0 0.0.0.255

! ========== 5. 配置 Crypto Map ==========
crypto map GOV_VPN_MAP 10 ipsec-isakmp
 description VPN_To_Hospital_ASA
 set peer 10.254.254.17
 set transform-set TRANS_AES_SHA
 match address 100
 exit

! ========== 6. 应用到接口 ==========
interface GigabitEthernet 0/0
 crypto map GOV_VPN_MAP
 exit

! ========== 保存配置 ==========
write memory
```

---

## 附录：VLAN 汇总表

| VLAN | 名称 | 网段 | 网关（HSRP VIP） | 用途 |
|------|------|------|------------------|------|
| 10 | Doctor_WS | 10.10.10.0/23 | 10.10.10.254 | 医生工作站 |
| 20 | Inpatient_Dept | 10.10.20.0/24 | 10.10.20.254 | 护士站/住院部 |
| 30 | Admin_Office | 192.168.30.0/24 | 192.168.30.254 | 行政办公 |
| 50 | Guest_WLAN | 192.168.50.0/24 | 192.168.50.254 | 访客无线 |
| 60 | Finance_Net | 192.168.60.0/24 | 192.168.60.254 | 财务专网 |
| 99 | AP_Management | 10.10.99.0/24 | 10.10.99.254 | AP管理 + 医疗设备 |
| 100 | (数据中心) | 10.10.100.0/24 | 10.10.100.254 | HIS/PACS 服务器 |

## 附录：设备互联 IP 表

| 设备A | 接口 | IP | 设备B | 接口 | IP | 子网 |
|-------|------|----|-------|------|----|------|
| Core-A_Inner | Fa0/5 | 10.254.254.1 | ASA1 | G1/1 | 10.254.254.2 | /30 |
| Core-A_Inner | Fa0/4 | 10.254.254.5 | ASA2 | G1/1 | 10.254.254.6 | /30 |
| Core-B_Inner | Fa0/6 | 10.254.254.9 | ASA1 | G1/2 | 10.254.254.10 | /30 |
| Core-B_Inner | Fa0/5 | 10.254.254.13 | ASA2 | G1/2 | 10.254.254.14 | /30 |
| ASA1 | G1/3 | 10.254.254.17 | Router0 (1941) | G0/0 | 10.254.254.18 | /30 |
| Router0 (1941) | G0/1 | 10.100.1.1 | 卫健委 | — | 10.100.1.254 | /24 |
| 外网核心A | Fa0/6 | 192.168.254.2 | ASA3 | G1/1 | 192.168.254.1 | /30 |
| 外网核心B | Fa0/6 | 192.168.253.2 | ASA3 | G1/2 | 192.168.253.1 | /30 |
| ASA3 | G1/3 | 202.101.1.1 | Router1 | G0/0 | 202.101.1.2 | /30 |
