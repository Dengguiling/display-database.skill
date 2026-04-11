# DRM Atomic Helper 函数集

> **条目ID:** KB-LINUX-DRM-atomic-003
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [state-mgmt](./state-mgmt.md)

---

## 核心摘要

DRM Atomic Helper 是 Linux 内核 DRM 子系统提供的**标准化辅助函数库**（`drm_atomic_helper.c/h`），为 GPU 驱动开发者提供了一套完整的 atomic mode setting 实现模板。其设计哲学是：**驱动只需实现硬件相关的回调，通用逻辑由 helper 函数处理**。这套函数覆盖了从状态校验、硬件提交、异步翻转到挂起恢复的完整生命周期，是几乎所有现代 DRM 驱动（i915、amdgpu、nouveau、rockchip、vc4 等）的基础设施。

---

## 函数分类总览

| 类别 | 函数数量 | 核心职责 |
|------|----------|----------|
| 状态校验 (Check) | 6 | 验证 atomic state 的合法性 |
| 提交执行 (Commit) | 5 | 执行 atomic commit 的各阶段 |
| 异步提交 (Async) | 2 | 非阻塞式页面翻转 |
| 等待同步 (Wait) | 3 | 等待 fence/vblank/flip 完成 |
| Modeset 操作 | 4 | CRTC/Connector 的启用与禁用 |
| Plane 操作 | 6 | Plane 的准备、提交、清理 |
| Legacy 兼容 | 6 | 旧式 KMS 接口的 atomic 封装 |
| 挂起/恢复 | 4 | PM suspend/resume 支持 |
| Page Flip | 2 | VSync 同步的页面翻转 |
| 非阻塞辅助 | 5 | setup_commit / hw_done / cleanup_done |
| Bridge 辅助 | 1 | Bridge bus format 传播 |
| 内联工具 | 4 | plane enabling/disabling 判断 + 遍历宏 |

---

## 1. 状态校验函数 (Check Functions)

### `drm_atomic_helper_check()`
```c
int drm_atomic_helper_check(struct drm_device *dev,
                            struct drm_atomic_state *state);
```
**顶层校验入口。** 按顺序调用以下三个子校验：
1. `drm_atomic_helper_check_modeset()` — 校验 CRTC/Connector/Encoder 的模式变更
2. `drm_atomic_helper_check_planes()` — 校验所有 Plane 的状态
3. `drm_atomic_helper_check_crtc_primary_plane()` — 校验 CRTC 主 plane 与 CRTC 模式的兼容性

驱动通常在 `drm_mode_config_funcs.atomic_check` 中直接调用此函数，或在其前后添加自定义校验逻辑。

### `drm_atomic_helper_check_modeset()`
```c
int drm_atomic_helper_check_modeset(struct drm_device *dev,
                                    struct drm_atomic_state *state);
```
**Modeset 校验核心。** 负责：
- 检查 CRTC 模式变更的合法性（mode clock、connector 支持的模式）
- 处理 CRTC 的启用/禁用状态转换
- 更新 `crtc_state->mode_changed` 和 `crtc_state->connectors_changed` 标志
- 调用 `connector->helper_private->atomic_check()` 回调
- 处理 encoder 的重路由（encoder re-routing）

### `drm_atomic_helper_check_planes()`
```c
int drm_atomic_helper_check_planes(struct drm_device *dev,
                                   struct drm_atomic_state *state);
```
**Plane 校验核心。** 遍历所有发生变更的 plane，调用：
- `plane->helper_private->atomic_check()` 回调
- 对每个 plane 调用 `drm_atomic_helper_check_plane_state()` 进行基础校验

### `drm_atomic_helper_check_plane_state()`
```c
int drm_atomic_helper_check_plane_state(
    struct drm_plane_state *plane_state,
    const struct drm_crtc_state *crtc_state,
    int min_scale, int max_scale,
    bool can_position, bool can_update_disabled);
```
**单 Plane 状态校验。** 验证：
- 源矩形（src_x/y/w/h，16.16 定点数）与目标矩形（crtc_x/y/w/h）的合法性
- 缩放比例是否在 `[min_scale, max_scale]` 范围内
- `can_position=false` 时，目标矩形必须从 (0,0) 开始
- `can_update_disabled=true` 时，允许在 CRTC 禁用时更新 plane

**缩放常量：** `DRM_PLANE_NO_SCALING` (= `1<<16`) 表示无缩放。

### `drm_atomic_helper_check_crtc_primary_plane()`
```c
int drm_atomic_helper_check_crtc_primary_plane(struct drm_crtc_state *crtc_state);
```
**主 Plane 与 CRTC 模式兼容性校验。** 确保主 plane 的可见区域完全覆盖 CRTC 的 active 区域。

### `drm_atomic_helper_check_wb_connector_state()`
```c
int drm_atomic_helper_check_wb_connector_state(
    struct drm_connector *connector,
    struct drm_atomic_state *state);
```
**Writeback Connector 校验。** 验证 writeback connector 的像素格式、buffer 大小等参数。

---

## 2. 提交执行函数 (Commit Functions)

### `drm_atomic_helper_commit()`
```c
int drm_atomic_helper_commit(struct drm_device *dev,
                             struct drm_atomic_state *state,
                             bool nonblock);
```
**标准提交入口。** 实现了完整的 atomic commit 流程：
1. 调用 `drm_atomic_helper_setup_commit()` 初始化提交上下文
2. 调用 `drm_atomic_helper_swap_state()` 交换新旧状态
3. 调用 `commit_tail()` 执行实际硬件变更

`nonblock=true` 时，提交在 worker 线程中异步执行。

### `drm_atomic_helper_commit_tail()`
```c
void drm_atomic_helper_commit_tail(struct drm_atomic_state *state);
```
**Commit Tail 标准实现。** 按严格顺序执行：
1. `drm_atomic_helper_wait_for_dependencies()` — 等待所有依赖的 fence
2. `drm_atomic_helper_commit_modeset_disables()` — 先禁用需要变更的 CRTC
3. `drm_atomic_helper_commit_modeset_enables()` — 再启用新的 CRTC
4. `drm_atomic_helper_commit_planes()` — 提交所有 plane 变更
5. `drm_atomic_helper_wait_for_vblanks()` — 等待 vblank 确认
6. `drm_atomic_helper_cleanup_planes()` — 清理 plane 状态

**关键约束：** 禁用必须在启用之前，避免输出管道中出现不一致状态。

### `drm_atomic_helper_commit_tail_rpm()`
```c
void drm_atomic_helper_commit_tail_rpm(struct drm_atomic_state *state);
```
**RPM 感知的 Commit Tail。** 与标准版类似，但在 modeset disable/enable 前后添加了 `pm_runtime_get/put()` 调用，适用于使用 runtime PM 的驱动。

### `drm_atomic_helper_swap_state()`
```c
int drm_atomic_helper_swap_state(struct drm_atomic_state *state, bool stall);
```
**交换新旧状态。** 将所有对象（CRTC、Plane、Connector）的 `state` 指针从旧状态切换到新状态。`stall=true` 时会等待所有正在进行的 flip 完成，确保硬件状态与软件状态一致。

---

## 3. 异步提交函数 (Async Commit)

### `drm_atomic_helper_async_check()`
```c
int drm_atomic_helper_async_check(struct drm_device *dev,
                                  struct drm_atomic_state *state);
```
**异步提交校验。** 验证当前 atomic state 是否可以异步提交（仅修改单个 plane 的 framebuffer，不涉及 modeset 变更）。

### `drm_atomic_helper_async_commit()`
```c
void drm_atomic_helper_async_commit(struct drm_device *dev,
                                    struct drm_atomic_state *state);
```
**异步提交执行。** 直接修改硬件寄存器完成 plane 更新，不经过标准的 commit_tail 流程。适用于 cursor plane 等需要低延迟更新的场景。

---

## 4. 等待同步函数 (Wait Functions)

### `drm_atomic_helper_wait_for_fences()`
```c
int drm_atomic_helper_wait_for_fences(struct drm_device *dev,
                                      struct drm_atomic_state *state,
                                      bool pre_swap);
```
**等待 DMA Fence。** 在 swap_state 之前（`pre_swap=true`）或之后等待所有关联的 fence 信号完成。确保 framebuffer 的 DMA 操作已完成，避免撕裂。

### `drm_atomic_helper_wait_for_vblanks()`
```c
void drm_atomic_helper_wait_for_vblanks(struct drm_device *dev,
                                        struct drm_atomic_state *old_state);
```
**等待 VBlank。** 等待所有受影响 CRTC 的 vblank 中断，确认上一帧已完全显示。这是 commit 完成的标准同步点。

### `drm_atomic_helper_wait_for_flip_done()`
```c
void drm_atomic_helper_wait_for_flip_done(struct drm_device *dev,
                                          struct drm_atomic_state *old_state);
```
**等待 Flip 完成。** 与 `wait_for_vblanks` 类似，但用于非阻塞提交场景，等待异步 flip 操作完成。

---

## 5. Modeset 操作函数

### `drm_atomic_helper_commit_modeset_disables()`
```c
void drm_atomic_helper_commit_modeset_disables(struct drm_device *dev,
                                               struct drm_atomic_state *state);
```
**禁用 Modeset。** 遍历所有需要禁用的 CRTC，按顺序：
1. 调用 `crtc->helper_private->atomic_disable()` 或 `dpms()` 回调
2. 调用 `encoder->helper_private->disable()` 回调

### `drm_atomic_helper_commit_modeset_enables()`
```c
void drm_atomic_helper_commit_modeset_enables(struct drm_device *dev,
                                              struct drm_atomic_state *old_state);
```
**启用 Modeset。** 遍历所有需要启用的 CRTC，按顺序：
1. 调用 `encoder->helper_private->enable()` 回调
2. 调用 `crtc->helper_private->atomic_enable()` 或 `dpms()` 回调

### `drm_atomic_helper_update_legacy_modeset_state()`
```c
void drm_atomic_helper_update_legacy_modeset_state(struct drm_device *dev,
                                                   struct drm_atomic_state *old_state);
```
**更新 Legacy 状态。** 将 atomic state 变更同步到 legacy KMS 数据结构（`crtc->state`、`connector->state` 等），确保 legacy ioctl 接口能读取到最新状态。

### `drm_atomic_helper_calc_timestamping_constants()`
```c
void drm_atomic_helper_calc_timestamping_constants(struct drm_atomic_state *state);
```
**计算 VBlank 时间戳常量。** 根据 CRTC 模式参数计算 vblank 时间戳相关的常量，用于精确的帧时间测量。

---

## 6. Plane 操作函数

### `drm_atomic_helper_prepare_planes()`
```c
int drm_atomic_helper_prepare_planes(struct drm_device *dev,
                                     struct drm_atomic_state *state);
```
**准备 Plane。** 在硬件提交前调用每个 plane 的 `prepare_fb()` 回调，用于 pin framebuffer、准备 DMA mapping 等。

### `drm_atomic_helper_commit_planes()`
```c
void drm_atomic_helper_commit_planes(struct drm_device *dev,
                                     struct drm_atomic_state *state,
                                     uint32_t flags);
```
**提交 Plane 变更。** 遍历所有变更的 plane，调用 `atomic_update()` 或 `atomic_disable()` 回调。

**Flags：**
| Flag | 值 | 含义 |
|------|----|------|
| `DRM_PLANE_COMMIT_ACTIVE_ONLY` | BIT(0) | 仅提交 active CRTC 上的 plane |
| `DRM_PLANE_COMMIT_NO_DISABLE_AFTER_MODESET` | BIT(1) | modeset 后不自动禁用 plane |

### `drm_atomic_helper_cleanup_planes()`
```c
void drm_atomic_helper_cleanup_planes(struct drm_device *dev,
                                      struct drm_atomic_state *old_state);
```
**清理 Plane。** 在 vblank 等待后调用每个 plane 的 `cleanup_fb()` 回调，释放 `prepare_fb()` 中分配的资源。

### `drm_atomic_helper_unprepare_planes()`
```c
void drm_atomic_helper_unprepare_planes(struct drm_device *dev,
                                        struct drm_atomic_state *state);
```
**反准备 Plane。** 在错误路径中调用，撤销 `prepare_planes()` 的操作。

### `drm_atomic_helper_commit_planes_on_crtc()`
```c
void drm_atomic_helper_commit_planes_on_crtc(struct drm_crtc_state *old_crtc_state);
```
**单 CRTC 上的 Plane 提交。** 仅提交指定 CRTC 关联的 plane 变更，适用于部分更新场景。

### `drm_atomic_helper_disable_planes_on_crtc()`
```c
void drm_atomic_helper_disable_planes_on_crtc(struct drm_crtc_state *old_crtc_state,
                                              bool atomic);
```
**禁用单 CRTC 上的所有 Plane。** 用于 CRTC 禁用时清理关联 plane。

---

## 7. 非阻塞提交辅助函数

### `drm_atomic_helper_setup_commit()`
```c
int drm_atomic_helper_setup_commit(struct drm_atomic_state *state,
                                   bool nonblock);
```
**初始化提交上下文。** 设置 `crtc_state->event`、`commit_work` 等，为非阻塞提交做准备。

### `drm_atomic_helper_wait_for_dependencies()`
```c
void drm_atomic_helper_wait_for_dependencies(struct drm_atomic_state *state);
```
**等待依赖完成。** 等待所有前序 atomic commit 完成，确保提交顺序的正确性。

### `drm_atomic_helper_fake_vblank()`
```c
void drm_atomic_helper_fake_vblank(struct drm_atomic_state *state);
```
**伪造 VBlank 事件。** 在 CRTC 未 active 时（如 modeset enable 阶段），手动发送 vblank 事件以推进 commit 流程。

### `drm_atomic_helper_commit_hw_done()`
```c
void drm_atomic_helper_commit_hw_done(struct drm_atomic_state *state);
```
**标记硬件提交完成。** 通知 DRM core 硬件已接受新的配置，可以开始处理下一个 commit。

### `drm_atomic_helper_commit_cleanup_done()`
```c
void drm_atomic_helper_commit_cleanup_done(struct drm_atomic_state *state);
```
**标记清理完成。** 通知 DRM core 所有资源已释放，commit 流程彻底结束。

---

## 8. Legacy 兼容函数

这些函数将旧式 KMS 接口（`drm_crtc_helper_funcs`）封装为 atomic 操作，使驱动可以同时支持 legacy 和 atomic API。

### `drm_atomic_helper_update_plane()`
```c
int drm_atomic_helper_update_plane(struct drm_plane *plane,
    struct drm_crtc *crtc, struct drm_framebuffer *fb,
    int crtc_x, int crtc_y, unsigned int crtc_w, unsigned int crtc_h,
    uint32_t src_x, uint32_t src_y, uint32_t src_w, uint32_t src_h,
    struct drm_modeset_acquire_ctx *ctx);
```
**Legacy plane 更新。** 将 `drm_plane_funcs.update_plane` 的参数转换为 atomic state 并提交。

### `drm_atomic_helper_disable_plane()`
```c
int drm_atomic_helper_disable_plane(struct drm_plane *plane,
                                    struct drm_modeset_acquire_ctx *ctx);
```
**Legacy plane 禁用。** 通过 atomic 接口禁用指定 plane。

### `drm_atomic_helper_set_config()`
```c
int drm_atomic_helper_set_config(struct drm_mode_set *set,
                                 struct drm_modeset_acquire_ctx *ctx);
```
**Legacy 模式设置。** 将 `drm_crtc_helper_funcs.set_config` 转换为 atomic commit。

### `drm_atomic_helper_disable_all()`
```c
int drm_atomic_helper_disable_all(struct drm_device *dev,
                                  struct drm_modeset_acquire_ctx *ctx);
```
**禁用所有输出。** 关闭所有 CRTC 和 encoder，用于驱动卸载或重置。

### `drm_atomic_helper_shutdown()`
```c
void drm_atomic_helper_shutdown(struct drm_device *dev);
```
**驱动关闭。** 禁用所有 CRTC 并释放相关资源，在 `drm_driver.shutdown` 中调用。

### `drm_atomic_helper_page_flip()`
```c
int drm_atomic_helper_page_flip(struct drm_crtc *crtc,
    struct drm_framebuffer *fb,
    struct drm_pending_vblank_event *event,
    uint32_t flags,
    struct drm_modeset_acquire_ctx *ctx);
```
**Legacy page flip。** 将 `drm_crtc_funcs.page_flip` 转换为 atomic commit。

### `drm_atomic_helper_page_flip_target()`
```c
int drm_atomic_helper_page_flip_target(struct drm_crtc *crtc,
    struct drm_framebuffer *fb,
    struct drm_pending_vblank_event *event,
    uint32_t flags, uint32_t target,
    struct drm_modeset_acquire_ctx *ctx);
```
**带目标的 Legacy page flip。** 支持指定目标 vblank count。

---

## 9. 挂起/恢复函数

### `drm_atomic_helper_duplicate_state()`
```c
struct drm_atomic_state *
drm_atomic_helper_duplicate_state(struct drm_device *dev,
                                  struct drm_modeset_acquire_ctx *ctx);
```
**复制当前全局状态。** 深拷贝所有 DRM 对象的当前状态，用于 suspend 保存。

### `drm_atomic_helper_suspend()`
```c
struct drm_atomic_state *drm_atomic_helper_suspend(struct drm_device *dev);
```
**挂起。** 保存当前显示状态并禁用所有输出。返回保存的状态对象，驱动需持久化此对象。

### `drm_atomic_helper_commit_duplicated_state()`
```c
int drm_atomic_helper_commit_duplicated_state(struct drm_atomic_state *state,
                                              struct drm_modeset_acquire_ctx *ctx);
```
**提交复制状态。** 将之前保存的状态重新应用到硬件，用于 resume 恢复。

### `drm_atomic_helper_resume()`
```c
int drm_atomic_helper_resume(struct drm_device *dev,
                             struct drm_atomic_state *state);
```
**恢复。** 将 suspend 时保存的状态重新应用到硬件，恢复显示输出。

---

## 10. 内联工具函数与宏

### `drm_atomic_plane_enabling()`
```c
static inline bool drm_atomic_plane_enabling(
    struct drm_plane_state *old_plane_state,
    struct drm_plane_state *new_plane_state);
```
判断 plane 是否正在被启用（旧状态无 CRTC，新状态有 CRTC）。若检测到不一致状态（CRTC 和 FB 不同时为 NULL/非-NULL），触发 WARN。

### `drm_atomic_plane_disabling()`
```c
static inline bool drm_atomic_plane_disabling(
    struct drm_plane_state *old_plane_state,
    struct drm_plane_state *new_plane_state);
```
判断 plane 是否正在被禁用（旧状态有 CRTC，新状态无 CRTC）。

### 遍历宏

```c
// 遍历当前附加到 CRTC 的所有 plane（基于当前 state）
drm_atomic_crtc_for_each_plane(plane, crtc)

// 遍历新 state 中将附加到 CRTC 的所有 plane
drm_atomic_crtc_state_for_each_plane(plane, crtc_state)

// 遍历新 state 中的 plane 并获取 plane_state
drm_atomic_crtc_state_for_each_plane_state(plane, plane_state, crtc_state)
```

### `drm_atomic_helper_bridge_propagate_bus_fmt()`
```c
u32 *drm_atomic_helper_bridge_propagate_bus_fmt(
    struct drm_bridge *bridge, struct drm_bridge_state *bridge_state,
    struct drm_crtc_state *crtc_state,
    struct drm_connector_state *conn_state,
    u32 output_fmt, unsigned int *num_input_fmts);
```
**Bridge Bus Format 传播。** 在 display pipeline 中传播像素格式，支持 bridge chain 的格式协商。

---

## 驱动集成模式

### 最小集成（Simple Pipeline 驱动）

```c
static const struct drm_mode_config_funcs my_driver_mode_config_funcs = {
    .atomic_check = drm_atomic_helper_check,
    .atomic_commit = drm_atomic_helper_commit,
};
```

### 自定义校验 + 标准 Commit

```c
static int my_driver_atomic_check(struct drm_device *dev,
                                   struct drm_atomic_state *state)
{
    int ret = drm_atomic_helper_check_modeset(dev, state);
    if (ret)
        return ret;
    ret = drm_atomic_helper_check_planes(dev, state);
    if (ret)
        return ret;
    /* 自定义校验逻辑 */
    return my_driver_custom_check(state);
}
```

### 自定义 Commit Tail

```c
static void my_driver_commit_tail(struct drm_atomic_state *state)
{
    struct drm_device *dev = state->dev;
    drm_atomic_helper_wait_for_dependencies(state);
    drm_atomic_helper_commit_modeset_disables(dev, state);
    /* 自定义硬件编程 */
    my_driver_program_hw(state);
    drm_atomic_helper_commit_modeset_enables(dev, state);
    drm_atomic_helper_commit_planes(dev, state, 0);
    drm_atomic_helper_wait_for_vblanks(dev, state);
    drm_atomic_helper_cleanup_planes(dev, state);
}
```

---

## 潜在影响

- **驱动开发效率：** Atomic helper 将驱动开发从"实现完整 KMS 协议"简化为"实现硬件回调"，大幅降低新驱动开发门槛。
- **标准化：** 所有使用 atomic helper 的驱动遵循相同的 commit 顺序和同步语义，减少了驱动间的行为差异。
- **演进方向：** 内核社区持续推动 helper 函数的完善，v6.x 中新增了 `calc_timestamping_constants`、`fake_vblank` 等函数，反映了对精确时间测量和异步场景的持续优化。

---

## 信源

- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm_atomic_helper.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_atomic_helper.h)
- [drm_atomic_helper.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_atomic_helper.c)
