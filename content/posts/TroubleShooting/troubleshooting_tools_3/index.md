---
slug: "tools_3"
title: "【TroubleShooting】故障排除的工具介紹與使用方式--Slowly"
date: "2022-07-28"
lastmod: "2022-08-12"
tags: ["Infra","Troubleshooting","debug","tools","mtr","tcpdump","iperf","wireshark"]
sidebar: auto 
categories: ["TroubleShooting"]
no_comments: false
draft : false
resources:
- name: "featured-image-preview"
  src: "troubleshooting_header.webp"
- name: "featured-image"
  src: "troubleshooting_header_content.jpg"
---

簡單提一下網路緩慢時的處理方式與工具。
<!--more-->
## 前言
自己在處理網路緩慢的問題經驗並不多，但基本上自己會先往幾個方向去思考問題點在哪：  
* 高延遲（High Latency），想一想我們前面提到 3 way handshake 時，假設一來一往，每一個步驟都要花 1000ms，建立一個連線要 3 秒鐘，這樣當然會慢啦。
* 使用較小的 MSS，意味著在同樣的時間內所輸的資料較少，這樣也會感覺慢慢的。
* 太多人使用，壅塞了！
* ~~其他我可能沒想到的原因。~~

因為緩慢可能會有很多原因，可以透過一些工具來幫助自己釐清這些問題。  
以下會對這些工具做一些簡單的介紹。

* mtr
* tcpdump, wireshark
* iperf

## mtr
mtr 就是把 traceroute(tracert) 跟 ping 組合起來，主要是用來觀察 latancy 或是 packet loss% 的工具。  
其實就很像 traceroute，但 mtr 顯示的資訊更豐富，並會計算所有 hop 回應的百分比與回應時間。  

```bash
$ mtr 8.8.8.8
```
那他會帶你進入一個介面做觀察，按 q 可以離開。  

{{<image src="mtr_1.png" caption="第一次看到 mtr 大概是這樣子" width="100%">}}

主要可以觀察到 packet 的遺失率，如果說有幾個不連續的幾個 node 掉了幾個封包，其實是不會造成太大的影響，但如果是某一個 node 後所有的 node 都有封包遺失，那一個 node 則會對整個傳輸造成影響，也就會讓使用者感受到緩慢，可以推測那個 node 上有問題。

另外可以觀察 icmp 封包的傳輸延遲，雖然 traceroute 也做得到，但 mtr 他提供的資訊比較多啦。
* Last : 最後一次的延遲數值
* Avg : 平均的延遲數值
* Best : 最佳的延遲數值（最短）
* Wrst : 最糟的延遲數值（最長）
* StDev : 標準偏差（Standard Deviation），愈大代表這個 node 愈不穩定。

以上圖的例子來說，第二個點的延遲就有點糟且不太穩定。  

另外可以加入一些參數做不同的使用：
```bash
## -n 可以強制以 IP address 顯示所有 node
$ mtr -n 8.8.8.8
```
{{<image src="mtr_2.png" caption="" width="100%">}}

前面我們提到 tcptraceroute，那在 mtr 上可以使用 tcp 或 udp 嗎？  
**可以的 !!** 有時候也需要觀察 tcp 或 udp 資料在這段網路上的傳輸品質。  
```bash
$ mtr --tcp 8.8.8.8 
$ mtr --udp 8.8.8.8
```

我想改變封包的 size 可以嗎？ `-s` 會幫助你！  
可以透過更改封包的大小來觀察網路使用狀況。  
```bash
$ mtr -s [PACKETSIZE] 8.8.8.8
```

最後，可以透過 `-r` 這個參數做紀錄。
```bash
### 預設會送 10 次做觀察。
$ mtr -r 8.8.8.8 > [filename] 
### 透過 -c 調整次數，這樣他就會送 100 次。
$ mtr -r -c 100 8.8.8.8 > [filename] 
### 這樣他就會在背景送 1000 次，然後你可以去喝咖啡。
$ mtr -r -c 1000 8.8.8.8 > [filename] & 
```

## tcpdump, Wireshark

tcpdump 是一個功能強大的，而且也是使用最廣泛的 sniffer 與分析工具，用來捕捉或過濾在特定介面上接收或傳輸的 TCP/IP packet。  
基本上在多數的環境中都可以使用，並且可以將捕捉的結果儲存為 **.pcap** 格式，可以透過 Wireshark 做更進一步的分析。  
坦白說，tcpdump 也是會在網路無法連線時使用到，像是 ssl 無法建立，或是 VLAN tag 設定錯之類的情形，我會把它放在這是因為，我自己在看 MSS 時會用到。MMS 的大小會影響傳輸效能。或觀察封包的往來時間。  
Wireshark 也具有相同的功能，並提供圖形化介面做更容易的使用。
這邊只會介紹 tcpdump，基本上，環境預設是不會有這個工具，所以我們要先自行安裝。
```bash
# macOS(OSX)
$ brew install tcpdump
# Debian, Ubuntu
$ sudo apt-get install tcpdump
```

首先，我們可以選擇指定的介面捕捉封包。透過 `-i` 這個參數可以指定介面。
```bash
$ tcpdump -i eth0
16:59:22.920016 IP 172.20.10.4.50011 > a104-115-254-134.deploy.static.akamaitechnologies.com.https: Flags [.], ack 213539838, win 2048, length 0
16:59:22.920017 IP 172.20.10.4.50007 > a173-222-181-125.deploy.static.akamaitechnologies.com.https: Flags [.], ack 2165220779, win 2048, length 0
16:59:22.920017 IP 172.20.10.4.50003 > 137.155.120.34.bc.googleusercontent.com.https: Flags [.], ack 281486443, win 2048, length 0
```

那我們要如何知道，有哪些介面可以監聽呢？~~ifconfig~~  
tcpdump 提供 `-D` 這個參數，讓我們可以知道可以使用哪些介面。
```bash
$ tcpdump -D

1.en0 [Up, Running, Wireless, Associated]
2.p2p0 [Up, Running, Wireless, Not associated]
3.awdl0 [Up, Running, Wireless, Associated]
4.llw0 [Up, Running, Wireless, Associated]
5.utun0 [Up, Running]
6.utun1 [Up, Running]
7.lo0 [Up, Running, Loopback]
8.en1 [Up, Running, Disconnected]
9.en2 [Up, Running, Disconnected]
10.gif0 [none]
11.stf0 [none]
12.XHC20 [none]
13.bridge0 [none, Disconnected]
14.en4 [none, Disconnected]
```

並可以透過 `-n` 再進一步，只捕獲這個介面上 IP 的封包。
```bash
$ tcpdump -n -i en0
```

另外還可以透過 `-XX` 這個參數，顯示封包的數據內容，並以 HEX 與 ASCII 格式顯示。  
```bash
$ tcpdump -XX -i en0

17:04:15.996062 IP6 2001-b400-e35d-11b9-e87c-0c6e-2d6e-27bf.emome-ip6.hinet.net.50145 > 2403:300:a41:b02::7.https: Flags [P.], seq 3353679317:3353679348, ack 252170061, win 2048, options [nop,nop,TS val 2584095760 ecr 718123086], length 31
	0x0000:  feaa 8116 f864 80e6 501d eed8 86dd 602b  .....d..P.....`+
	0x0010:  0700 003f 0640 2001 b400 e35d 11b9 e87c  ...?.@.....]...|
	0x0020:  0c6e 2d6e 27bf 2403 0300 0a41 0b02 0000  .n-n'.$....A....
	0x0030:  0000 0000 0007 c3e1 01bb c7e5 15d5 0f07  ................
	0x0040:  cf4d 8018 0800 0bba 0000 0101 080a 9a06  .M..............
	0x0050:  2c10 2acd b04e 1503 0300 1a00 0000 0000  ,.*..N..........
	0x0060:  0000 022b 2353 39dc 701e eb0a d4f2 219e  ...+#S9.p.....!.
	0x0070:  6eca 679d 37                             n.g.7
```

前面提到，可以將這些紀錄儲存為 **.pcap** 格式，只需要透過 `-w` 即可使用。
```bash
$ tcpdump -w myFileName.pcap -i en0
```

既然可以寫入，那應該也可以讀取吧？  
沒錯，透過 `-r` 這個參數可以讀取 **.pcap** 檔案。
```bash
$ tcpdump -r myFileName.pcap
```

接下來是一些更細部的參數，可以讓我們根據情境做調整：
```bash
# 只抓 TCP
$ tcpdump -i en0 tcp

# 只抓某個 port number 
$ tcpdump -i en0 port 22

# 只抓來源 IP 為 8.8.8.8 的
$ tcpdump -i en0 src 8.8.8.8
# 只抓目的 IP 為 168.95.1.1 的
$ tcpdump -i en0 src 168.95.1.1
```

因為這篇文章，主要是工具的使用，更細一步的分析，往後有時間會再撰寫吧。

## iperf
iperf 是一種測量網路上最大頻寬的工具，可以調整 timing, buffer 與 protocol(TCP, UDP, ICMP ...)，並產生頻寬與損耗等相關報告。  
一般來說，使用 iperf 要有兩個端點(node)，透過兩個端點進行測量。(Server, Client)

``` bash
# Server
$ iperf3 -s

# Client
$ iperf3 -c [Server IP address]
```

可...可是我只有一台電腦，那也可以透過公開的 iperf server 做測試。[連結](https://iperf.fr/iperf-servers.php)  
不過我自己用了幾個，都是在忙碌中...

比較常用的參數有 `-p`, `-u` 與 `-t` 等。
```bash
$ iperf -p 8888 -c [server] # 改 port number
$ iperf -u 8888 -c [server] # 改用 UDP，預設是 TCP
$ iperf -t 60 -c [server] # 更改傳輸的總時間
```

但這個工具我會建議用在做單個點的驗證，因為多個點時，使用 iperf 只會容易知道整體鏈路情況，但實際哪個點或哪幾個點有問題，可能還是要用 mtr 或類似的工具做補助。

## 參考資料

1. [mtr](https://github.com/traviscross/mtr)
2. [tcpdump](https://www.tcpdump.org)
3. [wireshark](https://www.wireshark.org)
4. [iperf](https://iperf.fr)
5. 各個工具的 `-h, --help` 指令，`man` 的說明。


