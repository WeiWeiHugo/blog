---
slug: "01_environment"
title: "Golang - 環境建置 (Environment)"
date: "2022-01-15"
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
本篇基本上是說明golang的環境建置，編譯，環境先建置好，後續才能進行開發。另外會再提到一些程式碼的品質工具等。基本上在開發時都希望程式碼能具有較好的品質與一致的規範，降低後續維護的成本。

<!--more-->

## Install go
我自己的開發環境是在 macos 底下，基本上透過 `brew install go` 就會安裝完成。~~如果網路沒問題的話?~~
```sh
$ brew install go
```

在 Windows 環境下，可以透過 [Chocolatey](https://chocolatey.org/) 進行安裝，此外[官方網站](https://go.dev/dl/)有提供相關壓縮檔與安裝檔，挑選自己使用的平台下載相關檔案後，解壓縮或是進行安裝即可。

安裝完成後，可以透過 `go version`指令，確定是否安裝完成。
```sh
$ go version
go version go1.15.2 darwin/amd64
```
## First program 
建立一個檔案，通常第一支程式都會是設法在 Terminal 上顯示 Hello world!，故這程式碼的檔名先命名為 hello.go，並編輯該檔案。

```sh
$ touch hello.go
$ vim hello.go
```
程式碼大致如下。
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello world!")
}
```
儲存該檔案後，在 Terminal 執行下列指令，此時應該能看到 `Hello world!` 顯示在Terminal上。
```sh
$ go run hello.go
Hello world!
```
`go run` 指令會將程式碼在臨時目錄中編譯成 binary 後執行，執行完成後刪除這個檔案，若要編譯成 binary 並使用，則使用 `go build` 。
```sh
$ go build hello.go
```
此時會產生一個hello的檔案，執行該檔案一樣會看到 `Hello world!` 顯示在Terminal上。
```sh
$ ./hello
Hello world!
```

## Format
go 語言對於撰寫格式相當嚴格，須嚴格使用標準格式，雖然在開發上會稍有不適，但對於多人開發與維護時，固定格式的程式碼會使這些工作更加容易。
也因為格式有嚴格標準，go也提供一個開發工具，`go fmt`，這個工具會自動重新格式化程式碼使齊符合標準格式。且目前有 `go fmt` 的加強版 `goimports`，有新的且更好用的工具，那就用新的工具吧。~~喜新厭舊~~

嘗試一下，修改前面提到的 `hello.go`，將其改成下面的程式碼，改動不大，僅是刪除一行縮排。
```go
package main

import "fmt"

func main() {
fmt.Println("Hello world!")
}
```

安裝 goimports，在 Terminal 中使用下列指令下載 `goimports`。
```sh
$ go install golang.org/x/tools/cmd/goimports@latest
```

安裝完成後，使用以下指令，重新開啟該檔案會發現那行縮排被重新加了回去。代表在函式中，程式碼要縮排為嚴格標準格式的規範。
```sh
$ goimports -l -w .
```
* -l : 告訴goimports，將格式不正確的檔案顯示在 Terminal上。
* -w : 告訴goimports，直接修改該文件。
* .  : 路徑的意思，這邊用 . 代表目前的目錄與所有子目錄的所有檔案。

注意，盡量在編譯前執行 `go fmt` 或 `goimports` ，確保程式碼的格式沒有問題。

## Linting and Veting

`goimports` 能確保程式碼是大家慣用的格式，但在其他規範上則不會做檢查，像是變數命名規則，程式碼樣式與潛在錯誤等。高品質的程式碼請參考 [Effective Go](https://go.dev/doc/effective_go) 與 [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) 兩個網站。當然，也有相關的工具能處理這些問題，目前常見的有 `golint`、`go vet` 工具。

* `golint`，嘗試確保程式碼會依循文件，會建議更改像是變數名稱、public method 等，他的建議並不代表是錯誤，只是希望程式碼具有特定的格式並遵循特定的規則。
* `go vet`，可以檢測一些有效的但有可能存在錯誤的程式碼。像是將錯誤數量的參數傳遞給 method，或使用不恰當的 Function。

除了這兩個工具以外，另外還有許多第三方的工具可以檢查程式碼樣式與潛在錯誤，然而愈多的工具，在進行檢查時就會花費愈多時間。其中，`golangci-lint`結合了上述兩項與其他相關的程式碼品質工具。[golangci-lint 文件](https://github.com/golangci/golangci-lint)

且這個工具可以在開發目錄的根目錄中，透過一個 `.golangci.yml` 的檔案，根據需求設定啟用那些工具與檢查那些檔案。[.golangci.yml 文件](https://golangci-lint.run/usage/configuration/)
```sh
$ golangci-lint run
```

注意，一樣盡量在編譯前執行 `golangci-lint`，或是其他相關的工具，盡可能在編譯前找出錯誤或有疑慮的部分，確保程式碼的品質。

## Makefile
也就是說，在寫完程式後要進行編譯，我需要經常做這些事情：
```sh
$ goimport -l .
$ golangci-lint run
$ go run [targetPath]
### 或是
$ go build [targetPath]
```

**太麻煩了 !!!**

透過 `Make`、`shellscript` 或其他的腳本語言，可以省略掉許多手動的步驟，降低重複動作的時間。
但此篇文章說的是關於程式設計，我會傾向用 `make` 來設計：

建立一個檔案名為Makefile，並編輯該檔案：
```makefile
.DEFAULT_GOAL := build

fmt:
    goimports -l -w .
.PHONY:fmt

lint:
    golangci-lint run *.go
.PHONY:lint

build: lint
    go build hello.go
.PHONY:build
```

並使用根目錄下使用 `make` 指令。
```sh
$ make
```

若順利的話，則會依序進行 `goimports`、`golangci-lint` 與 `go build`，且不用再重複下多次指令，一個 `Make` 就足夠了，只是要做一些前置準備。(Makefile)，另外，因為這篇主要是說明 golang 的環境建置， Makefile 的說明以後有空會再另外寫一篇說明(?)

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216)(書籍)
2. [Effective Go](https://go.dev/doc/effective_go)
3. [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
4. [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)
5. [golangci-lint](https://github.com/golangci/golangci-lint)
6. [.golangci.yml](https://golangci-lint.run/usage/configuration/)












