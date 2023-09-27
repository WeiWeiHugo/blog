---
slug: "07_Type_Methods_Interfaces"
title: "Golang - 類型、方法與介面(Types, Methods, and Interfaces)"
date: "2022-03-15"
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

本篇文章會介紹類型、方法和介面。

<!--more-->

## 類型(Types)

一種基本上就像我們前面用過的，由開發者自行定義的類型。
```go
type Person struct{
    Name string
    Age int
}
```

另外也可以使用原始類型，甚至是以複合的方式定義類型。
```go
type Score int
type Converter func(string)Score
type TeamScore map[string]Score 
```

基本上可以在任何的 code block 定義類型，但就只能在該 code block 的範圍內存取，唯一的例外是 exported package block level type。

## 方法(method)
### 介紹
聲明方法其實很像聲明函數，只是差在多了一個 receiver specification。
```go
type Person struct{
    Name string
    Age int
}
func (p Person) String() string {
    return fmt.Sprintf("%s, age %d", p.Name,  p.Age)
}
```
receiver 的名稱通常會是該類型的縮寫，通常是該類型的第一個字母。在其他語言中經常是使用 `this` 或 `self`。

方法不能重載（overloading），這是 go 語言的特性，或許從其他語言再過來使用時，使用上可能會感到有些限制。

那這是聲明方式，使用方式呢？也很簡單：
```go
me := Person {
    Name : "Wei",
    Age : 26,
}
output = me.String()
```
### 接收器(receiver)
前面的文章提到過，如果函數使用 pointer 類型的參數，代表這個傳入參數可能會被函式修改，相同的概念一樣在 receiver 上，receiver 有 pointer receiver 與 value receiver 兩種。

決定使用哪種 receiver 可以參考以下規則：
* 如果這個 method 需要修改 receiver，則必須使用 pointer receiver。
* 如果這個 method 要處理 nil 的情況，則必須使用 pointer receiver。
* 如果這個 method 不會修改 receiver，則可以使用 value receiver。
* 如果這個 type 內的 method 中，有一個以上的 pointer receiver，建議保持一致，讓所有的 method 都使用 pointer receiver。


### Methods are Function
Method 其實就很像前面提到的 Function，我在書中看到一個比較特別的用法是 method expression，直接從 type 中建立一個 function。
```go
type Adder struct {
    start int
}

func (a Adder) AddTo(val int) int {
    return a.start + val
}

myAdder := Adder{start: 10}

// method expression
f1 = Adder.AddTo
fmt.Println(f1(myAdder, 10)) // print 20
```

當然，從已經建立的 type 中將 method 取出來用也是可行的。
```go
f2 = myAdder.AddTo
fmt.Println(f2(15)) // print 25
```
{{< admonition Tips "Tips">}}
複習一下：Golang 是 Call by Value.
{{</ admonition >}}

## 介面(Interface)
interface 定義並描述了某些其他類型必須具有的確切方法。

聲明 Interface 也很簡單。interface 出現在 type 的名稱後方，通常會以 "er" 做為結尾命名。  
我們在標準函式庫中，可以看到這個 `fmt.Stringer` 介面。

```go
type Stringer interface {
    String() string
}
```
{{< admonition Tips "Tips">}}
看到 type 了嗎？所以也有一種類型是介面類型(interface type)。
{{</ admonition >}}

如果某個 type 有精確簽名(exact signature)的 method，就可以說他滿足該 interface，  
像下面這個範例，因為他有一個 `String() string` 的 method。

```go
type Book struct {
    Title  string
    Author string
}

func (b Book) String() string {
    return fmt.Sprintf("Book: %s - %s", b.Title, b.Author)
}
```
下方這個 `Count` 類型也滿足 `fmt.Stringer` 介面。
```go
type Count int

func (c Count) String() string {
    return strconv.Itoa(int(c))
}
```

我們有兩種不同的類型，但他們都滿足 `fmt.Stringer` 介面。  
反過來說，如果知道某個類型滿足 `fmt.Stringer` 介面，代表他有一個 method 是 `String() string`。

也就是說，當我們在任何地方看到具有 interface type 的聲明，都可以使用任何類型，只要他滿足該 interface。
這樣子我們不需要在意傳入的類型是什麼，我們需要在意的是該類型有什麼方法。
```go
func Printer(s fmt.Stringer) {
    log.Println(s.String())
}
book := Book{"I am learing Go", "Wei"}
count := Count(1)
Printer(book)
Printer(count)
// output
Book: I am learing Go - Wei
1
```
### 使用 interface 的理由
1. 減少重複或是樣板的程式碼。
2. 為了更容易在單元測試中使用模擬而不是實體類型
3. 作為一種架構工具，有助於強制執行程式碼庫各部分之間的解耦(decoupling)。

第一點有一部分是因為，函式庫內已經幫你實現了許多，就不需要再去實現。  
當然，要自己實現 interface 肯定是沒問題的。
這裡附上一些常用到的 interface type 。[link](https://www.alexedwards.net/blog/interfaces-explained#useful-interface-types)

第二點，當產品有時會用到 DB 或是 internet 時，測試就不太好做，  
比方說，我只是要測試一個 insert 的功能，但我在測試上我得先建立一個 Database 的測試實例，  
但我可以透過建立一個 interface 並模擬 Database 達到我要的目的，在測試上我就不需要使用實際的 Database。

最後也是 interface 最重要的一個特色，解耦。  
降低函式庫之間的依賴關係，讓程式依賴於抽象，而不是依賴於實現。  
以前面的例子來說，當我更改了 `Book` ，我不必去顧慮其他的程式，我只需要注意 `Book` 有沒有一個 method 滿足 interface 即可。
但我將 `Printer` 的傳入參數更改為 `Book` 時，一來是我要考量到 `Count` 需不需要使用 `Printer`，二來是要考量到修改 `Book` 會不會影響到 `Printer` 運作。  
不過就是剛開始開發時會比較累一點，畢竟多了一個 interface 要設計。

### nil
nil 也是 interface type 的 zero value。  
只是要注意一點，是 interface 的 type 與 value 都是 nil，才會被認為是 nil。
如果 interface 是 nil，使用他的 method 會造成 panic。
但就算 interface 不是 nil，如果 value 是 nil 且沒有處理好該情況，仍然有機會產生 panic。

### Empty interface
有時候需要一種方式，代表可以儲存任何類型的 value:
```go
var i interface{}
i = 20
i = "Hi"
i = struct{
    Name string
    Age int
}{"Wei", 26}
```
`interface{}` 可以保存任何類型的值。通常用於處理未知類型的函式，  
像我們很常用的 `fmt.Println()`。  
不過使用上要注意，因為會不知道儲存的是什麼類型的數值。而且 Golang 是強型態語言，語言特性說實在的，與 empty interface 有點衝突啦。

### Type Assertions and Type Switches
若我們把 value 存到 `interface{}`，則可以透過 Type Assertions 與 Type Switches 觀察是否有特定的具體類型，或是具體類型實現了另一個 interface。  
Type Assertions 實現該 interface 的類型，或是命名另一個 interface。
```go
type MyInt int

func main() {
    var i interface{}
    var mine MyInt = 20
    i = mine
    i2 := i.(MyInt)
    fmt.Println(i2 + 1)
}
```
此時 i2 的類型是 `MyInt`。
若我們更改成以下程式碼，會造成 panic，是因為類型不對而導致的錯誤。
```go
i2 := i.(string)
fmt.Println(i2)
```

Type switches 則是當 interface{} 可能會儲存多種類型的值時，透過 switch 進行判斷：
```go
func doThings(i interface{}) {
    switch j := i.(type) {
    case nil:
        // i is nil, type of j is interface{}
    case int:
        // j is of type int
    case MyInt:
        // j is of type MyInt
    case io.Reader:
        // j is of type io.Reader
    case string:
        // j is a string
    case bool, rune:
        // i is either a bool or rune, so j is of type interface{}
    default:
        // no idea what i is, so j is of type interface{}
    }
}
```

### Function Interface
前面介紹 type 與 function 時提到，function 也可以是一種類型，因此也可以用在 interface 上，透過 function type 實現 interface。  
常見的例子是 http 服務：
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
     f(w, r)
}

```
透過 `HandlerFunc` 進行類型轉換，將函式快速轉換為 `Handler` 的介面類型，使得任何具有簽名的函數`func(http.ResponseWriter,*http.Request)`都可以用作`http.Handler`，透過 `http.Handler` 可以使用 function, method 與 closures 實現 HTTP 服務。  
將符合 interface 的函數定義為 type，對這個 type 實現 interface 中的函式，使用的時候只要把自定義的函式做型態轉換就可以使用了。注意一點，原本的類型要相同才可以使用。


## 補充
關於解耦的部分，可以參考 solid 的依賴反轉原則，了解到解耦的優缺點。[DIP](https://web.archive.org/web/20110714224327/http://www.objectmentor.com/resources/articles/dip.pdf)

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [Golang Interfaces Explained](https://www.alexedwards.net/blog/interfaces-explained)