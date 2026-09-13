---
title: "[EDK2] 黑盒调试吐槽 - Trace32，你让人着迷"
date: 2026-05-25T18:07:25+08:00
draft: false
---

开篇自然是暴论，bootloader系/裸机Bare-Metal的套路无非都是:
```c
asmEntry -> CEntry -> Main -> target_drvInit(UART_Init) -> 'do Something'
```

## 剖析
既然平铺直叙，为什么又险象环生？\
所谓日志，归根究底得CLK、IOMUX和UART起来才有戏，熔断设备CrashDump的日志能力就是半残废；

不是流水线之类的嗝屁或者时序完蛋，也炸不到sbl去；

由lk二次引导拉起的edk2无非是鸠占鹊巢，时不时来点冷启动初始化，一看卧槽boot_linux后一行log都没了。\
天知道是SMP还是缓存一致性送走了，没有trace32的情况下A核真的就是黑盒，M核调试器什么山寨Link都能起飞。

## 少来意见
A：这么烦printk不好用，哪天把你JTAG也给熔断了。到时候让你QEMU调试virt & Cortex-A53就老实了。\
B：QEMU还真不错，remote-gdb调点东西绰绰有余，usermode-linux也是类似的产物。


A：那就点灯大法或者打印字符解耦做个shim的代码片段，插直观桩不就好了？\
B：说起来容易。