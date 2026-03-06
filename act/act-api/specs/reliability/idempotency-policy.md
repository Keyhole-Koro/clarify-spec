# Idempotency Policy（RunAct）

## 目的

`request_id` 再送時の挙動を固定し、重複実行や二重課金リスクを回避する。

## スコープ / 非スコープ

* スコープ: 保存先、キー構造、再送時レスポンス、TTL
* 非スコープ: 永続分析データ

## 前提・依存

* `act/specs/contracts/rpc-connect-schema.md`
* `act/act-api/specs/runtime/runact-implementation.md`
* `act/specs/quality/backend-parameter-index.md`

## 契約（I/O）

入力:

* `uid`, `workspace_id`, `request_id`

出力:

* `MISS`: 新規実行
* `IN_FLIGHT`: `ALREADY_EXISTS` + `retry_after_ms`
* `DONE`: 直近 terminal 結果を再送（same request）

## 正常フロー

1. キー生成: `idem:{uid}:{workspace_id}:{request_id}`
2. Redis `SET NX` で in-flight ロック
3. 実行成功時に terminal snapshot を保存
4. TTL内再送時は snapshot を返却

## 異常フロー（error/retryable/stage）

* in-flight重複: `ALREADY_EXISTS`, `retryable=true`, `stage=IDEMPOTENCY_CHECK`
* snapshot破損: `INTERNAL`, `retryable=true`, `stage=IDEMPOTENCY_CHECK`
* Redis不達（soft）: degrade継続 + 警告ログ
* Redis不達（strict）: `UNAVAILABLE`, `retryable=true`, `stage=IDEMPOTENCY_CHECK`

## 数値パラメータ

* `idempotency_key_ttl_ms=900000`
* `idempotency_inflight_ttl_seconds=120`
* `idempotency_snapshot_ttl_seconds=900`
* `retry_after_ms=3000`

## 受け入れ条件（DoD）

* 同一 `request_id` の二重起動が抑止される
* in-flight と done を区別して返却できる
* 失敗時に `stage=IDEMPOTENCY_CHECK` が付く
