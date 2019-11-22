---
layout:     post
title:      "iOS-Runtime"
subtitle:   "Runtime获取类中的属性"
date:       2017-02-04 12:00:00
author:     "ZP"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
catalog: true
tags:
    - iOS-Runtime
---

## 获取对象中属性名称、属性类型、属性值

**Ivar - 代码1**

```


    // 获取对象的类文件
    Class clazz = [obj class];

    // 获取该类的某个属性    
    Ivar v = class_getInstanceVariable([obj class], "_orgFullName");
    // 修改该对象中该属性的值
    object_setIvar(obj, v, @"联子科技");
    // 获取该对象中该属性的值
    NSLog(@"%@", object_getIvar(obj, v));
    
    int paramCount = 0;
    // 获取该类的所有属性列表，会取出该类的所有属性，包括写在.m文件中的私有属性
    Ivar *ivar = class_copyIvarList(clazz, &paramCount);
    // 遍历属性列表
    for (int i=0; i<paramCount; i++) {
        // 获取属性名称
        const char *paramName = ivar_getName(ivar[i]);
        // 获取属性类型
        const char *paramType = ivar_getTypeEncoding(ivar[i]);
        
        NSString *paramNameStr = [[NSString stringWithUTF8String:paramName] substringFromIndex:1];
        // 取出该属性的值
        id value = [obj valueForKey:paramNameStr];
    }
    
    // 释放内存
    free(ivar);
    
    
    －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
    
    动态给对象添加成员变量
```


**property - 代码2**

```


    // 获取对象的类文件
    Class clazz = objc_getClass(@"OrgTableModel".UTF8String);
    
    int paramCount = 0;
    // 获取该类的所有属性列表，会取出该类的所有属性，包括写在.m文件中的私有属性
    objc_property_t *property = class_copyPropertyList(clazz, &paramCount);
    // 遍历属性列表
    for (int i=0; i<paramCount; i++) {
        // 获取属性名称
        const char *paramName = property_getName(property0[i]);
        
        NSString *paramNameStr = [[NSString stringWithUTF8String:paramName] substringFromIndex:1];
        // 取出该属性的值
        id value = [obj valueForKey:paramNameStr];
    }
    
    // 释放内存
    free(property);
    
```

－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

**API介绍**

1. class_getInstanceVariable(Class s, char *paramName); // 获取该类的属性对象

2. object_setIvar(id obj, Ivar v, id value); // 设置该类的属性对象的值

3. object_getIvar(id obj, IVar v); // 获取该类的属性对象的值

4. class_copyIvarList(Class clazz, int *paramCount); // 获取该类的所有属性列表，会取出该类的所有属性，包括写在.m文件中的私有属性

5. ivar_getName(Ivar v); // 获取该属性的名称

6. ivar_getTypeEncoding(Ivar v); // 获取该属性的类型
7. free(Ivar *v); // 释放内存 


－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

**ivar_getTypeEncoding函数返回值**

1. 返回都是char * 类型字符串
2. 根据机子的类型不同，返回的参数值可能不同，NSInteger在32位对应为int，在64位对应long。
3. 对象类型 : "@\"NSString\""
4. NSInteger : q
5. int : i
6. long : l
7. double : d
8. BOOL : B

－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

**class_copyIvarList 与 class_copyPropertyList区别**

1. class_copyPropertyList ：对象成员属性(@propert关键字修饰的对象属性)
2. class_copyIvarList ：对象成员属性 ＋ 对象成员变量

```

@interface OrgTableModel : NSObject
{
    NSString *orgFullName;   -- 对象成员变量
    NSString *orgEmail;
}
// 组织ID
@property (nonatomic, assign) NSInteger orgId; -- 对象成员属性
// 组织类型(商会、协会等)
@property (nonatomic, copy) NSString *orgType;
// 组织简称
@property (nonatomic, copy) NSString *orgShortName;

@end

```


## 实例：Model转SQL语句

```

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.tableView.backgroundColor = [UIColor whiteColor];
    
    OrgTableModel *model = [[OrgTableModel alloc] init];
    model.orgId = 1112;
    model.orgFullName = @"dadada";
    model.orgShortName = @"AAAA";
    
    NSString *sql = [self Model2SQL:model];
    
    NSLog(@"%@", sql);
}

// 判断C字符串是否相等
BOOL isEquelChar(char *first, char *second)
{
    int charCount = sizeof(char);
    int firstCount = sizeof(first);
    int secondCount = sizeof(second);
    
    for (int i = 0; i < firstCount/charCount; i++) {
        if(*(first + i) != *(second + i)){
            return NO;
        }
    }
    
    if (firstCount == secondCount) {
        return YES;
    }else{
        return NO;
    }
}

- (NSString *)Model2SQL:(id)obj
{
    Class clazz = [obj class];
    
    unsigned int paramCount = 0;
    
    Ivar *ivar = class_copyIvarList(clazz, &paramCount);
    
    
    NSMutableString *sqlM = [NSMutableString stringWithFormat:@"insert into %@ ", NSStringFromClass(clazz)];
    // SQL中的参数
    NSMutableString *SqlParamStr = [NSMutableString stringWithFormat:@"("];
    // SQL中的值
    NSMutableString *SqlParamValue = [NSMutableString stringWithFormat:@"values("];
    for (int i=0; i<paramCount; i++) {
        const char *paramName = ivar_getName(ivar[i]);
        const char *paramType = ivar_getTypeEncoding(ivar[i]);
        

        // 删除对象属性的下划线前缀
        NSString *paramNameStr = [[NSString stringWithUTF8String:paramName] substringFromIndex:1];
        NSString *paramTypeStr = [NSString stringWithUTF8String:paramType];
        
        // 判断属性类型是否为整型
        if (*paramType == 'l' || *paramType == 'i' || *paramType == 'q') {
            NSInteger num = [[obj valueForKey:paramNameStr] integerValue];
            if (num > 0) {
                [SqlParamStr appendFormat:@"%@,", paramNameStr];
                [SqlParamValue appendFormat:@"%ld,",num];
            }
           // 判断属性类型是否为NSString 
        }else if ([paramTypeStr isEqualToString:@"@\"NSString\""]){
            
            id value = [obj valueForKey:paramNameStr];
            if (value != nil) {
                [SqlParamStr appendFormat:@"%@,", paramNameStr];
                [SqlParamValue appendFormat:@"'%@',",[obj valueForKey:paramNameStr]];
            }
        }

    }
    
    [SqlParamStr replaceCharactersInRange:NSMakeRange(SqlParamStr.length - 1, 1) withString:@")"];
    [SqlParamValue replaceCharactersInRange:NSMakeRange(SqlParamValue.length - 1, 1) withString:@")"];
    
    // 拼接字符串，形成SQL语句
    [sqlM appendString:SqlParamStr];
    [sqlM appendString:SqlParamValue];
    // 是否内存
    free(ivar);
    
    return sqlM;
}

@end

```

