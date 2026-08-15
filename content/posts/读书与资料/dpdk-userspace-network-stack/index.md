---
title: DPDK 用户态协议栈设计与实现
slug: dpdk-userspace-network-stack
description: 从 NIC、DMA、PMD 和 rte_mbuf 出发，逐层解析 Ethernet、IPv4、UDP 与 TCP，梳理 DPDK 环境、用户态 Socket 层与完整协议栈的实现边界
date: 2026-08-15T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - DPDK
  - 网络编程
  - Linux
  - TCP/IP
  - 用户态协议栈
categories:
  - 后端开发
---

> 学习目标：理解网卡、网卡驱动、TCP/IP 协议栈、POSIX Socket API 与应用程序之间的关系；搭建 DPDK 环境；使用 DPDK 收取以太网帧；逐步解析 Ethernet、IPv4、UDP、TCP；最终理解用户态协议栈需要补齐哪些能力。

---

## 一、先建立完整的网络数据路径

可以把网络程序拆成五个逻辑模块：

```text
1. 网卡 NIC
      ↕
2. 网卡驱动 Driver
      ↕
3. TCP/IP 协议栈
      ↕
4. POSIX Socket API：socket / bind / listen / accept / recv / send
      ↕
5. Application：Redis / Nginx / 自己编写的服务器
```

接收数据时，数据总体从左向右、从上向下流动；发送数据时方向相反。

不过，必须区分下面两条不同的数据路径。

### 1. Linux 内核协议栈路径

```text
物理网络
  → 网卡
  → Linux 内核网卡驱动
  → sk_buff
  → 内核 Ethernet/IP/TCP/UDP 协议栈
  → Socket 接收缓冲区
  → recv()/read()
  → 应用程序
```

应用程序使用的是 Linux 已经实现好的协议栈。Redis、Nginx 默认走的就是这条路径。

### 2. DPDK 用户态路径

```text
物理网络
  → 网卡
  → VFIO/UIO + DPDK PMD 用户态驱动
  → RX Descriptor Ring
  → rte_mbuf
  → 自己实现或引入的用户态协议栈
  → 自己实现的 Socket/POSIX 兼容层
  → 应用程序
```

网卡绑定给 DPDK 使用的驱动后，该端口的普通数据收发路径通常绕过 Linux 内核网卡驱动和内核 TCP/IP 协议栈。DPDK 负责高速收发原始数据包，但 **DPDK 本身不是 TCP/IP 协议栈**。

> 重要纠错：不能把 DPDK 主路径理解成“网卡 → DPDK → 内核驱动 → 内核 TCP/IP”。正常的 DPDK 用户态收包路径绕过内核协议栈。只有主动使用 KNI、TAP、virtio-user 等机制时，才会把选定的数据包重新送入内核。

---

## 二、网卡是什么

### 1. 网卡的基本职责

网卡（NIC，Network Interface Card）连接计算机和网络介质，主要完成：

- 接收和发送物理信号；
- 完成物理层编码、解码、时钟恢复等工作；
- 识别和生成以太网帧；
- 校验帧的 FCS；
- 按配置进行 MAC 地址过滤、VLAN 处理；
- 通过 DMA 在网卡和主机内存之间搬运数据；
- 一些网卡还能完成 checksum、TSO、LRO、RSS 等硬件卸载。

“网卡把模拟信号通过 AD/DA 转换为 0 和 1”可以帮助入门理解，但并不完全准确。现代以太网 PHY 会进行复杂的线路编码、调制、均衡和信号恢复，不能简单等同于普通 ADC/DAC。

### 2. 网卡属于哪一层

网卡不是完全独立于分层模型之外。通常可以这样理解：

- PHY 部分主要工作在物理层；
- MAC 部分主要工作在数据链路层；
- checksum、TSO、RSS 等卸载能力会辅助更高层处理。

因此，说网卡“主要横跨物理层和数据链路层”更准确。

### 3. DMA：网卡如何把数据交给内存

网卡不会要求 CPU 一个字节一个字节地读取数据。典型过程如下：

1. 驱动在内存中准备一组 RX 描述符和数据缓冲区；
2. 驱动把这些内存地址告诉网卡；
3. 网卡收到数据后，通过 DMA 直接把数据写入内存缓冲区；
4. 网卡更新 RX 描述符，标记该数据包已经完成；
5. CPU 再处理描述符所指向的数据。

这里的 RX 描述符通常按环形结构组织，所以常称为 RX Ring 或 Descriptor Ring。

---

## 三、网卡驱动做什么

网卡驱动是操作系统或 DPDK 与具体网卡硬件之间的适配层。它通常负责：

- 根据 PCI Vendor ID / Device ID 识别硬件；
- 初始化网卡寄存器；
- 配置 MAC、MTU、收发队列和硬件卸载；
- 分配 RX/TX Descriptor Ring；
- 建立 DMA 映射；
- 启停网卡；
- 处理链路状态和错误；
- 在中断模式或轮询模式下回收、提交数据包。

Linux 内核中有统一的网络设备框架，e1000、ixgbe、i40e、mlx5、vmxnet3 等具体驱动在这个框架下实现。DPDK 则提供 PMD（Poll Mode Driver），让用户态程序通过轮询直接与网卡队列交互。

### 1. 中断模式与轮询模式

传统内核路径通常结合中断和 NAPI 轮询：

- 流量较低时，网卡通过中断通知 CPU；
- 流量较高时，NAPI 暂时关闭频繁中断，改用批量轮询；
- 这样可以避免“中断风暴”。

DPDK 的典型方式是 Poll Mode：

```c
rte_eth_rx_burst(port_id, queue_id, mbufs, BURST_SIZE);
```

线程不断轮询指定的端口和队列，避免中断、调度和上下文切换开销。代价是即使没有数据，轮询线程也可能持续占用一个 CPU 核。

### 2. 多队列网卡

多队列网卡拥有多个 RX/TX 队列。其价值在于：

- 不同队列可以由不同 CPU 核或线程独立处理；
- 减少多个核心竞争同一个队列锁；
- 利用 RSS 把不同网络流分散到多个 RX 队列；
- 提升并行处理能力和总体吞吐量。

常见绑定关系是：

```text
RX Queue 0 → lcore 1
RX Queue 1 → lcore 2
RX Queue 2 → lcore 3
RX Queue 3 → lcore 4
```

RSS 通常根据源/目的 IP、源/目的端口、协议号计算哈希，将同一个流稳定地送到同一队列，避免同一 TCP 流的数据包在多个核心间乱序处理。

> 重要纠错：多队列能显著提高并行性，但“不是多队列网卡就一定无法绑定 DPDK”并不准确。单队列设备也可能被支持，只是扩展能力差。能否绑定和运行取决于设备、PMD、IOMMU、VFIO/UIO 以及虚拟化环境等条件。

### 3. 虚拟机中的 vmxnet3

VMware 中可以在 `.vmx` 文件中指定：

```text
ethernet0.virtualDev = "vmxnet3"
ethernet0.wakeOnPcktRcv = "TRUE"
```

`vmxnet3` 是 VMware 的半虚拟化高性能网卡，通常比模拟的 e1000 拥有更好的性能和多队列能力。修改配置前应关闭虚拟机，并确认实际使用的 `ethernet0`、`ethernet1` 与目标 MAC 地址相匹配。

---

## 四、sk_buff、rte_mbuf 与 Ring

### 1. Linux 的 sk_buff

Linux 内核用 `struct sk_buff` 描述网络包。它不是简单地把所有协议头复制一遍，而是保存：

- 数据缓冲区的位置；
- 当前数据起点和末尾；
- MAC、网络层和传输层头的位置；
- 数据包长度；
- 所属设备；
- checksum、VLAN、路由等元数据；
- 链表等组织信息。

协议层之间传递的通常是 `sk_buff` 指针。各层通过移动或记录头部位置来解析数据，而不是每经过一层就复制整个包。

### 2. 海量 sk_buff 如何组织

“每包一个 `sk_buff`，海量包如何组织”不能只用一种数据结构回答。不同阶段使用不同结构：

- 网卡收发描述符：环形队列 Ring；
- NAPI 待处理对象：调度列表；
- Socket 接收队列：链表/队列形式的 skb queue；
- TCP 乱序数据：红黑树等结构；
- TCP 重传数据：按发送顺序管理的队列/树；
- 内存对象分配：slab/slub 缓存。

所以，常用答案是“以 Ring 和各种队列为主，根据查找、排序、重传等需求再使用树结构”。

### 3. DPDK 的 rte_mbuf

`rte_mbuf` 与 `sk_buff` 的定位相似：都用来描述一个数据包及其元数据。DPDK 通常从预先创建的 mempool 中分配 mbuf，以减少频繁动态分配的开销。

```text
mempool
 ├── rte_mbuf + data buffer
 ├── rte_mbuf + data buffer
 ├── rte_mbuf + data buffer
 └── ...
```

网卡通过 DMA 把数据写入 mbuf 对应的数据缓冲区。`rte_eth_rx_burst()` 返回的是已经装有数据的 mbuf 指针数组，因此 CPU 不需要再把整个数据包从内核复制到用户态。

这通常称为零拷贝或少拷贝设计，但严格来说仍然可能发生 DMA、缓存行传输、跨 NUMA 访问以及应用层复制，不能把“零拷贝”理解成整个系统完全没有任何数据移动。

---

## 五、协议栈是什么

协议栈是一组按层协作的协议实现。接收一个 UDP/IPv4 以太网帧时，可依次解析：

```text
Ethernet Header
  → IPv4 Header
    → UDP Header
      → Application Payload
```

接收 TCP 时则是：

```text
Ethernet Header
  → IPv4 Header
    → TCP Header
      → TCP Segment Payload
```

解析协议头只是用户态协议栈的第一步。一个真正可用的协议栈还需要实现 ARP、路由、校验和、分片重组、TCP 状态机、重传、拥塞控制、Socket 缓冲区等功能。

---

## 六、以太网帧解析

不考虑 VLAN 标签时，Ethernet II 头部固定为 14 字节：

```text
0               5 6              11 12    13
+----------------+----------------+--------+
| 目的 MAC 6字节 | 源 MAC 6字节  | 类型 2 |
+----------------+----------------+--------+
```

常见 EtherType：

- `0x0800`：IPv4；
- `0x0806`：ARP；
- `0x86DD`：IPv6；
- `0x8100`：802.1Q VLAN。

DPDK 中对应：

```c
struct rte_ether_hdr *ethhdr =
    rte_pktmbuf_mtod(mbufs[i], struct rte_ether_hdr *);
```

`rte_pktmbuf_mtod()` 的作用是把 mbuf 中的数据起始地址转换成指定类型的指针。它并没有复制以太网头，只是在同一段内存上按 `struct rte_ether_hdr` 的布局解释字节。

判断是否为 IPv4：

```c
if (ethhdr->ether_type != rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
    continue;
}
```

网络协议字段通常采用大端字节序，所以要进行主机字节序与网络字节序转换。

### VLAN 注意事项

代码直接把以太网头后的数据当作 IPv4，只适用于没有 VLAN 标签的帧。若 EtherType 为 `0x8100`，还要解析 4 字节 VLAN Header，然后才能找到真正的上层 EtherType。

---

## 七、IPv4 解析

IPv4 基础头部最少 20 字节，结构包括：

```text
+---------+---------+-------------------+
|Version  |  IHL    | DSCP/ECN | Length |
+---------+---------+-------------------+
| Identification | Flags/Fragment Offset|
+---------------------------------------+
| TTL | Protocol | Header Checksum      |
+---------------------------------------+
| Source IPv4 Address                   |
+---------------------------------------+
| Destination IPv4 Address              |
+---------------------------------------+
| Options（可选，IHL > 5 时存在）       |
+---------------------------------------+
```

关键字段：

- Version：IPv4 时为 4；
- IHL：头部长度，以 4 字节为单位；最小值为 5，即 20 字节；
- Total Length：整个 IPv4 数据报长度；
- Fragment Offset / Flags：分片相关；
- TTL：每经过一个路由器通常减 1；
- Protocol：上层协议，UDP=17，TCP=6，ICMP=1；
- Header Checksum：只校验 IPv4 头；
- Source/Destination Address：源和目的 IPv4 地址。

原代码通过固定的 14 字节 Ethernet Header 偏移找到 IPv4 头：

```c
struct rte_ipv4_hdr *iphdr = rte_pktmbuf_mtod_offset(
    mbufs[i], struct rte_ipv4_hdr *, sizeof(struct rte_ether_hdr));
```

判断 UDP：

```c
if (iphdr->next_proto_id == IPPROTO_UDP) {
```

### IPv4 解析必须检查的内容

健壮的实现至少应检查：

1. mbuf 中是否有完整的 Ethernet 和 IPv4 基础头；
2. IPv4 Version 是否为 4；
3. IHL 是否至少为 5；
4. 包长度是否覆盖 `IHL * 4`；
5. Total Length 是否合理；
6. IPv4 Header Checksum 是否正确，或确认硬件已校验；
7. 是否为分片；
8. 再根据 Protocol 分发给 ICMP、UDP 或 TCP。

> 原代码使用 `(iphdr + 1)` 寻找 UDP 头，相当于默认 IPv4 头固定为 20 字节。若 IPv4 带 Options，IHL 会大于 5，UDP 头的真实起点应为 `IP Header 起点 + IHL * 4`。

---

## 八、UDP 解析

UDP 头固定为 8 字节：

```text
+-------------------+-------------------+
| Source Port       | Destination Port  |
+-------------------+-------------------+
| UDP Length        | UDP Checksum      |
+-------------------+-------------------+
| Payload ...                           |
+---------------------------------------+
```

关键字段：

- Source Port：源端口；
- Destination Port：目的端口；
- Length：UDP 头加 Payload 的总长度，最小为 8；
- Checksum：覆盖伪首部、UDP 头和 UDP 数据。

原代码：

```c
struct rte_udp_hdr *udphdr = (struct rte_udp_hdr *)(iphdr + 1);
printf("udp:%s\n ", (char *)(udphdr + 1));
```

第一行把 IPv4 基础头后的地址解释为 UDP Header；第二行把 UDP Header 后的数据解释为 C 字符串并打印。

### 为什么 `%s` 只适合当前演示

网络 Payload 不保证以 `\0` 结尾，而 `%s` 会持续读取，直到遇到 `\0`。因此实际代码必须：

- 根据 UDP Length 计算 Payload 长度；
- 确认长度没有超过 mbuf/IPv4 数据长度；
- 使用带长度的输出方式，例如 `%.*s`；
- 对二进制协议应使用十六进制打印，而不是字符串打印。

当前 Packet Sender 发送的是短 ASCII 文本，并且内存后面恰好出现零字节，所以演示中可能正常输出，但这不是安全的通用处理方法。

---

## 九、TCP 解析

TCP 头最少为 20 字节，可带 Options，最长可达 60 字节。

```text
+-------------------+-------------------+
| Source Port       | Destination Port  |
+---------------------------------------+
| Sequence Number                       |
+---------------------------------------+
| Acknowledgment Number                 |
+------+----------+---------------------+
|Offset| Flags    | Window              |
+-----------------+---------------------+
| Checksum        | Urgent Pointer      |
+---------------------------------------+
| Options（可选）                      |
+---------------------------------------+
| Payload ...                           |
+---------------------------------------+
```

关键字段：

- Sequence Number：本段第一个字节的序号；
- Acknowledgment Number：期望收到的下一个序号；
- Data Offset：TCP Header 长度，以 4 字节为单位；
- Flags：SYN、ACK、FIN、RST、PSH、URG 等；
- Window：接收窗口，参与流量控制；
- Checksum：覆盖 IPv4/IPv6 伪首部、TCP 头和 TCP 数据。

只把 TCP Header 拆出来还不能叫“实现 TCP”。至少还需要：

- 四元组查找连接：源 IP、源端口、目的 IP、目的端口；
- TCP 状态机；
- 三次握手和四次挥手；
- 序列号和 ACK；
- 乱序重组；
- 超时重传；
- RTT/RTO 估计；
- 滑动窗口和流量控制；
- 拥塞控制；
- TIME_WAIT 等定时器管理。

### TCP 状态机简图

```text
服务器：CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED
客户端：CLOSED → SYN_SENT → ESTABLISHED

主动关闭的一方：ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
被动关闭的一方：ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

---

## 十、POSIX Socket API 与用户态协议栈

POSIX API 是应用程序看到的接口：

- `socket()`：创建 Socket；
- `bind()`：绑定本地 IP 和端口；
- `listen()`：把 TCP Socket 设为监听状态；
- `accept()`：取出一个已经建立的 TCP 连接；
- `connect()`：主动发起 TCP 连接；
- `recv()/read()`：读取接收数据；
- `send()/write()`：发送数据；
- `close()`：关闭 Socket。

### 1. 内核中的关系

在 Linux 内核协议栈中，文件描述符 `fd` 会关联到内核 Socket 对象。TCP 连接还对应一个传输控制块，可广义称为 TCB，其中保存：

- 四元组；
- TCP 状态；
- 序列号；
- 收发窗口；
- 重传队列；
- 接收队列；
- 定时器等。

`recv(fd, ...)` 不是简单地“直接读取 TCB 某个位置”，而是通过 fd 找到 Socket，再从该 Socket 的接收缓冲区读取已经按序整理好的字节。如果没有数据，调用可能阻塞，或者在非阻塞模式下返回 `EAGAIN`。

### 2. 用户态协议栈怎样提供 POSIX API

如果应用完全绕过内核协议栈，Linux 原生 `recv()` 不会自动读取 DPDK mbuf。需要增加适配层，常见方法包括：

- 修改应用，让它调用用户态协议栈自己的 API；
- 使用库替换或 LD_PRELOAD 截获 Socket 调用；
- 提供兼容 POSIX 的用户态 Socket 库；
- 通过事件通知机制实现类似 epoll 的接口。

用户态需要自己维护：

```text
fd/table
  → socket object
    → UDP endpoint 或 TCP TCB
      → receive queue / send queue
```

对于 UDP，`recvfrom()` 通常从对应端口的报文队列取出一个 Datagram；对于 TCP，`recv()` 从已经重排并确认连续的字节流中读取数据。

---

## 十一、DPDK 是什么，以及为什么快

DPDK（Data Plane Development Kit）是一组用于高速数据包处理的库、驱动和工具。它的核心思想包括：

- 用户态 PMD 轮询网卡；
- 批量收发 Burst；
- 预分配 mempool/mbuf；
- Hugepage；
- CPU 亲和性和独占核心；
- NUMA 感知；
- 无锁 Ring；
- 硬件多队列与 RSS；
- Prefetch 和缓存友好布局；
- 减少中断、系统调用、上下文切换和数据复制。

### DPDK 优化的核心指标

DPDK 不只是“处理大包、提高吞吐量”。更准确地说，它常用于提高：

- 每秒处理的数据包数 PPS，尤其是大量小包；
- 总吞吐量；
- 延迟的可预测性和尾延迟；
- 多核心扩展能力。

是否能提升 Redis/Nginx 性能，取决于瓶颈：

- 如果瓶颈在业务逻辑、锁、存储或单线程执行，只有 DPDK 不会神奇地提高 QPS；
- 如果瓶颈在内核网络路径、软中断、系统调用或数据复制，用户态网络方案可能改善 QPS 或延迟；
- 把现有 Redis/Nginx 接到 DPDK 上需要完整协议栈和兼容层，并非简单绑定网卡即可。

### 常见使用场景

- 防火墙、ACL、DPI；
- 路由器、交换机、网关；
- 负载均衡器；
- 5G/电信数据面；
- 虚拟交换机；
- 流媒体或报文转发；
- 流量采集与监控；
- 高性能存储网络的数据面。

数据备份和 RDMA 可能使用类似的大页、DMA、零拷贝、用户态驱动思想，但 DPDK 与 RDMA 是不同的技术体系，不能简单说“底层一定用 DPDK，应用层用 RDMA”。

可研究的用户态协议栈/数据面项目包括 mTCP、lwIP、F-Stack、VPP、Seastar networking、基于 BSD 协议栈移植的实现等。项目是否仍活跃、DPDK 版本是否兼容，需要按实际版本确认。

---

## 十二、VFIO、UIO、PCI、KNI 与 NUMA

### 1. PCI 地址

网卡通常是 PCIe 设备。可以使用：

```bash
lspci
lspci -nn
lspci -k
```

PCI 地址形如：

```text
0000:1a:00.0
```

含义是：

```text
domain:bus:device.function
```

### 2. VFIO 与 UIO

它们帮助用户态程序访问 PCI 设备资源：

- UIO：较简单的通用用户态 I/O 框架；
- VFIO：通常配合 IOMMU，隔离性和安全性更好，现代环境一般优先使用；
- DPDK PMD：真正理解并操作具体网卡寄存器和队列的用户态驱动。

不能简单理解为 VFIO/UIO“截获 PCI 地址”；更准确地说，它们把设备资源安全地映射或暴露给用户态驱动，并管理中断、DMA/IOMMU 等能力。

### 3. KNI

KNI 曾用于在 DPDK 应用与 Linux 内核网络栈之间交换数据包。例如，把控制面流量送回内核处理。但它不是 DPDK 收包必须经过的组件，而且在较新的 DPDK 方案中通常会考虑 TAP、virtio-user 等替代路径。

### 4. NUMA

NUMA 是 Non-Uniform Memory Access，即非一致内存访问。多路 CPU 服务器中，每个 CPU Socket 通常拥有本地内存：

```text
CPU Socket 0 ↔ Local Memory 0
CPU Socket 1 ↔ Local Memory 1
```

CPU 访问本地内存比跨 Socket 访问远端内存更快。因此应尽量做到：

- 网卡所在 NUMA Node；
- 轮询该网卡的 CPU 核；
- mbuf mempool 所在内存；

三者位于同一个 NUMA Node。

代码中的 `rte_eth_dev_socket_id()` 用于查询设备所在 NUMA Socket，`rte_socket_id()` 返回当前 lcore 所在 Socket。二者不一定总是相同。

---

## 十三、Hugepage 大页

普通页通常为 4 KiB。Hugepage 常见规格为：

- 2 MiB；
- 1 GiB。

DPDK 使用大页主要是为了：

- 减少页表项数量；
- 降低 TLB Miss；
- 便于大块、稳定、可用于 DMA 的内存管理；
- 减少运行时内存分配的不确定性。

### 1. 示例 GRUB 配置

```text
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=2M hugepagesz=2M hugepages=1024"
```

这表示默认大页为 2 MiB，共预留 1024 页，即约 2 GiB 内存。虚拟机内存有限时，需要根据虚拟机总内存调整，不能机械照抄。

通常修改后还需要更新 GRUB 并重启，例如 Ubuntu/Debian：

```bash
sudo update-grub
sudo reboot
```

重启后验证：

```bash
grep -i huge /proc/meminfo
```

具体命令因发行版和 DPDK 版本而异。现代 DPDK 可能自动处理 hugetlbfs，也可以手动挂载。1 GiB 大页还依赖 CPU 支持，并通常需要在启动参数中预留。

### 2. Hugepage 不直接提升网卡物理转换速度

Hugepage 优化的是主机内存管理、TLB 和 DMA 缓冲区访问，并不会让网卡的光电信号转换本身更快。它提升的是数据包在 CPU/内存数据面中的处理效率和稳定性。

---

## 十四、VMware 三种网络模式

### 1. Bridged（桥接）

虚拟机像局域网中的独立主机：

```text
物理局域网
 ├── Windows/macOS 宿主机：172.26.185.5
 └── 虚拟机：172.26.185.250
```

虚拟机通常与宿主机处于同一二层网络，可从局域网 DHCP 获取地址，或配置同网段静态地址。桥接绑定到某个宿主机物理接口时，该接口断开可能影响虚拟机网络。

“复制物理网络连接状态”之类的选项，通常让虚拟网卡随宿主机物理接口的连接/断开状态变化，适合笔记本在 Wi-Fi、有线网络之间切换的场景。不同 VMware 产品中的名称和细节会略有不同。

### 2. NAT

VMware 在宿主机上创建虚拟子网，虚拟机通过宿主机做 NAT 访问外网：

```text
虚拟机 → VMware 虚拟网关/NAT → 宿主机 → 外网
```

外部设备通常不能直接主动访问虚拟机，除非做端口映射或额外配置。

### 3. Host-only（仅主机）

虚拟机主要与宿主机及同一 Host-only 网络中的虚拟机通信，默认不能直接访问外网。它适合隔离实验环境。

> “Host-only 一定可以 ping 通公网”是不对的；默认情况下它不能上网，除非宿主机另行开启路由/NAT。

---

## 十五、macOS Packet Sender 测试链路

已知 DPDK 端口信息：

```text
MAC：00:0c:29:aa:5b:f0
PCI：0000:1a:00.0
PMD：net_vmxnet3
Link：up
```

宿主机：

```text
IP：172.26.185.5
Mask：255.255.0.0
Interface：en0
```

ustack 测试地址：

```text
172.26.185.250
```

### 1. 为什么需要静态 ARP

应用把 UDP 发往 `172.26.185.250:8888` 时，操作系统需要构造以太网帧。因为目标 IP 在同一子网，macOS 必须知道目标 IP 对应的 MAC 地址。

正常情况是：

```text
macOS 广播 ARP Request：谁是 172.26.185.250？
ustack 单播 ARP Reply：172.26.185.250 是 00:0c:29:aa:5b:f0
```

当前 ustack 没有实现 ARP Reply，所以手工建立：

```bash
sudo arp -s 172.26.185.250 00:0c:29:aa:5b:f0
sudo arp -s 172.26.185.250 00:0c:29:aa:5b:f0 ifscope en0
```

验证：

```bash
arp -n 172.26.185.250
```

### 2. Packet Sender 配置

```text
Address：172.26.185.250
Port：8888
Protocol：UDP
ASCII：hello dpdk
```

完整路径：

```text
Packet Sender
  → macOS UDP/IP 处理
  → 静态 ARP 表查到目标 MAC
  → macOS en0
  → VMware Fusion 桥接
  → vmxnet3 虚拟网卡
  → DPDK port 0 / RX queue 0
  → rte_eth_rx_burst()
  → Ethernet / IPv4 / UDP 解析
  → 打印 hello dpdk
```

看到：

```text
udp:hello dpdk
```

说明至少以下环节已经连通：

- macOS 到 Fusion 桥接网络；
- 目标 MAC 投递；
- vmxnet3 链路；
- DPDK 端口初始化；
- RX Queue 0 收包；
- 当前测试报文的 Ethernet/IPv4/UDP 基础解析。

但它还不能单独证明 checksum、分片、VLAN、IP Options、异常长度处理等已经正确实现。

---

## 十六、DPDK 环境搭建思路

老版本 DPDK 常使用 `dpdk-setup.sh`、`RTE_SDK` 和 `RTE_TARGET`。典型含义是：

```bash
export RTE_SDK=/home/.../dpdk
export RTE_TARGET=...
```

- `RTE_SDK`：DPDK 源码/SDK 根目录；
- `RTE_TARGET`：旧版构建系统生成的目标目录名称。

`native` 一般表示按当前机器架构优化；`linuxapp` 是旧版目标名称的一部分；具体选项需要以所使用的旧版 DPDK 文档和脚本为准。

现代 DPDK 已主要使用 Meson + Ninja 构建，不再依赖旧式 `make config`、`RTE_TARGET` 流程：

```bash
meson setup build
ninja -C build
```

因此必须先确认教程使用的 DPDK 版本。不要把老教程里的菜单选项直接套到新版本。

### 通用搭建检查表

1. 确认 CPU 架构、Linux 发行版和 DPDK 版本；
2. 确认 vmxnet3/物理网卡被当前 DPDK PMD 支持；
3. 配置 Hugepage；
4. 若使用 VFIO，确认 IOMMU/VFIO 条件；
5. 查看 PCI 地址和当前驱动；
6. 将实验网卡绑定到合适驱动；
7. 保留管理网卡，避免把 SSH 所依赖的唯一网卡绑定走；
8. 用 testpmd 检查端口、队列、Link 和收包计数；
9. 再运行自己的程序。

> 安全提示：绑定网卡前一定确认 PCI 地址。将正在承载 SSH 或默认路由的网卡绑定给 DPDK，会立即让普通 Linux 网络连接中断。

---

## 十七、原始代码（保持不变）

下面代码完全按原内容保留，没有修改：

```c
#include<stdio.h>
#include<rte_eal.h>
#include<rte_ethdev.h>
#include<arpa/inet.h>

int global_portid=0;//网卡绑定id从0开始 发数据和收数据知道id就好
#define NUM_MBUFS 4096
#define BURST_SIZE 128
static const struct rte_eth_conf port_conf_default={
    .rxmode={.max_rx_pkt_len=RTE_ETHER_MAX_LEN}
};
static int  ustack_init_port(struct rte_mempool *mbuf_pool){

    uint16_t nb_sys_ports=rte_eth_dev_count_avail();
    if(nb_sys_ports==0){
        rte_exit(EXIT_FAILURE,"No Supported eth found/n");
    } 
    //struct rte_eth_dev_info dev_info;
    //rte_eth_dev_info_get(global_portid，&dev_info);
    //绑定的网卡
    //现在有八个 队列 弄1 2 4 8 
    const int num_rx_queues =1;
    const int num_tx_queues =0;
    rte_eth_dev_configure(global_portid,num_rx_queues,num_tx_queues,&port_conf_default);
  
    if(rte_eth_rx_queue_setup(global_portid,0,128,rte_eth_dev_socket_id(global_portid),NULL,mbuf_pool)<0){
        //在0号网卡的0号队列开辟了RX队列 准备了128个mbuffer
    rte_exit(EXIT_FAILURE," Could not setup RX queue/n");       
    }

    if (rte_eth_dev_start(global_portid)<0){//启动
    rte_exit(EXIT_FAILURE,"Could not start");    
    }
    return 0;
}

int main(int argc,char *argv[]){

    if(rte_eal_init(argc,argv)<0){//初始化 检查网卡信息 配置 是否绑定
        rte_exit(EXIT_FAILURE,"Error with EAL init\n");
    }
    struct rte_mempool *mbuf_pool= rte_pktmbuf_pool_create("mbuf pool",NUM_MBUFS,0,0,RTE_MBUF_DEFAULT_BUF_SIZE,rte_socket_id());
    //接收数据放在mbuff
    //mbuffer分配在内存上的 rte_socket_id 就是当前使用哪块内存卡槽ID分配的
    if(mbuf_pool==NULL){
        rte_exit(EXIT_FAILURE,"Could not create mbuf pool\n");
    }

    ustack_init_port(mbuf_pool);

    //接收数据
    while(1){
        struct rte_mbuf *mbufs[BURST_SIZE]={0};
        uint16_t num_recvd= rte_eth_rx_burst(global_portid,0,mbufs,BURST_SIZE);
        if (num_recvd>BURST_SIZE){
            rte_exit(EXIT_FAILURE,"Error receiving from eth\n");
        }

        int i=0;
        for(i=0;i<num_recvd;i++){
            struct rte_ether_hdr *ethhdr=rte_pktmbuf_mtod(mbufs[i],struct rte_ether_hdr *);
            if(ethhdr->ether_type != rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)){
                continue;
            }
            struct rte_ipv4_hdr *iphdr =rte_pktmbuf_mtod_offset(mbufs[i],struct rte_ipv4_hdr *,sizeof(struct rte_ether_hdr));
            if (iphdr->next_proto_id==IPPROTO_UDP){//两个以上都转
                struct rte_udp_hdr *udphdr =(struct rte_udp_hdr *)(iphdr+1);
                printf("udp:%s\n ",(char*)(udphdr+1));
            }
        }

    }
    printf("Hello dpdk\n");


}
```

---

## 十八、代码逐行对照理解

### 1. 头文件

```c
#include<stdio.h>
```

引入标准输入输出声明，本程序用它调用 `printf()`。

```c
#include<rte_eal.h>
```

引入 DPDK EAL 接口。EAL 是 Environment Abstraction Layer，负责 Hugepage、lcore、内存、PCI 设备等基础环境初始化，也声明了 `rte_eal_init()`、`rte_exit()` 等接口。

```c
#include<rte_ethdev.h>
```

引入以太网设备抽象接口，包括端口配置、RX Queue 设置、设备启动和 Burst 收包。

```c
#include<arpa/inet.h>
```

提供网络字节序、IP 地址转换等常用声明。当前代码中没有直接调用 `htons()`、`ntohs()` 等函数，但 `IPPROTO_UDP` 等定义可能通过相关系统头可见。实际项目通常还会显式包含 DPDK Ethernet/IP/UDP 头文件，具体取决于 DPDK 版本的包含关系。

### 2. 全局端口与常量

```c
int global_portid=0;
```

选择 DPDK 逻辑端口 0。这里的 port id 是 DPDK 枚举后的端口编号，不等于 PCI 地址，也不一定等于 Linux 的 `eth0`。

```c
#define NUM_MBUFS 4096
```

准备在 mempool 中创建 4096 个 mbuf。它们不仅供 128 个 RX 描述符使用，还要留给收包后尚未释放、缓存以及其他内部需求。

```c
#define BURST_SIZE 128
```

每次调用 `rte_eth_rx_burst()`，最多希望取回 128 个包。实际返回值可以是 0 到 128。

### 3. 默认端口配置

```c
static const struct rte_eth_conf port_conf_default={
    .rxmode={.max_rx_pkt_len=RTE_ETHER_MAX_LEN}
};
```

创建端口配置结构，把最大接收包长设为标准以太网最大长度对应的 DPDK 常量。

注意：DPDK 的结构体字段会随版本变化。某些新版本中 `max_rx_pkt_len` 的位置或用法已变化。如果发生编译错误，应首先对照当前版本 API，而不是认为网络原理有问题。

### 4. 端口初始化函数

```c
static int ustack_init_port(struct rte_mempool *mbuf_pool){
```

定义只在本源文件使用的函数，参数是已经创建好的 mbuf 内存池。

```c
uint16_t nb_sys_ports=rte_eth_dev_count_avail();
```

查询当前 EAL 环境中可用的 DPDK Ethernet 设备数量。`nb` 是 number 的常用缩写。它不是固定表示“绑定网口数量”，更准确地说是当前可用的 ethdev 数量。

```c
if(nb_sys_ports==0){
    rte_exit(EXIT_FAILURE,"No Supported eth found/n");
}
```

如果没有设备，打印错误并退出。字符串中的 `/n` 是普通字符，不是换行符；真正的换行符写法是 `\n`。本笔记遵守要求，不修改原代码，只指出区别。

```c
//struct rte_eth_dev_info dev_info;
//rte_eth_dev_info_get(global_portid，&dev_info);
```

这两行被注释，不会执行。设计意图是读取端口能力，比如最大队列数、RSS、offload 能力。第二行参数之间使用了中文逗号 `，`，若取消注释需要注意它不能作为 C 语言逗号使用。

```c
const int num_rx_queues =1;
const int num_tx_queues =0;
```

配置 1 个接收队列、0 个发送队列。此程序只收不发，所以 TX Queue 设为 0。若以后要回复 ARP、UDP 或 TCP，就必须配置 TX Queue。

```c
rte_eth_dev_configure(global_portid,num_rx_queues,num_tx_queues,&port_conf_default);
```

配置端口的 RX/TX 队列数量和总体参数。函数有返回值，当前代码没有检查；健壮程序应检查是否小于 0。

```c
if(rte_eth_rx_queue_setup(global_portid,0,128,
    rte_eth_dev_socket_id(global_portid),NULL,mbuf_pool)<0){
```

配置 RX Queue：

- `global_portid`：端口 0；
- `0`：RX Queue 0；
- `128`：描述符数量；
- `rte_eth_dev_socket_id(...)`：在网卡所在 NUMA Socket 分配队列资源；
- `NULL`：使用默认 RX Queue 配置；
- `mbuf_pool`：网卡收到的数据放入这个池提供的 mbuf。

注释“准备了 128 个 mbuffer”是便于理解的近似说法。准确说是请求建立 128 个 RX descriptors；描述符与 mbuf 的补充、占用和回收由 PMD 管理，mempool 总大小仍为 4096。

```c
rte_exit(EXIT_FAILURE," Could not setup RX queue/n");
```

配置失败就退出。同样，`/n` 不会产生换行。

```c
if (rte_eth_dev_start(global_portid)<0){
    rte_exit(EXIT_FAILURE,"Could not start");
}
```

启动端口。配置端口和队列只是准备工作，start 后网卡才进入正式收发状态。

```c
return 0;
```

表示初始化成功。

### 5. main 与 EAL 初始化

```c
int main(int argc,char *argv[]){
```

程序入口。DPDK 会先解析命令行中的 EAL 参数，应用自己的参数通常放在 `--` 后面并根据 `rte_eal_init()` 的返回值调整。

```c
if(rte_eal_init(argc,argv)<0){
    rte_exit(EXIT_FAILURE,"Error with EAL init\n");
}
```

初始化 DPDK 运行环境，涉及：

- 解析 EAL 参数；
- 发现 CPU lcore；
- 初始化 Hugepage 内存；
- 初始化内存管理；
- 探测并初始化设备和总线；
- 建立主从 lcore 等运行环境。

它不等于“自动保证所有网卡已经正确绑定”。绑定通常要在运行程序之前完成，EAL 负责发现当前可使用的设备。

### 6. 创建 mbuf pool

```c
struct rte_mempool *mbuf_pool=
    rte_pktmbuf_pool_create("mbuf pool",NUM_MBUFS,0,0,
                            RTE_MBUF_DEFAULT_BUF_SIZE,rte_socket_id());
```

参数逐个理解：

- `"mbuf pool"`：内存池名称；
- `NUM_MBUFS`：4096 个 mbuf；
- 第一个 `0`：每个 lcore 的 cache 数量为 0；
- 第二个 `0`：每个对象额外私有区域大小为 0；
- `RTE_MBUF_DEFAULT_BUF_SIZE`：每个 mbuf 默认数据缓冲区大小；
- `rte_socket_id()`：在当前运行核心所在 NUMA Socket 分配。

这里 `rte_socket_id()` 指 NUMA 节点编号，不是物理“内存卡槽 ID”。服务器硬件的 DIMM 插槽和 NUMA Node 不是同一个概念。

```c
if(mbuf_pool==NULL){
    rte_exit(EXIT_FAILURE,"Could not create mbuf pool\n");
}
```

创建失败立即退出。可能原因包括 Hugepage 不足、名称冲突、socket 内存不够或参数不合法。

```c
ustack_init_port(mbuf_pool);
```

用刚创建的内存池配置并启动端口。

### 7. 轮询收包

```c
while(1){
```

进入无限循环。这是 PMD 轮询模型的核心，线程会持续询问 RX Queue 是否有包。

```c
struct rte_mbuf *mbufs[BURST_SIZE]={0};
```

创建一个长度为 128 的指针数组。数组保存 `rte_mbuf *`，并不在栈上创建 128 个完整 mbuf。

```c
uint16_t num_recvd=
    rte_eth_rx_burst(global_portid,0,mbufs,BURST_SIZE);
```

从端口 0、队列 0 最多取 128 个已经收到的包：

- 返回 0：当前没有包；
- 返回 1～128：实际取到的包数；
- mbuf 指针写入 `mbufs[0 ... num_recvd-1]`。

`rte_eth_rx_burst()` 通常通过检查 RX descriptors 批量完成工作，不是每次都触发系统调用。

```c
if (num_recvd>BURST_SIZE){
    rte_exit(EXIT_FAILURE,"Error receiving from eth\n");
}
```

按 API 契约，返回值不会大于传入的 `BURST_SIZE`，所以这个判断正常情况下不会成立。它也不能检测一般收包错误，因为该 API 的返回值语义主要是“本次收到多少包”。

### 8. 遍历每一个包

```c
int i=0;
for(i=0;i<num_recvd;i++){
```

依次处理本轮收到的所有 mbuf。

```c
struct rte_ether_hdr *ethhdr=
    rte_pktmbuf_mtod(mbufs[i],struct rte_ether_hdr *);
```

取得包数据起点，把它解释成 Ethernet Header。

```c
if(ethhdr->ether_type != rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)){
    continue;
}
```

如果 EtherType 不是 IPv4，就跳过后续解析。ARP、IPv6、VLAN 等都会走 `continue`。

这里有一个重要的生命周期问题：跳过不等于释放。当前程序没有调用 `rte_pktmbuf_free(mbufs[i])`，因此收到的 mbuf 不会归还 mempool。运行一段时间后，RX Queue 可能因为拿不到空闲 mbuf 而停止继续收包。

```c
struct rte_ipv4_hdr *iphdr =rte_pktmbuf_mtod_offset(
    mbufs[i],struct rte_ipv4_hdr *,sizeof(struct rte_ether_hdr));
```

从数据起点向后移动一个 Ethernet Header 的长度，即 14 字节，再把地址解释成 IPv4 Header。

当前代码在解引用前没有检查包的实际长度；遇到截断帧或异常帧时可能越界读取。

```c
if (iphdr->next_proto_id==IPPROTO_UDP){
```

检查 IPv4 Protocol 字段是否等于 UDP 的协议号 17。

```c
struct rte_udp_hdr *udphdr =(struct rte_udp_hdr *)(iphdr+1);
```

`iphdr + 1` 会跳过一个 `struct rte_ipv4_hdr`，也就是按 20 字节基础 IPv4 头定位 UDP Header。它没有考虑 IHL 大于 5的 IPv4 Options。

```c
printf("udp:%s\n ",(char*)(udphdr+1));
```

`udphdr + 1` 跳过 8 字节 UDP Header，将后续 Payload 当作字符串打印。它没有依据 UDP Length 限制读取长度，也没有保证 Payload 以 `\0` 结尾，因此只适合受控的入门测试。

### 9. 循环之后的代码

```c
printf("Hello dpdk\n");
```

由于前面是无限循环，并且没有 `break`，正常情况下这一行不可达。

---

## 十九、今天内容的核心总结

1. 网卡把网络介质上的信号变成帧，并通过 DMA 把数据放入主机内存；它主要涉及物理层和数据链路层。
2. 驱动负责初始化网卡、配置队列和 DMA，并让软件能够操作具体硬件。
3. Linux 使用 `sk_buff` 描述数据包；DPDK 使用 `rte_mbuf`，两者都不只是数据本身，还包含元数据。
4. 多队列让多个 CPU 核并行处理流量，RSS 负责把流分配到不同队列；多队列很重要，但不是所有 DPDK 绑定的绝对前提。
5. DPDK 用用户态轮询、Burst、Hugepage、mempool、多队列和 NUMA 优化高速包处理。
6. 网卡绑定给 DPDK 后，常规数据路径绕过内核 TCP/IP；因此要么引入现有用户态协议栈，要么自己实现。
7. Ethernet 普通头为 14 字节，IPv4 最少 20 字节，UDP 固定 8 字节，TCP 最少 20 字节。
8. “能找到协议头并打印 Payload”只是解析器原型，不等于完整协议栈。
9. 完整 UDP 还需要 ARP、IP、校验和、端口分发、队列和 API；完整 TCP 还需要状态机、序列号、重传、窗口、定时器等。
10. POSIX `recv()` 读取的是 Socket 接收缓冲区；用户态协议栈必须自己建立 fd、Socket、TCB 和收发队列之间的关系。
11. 当前测试通过静态 ARP 绕过了尚未实现的 ARP Reply，成功证明了 macOS、Fusion 桥接、vmxnet3、DPDK RX 和基础 UDP 解析链路已打通。
12. 下一步最重要的不是立即写 TCP，而是先补齐 mbuf 释放、长度校验、ARP Reply、TX 和 UDP Echo。

---

## 二十、版本说明与官方参考

你的学习材料明显包含旧版 DPDK 构建流程，因此本笔记保留了 `dpdk-setup.sh`、`RTE_SDK`、`RTE_TARGET` 的历史解释，同时把现代版本单独说明。实际执行命令时，必须先运行 `dpdk-testpmd -v`、查看源码目录中的 release 信息或构建文件，确认本机 DPDK 版本。

- [DPDK Linux Getting Started Guide](https://doc.dpdk.org/guides/linux_gsg/)：现代 Linux 环境、编译、驱动和运行入口；
- [DPDK Poll Mode Driver](https://doc.dpdk.org/guides/prog_guide/ethdev/ethdev.html)：PMD 绕过传统内核网络栈、直接轮询 RX/TX Descriptor 的官方说明；
- [DPDK Linux Drivers](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html)：VFIO、UIO、设备绑定和安全性说明；
- [DPDK VMXNET3 PMD](https://doc.dpdk.org/guides/nics/vmxnet3.html)：vmxnet3 收发环、多队列/RSS、功能与限制；
- [DPDK Hugepage Tool](https://doc.dpdk.org/guides/tools/hugepages.html)：查看、预留和挂载大页；
- [DPDK 22.11 KNI 文档](https://doc.dpdk.org/guides-22.11/prog_guide/kernel_nic_interface.html)：KNI 的用途、弃用状态及 virtio-user 替代说明。
