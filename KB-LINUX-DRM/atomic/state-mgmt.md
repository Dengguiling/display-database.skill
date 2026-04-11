# Atomic State Management

> DRM Atomic 模式下状态对象（drm_atomic_state）的创建、获取、复制、交换与销毁的完整生命周期管理机制。

## Meta
- **ID:** KB-LINUX-DRM-atomic-002
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-12
- **适用版本:** Linux v4.14+（atomic helpers 稳定化），v6.6+ 推荐 `drm_private_state` 替代子类化 `drm_atomic_state`
- **关键词:** drm_atomic_state, atomic_duplicate_state, swap_state, state_to_destroy, drm_atomic_get_crtc_state, drm_atomic_get_plane_state, drm_atomic_get_connector_state, acquire_ctx
- **前置知识:** [→ KB-LINUX-DRM-atomic-001]
- **关联条目:** [→ KB-LINUX-DRM-crtc-001] [→ KB-LINUX-DRM-plane-001] [→ KB-LINUX-DRM-connector-001]

## 概述

`struct drm_atomic_state` 是 DRM Atomic 模式的核心数据容器，代表一次从旧状态到新状态的原子性状态转换。它收集所有受影响的显示对象（CRTC、Plane、Connector、Private Object）的旧状态和新状态。整个生命周期分为：**分配（alloc）→ 填充（get）→ 校验（check）→ 交换（swap）→ 销毁（destroy）** 五个阶段。关键设计约束：所有状态变更都是"相对的"（relative），即基于当前运行状态做增量修改，而非从零构建。

## 架构/调用链

```
用户空间: DRM_IOCTL_MODE_ATOMIC
    │
    ▼
DRM Core: drm_mode_atomic_ioctl()
    │
    ├─ drm_atomic_state_alloc()              ← 分配空壳
    │   ├─ config->funcs->atomic_state_alloc()  ← driver 自定义（可选）
    │   └─ drm_atomic_state_init()               ← 默认实现
    │       ├─ kref_init(&state->ref)
    │       ├─ kcalloc(crtcs)                    ← 按 num_crtc 分配
    │       └─ kcalloc(planes)                   ← 按 num_total_plane 分配
    │
    ├─ drm_atomic_get_crtc_state()            ← 按需填充 CRTC 状态
    │   ├─ drm_atomic_get_existing_crtc_state()  ← 已存在则直接返回
    │   ├─ drm_modeset_lock(&crtc->mutex)        ← 获取锁
    │   ├─ crtc->funcs->atomic_duplicate_state() ← 复制当前状态
    │   ├─ state->crtcs[i].new_state = dup       ← 挂入 atomic_state
    │   └─ state->crtcs[i].old_state = crtc->state
    │
    ├─ drm_atomic_get_plane_state()           ← 按需填充 Plane 状态
    │   ├─ plane->funcs->atomic_duplicate_state()
    │   ├─ state->planes[i].new_state = dup
    │   └─ 若 plane->crtc 非空，自动 get 对应 CRTC state
    │
    ├─ drm_atomic_get_connector_state()       ← 按需填充 Connector 状态
    │   ├─ connector->funcs->atomic_duplicate_state()
    │   └─ 动态扩容 connectors 数组（krealloc_array）
    │
    ├─ .atomic_check()                        ← 校验阶段（只读）
    │   └─ state->checked = true              ← 标记为已校验，禁止再修改
    │
    ├─ .atomic_commit()                       ← 提交阶段
    │   └─ commit_tail (worker thread)
    │       ├─ drm_atomic_helper_prepare_planes()
    │       ├─ drm_atomic_helper_swap_state()  ← ★ 关键交换点
    │       │   ├─ [stall] 等待前一次 commit 的 hw_done
    │       │   ├─ connector->state = new_conn_state
    │       │   ├─ crtc->state = new_crtc_state
    │       │   ├─ plane->state = new_plane_state
    │       │   └─ state->xxx[i].state = old_xxx_state  ← 旧状态回收到 state
    │       ├─ 硬件寄存器写入
    │       ├─ drm_atomic_helper_commit_hw_done()
    │       ├─ drm_atomic_helper_wait_for_vblanks()
    │       └─ drm_atomic_helper_cleanup_planes()
    │
    └─ drm_atomic_state_put()                ← 引用计数归零时释放
        └─ .atomic_state_free() / drm_atomic_state_default_release()
```

## 关键数据结构

### struct drm_atomic_state

```c
struct drm_atomic_state {
    struct kref ref;                          // 引用计数
    struct drm_device *dev;                   // 所属 DRM 设备

    /* 标志位 */
    bool allow_modeset : 1;                   // 是否允许 modeset（用户空间控制）
    bool legacy_cursor_update : 1;            // [已废弃] 旧 cursor 语义
    bool async_update : 1;                    // 异步 plane 更新提示
    bool duplicated : 1;                      // 是否通过 duplicate_state 创建
    bool checked : 1;                         // 已通过 atomic_check，禁止修改
    bool plane_color_pipeline : 1;            // 使用 color pipeline 客户端

    /* 状态数组 */
    struct __drm_planes_state *planes;        // Plane 状态数组
    struct __drm_crtcs_state *crtcs;          // CRTC 状态数组
    struct __drm_connnectors_state *connectors; // Connector 状态数组（动态扩容）
    int num_connector;                        // connectors 数组当前大小
    struct __drm_private_objs_state *private_objs; // Driver 私有状态
    int num_private_objs;

    /* 提交上下文 */
    struct drm_modeset_acquire_ctx *acquire_ctx; // 锁获取上下文
    struct drm_crtc_commit *fake_commit;      // 未绑定 CRTC 的 plane/connector 用
    struct work_struct commit_work;           // 非阻塞提交的 work item
};
```

### 内部状态包装结构（__drm_xxx_state）

所有对象类型共享相同的包装模式：

```c
struct __drm_crtcs_state {
    struct drm_crtc *ptr;                     // 指向 CRTC 对象
    struct drm_crtc_state *state_to_destroy;  // swap 前指向 new_state，swap 后指向 old_state
    struct drm_crtc_state *old_state;         // 交换前的状态（当前运行状态）
    struct drm_crtc_state *new_state;         // 交换后的状态（即将应用的状态）
    struct drm_crtc_commit *commit;           // CRTC commit 引用
};
// __drm_planes_state、__drm_connnectors_state、__drm_private_objs_state 结构类似
```

**`state_to_destroy` 的语义：** 用于 `atomic_state_clear` 时确定需要释放哪个状态副本。swap_state 之前指向 `new_state`（因为 new_state 是刚分配的副本，需要释放），swap_state 之后指向 `old_state`（因为 old_state 已被替换，需要释放）。

## 关键接口

### 状态获取（按需分配 + 锁保护）

| 函数 | 作用 | 锁获取 |
|------|------|--------|
| `drm_atomic_get_crtc_state(state, crtc)` | 获取/分配 CRTC 状态 | `crtc->mutex` |
| `drm_atomic_get_plane_state(state, plane)` | 获取/分配 Plane 状态 | `plane->mutex` |
| `drm_atomic_get_connector_state(state, conn)` | 获取/分配 Connector 状态 | `connection_mutex` |
| `drm_atomic_get_private_obj_state(state, obj)` | 获取/分配私有状态 | `obj->lock` |

**共同行为模式：**
1. 检查 `state` 中是否已存在该对象的状态（`get_existing_xxx_state`），存在则直接返回
2. 获取对象对应的 modeset 锁（可能返回 `-EDEADLK`，需调用 `drm_modeset_backoff()` 重试）
3. 调用 `obj->funcs->atomic_duplicate_state()` 复制当前运行状态
4. 将副本挂入 `drm_atomic_state` 的对应数组，设置 `old_state` 和 `new_state`

**WARNING：** `drm_atomic_get_crtc_state()` 要求 `allow_modeset == true` 或为 driver 内部 commit，否则不允许向 state 添加无关 CRTC。

### 状态交换（swap_state — 软件状态切换点）

```
drm_atomic_helper_swap_state(state, stall)
    │
    ├─ [stall=true] 等待所有受影响 CRTC/Connector/Plane 的前一次 commit hw_done
    │
    ├─ for_each_oldnew_connector_in_state:
    │   connector->state = new_conn_state    ← 软件状态切换到新值
    │   state->connectors[i].state = old_conn_state  ← 旧状态回收到 state
    │
    ├─ for_each_oldnew_crtc_in_state:
    │   crtc->state = new_crtc_state
    │   state->crtcs[i].state = old_crtc_state
    │   将 new commit 加入 crtc->commit_list
    │
    └─ for_each_oldnew_plane_in_state:
        plane->state = new_plane_state
        state->planes[i].state = old_plane_state
```

**swap_state 的位置至关重要：** 它在硬件寄存器写入之前执行，确保后续的 `prepare_planes` / `commit_planes` 回调通过 `plane->state` / `crtc->state` 访问到的是新状态，而 `drm_atomic_helper_cleanup_planes()` 通过 `state` 中的旧状态进行清理。

### 状态复制（duplicate_state — 保存/恢复用）

`drm_atomic_helper_duplicate_state(dev, ctx)` 遍历所有 CRTC、Plane、Connector，调用各自的 `get_xxx_state()` 获取完整副本。设置 `state->duplicated = true`。用于 suspend/resume 场景。

### 状态销毁

| 函数 | 作用 |
|------|------|
| `drm_atomic_state_put(state)` | 引用计数 -1，归零时调用 `atomic_state_free` |
| `drm_atomic_state_default_clear(state)` | 遍历所有对象，调用 `atomic_destroy_state` 释放副本 |
| `drm_atomic_state_default_release(state)` | 释放 crtcs/planes/connectors/private_objs 数组本身 |

## 状态生命周期时序

```
alloc ──→ get ──→ check ──→ swap ──→ hw_commit ──→ cleanup ──→ destroy
  │         │        │         │          │            │           │
  │         │        │         │          │            │           └─ kfree(state)
  │         │        │         │          │            └─ 释放旧 fb
  │         │        │         │          └─ 寄存器写入
  │         │        │         └─ obj->state = new, state->xxx = old
  │         │        └─ state->checked = true
  │         └─ atomic_duplicate_state() × N
  └─ kcalloc(crtcs) + kcalloc(planes)
```

## 常见陷阱

1. **`checked` 标志后禁止修改状态：** `atomic_check` 通过后 `state->checked = true`，此后任何对 state 的修改都会触发 WARN。所有属性设置必须在 check 之前完成。
2. **`allow_modeset` 是用户空间控制的：** Driver 不得修改此标志。在 atomic commit 路径中不得查询此标志，应使用 `drm_atomic_crtc_needs_modeset()` 判断。
3. **Connector 数组动态扩容：** 与 CRTC/Plane 不同，Connector 数组在 `get_connector_state` 时通过 `krealloc_array` 动态扩容，`num_connector` 记录当前大小。
4. **`-EDEADLK` 必须正确处理：** 所有 `get_xxx_state` 调用都可能因 w/w mutex 死锁返回 `-EDEADLK`，调用者必须释放所有锁并调用 `drm_modeset_backoff()` 后重试整个 atomic 序列。
5. **`state_to_destroy` 的 swap 前后语义变化：** swap_state 之前 `state_to_destroy == new_state`（释放新分配的副本），swap_state 之后 `state_to_destroy == old_state`（释放被替换的旧状态）。Driver 在子类化 `atomic_state_clear` 时必须理解这一点。
6. **`duplicated` 标志用于修复不一致：** 通过 `duplicate_state` 创建的副本可能包含不一致状态（如引用了已释放的 fb），Driver 和 helper 应检查此标志进行修复。

## Sources

- [P0] `include/drm/drm_atomic.h` — `struct drm_atomic_state` 定义（含完整字段注释）、`struct __drm_crtcs_state`、`struct __drm_planes_state`、`struct __drm_connnectors_state`、`struct __drm_private_objs_state` 定义
- [P0] `drivers/gpu/drm/drm_atomic.c` — `drm_atomic_state_init()`、`drm_atomic_state_alloc()`、`drm_atomic_state_default_clear()`、`drm_atomic_state_default_release()`、`drm_atomic_get_crtc_state()`、`drm_atomic_get_plane_state()`、`drm_atomic_get_connector_state()` 实现
- [P0] `drivers/gpu/drm/drm_atomic_helper.c` — `drm_atomic_helper_swap_state()`、`drm_atomic_helper_duplicate_state()` 实现
- [P1] https://docs.kernel.org/gpu/drm-kms.html — Kernel Mode Setting 文档，atomic state / atomic_state_alloc / atomic_state_clear / Handling Driver Private State 章节
- [P3] https://lwn.net/Articles/653466 — Daniel Vetter "Atomic mode setting design overview, part 2"（state duplicate/swap 设计理念）
