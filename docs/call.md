# `call.yml` 詳細仕様

対象ファイル: `.github/workflows/call.yml`

## 概要

`call.yml` は、Pull Request を契機に実行される呼び出し側ワークフローです。

このワークフロー自身が直接 PR へコメントするのではなく、`uses:` で `.github/workflows/reusable-workflows.yml` を呼び出します。呼び出し先が返した output `message` を、後続の `print` ジョブで表示する構成です。

## このワークフローの役割

`call.yml` は次の 2 つを担当します。

1. reusable workflow に必要な入力値と secrets を渡して実行する
2. reusable workflow の出力値を受け取り、後続ジョブで利用する

つまり、`.github/workflows/reusable-workflows.yml` の利用例をそのままワークフローとして表したファイルです。

## 起動条件

```yaml
on: pull_request
```

この設定により、Pull Request 関連イベントでワークフローが起動します。

このワークフローでは `github.event.pull_request.number` を使っているため、PR イベントで動くことに意味があります。PR 番号をそのまま reusable workflow へ渡せるためです。

## ジョブ構成

このワークフローには 2 つのジョブがあります。

1. `call`
2. `print`

実行順は次の通りです。

1. `call` ジョブが reusable workflow を呼び出す
2. `call` が成功すると `print` ジョブが起動する
3. `print` が `call` の output を表示する

## Job: `call`

### 役割

`call` ジョブは、`.github/workflows/reusable-workflows.yml` を reusable workflow として実行するジョブです。

### 定義

```yaml
call:
  uses: ./.github/workflows/reusable-workflows.yml
  with:
    pr-number: ${{ github.event.pull_request.number }}
  secrets:
    token: ${{ secrets.GITHUB_TOKEN }}
  permissions:
    contents: read
    pull-requests: write
```

### 何が行われるか

1. `.github/workflows/reusable-workflows.yml` を job の実体として呼び出す
2. `with.pr-number` で現在の PR 番号を渡す
3. `secrets.token` で `GITHUB_TOKEN` を渡す
4. `permissions` で呼び出し先が必要とする権限を与える

### `runs-on` がない理由

このジョブには `runs-on` がありません。理由は、この job 自体が通常の step 実行ジョブではなく、reusable workflow 呼び出しジョブだからです。

実際の実行環境は、呼び出し先である `.github/workflows/reusable-workflows.yml` 側の `runs-on` で決まります。このリポジトリでは呼び出し先の `comment` ジョブが `ubuntu-latest` を使います。

### `with` の意味

| 項目 | 値 | 意味 |
| --- | --- | --- |
| `pr-number` | `${{ github.event.pull_request.number }}` | 現在の Pull Request 番号を呼び出し先へ渡す |

この値は呼び出し先 reusable workflow の `inputs.pr-number` に対応します。

### `secrets` の意味

| 項目 | 値 | 意味 |
| --- | --- | --- |
| `token` | `${{ secrets.GITHUB_TOKEN }}` | GitHub CLI で PR コメントを書くためのトークンを呼び出し先へ渡す |

この値は呼び出し先 reusable workflow の `secrets.token` に対応します。

### `permissions` の意味

| 権限 | 用途 |
| --- | --- |
| `contents: read` | `actions/checkout@v4` でリポジトリを取得するため |
| `pull-requests: write` | PR へコメントを書き込むため |

呼び出し先 reusable workflow では `gh pr comment` を使ってコメント投稿するため、`pull-requests: write` が必要です。

### `call` ジョブの結果

このジョブが成功すると、呼び出し先 reusable workflow の output `message` を `call` ジョブの output として参照できるようになります。

このリポジトリの reusable workflow は `Welcome, <actor>` という文字列を返すため、`call` ジョブ完了後はその文字列が次のジョブへ引き渡されます。

## Job: `print`

### 役割

`print` ジョブは、`call` ジョブが返した output を受け取り、ログへ出力するためのジョブです。

### 定義

```yaml
print:
  needs: [call]
  runs-on: ubuntu-latest
  steps:
    - run: echo "Result> ${MESSAGE}"
      env:
        MESSAGE: ${{ needs.call.outputs.message }}
```

### 何が行われるか

1. `needs: [call]` により `call` の完了を待つ
2. `needs.call.outputs.message` を `MESSAGE` 環境変数へ格納する
3. `echo "Result> ${MESSAGE}"` を実行してログへ出力する

### `needs` の意味

`needs: [call]` には 2 つの意味があります。

1. `print` を `call` の後に実行する
2. `needs.call.outputs.message` として `call` の output を参照できるようにする

そのため、`print` は `call` が成功したときだけ実行されます。`call` が失敗した場合、通常は `print` はスキップされます。

### `MESSAGE` に入る値

`MESSAGE` には次の値が入ります。

```yaml
${{ needs.call.outputs.message }}
```

この値の元は、最終的に `.github/workflows/reusable-workflows.yml` が返した workflow output `message` です。

### ログ出力の例

呼び出し先が `Welcome, alice` を返した場合、`print` ジョブのログには次のように表示されます。

```text
Result> Welcome, alice
```

## データの流れ

このワークフロー全体の値の受け渡しは次の順番で進みます。

1. `pull_request` イベントでワークフローが起動する
2. `call` ジョブが `github.event.pull_request.number` を取得する
3. `call` ジョブが `reusable-workflows.yml` に PR 番号とトークンを渡す
4. 呼び出し先 reusable workflow が PR にコメントし、output `message` を返す
5. `print` ジョブが `needs.call.outputs.message` でその値を受け取る
6. `print` ジョブが `Result> ...` の形式でログへ表示する

## 実行結果のイメージ

Pull Request 番号が `123`、ワークフロー実行者が `alice` の場合、内部では次の流れになります。

1. `call` が `pr-number=123` を渡す
2. reusable workflow が PR `123` に `Welcome, alice` とコメントする
3. reusable workflow が `message=Welcome, alice` を返す
4. `print` が `Result> Welcome, alice` をログ出力する

## 呼び出し先との関係

`call.yml` だけではコメント処理は完結しません。実際の PR コメント投稿は、呼び出し先の `.github/workflows/reusable-workflows.yml` が担当しています。

役割分担は次の通りです。

- `call.yml`
  - 起動トリガーを持つ
  - 入力値と secrets を準備する
  - output を受け取って利用する
- `reusable-workflows.yml`
  - 実際に PR へコメントする
  - コメント本文を output として返す

## 注意点と制約

### 1. `pull_request` 以外ではこのまま使いにくい

`pr-number` に `${{ github.event.pull_request.number }}` を渡しているため、PR イベント以外で同じ書き方をすると値が取得できません。

### 2. 呼び出し先が失敗すると `print` は実行されない

`print` は `needs: [call]` に依存しているため、`call` ジョブが失敗すると通常は実行されません。

### 3. ログに出るのは output の内容だけ

`print` ジョブはコメント投稿そのものを行いません。表示するのは reusable workflow から返された文字列だけです。

### 4. 権限不足だと `call` 側で失敗する

`pull-requests: write` が不足すると、呼び出し先での PR コメント投稿が失敗します。

## まとめ

`call.yml` は reusable workflow を呼び出すためのサンプル兼実行ワークフローです。

要点は次の 3 つです。

1. `pull_request` で起動し、PR 番号を reusable workflow へ渡す
2. `GITHUB_TOKEN` を secret として渡し、呼び出し先に PR コメントを書かせる
3. 返ってきた output `message` を `print` ジョブでログ表示する
