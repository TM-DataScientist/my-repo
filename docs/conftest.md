# `conftest.yml` 詳細仕様

対象ファイル: `.github/workflows/conftest.yml`

## 概要

`conftest.yml` は、Pull Request を契機に実行される GitHub Actions ワークフローです。

このワークフローの目的は、`.github/workflows/` 配下の GitHub Actions ワークフローファイルを、`policy/` 配下の Rego ポリシーで検証することです。実行には Conftest を使い、GitHub Actions ランナー上では Docker コンテナとして `openpolicyagent/conftest` イメージを起動しています。

現在のリポジトリでは、主に [policy/workflow.rego](C:/Users/09027/Documents/05.portforio/書籍/20260224_Github_CI_CD_実践ガイド/my-repo/policy/workflow.rego) が評価対象ポリシーになります。

## 何をするワークフローか

このワークフローの処理は 1 ジョブだけです。

1. `pull_request` でワークフローが起動する
2. リポジトリをチェックアウトする
3. Docker で `openpolicyagent/conftest` コンテナを起動する
4. `policy/` 配下の Rego を使って `.github/workflows/` 配下を検証する
5. ポリシー違反があればジョブを失敗させる

つまり、このワークフローは「Pull Request 時に GitHub Actions ワークフロー定義のポリシーチェックを自動実行する CI」です。

## 起動条件

```yaml
on: pull_request
```

この設定により、Pull Request 関連イベントでワークフローが起動します。

`push` ではなく `pull_request` にしているため、レビュー前の変更を検査し、問題のある workflow 定義をマージ前に検出することが目的だと読み取れます。

## ジョブ構成

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          docker run --rm -v "$(pwd):$(pwd)" -w "$(pwd)" \
          openpolicyagent/conftest test --policy policy/ .github/workflows/
```

このワークフローには `test` という 1 つのジョブだけがあります。

## Job: `test`

### 役割

`test` ジョブは、Conftest を使って GitHub Actions workflow 定義を静的検証するジョブです。

### 実行環境

```yaml
runs-on: ubuntu-latest
```

このジョブは Ubuntu ランナー上で動作します。

この選択には次の意味があります。

1. `docker` コマンドが使える
2. `$(pwd)` を使うシェル構文で現在ディレクトリを渡せる
3. Conftest を直接インストールしなくても Docker コンテナ経由で実行できる

## 各 step の詳細

### 1. `actions/checkout@v4`

```yaml
- uses: actions/checkout@v4
```

最初の step でリポジトリをランナーへチェックアウトします。

この step が必要な理由は、後続の Conftest コンテナに次のファイル群を見せるためです。

1. `policy/` 配下の Rego ポリシー
2. `.github/workflows/` 配下の検査対象ファイル

checkout がないと、ランナー上にこれらのファイルが存在せず、Conftest で評価できません。

### 2. Conftest 実行 step

```sh
docker run --rm -v "$(pwd):$(pwd)" -w "$(pwd)" \
openpolicyagent/conftest test --policy policy/ .github/workflows/
```

この step がワークフロー本体です。

### コマンドの分解

この Docker コマンドは次のように読めます。

1. `docker run --rm`
   - コンテナを起動し、終了後に自動削除する
2. `-v "$(pwd):$(pwd)"`
   - 現在のワークスペースをコンテナ内の同じパスへマウントする
3. `-w "$(pwd)"`
   - コンテナ内の作業ディレクトリを、そのマウント先パスに設定する
4. `openpolicyagent/conftest`
   - Conftest のコンテナイメージを使う
5. `test --policy policy/ .github/workflows/`
   - `policy/` 配下の Rego を使って `.github/workflows/` 配下を検証する

## `conftest test` の意味

`conftest test` は、指定したファイル群をポリシーで評価し、違反があれば非ゼロ終了コードを返すコマンドです。

このワークフローでは次の指定になっています。

```sh
conftest test --policy policy/ .github/workflows/
```

意味は次の通りです。

1. `--policy policy/`
   - 使用する Rego ポリシーのディレクトリを `policy/` にする
2. `.github/workflows/`
   - 検査対象のファイル群を `.github/workflows/` 以下にする

## このリポジトリで実際に何を検査しているか

現時点で `policy/` 配下には [policy/workflow.rego](C:/Users/09027/Documents/05.portforio/書籍/20260224_Github_CI_CD_実践ガイド/my-repo/policy/workflow.rego) があります。

このポリシーは、workflow レベルの `permissions` について次の条件を強制します。

1. `permissions` が省略されていてはいけない
2. `permissions` は空オブジェクト `{}` でなければならない

そのため、`conftest.yml` の実質的な役割は「Pull Request で追加・変更された workflow 定義が、workflow レベル権限ポリシーに違反していないかを確認すること」です。

## 成功時の挙動

Conftest が `.github/workflows/` 配下の全対象ファイルを評価し、`deny` ルールに一致するものがなければ、この step は成功します。

その場合、`test` ジョブ全体も成功し、Pull Request 上ではこのチェックは pass になります。

## 失敗時の挙動

ポリシー違反がある workflow ファイルが見つかると、`conftest test` は非ゼロ終了コードを返します。

すると次のようになります。

1. Conftest 実行 step が失敗する
2. `test` ジョブが失敗する
3. ワークフロー全体が失敗する
4. Pull Request 上でチェック失敗として表示される

これにより、ポリシー違反のある workflow 変更を早い段階で検出できます。

## データの流れ

このワークフロー全体の処理を順番に整理すると、次の通りです。

1. Pull Request イベントでワークフローが起動する
2. ランナー上にリポジトリを checkout する
3. 現在のワークスペースを Docker コンテナへマウントする
4. Conftest が `policy/` の Rego を読み込む
5. Conftest が `.github/workflows/` 配下の YAML を入力として評価する
6. `deny` ルールに一致するものがあれば失敗、なければ成功する

## 実行対象の広さ

このコマンドは個別ファイルではなく `.github/workflows/` ディレクトリ全体を対象にしています。

つまり、Pull Request で変更したファイルだけではなく、その時点でディレクトリ内に存在する workflow 定義全体が評価対象になります。

この設計には次の特徴があります。

1. 既存 workflow の違反も検出できる
2. 変更対象が 1 ファイルでも、全体整合性を確認できる
3. 一方で、未修正の既存違反があると PR と無関係でも失敗し得る

## Docker 利用の意味

このワークフローは Conftest を GitHub Actions ランナーへ直接インストールしていません。代わりに公式イメージをその場で起動しています。

この方式には次の利点があります。

1. ランナーへのツール導入手順が不要
2. Conftest の実行環境をイメージに閉じ込められる
3. ローカルでも同じ Docker コマンドを使って再現しやすい

## ローカル再現コマンド

同じ検証はローカルでも次のコマンドで再現できます。

```powershell
docker run --rm -v "${PWD}:${PWD}" -w "${PWD}" openpolicyagent/conftest test --policy policy/ .github/workflows/
```

PowerShell ではパス展開の書き方が異なるため、GitHub Actions の `$(pwd)` とは別に `${PWD}` を使う方が自然です。

## 注意点と制約

### 1. Docker が前提

このワークフローは `docker run` に依存しています。ランナー側で Docker が利用できることが前提です。

### 2. workflow YAML を直接評価しているわけではない

Conftest はファイルを入力データとして評価しますが、ポリシーがどのフィールドを読むかは Rego 側に依存します。つまり、検査内容の本質は `policy/` 配下のポリシー定義で決まります。

### 3. 現状のポリシー範囲は限定的

今のポリシーは workflow レベルの `permissions` しか見ていません。job レベル権限、uses の固定バージョン、危険な shell 記述などはこのファイル単体では検査していません。

### 4. ディレクトリ全体評価なので既存違反の影響を受ける

`.github/workflows/` 全体を対象にしているため、対象 PR で触っていない既存ファイルの違反でもジョブが失敗する可能性があります。

## まとめ

`conftest.yml` は、Pull Request 時に `.github/workflows/` 配下の workflow 定義を Conftest と Rego ポリシーで検証する CI ワークフローです。

要点は次の 3 つです。

1. `pull_request` で起動する
2. Docker 版 Conftest で `policy/` を使って `.github/workflows/` 全体を検査する
3. ポリシー違反があればジョブを失敗させ、PR 上で検出できるようにする
