# GPU 内存层次结构

> GPU 从片上寄存器到片外显存的五级内存层次，延迟差距超三个数量级，是性能优化的核心瓶颈。

## Meta
- **ID:** KB-GPU-ARCH-gpu-pipeline-003
- **置信度:** 5
- **信源等级:** P1
- **验证日期:** 2026-04-13
- **适用版本:** NVIDIA Hopper/Ada, AMD RDNA 3, Intel Xe-HPG, Linux TTM (v6.13)
- **关键词:** GPU memory hierarchy, register file, shared memory, L1 cache, L2 cache, VRAM, HBM, GDDR, TTM, memory coalescing, bank conflict, tiling, page migration
- **前置知识:** [→ gpu-rendering-pipeline] [→ shader-core-arch]
- **关联条目:** [→ shader-core-arch]

## 概述

GPU 内存层次结构从最快的片上寄存器（~0 cycle, ~8 TB/s）到最慢的片外显存（~400-600 cycle, 1-2 TB/s），延迟差距超过三个数量级。现代 GPU 采用多级缓存 + HBM/GDDR 高带宽显存 + TTM 内存管理器的组合架构。理解内存层次对于 GPU 驱动开发（显存分配策略、页面驱逐、DMA 映射）和性能优化（coalescing、bank conflict、tiling）至关重要。

## 架构/调用链

### 五级内存层次

```
线程 (Thread)
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│                    GPU 内存层次结构                            │
│                                                              │
│  Level 0: 寄存器 (Register File)                              │
│  ├── 位置: 片上, per-thread                                   │
│  ├── 大小: 256 KB/SM (NVIDIA), 128 KB/CU (AMD)               │
│  ├── 延迟: ~0 cycle                                           │
│  ├── 带宽: ~8 TB/s                                            │
│  └── 作用域: 线程私有                                          │
│         │                                                    │
│         ▼                                                    │
│  Level 1: 共享内存 / L1 Cache (Shared Memory / L1)            │
│  ├── 位置: 片上, per-SM/CU                                    │
│  ├── 大小: 48-228 KB/SM (可配置)                               │
│  ├── 延迟: ~20-30 cycle                                       │
│  ├── 带宽: ~4 TB/s                                            │
│  └── 作用域: Block/Workgroup 内共享                            │
│         │                                                    │
│         ▼                                                    │
│  Level 2: L2 Cache                                           │
│  ├── 位置: 片上, GPU 全局                                     │
│  ├── 大小: 40-60 MB                                           │
│  ├── 延迟: ~200 cycle                                         │
│  ├── 带宽: ~2-3 TB/s                                          │
│  └── 作用域: GPU 全局缓存                                      │
│         │                                                    │
│         ▼                                                    │
│  Level 3: 显存 (VRAM)                                        │
│  ├── 位置: 片外 (HBM/GDDR)                                    │
│  ├── 大小: 16-80 GB                                           │
│  ├── 延迟: ~400-600 cycle                                     │
│  ├── 带宽: 0.5-3.35 TB/s                                      │
│  └── 作用域: GPU 全局                                          │
│         │                                                    │
│         ▼                                                    │
│  Level 4: 系统内存 (System RAM, via PCIe/CXL)                 │
│  ├── 位置: 主板                                               │
│  ├── 大小: 64-512 GB                                          │
│  ├── 延迟: ~1000+ cycle                                       │
│  ├── 带宽: ~32-64 GB/s (PCIe 5.0 x16)                        │
│  └── 作用域: CPU-GPU 共享                                      │
└──────────────────────────────────────────────────────────────┘
```

### Linux 内核 GPU 内存管理调用链

```
用户空间 (Vulkan/DX12 应用)
    │
    ▼ vkAllocateMemory / CreateCommittedResource
用户空间驱动 (Mesa RADV / NVIDIA UMD)
    │
    ▼ DRM_IOCTL (gem_create / ttm_alloc)
内核 DRM 子系统
    │
    ├── TTM (Translation Table Manager)
    │   ├── ttm_bo_create() — 创建 Buffer Object
    │   ├── ttm_bo_validate() — 验证/迁移 BO 到目标 placement
    │   ├── ttm_tt_populate() — 填充页表（DMA 映射）
    │   └── ttm_bo_evict() — 驱逐 BO（显存不足时）
    │
    ├── DMA-BUF (跨设备共享)
    │   ├── dma_buf_export() — 导出 DMA-BUF
    │   ├── dma_buf_attach() — 附加到另一设备
    │   └── dma_buf_map_attachment() — 映射到设备地址空间
    │
    └── 页面迁移 (Page Migration)
        ├── migrate_vma_setup() — 设置迁移范围
        └── migrate_vma_pages() — 执行页面迁移
```

## 关键接口

### 内核 TTM 接口

- `ttm_bo_create()` — 创建 Buffer Object（GPU 内存分配的核心入口）
  - 参数: `struct ttm_device *bdev`, `size_t size`, `enum ttm_bo_type type`, `struct ttm_placement *placement`, `struct ttm_buffer_object **p_bo`
  - 返回: 0 成功, 负数错误码
  - 源码: `drivers/gpu/drm/ttm/ttm_bo.c`
  - 注意: 分配后 BO 处于 evicted 状态，需调用 ttm_bo_validate() 激活

- `ttm_bo_validate()` — 验证 BO 的 placement 并触发迁移
  - 参数: `struct ttm_buffer_object *bo`, `struct ttm_placement *placement`, `struct ttm_operation_ctx *ctx`
  - 返回: 0 成功, -ENOMEM 显存不足
  - 源码: `drivers/gpu/drm/ttm/ttm_bo.c`
  - 注意: 若目标 placement 无可用空间，会触发 BO 驱逐链

- `ttm_bo_evict()` — 驱逐 BO（从显存移到系统内存）
  - 源码: `drivers/gpu/drm/ttm/ttm_bo.c`
  - 注意: 驱逐策略由 TTM 的 LRU 队列控制，最近最少使用的 BO 优先驱逐

### DMA-BUF 接口

- `dma_buf_export()` — 将 BO 导出为 DMA-BUF fd
  - 源码: `drivers/dma-buf/dma-buf.c`
  - 注意: 导出后其他设备（如显示控制器、视频解码器）可通过 fd 共享 GPU 内存

### 数据结构

- `struct ttm_buffer_object` — TTM Buffer Object（GPU 内存的内核抽象）
  - 关键字段:
    - `mem` — 当前 placement（显存/系统内存/不可移动）
    - `size` — 字节大小
    - `priority` — 驱逐优先级
    - `resource` — `struct drm_gem_object` (GEM 兼容层)
  - 源码: `include/drm/ttm/ttm_bo_api.h`

- `struct ttm_resource_manager` — TTM 内存区域管理器
  - 关键字段:
    - `use_type` — 内存类型（VRAM/TT/GTT）
    - `size` — 区域总大小
    - `lru` — LRU 驱逐队列
  - 源码: `include/drm/ttm/ttm_resource.h`

## 实现要点

### 显存类型对比

| 类型 | 带宽 | 容量 | 功耗 | 代表产品 | 适用场景 |
|------|------|------|------|----------|----------|
| **HBM3e** | 3.35-4.8 TB/s | 64-192 GB | 高 | H100/B200 | AI 训练/推理 |
| **HBM3** | 2-3.35 TB/s | 32-80 GB | 高 | H100, MI300X | AI 训练 |
| **GDDR7** | ~1.8 TB/s | 16-32 GB | 中 | RTX 5090 | 游戏/创作 |
| **GDDR6X** | ~1 TB/s | 12-24 GB | 中 | RTX 4090 | 游戏 |
| **GDDR6** | 0.5-0.72 TB/s | 8-16 GB | 中低 | RX 7900 XTX | 主流游戏 |
| **LPDDR5X** | 0.1-0.2 TB/s | 8-16 GB | 低 | 集成显卡 | 移动端 |

### TTM 内存驱逐策略

TTM 使用 LRU（Least Recently Used）策略管理显存。当显存不足时：

```
新 BO 请求分配显存
    │
    ▼
显存空间不足?
    │
    ├── 是 → 从 LRU tail 开始驱逐
    │       ├── BO 可驱逐? → 迁移到系统内存 (TTM_PL_FLAG_TT)
    │       ├── BO 被锁定? → 跳过
    │       └── BO 正在使用? → 跳过
    │       │
    │       ▼
    │   释放足够空间? → 分配成功
    │                  → 仍不足 → 返回 -ENOMEM
    │
    └── 否 → 直接分配
```

### 内存合并 (Coalescing)

GPU 内存控制器将连续线程的内存访问合并为单次事务，大幅提升带宽利用率：

```
未合并 (最差情况):
Thread 0: 读 addr 0  ─┐
Thread 1: 读 addr 128 ─┤ 4 次独立事务
Thread 2: 读 addr 256 ─┤ 带宽利用率 ~25%
Thread 3: 读 addr 384 ─┘

合并 (最佳情况):
Thread 0: 读 addr 0  ─┐
Thread 1: 读 addr 4  ─┤ 1 次事务 (128-byte cache line)
Thread 2: 读 addr 8  ─┤ 带宽利用率 ~100%
Thread 3: 读 addr 12 ─┘
```

### Texture Tiling（纹理分块）

GPU 将纹理数据按 tile（而非行优先）存储，以提升 2D 空间局部性：

```
Linear (行优先):           Tiled (分块):
┌──┬──┬──┬──┬──┬──┬──┬──┐ ┌──┬──┬──┬──┐
│00│01│02│03│04│05│06│07│ │00│01│04│05│
├──┼──┼──┼──┼──┼──┼──┼──┤ ├──┼──┼──┼──┤
│08│09│10│11│12│13│14│15│ │02│03│06│07│
├──┼──┼──┼──┼──┼──┼──┼──┤ ├──┼──┼──┼──┤
│16│17│18│19│20│21│22│23│ │08│09│12│13│
├──┼──┼──┼──┼──┼──┼──┼──┤ ├──┼──┼──┼──┤
│24│25│26│27│28│29│30│31│ │10│11│14│15│
└──┴──┴──┴──┴──┴──┴──┴──┘ └──┴──┴──┴──┘
  4x8 像素连续存储            4x4 tile 存储
```

## 版本差异

| 版本/架构 | 内存变更 |
|-----------|----------|
| NVIDIA Pascal (GP100) | HBM2 首次用于消费级 (Tesla P100) |
| NVIDIA Ampere (GA100) | HBM2e, 2 TB/s 带宽 |
| NVIDIA Hopper (GH100) | HBM3, 3.35 TB/s, TMA (Tensor Memory Accelerator) |
| NVIDIA Blackwell (GB200) | HBM3e, 4.8 TB/s/stack, 第二代 TMA |
| AMD RDNA 3 (Navi 31) | Chiplet 设计, 96MB Infinity Cache (L3) |
| Intel Xe-HPG (Alchemist) | 16MB L2, GDDR6 |
| Linux TTM v6.0+ | 引入 `ttm_resource_manager` 重构 |
| Linux TTM v6.8+ | 支持可移动 BO 的批量迁移 |

## 陷阱与注意事项

- **显存碎片化：** 频繁分配/释放不同大小的 BO 会导致显存碎片化。TTM 的 buddy 分配器可缓解但不能完全消除。驱动开发者应尽量复用 BO 而非频繁创建/销毁。
- **PCIe 带宽瓶颈：** 当数据需要在 GPU 和 CPU 之间频繁传输时，PCIe 带宽（~32 GB/s PCIe 4.0 x16）远低于显存带宽，成为瓶颈。应使用 DMA-BUF 零拷贝共享，或使用 Unified Memory (CUDA) / ReBAR (Resizable BAR)。
- **TLB Miss 开销：** GPU 的 TLB 容量有限，大范围随机访问会导致大量 TLB miss。大页（2MB/1GB）可显著减少 TLB miss。
- **共享内存 Bank Conflict：** 共享内存分为 32 个 bank（NVIDIA），同一 Warp 内不同线程访问同一 bank 的不同地址会序列化。通过 padding 或使用 `float4` 等宽类型访问可避免。
- **NUMA 感知：** 多 GPU 系统中，应将数据分配在将要使用它的 GPU 的本地显存中。跨 GPU 访问（通过 NVLink/CXL）延迟更高。

## 使用模式

### Vulkan: 显存分配与绑定

```c
// 查询内存类型
VkPhysicalDeviceMemoryProperties memProps;
vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProps);

// 选择合适的内存类型（设备本地 + 设备端可见）
uint32_t memTypeIndex = UINT32_MAX;
for (uint32_t i = 0; i < memProps.memoryTypeCount; i++) {
    if ((memProps.memoryTypes[i].propertyFlags &
         VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT) &&
        (typeFilter & (1 << i))) {
        memTypeIndex = i;
        break;
    }
}

// 分配显存
VkMemoryAllocateInfo allocInfo = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .allocationSize = requirements.size,
    .memoryTypeIndex = memTypeIndex,
};
vkAllocateMemory(device, &allocInfo, NULL, &deviceMemory);

// 绑定到 buffer
vkBindBufferMemory(device, buffer, deviceMemory, 0);
```

### CUDA: 统一内存 (Unified Memory)

```c
// CUDA 统一内存 — 自动在 CPU/GPU 间迁移页面
float* data;
cudaMallocManaged(&data, size * sizeof(float));

// GPU kernel 使用 — 首次访问触发页迁移到 GPU
kernel<<<blocks, threads>>>(data, size);

// CPU 使用 — 触发页迁移回 CPU（或使用 cudaMemPrefetchAsync 控制）
cudaMemPrefetchAsync(data, size * sizeof(float), cudaCpuDeviceId);
```

### 内核驱动: TTM BO 创建与驱逐

```c
// 创建 BO 并指定 placement
struct ttm_placement placement = {
    .num_placement = 1,
    .placement = &(struct ttm_place){
        .fpfn = 0,
        .lpfn = 0,  // 无上界
        .mem_type = TTM_PL_VRAM,  // 请求显存
        .flags = TTM_PL_FLAG_CONTIGUOUS,
    },
};

struct ttm_buffer_object *bo;
int ret = ttm_bo_create(bdev, size, ttm_bo_type_device,
                         &placement, &bo);
if (ret == -ENOMEM) {
    // 显存不足，TTM 已尝试驱逐但仍无法满足
    // 可尝试降级到系统内存
    placement.placement->mem_type = TTM_PL_TT;
    ret = ttm_bo_create(bdev, size, ttm_bo_type_device,
                         &placement, &bo);
}
```

## Sources

- [P1] https://www.abhik.ai/concepts/gpu/memory-hierarchy — GPU Memory Hierarchy & Optimization: 五级层次结构、coalescing、bank conflict
- [P1] https://resources.nvidia.com/en-us-tensor-core/nvidia-tensor-core-gpu-architecture — NVIDIA H100 Architecture: HBM3 带宽、TMA
- [P1] https://gpuopen.com/learn/rdna-3-architecture/ — AMD RDNA 3 Architecture (GPUOpen): Infinity Cache、GDDR6 带宽
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/ttm — drm/ttm: TTM Buffer Object 管理、驱逐策略、页面迁移
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/dma-buf — dma-buf: 跨设备内存共享
- [P0] include/drm/ttm/ttm_bo_api.h — TTM BO API: ttm_bo_create/validate/evict 函数声明
- [P0] include/drm/ttm/ttm_resource.h — TTM Resource Manager: 内存区域管理器定义
- [P1] https://docs.kernel.org/gpu/drm-mm.html — Linux kernel GPU memory management documentation
