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
+------------+---------+-------+------------+
| Size(byte) | 7..22   | 0..1  | 0..N       |
|            | FHDR    | FPort | FRMPayload |
+------------+---------+-------+------------+
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

在 LoRa 网络中，设备可以选择不同的 data rate 和 Tx power(Transmit Power, 传输功率) 进行数据的传输。我们当然希望设备的 data rate 越高越好，但实际情况下，data rate 越高，受到干扰的可能性也越大。所以我们会希望在环境好的地方，采用较高的 data rate, 在环境差的地方，采用较低的 data rate.

而 ADR 就是让 Network Server 来进行调度，为不同的设备设置合适的 data rate 和 Tx power, 从而达到优化传输速率的目的。

FCtrl 中的 ADR bit 就是一个控制开关，用来标记 ADR 功能的开启情况。

- 如果 uplink 里设置了 ADR bit, Network Server 随后会下发 MAC 指令来控制终端的 data rate 和 Tx power.
- 如果 uplink 里没有设置 ADR bit, Network Server 就不会试图去控制终端的 data rate 和 Tx power.
- 如果 downlink 里设置了 ADR bit, 则表明 Network Server 有能力设置终端的 data rate 和 Tx power. 终端可以选择是否在下次 uplink 的时候带上 ADR bit, 表明自己的意愿；
- 如果 downlink 里没有设置 ADR bit, 则表明 Network Server 因为一些原因无法评估出一个最好的 data rate. 这种情况下终端有两种选择：
    + 不在 uplink 里设置 ADR bit, 采用自己的策略来使用 data rate;
    + 在 uplink 里设置 ADR bit, 使用 data rate 衰减的方式来采用合适的 data rate;

终端默认采用的 Tx power 是在条件允许下的最大值，Network Server 可以通过 LinkADRReq Mac 指令来降低它。

如果终端的 data rate 经过调整被设置为比默认值更高，或者 Tx power 经过调整被设置为比默认值更低，这意味着受到干扰的可能性更大。那么终端需要周期性地确认 Network Server 仍然可以收到 uplink frame.  其确认方法是：
- 终端会在每次 uplink frame counter 递增后，递增 ADR_ACK_CNT 计数器；
- 当 ADR_ACK_CNT 计数器达到 ADR_ACT_LIMIT 的限制却仍然没有收到任何 downlink 回复，它就会在 uplink 中设置  ADRACKReq bit；
- 发送 ADRACKReq 之后，终端会进行检查，如果在接下来的 ADR_ACK_DELAY 个 uplink frame 之内收到了 downlink, 就认为干扰没那么严重，此时终端的 ADR_ACK_CNT 计数器会被重置；
- 如果在 ADR_ACK_DELAY 个 uplink frame 之内没有回应，终端就会尝试把 Tx Power 回复到默认值(最高)，并且逐渐降低自己的 data rate 直到最低；

如果终端的 Tx Power 和 data rate 已经是默认值了，那么 ADRACKReq bit 就无需设置，因为已经没办法改善了。

#### Message acknowledge bit (ACK in FCtrl)

FCtrl 里还有一个 ACK 字段，用于处理 comfirmed data 消息。接收方应该在收到这类消息后，返回一个带有 ACK bit 的 data frame.
- 如果发送方是终端，那么**网关** 应该在随后的接收窗口中回复 ACK;
- 如果发送方是网关，终端可以自由决定回复 ACK 的时机——终端可以立即回复一条空消息，也可以把 ACK 带在下一次 uplink frame 里；

#### Retransmission procedure 重传过程

**Downlink Frames**: 
downlink 的 confirmed 或 unconfirmed frame 都不应该使用同样的 frame counter 值进行重传。unconfirmed 自不必说，对于 confirmed downlink 来说，如果发送后没有收到 ACK，那么应该通知 Application Server, 由它决定是否重新发送一个新的 comfirmed frame.

**Uplink Frames**:

uplink 的 confirmed & unconfirmed frame 会被发送 "NbTrans" 次，除非在发送过程中收到了符合条件的 downlink frame.

"NbTrans" 可以交给 Network Manager 来配置，从而通过冗余发送的方式来获取指定的 Quality of Service.

对于终端来说，发送 confirmed frame 后，一旦收到带有 ACK 的 downlink frame, 就应当停止重传；

Class B&C 设备发送 unconfirmed frame 后，一旦在 RX1 窗口中收到 单播 downlink message, 就应当停止重传；

Class A 设备发送 unconfirmed frame 后，一旦在 RX1 或 RX2 窗口中收到合法的 downlink message, 就应当停止重传；

如果 Network Server 收到了超过 "NbTrans" 次相同的 uplink frame, 这意味着可能出现了重放攻击，或者设备坏了。此时 Network Server 不应该处理多余的 frame.

#### Frame pending bit(FPending in FCtrl, downlink only)

FPending 标记位只存在于 downlink 中，意思是 Network Server 还有更多数据要发送，请求终端继续发送 uplink 以便提供更多接收窗口。

#### Frame Counter (FCnt)

每个终端都对应了三个 frame counter：
- FCntUp 追踪发送给 Network Server 的 uplink data frame 数量；
- FCntDown 追踪 Network Server 发送的 downlink data frame 数量(有两个)；

对于 downlink 方向，有两种不同的计数方案：
- LoRaWAN 1.0 的单计数器方案，所有 port 都共享同一个 downlink 计数器 FCntDown;
- LoRaWAN 1.1 的双计数器方案，有一个单独的 NFCntDown 计数器用于计数 MAC 通信；另一个 AFCntDown 用于计数其它通信；

对于双计数器方案，NFCntDown 由 NetworkServer 管理，AFCntDown 由 Application Server 管理。

一旦一个 OTAA 设备成功处理了 Join-accept 消息，那么终端的 FCntUp 及网络上的 NFCntDown 和 AFCntDown 都会被重置为 0；

ABP 设备的 Frame Counter 会在制造时被初始化为 0. ABP 设备的 frame counter 在出厂后就无法更改，即使是换电池也要保证计数器不会改变；

每次 uplink, FCntUp 递增；每次发生在 FPort 0 或者 FPort 没有设定(意味着在下发 MAC 指令)，NFCntDown 递增；每次发生在 FPort 不是 0 时，AFCntDown 递增。

在接收端，相应的计数器应当与接收到的值保持同步。假如收到的值与当前计数器的值一致，并且消息的 MIC 与本地通过 network session key 计算出来的 MIC 一致，那么就进行同步。

FCnt 不会在 confirmed / unconfirmed frame 的重复发送过程中重复递增。Network Server 应当丢弃重复的 application payload，只把一个 frame 传给 application server.

Frame Counters 都有 32bit 宽度。FCnt 与 frame counter 的低 16 位保持一致。

终端不应该在同一应用或 network session keys 下重用同一个 FCntUp, 除非在重传的时候。

终端不应该处理相同 downlink frame 重传数据。

#### Frame Options(FOptsLen in FCtrl, FOpts)

FCtrl 中的 FOptsLen 字段用以注明 FOpts 的实际长度。

FOpts 意为 Frame Options, 它最多可以携带 15 字节的 MAC 指令。

如果 FOptsLen 不是 0, 就不能使用 port 0(FPort 要么不是 0, 要么不设置)。这意味着 MAC 指令要么存在于 FOpts 中，要么存在于 payload 中。

如果 FHDR 中携带了 FOpts, 那么 FOpts 必须在 MIC 计算前被加密。

加密方法基于 IEEE 802.15.4/2006 Annex B [IEEE802154] 中所描述的 AES 算法，其 key 的长度为 128 bit.

跟想象中不同，并不是直接加密 FOpts, 而是先加密如下 block:

```
+------------+------+----------+-----+---------+---------------------+------+------+
| Size(Byte) | 1    | 4        | 1   | 4       | 4                   | 1    | 1    |
|            | 0x01 | 4 * 0x00 | Dir | DevAddr | FCntUp or NFCntDown | 0x00 | 0x00 |
+------------+------+----------+-----+---------+---------------------+------+------+
```

上述 block 刚好是 16 个字节(128 bit), 正是 AES 加密算法中一个 block 的大小。我们对其进行加密，得到一个 S:

```
S = aes_encrypt(key, block);
```

加密过程中用到的 key 是 NwkSEncKey, 暂时先别管它是怎么来的。加密得到 S 后，再利用它与 FOpts 进行位运算：

> Encryption and decryption of the FOpts is done by truncating (pld | pad16) xor S to the first len(pld) octets.

原文看起来有点难懂，好像是把 FOpts 补齐到 16 字节，然后与 S 进行异或运算，最后再截取 FOptsLen 长度的字节。

#### Port Field(FPort)

如果 Frame Payload 字段不为空，则必须填写 FPort 字段。
- 如果 FPort 为 0, 则表示 FRMPayload 中只有 MAC 指令；
- 如果 FPort 为 1..223(0x01 .. 0xDF) 则表示 FRMPayload 中包含的是 Application 数据；
- 如果 FPort 为 224, 它专门用于 LoRaWAN MAC 层的测试。
- 如果 FPort 为 225..255, 则它们是保留字段；


#### MAC Frame Payload Encryption (FRMPayload) 

如果 data frame 携带了 payload, 则必须在计算 MIC 之前加密 FRMPayload。

加密算法由 IEEE 802.15.4/2006 Annex B [IEEE802154] 进行描述，使用 128 bit 长度秘钥的 AES 加密算法。

加密是所使用的 Key 取决于 FPort 的取值：
- FPort 为 0: 使用 NwkSEncKey;
- FPort 为 1..255: 使用 AppSKey;

加密 FRMPayload 时，并不是直接用 AES 算法进行加密，而是先生成 K 个 block, K = ceil(len(FRMPayload)/16), 每个 block 都是如下内容：

```
+------------+------+----------+-----+---------+---------------------+------+------+
| Size(Byte) | 1    | 4        | 1   | 4       | 4                   | 1    | 1    |
|            | 0x01 | 4 * 0x00 | Dir | DevAddr | FCntUp or NFCntDown | 0x00 | i    |
+------------+------+----------+-----+---------+---------------------+------+------+
```

i 的取值为 1..k.

接着，使用秘钥对每个 block 进行加密，加密得到的仍然是一个 16byte 的 block, 称为 Si:

Si = aes128_encrypt(key, blocks[i])

接着把所有 Si 拼接起来成为一个 S，然后再把 FRMPayload 补足成 16 字节的倍数，与 S 进行异或操作，最后把结果截取成为跟 FRMPayload 大小一致；

### Message Integrity Code (MIC)

MIC 码用于消息的完整性校验，这里的消息包括如下字段：

msg = MHDR | FHDR | FPort | FRMPayload

#### downlink 的 MIC

MIC 的计算方法为：

cmac = aes128_cmac(SNwkSIntKey, B0 | msg)

MIC = cmac[0..3]

这里的 CMAC(Cipher-based Message Authentication Code), 是一种类似于 Hash 的技术，其目的是确保发送方发来的数据没有经过篡改。但它是基于密码的，所以签名本身可以明文传递。

aes128_cmac 是一个计算签名的算法，它将返回一个经过签名的 16 byte 串。MIC 则是取这个签名的前 4 字节。

上述公式中的 B0 内容如下：

```
+-------------+------+----------+----------+------------+---------+-------------+------+----------+
| Size(bytes) | 1    | 2        | 2        | 1          | 4       | 4           | 1    | 1        |
|             | 0x49 | ConfFCnt | 2 * 0x00 | Dir = 0x01 | DevAddr | AFCntDwn or | 0x00 | len(msg) |
|             |      |          |          |            |         | NFCntDwn    |      |          |
+-------------+------+----------+----------+------------+---------+-------------+------+----------+
```

如果 downlink 的 ACK bit 被设置，说明这是在回复 comfirmed uplink frame. 这种情况下需要设置 ConfFCnt, ConfFCnt 是 confirmed uplink frame 被承认的 counter 对 2^16 取模的结果。

#### uplink 的 MIC

首先，定义 B0 block:

```
+-------------+------+--------+------------+---------+--------+------+----------+
| Size(bytes) | 1    | 4      | 1          | 4       | 4      | 1    | 1        |
| B0          | 0x49 | 0x0000 | Dir = 0x00 | DevAddr | FCntUp | 0x00 | len(msg) |
+-------------+------+--------+------------+---------+--------+------+----------+
```

接着，定义 B1 block:

```
+-------------+----------+------+------+-----------+---------+--------+------+----------+
| Size(bytes) | 2        | 1    | 1    | 1         | 4       | 4      | 1    | 1        |
| B1          | ConfFCnt | TxDr | TxCh | Dr = 0x00 | DevAddr | FCntUp | 0x00 | len(msg) |
+-------------+----------+------+------+-----------+---------+--------+------+----------+
```

上述内容中：
- TxDr 是此次 uplink 所使用的 data rate
- TxCh 是此次 uplink 所使用的 channel;
- 如果 uplink 的 ACK bit 被设置，说明之前收到的 comfirmed downlink frame 被承认，此时 ConfFCnt 是终端上承认 comfirmed downlink frame 计数器对 2 ^ 16 取余的结果。其它情况下 ConfFCnt 都是 0x0000

接着，使用 CMAC 算法进行计算；

cmacS = aes128_cmac(SNwkSIntKey, B1 | msg)
cmacF = aes128_cmac(FNwkSIntKey, B0 | msg)

如果是 LoRaWAN 1.0 协议，则 MIC = cmacF[0..3];

如果是 LoRaWAN 1.1 协议，则 MIC = cmacS[0..1]cmacF[0..1]


