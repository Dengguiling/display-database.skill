# DRM Connector 状态管理

> **条目ID:** KB-LINUX-DRM-atomic-007
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [state-mgmt](./state-mgmt.md) | [plane-state](./plane-state.md)

---

## 核心摘要

DRM Connector 是连接显示输出端（显示器、HDMI/DP 线缆、面板）的抽象层。`struct drm_connector_state` 描述了 Connector 的可变状态，包括：当前连接的 CRTC、选定的 Encoder、链路状态、HDCP 内容保护、HDR 元数据、色彩空间、HDMI Infoframe 等。Connector 状态的原子化管理确保了显示输出链路（CRTC → Encoder → Connector → Sink）的配置变更以事务方式执行。

---

## 1. Connector 状态核心字段

### `struct drm_connector_state` 完整字段表

| 字段 | 类型 | 说明 |
|------|------|------|
| `connector` | `struct drm_connector *` | 反向指针，指向所属 Connector |
| `crtc` | `struct drm_crtc *` | 当前连接的 CRTC，NULL 表示禁用 |
| `best_encoder` | `struct drm_encoder *` | 原子 helper 选定的 Encoder |
| `link_status` | `enum drm_link_status` | 链路状态（GOOD/BAD） |
| `state` | `struct drm_atomic_state *` | 反向指针，指向全局 atomic state |
| `commit` | `struct drm_crtc_commit *` | 追踪 pending commit |
| `tv` | `struct drm_tv_connector_state` | TV Connector 状态（模拟输出） |
| `self_refresh_aware` | `bool` | 是否支持自刷新模式 |
| `picture_aspect_ratio` | `enum hdmi_picture_aspect` | HDMI 宽高比 |
| `content_type` | `unsigned int` | HDMI 内容类型（游戏/电影/照片/文字） |
| `hdcp_content_type` | `unsigned int` | HDCP 内容类型 |
| `scaling_mode` | `unsigned int` | 缩放模式（内置面板） |
| `content_protection` | `unsigned int` | 内容保护状态（HDCP） |
| `colorspace` | `enum drm_colorspace` | 色彩空间（sRGB/BT2020 等） |
| `writeback_job` | `struct drm_writeback_job *` | Writeback 连接器任务 |
| `max_requested_bpc` | `u8` | 用户请求的最大位深 |
| `max_bpc` | `u8` | 实际最大位深（受 EDID 限制） |
| `privacy_screen_sw_state` | `enum drm_privacy_screen_status` | 隐私屏状态 |
| `hdr_output_metadata` | `struct drm_property_blob *` | HDR 输出元数据 |
| `hdmi` | `struct drm_connector_hdmi_state` | HDMI 专属状态 |

### `struct drm_connector_hdmi_state`（v6.13+）

| 字段 | 类型 | 说明 |
|------|------|------|
| `broadcast_rgb` | `enum drm_hdmi_broadcast_rgb` | RGB 广播范围选择 |
| `infoframes.avi` | `struct drm_connector_hdmi_infoframe` | AVI Infoframe |
| `infoframes.hdr_drm` | `struct drm_connector_hdmi_infoframe` | HDR DRM Infoframe |
| `infoframes.spd` | `struct drm_connector_hdmi_infoframe` | SPD Infoframe |
| `infoframes.hdmi` | `struct drm_connector_hdmi_infoframe` | HDMI Vendor Infoframe |
| `is_limited_range` | `bool` | 是否使用有限 RGB 量化范围 |
| `output_bpc` | `unsigned int` | 每通道输出位深 |
| `output_format` | `enum hdmi_colorspace` | 输出像素格式 |
| `tmds_char_rate` | `unsigned long long` | TMDS 字符速率 (Hz) |

---

## 2. Connector 类型

| 类型 | 常量 | 说明 |
|------|------|------|
| DP | `DRM_MODE_CONNECTOR_DisplayPort` | DisplayPort |
| HDMI | `DRM_MODE_CONNECTOR_HDMIA/B` | HDMI A/B |
| DVI | `DRM_MODE_CONNECTOR_DVII/D/A` | DVI |
| VGA | `DRM_MODE_CONNECTOR_VGA` | 模拟 VGA |
| eDP | `DRM_MODE_CONNECTOR_eDP` | 嵌入式 DisplayPort |
| LVDS | `DRM_MODE_CONNECTOR_LVDS` | 低压差分信号 |
| Writeback | `DRM_MODE_CONNECTOR_Writeback` | 写回连接器 |
| Virtual | `DRM_MODE_CONNECTOR_VIRTUAL` | 虚拟连接器 |
| DPI | `DRM_MODE_CONNECTOR_DPI` | 显示像素接口 |
| TV | `DRM_MODE_CONNECTOR_SVIDEO/Composite/...` | 电视输出 |

---

## 3. 标准 Connector 属性

### 通用属性

| 属性 | 类型 | 说明 |
|------|------|------|
| EDID | Blob | 显示器 EDID 数据 |
| DPMS | Enum | 电源管理状态 |
| link-status | Enum | 链路状态（Good/Bad） |
| non-desktop | Immutable | 非桌面设备（VR 头显等） |
| tile | Immutable | 平铺显示信息 |
| panel orientation | Enum | 面板方向（旋转） |

### HDMI 专属属性

| 属性 | 类型 | 说明 |
|------|------|------|
| broadcast-rgb | Enum | RGB 量化范围（Full/Limited/Auto） |
| content-type | Enum | 内容类型 |
| hdr-output-metadata | Blob | HDR 静态元数据 |
| max-bpc | Range | 最大位深 |
| hdcp-content-type | Enum | HDCP 内容类型 |
| content-protection | Enum | 内容保护状态 |
| picture-aspect-ratio | Enum | 宽高比 |
| colorspace | Enum | 色彩空间 |

### DP 专属属性

| 属性 | 类型 | 说明 |
|------|------|------|
| DP-MST | — | 多流传输 |
| DP-color-range | Enum | 颜色范围 |
| DP-color-depth | Enum | 颜色深度 |
| DP-link-rate | Enum | 链路速率 |
| DP-lane-count | Enum | 通道数 |

---

## 4. Connector Helper 回调

### `struct drm_connector_helper_funcs`

| 回调 | 说明 |
|------|------|
| `get_modes` | 从 EDID/DDC 获取支持的显示模式列表 |
| `detect_ctx` | 检测是否有设备连接 |
| `atomic_best_encoder` | 原子模式下选择最佳 Encoder |
| `best_encoder` | 旧式 Encoder 选择（非原子） |
| `atomic_check` | 校验 Connector 状态的硬件约束 |
| `atomic_commit` | 提交 Connector 状态变更 |
| `enable_hpd` | 启用热插拔检测 |
| `disable_hpd` | 禁用热插拔检测 |
| `dpms` | 旧式电源管理 |

### Encoder 选择逻辑

```
atomic_commit 流程
    │
    ├─ drm_atomic_helper_check_modeset()
    │   ├─ 遍历所有 Connector
    │   ├─ 调用 connector->helper_private->atomic_best_encoder()
    │   │   └─ 返回 best_encoder
    │   ├─ 设置 connector_state->best_encoder
    │   └─ 校验 Encoder 可用性
    │
    └─ commit_tail
        ├─ drm_atomic_helper_commit_modeset_disables()
        │   └─ 禁用旧 Encoder/Connector
        ├─ drm_atomic_helper_commit_modeset_enables()
        │   └─ 启用新 Encoder/Connector
        └─ ...
```

---

## 5. HDMI Connector（v6.13+ 新架构）

v6.13 引入了结构化的 HDMI Connector 框架，将 HDMI 状态管理从驱动私有实现提升为内核标准接口：

```c
// 初始化 HDMI Connector
int drm_hdmi_connector_init(struct drm_device *dev,
    struct drm_connector *connector,
    const struct drm_connector_funcs *funcs,
    const struct drm_connector_hdmi_funcs *hdmi_funcs,
    unsigned int max_bpc,
    struct i2c_adapter *ddc);
```

### HDMI Infoframe 管理

```c
// HDMI Infoframe 类型
struct drm_connector_hdmi_infoframe {
    enum hdmi_infoframe_type type;
    union {
        struct hdmi_avi_infoframe avi;
        struct hdmi_spd_infoframe spd;
        struct hdmi_hdr_infoframe hdr_drm;
        struct hdmi_vendor_infoframe hdmi;
    };
};
```

### HDMI 校验流程

```c
// 在 atomic_check 中调用
int drm_atomic_helper_connector_hdmi_check(struct drm_connector *connector,
    struct drm_atomic_state *state);
```

**校验内容：**
1. TMDS 字符速率是否在硬件支持范围内
2. 输出格式（RGB/YCbCr 4:4:4/4:2:2/4:2:0）是否合法
3. 输出位深是否在支持范围内
4. Infoframe 内容是否合法
5. 色彩空间和量化范围是否匹配

---

## 6. Writeback Connector

Writeback Connector 允许将 CRTC 输出写回到 Framebuffer（而非显示设备），用于截图、录制等场景：

```c
// 创建 Writeback Connector
int drm_writeback_connector_init(struct drm_device *dev,
    struct drm_writeback_connector *wb_connector,
    const struct drm_connector_funcs *funcs,
    const struct drm_writeback_connector_funcs *wb_funcs,
    const u32 *possible_crtcs,
    int n_formats,
    ...);
```

### Writeback 工作流

```
1. 用户空间创建 Writeback Framebuffer
2. 设置 Writeback Connector 的 WRITEBACK_FB_ID 属性
3. 设置 WRITEBACK_OUT_FENCE_PTR 属性获取完成 fence
4. 执行 atomic commit
5. 驱动在 commit_tail 中将 CRTC 输出写入 Writeback FB
6. 驱动调用 drm_writeback_signal_completion() 通知完成
```

---

## 7. 链路状态管理

### 热插拔检测 (HPD)

```c
// 驱动在 HPD 中断中调用
void drm_kms_helper_hotplug_event(struct drm_device *dev);
```

### 链路状态属性

```c
// 用户空间读取链路状态
// DRM_MODE_LINK_STATUS_GOOD — 链路正常
// DRM_MODE_LINK_STATUS_BAD — 链路异常，需要重新训练
```

### DP 链路训练

```
HPD 事件
    │
    ├─ drm_kms_helper_hotplug_event()
    ├─ 用户空间收到 uevent
    ├─ 重新读取 EDID
    ├─ 检测显示模式变更
    ├─ 执行 atomic modeset
    └─ DP 链路训练（如果需要）
```

---

## 8. 内容保护 (HDCP)

### HDCP 状态机

```
UNDESIRED ──用户请求──→ DESIRED
    │                      │
    │                      ├─ 链路训练成功 ──→ ENABLED
    │                      │                      │
    │                      │                      ├─ 链路异常 ──→ DESIRED（重新认证）
    │                      │                      └─ 用户取消 ──→ UNDESIRED
    │                      │
    │                      └─ 认证失败 ──→ DESIRED（重试）
    │
    └─ ...（循环）
```

### 属性值

| 值 | 说明 |
|------|------|
| `DRM_MODE_CONTENT_PROTECTION_UNDESIRED` | 未请求保护 |
| `DRM_MODE_CONTENT_PROTECTION_DESIRED` | 请求保护中 |
| `DRM_MODE_CONTENT_PROTECTION_ENABLED` | 保护已启用 |

---

## 潜在影响

- **HDMI 2.1/DP 2.1：** v6.13+ 的结构化 HDMI 框架为 HDMI 2.1 特性（DSC、FRL、VRR）的标准化实现奠定了基础。
- **HDR 支持：** `hdr_output_metadata` 和 HDR DRM Infoframe 的标准化管理，使得用户空间（Wayland compositor）可以正确传递 HDR 静态元数据到显示器。
- **Writeback：** Writeback Connector 是 PipeWire 屏幕录制、GNOME/Sway 截图功能的基础设施。
- **演进方向：** 社区在 v6.x 中持续完善 HDMI/DP Connector 的标准化，减少驱动私有实现，提升跨驱动一致性。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [DRM Connector — kernel.org](https://docs.kernel.org/gpu/drm-kms.html#connector-functions)
- [drm_connector.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_connector.h)
- [drm_writeback.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_writeback.c)
