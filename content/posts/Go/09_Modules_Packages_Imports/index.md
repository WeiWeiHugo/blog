---
slug: "09_Modules_Packages_Imports"
title: "Golang - 模組、包與導入(Modules, Packages, and Imports)"
date: "2022-03-23"
lastmod: "2022-03-23"
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

本篇文章會介紹如何使用 module 與 package 組織程式碼，並如何 import。

<!--more-->

## Repositories, Modules, and Packages
Repo 大家應該都很熟悉，版本控制中儲存程式碼的地方。  
Modules 像是程式碼或應用程式的根目錄，包括一個或多個 package ，存放在 Repo 中。  
如果這個根目錄有 go.mod，則代表這些程式碼的集合為一個 module，這個檔案請由指令去形成：
```shell
$ go mod init <MODULE_PATH>
```
裡面則會有 module 的名稱， go 的版本，需要哪些 module 等等的資訊。

## Import
import 可以讓我們使用另一個 package 的 constants, variables, functions 與 types，另外 identifier 是大寫開頭的才能由外部存取，如果是小寫開頭的則只能於內部進行存取，我們很常用的 `fmt.Println()` 就是一個好例子。

## Package

### Create Package
透過 package clause 實現，package clause 都會在檔案的第一行，且是非空白的非注釋行。
```go
package test

func Plus(a int, b int) int {
    return a+b
}
```
### Access Package
當我們需要使用自己建立的 package 時，透過前面提到的 import 做使用，如果你不是使用 standard library 內的 package，使用時要注意 package 的 path。
```go
package main

import {
    "fmt"
    "example/plus"
}
func main() {
    num = test.Plus(1,2)
    fmt.Println(num)
}
// Output
3
```

另外有一些規則要注意：
* import 是要使用路徑，而不是 import package clause，我自己的話會想說把 package clause 與檔案名稱一致會比較好管理。除了某些情況是不需要一致的。
* 不要用 main 做 import， main 是程式運作的起點。
* 不要用一些特別的字元在 path 上。

### 重複 import
有時候會有 import 相同 package clause 的情形發生，例如 `crypto/rand` 與 `math/rand`，這樣我們使用 rand 時會有問題。  
Go 可以讓我們 import 的 package 有別的名稱，但在維護上可能會有些問題，因為要去了解他是用別名的方式，還是直接 import 進來的。
```go
package main

import{
    crand "crypto/rand"
    "math/rand"
}
func main(){
    // Use crand ...
}
```

### godoc
作為 package 的文件使用，其實就很像在 linux 有疑問時會用 `man` 或是 `-H, --help` 一樣。  
基本上是程式的註解，沒有特殊的格式與符號。  
但有一些規則要注意一下：
* 將註解放在對象的前面(上一行)，之間並沒有空白行。
* 以兩個雙斜線開頭，後面要先放對象的名稱。
* 透過空白行分段。
* 透過縮排進行註解的格式化。
* package clause 則是要以 package clause 作為開頭，  
function 與 struct 則以名稱開頭即可。

可以實際觀察看看註解要如何撰寫。
```shell
$ go doc fmt
```
另外。在第一篇提到的 `golangci-lint` 可以幫助找出缺失註解的地方。

## Modules
透過前面提到的 `go mod`，我們會得到一個 go.mod 的檔案，裡面可能會紀錄 package 的版本資訊。另外編譯過後，會有一個 go.sum 的檔案，裡面記錄了依賴 package 的 hash。

### 版本控制
我們可以透過 `go list` ，觀察 module 的版本，並透過 `go get` 取得較舊的版本。  

```shell
$ go list -m -versions github.com/learning-go-book/simpletax
github.com/learning-go-book/simpletax v1.0.0 v1.1.0
[!] 有v1.0.0 與 v.1.1.0，我想用舊版本...

$ go get github.com/learning-go-book/simpletax@v1.0.0
[!] 此時回去看 go.mod 會發現版本已經改變了。
```
{{< admonition tips "Hint">}}
版本號的規則：major.minor.patch  
修復錯誤時，patch 更新，增加新功能時，minor 更新並將 patch 歸零。
{{</ admonition >}}

### Vendoring
為了確保從頭到尾都能使用相同的依賴項開發，透過 `go mod vendor` 可以產生一個目錄，保存所有依賴項的副本。  
如果又增加了新的依賴項，或是升級了依賴項，都要再執行一次 `go mod vendor`，否則會無法編譯。

## 補充
[pkg.go.dev](https://pkg.go.dev) 會自動索引開源的 Go project。

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
