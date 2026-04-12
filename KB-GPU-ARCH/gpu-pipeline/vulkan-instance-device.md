# Vulkan Instance 与 Device

> **条目ID:** KB-GPU-ARCH-gpu-pipeline-005
> **模块:** KB-GPU-ARCH / gpu-pipeline
> **置信度:** 5/5（基于 Vulkan 1.4 Specification + Khronos 官方教程 + 内核驱动源码）
> **时效性:** 2026-04-12 | **信源:** Khronos, vulkan-tutorial.com, kernel.org
> **关联:** [vulkan-overview](./vulkan-overview.md) | [vulkan-command-buffer](./vulkan-command-buffer.md)

---

## 核心摘要

Vulkan 的 Instance 和 Device 是 API 的两级抽象——Instance 代表与 Vulkan 库的连接（全局状态），Device 代表对物理 GPU 的逻辑绑定（设备状态）。创建 Instance 时指定应用信息、验证层和全局扩展；创建 Device 时选择物理设备、队列族、设备特性和设备扩展。这种两级设计允许一个应用同时使用多个 GPU（多 GPU 渲染），也允许多个应用共享同一个物理设备。Linux 上，Instance 创建触发 ICD（Installable Client Driver）加载，Device 创建触发 DRM 设备打开和内核驱动初始化。

---

## 1. Instance 创建

### VkApplicationInfo

```c
VkApplicationInfo appInfo = {
    .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
    .pApplicationName = "My App",
    .applicationVersion = VK_MAKE_VERSION(1, 0, 0),
    .pEngineName = "Custom Engine",
    .engineVersion = VK_MAKE_VERSION(1, 0, 0),
    .apiVersion = VK_API_VERSION_1_4,  // 请求 Vulkan 1.4
};
```

### VkInstanceCreateInfo

```c
VkInstanceCreateInfo createInfo = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pApplicationInfo = &appInfo,
    // 验证层（Debug）
    .enabledLayerCount = 1,
    .ppEnabledLayerNames = (const char*[]){ "VK_LAYER_KHRONOS_validation" },
    // 全局扩展
    .enabledExtensionCount = 2,
    .ppEnabledExtensionNames = (const char*[]){
        VK_KHR_SURFACE_EXTENSION_NAME,
        VK_KHR_WAYLAND_SURFACE_EXTENSION_NAME,  // Linux Wayland
    },
};
```

### pNext 链（扩展特性传递）

```c
// Vulkan 的扩展特性通过 pNext 链传递
VkDebugUtilsMessengerCreateInfoEXT debugInfo = { ... };
debugInfo.pNext = NULL;

VkValidationFeaturesEXT validationFeatures = {
    .sType = VK_STRUCTURE_TYPE_VALIDATION_FEATURES_EXT,
    .pNext = &debugInfo,
    .enabledValidationFeatureCount = 1,
    .pEnabledValidationFeatures = (VkValidationFeatureEnableEXT[]){
        VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT,
    },
};

createInfo.pNext = &validationFeatures;
```

---

## 2. 验证层（Validation Layers）

### 标准验证层

| 层 | 说明 |
|------|------|
| `VK_LAYER_KHRONOS_validation` | 标准验证层（推荐使用） |
| `VK_LAYER_KHRONOS_synchronization2` | 同步验证 |
| `VK_LAYER_KHRONOS_shader_object` | Shader Object 验证 |

### 验证层功能

- 参数校验（NULL 指针、越界访问）
- GPU 内存泄漏检测
- 线程安全检查
- 状态追踪（对象生命周期）
- 最佳实践警告
- 着色器验证（SPIR-V 验证）

### 运行时配置

```bash
# 启用验证层
export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation

# 验证层输出级别
export VK_LAYER_ENABLES=VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT

# GPU-Assisted Validation（GPU 端验证）
export VK_LAYER_ENABLES=VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_EXT
```

---

## 3. 物理设备选择

### 查询物理设备

```c
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, NULL);
VkPhysicalDevice devices[deviceCount];
vkEnumeratePhysicalDevices(instance, &deviceCount, devices);

// 查询设备属性
VkPhysicalDeviceProperties props;
vkGetPhysicalDeviceProperties(device, &props);

// 查询设备特性
VkPhysicalDeviceFeatures features;
vkGetPhysicalDeviceFeatures(device, &features);

// 查询队列族
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, NULL);
VkQueueFamilyProperties queueFamilies[queueFamilyCount];
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies);
```

### 设备评分选择策略

```c
int rateDevice(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties props;
    vkGetPhysicalDeviceProperties(device, &props);
    VkPhysicalDeviceFeatures features;
    vkGetPhysicalDeviceFeatures(device, &features);

    int score = 0;
    // 独立 GPU 优先
    if (props.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU)
        score += 1000;
    // 最大纹理大小
    score += props.limits.maxImageDimension2D;
    // 几何着色器
    if (features.geometryShader)
        score += 500;
    return score;
}
```

### 队列族类型

| 标志 | 说明 | 典型用途 |
|------|------|----------|
| `VK_QUEUE_GRAPHICS_BIT` | 图形操作 | 渲染、清屏、Blit |
| `VK_QUEUE_COMPUTE_BIT` | 计算操作 | Compute Shader、AI 推理 |
| `VK_QUEUE_TRANSFER_BIT` | 传输操作 | Buffer/Image 拷贝 |
| `VK_QUEUE_SPARSE_BINDING_BIT` | 稀疏绑定 | 大纹理流式加载 |
| `VK_QUEUE_PROTECTED_BIT` | 受保护操作 | DRM 内容 |
| `VK_QUEUE_VIDEO_DECODE_BIT_KHR` | 视频解码 | 硬件视频解码 |
| `VK_QUEUE_VIDEO_ENCODE_BIT_KHR` | 视频编码 | 硬件视频编码 |

---

## 4. Logical Device 创建

### VkDeviceCreateInfo

```c
// 队列创建信息
float queuePriority = 1.0f;
VkDeviceQueueCreateInfo queueCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
    .queueFamilyIndex = graphicsQueueFamily,
    .queueCount = 1,
    .pQueuePriorities = &queuePriority,
};

// 设备特性
VkPhysicalDeviceFeatures deviceFeatures = {
    .samplerAnisotropy = VK_TRUE,
    .fillModeNonSolid = VK_TRUE,    // 线框模式
    .wideLines = VK_TRUE,           // 宽线
    .shaderInt64 = VK_TRUE,         // 64 位整数
};

VkDeviceCreateInfo deviceCreateInfo = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = &queueCreateInfo,
    .enabledExtensionCount = 3,
    .ppEnabledExtensionNames = (const char*[]){
        VK_KHR_SWAPCHAIN_EXTENSION_NAME,
        VK_KHR_SHADER_DRAW_PARAMETERS_EXTENSION_NAME,
        VK_KHR_SYNCHRONIZATION_2_EXTENSION_NAME,
    },
    .pEnabledFeatures = &deviceFeatures,
};
```

### 设备扩展（常用）

| 扩展 | 说明 | 状态 |
|------|------|------|
| `VK_KHR_swapchain` | Swapchain 支持 | 核心（1.3+） |
| `VK_KHR_synchronization2` | 增强同步 API | 核心（1.3+） |
| `VK_KHR_shader_draw_parameters` | Base Instance/Vertex | 核心（1.3+） |
| `VK_KHR_maintenance4` | 维护特性 4 | 核心（1.3+） |
| `VK_KHR_dynamic_rendering` | 动态渲染 | 核心（1.3+） |
| `VK_KHR_fragment_shader_barycentric` | 重心坐标插值 | 扩展 |
| `VK_EXT_mesh_shader` | Mesh Shader | 扩展 |
| `VK_KHR_ray_tracing_pipeline` | 光追管线 | 扩展 |
| `VK_KHR_acceleration_structure` | 加速结构 | 扩展 |
| `VK_EXT_memory_budget` | 内存预算查询 | 扩展 |

---

## 5. Linux 上的实现细节

### ICD 加载机制

```
libvulkan.so (Vulkan Loader)
  ├─ 读取 /etc/vulkan/icd.d/*.json
  ├─ 读取 /usr/share/vulkan/icd.d/*.json
  ├─ 读取 $VK_ICD_FILENAMES
  └─ 加载 ICD .so
      ├─ NVIDIA: libGLX_nvidia.so → nvidia 驱动
      ├─ AMD:   libvulkan_radeon.so → amdgpu 驱动
      ├─ Intel: libvulkan_intel.so → i915/xe 驱动
      └─ Mesa:  libvulkan_lvp.so → llvmpipe (CPU)
```

### Device 创建与 DRM 的关系

```
vkCreateDevice()
  └─ ICD: radv_CreateDevice()  (AMD)
      ├─ open("/dev/dri/renderD128")  // DRM render node
      ├─ drmIoctl(DRM_IOCTL_GET_CAP)  // 查询设备能力
      ├─ amdgpu_bo_alloc()            // 分配显存
      └─ amdgpu_cs_submit()           // 提交命令流
```

### 多 GPU 支持

```c
// 枚举所有物理设备
VkPhysicalDevice devices[MAX_GPUS];
vkEnumeratePhysicalDevices(instance, &count, devices);

// 为每个 GPU 创建独立 Device
for (int i = 0; i < count; i++) {
    vkCreateDevice(devices[i], &createInfo, NULL, &logicalDevices[i]);
}

// 跨 GPU 共享：通过 VK_KHR_device_group
VkDeviceGroupDeviceCreateInfoKHR groupInfo = {
    .physicalDeviceCount = 2,
    .pPhysicalDevices = (VkPhysicalDevice[]){ gpu0, gpu1 },
};
```

---

## 6. 设备特性（vulkan12/vulkan13/vulkan14）

### Vulkan 1.2 特性（VkPhysicalDeviceVulkan12Features）

| 特性 | 说明 |
|------|------|
| `shaderSampledImageArrayNonUniformIndexing` | 非均匀索引纹理数组 |
| `descriptorIndexing` | 描述符索引（Bindless） |
| `timelineSemaphore` | 时间线信号量 |
| `bufferDeviceAddress` | Buffer 设备地址（GPU 指针） |
| `shaderFloat16` | 16 位浮点着色器 |

### Vulkan 1.3 特性（VkPhysicalDeviceVulkan13Features）

| 特性 | 说明 |
|------|------|
| `dynamicRendering` | 动态渲染（替代 RenderPass） |
| `synchronization2` | 增强同步 API |
| `maintenance4` | 维护特性 4 |
| `shaderIntegerDotProduct` | 整数点积（AI 推理） |
| `pipelineCreationCacheControl` | 管线缓存控制 |

### Vulkan 1.4 特性（VkPhysicalDeviceVulkan14Features）

| 特性 | 说明 |
|------|------|
| `pushDescriptor` | Push 描述符（1.4 核心） |
| `indexTypeUint8` | 8 位索引 |
| `shaderFloat16` | FP16 着色器（1.4 核心） |
| `globalPriorityQuery` | 全局队列优先级查询 |

---

## 潜在影响

- **驱动开发：** Vulkan Device 创建是 GPU 驱动初始化的入口点。Linux 上，ICD 加载机制决定了用户空间 Vulkan 驱动与内核 DRM 驱动的绑定关系。
- **多 GPU 渲染：** VK_KHR_device_group 是 Linux 多 GPU（混合显卡笔记本、多 GPU 工作站）的基础设施。
- **验证层：** GPU-Assisted Validation 在 GPU 端执行额外的检查（如越界访问），对驱动开发者调试着色器 bug 非常有用。
- **演进方向：** Vulkan 1.4 将更多扩展提升为核心特性，简化了应用开发。未来版本可能进一步整合 Mesh Shader 和 Ray Tracing。

---

## 信源

- [Vulkan Tutorial — Instance](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Instance)
- [Vulkan Tutorial — Logical device](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Logical_device_and_queues)
- [Vulkan 1.4 Specification — Khronos](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html)
- [Vulkan Loader — LunarG](https://github.com/KhronosGroup/Vulkan-Loader)
