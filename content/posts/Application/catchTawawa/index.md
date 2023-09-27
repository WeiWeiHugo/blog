---
slug: "catchTawawa"
title: "星期一就憂鬱？每週透過 Github Actions, Telegram bot 與 Twitter stream 給你豐滿的星期一"
date: "2022-03-14"
lastmod: "2023-02-08"
tags: ["Github Actions","Telegram bot","Twitter"]
sidebar: auto 
categories: ["Application"]
no_comments: false
resources:
- name: "featured-image-preview"
  src: "tawawa.png"
- name: "featured-image"
  src: "tawawa.jpeg"
---

本篇文章將說明如何透過 Github Actions、Telegram bot 與 Twitter stream，定時監聽特定的 Tweet 後發送訊息給 User。

<!--more-->

**2023/02/08 更新 : 因為 twitter API 於 2023/02/09 停止了免費存取，參考這篇文章開發時需要考慮到費用問題，或是選擇其他方式。[Link](https://twitter.com/TwitterDev/status/1621026986784337922)**

因為朋友的關係，知道 Twitter上有一位作家 比村奇石[@Strangestone](https://twitter.com/Strangestone)，而他筆下的 月曜日のたわわ 非常的好(?)。比村老師每週一早上都會在 Twitter 更新，故每週一早上都可以透過推特看到新的作品。月曜日のたわわ是我與朋友們週一時的話題，然而若當天忘記開 Twitter，當下會不知道如何與朋友們討論，或是大家直接忘記有月曜日のたわわ可以看。

為了改善這個問題。朋友開發出了 [TawawaBot](https://github.com/MDSK-UltraIN/TawawaBot_forTelegram)。使用 Telegram bot開發是因為我們主要用 Telegram 聯繫彼此，不過同學是使用 Twitter API v1.1 開發，現在已經有 v2.0 的 API。且同學使用的平台 heroku 最近不知為何，經常會發生問題而使 bot 無法正常執行，~~月曜日的動力就這樣不見了~~。

基於下列兩個原因，才有了這個應用的誕生，
* 第一點是想用 Twitter API v2，因為不知道 v1.1 何時會被棄用而無法運作。
* 第二點是想使用 Github Actions，一來是我不必另外準備環境，開發完成後，透過 Github Action 就會自動執行；二來是不用一直開啟設備，只要在週一的早上 Action 即可，降低一些資源的使用量。

## 建立帳號

前面提到了三個服務，理所當然的要建立三個服務的帳號。

* [Telegram](https://web.telegram.org/)
* [Twitter](https://twitter.com/)
* [Github](https://github.com/)

## Telegram bot
基本上當您建立好帳號後，我們要做兩件事情：
1. 透過 [BotFather](https://t.me/BotFather) 建立 bot。
2. 取得自己的 UserID，或是群組的 ChatID。

### 建立 bot
建立 bot 的方式很簡單，請找 `BotFather` 使用下面三招：
1. /newbot
2. 輸入 bot 的顯示名稱( name )
3. 輸入 bot 的 ID，這邊要注意的是必須以 Bot 或是 _bot 做為結尾，建立完成後會給您這個 bot 的 token，請盡量不要給任何人這個 token，當有這個 token 就可以任意存取你的 bot。

{{<image src="botfather.png" caption="建立完成後，應該會看到這樣的訊息。" width="100%">}}

### 取得 UserID 與 ChatID(group)
取得 UserID 的方式，先向自己剛剛建立的 bot 傳送一個簡單的訊息，然後更改下面的 url 並透過瀏覽器瀏覽，就可以得到 UserID。
```http
# YourBOTToken 改成 剛剛拿到的 token，前後的<>符號要拿掉。 
https://api.telegram.org/bot<YourBOTToken>/getUpdates
```

至於 chatID，則是要先將 bot 邀請到群組內，如果是管理員就直接加入即可，如果不是管理員，請有禮貌地向管理員說明。加入後簡單傳個訊息，並透過上述的方法，就能夠取得 chatID。

## 建立 Twitter 開發專案
### Sign up Twitter Developer Platform
我們先到[這裡](https://developer.twitter.com/en) Sign up，接著應該會導入您剛剛註冊 Twitter 的資訊。並填寫其他資訊。

關於 What's your use case? 這個問題，因為我們是要製作 bot ，所以選擇 Making a bot 這個選項即可。
{{< admonition info "Info">}}
這個選項請不要亂填，若審核沒過，這個帳號以後就不能再申請的樣子。
{{</ admonition >}}

關於 Will you make Twitter content or derived information available to a government entity or a government affiliated entity? 這個問題，選擇 No 即可。

{{<image src="twitter1.png" caption="" width="100%">}}

接著就是開發者的協議與政策，請閱讀過後點選下方的 checkbox。~~我知道很多人都直接點。~~
這個步驟結束後，應該就可以建立 project 了。

### 建立專案並取得 bearer token

請到 Developer Portal 並建立 Project 與 APP，建立完成後應該會給您 bearer token 與其他的 token，但我們只需要使用 bearer token 即可
如果忘記了，進入 APP 後點選 Key and Tokens 的標籤頁，點選 regenerate 後會重新產生一組 bearer token 給您。

{{< admonition warning "Warning">}}
token 務必保存好不要外流，有這個 token 就可以對你的 APP 做一些壞壞的事。
{{</ admonition >}}


{{<image src="twitter2.png" caption="忘記了嗎？點黑底白字的不是點白底黑字的唷" width="100%">}}

## Github Actions 

Github Actions 可以用來自動化、自定義和執行您的軟體開發工作流程，許多人用來做 CI/CD使用。
基本上每一個 Job 可以執行 6 小時，透過 twitter stream 來接收比村老師的推文已經足夠了。
另外有 cron 功能，可以安排行程在我們想要的時間執行，不過時間不太準確就是了。（我設定九點執行，實際到了十點多才執行呢～）

### workflow/main.yml
先在 github 上建立一個專案，待會要設定 Github Actions 進行 schedule 的相關設定。

建立完成後，請到專案的 Actions，點選 "New workflow"，會進入到 Choose a workflow 這個頁面，直接點選下方的 " set up a workflow yourself "，此時 Github 會自動幫你建立一個新的 main.yml，待會就是要編輯這個檔案，透過 Github Actions 進行排程與相關的工作。

我這邊直接提供我的 main.yml 給各位。 [連結](https://github.com/EKOISMYLOVE/CatchTawawa/blob/master/.github/workflows/main.yml)

如果有使用我的 main.yml，建議先把下列兩行**反註解**，這樣每一次 push 時都會即時進行 Action，等確認沒問題再註解即可。
```yaml
 #  push: 
 #    branches: [ main ]
```

另外，Telegram bot 傳送的部分訊息由該檔案產生，要修改傳送的訊息，請修改這裡與後續 Twitter Stream 使用到的程式碼。

### Secrets and Environments
前面提到的 userID(chatID), telegram bot token 與 twitter bearer token，因為這些 token 直接放到 repo 中，就是赤裸裸地給人家看。 Github 提供了 Secret 功能，Secret 是在 repo 中的加密環境變數，可以在 Github Actions 中做使用而不被其他人知道。

請到專案中的 setting 並點選左方的 Secrets -> Actions，並依照以下名稱建立三個 secrets。

* TELEGRAM_TO : Your telegram userId or chatId.
* TELEGRAM_TOKEN : Your telegram bot token.
* TWITTER_BEARER_TOKEN : Your twitter bearer_token.

{{< admonition tips "Tips">}}
因為 main.yml 我已經寫好的關係，這裡有要求名稱。使用上是可以更改名稱的，只要 main.yml 內的變數與 Repository secrets 有符合即可。
{{</ admonition >}}

## Twitter Stream

Twitter API 會透過 Stream 的方式，即時地把符合規則的 tweet 經過 stream 送給我們。  
我直接提供我的的 code。[連結](https://github.com/EKOISMYLOVE/CatchTawawa/blob/master/tawawa.py)  
如果有要更改規則，請修改 `set_rules()` 這個函式內的規則。規則可以參考[官方文件](https://developer.twitter.com/en/docs/twitter-api/tweets/filtered-stream/integrate/build-a-rule)。  
如果想對推文內容進行進一步的過濾，可以到 `get_stream()` 內修改成更詳細的規則。  
Telegram bot 發送的訊息由這個程式產生，若想要修改的話，請修改 `fp.write()` 的傳入參數內容即可。  
測試時，先建議把規則改成自己的 twitter ID 再上傳程式，確認 Github Actions 正常執行後，再推一篇符合規則的 tweet， Telegram bot 應該就會發送 tweet 的內容。

~~完成了，期待週一吧！~~

{{<image src="result.png" caption="如果你看到類似的訊息，代表你的程式正常。" width="100%">}}

## Reference

* 我的朋友 : MDSK-UltraIN
* [How to Create telegram bot](https://core.telegram.org/bots#6-botfather)
* [StackOverflow - Telegram Bot - how to get a group chat id?](https://stackoverflow.com/questions/32423837/telegram-bot-how-to-get-a-group-chat-id)
* [Twitter developer Documentation](https://developer.twitter.com/en/docs/platform-overview)
* [Filtered stream](https://developer.twitter.com/en/docs/twitter-api/tweets/filtered-stream/integrate/build-a-rule)
* [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
* [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)


使用或參考的 packages：
* [MDSK-UltraIN/TawawaBot_forTelegram](https://github.com/MDSK-UltraIN/TawawaBot_forTelegram)
* [twitterdev/Twitter-API-v2-sample-code](https://github.com/twitterdev/Twitter-API-v2-sample-code)
* [theskumar/python-dotenv](https://github.com/theskumar/python-dotenv)
* [appleboy/telegram-action](https://github.com/appleboy/telegram-action)