---
title: "[JNI查错] 口袋侦探闪退修复（二） - 安卓5及以上"
date: 2026-05-14T07:15:31+08:00
categories: ["逆向"]
tags: ["安卓", "JNI", "口袋侦探", "逆向"]
draft: false
---
**原理分析 见上一篇文章**

## 核心观点
安卓5-7作为早期ART限制REF总数<=512，不适用上一篇文章的暴力手段。\
（前一章**汇编覆盖**很优雅，只要改4-6个字节）\
本章是游戏bug的**根治**手段，提升局部引用到globalRef

## 切入点
（厂商懒缓存了java侧sqliteDBManager的局部引用，当全局用）\
**我们就坡下驴，把这个局部引用拦截提升到全局。** 
```bash
.text:000BEF12                 LDR             R1, [R4,#(dword_1473C0 - 0x1473B8)]
.text:000BEF14                 CMP             R1, #0
.text:000BEF16                 BEQ             loc_BEF2E
# 若R1==0,也就是无缓存，跳到findClass
# 否则直接复用
.text:000BEF18 loc_BEF18                               ; CODE XREF: sub_BEEFC+48↓j
.text:000BEF18                 LDR             R3, =(dword_1473B8 - 0xBEF1E)
.text:000BEF1A                 ADD             R3, PC  ; dword_1473B8
.text:000BEF1C                 LDR             R0, [R3,#(env - 0x1473B8)]
.text:000BEF1E                 MOVS            R3, #0x1C4
.text:000BEF22                 LDR             R2, [R0]
.text:000BEF24                 LDR             R4, [R2,R3]
.text:000BEF26                 MOVS            R2, R6
.text:000BEF28                 MOVS            R3, R5
.text:000BEF2A                 BLX             R4
.text:000BEF2C
.text:000BEF2C locret_BEF2C                            ; CODE XREF: sub_BEEFC+46↓j
.text:000BEF2C                 POP             {R4-R6,PC}
.text:000BEF2E ; ---------------------------------------------------------------------------
# 下面这段是喜闻乐见的(*env)->findClass
.text:000BEF2E loc_BEF2E                               ; CODE XREF: sub_BEEFC+1A↑j
.text:000BEF2E                 LDR             R0, [R4,#(env - 0x1473B8)]
.text:000BEF30                 LDR             R1, =(aComCreativefac_4 - 0xBEF38) ; "com/creativefactory/SqliteManager"
.text:000BEF32                 LDR             R3, [R0]
.text:000BEF34                 ADD             R1, PC  ; "com/creativefactory/SqliteManager"
.text:000BEF36                 LDR             R3, [R3,#0x18]   # 虚表vtab取函数，基操
.text:000BEF38                 BLX             R3
# 此处截胡执行frida脚本，提升局部引用到全局
.text:000BEF3A                 MOVS            R1, R0
.text:000BEF3C                 STR             R0, [R4,#(dword_1473C0 - 0x1473B8)]
.text:000BEF3E                 MOVS            R0, #0
.text:000BEF40                 CMP             R1, #0
.text:000BEF42                 BEQ             locret_BEF2C

.text:000BEF44                 B               loc_BEF18
```

## 修复手段
工具这个就各显神通了，**frida so劫持/段扩容/段注入**都行。\
下面是第一部的frida脚本，第二部修复思路一致。\
可以通过frida-gadget情境执行，具体分为在线调试版和离线版。

在线版:
```js
var logicModule = Process.findModuleByName("libgame_logic.so");
if (logicModule) {
	var returnAddr = logicModule.base.add(0xBEF3A).or(1); 

	Interceptor.attach(returnAddr, {
		onEnter: function (args) {
			var localRef = this.context.r0;
			if (!localRef.isNull()) {
				try {
					var env = Java.vm.getEnv().handle;
					var newGlobalRefPtr = env.readPointer().add(84).readPointer();
					var NewGlobalRef = new NativeFunction(newGlobalRefPtr, 'pointer', ['pointer', 'pointer']);
					
					var globalRef = NewGlobalRef(env, localRef);
					this.context.r0 = globalRef;
					console.log("[+] xxyyzz: Upgrade at 0xBEF3A: " + globalRef);
				} catch (e) {}
			}
		}
	});
	console.log("Hook reference PASS");
}else{
	console.log("Hook too early, logic.so is not ready");
}
```
离线版（JVM->JNIEnv->GlobalRef）：
```js
console.log("=== Script Start ===");

var gJavaVM = null;
function findJavaVM() {
    var getVMs = Module.getGlobalExportByName("JNI_GetCreatedJavaVMs");
    if (!getVMs) {
        getVMs = Module.findExportByName("libart.so", "JNI_GetCreatedJavaVMs");
    }
    
    if (getVMs) {
        var func = new NativeFunction(getVMs, 'int', ['pointer', 'int', 'pointer']);
        var vmBuf = Memory.alloc(Process.pointerSize);
        var count = Memory.alloc(4);
        vmBuf.writePointer(ptr(0));
        count.writeInt(0);
        
        var result = func(vmBuf, 1, count);
        if (result === 0 && count.readInt() > 0) {
            return vmBuf.readPointer();
        }
    }
    return null;
}

function getJNIEnv(vm) {
    if (!vm) return null;
    
    var envBuf = Memory.alloc(Process.pointerSize);
    var vtable = vm.readPointer();
    
    // GetEnv
    var getEnvOffset = (Process.pointerSize === 8) ? 48 : 24;
    var getEnvPtr = vtable.add(getEnvOffset).readPointer();
    var GetEnv = new NativeFunction(getEnvPtr, 'int', ['pointer', 'pointer', 'int']);
    
    if (GetEnv(vm, envBuf, 0x00010006) === 0) {
        return envBuf.readPointer();
    }
    
    // AttachCurrentThread
    var attachOffset = (Process.pointerSize === 8) ? 32 : 16;
    var attachPtr = vtable.add(attachOffset).readPointer();
    var AttachCurrentThread = new NativeFunction(attachPtr, 'int', ['pointer', 'pointer', 'pointer']);
    
    var args = Memory.alloc(12);
    args.writeU32(0x00010006);
    args.add(4).writePointer(ptr(0));
    args.add(8).writePointer(ptr(0));
    
    if (AttachCurrentThread(vm, envBuf, args) === 0) {
        console.log("[+] Thread attached");
        return envBuf.readPointer();
    }
    
    return null;
}

// ==================== 等待库加载并 Hook ====================
var targetLib = "libgame_logic.so";
var targetOffset = 0xBEF3A;

function tryHook() {
    var module = Process.findModuleByName(targetLib);
    if (!module) {
        setTimeout(tryHook, 100);
        return;
    }
    
    console.log("[+] Found module: " + targetLib);
    
    // 获取 JVM
    if (!gJavaVM) {
        gJavaVM = findJavaVM();
        if (!gJavaVM) {
            console.log("[-] JavaVM not found, retry...");
            setTimeout(tryHook, 100);
            return;
        }
        console.log("[+] JavaVM: " + gJavaVM);
    }
    
    var hookAddr = module.base.add(targetOffset).or(1);
    console.log("[+] Hook address: " + hookAddr);
    
    Interceptor.attach(hookAddr, {
        onEnter: function(args) {
			console.log("called onEnter");
            var localRef = this.context.r0;
            if (localRef.isNull()){
				return;
			}
            
            var env = getJNIEnv(gJavaVM);
            if (!env) {
                console.log("[-] No JNIEnv");
                return;
            }
            
            // NewGlobalRef
            var jniTable = env.readPointer();
            var newGlobalRefOffset = (Process.pointerSize === 8) ? 108 : 84;
            var newGlobalRefPtr = jniTable.add(newGlobalRefOffset).readPointer();
            var NewGlobalRef = new NativeFunction(newGlobalRefPtr, 'pointer', ['pointer', 'pointer']);
            
            var globalRef = NewGlobalRef(env, localRef);
            if (globalRef && !globalRef.isNull()) {
                this.context.r0 = globalRef;
                console.log("[+] GlobalRef: " + globalRef);
            }
        }
    });
    console.log("[+] Hook installed!");
}
// 启动
tryHook();
```

## 后记
笔者的安卓5.1/10设备测试通过，运行正常；\
点击“开始游戏”后，不再出现闪退。