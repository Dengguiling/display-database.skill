# KB-LINUX-DRM 术语表

| 术语 | 定义 | 首次出现 |
|------|------|----------|
| Atomic Commit | 将 DRM 状态变更原子性地提交到硬件的完整流程 | KB-LINUX-DRM-atomic-001 |
| atomic_check | Atomic commit 的同步校验阶段，验证状态变更合法性 | KB-LINUX-DRM-atomic-001 |
| commit_tail | 非阻塞提交中在 worker thread 执行的实际硬件更新函数 | KB-LINUX-DRM-atomic-001 |
| hw_done | completion 信号，表示硬件寄存器写入完成 | KB-LINUX-DRM-atomic-001 |
| flip_done | completion 信号，表示 VBlank 到来、新缓冲区已扫描输出 | KB-LINUX-DRM-atomic-001 |
| cleanup_done | completion 信号，表示旧缓冲区已释放 | KB-LINUX-DRM-atomic-001 |
| nonblock | atomic commit 的非阻塞模式标志 | KB-LINUX-DRM-atomic-001 |
| modeset | 显示模式变更（分辨率/时序等），区别于 fast path 的 plane 更新 | KB-LINUX-DRM-atomic-001 |
| CRTC | Cathode Ray Tube Controller，DRM 中表示显示控制器 | KB-LINUX-DRM-atomic-001 |
| Plane | 硬件叠加层，表示一个可独立扫描的图像源 | KB-LINUX-DRM-atomic-001 |
| dma-buf | Linux 内核共享缓冲区机制，用于跨设备/跨进程缓冲区共享 | KB-LINUX-DRM-atomic-001 |
