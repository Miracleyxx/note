
---

# H.264/AVC 视频流 RTP 负载格式详解 (RFC 6184)

## 1. 概述

在实时音视频通信（如 WebRTC、RTSP、IPTV）中，H.264 (MPEG-4 AVC) 是最常用的视频编码格式。RFC 6184 定义了如何将 H.264 的 NAL 单元（Network Abstraction Layer Unit）封装在 RTP 数据包中进行传输。

核心原则：**RTP 不传输 H.264 的起始码（Start Code, 如 `0x00000001`）**，而是通过长度字段或 RTP 边界来界定帧的范围。

---

## 2. 基础知识：NAL Unit (NALU)

H.264 流由一系列 NAL 单元组成。在封装到 RTP 之前，必须了解 NALU 的头部结构，因为 RTP 负载会直接利用或修改这个头部。

### 2.1 NALU 头部结构 (1 Byte)

标准 H.264 NALU 头部格式如下（**严格顺序**）：

```Plaintext
+---------------+---------------+---------------+
|0|1|2|3|4|5|6|7|
+-+-+-+-+-+-+-+-+
|F|NRI|  Type   |
+---------------+
```

- **F (Forbidden_zero_bit, 1 bit)**: 禁止位，协议规定必须为 **0**。如果为 1，表示该单元有错误，解码器通常丢弃。
    
- **NRI (nal_ref_idc, 2 bits)**: 重要性指示。`00` 表示该 NALU 不用于重建其他帧（可丢弃）；`11` 表示非常重要（如 SPS/PPS/IDR）。
    
- **Type (nal_unit_type, 5 bits)**: NAL 单元类型。
    

### 2.2 常见 NALU 类型表

|**Type (10进制)**|**描述**|**重要性**|
|---|---|---|
|**1**|非 IDR 图像的编码条带 (P帧/B帧)|高/中|
|**5**|IDR 图像的编码条带 (关键帧)|非常高|
|**6**|SEI (补充增强信息)|低|
|**7**|SPS (序列参数集)|**极高**|
|**8**|PPS (图像参数集)|**极高**|
|**24**|STAP-A (单一时间聚合包 A)|RTP专用|
|**28**|FU-A (分片单元 A)|RTP专用|

---

## 3. RTP 数据包结构

H.264 的 RTP 包遵循标准 RTP 头部（RFC 3550），后跟 H.264 负载。

### 3.1 RTP Header

```Plaintext
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           Timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Synchronization Source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |                                                               |
   |               RTP Payload (H.264 NALU data)                   |
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **M (Marker Bit)**: 标记位。在 H.264 中，当 RTP 包包含一帧图像的**最后一个** NALU（或其分片）时，该位置 1。这告诉接收端“这一帧数据接收完毕，可以送去解码了”。
    
- **PT (Payload Type)**: 动态负载类型，通常在 SDP 中协商（范围 96-127）。
    
- **Timestamp**: 时间戳。H.264 视频通常使用 **90,000 Hz** 的时钟频率。
    

---

## 4. RTP 负载模式 (Payload Structures)

RFC 6184 定义了三种主要的负载模式。根据 NALU 的大小和网络 MTU 限制，我们会选择不同的打包方式。

### 模式一：单 NAL 单元模式 (Single NAL Unit Mode)

**适用场景**：NALU 长度小于 MTU（通常 < 1400 字节）。例如 SEI 信息或较小的 P 帧。

封装方式：

直接将 NALU（去掉起始码）放入 RTP Payload。

- RTP Payload 的第一个字节就是 NALU Header。
    

```Plaintext
[RTP Header] + [NAL Header] + [NAL Payload]
```

### 模式二：聚合包 (Aggregation Packets - STAP-A)

适用场景：多个很小的 NALU（如 SPS 和 PPS），加起来仍然小于 MTU。为了节省 RTP 头部开销，将它们打包在一个 RTP 包中发送。

类型值 (Type)：24

结构：

STAP-A 头部（1字节） + [长度(2字节) + NALU] + [长度(2字节) + NALU] ...

```Plaintext
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|STAP-A Header  |         NALU 1 Size           | NALU 1 HDR    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| NALU 1 Data...                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         NALU 2 Size           | NALU 2 HDR    | NALU 2 Data...|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **STAP-A Header**: `F=0, NRI=(取所有子NALU中最大的NRI), Type=24`。
    
### 模式三：分片单元 (Fragmentation Units - FU-A)

适用场景：NALU 大于 MTU（例如高清视频的关键帧 I 帧）。必须拆分发送。

类型值 (Type)：28

结构：

原来的 NALU 头部被拆分，信息分散在 FU Indicator 和 FU Header 两个字节中。

```Plaintext
[RTP Header] + [FU Indicator] + [FU Header] + [Fragment Payload]
```

#### 1. FU Indicator (1 Byte)

结构与普通 NALU 头部几乎一样，但 Type 被替换为 28。

- `F`: 0
    
- `NRI`: 继承自原始 NALU 的 NRI。
    
- `Type`: **28** (表示这是 FU-A)。
    

#### 2. FU Header (1 Byte)

用于标记分片的顺序和恢复原始 NALU 类型。



```Plaintext
+---------------+
|0|1|2|3|4|5|6|7|
+-+-+-+-+-+-+-+-+
|S|E|R|  Type   |
+---------------+
```

- **S (Start)**: 1 表示这是分片的第一个包。
    
- **E (End)**: 1 表示这是分片的最后一个包。
    
- **R (Reserved)**: 保留位，必须为 0。
    
- **Type**: **原始 NALU 的类型**（例如 IDR 帧就是 5）。
    

分片规则举例：

假设有一个大 I 帧（Type=5）。

- **第一包**: `S=1, E=0`, Type=5。
    
- **中间包**: `S=0, E=0`, Type=5。
    
- **最后包**: `S=0, E=1`, Type=5。（此时 RTP Header 的 Marker 位也应设为 1）。
    

---

## 5. 关键代码实现 (C++)

这是一个修正后的、更严谨的打包逻辑伪代码，修复了你原始笔记中关于头部计算的逻辑错误。



```C++
#include <cstdint>
#include <vector>
#include <cstring>

// 定义常量
const uint8_t H264_NAL_TYPE_STAP_A = 24;
const uint8_t H264_NAL_TYPE_FU_A = 28;
const int MAX_RTP_PAYLOAD = 1400; // 假设 MTU 限制

struct RtpPacket {
    // 省略 RTP Header 定义...
    std::vector<uint8_t> payload;
    bool marker_bit;
};

/**
 * 将原始 NALU (不含 Start Code) 打包成 RTP 包列表
 * @param nalu_data 指向 NALU 数据的指针 (第一个字节是 NAL Header)
 * @param nalu_size NALU 数据长度
 */
std::vector<RtpPacket> PackNALUtoRTP(const uint8_t* nalu_data, size_t nalu_size, uint32_t timestamp) {
    std::vector<RtpPacket> packets;
    
    // 解析原始 NAL 头
    uint8_t nalu_header = nalu_data[0];
    uint8_t f_bit = (nalu_header & 0x80);
    uint8_t nri   = (nalu_header & 0x60);
    uint8_t type  = (nalu_header & 0x1F);

    // 情况 1: 单一 NAL 单元模式 (Single NAL Unit)
    if (nalu_size <= MAX_RTP_PAYLOAD) {
        RtpPacket packet;
        packet.marker_bit = true; // 单包即是一帧结束
        // 直接拷贝整个 NALU
        packet.payload.insert(packet.payload.end(), nalu_data, nalu_data + nalu_size);
        packets.push_back(packet);
    } 
    // 情况 2: FU-A 分片模式
    else {
        // 跳过原始 NAL 头，只对负载分片
        const uint8_t* payload_ptr = nalu_data + 1;
        size_t payload_remain = nalu_size - 1;
        
        bool is_first_fragment = true;

        while (payload_remain > 0) {
            size_t chunk_size = std::min((size_t)MAX_RTP_PAYLOAD - 2, payload_remain); // -2 是因为有 FU Indicator 和 FU Header
            RtpPacket packet;
            
            // 1. 构建 FU Indicator
            // F(1) | NRI(2) | Type(5 -> 28 for FU-A)
            uint8_t fu_indicator = f_bit | nri | H264_NAL_TYPE_FU_A;
            
            // 2. 构建 FU Header
            // S(1) | E(1) | R(1) | Original Type(5)
            uint8_t fu_header = type; // 原始类型写入低5位
            if (is_first_fragment) {
                fu_header |= 0x80; // Set S bit
                packet.marker_bit = false;
            } else if (payload_remain == chunk_size) {
                fu_header |= 0x40; // Set E bit
                packet.marker_bit = true; // 最后一包置 Marker
            } else {
                packet.marker_bit = false;
            }

            // 写入头部
            packet.payload.push_back(fu_indicator);
            packet.payload.push_back(fu_header);
            
            // 写入分片数据
            packet.payload.insert(packet.payload.end(), payload_ptr, payload_ptr + chunk_size);
            
            // 迭代更新
            payload_ptr += chunk_size;
            payload_remain -= chunk_size;
            is_first_fragment = false;
            
            packets.push_back(packet);
        }
    }
    return packets;
}
```

---

## 6. SDP 协商参数 (Session Description Protocol)

在实际互联网传输中（如 WebRTC），仅仅知道如何封包是不够的，必须在 SDP 中正确描述 H.264 的能力。

关键字段解释：

- **packetization-mode**:
    
    - `0`: 仅支持单 NAL 单元。
        
    - `1`: **(最常用)** 支持单 NAL 单元、STAP-A 和 FU-A。
        
- **profile-level-id**: 例如 `42e01f`。表示 Baseline Profile, Level 3.1。这决定了编解码的复杂度和分辨率上限。
    
- **sprop-parameter-sets**: Base64 编码的 SPS 和 PPS。这允许接收端在收到关键帧之前就能初始化解码器，加快首帧出图速度。
    

**SDP 示例**:

```Plaintext
m=video 49170 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42e01f;sprop-parameter-sets=Z0LgH5ZUBaHPbAA=,aM48gA==;
```

---

## 7. 专家提示与排错指南

1. **Start Code 问题**: 最常见的错误是将 `00 00 00 01` 也打包进 RTP。请务必在封装前剥离它们，在解包后如果送给解码器需要，再重新加上。
    
2. **Marker Bit**: 许多播放器（如 VLC）依赖 RTP 头部的 Marker bit 来触发显示。如果分片逻辑写错，没有正确设置 M 位，可能导致花屏或延迟。
    
3. **时间戳**: 同一帧的所有分片（FU-A）必须拥有**相同**的 RTP 时间戳。
    
4. **MTU 大小**: 互联网传输建议 MTU 设为 1200 左右，而非以太网的 1500，以避免 UDP 在 IP 层被分片（IP Fragmentation），这会增加丢包概率。
    

---