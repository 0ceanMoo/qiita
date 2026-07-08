---
title: fzf と ~/.ssh/config で SSH 接続先をインクリメンタルサーチする
tags:
  - SSH
  - fzf
  - shell
  - zsh
private: true
updated_at: '2026-07-08T14:03:51+09:00'
id: 79eb061f9e00e7336777
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## SSH 接続先を fzf で選びたい

SSH の接続先を覚えて正確に入力する代わりに、`sshi` を実行して `~/.ssh/config` の設定から選べるようにする。

たとえば、これまでこう打っていたものを、

```sh
ssh dev-server
```

こうできる。

```sh
sshi
```

![fzfでSSH接続先を選ぶ](../images/fzf-ssh-config/fzf_ssh_config.gif)

`sshi` を実行すると、`~/.ssh/config` に登録済みの `Host` が一覧表示される。あとは `fzf` で数文字入力して候補を絞り込み、Enter で接続する。

## セットアップ

### 必要なツール

必要なのは次の3つ。

| 必要なもの | 役割 |
|---|---|
| `~/.ssh/config` | SSH の接続先一覧として使う |
| `fzf` | 接続先をインクリメンタルサーチする |
| alias | `~/.ssh/config` から Host を取り出して SSH する |

この記事では `~/.ssh/config` の書き方は扱わない。すでに `ssh dev-server` のように Host 名で SSH できている前提。

### install

macOS なら Homebrew で入れる。

```sh
brew install fzf
```

```sh
fzf --version
```

### alias

`~/.zshrc` などに以下を追加する。

```sh
alias sshl="cat ~/.ssh/config | grep '^Host ' | sed -e 's/^Host //g'"
alias sshi="sshl | fzf | xargs -I{} sh -c 'ssh {} </dev/tty' ssh"
```

追加したら、設定を読み込み直す。

```sh
source ~/.zshrc
```

これで `sshi` が使える。

`sshl` を実行すると、`~/.ssh/config` の `Host` 一覧が表示される。

```sh
sshl
```

![sshlでSSH接続先を一覧表示する](../images/fzf-ssh-config/sshl.png)

## まとめ

`~/.ssh/config` に接続先をまとめているなら、`fzf` を組み合わせるだけで接続先を探しやすくできる。

`sshi` を実行して、表示された Host を絞り込んで選ぶだけなので、設定も使い方もシンプル。

### メリット

- Host 名を正確に覚えていなくても SSH 接続できる
- `~/.ssh/config` を開いて Host 名を確認する回数が減る
- 接続先が増えても `fzf` で絞り込める

### デメリット

- 接続先が少ないなら、普通に `ssh host-name` と打つほうが早い
- `fzf` を別途インストールする必要がある
- `~/.ssh/config` に登録していない接続先には使えない
- `ssh -i key.pem user@host` のような SSH コマンドのオプションを忘れやすくなる
