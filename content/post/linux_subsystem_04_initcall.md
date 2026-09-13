---
title: "[转]（四）Linux 内核驱动 - InitCall机制"
date: 2025-10-12T09:02:36+08:00
draft: false
---
### 1. **initcall机制的起源与背景**

当我们将驱动编译进内核时，需要在内核启动时调用相应的初始化函数。最简单的方式是，在系统启动时直接在代码中插入调用，但这显然不适用于大型的Linux内核，因为驱动程序数量庞大，且有一定的依赖关系，必须按特定顺序进行初始化。

因此，Linux提供了 **initcall** 机制，通过一种分级的方式管理内核模块的初始化过程。该机制能够自动化地调用各个驱动的初始化函数，并保证初始化的顺序。

### 2. **initcall的核心实现**

Linux的 `initcall` 系统利用了GCC的 **section属性** 将初始化函数放置到内核镜像的特定段中。每个初始化函数都会通过 `xxx_initcall` 宏被标记，并被自动加入到一个特定的段中（如 `.initcall0.init`，`.initcall1.init` 等）。内核启动时会扫描这些段，并按顺序执行初始化函数。

#### 2.1 **initcall的宏定义**

在 `init.h` 文件中，定义了多个 **initcall** 宏，每个宏对应不同的初始化优先级，优先级数字越小，初始化越早。宏展开后，每个函数会被放置到一个名为 `.initcall<level>.init` 的段中：

* `early_initcall(fn)`
* `pure_initcall(fn)`
* `core_initcall(fn)`
* `postcore_initcall(fn)`
* `arch_initcall(fn)`
* `subsys_initcall(fn)`
* `fs_initcall(fn)`
* `device_initcall(fn)`
* `late_initcall(fn)`

每个 `fn` 被宏包装后，会使用 GNU 的 `__attribute__((section(".initcall<level>.init")))` 将其放置在特定段中。`initcall` 的优先级由数字指定，数字越小，表示初始化越早。

#### 2.2 **initcall的段和链接**

这些初始化函数被放置到内核映像文件中的特定段内，具体的段由 `vmlinux.lds.h` 文件控制，以下为相关部分：

```c
#define INIT_CALLS_LEVEL(level) \
    VMLINUX_SYMBOL(__initcall##level##_start) = .; \
    KEEP(*(.initcall##level##.init)) \
    KEEP(*(.initcall##level##s.init))

#define INIT_CALLS \
    VMLINUX_SYMBOL(__initcall_start) = .; \
    KEEP(*(.initcallearly.init)) \
    INIT_CALLS_LEVEL(0) \
    INIT_CALLS_LEVEL(1) \
    INIT_CALLS_LEVEL(2) \
    INIT_CALLS_LEVEL(3) \
    INIT_CALLS_LEVEL(4) \
    INIT_CALLS_LEVEL(5) \
    INIT_CALLS_LEVEL(6) \
    INIT_CALLS_LEVEL(7) \
    VMLINUX_SYMBOL(__initcall_end) = .;
```

这个定义将初始化函数放置在不同的段中（`.initcall0.init`, `.initcall1.init` 等），并在内核启动时通过 `do_initcalls` 进行调用。

### 3. **初始化函数的执行流程**

内核启动时，会依次调用各个初始化函数。这个过程从 `start_kernel` 函数开始，具体的调用顺序如下：

1. `start_kernel` -> `rest_init()`
2. `rest_init()` -> `kernel_thread(kernel_init, NULL, CLONE_FS)`
3. `kernel_init()` -> `kernel_init_freeable()`
4. `do_basic_setup()` -> `do_initcalls()`

`do_initcalls` 会按照优先级执行各个初始化函数。优先级由 `initcall_levels` 数组管理，该数组包含了各个初始化段的指针数组：

```c
static initcall_t *initcall_levels[] __initdata = {
    __initcall0_start,
    __initcall1_start,
    __initcall2_start,
    __initcall3_start,
    __initcall4_start,
    __initcall5_start,
    __initcall6_start,
    __initcall7_start,
    __initcall_end,
};
```

`do_initcalls` 会遍历这些级别（0-7），依次执行每个段中的函数。

#### 3.1 **函数指针执行**

在 `do_initcall_level(level)` 中，系统会遍历某一段中的所有函数指针，并通过 `do_one_initcall(fn)` 执行：

```c
static void __init do_initcall_level(int level) {
    initcall_t *fn;
    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
        do_one_initcall(*fn);
}
```

每次执行函数时，会判断是否需要调试输出：

```c
int __init_or_module do_one_initcall(initcall_t fn) {
    if (initcall_debug)
        return do_one_initcall_debug(fn);
    else
        return fn();
}
```

实际的初始化函数就是通过 `fn()` 被调用。

### 4. **应用与实践**

#### 4.1 **驱动开发中的应用**

例如，一个名为 `beagle` 的驱动，开发者希望在内核启动时调用 `beagle_init()` 来初始化该驱动。使用 `core_initcall(beagle_init)` 宏后，`beagle_init()` 函数会被放置在 `.initcall1.init` 段中，并在系统启动时被执行。

#### 4.2 **模块与内核的区别**

需要注意的是，**initcall机制仅适用于内核编译进的驱动**，对于动态加载的模块，内核没有类似机制来控制它们的初始化顺序。对于模块，通常通过 `module_init` 宏来声明初始化函数。

### 5. **总结与思考**

* **优点**：initcall机制极大地简化了内核的初始化过程，自动化管理多个驱动和组件的初始化顺序，避免了开发者手动管理。
* **分级管理**：通过对初始化函数的分级管理（如 `early_initcall`, `core_initcall` 等），内核可以灵活地控制各个模块的启动顺序，确保依赖关系正确。
* **内存布局**：`__attribute__((section()))` 和链接器的 `.initcallX.init` 段的配合，使得内核能够有效地管理初始化函数并提高系统启动时的可扩展性。

这个机制的核心思想是通过特定的宏和段，将初始化函数和其优先级绑定，避免了手动管理启动顺序的繁琐。

