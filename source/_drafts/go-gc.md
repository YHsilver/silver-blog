---
title: Go 垃圾回收介绍
date: 2023-03-12 15:07:10
tags:
categories:
description:
cover:
---

本文基于go.v1.5版本

## 什么是垃圾回收机制

从内存分配机制讲起

三块区域

## Go的垃圾回收机制

### 标记-清除算法



### 算法优化：三色标记法

为什么要这样设计？

屏障技术

## Go垃圾回收器的实现原理



## Go GC调优



## 参考资料

1. The Go Blog: Go GC: Prioritizing low latency and simplicity, July 13, 2015 (https://blog.golang.org/go15gc)
2. Go 语言圣经: 垃圾回收, https://books.studygolang.com/gopl-zh/ch5/ch5-08.html
3. Go 中 GC 的优化和调优, https://tonybai.com/2016/09/23/optimizing-and-tuning-go-gc/
