# `workflow.rego` 詳細仕様

対象ファイル: `policy/workflow.rego`

## 概要

`workflow.rego` は、GitHub Actions のワークフロー定義に対して、ワークフローレベルの `permissions` 設定を検査する OPA/Rego ポリシーです。

このポリシーは `deny[msg]` ルールを 2 つ持っており、次の条件に一致した入力を拒否します。

1. workflow レベルの `permissions` が存在しない
2. workflow レベルの `permissions` が空オブジェクト `{}` ではない

つまり、このポリシーが許可するのは「workflow レベルで `permissions: {}` を明示している場合だけ」です。

## ポリシー全体

```rego
package main

deny[msg] {
    not input.permissions
    msg = "Workflow permissions are missing"
}

deny[msg] {
    input.permissions != {}
    msg = sprintf("Workflow permissions are not empty: %v", [input.permissions])
}
```

## 基本動作

このポリシーは、入力データ `input` に含まれる `permissions` フィールドを評価します。

評価結果は `deny` ルールの集合です。

1. `deny` が空
   - ポリシー上は許可
2. `deny` に 1 件以上のメッセージが入る
   - ポリシー上は拒否

このため、実際の利用側では通常 `data.main.deny` を見て、空かどうかで合否判定します。

## `package main`

```rego
package main
```

この宣言により、このポリシーは `main` パッケージに属します。

たとえば OPA からは、`data.main.deny` というパスで評価結果を参照します。

## ルール 1: `permissions` が未定義なら拒否

```rego
deny[msg] {
    not input.permissions
    msg = "Workflow permissions are missing"
}
```

### 役割

workflow レベルの `permissions` が省略されている場合に拒否メッセージを返します。

### 判定内容

```rego
not input.permissions
```

この条件は、`input.permissions` が存在しない場合に真になります。

このポリシーの意図としては、「workflow レベルで `permissions` を必ず明示しなければならない」という強制です。

### 返すメッセージ

```rego
"Workflow permissions are missing"
```

このメッセージは、設定漏れが原因で拒否されたことをそのまま示しています。

## ルール 2: `permissions` が空でなければ拒否

```rego
deny[msg] {
    input.permissions != {}
    msg = sprintf("Workflow permissions are not empty: %v", [input.permissions])
}
```

### 役割

workflow レベルの `permissions` が存在していても、その値が空オブジェクト `{}` でなければ拒否します。

### 判定内容

```rego
input.permissions != {}
```

この条件は、`permissions` に何らかの権限が書かれている場合に真になります。

たとえば次のような入力は拒否対象です。

1. `{"contents": "read"}`
2. `{"contents": "read", "pull-requests": "write"}`
3. `{"id-token": "write"}`

### 返すメッセージ

```rego
sprintf("Workflow permissions are not empty: %v", [input.permissions])
```

このメッセージは、拒否理由だけでなく、実際に指定されていた `permissions` の内容も含めて返します。

そのため、どの権限指定が問題だったかをログから確認しやすくなっています。

## このポリシーが許可する入力

このポリシーが許可するのは、次のように workflow レベルで `permissions: {}` を明示した入力です。

```json
{
  "permissions": {}
}
```

この場合の評価結果は、`deny` が空になります。

## このポリシーが拒否する入力

### 1. `permissions` 自体がない場合

入力例:

```json
{}
```

評価結果:

```json
[
  "Workflow permissions are missing"
]
```

### 2. `permissions` が空でない場合

入力例:

```json
{
  "permissions": {
    "contents": "read"
  }
}
```

評価結果のイメージ:

```json
[
  "Workflow permissions are not empty: {\"contents\":\"read\"}"
]
```

## GitHub Actions の文脈での意味

GitHub Actions では workflow レベルの `permissions` を設定すると、そのワークフロー内の `GITHUB_TOKEN` に与える権限範囲を制御できます。

このポリシーは、workflow レベルでは権限を一切与えず、必要な権限は job レベルで明示的に付与する運用を強制したい場合に有効です。

つまり、ポリシーの意図は次の通りです。

1. workflow 全体に広い権限を与えない
2. デフォルトを最小権限に固定する
3. 権限は必要な job にだけ限定して付ける

## 想定している安全性の考え方

`permissions: {}` を強制することで、workflow 全体の `GITHUB_TOKEN` 権限を空にできます。

そのうえで、必要な job にだけ次のような設定を個別に書く運用がしやすくなります。

```yaml
jobs:
  example:
    permissions:
      contents: read
```

この設計は、権限の意図しない広がりを防ぎやすいという利点があります。

## 評価結果の流れ

このポリシーの処理は次の順番で理解できます。

1. `input.permissions` の有無を確認する
2. なければ `"Workflow permissions are missing"` を `deny` に追加する
3. あれば、その値が `{}` かどうかを確認する
4. `{}` でなければ `"Workflow permissions are not empty: ..."` を `deny` に追加する
5. 最終的に `deny` が空なら許可、空でなければ拒否になる

## 注意点と制約

### 1. job レベルの `permissions` は見ていない

このポリシーが検査しているのは `input.permissions` だけです。job ごとの `permissions` 定義は、このファイル単体では評価していません。

### 2. workflow レベルで最小権限を宣言する方式に固定している

このポリシーでは、workflow レベルで `contents: read` のような最小権限を与える書き方も拒否されます。許容するのはあくまで `{}` だけです。

### 3. 入力形式は事前変換が必要

この Rego は GitHub Actions の YAML を直接読むのではなく、`input.permissions` を持つ形に変換された入力を前提としています。評価側で YAML から適切な JSON 入力を作る必要があります。

### 4. `permissions: null` などの扱いは入力側実装に依存しやすい

`not input.permissions` の挙動は、入力がどう正規化されているかに影響されます。評価パイプライン側で `permissions` をどう表現するかは明確にしておく必要があります。

## まとめ

`workflow.rego` は、workflow レベルの `permissions` を必須かつ空オブジェクト `{}` のみに制限する OPA/Rego ポリシーです。

要点は次の 3 つです。

1. `permissions` がなければ拒否する
2. `permissions` が `{}` 以外でも拒否する
3. workflow 全体の権限を空にし、必要な権限は job レベルで個別付与する運用を促す
