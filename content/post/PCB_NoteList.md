---
title: "[Cadence] 学习笔记录入"
date: 2024-10-17T11:07:25+08:00
draft: false
---
## 1. [光绘层Artwork] 设定自定义可见关系集
### 详情：
1. SOLDERMASK（阻焊） 
   * Pin/SOLDMASK 
   * Package Geometry/SOLDMASK
2. PASTEMASK （锡膏）
   * Pin/PASTEMASK 
   * Package Geometry/PASTEMASK
3. SILKSCREEN （丝印）
   * RefDes/SILKSCREEN
   * Package Geometry/SILKSCREEN
4. TOP/BOT （顶/底层预览）
   * Pin
   * Etch
   * Via class
5. ADT/ADB SMT装配（位号图）
   * 丝印
   * RefDes/SILKSCREEN
   * Package Geometry/SILKSCREEN
   * 引脚 -> Pin
   * 板框 -> Outline 
