---
slug: "blog_githubAction"
title: "Hugo 部落格建置 - 透過 Github Action 編譯與部署"
date: "2023-09-27"
lastmod: "2023-09-27" 
tags: ["Hugo","blog","Web","Github Action"]
sidebar: auto 
categories: ["blog"]
draft : true
resources:
- name: "featured-image-preview"
  src: "hugo_logo.png"
- name: "featured-image"
  src: "hugo_logo.png"
---
本篇文章介紹如何在 Github 建立 Hugo 的編譯環境，並且透過 Github Action 部署到Github 的個人網站上。
<!--more-->
---

## 前言

之所以會寫這邊，一來是因為有點忘記當初是怎麼架設的，二來是這個 blog 被看到了，對方敲碗詢問怎麼運作的，於是才有了這篇文章。

在看這篇文章之前，請先看[Hugo 部落格建置 - 安裝與部署](https://ekoismylove.github.io/posts/blog/blog_build/)這邊文章，我們先把本地的環境架設好，再開始做後續的事情！

---
## 建立 repo

在這之前，我們要先建立兩個 repo，一個請你用你的帳號名稱命名，格式為[username].github.io，我這次的範例是 weiweihugo.github.io，這邊是要存放編譯後所產生的網頁檔。這個 repo 請選擇 public，因為是要給別人看的。  

{{< admonition info "Info">}}
如果你使用其他名稱，那你連上去的url就要加上你的 repo name。  
像是我使用 test 的話，那我就要用 weiweihugo.github.io/test 才看得到。
{{</ admonition >}}

另一個 repo 請你用你自己好記的名稱命名即可，這邊是要存放你的編譯環境，我們之後會把我們在本地端的環境 push 上去。這個 repo 請務必選擇 private，因為會存放 key，避免 key 外流而使得他人可以對你的 repo 動手腳。

建立好了之後，我們先隨意上傳一個 index.html 並設定，確認我們是否能透過 [username].github.io 瀏覽到網頁。這邊的 index.html 就自己先隨意設計一個即可。

如果上傳成功後，你應該能直接透過 [username].github.io 瀏覽到網頁。

{{<image src="firstpush.png" caption="如果看到這個，表示沒問題。" width="100%">}}

## 建立 workflow
我們進入那個 private 屬性的 repo，直接點選上方的 action，並選擇 Simple workflow。

{{<image src="simple_workflow.png" caption="點這個" width="100%">}}



官方在 macOS 底下說明了使用 brew 與 port 兩種安裝方式，我會比較推薦使用 brew，如果環境是 Windows 或是 Linux 的使用者，[官方文件](https://gohugo.io/getting-started/quick-start/)有提到相關的安裝方式，這裡就不再加以說明。
```shell
$ brew install hugo
```

安裝後，可以使用 hugo version 確認是否安裝完成。
```shell
$ hugo version
hugo v0.92.2+extended darwin/amd64 BuildDate=unknown
```


若安裝完成後，到自己的專案路徑底下建立站台專案。
```shell
### hugo new site [sitename]
$ hugo new site quickstart
```
---
## 安裝主題 (theme)
Hugo 提供了許多種[主題](https://themes.gohugo.io/)，或是到 Google 搜尋，基本上挑選自己喜歡的即可，我這邊選擇的是 [LoveIt](https://hugoloveit.com/)。

進入到站台專案，並使用 git 將 LoveIt 主題下載下來。
```shell
$ cd quickstart 
$ git clone https://github.com/dillonzq/LoveIt.git themes/LoveIt
```
---
## 增加內容
通常內容都會放置在專案內的 content 資料夾中，可以手動新增，也可以使用 hugo new 的方式新增，會自動在 frontmatter 補上標題與時間的資訊。
```shell
$ hugo new posts/first.md
```

產生檔案後，就用 markdown 語法寫文章即可，這邊要提醒一點是使用圖片的方式，要使用 HTML tag，而不能使用 ```![name](url)``` 的方式。
```html
<!-- 請參考下面的程式碼，使用時請手動刪除 {{與< 和 >與}} 之間的空格。-->
{{ <image src="folder/filename" caption="This is a image." > }}
```
---
## 瀏覽
當內容新增完後，使用 hugo server -D，然後在瀏覽器輸入 http://localhost:1313/，就可以瀏覽了。
我自己會偏好使用 hugo serve -e production，使用生產環境的方式進行預覽，畢竟是要產生靜態網頁部署至 github page 做使用。只不過當directory 或是 path 更新時，會有一些問題發生，但基本上重新編譯即可。
```shell
$ hugo server -D
### Or
$ hugo serve -e production
```
---
## 部署至 github
瀏覽網站並確認完畢後，首先先執行一次 hugo，這樣會在專案資料夾底下產生一個 /public，裡面的檔案就是 hugo 產生的靜態網頁內容，進入到 /public，將這些檔案 push 到自己的 username.github.io 的專案，稍等一下下後到 ```https://username.github.io```，應該就可以看到網頁了。
```shell
cd public
git init
git add .
git commit -m 'my first deploy'
git remote add origin git@github.com:username/repo.git
git push -u origin master
```

若對 git 不太熟悉的話 ~~(像我)~~，git 與 github 的相關資訊，會附在參考資料作為參考。

---

基本上上傳完成且能看到網頁內容後，自己的部落格應該就完成了，往後新增或編輯文章後，就是再使用 hugo 重新產生靜態內容，並透過 git 將靜態內容上傳後即可。

後續有時間會再說明 theme 設定與一些功能的建置(SEO、algolia 與 disqus...等)。

---
## 參考資料
1. [Hugo 官方網站](https://gohugo.io/)
2. [Hugo quickstart](https://gohugo.io/getting-started/quick-start/)
3. [Hugo themes](https://themes.gohugo.io/)
4. [Github pages](https://pages.github.com/)
5. [Git 教學：如何 Push 上傳到 GitHub？](https://gitbook.tw/chapters/github/push-to-github)