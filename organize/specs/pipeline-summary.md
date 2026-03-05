# Organize パイプライン仕様（要約）

Version: 1.0 / schemaVersion: v1

この文書は要約版です。詳細は以下の正本を参照してください。

* `organize/specs/pipeline-core.md`
* `organize/specs/pipeline-agents.md`
* `organize/specs/pipeline-ops.md`

## 1. 方針

* Topic は 1 本: `mind-events`
* ルーティングは `attributes.type` に一本化
* Pub/Sub は at-least-once 前提
* 重複・競合は `event_ledger` + `lease` + `version(CAS)` で無害化
* fan-out は Subscription を複数作って実現（Topic は増やさない）

## 2. Topic / Subscription

* Main Topic: `mind-events`
* DLQ Topic: `mind-events-dlq`

| Agent | Subscription | Filter | Ordering |
| --- | --- | --- | --- |
| A0 MediaInterpreter | `sub-a0` | `attributes.type="media.received"` | OFF |
| A1 Atomizer | `sub-a1` | `attributes.type="input.received"` | OFF |
| A2 Router | `sub-a2` | `attributes.type="atom.created"` | ON推奨（`outlineId`） |
| A3b Bundler | `sub-a3b` | `attributes.type="draft.updated"` | ON推奨（`outlineId`） |
| A6 BundleDesc | `sub-a6` | `attributes.type="bundle.created"` | OFF |
| A3 Cleaner | `sub-a3` | `attributes.type="bundle.created"` | ON推奨（`outlineId`） |
| A4 Indexer | `sub-a4` | `attributes.type="outline.updated"` | ON推奨（`outlineId`） |
| A7 Rollup | `sub-a7` | `attributes.type="mindtree.node_changed" OR attributes.type="node.rollup_requested"` | ON推奨（`nodeId`） |
| A5 Balancer | `sub-a5` | `attributes.type="mindtree.metrics.updated"` | OFF |

## 3. Envelope / Attributes

Envelope の必須項目:

* `schemaVersion`
* `type`
* `traceId`
* `workspaceId`
* `idempotencyKey`
* `emittedAt`
* `payload`

Pub/Sub attributes の必須項目:

* `type`
* `schemaVersion`
* `workspaceId`

推奨 attributes:

* `outlineId`
* `nodeId`
* `inputId`
* `bundleId`

## 4. 競合対策（実装必須）

* Idempotency: `idempotencyKey` は「この仕事1回」を表す
* Ledger: `event_ledger/{sha256(idempotencyKey)}` を開始前に確保
* Lease: `leases/{resourceKey}`（例: `outline:ol_1`, `node:nd_1`）
* CAS: `draftVersion` / `outlineVersion` / `generation` / `version` の一致確認で更新
* 古いイベント: 現在版より古ければ ACK してスキップ

## 5. Ack / Retry / DLQ

* 成功時のみ ACK
* 一時エラーは nack/未ackで再試行
* 永続エラーは ledger を `failed` に記録し、最終的に DLQ
* 推奨設定: `maxDeliveryAttempts=10`（重い処理は 20）

## 6. Agent運用ルール

* `bundle.created` は A6 と A3 が fan-out で購読
* A3 Cleaner と A6 BundleDesc は同じ `bundleId` でも目的別 idempotencyKey を使う
* `mindtree.node_changed` は 1 イベント 1 `nodeId` 推奨
* `bundles` の状態は `bundleStatus` と `descStatus` を分離して衝突回避

## 7. 参照

* エントリ: `organize/specs/pipeline-spec.md`
* 中核仕様: `organize/specs/pipeline-core.md`
* Agent仕様: `organize/specs/pipeline-agents.md`
* 運用仕様: `organize/specs/pipeline-ops.md`
