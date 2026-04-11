# DRM Framebuffer 管理

> **条目ID:** KB-LINUX-DRM-atomic-010
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档 + Ben Widawsk Modifiers 博客）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13, bwidawsk.net
> **关联:** [plane-state](./plane-state.md) | [gem-objects](./gem-objects.md) | [dma-buf-overview](./dma-buf-overview.md)

---

## 核心摘要

DRM Framebuffer 是 DRM 子系统中描述像素缓冲区的核心对象——它封装了像素格式、分辨率、修饰符（Modifier）、底层 GEM Buffer 引用等元信息，是 Plane 绑定显示内容的载体。`struct drm_framebuffer` 本身不持有像素数据，而是引用一个或多个 GEM Buffer Object（通过 `obj[0..3]` 数组）。Framebuffer Modifier 机制是解决 GPU 渲染引擎与显示引擎之间缓冲区布局协商的关键方案，支持 Tiling、压缩（AFBC/CCS/UBWC）等硬件优化布局的端到端传递。

---

## 1. Framebuffer 核心结构

### `struct drm_framebuffer` 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct drm_device *` | 所属 DRM 设备 |
| `base` | `struct drm_mode_object` | 基类对象 |
| `format` | `const struct drm_format_info *` | 像素格式信息 |
| `width` | `unsigned int` | 宽度（像素） |
| `height` | `unsigned int` | 高度（像素） |
| `pitches` | `unsigned int[4]` | 每个平面行字节数（stride） |
| `offsets` | `unsigned int[4]` | 每个平面在 buffer 中的偏移 |
| `modifier` | `uint64_t` | 布局修饰符（tiling/compression） |
| `obj` | `struct drm_gem_object *[4]` | GEM Buffer 引用（最多 4 平面） |
| `filp_head` | `struct list_head` | per-fd 链表 |
| `funcs` | `const struct drm_framebuffer_funcs *` | 回调函数表 |

### 像素格式信息 `struct drm_format_info`

| 字段 | 类型 | 说明 |
|------|------|------|
| `format` | `u32` | DRM 四字符码（DRM_FORMAT_*） |
| `depth` | `u8` | 色深（位/像素） |
| `num_planes` | `u8` | 平面数量（1-4） |
| `cpp` | `u8[4]` | 每平面每像素字节数 |
| `hsub` | `u8` | 水平子采样因子 |
| `vsub` | `u8` | 垂直子采样因子 |
| `has_alpha` | `bool` | 是否包含 Alpha 通道 |
| `is_yuv` | `bool` | 是否为 YUV 格式 |

---

## 2. Framebuffer Modifier 机制

### 为什么需要 Modifier

```
传统方式（无 Modifier）：
  GPU 渲染 → 线性布局 → Compositor 读取 → 线性布局 → 显示引擎
  问题：GPU 通常使用 Tiling/Compression 布局，强制线性化导致：
  1. 渲染后需解压/重排 → 额外带宽和延迟
  2. 显示引擎无法利用硬件压缩 → 带宽浪费

Modifier 方式：
  GPU 渲染 → Tiling+Compression 布局 → Compositor 传递 Modifier → 显示引擎直接使用
  优势：端到端保持硬件优化布局，零拷贝
```

### 带宽分析（Ben Widawsk 计算）

```
Skylake 中端 GPU（24 EU）：
  GPU 渲染带宽：~24 GB/s
  4K@60Hz 显示扫描：3840 × 2160 × 4Bpp × 60Hz = 1.85 GB/s
  合计：~25.85 GB/s（超过单通道 DDR4 带宽）

  加入 Compositing 后：
  渲染 → 纹理采样 → 合成 → 显示扫描 → 每步都读/写一次
  总带宽可能翻 3-5 倍 → 严重超限

  解决方案：Modifier + 端到端压缩
  AFBC/CCS 可将带宽降低 50-75%
```

### 常见 Modifier 类型

| Modifier | 厂商 | 说明 |
|----------|------|------|
| `DRM_FORMAT_MOD_LINEAR` | 通用 | 线性布局（默认） |
| `I915_FORMAT_MOD_X_TILED` | Intel | X-Tiling（Sandy Bridge+） |
| `I915_FORMAT_MOD_Y_TILED` | Intel | Y-Tiling（Broadwell+） |
| `I915_FORMAT_MOD_Yf_TILED` | Intel | Yf-Tiling（Skylake+） |
| `I915_FORMAT_MOD_4_TILED` | Intel | 4-Tiling（XeHP+） |
| `DRM_FORMAT_MOD_ARM_AFBC()` | ARM | AFBC 帧缓冲压缩 |
| `DRM_FORMAT_MOD_QCOM_COMPRESSED` | Qualcomm | UBWC 压缩 |
| `DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK()` | NVIDIA | Block-linear 16B×2 |
| `DRM_FORMAT_MOD_AMDX925_TILED` | AMD | Tiled 模式 |
| `DRM_FORMAT_MOD_VIVANTE_TILED` | Vivante | Vivante Tiling |
| `DRM_FORMAT_MOD_BROADCOM_SAND128` | Broadcom | SAND128 列布局 |

### Modifier 协商流程

```
1. 驱动声明支持的 Modifier 列表
   drm_plane_add_modifier(plane, DRM_FORMAT_MOD_ARM_AFBC(...));

2. 用户空间查询支持的 Modifier
   DRM_IOCTL_MODE_GETPLANERESOURCES → modifier array

3. 用户空间创建带 Modifier 的 FB
   DRM_IOCTL_MODE_ADDFB2 → 传入 modifier

4. DRM Core 校验
   drm_mode_addfb2() → 调用 fb->funcs->create()
   驱动校验 modifier 是否与 format/size 兼容

5. 显示引擎使用 Modifier 信息
   Plane commit 时，驱动根据 modifier 配置 DMA 读取参数
```

---

## 3. Framebuffer 回调函数

### `struct drm_framebuffer_funcs`

| 回调 | 说明 |
|------|------|
| `create()` | 创建驱动特定的 FB 对象（替代默认实现） |
| `destroy()` | 销毁 FB，释放资源 |
| `create_handle()` | 为 FB 创建 GEM handle（DMA-BUF 导出） |
| `dirty()` | 标记脏区域（仅 legacy） |
| `prepare_fb()` | 准备 FB 用于扫描输出（设置 fence、pin buffer） |
| `cleanup_fb()` | 扫描输出完成后清理（unpin、释放 fence） |

### prepare_fb / cleanup_fb 生命周期

```
atomic_check()          ← 校验 FB 格式/Modifier 兼容性
    │
atomic_commit()
    ├─ prepare_fb()     ← Pin buffer、设置 fence、cache invalidate
    │   ├─ dma_resv_reserve_fences()
    │   ├─ dma_fence_get(plane_state->fence)
    │   └─ driver_hw_pin(fb)
    │
    ├─ 硬件编程          ← 配置 DMA 读取参数（含 modifier 信息）
    │
    ├─ wait_for_vblank() ← 等待翻转完成
    │
    └─ cleanup_fb()     ← Unpin buffer、释放引用
        ├─ driver_hw_unpin(fb)
        └─ dma_fence_put()
```

---

## 4. GEM FB Helper

DRM 提供了 `drm_gem_fb_helper` 简化 Framebuffer 创建：

### `drm_gem_fb_create_with_dirty()`
```c
// 最简创建方式（无 modifier）
struct drm_framebuffer *drm_gem_fb_create_with_dirty(
    struct drm_device *dev, struct drm_file *file,
    const struct drm_mode_fb_cmd2 *mode_cmd);
```

### `drm_gem_fb_create_with_modifier()`
```c
// 带 modifier 的创建方式
struct drm_framebuffer *drm_gem_fb_create_with_modifier(
    struct drm_device *dev, struct drm_file *file,
    const struct drm_mode_fb_cmd2 *mode_cmd,
    const uint64_t *modifiers, unsigned int modifier_count);
```

### `drm_gem_fb_prepare_fb()`
```c
// 标准 prepare_fb 实现（处理 fence + pin）
int drm_gem_fb_prepare_fb(struct drm_plane *plane,
                          struct drm_plane_state *new_state);
```

### `drm_gem_fb_cleanup_fb()`
```c
// 标准 cleanup_fb 实现
void drm_gem_fb_cleanup_fb(struct drm_plane *plane,
                           struct drm_plane_state *old_state);
```

---

## 5. 多平面 Framebuffer

YUV 格式（NV12、YUV420、YUV422）需要多个 GEM Buffer：

```
NV12 布局（2 平面）：
  Plane 0 (Y):  width × height       → obj[0]
  Plane 1 (UV): width/2 × height/2   → obj[1]（或同一 obj 的偏移）

YUV420 布局（3 平面）：
  Plane 0 (Y):  width × height       → obj[0]
  Plane 1 (U):  width/2 × height/2   → obj[1]
  Plane 2 (V):  width/2 × height/2   → obj[2]
```

### 子采样因子

| 格式 | hsub | vsub | 说明 |
|------|------|------|------|
| RGB | 1 | 1 | 无子采样 |
| NV12 | 2 | 2 | 4:2:0 |
| YUYV | 2 | 1 | 4:2:2 |
| YVYU | 2 | 1 | 4:2:2 |
| NV16 | 2 | 1 | 4:2:2 |

---

## 6. Framebuffer 在 Atomic Commit 中的角色

```
用户空间                    DRM Core                    驱动
    │                          │                          │
    ├─ DRM_IOCTL_MODE_ADDFB2 ─→├─ 校验 format/modifier ─→│
    │                          │←─ 创建 FB ──────────────┤
    │←─ 返回 FB ID ───────────┤                          │
    │                          │                          │
    ├─ ATOMIC_COMMIT ─────────→├─ 设置 plane_state->fb ─→│
    │   plane: fb=FB_ID        │                          │
    │                          ├─ atomic_check() ────────→│
    │                          │  校验 fb 格式兼容性       │
    │                          │                          │
    │                          ├─ prepare_fb() ──────────→│
    │                          │  pin + fence             │
    │                          │                          │
    │                          ├─ 硬件编程 ──────────────→│
    │                          │  根据 modifier 配置 DMA  │
    │                          │                          │
    │                          ├─ cleanup_fb() ──────────→│
    │                          │  unpin                   │
```

---

## 7. DMA-BUF 导出

Framebuffer 可以通过 DMA-BUF 机制在不同设备间共享：

```c
// 用户空间通过 ioctl 导出
int fd = drmIoctl(drm_fd, DRM_IOCTL_MODE_GETFB2, &req);
// req.fb_id → req.handles[i] → DMA-BUF fd

// 或通过 GEM handle 导出
int prime_fd = drmPrimeHandleToFD(drm_fd, gem_handle, 0, &prime_fd);
```

**跨设备共享场景：**
- GPU 渲染 → V4L2 视频编码（零拷贝）
- GPU 渲染 → ISP 图像处理
- GPU A 渲染 → GPU B 显示（混合 GPU 场景）

---

## 潜在影响

- **内存带宽：** Modifier + 端到端压缩是解决高分辨率（4K/8K）显示带宽瓶颈的关键技术。没有 Modifier 支持，GPU 和显示引擎之间需要额外的解压/重排操作。
- **跨设备共享：** DMA-BUF 导出是 Linux 多媒体管线（PipeWire、GStreamer）零拷贝的基础。
- **标准化挑战：** Ben Widawsk 指出，即使 Modifier 引入多年，全生态支持仍不完善。硬件设计者应优先考虑不需要软件参与的优化方案。
- **演进方向：** v6.x 中持续新增 Modifier 类型（Intel 4-Tiling、ARM AFBC 变体），`drm_format_info` 持续扩展。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [Framebuffer Modifiers Part 1 — bwidawsk.net](https://bwidawsk.net/blog/2021/2/modifiers)
- [drm_framebuffer.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_framebuffer.h)
- [drm_fourcc.h — libdrm](https://github.com/grate-driver/libdrm/blob/master/include/drm/drm_fourcc.h)
