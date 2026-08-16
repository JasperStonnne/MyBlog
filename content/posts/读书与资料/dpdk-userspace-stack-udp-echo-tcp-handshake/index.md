
---
title: DPDK 用户态协议栈（二）：从收包到 UDP Echo 与 TCP 握手
slug: dpdk-userspace-stack-udp-echo-tcp-handshake
description: 从 TX Queue 配置与 mbuf 所有权出发，完整梳理 UDP Echo 回包、IPv4/UDP/TCP 校验和、最小 TCP 三次握手状态机及 DPDK 流量生成工具的设计
date: 2026-08-17T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - DPDK
  - 用户态协议栈
  - UDP
  - TCP
  - 网络编程
categories:
  - 后端开发
---

> 本文承接《dpdk用户态协议栈设计与实现》。本文把重点放在一个新问题上：**收到数据以后，怎样在用户态构造一个符合协议的响应包并发送出去？** 为了让代码能够独立读懂，后文会从程序入口开始完整解释执行流程；网络分层等原理知识则只在影响代码理解时说明。

## 一、本篇要完成什么

上一篇的程序已经可以：

1. 初始化 EAL、mempool 和网口；
2. 通过 `rte_eth_rx_burst()` 轮询收包；
3. 识别 Ethernet、IPv4、UDP 和 TCP；
4. 打印报文中的地址、端口与数据。

本篇继续完成三件事：

1. 为网口配置 TX Queue，使程序具备发包能力；
2. 实现 UDP Echo：收到 UDP 数据后，构造响应包并原样返回 payload；
3. 实现最小 TCP 服务端：收到 SYN 后回复 SYN+ACK，再接收 ACK 和应用数据。

这里实现的是用于学习协议与验证收发链路的最小原型，不是可投入生产的完整协议栈。

---

## 二、用编译期开关隔离增量代码

开发新功能时，可以先用宏把新增代码包起来：

```c
#define ENABLE_SEND 1
#define ENABLE_TCP  1

#if ENABLE_SEND
/* TX Queue、UDP 回包等新增代码 */
#endif

#if ENABLE_TCP
/* TCP 状态机与 TCP 回包代码 */
#endif
```

这样做的价值不是“上线前把代码删掉”，而是：

- 新功能出问题时，可快速回退到仅收包版本；
- 可以分别验证 RX、UDP TX 和 TCP 三部分；
- 提交 patch 时，新增逻辑的边界更加清楚。

如果功能会长期存在，最终更适合使用构建选项或运行时配置；长期散落大量 `#if` 会增加维护成本。

---

## 三、能收包不等于能发包

仅配置 RX Queue 时，网卡只能供当前程序收包。要发送数据，还需要完成三步：

```c
const uint16_t num_rx_queues = 1;
const uint16_t num_tx_queues = ENABLE_SEND ? 1 : 0;

if (rte_eth_dev_configure(port_id,
                          num_rx_queues,
                          num_tx_queues,
                          &port_conf) < 0) {
    rte_exit(EXIT_FAILURE, "Could not configure port\n");
}

#if ENABLE_SEND
struct rte_eth_txconf txq_conf = dev_info.default_txconf;
txq_conf.offloads = port_conf.txmode.offloads;

if (rte_eth_tx_queue_setup(port_id,
                           0,
                           512,
                           rte_eth_dev_socket_id(port_id),
                           &txq_conf) < 0) {
    rte_exit(EXIT_FAILURE, "Could not setup TX queue\n");
}
#endif

if (rte_eth_dev_start(port_id) < 0) {
    rte_exit(EXIT_FAILURE, "Could not start port\n");
}
```

完整顺序是：

```text
rte_eth_dev_configure
        ↓
rte_eth_rx_queue_setup / rte_eth_tx_queue_setup
        ↓
rte_eth_dev_start
        ↓
rte_eth_rx_burst / rte_eth_tx_burst
```

### `rte_eth_tx_burst()` 的返回值

```c
uint16_t nb_tx = rte_eth_tx_burst(port_id, 0, &tx_mbuf, 1);

if (nb_tx == 0) {
    rte_pktmbuf_free(tx_mbuf);
}
```

返回值表示实际被 TX Queue 接收的 mbuf 数量：

- 返回 `1`：mbuf 的所有权已交给 DPDK/驱动，应用不能立即释放或继续修改；
- 返回 `0`：该 mbuf 没有成功入队，仍由应用负责，应重试或释放。

批量发送时，只能释放返回值之后的未发送部分：

```c
uint16_t nb_tx = rte_eth_tx_burst(port_id, 0, tx_pkts, nb_pkts);

for (uint16_t i = nb_tx; i < nb_pkts; i++) {
    rte_pktmbuf_free(tx_pkts[i]);
}
```

这是 DPDK 发包代码中很重要的内存所有权规则。

---

## 四、为什么把收到的 mbuf 直接发出去不是 Echo

最初的尝试是：

```c
rte_eth_tx_burst(port_id, 0, &mbufs[i], 1);
```

这只是把收到的原始帧再次发出。原包仍然是：

```text
客户端 MAC/IP/Port  →  服务端 MAC/IP/Port
```

而响应包必须是：

```text
服务端 MAC/IP/Port  →  客户端 MAC/IP/Port
```

因此至少要处理：

| 协议层 | 响应包需要修改的内容 |
|---|---|
| Ethernet | 源 MAC 与目的 MAC 对调 |
| IPv4 | 源 IP 与目的 IP 对调，重新计算 IPv4 头校验和 |
| UDP | 源端口与目的端口对调，重新计算 UDP 校验和 |
| TCP | 对调端口，并根据状态设置 Flags、SEQ、ACK 和校验和 |

以太网 FCS/CRC 通常由网卡在真正发出帧时生成，应用一般不需要在 mbuf 末尾手工追加；但 IPv4、UDP、TCP 的校验和必须由软件计算，或者明确配置硬件 checksum offload。

---

## 五、UDP Echo 的数据路径

UDP Echo 的完整过程是：

```text
RX Queue 收到请求
        ↓
检查 Ethernet / IPv4 / UDP 及各层长度
        ↓
保存并对调 MAC、IP、Port
        ↓
从 mempool 分配新的 TX mbuf
        ↓
依次写入 Ethernet、IPv4、UDP Header
        ↓
复制原 UDP payload
        ↓
计算 IPv4 与 UDP checksum
        ↓
rte_eth_tx_burst 发送
```

为便于学习，草稿使用了全局变量暂存一组通信端点：

```c
uint8_t  global_smac[RTE_ETHER_ADDR_LEN];
uint8_t  global_dmac[RTE_ETHER_ADDR_LEN];
uint32_t global_sip;
uint32_t global_dip;
uint16_t global_sport;
uint16_t global_dport;
```

收到请求后，对调方向：

```c
rte_memcpy(global_smac,
           eth->dst_addr.addr_bytes,
           RTE_ETHER_ADDR_LEN);
rte_memcpy(global_dmac,
           eth->src_addr.addr_bytes,
           RTE_ETHER_ADDR_LEN);

global_sip   = ip->dst_addr;
global_dip   = ip->src_addr;
global_sport = udp->dst_port;
global_dport = udp->src_port;
```

这些字段本身仍保持网络字节序，写回协议头时不需要再次 `hton*()`。

### 不要用全局变量保存真实连接

全局变量只适合单队列、单线程、一次处理一个包的教学程序。多个客户端或多个 lcore 并发时，后一包会覆盖前一包的端点信息。更合理的做法是把地址信息作为函数参数传入，TCP 则使用以四元组为 key 的连接表。

---

## 六、构造 UDP 响应包

### 1. 先算清三种长度

设 UDP payload 长度为 `payload_len`：

```c
uint16_t udp_len = sizeof(struct rte_udp_hdr) + payload_len;
uint16_t ip_len  = sizeof(struct rte_ipv4_hdr) + udp_len;
uint16_t frame_len = sizeof(struct rte_ether_hdr) + ip_len;
```

三者的含义不同：

- `udp->dgram_len`：UDP Header + UDP payload；
- `ip->total_length`：IPv4 Header + IPv4 payload，不含 Ethernet Header；
- `mbuf->pkt_len/data_len`：本例中的完整 Ethernet 帧长度，不含硬件生成的 FCS。

### 2. 分配并扩展 mbuf

更稳妥的写法是使用 `rte_pktmbuf_append()`，而不是只手工改长度字段：

```c
struct rte_mbuf *tx_mbuf = rte_pktmbuf_alloc(mbuf_pool);
if (tx_mbuf == NULL) {
    return -ENOMEM;
}

uint8_t *frame = rte_pktmbuf_append(tx_mbuf, frame_len);
if (frame == NULL) {
    rte_pktmbuf_free(tx_mbuf);
    return -ENOSPC;
}
```

`rte_pktmbuf_append()` 会检查尾部空间，并同步更新 `data_len` 与 `pkt_len`。

### 3. 写 Ethernet Header

```c
struct rte_ether_hdr *eth = (struct rte_ether_hdr *)frame;

rte_memcpy(eth->dst_addr.addr_bytes,
           global_dmac,
           RTE_ETHER_ADDR_LEN);
rte_memcpy(eth->src_addr.addr_bytes,
           global_smac,
           RTE_ETHER_ADDR_LEN);
eth->ether_type = rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4);
```

不同 DPDK 版本中，成员名可能是 `d_addr/s_addr` 或 `dst_addr/src_addr`，应以本机头文件为准。

### 4. 写 IPv4 Header

```c
struct rte_ipv4_hdr *ip = (struct rte_ipv4_hdr *)(eth + 1);

*ip = (struct rte_ipv4_hdr){0};
ip->version_ihl     = RTE_IPV4_VHL_DEF;
ip->total_length    = rte_cpu_to_be_16(ip_len);
ip->time_to_live    = 64;
ip->next_proto_id   = IPPROTO_UDP;
ip->src_addr        = global_sip;
ip->dst_addr        = global_dip;
ip->hdr_checksum    = 0;
ip->hdr_checksum    = rte_ipv4_cksum(ip);
```

`version_ihl = 0x45` 表示 IPv4，且头部长度为 `5 × 4 = 20` 字节。使用 `RTE_IPV4_VHL_DEF` 可读性更好。

`total_length` 是 16 位多字节字段，写入协议头前必须转为网络字节序。

### 5. 写 UDP Header 和 payload

```c
struct rte_udp_hdr *udp = (struct rte_udp_hdr *)(ip + 1);

udp->src_port  = global_sport;
udp->dst_port  = global_dport;
udp->dgram_len = rte_cpu_to_be_16(udp_len);
udp->dgram_cksum = 0;

rte_memcpy(udp + 1, payload, payload_len);

udp->dgram_cksum = rte_ipv4_udptcp_cksum(ip, udp);
```

这里最容易写错的是复制长度。`udp_len` 包含 8 字节 UDP Header，因此复制 payload 时必须使用：

```c
payload_len = udp_len - sizeof(struct rte_udp_hdr);
```

不能把整个 `udp_len` 字节都从 `udp + 1` 开始复制，否则会越界读取原报文，并可能越界写入新 mbuf。

### 6. 一个整理后的编码函数

```c
static int encode_udp_echo(uint8_t *frame,
                           const uint8_t *payload,
                           uint16_t payload_len)
{
    uint16_t udp_len = sizeof(struct rte_udp_hdr) + payload_len;
    uint16_t ip_len = sizeof(struct rte_ipv4_hdr) + udp_len;

    struct rte_ether_hdr *eth = (struct rte_ether_hdr *)frame;
    rte_memcpy(eth->dst_addr.addr_bytes, global_dmac,
               RTE_ETHER_ADDR_LEN);
    rte_memcpy(eth->src_addr.addr_bytes, global_smac,
               RTE_ETHER_ADDR_LEN);
    eth->ether_type = rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4);

    struct rte_ipv4_hdr *ip = (struct rte_ipv4_hdr *)(eth + 1);
    *ip = (struct rte_ipv4_hdr){0};
    ip->version_ihl   = RTE_IPV4_VHL_DEF;
    ip->total_length  = rte_cpu_to_be_16(ip_len);
    ip->time_to_live  = 64;
    ip->next_proto_id = IPPROTO_UDP;
    ip->src_addr      = global_sip;
    ip->dst_addr      = global_dip;
    ip->hdr_checksum  = rte_ipv4_cksum(ip);

    struct rte_udp_hdr *udp = (struct rte_udp_hdr *)(ip + 1);
    udp->src_port = global_sport;
    udp->dst_port = global_dport;
    udp->dgram_len = rte_cpu_to_be_16(udp_len);
    udp->dgram_cksum = 0;

    rte_memcpy(udp + 1, payload, payload_len);
    udp->dgram_cksum = rte_ipv4_udptcp_cksum(ip, udp);

    return 0;
}
```

---

## 七、安全取得 UDP payload

草稿中使用：

```c
printf("udp: %s\n", (char *)(udp + 1));
```

这只能用于确保 payload 末尾带 `\0` 的演示数据。网络报文是带长度的二进制数据，不保证是 C 字符串。

在不考虑 IPv4 Options 的最小版本中：

```c
uint16_t udp_len = rte_be_to_cpu_16(udp->dgram_len);
if (udp_len < sizeof(struct rte_udp_hdr)) {
    /* malformed packet */
}

uint16_t payload_len = udp_len - sizeof(struct rte_udp_hdr);
uint8_t *payload = (uint8_t *)(udp + 1);

printf("udp: %.*s\n", (int)payload_len, (char *)payload);
```

真正处理报文前，还要结合 `rte_pktmbuf_pkt_len(mbuf)` 检查 Ethernet、IP、UDP 声明的长度是否落在 mbuf 实际数据范围内。需要记住：**构造包和解析包都必须以显式长度为边界。**

### 打印 IP 和端口

```c
struct in_addr addr = { .s_addr = ip->src_addr };
printf("sip %s:%u --> ", inet_ntoa(addr),
       rte_be_to_cpu_16(udp->src_port));

addr.s_addr = ip->dst_addr;
printf("dip %s:%u\n", inet_ntoa(addr),
       rte_be_to_cpu_16(udp->dst_port));
```

`inet_ntoa()` 使用内部静态缓冲区，不适合并发代码；工程中优先使用 `inet_ntop()`。

---

## 八、从 UDP 过渡到 TCP：变化不只是 Header

UDP 收到一个数据报就可以立即生成一个独立响应。TCP 则必须维护连接状态，因为一个 TCP 报文的正确响应取决于此前发生过什么。

最小服务端路径为：

```text
客户端                         DPDK 服务端
  | -------- SYN, seq=x --------> |
  | <--- SYN+ACK, seq=y, ack=x+1 -|
  | -------- ACK, ack=y+1 ------> |
  |                               |  ESTABLISHED
  | ------ PSH+ACK, data --------> |
```

草稿定义了完整 TCP 状态枚举：

```c
typedef enum {
    USTACK_TCP_STATUS_CLOSED = 0,
    USTACK_TCP_STATUS_LISTEN,
    USTACK_TCP_STATUS_SYN_RCVD,
    USTACK_TCP_STATUS_SYN_SENT,
    USTACK_TCP_STATUS_ESTABLISHED,
    USTACK_TCP_STATUS_FIN_WAIT_1,
    USTACK_TCP_STATUS_FIN_WAIT_2,
    USTACK_TCP_STATUS_CLOSING,
    USTACK_TCP_STATUS_TIME_WAIT,
    USTACK_TCP_STATUS_CLOSE_WAIT,
    USTACK_TCP_STATUS_LAST_ACK
} ustack_tcp_status_t;
```

当前代码实际只实现了：

```text
LISTEN → SYN_RCVD → ESTABLISHED
```

因此准确的描述是“完成服务端三次握手的最小实验”，而不是“已经实现 TCP 协议栈”。重传、乱序、滑动窗口、拥塞控制、RST、FIN 四次挥手、TIME_WAIT、TCP Options 和多连接管理都还没有实现。

---

## 九、TCP 的 SEQ 与 ACK 应该怎样理解

收到客户端 SYN：

```text
client.seq = x
```

服务端回复 SYN+ACK：

```text
server.seq = y
server.ack = x + 1
```

SYN 虽然不携带应用数据，但会占用一个序列号，所以确认号是 `x + 1`。FIN 同样占用一个序列号。

对于普通数据段：

```text
下一确认号 = 当前 seq + TCP payload 长度
```

如果同一段还带 SYN 或 FIN，还应分别再加 1。

草稿中的：

```c
tcp->sent_seq = htonl(12345);
tcp->recv_ack = htonl(global_seqnum + 1);
```

足以演示 SYN+ACK，但 `12345` 应被视为教学用的初始序列号。真实实现需要为每条连接保存本端/对端序列号，并校验收到的 ACK 是否真正确认了自己发送的 SYN 或数据。

---

## 十、构造最小 SYN+ACK

TCP 响应与 UDP 的 Ethernet、IPv4 部分基本相同；区别在 TCP Header：

```c
struct rte_tcp_hdr *tcp = (struct rte_tcp_hdr *)(ip + 1);
*tcp = (struct rte_tcp_hdr){0};

tcp->src_port = global_sport;
tcp->dst_port = global_dport;
tcp->sent_seq = rte_cpu_to_be_32(server_isn);
tcp->recv_ack = rte_cpu_to_be_32(client_seq + 1);
tcp->data_off = (sizeof(struct rte_tcp_hdr) / 4) << 4;
tcp->tcp_flags = RTE_TCP_SYN_FLAG | RTE_TCP_ACK_FLAG;
tcp->rx_win = rte_cpu_to_be_16(TCP_INIT_WINDOW);
tcp->cksum = 0;
tcp->cksum = rte_ipv4_udptcp_cksum(ip, tcp);
```

几个容易忽略的点：

- `data_off` 的高 4 位表示 TCP Header 占几个 32 位字；基础头 20 字节，所以值为 `5 << 4`，即 `0x50`；
- `rx_win` 是多字节字段，应使用网络字节序；
- 计算 checksum 前，checksum 字段必须先清零；
- `struct rte_tcp_hdr` 最好整体清零，避免未初始化的 `cksum`、`tcp_urp` 等字段混入报文；
- IPv4 `total_length` 必须覆盖 IPv4 Header、TCP Header 和 TCP payload。

### 为什么 UDP 头有长度，而 TCP 头没有

UDP 自身以数据报为边界，因此 Header 中有 `dgram_len`：

```text
UDP 长度 = UDP Header + UDP payload
```

TCP 是字节流协议，段长度可以从 IPv4 总长度推导：

```text
TCP 段长度 = IP total_length - IP Header 长度
TCP payload 长度 = TCP 段长度 - TCP Header 长度
```

对应代码：

```c
uint16_t ip_hdr_len = (ip->version_ihl & 0x0f) * 4;
uint16_t tcp_hdr_len = (tcp->data_off >> 4) * 4;
uint16_t ip_total_len = rte_be_to_cpu_16(ip->total_length);

uint16_t tcp_payload_len =
    ip_total_len - ip_hdr_len - tcp_hdr_len;
```

因此，TCP 的确认号并不是“长度字段”，但它会随着已经确认的字节数向前推进，体现了 TCP 字节流的位置。

---

## 十一、最小 TCP 状态处理

### 1. LISTEN 收到 SYN

```c
if ((flags & RTE_TCP_SYN_FLAG) &&
    state == USTACK_TCP_STATUS_LISTEN) {
    client_next_seq = rte_be_to_cpu_32(tcp->sent_seq) + 1;
    send_syn_ack(...);
    state = USTACK_TCP_STATUS_SYN_RCVD;
}
```

### 2. SYN_RCVD 收到有效 ACK

不能只检查 ACK 位，还应验证确认号：

```c
if ((flags & RTE_TCP_ACK_FLAG) &&
    state == USTACK_TCP_STATUS_SYN_RCVD &&
    rte_be_to_cpu_32(tcp->recv_ack) == server_isn + 1) {
    state = USTACK_TCP_STATUS_ESTABLISHED;
}
```

### 3. ESTABLISHED 收到数据

`PSH` 只是提示接收端尽快把数据交给应用，不是“存在 payload”的唯一判断条件。更可靠的判断是计算 `tcp_payload_len > 0`：

```c
uint16_t payload_len = get_tcp_payload_len(ip, tcp);

if (state == USTACK_TCP_STATUS_ESTABLISHED && payload_len > 0) {
    const uint8_t *payload = (const uint8_t *)tcp + tcp_hdr_len;
    printf("tcp data: %.*s\n", (int)payload_len,
           (const char *)payload);

    /* 回复 ACK 时：ack = peer_seq + payload_len */
}
```

草稿能够打印 TCP 数据，但还没有为数据段回复 ACK，也没有处理 FIN；一些客户端因此会持续重传或无法正常关闭连接。

---

## 十二、从草稿中提炼出的典型 Bug

### 1. IPv4 地址只复制了 2 字节

错误版本：

```c
rte_memcpy(&global_sip, &ip->dst_addr, sizeof(uint16_t));
```

IPv4 地址是 32 位，应复制 `sizeof(uint32_t)`，或者直接赋值：

```c
global_sip = ip->dst_addr;
global_dip = ip->src_addr;
```

### 2. 把 UDP 总长度当成 payload 长度

错误思路：

```c
rte_memcpy(udp + 1, data, udp_len);
```

正确关系：

```c
payload_len = udp_len - sizeof(struct rte_udp_hdr);
rte_memcpy(udp + 1, data, payload_len);
```

### 3. 分配了新 mbuf，却传错发送指针

错误版本：

```c
rte_eth_tx_burst(port_id, 0, &mbufs, 1);
```

应发送刚构造的单个 `mbuf`：

```c
rte_eth_tx_burst(port_id, 0, &mbuf, 1);
```

`&mbufs` 是整个 RX 指针数组的地址，语义也不正确。

### 4. 多字节字段漏做字节序转换

典型字段包括：

```c
ip->total_length = rte_cpu_to_be_16(ip_len);
udp->dgram_len   = rte_cpu_to_be_16(udp_len);
tcp->sent_seq    = rte_cpu_to_be_32(seq);
tcp->recv_ack    = rte_cpu_to_be_32(ack);
tcp->rx_win      = rte_cpu_to_be_16(window);
```

MAC 地址和 payload 是字节数组，不需要转换。

### 5. 不检查 TX 返回值

TX Queue 满时，`rte_eth_tx_burst()` 可以少发甚至不发。未成功发送的 mbuf 若不释放，会产生内存泄漏。

### 6. 只看 Flags，不校验状态与序列号

“收到带 ACK 标志的包”不等于三次握手完成。至少还要确认当前状态是 `SYN_RCVD`，且 ACK 值为服务端初始序列号加 1。

---

## 十三、完整代码逐行讲解

这一节按照程序真正的运行顺序，从头文件开始解释，直到收到、解析、构造并发送一个包。代码前面的数字只是讲解用行号，不是源代码的一部分。

### 1. 头文件分别提供什么

```c
01  #include <stdio.h>
02  #include <stdint.h>
03  #include <errno.h>
04  #include <arpa/inet.h>
05
06  #include <rte_eal.h>
07  #include <rte_ethdev.h>
08  #include <rte_mbuf.h>
09  #include <rte_ether.h>
10  #include <rte_ip.h>
11  #include <rte_udp.h>
12  #include <rte_tcp.h>
```

- 第 1 行：提供 `printf()`。
- 第 2 行：提供 `uint8_t`、`uint16_t`、`uint32_t` 等定长整数类型。网络协议字段有固定宽度，因此比普通 `int` 更合适。
- 第 3 行：提供 `ENOMEM`、`ENOSPC` 等错误码。
- 第 4 行：提供 `inet_ntop()`、`htons()` 等网络地址与字节序函数。
- 第 6 行：提供 `rte_eal_init()`、`rte_exit()` 等 DPDK 运行环境接口。
- 第 7 行：提供网口、RX/TX Queue 和 `rte_eth_rx_burst()`、`rte_eth_tx_burst()`。
- 第 8 行：提供 mempool、mbuf 分配、释放和数据地址转换接口。
- 第 9～12 行：分别提供 Ethernet、IPv4、UDP、TCP Header 结构体和协议常量。

`#include` 的作用是把函数声明、结构体定义和宏定义提供给当前 `.c` 文件。它不会替代链接步骤；编译时仍要通过 DPDK 的 `pkg-config` 参数链接对应库。

### 2. 全局端口、mbuf 数量与 Burst 大小

```c
01  static uint16_t global_portid = 0;
02
03  #define NUM_MBUFS  4096
04  #define BURST_SIZE 128
```

- 第 1 行：选择 DPDK 端口 0。它表示 DPDK 枚举得到的端口编号，不一定等于 Linux 中的网卡名称或 PCI 地址。
- `static`：这里表示变量只在当前源文件内可见，减少与其他文件同名变量冲突的机会。
- 第 3 行：mempool 预备 4096 个 mbuf。RX 和新建的 TX 包都会消耗池中的对象。
- 第 4 行：一次 `rte_eth_rx_burst()` 最多取回 128 个 mbuf 指针。

`NUM_MBUFS` 是整个对象池规模，`BURST_SIZE` 是一次批量操作的上限，两者不是同一概念。

### 3. 功能开关与发送端点

```c
01  #define ENABLE_SEND 1
02  #define ENABLE_TCP  1
03  #define TCP_INIT_WINDOW 14600
04
05  #if ENABLE_SEND
06  uint8_t  global_smac[RTE_ETHER_ADDR_LEN];
07  uint8_t  global_dmac[RTE_ETHER_ADDR_LEN];
08  uint32_t global_sip;
09  uint32_t global_dip;
10  uint16_t global_sport;
11  uint16_t global_dport;
12  #endif
```

- 第 1 行：控制是否编译发包代码。关闭后可退回仅收包版本。
- 第 2 行：控制是否编译 TCP 实验代码，可以只验证 UDP。
- 第 3 行：定义 SYN+ACK 中通告给客户端的接收窗口大小。
- 第 5～12 行：只有启用发送功能时才定义响应端点。
- 第 6～7 行：分别保存响应包的源 MAC 与目的 MAC。
- 第 8～9 行：IPv4 地址是 32 位，因此类型必须是 `uint32_t`。
- 第 10～11 行：TCP/UDP 端口是 16 位。

这些变量保存的地址和端口直接来自收到的协议头，所以仍是网络字节序。只有打印或参与主机侧数值计算时，才需要 `rte_be_to_cpu_16()`、`rte_be_to_cpu_32()`。

### 4. 端口默认配置

```c
01  static const struct rte_eth_conf port_conf_default = {
02      .rxmode = {
03          .max_rx_pkt_len = RTE_ETHER_MAX_LEN
04      }
05  };
```

- 第 1 行：定义一个网口配置结构体，并用 `const` 表示运行中不修改它。
- `.rxmode = { ... }`：这是 C 语言的指定初始化器，只填写 `rxmode` 成员，其余成员自动初始化为 0。
- 第 3 行：把最大接收包长设为普通 Ethernet 帧允许的范围。

不同 DPDK 版本的 `struct rte_eth_conf` 字段可能变化。如果本机版本提示 `max_rx_pkt_len` 不存在，应检查安装版本的头文件和示例，而不是强行照抄字段名。

### 5. 查询可用端口和设备默认能力

```c
01  static int ustack_init_port(struct rte_mempool *mbuf_pool)
02  {
03      uint16_t nb_ports = rte_eth_dev_count_avail();
04
05      if (nb_ports == 0) {
06          rte_exit(EXIT_FAILURE,
07                   "No supported Ethernet device found\n");
08      }
09
10      if (!rte_eth_dev_is_valid_port(global_portid)) {
11          rte_exit(EXIT_FAILURE, "Invalid port id\n");
12      }
13
14      struct rte_eth_dev_info dev_info;
15      int ret = rte_eth_dev_info_get(global_portid, &dev_info);
16      if (ret != 0) {
17          rte_exit(EXIT_FAILURE,
18                   "Cannot get device info: %d\n", ret);
19      }
```

- 第 1 行：函数接收 mempool 指针，因为 RX Queue 需要知道从哪个池获取接收缓冲区。
- 第 3 行：查询当前由 DPDK 管理且可用的 Ethernet 端口数量。
- 第 5～8 行：一个可用端口都没有时直接退出。常见原因包括网卡没有绑定到合适驱动、权限不足或 EAL 参数错误。
- 第 10 行：即使系统存在端口，也要确认准备使用的 `global_portid` 有效。
- 第 14 行：声明设备信息结构体，用于接收驱动默认配置和网卡能力。
- 第 15 行：读取 0 号端口的信息。
- 第 16～18 行：检查 API 返回值；不能在查询失败后继续使用未初始化的 `dev_info`。

### 6. 配置 RX Queue 与 TX Queue 的数量

```c
01  const uint16_t num_rx_queues = 1;
02
03  #if ENABLE_SEND
04  const uint16_t num_tx_queues = 1;
05  #else
06  const uint16_t num_tx_queues = 0;
07  #endif
08
09  rte_eth_dev_configure(global_portid,
10                        num_rx_queues,
11                        num_tx_queues,
12                        &port_conf_default);
13
14  #if ENABLE_SEND
15  struct rte_eth_txconf txq_conf = dev_info.default_txconf;
16  txq_conf.offloads = port_conf_default.txmode.offloads;
17
18  if (rte_eth_tx_queue_setup(global_portid,
19                             0,
20                             512,
21                             rte_eth_dev_socket_id(global_portid),
22                             &txq_conf) < 0) {
23      rte_exit(EXIT_FAILURE, "Could not setup TX queue\n");
24  }
25  #endif
```

- 第 1 行：仍然只配置一个 RX Queue。
- 第 3～7 行：启用发送时创建一个 TX Queue，否则把 TX Queue 数量设为 0。
- 第 9～12 行：网口配置阶段必须提前声明 RX/TX Queue 的数量。
- 第 15 行：从设备能力信息中复制驱动提供的默认 TX 配置，避免凭空构造不兼容的参数。
- 第 16 行：把端口级 TX offload 配置同步给队列。草稿整理前曾写成 `rxmode.offloads`，此处应与 TX 配置对应。
- 第 18 行：开始配置 TX Queue。
- 第 19 行：队列编号为 0，因为当前只创建一个发送队列。
- 第 20 行：TX 描述符数量为 512，它描述队列容量，不是单次最多只能发送 512 个包。
- 第 21 行：让队列内存尽量位于网卡所属的 NUMA Socket。
- 第 22 行：传入刚才复制并调整的 TX 配置。
- 第 23 行：配置失败就退出，避免程序进入一个必然无法发送的循环。

### 7. 配置 RX Queue

```c
01  if (rte_eth_rx_queue_setup(global_portid,
02                             0,
03                             128,
04                             rte_eth_dev_socket_id(global_portid),
05                             NULL,
06                             mbuf_pool) < 0) {
07      rte_exit(EXIT_FAILURE, "Could not setup RX queue\n");
08  }
```

- 第 1 行：为指定网口建立接收队列。
- 第 2 行：队列编号为 0。
- 第 3 行：配置 128 个 RX 描述符。描述符用于让网卡和驱动追踪接收缓冲区，不等于只创建 128 个 mbuf。
- 第 4 行：选择网卡所在 NUMA Socket。
- 第 5 行：传 `NULL` 表示使用驱动默认 RX Queue 配置。
- 第 6 行：把前面创建的 mempool 交给 RX Queue。网卡收到包后，驱动会使用池中的 mbuf 承载数据。
- 第 7 行：队列创建失败就退出。

### 8. 启动端口并返回

```c
01  if (rte_eth_dev_start(global_portid) < 0) {
02      rte_exit(EXIT_FAILURE, "Could not start port\n");
03  }
04
05  return 0;
06  }
```

- 第 1 行：前面的 configure 和 queue setup 只是准备配置，这里才真正启动设备。
- 第 2 行：启动失败不能进入收发循环。
- 第 5 行：返回 0 表示端口初始化成功。
- 第 6 行：结束 `ustack_init_port()` 函数。

### 9. `main()` 的参数与 EAL 初始化

```c
01  int main(int argc, char **argv)
02  {
03      int ret = rte_eal_init(argc, argv);
04      if (ret < 0) {
05          rte_exit(EXIT_FAILURE, "Error with EAL init\n");
06      }
```

- 第 1 行：`argc` 是命令行参数数量，`argv` 是字符串指针数组。
- 第 3 行：初始化 DPDK EAL。它会处理 lcore、内存、设备等 DPDK 参数。
- 返回负数表示初始化失败。
- 返回非负数表示 EAL 消耗掉的命令行参数数量。如果程序还要解析自己的参数，可以使用 `argc -= ret; argv += ret;` 跳过 EAL 参数。
- 第 4～5 行：初始化失败时打印错误并结束进程。

EAL 必须先初始化，后面创建 mempool、查询端口和配置设备才有可用的 DPDK 运行环境。

### 10. 创建 mbuf pool

```c
07      struct rte_mempool *mbuf_pool =
08          rte_pktmbuf_pool_create("mbuf_pool",
09                                  NUM_MBUFS,
10                                  0,
11                                  0,
12                                  RTE_MBUF_DEFAULT_BUF_SIZE,
13                                  rte_socket_id());
14
15      if (mbuf_pool == NULL) {
16          rte_exit(EXIT_FAILURE,
17                   "Could not create mbuf pool\n");
18      }
```

- 第 7 行：声明 mempool 指针，用来保存创建结果。
- 第 8 行：池名称用于 DPDK 内部查找和调试，同一进程中应保持唯一。
- 第 9 行：池中创建 `NUM_MBUFS` 个 mbuf 对象。
- 第 10 行：per-lcore cache 大小设为 0，逻辑简单，但性能实验中通常会选择合适缓存值。
- 第 11 行：每个 mbuf 不添加额外的应用私有区。
- 第 12 行：每个 mbuf 的数据缓冲区使用 DPDK 默认大小。
- 第 13 行：在当前执行 lcore 所在 NUMA Socket 分配池内存。
- 第 15～18 行：创建失败时不能继续配置 RX Queue。

### 11. 初始化网口并进入无限循环

```c
19      ustack_init_port(mbuf_pool);
20
21      while (1) {
22          struct rte_mbuf *mbufs[BURST_SIZE] = {0};
```

- 第 19 行：把刚创建的 mempool 交给端口初始化函数，并建立 RX/TX Queue。
- 第 21 行：DPDK 常使用轮询模式，所以程序持续运行，不断检查 RX Queue。
- 第 22 行：在栈上创建一个数组，用来保存本次批量取回的 mbuf 指针。数组中不是报文数据本身，而是指向 mbuf 的指针。
- `= {0}` 会把所有数组元素初始化为空指针；虽然接下来有效元素会被 API 覆盖，但初始化有利于调试。

### 12. 从 RX Queue 批量取包

```c
23          uint16_t num_recvd =
24              rte_eth_rx_burst(global_portid,
25                               0,
26                               mbufs,
27                               BURST_SIZE);
28
29          for (uint16_t i = 0; i < num_recvd; i++) {
30              struct rte_mbuf *rx_mbuf = mbufs[i];
31              uint32_t frame_len = rte_pktmbuf_pkt_len(rx_mbuf);
```

- 第 23～27 行：从 0 号端口、0 号 RX Queue 最多取 `BURST_SIZE` 个包。
- 返回 0 表示当前没有包，不是错误；`while` 会立即进入下一轮轮询。
- 返回值不会大于传入的 `BURST_SIZE`，所以原草稿中的 `num_recvd > BURST_SIZE` 棬查没有实际意义。
- 第 29 行：只遍历 API 确认有效的前 `num_recvd` 个元素。
- 第 30 行：为当前包取一个更易读的局部变量名。
- 第 31 行：读取整个包的实际字节数，后面定位 Header 前必须先检查长度。

### 13. 检查并定位 Ethernet Header

```c
32              if (frame_len < sizeof(struct rte_ether_hdr)) {
33                  rte_pktmbuf_free(rx_mbuf);
34                  continue;
35              }
36
37              struct rte_ether_hdr *ethhdr =
38                  rte_pktmbuf_mtod(rx_mbuf,
39                                   struct rte_ether_hdr *);
40
41              if (ethhdr->ether_type !=
42                  rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
43                  rte_pktmbuf_free(rx_mbuf);
44                  continue;
45              }
```

- 第 32 行：报文连一个 Ethernet Header 都放不下时，不能继续强制转换并读取字段。
- 第 33 行：当前 mbuf 没有交给 TX，应用仍拥有它，因此需要释放。
- 第 34 行：跳过当前包，继续处理 Burst 中的下一个包。
- 第 37～39 行：`rte_pktmbuf_mtod()` 取得 mbuf 数据起始地址，并转换为 Ethernet Header 指针。
- 第 41～42 行：EtherType 等于 IPv4 才进入本实验的 IPv4 处理路径。
- 第 43～44 行：非 IPv4 包当前不处理，但仍必须释放 mbuf。

`rte_pktmbuf_mtod()` 只是做地址取得与类型转换，不会自动校验数据是否足够长，所以第 32 行不能省略。

### 14. 检查并定位 IPv4 Header

```c
46              uint32_t min_ipv4_frame =
47                  sizeof(struct rte_ether_hdr) +
48                  sizeof(struct rte_ipv4_hdr);
49
50              if (frame_len < min_ipv4_frame) {
51                  rte_pktmbuf_free(rx_mbuf);
52                  continue;
53              }
54
55              struct rte_ipv4_hdr *iphdr =
56                  rte_pktmbuf_mtod_offset(
57                      rx_mbuf,
58                      struct rte_ipv4_hdr *,
59                      sizeof(struct rte_ether_hdr));
60
61              uint16_t ip_hdr_len =
62                  (iphdr->version_ihl & 0x0f) * 4;
63
64              if (ip_hdr_len < sizeof(struct rte_ipv4_hdr) ||
65                  frame_len < sizeof(struct rte_ether_hdr) +
66                              ip_hdr_len) {
67                  rte_pktmbuf_free(rx_mbuf);
68                  continue;
69              }
```

- 第 46～48 行：计算 Ethernet Header 加最小 IPv4 Header 需要多少字节。
- 第 50～53 行：实际帧比这个值短时直接丢弃。
- 第 55～59 行：从 mbuf 数据起点偏移一个 Ethernet Header，取得 IPv4 Header 地址。
- 第 61～62 行：IPv4 IHL 位于 `version_ihl` 的低 4 位，单位是 4 字节。
- 第 64 行：IHL 小于 20 字节不合法。
- 第 65～66 行：实际帧必须能容纳 IHL 声明的完整 IPv4 Header。
- 第 67～68 行：格式不合法就释放并跳过。

当前示例默认 Header 位于 mbuf 的第一个连续数据段。若接收的是 multi-segment mbuf，直接取指针前还要确保所需字节连续，或使用 `rte_pktmbuf_read()`。

### 15. 根据 IHL 定位 UDP 或 TCP Header

```c
70              uint8_t *l4 = (uint8_t *)iphdr + ip_hdr_len;
71
72              if (iphdr->next_proto_id == IPPROTO_UDP) {
73                  if (frame_len <
74                      sizeof(struct rte_ether_hdr) + ip_hdr_len +
75                      sizeof(struct rte_udp_hdr)) {
76                      rte_pktmbuf_free(rx_mbuf);
77                      continue;
78                  }
79
80                  struct rte_udp_hdr *udphdr =
81                      (struct rte_udp_hdr *)l4;
82                  /* 进入 UDP Echo 处理 */
83
84              } else if (iphdr->next_proto_id == IPPROTO_TCP) {
85                  if (frame_len <
86                      sizeof(struct rte_ether_hdr) + ip_hdr_len +
87                      sizeof(struct rte_tcp_hdr)) {
88                      rte_pktmbuf_free(rx_mbuf);
89                      continue;
90                  }
91
92                  struct rte_tcp_hdr *tcphdr =
93                      (struct rte_tcp_hdr *)l4;
94                  /* 进入 TCP 状态处理 */
95              }
```

- 第 70 行：不能总用 `iphdr + 1` 找四层 Header，因为 IPv4 可能带 Options；这里使用前面算出的真实 IHL。
- 第 72 行：`next_proto_id` 表示 IPv4 payload 使用哪种上层协议。
- 第 73～78 行：读取 UDP Header 之前，确认帧中至少有 8 字节 UDP Header。
- 第 80～81 行：把四层起点解释为 UDP Header。
- 第 84 行：如果不是 UDP，再检查是否为 TCP。
- 第 85～90 行：TCP 基础 Header 至少 20 字节，读取前同样检查实际长度。
- 第 92～93 行：把四层起点解释为 TCP Header。

到这里，程序才安全地取得 UDP 或 TCP Header。下面开始解释各协议的响应构造。

### 16. 收到 UDP 包后保存响应方向

```c
01  if (iphdr->next_proto_id == IPPROTO_UDP) {
02      struct rte_udp_hdr *udphdr =
03          (struct rte_udp_hdr *)((uint8_t *)iphdr + ip_hdr_len);
04
05      rte_memcpy(global_smac,
06                 ethhdr->dst_addr.addr_bytes,
07                 RTE_ETHER_ADDR_LEN);
08      rte_memcpy(global_dmac,
09                 ethhdr->src_addr.addr_bytes,
10                 RTE_ETHER_ADDR_LEN);
11
12      global_sip = iphdr->dst_addr;
13      global_dip = iphdr->src_addr;
14      global_sport = udphdr->dst_port;
15      global_dport = udphdr->src_port;
```

- 第 1 行：只进入 UDP 分支。
- 第 2～3 行：把 IPv4 Header 起点加上刚计算出的 `ip_hdr_len`，定位 UDP Header；这样 IPv4 带 Options 时也不会找错位置。
- 第 5～7 行：请求包的目的 MAC 是本机 MAC，它应成为响应包的源 MAC。
- 第 8～10 行：请求包的源 MAC 是客户端 MAC，它应成为响应包的目的 MAC。
- 第 12～13 行：用直接赋值对调 IPv4 地址，避免误写复制长度。
- 第 14～15 行：对调 UDP 端口，使响应发回请求客户端。

这里的“对调”不是在原协议头中原地交换，而是先保存响应方向，稍后写入一个新的 TX mbuf。

### 17. 计算收到的 UDP 数据长度

```c
16      uint16_t udp_len =
17          rte_be_to_cpu_16(udphdr->dgram_len);
18
19      if (udp_len < sizeof(struct rte_udp_hdr)) {
20          rte_pktmbuf_free(mbufs[i]);
21          continue;
22      }
23
24      uint16_t payload_len =
25          udp_len - sizeof(struct rte_udp_hdr);
26
27      uint8_t *payload = (uint8_t *)(udphdr + 1);
28      uint16_t ip_len =
29          sizeof(struct rte_ipv4_hdr) + udp_len;
30      uint16_t frame_len =
31          sizeof(struct rte_ether_hdr) + ip_len;
```

- 第 16～17 行：从 UDP Header 读取总长度，并转换为主机字节序，方便做减法和比较。
- 第 19 行：合法 UDP 长度至少为 8 字节，即一个 UDP Header。
- 第 20 行：这个示例假定当前分支负责释放收到的 mbuf。若外层统一释放，此处不能再次释放。
- 第 24～25 行：UDP payload 长度等于 UDP 总长度减去 UDP Header。
- 第 27 行：`udphdr + 1` 跨过一个完整的 UDP Header，指向 payload 起始位置。
- 第 28～29 行：IPv4 总长度由 IPv4 Header 和整个 UDP 数据报组成。
- 第 30～31 行：完整帧长度再加上 Ethernet Header。

在使用 UDP Header 声明的长度前，还必须确认它没有超过 IPv4 `total_length` 和 RX mbuf 的实际数据范围。前面的 Header 存在性检查只能证明 UDP Header 本身可读，不能证明对方声明的整个 payload 都真实存在。

### 18. 为响应包分配 TX mbuf

```c
32      struct rte_mbuf *tx_mbuf =
33          rte_pktmbuf_alloc(mbuf_pool);
34
35      if (tx_mbuf == NULL) {
36          printf("rte_pktmbuf_alloc failed\n");
37          continue;
38      }
39
40      uint8_t *frame =
41          rte_pktmbuf_append(tx_mbuf, frame_len);
42
43      if (frame == NULL) {
44          rte_pktmbuf_free(tx_mbuf);
45          continue;
46      }
```

- 第 32～33 行：从创建好的 mempool 中取一个 mbuf，用于保存响应包。
- 第 35 行：mempool 耗尽时分配会失败，不能继续解引用空指针。
- 第 40～41 行：在 mbuf 尾部追加 `frame_len` 字节，并取得可写的数据地址。
- 第 41 行：`rte_pktmbuf_append()` 同时更新 `data_len` 和 `pkt_len`，比手工写两个长度字段更安全。
- 第 43～45 行：如果 mbuf 的可用尾部空间不足，释放刚分配的 mbuf。

这里选择“新建 TX mbuf”，优点是 RX 包保持不变，逻辑直观。后续追求性能时，也可以在满足可写、连续性和所有权条件的前提下原地改写 RX mbuf 再发送。

### 19. 逐层写入 UDP 响应包

```c
47      struct rte_ether_hdr *eth =
48          (struct rte_ether_hdr *)frame;
49
50      rte_memcpy(eth->dst_addr.addr_bytes,
51                 global_dmac,
52                 RTE_ETHER_ADDR_LEN);
53      rte_memcpy(eth->src_addr.addr_bytes,
54                 global_smac,
55                 RTE_ETHER_ADDR_LEN);
56      eth->ether_type =
57          rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4);
```

- 第 47～48 行：把帧首地址解释成 Ethernet Header。
- 第 50～52 行：写目的 MAC，即原请求的源 MAC。
- 第 53～55 行：写源 MAC，即原请求的目的 MAC。
- 第 56～57 行：声明 Ethernet payload 是 IPv4。EtherType 是 16 位字段，要使用网络字节序。

接着写 IPv4 Header：

```c
58      struct rte_ipv4_hdr *ip =
59          (struct rte_ipv4_hdr *)(eth + 1);
60      *ip = (struct rte_ipv4_hdr){0};
61
62      ip->version_ihl = RTE_IPV4_VHL_DEF;
63      ip->total_length = rte_cpu_to_be_16(ip_len);
64      ip->time_to_live = 64;
65      ip->next_proto_id = IPPROTO_UDP;
66      ip->src_addr = global_sip;
67      ip->dst_addr = global_dip;
68      ip->hdr_checksum = 0;
69      ip->hdr_checksum = rte_ipv4_cksum(ip);
```

- 第 58～59 行：`eth + 1` 跨过 Ethernet Header，得到 IPv4 Header 地址。
- 第 60 行：先把整个 Header 清零，避免未初始化字段影响报文和 checksum。
- 第 62 行：写入 IPv4 与 20 字节基础头长度。
- 第 63 行：IPv4 总长度不包含 Ethernet Header。
- 第 64 行：TTL 设置为 64。
- 第 65 行：声明上层协议为 UDP。
- 第 66～67 行：写入已经对调好的源/目的 IPv4 地址。
- 第 68 行：计算 IPv4 Header checksum 前先把 checksum 字段清零。
- 第 69 行：计算并写入新的 IPv4 Header checksum。

最后写 UDP Header 和 payload：

```c
70      struct rte_udp_hdr *udp =
71          (struct rte_udp_hdr *)(ip + 1);
72
73      udp->src_port = global_sport;
74      udp->dst_port = global_dport;
75      udp->dgram_len = rte_cpu_to_be_16(udp_len);
76      udp->dgram_cksum = 0;
77
78      rte_memcpy(udp + 1, payload, payload_len);
79      udp->dgram_cksum =
80          rte_ipv4_udptcp_cksum(ip, udp);
```

- 第 70～71 行：基础 IPv4 Header 后紧跟 UDP Header。
- 第 73～74 行：写入对调后的端口；它们已经是网络字节序。
- 第 75 行：UDP 长度包括 UDP Header 和 UDP payload。
- 第 76 行：计算 UDP checksum 前先清零。
- 第 78 行：只复制 `payload_len`，不能复制包含 UDP Header 的 `udp_len`。
- 第 79～80 行：使用 IPv4 伪首部、UDP Header 和 payload 计算 UDP checksum。

### 20. 发送 UDP 响应并处理所有权

```c
81      uint16_t nb_tx =
82          rte_eth_tx_burst(global_portid,
83                           0,
84                           &tx_mbuf,
85                           1);
86
87      if (nb_tx == 0) {
88          rte_pktmbuf_free(tx_mbuf);
89      }
90  }
```

- 第 81～85 行：请求 0 号 TX Queue 发送一个 mbuf。
- 第 84 行：参数必须是 `&tx_mbuf`，即“mbuf 指针数组的首地址”。只有一个包时，单个指针的地址可以充当长度为 1 的数组。
- 第 85 行：本次希望发送 1 个包。
- 第 87 行：若返回 0，说明驱动没有接管这个 mbuf。
- 第 88 行：发送失败时所有权仍属于应用，所以由应用释放。
- 若返回 1，不能在这里释放；后续由驱动在发送完成后回收。

外层还必须释放没有转交给 TX 的 RX mbuf。若选择原地改写 RX mbuf 并成功发送，则该 RX mbuf 的所有权已经交给 TX，不能再按接收包释放一次。

### 21. TCP 状态与每条连接需要保存的数据

草稿中使用一个全局状态：

```c
01  uint8_t tcp_status = USTACK_TCP_STATUS_LISTEN;
02  uint32_t global_seqnum;
03  uint32_t global_acknum;
04  uint32_t server_isn = 12345;
```

- 第 1 行：服务端从监听状态开始。
- 第 2 行：保存收到报文的 SEQ，转换为主机字节序后便于加减。
- 第 3 行：保存收到报文的 ACK。
- 第 4 行：教学代码使用固定初始序列号；真实实现应生成合适的 ISN。

这些字段实际上都属于一条 TCP 连接。支持多个客户端时，应把它们放入连接控制块：

```c
struct tcp_control_block {
    uint32_t local_ip;
    uint32_t remote_ip;
    uint16_t local_port;
    uint16_t remote_port;
    uint32_t snd_nxt;
    uint32_t rcv_nxt;
    ustack_tcp_status_t state;
};
```

前四项构成连接四元组，收到 TCP 包后用四元组查找对应状态，而不是共享一个全局 `tcp_status`。

### 22. 收到 TCP 包并读取关键字段

```c
01  } else if (iphdr->next_proto_id == IPPROTO_TCP) {
02      struct rte_tcp_hdr *tcphdr =
03          (struct rte_tcp_hdr *)(iphdr + 1);
04
05      global_sip = iphdr->dst_addr;
06      global_dip = iphdr->src_addr;
07      global_sport = tcphdr->dst_port;
08      global_dport = tcphdr->src_port;
09
10      uint8_t flags = tcphdr->tcp_flags;
11      uint32_t peer_seq =
12          rte_be_to_cpu_32(tcphdr->sent_seq);
13      uint32_t peer_ack =
14          rte_be_to_cpu_32(tcphdr->recv_ack);
```

- 第 1 行：进入 TCP 分支。
- 第 2～3 行：教学版本仍假设 IPv4 Header 固定为 20 字节。
- 第 5～8 行：对调 IP 和端口，准备构造服务端响应。
- 第 10 行：TCP Flags 只有一个字节，不需要字节序转换。
- 第 11～12 行：SEQ 是 32 位字段，转换为主机字节序后保存。
- 第 13～14 行：ACK 同样转换为主机字节序。

TCP 响应同样需要保存 MAC 方向：

```c
rte_memcpy(global_smac,
           ethhdr->dst_addr.addr_bytes,
           RTE_ETHER_ADDR_LEN);
rte_memcpy(global_dmac,
           ethhdr->src_addr.addr_bytes,
           RTE_ETHER_ADDR_LEN);
```

第一段把收到包的目的 MAC 保存为响应源 MAC；第二段把收到包的源 MAC 保存为响应目的 MAC。

### 23. LISTEN 状态收到 SYN

```c
15      if ((flags & RTE_TCP_SYN_FLAG) != 0 &&
16          tcp_status == USTACK_TCP_STATUS_LISTEN) {
17
18          uint32_t ack_to_peer = peer_seq + 1;
19          send_syn_ack(server_isn, ack_to_peer);
20          tcp_status = USTACK_TCP_STATUS_SYN_RCVD;
21      }
```

- 第 15 行：使用按位与检查 SYN 标志是否存在，因为一个 TCP 包可以同时带多个 Flags。
- 第 16 行：只有监听状态才把这个 SYN 当作新连接请求。
- 第 18 行：SYN 占用一个序列号，因此响应 ACK 为客户端 SEQ+1。
- 第 19 行：构造并发送 `SYN | ACK` 报文。
- 第 20 行：发送后进入 `SYN_RCVD`，等待三次握手的最后一个 ACK。

学习版还应继续改进：如果 SYN+ACK 实际没有成功进入 TX Queue，不能直接认为它已经发送；收到重复 SYN 时也应考虑重发 SYN+ACK。

### 24. 逐行构造 SYN+ACK

Ethernet 和 IPv4 Header 的写法与 UDP 响应相同，只需把 IP 上层协议及总长度改为 TCP：

```c
01  uint16_t tcp_len = sizeof(struct rte_tcp_hdr);
02  uint16_t ip_len = sizeof(struct rte_ipv4_hdr) + tcp_len;
03  uint16_t frame_len = sizeof(struct rte_ether_hdr) + ip_len;
04
05  ip->total_length = rte_cpu_to_be_16(ip_len);
06  ip->next_proto_id = IPPROTO_TCP;
07  ip->hdr_checksum = 0;
08  ip->hdr_checksum = rte_ipv4_cksum(ip);
09
10  struct rte_tcp_hdr *tcp =
11      (struct rte_tcp_hdr *)(ip + 1);
12  *tcp = (struct rte_tcp_hdr){0};
13
14  tcp->src_port = global_sport;
15  tcp->dst_port = global_dport;
16  tcp->sent_seq = rte_cpu_to_be_32(server_isn);
17  tcp->recv_ack = rte_cpu_to_be_32(peer_seq + 1);
18  tcp->data_off =
19      (sizeof(struct rte_tcp_hdr) / sizeof(uint32_t)) << 4;
20  tcp->tcp_flags = RTE_TCP_SYN_FLAG | RTE_TCP_ACK_FLAG;
21  tcp->rx_win = rte_cpu_to_be_16(TCP_INIT_WINDOW);
22  tcp->cksum = 0;
23  tcp->cksum = rte_ipv4_udptcp_cksum(ip, tcp);
```

- 第 1 行：SYN+ACK 不带 payload，本例 TCP 段只有基础 TCP Header。
- 第 2～3 行：依次得到 IPv4 总长度与完整帧长度，用于 Header 和 mbuf。
- 第 5 行：IPv4 总长度包含 IPv4 Header 与 TCP Header，不含 Ethernet Header。
- 第 6 行：把 IPv4 上层协议改为 TCP。
- 第 7～8 行：写完影响 IPv4 Header 的字段后重新计算 Header checksum。
- 第 10～11 行：TCP Header 紧跟在基础 IPv4 Header 后面。
- 第 12 行：清零整个 TCP Header，避免窗口、紧急指针等字段含有随机值。
- 第 14～15 行：写入对调后的端口。
- 第 16 行：服务端第一次发送 SYN，SEQ 等于本端 ISN。
- 第 17 行：确认客户端的 SYN，所以 ACK 等于客户端 SEQ+1。
- 第 18～19 行：TCP Header 长度以 32 位字为单位，并放在 `data_off` 的高 4 位。
- 第 20 行：同时置位 SYN 与 ACK。
- 第 21 行：窗口是 16 位字段，要转换为网络字节序。
- 第 22～23 行：清零后计算覆盖 IPv4 伪首部和 TCP 段的 checksum。

完成 Header 后，仍要像 UDP 一样分配/扩展 mbuf、调用 `rte_eth_tx_burst()` 并处理未发送的 mbuf。

### 25. 验证三次握手最后一个 ACK

```c
01      if ((flags & RTE_TCP_ACK_FLAG) != 0 &&
02          tcp_status == USTACK_TCP_STATUS_SYN_RCVD &&
03          peer_ack == server_isn + 1) {
04
05          tcp_status = USTACK_TCP_STATUS_ESTABLISHED;
06          printf("enter established\n");
07      }
```

- 第 1 行：检查 ACK 标志存在。
- 第 2 行：只有已经发送 SYN+ACK 的连接才能通过这个分支建立。
- 第 3 行：客户端必须确认服务端的 SYN；因为 SYN 占一个序列号，所以期望值是 `server_isn + 1`。
- 第 5 行：状态迁移到 `ESTABLISHED`。
- 第 6 行：打印日志，方便结合抓包确认状态变化时机。

草稿最初只检查“有 ACK 标志且当前是 SYN_RCVD”，没有验证 ACK 数值。补上第 3 行后，三次握手判断才更完整。

### 26. 计算 TCP payload，而不是只检查 PSH

```c
01      uint16_t ip_hdr_len =
02          (iphdr->version_ihl & 0x0f) * 4;
03      uint16_t tcp_hdr_len =
04          (tcphdr->data_off >> 4) * 4;
05      uint16_t ip_total_len =
06          rte_be_to_cpu_16(iphdr->total_length);
07
08      if (ip_total_len < ip_hdr_len + tcp_hdr_len) {
09          continue;
10      }
11
12      uint16_t payload_len =
13          ip_total_len - ip_hdr_len - tcp_hdr_len;
14      uint8_t *payload =
15          (uint8_t *)tcphdr + tcp_hdr_len;
16
17      if (tcp_status == USTACK_TCP_STATUS_ESTABLISHED &&
18          payload_len > 0) {
19          printf("tcp data: %.*s\n",
20                 (int)payload_len,
21                 (char *)payload);
22      }
```

- 第 1～2 行：从 IHL 低 4 位得到 IPv4 Header 的 32 位字数量，再乘 4 得到字节数。
- 第 3～4 行：从 `data_off` 高 4 位得到 TCP Header 的字节数，因此 TCP Options 也会被正确跳过。
- 第 5～6 行：读取 IPv4 Header 声明的整个 IP 包长度。
- 第 8～10 行：总长度不能比两个 Header 相加还小，否则减法会发生无符号下溢。
- 第 12～13 行：用 IP 总长度减去两个 Header，得到 TCP payload 长度。
- 第 14～15 行：根据真实 TCP Header 长度定位 payload，而不是固定写 `tcphdr + 1`。
- 第 17 行：只有已建立连接才把数据交给应用层。
- 第 18 行：以实际 payload 长度判断是否有数据，不依赖 PSH 标志。
- 第 19～21 行：使用带长度的格式打印，避免把网络数据误当作以 `\0` 结尾的字符串。

收到数据后还应该回复 ACK：

```c
ack_to_peer = peer_seq + payload_len;
```

如果该段带 SYN 或 FIN，还应为每个标志额外加 1。当前草稿尚未实现数据 ACK，因此还只是握手与数据解析原型。

### 27. RX mbuf 的最终释放位置

接收循环应为每个包建立唯一、清楚的释放路径。使用新 TX mbuf 构造响应时，可以在本轮处理结束后释放 RX mbuf：

```c
for (uint16_t i = 0; i < num_recvd; i++) {
    bool rx_forwarded_to_tx = false;

    /* 解析并处理 mbufs[i] */

    if (!rx_forwarded_to_tx) {
        rte_pktmbuf_free(mbufs[i]);
    }
}
```

当前 UDP/TCP 示例新建了 `tx_mbuf`，所以原 `mbufs[i]` 没有转交给 TX，应由接收循环释放。草稿中缺少这一步，会导致 RX mbuf 持续泄漏，最终耗尽 mempool。

如果以后改为原地改写并发送 RX mbuf，则发送成功后应把 `rx_forwarded_to_tx` 设为 `true`，防止重复释放。

---

## 十四、怎样验证实现是否正确

建议按层验证，不要一上来同时调 UDP 和 TCP。

### 阶段一：只验证 TX Queue

- 网口成功配置一个 TX Queue；
- `rte_eth_tx_burst()` 返回 1；
- 抓包能看到 DPDK 网卡发出的帧。

### 阶段二：验证 UDP Echo

- 请求和响应的 MAC、IP、Port 方向相反；
- Echo payload 与请求完全一致；
- IPv4 `total_length`、UDP `dgram_len` 正确；
- Wireshark 未报告 IPv4/UDP checksum 错误；
- 连续发送大量数据时，mempool 可用数量没有持续下降。

若开启了 TX checksum offload，抓包点可能位于网卡真正计算 checksum 之前，抓包工具会显示“校验和错误”；需要结合 offload 配置和抓包位置判断，不能只看这一条提示。

### 阶段三：验证 TCP 握手

- SYN 到达时，状态为 `LISTEN`；
- SYN+ACK 的 ACK 等于客户端 SEQ+1；
- 客户端最终 ACK 等于服务端 SEQ+1；
- 只有 ACK 校验通过后才进入 `ESTABLISHED`；
- 发送 payload 后，能正确计算 TCP Header 长度和 payload 长度。

常用观察点：

```text
源/目的 MAC
源/目的 IP
源/目的端口
EtherType / IP Protocol
IP total_length
UDP length 或 TCP data offset
TCP flags / seq / ack
checksum
rte_eth_tx_burst 返回值
```

---

## 十五、把实验抽象成 pktgen 发包工具

完成 UDP/TCP 编码后，可以把“回包程序”进一步抽象为一个简单的发包工具。命令行可以设计为：

```bash
./ustack-pktgen \
  --udp \
  --src-ip 192.168.1.10 \
  --dst-ip 192.168.1.20 \
  --src-port 10000 \
  --dst-port 20000 \
  --dst-mac 00:11:22:33:44:55 \
  --count 1000 \
  --payload "hello dpdk"
```

TCP 模式可改用 `--tcp`，并增加：

```text
--flags syn|ack|psh|fin|rst
--seq N
--ack N
--window N
```

程序结构可以拆成：

```text
参数解析
  ├── 公共参数：源/目的 MAC、IP、数量、长度
  ├── UDP 参数：源/目的端口、payload
  └── TCP 参数：flags、seq、ack、window
        ↓
统一 packet description
        ↓
encode_ether + encode_ipv4
        ↓
encode_udp 或 encode_tcp
        ↓
burst send + 统计
```

这个小工具的价值在于：

- 为协议栈、网关、防火墙等项目构造可重复的测试流量；
- 精确控制五元组、TCP Flags、SEQ/ACK 等字段；
- 练习 DPDK 多包批量构造和发送；
- 后续可增加速率限制、不同包长、随机地址、时延与丢包统计。

在简历中应写清“实现了哪些能力”和“测得什么结果”，不要泛写成“实现 TCP/IP 协议栈”。例如：

> 基于 DPDK 实现用户态 UDP Echo 与最小 TCP 服务端握手原型，完成 Ethernet/IPv4/UDP/TCP 报文构造、校验和计算、TCP 状态迁移及 mbuf 批量收发；进一步抽象可配置五元组与报文数量的流量生成工具，用于协议功能测试。

若有实测结果，可以继续补充包长、队列数、CPU 核数、吞吐量和丢包率。

---

## 十六、本篇回顾清单

读完代码后，应能不看资料回答这些问题：

1. 为什么配置了 RX Queue 仍不能发送数据？
2. 为什么把收到的 mbuf 原样送进 TX Queue 不构成 Echo？
3. UDP Echo 需要交换哪三层的哪些字段？
4. Ethernet FCS 与 IPv4/UDP/TCP checksum 分别由谁处理？
5. `ip->total_length`、`udp->dgram_len`、`mbuf->pkt_len` 各自包含什么？
6. 为什么 payload 复制长度不能直接使用 UDP `dgram_len`？
7. `rte_eth_tx_burst()` 返回 0 时，mbuf 应由谁释放？
8. SYN 和 FIN 为什么会各占用一个序列号？
9. 为什么 TCP Header 没有类似 UDP 的长度字段？
10. 为什么仅凭 PSH 标志不能判断 TCP 包一定携带数据？
11. 为什么一个全局 `tcp_status` 无法支持多个 TCP 连接？
12. 当前原型距离完整 TCP 协议栈还缺哪些关键能力？

---

## 十七、总结

本篇真正跨过的一步，是从“解析别人构造好的报文”走向“自己构造一个能被对端接受的报文”。

UDP Echo 说明了发包的基本方法：

```text
分配 mbuf → 填各层 Header → 复制 payload
→ 写网络字节序 → 计算 checksum → 交给 TX Queue
```

TCP 实验则进一步说明：协议栈不只是报文格式的集合，还必须维护状态和时序。能生成 SYN+ACK 只是起点；要成为完整 TCP 实现，还需要可靠传输、序列号校验、超时重传、窗口管理、连接表和连接关闭等机制。

从工程实践看，本次代码最值得记住的不是某个固定 API，而是四条原则：

1. 每一层的长度边界必须算清楚；
2. 所有多字节协议字段都要明确主机/网络字节序；
3. checksum 计算前先清零，并明确由软件还是硬件负责；
4. 任何 mbuf 都必须有清晰的所有权和释放路径。
