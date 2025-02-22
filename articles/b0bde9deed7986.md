---
title: "特定の Web サイト を ブロックする Chrome 拡張の作り方"
emoji: "🧱"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Chrome"]
published: true
---

# 概要

特定の Web サイト を ブロックする Chrome 拡張を作った時の備忘録です。
同じような拡張機能は存在しますが、勉強がてら作りました。
ストアには公開せず、デベロッパーモードを前提に作成しています。

# 方法

## 1. コード準備

下記のような構成のディレクトリを用意します。
ブロックする url を background.js 内の`blockedUrls`に指定してください。

```text
my-extension/
│── manifest.json
│── background.js
```

:::details manifest.json

```json
{
  "manifest_version": 3,
  "name": "WebSite Blocker",
  "version": "1.0",
  "description": "指定したURLのページをブロックする",
  "permissions": ["declarativeNetRequest", "browsingData"],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background.js"
  }
}
```

:::

:::details background.js

```js
async function updateBlockedUrls() {
  // ここにブロックするurlを指定
  const blockedUrls = ["youtube.com"];
  const rules = blockedUrls.map((url, index) => ({
    id: index + 1,
    priority: 1,
    action: { type: "block" },
    condition: {
      urlFilter: "||" + url + "/",
      resourceTypes: ["main_frame", "sub_frame"],
    },
  }));

  const oldRules = await chrome.declarativeNetRequest.getDynamicRules();
  const oldRuleIds = oldRules.map((rule) => rule.id);

  chrome.declarativeNetRequest.updateDynamicRules(
    {
      addRules: rules,
      removeRuleIds: oldRuleIds,
    },
    () => {
      const origins = blockedUrls.map((url) => "https://" + url);
      if (origins.length == 0) {
        return;
      }
      // サービスワーカーのキャッシュを削除
      // 削除しないとキャッシュが残っているため、ページを読み込めてしまう。
      chrome.browsingData.remove(
        {
          origins: origins,
        },
        {
          serviceWorkers: true,
        }
      );
    }
  );
}

// 拡張機能の起動時にルールを適用
chrome.runtime.onInstalled.addListener(updateBlockedUrls);
chrome.runtime.onStartup.addListener(updateBlockedUrls);
```

:::

## 2. 拡張機能読み込み

![](https://storage.googleapis.com/zenn-user-upload/9fdf9487a972-20250223.png)

1. chrome://extensions/ にアクセス
2. 「Developer mode」を ON
3. 「Load unpacked」 をクリック
4. 作成したフォルダ（my-extension/）を選択

# 感想

デベロッパーモードで作る分には、簡単に Chrome の拡張機能を作れることがわかりました。
また、ブロックするサイトを Chrome 上ではなく手元のコードから変更しないといけないため、気軽に変更できないのがお気に入りポイントです 🙌
