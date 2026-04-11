# KB-LINUX-DRM — Linux DRM/KMS 子系统

## 概述

DRM (Direct Rendering Manager) 是 Linux 内核的图形子系统核心框架，KMS (Kernel Mode Setting) 负责显示模式管理。Atomic 模式是 KMS 的现代接口，支持原子性状态变更。

## 子模块列表

| 子模块 | 说明 | 状态 |
|--------|------|------|
| [atomic](atomic/) | Atomic 模式：状态管理、commit 流程、异步提交 | 🟡 建设中 |
| crtc | CRTC：显示控制器、VBlank、事件 | ⬜ 待建设 |
| plane | Plane：硬件叠加层、更新与禁用 | ⬜ 待建设 |
| connector | Connector：显示输出连接器 | ⬜ 待建设 |
| encoder | Encoder：信号编码器 | ⬜ 待建设 |
| fb | Framebuffer：帧缓冲管理 | ⬜ 待建设 |
| property | Property：原子属性系统 | ⬜ 待建设 |
| gem | GEM：图形执行管理器（显存管理） | ⬜ 待建设 |
| dma-buf | DMA-BUF：共享缓冲区 | ⬜ 待建设 |

## 快速定位

| 你可能的问题 | 定位 |
|-------------|------|
| DRM atomic commit 的完整流程是什么？ | [→ atomic/commit-flow] |
| atomic_check 和 atomic_commit 的区别？ | [→ atomic/commit-flow] |
| 非阻塞提交和同步提交有什么不同？ | [→ atomic/commit-flow] |
| hw_done / flip_done / cleanup_done 三个信号的含义？ | [→ atomic/commit-flow] |
