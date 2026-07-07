---
title: fzf と ~/.ssh/config で SSH 接続先をインクリメンタルサーチする
tags:
  - SSH
  - fzf
  - shell
  - zsh
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## やりたいこと

`~/.ssh/config` に書いてある `Host` を一覧化して、`fzf` でインクリメンタルサーチして、そのまま SSH 接続できるようにする。

サーバー名を覚えていなくても、うろ覚えの文字を打てば候補を絞り込める。

![fzfでSSH接続先を選ぶ](../images/fzf-ssh-config/fzf_ssh_config.gif)

## alias

使っている alias はこれ。

```sh
alias sshl="cat ~/.ssh/config |grep ^Host\  |sed -e 's/^Host\ //g'"
alias sshi="sshl | fzf | xargs -I{} sh -c 'ssh {} </dev/tty' ssh"
```

`sshi` を実行すると、`~/.ssh/config` の `Host` が `fzf` に流れて、選んだ接続先へ SSH できる。

```sh
sshi
```

## ~/.ssh/config の例

たとえば `~/.ssh/config` にこう書いておく。

```sshconfig
Host dev-server
  HostName 192.168.10.20
  User ubuntu
  IdentityFile ~/.ssh/dev-server.pem

Host bastion
  HostName bastion.example.com
  User ec2-user
  IdentityFile ~/.ssh/bastion.pem

Host raspberrypi
  HostName 192.168.68.74
  User pi
```

これで `sshi` を実行すると、以下のような候補から選べる。

```text
dev-server
bastion
raspberrypi
```

## sshl の中身

```sh
alias sshl="cat ~/.ssh/config |grep ^Host\  |sed -e 's/^Host\ //g'"
```

これは `~/.ssh/config` から `Host ` で始まる行だけを抜き出して、先頭の `Host ` を消している。

```sh
cat ~/.ssh/config
```

で設定ファイルを読み、

```sh
grep ^Host\ 
```

で `Host ` から始まる行だけを取り出し、

```sh
sed -e 's/^Host\ //g'
```

で `Host ` を削る。

結果として、SSH 接続先の一覧だけが出る。

```text
dev-server
bastion
raspberrypi
```

## sshi の中身

```sh
alias sshi="sshl | fzf | xargs -I{} sh -c 'ssh {} </dev/tty' ssh"
```

`sshl` の結果を `fzf` に渡して、選んだ1件を `ssh` に渡している。

流れとしてはこう。

```text
~/.ssh/config
  ↓
sshl で Host 一覧にする
  ↓
fzf で絞り込む
  ↓
選んだ Host に ssh する
```

`fzf` はインクリメンタルサーチできるので、接続先が増えても探しやすい。

## </dev/tty を付けている理由

```sh
ssh {} </dev/tty
```

`fzf` や `xargs` をパイプでつないだあとにそのまま `ssh` すると、標準入力の扱いが微妙になることがある。

`</dev/tty` を付けておくと、SSH の対話入力をちゃんと端末から受け取れる。

たとえば、パスフレーズ入力や接続後の操作が必要なときに困りにくい。

## 便利なところ

一番便利なのは、SSH の接続先名を覚えなくてよくなること。

```sh
ssh dev-server
```

のように Host 名を正確に打たなくても、`sshi` を実行して候補から選べばいい。

`~/.ssh/config` を接続先データベースのように使えるので、接続先が増えるほど便利になる。

## デメリット

便利な反面、`ssh` コマンドのオプションを忘れやすくなる。

たとえば鍵を指定する `-i`。

```sh
ssh -i ~/.ssh/dev-server.pem ubuntu@192.168.10.20
```

普段から `~/.ssh/config` に寄せていると、こういうオプションを直接打つ機会が減る。

ただ、これは悪いことばかりでもない。

`IdentityFile` や `User`、`HostName` を `~/.ssh/config` に書いておけば、毎回 `-i` やユーザー名を打たなくてよくなる。

```sshconfig
Host dev-server
  HostName 192.168.10.20
  User ubuntu
  IdentityFile ~/.ssh/dev-server.pem
```

設定を `~/.ssh/config` に寄せる運用だと割り切れば、むしろコマンドを短くできる。

## まとめ

`~/.ssh/config` と `fzf` を組み合わせると、SSH 接続先をインクリメンタルサーチして選べるようになる。

接続先が少ないうちはありがたみが小さいが、増えてくるとかなり楽になる。

```sh
alias sshl="cat ~/.ssh/config |grep ^Host\  |sed -e 's/^Host\ //g'"
alias sshi="sshl | fzf | xargs -I{} sh -c 'ssh {} </dev/tty' ssh"
```
