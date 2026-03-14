# Topic Activity Spec

目的: input ごとの routing / draft / bundle / outline 更新を、1 本の activity として表示する。

## 1. Source of Truth

* `frontend/frontend-spec.md`
* `organize/specs/pipeline/agents.md`
* `organize/specs/pipeline/topic-resolution.md`
* `firestore/schema.md`

## 2. 役割

`topic-activity` は次を一列で見せる。

* upload progress
* TopicResolver 判定
* draft 反映
* bundle 生成
* outline 更新

これは node detail とは別責務とし、右ペインの独立モードで表示する。

## 3. 1アイテムの単位

activity の 1 行は「1 input に紐づく更新列」とする。

最低限まとめる対象:

* input metadata
* progress status
* routing result
* draft update summary
* bundle preview summary
* outline update summary

## 4. 表示構造

1 行の構造:

1. Header
* input title / filename
* createdAt
* current status

2. Routing block
* `attach_existing | create_new`
* confidence level
* reason
* topic title または `resolvedTopicId`

3. Draft block
* 追加 atom 数
* draft version
* 差分要約

4. Bundle block
* bundle id
* bundle description summary
* schema change の有無

5. Outline block
* outline version
* 反映結果サマリー

## 5. Routing block の仕様

表示必須:

* `resolutionMode`
* `resolutionReason`
* `resolvedTopicId`

表示推奨:

* confidence level (`high | medium | low`)

表示禁止:

* raw candidate score 一覧を通常 UI にそのまま出すこと

## 6. Bundle block の仕様

A6 `bundle_desc` を primary 表示に使ってよい。

表示必須:

* bundle の存在
* 人間向け要約

表示推奨:

* schema proposal 有無
* `要検証` 項目の有無

## 7. Outline block の仕様

表示必須:

* `outline.updated` 到達有無
* 最新 outline version

表示推奨:

* changed nodes 数
* 「新規ノードが追加された」「既存ノードに統合された」等の文言

## 8. 並び順

* 既定は `createdAt desc`
* progress 中の input は上に pin してよい
* `failed` は progress 中の次に優先表示してよい

## 9. フィルタ

MVP 最小:

* `all`
* `processing`
* `completed`
* `failed`

## 10. 開発モード表示

開発モードでは補助表示してよい。

* `traceId`
* `inputId`
* `bundleId`
* `draftVersion`
* `outlineVersion`

## 11. 未決事項

人間判断が必要:

* bundle preview を HTML render にするか plain summary にするか
* `draft diff` を本文で見せるか件数だけにするか
* processing 中アイテムの pin ルール
