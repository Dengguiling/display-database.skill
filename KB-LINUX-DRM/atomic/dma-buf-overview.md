# DMA-BUF 跨设备缓冲区共享

> **条目ID:** KB-LINUX-DRM-atomic-013
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [gem-objects](./gem-objects.md) | [framebuffer](./framebuffer.md) | [plane-state](./plane-state.md)

---

## 核心摘要

DMA-BUF 是 Linux 内核的**跨设备缓冲区共享框架**——它提供了一套标准化的 API，允许不同设备驱动（GPU、V4L2、ISP、DMA 引擎等）之间零拷贝共享内存缓冲区。DMA-BUF 由三大原语组成：**dma-buf**（缓冲区容器，以文件描述符形式暴露给用户空间）、**dma-fence**（异步操作完成信号）、**dma-resv**（Reservation 锁，管理 fence 集合实现隐式同步）。这套框架是 Linux 多媒体管线（PipeWire、GStreamer、Chrome GPU 进程）和 Wayland compositor 的基础设施。

---

## 1. 三大原语

### dma-buf — 缓冲区容器

| 特性 | 说明 |
|------|------|
| 本质 | 封装 `sg_table`（Scatter-Gather 表）的内核对象 |
| 用户空间接口 | 文件描述符（FD），可通过 Unix Socket、SCM_RIGHTS 传递 |
| 生命周期 | 引用计数管理，FD 关闭 + 内核引用归零时释放 |
| 大小查询 | `lseek(fd, 0, SEEK_END)` |
| CPU 访问 | `mmap()` + `DMA_BUF_IOCTL_SYNC` |
| 轮询 | `poll()` 支持，等待 implicit fence 信号 |
| 安全 | `O_CLOEXEC` 防止 exec 泄露 |

### dma-fence — 异步完成信号

| 特性 | 说明 |
|------|------|
| 本质 | 一次性信号原语（不可重置） |
| 语义 | 表示"某个硬件操作已完成" |
| 等待 | `dma_fence_wait()` 阻塞等待，`dma_fence_add_callback()` 异步回调 |
| 上下文 | `dma_fence_context_alloc()` 分配唯一上下文，保证同上下文内序号递增 |
| 错误 | `dma_fence_set_error()` 标记操作失败 |
| 超时 | `dma_fence_wait_timeout()` 支持超时等待 |

### dma-resv — Reservation 锁

| 特性 | 说明 |
|------|------|
| 本质 | 读写信号量 + fence 列表管理器 |
| 作用 | 管理一个 dma-buf 上的所有 pending fence |
| Shared Fence | 多个只读操作可并行（如多个 GPU 同时读取） |
| Exclusive Fence | 写操作独占（如 GPU 写入 + 显示扫描） |
| 用法 | `dma_resv_reserve_fences()` → `dma_resv_add_fence()` → `dma_resv_wait_timeout()` |

---

## 2. Exporter/Importer 模型

### 角色定义

```
┌─────────────┐     dma-buf FD     ┌─────────────┐
│   Exporter   │ ─────────────────→ │  Importer    │
│  (GPU 驱动)  │                    │ (V4L2/ISP)  │
│              │                    │              │
│ 创建 buffer  │                    │ attach       │
│ 实现 ops     │                    │ map/unmap    │
│ 管理 backing │                    │ DMA 读写     │
│ storage      │                    │ detach       │
└─────────────┘                    └─────────────┘
```

### Exporter 职责

1. 分配 backing storage（物理内存/VRAM）
2. 实现 `struct dma_buf_ops` 回调
3. 通过 `dma_buf_export()` 导出为 dma-buf
4. 通过 `dma_buf_fd()` 转换为 FD

### Importer 职责

1. 通过 FD 获取 dma-buf：`dma_buf_get()`
2. 附加到设备：`dma_buf_attach()`
3. 映射到设备地址空间：`dma_buf_map_attachment()`
4. 执行 DMA 读写
5. 解除映射：`dma_buf_unmap_attachment()`
6. 分离：`dma_buf_detach()`
7. 释放引用：`dma_buf_put()`

---

## 3. dma_buf_ops 回调表

| 回调 | 必选 | 说明 |
|------|------|------|
| `attach` | 是 | Importer 附加时调用，返回 `dma_buf_attachment` |
| `detach` | 是 | Importer 分离时调用 |
| `map_dma_buf` | 是 | 将 buffer 映射到 Importer 的 DMA 地址空间，返回 `sg_table` |
| `unmap_dma_buf` | 是 | 解除 DMA 映射 |
| `release` | 是 | dma-buf 引用归零时调用 |
| `begin_cpu_access` | 否 | CPU 访问前调用（处理 cache coherency） |
| `end_cpu_access` | 否 | CPU 访问结束 |
| `mmap` | 否 | 支持 `mmap()` 系统调用 |
| `vmap` | 否 | 内核虚拟地址映射 |
| `vunmap` | 否 | 解除内核虚拟映射 |
| `map_atomic` | 否 | 原子映射（旧接口，已废弃） |
| `unmap_atomic` | 否 | 解除原子映射（旧接口，已废弃） |
| `pin` | 否 | 固定 backing storage（防止迁移） |
| `unpin` | 否 | 解除固定 |
| `enumerate_shared_pages` | 否 | 枚举共享页面（用于 RDMA） |

---

## 4. 典型使用流程

### 设备间 DMA 共享

```c
// === Exporter 端 ===
DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
exp_info.ops = &my_exporter_ops;
exp_info.size = buffer_size;
exp_info.priv = my_private_buffer;
exp_info.flags = O_CLOEXEC;

struct dma_buf *dmabuf = dma_buf_export(&exp_info);
int fd = dma_buf_fd(dmabuf, O_CLOEXEC);

// === 用户空间传递 FD ===
// 通过 Unix Socket SCM_RIGHTS 传递给另一个进程

// === Importer 端 ===
struct dma_buf *dmabuf = dma_buf_get(fd);
struct dma_buf_attachment *attach = dma_buf_attach(dmabuf, my_device);
struct sg_table *sgt = dma_buf_map_attachment(attach, DMA_BIDIRECTIONAL);

// 使用 sgt 进行 DMA 传输...

dma_buf_unmap_attachment(attach, sgt, DMA_BIDIRECTIONAL);
dma_buf_detach(dmabuf, attach);
dma_buf_put(dmabuf);
```

### CPU 访问流程

```c
// 用户空间
void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

// 每次访问前同步 cache
struct dma_buf_sync sync = { .flags = DMA_BUF_SYNC_START | DMA_BUF_SYNC_RW };
ioctl(fd, DMA_BUF_IOCTL_SYNC, &sync);

// 读写 ptr...

sync.flags = DMA_BUF_SYNC_END | DMA_BUF_SYNC_RW;
ioctl(fd, DMA_BUF_IOCTL_SYNC, &sync);

munmap(ptr, size);
```

### 内核 vmap 访问

```c
struct iosys_map map;
void *ptr = dma_buf_vmap(dmabuf, &map);
if (IS_ERR(ptr)) { /* handle error */ }

// 读写 ptr...

dma_buf_vunmap(dmabuf, &map);
```

---

## 5. 隐式同步 vs 显式同步

### 隐式同步（Implicit Sync）

```
GPU 渲染完成 ──→ 添加 fence 到 dma-resv ──→ 显示引擎自动等待 fence
```

- **机制：** 每次写入操作完成后，将 fence 添加到 dma-buf 的 `dma_resv` 中
- **优点：** 用户空间无需管理同步，内核自动保证顺序
- **缺点：** 过度同步，可能导致不必要的等待
- **现状：** DRM 默认模式，Android 也使用此模式

### 显式同步（Explicit Sync）

```
用户空间获取 fence FD ──→ 传递给下一个消费者 ──→ 消费者等待 fence
```

- **机制：** 用户空间通过 `sync_file` FD 显式传递 fence
- **优点：** 精细控制同步，减少不必要的等待
- **缺点：** 用户空间复杂度增加
- **现状：** Wayland (linux-dmabuf v4)、Vulkan、Chrome 正在迁移到此模式
- **趋势：** 社区正在推动从隐式同步向显式同步迁移

### DRM Sync Object

```c
// 创建 sync object
uint32_t handle;
drmIoctl(fd, DRM_IOCTL_SYNCOBJ_CREATE, &create);

// 将 fence 导入 sync object
drmIoctl(fd, DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE, &fd_to_handle);

// 提交时等待 sync object
struct drm_syncobj_wait wait = { .handles = &handle, .count_handles = 1 };
drmIoctl(fd, DRM_IOCTL_SYNCOBJ_WAIT, &wait);

// 将 sync object 转为 fence FD 传递给另一个进程
drmIoctl(fd, DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD, &handle_to_fd);
```

---

## 6. DMA-BUF Heap（用户空间分配）

v5.4+ 引入 DMA-BUF Heap 框架，允许用户空间直接分配 DMA 缓冲区：

```c
// 打开 heap
int heap_fd = open("/dev/dma_heap/system", O_RDWR);
int heap_fd = open("/dev/dma_heap/cma", O_RDWR);

// 分配
struct dma_heap_allocation_data alloc = {
    .len = size,
    .fd_flags = O_CLOEXEC | O_RDWR,
};
ioctl(heap_fd, DMA_HEAP_IOCTL_ALLOC, &alloc);
// alloc.fd 是 dma-buf FD
```

### 内置 Heap 类型

| Heap | 说明 | 典型用途 |
|------|------|----------|
| `system` | 页表映射内存（可 swap） | 通用 |
| `system-uncached` | 无 cache 页表映射 | DMA 引擎 |
| `cma` | 物理连续内存 | 嵌入式显示/视频 |
| 自定义 | 驱动注册（如 `qcom,system`） | SoC 特定 |

---

## 7. 性能考量

### 零拷贝路径

```
GPU 渲染 ──→ dma-buf ──→ V4L2 编码 ──→ 网络发送
           (零拷贝)       (零拷贝)
```

- 无需 CPU 拷贝，全程 DMA 传输
- 典型带宽节省：4K@60fps NV12 = ~1.5 GB/s 的拷贝消除

### Cache Coherency 开销

- `DMA_BUF_IOCTL_SYNC` 可能涉及 cache flush/invalidate
- ARM Mali GPU + CPU 共享时，sync 开销可达数毫秒
- **优化：** 尽量使用显式同步，减少不必要的 sync 调用

### 内存迁移

- `pin()` 回调允许 exporter 在 attach 阶段迁移 backing storage
- TTM 支持在 VRAM ↔ GTT ↔ System RAM 之间迁移
- 迁移开销可能抵消零拷贝的收益

---

## 潜在影响

- **多媒体管线：** DMA-BUF 是 PipeWire、GStreamer、Chrome GPU 进程间零拷贝共享的基础。没有 DMA-BUF，每帧需要 1-2 次 CPU 拷贝，4K@60fps 场景下是不可接受的。
- **显式同步迁移：** Wayland 协议正在从隐式同步向显式同步迁移（linux-dmabuf v4），这是 Linux 图形栈的重大架构变更，影响所有 GPU 驱动和 compositor。
- **Android 兼容：** Android 的 ION 分配器已被 DMA-BUF Heap 替代，但隐式同步仍是 Android 的默认模式。
- **演进方向：** 社区在 v6.x 中持续完善 DMA-BUF 的安全性和性能，包括 DMA-BUF layout（v6.11+）用于描述非线性布局、改进的 fence signaling 机制等。

---

## 信源

- [Buffer Sharing and Synchronization (dma-buf) — kernel.org](https://docs.kernel.org/driver-api/dma-buf.html)
- [DRM Memory Management — kernel.org](https://docs.kernel.org/gpu/drm-mm.html)
- [dma-buf.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/linux/dma-buf.h)
- [dma-fence.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/linux/dma-fence.h)
- [dma-resv.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/linux/dma-resv.h)
