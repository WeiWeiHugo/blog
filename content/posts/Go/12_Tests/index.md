---
slug: "12_Tests"
title: "Golang - 測試(Tests)"
date: "2022-03-25"
lastmod: "2022-03-26"
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

簡單說明如何進行程式碼測試，介紹檢查代碼覆蓋率與編寫基準測試。

<!--more-->

## Basic
測試分成兩個部分：
* package : `testing`
* tool : `go test`

`testing` 提供了測試的類型與函數，而 `go test` 則是將相關工具綁在一起進行測試後產生報告。  
另外， Go 的測試與程式碼會放在同一個目錄，或是同一個 package 當中，也因為在同一個 package 中，所以可以測試未導出的變數與函數。

一個簡單的測試，先寫好一個要被測試的函數，並存放在 adder/adder.go 中：
```go
package adder

func addNumbers(x, y int) int {
	return x + x
}

```
每個測試都寫在一個以 _test.go 結尾的檔案中。  
因此，測試會放在 adder/adder_test.go：
```go
package adder

import (
	"testing"
)
func Test_Adder(t *testing.T){
	result := addNumbers(2,3)
	if result != 5{
		t.Error("incorrect result")
	}
}
```

測試函數開頭，會以 `Test` 為開頭，並傳入 `*testing.T`。  
測試函數不會返回任何數值。這個測試中會使用 `t.Error` 返回錯誤訊息，用法與 `fmt.Println` 類似。  
完成之後使用 `go test` 進行測試：
```shell
$ go test 
PASS 
ok test_examples/adder 0.006s
```

### 測試失敗
剛剛使用的 `t.Error` 是報告的其中一個方法，也可使用 `t.Errorf` 格式化輸出。  
另外也可以使用 `t.Fatal`或是 `t.Fatalf`，它會在錯誤發生時，中斷測試，而使用 `t.Error` 則不會。  
看自己要怎麼應用測試而使用。

## TestMain
有時候在測試時，要先設定一些環境或變數再開始進行測試，透過 `TestMain` 可以讓我們在測試前或測試後做一些事情，像是我要從 database 或是外部撈數據之後，透過這些事情的結果再進行測試。
```go
var testTime time.Time

func TestMain(m *testing.M) {
	fmt.Println("Set up stuff for tests here")
	testTime = time.Now()
	exitVal := m.Run()
	fmt.Println("Clean up stuff after tests here")
	os.Exit(exitVal)
}

func TestFirst(t *testing.T) {
	fmt.Println("TestFirst uses stuff set up in TestMain", testTime)
}

func TestSecond(t *testing.T) {
	fmt.Println("TestSecond also uses stuff set up in TestMain", testTime)
}
```
我們建立了一個 TestMain，在裡面初始化了需要使用的變數，完成後使用 `Run` 去執行 test，這時就會開始進行測試。  
要注意一點是，最後必須從 `Run` 裡面使用 `os.Exit` 來結束測試。
TestMain 他不會單獨的在測試函式前後執行，另外要注意的是，每個 package 內只能有一個 TestMain。

### testdata
Go 有保留這個目錄名稱，作為存放初始測試資料的目錄，從這個目錄存取檔案時，請使用相對路徑。

### 測試 public api
前面我們提到過，函數的大小寫字首決定他是否為 public，如果我們只想測試 public api，透過下列的方法即可，一樣會以前面的 adder 為例子。  
`adder.go`
```go
package adder
// public : 字首大寫
func AddNumbers(x, y int) int {
	return x + x
}
```
`adder_test.go`
```go
// Package 的名稱後面加上 _test
package adder_test	

import (
	"testing"
	// 導入 adder，使用時注意路徑
	"adder"
)
func Test_Adder(t *testing.T){
	result := adder.AddNumbers(2,3)
	if result != 5{
		t.Error("incorrect result")
	}
}
```
## Code Coverage
Code Coverage 可以幫助我們了解，程式中是否有遺漏明顯的地方，雖然覆蓋率 100% 不代表程式運作 100% 正常，但至少可以降低錯誤發生的機率(吧)  
透過 `go test -v -cover` 可以顯示覆蓋率，如果再加上 `-coverprofile=[filename]` 可以把訊息輸出。
```shell
$ go test -v -cover -coverprofile=test.out
```

輸出後可以用 html 的方式開啟，會告訴你哪邊有被覆蓋，哪邊尚未被覆蓋。
```shell
$ go tool cover -html=test.out
```
{{<image src="cover.png" width="100%">}}

## Benchmarks
當我們的程式都沒問題了之後，下一步要測試的可能是程式的快慢，Go 提供了 `test_examples/bench` 讓我們可以進行一些基準測試：
* 要以 `Benchmark` 作為開頭。
* 接收一個 `*testing.B` 類型的參數
```go
func BenchmarkSay(b *testing.B){
	for i:= 0; i< b.N; i++ {
		adder.Say("Hi")
	}
}
```
每個 Go 基準測試都必須有一個從 0 迭代到 b.N，測試框架一遍又一遍地用越來越大的 N 值調用我們的基準函數，直到確定計時結果準確為止。  
測試時使用 `-bench=.` 執行所有的基準測試，再加上 `-benchmem` 會顯示記憶體的資訊。  
如果測試程式中有一般的測試(*testing.T)與基準測試，則會先執行一般的測試再進行基準測試。
```shell
$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
pkg: adder
BenchmarkSay-4   	       2	 512890848 ns/op	      48 B/op	       1 allocs/op
PASS
ok  	adder	1.555s
```

* BenchmarkSay-4 : 名稱
* 2 : 測試運行已產生穩定結果的次數
* 512890848 ns/op : 執行一次測試需要花費的時間
* 48 B/op : 每次測試使用的記憶體空間
* 1 allocs/op : 每次測試發生了多少由 heap 分配的次數。

## Stubs
因為函數之間通常會有依賴關係，在測試中可能會因為某個函數需要前一個動作處理後的結果，才能進行測試，比方說，我需要從 database 取得資料，故我要先與 database 進行連線，但我只是要測試取得資料這個函式而已，能不能不要與 database 進行連線？  
在前面我們提到了 function type 與 interface type，我們可以透過這兩種類型降低測試時的函式依賴。  

**範例待補**

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)


