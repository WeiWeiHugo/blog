---
slug: "04_Inventory_part1"
title: "Ansible - Inventory(上)"
date: "2022-12-06"
lastmod: "2022-12-06"
tags: ["Ansible","Infra","inventory",""]
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
簡單介紹 inventory，我們先前是管理一台主機(hosts)，而多個主機的集合在 Ansible 中會被稱為 inventory。

<!--more-->
## 事前準備 
### ansible.cfg

我們先編輯 ansible.cfg，並啟用所有的 plugin。
```
[defaults]
inventory = inventory

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml
```

Ansible invertory 是一個非常靈活的物件，它可以是一個 file，一個 directory，甚至是 executable。  
inventory 可以跟 playbook 分開儲存，意味著我們可以在本地端設計 inventory，並在遠端機器上運行，像是 EC2, GCP 等等。

### Ｍultiple Vagrant Machines
前面也提到了，我們會管理多台主機，那我們就要先弄出多台主機！  
在這裡我們會配置 Vagrant 來啟動三台主機(你要弄更多台也可以)。

在這之前，我們先把先前使用的虛擬機摧毀掉(destory)：
```bash
$ vagrant destory [--force]
# 如果沒有使用 --force，則會提示並請確認是否要刪除列出的虛擬機。
```
接下來建立新的 Vagrantfile，以建立三台虛擬機：
```
VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use the same key for each machine
  config.ssh.insert_key = false

  config.vm.define "vagrant1" do |vagrant1|
    vagrant1.vm.box = "ubuntu/focal64"
    vagrant1.vm.network "forwarded_port", guest: 80, host: 8080
    vagrant1.vm.network "forwarded_port", guest: 443, host: 8443
  end
  config.vm.define "vagrant2" do |vagrant2|
    vagrant2.vm.box = "ubuntu/focal64"
    vagrant2.vm.network "forwarded_port", guest: 80, host: 8081
    vagrant2.vm.network "forwarded_port", guest: 443, host: 8444
  end
  config.vm.define "vagrant3" do |vagrant3|
    vagrant3.vm.box = "centos/stream8"
    vagrant3.vm.network "forwarded_port", guest: 80, host: 8082
    vagrant3.vm.network "forwarded_port", guest: 443, host: 8445
  end
end

`config.ssh.insert_key = false` 這行設定讓我們可以透過一組 key 連線多個虛擬機。

```
{{< admonition question "Question">}}
既然開啟了 80 跟 443 port，所以這三台主機有很大的機率是...?
{{</ admonition >}}

設定完成後我們就啟動吧，UP !!!  
這次有一台機器是 CentOS，也需要一點時間下載。
```bash
$ vagrant up
```

然後我們要知道，要使用哪些 port 才可以連線到 VM。
```bash
$ vagrant ssh-config
```

在預設情況下，Ansible 會使用 local SSH client，我們可以編輯 ssh config 讓我們更容易存取。  
雖然這個範例中只有三台，而且 config 要新增的有點多，但如果遇到 300,3000 台呢？  
編輯 ~/.ssh/config :

```
Host vagrant*
  Hostname 127.0.0.1
  User vagrant
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile ~/.vagrant.d/insecure_private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

接著我們新增 hosts 在 inventory 底下 (inventory/hosts)，port number 請依照前面觀察到的填寫。
```
vagrant1 ansible_port=2222
vagrant2 ansible_port=2200
vagrant3 ansible_port=2201
```

最後我們要確認能不能存取這些虛擬機，我們可以透過取得網卡資訊來確認：
```bash
$ ansible vagrant1 -a "ip addr"
```

這邊提供目前的 tree 做參考。如果你是用前面用到的目錄來做也沒關係，只要確保三台虛擬機有正確新增，並且能觀察到網卡資訊即可。
```
.
├── Vagrantfile
├── ansible.cfg
└── inventory
    └── hosts
```

到這邊我們的事前準備已經完成。~~先休息吧~~

## Inventory parameters

我們在設定 Inventory 時，有使用 ansible_port 作為參數，在前面的文章也有用到其他的參數。  
如果我們想要覆蓋一些預設值時，可以使用下列的參數。  

* `ansible_host` : 主機名稱，代表我們要透過 SSH 連接到的主機名稱(或是 IP address)。
* `ansible_port` : port number，代表我們要透過哪個 port 來連線。
* `ansible_user` : 使用者名稱，透過 SSH 連線時所使用的使用者名稱。
* `ansible_password` : 密碼，用於身份驗證時的密碼。
* `ansible_connection` : 連線方式，ansible 也可以使用 SSH 以外的連線方式。
* `ansible_ssh_private_key_file` : key，用於身份驗證時所使用的 SSH private key。
* `ansible_shell_type` : shell，決定 shell 的種類。
* `ansible_python_interpreter` : python_interpreter，決定在該 host 上的 python interpreter。

另外可以透過對應的變數，在 ansible.cfg 中覆蓋特定的預設值。
|  *Inventory parameters*   |  *ansible.cfg*  |
|  ----  | ----  |
| `ansible_port`  | `remote_port` |
| `ansible_user`  | `remote_user` |
| `ansible_ssh_private_key_file` | `ssh_private_key_file` |
| `ansible_shell_type` | `executable` |

## Groups
*所以我說要怎麼一次管多台？？*

我們可以透過 `all` 或是將 inventory 裡的 hosts 做 group 來進行多台主機的同時管理。
### all
直接給他用 `all` 就對了！  
```bash
$ ansible all -a "date"

### 也可以用這種方式
$ ansible '*' -a "date"
```

### 使用 groups
我們編輯 inventory/hosts：
```
[vagrant]
vagrant1 ansible_port=2222
vagrant2 ansible_port=2200
vagrant3 ansible_port=2201
```

也可以先設定主機的資訊，後面再做 group :
```
vagrant1 ansible_port=2222
vagrant2 ansible_port=2200
vagrant3 ansible_port=2201

[vagrant]
vagrant1
vagrant2
vagrant3
```
那假如我們有 vagrant999，這樣我建 group 不就要弄很多次？  
Ansible 支援 mumeric patterns：
```
[vagrant]
vagrant[1:3]
```

接著我們使用 Group name 下對應的指令，應該就會有管理多台主機的效果。  
怎麼樣編輯 group 就依照自己的喜好或是需求編輯即可。
```bash
# 我想看 vagrant 這個群組內的 hosts 時間是否一致？
$ ansible vagrant -a "date"
vagrant3 | CHANGED | rc=0 >>
Tue Dec  6 07:55:56 UTC 2022
vagrant2 | CHANGED | rc=0 >>
Tue Dec  6 07:55:56 UTC 2022
vagrant1 | CHANGED | rc=0 >>
Tue Dec  6 07:55:56 UTC 2022
```

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)