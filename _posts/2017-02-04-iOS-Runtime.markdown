---
layout:     post
title:      "iOS-Runtime"
subtitle:   "Runtime－消息发送、消息转发"
date:       2017-02-04 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
catalog: true
tags:
    - iOS-Runtime
---

# OC RunTime消息转发机制

1. Method resolution -- 方法补救
2. Fast forwarding -- 快速转发
3. Normal forwarding -- 真正的转发


```
OC的动态语言特性通过发送消息来实现的，如果调用对象的方法时，会向该对象发送消息。对象收到消息寻找方法步骤：
1、通过实例对象的isa指针，找到类对象，获取到类对象中的方法列表缓存，查询是否存在。
2、获取类对象的方法列表，查询是否存在。
3、获取父类的类对象的方法列表缓存，查询是否存在。
4、查询父类的类对象的方法列表，查询是否存在。
5、上述方法都查询不到时，开启转发三部曲：
    1)实现resolveInstanceMethod()方法，在该方法中为不存在的方法添加实现。
    2)实现forwardingTargetForSelector()方法，该方法将消息转发到其他对象中，让其他对象处理。
    3)实现methodSignatureForSelector()和forwardInvocation()，做一次标准的消息转发。
```

## 代码

```
-- Error类
#import <Foundation/Foundation.h>

@interface Error : NSObject

void errorFun();
@end

#import "Error.h"

@implementation Error


void errorFun(){
    NSLog(@"找不到该方法");
}

@end

-- Person类
#import "Person.h"
#import "Error.h"
#import <objc/objc-runtime.h>
@implementation Person

void fooMethod(id obj, SEL _cmd)
{
    NSLog(@"Doing foo");
}


// 手动向该类添加一个方法名为 "runPerson" 实例方法
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if(aSEL == @selector(runPerson)){
        class_addMethod([self class], aSEL, (IMP)fooMethod, "v@");
        return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}

// 将消息转发给其他类处理，返回该类的实例对象
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if (aSelector == @selector(runPerson)) {
        Error *error = [[Error alloc] init];
        class_addMethod([error class], aSelector, (IMP)errorFun, "v@");
        return error;
    }
    
    return nil;
}

// 生成一个NSMethodSignature对象，让forwardInvocation()方法处理
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

// 将消息转发给其他类的实例对象
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    NSLog(@"%@", anInvocation);
        Error *error = [[Error alloc] init];
    if ([error respondsToSelector:anInvocation.selector]) {

        [anInvocation invokeWithTarget:error];
    }
}

```



