## 🏗️ 第一章：源码构建 (Compilation)

_针对自定义裁剪、调试分析的编译配置。_

### 1. 调试模式构建 (Debug Build)

> 💡 **用途**：开发阶段使用，关闭优化，开启符号表，便于 GDB 调试。

```shell
# 设置依赖库路径 (根据实际情况调整)
export PKG_CONFIG_PATH=/opt/ffmpeg3rd/x264/lib:/opt/ffmpeg3rd/opus/lib:/opt/ffmpeg3rd/x265/lib:/opt/ffmpeg3rd/vpx/lib

./configure \
    --prefix=./ffmpeg_lib_debug \
    --enable-debug=3 \          # 开启最高级别调试信息
    --disable-optimizations \   # 禁用编译器优化(防止指令重排影响调试)
    --disable-asm \             # 禁用汇编优化(便于单步跟踪)
    --disable-stripping \       # 禁止剥离符号表
    --enable-libfreetype \
    --enable-libfontconfig \
    --enable-libfribidi \
    --enable-libxml2 \
    --enable-libharfbuzz
```

### 2. 生产环境构建 (Release Build)

> 💡 **用途**：线上部署，追求极致性能和小体积。

```shell
./configure \
    --prefix="/opt/ffmpeg_release" \
    --enable-gpl \              # 启用 GPL 协议(必须，如果使用了 x264/x265)
    --env='ENV=override' \
    --enable-asm \              # 启用汇编加速 (性能关键)
    --enable-small \            # 优化二进制大小
    --enable-pthreads \         # 启用多线程
    --pkg-config="pkg-config --static" \
    --extra-libs="-lpthread" \
    --enable-libx264 \
    --enable-libx265 \
    --enable-libopus \
    --enable-libvpx \
    --disable-doc               # 不编译文档，节省时间
```

---
