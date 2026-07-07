---
title: テスト記事
tags:
  - テスト
private: true
updated_at: '2026-07-08T02:23:43+09:00'
id: 3f3b770e00b87c88645b
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## Zenn記法変換テスト

ZennからQiitaへの記法変換が正しく動作するか確認するためのテスト記事です。

### :::message の変換確認

以下はZennの `:::message` 記法です。Qiitaでは `:::note` に変換されるはずです。

:::note
これは通常のメッセージです。Qiitaでは note スタイルで表示されるはずです。
:::

### :::message alert の変換確認

以下はZennの `:::message alert` 記法です。Qiitaでは `:::note warn` に変換されるはずです。

:::note warn
これは警告メッセージです。Qiitaでは warn スタイルで表示されるはずです。
:::

### 通常のコンテンツ

変換処理が通常のテキストに影響しないことも確認します。

```bash
echo "hello world"
```

## publish 更新確認

この行は `npx qiita publish test-hello-world` で既存記事が更新されるか確認するための追記です。

## 画像表示テスト

![テスト画像](../images/test-hello-world/profile.jpeg)
