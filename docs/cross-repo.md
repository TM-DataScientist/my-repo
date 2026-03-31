# `cross-repo.yml` 詳細仕様

対象ファイル: `.github/workflows/cross-repo.yml`

## 概要

`cross-repo.yml` は、`push` を契機に実行される GitHub Actions ワークフローです。

このワークフローの目的は、GitHub App の資格情報を使って同一オーナー配下の別リポジトリへアクセスし、そのリポジトリを checkout してファイルを読むことです。

このファイルでは、`another-repo` という別リポジトリ向けの GitHub App トークンを生成し、そのトークンで対象リポジトリを checkout したうえで `README.md` を表示します。

## 何をするワークフローか

このワークフローの処理は 1 ジョブだけです。

1. `push` でワークフローが起動する
2. `TARGET_REPO` に指定した別リポジトリ名を参照する
3. GitHub App の `APP_ID` と `PRIVATE_KEY` から一時トークンを生成する
4. そのトークンを使って別リポジトリを checkout する
5. checkout した別リポジトリの `README.md` を表示する

つまり、このワークフローは「GitHub App 認証でクロスリポジトリアクセスする最小サンプル」です。

## 起動条件

```yaml
on: push
```

この設定により、このリポジトリで `push` が発生するたびにワークフローが起動します。

起動対象は現在のリポジトリですが、ワークフロー本体がアクセスするのは `TARGET_REPO` で指定した別リポジトリです。

## top-level `env`

```yaml
env:
  TARGET_REPO: another-repo
```

この `env` はワークフロー全体で参照される環境変数です。

この値は次の 3 箇所で使われています。

1. GitHub App トークンの対象リポジトリ指定
2. `actions/checkout` の checkout 対象指定
3. checkout 後に読むファイルパス指定

### `TARGET_REPO` の意味

`TARGET_REPO` には、GitHub App がインストールされており、アクセスを許可されているリポジトリ名を入れる前提です。

このワークフローではオーナー名は別で組み立てているため、ここにはリポジトリ名のみを入れています。

## ジョブ構成

```yaml
jobs:
  checkout:
    runs-on: ubuntu-latest
    steps:
      - id: create
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.PRIVATE_KEY }}
          repositories: ${{ env.TARGET_REPO }}
      - uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/${{ env.TARGET_REPO }}
          path: ${{ env.TARGET_REPO }}
          token: ${{ steps.create.outputs.token }}
      - run: cat "${TARGET_REPO}/README.md"
```

このワークフローには `checkout` という 1 つのジョブだけがあります。

## Job: `checkout`

### 役割

`checkout` ジョブは、GitHub App トークンを作り、そのトークンで別リポジトリを checkout し、中のファイルを読むジョブです。

### 実行環境

```yaml
runs-on: ubuntu-latest
```

このジョブは Ubuntu ランナー上で実行されます。

この選択には次の意味があります。

1. `cat` コマンドがそのまま使える
2. `actions/checkout@v4` の利用に一般的な環境が用意される
3. GitHub App トークン生成アクションを標準ランナー上で実行できる

## 各 step の詳細

### 1. GitHub App トークン生成 step

```yaml
- id: create
  uses: actions/create-github-app-token@v1
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.PRIVATE_KEY }}
    repositories: ${{ env.TARGET_REPO }}
```

この step は、GitHub App の資格情報を使ってインストールトークンを生成します。

### 入力値

| 項目 | 値 | 意味 |
| --- | --- | --- |
| `app-id` | `${{ secrets.APP_ID }}` | GitHub App の App ID |
| `private-key` | `${{ secrets.PRIVATE_KEY }}` | GitHub App の秘密鍵 |
| `repositories` | `${{ env.TARGET_REPO }}` | アクセス対象として絞り込むリポジトリ |

### この step がやっていること

1. `APP_ID` と `PRIVATE_KEY` から GitHub App として認証する
2. `TARGET_REPO` を対象にしたインストールトークンを発行する
3. 発行したトークンを step output として返す

### `id: create` の意味

この `id` によって、後続 step から `steps.create.outputs.token` を参照できます。

このワークフローでは、その値を `actions/checkout` の `token` に渡しています。

## GitHub App トークンとは何か

ここで使っているトークンは `GITHUB_TOKEN` ではなく、GitHub App から発行したインストールトークンです。

この方式を使う理由は、現在のリポジトリの自動トークンだけでは別リポジトリへアクセスできない、またはアクセス範囲を GitHub App 側で明示的に制御したいケースがあるためです。

特にクロスリポジトリアクセスでは、GitHub App を使うと次の点が明確になります。

1. どのリポジトリにアクセスできるか
2. どの権限を持つか
3. その権限が App インストール設定に基づくこと

## 2. 別リポジトリ checkout step

```yaml
- uses: actions/checkout@v4
  with:
    repository: ${{ github.repository_owner }}/${{ env.TARGET_REPO }}
    path: ${{ env.TARGET_REPO }}
    token: ${{ steps.create.outputs.token }}
```

この step は、生成した GitHub App トークンを使って別リポジトリを checkout します。

### 各パラメータの意味

| 項目 | 値 | 意味 |
| --- | --- | --- |
| `repository` | `${{ github.repository_owner }}/${{ env.TARGET_REPO }}` | checkout 対象のリポジトリ |
| `path` | `${{ env.TARGET_REPO }}` | ランナー上で展開するディレクトリ名 |
| `token` | `${{ steps.create.outputs.token }}` | GitHub App から発行したアクセストークン |

### `repository` の組み立て

`repository` は次の形になります。

```text
<現在のリポジトリオーナー>/another-repo
```

つまり、このワークフローは「現在のリポジトリと同じ owner 配下にある別リポジトリ」を前提にしています。

### `path` の意味

```yaml
path: ${{ env.TARGET_REPO }}
```

この指定により、checkout した別リポジトリはランナー上で `another-repo/` というディレクトリに展開されます。

そのため、後続 step では `another-repo/README.md` のようなパスでファイルにアクセスできます。

### この step がやっていること

1. checkout 対象リポジトリを `<owner>/<repo>` 形式で決定する
2. GitHub App トークンでそのリポジトリへ認証する
3. 対象リポジトリの内容をランナー上の `TARGET_REPO` ディレクトリへ取得する

## 3. README 表示 step

```yaml
- run: cat "${TARGET_REPO}/README.md"
```

この step は、checkout した別リポジトリ内の `README.md` を標準出力へ表示します。

### 何が行われるか

1. `TARGET_REPO` 環境変数をディレクトリ名として使う
2. `${TARGET_REPO}/README.md` を開く
3. その内容を Actions ログへ出力する

### 実行結果のイメージ

`TARGET_REPO=another-repo` なら、この step は実質的に次のファイルを読みます。

```text
another-repo/README.md
```

そのファイルの内容がジョブログに表示されます。

## データの流れ

このワークフロー全体の値の流れは次の通りです。

1. `push` でワークフローが起動する
2. `TARGET_REPO` から対象リポジトリ名を取得する
3. GitHub App の `APP_ID` と `PRIVATE_KEY` で一時トークンを生成する
4. 生成したトークンを `steps.create.outputs.token` として受け取る
5. そのトークンを使って `<owner>/<TARGET_REPO>` を checkout する
6. checkout 済みディレクトリ配下の `README.md` を読む
7. README 内容をログへ表示する

## 成功時の挙動

次の条件をすべて満たすと、ジョブは成功します。

1. `APP_ID` と `PRIVATE_KEY` が正しい
2. GitHub App が `TARGET_REPO` にインストールされている
3. GitHub App が対象リポジトリを読む権限を持っている
4. 対象リポジトリに `README.md` が存在する

この場合、別リポジトリの README 内容がログに出力されてワークフローは成功します。

## 失敗時の挙動

次のような場合、ワークフローは失敗します。

1. `APP_ID` または `PRIVATE_KEY` が不正
2. GitHub App が対象リポジトリにインストールされていない
3. `repositories` で指定したリポジトリ名が実在しない
4. GitHub App に対象リポジトリを読む権限がない
5. checkout 後に `README.md` が存在しない

どこかの step が失敗すると、後続 step は通常実行されず、ジョブ全体が失敗として終わります。

## このワークフローが示しているポイント

このファイルのポイントは次の 4 つです。

1. GitHub App を使ってクロスリポジトリアクセスできる
2. `actions/create-github-app-token@v1` で一時トークンを生成できる
3. `actions/checkout@v4` の `repository` と `token` を組み合わせて別リポジトリを取得できる
4. checkout した別リポジトリのファイルを、通常のローカルファイルとして扱える

## 注意点と制約

### 1. 同一 owner 配下を前提にしている

`repository: ${{ github.repository_owner }}/${{ env.TARGET_REPO }}` という組み立てなので、現在のリポジトリと同じ owner 配下の別リポジトリを前提にしています。

### 2. GitHub App の権限とインストール範囲に依存する

実際にアクセスできるかどうかは、ワークフロー YAML ではなく GitHub App 側の設定に依存します。

### 3. `README.md` がないと最後の step で失敗する

checkout 自体が成功しても、対象リポジトリに `README.md` が存在しなければ `cat` で失敗します。

### 4. 現在のリポジトリ自体は checkout していない

このワークフローは `actions/checkout@v4` を使っていますが、取得しているのは現在のリポジトリではなく別リポジトリです。そのため、現在のリポジトリのファイルを前提にした step は入っていません。

## まとめ

`cross-repo.yml` は、GitHub App トークンを生成して別リポジトリを checkout し、その中のファイルを読むクロスリポジトリアクセスのサンプルワークフローです。

要点は次の 3 つです。

1. `APP_ID` と `PRIVATE_KEY` から GitHub App トークンを作る
2. そのトークンで `<owner>/<TARGET_REPO>` を checkout する
3. checkout した別リポジトリの `README.md` をログへ表示する
