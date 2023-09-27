---
slug: "10_Faster"
title: "Ansible - How to go faster"
date: "2023-08-25"
lastmod: "2023-08-30"
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
當我們開始定期使用 Ansible 時，有時會想讓自己的 playbook 運行得更快。  
本篇文章將簡述減少 Ansible 執行 playbook 之所需時間的一些方法。

<!--more-->
## SSH
### Multiplexing and ControlPersist
Ansible 使用 SSH 作為主要的傳輸機制，SSH 是基於 TCP 傳輸協定，設備之間要先協商該次連接，完成後才實際進行傳輸行為。  
但是，多次的協商累積的時間下來也不小，這時候就造成了時間浪費。
而且 Ansible 執行 playbook 時，會建立許多 SSH 連接，這都是造成浪費的兇手。
我們可以透過 Multiplexing 來降低這些浪費。
通常在 Linux or macOS，是使用 openSSH，而 openSSH 支援 Multiplexing 的最佳化，也被稱為 ControlPersist。
啟用 Multiplexing 後，將發生以下情況：

* 第一次用 SSH 連接到主機時，OpenSSH 會啟動一個連接。
* OpenSSH 建立與遠端主機關聯的 Unix domain Socket（Control Socket）。
* 下次用 SSH 連接到主機時，OpenSSH 將使用 Control Socket 與進行傳輸，而不是建立新的 TCP 連接。

不過呢，Ansible 自動使用 Multiplexing，這邊只要知道 Multiplexing 是什麼就好了(吧)

## Pipelining
```
[connection]
pipelining = True
```
~~好了，把他設定為 True 就好。~~  
複習一下 Ansible 執行流程：
1. 將根據正在使用的模組產生 python script。
2. 將 python script 複製到主機。
3. 執行 python script。

透過啟用 pipelining，則 Ansible 會透過執行許多模組，而不是將檔案傳輸到遠端主機上來執行。  
一來減少檔案傳輸的時間，二來減少了網路連接，透過這個方式來加快速度。  
不過這個方式，經常一開始會因為 `requiretty` 出錯，這時候使用 `visudo` 修改 `sudo` 即可 :
```bash

$ visudo
#找到這行
Defaults requiretty 

# 加上註解
#Defaults requiretty
```

## Mitogen
Mitogen是一個第三方 Python lib，用於設計分散式自我複製程式。  
Mitogen 在 Ansible 上運行時。只需要更改很少的設定，它使用純 Python 取代 Ansible 以 shell 為中心的實現，  
並透過 SSH tunnel 使用更高速的 remote procedure call。

{{< admonition warning "Warning">}}
請注意，我在撰寫本篇文章時，Mitogen 僅支援 Ansible 2.10 以下的版本。
{{</ admonition >}}

在 Ansible Controller 上安裝 Mitogen：
```bash
$ pip3 install --user mitogen
```
也可以去下載 Mitogen，這邊我不提供路徑了。

要將 mitogen 設定為 ansible 中的strategy，修改 ansible.cfg：

```
[defaults]
strategy_plugins = /path/to/strategy
strategy = mitogen_linear
```

## Cache
我們先前有使用動態變數，那時候也有提到 `gather_facts: false`，這是將 fact cache 關閉。
因為動態變數是需要花時間去執行相關程式並取值回來，這邊也是要花費時間的。  
所以我們使用 fact cache 來暫時儲存相關的數值，此時就不用每次執行時都重新取一次數值，達到節省時間的目的。
```yaml
[default]
gathering = smart
fact_caching_timeout = 86400
```
這時候就不要在 playbook 內設定 `gather_facts`，會影響 gathering 的運作。  
gathering 在 cache 中找不到 fact ，才會去做取值的動作。

### JSON
我們可以將 cache 存成 json 檔案。
```yaml
[default]
gathering = smart
fact_caching_timeout = 86400
fact_caching = jsonfile
fact_caching_connection = /path/of/cache
```

### Redis
除了 JSON 以外，也可以將 cache 寫進 Redis 內，但 Redis 我不熟，這邊簡單提一下設定。
```yaml
[default]
gathering = smart
fact_caching_timeout = 86400
fact_caching = redis
```

## Parallelism
對於每一個 task，Ansible 都會以平行的方式連接到目標主機並執行任務，但不一定會平行連接到所有主機。  
連接數是透過 `ANSIBLE_FORKS` 這個參數來決定，預設值為 5，代表同時有五個 Parallelism Processes 在執行。  
```bash
$ export ANSIBLE_FORKS=10 # 這樣就修改為 10 個
```

也可以在 `ansible.cfg` 中設定：
```yaml
[defaults]
forks = 10
```

不過，取決於電腦性能，太多 forks 反而會導致執行速度降低。

## Async
在執行 tasks 時，會是依序進行，先完成 task1 後再緊接著執行 task2，但兩件事情有時是不相干的，是可以同時進行的，這時候我們可以透過 async 讓 task2 在 task1 完成前執行。  
另外，有些 tasks 花費的時間會很長，會比 ssh 超時時間還要長，一樣可以用 async ，並使用 polling 的方式直到 tasks 執行完畢。

```yaml
task:
- name: This is sleeping.
  command: /bin/sleep 30
  async: 90 # 允許執行多長時間，如果超過這個數值，會自動停止與這個 task 相關的 process
  poll: 0 # 確認多久輪詢一次，如果為 0 就是直接執行下一個 task。
  register: linux_sleep # 後續要取得 result 要用的。

- name: Get result
  async_status: # 用來檢查 task 的狀態
    jid: "{{ linux_sleep.ansible_job_id }}" # 透過我們在上一個 task 所註冊的變數。
  register: result
  until: result.finished # 直到 result 變數中的 finished 為 True（即操作完成）之前，將不斷重試此檢查。
  retries: 3600 # 最多重試3600次
```

## 結論
簡單結論一下，當我們撰寫好 playbook 與相關設定後，我們可以透過前述的設定，讓 ansible 運行得更快。  
不過呢，有些工具不能在最新的環境上使用，此時就需要等待，或是自己改寫相關工具，另外有些設定不一定會讓 ansible 運行的更快，可能需要多次嘗試才能夠找出較佳的設定。

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [Mitogen for Ansible](https://mitogen.networkgenomics.com/ansible_detailed.html)
3. [Asynchronous actions and polling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_async.html)
