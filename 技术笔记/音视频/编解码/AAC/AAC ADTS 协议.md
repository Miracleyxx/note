
---

# 📘 AAC ADTS 协议：位流级封装

> ℹ️ 核心定义：ADTS (Audio Data Transport Stream) 头部通常为 7 字节 (56 bits)（无 CRC）或 9 字节（有 CRC）。
> 
 7 字节无 CRC 的标准封装，这是 RTMP/FLV/HLS 直播中最常用的格式。

## 🧩 1. ADTS 7字节位图详解 (The Bit Layout)

这是代码编写的**绝对依据**。必须清楚这 56 个 bit 是如何分布在 `uint8_t header[7]` 中的。

### 📊 字节结构总览表

|字段名 (Field Name)|起始比特 (Bit Offset)|位数 (Bits)|描述 / 含义|
|---|---|---|---|
|**ADTS 固定头 (Fixed Header) - 7 字节**||||
|`syncword`|0|12|**同步字:** 恒为 `0xFFF` (全 1)。用于在数据流中定位 ADTS 帧的开始。|
|`ID`|12|1|**MPEG 标识:** `0` = MPEG-4，`1` = MPEG-2。|
|`layer`|13|2|**层级:** 对 AAC 总是 `00`。|
|`protection_absent`|15|1|**保护缺席:** `1` 表示**无** CRC 校验 (头长 7 字节)，`0` 表示**有** CRC 校验 (头长 9 字节)。|
|`profile_ObjectType`|16|2|**Profile (对象类型):** 基于 Audio Object Type (AOT) **减 1**。主要表示基础 Profile: - `00`(0): AAC Main (AOT 1) - `01`(1): AAC LC (AOT 2) - `10`(2): AAC SSR (AOT 3) - `11`(3): AAC LTP (AOT 4) 或 保留 _不能直接表示 HE-AAC(AOT 5) 等_|
|`sampling_frequency_index`|18|4|**采样率索引:** 同 ASC 索引表，但 ADTS 中不允许 `0xf`。 - `0x3`: 48000 Hz - `0x4`: 44100 Hz - `0x8`: 16000 Hz - ... (其他见 ASC 表)|
|`private_bit`|22|1|**私有比特:** 标准未使用，通常为 `0`。|
|`channel_configuration`|23|3|**声道配置索引:** - `1`: Mono - `2`: Stereo - `3`: 3 Channels - ... - `7`: 7.1 Channels (`0` 在 ADTS 中非法或保留)|
|`original_copy`|26|1|**原始/拷贝:** `1` = 原始, `0` = 拷贝。|
|`home`|27|1|**主页 (Home):** `1` = 原始, `0` = 拷贝。|
|**ADTS 可变部分 (Variable Part) - 位于固定头内**||||
|`copyright_identification_bit`|28|1|**版权标识位:** `1` = 有版权。|
|`copyright_identification_start`|29|1|**版权开始位:** `1` = 版权材料起始。|
|`aac_frame_length`|30|13|**AAC 帧长度:** **非常重要！** 表示**整个 ADTS 帧** (头+数据+[CRC]) 的总字节数。最大 8191 字节。|
|`adts_buffer_fullness`|43|11|**ADTS 缓冲区充满度:** `0x7FF` (全 1) 表示 VBR 码流。CBR 时指示解码器缓冲状态。|
|`number_of_raw_data_blocks_in_frame`|54|2|**帧内原始数据块数:** 值为实际块数**减 1**。`00` 表示 1 个 AAC Raw Data Block (最常见)。|
|**ADTS CRC 校验 (Optional) - 2 字节**|||**(仅当 `protection_absent == 0` 时存在)**|
|`crc_check`|56|16|**CRC 校验码:** 覆盖 ADTS 帧（除 CRC 自身外）的 16 位校验和。|

### 📊 字节结构概述表

| **字节序号**   | **包含字段 (Fields)**                                  | **位宽分布 (Bit Width)**                  | **关键二进制值/说明**                  |
| ---------- | -------------------------------------------------- | ------------------------------------- | ------------------------------ |
| **Byte 0** | **Syncword (High)**                                | `8`                                   | 必须全是 `1` (`0xFF`)              |
| **Byte 1** | **Sync(Low)** / ID / Layer / Prot                  | `4` / `1` / `2` / `1`                 | 通常为 `1111 0 00 1` = **`0xF1`** |
| **Byte 2** | Profile / FreqIdx / Private / Chan(H)              | `2` / `4` / `1` / `1`                 | 混合字段，需移位拼接                     |
| **Byte 3** | Chan(L) / Orig / Home / Copy / CopySt / **Len(H)** | `2` / `1` / `1` / `1` / `1` / **`2`** | **Len 开始跨字节**                  |
| **Byte 4** | **Frame Length (Middle)**                          | **`8`**                               | 长度的中间 8 位                      |
| **Byte 5** | **Frame Length (Low)** / Buffer(H)                 | **`3`** / `5`                         | 长度结束，Buffer 开始                 |
| **Byte 6** | Buffer(L) / NumBlocks                              | `6` / `2`                             | 通常结尾是 `111111 00` (`0xFC`)     |
|            |                                                    |                                       |                                |
	**总头长:** 7 字节 (56 bits) 或 9 字节 (72 bits)。

---

## ⚙️ 2. 核心字段与跨字节逻辑 (Specs & Logic)

代码中最晦涩的部分是**跨字节字段**（特别是 `aac_frame_length`）。

### 2.1 关键字段定义

|**字段名**|**长度**|**说明**|**代码中的处理**|
|---|---|---|---|
|**Syncword**|12 bit|帧同步标识，固定 `0xFFF`。|`header[0]=0xFF`, `header[1]|
|**Profile**|2 bit|`AOT - 1`。AAC LC 原值是 2，这里填 1 (`01`)。|位于 Byte 2 高位，需 `<< 6`|
|**Freq Index**|4 bit|采样率下标。44.1k=4 (`0100`), 48k=3 (`0011`)。|位于 Byte 2 中位，需 `<< 2`|
|**Channel**|3 bit|声道数。双声道=2 (`010`)。|**跨越 Byte 2 和 Byte 3**|
|**Frame Len**|13 bit|**最坑的点**。ADTS头+数据总长。|**跨越 Byte 3, 4, 5 三个字节**|

### 2.2 `aac_frame_length` (13 bits) 的位操作图解

假设帧长为 13 bit 的整数 `Len`。我们需要把它切成三段塞进 `header`：

- **Byte 3 (低 2 位)**: 存放 `Len` 的**第 12, 11 位** (最高 2 位)。
    
    - 逻辑: `len >> 11`
        
- **Byte 4 (全 8 位)**: 存放 `Len` 的**第 10 到 3 位** (中间 8 位)。
    
    - 逻辑: `len >> 3`
        
- **Byte 5 (高 3 位)**: 存放 `Len` 的**第 2 到 0 位** (最低 3 位)。
    
    - 逻辑: `(len & 0x7) << 5` (因为这3位在 Byte 5 的高位)
        

---

## 💻 3. 生产级封装代码 (Implementation)


```C++
#include <vector>
#include <cstdint>
#include <iostream>

/**
 * @brief 生成 7 字节 ADTS 头
 * * @param profile AOT 类型 (AAC LC = 1, HE-AAC = 4 等，注意是 AOT-1)
 * @param freqIdx 采样率索引 (3=48000, 4=44100, 见标准表)
 * @param channelCfg 声道配置 (1=单声道, 2=双声道)
 * @param packetLen ADTS 帧总长度 = (7字节头 + AAC裸流长度)
 * @param packet 输出缓冲区，必须预先分配至少 7 字节
 */
void adts_header(uint8_t profile, uint8_t freqIdx, uint8_t channelCfg, 
                 uint32_t packetLen, uint8_t* packet) 
{
    // --- Byte 0 ---
    // [1111 1111] Syncword 前 8 位
    packet[0] = 0xFF; 

    // --- Byte 1 ---
    // [1111] Syncword 后 4 位
    // [0]    MPEG ID (0=MPEG4, 1=MPEG2)
    // [00]   Layer (固定 00)
    // [1]    Protection Absent (1=无CRC, 0=有CRC)
    // 二进制: 1111 0 00 1 = 0xF1
    packet[1] = 0xF1; 

    // --- Byte 2 ---
    // [00]   Profile (2 bit) -> 移位到 7-6 位
    // [0000] FreqIdx (4 bit) -> 移位到 5-2 位
    // [0]    Private (1 bit) -> 固定 0
    // [0]    Channel (High 1 bit) -> 声道配置的第 3 位 (最高位)
    packet[2] = ((profile & 0x03) << 6) +  // 取 Profile 低2位，左移6
                ((freqIdx & 0x0F) << 2) +  // 取 FreqIdx 低4位，左移2
                ((channelCfg >> 2) & 0x01); // 取 Channel 第3位(高位)，放到最后

    // --- Byte 3 ---
    // [00]   Channel (Low 2 bits) -> 声道配置的低 2 位，移位到 7-6 位
    // [0]    Original (1 bit) -> 0
    // [0]    Home (1 bit) -> 0
    // [0]    Copy (1 bit) -> 0
    // [0]    Copy Start (1 bit) -> 0
    // [00]   Frame Length (High 2 bits) -> 长度的第 12,11 位，移位到 1-0 位
    packet[3] = ((channelCfg & 0x03) << 6) + // Channel 低2位左移到头部
                ((packetLen >> 11) & 0x03);  // Length 右移11位，取最后2位

    // --- Byte 4 ---
    // [0000 0000] Frame Length (Middle 8 bits) -> 长度的第 10-3 位
    packet[4] = (packetLen >> 3) & 0xFF; // 右移3位，截断取8位

    // --- Byte 5 ---
    // [000]  Frame Length (Low 3 bits) -> 长度的第 2-0 位，移位到 7-5 位
    // [11111] Buffer Fullness (High 5 bits) -> 0x7FF 的高 5 位
    // 0x7FF 是可变码率(VBR)的标志码
    packet[5] = ((packetLen & 0x07) << 5) + // Length 低3位左移到头部
                0x1F;                       // Buffer Fullness 高5位 (11111)

    // --- Byte 6 ---
    // [111111] Buffer Fullness (Low 6 bits) -> 0x7FF 的低 6 位
    // [00]     Number of Raw Data Blocks -> (1帧 - 1) = 0
    packet[6] = 0xFC; // 1111 1100
}
```

### 💻 调用示例

```C++
int main() {
    // 假设：AAC LC(1), 44.1kHz(4), Stereo(2)
    // 裸流长度 414 字节 -> 总长 421 (0x1A5)
    uint32_t aac_len = 414;
    uint32_t total_len = aac_len + 7;
    
    uint8_t adts[7];
    adts_header(1, 4, 2, total_len, adts);

    // 预期输出 Byte 3-5 逻辑验证:
    // Len = 421 (000 11010 0101)
    // Byte 3 尾部应为 00 (0)
    // Byte 4 应为 00110100 (0x34 = 52)
    // Byte 5 头部应为 101 (5)
    
    printf("ADTS Header: ");
    for(int i=0; i<7; i++) printf("%02X ", adts[i]);
    printf("\n");
    return 0;
}
```

---

## 🔍 4. 解析器实现 (Parser Logic)

解析是封装的逆过程。架构师需要关注如何从字节流中**还原**数值。

> 💡 **技巧**：解析代码通常使用 `(buffer[i] & mask) << shift` 的方式将分散的位拼回来。

### 详细解析代码

```C++
bool parse_adts_header(uint8_t* buf, int size, uint32_t &profile, uint32_t &freqIdx, uint32_t &channel, uint32_t &len) {
    if(size < 7) return false;

    // 1. 检查 Syncword (FFF)
    // buf[0] 必须是 FF
    // buf[1] 的高4位必须是 F (即 & 0xF0 == 0xF0)
    if (buf[0] != 0xFF || (buf[1] & 0xF0) != 0xF0) {
        return false;
    }

    // 2. 解析 Profile (Byte 2 的高2位)
    // buf[2]: [PP][FFFF][P][C] -> 右移6位取出 [PP]
    profile = (buf[2] & 0xC0) >> 6; 

    // 3. 解析 Freq Index (Byte 2 的中间4位)
    // buf[2]: [PP][FFFF][P][C] -> 0x3C 掩码取出 [FFFF], 右移2位
    freqIdx = (buf[2] & 0x3C) >> 2;

    // 4. 解析 Channel (跨 Byte 2 和 Byte 3)
    // Byte 2 最低位 [C]
    // Byte 3 最高2位 [CC]
    // 拼接: (Byte2最后1位 << 2) | (Byte3前2位 >> 6)
    channel = ((buf[2] & 0x01) << 2) | ((buf[3] & 0xC0) >> 6);

    // 5. 解析 Frame Length (跨 Byte 3, 4, 5)
    // 逻辑：
    // Part 1: Byte 3 的最后2位 (作为 Len 的 12,11 位) -> buf[3]&0x03 -> 左移11
    // Part 2: Byte 4 的全部8位 (作为 Len 的 10-3 位)  -> buf[4]      -> 左移3
    // Part 3: Byte 5 的前3位   (作为 Len 的 2-0 位)   -> buf[5]&0xE0 -> 右移5
    len = ((buf[3] & 0x03) << 11) | 
          (buf[4] << 3) | 
          ((buf[5] & 0xE0) >> 5);

    return true;
}
```

---

## 🧐 5. 附录：采样率索引表 (Frequency Index)

这是 `freqIdx` 参数的查表依据，写死在代码里时务必核对。

|**Index (Bin)**|**Frequency (Hz)**|**常用场景**|
|---|---|---|
|`0011` (3)|**48000**|影视、DVD|
|`0100` (4)|**44100**|CD、MP3、大部分直播|
|`0101` (5)|32000|语音、广播|
|`0110` (6)|24000|低质量语音|
|`1000` (8)|16000|VoIP|
