# Vulkan 命令缓冲区

> **条目ID:** KB-GPU-ARCH-gpu-pipeline-006
> **模块:** KB-GPU-ARCH / gpu-pipeline
> **置信度:** 5/5（基于 Vulkan 1.4 Specification + Khronos 官方文档）
> **时效性:** 2026-04-12 | **信源:** Khronos Vulkan 1.4 Spec, vulkan.lunarg.com
> **关联:** [vulkan-overview](./vulkan-overview.md) | [vulkan-instance-device](./vulkan-instance-device.md) | [vulkan-pipeline](./vulkan-pipeline.md)

---

## 核心摘要

Vulkan 命令缓冲区（Command Buffer）是 Vulkan API 的核心执行载体——所有 GPU 操作（绘制、计算、拷贝、同步）都通过 `vkCmd*` 系列函数录制到命令缓冲区中，然后提交到设备队列执行。命令缓冲区分为两级：Primary（可直接提交到队列，可执行 Secondary）和 Secondary（只能被 Primary 执行，不可直接提交）。Vulkan 的命令缓冲区设计支持**多线程录制**（每个线程独立录制到不同的命令缓冲区）、**预录制和复用**（一次录制多次提交），以及**Secondary 嵌套**（将复杂渲染拆分为可复用的子命令序列）。

---

## 1. 命令缓冲区生命周期

### 五种状态

```
  allocate
     │
     ▼
┌─────────┐  vkBeginCommandBuffer  ┌───────────┐  vkEndCommandBuffer  ┌───────────┐
│ Initial  │ ─────────────────────→ │ Recording  │ ──────────────────→ │ Executable │
└─────────┘                        └───────────┘                      └───────────┘
     ▲                                  │                                    │
     │                                  │                                    │ vkQueueSubmit
     │  vkResetCommandBuffer            │                                    ▼
     │  vkResetCommandPool              │                              ┌─────────┐
     │  vkBeginCommandBuffer            │                              │ Pending  │
     └──────────────────────────────────┘                              └─────────┘
     │                                                                       │
     │  ONE_TIME_SUBMIT or object destroyed                                  │ execution complete
     │                                                                       ▼
     │                                                                ┌─────────┐
     └──────────────────────────────────────────────────────────────── │ Invalid  │
                                                                       └─────────┘
```

| 状态 | 说明 | 允许操作 |
|------|------|----------|
| **Initial** | 分配后的初始状态 | `vkBeginCommandBuffer`、释放 |
| **Recording** | 正在录制命令 | `vkCmd*` 系列函数、`vkEndCommandBuffer` |
| **Executable** | 录制完成，可提交 | `vkQueueSubmit`、`vkBeginCommandBuffer`（重录）、重置 |
| **Pending** | 已提交到队列，GPU 执行中 | 等待完成（fence）、不可修改 |
| **Invalid** | 一次性提交后或依赖对象被销毁 | 重置、释放 |

### 关键约束

- **无状态继承：** Primary 和 Secondary 之间**不继承任何状态**（RenderPass 除外）
- **录制时状态未定义：** `vkBeginCommandBuffer` 后所有状态未定义，必须显式绑定
- **Pending 不可修改：** 命令缓冲区在 GPU 执行期间绝对不可修改

---

## 2. Command Pool

### 创建

```c
VkCommandPoolCreateInfo poolInfo = {
    .sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
    .flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT,  // 允许单独重置
    .queueFamilyIndex = graphicsQueueFamily,  // 绑定到特定队列族
};
vkCreateCommandPool(device, &poolInfo, NULL, &commandPool);
```

### 标志位

| 标志 | 说明 |
|------|------|
| `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT` | 短生命周期命令缓冲区，优化分配策略 |
| `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` | 允许单独重置命令缓冲区（否则只能重置整个 Pool） |

### 线程安全

- **Command Pool 本身不是线程安全的** — 不能在多线程中同时使用同一个 Pool
- **解决方案：** 每个线程创建独立的 Command Pool，或使用外部同步

---

## 3. 命令缓冲区分配

```c
VkCommandBufferAllocateInfo allocInfo = {
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
    .commandPool = commandPool,
    .level = VK_COMMAND_BUFFER_LEVEL_PRIMARY,  // 或 SECONDARY
    .commandBufferCount = 1,
};
vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
```

### Primary vs Secondary

| 维度 | Primary | Secondary |
|------|---------|-----------|
| 提交 | 可直接提交到队列 | 只能被 Primary 执行 |
| 执行 Secondary | ✅ `vkCmdExecuteCommands` | ❌ |
| RenderPass | 可开始/结束 RenderPass | 只能在 RenderPass 内执行 |
| 继承 | 无状态继承 | 可继承 RenderPass 状态 |
| 典型用途 | 帧提交、Compute Dispatch | 预录制的物体渲染、阴影 Pass |

---

## 4. 命令录制

### 录制流程

```c
vkBeginCommandBuffer(cmdBuf, &(VkCommandBufferBeginInfo){
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
    .flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT,  // 一次性使用
});

// 绑定管线
vkCmdBindPipeline(cmdBuf, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);

// 绑定描述符集
vkCmdBindDescriptorSets(cmdBuf, VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipelineLayout, 0, 1, &descriptorSet, 0, NULL);

// 设置动态状态
vkCmdSetViewport(cmdBuf, 0, 1, &viewport);
vkCmdSetScissor(cmdBuf, 0, 1, &scissor);

// 绘制
vkCmdDraw(cmdBuf, vertexCount, instanceCount, firstVertex, firstInstance);

vkEndCommandBuffer(cmdBuf);
```

### 录制标志位

| 标志 | 说明 |
|------|------|
| `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT` | 一次性提交，执行后自动进入 Invalid 状态 |
| `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT` | Secondary 在 RenderPass 内继续录制 |
| `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT` | 可同时被多个队列提交（需要外部同步） |

---

## 5. 命令分类

### 绘制命令

| 命令 | 说明 |
|------|------|
| `vkCmdDraw` | 直接绘制（顶点在 VRAM 中） |
| `vkCmdDrawIndexed` | 索引绘制 |
| `vkCmdDrawIndirect` | 间接绘制（参数在 buffer 中） |
| `vkCmdDrawIndexedIndirect` | 索引间接绘制 |
| `vkCmdDrawMeshTasksEXT` | Mesh Shader 绘制 |
| `vkCmdDrawMeshTasksIndirectEXT` | Mesh Shader 间接绘制 |

### 计算命令

| 命令 | 说明 |
|------|------|
| `vkCmdDispatch` | 直接计算调度 |
| `vkCmdDispatchIndirect` | 间接计算调度 |

### 拷贝命令

| 命令 | 说明 |
|------|------|
| `vkCmdCopyBuffer` | Buffer → Buffer |
| `vkCmdCopyImage` | Image → Image |
| `vkCmdCopyBufferToImage` | Buffer → Image |
| `vkCmdCopyImageToBuffer` | Image → Buffer |
| `vkCmdBlitImage` | Image → Image（带缩放/格式转换） |
| `vkCmdResolveImage` | MSAA 解析 |

### 同步命令

| 命令 | 说明 |
|------|------|
| `vkCmdPipelineBarrier` | 管线屏障（内存/执行依赖） |
| `vkCmdWaitEvents` | 等待事件 |
| `vkCmdSetEvent` | 设置事件 |
| `vkCmdResetEvent` | 重置事件 |

### 状态绑定命令

| 命令 | 说明 |
|------|------|
| `vkCmdBindPipeline` | 绑定管线 |
| `vkCmdBindDescriptorSets` | 绑定描述符集 |
| `vkCmdBindVertexBuffers` | 绑定顶点缓冲区 |
| `vkCmdBindIndexBuffer` | 绑定索引缓冲区 |
| `vkCmdPushConstants` | 设置 Push Constants |
| `vkCmdSetViewport` / `vkCmdSetScissor` | 设置视口/裁剪 |
| `vkCmdSetLineWidth` | 设置线宽 |
| `vkCmdSetBlendConstants` | 设置混合常数 |

---

## 6. Secondary 命令缓冲区

### 创建与录制

```c
// 分配 Secondary
VkCommandBufferAllocateInfo allocInfo = {
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
    .commandPool = pool,
    .level = VK_COMMAND_BUFFER_LEVEL_SECONDARY,
    .commandBufferCount = 1,
};
vkAllocateCommandBuffers(device, &allocInfo, &secondaryCmd);

// 录制（在 RenderPass 内继续）
VkCommandBufferInheritanceInfo inheritanceInfo = {
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_INHERITANCE_INFO,
    .renderPass = renderPass,
    .subpass = 0,
    .framebuffer = framebuffer,  // 可选
};

vkBeginCommandBuffer(secondaryCmd, &(VkCommandBufferBeginInfo){
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
    .flags = VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT,
    .pInheritanceInfo = &inheritanceInfo,
});

// 录制绘制命令...
vkCmdDraw(secondaryCmd, ...);

vkEndCommandBuffer(secondaryCmd);
```

### 执行 Secondary

```c
// 在 Primary 中执行
vkCmdExecuteCommands(primaryCmd, 1, &secondaryCmd);
```

### Secondary 的生命周期耦合

```
Primary (Executable) ── vkCmdExecuteCommands ──→ Secondary (Executable)
        │                                                │
        │  vkQueueSubmit                                 │
        ▼                                                ▼
Primary (Pending) ──────────────────────────────→ Secondary (Pending)
        │                                                │
        │  execution complete                            │
        ▼                                                ▼
Primary (Executable) ─────────────────────────→ Secondary (Executable/Invalid)
```

---

## 7. 提交与同步

### 提交

```c
VkSubmitInfo submitInfo = {
    .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &imageAvailableSemaphore,
    .pWaitDstStageMask = (VkPipelineStageFlags[]){ VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT },
    .commandBufferCount = 1,
    .pCommandBuffers = &commandBuffer,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};

vkQueueSubmit(queue, 1, &submitInfo, fence);
```

### Fence 等待

```c
// CPU 等待 GPU 完成
vkWaitForFences(device, 1, &fence, VK_TRUE, UINT64_MAX);
vkResetFences(device, 1, &fence);
```

### 多线程录制模式

```
Thread 1          Thread 2          Thread 3
   │                 │                 │
   ├─ begin CB       ├─ begin CB       ├─ begin CB
   ├─ record shadow  ├─ record opaque  ├─ record transparent
   ├─ end CB         ├─ end CB         ├─ end CB
   │                 │                 │
   └─────────────────┴─────────────────┘
                     │
                     ▼
              Primary CB: execute all secondaries
                     │
                     ▼
              vkQueueSubmit
```

---

## 8. 性能优化

### 命令缓冲区复用策略

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| 一次性（ONE_TIME_SUBMIT） | 每帧不同的命令 | 无状态残留 | 每帧重新分配 |
| 重录（Reset + Re-record） | 命令结构相同，数据不同 | 复用分配 | 每帧重新录制 |
| 复用（SIMULTANEOUS_USE） | 静态场景 | 零录制开销 | 需要外部同步 |

### Command Pool 分组策略

```c
// 推荐的 Pool 分组方式
Pool-Graphics-Transient   // 每帧的图形命令（短期）
Pool-Compute-Transient    // 每帧的计算命令（短期）
Pool-Static               // 预录制的静态命令（长期）
Pool-Upload               // 资源上传命令（短期）
```

---

## 潜在影响

- **多线程渲染：** 命令缓冲区的无状态设计是 Vulkan 多线程渲染的基础。Unreal Engine 5 的 RDG（Render Dependency Graph）利用 Secondary 命令缓冲区实现并行渲染 Pass 录制。
- **驱动开销：** Vulkan 的命令缓冲区设计使得驱动端开销极低（thin driver），CPU 端的录制和提交几乎零开销。
- **Linux 图形栈：** Mesa Vulkan 驱动将命令缓冲区翻译为 GPU 特定的命令流（AMD: PM4 packets, Intel: batch buffer），直接提交给内核 DRM scheduler。
- **演进方向：** Vulkan 的 Device-Generated Commands（DGC）扩展允许 GPU 自主生成命令，进一步减少 CPU-GPU 交互。

---

## 信源

- [Vulkan 1.4 Specification — Command Buffers](https://vulkan.lunarg.com/doc/view/1.4.341.0/mac/antora/spec/latest/chapters/cmdbuffers.html)
- [Vulkan Tutorial — Command Buffers](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Command_buffers)
- [Vulkan Reference Pages — vkBeginCommandBuffer](https://docs.vulkan.org/refpages/latest/refpages/source/vkBeginCommandBuffer.html)
