# `flow-control.yml` 詳細仕様

対象ファイル: `.github/workflows/flow-control.yml`

## 概要

`flow-control.yml` は、`push` を契機に実行される GitHub Actions ワークフローです。

このワークフローの目的は、step の失敗後に `if:` 条件で後続 step の実行可否を制御する方法を示すことです。特に、`failure()` と `steps.<id>.outcome` を組み合わせて、「ジョブ内で失敗が発生したこと」と「特定の step が失敗したこと」を同時に判定しています。

## 何をするワークフローか

このワークフローには `run` という 1 つのジョブだけがあります。

1. 最初の step がランダムに成功または失敗する
2. 2 番目の step は常に `exit 1` で失敗する
3. 3 番目の step は、`failure()` と `steps.error.outcome == 'failure'` の両方が真のときだけ実行される

この構成により、後続 step の条件実行と、失敗した step の識別方法を確認できます。

## 起動条件

```yaml
on: push
```

この設定により、このワークフローは `push` のたびに起動します。

## ジョブ構成

```yaml
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - run: exit "$(( RANDOM % 2 ))"
      - run: exit 1
        id: error
      - run: echo "Catch error step"
        if: ${{ failure() && steps.error.outcome == 'failure' }}
```

ジョブは `ubuntu-latest` で動作し、3 つの step を上から順に評価します。

## Job: `run`

### 役割

`run` ジョブは、step 失敗時のデフォルト挙動と、`if:` を使った条件付き実行を確認するためのジョブです。

### 実行環境

```yaml
runs-on: ubuntu-latest
```

このジョブは Ubuntu ランナーで実行されます。`RANDOM` 変数や `exit` の書き方から見ても、`run` step は bash 系シェルでの実行を前提にした例です。

## 各 step の詳細

### 1. ランダム成功・失敗 step

```yaml
- run: exit "$(( RANDOM % 2 ))"
```

この step は `RANDOM % 2` を計算し、その結果を終了コードとして返します。

結果は次のどちらかです。

1. `0`
2. `1`

意味は次の通りです。

1. `0` の場合
   - step は成功する
2. `1` の場合
   - step は失敗する

そのため、この step は実行ごとに成功する場合と失敗する場合があります。

### 2. `error` step

```yaml
- run: exit 1
  id: error
```

この step は常に終了コード `1` を返すため、実行された場合は必ず失敗します。

ただし、この step が「実行されるかどうか」は 1 つ前の step の結果に依存します。GitHub Actions の step は、`if:` を明示しない限り、実質的に `success()` 条件で動きます。

つまり次のようになります。

1. 最初の step が成功した場合
   - `error` step は実行される
   - 実行結果は必ず `failure`
2. 最初の step が失敗した場合
   - `error` step はスキップされる
   - `steps.error.outcome` は `skipped` になる

### `id: error` の意味

この `id` は、後続 step から `steps.error.outcome` としてこの step の状態を参照するために必要です。

このワークフローでは、3 番目の step がその値を使って条件分岐しています。

### 3. エラー捕捉 step

```yaml
- run: echo "Catch error step"
  if: ${{ failure() && steps.error.outcome == 'failure' }}
```

この step は、無条件では実行されません。次の 2 条件が両方とも真のときだけ実行されます。

1. `failure()`
2. `steps.error.outcome == 'failure'`

## `failure()` の意味

`failure()` は、ジョブ内のそれ以前の処理で失敗が起きているかどうかを返す関数です。

このワークフローでは、次のように解釈できます。

1. それまでの step に失敗が 1 つでもある
   - `true`
2. それまでの step がすべて成功している
   - `false`

## `steps.error.outcome` の意味

`steps.error.outcome` は、`id: error` を持つ step の結果です。

このワークフローでは主に次の値を取り得ます。

1. `failure`
   - `error` step が実行されて失敗した
2. `skipped`
   - `error` step 自体が実行されなかった

## 条件式の意味

```yaml
if: ${{ failure() && steps.error.outcome == 'failure' }}
```

この条件式は、「ジョブ内で失敗が起きていて、しかもその失敗した対象が `error` step である」ときにだけ `Catch error step` を実行する、という意味です。

ポイントは次の通りです。

1. `failure()` だけでは「どの step が失敗したか」は分からない
2. `steps.error.outcome == 'failure'` を足すことで、「`error` step が失敗したケース」に絞り込める

## 実行パターン

### パターン 1: 最初の step が成功する場合

実行順は次のようになります。

1. step 1 が成功する
2. `error` step が実行される
3. `error` step が `exit 1` で失敗する
4. `failure()` は `true`
5. `steps.error.outcome` は `failure`
6. 条件式が真になる
7. `Catch error step` が実行される

このケースでは 3 番目の step が実行されます。

### パターン 2: 最初の step が失敗する場合

実行順は次のようになります。

1. step 1 が失敗する
2. `error` step はデフォルト条件のためスキップされる
3. `failure()` は `true`
4. `steps.error.outcome` は `skipped`
5. 条件式は `true && false` になり偽になる
6. `Catch error step` は実行されない

このケースでは 3 番目の step はスキップされます。

## 実行結果のまとめ

このワークフローで 3 番目の step が実行されるのは、次の条件を満たすときだけです。

1. 最初の step が成功する
2. `error` step が実行される
3. `error` step が失敗する

逆に、最初の step が失敗した場合は `error` step 自体が動かないため、3 番目の step も動きません。

## ジョブ全体の成否

このワークフローは、`Catch error step` が実行されても成功には戻りません。

理由は、`error` step が `exit 1` で失敗しているからです。`if:` 付きの後続 step は「失敗後に追加処理を行う」ためのものであり、失敗そのものを打ち消すものではありません。

そのため、`error` step が実行されて失敗したパターンでは、ジョブ全体は失敗として終わります。

## このワークフローが示しているポイント

このファイルのポイントは次の 4 つです。

1. step が失敗すると、後続 step はデフォルトでは実行されない
2. `if:` を使えば失敗後でも後続 step を実行できる
3. `failure()` で「何かが失敗した」ことを判定できる
4. `steps.<id>.outcome` で「どの step がどう終わったか」を判定できる

## 注意点と制約

### 1. 最初の step は非決定的

`RANDOM % 2` を使っているため、毎回同じ結果にはなりません。再現性が必要な実運用向けの書き方ではなく、挙動確認用のサンプルです。

### 2. `error` step は常に失敗する

`exit 1` が固定されているため、実行された場合は必ず失敗します。

### 3. `Catch error step` は失敗回復ではない

この step は追加の処理を行うだけで、ジョブを成功に戻すものではありません。

### 4. `id` がないと step 単位の結果を参照できない

`steps.error.outcome` を使うには、対象 step に `id: error` のような識別子が必要です。

## まとめ

`flow-control.yml` は、step 失敗後の制御を `failure()` と `steps.error.outcome` で分岐する最小構成のサンプルです。

要点は次の 3 つです。

1. 最初の step はランダムに成功または失敗する
2. `error` step は実行されれば必ず失敗する
3. 最終 step は「ジョブ内で失敗があり、かつ `error` step が失敗した」ときだけ実行される
