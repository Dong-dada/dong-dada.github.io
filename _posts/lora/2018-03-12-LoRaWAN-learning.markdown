---
layout: post
title:  "LoRaWAN 协议学习"
date:   2018-03-12 12:03:30 +0800
categories: lora
---

* TOC
{:toc}

## 简介

LoRaWAN(Long Range WAN) 是一种低功耗广域网络。

LoRaWAN 是一种典型的星型网络： **end-device** 通过 **gateways** 的中继与 **Network Server** 进行通信，随后 **Network Server** 将每个设备发来的 packet 分发到关联的 **Application Server** 上。

LoRaWAN 使用对称加密技术来保证无线传输过程(end-device 和 gateways 之间)的安全。在这期间需要一对 session keys, 这个对称密钥通过 device's root key 来生成。root key 的存储及对称密钥的生成过程，由 **Join Server** 来保证。

Gateway 和 **Network Server** 之间通过 IP 协议连接；节点使用单跳 LoRa 或者 FSK 与一个或多个 gateway 进行通信；

终端与基站之间的通信，可以使用多种不同的 frequency channels(频道) 和 data rates(数据传输速率). data rate 的选择取决于通信范围和耗时，不同的 data rate 之间不会互相干涉。LoRa data rate 的范围大约是 0.3kbps 到 50 kbps. 为了节省终端电量及最大化网络容量，LoRa 网络可以通过 adaptive data rate(ADR) 技术来管理每个终端各自的 data rate 和 RF 输出。

终端可以在任意时间任何频道中使用任意可用的 data rate 来进行通信，但必须遵守以下规则：
- 每次通信，终端都应该以随机的方式来改变频道，这样产生的具有多样性的频率能够跟好地面对干扰的情况；
- 终端所能使用的最大传输占空比(maximum transmit duty cycle) 取决于它所处的子频带以及当地法规；
- 终端所能使用的最大传输时间(maximum transmit duration) 取决于它所处的子频带及当地法规；

所有的 LoRaWAN 设备必须实现 Class A, 此外它们可以选择是否实现 Class B 或 Class C. 后两者与 Class A 要保持兼容：
- Class A: 每当终端向基站发送上行数据，随后就会开启两个接收窗口。这意味着基站没办法随时向终端发送下行数据，除非终端主动联系。
- Class B: 终端会每隔一段时间开启接收窗口。间隔时间需要通过基站下发的 Beacon 来确定，不然基站无法确定发送的时机。
- Class C: 终端始终能够接受下行数据，除了正在上行的时候。

## Physical Message Formats

上行消息：

```
+----------+------+----------+------------+-----+
| Preamble | PHDR | PHDR_CRC | PHYPayload | CRC |
+----------+------+----------+------------+-----+
```

下行消息

```
+----------+------+----------+------------+
| Preamble | PHDR | PHDR_CRC | PHYPayload |
+----------+------+----------+------------+
```

### 接收窗口

![]( {{site.url}}/asset/lora-class-a-receive-window.png )

第一个接收窗口 RX1 所使用的 data rate 取决于之前上行时所使用的 data rate. 这个映射关系由 regional parameters 文档进行描述，不同地区的映射关系并不相同。

第二个接收窗口 RX2 使用固定配置的频率和 data rate 来接收。这一配置可以通过 MAC 指令来修改。默认的频率和 data rate 由 regional parameters 文档进行描述。

如果在接收窗口中收到了 Preamable(报头)，那么接收窗口必须保持打开，直到下行 frame 接收完毕。如果在第一个接收窗口中收到了 frame, 并且 address 和 MIC 检查通过，该 frame 被应用于此终端，那么终端不应该再打开第二个接收窗口。

如果网络想要发送下行数据给终端，它必须在两个接收窗口的开始处进行下行数据的发送。如果下行数据希望在两个窗口中都进行发送，那么同一 frame 不能跨越两个接收窗口。

终端不应该在两个接收窗口关闭前发送另一个上行数据，否则第二个接收窗口将会过期。


## MAC Message Formats

MAC 消息保存在 PHY 层的 PHYPayload 中。其结构如下：

普通消息(上行或下行)：

```
+------+------------+-----+
| MHDR | MACPayload | MIC |
+------+------------+-----+
```

入网消息(上行)：

```
+------+--------------------------------+-----+
| MHDR | Join-Request or Rejoin-Request | MIC |
+------+--------------------------------+-----+
```

入网回应(下行)：

```
+------+-------------+
| MHDR | Join-Accept |
+------+-------------+
```

上述字段中, MHDR 即 MAC Header, 其结构如下：

```
+-------+------+-------+
| MType | RFU  | Major |
+-------+------+-------+
```

- MType: 消息类型;
    + Join-request
    + Join-accept
    + Unconfirmed Data Up
    + Unconfirmed Data Down
    + Confirmed Data Up
    + Confirmed Data Down
    + Rejoin-request
    + Proprietary
- RFU: 保留位
- Major: 指定 frame 的编码格式所属的 LoRaWAN 协议版本；

### Frame (MACPayload)

如果 MType 是 Data Up/Down 类型，那么这条消息是普通消息，MHDR 后面会跟着 MACPayload 字段, 其结构如下：

```
+---------+-------+------------+
| FHDR    | FPort | FRMPayload |
+---------+-------+------------+
```

FHDR 即 Frame Header, 其结构如下：

```
+-------------+---------+-------+------+---------+
| Size(bytes) | 4       | 1     | 2    | 0..15   |
|             | DevAddr | FCtrl | FCnt | FOpts   |
+-------------+---------+-------+------+---------+
```

- DevAddr: 终端设备地址；
- FCtrl: frame control;
- FCnt: frame counter;
- FOpts: frame options 用于传输 Mac 指令，应该用 NwkSEncKey 进行加密；

FCtrl 在上行和下行时有所不同，对于 downlink, 其结构如下：

```
+------+-----+-----+-----+----------+----------+
| Bit# | 7   | 6   | 5   | 4        | [3..0]   |
|      | ADR | RFU | ACK | FPending | FOptsLen |
+------+-----+-----+-----+----------+----------+
```

对于 uplink, 其结构如下：

```
+------+-----+-----------+-----+--------+----------+
| Bit# | 7   | 6         | 5   | 4      | [3..0]   |
|      | ADR | ADRACKReq | ACK | ClassB | FOptsLen |
+------+-----+-----------+-----+--------+----------+
```

#### Adaptive Data Rate

在 LoRa 网络中，设备可以选择不同的 data rate 和 Tx power 进行数据的传输。所谓的 ADR 就是让 Network Server 来进行调度，为不同的设备设置合适的 data rate 和 Tx power, 从而达到优化传输速率的目的。

FCtrl 中的 ADR bit 就是一个控制开关。用来标记 ADR 功能的开启情况。

- 如果 uplink 里设置了 ADR bit, 则随后 Network Server 应该通过合适的 MAC 指令来设置该终端 data rate 和 Tx power
- 如果 uplink 没有设置 ADR bit，则 Network Server 不应该对终端的 data rate 和 Tx power 进行设置；
- 如果 downlink 里设置了 ADR bit, 则表示 Network Server 希望发送 ADR 指令来设置 data rate, 终端可以选择是否设置 uplink 中的 ADR bit;
- 如果 downlink 没有设置 ADR bit, 表示因为某些原因，Network Server 无法设置合适的 data rate 和 Tx power, 这时终端仍然可以选择是否需要设置 data rate;