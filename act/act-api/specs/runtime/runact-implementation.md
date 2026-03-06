# RunAct 実装仕様（Go API + ADK Worker）

## 目的

`RunAct` を `topic_id` 中心で実行し、Go API から ADK Worker を利用して安定ストリームを返す。

## スコープ / 非スコープ

* スコープ: request検証、ADK委譲、stream返却
* 非スコープ: Organize write path、UI描画

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/context/core.md`
* `act/specs/context/bundle-schema.md`
* `act/act-api/specs/runtime/adk-service-integration.md`
* `act/act-adk-worker/specs/context/context-assembly-execution-profile.md`
* `act/act-api/specs/reliability/idempotency-policy.md`
* `act/act-api/specs/reliability/error-mapping-matrix.md`
* `organize/specs/model/topic-model.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`

## 契約（I/O）

入力:

* `RunActRequest`（`topic_id`, `request_id` 必須）
* `Authorization`, Cookie sid/csrf

出力:

* `RunActEvent` stream
* 失敗時 `ErrorInfo { code, retryable, stage, trace_id, retry_after_ms? }`

## 正常フロー

1. `AUTHN`: Firebase token 検証
2. token claim から `token_uid` を確定（request `uid` は互換用）
3. `SID_VALIDATE`: sid検証（soft許容）
4. `CSRF_VALIDATE`: Double Submit 照合
5. `AUTHZ`: token_uid -> workspace membership -> topic access
6. `VALIDATE_REQUEST`: `topic_id`, `request_id`, 上限検証
7. request `uid` が存在する場合は `token_uid` と一致検証
8. 冪等判定 `(token_uid, workspace_id, request_id)`
9. Go API が ADK Worker へ実行委譲
10. ADK Worker が `ASSEMBLY_* -> GENERATE_WITH_MODEL -> NORMALIZE_PATCH_OPS`
11. Go API が `RunActEvent` へ変換して `EMIT_STREAM`
12. `FINALIZE`: `done=true` で終端

## 異常フロー（error/retryable/stage）

* topic欠落: `INVALID_ARGUMENT`, `retryable=false`, `stage=VALIDATE_REQUEST`
* topicアクセス不可: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* ADK worker接続失敗: `UNAVAILABLE`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* ADK worker timeout: `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* assembly参照失敗: `UNAVAILABLE`, `retryable=true`, `stage=ASSEMBLY_RETRIEVE`
* request重複: `ALREADY_EXISTS`, `retryable=false`, `stage=VALIDATE_REQUEST`
* 返却形式不正: `INTERNAL`, `retryable=false`, `stage=NORMALIZE_PATCH_OPS`

## 数値パラメータ

* `request_timeout_ms=90000`
* `adk_worker_timeout_ms=60000`
* `adk_worker_retry_max=1`
* `patch_ops_max_per_run=400`
* `context_node_ids_max=120`
* `thought_flush_interval_ms=500`

## 受け入れ条件（DoD）

* `topic_id` なし request は必ず拒否
* `RunAct` 契約を変えず ADK経由で実行できる
* `ErrorInfo.stage` が assembly/生成段階まで一貫
* `done/error` 排他が守られる

## 実装メモ（最小）

* Go API は契約ゲートウェイとして維持する
* ADK Worker は read-only実行に限定する
* `tree_id` は UI scope で扱い、正本キーには使わない
