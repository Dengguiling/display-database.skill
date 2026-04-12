# Vulkan API 概览

> Khronos 制定的跨平台低级图形和计算 API，核心设计哲学是显式控制——驱动不做任何隐式状态管理。

## Meta
- **ID:** KB-GPU-ARCH-vulkan-001
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Vulkan 1.0 ~ 1.4
- **关键词:** Vulkan, Khronos, SPIR-V, explicit control, low-level API, graphics API, compute API, ray tracing, mesh shader, validation layers, ICD
- **前置知识:** [→ gpu-rendering-pipeline]
- **关联条目:** [→ vulkan-instance-device] [→ vulkan-command-buffer] [→ vulkan-pipeline] [→ vulkan-memory]

## 概述

Vulkan 是 Khronos Group 制定的跨平台低级图形和计算 API（2016 年发布），旨在替代 OpenGL。核心设计哲学是**显式控制**：驱动不做任何隐式状态管理或资源追踪，所有操作（内存分配、同步、管线状态）由应用程序明确指定。Vulkan 1.4（2024）已将光追、Mesh Shader、Descriptor Indexing 等关键扩展提升为核心特性。在 Linux 图形栈中，Vulkan 是核心 API——Mesa 的 RADV/ANV 驱动直接对接内核 DRM 子系统。

## 架构/调用链

### Vulkan 系统架构

```
应用程序 (Vulkan API 调用)
    │
    ▼
Vulkan Loader (libvulkan.so)
    ├── 加载 ICD (Installable Client Driver)
    │   ├── NVIDIA: libGLX_nvidia.so
    │   ├── AMD:    libvulkan_radeon.so (Mesa RADV)
    │   ├── Intel:  libvulkan_intel.so (Mesa ANV)
    │   └── Qualcomm: libvulkan_adreno.so (Turnip)
    ├── 加载 Layers (验证层/调试层)
    │   ├── VK_LAYER_KHRONOS_validation
    │   └── GPU-Assisted Validation
    └── 分发 API 调用到对应 ICD
         │
         ▼
用户空间驱动 (ICD)
    ├── SPIR-V → GPU ISA 编译
    ├── 命令缓冲区 → GPU 命令流翻译
    └── 资源管理 → DRM ioctl
         │
         ▼
内核 DRM 子系统
    ├── DRM ioctl (设备管理)
    ├── GEM/TTM (内存管理)
    ├── DRM Scheduler (命令调度)
    └── DMA-BUF (跨设备共享)
         │
         ▼
GPU 硬件
```

### Vulkan API 层次结构

```
Vulkan API
    │
    ├── Instance 层 (全局)
    │   ├── vkCreateInstance()
    │   ├── vkEnumeratePhysicalDevices()
    │   └── vkCreateDevice()
    │
    ├── Device 层 (设备)
    │   ├── Queue (命令提交)
    │   │   ├── vkQueueSubmit()
    │   │   └── vkQueuePresentKHR()
    │   ├── Command Buffer (命令录制)
    │   │   ├── vkBeginCommandBuffer()
    │   │   ├── vkCmdDraw*()
    │   │   └── vkEndCommandBuffer()
    │   ├── Pipeline (管线状态)
    │   │   ├── vkCreateGraphicsPipelines()
    │   │   ├── vkCreateComputePipelines()
    │   │   └── vkCreateRayTracingPipelinesKHR()
    │   ├── Memory (内存管理)
    │   │   ├── vkAllocateMemory()
    │   │   ├── vkMapMemory()
    │   │   └── vkBindBufferMemory()
    │   ├── Descriptor (资源绑定)
    │   │   ├── vkCreateDescriptorSetLayout()
    │   │   ├── vkAllocateDescriptorSets()
    │   │   └── vkUpdateDescriptorSets()
    │   ├── Synchronization (同步)
    │   │   ├── vkCreateFence()
    │   │   ├── vkCreateSemaphore()
    │   │   └── vkCmdPipelineBarrier()
    │   └── RenderPass / Framebuffer
    │       ├── vkCreateRenderPass()
    │       └── vkCreateFramebuffer()
    │
    └── WSI 层 (窗口系统集成)
        ├── VkSurfaceKHR
        ├── VkSwapchainKHR
        └── vkAcquireNextImageKHR()
```

## 关键接口

### Instance 层

- `vkCreateInstance(const VkInstanceCreateInfo*, const VkAllocationCallbacks*, VkInstance*)` — 创建 Vulkan Instance（全局入口）
  - 参数: `pCreateInfo` 含 `pApplicationInfo`（API 版本）和 `ppEnabledLayerNames`（验证层）
  - 返回: `VK_SUCCESS` 或错误码
  - 源码: Vulkan Loader — `loader/vk_loader_platform.h`
  - 注意: 触发 ICD 扫描和加载，Linux 上搜索 `/etc/vulkan/icd.d/*.json` 和 `/usr/share/vulkan/icd.d/*.json`

- `vkEnumeratePhysicalDevices(VkInstance, uint32_t*, VkPhysicalDevice*)` — 枚举可用物理设备
  - 注意: 返回的 VkPhysicalDevice 是轻量句柄，不持有资源

### Device 层

- `vkCreateDevice(VkPhysicalDevice, const VkDeviceCreateInfo*, const VkAllocationCallbacks*, VkDevice*)` — 创建逻辑设备
  - 参数: `pCreateInfo` 含 `queueCreateInfoCount`（队列族配置）和 `ppEnabledExtensionNames`（设备扩展）
  - 注意: Linux 上触发 DRM 设备打开 (`open("/dev/dri/renderD128")`)

- `vkQueueSubmit(VkQueue, uint32_t, const VkSubmitInfo*, VkFence)` — 提交命令缓冲区到队列
  - 参数: `pSubmitInfo` 含 `commandBufferCount` 和 `pCommandBuffers`
  - 注意: 提交后命令缓冲区进入 Pending 状态，Fence 用于 CPU 端等待

- `vkCmdPipelineBarrier(VkCommandBuffer, VkPipelineStageFlags, VkPipelineStageFlags, VkDependencyFlags, uint32_t, const VkMemoryBarrier*, uint32_t, const VkBufferMemoryBarrier*, uint32_t, const VkImageMemoryBarrier*)` — 显式内存/执行屏障
  - 注意: Vulkan 同步的核心机制，必须正确指定 srcStageMask 和 dstStageMask

### WSI 层

- `vkCreateSwapchainKHR(VkDevice, const VkSwapchainCreateInfoKHR*, const VkAllocationCallbacks*, VkSwapchainKHR*)` — 创建交换链
  - 注意: Linux Wayland 上通过 `vkCreateWaylandSurfaceKHR` 创建 Surface

## 实现要点

### ICD 加载机制 (Linux)

Vulkan Loader 在 Linux 上通过 JSON manifest 发现 ICD：

```
搜索路径:
1. /etc/vulkan/icd.d/*.json
2. /usr/share/vulkan/icd.d/*.json
3. /usr/local/share/vulkan/icd.d/*.json

JSON 格式:
{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "libvulkan_radeon.so",
        "api_version": "1.4.300"
    }
}
```

### SPIR-V 编译流程

```
GLSL/HLSL 源码
    │
    ▼ glslangValidator / DXC
SPIR-V 中间表示
    │
    ▼ Vulkan 驱动 (ICD)
    ├── 前端: SPIR-V → 驱动 IR
    ├── 优化: LLVM / 内部优化器
    └── 后端: 驱动 IR → GPU ISA
         │
         ▼
GPU 可执行代码
```

### Linux Mesa Vulkan 驱动映射

| Mesa 驱动 | GPU 厂商 | 内核驱动 | SPIR-V 编译器 |
|-----------|----------|----------|---------------|
| RADV | AMD | amdgpu | ACO / LLVM |
| ANV | Intel | i915/xe | Intel ISPC |
| Turnip | Qualcomm | msm/msm-drm | Turnip Compiler |
| NVK | NVIDIA | nouveau | Nak (WIP) |
| LVP | CPU (llvmpipe) | — | LLVM |
| V3DV | Broadcom VC5 | v3d | V3D Compiler |

### Vulkan vs OpenGL vs DX12 vs Metal

| 维度 | OpenGL | Vulkan | DX12 | Metal |
|------|--------|--------|------|-------|
| 抽象层级 | 高（隐式状态机） | 低（显式控制） | 低（显式控制） | 低（显式控制） |
| 多线程录制 | ❌ 全局状态锁 | ✅ 无全局状态 | ✅ 无全局状态 | ✅ 无全局状态 |
| 内存管理 | 驱动管理 | 应用管理 | 应用管理 | 应用管理 |
| 同步 | 隐式 flush/finish | 显式 fence/semaphore/barrier | 显式 fence/barrier | 显式 fence/barrier |
| 管线状态 | 全局状态对象 | 不可变 PSO | 不可变 PSO | 不可变 PSO |
| 着色器语言 | GLSL | SPIR-V | HLSL | MSL |
| 跨平台 | ✅ | ✅ | ❌ Windows/Xbox | ❌ Apple |
| 验证层 | 无 | 可选验证层 | Debug Layer | Metal Validation |

## 版本差异

| 版本 | 年份 | 核心变更 |
|------|------|----------|
| Vulkan 1.0 | 2016 | 初始版本，核心图形/计算 API |
| Vulkan 1.1 | 2018 | Subgroup 操作、Multiview、Protected Memory |
| Vulkan 1.2 | 2020 | Descriptor Indexing、Timeline Semaphore、SPIR-V 1.5 |
| Vulkan 1.3 | 2022 | Dynamic Rendering、Enhanced Subgroup、Shader Integer Dot Product |
| Vulkan 1.4 | 2024 | Mesh Shader、Ray Tracing (核心)、Push Descriptor、Index Type Uint8 |

## 陷阱与注意事项

- **同步原语误用：** Vulkan 的 Fence（CPU-GPU）、Semaphore（GPU-GPU）、Barrier（同一队列内）语义不同。混用会导致未定义行为。Fence 只能被 `vkResetFences` 重置一次。
- **内存泄漏：** Vulkan 无自动垃圾回收。每个 `vkCreate*` 必须对应 `vkDestroy*`，每个 `vkAllocate*` 必须对应 `vkFree*`。验证层可检测泄漏。
- **Command Buffer 重入：** 一个 Command Buffer 在 `vkQueueSubmit` 后进入 Pending 状态，不能同时被另一个 `vkQueueSubmit` 提交，除非设置了 `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`。
- **RenderPass 兼容性：** Secondary Command Buffer 的 RenderPass 必须与 Primary 的 RenderPass 兼容（subpass 索引和 attachment 格式匹配），否则行为未定义。
- **Descriptor 更新时机：** `vkUpdateDescriptorSets` 修改的 Descriptor Set 在下一次 `vkCmdBindDescriptorSets` 或使用该 set 的 `vkCmd*` 调用时生效，不是立即生效。

## 使用模式

### 最小 Vulkan 初始化

```c
// 1. 创建 Instance
VkInstance instance;
vkCreateInstance(&(VkInstanceCreateInfo){
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pApplicationInfo = &(VkApplicationInfo){
        .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
        .apiVersion = VK_API_VERSION_1_4,
    },
}, NULL, &instance);

// 2. 选择物理设备
VkPhysicalDevice physDev;
uint32_t count = 1;
vkEnumeratePhysicalDevices(instance, &count, &physDev);

// 3. 创建 Device (获取图形队列)
uint32_t queueFamily;
vkGetPhysicalDeviceQueueFamilyProperties(physDev, &count, NULL);
// ... 找到支持 VK_QUEUE_GRAPHICS_BIT 的 queueFamily

VkDevice device;
vkCreateDevice(physDev, &(VkDeviceCreateInfo){
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = &(VkDeviceQueueCreateInfo){
        .sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
        .queueFamilyIndex = queueFamily,
        .queueCount = 1,
    },
}, NULL, &device);
```

### 验证层启用（调试模式）

```c
const char* layers[] = {"VK_LAYER_KHRONOS_validation"};

VkInstanceCreateInfo createInfo = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .enabledLayerCount = 1,
    .ppEnabledLayerNames = layers,
    // 生产环境应禁用验证层（性能开销）
};
```

## Sources

- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html — Vulkan 1.4 Specification: API 完整定义、各章节功能规范
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#fundamentals — Vulkan Spec §Fundamentals: 执行模型、内存模型、同步语义
- [P0] https://github.com/KhronosGroup/Vulkan-Loader — Vulkan Loader 源码: ICD 加载机制、Layer 分发
- [P1] https://vulkan.lunarg.com/doc/view/1.4.335.1/mac/antora/tutorial/latest/01_Overview.html — Vulkan Tutorial (LunarG): 入门教程
- [P1] https://docs.vulkan.org/spec/latest/chapters/pipelines.html — Vulkan Spec §Pipelines: 管线创建与状态管理
- [P1] https://mesa3d.org/ — Mesa 3D: 开源 Vulkan 驱动 (RADV/ANV/Turnip)
- [P2] https://alain.xyz/blog/comparison-of-modern-graphics-apis — A Comparison of Modern Graphics APIs: Vulkan vs DX12 vs Metal vs OpenGL
