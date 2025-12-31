
---

# 🛠️ FFmpeg 生产级命令

> ℹ️ **说明**：FFmpeg 源码编译、流媒体分析、转码处理及推拉流实战。适用于流媒体服务器开发、嵌入式音视频调试及日常运维。


## 🔍 第一章：流媒体探测与分析 (Probe & Analysis)

_在处理问题前，先看清流的本质。`ffprobe` 是最好的医生。_

### 1. 深度探测 (输出 JSON)

将流的详细信息（封装格式、流信息、具体帧数据）导出为 JSON，便于程序解析。

```shell
# 探测所有信息
ffprobe -v error \
    -show_format -show_streams -show_data -show_error -show_frames -show_packets \
    -of json \
    input.mp4 > info.json

# 🎯 指定只探测视频流 (v:0)
ffprobe -v error \
    -select_streams v:0 \
    -show_format -show_streams -show_frames \
    -of json \
    input.mp4 > video_info.json
```

### 2. 关键帧与 PTS 分析 (CSV 格式)

快速检查 I 帧间隔（GOP）或时间戳是否均匀。

```shell
# 输出：编码帧序号, PTS时间, 是否关键帧, 帧类型(I/P/B)
ffprobe -v error \
    -select_streams v:0 \
    -show_entries frame=coded_picture_number,pts_time,key_frame,pict_type \
    -of csv=p=0 \
    input.mp4

# 实时流帧率/帧类型监测
ffprobe -v error \
    -select_streams v:0 \
    -show_entries frame=pict_type \
    -of csv=p=0 \
    rtsp://192.168.20.78:554/live/stream
```

---

## 📡 第二章：推流与拉流实战 (Streaming)

_RTSP/RTMP 协议的生产级应用，重点关注低延迟和参数兼容性。_

### 1. RTSP 低延迟推流 (H.264)

> ⚠️ **注意**：`sps_id=0:pps_id=0` 对于某些对 SPS/PPS 位置敏感的播放器（如 WebRTC 网关）非常关键。

```shell
# 循环推流 (-stream_loop -1) + 读取本地帧率 (-re)
ffmpeg -re -stream_loop -1 -i input.mp4 \
    -c:v libx264 \
    -x264-params "sps_id=0:pps_id=0" \
    -bf 0 \             # 🚫 禁用 B 帧 (降低解码延迟)
    -f rtsp rtsp://192.168.207.129/live/test
```

### 2. 音频转码推流 (多编码格式)

测试不同音频编码在 RTSP 中的兼容性（AAC, G.711 PCMA/PCMU）。

```shell
# 1. AAC (最常用)
nohup ffmpeg -re -stream_loop -1 -i input.mp4 \
    -c:v copy -c:a aac -ar 44100 -ac 2 -b:a 64k \
    -f rtsp rtsp://server/live/aac_stream >/dev/null 2>&1 &

# 2. G.711 mu-law (安防常用)
ffmpeg -re -stream_loop -1 -i input.mp4 \
    -c:v copy -c:a pcm_mulaw -ar 8000 -ac 1 -b:a 64k \
    -f rtsp rtsp://server/live/pcmu_stream

# 3. G.711 A-law
ffmpeg -re -stream_loop -1 -i input.mp4 \
    -c:v copy -c:a pcm_alaw -ar 8000 -ac 1 -b:a 64k \
    -f rtsp rtsp://server/live/pcma_stream
```

### 3. FFplay 播放器调试

用于验证服务端流是否正常，支持不同协议层。

```shell
# ✅ 推荐：强制使用 TCP 传输 (防止 UDP 丢包花屏)
ffplay -rtsp_transport tcp -x 1920 -y 1080 rtsp://server/live/stream

# ⚡ 极速模式：无缓冲播放 (用于测试最低延迟)
ffplay -fflags nobuffer -sync video \
    rtsp://server/live/stream \
    -loglevel debug

# 🔊 播放裸流音频 (PCM) - 必须指定参数，因为 PCM 没有头部信息
ffplay -f s16le -ar 16000 -ac 1 -i raw_audio.pcm
```

---

## 🎬 第三章：转码与编辑 (Transcoding & Editing)

### 1. 视频处理 (缩放、水印、编码)

|**任务**|**命令片段**|**说明**|
|---|---|---|
|**画面缩放**|`-vf "scale=1280:720"`|强制缩放到 720p。若保持比例可用 `scale=-1:720`。|
|**水印添加**|`-vf "drawtext=..."`|需指定字体路径。表达式 `(w-text_w)/2` 实现了**居中**效果。|
|**转码 VP9**|`-c:v libvpx-vp9`|Google 开源编码，适合 Web 播放，压缩率优于 H.264。|


```shell
# 典型场景：压制 1080p H.264 (中等预设)
ffmpeg -i input.mp4 \
    -vf "scale=1920:1080" \
    -c:v libx264 -preset medium -crf 23 -bf 0 \
    -c:a copy \
    output.mp4

# 典型场景：居中红字水印 (字号 200)
ffmpeg -i input.mp4 -vf "drawtext=fontfile=/path/to/font.ttf: text='DANGER': x=(w-text_w)/2: y=(h-text_h)/2: fontsize=200: fontcolor=red" output.mp4

# 典型场景：转码 VP9 (开启多线程加速)
ffmpeg -i input.mp4 \
    -c:v libvpx-vp9 -speed 4 -row-mt 1 -tile-columns 6 \
    -crf 30 -b:v 1M \
    -c:a copy -f matroska out.mkv
```

### 2. 音频处理 (音量、混合)

```shell
# 🔊 音量检测 (找出最大音量 max_volume)
ffmpeg -i input.mp4 -t 60 -af "volumedetect" -f null -

# 🎛️ 音频混合 (Mix)
# 将两个 wav 合并，权重各 0.5，时长取最长者
ffmpeg -i 1.wav -i 2.wav \
    -filter_complex "[0:a][1:a]amix=inputs=2:duration=longest:weights=0.5|0.5" \
    out_mix.wav
```

### 3. 封装格式转换 (Muxing)

FFmpeg 的强项，`-c copy` 极速复制流，不涉及重编码。

```shell
# 📦 快速解包/封包 (无需重编码)
ffmpeg -i input.mp4 -c copy output.mkv
ffmpeg -i input.mp4 -c copy output.flv
ffmpeg -i input.mp4 -c copy output.mov

# 🎞️ 视频抽帧 (提取为图片)
ffmpeg -i input.mp4 frame_%04d.jpg

# 🔗 多文件无缝拼接 (Concat)
# 需创建 file.txt:
# file '1.mp4'
# file '2.mp4'
ffmpeg -f concat -safe 0 -i file.txt -c copy merged.mp4
```

---

## 📝 附录：核心参数详解字典

关键参数含义：

|**参数 (Flag)**|**全称/含义**|**详细解释 (Tech Detail)**|
|---|---|---|
|**`-re`**|Read Native|**推流必加**。按文件原本的帧率读取数据。如果不加，FFmpeg 会全速读取，导致流媒体服务器瞬间缓存溢出。|
|**`-stream_loop -1`**|Loop Input|输入流循环次数。`-1` 表示无限循环，常用于用短视频模拟 7x24h 直播流。|
|**`-bf 0`**|B-Frames|**低延迟必加**。禁用 B 帧（双向预测帧）。B 帧虽然压缩率高，但需要缓冲未来的帧才能解码，会增加显著延迟。|
|**`-x264-params`**|H.264 Opts|传递参数给 x264 库。如 `sps_id=0` 可强制重置参数集 ID，解决某些拼接流的解码兼容性问题。|
|**`-rtsp_transport tcp`**|RTSP via TCP|强制 RTSP 基于 TCP 传输（默认可能是 UDP）。在弱网或防火墙环境下，TCP 更稳定，无花屏。|
|**`-filter_complex`**|Complex Filter|复杂滤镜图。当有多个输入源（如音频混合、画中画）时，必须用此参数构建处理链路。|
|**`-vn` / `-an`**|No Video/Audio|剔除视频流 / 剔除音频流。|