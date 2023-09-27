---
slug: "06_Variables"
title: "Ansible - Variables"
date: "2023-02-20"
lastmod: "2023-02-20"
tags: ["Ansible","Infra","Magic Variables"]
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
本篇文章將介紹在 Ansible 中的 Variables 跟 facts，並簡單介紹一下 Magic Variable。
<!--more-->

## 定義
在其他程式語言中也是有變數，而 Ansible 中也有，變數可以讓我們更靈活的使用。  
基本上最簡單的方法就是宣告一個變數名稱，跟你想賦予該變數的數值：

```yaml
vars:
  tls_dir: /etc/nginx/ssl/
  key_file: nginx.key
  cert_file: nginx.crt
  conf_file: /etc/nginx/sites-available/default
  server_name: localhost
```

以上面這個例子來說明：
* `tls_dir` : 就是我們宣告的變數名稱。
* `/etc/nginx/ssl/` : 我們要賦予該變數的數值。

我們也可以另外將變數放到一個或是多個檔案中，

```yaml
var:
  - nginx.yml
```

*nginx.yml*
```yaml
tls_dir: /etc/nginx/ssl/
key_file: nginx.key
cert_file: nginx.crt
conf_file: /etc/nginx/sites-available/default
server_name: localhost
```

這種方式可以讓我們分離一些較敏感的資訊，像是 key, token 之類的。  
這些敏感資訊就可以另外存放在一個檔案，透過 gitignore 或是其他類似工具，依據自己的需求做使用，使這些敏感資訊不被記錄。

{{< admonition question "Question">}}
那我們把變數都放在同一個檔案，有沒有較好管理的方法？
{{</ admonition >}}

其實 Ansible 允許我們定義 groups 或是 hosts 關聯的變數，這些變數需要在特定的目錄下操作，分別為 `group_vars` 跟 `host_vars`，根據這些名稱，應該就能知道哪個目錄是跟 groups 搭配，哪個是跟 hosts 搭配了吧？

## debug, register and stat
### debug
那要如何查看變數實際是什麼數值呢？我們需要使用 `debug` 這個指令：
```yaml
  - debug: var=myVarName
```

如果需要加上一些訊息，那就要用兩個大括弧包起來：  
~~這讓我想到用 hugo 開發的回憶...~~

```yaml
  - debug: "【Info】 debug : {{ myVarname }}"
```

### register
另外，我們會經常將 task 的結果，用來設置一個變數的數值，這個時候我們就可以使用 register 建立一個變數，並將 task 的結果設為該變數的數值。

我們在上一章 Dynamic Inventory 其實就有使用到了，那這邊再提供一個新的 playbook 做使用好了，環境可以保留之前的就好。這邊我把它命名為 `show.yml`。  
下面的例子中你可以看到他是以 JSON 格式回傳結果的。

```yaml
---

- name: Do something to host
  hosts: vagrant1
  become: true
  tasks:
    - name: Show id
      command: id -un
      register: login

    - name: print stdout
      debug:
        msg: "login info : {{ login.stdout }}"
```

```bash
$ ansible-playbook show.yml

TASK [Show id] **************************************************************************
changed: [vagrant1]

TASK [print stdout] *********************************************************************
ok: [vagrant1] => {
    "msg": "login info : root"
}

PLAY RECAP ******************************************************************************
vagrant1                   : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

在 task 失敗時，`debug` 跟 `ignore_errors` 也是很常用的工具。 ~~(名稱那麼明顯...)~~ 
例如在 task 失敗時， ansible 就會因為 task 失敗而停止。
透過 `ignore_errors` 可以忽略這個錯誤，再透過 `debug` 將結果顯示出來，以利後續的故障排除，而又不會因為這個錯誤影響到後面的 tasks。  

```yaml
---

- name: Do something to host
  hosts: vagrant1
  become: true
  tasks:
    # 這個 task 會有問題，因為沒有 ifconfig 這個 command 做使用。可以自己改 ignore_errors 的數值來觀察結果。
    - name: Show ip
      command: ifconfig
      register: info
      ignore_errors: true

    - name: print stdout
      debug:
        msg: "login info : {{ info.stdout }}"

    - name: Show id
      command: id -un
      register: login

    - name: print stdout
      debug:
        msg: "login info : {{ login.stdout }}"
```

### stat
在 debug 時，我們也可以透過 stat 來取得檔案的資訊做進一步的分析。 
ansible 仍會以 JSON 格式回傳該檔案的資訊。 
```yaml
---

- name: Do something to host
  hosts: vagrant1
  become: true
  tasks:
    - name: Get stat
      stat:
        path: /etc/passwd
      register: file

    - name: print stat
      debug:
        msg: "file info : {{ file.stat }}"
```

```bash
## Output
TASK [Get stat] *************************************************************************
ok: [vagrant1]

TASK [print stat] ***********************************************************************
ok: [vagrant1] => {
    "msg": "file info : {'exists': True, 'path': '/etc/passwd', 'mode': '0644', 'isdir': False, 'ischr': False, 'isblk': False, 'isreg': True, 'isfifo': False, 'islnk': False, 'issock': False, 'uid': 0, 'gid': 0, 'size': 1808, 'inode': 71460, 'dev': 2049, 'nlink': 1, 'atime': 1676858098.308, 'mtime': 1670307804.2306128, 'ctime': 1670307804.2306128, 'wusr': True, 'rusr': True, 'xusr': False, 'wgrp': False, 'rgrp': True, 'xgrp': False, 'woth': False, 'roth': True, 'xoth': False, 'isuid': False, 'isgid': False, 'blocks': 8, 'block_size': 4096, 'device_type': 0, 'readable': True, 'writeable': True, 'executable': False, 'pw_name': 'root', 'gr_name': 'root', 'checksum': '80b9dbad5183c2671c8a9639086593820c7988f0', 'mimetype': 'text/plain', 'charset': 'us-ascii', 'version': '2549813272', 'attributes': ['extents'], 'attr_flags': 'e'}"
}
```

以這個例子來說，我們可以透過 key 更進一步的存取。  
例如這個結果中的 `mode`: `0644`，你可以透過 dot 或是中括弧來存取，且兩者可以交互使用，只要不會因為一些特殊字元影響即可（像是dot, space ...），所以取得這個 mode 的資訊就有四種方法。
```yaml
msg: "file mode : {{ file.stat['mode'] }}"
msg: "file mode : {{ file.stat.mode }}"
msg: "file mode : {{ file.['stat']['mode'] }}"
msg: "file mode : {{ file.['stat'].mode }}"
```

## facts
在我們剛剛的測試中，都可以發現一開始都會有一個情況：  
```bash
TASK [Gathering Facts] ******************************************************************
ok: [vagrant1]
```

這代表 ansible 正在連接到 hosts，並查詢有關 hosts 的各種資訊，這些資訊都可以透過開頭為 `ansible_facks` 的變數來讀取。  
修改一下我們前面用的 show.yml :
```yaml
--

- name: Do something to host
  hosts: all
  gather_facts: true
  tasks:
    - name: Print system details
      debug:
        msg: >-
          os_family:
          {{ ansible_facts.os_family }},
          distro:
          {{ ansible_facts.distribution }}
          {{ ansible_facts.distribution_version }},
          kernel:
          {{ ansible_facts.kernel }}

...
```

```bash
## Output
TASK [Gathering Facts] ******************************************************************
ok: [vagrant3]
ok: [vagrant1]
ok: [vagrant2]

TASK [Print system details] *************************************************************
ok: [vagrant1] => {
    "msg": "os_family: Debian, distro: Ubuntu 20.04, kernel: 5.4.0-131-generic"
}
ok: [vagrant2] => {
    "msg": "os_family: Debian, distro: Ubuntu 20.04, kernel: 5.4.0-135-generic"
}
ok: [vagrant3] => {
    "msg": "os_family: RedHat, distro: CentOS 8, kernel: 4.18.0-277.el8.x86_64"
}
```

{{< admonition question "Question">}}
那我要如何知道，有哪些 ansible_fact 可以使用呢？
{{</ admonition >}}

ansible 會自動幫我們收集這些資訊，只要透過 setup 這個 module 就可以知道搜集了哪些資訊。  
輸出因為很多，我就不附上了，但可以知道他會回傳一個 key 為 `ansible_facts` 的 dictionary。
```bash
$ ansible [hostname or groupname] -m setup
## e.g. 
$ ansible vagrant2 -m setup
vagrant2 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
(more facts ....)
```

因為 ansible_facts 收集了很多資訊，我們可以使用 fact name 跟 filter 來找出我們要的 fact：
```bash
## 使用我們之前用的 groupname
$ ansible vagrant -m setup -a 'filter=ansible_all_ipv4_addresses'
## Output
vagrant3 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false
}
vagrant2 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
vagrant1 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
```

### local facts
Ansible 提供了一個機制，讓我們可以建立 facts 並放在遠端主機上，只要放在 `/etc/ansible/facts.d` 底下且檔案符合以下機制：

* 為 .ini 格式
* 為 JSON 格式
* 不需要任何參數，便可以輸出 JSON 的執行檔

隨意建立一個吧，/etc/ansible/facts.d/example.fact
```yaml
[info]
date=20230220
weather=Rainy
```

這時候你就可以透過 `ansible_local` 取得自己定義的 facts：
```bash
$ ansible vagrant1 -m setup -a 'filter=ansible_local'
## Output
{
    "ansible_local": {
        "example": {
            "info": {
                "date" : "20230220",
                "weather"  : "Rainy"
            }
        }
    }
}
## 當然，你也可以用 playbook，透過 debug 取得。
```

### set_fact
我們也可以在 task 當中，設置 fact，這用起來就跟設定新的 variable 一樣。
```yaml
  - name: Set nginx state
    when: ansible_facts.services.nginx.state is defined
    set_fact:
      nginx_state: "{{ ansible_facts.services.nginx.state }}"
```

fact 提供給我們許多資訊，透過 ansible_fact 可以做更多運用，也可以透過 ansible_local 在遠端主機上建立 fact。  
set_fact 通常被用來重新定義，因為存取 ansible_fact 往往都會是一大串的變數名稱，透過重新定義可以較容易的做存取。

## Magic Variables

Ansible 有定義幾個在 playbook 中可以用的變數，這些變數被稱為 Magic Variables，也有人稱為 Built-in Variables。 
通常建議保留而不建議覆蓋這些變數。   
以下是幾個常用的 magic variables：  
* `hostvars`
* `group_names`
* `groups`
* `environment`

可以試著用 debug 來觀察 hostvars 會有什麼資訊，基本上就是各個 node 的 facts。  
另外，`inventory_hostname` 也是經常被拿來搭配使用。
```yaml
- name: Retrieve host vars
  hosts:
    - vagrant1
    - vagrant3
  tasks:
    - name: Show IP address by hostvars
      debug: var=hostvars[inventory_hostname]['ansible_default_ipv4']
```

這邊就不附上結果了，請自行觀察。
更多 Magic Variables 的資訊跟使用方式，可以參考下列的文件：
* [Discovering variables: facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
* [魔法變數](https://chusiang.github.io/ansible-docs-translate/playbooks_variables.html#magic-variables-and-hostvars)

~~因為我覺得 Magic variables 又可以寫一篇，但我好懶惰所以...以後有可能補上。~~

## 結論
這次簡單介紹了幾種定義變數、存取變數與存取 facts 的方法，透過 variables 跟 facts ，可以讓我們的 tasks 更加靈活，像是取得 IP address 就直接透過 facts 取得就好，不必再使用 command。  
另外也使用了 debug 這個指令，可以讓我們知道 variables 內到底有什麼數值。  
最後，簡單介紹了一下 Magic variables。


## 參考資料
1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [Discovering variables: facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
3. [魔法變數](https://chusiang.github.io/ansible-docs-translate/playbooks_variables.html#magic-variables-and-hostvars)
4. [Ansible: Working with Variables and Hostvars](https://admantium.medium.com/ansible-working-with-variables-and-hostvars-479c9d3f4f54)
