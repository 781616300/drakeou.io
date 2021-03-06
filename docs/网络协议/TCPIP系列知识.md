## TCP/IP系列知识，传输控制相关

这里不讨论TCP头信息里有什么，也不讨论三握四挥，仅总结TCP协议下数据传输控制相关的知识点。

### 前提须知，MTU和MSS

数据在TCP层封装好交给网络层，再到数据链路层，那么最终在数据链路层传输的数据包大小真的是TCP层封装好的大小么？

结果显然不是，数据链路层传输的帧大小是有限制的，不能把一个太大的包直接塞给链路层，这个限制被称为「最大传输单元（Maximum Transmission Unit, MTU）」

下图是以太网的帧格式，以太网的帧最小的帧是 64 字节，除去 14 字节头部和 4 字节 CRC 字段，有效荷载最小为 46 字节。最大的帧是 1518 字节，除去 14 字节头部和 4 字节 CRC，有效荷载最大为 1500，这个值就是以太网的 MTU。因此如果传输 100KB 的数据，至少需要 （100 * 1024 / 1500) = 69 个以太网帧。

![以太网帧](..\static\以太网帧.png)

> IP分段

IPv4 数据报的最大大小为 65535 字节，这已经远远超过了以太网的 MTU，而且有些网络还会开启巨帧（Jumbo Frame）能达到 9000 字节。 当一个 IP 数据包大于 MTU 时，IP 会把数据报文进行切割为多个小的片段(小于 MTU），使得这些小的报文可以通过链路层进行传输。

IP头中也有表示分段偏移的字段。也就是**当数据链路层限制了MTU时**，IP层会主动将数据进行分段传输，也就是在IP层**TCP提供的一个报文可能被再次切分**。而我们直到`IP层`是不可靠的，IP协议只管发送不确认任何接收。

所以TCP层的数据被IP层被动分片的坏处就是，看起来是一次TCP数据传输**实际是多次传输**，而**其中一个IP分段丢失就导致TCP协议无法确认接收**，也就导致**一个分段丢失TCP层无法感知，只能全部重发数据**。

> MSS, 	Max Segment Size

为了避免这种对于TCP报文的被动划分，TCP协议也支持根据整个传输环境中最小的带宽调整自己的报文长度。

**MSS = MTU - IP header头大小 - TCP 头大小**

这样IP分段给TCP带来的问题就解决了



### Nagle算法，和延迟确认

将Nagle算法和延迟确认一起说明是因为两者分别是对`发送/确认`操作进行延迟控制，都有减轻网络负担避免带宽阻塞的意义。

> Nagle算法的核心目的是减少发送端频繁的发送小包给对方

算法要求，当一个TCP连接中有再传数据时，所有小于MSS的数据都不可以被直接发送，直到接收到在传数据的ACK，才会将所有小包合并在一起进行发送。

下图是分5次发送10kb数据的两种情况：

![粘包发送](..\static\粘包发送.png)

可以看到开启Nagle之后，在第一个RTT内只有大于MSS的数据可以被直接发送，而小于MSS的包都被延迟到第一个RTT（接收到ACK之后）才发送。

其实Nagle算法的设计是处于带宽较小的考虑，频繁的小包可能占满带宽出现过多重传，合并在一起发送产生了部分延时但提高了传输效率。

> 延时确认

收到一个数据包之后如果自身没有要发送的数据包是理论上不需要立刻进行回复Ack的，它可以等待一段时间再进行确认（不超过对方认为包丢失的时间），如果还有数据包需要发送就可以一起发送。

这种设计的原因和Nagle一样，仅仅回复Ack而不在窗口中携带数据比较浪费，总结如下三点

- 如果接收端这个时候恰好有数据要回复客户端，那么 ACK 搭上顺风车一块发送。
- 如果期间又有客户端的数据传过来，那可以把多次 ACK 合并成一个立刻发送出去
- 如果一段时间没有顺风车，那么没办法，不能让接收端等太久，一个空包也得发。

### 滑动窗口

烦死了，没啥意思不写了

### 拥塞控制