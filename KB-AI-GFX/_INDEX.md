# KB-AI-GFX — AI 图形显示技术

## 概述

KB-AI-GFX 覆盖 AI/ML 技术在实时图形渲染与显示领域的应用，包括超分辨率重建、帧生成、AI 降噪、神经渲染和 AI 编译优化。本模块聚焦于图显相关 AI 技术，不涉及通用 LLM/训练框架。

## 子模块导航

| 子模块 | 说明 | 状态 |
|--------|------|------|
| [super-resolution](super-resolution/) | AI 超分辨率：DLSS/FSR/XeSS/DirectSR | 🟡 建设中 |
| frame-gen | 帧生成：DLSS MFG/AFMF/XeSS-FG | ⬜ 待建设 |
| denoising | AI 降噪：Ray Reconstruction/RT Denoise | ⬜ 待建设 |
| neural-rendering | 神经渲染：Neural Radiance Fields | ⬜ 待建设 |
| ai-compilation | AI 编译：Shader AI 编译优化 | ⬜ 待建设 |

## 快速定位

| 你可能的问题 | 定位 |
|-------------|------|
| DLSS/FSR/XeSS 的核心原理和区别？ | [→ super-resolution/overview] |
| DLSS 4 的 Transformer 架构有什么改进？ | [→ super-resolution/dlss] |
| FSR 4 的 ML 重建如何工作？ | [→ super-resolution/fsr] |
| XeSS 的 XMX 和 DP4a 双路径是什么？ | [→ super-resolution/xess] |
| DirectSR 统一了哪些功能？ | [→ super-resolution/overview] |
| DLSS Multi Frame Generation 如何实现？ | [→ frame-gen/dlss-frame-gen] |
| AMD AFMF 驱动级帧生成原理？ | [→ frame-gen/afmf] |
| AI 光追降噪如何替代传统降噪器？ | [→ denoising/rt-denoise] |
