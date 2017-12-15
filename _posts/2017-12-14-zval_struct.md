---
layout: post
title: "php7底层变化"
categories: 软件工具
tags:  php  
author: sunzhen
---

php7修改了底层zval结构体，现在的结构体占用内存更小。要理解为什么php中zval占用内存变小，首先需要理解c语言的字节对齐原则。
# 字节对齐

## 例子

```c
#include <stdio.h>
struct {
    int a;
    char b;
    short c;
}s1;
struct{
    char b;
    int a;
    short c;
}s2;
int main()
{
    printf("s1: %d \n",sizeof(s1));
    printf("s2: %d \n",sizeof(s2));
    return 0;
}

```
执行结果：
```shell
s1: 8 
s2: 12 
```
s1和s1大小不同，这就涉及到内存对齐

## 内存对齐原则

- 数据成员对齐规则：结构(struct)(或联合(union))的数据成员，第一个数据成员放在offset为0 的地方，以后每个数据成员的对齐按照#pragma pack 指定的数值和这个数据成员自身长度中，比较小的那个进行。
- 结构(或联合)的整体对齐规则：在数据成员完成各自对齐之后，结构(或联合)本身也要进行对齐，对齐将按照#pragma pack 指定的数值和结构(或联合)最大数据成员长度中，比较小的那个进行。
- 如果#pragma pack(n)比结构中任何一个数据成员类型都大，则对齐系数不起任何作用。

分析一下上面s1和s2的结构体。

- s1：

int a类型占用地址0~3，b 1字节，偏移4，c两字节，偏移需要是2的倍数，所以偏移6。此时在内存中占用8个字节，正好是int型的两倍，根据结构体对齐不需要添加额外字节。

- s2：

char占一个字节，起始偏移为0 ，int 占4个字节，所以int按4字节对齐，起始偏移必须为4的倍数，所以起始偏移为4，在char后编译器会添加3个字节的额外字节，不存放任意数据。short占2个字节，按2字节对齐，起始偏移为8，正好是2的倍数，无须添加额外字节。此时总共10个字节，按照结构体整体对齐原则，占用字节数需是int型的整数倍，所以后面需要添加两个字节，总共占用12字节。

# 变量在php5内部的实现

PHP5 中 zval 结构体定义如下：
```c
typedef struct _zval_struct {
 zvalue_value value;
 zend_uint refcount__gc;
 zend_uchar type;
 zend_uchar is_ref__gc;
} zval;
```
如上，zval 包含一个 value、一个 type 以及两个 __gc 后缀的字段。value 是个联合体，用于存储不同类型的值：

```c
typedef union _zvalue_value {
 long lval;     // 用于 bool 类型、整型和资源类型
 double dval;    // 用于浮点类型
 struct {     // 用于字符串
  char *val;
  int len;
 } str;
 HashTable *ht;    // 用于数组
 zend_object_value obj;  // 用于对象
 zend_ast *ast;    // 用于常量表达式(PHP5.6 才有)
} zvalue_value;
```
因为要支持循环回收，实际使用的 zval 的结构实际上如下：
```c
typedef struct _zval_gc_info {
 zval z;
 union {
  gc_root_buffer  *buffered;
  struct _zval_gc_info *next;
 } u;
} zval_gc_info;
```
在 64 位的系统上。zvalue_value 这个联合体占用 16 个字节的内存,整个 zval 结构体占用的内存是 24 个字节（考虑到内存对齐）, 再加上zval_gc_info,需要32个字节。

zval 中 refcount__gc 的值用于保存 zval 本身被引用的次数，和引用计数相关的概念就是引用计数和循环引用。

## 写时复制
```php
$a = 42; // $a   -> zval_1(type=IS_LONG, value=42, refcount=1)
$b = $a; // $a, $b  -> zval_1(type=IS_LONG, value=42, refcount=2)
$c = $b; // $a, $b, $c -> zval_1(type=IS_LONG, value=42, refcount=3)

// 下面几行是关于 zval 分离的
$a += 1; // $b, $c -> zval_1(type=IS_LONG, value=42, refcount=2)
   // $a  -> zval_2(type=IS_LONG, value=43, refcount=1)

unset($b); // $c -> zval_1(type=IS_LONG, value=42, refcount=1)
   // $a -> zval_2(type=IS_LONG, value=43, refcount=1)

unset($c); // zval_1 is destroyed, because refcount=0
   // $a -> zval_2(type=IS_LONG, value=43, refcount=1)
```
在$b=$a,$c=$b时实际上是没有内存开销的，只有在$a值改变时重新分配一块内存空间。

## 循环引用

PHP5.2以前引擎在判断一个变量空间是否能够被释放的时候是依据这个变量的zval的refcount的值，如果refcount为0，那么变量的空间可以被释放，否则就不释放，这是一种非常简单的GC实现。

比如如下：
```php
$a = array(1,2);
```
$a实现为：
```c
a: (refcount=1, is_ref=0)=array (
   0 => (refcount=1, is_ref=0)=1,
   1 => (refcount=1, is_ref=0)=2
)
```
那么如下代码：
```php
$a = array(1);
$a[] = &$a;
//unset($a);
```
内部实现：
```c
a: (refcount=2, is_ref=1)=array (
   0 => (refcount=1, is_ref=0)=1,
   1 => (refcount=2, is_ref=1)=...
)
```
“...”表示1指向a自身，是一个环形引用,这时候加上unset($a)会发生如下情况：$a不在符号表中，但是$a之前指向的zval的refcount变为1而不是0，因此不能被回收，这样产生了内存泄露。所以在5.3之后使用zval_gc_info为单位分配值，并且设计新的垃圾回收算法。

# 变量在php7内部的实现

php7中结构体定义：
```c
struct _zval_struct {
 zend_value  value;   /* value */
 union {
  struct {
   ZEND_ENDIAN_LOHI_4(
    zend_uchar type,   /* active type */
    zend_uchar type_flags,
    zend_uchar const_flags,
    zend_uchar reserved)  /* call info for EX(This) */
  } v;
  uint32_t type_info;
 } u1;
 union {
  uint32_t  var_flags;
  uint32_t  next;     /* hash collision chain */
  uint32_t  cache_slot;   /* literal cache slot */
  uint32_t  lineno;    /* line number (for ast nodes) */
  uint32_t  num_args;    /* arguments number for EX(This) */
  uint32_t  fe_pos;    /* foreach position */
  uint32_t  fe_iter_idx;   /* foreach iterator index */
 } u2;
};

typedef union _zend_value {
 zend_long   lval;    /* long value */
 double   dval;    /* double value */
 zend_refcounted *counted;
 zend_string  *str;
 zend_array  *arr;
 zend_object  *obj;
 zend_resource *res;
 zend_reference *ref;
 zend_ast_ref  *ast;
 zval    *zv;
 void    *ptr;
 zend_class_entry *ce;
 zend_function *func;
 struct {
  uint32_t w1;
  uint32_t w2;
 } ww;
} zend_value;
```
现在 value 联合体需要的内存是 8 个字节。它只会直接存储整型（lval）或者浮点型（dval）数据，其他情况下都是指针。与php5相比，主要变化是zval不再自己存储引用计数，交由复杂数据类型（比如字符串、数组和对象）结构体自身存储。
值得注意的一点是php7重新设计了string类型和数组类型的结构体。

## 字符串的变化

PHP7 中定义了一个新的结构体 zend_string 用于存储字符串变量：
```c
struct _zend_string {
 zend_refcounted gc;
 zend_ulong  h;
 size_t   len;
 char    val[1];
};
```
使用char数组而不是char *可以降低cpu的cache miss。

## 数组变化

在php5的hashtable中，大量使用指针，结构体大小有72字节。php7中：
```c
typedef struct _Bucket {
    zval              val;
    zend_ulong        h;                /* hash value (or numeric index)   */
    zend_string      *key;              /* string key or NULL for numerics */
} Bucket;

typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                    zend_uchar    flags,
                    zend_uchar    nApplyCount,
                    zend_uchar    nIteratorsCount,
                    zend_uchar    reserve)
        } v;
        uint32_t flags;
    } u;
    uint32_t          nTableMask; //哈希值计算掩码，等于nTableSize的负值(nTableMask = ~nTableSize + 1)
    Bucket           *arData; //存储元素数组，指向第一个Bucket
    uint32_t          nNumUsed; //已用Bucket数
    uint32_t          nNumOfElements; //哈希表已有元素数
    uint32_t          nTableSize; //哈希表总大小，为2的n次方
    uint32_t          nInternalPointer;
    zend_long         nNextFreeElement; //下一个可用的数值索引,如:arr[] = 1;arr["a"] = 2;arr[] = 3;  则nNextFreeElement = 2;
    dtor_func_t       pDestructor;
};
```
几个重大的变化是：

- buckets直接存zval，而不再是buckets的pdata指针指向独立的zval结构，
- 原来的hash冲突是buket结构体本身用拉链法处理的，现在交由zval_struct.u2.next字段处理。
- 数组元素的Bucket的内存空间是一同分配的，按顺序存放。原来是通过指针，减少了cache miss。
- HashTable的大小从72下降到56字节
- Buckets的大小从72下降到32字节