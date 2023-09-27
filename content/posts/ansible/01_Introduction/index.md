---
slug: "01_Introduction"
title: "Ansible - Intro"
date: "2022-04-19"
lastmod: "2022-04-19"
tags: ["Ansible","Infra"]
sidebar: auto 
categories: ["Infra"]
no_comments: false
draft : false
resources:
- name: "featured-image-preview"
  src: "ansible-logo.png"
- name: "featured-image"
  src: "ansible-logo.png"
---

本篇文章會簡單介紹 ansible 在做什麼與他的優勢，並介紹安裝方式與環境架設。

<!--more-->
## 前言
基本上會看 ansible 是因為以前接觸過，但沒有深入去研究，不過 ansible 仍然為自己想學習的技能之一，故這次想透過這本書做進一步的學習。  
這系列的文章應該會更新的很慢，因為手邊還有一些事情要處理，並不像之前更新 Go 系列，有較頻繁的更新頻率。 ~~（ 而且 Go 系列到後面都在亂寫...）~~

基本上因為雲的高擴展性與容易性（相較容易使用），許多服務逐漸放上雲去運作，包含常見的 Web server、應用程式伺服器與資料庫等，基本上 IT 人員要負責這些服務的運作，監控與紀錄並進一步分析，甚至要安排冗餘，使得故障時，服務能不受大影響。  

為了維護這些服務，可以手動處理，但手動設定這些服務非常耗時，而且容易出錯，同樣的事情可能做個三次四次就會覺得煩躁，如果遇到困難的任務或是設定，可能還沒弄好，心情先差到極點。  

透過 ansible，我們可以降低配置服務時的時間，降低出錯的機率，並且這是自動化的，只要一開始願意花時間配置 ansible，後面就會多一些時間喝咖啡（但不要灑出來）

{{< admonition note "Note">}}
我們要效能。
{{</ admonition >}}
{{<image src="performance.png" caption="看看就好" width="50%" >}}

## 簡介
Ansible 通常被稱為 **配置管理工具** (configuration management tool)，通常在談論配置管理時，代表我們會替伺服器編寫特定的狀態描述，並以工具使伺服器確實處在該狀態。  
Ansible 也能幫助我們進行部署的工作，像是開發時產生的編譯檔，將這個編譯檔部署至伺服器上並運行。  
另外也經常用來配置新的伺服器，簡單來說我需要一台安裝好 python 並且已經設定好 vim 參數的虛擬機，可以透過 ansible 協助我們產生這樣的虛擬機。另外也可以配置關於雲的服務等（包括 EC2、Azure、Digital Ocean、Google Compute Engine、Linode 和 Rackspace，以及任何支持 OpenStack API 的雲。）

## 工作原理
舉個例子：  
有個使用者需要透過 ansible，在三台 Ubuntu 上安裝 nginx，三台分別稱為 web1, web2 與 web3，使用者編寫了一個名為 webservers.yml 的 Ansible 腳本。在 Ansible 中，腳本稱為 **playbook**。playbook 描述了要配置的主機（Ansible 稱之為遠端伺服器），以及要在這些主機上執行的任務的有序列表。  

webservers.yml 的內容會在後面提到，這邊只是單純說明原理。使用者寫好 webservers.yml 後會透過 ```ansible-playbook``` 運行 playbook：
```shell
$ ansible-playbook webservers.yml
```
基本上任務會像這樣：
```yaml
- name: Install nginx
  package:
    name: nginx
```

Ansible 將執行以下操作：
* 生成安裝 Nginx package 的 Python 腳本
* 將腳本複製到 web1、web2 和 web3
* 在 web1、web2 和 web3 上執行腳本
* 等待腳本在所有主機上完成執行

{{< admonition tips "Tips">}}
注意以下幾點：
* Ansible 在所有主機上並行運行每個任務。
* Ansible 會等到所有主機都完成一個任務，然後再移動到下一個任務。
* Ansible 按照您指定的順序運行任務。
{{</ admonition >}}

## 優勢

* 相較於其他配置管理工具， ansible 學習起來較容易些。  
* ansible 使用 yaml 與 Jinja2 模板，兩者都很容易上手。 yaml 我在使用 github actions 就有經驗了，非常的直觀。  
* 可以透過多種方式檢查 ansible playbook，例如列出所有涉及操作的主機。如果要測試 playbook，可以使用 `ansible-playbook --check` 來測試，並且有 log 可以觀察誰在哪裡做了什麼。
* 遠端伺服器也不需要預先安裝許多服務，在 Linux 上只需要安裝 SSH 與 python，而在 Windows 上需要啟用 WinRM。
* 可以在 Raspberry Pi 或是舊電腦上運作，不需要很好的硬體資源。
* 有社群可以分享自己的 playbook ，或是參考別人的 playbook。
* 使用抽象的方式運作：  
e.g. 使用 shell 建立目錄並設定權限：
```shell
mkdir -p /etc/skel/.ssh 
chown root:root /etc/skel/.ssh 
chmod go-wrx /etc/skel/.ssh
```
透過 ansible 運作：
```yaml
- name: Create .ssh directory in user skeleton
  file:
    path: /etc/skel/.ssh
    mode: 0700
    owner: root
    group: root
    state: directory
```

### 

## 需要的基本技能
基本上要熟悉至少一種 Linux ，像是 Ubuntu, CentOS 等，並了解以下技能或資訊：
* 使用 SSH 連接到遠端設備
* Bash ,shell 交互（pipes and redirection）
* 安裝 package
* sudo
* 檢查和設置文件權限
* 啟動和停止服務(service)
* 設置環境變數(env)
* 編寫 script


## 安裝
基本上安裝相當簡單：
```shell
## macOS
$ brew install ansible
```
或是使用 `pip` 進行安裝：
```shell
## Unix/Linux/macOS
$ python -m pip install --user ansible
```
雖然 ansible 可以管理 windows，但不能在 windows 上運行 ansible。

## inventory
基本上我們會建立一個 inventory 目錄來存放遠端設備的相關資訊。  
並依據不同的設備名稱，儲存不同的資訊：
```
### inventory/webserver.ini
[webservers]
testserver ansible_port=2222

[webservers:vars]
ansible_host = 192.168.0.1
ansible_user = user
ansible_private_key_file = /path/of/private_key
```
{{< admonition note "Note">}}
ansible 支援 ssh-agent，所以我們不需要明確指定密鑰。
{{</ admonition >}}

接著我們就可以進行測試，像是連結到遠端伺服器並使用 `ping`：
```shell
$ ansible testserver -i inventory/vagrant.ini -m ping
```

## ansible.cfg
然而在上一節最後的部分，那個指令還是太長了，我們還是要記一些參數才能使用。  
透過預先設定好的 ansible.cfg，我們在往後使用時可以更加快速。  
ansible.cfg 通常會建議與 `playbook` 放在一起，這樣做版本控制時較好管理。  
```
[defaults]
inventory = inventory/webserver.ini
host_key_checking = False
stdout_callback = yaml
callback_enabled = timer
```

接著我們就可以用這個方式連結到遠端伺服器並使用 `ping`：
```shell
$ ansible testserver -m ping
```

## 一些參數
透過 `command`，我們可以在遠端設備上執行指令：
```shell
### 這樣遠端設備上會執行 uptime
$ ansible testserver -m command -a uptime
```

如果說我們的指令有空格，請使用雙引號傳送整個字串：
```shell
$ ansible testserver -m command -a "tail /var/log/syslog"
```

如果說這個指令需要 root 權限，請加上 `-b`
```shell
$ ansible testserver -m command -b -a "tail /var/log/syslog"
```

## 小結論
基本上介紹了 Ansible 的基本概念，包括它如何與遠端設備連線以及它與其他配置管理工具的不同之處。  
剛開始都先確保環境沒問題之後，進行 1 對 1 的連線測試，完成後再進行後續的學習。

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)