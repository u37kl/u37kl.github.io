---
layout:     post
title:      "iOS-Runtime"
subtitle:   "Runtime－关联对象原理"
date:       2017-02-05 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
tags:
    - iOS-Runtime
---

# OC RunTime关联对象
## 核心方法

1. void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)
2. id objc_getAssociatedObject(id object, void *key)
3. void objc_removeAssociatedObjects(id object)

## 方法介绍
```
objc_setAssociatedObject：
    1. 该方法用来，向实例对象中添加关联属性，和删除实例对象中的某个关联属性
    2. 参数介绍：
        1) object: 实例对象
        2) key:    关联属性所需的key，用key来找到实例对象下的对应关联属性
        3) value:  关联属性
        3) policy: 关联策略
    3. 一般key使用getter方法的SEL指针标示，不需要额外再定义指针了。
        

    
objc_removeAssociatedObjects:
    1. 删除实例对象中所有的关联属性。

```
## 关联策略

```
    typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
        OBJC_ASSOCIATION_ASSIGN = 0,  // (nonatomic, assign)
        OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, // (nonatomic, retain)
        OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  // (nonatomic, copy)     
        OBJC_ASSOCIATION_RETAIN = 01401,   // (atomic, retain)     
        OBJC_ASSOCIATION_COPY = 01403     // (atomic, copy) 
    };
    
    note:
        1. 使用 "OBJC_ASSOCIATION_ASSIGN" 方式的关联策略必须注意，当关联属性被释放时，指针不会被置为
           0x00，这样就会出现野指针现象。通过objc_getAssociatedObject()方法获取指向关联属性的指针时，
           程序必崩溃。

```

## 原理:

```
关联属性存储结构:
    1、AssociationsManager管理对象中维护AssociationsHashMap哈希Map，保存着实例对象地址与ObjectAssociationMap映射关系。
    2、ObjectAssociationMap还是一个哈希Map，保存key与关联属性的映射关系。
    3、使用objc_removeAssociatedObjects()方法，直接将ObjectAssociationMap干掉，所以该实例对象下的所有关联属性，都没有了。

```

## 源代码:
```

objc_setAssociatedObject

我们可以在 objc-references.mm 文件中找到 objc_setAssociatedObject 函数最终调用的函数：

void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
在看这段代码前，我们需要先了解一下几个数据结构以及它们之间的关系：

AssociationsManager 是顶级的对象，维护了一个从 spinlock_t 锁到 AssociationsHashMap 哈希表的单例键值对映射；
AssociationsHashMap 是一个无序的哈希表，维护了从对象地址到 ObjectAssociationMap 的映射；
ObjectAssociationMap 是一个 C++ 中的 map ，维护了从 key 到 ObjcAssociation 的映射，即关联记录；
ObjcAssociation 是一个 C++ 的类，表示一个具体的关联结构，主要包括两个实例变量，_policy 表示关联策略，_value 表示关联对象。
每一个对象地址对应一个 ObjectAssociationMap 对象，而一个 ObjectAssociationMap 对象保存着这个对象的若干个关联记录。


objc_getAssociatedObject

同样的，我们也可以在 objc-references.mm 文件中找到 objc_getAssociatedObject 函数最终调用的函数：

id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
    }
    return value;
}
看懂了 objc_setAssociatedObject 函数后，objc_getAssociatedObject 函数对我们来说就是小菜一碟了。这个函数先根据对象地址在 AssociationsHashMap 中查找其对应的 ObjectAssociationMap 对象，如果能找到则进一步根据 key 在 ObjectAssociationMap 对象中查找这个 key 所对应的关联结构 ObjcAssociation ，如果能找到则返回 ObjcAssociation 对象的 value 值，否则返回 nil 。


objc_removeAssociatedObjects

同理，我们也可以在 objc-references.mm 文件中找到 objc_removeAssociatedObjects 函数最终调用的函数：

void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
这个函数负责移除一个对象的所有关联对象，具体实现也是先根据对象的地址获取其对应的 ObjectAssociationMap 对象，然后将所有的关联结构保存到一个 vector 中，最终释放 vector 中保存的所有关联对象。根据前面的实验观察到的情况，在一个对象被释放时，也正是调用的这个函数来移除其所有的关联对象。
```

## note: 

```
1. 类对象也可以添加关联属性。
2.

```



