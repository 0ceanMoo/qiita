---
title: "Semgrepのルールセットとカスタムルールを実例で比較する"
tags:
  - Semgrep
  - SAST
  - SecurityTools
  - CI
  - DevOps
private: false
updated_at: '2026-08-19T14:43:40+09:00'
id: dfa5c951dc8ab6bb4332
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

コードに潜む脆弱性を検出するSASTツールとしてSemgrepを使っている。evalの使用やSQLの文字列結合のような危険な書き方がその代表例だ。普段は`--config auto`で回しているが、これが何をしているのか、OSS版とPro版で何が違うのか、`p/xxx`という名前のルールセットをどう選べばいいのか、なんとなくでしか分かっていなかった。

この記事では、実際にサンプルの脆弱性コードを用意してSemgrepを動かしながら、次を確認する。

1. **OSS / Code(SAST) / Supply Chain(SCA)は何が違うのか**
2. **`--config auto`はルールをどう読み込んでいるのか**
3. **ルールセット(`p/xxx`)ごとにどれくらい検出範囲が違うのか**
4. **誤検知はどう抑制するのか、その注意点**
5. **`auto`はどこまで検出できて、どこを見逃すのか**
6. **カスタムルールがなぜ必要になるのか、公開されているルールで十分なのか**
7. **`semgrep scan`と`semgrep ci`はどう使い分けるのか**

すべて実際に手元で実行した結果に基づいている。検証環境は以下の通り。

- Semgrep 1.172.0(Homebrewでインストール)
- 実行日: 2026年8月
- サンプルコードはリポジトリとして公開していないため、この記事に出てくるコード片は、再現に必要な範囲をその都度掲載する形にしている

Semgrepはレジストリ側のルールが随時更新されるため、ここに書かれているルール数や挙動は将来変わりうる。同じ結果になるとは限らないので、気になる箇所は手元で再検証してほしい。

## Semgrepとは

この記事に関係するSemgrepの構成要素として、次の3つがある。

- **Semgrep OSS**: 無料のCLIエンジン本体。Communityルールで、自分たちが書いたコードをパターンマッチで検査し、危険な書き方(`eval()`の使用、SQLの文字列結合など)を検出する。アカウント登録・ログイン不要
- **Semgrep Code(SAST)**: Communityルールに加えてProルールを利用でき、Pro Engineによるファイルをまたいだ解析(cross-file taint analysis)なども有効になる。利用にはログインが必要(2026年8月時点の公式ドキュメントによれば、10 contributors以下の場合は無料)
- **Semgrep Supply Chain(SCA)**: 依存ライブラリの脆弱性(CVE)を検査するもの。SASTとは検査対象が別(自分のコード vs 依存ライブラリ)。この記事では検証対象に含めていない

この記事で扱うのはSAST(OSS・Code)の方。以降、単に「ルール数」と書いている箇所は、この記事で比較したルール構成における数字であって、Pro Engineによる解析能力自体の違いは含んでいない。

ログイン有無で同じ5ファイルをスキャンすると、実行されるルール数はこう変わる。

| 状態 | レジストリ上の総ルール数 | 実行されたルール数 | 検出数 |
|---|---|---|---|
| ログアウト(無料のOSS) | 1074(Communityのみ) | 448 | 5 |
| ログイン済み(Code込み) | 2930(Community 1074 + Pro 1856) | 1524 | 5 |

今回の5ファイルでは、ログイン前後で検出数は変わらなかった。検出されたルールIDが同一かどうかまでは比較していない。

## `--config auto`のルール読み込み

`--config auto`を実行すると、`--debug`ログでこういう順序が見える。

```
1. git ls-remote でプロジェクトURLを取得しようとする
2. Downloading config from https://semgrep.dev/c/auto
3. Downloaded config from https://semgrep.dev/c/auto
4. get_targets request: ...
```

debugログでは、configの取得が先に記録され、対象ファイルの選択(`get_targets request`)はその後に記録されていた。実際、Dockerfileだけの時もPython単体の時も、ヘッダーに表示される総ルール数は常に同じ(ログイン時なら2930)で、変わるのは`Rules run`(実行されたルール数)の方だった。

```
# Pythonファイル1つだけのリポジトリでも
Scanning 1 file tracked by git with 2930 Code rules:
  python  1151  1   Pro rules  1856
  <multilang>  47  1   Community  1074
Rules run: 1198   ← python(1151) + multilang(47) だけが実行される
```

`Rules run`は、対象ファイルの言語などで絞り込んだ後に実際に適用されたルール数を指す方だ。ヘッダーの総ルール数とこの数字がどう関係しているのか(configの取得方法や、ダウンロード自体が言語を問わず行われるのかどうか)までは、debugログの範囲では確認できていない。

## ルールセット(`p/xxx`)の比較

`--config auto`の代わりに、名前の付いたルールセット(`p/xxx`)を個別に指定することもできる。前節の5ファイルにMD5利用のサンプルを1つ追加し、6ファイルにした上で比較した(`--config auto`のfindingは5件から6件に増えた)。

| Config | 実行ルール数 | 検出数 |
|---|---|---|
| `--config auto` | 1524 | 6 |
| `p/default` | 1524 | 6(autoと一致) |
| `p/security-audit` | 100 | 1 |
| `p/owasp-top-ten` | 1293 | 2 |
| `p/cwe-top-25` | 876 | 1 |
| `p/secrets` | 131 | 2 |
| `p/javascript` | 309 | 0 |
| `p/python` | 1066 | 1 |
| `p/dockerfile` | 7 | 1 |
| `p/jwt` | 12 | 0(JWT関連コードがないので妥当) |

今回の検証環境(2026年8月、ログイン状態)では、`--config auto`と`p/default`のfindingsもルールIDも完全に一致した。ただしレジストリ側の設定は更新されるため、常に同一という保証はない。今回のサンプルでは、`p/javascript`のような言語名のパックはevalの検出もシークレット検出も拾わなかった。`p/owasp-top-ten`や`p/cwe-top-25`も、evalやSQLインジェクション(SQLAlchemy)は拾わなかった。ルールセット内に該当ルールがそもそも存在しないのか、対象言語の絞り込みで除外されただけなのかは、ルールID一覧までは突き合わせていない。

:::note warn
これらのパック名が実際に「人気」「定番」と呼べるかどうかの利用数・採用率は確認できていない。`p/default`については、Semgrep公式が出発点として推奨する記述がある。
:::

### `auto`に足すと増えるルールもある

最初、`p/owasp-top-ten`や`p/secrets`を`auto`に追加しても検出数が変わらなかったので「足す意味は薄い」と考えていたが、これはサンプルコードの数が少なすぎて、追加分のルールがたまたま何も引っかからなかっただけだった。実際に`auto`とのルールID差分を取ると、`p/owasp-top-ten`には`auto`に含まれないルールが52個、`p/cwe-top-25`には21個あり、その大半はPython(Flask/Django/Pyramid)やJavaScript(Express)、Go向けで、今回使っている言語構成にも関わるルールだった。

例えば同じMD5利用の行に対して、`auto`と`p/owasp-top-ten`はそれぞれ別のルールで反応する。

```
# auto
python.lang.security.audit.md5-used-as-password.md5-used-as-password

# p/owasp-top-ten(autoには含まれないルール)
python.lang.security.insecure-hash-algorithms-md5.insecure-hash-algorithm-md5
```

どちらも同じ`hashlib.md5(...)`の行を指摘するが、ルールとしては別物で、後者は`auto`単体では出てこない。今回のサンプルコードがたまたま両方のルールに引っかかる書き方だったので気づけたが、サンプル次第では片方しか反応しないこともありうる。パックを組み合わせる価値があるかどうかは、サンプル数を絞った検証だけでは判断できない。

## 誤検知の抑制

### `// nosemgrep: rule-id`

1行単位でルールを抑制できる。

```js
// nosemgrep: generic.secrets.security.detected-stripe-api-key.detected-stripe-api-key
const TEST_STRIPE_KEY = "sk_live_EXAMPLE_NOT_A_REAL_KEY";
```

### `.semgrepignore`

ファイル・ディレクトリ単位で除外できる。ただし2つ注意点がある。

**注意点1: ディレクトリ経由でスキャンした時しか効かない。** ファイルパスを直接指定してスキャンすると、`.semgrepignore`は無視される。

```bash
# .semgrepignoreに書いてあっても、ファイルを直接指定すると効かない
semgrep scan --config auto samples/js/tests/payment.test.js
```

lint-stagedやpre-commitで差分ファイルのパスを直接semgrepに渡す運用だと、除外したつもりのファイルが実は除外されない、という事故につながりやすい。

**注意点2: Semgrep 1.172.0では、`test/`・`tests/`・`testsuite/`などの名前のディレクトリがデフォルトで除外された。** `.semgrepignore`の中身に関係なく、これらの名前のディレクトリが除外される挙動を確認した。

```bash
$ semgrep scan --config auto samples/ --verbose
...
  Skipped by .semgrepignore:
  - https://semgrep.dev/docs/ignoring-files-folders-code/#understand-semgrep-defaults
   • samples/js/tests/payment.test.js
   • samples/python/tests/test_auth.py
```

`.semgrepignore`の中身を空(改行のみ)にしても、この2ファイルは除外され続けた。公式ドキュメントによれば、デフォルトの除外パターンには他に`node_modules/`・`build/`・`dist/`・`.git`なども含まれる。自分の`.semgrepignore`の効果を検証したいなら、`test`という名前のディレクトリは避けた方がいい。

## `auto`が拾えない範囲: 網羅性の限界

「autoは一般的な脆弱性を網羅している」と思いがちだが、実際に10カテゴリの脆弱性を、実在するライブラリを使ったサンプルコードで検証したところ、多くは検出された一方で未検出のものもあった。結果はこちら。

| カテゴリ | 言語/ライブラリ | 結果 |
|---|---|---|
| SQLインジェクション | JS(mysql2) / Python(psycopg2) / Go(database/sql) | ✅ 全部検出 |
| コマンドインジェクション | JS(child_process) / Python(subprocess) / Go(os/exec) | ✅ 全部検出 |
| XSS | JS/React(dangerouslySetInnerHTML) | ✅ 検出 |
| ハードコードされたシークレット | JS(Stripeキー形式) | ✅ 検出 |
| 安全でないデシリアライズ | Python(pickle) | ✅ 検出 |
| 安全でないYAML読み込み | Python(yaml.load) | ✅ 検出 |
| 脆弱な暗号(MD5) | Python(hashlib) / Go(crypto/md5) | ✅ 検出 |
| パストラバーサル | JS(fs+path) | ✅ 検出 |
| **パストラバーサル** | **Python(os.path.join+open)** | ❌ 未検出 |
| **SSRF** | **JS(axios.get(userUrl))** | ❌ 未検出 |
| Dockerfile: rootユーザー | - | ✅ 検出 |
| **Dockerfile: `latest`タグ** | - | ❌ 未検出 |

今回用意したパストラバーサルのサンプルでは、JavaScript版は検出され、Python版は検出されなかった。同じカテゴリでも、言語やコードの書き方によって結果が異なることがある。

### ライブラリ固有ルールの検出条件

`db.query()`という架空のオブジェクトにSQLインジェクションのコードを書いても、`--config auto`は**0件**だった。

```js
// 検出されない(架空のdbオブジェクト)
function getUserByEmail(db, email) {
  return db.query(`SELECT * FROM users WHERE email = '${email}'`);
}
```

同じ形のコードでも、実在のライブラリ(`mysql2`)をimportして書き直すと検出された。

```js
// 検出される(実在のmysql2)
const mysql = require("mysql2");
const connection = mysql.createConnection({ host: "localhost" });

function getUserByEmail(email) {
  return connection.query(`SELECT * FROM users WHERE email = '${email}'`);
}
```

```
javascript.lang.security.audit.sqli.node-mysql-sqli.node-mysql-sqli
Detected a `mysql2` SQL statement that comes from a function argument...
```

今回反応した`node-mysql-sqli`ルールは、`mysql2`という既知のライブラリのimportと、そのAPI呼び出しを条件にしていた。このようにライブラリ固有のAPIを対象にしたルールでは、自社製の薄いDBラッパーのような、レジストリが認識していない抽象化を挟んでいると検出できない場合がある。Semgrepには構文パターンだけを見るジェネリックなルールもあるため、すべてのルールがライブラリのimportを条件にしているわけではない。

### 見つけたルールが想定するコードの形式

「autoが見逃すなら、個人・組織が公開しているルールで補えないか」と考え、実際に探して試した。

**SSRF**: [imfht/my-semgrep-rules](https://github.com/imfht/my-semgrep-rules)にaxios向けのSSRF検出ルールがあった。試したが、今回のサンプル(汎用的な関数)では検出されなかった。ルールの中身を見ると、**Express.jsの`function(req, res)`という特定の関数シグネチャの中で`req.query.xxx`のような形でURLを受け取るケースだけ**を想定していた。

```js
// req.query.url を経由するExpress風の書き方に直すと検出された
app.get("/fetch", function (req, res) {
  axios.get(req.query.url).then((response) => {
    res.send(response.data);
  });
});
```

**Dockerfileの`latest`タグ**: Semgrep RegistryにCommunityルール`dockerfile.best-practice.missing-image-version.missing-image-version`が実在した。だがパターンを見ると、

```yaml
pattern-either: [pattern: "FROM $IMAGE"]
pattern-not: "FROM $IMAGE:$VERSION"   # ← "latest"もここに含まれてしまう
pattern-not: "FROM $IMAGE@$DIGEST"
pattern-not: "FROM $IMAGE:$VERSION@$DIGEST"
pattern-not: "FROM scratch"   # マルチステージビルドの誤検知対応で後から追加された除外
```

このルールはタグの有無を検査するもので、タグの値までは評価していない。`:latest`もタグの一種として`$IMAGE:$VERSION`にマッチするため、検出対象から外れる。「タグが完全にない場合」だけを検出する作りになっている。

**「そのバグ用のルールが存在するかどうか」と「実際に自分のコードで検出できるかどうか」は別問題。** 今回検証したSSRFルールもDockerfileルールも、それぞれが想定する特定のコードの形・条件にピンポイントで作られていた。使う前に、そのルールの`pattern`や除外条件を確認する必要がある。taint modeのルールであれば、source・sink・sanitizerの条件も合わせて確認する。

## カスタムルールの書き方

公開ルールでは拾えない、プロジェクト固有の規約を検出したい場合はカスタムルールを書く。例として「`db.query()`や`db.execute()`を直接呼ばず、`repository/`配下のラッパー経由でアクセスする」というチーム規約を検出するルール。

```yaml
rules:
  - id: no-direct-db-query
    languages: [javascript, typescript]
    severity: WARNING
    message: >
      db.query() / db.execute() を直接呼び出さず、repository/ 配下のラッパー経由でアクセスしてください。
    pattern-either:
      - pattern: db.query(...)
      - pattern: db.execute(...)
    paths:
      exclude:
        - "**/repository/**"
```

- `pattern-either`: 複数のパターンのうち、どれか1つでも一致すれば検出する
- `...`: 引数は何でもいいというワイルドカード
- `paths.exclude`: **このルールの適用対象**からファイル・ディレクトリを除外する(スキャン全体から除外するわけではなく、他のルールは同じファイルを引き続き検査できる)

:::note warn
`paths.exclude`に`"repository/**"`と書くと、将来的に`"/repository/**"`(先頭に固定)として解釈されるようになるという非推奨警告が出た。ディレクトリ名がどこにあっても除外したい場合は`"**/repository/**"`と書く必要がある。
:::

このルールは、コードが安全かどうかを判定しているわけではない。`repository/`配下の正当なラッパー実装でも同じ`db.query(...)`を呼んでいるが、`paths.exclude`でディレクトリごと対象から外しているから検出されないだけ。チェックしているのは「規約を守っているか」だけで、実際に危険なコードかどうかは見ていない。

なお`pattern: db.query(...)`は、変数名`db`に対する`query()`呼び出しだけにマッチする。`database.query()`や`ctx.db.query()`のような別の書き方は検出しない。任意の変数名を対象にしたい場合は`$DB.query(...)`のようにmetavariableを使う書き方もあるが、その分マッチする範囲が広がり、誤検知も増えうる。

### `auto`とカスタムルールの併用

`--config`は複数回指定できるので、2回実行する必要はない。

```bash
semgrep scan --config auto --config rules/no-direct-db-query.yaml samples/
```

```
Ran 369 rules on 2 files: 2 findings.
  Origin      Rules
  Pro rules    1856
  Community    1074
  Custom          1
```

この出力では、`Origin`欄に2931ルール(Pro rules 1856、Community 1074、Custom 1)の内訳が表示される一方、実際に実行されたルール数は`Ran 369 rules`(対象2ファイルに対して368本 + カスタムルール1本)だった。両者の正確な集計条件(言語による絞り込みなのか、他の条件も含むのか)までは確認していない。

この記事では、カスタムルールをリポジトリ内の専用ディレクトリ(`.semgrep/`など)にまとめて置き、

```bash
semgrep scan --config auto --config .semgrep/
```

のように、レジストリのルールセットと自前のディレクトリを両方`--config`で渡す。

### カスタムルールを書く前に

いきなり書き始める前に、[Semgrep Registry](https://semgrep.dev/explore)で同じニーズのルールが既にないか検索する。個人・組織が公開している既存ルール集もあり、[semgrep/semgrep-rules](https://github.com/semgrep/semgrep-rules)(公式のCommunity Editionルール本体)以外に、[trailofbits/semgrep-rules](https://github.com/trailofbits/semgrep-rules)(Trail of Bitsのセキュリティ監査由来のルール、`p/trailofbits`としてレジストリからも使える)や、複数の公開ルールをまとめた[j3ssie/curated-semgrep-rules](https://github.com/j3ssie/curated-semgrep-rules)のようなリポジトリもある。

見つけただけで満足せず、この2つも実際にクローンして今回のサンプルコードに対して動かしてみた。

`p/trailofbits`は0件だった。リポジトリの内容を見た限りC/C++やブロックチェーン寄りのルールが中心のようだが、実行されたルールの言語別内訳までは確認しておらず、0件だった理由を特定できていない。

`j3ssie/curated-semgrep-rules`は21ファイルに対して138ルールが実行され、15件検出した。ただし内訳を見ると、11件が同じ1つのルール(コード中の`BUG`や`TODO`のような疑わしいコメントを検出するだけのもので、脆弱性検出ではない)、1件はマジックナンバーの指摘で、セキュリティに関係する新しい検出は`kondukto`由来のDockerfile向けルール1件だけだった。加えて、リポジトリをそのまま`--config`に渡すと、ルールファイルではないCIの設定ファイル(`.github/workflows/*.yml`など)が混じっていてパースエラーになり、そのままでは使えなかった。

名前が知られている、複数のルールをまとめている、という触れ込みだけでは中身の質は分からず、見つけたルールは自分のプロジェクトの実際のコードに対して動かして確認する必要がある。

[Trail of Bitsの記事](https://blog.trailofbits.com/2024/01/12/how-to-introduce-semgrep-to-your-organization/)では、カスタムルールの用途として6つが挙げられている。セキュリティ脆弱性、**ベストプラクティス/社内規約の強制**、パフォーマンス最適化、ビジネス要件の検証、非推奨コードの検出、設定ミスの検出。`db.query`の例は、このうち「ベストプラクティス/社内規約の強制」にあたる。

### 既存のルールを土台にする

前に、自社製の薄いDBラッパーは`--config auto`では検出できないという例を扱った。`node-mysql-sqli`ルールが、importが`mysql`か`mysql2`であることを条件にしていたためだった。

このルールをゼロから書き直す必要はない。[semgrep/semgrep-rules](https://github.com/semgrep/semgrep-rules)をクローンすると、この`node-mysql-sqli`ルールの実体(`javascript/lang/security/audit/sqli/node-mysql-sqli.yaml`)が手に入る。中身を見ると、`mode: taint`で書かれた本格的なルールで、「関数の引数から来た値(source)」が「`.query()`/`.execute()`(sink)」に渡り、「`parseInt()`のような処理(sanitizer)」を経ていない場合に検出する、という構成になっている。

このうち、importを判定している箇所を1行変えるだけで、自社製ラッパーにも対応できる。

```yaml
# 変更前
    - metavariable-regex:
        metavariable: $IMPORT
        regex: (mysql|mysql2)

# 変更後(架空の @myorg/db-client に対応させた例)
    - metavariable-regex:
        metavariable: $IMPORT
        regex: (mysql|mysql2|@myorg/db-client)
```

`@myorg/db-client`という架空のライブラリをimportして`db.query()`を呼ぶコードで試すと、変更前は0件、変更後は検出できた。

```js
const db = require("@myorg/db-client");

function getUserByEmail(email) {
  return db.query(`SELECT * FROM users WHERE email = '${email}'`);
}
```

```
rules.custom-db-sqli
Detected a `@myorg/db-client` SQL statement that comes from a function argument...
    6┆ return db.query(`SELECT * FROM users WHERE email = '${email}'`);
```

source・sink・sanitizerを含むtaint modeのルールを一から書くのは、`pattern-either`だけで済む`no-direct-db-query`のようなルールに比べて手間がかかる。近い既存ルールを見つけて、条件を1〜2箇所だけ書き換える方が現実的なことが多い。

## `semgrep scan`と`semgrep ci`の違い

ここまでずっと`semgrep scan --config auto`を使ってきたが、公式のクイックスタートは`semgrep login` → `semgrep ci`という別の手順を案内している。

```
semgrep ci - the recommended way to run semgrep in CI

In pull_request/merge_request (PR/MR) contexts, `semgrep ci` will only
report findings that were introduced by the PR/MR.

When logged in, `semgrep ci` runs rules configured on Semgrep App and
sends findings to your findings dashboard.

Only displays findings that were marked as blocking.
```

| | `semgrep scan` | `semgrep ci` |
|---|---|---|
| 主な用途 | ローカル実行、ルール開発、任意のスキャン | CI上でのポリシー適用 |
| ルール設定 | 主に`--config`で指定 | Semgrep Appの設定、または実行条件に応じたconfig |
| 差分の扱い | 指定した対象のfindingを出力 | PR/MR情報とbaselineを利用できる場合、新規findingを選別 |
| 出力先 | ターミナル、JSON、SARIFなど | CI出力、ログイン時はSemgrep Appへの送信も可能 |

`semgrep ci`は、ログインしてSemgrep App側でルールを一元管理する運用と相性がいい。アカウント登録・ログイン不要のOSS CLIとして使いたいだけなら、この記事で使ってきた`semgrep scan --config auto`の方が意図に合う。

## 結局どう使うのがいいか

ここまでの内容を踏まえて、実際の構成としてはこうなる。

```bash
semgrep scan --config auto --config .semgrep/
```

- **ベースは`--config auto`にする。** `p/xxx`を自分で選んで組み合わせるより、対象言語に応じて広く選んでくれる`auto`に任せた方が、今回の検証では手間の割に抜け漏れが少なかった
- **ローカルとCIでログイン状態を揃える。** `auto`はログインの有無で検出されるルール数が変わる。片方だけログインしていると、同じ`--config auto`でも環境によって結果が変わりうる
- **自分のプロジェクト固有の規約は、カスタムルールを`.semgrep/`のような専用ディレクトリにまとめて`--config`に追加する。** 今回試した`db.query()`の直接呼び出し禁止のような、公開ルールが存在しようがない社内規約はここでしか拾えない
- **自社製のライブラリ・ラッパーが絡む検出は、ゼロから書くより[semgrep/semgrep-rules](https://github.com/semgrep/semgrep-rules)から近いルールを探して改造する方が早い。** 今回は`mysql2`向けのtaint modeルールの`metavariable-regex`を1行変えるだけで、架空の自社製DBラッパーにも対応できた
- **外部で公開されているルール集を追加する場合は、事前に自分のプロジェクトのコードに対して実際に動かして確認する。** 複数のルールをまとめているという触れ込みだけでは中身の質は分からず、今回試した中には有用な検出が1件しかなかったものもあった
- **`auto`が拾わなかった実例(パストラバーサル、SSRF、Dockerfileの`latest`タグなど)は、自分たちの言語・フレームワークでも同じように試して、実際に検出されるか確認しておく価値がある。** 見逃しに気づいた場合、そこを埋めるカスタムルールを書くかどうかは、実際の被害の大きさと運用コストを見て判断する
- **CIへの組み込みは`semgrep scan`と`semgrep ci`のどちらでもいい。** アカウント登録・ログイン不要のCLI運用で完結させたいなら`semgrep scan --config auto`、Semgrep Appのダッシュボードで指摘を一元管理したいなら`semgrep ci`

## まとめ

各セクションで確認した内容を短くまとめると、次の通り。

- **今回の検証環境では`--config auto`と`p/default`の結果が一致した。** ただしレジストリ側の設定は更新されるため、常に同一とは限らない。`p/owasp-top-ten`などのパックには`auto`に含まれないルールも一定数あり、組み合わせる価値があるかはサンプル数を絞った検証だけでは判断できない
- **`auto`は今回試した範囲では多くを検出したが、完全ではなかった。** 今回確認したようなライブラリ固有のルールでは、自社製ラッパー経由のコードが見逃されることがある
- **「ルールが存在する」と「自分のコードで検出できる」は別問題。** 公開されているルールも、それぞれが想定する特定のコードの形・条件に絞って作られている
- **`.semgrepignore`には2つ注意点がある。** ファイル直接指定だと効かないこと、`test/`のような名前はデフォルトで除外されること
- **カスタムルールは「autoの代わり」ではなく「補完」。** `auto`と併用して、プロジェクト固有の規約を検査するのに使う
- **カスタムルールはゼロから書く必要はない。** [semgrep/semgrep-rules](https://github.com/semgrep/semgrep-rules)から近いルールを見つけて、条件を書き換える方が早いことが多い
- **`semgrep scan`と`semgrep ci`は、主な用途と既定のワークフローが異なる。** アカウント不要のCLI運用なら前者、ダッシュボードで一元管理したいなら後者
