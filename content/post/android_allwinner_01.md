---
title: "[AOSP] 古早编译常见CASE"
date: 2026-08-13T10:24:58+08:00
draft: false
---
旧版sunxi如A13 A20 A33出厂多于ubuntu 12.04-18.04时代，现今主流宿主机API迁移，行为变更导致诸多不切合。\
至于安卓4后，虚拟机需要32还是64位这里不多言。
### 32位库依赖
```bash
sudo apt-get install libc6-dev gcc-multilib lib32z1-dev \
                    flex m4 bc -yq
```

### JDK6-7安装
```bash
# Fetch and Installing
wget -q https://repo.huaweicloud.com/java/jdk/6u45-b06/jdk-6u45-linux-x64.bin
chmod +x jdk-6u45-linux-x64.bin
./jdk-6u45-linux-x64.bin
mkdir -p /usr/java
mv jdk1.6.0_45 /usr/java

# Setting for JDK ENV
cat <<EOF >>/etc/profile
export JAVA_HOME=/usr/java/jdk1.6.0_45
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
EOF
```

### 绕过MAKE版本限制
编辑build/core/main.mk，修订字面量为本机版本
```bash
-44 ifeq (0,$(shell expr $$(echo $(MAKE_VERSION) | sed "s/[^0-9\.].*//") = 3.82))
+45 ifeq (0,$(shell expr $$(echo $(MAKE_VERSION) | sed "s/[^0-9\.].*//") = 4.1))          #<----------------------在这里让make4.1可以编译
```

### BISON降级
bison-2.7左右恰好兼容，v3版本对老源码树SDK水土不服，会出现"*.hpp"和"*.h"混淆的历史遗留问题，不降级会要手动重命名。
```bash
wget http://ftp.gnu.org/gnu/bison/bison-2.7.tar.gz
tar -xf bison-2.7.tar.gz
cd bison-2.7
./configure && make -j8 && make install

# 若出现fseterror错误，使用下面的补丁
cat > fix-fseterr.patch << 'EOF'
#if !defined _IO_IN_BACKUP && defined _IO_EOF_SEEN
# define _IO_IN_BACKUP 0x100
#endif
--- a/lib/fseterr.c
+++ b/lib/fseterr.c
@@ -29,7 +29,7 @@
-#if defined _IO_ftrylockfile || __GNU_LIBRARY__ == 1
+#if defined _IO_EOF_SEEN || __GNU_LIBRARY__ == 1
    fp->_flags |= _IO_ERR_SEEN;
#elif defined __sferror || defined __DragonFly__
    fp_->_flags |= __SERR;
EOF
# 应用补丁
patch -p1 < fix-fseterr.patch

```

### 内核单独编译，嵌套头文件无视
当foo.c指向另一个路径的bar.h后，bar.h又依赖了foo.h这个和最初编译单元相关的配套头文件。\
在源码树Makefile顶层，使用追加宏"NOSTDINC_FLAGS"
```bash
diff --git a/kernel-3.10/Makefile b/kernel-3.10/Makefile

+NOSTDINC_FLAGS += -I$$(srctree)/$$(src)
 KBUILD_AFLAGS_KERNEL :=
 KBUILD_CFLAGS_KERNEL :=
 KBUILD_AFLAGS   := -D__ASSEMBLY__
```
