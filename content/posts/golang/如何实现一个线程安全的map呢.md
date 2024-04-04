---
title: "如何实现一个线程安全的 map 呢？"
date: 2022-03-27T21:30:20+08:00
tags: ["GO", "GO 基础"]
draft: false

---

> Golang 的 Map 不但是 `无序` 的，而且还是 `非线程安全的`。

## `读写锁` 实现线程安全 map

### 定义

```go
type SafeMap struct {
	mu sync.RWMutex
	m  map[string]int
}

func NewSafeMap() *SafeMap {
	return &SafeMap{
		m: make(map[string]int),
	}
}

func (sm *SafeMap) Get(key string) (int, bool) {
	sm.mu.RLock()
	defer sm.mu.RUnlock()
	val, ok := sm.m[key]
	return val, ok
}

func (sm *SafeMap) Set(key string, value int) {
	sm.mu.Lock()
	defer sm.mu.Unlock()
	sm.m[key] = value
}
```

### 使用

```go
safeMap := NewSafeMap()

// 读操作
val, ok := safeMap.Get("key")
if ok {
	println("Read:", val)
}

// 写操作
safeMap.Set("key", 1)
```

## `sync.Map` 实现线程安全 map

## `分段锁` 实现高性能线程安全 map

## `乐观锁` 实现线程安全 map