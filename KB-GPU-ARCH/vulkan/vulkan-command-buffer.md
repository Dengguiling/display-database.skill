# Vulkan Command Buffer

> Vulkan 的命令录制机制——将 GPU 操作预录制到 Command Buffer 中，支持多线程录制、重用和批量提交。

## Meta
- **ID:** KB-GPU-ARCH-vulkan-003
- **置信度:** 5
- **信源等级:** P0
- **验证日期:** 2026-04-13
- **适用版本:** Vulkan 1.0 ~ 1.4
- **关键词:** command buffer, command pool, primary/secondary, vkBeginCommandBuffer, vkEndCommandBuffer, vkQueueSubmit, one-time submit, reset, recording state
- **前置知识:** [→ vulkan-overview] [→ vulkan-instance-device]
- **关联条目:** [→ vulkan-pipeline] [→ vulkan-memory]

## 概述

Vulkan Command Buffer 是 GPU 命令的录制容器。与 OpenGL 的即时模式不同，Vulkan 将所有 GPU 操作（绘制、调度、内存屏障、管线绑定）预录制到 Command Buffer 中，然后通过 `vkQueueSubmit` 批量提交到 GPU 队列执行。Command Buffer 分为 Primary（可直接提交）和 Secondary（嵌入 Primary）两级，支持多线程录制、一次性和可重用两种模式。在 Linux Mesa 驱动中，Command Buffer 最终被翻译为 GPU 特定的命令流，通过 DRM Scheduler 提交到内核。

## 架构/调用链

### Command Buffer 生命周期

```
vkAllocateCommandBuffers()
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Initial 状态                                                │
│  (刚分配，未开始录制)                                         │
└─────────────────────────────────────────────────────────────┘
    │ vkBeginCommandBuffer()
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Recording 状态                                              │
│  (录制命令: vkCmdDraw*, vkCmdDispatch, vkCmdPipelineBarrier) │
└─────────────────────────────────────────────────────────────┘
    │ vkEndCommandBuffer()
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Executable 状态                                             │
│  (可提交到队列)                                              │
└─────────────────────────────────────────────────────────────┘
    │ vkQueueSubmit()
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Pending 状态                                                │
│  (GPU 正在执行)                                              │
└─────────────────────────────────────────────────────────────┘
    │ Fence signaled
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Executable 状态 (可再次提交)                                 │
└─────────────────────────────────────────────────────────────┘
```

### Primary 与 Secondary Command Buffer

```
Primary Command Buffer
    │
    ├── vkCmdBeginRenderPass() / vkCmdBeginRenderingKHR()
    │   │
    │   ├── Secondary Command Buffer #1
    │   │   ├── vkCmdBindPipeline()
    │   │   ├── vkCmdBindDescriptorSets()
    │   │   ├── vkCmdDraw()
    │   │   └── vkCmdEndRenderPass() ← 错误! Secondary 不能结束 RP
    │   │
    │   ├── Secondary Command Buffer #2
    │   │   ├── vkCmdBindPipeline()
    │   │   ├── vkCmdDraw()
    │   │   └── vkCmdNextSubpass() ← 错误! Secondary 不能切换 subpass
    │   │
    │   └── vkCmdEndRenderPass() / vkCmdEndRenderingKHR()
    │
    ├── vkCmdPipelineBarrier()
    ├── vkCmdDispatch()
    └── vkQueueSubmit()
```

### 用户空间到内核的命令流翻译

```
应用程序 vkCmd*() 调用
    │
    ▼
Vulkan ICD (Mesa RADV/ANV)
    │
    ├── 命令录制到驱动内部 CSO (Command Stream Object)
    ├── SPIR-V → GPU ISA 编译（管线创建时已完成）
    ├── 资源地址解析 (VA → GPU 物理地址)
    └── 生成 GPU 命令流
         │
         ▼
DRM Scheduler (drm_sched)
    │
    ├── 命令流封装为 drm_sched_job
    ├── 依赖排序 (fence 链)
    └── 提交到硬件队列
         │
         ▼
GPU 执行
```

## 关键接口

### Command Pool

- `vkCreateCommandPool(VkDevice, const VkCommandPoolCreateInfo*, const VkAllocationCallbacks*, VkCommandPool*)` — 创建命令池
  - `flags`: `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`（短期使用）、`VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`（允许单独重置）
  - `queueFamilyIndex`: 命令池关联的队列族
  - 注意: Command Pool 是 Command Buffer 的内存分配器，销毁池会释放所有关联的 Command Buffer

- `vkResetCommandPool(VkDevice, VkCommandPool, VkCommandPoolResetFlags)` — 重置命令池
  - `VK_COMMAND_POOL_RESET_RELEASE_RESOURCES_BIT`: 释放所有 Command Buffer 的内存
  - 注意: 比逐个重置 Command Buffer 更高效

### Command Buffer

- `vkAllocateCommandBuffers(VkDevice, const VkCommandBufferAllocateInfo*, VkCommandBuffer*)` — 分配命令缓冲区
  - `level`: `VK_COMMAND_BUFFER_LEVEL_PRIMARY` 或 `VK_COMMAND_BUFFER_LEVEL_SECONDARY`
  - `commandBufferCount`: 一次分配多个
  - 注意: 从 Command Pool 分配，不涉及 GPU 内存

- `vkBeginCommandBuffer(VkCommandBuffer, const VkCommandBufferBeginInfo*)` — 开始录制
  - `flags`:
    - `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`: 一次性提交（录制后只能提交一次）
    - `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`: Secondary CB，在 RenderPass 内继续
    - `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`: 允许同时被多个队列提交
  - `pInheritanceInfo`: Secondary CB 的继承信息（RenderPass、子 pass、帧缓冲区）

- `vkEndCommandBuffer(VkCommandBuffer)` — 结束录制
  - 注意: 录制结束后不能再修改，除非重置

- `vkResetCommandBuffer(VkCommandBuffer, VkCommandBufferResetFlags)` — 重置单个命令缓冲区
  - 注意: Command Pool 必须创建时设置了 `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`

### 提交

- `vkQueueSubmit(VkQueue, uint32_t submitCount, const VkSubmitInfo* pSubmits, VkFence fence)` — 提交命令到队列
  - `pSubmits->pCommandBuffers`: 提交的 Command Buffer 数组
  - `pSubmits->pWaitSemaphores`: 等待的信号量（GPU-GPU 同步）
  - `pSubmits->pSignalSemaphores`: 发送的信号量
  - `fence`: CPU-GPU 同步（可选）
  - 注意: 提交后 Command Buffer 进入 Pending 状态

- `vkQueueWaitIdle(VkQueue)` — 等待队列所有提交完成
  - 注意: 重量级操作

## 实现要点

### 多线程录制模式

```c
// 每个线程一个 Command Pool（避免锁竞争）
VkCommandPool threadPools[MAX_THREADS];
for (int i = 0; i < MAX_THREADS; i++) {
    vkCreateCommandPool(device, &(VkCommandPoolCreateInfo){
        .sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
        .flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT,
        .queueFamilyIndex = graphicsQF,
    }, NULL, &threadPools[i]);
}

// 每个线程独立录制
void* recordThread(void* arg) {
    int tid = *(int*)arg;
    VkCommandBuffer cb;
    vkAllocateCommandBuffers(device, &(VkCommandBufferAllocateInfo){
        .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO,
        .commandPool = threadPools[tid],
        .level = VK_COMMAND_BUFFER_LEVEL_PRIMARY,
        .commandBufferCount = 1,
    }, &cb);

    vkBeginCommandBuffer(cb, &(VkCommandBufferBeginInfo){
        .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
        .flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT,
    });
    // ... 录制命令 ...
    vkEndCommandBuffer(cb);

    // 将 cb 传回主线程提交
    return NULL;
}
```

### Secondary Command Buffer 使用模式

```c
// 场景: 将不同物体的绘制命令录制到不同的 Secondary CB
// 主线程: Primary CB
VkCommandBuffer primaryCB;
vkBeginCommandBuffer(primaryCB, &(VkCommandBufferBeginInfo){
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
});

vkCmdBeginRenderPass(primaryCB, &(VkRenderPassBeginInfo){
    .sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
    .renderPass = renderPass,
    .framebuffer = framebuffer,
    .renderArea = {{0, 0}, {width, height}},
    .clearValueCount = 2,
    .pClearValues = clearValues,
}, VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);

// 执行所有 Secondary CB
vkCmdExecuteCommands(primaryCB, secondaryCBCount, secondaryCBs);

vkCmdEndRenderPass(primaryCB);
vkEndCommandBuffer(primaryCB);
```

### DRM Scheduler 中的命令流

```c
// Mesa RADV 中 Command Buffer 提交到 DRM Scheduler 的简化流程
// (概念性代码，非实际源码)

// 1. 驱动将 Vulkan Command Buffer 翻译为 GPU 命令流
struct radv_cmd_buffer *cmd_buffer = to_radv_cmd_buffer(vk_cb);

// 2. 封装为 drm_sched_job
struct drm_sched_job *job = drm_sched_job_alloc(sched, ...);

// 3. 设置 fence 依赖
drm_sched_job_add_dependency(job, wait_fence);

// 4. 提交到调度器
drm_sched_entity_push_job(&entity, job);

// 5. DRM Scheduler 在硬件队列空闲时执行 job
// 内核: drivers/gpu/drm/scheduler/sched_main.c
```

## 版本差异

| 版本 | Command Buffer 变更 |
|------|---------------------|
| 1.0 | 基础 Primary/Secondary CB、Command Pool |
| 1.1 | 无重大变更 |
| 1.2 | `VK_KHR_buffer_device_address` 影响命令缓冲区中的地址引用 |
| 1.3 | `VK_KHR_maintenance4`: `vkCmdBlitImage2`、`vkCmdCopyBuffer2` 等扩展命令 |
| 1.4 | `VK_KHR_maintenance5`: `vkCmdBindIndexBuffer2KHR`、`vkCmdBindVertexBuffers2` |

## 陷阱与注意事项

- **Command Buffer 状态机违规：** 在 Recording 状态外调用 `vkCmd*` 是未定义行为。在 Executable 状态调用 `vkCmd*` 会导致验证层报错。
- **Secondary CB 的 RenderPass 兼容性：** Secondary CB 的 `pInheritanceInfo->renderPass` 必须与 Primary CB 的 RenderPass 兼容（attachment 格式和数量匹配）。不兼容时行为未定义。
- **一次性 CB 的重用：** 设置了 `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT` 的 CB 提交后不能再次提交，必须重置后重新录制。
- **Command Pool 线程安全：** 从同一个 Command Pool 分配 Command Buffer 不是线程安全的。多线程录制必须使用不同的 Command Pool。
- **vkQueueSubmit 后修改 CB：** 提交后 CB 进入 Pending 状态，不能修改。如果设置了 `SIMULTANEOUS_USE_BIT`，可以再次提交但不能修改。
- **Fence 复用：** `vkQueueSubmit` 使用的 Fence 在信号后必须 `vkResetFences` 后才能再次使用。未重置直接使用会导致未定义行为。

## 使用模式

### 帧录制-提交循环

```c
// 每帧: 录制 → 提交 → 等待 → 重置
VkCommandBuffer cb = frameCBs[currentFrame];

vkResetCommandBuffer(cb, 0);
vkBeginCommandBuffer(cb, &(VkCommandBufferBeginInfo){
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
    .flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT,
});

// 录制渲染命令
vkCmdBeginRenderingKHR(cb, &(VkRenderingInfo){
    .sType = VK_STRUCTURE_TYPE_RENDERING_INFO,
    .renderArea = {{0, 0}, {width, height}},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &(VkRenderingAttachmentInfo){
        .sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO,
        .imageView = swapchainViews[currentFrame],
        .imageLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
        .loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR,
        .storeOp = VK_ATTACHMENT_STORE_OP_STORE,
        .clearValue = {.color = {{0.1f, 0.1f, 0.1f, 1.0f}}},
    },
});

vkCmdBindPipeline(cb, VK_PIPELINE_BIND_POINT_GRAPHICS, pipeline);
vkCmdBindDescriptorSets(cb, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout,
                         0, 1, &descriptorSet, 0, NULL);
vkCmdDraw(cb, vertexCount, 1, 0, 0);

vkCmdEndRenderingKHR(cb);
vkEndCommandBuffer(cb);

// 提交
VkSubmitInfo submit = {
    .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
    .commandBufferCount = 1,
    .pCommandBuffers = &cb,
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &imageAvailableSemaphore,
    .pWaitDstStageMask = &(VkPipelineStageFlags){VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT},
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &renderFinishedSemaphore,
};
vkQueueSubmit(queue, 1, &submit, inFlightFences[currentFrame]);
```

## Sources

- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#commandbuffers — Vulkan Spec §Command Buffers: 状态机、生命周期、录制规则
- [P0] https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#commandbuffer-lifecycle — Vulkan Spec §Command Buffer Lifecycle: 状态转换图
- [P0] https://github.com/torvalds/linux/tree/v6.13/drivers/gpu/drm/scheduler — drm/scheduler: DRM Scheduler 实现、job 提交
- [P1] https://vulkan.lunarg.com/doc/view/1.4.335.1/mac/antora/tutorial/latest/04_Command_Buffer.html — Vulkan Tutorial §Command Buffers: 录制和提交
- [P1] https://docs.vulkan.org/spec/latest/chapters/cmdbuffers.html — Vulkan Spec §Command Buffers: 详细规范
- [P2] https://themaister.net/blog/2022/08/28/the-vulkan-command-buffer-model-explained/ — The Vulkan Command Buffer Model Explained (Themaister): 深度解析
