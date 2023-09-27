---
slug: "08_Error"
title: "Golang - 錯誤(Error)"
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

本篇文章會介紹如何在 Go 中處理錯誤，並簡單提一下 panic 與 recover。

<!--more-->

## 前言
基本上這本書已經讀完一半了，給自己個小掌聲！  
但因為近期有些事情要處理，想先把這本書的筆記做完，把這邊的時間給空出來。  
如果想說以後再回來寫，依照我的個性應該是會懶得寫...
近期應該會大量生產一堆品質低下的筆記。 ~~(其實本來就沒多好)~~

## 錯誤
Go 會透過 return 一個 error type 的數值，作為函數的最後一個返回值。如果函式正常運作並返回數值，則會返回 `nil`，如果發生錯誤，當然就會返回錯誤值。
```go
func calcRemainderAndMod(numerator, denominator int) (int, int, error) {
    if denominator == 0 {
        return 0, 0, errors.New("denominator is 0")
    }
    return numerator / denominator, numerator % denominator, nil
}
```
一些小原則：
* 錯誤訊息不應該大寫。
* 不應該以標點符號結尾。

另外， Go 沒有特殊的方法去檢測，是否返回了錯誤，但可以用判斷式去判斷：
```go
if err != nil {
    // TODO
}
```

------

Go 使用返回錯誤的方式設計，而不是使用跳出異常(Exception)，一來是異常處理的方式有時會無法容易掌握，二來是程式碼遇到錯誤時可能不會崩潰，但數值會未正確初始化，修改等。

基本上我們可以使用 `error.New()` 與 `fmt.Errorf()` 兩種方式，透過  string 處理簡單的錯誤。
```go
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, errors.New("only even numbers are processed")
    }
    return i * 2, nil
}
// OR
func doubleEven(i int) (int, error) {
    if i % 2 != 0 {
        return 0, fmt.Errorf("%d isn't an even number", i)
    }
    return i * 2, nil
}
```
### Sentinel Errors
代表用一個特定值，來表示一個不能進一步處理的做法。
```go
func main() {
    data := []byte("This is not a zip file")
    notAZipFile := bytes.NewReader(data)
    _, err := zip.NewReader(notAZipFile, int64(len(data)))
    if err == zip.ErrFormat {
        fmt.Println("Told you so")
    }
}
```
這個程式代表要讀取 zip 格式，但傳入的參數並不是 zip，因此發生了錯誤，透過一個 `ErrFormat`代表傳入格式不正確時會發生的錯誤，然而當錯誤太多種時，則需要一個一個去定義錯誤的特定值並比對，撇開麻煩不說，當錯誤比對時，該錯誤沒有在自己定義的比對值，這樣會發生問題。

除了一些極端情況，不然應該會很少用 Sentinel Errors。

### Error structure
`error` 是一個內置的 interface：
```go
type error interface{
    Error() string
}
```

所以可以透過這個 interface ，自己定義錯誤訊息：
```go
// define status type .
type Status int
const (
    InvalidLogin Status = iota + 1
    NotFound
)
// define statusErr type.
type StatusErr struct {
    Status    Status
    Message   string
}
func (se StatusErr) Error() string {
    return se.Message
}
```

這樣就可以透過 StatusErr 定義較多的詳細訊息：
```go
func LoginAndGetData(uid, pwd, file string) ([]byte, error) {
    err := login(uid, pwd)
    if err != nil {
        return nil, StatusErr{
            Status:    InvalidLogin,
            Message: fmt.Sprintf("invalid credentials for user %s", uid),
        }
    }
    data, err := getData(file)
    if err != nil {
        return nil, StatusErr{
            Status:    NotFound,
            Message: fmt.Sprintf("file %s not found", file),
        }
    }
    return data, nil
}
```

## Panic and Recover
Panic 主要是程式運作時，無法確定接下來應該發生什麼事，就會有 panic 的發生。另外有個內置函數稱為 `panic` ，可以使用任何類型，通常他會是 `string`。
```go
func doPanic(msg string){
    panic(msg)
}
```
這時候在 CLI 上會顯示一些相關資訊，並執行該函式的延遲函式(defer)，如果他是被其他函式呼叫的，則會往上追蹤，一直到 main 函數為止。

Recover 則是一種處理 panic 的方式，透過 `defer` 來檢查是否發生了 panic：  
這個例子中會發生 panic 而使得程式進入 defer，印出錯誤訊息，但因為 recover 的關係，程式會運作下去。
```go
func div60(i int) {
    defer func() {
        if v := recover(); v != nil {
            fmt.Println(v)
        }
    }()
    fmt.Println(60 / i)
}

func main() {
    for _, val := range []int{1, 2, 0, 6} {
        div60(val)
    }
}
```

也就是說，recover 會在 panic 發生後讓程式運作下去，很像其他語言的例外處理，但 recover 會不知道為何發生 panic，只會知道發生了 panic 之後應該做的事情。
e.g. 當我打開檔案讀取與寫入，發生 panic 了，透過 recover 會幫我處理像是關閉檔案，或是紀錄下錯誤訊息並繼續運作，而不會因為 panic 發生而導致程式終止。


## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
