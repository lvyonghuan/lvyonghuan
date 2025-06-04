**校园网双核心 (MSTP+VRRP) 技术复习笔记**

**核心目标：** 构建一个高可靠、无环路、可实现负载均衡的校园网核心。

**两大关键技术：**
1.  **MSTP (Multiple Spanning Tree Protocol 多生成树协议)：** L2层面，解决冗余链路带来的环路问题，并实现基于VLAN的负载均衡。
2.  **VRRP (Virtual Router Redundancy Protocol 虚拟路由冗余协议)：** L3层面，提供网关冗余备份，确保网关的高可用性。

---

**一、 MSTP 技术概述**

1.  **基本概念：**
    *   简单来说，MSTP 是基于VLAN的RSTP (快速生成树协议)。
    *   它在传统STP/RSTP基础上发展而来，解决了STP/RSTP中所有VLAN共享一棵生成树导致的次优路径和无法有效利用冗余链路的问题。
    *   包含RSTP的快速收敛机制。

2.  **核心思想：**
    *   **MST Instance (多生成树实例)：** 将一个或多个VLAN映射到一个MST实例。
    *   **MST Region (多生成树域)：** 具有相同MSTP配置（域名、修订级别、VLAN到实例的映射）的交换机组成一个MST域。在域内，每个MST实例独立运行自己的生成树算法，称为**IST (Internal Spanning Tree 内部生成树)**。
    *   **CST (Common Spanning Tree 公共生成树)：** 不同MST域之间，以及MST域与STP/RSTP交换机之间，通过CST进行互通。每个MST域在CST中表现为一个逻辑上的“大交换机”。

3.  **解决的问题 (对比STP/RSTP)：**
    *   **场景 (PPT P4-P5)：**
        *   交换机A、B在VLAN1内；交换机C、D在VLAN2内，形成环路。
        *   若使用STP/RSTP，可能会将A-B之间的链路阻塞。
        *   结果：VLAN1内A、B无法通信（正常，因为链路阻塞）。但VLAN2内的C、D之间如果也依赖这条被阻塞的物理路径（例如C-A-B-D），则VLAN2的流量也受影响，即使它们的逻辑环路可能不同。
    *   **MSTP的改进 (PPT P7)：**
        *   可以将VLAN1划入instance 1 (Region 1, 交换机A,B)，VLAN2划入instance 2 (Region 2, 交换机C,D)。
        *   Region 1内部无环路（或有环路但IST会阻塞）。Region 2内部同理。
        *   Region 1和Region 2之间看作两个“大交换机”，它们之间若有环路，CST会选择一条链路进行阻塞。
        *   **关键优势：** 不同实例可以有不同的根桥，从而实现不同VLAN组的流量走不同的路径，达到负载均衡。例如，VLAN1的流量可能主要走A-C，VLAN2的流量可能主要走B-D。

4.  **配置要点：**
    *   `spanning-tree mode mstp`：启用MSTP模式。
    *   `spanning-tree mst configuration`：进入MST配置模式。
        *   `name <region-name>`：配置域名（同一域内必须一致）。
        *   `revision <revision-level>`：配置修订级别（同一域内必须一致）。
        *   `instance <instance-id> vlan <vlan-range>`：将VLAN映射到实例。
    *   `spanning-tree mst <instance-id> priority <value>`：为指定实例配置优先级，选举根桥（值越小越优先）。
    *   `spanning-tree mst <instance-id> root primary/secondary`：快速配置根桥/备份根桥。

---

**二、 VRRP 技术概述**

1.  **基本概念 (PPT P9-P10)：**
    *   VRRP是一种容错协议，用于提供网关冗余。
    *   当主机的下一跳路由器（网关）失效时，可以由另一台路由器及时替代，保持通信的连续性和可靠性。
    *   适用于多播或广播能力的局域网。

2.  **工作原理：**
    *   **虚拟路由器：** 将局域网内的一组物理路由器（一个Master主控路由器，若干Backup备份路由器）组织成一个虚拟路由器，称为一个**备份组 (VRRP Group)**。
    *   **虚拟IP地址和虚拟MAC地址：**
        *   虚拟路由器拥有一个唯一的虚拟IP地址。网络内的主机将此虚拟IP地址配置为其默认网关。
        *   虚拟MAC地址格式固定为：`00-00-5E-00-01-[VRID]`，其中VRID是备份组号。
    *   **Master选举：**
        *   备份组中的路由器根据优先级选举Master。优先级高的成为Master。
        *   如果优先级相同，则比较接口IP地址，IP地址大的成为Master。
        *   Master路由器负责响应对虚拟IP地址的ARP请求（使用虚拟MAC地址），并转发发往虚拟网关的IP报文。
    *   **Backup角色：** 备份路由器侦听Master发送的VRRP通告报文。
    *   **切换机制：**
        *   如果Backup在一定时间内未收到Master的通告报文，则认为Master失效，Backup中优先级最高的路由器将转变为新的Master。
        *   **抢占模式 (Preempt)：** 如果原Master恢复，且其优先级高于当前Master，它可以重新成为Master（如果配置了抢占）。

3.  **图示说明 (PPT P11-P13)：**
    *   路由器A (192.168.66.5, MASTER) 和路由器B (192.168.66.6, BACKUP) 组成一个VRRP备份组。
    *   虚拟路由器IP地址：192.168.66.1。
    *   局域网内主机配置的默认网关都是：192.168.66.1。
    *   主机不关心实际是A还是B在转发数据包，只认虚拟网关。

4.  **配置要点：**
    *   `interface vlan <vlan-id>` 或 `interface <physical-interface>`：进入接口配置模式。
    *   `ip address <real-ip> <subnet-mask>`：配置接口的物理IP地址。
    *   `vrrp <group-id> ip <virtual-ip>` (华为/H3C) 或 `standby <group-id> ip <virtual-ip>` (思科)：配置VRRP组号和虚拟IP地址。
    *   `vrrp <group-id> priority <value>` 或 `standby <group-id> priority <value>`：配置VRRP优先级（值越大越优先，通常默认100）。
    *   `vrrp <group-id> preempt-mode timer delay <seconds>` 或 `standby <group-id> preempt [delay <seconds>]`：配置抢占模式和抢占延迟。

---

**三、 校园网双核心 (MSTP+VRRP) 的拓扑实现与配置实例**

1.  **目标与特点 (PPT P15)：**
    *   **冗余备份：** 核心设备故障时，业务能自动切换。
    *   **负载均衡：** 流量尽可能地在多条路径和多个设备间分担。

2.  **典型拓扑 (PPT P15, P18)：**
    *   **双核心层：** 两台高性能三层交换机 (如RG-S35A, RG-S35B) 作为核心。
    *   **接入层：** 多台二层或三层交换机 (如RG-S21A, RG-S21B) 连接用户。
    *   **连接方式：**
        *   核心A与核心B之间通过高速链路互联（通常为Trunk链路）。
        *   接入交换机通常双上联到两台核心交换机（形成冗余链路）。
    *   **Internet出口：** 核心交换机连接到路由器 (如RG-S35C) 或防火墙，再连接到Internet。

3.  **配置思路与案例解读 (PPT P18-P24，双核心配置实例一)：**

    *   **VLAN规划：** 假设有VLAN10, VLAN20, VLAN30, VLAN40。
    *   **IP地址规划：**
        *   RG-S35A: VLAN10 IP 192.168.10.254, VLAN20 IP 192.168.20.253
        *   RG-S35B: VLAN10 IP 192.168.10.253, VLAN20 IP 192.168.20.254
        *   虚拟IP: VLAN10 192.168.10.250, VLAN20 192.168.20.250

    *   **VRRP配置实现网关负载均衡 (PPT P19-P21)：**
        *   **目标：** S35A做VLAN10的主网关，S35B做VLAN20的主网关。
        *   **S35A上：**
            *   `interface Vlan10`
                *   `ip address 192.168.10.254 255.255.255.0`
                *   `standby 1 ip 192.168.10.250` (VRID 1, 虚拟IP)
                *   `standby 1 priority 254` (高优先级，成为Master)
                *   `standby 1 preempt` (允许抢占)
            *   `interface Vlan20`
                *   `ip address 192.168.20.253 255.255.255.0`
                *   `standby 2 ip 192.168.20.250` (VRID 2, 虚拟IP)
                *   `standby 2 priority 100` (默认或较低优先级，成为Backup)
                *   `standby 2 preempt`
        *   **S35B上 (配置与S35A相反)：**
            *   `interface Vlan10`
                *   `standby 1 priority 100` (Backup)
            *   `interface Vlan20`
                *   `standby 2 priority 254` (Master)
        *   **结果：** VLAN10的流量默认网关是S35A，VLAN20的流量默认网关是S35B。

    *   **MSTP配置实现L2负载均衡 (PPT P23-P24)：**
        *   **目标：** 使VLAN10,30的L2路径根桥为S35A，VLAN20,40的L2路径根桥为S35B，使L2路径与L3网关尽量一致。
        *   **在所有参与MSTP的交换机上 (S35A, S35B, S21A, S21B)：**
            *   `spanning-tree mode mstp`
            *   `spanning-tree mst configuration`
                *   `name CAMPUS_CORE`
                *   `revision 1`
                *   `instance 1 vlan 10,30`
                *   `instance 2 vlan 20,40`
                *   `exit`
        *   **S35A上 (作为Instance 1的根，Instance 2的备份根)：**
            *   `spanning-tree mst 1 priority 4096` (Instance 1根桥，值小优先)
            *   `spanning-tree mst 2 priority 8192` (Instance 2备份根桥)
        *   **S35B上 (作为Instance 2的根，Instance 1的备份根)：**
            *   `spanning-tree mst 1 priority 8192`
            *   `spanning-tree mst 2 priority 4096`
        *   **接入交换机S21A, S21B上：**
            *   `spanning-tree mst 1 priority 32768` (默认或更高值)
            *   `spanning-tree mst 2 priority 32768`
        *   **解释 (PPT P24)：** 配置S35A为Instance 1的根桥，是为了让VLAN10,30的流量优先通过S35A。如果S35B成为Instance 1的根桥，那么VLAN10的流量会先走L2路径到S35B，但VLAN10的VRRP Master是S35A，数据包可能需要再从S35B折返到S35A处理，形成次优路径。**核心原则：L2的根桥路径应尽可能与L3的VRRP Master路径一致。**

    *   **OSPF Cost调整实现下行负载均衡 (PPT P22)：**
        *   **场景：** 上行流量（用户->Internet）可以通过VRRP和MSTP实现负载均衡。下行流量（Internet->用户）到达S35C后，S35C如何选择路径到S35A或S35B？通常通过动态路由协议（如OSPF）的Cost值。
        *   **目标：** VLAN10的下行流量走S35C->S35A，VLAN20的下行流量走S35C->S35B。
        *   **S35A上：**
            *   `interface vlan 20` (这是S35A与S35C互联的VLAN，或者S35A上代表VLAN20用户网段的SVI)
                *   `ip ospf cost 65535` (调高VLAN20在S35A上的OSPF cost，使其不优)
        *   **S35B上：**
            *   `interface vlan 10`
                *   `ip ospf cost 65535` (调高VLAN10在S35B上的OSPF cost)
        *   **结果：** S35C学习到的去往192.168.10.0/24网段的路径，通过S35A的cost较低；去往192.168.20.0/24网段的路径，通过S35B的cost较低。

4.  **案例二解读 (PPT P25-P31，北京工行支行局域网)：**
    *   **VLAN规划 (P26)：** 按业务类型划分VLAN，如生产用户、OA用户、服务器等。
    *   **MSTP优先级 (P27, P29)：** 主核心优先级4096，备核心8192。这里使用了`mst 0`，通常指CIST (公共和内部生成树)或默认实例。如果所有VLAN都在一个实例内，则可以这样配置，但负载均衡效果不如多实例精细。
    *   **VRRP组规划 (P28, P30)：**
        *   例如VLAN 400 (生产用户)，VRRP组号为1。
        *   主核心配置：`interface vlan 400`, `ip address 85.34.192.251`, `standby 1 ip 85.34.192.254`, `standby 1 priority 105`。
        *   备份核心配置：`interface vlan 400`, `ip address 85.34.192.252`, `standby 1 ip 85.34.192.254` (优先级默认为100)。
    *   **参考拓扑与分析 (P31-P33)：**
        *   P32的问答说明了MSTP中实例的根桥选举（基于优先级）和根端口选择（离根桥最近）。
        *   P33的问答说明了VRRP主备切换的透明性，以及正常情况下PC2访问Internet的路径 (PC2 -> S2924G-1 -> S7606-1 -> 防火墙 -> 边界路由 -> Internet)。这个路径是基于S7606-1是VLAN10的VRRP Master。

---

**四、 MSTP+VRRP 注意事项 (PPT P35)**

1.  **VRRP中出现多个Master：**
    *   **短期现象：** 在Master故障，Backup接管的瞬间，或Master恢复并抢占的瞬间，可能短暂出现，属于正常。
    *   **长期现象（问题）：**
        *   **原因：** Master之间收不到VRRP通告报文（如中间链路故障、防火墙阻挡了VRRP报文的多播地址224.0.0.18），或VRRP配置不一致（如虚拟IP、认证、定时器等在同一组内不匹配）。
        *   **后果：** 网络中存在多个设备响应同一个虚拟IP，导致IP冲突、通信时断时续。
        *   **解决：**
            *   检查Master设备之间的网络连通性（ping物理IP）。
            *   确保同一VRRP组内所有路由器的虚拟IP、VRID、认证（如果配置）、定时器等参数完全一致。
            *   检查中间设备是否允许VRRP报文通过。

2.  **MSTP与VRRP配合的关键：**
    *   **L2路径与L3网关一致性：** 确保特定VLAN的MSTP根桥（流量汇聚点）与该VLAN的VRRP Master路由器是同一台设备，或者它们之间的路径最优，避免流量绕行。
    *   **MSTP域配置一致：** 同一MSTP域内的所有交换机必须有相同的域名、修订级别和VLAN-实例映射。否则无法形成正确的MST域。

3.  **其他可能问题：**
    *   **链路类型：** 核心之间、核心与接入之间的链路应配置为Trunk，并允许所有需要的VLAN通过。
    *   **端口状态：** 确保MSTP没有错误地阻塞了关键路径。
    *   **资源消耗：** MSTP和VRRP都会消耗设备CPU资源，尤其是在大型网络或频繁切换时。

---

**五、 补充知识点 (PPT P36-P55中提及的相关技术)**

1.  **交换机堆叠/集群 (PPT P36-P39)：**
    *   如华为CSS/iStack，思科VSS/StackWise。
    *   将多台物理交换机虚拟化为一台逻辑交换机。
    *   **优点：** 简化管理（一个管理IP），增加带宽（成员间链路聚合），提高可靠性（一台故障不影响逻辑设备）。
    *   **与MSTP/VRRP关系：** 堆叠后，逻辑上是一台设备，堆叠成员间不需要运行STP。如果核心由堆叠系统构成，VRRP仍然可以在逻辑设备上运行SVI接口，或者两套堆叠系统分别运行VRRP。

2.  **VTP (VLAN Trunking Protocol) (PPT P47-P48)：**
    *   思科私有协议，用于在交换网络中同步VLAN数据库信息。
    *   模式：Server, Client, Transparent。
    *   **与MSTP对比：** VTP管理VLAN的创建/删除/同步，MSTP管理VLAN的L2路径。它们可以一起使用，但MSTP是标准协议，VTP是私有协议。

3.  **Q-in-Q (IEEE 802.1ad) (PPT P51-P53)：**
    *   VLAN堆叠技术，在用户VLAN标签外再封装一层运营商VLAN标签 (S-TAG)。
    *   用于城域网，帮助运营商透传用户VLAN，解决VLAN ID数量限制问题。

---

**总结：**

MSTP+VRRP是构建可靠、高效校园网或企业网核心的经典组合。MSTP负责L2的无环和负载均衡，VRRP负责L3网关的高可用。配置时需仔细规划VLAN、IP、MSTP实例与优先级、VRRP组与优先级，并力求L2转发路径与L3活动网关一致，以达到最佳性能。同时，注意排查可能导致VRRP多Master或MSTP计算错误的配置问题和连通性问题。