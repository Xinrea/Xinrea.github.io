---
layout: post
title: "ðç»æä½çæ°æ®å¯¹é½é®é¢"
date: 2018-08-01
tags: C++
---
> ä»¥ä¸æ¹æ³åä¸ºGCCæ©å±æ¹æ³

### E1 é»è®¤

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

çç¥`aligned(X)`æ¶ï¼ç¸å½äºé»è®¤X=1

å¶ä¸­`packed`æç»æä½åé¨æåé´åæ¶paddingï¼ç´§åæåã

`aligned`æç»æä½æç»å¤§å°åºä¸ºXçæ´æ°åã**æ³¨æï¼æªè®¾ç½®`packed`æ¶ï¼Xçå¼åªä¼å¨å¤§äºç­äºç»æä½ä¸­æå¤§çæåå¤§å°æ¶çæ**

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

çç¥`X`æ¶ï¼`pack(push)`æä½æ æ

ä¸é¢æä¾å³äº`packed`å`aligned`çæµè¯æ°æ®ï¼

### ç¬¬ä¸ç»

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

### ç¬¬äºç»

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