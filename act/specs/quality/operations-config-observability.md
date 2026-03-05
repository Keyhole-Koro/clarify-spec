# 運用設定仕様（Config / Logging / Metrics）

## 目的

当日運用で "動く/壊れた" を即時判定し、復旧時間を短縮する。

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

## ログ要件（MUST）

必須フィールド:

* `traceId`
* `treeId`
* `uid`
* `actType`
* `stage`
* `latencyMs`
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

## アラート指標（SHOULD）

* error率急増
* p95 latency悪化
* timeout増加

## リリース設定（SHOULD）

* concurrency上限: 40
* request timeout: 120秒
* memory: 2Gi
* min instances: 0
* max instances: 20
* rollback条件の事前定義

## 障害時オペレーション

1. traceIdでログ検索
2. `INVALID_ARGUMENT` と `INTERNAL` を切り分け
3. 外部依存障害か内部障害か判定
4. 必要なら即時ロールバック

## 完了条件（DoD）

* 環境差分が文書化済み
* 成功率・遅延を可視化できる
* 当日運用手順を担当者が共有済み
* 数値正本 `act/specs/quality/backend-parameter-index.md` と整合している
