# Atomic Commit Flow

> DRM Atomic 模式下将状态变更提交到硬件的完整流程，涵盖 check → commit → commit_tail → hw_done → flip_done → cleanup_done 六个阶段。

## Meta
- **ID:** KB-LINUX-DRM-atomic-001
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-11
- **适用版本:** Linux v4.14+（atomic helpers 稳定化），v6.6+ 推荐使用 async commit helpers
- **关键词:** atomic commit, drm_atomic_commit, commit_tail, hw_done, flip_done, cleanup_done, nonblocking
- **前置知识:** [→ KB-LINUX-DRM-crtc-001] [→ KB-LINUX-DRM-plane-001]
- **关联条目:** [→ KB-LINUX-DRM-atomic-002] [→ KB-LINUX-DRM-property-001]

## 概述

Atomic Commit 是 DRM Atomic 模式的核心操作：将用户空间通过 `DRM_IOCTL_MODE_ATOMIC` 提交的状态变更（plane/crtc/connector 的属性修改）经过校验后应用到硬件。整个流程分为**同步校验阶段**（atomic_check）和**异步提交阶段**（atomic_commit），后者通过三组 completion 信号（hw_done / flip_done / cleanup_done）协调硬件更新与缓冲区生命周期管理。

## 架构/调用链

```
用户空间: DRM_IOCTL_MODE_ATOMIC
    │
    ▼
DRM Core: drm_mode_atomic_ioctl()
    │
    ├─ drm_atomic_check()                    ← 同步阶段
    │   ├─ drm_atomic_state_check()
    │   │   ├─ drm_atomic_check_planes()
    │   │   ├─ drm_atomic_check_crtcs()
    │   │   ├─ drm_atomic_check_connectors()
    │   │   └─ .atomic_check()                ← driver hook
    │   └─ return 0 / -EINVAL
    │
    ▼
DRM Core: drm_atomic_commit(state)           ← 异步阶段入口
    │
    ├─ nonblock == false (同步提交)
    │   └─ .atomic_commit(state, false)
    │       └─ drm_atomic_helper_commit()
    │           ├─ drm_atomic_helper_commit_modeset_enables()
    │           ├─ drm_atomic_helper_commit_planes()
    │           │   ├─ .atomic_update()       ← fast path (无 modeset)
    │           │   └─ .atomic_disable/.enable ← modeset path
    │           ├─ drm_atomic_helper_commit_hw_done()   ← 信号 1
    │           ├─ drm_atomic_helper_wait_for_vblanks()
    │           ├─ drm_atomic_helper_cleanup_planes()   ← 释放旧 fb
    │           └─ drm_atomic_helper_commit_cleanup_done() ← 信号 3
    │
    └─ nonblock == true (非阻塞提交)
        └─ .atomic_commit(state, true)
            └─ drm_atomic_helper_commit()
                ├─ drm_atomic_helper_setup_commit()     ← 初始化 completion
                ├─ queue commit_work
                │   └─ [worker thread]
                │       └─ .atomic_commit_tail(state)   ← driver hook
                │           ├─ drm_atomic_helper_wait_for_dependencies()
                │           ├─ drm_atomic_helper_commit_modeset_enables()
                │           ├─ drm_atomic_helper_commit_planes()
                │           ├─ drm_atomic_helper_commit_hw_done()   ← 信号 1
                │           ├─ [wait for hw_done]
                │           ├─ drm_atomic_helper_wait_for_vblanks()
                │           ├─ drm_atomic_helper_cleanup_planes()
                │           └─ drm_atomic_helper_commit_cleanup_done() ← 信号 3
                │
                └─ return immediately (不阻塞调用者)
```

### 三组 Completion 信号

```
hw_done ──────► 硬件寄存器写入完成
                  │
                  ▼
flip_done ─────► VBlank 到来，新缓冲区已扫描输出
                  │  (同时发送 drm_event 给用户空间)
                  ▼
cleanup_done ──► 旧缓冲区已释放，可回收
```

## 关键接口

### 函数

- `drm_atomic_commit(state)` — 提交 atomic 状态变更到硬件
  - 参数: `struct drm_atomic_state *state`
  - 返回: 0 成功 / -EBUSY 有 pending commit / -ENOMEM / -EIO 硬件故障
  - 源码: `drivers/gpu/drm/drm_atomic.c`
  - 说明: 这是 `drm_mode_config_funcs.atomic_commit` 的 driver hook 入口

- `drm_atomic_helper_commit(state, nonblock)` — atomic helper 提供的默认 commit 实现
  - 参数: `struct drm_atomic_state *state`, `bool nonblock`
  - 返回: 0 成功 / 负错误码
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: 同步提交时直接在调用上下文执行；非阻塞提交时将工作推入 worker thread

- `drm_atomic_helper_commit_tail(state)` — 非阻塞提交的实际执行函数
  - 参数: `struct drm_atomic_state *state`
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: 在 worker thread 中执行，driver 可通过 `atomic_commit_tail` hook 自定义

- `drm_atomic_helper_commit_hw_done(state)` — 信号 hw_done completion
  - 参数: `struct drm_atomic_state *state`
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: 表示所有硬件寄存器变更已写入，可以开始等待 VBlank

- `drm_atomic_helper_wait_for_vblanks(state)` — 等待所有受影响 CRTC 的 VBlank
  - 参数: `struct drm_atomic_state *state`
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: flip_done 信号由 `drm_crtc_send_vblank_event()` 隐式触发

- `drm_atomic_helper_cleanup_planes(state)` — 释放旧 framebuffer
  - 参数: `struct drm_atomic_state *state`
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: 调用 `drm_framebuffer_put()` 释放不再使用的旧 fb

- `drm_atomic_helper_commit_cleanup_done(state)` — 信号 cleanup_done completion
  - 参数: `struct drm_atomic_state *state`
  - 源码: `drivers/gpu/drm/drm_atomic_helper.c`
  - 说明: 用于节流，防止硬件更新跑在缓冲区清理前面太多

### 结构体

- `struct drm_atomic_state` — 描述一次 atomic 更新的完整状态
  - 源码: `include/drm/drm_atomic.h`
  - 关键字段:
    - `planes` — 所有受影响 plane 的新旧状态
    - `crtcs` — 所有受影响 CRTC 的新旧状态
    - `connectors` — 所有受影响 connector 的新旧状态
    - `acquire_ctx` — modeset lock 上下文
    - `commit_work` — 非阻塞提交的 work_struct

- `struct drm_crtc_commit` — 跟踪每个 CRTC 的 pending commit
  - 源码: `include/drm/drm_crtc.h`
  - 关键字段:
    - `flip_done` — 硬件翻转到新缓冲区时信号
    - `hw_done` — 硬件寄存器写入完成时信号
    - `cleanup_done` — 旧缓冲区清理完成时信号
    - `commit_entry` — per-CRTC pending commit 链表节点

## 关键数据流

```
旧状态 (old_state)          新状态 (new_state)
     │                           │
     │    ┌──────────────┐       │
     └───►│  atomic_check │◄──────┘
          │  校验新状态    │
          └──────┬───────┘
                 │ PASS
                 ▼
          ┌──────────────┐
          │  atomic_commit│
          │  提交到硬件    │
          └──────┬───────┘
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
  hw_done    flip_done   cleanup_done
  寄存器写完   VBlank到    旧fb释放
```

## 常见陷阱

1. **atomic_commit 中禁止返回 -EINVAL**：所有参数校验必须在 atomic_check 中完成，commit 阶段只处理硬件操作错误（-EIO/-ENOMEM/-EBUSY）
2. **非阻塞提交必须保证管道不关闭**：driver 不能自行关闭已通过 atomic commit 成功启用的显示管道，否则 compositor 会因 page flip 被拒绝而崩溃
3. **dma-buf 共享缓冲区需等待渲染完成**：commit 前必须等待新 framebuffer 的渲染完成（包括其他 driver 的渲染），否则会出现撕裂
4. **同步提交不能返回 -EBUSY**：当 nonblock=false 时，driver 必须等待前一次 commit 完成后再执行，不允许失败
5. **flip_done 信号时机**：对于大多数硬件，flip_done 在 hw_done 之后触发；driver 必须通过 `drm_crtc_send_vblank_event()` 隐式触发，不能手动 complete

## Sources

- [P0] `drivers/gpu/drm/drm_atomic_helper.c` — `drm_atomic_helper_commit()`, `drm_atomic_helper_commit_tail()`, `drm_atomic_helper_commit_hw_done()`, `drm_atomic_helper_wait_for_vblanks()`, `drm_atomic_helper_cleanup_planes()`
- [P0] `include/drm/drm_atomic.h` — `struct drm_atomic_state` 定义
- [P0] `include/drm/drm_crtc.h` — `struct drm_crtc_commit` 定义
- [P0] `drivers/gpu/drm/drm_atomic.c` — `drm_atomic_commit()` 实现
- [P1] https://docs.kernel.org/gpu/drm-kms.html — Kernel Mode Setting 文档，atomic_commit / atomic_commit_tail / drm_crtc_commit 章节
