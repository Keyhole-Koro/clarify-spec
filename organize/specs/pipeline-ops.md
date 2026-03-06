# Organize 運用仕様（状態管理 / 監視）

## 目的

`topic_id` 基準で運用観測と障害切り分けを行い、競合吸収を安定運用する。

## スコープ / 非スコープ

* スコープ: 状態フィールド、監視キー、runbook
* 非スコープ: 実装監視ツール固有設定

## 前提・依存

* `organize/specs/pipeline-core.md`
* `organize/specs/pipeline-agents.md`
* `firestore/schema.md`

## 契約（I/O）

入力:

* Agent logs/metrics
* Firestore status docs

出力:

* 障害切り分け手順
* topic単位の復旧判断

## 正常フロー

1. 各Agentはログへ `traceId`, `topicId`, `idempotencyKey`, `type` を出力
2. ledger/lease/CAS の結果をメトリクス化
3. `bundle` 系は apply と describe の状態を分離管理

## 異常フロー（error/retryable/stage）

* ledger競合: retry（`retryable=true`）
* CAS不一致: skip/ack（正常吸収）
* topic跨ぎ参照: `FAILED_PRECONDITION`, `retryable=false`

## 数値パラメータ

* DLQ max delivery: 10（重処理20）
* lease TTL: 60〜300秒
* ack deadline: 60〜600秒

## 状態フィールド（推奨）

`bundles/{bundleId}`:

* `bundleStatus`: `created|applied|error`
* `descStatus`: `pending|described|error`
* `topicId`: string
* `sourceDraftVersion`: int
* `appliedAt`: timestamp
* `descRef`: ref

## 監視キー（MUST）

* `traceId`
* `topicId`
* `workspaceId`
* `idempotencyKey`
* `type`
* optional: `nodeId`, `bundleId`, `draftVersion`, `outlineVersion`

## Runbook（topic単位）

1. `topicId` でログを絞る
2. `idempotencyKey` で重複処理有無を確認
3. `bundleStatus/descStatus` を確認
4. `latestDraftVersion` / `latestOutlineVersion` の進みを確認
5. 必要なら DLQ を topic単位で再投入

## 受け入れ条件（DoD）

* topic単位で障害切り分けが完結できる
* CAS失敗が誤ってエラー扱いされない
* bundle status衝突が起きない

## 実装メモ（最小）

* ダッシュボードの第一キーは `topicId`
* `outlineId` ベースの集計は残さない
