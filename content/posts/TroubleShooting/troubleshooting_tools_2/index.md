---
slug: "tools_2"
title: "【TroubleShooting】故障排除的工具介紹與使用方式--服務中斷"
date: "2022-06-17"
lastmod: "2022-06-20"
tags: ["Infra","Troubleshooting","debug","tools","netstat","nmap","nc"]
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

這篇會簡單提一下自己在網路正常卻無法存取服務時，自己會使用的工具與使用方式。
<!--more-->
## 前言
上一篇文章基本上是自己在找網路中斷的問題時會用到的工具，但許多時候網路是正常的，問題點則是在伺服器或服務上。  
我通常會看服務是否有啟用，並根據相對應的服務做後續的處理，下面是我在這種情境中，我經常用到的工具。

* netstat(ss)
* nmap
* nc

## netstat
基本上許多作業系統都已預設安裝 netstat 這個工具，用來顯示目前網路連結的狀態 (connetion)。  

* -s : 會顯示統計數據。可以在後面加上 t(tcp)或是u(udp) 對於協定做進一步的查詢。
```bash
$ netstat -st
IcmpMsg:
    InType0: 33
    InType11: 319
    OutType3: 7
    OutType8: 352
Tcp:
    50 active connection openings
    14 passive connection openings
    0 failed connection attempts
    1 connection resets received
    19 connections established
    7193 segments received
    7100 segments sent out
    0 segments retransmitted
    0 bad segments received
    2 resets sent
UdpLite:
TcpExt:
    39 TCP sockets finished time wait in fast timer
    31 delayed acks sent
    1818 packet headers predicted
    2219 acknowledgments not containing data payload received
    1497 predicted acknowledgments
    TCPBacklogCoalesce: 11
    TCPRcvCoalesce: 293
    TCPAutoCorking: 28
    TCPOrigDataSent: 1900
    TCPKeepAlive: 1830
    TCPDelivered: 1950
IpExt:
    InOctets: 1865047
    OutOctets: 207989
    InNoECTPkts: 2521
```
* -r : 可以顯示目前的路由表(routing table)。也可以透過 route 指令去顯示。(依據作業系統可能會是 route PRINT 或 route -n)
```bash
$ netstat -r
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         10.0.2.2        0.0.0.0         UG        0 0          0 enp0s3
10.0.2.0        0.0.0.0         255.255.255.0   U         0 0          0 enp0s3
```
* -p : 可以觀察到是哪個 process 在存取。在 Windows 上的會是選擇觀察哪種協定(TCP, UDP)。

netstat 還有許多參數可以使用，請依據自己遇到的情境挑選適合的參數。

### ss
基本上他就是 netstat 的加強版，基本上參數的使用也跟 netstat 差不多，個人覺得 ss 比較好用，看起來比較舒服一點，但 netstat 通常會是預設安裝，而 ss 則不是。 
ss 目前在 iproute2 中，要使用請安裝 iproute2 :
```bash
$ sudo apt-get install iproute2
```
{{<image src="ss.JPG" caption="" width="100%">}}


### TCP states
**請你暫時忍耐一下，關閉夜間模式 !**  

{{<image src="tcp-state-diagram-v2.svg" caption="如同聖經般的 TCP 狀態圖。" width="100%">}}

之所以會放這張圖是因為，有時候是 Service 在設計上時有問題，又或者是遇到服務被攻擊而造成的 breakdown，此時可以透過狀態來分析並進行後一步的處理。
* 好比說我在 Server 上看到許多 `SYN_RECV` 狀態，可能代表在 3-way handshake 上遇到問題，Server 收到了 Client 發送的 SYN packet，於是 Server 的狀態為 `SYN_RECV` 並回傳一個 SYN+ACK 的封包並等待對方回傳 ACK，然而一直沒收到就保持在這個狀態許久。  
通常遇到這種情形，可以先懷疑是 TCP SYN FLOOD 攻擊。

## nmap
坦白說他是掃描 port 的工具，是自己以前在接觸 cyber security 時所學到的。  
我會提到 nmap 是因為他可以讓我快速知道遠端機器是否存活，服務是否有掛上。  
除此之外，nmap 也很常使用在網路安全上，並可以透過 script 和微調設定，掃瞄系統並檢查常見的漏洞，或是找到在常用的伺服器上具有的致命設定。(Web, Database and Mail ... etc)  
~~有點扯太遠了，我還是拿來看 port 有沒有開即可 ...~~

{{< admonition warning "Warning">}}
請不要對某個特定目標用 nmap 執行大量的 scan，你有可能因此被 block。  
{{</ admonition >}}

```bash
# 最常使用的 scan command.
$ nmap -sS example.com 

# UDP Scan
$ nmap -sU example.com

# 針對某幾個 port number 做 Scan
$ nmap -sS -p 80,443 8.8.8.8
Starting Nmap 7.80 ( https://nmap.org ) at 2022-06-16 19:34 HST
Nmap scan report for dns.google (8.8.8.8)
Host is up (0.0014s latency).

PORT    STATE    SERVICE
80/tcp  filtered http
443/tcp open     https

Nmap done: 1 IP address (1 host up) scanned in 1.23 seconds
```

## nc (netcat)
一個很強但我不太會用的工具。(功能太多)  
會簡單提一下我比較常用的功能:

確認 Port 是否有開啟：
```bash
$ nc -v example.com 80
Warning: inverse host lookup failed for 93.184.216.34: Unknown host
example.com [93.184.216.34] 80 (http) open
```
如果沒有開啟則會得到這個結果：
```bash
$ nc -v example.com 2022
nc: connect to 192.168.233.208 2022 (tcp) failed: Connection refused
### 也有可能像到進黑洞，下完指令後就卡在那邊><
### 這時要看你的目標的防火牆或是相關的規則，進而導致有不同的結果。
```
也因此 `nc` 可以用來做 port scanning ! 
不過這不代表可以取代 `nmap`，我覺得工具各有優缺點，像我做 scanning 應該還是會用 `nmap`。

{{< admonition warning "Warning">}}
再一次提醒，請不要對某個特定目標執行大量的 scan，你有可能被 block。  
{{</ admonition >}} 

```bash
# nc -vnz [parameter] [Target IP address] [port range]
# e.g.
$ nc -vz -w 1 8.8.8.8 2000-3000
## -w 是 timeout。
## 這樣會是測 TCP 的，如果想要測 UDP ，就給他個 U 吧 !!
# e.g.
$ nc -vzu  8.8.8.8 2000-3000
```

如果你能同時控制兩台電腦，nc 也很常被用來傳輸檔案，  
這邊要注意一下優先順序， Receiver 要先開啟後再由 Transmitter 傳輸。  
~~(冰箱沒有打開，要怎麼把大象放進去???)~~
```bash
## Receiver
$ nc -l -p 3333 > name.file

## Transmitter
$ nc [receiver IP address] 3333 < name.file
```
自己比較常常用來把系統的日誌檔扔到自己的電腦，做一些分析~~(抱歉我的 grep, awk, sed 沒學好)~~  
或許用 `scp` 與其他的 command 會比較快一點，但多學一個方式也不是壞處。  
~~可以偷偷把 Linux 上的 shadow 跟 passwd 拿出去然後做一些壞壞的事情(?~~

也可以連線到服務，做簡單的測試：
```bash
$ nc example.com 80
GET / HTTP/1.1 # 這一行是要自己 key in 的。
# 但如果要測網頁，我會建議你用 curl, wget 或是我們常用的 browser 會比較快速也比較方便一點。
```

## 小結論
在確定網路正常，但使用者仍無法連線至服務的情況下，我會使用上述的工具，快速檢查伺服器端的上的 service port 是否正常開啟，若不是正常開啟，又會是甚麼原因造成的。  
並根據不同的服務，進一步使用不同的工具來做故障排除。  
像是 DNS 就再用 dig 看一下紀錄是否正常，Web 就用 curl, wget 等。  
當然，還有很多原因會造成網路正常但服務無法存取，至少希望能透過這些工具進一步的判斷故障點在何處，或是大概的範圍。

最後，nmap 與 netcat(nc)都是很有力的工具，甚至可以單獨為一本書，在參考資料我會提到相關書籍可以參考。

## 參考資料
1. [Windows-commands: netstat](https://docs.microsoft.com/zh-tw/windows-server/administration/windows-commands/netstat)
2. [manpages-ss, Debian testing](https://manpages.debian.org/testing/iproute2/ss.8.en.html)
3. [Nmap Network Exploration and Security Auditing Cookbook](https://www.amazon.com/Nmap-Exploration-Security-discovery-fingertips/dp/1786467453)
4. [Netcat Power Tools](https://www.amazon.com/Netcat-Power-Tools-Jan-Kanclirz/dp/1597492574)
