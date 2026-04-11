# PLAN.md — 藏经阁建设计划

> 本文档是藏经阁的建设路线图，持续迭代更新。
> 每完成一个阶段，更新状态为 ✅ 并记录实际产出。

---

## 阶段一：基础设施 ✅

> 建立藏经阁的物理结构和执行规范

| 任务 | 状态 | 产出 |
|------|------|------|
| Git 仓库初始化 | ✅ | `git@github.com:Dengguiling/self-knowledge-database.git` |
| 目录结构搭建 | ✅ | 7个知识领域目录 |
| SPEC.md 规范制定 | ✅ | 信源/模板/评审/变更管理规范 |
| PLAN.md 计划制定 | ✅ | 本文档 |
| SSH 通信配置 | ✅ | paramiko wrapper + ed25519 密钥 |

---

## 阶段二：DRM 子系统知识库（当前阶段）

> 以 Linux DRM/KMS 为首个完整试点，验证整套规范的可行性

### 2.1 子模块规划

| 子模块 | 说明 | 预计条目数 | 优先级 |
|--------|------|------------|--------|
| atomic/ | Atomic 模式与状态管理 | 5-8 | P0 |
| crtc/ | CRTC 子系统 | 3-5 | P0 |
| plane/ | Plane 子系统 | 3-5 | P0 |
| connector/ | Connector 与显示输出 | 3-5 | P1 |
| encoder/ | Encoder 子系统 | 2-3 | P1 |
| fb/ | Framebuffer 管理 | 2-3 | P1 |
| property/ | DRM Property 系统 | 2-3 | P1 |
| gem/ | GEM 内存管理 | 3-5 | P2 |
| dma-buf/ | DMA-BUF 共享内存 | 2-3 | P2 |
| debugfs/ | 调试接口 | 1-2 | P3 |

### 2.2 执行步骤

| # | 任务 | 状态 | 说明 |
|---|------|------|------|
| 1 | 创建 DRM 模块目录结构 | ⬜ | atomic/, crtc/, plane/ 等子目录 |
| 2 | 编写 DRM `_INDEX.md` | ⬜ | 模块概述 + 子模块导航 |
| 3 | 编写 DRM `_GLOSSARY.md` | ⬜ | CRTC/Plane/Encoder 等核心术语 |
| 4 | 编写 atomic/commit-flow.md | ⬜ | 首个完整条目，验证模板可行性 |
| 5 | 评审 commit-flow.md | ⬜ | 执行 8 项 checklist |
| 6 | 根据评审结果调整模板 | ⬜ | 若模板有问题，反馈到 SPEC.md |
| 7 | 批量填充 atomic/ 子模块 | ⬜ | state-mgmt, helpers, nonblocking 等 |
| 8 | 填充 crtc/ 子模块 | ⬜ | |
| 9 | 填充 plane/ 子模块 | ⬜ | |
| 10 | 填充其余子模块 | ⬜ | 按优先级逐步推进 |

### 2.3 信源清单

| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| `kernel/Documentation/gpu/drm-kms.rst` | P0 | KMS 总体架构 |
| `kernel/Documentation/gpu/drm-atomic.rst` | P0 | Atomic 模式 |
| `drivers/gpu/drm/drm_atomic.c` | P0 | Atomic 实现 |
| `drivers/gpu/drm/drm_atomic_helper.c` | P0 | Atomic Helper 实现 |
| `include/drm/drm_atomic.h` | P0 | Atomic 数据结构 |
| `include/drm/drm_crtc.h` | P0 | CRTC/Plane/Encoder 数据结构 |
| `drivers/gpu/drm/drm_plane.c` | P0 | Plane 实现 |
| LWN DRM 文章系列 | P2 | 补充背景知识 |

---

## 阶段三：GPU 驱动开发知识库

> 基于 DRM 基础，扩展到具体 GPU 驱动开发

| 子模块 | 说明 | 预计条目数 |
|--------|------|------------|
| driver-model/ | DRM 驱动模型（driver/load/probe） | 3-5 |
| modeset/ | Modeset 流程与实现 | 3-5 |
| hw-specific/ | 厂商特定实现（AMD/Intel/NVIDIA） | 5-10 |
| firmware/ | GPU 固件加载 | 2-3 |
| debug/ | GPU 驱动调试方法 | 2-3 |

---

## 阶段四：GPU 架构知识库

> 从驱动层上升到硬件架构层

| 子模块 | 说明 | 预计条目数 |
|--------|------|------------|
| overview/ | 现代 GPU 架构总览 | 2-3 |
| nvidia/ | NVIDIA GPU 架构（Turing/Ampere/Ada/Blackwell） | 5-8 |
| amd/ | AMD GPU 架构（RDNA/CDNA） | 5-8 |
| intel/ | Intel GPU 架构（Xe/Xe2） | 5-8 |
| memory/ | GPU 显存架构（HBM/GDDR/VRAM） | 2-3 |
| compute/ | 计算单元与调度 | 2-3 |

---

## 阶段五：图形 API 知识库

| 子模块 | 说明 | 预计条目数 |
|--------|------|------------|
| vulkan/ | Vulkan API 核心概念 | 5-8 |
| dx12/ | DirectX 12 API 核心概念 | 5-8 |
| opengl/ | OpenGL（遗留，仅保留关键差异） | 2-3 |
| swapchain/ | 画面呈现与同步 | 2-3 |
| shader/ | 着色器与管线 | 3-5 |

---

## 阶段六：AI 知识库

| 子模块 | 说明 | 预计条目数 |
|--------|------|------------|
| llm/ | 大语言模型架构（Transformer/注意力机制） | 5-8 |
| training/ | 训练框架与优化 | 3-5 |
| inference/ | 推理优化（量化/剪枝/蒸馏） | 3-5 |
| agent/ | AI Agent 框架与工具链 | 3-5 |
| chip/ | AI 芯片（NVIDIA GPU/TPU/NPU） | 3-5 |

---

## 运维计划

### 定期任务

| 任务 | 频率 | 说明 |
|------|------|------|
| 上游变更巡检 | 每周 | 检查内核/API/框架是否有 breaking change |
| 条目复评 | 每周 | 验证已有条目的时效性 |
| 评审摘要报告 | 每周 | 向 Ling 汇报本周入库/更新/过期情况 |
| 术语表维护 | 每月 | 检查术语一致性，补充新术语 |
| 知识库健康度检查 | 每月 | 检查断链、孤立条目、缺失索引 |

### 质量指标

| 指标 | 目标 |
|------|------|
| 信源覆盖率 | 100% 条目有 P0/P1 信源 |
| 评审通过率 | 首次评审通过率 > 80% |
| 交叉引用完整率 | 100% 引用有效，0 断链 |
| 版本时效性 | 100% 条目标注在 30 天内验证过 |

---

## 版本历史

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-11 | v1.0 | 初始计划，完成阶段一 |
