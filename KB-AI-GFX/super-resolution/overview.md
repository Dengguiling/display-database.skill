# AI Super Resolution Overview

> 利用 AI/ML 技术将低分辨率渲染帧重建为高分辨率输出，在保持画质的同时大幅降低 GPU 渲染负载。

## Meta
- **ID:** KB-AI-GFX-super-resolution-001
- **置信度:** 5
- **信源等级:** P1
- **验证日期:** 2026-04-12
- **适用版本:** DLSS 4.x (RTX 50 系列), FSR 4.x (RDNA 4), XeSS 2.x (Intel Arc), DirectSR Preview (D3D12 Agility SDK 1.714+)
- **关键词:** super resolution, upscaling, temporal reconstruction, DLSS, FSR, XeSS, DirectSR, TAA, neural upscaling
- **前置知识:** [→ KB-GPU-ARCH/gpu-rendering-pipeline]
- **关联条目:** [→ KB-AI-GFX/super-resolution/dlss] [→ KB-AI-GFX/super-resolution/fsr] [→ KB-AI-GFX/super-resolution/xess]

## 概述

AI 超分辨率（Super Resolution, SR）是实时图形渲染中的关键技术：以低于目标分辨率的内部渲染（如 1080p 渲染 → 4K 输出），通过时域信息积累与 AI 重建算法恢复细节，实现"以低分辨率算力获得接近原生高分辨率画质"。当前三大厂商方案为 NVIDIA DLSS、AMD FSR、Intel XeSS，Microsoft 推出的 DirectSR API 旨在统一跨厂商 SR 接口。

## 架构/调用链

```
游戏引擎渲染循环
    │
    ├─ 以低分辨率（如 1/2, 1/3, 1/4）渲染场景
    │   ├─ 生成: Color Buffer, Motion Vectors, Depth Buffer
    │   ├─ 可选: Exposure, Auto-Exposure, Ray Tracing Output
    │   └─ 输出: 低分辨率帧 + 辅助缓冲区
    │
    ▼
SR 模块（DLSS / FSR / XeSS / DirectSR）
    │
    ├─ Step 1: 时域累积 (Temporal Accumulation)
    │   └─ 利用 Motion Vectors 将历史帧信息重投影到当前帧
    │
    ├─ Step 2: 空间重建 (Spatial Reconstruction)
    │   ├─ 传统方案: 边缘感知滤波 + 锐化 (FSR 1.x, TAAU)
    │   └─ AI 方案: 神经网络推理 (DLSS, XeSS, FSR 4.x)
    │       ├─ DLSS: Transformer-based autoencoder (Tensor Core 加速)
    │       ├─ XeSS: 空间自适应网络 (XMX 引擎 / DP4a fallback)
    │       └─ FSR 4: ML-based 重建 (RDNA 4 硬件加速)
    │
    ├─ Step 3: 抗锯齿 + 细节恢复
    │   └─ 消除重影/闪烁，恢复边缘锐度
    │
    └─ 输出: 高分辨率帧 → 后处理 → 显示
```

### DirectSR 统一接口层

```
游戏引擎 (D3D12)
    │
    ▼
DirectSR API (DX12 Agility SDK)
    │
    ├─ 查询可用 SR 实现
    │   ├─ NVIDIA DLSS SR (driver-embedded, RTX 20+)
    │   ├─ Intel XeSS (driver-embedded, 11th Gen+ / Arc)
    │   └─ AMD FSR 2.2 (built-in, GPU-agnostic)
    │
    ├─ 统一输入: Color, Depth, MotionVectors, Exposure, OutputSize, RenderSize
    │
    └─ 统一输出: 超分辨率帧
```

## 关键接口

### DirectSR API (D3D12)

- `ID3D12Device::CheckFeatureSupport(D3D12_FEATURE_DIRECTSR, ...)` — 查询 DirectSR 支持状态
  - 参数: `D3D12_FEATURE_DATA_DIRECTSR *pFeatureSupportData`
  - 返回: S_OK 表示支持
  - 源码: DirectX 12 Agility SDK 1.714.0+

- `ID3D12Device5::CreateCommandList` + DirectSR 扩展 — 执行 SR 推理
  - 输入资源: Color (render resolution), Depth, Motion Vectors, Exposure
  - 输出资源: Color (display resolution)
  - 注意: 需要 GPU vendor driver 支持（NVIDIA 560.38+, Intel Arc driver）

### 引擎集成接口（各厂商 SDK）

- **DLSS:** `NVSDK_NGX_Parameter` — 设置输入/输出纹理、渲染分辨率、输出分辨率、质量模式
  - 质量模式: Ultra Performance (1/9), Performance (1/4), Balanced (1/3), Quality (1/2), DLAA (1/1)
  - 源码: NVIDIA NGX SDK / DLSS SDK

- **FSR:** `FfxFsr2Context` + `ffxFsr2ContextCreate` / `ffxFsr2Dispatch` — FSR 2.x API
  - 质量模式: Ultra Quality (1.3x), Quality (1.5x), Balanced (1.7x), Performance (2x), Ultra Performance (3x)
  - 源码: AMD GPUOpen FidelityFX SDK (MIT license, open source)

- **XeSS:** `XeSS_D3D12_Create` / `XeSS_D3D12_Run` — XeSS D3D12 接口
  - 质量模式: Ultra Quality (1.3x), Quality (1.5x), Balanced (1.7x), Performance (2x), Ultra Performance (3x)
  - 源码: Intel XeSS SDK

## 实现要点

### 时域重建核心原理

所有现代 SR 方案共享同一基础架构——**时域反锯齿（TAA）的增强版**：

1. **运动向量重投影（Motion Vector Reprojection）**：利用逐像素运动向量将前一帧的渲染结果投影到当前帧坐标空间，积累多帧采样信息
2. **邻域钳制（Neighborhood Clamping）**：对比当前帧与重投影历史帧的像素值，防止重影（ghosting），检测遮挡/显露区域
3. **空间滤波（Spatial Filtering）**：在重投影后的样本上执行边缘感知的空间滤波，利用邻域信息填补缺失采样

### AI 方案 vs 传统方案的核心差异

| 维度 | 传统方案 (TAAU/FSR 1.x) | AI 方案 (DLSS/XeSS/FSR 4) |
|------|------------------------|--------------------------|
| 重建算法 | 手工设计的边缘感知滤波器 | 神经网络（CNN/Transformer） |
| 细节恢复 | 依赖锐化后处理，易产生过锐化伪影 | 从训练数据学习高频细节模式 |
| 时域稳定性 | 邻域钳制启发式规则 | 网络隐式学习时序一致性 |
| 硬件依赖 | 无特殊硬件要求 | 需要专用 AI 加速单元 |
| 跨厂商兼容 | 完全兼容 | 部分兼容（DP4a fallback 或纯 shader） |

### DLSS 4.x Transformer 架构

DLSS 4 将 SR 模型从 CNN 升级为 Transformer-based autoencoder：
- 利用 Transformer 的注意力机制建模复杂空间关系，在稀疏采样模式下保持更一致的画质
- 第五代 Tensor Core 提供硬件加速，FP4/FP8 推理精度
- 支持 Multi Frame Generation（每渲染帧生成 3 个额外帧），实现 4x 帧率提升

### FSR 4.x ML 转型

FSR 4 标志着 AMD 从纯算法方案转向 ML-based 重建：
- 利用 RDNA 4 架构的 AI 加速器（硬件矩阵运算单元）
- 提供 Ray Regeneration（光线重建）作为光线追踪降噪方案
- FSR 4.1 增加了更精细的细节恢复和更高的帧率

### XeSS 2.x 架构

XeSS 使用空间自适应神经网络：
- Intel XMX（Xe Matrix Extensions）提供 INT8/FP16 矩阵加速
- 无 XMX 硬件时 fallback 到 DP4a（4 元素点积）shader 实现
- XeSS 2 引入 Frame Generation（帧生成）和 XeLL（低延迟）

## 版本差异

| 版本 | 变更 |
|------|------|
| DLSS 1.0 (2019) | 首版，基于 CNN，需要游戏特定训练 |
| DLSS 2.0 (2020) | 通用模型，无需 per-game 训练，Temporal AI Super Resolution |
| DLSS 3.0 (2022) | 新增 Frame Generation（光流法插帧）+ Reflex |
| DLSS 3.5 (2023) | 新增 Ray Reconstruction（AI 光追降噪） |
| DLSS 4.0 (2025) | Transformer-based SR + Multi Frame Generation (3 frames/gen)，需 RTX 50 系列 |
| DLSS 4.5 (2025) | 第二代 Transformer SR 模型，6x MFG 模式，Dynamic MFG |
| FSR 1.0 (2021) | 纯空间放大 + CAS 锐化，无时域信息 |
| FSR 2.0 (2022) | 时域放大，开源，类 TAA 架构 |
| FSR 3.0 (2023) | 新增 Frame Generation + AFMF (driver-level) |
| FSR 4.0 (2025) | ML-based 重建，需 RDNA 4 硬件加速 |
| FSR 4.1 (2025) | 改进 Ray Regeneration，更精细细节恢复 |
| XeSS 1.0 (2022) | 首版，XMX/DP4a 双路径 |
| XeSS 2.0 (2024) | 新增 Frame Generation + XeLL 低延迟 |
| XeSS 3.0 (2025) | 改进帧生成质量，扩展硬件支持 |
| DirectSR Preview (2024.05) | 统一 D3D12 SR API，内置 FSR 2.2，driver 支持 DLSS/XeSS |

## 陷阱与注意事项

1. **运动向量精度至关重要**：SR 质量高度依赖运动向量的准确性。引擎必须提供逐像素、亚像素精度的运动向量；物体运动过快或运动向量缺失的区域会出现重影
2. **UI/HUD 层不能经过 SR 处理**：文字、图标等 UI 元素必须在 SR 之后叠加，否则会出现模糊/闪烁。引擎需将 UI 渲染到独立 layer
3. **Alpha 混合区域质量下降**：半透明物体（粒子、烟雾）的 SR 重建质量通常低于不透明物体，因为运动向量在 alpha 混合区域不够准确
4. **DirectSR 当前仅支持 upscaler**：DirectSR Preview 仅统一超分辨率接口，Frame Generation 和 Ray Reconstruction 仍需各厂商独立 SDK
5. **FSR 4.x 不向后兼容旧硬件**：FSR 4 的 ML 重建依赖 RDNA 4 的 AI 加速器，在 RDNA 2/3 上需 fallback 到 FSR 3.x
6. **DLSS 4 MFG 仅限 RTX 50 系列**：Multi Frame Generation 需要第五代 Tensor Core，RTX 20/30/40 系列仅支持 SR 和单帧生成

## 使用模式

### 引擎集成 SR 的标准流程

```cpp
// 1. 初始化 SR 模块（以 DirectSR 为例）
D3D12_FEATURE_DATA_DIRECTSR featureData = {};
device->CheckFeatureSupport(D3D12_FEATURE_DIRECTSR, &featureData, sizeof(featureData));

// 2. 每帧渲染：以低分辨率渲染场景
//    - 输出: Color (renderWidth x renderHeight)
//    - 输出: MotionVectors (renderWidth x renderHeight)
//    - 输出: Depth (renderWidth x renderHeight)

// 3. 执行 SR 重建
//    - 输入: Color, Depth, MotionVectors, Exposure
//    - 输出: UpscaledColor (displayWidth x displayHeight)

// 4. 后处理 + UI 叠加 → 显示
```

### 质量模式选择策略

| 场景 | 推荐模式 | 缩放比 | 说明 |
|------|----------|--------|------|
| 追求极致画质 | Quality / DLAA | 1.5x-1.7x | 画质损失最小 |
| 平衡画质与性能 | Balanced | 1.7x-2x | 最常用模式 |
| 高帧率需求 | Performance | 2x-3x | 适合竞技游戏 |
| 极限性能 | Ultra Performance | 3x-4x | 画质牺牲较大 |

## Sources

- [P1] https://research.nvidia.com/labs/adlr/DLSS4 — DLSS 4 技术概述：Transformer-based SR、Multi Frame Generation、Ray Reconstruction
- [P1] https://developer.nvidia.com/blog/nvidia-dlss-4-5-delivers-super-resolution-upgrades-and-new-dynamic-multi-frame-generation — DLSS 4.5：第二代 Transformer SR 模型、6x MFG、Dynamic MFG
- [P1] https://www.nvidia.com/en-us/geforce/technologies/dlss — NVIDIA DLSS 官方技术页面：版本历史、质量模式、硬件支持
- [P1] https://gpuopen.com/learn/amd-fsr4-gpuopen-release — AMD FSR 4 GPUOpen 发布：ML-based 重建、RDNA 4 硬件加速
- [P1] https://gpuopen.com/manuals/fsr_sdk/techniques/super-resolution-ml — AMD FSR Upscaling 4.1.0 SDK 手册：技术细节与集成指南
- [P1] https://www.intel.com/content/www/us/en/developer/articles/guide/xe-super-sampling-developer-guide.html — Intel XeSS API Developer Guide v1.1：XMX/DP4a 双路径架构
- [P1] https://www.intel.com/content/www/us/en/developer/topic-technology/gamedev/xess.html — Intel XeSS 3 开发者页面：Frame Generation、硬件支持范围
- [P1] https://devblogs.microsoft.com/directx/directsr-preview — Microsoft DirectSR Preview：统一 D3D12 SR API 设计、多厂商支持
- [P1] https://microsoft.github.io/DirectX-Specs/DirectSR/DirectSR.html — DirectSR 规范文档：API 接口定义、输入输出规范
- [P1] https://www.nvidia.com/en-us/geforce/news/nvidia-dlss-3-5-ray-reconstruction — DLSS 3.5 Ray Reconstruction：AI 光追降噪替代手工降噪器
- [P2] https://wccftech.com/roundup/nvidia-dlss-vs-amd-fsr-vs-intel-xess-everything-you-need-to-know — DLSS vs FSR vs XeSS 全版本对比综述
