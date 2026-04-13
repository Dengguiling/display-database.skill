# HWC 合成类型详解

> HWC2 定义的四类合成方式：Device/Client/Cursor/Sideband，以及 SurfaceFlinger 的合成策略选择。

## Meta
- **ID:** KB-ANDROID-GFX-hwcomposer-002
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Android 7.0+（HWC2.0 Composition 枚举），Android 14+（Composer 3.x）
- **关键词:** Composition, Device Composition, Client Composition, Cursor, Sideband, overlay plane, FBTarget, GPU composition, RenderEngine, GLES composition, pre-multiplied alpha
- **前置知识:** [→ KB-ANDROID-GFX-hwcomposer-001]
- **关联条目:** [→ KB-ANDROID-GFX-surfaceflinger-001]

## 概述

HWC2 定义了四种合成类型（Composition enum），SurfaceFlinger 根据每个 Layer 的属性和 HWC 的能力选择最优合成方式。**Device Composition** 利用显示硬件的 overlay plane 直接合成，功耗最低；**Client Composition** 通过 GPU（RenderEngine/GLES）合成，灵活性最高；**Cursor** 是低延迟的光标 overlay；**Sideband** 用于视频解码器直接输出。SurfaceFlinger 的合成策略核心目标是：最大化 Device Composition 比例，最小化 GPU 使用。

## 架构/调用链

### 四种合成类型

```
┌──────────────────────────────────────────────────────────────┐
│                    HWC 合成类型决策树                          │
│                                                              │
│  Layer 属性检查                                               │
│  ├── 是否是光标层？ ──────────────── YES → CURSOR            │
│  │   (小尺寸、低延迟、跟随指针)                                │
│  │                                                          │
│  ├── 是否是视频 Sideband？ ───────── YES → SIDEBAND          │
│  │   (视频解码器直接输出到显示)                                │
│  │                                                          │
│  ├── HWC 是否支持该 Layer？                                   │
│  │   ├── 有可用 overlay plane？                               │
│  │   ├── 格式是否支持？(RGBA/YUV/RGBX)                       │
│  │   ├── 是否需要 per-pixel alpha？                          │
│  │   ├── 是否需要缩放/旋转？                                  │
│  │   ├── 是否超出屏幕范围？                                   │
│  │   └── 是否是 DRM 保护内容？                                │
│  │                                                          │
│  ├── 全部满足 ────────────────────── YES → DEVICE            │
│  │   (HWC overlay 直接合成)                                   │
│  │                                                          │
│  └── 任一不满足 ──────────────────── NO  → CLIENT            │
│      (GPU RenderEngine 合成 → FBTarget → HWC 显示)            │
└──────────────────────────────────────────────────────────────┘
```

### 典型合成场景

```
场景 1: 简单 UI（状态栏 + 应用 + 导航栏）
┌─────────────────────────────────┐
│ 状态栏    │  DEVICE (overlay 0)  │
├───────────┤                      │
│           │                      │
│  应用层    │  DEVICE (overlay 1)  │
│           │                      │
├───────────┤                      │
│ 导航栏    │  DEVICE (overlay 2)  │
└─────────────────────────────────┘
→ 3 个 overlay，0 次 GPU 合成，功耗最低

场景 2: 复杂 UI（带 alpha 混合 + 视频播放）
┌─────────────────────────────────┐
│ 状态栏    │  DEVICE (overlay 0)  │
├───────────┤                      │
│  半透明    │                      │
│  对话框    │  CLIENT (GPU 合成)   │ ← alpha 混合需要 GPU
│           │  → FBTarget          │
├───────────┤                      │
│ 视频层    │  SIDEBAND (解码器)   │ ← 视频直接输出
└─────────────────────────────────┘
→ 1 overlay + 1 GPU + 1 Sideband

场景 3: 全屏游戏（隐藏系统栏）
┌─────────────────────────────────┐
│                                 │
│                                 │
│  游戏画面  │  DEVICE (overlay 0)  │
│                                 │
│                                 │
└─────────────────────────────────┘
→ 1 个 overlay，最优路径
```

## 关键接口

### 函数
- `setLayerCompositionType(display, layer, type)` — 设置 Layer 的合成类型
  - 参数: display=显示设备, layer=Layer handle, type=Composition 枚举值
  - 返回: void
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: SurfaceFlinger 在 validateDisplay 后根据 HWC 返回值设置

- `setLayerBlendMode(display, layer, mode)` — 设置 Layer 的混合模式
  - 参数: mode=BlendMode 枚举（NONE/PREMULTIPLIED/COVERAGE）
  - 返回: void
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/IComposer.aidl`
  - 注意: PREMULTIPLIED 是最常用模式，要求 Layer 数据已经预乘 alpha

### 数据结构
- `enum BlendMode` — 混合模式
  - 关键值:
    - `NONE` — 无混合（不透明）
    - `PREMULTIPLIED` — 预乘 alpha 混合（标准模式）
    - `COVERAGE` — Coverage 混合（旧模式，不推荐）
  - 源码: `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/BlendMode.aidl`

## 实现要点

### SurfaceFlinger 合成策略
SurfaceFlinger::rebuildLayerStacks() 中的决策逻辑：
1. **收集可见 Layer**：根据 Z-order 和可见区域过滤
2. **优先 Device Composition**：简单 Layer（无 alpha、无缩放、格式支持）优先分配 overlay
3. **强制 Client 的条件**：
   - Layer 有 per-pixel alpha 且 HWC 不支持
   - Layer 需要非轴对齐旋转
   - Layer 使用非标准像素格式
   - overlay plane 数量不足
4. **FBTarget 处理**：所有 Client Layer 由 GPU 合成到 FBTarget，FBTarget 作为特殊 Layer 交给 HWC

### RenderEngine GPU 合成
当 Layer 需要 Client Composition 时，SurfaceFlinger 使用 RenderEngine：
- RenderEngine 基于 OpenGL ES（或 Vulkan，Android 12+）
- 支持的混合操作：alpha blending、color space conversion、rounded corners、blur
- GPU 合成结果写入 FBTarget GraphicBuffer
- FBTarget 通过 setClientTarget() 传给 HWC

### Sideband 合成
Sideband 合成用于视频播放场景：
- 视频解码器（MediaCodec）直接将解码帧送到显示硬件
- 不经过 SurfaceFlinger 的 BufferQueue
- 通过 setLayerSidebandStream() 设置视频流句柄
- 适用于 DRM 保护视频（GPU 无法访问解码帧）

## 版本差异
| 版本 | 变更 |
|------|------|
| Android 7.0 | HWC2.0 引入四种 Composition 枚举 |
| Android 8.0 | 增加 BlendMode 支持 |
| Android 10 | RenderEngine 支持 Vulkan 后端（实验性） |
| Android 12 | RenderEngine Vulkan 后端正式可用，支持 Skia 渲染 |
| Android 14 | Composer 3.x，Composition 类型不变，接口优化 |

## 陷阱与注意事项
- **overlay plane 数量限制**：大多数 HWC 实现支持 3-6 个 overlay plane。当 Layer 数量超过 overlay 数量时，多余的 Layer 必须回退到 GPU 合成。
- **per-pixel alpha**：如果 HWC 不支持 per-pixel alpha 的 Device Composition，带 alpha 的 Layer 必须使用 Client Composition。这是最常见的回退原因。
- **FBTarget 格式**：FBTarget 的像素格式必须与显示设备匹配。如果 Layer 使用了 HDR 格式但显示不支持，GPU 合成时需要进行 tone mapping。
- **预乘 alpha**：使用 PREMULTIPLIED 混合模式时，Layer 数据必须是预乘后的值（color * alpha）。如果数据未预乘，会产生暗边。

## 使用模式

### 查看 Layer 合成类型
```bash
# dumpsys SurfaceFlinger 输出中的 HWC 合成信息
adb shell dumpsys SurfaceFlinger | grep -A 20 "HWC layers"

# 输出示例:
# HWC layers:
#   com.android.systemui#0 | Z=210000 | Comp=DEVICE | ...
#   com.example.app#0     | Z=100000 | Comp=CLIENT | ...
#   SurfaceView#0         | Z=0      | Comp=SIDEBAND | ...

# Perfetto 中查看:
# SurfaceFlinger::rebuildLayerStacks
# SurfaceFlinger::doComposition
# 关注每个 Layer 的 Composition 类型
```

## Sources

- [P0] `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/Composition.aidl` — Composition 枚举定义
- [P0] `hardware/interfaces/composer/aidl/android/hardware/graphics/composer3/BlendMode.aidl` — BlendMode 枚举定义
- [P0] `frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp` — SurfaceFlinger HWC 合成策略实现
- [P0] `frameworks/native/services/surfaceflinger/RenderEngine/RenderEngine.h` — RenderEngine GPU 合成接口
- [P1] https://source.android.com/docs/core/graphics/hwc — Android 官方 HWC 文档：合成类型说明
- [P2] https://www.programmersought.com/article/86921043167 — Android P graphic display system: HWC2 合成类型详解
