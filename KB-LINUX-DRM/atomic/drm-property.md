# DRM 属性系统

> **条目ID:** KB-LINUX-DRM-atomic-011
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [state-mgmt](./state-mgmt.md) | [plane-state](./plane-state.md) | [crtc-state](./crtc-state.md) | [connector-state](./connector-state.md)

---

## 核心摘要

DRM 属性系统（Property System）是 Atomic Mode Setting 的**元数据基础设施**——它定义了用户空间可查询和修改的 KMS 对象状态参数。每个属性具有类型（Range/Enum/Blob/Object/Bitmask）、取值范围、名称和标志位。Atomic Commit 本质上就是"将一组属性值打包成 atomic state，校验后原子化提交"。属性系统是 DRM 用户空间 API（libdrm/Mesa/Wayland compositor）与内核驱动之间的契约接口。

---

## 1. 属性类型体系

### 六大属性类型

| 类型 | 标志位 | 说明 | 创建函数 | 典型用途 |
|------|--------|------|----------|----------|
| **Range** | `DRM_MODE_PROP_RANGE` | 无符号整数范围 | `drm_property_create_range()` | Brightness, Contrast, Alpha |
| **Signed Range** | `DRM_MODE_PROP_SIGNED_RANGE` | 有符号整数范围 | `drm_property_create_signed_range()` | Hue, Saturation |
| **Enum** | `DRM_MODE_PROP_ENUM` | 枚举值（0~N-1） | `drm_property_create_enum()` | Rotation, Blend Mode |
| **Bitmask** | `DRM_MODE_PROP_BITMASK` | 位掩码（0~63 位） | `drm_property_create_bitmask()` | Blending, Color Range |
| **Object** | `DRM_MODE_PROP_OBJECT` | KMS 对象引用 | `drm_property_create_object()` | CRTC ID, FB ID, Plane ID |
| **Blob** | `DRM_MODE_PROP_BLOB` | 二进制数据块 | `drm_property_create()` | EDID, Gamma LUT, HDR Metadata |

### 附加标志位

| 标志位 | 说明 |
|--------|------|
| `DRM_MODE_PROP_ATOMIC` | 仅 Atomic 驱动使用，不暴露给 Legacy 用户空间 |
| `DRM_MODE_PROP_IMMUTABLE` | 用户空间只读，内核可更新（如 EDID、Connector Path） |
| `DRM_MODE_PROP_EXTENDED_TYPE` | 使用扩展类型编码（v5.10+，解决类型标志位冲突） |

---

## 2. 核心数据结构

### `struct drm_property`

```c
struct drm_property {
    struct list_head head;          // per-device 属性链表
    struct drm_mode_object base;    // 基类对象（ID、类型）
    uint32_t flags;                 // 类型 + 标志位
    char name[DRM_PROP_NAME_LEN];   // 属性名称（唯一标识）
    uint32_t num_values;            // values 数组大小
    uint64_t *values;               // 范围限制值
    struct list_head enum_list;     // 枚举值名称列表
    struct drm_device *dev;         // 所属设备
};
```

### `struct drm_property_blob`

```c
struct drm_property_blob {
    struct drm_mode_object base;    // 基类对象
    struct drm_device *dev;
    struct list_head head_global;   // 全局 blob 列表
    struct list_head head_file;     // per-fd blob 列表
    size_t length;                  // 数据长度（字节）
    void *data;                     // 数据指针（内嵌在结构体末尾）
};
```

### `struct drm_prop_enum_list`

```c
struct drm_prop_enum_list {
    int type;       // 枚举数值
    const char *name; // 枚举名称
};
```

---

## 3. 属性创建 API

### Range 属性

```c
// 无符号范围
struct drm_property *drm_property_create_range(
    struct drm_device *dev, int flags,
    const char *name, uint64_t min, uint64_t max);

// 示例：创建 Alpha 属性（0~65535）
plane->alpha_property = drm_property_create_range(dev, 0,
    "alpha", 0, DRM_BLEND_ALPHA_OPAQUE);
```

### Enum 属性

```c
struct drm_property *drm_property_create_enum(
    struct drm_device *dev, int flags,
    const char *name,
    const struct drm_prop_enum_list *props, int num_props);

// 示例：创建旋转属性
static const struct drm_prop_enum_list rotation_props[] = {
    { DRM_MODE_ROTATE_0,   "rotate-0" },
    { DRM_MODE_ROTATE_90,  "rotate-90" },
    { DRM_MODE_ROTATE_180, "rotate-180" },
    { DRM_MODE_ROTATE_270, "rotate-270" },
    { DRM_MODE_REFLECT_X,  "reflect-x" },
    { DRM_MODE_REFLECT_Y,  "reflect-y" },
};
plane->rotation_property = drm_property_create_enum(dev,
    DRM_MODE_PROP_ATOMIC, "rotation", rotation_props, 6);
```

### Blob 属性

```c
// 创建 Blob 属性定义
struct drm_property *drm_property_create(
    struct drm_device *dev, int flags,
    const char *name, int num_values, ...);

// 创建 Blob 数据实例
struct drm_property_blob *drm_property_create_blob(
    struct drm_device *dev, size_t length, const void *data);

// 示例：创建 Gamma LUT 属性
crtc->gamma_lut_property = drm_property_create(dev,
    DRM_MODE_PROP_BLOB | DRM_MODE_PROP_ATOMIC,
    "GAMMA_LUT", 0);
```

### Object 属性

```c
struct drm_property *drm_property_create_object(
    struct drm_device *dev, int flags,
    const char *name, uint32_t obj_type);

// 示例：创建 CRTC ID 属性
connector->crtc_id_property = drm_property_create_object(dev,
    DRM_MODE_PROP_ATOMIC, "CRTC_ID", DRM_MODE_OBJECT_CRTC);
```

---

## 4. 标准属性一览

### Plane 标准属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `type` | Enum | Plane 类型（Primary/Overlay/Cursor） |
| `SRC_X`, `SRC_Y` | Range | 源矩形坐标（16.16 定点数） |
| `SRC_W`, `SRC_H` | Range | 源矩形尺寸（16.16 定点数） |
| `CRTC_X`, `CRTC_Y` | Range | 目标矩形坐标 |
| `CRTC_W`, `CRTC_H` | Range | 目标矩形尺寸 |
| `FB_ID` | Object | Framebuffer 引用 |
| `CRTC_ID` | Object | CRTC 引用 |
| `IN_FENCE_FD` | Signed Range | 输入 fence 文件描述符 |
| `rotation` | Bitmask | 旋转/反射 |
| `zpos` | Range | Z 轴优先级 |
| `alpha` | Range | 不透明度 |
| `pixel_blend_mode` | Enum | Alpha 混合模式 |
| `color_encoding` | Enum | 颜色编码（BT.601/709/2020） |
| `color_range` | Enum | 颜色范围（全/有限） |
| `scaling_filter` | Enum | 缩放滤波器 |

### CRTC 标准属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `ACTIVE` | Range | CRTC 是否活跃 |
| `MODE_ID` | Blob | 显示模式（`drm_mode_modeinfo`） |
| `OUT_FENCE_PTR` | Range | 输出 fence 指针（用户空间地址） |
| `VRR_ENABLED` | Range | 可变刷新率 |
| `GAMMA_LUT` | Blob | Gamma 查找表 |
| `GAMMA_LUT_SIZE` | Immutable Range | Gamma LUT 大小 |
| `DEGAMMA_LUT` | Blob | Degamma 查找表 |
| `DEGAMMA_LUT_SIZE` | Immutable Range | Degamma LUT 大小 |
| `CTM` | Blob | 颜色转换矩阵 |
| `SCALING_FILTER` | Enum | CRTC 缩放滤波器 |

### Connector 标准属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `CRTC_ID` | Object | 绑定的 CRTC |
| `EDID` | Immutable Blob | EDID 数据 |
| `DPMS` | Enum | 电源管理状态 |
| `link-status` | Enum | 链路状态（Good/Bad） |
| `content-protection` | Enum | HDCP 状态 |
| `HDR_OUTPUT_METADATA` | Blob | HDR 静态元数据 |
| `max_bpc` | Range | 最大位深 |
| `colorspace` | Enum | 颜色空间 |
| `content-type` | Enum | 内容类型（游戏/电影/照片/文字） |
| `scaling_mode` | Enum | 缩放模式 |
| `panel_orientation` | Immutable Enum | 面板方向 |
| `WRITEBACK_FB_ID` | Object | Writeback Framebuffer |
| `WRITEBACK_OUT_FENCE_PTR` | Range | Writeback 输出 fence |

---

## 5. 属性与 Atomic State 的关系

### 属性 → State 映射

```
用户空间 Atomic IOCTL
    │
    ├─ 属性 ID → drm_property 对象
    ├─ 属性值 → atomic_state 中的对应字段
    │
    ├─ Plane 属性 → drm_plane_state 字段
    │   ├─ FB_ID → plane_state->fb
    │   ├─ CRTC_ID → plane_state->crtc
    │   ├─ SRC_X/Y/W/H → plane_state->src_x/y/w/h
    │   ├─ CRTC_X/Y/W/H → plane_state->crtc_x/y/w/h
    │   ├─ rotation → plane_state->rotation
    │   ├─ alpha → plane_state->alpha
    │   └─ zpos → plane_state->zpos
    │
    ├─ CRTC 属性 → drm_crtc_state 字段
    │   ├─ MODE_ID → crtc_state->mode
    │   ├─ ACTIVE → crtc_state->active
    │   ├─ GAMMA_LUT → crtc_state->gamma_lut
    │   ├─ CTM → crtc_state->ctm
    │   └─ VRR_ENABLED → crtc_state->vrr_enabled
    │
    └─ Connector 属性 → drm_connector_state 字段
        ├─ CRTC_ID → conn_state->crtc
        ├─ content-protection → conn_state->content_protection
        ├─ HDR_OUTPUT_METADATA → conn_state->hdr_output_metadata
        └─ max_bpc → conn_state->max_requested_bpc
```

### Blob 属性的特殊处理

Blob 属性不直接存储 64 位值，而是引用一个 `drm_property_blob` 对象：

```c
// 用户空间创建 Blob
struct drm_mode_create_blob blob_req = {
    .length = sizeof(hdr_metadata),
    .data = (uintptr_t)&hdr_metadata,
};
drmIoctl(fd, DRM_IOCTL_MODE_CREATEPROPBLOB, &blob_req);
// blob_req.blob_id → 用于 atomic commit

// 内核侧读取
blob = drm_property_lookup_blob(dev, state->hdr_output_metadata);
if (blob) {
    hdr_info = (struct hdr_output_metadata *)blob->data;
}
```

---

## 6. 属性生命周期

### 创建阶段（驱动初始化）

```c
int my_driver_modeset_init(struct drm_device *dev)
{
    // 1. 创建属性
    dev->mode_config.rotation_property =
        drm_property_create_enum(dev, DRM_MODE_PROP_ATOMIC,
            "rotation", rotation_props, 6);

    // 2. 附加到 KMS 对象
    drm_object_attach_property(&plane->base,
        dev->mode_config.rotation_property, 0);

    // 3. 设置默认值
    drm_object_property_set_value(&plane->base,
        dev->mode_config.rotation_property, DRM_MODE_ROTATE_0);
}
```

### 查询阶段（用户空间）

```c
// 枚举所有属性
drmModeObjectGetProperties(fd, object_id, object_type);
// → 返回属性 ID 数组和值数组

// 查询属性详情
drmModeGetProperty(fd, prop_id);
// → 返回名称、类型、范围、枚举值列表
```

### 销毁阶段

```c
// 属性随 DRM 设备销毁自动清理
// drm_mode_config_cleanup() 遍历属性链表释放
// Blob 引用计数归零时自动释放
```

---

## 7. 扩展类型机制

v5.10+ 引入扩展类型，解决类型标志位冲突问题：

```c
// 旧方式（类型标志位可能冲突）
flags = DRM_MODE_PROP_RANGE | DRM_MODE_PROP_ATOMIC;

// 新方式（使用扩展类型字段）
flags = DRM_MODE_PROP_ATOMIC | DRM_MODE_PROP_EXTENDED_TYPE;
// 实际类型存储在扩展类型字段中
```

`drm_property_type_is()` 函数自动处理两种编码：

```c
static inline bool drm_property_type_is(struct drm_property *property,
                                        uint32_t type)
{
    if (property->flags & DRM_MODE_PROP_EXTENDED_TYPE)
        return (property->flags & DRM_MODE_PROP_EXTENDED_TYPE) == type;
    return property->flags & type;
}
```

---

## 潜在影响

- **API 契约：** 属性系统是 DRM 用户空间 API 的核心契约。新增属性需要 libdrm、Mesa、Wayland compositor 的同步适配，变更成本高。
- **向后兼容：** `DRM_MODE_PROP_ATOMIC` 标志确保新属性不会暴露给 Legacy 用户空间，保证了 API 的向后兼容性。
- **Blob 属性：** HDR 元数据、Gamma LUT 等 Blob 属性是 HDR 显示管线的基础，其数据格式需要用户空间和内核的严格对齐。
- **演进方向：** 社区持续标准化 Connector 属性（如 HDMI Infoframe、DP DSC），减少驱动私有属性，提升跨驱动一致性。

---

## 信源

- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [drm_property.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_property.h)
- [drm_property.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_property.c)
