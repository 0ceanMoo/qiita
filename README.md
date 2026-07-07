# Qiita 記事管理

このディレクトリは、Qiita CLI で Qiita の記事を管理・プレビュー・投稿するための場所です。
Zenn の構成は考えず、Qiita CLI の仕様に合わせて `public/` を記事置き場として使います。

## ディレクトリ構成

```text
Qiita/
├── public/
│   ├── tailscale-wol-rdp.md
│   └── test-hello-world.md
├── public/.remote/
├── images/
├── qiita.config.json
├── package.json
└── README.md
```

- `public/`: Qiita CLI が扱う記事ファイル置き場
- `public/.remote/`: Qiita CLI の同期用ファイル。通常は手で編集しない
- `images/`: 記事で使う画像置き場
- `qiita.config.json`: Qiita Preview の設定

## 管理中の記事

| ファイル | 記事 |
| --- | --- |
| `public/tailscale-wol-rdp.md` | 外出先から自宅のスリープ中Windowsをリモート操作する |
| `public/test-hello-world.md` | テスト記事 |

既存記事の更新は、各 Markdown の frontmatter にある `id` で Qiita 上の記事と紐づきます。
`id` を消すと別記事として扱われる可能性があるため、既存記事では削除しないでください。

## 初回セットアップ

このディレクトリで作業します。

```bash
cd /Users/wataru/WorkSpace/01_Hot/Blog/Qiita
```

依存関係を入れ直す場合:

```bash
npm install
```

Qiita CLI のバージョン確認:

```bash
npx qiita version
```

Qiita CLI にログイン:

```bash
npx qiita login
```

ログインには Qiita のアクセストークンが必要です。
トークンには `read_qiita` と `write_qiita` を付けます。

https://qiita.com/settings/tokens/new

## プレビュー

```bash
npx qiita preview
```

通常は以下で確認できます。

```text
http://localhost:8888
```

表示対象は `public/` 配下の Markdown ファイルです。

## 投稿・更新

通常はローカルから直接 publish せず、`main` に push して GitHub Actions から publish します。
Actions は `public/**`、`images/**`、`.github/workflows/publish.yml` が変わったときだけ起動します。

```bash
git add public images
git commit -m "記事を更新"
git push
```

Actions 内では `npx qiita publish --all` を実行します。
Qiita CLI は Qiita 上の記事との差分を確認し、差分がある記事と新規記事だけを publish 対象にします。

手元から確認目的で publish する場合:

1記事だけ投稿・更新する場合:

```bash
npx qiita publish tailscale-wol-rdp
```

テスト記事を投稿・更新する場合:

```bash
npx qiita publish test-hello-world
```

全記事をまとめて反映する場合:

```bash
npx qiita publish --all
```

強制的に手元の内容を反映する場合:

```bash
npx qiita publish tailscale-wol-rdp --force
```

## 画像

ローカルでは `images/` に画像を置き、記事から相対パスで参照します。

```text
Qiita/
├── public/
│   └── tailscale-wol-rdp.md
└── images/
    └── tailscale-wol-rdp/
        └── rdp-connected.png
```

記事内では以下のように書きます。

```md
![RDP接続画面](../images/tailscale-wol-rdp/rdp-connected.png)
```

GitHub Actions 内で、投稿用の記事だけ GitHub raw URL に変換します。
ローカルの `public/*.md` は相対パスのまま維持されます。

## Qiita 側の記事を取り込む

Qiita 上で編集した内容をローカルに取り込む場合:

```bash
npx qiita pull
```

強制的に Qiita 側の内容で上書きする場合:

```bash
npx qiita pull --force
```

## frontmatter

Qiita CLI の記事は、先頭に以下のような frontmatter を持ちます。

```yaml
---
title: 記事タイトル
tags:
  - Qiita
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
```

既存記事では `id` が `null` ではなく、Qiita の記事IDになっています。

## 注意点

- Qiita CLI から記事の削除はできません。削除は Qiita の画面から行います。
- `public/` から Markdown を消しても Qiita 上の記事は削除されません。
- `public/.remote/` は Qiita CLI の管理用なので、通常は編集しません。
- 画像を使う場合、Qiita 上で表示できるパス・URLになっているか確認してください。

## 参考

- Qiita CLI: https://github.com/increments/qiita-cli
