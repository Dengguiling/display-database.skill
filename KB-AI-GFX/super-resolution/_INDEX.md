# Super Resolution（AI 超分辨率）

## 概述

AI 超分辨率是实时图形渲染中的核心技术方向：利用 AI/ML 将低分辨率渲染帧重建为高分辨率输出。当前三大厂商方案为 NVIDIA DLSS、AMD FSR、Intel XeSS，Microsoft DirectSR 提供统一 D3D12 接口。

## 条目列表

| 条目 | 说明 | 关联 |
|------|------|------|
| [overview](overview.md) | AI 超分辨率总览：原理、厂商方案对比、DirectSR 统一接口 | → dlss, fsr, xess |
| [dlss](dlss.md) | NVIDIA DLSS：从 CNN 到 Transformer 的演进 | → fsr, xess |
| [fsr](fsr.md) | AMD FSR：从空间滤波到 ML 重建 | → dlss, xess |
| [xess](xess.md) | Intel XeSS：XMX/DP4a 双路径架构 | → dlss, fsr |
| [aisr](aisr.md) | 学术前沿：AISR/GameSR 等开源方案 | → dlss |

## 关联图

```
overview ──→ dlss ──→ frame-gen/dlss-frame-gen
   │          │
   │          ├────→ denoising/rt-denoise (Ray Reconstruction)
   │          │
   ├──→ fsr ──→ frame-gen/afmf
   │
   ├──→ xess ──→ frame-gen/xess-fg
   │
   └──→ aisr (学术参考)
```
