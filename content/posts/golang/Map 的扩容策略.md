---
title: "map “扩容策略” 和 “如何应对哈希冲突” "
date: 2022-03-27T21:30:20+08:00
tags: ["GO", "GO 基础"]
draft: false

---

> Go 的 Map 有两种扩容策略："等量扩容" 和 "double扩容"

## double扩容

### 触发原因

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当`装载因子`> `6.5` 的时候就会触发 `double扩容`，因为此时数据太多，会导致查询和写入的效率大大降低。（装载因子=kv数/bucket）

### 扩容过程


## 等量扩容

### 触发原因

### 扩容过程