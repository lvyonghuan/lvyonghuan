**VLAN 规划与设计讲义**

**引言**
VLAN（虚拟局域网）是一种将物理局域网在逻辑上划分为多个广播域的技术。通过VLAN，可以实现网络资源的灵活分配、提高网络安全性、简化网络管理并控制广播风暴。本讲义将通过几个典型示例，介绍基于端口、基于MAC地址、基于协议以及基于IP子网的VLAN规划与设计方法。

---

**一、VLAN 规划与设计示例 - 概述**

*   **目标**：学习不同场景下VLAN的划分方法和配置。
*   **核心技术**：VLAN的创建、端口类型（Access, Trunk, Hybrid）、VLAN与端口的关联。

---

**二、基于端口的VLAN（Port-based VLAN）**

**1. 概念**
基于端口的VLAN是最常见和最简单的VLAN划分方式。交换机的每个物理端口被分配给一个特定的VLAN。连接到该端口的设备自动成为该VLAN的成员。

**2. 组网需求与示例 (参考图1-6)**
*   **需求**：
    *   Host A 和 Host C 属于部门A (VLAN 100)，通过不同交换机接入。
    *   Host B 和 Host D 属于部门B (VLAN 200)，通过不同交换机接入。
    *   实现部门间二层流量隔离，但同一VLAN内的主机（即使跨交换机）能够互通。
*   **网络拓扑**：
    *   两台交换机 (Device A, Device B) 通过 GE3/0/3 端口互连。
    *   Device A: GE3/0/1 连接 Host A (VLAN 100), GE3/0/2 连接 Host B (VLAN 200)。
    *   Device B: GE3/0/1 连接 Host C (VLAN 100), GE3/0/2 连接 Host D (VLAN 200)。
*   **配置思路**：
    *   在两台交换机上均创建 VLAN 100 和 VLAN 200。
    *   将连接主机的端口划入相应的VLAN (Access模式)。
    *   将交换机互连的端口配置为 Trunk模式，并允许VLAN 100和VLAN 200的报文通过。
    *   同一VLAN内的主机配置在同一IP网段。

*   **关键配置 (以Device A为例)**：
    ```nohighlight
    <DeviceA> system-view
    [DeviceA] vlan 100
    [DeviceA-vlan100] port GigabitEthernet 3/0/1
    [DeviceA-vlan100] quit

    [DeviceA] vlan 200
    [DeviceA-vlan200] port GigabitEthernet 3/0/2
    [DeviceA-vlan200] quit

    [DeviceA] interface GigabitEthernet 3/0/3
    [DeviceA-GigabitEthernet3/0/3] port link-type trunk
    [DeviceA-GigabitEthernet3/0/3] port trunk permit vlan 100 200
    [DeviceA-GigabitEthernet3/0/3] quit
    ```
*   **IP规划**：
    *   VLAN 100 (Host A, Host C): 如 192.168.100.0/24
    *   VLAN 200 (Host B, Host D): 如 192.168.200.0/24

---

**三、基于MAC地址的VLAN (MAC-based VLAN)**

**1. 概念**
基于MAC地址的VLAN是根据设备的源MAC地址来划分VLAN成员。当交换机端口收到untagged报文时，会检查其源MAC地址，并根据预设的MAC-VLAN映射关系，将报文划分到相应的VLAN中并打上Tag。
*   **手动配置静态MAC VLAN**：管理员手动配置MAC地址与VLAN的对应关系。
*   **动态MAC VLAN**：通常与802.1X认证配合，认证服务器下发VLAN信息，交换机动态创建MAC-VLAN表项。

**2. 组网需求与示例 (参考图1-7)**
*   **需求**：
    *   Laptop1 (MAC: 000d-88f8-4e71) 属于VLAN 100，只能访问 Server1。
    *   Laptop2 (MAC: 0014-222c-aa69) 属于VLAN 200，只能访问 Server2。
    *   两台笔记本电脑可能在不同会议室（连接到Device A或Device C的GE3/0/1）移动使用。
*   **网络拓扑**：
    *   Device A 和 Device C 为接入交换机，Device B 为汇聚/核心交换机。
    *   Laptop1/2 连接 Device A/C 的 GE3/0/1。
    *   Server1 (VLAN 100) 连接 Device B 的 GE3/0/13。
    *   Server2 (VLAN 200) 连接 Device B 的 GE3/0/14。
*   **配置思路**：
    *   在所有交换机上创建 VLAN 100 和 VLAN 200。
    *   Device A 和 Device C 的接入端口 (GE3/0/1) 配置为 Hybrid 模式，启用 MAC-VLAN 功能，并允许 VLAN 100 和 200 的报文untagged通过。
    *   在 Device A 和 Device C 上配置 MAC 地址与 VLAN 的映射关系。
    *   交换机间的连接端口配置为 Trunk 模式。
    *   Device B 上连接服务器的端口划入相应的 VLAN。
*   **关键配置 (以Device A为例)**：
    ```nohighlight
    <DeviceA> system-view
    [DeviceA] vlan 100
    [DeviceA] vlan 200
    [DeviceA] quit

    [DeviceA] mac-vlan mac-address 000d-88f8-4e71 vlan 100
    [DeviceA] mac-vlan mac-address 0014-222c-aa69 vlan 200

    [DeviceA] interface GigabitEthernet 3/0/1  // 连接Laptop的端口
    [DeviceA-GigabitEthernet3/0/1] port link-type hybrid
    [DeviceA-GigabitEthernet3/0/1] port hybrid vlan 100 200 untagged // 允许这些VLAN的报文untagged出端口
    [DeviceA-GigabitEthernet3/0/1] mac-vlan enable
    [DeviceA-GigabitEthernet3/0/1] quit

    [DeviceA] interface GigabitEthernet 3/0/2  // 连接Device B的端口
    [DeviceA-GigabitEthernet3/0/2] port link-type trunk
    [DeviceA-GigabitEthernet3/0/2] port trunk permit vlan 100 200
    [DeviceA-GigabitEthernet3/0/2] quit
    ```
*   **注意事项**：
    *   基于MAC的VLAN只能在Hybrid端口上配置。
    *   主要用于用户接入设备的下行端口，不与聚合功能同时使用。

---

**四、基于协议的VLAN (Protocol-based VLAN)**

**1. 概念**
基于协议的VLAN是根据端口接收到的untagged报文的协议类型（如IP, IPX, AppleTalk）及封装格式（如Ethernet II, 802.3 raw）来将其划分到不同的VLAN。
*   通过定义“协议类型+封装格式”的协议模板，并将其与VLAN ID及端口绑定。
*   此特性只对Hybrid端口配置才有效。

**2. 组网需求与示例 (参考图1-8)**
*   **需求**：
    *   实验室内有运行IPv4协议的主机和服务器，也有运行IPv6协议的主机和服务器。
    *   要求将IPv4流量和IPv6流量在二层进行隔离。
*   **网络拓扑**：
    *   一台核心交换机 (Device) 连接 IPv4 Server (VLAN 100) 和 IPv6 Server (VLAN 200)。
    *   两台接入交换机 (L2 Switch A, L2 Switch B) 连接各类主机。
*   **配置思路**：
    *   在核心交换机 Device 上创建 VLAN 100 (用于IPv4) 和 VLAN 200 (用于IPv6)。
    *   将服务器端口划入相应VLAN。
    *   在 VLAN 视图下创建协议模板，绑定协议类型 (IPv4, IPv6)。
    *   将连接L2 Switch的端口配置为Hybrid模式，并绑定协议模板，使得IPv4报文进入VLAN 100，IPv6报文进入VLAN 200。
*   **关键配置 (以Device为例)**：
    ```nohighlight
    <Device> system-view
    [Device] vlan 100
    [Device-vlan100] description protocol VLAN for IPv4
    [Device-vlan100] port GigabitEthernet 3/0/11  // 连接IPv4 Server
    [Device-vlan100] protocol-vlan 1 ipv4         // 定义协议模板1为IPv4
    [Device-vlan100] quit

    [Device] vlan 200
    [Device-vlan200] description protocol VLAN for IPv6
    [Device-vlan200] port GigabitEthernet 3/0/12  // 连接IPv6 Server
    [Device-vlan200] protocol-vlan 1 ipv6         // 定义协议模板1为IPv6
    [Device-vlan200] quit

    [Device] interface GigabitEthernet 3/0/1      // 连接L2 Switch A
    [Device-GigabitEthernet3/0/1] port link-type hybrid
    [Device-GigabitEthernet3/0/1] port hybrid vlan 100 200 untagged
    [Device-GigabitEthernet3/0/1] port hybrid protocol-vlan vlan 100 1 // IPv4报文进入VLAN 100 (使用VLAN 100下定义的模板1)
    [Device-GigabitEthernet3/0/1] port hybrid protocol-vlan vlan 200 1 // IPv6报文进入VLAN 200 (使用VLAN 200下定义的模板1)
    [Device-GigabitEthernet3/0/1] quit
    ```
    *   (GE3/0/2 连接 L2 Switch B 的配置类似 GE3/0/1)

---

**五、基于IP子网的VLAN (IP Subnet-based VLAN)**

**1. 概念**
基于IP子网的VLAN是根据untagged报文的源IP地址及子网掩码来确定其所属的VLAN。此特性通常通过ACL（访问控制列表）和QoS策略（服务质量）来实现。
*   设备从端口接收到untagged报文后，根据源IP地址匹配ACL规则，进而通过QoS策略将报文划分到指定VLAN。

**2. 组网需求与示例 (参考图1-9)**
*   **需求**：
    *   实验室内不同业务的电脑分配了不同网段的IP地址。
    *   需要根据不同的IP网段来划分VLAN。
    *   PC A (网段 1.1.1.0/24) 属于 VLAN 10。
    *   PC B (网段 2.1.1.0/24) 属于 VLAN 20。
*   **网络拓扑**：
    *   PC A, PC B 通过一台 L2 Switch 连接到核心交换机 Switch 的 GE3/0/1 端口。
*   **配置思路**：
    *   在核心交换机 Switch 上创建 VLAN 10 和 VLAN 20。
    *   配置 GE3/0/1 为 Hybrid 端口，PVID为1 (或其他默认VLAN)，并允许 VLAN 10, 20 的报文untagged通过。
    *   创建 ACL 规则，分别匹配源IP为 1.1.1.0/24 和 2.1.1.0/24 的报文。
    *   创建流分类器 (Traffic Classifier)，分别匹配上述 ACL 及对应网段的 ARP 报文。
    *   创建流行为 (Traffic Behavior)，分别将匹配的流量重标记 (remark) 到 VLAN 10 和 VLAN 20。
    *   创建QoS策略，绑定分类器和行为。
    *   将QoS策略应用于 GE3/0/1 端口的入方向。
*   **关键配置 (Switch)**：
    ```nohighlight
    [Switch] vlan 10
    [Switch] vlan 20
    [Switch] quit

    [Switch] interface GigabitEthernet 3/0/1
    [Switch-GigabitEthernet3/0/1] port link-type hybrid
    [Switch-GigabitEthernet3/0/1] port hybrid vlan 10 20 1 untagged // 允许VLAN 1, 10, 20的untagged报文
    [Switch-GigabitEthernet3/0/1] port hybrid pvid vlan 1         // 设置PVID为1
    [Switch-GigabitEthernet3/0/1] quit

    // ACL配置
    [Switch] acl number 3000 // 匹配 1.1.1.0/24 网段
    [Switch-acl-adv-3000] rule 0 permit ip source 1.1.1.0 0.0.0.255
    [Switch-acl-adv-3000] quit
    [Switch] acl number 3001 // 匹配 2.1.1.0/24 网段
    [Switch-acl-adv-3001] rule 0 permit ip source 2.1.1.0 0.0.0.255
    [Switch-acl-adv-3001] quit

    // 流分类器
    [Switch] traffic classifier c1 type or // 匹配1.1.1.0/24的IP报文
    [Switch-classifier-c1] if-match acl 3000
    [Switch-classifier-c1] quit
    [Switch] traffic classifier c2 type and // 匹配1.1.1.0/24的ARP报文
    [Switch-classifier-c2] if-match acl 3000
    [Switch-classifier-c2] if-match protocol arp
    [Switch-classifier-c2] quit
    // (c3, c4 类似，匹配ACL 3001)

    // 流行为
    [Switch] traffic behavior b1
    [Switch-behavior-b1] remark service-vlan-id 10 // 动作：打上VLAN 10的Tag
    [Switch-behavior-b1] quit
    [Switch] traffic behavior b2
    [Switch-behavior-b2] remark service-vlan-id 20 // 动作：打上VLAN 20的Tag
    [Switch-behavior-b2] quit

    // QoS策略
    [Switch] qos policy p1
    [Switch-qospolicy-p1] classifier c1 behavior b1
    [Switch-qospolicy-p1] classifier c2 behavior b1
    [Switch-qospolicy-p1] classifier c3 behavior b2 // (c3匹配ACL 3001的IP)
    [Switch-qospolicy-p1] classifier c4 behavior b2 // (c4匹配ACL 3001的ARP)
    [Switch-qospolicy-p1] quit

    // 应用策略到端口入方向
    [Switch] interface GigabitEthernet 3/0/1
    [Switch-GigabitEthernet3/0/1] qos apply policy p1 inbound
    [Switch-GigabitEthernet3/0/1] quit
    ```

---

**总结**

本讲义通过四个示例展示了不同VLAN划分技术的应用：
1.  **基于端口**：简单直接，适用于固定设备和VLAN分配。
2.  **基于MAC**：适用于需要根据设备物理地址进行VLAN分配，且设备可能移动的场景，通常需要Hybrid端口。
3.  **基于协议**：适用于需要根据流量的协议类型进行隔离的场景，通常需要Hybrid端口。
4.  **基于IP子网**：适用于根据IP地址段进行VLAN划分，配置较为复杂，通常涉及ACL和QoS，也需要Hybrid端口。

选择合适的VLAN划分方式取决于具体的网络需求、管理复杂度和设备支持能力。

---