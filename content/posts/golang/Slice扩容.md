---
title: "slice 扩容策略"
date: 2022-04-27T21:30:20+08:00
tags: ["GO", "GO 基础"]
draft: false

---

> 各个版本的扩容策略有些区别，我们谈谈 1.18 之前和之后

## <1.18 版本

### 源码

```go
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
    newcap = cap
} else {
    if old.cap < 1024 {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < cap {
            newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = cap
        }
    }
}
```

### 分析

- 当 `需要容量` > `旧容量` 的 `2倍`
    - 那么 `新容量` = `需要容量`；
- 当 `需要容量` <= `旧容量` 的 `2倍`;
    - 如果 `旧容量` < `1024`，那么 `新容量` = `旧容量 * 2`；
    - 如果 `旧容量` >= `1024`，那么 `新容量` = `旧容量 + 旧容量/4`；

## >=1.18 版本

### 源码

```go
newcap := oldCap
doublecap := newcap + newcap
if newLen > doublecap {
    newcap = newLen
} else {
    const threshold = 256
    if oldCap < threshold {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < newLen {
            // Transition from growing 2x for small slices
            // to growing 1.25x for large slices. This formula
            // gives a smooth-ish transition between the two.
            newcap += (newcap + 3*threshold) / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = newLen
        }
    }
}
```

### 分析

- 当 `需要容量` > `旧容量` 的 `2倍`
    - 那么 `新容量` = `需要容量`；
- 当 `需要容量` <= `旧容量` 的 `2倍`;
    - 如果 `旧容量` < `256`，那么 `新容量` = `旧容量 * 2`；
    - 如果 `旧容量` >= `256`，那么 `新容量` = `旧容量 + (旧容量+3*256)/4`；


## 扩容策略升级的原因

> 1.18版本在 `需要容量` < `2 * 旧容量` && `旧容量` >= `x` 的时候做了优化，把 `x` 从 1024 调整成了 256，并且扩容算法从每次扩 `旧容量/4` 调整为每次扩 `(旧容量+3*256)/4`。这么做的原因就是为了使扩容更加平滑。<br />

### 场景验证（旧版本）

- （场景1）现在有一个 1023 的 slice 需要 append 一个 len=2 的切片，那么这个时候就触发了扩容。
- （场景1）扩容之后就是 1023 * 2 = 2046；
- （场景2）现在有一个 1024 的 slice 需要 append 一个 len=2 的切片，那么这个时候就触发了扩容。
- （场景2）扩容之后就是 1024 + 1024/4 = 1280；
- <mark>我们发现 `小` 的扩容后反而比 `大` 的扩容后还大，这不符合线性规律，扩容不够平滑。</mark>

### 场景验证（新版本）

- （场景1）现在有一个 255 的 slice 需要 append 一个 len=2 的切片，那么这个时候就触发了扩容。
- （场景1）扩容之后就是 255 * 2 = 510；
- （场景2）现在有一个 256 的 slice 需要 append 一个 len=2 的切片，那么这个时候就触发了扩容。
- （场景2）扩容之后就是 256 + (256 + 256 * 3)/4 = 512；
- <mark>我们发现解决了上面旧版本的扩容不平滑的问题。</mark>

