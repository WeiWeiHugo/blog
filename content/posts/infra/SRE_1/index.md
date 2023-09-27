---
slug: "SRE_1"
title: "Site Reliability Engineering - Introduction"
date: "2022-08-30"
lastmod: "2022-08-30"
tags: ["Infra","SRE"]
sidebar: auto 
categories: ["SRE"]
no_comments: false
draft : false

---

<!--more-->
## 前言
這邊是我自己想寫的廢話，可以 skip 不看。  

基本上現在處於一種 Chaos 的狀態吧，一邊點著 coding 的技能，另一邊又不願意放開 system & networking administration(management) 的技能樹，有一陣子去看了 SRE 的書籍，坦白說可能也不是很適合自己，但仍以學習新知識的角度來學習這項技能(或是知識)。  

參考書籍 Google 已經提供，我有附在後面的參考資料。  
基本上會是以 **Site Reliability Engineering** 這本書來作筆記。(~~其實是有想起來這本書看了一半，就乾脆把書看完吧~~)

## Tradition
公司透過系統來提供服務，而這些系統並不會自動運作，歷史上來看會需要雇用系統管理員(Administrator, Admin)來運行或維護系統(IT,MIS)，Admin的任務基本上是做部屬，讓系統之間能協同工作並提供服務，還有在問題發生時進行處理(Update, troubleshooting ...)，由於技能樹點的不同，基本上在大部分的公司中，軟體或服務開發人員，與系統管理員，會分為兩個團隊。  

目前在系統管理上，有許多現成的工具，並且網路上也有許多例子可以做學習，我們不必花費太多時間在重造輪子上。

然而當系統成長時，管理的成本也逐漸提高。例如原先手動一次就可以完成的事項，在系統成長後需要做到 100 次，甚至 1000 次。或是團隊之間對服務的看法不同，這經常發生在開發團隊與管理團隊之間。這是會影響團隊之間是合作或是分裂的成本。
* 開發團隊想推出新功能，管理團隊想要穩定服務(誰知道你的新功能會不會出包)
* 安全性，可靠性，服務是否在這次更新後正常運作，有沒有 bug，會不會造成其他功能 crash ?

我自己的經驗真的是這樣，兩個團隊的目標不一樣，之前我在管理團隊中就是不太想去做 patch update，只希望服務能夠穩定提供，甚至是這台伺服器就都不要去動他。但在開發團隊時，又很想趕快做出新功能後，進行更新並提供給使用者做使用。(~~工讀時兩個團隊都待過。~~)

但不可能都不推出新功能阿，只是，為了服務的可靠性，管理團隊會使用許多方式來檢查、驗證或測試新的功能是否有問題，或是會造成其他問題的可能。  
開發團隊也可以透過發布更少的 patch、flag flips、incremental updates 與 cherrypicks 來減緩或降低這些事情發生的次數。 

{{< admonition info "Info">}}
即使兩個團隊的價值觀不盡相同，但大家都是為了提供更好且穩定的服務，請盡量不要因為目標不同而真的吵起來...
{{</ admonition >}}

## In Google

基本上在 Google 內部是透過軟體工程師建立系統來運行由系統管理員手動執行的工作。  

這個 SRE 團隊一半是軟體工程師，一半是具備 SRE 技能的工程師，SRE技能基本上是 Linux internals 與 networking(TCP/IP)(但也會軟體工程)。
基本上是以軟體工程的方式，來處理系統管理的問題，盡可能的使用軟體設計實現自動化以降低手動執行的工作，甚至是獲得自動化的系統，這個系統能夠自動運行並自我修復。

限定 50% 的時間會用在管理，另外的時間會用於實際開發，一來是可以參與主要工程，了解這次的開發是為了甚麼，是如何運作的，有沒有能再更好的地方，當知道系統怎麼維運時，在進行開發上就會知道那些地方是要注意的，並且這些經驗可以與開發團隊分享，達到提升整體的技能之目的。

偷渡個 SRE 的技能書 (~~面試用書~~)，**但請記得，軟體開發仍是最重要的技能。**

1. **Algorithms and data structures** :   
Book: Cracking the Coding Interview, 6th Edition, ISBN 0984782869
(arrays, stacks, linked lists, binary trees, graphs, recursion)
2. **System design** :  
Book: System Design Interview – An insider's guide, 2nd Edition, ISBN 979-8664653403
3. **Linux internals** :   
Book: Advanced Programming in the UNIX Environment, 3rd Edition, ISBN 0321637739
4. **TCP/IP** :  
Book: TCP/IP Illustrated, Volume 1: The Protocols, ISBN 0321336313


## SRE 宗旨
負責並專注在服務的可用性、延遲、性能、效率、變更管理、監控、緊急事項回應與容量規劃。  
不僅僅是專注在系統服務環境，還包含開發團隊、測試團隊與使用者等。

## 參考資料
1. [SRE-Books](https://sre.google/books/)