---
slug: "blog_build"
title: "Hugo 部落格建置 - 安裝與部署"
date: "2022-02-17"
lastmod: "2022-02-20" 
tags: ["Hugo","blog","Web"]
sidebar: auto 
categories: ["blog"]
resources:
- name: "featured-image-preview"
  src: "hugo_logo.png"
- name: "featured-image"
  src: "hugo_logo.png"
---
本篇文章基本上是介紹如何用 Hugo 建置 blog，並部署至 Github pages 上。
<!--more-->
---

## 前言
{{<image src="./hugo_logo.png" width="100%">}}


基本上建置這個部落格是為了紀錄自己的筆記，與自己遇到的問題和如何解決的方式，但從無到有建置一個非常麻煩，又不太想用像是 Medium 之類的網站，會少了一點自由，故想說使用框架產生靜態網頁的方式建置。

最一開始使用 Vuepress，使用了一陣子之後覺得用起來不是很順手，過了一陣子則有了換框架的想法，於是先換了 Docusaurus，~~因為小恐龍很可愛~~，沒想到 Docusaurus 對於自己來說，更難用！不到一天的使用時間，決定再次換框架，使用了 Hugo 建置，雖然搬遷很煩躁，但開始搬遷時第一個反應是：
> 哇塞，太快了吧！

應該會有很長的一段時間，使用 Hugo，因此也想紀錄這些建置過程，作為自己的筆記，也希望可以分享給大家。

---
## 安裝 Hugo

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








