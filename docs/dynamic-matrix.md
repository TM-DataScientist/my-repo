# `dynamic-matrix.yml` 詳細仕様

対象ファイル: `.github/workflows/dynamic-matrix.yml`

## 概要

`dynamic-matrix.yml` は、`push` を契機に実行される GitHub Actions ワークフローです。

このワークフローの目的は、固定の `strategy.matrix` を直接 YAML に書く代わりに、先行ジョブで JSON 文字列を生成し、その JSON を `fromJSON()` で動的マトリックスへ変換して後続ジョブを複数実行することです。

このファイルでは、`prepare` ジョブが `ubuntu-latest` と `macos-latest` の 2 つの runner 定義を JSON として返し、`print` ジョブがその内容に基づいて 2 回実行されます。

## 何をするワークフローか

このワークフローの処理は大きく 2 段階です。

1. `prepare` ジョブがマトリックス定義を JSON 文字列として作成する
2. `print` ジョブがその JSON を `fromJSON()` で展開し、runner ごとに並列実行される

今回の設定では、最終的に `print` ジョブは次の 2 環境で実行されます。

1. `ubuntu-latest`
2. `macos-latest`

各実行では `RUNNER_OS` を表示するだけなので、ログには OS 名が出力されます。

## 起動条件

```yaml
on: push
```

この設定により、このワークフローは `push` のたびに起動します。

ブランチやタグの絞り込みは設定されていないため、デフォルトではこのリポジトリで発生したすべての `push` が対象です。

## ジョブ構成

このワークフローには 2 つのジョブがあります。

1. `prepare`
2. `print`

処理順は次の通りです。

1. `prepare` が JSON 文字列を job output として返す
2. `print` が `needs.prepare.outputs.matrix-json` を読む
3. `fromJSON()` で動的に matrix を生成する
4. matrix の各要素ごとに `print` が個別ジョブとして実行される

## Job: `prepare`

### 役割

`prepare` ジョブは、後続ジョブで使うマトリックス定義を JSON 文字列として組み立てるためのジョブです。

### 定義

```yaml
prepare:
  runs-on: ubuntu-latest
  steps:
    - id: dynamic
      run: |
        json='{"runner": ["ubuntu-latest", "macos-latest"]}'
        echo "json=${json}" >> "${GITHUB_OUTPUT}"
  outputs:
    matrix-json: ${{ steps.dynamic.outputs.json }}
```

### 何が行われるか

1. `ubuntu-latest` 上でジョブを開始する
2. step `dynamic` で JSON 文字列を変数 `json` に代入する
3. `echo "json=${json}" >> "${GITHUB_OUTPUT}"` で step output `json` を設定する
4. job output `matrix-json` に `steps.dynamic.outputs.json` を渡す

### JSON の内容

`prepare` ジョブが生成している JSON は次の文字列です。

```json
{"runner": ["ubuntu-latest", "macos-latest"]}
```

これは GitHub Actions の matrix 形式として見ると、次の YAML と同等です。

```yaml
matrix:
  runner:
    - ubuntu-latest
    - macos-latest
```

つまり、`runner` というキーに 2 つの値が入っているため、後続の `print` ジョブは 2 回実行されます。

### `GITHUB_OUTPUT` の意味

`GITHUB_OUTPUT` は、step から output を設定するための特殊ファイルです。

このワークフローでは次の行で step output を作っています。

```sh
echo "json=${json}" >> "${GITHUB_OUTPUT}"
```

これにより、step `dynamic` の output `json` が作成され、後続の job output で参照できるようになります。

### job output への変換

`prepare` ジョブは step output をそのまま job output に変換しています。

```yaml
outputs:
  matrix-json: ${{ steps.dynamic.outputs.json }}
```

そのため、後続ジョブは `needs.prepare.outputs.matrix-json` で JSON 文字列を参照できます。

## Job: `print`

### 役割

`print` ジョブは、`prepare` が返した JSON をもとに動的 matrix を生成し、各 runner 上で処理を実行するジョブです。

### 定義

```yaml
print:
  needs: [prepare]
  strategy:
    matrix: ${{ fromJSON(needs.prepare.outputs.matrix-json) }}
  runs-on: ${{ matrix.runner }}
  steps:
    - run: echo "${RUNNER_OS}"
```

### 何が行われるか

1. `needs: [prepare]` により `prepare` の完了を待つ
2. `needs.prepare.outputs.matrix-json` の JSON 文字列を取得する
3. `fromJSON()` でその文字列を matrix オブジェクトへ変換する
4. `matrix.runner` の値ごとに job を複製する
5. 各 job が対応する runner 上で `echo "${RUNNER_OS}"` を実行する

### `fromJSON()` の意味

`fromJSON()` は、文字列として持っている JSON を GitHub Actions の式の中でオブジェクトや配列として扱えるように変換する関数です。

このワークフローでは次の式が核心です。

```yaml
matrix: ${{ fromJSON(needs.prepare.outputs.matrix-json) }}
```

これにより、`prepare` ジョブが文字列として返した JSON が、`strategy.matrix` に使える構造へ変換されます。

### `runs-on: ${{ matrix.runner }}` の意味

matrix 展開後は、各ジョブごとに `matrix.runner` の値が変わります。

今回の組み合わせは次の 2 つです。

1. `matrix.runner = ubuntu-latest`
2. `matrix.runner = macos-latest`

そのため、`print` ジョブは実際には 2 つに分かれて実行されます。

### ログ出力の内容

`echo "${RUNNER_OS}"` は、そのジョブが動いている runner の OS 名を出力します。

一般的には次のような値になります。

1. Ubuntu runner では `Linux`
2. macOS runner では `macOS`

## データの流れ

このワークフロー全体の値の流れは次の通りです。

1. `push` でワークフローが起動する
2. `prepare` が JSON 文字列 `{"runner": ["ubuntu-latest", "macos-latest"]}` を作る
3. step output `json` が `GITHUB_OUTPUT` に書き込まれる
4. job output `matrix-json` がその値を公開する
5. `print` が `needs.prepare.outputs.matrix-json` でその値を受け取る
6. `fromJSON()` が文字列を matrix オブジェクトに変換する
7. `print` が runner ごとに複製される
8. 各ジョブが対応する OS 上で `RUNNER_OS` を出力する

## 実行結果のイメージ

このワークフローが 1 回起動すると、実質的には次のようなジョブ構成になります。

```yaml
print (runner=ubuntu-latest)
print (runner=macos-latest)
```

ログのイメージは次の通りです。

```text
print (ubuntu-latest) -> Linux
print (macos-latest) -> macOS
```

## 静的 matrix との違い

通常の静的 matrix なら、最初から次のように書けます。

```yaml
strategy:
  matrix:
    runner:
      - ubuntu-latest
      - macos-latest
```

このワークフローが動的 matrix を使っている理由は、matrix の内容を前段ジョブの結果から決められる点にあります。

たとえば実運用では、次のような用途に拡張できます。

1. 変更されたディレクトリに応じて対象ジョブを絞る
2. API や設定ファイルから対象環境一覧を生成する
3. 前段の解析結果に応じて実行対象を増減させる

## 注意点と制約

### 1. JSON 文字列が不正だと `print` が失敗する

`fromJSON()` は正しい JSON を前提にしています。`prepare` が壊れた JSON を返すと、`strategy.matrix` の評価時点で `print` ジョブは失敗します。

### 2. matrix のキー名と参照名は一致している必要がある

このワークフローでは JSON のキーが `runner` で、参照側も `matrix.runner` です。ここが一致しないと `runs-on` が解決できません。

### 3. 利用可能でない runner を返すと実行できない

たとえば存在しない runner 名や利用権限のない runner 名を JSON に含めると、その matrix 要素のジョブは実行できません。

### 4. `prepare` が失敗すると `print` は動かない

`print` は `needs: [prepare]` に依存しているため、`prepare` が失敗した場合は通常実行されません。

### 5. この例では include / exclude は使っていない

matrix は単純な 1 キー構成です。`include` や `exclude`、複数軸の組み合わせは定義していません。

## まとめ

`dynamic-matrix.yml` は、前段ジョブが作った JSON を `fromJSON()` で matrix 化する最小構成のサンプルです。

要点は次の 3 つです。

1. `prepare` が JSON 文字列として matrix 定義を返す
2. `print` が `fromJSON(needs.prepare.outputs.matrix-json)` で動的 matrix を生成する
3. 生成された各 runner ごとにジョブが複製され、対応する OS 上で実行される
