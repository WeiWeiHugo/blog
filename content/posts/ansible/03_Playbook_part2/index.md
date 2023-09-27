---
slug: "03_Playbook_part2"
title: "Ansible - Playbook(下)"
date: "2022-11-25"
lastmod: "2022-11-25"
tags: ["Ansible","Infra","playbook"]
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
本篇會接續上篇文章做進一步的說明與示範。

<!--more-->

## 事前準備 : TLS
### Info
在前面的例子中，可以觀察到我們開啟了 443 port ， 這通常用在 https 的應用中，透過 SSL/TLS 進行加密達到傳輸安全。  
SSL/TLS 的資訊可以參考[High Performance Browser Networking - Transport Layer Security (TLS)](https://hpbn.co/transport-layer-security-tls/)
{{< admonition info "Info">}}
既然提到了 High Performance Browser Networking 這本書，如果對網路很有興趣的，這邊推推一下這本書給各位。
{{</ admonition >}}

### Generating a TLS Certificate
基本上證書大都是從 Certificate authority (CA) 所頒發，代表這個證書有效，但在後面的例子中我們會使用自行簽署的證書。
```bash
## 注意下指令時的 Directory !
$ openssl req -x509 -nodes -days 365 -newkey rsa:4096 -sha256 -subj /CN=localhost -keyout files/nginx.key -out files/nginx.crt
```

然後我們複製上一篇所使用的 `webservers.yml` 並重新命名為 `webservers-tls.yml`

## Variables (vars)
### Example
在這個範例中，會定義五個變數，並為每個變數賦值：
```yaml
vars:
  tls_dir: /etc/nginx/ssl/
  key_file: nginx.key
  cert_file: nginx.crt
  conf_file: /etc/nginx/site-available/default
  server_name: localhost
```
在這個範例中，每一個值都是字串，但也能使用 boolean, list 跟 dictionary。  
這些變數都是在 nginx 中與 https 有關的參數。 

接著，加入這個 task 到 playbook 中：
```yaml
    - name: Manage nginx config template
      template:
        src: nginx.conf.j2
        dest: "{{ conf_file }}"
        mode: '0644'
      notify: Restart nginx
```

在執行這個 task 的時候，dest 就會被更改為我們前面所設定的 /etc/nginx/site-available/default。

### Quoting in Ansible Strings
有時候我們會想在變數後面直接做一些事情：
```yaml
- name: Perform some task
  command : {{key_file}} -a foo
```

但這樣子，Ansible 會將其視為 dictionary 而返回錯誤，我們必須使用引號：
```yaml
- name: Perform some task
  command : "{{key_file}} -a foo"
```

另外，如果我們的參數中有冒號 `:` ，也需要使用引號：
```yaml
- name: Show msg
  debug:
    msg: "Error : wrong parameter ..."
```

## Template
### Intro
Ansible 主要是做 configure，如果可以避免，大家都不會希望手動編輯一堆 config(cfg)，如果說多個 config 中有重複使用的特定欄位或是數據的位置，會建議另外取得這些資訊，並記錄在一個位置，透過 template 來生成需要這個資訊的 config。  

Ansible 使用 Jinia2 來實現範本化(templating)，就像 Flask、ERB 與 Django 一樣。

Nginx 的設定檔中，需要 TLS 金鑰與證書的路徑，這邊將透過 Ansible 的範本功能來定義這個設定檔。
```bash
$ touch template/nginx.conf.j2
$ vim nginx.conf.j2
## 修改為下面的內容
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        listen 443 ssl;
        ssl_protocols TLSv1.2;
        ssl_prefer_server_ciphers on;
        root /usr/share/nginx/html;
        index index.html;
        server_tokens off;
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;

        server_name {{ server_name }};
        ssl_certificate {{ tls_dir }}{{ cert_file }};
        ssl_certificate_key {{ tls_dir }}{{ key_file }};

        location / {
            try_files $uri $uri/ =404;
        }
}
```

我們在範本檔後面加上 .j2 是來表示這是 jinja2 template，但其實不使用 .j2 也不會影響。  
在這個檔案中，我們定義了先前定義的四個參數。

### Loop
當我們想對 list 中的每一項執行一個 task 時，可以使用 loop，循環多次執行 task，並且每一次都會使用所指定的 list 中的不同值來執行。

```yaml
- name: Copy TLS files
  copy:
    src: "{{ item }}"
    dest: "{{ tls_dir }}"
    mode: '0600'
  loop:
    - "{{ key_file }}"
    - "{{ cert_file }}"
  notify: Restart nginx
```

## Handler
我們在 playbook 中加入 handler :
```yaml
  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```
handler 他類似 task ，但只有在 task 通知後才會運作。通常用在重新啟動服務的時候。  
如果 Ansible 識別出， task 已經更改系統的狀態。那 task 就會發出通知，並將處理程序的名稱做為參數來傳遞。  
在這個例子中，處理程序的名稱是 nginx，如果有下列情況發生，則會重新啟動 nginx：
* TLS key 改變。
* TLS certificate 改變。
* configuration file 改變。
* 站台(site)中的內容改變。

因為在更改服務或系統的設定或是檔案時，大部分的情況需要重新啟動讓服務或系統重新讀取這些設定檔。
後續我們會在每個 task 的後面，加上 notify 讓這些 task 可以做通知。  
另外我們可以在 playbook 後面無條件的重新啟動服務，重新啟動服務並不會浪費很多時間，但要注意服務的性質，例如 nginx 重啟的話，會影響到客戶的 session。  
這個方式也可以在自己想要的位置使用，執行某個或某些 task 之後就強制使用 handler。  

```yaml
    - name: Restart nginx
      meta: flush_handlers
```

最後應該會有一個類似這樣的結果：
```yaml
---

- name: Configure webserver with nginx
  hosts: webservers
  become: True
  # 關掉這個可以讓他跑快一點！
  gather_facts: false

  vars:
    tls_dir: /etc/nginx/ssl/
    key_file: nginx.key
    cert_file: nginx.crt
    conf_file: /etc/nginx/sites-available/default
    server_name: localhost

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted

  tasks:
    - name: Ensure nginx is installed
      package: 
        name: nginx 
        update_cache: true
      notify: Restart nginx
    
    - name: Create directories for TLS certificates
      file:
        path: "{{ tls_dir }}"
        state: directory
        mode: '0750'
      notify: Restart nginx

    - name: Copy TLS files
      copy:
        src: "{{ item }}"
        dest: "{{ tls_dir }}"
        mode: '0600'
      loop:
        - "{{ key_file }}"
        - "{{ cert_file }}"
      notify: Restart nginx

    - name: Copy nginx config file
      copy:
        src: nginx.conf
        dest: /etc/nginx/sites-available/default

    - name: Manage nginx config template
      template:
        src: nginx.conf.j2
        dest: "{{ conf_file }}"
        mode: '0644'
      notify: Restart nginx

    - name: Enable configuration
      file: 
        dest: /etc/nginx/sites-enabled/default
        src: /etc/nginx/sites-available/default
        state: link

    - name: Install home page
      template:
        src: index.html.j2
        dest: /usr/share/nginx/html/index.html
        mode: '0644'

    - name: Restart nginx
      meta: flush_handlers
...
```

## 測試
在進行測試以前，我們應該先檢查語法！
```bash
$ ansible-playbook --syntax-check webservers-tls.yml
$ ansible-lint webservers-tls.yml
$ yamllint webservers-tls.yml
$ ansible-inventory --host testserver -i inventory/vagrant.ini
$ vagrant validate
```

檢查一下目錄！
```bash
.
├── Vagrantfile
├── ansible.cfg
├── files
│   ├── nginx.conf
│   ├── nginx.crt
│   └── nginx.key
├── inventory
│   └── vagrant.ini
├── templates
│   ├── index.html.j2
│   └── nginx.conf.j2
├── webservers-tls.yml
└── webservers.yml
```

如果沒有問題，跟以前一樣！
```bash
$ ansible-playbook webservers-tls.yml
```

應該能夠透過 `https://localhost:8443` 看到網頁，只是說因為我們是用自簽署憑證。瀏覽器會有警告訊息！

## 結論
本篇文章簡單說明了 vars 跟 handler，並建立了簡單的 template 來運行。

其實這邊應該是要跟上一篇一起寫完的，只是覺得內容有點多，就分成了兩篇來寫。  
下一篇應該會介紹 Inventory 吧，~~應該吧。~~

## 參考資料

1. [Ansible: Up and Running: Automating Configuration Management and Deployment the Easy Way (3rd edition)](https://www.amazon.com/Ansible-Automating-Configuration-Management-Deployment/dp/1491979801)
2. [High Performance Browser Networking](https://hpbn.co/)