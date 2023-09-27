---
slug: "08_roles"
title: "Ansible - Roles"
date: "2023-07-07"
lastmod: "2023-07-07"
tags: ["Ansible","Infra","roles","ansible-galaxy"]
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
這篇文章會簡單說明 roles 是什麼， roles 的結構與如何設計一個簡單的 roles，另外會稍微說明 ansible-galaxy，一個知名的共享 roles 工具。

<!--more-->
## Roles
Roles 可以讓我們根據已知的資訊，自動載入相關的變數、文件與 task 等。  
透過 Roles 可以讓我們更容易重複使用，也方便與他人分享。  

### 產生 roles
我們先用 `ansible-galaxy` 來產生一個 roles ~~（這樣比較快啦）~~。
```bash
$ ansible-galaxy role init --init-path playbooks/roles database
- Role database was created successfully
```

這樣會以 database 為名稱，初始化一個 roles。  
ansible-galaxy 是一個很好用的工具，後面會提到他。

### 基本架構
基本上每一個 role 都會有一個名稱，並且會放置在 roles 這個目錄底下。
```bash
$ tree   # 這邊我拿之前實驗的目錄當例子。
roles
├── database
│   ├── README.md
│   ├── defaults
│   │   └── main.yml
│   ├── files
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   ├── tests
│   │   ├── inventory
│   │   └── test.yml
│   └── vars
│       └── main.yml
```

基本上不同的目錄代表不同的含義：
* task : 主要的 task ，這邊會是 roles 執行的入口。
* file : 這邊會存放要上傳到 host 的檔案或 script。
* templete : 這邊保存要上傳到 host 的 Jinja2 template。
* handler : 主要的 handler。
* vars : 通常不該被改變的變數。
* defaults : 可以更改的預設變數。
* meta : 有關這個 role 的資訊。
* test : 測試的地方。

每個單獨的檔案都是可選的，例如說我們的 role 沒有 handler，就不需要去建立一個空的。

另外許多功能先前已經說明過，這邊只會先介紹最基本的 task 跟 variable，如果其他的功能不太熟悉，可以看先前的文章，或是後面待會說明 ansible-galaxy 時，下載其他大神的 roles 來參考。~~(偷懶啊！)~~

### task
前面提到 task 是 roles 執行的入口。我們先修改 task 中的 main.yml，並讓他 print 一些系統資訊。
```yaml
---
- name: Echo some message.
  debug:
    msg: >-
      os_family:
      {{ ansible_facts.os_family }},
      distro:
      {{ ansible_facts.distribution }}
      {{ ansible_facts.distribution_version }},
      kernel:
      {{ ansible_facts.kernel }}
```

接著修改或是新增 playbook，我這邊是又新增 `roles.yml` 來測試。
```yaml
---
  - hosts: vagrant1
    roles:
      - role: database
```

然後使用 `ansible-playbook` 執行 `roles.yml`，可以看到該台主機相關的資訊。  
{{< admonition question "Question">}}
這樣跟我們之前用的有什麼不同，不就只是把 playbook 換個地方然後引進來使用嗎？
{{</ admonition >}}
我們目前就只有數個 playbook，管理三台由 vagrant 產生的機器，如果我們有數百甚至數千的 playbook 跟 host，那使用上可能會遇到大量的重複使用，且在管理上也是要花費很多時間且難以理解。    
將做好的功能做成 role，來讓其他 playbook 可以重複使用，以提高 ansible 使用上的靈活度。

### default, var
前面提到，這兩個地方是放置變數的，差別只在一個經常變動，另一個則不經常變動。
我們修改 default 跟 var 中的 main.yml

`default/main.yml`
```yaml
---
# defaults file for database
default_test: "I am default"
```

`vars/main.yml`
```yaml
---
# vars file for database
vars_test: "I am vars"
```

然後再修改前面所用的 `task/main.yml`
```yaml
---
- name: Echo some message.
  debug:
    msg: >-
      test:
      {{ default_test }}
      {{ vars_test }}
```

接著執行 playbook，你應該可以看到 default 跟 vars 中的變數內容 ~~(連在一起了啦)~~：  
```bash
$ ansible-playbook -C roles.yml

PLAY [vagrant1] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [vagrant1]

TASK [database : Echo some message.] *******************************************
ok: [vagrant1] => {}

MSG:

test: I am default I am vars

PLAY RECAP *********************************************************************
vagrant1                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

{{< admonition Info "Info">}}
通常我們改變數只會改 default 中的變數，vars 比較不會去動它。  
另外在優先權的部分，default 是最小的，所以 vars 或是其他檔案中有相同名稱的變數，則會覆蓋掉 default 中的變數。  
由於 vars 的優先權極高，建議是在 task 執行時進行傳遞或是覆蓋。
{{</ admonition >}}



## Ansible-galaxy
`ansible-galaxy` 主要的目的是使用社群上大家所分享的 role。 另外也可以初始化 roles，提供一個簡單的目錄給我們開發，就像前面使用的功能。  


ansible-galaxy的指令有搜尋、安裝與移除等：
```bash
$ ansible-galaxy search database
Found 1997 roles matching your search. Showing first 1000.

 Name                                                 Description
 ----                                                 -----------
 0x0i.grafana                                         Grafana - an analytics and monitoring observability platform
 0x0i.prometheus                                      Prometheus - a multi-dimensional time-series data monitoring and alerting toolkit
 0x5a17ed.ansible_role_netbox                         Installs and configures NetBox, a DCIM suite, in a production setting.
 0x_peace.mariadb                                     Mariadb role
 123mwanjemike.ansible_mongodb                        Configure the components of a MongoDB Cluster
 1it.riak                                             Installs and configures Riak KV and TS, a distributed, highly available NoSQL and TimeSeries database.
 1mr.timezone                                         change timezone
 ...
```
如果有找到喜歡的，我們可以用 `install` 進行安裝，這邊我就以安裝 0x_peace.mariadb 為例子：
```bash
$ ansible-galaxy install 0x_peace.mariadb
Starting galaxy role install process
- downloading role 'mariadb', owned by 0x_peace
- downloading role from https://github.com/0x-peace/mariadb/archive/master.tar.gz
- extracting 0x_peace.mariadb to /Users/user/.ansible/roles/0x_peace.mariadb
- 0x_peace.mariadb (master) was installed successfully

# 我想要換路徑啦！那就用下面這個！
$ ansible-galaxy install -f -p [path] 0x_peace.mariadb
```

### List
list 可以讓我們知道已經安裝了哪一些 roles，在我的環境中，他會去抓一些預設的路徑。
```bash
$ ansible-galaxy list
# /Users/user/.ansible/roles
- 0x_peace.mariadb, master
[WARNING]: - the configured path /usr/share/ansible/roles does not exist.
[WARNING]: - the configured path /etc/ansible/roles does not exist.

# 一樣也可以指定路徑。
$ ansible-galaxy list -p [path]
# /Users/user/Desktop/playbooks/roles
- 0x_peace.mariadb, master
# /Users/user/.ansible/roles
- 0x_peace.mariadb, master
[WARNING]: - the configured path /usr/share/ansible/roles does not exist.
[WARNING]: - the configured path /etc/ansible/roles does not exist.
```

### Remove
~~知道了我們有裝哪些 roles 後可以幹嘛呢？沒錯就是要來把它刪掉~~
Remove 就如同字面上，用來把想刪的 roles 刪除！
```bash
$ ansible-galaxy remove 0x_peace.mariadb
- successfully removed 0x_peace.mariadb

# 一樣可以用 -p 來指定路徑
$ ansible-galaxy remove -p [path ]0x_peace.mariadb
- successfully removed 0x_peace.mariadb
```

### 上傳 roles
*我寫好了，我想上傳跟大家分享！*

上傳 roles 的話，首先要有一個 github 帳號，然後把你的 roles push 到 github 上。  

接著登入到 [Galaxy Ansible](https://galaxy.ansible.com/)，網站會幫你同步你的 repository，你只要把你存放 roles 的 repository 權限打開即可。

因為網站經常改版，雖然步驟都是一樣但功能的相對位置會改變，這邊就不附上圖片說明。  

## Requirements
有時候你會發現 role 內有一個 requirements.yml，這個是用來列出這個 role 的依賴項，通常會在 roles 內，如果是使用 AWX 或是 Ansible Tower 時會自動安裝列出的依賴項。  
這邊可以使用 galaxy 中的 role 或是其他 repository 中的 role。  
不同的來源在 requirements 中用法也不同，這邊可以參考[官方的說明文件](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html#install-multiple-collections-with-a-requirements-file)，我在這也附上相關用法。 

```yaml
# from galaxy
- name: yatesr.timezone

# from locally cloned git repository (git+file:// requires full paths)
- src: git+file:///home/bennojoy/nginx

# from GitHub
- src: https://github.com/bennojoy/nginx

# from GitHub, overriding the name and specifying a specific tag
- name: nginx_role
  src: https://github.com/bennojoy/nginx
  version: main

# from GitHub, specifying a specific commit hash
- src: https://github.com/bennojoy/nginx
  version: "ee8aa41"

# from a webserver, where the role is packaged in a tar.gz
- name: http-role-gz
  src: https://some.webserver.example.com/files/main.tar.gz

# from a webserver, where the role is packaged in a tar.bz2
- name: http-role-bz2
  src: https://some.webserver.example.com/files/main.tar.bz2

# from a webserver, where the role is packaged in a tar.xz (Python 3.x only)
- name: http-role-xz
  src: https://some.webserver.example.com/files/main.tar.xz

# from Bitbucket
- src: git+https://bitbucket.org/willthames/git-ansible-galaxy
  version: v1.4

# from Bitbucket, alternative syntax and caveats
- src: https://bitbucket.org/willthames/hg-ansible-galaxy
  scm: hg

# from GitLab or other git-based scm, using git+ssh
- src: git@gitlab.company.com:mygroup/ansible-core.git
  scm: git
  version: "0.1"  # quoted, so YAML doesn't parse this as a floating-point value
```

## 結論
role 是組織 playbook 的好方法，這篇文章介紹了如何使用 role，如何設計自己的 role 跟使用別人所設計的 role。  

這篇筆記是自己受傷後所寫的第一篇，希望自己能趕快恢復狀況，維持持續學習的習慣。

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [Galaxy User Guide](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html)