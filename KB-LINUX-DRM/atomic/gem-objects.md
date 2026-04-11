# DRM GEM Buffer Object

> **条目ID:** KB-LINUX-DRM-atomic-012
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [framebuffer](./framebuffer.md) | [dma-buf-overview](./dma-buf-overview.md) | [plane-state](./plane-state.md)

---

## 核心摘要

GEM（Graphics Execution Manager）是 Linux DRM 子系统的图形内存管理框架——它提供了 Buffer Object 的创建、销毁、映射、共享和生命周期管理。`struct drm_gem_object` 是所有图形缓冲区的基类，驱动通过嵌入此结构体来扩展私有数据。GEM 的核心设计哲学是：**将 Buffer Object 的创建与内存分配分离**——GEM 对象是逻辑容器，内存分配策略由驱动决定（shmem、contiguous、IOMMU、TTM 等）。PRIME 机制基于 DMA-BUF 实现跨设备、跨进程的零拷贝缓冲区共享。

---

## 1. GEM 对象核心结构

### `struct drm_gem_object` 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `refcount` | `struct kref` | 引用计数 |
| `dev` | `struct drm_device *` | 所属 DRM 设备 |
| `filp` | `struct file *` | shmem backing file（匿名可分页内存） |
| `size` | `size_t` | 缓冲区大小（字节） |
| `dev_name` | `char *` | 设备名称（调试用） |
| `name` | `char [32]` | 对象名称（调试用） |
| `dma_buf` | `struct dma_buf *` | 关联的 DMA-BUF（PRIME 导出时设置） |
| `import_attach` | `struct dma_buf_attachment *` | PRIME 导入时的 attachment |
| `sg_table` | `struct sg_table *` | Scatter-gather 表（DMA 映射用） |
| `resv` | `struct dma_resv *` | Reservation 互斥锁（fence 管理） |
| `funcs` | `const struct drm_gem_object_funcs *` | 回调函数表 |
| `handle_count` | `int` | 用户空间 handle 引用计数 |
| `access_count` | `int` | 当前 mmap 访问计数 |

### GEM 对象回调函数表

| 回调 | 说明 |
|------|------|
| `free` | **必选。** 释放 GEM 对象及关联资源 |
| `open` | handle 创建时调用 |
| `close` | handle 销毁时调用 |
| `print_info` | 调试信息输出 |
| `pin` | 固定物理页面（不可换出） |
| `unpin` | 解除固定 |
| `get_sg_table` | 获取 scatter-gather 表 |
| `vmap` | 创建内核虚拟映射 |
| `vunmap` | 销毁内核虚拟映射 |
| `mmap` | 实现 mmap 操作 |
| `vm_ops` | 页错误处理 |
| `evict` | TTM 驱动：驱逐回调 |
| `export` | PRIME 导出回调 |

---

## 2. GEM 对象创建与销毁

### 创建流程

```
用户空间 ioctl(GEM_CREATE)
    │
    ├─ 方式1: shmem-backed（UMA 设备）
    │   ├─ 分配 driver-private 结构体
    │   ├─ drm_gem_object_init(dev, obj, size)
    │   │   └─ 创建 shmfs file，设置 obj->filp
    │   ├─ shmem_read_mapping_page_gfp() 分配物理页面
    │   └─ drm_gem_handle_create(file, obj, &handle)
    │
    ├─ 方式2: private GEM（嵌入式设备）
    │   ├─ 分配 driver-private 结构体
    │   ├─ drm_gem_private_object_init(dev, obj, size)
    │   │   └─ 不创建 shmfs file
    │   ├─ 驱动自行管理物理内存（DMA coherent / CMA / IOMMU）
    │   └─ drm_gem_handle_create(file, obj, &handle)
    │
    └─ 方式3: TTM-backed（独立 GPU）
        ├─ ttm_bo_create() 创建 TTM Buffer Object
        ├─ TTM 管理 VRAM/GTT/SysRAM 之间的迁移
        └─ drm_gem_handle_create(file, obj, &handle)
```

### 销毁流程

```
drm_gem_object_put(obj)
    │
    ├─ 引用计数 -1
    ├─ 若归零 → 调用 obj->funcs->free(obj)
    │   ├─ 驱动释放私有资源
    │   ├─ drm_gem_object_release(obj)
    │   │   ├─ 释放 mmap offset
    │   │   ├─ 释放 dma_buf（如果有）
    │   │   ├─ 释放 sg_table
    │   │   └─ 释放 shmem filp
    │   └─ kfree(obj)
    └─ 若未归零 → 继续存活
```

---

## 3. GEM 对象命名与引用

### 三种引用方式

| 方式 | 作用域 | 安全性 | 状态 | 说明 |
|------|--------|--------|------|------|
| **Handle** | per-fd | 中 | 推荐 | 32位整数，fd 关闭时自动释放 |
| **Global Name** | 全局 | 低 | 已废弃 | 可猜测，不安全 |
| **DMA-BUF fd** | 跨设备/进程 | 高 | 推荐 | 基于 fd 传递，安全且支持跨设备 |

### Handle 生命周期

```c
// 创建 handle
int drm_gem_handle_create(struct drm_file *file_priv,
                          struct drm_gem_object *obj,
                          u32 *handle);

// 通过 handle 查找 GEM 对象
struct drm_gem_object *drm_gem_object_lookup(struct drm_file *file_priv,
                                              u32 handle);

// 删除 handle（不释放 GEM 对象，仅减引用）
void drm_gem_handle_delete(struct drm_file *file_priv, u32 handle);
```

**关键规则：** Handle 不拥有 GEM 对象的所有权，仅持有一个引用。驱动在创建 handle 后必须释放自己的初始引用，否则会泄漏。

---

## 4. GEM 对象映射

### mmap 机制

```
用户空间 mmap(fd, offset)
    │
    ├─ offset 是 fake offset（通过 drm_gem_create_mmap_offset 创建）
    ├─ drm_gem_mmap() 处理
    │   ├─ 通过 offset 查找 GEM 对象
    │   ├─ 调用 obj->funcs->mmap() 或 vmap()
    │   └─ 建立 VMA → 物理页面的映射
    └─ 用户空间获得可访问的虚拟地址
```

### CPU 访问方式

| 方式 | API | 适用场景 |
|------|-----|----------|
| mmap | `drm_gem_mmap()` | 随机访问（软件渲染） |
| read/write | driver-specific ioctl | 顺序访问（小缓冲区） |
| kmap | `drm_gem_vmap()` | 内核态访问 |
| pwrite/pread | `DRM_IOCTL_GEM_PREAD/PWRITE` | Legacy 兼容 |

---

## 5. PRIME 缓冲区共享

### 导出流程

```
DRM_IOCTL_PRIME_HANDLE_TO_FD(handle, flags)
    │
    ├─ drm_gem_prime_handle_to_fd()
    │   ├─ 查找 GEM 对象
    │   ├─ 若已导出 → 增加 dma_buf 引用，返回已有 fd
    │   ├─ 若未导出 → 调用 obj->funcs->export()
    │   │   └─ drm_gem_dmabuf_export()
    │   │       ├─ 创建 dma_buf
    │   │       ├─ 设置 dma_buf->ops
    │   │       └─ 关联到 obj->dma_buf
    │   └─ 返回 dma_buf fd
    └─ 用户空间获得可传递的 fd
```

### 导入流程

```
DRM_IOCTL_PRIME_FD_TO_HANDLE(fd, flags)
    │
    ├─ drm_gem_prime_fd_to_handle()
    │   ├─ dma_buf_get(fd) 获取 dma_buf
    │   ├─ 检查是否已导入（避免重复导入）
    │   ├─ 调用 driver import callback
    │   │   └─ drm_gem_prime_import_dev()
    │   │       ├─ 检查同设备导出 → 直接使用原 GEM 对象
    │   │       └─ 跨设备 → 创建新 GEM 对象 + dma_buf_attachment
    │   └─ 创建 handle
    └─ 用户空间获得本地 handle
```

### 同设备优化

```
GPU-A 导出 → GPU-A 导入（同设备）
    └─ 直接返回原 GEM 对象，零拷贝，零额外开销

GPU-A 导出 → GPU-B 导入（跨设备）
    └─ 创建新 GEM 对象 + dma_buf_attachment + sg_table
    └─ DMA 映射到 GPU-B 的地址空间
```

---

## 6. GEM vs TTM

| 维度 | GEM | TTM |
|------|-----|-----|
| 设计哲学 | 轻量级支持库 | 完整内存管理器 |
| 适用设备 | UMA（集成 GPU） | 独立 GPU（有 VRAM） |
| VRAM 管理 | ❌ 不支持 | ✅ 支持 |
| 内存迁移 | ❌ | ✅ VRAM↔GTT↔SysRAM |
| 页面驱逐 | ❌ | ✅ LRU 驱动 |
| 复杂度 | 低 | 高 |
| 典型用户 | rockchip, vc4, simpledrm | amdgpu, nouveau, i915 |
| 关系 | 可独立使用 | 内部使用 GEM 作为对象层 |

**现代趋势：** 大多数驱动同时使用 GEM（对象管理）+ TTM（内存管理），GEM 作为 TTM 的上层抽象。

---

## 7. Fence 与 Reservation

### `struct dma_resv`

```c
struct dma_resv {
    struct ww_mutex lock;          // 互斥锁
    struct dma_resv_list *fences;  // fence 列表
};
```

- 每个 GEM 对象有一个 `resv` 字段
- 用于管理对该对象的并发访问
- 支持 shared fence（读）和 exclusive fence（写）
- 是 DRM Sync Object 的底层实现

### Fence 使用模式

```c
// 写操作：添加 exclusive fence
dma_resv_reserve_fences(obj->resv, 1);
dma_resv_add_fence(obj->resv, write_fence, DMA_RESV_USAGE_WRITE);

// 读操作：添加 shared fence
dma_resv_add_fence(obj->resv, read_fence, DMA_RESV_USAGE_READ);

// 等待所有 fence 完成
dma_resv_wait_timeout(obj->resv, DMA_RESV_USAGE_READ, false, MAX_SCHEDULE_TIMEOUT);
```

---

## 8. Contiguous Memory Allocator (CMA)

嵌入式设备常用 CMA 分配物理连续内存：

```c
struct drm_gem_cma_object {
    struct drm_gem_object base;
    struct sg_table *sgt;
    void *vaddr;
    dma_addr_t paddr;
};
```

- `drm_gem_cma_create()` — 从 CMA 区域分配
- `paddr` 可直接用于没有 IOMMU 的 DMA 引擎
- 典型用户：rockchip, stm32, nxp

---

## 潜在影响

- **零拷贝管线：** PRIME + DMA-BUF 是 Linux 多媒体管线（PipeWire、GStreamer、Chrome）零拷贝的基础。GPU 渲染 → V4L2 编码 → 网络发送的全程零拷贝依赖此机制。
- **内存管理演进：** TTM 作为独立 GPU 的内存管理器持续演进，v6.x 中新增了 cgroup 内存 accounting、改进的页面驱逐策略等。
- **安全隔离：** Global Name 机制因可猜测性被废弃，DMA-BUF fd 成为唯一推荐的跨进程共享方式，提升了安全性。
- **演进方向：** 社区在 v6.x 中持续优化 GEM/TTM 的交互，推动 `drm_gem_object_funcs` 回调表的标准化，减少驱动私有实现。

---

## 信源

- [DRM Memory Management — kernel.org](https://docs.kernel.org/gpu/drm-mm.html)
- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [drm_gem.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_gem.h)
- [drm_prime.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_prime.c)
