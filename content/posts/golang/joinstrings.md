---
title: "Golang 如何高效拼接字符串"
date: 2022-03-27T21:30:20+08:00
tags: ["GO", "GO 高性能", "GO 技巧"]
draft: false

---

> 在 Go 语言中，字符串(string) 是不可变的，拼接字符串事实上是创建了一个新的字符串对象。如果代码中存在大量的字符串拼接，对性能会产生严重的影响。

## 几种常见的拼接方式

1. 使用 <span style="color:#87CEEB; font-size:20px"> + </span> 进行拼接；
2. 使用 <span style="color:#87CEEB; font-size:20px"> fmt.Spintf() </span> 进行拼接;
3. 使用 <span style="color:#87CEEB; font-size:20px"> strings.Builder（可预分配） </span> 进行拼接（推荐）;
4. 使用 <span style="color:#87CEEB; font-size:20px"> bytes.Buffer </span> 进行拼接;
5. 使用 <span style="color:#87CEEB; font-size:20px"> []byte（可预分配） </span> 进行拼接；


## 性能比较

### 比较逻辑代码

> go test -bench="Concat$" -benchmem .  // 正则执行以 Concat 结尾的 benchmark 函数，-benchmem 表示显示内存分配信息

```go
package main

import (
	"bytes"
	"fmt"
	"math/rand"
	"strings"
	"testing"
)

const letterBytes = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func randomString(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = letterBytes[rand.Intn(len(letterBytes))]
	}
	return string(b)
}

func plusConcat(n int, str string) string {
	s := ""
	for i := 0; i < n; i++ {
		s += str
	}
	return s
}

func sprintfConcat(n int, str string) string {
	s := ""
	for i := 0; i < n; i++ {
		s = fmt.Sprintf("%s%s", s, str)
	}
	return s
}

func builderConcat(n int, str string) string {
	var builder strings.Builder
	for i := 0; i < n; i++ {
		builder.WriteString(str)
	}
	return builder.String()
}

func preBuilderConcat(n int, str string) string {
	var builder strings.Builder
	builder.Grow(n * len(str))
	for i := 0; i < n; i++ {
		builder.WriteString(str)
	}
	return builder.String()
}

func bufferConcat(n int, s string) string {
	buf := new(bytes.Buffer)
	for i := 0; i < n; i++ {
		buf.WriteString(s)
	}
	return buf.String()
}

func byteConcat(n int, str string) string {
	buf := make([]byte, 0)
	for i := 0; i < n; i++ {
		buf = append(buf, str...)
	}
	return string(buf)
}

/*
[]byte 的升级版：预分配容量
*/
func preByteConcat(n int, str string) string {
	buf := make([]byte, 0, n*len(str))
	for i := 0; i < n; i++ {
		buf = append(buf, str...)
	}
	return string(buf)
}

func benchmark(b *testing.B, f func(int, string) string) {
	var str = randomString(10)
	for i := 0; i < b.N; i++ {
		f(10000, str)
	}
}

func BenchmarkPlusConcat(b *testing.B)       { benchmark(b, plusConcat) }
func BenchmarkSprintfConcat(b *testing.B)    { benchmark(b, sprintfConcat) }
func BenchmarkBuilderConcat(b *testing.B)    { benchmark(b, builderConcat) }
func BenchmarkPreBuilderConcat(b *testing.B) { benchmark(b, preBuilderConcat) }
func BenchmarkBufferConcat(b *testing.B)     { benchmark(b, bufferConcat) }
func BenchmarkByteConcat(b *testing.B)       { benchmark(b, byteConcat) }
func BenchmarkPreByteConcat(b *testing.B)    { benchmark(b, preByteConcat) }

```

### 结果分析

```bash
BenchmarkPlusConcat-16                        22          55582405 ns/op        530998167 B/op     10026 allocs/op
BenchmarkSprintfConcat-16                     12          95173333 ns/op        833644274 B/op     34183 allocs/op
BenchmarkBuilderConcat-16                  12667            107742 ns/op          514801 B/op         23 allocs/op
BenchmarkPreBuilderConcat-16               29996             40795 ns/op          106496 B/op          1 allocs/op
BenchmarkBufferConcat-16                   14992             81504 ns/op          368577 B/op         13 allocs/op
BenchmarkByteConcat-16                      9874            112861 ns/op          621297 B/op         24 allocs/op
BenchmarkPreByteConcat-16                  24306             50880 ns/op          212992 B/op          2 allocs/op
```

- 四列结果分别表示：1s内该方法一共执行了多少次（越大越好）、ns/op 为每次执行该方法需要的时间（越小越好）、B/op 为每次执行函数分配多少内存（越小越好）、allocs/op 为每次执行函数一共分配了多少次内存（越小越好）；
- 使用 fmt.Sprintf 最差，分配的内存次数和大小最大；+ 性能也很差；
- strings.Builder（预分配）> []byte（预分配）> bytes.Buffer > strings.Builder > []byte > +号 > fmt.Spirntf

## 性能分析

