# Organize パイプライン仕様（要約）

Version: 1.1 / schemaVersion: v1

この文書は要約版。詳細は正本を参照。

* `specs/shared/topic-model.md`
* `specs/shared/context-assembly-core.md`
* `organize/specs/pipeline-core.md`
* `organize/specs/pipeline-agents.md`
* `organize/specs/pipeline-ops.md`

## 1. 方針

* Topic は 1 本: `mind-events`
* ルーティングは `attributes.type`
* Pub/Sub は at-least-once
* 重複・競合は `event_ledger + lease + version(CAS)` で無害化
* fan-out は Subscription 複数化で実現
* 知識正本キーは `topic_id`

## 2. Topic / Subscription

* Main Topic: `mind-events`
* DLQ Topic: `mind-events-dlq`

| Agent | Subscription | Filter | Ordering |
| --- | --- | --- | --- |
| A0 MediaInterpreter | `sub-a0` | `attributes.type="media.received"` | OFF |
| A1 Atomizer | `sub-a1` | `attributes.type="input.received"` | OFF |
| A2 Router | `sub-a2` | `attributes.type="atom.created"` | ON（`topicId`） |
| A3b Bundler | `sub-a3b` | `attributes.type="draft.updated"` | ON（`topicId`） |
| A6 BundleDesc | `sub-a6` | `attributes.type="bundle.created"` | OFF |
| A3 Cleaner | `sub-a3` | `attributes.type="bundle.created"` | ON（`topicId`） |
| A4 Indexer | `sub-a4` | `attributes.type="outline.updated"` | ON（`topicId`） |
| A7 Rollup | `sub-a7` | `attributes.type="mindtree.node_changed" OR attributes.type="node.rollup_requested"` | ON（`nodeId`） |
| A5 Balancer | `sub-a5` | `attributes.type="mindtree.metrics.updated"` | OFF |

## 3. Envelope / Attributes

Envelope 必須:

* `schemaVersion`
* `type`
* `traceId`
* `workspaceId`
* `topicId`
* `idempotencyKey`
* `emittedAt`
* `payload`

Pub/Sub attributes 必須:

* `type`
* `schemaVersion`
* `workspaceId`
* `topicId`

推奨 attributes:

* `nodeId`
* `inputId`
* `bundleId`
* `draftVersion`
* `outlineVersion`

## 4. 競合対策（必須）

* Idempotency: `idempotencyKey` は「この仕事1回」を表す
* Ledger: `event_ledger/{sha256(idempotencyKey)}`
* Lease: `leases/{resourceKey}`（例: `topic:tp_1`, `node:nd_1`）
* CAS: `draftVersion` / `outlineVersion` / `generation` / `version` を一致確認
* 古いイベント: 現在版より古ければ ACK して skip

## 5. Ack / Retry / DLQ

* 成功時のみ ACK
* 一時エラーは nack/未ack で再試行
* 永続エラーは ledger を `failed` 記録して DLQ
* 推奨設定: `maxDeliveryAttempts=10`（重処理は20）
