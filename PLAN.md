# PLAN.md — 藏经阁持续运营计划

> 本文档是藏经阁的**活文档**，记录建设路线、每日执行状态、质量指标。
> 每次任务执行后自动更新，通过 git history 追溯所有变更。

---

## 1. 总体架构

### 1.1 每日知识更新流水线

```
每日定时触发（03:00 Asia/Shanghai）
    │
    ▼
┌─────────────────────────────────┐
│  总调度：读取 PLAN.md + _PROGRESS.md  │
│  识别今日需要更新的知识方向            │
│  生成 3 个方向的子任务               │
└──────────┬──────────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
  ┌─────┐┌─────┐┌─────┐
  │ AI  ││ GPU ││Linux│  ← 3 个子 Agent 并行
  │知识  ││图显  ││驱动  │
  └──┬──┘└──┬──┘└──┬──┘
     │     │     │
     ▼     ▼     ▼
  抓取数据（P0/P1 优先）
     │
     ▼
  编写知识条目（遵循 SPEC.md 模板）
     │
     ▼
  ┌──────────────┐
  │  评审 Agent   │ ← 8 项 checklist
  └──┬───────┬───┘
     │       │
   PASS    FAIL
     │       │
     ▼       ▼
  入库     打回修改
  commit   → 重新评审
  push     → 直到 PASS
     │
     ▼
  更新 _PROGRESS.md
     │
     ▼
  所有方向完成 → 暂停今日任务
```

### 1.2 上下文窗口管理

子 Agent 在执行过程中必须监控上下文消耗：
- 每完成一个条目后，评估剩余上下文
- 若剩余不足 30%，立即保存进度到 `_PROGRESS.md`
- 标注断点：当前子模块、当前条目、已完成列表、待完成列表
- 下次触发时从断点恢复，不重复已完成的工作

### 1.3 评审机制

评审是**自动执行**的，不需要 Ling 人工审批：
- 每个条目完成后立即触发评审
- 评审严格遵循 SPEC.md 第8节的 8 项 checklist
- PASS → 入库；FAIL → 打回修改，最多重试 3 次
- 3 次仍不通过 → 标记为 `[需人工介入]`，在日报中汇报给 Ling
- 所有评审结果记录到对应模块的 `_REVIEW-LOG.md`

---

## 2. 知识方向与优先级

### 2.1 三大方向

| 方向 | 模块 | 子模块 | 当前状态 |
|------|------|--------|----------|
| **Linux 驱动** | KB-LINUX-DRM | atomic, crtc, plane, connector, encoder, fb, property, gem, dma-buf | 🟡 建设中 |
| **GPU/图显** | KB-LINUX-GPU-DRIVER, KB-GPU-ARCH, KB-GFX-API | driver-model, modeset, hw-specific, vulkan, dx12 | ⬜ 待启动 |
| **AI** | KB-AI-LLM, KB-AI-INFRA, KB-AI-AGENT | llm, training, inference, agent, chip | ⬜ 待启动 |

### 2.2 方向识别逻辑

每日触发时，按以下优先级识别需要更新的方向：

1. **断点恢复**：`_PROGRESS.md` 中有未完成的任务 → 优先恢复
2. **上游变更**：检查内核/API/框架是否有新版本发布 → 触发相关条目复评
3. **知识缺口**：PLAN.md 中标记为 ⬜ 的子模块 → 按优先级推进
4. **Ling 反馈**：日常对话中 Ling 提到的知识盲区 → 优先补充

---

## 3. 子 Agent 任务分配

### 3.1 Agent-A：Linux 驱动方向

**负责模块：** KB-LINUX-DRM, KB-LINUX-GPU-DRIVER

**信源清单：**
| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| `kernel/Documentation/gpu/` | P0 | DRM/KMS 全部文档 |
| `drivers/gpu/drm/` | P0 | DRM 核心实现 |
| `include/drm/` | P0 | DRM 数据结构与 API |
| LKML patch & discussion | P2 | 最新变更与设计决策 |
| LWN GPU/DRM 文章 | P3 | 背景补充 |

**当前子任务队列：**
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | atomic | commit-flow | ⬜ |
| 2 | atomic | state-mgmt | ⬜ |
| 3 | atomic | helpers | ⬜ |
| 4 | atomic | nonblocking | ⬜ |
| 5 | crtc | crtc-state | ⬜ |
| 6 | crtc | vblank | ⬜ |
| 7 | plane | plane-state | ⬜ |
| 8 | connector | connector-state | ⬜ |
| 9 | encoder | encoder-state | ⬜ |
| 10 | fb | framebuffer | ⬜ |
| 11 | property | drm-property | ⬜ |
| 12 | gem | gem-objects | ⬜ |
| 13 | dma-buf | dma-buf-overview | ⬜ |

### 3.2 Agent-B：GPU/图显方向

**负责模块：** KB-GPU-ARCH, KB-GFX-API

**信源清单：**
| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| Vulkan Specification | P0 | Vulkan API 全部 |
| DirectX 12 Documentation | P0 | DX12 API |
| NVIDIA/A MD/Intel 白皮书 | P1 | GPU 架构细节 |
| GPU Open | P2 | AMD 开源驱动文档 |
| Mesa3D source & docs | P2 | 开源图形栈实现 |

**当前子任务队列：**
（待 Agent-A 完成后启动，队列待规划）

### 3.3 Agent-C：AI 方向

**负责模块：** KB-AI-LLM, KB-AI-INFRA, KB-AI-AGENT

**信源清单：**
| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| arXiv / NeurIPS / ICML 论文 | P0 | 前沿研究 |
| HuggingFace / PyTorch / TensorFlow 官方文档 | P1 | 框架用法 |
| NVIDIA CUDA/cuDNN 文档 | P1 | 推理/训练基础设施 |
| OpenAI / Anthropic 技术报告 | P1 | 模型架构 |
| 技术博客与社区讨论 | P3 | 趋势与补充 |

**当前子任务队列：**
（待 Agent-A 完成后启动，队列待规划）

---

## 4. 每日执行日志

> 每日任务完成后自动追加记录

### 2026-04-11

| 方向 | Agent | 状态 | 新增条目 | 评审结果 | 备注 |
|------|-------|------|----------|----------|------|
| — | — | ⬜ 未执行 | — | — | 基础设施搭建日 |

---

## 5. 质量指标

### 5.1 累计统计

| 指标 | 当前值 | 目标 |
|------|--------|------|
| 总条目数 | 0 | — |
| 信源覆盖率（P0/P1） | — | 100% |
| 评审首次通过率 | — | > 80% |
| 交叉引用断链数 | — | 0 |
| 平均置信度 | — | ≥ 4 |

### 5.2 每周质量报告

每周日生成一次质量报告，包含：
- 本周新增/修改/过期条目统计
- 评审通过率趋势
- 信源分布（P0/P1/P2/P3 占比）
- 断链与冲突清单
- 下周计划

---

## 6. 版本历史

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-11 | v1.0 | 初始计划 |
| 2026-04-11 | v2.0 | 重构为持续运营计划：每日流水线 + 3子Agent + 评审机制 + 进度追踪 |
