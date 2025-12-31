## `NetEq`介绍

```C++
/*
 * Copyright (c) 2012 The WebRTC project authors. All Rights Reserved.
 * ... (版权声明)
 */

#ifndef API_NETEQ_NETEQ_H_
#define API_NETEQ_NETEQ_H_

#include <stddef.h>  // 提供 size_t 访问权限.

#include <map>
#include <string>
#include <vector>

#include "absl/types/optional.h"
#include "api/audio_codecs/audio_codec_pair_id.h"
#include "api/audio_codecs/audio_decoder.h"
#include "api/audio_codecs/audio_format.h"
#include "api/rtp_headers.h"
#include "api/scoped_refptr.h"

namespace webrtc {

// 前向声明，减少头文件依赖
class AudioFrame;
class AudioDecoderFactory;
class Clock;

// ============================================================================
// 统计信息结构体定义
// ============================================================================

// NetEq 网络统计信息。
// 这里的统计数据主要反映当前的缓冲状态和最近的处理操作。
struct NetEqNetworkStatistics {
  uint16_t current_buffer_size_ms;    // 当前抖动缓冲区的大小（毫秒）。
  uint16_t preferred_buffer_size_ms;  // 目标（理想）缓冲区大小（毫秒），由 NetEq 根据网络抖动自动计算。
  uint16_t jitter_peaks_found;        // 1 表示因检测到抖动峰值而增加了额外延迟；否则为 0。
  
  // 以下比率通常以 Q14 格式表示（即数值 16384 代表 1.0 或 100%）。
  uint16_t expand_rate;         // 通过“扩展”（Expand，即丢包隐藏/PLC）插入的合成音频占原始流的比例。
  uint16_t speech_expand_rate;  // 通过“扩展”插入的合成**语音**占原始流的比例。
  uint16_t preemptive_rate;     // 通过“抢占式扩展”（Pre-emptive expand，即拉伸音频使播放变慢）插入的数据比例。通常用于增加缓冲水位。
  uint16_t accelerate_rate;     // 通过“加速”（Accelerate，即压缩音频使播放变快）移除的数据比例。通常用于降低缓冲水位。
  uint16_t secondary_decoded_rate;    // 来自 FEC（前向纠错）或 RED（冗余编码）解码的数据比例。
  uint16_t secondary_discarded_rate;  // 丢弃的 FEC/RED 数据比例。

  // 数据包等待时间的统计信息。
  // 即：从数据包到达 NetEq 到被解码取出的时间间隔。
  int mean_waiting_time_ms;   // 平均等待时间
  int median_waiting_time_ms; // 中位数等待时间
  int min_waiting_time_ms;    // 最小等待时间
  int max_waiting_time_ms;    // 最大等待时间
};

// NetEq 生命周期内的累积统计信息。
// 这些指标从 NetEq 创建开始累积，永远不会重置。
struct NetEqLifetimeStatistics {
  // 下面的统计数据对应于 WebRTC 统计规范中的字段：
  // https://w3c.github.io/webrtc-stats/#dom-rtcinboundrtpstreamstats
  uint64_t total_samples_received = 0; // 接收到的总样本数。
  uint64_t concealed_samples = 0;      // 隐藏（PLC）生成的样本数。
  uint64_t concealment_events = 0;     // 发生丢包隐藏事件的次数。
  uint64_t jitter_buffer_delay_ms = 0; // 抖动缓冲区的累积延迟（用于计算平均延迟）。
  uint64_t jitter_buffer_emitted_count = 0; // 从抖动缓冲区输出的计数（用于计算平均延迟）。
  uint64_t jitter_buffer_target_delay_ms = 0; // 目标延迟的累积值。
  uint64_t inserted_samples_for_deceleration = 0; // 为了减速（增加延迟）而插入的样本数。
  uint64_t removed_samples_for_acceleration = 0;  // 为了加速（减少延迟）而移除的样本数。
  uint64_t silent_concealed_samples = 0; // 生成静音隐藏样本的数量（通常在长时间丢包后的处理）。
  uint64_t fec_packets_received = 0;     // 接收到的 FEC 数据包数量。
  uint64_t fec_packets_discarded = 0;    // 丢弃的 FEC 数据包数量。
  
  // 以下统计数据不是规范的一部分。
  uint64_t delayed_packet_outage_samples = 0; // 因包到达太晚而导致的样本丢失数。
  
  // 相对包到达延迟的总和。
  // 由于很难测量端到端的绝对延迟，这里报告相对延迟。
  // 定义为：相比于第一个接收到的包（假设延迟为0），当前包的到达延迟。
  // 为了避免时钟漂移，“第一个”包可能会动态调整。
  uint64_t relative_packet_arrival_delay_ms = 0;
  uint64_t jitter_buffer_packets_received = 0; // 抖动缓冲区接收到的包数量。
  
  // “中断”定义为持续至少 150 毫秒的丢包隐藏事件。
  // 下面两个统计数据计算此类事件的次数和总时长。
  int32_t interruption_count = 0;
  int32_t total_interruption_duration_ms = 0;
};

// 描述 NetEq 执行的操作和内部状态的指标。
struct NetEqOperationsAndState {
  // 这些样本计数器是累积的，不会重置。
  uint64_t preemptive_samples = 0; // 抢占式扩展产生的样本数。
  uint64_t accelerate_samples = 0; // 加速操作处理的样本数。
  
  // 缓冲区清空的次数（通常在发生严重错误或流重置时发生）。
  uint64_t packet_buffer_flushes = 0;
  
  // 被丢弃的主要（Primary）数据包数量。
  uint64_t discarded_primary_packets = 0;
  
  // 以下统计数据不是累积的，反映当前状态。
  // 最后一个被解码的数据包的等待时间。
  uint64_t last_waiting_time_ms = 0;
  // 当前包缓冲区和同步缓冲区的大小总和（毫秒）。
  uint64_t current_buffer_size_ms = 0;
  // 当前帧的大小（毫秒）。
  uint64_t current_frame_size_ms = 0;
  // 标志：指示下一个数据包是否已经可用。
  bool next_packet_available = false;
};

// ============================================================================
// NetEq 主接口类
// ============================================================================
class NetEq {
 public:
  // 配置结构体，用于在创建 NetEq 时设置参数。
  struct Config {
    Config();
    Config(const Config&);
    Config(Config&&);
    ~Config();
    Config& operator=(const Config&);
    Config& operator=(Config&&);

    std::string ToString() const; // 打印配置信息的辅助函数。

    int sample_rate_hz = 16000;  // 初始采样率。会随着输入数据自动改变。
    bool enable_post_decode_vad = false; // 是否启用解码后的语音活动检测 (VAD)。
    size_t max_packets_in_buffer = 200;  // 缓冲区中允许的最大包数。
    int max_delay_ms = 0; // 最大延迟限制（毫秒），0 表示不限制。
    int min_delay_ms = 0; // 最小延迟限制（毫秒），0 表示无额外限制。
    bool enable_fast_accelerate = false; // 是否启用快速加速模式（更激进地追赶时间）。
    bool enable_muted_state = false;     // 是否启用静音状态（长时间无数据时输出静音而不是持续生成噪音）。
    bool enable_rtx_handling = false;    // 是否在 NetEq 内部处理 RTX（重传）包。
    absl::optional<AudioCodecPairId> codec_pair_id; // 编解码器对 ID（用于统计）。
    bool for_test_no_time_stretching = false;  // 仅供测试：禁用时间拉伸（加速/减速）。
    // 给 NetEq 的输出增加额外的延迟，不影响抖动或丢包行为。
    // 主要用于测试音视频同步。值必须是 10ms 的非负倍数。
    int extra_output_delay_ms = 0;
  };

  enum ReturnCodes { kOK = 0, kFail = -1 };

  // NetEq 内部决定的操作类型。
  enum class Operation {
    kNormal,            // 正常解码播放。
    kMerge,             // 融合操作（通常在丢包恢复后，将生成的数据与新数据平滑连接）。
    kExpand,            // 扩展/PLC（没数据了，造一些数据出来）。
    kAccelerate,        // 加速（数据积压了，通过移除波形周期来快速播放）。
    kFastAccelerate,    // 快速加速。
    kPreemptiveExpand,  // 抢占式扩展（为了增加缓冲水位，通过拉伸波形来慢速播放）。
    kRfc3389Cng,        // 播放舒适噪音 (CNG)。
    kRfc3389CngNoPacket,// CNG 状态，但没有收到新的 SID 包。
    kCodecInternalCng,  // 解码器内部产生的 CNG。
    kDtmf,              // 播放 DTMF 音（电话拨号音）。
    kUndefined,
  };

  // NetEq 操作后的最终模式/状态。
  enum class Mode {
    kNormal,
    kExpand,
    kMerge,
    kAccelerateSuccess,    // 加速成功。
    kAccelerateLowEnergy,  // 因为能量低而加速（通常效果更好）。
    kAccelerateFail,       // 加速失败（可能找不到合适的拼接点）。
    kPreemptiveExpandSuccess,
    kPreemptiveExpandLowEnergy,
    kPreemptiveExpandFail,
    kRfc3389Cng,
    kCodecInternalCng,
    kCodecPlc,             // 解码器自带的丢包隐藏。
    kDtmf,
    kError,
    kUndefined,
  };

  // GetDecoderFormat 的返回类型。
  struct DecoderFormat {
    int sample_rate_hz;
    int num_channels;
    SdpAudioFormat sdp_format; // SDP 格式描述（如 "opus", "pcmu" 等）。
  };

  // [工厂方法] 创建一个新的 NetEq 对象。
  // `config` 参数仅在调用期间有效。
  // 需要传入时钟对象和解码器工厂（用于创建具体的音频解码器）。
  static NetEq* Create(
      const NetEq::Config& config,
      Clock* clock,
      const rtc::scoped_refptr<AudioDecoderFactory>& decoder_factory);

  virtual ~NetEq() {}

  // [输入/Push] 向 NetEq 插入一个新的 RTP 数据包。
  // `rtp_header`: 解析后的 RTP 头信息（包含序列号、时间戳等）。
  // `payload`: 负载数据。
  // 成功返回 0，失败返回 -1。
  virtual int InsertPacket(const RTPHeader& rtp_header,
                           rtc::ArrayView<const uint8_t> payload) = 0;

  // [输入/Push] 通知 NetEq 到达了一个空负载的数据包。
  // 通常用于网络探测包，它们使用与音频包相同的序列号流。
  virtual void InsertEmptyPacket(const RTPHeader& rtp_header) = 0;

  // [输出/Pull] 指示 NetEq 交付 10 毫秒的音频数据。
  // 数据会被写入 `audio_frame`。
  // 如果成功，会更新 audio_frame 中的 data_, speech_type_, num_channels_ 等字段。
  // 
  // `muted`: 输出参数。如果启用了 enable_muted_state，在长时间扩展后可能会被置为 true。
  //          此时 audio_frame 中的数据不应被使用，应视为全 0。
  // `current_sample_rate_hz`: 可选输出参数，返回当前采样率。
  // `action_override`: 可选参数，用于测试，强制 NetEq 执行特定操作（如强制 Expand）。
  virtual int GetAudio(
      AudioFrame* audio_frame,
      bool* muted,
      int* current_sample_rate_hz = nullptr,
      absl::optional<Operation> action_override = absl::nullopt) = 0;

  // 替换当前使用的解码器集合。
  virtual void SetCodecs(const std::map<int, SdpAudioFormat>& codecs) = 0;

  // 注册负载类型 (Payload Type)。
  // 将 `rtp_payload_type` (如 111) 与音频格式 (如 Opus) 关联。
  // NetEq 会在需要时根据此信息实例化解码器。
  virtual bool RegisterPayloadType(int rtp_payload_type,
                                   const SdpAudioFormat& audio_format) = 0;

  // 移除指定的负载类型。
  virtual int RemovePayloadType(uint8_t rtp_payload_type) = 0;

  // 移除所有负载类型。
  virtual void RemoveAllPayloadTypes() = 0;

  // 设置包缓冲区的最小延迟（毫秒）。
  // 即使信道条件允许更低的延迟，NetEq 也会保持至少这个延迟。
  // 这常用于实现固定延迟播放（例如在直播场景中）。
  virtual bool SetMinimumDelay(int delay_ms) = 0;

  // 设置包缓冲区的最大延迟（毫秒）。
  // 即使信道条件很差需要更多缓冲，延迟也不会超过此值（会导致丢包率上升）。
  virtual bool SetMaximumDelay(int delay_ms) = 0;

  // 设置基础最小延迟（毫秒）。
  // SetMinimumDelay 设置的值不能低于这个基础值。
  virtual bool SetBaseMinimumDelayMs(int delay_ms) = 0;

  // 获取当前的基础最小延迟。
  virtual int GetBaseMinimumDelayMs() const = 0;

  // 获取当前的目标延迟（毫秒）。
  // 这包括了通过 SetMinimumDelay 请求的额外延迟。
  virtual int TargetDelayMs() const = 0;

  // 获取当前的平滑总延迟（毫秒）。
  // 包含包缓冲区和同步缓冲区的延迟，并经过平滑处理以过滤短期抖动波动。
  // 在 DTX/CNG（静音）期间，延迟统计不会更新。
  virtual int FilteredCurrentDelayMs() const = 0;

  // 将当前的网络统计信息写入 `stats`。
  // 调用后，内部的统计状态会重置（清零）。
  virtual int NetworkStatistics(NetEqNetworkStatistics* stats) = 0;

  // 获取当前的网络统计信息，但不重置内部状态。
  virtual NetEqNetworkStatistics CurrentNetworkStatistics() const = 0;

  // 获取生命周期统计信息的副本。这些统计永远不会重置。
  virtual NetEqLifetimeStatistics GetLifetimeStatistics() const = 0;

  // 获取关于执行操作和内部状态的统计信息。永远不会重置。
  virtual NetEqOperationsAndState GetOperationsAndState() const = 0;

  // 启用解码后 VAD（语音活动检测）。
  // 启用后，如果信号不包含语音，GetAudio() 将返回 kOutputVADPassive。
  virtual void EnableVad() = 0;

  // 禁用解码后 VAD。
  virtual void DisableVad() = 0;

  // 返回上一次 GetAudio() 调用交付的样本的 RTP 时间戳。
  // 如果没有有效时间戳，返回空。
  virtual absl::optional<uint32_t> GetPlayoutTimestamp() const = 0;

  // 返回上一次 GetAudio() 产生的音频采样率 (Hz)。
  virtual int last_output_sample_rate_hz() const = 0;

  // 获取指定负载类型的解码器信息。如果未注册则返回空。
  virtual absl::optional<DecoderFormat> GetDecoderFormat(
      int payload_type) const = 0;

  // 清空包缓冲区和同步缓冲区。
  // 此时所有缓存的数据都会丢失，通常在从错误恢复或重置流时调用。
  virtual void FlushBuffers() = 0;

  // 启用 NACK (Negative Acknowledgement, 丢包重传请求)。
  // 并设置 NACK 列表的最大大小。
  virtual void EnableNack(size_t max_nack_list_size) = 0;

  // 禁用 NACK。
  virtual void DisableNack() = 0;

  // 获取需要重传的包的 RTP 序列号列表。
  // `round_trip_time_ms`: 当前的往返时间 (RTT) 估计值，用于决定哪些包值得重传（如果 RTT 太大，重传回来可能已经过期了）。
  virtual std::vector<uint16_t> GetNackList(
      int64_t round_trip_time_ms) const = 0;

  // 返回上一次 GetAudio 调用中被解码的包的时间戳列表。
  // 主要用于测试。
  virtual std::vector<uint32_t> LastDecodedTimestamps() const = 0;

  // 返回同步缓冲区中尚未播放的音频长度（毫秒）。
  // 主要用于测试。
  virtual int SyncBufferSizeMs() const = 0;

  // 返回上一次的操作类型（Operation）。
  virtual int LastOperation() const = 0;

  // 设置最大播放速度（相对于正常速度的倍数）。
  // 例如，可以限制加速操作不让声音听起来太快。
  virtual void SetMaxSpeed(double speed) const = 0;

};

}  // namespace webrtc
#endif  // API_NETEQ_NETEQ_H_
```

