---
layout: post
title:  "Lua 源码学习——数据结构"
date:   2018-04-17 01:47:30 +0800
categories: lua
---

* TOC
{:toc}


## 数据结构简介

Lua 中支持的数据结构包括：

| 宏                 | 类型            | 对应数据结构         |
| ------------------ | -------------- | ------------------ |
| LUA_TNONE          |                |                    |
| LUA_TNIL           | nil            |                    |
| LUA_TBOOLEAN       | boolean        | int                |
| LUA_TLIGHTUSERDATA | pointer        | void *             |
| LUA_TNUMBER        | number         | lua_Number         |
| LUA_TSTRING        | string         | TString            |
| LUA_TTABLE         | table          | Table              |
| LUA_TFUNCTION      | function       | CClosure, LClosure |
| LUA_TUSERDATA      | pointer        | void *             |
| LUA_TTHREAD        | lua vm, thread | lua_State          |

注意：
- LUA_TLIGHTUSERDATA 与 LUA_TUSERDATA 都是 void * 指针，它们的区别是，前者的分配释放由 Lua 外部的使用者负责、后者的分配释放由 Lua 内部完成；
- LUA_TSTRING 及以后的类型，需要进行 GC 操作；

之前已经多次说过，Lua 使用联合体 lua_TValue 来同时表示上述类型的数据。其定义类似于：

```c
union GCObject {
    GCheader gch;
    union TString ts;       // string
    union Udata u;          // userdata
    union Closure cl;       // function
    struct Table h;         // table
    struct Proto p;
    struct UpVal uv;
    struct lua_state th;    // thread
};

union Value {
    GCObject* gc;   // 所有可 GC 的数据，包括 string, table, function, userdata, thread
    void* p;        // light userdata
    lua_Number n;   // number
    int b;          // boolean
};

struct lua_TValue {
    int tt;         // 数据类型
    Value value;    // 数据
};
```


## 字符串

正如之前所了解的，Lua 中的字符串是一个不可变的数据。事实上，Lua 虚拟机存在一块全局的数据区，用来存放此虚拟机中的所有字符串。



```c
/*
** String headers for string table
*/
typedef union TString {
  L_Umaxalign dummy;  /* ensures maximum alignment for strings */
  struct {
    CommonHeader;
    lu_byte reserved;
    unsigned int hash;
    size_t len;
  } tsv;
} TString;
```