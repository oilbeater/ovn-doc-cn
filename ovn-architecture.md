# OVN 架构

[源文档连接](https://github.com/ovn-org/ovn/blob/master/ovn-architecture.7.xml)

## 概述

OVN(Open Virtual Network) 是一个为虚拟机和容器环境提供逻辑网络抽象的系统。
OVN 在 OVS 已有的能力之上增加了对逻辑网络抽象的原生支持，例如：L2/L3 的 overlay 网络，
安全组（Security Group）以及 DHCP 等网络服务。和 OVS 的设计目标相同，OVN 也是针对生产环境而实现，
并且能在超大规模下顺利运转。

物理网络包括物理的线缆，交换机以及路由器。虚拟网络将物理网络扩展到虚拟机或容器平台，将虚拟机或容器
桥接到物理网络中。OVN 中的虚拟网络通过隧道或者封装的方式以软件来实现，这样这个网络可以和物理网络
或者其他虚拟网络相互隔离。这样逻辑网络中的地址空间可以和物理网络相互重叠并且不会造成冲突。逻辑网络的拓扑
可以不依赖底层物理网络的拓扑。因此逻辑网络中的虚拟机可以从一台物理机迁移到另一台物理机而不会造成网络中断。
请参考下面“逻辑网络”部分获取更多信息。

网络的封装层可以阻止逻辑网络中的虚拟机或容器直接访问底层物理网络的节点。对于虚拟机或者容器机器，通常
这是一个可接受甚至预期的特性。但是有些情况下虚拟机和容器需要能直接访问物理网络。OVN 提供了多种形式的
网关（Gateway）来完成这个目的。请参考下面“网关”部分获取更多信息。


OVN 的主要组成部分

- 云管系统(CMS, Cloud Management System),它是 OVN 的最终客户（用户或管理员通过 CMS 来使用 OVN）。
CMS 如果要集成 OVN，需要和 CMS 相关的插件以及相关软件。OVN 最早的目标 CMS 是 OpenStack。通常我们将 CMS 和 OVN 一一对应，但是在一些场景下，多个 CMS 可以共享使用同一个 OVN。

- 部署 OVN 数据库的物理或虚拟节点，在集群数据库模式下可能是多个节点。

- 一或多个 hypervisor (承载虚拟机或者容器的节点)。Hypervisor 上必须运行 OVS 以及其他相关软件

- 零或多个网关。网关通过在物理网络的以太网端口和隧道端口之间双向转发数据包来打通逻辑网络和物理网络。
这也使得非虚拟化的机器可以加入逻辑网络。网关可以是物理机，虚拟机也可以是支持 vtep 协议的 ASIC 硬件交换机。

Hypervisor 和网关被合称为传输节点（Transport node）或者 chassis。

下图展示了OVN主要组件和相关软件的交互过程。从上到下分别是：

- 云管系统
- CMS 和 OVN 进行交互的 OVN/CMS 插件，在 OpenStack 中这个插件是 Neutron。插件的主要作用是将
存储在 CMS 数据库中， CMS 相关的逻辑网络的配置翻译为 OVN 可以理解的语言。由于每个 CMS 关于逻辑网络
的配置不同，需要针对每个CMS专门开发相应的网络插件。CMS 插件之下的组件都是和 CMS 无关的。

- OVN 北向数据库（Northbound Database）,接受从 CMS 插件下发的逻辑网络配置。北向数据库的数据模式
应该和 CMS 中的概念是“阻抗匹配（impedance matched, 可以理解为一一对应）”的，因此在北向数据库中直接
支持逻辑交换机，逻辑路由器，访问规则控制等概念。请参考 ovn-nb 的文档获取更多信息。OVN 北向数据库只有两个客户，在其上游的 CMS 插件，以及下游的 ovn-northd。

- ovn-northd 连接其上游的 OVN 北向数据库和其下游的 OVN 南向数据库。它将北向数据库中常规意义的网络配置
翻译成南向数据库中的逻辑数据链路（logical datapath）。

OVN 南向数据库的性能必须能够适应不断增加的传输节点。我们现在发现了 ovsdb-server 存在性能瓶颈需要去攻克。
此外 ovsdv-server 还需要有集群模式的高可用能力。

下面几个组件需要在每台 hypervisor 上运行：

- ovn-controller: 作为 OVN 的 agent 和软件网关运行在每个 hypervisor 上。在北向，ovn-controller连接 OVN 南向数据库，
来学习 OVN 的配置和状态变化，并将 hypervisor 的状态更新到 PN 表以及 Binding 表的 chassis 列。在南向，ovn-controller 
作为一个 OpenFlow 控制器连接 ovs-vswitchd 来控制网络流量，并通过本地的 ovsdb-server 来监控和管理 Open vSwitch 的配置不同，需要针对每个CMS专门开发相应的网络插件。CMS

- ovs-vswitchd 和 ovsdb-server 是常规的 Open vSwitch 组件

```bash
                                  CMS
                                   |
                                   |
                       +-----------|-----------+
                       |           |           |
                       |     OVN/CMS Plugin    |
                       |           |           |
                       |           |           |
                       |   OVN Northbound DB   |
                       |           |           |
                       |           |           |
                       |       ovn-northd      |
                       |           |           |
                       +-----------|-----------+
                                   |
                                   |
                         +-------------------+
                         | OVN Southbound DB |
                         +-------------------+
                                   |
                                   |
                +------------------+------------------+
                |                  |                  |
  HV 1          |                  |    HV n          |
+---------------|---------------+  .  +---------------|---------------+
|               |               |  .  |               |               |
|        ovn-controller         |  .  |        ovn-controller         |
|         |          |          |  .  |         |          |          |
|         |          |          |     |         |          |          |
|  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
|                               |     |                               |
+-------------------------------+     +-------------------------------+
```

### OVN 中的信息流向

在 OVN 中配置信息从北向南流动。CMS 通过 OVN/CMS 插件将逻辑网络配置下发给北向数据库。接下来 ovn-northd 
将北向数据库的信息编译成低级语言形式的逻辑流表配置下发到南向数据库，南向数据库再将逻辑流表下发至每个 chassis。

在 OVN 中状态信息反向从南向北流动。OVN 目前只提供少量的状态信息。首先是逻辑交换机端口的 up 状态，ovn-northd
将会更新这个字段。如果南向数据库中这个逻辑接口在 Port_Binding 表的 chassis 字段非空，则 ovn-northd 会将
北向数据库中的 up 字段设置为 true，否则为 false。通过这种方式，CMS 可以探测虚拟机的网络是否就绪。

其次，OVN 提供了让 CMS 了解它的配置是否生效的反馈机制。这需要 CMS 参与序列号协议之中，该协议的工作方式如下：

1. 当 CMS 更新北向数据库配置时，在同一个事务中需要将 NB_Global 表中的 nb_cfg 值加一。（只在CMS希望知道配置
是否最终下发成功时需要这样做）

2. 当 ovn-northd 将某个北向数据库的快照更新至南向数据库时，ovn-northd 会在同一个事务中将北向数据库中 NB_Global 表的 nb_cfg
值更新至南向数据库的 SB_Global 表。（因此如果一个观察者同时监控两个数据库就可以确认南向数据库是否已经和北向数据库同步）

3. 当 ovn-northd 收到南向数据库事务已提交的确认信息后，它会将北向数据库 NB_Global 表的 sb_cfg 字段更新为所下发
到南向数据库快照中的 nb_cfg 的值。（因此 CMS 或者观察者可以无需监控南向数据库，只监控北向数据库的这个值就可以确定数据库是否已经同步）

4. 在每个 chassis 上的 ovn-controller 接收到了南向数据库的更新，同时得到了最新的 nb_cfg 的值。接下来
ovn-controller 更新 chassis 上 Open vSwitch 实例的流表。当收到流表更新成功的确认信息后，ovn-controller 将会
更新南向数据库中对应 Chassis 记录的 nb_cfg 字段。

5. ovn-northd 监控所有 Chassis 记录的 nb_cfg 字段。它将所有 nb_cfg 字段中的最小值复制到北向数据库 NB_Global
表的 hv_cfg 字段。（这样一来，CMS 或者其他观察者只需要监控这个字段就可以确认是否所有 hypervisor 是否已经更新至最新的配置）

### Chassis 设置

在 OVN 中，每个 Chassis 上的 Open vSwitch 都必须有一个专门的网桥（integration bridge）供 OVN 使用。
系统启动脚本会在 ovn-controller 启动时创建这个网桥，如果这个网桥不存在将会按照下列的配置自动创建，默认为 br-int 网桥。在 integration bridge
上有一下几种类型的端口：

- 隧道端口，OVN 维护这些端口来保证逻辑网络的连通性。由 ovn-controller 来负责增加，删除以及更改这些隧道端口。

- 虚拟接口（VIF）,在每个 hypervisor 上接入逻辑网络。

- Gateway 上的物理接口。系统启动脚本会在 ovn-controller 启动前将这些端口加入到网桥中。在一些更复杂的场景中，
也可能是连接到另一个网桥的 patch 端口。

其他类型的端口不应该被加入这个 integration bridge。接入底层 underlay 的物理端口（gateway port相反，是接入逻辑网络的物理端口）
绝对不应该接入这个网桥。实际上 underlay 的物理端口不应该接入任何一个网桥。

Integration bridge 应该按照如下的配置启动，每个配置的具体作用可以参考 ovs-vswitchd.conf.db(5):

- fail-mode=secure

禁止相互隔离的逻辑网络在 ovn-controller 启动前转发数据包。参考 ovs-vsctl(8)中的 Controller Failure Settings
获取更多信息。

- other-config:disable-in-band=true

禁止在 integration bridge 上生成 in-band flow。

Integration bridge 的默认名为 br-int，用户也可以自定义使用其他名字。
