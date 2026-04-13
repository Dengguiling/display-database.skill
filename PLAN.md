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
│  生成 4 个方向的子任务               │
└──────────┬──────────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼     ▼
  ┌─────┐┌─────┐┌─────┐┌─────┐
  │ AI  ││ GPU ││Linux││Andr │  ← 4 个子 Agent 并行
  │图显  ││图显  ││驱动  ││图显  │
  └──┬──┘└──┬──┘└──┬──┘└──┬──┘
     │     │     │     │
     ▼     ▼     ▼     ▼
  抓取数据（P0/P1 优先）
     │
     ▼
  编写知识条目（遵循 SPEC.md 模板）
     │
     ▼
  ┌──────────────────────────────────┐
  │  对抗式评审（提问 Agent vs 回答 Agent）│
  │  提问 Agent 基于新入库内容出题        │
  │  回答 Agent 仅凭 skill 知识库作答     │
  │  评分：优秀 → 入库 / 不达标 → 打回    │
  └──────────┬───────────┬────────────┘
             │           │
          优秀        不达标
             │           │
             ▼           ▼
          入库        打回修改
          commit      → 重新编写
          push        → 重新评审
             │           │
             ▼           ▼
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

### 1.3 对抗式评审机制

评审不是自审 checklist，而是**实测验证**：知识库写得好不好，让另一个 Agent 试着用一下就知道了。

#### 评审流程

```
知识条目编写完成
    │
    ▼
┌─────────────────────────────────────────────┐
│  Phase 1: 格式预检（自动，快速）              │
│  检查：信源是否完整 / 模板是否齐全 / 版本标注  │
│  不通过 → 直接打回，不进入 Phase 2            │
└──────────┬──────────────────────────────────┘
           │ PASS
           ▼
┌─────────────────────────────────────────────┐
│  Phase 2: 对抗式实测（核心评审）              │
│                                              │
│  提问 Agent（评审者）：                        │
│    - 读取刚入库的条目内容                      │
│    - 基于条目内容设计 3-5 个问题               │
│    - 问题覆盖：概述理解 / 关键接口 / 实现细节  │
│    - 问题难度：基础 → 进阶 → 边界情况          │
│                                              │
│  回答 Agent（被评审者）：                      │
│    - 仅能访问 skill 知识库文件                │
│    - 不能使用网络搜索                          │
│    - 模拟真实使用场景：Ling 提问 → 检索知识库  │
│    - 按渐进式查询规范回答                      │
│                                              │
│  提问 Agent 评分：                            │
│    - 准确性（40%）：回答是否与条目内容一致      │
│    - 完整性（30%）：是否覆盖了条目的关键信息    │
│    - 可检索性（20%）：能否通过索引快速定位      │
│    - 关联性（10%）：交叉引用是否有效引导深入    │
│                                              │
│  评分标准：                                   │
│    ≥ 85分 → 优秀 → PASS，入库                 │
│    70-84分 → 良好 → 有条件 PASS，记录改进建议  │
│    < 70分 → 不达标 → FAIL，打回重写           │
└──────────┬──────────────────────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
   PASS  条件PASS  FAIL
     │     │       │
     ▼     ▼       ▼
   入库  入库+    打回
        改进建议  重新编写
                  最多3次
                  仍不通过→[需人工介入]
```

#### 评审打分卡模板

```markdown
## {条目ID} 对抗式评审

### 提问 Agent 出题
| # | 问题 | 难度 | 考察维度 |
|---|------|------|----------|
| 1 | {问题} | 基础 | 概述理解 |
| 2 | {问题} | 进阶 | 关键接口 |
| 3 | {问题} | 进阶 | 实现细节 |
| 4 | {问题} | 边界 | 陷阱/版本差异 |
| 5 | {问题} | 综合 | 关联知识 |

### 回答 Agent 作答
| # | 回答摘要 | 能否正确定位条目 | 引用是否准确 |
|---|----------|------------------|--------------|
| 1 | {摘要} | ✅/❌ | ✅/❌ |
| ... | ... | ... | ... |

### 评分
| 维度 | 权重 | 得分 | 说明 |
|------|------|------|------|
| 准确性 | 40% | {分} | {说明} |
| 完整性 | 30% | {分} | {说明} |
| 可检索性 | 20% | {分} | {说明} |
| 关联性 | 10% | {分} | {说明} |
| **总分** | 100% | **{分}** | |

### 结论
**{PASS / 条件PASS / FAIL}**
{改进建议（如有）}
```

---

## 2. 知识方向与优先级

### 2.1 四大方向

| 方向 | 模块 | 子模块 | 当前状态 |
|------|------|--------|----------|
| **Linux 驱动** | KB-LINUX-DRM | atomic, crtc, plane, connector, encoder, fb, property, gem, dma-buf | 🟡 建设中 |
| **GPU/图显** | KB-LINUX-GPU-DRIVER, KB-GPU-ARCH, KB-GFX-API | driver-model, modeset, hw-specific, vulkan, dx12 | ⬜ 待启动 |
| **Android 图显** | KB-ANDROID-GFX | surfaceflinger, hwcomposer, gralloc, vulkan-android, display-hal | ⬜ 待启动 |
| **AI 图显** | KB-AI-GFX | super-resolution, frame-gen, denoising, neural-rendering, ai-compilation | ⬜ 待启动 |

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

**当前子任务队列（Phase 2 — 2026-04-14）：**
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | driver | drm-driver-model | ⬜ |
| 2 | driver | drm-device | ⬜ |
| 3 | kms-core | kms-init | ⬜ |
| 4 | connector | connector-detection | ⬜ |
| 5 | connector | connector-modes | ⬜ |
| 6 | encoder | bridge-framework | ⬜ |
| 7 | dma-buf | dma-fence | ⬜ |
| 8 | dma-buf | dma-resv | ⬜ |
| 9 | gem | gem-mmap | ⬜ |
| 10 | fb | damage-tracking | ⬜ |

**Phase 1 已完成（2026-04-11 ~ 2026-04-13）：**
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | atomic | commit-flow | ✅ |
| 2 | atomic | state-mgmt | ✅ |
| 3 | atomic | helpers | ✅ |
| 4 | atomic | nonblocking | ✅ |
| 5 | crtc | crtc-state | ✅ |
| 6 | crtc | vblank | ✅ |
| 7 | plane | plane-state | ✅ |
| 8 | connector | connector-state | ✅ |
| 9 | encoder | encoder-state | ✅ |
| 10 | fb | framebuffer | ✅ |
| 11 | property | drm-property | ✅ |
| 12 | gem | gem-objects | ✅ |
| 13 | dma-buf | dma-buf-overview | ✅ |

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
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | gpu-pipeline | gpu-rendering-pipeline | ✅ |
| 2 | gpu-pipeline | shader-core-arch | ⬜ |
| 3 | gpu-pipeline | gpu-memory-hierarchy | ⬜ |
| 4 | vulkan | vulkan-overview | ⬜ |
| 5 | vulkan | vulkan-instance-device | ⬜ |
| 6 | vulkan | vulkan-command-buffer | ⬜ |
| 7 | vulkan | vulkan-pipeline | ⬜ |
| 8 | vulkan | vulkan-memory | ⬜ |
| 9 | dx12 | dx12-overview | ⬜ |
| 10 | dx12 | dx12-command-queue | ⬜ |
| 11 | dx12 | dx12-pipeline | ⬜ |
| 12 | dx12 | dx12-resource-memory | ⬜ |

### 3.3 Agent-C：Android 图显方向

**负责模块：** KB-ANDROID-GFX

**信源清单：**
| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| Android Source (cs.android.com) | P0 | SurfaceFlinger, HWComposer, Gralloc 源码 |
| Android Developer Documentation | P0 | 官方 API 文档与架构指南 |
| AOSP `frameworks/native/` | P0 | 图形栈核心实现 |
| AOSP `hardware/interfaces/graphics/` | P1 | HAL 接口定义 |
| Android GPU Inspector (AGI) 文档 | P2 | GPU 调试与分析工具 |
| Chromium GPU 相关文档 | P2 | Android WebView 图形栈 |

**当前子任务队列：**
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | surfaceflinger | architecture-overview | ⬜ |
| 2 | surfaceflinger | bufferqueue-flow | ⬜ |
| 3 | surfaceflinger | vsync-model | ⬜ |
| 4 | hwcomposer | hwc2-interface | ⬜ |
| 5 | hwcomposer | composition-types | ⬜ |
| 6 | gralloc | gralloc-allocator | ⬜ |
| 7 | gralloc | buffer-usage | ⬜ |
| 8 | vulkan-android | vulkan-on-android | ⬜ |
| 9 | display-hal | display-hal-aidl | ⬜ |
| 10 | display-hal | display-color | ⬜ |

### 3.4 Agent-D：AI 图显方向

**负责模块：** KB-AI-GFX

**定位：** 聚焦 AI 技术在图形显示领域的应用，不涉及通用 AI（LLM/训练等）。

**信源清单：**
| 信源 | 等级 | 覆盖范围 |
|------|------|----------|
| NVIDIA DLSS/Frame Generation 白皮书 | P0 | 超分辨率、帧生成技术原理 |
| AMD FSR 技术文档 | P0 | FSR 开源实现与架构 |
| Intel XeSS 技术文档 | P0 | XeSS 超分辨率方案 |
| Vulkan/DX12 AI 扩展规范 | P0 | AI 图形 API 集成 |
| SIGGRAPH/EG/GDC 论文与演讲 | P1 | 神经渲染、AI 降噪前沿研究 |
| AISR / AI 显示增强相关论文 | P1 | AI Super Resolution for Display |
| 开源实现（FSR source, OpenRTX） | P2 | 实际代码参考 |

**当前子任务队列：**
| # | 子模块 | 条目 | 状态 |
|---|--------|------|------|
| 1 | super-resolution | overview | ⬜ |
| 2 | super-resolution | dlss | ⬜ |
| 3 | super-resolution | fsr | ⬜ |
| 4 | super-resolution | xess | ⬜ |
| 5 | super-resolution | aisr | ⬜ |
| 6 | frame-gen | dlss-frame-gen | ⬜ |
| 7 | frame-gen | afmf | ⬜ |
| 8 | denoising | rt-denoise | ⬜ |
| 9 | neural-rendering | neural-radiance | ⬜ |
| 10 | ai-compilation | shader-ai-compile | ⬜ |

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
| 2026-04-11 | v2.1 | 新增 Agent-C Android 图显方向（SurfaceFlinger/HWComposer/Gralloc/Display HAL） |
| 2026-04-11 | v3.0 | 迁移为标准 Skill（display-database.skill），适配 Skill 架构 |
| 2026-04-11 | v3.1 | AI 方向重定义为图显 AI（AISR/DLSS/FSR/帧生成）；评审机制升级为对抗式实测 |
