---
slug: "13_Reflect_Unsafe_Cgo"
title: "Golang - 反射，不安全與Cgo (Reflect, Unsafe, and Cgo)"
date: "2022-03-26"
lastmod: "2022-03-27"
tags: ["go","golang"]
sidebar: auto 
categories: ["Go"]
no_comments: false
draft : false
resources:
- name: "featured-image-preview"
  src: "go_header.png"
- name: "featured-image"
  src: "go_header.png"
---

簡單說明 reflect, unsafe 與 cgo。

<!--more-->

## Reflect
Go 是靜態強型態語言，因此類型是 Go 很重要的一個部分。  
但有些時候我們會不清楚該資料的型態，或是想設計支援多個類型的函式，Go 提供了 `Reflect` 讓我們在程式運作中檢查類型，甚至能進一步修改與建立函式、結構的能力。  
一個很經典的例子： `fmt.Println`

### TypeOf
我們可以用 TypeOf 取得類型的名稱，但像 slice 或是 map 等指標類型，會回傳一個空字串 `""`。
```go
var v int
vType := reflect.TypeOf(v) 
fmt.Println(vType.Name()) // Output int
```
`Kind` 會回傳類型是由什麼組成的，通常用在 slice 或是 map 等指標類型。  
有些類型引用了其他類型，我們可以透過 `Elem` 找出被引用的類型是什麼：
```go
var v int 
vType := reflect.TypeOf(&v) 
fmt.Println(vType.Name()) 
fmt.Println(vType.Kind())
fmt.Println(vType.Elem().Name())
fmt.Println(vType.Elem().Kind())
```

reflect 也可以反映結構，透過 `NumField` 取得架構內部的字段個數，並透過 `Field` 取得架構內的字段，如果架構內有 tag 的話，可以透過 `Get` 取得 tag 的資訊：
```go
type Foo struct {
    A int    `myTag:"value"`
    B string `myTag:"value2"`
}

var f Foo
ft := reflect.TypeOf(f)
for i := 0; i < ft.NumField(); i++ {
    curField := ft.Field(i)
    fmt.Println(curField.Name, curField.Type.Name(),
        curField.Tag.Get("myTag"))
}
```

### Value
我們可以透過 `reflect.ValueOf` 實現 `reflect.Value`:
```go
vValue := reflect.ValueOf(v)
```
透過這個與 `Set`，我們也可以用來設定變數的數值：
```go
i := 10
iv := reflect.ValueOf(&i)
ivv := iv.Elem()
ivv.SetInt(20)
```

### New
New 透過 Type 返回一個指定類型的 pointer，我們可以透過修改這個 pointer 與使用 Interface 將修改過後的數值給一個變數：

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    a := 1
    intPtr := reflect.New(reflect.TypeOf(a))
	intPtr.Elem().SetInt(2)
    b := intPtr.Elem().Interface().(int)
    fmt.Println(b)
}
```

## Unsafe
unsafe，可以讓我們操作記憶體，反過來說，操作記憶體不安全，所以叫做 unsafe。  
為何要使用 unsafe 呢？
* 許多 package 內使用過。
* 大部分是要操作系統而使用。
* 使用 unsafe 可以有更好的 performance。

基本上有三種函數與一種類型：
* SizeOf : 回傳變數所使用的記憶體大小
* AlignOf : 回傳在記憶體中進行記憶體對齊需要的倍數。
* OffsetOf : 回傳兩個變數之間的位址差距。
* unsafe.pointer : 任何類型的 pointer 都可以轉換為 unsafe.Pointer

## cgo
C 語言已經有很久的歷史，但至今仍是重要的程式語言之一。許多的作業系統是以C/C++實現的，這也意味著許多程式語言都提供了使用 C 的函式庫之方法（FFI）。  Go 語言將這個外部函數介面稱為 cgo。

基本上要先 `import "C"`，這行會讓 Go 編譯前先運行 cgo。cgo 會運行與前面有關的註解。並將 C 的程式碼( .c, .h )放置到同一個目錄中，另外要安裝 C 的編譯器，最後再透過 `go build `即可。
```go
package main

import "fmt"

/*
    #cgo LDFLAGS: -lm
    #include <stdio.h>
    #include <math.h>
    #include "mylib.h"

    int add(int a, int b) {
        int sum = a + b;
        printf("a: %d, b: %d, sum %d\n", a, b, sum);
        return sum;
    }
*/
import "C"

func main() {
    sum := C.add(3, 2)
    fmt.Println(sum)
    fmt.Println(C.sqrt(100))
    fmt.Println(C.multiply(10, 20))
}
```

不過因為兩種語言的不同之處，cgo 在使用上有極大機率造成效能降低：
* Go 會收集 garbage，但 C 不會。
* 一些以 pointer 傳遞的類型無法在兩種語言之間運作。
* 部分 C library 不支援(e.g. printf)。
除非是已經有一個很好的 C 函式庫需要使用，不然不建議使用。


## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [StackOverFlow - How do you create a new instance of a struct from its type at run time in Go?](https://stackoverflow.com/questions/7850140/how-do-you-create-a-new-instance-of-a-struct-from-its-type-at-run-time-in-go)

