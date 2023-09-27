---
slug: "tools_1"
title: "【TroubleShooting】故障排除的工具介紹與使用方式--網路中斷"
date: "2022-06-11"
lastmod: "2022-06-11"
tags: ["Infra","Troubleshooting","debug","tools"]
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

基本上分享自己在進行網路中斷的故障排除時，經常使用到的工具與使用方式。
<!--more-->
## 前言
在網路中斷或是緩慢時，總需要有人去處理，緩慢還可以用 Google 大法協助，不過個人認為緩慢比中斷還要難排除。  
這篇會先單純以網路斷線為主題撰寫，分享幾個自己常用的工具。  
~~(只要修好網路斷線，剩下的問題就拿去問 Google !!!)~~

* ifconfig, ipconfig, ip addr
* ping
* traceroute(tracert), tcptraceroute
* dig, nslookup
* mtr
* netstat
* nmap
* nc
* curl, wget
* tcpdump, wireshark

這邊基本上會先說明一些工具，來驗證兩地之間的網路是正常連線，還沒考慮到應用層與緩慢的問題。    
關於應用層與緩慢的問題，會在後續的文章中加以說明。

## ifconfig (ip addr, ipconfig on Windows)
我一直以來都是喜歡用 ifconfig ，或是在 Windows 上使用 ipconfig，因為這兩個指令相近，自然而然就會在 Windows 上使用 ifconfig ~~（不要笑）~~  
後來 Debian 改版之後，不再預安裝 `net-tools`，所以改用 ip addr 這個指令。  
Windows 上的 ipconfig 會建議再加上 `-all` 一次獲得更多資訊。

基本上這些指令都可以看網路卡的 IP 資訊，有時是 DHCP Server 故障沒有取到 IP address，或是 IP address 設錯等問題，改一下就排除了。

------

## ping 
`ping` 應該是最常被用到的工具之一，我相信很多人做過這件事：
```bash
$ ping 8.8.8.8
```

如果我們的 DNS 正常，也可以透過 FQDN 進行：
```bash
$ ping aws.amazon.com
PING dr49lng3n1n2s.cloudfront.net (143.204.75.75): 56 data bytes
64 bytes from 143.204.75.75: icmp_seq=0 ttl=233 time=49.306 ms
64 bytes from 143.204.75.75: icmp_seq=1 ttl=233 time=561.229 ms
64 bytes from 143.204.75.75: icmp_seq=2 ttl=233 time=53.671 ms
64 bytes from 143.204.75.75: icmp_seq=3 ttl=233 time=68.874 ms
64 bytes from 143.204.75.75: icmp_seq=4 ttl=233 time=49.972 ms
```
至於為什麼變成是 `PING dr49lng3n1n2s.cloudfront.net` 呢，~~CNAME什麼的下回解析~~  
我自己比較加上的參數應該是 `-c` or `-m`，畢竟使用 `ping` 通常只是用來驗證網路是否有連線而已。
```bash
$ ping -c 10 1.1.1.1 # 對 1.1.1.1 進行 10 次 icmp 封包的傳送並等待回覆。
$ ping -m 10 1.1.1.1 # 將 ttl 改為 10 後進行 10 次 icmp 封包的傳送並等待回覆。
```

我自己是用 ping 的法則大概是這樣的順序，基本上就是由 Client 慢慢到 Server：
* ping localhost 
* ping [host IP address]
* ping [Gateway IP address]
* ping [Server IP Address]

但沒有一定的方法，我覺得依據個人喜歡的方式使用即可，畢竟我們的目標是故障排除而不是在那邊計較先後順序。

------

## traceroute (tracert on Windows)
透過更改 ttl 的方式，去 trace 網路路徑的工具，可以讓我們較容易知道網路到哪個 node 時出問題：
```bash
$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 64 hops max, 52 byte packets
 1  172.20.10.1 (172.20.10.1)  1.182 ms  0.606 ms  0.580 ms
 2  * * *
 3  * * *
 4  * * *
 5  tpdb-3312.hinet.net (210.65.126.98)  38.918 ms  19.277 ms  25.854 ms
 6  tpdb-3031.hinet.net (220.128.1.254)  25.584 ms
    tpdb-3031.hinet.net (220.128.1.114)  17.709 ms
    tpdb-3031.hinet.net (220.128.1.254)  23.210 ms
 7  220-128-9-121.hinet-ip.hinet.net (220.128.9.121)  21.327 ms *  57.736 ms
 8  tpdt-3302.hinet.net (220.128.12.61)  29.946 ms
    pcpd-4102.hinet.net (220.128.13.109)  28.238 ms
    220-128-13-169.hinet-ip.hinet.net (220.128.13.169)  20.401 ms
 9  72.14.202.178 (72.14.202.178)  19.526 ms
    72.14.209.178 (72.14.209.178)  34.970 ms
    142.250.169.122 (142.250.169.122)  26.003 ms
10  * * *
11  dns.google (8.8.8.8)  24.621 ms  19.687 ms
    209.85.245.64 (209.85.245.64)  21.766 ms
```

### tcptraceroute
相較於 traceroute 使用 icmp，這是使用 tcp 去進行 trace 的工具。  
因為在現代網路上，防火牆通常會阻擋 `icmp` 的封包進入，  
然而我們在瀏覽網頁或是發送信件時，防火牆大都會讓 tcp 封包進入，為了進行 3-way handshake 進行後續的連線建立，client 首先會發送 SYN 封包。  
而 tcptraceroute 就是透過這個 `TCP-SYN` 封包進行 trace 而實現。~~讓你可以做更深入的測試?~~

通常你的電腦內不會有這個工具，必須自己去下載安裝。  
使用方式也很簡單，基本上怎麼用 traceroute 就怎麼使用 tcptraceroute。

比較常用的方式是在加上 port number ，針對你遇到的問題，或是你想針對特定的某個服務，去進行 trace。
```bash
& sudo traceroute -p 443 example.com
```

{{< admonition note "Note">}}
可以自己找個目標，嘗試並比較 traceroute 與 tcptraceroute 的差異。
{{</ admonition >}}

------

## dig (nslookup)
這邊會提到 dig 是因為有時候是 DNS 紀錄錯誤，導致 host 一直連到別的 IP address，當然就無法連線囉！！  
基本上我在網路連線中斷排除時，會用這個單純看目標 IP address 看是否正確而已。  
其實兩個工具很類似，但我認為 `dig` 給的資訊量較大，也有較多的功能可以使用，所以這邊只會講 dig。

dig 也不是預先安裝的工具，那要怎麼用？？ ~~(安裝啊)~~  
```bash
### macOS
$ brew install bind
### Debian (你可能會遇到更新與權限問題)
$ apt install dnsutils
```

裝好了我們就趕快測試一下！！
```bash
$ dig example.com

; <<>> DiG 9.10.6 <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41551
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		4502	IN	A	93.184.216.34

;; Query time: 20 msec
;; SERVER: 172.20.10.1#53(172.20.10.1)
;; WHEN: Sun Jun 12 21:38:04 CST 2022
;; MSG SIZE  rcvd: 56
```
我們可以透過 `@` 來指定某台 DNS server 做查詢。
```bash
$ dig example.com @168.95.1.1
```

或是指令紀錄的種類去查詢：
```bash
$ dig gmail.com MX
```

{{< admonition note "Note">}}
試試看吧！  
$ dig example.com +trace
{{</ admonition >}}


## 小結論
這篇的工具大部分是我用來驗證，我的 host 是否能到達我想去的目標。  
當我們可以到達目標時，問題大多數會是在 Application Layer 或是 Transport layer 上。  
此時就會使用別的工具去做故障排除了。


## 參考資料
1. [tcptraceroute](https://linux.die.net/man/1/tcptraceroute)
2. [Dig Command in Linux (DNS Lookup)](https://linuxize.com/post/how-to-use-dig-command-to-query-dns-in-linux/)