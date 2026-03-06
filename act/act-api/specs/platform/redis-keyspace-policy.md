# Redis Keyspace Policy（Act API）

## 目的

sid・ロック・レート制限・冪等キーの衝突を防ぎ、運用時の切り分けを容易にする。

## スコープ / 非スコープ

* スコープ: key命名、TTL、用途、削除ポリシー
* 非スコープ: Redisクラスタ構成

## 前提・依存

* `act/act-api/specs/platform/cloudrun-redis-topology.md`
* `act/act-api/specs/reliability/idempotency-policy.md`
* `act/specs/quality/backend-parameter-index.md`

## 契約（I/O）

入力:

* `sid`, `uid`, `workspace_id`, `request_id`, `trace_id`

出力:

* key hit/miss
* lock acquired/rejected
* rate gate allow/reject

## Key 設計（固定）

* sid:
  * `sid:{sid}` -> user/session metadata
* request lock:
  * `lock:req:{uid}:{workspace_id}:{request_id}`
* idempotency:
  * `idem:{uid}:{workspace_id}:{request_id}`
* rate uid:
  * `rate:uid:{uid}:{epoch_second}`
* rate workspace:
  * `rate:ws:{workspace_id}:{epoch_second}`

## TTL（固定）

* `sid:{sid}`: 86400s
* `lock:req:*`: 10s
* `idem:*`: 900s
* `rate:*`: 3s

## 正常フロー

1. sid 検証キー照会
2. request lock 取得
3. rate gate 判定
4. idempotency 判定
5. RunAct 終了時に必要なキーを更新

## 異常フロー（error/retryable/stage）

* lock競合: `RESOURCE_EXHAUSTED`, `retryable=true`, `stage=SID_VALIDATE`
* rate超過: `RESOURCE_EXHAUSTED`, `retryable=true`, `stage=VALIDATE_REQUEST`
* keyspace不整合: `INTERNAL`, `retryable=true`, `stage=SID_VALIDATE`

## 数値パラメータ

* `sid_ttl_seconds=86400`
* `sid_lock_ttl_seconds=10`
* `sid_req_ttl_seconds=900`
* `retry_after_seconds=3`

## 受け入れ条件（DoD）

* key prefix が衝突しない
* lock / rate / idempotency を独立に観測できる
* TTL が parameter index と一致する
