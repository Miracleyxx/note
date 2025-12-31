
---

# 📝 RTP (Real-time Transport Protocol) 技术规范文档

> 💡 **技术摘要**：RTP (RFC 3550) 是构建在 UDP 之上的应用层协议。其核心机制是通过 **Sequence Number (16-bit)** 解决乱序，通过 **Timestamp (32-bit)** 解决抖动与同步。协议本身**不保证** QoS（丢包/重传），需配合 RTCP 或应用层 FEC/NACK 机制使用。

## ⚙️ 协议架构与工作流

RTP 位于传输层（UDP）之上，应用层之下，核心处理流程如下：

1. **分片 (Fragmentation)**：将音视频帧（如 H.264 NALU）分割为适应 MTU 的 Payload。
    
2. **封装 (Encapsulation)**：添加 RTP 头部，注入时序与同步元数据。
    
3. **传输 (Transport)**：通过 UDP Socket 发送（低延迟，不可靠）。
    
4. **去抖与重排 (De-jitter & Re-ordering)**：接收端利用 Jitter Buffer，根据 `Seq` 重排乱序包，根据 `TS` 依序解码播放。
    

## 🧬 数据包位级结构

RTP 头部固定长度为 **12 字节**（不含 CSRC 和扩展头）。

```Plaintext
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                           Timestamp                           |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |           Synchronization Source (SSRC) identifier            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |            Contributing Source (CSRC) identifiers             |
 |                             ....                              |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 字段定义表 (Field Specification)

|**字段**|**位宽 (Bits)**|**符号**|**定义与工程逻辑**|
|---|---|---|---|
|**Version**|2|`V`|协议版本。当前标准固定为 `10` (Binary) 即 **2**。|
|**Padding**|1|`P`|填充位。若置 `1`，Payload 末尾包含填充字节（用于加密或块对齐），最后一个字节记录填充长度。|
|**Extension**|1|`X`|扩展位。若置 `1`，固定头后紧跟扩展头（通常用于携带自定义元数据）。|
|**CSRC Count**|4|`CC`|CSRC 计数器。表示头部后跟随的 CSRC 标识符数量（0~15），用于混音场景。|
|**Marker**|1|`M`|标记位。**关键控制位**。视频中通常标识一帧的结束（Last Packet of Frame）；音频中标识谈话突发开始。|
|**Payload Type**|7|`PT`|负载类型。映射编码格式：<br><br>  <br><br>• **静态映射**：`0`=PCMU, `8`=PCMA<br><br>  <br><br>• **动态映射 (96-127)**：需经 SDP 协商，如 `96`=H.264, `97`=H.265。|
|**Sequence Number**|16|`Seq`|序列号。初始值为随机数，每包 `+1`。用于检测丢包和重排。<br><br>  <br><br>⚠️ **溢出处理**：值达 $65535$ 后回绕至 $0$。|
|**Timestamp**|32|`TS`|时间戳。反映采样时刻。单位取决于时钟频率（如视频 90kHz）。<br><br>  <br><br>⚠️ **逻辑**：同一帧的所有分片包使用**相同**的时间戳。|
|**SSRC**|32|`SSRC`|同步源标识。随机生成，会话内唯一。用于区分不同的流（如区分不同的参会者）。|
|**CSRC**|32 × N|`CSRC`|贡献源标识。仅在混流（Mixer）场景出现，列出所有参与混音的原始 SSRC。|

## 💻 C++ 实现

在工程实现中，需严格处理**网络字节序 (Big-Endian)** 与**位域 (Bit-field)** 的内存布局。

### 1. 数据结构定义

```C++
/* ⚠️ 注意：位域顺序依赖于编译器实现（通常小端机低位在前），
   网络传输需序列化处理，此处为逻辑结构示意 */
struct RTPHeader {
#ifdef IS_LITTLE_ENDIAN
    uint8_t csrc_count:4;   // CC
    uint8_t extension:1;    // X
    uint8_t padding:1;      // P
    uint8_t version:2;      // V
    
    uint8_t payload_type:7; // PT
    uint8_t marker:1;       // M
#else
    uint8_t version:2;
    uint8_t padding:1;
    uint8_t extension:1;
    uint8_t csrc_count:4;
    
    uint8_t marker:1;
    uint8_t payload_type:7;
#endif
    uint16_t sequence_number;
    uint32_t timestamp;
    uint32_t ssrc;
    // CSRC 列表和 Payload 紧随其后，通常使用变长指针处理
};
```

### 2. 包构建逻辑 (Builder)

```C++
RTPPacket createRTPPacket(uint16_t seq, uint32_t ts, uint32_t ssrc, uint8_t pt, bool mark, uint8_t* data, size_t len) {
    RTPPacket pkt;
    // 1. 设置头部控制位
    pkt.header.version = 2;
    pkt.header.padding = 0;
    pkt.header.extension = 0;
    pkt.header.csrc_count = 0;
    
    // 2. 关键元数据注入
    pkt.header.marker = mark ? 1 : 0; 
    pkt.header.payload_type = pt;
    
    // 3. 字节序转换 (Host to Network)
    pkt.header.sequence_number = htons(seq); 
    pkt.header.timestamp = htonl(ts);
    pkt.header.ssrc = htonl(ssrc);
    
    // 4. 负载挂载
    pkt.payload = data; 
    pkt.payload_len = len;
    
    return pkt;
}
```

## ⚖️ 版本演进 

_(仅做简要技术对比，V1 已淘汰)_

|**版本**|**关键差异**|**状态**|
|---|---|---|
|**RTP v1**|缺少扩展位 (X) 与 CSRC 计数，灵活性低。|❌ Deprecated (已废弃)|
|**RTP v2**|增加 `Extension` 机制，支持复杂拓扑（Mixer/Translator），标准化 PT 定义。|✅ **Current Standard (现行标准)**|

## 🚀 开发者备忘

> **关键工程考量**：RTP 只是载体，真正的难点在于接收端的 Jitter Buffer 策略和发送端的 Pacing（平滑发送）。

- ✅ **Sequence Number 必须处理回绕**：在计算丢包率时，需处理 `65535` -> `0` 的跳变逻辑。
    
- ✅ **Timestamp 不是系统时间**：它是采样时间。对于 90kHz 的视频时钟，每秒 TS 增加 90,000。
    
- ✅ **MTU 限制**：构建 RTP 包时，Payload 大小 + 12字节头 + UDP/IP 头 < MTU (通常 1500)，否则会导致 IP 层分片，严重影响传输效率。
    

---