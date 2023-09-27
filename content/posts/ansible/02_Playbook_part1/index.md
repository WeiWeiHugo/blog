---
slug: "02_Playbook"
title: "Ansible - Playbook(上)"
date: "2022-10-24"
lastmod: "2022-10-31"
tags: ["Ansible","Infra","vagrant","playbook"]
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

使用 ansible 第一步就是編寫 playbook，本篇文章會簡單介紹 playbook。  
會以設定一台主機，並透過 nginx 運作一個簡單的 HTTP 伺服器作為範例。運作完成後再進行介紹。

<!--more-->

## 事前準備 : 安裝 VirtualBox, Vagrant
雖然沒提到，但前一篇文章我是使用 VMware workstation 並建立虛擬機做測試，後來因為某些原因，我自己的電腦出了問題而無法使用，直到最近才重新弄回來，想說要重新建環境，那試試看新的方式好了。

VirtualBox 的安裝很簡單，到官網直接下載就好！！ [VirtualBox](https://www.virtualbox.org/)  
Vagrant 的安裝方式，我是在 MacOS 上，直接用 `brew` 搞定 ！
```bash
$ brew install vagrant
```

如果很順利的話，我們來建立一個目錄，等等要放我們會用到的 playbook 與相關文件。首先來建立 Vagrant configuration file 並測試我們剛剛安裝的 Virtualbox 與 Vagrant 是否運作正常。
```bash
$ mkdir playbooks
$ cd playbooks
$ vagrant init ubuntu/focal64 # 會建立一個 Vrgrantfile
$ vagrant up                  # 第一次做這件事情會花一些時間！

$ vagramt ssh ### 如果安裝好了，使用這個指令，應該會看到類似下面的畫面

Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-131-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Oct 21 05:32:26 UTC 2022

  System load:  0.59              Processes:               122
  Usage of /:   3.5% of 38.70GB   Users logged in:         0
  Memory usage: 21%               IPv4 address for enp0s3: 10.0.2.15
  Swap usage:   0%


0 updates can be applied immediately.

New release '22.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@ubuntu-focal:~$
```
如果可以看到上述畫面，代表我們的 VirtualBox 跟 Vagrant 是運作正常的！

接著修改 `Vrgrantfile` 如下，以利做後續的實作。主要是要做 forwarding，以 http 來說明，就是把127.0.0.1:8080 映射到 VM 的 443 埠。
```
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "testserver"
  config.vm.network "forwarded_port",
    id: 'ssh', guest: 22, host: 2202, host_ip: "127.0.0.1", auto_correct: false
  config.vm.network "forwarded_port",
    id: 'http', guest: 80, host: 8080, host_ip: "127.0.0.1"
  config.vm.network "forwarded_port",
    id: 'https', guest: 443, host: 8443, host_ip: "127.0.0.1"
  # disable updating guest additions
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end
  config.vm.provider "virtualbox" do |virtualbox|
    virtualbox.name = "test"
  end
end
```

修改完之後，請使用 `vagrant up` 來實現這些修改。應該會看到下面的輸出。
```bash
==> default: Forwarding ports...
    default: 22 (guest) => 2202 (host) (adapter 1)
    default: 80 (guest) => 8080 (host) (adapter 1)
    default: 443 (guest) => 8443 (host) (adapter 1)
```

到這邊，我們的虛擬機環境建立完成，接下來可以開始寫 playbook 了 ！

## 初體驗
建立 `playbooks/webservers.yml`，並貼上下列內容。
```
---

- name: Configure webserver with nginx
  hosts: webservers
  become: True
  tasks:
    - name: Ensure nginx is installed
      package: name=nginx update_cache=yes

    - name: Copy nginx config file
      copy:
        src: nginx.conf
        dest: /etc/nginx/sites-available/default

    - name: Enable configuration
      file: >
        dest=/etc/nginx/sites-enabled/default
        src=/etc/nginx/sites-available/default
        state=link

    - name: Copy index.html
      template: >
        src=index.html.j2
        dest=/usr/share/nginx/html/index.html

    - name: Restart nginx
      service: name=nginx state=restarted
...
```
另外，因為運行 nginx 需要設定檔，這裡也一並附上。    
需要什麼檔案，我們需要另外放在 file 目錄底下。
```
$ mkdir file
$ cd file
$ touch nginx.conf
$ vim nginx.conf
```

這裡也先附上一個簡單的 nginx.conf。
```
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        root /usr/share/nginx/html;
        index index.html index.htm;

        server_name localhost;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

接下來我們要建立一個簡單的網頁範本。~~如果沒有網頁，我們要看什麼？~~
把這個範本放在 playbooks/templates 底下，命名為 `index.html.j2`
```html
<html>
  <head>
    <title>Welcome to ansible</title>
  </head>
  <body>
  <h1>Nginx, configured by Ansible</h1>
  <p>If you can see this, Ansible successfully installed nginx.</p>

  <p>Running on {{ inventory_hostname }}</p>
  </body>
</html>
```

建立 inventory，存放這台 VM 的遠端資訊。  
我們把這個檔案命名為 `vagrant.ini`，放在 playbooks/inventory 底下
```
[webservers]
testserver ansible_port=2202

[webservers:vars]
ansible_user = vagrant
ansible_host = 127.0.0.1
ansible_private_key_file = .vagrant/machines/default/virtualbox/private_key
```

最後，把我們前一章所用到的 ansible.cfg 修改一下後放到 playbooks 裏面。  
```
[defaults]
inventory = inventory/vagrant.ini
host_key_checking = False
stdout_callback = yaml
callback_enabled = timer
```

現在我們的目錄應該要有這些檔案：
```
.
├── Vagrantfile
├── ansible.cfg
├── files
│   └── nginx.conf
├── inventory
│   └── vagrant.ini
├── templates
│   └── index.html.j2
└── webservers.yml
```

確認沒問題之後，我們執行這個指令：
```bash
$ ansible-playbook webservers.yml
```

接著我們打開瀏覽器，連線到 `http://localhost:8080` ，應該能看到 html 的畫面！
{{<image src="example.png" caption="順利完成啦～ 觀察一下 templates 內的 inventory_hostname 在哪，內容又是什麼？" width="50%" >}}

## YAML
### 開頭、結尾
YAML 會以 `---`跟`...`作為開頭與結尾，每一個 Ansible flie 都只會有一個 YAML document。  
習慣上會以這兩個作為開頭與結尾，但就算沒有使用，在 Ansible 中是不會出問題的。

### 註解
註解的方式與許多程式語言相識，使用 `#` 開頭作為註解。
``` YAML
# This is comment
```

### 縮排
沒有硬性規定，經常使用兩個空格（space） 做縮排。

### 字串
與許多程式語言不同，字串不需要特別加上單引號或是雙引號。
```YAML
This is STRING
```

但在 Ansible 的環境中，某些情況下會建議替字串加上引號，後面將會對這些內容進行說明。

### 布林值
在 YAML 中， Boolean 有許多種表示方法，使用上非常靈活。
```YAML
# These are true
true, True, TRUE, yes, Yes, YES, on, On, ON
# These are false
false, False, FALSE, no, No, NO, off, Off, OFF
```

### 清單（list）
他其實就像陣列，或是 python 的 list，基本上會用縮排與 `-` 做分隔，而名稱的後面會加上分號：
```YAML
Dinner:
  - Hamburger
  - French fries
  - Cola
```
也可以用這種表示方式：
```YAML
Dinner: [ Hamburger, French fries, Cola]
```

### 字典 (Dictionary)
基本上就像 hashes 或是其他程式語言的 dictionary。
```YAML
Dinner:
  main : Hamburger
  side dish : French fries
  drink : Cola
```
或是這種表示方法：
```YAML
Dinner: { main : Hamburger , side dish : French fries, drink : Cola}
```

## Anatomy 
### YAML Format
網路上有許多 ansible file 的寫法說明，但以我自己在 Github actions 的經驗中，會建議使用純 YAML 樣式，因為可以透過工具 `yamllist` 做檢查。  
以前面的 `webservers.yml` 為例子：
```YAML
# Before
- name: Ensure nginx is installed
  package: name=nginx update_cache=true

# After
- name: Ensure nginx is installed
  package: 
    name: nginx 
    update_cache: true
```

來修改一下 `webserver.yml` 吧：
```YAML
---
- name: Configure webserver with nginx
  hosts: webservers
  become: true
  tasks:
    - name: Ensure nginx is installed
      package:
        name: nginx
        update_cache: true

    - name: Copy nginx config file
      copy:
        src: nginx.conf
        dest: /etc/nginx/sites-available/default

    - name: Enable configuration
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link

    - name: Copy home page template
      template:
        src: index.html.j2
        dest: /usr/share/nginx/html/index.html

    - name: Restart nginx
      service:
        name: nginx
        state: restarted
...
```

在這個檔案中，用到了許多 configure，透過這些 configure 來設計我們要在這台 host 上要做的事情。  
這邊先提一下常見的三個參數：

1. _name:_ 描述這個 play 所做的內容，會建議用大寫字母作為開頭。
2. _become:_ 如果這個參數為 True，代表要運行 tasks，另外可以用 `become_user` 來設定為有權限的 user。
3. _vars:_ 變數或是數值的清單。

### Modules
Modules 是隨著 Ansible 在主機上執行某種操作的 script(?)  
不同的 Modules 之間有巨大的差異，以前面的例子來說就用了不少 Modules，也處理不同的事情。
* package : 使用 host 的 package manager 安裝或是刪除 package。
* copy : 把檔案從用作 ansible 的 host 複製到遠端 host
* file : 設置檔案、目錄的屬性
* service : 啟動、停止或重新啟動服務
* template : 從 template 產生檔案，並複製到遠端 host。


如果想要了解 Module 更深入的使用方式，可以使用 `ansible-doc [module name]` 來看說明文件並作進一步的使用：
```bash
## Example :
$ ansible-doc service
```

### Structure

playboos 包含一個或多個 play，而 play 將 host 與 Task 做關聯，每個 Task 只與一個 Module 做關聯，如果以圖來表示關係的話，大概會是像下圖這樣子：
{{<image src="structure.png" caption="關係圖" width="50%" >}}

## Status
當開始運作 Ansible 時，會發現某些任務有狀態，以稍早我們操作的結果，會看到 ok 與 Change 兩種狀態，在最後也會有一個關於 status 的統計：
```bash
$ ansible-playbook webservers.yml

PLAY [Configure webserver with nginx] ******************************************

TASK [Gathering Facts] *********************************************************
ok: [testserver]

TASK [Ensure nginx is installed] ***********************************************
changed: [testserver]

TASK [Copy nginx config file] **************************************************
changed: [testserver]

TASK [Enable configuration] ****************************************************
ok: [testserver]

TASK [Copy index.html] *********************************************************
changed: [testserver]

TASK [Restart nginx] ***********************************************************
changed: [testserver]

PLAY RECAP *********************************************************************
testserver                 : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

* ok : 如果遠端主機跟 module 的設置匹配，則不會進行任何操作，會以 ok 來表示。
* change : 與 ok 不同，如果遠端主機跟 module 的設置不匹配， Ansible 會更改遠端主機的狀態，並以 change 來表示。

另外還有 unreachable、failed、skipped、rescued 和 ignored 等狀態，當錯誤或是特定狀況發生時，會以這些 status 來表示，並提供相對的訊息。  
也因此，ansible 不僅可以更改遠端主機上的配置，也可以透過 status 來觀察遠端主機的配置內容，或是檢查配置內容是否有改變。

## 小結
基本上透過這個 playbook 與環境設置，應該能夠建立並設定 nginx 來提供服務，下篇文章會再透過 vars 做進一步的介紹。

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [Vagrant](https://www.vagrantup.com/)
3. [VirtualBox](https://www.virtualbox.org/)  