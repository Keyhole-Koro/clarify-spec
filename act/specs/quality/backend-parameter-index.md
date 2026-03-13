# Backend Parameter Index (Act)

バックエンドの細かい数値設定を一元管理する索引。
後日レビュー時はこの文書を起点に確認する。

## Runtime Limits

* `request_timeout_ms`: 90000
* `idempotency_key_ttl_ms`: 900000
* `node_timeout_ms`: 60000
* `model_max_retries`: 2
* `reasoning_loop_max`: 5
* `patch_ops_max_per_run`: 400
* `anchor_node_ids_max`: 40
* `context_node_ids_max`: 120
* `user_message_max_chars`: 6000
* `text_delta_total_max_chars`: 50000
* `patch_batch_max`: 40
* `text_batch_max_chars`: 800
* `stream_recovery_mode`: `reset_buffer` # reconnect時はappendではなく全量置換またはバッファクリアを指示する
* `thought_flush_interval_ms`: 500

## Session / CSRF / Redis

* `sid_ttl_seconds`: 86400
* `sid_req_ttl_seconds`: 900
* `sid_lock_ttl_seconds`: 10
* `csrf_ttl_seconds`: 86400
* `redis_connect_timeout_ms`: 200
* `redis_read_timeout_ms`: 200
* `redis_write_timeout_ms`: 200
* `cors_preflight_max_age_seconds`: 600
* `invite_token_ttl_seconds_default`: 172800
* `invite_token_ttl_seconds_max`: 604800
* `invite_token_single_use`: true
* `internal_auth_token_cache_ttl_seconds`: 300
* `idempotency_inflight_ttl_seconds`: 120
* `idempotency_snapshot_ttl_seconds`: 900

## Load Context Limits

* `focus_nodes_soft_max`: 10
* `focus_nodes_hard_max`: 30
* `load_nodes_max`: 500
* `load_edges_max`: 1000
* `load_evidence_max`: 1000
* `load_hop`: 2

## Deep Research

* `deep_research_poll_interval_ms`: 3000
* `deep_research_max_wait_ms`: 45000
* `deep_research_fallback_5xx_threshold`: 2
* `deep_research_auto_retry`: 0
* `deep_research_fallback_only_once`: true

## Rate Limits

* `run_concurrency_per_uid`: 4
* `run_concurrency_per_workspace`: 60
* `retry_after_seconds`: 3

## Cloud Run Defaults

* `cloudrun_concurrency`: 40
* `cloudrun_timeout_seconds`: 120
* `cloudrun_memory`: `2Gi`
* `cloudrun_min_instances`: 0
* `cloudrun_max_instances`: 20

## Notes

* 変更時は `act/act-api/specs/runtime/runact-implementation.md` と `act/act-adk-worker/specs/act-adk-runtime.md` を同時更新する
* セッション境界を変える場合は `act/act-api/specs/security/session-and-auth-boundary.md` も同時更新する
* 仕様値と実装値が乖離した場合、この文書を先に修正する
* sid 検証は全環境で strict に行う
