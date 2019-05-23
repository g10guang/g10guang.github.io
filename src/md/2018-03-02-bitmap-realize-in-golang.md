---
layout: post
title:  "bitmap 位图的 Go 实现"
date:   2018-03-2 12:00:05 +0800
categories: 算法
---

# bitmap 实现

完成《Go程序设计语言》的一个练习，感觉位图在算法中是一个挺重要的，特别是在《编程珠玑》中提到位图在当初内存昂贵，计算资源匮乏的时代的使用。

```go
import (
    "bytes"
    "fmt"
)

// 用于存储数字的集和
type IntSet struct {
    words  []uint64
    length int
}

func (s *IntSet) Has(x int) bool {
    word, bit := x/64, uint(x%64)
    return word < len(s.words) && (s.words[word]&(1<<bit)) != 0
}

func (s *IntSet) Add(x int) {
    word, bit := x/64, uint(x%64)
    for word >= len(s.words) {
        s.words = append(s.words, 0)
    }
    // 判断 x 是否已经存在 s 中
    if s.words[word]&(1<<bit) == 0 {
        s.words[word] |= 1 << bit
        s.length++
    }
}

func (s *IntSet) UnionWith(t *IntSet) {
    originLenOfSet := len(s.words)
    for i, v := range t.words {
        if i < originLenOfSet {
            if v != 0 {
                for j := uint(0); j < 64; j++ {
                    if s.words[i]&(1<<j) == 0 && (v&(1<<j) != 0) {
                        s.length++
                    }
                }
                s.words[i] |= v
            }
        } else {
            s.words = append(s.words, v)
            for j := uint(0); j < 64; j++ {
                if v&(1<<j) != 0 {
                    s.length++
                }
            }
        }
    }
}

func (s *IntSet) String() string {
    var buf bytes.Buffer
    buf.WriteByte('{')
    for i, v := range s.words {
        if v == 0 {
            continue
        }
        for j := uint(0); j < 64; j++ {
            if v&(1<<j) != 0 {
                if buf.Len() > len("{") {
                    buf.WriteByte(' ')
                }
                fmt.Fprintf(&buf, "%d", 64*uint(i)+j)
            }
        }
    }
    buf.WriteByte('}')
    fmt.Fprintf(&buf, "\nLength: %d", s.length)
    return buf.String()
}

func (s *IntSet) Remove(x int) {
    word, bit := x/64, uint(x%64)
    if word < len(s.words) {
        if s.words[word] & (1 << bit) != 0 {
            s.words[word] &= ^(1 << bit)
            s.length--
        }
    }
}

func (s *IntSet) Len() int {
    return s.length
}

func (s *IntSet) Clear() {
    s.words = nil
    s.length = 0
}

func (s *IntSet) Copy() (cp *IntSet) {
    cp = new(IntSet)
    cp.words = make([]uint64, len(s.words))
    cp.length = s.length
    copy(cp.words, s.words)
    return cp
}
```