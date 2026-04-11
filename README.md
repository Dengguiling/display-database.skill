# Display Database — 藏经阁

> glm-扫地僧 守阁之所。经过验证、去噪、提纯后的结构化技术知识库。
> 不是文档库，不是教程库，是给 AI 用的知识索引。

## 定位

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

## 目录结构

```
display-database.skill/
├── SKILL.md              # Skill 入口（触发条件 + 使用方式）
├── SPEC.md               # 执行规范（信源/模板/评审/变更管理）
├── PLAN.md               # 持续运营计划（任务队列/Agent调度/质量指标）
├── _PROGRESS.md          # 每日进度追踪（断点恢复）
│
├── KB-LINUX-DRM/         # Linux DRM/KMS
│   ├── _INDEX.md         # 模块索引
│   ├── _GLOSSARY.md      # 术语表
│   ├── _REVIEW-LOG.md    # 评审日志
│   ├── atomic/           #   子模块
│   ├── crtc/
│   ├── plane/
│   ├── connector/
│   ├── encoder/
│   ├── fb/
│   ├── property/
│   ├── gem/
│   └── dma-buf/
│
├── KB-LINUX-GPU-DRIVER/  # Linux GPU 驱动
├── KB-GPU-ARCH/          # GPU 架构
├── KB-GFX-API/           # 图形 API
├── KB-ANDROID-GFX/       # Android 图形栈
│   ├── surfaceflinger/
│   ├── hwcomposer/
│   ├── gralloc/
│   ├── vulkan-android/
│   └── display-hal/
│
└── KB-AI-GFX/            # AI 图显技术
    ├── super-resolution/ #   DLSS/FSR/XeSS/AISR
    ├── frame-gen/        #   帧生成
    ├── denoising/        #   AI 降噪
    ├── neural-rendering/ #   神经渲染
    └── ai-compilation/   #   AI 编译优化
```

## 核心规范

### 信源优先级

| 等级 | 类型 | 置信度上限 |
|------|------|------------|
| P0 | 内核源码 / 官方规范 | 5 |
| P1 | 官方文档 / 白皮书 | 5 |
| P2 | 权威社区（LKML/LWN） | 4 |
| P3 | 技术博客（仅补充） | 3 |

### 评审机制（两阶段）

**Phase 1 — 格式预检：** 8 项 checklist（信源/版本/引用/术语/密度/模板）

**Phase 2 — 对抗式实测：**
- 提问 Agent 基于新条目出 3-5 题
- 回答 Agent **仅凭知识库**作答，禁止网络搜索
- 评分：准确性 40% + 完整性 30% + 可检索性 20% + 关联性 10%
- ≥ 85 分 PASS / 70-84 条件PASS / < 70 FAIL 打回重写

### 每日自动采集

```
03:00  Agent-A  Linux 驱动（DRM/KMS/GPU驱动）
03:10  Agent-C  Android 图显（SurfaceFlinger/HWComposer/Gralloc）
03:15  Agent-B  GPU/图显（架构/Vulkan/DX12）
03:20  Agent-D  AI 图显（DLSS/FSR/AISR/帧生成）
```

## 维护者

- **glm-扫地僧** — 知识采集、提纯、评审、入库
- **Ling** — 方向指导

## 文档

- [SPEC.md](SPEC.md) — 执行规范完整版
- [PLAN.md](PLAN.md) — 持续运营计划
- [SKILL.md](SKILL.md) — Skill 入口与触发规则
