# Gralloc Buffer Usage 标志位

> Gralloc 的 usage 标志位系统，定义图形缓冲区的预期使用场景，驱动内存类型和属性选择。

## Meta
- **ID:** KB-ANDROID-GFX-gralloc-002
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 8.0+（Gralloc 4.0），Android 12+（AIDL），Android 14+（Gralloc 5.0 扩展）
- **关键词:** buffer usage, GRALLOC_USAGE, AHARDWAREBUFFER_USAGE, GPU_RENDER_TARGET, GPU_TEXTURE, COMPOSER_OVERLAY, PROTECTED, CPU_READ, CPU_WRITE, VIDEO_DECODER, CAMERA, sensor, frontend
- **前置知识:** [→ KB-ANDROID-GFX-gralloc-001]
- **关联条目:** [→ KB-ANDROID-GFX-surfaceflinger-002] [→ KB-ANDROID-GFX-hwcomposer-001]

## 概述

Gralloc Buffer Usage 是一组 bitmask 标志位，用于告知 Gralloc HAL 缓冲区的预期使用场景。Gralloc 根据 usage 组合决定内存类型（ION/CMA/Secure）、缓存策略（Cached/Uncached）、物理连续性（Contiguous/Scatter-gather）等关键属性。Usage 标志位分为两大体系：C++ 侧的 `GRALLOC_USAGE_*`（`<gralloc.h>`）和 NDK 侧的 `AHARDWAREBUFFER_USAGE_*`（`<hardware_buffer.h>`），两者有映射关系。正确的 usage 设置对性能和功耗至关重要——错误的 usage 可能导致 GPU/Display 无法访问缓冲区或产生不必要的 cache flush 开销。

## 架构/调用链

### Usage 标志位分类

```
Gralloc Buffer Usage
    │
    ├── GPU 相关
    │   ├── GRALLOC_USAGE_HW_RENDER (0x100) — GPU 渲染目标
    │   ├── GRALLOC_USAGE_HW_TEXTURE (0x200) — GPU 纹理采样
    │   └── GRALLOC_USAGE_HW_COMPOSER (0x400) — HWC 合成
    │
    ├── 显示相关
    │   ├── GRALLOC_USAGE_HW_FB (0x800) — Framebuffer (已废弃)
    │   └── GRALLOC_USAGE_EXTERNAL_DISP (0x90) — 外部显示
    │
    ├── CPU 相关
    │   ├── GRALLOC_USAGE_SW_READ_RARELY (0x1) — CPU 偶尔读
    │   ├── GRALLOC_USAGE_SW_READ_OFTEN (0x2) — CPU 经常读
    │   ├── GRALLOC_USAGE_SW_WRITE_RARELY (0x4) — CPU 偶尔写
    │   ├── GRALLOC_USAGE_SW_WRITE_OFTEN (0x8) — CPU 经常写
    │   └── GRALLOC_USAGE_PROTECTED (0x4000) — DRM 保护
    │
    ├── 多媒体相关
    │   ├── GRALLOC_USAGE_HW_VIDEO_ENCODER (0x10000) — 视频编码
    │   ├── GRALLOC_USAGE_HW_VIDEO_DECODER (0x40000) — 视频解码
    │   └── GRALLOC_USAGE_HW_CAMERA_WRITE (0x20000) — 相机写入
    │
    └── 传感器/特殊
        ├── GRALLOC_USAGE_SENSOR_DIRECT_DATA (0x8000000) — 传感器数据
        └── GRALLOC_USAGE_FRONTEND (0x10000000) — 前端渲染
```

### Usage → 内存属性映射

```
Usage 组合                    →  内存类型选择
─────────────────────────────────────────────────
GPU_RENDER_TARGET             →  Contiguous, Uncached, Write-combine
GPU_TEXTURE                   →  ION (可能非连续), Tiling
COMPOSER_OVERLAY              →  Contiguous (DMA 需要)
VIDEO_DECODER                 →  ION (厂商定制格式)
PROTECTED                     →  Secure Memory (TEE)
CPU_READ_OFTEN | CPU_WRITE    →  Cached (带 CPU cache)
SENSOR_DIRECT_DATA            →  CMA (低延迟, 连续)
```

## 关键接口

### 数据结构
- `enum AHARDWAREBUFFER_USAGE` — NDK 侧 usage 标志位
  - 关键值:
    - `AHARDWAREBUFFER_USAGE_GPU_FRAMEBUFFER` — GPU framebuffer
    - `AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE` — GPU 纹理
    - `AHARDWAREBUFFER_USAGE_GPU_COLOR_OUTPUT` — GPU 渲染输出
    - `AHARDWAREBUFFER_USAGE_COMPOSER_OVERLAY` — HWC overlay
    - `AHARDWAREBUFFER_USAGE_PROTECTED_CONTENT` — DRM 保护
    - `AHARDWAREBUFFER_USAGE_VIDEO_ENCODE` — 视频编码
    - `AHARDWAREBUFFER_USAGE_VIDEO_DECODE` — 视频解码
    - `AHARDWAREBUFFER_USAGE_CPU_READ_OFTEN` — CPU 经常读
    - `AHARDWAREBUFFER_USAGE_CPU_WRITE_OFTEN` — CPU 经常写
  - 源码: `frameworks/native/include/android/hardware_buffer.h`

- `struct AHardwareBuffer_Desc` — 缓冲区描述符（NDK）
  - 关键字段:
    - `width`, `height` — 尺寸
    - `format` — 像素格式
    - `layers` — 图层数
    - `usage` — usage bitmask
    - `stride` — 行跨度（输出参数）
  - 源码: `frameworks/native/include/android/hardware_buffer.h`

## 实现要点

### 常见 Usage 组合
| 场景 | Usage 组合 | 说明 |
|------|-----------|------|
| App 渲染 | `RENDER_TARGET \| COMPOSER_OVERLAY` | GPU 渲染 + HWC 显示 |
| SurfaceFlinger FBTarget | `RENDER_TARGET \| COMPOSER_OVERLAY` | GPU 合成结果 |
| 视频播放 | `VIDEO_DECODER \| COMPOSER_OVERLAY` | 解码器输出 + HWC 显示 |
| 相机预览 | `CAMERA_WRITE \| COMPOSER_OVERLAY` | 相机写入 + HWC 显示 |
| 截图 | `CPU_READ_OFTEN \| GPU_TEXTURE` | CPU 读取 GPU 纹理 |
| DRM 视频 | `VIDEO_DECODER \| PROTECTED` | 受保护视频解码 |

### Gralloc HAL 的 Usage 处理逻辑
厂商 Gralloc HAL 实现中的决策流程：
1. **检查 PROTECTED**：如果设置，分配 Secure Memory，CPU 不可访问
2. **检查 SENSOR_DIRECT_DATA**：如果设置，使用 CMA 分配连续物理内存
3. **检查 GPU_RENDER_TARGET**：如果设置，确保物理连续（GPU tile 需要）
4. **检查 CPU_READ/WRITE**：如果设置，启用 CPU cache
5. **检查 VIDEO_DECODER/ENCODER**：如果设置，使用厂商定制的 ION heap
6. **组合冲突处理**：如 PROTECTED + CPU_READ 冲突，优先 PROTECTED

### Cache 策略
Usage 标志位影响 CPU cache 行为：
- **Cached**（CPU_READ_OFTEN/WRITE_OFTEN）：CPU 读写通过 cache，性能好但需要 cache flush
- **Uncached**（默认 GPU/Display）：CPU 直接访问物理内存，无 cache 开销
- **Write-combine**（GPU_RENDER_TARGET）：CPU 写入合并，适合 GPU 渲染目标

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 3.0 | GRALLOC_USAGE_* 初始定义 |
| Android 8.0 | 引入 AHARDWAREBUFFER_USAGE_* NDK 接口 |
| Android 10 | 增加 GRALLOC_USAGE_FRONTEND |
| Android 11 | 增加 AHARDWAREBUFFER_USAGE_GPU_FRAMEBUFFER |
| Android 14 | Gralloc 5.0 扩展 usage flags |

## 陷阱与注意事项
- **PROTECTED + CPU 冲突**：设置 PROTECTED 的缓冲区 CPU 无法 lock（返回 NULL）。如果同时设置了 CPU_READ/WRITE，Gralloc 应忽略 CPU 标志。
- **缺少 COMPOSER_OVERLAY**：如果 App 渲染缓冲区未设置 COMPOSER_OVERLAY，HWC 可能无法使用 overlay 合成，回退到 GPU 合成，增加功耗。
- **CPU cache 一致性**：当 GPU 和 CPU 共享缓冲区时，必须在 lock/unlock 时处理 cache flush。遗漏 flush 会导致画面撕裂或数据错误。
- **Usage 不能随意组合**：某些组合语义矛盾（如 SENSOR_DIRECT_DATA + PROTECTED），Gralloc HAL 需要定义优先级。

## 使用模式

### NDK 分配缓冲区
```c
// C (NDK)
AHardwareBuffer_Desc desc = {
    .width = 1080,
    .height = 1920,
    .format = AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM,
    .layers = 1,
    .usage = AHARDWAREBUFFER_USAGE_GPU_COLOR_OUTPUT
           | AHARDWAREBUFFER_USAGE_COMPOSER_OVERLAY,
};

AHardwareBuffer* buffer = NULL;
int ret = AHardwareBuffer_allocate(&desc, &buffer);
if (ret == 0) {
    // 使用 buffer...
    AHardwareBuffer_release(buffer);
}
```

### 调试 Usage 问题
```bash
# 查看 GraphicBuffer 的 usage 信息
adb shell dumpsys SurfaceFlinger | grep -A 5 "GraphicBuffer"

# 查看 DMA-BUF 的 cache 状态
adb shell cat /sys/kernel/debug/dma_buf/bufinfo
```

## Sources

- [P0] `frameworks/native/include/android/hardware_buffer.h` — AHARDWAREBUFFER_USAGE 定义
- [P0] `frameworks/native/include/system/graphics.h` — GRALLOC_USAGE 定义（旧版兼容）
- [P0] `hardware/interfaces/graphics/allocator/aidl/android/hardware/graphics/allocator/BufferDescriptor.aidl` — BufferDescriptor 定义
- [P0] `frameworks/native/libs/ui/GraphicBuffer.cpp` — GraphicBuffer usage 处理逻辑
- [P1] https://source.android.com/docs/core/graphics/ahardwarebuffer — Android AHardwareBuffer 文档：usage 标志位说明
- [P2] https://yjy239.github.io/2020/01/31/android-chong-xue-xi-lie-graphicbuffer-de-dan-sheng — GraphicBuffer 的诞生：usage 与内存类型映射
