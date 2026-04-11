# DRM VBlank 事件机制

> **条目ID:** KB-LINUX-DRM-atomic-005
> **模块:** KB-LINUX-DRM / atomic
> **置信度:** 5/5（基于 Linux v6.13 内核源码 + kernel.org 官方文档）
> **时效性:** 2026-04-12 | **信源:** kernel.org, torvalds/linux v6.13
> **关联:** [commit-flow](./commit-flow.md) | [nonblocking](./nonblocking.md) | [helpers](./helpers.md)

---

## 核心摘要

VBlank（垂直消隐）是 CRT 显示器遗留的概念，指电子束从屏幕底部回到顶部的扫描间隔。在现代 GPU 驱动中，VBlank 事件是**帧同步的核心原语**——它标记着一帧渲染的完成和下一帧的开始。DRM 子系统通过 `drm_vblank.c` 实现了完整的 VBlank 管理：硬件中断驱动的高精度计数器、基于 seqlock 的时间戳、引用计数的中断启停控制、以及事件队列驱动的异步通知机制。这套机制是页面翻转、VSync 同步、帧率统计和 tearing-free 渲染的基础。

---

## 1. VBlank 基础概念

### 什么是 VBlank

```
帧 N 扫描                    帧 N+1 扫描
┌──────────────────┐         ┌──────────────────┐
│  Line 0          │         │  Line 0          │
│  Line 1          │         │  Line 1          │
│  ...             │         │  ...             │
│  Line (H-1)      │← VBlank→│  Line (H-1)      │
└──────────────────┘         └──────────────────┘
                    ↑
              VBlank 中断触发
              交换 Framebuffer
              发送 VBlank 事件
```

- **VBlank 间隔：** 从最后一行扫描结束到第一行扫描开始的时间窗口
- **典型时长：** 60Hz 显示器约 1.1ms（占总帧时间 16.67ms 的 ~6.5%）
- **关键用途：** 在此期间交换 Framebuffer 可避免 Tearing

### DRM 中的 VBlank 角色

| 角色 | 说明 |
|------|------|
| 帧同步 | Compositor 在 VBlank 时翻转页面，确保 Tearing-free |
| 帧率控制 | 通过 VBlank 计数器计算实际帧率 |
| 时间戳 | 高精度 VBlank 时间戳用于 A/V 同步和输入延迟测量 |
| 流水线同步 | Atomic commit 在 VBlank 时应用新的显示状态 |
| 性能监控 | `perf` 等工具通过 VBlank 事件统计 GPU 渲染延迟 |

---

## 2. 核心数据结构

### `struct drm_vblank_crtc`
```c
struct drm_vblank_crtc {
    struct drm_device *dev;          // 所属 DRM 设备
    wait_queue_head_t queue;         // VBlank 等待队列
    struct timer_list disable_timer; // 延迟禁用定时器
    seqlock_t seqlock;               // 保护 count 和 time 的读写锁
    atomic64_t count;                // 软件 VBlank 计数器
    ktime_t time;                    // 最近 VBlank 时间戳
    atomic_t refcount;               // 引用计数（0 时可禁用中断）
    u32 last;                        // 上次读取的硬件计数器值（wraparound 处理）
    u32 max_vblank_count;            // 硬件计数器最大值（0 = 无限制）
    bool enabled;                    // VBlank 中断是否已启用
    bool inmodeset;                  // 是否处于 modeset 期间
    struct drm_vblank_crtc_config config; // 配置参数
};
```

### `struct drm_vblank_crtc_config`
```c
struct drm_vblank_crtc_config {
    int offdelay_ms;            // VBlank 禁用延迟（毫秒）
    bool disable_immediate;     // 是否立即禁用（无延迟）
};
```

### `drm_device` 中的 VBlank 相关字段
```c
struct drm_device {
    bool vblank_disable_immediate;     // 全局默认：是否立即禁用 VBlank
    struct drm_vblank_crtc *vblank;    // per-CRTC VBlank 状态数组
    spinlock_t vblank_time_lock;       // 保护 VBlank 时间戳
    spinlock_t vbl_lock;               // 保护 VBlank 状态变更
    u32 max_vblank_count;              // 全局最大 VBlank 计数
    struct list_head vblank_event_list; // 待发送的 VBlank 事件列表
    spinlock_t event_lock;             // 保护事件列表
};
```

---

## 3. VBlank 生命周期管理

### 3.1 初始化

```c
int drm_vblank_init(struct drm_device *dev, unsigned int num_crtcs);
```

- 在驱动 `probe()` 阶段调用
- 分配 `num_crtcs` 个 `drm_vblank_crtc` 实例
- 初始化等待队列、定时器、seqlock、自旋锁

### 3.2 引用计数控制

```c
int drm_crtc_vblank_get(struct drm_crtc *crtc);   // 引用 +1，必要时启用中断
void drm_crtc_vblank_put(struct drm_crtc *crtc);   // 引用 -1，0 时启动禁用定时器
```

**引用计数语义：**
- `get()` 时：如果 `refcount` 从 0→1，启用 VBlank 中断
- `put()` 时：如果 `refcount` 从 1→0，启动 `disable_timer`
- `disable_timer` 到期后：实际禁用 VBlank 中断
- **延迟禁用（hysteresis）：** 避免频繁启停中断的开销

### 3.3 禁用策略

```
drm_crtc_vblank_put()
    │
    ├─ refcount > 0 → 仅递减，不做其他操作
    │
    └─ refcount == 0
        │
        ├─ disable_immediate = true → 立即禁用中断
        │
        └─ disable_immediate = false → 启动 disable_timer
            │
            └─ offdelay_ms 后（默认 5 秒）
                │
                └─ 实际禁用 VBlank 中断
```

**配置方式：**
- 全局默认：`drm_device.vblank_disable_immediate`（模块参数 `vblankoffdelay`）
- Per-CRTC 覆盖：`drm_vblank_crtc_config.disable_immediate`
- Per-CRTC 延迟：`drm_vblank_crtc_config.offdelay_ms`

---

## 4. VBlank 计数器与时间戳

### 4.1 计数器读取

```c
u64 drm_crtc_vblank_count(struct drm_crtc *crtc);
u64 drm_crtc_vblank_count_and_time(struct drm_crtc *crtc, ktime_t *vblanktime);
```

**计数器语义：**
- 使用 `seqlock` 保护，保证读取的原子性
- `count` 是 64 位原子变量，不会溢出
- **内存屏障保证：** `drm_crtc_handle_vblank()` 之前的写入对后续 `drm_crtc_vblank_count()` 调用可见（当 count 相同或更大时）

### 4.2 硬件计数器 vs 软件计数器

| 类型 | 来源 | 精度 | 适用场景 |
|------|------|------|----------|
| 硬件计数器 | GPU 寄存器 | 取决于硬件 | 大多数现代 GPU |
| 软件计数器 | 中断驱动递增 | 中断精度 | 无硬件计数器的设备 |

**硬件计数器配置：**
```c
// 驱动设置最大值（wraparound 处理）
drm_crtc_set_max_vblank_count(crtc, max_count);

// 驱动提供硬件计数器读取回调
struct drm_crtc_funcs {
    u32 (*get_vblank_counter)(struct drm_crtc *crtc);
};
```

**软件计数器回退：**
- 当 `max_vblank_count == 0` 时，DRM 核心使用高精度系统时间戳推算遗漏的 VBlank
- 适用于 VBlank 中断被禁用期间的计数补偿

### 4.3 高精度时间戳

```c
bool drm_crtc_vblank_helper_get_vblank_timestamp(
    struct drm_crtc *crtc, int *max_error,
    ktime_t *vblank_time, bool in_vblank_irq);
```

- 驱动通过 `drm_crtc_funcs.get_vblank_timestamp` 回调提供
- 使用 `gettime64()` 系统调用获取高精度时间
- DRM 核心会多次重试以获取一致的时间戳（处理寄存器读取竞争）
- **用途：** A/V 同步、帧时间测量、渲染延迟分析

---

## 5. VBlank 事件机制

### 5.1 事件发送

```c
void drm_crtc_handle_vblank(struct drm_crtc *crtc);
void drm_send_vblank_event(struct drm_device *dev, unsigned int pipe,
                           struct drm_pending_vblank_event *e);
```

**`drm_crtc_handle_vblank()` 流程：**
```
VBlank IRQ 触发
    │
    ├─ 递增 vblank->count
    │
    ├─ 更新 vblank->time（高精度时间戳）
    │
    ├─ 唤醒 vblank->queue 上的等待者
    │
    └─ 遍历 vblank_event_list
        │
        └─ 对每个到期事件调用 drm_send_vblank_event()
            │
            ├─ 通过 event_lock 保护
            │
            ├─ 将事件添加到 fasync 列表
            │
            └─ 唤醒 poll() 等待者
```

### 5.2 事件等待

```c
int drm_wait_vblank(struct drm_device *dev, union drm_wait_vblank *vbl);
```

- 用户空间通过 `DRM_IOCTL_WAIT_VBLANK` 调用
- 支持两种模式：
  - **相对等待：** 等待 N 个 VBlank 后返回
  - **绝对等待：** 等待 VBlank 计数器达到指定值
- 使用 `wait_queue_head_t` 实现阻塞等待

### 5.3 事件结构

```c
struct drm_pending_vblank_event {
    struct drm_pending_event base;
    unsigned int pipe;
    u64 sequence;
    struct timespec64 tv;
    struct drm_file *file_priv;
};
```

---

## 6. Fake VBlank（无硬件中断场景）

```c
// 在 drm_crtc_state 中设置
crtc_state->no_vblank = true;
```

**适用场景：**
- 简单的 LCD 控制器（无 VBlank 中断）
- 虚拟 GPU（如 vkms）
- 某些嵌入式显示控制器

**行为：**
- Atomic commit helper 在 commit 完成后发送 fake VBlank 事件
- 不依赖硬件中断，使用定时器模拟
- **限制：** 无精确时间戳，帧率统计不准确

---

## 7. VBlank 与 Atomic Commit 的交互

### 在 Commit 流程中的位置

```
atomic_commit(nonblock=true)
    │
    ├─ drm_atomic_helper_setup_commit()
    │   └─ 对每个受影响的 CRTC 调用 drm_crtc_vblank_get()
    │
    ├─ [worker thread] commit_tail()
    │   ├─ 硬件编程
    │   ├─ drm_atomic_helper_wait_for_vblanks()
    │   │   └─ 等待所有受影响 CRTC 的 VBlank 事件
    │   ├─ drm_atomic_helper_commit_hw_done()
    │   │   └─ 完成 hw_done 信号
    │   └─ drm_atomic_helper_commit_cleanup_done()
    │       └─ 对每个 CRTC 调用 drm_crtc_vblank_put()
    │
    └─ 用户空间收到 VBlank 事件 → 翻转完成
```

### VBlank 等待策略

```c
void drm_atomic_helper_wait_for_vblanks(struct drm_device *dev,
                                         struct drm_atomic_state *old_state);
```

- 对每个受影响的 CRTC 等待一个 VBlank
- 使用 `wait_event_timeout()` 实现
- **超时处理：** 如果 VBlank 未在合理时间内到达，强制继续（避免死锁）

---

## 8. 驱动实现要点

### 最小驱动实现

```c
// 1. 初始化
drm_vblank_init(dev, num_crtcs);

// 2. 提供 VBlank 计数器回调
static u32 my_driver_get_vblank_counter(struct drm_crtc *crtc)
{
    return readl(crtc_regs + VBLANK_COUNTER);
}

// 3. 在 IRQ handler 中调用
static irqreturn_t my_driver_irq(int irq, void *arg)
{
    // ... 检测 VBlank IRQ ...
    drm_crtc_handle_vblank(crtc);
    return IRQ_HANDLED;
}

// 4. 启用/禁用 VBlank 中断
static int my_driver_enable_vblank(struct drm_crtc *crtc)
{
    writel(VBLANK_IRQ_EN, crtc_regs + IRQ_CTRL);
    return 0;
}

static void my_driver_disable_vblank(struct drm_crtc *crtc)
{
    writel(0, crtc_regs + IRQ_CTRL);
}
```

### 高精度时间戳实现

```c
static bool my_driver_get_vblank_timestamp(struct drm_crtc *crtc,
    int *max_error, ktime_t *vblank_time, bool in_vblank_irq)
{
    struct timeval tv;
    // 使用硬件寄存器获取精确时间
    u32 raw = readl(crtc_regs + VBLANK_TIMESTAMP);
    tv.tv_sec = raw >> 16;
    tv.tv_usec = (raw & 0xFFFF) * 64; // 假设 64us 精度
    *vblank_time = timeval_to_ktime(tv);
    return true;
}
```

---

## 9. VBlank Timer（无硬件中断的替代方案）

对于没有 VBlank 中断的硬件，DRM 提供了基于 hrtimer 的模拟方案：

```c
void drm_vblank_init_timer(struct drm_device *dev, unsigned int pipe);
```

- 使用高精度定时器模拟 VBlank 事件
- 基于当前显示模式的刷新率计算定时器间隔
- **精度限制：** 依赖系统定时器精度（通常 1-4ms），不如硬件中断精确
- **适用：** 简单嵌入式显示控制器

---

## 潜在影响

- **帧率稳定性：** VBlank 事件的精确性直接影响 compositor 的帧调度质量。高精度时间戳（ns 级）是 Wayland PipeWire A/V 同步的基础。
- **功耗优化：** `disable_immediate` 和 `offdelay_ms` 的配置直接影响 GPU 空闲功耗。移动设备通常使用较短的 offdelay 以快速进入低功耗状态。
- **VR/AR 场景：** VBlank 时间戳的精度直接影响运动到光子（MTP）延迟的测量和优化。
- **演进方向：** 社区在 v6.x 中持续优化 VBlank 时间戳精度，新增了 `calc_timestamping_constants` 辅助函数，简化了驱动的时间戳计算。

---

## 信源

- [DRM Internals — kernel.org](https://docs.kernel.org/gpu/drm-internals.html)
- [DRM KMS — kernel.org](https://docs.kernel.org/gpu/drm-kms.html)
- [DRM KMS Helpers — kernel.org](https://docs.kernel.org/gpu/drm-kms-helpers.html)
- [drm_vblank.h — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/include/drm/drm_vblank.h)
- [drm_vblank.c — Linux v6.13](https://github.com/torvalds/linux/blob/v6.13/drivers/gpu/drm/drm_vblank.c)
