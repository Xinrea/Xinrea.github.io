---
title: "为 Go 添加三目运算符"
date: 2023-06-28
draft: true
tags: ["Compiler", "Go", "TypeCheck", "Ternary"]
ShowToc: true
---

最近在用 Go 刷算法题的过程中，切实体验到了三目运算缺失的痛苦。正好之前在探讨 Go 中 init 的处理时，了解了一些 Go 编译器的具体工作流程，因此有能力在其基础上进行修改了，于是便想着为 Go 添加三目运算。

实验仓库为 [go-with-ternary](https://github.com/Xinrea/go-with-ternary)，Commit 为 [compiler: add ternary support](https://github.com/Xinrea/go-with-ternary/commit/d68053553fb98988f01708a89f31f4888edafcc1)

> 修改所基于的分支为 release-branch.go1.21，版本为 1.21rc2；但是在写这篇文章的过程中，Go 1.21 正式版本于 2023/08/08 发布；而且修改未涉及 go/types，导致各种 lint 工具暂时不支持三目运算（但是能够正常编译运行）。基于以上原因，将会在 release 版本代码的基础上进行修改，并加上 lint 工具的支持。
