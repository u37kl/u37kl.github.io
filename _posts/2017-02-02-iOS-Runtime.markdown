---
layout:     post
title:      "iOS-Runtime"
subtitle:   "Runtime－方法解释"
date:       2017-02-02 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
tags:
    - iOS-Runtime
---

## OC中的结构体

**Example:**

```
// 父类结构体
struct objc_super {

    __unsafe_unretained id receiver;   // 当前对象


#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;  // 当前对象的父类的类对象
#endif
    /* super_class is the first class to search */
};

// 类对象
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;                       // 类对象指针

#if !__OBJC2__
    Class super_class                                       // 父类对象指针;
    const char *name                                        // 当前类名称;
    long version                                            // 当前类版本;
    long info                                               // 类对象指针;
    long instance_size                                      // 类对象指针;
    struct objc_ivar_list *ivars                            // 实例对象的成员变量列表;
    struct objc_method_list **methodLists                   // 实例对象的方法列表;
    struct objc_cache *cache                                // 实例对象的方法缓存;
    struct objc_protocol_list *protocols                    // 实例对象所实现的协议列表;
#endif

} OBJC2_UNAVAILABLE;

// 实例对象
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

// 实例对象的分类结构体
struct objc_category {
    char *category_name                                      // 分类名称;
    char *class_name                                         // 类名称;
    struct objc_method_list *instance_methods                // 对象方法列表;
    struct objc_method_list *class_methods                   // 类方法类表;
    struct objc_protocol_list *protocols                     // 遵循协议列表;
}                                                            OBJC2_UNAVAILABLE;

// 对象的成员变量
struct objc_ivar {
    char *ivar_name                                         //变量的名称;
    char *ivar_type                                         //变量的类型;
    int ivar_offset                                         // 实例对;
#ifdef __LP64__
    int space                                               // 实例对象的成员变量列;
#endif
}                                                            OBJC2_UNAVAILABLE;

struct objc_ivar_list {
    int ivar_count                                           OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

// 对象方法变量
struct objc_method {
    SEL method_name                                          // SEL方法名;
    char *method_types                                       // 方法类型;
    IMP method_imp                                           // 函数指针;
}                                                            OBJC2_UNAVAILABLE;

struct objc_method_list {
    struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;

    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
} 
```

## RunTime类对象方法

**Example:**

```


```



