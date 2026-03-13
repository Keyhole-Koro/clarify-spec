# 運用設定仕様（Config / Logging / Metrics）

## 目的

当日運用で「動く/壊れた」を即時判定し、復旧時間を短縮する。

## スコープ / 非スコープ

* スコープ: Act backend（Cloud Run）での設定、可観測性、切り戻し
* 非スコープ: 長期SLO設計、BI分析

## 前提 / 参照

* `act/specs/quality/backend-parameter-index.md`
* `act/act-api/specs/security/session-and-auth-boundary.md`
* `act/act-api/specs/platform/cloudrun-redis-topology.md`

## 契約（I/O）

入力:

* Cloud Run 環境変数（LLM/Redis/Session/CSRF）
* 構造化ログとメトリクス出力

出力:

* traceで追跡可能なログ
* stage別の失敗可視化
* リリース可否判断に使えるメトリクス

## 処理フロー（正常）

1. リリース時に parameter index と同値を環境変数へ反映する
2. すべての `RunAct` request に `traceId` を付与してログへ出力する
3. 成功/失敗メトリクスを stage 単位で記録する
4. Redis/sid/csrf 系の検証結果を専用メトリクスへ記録する

## 異常フロー（エラーコード・retryable）

* `stage=AUTHN/AUTHZ/CSRF_VALIDATE`: `retryable=false` を基本とする
* `stage=SID_VALIDATE` で Redis 不達: fail-closed で `UNAVAILABLE` を返し、復旧手順へ入る
* `stage=GENERATE_WITH_MODEL` の外部障害: `retryable=true` で返す
* `stage=LOAD_CONTEXT` の Firestore 障害: `retryable=true` を基本とする

## 環境変数（MUST）

* `RPC_BASE_URL`
* `VERTEX_PROJECT_ID`
* `VERTEX_LOCATION`
* `LLM_PROVIDER`
* `LLM_MODEL`
* `LLM_MODEL_FLASH`
* `LLM_MODEL_DEEP_RESEARCH`
* `VERTEX_USE_GROUNDING`
* `VERTEX_INTERACTIONS_AGENT`
* `ACT_TIMEOUT_MS`
* `ACT_MAX_RETRIES`
* `LOG_LEVEL`
* `REDIS_HOST`
* `REDIS_PORT`
* `REDIS_DB`
* `SID_TTL_SECONDS`
* `SID_REQ_TTL_SECONDS`
* `SID_LOCK_TTL_SECONDS`
* `CSRF_TTL_SECONDS`

## ログ要件（MUST）

必須フィールド:

* `traceId`
* `requestId`
* `workspaceId`
* `treeId`
* `uid`
* `actType`
* `stage`
* `latencyMs`
* `redisAvailable`（`true|false`）
* `error.code`（失敗時）

## メトリクス要件（MUST）

* `run_act_total`
* `run_act_error_total`
* `run_act_latency_ms`
* `emitted_patch_ops_total`
* `emitted_text_chars_total`
* `run_act_grounded_total`
* `run_act_profile_total{profile}`
* `run_act_thought_tokens_total`
* `run_act_deep_research_total`
* `sid_validate_total`
* `sid_validate_fail_total`
* `csrf_validate_fail_total`
* `redis_cmd_latency_ms`
* `redis_unavailable_total`
* `sid_recovery_attempt_total`

## 数値パラメータ

* `cloudrun_concurrency=40`
* `cloudrun_timeout_seconds=120`
* `cloudrun_memory=2Gi`
* `cloudrun_min_instances=0`
* `cloudrun_max_instances=20`
* sid 検証は全環境で strict

## 受け入れ条件（DoD）

* Redis停止時は fail-closed で `UNAVAILABLE` になり、復旧フローへ入る
* `ErrorInfo.stage/retryable/trace_id` がログと一致する
* sid/csrf 系メトリクスがダッシュボードで確認できる
* 数値正本 `act/specs/quality/backend-parameter-index.md` と整合している

## 実装メモ（最小）

* alertは `run_act_error_total` と `redis_unavailable_total` の急増を優先
* rollback条件を「5分連続で成功率閾値未達」に固定しておく

## 復旧フロー（Redis / sid）

1. `redis_unavailable_total` の急増または `stage=SID_VALIDATE` の `UNAVAILABLE` を検知する
2. Redis / VPC / Cloud Run 接続を復旧する
3. sid read/write ヘルスチェックが成功することを確認する
4. 必要ならユーザーに再ログインまたはセッション再発行を促す
5. frontend は再試行導線を出し、ユーザー操作で `RunAct` を再送する
