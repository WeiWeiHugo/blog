---
slug: "06_Pointer"
title: "Golang - 指標(Pointer)"
date: "2022-03-07"
lastmod: "2022-03-10"
tags: ["go","golang"]
sidebar: auto 
categories: ["Go"]
no_comments: false
resources:
- name: "featured-image-preview"
  src: "go_header.png"
- name: "featured-image"
  src: "go_header.png"
---

本篇文章會簡單介紹指標，並學習如何使用指標與記憶體空間，使程式執行上能更有效能。

<!--more-->

## Pointer 
### 簡介
***學習 pointer 的第一條規則 ： 不要害怕 ！！***

pointer 是一種 variable，他的內容是儲存另一個 variable 的 address，Address 則是每一個 variable 儲存在一個或多個連續的記憶體位置。

不同類型的 variable 會佔用不同的記憶體空間，像是 bool 只要一個 byte 就能代表 true 或 false，(因為可以獨立尋找 address 的最小空間是 byte)，而 int32 需要 4 個 byte 的儲存空間。

雖然不同類型的 variable 可以佔用不同的記憶體空間，但每個 pointer 無論指向任何類型的 variable，都會是相同的大小。

```go
var x int32 = 10
var y bool = true
pointerX := &x
pointerY := &y
var pointerZ *string
```

* pointerX 會是 x 的 address
* pointerY 會是 y 的 address
* pointerZ 不會指向任何東西，Value 會是 Zero value。

而 pointer 的 Zero value 是 ```nil``` 而不是 0，與 C 語言的 null 不同， ```nil``` 不代表 0，故不能將 nil 轉換成 0。

### 運算符
基本上 golang 的 pointer 運算符與 C/C++ 的相似。

```&``` : address operator，用在變數前的話，會返回該變數的 address。

```*``` : indirection operator，用於指針變數，會返回該 pointer 指向的 value。也被稱為 dereferencing。
```go
x := 5
pointerX := &x
y := 5 + *pointerX
fmt.Println(y) // 10
```

dereferencing 要確保 pointer 不是 ```nil```，否則會造成 panic。
```go
var x *int
fmt.Println(*x) // panic
```

### 指標類型
其實就是有一種類型叫做指標類型啦。用來表示 pointer，基本上可以基於任何類型。
```go
x := 10
var pointerX *int
pointerX = &x
```

另外透過 ```new``` 聲明一個指針變數時，他的初始值會是 0 而不是 ```nil```。
```go
var x = new(int)
fmt.Println(*x)  // prints 0
```

## 傳遞(call)
### 簡介
當原始值分配給另一個變數或傳遞給 function or method 時，對其他變數所做的任何更改都不會反映在原始值中。
```Java
// Java
int x = 10;
int y = x;
y = 20;
System.out.println(x); // prints 10
```
然而在透過類型所建立的 instance，分配給另一個 instance 或傳遞給 function or method 時，又會是不同的結果。
``` Python
// python
class Foo:
    def __init__(self, x):
        self.x = x


def outer():
    f = Foo(10)
    inner1(f)
    print(f.x)
    inner2(f)
    print(f.x)
    g = None
    inner2(g)
    print(g is None)


def inner1(f):
    f.x = 20


def inner2(f):
    f = Foo(30)


outer()
// Output
20
20
True
```

在許多程式語言中(e.g. Java, Python)，會有以下特性：
* 如果將 instance 傳遞給函式並更改 field 的值，則這次更改會反映在傳遞進去的 instance。
* 如果重新分配參數，則更改不會反應在傳遞進去的變數。
* 如果傳遞 `nil/null/None` 等參考值，會將傳入參數本身設定為新的數值，而不會影響原先原先的數值。

因為這些語言的 instance 是透過 pointer 實現的，當 instance 傳遞給 function 或是 method 時，被複製的數值是指向該 instance 的 pointer。在 `inner1` 是指向相同的 address，而在 `inner2` 則是建立一個新的 instance，會指向不同的 address，因此不會影響到原先傳入的 instance。

基本上在 golang 中，會有一樣的特性，但 golang 不同的是可以對原始類型與架構使用 pointer 或是 value。

### 傳遞 pointer
上篇文章提到，golang 是 call by value，但可以透過將 pointer 傳遞給函式的方式，使原始變數被函式進行修改。只不過有幾點是要注意的。

如果傳遞 `nil` pointer，則不能將數值修改為非 `nil`。如果已經將該 pointer 分配了一個數值，則只能 reassign 這個數值。因為 call by value 的關係，會複製一份 pointer 變數，並在函式內處理，而原先的 pointer 當然就不會被函式修改到。
```go
func failedUpdate(g *int) {
    x := 10
    g = &x
}

func main() {
    var f *int // f is nil
    failedUpdate(f)
    fmt.Println(f) // prints nil
}
```

如果希望 pointer 參數傳入後修改的值在退出函式時仍然存在，不要在函式內建立一個新的變數並透過修改 pointer 來修改，而是透過 pointer 指向要修改的數值並進行修改。
```go
func failedUpdate(px *int) {
    x2 := 20
    px = &x2
}

func update(px *int) {
    *px = 20
}

func main() {
    x := 10
    failedUpdate(&x)
    fmt.Println(x) // prints 10
    update(&x)
    fmt.Println(x) // prints 20
}
```

### 小心使用
基本上在 golang 是不建議使用 pointer，會影響數據傳遞的理解，除了在某些情況下使用會好些，例如在使用函式時需要一個 interface、部分類型需要以指針傳遞、數據類型中存在需要修改的狀態，會建議返回值使用指針類型。

### 效能
不過，pointer 的好處是無論任何的類型，pointer 都會是一樣的大小，也就是說當我們傳遞大的數值給函式時，花費的時間也會較多，但傳遞 pointer 則會減少原先所需傳遞的時間。

### Slice
上一篇我們提到，Map 與 Slice 是以 pointer 實現的，故直接傳遞 Map 與 Slice 給函式是可以直接修改數值的，這邊提一下 Slice ，在函式內雖然可以修改內部數值，但不能修改 slice 的容量。

Slice 基本上為三個單元所組成：
* 資料，以 pointer 指向某一段記憶體空間。
* 長度，使用了多少空間。
* 容量，這個 slice 總共有多大。

因為 call by value 的關係，傳遞至函式內的 slice 會被複製，因為資料是以 pointer 指向某一段記憶體空間，所以可以直接修改，但容量部分只會修改函式內被複製的這份 slice，因此原先的 slice 並不會被修改到容量。

另外，slice 也很適合作為 buffer 使用，後面的文章會再提起 slice。

## Garbage Collector
Garbage 是指沒有 pointer 指向該筆數據，則代表該數據存在記憶體空間中但沒有做使用，該數據就是 Garbage，這些空間就會被拿來重新使用，避免記憶體使用量一直增加。

Golang 有 Garbage Collector，但不代表可以隨性的使用記憶體空間。因為 Garbage Collector 也是需要資源去處理這些 Garbage，會降低程式的效能。

另外， Golang 會將原始類型、結構等放至 stack 中，並順序排列，透過建立最少的 Garbage，達到低延遲的目標。

寫著寫著發現 Stack, Garbage Collector 是個坑，之後有空可能會另外寫一篇出來吧。

後面會附上一些連結，有興趣的話可以參考。

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [Getting to Go: The Journey of Go's Garbage Collector](https://go.dev/blog/ismmkeynote)
3. [Memory Bandwidth Napkin Math](https://www.forrestthewoods.com/blog/memory-bandwidth-napkin-math/)
4. [Language Mechanics On Stacks And Pointers](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html)
5. [Allocation efficiency in high-performance Go services](https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/)