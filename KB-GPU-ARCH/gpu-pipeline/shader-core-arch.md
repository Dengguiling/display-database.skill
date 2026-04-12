# GPU Shader Core 架构

> 现代 GPU 的核心计算单元，实现 SIMT 执行模型，通过 Warp/Wavefront 调度器管理大规模并行线程。

## Meta
- **ID:** KB-GPU-ARCH-gpu-pipeline-002
- **置信度:** 5
- **信源等级:** P1
- **验证日期:** 2026-04-13
- **适用版本:** NVIDIA Ada Lovelace / Hopper, AMD RDNA 3 / CDNA 3, Intel Xe-HPG / Xe-HPC
- **关键词:** shader core, SIMT, warp, wavefront, SM, CU, streaming multiprocessor, compute unit, register file, shared memory, occupancy, tensor core, RT core
- **前置知识:** [→ gpu-rendering-pipeline]
- **关联条目:** [→ gpu-memory-hierarchy]

## 概述

GPU Shader Core 是现代 GPU 的核心计算单元，实现了 SIMT（Single Instruction, Multiple Threads）执行模型。三大厂商各有命名：NVIDIA 称为 Streaming Multiprocessor（SM），AMD 称为 Compute Unit（CU），Intel 称为 Xe Core（Xe-HPC）或 Execution Unit（Xe-LP）。Shader Core 内部包含 ALU 阵列、寄存器文件、共享内存和 Warp/Wavefront 调度器，通过在大量并行线程间零成本切换来隐藏内存延迟。理解 Shader Core 架构是 GPU 驱动开发和性能优化的基础。

## 架构/调用链

### SIMT 执行模型

```
应用程序线程 (Thread)
    │
    ▼
工作组 (Workgroup / Thread Group)
    │  CUDA: Block (最多 1024 线程)
    │  Vulkan: Workgroup (gl_WorkGroupSize)
    ▼
线程束 (Warp / Wavefront / Subgroup)
    │  NVIDIA: Warp = 32 线程
    │  AMD: Wavefront = 64 线程
    │  Vulkan: Subgroup (大小由设备决定)
    ▼
SIMD 执行单元
    │  NVIDIA: 32-wide FP32 ALU
    │  AMD: 32-wide SIMD32 (RDNA 3)
    │  Intel: 16-wide SIMD16 (Xe-HPG)
    ▼
ALU 阵列 (per SM/CU/Xe Core)
```

### NVIDIA SM 内部架构 (Ada Lovelace)

```
┌─────────────────────────────────────────────────┐
│              Streaming Multiprocessor (SM)       │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ Warp Scheduler│  │ Dispatch Unit│  ×4 组       │
│  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                       │
│  ┌──────▼─────────────────▼──────┐               │
│  │     128 FP32 CUDA Cores       │               │
│  │     64 FP32/INT32 (dual-issue)│               │
│  │     4 Tensor Cores            │               │
│  │     1 RT Core                 │               │
│  └──────────────┬────────────────┘               │
│                 │                                 │
│  ┌──────────────▼────────────────┐               │
│  │     Register File (256 KB)    │               │
│  │     65536 × 32-bit registers  │               │
│  └──────────────┬────────────────┘               │
│                 │                                 │
│  ┌──────────────▼────────────────┐               │
│  │     Shared Memory (100 KB)    │               │
│  │     L1 Cache (128 KB)         │  可配置比例    │
│  └───────────────────────────────┘               │
└─────────────────────────────────────────────────┘
```

### AMD CU 内部架构 (RDNA 3)

```
┌─────────────────────────────────────────────────┐
│              Compute Unit (CU)                   │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ Wavefront    │  │ Wavefront    │  ×2 组       │
│  │ Scheduler    │  │ Scheduler    │              │
│  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                       │
│  ┌──────▼─────────────────▼──────┐               │
│  │     SIMD32 × 2 (64 ALU)       │               │
│  │     1 AI Accelerator          │               │
│  │     1 Ray Accelerator         │               │
│  └──────────────┬────────────────┘               │
│                 │                                 │
│  ┌──────────────▼────────────────┐               │
│  │     VGPRs (128 KB)            │               │
│  │     SGPRs (6 KB)              │               │
│  └──────────────┬────────────────┘               │
│                 │                                 │
│  ┌──────────────▼────────────────┐               │
│  │     LDS (128 KB)              │               │
│  │     L0 Cache (32 KB)          │               │
│  └───────────────────────────────┘               │
└─────────────────────────────────────────────────┘
```

### Intel Xe Core 内部架构 (Xe-HPG)

```
┌─────────────────────────────────────────────────┐
│              Xe Core (Xe-HPG)                    │
│                                                  │
│  ┌──────────────────────────────────┐            │
│  │     16 EU × SIMD16 = 256 ALU     │            │
│  │     Thread Dispatch Unit         │            │
│  └──────────────┬───────────────────┘            │
│                 │                                 │
│  ┌──────────────▼───────────────────┐            │
│  │     Register File (per Thread)   │            │
│  │     4096 × 32-bit registers      │            │
│  └──────────────┬───────────────────┘            │
│                 │                                 │
│  ┌──────────────▼───────────────────┐            │
│  │     Shared Local Memory (64 KB)  │            │
│  │     L1 Cache (128 KB/Slice)      │            │
│  └──────────────────────────────────┘            │
└─────────────────────────────────────────────────┘
```

## 关键接口

### 内核驱动接口

#### NVIDIA (nouveau)
- `nvkm_falcon_ctor()` — Falcon 微控制器初始化（管理 SM 固件加载）
  - 源码: `drivers/gpu/drm/nouveau/nvkm/engine/falcon.c`
  - 注意: Falcon 是 NVIDIA GPU 上的通用微控制器，负责管理 SM 的固件和调度

- `nvkm_grctx_generate()` — 生成 GPU 上下文（SM 状态保存/恢复）
  - 源码: `drivers/gpu/drm/nouveau/nvkm/engine/gr/ctx*.c`
  - 注意: 上下文切换时需要保存/恢复 SM 的寄存器文件和调度器状态

#### AMD (amdgpu)
- `amdgpu_gfx_select_cu_sh_mask()` — CU/SH 屏蔽配置（控制哪些 CU 可用）
  - 源码: `drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c` (RDNA 2/3)
  - 注意: 用于 CU 级别的电源管理和故障隔离

- `amdgpu_gfx_compute_queue_acquire()` — 计算队列获取（分配 CU 资源给计算任务）
  - 源码: `drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c`
  - 注意: 图形和计算任务共享 CU 资源，驱动负责仲裁

#### Intel (xe/drm)
- `xe_hw_engine_enable()` — 启用硬件引擎（EU 集群）
  - 源码: `drivers/gpu/drm/xe/xe_hw_engine.c`
  - 注意: Xe 驱动使用 hw_engine 抽象管理 EU 集群

### 数据结构

- `struct drm_sched_entity` — DRM 调度器实体（代表一个提交上下文）
  - 关键字段:
    - `rq` — 运行队列（关联到特定硬件队列）
    - `priority` — 调度优先级
  - 源码: `include/drm/gpu_scheduler.h`

## 实现要点

### Warp Divergence（线程分歧）

当 Warp 内的线程走不同分支时，硬件会序列化执行——先执行一个分支的线程（其他线程 masked off），再执行另一个分支。这是 GPU 性能损失的主要来源之一。

```
Warp 32 线程执行 if-else:
┌─────────────────────────────────────────┐
│ if (tid < 16) {    ← 16 线程执行       │
│     result = a * b;                     │
│ } else {             ← 16 线程执行       │
│     result = c + d;                     │
│ }                                        │
│ 实际执行: 2 次序列化（而非 1 次并行）     │
└─────────────────────────────────────────┘
```

### Occupancy（占用率）

Occupancy 是衡量 SM/CU 资源利用率的指标，受三个因素制约：

| 限制因素 | 说明 | 典型值 |
|----------|------|--------|
| **寄存器** | 每个 SM 的寄存器总量固定，每个线程占用越多，可并发 Warp 越少 | 256 KB/SM (NVIDIA) |
| **共享内存** | Block 请求的共享内存越多，可并发 Block 越少 | 100 KB/SM (NVIDIA) |
| **线程数** | 每个 SM 最大并发线程数有上限 | 1536-2048 线程/SM |

Occupancy 计算公式（NVIDIA）：
```
Occupancy = min(活跃 Warp 数 / 最大 Warp 数, 1.0)
最大 Warp 数 = SM 寄存器总量 / (每线程寄存器数 × Warp 大小)
```

### Tensor Core（张量核心）

| 代际 | 架构 | 矩阵运算 | 精度支持 | 应用 |
|------|------|----------|----------|------|
| Gen 3 | Ampere (A100) | 16×16×16 | FP16/BF16/TF32/INT8/INT4 | AI 训练/推理 |
| Gen 4 | Hopper (H100) | FP8/FP16/BF16/TF32/INT8 | FP8 新增 | 大模型训练 |
| Gen 4 | Ada (RTX 4090) | FP16/BF16/TF32/INT8 | 无 FP8 | 游戏/创作 |
| Gen 5 | Blackwell (B200) | FP4/FP6/FP8/FP16/BF16 | FP4/FP6 新增 | 下一代训练 |

### RT Core（光线追踪核心）

| 代际 | 架构 | 功能 | 性能 |
|------|------|------|------|
| Gen 1 | Turing (RTX 20) | BVH 遍历 + 光线-三角形相交 | ~10 Giga-rays/s |
| Gen 2 | Ampere (RTX 30) | 运动模糊 + 透明度 | ~19 Giga-rays/s |
| Gen 3 | Ada (RTX 40) | SER (Shader Execution Reordering) | ~78 Giga-rays/s |
| Gen 4 | Blackwell (RTX 50) | 增强压缩 + 更快遍历 | ~100+ Giga-rays/s |

## 版本差异

| 架构 | 代际 | SM/CU 变更 |
|------|------|------------|
| NVIDIA Maxwell | GM200 | 无 Tensor/RT Core，纯 FP32 CUDA Core |
| NVIDIA Pascal | GP100 | 引入 FP16 (2:1 rate) |
| NVIDIA Volta | GV100 | 首次引入 Tensor Core (Gen 1) |
| NVIDIA Turing | TU102 | 首次引入 RT Core (Gen 1) |
| NVIDIA Ampere | GA100 | Tensor Core Gen 3，稀疏化支持 |
| NVIDIA Ada | AD102 | Tensor Core Gen 4，SER 光追重排序 |
| NVIDIA Hopper | GH100 | Tensor Core Gen 4，FP8，Transformer Engine |
| NVIDIA Blackwell | GB200 | Tensor Core Gen 5，FP4/FP6，第二代 Transformer Engine |
| AMD GCN | Vega | 无 AI 加速器，纯 ALU |
| AMD RDNA 2 | Navi 21 | 引入 Ray Accelerator |
| AMD RDNA 3 | Navi 31 | 引入 AI Accelerator，Chiplet 设计 |
| Intel Xe-LP | Tiger Lake | 96 EU，无 AI 加速 |
| Intel Xe-HPG | Alchemist | 256 EU，Ray Tracing Unit |
| Intel Xe-HPC | Ponte Vecchio | 4096 EU，矩阵扩展引擎 |

## 陷阱与注意事项

- **寄存器溢出（Register Spilling）：** 当 shader 使用的寄存器超过 SM 可分配量时，编译器会将溢出寄存器存储到本地内存（实际映射到 L1/L2 缓存），导致显著性能下降。CUDA 中可通过 `__launch_bounds__` 提示编译器优化寄存器分配。
- **Bank Conflict：** 共享内存被分为 32 个 bank（NVIDIA），同一 Warp 内的线程访问同一 bank 的不同地址时会发生序列化。通过 padding 或重排访问模式可避免。
- **Warp Divergence 不可消除：** 编译器无法完全消除 divergence（如循环边界不整齐时），应从算法层面减少分支。
- **Occupancy 不是越高越好：** 过高 occupancy 可能导致 L1/共享内存容量不足，反而降低性能。需要通过 profiling 工具（Nsight Compute、ROCm Profiler）找到最优 occupancy。
- **AMD Wavefront 64 线程：** AMD 的 Wavefront 是 64 线程（NVIDIA 是 32），这意味着同样的 divergence 在 AMD 上可能影响更多线程。RDNA 3 引入了 Wave32 模式可缓解此问题。

## 使用模式

### CUDA: 控制 Occupancy

```c
// 通过 __launch_bounds__ 提示编译器
__global__ __launch_bounds__(256, 4)  // 每块 256 线程，最少 4 块并发
void matmul_kernel(float* C, const float* A, const float* B, int N) {
    // kernel 实现...
}

// 运行时查询 occupancy
int blockSize = 256;
int minGridSize, gridSize;
cudaOccupancyMaxPotentialBlockSize(&minGridSize, &gridSize,
                                    matmul_kernel, 0, 0);
```

### Vulkan: 查询 Subgroup 信息

```c
// 查询 subgroup 大小（对应 Warp/Wavefront 大小）
VkPhysicalDeviceSubgroupProperties subgroupProps = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SUBGROUP_PROPERTIES,
};
vkGetPhysicalDeviceProperties2(physicalDevice, &props2);

// subgroupProps.subgroupSize = 32 (NVIDIA) 或 64 (AMD)
// 在 shader 中使用:
// gl_SubgroupSize, gl_SubgroupID, gl_SubgroupInvocationID
```

### 驱动: CU 资源仲裁 (AMD)

```c
// amdgpu 驱动中配置 CU 屏蔽（用于电源管理或故障隔离）
amdgpu_gfx_select_cu_sh_mask(adev, cu_mask, se_mask);
// cu_mask: 位掩码，每个 bit 代表一个 CU
// se_mask: Shader Engine 掩码
```

## Sources

- [P1] https://www.nvidia.com/en-us/geforce/technologies/ada-lovelace-architecture/ — NVIDIA Ada Lovelace Architecture Whitepaper: SM 结构、Tensor Core Gen 4、RT Core Gen 3
- [P1] https://gpuopen.com/learn/rdna-3-architecture/ — AMD RDNA 3 Architecture (GPUOpen): CU 结构、Wavefront 调度、AI Accelerator
- [P1] https://en.wikichip.org/wiki/intel/microarchitectures/xe_hpg — Intel Xe-HPG Microarchitecture (WikiChip): EU 阵列、线程调度
- [P1] https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#streaming-multiprocessors — CUDA C++ Programming Guide §Streaming Multiprocessors: SM 资源限制、Occupancy 计算
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/nouveau — drm/nouveau: Falcon 微控制器、SM 上下文管理
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/amd — drm/amd: CU 屏蔽配置、计算队列管理
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/xe — drm/xe: Xe 硬件引擎管理
- [P0] include/drm/gpu_scheduler.h — DRM GPU Scheduler: 调度实体和运行队列定义
