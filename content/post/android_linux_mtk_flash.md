---
title: "[Debian12]  FlashTool 无法连接到BROM/PRELOADER"
date: 2026-09-08T10:24:58+08:00
draft: false
---
近期在从ubuntu22.04迁移到debian12时，内核也从5.15到了6.1，随即发现sp flash握手失败。

### 前言
常规都是排查udev和ModemManager这俩老古董，但是笔者遇到的情况和ioctl相关：
1. udev其实Linux的sp_flashtool，会自动注入ttyACM的规则 
2. ModemManager作为早期USB-AT基带探测，目前其实抢占少见
3. 当前用户对串口没有权限
```bash
# 对于UDEV
# ATTRS{idVendor}=="0e8d", ENV{ID_MM_DEVICE_IGNORE}="1"
sudo install -m 644 99-ttyacms.rules /etc/udev/rules.d/99-ttyacms.rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# 对于ModemManager
sudo service ModemManager stop >> /dev/null 2>&1 &

# 对于无权限
sudo usermod -aG plugdev,dialout,uucp,tty $USER && newgrp dialout

```

---
### 特例分析
首先是FlashTool log，错误码都是libFLashTool_v1.so的没看点
```c
Total wait time = -1262318425.000000
USB port is obtained. path name(/dev/ttyACM0), port name(/dev/ttyACM0)
USB port detected: /dev/ttyACM0
Connect BROM failed: S_COM_PORT_OPEN_FAIL(1013)
Disconnect!
BROM Exception! ( ERROR : S_COM_PORT_OPEN_FAIL (1013)

[COM] Failed to open COM port.
[HINT]:
Please retry the following steps:
1. retry a new cable;
2. retry a new com port;
3. retry a new device;
4. restart the computer then retry again.)((ConnectBROM,../../../flashtool/Conn/Connection.cpp,103))
```

然后是BROM log，明显open已经成功，问题的是TIOCCBRK
```c
BROM_DLL[7929][7934]: ERROR: com_base::open(/dev/ttyACM0): open fail! , 13(Permission denied) (com_base.cpp:414)
com_base::open(/dev/ttyACM0): retry (com_base.cpp:418)
com_base::reset(11): TIOCCBRK fail! 95(Operation not supported). (com_base.cpp:375)
com_base::open(/dev/ttyACM0): reset fail! , 95(Operation not supported) (com_base.cpp:428)
com_sentry::Open(0xffffffffffffffff): open fail, 95(Operation not supported) (com_sentry.cpp:359)
com_base::bOK(-1): not ready (,255,0x434f4d5f), line[ 453] (com_base.cpp:214)
```
---
### 修复逻辑
重新执行编译命令，修订flash_tool.sh注入LD_PRELOAD
```bash
gcc -shared -fPIC hack_ioctl.c -o libhack_ioctl.so -ldl
sed -i 's|\$dirname/\$appname "\$@"|LD_PRELOAD="\$dirname/libhack_ioctl.so" \$dirname/\$appname "\$@"|g' flash_tool.sh
```
hack_ioctl.c内容如下：
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <dlfcn.h>
#include <sys/ioctl.h>
#include <termios.h>
#include <stdarg.h>
#include <errno.h>

typedef int (*orig_ioctl_type)(int fd, unsigned long request, ...);

int ioctl(int fd, unsigned long request, ...) {
    orig_ioctl_type orig_ioctl = (orig_ioctl_type)dlsym(RTLD_NEXT, "ioctl");
    
    va_list args;
    va_start(args, request);
    void *arg = va_arg(args, void *);
    va_end(args);

    if (request == TIOCCBRK || request == TIOCSBRK) {
        int ret = orig_ioctl(fd, request, arg);
        if (ret < 0) {
            printf("[Hook Log] 拦截到 Break ioctl (0x%LX) 失败 (errno=%d)! 伪造返回.\n", request, errno);
            return 0; 
        }
        return ret;
    }

    return orig_ioctl(fd, request, arg);
}
```

---
# 后记
这个TIOCCBRK没有细看，可能是类似CAN的SYNC头，用于VCOM/BROM与PC通信同步的。拦下老古董的强制判断，迎接喜闻乐见的flashtool红条。
![no](/images/spflash_daOK.png)
