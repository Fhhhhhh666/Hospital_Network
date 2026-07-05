# Hospital Network

同济大学计算机网络课程设计：医院信息化网络改造方案。

本仓库包含医院园区网络的 Cisco Packet Tracer 仿真文件、设备配置命令、课程设计报告和原始配置指令文档。项目面向三级甲等医院网络升级改造场景，采用医疗内网与办公外网物理隔离的双平面架构，并通过核心层、汇聚层、接入层、防火墙、VPN、无线网络等模块完成整体设计与验证。

## Cisco 版本

- Cisco Packet Tracer：`E:\Cisco Packet Tracer 9.0.0`
- 建议使用 Cisco Packet Tracer 9.0.0 或兼容版本打开 `.pkt` 文件。

## 仓库内容

```text
.
├── README.md
├── cisco/
│   └── hospital_network_config.md
├── docs/
│   ├── 2354008-冯浩航-计算机网络课程设计报告.docx
│   └── command.md
└── packet_tracer/
    └── hospital_network.pkt
```

## 文件说明

- `packet_tracer/hospital_network.pkt`：Cisco Packet Tracer 网络拓扑与仿真文件。
- `cisco/hospital_network_config.md`：整理后的 Cisco 设备配置命令，包含核心层、汇聚层、接入层、防火墙、路由、VPN、无线网络等配置。
- `docs/command.md`：课程设计过程中的原始配置指令记录。
- `docs/2354008-冯浩航-计算机网络课程设计报告.docx`：课程设计报告。

## 设计概述

项目为医院园区网络设计与仿真实现，主要目标包括：

- 构建医疗内网与办公外网物理隔离的双平面网络。
- 通过 VLAN 划分不同业务区域，例如门诊、住院部、医技、急诊、无线 AP 管理等。
- 在核心交换机之间配置链路聚合与 Trunk，提高核心链路可靠性。
- 使用三层交换、动态/静态路由、防火墙策略和 NAT 实现业务互联与边界隔离。
- 配置 IPsec Site-to-Site VPN，满足跨站点安全通信需求。
- 部署无线网络相关配置，支持终端无线接入。
- 通过 Packet Tracer 完成拓扑仿真和连通性验证。

## 使用方式

1. 使用 Cisco Packet Tracer 9.0.0 打开 `packet_tracer/hospital_network.pkt`。
2. 按需参考 `cisco/hospital_network_config.md` 中的设备配置命令。
3. 查看 `docs/command.md` 了解原始配置步骤。
4. 查看 `docs/2354008-冯浩航-计算机网络课程设计报告.docx` 获取完整课程设计说明。

