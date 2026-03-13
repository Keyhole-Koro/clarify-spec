# Act System Map（Human View）

## 3層で把握する

1. Interaction  
`frontend -> act-api -> stream`

2. Context Assembly  
`act-adk-worker` が context を読んで bundle を作る（read-only）

3. Knowledge Pipeline  
`organize` が write path を担当

## データの置き場所

* Firestore:
  * 確定済みの軽量メタ、関係、権限境界、検索キー
* GCS:
  * 本文、raw observation、生成物、versioned snapshot
* Act memory:
  * 実行中の ephemeral node、未確定候補、途中推論
  * UI 上に見えていても commit 前なら unsaved draft のまま

## 実装責務の境界

* `act-api`:
  * 認証認可
  * sid/csrf
  * idempotency
  * stream公開
* `act-adk-worker`:
  * ResolveIntent / RetrieveContext / RankAndBudget
  * Gemini呼び出し
  * Patch正規化
  * 実行中メモリの保持

## 昇格ルール

* Act memory は正本ではない
* 確定データは Organize を通して Firestore/GCS に昇格する
* frontend は draft と persisted を見分けられる表示を持つ

## 正本参照

* `act/specs/overview/act-architecture.md`
* `act/specs/behavior/act-flow.md`
* `act/act-api/specs/README.md`
* `act/act-adk-worker/specs/README.md`
