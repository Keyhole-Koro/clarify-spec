# RunAct 実装仕様（Go / Topic-Centric）

## 目的

`RunAct` を `topic_id` 中心で実行し、認証/認可後に Context Assembly を経由して安定ストリームを返す。

## スコープ / 非スコープ

* スコープ: request検証、context assembly、model実行、stream返却
* 非スコープ: Organize write path、UI描画

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/context/core.md`
* `act/specs/context/bundle-schema.md`
* `organize/specs/model/topic-model.md`
* `act/specs/behavior/session-and-auth-boundary.md`

## 契約（I/O）

入力:

* `RunActRequest`（`topic_id`, `request_id` 必須）
* `Authorization`, Cookie sid/csrf

出力:

* `RunActEvent` stream
* 失敗時 `ErrorInfo { code, retryable, stage, trace_id, retry_after_ms? }`

## 正常フロー

1. `AUTHN`: Firebase token 検証
2. `SID_VALIDATE`: sid検証（soft許容）
3. `CSRF_VALIDATE`: Double Submit 照合
4. `AUTHZ`: uid -> workspace membership -> topic access
5. `VALIDATE_REQUEST`: `topic_id`, `request_id`, 上限検証
6. 冪等判定 `(uid, workspace_id, request_id)`
7. `ASSEMBLY_*`: shared仕様に従って context bundle 生成
8. `GENERATE_WITH_MODEL`: bundle を入力に推論
9. `NORMALIZE_PATCH_OPS`: `upsert/append_md` のみへ正規化
10. `EMIT_STREAM`: patch/text/thought 逐次返却
11. `FINALIZE`: `done=true` で終端

## 異常フロー（error/retryable/stage）

* topic欠落: `INVALID_ARGUMENT`, `retryable=false`, `stage=VALIDATE_REQUEST`
* topicアクセス不可: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* assembly参照失敗: `UNAVAILABLE`, `retryable=true`, `stage=ASSEMBLY_RETRIEVE`
* assembly予算不足: degrade継続（原則エラー化しない）
* request重複: `ALREADY_EXISTS`, `retryable=false`, `stage=VALIDATE_REQUEST`
* モデル障害: `UNAVAILABLE` or `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`

## 数値パラメータ

* `request_timeout_ms=90000`
* `model_max_retries=2`
* `patch_ops_max_per_run=400`
* `context_node_ids_max=120`
* `thought_flush_interval_ms=500`

## 受け入れ条件（DoD）

* `topic_id` なし request は必ず拒否
* Assembly が read-only で実行される
* `ErrorInfo.stage` が assembly段階を含めて一貫
* `done/error` 排他が守られる

## 実装メモ（最小）

* `tree_id` は UI scope として扱い、正本参照キーには使わない
* diagnostics はログに出して UIへは必要最小限返す
