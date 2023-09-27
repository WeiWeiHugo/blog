---
slug: "github_actions_guide"
title: "自己動起來！ Github Actions 極簡單介紹與使用"
date: "2022-04-01"
lastmod: "2022-04-01"
tags: ["Github Actions"]
sidebar: auto 
categories: ["Infra"]
no_comments: false
draft : false
resources:
- name: "featured-image-preview"
  src: "github_actions_header.png"
- name: "featured-image"
  src: "github_actions_header.png"
---

簡單介紹 Github Actions，因為有人看了星期一就憂鬱那篇文章而詢問 Github Actions 的使用，故想說透過這次機會簡單介紹下。

<!--more-->

## 簡介

Github Actions 可以用來自動化、自定義和執行您的軟體開發工作流程。  
許多人用來做 CI/CD 使用，透過 Github Actions 進行程式的建構與測試，並進行相關的部署。    

在 Github 上，可以透過 push, create issue 或是排程的方式啟動 workflow。
並提供了多種 Runner，依據不同的需求使用不同的 Runner，能夠輕易的在不同的平台上做測試或是建構。

使用上也可以透過 log 觀察運作的情形並進行修改；且提供了 secret 的功能，讓一些需要 token 或是 key 的應用能夠安全的運作。  
Github Actions 也有相關的社群，且能夠透過 Marketplace 取得其他開發者設計完成的 Actions，或者是發布自己的 Actions。

**簡單來說，Github 給你一個平台，你可以在上面進行 CI/CD。**

{{< admonition warning "Warning">}}
Github Actions 有使用時間與容量限制，使用上請注意。[(Billing for Github Actions)](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)
{{</ admonition >}}

## 基本使用
### 建立 workflow
基本上點選專案上方的 Actions，會進入到這個畫面：

{{<image src="github_1.png" caption="修改前的檔案" width="100%">}}

Github 會推薦給您一些 Actions，如果有需要的人可以點選做使用，就不必再重造一次輪子，但這次是要做個簡單介紹與使用，這邊我們點選上方的 `set up a workflow yourself`。會提供一個範例檔案：

```yaml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v3

      # Runs a single command using the runners shell
      - name: Run a one-line script
        run: echo Hello, world!

      # Runs a set of commands using the runners shell
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```
當滿足 `push` 或是 `pull_request` 時，都會在 Github actions 內看到相關的資訊與 `echo` 出來的文字。

基本上我會把這個 main.yml 分成三個部分：
* name
* on
* job

name : 代表這個 workflow 的名稱，就單純是個名稱，這個欄位可以省略不寫，省略時會使用 *.yml 的名稱。  
on : 控制這個 workflow 如何觸發或是何時要運作。  
job : 就是我們希望他能幫我們運作的工作，job 可以依序進行，或是以平行方式同時進行。  
除了 name 以外， on 與 jobs 會簡單介紹與使用。

### on
觸發方式有很多種，我目前是比較常使用 `schedule` 與 `push` 兩種方式。  
`schedule` 通常會搭配 `cron` 做排程使用，但我自己的經驗是觸發時間不太準確，如果專案很在意觸發時間的請使用其他方式。  
`push` 則是當你推送內容至 branch 時，就會觸發，我編輯 yaml 並測試時是採用這個方式進行測試。  
```yaml
on:
  push:
    branches: ["main"]
  schedule:
    - cron: '0 1 * * 1'
```

Github 有個說明文件提供了可使用的觸發方法：[Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push)  
`cron` 的使用方式，就像你在 Linux 上使用 `crontab` 是一樣的，這邊提供時間設定的方式參考：[crontab guru](https://crontab.guru/)

### jobs
由許多個 step 所組成，step 則是實際要運作的指令，job 之間是平行處理，若 job 之間有依賴性，需要透過 need 進行設定。  
如下列的程式碼， `deploy` 需要等待 `build` 完成後才會進行。  
進一步的話可以透過 `if` 判斷要依賴的 `jobs` 的狀態再進一步處理。
```yaml
jobs:
  build:
    steps:
        // steps, steps and steps.
  deploy:
    needs: build
    if: needs.build.result == 'success'
    steps:
        // steps, steps and steps.
```

另外，我們使用的 runner 也是在這邊透過 `runs-on` 做設定：
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

`runs-on` 可以設定要使用 Github 提供的 runner 其中之一，或是自行提供相關設備連線進行使用。  
關於更多 `runs-on` 的使用方式，可以參考官方文件：[Hosting your own runners](https://docs.github.com/en/actions/hosting-your-own-runners)

## steps
基本上運行 steps 都會先建議用一次 `actions/checkout`，看設定是否有問題，這邊我們會透過 `uses: actions/checkout@v3` 做這件事情，`uses` 可以讓我們使用在 github 上，其他開發者寫好的 action。 
{{< admonition tips "Tips">}}
有些 actions 在 `actions/checkout@v3` 會發生問題，可以採用 `actions/checkout@v2` ，或是看用的 actions 是否有釋出新版本來解決這個問題。
{{</ admonition >}}

或是自行定義，以 github 提供的範本做說明：
``` yaml
- name: Run a one-line script
  run: echo Hello, world!
```
我定義了一個名稱為 `Run a one-line script` 的 workflow，並讓他執行 `echo Hello, world!` 這行指令。  
當我們透過這種方式定義時，還可以使用一些參數協助設定，像是從 secrets 中取出我們的 secret 並放到環境變數中。(env)
```yaml
- name: Export Variables
  env:
    BEARER_TOKEN: ${{ secrets.TWITTER_BEARER_TOKEN }}
```
或是透過 with 加入更多的變數做使用。

```yaml
- name: send tweet content for telegram message
  uses: appleboy/telegram-action@master
    # These are environments secrets.
    with:
      to: ${{ secrets.TELEGRAM_TO }}
      token: ${{ secrets.TELEGRAM_TOKEN }}
      message: ${{ steps.tweet.outputs.result }}
```

更多 workflow 的使用方式，可以參考官方文件：[Using workflows](https://docs.github.com/en/actions/using-workflows)

## Secret
有一些敏感的資訊，不能放在程式碼內，因為會被他人看到進而被做其他利用。  Github 提供了 secret 的功能，讓我們可以存放這些敏感資訊並使用。  
進入專案的 setting，並點選左側的 Secret 標籤即可設定。
{{<image src="github_2.png" caption="修改前的檔案" width="100%">}}

secret 分成兩種：
* Environment secrets
* Repository secrets

Environment secrets 主要是給 Github actions 作為環境參數使用，而 Repository secrets 則是給 Repository 做使用。  
依據需求建立即可，另外設定好的 secrets 自己要另外儲存一份，因為儲存過後的 secrets 就真的是秘密，看不到了！

## 參考資料
1. [Github Actions](https://github.com/features/actions)
2. [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push)
3. [crontab guru](https://crontab.guru/)
4. [Hosting your own runners](https://docs.github.com/en/actions/hosting-your-own-runners)
5. [GitHub Marketplace · Actions to improve your workflow](https://github.com/marketplace?type=actions)