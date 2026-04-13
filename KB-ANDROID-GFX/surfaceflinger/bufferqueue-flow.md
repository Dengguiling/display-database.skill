# BufferQueue 数据流

> Android 图形缓冲区的生产者-消费者模型核心机制，连接应用渲染与 SurfaceFlinger 合成的桥梁。

## Meta
- **ID:** KB-ANDROID-GFX-surfaceflinger-002
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 7.0+（BufferQueue 核心机制稳定），Android 12+ 引入 BufferHub/BLASTBufferQueue
- **关键词:** BufferQueue, producer, consumer, dequeueBuffer, queueBuffer, acquireBuffer, releaseBuffer, BufferSlot, GraphicBuffer, fence, triple buffering, BLASTBufferQueue
- **前置知识:** [→ KB-ANDROID-GFX-surfaceflinger-001]
- **关联条目:** [→ KB-ANDROID-GFX-surfaceflinger-003] [→ KB-ANDROID-GFX-hwcomposer-001]

## 概述

BufferQueue 是 Android 图形系统的核心数据结构，实现了生产者-消费者模型：生产者（App 的 RenderThread）通过 `dequeueBuffer` 获取空闲缓冲区并填充内容，通过 `queueBuffer` 提交已渲染帧；消费者（SurfaceFlinger）通过 `acquireBuffer` 获取待合成帧，通过 `releaseBuffer` 释放已使用帧。BufferQueue 由消费者创建和管理，通过 Binder IPC 跨进程通信。Android 12 引入 BLASTBufferQueue（BufferLayer 的优化版本），每个 Layer 独立拥有 BufferQueue，SurfaceFlinger 不再需要为每个 BufferQueue 维护独立的消费者。

## 架构/调用链

### BufferQueue 生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                    BufferQueue 数据流                          │
│                                                              │
│  Producer (App RenderThread)          Consumer (SurfaceFlinger)│
│  ┌──────────────┐                    ┌──────────────┐        │
│  │              │  dequeueBuffer()   │              │        │
│  │  1. 获取空缓冲 │ ────────────────→ │  BufferQueue  │        │
│  │     区        │                    │  (Slot 数组)  │        │
│  │              │  ← 返回 slot + fence│              │        │
│  │  2. GPU 渲染  │                    │              │        │
│  │     到缓冲区  │                    │              │        │
│  │              │  queueBuffer()     │              │        │
│  │  3. 提交帧    │ ────────────────→ │  acquireBuffer│        │
│  │              │                    │  ()           │        │
│  │              │                    │  4. 获取待合成 │        │
│  │              │                    │     帧        │        │
│  │              │                    │  5. 合成并显示 │        │
│  │              │                    │  releaseBuffer│        │
│  │              │                    │  ()           │        │
│  │              │  ← 缓冲区回收      │  6. 释放缓冲区 │        │
│  └──────────────┘                    └──────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

### BufferSlot 状态机

```
                   dequeueBuffer()
    FREE ─────────────────────────────→ DEQUEUED
      ↑                                  │
      │         releaseBuffer()          │ queueBuffer()
      │                                  ↓
    FREE ←─────────────────────────── QUEUED
      ↑                                  │
      │         releaseBuffer()          │ acquireBuffer()
      │                                  ↓
    FREE ←─────────────────────────── ACQUIRED
```

### 三缓冲机制

```
时间线 (VSync 周期):
    VSync-1          VSync-2          VSync-3          VSync-4
    ┌────────┐      ┌────────┐      ┌────────┐      ┌────────┐
    │ App    │      │ App    │      │ App    │      │ App    │
    │ 渲染 A │      │ 渲染 B │      │ 渲染 C │      │ 渲染 D │
    └───┬────┘      └───┬────┘      └───┬────┘      └───┬────┘
        │               │               │               │
    queueBuffer(A)  queueBuffer(B)  queueBuffer(C)  queueBuffer(D)
        │               │               │               │
    ┌───▼───────────────▼───────────────▼───────────────▼────┐
    │                    BufferQueue                            │
    │  Slot 0: A(QUEUED) → A(ACQUIRED) → A(FREE) → D(QUEUED) │
    │  Slot 1: (FREE)     → B(QUEUED) → B(ACQUIRED) → B(FREE)│
    │  Slot 2: (FREE)     → (FREE)     → C(QUEUED) → C(ACQ.) │
    └──────────────────────────────────────────────────────────┘
        │               │               │               │
    ┌───▼────┐      ┌───▼────┐      ┌───▼────┐      ┌───▼────┐
    │ SF     │      │ SF     │      │ SF     │      │ SF     │
    │ 合成 A │      │ 合成 B │      │ 合成 C │      │ 合成 D │
    └────────┘      └────────┘      └────────┘      └────────┘
```

## 关键接口

### 函数
- `dequeueBuffer(width, height, format, usage, outSlot, outFence)` — 生产者获取空闲缓冲区槽位
  - 参数: width/height=缓冲区尺寸, format=像素格式(如RGBA_8888), usage=使用标志, outSlot=返回槽位号, outFence=返回 fence
  - 返回: OK 成功, NO_AVAILABLE_BUFFERS 无空闲槽位, WOULD_BLOCK 非阻塞模式下无可用缓冲区
  - 源码: `frameworks/native/libs/gui/BufferQueueProducer.cpp:BufferQueueProducer::dequeueBuffer()`
  - 注意: 返回的 fence 表示 GPU 正在使用该缓冲区，生产者必须等待 fence signal 后才能写入

- `queueBuffer(slot, input, output)` — 生产者提交已渲染帧
  - 参数: slot=槽位号, input=包含 fence/timestamp/crop/metadata, output=返回帧是否被消费
  - 返回: OK 成功
  - 源码: `frameworks/native/libs/gui/BufferQueueProducer.cpp:BufferQueueProducer::queueBuffer()`
  - 注意: 触发消费者（SurfaceFlinger）的 onFrameAvailable 回调

- `acquireBuffer(outBuffer, expectedPresent)` — 消费者获取待合成帧
  - 参数: outBuffer=返回 BufferItem(含 slot/GraphicBuffer/fence/timestamp), expectedPresent=期望显示时间
  - 返回: OK 成功, NO_BUFFER_AVAILABLE 无可用帧
  - 源码: `frameworks/native/libs/gui/BufferQueueConsumer.cpp:BufferQueueConsumer::acquireBuffer()`
  - 注意: SurfaceFlinger 在 VSync-SF 信号到达时调用

- `releaseBuffer(slot, fence)` — 消费者释放已使用帧
  - 参数: slot=槽位号, fence=释放 fence（GPU 合成完成后 signal）
  - 返回: OK 成功
  - 源码: `frameworks/native/libs/gui/BufferQueueConsumer.cpp:BufferQueueConsumer::releaseBuffer()`
  - 注意: fence signal 后缓冲区回到 FREE 状态，可被 dequeueBuffer 重新获取

### 数据结构
- `struct BufferItem` — BufferQueue 中缓冲区的元数据描述
  - 关键字段:
    - `mGraphicBuffer` — GraphicBuffer 对象（共享内存句柄）
    - `mFence` — Fence 对象（同步 GPU 操作）
    - `mTimestamp` — 帧时间戳（用于 VSync 偏移计算）
    - `mCrop` — 裁剪区域（Source Crop）
    - `mTransform` — 变换矩阵（旋转/翻转）
    - `mScalingMode` — 缩放模式
    - `mSlot` — 槽位号
  - 源码: `frameworks/native/libs/gui/include/gui/BufferItem.h`

- `class BufferQueueCore` — BufferQueue 核心状态管理
  - 关键字段:
    - `mSlots[MAX_BUFFER_COUNT]` — 缓冲区槽位数组（默认 64 个）
    - `mQueue` — 已排队帧的 BufferItem 队列
    - `mConnectedProducer` — 生产者连接状态
    - `mConnectedConsumer` — 消费者连接状态
    - `mMaxBufferCount` — 最大缓冲区数量（默认 2，三缓冲时为 3）
  - 源码: `frameworks/native/libs/gui/BufferQueueCore.h`

## 实现要点

### Fence 同步机制
BufferQueue 的核心同步原语是 **Fence**（对应 Linux kernel 的 sync_fence）：
- `dequeueBuffer` 返回的 fence：上一帧 GPU 渲染尚未完成，生产者必须 `wait(fence)` 后才能写入
- `queueBuffer` 传入的 fence：当前帧 GPU 渲染刚完成，消费者必须 `wait(fence)` 后才能读取
- `releaseBuffer` 传入的 fence：消费者 GPU 合成尚未完成，缓冲区不能被重新分配

### 跨进程共享
BufferQueue 通过 **Binder 传递 GraphicBuffer 的 fd（文件描述符）** 实现跨进程零拷贝共享：
- GraphicBuffer 内部持有 `buffer_handle_t`（native_handle），包含 DMA-BUF fd
- dequeueBuffer 通过 Binder 传递 fd 给生产者进程
- 生产者通过 mmap 将 DMA-BUF 映射到用户空间，GPU 直接写入

### BLASTBufferQueue (Android 12+)
传统 BufferQueue 中 SurfaceFlinger 是所有 Layer 的统一消费者，存在锁竞争。BLASTBufferQueue 的改进：
- 每个 Layer 独立拥有 BufferQueue，SurfaceFlinger 不再作为中间消费者
- 应用直接将缓冲区提交到 SurfaceFlinger 的 BufferQueue
- 减少了 SurfaceFlinger 主线程的 Binder 调用和锁竞争
- 源码: `frameworks/native/libs/gui/BLASTBufferQueue.cpp`

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 3.0 (Honeycomb) | BufferQueue 首次引入，替代 FrameBuffer 直接写入 |
| Android 4.3 | BufferQueue 支持异步模式（ASYNC 模式），允许生产者提前 dequeue |
| Android 7.0 | BufferQueue 重构为 BufferQueueProducer/Consumer/Core 三类分离 |
| Android 8.0 | 引入 BufferQueue::setMaxBufferCount 动态调整缓冲区数量 |
| Android 10 | 引入 BufferLayer 替代传统 BufferStateLayer |
| Android 12 | 引入 BLASTBufferQueue，每个 Layer 独立 BufferQueue |

## 陷阱与注意事项
- **双缓冲 vs 三缓冲**：双缓冲（maxBufferCount=2）在 App 渲染耗时超过 VSync 周期时会导致 SurfaceFlinger 无可用帧，产生掉帧。三缓冲增加一帧延迟但提升流畅度。
- **Fence 泄漏**：如果生产者忘记 wait dequeueBuffer 返回的 fence 就写入缓冲区，会导致画面撕裂。如果消费者忘记传递 releaseBuffer fence，会导致缓冲区过早回收。
- **BufferQueue 断连**：生产者或消费者进程崩溃时，BufferQueueCore 会检测到 death notification 并清理资源。但清理过程中可能短暂阻塞。
- ** dequeueBuffer 阻塞**：默认模式下 dequeueBuffer 在无可用缓冲区时会阻塞。UI 线程调用时必须使用非阻塞模式（`BUFFER_FLAG_NON_BLOCKING`），否则可能导致 ANR。

## 使用模式

### 应用端典型流程（RenderThread）
```cpp
// 1. dequeueBuffer - 获取空闲缓冲区
int slot;
sp<Fence> fence;
status_t err = mBufferQueue->dequeueBuffer(&slot, &fence,
    mWidth, mHeight, PIXEL_FORMAT_RGBA_8888, mUsage);

// 2. 等待 fence signal（GPU 完成上一帧）
if (fence != nullptr && fence->wait(300ms) != OK) {
    // fence 超时处理
}

// 3. 获取 GraphicBuffer 并映射
sp<GraphicBuffer> buffer = mBufferQueue->requestBuffer(slot);
ANativeWindowBuffer* anwb = buffer->getNativeBuffer();

// 4. GPU 渲染到缓冲区（通过 EGL/Vulkan）
// ... OpenGL draw calls or Vulkan rendering ...

// 5. queueBuffer - 提交已渲染帧
int64_t timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
BufferItem output;
mBufferQueue->queueBuffer(slot, { .mFence = fence, .mTimestamp = timestamp }, &output);
```

### SurfaceFlinger 端典型流程
```cpp
// 在 handleMessageRefresh() 中
void SurfaceFlinger::handleMessageRefresh() {
    // 1. 遍历所有 Layer，acquireBuffer
    for (auto& layer : mDrawingLayers) {
        BufferItem item;
        if (layer->mBufferQueue->acquireBuffer(&item, expectedPresent) == OK) {
            layer->setBuffer(item.mGraphicBuffer);
            // item.mFence 需要在合成前 wait
        }
    }

    // 2. 合成（GPU 或 HWC）
    doComposition(display);

    // 3. releaseBuffer
    for (auto& layer : mDrawingLayers) {
        mBufferQueue->releaseBuffer(layer->getSlot(), releaseFence);
    }
}
```

## Sources

- [P0] `frameworks/native/libs/gui/BufferQueueProducer.cpp` — BufferQueueProducer 实现：dequeueBuffer/queueBuffer 的完整逻辑
- [P0] `frameworks/native/libs/gui/BufferQueueConsumer.cpp` — BufferQueueConsumer 实现：acquireBuffer/releaseBuffer 的完整逻辑
- [P0] `frameworks/native/libs/gui/BufferQueueCore.h` — BufferQueueCore 定义：BufferSlot 状态机、mQueue、mMaxBufferCount
- [P0] `frameworks/native/libs/gui/include/gui/BufferItem.h` — BufferItem 结构体定义：GraphicBuffer/Fence/Timestamp/Crop
- [P0] `frameworks/native/libs/gui/BLASTBufferQueue.cpp` — BLASTBufferQueue 实现（Android 12+）
- [P1] https://androidperformance.com/en/2020/02/14/Android-Systrace-SurfaceFlinger — Android Systrace Basics - SurfaceFlinger Explained: BufferQueue 数据流详解
- [P1] https://source.android.com/docs/core/graphics/bufferqueue — Android 官方 BufferQueue 文档
- [P2] https://betterprogramming.pub/android-internals-for-rendering-a-view-430cd394e225 — Android Internals for Rendering a View: BufferQueue 生产者-消费者模型
