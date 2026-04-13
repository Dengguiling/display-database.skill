# HWC2 接口

> Android Hardware Composer HAL 2.x 接口，SurfaceFlinger 与显示硬件合成器之间的通信桥梁。

## Meta
- **ID:** KB-ANDROID-GFX-hwcomposer-001
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 7.0+（HWC2.0），Android 8.0+（HWC2.1），Android 12+（AIDL Composer 2.x），Android 14+（Composer 3.x）
- **关键词:** HWC2, Hardware Composer, Composer HAL, AIDL, overlay, composition types, client target, virtual display, hotplug, validateDisplay, presentDisplay, ColorMode, HDR
- **前置知识:** [→ KB-ANDROID-GFX-surfaceflinger-001]
- **关联条目:** [→ KB-ANDROID-GFX-hwcomposer-002] [→ KB-ANDROID-GFX-display-hal-001]

## 概述

HWC2（Hardware Composer 2）是 Android 的硬件合成器 HAL 接口，定义了 SurfaceFlinger 与显示硬件之间的通信协议。HWC2 的核心职责是**决定哪些 Layer 可以由硬件直接合成（Device Composition），哪些需要回退到 GPU 合成（Client Composition）**。HWC2 通过 overlay plane 直接将多个缓冲区送到屏幕，避免 GPU 合成的功耗开销。Android 12+ 将 HWC HAL 从 HIDL 迁移到 AIDL，Android 14+ 引入 Composer 3.x 支持 ComposerCommandBuffer 批处理。

## 架构/调用链

### SurfaceFlinger 与 HWC 交互流程

```
SurfaceFlinger                    HWC HAL (AIDL)
    │                                  │
    │  createLayer()                   │
    │ ─────────────────────────────→   │
    │  返回 Layer handle               │
    │ ←─────────────────────────────   │
    │                                  │
    │  validateDisplay()               │
    │ ─────────────────────────────→   │
    │  HWC 决定合成类型:                │
    │  Layer 0: Device (overlay)       │
    │  Layer 1: Device (overlay)       │
    │  Layer 2: Client (GPU)           │
    │  Layer 3: Client (FBTarget)      │
    │ ←─────────────────────────────   │
    │  返回 ChangedCompositionTypes    │
    │                                  │
    │  [GPU 合成 Client Layer → FBTarget]│
    │                                  │
    │  acceptDisplayChanges()          │
    │ ─────────────────────────────→   │
    │                                  │
    │  setLayerBuffer() × N            │
    │ ─────────────────────────────→   │
    │                                  │
    │  presentDisplay(fence)           │
    │ ─────────────────────────────→   │
    │  返回 presentFence               │
    │ ←─────────────────────────────   │
    │  presentFence signal = 显示完成   │
    │                                  │
    │  onVsync(timestamp)              │
    │ ←─────────────────────────────   │
    │  onHotplug(display, connected)   │
    │ ←─────────────────────────────   │
```

### 合成类型决策

```
SurfaceFlinger 收集所有可见 Layer
    │
    ▼
validateDisplay()
    │
    ├── HWC 检查每个 Layer:
    │   ├── 是否有可用 overlay plane？
    │   ├── Layer 格式是否支持？
    │   ├── Layer 尺寸是否超出屏幕？
    │   ├── 是否需要 Alpha 混合？
    │   └── 是否是 DRM 保护内容？
    │
    ├── Device Composition (HWC 直接合成)
    │   ├── 优点: 低功耗、高效率
    │   ├── 限制: overlay plane 数量有限（通常 3-6 个）
    │   └── 限制: 部分 HWC 不支持 per-pixel alpha
    │
    └── Client Composition (GPU 合成)
        ├── 优点: 无 Layer 数量限制、支持任意 alpha
        ├── 缺点: 高功耗、占用 GPU 资源
        └── GPU 合成结果写入 FBTarget → 作为特殊 Layer 交给 HWC
```

## 关键接口

### 函数
- `validateDisplay(display, outChangedTypes, outDisplayRequest)` — 让 HWC 决定每个 Layer 的合成类型
  - 参数: display=显示设备 ID, outChangedTypes=返回合成类型变更的 Layer 列表, outDisplayRequest=返回显示请求标志
  - 返回: OK, NO_RESOURCES（HWC 无法处理，需全部回退 GPU）
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: SurfaceFlinger 在每次合成前必须调用 validateDisplay。如果返回 NO_RESOURCES，所有 Layer 回退到 GPU 合成

- `presentDisplay(display, outPresentFence)` — 提交合成结果到显示设备
  - 参数: display=显示设备 ID, outPresentFence=返回 present fence（signal 时表示显示完成）
  - 返回: OK
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: presentFence 是关键同步原语，SurfaceFlinger 必须等待 fence signal 后才能释放缓冲区

- `setLayerBuffer(display, layer, buffer, acquireFence)` — 设置 Layer 的图形缓冲区
  - 参数: display=显示设备, layer=Layer handle, buffer=GraphicBuffer, acquireFence=缓冲区可用 fence
  - 返回: void
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: acquireFence signal 后 HWC 才能读取缓冲区内容

- `createVirtualDisplay(width, height, format, layer)` — 创建虚拟显示设备
  - 参数: width/height=分辨率, format=像素格式, layer=输出 Layer
  - 返回: display ID
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: 用于录屏、无线投屏、模拟器等场景

### 数据结构
- `struct Layer` — HWC Layer 句柄（不透明类型）
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/`

- `enum Composition` — 合成类型枚举
  - 关键值:
    - `INVALID` — 无效
    - `CLIENT` — GPU 合成（SurfaceFlinger 使用 RenderEngine）
    - `DEVICE` — HWC 直接合成（overlay）
    - `CURSOR` — 光标层（低延迟 overlay）
    - `SIDEBAND` — 侧带合成（视频解码器直接输出）
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/Composition.aidl`

## 实现要点

### HWC 合成决策算法
SurfaceFlinger 调用 validateDisplay 后，HWC HAL 实现的决策逻辑：
1. 遍历所有 Layer，检查是否有可用 overlay plane
2. 优先分配 Device Composition 给简单 Layer（无 alpha、无缩放、格式支持）
3. 复杂 Layer（需要 alpha 混合、缩放、旋转）回退到 Client Composition
4. Client Composition 的结果作为 FBTarget Layer，由 HWC 最终显示
5. 如果 overlay plane 不够，HWC 返回 NO_RESOURCES，SurfaceFlinger 全部回退 GPU

### ComposerCommandBuffer (Android 14+)
传统 HWC 每次调用都是 Binder IPC，开销大。ComposerCommandBuffer 将多个命令打包：
- SurfaceFlinger 将 setLayerBuffer/setLayerState 等命令写入共享内存
- 通过一次 Binder 调用 executeCommands() 提交整个命令序列
- 显著减少 Binder 调用次数，降低合成延迟

### Hotplug 处理
HWC 通过 ComposerCallback 回调通知 SurfaceFlinger 显示设备的热插拔事件：
- `onHotplugReceived(display, connected)` — 屏幕/外接显示器连接/断开
- SurfaceFlinger 收到后创建/销毁 DisplayDevice，更新 Layer 列表

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 4.0 | HWC 1.0 引入，HIDL 接口 |
| Android 5.0 | HWC 1.1，增加颜色模式支持 |
| Android 6.0 | HWC 1.3，增加虚拟显示 |
| Android 7.0 | HWC 2.0，完全重新设计接口，引入 validate/present 模型 |
| Android 8.0 | HWC 2.1，增加 ColorMode/HDR 支持 |
| Android 10 | HWC 2.3，增加 DisplayIdleTimer |
| Android 12 | HWC 迁移到 AIDL（Composer 2.x），减少 Binder 开销 |
| Android 14 | Composer 3.x，引入 ComposerCommandBuffer 批处理 |

## 陷阱与注意事项
- **validateDisplay 必须在 presentDisplay 之前调用**：这是 HWC2 的硬性要求。如果 Layer 状态发生变化（新增/删除/修改），必须重新 validate。
- **NO_RESOURCES 处理**：当 HWC 返回 NO_RESOURCES 时，SurfaceFlinger 必须将所有 Layer 回退到 GPU 合成。不能部分使用 HWC 合成。
- **FBTarget 必须是最后一个 Layer**：Client Composition 的结果（FBTarget）必须作为 HWC 的最后一个 Layer，z-order 最高。
- **DRM 内容**：受 DRM 保护的视频内容只能通过 Device Composition 显示（直接送到 overlay），GPU 无法访问。如果 overlay 不够，DRM 内容的 Layer 必须使用 SIDEBAND 合成。

## 使用模式

### 调试 HWC 合成类型
```bash
# 查看 SurfaceFlinger dump 中的 HWC 合成信息
adb shell dumpsys SurfaceFlinger

# 关注输出中的:
# HWC layers:
# Layer name | Z | Comp Type | Disp Frame | Source Crop
# Comp Type: Device = HWC 合成, Client = GPU 合成, Cursor = 光标层

# 查看 HWC HAL 版本
adb shell dumpsys SurfaceFlinger | grep "HWC version"
```

### SurfaceFlinger 合成决策日志
```bash
# 启用 SurfaceFlinger 合成调试
adb shell setprop debug.sf.dumpport 8080
# 或通过 Systrace/Perfetto 查看:
# SurfaceFlinger::rebuildLayerStacks()
# SurfaceFlinger::doComposition()
```

## Sources

- [P0] `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl` — Composer 3.x AIDL 接口定义：validateDisplay/presentDisplay/setLayerBuffer
- [P0] `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/Composition.aidl` — Composition 枚举定义：CLIENT/DEVICE/CURSOR/SIDEBAND
- [P0] `frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp` — SurfaceFlinger 侧 HWC 封装实现
- [P0] `frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.h` — Composer HAL 抽象层
- [P1] https://source.android.com/docs/core/graphics/hwc — Android 官方 HWC 文档
- [P2] https://androidperformance.com/en/2020/02/14/Android-Systrace-SurfaceFlinger — Android Systrace: HWComposer Section
- [P2] https://www.programmersought.com/article/86921043167 — Android P graphic display system: HWC2 技术详解
