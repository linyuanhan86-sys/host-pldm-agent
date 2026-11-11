# Host PLDM Agent over PCIe MMBI

Author: linyuanhan

Created: November 11, 2026

Status: Design draft

## Problem Description

BMC 的带外管理能力依赖其可访问的硬件接口。部分设备的逻辑拓扑、软件配置和运行状态
由 Host OS 维护，需要通过主机软件提供给 BMC。
本设计解决这些主机可见信息如何通过标准管理协议接入 BMC 的问题，
使资产信息、传感器数据和 Redfish 资源能够纳入现有服务器管理服务。

设计目标是在 Host OS 运行期间，向 BMC 提供资产查询、传感器访问、Redfish 资源访问
和管理事件通知能力，并支持不依赖目标 BMC 硬件的本地开发与验证。
方案采用用户态 Host PLDM Agent，以 ASPEED PCIe 共享内存承载 MMBI，
通过 MCTP 传输 PLDM 消息；BMC 侧扩展既有 MCTP 与 PLDM 服务完成接入。

本地验证使用 Unix 域套接字（UDS）传输 MCTP 数据包，由协议客户端访问 Host Agent，
并通过 HTTP 观察 Redfish 资源内容。开发者可以在目标硬件可用之前验证资源语义、
PLDM/RDE 交互和 BEJ 编解码，为系统集成提供可复现的验证基础。

## Background and References

### Hardware and software references

ASPEED Host–BMC PCIe 功能提供 Host 访问 BMC 共享内存及双方通知的硬件基础。
两端的参考驱动具有以下职责：

| 所在端 | 参考 | 参考范围 |
| --- | --- | --- |
| BMC | [aspeed-bmc-dev.c][1] | BMC 侧 PCIe 共享内存映射与通知能力 |
| Host OS | [aspeed-host-bmc-dev.c][2] | Host 侧 PCIe 设备访问与共享内存映射能力 |

[BMC eSPI MMBI 实现][3] 提供 MMBI 通信及内核应用接口的参考。
本设计将相应的 MMBI 通信能力适配到既有 PCIe 硬件通道，
并保持 BMC 侧 MMBI 应用 ABI 的兼容性。

BMC 用户态软件以 OCP 的 [mctpd][4]、相关 MCTP 库和 [pldmd][5] 为扩展基础。
MCTP 侧扩展负责接入 MMBI 传输；pldmd 扩展负责访问 Host 管理能力、接收事件，
并与 BMC 现有管理服务集成。

### Protocol layering

协议依据为 [MMBI][6]、[MCTP over MMBI][7]、[PLDM Platform][8]、
[PLDM FRU][9] 和 [RDE][10]。各层职责如下：

| 层次 | 职责 |
| --- | --- |
| PCIe 硬件通道 | 提供共享内存访问与通知能力 |
| MMBI | 通过共享缓冲区提供 Host 与 BMC 之间的通信接口 |
| MCTP over MMBI | 定义 MCTP 数据包在 MMBI 上的传输绑定 |
| MCTP | 为上层提供管理消息传输，包括必要的分片与重组 |
| PLDM | 定义能力发现、资产、监控、RDE 访问和事件交互 |

本设计采用 MCTP over MMBI。PCIe 是其底层硬件通道，所用的传输绑定与
MCTP over PCIe VDM 不同。

## Requirements

### Functional requirements

| 能力 | Host 侧职责 | BMC 侧职责 |
| --- | --- | --- |
| PLDM 能力发现 | 提供 PLDM 管理端点标识及支持的管理能力 | 识别 Host 的 PLDM 管理端点并确定可用功能 |
| FRU | 提供主机可获取的设备资产数据 | 读取资产数据并接入 BMC 资产管理 |
| Sensor | 通过 PLDM Platform 提供传感器描述、数值和状态 | 识别并访问 Host 传感器，接入 BMC 监控服务 |
| RDE | 提供可通过 PLDM 访问的 Redfish 资源 | 承担 RDE 请求方职责，并与 BMC Redfish 服务集成 |
| PLDM Event | 通过 PLDM Platform 事件机制通知管理变化 | 接收支持的事件类别并执行对应的管理处理 |

基础功能范围为信息读取和状态监控。配置修改、设备控制和固件更新可在后续扩展中定义。
事件类别及处理语义以双方实际支持的能力为边界。

### Architectural requirements

1. **Host 用户态协议处理。** Host Agent 通过 PCIe 共享内存映射访问通信区域，
   在用户态完成 MMBI、MCTP 和 PLDM 处理。
2. **BMC 应用层 MCTP 集成。** 在 libmctp、mctpd 及相关库中扩展 MMBI 支持，
   以 OCP pldmd 为基础扩展 Host 管理功能。
3. **BMC MMBI 应用 ABI 兼容。** 保留既有 PCIe 硬件基础，
   BMC PCIe MMBI 驱动向应用提供与既有 eSPI MMBI 接口兼容的能力。
4. **独立的本地验证能力。** 在没有目标 BMC 和 PCIe MMBI 链路的环境中，
   支持通过 UDS 进行 MCTP/PLDM/RDE 交互及 BEJ 转换验证，并通过 HTTP 检查资源内容。

### Scope

本设计覆盖 Host OS 运行期间，一个 Host 与一个 BMC 之间的管理通信，
以及不依赖目标 BMC 的本地协议与资源验证。
Host Agent 补充 BMC 的带外管理；Host 停止运行时所需的独立管理能力继续由 BMC 提供。
BIOS 阶段的协议接入及通道交接由平台集成另行定义。

本文约定组件对外能力、协议角色和接口兼容边界。
设备发现方式、内部数据模型、资源标识策略、模块划分、缓存及状态管理由实现者选择。

## Proposed Design

### Architecture overview

Host Agent 提供 FRU、Sensor、RDE 和事件能力，经 PCIe MMBI 通道与 BMC 交换管理消息。
BMC 应用层 MCTP 栈负责传输接入，OCP pldmd 扩展负责 PLDM 管理交互，
相关结果由 BMC 的资产、监控、事件和 Redfish 服务使用。

```text
+----------------------------------------------------------------------------+
|  Host OS (userspace)                                                       |
|                                                                            |
|  +-------------------------------+                 +-------------------+   |
|  | Host PLDM Agent               |                 | MCTP / MMBI       |   |
|  | FRU / Sensor / RDE / Event    |<--------------->| userspace stack   |   |
|  +-------------------------------+                 +-------------------+   |
|                                                              ^             |
+--------------------------------------------------------------+-------------+
                                                               |
                                                               v
                                                 +---------------------------+
                                                 | PCIe shared memory        |
                                                 | and notifications         |
                                                 +---------------------------+
                                                               ^
                                                               |
                                                               |
+--------------------------------------------------------------+-------------+
|  BMC                                                         |             |
|                                                              v             |
|  +-------------------+    +-------------------+    +-------------------+   |
|  | OCP pldmd         |<-->| Application MCTP  |<-->| PCIe MMBI driver  |   |
|  | extensions        |    | stack             |    |                   |   |
|  +-------------------+    +-------------------+    +-------------------+   |
|      |           ^                  RDE access                             |
|      |           +-------------------------------------------+             |
|      |                                                       |             |
|      |                                                       |             |
|      v                                                       v             |
|  +-------------------+       Management data       +-------------------+   |
|  | Assets / sensors  |---------------------------->| BMC Redfish       |   |
|  | and events        |                             | service           |   |
|  +-------------------+                             +-------------------+   |
|                                                                            |
+----------------------------------------------------------------------------+
```

图中双向连接表示协议或服务交互，单向连接表示管理信息的发布与使用。
RDE 资源访问通过 pldmd 扩展与 Redfish 服务衔接；资产、传感器和事件通过相应管理服务接入。

### Host PLDM terminus

Host Agent 提供主机的 PLDM 管理端点（terminus），通过 MCTP 端点与 BMC 交换消息。
主机信息可以来自操作系统接口、设备管理库或其他可用的数据来源。

在资产、传感器和 RDE 访问事务中，BMC 通常作为请求方（requester），Host 作为响应方
（responder）。在 Host 采用请求／响应方式主动上报事件的事务中，Host 作为请求方，
BMC 作为事件接收方和响应方。请求方与响应方是具体事务中的角色。

Host 可通过软件承担 RDE device 角色，将主机可获取的信息表达为 Redfish 资源。
BMC 承担 RDE 管理控制器角色，通过 PLDM 访问这些资源，再由 BMC Redfish 服务
向管理客户端提供相应接口。这一职责划分依据 [RDE 规范][10]。
BEJ 用于 RDE 的资源数据表示，相关编解码可参考 [libbej][11]。

### PCIe MMBI transport

BMC 侧结合 ASPEED PCIe 硬件能力与 eSPI MMBI 接口参考，实现 PCIe MMBI 通信支持。
Host OS 提供 PCIe 设备访问与共享内存映射能力；Host Agent 通过内存映射访问通信区域，
在用户态完成本端 MMBI 及上层协议处理。

BMC PCIe MMBI 驱动向应用提供与既有 eSPI MMBI 接口兼容的数据包收发和控制能力。
该 ABI 兼容范围限定于 BMC 内核与 BMC 应用之间的接口。
Host 侧使用独立的 PCIe 访问接口，两端通信遵循 MMBI 及其 MCTP 传输绑定规范。

MCTP over MMBI 使用固定的点对点通信，不执行动态 EID 分配或 MCTP 端点发现。
通道可用后，BMC 通过 PLDM 识别 Host 提供的管理能力。
[MCTP over MMBI 规范][7] 定义了这一通信边界。

### BMC MCTP integration

BMC 侧扩展 libmctp 的 MMBI 传输绑定，通过驱动收发 MCTP 数据包，
由 MCTP 层完成消息的分片与重组。
mctpd 向管理服务提供完整的 MCTP 消息通信，mctpwplus 等相关库衔接 PLDM 与 MCTP 服务。

MCTP 侧负责将 PCIe MMBI 接入既有消息通信体系；
pldmd 侧通过该通信接口处理 Host 管理请求与事件。
两类扩展分别承担传输接入和管理协议处理职责。

### Extending OCP pldmd

以 [OCP pldmd][5] 为 BMC 侧扩展基础，围绕 Host 管理端点增加以下能力：

| 扩展方向 | 对外能力 |
| --- | --- |
| Host 接入 | 通过 MCTP 服务访问 MMBI 对端，发现 Host 的 PLDM 管理能力 |
| FRU 与 Sensor 适配 | 获取 Host 资产与传感器信息，提供给相应的 BMC 管理服务 |
| RDE 请求方支持 | 获取资源描述、字典和资源访问结果，向 BMC Redfish 服务提供 Host 资源访问能力 |
| Host 事件接收 | 接收支持的 PLDM 事件，并触发相应的刷新、记录或管理通知 |
| 管理接口集成 | 通过适用的 D-Bus、Redfish 或日志接口提供 Host 信息及事件结果 |

例如，PDR 仓库变化通知可用于提示 BMC 重新获取相关管理描述。
具体处理依据事件语义和 BMC 管理需求确定。

pldmd 的 RDE 扩展负责与 Host 进行协议交互；BMC Redfish 服务负责资源在对外资源树中的
呈现、管理客户端请求处理和访问控制。
两者之间的接口形式与资源挂载方式由 BMC 集成方案确定。

### Standalone development and validation

本地验证在不依赖目标 BMC 和 PCIe MMBI 链路的条件下，覆盖 Host Agent 的资源与协议行为。
开发者可以使用当前 Linux 主机数据或预设测试数据运行验证，随后接入目标平台进行系统联调。

两条访问路径承担不同的验证职责：

- **HTTP 资源观察：** 获取 Redfish JSON，检查资源属性、集合、关联及读取结果。
- **UDS 协议验证：** 通过本地 Socket 传输 MCTP 数据包，由客户端执行 PLDM/RDE
  交互，检查能力发现、字典获取、资源访问和 BEJ 编解码。

```text
                       +------------------------------+
                       | Host PLDM Agent              |
                       +------------------------------+
                              |                ^
               +--------------+                +--------------+
               |                                              |
               v                                              v
+-----------------------------+                +-----------------------------+
| HTTP resource access        |                | UDS / MCTP packets          |
| Redfish JSON                |                |                             |
+-----------------------------+                +-----------------------------+
               |                                              ^
               |                                              |
               |                                              v
               |                               +-----------------------------+
               |                               | Local PLDM / RDE client     |
               |                               |                             |
               |                               +-----------------------------+
               |                                              |
               |                                              v
               |                               +-----------------------------+
               |                               | bej decoded JSON            |
               |                               |                             |
               |                               +-----------------------------+
               |                                              |
               +--------------+                +--------------+
                              v                v
                       +------------------------------+
                       | Common resource semantics    |
                       +------------------------------+
```

UDS 是开发环境中的本地数据包传输通道。客户端通过 MCTP、PLDM 和 RDE 交互取得资源，
HTTP 则提供资源内容的直接观察入口。
该验证方式保留协议交互，但不模拟 PCIe 映射、MMBI 共享缓冲区和硬件通知。

对两条路径共同提供的资源，在输入条件可比较时进行语义对照，并结合预期样例及适用的
Redfish Schema 检查属性和关联。动态数据的采样时间与资源挂载位置应纳入比较条件，
HTTP 本地入口可以提供额外的导航资源。

本地验证给出资源和协议行为的验证结果；目标平台测试负责验证 PCIe MMBI 通道，
以及 BMC 管理对象、事件处理和 Redfish 资源接入。

## Alternatives Considered

### A network management service on the Host

Host 可以直接提供网络管理接口，由 BMC 或管理平台访问。
该方式能够利用现有网络工具，但需要维护网络可达性、认证及服务接入。
本设计选择通过 PLDM 将 Host 能力接入 BMC，以复用板内通信通道和 BMC 对外管理入口；
相应代价是需要完成 MMBI、MCTP 和 PLDM 的两端集成。
HTTP 入口在本方案中用于本地开发和资源观察。

### A kernel managed Host protocol stack

将 MMBI 或 MCTP 处理放入内核，有利于结合内核的设备与资源管理机制进行集成，
同时会增加协议功能部署与目标内核环境之间的关联。
本设计选择用户态协议栈，以支持独立迭代和本地协议验证。
这一选择仍需满足目标平台的 PCIe 访问条件和权限要求，硬件互通性通过目标环境验证。

### A private shared memory management protocol

自定义共享内存协议可以直接满足特定的信息交换需求，
同时需要自行维护数据定义、能力发现和事件语义。
本设计采用 MMBI、MCTP 和 PLDM，以既有规范和开源组件建立互通基础；
其代价是两端需要实现相应的协议能力，并进行跨组件兼容性验证。

## Impacts

### API and compatibility

兼容性评估覆盖三个边界：BMC 内核与应用之间的 MMBI ABI、Host 与 BMC 之间的协议交互，
以及 pldmd 与 BMC 管理服务之间的接口。
BMC PCIe MMBI 驱动保持既有 eSPI MMBI 应用 ABI 兼容，
PLDM 功能以双方实际支持的能力集合为准。

RDE 集成的验证范围覆盖 Host 资源、PLDM/RDE 访问和 BMC Redfish 接口，
以确认协议结果能够在对外管理接口中正确呈现。

### System integration

Host 需要部署 Agent，并获得所需的 PCIe 访问及主机信息读取权限。
BMC 侧集成涉及 PCIe MMBI 驱动、libmctp、mctpd、相关接口库、OCP pldmd 扩展，
以及 Redfish 和其他管理服务的接入。

扩展既有 OCP 组件可以沿用其服务基础和管理接口。
集成时需要确认新增 Host 管理能力与 BMC 原有设备管理功能能够同时使用。

### Security and access

BMC Redfish 服务继续承担对外管理接口的访问控制。
Host 提供的资源、事件和协议输入由 BMC 接收方进行必要校验，
共享内存访问和主机数据读取权限与所需能力匹配。
开发用 HTTP 与 UDS 入口的访问范围由部署环境限定。

### Performance and resource usage

需要评估 FRU 读取、传感器访问、RDE 请求及事件通知带来的响应时间、
Host/BMC 资源开销和共享通道负载。
采样频率、缓存、并发和调度方式由实现选择，性能指标依据平台集成要求确定。

### Developer experience

HTTP 资源观察与 UDS 协议验证分别提供资源层和协议层的检查入口，
便于在硬件联调前定位数据表达或协议交互问题。
预设测试场景和可比较的输出结果支持外部贡献者复现问题、验证变更并参与互通测试。

## Testing

验证分为本地资源与协议验证、目标平台集成验证两个层次。
本地验证通过 HTTP 和 UDS 执行，目标平台验证覆盖 PCIe MMBI 通信及 BMC 管理集成。
各项验证以可观察结果为判断依据。

| 验证范围 | 预期结果 |
| --- | --- |
| HTTP 资源内容 | 所选资源的属性、集合和关联符合预期，并满足适用的 Redfish Schema |
| 本地 MCTP/PLDM | 客户端能够经 UDS 发现支持的能力、读取 FRU 与传感器数据；不支持的请求得到相应协议响应 |
| 本地 RDE 与 BEJ | 客户端能够获取资源及适用字典，正确解码 BEJ 数据并解析资源关联 |
| HTTP 与 RDE 对照 | 在可比较的输入条件下，共同资源的语义一致，且符合独立的预期输出 |
| PLDM Event | 选定事件完成约定的协议交互，在 BMC 上产生可观察的管理处理结果 |
| PCIe MMBI 通道 | Host 与 BMC 能够双向交换管理消息，BMC 应用能够通过约定的兼容接口使用通道 |
| BMC 管理集成 | Host 信息与事件可通过选定的 BMC 管理接口访问或观察，内容与协议结果一致 |
| 重启与异常 | 在 Host、Agent 或 BMC 重启、链路复位或通信失败后，系统能够恢复通信和管理访问 |

验证结果应关联所使用的组件版本、输入场景和预期输出，以支持互通性评审及回归对比。
硬件访问、通知和共享内存通信在目标平台验证，平台相关的性能指标依据具体集成要求确定。

[1]: https://github.com/AspeedTech-BMC/linux/blob/d0202dc0f7bcd587bfa6a7860334c6d0968df1a8/drivers/soc/aspeed/aspeed-bmc-dev.c "ASPEED BMC PCIe reference"
[2]: https://github.com/AspeedTech-BMC/linux/blob/d0202dc0f7bcd587bfa6a7860334c6d0968df1a8/drivers/soc/aspeed/aspeed-host-bmc-dev.c "ASPEED Host OS PCIe reference"
[3]: https://github.com/ocp-hm-openbmc-opf-ami/meta-core/blob/3ae43b2903acd28f2ebd01ccbe08b70048584019/meta-common/recipes-kernel/linux/linux-onetree/0006-adding-intel-espi-and-lpc-sio-driver.patch "OCP BMC eSPI MMBI reference"
[4]: https://github.com/ocp-hm-openbmc-opf-ami/mctpd/tree/18a51b335cba79a36065ce4533487482e1c7f8f5 "OCP application-level MCTP daemon"
[5]: https://github.com/ocp-hm-openbmc-opf-ami/pldmd/tree/dd3037196944ada8b9f3b9d8a5f37c7cd3d899e4 "OCP PLDM daemon extension base"
[6]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0282_1.0.2.pdf "DSP0282 Memory-Mapped Buffer Interface"
[7]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0284_1.0.1.pdf "DSP0284 MCTP MMBI Transport Binding"
[8]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.3.0.pdf "DSP0248 PLDM for Platform Monitoring and Control"
[9]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.1_0.pdf "DSP0257 PLDM for FRU Data"
[10]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0218_1.2.0.pdf "DSP0218 PLDM for Redfish Device Enablement"
[11]: https://github.com/openbmc/libbej "OpenBMC libbej"
