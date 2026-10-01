# TermVid · 终端 ASCII 视频播放器

TermVid 把视频文件的每一帧解码成 ASCII 字符画，在终端里实时渲染播放。它把"看视频"变成"在命令行里看字符动画"，适合在远程服务器、复古终端或纯文本环境下欣赏视频。

## 特性

| 特性 | 说明 |
| --- | --- |
| 实时 ASCII 渲染 | 将 MP4/AVI/... 等视频逐帧解码成字符画，在终端里播放 |
| 零分配帧处理 | 环形缓冲区在启动时一次性分配，播放过程不再 `malloc`/`free` |
| SIMD 加速 | 像素→亮度映射提供 SSSE3 向量化路径 |
| 跨平台 | Linux / macOS / WSL，CMake 构建 |
| 依赖少 | 核心仅依赖 FFmpeg 的 libav* 系列 |
| 可选音频 | `-DWITH_AUDIO` 链接 PortAudio 播放原声 |
| 可选导出 | `--gif` 把 ASCII 动画导出为 GIF 分享 |

## 依赖

- 编译器：gcc / clang，支持 C11
- FFmpeg 开发库（核心依赖，必需）：`libavformat`、`libavcodec`、`libswscale`、`libavutil`
- PortAudio（可选，仅 `WITH_AUDIO`）：`portaudio`、`libswresample`
- CMake ≥ 3.10

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y cmake ffmpeg libavformat-dev libavcodec-dev \
     libswscale-dev libavutil-dev portaudio19-dev libswresample-dev
```

### macOS (Homebrew)

```bash
brew install cmake ffmpeg portaudio
```

## 构建

项目使用 CMake 构建。最小 `CMakeLists.txt` 示例：

```cmake
cmake_minimum_required(VERSION 3.10)
project(termvid C)

set(CMAKE_C_STANDARD 11)

find_package(PkgConfig REQUIRED)
pkg_check_modules(FFMPEG REQUIRED IMPORTED_TARGET
    libavformat libavcodec libswscale libavutil)

add_executable(termvid termvid.c)
target_link_libraries(termvid PkgConfig::FFMPEG)

# 可选：开启 SIMD 加速
# target_compile_definitions(termvid PRIVATE USE_SIMD)
# target_compile_options(termvid PRIVATE -mssse3)

# 可选：开启音频
# target_compile_definitions(termvid PRIVATE WITH_AUDIO)
# target_link_libraries(termvid PkgConfig::FFMPEG portaudio)
```

构建命令：

```bash
mkdir -p build && cd build
cmake ..
cmake --build .
```

## 使用方法

```bash
termvid [选项] <视频文件>
```

### 选项

| 选项 | 说明 |
| --- | --- |
| `-w <n>` | ASCII 网格宽度（默认：终端列数） |
| `-m <mode>` | 颜色模式 `gray` \| `256` \| `true`（默认 `256`） |
| `-f <fps>` | 目标帧率（默认：视频原始帧率） |
| `--gif <out.gif>` | 把 ASCII 动画导出为 GIF（不渲染到终端） |
| `--gif-delay <cs>` | GIF 帧间隔，单位 1/100 秒（默认 4） |
| `-h` | 显示帮助 |

### 示例

```bash
# 默认彩色（256 色）播放
termvid movie.mp4

# 灰度字符播放，宽度 120
termvid -m gray -w 120 movie.mp4

# 24 位真彩播放，固定 30fps
termvid -m true -f 30 movie.mp4

# 导出为 GIF（不渲染终端）
termvid --gif out.gif --gif-delay 5 movie.mp4
```

## 交互控制

播放过程中可用键盘控制：

| 按键 | 功能 |
| --- | --- |
| `空格` | 暂停 / 继续 |
| `←` / `→` | 快退 / 快进 10 秒 |
| `q` | 退出 |

> 提示：播放时终端会进入原始模式（raw mode），退出后自动恢复光标与回显。

## 颜色模式

TermVid 支持三种渲染模式：

- **gray**：纯灰度字符画，关闭所有 ANSI 颜色，兼容性最强。
- **256**：映射到 xterm 256 色立方体（`16..231`），色彩与性能均衡（默认）。
- **true**：24 位真彩色（`\033[38;2;R;G;Bm`），色彩最准但输出体积最大。

字符亮度对照表（`CHARSET`）共 10 级，由暗到亮：

```
 .:-=+*#%@
```

## GIF 导出

使用 `--gif` 时，程序把每帧 ASCII 字符用内置 8×8 点阵字体光栅化为位图，逐帧写入 GIF89a。导出要点：

- 自带干净、可验证的 **GIF LZW 编码器**（已用 Pillow 逐帧解码交叉验证）。
- 颜色超过 256 色时自动量化到 6×6×6 立方体（最多 216 色）。
- 逻辑屏幕尺寸与光栅尺寸严格一致（每字符 8×8 像素），避免严格解码器裁切。
- 结束时写入 trailer，确保文件完整可播放。

> 注意：GIF 模式不会渲染到终端，也不会进入交互控制，仅做导出。程序退出时会正确写 trailer，否则文件会损坏。

## 实现要点

- **环形缓冲区**：`RING_FRAMES = 3` 个 RGB24 缓冲在启动时 `av_malloc` 一次性分配，配合 `g_ring_head` / `g_ring_tail` 读写指针实现零运行中分配；解码满则覆盖最旧帧。
- **像素→亮度**：`Y = (77R + 150G + 29B) >> 8`（整数化的 0.299/0.587/0.114 权重）。`USE_SIMD` 时走 SSSE3 路径，每次处理 5 像素，尾部不足部分回退标量实现。
- **帧率控制**：基于 `av_gettime_relative()` 的目标时间轴调度，`frame_us = 1e6 / fps`，动态 `usleep` 对齐下一帧时刻。
- **跳转（seek）**：`←/→` 调用 `seek_to()`，`av_seek_frame(..., AVSEEK_FLAG_BACKWARD)` 后清空环形缓冲并允许继续读取。
- **优雅退出**：`SIGINT` 置 `g_quit`；GIF 模式下会正确写 trailer 再退出，避免文件损坏。

## 限制

- ASCII 网格上限：`MAX_W = 512`，`MAX_H = 512`；超出会被裁剪。
- 字符宽高比按约 2:1 估算（`rows = cols * aspect / 2`），非精确几何校正。
- 音频为可选功能，默认构建不包含。

## 许可证

本项目基于 [BSD 3-Clause 许可证](LICENSE) 开源。详见仓库根目录的 `LICENSE` 文件。
