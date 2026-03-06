# Act System Map（Human View）

## 3層で把握する

1. Interaction  
`frontend -> act-api -> stream`

2. Context Assembly  
`act-adk-worker` が context を読んで bundle を作る（read-only）

3. Knowledge Pipeline  
`organize` が write path を担当

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

## 正本参照

* `act/specs/overview/act-architecture.md`
* `act/specs/behavior/act-flow.md`
* `act/act-api/specs/README.md`
* `act/act-adk-worker/specs/README.md`
