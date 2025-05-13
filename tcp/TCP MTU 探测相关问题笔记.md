### **TCP MTU 探测相关问题笔记**

不同网络环境下，MTU 可能不同：

| 网络类型        | MTU 大小（字节） |
| --------------- | ---------------- |
| 以太网          | 1500             |
| Wi-Fi（802.11） | 2304             |
| PPPoE（DSL）    | 1492             |
| VPN (IPSec)     | 1400~1420        |
| IPv6（隧道）    | 1280+            |
| Jumbo Frame     | 9000             |

🔹 **固定 MTU 可能导致数据包过大而丢失。**

#### **1. MTU 探测的作用**

MTU（Maximum Transmission Unit）探测用于 **动态发现路径上的最小 MTU**，确保 TCP 发送的数据包不会因为路径 MTU 过小而导致丢包或性能下降。其主要作用包括：

- **避免 IP 分片**：IP 分片会增加重组开销，并可能被防火墙丢弃。
- **提高网络传输效率**：使用适当的 MSS（Maximum Segment Size），减少重传和丢包。
- **解决 TCP 黑洞问题**：部分网络会丢弃超出 MTU 限制的数据包，并屏蔽 ICMP 消息，导致 TCP 连接受阻。

------

#### **2. MTU 大小的设置方式**

MTU 大小的计算主要依赖以下几个因素：

- **协商的 MSS**：TCP 三次握手时，双方会协商 MSS（通常基于链路 MTU 减去 IP 和 TCP 头部长度）。
- **系统默认 MTU 设置**：不同网络接口的默认 MTU 可能不同（如以太网通常为 1500 字节）。
- **动态 MTU 探测（PMTUD / PLPMTUD）**：TCP 可以通过 ICMP 消息或主动探测方法调整 MSS，以适应不同网络路径上的 MTU 限制。

在 `tcp_mtup_init()` 函数中，MTU 相关参数的初始化过程如下：

1. **是否启用 MTU 探测**：由 `net.ipv4.sysctl_tcp_mtu_probing` 参数控制，值大于 1 时启用。

2. 最大可用 MTU 计算：

    ```c
    icsk->icsk_mtup.search_high = tp->rx_opt.mss_clamp + sizeof(struct tcphdr) + icsk->icsk_af_ops->net_header_len;
    ```

    其中，

    ```bash
    tp->rx_opt.mss_clamp
    ```

     代表协商的 MSS，

    ```bash
    net_header_len
    ```

     代表 IP 头部长度（IPv4/IPv6）。

3. 最小 MTU 计算

    ：

    ```c
    icsk->icsk_mtup.search_low = tcp_mss_to_mtu(sk, READ_ONCE(net->ipv4.sysctl_tcp_base_mss));
    ```

    ```bash
    sysctl_tcp_base_mss
    ```

     作为最低可用 MSS，确保不会设置过低的 MTU。

------

#### **3. TCP MTU 探测发生的时机**

TCP MTU 探测主要发生在以下情况下：

- **TCP 连接建立后**（启用 `tcp_mtu_probing=1` 或 `2`）。
- **发送数据时发现丢包，可能由于路径 MTU 过小导致**。
- **PMTUD（Path MTU Discovery）收到 ICMP “Fragmentation Needed” 消息**，TCP 发送方调整 MSS 重新发送数据。
- **PLPMTUD（Packetization Layer Path MTU Discovery）主动探测 MTU**，无需依赖 ICMP 消息，通过逐步增加数据包大小进行探测。

------

#### **4. TCP 协商的 MSS 是否会被 MTU 探测覆盖？**

- **默认情况下，TCP 遵循三次握手时协商的 MSS**，即由双方根据链路 MTU 计算出的 MSS 进行数据传输。

- **如果路径上的 MTU 小于协商的 MSS，TCP 可能会遇到丢包问题**，此时如果启用了 **MTU 探测**，则 TCP 可以调整 MSS 以适应实际路径 MTU。

- 如果未启用 MTU 探测

    ，但路径 MTU 小于 MSS，可能会导致：

    - **IP 分片（IPv4 环境）**，但可能增加负担或被丢弃。
    - **依赖 ICMP “需要分片” 消息调整 MSS**，但若 ICMP 被防火墙拦截，则可能导致 TCP 黑洞问题。
    - **数据包被丢弃，TCP 重传超时，连接受影响**。

✅ **最佳实践**：建议启用 `net.ipv4.tcp_mtu_probing=2`，确保 TCP 适应不同路径上的 MTU 限制，提高传输效率并避免 TCP 黑洞问题。