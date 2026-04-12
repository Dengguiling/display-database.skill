# KB-LINUX-DRM — Linux DRM/KMS 子系统

## 概述

DRM (Direct Rendering Manager) 是 Linux 内核的图形子系统核心框架，KMS (Kernel Mode Setting) 负责显示模式管理。Atomic 模式是 KMS 的现代接口，支持原子性状态变更。

## 子模块列表

| 子模块 | 说明 | 状态 |
|--------|------|------|
| [atomic](atomic/) | Atomic 模式：状态管理、commit 流程、异步提交、VBlank、GEM、DMA-BUF | ✅ 已完成 (13条) |
| crtc | CRTC：显示控制器、VBlank、事件 | ⬜ 待建设 |
| plane | Plane：硬件叠加层、更新与禁用 | ⬜ 待建设 |
| connector | Connector：显示输出连接器 | ⬜ 待建设 |
| encoder | Encoder：信号编码器 | ⬜ 待建设 |
| fb | Framebuffer：帧缓冲管理 | ⬜ 待建设 |
| property | Property：原子属性系统 | ⬜ 待建设 |
| gem | GEM：图形执行管理器（显存管理） | ⬜ 待建设 |
| dma-buf | DMA-BUF：共享缓冲区 | ⬜ 待建设 |

## atomic 子模块条目索引

| # | 条目ID | 文件 | 核心主题 | 评审 |
|---|--------|------|----------|------|
| 1 | atomic-001 | [commit-flow](atomic/commit-flow.md) | Atomic Commit 完整流程（check→commit→swap→cleanup） | ✅ 96分 |
| 2 | atomic-002 | [state-mgmt](atomic/state-mgmt.md) | Atomic State 生命周期管理 | ✅ PASS |
| 3 | atomic-003 | [helpers](atomic/helpers.md) | Atomic Helper 函数集（校验/提交/翻转/挂起恢复） | ✅ PASS |
| 4 | atomic-004 | [nonblocking](atomic/nonblocking.md) | 非阻塞提交机制（commit_work/fence/async flip） | ✅ PASS |
| 5 | atomic-005 | [vblank](atomic/vblank.md) | VBlank 事件机制（中断/计数/等待/fake vblank） | ✅ PASS |
| 6 | atomic-006 | [plane-state](atomic/plane-state.md) | Plane 状态管理（zpos/格式/裁剪/缩放） | ✅ PASS |
| 7 | atomic-007 | [connector-state](atomic/connector-state.md) | Connector 状态管理（DPMS/音视频/广播RGB） | ✅ PASS |
| 8 | atomic-008 | [crtc-state](atomic/crtc-state.md) | CRTC 状态管理（enable/active/mode/VRR/颜色管理） | ✅ 95分 |
| 9 | atomic-009 | [encoder-state](atomic/encoder-state.md) | Encoder 与 Bridge 状态管理 | ✅ PASS |
| 10 | atomic-010 | [framebuffer](atomic/framebuffer.md) | Framebuffer 管理与 Modifier 机制 | ✅ PASS |
| 11 | atomic-011 | [drm-property](atomic/drm-property.md) | DRM 属性系统（原子属性/枚举/Blob） | ✅ PASS |
| 12 | atomic-012 | [gem-objects](atomic/gem-objects.md) | GEM Buffer Object（创建/映射/PRIME/TTM） | ✅ 96分 |
| 13 | atomic-013 | [dma-buf-overview](atomic/dma-buf-overview.md) | DMA-BUF 跨设备缓冲区共享（三大原语/显隐式同步） | ✅ 97分 |

## 快速定位

| 你可能的问题 | 定位 |
|-------------|------|
| DRM atomic commit 的完整流程是什么？ | [→ atomic/commit-flow] |
| atomic_check 和 atomic_commit 的区别？ | [→ atomic/commit-flow] |
| 非阻塞提交和同步提交有什么不同？ | [→ atomic/commit-flow] |
| hw_done / flip_done / cleanup_done 三个信号的含义？ | [→ atomic/commit-flow] |
| drm_atomic_state 的生命周期是什么？ | [→ atomic/state-mgmt] |
| swap_state 在 commit 流程中的位置和作用？ | [→ atomic/state-mgmt] |
| drm_atomic_get_crtc_state 什么时候分配新状态？ | [→ atomic/state-mgmt] |
| state_to_destroy 的 swap 前后语义变化？ | [→ atomic/state-mgmt] |
| drm_atomic_helper_commit_tail 的标准执行顺序？ | [→ atomic/helpers] |
| 驱动如何自定义 commit 流程？ | [→ atomic/helpers] |
| 非阻塞提交时 commit_tail 在哪个上下文执行？ | [→ atomic/nonblocking] |
| DRM_MODE_ATOMIC_NONBLOCK 标志的作用？ | [→ atomic/nonblocking] |
| VBlank 事件如何发送和等待？ | [→ atomic/vblank] |
| drm_crtc_vblank_get/put 的引用计数语义？ | [→ atomic/vblank] |
| Plane 的 zpos 如何决定合成顺序？ | [→ atomic/plane-state] |
| Plane 的 src/dst 坐标系和裁剪规则？ | [→ atomic/plane-state] |
| CRTC enable 和 active 的区别？ | [→ atomic/crtc-state] |
| drm_atomic_crtc_needs_modeset() 检查哪些标志？ | [→ atomic/crtc-state] |
| CRTC 颜色管理管线（degamma→CTM→gamma）？ | [→ atomic/crtc-state] |
| GEM 对象的三种引用方式？ | [→ atomic/gem-objects] |
| PRIME 导出/导入的完整流程？ | [→ atomic/gem-objects] |
| GEM 和 TTM 的关系？ | [→ atomic/gem-objects] |
| DMA-BUF 三大原语（dma-buf/dma-fence/dma-resv）？ | [→ atomic/dma-buf-overview] |
| 隐式同步 vs 显式同步？ | [→ atomic/dma-buf-overview] |
| DRM Sync Object 的使用方式？ | [→ atomic/dma-buf-overview] |
| DRM 属性系统的核心数据结构？ | [→ atomic/drm-property] |
| Framebuffer Modifier 的作用？ | [→ atomic/framebuffer] |
| Connector 状态中的 DPMS 管理？ | [→ atomic/connector-state] |
| Encoder 和 Bridge 的关系？ | [→ atomic/encoder-state] |
