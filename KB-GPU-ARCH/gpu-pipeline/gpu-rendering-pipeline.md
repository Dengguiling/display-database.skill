# GPU Rendering Pipeline

> GPU 图形渲染管线的完整阶段定义与数据流，涵盖 Vulkan/DX12 两大现代图形 API 的管线架构对比。

## Meta
- **ID:** KB-GPU-ARCH-gpu-pipeline-001
- **置信度:** 5
- **信源等级:** P1
- **验证日期:** 2026-04-12
- **适用版本:** Vulkan 1.0+ / DirectX 12 (D3D12)
- **关键词:** rendering pipeline, graphics pipeline, vertex shader, fragment shader, pixel shader, rasterization, input assembler, output merger, tessellation, geometry shader, mesh shader
- **前置知识:** （无，本条目为 GPU 架构模块入口）
- **关联条目:** （待创建：vulkan-pipeline, dx12-pipeline）

## 概述

GPU 渲染管线是将三维场景数据转化为二维屏幕像素的处理流水线，由多个固定功能（fixed-function）和可编程（programmable）阶段组成。现代图形 API（Vulkan / DX12）将管线抽象为**图形管线**（Graphics Pipeline）、**计算管线**（Compute Pipeline）和**光线追踪管线**（Ray Tracing Pipeline）三大类。图形管线是实时渲染的核心路径，从顶点数据输入到像素输出，经历输入装配、顶点处理、曲面细分、几何处理、光栅化、片段处理、输出合并等阶段。

## 架构/调用链

### Vulkan 图形管线阶段

```
Command Buffer: vkCmdDraw*()
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                    Graphics Pipeline                      │
│                                                          │
│  ┌──────────────┐                                        │
│  │Input Assembler│ ← 顶点数据装配为图元（点/线/三角形）     │
│  └──────┬───────┘                                        │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │Vertex Shader │ ← 逐顶点变换、光照计算                    │
│  └──────┬───────┘                                        │
│         ▼                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │Tessellation  │→ │Tessellator   │→ │Domain Shader │   │
│  │Control (HS)  │  │(Fixed-Func)  │  │(DS)          │   │
│  └──────────────┘  └──────────────┘  └──────┬───────┘   │
│                                              │           │
│  ┌──────────────┐                            │           │
│  │Geometry      │←───────────────────────────┘           │
│  │Shader (GS)   │                                        │
│  └──────┬───────┘                                        │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │Rasterization │ ← 图元→片段，插值属性                     │
│  └──────┬───────┘                                        │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │Fragment      │ ← 逐片段着色、纹理采样                    │
│  │Shader (FS)   │                                        │
│  └──────┬───────┘                                        │
│         ▼                                                │
│  ┌──────────────┐                                        │
│  │Color Blend & │ ← 深度/模板测试、颜色混合、写入 RT         │
│  │Framebuffer   │                                        │
│  └──────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

### Vulkan Mesh Shading 管线（替代路径）

```
vkCmdDrawMeshTasks*()
    │
    ▼
┌──────────────────┐
│  Task Shader     │ ← 可选，生成 mesh shader 任务组
│  (Compute-like)  │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Mesh Shader     │ ← 替代 IA+VS+TS+GS，显式生成图元和顶点
│  (Compute-like)  │
└──────┬───────────┘
       ▼
  Rasterization → Fragment Shader → Framebuffer
```

### DX12 图形管线阶段

```
Command List: Draw*()
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              D3D12 Graphics Pipeline                     │
│                                                          │
│  Input Assembler (IA)     ← 装配顶点数据为图元             │
│         ▼                                                │
│  Vertex Shader (VS)       ← 逐顶点处理（必须启用）          │
│         ▼                                                │
│  Hull Shader (HS)         ← 曲面细分控制着色器              │
│         ▼                                                │
│  Tessellator (TS)         ← 固定功能曲面细分单元            │
│         ▼                                                │
│  Domain Shader (DS)       ← 曲面细分域着色器               │
│         ▼                                                │
│  Geometry Shader (GS)     ← 图元级处理，可生成/丢弃图元      │
│         ▼                                                │
│  Stream Output (SO)       ← 顶点数据回写到内存（可选）       │
│         ▼                                                │
│  Rasterizer (RS)          ← 裁剪、透视除法、视口变换、光栅化  │
│         ▼                                                │
│  Pixel Shader (PS)        ← 逐像素/采样点着色               │
│         ▼                                                │
│  Output Merger (OM)       ← 深度/模板测试、混合、输出写入     │
└─────────────────────────────────────────────────────────┘
```

## 关键接口

### Vulkan 管线创建

- `vkCreateGraphicsPipelines(device, pipelineCache, createInfoCount, pCreateInfos, pAllocator, pPipelines)` — 创建图形管线
  - 参数: `VkDevice device`, `VkPipelineCache pipelineCache`, `uint32_t createInfoCount`, `const VkGraphicsPipelineCreateInfo* pCreateInfos`, `const VkAllocationCallbacks* pAllocator`, `VkPipeline* pPipelines`
  - 返回: `VkResult` (VK_SUCCESS / VK_ERROR_OUT_OF_HOST_MEMORY / VK_ERROR_OUT_OF_DEVICE_MEMORY / VK_ERROR_INVALID_SHADER_NV)
  - 源码: Vulkan Specification §9.5 (https://docs.vulkan.org/spec/latest/chapters/pipelines.html)
  - 注意: 管线创建是重量级操作，应使用 Pipeline Cache 或 Pipeline Library 复用

- `struct VkGraphicsPipelineCreateInfo` — 图形管线创建描述
  - 关键字段:
    - `VkPipelineShaderStageCreateInfo* pStages` — 着色器阶段数组
    - `VkPipelineVertexInputStateCreateInfo* pVertexInputState` — 顶点输入格式
    - `VkPipelineInputAssemblyStateCreateInfo* pInputAssemblyState` — 图元拓扑
    - `VkPipelineViewportStateCreateInfo* pViewportState` — 视口与裁剪
    - `VkPipelineRasterizationStateCreateInfo* pRasterizationState` — 光栅化状态
    - `VkPipelineMultisampleStateCreateInfo* pMultisampleState` — 多重采样
    - `VkPipelineDepthStencilStateCreateInfo* pDepthStencilState` — 深度/模板
    - `VkPipelineColorBlendStateCreateInfo* pColorBlendState` — 颜色混合
    - `VkPipelineDynamicStateCreateInfo* pDynamicState` — 动态状态
    - `VkPipelineLayout layout` — 管线布局（描述符集 + push constant）
    - `VkRenderPass renderPass` — 渲染通道（Vulkan 1.2+ 可设 VK_NULL_HANDLE）
    - `uint32_t subpass` — 子通道索引

### DX12 管线创建

- `ID3D12Device::CreateGraphicsPipelineState(pDesc, riid, ppPipelineState)` — 创建图形管线状态对象
  - 参数: `const D3D12_GRAPHICS_PIPELINE_STATE_DESC* pDesc`, `REFIID riid`, `void** ppPipelineState`
  - 返回: `HRESULT` (S_OK / E_OUTOFMEMORY)
  - 源码: https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-creategraphicspipelinestate
  - 注意: PSO 创建涉及驱动编译着色器，开销大；应使用 PSO Cache

- `struct D3D12_GRAPHICS_PIPELINE_STATE_DESC` — DX12 图形管线描述
  - 关键字段:
    - `ID3D12RootSignature* pRootSignature` — 根签名
    - `D3D12_SHADER_BYTECODE VS/HS/DS/GS/PS` — 各阶段着色器字节码
    - `D3D12_BLEND_DESC BlendState` — 混合状态
    - `D3D12_RASTERIZER_DESC RasterizerState` — 光栅化状态
    - `D3D12_DEPTH_STENCIL_DESC DepthStencilState` — 深度模板状态
    - `D3D12_INPUT_LAYOUT_DESC InputLayout` — 顶点输入布局
    - `D3D12_INDEX_BUFFER_STRIP_CUT_VALUE IBStripCutValue` — 索引缓冲区 strip cut 值
    - `D3D12_PRIMITIVE_TOPOLOGY_TYPE PrimitiveTopologyType` — 图元拓扑类型
    - `UINT NumRenderTargets` — 渲染目标数量
    - `DXGI_FORMAT RTVFormats[8]` — 渲染目标格式
    - `DXGI_FORMAT DSVFormat` — 深度/模板视图格式

## 实现要点

### 各阶段功能对比

| 阶段 | Vulkan 名称 | DX12 名称 | 类型 | 功能 |
|------|------------|-----------|------|------|
| 输入装配 | Input Assembler | Input Assembler (IA) | 固定功能 | 从缓冲区读取顶点数据，按拓扑装配图元 |
| 顶点着色 | Vertex Shader | Vertex Shader (VS) | 可编程 | 逐顶点变换（MVP 矩阵）、法线变换、属性传递 |
| 细分控制 | Tessellation Control | Hull Shader (HS) | 可编程 | 控制细分因子，变换控制点 |
| 细分器 | Tessellation | Tessellator (TS) | 固定功能 | 根据 HS 输出的因子生成新顶点 |
| 细分域 | Tessellation Evaluation | Domain Shader (DS) | 可编程 | 在细分域上计算顶点位置 |
| 几何着色 | Geometry Shader | Geometry Shader (GS) | 可编程 | 图元级操作，可增删图元 |
| 流输出 | — | Stream Output (SO) | 固定功能 | 将顶点数据回写到缓冲区 |
| 光栅化 | Rasterization | Rasterizer (RS) | 固定功能 | 裁剪、透视除法、视口变换、三角形设置、插值 |
| 片段着色 | Fragment Shader | Pixel Shader (PS) | 可编程 | 逐像素/采样点着色、纹理采样 |
| 输出合并 | Color Blend & FB | Output Merger (OM) | 固定功能 | 深度/模板测试、颜色混合、逻辑操作、写入 RT |

### 管线状态模型差异

**Vulkan**: 管线状态在创建时（`vkCreateGraphicsPipelines`）几乎全部固定，运行时仅能修改通过 `VkPipelineDynamicStateCreateInfo` 声明的少量状态（视口、裁剪矩形、线宽、深度偏移等）。这使驱动可以在管线创建时进行深度优化。

**DX12**: PSO（Pipeline State Object）同样在创建时固定大部分状态，但通过 `ID3D12GraphicsCommandList::SetPipelineState()` 切换 PSO 的开销低于 Vulkan 的管线绑定（两者设计理念相同，均追求最小化运行时状态切换）。

### Mesh Shading 管线

Vulkan (VK_EXT_mesh_shader / Vulkan 1.3+) 和 DX12 (DX12 Ultimate / Shader Model 6.5+) 均引入了 Mesh Shading 管线，作为传统 IA→VS→TS→GS 路径的替代：
- **Task Shader**（Vulkan）/ **Amplification Shader**（DX12）：可选阶段，类似 compute shader，生成 Task/Mesh Shader 工作组
- **Mesh Shader**：替代 IA+VS+TS+GS，显式生成顶点和图元，以 compute-like 模型运行
- 优势：避免固定功能 IA 瓶颈，支持 GPU-driven 渲染、集群剔除（cluster culling）、更灵活的几何生成

## 版本差异

| 版本 | 变更 |
|------|------|
| Vulkan 1.0 | 基础图形管线（IA→VS→TS→GS→FS→FB） |
| Vulkan 1.1 | 新增 `vkCmdDrawIndirectCount`，支持设备端绘制数量 |
| Vulkan 1.2 | 支持无 RenderPass 创建管线（`VK_EXT_graphics_pipeline_library` 前身） |
| Vulkan 1.3 | 整合 `VK_EXT_extended_dynamic_state`，更多动态状态 |
| VK_EXT_mesh_shader | 引入 Task Shader + Mesh Shader 替代路径 |
| VK_KHR_fragment_shading_rate | 引入片段着色率（Variable Rate Shading） |
| D3D12 (DX12) | 基础管线与 D3D11.3 功能规范一致 |
| DX12 Ultimate | 引入 Mesh Shader、Amplification Shader、Ray Tracing |
| SM 6.5+ | Mesh Shader / Amplification Shader 支持 |
| SM 6.6+ | 引入 Work Graphs（计算管线增强） |

## 陷阱与注意事项

1. **Vulkan 管线创建开销极大**：涉及驱动编译/优化着色器，首次创建可能耗时数十毫秒。必须使用 `VkPipelineCache` 跨帧/跨运行复用，或使用 `VK_KHR_pipeline_library` 延迟链接
2. **DX12 PSO 同样昂贵**：`CreateGraphicsPipelineState` 会触发着色器编译，应预创建所有需要的 PSO，运行时仅做 `SetPipelineState`
3. **Vulkan 动态状态必须显式声明**：未在 `VkPipelineDynamicStateCreateInfo` 中列出的状态在管线创建时固定，运行时修改无效
4. **Geometry Shader 性能陷阱**：GS 在多数 GPU 架构上性能较差（NVIDIA/AMD 均如此），应优先考虑 Mesh Shader 或 Compute Shader 替代方案
5. **Tessellation 的 HS/DS 调用频率**：HS 每个 patch 调用一次，DS 每个细分顶点调用一次。细分因子过高会导致 DS 成为瓶颈
6. **Vulkan Cluster Culling Shader 与 Mesh Shader 互斥**：`VK_EXT_cluster_culling_shader` 和 `VK_EXT_mesh_shader` 不能在同一管线中同时使用

## 使用模式

### Vulkan 最小图形管线创建

```c
// 定义着色器阶段
VkPipelineShaderStageCreateInfo stages[] = {
    { .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
      .stage = VK_SHADER_STAGE_VERTEX_BIT,
      .module = vsModule, .pName = "main" },
    { .sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
      .stage = VK_SHADER_STAGE_FRAGMENT_BIT,
      .module = fsModule, .pName = "main" },
};

// 创建管线（需填充所有状态结构体）
VkGraphicsPipelineCreateInfo info = {
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    .stageCount = 2, .pStages = stages,
    .pVertexInputState = &vertexInput,
    .pInputAssemblyState = &inputAssembly,
    .pViewportState = &viewportState,
    .pRasterizationState = &rasterState,
    .pMultisampleState = &multisampleState,
    .pDepthStencilState = &depthStencilState,
    .pColorBlendState = &colorBlendState,
    .pDynamicState = &dynamicState,
    .layout = pipelineLayout,
    .renderPass = renderPass,
    .subpass = 0,
};

VkPipeline pipeline;
vkCreateGraphicsPipelines(device, cache, 1, &info, NULL, &pipeline);
```

### DX12 最小 PSO 创建

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC desc = {};
desc.pRootSignature = rootSignature;
desc.VS = { vsBytecode, vsSize };
desc.PS = { psBytecode, psSize };
desc.BlendState = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
desc.RasterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
desc.DepthStencilState = CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT);
desc.SampleMask = UINT_MAX;
desc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
desc.NumRenderTargets = 1;
desc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
desc.DSVFormat = DXGI_FORMAT_D24_UNORM_S8_UINT;
desc.SampleDesc.Count = 1;

ID3D12PipelineState* pso;
device->CreateGraphicsPipelineState(&desc, IID_PPV_ARGS(&pso));
```

## Sources

- [P1] https://docs.vulkan.org/spec/latest/chapters/pipelines.html — Vulkan Specification §9 Pipelines: 图形管线阶段定义、管线创建、Pipeline Cache、Pipeline Library
- [P1] https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm — D3D11.3 Functional Specification §2 Rendering Pipeline Overview: 各阶段功能描述（IA/VS/HS/TS/DS/GS/SO/RS/PS/OM）
- [P1] https://learn.microsoft.com/en-us/windows/win32/direct3d12/pipelines-and-shaders-with-directx-12 — Pipelines and Shaders with Direct3D 12: PSO 创建与使用
- [P1] https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_graphics_pipeline_state_desc — D3D12_GRAPHICS_PIPELINE_STATE_DESC 结构体定义
- [P1] https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#streaming-multiprocessors — CUDA C++ Programming Guide: Streaming Multiprocessor 架构（GPU 硬件执行模型参考）
