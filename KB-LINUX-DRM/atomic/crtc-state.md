# DRM CRTC 状态管理

> **条目ID:** KB-LINUX-DRM-atomic-008
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [state-mgmt](./state-mgmt.md) | [plane-state](./plane-state.md) | [connector-state](./connector-state.md)

---

## 核心摘要

CRTC（Cathode Ray Tube Controller）是 DRM KMS 的核心显示控制器——它代表一个独立的显示管线，负责将 Plane 合成后的像素数据按指定时序扫描输出到 Encoder。`struct drm_crtc_state` 是 CRTC 的可变状态描述符，包含：显示模式（用户请求 vs 硬件实际）、启用/活跃状态、颜色管理 LUT/CTM、VRR、Self-Refresh、缩放滤波、VBlank 事件等。CRTC 状态是 atomic commit 的决策中枢——`drm_atomic_crtc_needs_modeset()` 根据其标志位判断是否需要完整的模式切换。

---

## 1. CRTC 状态核心字段

### `struct drm_crtc_state` 完整字段表

| 字段 | 类型 | 说明 |
|------|------|------|
| `crtc` | `struct drm_crtc *` | 反向指针，指向所属 CRTC |
| `enable` | `bool` | 资源分配开关。控制共享资源的预留，不直接控制硬件 |
| `active` | `bool` | 硬件活跃状态（DPMS）。`active=false` 时硬件应尽可能关闭 |
| `planes_changed` | `bool` | Plane 状态已变更 |
| `mode_changed` | `bool` | 显示模式或 enable 已变更，需要完整 modeset |
| `active_changed` | `bool` | active 状态已切换 |
| `connectors_changed` | `bool` | Connector 路由已变更 |
| `zpos_changed` | `bool` | Plane zpos 已变更 |
| `color_mgmt_changed` | `bool` | 颜色管理属性已变更 |
| `no_vblank` | `bool` | 无 VBlank 中断能力，需发送 fake vblank 事件 |
| `plane_mask` | `u32` | 已绑定 Plane 的位掩码 |
| `connector_mask` | `u32` | 已绑定 Connector 的位掩码 |
| `encoder_mask` | `u32` | 已绑定 Encoder 的位掩码 |
| `adjusted_mode` | `struct drm_display_mode` | 驱动内部调整后的显示时序 |
| `mode` | `struct drm_display_mode` | 用户空间请求的显示时序 |
| `mode_blob` | `struct drm_property_blob *` | mode 的 blob 属性 |
| `degamma_lut` | `struct drm_property_blob *` | 逆 Gamma 查找表 |
| `ctm` | `struct drm_property_blob *` | 颜色转换矩阵 (3×3 s3131) |
| `gamma_lut` | `struct drm_property_blob *` | Gamma 查找表（也用于 CLUT/调色板） |
| `target_vblank` | `u32` | 页面翻转的目标 VBlank 周期 |
| `async_flip` | `bool` | 异步翻转标志（`DRM_MODE_PAGE_FLIP_ASYNC`） |
| `vrr_enabled` | `bool` | 可变刷新率启用 |
| `self_refresh_active` | `bool` | 自刷新模式活跃 |
| `scaling_filter` | `enum drm_scaling_filter` | 缩放滤波器类型 |
| `event` | `struct drm_pending_vblank_event *` | 待发送的 VBlank 事件 |
| `state` | `struct drm_atomic_state *` | 反向指针，指向全局 atomic state |

---

## 2. enable vs active：微妙但关键的区别

这是 DRM 中最容易混淆的概念之一：

```
enable = true, active = true   →  正常显示，硬件全开
enable = true, active = false  →  资源已预留，硬件关闭（DPMS OFF）
enable = false, active = false →  完全禁用，资源已释放
enable = false, active = true  →  非法状态
```

### 设计意图

| 维度 | `enable` | `active` |
|------|----------|----------|
| 控制对象 | 共享资源预留（PLL、带宽） | 硬件扫描输出状态 |
| DPMS 对应 | — | DPMS On/Off |
| `atomic_check` 可拒绝？ | ✅ 可以拒绝 | ❌ 不应拒绝（DPMS ON 必须成功） |
| 资源释放 | `enable=false` 时释放 | `active=false` 时保留资源 |
| 用户空间假设 | — | DPMS ON 永远成功 |

### 实际影响

```c
// 驱动在 atomic_check 中：
// ✅ 正确：基于 enable 拒绝（资源不足）
if (new_crtc_state->enable && !has_bandwidth(state))
    return -ENOSPC;

// ❌ 错误：基于 active 拒绝（违反 DPMS 语义）
if (new_crtc_state->active && !has_bandwidth(state))
    return -ENOSPC;  // 用户空间 DPMS ON 可能失败！
```

---

## 3. Modeset 判断逻辑

### `drm_atomic_crtc_needs_modeset()`

```c
static inline bool drm_atomic_crtc_needs_modeset(const struct drm_crtc_state *state)
{
    return state->active_changed || state->mode_changed ||
           state->connectors_changed;
}
```

### 标志位设置时机

| 标志位 | 设置者 | 含义 |
|--------|--------|------|
| `mode_changed` | DRM core（mode/enable 变更时）或驱动 | 需要完整 modeset |
| `active_changed` | DRM core（active 变更时） | DPMS 状态切换 |
| `connectors_changed` | DRM core 或驱动（Encoder 路由变更时） | 输出路由变更 |
| `planes_changed` | DRM core（Plane 绑定变更时） | 仅 Plane 更新 |
| `zpos_changed` | DRM core（zpos 属性变更时） | Plane 层叠顺序变更 |
| `color_mgmt_changed` | DRM core（LUT/CTM 变更时） | 颜色管理参数变更 |

### Commit 流程中的决策树

```
drm_atomic_crtc_needs_modeset(state)?
├─ YES → 完整 modeset
│   ├─ 禁用旧 CRTC（释放资源）
│   ├─ 重新配置 PLL/时序
│   ├─ 重新训练链路
│   └─ 启用新 CRTC
│
└─ NO → 快速更新
    ├─ planes_changed? → 仅更新 Plane
    ├─ color_mgmt_changed? → 仅更新颜色管理
    └─ 其他属性变更 → 驱动特定处理
```

---

## 4. 显示模式：mode vs adjusted_mode

### `mode` — 用户空间请求的时序

- 由用户空间通过 `drm_mode_modeinfo` 设置
- **active width/height 必须精确匹配**（Plane 定位依赖此值）
- 刷新率应尽量匹配，但允许微小偏差（HDMI 模式差异 <1%）
- 对于外部 Connector，应与线缆上的物理时序完全一致

### `adjusted_mode` — 驱动内部调整后的时序

- 驱动在 `atomic_check` 中修改
- 用于处理用户请求时序与硬件能力之间的差异
- **Bridge 驱动：** 存储 CRTC 到第一个 Bridge 之间的硬件时序
- **其他驱动：** 存储内部使用的硬件时序

### 典型调整场景

```c
static int my_driver_crtc_atomic_check(struct drm_crtc *crtc,
                                        struct drm_atomic_state *state)
{
    struct drm_crtc_state *crtc_state = drm_atomic_get_new_crtc_state(state, crtc);

    // 复制用户请求的时序
    memcpy(&crtc_state->adjusted_mode, &crtc_state->mode,
           sizeof(crtc_state->adjusted_mode));

    // 驱动特定调整：例如对齐到硬件精度
    crtc_state->adjusted_mode.clock =
        round_up(crtc_state->adjusted_mode.clock, CLOCK_STEP);
}
```

---

## 5. 颜色管理

### 管线结构

```
Framebuffer 像素 → [degamma_lut] → [CTM] → [gamma_lut] → 输出
```

### LUT 格式

| 组件 | 结构体 | 说明 |
|------|--------|------|
| `degamma_lut` | `struct drm_color_lut[]` | 逆 Gamma，将 FB 像素从非线性转为线性 |
| `ctm` | `struct drm_color_ctm` | 3×3 颜色转换矩阵，s3131 定点格式 |
| `gamma_lut` | `struct drm_color_lut[]` | Gamma 校正，也用于 C8 格式的调色板 |

### 启用颜色管理

```c
// 驱动初始化时声明支持的颜色管理能力
drm_crtc_enable_color_mgmt(crtc,
    degamma_lut_size,    // 0 = 不支持
    has_ctm,             // 是否支持 CTM
    gamma_lut_size);     // 0 = 不支持
```

---

## 6. VRR（可变刷新率）

```c
// 用户空间设置 VRR
crtc_state->vrr_enabled = true;

// 驱动行为：
// - 不再严格按 VBlank 边界翻转
// - 允许在任意时刻提交新帧（减少输入延迟）
// - 需要硬件支持（FreeSync / G-Sync Compatible）
```

### VRR 与 Async Flip 的区别

| 维度 | VRR | Async Flip |
|------|-----|------------|
| 刷新率 | 动态调整（48-144Hz 等） | 固定刷新率 |
| 翻转时机 | 最近的 VBlank | 立即（不等 VBlank） |
| Tearing | 无 | 可能有 |
| 适用场景 | 游戏低延迟 | 游戏低延迟 + 低帧率 |

---

## 7. Self-Refresh（自刷新）

```
正常模式:
  GPU 渲染 → CRTC 扫描输出 → Panel 显示

自刷新模式:
  Panel 内部 RAM 保持最后一帧 → GPU/CRTC 进入低功耗
  （屏幕内容不变时自动触发）
```

- `self_refresh_active` 标志在 enable/disable 回调中设置
- 驱动可检查此标志决定是否完全关闭 CRTC
- **适用：** 移动设备、笔记本 OLED 屏幕

---

## 8. Fake VBlank

当 CRTC 不支持 VBlank 中断时（`no_vblank=true`）：

```c
// drm_atomic_helper_check_modeset() 自动设置
// drm_atomic_helper_fake_vblank() 在 commit 后发送
// 驱动无需额外处理
```

**适用场景：**
- 没有 VBlank 中断的简单硬件
- Writeback Connector（oneshot 模式）
- 驱动未调用 `drm_vblank_init()` 的 CRTC

---

## 9. Scaling Filter

```c
enum drm_scaling_filter {
    DRM_SCALING_FILTER_DEFAULT,    // 驱动默认
    DRM_SCALING_FILTER_NEAREST_NEIGHBOR,  // 最近邻（像素风格）
    DRM_SCALING_FILTER_BILINEAR,   // 双线性
};
```

- 通过 `DRM_PROP_SCALING_FILTER` 属性设置
- 驱动需声明支持：`drm_plane_create_scaling_filter_property()`
- **用途：** 整数缩放（RetroArch 风格）、低分辨率游戏显示

---

## 潜在影响

- **Compositor 决策：** `mode_changed` / `connectors_changed` 标志是 Wayland compositor 判断是否需要完整 modeset 的依据，直接影响热插拔和模式切换的用户体验。
- **VRR 生态：** VRR 支持是 Linux 游戏体验的关键差异化特性，Gamescope、MangoHud 等工具依赖 CRTC 的 VRR 状态。
- **HDR 管线：** `degamma_lut` → `CTM` → `gamma_lut` 管线是 HDR 显示的基础，Wayland 的 color-management 协议直接映射到此管线。
- **演进方向：** 社区在 v6.x 中持续增强 CRTC 状态管理，新增了 `async_flip`（v6.8）、scaling filter（v5.4+）、VRR 改进等特性。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm_crtc.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_crtc.h)
- [drm_crtc.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_crtc.c)
