---
title: "[udev] device_create()与/dev/xx"
date: 2025-05-06T00:00:37+08:00
draft: false
---

&emsp;理论上，device_create()会发送uevent触发用户态udev进行/dev/xxx设备文件的建立，但就算禁止udev，devtmpfs也会在device_add()函数被调用时收到信息，建立基本设备文件节点。

&emsp;综上所述，device_create()会在sysfs的设备类class下建立设备实例的同时，触发/dev/xxx设备文件的创建。



