---
slug: "07_Debug"
title: "Ansible - Debug"
date: "2023-03-04"
lastmod: "2023-03-04"
tags: ["Ansible","Infra","debug"]
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

錯誤經常發生，不論是在 playbook 或是 configuration 上，出錯是正常的，我們要知道的是問題點在哪，並找出方法解決問題。本篇文章將介紹一些在 Ansible 上的故障排除方法與工具。
<!--more-->

## Error Message
舉例來說，我使用上次 show.yml 那個為範例，我將相關的設備關機後並執行：
```bash
$ ansible-playbook show.yml

PLAY [Do something to host] *************************************************************

TASK [Gathering Facts] ******************************************************************
fatal: [vagrant1]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ", "unreachable": true}
fatal: [vagrant3]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ", "unreachable": true}

PLAY RECAP ******************************************************************************
vagrant1                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
vagrant3                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
```

你可以發現到 ansible 回報了錯誤，並提供一些相關資訊，在最下方也有相關的紀錄。（unreachable）  

我們可以透過修改 ansible.cfg 中的 `[default]`，加入`stdout_callback = debug` 讓訊息更容易閱讀：
```yaml
[defaults]
inventory = inventory
stdout_callback = debug
```
```bash
$ ansible-playbook show.yml

PLAY [Do something to host] *************************************************************

TASK [Gathering Facts] ******************************************************************
fatal: [vagrant1]: UNREACHABLE! => {
    "changed": false,
    "unreachable": true
}

MSG:

Failed to connect to the host via ssh:
fatal: [vagrant3]: UNREACHABLE! => {
    "changed": false,
    "unreachable": true
}

MSG:

Failed to connect to the host via ssh:

PLAY RECAP ******************************************************************************
vagrant1                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
vagrant3                   : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0
```

## Troubleshooting

### module
我們可以透過 -m 這個參數，來使用 module 中的一些指令進行 Troubleshooting :

```bash
$ ansible vagrant1 -m ping
vagrant1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ",
    "unreachable": true
}
```
{{< admonition question "Question">}}
那這樣跟我直接用 $ ping vagrant1 有什麼不一樣？我還要打更多的字耶！ 
{{</ admonition >}}

的確是沒有什麼不一樣，但如果你有上百台機器要處理呢？還記得我們前面提到的 Group 嗎？你甚至可以使用 `all` 來協助你同時處理更多機器！
```bash
$ ansible all -m ping
vagrant1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ",
    "unreachable": true
}
vagrant2 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ",
    "unreachable": true
}
vagrant3 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ",
    "unreachable": true
}
```

當然還有許多好用的 module，像是 `command`, `file`, `stat`, `service` 等等。  
官方文件有提供所有的內建 module，有空可以參考（因為真的太多啦！）
* [All modules](https://docs.ansible.com/ansible/2.9/modules/list_of_all_modules.html)

### -v
這個參數可以提供給我們更多的資訊來進行 troubleshooting：
```bash
$ ansible all -v -m ping
```

{{< admonition question "Question">}}
你可以試試看 `-v`, `-vv`, `-vvv`, `-vvvv` 有什麼不同？
{{</ admonition >}}

知道相關資訊之後，可以開始推測原因，也許是我們根本沒辦法到目的機器，也許是機器上的 port 被關了，也許只是網路一時抽筋等等，後續的 troubleshooting 又是另一個世界了。  
一些 troubleshooting 的方式與工具可以參考我的文章。
* [網路中斷](https://ekoismylove.github.io/posts/infra/troubleshooting_1/)
* [服務中斷](https://ekoismylove.github.io/posts/infra/troubleshooting_2/)
* [Slowly](https://ekoismylove.github.io/posts/infra/troubleshooting_3/)

## debugger
在 ansible 2.5 以後的版本中，加入的 debugger 這個功能，讓我們可以逐步執行 playbook。  
```yaml
name: Do something to host
  hosts: 
    - vagrant1
    - vagrant3
  debugger: always
```

你可以輸入 `c` 讓他進行下一步。

```bash
$ ansible-playbook show.yml

PLAY [Do something to host] *************************************************************

TASK [Gathering Facts] ******************************************************************
ok: [vagrant3]
[vagrant3] TASK: Gathering Facts (debug)>
```

這邊提供一些 debugger 支援的指令：

| Command | key | Action |
| :----:| :----: | :----- |
| print | p | 印出 task 的相關資訊 |
| update-task | u | 使用新的 vars 來重新建立 task |
| redo | r | 重新執行 task |
| continue | c | 繼續執行下個 task |
| quit | q | 離開 debugger | 

其中 `p` 後面還可以加上變數來印出所需的訊息。
| Variable | Description |
| :----- | :-----|
| `p task` | 失敗的 task 名稱 |
| `p task.args` | Modules 的參數 |
| `p result` | 失敗的 task 返回的結果 |
| `p vars` | 所有的 vars |
| `p vars[key]` | 特定 variable 的值 | 

## assert
assert 通常是用來做檢查的工具，如果 assert 不滿足指定條件，那就會發生錯誤並終止。
```yaml
task:
  - assert:
      that:
        - 1 > 2
```

因為這個條件不成立，所以會發生錯誤：
```bash
TASK [assert] ***************************************************************************
fatal: [vagrant1]: FAILED! => {
    "assertion": "1 > 2",
    "changed": false,
    "evaluated_to": false
}
```
我們可以用這個方式，來檢查一些 vars 或是 result 是否為我們所想要的數值。  
這邊以 stat 來舉例，把這些 code 放到上次用到的 `show.yml` 後面並做測試。
```yaml
---

- name: Do something to host
  hosts: 
    - vagrant1
    - vagrant3
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

    - name: Print ip address by hostvar
      debug: var=hostvars[inventory_hostname]['ansible_default_ipv4']

    - stat: 
        path: /boot/grub
      register: p

    - assert:
        that:
          - p.stat.exists and p.stat.isdir
...
```
測試結果：~~原來Ubuntu 跟 CentOS 的 grub 不一樣呀~~
```bash
TASK [assert] ***************************************************************************
ok: [vagrant1] => {
    "changed": false
}

MSG:

All assertions passed
fatal: [vagrant3]: FAILED! => {
    "assertion": "p.stat.exists and p.stat.isdir",
    "changed": false,
    "evaluated_to": false
}

MSG:

Assertion failed
```

這邊我們看到了 MSG，感覺就是可以修改 MSG 嘛！  
我們可以透過 `fail_msg` 跟 `success_msg` 來顯示對應的訊息。

```yaml
- assert:
  that:
    - p.stat.exists and p.stat.isdir
  fail_msg: "It's failed."
  success_msg: "It's fine."
```

## Check playbook
在用 playbook 之前呢，我們可以先做一些檢查。

### 語法檢查（--syntax-check）
可以使用 `--syntax-check` 來檢查語法是否有效：
```bash
$ ansible-playbook --syntax-check show.yml
```
### 列出主機（--list-hosts）
透過 `--list-hosts` 可以列出這個 playbook 會用到的 host：
```bash
$ ansible-playbook --list-hosts show.yml
```

那如果我是用 Dynamic 的呢？？  
那就也把他加進來一起檢查囉！你可以在我們在介紹 inventory 時所用的環境做測試。
```bash
$ ansible-playbook --list-hosts -i [script] [your.yml]
```

### 列出任務（--list-tasks）
透過 `--list-tasks` 可以列出 task :
```bash
$ ansible-playbook --list-tasks show.yml
```

### Diff
如果你的 playbook 會在 host 上做修改，你可以使用 `diff` 來觀察修改的情況，他會以 lineinfile 的方式呈現。  
```bash
$ ansible-playbook -D --check playbook.yml
```

## 結論
這次簡單介紹了一些 troubleshooting 的方式，還有在運行 playbook 前可以做哪一些檢查。  
因為 troubleshooting 的方式又是另一個世界，不同的問題需要用不同的方式來找出問題點，並進行近一步的排除，很難針對所有情況來做介紹。  
先前有介紹一些工具可以協助 troubleshooting，如果有空可以去參考一下。

在 playbook 部分有說明的 assert ，可以幫助我們在開發時檢查 vars 跟 result，另外也可以在運行前透過一些方式來檢查 playbook，畢竟建立 connection 也是需要時間。

最後希望大家可以不常常做故障排除。~~傷心傷身~~

## 參考資料
1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [ansible.builtin.assert module – Asserts given expressions are true](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)
3. [All modules](https://docs.ansible.com/ansible/2.9/modules/list_of_all_modules.html)
