---
slug: "05_Inventory_part2"
title: "Ansible - Inventory(下)"
date: "2023-01-26"
lastmod: "2023-01-26"
tags: ["Ansible","Infra","invertory","Dynamic inventory"]
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
接續上篇 Invertory 做後續的介紹。大致上會介紹動態清單(Dynamic Inventory)跟一些相關指令。

<!--more-->
## 前言
因為這個月事情比較多，又剛好遇到過年想休息，這篇文章的內容可能會少一點。~~(一直都很少)~~

## Dynamic Inventory
### 簡介
先前都是在 `inventory/hosts` 中直接指定 hosts 跟 Group，Dynamic Inventory 通常是指可以透過 Script 獲得的 Inventory ，且這個 Inventory 也是符合 ansible 所需格式的。  
由於一些系統資源會是動態的進行增減，像是目前眾所皆知的 Cloud，我們可以透過 Script 與相關的 API 來取得 Inventory。

### 格式
ansible 所需格式是 json，那格式長什麼樣子呢？  
我們可以透過一些指令來取得：
```bash
$ ansible-inventory -i inventory/hosts --list
```

你會獲得一個 json 格式的輸出：
```json
{
    "_meta": {
        "hostvars": {
            "vagrant1": {
                "ansible_port": 2222
            },
            "vagrant2": {
                "ansible_port": 2200
            },
            "vagrant3": {
                "ansible_port": 2201
            }
        }
    },
    "all": {
        "children": [
            "ungrouped",
            "vagrant"
        ]
    },
    "vagrant": {
        "hosts": [
            "vagrant1",
            "vagrant2",
            "vagrant3"
        ]
    }
}
```
`_meta` 這邊會包含所有 host 的資訊。  
如果你只想觀察單台 host ，可以使用 `--host` 這個參數：  
```bash
$ ansible-inventory -i inventory/hosts --host=vagrant3
{
    "ansible_port": 2201
}
```

### 範例
因為我們使用的環境是 vagrant，我們可以透過 vagrant 的指令取得一些資訊，以利後面的 Dynamic Inventory 做使用。
```bash
$ vagrant status
Current machine states:

vagrant1                  running (virtualbox)
vagrant2                  running (virtualbox)
vagrant3                  running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.

# 用這個指令也會取得許多資訊，因資訊量蠻多的，就不附上 Output。
$ vagrant status --machine-readable
```

接著我們建立一個 .py 檔，使用網路上大佬已經寫好的 Script。[來源](https://charlesreid1.com/wiki/Ansible/Vagrant/Dynamic_Inventory)  

你可以透過執行這個 Script，來觀察 ansible 的格式。
```bash
$ python test.py --list
# 或是
$ python test.py --host=vagrant
```

建立完之後，我們需要使用 `-i` 這個 flag 來傳遞這個 Script，  
跟上一篇文章一樣，我想看 vagrant 這個群組內的 hosts 時間是否一致？

```bash
$ ansible -i test.py vagrant -a "date"
[WARNING]:  * Failed to parse /U/cant/see/mypath/playbook/test.py with script plugin:
problem running /U/cant/see/mypath/playbook/test.py --list ([Errno 13] Permission
denied: '/U/cant/see/mypath/playbook/test.py')
[WARNING]:  * Failed to parse /U/cant/see/mypath/playbook/test.py with ini plugin:
/U/cant/see/mypath/playbook/test.py:6: Expected key=value host variable assignment,
got: argparse
[WARNING]: Unable to parse /U/cant/see/mypath/playbook/test.py as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: Could not match supplied host pattern, ignoring: vagrant
[WARNING]: No hosts matched, nothing to do
```

發生了什麼？？？別慌張，權限不足而已。
讓我們更改一下，~~發現有一台壞孩子！~~
```bash
$ chmod 755 test.py
$
$ ansible -i test.py vagrant -a "date"
vagrant3 | CHANGED | rc=0 >>
Fri Jan 27 12:45:33 UTC 2023
vagrant2 | CHANGED | rc=0 >>
Sat Jan 28 11:35:43 UTC 2023
vagrant1 | CHANGED | rc=0 >>
Sat Jan 28 11:35:43 UTC 2023

```

這樣當我們往後新增或刪除主機時，透過 Dynamic Inventory 可以減少一些工作量，但就是先前準備要比較辛苦。  
另外在 EC2, GCP, Azure 或我們需要的地方做使用。  

{{< admonition question "Question">}}
如果你還留著說明 playbook 時的環境，  
試一下或猜一下 `$ ansible-playbook -i test.py webservers.yml`，會有什麼結果？
{{</ admonition >}}

## add_host, group_by
有的時候我們在運作 playbook 時，有新的 host 上線了，即使我們使用 Dynamic Inventory 也不會偵測到這台 host，因為 Dynamic Inventory 是在 playbook 運作前執行的。  

### add_host
add_host 可以指定 Group 跟一些自定義變數：
```yml
- name: Add the vagrant machine to the inventory
      add_host:
        name: default
        group: web
        ansible_host: 127.0.0.1
        ansible_port: 2222
        ansible_user: vagrant
        ansible_private_key_file: >
          .vagrant/machines/default/virtualbox/private_key
```

如果運作成功，這之後你就可以使用 `hosts: default` 這台 host 做後續的工作了。

### group_by
一樣的，我們也可以在運作 playbook 時建立新的 Group。  
像是作業系統的位元(x86, 64)，或是作業系統版本之類的(Ubuntu, CentOS)。

以我們現在的例子來說，有 Ubuntu 跟 CentOS，就可以試試看用作業系統來建立群組：
```yml
---

- name: Group hosts by distribution
  hosts: all
  gather_facts: true
  tasks:
    - name: Create groups based on distro
      group_by:
        key: "{{ ansible_facts.distribution }}"

- name: Do something to Ubuntu hosts
  hosts: Ubuntu
  become: true
  tasks:
    - name: Say I am Ubuntu
      command: echo "I am Ubuntu."
      register: result

    - name: print stdout
      debug:
        msg: "{{ result.stdout }}"
      

- name: Do something else to CentOS hosts
  hosts: CentOS
  become: true
  tasks:
    - name: Say I am CentOS
      command: echo "I am CentOS."
      register: result

    - name: print stdout
      debug:
        msg: "{{ result.stdout }}"

```

如果上面的 Question 你不想試，那就用下面的吧。  
如果順利的話，你應該會看到 vagrant1,2 跟 vagrant3 會有不一樣的訊息，因為他們已經被用版本分成不一樣的 Group。
```bash
$ ansible-playbook -i test.py distribution.yml
PLAY [Group hosts by distribution] ******************************************************

TASK [Gathering Facts] ******************************************************************
ok: [vagrant3]
ok: [vagrant2]
ok: [vagrant1]

TASK [Create groups based on distro] ****************************************************
changed: [vagrant1]
changed: [vagrant2]
changed: [vagrant3]

PLAY [Do something to Ubuntu hosts] *****************************************************

TASK [Gathering Facts] ******************************************************************
ok: [vagrant2]
ok: [vagrant1]

TASK [Say I am Ubuntu] ******************************************************************
changed: [vagrant2]
changed: [vagrant1]

TASK [print stdout] *********************************************************************
ok: [vagrant1] => {
    "msg": "I am Ubuntu."
}
ok: [vagrant2] => {
    "msg": "I am Ubuntu."
}

PLAY [Do something else to CentOS hosts] ************************************************

TASK [Gathering Facts] ******************************************************************
ok: [vagrant3]

TASK [Say I am CentOS] ******************************************************************
changed: [vagrant3]

TASK [print stdout] *********************************************************************
ok: [vagrant3] => {
    "msg": "I am CentOS."
}

PLAY RECAP ******************************************************************************
vagrant1                   : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
vagrant2                   : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
vagrant3                   : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## 結論
這篇文章簡單介紹了 Dynamic Inventory 跟 add_host, group_by 兩個指令。  
透過 Dynamic Inventory 可以減少在 hosts 增減時的工作量，add_host, group_by 這兩個指令可以更靈活的使用 playbook。

下一篇文章會介紹 Variable 跟 facts，關於 playbook 的使用跟運作 ansible-playbook 所回饋的結果。

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [Dynamic Inventory by charlesreid1](https://charlesreid1.com/wiki/Ansible/Vagrant/Dynamic_Inventory)