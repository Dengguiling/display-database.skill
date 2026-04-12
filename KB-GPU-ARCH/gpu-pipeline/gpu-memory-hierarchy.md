# GPU 内存层次结构

> **条目ID:** KB-GPU-ARCH-gpu-pipeline-003
> **模块:** KB-GPU-ARCH / gpu-pipeline
> **置信度:** 5/5（基于 NVIDIA/AMD/Intel 官方架构文档 + 内核驱动源码 + 技术分析文章）
> **时效性:** 2026-04-12 | **信源:** NVIDIA, AMD, Intel, kernel.org, abhik.ai
> **关联:** [shader-core-arch](./shader-core-arch.md) | [gpu-rendering-pipeline](./gpu-rendering-pipeline.md)

---

## 核心摘要

GPU 内存层次结构是决定 GPU 性能的关键因素——从最快的片上寄存器（~0 cycle, ~8 TB/s）到最慢的片外显存（~400-600 cycle, 1-2 TB/s），延迟差距超过三个数量级。现代 GPU 采用多级缓存 + HBM/GDDR 高带宽显存 + TTM 内存管理器的组合架构。理解内存层次对于 GPU 驱动开发（显存分配策略、页面驱逐、DMA 映射）和性能优化（coalescing、bank conflict、tiling）至关重要。

---

## 1. 内存层次总览

### 五级层次结构

| 层级 | 位置 | 大小 | 延迟 | 带宽 | 作用域 |
|------|------|------|------|------|--------|
| **寄存器** | 片上，per-thread | 256 KB/SM | ~0 cycle | ~8 TB/s | 线程私有 |
| **共享内存** | 片上，per-SM | 48-228 KB/SM | ~20-30 cycle | ~4 TB/s | Block 内共享 |
| **L1 Cache** | 片上，per-SM | 128 KB/SM | ~30-40 cycle | ~4 TB/s | SM 内缓存 |
| **L2 Cache** | 片上，GPU 全局 | 40-60 MB | ~200 cycle | ~2-3 TB/s | GPU 全局缓存 |
| **显存 (VRAM)** | 片外 | 24-80 GB | ~400-600 cycle | 1-2 TB/s | GPU 全局 |

### 延迟对比

```
延迟 (cycles, log scale)
│
│ 600 ┤                                          ████ VRAM
│     │                                     ████
│ 400 ┤                                ████
│     │                           ████
│ 200 ┤                      ████ L2
│     │                 ████
│  40 ┤            ████ L1
│     │       ████
│  30 ┤  ████ Shared Mem
│     │
│   0 ┤███ Registers
│     └──────────────────────────────────────────
       Reg  Shared  L1    L2    VRAM
```

---

## 2. 显存类型对比

### GDDR vs HBM

| 维度 | GDDR6X | GDDR7 | HBM3 | HBM3E |
|------|--------|-------|------|-------|
| **接口宽度** | 32-bit/ch | 32-bit/ch | 1024-bit/stack | 1024-bit/stack |
| **数据速率** | 21 Gbps | 32 Gbps | 6.4 Gbps/pin | 9.2 Gbps/pin |
| **单芯片带宽** | 84 GB/s | 128 GB/s | 819 GB/s/stack | 1.2 TB/s/stack |
| **容量** | 1-2 GB/chip | 2-4 GB/chip | 8-16 GB/stack | 8-16 GB/stack |
| **功耗** | 中 | 中 | 低 | 低 |
| **封装** | 独立芯片 | 独立芯片 | 2.5D SiP | 2.5D SiP |
| **代表产品** | RTX 4090 | RTX 5090 | H100 | B200 |
| **适用场景** | 消费级 GPU | 消费级 GPU | 数据中心 | 数据中心 |

### 关键差异

- **GDDR：** 高引脚速率，窄接口，适合 PCB 布局，成本较低
- **HBM：** 宽接口，低引脚速率，需要 2.5D 封装（硅中介层），成本高但带宽密度极大
- **LPDDR：** 移动 GPU 使用，低功耗优先

---

## 3. 各级内存详解

### 3.1 寄存器

- **分配：** 编译器自动分配，每个线程最多 255 个 32-bit 寄存器
- **溢出 (Spill)：** 寄存器不足时溢出到 Local Memory（实际在 L2/VRAM），性能大幅下降
- **Occupancy 影响：** 每线程寄存器越多，可同时运行的 Warp 越少

### 3.2 共享内存

- **用途：** Block 内线程间数据共享，手动管理
- **Bank Conflict：** 32 个 Bank，每个 4 字节宽。多线程访问同一 Bank 的不同地址会产生串行化
- **优化：** 添加 padding（如 `shared float tile[32][33]`）避免 bank conflict
- **L1/Shared 配置：** 可通过 `cudaFuncSetCacheConfig()` 调整比例

### 3.3 L1 Cache

- **与共享内存共享物理存储**（NVIDIA Ada/Hopper）
- **缓存全局内存和局部内存访问**
- **不可编程：** 硬件自动管理

### 3.4 L2 Cache

- **GPU 全局共享**，所有 SM 的内存请求都经过 L2
- **持久化：** 跨 kernel launch 保持数据
- **原子操作：** 支持缓存行级别的原子操作
- **流式优化：** `cudaStreamAttrValue` 可设置 L2 persisting 访问窗口

### 3.5 显存 (VRAM)

- **HBM：** 通过硅中介层与 GPU die 邻接，超宽接口提供极高带宽
- **GDDR：** 通过 PCB 走线连接，单通道带宽较低但通道数多
- **ECC：** 数据中心 GPU 支持，占用 ~6.25% 显存容量

---

## 4. 内存访问模式

### Coalescing（合并访问）

| 模式 | 效率 | 事务数 | 说明 |
|------|------|--------|------|
| **Coalesced** | 100% | 1 | 连续线程访问连续地址 |
| **Misaligned** | 80-100% | 1-2 | 跨缓存行边界 |
| **Strided (2x)** | 50% | 2 | 步长为 2 |
| **Strided (32x)** | 3% | 32 | 步长为 32（最差） |
| **Random** | 3-10% | 32 | 随机访问 |

### Bank Conflict

```
32 Banks, 每个 4 字节宽
Thread 0 → Bank 0, offset 0
Thread 1 → Bank 1, offset 0
...
Thread 31 → Bank 31, offset 0  ← 无冲突

// 冲突示例
Thread 0 → Bank 0, offset 0
Thread 1 → Bank 0, offset 1  ← 2-way conflict
Thread 2 → Bank 0, offset 2  ← 3-way conflict
...
```

---

## 5. TTM 内存管理器

### TTM (Translation Table Manager)

Linux 内核的 GPU 显存管理框架，被 amdgpu、nouveau、vmwgfx 等驱动使用：

| Placement | 说明 | 驱逐策略 |
|-----------|------|----------|
| `TTM_PL_VRAM` | 显存（最快） | 可驱逐到 GTT |
| `TTM_PL_TT` | GTT（Graphics Translation Table） | 可驱逐到 System RAM |
| `TTM_PL_SYSTEM` | 系统内存（最慢） | 可 swap |

### TTM 工作流程

```
1. 应用请求 VRAM 中的 buffer
2. TTM 检查 VRAM 是否有空间
3. 若无空间，驱逐 LRU buffer 到 GTT
4. 若 GTT 也满，驱逐到 System RAM
5. 分配 VRAM 空间
6. 迁移 buffer 到 VRAM
7. 更新页表映射
```

### 内核驱动接口

```c
// amdgpu 显存分配
amdgpu_bo_create(adev, size, align, true, AMDGPU_GEM_DOMAIN_VRAM,
                 0, NULL, NULL, &bo);

// TTM buffer eviction
ttm_bo_evict(bo, ctx);
ttm_bo_validate(bo, &placement, &ctx);
```

---

## 6. DMA-BUF 与显存共享

### 跨设备显存访问

```
GPU VRAM ──→ DMA-BUF ──→ Video Encoder (V4L2)
           (零拷贝)       (直接 DMA 读取 VRAM)
```

- GPU 驱动实现 `dma_buf_ops` 接口
- `pin()` 回调确保 buffer 驻留在 VRAM
- `map_dma_buf()` 返回 scatter-gather 表

### PCIe DMA

```
System RAM ──→ PCIe ──→ GPU VRAM
  (CPU)      (带宽 ~16-32 GB/s, 延迟 ~1-2 μs)
```

- PCIe 4.0 x16: ~32 GB/s
- PCIe 5.0 x16: ~64 GB/s
- CXL: 潜在的 GPU-CPU 内存扩展方案

---

## 7. 三大厂商内存架构对比

| 维度 | NVIDIA (Hopper H100) | AMD (RDNA 3) | Intel (Xe-HPG) |
|------|---------------------|--------------|----------------|
| L2 | 50 MB | 16 MB | 16 MB |
| VRAM 类型 | HBM3 | GDDR6 | GDDR6 |
| VRAM 容量 | 80 GB | 24 GB | 16 GB |
| VRAM 带宽 | 3.35 TB/s | 576 GB/s | 512 GB/s |
| 共享内存 | 228 KB/SM | 128 KB/CU | 64 KB/Slice |
| L1 | 256 KB/SM | 32 KB/CU | 128 KB/Slice |
| 内存管理器 | 自研 (NVMM) | TTM | 自研 |

---

## 潜在影响

- **AI 训练：** 大模型训练的显存需求（70B 参数 FP16 = ~140 GB）远超单卡容量，多卡 NVLink/CXL 互联和显存聚合是关键。
- **游戏性能：** 4K 纹理流式加载依赖 VRAM 带宽和 L2 缓存命中率。RTX 5090 的 GDDR7 提供 ~1.8 TB/s 带宽，缓解了 4K 渲染的带宽瓶颈。
- **驱动开发：** TTM 的驱逐策略直接影响 GPU 在显存不足时的性能表现。不合理的驱逐策略可能导致帧率骤降。
- **演进方向：** HBM4 预计 2026 年量产，带宽将超过 2 TB/s/stack。CXL 3.0+ 为 GPU 扩展系统内存提供了新路径。

---

## 信源

- [GPU Memory Hierarchy & Optimization — abhik.ai](https://www.abhik.ai/concepts/gpu/memory-hierarchy)
- [NVIDIA H100 Architecture — NVIDIA](https://resources.nvidia.com/en-us-tensor-core/nvidia-tensor-core-gpu-architecture)
- [AMD RDNA 3 Architecture — GPUOpen](https://gpuopen.com/learn/rdna-3-architecture/)
- [TTM — Linux kernel documentation](https://docs.kernel.org/gpu/drm-mm.html)
- [drm/ttm — Linux v6.13](https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/ttm)
