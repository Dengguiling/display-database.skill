# DRM Encoder 与 Bridge 状态管理

> **条目ID:** KB-LINUX-DRM-atomic-009
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档 + Bootlin KMS API 概览）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13, bootlin.com
> **关联:** [commit-flow](./commit-flow.md) | [crtc-state](./crtc-state.md) | [connector-state](./connector-state.md)

---

## 核心摘要

DRM Encoder 是 CRTC 与 Connector 之间的信号转换抽象层——它将 CRTC 输出的数字像素流转换为特定物理接口的信号（TMDS/HDMI、LVDS、DP、DSI 等）。**Encoder 没有独立的 atomic state 结构体**，这是 DRM 的一个关键设计决策：Encoder 的状态完全由 CRTC State 和 Connector State 推导。然而，Encoder 的 helper 回调（`drm_encoder_helper_funcs`）在 atomic commit 流程中扮演重要角色。Bridge（`drm_bridge`）作为 Encoder 的链式扩展，拥有自己的 atomic state（`drm_bridge_state`），用于管理多级信号转换管线。

---

## 1. Encoder 核心结构

### `struct drm_encoder` 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct drm_device *` | 所属 DRM 设备 |
| `base` | `struct drm_mode_object` | 基类对象（ID、类型） |
| `name` | `char *` | Encoder 名称 |
| `encoder_type` | `int` | 编码器类型（见下表） |
| `index` | `unsigned` | 在 mode_config 中的索引 |
| `possible_crtcs` | `uint32_t` | 可连接的 CRTC 位掩码 |
| `possible_clones` | `uint32_t` | 可克隆的 Encoder 位掩码 |
| `crtc` | `struct drm_crtc *` | 当前绑定的 CRTC（仅 legacy） |
| `bridge_chain` | `struct list_head` | 附加的 Bridge 链表 |
| `funcs` | `const struct drm_encoder_funcs *` | 核心回调 |
| `helper_private` | `const struct drm_encoder_helper_funcs *` | Helper 回调 |

### Encoder 类型

| 类型 | 常量 | 典型用途 |
|------|------|----------|
| DAC | `DRM_MODE_ENCODER_DAC` | VGA、DVI-A 模拟输出 |
| TMDS | `DRM_MODE_ENCODER_TMDS` | DVI-D、HDMI、嵌入式 DP |
| LVDS | `DRM_MODE_ENCODER_LVDS` | 笔记本面板、并行接口 |
| TVDAC | `DRM_MODE_ENCODER_TVDAC` | 复合视频、S-Video、分量 |
| DSI | `DRM_MODE_ENCODER_DSI` | MIPI DSI 串行面板 |
| DPI | `DRM_MODE_ENCODER_DPI` | MIPI DPI 并行面板 |
| Virtual | `DRM_MODE_ENCODER_VIRTUAL` | 虚拟机显示 |
| DPMST | `DRM_MODE_ENCODER_DPMST` | DP MST 虚拟 Encoder |

---

## 2. 为什么 Encoder 没有 Atomic State

这是 DRM 架构的一个关键设计决策（Bootlin 2023 KMS API 概览中明确指出）：

```
┌─────────┐    ┌──────────┐    ┌──────────┐
│  CRTC   │───→│ Encoder  │───→│Connector │
│ State ✓ │    │ State ✗  │    │ State ✓  │
└─────────┘    └──────────┘    └──────────┘
```

**设计理由：**
1. **Encoder 是无状态的信号转换器：** 它不持有任何需要原子化更新的可变状态
2. **状态由端点推导：** CRTC State 描述"输出什么"，Connector State 描述"输出到哪里"，Encoder 只需在两者之间做信号转换
3. **简化状态管理：** 减少 atomic state 的复杂度，避免冗余状态同步
4. **Bridge 弥补了这一空白：** 当需要中间状态时，使用 Bridge 的 atomic state

### 对比：Bridge 有 Atomic State

```c
struct drm_bridge_state {
    struct drm_private_obj base;
    struct drm_atomic_state *state;  // 反向指针
    struct drm_bridge *bridge;       // 反向指针

    // 输入/输出总线格式
    u32 input_bus_cfg;
    u32 output_bus_cfg;

    // 输入/输出色彩编码
    u32 input_bus_flags;
    u32 output_bus_flags;

    // 定时调整
    struct drm_display_mode adjusted_mode;
};
```

---

## 3. Encoder Helper 回调

### `struct drm_encoder_helper_funcs`

| 回调 | 说明 |
|------|------|
| `atomic_check` | 校验 Encoder 配置（总线格式、时钟等） |
| `atomic_enable` | 启用 Encoder（上电、配置信号输出） |
| `atomic_disable` | 禁用 Encoder（关闭信号输出） |
| `mode_set` | Legacy 模式设置（非 atomic） |
| `mode_fixup` | 调整显示时序以适配 Encoder 能力 |
| `detect` | 检测连接状态（仅 legacy） |
| `get_modes` | 获取支持的显示模式（仅 legacy） |
| `dpms` | Legacy DPMS 控制 |

### Atomic Commit 中的调用时机

```
drm_atomic_helper_commit_modeset_enables()
├─ 遍历所有需要启用的 CRTC
│   ├─ 调用 CRTC enable 回调
│   ├─ 调用 Encoder atomic_enable 回调
│   └─ 调用 Bridge chain enable 回调

drm_atomic_helper_commit_modeset_disables()
├─ 遍历所有需要禁用的 CRTC
│   ├─ 调用 Bridge chain disable 回调
│   ├─ 调用 Encoder atomic_disable 回调
│   └─ 调用 CRTC disable 回调
```

**关键顺序：** Enable 时 CRTC 先于 Encoder，Disable 时 Encoder 先于 CRTC。

---

## 4. Bridge Chain 架构

### Bridge 的角色

Bridge 是 Encoder 的可插拔扩展，用于处理多级信号转换：

```
CRTC → Encoder → Bridge A → Bridge B → Bridge C → Connector
                  (PHY)      (SCDC)     (Panel)
```

### 典型 Bridge 类型

| Bridge | 职责 | 示例 |
|--------|------|------|
| PHY Bridge | 物理层信号转换 | `ti-sn65dsi86`（DSI→DP） |
| Panel Bridge | 面板时序控制 | `panel-simple` |
| HDMI TX Bridge | HDMI 信号编码 | `dw-hdmi` |
| DP Bridge | DisplayPort 协议 | `analogix_dp` |
| Throttle Bridge | 带宽节流 | `ps8640` |
| MST Bridge | DP MST 分流 | `i915` 内部实现 |

### Bridge Atomic State

```c
// Bridge 驱动声明 atomic state
struct my_bridge_state {
    struct drm_bridge_state base;
    u32 bus_format;
    u32 output_mode;
};

static const struct drm_bridge_funcs my_bridge_funcs = {
    .atomic_duplicate_state = drm_atomic_helper_bridge_duplicate_state,
    .atomic_destroy_state = drm_atomic_helper_bridge_destroy_state,
    .atomic_reset = drm_atomic_helper_bridge_reset,
    .atomic_get_state = drm_atomic_helper_bridge_get_state,
    .atomic_check = my_bridge_atomic_check,
    .atomic_enable = my_bridge_atomic_enable,
    .atomic_disable = my_bridge_atomic_disable,
    .atomic_pre_enable = my_bridge_atomic_pre_enable,
    .atomic_post_disable = my_bridge_atomic_post_disable,
};
```

### Bridge Commit 顺序

```
Pre-enable（从 Connector 端向 CRTC 端）:
  Bridge C pre_enable → Bridge B pre_enable → Bridge A pre_enable

Enable（从 CRTC 端向 Connector 端）:
  Bridge A enable → Bridge B enable → Bridge C enable

Disable（从 Connector 端向 CRTC 端）:
  Bridge C disable → Bridge B disable → Bridge A disable

Post-disable（从 CRTC 端向 Connector 端）:
  Bridge A post_disable → Bridge B post_disable → Bridge C post_disable
```

---

## 5. Encoder 与 Connector 的路由

### `best_encoder` 选择

在 atomic check 阶段，helper 函数为每个 Connector 选择最佳 Encoder：

```c
// drm_atomic_helper_check_modeset() 中：
connector_state->best_encoder =
    connector->helper_private->atomic_best_encoder(connector, connector_state);
```

### 路由约束

```
Encoder.possible_crtcs ──→ 限制可连接的 CRTC
Connector.possible_encoders ──→ 限制可使用的 Encoder
```

### MST 特殊处理

DP MST（Multi-Stream Transport）使用虚拟 Encoder：

```
物理 DP Encoder
├─ MST Virtual Encoder 1 → CRTC 1 → Connector 1（显示器 1）
├─ MST Virtual Encoder 2 → CRTC 2 → Connector 2（显示器 2）
└─ MST Virtual Encoder 3 → CRTC 3 → Connector 3（显示器 3）
```

- 虚拟 Encoder 类型为 `DRM_MODE_ENCODER_DPMST`
- 每个 MST 端口有独立的虚拟 Encoder
- 共享物理链路的带宽分配由 DP MST 拓扑管理器处理

---

## 6. Clone 模式（双屏复制）

```
                    ┌─ Encoder A → Connector A → 显示器 A
CRTC ──possible_clones──┤
                    └─ Encoder B → Connector B → 显示器 B
```

- `possible_clones` 位掩码定义哪些 Encoder 可以同时绑定到同一 CRTC
- 两个 Encoder 必须互相设置对方的 bit
- **限制：** Clone 模式下两个显示器必须使用相同的显示时序

---

## 7. 驱动实现示例

### 简单 Encoder（无 Bridge）

```c
static const struct drm_encoder_helper_funcs my_encoder_helper_funcs = {
    .atomic_check = my_encoder_atomic_check,
    .atomic_enable = my_encoder_atomic_enable,
    .atomic_disable = my_encoder_atomic_disable,
};

static int my_encoder_atomic_check(struct drm_encoder *encoder,
                                    struct drm_crtc_state *crtc_state,
                                    struct drm_connector_state *conn_state)
{
    // 校验总线格式兼容性
    struct drm_bridge *bridge;
    drm_for_each_bridge_in_chain(encoder, bridge) {
        if (bridge->funcs->atomic_check) {
            int ret = bridge->funcs->atomic_check(bridge,
                bridge->funcs->atomic_get_state(bridge, crtc_state->state),
                crtc_state, conn_state);
            if (ret)
                return ret;
        }
    }
    return 0;
}

static void my_encoder_atomic_enable(struct drm_encoder *encoder,
                                      struct drm_atomic_state *state)
{
    // 上电 PHY，配置信号输出
    my_phy_power_on(encoder);
    my_encoder_configure_timing(encoder);
}
```

### 带 Bridge Chain 的 Encoder

```c
// 初始化时构建 Bridge Chain
static int my_driver_bind(struct device *dev)
{
    struct drm_encoder *encoder = &my_dev->encoder;
    struct drm_bridge *panel_bridge, *phy_bridge;

    panel_bridge = devm_drm_of_get_bridge(dev, node, 0, 0);
    phy_bridge = my_phy_bridge_create(dev);

    // Bridge Chain: Encoder → PHY Bridge → Panel Bridge
    drm_bridge_attach(encoder, phy_bridge, NULL, 0);
    drm_bridge_attach(encoder, panel_bridge, phy_bridge, 0);
}
```

---

## 潜在影响

- **Bridge 标准化：** 社区持续推动 Encoder 逻辑向 Bridge 迁移，新驱动推荐使用 Bridge 架构而非直接实现 Encoder 回调。这使得 PHY/Panel/TX 逻辑可复用。
- **MST 生态：** DP MST 是多显示器 Dock 的基础，虚拟 Encoder 的正确实现直接影响 USB-C Dock 的使用体验。
- **总线格式传播：** Bridge atomic state 的 `input_bus_cfg`/`output_bus_cfg` 是确保多级 Bridge 之间格式兼容的关键机制。
- **演进方向：** v6.x 中新增了 Bridge 的 `atomic_get_output_bus_fmt`/`atomic_get_input_bus_fmt` 回调，进一步标准化了总线格式协商。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm_encoder.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_encoder.h)
- [drm_bridge.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_bridge.h)
- [A Current Overview of the DRM KMS Driver-Side APIs — Bootlin 2023](https://bootlin.com/pub/conferences/2023/eoss/kocialkowski-current-overview-drm-kms-driver-side-apis/kocialkowski-current-overview-drm-kms-driver-side-apis.pdf)
