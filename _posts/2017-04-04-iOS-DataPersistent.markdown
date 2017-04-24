---
layout:     post
title:      "数据库之旅"
subtitle:   "SQL－C语言API"
date:       2017-02-03 12:00:00
author:     "Hux"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
tags:
    - 数据库
---

## sqlite3的C语言API


**数据库连接、释放API**

```
// 创建数据库连接

 int SQLITE_STDCALL sqlite3_open(
  const char *filename,   /* Database filename (UTF-8) */
  sqlite3 **ppDb          /* OUT: SQLite db handle */
);

 int SQLITE_STDCALL sqlite3_open16(
  const void *filename,   /* Database filename (UTF-16) */
  sqlite3 **ppDb          /* OUT: SQLite db handle */
);

 int SQLITE_STDCALL sqlite3_open_v2(
  const char *filename,   /* Database filename (UTF-8) */
  sqlite3 **ppDb,         /* OUT: SQLite db handle */
  int flags,              /* Flags 可读可写、只读、创建数据库文件*/
  const char *zVfs        /* Name of VFS module to use */
);

// 释放数据库连接

int sqlite3_close(sqlite3*); // 释放sqlite3_open或sqlite3_open16函数创建的数据库连接
int sqlite3_close_v2(sqlite3*); // 释放sqlite3_open_v2函数创建的数据库连接

注释：
1、sqlite3_open() == sqlite3_open16()，区别在于db文件的编码UTF8或者UTF16。
2、sqlite3_open_v2()函数是sqlite3_open()函数功能加强版，修改文件打开方式。
3、sqlite3_open()和sqlite3_open16()，默认以可读可写方式打开文件，如果不能打开则以只读方式打
   开文件，如果文件不存在，则创建该文件。
4、sqlite3_open_v2()函数：
    1)如果设为可读可写，如果文件只有读操作时，则以只读方式打开，如果文件不存在，则返回"SQLITE_CANTOPEN"参数，返回文件不能打开；
    2)如果设为"SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE"，如果文件不存在则创建一个新文件。
    
```


**数据库SQL编译、执行API**

```

int sqlite3_prepare(
  sqlite3 *db,            /* Database handle */
  const char *zSql,       /* SQL statement, UTF-8 encoded */
  int nByte,              /* Maximum length of zSql in bytes. */
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */
  const char **pzTail     /* OUT: Pointer to unused portion of zSql */
);

int sqlite3_prepare_v2(
  sqlite3 *db,            /* Database handle */
  const char *zSql,       /* SQL statement, UTF-8 encoded */
  int nByte,              /* Maximum length of zSql in bytes. */
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */
  const char **pzTail     /* OUT: Pointer to unused portion of zSql */
);

int sqlite3_prepare16(
  sqlite3 *db,            /* Database handle */
  const void *zSql,       /* SQL statement, UTF-16 encoded */
  int nByte,              /* Maximum length of zSql in bytes. */
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */
  const void **pzTail     /* OUT: Pointer to unused portion of zSql */
);

int sqlite3_prepare16_v2(
  sqlite3 *db,            /* Database handle */
  const void *zSql,       /* SQL statement, UTF-16 encoded */
  int nByte,              /* Maximum length of zSql in bytes. */
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */
  const void **pzTail     /* OUT: Pointer to unused portion of zSql */
);

int sqlite3_step(sqlite3_stmt*); // 执行SQL语句



注释：
1、该方法将sql语句编译成二进制可执行数据，相当于java的class文件。
2、参数说明：
    1)数据库连接
    2)SQL语句utf8、utf16编码
    3)SQL语句长度
    4)语句句柄，保存SQL编译后的数据，和查询结果的结构体数据结构
    5)下一条要执行的sql语句

```



**数据库参数化设置API**


**数据库结果查询APIAPI**

