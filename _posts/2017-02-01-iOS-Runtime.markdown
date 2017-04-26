---
layout:     post
title:      "iOS-Runtime"
subtitle:   "OC中实例对象、类对象、元对象"
date:       2017-02-01 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
tags:
    - iOS-Runtime
---

## OC对象的结构

![](/img/iOS/RunTime_1.png)

**上图讲解(虚线: isa，实线: superClass)**

```
上图中有三种类型的对象，实例对象、类对象、元对象。
1、实例对象：
   它是我们编程常用的东西，其中保存着程序中产生的数据，在内存中可以存在多个，在OC中实例对象使用结构体表示。
它是由类对象创建出来的，类对象中保存了实例对象的定义，包括成员变量、实例方法、遵循的协议。

2、类对象：
   它是我们编程时用来创造实例对象的东西，相当于现实生活中的设计图纸，其中保存着构造实例对象的成员变量定义
(类型、名称)，实例方法定义、遵循的协议、谁的子类。类对象与实例对象不同的是类方法创建的实际为程序运行时由系
统来生成，并且在内存中只有一份，而实例对象是由程序员创建，并且我们可以控制实例对象的生命周期。

3、元类对象：
    它是我们编程时极少接触的东西，有可能都不知道它的存在，它是用来创造类对象的东西，其中保存着构造类对象的类方法的定义、类方法的父类定义。
    
note：
    1>类对象和元类对象中有isa指针(指向元类)，super指针(指向父类类对象)。
    2>元类对象的isa指针指向根元类，即由根元类创建的元类。根元类的isa指针指向自己。


```

## super解释

```
    super是编译器字段，编译器会自动将其调用父类的方法。super关键字对应C结构体为objc_super，当使用super时，
会转成objc_super结构体，并调用objc_msgSendSuper方法。

    [self class]与[super class]都返回self对应的类；objc_super结构体中的两个参数，self，父类的class，
class方法应该做个特殊处理，[super class]调用的还是[self class]。

```

## 问题

1. super调用方法时，它是如何调用的方法，为何[super class]返回的不是父类，[super init]返回的时父类方法。super调用细节？

## 解答：

```
补充：OC方法中有两个隐藏的参数，self和SEL，即调用该方法的对象和该方法的SEL。

父类：class_Person
子类：class_child
父类对象： instance_Person
子类对象：instance_child
[instance_child study];
[super study];
    super关键字经过编译器，调用objc_msgSendSuper()方法，第一个参数为objc_super(instance_child, class_Person)，
第二个参数为@selector(study)；调用父类的study方法，方法中的隐形参数self＝instance_child。

[instance_child class] == [super class]原因：
    因为class定义在它们共同的父类NSObject中，因此都是调用的NSObject类中的class方法，但是方法的隐形参数
self = instance_child;因为objc_super(instance_child, class_Person)中的第一个参数为instance_child。
```

## 感悟：

```
    OC中的对象其实就是某一类方法库的地址，通过对象中的isa指针找到藏在类对象或者元类对象中的方法列表，方法中
的第一个隐藏参数的值为发起消息的对象。

举个例子：

父类对象：instance_Person，父类的方法eat()。
子类对象：instance_Student。

[instance_Student eat]; instance_Student对象发送了一个eat消息，经过runtime解析后的结果为：
C函数调用：eat(instance_Student, @selector(eat)); 虽然调用的是父类的方法，但是方法的第一个隐藏参数还是
子类对象。

```









