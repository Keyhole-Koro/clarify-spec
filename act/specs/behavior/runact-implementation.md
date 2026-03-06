# RunAct 実装仕様（LangGraph / Go）

## 目的

`ActService.RunAct` を仕様どおりの server-streaming として実装し、フロントへ安定した Draft Patch を供給する。

## スコープ / 非スコープ

* スコープ: `RunActRequest` の検証、冪等、モデル実行、`RunActEvent` ストリーム送信
* 非スコープ: Firestore永続化、Organize更新、UIレイアウト

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
* `act/specs/behavior/session-and-auth-boundary.md`
* `act/specs/behavior/access-control-middleware.md`
* `act/specs/behavior/act-langgraph-runtime.md`
* `act/specs/quality/backend-parameter-index.md`

## 契約（I/O）

入力:

* `RunActRequest`（`request_id` 必須、`sid` は互換フィールド）
* `Authorization`（Firebase ID Token）
* Cookie `sid`（正本）
* Cookie/Header の CSRF ペア（Double Submit）

出力:

* `RunActEvent` stream
* 失敗時 `ErrorInfo { code, message, retryable, stage, trace_id, retry_after_ms? }`

## 処理フロー（正常）

1. `AUTHN`: Firebase token を検証し `uid` を確定する
2. `SID_VALIDATE`: sid Cookie を検証する（`SID_ENFORCE_MODE=soft` では欠落/Redis不達を許容）
3. `CSRF_VALIDATE`: `csrf_token` Cookie と `X-CSRF-Token` を照合する
4. `AUTHZ`: `token user -> workspace membership -> tree access` を検証する
5. `VALIDATE_REQUEST`: `request_id` / `act_type` / 各種上限を検証する
6. 冪等チェック: `(uid, workspace_id, request_id)` で重複判定する
7. `LOAD_CONTEXT` -> `BUILD_PROMPT` -> `GENERATE_WITH_MODEL`
8. `NORMALIZE_PATCH_OPS`: `upsert` / `append_md` のみへ正規化する
9. `EMIT_STREAM`: 親->子順で patch と text/thought を送信する
10. `FINALIZE`: `done=true` を1回返して終了する

## 異常フロー（エラーコード・retryable）

* token不正 / provider不一致: `UNAUTHENTICATED`, `retryable=false`, `stage=AUTHN`
* sid不正（strict）: `UNAUTHENTICATED`, `retryable=true`, `stage=SID_VALIDATE`
* csrf不一致: `PERMISSION_DENIED`, `retryable=false`, `stage=CSRF_VALIDATE`
* workspace未所属 / tree越境: `PERMISSION_DENIED`, `retryable=false`, `stage=AUTHZ`
* 入力不正: `INVALID_ARGUMENT`, `retryable=false`, `stage=VALIDATE_REQUEST`
* `request_id` 重複: `ALREADY_EXISTS`, `retryable=false`, `stage=VALIDATE_REQUEST`
* Redis不達（soft）: 処理継続、警告ログのみ（degrade）
* モデル/外部依存障害: `UNAVAILABLE` または `DEADLINE_EXCEEDED`, `retryable=true`, `stage=GENERATE_WITH_MODEL`
* 部分送信後の破綻: `error` で終端し、`done` は返さない

## 数値パラメータ

* `request_timeout_ms=90000`
* `model_max_retries=2`
* `reasoning_loop_max=5`
* `patch_ops_max_per_run=400`
* `anchor_node_ids_max=40`
* `context_node_ids_max=120`
* `thought_flush_interval_ms=500`
* `idempotency_key_ttl_ms=900000`

## 受け入れ条件（DoD）

* 正常入力で `done=true` まで返る
* `request_id` 重複で `ALREADY_EXISTS` を返す
* sid不達時の soft/strict 差が仕様どおりに再現できる
* `ErrorInfo.stage/retryable/trace_id` が全失敗ケースで埋まる
* thought有効時に `stream_parts[].thought=true` が1回以上返る

## 実装メモ（最小）

* sid は常に Cookie 正本を優先し、request `sid` は互換用途のみに使う
* Go実装は goroutine/chan で stream backpressure を制御する
* Deep Research timeout時は `GEMINI_3_FLASH` フォールバックを設定で許可する
