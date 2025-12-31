
---

# AAC 音频流 RTP 负载格式详解 (RFC 3640)

## 1. 概述

AAC (Advanced Audio Coding) 是目前互联网最通用的音频编码格式。与 H.264/H.265 不同，AAC 的 RTP 封装通常**不使用 ADTS 头部**传输，而是传输“裸”的音频帧（Access Unit, AU），并将帧的大小和位置信息放在 RTP 负载的专用头部区域（AU Header Section）中。

核心原则：**去掉 ADTS 头，提取 Raw Data，通过 RTP 负载头描述帧长度。**

---

## 2. 基础知识：AAC 帧与配置

在封装 RTP 之前，你需要区分“传输流格式”和“RTP 负载格式”。

### 2.1 ADTS vs Raw AAC

- **ADTS (Audio Data Transport Stream)**: 常见于 `.aac` 文件。每一帧前面都有 7 或 9 字节的头部（包含采样率、声道等信息）。
    
- **Raw AAC (Access Unit)**: 去掉 ADTS 头部后的纯音频数据。**RTP 传输通常只带这部分**。
    

### 2.2 AudioSpecificConfig (ASC)

由于去掉了 ADTS 头，接收端如何知道采样率和声道数？

答案在于 SDP。AAC 将这些元数据编码为一串 Hex 字符串（称为 AudioSpecificConfig），通过 SDP 的 config 字段带给接收端。这类似于 H.264 的 SPS/PPS。

---

## 3. RTP 数据包结构

AAC (RFC 3640) 的 RTP 包结构比视频稍微复杂一点，因为它允许在一个 RTP 包中携带多个音频小帧。

结构如下：

Plaintext

```
[RTP Header] + [AU Header Section] + [Access Unit 1] + [Access Unit 2] ...
```

### 3.1 RTP Header

标准 RTP 头部（RFC 3550）。

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
   |            AU Header Section (长度 + 实际Header)              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |            Access Unit (Audio Frame) Data ...                 |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **M (Marker Bit)**: 在音频中，通常置 **1**，表示这是一个完整的访问单元或包含了时间跨度的结束。但在 `AAC-hbr` 模式下，通常每个包都置 1（除非发生了分片，但音频极少分片）。
    
- **Timestamp**: 采样率时钟（例如 44100 Hz 或 48000 Hz）。
    

---

## 4. 负载详情：AU Header Section

这是 RFC 3640 的核心。因为一个 RTP 包可能包含多个 AAC 帧（虽然通常是 1 个），所以需要一个“目录”来告诉接收端每个帧有多长。

### 4.1 AU Header Section 结构

Plaintext

```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|AU-headers-length| AU-Header-1 | (AU-Header-2...)
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

1. **AU-headers-length (16 bits)**: 表示后面紧接着的“AU Headers”总共占用了多少 **位 (Bits)**。注意是位，不是字节！
    
2. **AU-Header (通常 16 bits)**: 每个 AAC 帧对应一个 AU-Header。
    

### 4.2 单个 AU-Header 的结构

在 `mode=AAC-hbr` 下，AU-Header 的大小通常配置为 16 bits，由两部分组成：

Plaintext

```
+-----------------------+-------+
|   AU-Size (13 bits)   | Index |
+-----------------------+-------+
```

- **AU-Size (13 bits)**: 这一帧 AAC 数据的长度（字节数）。
    
- **AU-Index / AU-Index-Delta (3 bits)**: 通常为 0。用于交错传输，但在常规流媒体中很少使用。
    

示例计算：

假设有一个 AAC 帧，长度 300 字节。

- **AU-headers-length**: 16 (因为只有一个 16 bit 的头)。
    
- **AU-Header**: 300 (二进制 `0000100101100`) << 3 | 0 = `0x0960`。
    

---

## 5. 关键代码实现 (C++)

下面的代码展示了如何将一个**裸 AAC 帧**（即已去掉 ADTS 头）封装进 RTP 包。

C++

```
#include <cstdint>
#include <vector>
#include <arpa/inet.h> // for htons

struct RtpPacket {
    // RTP Header 省略...
    std::vector<uint8_t> payload;
    bool marker_bit;
};

/**
 * 将 Raw AAC 帧打包成 RTP (RFC 3640)
 * @param aac_data 指向裸 AAC 数据的指针 (不含 ADTS)
 * @param aac_size AAC 数据长度
 */
RtpPacket PackAACtoRTP(const uint8_t* aac_data, size_t aac_size) {
    RtpPacket packet;
    packet.marker_bit = true; // 音频帧通常设为 1

    // 1. 构建 AU Header Section
    // 这里假设每个 RTP 包只发 1 个 AAC 帧 (最常见做法)
    // SizeOf(AU-Header) = 16 bits (13 bits size + 3 bits index)
    
    // 字段 A: AU-headers-length (16 bits)
    // 值 = 16 (表示接下来的 header 只有 16 bits 长)
    uint16_t headers_length_field = htons(16); 

    // 字段 B: AU-Header (16 bits)
    // 结构: size(13) | index(3)
    uint16_t au_header_field = (uint16_t)aac_size;
    au_header_field = au_header_field << 3; // 腾出后 3 位给 index
    au_header_field = htons(au_header_field);

    // 2. 填充 Payload
    // 写入 AU-headers-length
    uint8_t* ptr = (uint8_t*)&headers_length_field;
    packet.payload.push_back(ptr[0]);
    packet.payload.push_back(ptr[1]);

    // 写入 AU-Header
    ptr = (uint8_t*)&au_header_field;
    packet.payload.push_back(ptr[0]);
    packet.payload.push_back(ptr[1]);

    // 3. 写入实际 AAC 数据
    packet.payload.insert(packet.payload.end(), aac_data, aac_data + aac_size);

    return packet;
}
```

---

## 6. SDP 协商参数

AAC 的 SDP 非常关键，如果不匹配，接收端只能听到静音或噪声。

关键字段：

- **streamtype=5**: 表示音频。
    
- **mode=AAC-hbr**: High Bit-rate 模式（强制要求）。
    
- **sizelength=13**: 指定 AU-Header 中 Size 字段占 13 位。
    
- **indexlength=3**: 指定 AU-Header 中 Index 字段占 3 位。
    
- **indexdeltalength=3**: 指定 Delta 字段占 3 位。
    
- **config**: **最重要**。AudioSpecificConfig 的十六进制表示。
    
    - 例如：`1210` (Hex) -> `00010 010 0001 0000` (Bin)
        
    - AAC-LC (5 bits = 2), 44100Hz (4 bits = 4), Stereo (4 bits = 2)。
        

**SDP 示例**:

Plaintext

```
m=audio 49170 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/44100/2
a=fmtp:97 streamtype=5; profile-level-id=1; mode=AAC-hbr; sizelength=13; indexlength=3; indexdeltalength=3; config=1210;
```

---

## 7. 专家提示与排错指南

1. **AU-headers-length 单位**: 必须记住它表示的是 **Bit** 长度，不是 Byte。如果你打包一个帧，长度填 `16` (0x0010)；如果打包两个帧，长度填 `32` (0x0020)。
    
2. **ADTS 剥离**: 如果你的源文件是 `.aac`，必须手动解析每一帧的 ADTS 头，获取帧长，去掉头（通常 7 字节），只把剩下的数据给 RTP 打包函数。
    
3. **Config 字符串**: 如果你不知道怎么生成 `config` 字符串，可以用 `ffmpeg` 命令行推流并抓包查看 SDP，或者查阅 ISO 14496-3 标准手动拼凑位字段。
    
4. **采样率**: RTP Header 里的 Timestamp 增量必须等于 `1024` (对于 AAC-LC)，因为 AAC 一帧包含 1024 个采样点。
    

---
