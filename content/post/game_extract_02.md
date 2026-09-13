---
title: "[JNI查错] 口袋侦探闪退修复（一） - 安卓8及以上"
date: 2026-05-11T13:45:31+08:00
categories: ["逆向"]
tags: ["安卓", "JNI", "口袋侦探", "逆向"]
draft: false
---

这个古文物游戏，由于dvm转art虚拟机的缘故，只能用旧版本安卓/VMOS虚拟机玩已经是老生常谈了，\
至于IAP的处理思路也被前人写烂了。我来写写闪退修复吧。\
放一下问题表现、分析过程、修复手段，仅此而已。\
**本质原因：安卓5以前dvm虚拟机检查不严格，厂家存在池化滥用JNI LocalRef（局部引用）的黑魔法编程**

## 核心观点
1. 安卓8以上的思路（**测试可行**）\
局部引用表的上限提升到了8388608个引用, 可以使用本文的暴力手段。

2. 安卓5-7的思路（有点麻烦）\
作为早期ART限制REF总数<=512，不适用本文的暴力手段。\
需要考虑配套deleteLocalRef，或者是提升局部引用到globalRef，\
工具这个就各显神通了，**frida so劫持/段扩容/段注入/汇编覆盖**都行。

## 问题表现
点击开始/章节选择后闪退 \
logcat日志：
```sh
第一部：
F creativefactor: indirect_reference_table.cc:60] JNI ERROR (app bug): accessed deleted Local 0x21 
第二部：
F DEBUG   : Abort message: 'JNI ERROR (app bug): accessed stale Local 0x29  (index 2 in a table of size 1)'
```


## 分析过程
### 系统侧日志意义不大
（一、二两部的BUG是同类型，这里只放第1部的）\
主要提示就是“**accessed deleted Local**”。
```bash
 F creativefactor: indirect_reference_table.cc:60] JNI ERROR (app bug): accessed deleted Local 0x21
 F creativefactor: runtime.cc:655] Runtime aborting...
 F creativefactor: runtime.cc:655] All threads:
 F creativefactor: runtime.cc:655] DALVIK THREADS (22):
 F creativefactor: runtime.cc:655] "GLThread 134" prio=5 tid=18 Runnable
 F creativefactor: runtime.cc:655]   native: #06 pc 0040b6bb  /apex/com.android.art/lib/libart.so (art::Runtime::Abort(char const*)+1510)
 F creativefactor: runtime.cc:655]   native: #07 pc 0000d9a5  /system/lib/libbase.so (android::base::SetAborter(std::__1::function<void (char const*)>&&)::$_3::__invoke(char const*)+48)
 F creativefactor: runtime.cc:655]   native: #08 pc 0000d2b1  /system/lib/libbase.so (android::base::LogMessage::~LogMessage()+224)
 F creativefactor: runtime.cc:655]   native: #09 pc 00228fa7  /apex/com.android.art/lib/libart.so (art::IndirectReferenceTable::AbortIfNoCheckJNI(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)+150)
 F creativefactor: runtime.cc:655]   native: #10 pc 0029c779  /apex/com.android.art/lib/libart.so (art::IndirectReferenceTable::GetChecked(void*) const+292)
 F creativefactor: runtime.cc:655]   native: #11 pc 004532c5  /apex/com.android.art/lib/libart.so (art::Thread::DecodeJObject(_jobject*) const+52)
 F creativefactor: runtime.cc:655]   native: #12 pc 002a0589  /apex/com.android.art/lib/libart.so (art::FindMethodJNI(art::ScopedObjectAccess const&, _jclass*, char const*, char const*, bool)+32)
 F creativefactor: runtime.cc:655]   native: #13 pc 002c9937  /apex/com.android.art/lib/libart.so (art::JNI<false>::GetStaticMethodID(_JNIEnv*, _jclass*, char const*, char const*)+546)
 F creativefactor: runtime.cc:655]   native: #14 pc 000bdf2b ==/lib/arm/libgame_logic.so (???)
```

### 应用侧的末端逻辑

我们以SqliteManagerJni::openCommonDBJni()为切入点
```bash
 F creativefactor: runtime.cc:655]   native: #15 pc 000be253 ==/lib/arm/libgame_logic.so (SqliteManagerJni::openCommonDBJni()+10)
 F creativefactor: runtime.cc:655]   native: #16 pc 000bd841 ==/lib/arm/libgame_logic.so (SqliteManager::_openDatabase(int)+20)
 F creativefactor: runtime.cc:655]   native: #17 pc 000bd90b ==/lib/arm/libgame_logic.so (SqliteManager::openCommonDB()+26)
 F creativefactor: runtime.cc:655]   native: #18 pc 0009aaed ==/lib/arm/libgame_logic.so (EpisodeSelectLayer::_loadEpisodeSelectDisplay()+4276)
 F creativefactor: runtime.cc:655]   native: #19 pc 0009baab ==/lib/arm/libgame_logic.so (EpisodeSelectLayer::openMenu(cocos2d::SelectorProtocol*, void (cocos2d::SelectorProtocol::*)())+34)
 F creativefactor: runtime.cc:655]   native: #20 pc 000a2993 ==/lib/arm/libgame_logic.so (StandardMenuLayer::openMenu()+22)
 F creativefactor: runtime.cc:655]   native: #21 pc 00076699 ==/lib/arm/libgame_logic.so (MainMenuScene::replaceMenu(int)+60)
 F creativefactor: runtime.cc:655]   native: #22 pc 000769ef ==/lib/arm/libgame_logic.so (MainMenuLayer::_completeCloseLayer()+54)
```

## 修复手段

我们就去libgame_logic.so里会一会这个函数，显然sub_BDEFC是个JNI引用池化的查找函数：\
我们需要（无论池化缓存与否，强行触发jni findXXX查找获取新引用）	
```c
bool __fastcall SqliteManagerJni::openCommonDBJni(SqliteManagerJni *this)
{
  int v1; // r2
  _BOOL4 result; // r0

  v1 = sub_BDEFC("openCommonDB", "()Z");
  result = 0;
  if ( v1 )
    return _JNIEnv::CallStaticBooleanMethod(dword_1463BC, dword_1463C0) != 0;
  return result;
}

// JNI修复后，导入Jni.h的流程不做赘述
jmethodID __fastcall sub_BEEFC(const char *a1, const char *a2)
{
  jclass v4; // r1
  jmethodID result; // r0

  (*(void (__fastcall **)(int, JNIEnv **, _DWORD))(*(_DWORD *)dword_1473B8 + 16))(dword_1473B8, &env, 0);
  v4 = (jclass)dword_1473C0;

  /* 
    判断是否存在jclass缓存，存在就复用。
    拿掉这个判断即可，将BEQ/BNE判断改成强制B跳转。
  */
  if ( dword_1473C0 )
    return (*env)->GetStaticMethodID(env, v4, a1, a2);
  
  v4 = (*env)->FindClass(env, "com/creativefactory/SqliteManager");
  dword_1473C0 = (int)v4;
  result = 0;
  if ( v4 )
    return (*env)->GetStaticMethodID(env, v4, a1, a2);
  return result;
}
```

## 后记
本文的方法存在长时间游玩闪退的风险，毕竟8388608也不是无穷大；\
建议勤存档，至少本文的手段可以让游戏正常游玩，完美修改得等下一篇文章。