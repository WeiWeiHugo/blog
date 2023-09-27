---
slug: "04_Blocks_Shadows_ControlStructure"
title: "Golang - 代碼塊、陰影與控制結構(Code Blocks, Shadows, and Control Structures)"
date: "2022-02-25"
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

本篇文章將介紹 Block 與 Shadows，接著會說明控制結構（if、for、goto）等。

<!--more-->
## Code Blocks

### Types of blocks
其實用 blocks 作為關鍵字去尋找相關資訊並不太好找，反倒是以 Code Blocks 會比較容易些，基本上有聲明的地方，都會被稱為 blocks，在任何函數以外聲明的常數、變數、類型與函數都是放在 package block 內。而使用 import 內的時statement時，這些名稱則是放在 file block。

在 golang 中，有四種 blocks：
* universe block，包含了整個 project 的 source code。
* package block，每一個 package 都有一個包含全部 code 的 block，但並不包含聲明。
* file block，每一個 file 都有一個包含全部 code 的 block，也包含了聲明。
* local blocks，基本上，在一個函數中，每一個大括弧 ```{}``` 都定義了一個 block。
local blocks 又分為 explicit local blocks 與 implicit local blocks，基本上要由 {} 與控制結構識別。

### Hierarchies
透過圖片，會更了解 block 之間的層次結構。
![](./blocks.png)

## Shadows
我們隨時可以從內部 block 訪問外部 block 的 identifer，如果內部與外部有相同的 identifer 時，會發生什麼事呢？

### Shadowing variables
```go
func main() {
    x := 10
    if x > 5 {
        fmt.Println(x)
        x := 5
        fmt.Println(x)
    }
    fmt.Println(x)
}
```
可以先想一下這個函式的 blocks，再來想一下這個程式會印出什麼。在這個函式中發生了內部與外部的 identifer 相同的情況，此時外部的 identifer 會被隱藏。印出來的結果會是：
```go
10
5
10
```
Shadowing Variable 是指在 package block 中的 variable 相同名稱的 variable，只要 Shadowsing variables 存在，就不能由外部訪問 Shadowing variables。該函式首先定義了 x 為 10，後來有一個 if statement，並遇到了大括號，記得前面說的每一個大括弧會是一個 block，這個 statement 內的 x 則是 shadowing variable，當離開這個 statement 時，這個 shadowing variable 就不能被訪問，故最後的 ```fmt.Println()``` 會是訪問頂部的 x 。

若善用 shadowing variables 在測試~~或是懶得想變數~~時會很方便，但記得使用的 identifer，若使用不慎可是會出問題的。
```go
func main() {
    x := 10
    fmt.Println(x)
    fmt := "oops"
    fmt.Println(fmt)
}
// Output
fmt.Println undefined (type string has no field or method Println)
```
這個例子中，fmt 被聲明為 local variable，故會隱藏原先 fmt 所擁有的函式，也因此會有錯個錯誤發生。

> 個人認爲在測試內使用 shadowing variable 即可，開發的專案就不建議。

### Detect
還記得在第一章提到的 vet 與 lint 可以做檢測用，但是 go vet 與 golangci-lint 是沒有檢測 shadow 的功能的(寫這篇文章時應該是沒有吧...)，但的確是有 shadow detect 工具可以使用的。
```shell
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
```

## Control Structures
Control Structures 其實跟許多語言的用法極為相似，但 golang 有一個與眾不同的 ```goto``` 可以使用。
### if
if 的用法也與其他語言的用法相似 ~~(很懶得寫)~~， if 的 condition 不用括號刮起來。
```go
n := rand.Intn(10)
if n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
```
主要提一點是，條件式內可以宣告變數的。
```go
if n := rand.Intn(10); n == 0 {
    fmt.Println("That's too low")
} else if n > 5 {
    fmt.Println("That's too big:", n)
} else {
    fmt.Println("That's a good number:", n)
}
```
在第一句的部分，我們新宣告了一個變數，後續的條件式則可以直接透過這個新的變數進行判斷。在某些場合會很好用。

但使用 if/else 時，請盡量將條件簡單化，另外是在 if statement 中宣告的變數，也是 shadowing variables，注意聲明的名稱與使用方式。

### for
for 基本上也與其他語言一樣，作為循環使用，在 golang 中，for 有四種使用方式
* 標準型 for
* 條件型 for
* 無窮型 for
* for-range

---
#### 標準型 for
```go
for i := 0; i < 10; i++{
    fmt.Println(i)
}
```
這就像許多語言的用法一樣，有一個初始值，一個需要滿足的條件式，跟一個會遞增或遞減的行為，這個行為會在每一次的迴圈結束後進行。

#### 條件型 for
```go
i := 0
for i < 100 {
        fmt.Println(i)
        i = i + 1
}
```
條件型的 for 省略了初始值與行為，但仍須保留條件式。

#### 無窮型 for
```go
for{
    fmt.Println("Hello!")
}
```
取消了條件，亦同這個 for 會一直滿足且運作下去，當然也可以使用標準型或是條件型 for 達到相同的目的。如果要程式運作時需要結束這個無窮型迴圈，設計時可以使用條件式與 ```break``` 跳出這個 for。
```go
i := 0
for{
    i = i + 1
    if i > 10 {
        break
    }
}
```

#### for-range
通常像其他語言的迭代器(iterator) ~~我以前學不好的部分~~， for-range 常用在 string, array, slice 和 map 上。
```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for i, v := range evenVals {
    fmt.Println(i, v)
}
// Output
0 2 
1 4 
2 6 
3 8 
4 10 
5 12
```
用 for-range 會得到兩個變量，一個通常被稱為index或是key，我在這邊稱為位置，另一個則是該位置的數值。

如果不需要位置時，可以使用下底線進行隱藏。
```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for _, v := range evenVals {
    fmt.Println(v)
}
// Output
2 
4 
6 
8 
10 
12
```

如果需要位置而不需要值時，使用下底線嗎？也可以，但可以直接省略。
```go
evenVals := []int{2, 4, 6, 8, 10, 12}
for i := range evenVals {
    fmt.Println(i)
}
// Output
0
1
2
3
4
5
```

for-range 在遍歷 map 上會有特別的地方，會有每一次遍歷的順序結果不同的情形，這是 go 語言為了安全的設計，每一次的 for-range 迭代在遍歷 map 會有不同的結果。

for-range 所遍歷的會是一個副本而不是原變數的值，故在這個副本內修改的，並不會影響到原本的數值。

### Continue
```continue``` 是用來跳過剩餘的部分，並進行下一次的迭代。用起來也與其他語言相似，但有一個特別的用法，透過 OUTER 標籤進行的 for。
```go
OUTER:
for _, item := range list.Items {
	for _, reserved := range reserved.Items {
		if reserved.ID == item.ID {
			continue OUTER
		}
		... do some other work ...
	}
	... do some other work ...
}
```
透過這種方式，可以跳出或跳過外部循環的迭代器。~~聽說這種方法很少用就是了~~

### switch
~~Nintendo Switch !!~~ ```switch```也是很常在其他語言中看到的 statement，用法也相似，很多人不喜歡使用 switch (我自己也是)，但 switch 在 go 語言中有一些令人驚訝的地方(?)

```go
words := []string{"a", "cow", "smile", "gopher","octopus", "anthropologist"}
for _, word := range words {
    switch size := len(word); size {
    case 1, 2, 3, 4:
        fmt.Println(word, "is a short word!")
    case 5:
        wordLen := len(word)
        fmt.Println(word, "is exactly the right length:", wordLen)
    case 6, 7, 8, 9:
    default:
        fmt.Println(word, "is a long word!")
    }
}
```
* 多個 case 要進行相同的邏輯，可以在同一個 case 寫多個條件。
* 每一個 case 都會是一個 block，像是 case 5 的 wordLen 是一個新的 variable，只能在這邊使用。
* 不必再每個 case 後面加上 break， go 只會進行符合的 case。
* 如果沒有滿足 case，不會發生任何事。或是使用 default 決定沒有滿足 case 時需要做什麼。

通常使用 break 代表要跳出這次的 switch。但與其他語言不同，在 case 底下使用 break，go 只會認為你想跳出該 case 而不是 switch，若需要跳出 switch 需搭配 label 做使用。使用方式如下列的程式碼：
```go
loop:
	for i := 0; i < 10; i++ {
		switch {
		case i%2 == 0:
			fmt.Println(i, "is even")
		case i%3 == 0:
			fmt.Println(i, "is divisible by 3 but not 2")
		case i%7 == 0:
			fmt.Println("exit the loop!")
			break loop
		default:
			fmt.Println(i, "is boring")
		}
	}
```

### goto
簡單介紹一下就好，如同字面上的意思，會去到程式的某個地方。但因為 goto 接近為所欲為，想去哪就去哪，在撰寫程式、運作與維護頗為麻煩 ~~(自己廢就說)~~，通常會被用在要跳過程式某些部分或是跳出迴圈、switch 等，並執行程式後面的部分時使用。
```go
func main() {
    a := rand.Intn(10)
    for a < 100 {
        if a%5 == 0 {
            goto done
        }
        a = a*2 + 1
    }
    fmt.Println("do something when the loop completes normally")
done:
    fmt.Println("do complicated stuff no matter why we left the loop")
    fmt.Println(a)
}
```
> tips : 非必要，盡量盡量不要使用 goto。

---
函式內可以提的內容應該都說明了，這章應該還有些東西可以補充，但可能之後想到比較容易解釋的方式，或是例子，再回來補充。

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [Code Blocks and Identifier Scopes](https://go101.org/article/blocks-and-scopes.html)
3. [Continue statements with Labels in Go (golang)](https://relistan.com/continue-statement-with-labels-in-go)
4. [Golang switch case 用法](https://matthung0807.blogspot.com/2021/06/go-switch-case.html)



