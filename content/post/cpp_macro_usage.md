---
title: "[C/CPP] 宏技巧收集"
date: 2026-04-25T05:07:25+08:00
draft: false
---

# 防止重定义
```c
// 以本文件定义为准，解除CDBG防止编译器重复定义报错
#undef CDBG
#ifdef ENABLE_XXX
	#define CDBG printk
#else
	#define CDBG
#endif /* ENABLE_XXX */
```

# 宏重言（智能枚举）
```c
#ifdef SPECIAL_ENUM_META

// auxiliary macros for CONCAT
#define _CONCAT(x,y) x##y
#define CONCAT(x,y)	_CONCAT(x,y)

#define LABEL_CONNAME   CONCAT(label_,SPECIAL_ENUM_NAME)
#define ENUM_CONNAME   CONCAT(enum_,SPECIAL_ENUM_NAME)
// ENUM_TO
#define ENUM_TO_STR(x)  LABEL_CONNAME[x]

// create info
#ifdef MAKE_INSTANCE    // const char & 
const char* LABEL_CONNAME[] = {
    #define SPECIAL_ENUM(x) #x
    SPECIAL_ENUM_META
    #undef SPECIAL_ENUM
};
#else
extern const char* LABEL_CONNAME[];
#endif

// create enum, AS IS mode
enum ENUM_CONNAME {
    #define SPECIAL_ENUM(x) x
    SPECIAL_ENUM_META
    #undef SPECIAL_ENUM
};


#undef SPECIAL_ENUM_META
#endif
```

