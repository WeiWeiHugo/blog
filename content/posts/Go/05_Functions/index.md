---
slug: "05_Functions"
title: "Golang - 函式(Functions)"
date: "2022-02-26"
lastmod: "2022-03-01"
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

本篇文章會說明如何使用 golang 撰寫函式，瞭解傳入值與返回值，介紹匿名函式與defer，並瞭解可以對函式所做的事情。

<!--more-->

坦白說書對於這個章節寫的有點模糊，~~好啦其實是我自己理解能力差~~，這次看了不少 reference，若以後覺得還有要補充的地方會補上。

## 宣告與使用函式

基本上宣告函式與其他程式語言幾乎相同，聲明會有四個部分：
* keyword, ```func```
* 函數名稱
* 輸入參數
* 返回類型

```go
func example(input int) int {
  fmt.Println("This is example.")
  return 0
}
```

如果說這個函式沒有傳入參數，則小括號裡面則不需要填寫任何參數；如果說這個函式不會有任何的返回值，則宣告時也不需要寫返回類型。常使用的 ```main()``` 函式就是一個很好的例子。
```go
func main(){
  fmt.Println("Hello world")
}
```

若有許多傳入參數，且都為同一種類型時，可以稍微省略一些輸入參數的類型，但我是不建議就是了，每一個參數都寫清楚，後續維護會比較好處理一些。
```go
func example(number1, number2 int) int {
  return number1+number2
}
```

## 具名與選擇性參數 (named and optional)

基本上許多程式語言都具有這兩個性質，但在 golang 中並沒有這兩個性質，基本上 golang 要求你宣告的輸入參數，全部都要使用（但也有例外）。
* 具名參數：用於識別傳入參數為何。許多情況下我們會傳入很類似的參數，透過具名參數的方式較容易識別。
* 選擇性參數：代表這個參數是選擇性傳入的，宣告函式時，選擇性參數必須擺在後面。若要選擇多個選擇性參數使用時，可以使用具名參數。

雖然 golang 無法使用這兩種參數，但可以用模擬的方式。

首先要訂一個 struct，代表這個函式會使用到的所有參數：
```go
type MyFuncOpt struct {
    FirstName string
    LastName string
    Age int
} 
```

接著，宣告函式時要使用前面宣告的 struct 作為傳入參數。使用時則傳入所需的數值即可。
```go
func MyFunc(opts MyFuncOpt) error{
  // do something
}
// 可以由外部傳入值
var i int = 55
// How to use.
func main() {
	MyFunc(MyFuncOpts{
		FirstName: "Joe",
		Age:       i,
	})
}
```

## 可變參數
Golang 支援可變參數，可以允許任意數量的輸入參數。可變參數必須是函式聲明時的唯一一個參數，或是最後一個參數，並在類型前面加上 ```...```表示，這時會以這個類型的 slice 表示這個參數。
```go
func number(base int, value ...int) int{
  //do something.
}
```
上述的例子中， value 已經是一個類型為 int 的 slice，所以也可以傳 slice 給他，只是說傳入時要在後面加上 ```...``` 才能使用，否則會發生編譯錯誤。
```go
number(1, []int{2,3,4,5}...)
```

## 多個返回值
Golang 的一個特色是，允許多個返回值。
```go
func number(base int, value int)(int,int){
  return base+value, base*value
}
```
宣告函式時，多個返回值的類別需要以括號包起來，並以逗號隔開；函式內回傳值則是以逗號隔開即可。

多個回傳值則需要分配給多個變數，或是都被使用到，而不能像 python 一樣只分配給一個變數，因為 golang 多個回傳值就是回傳多個，不是像 python 一樣回傳一個數組。
```go
numbers := number(1,2) // error
number1, number2 := number(1,2) // success
```

返回值也是可以被具名的(named)，在宣告返回值類別時就都要加上括號，即便只有一個返回值也是，若只想命名部分的返回值，可以使用 ```_```。另外具名的返回值可以分配在不同名稱的變數上，沒問題的。
```go
func number(base int, value int)(ans1 int, ans2 int, _ int){
  ans1, ans2 = base+value, base*value
  ans3 := 0
  return ans1,ans2,ans3
}
```
不過，即使命名了返回值，也不代表一定要返回該值，golang 是允許不返回命名返回值的。使用上請小心，因為編譯器會將 return 的值分配給命名的返回值。
```go
func number(base int, value int)(ans1 int, ans2 int, _ int){
  ans1, ans2 = 1,2
  ans3 := 0
  return base+value,base*value,ans3
}

func main(){
  fmt.Println(number(5,2))
}
// Output
7 10 0
```

## 空白返回 (Blank returns)
空白返回適用在當我們命名了所有的返回值時，可以直接使用 return 而返回數值，不需要在 return 後面補上變數。
```go
func number(base int, value int) (ans1 int, ans2 int) {
	ans1, ans2 = 1, 2
	return
}
```
雖然會節省許多時間，但維護上會很困擾。因為要去尋找返回值是從哪裡返回的，當函式一長，或是有多個 return，由判斷式決定的函式，看到都是 return，~~頭會很痛。~~
```go
// 舉一個超超超簡單的例子。
func number(base int, value int) (ans1 int, ans2 int) {
	return // ???????? 哦哦回傳 zero value
}
func main(){
  fmt.Println(number(5,2))
}
// Output
0 0
```

## 匿名函式
在函式中定義新函式，並將其分配給新的變數。而這些內部的新函式則是匿名函式。宣告方式基本上與一般函式相同，主要的差別在匿名函式不需要命名。使用時在後面用小括號傳入變數即可。
```go
func main() {
    for i := 0; i < 5; i++ {
        func(j int) {
            fmt.Println("printing", j)
        }(i)
    }
}
//output
printing 0 
printing 1 
printing 2 
printing 3 
printing 4 
```

但我在 for loop 使用 fmt.Println() 就好了，直接使用內部的程式碼不是更好嗎？匿名函式的好處在哪裡呢？後續會在 defer 與 launching goroutines 這兩種情境中提到，匿名函式在這兩種情境中會是有用的。

## Closures
### 介紹
中文翻譯好像稱為閉包 ~~(不喜歡)~~，Closures 可以限制函式的使用範圍，如果一個函式只會被另一個函式使用，但他會被多次使用，可以使用一個內部函式隱藏被使用的函式，正常函式呼叫完後內部的變數就會銷毀，但閉包卻能使本該銷毀的變數一直保留。
```go
func intSeq() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}

func main() {
	nextInt := intSeq()

	fmt.Println(nextInt())
	fmt.Println(nextInt())
	fmt.Println(nextInt())

	newInts := intSeq()
	fmt.Println(newInts())
}
// Output
1
2
3
1
```
宣告一個 intSeq 的函式，返回值是函式，函式內第一個 return 使用了外部的 i，使得這個匿名函式成為了 Closures，而 i 則會被保留狀態並等待下次使用。

> tips : 把函式想成一個數值會比較好理解，而運作方式也的確是如此。

### 從函式返回函式
不只是能返回變數的狀態，也能夠返回函式的 closure：
```go
func makeMult(base int) func(int) int {
	return func(factor int) int {
		return base * factor
	}
}

func main() {
	twoBase := makeMult(2)
	for i := 0; i < 3; i++ {
		fmt.Println(twoBase(i))
	}
}
// Output
0
2
4
```

twoBase 的值是 makeMult(2) **(記得函式也是一個數值的想法)**，此時賦予 base 為 2。後面的 for loop 時，因為對於 twoBase 傳入了數值，故回傳內部的匿名函式。

## defer
通常使用 defer 是用來釋放資源，程式運作時需要常常釋放資源，例如開啟文件後要關閉，或是網路連線完成後，透過 defer 進行清理的工作。通常擺在函式的後面，與其他語言的 finally 相似。
```go
func main() {
    f := createFile("/tmp/defer.txt")
    defer closeFile(f)
    writeFile(f)
}

func createFile(p string) *os.File {
    fmt.Println("creating")
    f, err := os.Create(p)
    if err != nil {
        panic(err)
    }
    return f
}

func writeFile(f *os.File) {
    fmt.Println("writing")
    fmt.Fprintln(f, "data")
}

func closeFile(f *os.File) {
    fmt.Println("closing")
    err := f.Close()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}
// output
creating
writing
closing
```
比方說要產生一個檔案，並寫入檔案後關閉對該檔案的存取，但我們在 main 中撰寫的順序是 createFile, closeFile，最後是 writeFile，但因為 defer 的關係， closeFile會在函數結束前執行。

補充一下 defer 的事情：
* 可以透過 defer 在函式中使用多個 closure，會以先入後出的方式進行，最後使用 defer 的會先運作。
* defer 會是在 return 之後運行的。
* defer 會造成延遲，若是要求速度至上的程式，請盡量減少使用。

defer 另一個好處是，可以用來檢查或修改返回值。且這也是前面提到命名返回值的理由。
```go
// 書裡面的例子。
func DoSomeInserts(ctx context.Context, db *sql.DB, value1, value2 string)
                  (err error) {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if err == nil {
            err = tx.Commit()
        }
        if err != nil {
            tx.Rollback()
        }
    }()
    _, err = tx.ExecContext(ctx, "INSERT INTO FOO (val) values $1", value1)
    if err != nil {
        return err
    }
    // use tx to do more database inserts here
    return nil
}
```
函式最後會回傳一個 error 類型的 err 變數，此時中間的 defer 會對這個數值進行判斷，如果沒有錯誤，則會提交，如果有錯誤的會則會 rollback。

## Call by Value
當我們傳遞變數至函數時， Golang 會將變數的值複製並傳遞進去。函式並不會改變該變數原本的數值。與其他語言不同的地方是，即使是傳遞一個 struct 至參數內，也不會改變在外部 struct 內的數值。

然而，傳遞 map 與 slice 進入參數時，函式進行修改是會影響到外部變數內的數值的，因為 map 與 slice 是用 pointer 實現的，對這個 pointer 所指的數值進行修改，就會影響到外部變數的數值。另外，傳遞至函式內的 slice，是無法延長的。

若我們需要像是 int, string 類型的變數進入函式進行修改時，要怎麼做呢？

> **pointer !!**

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [具名和選擇性引數 (C# 程式設計手冊)](https://docs.microsoft.com/zh-tw/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments)
3. [More on Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-type-expressions)
4. [談談我所理解的閉包，js、php、golang裡的closure](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/107344/#outline__3)
5. [Go by Example: Closures](https://gobyexample.com/closures)
6. [Go初識-day10-閉包(closure)](http://www.taroballz.com/2018/04/01/Go_closure/)
7. [Go by Example: Defer](https://gobyexample.com/defer)