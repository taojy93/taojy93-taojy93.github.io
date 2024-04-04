---
title: "Golang 的一些陷阱"
date: 2022-03-27T21:30:20+08:00
tags: ["GO", "GO 基础"]
draft: false

---

## for range 的陷阱

```go
m := map[int]string{
	1: "one",
	2: "two",
	3: "three",
}

sli := []*string{}
for _, v := range m {
	sli = append(sli, &v)
}

fmt.Println(*sli[0], *sli[1], *sli[2]) // three three three
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为什么每次输出都一样呢？因为 for range 循环的时候 v 是一个副本，地址是不变的，所以不要把 &v 作为值复制给别的变量值，要不然每次读取的都是同一个地址上的值，当然是一样的了。