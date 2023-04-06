---
layout: post
title:  "notification-server"
gh-repo: daattali/beautiful-jekyll
tags: [iOS]
date:   2023-04-06 01:02:50 +0800
categories: Tony update
---
# iOS遠端推播伺服器教學

iOS遠端推播（Remote Notification）是一種讓伺服器能夠向使用者的裝置發送通知的機制，例如訊息、新聞、更新等。要使用iOS遠端推播，需要先在Apple Developer Portal上註冊一個App ID，並啟用Push Notification功能。然後，需要產生一個p8憑證（APNs Auth Key），並下載到本地端。這個憑證可以用來驗證伺服器的身份，並向Apple Push Notification service (APNs)發送推播請求。

## 什麼是p8憑證？

p8憑證是一種用於驗證與APNs的連線的檔案，它包含了一個私鑰和一個團隊ID。p8憑證可以在Apple Developer網站上產生，並且可以用於多個App ID。使用p8憑證的好處是，不需要每年更新，也不需要管理多個設備的token。

## 如何產生和使用p8憑證？

要產生p8憑證，你需要有一個Apple Developer帳號，並登入到[Apple Developer網站](https://developer.apple.com/)。然後，按照以下步驟操作：

1. 在左側選單中，選擇Certificates, Identifiers & Profiles。
2. 在左側選單中，選擇Keys。
3. 在右上角，點選+按鈕。
4. 輸入一個Key Name，例如My Push Key。
5. 在Key Services中，勾選Apple Push Notifications service (APNs)。
6. 點選Continue。
7. 點選Register。
8. 點選Download，將p8憑證下載到你的電腦上。
9. 點選Done。

下載好p8憑證後，你需要記住以下兩個資訊：

- Key ID：這是一個10位數的字母和數字組成的代碼，例如ABCD1234EF。
- Team ID：這是一個10位數的字母組成的代碼，例如ABCDEFGHIJ。

你可以在Apple Developer網站上的Membership頁面中找到你的Team ID。

## 推播請求的格式和內容

要發送一個推播請求，你需要準備以下三個部分：

- 一個HTTP/2連線，連接到APNs的API端點。目前有兩個端點可供選擇：開發環境（api.development.push.apple.com）和生產環境（api.push.apple.com）。你應該根據你的應用程式的環境來選擇合適的端點。
- 一個HTTP/2 POST請求，包含以下資訊：
  - 一個URL路徑，格式為`/3/device/{deviceToken}`，其中`{deviceToken}`是你要發送推播的目標裝置的唯一識別碼。你可以在你的應用程式中使用`didRegisterForRemoteNotificationsWithDeviceToken`方法來取得這個裝置權杖。
  - 一組HTTP標頭，包含以下資訊：
    - `apns-id`：一個UUID，用來識別這個推播請求。如果你不提供這個標頭，APNs會自動生成一個。
    - `apns-expiration`：一個UNIX時間戳記，表示這個推播請求的有效期限。如果你不提供這個標頭，或者提供0值，則表示這個推播請求只會嘗試發送一次。如果你提供一個非0值，則表示如果目標裝置暫時無法收到推播，APNs會在這個時間戳記之前重試發送。
    - `apns-priority`：一個數字，表示這個推播請求的優先級。你可以提供10或5兩種值。10表示立即發送，5表示可以延遲發送。如果你不提供這個標頭，則預設為10。
    - `apns-topic`：一個字串，表示這個推播請求的主題。通常是你的應用程式的bundle ID。如果你使用p8憑證來驗證身份，則必須提供這個標頭。
    - `authorization`：一個字串，表示你的身份驗證資訊。如果你使用p8憑證來驗證身份，則必須提供使用p8憑證生成的JWT（JSON Web Token）。
- Payload：一個JSON物件，payload必須包含一個`aps`鍵，其值是一個物件，包含了推播的基本資訊，例如：

    ```json
    {
      "aps" : {
        "alert" : {
          "title" : "標題",
          "body" : "內容"
          },
          "sound" : "default",
          "badge" : 1
      },
      "customKey" : "customValue"
    }
    ```

    - `alert`：推播的訊息內容，可以是一個字串或一個物件。如果是一個物件，可以包含以下的鍵：
    - `title`：推播的標題，顯示在通知中心和鎖定畫面上。
    - `body`：推播的內容，顯示在通知中心和鎖定畫面上。
    - `subtitle`：推播的副標題，顯示在通知中心和鎖定畫面上。
    - `launch-image`：推播觸發時要顯示的圖片名稱。
    - `sound`：推播觸發時要播放的聲音名稱，可以是預設的`default`或自定義的聲音檔案名稱。
    - `badge`：推播觸發時要更新的App圖示上的數字。
    - `content-available`：如果設為1，表示這是一個靜默推播（silent notification），不會有任何視覺或聽覺效果，但會喚醒App並執行一些背景任務。
    - `mutable-content`：如果設為1，表示這是一個可變更內容的推播（mutable-content notification），可以讓App在收到推播後修改其內容或附件，例如加入圖片或影片等。

  除了`aps`鍵之外，payload還可以包含其他自定義的鍵值對，例如上面例子中的`customKey`和`customValue`。這些自定義的資料可以在App收到推播後取得並處理。

  payload的大小限制為4KB，如果超過這個限制，APNs會拒絕發送。

## 產生JWT

要使用p8憑證發送推播訊息，你需要將它轉換成一個JWT（JSON Web Token），這是一種用於驗證和傳輸資料的格式。使用python來產生JWT，我們需要安裝pyjwt套件。這是一個用來編碼和解碼JWT的python庫，可以使用pip指令安裝：

```bash
conda install pyjwt
```

我們需要使用pyjwt套件的encode方法。這個方法接受三個參數：payload、key和algorithm。payload是一個字典，包含了JWT的內容資訊。key是一個字串或一個bytes物件，表示我們的p8憑證。algorithm是一個字串，表示我們使用的加密演算法。在這裡，我們使用ES256演算法，這是Apple官方建議的演算法。

我們的payload需要包含以下三個欄位：

- iss：表示我們的Team ID
- iat：表示JWT的發行時間（以秒為單位）
- kid：表示我們的Key ID

```python
import jwt
import time

# 設定p8憑證的路徑、Key ID、Team ID和App Bundle ID
p8_file = 'path/to/p8/file'
key_id = 'ABCD1234EF'
team_id = 'ABCDEFGHIJ'
bundle_id = 'com.example.app'

# 產生JWT
token = jwt.encode(
# 設定JWT的payload，包含iss（發行者）、iat（發行時間）和aud（接收者）
{
'iss': team_id,
'iat': time.time()
},
# 讀取p8憑證的內容
open(p8_file, 'r').read(),
# 設定JWT的header，包含alg（演算法）和kid（Key ID）
headers={
'alg': 'ES256',
'kid': key_id
}
)
```

## 遠端推播的 HTTP/2 請求格式

要向 Apple 推播服務發送遠端推播通知，您需要使用 HTTP/2 協議發送 POST 請求到 https://api.push.apple.com/3/device/<device-token> 端點，其中 <device-token> 是目標裝置的裝置令牌。

我們可以使用 Hyper HTTP/2 客戶端與 APNs 進行 HTTP/2 通信。以下是建立 HTTP/2 連接的 Python 代碼範例：

```python
import hyper

push_host = 'api.push.apple.com'
push_port = 443

conn = hyper.HTTP20Connection(push_host, port=push_port, secure=True)

headers = {
    'authorization': 'bearer {}'.format(jwt_token),
    'content-type': 'application/json'
}

path = '/3/device/{}'.format(device_token)
```
### 發送推播通知

```python
import json

payload = {
    'aps': {
        'alert': {
            'title': 'Hello, World!',
            'body': 'This is a remote notification!'
        },
        'badge': 1
    }
}

conn.request('POST', path, body=json.dumps(payload), headers=headers)
resp = conn.get_response()
print(resp.status)
print(resp.read())
```
