---
title: "典型电路设计收集"
date: 2025-03-15T20:13:40+08:00
draft: false
---

## 1.连接器保护电路（RC缓冲/TVS管）
* RC缓冲中，C保持两端电压不变有效抑制尖峰电压，而R作为限值分流电阻抑制住了尖峰电流。
* 目测全志的EVB电路中都有此电路，相当于滤波+TVS替代。

## 2.ESP复位电流需求>=500ma
* AMS1117 - 1A
* ME6217 - 800mA
* JY1106-ADJ - 600ma
* AP2127K - 300ma


## 3.PLL倍频和晶振
* 晶振起振提供ref_clk
* 鉴相器比较ref_clk与pll_out误差
* 误差经LF滤波拟合
* 电压进入VCO产生频率
![](/images/pll_rep.png)
