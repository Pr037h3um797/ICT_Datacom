# Rapid Spanning Tree Protocol IEEE 802.1w (802.1D-2004)

## 快速生成树协议概述
RSTP可以视为STP的改进版本，RSTP在许多方面对STP进行了优化，它的收敛速度更快，而且能够兼容STP

在RSTP中，​​每一台交换机​​都会自主地、周期性地（基于Hello Time）从其指定端口发送BPDU。

这使得任何一台交换机都能在​​连续3个Hello Time（默认6秒）​​ 内未收到邻居的BPDU时，立即判定与该邻居的直连链路失效，从而快速触发收敛。

## RSTP 桥角色
和STP一致，我懒得写了。

## RSTP Port
### RSTP 四种端口角色
* 根接口 **Root**
    NRB通往根桥的最优路径<br>
    *最终状态 Forwarding*

* 指定接口 **Designated**
    每条活跃链路（冲突域）上，有且只有一个。根桥的所有端口都是指定端口<br>
    *最终状态 Forwarding*

* 预备/替代接口 **Alternate**
    此端口收到的BPDU等于或优于根端口保存的BPDU，而这个端口又不是根桥，则成为AP<br>
    从流量角度看：提供**指定桥到根**的另一条可切换路径，**作为RP的备份**<br>
    从BPDU角度看：AP就是收到来自另一台交换机发来的更优BPDU<br>
    作为"Blocked Port"的细化，是根端口的备份，它从其他桥接收更优的BPDU，从而知道另一条到达根桥的路径。<br>
    *最终状态：Discarding*

* 备份接口 **Backup**
    从BPDU角度看：接收到本交换机从另一个端口发送的BPDU（环路）<br>
    从流量角度看：提供连接到同一台下游设备（或者共享介质）的冗余链路，**作为本设备指定接口的备份**<br>
    RSTP新增的角色，是指定端口的备份。<br>
    *最终状态：Discarding*

    通过明确的Alternate和Backup角色，提供清晰的备用路径信息，这是快速收敛的基础。

* **边缘端口 Edge Port**
  
    这是RSTP/MSTP中的特殊端口角色，由管理员手动配置产生。华为 *edged-port*、Cisco *portfast*

    处于生成树边缘，不与桥相接的端口，一般连接终端设备，直接从丢弃状态进入转发状态，无需计时器等待。

    一般搭配BPDU保护功能，边缘端口设计上不应接收BPDU，一旦接收到BPDU，边缘端口将直接关闭

### RSTP 三种端口状态
RSTP将原来的5中状态缩减为3种
* 丢弃(Discarding)
    不转发流量，不学习MAC，合并了STP的禁用、阻塞和侦听


* 学习(Learning)
    不转发流量，学习MAC，


* 转发(Forwarding)
    转发流量，学习MAC，



## 计时器优化
RSTP的快速收敛能力，核心不在于修改计时器默认值，在于它引入一套**不依赖计时器等待的主动协商机制**，在绝大多数场景绕过了计时器的被动等待过程。

* **Hello Time** 2s

在802.1d中，是根桥发送BPDU的间隔，在802.1w中是所有桥主动发送BPDU的间隔，默认2s. 如果连续3个 Hello Time 未收到对端BPDU，认为邻居失效，触发重新计算。

* **Forward Delay** 15s

在802.1d中，用于控制端口在侦听和学习状态的停留时间，在802.1w中，大多数情况下RSTP端口可以绕过此计时器，作为P/A等机制失败时的回退收敛保障

**Max Age** 20s
在802.1d中，是BPDU在交换机内最大存活时间，在802.1w中，主要用于防范网络边缘因BPDU丢失，可能导致的环路。对于直连链路故障，基本不再依靠Max Age


## P/A（提议/同意 Proposal/Agreement）握手协议​​：

STP，拓扑变化后，端口需要等待30s（2倍 Forward Delay计时器）才能进入转发状态。

P/A机制是RSTP实现秒级收敛的核心机制。在*点对点全双工链路*上，当新的链路接通时，两端端口会通过交换特殊的BPDU进行握手协商。*在半双工链路上，退回到传统STP计时器模式*，需要等待两个 Forward Delay

P/A协商不成功，指定端口的选举需要等待两个 Forward Delay（侦听+学习）

* ***只有被选举为指定端口的端口才能发起P/A过程***

现有设备SW1，新设备SW2

0. 另一种情况：当一个原处于丢弃状态的端口被选举为新的指定端口，也会触发P/A过程。

1. 新设备接入现有网络，双方交换RST BPDU，SW1因其BPDU更优，选举为指定端口，SW2端口选举为根端口。SW1的指定端口，在角色确定后，立即向SW2发送 Proposal 置位 1 的RST BPDU，此时该SW1指定端口处于丢弃状态。

2. ​​SW2设备​​收到Proposal BPDU后，必须先​立即 *​同步​​* 其他端口，以确保无环，然后从其根端口回复一个​​Agreement​​置位的BPDU，进行统一。

    * 同步过程
      1. 立即阻塞所有非边缘端口：除了收到Proposal的根端口，SW2将其他所有非边缘端口强制设置为Discarding丢弃状态，相当于暂时隔离了下游分支。
      2. 边缘端口不受影响，保持转发状态。
      3. 所有非边缘端口都进入丢弃状态，同步完成，确保无环路风险。

3. SW2通过根端口，向SW1回复 Agreement 置位 1 的 RST BPDU，直接回应，非周期发送。


4. SW1 ​​收到Agreement后，其指定端口可​**​立即​**​进入Forwarding状态，这个过程完全绕过了Forward Delay计时器的等待，使得端口能在秒级内进入转发状态。

5. SW2被临时阻塞的非边缘指定端口，现在开始与它们的下游邻居重新进行独立的P/A握手过程，级联传递到网络边缘。

* 如果SW 1指定端口发送Proposal后，在超时内（似乎是BPDU超时，18s）未收到Agreement，该端口退回传统STP，依靠计时器，大约30s才能进入Forwarding状态。

Proposal Timeout似乎没有规定，利用的是BPDU超时，超时间隔 = Hello Time × 3 × Timer Factor，Hello Time默认2s

华为设备Timer Factor默认为3，BPDU超时时间18s

* 如果SW2运行STP，而非RSTP，在RSTP端口连续3 Hello Time收到STP 格式BPDU后，就会切换到STP工作模式，但从P/A失败并回退STP计时器模式，可能要等待BPDU超时18s

## BPDU

桥协议数据单元 Bridge Protocol Data Unit
在交换机之间交换、用于管理和维护生成树拓扑的信息。
类型：
### RST BPDU
这是在RSTP网络中使用的标准配置报文的正式名称，用于取代配置BPDU和 TCN BPDU

* TC=0 时，是普通，周期性的配置BPDU。
* TC=1 时，是TCN BPDU的代替。

**与STP不同，RST BPDU，由网络中的每个桥主动，周期(两个 Hello Time，共4s)生成发送**，其中包含本桥认为的根桥信息。


### RST BPDU的比较
RSTP与STP相同，按照以下顺序选择最优BPUD
1. 最小 RBID 根桥ID
2. 最小 RPC 根桥路径开销
3. 最小发送者BID
4. 最小发送者PID

### TC置位 BPDU

STP的TCN机制比较复杂，由NRB检测拓扑变化，然后从根端口发送TCN BDPU，上游桥接收到TCN，回复一个TCA（拓扑变更确认，再继续发送TCN，直到根桥收到TCN，根桥收到TCN后，全网发送TC置位的配置BPDU。

**RSTP取消了独立的TCN BPDU**，由 TC置位BPDU 代替

1. 当任何交换机（包括根桥）任何非边缘端口，进入 Forwarding 状态，或者从 Forwarding/Learning 状态变为 丢弃Discarding 状态，它就会**立即清除从其他非边缘端口上学习到的MAC地址**，这是为了防止临时环路。与STP加快老化不同，RSTP是立即清除。

2. 完成本地清理后，它会**向外泛洪** TC置位的 RST BPDU，泛洪时间是一个tcWhile计时器，默认为4s，两倍的Hello Time，在此期间周期发送TC BPDU

3. 接收到TC BPDU的桥，立即清除**除了接收到TC BPDU的端口学习的MAC**以外的，从非边缘端口学习到的MAC，并继续泛洪。

与STP上报再下发相比，RSTP变更通知更高效，收敛时间更快。

### RSTP 数据帧BPDU字段
**一个标准的RSTP (IEEE 802.1w) BPDU报文（35+1 Byte，一个空字节）包含以下字段，按顺序排列：**

*   **Protocol Identifier 协议标识符 (2字节)**
    *   **定义**：固定为 `0x0000`，标识此协议为生成树协议。（与STP相同）
*   **Protocol Version Identifier 协议版本标识符 (1字节)**
    *   **定义**：固定为 `0x02`，标识此BPDU为Rapid Spanning Tree Protocol版本。
*   **BPDU Type BPDU类型 (1字节)**
    *   **定义**：固定为 `0x02`，在RSTP中只有一种BPDU类型，由TC位来区分。
*   **Flags 标志 (1字节)**
    *   **定义**：标志位，RSTP使用了8个bit。位结构为：`[TCA][Agreement][Forwarding][Learning][Port Role][Port Role][Proposal][TC]` (LSB在左)。
        *   0. TCA ​- 在RSTP中此位已被弃用，为向后兼容保留，通常置0。
        *   1. Agreement​ - 同意标志。用于**P/A快速收敛机制中的“同意”消息。**
        *   2. Forwarding​ - 转发状态。置位表示端口处于转发状态。
        *   3. Learning​ - 学习状态。置位表示端口处于学习状态。
        *   4. Port Role​ - 端口角色。00=未知/未用，01=Alternate/Backup。
        *   5. Port Role​ - 端口角色。10=根端口，11=指定端口。
        *   6. Proposal​ - 提议标志。用于**P/A快速收敛机制中的“提议”消息。**
        *   7. TC (Topology Change)​ - 拓扑变更标志。置位时表示网络拓扑发生变化，收到者需刷新MAC表。

    * 几乎所有的有线网络协议，都约定俗成先发送字节的最低有效位。因此，按照物理捕获顺序，笔记中的位结构LSB在左，MSB在右
    * 最高有效位(most mignificant bit，msb)指的是一个n位二进制数字中的n-1位，具有最高的权值2^(n-1)。 有时也指Most Significant Byte(MSB),指多字节序列中具有最大权重的字节。
    * 最低有效位(least significant bit，lsb)和的是一个n位二进制数字中的0位，具有最低的权值2^0。有时也指Least Significant Byte(LSB),指多字节序列中具有最小权重的字节。[https://developer.aliyun.com/article/935525]

*   **Root Identifier 根ID (8字节)**
    *   **定义**：当前根桥的桥ID，由桥优先级和桥MAC地址组成。（与STP相同）
*   **Root Path Cost 根路径开销 (4字节)**
    *   **定义**：从发送此BPDU的交换机到根桥的累计路径开销
*   **Bridge Identifier 桥ID (8字节)**
    *   **定义**：发送此BPDU的交换机（指定桥）自身的桥ID。（与STP相同）
*   **Port Identifier 端口ID (2字节)**
    *   **定义**：发送此BPDU的端口ID，端口优先级+端口号。（与STP相同）
*   **Message Age 消息老化时间 (2字节)**
    *   **定义**：此BPDU自根桥发出以来经过的时间，每经过一个桥增加1。**单位：1/256秒**。由于RSTP主动发送机制，此计时器用于检测简介链路故障。
*   **Max Age 最大老化时间 (2字节)**
    *   **定义**：BPDU允许存活的最大消息老化时间，默认值为20秒（`5120`）。（与STP相同）
*   **Hello Time Hello时间 (2字节)**
    *   **定义**：每台交换机（不仅仅是根桥）发送配置BPDU的时间间隔，默认值为2秒（`512`）。（与STP相同）
*   **Forward Delay 转发延迟 (2字节)**
    *   **定义**：端口在侦听/学习状态(RSTP中为临时的阻塞状态)，各自需要停留的时间，默认值为15秒（`3840`）。在**P/A**机制生效的链路上，此计时器被绕过。
*   **Version 1 Length 版本1长度 (1字节)**
    *   **定义**：固定为0x00，为与 STP 的 BPDU 长度兼容而保留。
---

## 选举流程

* STP 桥角色选举，端口角色选举，几乎都是串行的、有计时器、高度中心化、逐步执行的
* RSTP 桥角色选举和端口角色选举，是并行、无计时器、事件驱动、主动同步的

### RSTP的RB选举

1. 初始化、端口状态、BPDU发送
   
   所有桥在一开始都认为自己是根桥，从所有指定端口主动发送BPDU，其中：
   * RBID：自己的BID
   * 根路径开销：0
   * 发送者BID：自己的BID
   * 标志位：端口角色 “指定端口”

2. BPDU比较
   桥接收到BPDU，进行比较BID，与STP相同，（比较根桥ID，根路径ID，发送方桥ID，发送端口ID。越小越优），并更新自己的根桥信息，如果别人的BPDU更优，它会在其BPDU计算中使用这个更新的根桥ID


### 根端口RP选举
1. 根端口是非根桥上唯一的端口，它是NRB去往RB的最优路径，对于一台NRB，它会缓存每个激活的端口的BPDU、接收到最优BPDU的端口作为根端口。
   * 最低根路径开销
   * 最低发送者BID
   * 最低发送者PID
   * 最低本机端口ID

### 指定端口DP选举

1. 根端口选举完成后，需要在 连接到同一个网段上的所有端口间，选举一个指定端口，负责向该网段转发发往根桥方向的流量。选举范围是同一条链路两端的两个端口。

2. 连接到同网段的各个桥，各自准备一个BPDU，包含：
   * RBID
   * 本桥到达根桥的根路径开销
   * 发送者BID
   * 发送者PID

3. 谁的BPDU更优，谁就是指定端口。

### 替代端口AP，备份端口BP

1. 当一个端口既不是根端口，也不是指定端口，那么它将成为预备端口。RSTP进一步确认它是替代端口还是备份端口

2. **替代端口** 如果端口收到的BPDU优于或等于本机从根端口发出的BPDU（如果NRP收到比RP更优的BPDU，但NRP不是RP，可能是这个更优BPDU是间接获取的，如果真的更优，会触发重新收敛），那么它将成为替代端口。是根端口的备份，作为到达RB的另一条可用路径。处于**丢弃状态**，能触发P/A过程，快速进入转发状态。

3. **备份端口** 如果该端口收到的BPDU是本机发出的BPDU(从另一个方向绕回来了)，那么它成为备份端口，是指定端口的备份。处于**丢弃状态**，防止本地连接错误导致的环路

## 故障检测恢复机制

### BPDU保活
* 周期发送：

  RSTP中，每个桥都主动地，周期性地，每两个Hello Time，从所有DP和RP发送BPDU。

* 快速老化

  RSTP中，一个端口在3个连续Hello Time未收到邻居指定桥的BPDU，它就会认为故障，6s的老化时间比STP中默认老化时间20s更快

* 端口角色快速切换

  AP作为RP的备份，平时处于丢弃状态，一旦RP失效，AP立即被提升为新的RP，并进入转发状态。

  当一个DP失效，某个BP需要接替角色时，会经历完整的P/A过程。但P/A过程也比STP收敛更快。

## 保护，故障预防

### BPDU保护

BPDU保护用于**边缘端口edged port**（连接终端设备的端口），正常情况下，边缘端口不应收到任何BPDU，一旦接受，表明可能有未授权的交换机接入，此机制会立即关闭端口，并根据厂商策略或管理员配置产生告警。


### **实现对比**
**华为实现：**
*   **核心命令**：全局启用：`stp bpdu-protection`（必须先配置边缘端口：`stp edged-port enable`）。
*   **依赖条件**：**必须与边缘端口（`stp edged-port enable`）配合使用**。边缘端口默认不参与STP计算，直接进入转发状态。
*   **触发后行为**：端口状态变为 **`Error-Down`**。恢复方式：
    1.  **手动**：在接口视图下执行 `restart` 或先`shutdown`再`undo shutdown`。
    2.  **自动**：全局配置 `error-down auto-recovery cause bpdu-protection interval <秒数>`。
*   **查看验证**：
    *   执行 `display stp`，查看输出中是否包含 **`BPDU-Protection: Enabled`**。
    *   执行 `display stp interface <接口>` 查看具体端口的保护状态。
*   **设计理念**：**全局开关，边缘端口联动**。一旦全局开启，所有配置为边缘端口的接口自动受到保护。

**思科实现：**
*   **核心命令**：
    1.  **接口级**：`spanning-tree bpduguard enable`
    2.  **全局级**：`spanning-tree portfast bpduguard default`
*   **依赖条件**：**必须与PortFast特性配合使用**。PortFast端口跳过Listening/Learning状态，直接转发。
*   **触发后行为**：端口状态变为 **`err-disabled`**。恢复方式：
    1.  **手动**：在接口视图下先执行 `shutdown`，再执行 `no shutdown`。
    2.  **自动**：全局配置 `errdisable recovery cause bpduguard` 和 `errdisable recovery interval <秒数>`。
*   **查看验证**：
    *   执行 `show spanning-tree summary` 查看摘要。
    *   执行 `show interfaces status err-disabled` 查看所有因错误而禁用的接口。
*   **设计理念**：**灵活配置，可与PortFast深度绑定**。支持全局默认启用，也允许接口单独配置或例外。

### 根保护

根保护用于**指定端口Designated**，保护合法Route Bridge的地位。配置在RB（或者NRB的不应该成为RP的DP）上。

当该端口收到**更优BPDU**时，端口进入 **`丢弃(Discarding)`**状态，阻塞报文转发。

当停止接收BPDU一段时间（Forward Delay * 2），端口恢复转发，不会触发全网收敛，并且不会产生TC置位BPDU，因为这不属于拓扑变更。

若持续收到，端口保持丢弃状态。

根保护行为是安静的，局部的，由于它的生效和恢复导致的Forwarding > Discarding，Discarding > Forwarding，都不会产生TC置位BPDU.

### **实现对比**

**华为实现：**
*   **核心命令**：在接口视图下启用：`stp root-protection`。
*   **生效条件**：**仅在端口角色为“指定端口”时生效**。一般配置在根桥或不应接收更优BPDU的指定端口上。
*   **触发后行为**：当端口收到更优的BPDU时，其状态变为 **`Discarding`**，停止转发数据。当持续一段时间（通常为两倍Forward Delay）未收到更优BPDU后，端口会自动恢复。
*   **互斥特性**：**不能与环路保护（`stp loop-protection`）在同一端口配置**。
*   **查看验证**：执行 `display stp brief`，在输出表格的 **`Protection`** 列中，受保护的端口会显示 **`ROOT`**。
*   **应用场景**：防止因下游设备配置错误或恶意攻击，发出更优BPDU，从而篡改网络中合法根桥的地位。

**思科实现：**
*   **核心命令**：在接口视图下启用：`spanning-tree guard root`。
*   **生效条件**：同样，需在指定端口上配置才能生效。
*   **触发后行为**：端口进入 **`root-inconsistent`** 状态（阻塞状态的一种）。
*   **互斥特性**：与 Loop Guard 等功能互斥。
*   **查看验证**：执行 `show spanning-tree interface <接口> detail` 查看端口的详细状态和保护信息。
*   **应用场景**：与华为相同，常用于接入层交换机的上行端口，以保护汇聚层或核心层指定的根桥。