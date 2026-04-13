# Gralloc 内存分配器

> Android 图形缓冲区的底层内存分配器，通过 HAL 接口管理 GPU/Display/Video Decoder 的共享内存。

## Meta
- **ID:** KB-ANDROID-GFX-gralloc-001
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 8.0+（Gralloc 4.0 IMapper/IAllocator），Android 12+（Gralloc 4.0 AIDL），Android 14+（Gralloc 5.0）
- **关键词:** Gralloc, IMapper, IAllocator, GraphicBuffer, buffer handle, buffer descriptor, usage flags, stride, DMA-BUF, ion, cma, contiguous, protected, AHARDWAREBUFFER
- **前置知识:** [→ KB-ANDROID-GFX-surfaceflinger-001]
- **关联条目:** [→ KB-ANDROID-GFX-gralloc-002] [→ KB-ANDROID-GFX-surfaceflinger-002]

## 概述

Gralloc（Graphics Allocator）是 Android 图形系统的底层内存分配器 HAL，负责分配和管理图形缓冲区的物理内存。Gralloc 的核心价值是**跨设备共享**：同一块物理内存可以被 GPU（渲染）、Display（合成）、Video Decoder（解码）同时访问，通过 DMA-BUF fd 实现零拷贝共享。Gralloc 4.0 引入 IMapper/IAllocator 双接口架构：IAllocator 负责分配/释放，IMapper 负责锁定/解锁/导入/导出。Android 14 引入 Gralloc 5.0 支持 YCbCr P010 格式和更细粒度的 usage flags。

## 架构/调用链

### Gralloc HAL 架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Gralloc 内存分配架构                        │
│                                                              │
│  App / SurfaceFlinger / MediaCodec                           │
│  ┌──────────────┐                                            │
│  │ GraphicBuffer │ ← 用户空间缓冲区封装                        │
│  │  (handle +    │                                            │
│  │   width/      │                                            │
│  │   height/     │                                            │
│  │   format/     │                                            │
│  │   usage)      │                                            │
│  └──────┬───────┘                                            │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐     ┌──────────────┐                      │
│  │   IMapper    │     │  IAllocator  │                      │
│  │ (lock/unlock │     │ (allocate/   │                      │
│  │  import/     │     │  deallocate)  │                      │
│  │  export/     │     └──────┬───────┘                      │
│  │  flush)      │            │                               │
│  └──────┬───────┘            │                               │
│         │                    │                               │
│         ▼                    ▼                               │
│  ┌──────────────────────────────────┐                        │
│  │        Gralloc HAL 实现           │                        │
│  │  (厂商定制: Qualcomm/MTK/Exynos)  │                        │
│  │                                  │                        │
│  │  ┌─────────┐  ┌──────────────┐  │                        │
│  │  │   ION   │  │    CMA       │  │                        │
│  │  │ (通用)  │  │ (连续物理内存) │  │                        │
│  │  └─────────┘  └──────────────┘  │                        │
│  └──────────────────────────────────┘                        │
│         │                    │                               │
│         ▼                    ▼                               │
│  ┌──────────────────────────────────┐                        │
│  │           Linux Kernel            │                        │
│  │  ION allocator / CMA / DMA-BUF   │                        │
│  └──────────────────────────────────┘                        │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────────────────────────┐                        │
│  │     硬件 (GPU/Display/VPU)       │                        │
│  │  通过 DMA-BUF fd 共享物理内存     │                        │
│  └──────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

### 内存分配流程

```
GraphicBuffer::create(width, height, format, usage)
    │
    ▼
IAllocator.allocate(descriptor, count, stride, handle)
    │
    ├── descriptor 包含:
    │   ├── width, height
    │   ├── format (RGBA_8888, YCBCR_420_888, ...)
    │   ├── layerCount
    │   └── usage (RENDERSCRIPT | GPU_TEXTURE | COMPOSER_OVERLAY | ...)
    │
    ├── Gralloc HAL 实现:
    │   ├── 根据 usage 选择内存类型:
    │   │   ├── GPU_RENDER_TARGET → ION/Contiguous (GPU 需要连续内存)
    │   │   ├── COMPOSER_OVERLAY → ION/CMA (Display DMA 需要)
    │   │   ├── VIDEO_DECODER → ION (视频解码器)
    │   │   ├── PROTECTED → Secure Memory (DRM 内容)
    │   │   └── CPU_READ/WRITE → Cached Memory
    │   │
    │   ├── 分配物理内存 (ION/CMA)
    │   ├── 创建 DMA-BUF fd
    │   └── 返回 buffer handle (不透明类型)
    │
    └── 返回:
        ├── stride (行跨度，可能 > width，用于对齐)
        └── handle (buffer 句柄，跨进程通过 Binder 传递)
```

## 关键接口

### 函数
- `IAllocator.allocate(descriptor, count, outStride, outHandle)` — 分配图形缓冲区
  - 参数: descriptor=BufferDescriptor(尺寸/格式/usage), count=分配数量, outStride=返回行跨度, outHandle=返回 buffer handle
  - 返回: OK, BAD_VALUE, NO_RESOURCES
  - 源码: `hardware/interfaces/graphics/allocator/aidl/android/hardware/graphics/allocator/IAllocator.aidl`
  - 注意: handle 是不透明类型，跨进程通过 Binder 传递 fd

- `IMapper.lock(handle, usage, accessRegion, fence, outData)` — 锁定缓冲区获取 CPU 访问指针
  - 参数: handle=buffer handle, usage=访问用途(CPU_READ/WRITE), accessRegion=访问区域, fence=等待 fence, outData=返回内存指针
  - 返回: void* (CPU 可访问的内存地址)
  - 源码: `hardware/interfaces/graphics/mapper/aidl/android/hardware/graphics/mapper/IMapper.aidl`
  - 注意: lock 前必须等待 fence signal。lock 后 CPU 才能访问缓冲区内容

- `IMapper.unlock(handle, outFence)` — 解锁缓冲区
  - 参数: handle=buffer handle, outFence=返回 fence（signal 时表示硬件已完成访问）
  - 返回: OK
  - 源码: `hardware/interfaces/graphics/mapper/aidl/android/hardware/graphics/mapper/IMapper.aidl`
  - 注意: unlock 后 CPU 不应再访问缓冲区。fence signal 后 GPU/Display 才能安全访问

- `IMapper.importBuffer(handle)` — 导入已有缓冲区（跨进程共享）
  - 参数: handle=从其他进程接收的 buffer handle
  - 返回: OK, BAD_BUFFER
  - 源码: `hardware/interfaces/graphics/mapper/aidl/android/hardware/graphics/mapper/IMapper.aidl`
  - 注意: 跨进程传递 GraphicBuffer 时，接收方必须调用 importBuffer 注册到本地 Gralloc

### 数据结构
- `struct BufferDescriptor` — 缓冲区描述符
  - 关键字段:
    - `width` — 宽度（像素）
    - `height` — 高度（像素）
    - `format` — 像素格式（PixelFormat 枚举）
    - `layerCount` — 图层数（1=普通，2+=立体/多平面）
    - `usage` — 使用标志位（bitmask）
  - 源码: `hardware/interfaces/graphics/allocator/aidl/android/hardware/graphics/allocator/BufferDescriptor.aidl`

## 实现要点

### Usage Flags 与内存类型映射
Gralloc HAL 根据 usage flags 选择最优内存类型：
| Usage Flag | 含义 | 典型内存类型 |
|------------|------|-------------|
| `GPU_RENDER_TARGET` | GPU 渲染目标 | Contiguous (连续物理内存) |
| `GPU_TEXTURE` | GPU 纹理采样 | ION (可能非连续) |
| `COMPOSER_OVERLAY` | HWC overlay 显示 | Contiguous (DMA 需要) |
| `VIDEO_DECODER` | 视频解码输出 | ION (厂商定制) |
| `PROTECTED` | DRM 保护内容 | Secure Memory (TEE) |
| `CPU_READ/WRITE` | CPU 直接访问 | Cached (带 cache) |
| `SENSOR_DIRECT_DATA` | 相机传感器数据 | CMA (低延迟) |

### DMA-BUF 共享机制
Gralloc 分配的缓冲区底层是 Linux DMA-BUF：
1. Gralloc HAL 通过 ION/CMA 分配物理内存，创建 DMA-BUF fd
2. fd 通过 Binder 传递给其他进程（SurfaceFlinger、Video Decoder）
3. 接收方通过 importBuffer() 注册 fd 到本地 Gralloc
4. GPU/Display/VPU 通过 fd mmap 或 DMA 直接访问物理内存
5. fence 机制保证跨设备访问的同步

### Stride（行跨度）
Gralloc 返回的 stride 可能大于 width：
- GPU 通常要求 64/128 字节对齐
- 例如：width=1080, format=RGBA_8888(4 bytes/pixel), stride 可能是 1088
- stride 影响 CPU 访问时的行指针计算：`row_ptr = base + y * stride * bytes_per_pixel`

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 3.0 | Gralloc 0.x，单接口 gralloc_alloc/gralloc_free |
| Android 8.0 | Gralloc 4.0，拆分为 IMapper/IAllocator 双接口 |
| Android 10 | Gralloc 4.0 HIDL 稳定版 |
| Android 12 | Gralloc 4.0 迁移到 AIDL |
| Android 14 | Gralloc 5.0，支持 YCbCr P010、更细粒度 usage |

## 陷阱与注意事项
- **stride ≠ width**：CPU 访问缓冲区时必须使用 stride 而非 width 计算行偏移。使用 width 会导致图像错位。
- **fence 同步**：lock 前必须等待 fence signal，否则可能读到 GPU 还在写入的数据。unlock 后返回的 fence 必须传递给下一个访问者。
- **跨进程传递**：GraphicBuffer 通过 Binder 传递时，接收方必须调用 importBuffer()。直接使用 handle 会导致访问失败。
- **PROTECTED 内存**：受 DRM 保护的缓冲区 CPU 无法访问（lock 返回 NULL）。只能通过 GPU/Display/VPU 硬件访问。

## 使用模式

### 分配图形缓冲区
```cpp
// C++ (Native)
GraphicBufferAllocator& alloc = GraphicBufferAllocator::get();
buffer_handle_t handle;
uint32_t stride;
int ret = alloc.allocate(width, height, PIXEL_FORMAT_RGBA_8888,
    GRALLOC_USAGE_RENDERSCRIPT | GRALLOC_USAGE_HW_COMPOSER,
    &handle, &stride);
if (ret == OK) {
    // 使用 handle...
    // 释放
    alloc.free(handle);
}
```

### 调试 Gralloc
```bash
# 查看 Gralloc HAL 信息
adb shell dumpsys SurfaceFlinger | grep -i gralloc

# 查看 DMA-BUF 信息
adb shell cat /proc/dma_buf/bufinfo

# 查看 ION heap 信息
adb shell cat /sys/kernel/debug/ion/heaps/ion_system_heap
```

## Sources

- [P0] `hardware/interfaces/graphics/allocator/aidl/android/hardware/graphics/allocator/IAllocator.aidl` — IAllocator 接口定义：allocate/deallocate
- [P0] `hardware/interfaces/graphics/mapper/aidl/android/hardware/graphics/mapper/IMapper.aidl` — IMapper 接口定义：lock/unlock/import/export/flush
- [P0] `frameworks/native/libs/ui/GraphicBufferAllocator.cpp` — GraphicBufferAllocator 实现：allocate/free 封装
- [P0] `frameworks/native/libs/ui/GraphicBuffer.cpp` — GraphicBuffer 实现：lock/unlock/importBuffer
- [P0] `frameworks/native/libs/ui/include/ui/GraphicBuffer.h` — GraphicBuffer 类定义
- [P1] https://source.android.com/docs/core/graphics/ahardwarebuffer — Android AHardwareBuffer 文档
- [P2] https://yjy239.github.io/2020/01/31/android-chong-xue-xi-lie-graphicbuffer-de-dan-sheng — GraphicBuffer 的诞生：Gralloc 内存分配深度分析
