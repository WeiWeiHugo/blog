---
slug: "upgradeDebian"
title: "有新的就要嘗試！教你如何從 Debian 10 升級至 Debian 11"
date: "2021-12-29"
tags: ["Linux","Debian"]
sidebar: auto 
footer: Copyright © 2022-present WeiWei, Powered by VuePress
categories: ["Infra"]
no_comments: false
resources:
- name: "featured-image-preview"
  src: "debian11.png"
- name: "featured-image"
  src: "debian11.png"
---

自己常用的 Linux 是 Debian，從 Debian 6(squeeze) 用到 Debian 10(buster)。Debian 在 2021.08.14 時釋出了 Debian 11(bullseye)，自己以往都是至官方連結下載新版的 ISO 重新安裝，這次則想說透過升級的方式進行更新，且這也是 Debian 著名的功能，故想於這次嘗試之。 ~~(其實是有Service在運作並做一些測試，不想重弄)~~
<!--more-->

## 準備
* 備份你所有的資料。（文件、圖片、**設定檔**、**驅動程式**等）
* 關閉所有的應用程式與服務。
* 關閉或刪除任何的個人套件庫(Personal Package Archive, PPA)，更新完成後再開啟或新增即可。
* 盡可能確保網路是穩定的。
* 保留一些時間進行升級。

## 更新步驟
### 更新現有的 package
 開啟terminal，輸入 `apt update && apt upgrade` 更新套件索引(package indexes)與套件（packges），需要先將更新目前的packages。
``` bash
root@server:~# apt update && apt upgrade

### 注意使用者身份，不是root的話，請加上sudo ###
user@server:~$ sudo apt update && sudo apt upgrade
```

### 更新來源(source.list)
修改apt的source.list，將來源由buster更改為新的bullseye。修改 `/etc/apt/source.list` (注意編輯權限)，不一定要使用vim，用自己喜歡的編輯器即可(e.g. emacs, nano)，**請記得編輯設定檔前記得備份！！！**
``` bash
user@server:~$ sudo cp /etc/apt/source.list /etc/apt/source.list.bak
user@server:~$ sudo vim /etc/apt/source.list
```
修改前的檔案

{{<image src="updateDebian_1.jpeg" caption="修改前的檔案" width="100%">}}

修改後的檔案

{{<image src="updateDebian_2.jpeg" caption="修改後的檔案" width="100%">}}

這裡再補充介紹 `main, contrib, non-free`
* main: 主要為完全符合 Debian 自由軟體指南(Debian Free Software Guidelines, DFSG)的所有package。
* contrib: 為開源但依賴於 non-free 的 package。
* non-free: 為不符合 Debian 自由軟體指南的 package。 

### 升級
先使用 `apt update` 確認第二步的編輯是否沒有問題，若無錯誤訊息再進行 `apt full-upgrade`。
``` bash
user@server:~$ sudo apt update
user@server:~$ sudo apt full-upgrade
```
途中會有一些訊息需要選擇。

顯示有關packages更新的新聞。按` q `退出。

{{<image src="updateDebian_3.jpeg" caption="Information about package update" width="100%">}}

Package configuration，請選擇 `<Yes>`

{{<image src="updateDebian_4.jpeg" caption="Package configuration" width="100%">}}

相關package的configuration，請依據需求設定，建議用 N 保留設定。

{{<image src="updateDebian_5.jpeg" width="100%">}}

Options description : 
* Y or I : install the package maintainer's version (安裝維護者版本的package，會覆蓋掉該檔案的設定。)
* N or O : keep your currently-installed verstion (保留目前安裝的版本。)
* D : show the differences between the versions (顯示版本之間的差異。)
* Z : start a shell to examine the situation (啟動一個shell檢查情況。)

完成升級後，重新啟動系統。
``` bash
user@server:~$ sudo systemctl reboot
```
### 確認
透過 `lsb_release -a ` 指令進行確認，可以發現已經從 buster 升級至 bullseye了。
``` bash
user@server:~$ lsb_release -a
Distributor ID: Debian
Description:    Debian GNU/Linux 11 (bullseye)
Release:        11
Codename:       bullseye
```

確認完成後，使用 `apt --purge autoremove` 刪除不再需要且不必要的packages。

``` bash
user@server:~$ sudo apt --purge autoremove
```

## 結論
第一次自己升級 Debian ，坦白說並沒有想像中的困難，有做好備份可以降低升級上的壓力。基本上都是`apt`在負責，只要有依據文件設定 `/etc/apt/source.list`，應該不會構成太大的問題。

## 參考資料
* `man apt`
* [debian.org](https://debian.org)
* [Debian管理者手冊](https://debian-handbook.info/browse/zh-TW/stable/)
* [Debian自由軟體指南,DFSG](https://www.debian.org/social_contract.html#guidelines)
* [main,contrib,nonfree解釋](https://askubuntu.com/questions/27513/what-is-the-difference-between-debian-contrib-non-free-and-how-do-they-corresp/27514#27514)