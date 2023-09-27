---
slug: "10_Concurrency"
title: "Golang - 並發(Concurrency)"
date: "2022-03-23"
lastmod: "2022-03-24"
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
本篇文章將簡單介紹 Concurrency，並說明 goroutine, channel 和 select。

<!--more-->
## Concurrency
可能有聽過 “asynchronous”,“Parallelism” 或是 “threaded”，雖然很像但有點不太一樣。
主要可以解釋成，一個或多個 processes 同時發生的 process，好比說你正在看我的筆記，而其他人正在世界上做自己的事情，這些人同時與您存在。
~~但我認為一樣的都是，很難正確處理。~~

### 什麼時候使用
首先要確定，使用並發會讓效能變好再使用，因為並發並不是並行(Parallelism)，過多的並發只會導致程式難以理解，並發的數量也不會與效能成真正的正比關係。  
基本上程式的邏輯是：
1. 獲取數據
2. 計算
3. 輸出結果

基本上要使用並發，取決於數據如何通過程式的步驟流動，有時候多個步驟可以並發，因為他們沒有需要別的步驟執行後才能執行，反過來說，如果步驟是像串聯一般地執行，則不應使用並發。  
另外，如果並發運行的程式不會花費太多時間，也不推薦使用並發，因為硬體資源並不是免費的，如果不確定使用並發會不會提升效能，可以透過寫測試的方式驗證。

## Goroutine
Goroutine 是 Go 在運行時管理的輕量化 processes。  
當程式啟動時，運行中會建立許多 thread 並運行一個 goroutine 來運行我們所寫的程式。  
程式所建立的所有 goroutine（包含一開始的），都會由 go 去調度並分配給 thread，類似我們設計跨 CPU kernel 的程式。  
但作業系統已經能做到這件事，為什麼 Go 還要再實現類似的機制呢？
* Goroutine 建立的比 thread 還快。
* Goroutine 初始化 stack 比 thread stack 還小，代表相同的記憶體空間下可以放更多的 goroutine。
* Goroutine 之間切換的速度比 thread 還快，因為 Goroutine 完全發生在 process 內。
* Go 有 scheduler 能做最佳化。

### 如何使用
我們在函數前面輸入 `go` 就能夠啟動了，只是這個函式的返回值會被忽略。  
任何函式都能夠這樣子啟用並發，但通常會在 closure 內執行，這樣會使程式較容易測試與模組化，並使得 API 不具有並發性。
```go
// 我們要並發的函式
func process(val int) int {
    // do something with val
}
// 透過一個函式去呼叫
func runThingConcurrently(in <-chan int, out chan<- int) {
    // closure
    go func() {
        for val := range in {
            result := process(val)
            out <- result
        }
    }()
}
```

## Channel
goroutine 透過 channel 進行溝通。  
聲明 `chan` 後透過 `make` 實現：
```go
ch := make(chan int)
```
channel 是參考類型(reference)，與 map, slice一樣，傳遞時是傳遞 channel 的 pointer。 channel 的 zero value 是 `nil`。

### 讀取與寫入
透過一個 `<-`運算符進行讀取與寫入的動作，與C++的 `->` 是相反過來的。
```go
a := <-ch // reads a value from ch and assigns it to a
ch <- b   // write the value in b to ch
```

另外我們也可以用 `for-range` 從 channel 中讀取，這種讀取方式，會一直持續到 channel 關閉，或是遇到 `break` 與 `return` 才會結束。
```go
for v := range ch {
    fmt.Println(v)
}
```

每個被寫入 channel 的數值只能被讀取一次。如果有多個 goroutine 從同一個 channel 讀取，寫入 channel 的數值只會被其中一個 goroutine 讀取。  

Channel 也可以設定為單向讀取或是單向寫入：
```go
var writeOnly chan <- int = ch1
var readOnly <- chan int = ch2
```

Channel 預設是無緩衝的，每一次寫入了一個開放的無緩衝 channel，都會導致寫入 goroutine 暫停，直到另一個 goroutine 從這個 channel 讀取，反過來說，每一次讀取這個 channel，都會導致讀取 goroutine 暫停，一直到有一個 goroutine 寫入這個 channel，也就是說至少要有兩個 goroutine 才能寫入與讀取 channel。  

Go 也有緩衝的 channel，如果在讀取這個 channel 前， channel 的緩衝區已經滿了，則後面的寫入將會暫停寫入，一直到讀取這個 channel 為止。~~就已經滿了你不給他空間不然你是想怎樣！？~~

有緩衝區的 channel 怎麼實現呢？
```go
ch := make(chan int, 10)
```

最後，大部分的時候會建議使用無緩衝區的 channel。

### 關閉
關閉，就是使用，`close`。
```go
close(ch)
```
這樣就能夠關閉 channel，此時再寫入這個 channel 或是再次關閉都會造成 panic，但關閉後的 channel 是可以讀取的，若裡面還有未讀取的值，則會依序返回，如果沒有未讀取的值，則會返回 zero value。

但這樣當我們讀取到 zero value 時，我們要怎麼知道，這個 zero value 是被關閉的 channel 的，還是還沒讀取的值？ ~~偷看第三篇~~

```go
v, ok := <-ch
```

## select
select 是並發的控制結構。  
透過 select 可以允許 goroutine 讀取或寫入一組多個 channel 中的一個，他用起來很像 `switch`。
```go
select {
case v := <-ch1:
    fmt.Println(v)
case ch2 <- x:
    fmt.Println("wrote", x)
case <-ch3:
    fmt.Println("got value on ch4, but ignored it")
}
```
如果有多個 case 可以讀取或寫入的話，會發生什麼？ Go 的設計上會隨機選擇一個進行，可以防止飢餓的問題發生，另外也可以預防一種 deadlock 的發生：acquiring locks in an inconsistent order。

因為 select 用來負責讓多個 channel 進行聯繫，因此會用在一個 `for-loop` 中，這裡也被稱為 `for-select` 循環，使用 `for-select` 記得必須要有退出的方法。
```go
for{
    select {
    case v := <-ch1:
        fmt.Println(v)
    case ch2 <- x:
        fmt.Println("wrote", x)
    case <-ch3:
        fmt.Println("got value on ch4, but ignored it")
    }
    case <-done:
        return
}
```

## Remenber
* 保持 API 沒有並發：
前面也提到了，盡量在 closure 內設計並發，並盡量隱藏，當使用者知道可以使用 API 來執行並發時。摁...
* 每當您的 goroutine 使用的數值是可能會改變的變量時，請將變量的當前值傳遞給 goroutine。  
```go
for _, v := range a {
    go func(val int) {
        ch <- val * 2
    }(v)
}
// 下面這個就沒傳值，goroutine內的v會是迴圈結束後的v值
for _, v := range a {
    go func() {
        ch <- v * 2
    }()
}
```
* 清理與關閉 goroutine：  
如果一個 goroutine 沒有退出，scheduler仍然會定期給它時間，但他可能什麼都不做，這會降低程式的效能。
* 何時使用緩衝與非緩衝 channel：
當知道已經啟動了多少個 goroutine，想要限制將啟動的 goroutine 的數量，或者想要限制排隊的工作量時，可以使用緩衝 channel。  


## 備註
還有一些像是 Backpressure, sync.Waitgroup, sync.Once 跟 Mutex 的作用與用法想做筆記，但一時之間還想不到該怎麼寫，可能等之後有時間再來補充。

更多關於 Concurrency 的資訊，可以參考 Katherine Cox-Buday 所寫的 Concurrency in Go。（歐萊禮也有這本所以...又是坑）

另外，後續還會有一篇 Context 的筆記，但我會想看完 Concurrency in Go 再來寫筆記分享。

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [Concurrency in Go](https://www.amazon.com/Concurrency-Go-Tools-Techniques-Developers/dp/1491941197) (書籍)
3. [Concurrency與Parallelism的不同之處](https://medium.com/mr-efacani-teatime/concurrency%E8%88%87parallelism%E7%9A%84%E4%B8%8D%E5%90%8C%E4%B9%8B%E8%99%95-1b212a020e30)

