
---

# H.265/HEVC 视频流 RTP 负载格式详解 (RFC 7798)

## 1. 概述

H.265 (High Efficiency Video Coding, HEVC) 是 H.264 的继任者，旨在提供更高的压缩效率。RFC 7798 定义了 HEVC 在 RTP 中的负载格式。

核心原则：与 H.264 一样，RTP 不传输 H.265 的起始码（Start Code, `0x00000001`），而是依赖 NAL Unit 长度或 RTP 边界来界定。**最大的区别在于 H.265 的 NALU 头部变成了 2 个字节**。

---

## 2. 基础知识：NAL Unit (NALU)

在封装前，必须理解 H.265 的 NALU 头部，因为 RTP 封装会解析并重组这些信息。

### 2.1 NALU 头部结构 (2 Bytes)

H.265 的头部比 H.264 多了一个字节，用于支持分层编码（L-HEVC）和并行处理。

Plaintext

```
    0                   1
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |F|   Type    |  LayerId  | TID |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **F (Forbidden_zero_bit, 1 bit)**: 必须为 **0**。
    
- **Type (NAL Unit Type, 6 bits)**: NAL 单元类型（范围 0-63）。注意 H.264 是 5 bits。
    
- **LayerId (6 bits)**: 层 ID。用于可伸缩编码（SHVC）或多视点（MV-HEVC）。普通流通常为 **0**。
    
- **TID (Temporal ID, 3 bits)**: 时间层 ID。用于帧率自适应，值越小越重要。**注意：TID 的实际值是该字段值 - 1**。
    

### 2.2 常见 NALU 类型表

H.265 的类型定义与 H.264 不同，关键类型如下：

|**Type (10进制)**|**描述**|**类别**|
|---|---|---|
|**0-9**|TRAIL (非关键帧，类似 P/B 帧)|VCL (视频编码层)|
|**19/20**|IDR (即时解码刷新，关键帧)|VCL|
|**21**|CRA (清洁随机访问，另一种关键帧)|VCL|
|**32**|**VPS (视频参数集)** - _H.265 新增_|Non-VCL|
|**33**|SPS (序列参数集)|Non-VCL|
|**34**|PPS (图像参数集)|Non-VCL|
|**48**|**AP (聚合包)** - _RTP 专用_|RTP|
|**49**|**FU (分片单元)** - _RTP 专用_|RTP|

---

## 3. RTP 数据包结构

H.265 的 RTP 头部与标准 RTP (RFC 3550) 一致。

### 3.1 RTP Header

Plaintext

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           Timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Synchronization Source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |             RTP Payload (H.265 NALU data)                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **M (Marker Bit)**: 一帧的最后一个包置 1。
    
- **PT (Payload Type)**: 动态协商（96-127）。
    
- **Timestamp**: 90,000 Hz。
    

---

## 4. RTP 负载模式 (Payload Structures)

RFC 7798 定义了以下打包模式。

### 模式一：单 NAL 单元模式 (Single NAL Unit)

**适用场景**：NALU 长度小于 MTU（如 VPS/SPS/PPS 或小的 P 帧）。

封装方式：

去掉起始码，直接拷贝 NALU。Payload 的前两个字节即为 H.265 的 NAL Header。

Plaintext

```
[RTP Header] + [HEVC NAL Header (2 Bytes)] + [NAL Payload]
```

### 模式二：聚合包 (Aggregation Packets - AP)

适用场景：多个小 NALU 打包在一个 RTP 包中（如一次性发送 VPS+SPS+PPS）。

类型值 (Type)：48

结构：

AP 头部（2字节）+ [长度(2字节) + NALU] + [长度(2字节) + NALU] ...

Plaintext

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Payload Header (Type=48)   |          NALU 1 Size          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          NALU 1 Header        |      NALU 1 Data...           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          NALU 2 Size          |      NALU 2 Header...         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Payload Header**: `F=0, Type=48, LayerId=0, TID=0`（通常取子单元中最小的 TID）。
    

### 模式三：分片单元 (Fragmentation Units - FU)

适用场景：NALU 大于 MTU（如 I 帧）。

类型值 (Type)：49

结构：

H.265 的 FU 结构包含 3 个字节 的头部开销：2 字节的 Payload Header + 1 字节的 FU Header。

Plaintext

```
[RTP Header] + [Payload Header (2B)] + [FU Header (1B)] + [Fragment Payload]
```

#### 1. Payload Header (2 Bytes)

作为 RTP Payload 的起始，伪装成一个普通的 NALU 头部。

- `F`: 0
    
- `Type`: **49** (表示 FU)。
    
- `LayerId`: 继承原始 NALU。
    
- `TID`: 继承原始 NALU。
    

#### 2. FU Header (1 Byte)

包含分片标记和原始类型。

Plaintext

```
+---------------+
|0|1|2|3|4|5|6|7|
+-+-+-+-+-+-+-+-+
|S|E|  FuType   |
+---------------+
```

- **S (Start)**: 1 表示起始分片。
    
- **E (End)**: 1 表示结束分片。
    
- **FuType (6 bits)**: **原始 NALU 的类型**（例如 IDR 帧就是 19）。
    

**注意**：H.264 的 FU Header 保留了一位 `R`，而 H.265 因为类型占 6 位，所以填满了整个字节，没有保留位。

---

## 5. 关键代码实现 (C++)

这是针对 H.265 调整后的打包逻辑，重点在于处理 2 字节的头部和 6 位的 Type 字段。

C++

```
#include <cstdint>
#include <vector>
#include <algorithm>
#include <cstring>

// H.265 定义
const uint8_t H265_NAL_TYPE_AP = 48;
const uint8_t H265_NAL_TYPE_FU = 49;
const int MAX_RTP_PAYLOAD = 1400;

struct RtpPacket {
    // RTP Header 省略...
    std::vector<uint8_t> payload;
    bool marker_bit;
};

/**
 * 将 H.265 NALU 打包成 RTP (不含 Start Code)
 * @param nalu_data 指向 NALU 数据 (前2字节是 NAL Header)
 */
std::vector<RtpPacket> PackHEVCtoRTP(const uint8_t* nalu_data, size_t nalu_size) {
    std::vector<RtpPacket> packets;

    // 解析 H.265 头部 (2 Bytes)
    uint8_t nal_header_1 = nalu_data[0];
    uint8_t nal_header_2 = nalu_data[1];
    
    // 提取字段
    // Byte 0: F(1) | Type(6) | LayerId_High(1)
    // Byte 1: LayerId_Low(5) | TID(3)
    uint8_t nal_type = (nal_header_1 & 0x7E) >> 1;
    
    // 模式 1: 单包发送
    if (nalu_size <= MAX_RTP_PAYLOAD) {
        RtpPacket packet;
        packet.marker_bit = true;
        packet.payload.insert(packet.payload.end(), nalu_data, nalu_data + nalu_size);
        packets.push_back(packet);
    }
    // 模式 2: FU 分片
    else {
        // 跳过 2 字节的 NAL Header，只分片负载
        const uint8_t* payload_ptr = nalu_data + 2;
        size_t payload_remain = nalu_size - 2;
        
        bool is_first_fragment = true;

        while (payload_remain > 0) {
            // FU 开销是 3 字节 (2 byte PayloadHeader + 1 byte FU Header)
            size_t chunk_size = std::min((size_t)MAX_RTP_PAYLOAD - 3, payload_remain);
            RtpPacket packet;

            // 1. 构建 Payload Header (2 Bytes)
            // 将 Type 替换为 49 (FU)，其余 F, LayerId, TID 保持不变
            // 注意：要小心位操作，保持 LayerId 最高位不动
            uint8_t ph1 = (nal_header_1 & 0x81) | (H265_NAL_TYPE_FU << 1); 
            uint8_t ph2 = nal_header_2; 

            // 2. 构建 FU Header (1 Byte)
            // S(1) | E(1) | FuType(6)
            uint8_t fu_header = nal_type; // 原始类型 (6 bits)
            if (is_first_fragment) {
                fu_header |= 0x80; // S=1
                packet.marker_bit = false;
            } else if (payload_remain == chunk_size) {
                fu_header |= 0x40; // E=1
                packet.marker_bit = true;
            } else {
                packet.marker_bit = false;
            }

            // 写入头部
            packet.payload.push_back(ph1);
            packet.payload.push_back(ph2);
            packet.payload.push_back(fu_header);

            // 写入数据
            packet.payload.insert(packet.payload.end(), payload_ptr, payload_ptr + chunk_size);

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

H.265 的 SDP 描述比 H.264 稍微复杂一些，因为它多了一个 VPS。

关键字段：

- **encoding-name**: 必须是 `H265` (或者旧草案中的 `HEVC`)。
    
- **sprop-vps**: Base64 编码的 VPS (Video Parameter Set)。
    
- **sprop-sps**: Base64 编码的 SPS。
    
- **sprop-pps**: Base64 编码的 PPS。
    
- **profile-id**: 指定 Profile (如 Main Profile)。
    

**SDP 示例**:

Plaintext

```
m=video 49170 RTP/AVP 98
a=rtpmap:98 H265/90000
a=fmtp:98 profile-id=1;sprop-vps=QAEMAf//AWAAAAMAkAAAAwAAAwB/kA==;sprop-sps=QgEBAWAAAAMAkAAAAwAAAwB/kAYJ;sprop-pps=RAHA8vA8kAA=;
```

---

## 7. 专家提示与排错指南

1. **VPS 是必须的**: 与 H.264 不同，H.265 引入了 VPS。如果你的 SDP 中没有 `sprop-vps`，或者发送关键帧时没有带上 VPS NALU，大多数 H.265 解码器将无法工作。
    
2. **头部解析错误**: H.265 的 `Type` 位于第一个字节的中间 6 位，很多开发者习惯了 H.264 的低 5 位，容易写错位移逻辑（`>>1` 是关键）。
    
3. **Payload Type**: H.265 没有固定的静态 Payload Type，必须使用 96-127 之间的动态值。
    
4. **iOS/Safari 兼容性**: 在 HLS 或 WebRTC 中使用 H.265 时，Apple 设备对 Tag 顺序和 `hvc1`/`hev1` 标识符非常敏感，但在 RTP 层面上主要关注上述封包即可。
    

---
