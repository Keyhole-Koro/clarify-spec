# RunAct 実装仕様（LangGraph）

## 目的

`ActService.RunAct` を仕様どおりの server-streaming として実装し、フロントへ安定した Draft Patch を供給する。

## 境界

* 入力: `RunActRequest`
* 出力: `RunActEvent` stream
* 非スコープ: Firestore永続化、layout固定、Organize更新

## 前提仕様

* `act/specs/rpc-connect-schema.md`
* `act/backend/act-langgraph-spec.md`
* `act/specs/act-flow.md`

## 状態遷移（MUST）

1. `ValidateRequest`
2. `LoadContext`
3. `BuildPrompt`
4. `GenerateWithModel`
5. `NormalizePatchOps`
6. `EmitStream`
7. `Finalize`
8. `HandleError`

## 実装ルール（MUST）

* `PatchOp` は `upsert` / `append_md` のみ許可
* `done` と `error` は同一Runで排他
* stream終端は最後に1回だけ
* `ACT_TYPE_UNSPECIFIED` は `INVALID_ARGUMENT`
* 親 -> 子の順序で patch を送る
* `append_md` の対象未存在時は `upsert` 補完を先行

## 制限値（MUST）

* request timeout: 45s
* model retry: 1回
* 推論ループ上限: 3
* patch_ops上限: 200/run
* anchor_node_ids: <= 20
* context_node_ids: <= 50

## エラーハンドリング

* 入力不正: 即 `error` 終端
* 外部依存障害: 再試行後 `error` 終端
* 部分送信後の破綻: `error` で終端（`done` は返さない）

## 完了条件（DoD）

* 正常入力で `done=true` まで返る
* 異常入力で `INVALID_ARGUMENT` を返す
* 禁止op混入が発生しない
