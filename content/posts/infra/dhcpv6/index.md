---
slug: "BuildDHCPv6OnDebian"
title: "在 Ubuntu/Debian 上架設 DHCPv6 服務"
date: "2022-11-30"
lastmod: "2023-02-16"
tags: ["DHCP","DHCPv6","DHCP Server","DHCPv6 Server"]
sidebar: auto 
categories: ["Infra"]
no_comments: false
draft : false

---
本篇文章將簡單說明，如何在 Ubuntu/Debian 上架設 DHCPv6 伺服器。

<!--more-->
## 前言
基本上是因為工作遇到問題，突然要使用 DHCPv6，但網路上的資料有點亂，嘗試了不少來源才試成功，故想紀錄下來供以後做參考。  
如果你進來是想看 configure 的，請直接跳到那邊就好了！  
~~其他相關的參數，以後有空再說明吧！~~

其實原先是想建一台 Windows Server 的虛擬機器，然後在虛擬機上面做 DHCPv6 Server，因為用 GUI 可以輕鬆解決，  
但因為手邊硬體關係，不太能使用 Windows Server 來做，於是選擇在 Ubuntu 上架設！~~（我也沒買 Windows Server ...）~~  
但因為自己順手的還是 Debian，故也有在 Debian 上架設一次，設定上並沒有差很多。

## Environment
這邊選擇在 Ubuntu 上做架設，但我也有在 Debian 上的架設完成，方式也類似，  
這邊會使用 isc-dhcp-server 做架設。
環境: 
```bash
$ user@ubuntu:~$ lsb_release -a
No LSB modules are avariable.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.1 LTS
Release:        22.04
Codename:       jammy
```

## Install
我們透過 `apt` 來安裝 `isc-dhcp-server` 這個套件。
```bash
$ sudo apt-get install isc-dhcp-server -y
```
因為現在的 Ubuntu 好像預設不會裝 net-tools，這邊也一併裝起來好了。主要是因為我們後續要在介面上設定一個 static IP address。~~其實因為習慣了 ifconfig。~~  
如果你有其他習慣使用的工具，你可以跳過這一步！
```bash
$ sudo apt-get install net-tools -y
```

接著在介面上設定 static IP address。  
enp0s3 是我的介面名稱。6666::a/64 是我要設定的 IPv6 address。  
**當然，你要依據你的環境來做適當的設定。**  

```bash
$ sudo ifconfig enp0s3 inet6 add 6666::a/64
```

大概到這邊，基本的環境已經安裝與設置完成。

## Configure
在 `/etc/dhcp` 底下有一個 `dhcpd6.conf` 的檔案，這個檔案就做為參考用即可，我們將它備份然後建立一個 `dhcpd6.conf`，
我喜歡在備份的檔案後面加上 `.bak`。 
```bash
$ cd /etc/dhcp
$ mv dhcpd6.conf dhcpd6.conf.bak
$ touch dhcpd6.conf
```

接著編輯 dhcpd6.conf，可以參考下面的 config。
```
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
subnet6 6666::/64
{
    range6 6666::1 6666::9;
}
```
* default-lease-time : 預設租期時間長度(秒)
* max-lease-time : 最大租期時間長度(秒)
* log-facility : 決定 facility 的等級(level)，這邊採用 local 7 即可。透過 log 可以較容易除錯。
* subnet6 : 網段的部分自行決定即可。

接著要讓 dhcpd6 有修改 dhcpd6.leases 的權限，我們使用 `chown` 來做設定。 
dhcpd6.leases 的路徑在 `/var/lib/dhcp` 底下。   
```bash
# 先注意一下你的環境中有沒有 /var/lib/dhcp/dhcpd6.leases 
# 我的環境中是有這個檔案，如果你的環境中沒有，請你用 touch 建立一個！
$ touch /var/lib/dhcp/dhcpd6.leases

# 設定一下擁有者跟擁有群組。
$ chown dhcpd:dhcpd /var/lib/dhcp/dhcpd6.leases 

# Debian 中可能會遇到沒有 dhcpd 這個 user 跟 group，
# 那就要控制擁有者跟群組對檔案的存取權限，使用 chmod。
$ chmod 666 /var/lib/dhcp/dhcpd6.leases
```

接著使用下面這行指令，進行手動啟動測試，他會套用你的設定檔與網路介面提供 DHCPv6 的服務。  
我們可以透過下面這個指令來觀察你的設備有沒有取得 IPv6 address 並且是在設定的範圍內，如果沒有問題就接續後面的步驟。
```bash
$ sudo dhcpd -6 -f -cf /etc/dhcp/dhcpd6.conf enp0s3
```
另外如果有遇到問題，可以使用 `journalctl -u isc-dhcp-server -e` 來觀察 log ，通常會告訴你哪裡有問題，處理掉該問題即可。

最後一步，我們要設定哪一個介面來處理 DHCP request。  
```bash
$ sudo vim /etc/default/isc-dhcp-server
# 編輯最下面的 INTERFACEv6
INTERFACEv6="enp0s3"

# 重新啟動吧！
$ sudo service isc-dhcp-server restart
```

## leases
最後，要如何在 server 上看到我們 leases 了哪些 IP 給 client 呢？  
就在我們前面設定的 `/var/lib/dhcp/dhcpd6.leases` 裡面唷，用編輯器或是 `cat` 之類的工具瀏覽即可。

## 小節
其實以前就架設過 DHCPv6 Server，但是在較舊版本的 Debian 上，當時所做的筆記用到現在卻不能用。印象中以前只要設定好 `dhcpd6.conf` 然後 restart service 就好的說。  
另外也有一些不常用的系統管理工具(systemctl, journalctl)，也藉著這個機會學習。~~以前都是傻傻的看 syslog，很累的~~

## 參考資料
1. ["Can't open /var/lib/dhcp/dhcpd6.leases for append." during start of ISC DHCP IPv6 Server](https://askubuntu.com/questions/173444/cant-open-var-lib-dhcp-dhcpd6-leases-for-append-during-start-of-isc-dhcp-ip)
2. [SUSE - journalctl](https://documentation.suse.com/zh-tw/sles/12-SP5/html/SLES-all/cha-journalctl.html)