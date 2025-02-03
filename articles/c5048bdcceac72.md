---
title: "コピペで出来るAdMob Slack連携方法🎉"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "AdMob", "Slack", "Android", "iOS"]
published: true
---

# 概要

AdMob を使用してる皆さん、収益の確認はどのようにしていますか？
僕は今まで毎日収益ページを開いて確認していたのですが、日に日に面倒になってしまい、最近では開かないことが多くなってしまいました。
かといって、収益はある程度毎日確認して、日々のモチベにしたい思いもあります。。
ということで、GAS を使用して**自動で前日の AdMob の収益を Slack に連携する仕組み**を作ったので、今回はその方法を紹介します 🙌
大体コピペで完成するはずです 🙆‍♂️
この方法を使うことで、下記のように毎日 Slack に収益の結果が報告されます！

![](https://storage.googleapis.com/zenn-user-upload/790cb048714e-20240512.png)

# 全体の流れ

使用技術は、[GAS](https://www.google.com/script/start/)、[Slack App](https://api.slack.com/messaging/webhooks), [AdMob API](https://developers.google.com/admob/api/v1/getting-started?hl=ja) です。
流れとしては、GAS 上で AdMob API を叩き、その結果をもとに Slack へ送信します。
また、GAS では定期実行の設定ができるため、そちらを使用して毎日の収益レポートを Slack へ送信します。

# 手順

## 1. AdMob API の準備

https://developers.google.com/admob/api/v1/getting-started?hl=ja

AdMob API を使用できるように GCP プロジェクト作成します。
上記で紹介されている通りなので、ざっと説明します。

1. GCP プロジェクトの作成
   - AdMob API にアクセスするには GCP プロジェクトが必要なので、新しく作成します。
2. AdMob API の有効化
   - GCP では特定の API を使用するために、有効化しないといけないので、AdMob API を有効化します。
   - https://console.cloud.google.com/apis/library/admob.googleapis.com?hl=ja&project=pc-api-7615661539720297960-589
3. OAuth2 認証情報を作成
   - [認証情報を作成] を作成から [OAuth クライアント ID] を作成します。
   - 僕はアプリケーションの種類として、ウェブアプリケーションを選択しました。
   - シークレットの JSON を保存します。後述する Apps Script の処理で使用します。

## 2. Slack App の準備

https://api.slack.com/messaging/webhooks

Slack App を作成し、Webhook URL 経由でメッセージを送信できるようにします。
上記で説明されている通りなので、ざっと説明します。

1. Slack App を作成します。
2. incoming webhooks を有効化&作成します。
3. 作成した webhooks URL を保存します。後述する Apps Script の処理で使用します。

## 3. スプレッドシート & Apps Script 用意

AdMob API を叩いて、Slack に連携する下準備ができたので、ここからは具体的に送信する処理について説明します。

1. 新しくスプレッドシートを作成します。
2. 「拡張機能＞ Apps Script」から Apps Script プロジェクト作成します。

:::message
必ずスプレッドシートから Apps Script プロジェクト作成しましょう。
Apps Script から直でプロジェクトを作成してしまうと、スプレッドシートと Apps Script が連携してない状態になり、後述の手順でエラーになります。
:::

## 4. appsscript.json の追加

Apps Script プロジェクト左のサイドバーにある「プロジェクトの設定」を選択し、「「appsscript.json」マニフェスト ファイルをエディタで表示する」を有効にしてください。
エディタに appsscript.json が追加されるので、下記のような形式と`oauthScopes` を追加してください。

```json:appsscript.json
{
  "timeZone": "Asia/Tokyo",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/admob.report",
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/script.container.ui"
  ]
}
```

## 5. apps-script-oauth2 のセットアップ

https://github.com/googleworkspace/apps-script-oauth2

GAS 上で oauth2 を使用するために、[apps-script-oauth2](https://github.com/googleworkspace/apps-script-oauth2)を使用します。
導入するためにやることは下記です。

1. GAS プロジェクトのライブラリで、`1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF`で検索し、ライブラリを追加する
2. リダイレクト URL を作成し、GCP プロジェクトで作成したクライアント ID の承認済みリダイレクト URL に登録する
   - `https://script.google.com/macros/d/{SCRIPT ID}/usercallback`
     ![](https://storage.googleapis.com/zenn-user-upload/26b4180d3ed6-20240512.png)

## 6. 認証処理

認証に関する処理をまとめた `auth.gs` ファイルを追加します。

:::details auth.gs

```js:auth.gs
var CLIENT_ID = PropertiesService.getScriptProperties().getProperty('client_id'); // OAuth 2.0 クライアント IDのシークレットjsonに含まれる
var CLIENT_SECRET = PropertiesService.getScriptProperties().getProperty('client_secret'); // OAuth 2.0 クライアント IDのシークレットjsonに含まれる
const admobAPIService = getAdmobAPIService();

// スプレッドシートのメニューにAdmob API認可用のボタンを設置
// スプレッドシートを開いたタイミングで、「Admob API連携」メニューが追加される。
function onOpen(e) {
  SpreadsheetApp.getUi()
    .createMenu("Admob API連携")
    .addItem("認可処理", "initAuth")
    .addToUi();
}

// Admob API認可URLをダイアログに表示
function initAuth() {
  const authorizationUrl = admobAPIService.getAuthorizationUrl();
  const template = HtmlService.createTemplate(`
    <html>
      <head>
        <style>
          body, html {
            font-family: 'Arial', sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f4f4f4;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100%;
          }
          .button {
            padding: 10px 20px;
            font-size: 16px;
            text-decoration: none;
            color: white;
            background-color: #4CAF50;
            border: none;
            border-radius: 5px;
            cursor: pointer;
          }
          .button:hover {
            background-color: #45a049;
          }
        </style>
      </head>
      <body>
        <a href="<?= authorizationUrl ?>" target="_blank" class="button">認可する</a>
      </body>
    </html>
  `);
  template.authorizationUrl = authorizationUrl;
  const html = template.evaluate();
  const title = "Admobアプリの認可処理";
  const htmlOutput = HtmlService.createHtmlOutput(html)
    .setWidth(400)
    .setHeight(150);
  SpreadsheetApp.getUi().showModelessDialog(htmlOutput, title);
}

// OAuthの準備
function getAdmobAPIService() {
  return OAuth2.createService("AdmobAPI")
    .setAuthorizationBaseUrl("https://accounts.google.com/o/oauth2/auth")
    .setTokenUrl("https://oauth2.googleapis.com/token")
    .setClientId(CLIENT_ID)
    .setClientSecret(CLIENT_SECRET)
    .setCallbackFunction("authCallback")
    .setPropertyStore(PropertiesService.getUserProperties())
    .setScope("https://www.googleapis.com/auth/admob.readonly")
    .setParam("access_type", "offline")
    .setParam('prompt', 'consent');
}

// 認証処理後のコールバックの処理を定義
function authCallback(request) {
  const isAuthorized = admobAPIService.handleCallback(request);
  if (isAuthorized) {
    var htmlContent = `
    <!DOCTYPE html>
    <html>
    <head>
      <style>
        body {
          font-family: Arial, sans-serif;
          background-color: #f8f9fa;
          text-align: center;
          padding: 50px;
        }
        .message {
          background-color: #c8e6c9;
          color: #256029;
          padding: 20px;
          border-radius: 8px;
          display: inline-block;
          max-width: 300px;
        }
      </style>
    </head>
    <body>
      <div class="message">
        認可処理が正常に終了しました。このタブを閉じてください。
      </div>
    </body>
    </html>
    `;
    return HtmlService.createHtmlOutput(htmlContent);
  } else {
    var htmlContent = `
    <!DOCTYPE html>
    <html>
      <head>
        <style>
          body {
            font-family: Arial, sans-serif;
            background-color: #f8f9fa;
            text-align: center;
            padding: 50px;
          }
          .message {
            background-color: #ffcdd2;
            color: #c63737;
            padding: 20px;
            border-radius: 8px;
            display: inline-block;
            max-width: 300px;
          }
        </style>
      </head>
      <body>
        <div class="message">
          認可処理に失敗し、リソースへのアクセスが拒否されました。設定を確認してください。
        </div>
      </body>
    </html>
    `;
    return HtmlService.createHtmlOutput(htmlContent);
  }
}
```

:::

やってることは下記です。

1. `onOpen`でスプレットシート開いた時に、「Admob API 連携」メニューを表示させる。
   ![](https://storage.googleapis.com/zenn-user-upload/b10b9cd6d557-20240512.png)
2. スプレッドシート上で認可処理が選択されたら、`initAuth`が走り認証処理が始まる。
   ![](https://storage.googleapis.com/zenn-user-upload/98dfca5acdf4-20240512.png)
   ![](https://storage.googleapis.com/zenn-user-upload/89b53e0ba52d-20240512.png)
3. 認証処理が完了すると、`admobAPIService`を使用して AdMobAPI を使用できるようになります 🎉

## 7. モデル追加

モデルに関する処理をまとめた `model.gs` ファイルを追加します。
API のデータを受け取るためのモデルです。

:::details model.gs

```js
// 収入レポート
class RevenueReport {
  constructor() {
    this.totalEarnings = 0;
    this.entries = [];
  }

  addEntry(revenueData) {
    this.entries.push(revenueData);
    this.totalEarnings += revenueData.earnings;
  }
}

// 各アプリの収入
class RevenueData {
  constructor(platform, appName, earnings) {
    this.platform = platform;
    this.appName = appName;
    this.earnings = earnings;
  }
}
```

:::

## 8. AdMob API 通信処理

AdMob API に関する処理をまとめた`admob.gs` ファイルを追加します。

:::details admob.gs

```js:admob.gs
var ADMOB_ACCOUNT_ID = PropertiesService.getScriptProperties().getProperty('admob_account_id'); // ex. pub-xxx

// 昨日のアプリ収益を取得する
// Admob APIをを叩いている
function fetchYesterdayRevenueReport() {
  var yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1); // 前日の日付を設定
  var accessToken = admobAPIService.getAccessToken();  // 認証情報としてアクセストークンを設定

  var options = {
    'method': 'post',
    'contentType': 'application/json',
    'headers': {
      'Authorization': 'Bearer ' + accessToken
    },
    'payload': JSON.stringify({
      "reportSpec": {
        "dateRange": {
          "startDate": {"year": yesterday.getFullYear(), "month": yesterday.getMonth() + 1, "day": yesterday.getDate()},
          "endDate": {"year": yesterday.getFullYear(), "month": yesterday.getMonth() + 1, "day": yesterday.getDate()}
        },
        "dimensions": ["APP", "PLATFORM"],
        "metrics": ["ESTIMATED_EARNINGS"]
      }
    }),
    'muteHttpExceptions': true
  };

  var apiResponse = UrlFetchApp.fetch('https://admob.googleapis.com/v1/accounts/' + ADMOB_ACCOUNT_ID + '/mediationReport:generate', options);
  var responseContent = JSON.parse(apiResponse.getContentText());

  // 200以外の時、エラー
  if (apiResponse.getResponseCode() != 200) {
    Logger.log('Error fetching data: ' + apiResponse.getResponseCode());
    return null
  }

  var revenueReport = new RevenueReport();
  responseContent.forEach(item => {
      if (item.row) {
          let platform = item.row.dimensionValues.PLATFORM.value;
          let appName = item.row.dimensionValues.APP.displayLabel;
          let earnings = parseFloat(item.row.metricValues.ESTIMATED_EARNINGS.microsValue) / 1e6;
          let revenueData = new RevenueData(platform, appName, earnings);
          revenueReport.addEntry(revenueData);
      }
  });

  return revenueReport;
}
```

:::

やってることは下記です。

1. 事前にスプレッドシート経由でログインして、アクセストークンを取得できるようにします。
2. `dateRange`を指定して、昨日の収益を取得するようにします。
3. `dimensions`を指定して、取得する情報を指定します。他にも日付や AdUnit を指定できます。[詳しくはこちら](https://developers.google.com/admob/api/reference/rest/v1/accounts.mediationReport/generate#Dimension)。
4. `metrics`を指定して、収益を取得します。他にもクリック数やインプレッション数を指定できます。[詳しくはこちら](https://developers.google.com/admob/api/reference/rest/v1/accounts.mediationReport/generate#Metric)。
5. AdMob API を叩いて、`RevenueReport` に変換します

https://developers.google.com/admob/api/reference/rest/v1/accounts.mediationReport/generate

## 9. Slack に送信処理

slack に関連する処理をまとめた `slack.gs` ファイルを追加します。

:::details slack.gs

````js:slack.gs
var SLACK_WEB_HOOKS_URL = PropertiesService.getScriptProperties().getProperty('slack_web_hooks_url'); // Slack Appで登録したWeb Hooks URL

// #admob-revenueにRevenueReportの結果を送信する
function sendSlackForRevenueReport(revenueReport) {
  let message = formatReport(revenueReport);
  Logger.log(message);
  sendSlackForMessage(message)
}

// #admob-revenueにメッセージを送信する
function sendSlackForMessage(message) {
  const jsonData =
  {
    "text": message,
  };
  let options = {
    "method": "post",
    "contentType": "application/json",
    "payload": JSON.stringify(jsonData),
  };
  UrlFetchApp.fetch(SLACK_WEB_HOOKS_URL, options);
}

// RevenueReportをslackメッセージ用にフォーマットする
function formatReport(revenueReport) {
  revenueReport.entries.sort((a, b) => {
    // 収入で降順にソート
    if (a.earnings > b.earnings) return -1;
    if (a.earnings < b.earnings) return 1;
    // 収入が同じ場合はappNameで昇順にソート
    if (a.appName < b.appName) return -1;
    if (a.appName > b.appName) return 1;
    // appNameも同じ場合はplatformで昇順にソート
    if (a.platform < b.platform) return -1;
    if (a.platform > b.platform) return 1;
    return 0;
  });

  // テーブルのヘッダー
  let reportSummary = "```\n| Earnings    | Platform    | App Name   \n";
  reportSummary += "|-------------|-------------|------------|\n";

  // 各エントリのデータを追加
  revenueReport.entries.forEach(entry => {
      // 各セルを適切な幅に揃える
      // プラットフォーム名と収益を適切な幅に揃える
      let earnings = `${entry.earnings.toFixed(0)}円`.padStart(10, ' '); // 収益を10文字幅で右揃え
      let platform = entry.platform.padEnd(10, ' '); // プラットフォーム名を10文字幅で右揃え
      let appName = entry.appName.padEnd(24, ' '); // アプリ名を24文字幅で右揃え
      reportSummary += `| ${earnings} | ${platform} | ${appName} \n`;
  });

  // テーブルのフッターに合計収益を追加
  reportSummary += "|-------------|-------------|------------|\n";
  reportSummary += `| ${revenueReport.totalEarnings.toFixed(0).padStart(10, ' ')}円 |             |             |\n`;
  reportSummary += "```\n";

  return reportSummary;
}
````

:::

やってることは下記です。

1. `sendSlackForRevenueReport`で`RevenueReport`を受け取り、slack 用にメッセージをフォーマットします。
2. `sendSlackForMessage`でフォーマットしたメッセージを slack に送信します。

## 10. main 処理

実際の処理起点となる`main.gs`を追加します

:::details main.gs

```js:main.gs
function main() {
  // トークンがなければ認可処理を促す
  if (!admobAPIService.hasAccess()) {
    let message = "⛔️トークンの期限が切れました。スプレッドシートを開いて再認証してください。";
    sendSlackForMessage(message);
    return;
  }

  // 収入レポート取得
  var revenueReport = fetchYesterdayRevenueReport();
  sendSlackForRevenueReport(revenueReport);
}
```

:::

やっていることは下記です。

1. アクセストークンが無ければ、スプレッドシート経由で認証が必要であることのメッセージを Slack に送信します。
2. アクセストークンが存在すれば、AdMob API から`RevenueReport`を取得し、それを Slack に送信します。

ちなみにここまでやると下記のようなファイル構成になります。

![](https://storage.googleapis.com/zenn-user-upload/cf85202ae26f-20240512.png)
_main.gs_

## 11. 定期実行処理

最後に、左のメニューにある「トリガー」この`main`関数の処理を毎日実行するように設定します。
毎日１回実行すれば十分なので、下記のように設定すれば十分でしょう。

- 実行する関数を選択:main
- 実行するデプロイ：Head
- イベントのソースを選択：時間主導型
- 時間ベースのトリガーのタイプを選択：日付ベースのタイマー
- 時刻を選択：午前５〜６時

![](https://storage.googleapis.com/zenn-user-upload/24b8fbc3693b-20240512.png)

# 備考

思ったよりも長くなってしまいました。。
全ての情報を Slack に集約させるのはとても便利なので、ぜひやってみてください！
毎日収益を確認するは、結構モチベーション向上につながります 🙆‍♂️
