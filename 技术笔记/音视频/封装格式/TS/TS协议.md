

---
# MPEG-2 Transport Stream (TS) 技术详解

## 1. 概述与核心概念

**MPEG-2 TS (Transport Stream)** 是一种标准的容器格式（ISO/IEC 13818-1），专门为在不可靠信道（如地面广播、卫星、有线电视、IP 网络）上传输视频数据而设计。

### 1.1 TS 包 (Packet)

TS 的基本原子单位。固定长度为 188 字节。

无论是在磁盘文件里，还是在网络线缆中，TS 始终由一个个 188 字节的包首尾相连组成。

### 1.2 TS 文件 vs TS 流

虽然数据载体（TS Packet）结构一致，但在应用形态上有本质区别：

|**特性**|**TS 文件 (Storage)**|**TS 流 (Transmission)**|
|---|---|---|
|**存在形式**|硬盘上的静态文件（如 `.ts`）。|网络上的动态数据流（如 UDP/RTP）。|
|**排列方式**|成千上万个 188 字节包紧密排列。|为了适应 MTU，通常每 **7** 个 TS 包打成一个 UDP/RTP 包传输。|
|**时间基准**|**被动**。解码速度取决于读取磁盘的速度（Seek 操作依赖字节偏移）。|**主动**。严格依赖 **PCR (Program Clock Reference)** 恢复时钟，必须按固定码率发送。|
|**典型大小**|文件总大小是 188 的整数倍。|每次网络传输 Payload 大小为 **1316 字节** ($188 \times 7$)。|

---

## 2. TS 包结构全景图 (188 Bytes)

一个标准的 TS 包由 4 字节头部和 184 字节数据区域组成。数据区域的具体结构由头部中的 **AFC** 字段决定。

### 三种载荷模式

1. **仅有效负载 (Payload Only)**：最常见，用于满载音视频数据。
    
    Plaintext
    
    ```
    | TS Header (4B) | Payload (184B) |
    ```
    
2. **仅自适应区 (Adaptation Only)**：用于填充空包或仅发送 PCR 时钟。
    
    Plaintext
    
    ```
    | TS Header (4B) | Adaptation Field (184B) |
    ```
    
3. **混合模式 (Mixed)**：既有控制信息又有数据。
    
    Plaintext
    
    ```
    | TS Header (4B) | Adaptation Field (X Byte) | Payload (184-X Byte) |
    ```
    

---

## 3. TS 头部详解 (TS Header - 4 Bytes)

这是 TS 协议的核心，共 32 位。

### 3.1 头部比特布局图



```Plaintext
    Byte 1      |      Byte 2       |      Byte 3       |      Byte 4
0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7 | 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Sync Byte   |T|P|T|     PID (13 bits)           |T|A|   CC  |
|     (0x47)    |E|U|P|                             |S|F|       |
|               |I|S| |                             |C|C|       |
|               | |I| |                             | | |       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 3.2 字段详细定义

#### **第 1 字节**

|**字段**|**位宽**|**值**|**描述**|
|---|---|---|---|
|**Sync Byte**|8|`0x47`|**同步字节**。固定为 `0x47` ('G')。解码器通过检测该字节来锁定包的边界。如果每隔 188 字节不是 0x47，说明流同步丢失。|

#### **第 2-3 字节**

|**字段**|**位宽**|**描述**|
|---|---|---|
|**TEI** (Transport Error Indicator)|1|**传输错误指示**。<br><br>  <br><br>`1`: 表示该包在物理传输中损坏（无法纠错），接收端应丢弃。<br><br>  <br><br>`0`: 正常。|
|**PUSI** (Payload Unit Start Indicator)|1|**负载单元起始标记**（非常重要）。<br><br>  <br><br>`1`: 表示该包的 Payload 开始处包含了一个新数据单元的头部。<br><br>  <br><br>• 若是 PSI (PAT/PMT)，表示 Pointer Field 存在。<br><br>  <br><br>• 若是 PES (视频/音频)，表示 **PES Header** (及 PTS/DTS) 从这里开始。<br><br>  <br><br>**开发提示**：找 I 帧或 PTS 时，必须先过滤 `PUSI=1` 的包。|
|**TP** (Transport Priority)|1|**传输优先级**。通常为 0，用于指示该包比同 PID 的其他包更重要。|
|**PID** (Packet Identifier)|13|**包标识符**（身份证）。用于区分流的类型。<br><br>  <br><br>• `0x0000`: **PAT 表** (根目录)。<br><br>  <br><br>• `0x0001`: CAT 表。<br><br>  <br><br>• `0x1FFF`: **Null Packet** (填充空包)。<br><br>  <br><br>• `0x0020`~`0x1FFE`: 节目流 (视频、音频、PMT)。|

#### **第 4 字节**

|**字段**|**位宽**|**描述**|
|---|---|---|
|**TSC** (Transport Scrambling Control)|2|**加扰控制**。<br><br>  <br><br>`00`: 未加密。<br><br>  <br><br>其他值: 使用了 DVB-CSA 等加密方式。|
|**AFC** (Adaptation Field Control)|2|**自适应区控制**（决定包结构）。<br><br>  <br><br>`01`: 仅含 Payload (无自适应区)。<br><br>  <br><br>`10`: 仅含自适应区 (无 Payload)。<br><br>  <br><br>`11`: **两者都有** (Adaptation Field 紧跟 Header，剩余部分为 Payload)。<br><br>  <br><br>`00`: 保留。|
|**CC** (Continuity Counter)|4|**连续性计数器**。<br><br>  <br><br>范围 0-15 循环递增。用于检测丢包。<br><br>  <br><br>**注意**：仅当包含有 Payload 时 CC 才增加；空包或仅含 AF 的包不增加。|

---

## 4. 自适应区详解 (Adaptation Field)

当 Header 中的 `AFC` 为 `10` 或 `11` 时，Header 后紧跟自适应区。这是 TS 流进行**时钟同步**的关键区域。

### 4.1 结构布局

Plaintext

```
+--------+------------+-------------+------------------+
| Length | Indicators | PCR (Optional)| Stuffing Bytes...|
| (1 B)  |  (1 B)     | (6 Bytes)   | (Variable)       |
+--------+------------+-------------+------------------+
```

1. **AF Length (8 bits)**: 指定后续自适应区的长度。
    
2. **Indicators (8 bits)**: 标志位。
    
    - **PCR_flag (bit 4)**: `1` 表示包含 PCR 字段。
        
    - **Random_Access_indicator (bit 6)**: `1` 表示这是随机访问点（通常对应视频的关键帧起始）。
        
3. **PCR (Program Clock Reference)**: **48 bits (6 Bytes)**。
    
    - 由 33-bit 的 Base (90kHz) 和 9-bit 的 Extension (27MHz) 组成。
        
    - **作用**：接收端利用 PCR 锁相环（PLL）恢复出与发送端完全一致的 27MHz 系统时钟，防止缓冲区溢出或欠载。
        

---

## 5. 逻辑解复用流程 (Demuxing)

如何从一堆 188 字节的包中还原出 H.264 视频？

**步骤 1: 找 PAT (PID=0)**

- 解析 PID=0 的包。
    
- 获取节目映射信息：`Program 1 -> PMT PID = 100`。
    

**步骤 2: 找 PMT (PID=100)**

- 解析 PID=100 的包。
    
- 获取流信息：
    
    - Stream Type `0x1B` (H.264 Video) -> **PID 256**。
        
    - Stream Type `0x0F` (AAC Audio) -> **PID 257**。
        

**步骤 3: 重组 PES (PID=256)**

- 过滤出所有 PID=256 的 TS 包。
    
- **起始**：找到 `PUSI=1` 的包，提取 PES Header (读取 PTS)。
    
- **拼接**：将后续 `PUSI=0` 的包的 Payload 按顺序拼接到后面。
    
- **结束**：直到遇到下一个 `PUSI=1` 的包，说明当前 PES 结束。
    

**步骤 4: 提取 ES**

- 剥离 PES Header，剩下的即为 H.264 裸流 (NAL Units)。
    

---

## 6. 网络传输封装 (UDP/RTP Mapping)

在网络传输中，为了提高效率，遵循 **"7包定律"**。

### 6.1 封装公式

$$\text{MTU Limit} \approx 1500 \text{ Bytes}$$

$$188 \text{ Bytes} \times 7 = 1316 \text{ Bytes}$$

### 6.2 协议栈结构

**RTP over UDP 模式 (推荐)**:

Plaintext

```
+-----------------------+ 20 Bytes
|       IP Header       |
+-----------------------+ 8 Bytes
|       UDP Header      |
+-----------------------+ 12 Bytes
|       RTP Header      | PT=33 (MP2T), Timestamp
+-----------------------+
|      TS Packet 1      | 188 Bytes
+-----------------------+
|      TS Packet 2      | 188 Bytes
+-----------------------+
|          ...          |
+-----------------------+
|      TS Packet 7      | 188 Bytes
+-----------------------+
Total Length: 1356 Bytes
```

---

## 7.C++解析
```cpp
#include <iostream>
#include <fstream>
#include <vector>
#include <cstdint>
#include <iomanip>
#include <map>

// ==========================================
// 常量定义
// ==========================================
const int TS_PACKET_SIZE = 188;
const uint8_t SYNC_BYTE = 0x47;

// 常见的 Stream Type 定义 (ISO/IEC 13818-1)
const std::map<uint8_t, std::string> STREAM_TYPES = {
    {0x0F, "AAC Audio"},
    {0x1B, "H.264 Video"},
    {0x24, "H.265 (HEVC) Video"},
    {0x03, "MP3 Audio"},
    {0x02, "MPEG-2 Video"}
};

// ==========================================
// 数据结构
// ==========================================

// TS 包头部结构
struct TsHeader {
    bool tei;           // Transport Error Indicator
    bool pusi;          // Payload Unit Start Indicator
    bool priority;      // Transport Priority
    uint16_t pid;       // Packet Identifier
    uint8_t tsc;        // Transport Scrambling Control
    uint8_t afc;        // Adaptation Field Control
    uint8_t cc;         // Continuity Counter
    bool has_af;        // 是否包含自适应区
    bool has_payload;   // 是否包含负载
};

// 全局状态管理
struct TsContext {
    uint16_t pmt_pid = 0;       // 从 PAT 中解析出的 PMT PID
    uint16_t video_pid = 0;     // 从 PMT 中解析出的 Video PID
    uint16_t audio_pid = 0;     // 从 PMT 中解析出的 Audio PID
    bool pat_found = false;
    bool video_found = false;
};

// ==========================================
// 辅助函数
// ==========================================
void printHex(const uint8_t* data, size_t size) {
    for (size_t i = 0; i < size; ++i) {
        std::cout << std::hex << std::setw(2) << std::setfill('0') << (int)data[i] << " ";
    }
    std::cout << std::dec << std::endl;
}

// ==========================================
// 核心解析逻辑
// ==========================================

// 1. 解析 4 字节的 TS 头部
TsHeader parseTsHeader(const uint8_t* packet) {
    TsHeader h;
    
    // Byte 1: Sync Byte (这里假设外部已经检查过 0x47)
    
    // Byte 2 & 3
    // TEI: bit 7 of Byte 2
    h.tei = (packet[1] & 0x80) != 0;
    // PUSI: bit 6 of Byte 2
    h.pusi = (packet[1] & 0x40) != 0;
    // PID: lower 5 bits of Byte 2 + all 8 bits of Byte 3
    h.pid = ((packet[1] & 0x1F) << 8) | packet[2];

    // Byte 4
    // TSC: bit 7-6
    h.tsc = (packet[3] & 0xC0) >> 6;
    // AFC: bit 5-4
    h.afc = (packet[3] & 0x30) >> 4;
    // CC: bit 3-0
    h.cc = (packet[3] & 0x0F);

    // AFC 判读
    // 01: Payload only, 10: AF only, 11: Both
    h.has_af = (h.afc == 2 || h.afc == 3);
    h.has_payload = (h.afc == 1 || h.afc == 3);

    return h;
}

// 2. 解析 PCR (仅展示逻辑，不深究 33bit 运算)
void parseAdaptationField(const uint8_t* packet) {
    // AF Length 位于 Byte 4
    uint8_t af_len = packet[4];
    if (af_len > 0) {
        uint8_t flags = packet[5];
        bool pcr_flag = (flags & 0x10) != 0; // Bit 4
        if (pcr_flag && af_len >= 7) {
            // PCR 占用 6 字节 (Byte 6-11)
            // 这里简单打印高位，证明解析成功
            uint64_t pcr_base_high = packet[6]; 
            // std::cout << "    [PCR Found]" << std::endl; 
        }
    }
}

// 3. 解析 PAT 表 (PID = 0)
void parsePAT(const uint8_t* payload, int payload_len, TsContext& ctx) {
    // 略过 pointer field (如果是 PUSI=1，第一个字节是 pointer field)
    // 简化处理：假设 pointer field = 0，即表头紧跟其后
    int offset = 0;
    
    // Table ID (1B) -> 必须是 0x00
    if (payload[offset] != 0x00) return;

    // 略过 Section Length 等固定头 (8 Bytes)
    // 实际数据从 offset + 8 开始循环
    // Program Number (2B) | Reserved (3bits) | PID (13bits)
    offset += 8;
    
    while (offset < payload_len - 4) { // -4 是 CRC32
        uint16_t program_num = (payload[offset] << 8) | payload[offset+1];
        uint16_t pmt_pid = ((payload[offset+2] & 0x1F) << 8) | payload[offset+3];
        
        if (program_num != 0) { // Program 0 是 Network Information
            ctx.pmt_pid = pmt_pid;
            ctx.pat_found = true;
            std::cout << ">>> [PAT Parsed] Found PMT PID: " << ctx.pmt_pid << " for Program: " << program_num << std::endl;
            return; // 找到一个就返回演示
        }
        offset += 4;
    }
}

// 4. 解析 PMT 表 (PID = pmt_pid)
void parsePMT(const uint8_t* payload, int payload_len, TsContext& ctx) {
    int offset = 0;
    if (payload[offset] != 0x02) return; // PMT Table ID 必须是 0x02

    // Section Length (12 bits)
    uint16_t section_len = ((payload[offset+1] & 0x0F) << 8) | payload[offset+2];
    
    // 跳过固定头 12 字节 (包含 PCR PID, Program Info Length 等)
    // 实际上 Program Info Length 在 offset+10, 11
    uint16_t program_info_len = ((payload[offset+10] & 0x0F) << 8) | payload[offset+11];
    
    offset += 12 + program_info_len; // 跳到 Stream Loop 开始
    
    // 循环读取流信息
    // Stream Type (1B) | Reserved | Elementary PID (13bits) | ES Info Len (12bits)
    while (offset < section_len + 3 - 4) { // +3 是因为 section_len 从 byte 3 开始算
        uint8_t stream_type = payload[offset];
        uint16_t elem_pid = ((payload[offset+1] & 0x1F) << 8) | payload[offset+2];
        uint16_t es_info_len = ((payload[offset+3] & 0x0F) << 8) | payload[offset+4];
        
        std::string type_name = "Unknown";
        if (STREAM_TYPES.count(stream_type)) {
            type_name = STREAM_TYPES.at(stream_type);
        }

        std::cout << "    -> [Stream] PID: " << elem_pid << " | Type: 0x" 
                  << std::hex << (int)stream_type << std::dec << " (" << type_name << ")" << std::endl;

        // 简单的逻辑记录视频 PID
        if (stream_type == 0x1B || stream_type == 0x24) {
            ctx.video_pid = elem_pid;
            ctx.video_found = true;
        } else if (stream_type == 0x0F) {
            ctx.audio_pid = elem_pid;
        }

        offset += 5 + es_info_len;
    }
}

// ==========================================
// 主程序
// ==========================================
int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <ts_file>" << std::endl;
        return 1;
    }

    std::ifstream file(argv[1], std::ios::binary);
    if (!file) {
        std::cerr << "Error opening file." << std::endl;
        return 1;
    }

    uint8_t buffer[TS_PACKET_SIZE];
    TsContext ctx;
    int packet_count = 0;

    std::cout << "Starting TS Parser..." << std::endl;

    while (file.read(reinterpret_cast<char*>(buffer), TS_PACKET_SIZE)) {
        // 1. 检查 Sync Byte
        if (buffer[0] != SYNC_BYTE) {
            std::cerr << "Sync byte loss at packet " << packet_count << "! Resync needed." << std::endl;
            // 简单处理：跳过 1 字节尝试重找 (实际项目需要更复杂的 Resync 逻辑)
            file.seekg(-187, std::ios::cur); 
            continue;
        }

        // 2. 解析头部
        TsHeader header = parseTsHeader(buffer);
        
        // 计算 Payload 的偏移量
        int payload_offset = 4;
        if (header.has_af) {
            uint8_t af_len = buffer[4];
            parseAdaptationField(buffer);
            payload_offset += (1 + af_len); // 1 byte for AF length field itself
        }

        // 3. 处理 Payload (如果有)
        if (header.has_payload && payload_offset < TS_PACKET_SIZE) {
            uint8_t* payload = buffer + payload_offset;
            int payload_len = TS_PACKET_SIZE - payload_offset;

            // 处理 PSI 表格时的 Pointer Field (仅当 PUSI=1 且是 PSI 表时)
            bool is_psi = (header.pid == 0 || header.pid == ctx.pmt_pid);
            if (header.pusi && is_psi) {
                int pointer_field = payload[0];
                payload += (1 + pointer_field);
                payload_len -= (1 + pointer_field);
            }

            // 路由逻辑
            if (header.pid == 0) {
                // PAT 表
                parsePAT(payload, payload_len, ctx);
            } 
            else if (ctx.pat_found && header.pid == ctx.pmt_pid) {
                // PMT 表 (只解析一次以减少刷屏)
                if (!ctx.video_found) {
                    std::cout << ">>> [PMT Parsed] Found PMT Section." << std::endl;
                    parsePMT(payload, payload_len, ctx);
                }
            }
            else if (ctx.video_found && header.pid == ctx.video_pid) {
                // 视频数据包
                if (header.pusi) {
                     // PUSI=1, 这通常是 PES 头的开始
                     // PES Start Code Prefix 是 00 00 01
                     if (payload[0] == 0x00 && payload[1] == 0x00 && payload[2] == 0x01) {
                         std::cout << "[Video PES Start] PID: " << header.pid 
                                   << " Seq: " << packet_count << std::endl;
                     }
                }
            }
        }

        packet_count++;
        // 限制输出，防止刷屏
        if (packet_count > 5000) break;
    }

    std::cout << "Parsed " << packet_count << " packets." << std::endl;
    return 0;
}
```
### 代码详细解读

#### 1. 位运算提取字段 (Bitwise Operations)

这是解析二进制协议的核心。在 `parseTsHeader` 函数中：

C++

```
// 提取 13位的 PID
// buffer[1] 的后 5 位 + buffer[2] 的全 8 位
h.pid = ((packet[1] & 0x1F) << 8) | packet[2];
```

- `& 0x1F`: 掩码操作，保留低 5 位，清零高 3 位。
    
- `<< 8`: 左移 8 位，腾出低位空间。
    
- `|`: 或操作，拼接两个字节。
    

#### 2. 自适应区 (Adaptation Field) 跳过逻辑

在 `main` 循环中，我们计算 `payload_offset`：

C++

```
if (header.has_af) {
    uint8_t af_len = buffer[4]; // 第4字节是 AF 长度
    payload_offset += (1 + af_len); 
}
```

这完美体现了 TS 包**变长头部**的特性。如果有 PCR，payload 就会被往后挤。

#### 3. Pointer Field 处理

这是 TS 解析最容易踩的坑。 在 PAT 或 PMT 表中，如果 `PUSI=1`，Payload 的**第一个字节不是数据，而是偏移量 (Pointer Field)**。

C++

```
if (header.pusi && is_psi) {
    int pointer_field = payload[0];
    payload += (1 + pointer_field); // 跳过之前的数据（通常是上一个包剩下的尾巴）
}
```

代码中处理了这一逻辑，确保 `parsePAT` 读到的是真正的 Table ID。

## 7. 专家速查总结

1. **Sync Byte 丢失**：如果找不到 `0x47`，说明文件损坏或不是 TS 格式。
    
2. **CC Error**：如果 PID 相同的连续包 `CC` 字段不连续（如 4 跳到 6），说明发生**丢包**。
    
3. **找关键帧**：先找 PID=视频PID，再找 `PUSI=1`，再看 Payload 里是否有 H.264 SPS/PPS 或 IDR 标志。
    
4. **填充数据**：如果看到 PID=`0x1FFF`，这是空包，仅用于维持固定码率（CBR），解析时应直接丢弃。