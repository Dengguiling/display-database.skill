# GPU Shader Core 架构

> **条目ID:** KB-GPU-ARCH-gpu-pipeline-002
> **模块:** KB-GPU-ARCH / gpu-pipeline
> **置信度:** 5/5（基于 NVIDIA/AMD/Intel 官方架构文档 + 内核驱动源码 + 技术分析文章）
> **时效性:** 2026-04-12 | **信源:** NVIDIA, AMD, Intel, kernel.org, abhik.ai
> **关联:** [gpu-rendering-pipeline](./gpu-rendering-pipeline.md) | [gpu-memory-hierarchy](./gpu-memory-hierarchy.md)

---

## 核心摘要

GPU Shader Core 是现代 GPU 的核心计算单元——它实现了 SIMT（Single Instruction, Multiple Threads）执行模型，通过 Warp/Wavefront 调度器在大量并行线程间零成本切换，以隐藏内存延迟。三大 GPU 厂商各有不同的架构命名和实现策略：NVIDIA 的 Streaming Multiprocessor（SM）、AMD 的 Compute Unit（CU）、Intel 的 Xe Core（Xe-HPC）或 Execution Unit（Xe-LP）。理解 Shader Core 的内部结构（ALU 阵列、寄存器文件、共享内存、Warp 调度器）是 GPU 驱动开发和性能优化的基础。

---

## 1. SIMT 执行模型

### 核心概念

| 概念 | 说明 |
|------|------|
| **SIMT** | 单指令多线程。一组线程（Warp/Wavefront）共享一个指令指针，执行相同指令但操作不同数据 |
| **Warp** | NVIDIA 术语，32 线程一组 |
| **Wavefront** | AMD 术语，64 线程一组（RDNA）或 64 线程（CDNA） |
| **SIMD** | 单指令多数据。硬件执行单元，NVIDIA=32-wide, AMD=32-wide (SIMD32) |
| **Subgroup** | Vulkan/SPIR-V 术语，与 Warp/Wavefront 对应 |

### Warp Divergence（线程分歧）

```
Warp 32 线程执行 if-else:
┌─────────────────────────────────────────┐
│ if (tid < 16) {    ← 16 线程执行       │
│     do_A();        ← 16 线程空闲       │
│ } else {           ← 16 线程空闲       │
│     do_B();        ← 16 线程执行       │
│ }                  ← 串行化！吞吐量减半 │
└─────────────────────────────────────────┘
```

- 最坏情况：32-way divergence → 吞吐量降至 1/32
- **优化策略：** 将分支条件按 warp 对齐，避免 warp 内分歧

---

## 2. 三大厂商架构对比

### 总览

| 维度 | NVIDIA (Ada Lovelace) | AMD (RDNA 3) | Intel (Xe-HPG) |
|------|----------------------|-------------|----------------|
| 计算单元名 | SM (Streaming Multiprocessor) | CU (Compute Unit) | Xe Core |
| 线程组 | Warp (32 threads) | Wavefront (64 threads) | Subgroup (variable) |
| SIMD 宽度 | 32-wide × 4 组 = 128 FP32 | SIMD32 × 2 组 = 64 FP32 | SIMD16 × 8 组 = 128 FP32 |
| FP32 CUDA/核心 | 128 per SM | 64 per CU | 256 per Xe Core (Battlemage) |
| Tensor 单元 | Tensor Core (4×4×4) | Matrix Core (WMMA) | XMX (矩阵引擎) |
| 光追单元 | RT Core (BVH + Ray-Tri) | Ray Accelerator | Ray Tracing Unit |
| Warp 调度器 | 4 个 | 2 个 (Wave Scheduler) | Thread Dispatch |
| 寄存器文件 | 256 KB (65,536 × 32-bit) | 128 KB (per CU) | 192 KB (per Xe Core) |
| 共享内存 | 0-128 KB (可配置) | 32-128 KB (per CU) | 64-128 KB (per Xe Core) |
| L1 Cache | 与共享内存共享 128 KB | 32 KB (独立) | 128 KB (独立) |
| 最大驻留线程 | 1,536 | 1,024 (RDNA3) | ~1,024 |
| 最大驻留 Warp | 48 | 32 | ~64 |

### NVIDIA SM 架构详解

```
┌─────────────────────────────────────────────────────┐
│                    Streaming Multiprocessor          │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────┐│
│  │Warp Sched│  │Warp Sched│  │Warp Sched│  │Warp ││
│  │   #0     │  │   #1     │  │   #2     │  │Sched││
│  │  + Disp  │  │  + Disp  │  │  + Disp  │  │ #3  ││
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──┬──┘│
│       │              │              │           │   │
│  ┌────▼──────────────▼──────────────▼───────────▼──┐│
│  │              128 FP32 CUDA Cores                ││
│  │  (4 × SIMD32, 每组 32 个 FP32 ALU)             ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │              4 Tensor Cores                     ││
│  │  (4×4×4 FP16/BF16/TF32/INT8/INT4)              ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │              1 RT Core                          ││
│  │  (BVH Traversal + Ray-Triangle Intersection)    ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │         Register File (256 KB)                  ││
│  │         65,536 × 32-bit registers               ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Shared Memory / L1 Cache (128 KB)            ││
│  │    (可配置比例: 0/16/32/64/100 KB shared)       ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Load/Store Units (32-bit address, 128-bit data)│
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Special Function Units (SFU)                 ││
│  │    (sin, cos, rsqrt, exp, log)                 ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

### AMD RDNA 3 CU 架构详解

```
┌─────────────────────────────────────────────────────┐
│                  Compute Unit (CU)                   │
│                                                     │
│  ┌──────────┐  ┌──────────┐                         │
│  │ Wave     │  │ Wave     │                         │
│  │ Scheduler│  │ Scheduler│                         │
│  │   #0     │  │   #1     │                         │
│  └────┬─────┘  └────┬─────┘                         │
│       │              │                               │
│  ┌────▼──────────────▼──────────────────────────────┐│
│  │          SIMD32 × 2 = 64 FP32 Cores              ││
│  │  (每组 SIMD32: 16 FP32 + 16 FP32, 共 64)        ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          2 Matrix Cores (AI)                    ││
│  │  (WMMA: FP16/BF16/INT8)                        ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          Register File (128 KB)                 ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          LDS (Local Data Share) 32-128 KB       ││
│  │          (CU 内线程共享)                         ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          L1 Data Cache (32 KB)                  ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          SALU (Scalar ALU)                      ││
│  │          (标量运算，wavefront 内广播)            ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │          Ray Accelerator                        ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

### Intel Xe-HPG Xe Core 架构详解

```
┌─────────────────────────────────────────────────────┐
│                     Xe Core                         │
│                                                     │
│  ┌─────────────────────────────────────────────────┐│
│  │    Thread Dispatch + Instruction Cache          ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    8 × SIMD16 = 128 FP32 EUs                   ││
│  │    (每个 EU: 2 FPU pipelines + 1 INT pipeline)  ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    XMX Matrix Engines (8 个)                    ││
│  │    (INT8/FP16/BF16 矩阵乘加)                   ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Register File (192 KB)                       ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Shared Local Memory (64-128 KB)              ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    L1 Cache (128 KB) + Data Port                ││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │    Ray Tracing Unit                             ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

---

## 3. Warp 调度与延迟隐藏

### 零成本上下文切换

```
时钟周期:  0    1    2    3    4    5    6    7    8    9
Warp 0:   [ALU][ALU][MEM──── 等待 ────][ALU][ALU]
Warp 1:        [ALU][ALU][ALU][ALU][ALU][ALU][ALU][ALU]
Warp 2:             [ALU][ALU][ALU][ALU][ALU][ALU][ALU]
                    ↑ Warp 0 等待内存时，调度器切换到 Warp 1
```

- 所有线程状态存储在寄存器文件中，无需保存/恢复
- 调度器每周期选择一个就绪的 Warp 发射指令
- **关键：** 足够多的活跃 Warp 才能充分隐藏内存延迟

### Occupancy（占用率）

| 资源限制 | 说明 | 影响 |
|----------|------|------|
| 寄存器/线程 | 每线程 0-255 寄存器，SM 总共 65,536 | 寄存器越多，并发线程越少 |
| 共享内存/Block | SM 共享内存有限 | 共享内存越多，并发 Block 越少 |
| 线程/Block | Block 大小上限 1,024 | Block 太小浪费调度槽 |
| 最大 Warp 数 | 48 Warp/SM (NVIDIA) | 硬件上限 |

**经验法则：**
- 内存密集型：目标 Occupancy 50-75%（需要更多 Warp 隐藏延迟）
- 计算密集型：目标 Occupancy 25-50%（足够 ALU 利用率即可）

---

## 4. 专用单元

### Tensor Core / Matrix Core / XMX

| 厂商 | 名称 | 操作 | 吞吐量（相对 FP32） |
|------|------|------|---------------------|
| NVIDIA | Tensor Core | D = A×B + C (4×4×4) | ~8× FP32 |
| AMD | Matrix Core | WMMA (16×16×16) | ~4× FP32 |
| Intel | XMX | D = A×B + C (8×8×8) | ~8× FP32 |

**支持精度：** FP16, BF16, TF32, INT8, INT4, FP8 (Hopper+)

### RT Core / Ray Accelerator

| 厂商 | 名称 | 功能 |
|------|------|------|
| NVIDIA | RT Core | BVH 遍历 + 射线-三角形求交 |
| AMD | Ray Accelerator | BVH 遍历 + 射线-盒子/三角形求交 |
| Intel | RT Unit | BVH 遍历 + 射线求交 |

**性能提升：** 相比纯软件实现，硬件光追加速约 10×。

---

## 5. 内存层次（Shader Core 视角）

| 层级 | 大小 | 延迟 | 作用域 | 用途 |
|------|------|------|--------|------|
| 寄存器 | 256 KB/SM | 0 周期 | 线程私有 | 局部变量、中间结果 |
| 共享内存 | 0-128 KB/SM | ~20 周期 | Block 内共享 | 线程间通信、数据复用 |
| L1 Cache | 128 KB/SM | ~30 周期 | SM 内 | 缓存全局内存访问 |
| L2 Cache | 40-60 MB/GPU | ~200 周期 | GPU 全局 | 跨 SM 数据共享 |
| 全局内存 (HBM) | 24-80 GB | ~500 周期 | GPU 全局 | 主数据存储 |

**关键洞察：** 寄存器到全局内存的延迟差距达 500 周期。共享内存的存在就是为了弥补这个差距——让 Block 内线程协作加载全局内存数据到快速 scratchpad，然后多次复用。

---

## 6. Coalesced Memory Access（合并内存访问）

```
良好合并（单次事务）:
Warp 线程: 0  1  2  3  4  5  6  7  ...  31
访问地址: 0  4  8  12 16 20 24 28 ...  124
          └────────── 128 字节连续 ──────────┘

不良合并（32 次事务）:
Warp 线程: 0  1  2  3  4  5  6  7  ...  31
访问地址: 0  1M 2M 3M 4M 5M 6M 7M ...  31M
          └────── 完全分散，带宽浪费 ──────┘
```

- 32 线程访问 32 个连续 4 字节地址 → 合并为单次 128 字节事务
- 分散访问可能导致有效带宽降低 10× 以上
- **优化：** 数据布局按 warp 访问模式对齐（AoS → SoA 转换）

---

## 7. 驱动开发视角

### 内核驱动中的 Shader Core 管理

```c
// NVIDIA (nouveau) — SM 时钟控制
nvkm_therm_clkgate_init(device, &sm_clk);

// AMD (amdgpu) — CU 仲裁配置
amdgpu_gfx_select_cu_sh_mask(adev, cu_mask, se_mask);

// Intel (i915/xe) — EU 配置
xe_hw_engine_enable(eu);
```

### Compute Shader 调度

```c
// 驱动将 compute dispatch 映射到硬件
// 1. 计算所需的 SM/CU 数量
// 2. 分配寄存器和共享内存
// 3. 设置 Warp/Wavefront 调度参数
// 4. 发射 kernel
```

---

## 潜在影响

- **AI 训练/推理：** Tensor Core 的利用率直接决定了大模型训练的性价比。理解 SM 资源分配（寄存器/共享内存/occupancy）是编写高效 CUDA kernel 的基础。
- **光追游戏：** RT Core 的 BVH 遍历效率决定了光追游戏的帧率。驱动需要高效管理 SM 在光追和光栅化之间的资源分配。
- **驱动优化：** GPU 驱动的 shader 编译器（NVIDIA PTX、AMD GCN/RDNA ISA、Intel SIMD ISA）需要理解硬件资源限制，生成最优的寄存器分配和指令调度。
- **演进方向：** NVIDIA Blackwell 引入了 Transformer Engine（动态精度），AMD RDNA 4 预计增强 AI 能力，Intel Battlemage 已量产。三大厂商都在 Shader Core 中增加 AI 专用硬件。

---

## 信源

- [GPU Streaming Multiprocessor (SM) — abhik.ai](https://www.abhik.ai/concepts/gpu/shared-multiprocessor)
- [NVIDIA Ada Lovelace Architecture Whitepaper](https://www.nvidia.com/en-us/geforce/technologies/ada-lovelace-architecture/)
- [AMD RDNA 3 Architecture — GPUOpen](https://gpuopen.com/learn/rdna-3-architecture/)
- [Intel Xe-HPG Microarchitecture — wikichip](https://en.wikichip.org/wiki/intel/microarchitectures/xe_hpg)
- [drm/nouveau — Linux v6.13](https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/nouveau)
- [drm/amd — Linux v6.13](https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/amd)
