# KB-LINUX-DRM-driver-001 | DRM 驱动模型

## Meta

| 字段 | 值 |
|------|-----|
| 条目ID | KB-LINUX-DRM-driver-001 |
| 子模块 | driver |
| 核心主题 | DRM 驱动注册、struct drm_driver、设备实例生命周期、managed resource |
| 置信度 | 5 |
| 信源等级 | P0×4 + P1×1 |
| 内核版本 | v6.6+（当前主线） |
| 验证日期 | 2026-04-14 |
| 前置知识 | 无（本条目为 Linux DRM 入口） |
| 关联条目 | [→ atomic/commit-flow]（atomic commit 依赖 drm_device 注册） |

## 概述

DRM 驱动模型是 Linux 图形子系统的核心框架。每个 DRM 驱动通过 `struct drm_driver` 描述自身能力和回调方法，每个物理设备对应一个 `struct drm_device` 实例。现代驱动使用 `devm_drm_dev_alloc()` + `drm_dev_register()` 的两阶段初始化模式，配合 `drmm_*` managed resource API 实现自动资源回收。

## 架构 / 调用链

### 驱动注册 → 设备实例 → 用户空间访问

```
模块加载
  │
  ▼
drm_module_pci_driver() / drm_module_platform_driver()
  │  注册 bus driver（PCI/Platform/...）
  ▼
bus → probe() 回调
  │
  ▼
devm_drm_dev_alloc()          ← 分配 drm_device，嵌入到驱动私有结构体
  │  内部：drm_dev_alloc() + drmm_init()
  ▼
drmm_mode_config_init()       ← 初始化 KMS mode_config
  │
  ▼
drmm_kzalloc() / drmm_mutex_init() / drmm_add_action()  ← 分配 managed 资源
  │
  ▼
drm_mode_config_reset()       ← 重置所有 KMS 对象状态
  │
  ▼
drm_dev_register()            ← 发布设备，创建 /dev/dri/cardN 节点
  │  内部：drm_minor_alloc() + device_register()
  ▼
用户空间可通过 libdrm ioctl 访问
```

### 设备移除（逆序）

```
bus → remove() 回调
  │
  ▼
drm_dev_unregister()          ← 撤销 /dev/dri/cardN
  │
  ▼
drm_dev_put()                 ← 释放最后一个引用
  │  内部：按逆序执行所有 drmm 注册的 cleanup action
  ▼
drm_device 被释放
```

## 关键数据结构

### struct drm_driver（`include/drm/drm_drv.h`）

描述驱动族的静态信息和方法表。一个驱动对应一个 `drm_driver` 实例，多个设备共享。

| 字段 | 类型 | 说明 | 状态 |
|------|------|------|------|
| `driver_features` | `u32` | 特性标志位组合（见下表） | 必填 |
| `name` | `const char *` | 驱动名称，用于日志和 ioctl | 必填 |
| `desc` | `const char *` | 驱动描述（仅信息用途） | 可选 |
| `date` | `const char *` | 驱动日期 | 可选 |
| `major/minor/patchlevel` | `int` | 版本号 | 必填 |
| `open` | 回调 | drm_file 打开时调用 | 推荐 |
| `postclose` | 回调 | drm_file 关闭时调用 | 推荐 |
| `debugfs_init` | 回调 | 创建 debugfs 文件 | 推荐 |
| `gem_create_object` | 回调 | GEM 对象构造器 | GEM 驱动 |
| `dumb_create` | 回调 | 创建 dumb buffer | KMS 驱动 |
| `dumb_map_offset` | 回调 | dumb buffer mmap offset | KMS 驱动 |
| `load` | 回调 | **已废弃** — 旧式初始化 | 禁止新驱动使用 |
| `unload` | 回调 | **已废弃** — 旧式清理 | 禁止新驱动使用 |
| `release` | 回调 | **已废弃** — 用 drmm 替代 | 禁止新驱动使用 |

### enum drm_driver_feature（`include/drm/drm_drv.h`）

| 标志 | Bit | 说明 | 适用场景 |
|------|-----|------|----------|
| `DRIVER_GEM` | 0 | 使用 GEM 内存管理器 | 所有现代驱动 |
| `DRIVER_MODESET` | 1 | 支持 KMS 接口 | 显示驱动 |
| `DRIVER_RENDER` | 3 | 支持独立 render node | GPU 计算/渲染驱动 |
| `DRIVER_ATOMIC` | 4 | 支持完整 atomic userspace API | 现代 KMS 驱动 |
| `DRIVER_SYNCOBJ` | 5 | 支持 drm_syncobj 显式同步 | GPU 驱动 |
| `DRIVER_SYNCOBJ_TIMELINE` | 6 | 支持 timeline syncobj | GPU 驱动 |
| `DRIVER_COMPUTE_ACCEL` | 7 | 计算加速设备（与 RENDER/MODESET 互斥） | AI 加速卡 |
| `DRIVER_GEM_GPUVA` | 8 | 支持用户自定义 GPU VA 绑定 | GPU 驱动 |
| `DRIVER_CURSOR_HOTSPOT` | 9 | 需要 cursor hotspot 信息 | 特定显示驱动 |
| `DRIVER_USE_AGP` | 25 | AGP 支持 | **遗留** |
| `DRIVER_LEGACY` | 26 | 遗留驱动 | **遗留** |

典型组合：
- KMS 显示驱动：`DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- GPU 渲染驱动：`DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_GEM_GPUVA`

### struct drm_device（`include/drm/drm_device.h`）

每个物理设备一个实例，表示一张完整的显卡。

| 字段 | 说明 |
|------|------|
| `dev` | 底层 bus device（PCI/Platform/...） |
| `dma_dev` | DMA 设备（若 dev 本身不能 DMA） |
| `driver` | 指向 `const struct drm_driver` |
| `dev_private` | **已废弃** — 用嵌入方式替代 |
| `primary` | Primary minor node（/dev/dri/cardN） |
| `render` | Render minor node（/dev/dri/renderD128+） |
| `accel` | Compute accel node |
| `driver_features` | 设备级特性覆盖（可清除驱动级标志） |
| `registered` | 是否已调用 drm_dev_register() |
| `unplugged` | 设备是否已拔出 |
| `master` | 当前 master（受 master_mutex 保护） |
| `open_count` | 打开的文件数 |
| `filelist` | 用户空间 drm_file 链表 |
| `managed` | managed resource 链表和 final_kfree 指针 |

## 关键接口

### 设备分配与注册

```c
// 分配 drm_device（嵌入到驱动私有结构体）
struct driver_device {
    struct drm_device drm;
    // ... 驱动私有字段
};

priv = devm_drm_dev_alloc(&pdev->dev, &my_driver,
                          struct driver_device, drm);
// 返回 &priv->drm 对应的 driver_device 指针

// 发布设备（创建 /dev/dri/cardN）
ret = drm_dev_register(drm, 0);

// 撤销设备
drm_dev_unregister(drm);

// 释放引用（触发所有 drmm cleanup）
drm_dev_put(drm);
```

### Managed Resource API（`include/drm/drm_managed.h`）

所有通过 `drmm_*` 分配的资源在 `drm_dev_put()` 时自动释放，释放顺序与注册顺序**相反**。

```c
// 内存分配（自动释放）
ptr = drmm_kzalloc(drm, size, GFP_KERNEL);
str = drmm_kstrdup(drm, "hello", GFP_KERNEL);

// 锁初始化（自动销毁）
drmm_mutex_init(drm, &my_lock);
drmm_spin_lock_init(drm, &my_spin);

// 注册 cleanup 回调
drmm_add_action(drm, my_cleanup, data);
// 或：失败时立即执行 cleanup
drmm_add_action_or_reset(drm, my_cleanup, data);

// 手动提前释放
drmm_kfree(drm, ptr);
drmm_release_action(drm, my_cleanup, data);
```

### drm_file 生命周期回调

```c
// struct drm_driver 中的回调
int (*open)(struct drm_device *dev, struct drm_file *file);
void (*postclose)(struct drm_device *dev, struct drm_file *file);
```

- `open`：用户空间打开 /dev/dri/cardN 时调用，用于初始化驱动私有上下文
- `postclose`：关闭时调用，释放 open 中分配的资源
- **重要**：modeset 资源不应在 open/postclose 中管理（master 机制保证只有一个 owner）

## 数据流

### 用户空间打开设备

```
open("/dev/dri/cardN")
  → drm_open()                     ← DRM core
    → drm_driver.open()            ← 驱动回调（可选）
    → 创建 struct drm_file
    → 加入 drm_device.filelist
```

### 用户空间查询驱动信息

```
ioctl(fd, DRM_IOCTL_VERSION, ...)
  → drm_version()                  ← 返回 name, desc, date, version
```

## 常见陷阱

1. **不要使用 load/unload 回调**：这两个回调已废弃，存在竞态条件。新驱动必须使用 `devm_drm_dev_alloc()` + `drm_dev_register()` 模式。

2. **不要使用 dev_private**：应将 `struct drm_device` 嵌入到驱动私有结构体中，通过 `container_of()` 或 `devm_drm_dev_alloc()` 的返回值获取。

3. **devm_* vs drmm_* 的区别**：
   - `devm_*`：绑定到底层 `struct device` 生命周期，在 bus unbind 时释放
   - `drmm_*`：绑定到 `drm_device` 引用计数，在最后一个引用释放时才清理
   - 对用户空间可见的资源**必须**用 `drmm_*`，因为 unbind 后用户空间可能仍持有 fd

4. **driver_features 的设备级覆盖**：`drm_device.driver_features` 可以清除 `drm_driver.driver_features` 中的标志，用于同一驱动族中不同能力的设备。只能清除，不能新增。

5. **DRIVER_ATOMIC 的语义**：设置此标志意味着驱动支持**完整**的 atomic userspace API。仅内部使用 atomic 但未完全转换所有 property 的驱动不应设置此标志。

6. **drm_dev_enter/exit 保护**：在可能被 unbind 后仍被调用的代码路径中，必须使用 `drm_dev_enter()` / `drm_dev_exit()` 保护，否则会访问已释放的硬件。

## Sources

- **P0** `include/drm/drm_drv.h` — struct drm_driver 定义、enum drm_driver_feature、所有回调注释
- **P0** `include/drm/drm_device.h` — struct drm_device 定义、所有字段注释
- **P0** `include/drm/drm_managed.h` — drmm_add_action/drmm_kzalloc/drmm_mutex_init 等 managed resource API
- **P0** `drivers/gpu/drm/drm_drv.c` — drm_dev_alloc/drm_dev_register/drm_dev_put 实现
- **P1** `Documentation/gpu/drm-internals.rst` — Driver Initialization 章节，完整初始化流程和示例代码
