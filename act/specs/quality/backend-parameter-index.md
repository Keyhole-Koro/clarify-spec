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
* `thought_flush_interval_ms`: 500

## Load Context Limits

* `load_nodes_max`: 500
* `load_edges_max`: 1000
* `load_evidence_max`: 1000
* `load_hop`: 2

## Deep Research

* `deep_research_poll_interval_ms`: 3000
* `deep_research_max_wait_ms`: 45000
* `deep_research_fallback_5xx_threshold`: 2

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

* 変更時は `runact-implementation.md` と `act-langgraph-runtime.md` を同時更新する
* 仕様値と実装値が乖離した場合、この文書を先に修正する
