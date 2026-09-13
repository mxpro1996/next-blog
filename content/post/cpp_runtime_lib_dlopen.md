---
title: "[C/C++] 运行时装载 - libdl"
date: 2025-10-20
draft: false
tags: ["cpp", "动态加载", "共享库"]
categories: ["C++"]
---

## 概述

`<dlfcn.h>` 是 C 标准库中用于**动态加载共享库**的头文件，主要用于在运行时加载共享库（如 `.so` 文件或 `.dll` 文件），获取符号地址并调用其中的函数。

## 核心函数

| 函数 | 说明 |
|------|------|
| `dlopen` | 打开共享库文件，返回句柄 |
| `dlsym` | 从共享库中获取符号（函数/变量）地址 |
| `dlclose` | 关闭共享库 |
| `dlerror` | 获取最近一次 dl 函数的错误信息 |

---

## dlopen 模式

| 模式 | 说明 |
|------|------|
| `RTLD_LAZY` | 懒加载（首次调用时解析符号） |
| `RTLD_NOW` | 立即加载（返回前解析所有符号） |
| `RTLD_GLOBAL` | 符号可被后续加载的库使用 |
| `RTLD_LOCAL` | 符号仅当前库可见（默认） |

---

## 关键特性

### dlsym 查找符号
- 可以查找**函数**或**全局变量**的地址
- 返回 `void*`，需强转为正确的函数指针类型

### dlerror 错误处理
- 返回最近一次 dl* 函数的错误信息（字符串）
- 如果 `dlopen`/`dlsym` 失败，应检查error信息

---

## 典型应用场景

### 1. 插件（Plugin）
程序运行时加载插件（如浏览器插件、游戏模组）

### 2. 模块化设计（Module）
动态加载可选功能模块（如数据库驱动）

### 3. 热更新（HotReload）
不重启程序，替换共享库以实现更新

---
## 代码示例

### 1. 创建共享库

**math.c**（编译为共享库）：

```c
// 编译命令（Linux）: gcc -shared -fPIC -o libmath.so math.c
// Windows（MinGW）: gcc -shared -o math.dll math.c

int add(int a, int b) {
    return a + b;
}
```

### 2. 动态加载共享库并调用函数

**main.c**（动态加载 libmath.so 并调用 add 函数）：

```c
#include <stdio.h>
#include <dlfcn.h>  // Linux/macOS

int main() {
    void *handle;           // 共享库句柄
    int (*add_func)(int, int);  // 函数指针

    // 1. 加载共享库
    handle = dlopen("./libmath.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "Error: %s\n", dlerror());
        return 1;
    }

    // 2. 获取函数地址
    add_func = (int (*)(int, int)) dlsym(handle, "add");
    if (!add_func) {
        fprintf(stderr, "Error: %s\n", dlerror());
        dlclose(handle);
        return 1;
    }

    // 3. 调用动态加载的函数
    int result = add_func(3, 4);
    printf("3 + 4 = %d\n", result);

    // 4. 关闭共享库
    dlclose(handle);
    return 0;
}
```

**编译并运行（Linux）**：
```bash
gcc -o main main.c -ldl  # 链接 libdl 库
./main
3 + 4 = 7
```

---

## Windows 下的动态加载（对比）

Windows 使用 `<windows.h>` 中的 `LoadLibrary`/`GetProcAddress`：

```c
#include <stdio.h>
#include <windows.h>

int main() {
    HINSTANCE handle = LoadLibrary("math.dll");
    if (!handle) {
        fprintf(stderr, "Failed to load DLL\n");
        return 1;
    }

    // 获取函数地址
    int (*add_func)(int, int) = (int (*)(int, int)) GetProcAddress(handle, "add");
    if (!add_func) {
        fprintf(stderr, "Failed to find function\n");
        FreeLibrary(handle);
        return 1;
    }

    int result = add_func(3, 4);
    printf("3 + 4 = %d\n", result);

    FreeLibrary(handle);
    return 0;
}
```

---

## 注意事项

### 1. 跨平台兼容性
- **Linux/macOS**：`<dlfcn.h>`
- **Windows**：`LoadLibrary`/`GetProcAddress`

### 2. 名称修饰（C++）
C++ 函数名会编译成修饰名（如 `_Z3addii`），需用 `extern "C"` 避免：

```cpp
extern "C" {
    int add(int a, int b) { return a + b; }
}
```

### 3. 内存泄漏
确保 `dlclose` 关闭不再使用的库

