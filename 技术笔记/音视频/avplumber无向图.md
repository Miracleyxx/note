# AVPlumber项目详细技术文档

## 1. 项目概述

AVPlumber是一个基于图(Graph)的实时媒体处理框架，允许用户通过文本API动态配置和重构处理流程。该项目基于FFmpeg的核心库(libavcodec、libavformat和libavfilter)构建，提供了比FFmpeg更灵活的媒体处理能力，尤其适合需要动态调整的实时媒体处理场景。

### 1.1 核心理念

AVPlumber的核心理念是将媒体处理表示为一个由处理节点(Node)和连接它们的边(Edge)组成的有向无环图(DAG)。每个节点代表一个特定的媒体处理功能(如解码、滤镜、编码等)，而边则代表节点间的数据流动路径，通常实现为线程安全的队列。

这种基于图的设计具有以下优势：
- **灵活性**：可以动态重新配置处理图，无需重启整个应用
- **模块化**：每个节点专注于单一功能，便于维护和扩展
- **并行处理**：不同节点可以在不同线程中并行执行
- **错误隔离**：单个节点的失败不一定导致整个处理链崩溃

### 1.2 主要功能特点

与传统的FFmpeg相比，AVPlumber提供了许多增强特性：

- **动态重配置**：处理图可以在运行时通过文本命令修改
- **一次编码多次输出**：编码后的数据可以发送到多个输出目标
- **多线程视频滤镜**：支持在多个线程中执行FFmpeg滤镜操作
- **时间戳连续性**：即使输入时间戳有跳跃，也能保持输出时间戳的连续性和音视频同步
- **容错机制**：在输入流中断时可以自动插入备用画面("slate")
- **简化开发**：简化了新视频和音频处理模块的原型设计，减少了样板代码
- **分组控制**：节点可以按功能分组，便于统一管理
- **高级播放控制**：支持暂停、恢复、定位和速度控制
- **实时监控**：提供详细的队列和处理状态统计信息

## 2. 系统架构详解

### 2.1 层次架构图

```
+-----------------------------------------------------+
|                    应用层                           |
| +---------------------+  +----------------------+   |
| |    命令行应用       |  |  作为库集成的应用    |   |
| +---------------------+  +----------------------+   |
+-----------------------------------------------------+
|                    控制层                           |
| +---------------------+  +----------------------+   |
| |   TCP命令接口       |  |     C++ API接口      |   |
| +---------------------+  +----------------------+   |
+-----------------------------------------------------+
|                    管理层                           |
| +---------------------+  +----------------------+   |
| |  节点管理           |  |    边/队列管理       |   |
| | (NodeManager)       |  |   (EdgeManager)      |   |
| +---------------------+  +----------------------+   |
+-----------------------------------------------------+
|                    核心层                           |
| +---------------------+  +----------------------+   |
| |   处理图(DAG)       |  |  实例共享对象        |   |
| | (节点+边)           |  | (InstanceShared)     |   |
| +---------------------+  +----------------------+   |
| +---------------------+  +----------------------+   |
| |   事件系统          |  |    统计系统          |   |
| | (Event/EventLoop)   |  |    (Stats)           |   |
| +---------------------+  +----------------------+   |
+-----------------------------------------------------+
|                    节点层                           |
| +----------+  +----------+  +----------+  +------+ |
| | 输入节点 |  | 处理节点 |  | 编码节点 |  | 输出 | |
| +----------+  +----------+  +----------+  +------+ |
+-----------------------------------------------------+
|                    适配层                           |
| +---------------------+  +----------------------+   |
| |   FFmpeg包装        |  |  硬件加速支持        |   |
| | (avcpp)             |  |  (CUDA等)            |   |
| +---------------------+  +----------------------+   |
+-----------------------------------------------------+
|                    底层库                           |
| +----------+  +----------+  +----------+  +------+ |
| | libavcodec| |libavformat| |libavfilter| | 其他  | |
| +----------+  +----------+  +----------+  +------+ |
+-----------------------------------------------------+
```

### 2.2 核心组件详解

#### 2.2.1 控制接口

AVPlumber提供多种控制接口：

- **TCP文本命令接口**：允许通过网络连接发送控制命令
- **命令行参数**：可以通过命令行参数控制基本行为
- **C++ API**：可以直接通过C++ API创建和管理处理图
- **脚本文件**：支持从文件中加载和执行命令序列

#### 2.2.2 图管理系统

图管理系统由两个主要组件组成：

- **NodeManager**：负责节点的创建、连接、启动和停止，以及节点分组管理
- **EdgeManager**：管理节点间的连接（队列），控制队列容量和数据流

核心功能包括：
- 节点的创建和销毁
- 节点状态管理（启动/停止）
- 节点组管理
- 处理节点间的连接关系
- 处理节点之间的数据流控制

#### 2.2.3 处理图(DAG)

处理图是一个有向无环图，由以下部分组成：

- **节点(Node)**：实现特定媒体处理功能的组件
- **边(Edge)**：连接节点的数据通道，通常是线程安全的队列
- **元数据**：跟随媒体数据的附加信息，如时间戳、格式信息等

#### 2.2.4 实例共享对象

实例共享对象系统允许不同节点之间共享状态和资源：

- **同步对象**：如暂停控制、速度控制等
- **资源对象**：如硬件加速上下文等
- **全局对象**：可以跨实例共享的对象

#### 2.2.5 事件系统

事件系统提供以下功能：

- **事件循环**：处理非阻塞节点的事件驱动执行
- **事件通知**：支持节点之间的事件通知
- **等待机制**：支持等待特定事件的发生

### 2.3 数据流模型

AVPlumber使用基于队列的数据流模型：

- 每个队列有一个写入端(上游节点)和一个读取端(下游节点)
- 队列是线程安全的，支持多线程访问
- 队列有容量限制，可以配置以平衡延迟和内存使用
- 不同队列可以传输不同类型的数据(编码包、视频帧、音频样本)

### 2.4 多线程模型

AVPlumber采用多线程模型以实现并行处理：

- **一个节点一个线程**：每个节点通常在自己的线程中运行
- **非阻塞节点**：可以在事件循环中运行，减少线程切换开销
- **线程安全队列**：使用无锁队列实现高效的线程间通信
- **事件循环**：处理非阻塞节点的事件驱动执行

## 3. 核心数据流详解

### 3.1 典型处理流程图

```
                       +----------------+
                       |    输入源      |
                       | (文件/网络流)   |
                       +----------------+
                              |
                              v
+-----------------+    +----------------+    +----------------+
|   输入节点       |    |    解复用节点   |    |   媒体包选择   |
| (input/input_rec)|--→| (demux)        |--→| (stream路由)   |
+-----------------+    +----------------+    +----------------+
                                                  |       |
                              +-----------------+ |       | +----------------+
                              |                   v       v                  |
                              |          +---------------+  +---------------+|
                              |          |  视频解码器    |  |  音频解码器    ||
                              |          | (dec_video)   |  | (dec_audio)   ||
                              |          +---------------+  +---------------+|
                              |                 |                |           |
                              |                 v                v           |
                              |          +---------------+  +---------------+|
                              |          | 视频帧处理链   |  | 音频帧处理链   ||
                              |          | (处理节点组)   |  | (处理节点组)   ||
                              |          +---------------+  +---------------+|
                              |                 |                |           |
                              |                 v                v           |
                              |          +---------------+  +---------------+|
                              |          |  视频编码器    |  |  音频编码器    ||
                              |          | (enc_video)   |  | (enc_audio)   ||
                              |          +---------------+  +---------------+|
                              |                 |                |           |
                              |                 v                v           |
                              |          +----------------------------+      |
                              +--------->|        复用节点            |      |
                                         | (mux)                     |<-----+
                                         +----------------------------+
                                                      |
                                                      v
                                         +----------------------------+
                                         |        输出节点            |
                                         | (output)                  |
                                         +----------------------------+
                                                      |
                                                      v
                                         +----------------------------+
                                         |        输出目标            |
                                         | (文件/网络流)              |
                                         +----------------------------+
```

### 3.2 数据类型流转图

```
                 +----------------+
                 |    输入节点    |
                 +----------------+
                        | av::Packet
                        v
          +---------------------------+
          |        解复用节点         |
          +---------------------------+
            |                  |
 av::Packet |                  | av::Packet
            v                  v
+------------------+  +------------------+
|   视频解码节点    |  |   音频解码节点    |
+------------------+  +------------------+
            |                  |
av::VideoFrame |                  | av::AudioSamples
            v                  v
+------------------+  +------------------+
|   视频处理节点    |  |   音频处理节点    |
+------------------+  +------------------+
            |                  |
av::VideoFrame |                  | av::AudioSamples
            v                  v
+------------------+  +------------------+
|   视频编码节点    |  |   音频编码节点    |
+------------------+  +------------------+
            |                  |
 av::Packet |                  | av::Packet
            v                  v
          +---------------------------+
          |         复用节点          |
          +---------------------------+
                        | av::Packet
                        v
                 +----------------+
                 |    输出节点     |
                 +----------------+
```

### 3.3 详细处理节点时序图

```
输入节点          解码节点          处理节点          编码节点          输出节点
   |                |                |                |                |
   |--读取媒体包---->|                |                |                |
   |                |--解码为原始帧--->|                |                |
   |                |                |--处理原始帧----->|                |
   |                |                |                |--编码为媒体包--->|
   |                |                |                |                |--写出媒体包-->
   |                |                |                |<--请求更多数据--|
   |                |                |<--请求更多数据--|                |
   |                |<--请求更多数据--|                |                |
   |<--请求更多数据--|                |                |                |
   |                |                |                |                |
   |--读取媒体包---->|                |                |                |
   |                |--解码为原始帧--->|                |                |
   |                |                |--处理原始帧----->|                |
   |                |                |                |--编码为媒体包--->|
   |                |                |                |                |--写出媒体包-->
   |                |                |                |                |
```

## 4. 节点类型与功能详解

### 4.1 输入节点详解

#### 4.1.1 input节点

功能：从URL读取媒体数据

主要参数：
- `url` (string): 媒体源URL，支持FFmpeg支持的所有协议(file、http、rtsp等)
- `timeout` (float): 数据包读取超时时间(秒)
- `initial_timeout` (float): URL打开超时时间(秒)
- `options` (dictionary): 传递给libavformat的选项

示例：
```json
{
  "url": "rtsp://example.com/stream",
  "timeout": 10.0,
  "initial_timeout": 20.0,
  "options": {
    "probesize": 5000000,
    "analyzeduration": 5000000,
    "fflags": "discardcorrupt"
  },
  "type": "input",
  "name": "input_main",
  "group": "in",
  "dst": "input_packets"
}
```

核心实现：
- 使用FFmpeg的AVFormatContext打开媒体源
- 在单独线程中读取媒体包并发送到输出队列
- 支持自动重连和错误处理

#### 4.1.2 input_rec节点

功能：支持播放和定位录制文件的特殊输入节点

除了input节点的所有参数外，还支持：
- `live_delay` (float): 实时模式下，当前显示帧与最新可用帧之间的延迟(秒)
- `start_ts` (string): 开始时间戳
- `stop_ts` (string): 结束时间戳
- `loop` (bool): 是否循环播放
- `seek_table` (string): 快速定位偏移表文件路径
- `ts_offsets` (string): 时间戳偏移文件路径

示例：
```json
{
  "url": "file:///videos/recorded.mp4",
  "type": "input_rec",
  "name": "playback",
  "group": "in",
  "dst": "playback_packets",
  "loop": true,
  "start_ts": "00:01:30",
  "stop_ts": "00:05:00"
}
```

核心实现：
- 扩展input节点功能，增加定位和循环播放支持
- 实现预读取机制以提高定位效率
- 支持与其他input_rec节点同步播放

### 4.2 解复用和解码节点

#### 4.2.1 demux节点

功能：解析容器格式，分离音视频流

主要参数：
- `routing` (dictionary): 流路由配置，指定哪个输入流发送到哪个输出队列

示例：
```json
{
  "routing": {
    "v:0": "video_in",  // 第一个视频流路由到video_in队列
    "a:0": "audio_in"   // 第一个音频流路由到audio_in队列
  },
  "type": "demux",
  "name": "demuxer",
  "group": "in",
  "src": "input_packets"
}
```

核心实现：
- 解析每个媒体包的流索引
- 根据路由表将媒体包分发到对应的输出队列
- 处理节目(Program)和流(Stream)映射关系 

#### 4.2.2 dec_video节点

功能：解码视频媒体包为原始视频帧

主要参数：
- `hw_device`: 硬件加速设备名称（可选）
- `codec_params`: 解码器参数（可选）

示例：
```json
{
  "type": "dec_video",
  "name": "video_decoder",
  "group": "in",
  "src": "video_in",
  "dst": "video_decoded",
  "hw_device": "cuda1"
}
```

核心实现：
- 根据输入媒体包信息自动选择或指定解码器
- 支持硬件加速解码（如CUDA）
- 保留和传递帧元数据（时间戳、宽高等）

#### 4.2.3 dec_audio节点

功能：解码音频媒体包为原始音频样本

主要参数：
- `codec_params`: 解码器参数（可选）

示例：
```json
{
  "type": "dec_audio",
  "name": "audio_decoder",
  "group": "in",
  "src": "audio_in",
  "dst": "audio_decoded"
}
```

核心实现：
- 根据输入媒体包信息自动选择或指定解码器
- 保留和传递样本元数据（时间戳、采样率等）

### 4.3 处理节点详解

#### 4.3.1 filter_video节点

功能：使用FFmpeg滤镜语法处理视频帧

主要参数：
- `graph`: FFmpeg滤镜图语法
- `hw_device`: 硬件加速设备名称（可选）

示例：
```json
{
  "graph": "[v:0]scale=1280:720,boxblur=2:1[v]",
  "type": "filter_video",
  "name": "video_filter",
  "group": "process",
  "src": "video_decoded",
  "dst": "video_filtered"
}
```

核心实现：
- 初始化FFmpeg滤镜图
- 支持硬件加速滤镜（取决于滤镜类型和硬件支持）
- 保留帧时间戳以维持同步

#### 4.3.2 filter_audio节点

功能：使用FFmpeg滤镜语法处理音频样本

主要参数：
- `graph`: FFmpeg音频滤镜图语法

示例：
```json
{
  "graph": "[a:0]volume=0.8,equalizer=f=1000:width_type=h:width=200:g=-10[a]",
  "type": "filter_audio",
  "name": "audio_filter",
  "group": "process",
  "src": "audio_decoded",
  "dst": "audio_filtered"
}
```

核心实现：
- 初始化FFmpeg音频滤镜图
- 处理样本时间戳以维持音视频同步

#### 4.3.3 rescale_video节点

功能：改变视频尺寸和像素格式

主要参数：
- `dst_width`: 目标宽度
- `dst_height`: 目标高度
- `dst_pixel_format`: 目标像素格式
- `flags`: 缩放算法标志

示例：
```json
{
  "dst_width": 1280,
  "dst_height": 720,
  "dst_pixel_format": "yuv420p",
  "flags": ["SWS_LANCZOS"],
  "type": "rescale_video",
  "name": "scaler",
  "group": "process",
  "src": "video_decoded",
  "dst": "video_scaled"
}
```

核心实现：
- 使用libswscale进行视频缩放和格式转换
- 优化缩放上下文重用以提高性能

#### 4.3.4 resample_audio节点

功能：改变音频采样率、声道布局和样本格式

主要参数：
- `dst_sample_rate`: 目标采样率
- `dst_channel_layout`: 目标声道布局
- `dst_sample_format`: 目标样本格式
- `compensation`: 时间补偿（毫秒）

示例：
```json
{
  "dst_sample_rate": 48000,
  "dst_channel_layout": "stereo",
  "dst_sample_format": "fltp",
  "compensation": 0,
  "type": "resample_audio",
  "name": "resampler",
  "group": "process",
  "src": "audio_decoded",
  "dst": "audio_resampled"
}
```

核心实现：
- 使用libswresample进行音频重采样和格式转换
- 管理重采样上下文以优化性能
- 支持时间补偿以解决音视频同步问题

#### 4.3.5 force_fps节点

功能：强制设置视频帧率

主要参数：
- `fps`: 目标帧率（分数形式，如"30000/1001"）

示例：
```json
{
  "fps": "24/1",
  "type": "force_fps",
  "name": "fps_adjuster",
  "group": "process",
  "src": "video_decoded",
  "dst": "video_fps_adjusted"
}
```

核心实现：
- 重新计算每帧的PTS/DTS
- 保持帧序和时长不变
- 将帧率信息传递给下游节点

#### 4.3.6 realtime节点

功能：速率限制，使输出符合实时时钟

主要参数：
- `speed`: 播放速度因子（默认1.0）
- `leak_after`: 持续有数据时，多少秒后禁用限速（可选）
- `tick_period`: 基于tick_source的抖动过滤器周期

示例：
```json
{
  "speed": 1.0,
  "type": "realtime",
  "name": "rate_limiter",
  "group": "process",
  "src": "muxed_packets",
  "dst": "realtime_packets"
}
```

核心实现：
- 根据媒体时间戳和壁钟时间控制帧发送速率
- 支持速度调整和抖动过滤
- 可以根据输入状态自动禁用限速

### 4.4 编码和复用节点详解

#### 4.4.1 enc_video节点

功能：将原始视频帧编码为压缩媒体包

主要参数：
- `codec`: 编码器名称
- `options`: 编码器选项
- `hw_device`: 硬件加速设备名称（可选）

示例：
```json
{
  "codec": "libx264",
  "options": {
    "preset": "medium",
    "b": "4M",
    "maxrate": "5M",
    "bufsize": "8M",
    "profile": "high",
    "level": "4.1"
  },
  "type": "enc_video",
  "name": "video_encoder",
  "group": "out",
  "src": "video_processed",
  "dst": "video_encoded"
}
```

核心实现：
- 初始化指定的编码器
- 配置编码参数和质量设置
- 支持硬件加速编码
- 处理时间戳以保持连续性

#### 4.4.2 enc_audio节点

功能：将原始音频样本编码为压缩媒体包

主要参数：
- `codec`: 编码器名称
- `options`: 编码器选项

示例：
```json
{
  "codec": "aac",
  "options": {
    "b": "128k",
    "ar": "48000",
    "ac": "2"
  },
  "type": "enc_audio",
  "name": "audio_encoder",
  "group": "out",
  "src": "audio_processed",
  "dst": "audio_encoded"
}
```

核心实现：
- 初始化指定的音频编码器
- 配置编码参数和质量设置
- 处理时间戳以保持与视频同步

#### 4.4.3 mux节点

功能：将音视频流合并到容器中

主要参数：
- 多个输入源（音频和视频编码流）

示例：
```json
{
  "type": "mux",
  "name": "muxer",
  "group": "out",
  "src": ["video_encoded", "audio_encoded"],
  "dst": "muxed"
}
```

核心实现：
- 将多个媒体流（音频、视频）合并到单个输出流
- 处理媒体包的时间戳确保同步
- 管理复用器上下文和流映射

### 4.5 输出节点详解

#### 4.5.1 output节点

功能：将媒体写入URL指定的目标

主要参数：
- `url`: 输出目标URL
- `format`: 输出格式（如"flv"、"mp4"等）
- `options`: 输出格式选项

示例：
```json
{
  "format": "flv",
  "url": "rtmp://streaming.example.com/live/stream1",
  "options": {
    "flvflags": "no_duration_filesize",
    "fflags": "nobuffer",
    "flush_packets": "1"
  },
  "type": "output",
  "name": "rtmp_output",
  "group": "out",
  "src": "muxed"
}
```

核心实现：
- 初始化指定格式的输出上下文
- 写入媒体头信息
- 持续写入媒体包
- 处理网络错误和重试

### 4.6 特殊节点详解

#### 4.6.1 split节点

功能：将一个输入分发到多个输出

主要参数：
- 多个输出队列名称

示例：
```json
{
  "type": "split<av::VideoFrame>",
  "name": "video_splitter",
  "group": "process",
  "src": "video_processed",
  "dst": ["output1_video", "output2_video", "preview_video"]
}
```

核心实现：
- 接收输入数据
- 克隆或引用数据到多个输出队列
- 根据数据类型处理引用计数

#### 4.6.2 sentinel节点

功能：监控和控制流状态，插入备用画面

主要参数：
- `slate`: 备用画面配置
- `timeout`: 切换到备用画面的超时时间

示例：
```json
{
  "type": "sentinel_video",
  "name": "video_sentinel",
  "group": "process",
  "src": "video_decoded",
  "dst": "video_monitored",
  "timeout": 5.0,
  "slate": {
    "type": "color",
    "width": 1920,
    "height": 1080,
    "color": "#000080",
    "text": "Stream will be back shortly"
  }
}
```

核心实现：
- 监控输入流的状态和延迟
- 在输入中断时切换到备用源
- 在输入恢复时切回主源
- 记录切换事件和状态

#### 4.6.3 packet_relay节点

功能：中继编码数据包

主要参数：
- `timebase_num` 和 `timebase_den`: 时间基准（可选）

示例：
```json
{
  "type": "packet_relay",
  "name": "relay",
  "group": "process",
  "src": "encoded_stream1",
  "dst": "relayed_stream"
}
```

核心实现：
- 转发媒体包
- 可选择性重写时间基准
- 保持元数据不变

#### 4.6.4 pause节点

功能：支持暂停和恢复播放

主要参数：
- `team`: 控制组名称

示例：
```json
{
  "type": "pause",
  "name": "video_pause",
  "group": "process",
  "src": "video_decoded",
  "dst": "video_pausable",
  "team": "playback"
}
```

核心实现：
- 响应暂停/恢复命令
- 维护时间戳连续性
- 支持团队同步暂停

## 5. 队列系统详解

### 5.1 队列类型与特性

AVPlumber使用[moodycamel::ReaderWriterQueue](https://github.com/cameron314/readerwriterqueue)实现高性能线程安全队列：

- **无锁设计**：最小化线程同步开销
- **单生产者/单消费者**：每个队列连接一个上游节点和一个下游节点
- **容量限制**：队列容量可配置，用于平衡延迟和内存使用
- **数据类型安全**：队列中的数据类型在创建时确定

### 5.2 队列数据类型详解

#### 5.2.1 av::Packet

编码的媒体数据包，包含：
- 压缩的音频或视频数据
- 时间戳(PTS/DTS)
- 流索引
- 标志(关键帧等)
- 附加元数据

#### 5.2.2 av::VideoFrame

原始视频帧，包含：
- 未压缩的像素数据
- 宽度和高度
- 像素格式
- 时间戳(PTS)
- 场序(针对隔行视频)
- 附加元数据

#### 5.2.3 av::AudioSamples

原始音频样本，包含：
- 未压缩的音频样本数据
- 采样率
- 声道布局
- 样本格式
- 时间戳(PTS)
- 附加元数据

### 5.3 队列配置详解

队列容量可通过命令进行配置：

```
queue.plan_capacity queue_name capacity
```

参数：
- `queue_name`: 队列名称，可使用`*`设置默认容量
- `capacity`: 队列最大容量(整数)

队列容量会影响系统行为：
- **较小的容量**：较低的延迟，但可能导致更频繁的节点阻塞
- **较大的容量**：更高的吞吐量，但可能增加延迟和内存使用

### 5.4 队列监控与统计

可以通过命令监控队列状态：

```
queues.stats
```

输出示例：
```
[#####..............]  1821.1  videoin
[...................]  243265  aenc1
[####................]  243265  audioin
```

输出解释：
- 队列填充百分比可视化
- 最后一个数据包或帧的时间戳
- 队列名称

## 6. 控制协议详解

### 6.1 协议概述

AVPlumber使用类似MVCP(MLT Video Control Protocol)的文本控制协议。通过TCP连接，客户端可以发送命令和接收响应。

连接建立后，服务器发送：`100 VTR READY`

客户端发送命令后，服务器响应：
- `200 OK` - 命令成功，无返回数据
- `201 OK` - 命令成功，返回数据(后跟数据行，以空行结束)
- `400 Unknown command: ...` - 未知命令
- `500 ERROR: ...` - 命令执行错误

### 6.2 节点管理命令详解

#### 6.2.1 节点创建与删除

```
node.add { ...json对象... }
```
添加节点定义到系统中，但不创建或启动它。

```
node.add_create { ...json对象... }
```
添加节点定义并创建节点实例，但不启动它。

```
node.add_start { ...json对象... }
```
添加节点定义，创建并启动节点实例。

```
node.delete name
```
删除指定名称的节点。

#### 6.2.2 节点控制命令

```
node.start name
```
启动指定名称的节点。

```
node.stop name
```
停止指定名称的节点（禁止自动重启）。

```
node.stop_wait name
```
停止指定名称的节点，并等待直到节点真正停止后才返回。

```
node.auto_restart name
```
停止节点并触发其自动重启动作。

#### 6.2.3 节点参数命令

```
node.param.set node_name param_name new_json_value
```
设置节点参数。注意：节点必须重启才能使新参数生效。

```
node.param.get node_name
```
获取节点的完整JSON配置。

```
node.param.get node_name param_name
```
获取节点的单个参数值。

```
node.object.get node_name object_name
```
获取节点中的对象。

```
node.object.set node_name object_name object_value
```
设置节点中的对象。

### 6.3 组管理命令详解

```
group.restart group
```
重启指定组中的所有节点。

```
group.stop group
```
停止指定组中的所有节点。

```
group.start group
```
启动指定组中的所有节点。

### 6.4 播放控制命令详解

```
pause team_name now
```
立即暂停指定团队中的播放。

```
pause team_name at timestamp
```
在指定时间戳处暂停播放。

```
resume team_name
```
恢复指定团队的播放。

```
seek team_name now timestamp
```
立即定位到指定时间戳。

```
seek team_name frame frame_number
```
定位到指定帧号。

```
speed.set team_name speed
```
设置播放速度。

### 6.5 统计与事件命令

```
stats.subscribe { ... json对象 ... }
```
订阅统计信息。

```
event.on.node.finished event_name node_name
```
当节点完成时触发事件。

```
event.wait event_name
```
等待事件触发。

### 6.6 特殊命令

```
retry command arguments ...
```
重复执行命令直到成功。

```
detach command arguments ...
```
在后台线程中执行命令，立即返回。

## 7. 实际应用案例分析

### 7.1 基本转码流水线

以下是一个基本的转码流水线，将RTSP流转换为RTMP流：

```
// 设置队列容量
queue.plan_capacity in_v 200
queue.plan_capacity in_a 200

// 输入和解复用
node.add {"url":"rtsp://example.com/stream","type":"input","name":"input","group":"in","dst":"in_mux"}
node.add {"routing":{"v:0":"in_v","a:0":"in_a"},"type":"demux","name":"demux","group":"in","src":"in_mux"}

// 视频处理
node.add {"type":"dec_video","name":"vdec","group":"in","src":"in_v","dst":"vraw"}
node.add {"dst_width":1280,"dst_height":720,"type":"rescale_video","name":"scale","group":"process","src":"vraw","dst":"vscaled"}
node.add {"codec":"libx264","options":{"preset":"ultrafast","b":"2M"},"type":"enc_video","name":"venc","group":"out","src":"vscaled","dst":"venc_out"}

// 音频处理
node.add {"type":"dec_audio","name":"adec","group":"in","src":"in_a","dst":"araw"}
node.add {"dst_sample_rate":44100,"dst_channel_layout":"stereo","type":"resample_audio","name":"resample","group":"process","src":"araw","dst":"aresampled"}
node.add {"codec":"aac","options":{"b":"128k"},"type":"enc_audio","name":"aenc","group":"out","src":"aresampled","dst":"aenc_out"}

// 复用和输出
node.add {"type":"mux","name":"muxer","group":"out","src":["venc_out","aenc_out"],"dst":"muxed"}
node.add {"format":"flv","url":"rtmp://streaming.example.com/live/stream1","type":"output","name":"output","group":"out","src":"muxed"}

// 启动处理组
group.start out
group.start process
group.start in
```

### 7.2 添加画中画效果

以下示例展示如何创建画中画效果：

```
// 第二路输入
node.add {"url":"rtsp://example.com/stream2","type":"input","name":"input2","group":"in2","dst":"in_mux2"}
node.add {"routing":{"v:0":"in_v2"},"type":"demux","name":"demux2","group":"in2","src":"in_mux2"}
node.add {"type":"dec_video","name":"vdec2","group":"in2","src":"in_v2","dst":"vraw2"}
node.add {"dst_width":320,"dst_height":180,"type":"rescale_video","name":"scale2","group":"process","src":"vraw2","dst":"vscaled2"}

// 使用filter_video创建画中画效果
node.add {
  "graph": "[0:v][1:v]overlay=main_w-overlay_w-10:main_h-overlay_h-10[v]",
  "type": "filter_video",
  "name": "pip",
  "group": "process",
  "src": ["vscaled", "vscaled2"],
  "dst": "vpip"
}

// 将pip输出连接到编码器
node.param.set venc src vpip

// 启动第二组输入
group.start in2
```

### 7.3 故障恢复与监控

以下示例展示如何使用sentinel节点实现故障恢复：

```
// 在解码和缩放之间添加sentinel节点
node.add {
  "type": "sentinel_video",
  "name": "vsent",
  "group": "process",
  "src": "vraw",
  "dst": "vmonitored",
  "timeout": 5.0,
  "slate": {
    "type": "color",
    "width": 1280,
    "height": 720,
    "color": "#000080",
    "text": "Stream will be back shortly"
  }
}

// 修改缩放节点的输入源
node.param.set scale src vmonitored

// 添加状态监控
stats.subscribe {
  "url": "http://monitoring.example.com/stats",
  "name": "stream_stats",
  "interval": 1,
  "streams": {
    "video": [
      {
        "q_pre_dec": "in_v",
        "q_post_dec": "vraw",
        "decoder": "vdec"
      }
    ],
    "audio": [
      {
        "q_pre_dec": "in_a",
        "q_post_dec": "araw",
        "decoder": "adec"
      }
    ]
  },
  "sentinel": "vsent"
}
```

## 8. 性能优化与调优指南

### 8.1 队列容量优化

队列容量设置直接影响系统的延迟和吞吐量：

```
// 输入队列较大，减少输入阻塞可能性
queue.plan_capacity in_v 200
queue.plan_capacity in_a 200

// 处理队列适中，平衡延迟和吞吐量
queue.plan_capacity vraw 20
queue.plan_capacity araw 30

// 编码后队列较小，减少输出延迟
queue.plan_capacity venc_out 5
queue.plan_capacity aenc_out 5
```

### 8.2 节点分组策略

合理的节点分组可以提高系统的可维护性和恢复能力：

```
// 输入组：包含输入、解复用和解码
node.add {"type":"input","name":"input","group":"in",...}
node.add {"type":"demux","name":"demux","group":"in",...}
node.add {"type":"dec_video","name":"vdec","group":"in",...}
node.add {"type":"dec_audio","name":"adec","group":"in",...}

// 处理组：包含各种帧处理节点
node.add {"type":"rescale_video","name":"scale","group":"process",...}
node.add {"type":"filter_video","name":"filter","group":"process",...}
node.add {"type":"resample_audio","name":"resample","group":"process",...}

// 输出组：包含编码、复用和输出
node.add {"type":"enc_video","name":"venc","group":"out",...}
node.add {"type":"enc_audio","name":"aenc","group":"out",...}
node.add {"type":"mux","name":"muxer","group":"out",...}
node.add {"type":"output","name":"output","group":"out",...}
```

### 8.3 硬件加速

利用硬件加速可以显著提高处理性能：

```
// 初始化CUDA加速器
hwaccel.init { "name": "cuda1", "type": "cuda" }

// 使用CUDA加速解码
node.add {
  "type": "dec_video",
  "name": "vdec",
  "group": "in",
  "src": "in_v",
  "dst": "vraw",
  "hw_device": "cuda1"
}

// 使用CUDA加速编码
node.add {
  "codec": "h264_nvenc",
  "options": {
    "preset": "llhp",
    "b": "4M",
    "maxrate": "5M"
  },
  "type": "enc_video",
  "name": "venc",
  "group": "out",
  "src": "vscaled",
  "dst": "venc_out",
  "hw_device": "cuda1"
}
```

### 8.4 实时处理优化

对于实时处理场景，可以使用realtime节点优化：

```
// 在复用后添加realtime节点
node.add {
  "type": "realtime",
  "name": "rtlimit",
  "group": "out",
  "src": "muxed",
  "dst": "rtmuxed",
  "leak_after": 30.0
}

// 修改输出节点的输入源
node.param.set output src rtmuxed
```

### 8.5 多输出优化

对于多输出场景，使用split节点避免重复编码：

```
// 在编码后使用split节点
node.add {
  "type": "split<av::Packet>",
  "name": "vsplit",
  "group": "out",
  "src": "venc_out",
  "dst": ["venc_out1", "venc_out2"]
}

// 添加第二个复用器和输出
node.add {"type":"mux","name":"muxer2","group":"out","src":["venc_out2","aenc_out"],"dst":"muxed2"}
node.add {"format":"flv","url":"rtmp://backup.example.com/live/stream1","type":"output","name":"output2","group":"out","src":"muxed2"}
```

## 9. 扩展开发指南

### 9.1 作为库集成AVPlumber

AVPlumber可以作为静态库集成到其他应用中：

```cpp
#include "src/avplumber.hpp"

int main() {
    // 创建AVPlumber实例
    std::unique_ptr<AVPlumber> avplumber(new AVPlumber());
    
    // 执行设置命令
    avplumber->executeCommandsFromString("node.add {...}");
    
    // 设置就绪
    avplumber->setReady();
    
    // 启动主循环
    avplumber->mainLoop();
    
    // 关闭avplumber
    avplumber->shutdown();
    
    return 0;
}
```

编译方法：
```bash
make static_library  # 生成libavplumber.a
```

### 9.2 开发自定义节点

开发自定义节点需要以下步骤：

1. 创建节点类，继承自适当的基类(如NodeAVBase)
2. 实现必要的方法(process、prepare等)
3. 注册节点类型到工厂

示例：
```cpp
// 自定义节点实现
class MyCustomNode : public NodeAVBaseIn<av::VideoFrame, av::VideoFrame> {
public:
    MyCustomNode(const json &cfg) : NodeAVBaseIn(cfg) {}
    
    virtual bool prepare() override {
        // 初始化代码
        return true;
    }
    
    virtual void process(av::VideoFrame &&frame) override {
        // 处理代码
        
        // 发送处理后的帧
        sendOutput(std::move(frame));
    }
};

// 注册节点类型
REGISTER_NODE_FACTORY("my_custom", MyCustomNode);
```

### 9.3 与其他系统集成

#### 9.3.1 OBS Studio集成

AVPlumber可以与OBS Studio集成作为源插件：

```cpp
// 在OBS插件中初始化AVPlumber
void initialize_avplumber() {
    avplumber = std::make_unique<AVPlumber>();
    
    // 设置处理图
    avplumber->executeCommandsFromString("...");
    
    // 使用tick_source将OBS和AVPlumber同步
    avplumber->executeCommandsFromString(
        "node.add {\"type\":\"filter_video\",\"name\":\"sync_filter\",\"tick_source\":\"obs\",...}"
    );
    
    avplumber->setReady();
}
```

#### 9.3.2 JACK音频系统集成

AVPlumber支持JACK音频系统集成：

```
// 添加JACK音频输入
node.add {
  "type": "jack_audio_source",
  "name": "jack_in",
  "group": "in",
  "dst": "jack_audio",
  "client_name": "avplumber",
  "connect_to": ["system:capture_1", "system:capture_2"]
}
```

## 10. 故障排除与常见问题

### 10.1 常见错误及解决方案

1. **输入连接失败**
   - 检查URL格式和可访问性
   - 增加initial_timeout参数
   - 查看详细日志

2. **解码器初始化失败**
   - 确保输入流包含兼容编解码器
   - 检查是否安装了必要的编解码器库
   - 尝试指定具体的解码器名称

3. **编码器配置错误**
   - 验证编码器选项格式和值
   - 确保编码器与输入格式兼容
   - 检查硬件加速器可用性

4. **实时处理延迟高**
   - 调整队列容量
   - 检查CPU/GPU使用率
   - 考虑使用硬件加速

5. **节点组重启循环**
   - 检查auto_restart设置
   - 确保输入源稳定
   - 添加适当的超时和重试逻辑

### 10.2 性能诊断

使用统计信息诊断性能问题：

```
stats.subscribe {
  "url": "",  // 空URL表示输出到控制台
  "name": "perf_stats",
  "interval": 1,
  "streams": {
    "video": [
      {
        "q_pre_dec": "in_v",
        "q_post_dec": "vraw",
        "decoder": "vdec"
      }
    ],
    "audio": [
      {
        "q_pre_dec": "in_a",
        "q_post_dec": "araw",
        "decoder": "adec"
      }
    ]
  }
}
```

### 10.3 调试技巧

1. **使用队列统计**
   ```
   queues.stats
   ```

2. **获取节点参数**
   ```
   node.param.get node_name
   ```

3. **使用调试节点**
   ```
   node.add {"type":"debug.print<av::VideoFrame>","name":"debug","src":"vraw","dst":"vraw_debug"}
   ```

4. **检查节点对象**
   ```
   node.object.get input streams
   ```

## 11. 总结与最佳实践

### 11.1 设计原则

AVPlumber基于以下设计原则：

1. **模块化**：每个节点专注于单一功能
2. **灵活性**：支持动态重配置处理图
3. **容错性**：节点可以独立失败和恢复
4. **性能优化**：利用多线程和硬件加速
5. **易扩展**：简化自定义节点开发

### 11.2 最佳实践

1. **合理分组节点**
   - 将功能相关的节点放在同一组
   - 利用auto_restart策略处理故障

2. **优化队列容量**
   - 输入和解码队列较大
   - 处理队列适中
   - 输出队列较小

3. **利用硬件加速**
   - 尽可能使用硬件编解码
   - 考虑GPU加速滤镜

4. **实现监控和错误处理**
   - 使用sentinel节点处理输入故障
   - 配置stats监控关键指标

5. **利用事件系统**
   - 使用事件处理异步操作
   - 在启动脚本中使用retry命令

### 11.3 应用场景建议

1. **直播转码**
   - 使用input节点连接RTMP/RTSP源
   - 添加实时监控和错误恢复
   - 使用realtime节点控制输出速率

2. **文件处理**
   - 使用input_rec节点支持定位和循环
   - 添加更复杂的滤镜效果
   - 考虑多线程处理提高速度

3. **多输出分发**
   - 使用split节点避免重复编码
   - 为不同目标配置不同编码参数
   - 实现独立输出控制

AVPlumber提供了强大而灵活的媒体处理能力，适合各种实时和非实时媒体处理场景，特别是对于需要动态重配置和高度可靠性的应用。 

## 12. 附录：系统UML图

### 12.1 类图

```
+-------------------+         +------------------+
|   NodeManager     |<>-------| InstanceShared   |
+-------------------+         +------------------+
| +createNode()     |         | +get()           |
| +startNode()      |         | +create()        |
| +stopNode()       |         +------------------+
| +startGroup()     |                 ^
| +stopGroup()      |                 |
+-------------------+         +------------------+
        ^                     | EventLoop        |
        |                     +------------------+
        |                     | +run()           |
        |                     | +stop()          |
        |                     +------------------+
+-------------------+
|   EdgeManager     |         +------------------+
+-------------------+         | Edge<T>          |
| +planCapacity()   |<>-------+------------------+
| +getInstance()    |         | +enqueue()       |
+-------------------+         | +dequeue()       |
                              | +size()          |
                              +------------------+
                                      ^
                                      |
                             +------------------+
                             | EdgeImpl<T>      |
                             +------------------+
                             | -queue           |
                             | -stats           |
                             +------------------+

+-------------------+
|     Node          |
+-------------------+
| +prepare()        |
| +start()          |
| +stop()           |
| +isRunning()      |
+-------------------+
        ^
        |
+-------------------+         +------------------+
| NodeAVBase<I,O>   |<>-------| Stats            |
+-------------------+         +------------------+
| #processLoop()    |         | +update()        |
| #sendOutput()     |         | +subscribe()     |
| #getInput()       |         +------------------+
+-------------------+
        ^
        |
+-----------------------+
| NodeInput             |-------> av::Packet
+-----------------------+
| -formatContext        |
| -interruptibleReader  |
+-----------------------+

+-----------------------+
| NodeDemux             |<------ av::Packet
+-----------------------+
| -routingTable         |-------> av::Packet
| -streamInfo           |
+-----------------------+

+-----------------------+
| NodeDecVideo          |<------ av::Packet
+-----------------------+
| -codecContext         |
| -hwDevice             |-------> av::VideoFrame
+-----------------------+

+-----------------------+
| NodeFilterVideo       |<------ av::VideoFrame
+-----------------------+
| -filterGraph          |
| -filterContext        |-------> av::VideoFrame
+-----------------------+

+-----------------------+
| NodeEncVideo          |<------ av::VideoFrame
+-----------------------+
| -codecContext         |
| -options              |-------> av::Packet
+-----------------------+

+-----------------------+
| NodeMux               |<------ av::Packet
+-----------------------+
| -formatContext        |
| -streamMappings       |-------> av::Packet
+-----------------------+

+-----------------------+
| NodeOutput            |<------ av::Packet
+-----------------------+
| -formatContext        |
| -url                  |
+-----------------------+
```

### 12.2 节点状态图

```
                     +-------------+
                     |   创建中    |
                     +-------------+
                            |
                            v
                     +-------------+
                     |   已创建    |
                     +-------------+
                            |
                            v
+-------------+      +-------------+      +-------------+
|   已停止    |<---->|   运行中    |----->|   错误状态   |
+-------------+      +-------------+      +-------------+
      |                                        |
      |                                        |
      v                                        v
+-------------+                         +-------------+
|   已销毁    |                         |   重启中    |
+-------------+                         +-------------+
                                              |
                                              v
                                        +-------------+
                                        |  重启已触发  |
                                        +-------------+
```

### 12.3 处理图示例序列图

```
Client         NodeManager      Input节点      Demux节点      DecVideo节点    EncVideo节点    Output节点
   |                |              |              |              |              |              |
   |--创建处理图---->|              |              |              |              |              |
   |                |--创建节点---->|              |              |              |              |
   |                |--创建节点------------------->|              |              |              |
   |                |--创建节点---------------------------------->|              |              |
   |                |--创建节点------------------------------------------------->|              |
   |                |--创建节点---------------------------------------------------------------->|
   |                |              |              |              |              |              |
   |--启动处理图---->|              |              |              |              |              |
   |                |--启动节点---->|              |              |              |              |
   |                |              |--开始读取数据->|              |              |              |
   |                |              |              |--分离视频流--->|              |              |
   |                |              |              |              |--解码视频帧--->|              |
   |                |              |              |              |              |--编码视频包--->|
   |                |              |              |              |              |              |--写出数据-->
   |                |              |              |              |              |              |
   |                |              |<--读取更多数据-|              |              |              |
   |                |              |              |<--需要更多包--|              |              |
   |                |              |              |              |<--需要更多帧--|              |
   |                |              |              |              |              |<--需要更多包--|
   |--停止处理图---->|              |              |              |              |              |
   |                |--停止节点---->|              |              |              |              |
   |                |--停止节点------------------->|              |              |              |
   |                |--停止节点---------------------------------->|              |              |
   |                |--停止节点------------------------------------------------->|              |
   |                |--停止节点---------------------------------------------------------------->|
   |                |              |              |              |              |              |
```

### 12.4 组和自动重启时序图

```
Client         NodeManager      组"in"          组"process"     组"out"        auto_restart
   |                |              |              |              |              |
   |--启动处理图---->|              |              |              |              |
   |                |--启动组---------------------->|              |              |
   |                |--启动组-------------------------------------------------->|
   |                |--启动组------>|              |              |              |
   |                |              |              |              |              |
   |                |              |<--节点崩溃----|              |              |
   |                |<--节点崩溃通知-|              |              |              |
   |                |--检查策略------------------------------------>|
   |                |              |              |              |              |
   |                |<--应用group策略----------------------------------------------|
   |                |              |              |              |              |
   |                |--停止组------>|              |              |              |
   |                |--重启组------>|              |              |              |
   |                |              |              |              |              |
```

### 12.5 组件交互图

```
              +---------------+
              |     Client    |
              +---------------+
                     |
                     v
+---------------+    |    +---------------+
|  TCP服务器    |<---+--->|   命令解析器   |
+---------------+         +---------------+
                               |
                               v
+---------------+    +---------------+    +---------------+
|  EdgeManager  |<-->|  NodeManager  |<-->| InstanceShared |
+---------------+    +---------------+    +---------------+
       ^                    ^                    ^
       |                    |                    |
       v                    v                    v
+---------------+    +---------------+    +---------------+
|     队列      |<-->|     节点      |<-->|   共享对象    |
+---------------+    +---------------+    +---------------+
                          |   |
                          v   v
              +---------------+---------------+
              |  解码/编码/滤镜/复用/输出等    |
              +---------------+---------------+
                          |
                          v
              +---------------+---------------+
              |        FFmpeg库/avcpp        |
              +---------------+---------------+
```

## 13. 实用工具和辅助功能

### 13.1 内置工具

AVPlumber提供了多种内置工具，辅助处理常见任务：

#### 13.1.1 命令行工具

- **avplumber**: 主程序，支持TCP控制和脚本执行
- **generate_node_list**: 生成可用节点类型列表

#### 13.1.2 脚本示例

在examples目录下提供了多种实用脚本示例：

- 基本转码流程
- 多输出配置
- 备用输入源切换
- 实时监控配置
- HLS/DASH打包示例

### 13.2 网络控制工具

可以使用标准网络工具控制AVPlumber：

```bash
# 使用netcat连接到AVPlumber
nc localhost 20200

# 使用curl发送命令
echo 'node.param.get input' | curl -d @- http://localhost:20200/command
```

### 13.3 统计信息可视化

AVPlumber支持将统计信息发送到外部系统：

```
stats.subscribe {
  "url": "http://grafana-push-gateway:9091/metrics/job/avplumber",
  "name": "metrics",
  "interval": 1,
  "streams": {...}
}
```

支持多种格式：
- JSON格式（标准）
- Prometheus格式（适用于监控系统）
- 自定义格式（通过回调函数）

## 14. 实战开发案例

### 14.1 直播转码服务

以下是一个完整的直播转码服务配置：

```
// 输入和解复用
queue.plan_capacity in_v 200
queue.plan_capacity in_a 200

node.add {"url":"rtmp://live.example.com/stream/input","type":"input","name":"input","group":"in","dst":"in_mux"}
node.add {"routing":{"v:0":"in_v","a:0":"in_a"},"type":"demux","name":"demux","group":"in","src":"in_mux"}

// 添加sentinel监控和备用画面
node.add {"type":"dec_video","name":"vdec","group":"in","src":"in_v","dst":"vraw"}
node.add {
  "type": "sentinel_video",
  "name": "vsent",
  "group": "process",
  "src": "vraw",
  "dst": "vmonitored",
  "timeout": 5.0,
  "slate": {
    "type": "color",
    "width": 1920,
    "height": 1080,
    "color": "#000080",
    "text": "Live stream will resume shortly"
  }
}

// 视频处理流程
node.add {"dst_width":1920,"dst_height":1080,"type":"rescale_video","name":"scale_1080p","group":"process","src":"vmonitored","dst":"v1080p"}
node.add {"dst_width":1280,"dst_height":720,"type":"rescale_video","name":"scale_720p","group":"process","src":"vmonitored","dst":"v720p"}
node.add {"dst_width":854,"dst_height":480,"type":"rescale_video","name":"scale_480p","group":"process","src":"vmonitored","dst":"v480p"}

// 视频编码 - 多种分辨率
node.add {"codec":"libx264","options":{"preset":"veryfast","b":"5M","maxrate":"6M","bufsize":"8M","profile":"high","level":"4.1"},"type":"enc_video","name":"venc_1080p","group":"out","src":"v1080p","dst":"v1080p_enc"}
node.add {"codec":"libx264","options":{"preset":"veryfast","b":"3M","maxrate":"4M","bufsize":"5M","profile":"high","level":"4.0"},"type":"enc_video","name":"venc_720p","group":"out","src":"v720p","dst":"v720p_enc"}
node.add {"codec":"libx264","options":{"preset":"veryfast","b":"1M","maxrate":"1.5M","bufsize":"2M","profile":"main","level":"3.1"},"type":"enc_video","name":"venc_480p","group":"out","src":"v480p","dst":"v480p_enc"}

// 音频处理
node.add {"type":"dec_audio","name":"adec","group":"in","src":"in_a","dst":"araw"}
node.add {"dst_sample_rate":48000,"dst_channel_layout":"stereo","type":"resample_audio","name":"resample_high","group":"process","src":"araw","dst":"a_high"}
node.add {"dst_sample_rate":44100,"dst_channel_layout":"stereo","type":"resample_audio","name":"resample_low","group":"process","src":"araw","dst":"a_low"}

// 音频编码 - 两种码率
node.add {"codec":"aac","options":{"b":"192k"},"type":"enc_audio","name":"aenc_high","group":"out","src":"a_high","dst":"a_high_enc"}
node.add {"codec":"aac","options":{"b":"128k"},"type":"enc_audio","name":"aenc_low","group":"out","src":"a_low","dst":"a_low_enc"}

// 复用和输出 - 多个输出
node.add {"type":"mux","name":"muxer_1080p","group":"out","src":["v1080p_enc","a_high_enc"],"dst":"muxed_1080p"}
node.add {"type":"mux","name":"muxer_720p","group":"out","src":["v720p_enc","a_high_enc"],"dst":"muxed_720p"}
node.add {"type":"mux","name":"muxer_480p","group":"out","src":["v480p_enc","a_low_enc"],"dst":"muxed_480p"}

// RTMP输出
node.add {"format":"flv","url":"rtmp://stream.example.com/live/high","type":"output","name":"output_1080p","group":"out","src":"muxed_1080p"}
node.add {"format":"flv","url":"rtmp://stream.example.com/live/medium","type":"output","name":"output_720p","group":"out","src":"muxed_720p"}
node.add {"format":"flv","url":"rtmp://stream.example.com/live/low","type":"output","name":"output_480p","group":"out","src":"muxed_480p"}

// 添加监控
stats.subscribe {
  "url": "http://monitoring.example.com/stats",
  "name": "livestream",
  "interval": 1,
  "streams": {
    "video": [
      {
        "q_pre_dec": "in_v",
        "q_post_dec": "vraw",
        "decoder": "vdec"
      }
    ],
    "audio": [
      {
        "q_pre_dec": "in_a",
        "q_post_dec": "araw",
        "decoder": "adec"
      }
    ]
  },
  "sentinel": "vsent"
}

// 启动处理组
group.start out
group.start process
group.start in
```

### 14.2 文件编辑与回放服务

以下是一个文件编辑与回放服务配置：

```
// 输入和解复用
node.add {"url":"file:///videos/source.mp4","type":"input_rec","name":"input","group":"in","dst":"in_mux","seek_table":"file:///videos/source.mp4.seek","ts_offsets":"file:///videos/source.mp4.ts","team":"playback"}
node.add {"routing":{"v:0":"in_v","a:0":"in_a"},"type":"demux","name":"demux","group":"in","src":"in_mux"}

// 解码
node.add {"type":"dec_video","name":"vdec","group":"in","src":"in_v","dst":"vraw"}
node.add {"type":"dec_audio","name":"adec","group":"in","src":"in_a","dst":"araw"}

// 添加暂停控制
node.add {"type":"pause","name":"vpause","group":"process","src":"vraw","dst":"vpaused","team":"playback"}
node.add {"type":"pause","name":"apause","group":"process","src":"araw","dst":"apaused","team":"playback"}

// 添加速度控制
node.add {"type":"speed","name":"vspeed","group":"process","src":"vpaused","dst":"vspeeded","team":"playback"}
node.add {"type":"speed","name":"aspeed","group":"process","src":"apaused","dst":"aspeeded","team":"playback"}

// 视频处理
node.add {"graph":"[v:0]drawtext=text='Edited Version':fontsize=48:fontcolor=white:x=(w-text_w)/2:y=h-120[v]","type":"filter_video","name":"watermark","group":"process","src":"vspeeded","dst":"vfiltered"}

// 编码
node.add {"codec":"libx264","options":{"preset":"medium","crf":"23"},"type":"enc_video","name":"venc","group":"out","src":"vfiltered","dst":"venc_out"}
node.add {"codec":"aac","options":{"b":"192k"},"type":"enc_audio","name":"aenc","group":"out","src":"aspeeded","dst":"aenc_out"}

// 复用和输出
node.add {"type":"mux","name":"muxer","group":"out","src":["venc_out","aenc_out"],"dst":"muxed"}
node.add {"format":"mp4","url":"file:///videos/output.mp4","type":"output","name":"output","group":"out","src":"muxed"}

// 启动处理组
group.start out
group.start process
group.start in

// 设置播放控制
speed.set playback 1.0
```

使用此配置，可以通过以下命令控制播放：

```
// 暂停
pause playback now

// 恢复
resume playback

// 调整播放速度
speed.set playback 2.0

// 定位到特定时间点
seek playback now 00:10:30

// 定位到特定帧
seek playback frame 1500
```
