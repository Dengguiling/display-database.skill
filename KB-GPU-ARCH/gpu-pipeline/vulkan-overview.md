# Vulkan API 概览

> **条目ID:** KB-GPU-ARCH-gpu-pipeline-004
> **模块:** KB-GPU-ARCH / gpu-pipeline
> **置信度:** 5/5（基于 Vulkan 1.4 Specification + Khronos 官方文档 + NVIDIA/AMD 架构文档）
> **时效性:** 2026-04-12 | **信源:** Khronos, NVIDIA, AMD, vulkan.lunarg.com
> **关联:** [gpu-rendering-pipeline](./gpu-rendering-pipeline.md) | [shader-core-arch](./shader-core-arch.md)

---

## 核心摘要

Vulkan 是 Khronos Group 制定的**跨平台低级图形和计算 API**，于 2016 年发布，旨在替代 OpenGL，提供更接近硬件的编程模型。Vulkan 的核心设计哲学是**显式控制**（explicit control）：驱动不做任何隐式状态管理或资源追踪，所有操作（内存分配、同步、管线状态）由应用程序明确指定。这种设计消除了驱动端的猜测开销，使得多线程命令录制、异步计算、精细同步成为可能。Vulkan 1.4（2024）已将光追、Mesh Shader、Descriptor Indexing 等关键扩展提升为核心特性。

---

## 1. 设计哲学

### Vulkan vs OpenGL vs DirectX 12

| 维度 | OpenGL | Vulkan | DirectX 12 | Metal |
|------|--------|--------|------------|-------|
| 抽象层级 | 高（隐式状态机） | 低（显式控制） | 低（显式控制） | 低（显式控制） |
| 多线程录制 | ❌ 全局状态锁 | ✅ 无全局状态 | ✅ 无全局状态 | ✅ 无全局状态 |
| 内存管理 | 驱动管理 | 应用管理 | 应用管理 | 应用管理 |
| 同步 | 隐式（flush/finish） | 显式（fence/semaphore/barrier） | 显式（fence/barrier） | 显式（fence/barrier） |
| 管线状态 | 全局状态对象 | 不可变 PSO | 不可变 PSO | 不可变 PSO |
| 着色器语言 | GLSL | SPIR-V | HLSL | MSL |
| 验证层 | 无 | 可选验证层 | Debug Layer | Metal Validation |
| 跨平台 | ✅ | ✅ | ❌ Windows/Xbox | ❌ Apple |
| 开销 | 高（驱动验证多） | 低（thin driver） | 低（thin driver） | 低（thin driver） |

### 显式控制的核心原则

```
OpenGL 模型:
  glBindTexture(unit, tex);  // 隐式绑定到全局槽位
  glDrawArrays(...);         // 驱动在 draw 时检查所有状态

Vulkan 模型:
  vkCmdBindDescriptorSets(...);  // 显式绑定到命令缓冲区
  vkCmdDraw(...);                // 所有状态已预先绑定，无运行时检查
```

---

## 2. 核心对象层次

### 对象创建链

```
VkInstance
  ├─ VkPhysicalDevice (枚举)
  │   ├─ VkPhysicalDeviceProperties
  │   ├─ VkPhysicalDeviceFeatures
  │   └─ VkQueueFamilyProperties[]
  │
  ├─ VkDevice
  │   ├─ VkQueue[] (Graphics, Compute, Transfer, Sparse)
  │   ├─ VkCommandPool
  │   │   └─ VkCommandBuffer[]
  │   ├─ VkDescriptorPool
  │   │   └─ VkDescriptorSet[]
  │   ├─ VkPipelineCache
  │   ├─ VkPipelineLayout
  │   │   └─ VkDescriptorSetLayout[]
  │   └─ VkRenderPass
  │       └─ VkFramebuffer
  │
  ├─ VkSurfaceKHR
  │   └─ VkSwapchainKHR
  │       └─ VkImage[] + VkImageView[]
  │
  └─ VkDebugUtilsMessengerEXT (验证层)
```

### 生命周期管理

| 阶段 | 操作 | 说明 |
|------|------|------|
| 初始化 | `vkCreateInstance` | 加载全局扩展和验证层 |
| 设备选择 | `vkEnumeratePhysicalDevices` | 查询可用 GPU |
| 设备创建 | `vkCreateDevice` | 创建逻辑设备，请求队列族 |
| 资源创建 | `vkCreate*` | 创建 Buffer、Image、Pipeline 等 |
| 渲染循环 | `vkAcquireNextImage` → 录制命令 → `vkQueueSubmit` → `vkQueuePresentKHR` | 每帧执行 |
| 清理 | `vkDestroy*` | 按创建逆序销毁 |

---

## 3. 渲染管线

### 固定功能阶段（Vulkan 管线）

```
Input Assembly ──→ Vertex Shader ──→ Tessellation ──→ Geometry Shader ──→
     │
     └─ Mesh Shader (替代路径)
         │
         ▼
Rasterization ──→ Fragment Shader ──→ Color Blend ──→ Framebuffer
     │
     └─ Ray Tracing (替代路径)
         │
         ▼
Acceleration Structure ──→ Ray Gen ──→ Intersection ──→ Closest Hit ──→ Miss
```

### Pipeline State Object (PSO)

```c
VkGraphicsPipelineCreateInfo pipelineInfo = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .stageCount = 2,
    .pStages = shaderStages,        // VS + FS
    .pVertexInputState = &vertexInputInfo,
    .pInputAssemblyState = &inputAssembly,
    .pViewportState = &viewportState,
    .pRasterizationState = &rasterizer,
    .pMultisampleState = &multisampling,
    .pDepthStencilState = &depthStencil,
    .pColorBlendState = &colorBlending,
    .pDynamicState = &dynamicState,
    .layout = pipelineLayout,
    .renderPass = renderPass,
    .subpass = 0,
};
```

**PSO 的不可变性：** 创建后不可修改。需要不同状态时必须创建新的 PSO。驱动可在创建时进行深度优化（指令缓存、状态合并）。

---

## 4. 内存管理

### 内存分配模型

```
应用程序:
  1. vkCreateBuffer() / vkCreateImage()  → 获取内存需求
  2. vkGetBufferMemoryRequirements()     → size, alignment, memoryTypeBits
  3. vkAllocateMemory()                  → 从合适的 heap 分配
  4. vkBindBufferMemory()                → 绑定到 buffer/image

内存类型 (memoryType):
  ┌──────────────────────────────────────────────┐
  │ Heap 0: Device Local (VRAM)                  │
  │   Type 0: Device Local, LAZILY_ALLOCATED     │
  │   Type 1: Device Local                       │
  │ Heap 1: Host Visible (System RAM)            │
  │   Type 2: Host Visible | Host Coherent       │
  │   Type 3: Host Visible | Host Cached         │
  └──────────────────────────────────────────────┘
```

### 内存属性标志

| 标志 | 说明 | 典型用途 |
|------|------|----------|
| `VK_MEMORY_PROPERTY_DEVICE_LOCAL` | GPU 可高速访问 | 纹理、顶点缓冲、FrameBuffer |
| `VK_MEMORY_PROPERTY_HOST_VISIBLE` | CPU 可访问 | Staging Buffer |
| `VK_MEMORY_PROPERTY_HOST_COHERENT` | CPU 写入立即可见 | Uniform Buffer（小数据） |
| `VK_MEMORY_PROPERTY_HOST_CACHED` | CPU 可缓存 | CPU 频繁读取的数据 |
| `VK_MEMORY_PROPERTY_LAZILY_ALLOCATED` | 延迟分配 | 大型稀疏纹理 |

### Staging Buffer 模式

```
CPU 端:
  vkMapMemory(stagingMem, ...) → memcpy(data) → vkUnmapMemory(stagingMem)

GPU 端:
  vkCmdCopyBuffer(cmdBuf, stagingBuf, gpuBuf, ...)
  // 或使用 vkCmdCopyBufferToImage()
```

---

## 5. 同步原语

### 三大同步机制

| 原语 | 作用域 | 类型 | 典型用途 |
|------|--------|------|----------|
| **Fence** | Host ↔ Device | 信号量（可重置/二值） | 等待 GPU 完成，获取下一帧 |
| **Semaphore** | Device ↔ Device | 信号量（二值） | Image 获取 → 渲染 → 呈现 |
| **Barrier** | Device 内部 | 执行屏障 | Layout 转换、Cache flush |

### 帧同步流程

```
Frame N:
  vkAcquireNextImageKHR(swapchain, semaphore_acquire)
  vkQueueSubmit(queue, {
      .waitSemaphoreCount = 1,
      .pWaitSemaphores = &semaphore_acquire,  // 等待 image 可用
      .signalSemaphoreCount = 1,
      .pSignalSemaphores = &semaphore_render, // 渲染完成信号
      .commandBufferCount = 1,
      .pCommandBuffers = &cmdBuf,
      .fence = fence                           // CPU 等待用
  })
  vkQueuePresentKHR(queue, {
      .waitSemaphoreCount = 1,
      .pWaitSemaphores = &semaphore_render,    // 等待渲染完成
  })
  vkWaitForFences(fence)  // CPU 等待
  vkResetFences(fence)
```

### Memory Barrier

```c
VkMemoryBarrier barrier = {
    .sType = VK_STRUCTURE_TYPE_MEMORY_BARRIER,
    .srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_SHADER_READ_BIT,
};
vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,  // 源阶段
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,           // 目标阶段
    0, 1, &barrier, 0, NULL, 0, NULL);
```

---

## 6. Vulkan 1.4 核心特性

### 从扩展提升为核心

| 特性 | 原扩展 | 说明 |
|------|--------|------|
| Mesh Shader | `VK_EXT_mesh_shader` | 替代 VS+GS，GPU 驱动几何生成 |
| Ray Tracing | `VK_KHR_ray_tracing_pipeline` | 硬件加速光线追踪 |
| Descriptor Indexing | `VK_EXT_descriptor_indexing` | 动态绑定描述符，bindless 资源 |
| Fragment Shading Rate | `VK_KHR_fragment_shading_rate` | VRS 可变着色率 |
| Dynamic Rendering | `VK_KHR_dynamic_rendering` | 无需 RenderPass 对象 |
| Maintenance 4/5 | `VK_KHR_maintenance_*` | 各种 API 改进 |
| Shader Subgroup | `VK_KHR_shader_subgroup_*` | Warp/Wavefront 级操作 |

### 关键扩展生态

| 扩展 | 说明 |
|------|------|
| `VK_KHR_swapchain` | 显示交换链（核心依赖） |
| `VK_KHR_surface` | 平台抽象（Wayland/X11/Win32） |
| `VK_EXT_memory_budget` | 查询 VRAM 使用量 |
| `VK_EXT_debug_utils` | 调试标签和消息 |
| `VK_KHR_timeline_semaphore` | 时间线信号量（monotonic counter） |
| `VK_KHR_synchronization2` | 简化的 barrier API |
| `VK_EXT_fragment_density_map` | 虚拟现实 foveated rendering |
| `VK_KHR_video_encode_queue` | 硬件视频编码 |

---

## 7. 驱动开发视角

### Vulkan ICD (Installable Client Driver)

```
libvulkan.so (Loader)
  ├─ 查询 JSON manifest → 加载 ICD .so
  ├─ NVIDIA: libGLX_nvidia.so
  ├─ AMD:   libvulkan_radeon.so
  ├─ Intel: libvulkan_intel.so
  └─ Mesa:  libvulkan_lvp.so (lavapipe, 软件渲染)
```

### Mesa Vulkan 驱动架构

```
Vulkan API
  └─ Mesa Vulkan Layer (common code)
      ├─ RADV (AMD GPU)
      ├─ ANV (Intel GPU)
      ├─ TU (Qualcomm Adreno)
      ├─ V3DV (Broadcom VC5)
      └─ LVP (llvmpipe, CPU)
```

### 内核驱动交互

```c
// Mesa RADV → amdgpu kernel driver
// 1. 通过 DRM ioctl 创建 BO (Buffer Object)
// 2. 通过 DMA-BUF 共享 buffer
// 3. 通过 DRM Sync Object 同步
// 4. 通过 DRM scheduler 提交命令
```

---

## 潜在影响

- **跨平台图形：** Vulkan 是唯一真正跨平台（Linux/Windows/Android/Steam Deck）的现代图形 API，是游戏引擎（Unreal/Unity/Godot）和 Wayland compositor 的首选。
- **Linux 图形栈：** Vulkan 是 Linux 图形栈的核心——Mesa 的 RADV/ANV 驱动直接对接内核 DRM 子系统，是 Linux GPU 驱动开发的主要 API 目标。
- **AI 推理：** Vulkan Compute Shader 被用于 AI 推理加速（如 llama.cpp 的 Vulkan backend），在无 CUDA 的平台上提供 GPU 加速能力。
- **演进方向：** Vulkan 1.4 已将 Mesh Shader、Ray Tracing 等提升为核心特性。WebGPU 的出现为 Web 端提供了类似的低级 API，但 Vulkan 在桌面/嵌入式领域仍是主流。

---

## 信源

- [Vulkan 1.4 Specification — Khronos](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html)
- [Vulkan Tutorial — vulkan.lunarg.com](https://vulkan.lunarg.com/doc/view/1.4.335.1/mac/antora/tutorial/latest/01_Overview.html)
- [A Comparison of Modern Graphics APIs — alain.xyz](https://alain.xyz/blog/comparison-of-modern-graphics-apis)
- [NVIDIA Vulkan Driver Support](https://developer.nvidia.com/vulkan-driver)
- [Mesa 3D — mesa3d.org](https://mesa3d.org/)
