---
layout: post
title: "结构体的数据对齐问题"
date: 2018-08-01
tags: c c++ alignment
---
> 以下方法均为GCC扩展方法

### E1 默认

```clike
struct test {
        char a;
        int b;
};
test t1;
```

`sizeof(t1) = 8`

### E2 \_\_attribute\_\_((packed, aligned(X)))

```clike
struct __attribute__((packed,aligned(1))) test2{
        char a;
        int b;
};
test2 t2;
```

`sizeof(t2) = 5`

省略`aligned(X)`时，相当于默认X=1

其中`packed`指结构体内部成员间取消padding，紧凑排列。

`aligned`指结构体最终大小应为X的整数倍。**注意，未设置`packed`时，X的值只会在大于等于结构体中最大的成员大小时生效**

### E3 \#pragma pack(push,X)

```clike
#pragma pack(push,1)
struct test3 {
        char a;
        int b;
};
#pragma pack(pop)
test3 t3;
```

`sizeof(t3) = 5`

省略`X`时，`pack(push)`操作无效

下面提供关于`packed`和`aligned`的测试数据：

### 第一组

```clike
struct st {
  char a;
  int b;
  char c;
};
```

Aligned | 1 | 2 | 4 | 8 |
:------:|:-:|:-:|:-:|:-:|
Packed  | 6 | 6 | 8 | 8 |
Unpacked| 12 | 12 | 12 | 16 |

### 第二组

```clike
struct st {
  char a;
  double b;
  char c;
};
```

Aligned | 1 | 2 | 4 | 8 | 16 |
:------:|:-:|:-:|:-:|:-:|:--:|
Packed  | 10 | 10 | 12 | 16 | 16 |
Unpacked| 24 | 24 | 24 | 24 | 32 |