---
title: "[OV2640] 记一次设备树盲目CV的闹剧"
date: 2025-05-06T00:00:33+08:00
draft: false
---

&emsp;在编写ov2640节点时，直接抄了部分5640的内容，差异点导致probe直接失败。作为盲目CV的教训
1. 'xvclk'而不是'xclk'[OV5640]
2. pwdn-gpios而不是powerdown-gpios
3. rstb-gpios而不是resetb-gpios \

&emsp;建议往后调试启用dbg_print日志，以免不知驱动出错详情。

