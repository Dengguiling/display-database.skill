# Vulkan Instance 与 Device

> Vulkan 的两级初始化模型：Instance 管理全局状态和 ICD 分发，Device 管理设备资源和命令队列。

## Meta
- **ID:** KB-GPU-ARCH-vulkan-002
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Vulkan 1.0 ~ 1.4
- **关键词:** VkInstance, VkPhysicalDevice, VkDevice, queue family, ICD, validation layers, device features, device extensions, DRM render node
- **前置知识:** [→ vulkan-overview]
- **关联条目:** [→ vulkan-command-buffer] [→ vulkan-memory]

## 概述

Vulkan 采用两级初始化模型：**Instance** 是全局入口，管理验证层和 ICD 分发；**Device** 是逻辑设备，管理队列族、设备特性和扩展。中间通过 `VkPhysicalDevice`（物理设备枚举）桥接。在 Linux 上，`vkCreateDevice` 会打开 DRM render node（`/dev/dri/renderD128`），建立用户空间驱动与内核 DRM 子系统的通信通道。

## 架构/调用链

### 初始化流程

```
vkCreateInstance()
    │
    ├── 扫描 ICD manifest (JSON)
    │   ├── /etc/vulkan/icd.d/*.json
    │   ├── /usr/share/vulkan/icd.d/*.json
    │   └── 加载匹配的 ICD .so
    │
    ├── 加载 Layers (验证层/调试层)
    │   ├── VK_LAYER_KHRONOS_validation
    │   └── VK_LAYER_KHRONOS_synchronization2
    │
    └── 创建 Instance 句柄
         │
         ▼
vkEnumeratePhysicalDevices()
    │
    ├── 查询每个 ICD 的物理设备列表
    ├── 返回 VkPhysicalDevice[] (轻量句柄)
    └── 每个 VkPhysicalDevice 关联一个 ICD
         │
         ▼
vkGetPhysicalDeviceQueueFamilyProperties()
    │
    └── 获取队列族属性:
        ├── queueFlags (GRAPHICS | COMPUTE | TRANSFER | SPARSE)
        ├── queueCount
        └── timestampValidBits
         │
         ▼
vkGetPhysicalDeviceFeatures2() / Extensions
    │
    └── 查询设备支持的特性和扩展
         │
         ▼
vkCreateDevice()
    │
    ├── 指定队列族配置
    ├── 启用设备特性和扩展
    ├── ICD 内部: open("/dev/dri/renderD128")
    ├── ICD 内部: DRM ioctl 初始化
    └── 创建 Device 句柄
         │
         ▼
vkGetDeviceQueue()
    │
    └── 获取队列句柄 (VkQueue)
```

### Instance 与 Device 的职责边界

```
┌─────────────────────────────────────────────────┐
│                 VkInstance                       │
│  ┌─────────────────────────────────────────────┐ │
│  │ 全局职责:                                    │ │
│  │ • ICD 加载与分发                             │ │
│  │ • 验证层管理                                 │ │
│  │ • 物理设备枚举                               │ │
│  │ • Surface 创建 (WSI)                         │ │
│  │ • Debug Report Callback                      │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
         │ vkCreateDevice()
         ▼
┌─────────────────────────────────────────────────┐
│                  VkDevice                        │
│  ┌─────────────────────────────────────────────┐ │
│  │ 设备职责:                                    │ │
│  │ • 队列管理 (Queue)                           │ │
│  │ • 命令缓冲区池 (Command Pool)                │ │
│  │ • 内存分配 (Device Memory)                   │ │
│  │ • 管线创建 (Pipeline)                        │ │
│  │ • Descriptor 管理                            │ │
│  │ • 同步原语 (Fence/Semaphore)                 │ │
│  │ • Buffer/Image 创建                          │ │
│  │ • RenderPass / Framebuffer                   │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## 关键接口

### Instance 层

- `vkCreateInstance(const VkInstanceCreateInfo*, const VkAllocationCallbacks*, VkInstance*)` — 创建 Instance
  - `pCreateInfo->pApplicationInfo->apiVersion`: 请求的 API 版本
  - `pCreateInfo->ppEnabledLayerNames`: 启用的验证层
  - `pCreateInfo->ppEnabledExtensionNames`: Instance 扩展（如 `VK_KHR_surface`）
  - 注意: 必须在所有其他 Vulkan 调用之前
  - 源码: Vulkan Loader `loader/vk_loader_platform.h`

- `vkEnumeratePhysicalDevices(VkInstance, uint32_t*, VkPhysicalDevice*)` — 枚举物理设备
  - 第一次调用 `count=NULL` 获取数量，第二次调用获取数组
  - 返回的 VkPhysicalDevice 是轻量句柄，不持有资源

- `vkGetPhysicalDeviceProperties2(VkPhysicalDevice, VkPhysicalDeviceProperties2*)` — 查询设备属性
  - `VkPhysicalDeviceProperties`: vendorID, deviceID, driverVersion, limits
  - `VkPhysicalDeviceSubgroupProperties`: subgroupSize, supportedStages
  - `VkPhysicalDeviceDriverProperties`: driverName, driverInfo, conformanceVersion

- `vkGetPhysicalDeviceQueueFamilyProperties2(VkPhysicalDevice, uint32_t*, VkQueueFamilyProperties2*)` — 查询队列族属性
  - `queueFlags`: `VK_QUEUE_GRAPHICS_BIT | VK_QUEUE_COMPUTE_BIT | VK_QUEUE_TRANSFER_BIT | VK_QUEUE_SPARSE_BINDING_BIT`
  - `queueCount`: 该族可用队列数
  - `timestampValidBits`: 时间戳有效位数

### Device 层

- `vkCreateDevice(VkPhysicalDevice, const VkDeviceCreateInfo*, const VkAllocationCallbacks*, VkDevice*)` — 创建逻辑设备
  - `pCreateInfo->pQueueCreateInfos`: 队列族配置（每个族请求多少队列）
  - `pCreateInfo->ppEnabledExtensionNames`: 设备扩展（如 `VK_KHR_swapchain`）
  - `pCreateInfo->pEnabledFeatures`: 启用的设备特性
  - Linux 内部: ICD 调用 `open("/dev/dri/renderD128")` + DRM auth
  - 注意: 一个 PhysicalDevice 可创建多个 Device（罕见）

- `vkGetDeviceQueue(VkDevice, uint32_t queueFamilyIndex, uint32_t queueIndex, VkQueue*)` — 获取队列句柄
  - 注意: 返回的 VkQueue 不需要销毁，随 Device 自动销毁

- `vkDeviceWaitIdle(VkDevice)` — 等待设备所有队列空闲
  - 注意: 重量级操作，应避免频繁调用

- `vkDestroyDevice(VkDevice, const VkAllocationCallbacks*)` — 销毁设备
  - 注意: 必须在所有使用该 Device 的操作完成后调用

## 实现要点

### Linux DRM Render Node 映射

```
物理设备 → DRM 设备节点映射:
┌──────────────────────────────────────────────┐
│ /dev/dri/card0    → DRM master node (root)   │
│ /dev/dri/renderD128 → DRM render node (无root)│
│ /dev/dri/renderD129 → 第二个 GPU              │
│ /dev/dri/card1    → 第二个 GPU master node    │
│ /dev/dri/renderD130 → 第三个 GPU              │
└──────────────────────────────────────────────┘

Vulkan ICD 在 vkCreateDevice() 中:
1. 根据 VkPhysicalDevice 确定 DRM 节点
2. open(render_node, O_RDWR | O_CLOEXEC)
3. 通过 DRM ioctl 初始化 GEM/TTM 内存管理
4. 创建 DRM scheduler 上下文
```

### 多 GPU 系统中的设备选择

```c
// 选择离散 GPU 优先于集成 GPU
VkPhysicalDevice selectDiscreteGPU(VkPhysicalDevice* devices, uint32_t count) {
    for (uint32_t i = 0; i < count; i++) {
        VkPhysicalDeviceProperties props;
        vkGetPhysicalDeviceProperties(devices[i], &props);
        if (props.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
            return devices[i];
        }
    }
    return devices[0]; // fallback
}
```

### 验证层配置

```c
// 调试模式: 启用标准验证层
const char* validationLayers[] = {
    "VK_LAYER_KHRONOS_validation",
};

// 检查验证层是否可用
uint32_t layerCount;
vkEnumerateInstanceLayerProperties(&layerCount, NULL);
VkLayerProperties* layers = malloc(sizeof(VkLayerProperties) * layerCount);
vkEnumerateInstanceLayerProperties(&layerCount, layers);
// 遍历 layers 确认 "VK_LAYER_KHRONOS_validation" 存在

// 生产环境: 禁用验证层（零开销）
// enabledLayerCount = 0
```

### 设备特性协商

```c
// Vulkan 1.1+ 使用 pNext 链查询扩展特性
VkPhysicalDeviceVulkan12Features features12 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_FEATURES,
};

VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &features12,
};
vkGetPhysicalDeviceFeatures2(physDev, &features2);

// 按需启用特性
features12.shaderSampledImageArrayNonUniformIndexing = VK_TRUE;
features12.descriptorIndexing = VK_TRUE;
features12.timelineSemaphore = VK_TRUE;
```

## 版本差异

| 版本 | Instance 层变更 | Device 层变更 |
|------|-----------------|---------------|
| 1.0 | 基础 Instance/Device 创建 | 基础队列和特性 |
| 1.1 | `vkEnumeratePhysicalDeviceGroups` | Subgroup 操作、Protected Memory |
| 1.2 | `vkEnumerateInstanceVersion` | Descriptor Indexing、Timeline Semaphore、SPIR-V 1.5 |
| 1.3 | — | Dynamic Rendering、Maintenance4 |
| 1.4 | — | Mesh Shader (核心)、Push Descriptor (核心)、Index Type Uint8 |

## 陷阱与注意事项

- **Instance 重复创建：** 一个应用程序通常只需要一个 Instance。创建多个 Instance 不会带来性能优势，反而增加 ICD 加载开销。
- **队列族选择错误：** `VK_QUEUE_COMPUTE_BIT` 不一定包含 `VK_QUEUE_GRAPHICS_BIT`。某些 GPU（如 NVIDIA）的 Compute 队列是独立的，不能执行图形命令。必须通过 `vkGetPhysicalDeviceQueueFamilyProperties` 查询。
- **Device 扩展不等于 Instance 扩展：** `VK_KHR_swapchain` 是 Device 扩展，`VK_KHR_surface` 是 Instance 扩展。混淆会导致创建失败。
- **Render Node 权限：** Linux 上普通用户需要 `video` 组权限才能打开 `/dev/dri/renderD128`。无权限时 `vkCreateDevice` 返回 `VK_ERROR_INITIALIZATION_FAILED`。
- **物理设备句柄生命周期：** `VkPhysicalDevice` 的生命周期由 Instance 管理。销毁 Instance 后使用 PhysicalDevice 句柄是未定义行为。
- **特性链断裂：** 使用 `pNext` 链查询扩展特性时，必须确保链中每个 `sType` 对应的扩展已被启用，否则返回的特性值不可靠。

## 使用模式

### 完整初始化流程

```c
// 1. 创建 Instance（带验证层）
VkInstance instance;
const char* layers[] = {"VK_LAYER_KHRONOS_validation"};
const char* extensions[] = {"VK_KHR_surface", "VK_KHR_wayland_surface"};

vkCreateInstance(&(VkInstanceCreateInfo){
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pApplicationInfo = &(VkApplicationInfo){
        .sType = VK_STRUCTURE_TYPE_APPLICATION_INFO,
        .pApplicationName = "MyApp",
        .apiVersion = VK_API_VERSION_1_4,
    },
    .enabledLayerCount = 1,
    .ppEnabledLayerNames = layers,
    .enabledExtensionCount = 2,
    .ppEnabledExtensionNames = extensions,
}, NULL, &instance);

// 2. 枚举并选择物理设备
VkPhysicalDevice physDev;
uint32_t devCount = 0;
vkEnumeratePhysicalDevices(instance, &devCount, NULL);
VkPhysicalDevice* devs = malloc(sizeof(VkPhysicalDevice) * devCount);
vkEnumeratePhysicalDevices(instance, &devCount, devs);
physDev = selectDiscreteGPU(devs, devCount); // 见实现要点

// 3. 查询队列族
uint32_t qfCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(physDev, &qfCount, NULL);
VkQueueFamilyProperties* qfProps = malloc(sizeof(VkQueueFamilyProperties) * qfCount);
vkGetPhysicalDeviceQueueFamilyProperties(physDev, &qfCount, qfProps);

// 找到支持图形的队列族
uint32_t graphicsQF = UINT32_MAX;
for (uint32_t i = 0; i < qfCount; i++) {
    if (qfProps[i].queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        graphicsQF = i;
        break;
    }
}

// 4. 创建 Device
VkDevice device;
const char* devExts[] = {"VK_KHR_swapchain", "VK_KHR_dynamic_rendering"};
float qPriority = 1.0f;

vkCreateDevice(physDev, &(VkDeviceCreateInfo){
    .sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = &(VkDeviceQueueCreateInfo){
        .sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO,
        .queueFamilyIndex = graphicsQF,
        .queueCount = 1,
        .pQueuePriorities = &qPriority,
    },
    .enabledExtensionCount = 2,
    .ppEnabledExtensionNames = devExts,
}, NULL, &device);

// 5. 获取队列
VkQueue queue;
vkGetDeviceQueue(device, graphicsQF, 0, &queue);

// 6. 清理（程序退出时）
vkDestroyDevice(device, NULL);
vkDestroyInstance(instance, NULL);
```

## Sources

- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#initialization — Vulkan Spec §Initialization: Instance/Device 创建流程、生命周期
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#devsandqueues — Vulkan Spec §Devices and Queues: 队列族、设备特性、扩展
- [P0] https://github.com/KhronosGroup/Vulkan-Loader — Vulkan Loader 源码: ICD manifest 解析、Layer 加载
- [P0] https://github.com/KhronosGroup/Vulkan-ValidationLayers — Vulkan Validation Layers: 验证层实现
- [P1] https://vulkan.lunarg.com/doc/view/1.4.335.1/mac/antora/tutorial/latest/03_Initialization.html — Vulkan Tutorial §Initialization: 完整初始化代码
- [P1] https://docs.kernel.org/gpu/drm-usage-stats.html — Linux kernel DRM usage stats: render node 权限
- [P2] https://vulkan-tutorial.com/Dedicated_resources — Vulkan Tutorial: 多 GPU 设备选择
