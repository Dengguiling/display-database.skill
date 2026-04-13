# Vulkan Memory

> Vulkan 的显式内存管理体系——应用程序直接控制设备内存的分配、绑定与生命周期，区分 Memory Heap（物理资源）和 Memory Type（属性组合）。

## Meta
- **ID:** KB-GPU-ARCH-vulkan-005
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-14
- **适用版本:** Vulkan 1.0 ~ 1.4
- **关键词:** device memory, memory heap, memory type, VkMemoryAllocateInfo, VkMemoryRequirements, vkAllocateMemory, vkBindBufferMemory, vkBindImageMemory, memory barrier, VMA, VulkanMemoryAllocator, VK_KHR_memory_budget, sparse memory, host visible, device local, host coherent, host cached
- **前置知识:** [→ vulkan-overview] [→ vulkan-instance-device] [→ vulkan-pipeline]
- **关联条目:** [→ gpu-memory-hierarchy] [→ vulkan-command-buffer]

## 概述

Vulkan 的内存管理是"完全显式"的——驱动不自动管理任何 GPU 内存。应用程序必须自行查询物理设备的 Memory Heap（物理内存资源，如 VRAM、系统 RAM）和 Memory Type（属性组合，如 device-local、host-visible），然后通过 `vkAllocateMemory` 分配 `VkDeviceMemory`，再通过 `vkBindBufferMemory` / `vkBindImageMemory` 将 Buffer/Image 绑定到已分配的内存上。这种设计消除了驱动端的隐式内存分配和迁移开销，但也要求应用程序自行处理内存碎片、对齐、类型匹配等问题。实际工程中，绝大多数项目使用 AMD 开源的 VulkanMemoryAllocator（VMA）库来管理内存分配。

## 架构/调用链

### 设备内存分配与绑定流程

```
vkGetPhysicalDeviceMemoryProperties()
    │
    ├── memoryHeaps[]    ← 物理内存资源（VRAM、系统 RAM）
    │   └── size, flags (DEVICE_LOCAL, MULTI_INSTANCE)
    │
    └── memoryTypes[]    ← 属性组合（从 Heap 中分配）
        └── heapIndex, propertyFlags
            (DEVICE_LOCAL, HOST_VISIBLE, HOST_COHERENT,
             HOST_CACHED, LAZILY_ALLOCATED, PROTECTED)
    │
    ▼
vkGetBufferMemoryRequirements() / vkGetImageMemoryRequirements()
    │
    ├── size          ← 所需字节数
    ├── alignment     ← 对齐要求
    └── memoryTypeBits ← 位掩码，指示兼容的 Memory Type
    │
    ▼
vkAllocateMemory(VkMemoryAllocateInfo)
    │
    └── VkDeviceMemory ← 已分配的设备内存块
    │
    ▼
vkBindBufferMemory(buffer, memory, offset)
vkBindImageMemory(image, memory, offset)
    │
    ▼
vkMapMemory() / vkFlushMappedMemoryRanges() / vkInvalidateMappedMemoryRanges()
    │  ← 仅 HOST_VISIBLE 内存可 Map
    ▼
vkFreeMemory() ← 释放
```

### Memory Heap 与 Memory Type 关系

```
Physical Device
    │
    ├── Heap 0: VRAM (DEVICE_LOCAL)
    │   ├── Type 0: DEVICE_LOCAL          ← GPU 专用（纹理、渲染目标）
    │   └── Type 1: DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT  ← ReBAR/UMA
    │
    └── Heap 1: System RAM
        ├── Type 2: HOST_VISIBLE | HOST_COHERENT  ← Staging Buffer（CPU→GPU）
        └── Type 3: HOST_VISIBLE | HOST_CACHED    ← Readback Buffer（GPU→CPU）
```

### VMA（VulkanMemoryAllocator）抽象层

```
vmaCreateAllocator()
    │
    ├── vmaCreateBuffer(bufferInfo, allocInfo, &buffer, &alloc, &allocInfo)
    │   └── 内部: vkCreateBuffer → vkGetBufferMemoryRequirements
    │            → 选择 Memory Type → vkAllocateMemory → vkBindBufferMemory
    │
    ├── vmaCreateImage(imageInfo, allocInfo, &image, &alloc, &allocInfo)
    │
    ├── vmaMapMemory(alloc, &data) / vmaUnmapMemory(alloc)
    │
    ├── vmaDestroyBuffer(buffer, alloc)
    └── vmaDestroyAllocator()
```

## 关键接口

### 函数

- `vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memoryProperties)` — 查询设备的 Memory Heap 和 Memory Type 列表
  - 参数: `physicalDevice` 物理设备句柄, `memoryProperties` 输出结构体
  - 返回: void
  - 注意: 此函数不分配资源，仅查询信息；结果在设备生命周期内不变

- `vkGetBufferMemoryRequirements(device, buffer, &memoryRequirements)` — 查询 Buffer 的内存需求
  - 参数: `device` 逻辑设备, `buffer` Buffer 句柄, `memoryRequirements` 输出
  - 返回: void
  - 注意: 必须在 `vkCreateBuffer` 之后调用；`memoryTypeBits` 是关键——只有对应位为 1 的 Memory Type 才兼容

- `vkGetImageMemoryRequirements(device, image, &memoryRequirements)` — 查询 Image 的内存需求
  - 参数: 同上，替换 buffer 为 image
  - 返回: void
  - 注意: Image 的对齐要求通常比 Buffer 更严格（如 256 字节对齐）；稀疏 Image 使用 `vkGetImageSparseMemoryRequirements`

- `vkAllocateMemory(device, &allocateInfo, allocator, &memory)` — 分配设备内存
  - 参数: `allocateInfo` 包含 `memoryTypeIndex` 和 `allocationSize`
  - 返回: `VkResult`（VK_SUCCESS / VK_ERROR_OUT_OF_DEVICE_MEMORY / VK_ERROR_OUT_OF_HOST_MEMORY）
  - 注意: `allocationSize` 必须 ≥ `memoryRequirements.size`；`memoryTypeIndex` 必须满足 `memoryTypeBits` 约束

- `vkFreeMemory(device, memory, allocator)` — 释放设备内存
  - 参数: `memory` 要释放的内存句柄
  - 返回: void
  - 注意: 释放前必须确保所有使用该内存的 GPU 操作已完成（通过 Fence 或等待 Queue 空闲）；绑定在该内存上的 Buffer/Image 必须先销毁

- `vkBindBufferMemory(device, buffer, memory, offset)` — 将 Buffer 绑定到设备内存
  - 参数: `offset` 必须满足 `memoryRequirements.alignment` 对齐
  - 返回: `VkResult`
  - 注意: 一个 Buffer 只能绑定到一个 Memory 区域；一个 Memory 可以绑定多个 Buffer（子分配）

- `vkBindImageMemory(device, image, memory, offset)` — 将 Image 绑定到设备内存
  - 参数: 同上
  - 返回: `VkResult`
  - 注意: 某些 Image 格式要求 Dedicated Allocation（`VK_KHR_dedicated_allocation`），即一个 Image 独占一个 Memory

- `vkMapMemory(device, memory, offset, size, flags, &ppData)` — 将设备内存映射到 Host 地址空间
  - 参数: `offset` 和 `size` 定义映射范围；`flags` 保留，必须为 0
  - 返回: `VkResult`
  - 注意: 仅 `HOST_VISIBLE` 内存可 Map；Map 后需 `vkFlushMappedMemoryRanges`（写入后）和 `vkInvalidateMappedMemoryRanges`（读取前）确保一致性，除非 `HOST_COHERENT`

- `vkFlushMappedMemoryRanges(device, rangeCount, ranges)` — 刷新 Host 写入到 Device
  - 注意: 仅 `HOST_VISIBLE` 但非 `HOST_COHERENT` 的内存需要调用；保证写入对 Device 可见

- `vkInvalidateMappedMemoryRanges(device, rangeCount, ranges)` — 使 Device 写入对 Host 可见
  - 注意: 仅 `HOST_VISIBLE` 但非 `HOST_COHERENT` 的内存需要调用；保证 Device 写入在 Host 读取前完成

### 数据结构

- `struct VkPhysicalDeviceMemoryProperties` — 设备内存属性查询结果
  - 关键字段:
    - `memoryHeapCount` / `memoryHeaps[VK_MAX_MEMORY_HEAPS]` — Heap 数组和数量
    - `memoryTypeCount` / `memoryTypes[VK_MAX_MEMORY_TYPES]` — Type 数组和数量
  - 源码: `vulkan/vulkan_core.h`

- `struct VkMemoryHeap` — 单个内存堆描述
  - 关键字段:
    - `size` — 堆大小（字节）
    - `flags` — `VK_MEMORY_HEAP_DEVICE_LOCAL_BIT`, `VK_MEMORY_HEAP_MULTI_INSTANCE_BIT`

- `struct VkMemoryType` — 单个内存类型描述
  - 关键字段:
    - `propertyFlags` — 属性位掩码
    - `heapIndex` — 所属 Heap 索引

- `struct VkMemoryAllocateInfo` — 内存分配参数
  - 关键字段:
    - `sType` — `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO`
    - `allocationSize` — 分配大小
    - `memoryTypeIndex` — Memory Type 索引

- `struct VkMemoryRequirements` — 资源内存需求
  - 关键字段:
    - `size` — 所需最小字节数
    - `alignment` — 对齐要求
    - `memoryTypeBits` — 兼容 Memory Type 位掩码

## 实现要点

### 1. Memory Type 选择算法

应用程序必须根据资源使用场景选择最优 Memory Type。标准选择逻辑：

```
需求: DEVICE_LOCAL + 非 HOST_VISIBLE
  → Type 0 (VRAM, GPU 专用)
  用途: 纹理、渲染目标、深度缓冲、Storage Buffer/Image

需求: HOST_VISIBLE + HOST_COHERENT (可选 DEVICE_LOCAL)
  → Type 1 (ReBAR) 或 Type 2 (系统 RAM Staging)
  用途: Uniform Buffer (小量频繁更新)、Staging Buffer (CPU→GPU 传输)

需求: HOST_VISIBLE + HOST_CACHED
  → Type 3 (系统 RAM Cached)
  用途: Readback Buffer (GPU→CPU 回读)
```

关键约束：`memoryTypeBits & (1 << memoryTypeIndex)` 必须非零。

### 2. 内存碎片与子分配

`vkAllocateMemory` 是重量级操作（驱动层面可能涉及内核 ioctl、页表操作），不适合频繁小粒度调用。标准做法是：
- 分配大块 Memory，在其中进行子分配（buddy allocator、free-list allocator）
- VMA 库内部实现了多种子分配策略：`VMA_POOL_CREATE_LINEAR_ALGORITHM_BIT`（线性）、默认（buddy-like）

### 3. 内存屏障（Memory Barrier）

Vulkan 不保证执行顺序和内存可见性，必须通过 Pipeline Barrier 显式声明：

```c
vkCmdPipelineBarrier(
    commandBuffer,
    VK_PIPELINE_STAGE_TRANSFER_BIT,        // srcStageMask
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, // dstStageMask
    0, dependencyFlags,
    0, NULL,                                // memory barriers
    0, NULL,                                // buffer memory barriers
    1, &imageMemoryBarrier                  // image memory barriers
);
```

- `VkMemoryBarrier` — 全局内存屏障
- `VkBufferMemoryBarrier` — 特定 Buffer 的访问权限转换
- `VkImageMemoryBarrier` — 特定 Image 的 Layout 转换 + 访问权限转换

### 4. Dedicated Allocation

某些资源（大尺寸 Image、采样 Image、具有 `VK_IMAGE_CREATE_SPARSE_BINDING_BIT` 的 Image）要求或推荐使用 Dedicated Allocation——一个 `VkDeviceMemory` 只绑定一个资源。通过 `VK_KHR_dedicated_allocation` 扩展，驱动可以提示哪些资源需要 Dedicated Allocation（`VkMemoryDedicatedRequirements`）。

### 5. Sparse Memory（稀疏内存）

`VK_KHR_sparse_binding` 允许将 Image/Buffer 的不同区域绑定到不同的 `VkDeviceMemory` 块，实现：
- 虚拟纹理（Virtual Texturing）：按需加载 Mip Level
- 大型资源分页：超出 VRAM 的纹理部分驻留在系统 RAM
- 稀疏缓冲区：仅分配实际使用的区域

### 6. VK_KHR_memory_budget

Vulkan 1.1 引入 `vkGetPhysicalDeviceMemoryProperties2` + `VkPhysicalDeviceMemoryBudgetPropertiesEXT`，允许查询每个 Heap 的当前使用量和预算值，用于实现内存压力感知的资源管理。

### 7. VMA（VulkanMemoryAllocator）核心设计

AMD 开源的单头文件库（`vk_mem_alloc.h`），提供：
- **自动 Memory Type 选择**：基于 `VmaMemoryUsage` 枚举（GPU_ONLY、CPU_ONLY、CPU_TO_GPU、GPU_TO_CPU、AUTO）
- **内存池管理**：支持自定义池策略（线性、buddy）
- **碎片整理**：`vmaDefragment()` 支持在线碎片整理
- **统计与调试**：内置内存使用统计、JSON 导出
- **持久映射**：`VMA_ALLOCATION_CREATE_MAPPED_BIT` 支持创建即映射

## 版本差异

| 版本 | 变更 |
|------|------|
| Vulkan 1.0 | 基础内存管理：Heap/Type 查询、分配/绑定/映射/刷新 |
| Vulkan 1.1 | `vkGetPhysicalDeviceMemoryProperties2` 支持扩展属性链；`VK_KHR_dedicated_allocation` 升级为核心 |
| Vulkan 1.3 | `VK_EXT_extended_dynamic_state` 升级为核心，减少 PSO 数量从而间接减少内存分配次数 |
| Vulkan 1.4 | `VK_KHR_maintenance5` 升级为核心，放宽内存绑定约束；`VK_KHR_pipeline_binary` 提供显式管线缓存控制 |

## 陷阱与注意事项

- **Memory Type 选择错误**：最常见错误。必须同时满足 `memoryTypeBits` 约束和属性需求。例如，Staging Buffer 需要 `HOST_VISIBLE`，但如果选择了 `DEVICE_LOCAL` 但非 `HOST_VISIBLE` 的 Type，`vkMapMemory` 会返回 `VK_ERROR_MEMORY_MAP_FAILED`
- **忽略 `vkFlushMappedMemoryRanges`**：对于 `HOST_VISIBLE` 但非 `HOST_COHERENT` 的内存，写入后必须 Flush，否则 GPU 可能读到旧数据。`HOST_COHERENT` 内存无需 Flush/Invalidate，但性能可能略低
- **过度分配小内存块**：`vkAllocateMemory` 是重量级操作。每帧分配多个小 Buffer 会导致严重性能问题。应使用 VMA 或自定义子分配器
- **未等待 GPU 完成就释放内存**：`vkFreeMemory` 不会等待 GPU 操作完成。如果在 GPU 仍在使用该内存时释放，会导致未定义行为。必须通过 Fence 或 `vkDeviceWaitIdle` 确保同步
- **Image Layout 未转换就使用**：Image 绑定内存后初始 Layout 为 `VK_IMAGE_LAYOUT_UNDEFINED`，必须通过 Pipeline Barrier 转换到目标 Layout（如 `SHADER_READ_OPTIMAL`）后才能在着色器中采样
- **UMA 架构的特殊性**：在集成 GPU（Intel/AMD APU、移动端）上，VRAM 和系统 RAM 共享物理内存，`DEVICE_LOCAL | HOST_VISIBLE` 类型的 Memory 可能同时具备两种属性，此时 Staging Buffer 和 GPU 资源可以使用同一块内存，省去 Copy 操作
- **对齐要求**：`VkMemoryRequirements.alignment` 必须严格遵守。Buffer 通常 64/256 字节对齐，Image 可能要求 4096 字节甚至更大对齐。子分配时偏移量必须满足对齐

## 使用模式

### 模式 1：GPU 专用资源（纹理、渲染目标）

```c
// 1. 查询内存需求
VkMemoryRequirements memReq;
vkGetImageMemoryRequirements(device, image, &memReq);

// 2. 选择 Memory Type: DEVICE_LOCAL
uint32_t memTypeIndex = findMemoryType(memReq.memoryTypeBits,
    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

// 3. 分配
VkMemoryAllocateInfo allocInfo = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .allocationSize = memReq.size,
    .memoryTypeIndex = memTypeIndex,
};
vkAllocateMemory(device, &allocInfo, NULL, &memory);

// 4. 绑定
vkBindImageMemory(device, image, memory, 0);
```

### 模式 2：Staging Buffer（CPU→GPU 数据传输）

```c
// 1. 创建 HOST_VISIBLE | HOST_COHERENT 的 Staging Buffer
VkBufferCreateInfo bufInfo = {
    .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
    .size = dataSize,
    .usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
};
vkCreateBuffer(device, &bufInfo, NULL, &stagingBuffer);

// 2. 分配 HOST_VISIBLE 内存
uint32_t memTypeIndex = findMemoryType(memReq.memoryTypeBits,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
    VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
vkAllocateMemory(device, &allocInfo, NULL, &stagingMemory);
vkBindBufferMemory(device, stagingBuffer, stagingMemory, 0);

// 3. Map + 填充 + Copy
void* data;
vkMapMemory(device, stagingMemory, 0, dataSize, 0, &data);
memcpy(data, vertexData, dataSize);
vkUnmapMemory(device, stagingMemory);

// 4. 通过 Command Buffer Copy 到 GPU 专用 Buffer
vkCmdCopyBuffer(cmdBuf, stagingBuffer, gpuBuffer, 1, &copyRegion);
```

### 模式 3：使用 VMA 简化分配

```c
// 初始化
VmaAllocatorCreateInfo allocatorInfo = {
    .physicalDevice = physicalDevice,
    .device = device,
    .instance = instance,
    .vulkanApiVersion = VK_API_VERSION_1_3,
};
vmaCreateAllocator(&allocatorInfo, &allocator);

// 创建 GPU 专用 Image
VmaAllocationCreateInfo allocInfo = {
    .usage = VMA_MEMORY_USAGE_AUTO,
    .flags = VMA_ALLOCATION_CREATE_DEDICATED_MEMORY_BIT,
};
vmaCreateImage(allocator, &imgCreateInfo, &allocInfo,
               &image, &alloc, &allocInfo_out);

// 创建 Staging Buffer（持久映射）
VmaAllocationCreateInfo stagingAllocInfo = {
    .usage = VMA_MEMORY_USAGE_AUTO,
    .flags = VMA_ALLOCATION_CREATE_HOST_ACCESS_SEQUENTIAL_WRITE_BIT |
             VMA_ALLOCATION_CREATE_MAPPED_BIT,
};
vmaCreateBuffer(allocator, &bufCreateInfo, &stagingAllocInfo,
                &buffer, &alloc, &allocInfo_out);
memcpy(allocInfo_out.pMappedData, data, size);  // 直接写入
```

## Sources

- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#memory — Vulkan Spec §Memory Allocation: Heap/Type 定义、分配/释放/映射/刷新完整规范
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#memory-device — Vulkan Spec §Device Memory: VkDeviceMemory 生命周期、Host 访问规则
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-memory-barriers — Vulkan Spec §Memory Barriers: Pipeline Barrier、Buffer/Image Memory Barrier
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#resources-association — Vulkan Spec §Resource Memory Association: Buffer/Image 与 Memory 的绑定规则
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceMemoryProperties.html — Vulkan Ref Pages: VkPhysicalDeviceMemoryProperties 字段说明
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/VkMemoryAllocateInfo.html — Vulkan Ref Pages: VkMemoryAllocateInfo 字段说明
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/vkBindBufferMemory.html — Vulkan Ref Pages: vkBindBufferMemory 规范
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/vkMapMemory.html — Vulkan Ref Pages: vkMapMemory 规范
- [P1] https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator — AMD VulkanMemoryAllocator (VMA): 开源内存管理库
- [P1] https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/usage_patterns.html — VMA Documentation: 推荐使用模式（GPU-only、Staging、Readback、Persistent Mapping）
- [P1] https://gpuopen.com/learn/vulkan-device-memory — AMD GPUOpen: Using Vulkan Device Memory（2016-07）
- [P2] https://asawicki.info/news_1740_vulkan_memory_types_on_pc_and_how_to_use_them — Vulkan Memory Types on PC and How to Use Them（2021-02）: 实际硬件上的 Memory Type 分布
