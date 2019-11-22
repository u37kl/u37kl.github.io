---
layout:     post
title:      "iOS-Runtime"
subtitle:   "Runtime－方法解释"
date:       2017-02-02 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
catalog: true
tags:
    - iOS-Runtime
---
[TOC]

# OC中的结构体

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

# 实验场景
```
-- Person定义

#import <Foundation/Foundation.h>
#import "Man.h"
@interface Person : NSObject <Man>

@property (nonatomic, copy) NSString *name;

- (void)eat;
@end

#import "Person.h"
#import "Error.h"
#import <objc/objc-runtime.h>
@implementation Person

- (void)eat{
    NSLog(@"%@", self);
    NSLog(@"Person");
}

- (NSString *)eat1:(NSString *)str{
    return str;
}

- (void)eat2:(NSString *)str{
    NSLog(@"test, %@, %@", str, [super class]);
}

void fooMethod(id obj, SEL _cmd)
{
    NSLog(@"Doing foo");
}



+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if(aSEL == @selector(runPerson)){
//        class_addMethod([self class], aSEL, (IMP)fooMethod, "v@");
        return NO;
    }
    return [super resolveInstanceMethod:aSEL];
}


- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    NSString *sel = NSStringFromSelector(aSelector);
    
    if ([sel isEqualToString:@"runPerson"]) {
        class_addMethod([[Error new] class], aSelector, (IMP)errorFun, "v@:@");
        NSMethodSignature *sig = [[Error new] methodSignatureForSelector:aSelector];
        return sig;
    }
    return [super methodSignatureForSelector:aSelector];
}

- (id)forwardingTargetForSelector:(SEL)aSelector
{
//    if (aSelector == @selector(runPerson)) {
//        Error *error = [[Error alloc] init];
//        class_addMethod([error class], aSelector, (IMP)errorFun, "v@");
//        return error;
//    }
    
    return nil;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    NSLog(@"%@", anInvocation);
        Error *error = [[Error alloc] init];
    if ([error respondsToSelector:anInvocation.selector]) {

        [anInvocation invokeWithTarget:error];
    }
}
@end

--Student类定义

#import "Person.h"
#import "Man.h"

// 定义结构体
struct stu{
    char * name;
    int arg;
};
@interface Student : Person

@property(nonatomic, strong) NSString *firstName;
- (void)study;

+ (void)study;

@end

#import "Student.h"

@implementation Student

// 返回结构体的方法
struct stu haha(){
    struct stu s = {"aaa", 12};
    return s;
}

- (struct stu) haha
{
    struct stu s = haha();
    NSLog(@"%s", s.name);
    return s;
}

- (void)eat{
    NSLog(@"Student");
    
    [super eat];
}

- (void)study
{
    NSLog(@"学习");
}

+ (void)study
{
    NSLog(@"学习");
}

@end

#import "Student.h"
@interface TestViewController ()

@end

```

# class_

## 获取类属性

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

## 类属性例子

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



## 获取实例对象的成员变量

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

## 成员变量例子

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

![](/img/iOS/2017-02-02-iOS-Runtime_2.png)




## 获取实例对象的成员属性


```


// 获取成员属性
objc_property_t class_getProperty(Class cls, const char *name)

// 替换成员属性
void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)

// 添加成员属性
BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount) －－－ 有待测试，注意objc_property_attribute_t中顺序


```

## 成员属性例子

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

## 获取方法

```
// 向类对象、元类对象的方法列表中添加方法(分别添加的是对象方法、类方法)，如果添加了已存在的SEL，添加失败
BOOL class_addMethod(Class cls, SEL name,  IMP imp, const char *types)
// 获取实例对象的实例方法结构体
Method class_getInstanceMethod(Class aClass, SEL aSelector);
// 获取类对象的类方法的结构体
Method class_getClassMethod(Class aClass, SEL aSelector);
// 更换对象方法、类方法的函数指针
IMP  class_replaceMethod(Class cls, SEL name, IMP  imp, const char *types)
// 获取对象方法、类方法的函数指针
IMP  class_getMethodImplementation(Class cls, SEL name)
// 获取类对象、元类对象中的方法列表
Method * class_copyMethodList(Class cls, unsigned int *outCount);
// 替换替换类中方法的函数指针，返回原来方法的函数指针
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types) 

```
## 获取方法例子

```


@implementation TestViewController

- (void)methodTest
{

    // 获取类方法
    Method class_M = class_getClassMethod([Student class], @selector(study));
    
    // 向Student类实例添加对象方法
    class_addMethod([Student class], @selector(testMethod:), (IMP)testFunc, "v@:i");
    // 获取类实例的对象方法
    Method instance_M = class_getInstanceMethod([Student class], @selector(testMethod:));
    Student *stu = [[Student alloc] init];
    // 调用实例对象的方法
    ((void (*)(id, Method, int))method_invoke)(stu, instance_M, 1);
    
    // 获取保存在类对象中的对象方法列表
    unsigned int outCount = 0;
    Method *method = class_copyMethodList([Student class], &outCount);
    
    for (int i = 0; i<outCount; i++) {
        // 获取方法SEL
        SEL method_name_sel = method_getName(method[i]);
        const char * method_name = sel_getName(method_name_sel);
        // 获取方法的返回类型 + 参数类型
        const char * mehtod_type= method_getTypeEncoding(method[i]);
        // 获取方法的返回类型
        const char * method_returnType = method_copyReturnType(method[i]);
        // 获取方法对应的函数指针
        IMP imp = method_getImplementation(method[i]);
        // 方法的参数个数
        int method_arg = method_getNumberOfArguments(method[i]);
        // 返回方法中的第n个参会
        char *argType = method_copyArgumentType(method[i], 0);
        
        NSLog(@"SEL = %s, methodName = %s, methodType = %s, methodRetype = %s, args = %i", sel_getName(method_name_sel), method_name, mehtod_type, method_returnType, method_arg);
    }
    
    IMP imp = class_getMethodImplementation([Student class], @selector(studyMath));
    // 返回Student类中的haha方法的函数指针，haha方法的返回值为结构体，所以使用该方法。
    IMP imp1 = class_getMethodImplementation_stret([Student class], @selector(haha));
    // 使用class_getMethodImplementation方法返回haha()，程序直接crash。
    struct stu s1 = ((struct stu(*)())imp)();
    // 使用class_getMethodImplementation_stret方法返回haha()，该方法就是为了哪些返回结构体方法准备的
    struct stu s = ((struct stu(*)())imp1)();
    NSLog(@"%s   %i", s.name, s.arg);
    
    Student *stu1 = [Student new];
    // 替换student类的study方法的函数指针。
    IMP fun = class_replaceMethod([stu class], @selector(study), (IMP)run, "v@");
    [stu study];
    fun();
}


void run()
{
    NSLog(@"run");
}
@end

```

## 获取协议

```
／／ 添加向实例对象、类对象、元类对象添加协议
 BOOL class_addProtocol(Class cls, Protocol *protocol) 
 Protocol * __unsafe_unretained *class_copyProtocolList(Class cls, unsigned int *outCount)
```

## 获取协议例子

```

- (void)protocolTest
{
    Class stu = object_getClass([Student new]);
    Class oldStu = object_getClass(stu);
    NSLog(@"%p   ,  %p", stu, oldStu);
    
    // 向Student类添加协议
    BOOL isSuccess = class_addProtocol(oldStu, @protocol(Man));
    //  判断Student类是否遵循是否协议
    BOOL isGet = class_conformsToProtocol([Student class], @protocol(Man));
    
    // 根据字符串获取协议对象
    Protocol *protocols = objc_getProtocol("Man");
    
    // 动态创建协议定义
    Protocol *newProtocol = objc_allocateProtocol("Animal");
    // 添加协议中方法声明
    protocol_addMethodDescription(newProtocol, @selector(doSomthing), "v@:", NO, YES);
    // 添加该协议遵循的父协议。
    protocol_addProtocol(newProtocol, protocols);
    // 注册创建的协议定义
    objc_registerProtocol(newProtocol);
    
    unsigned int outCount = 0;
    class_addProtocol([Student class], newProtocol);
    // 遍历Student类中遵循的协议列表
    Protocol * __unsafe_unretained * protocol = class_copyProtocolList([Student class], &outCount);
    
    for (int i = 0; i<outCount; i++) {
        // 获取协议名称
        const char *name = protocol_getName(protocol[i]);
        NSLog(@"%s", name);
    }
}

```
# object_

## object_方法介绍

```
// 设置成员变量的值
void object_setIvar(id object, Ivar ivar, id value)
// 获取成员变量的值
id object_getIvar(id object, Ivar ivar)
// 获取类名称
const char *object_getClassName(id obj)
// 获取实例对象的类对象、获取类对象的元类对象
Class object_getClass(id object)
// 判断该对象是否不是实例对象
BOOL object_isClass(id obj)
```

## 例子

```
    Student *stu = [Student new];
    // 获取对象的类对象
    Class clazz = object_getClass(stu);
    // 获取类对象的元类对象
    Class metaClazz = object_getClass(clazz);
    // 获取对象的类名称
    const char *className = object_getClassName(stu);
    // 该对象是否为类对象
    BOOL isClass = object_isClass(stu);
    BOOL isClass1 = object_isClass(clazz);
    BOOL isClass2 = object_isClass(metaClazz);
    // 获取Student实例对象的成员变量firstName
    Ivar ivar = class_getInstanceVariable([Student class], "_firstName");
    // 设置Student实例对象的成员变量firstName的值
    object_setIvar(stu, ivar, @"zp3sss");
    // 获取成员变量的值
    NSString *fristName = object_getIvar(stu, ivar);

```

![](/img/iOS/2017-02-02-iOS-Runtime_3.png)


# objc_

## objc_方法介绍

```
// 根据字符串获取类对象，如果没有找到，则objc_getClass调用类处理程序回调
id objc_getClass(const char *name) 
// 根据字符串获取类对象，如果没有找到，则objc_lookUpClass不会调用类处理程序回调
id objc_lookUpClass(const char *name)
// 根据字符串获取类对象，如果没有找到，则程序崩溃，该函数认为编译时没有ZeroLink链接错误
id objc_getRequiredClass(const char *name)
// 根据字符串返回类对象的元类对象
id objc_getMetaClass(const char *name)
// 根据字符串获取协议对象
Protocol *objc_getProtocol(const char *name)

//  添加关联对象，如果value置为nil，将会删除关联对象，所有该方法即是添加，也是删除
void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)
// 获取关联对象
id objc_getAssociatedObject(id object, void *key)
// 移除该对象所有关联对象
void objc_removeAssociatedObjects(id object)

// runtime发送消息，调用返回基本类型、对象类型的函数(返回结构体试了试也行)，使用该函数时，要强转它的函数参数类型
id objc_msgSend(id self, SEL op, ...)
// runtime发送消息，调用返回结构体的函数
void objc_msgSend_stret(void * stretAddr, id theReceiver, SEL theSelector, ...)
// 调用父类的方法，前提也是强转它的函数参数类型
id objc_msgSendSuper(struct objc_super *super, SEL op, ...)
void objc_msgSendSuper_stret(struct objc_super *super, SEL op, ...)
// 创建一个协议
Protocol *objc_allocateProtocol(const char *name)
// 将协议注册进入项目中
void objc_registerProtocol(Protocol *proto)
// 创建一个类定义,最后一个参数为设置一个额外空间，保存成员变量，如果不设置传入0
objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes)
// 将类定义注册进入项目中
void objc_registerClassPair(Class cls)
// 从项目中删除类定义
void objc_disposeClassPair(Class cls)

```

##例子

```
- (void)objcRuntime
{
    id className = objc_getClass("Student");
    id className1 = object_getClass(className); // 返回Student类的元类对象
    id classNameStr = objc_lookUpClass("Student");
    id requiredClass = objc_getRequiredClass("Student");
    id metaClass = objc_getMetaClass("Student");
    Protocol *protocol = objc_getProtocol("Man");
    Student *stu = [Student new];
    // 将关联对象绑定在stu对象上。
    objc_setAssociatedObject(stu, @selector(className), @"className", OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    // 获取stu对象上的关联对象
    objc_getAssociatedObject(stu, @selector(className));
    // 删除stu的关联对象
    objc_setAssociatedObject(stu, @selector(className), nil, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    // 生成objc_super结构体，作为参数传入objc_msgSendSuper函数中
    struct objc_super su = {(id)stu1, class_getSuperclass([Student class])};
    // 调用父类的eat方法
    ((void(*)(struct objc_super * , SEL))objc_msgSendSuper)(&su, @selector(eat));
        


    // 调用返回结构体的函数，使用objc_msgSend()也行。使用 "@@:i" 替换 "{wonman=*i}@:i"也可以。
    class_addMethod([self class], @selector(testFunc1:), (IMP)testFunc1, "{wonman=*i}@:i");
    struct WonMan won1 = ((struct WonMan  (*)(id, IMP, int))objc_msgSend)(self, (IMP)testFunc1, 12);
    
    －－调用返回结构体的方法
    struct WonMan wonman;
    class_addMethod([self class], @selector(testFunc2:), (IMP)testFunc2, "v{wonman=*i}@:i");
    // 传入结构体指针，为结构体赋值。
    ((void(*)(struct WonMan *, id, SEL, int))objc_msgSend_stret)(&wonman, self, @selector(testFunc2:), 13);
    
    －－ 调用返回OC对象的方法
    class_addMethod([self class], @selector(testFunc3:), (IMP)testFunc3, "@@:i");
    Student *stu2 = ((Student * (*)(id, SEL, int))objc_msgSend)(self, @selector(testFunc3:), 12);
    
    －－ 调用父类的方法
    struct objc_super su = {(id)stu2, class_getSuperclass([Student class])};
    ((void(*)(struct objc_super * , SEL))objc_msgSendSuper)(&su, @selector(eat));
    
    
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
    
}

// C函数定义  

void testFunc(id obj, SEL s, int a){
    NSLog(@"%@___%i", @"测试", a);
}

struct WonMan testFunc1(id obj, SEL s, int a){
    
    struct WonMan wonman = {
        "haha",
        a
    };
    
    return wonman;
}

void testFunc2(struct WonMan * wonman, id obj, SEL s, int a){
    
    wonman->name = "wonman1";
    wonman->age = a;
}

Student * testFunc3(id obj, SEL s, int a){
    
    Student *stu = [[Student alloc] init];
    stu.firstName = @"Student对象";
    
    return stu;
}

```
![](/img/iOS/2017-02-02-iOS-Runtime_4.png)








# note:

```
objc_与object_区别，含object_前缀的方法涉及到OC对象，含objc_前缀的方法不会涉及到OC对象，即通过字符串获取OC对象。但是也有特殊的函数涉及到OC对象。

给实例对象添加方法、成员变量、成员属性、其实就是向class结构体中添加东西，因为它们的定义都保存在class结构体中


```

# 问题：
1. 不知道objc_property_t这个参数有啥用？






