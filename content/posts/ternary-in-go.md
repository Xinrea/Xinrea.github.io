---
title: "为 Go 添加三目运算符"
date: 2023-06-28
draft: true
tags: ["Compiler", "Go", "TypeCheck", "Ternary"]
ShowToc: true
---

最近在用 Go 刷算法题的过程中，切实体验到了三目运算缺失的痛苦。正好之前在探讨 Go 中 init 的处理时，了解了一些 Go 编译器的具体工作流程，因此有能力在其基础上进行修改了，于是便想着为 Go 添加三目运算。
