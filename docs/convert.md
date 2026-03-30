# `convert.yml` 詳細仕様

対象ファイル: `.github/workflows/convert.yml`

## 概要

`convert.yml` は、`push` を契機に実行される GitHub Actions ワークフローです。

このワークフローの目的は、環境変数として定義した値を `fromJSON()` で型変換し、`timeout-minutes` に渡す方法を示すことです。

このファイルでは `TIMEOUT` を `1` として定義し、step の `timeout-minutes` に `${{ fromJSON(env.TIMEOUT) }}` を設定しています。その結果、`sleep 120` を実行しても step は 1 分でタイムアウトします。

## 何をするワークフローか

このワークフローの実処理は 1 ジョブだけです。

1. `push` でワークフローが起動する
2. top-level `env` で `TIMEOUT=1` を定義する
3. `sleep` ジョブを `ubuntu-latest` で実行する
4. `sleep 120` を実行する
5. `timeout-minutes` に設定した 1 分に達すると step を打ち切る

つまり、このワークフローは「文字列として持っている値を数値へ変換して、数値を要求する項目に渡す」ための最小サンプルです。

## 起動条件

```yaml
on: push
```

この設定により、ブランチやタグを問わず `push` のたびにワークフローが起動します。

## top-level `env`

```yaml
env:
  TIMEOUT: 1
```

この `env` はワークフロー全体で参照できる環境変数です。

見た目は数値の `1` ですが、GitHub Actions の `env` で扱われる値は式の文脈では文字列として扱われます。そのため、数値型を期待するフィールドへそのまま渡すのではなく、必要に応じて型変換が必要になります。

## ジョブ構成

このワークフローには `sleep` という 1 つのジョブだけがあります。

```yaml
jobs:
  sleep:
    runs-on: ubuntu-latest
    steps:
      - run: sleep 120
        timeout-minutes: ${{ fromJSON(env.TIMEOUT) }}
```

## Job: `sleep`

### 役割

`sleep` ジョブは、`timeout-minutes` に値を渡すときに `fromJSON()` を使って文字列を数値へ変換する例を示すジョブです。

### 何が行われるか

1. `ubuntu-latest` runner でジョブを開始する
2. `run: sleep 120` で 120 秒待機するコマンドを実行する
3. 同時に `timeout-minutes` に 1 を設定する
4. 60 秒を超えた時点で GitHub Actions が step をタイムアウトさせる

### `runs-on`

```yaml
runs-on: ubuntu-latest
```

このジョブは Ubuntu ランナー上で実行されます。`sleep 120` は POSIX 系シェルの標準的な待機コマンドとして動作します。

## `timeout-minutes` の意味

```yaml
timeout-minutes: ${{ fromJSON(env.TIMEOUT) }}
```

`timeout-minutes` は、その step が最大何分まで実行を許可されるかを指定するフィールドです。

今回の設定では `TIMEOUT=1` なので、step の許容時間は 1 分です。

一方で実行しているコマンドは `sleep 120` なので、完走には約 2 分かかります。そのため、この step は正常終了せず、1 分経過時点でタイムアウト終了します。

## `fromJSON()` を使う理由

### なぜ変換が必要か

`env.TIMEOUT` は環境変数参照なので、式の中では文字列として扱われます。

しかし `timeout-minutes` は数値を期待する設定項目です。そのため、文字列 `"1"` を数値 `1` として扱えるように変換する必要があります。

### このワークフローでの変換

```yaml
${{ fromJSON(env.TIMEOUT) }}
```

`fromJSON()` は JSON として解釈できる文字列を、GitHub Actions の式の中で適切な型へ変換します。

今回の `env.TIMEOUT` は実質的に `"1"` として扱われるため、`fromJSON("1")` によって数値 `1` に変換されます。

### 変換しない場合の意図

このファイルの主題は「明示的に型変換して数値項目へ渡す」ことです。`fromJSON()` を使うことで、文字列と数値の違いを意識した書き方になっています。

特に、将来的に `true` / `false`、数値、配列、オブジェクトなどを文字列経由で扱う場合にも、同じ考え方を応用できます。

## 実行結果のイメージ

このワークフローが起動すると、`sleep` ジョブは次の流れになります。

1. step 開始
2. `sleep 120` 実行
3. 1 分経過
4. step がタイムアウト
5. ジョブは失敗として終了

ログのイメージとしては、`sleep 120` が完了する前にタイムアウト関連のメッセージが出ます。

## データと評価の流れ

値の流れは次の通りです。

1. top-level `env` に `TIMEOUT: 1` を定義する
2. step の式 `${{ fromJSON(env.TIMEOUT) }}` が評価される
3. `env.TIMEOUT` が文字列として参照される
4. `fromJSON()` がその文字列を数値へ変換する
5. `timeout-minutes` に数値 `1` が設定される
6. step 実行時間が 1 分を超えるとタイムアウトする

## このワークフローが示しているポイント

このファイルのポイントは次の 3 つです。

1. `env` の値をそのまま使うのではなく、必要に応じて型変換できる
2. `fromJSON()` は文字列から数値や真偽値などへ変換する手段として使える
3. `timeout-minutes` のような数値項目では、明示的な変換が有効な場合がある

## 注意点と制約

### 1. `TIMEOUT` が不正な値だと式評価に失敗する

`fromJSON()` は JSON として解釈できる文字列を前提にしています。たとえば `TIMEOUT: abc` のような値にすると、`fromJSON(env.TIMEOUT)` は失敗します。

### 2. この設定では step は必ずタイムアウトする

`sleep 120` は約 2 分かかる一方で、`timeout-minutes` は 1 分です。そのため、現在の設定は成功例ではなくタイムアウト例です。

### 3. タイムアウト対象は step 単位

このファイルで設定している `timeout-minutes` は step に付いています。job 全体のタイムアウトではありません。

### 4. top-level `env` はワークフロー全体に見える

今回は 1 ジョブしかありませんが、top-level `env` に置いた `TIMEOUT` は他のジョブや step でも参照できます。

## まとめ

`convert.yml` は、環境変数の文字列値を `fromJSON()` で数値へ変換し、`timeout-minutes` に渡す最小構成のサンプルです。

要点は次の 3 つです。

1. `TIMEOUT` は環境変数として定義されている
2. `fromJSON(env.TIMEOUT)` で数値 `1` に変換している
3. `sleep 120` は 1 分でタイムアウトするため、ジョブは失敗終了する
