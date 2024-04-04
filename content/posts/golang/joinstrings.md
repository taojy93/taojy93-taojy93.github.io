---
title: "Golang 如何高效拼接字符串"
date: 2022-03-27T21:30:20+08:00
tags: ["GO", "GO 高性能", "GO 技巧"]
draft: false

---

> 在 Go 语言中，字符串(string) 是不可变的，拼接字符串事实上是创建了一个新的字符串对象。如果代码中存在大量的字符串拼接，对性能会产生严重的影响。

## 几种常见的拼接方式

1. 使用 <span style="color:#87CEEB; font-size:20px">` + `</span> 进行拼接；
2. 使用 <span style="color:#87CEEB; font-size:20px">` fmt.Spintf() `</span> 进行拼接;
3. 使用 <span style="color:#87CEEB; font-size:20px">` strings.Builder（可预分配） `</span> 进行拼接（推荐）;
4. 使用 <span style="color:#87CEEB; font-size:20px">` bytes.Buffer `</span> 进行拼接;
5. 使用 <span style="color:#87CEEB; font-size:20px">` []byte（可预分配） `</span> 进行拼接；


## 性能比较

### 比较逻辑代码

> `go test -bench="Concat$" -benchmem .`  // 正则执行以 Concat 结尾的 benchmark 函数，-benchmem 表示显示内存分配信息

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

> 我们发现上面 `内存分配次数越少`，性能越高；`内存分配大小越小`，性能越高；

### `strings.Builder` 的性能会比 `+` 提高那么多的原因

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为字符串的修改原理是：因为字符串可以看做是分配在 `只读内存空间`上的 []byte，每次修改需要先在 `可写内存空间` 上面分配内存变量 []byte，把 `只读内存空间` 上面的 []byte 转移到 `可写内存空间` []byte 上，然后修改，修改完成之后需要再写入到 `只读内存空间` （这个 `新只读空间` 和之前那个 `旧只读空间` 完全没有关系）。<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以每次 + 拼接字符串其实就是一次修改字符串，都需要重复上面的分配内存操作；而且因为前后两次的 `只读内存空间` 完全不一致，所以需要更多的内存分配。<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可是 `strings.Builder` 其实底层是一个 `[]byte` 和一个 `指向 []byte首位空间的指针地址` 组成；所以并不是每次拼接都需要重新分配内存，只有在 []byte 现需要扩容的时候才会重新分配内存；所以它的分配次数会大大降低；<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如：现在有一个字符串 "a"，循环拼接 1024 次；<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 `+` 拼接：因为每次都需要分配内存，所以一共需要分配 1024 次内存，总共需要的内存空间是 1 + 1 * 2 + 1 * 3 + 1 * 4 + ... + 1 * 1024 byte = 5MB<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 `strings.Builder` 拼接：因为只有触发 slice 扩容的时候才需要分配内存，触发扩容的时机是：第 1 次、 第 2 次、第 4 次，第 8 次 ... 第 29 次，一共需要分配 10 次即可，总共需要分配的内存大小：1 + 2 + 4 + ... + 1024 byte = 5KB

### `预分配` 的性能得到大幅提升的原因

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为采用 `预分配` 几乎完全避免了内存的分配，我们知道内存的频繁分配是造成性能低下的主要原因，预分配只用在初次分配一次即可，所以相比较 `+` 拼接每次都需要分配一次内存，以及相比较 `strings.Builder` 和 `bytes.Buffer` 借助 slice 的扩容策略进行分配内存，有了质的提升。所以性能得到了大幅度的提升。
