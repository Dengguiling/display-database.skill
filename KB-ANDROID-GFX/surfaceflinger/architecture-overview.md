# SurfaceFlinger Architecture Overview

> Android 系统核心图形合成服务，负责接收多源缓冲区、合成图层并提交到显示设备。

## Meta
- **ID:** KB-ANDROID-GFX-surfaceflinger-001
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 7.0+（CompositionEngine 引入于 Android 10），Android 14+ 推荐参考
- **关键词:** SurfaceFlinger, composition, Layer, BufferQueue, HWC, RenderEngine, CompositionEngine, VSync, EventThread
- **前置知识:** [→ KB-LINUX-DRM-atomic-001]
- **关联条目:** [→ KB-ANDROID-GFX-surfaceflinger-002] [→ KB-ANDROID-GFX-surfaceflinger-003] [→ KB-ANDROID-GFX-surfaceflinger-004]

## 概述

SurfaceFlinger 是 Android 系统中运行在独立进程（`system_server` 启动）的核心图形合成服务。它作为所有应用窗口和系统 UI 的最终汇聚点，从各应用的 BufferQueue 中获取已渲染的图形缓冲区，通过 GPU 合成（GLES）或硬件合成（HWC）将多个 Layer 合成为最终帧，提交到显示设备。SurfaceFlinger 同时管理 VSync 信号的分发，驱动整个 Android UI 渲染管线。

## 架构/调用链

```
┌─────────────────────────────────────────────────────────────────┐
│                    Android Graphics Stack                        │
│                                                                  │
│  App Process                SurfaceFlinger Process               │
│  ┌──────────┐              ┌──────────────────────┐             │
│  │ ViewRoot │              │   SurfaceFlinger     │             │
│  │  Impl    │              │   (main thread)      │             │
│  │    │     │              │                      │             │
│  │    ▼     │   Binder     │  ┌────────────────┐  │             │
│  │ Surface  │◄────────────►│  │ Layer 管理     │  │             │
│  │  Control │   IPC        │  │ (mCurrentState)│  │             │
│  └────┬─────┘              │  └───────┬────────┘  │             │
│       │                    │          │           │             │
│       ▼                    │          ▼           │             │
│  ┌──────────┐              │  ┌────────────────┐  │             │
│  │BufferQueue│              │  │CompositionEngine│  │             │
│  │ Producer │              │  │  (Android 10+) │  │             │
│  └────┬─────┘              │  └───────┬────────┘  │             │
│       │                    │          │           │             │
│       │ Buffer (handle)    │          ▼           │             │
│       │───────────────────►│  ┌────────────────┐  │             │
│       │                    │  │ RenderEngine   │  │             │
│       │                    │  │ (GLES 合成)    │  │             │
│       │                    │  └───────┬────────┘  │             │
│       │                    │          │           │             │
│       │                    │          ▼           │             │
│       │                    │  ┌────────────────┐  │             │
│       │                    │  │   HWC / HWC2   │  │             │
│       │                    │  │ (硬件合成 HAL) │  │             │
│       │                    │  └───────┬────────┘  │             │
│       │                    │          │           │             │
│       │                    │          ▼           │             │
│       │                    │  ┌────────────────┐  │             │
│       │                    │  │  Display HAL   │  │             │
│       │                    │  │  (显示输出)     │  │             │
│       │                    │  └────────────────┘  │             │
│       │                    │                      │             │
│       │                    │  ┌────────────────┐  │             │
│       │                    │  │  EventThread    │  │             │
│       │                    │  │ (VSync 分发)   │  │             │
│       │                    │  └────────────────┘  │             │
│       │                    └──────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### 核心线程模型

```
SurfaceFlinger 进程
    │
    ├─ main thread (SurfaceFlinger 主线程)
    │   ├─ 处理 Binder 调用 (createLayer, setTransactionState, etc.)
    │   ├─ VSync 到来时触发合成 (onMessageReceived)
    │   └─ 合成决策：哪些 Layer 走 HWC，哪些走 GLES
    │
    ├─ EventThread (vsync-app)
    │   └─ 向应用进程分发 VSync 信号 (通过 BitTube/DisplayEventReceiver)
    │
    ├─ EventThread (vsync-sf)
    │   └─ 向 SurfaceFlinger 自身分发 VSync 信号
    │
    └─ RenderEngine (GLES 线程)
        └─ 执行 GPU 合成操作 (drawLayers)
```

### 合成决策流程

```
VSync 到来
    │
    ▼
SurfaceFlinger::onMessageReceived(INVALIDATE)
    │
    ├─ 遍历所有 DisplayDevice
    │   ├─ 收集可见 Layer 列表
    │   ├─ 调用 HWC::validate()
    │   │   └─ HWC 决定每个 Layer 的合成类型:
    │   │       ├─ HWC_OVERLAY  → 硬件直接合成（最优）
    │   │       ├─ HWC_CURSOR   → 硬件光标层
    │   │       └─ HWC_FRAMEBUFFER → 需要 GLES 合成
    │   │
    │   ├─ 对 HWC_FRAMEBUFFER 类型的 Layer 执行 GLES 合成
    │   │   └─ RenderEngine::drawLayers() → 输出到目标缓冲区
    │   │
    │   └─ 调用 HWC::present() → 提交到显示设备
    │
    ▼
SurfaceFlinger::onMessageReceived(REFRESH)
    └─ 释放旧缓冲区，更新状态
```

## 关键接口

### 核心类

- `SurfaceFlinger` — 图形合成服务主类
  - 源码: `frameworks/native/services/surfaceflinger/SurfaceFlinger.h`
  - 继承: `ISurfaceComposer`, `ComposerCallback`, `BnSurfaceComposer`
  - 关键职责: Layer 生命周期管理、合成调度、VSync 分发、Display 管理

- `CompositionEngine` — 合成引擎抽象层（Android 10+ 引入）
  - 源码: `frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.h`
  - 说明: 将合成逻辑从 SurfaceFlinger 主类中解耦，提供可测试的合成决策接口
  - 关键方法: `present()`, `getOutputLayersForDisplay()`

- `RenderEngine` — GPU 渲染引擎抽象
  - 源码: `frameworks/native/services/surfaceflinger/RenderEngine/RenderEngine.h`
  - 说明: 封装 GLES/EGL 操作，提供 `drawLayers()` 接口执行 GPU 合成
  - 实现: `GLESRenderEngine` (GLES 2.0/3.0)

- `Layer` — 图层基类
  - 源码: `frameworks/native/services/surfaceflinger/Layer.h`
  - 说明: 每个应用窗口对应一个 Layer，持有 BufferQueue 的 Consumer 端
  - 子类: `BufferStateLayer`, `BufferLayer`, `ColorLayer`, `ContainerLayer`

- `DisplayDevice` — 显示设备抽象
  - 源码: `frameworks/native/services/surfaceflinger/DisplayDevice.h`
  - 说明: 表示一个物理或虚拟显示设备，关联 HWC Display 和 Layer Stack

- `EventThread` — VSync 事件分发线程
  - 源码: `frameworks/native/services/surfaceflinger/EventThread.h`
  - 说明: 从 DispSync/VsyncModulator 获取 VSync 信号，通过 BitTube 分发给注册的连接
  - 关键方法: `requestNextVsync()`, `onVSyncEvent()`

### Binder 接口 (ISurfaceComposer)

- `createConnection()` — 创建与 SurfaceFlinger 的连接，返回 ISurfaceComposerClient
- `createDisplay(String name, boolean secure)` — 创建虚拟显示
- `setTransactionState(...)` — 提交 Layer 属性变更事务（位置、大小、Z序等）
- `captureLayers(...)` — 截取指定 Layer 的内容
- `getSupportedFrameTimestamps()` — 获取支持的帧时间戳类型

## 实现要点

### 1. 双状态模型 (mCurrentState / mDrawingState)

SurfaceFlinger 维护两套 Layer 状态：
- `mCurrentState`: 当前最新状态，接收来自应用的 Transaction 更新
- `mDrawingState`: 正在用于绘制/合成的状态

每次 VSync 合成开始时，将 `mCurrentState` swap 到 `mDrawingState`，确保合成过程中状态一致性。

### 2. 合成类型决策

SurfaceFlinger 通过 HWC2 的 `validate()` 接口让硬件决定每个 Layer 的合成方式：
- **Device Composition (HWC_OVERLAY)**: 硬件直接合成，零 GPU 开销，最优功耗
- **GPU Composition (HWC_FRAMEBUFFER)**: SurfaceFlinger 通过 RenderEngine (GLES) 合成后，将结果作为单个缓冲区交给 HWC
- **混合模式**: 部分层走硬件，部分走 GPU，最终由 HWC 合并输出

### 3. BufferQueue 生产者-消费者模型

每个 Layer 持有 BufferQueue 的 Consumer 端。应用进程通过 Surface (Producer 端) 将渲染完成的图形缓冲区入队。SurfaceFlinger 在合成时从 BufferQueue dequeue 缓冲区。缓冲区通过 Gralloc HAL 分配，使用 DMA-BUF handle 传递，零拷贝。

### 4. VSync 信号链

```
Display Hardware (HWC)
    │ vsync event
    ▼
SurfaceFlinger::onVsyncReceived()
    │
    ▼
DispSync / VsyncModulator (模型化 vsync 周期)
    │
    ├─► EventThread (vsync-sf) → SurfaceFlinger 自身合成触发
    │
    └─► EventThread (vsync-app) → 通过 BitTube 分发给应用
                                    → Choreographer 接收
                                    → 触发应用 UI 渲染
```

### 5. Transaction 机制

应用通过 `SurfaceControl.Transaction` 提交 Layer 属性变更（位置、大小、透明度、Z序等）。Transaction 支持批量操作和原子性应用，通过 `setTransactionState()` Binder 调用传递给 SurfaceFlinger。

## 版本差异

| 版本 | 变更 |
|------|------|
| Android 4.4 (KitKat) | 引入 HWC2 概念，支持 4 个 overlay plane |
| Android 7.0 (Nougat) | SurfaceFlinger 重构，引入 ColorLayer、BufferStateLayer 等新 Layer 类型 |
| Android 8.0 (Oreo) | HWC2 正式取代 HWC1，HAL 接口从 `hwcomposer.h` 迁移到 `composer@2.0` HIDL |
| Android 10 (Q) | 引入 CompositionEngine，将合成逻辑从 SurfaceFlinger 主类解耦；引入 DispSync 替换为 VsyncModulator |
| Android 12 (S) | 引入 SurfaceControl.Transaction 的 merge API；优化了 Transaction 的 apply 机制 |
| Android 14 (U) | Display HAL 从 HIDL 迁移到 AIDL (`android.hardware.graphics.composer3`); Frame Rate API 增强 |

## 陷阱与注意事项

1. **SurfaceFlinger 运行在独立进程中**：与 `system_server` 不同，SurfaceFlinger 有自己的进程空间。Binder 调用涉及跨进程序列化，频繁的 Transaction 提交会增加 IPC 开销
2. **GPU 合成是 fallback 路径**：当 HWC 无法处理所有 Layer（如 Layer 数量超过硬件 overlay 数量、使用不支持的颜色空间等），回退到 GLES 合成，功耗显著增加
3. **VSync 偏移 (VsyncOffset)**: vsync-app 和 vsync-sf 有不同的偏移量，vsync-app 先于 vsync-sf 触发，为应用渲染预留时间
4. **虚拟显示无独立 VSync**: 虚拟显示的合成由内部显示的 VSync 触发，录制/投屏场景需注意帧率匹配
5. **BufferQueue 的缓冲区数量**: 默认为 3（triple buffering），过多缓冲区增加延迟，过少导致应用阻塞等待

## 使用模式

### 1. 应用创建 Surface 并渲染

```cpp
// App 侧 (Java)
SurfaceControl sc = new SurfaceControl.Builder(session)
    .setName("MyLayer")
    .setBufferSize(width, height)
    .build();

Surface surface = new Surface(sc);
Canvas canvas = surface.lockCanvas(null);
canvas.drawColor(Color.RED);
surface.unlockCanvasAndPost(canvas);
```

### 2. 调试 SurfaceFlinger 状态

```bash
# 查看所有 Layer 及其合成类型
adb shell dumpsys SurfaceFlinger

# 查看 HWC 合成详情（含每个 Layer 的合成方式）
adb shell dumpsys SurfaceFlinger --display-id 0

# 开启 SurfaceFlinger 调试日志
adb shell setprop debug.sf.dump true
```

### 3. 查看 VSync 状态

```bash
# 查看 VSync 周期和偏移
adb shell dumpsys SurfaceFlinger --vsync

# Systrace/Perfetto 中查看 SurfaceFlinger 合成耗时
# 关注: SurfaceFlinger::onMessageReceived(INVALIDATE) 的持续时间
```

## Sources

- [P0] `frameworks/native/services/surfaceflinger/SurfaceFlinger.h` — SurfaceFlinger 主类定义，Layer 管理、合成调度、VSync 分发
- [P0] `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` — SurfaceFlinger 核心实现，`init()`, `onMessageReceived()`, `onVsyncReceived()`
- [P0] `frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.h` — CompositionEngine 抽象层定义（Android 10+）
- [P0] `frameworks/native/services/surfaceflinger/RenderEngine/RenderEngine.h` — RenderEngine GPU 合成接口
- [P0] `frameworks/native/services/surfaceflinger/Layer.h` — Layer 基类定义
- [P0] `frameworks/native/services/surfaceflinger/EventThread.h` — EventThread VSync 分发机制
- [P0] `frameworks/native/services/surfaceflinger/DisplayDevice.h` — DisplayDevice 显示设备抽象
- [P1] https://source.android.com/docs/core/graphics/architecture — Android 官方图形架构文档
- [P1] https://cs.android.com/android/platform/superproject/+/master:frameworks/native/services/surfaceflinger/ — AOSP SurfaceFlinger 源码目录
- [P2] https://utzcoz.github.io/2020/05/02/Analyze-AOSP-vsync-model.html — AOSP VSync 模型分析（基于 Android 9.0）
