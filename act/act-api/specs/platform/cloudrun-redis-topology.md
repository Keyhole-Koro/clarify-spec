# Cloud Run Redis Topology 仕様

## 目的

Cloud Run と Redis（Memorystore）の接続構成と運用境界を固定し、当日のインフラ接続ミスを防ぐ。

## スコープ / 非スコープ

* スコープ: Cloud Run -> VPC -> Memorystore の接続、障害時挙動
* 非スコープ: Terraform/IaC 実装詳細

## 前提 / 参照

* `act/act-api/specs/runtime/connect-server.md`
* `act/act-api/specs/platform/redis-keyspace-policy.md`
* `act/specs/quality/backend-parameter-index.md`
* `act/specs/quality/operations-config-observability.md`

## 契約（I/O）

入力:

* `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`
* `SID_TTL_SECONDS`, `SID_REQ_TTL_SECONDS`, `SID_LOCK_TTL_SECONDS`

出力:

* sid検証結果（ok / reject）
* redis関連メトリクスとログ

## 処理フロー（正常）

1. Cloud Run が Serverless VPC Access で private network 接続
2. Memorystore Redis へ接続し sidキーを検証
3. sid / request lock / rate gate を必要に応じて評価
4. handler実行後に last_seen 更新

## 異常フロー（エラーコード・retryable）

* Redis接続失敗: `UNAVAILABLE`, `retryable=true`, `stage=SID_VALIDATE`
* Lock取得失敗: `RESOURCE_EXHAUSTED`, `retry_after_ms` 付与

## 数値パラメータ

* `cloudrun_concurrency=40`
* `cloudrun_timeout_seconds=120`
* `cloudrun_memory=2Gi`
* `cloudrun_min_instances=0`
* `cloudrun_max_instances=20`
* `SID_REQ_TTL_SECONDS=900`
* `SID_LOCK_TTL_SECONDS=10`

## 受け入れ条件（DoD）

* Redis不達時の fail-closed 動作が明記されている
* Cloud Run 側の必要設定が仕様化されている
* 運用監視に必要なメトリクス名が仕様化されている

## 実装メモ（最小）

* Redis 復旧後は sid read/write ヘルスチェック成功を確認してから RunAct を再開する
* 接続タイムアウトは短く（100-300ms）して障害波及を避ける
