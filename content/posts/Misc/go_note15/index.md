---
slug: "learning_Go_Summary"
title: "Learning Go 讀後感(Summary)"
date: "2022-03-30"
lastmod: "2022-03-30"
tags: ["Misc"]
sidebar: auto 
categories: ["Misc"]
no_comments: false
draft : false
resources:
- name: "featured-image-preview"
  src: "go_header.png"
- name: "featured-image"
  src: "go_header.png"
---

簡單分享一下看完 Learning Go 的心得，與未來對該語言的想法。

<!--more-->
## 心得

因為 Hugo 的關係，體會到 Go 的速度非常快，且有一些有名的專案(Docker, K8s, Terraform) 是由 Go 實現的，故才會想花些時間去了解 Golang。  
基本上對於 Go 的使用方法有一定的了解，雖然從 `interface` 就認為有點苦手，但後面的像是 standard library 或是 test 等，基本上還算可以理解。  
後續花時間寫自己想做的東西，應該能了解的快一些吧。

## 未來

在我閱讀這本書，與撰寫筆記的這段期間，Go 釋出了 1.18 的更新，我認為很重要的兩項：  
* 泛型(Generics)
* 模糊測試(Fuzzing)

書裡面是有 Generics ，原先想說還沒發佈，且我現在是使用 1.15 版，就想說等發佈了再看。~~之後補上~~  
另外像 `Concurrency` 與 `Context`，以及一些欠債要還的資訊(e.g. garbage collector) 都會再補上。 
還有我在第八篇說過，後面品質不太好的部分，都會再做補充。~~如果有空~~  
過一陣子再回來看自己寫的，也會比較容易發現說當初哪裡寫得不太恰當或是錯誤的，再行修改。  

不過，會暫時休息，換去網路跑道跑一陣子之後再回來繼續學習。
感謝自己這陣子願意花時間去學習 golang。

{{<image src="crab.jpeg" caption="所以我就帶自己去吃個小螃蟹(?" width="100%">}}

## 參考資料(Reference)

1. [Learning Go](https://www.amazon.com/Learning-Go-Idiomatic-Real-World-Programming/dp/1492077216) (書籍)
2. [Go 1.18 is released!](https://go.dev/blog/go1.18)

