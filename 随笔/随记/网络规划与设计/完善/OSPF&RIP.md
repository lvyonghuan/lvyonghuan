**网络路由协议复习笔记：OSPF & RIP**

**第一部分：OSPF (开放最短路径优先)**

**一、OSPF 简介与核心概念**

1.  **定义**:
    *   OSPF (Open Shortest Path First) 是一种开放标准的、**链路状态 (Link-State)** 路由协议。
    *   专为 IP 网络设计，使用 **SPF (Dijkstra) 算法**计算到达每个目标网络的最短路径。
    *   OSPFv2 (RFC 2328) 用于 IPv4，OSPFv3 (RFC 5340) 用于 IPv6。
    *   **无类路由协议 (Classless)**，支持 VLSM (可变长子网掩码) 和 CIDR (无类域间路由)。

2.  **特点**:
    *   **管理距离 (AD)**: 默认为 **110**。
    *   **度量值 (Metric)**: **Cost (开销)**，基于接口带宽计算 (Cost = 参考带宽 / 接口带宽)。参考带宽默认为 100 Mbps。
    *   **封装**: 直接封装在 IP 数据包中，**协议号为 89**。不使用 TCP 或 UDP。
    *   **更新方式**: 触发式更新和周期性更新 (LSA 每30分钟刷新一次)。
    *   **组播地址**:
        *   `224.0.0.5`: 发送给所有 OSPF 路由器 (AllSPFRouters)。
        *   `224.0.0.6`: 发送给指定路由器 DR 和备份指定路由器 BDR (AllDRouters)。
    *   **区域 (Area)**: 支持分层结构，通过将网络划分为多个区域来提高可扩展性。

3.  **OSPF 开发历程 (参考PDF Page 5)**:
    *   1987: IETF OSPF 工作组成立
    *   1991: OSPFv2 在 RFC 1247 中发布
    *   1998: OSPFv2 现行规范在 RFC 2328 中发布
    *   1999: 用于 IPv6 的 OSPFv3 在 RFC 2740 (后更新为 RFC 5340) 中发布

**二、OSPF 数据包类型 (OSPF Packet Types)**

OSPF 定义了5种主要的数据包类型：

1.  **Hello (类型 1)**:
    *   **功能**: 发现和维持邻居关系；选举 DR (Designated Router) 和 BDR (Backup Designated Router)；通告建立邻居关系必须统一的参数。
    *   **Hello 包关键参数 (必须匹配才能形成邻居)**:
        *   区域 ID (Area ID)
        *   认证类型和密码 (Authentication Type & Password)
        *   Hello 和 Dead 间隔 (Hello and Dead Intervals)
        *   末梢区域标记 (Stub Area Flag)
        *   MTU (Maximum Transmission Unit) - 在某些平台上，MTU不匹配会导致邻居卡在ExStart/Exchange状态。
    *   **Hello 间隔**: 默认在广播和点对点网络是 10 秒；在 NBMA 网络是 30 秒。
    *   **Dead 间隔**: 默认是 Hello 间隔的 4 倍 (广播和P2P是40秒, NBMA是120秒)。如果在 Dead 间隔内未收到邻居的 Hello 包，则认为邻居失效。

2.  **DBD (Database Description - 类型 2)**:
    *   **功能**: 描述本地 LSDB (Link-State Database) 的摘要信息，用于邻居间同步 LSDB。
    *   包含 LSA 头部信息列表。
    *   在邻居关系建立的 ExStart 和 Exchange 状态使用。

3.  **LSR (Link-State Request - 类型 3)**:
    *   **功能**: 当一台路由器收到 DBD 包并发现自己缺少某些 LSA 或 LSA 版本较旧时，会发送 LSR 请求完整的 LSA 信息。

4.  **LSU (Link-State Update - 类型 4)**:
    *   **功能**: 响应 LSR，发送邻居请求的完整 LSA 信息。也用于泛洪新的或更新的 LSA。
    *   LSU 可以包含一个或多个 LSA。

5.  **LSAck (Link-State Acknowledgement - 类型 5)**:
    *   **功能**: 对收到的 LSU 进行确认，确保 LSA 传输的可靠性。

**三、OSPF 邻居与邻接关系 (Neighbor & Adjacency)**

1.  **邻居 (Neighbor)**: 两台 OSPF 路由器通过 Hello 包互相发现对方。
2.  **邻接 (Adjacency)**: 邻居关系发展到可以交换 LSA 的阶段。只有形成邻接关系的路由器才会同步 LSDB。
3.  **邻居状态机**:
    *   **Down**: 初始状态，未收到 Hello。
    *   **Init**: 已收到对方的 Hello 包，但对方的 Hello 包中没有包含自己的 Router ID。
    *   **2-Way**: 双向通信已建立。在广播和NBMA网络中，如果双方都不是DR或BDR，则邻居关系停留在此状态。如果是点对点网络或与DR/BDR之间，会继续发展。
    *   **ExStart**: 选举主从路由器 (Master/Slave) 以决定 DBD 包的初始序列号。Router ID 大的成为 Master。
    *   **Exchange**: 交换 DBD 包，描述各自的 LSDB。
    *   **Loading**: 发送 LSR 请求缺失的 LSA，通过 LSU 接收 LSA，并通过 LSAck 确认。
    *   **Full**: LSDB 同步完成，形成邻接关系。

**四、DR 与 BDR (Designated Router & Backup Designated Router)**

1.  **目的**: 在多路访问网络 (如以太网) 中，为减少 OSPF 流量和邻接数量。
    *   若没有 DR/BDR，n 台路由器需要 n(n-1)/2 个邻接关系。
    *   有了 DR/BDR，其他路由器 (DROther) 只与 DR 和 BDR 形成 Full 邻接关系，DROther 之间是 2-Way 邻居。
2.  **选举过程**:
    *   **网络类型**: 只在广播 (Broadcast) 和非广播多路访问 (NBMA) 网络类型中选举。点对点 (Point-to-Point) 网络不选举。
    *   **标准**:
        1.  比较接口的 OSPF **优先级 (Priority)**: 默认为 1 (0-255)。优先级最高的成为 DR，次高的成为 BDR。优先级为 0 表示不参与选举。
        2.  如果优先级相同，则比较 **Router ID**: Router ID 最高的成为 DR，次高的成为 BDR。
    *   **非抢占性**: 一旦 DR 和 BDR 选举完成，即使有更高优先级或 Router ID 的路由器加入，DR/BDR 也不会改变，除非 DR 或 BDR 故障。
3.  **Router ID 确定顺序**:
    1.  通过 `router-id <A.B.C.D>` 命令手动配置。
    2.  如果没有手动配置，选择所有 Loopback 接口中最高的 IP 地址。
    3.  如果没有 Loopback 接口，选择所有活动的物理接口中最高的 IP 地址。
    *   **注意**: Router ID 一旦选定，在 OSPF 进程重启 (`clear ip ospf process`) 或路由器重启前不会改变，即使之后配置了更高优先级的 IP 地址。

**五、链路状态通告 LSA (Link-State Advertisement)**

LSA 是 OSPF 的核心，用于描述网络拓扑信息。

| LSA 类型 | 名称                   | 产生者          | 范围 (Scope)      | 路由表显示 | 描述                                                                 |
| :------- | :--------------------- | :-------------- | :---------------- | :--------- | :------------------------------------------------------------------- |
| **1**    | **Router LSA**         | 所有路由器      | 区域内 (Intra-area) | O          | 描述路由器自身所有链路状态、接口 Cost、邻居信息。                       |
| **2**    | **Network LSA**        | DR              | 区域内 (Intra-area) | O          | 描述多路访问网络上连接的所有路由器列表，仅在有 DR 的网络段产生。        |
| **3**    | **Summary LSA**        | ABR             | 区域间 (Inter-area) | O IA       | 将一个区域内的路由信息（来自LSA1/2）汇总后通告到其他区域。               |
| **4**    | **ASBR Summary LSA**   | ABR             | 区域间 (Inter-area) | (不直接显示) | 告诉其他区域如何到达 ASBR。                                            |
| **5**    | **AS External LSA**    | ASBR            | 整个 OSPF 自治系统 | O E1/E2    | 描述从其他自治系统（如RIP、EIGRP、静态）重分布进来的外部路由。          |
| **7**    | **NSSA External LSA**  | NSSA区域内的ASBR | NSSA 区域内       | O N1/N2    | 在 NSSA 区域内通告外部路由，到达 NSSA ABR 时会转换为 LSA-5 传播到其他区域。 |

*   **LSA 泛洪**: LSA 在其定义的范围内进行泛洪。例如 LSA-1 和 LSA-2 只在区域内泛洪。LSA-3 由 ABR 产生并泛洪到其他区域。
*   **LSDB (Link-State Database)**: 每台 OSPF 路由器都维护一个 LSDB，包含其所在区域的所有 LSA 以及从其他区域学习到的 LSA 摘要。同一区域内的所有路由器的 LSDB 最终应该是一致的。

**六、OSPF 区域 (Areas)**

1.  **目的**:
    *   **减小 LSDB 大小**: 每个区域维护自己的 LSDB。
    *   **减少 SPF 算法计算开销**: 拓扑变化限制在区域内，只有区域内路由器重新计算 SPF。
    *   **减少路由表条目**: 通过 ABR 进行区域间路由汇总。
    *   **提高网络稳定性**: 区域故障隔离。
2.  **区域类型**:
    *   **骨干区域 (Backbone Area - Area 0)**:
        *   所有其他非骨干区域必须直接连接到 Area 0 (逻辑上或通过虚链路)。
        *   区域间路由必须经过 Area 0。
        *   可以接收和发起所有类型的 LSA (除了 LSA-7)。
    *   **标准区域 (Standard Area / Normal Area)**:
        *   默认区域类型。
        *   可以接收和发起 LSA-1, 2, 3, 4, 5。
    *   **末梢区域 (Stub Area)**:
        *   **不允许 LSA-5 (外部路由)** 进入。ASBR 不能存在于 Stub Area。
        *   ABR 会自动向 Stub Area 注入一条 LSA-3 类型的默认路由 (0.0.0.0/0)。
        *   允许 LSA-1, 2, 3。
        *   配置: 在区域内所有路由器上配置 `area <area-id> stub`。
    *   **完全末梢区域 (Totally Stubby Area - Cisco 私有)**:
        *   不仅**不允许 LSA-5**，也**不允许 LSA-3 (区域间路由)** 进入，除了 ABR 自动注入的 LSA-3 默认路由。
        *   进一步减小路由表和 LSDB。
        *   配置: 区域内普通路由器配置 `area <area-id> stub`；ABR 上配置 `area <area-id> stub no-summary`。
    *   **非纯末梢区域 (Not-So-Stubby Area - NSSA)**:
        *   **不允许 LSA-5** 进入，但**允许 ASBR 存在**于 NSSA 区域内。
        *   NSSA 内的 ASBR 产生的外部路由以 **LSA-7** 的形式在 NSSA 区域内泛洪。
        *   NSSA ABR 会将 LSA-7 转换为 LSA-5 并通告到其他区域。
        *   ABR **不会自动**向 NSSA 区域注入默认路由，除非手动配置。
        *   配置: 在区域内所有路由器上配置 `area <area-id> nssa`。若需默认路由，在 NSSA ABR 上配置 `area <area-id> nssa default-information-originate`。
    *   **完全非纯末梢区域 (Totally NSSA - Cisco 私有)**:
        *   NSSA 的增强，不仅不允许 LSA-5，也不允许 LSA-3 (除了 ABR 自动注入的 LSA-3 默认路由)。
        *   配置: 区域内普通路由器配置 `area <area-id> nssa`；ABR 上配置 `area <area-id> nssa no-summary`。若需从 ABR 注入默认路由到 Totally NSSA，ABR 上配置 `area <area-id> nssa no-summary default-information-originate` (某些IOS版本此命令会自动注入默认路由，某些需要明确的 `default-information-originate` )。

3.  **OSPF 路由器类型**:
    *   **内部路由器 (Internal Router - IR)**: 所有接口都在同一个区域内。
    *   **区域边界路由器 (Area Border Router - ABR)**: 接口连接不同区域 (至少一个接口在 Area 0)。负责区域间路由汇总和 LSA-3 的生成。
    *   **骨干路由器 (Backbone Router - BR)**: 至少有一个接口在 Area 0。所有 ABR 都是 BR。
    *   **自治系统边界路由器 (Autonomous System Boundary Router - ASBR)**: 将来自其他路由协议 (如 RIP, EIGRP, BGP) 或静态路由重分布到 OSPF 自治系统中的路由器。负责生成 LSA-5 (或 LSA-7)。

**七、OSPF 配置与验证**

1.  **基本配置步骤**:
    1.  **启动 OSPF 进程**:
        ```routeros
        Router(config)# router ospf <process-id>
        ```
        *   `<process-id>`: 1-65535，本地有效，不同路由器上可以不同。
    2.  **配置 Router ID (推荐)**:
        ```routeros
        Router(config-router)# router-id <A.B.C.D>
        ```
        或配置 Loopback 接口并分配 IP 地址。
    3.  **宣告参与 OSPF 的网络及区域**:
        ```routeros
        Router(config-router)# network <network-address> <wildcard-mask> area <area-id>
        ```
        *   `<network-address>`: 可以是网络号、子网号或接口IP。
        *   `<wildcard-mask>`: 反掩码。
        *   `<area-id>`: 区域号，可以是十进制或点分十进制。
    4.  **(可选) 修改接口 Cost**:
        ```routeros
        Router(config-if)# ip ospf cost <value>
        ```
    5.  **(可选) 修改参考带宽 (全局影响)**:
        ```routeros
        Router(config-router)# auto-cost reference-bandwidth <Mbps>
        ```
        *   例如: `auto-cost reference-bandwidth 1000` (将参考带宽设为1Gbps)。
    6.  **(可选) 修改接口 OSPF 优先级 (影响 DR/BDR 选举)**:
        ```routeros
        Router(config-if)# ip ospf priority <0-255>
        ```
    7.  **(可选) 配置认证**:
        *   接口认证:
            ```routeros
            Router(config-if)# ip ospf authentication [message-digest | null]
            Router(config-if)# ip ospf authentication-key <password>  (明文)
            Router(config-if)# ip ospf message-digest-key <key-id> md5 <password> (MD5)
            ```
        *   区域认证 (简化配置，但底层仍是接口认证):
            ```routeros
            Router(config-router)# area <area-id> authentication [message-digest]
            ```

2.  **验证命令**:
    *   `show ip ospf neighbor`: 查看邻居状态和 DR/BDR 信息。
    *   `show ip ospf interface [type number]`: 查看接口 OSPF 参数 (Cost, Hello/Dead, Area, Priority, Network Type, DR/BDR)。
    *   `show ip protocols`: 查看 OSPF 进程 ID, Router ID, 宣告的网络, AD 等。
    *   `show ip route ospf`: 查看 OSPF 学习到的路由。
    *   `show ip ospf database`: 查看 LSDB 内容。
        *   `show ip ospf database router`: 查看 LSA-1
        *   `show ip ospf database network`: 查看 LSA-2
        *   `show ip ospf database summary`: 查看 LSA-3
        *   `show ip ospf database asbr-summary`: 查看 LSA-4
        *   `show ip ospf database external`: 查看 LSA-5
        *   `show ip ospf database nssa-external`: 查看 LSA-7
    *   `debug ip ospf hello`: 调试 Hello 包。
    *   `debug ip ospf adj`: 调试邻接关系建立过程。
    *   `clear ip ospf process`: 重启 OSPF 进程 (慎用，会导致邻居中断)。

**八、OSPF 路由控制**

1.  **路由汇总 (Route Summarization)**:
    *   **区域间汇总 (Inter-Area Summarization)**: 在 **ABR** 上配置。
        ```routeros
        Router(config-router)# area <source-area-id> range <summary-address> <summary-mask> [not-advertise | cost <value>]
        ```
        *   `<source-area-id>`: 要汇总的路由所在的区域。
        *   汇总后的路由作为 LSA-3 发送到其他区域。
    *   **外部路由汇总 (External Route Summarization)**: 在 **ASBR** 上配置。
        ```routeros
        Router(config-router)# summary-address <summary-address> <summary-mask> [not-advertise | tag <value>]
        ```
        *   汇总的是重分布进来的外部路由。
        *   汇总后的路由作为 LSA-5 发布。

2.  **路由重分布 (Route Redistribution)**:
    *   在 **ASBR** 上配置，将其他路由协议的路由导入 OSPF。
        ```routeros
        Router(config-router)# redistribute <source-protocol> [process-id | tag] [metric <value>] [metric-type <1|2>] [subnets] [route-map <map-name>]
        ```
        *   `<source-protocol>`: rip, eigrp, static, connected 等。
        *   `metric <value>`: 指定外部路由的初始 OSPF Cost。
        *   `metric-type <1|2>`:
            *   **Type 2 (E2/N2 - 默认)**: Cost 固定为初始重分布时的 metric，不累加内部路径 Cost。
            *   **Type 1 (E1/N1)**: Cost = 初始 metric + 到达 ASBR 的内部路径 Cost。
        *   `subnets`: 导入子网路由 (如果源协议支持)。
        *   `route-map`: 用于过滤或修改重分布的路由属性。

3.  **默认路由 (Default Route)**:
    *   **在 ASBR 上向 OSPF 注入默认路由**:
        ```routeros
        Router(config-router)# default-information originate [always] [metric <value>] [metric-type <1|2>]
        ```
        *   ASBR 必须自身有一条默认路由才能注入，除非使用 `always` 关键字。
        *   产生的 LSA-5 默认路由。
    *   **在 ABR 上向 Stub/Totally Stub/Totally NSSA 区域注入默认路由**: 通常自动发生或通过特定区域配置命令。

**九、OSPF 案例**

**案例1: 单区域 OSPF 配置 (点对点)**
R1 (Fa0/0: 10.1.1.1/24, Lo0: 1.1.1.1/32) --- R2 (Fa0/0: 10.1.1.2/24, Lo0: 2.2.2.2/32)

```routeros
! R1 Configuration
hostname R1
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
interface FastEthernet0/0
 ip address 10.1.1.1 255.255.255.0
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.255 area 0
!
! R2 Configuration
hostname R2
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
interface FastEthernet0/0
 ip address 10.1.1.2 255.255.255.0
 no shutdown
!
router ospf 1
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0
 network 10.1.1.0 0.0.0.255 area 0
```
验证: `show ip ospf neighbor` (状态应为 FULL), `show ip route ospf` (应能看到对方的 Lo0)。

**案例2: 多区域 OSPF 与 DR/BDR (广播网络)**
R1 (Area 1, Fa0/0: 192.168.1.1/24, Lo0: 1.1.1.1/32)
R2 (ABR, Fa0/0: 192.168.1.2/24 Area 1, Fa0/1: 172.16.0.1/24 Area 0, Lo0: 2.2.2.2/32)
R3 (Area 0, Fa0/1: 172.16.0.2/24, Lo0: 3.3.3.3/32)
*假设 192.168.1.0/24 是广播网络，R2 优先级设为 100，R1 优先级设为 50，R2 应为 DR。*

```routeros
! R1 Configuration (Area 1)
hostname R1
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 ip ospf priority 50
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 1
 network 192.168.1.0 0.0.0.255 area 1
!
! R2 Configuration (ABR - Area 1 & Area 0)
hostname R2
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
interface FastEthernet0/0
 ip address 192.168.1.2 255.255.255.0
 ip ospf priority 100
 no shutdown
interface FastEthernet0/1
 ip address 172.16.0.1 255.255.255.0
 no shutdown
!
router ospf 1
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0  ! Lo0 放在 Area 0 (通常做法)
 network 192.168.1.0 0.0.0.255 area 1
 network 172.16.0.0 0.0.0.255 area 0
!
! R3 Configuration (Area 0)
hostname R3
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
interface FastEthernet0/1
 ip address 172.16.0.2 255.255.255.0
 no shutdown
!
router ospf 1
 router-id 3.3.3.3
 network 3.3.3.3 0.0.0.0 area 0
 network 172.16.0.0 0.0.0.255 area 0
```
验证:
在 R1: `show ip ospf neighbor` (R2 应为 DR，状态 FULL)。
在 R2: `show ip ospf neighbor` (对 R1, R1为DROTHER, 状态FULL; 对R3, 若172.16.0.0/24也为广播，则根据优先级/RID选举DR/BDR)。
在 R1: `show ip route ospf` (应能看到 O IA 3.3.3.3/32)。
在 R3: `show ip route ospf` (应能看到 O IA 1.1.1.1/32)。

**案例3: Stub Area 配置**
沿用案例2，将 Area 1 配置为 Stub Area。
```routeros
! R1 Configuration (Area 1 - Stub)
router ospf 1
 area 1 stub
 ! 其他配置不变
!
! R2 Configuration (ABR - Area 1 Stub)
router ospf 1
 area 1 stub
 ! 其他配置不变
```
验证: 在 R1 上 `show ip route ospf`，将看到一条来自 R2 的默认路由 `O*IA 0.0.0.0/0`，并且不会看到 Area 0 的具体路由 (如 3.3.3.3/32 和 172.16.0.0/24 会被默认路由取代)。

**案例4: 路由汇总 (ABR)**
沿用案例2，R1 有多个 Loopback (Lo1: 10.0.1.1/32, Lo2: 10.0.2.1/32) 都在 Area 1。在 R2 (ABR) 上汇总它们。
```routeros
! R1 - 增加 Loopbacks
interface Loopback1
 ip address 10.0.1.1 255.255.255.255
interface Loopback2
 ip address 10.0.2.1 255.255.255.255
!
router ospf 1
 network 10.0.1.1 0.0.0.0 area 1
 network 10.0.2.1 0.0.0.0 area 1
 ! 其他配置不变
!
! R2 - 配置区域间汇总 (Area 1 的路由汇总后通告给 Area 0)
router ospf 1
 area 1 range 10.0.0.0 255.255.252.0  ! 假设汇总为 10.0.0.0/22
 ! 其他配置不变
```
验证: 在 R3 上 `show ip route ospf`，应能看到一条汇总路由 `O IA 10.0.0.0/22`，而不是 10.0.1.1/32 和 10.0.2.1/32。

**案例5: 重分布 RIP 路由到 OSPF**
R3 连接到一个 RIP 网络 (例如，通过 S0/0 接口)，R3 作为 ASBR。
RIP 网络: 100.1.1.0/24
```routeros
! R3 Configuration (ASBR)
! 假设 S0/0 连接到 RIP 网络
interface Serial0/0
 ip address 100.1.1.3 255.255.255.0
 encapsulation ppp
 no shutdown
!
router rip
 version 2
 network 100.0.0.0
 no auto-summary
!
router ospf 1
 router-id 3.3.3.3
 redistribute rip subnets metric 20 metric-type 2
 network 3.3.3.3 0.0.0.0 area 0
 network 172.16.0.0 0.0.0.255 area 0
 ! 如果S0/0也想运行OSPF, 则也宣告它，但通常重分布的接口不运行被重分布进的协议
```
验证: 在 R1 和 R2 上 `show ip route ospf`，应能看到 `O E2 100.1.1.0/24 [110/20]`。

---

**第二部分：RIP (路由信息协议)**

**一、RIP 简介与核心概念**

1.  **定义**:
    *   RIP (Routing Information Protocol) 是一种**距离矢量 (Distance Vector)** 路由协议。
    *   使用**跳数 (Hop Count)** 作为度量值。最大跳数为 15，16跳表示网络不可达。
    *   简单，适用于小型网络。

2.  **版本**:
    *   **RIPv1 (RFC 1058)**:
        *   **有类路由协议 (Classful)**: 不发送子网掩码信息，不支持 VLSM。
        *   使用**广播 (255.255.255.255)** 发送路由更新。
        *   不支持认证。
    *   **RIPv2 (RFC 2453)**:
        *   **无类路由协议 (Classless)**: 在路由更新中包含子网掩码，支持 VLSM 和 CIDR。
        *   使用**组播 (224.0.0.9)** 发送路由更新。
        *   支持明文和 MD5 认证。
        *   支持路由标记 (Route Tag)。

3.  **特点**:
    *   **管理距离 (AD)**: **120**。
    *   **封装**: UDP 协议，**端口号 520**。
    *   **更新方式**: 周期性广播/组播路由表 (默认每 30 秒)。

**二、RIP 工作原理与计时器**

1.  **工作原理**:
    *   路由器启动 RIP 后，向邻居发送 Request 请求路由信息。
    *   收到请求的路由器发送 Response 包含自己的路由表。
    *   周期性 (默认30秒) 发送完整的路由表给邻居。
    *   基于贝尔曼-福特算法 (Bellman-Ford) 的变种。

2.  **计时器**:
    *   **更新计时器 (Update Timer)**: 默认 **30 秒**，发送路由更新的时间间隔。
    *   **无效计时器 (Invalid Timer / Timeout Timer)**: 默认 **180 秒**。如果在该时间内未收到某条路由的更新，则将该路由标记为可能失效 (跳数设为16)。
    *   **抑制计时器 (Hold-down Timer - Cisco 特有)**: 默认 **180 秒**。当一条路由变为不可达后，在 Hold-down 期间，路由器不接受关于此路由的任何更新，除非新路由的度量值优于原路由。
    *   **刷新计时器 (Flush Timer)**: 默认 **240 秒**。如果在该时间内未收到某条路由的更新，则从路由表中删除该路由。

**三、RIP 环路防止机制**

1.  **最大跳数 (Maximum Hop Count)**: 15跳，16跳为不可达，防止路由在环路中无限传递。
2.  **水平分割 (Split Horizon)**: 从一个接口学到的路由，不会再从这个接口发回。
3.  **毒性逆转水平分割 (Poison Reverse with Split Horizon)**: 从一个接口学到的路由，会从这个接口发回，但跳数设为16 (不可达)。更强力地通告路由失效。
4.  **路由中毒 (Route Poisoning)**: 当一条网络失效时，路由器会立即将该路由的跳数设为16并通告出去。
5.  **触发更新 (Triggered Updates)**: 当网络拓扑发生变化 (如路由失效) 时，立即发送更新，而不等待下一个更新周期。

**四、RIP 配置与验证**

1.  **基本配置步骤**:
    1.  **启动 RIP 进程**:
        ```routeros
        Router(config)# router rip
        ```
    2.  **选择 RIP 版本 (推荐 RIPv2)**:
        ```routeros
        Router(config-router)# version 2
        ```
    3.  **宣告参与 RIP 的网络 (基于主类网络)**:
        ```routeros
        Router(config-router)# network <classful-network-address>
        ```
        *   例如，接口 IP 是 192.168.1.1/24，则宣告 `network 192.168.0.0`。
    4.  **关闭自动汇总 (推荐用于 RIPv2 和不连续子网)**:
        ```routeros
        Router(config-router)# no auto-summary
        ```
    5.  **(可选) 配置认证 (RIPv2)**:
        ```routeros
        Router(config-if)# ip rip authentication mode [text | md5]
        Router(config-if)# ip rip authentication key-chain <key-chain-name>
        !
        key chain <key-chain-name>
         key <key-id>
          key-string <password>
        ```
    6.  **(可选) 配置被动接口 (只接收更新，不发送更新)**:
        ```routeros
        Router(config-router)# passive-interface <interface-type_number>
        ```
        *   通常用于连接到 LAN 的接口，防止向 LAN 发送路由更新。

2.  **验证命令**:
    *   `show ip route rip`: 查看 RIP 学习到的路由。
    *   `show ip protocols`: 查看 RIP 配置信息 (版本, 宣告网络, 计时器等)。
    *   `show ip rip database`: (某些IOS版本) 查看 RIP 数据库。
    *   `debug ip rip [events]`: 调试 RIP 事件 (更新发送和接收)。
    *   `show ip interface <type number>`: 查看接口RIP是否启用。

**五、RIP 案例**

**案例1: RIPv1 基本配置**
R1 (Fa0/0: 192.168.1.1/24) --- R2 (Fa0/0: 192.168.1.2/24, Fa0/1: 10.1.1.1/24)

```routeros
! R1 Configuration
hostname R1
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
!
router rip
 network 192.168.1.0
!
! R2 Configuration
hostname R2
interface FastEthernet0/0
 ip address 192.168.1.2 255.255.255.0
 no shutdown
interface FastEthernet0/1
 ip address 10.1.1.1 255.255.255.0
 no shutdown
!
router rip
 network 192.168.1.0
 network 10.0.0.0
```
验证: 在 R1 `show ip route rip` 应能看到 `10.0.0.0/8 [120/1] via 192.168.1.2` (因为是v1，所以是主类)。

**案例2: RIPv2 与 no auto-summary (不连续子网)**
R1 (Lo0: 172.16.1.1/24) --- (S0/0: 10.0.0.1/30) R2 (S0/0: 10.0.0.2/30) --- (Lo0: 172.16.2.1/24)
*这是一个不连续网络，172.16.0.0 网络被 10.0.0.0 分隔。*

```routeros
! R1 Configuration
hostname R1
interface Loopback0
 ip address 172.16.1.1 255.255.255.0
interface Serial0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
router rip
 version 2
 network 172.16.0.0
 network 10.0.0.0
 no auto-summary
!
! R2 Configuration
hostname R2
interface Loopback0
 ip address 172.16.2.1 255.255.255.0
interface Serial0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
router rip
 version 2
 network 172.16.0.0
 network 10.0.0.0
 no auto-summary
```
验证:
在 R1 `show ip route rip` 应能看到 `172.16.2.0/24 [120/1] via 10.0.0.2`。
在 R2 `show ip route rip` 应能看到 `172.16.1.0/24 [120/1] via 10.0.0.1`。
如果 `auto-summary` (默认开启)，则路由无法正确学习。

---

**OSPF vs RIP 简要对比**

| 特性           | OSPF                                        | RIP                                                |
| :------------- | :------------------------------------------ | :------------------------------------------------- |
| **类型**       | 链路状态 (Link-State)                       | 距离矢量 (Distance Vector)                         |
| **算法**       | SPF (Dijkstra)                              | Bellman-Ford (变种)                                |
| **度量值**     | Cost (基于带宽)                             | 跳数 (Hop Count)                                   |
| **最大跳数**   | 无 (实际受限于SPF计算能力)                  | 15 (16为不可达)                                    |
| **收敛速度**   | 快                                          | 慢                                                 |
| **网络规模**   | 大型网络                                    | 小型网络                                           |
| **VLSM/CIDR**  | 支持                                        | RIPv1 不支持, RIPv2 支持                           |
| **更新方式**   | 触发式更新, 周期性LSA刷新 (30分钟)          | 周期性全路由表更新 (30秒)                          |
| **管理距离**   | 110                                         | 120                                                |
| **资源消耗**   | 较高 (CPU, 内存)                            | 较低                                               |
| **复杂性**     | 复杂                                        | 简单                                               |
| **分层结构**   | 支持区域 (Area)                             | 不支持                                             |
| **组播/广播**  | 组播 (224.0.0.5, 224.0.0.6)                 | RIPv1 广播, RIPv2 组播 (224.0.0.9)                 |
| **封装**       | IP (协议号 89)                              | UDP (端口 520)                                     |
| **认证**       | 支持明文和MD5                               | RIPv1 不支持, RIPv2 支持明文和MD5                  |