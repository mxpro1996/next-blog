---
title: （零）Linux内核驱动 - 资料推荐
date: 2025-10-01T08:07:25+08:00
draft: false
---

### 驱动开发
1. 《Linux设备驱动开发_（法）约翰·马迪厄（John Madieu）》
2. 《Linux设备驱动开发详解_宋宝华》
3. 《Android驱动开发权威指南_杨柳》
---
### 内核扫盲
1. 《Linux内核深度解析 基于ARM64架构的Linux 4.x内核_余华兵》
2. 《从零开始写Linux内核 基于0.12_海纳》
---
### 应用编程
1. 《Linux/UNIX系统编程手册（上、下册）》
---
### 源代码/文档检索
1. [Bootlin 在线看](https://elixir.bootlin.com)
2. [Linux_官方文档](https://www.kernel.org/doc/html/latest/index.html)
3. [DTS设备树_官方白皮书](https://www.devicetree.org/specifications/)
---
### 分层模型
```markdown
/real_kernel (实际内核体)
    /kernel        (内核核心功能：调度、时钟、系统调用等)
    /mm            (内存管理：虚拟内存、页表管理、内存分配等)
    /ipc           (进程间通信：信号量、消息队列、共享内存等)
    /init          (系统初始化：内核环境准备、启动过程)
-----------------------------
/resource_core （系统资源抽象，主体Core层）
    /fs             (文件系统：ext4、Btrfs、NTFS 等)
    /block          (块设备：磁盘、存储设备等)
    /net            (网络协议栈：TCP/IP、IPv6、路由等)
    /crypto         (加密与解密支持)
    /sched          (调度器：CPU 调度、负载均衡等)
-----------------------------
/drivers （驱动实化具体资源，常见接口也有core封装的协议栈）
    /drivers        (硬件驱动：网络、存储、音频、显示等设备驱动)
    /arch           (特定硬件架构支持：ARM、x86等)
    /dts            (设备树：硬件资源配置)

```
