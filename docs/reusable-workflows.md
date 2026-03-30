# `reusable-workflows.yml` 詳細仕様

対象ファイル: `.github/workflows/reusable-workflows.yml`

## 概要

`reusable-workflows.yml` は `workflow_call` で呼び出す再利用可能ワークフローです。呼び出し元から PR 番号とトークンを受け取り、対象の Pull Request に `Welcome, <actor>` というコメントを投稿します。

このワークフローは、投稿したコメント本文をワークフロー出力 `message` として呼び出し元へ返します。

## `workflow_call` とは何か

`workflow_call` は、GitHub Actions のワークフローを別のワークフローから部品として呼び出すためのイベントです。

通常の `push` や `pull_request` は GitHub 上の出来事をきっかけに実行されますが、`workflow_call` は「別のワークフローから `uses:` で呼ばれたとき」に実行されます。そのため、`workflow_call` を定義した YAML は単体で自動起動するのではなく、呼び出し元ワークフローから利用されることを前提にした共通処理として使います。

この仕組みを使うと、複数のワークフローで共通の処理を 1 か所にまとめられます。今回の `reusable-workflows.yml` では、PR へ歓迎コメントを投稿する処理を再利用可能な形に切り出しています。

### `workflow_call` で定義できるもの

`workflow_call` では主に次の 3 つを宣言できます。

1. `inputs`
2. `secrets`
3. `outputs`

このファイルでもその 3 つを使っています。

- `inputs.pr-number`
  - コメント対象の PR 番号を呼び出し元から受け取る
- `secrets.token`
  - GitHub CLI でコメント投稿するためのトークンを受け取る
- `outputs.message`
  - 投稿したコメント本文を呼び出し元へ返す

### 呼び出し元から見る `workflow_call`

呼び出し元ワークフローは、通常の job 定義の代わりに `uses:` で reusable workflow を指定します。

```yaml
jobs:
  welcome-comment:
    uses: ./.github/workflows/reusable-workflows.yml
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

このように書くことで、`welcome-comment` ジョブの実体として `reusable-workflows.yml` が実行されます。

### このワークフローにおける `workflow_call` の意味

このファイルの `on.workflow_call` は、次のような役割を持っています。

1. 呼び出し元から `pr-number` を受け取れるようにする
2. 呼び出し元から `token` を安全に受け取れるようにする
3. 実行結果として `message` を返せるようにする

つまり、このワークフローは「PR コメント投稿を行う再利用可能なジョブ部品」として設計されています。

## 何をするワークフローか

このワークフローの実処理は 1 ジョブだけです。

1. `workflow_call` で呼び出される
2. `ubuntu-latest` ランナーで `comment` ジョブを実行する
3. `actions/checkout@v4` でリポジトリをチェックアウトする
4. `Welcome, ${GITHUB_ACTOR}` というコメント本文を組み立てる
5. `gh pr comment` で指定 PR にコメントする
6. コメント本文を step output に保存する
7. job output を経由して workflow output `message` として返す

## インターフェース

### Inputs

| 名前 | 型 | 必須 | 既定値 | 用途 |
| --- | --- | --- | --- | --- |
| `pr-number` | `string` | いいえ | `${{ github.event.pull_request.number }}` | コメント対象の Pull Request 番号 |

補足:

- `pr-number` を明示指定しない場合、YAML 上の既定値は `${{ github.event.pull_request.number }}` です。
- 呼び出し元が PR 関連イベントでない場合、この値は空になる可能性があるため、明示的に `with.pr-number` を渡す方が安全です。

### Secrets

| 名前 | 必須 | 用途 |
| --- | --- | --- |
| `token` | はい | `gh pr comment` を実行するための GitHub トークン |

補足:

- このトークンには対象 PR へコメントを書き込める権限が必要です。
- ワークフロー内では `GITHUB_TOKEN` 環境変数として `gh` CLI に渡されています。

### Outputs

| 名前 | 値 | 説明 |
| --- | --- | --- |
| `message` | `${{ jobs.comment.outputs.pr-comment }}` | 投稿したコメント本文 |

返される値は、実際には step `pr-comment` が `GITHUB_OUTPUT` に書き込んだ `body` の値です。

## ジョブ構成

### Job: `comment`

`comment` ジョブはこのワークフローの本体です。

| 項目 | 値 |
| --- | --- |
| 実行環境 | `ubuntu-latest` |
| job permissions | `contents: read`, `pull-requests: write` |
| job output | `pr-comment: ${{ steps.pr-comment.outputs.body }}` |

### permissions の意味

- `contents: read`
  - `actions/checkout@v4` でリポジトリを取得するための最小限の読み取り権限です。
- `pull-requests: write`
  - PR へのコメント投稿に必要な権限です。

## 各 step の詳細

### 1. `actions/checkout@v4`

最初の step でリポジトリをチェックアウトします。

このワークフローではソースコードのビルドやテストはしていませんが、`gh pr comment` が現在の Git リポジトリ情報を使って対象リポジトリを解決するため、チェックアウトには意味があります。

### 2. `pr-comment` step

次のシェルスクリプトを実行します。

```sh
body="Welcome, ${GITHUB_ACTOR}"
gh pr comment "${PR_NUMBER}" --body "${body}"
echo "body=${body}" >>"${GITHUB_OUTPUT}"
```

この step の動きは以下の通りです。

1. `GITHUB_ACTOR` を使ってコメント文字列を生成する
2. `gh pr comment` で対象 PR にコメントを投稿する
3. 同じ文字列を `body` という名前で step output に保存する

### step に渡される環境変数

| 環境変数 | 値 |
| --- | --- |
| `PR_NUMBER` | `${{ inputs.pr-number }}` |
| `GITHUB_TOKEN` | `${{ secrets.token }}` |

補足:

- `GITHUB_ACTOR` は GitHub Actions が自動で提供する組み込み環境変数です。
- そのため、コメント本文は常にワークフロー実行者に応じて変わります。

## 出力値の流れ

出力は次の順番で受け渡されます。

1. step `pr-comment` が `body=<コメント本文>` を `GITHUB_OUTPUT` に書き込む
2. job output `pr-comment` が `${{ steps.pr-comment.outputs.body }}` を参照する
3. workflow output `message` が `${{ jobs.comment.outputs.pr-comment }}` を参照する

たとえば `GITHUB_ACTOR` が `octocat` なら、最終的な workflow output `message` は次の値になります。

```text
Welcome, octocat
```

## 呼び出し例

同一リポジトリ内から呼び出す最小例です。

```yaml
name: Call Reusable Workflow

on:
  pull_request:

jobs:
  welcome-comment:
    uses: ./.github/workflows/reusable-workflows.yml
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

呼び出し後に output を利用する例です。

```yaml
name: Call Reusable Workflow With Output

on:
  pull_request:

jobs:
  welcome-comment:
    uses: ./.github/workflows/reusable-workflows.yml
    with:
      pr-number: ${{ github.event.pull_request.number }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}

  consume-output:
    runs-on: ubuntu-latest
    needs: welcome-comment
    steps:
      - run: echo "${{ needs.welcome-comment.outputs.message }}"
```

## 実行結果のイメージ

PR 番号が `123`、実行者が `alice` の場合、このワークフローは次のコメントを投稿します。

```text
Welcome, alice
```

そして呼び出し元では、output `message` として同じ値を受け取れます。

## 注意点と制約

### 1. コメント本文は固定

本文は `Welcome, ${GITHUB_ACTOR}` に固定されています。呼び出し元から本文を変更する input はありません。

### 2. 実行のたびに新しいコメントを追加する

既存コメントの更新や重複チェックはしていません。そのため、同じ PR に対して複数回実行すると、その回数分だけコメントが追加されます。

### 3. `pr-number` が空または不正だと失敗する

`gh pr comment` は有効な PR 番号が必要です。`pr-number` が空、数値でない、または存在しない PR を指している場合は step が失敗します。

### 4. トークン権限が不足すると失敗する

`token` に PR コメントを書き込む権限がない場合、`gh pr comment` は失敗します。

### 5. `gh` CLI に依存している

コメント投稿は GitHub CLI で行っています。そのため、このワークフローは `gh` コマンドが使える実行環境を前提にしています。現在の設定では `ubuntu-latest` を使っているため、その前提で構成されています。

## まとめ

このワークフローは、指定された Pull Request に歓迎コメントを投稿し、その本文を output として返すための最小構成の reusable workflow です。

要点は次の 3 つです。

1. 入力は `pr-number`、秘密情報は `token`
2. 実処理は `gh pr comment` による PR コメント投稿
3. 出力 `message` でコメント本文を呼び出し元へ返す
