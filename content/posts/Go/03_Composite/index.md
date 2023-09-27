---
slug: "03_Composite"
title: "Golang - 複合類型 (Composite Types)"
date: "2022-02-16"
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
本篇文章基本上是介紹 Go 的複合類型（Array, Slice, Map）與內置函式，另外會簡單介紹 struct。
<!--more-->

## Arrays
如同其他程式語言，Go 也有 Array，Array 中的所有元素，都必須是指定的類型。
```go
var x [3]int
```
此時會建立長度為 3 的 Array，但未指定初始值，上一章提到若未指定初始值，則預設值會是 Zero Value，在這個 Array 中，類型是 ```int```，故初始值會是 **0** 。

若要指定初始值，則可以使用這個方式。
```go
var x = [3]int{10,20,30}
```

也可以用 index 的方式指定初始值。
```go
// index:value
var x = [10]int{1,5:2}
// [1,0,0,0,0,2,0,0,0,0]
```

若已經很明確知道，陣列內的所有數值，在宣告時則可以用 ```[...]``` 省略陣列的長度。
```go
var x = [...]int{10,20,30}
var y = [3]int{10,20,30}
// 這兩個 Array 是一樣的。
```

在 Go 中只有一維陣列，但可以模擬出多維陣列。
```go
var x [5][10]int
```

可以透過內置函式 ```len()``` 得到 Array 的長度。
```go
fmt.Println(len(x))
```

但在 Go 中，很少使用 Array，因為 Array 的大小會被視為是 Array 類型的一部分，```[3]int```是一種類型，```[4]int```是另一種類型。
因此：
* 不能使用變數來指定 Array 的大小，因為類型必須在編譯前解析，而不是在運作時解析。
* 不能使用類型轉換，無法將不同大小的 Array 轉換成相同的類型。
* 最好是知道需要的長度，否則盡量不要使用 Array。
---
## Slices
若資料的長度會變化，且不想受到 Array 的類型限制時，應該使用 Slices，宣告的方式與 Array 極為相似，但不指定大小。

這樣會宣告一個長度為 3 的 Slice。
```go
var x = []int{10,20,30}
```

也可以用 index 的方式指定初始值。
```go
// index:value
var x = []int{1,5:2,88}
// [1,0,0,0,0,2,88]
```

Slice 也可以模擬出多維的 Slice。
```go
var x [][]int
```

比較大的差別在於，Slice 的 Zero Value 並不會是宣告時之類型的 Zero Value，而會是 ```nil```，它是一種標示符號，代表某些類型缺少數值。 ```nil``` 也沒有類型，因此可以賦予值或是與其他不同類型的數值進行比較。

### Slice 內置函數
#### len
在前面的例子中有使用過了 len ，傳入 Array 會獲得該 Array 的長度，len 也適用於 Slice。若傳入 nil Slice 會獲得 0。

### append
```append``` 用於附加值至 Slice。最少要兩個傳入參數：
* 一個任意類型的 Slice
* 一個該類型的數值。
```go
var x []int
x = append(x,10)// [10]
// 可以同時附加多個數值。
x = append(x,20,30,40) //[10,20,30,40]
// 甚至是附加 Slice。但需要使用...這個運算符
y := []int{50,60,70}
x = append(x,y...) //[10,20,30,40,50,60,70]
```

#### cap (Capacity)
Slice 是一系列連續的數值，每個元素被分配到連續的記憶體位置，這樣可以快速讀取或寫入這些數值。每個 Slice 都有一個 Capacity，即保留的連續記憶體位置的數量。

當使用 append 附加數值至 Slice 時，長度也會增加，當長度達到 Capacity，代表沒有空間存放資料了，若又使用 append 附加數值時，則會分配有更大 Capacity 的 Slice，並將原本 Slice 的數值複製到新的 Slice 後，將新的數值附加到 Slice 的最後，並返回新的 Slice。

```cap()``` 可以獲得該 Slice 目前的 Capacity。

但 Capacity 增長，視 Go 語言的版本不同，基本上都會是增長原本大小的一倍，當已經知道某組數值的確切長度時，是否能指定 Capacity 的大小而不造成記憶體的浪費呢? 此時可以透過 ```make``` 內置函數建立確切 Capacity 的 Slice。

#### make
```make``` 可以聲明指定長度、類型與容量的 Slice。

這樣會建立一個類型為 ```int```、長度與容量為5的slice，由於它的長度為5，故它的第0個至第4個元素是有效元素，會被初始化為 ```int``` 的 Zero Value，也就是 0 。
```go
x := make([]int, 5)
```

若要指定容量，則在傳入參數內加入所需的容量大小。
```go
x := make([]int, 5, 10)
```

甚至建立一個長度為 0 的 Slice 也是可行的。
```go
x := make([]int, 0, 10)
// 但後面使用，記得用 append 附加數值，長度為 0 是沒辦法做 index 的。
```

* **記住，絕對不要指定一個小於長度的 Capacity，也盡量不要使用變數指定 Capacity。**

### 宣告 Slice
Slice 在宣告時有不同的方式，但最主要的目標是將 Slice 增長的次數最小化，若 Slice 不會增長，請使用 var 建立 nil slice。
```go
var data []int
```

若有初始值，或是 Slice 的數值不會改變，會建議就是以賦值方式宣告。
```go
data := []int(1, 2, 3, 4)
```

若很清楚 Slice 需要多大，但不清楚內部的數值會是甚麼，請使用 make 宣告 Slice。
* 若以 Slice 作為 buffer 使用，則指定一個非 0 長度的 Slice。
* 若確定 Slice 的大小，則可以指定 Slice 的長度。
* 在其他情況下，使用 make 宣告 0 長度與指定 Capacity 的 Slice，並使用 append 增加數值。

### Slicing Slices

Slice 的表達式由一個起始偏移量和一個結束偏移量組成，並用 ```:``` 分隔。若省略起始偏移量，則為0，若省略結束偏移量，則為結尾。

```go
x := []int{1, 2, 3, 4}
y := x[:2]
z := x[1:]
d := x[1:3]
e := x[:]
fmt.Println("x:", x)
fmt.Println("y:", y)
fmt.Println("z:", z)
fmt.Println("d:", d)
fmt.Println("e:", e)
// 輸出
x: [1 2 3 4] 
y: [1 2] 
z: [2 3 4] 
d: [2 3] 
e: [1 2 3 4]
```

另外以上述的 Slice 表達式，他們之間是共享記憶體的。**使用上請小心。**
```go
x := []int{1, 2, 3, 4}
y := x[:2]
z := x[1:]
x[1] = 20
y[0] = 10
z[1] = 30
fmt.Println("x:", x)
fmt.Println("y:", y)
fmt.Println("z:", z)
// 輸出
x：[10 20 30 4] 
y：[10 20] 
z：[20 30 4]
```
#### Copy
若我想要建立一個 Slice，並使用原始 Slice 的數值，但不共享記憶體的獨立 Slice 時，請使用內置的 ```Copy``` 函式。

```Copy``` 函式有兩個參數，第一個是目標 Slice，第二個是來源 Slice，並會盡量把數值複製到目標，並返回複製的 element 數量。
```go
x := []int{1, 2, 3, 4}
y := make([]int, 4)
num := copy(y, x)
fmt.Println(y, num)
z := make([]int, 2)
num := copy(z, x)
fmt.Println(z, num)
// 輸出
[1 2 3 4] 4
[1 2] 2
```

也可以透過 Slice 表達式，複製部分的數值。
```go
x := []int{1, 2, 3, 4}
y := make([]int, 2)
copy(y, x[2:]) // 若不需要返回值，不用特地設一個變數然後執行 copy()。
```

```copy``` 也可以用來把 Slice 的某部分覆蓋至別的部分。
```go
x := []int{1, 2, 3, 4}
num = copy(x[:3], x[1:])
fmt.Println(x, num)
// 輸出
[2 3 4 4] 3
```
---

## Strings and Runes and Bytes
Go 使用一個 Bytes 序列代表一個 String。

```go
var s string = "Hello there"
var b byte = s[6] // t
```

前述提到的 Slice 表達式，也可以用在 String。
但建議在 String 每一個字元都是一個 Byte 大小時再使用。(e.g. emoji是4個bytes)

單個 rune 或是 byte 可以使用類型轉換成 string。

但常使用的 int ，透過類型轉換的話，會變成 ascii 而不是直接轉換，若要單純的把 int 轉換成 string，請使用 ```strconv.Itoa()```。 [參考資料](https://stackoverflow.com/questions/10105935/how-to-convert-an-int-value-to-string-in-go)

字串也可以轉成 rune slice 或是 byte slice，使用方式也不困難，通常會使用 byte slice 做轉換。
```go
var s string = "Hello, world!"
var bs []byte = []byte(s)
var rs []rune = []rune(s)
```
---

## Map
### intro
Map 其實與其他程式語言相似，將一個數值關聯到另一個數值的類型。Map 的 Zero Value 是 nil。
```go
var nilMap map[string]int
```
但這種方式，在寫入 nil 時會導致恐慌，可以透過另一個方法建立映射變量。
```go
myMap := map[string]int{}
```
若知道確切的數值，可以用賦值宣告的方式建立Map。

宣告方式為 key:value，每組數值都用 ```,``` 分隔，即使是最後一組也要加上逗號。
```go
rank := map[string]int{
    "I" : 1,
    "you" : 2,
    "she" : 3,
}
```
若知道 Map 的確切大小但不清楚內部數值，可以使用 ``` make ``` 建立有默認大小的 ```map```。
```go
ages := make(map[int][]string, 10)
```

Map 與 Slice 相似的地方：
* 增加 key:value pair 數據時，Map 會自動增長。
* 若知道會有多少筆數據，則可以使用 make 建立有初數大小的 Map。
* 將 Map 傳遞給 len ，可以獲得 key:value pair 的數量。
* Map 的 Zero Value 是 nil。
* Map 沒有可比性，只能檢查是否等於 nil，但無法檢查兩個 Map 是否有相同的 key:value pair。

另外有些要注意的點：
* key 必須是可以比較的類型，像是slice或是map這種無法比較，無可比性的類型就不能使用。
* 若數據要按照順序處理，建議使用 Slice，若數據不用嚴格按照順序處理，則可以使用 Map。

### Read and Write

```go
myMap := map[string]int{}
myMap["Taipei"] = 1
fmt.Println(totalWins["Taipei"])
fmt.Println(totalWins["I-lan"])
myMap["I-lan"]++
fmt.Println(totalWins["I-lan"])
// 輸出
1
0
1
```
透過 key 的方式分配 value，這邊要注意是使用 ```=``` 而不能使用 ```:=```，若要讀取未分配 value 的 key 之 value 時，則會返回 Zero Value。

### The comma ok Idiom
那要如何知道 Map 中，我所需要的 key:value pair 是否在 Map 中？可以使用 comma ok Idiom 的方式區分 key:value pair 是否在 Map。
```go
m := map[string]int{
    "hello": 5,
    "world": 0,
}
v, ok := m["hello"]
fmt.Println(v, ok)

v, ok = m["world"]
fmt.Println(v, ok)

v, ok = m["goodbye"]
fmt.Println(v, ok)
// 輸出
5 true
0 true
0 false
```

### Deleting from Maps
刪除 key:value pair 的方式，透過內置函數 ```delete```即可，
```go
m := map[string]int{
    "hello": 5,
    "world": 10,
}
delete(m, "hello")
```
若 key 不存在於 map 內，或是說 map 是 nil，則甚麼都不會發生 !!

### (補充 : Using Maps as Sets)
因為 Go 沒有 set 這個類型，但可以透過 Map 去實現。

```go
intSet := map[int]bool{}
vals := []int{5, 10, 2, 5, 8, 7, 3, 9, 1, 2, 10}
for _, v := range vals {
    intSet[v] = true
```
透過 map 與迴圈，將設定的數值與 bool 連結，若有的數值則設定為 true，其他未在內的數值，因為 bool 的 Zero Value，都會是 false。這樣使用上就可以達到 set 的功能。

---
## Struct

若有想要組合在一起的相關數據時，應該訂意一個 struct。
```go
type person struct{
    name string
    age int
    pet string
}
```
一個 struct 透過關鍵字 type、結構類型的名稱與struct組成。struct 內部則是field，由變數名稱與變數類型。
聲明 struct 後，就可以定義該類型的變數。

基本上這兩種方式，都會將 struct 內的所有 field 設定為 Zero Value。
```go
var renne person
lapis := person{}
```

若有初始值的話則是依據 field 宣告，**記得按照順序，依據類型宣告。**

```go
nadia := person{
    "Nadia",
    "18",
    "cat",
}
```
或者是以類似 key:value pair 的方式宣告。**可以不必按照順序宣告，可以指定部分變數即可，沒被指定的會被設定為 Zero Value。**(建議用這種 !!)
```go
Tio := person{
    name: "Tio",
    age: "18",
    pet: "cat",
}
```

struct 內的 field 用 ```.``` 進行訪問。
```go
Tio.name = "Tio Plato"
fmt.Println(Tio.name)
```

### Anonymous Structs(匿名結構)
簡單來說，就是實現一個 struct 但不需要先命名，稱為匿名結構。通常用在將外部數據轉換成 struct，或是將 struct 轉換成外部數據(e.g. json)，這被稱為*unmarshaling and marshaling data*。
```go
pet := struct {
    name string
    kind string
}{
    name: "Cute",
    kind: "cat",
}
```

### 比較與轉換結構
不同類型的結構之變數之間，是不能比較的，除非兩個struct的field具有相同的名稱、順序與類型，才允許進行比較與類型轉換。

在 struct 的比較中，若其中至少有一個匿名結構的話，若兩個結構的 field 有相同的名稱，則可以在不進行類型轉換的情況下進行比較，若兩個結構的 field 具有相同的名稱、順序與類型，還可以在兩個結構之間進行 assign。

```go
type firstPerson struct {
    name string
    age  int
}
f := firstPerson{
    name: "Bob",
    age:  50,
}
var g struct {
    name string
    age  int
}

// compiles -- can use = and == between identical named and anonymous structs
g = f
fmt.Println(f == g)
```


## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [How to convert an int value to string in go](https://stackoverflow.com/questions/10105935/how-to-convert-an-int-value-to-string-in-go)



