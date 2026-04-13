# Android VSync 模型

> Android 基于 VSync 信号的帧同步机制，通过 Choreographer 协调 App 渲染与 SurfaceFlinger 合成的时序。

## Meta
- **ID:** KB-ANDROID-GFX-surfaceflinger-003
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 9.0+（DispSync 模型），Android 11+ 引入 VsyncModulator/FrameTimeline，Android 14+ 引入 VsyncDispatch
- **关键词:** VSync, Choreographer, DispSync, VsyncDispatch, vsync-app, vsync-sf, vsync-offset, EventThread, DisplayEventReceiver, frame pacing, VsyncModulator, FrameTimeline
- **前置知识:** [→ KB-ANDROID-GFX-surfaceflinger-001] [→ KB-ANDROID-GFX-surfaceflinger-002]
- **关联条目:** [→ KB-ANDROID-GFX-hwcomposer-001]

## 概述

Android VSync 模型是整个 UI 渲染管线的时序基础。硬件 VSync 信号由 Display HAL 通过 HWC 产生，SurfaceFlinger 接收后通过 DispSync（Android 9-）或 VsyncDispatch（Android 14+）软件模型生成两个偏移信号：**vsync-app**（通知 App 开始渲染）和 **vsync-sf**（通知 SurfaceFlinger 开始合成）。App 端通过 Choreographer 接收 vsync-app，协调 Input/Animation/Draw 三个回调的执行。整个管线设计确保：App 渲染 → GPU 绘制 → SF 合成 → Display 显示，在单个 VSync 周期内流水线化完成。

## 架构/调用链

### VSync 信号分发链

```
Display Hardware (Panel)
    │
    │ VSync HW interrupt
    ▼
HWC HAL (hardware/interfaces/composer)
    │
    │ onVsyncReceived(timestamp)
    ▼
SurfaceFlinger
    │
    ├── DispSync / VsyncDispatch (软件 VSync 模型)
    │   ├── mPeriod: VSync 周期 (16.6ms @60Hz)
    │   ├── mPhase: 相位偏移
    │   └── mReferenceTime: 参考时间
    │
    ├── vsync-app (偏移量: ~2ms before VSync)
    │   └── EventThread → BitTube → DisplayEventReceiver
    │       └── Choreographer.doFrame()
    │           ├── CALLBACK_INPUT: 处理输入事件
    │           ├── CALLBACK_ANIMATION: 执行动画
    │           └── CALLBACK_TRAVERSAL: 执行 View 树绘制
    │               └── RenderThread.dequeueBuffer() → GPU 渲染
    │
    └── vsync-sf (偏移量: ~1ms before VSync)
        └── EventThread → SurfaceFlinger.onMessageReceived(INVALIDATE)
            ├── acquireBuffer() (获取 App 提交的帧)
            ├── 合成 (GPU/HWC)
            └── 提交到 Display
```

### 单帧时序图（60Hz）

```
时间轴 (ms):
0        2        6        10       14       16.6     18.6
│        │        │        │        │        │        │
│←vsync-app→│                │                │        │
│  Input   │  Animation     │                │        │
│  Event   │  Callback      │                │        │
│        │←──Draw──→│        │                │        │
│        │  Callback │  GPU   │                │        │
│        │  (measure/  │ Render│                │        │
│        │  layout/    │ Thread│                │        │
│        │  draw)      │        │                │        │
│        │        queueBuffer()                │        │
│        │            │←vsync-sf→│        │        │
│        │            │  acquire  │        │        │
│        │            │  Buffer   │        │        │
│        │            │  合成(GPU)│        │        │
│        │            │        │提交Display│        │
│        │            │        │        │←VSync HW→│
│        │            │        │        │  显示帧A  │
│        │            │        │        │        │←vsync-app→
│        │            │        │        │        │  渲染帧B
```

### Choreographer 内部流程

```
Choreographer.doFrame()
    │
    ├── doCallbacks(CALLBACK_INPUT, frameData)
    │   └── InputEventReceiver.dispatchInputEvent()
    │       └── ViewRootImpl.deliverInputEvent()
    │
    ├── doCallbacks(CALLBACK_ANIMATION, frameData)
    │   └── ValueAnimator.doAnimationFrame()
    │       └── PropertyValuesHolder.setAnimatedValue()
    │
    └── doCallbacks(CALLBACK_TRAVERSAL, frameData)
        └── ViewRootImpl.performTraversals()
            ├── performMeasure() → View.measure()
            ├── performLayout()  → View.layout()
            └── performDraw()    → View.draw()
                └── ThreadedRenderer.draw()
                    └── RenderThread.queueBuffer()
```

## 关键接口

### 函数
- `Choreographer.postFrameCallback(callback)` — 注册 VSync 回调
  - 参数: callback=FrameCallback 接口，doFrame 时触发
  - 返回: void
  - 源码: `frameworks/base/core/java/android/view/Choreographer.java:Choreographer.postFrameCallback()`
  - 注意: 回调在 vsync-app 信号到达时执行，在主线程

- `Choreographer.doFrame(frameTimeNanos, frameIntervalNanos)` — 处理一帧的所有回调
  - 参数: frameTimeNanos=帧时间戳, frameIntervalNanos=帧间隔
  - 返回: void
  - 源码: `frameworks/base/core/java/android/view/Choreographer.java:Choreographer.doFrame()`
  - 注意: 依次执行 INPUT → ANIMATION → TRAVERSAL 三个回调阶段

- `SurfaceFlinger::onVsyncReceived(sequenceId, displayId, timestamp)` — 接收硬件 VSync
  - 参数: sequenceId=Composer 序列号, displayId=显示设备 ID, timestamp=VSync 时间戳
  - 返回: void
  - 源码: `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:SurfaceFlinger::onVsyncReceived()`
  - 注意: 将硬件 VSync 喂给 DispSync/VsyncDispatch 模型

- `DispSync::addResyncSample(timestamp)` — 向 DispSync 模型添加硬件 VSync 样本
  - 参数: timestamp=硬件 VSync 时间戳
  - 返回: bool（true=需要继续接收硬件 VSync 校准，false=模型已锁定）
  - 源码: `frameworks/native/services/surfaceflinger/DispSync.cpp:DispSync::addResyncSample()`
  - 注意: DispSync 使用最近 6 个样本计算 mPeriod 和 mPhase

### 数据结构
- `class DispSync` — 软件 VSync 模型（Android 9-）
  - 关键字段:
    - `mPeriod` — VSync 周期（ns），如 16666667ns @60Hz
    - `mPhase` — 相位偏移（ns）
    - `mReferenceTime` — 参考时间戳
    - `mResyncSamples[MAX_RESYNC_SAMPLES]` — 硬件 VSync 样本缓冲区
  - 源码: `frameworks/native/services/surfaceflinger/DispSync.h`

- `class VsyncDispatch` — VSync 调度器（Android 14+，替代 DispSync）
  - 关键字段:
    - `mTimerSlot` — 定时器槽位
    - `mCallbacks` — 注册的回调列表
    - `mPeriod` — VSync 周期
  - 源码: `frameworks/native/services/surfaceflinger/Scheduler/VsyncDispatch.h`

## 实现要点

### DispSync 内部模型（Android 9-）
DispSync 维护硬件 VSync 的软件模型，核心算法：
1. **周期计算**：收集最近 6 个硬件 VSync 时间戳，去掉最大最小值后取平均，得到 mPeriod
2. **相位计算**：使用"平均角度法"（mean of angles）计算 mPhase，减少噪声影响
3. **模型校准**：通过 addPresentFence 检查模型误差，误差超过阈值时重新启用硬件 VSync 校准
4. **偏移分发**：DispSyncThread 根据 mPeriod/mPhase/mReferenceTime 计算各回调的目标时间，在精确时刻触发 vsync-app 和 vsync-sf

### VsyncModulator（Android 11+）
动态调整 VSync 偏移量以优化特定场景：
- **低延迟模式**：减小 vsync-app 偏移，让 App 更早开始渲染（适合触摸响应）
- **高帧率模式**：增大偏移，给 App 更多渲染时间（适合动画密集场景）
- 源码: `frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.h`

### FrameTimeline（Android 11+）
追踪每帧在管线各阶段的耗时，用于 Jank 检测和性能分析：
- 记录每帧的 expected vs actual present time
- SurfaceFlinger 在合成时打时间戳
- 通过 `adb shell dumpsys gfxinfo` 查看 FrameTimeline 数据

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 4.1 (JB) | 引入 VSync + Choreographer + Triple Buffering，解决 Project Butter 卡顿 |
| Android 5.0 | 引入 Choreographer 的 INPUT/ANIMATION/TRAVERSAL 三阶段回调 |
| Android 9.0 | DispSync 模型稳定，支持多刷新率 |
| Android 11 | 引入 VsyncModulator 动态偏移 + FrameTimeline 帧追踪 |
| Android 12 | 引入 SurfaceControl::setFrameRate API，App 可请求目标帧率 |
| Android 14 | VsyncDispatch 替代 DispSync，支持更灵活的多回调调度 |

## 陷阱与注意事项
- **vsync-app 偏移量**：vsync-app 通常比硬件 VSync 提前约 2ms，给 App 留出渲染时间。如果 App 渲染超过 VSync 周期（16.6ms@60Hz），帧会被推迟到下一个 VSync，产生掉帧。
- **Choreographer 回调顺序**：INPUT → ANIMATION → TRAVERSAL 是固定顺序。如果 INPUT 回调耗时过长，会挤压 ANIMATION 和 TRAVERSAL 的时间。
- **多刷新率设备**：在 120Hz 设备上，VSync 周期为 8.3ms，App 渲染时间预算减半。需要配合 setFrameRate API 动态调整。
- **VSync 关闭**：当屏幕关闭时，SurfaceFlinger 会停止分发 VSync。App 的 Choreographer 不会收到回调，渲染暂停。

## 使用模式

### App 端使用 Choreographer
```java
// 注册 VSync 回调
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        // 在 VSync 信号到达时执行
        // frameTimeNanos 是帧的预期显示时间
        doWork();
        // 注册下一帧回调
        Choreographer.getInstance().postFrameCallback(this);
    }
});
```

### 调试 VSync 问题
```bash
# 查看 VSync 状态
adb shell dumpsys SurfaceFlinger --vsync

# 查看 FrameTimeline（Jank 检测）
adb shell dumpsys gfxinfo <package>

# Perfetto 中查看 VSync 信号
# 关注: vsync-app, vsync-sf, Choreographer.doFrame 的时序关系
```

## Sources

- [P0] `frameworks/native/services/surfaceflinger/DispSync.cpp` — DispSync 软件模型实现：addResyncSample/updateModelLocked
- [P0] `frameworks/native/services/surfaceflinger/DispSync.h` — DispSync 类定义：mPeriod/mPhase/mReferenceTime
- [P0] `frameworks/native/services/surfaceflinger/EventThread.h` — EventThread VSync 分发机制
- [P0] `frameworks/base/core/java/android/view/Choreographer.java` — Choreographer 实现：doFrame/INPUT/ANIMATION/TRAVERSAL
- [P0] `frameworks/native/services/surfaceflinger/Scheduler/VsyncDispatch.h` — VsyncDispatch 定义（Android 14+）
- [P0] `frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.h` — VsyncModulator 动态偏移
- [P1] https://utzcoz.github.io/2020/05/02/Analyze-AOSP-vsync-model.html — Analyze AOSP vsync model: DispSync 内部模型深度分析（基于 Android 9.0）
- [P1] https://source.android.com/docs/core/graphics/vsync — Android 官方 VSync 文档
- [P2] https://androidperformance.com/en/2020/02/14/Android-Systrace-SurfaceFlinger — Android Systrace Basics: VSync 信号在 Systrace 中的表现
