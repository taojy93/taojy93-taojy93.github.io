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

### 使用

```go
// 创建一个 sync.Map 实例
var m sync.Map

// 在多个 goroutine 中并发地写入键值对
go func() {
    for i := 0; i < 10; i++ {
        m.Store(fmt.Sprintf("key%d", i), i)
    }
}()

// 在多个 goroutine 中并发地读取键值对
go func() {
    m.Range(func(key, value interface{}) bool {
        fmt.Printf("Read key: %v, value: %v\n", key, value)
        return true
    })
}()

// 在多个 goroutine 中并发地读取键值对
go func() {
    defer wg.Done()
    for i := 0; i < 10; i++ {
        v, _ := m.Load(fmt.Sprintf("key%d", i))
        fmt.Printf("Read key: %v, value: %v\n", fmt.Sprintf("key%d", i), v)
    }
}()
```

## `分段锁` 实现高性能线程安全 map

### 定义

```go
package shardlock

import "sync"

// smallMap 并发安全的小map，ShardCount 个这样的小map数组组成一个大map
type smallMap struct {
	items        map[string]interface{}
	sync.RWMutex // 读写锁，保护items
}

// ShardCount 底层小shareMap数量
var ShardCount = 32

// bigMap 并发安全的大map，由 ShardCount 个小map数组组成，方便实现分段锁机制
type bigMap []*smallMap

// New 创建一个新的concurrent map.
func New() bigMap {
	m := make(bigMap, ShardCount)
	for i := 0; i < ShardCount; i++ {
		m[i] = &smallMap{
			items: make(map[string]interface{}),
		}
	}
	return m
}

// GetShardMap 返回给定key的sharedMap
func (m bigMap) GetShardMap(key string) *smallMap {
	return m[uint(fnv32(key))%uint(ShardCount)]
}

// fnv32 hash函数
func fnv32(key string) uint32 {
	// 著名的fnv哈希函数，由 Glenn Fowler、Landon Curt Noll和 Kiem-Phong Vo 创建
	hash := uint32(2166136261)
	const prime32 = uint32(16777619)
	keyLength := len(key)
	for i := 0; i < keyLength; i++ {
		hash *= prime32
		hash ^= uint32(key[i])
	}
	return hash
}

// Set 添加 key-value
func (m bigMap) Set(key string, value interface{}) {
	// Get map shard.
	shard := m.GetShardMap(key)
	shard.Lock()
	shard.items[key] = value
	shard.Unlock()
}

// Get 返回指定key的value值
func (m bigMap) Get(key string) (interface{}, bool) {
	shard := m.GetShardMap(key)
	shard.RLock()
	val, ok := shard.items[key]
	shard.RUnlock()
	return val, ok
}

// Remove 删除一个元素
func (m bigMap) Remove(key string) {
	// Try to get shard.
	shard := m.GetShardMap(key)
	shard.Lock()
	delete(shard.items, key)
	shard.Unlock()
}

// Has 判断元素是否存在
func (m bigMap) Has(key string) bool {
	// Get shard
	shard := m.GetShardMap(key)
	shard.RLock()
	// See if element is within shard.
	_, ok := shard.items[key]
	shard.RUnlock()
	return ok
}

// Count 统计元素总数
func (m bigMap) Count() int {
	count := 0
	for i := 0; i < ShardCount; i++ {
		shard := m[i]
		shard.RLock()
		count += len(shard.items)
		shard.RUnlock()
	}
	return count
}

// Keys 以字符串数组的形式返回所有key
func (m bigMap) Keys() []string {
	count := m.Count()
	ch := make(chan string, count)
	go func() {
		wg := sync.WaitGroup{}
		wg.Add(ShardCount)
		for _, shard := range m {
			go func(shard *smallMap) {
				shard.RLock()
				for key := range shard.items {
					ch <- key
				}
				shard.RUnlock()
				wg.Done()
			}(shard)
		}
		wg.Wait()
		close(ch)
	}()

	keys := make([]string, 0, count)
	for k := range ch {
		keys = append(keys, k)
	}
	return keys
}
```

### 使用

```go
m := shardlock.New()
m.Set("key", 2)
v, _ := m.Get("key")
```

## 第三方分段锁包实现线程安全 map

```bash
https://github.com/orcaman/concurrent-map
```


## `乐观锁` 实现线程安全 map