

---

# RTCP 协议详解：统计、控制与抗丢包

## 1. RTCP 的核心作用(**基础统计 (RFC 3550)**、**低延迟反馈 (RFC 4585)** 和 **编解码控制 (RFC 5104)** )

RTP 负责运送“货物”（音视频数据），RTCP 负责“物流监控”。

它的核心功能只有三个：

1. **服务质量监控 (QoS)**：统计丢包率、抖动、延时。
    
2. **媒体同步 (Synchronization)**：通过 NTP 时间戳将音频和视频对应起来。
    
3. **反馈控制 (Feedback)**：在丢包时请求重传 (NACK)，或请求关键帧 (Keyframe)。
    

**核心规则**：RTCP 包通常不单独发送，而是将多个类型的 RTCP 包打包在一个 UDP 包中（称为 **Compound Packet**，复合包）。例如：`SR + SDES + NACK`。

---

## 2. 第一层：基础报告 (RFC 3550)

这是 RTCP 最原始的功能，用于周期性汇报网络状况。

### 2.1 RTCP 通用头部 (4 Bytes)

所有 RTCP 包都以这个头部开始。

Plaintext

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|    RC   |   PT (Type)   |             Length            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **RC (Reception Report Count)**: 包含多少个接收报告块（用于 SR/RR）。在其他类型中可能作他用（如 FMT）。
    
- **PT (Payload Type)**: 决定了这个包是干嘛的。
    
    - **200**: SR (Sender Report) - 发送端报告
        
    - **201**: RR (Receiver Report) - 接收端报告
        
    - **202**: SDES (Source Description) - 源描述
        
    - **203**: BYE - 离开会话
        
    - **204**: APP - 应用自定义
        
    - **205**: RTPFB - 传输层反馈 (NACK, TWCC)
        
    - **206**: PSFB - 负载层反馈 (PLI, FIR, REMB)
        

### 2.2 SR: 发送端报告 (Sender Report, PT=200)

谁发？ 正在发送音视频流的一方。

作用？ 音视频同步 (Lip-sync) 的唯一依据。

关键字段：

1. **NTP Timestamp (64 bits)**: 绝对时间（墙上时钟）。
    
2. **RTP Timestamp (32 bits)**: 对应的 RTP 时间戳。
    
    - _原理_：接收端收到 SR 后，就知道了“这个 RTP 时间戳对应这个真实时间”。音频和视频都有各自的 SR，通过对比 NTP 时间，就能算出它们的时间差，从而实现对齐。
        
3. **Packet/Octet Count**: 我一共发了多少包/字节（用于计算发送速率）。
    

### 2.3 RR: 接收端报告 (Receiver Report, PT=201)

谁发？ 接收流的一方（如果不发流只收流，就只发 RR）。

作用？ 告诉发送端“我收的情况咋样”。

关键字段：

1. **Fraction Lost (8 bits)**: 最近一次报告以来的丢包率（0-255，255代表100%丢包）。
    
2. **Cumulative Loss (24 bits)**: 总共丢了多少包。
    
3. **Jitter (32 bits)**: 包到达的抖动（用于调整 Jitter Buffer）。
    

---

## 3. 第二层：丢包与重传 (RFC 4585 AVPF)

传统的 SR/RR 是几秒发一次，对于实时互动太慢了。RFC 4585 引入了 **RTPFB (Transport Layer Feedback, PT=205)**，允许接收端发现丢包**立刻**发送反馈。

### 3.1 NACK: 否定应答 (Generic NACK)

PT=205, FMT=1

接收端告诉发送端：“我没收到 Seq=100 的包，请重传”。

结构：

Plaintext

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |            PID (Packet ID)    |             BLP               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **PID (16 bits)**: 丢失的第一个包的序列号（Base Seq）。
    
- **BLP (16 bits)**: Bitmask of Lost Packets。位掩码。
    
    - 如果 PID=100，BLP 的第 0 位是 1，表示 Seq=101 也丢了。
        
    - BLP 的第 2 位是 1，表示 Seq=103 也丢了。
        
    - _优势_：一个 NACK 包可以描述连续 17 个包的丢失情况，非常高效。
        

### 3.2 TWCC: 传输层拥塞控制 (Transport-wide Congestion Control)

PT=205, FMT=15

WebRTC 目前主流的带宽估计算法。

- 接收端**不计算**丢包率，而是记录“每个包收到的精确时间（微秒级）”，打包发回给发送端。
    
- 发送端根据这些到达时间，计算网络拥塞程度（GCC 算法），从而主动降低码率。
    

---

## 4. 第三层：关键帧请求 (RFC 5104)

当丢包太严重，NACK 救不回来，画面已经花了，或者解码器崩溃了，接收端需要请求“刷新画面”。这属于 **PSFB (Payload Specific Feedback, PT=206)**。

这里有两个容易混淆的概念：PLI 和 FIR。

### 4.1 PLI: 图像丢失指示 (Picture Loss Indication)

**PT=206, FMT=1**

- **含义**：“我不懂这帧数据（参考帧丢了），解码器报错了，请给我发一个关键帧（IDR）让我恢复”。
    
- **场景**：最常用。当接收端检测到无法解码时发送。
    
- **负载**：无额外参数，只有头部。
    

### 4.2 FIR: 完整帧内刷新请求 (Full Intra Request)

**PT=206, FMT=4**

- **含义**：“我要一个新的开始”。
    
- **场景**：通常由应用层触发。例如：
    
    - 新用户加入会议，需要立刻看到画面。
        
    - 切换了摄像头。
        
    - 录制开始。
        
- **区别**：PLI 是解码器层面的被动补救；FIR 是控制层面的主动请求。FIR 带有一个 Sequence Number，发送端收到重复的 FIR 序列号会忽略。
    

### 4.3 REMB / TMMBR (旧时代的带宽限制)

- **REMB (Receiver Estimated Maximum Bitrate)**: 接收端估算带宽，告诉发送端“你最多发 1Mbps，不然我收不过来”。（PT=206, FMT=15）。目前正被 TWCC 取代。
    

---

## 5. RTCP 复合包结构 (Compound Packet)

这是实际开发中最需要注意的。为了节省 UDP/IP 头部开销，RTCP 总是打包发送。

**典型场景：接收端汇报**

Plaintext

```
[UDP Header]
  + [RTCP RR]   (汇报收到多少包，丢包率)
  + [RTCP SDES] (我是谁，CNAME=user1)
  + [RTCP NACK] (顺便告诉你，Seq=99 丢了，快补发)
```

**规则**：

1. 第一个包必须是 SR 或 RR。
    
2. 必须包含 SDES（CNAME），否则无法做音画同步。
    

---

## 6. C++ 代码示例：构建 NACK 包

这是一个构建 NACK RTCP 包的核心逻辑。

C++

```
#include <cstdint>
#include <vector>
#include <arpa/inet.h>

// RTCP 头部定义
struct RtcpHeader {
    uint8_t version_rc;      // V(2) | P(1) | FMT/RC(5)
    uint8_t payload_type;    // PT
    uint16_t length;         // Length (in 32-bit words) - 1
    uint32_t ssrc;           // Sender SSRC
    uint32_t media_ssrc;     // Media SSRC (target)
};

// NACK 块定义
struct NackBlock {
    uint16_t pid;
    uint16_t blp;
};

/**
 * 构建一个 NACK 包
 * @param sender_ssrc 发送者 SSRC (我)
 * @param media_ssrc 目标媒体 SSRC (我要报错的流)
 * @param lost_seqs 丢失的序列号列表 (如: 100, 101, 103)
 */
std::vector<uint8_t> BuildNackPacket(uint32_t sender_ssrc, uint32_t media_ssrc, const std::vector<uint16_t>& lost_seqs) {
    if (lost_seqs.empty()) return {};

    std::vector<uint8_t> packet;
    
    // 1. 预留头部空间 (Header + Media SSRC) = 8 bytes
    // RFC 4585 定义的 RTCPFB 头部比普通 RTCP 多一个 Media SSRC
    packet.resize(12); 
    
    RtcpHeader* hdr = (RtcpHeader*)packet.data();
    hdr->version_rc = 0x81; // V=2, P=0, FMT=1 (Generic NACK)
    hdr->payload_type = 205; // RTPFB
    hdr->ssrc = htonl(sender_ssrc);
    hdr->media_ssrc = htonl(media_ssrc);

    // 2. 填充 NACK Blocks
    // 简化逻辑：这里只处理首个 seq，实际需要遍历列表计算 BLP
    uint16_t pid = lost_seqs[0];
    uint16_t blp = 0;
    
    // 简单的位图计算逻辑 (仅示例)
    for (size_t i = 1; i < lost_seqs.size(); ++i) {
        int diff = lost_seqs[i] - pid - 1;
        if (diff >= 0 && diff < 16) {
            blp |= (1 << diff);
        }
    }

    // 添加到包体
    uint16_t n_pid = htons(pid);
    uint16_t n_blp = htons(blp);
    packet.insert(packet.end(), (uint8_t*)&n_pid, (uint8_t*)&n_pid + 2);
    packet.insert(packet.end(), (uint8_t*)&n_blp, (uint8_t*)&n_blp + 2);

    // 3. 回写长度
    // Length count in 32-bit words, minus 1.
    // Total bytes = 12 (Head) + 4 (NackBlock) = 16 bytes
    // Words = 4. Length field = 3.
    hdr->length = htons((packet.size() / 4) - 1);

    return packet;
}
```

---

## 7. 专家速查表：RTCP 类型汇总

在 Wireshark 分析时，死记硬背这些 PT 值：

|**PT (Payload Type)**|**名称**|**类别**|**核心功能**|**抓包特征**|
|---|---|---|---|---|
|**200**|**SR**|基础|发送端发 NTP 时间，**做音画同步**|包含 NTP Timestamp|
|**201**|**RR**|基础|接收端发丢包统计|Fraction Lost > 0 表示有丢包|
|**202**|**SDES**|基础|携带 CNAME，关联 Audio/Video 流|看到 CNAME 字符串|
|**205**|**RTPFB**|传输反馈|**FMT=1 (NACK)**: 补包<br><br>  <br><br>**FMT=15 (TWCC)**: 带宽估计|出现频繁，通常伴随丢包|
|**206**|**PSFB**|负载反馈|**FMT=1 (PLI)**: 请求关键帧<br><br>  <br><br>**FMT=4 (FIR)**: 重置编码器<br><br>  <br><br>**FMT=15 (REMB)**: 限速|画面花屏或卡顿后出现|

---

## 8. 常见疑问与坑

1. **为什么我看不到 SR？**
    
    - 只有发送 RTP 流的一方才发 SR。如果你只收不发（比如观众），你只发 RR。
        
    - SR 发送频率较低（通常几秒一次），抓包时间短可能抓不到。
        
2. **PLI 和 NACK 的优先级？**
    
    - **NACK 优先**：刚丢包时，先发 NACK 请求重传。
        
    - **PLI 兜底**：如果 NACK 重传回来还是晚了（Buffer 溢出），或者重传包也丢了，导致无法解码，这时候才发 PLI 请求 IDR 帧。
        
3. **什么是 "Reduced-Size" RTCP (RFC 5506)？**
    
    - 标准规定 RTCP 必须是复合包且必须带 RR/SR。
        
    - 但在弱网下，为了让 NACK/PLI 更快发送，可以开启 Reduced-Size 模式，允许**单独**发送一个小小的 NACK 包，不带前面的 SR/RR。这在 WebRTC 中很常见。
        
4. **音画不同步怎么办？**
    
    - 先抓包看发送端的 **SR** 包。
        
    - 检查 Audio SR 和 Video SR 的 NTP 时间戳是否准确（是否使用的是同一个系统时钟源）。
        
    - 如果 SR 没问题，检查接收端的逻辑：是否正确利用 SR 计算了 `ntp_time - rtp_timestamp` 的偏移量。
        
