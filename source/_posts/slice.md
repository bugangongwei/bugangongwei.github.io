---
title: slice
date: 2021-08-20 00:00:00
tags: [GO]
categories: GO
comments: true
---

slice 的结构是:
````go
sliceHeader{
  Length int
  Capacity int
  ZerothElement *byte
}

var s []string 是一个 slice, 其实就是一个 sliceHeader{Length: 0, Capacity: 0, ZerothElement: nil} 结构体
````

string 的结构是 []byte
```go
// for i, v := range s 方式访问字符串, i 表示的是字节的下标, v 表示的是字符的(rune), 所以是按字符遍历;
// for i:=0;i<len(s);i++{} s[i] 表示的是一个字节
[]byte(s)
[]rune(s)
```

