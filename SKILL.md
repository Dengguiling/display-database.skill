---
name: display-database
version: 1.0.0
description: >
  藏经阁 — 提纯后的技术知识库，覆盖 Linux DRM/KMS、GPU 驱动与架构、图形 API（Vulkan/DX12）、
  Android 图形栈（SurfaceFlinger/HWComposer/Gralloc）、AI（LLM/推理/Agent）。
  当用户询问上述领域的技术问题时触发，优先从知识库检索已验证的知识，而非网络搜索。
  也用于每日自动知识采集、评审与入库的持续运营。
---

# Display Database — 藏经阁

> 经过验证、去噪、提纯后的结构化技术知识库。
> 不是文档库，不是教程库，是给 AI 用的知识索引。

## 核心理念

网络搜索夹杂错误信息、过时内容、以讹传讹。藏经阁的价值是：**经过多源交叉验证的结构化知识**，替代不可靠的网络搜索。

## 知识领域

| 领域 | 模块 | 覆盖范围 |
|------|------|----------|
| Linux 驱动 | `KB-LINUX-DRM/` | DRM/KMS, Atomic, CRTC, Plane, Connector, GEM, DMA-BUF |
| GPU 驱动 | `KB-LINUX-GPU-DRIVER/` | 驱动模型, Modeset, 厂商实现 |
| GPU 架构 | `KB-GPU-ARCH/` | NVIDIA/AMD/Intel 架构, 显存, 计算单元 |
| 图形 API | `KB-GFX-API/` | Vulkan, DX12, OpenGL |
| Android 图显 | `KB-ANDROID-GFX/` | SurfaceFlinger, HWComposer, Gralloc, Display HAL |
| AI 图显 | `KB-AI-GFX/` | DLSS/FSR/XeSS/AISR, 帧生成, AI 降噪, 神经渲染, AI 编译 |

## 文件结构

```
display-database.skill/
├── SKILL.md              ← 你正在读的文件（skill 入口）
├── SPEC.md              ← 执行规范（信源/模板/评审/变更管理）
├── PLAN.md              ← 持续运营计划（任务队列/Agent调度/质量指标）
├── _PROGRESS.md         ← 每日进度追踪（断点恢复用）
│
├── KB-LINUX-DRM/        ← Linux DRM/KMS 知识
│   ├── _INDEX.md        ← 模块索引（导航节点）
│   ├── _GLOSSARY.md     ← 术语表
│   ├── _REVIEW-LOG.md   ← 评审日志
│   ├── atomic/          ← 子模块
│   │   ├── _INDEX.md
│   │   ├── commit-flow.md
│   │   └── ...
│   ├── crtc/
│   └── ...
│
├── KB-LINUX-GPU-DRIVER/
├── KB-GPU-ARCH/
├── KB-GFX-API/
├── KB-ANDROID-GFX/
├── KB-AI-LLM/
├── KB-AI-INFRA/
└── KB-AI-AGENT/
```

## 使用方式

### 场景一：回答技术问题（渐进式查询）

当 Ling 询问某个技术问题时：

1. **先检索知识库**：读取对应模块的 `_INDEX.md`，定位到相关条目
2. **返回概述**：只返回条目的 `概述` + `架构/调用链`，不倾倒全部内容
3. **按需深入**：Ling 追问时，再返回对应章节
4. **引用不复制**：涉及关联知识时，指向关联条目 `[→ 条目名]`
5. **库中没有时**：才去网络搜索，搜索后按规范提纯入库

### 场景二：每日自动知识采集（Cron 触发）

每日 03:00-03:20 自动触发 4 个子 Agent 并行采集：

```
03:00  Agent-A  Linux 驱动（DRM/KMS/GPU驱动）
03:10  Agent-C  Android 图显（SurfaceFlinger/HWComposer/Gralloc）
03:15  Agent-B  GPU/图显（架构/Vulkan/DX12）
03:20  Agent-D  AI（LLM/基础设施/Agent）
```

每个 Agent 的执行流程：
1. 读取 `SPEC.md`（规范）+ `PLAN.md`（任务队列）+ `_PROGRESS.md`（断点）
2. 从断点恢复，或从队列头开始
3. 逐条执行：**抓取 → 编写 → 评审（8项checklist）→ 入库或打回**
4. 每完成一条更新 `_PROGRESS.md`
5. 上下文接近极限时保存断点退出，下次自动接力

### 场景三：知识库维护

- **新增条目**：按 SPEC.md 模板编写 → 8项评审 → PASS 入库
- **修改条目**：说明修改原因 → 重新评审 → 更新验证日期
- **删除条目**：禁止单方面删除，标 `[已过期]` 保留追溯
- **定期复评**：每周检查上游 breaking change，更新过期条目

## 关键规范速查

### 信源优先级

| 等级 | 类型 | 置信度上限 |
|------|------|------------|
| P0 | 内核源码/官方规范源码 | 5 |
| P1 | 官方文档/白皮书 | 5 |
| P2 | 权威社区（LKML/LWN） | 4 |
| P3 | 技术博客（仅补充） | 3 |

### 评审 Checklist（Phase 1 格式预检）

1. 信源可追溯（每条 Sources 有具体路径/URL）
2. 信源等级合理（P3 置信度 ≤ 3）
3. 版本标注完整（@since/@changed/@deprecated）
4. 交叉引用有效（关联条目存在）
5. 内容无冲突（与已有条目一致）
6. 术语一致性（与 _GLOSSARY.md 一致）
7. 信息密度合格（无空话/废话）
8. 模板完整（所有必填章节不空）

### 对抗式实测（Phase 2 核心评审）

Phase 1 通过后，**提问 Agent** 基于新条目出题，**回答 Agent** 仅凭知识库作答：
- 准确性 40% + 完整性 30% + 可检索性 20% + 关联性 10%
- ≥ 85 分 PASS / 70-84 条件PASS / < 70 FAIL 打回重写

### Git 提交规范

```
{类型}: {模块}/{子模块} {简述}
类型: add / update / review / expire / refactor
```

## 详细文档

- **执行规范**：读取 `SPEC.md` — 信源/模板/评审/变更管理的完整规则
- **运营计划**：读取 `PLAN.md` — 任务队列/Agent调度/质量指标/执行日志
- **当前进度**：读取 `_PROGRESS.md` — 今日各 Agent 的执行状态与断点

## 版本历史

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-11 | v1.0.0 | 初始版本：从 self-knowledge-database 迁移，重构为标准 Skill |
