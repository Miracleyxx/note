
---

# VP9 视频流 RTP 负载格式详解 (RFC 9054)

## 1. 概述

VP9 是一种开源、免版税的视频编码格式。与 H.264/H.265 不同，VP9 没有 NAL Unit 的概念，而是直接处理**视频帧 (Video Frames)**。

核心原则：VP9 的 RTP 封装非常灵活，支持**SVC (Scalable Video Coding)**。这意味着 RTP 头部不仅仅是分片标记，还包含了丰富的**空间层 (Spatial Layer)** 和 **时间层 (Temporal Layer)** 信息，允许中间网关（SFU）直接根据网络状况丢弃某些层级的包，而无需重新编码。

---

## 2. 基础知识：VP9 帧结构

在 H.264 中我们关注 NALU，而在 VP9 中，我们关注**Payload Descriptor**。

### 2.1 关键差异

- **无起始码**：不需要 `0x00000001`。
    
- **Payload Descriptor**：RTP Payload 的第一部分不是原始视频数据，而是 RTP 协议层面的描述符，用于告知接收端这是帧的开始/结束、属于哪一层、参考了哪些帧等。
    

---

## 3. RTP 数据包结构

VP9 的 RTP 包由标准 RTP 头、VP9 负载描述符（Payload Descriptor）和 VP9 压缩数据组成。

Plaintext

```
[RTP Header] + [VP9 Payload Descriptor] + [VP9 Compressed Frame Data]
```

### 3.1 RTP Header

标准 RFC 3550 头部。

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
   |            VP9 Payload Descriptor (变长)                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   |            VP9 Compressed Video Data ...                      |
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **M (Marker Bit)**: 标记一帧的**最后一个** RTP 包。
    
- **PT**: 动态协商 (96-127)。
    
- **Timestamp**: 90,000 Hz。
    

---

## 4. 核心：VP9 Payload Descriptor

这是 VP9 RTP 封装最复杂也是最强大的部分。它至少包含 1 个字节，但通常会扩展到多个字节以携带 PictureID 或 Layer 信息。

### 4.1 必需头 (Mandatory Header - 1 Byte)

所有 VP9 RTP 包必须包含这第一个字节。

Plaintext

```
     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |I|P|L|F|B|E|V|Z|
    +-+-+-+-+-+-+-+-+
```

- **I (Picture ID)**: 1 表示后面紧跟着 Picture ID 字段（用于跨包参考和丢包恢复）。**WebRTC 中通常置 1**。
    
- **P (Inter-picture predicted)**: 0 表示这是关键帧（Key Frame），且没有参考之前的帧；1 表示是 Inter 帧（P帧）。
    
- **L (Layer indices)**: 1 表示后面包含层级信息（Temporal/Spatial ID）。SVC 必备。
    
- **F (Flexible mode)**: 1 表示灵活模式（使用 RTP 头指定参考帧），0 表示非灵活模式。
    
- **B (Start of Frame)**: 1 表示这是该视频帧的第一个 RTP 包。
    
- **E (End of Frame)**: 1 表示这是该视频帧的最后一个 RTP 包。
    
- **V (Scalability Structure)**: 1 表示包含 SS 数据（描述整个流的分层结构，通常只在关键帧出现）。
    
- **Z (Not a reference)**: 1 表示该帧不会被后续帧参考（可以直接丢弃）。
    

### 4.2 扩展字段：Picture ID (Variable Length)

如果 `I=1`，则必需头后面紧跟 Picture ID。这在 UDP 传输中至关重要，因为 RTP 序列号只能保证包顺序，而 Picture ID 保证帧的完整性。

- **7-bit 模式** (首位为 0): `0xxxxxxx`
    
- **15-bit 模式** (首位为 1): `1xxxxxxx xxxxxxxx`
    

### 4.3 扩展字段：Layer Indices (Optional)

如果 `L=1`，则包含层级信息。

Plaintext

```
     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    | T | U | S | D |
    +-+-+-+-+-+-+-+-+
```

- **T (Temporal ID)**: 3 bits，时间层 ID。
    
- **U (Switching up point)**: 1 bit，表示是否可以向上切换层级。
    
- **S (Spatial ID)**: 3 bits，空间层 ID。
    
- **D (Inter-layer dependency)**: 1 bit。
    

---

## 5. 分片机制 (Fragmentation)

与 H.264/H.265 需要专门的 FU-A NALU 不同，VP9 的分片机制直接集成在 **Mandatory Header** 的 `B` 和 `E` 位中。这使得逻辑更加扁平。

### 场景 1：小帧，单包发送 (Single Packet)

帧大小 < MTU。

- **B = 1**: 是开始。
    
- **E = 1**: 也是结束。
    
- **Payload Descriptor**: `I=1, B=1, E=1, ...`
    

### 场景 2：大帧，分片发送 (Fragmentation)

帧大小 > MTU。

1. **第一包 (Start)**:
    
    - `B = 1`, `E = 0`
        
    - RTP Marker = 0
        
2. **中间包 (Middle)**:
    
    - `B = 0`, `E = 0`
        
    - RTP Marker = 0
        
    - 注意：中间包通常也建议携带 Picture ID (I=1)，以便接收端能组装。
        
3. **最后包 (End)**:
    
    - `B = 0`, `E = 1`
        
    - RTP Marker = 1
        

---

## 6. 关键代码实现 (C++)

下面的代码展示了最通用的 VP9 打包逻辑（包含 PictureID 支持，这是 WebRTC 的标准做法）。

C++

```
#include <cstdint>
#include <vector>
#include <algorithm>
#include <cstring>
#include <arpa/inet.h>

const int MAX_RTP_PAYLOAD = 1200; // 保守 MTU

struct RtpPacket {
    // RTP Header 省略...
    std::vector<uint8_t> payload;
    bool marker_bit;
};

// 全局状态模拟 (实际应在类成员变量中)
uint16_t g_picture_id = 0; 

/**
 * 将 VP9 帧打包成 RTP
 * @param frame_data VP9 压缩帧数据
 * @param frame_size 数据长度
 * @param is_key_frame 是否为关键帧
 */
std::vector<RtpPacket> PackVP9toRTP(const uint8_t* frame_data, size_t frame_size, bool is_key_frame) {
    std::vector<RtpPacket> packets;
    
    size_t offset = 0;
    size_t remaining = frame_size;
    
    // 增加 PictureID (15-bit mode for safety wraps)
    g_picture_id++; 
    g_picture_id &= 0x7FFF; // 保持 15 bit

    while (remaining > 0) {
        RtpPacket packet;
        
        // --- 1. 计算 Payload Descriptor ---
        
        // 计算可用空间: Max - (Mandatory Byte + 2 Bytes PictureID)
        size_t header_overhead = 1 + 2; 
        size_t chunk_size = std::min((size_t)MAX_RTP_PAYLOAD - header_overhead, remaining);

        uint8_t mandatory_byte = 0;
        
        // 设置 I 位 (Picture ID present)
        mandatory_byte |= 0x80; 
        
        // 设置 P 位 (0 for KeyFrame, 1 for InterFrame)
        if (!is_key_frame) {
            mandatory_byte |= 0x40;
        }

        // 设置 B 位 (Start of Frame)
        if (offset == 0) {
            mandatory_byte |= 0x08;
        }

        // 设置 E 位 (End of Frame)
        if (remaining == chunk_size) {
            mandatory_byte |= 0x04;
            packet.marker_bit = true; // RTP Header Marker
        } else {
            packet.marker_bit = false;
        }

        // --- 2. 写入头部 ---
        
        // Byte 0: Mandatory Header
        packet.payload.push_back(mandatory_byte);
        
        // Byte 1-2: Extended Picture ID (15-bit mode)
        // M=1 (bit 7 of first byte indicates 15-bit extension)
        uint8_t pid_byte1 = 0x80 | ((g_picture_id >> 8) & 0x7F);
        uint8_t pid_byte2 = g_picture_id & 0xFF;
        
        packet.payload.push_back(pid_byte1);
        packet.payload.push_back(pid_byte2);
        
        // --- 3. 写入数据 ---
        packet.payload.insert(packet.payload.end(), frame_data + offset, frame_data + offset + chunk_size);
        
        offset += chunk_size;
        remaining -= chunk_size;
        
        packets.push_back(packet);
    }
    
    return packets;
}
```

---

## 7. SDP 协商参数

VP9 的 SDP 相对简单，因为它不需要像 H.264 那样传输 SPS/PPS（VP9 关键帧自带这些信息）。

关键字段：

- **encoding-name**: `VP9`
    
- **profile-id**:
    
    - `0`: 8-bit color, 4:2:0 (最常用)
        
    - `2`: 10-bit / 12-bit color (HDR)
        

**SDP 示例**:

Plaintext

```
m=video 49170 RTP/AVP 98
a=rtpmap:98 VP9/90000
a=fmtp:98 profile-id=0;
```

---

## 8. 专家提示与排错指南

1. **Picture ID 的连续性**: 很多 VP9 解码器（特别是 libvpx）在 RTP 模式下非常依赖 `Picture ID` 来检测帧的丢失或乱序。如果在 SFU 转发时修改了 SSRC，必须重新映射并维护连续的 Picture ID，否则会导致解码画面闪烁或卡死。
    
2. **Flexible Mode**: 如果你在做高级的 WebRTC 开发，注意 `F=1`（Flexible Mode）。这种模式下，RTP 头里会直接写明“我参考了之前哪个 Picture ID 的帧”。这允许非常复杂的参考结构（如 SVC 的不同层级参考），但也增加了实现的复杂度。
    
3. **关键帧识别**: 与 H.264 不同，你不能只看 RTP 头里的 `P` 位来判断是不是 I 帧。`P=0` 只是必要条件。真正的关键帧判断通常需要解析 VP9 Payload 内部的 raw header。
    
4. **SVC 丢包逻辑**: 如果你通过 VP9 做 Simulcast 或 SVC，当网络拥塞时，可以直接丢弃 `TID` (Temporal ID) 较高的包（例如丢弃 T=2 的包，保留 T=0,1），这不会影响基础画质，是 VP9 的一大优势。
    

---


