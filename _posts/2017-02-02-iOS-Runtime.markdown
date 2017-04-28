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

## class_

**获取类属性**

```
// 获取类属性

const char * class_getName(Class cls); －－ 获取类的名称
Class class_getSuperclass(Class cls);  －－ 获取类的父类
size_t class_getInstanceSize(Class cls) －－ 获取该类实例对象的字节数
int class_getVersion(Class theClass)  －－ 获取该类的版本号

// 类中方法、协议存在性判断
BOOL class_respondsToSelector(Class cls, SEL sel) －－ 判断该类是否实现该方法
BOOL class_conformsToProtocol(Class cls, Protocol *protocol)－－判断该类是否遵循某个协议

```

**例子**

```
类：@interface TestViewController : UIViewController

- (void)testAFN{
    
    const char *className = class_getName([self class]);
    int classVersion = class_getVersion([self class]);
    Class superClazz = class_getSuperclass([self class]);
    size_t size = class_getInstanceSize([self class]);
    
    // 判断实例对象是否有run方法
    BOOL isInstanceExist = class_respondsToSelector([self class], @selector(run));
    
    // 判断类对象是否有run方法
    BOOL isClassExit = class_respondsToSelector(object_getClass(object_getClass(self)), @selector(run));
}
- (void)run
{

}

```
![](/img/iOS/2017-02-02-iOS-Runtime_1.png)



**获取实例对象的成员变量**

```
// 获取实例对象的成员变量
Ivar class_getInstanceVariable(Class cls, const char* name) 
// 获取类对象的成员变量
Ivar class_getClassVariable(Class cls, const char* name)
// 给实例对象添加成员变量
BOOL class_addIvar(Class cls, const char *name, size_t size, uint8_t alignment, const char *types)
// 获取类对象中的成员变量列表
Ivar * class_copyIvarList(Class cls, unsigned int *outCount)

note：

class_addIvar()函数只能在用户自定义创建类定义时才能使用，不能在使用实例对象时，动态添加成员变量，因为实例对象的成员变量列表保存在类对象中，这样会造成之前创建的实例对象没有刚添加的成员变量，由此导致之前创建的类废了。

```

**例子**

```
- (void)testAFN{
    NSLog(@"%s", class_getName([self class]));
    // 运行时创建类
    Class clazz = objc_allocateClassPair([Person class], "WoMan", 0);
    
    // 添加实例对象的成员变量，指针类型使用log2()函数包裹
    class_addIvar(clazz, "firstName", class_getInstanceSize([NSString class]), log2(class_getInstanceSize([NSString class])), @encode(NSString *));
    class_addIvar(clazz, "secondName", class_getInstanceSize([NSString class]), log2(class_getInstanceSize([NSString class])), @encode(NSString *));
    class_addIvar(clazz, "sex", sizeof(BOOL), sizeof(BOOL), @encode(BOOL));
    class_addIvar(clazz, "grade", sizeof(NSInteger), sizeof(NSInteger), @encode(NSInteger));
    class_addIvar(clazz, "student", class_getInstanceSize([Student class]), log2(class_getInstanceSize([Student class])), @encode(Student *));
    
    // 将类注册进去
    objc_registerClassPair(clazz);
    // 创建自定义类的实例对象
    id s = [[clazz alloc] init];
    // 根据类对象和成员变量名称获取成员变量定义
    Ivar var0 = class_getInstanceVariable(clazz, "firstName");
    // 为实例变量设置值
    object_setIvar(s, var0, @"peter");
    
    
    Ivar var1 = class_getInstanceVariable(clazz, "secondName");
    object_setIvar(s, var1, @"zpsss");
    
    
    Ivar var2 = class_getInstanceVariable(clazz, "sex");
    object_setIvar(s, var2, @(1));
    
    
    Ivar var3 = class_getInstanceVariable(clazz, "grade");
    object_setIvar(s, var3, @(92));
    
    
    Student *stu = [Student new];
    stu.firstName = @"mary";
    
    Ivar var4 = class_getInstanceVariable(clazz, "student");
    object_setIvar(s, var4, stu);
    
    // 获取实例对象的某成员变量的值
    Student *temp = object_getIvar(s, var4);
    NSLog(@"%@", temp.firstName);
    
    unsigned int outCount = 0;
    // 获取类对象中的成员变量列表
    Ivar *var = class_copyIvarList(clazz, &outCount);
    
    for (int i=0; i< outCount; i++) {
        // 获取成员变量名吃
        const char *ivarName = ivar_getName(var[i]);
        // 获取成员变量的类型
        const char *ivarType = ivar_getTypeEncoding(var[i]);
        NSLog(@"%s,   %s", ivarName, ivarType);
    }
}

```

![](/img/iOS/2017-02-02-iOS-Runtime_1.png)




**获取实例对象的成员属性**


```


// 获取成员属性
objc_property_t class_getProperty(Class cls, const char *name)

// 替换成员属性
void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)

// 添加成员属性
BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount) －－－ 有待测试，注意objc_property_attribute_t中顺序


```

**例子**

```

 objc_property_attribute_t type = {"T", "@\"NSString\""};
    objc_property_attribute_t ownerShip = {"&", ""};
    objc_property_attribute_t nonatomic = {"N", ""};
    objc_property_attribute_t backingIvar = {"V", "_myName"};
    objc_property_attribute_t attrs[] = {type, ownerShip, nonatomic, backingIvar};
    
    class_addProperty([Student class], "myName", attrs, 4);
    
    
    objc_property_attribute_t type1 = {"T", "@\"BOOL\""};
    objc_property_attribute_t ownerShip1 = {"", ""};
    objc_property_attribute_t nonatomic1 = {"N", ""};
    objc_property_attribute_t backingIvar1 = {"V", "_sex"};
    objc_property_attribute_t attrs1[] = {type1, ownerShip1, nonatomic1, backingIvar1};
    
    class_addProperty([Student class], "sex", attrs1, 4);
    
    
    objc_property_t myName = class_getProperty([Student class], "myName");
    
    unsigned int outCount = 0;
    
    objc_property_t *property = class_copyPropertyList([Student class], &outCount);
    
    for (int i = 0; i< outCount; i++) {
        
        NSLog(@"属性名：%s", property_getName(property[i]));
        NSLog(@"属性类型：%s", property_getAttributes(property[i]));
    }

note:
    T : 成员属性类型
    C，W，空，& ＝＝》 copy，weak，assign，strong
    N，空 ＝＝》 nonatomic， atomic
    V ＝＝》 变量的名称
    objc_property_attribute_t变量保存的就是 " @property (nonatomic, weak) UIButton *btn " 声明。
    

```

**获取方法**

```
BOOL class_addMethod(Class cls, SEL name,  IMP imp, const char *types)

Method class_getInstanceMethod(Class aClass, SEL aSelector);

Method class_getClassMethod(Class aClass, SEL aSelector);

IMP  class_replaceMethod(Class cls, SEL name, IMP  imp, const char *types) —— 替换类中的方法，返回被替换的方法的函数指针

IMP  class_getMethodImplementation(Class cls, SEL name)

Method * class_copyMethodList(Class cls, unsigned int *outCount);

```
**例子**

```


```

```
给实例对象添加方法、成员变量、成员属性、其实就是向class结构体中添加东西，因为它们的定义都保存在class结构体中

```

## 问题：
1. 不知道这个参数有啥用？






