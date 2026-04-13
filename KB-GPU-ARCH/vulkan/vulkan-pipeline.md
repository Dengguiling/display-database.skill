# Vulkan Pipeline

> Vulkan 的管线状态对象（PSO）机制——将所有 GPU 状态（着色器、固定功能、资源布局）预编译为不可变对象，绑定后直接执行。

## Meta
- **ID:** KB-GPU-ARCH-vulkan-004
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-14
- **适用版本:** Vulkan 1.0 ~ 1.4
- **关键词:** pipeline state object, PSO, VkGraphicsPipelineCreateInfo, VkComputePipelineCreateInfo, pipeline layout, descriptor set layout, push constants, pipeline cache, dynamic state, VK_EXT_graphics_pipeline_library, VK_KHR_pipeline_binary, pipeline derivative
- **前置知识:** [→ vulkan-overview] [→ vulkan-instance-device] [→ vulkan-command-buffer]
- **关联条目:** [→ gpu-rendering-pipeline] [→ vulkan-memory]

## 概述

Vulkan Pipeline 是 GPU 执行的完整状态描述对象（Pipeline State Object, PSO）。与 OpenGL 的隐式状态切换不同，Vulkan 要求应用程序在创建管线时一次性指定所有状态——包括着色器阶段、顶点输入格式、图元拓扑、光栅化参数、混合模式、深度/模板测试、资源布局等。创建后的 PSO 是不可变的（immutable），运行时只能通过 `vkCmdBindPipeline` 绑定切换。这种"预编译一切"的设计消除了驱动端的隐式状态追踪开销，但也带来了管线创建耗时较长的问题，因此 Vulkan 1.3+ 引入了 Pipeline Cache、Graphics Pipeline Library、Extended Dynamic State 等机制来缓解。

## 架构/调用链

### 图形管线创建流程

```
SPIR-V 着色器编译
    │
    ▼
VkShaderModule (vkCreateShaderModule)
    │
    ▼
Descriptor Set Layout (vkCreateDescriptorSetLayout)
    │  ← 定义着色器可访问的资源槽位
    ▼
Pipeline Layout (vkCreatePipelineLayout)
    │  ← 集合 Descriptor Set Layout + Push Constant Ranges
    ▼
Render Pass (vkCreateRenderPass)          ← 定义子通道和附件格式
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  VkGraphicsPipelineCreateInfo                            │
│  ├── pStages[]          ← 着色器阶段 (VS/FS/...)        │
│  ├── pVertexInputState  ← 顶点输入格式                   │
│  ├── pInputAssemblyState← 图元拓扑 (triangle/list/strip) │
│  ├── pViewportState    ← 视口和裁剪矩形                 │
│  ├── pRasterizationState← 光栅化参数                     │
│  ├── pMultisampleState ← MSAA 配置                      │
│  ├── pDepthStencilState← 深度/模板测试                   │
│  ├── pColorBlendState  ← 颜色混合                        │
│  ├── pDynamicState     ← 动态状态列表                    │
│  ├── layout            ← Pipeline Layout                │
│  ├── renderPass        ← Render Pass                     │
│  └── basePipelineHandle← 派生管线（可选）                │
└─────────────────────────────────────────────────────────┘
    │
    ▼
vkCreateGraphicsPipelines(device, pipelineCache, ...)
    │
    ▼
VkPipeline (不可变，绑定后执行)
    │ vkCmdBindPipeline(cmdBuf, bindPoint, pipeline)
    ▼
Command Buffer 录制绘制命令
```

### 计算管线创建流程

```
VkShaderModule (compute shader SPIR-V)
    │
    ▼
Pipeline Layout (descriptor set layout + push constants)
    │
    ▼
┌─────────────────────────────────────┐
│  VkComputePipelineCreateInfo         │
│  ├── stage (compute shader)          │
│  ├── layout                          │
│  └── basePipelineHandle (可选)       │
└─────────────────────────────────────┘
    │
    ▼
vkCreateComputePipelines(device, pipelineCache, ...)
    │
    ▼
VkPipeline
```

### 管线状态分组（Vulkan 1.3+ Graphics Pipeline Library）

```
┌──────────────────────────────────────────────────────────┐
│  Graphics Pipeline Library (VK_EXT_graphics_pipeline_library) │
│                                                           │
│  ┌─────────────────────┐  ┌────────────────────────────┐ │
│  │ Vertex Input State   │  │ Pre-Rasterization State    │ │
│  │ - Vertex Input       │  │ - Vertex Shader            │ │
│  │ - Input Assembly     │  │ - Tessellation (optional)  │ │
│  │ - (Vertex Buffers)   │  │ - Geometry Shader (opt)    │ │
│  └─────────────────────┘  │ - Mesh Shader (opt)         │ │
│                            │ - Task Shader (opt)         │ │
│                            │ - Viewport State            │ │
│                            │ - Rasterization State       │ │
│                            └────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────┐  ┌────────────────────────────┐ │
│  │ Fragment Shader      │  │ Fragment Output State       │ │
│  │ State                │  │ - Color Blend State         │ │
│  │ - Fragment Shader    │  │ - Multisample State         │ │
│  │ - Depth/Stencil      │  │ - (Render Pass)             │ │
│  └─────────────────────┘  └────────────────────────────┘ │
│                                                           │
│  各部分可独立编译，运行时链接为完整 PSO                       │
└──────────────────────────────────────────────────────────┘
```

## 关键接口

### 函数

- `vkCreateGraphicsPipelines(device, pipelineCache, createInfoCount, pCreateInfos, allocator, pPipelines)` — 创建图形管线
  - 参数: `pipelineCache` 可为 VK_NULL_HANDLE（无缓存）或有效缓存对象；`pCreateInfos` 数组支持批量创建
  - 返回: `VkResult`；管线句柄写入 `pPipelines` 数组
  - 注意: 创建耗时可能很长（着色器编译 + 驱动优化），不应在渲染循环中同步创建

- `vkCreateComputePipelines(device, pipelineCache, createInfoCount, pCreateInfos, allocator, pPipelines)` — 创建计算管线
  - 参数: 同上，但 `VkComputePipelineCreateInfo` 只含单个着色器阶段
  - 返回: 同上
  - 注意: 计算管线比图形管线简单得多，只有一个 CS 阶段

- `vkCreatePipelineCache(device, pCreateInfo, allocator, pPipelineCache)` — 创建管线缓存
  - 参数: `initialDataSize` + `pInitialData` 可加载之前序列化的缓存数据
  - 返回: `VkPipelineCache`
  - 注意: 缓存对象可跨多个 `vkCreate*Pipelines` 调用共享，驱动自动复用内部编译结果

- `vkGetPipelineCacheData(device, pipelineCache, pDataSize, pData)` — 序列化管线缓存到内存
  - 用途: 应用退出时持久化缓存到磁盘，下次启动时加载以避免重复编译
  - 注意: 数据格式由驱动定义，不保证跨驱动/跨版本兼容

- `vkMergePipelineCaches(device, dstCache, srcCacheCount, pSrcCaches)` — 合并多个管线缓存
  - 用途: 多线程创建管线时，各线程使用独立缓存，创建完成后合并

- `vkCmdBindPipeline(commandBuffer, pipelineBindPoint, pipeline)` — 绑定管线到命令缓冲
  - 参数: `pipelineBindPoint` 为 `VK_PIPELINE_BIND_POINT_GRAPHICS` 或 `VK_PIPELINE_BIND_POINT_COMPUTE`
  - 注意: 绑定操作本身开销极低（仅切换内部指针），是 Vulkan 高效状态切换的基础

- `vkCreateShaderModule(device, pCreateInfo, allocator, pShaderModule)` — 从 SPIR-V 字节码创建着色器模块
  - 参数: `pCode` 指向 SPIR-V 字节码（`uint32_t` 数组），`codeSize` 为字节数
  - 注意: 着色器模块可被多个管线共享引用；管线创建后模块可安全销毁

- `vkCreatePipelineLayout(device, pCreateInfo, allocator, pPipelineLayout)` — 创建管线布局
  - 参数: `pSetLayouts` 指定 Descriptor Set Layout 数组；`pPushConstantRanges` 指定 Push Constant 范围
  - 返回: `VkPipelineLayout`
  - 注意: 管线布局定义了着色器如何访问资源，是管线与资源绑定的桥梁

- `vkCreateDescriptorSetLayout(device, pCreateInfo, allocator, pDescriptorSetLayout)` — 创建描述符集布局
  - 参数: `pBindings` 数组，每个 binding 指定 `descriptorType`、`descriptorCount`、`stageFlags`、`pImmutableSamplers`
  - 返回: `VkDescriptorSetLayout`
  - 注意: 布局对象可被多个 Pipeline Layout 引用；binding 编号在着色器 SPIR-V 中通过 `layout(binding = N)` 对应

### 数据结构

- `struct VkGraphicsPipelineCreateInfo` — 图形管线创建信息，包含所有图形管线状态
  - 关键字段:
    - `sType` — 必须为 `VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO`
    - `flags` — `VK_PIPELINE_CREATE_DERIVATIVE_BIT`（派生管线）、`VK_PIPELINE_CREATE_ALLOW_DERIVATIVES_BIT`（允许被派生）
    - `stageCount` / `pStages` — 着色器阶段数组（`VkPipelineShaderStageCreateInfo`）
    - `pVertexInputState` — 顶点输入状态（`VkPipelineVertexInputStateCreateInfo`）
    - `pInputAssemblyState` — 输入装配状态（图元拓扑）
    - `pTessellationState` — 曲面细分状态（可选）
    - `pViewportState` — 视口和裁剪状态
    - `pRasterizationState` — 光栅化状态
    - `pMultisampleState` — 多重采样状态
    - `pDepthStencilState` — 深度/模板状态
    - `pColorBlendState` — 颜色混合状态
    - `pDynamicState` — 动态状态列表
    - `layout` — `VkPipelineLayout`
    - `renderPass` / `subpass` — Render Pass 和子通道索引
    - `basePipelineHandle` / `basePipelineIndex` — 派生管线的父管线

- `struct VkPipelineShaderStageCreateInfo` — 单个着色器阶段信息
  - 关键字段:
    - `stage` — `VK_SHADER_STAGE_VERTEX_BIT`、`VK_SHADER_STAGE_FRAGMENT_BIT`、`VK_SHADER_STAGE_COMPUTE_BIT` 等
    - `module` — `VkShaderModule` 句柄
    - `pName` — 入口点函数名（通常为 `"main"`）
    - `pSpecializationInfo` — 特化常量（编译时常量替换）

- `struct VkPipelineVertexInputStateCreateInfo` — 顶点输入状态
  - 关键字段:
    - `vertexBindingDescriptionCount` / `pVertexBindingDescriptions` — 顶点缓冲绑定（binding、stride、inputRate）
    - `vertexAttributeDescriptionCount` / `pVertexAttributeDescriptions` — 顶点属性（location、binding、format、offset）

- `struct VkPipelineLayoutCreateInfo` — 管线布局
  - 关键字段:
    - `setLayoutCount` / `pSetLayouts` — Descriptor Set Layout 数组（最多 `maxBoundDescriptorSets`，通常 4-8）
    - `pushConstantRangeCount` / `pPushConstantRanges` — Push Constant 范围（每阶段最多 `maxPushConstantsSize`，通常 128-256 字节）

- `struct VkPipelineCacheCreateInfo` — 管线缓存创建信息
  - 关键字段:
    - `initialDataSize` / `pInitialData` — 初始缓存数据（从 `vkGetPipelineCacheData` 获取）
    - `flags` — 目前保留，必须为 0

## 实现要点

### 管线创建的内部过程

驱动在 `vkCreateGraphicsPipelines` 内部执行以下步骤：
1. **SPIR-V 验证与前端编译**：验证 SPIR-V 合法性，将 SPIR-V 翻译为驱动内部 IR（如 LLVM IR、AMDIL 等）
2. **着色器优化**：执行死代码消除、常量折叠、循环展开等编译器优化 pass
3. **后端代码生成**：将优化后的 IR 翻译为目标 GPU 的机器码
4. **管线状态编译**：将固定功能状态（混合、深度、光栅化等）编译为 GPU 硬件配置
5. **管线缓存查询/写入**：检查 Pipeline Cache 中是否有匹配的已编译结果，有则复用；无则将新结果写入缓存

### Pipeline Cache 机制

Pipeline Cache 是驱动管理的键值存储，键为管线状态的哈希值，值为编译产物（shader binary + 硬件配置 blob）：
- **跨管线复用**：如果两个管线共享部分着色器或状态，缓存可复用编译产物
- **跨运行持久化**：通过 `vkGetPipelineCacheData` 序列化到磁盘，下次启动时加载
- **线程安全**：多个线程可同时使用同一 Pipeline Cache 创建管线（内部加锁）
- **VK_KHR_pipeline_binary（Vulkan 1.4）**：新扩展，允许应用直接获取管线关联的二进制数据，绕过 VkPipelineCache 的不透明性，支持 LRU 缓存等高级场景

### 派生管线（Pipeline Derivatives）

通过 `VK_PIPELINE_CREATE_DERIVATIVE_BIT` 和 `VK_PIPELINE_CREATE_ALLOW_DERIVATIVES_BIT` 标志，子管线可复用父管线的部分编译结果：
- 父管线必须设置 `VK_PIPELINE_CREATE_ALLOW_DERIVATIVES_BIT`
- 子管线必须设置 `VK_PIPELINE_CREATE_DERIVATIVE_BIT` 并指定 `basePipelineHandle` 或 `basePipelineIndex`
- 典型用途：同一着色器但不同混合模式/深度状态的多个管线变体

### Graphics Pipeline Library（VK_EXT_graphics_pipeline_library，Vulkan 1.3 核心）

将图形管线拆分为 4 个独立可编译的子管线库：
1. **Vertex Input Interface**：顶点输入 + 输入装配
2. **Pre-Rasterization Shaders**：VS/Tess/GS/Mesh/Task + 视口 + 光栅化
3. **Fragment Shader**：FS + 深度/模板
4. **Fragment Output**：颜色混合 + MSAA + Render Pass

各子库可独立创建（提前编译着色器），运行时通过 `VkPipelineLibraryCreateInfoKHR` 链接为完整管线。这解决了 D3D11 风格引擎无法在绘制前知道全部状态的问题（Valve Source 2 引擎的实际用例）。

### Extended Dynamic State（Vulkan 1.3 核心）

将越来越多的固定功能状态转为动态可修改（通过 `vkCmdSet*` 命令在 Command Buffer 录制时修改）：
- **Vulkan 1.0 动态状态**：viewport、scissor、lineWidth、depthBias、blendConstants、depthBounds、stencilCompareMask、stencilWriteMask、stencilReference
- **VK_EXT_extended_dynamic_state（1.3 核心）**：cullMode、frontFace、primitiveTopology、viewportWithCount、scissorWithCount、depthTestEnable、depthWriteEnable、depthCompareOp、depthBoundsTestEnable、stencilTestEnable、stencilOp
- **VK_EXT_extended_dynamic_state2（1.3 核心）**：rasterizerDiscardEnable、depthBiasEnable、primitiveRestartEnable、patchControlPoints
- **VK_EXT_extended_dynamic_state3**：logicOp、colorWriteEnable、depthClampEnable 等

动态状态越多 → PSO 变体越少 → 管线创建开销越低，但每次绘制时的状态设置命令略有增加。

## 版本差异

| 版本 | 变更 |
|------|------|
| Vulkan 1.0 | 基础 PSO 创建、Pipeline Cache、派生管线、9 个核心动态状态 |
| Vulkan 1.1 | 支持 shaderSubgroupExtendedTypes、支持 multiview 管线 |
| Vulkan 1.2 | 支持 8-bit storage、scalar block layouts、descriptor indexing（影响管线布局） |
| Vulkan 1.3 | VK_EXT_graphics_pipeline_library、VK_EXT_extended_dynamic_state/2/3 提升为核心；VK_KHR_pipeline_creation_cache_control 提升 |
| Vulkan 1.4 | VK_KHR_pipeline_binary 提升；maintenance5 简化管线创建参数；VK_EXT_shader_object 进一步模糊 PSO 边界 |

## 陷阱与注意事项

- **管线创建耗时**：首次创建（冷缓存）可能耗时数毫秒到数百毫秒（取决于着色器复杂度和 GPU），绝对不要在渲染热路径中同步创建管线。应在加载阶段预创建，或使用异步创建 + Pipeline Library 提前编译着色器部分。
- **PSO 变体爆炸**：每个状态组合都需要一个独立 PSO。例如 3 种着色器 × 2 种混合模式 × 2 种深度状态 = 12 个 PSO。使用 Extended Dynamic State 将部分状态转为动态可显著减少变体数量。
- **Pipeline Cache 跨驱动不兼容**：`vkGetPipelineCacheData` 输出的数据格式是驱动特定的，NVIDIA 的缓存不能用于 AMD 驱动，甚至同一驱动的大版本升级也可能导致缓存失效。
- **Descriptor Set Layout 兼容性**：绑定管线时，当前绑定的 Descriptor Set 必须与管线的 Pipeline Layout 兼容（set 0 到 N-1 逐集检查）。不兼容的绑定是未定义行为，验证层会报错。
- **Push Constant 大小限制**：`maxPushConstantsSize` 通常为 128 字节（部分 GPU 支持 256 字节），超出此限制的 Push Constant Range 会导致管线创建失败。
- **Render Pass 兼容性**：图形管线创建时绑定了 Render Pass，使用时必须绑定兼容的 Render Pass 或 Framebuffer。Vulkan 1.2+ 的 `VK_KHR_create_renderpass2` 和 Vulkan 1.3 的 Dynamic Rendering 可缓解此限制。
- **特化常量 vs Push Constant**：特化常量（Specialization Constants）在管线创建时编译进着色器，适合真正不变的常量（如数组大小）；Push Constant 在命令录制时设置，适合每帧/每绘制变化的少量数据。

## 使用模式

### 基础图形管线创建

```c
// 1. 着色器阶段
VkPipelineShaderStageCreateInfo stages[] = {
    { .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
      .stage = VK_SHADER_STAGE_VERTEX_BIT,
      .module = vertModule, .pName = "main" },
    { .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
      .stage = VK_SHADER_STAGE_FRAGMENT_BIT,
      .module = fragModule, .pName = "main" },
};

// 2. 固定功能状态
VkPipelineVertexInputStateCreateInfo vertexInput = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO,
    .vertexBindingDescriptionCount = 1,
    .pVertexBindingDescriptions = &(VkVertexInputBindingDescription){
        .binding = 0, .stride = sizeof(Vertex),
        .inputRate = VK_VERTEX_INPUT_RATE_VERTEX },
    .vertexAttributeDescriptionCount = 3,
    .pVertexAttributeDescriptions = (VkVertexInputAttributeDescription[]){
        {0, 0, VK_FORMAT_R32G32_SFLOAT, offsetof(Vertex, pos)},
        {1, 0, VK_FORMAT_R32G32B32_SFLOAT, offsetof(Vertex, color)},
        {2, 0, VK_FORMAT_R32G32_SFLOAT, offsetof(Vertex, uv)},
    },
};

VkPipelineInputAssemblyStateCreateInfo inputAssembly = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO,
    .topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST,
    .primitiveRestartEnable = VK_FALSE,
};

VkPipelineViewportStateCreateInfo viewportState = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO,
    .viewportCount = 1, .scissorCount = 1,
    // 使用动态状态时，pViewports/pScissors 可为 NULL
};

VkPipelineRasterizationStateCreateInfo rasterizer = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO,
    .depthClampEnable = VK_FALSE,
    .rasterizerDiscardEnable = VK_FALSE,
    .polygonMode = VK_POLYGON_MODE_FILL,
    .cullMode = VK_CULL_MODE_BACK_BIT,
    .frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE,
    .lineWidth = 1.0f,
};

VkPipelineMultisampleStateCreateInfo multisampling = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO,
    .rasterizationSamples = VK_SAMPLE_COUNT_1_BIT,
    .sampleShadingEnable = VK_FALSE,
};

VkPipelineColorBlendAttachmentState colorBlendAttachment = {
    .colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT |
                      VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT,
    .blendEnable = VK_FALSE,
};

VkPipelineColorBlendStateCreateInfo colorBlending = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO,
    .logicOpEnable = VK_FALSE,
    .attachmentCount = 1,
    .pAttachments = &colorBlendAttachment,
};

// 3. 动态状态
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT, VK_DYNAMIC_STATE_SCISSOR
};
VkPipelineDynamicStateCreateInfo dynamicState = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO,
    .dynamicStateCount = 2,
    .pDynamicStates = dynamicStates,
};

// 4. 管线布局
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {
    .sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
    .setLayoutCount = 1,
    .pSetLayouts = &descriptorSetLayout,
    .pushConstantRangeCount = 1,
    .pPushConstantRanges = &(VkPushConstantRange){
        .stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
        .offset = 0, .size = 128,
    },
};

// 5. 创建管线
VkGraphicsPipelineCreateInfo pipelineInfo = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .stageCount = 2, .pStages = stages,
    .pVertexInputState = &vertexInput,
    .pInputAssemblyState = &inputAssembly,
    .pViewportState = &viewportState,
    .pRasterizationState = &rasterizer,
    .pMultisampleState = &multisampling,
    .pColorBlendState = &colorBlending,
    .pDynamicState = &dynamicState,
    .layout = pipelineLayout,
    .renderPass = renderPass,
    .subpass = 0,
};

VkPipeline pipeline;
vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo,
                          NULL, &pipeline);
```

### Pipeline Cache 持久化

```c
// 创建时加载磁盘缓存
size_t cacheSize = 0;
FILE *f = fopen("pipeline_cache.bin", "rb");
if (f) {
    fseek(f, 0, SEEK_END);
    cacheSize = ftell(f);
    fseek(f, 0, SEEK_SET);
    void *cacheData = malloc(cacheSize);
    fread(cacheData, 1, cacheSize, f);
    fclose(f);

    VkPipelineCacheCreateInfo cacheInfo = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO,
        .initialDataSize = cacheSize,
        .pInitialData = cacheData,
    };
    vkCreatePipelineCache(device, &cacheInfo, NULL, &pipelineCache);
    free(cacheData);
}

// 退出时序列化缓存到磁盘
size_t dataSize = 0;
vkGetPipelineCacheData(device, pipelineCache, &dataSize, NULL);
void *data = malloc(dataSize);
vkGetPipelineCacheData(device, pipelineCache, &dataSize, data);
f = fopen("pipeline_cache.bin", "wb");
fwrite(data, 1, dataSize, f);
fclose(f);
free(data);
```

### 计算管线创建

```c
VkComputePipelineCreateInfo computeInfo = {
    .sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO,
    .stage = {
        .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
        .stage = VK_SHADER_STAGE_COMPUTE_BIT,
        .module = computeModule,
        .pName = "main",
    },
    .layout = computePipelineLayout,
};

VkPipeline computePipeline;
vkCreateComputePipelines(device, pipelineCache, 1, &computeInfo,
                         NULL, &computePipeline);
```

## Sources

- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#pipelines — Vulkan Spec §Pipelines: 管线创建、状态分组、派生机制、动态状态完整规范
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#pipelines-cache — Vulkan Spec §Pipeline Cache: 缓存创建、序列化、合并
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#descriptorsets-pipelinelayout — Vulkan Spec §Pipeline Layout: 资源布局、Push Constant、兼容性规则
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#vkCreateGraphicsPipelines — Vulkan Spec §vkCreateGraphicsPipelines: 创建参数、返回值、错误码
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/VkGraphicsPipelineCreateInfo.html — Vulkan Ref Pages: VkGraphicsPipelineCreateInfo 字段详细说明
- [P1] https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineLayoutCreateInfo.html — Vulkan Ref Pages: VkPipelineLayoutCreateInfo 字段说明
- [P1] https://docs.vulkan.org/guide/latest/push_constants.html — Vulkan Guide §Push Constants: 使用方式、偏移规则、布局兼容性
- [P1] https://www.khronos.org/blog/bringing-explicit-pipeline-caching-control-to-vulkan — Khronos Blog: VK_KHR_pipeline_binary 扩展动机与设计（2024-08）
- [P1] https://www.khronos.org/blog/reducing-draw-time-hitching-with-vk-ext-graphics-pipeline-library — Khronos Blog: VK_EXT_graphics_pipeline_library 设计动机与 Source 2 集成实践（Valve, 2022-03）
- [P1] https://www.vkguide.dev/docs/new_chapter_3/building_pipeline — Vulkan Guide: 图形管线创建实战教程
