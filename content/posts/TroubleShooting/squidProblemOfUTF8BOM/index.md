---
slug: "Squid_UTF8BOM"
title: "【TroubleShooting】Invalid Response on Squid(Duo to BOM)"
date: "2023-06-01"
lastmod: "2023-06-01"
tags: ["Infra","squid","encoding","UTF-8","UTF-8 with BOM","UTF8BOM"]
sidebar: auto 
categories: ["TroubleShooting"]
no_comments: false
draft : false

---

關於遇到 Squid 因為 UTF8-BOM 而導致 Invalid Response 的問題。

<!--more-->
## 問題
客戶端使用 Squid 作為 proxy server 並連線到一台帶有網頁管理介面的 Layer3 Switch，發生了錯誤導致網頁無法瀏覽。  
但是，不透過 Squid，直接連線到設備時，網頁又正常運作。  
{{<image src="error.png" caption="Squid 給我的錯誤訊息，真不想看到這個。(已刪除敏感訊息)" width="100%">}}

此時要先釐清問題點是在設備上，還是 Squid 上。~~如果是 Squid 就不甘我的事了！！~~

## 過程

### 拓墣
拓墣是 Squid 跟 Switch 之間直接連線。客戶端再與 Squid 直接連線。  
Switch 192.168.0.1  <--->  Squid 192.168.0.111  <--->  Client 192.168.0.10

### Log
因為看到錯誤訊息是 Invalid Response，在想應該是 Squid 跟 設備之間有傳輸問題，先看了一下 `/var/log/squid/access.log` 跟 `var/log/squid/cache.log`  
(這邊我會調整一下資訊，會與我真實遇到的情況不同，但我盡可能保留原本的樣子。) 

```bash
$ sudo tail -f /var/log/squid/access.log
1685583425.309     89 192.168.0.10 TCP_REFRESH_IGNORED/200 10211 GET http://192.168.0.1/login.lsp - HIER_DIRECT/192.168.0.1 text/html
1685583433.755     97 192.168.0.10 TCP_MISS/200 563 POST http://192.168.0.1/login.lua - HIER_DIRECT/192.168.0.1 application/json
1685583433.871    103 192.168.0.10 TCP_REFRESH_FAIL_ERR/502 5077 GET http://192.168.0.1/main.lsp - HIER_DIRECT/192.168.0.1 text/html
```

```bash
$ sudo tail -f /var/log/squid/cache.log
2023/05/31 09:38:12 kid1| WARNING: HTTP: Invalid Response: Bad header encountered from http://192.168.0.1/main.lsp AKA http://192.168.0.1/main.lsp
```

在觀察到 `access.log` 產生 502 錯誤時， `cache.log` 也紀錄了一筆警告資訊，遇到錯誤的 header。
> **... Bad header encountered ...**

好的，看起來是 header 有問題，這邊可以用 `curl` 去觀察 header，也可以在 squid.conf 中加入 `log_mime_hdrs on` 這個參數，這樣會紀錄傳輸時的 header。

```sh
1685499970.322     95 192.168.0.10 TCP_MISS/200 588 POST http://192.168.0.1/login.lua - HIER_DIRECT/192.168.0.1 application/json [User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/113.0\r\nAccept: application/json, text/javascript, */*; q=0.01\r\nAccept-Language: zh-TW,zh;q=0.8,en-US;q=0.5,en;q=0.3\r\nAccept-Encoding: gzip, deflate\r\nContent-Type: application/x-www-form-urlencoded; charset=UTF-8\r\nX-Requested-With: XMLHttpRequest\r\nContent-Length: 119\r\nOrigin: http://192.168.0.1\r\nConnection: keep-alive\r\nReferer: http://192.168.0.1/login.lsp\r\nCookie:\r\nHost: 192.168.0.1\r\n] [HTTP/1.1 200 OK\r\nTransfer-Encoding: chunked\r\nX-Frame-Options: SAMEORIGIN\r\nDate: Wed, 02 Jan 2019 19:36:58 GMT\r\nServer: lighttpd\r\nContent-Type: application/json\r\nExpires: Wed, 02 Jan 2019 19:36:58 GMT\r\nLast-Modified: Wed, 02 Jan 2019 19:36:58 GMT\r\nCache-Control: no-cache\r\nSet-Cookie:\r\n\r]
1685599970.433     84 192.168.0.10 TCP_MISS/502 5078 GET http://192.168.0.1/main.lsp - HIER_DIRECT/192.168.0.1 text/html [User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:109.0) Gecko/20100101 Firefox/113.0\r\nAccept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8\r\nAccept-Language: zh-TW,zh;q=0.8,en-US;q=0.5,en;q=0.3\r\nAccept-Encoding: gzip, deflate\r\nConnection: keep-alive\r\nReferer: http://192.168.0.1/login.lsp\r\nCookie: \r\nUpgrade-Insecure-Requests: 1\r\nHost: 192.168.0.1\r\n] [HTTP/1.1 502 Bad Gateway\r\nServer: squid/4.13\r\nMime-Version: 1.0\r\nDate: Wed, 31 May 2023 02:26:10 GMT\r\nContent-Type: text/html;charset=utf-8\r\nContent-Length: 4710\r\nX-Squid-Error: ERR_INVALID_RESP 0\r\nVary: Accept-Language\r\nContent-Language: zh-tw\r\n\r]
```

**Header 看起來沒有問題呀！！！**  
但是他就是寫 Header 有錯啊！！！

這時我做了兩件事情：  
1. 問我的大神朋友。
2. 通靈！土法煉鋼！wireshark 用起來。

最後都指出了一樣的事情。UTF-8 with BOM(Byte Order Mark)。

## 轉折點


觀察了 Squid 回傳 200 跟跟 Squid 回傳 502 前，Switch 的傳輸內容，發現在 Header 的部分多了 `ef bb bf`。  
這三個 Byte 扔到 Google 發現是 BOM(Byte Order Mark)，是關於編碼的 Mark。  
但我在 `curl` 跟 `access.log` 都沒看到，隱隱約約覺得是這個 BOM 而導致 header 有問題。   

{{<image src="no_bom.png" caption="正常結果。" width="100%">}}

{{<image src="have_bom.png" caption="有 BOM 的結果。" width="100%">}}

------

同時我的大神朋友也回傳給我了他的分析結果。  
~~大神的 wireshark 會給 Expert info，不愧是大神！！！~~

{{<image src="expert_ana.jpeg" caption="Illegal Characters." width="100%">}}

到這邊就確定是 BOM 在搞事，影響了 header，重新編碼有問題的網頁後，這個問題就解決了。

{{<image src="diff.jpeg" caption="Code review 時還花了一堆時間解釋..." width="50%">}}

雖然還有個問題，是要找出哪些網頁也是用這個編碼方式儲存，但這個屬於小問題好解決啦～

```bash
$ find . -type f -name "*.html" | xargs file | grep "UTF-8 Unicode (with BOM)"
./path/to/file/index.html:              HTML document, UTF-8 Unicode(with BOM) text, with CRLF line termintors
# 建議先用自己環境的 find 跟 file，可能需要依據結果做調整這段 script。
# html 可以依據自己需求更改檔案類型或名稱，反正根據需求使用 regex 即可。
```

~~我要來問大神為什麼他的 wireshark 像個 expert 而我的不像了。~~

## Byte Order Mark (BOM)

### 簡介
網路上其實很多資源說的蠻清楚的，我這邊就簡單說明一下。

Byte Order Mark 常被用來當做標示檔案是以 UTF-8、UTF-16或 UTF-32 編碼的記號。  
他會在檔案開始處使用特定字元來標記，一般常用的編輯器是看不到這些特定字元，通常要使用 binary 或是 hex 的方式來打開，像是用 `hexdump` 或是 `vim` 搭配 `xxd`。  
要不要使用看個人的設計與需求，但經常會干擾應用程式的運作，而導致檔案出現空白或是亂碼的情形。


### 建議
1. 檔案中有架構可以明確給出編碼的，盡量不要使用 UTF-8 with BOM 做編碼；但如果是不會明確給出編碼的，你要怎麼確保別人開啟時會用 UTF-8？說不定他用別的編碼方式呀，此時用 UTF-8 with BOM 是沒問題的。  
下面舉了兩個例子：html, xml
* html
```html
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
</head>
```
* xml
```xml
<?xml version = "1.0" encoding = "UTF-8" standalone = "no" ?>
```


2. 注意編輯器的編碼方式，~~據說 notepad 常常使用 UTF-8 with BOM。~~
3. 如果有發現使用錯誤的編碼方式，可以透過編輯器用別的編碼方式再儲存一次，但要記得編碼後作驗證，避免有其他問題發生。

## Wireshark 
因應這次的問題，請教了朋友，朋友也給了一些建議，故紀錄：

### Allow subdissector to reassemble tcp streams
{{<image src="unAllow.png" caption="這個預設是開啟的。" width="100%">}}
在傳輸時，往往會因為資料大於 MTU、MSS 的關係而分段傳輸。  
Wireshark 會將屬於同一個 PDU 的封包集合起來，關閉這個功能可以呈現較真實的傳輸情形。  

### Follow
{{<image src="follow.png" caption="追蹤。" width="100%">}}
透過這個功能，可以較容易觀察該 stream 收發了什麼資料。

## 參考資料
1. 我的神朋友 Oscar.
2. [What's the difference between UTF-8 and UTF-8 with BOM?](https://stackoverflow.com/questions/2223882/whats-the-difference-between-utf-8-and-utf-8-with-bom)
3. [wikipedia-位元組順序記號](https://zh.wikipedia.org/zh-tw/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F)
4. [「带 BOM 的 UTF-8」和「无 BOM 的 UTF-8」有什么区别？网页代码一般使用哪个？](https://www.zhihu.com/question/20167122)