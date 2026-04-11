# DRM Plane 状态管理

> **条目ID:** KB-LINUX-DRM-atomic-006
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [state-mgmt](./state-mgmt.md) | [helpers](./helpers.md)

---

## 核心摘要

DRM Plane 是 DRM KMS 中最灵活的显示对象——它代表一个可独立控制的图像层，支持位置、缩放、旋转、透明度、混合模式等属性的原子化更新。`struct drm_plane_state` 是 Plane 的可变状态描述符，包含了从源矩形（16.16 定点数）到目标矩形（整数坐标）、从 Framebuffer 引用到 Fence 同步、从颜色管理到 Damage Tracking 的完整状态信息。Plane 状态的原子化管理是 DRM 实现多平面合成（Multi-Plane Composition, MPC）的基础。

---

## 1. Plane 状态核心字段

### `struct drm_plane_state` 完整字段表

| 字段 | 类型 | 说明 |
|------|------|------|
| `plane` | `struct drm_plane *` | 反向指针，指向所属 Plane 对象 |
| `crtc` | `struct drm_crtc *` | 当前绑定的 CRTC，NULL 表示禁用 |
| `fb` | `struct drm_framebuffer *` | 当前绑定的 Framebuffer |
| `fence` | `struct dma_fence *` | 可选的显式 fence，扫描输出前等待 |
| `crtc_x/y` | `int32_t` | 目标矩形左上角坐标（CRTC 坐标系，有符号） |
| `crtc_w/h` | `uint32_t` | 目标矩形宽高（CRTC 坐标系） |
| `src_x/y` | `uint32_t` | 源矩形左上角坐标（FB 坐标系，16.16 定点数） |
| `src_w/h` | `uint32_t` | 源矩形宽高（FB 坐标系，16.16 定点数） |
| `hotspot_x/y` | `int32_t` | 鼠标光标热点偏移（仅 Cursor Plane） |
| `alpha` | `u16` | 不透明度，0=全透明，0xFFFF=全不透明 |
| `pixel_blend_mode` | `uint16_t` | Alpha 混合方程，`DRM_MODE_BLEND_*` |
| `rotation` | `unsigned int` | 旋转角度，`DRM_MODE_ROTATE_*` |
| `zpos` | `unsigned int` | Z 轴优先级（用户可设置，可重复） |
| `normalized_zpos` | `unsigned int` | 归一化 Z 值（0~N-1，唯一，驱动计算） |
| `color_encoding` | `enum drm_color_encoding` | 非 RGB 格式的颜色编码 |
| `color_range` | `enum drm_color_range` | 非 RGB 格式的颜色范围 |
| `fb_damage_clips` | `struct drm_property_blob *` | Damage clips 数组（`drm_mode_rect`） |
| `ignore_damage_clips` | `bool` | 驱动设置，忽略 damage clips |
| `src` | `struct drm_rect` | 校验后的源矩形（由 helper 计算） |
| `dst` | `struct drm_rect` | 裁剪后的目标矩形（由 helper 计算） |
| `visible` | `bool` | Plane 是否可见（裁剪后可能为 false） |
| `scaling_filter` | `enum drm_scaling_filter` | 缩放滤波器 |
| `commit` | `struct drm_crtc_commit *` | 追踪 pending commit，防止 use-after-free |
| `state` | `struct drm_atomic_state *` | 反向指针，指向全局 atomic state |
| `color_mgmt_changed` | `bool` | 颜色管理属性是否变更 |

### 源矩形 vs 目标矩形

```
Framebuffer 坐标系 (16.16 定点数)          CRTC 坐标系 (整数)
┌────────────────────────────┐
│                            │
│    ┌──────────┐            │         ┌──────────────────┐
│    │  src     │            │         │                  │
│    │  矩形    │   ──缩放──→│  dst    │    CRTC 显示区域  │
│    │ (src_x,  │            │  矩形   │                  │
│    │  src_y,  │            │(crtc_x, │                  │
│    │  src_w,  │            │ crtc_y, │                  │
│    │  src_h)  │            │ crtc_w, │                  │
│    └──────────┘            │ crtc_h) │                  │
│                            │         └──────────────────┘
└────────────────────────────┘
```

**关键区别：**
- **源矩形 (src)：** 16.16 定点数，允许亚像素精度。`src_w = 1920 << 16` 表示 1920 像素宽
- **目标矩形 (dst/crtc)：** 整数坐标，有符号（允许部分超出屏幕）
- **缩放：** 硬件自动将 src 矩形缩放到 dst 矩形大小

---

## 2. Plane 类型

| 类型 | 常量 | 说明 | 典型用途 |
|------|------|------|----------|
| Primary | `DRM_PLANE_TYPE_PRIMARY` | 每个 CRTC 一个 | 桌面/应用主画面 |
| Cursor | `DRM_PLANE_TYPE_CURSOR` | 每个 CRTC 一个 | 鼠标光标 |
| Overlay | `DRM_PLANE_TYPE_OVERLAY` | 可多个 | 视频叠加、UI 层 |

```c
// 创建 Plane
int drm_universal_plane_init(struct drm_device *dev,
    struct drm_plane *plane,
    unsigned long possible_crtcs,
    const struct drm_plane_funcs *funcs,
    const uint32_t *formats,
    unsigned int format_count,
    enum drm_plane_type type);
```

---

## 3. 标准 Plane 属性

| 属性 | 创建函数 | 类型 | 说明 |
|------|----------|------|------|
| SRC_X/Y/W/H | 自动创建 | Range | 源矩形（16.16 定点数） |
| CRTC_X/Y/W/H | 自动创建 | Range | 目标矩形（整数） |
| FB_ID | 自动创建 | Object | Framebuffer ID |
| CRTC_ID | 自动创建 | Object | CRTC ID |
| IN_FENCE | 自动创建 | Fence | 输入 fence |
| alpha | `drm_plane_create_alpha_property()` | Range [0, 0xFFFF] | 不透明度 |
| rotation | `drm_plane_create_rotation_property()` | Bitmask | 旋转/反射 |
| zpos | `drm_plane_create_zpos_property()` | Range | Z 轴优先级 |
| zpos (immutable) | `drm_plane_create_zpos_immutable_property()` | Immutable | 固定 Z 值 |
| pixel_blend_mode | `drm_plane_create_blend_mode_property()` | Enum | 混合模式 |
| color_encoding | `drm_plane_create_color_properties()` | Enum | 颜色编码 |
| color_range | `drm_plane_create_color_properties()` | Enum | 颜色范围 |
| scaling_filter | `drm_plane_create_scaling_filter_property()` | Enum | 缩放滤波器 |
| fb_damage_clips | `drm_plane_create_damage_clips_property()` | Blob | Damage 区域 |

---

## 4. Plane 状态校验

### `drm_atomic_helper_check_plane_state()`

```c
int drm_atomic_helper_check_plane_state(
    struct drm_plane_state *plane_state,
    const struct drm_crtc_state *crtc_state,
    int min_scale,
    int max_scale,
    bool can_position,
    bool can_update_disabled);
```

**校验逻辑：**
1. 检查 CRTC 是否有效（除非 `can_update_disabled=true`）
2. 检查 Framebuffer 是否有效
3. 校验源矩形和目标矩形是否合法
4. 检查缩放比例是否在 `[min_scale, max_scale]` 范围内
5. 计算裁剪后的 `src` 和 `dst` 矩形
6. 设置 `visible` 标志

### 校验流程

```
用户空间设置属性
    │
    ▼
drm_atomic_check()
    │
    ├─ drm_atomic_helper_check_modeset()
    │   └─ 校验 CRTC/Connector/Encoder
    │
    ├─ drm_atomic_helper_check_planes()
    │   ├─ 遍历所有 Plane
    │   ├─ 调用 plane->helper_private->atomic_check()
    │   │   └─ 驱动自定义校验
    │   └─ drm_atomic_normalize_zpos()
    │       └─ 计算 normalized_zpos
    │
    └─ 返回校验结果
```

---

## 5. Plane Helper 回调

### `struct drm_plane_helper_funcs`

| 回调 | 说明 |
|------|------|
| `prepare_fb` | 准备 Framebuffer（pin、map、获取 fence） |
| `cleanup_fb` | 清理 Framebuffer（unpin、unmap） |
| `atomic_check` | 校验 Plane 状态的硬件约束 |
| `atomic_update` | 原子更新 Plane（在 VBlank 中调用） |
| `atomic_disable` | 原子禁用 Plane |
| `atomic_async_check` | 异步更新校验（v6.8+ async flip） |
| `atomic_async_update` | 异步更新执行（v6.8+ async flip） |
| `begin_fb_update` | 开始 Framebuffer 更新（v6.14+） |
| `end_fb_update` | 结束 Framebuffer 更新（v6.14+） |

### 典型实现模式

```c
static const struct drm_plane_helper_funcs my_plane_helper_funcs = {
    .prepare_fb = drm_gem_plane_helper_prepare_fb,
    .cleanup_fb = my_plane_cleanup_fb,
    .atomic_check = my_plane_atomic_check,
    .atomic_update = my_plane_atomic_update,
    .atomic_disable = my_plane_atomic_disable,
};
```

---

## 6. Plane 状态的 Old/New 访问模式

### 推荐模式：通过 atomic_state 访问

```c
static int my_plane_atomic_check(struct drm_plane *plane,
                                  struct drm_atomic_state *state)
{
    struct drm_plane_state *new_plane_state =
        drm_atomic_get_new_plane_state(state, plane);
    struct drm_plane_state *old_plane_state =
        drm_atomic_get_old_plane_state(state, plane);

    // 使用 new/old state 进行差异比较
    if (new_plane_state->fb != old_plane_state->fb) {
        // Framebuffer 变更
    }
    return 0;
}
```

### ⚠️ 已废弃模式：直接访问 plane->state

```c
// ❌ 不推荐 — 在非阻塞提交中可能指向错误的状态
struct drm_plane_state *old_state = plane->state;

// ✅ 推荐 — 始终通过 atomic_state 获取
struct drm_plane_state *old_state =
    drm_atomic_get_old_plane_state(state, plane);
```

---

## 7. Damage Tracking

Damage Tracking 允许用户空间告知驱动 Framebuffer 中哪些区域发生了变化，驱动可据此优化 DMA 传输：

```c
// 驱动使用 damage iterator
struct drm_atomic_helper_damage_iter iter;
struct drm_rect clip;
drm_atomic_helper_damage_iter_init(&iter, old_plane_state, new_plane_state);
while (drm_atomic_helper_damage_iter_next(&iter, &clip)) {
    // 仅更新 clip 区域
    my_hw_partial_update(clip.x1, clip.y1, clip.x2 - clip.x1, clip.y2 - clip.y1);
}
```

---

## 8. Fence 集成

### 显式 Fencing

```c
// 用户空间通过 IN_FENCE 属性传递 fence
// DRM core 自动设置 plane_state->fence
// 驱动在 prepare_fb 中处理
static int my_plane_prepare_fb(struct drm_plane *plane,
                                struct drm_plane_state *new_state)
{
    // drm_gem_plane_helper_prepare_fb() 自动处理 fence
    return drm_gem_plane_helper_prepare_fb(plane, new_state);
}
```

### 隐式 Fencing

```c
// 驱动在 prepare_fb 中设置隐式 fence
static int my_plane_prepare_fb(struct drm_plane *plane,
                                struct drm_plane_state *new_state)
{
    struct dma_fence *fence = my_gpu_get_render_fence(new_state->fb);
    drm_atomic_set_fence_for_plane(new_state, fence);
    return 0;
}
```

---

## 潜在影响

- **多平面合成 (MPC)：** Plane 状态管理是 GPU 硬件多平面合成的基础。现代 GPU 通常支持 4-8 个 Overlay Plane，可同时显示视频、UI、Cursor 等多层内容，无需 GPU 渲染合成。
- **性能优化：** Damage Tracking 和 Fence 集成是减少不必要 DMA 传输和同步等待的关键机制，对嵌入式和移动 GPU 的功耗优化尤为重要。
- **Async Flip：** v6.8+ 的 `atomic_async_check/update` 回调为低延迟场景（游戏、VR）提供了不等待 VBlank 的 Plane 更新路径。
- **演进方向：** v6.14+ 新增的 `begin_fb_update/end_fb_update` 回调为 Framebuffer 更新提供了更精细的生命周期管理。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm_plane.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_plane.h)
- [drm_plane.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_plane.c)
