---
title: "[MTK] BringUp 01 - LCM Driver 移植流程"
date: 2026-04-20  
Category: Embedded System / Architecture  
Tags: [MT6797, AOSP, 嵌入式开发]
---


## 1. 硬件资源配置：从 DrvGen 到 DTS
在点亮屏幕之前，必须确保电源、复位（RST）及背光（PWM）的通路正确。

*   **Legacy 模式 (Kernel ≤ 3.10)**：使用 `mtk_legacy` 的 GPIO 控制方式，开发者需通过 `DrvGen.exe` 工具配置表格并生成头文件。
*   **现代模式 (Kernel > 3.10)**：转向 `gpiod` 封装配合 **Device Tree (DTS)** 描述。
    *   **关键点**：检查总线类型。MIPI 接口需关注 Lane 数量及 D-PHY 规格；RGB 接口则需确认并行数据位的极性。
---
## 2. 驱动注册：
MTK封装的子系统mtk_fb采用列表注册机制，允许内核在启动时探测不同的模组。

*   **修改文件**：`mt65xx_lcm_list.c` 与 `mt65xx_lcm_list.h`。
*   **操作流**：
    1.  在头文件中追加 `extern` 声明（例如 `extern LCM_DRIVER your_lcm_drv;`）。
    2.  在 `lcm_driver_list[]` 结构体数组中注册该指针。
*   **探测逻辑**：系统会根据 `lcm_name`做匹配, lk/kernel早期阶段LCM NAME失配会触发panic
```c
.../misc/mediatek/lcm/mt65xx_lcm_list.h 
+#ifdef VENDOR_EDIT
+extern LCM_DRIVER otm1284a_hd720_dsi_vdo_tm_lcm_drv;
+#endif

.../misc/mediatek/lcm/mt65xx_lcm_list.c 
LCM_DRIVER *lcm_driver_list[] = {
	...
+	#if defined(OTM1284A_HD720_DSI_VDO_TM)
+	&otm1284a_hd720_dsi_vdo_tm_lcm_drv,
+	#endif
	...
};
```
---

## 3. 环境变量与编译定义
*   **defconfig 修改**：在相应的 Project Config 文件（如 `CONFIG_CUSTOM_KERNEL_LCM`）中追加宏定义。
*   **Makefile 关联**：确保驱动目录下的 `Makefile` 能够根据该宏正确索引到LCD厂家的源文件。
---
## 4. Checklist
移植过程中，若屏幕未如期点亮，排查：

| 检查项 | 关键点 |
| :--- | :--- |
| **电压 (VCC/VSP/VSN)** | 确认 1.8V 和 5.xV 差分电压是否由 PMIC 正常输出。 |
| **时序参数** | HFP/HBP/VFP/VBP 是否严格符合 DataSheet 规格。 |
| **LP 模式指令** | MIPI 初始化指令是否需要在 Low Power 模式下发送。 |
| **内核日志** | 通过`dmesg \| grep lcm`查看探测阶段是否报错。 |
| **CrashDump** | 通过aee异常捕获的mtklog和expdb解析得到挂死现场 |
| **ID手误** | lk/kernel早期lcd_name失配会触发panic |