---
slug: "11_StandardLib"
title: "Golang - 標準函式庫(Standard Library)"
date: "2022-03-24"
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

本篇文章將簡單介紹 Standard Library 內的幾個較重要的 package，像是io, time, encoding/json, net/http。

<!--more-->
## io

io 是個很常被使用的 package，尤其是 `io.Reader` 與 `io.Writer`。
看到 er 結尾了嗎？所以他應該是... interface ?

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Read 會將資料讀入 p，一個 byte 類型的 slice，並傳回讀取的 byte 數量，也就是 n，另外還會返回遇到的錯誤。  
Write 則會將 p 輸出並回傳寫入的 byte 數量與遇到的錯誤。  

`io.reader` 表示 data stream 的讀取端，標準函式庫已經有許多包含此 interface 的 implement。當 stream 結束時會返回 `io.EOF` 的錯誤。

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("Hello, Reader!")

	b := make([]byte, 8)
	for {
		n, err := r.Read(b)
		fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
		fmt.Printf("b[:n] = %q\n", b[:n])
		if err == io.EOF {
			break
		}
	}
}
```

`io.writer` 則是反過來，把 data 寫進 stream。
```go
package main

import (
	"fmt"
	"os"
)

func main() {
    f, err := os.OpenFile("/tmp/123.txt", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0600)
    if err != nil {
        panic(err)
    }
    defer f.Close()
    // Here
    n, err := f.Write([]byte("writing some data into a file"))
    if err != nil {
        panic(err)
    }
    fmt.Println("wrote %d bytes", n)
}
```

另外，io 還有一些像是 `io.Closer`, `io.seeker` 等 interface。使用上其實都很容易。

### ioutil
`ioutil` 則實現了更多更方便的函數。
* ReadAll : 一次性讀取 `io.reader` 內的數據。
* ReadDir : 讀取目錄底下所有的文件與子資料夾名稱。
* ReadFile : 讀取檔案。
* WriteFile : 寫入檔案。

還有一些像是 `TempDir`, `TempFile` 等函數，有興趣的可以用 `go doc ioutil` 觀看。

## time
基本上有兩種類型表示時間：
* `time.Duration`
* `time.Time`

`time.Duration` 用來表示一段時間，基於 `int64` 實現的，也可以使用 `time.ParseDuration`，用字串來表示時間。
例如我寫這篇文章花費了五小時又三十分鐘：
```go
cost := 5 * time.Hour + 30 * time.Minute
costStr, _ := time.ParseDuration("5h30m")
```

另外，`time.Duration` 已經滿足了 `fmt.Stringer`，所以會返回一個格式化過的字串。
```go
fmt.Println(cost)
// Output
5H30M0S
```
`time.Time`，可以用使用 `time.Now` 取得現在時間，或是填入想設定的時間
```go
type Time struct{ ... }
    func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time
    func Now() Time
```

最後，Go 是使用 monotonic clock 來跟蹤時間的。

### Timer and Ticker
Timer 基本上就是計時器，通常用在 select 對於多個 channel 的超時，或是一些讀寫行為的超時情形，他是一次性的，這也是與 Ticker 不同的地方，Ticker 是每隔一段時間就進行一次。
```go
type Timer struct{ ... }
    func AfterFunc(d Duration, f func()) *Timer
    func NewTimer(d Duration) *Timer

type Ticker struct{ ... }
    func NewTicker(d Duration) *Ticker
```

## encoding/json
RESTful API 讓 json 成為資料傳遞的重要格式，也因此 Go 也支援將 Go 的數據類型轉換成 json，或是由 json 轉換回來。  
假設說我們有筆 json 資料：
```json
{
    "id":"12345",
    "date_ordered":"2020-05-01T13:01:02Z",
    "customer_id":"3",
    "items":[{"id":"xyz123","name":"Thing 1"},{"id":"abc789","name":"Thing 2"}]
}
```
那我們就需要建立 struct 去對應名稱與類型，進行轉換。
```go
type Order struct {
    ID            string        `json:"id"`
    DateOrdered   time.Time     `json:"date_ordered"`
    CustomerID    string        `json:"customer_id"`
    Items         Item          `json:"items"`
}

type Item struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}
```
有一些地方要注意的：
1. 名稱前要加上 json 標籤。
2. 要使用反引號刮起來。
3. 如果輸出 json 時，因為某些欄位沒有數值而要忽略某些欄位，請在後方加上 `,omitempty`
```go
type Item struct {
    ID   string `json:"id"`
    Name string `json:"name,omitempty"`
}
```
4. 使用 `,omitempty` 要注意到 struct 的 zero value 不是代表空值，所以 `,omitempty` 仍然會轉換成 json，如果 struct 內部有 struct 類型的資料，又想在沒資料時忽略該欄位，請在 struct 上使用 pointer。
```go
type Order struct {
    ID            string        `json:"id"`
    DateOrdered   time.Time     `json:"date_ordered"`
    CustomerID    string        `json:"customer_id"`
    Items         *Item         `json:"items"`
}

type Item struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}
```

### 編碼與解碼
透過 `json.Unmarshal` 與 `json.Marshal` 進行 json 與 struct 之間的轉換。  
`json.Unmarshal` 可以把 json 字串轉成 struct，而 `json.Marshal` 可以將 struct 轉成 json。
```go
var o Order
err := json.Unmarshal([]byte(data), &o)
if err != nil {
    return err
}
out, err := json.Marshal(o)
```

另外也可以透過 `json.Decoder` 與 `json.Encoder` ，對檔案進行解碼與編碼，這樣就可以用 `io.Reader` 與 `io.Writer` 進行 io 讀取與寫入的操作。
```go
err1 = json.NewEncoder(tmpFile1).Encode(toFile)
err2 = json.NewDecoder(tmpFile2).Decode(&fromFile)
```


## net/http

我覺得 go 提供的一個很棒的 package，它可以讓我們開發 http server 與 client 相關的程式！！

### Client
我們可以建立一個 `http.Client`，並透過 `http.Request` 等方式進行用戶端的需求。如果說沒有 context 要傳送的話，請記得使用 `nil`。  
Method 也有常見的 GET, POST, PATCH 可以使用。
```go
client := &http.Client{
    Timeout: 30 * time.Second,
}
req, err := http.NewRequestWithContext(context.Background(),
    http.MethodGet, "https://google.com", nil)
if err != nil {
    panic(err)
}
```

我們也還可以添加一些關於 header 的資訊，添加後使用 Do 去處理，會得到相對的 response：
```go
req.Header.Add("X-My-Client", "Learning Go")
res, err := client.Do(req)
if err != nil {
    panic(err)
}
```

Response 則會有許多資料：
* status
* statusCode
* header
* body

### Server
Server 的用途就是監聽 http 的請求，另外他也支援 tls 與 http/2。~~好神~~
```go
package main

import (
    "fmt"
    "net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world")
}

func main() {
    http.HandleFunc("/", indexHandler)
    http.ListenAndServe(":8080", nil)
}
```
這樣我們會在 "/" 上面建立一個 indexHandler，當有 Request 時，則會進行 ResponseWriter ，產生 response。

但這樣子只能處理單個 request，因此標準函式庫內有一個請求路由器 (`*http.ServeMux`)，並透過 `http.NewServeMux` 實現，透過ServeMux，我們可以傳入路徑與 handler，當 mux 收到對應的請求時，會根據路徑而使用對應的 handler 進行處理。

{{<image src="router.png" width="100%">}}

```go
package main

import (
	"net/http"
)

func main() {
	cat := http.NewServeMux()
	cat.HandleFunc("/voice", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("meow!\n"))
	})
	dog := http.NewServeMux()
	dog.HandleFunc("/voice", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("wow!\n"))
	})
	mux := http.NewServeMux()
	mux.Handle("/cat/", http.StripPrefix("/cat", cat))
	mux.Handle("/dog/", http.StripPrefix("/dog", dog))

	http.ListenAndServe(":3000", mux)
}
```

## 補充
更多的使用方式，`go doc [package name]`  
或是問 Google，畢竟標準函式庫提供了不少函式可以使用...

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [An Introduction to Handlers and Servemuxes in Go](https://www.alexedwards.net/blog/an-introduction-to-handlers-and-servemuxes-in-go)
3. [Golang 的 “omitempty” 关键字略解](https://old-panda.com/2019/12/11/golang-omitempty/)
4. [Golang Reader Example](https://golang.cafe/blog/golang-reader-example.html)


