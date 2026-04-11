# DRM Atomic 非阻塞提交机制

> **条目ID:** KB-LINUX-DRM-atomic-004
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档 + LWN 补丁分析）
> **时效性:** 2026-04-12 | **信源:** kernel.org, LWN.net, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [helpers](./helpers.md) | [state-mgmt](./state-mgmt.md)

---

## 核心摘要

DRM Atomic 非阻塞提交（Nonblocking Commit）是 DRM 子系统实现**零延迟页面翻转**的核心机制。当用户空间设置 `DRM_MODE_ATOMIC_NONBLOCK` 标志时，`atomic_commit()` 回调必须在调用者上下文中完成所有准备工作，然后将实际硬件提交卸载到工作线程（worker thread）执行。这套机制通过 `drm_crtc_commit` 追踪结构、`commit_work` 工作队列和 fence 同步原语，确保了多个并发提交之间的正确排序与互斥，是 Wayland compositor 实现流畅渲染循环的基础设施。

---

## 1. 非阻塞 vs 阻塞提交

| 维度 | 阻塞提交 (nonblock=false) | 非阻塞提交 (nonblock=true) |
|------|---------------------------|---------------------------|
| 调用上下文 | 同步等待，直到硬件完成翻转 | 立即返回，硬件提交异步执行 |
| 等待 fence | 在 `atomic_commit()` 中等待 | 在 worker thread 中等待 |
| 返回时机 | VBlank 事件已发送 | 仅完成准备工作即返回 |
| 适用场景 | 模式切换、全屏 modeset | 日常页面翻转（compositor 主循环） |
| 错误码 | -EINTR/-EAGAIN 可重启 | -EBUSY 表示有未完成的提交 |
| 并发限制 | 无（串行等待） | 当前无驱动支持提交队列，必须等前一次完成 |

---

## 2. 核心数据结构

### `struct drm_crtc_commit`
```c
struct drm_crtc_commit {
    struct kref ref;
    struct drm_crtc *crtc;
    struct completion flip_done;
    struct completion hw_done;
    struct completion cleanup_done;
    struct work_struct work;
    bool hw_done_sent;
    bool flip_done_sent;
};
```

**三阶段完成信号：**
- `flip_done` — 页面翻转完成（VBlank 事件已发送）
- `hw_done` — 硬件编程完成
- `cleanup_done` — 所有旧资源可安全释放

### `struct drm_atomic_state` 中的非阻塞字段
```c
struct drm_atomic_state {
    ...
    bool async_update : 1;       /* 异步更新标志 */
    struct work_struct commit_work;  /* 非阻塞提交的工作队列 */
    ...
};
```

---

## 3. 提交流程详解

### 3.1 阻塞路径（nonblock=false）

```
用户空间 ioctl
    │
    ▼
drm_atomic_helper_commit()
    │
    ├─ drm_atomic_helper_setup_commit()  ← 设置 commit 追踪
    │
    ├─ drm_atomic_helper_swap_state()    ← 交换 new/old state
    │
    ├─ commit_tail()                     ← 同步执行
    │   ├─ wait_for_dependencies()
    │   ├─ commit_modeset_disables()
    │   ├─ [驱动硬件编程]
    │   ├─ commit_modeset_enables()
    │   ├─ commit_planes()
    │   ├─ wait_for_vblanks()
    │   └─ cleanup_planes()
    │
    └─ 返回用户空间
```

### 3.2 非阻塞路径（nonblock=true）

```
用户空间 ioctl (DRM_MODE_ATOMIC_NONBLOCK)
    │
    ▼
drm_atomic_helper_commit()
    │
    ├─ drm_atomic_helper_setup_commit()  ← 设置 commit 追踪
    │
    ├─ drm_atomic_helper_swap_state()    ← 交换 new/old state
    │
    ├─ INIT_WORK(&state->commit_work, commit_tail)
    │
    ├─ queue_work(system_unbound_wq, &state->commit_work)
    │
    └─ 立即返回用户空间（不等待硬件完成）
    
    ──── 异步执行（worker thread）────
    
commit_tail()  [在 worker thread 中]
    │
    ├─ drm_atomic_helper_wait_for_dependencies()
    │   └─ 等待同一 CRTC 上所有未完成的 commit 完成
    │
    ├─ drm_atomic_helper_commit_modeset_disables()
    ├─ [驱动硬件编程]
    ├─ drm_atomic_helper_commit_modeset_enables()
    ├─ drm_atomic_helper_commit_planes()
    ├─ drm_atomic_helper_wait_for_vblanks()
    ├─ drm_atomic_helper_commit_hw_done()  ← 通知 hw_done
    ├─ drm_atomic_helper_commit_cleanup_done()  ← 通知 cleanup_done
    └─ drm_atomic_state_put(state)  ← 释放引用
```

---

## 4. 关键函数详解

### `drm_atomic_helper_setup_commit()`
```c
int drm_atomic_helper_setup_commit(struct drm_atomic_state *state,
                                    bool nonblock);
```
**职责：** 为受影响的每个 CRTC 分配 `drm_crtc_commit` 追踪结构，设置三阶段完成信号。

**关键逻辑：**
- 如果 `nonblock=true`，检查是否有未完成的提交 → 返回 `-EBUSY`
- 为每个需要 modeset 的 CRTC 创建 `drm_crtc_commit` 对象
- 将 `flip_done`、`hw_done`、`cleanup_done` 初始化为未完成状态
- 如果 `nonblock=false`，直接使用 `state->fake_commit`（栈上分配，无需释放）

### `drm_atomic_helper_wait_for_dependencies()`
```c
void drm_atomic_helper_wait_for_dependencies(struct drm_atomic_state *state);
```
**职责：** 在 `commit_tail` 入口处等待同一 CRTC 上所有前序非阻塞提交完成。

**为什么需要：** 非阻塞提交可能被排入工作队列但尚未执行。新提交必须等待前序提交完成，否则会破坏硬件状态的一致性。

### `drm_atomic_helper_swap_state()`
```c
int drm_atomic_helper_swap_state(struct drm_atomic_state *state,
                                  bool stall);
```
**职责：** 将 `drm_atomic_state` 中的 new state 安装到各对象（CRTC/Plane/Connector）的 `.state` 指针上，同时将旧 state 保存回 `drm_atomic_state`。

**`stall` 参数：**
- `stall=true` — 等待所有正在运行的 commit 完成后再交换（用于驱动直接访问 `drm_plane.state` 的场景）
- `stall=false` — 不等待，直接交换（用于驱动使用 `drm_atomic_get_old/new_*_state()` 的场景）

### `drm_atomic_helper_commit_hw_done()`
```c
void drm_atomic_helper_commit_hw_done(struct drm_atomic_state *state);
```
**职责：** 通知 DRM core 硬件编程已完成。触发 `hw_done` completion，允许后续提交开始执行。

### `drm_atomic_helper_commit_cleanup_done()`
```c
void drm_atomic_helper_commit_cleanup_done(struct drm_atomic_state *state);
```
**职责：** 通知 DRM core 所有旧资源可安全释放。触发 `cleanup_done` completion。

---

## 5. Stall 机制

### 什么是 Stall？

当驱动在 `atomic_commit` 或 `commit_tail` 中**直接访问** `drm_plane.state`、`drm_crtc.state` 等对象指针时（而非通过 `drm_atomic_get_old/new_*_state()` 获取），就存在竞态风险：

```
线程A (commit_tail)          线程B (新的 atomic_commit)
    │                              │
    ├─ 读取 plane->state           ├─ swap_state() → 修改 plane->state
    ├─ 基于旧 state 编程硬件       ├─ 新 commit_tail 开始
    └─ 硬件状态不一致！            └─ 基于新 state 编程硬件
```

### Stall 的解决方案

**方案1：设置 `stall=true`（传统方式）**
```c
drm_atomic_helper_swap_state(state, true);  // stall=true
```
调用 `swap_state` 时会等待所有正在运行的 commit 完成，确保 `plane->state` 指针在交换时是稳定的。

**方案2：使用 old/new state 访问器（推荐方式）**
```c
// 不直接访问 plane->state，而是通过 atomic state 获取
struct drm_plane_state *old_plane_state =
    drm_atomic_get_old_plane_state(state, plane);
struct drm_plane_state *new_plane_state =
    drm_atomic_get_new_plane_state(state, plane);
```
这样 `swap_state` 可以使用 `stall=false`，避免不必要的等待。

### 驱动如何声明 Stall 需求

在 `drm_plane_helper_funcs` 中设置：
```c
static const struct drm_plane_helper_funcs my_plane_helper_funcs = {
    .prepare_fb = my_prepare_fb,
    .cleanup_fb = my_cleanup_fb,
    .atomic_check = my_plane_atomic_check,
    .atomic_update = my_plane_atomic_update,
    .atomic_disable = my_plane_atomic_disable,
};
```

如果 `atomic_update` / `atomic_disable` 直接访问 `plane->state`，则必须在 `commit_tail` 中使用 `stall=true`。内核文档明确建议驱动迁移到 old/new state 访问器模式。

---

## 6. Async Page Flip（v6.8+）

### 背景

传统非阻塞提交仍然等待下一个 VBlank 才执行翻转。`DRM_MODE_PAGE_FLIP_ASYNC` 允许**立即翻转**，即使错过了 VBlank，代价是可能产生画面撕裂（tearing）。

### v6.8 的关键变更

由 André Almeida 和 Simon Ser 提交的补丁系列（[PATCH v7 0/6]）将 async page-flip 引入 atomic API：

| 补丁 | 内容 |
|------|------|
| `drm: allow DRM_MODE_PAGE_FLIP_ASYNC for atomic commits` | 允许 atomic commit 使用 ASYNC 标志 |
| `drm: introduce DRM_CAP_ATOMIC_ASYNC_PAGE_FLIP` | 新增 capability 查询 |
| `drm: introduce drm_mode_config.atomic_async_page_flip_not_supported` | 驱动可声明不支持 |
| `drm: Refuse to async flip with atomic prop changes` | 仅允许 `fb_id` 变更，拒绝其他属性修改 |
| `drm/doc: Define KMS atomic state set` | 文档定义 atomic state set 语义 |
| `amd/display: indicate support for atomic async page-flips` | AMD DC 首个支持者 |

### Async Flip 的限制

```
✅ 允许：仅修改 fb_id（更换帧缓冲）
❌ 禁止：修改任何其他属性（mode_id, crtc_x/y, crtc_w/h, rotation 等）
```

**设计理由：** 异步翻转的快速路径与 modeset 变更不兼容。modeset 变更需要完整的 disable→reconfigure→enable 流程，无法在"立即翻转"的语义下安全执行。

### 驱动支持状态

| 驱动 | 支持 Async Atomic Flip | 备注 |
|------|----------------------|------|
| amdgpu (DC) | ✅ v6.8+ | 首个支持者 |
| i915 | ✅ v6.8+ | |
| nouveau | ✅ v6.8+ | |
| atmel-hlcdc | ✅ v6.8+ | |
| 其他 | ❌ | 通过 `atomic_async_page_flip_not_supported` 声明 |

---

## 7. 错误处理

| 错误码 | 含义 | 处理方式 |
|--------|------|----------|
| `-EBUSY` | 有未完成的非阻塞提交 | 用户空间应等待后重试 |
| `-ENOMEM` | 内存分配失败（如 pin framebuffer） | 用户空间应释放资源后重试 |
| `-ENOSPC` | VRAM/地址空间不足 | 用户空间应减少缓冲区使用 |
| `-EIO` | 硬件完全失效 | 严重错误，需重置 |
| `-EINTR` | 被信号中断 | 用户空间应重启 ioctl |
| `-EAGAIN` | 需要重试 | 用户空间应重启 ioctl |

**关键规则：** 驱动**不允许**返回 `-EINVAL`（应在 `atomic_check` 中捕获）或 `-EDEADLK`（`atomic_commit` 中不允许获取额外的 modeset lock）。

---

## 8. 用户空间使用示例

### 标准 VSync 同步翻转（非阻塞）
```c
struct drm_mode_atomic atomic = {0};
atomic.flags = DRM_MODE_ATOMIC_NONBLOCK;
atomic.flags |= DRM_MODE_PAGE_FLIP_EVENT;  // 请求 VBlank 事件通知

// 设置 plane 的 fb_id 属性
drm_object_set_property(&plane->base, props.fb_id, fb_id);

ret = drmIoctl(fd, DRM_IOCTL_MODE_ATOMIC, &atomic);
if (ret == -EBUSY) {
    // 有未完成的提交，等待后重试
}
```

### Async 立即翻转（v6.8+）
```c
struct drm_mode_atomic atomic = {0};
atomic.flags = DRM_MODE_ATOMIC_NONBLOCK;
atomic.flags |= DRM_MODE_PAGE_FLIP_EVENT;
atomic.flags |= DRM_MODE_PAGE_FLIP_ASYNC;  // 立即翻转，不等待 VBlank

// 仅修改 fb_id
drm_object_set_property(&plane->base, props.fb_id, new_fb_id);

ret = drmIoctl(fd, DRM_IOCTL_MODE_ATOMIC, &atomic);
```

---

## 9. Compositor 实践要点

### Weston/Wayland 中的典型模式

```
渲染线程                          DRM 提交线程
    │                                  │
    ├─ 渲染帧 N                        │
    ├─ 提交 atomic (nonblock) ────────→├─ queue_work()
    ├─ 立即返回                         ├─ commit_tail()
    ├─ 渲染帧 N+1                      │   ├─ wait_for_dependencies()
    ├─ 提交 atomic (nonblock) ──EBUSY→ │   ├─ 硬件编程
    ├─ 等待 VBlank 事件                 │   ├─ wait_for_vblanks()
    ├─ 收到帧 N 的 VBlank 事件          │   └─ 发送 VBlank 事件
    ├─ 提交帧 N+1 ────────────────────→├─ 新的 commit_tail()
    └─ ...                             └─ ...
```

### 性能考量

- **提交频率：** 非阻塞允许 compositor 以渲染帧率提交，不受 VBlank 限制
- **延迟：** 从提交到显示的延迟 = worker 调度延迟 + 硬件编程时间 + VBlank 等待
- **背压：** `-EBUSY` 是自然的背压机制，防止提交堆积
- **fence 集成：** `IN_FENCE` 属性允许 compositor 将 GPU 渲染完成 fence 传递给 DRM，确保渲染完成后才翻转

---

## 潜在影响

- **游戏场景：** Async page-flip（v6.8+）为 Linux 游戏提供了与 Windows DXGI_FLIP_PRESENT_ALLOW_TEARING 等价的能力，对 Gamescope 等游戏合成器至关重要。
- **VR/AR：** 低延迟翻转是 VR 渲染的基础，非阻塞提交确保了渲染循环不被 VBlank 阻塞。
- **驱动迁移：** 社区持续推动驱动从 `stall=true` 迁移到 old/new state 访问器模式，以消除不必要的同步开销。
- **未来方向：** 当前无驱动支持非阻塞提交队列（即多个并发提交排队），这是潜在的优化方向。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)（atomic_commit 文档、swap_state 语义）
- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm: Add support for atomic async page-flip — LWN.net](https://lwn.net/Articles/948020)
- [Linux 6.8 Atomic Async Page Flips — Phoronix](https://www.phoronix.com/news/Linux-6.8-Atomic-Async-Flips)
- [drm_atomic_helper.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_atomic_helper.c)
