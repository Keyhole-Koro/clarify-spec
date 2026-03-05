# Organize パイプライン中核仕様（Pub/Sub + Firestore + GCS）

Version: 1.0 / schemaVersion: v1

## 0. 目的

本仕様は、A0〜A7/A5 のイベント駆動パイプラインを **Pub/Sub (at-least-once)** 上で安全に運用するために以下を定義する。

* Topic / Subscription の構成（1 topic + type分岐）
* メッセージ（Envelope）と attributes の仕様
* 競合対策：**Idempotency（冪等） / Lease（分散ロック） / Version(CAS)（楽観ロック）**
* DLQ / retry / ordering の運用規約
* 各Agentの入力/出力/emit と「必須の競合対策」

---

## 1. 基本方針（超重要）

1. Pub/Sub は **at-least-once**：重複配信・遅延・順序入れ替わりを前提とする
2. 二重処理を「防ぐ」ではなく「無害化」する：

   * **event_ledger（処理台帳）**で“同じ仕事は1回”を保証
   * Firestore の **status/sha256/generation/version** で二重処理を吸収
3. 競合は二段で対処：

   * **Lease（短期ロック）**：同一outline/nodeの同時更新を抑止
   * **Version(CAS)**：古い版・競合更新を確実に弾く（成功/失敗が判定可能）
4. Pub/Sub ルーティングは **Subscription filter（attributes.type）**で行う
5. fan-out（1イベントを複数Agentが消費）は **Subscriptionを複数作る**ことで実現（Topicは増やさない）

---

## 2. Pub/Sub リソース仕様

### 2.1 Topic

* `mind-events`（環境別に `mind-events-dev/stg/prod` 推奨）
* DLQ用 Topic：`mind-events-dlq`（同様に環境別推奨）

### 2.2 Subscription（Agent単位）

Subscription名は `sub-{agent}` を基本とする。

| Agent               | Subscription | Filter（必須）                                                                           | Ordering推奨            |
| ------------------- | ------------ | ------------------------------------------------------------------------------------ | --------------------- |
| A0 MediaInterpreter | sub-a0       | `attributes.type="media.received"`                                                   | OFF                   |
| A1 Atomizer         | sub-a1       | `attributes.type="input.received"`                                                   | OFF                   |
| A2 Router           | sub-a2       | `attributes.type="atom.created"`                                                     | **ON（outlineId）推奨**   |
| A3b Bundler         | sub-a3b      | `attributes.type="draft.updated"`                                                    | **ON（outlineId）必須推奨** |
| A6 BundleDesc       | sub-a6       | `attributes.type="bundle.created"`                                                   | OFF                   |
| A3 Cleaner          | sub-a3       | `attributes.type="bundle.created"`                                                   | **ON（outlineId）推奨**   |
| A4 Indexer          | sub-a4       | `attributes.type="outline.updated"`                                                  | **ON（outlineId）推奨**   |
| A7 Rollup           | sub-a7       | `attributes.type="mindtree.node_changed" OR attributes.type="node.rollup_requested"` | **ON（nodeId）推奨**      |
| A5 Balancer         | sub-a5       | `attributes.type="mindtree.metrics.updated"`                                         | OFF                   |

> `bundle.created` は A6 と A3 が両方読む（fan-out）ため、**sub-a6 と sub-a3 を別々に持つ**。

---

## 3. メッセージ仕様（Envelope + Attributes）

### 3.1 Envelope（本文JSON）※全イベント共通

```json
{
  "schemaVersion": "v1",
  "type": "input.received",
  "traceId": "t_xxx",
  "workspaceId": "ws_xxx",
  "uid": "u_xxx",

  "idempotencyKey": "type:input.received/inputId:in_123",
  "emittedAt": "2026-03-05T12:34:56Z",

  "payload": {}
}
```

#### 必須フィールド

* `schemaVersion`, `type`, `traceId`, `workspaceId`, `idempotencyKey`, `emittedAt`, `payload`
* `uid` は存在するなら入れる（ユーザ起点の追跡が必要なため）

### 3.2 Pub/Sub message attributes（必須）

Subscription filter の基準は **attributes** とするため、publish時に必ず複製する。

#### MUST attributes

* `type`：Envelope.type と同値
* `schemaVersion`：Envelope.schemaVersion と同値
* `workspaceId`：Envelope.workspaceId と同値

#### SHOULD attributes（存在する場合）

* `outlineId`：outlineに紐づくイベントで付与
* `nodeId`：nodeに紐づくイベントで付与
* `inputId`, `bundleId`：該当する場合は付与（運用・監視が楽）

---

## 4. 参照型（共通）

```json
{
  "GcsRef": { "gcsUri": "gs://...", "sha256": "hex", "version": 12, "mimeType": "text/markdown" },
  "SpanRef": { "kind": "char|line|slack_message", "start": 120, "end": 240, "messageId": "..." }
}
```

---

## 5. 競合対策（Idempotency / Ledger / Lease / Version）

### 5.1 IdempotencyKey（MUST）

`idempotencyKey` は **「この仕事1回」** を表す一意キーであること。

推奨パターン（versionがある場合は含める）：

* `type:{type}/{primaryKey}:{id}`
* 例：

  * `type:input.received/inputId:in_123`
  * `type:draft.updated/outlineId:ol_1/draftVersion:12`
  * `type:bundle.created/bundleId:bd_123`
  * `type:outline.updated/outlineId:ol_1/outlineVersion:21`

---

### 5.2 event_ledger（処理台帳）— MUST（強い推奨ではなく必須にする）

重複配信・並行ワーカー・途中死を吸収するため、各Agentは **処理開始前に ledger を確保**する。

### コレクション

* `event_ledger/{ledgerId}`
* `ledgerId` は `sha256(idempotencyKey)` 等の短縮を推奨

### ドキュメント例

```json
{
  "idempotencyKey": "...",
  "agent": "A3b",
  "status": "started|succeeded|failed",
  "traceId": "t_xxx",
  "startedAt": "...",
  "finishedAt": "...",
  "result": { "kind": "bundle", "id": "bd_123" },
  "error": { "code": "...", "message": "..." }
}
```

### 取得ルール（MUST）

* トランザクションで以下を原子的に行う：

  * ledgerが存在しない → 作成（status=started）して処理続行
  * status=succeeded → **即ACKしてスキップ**
  * status=started かつ startedAt が新しい → **別ワーカー処理中**（nack/未ackでリトライ）
  * status=failed → 方針次第（再試行 or DLQ）。デフォルトは **再試行**（一時エラー想定）

> これで「同じ idempotencyKey を同時に2人が処理」は起こらない。

---

### 5.3 Lease（分散ロック）— MUST（outline/node更新があるAgent）

Draft/Outline/MindTree/Index/Rollupのように「同一エンティティ更新」を伴うAgentは lease を使う。

### コレクション

* `leases/{resourceKey}`
* resourceKey例：`outline:ol_1`, `node:nd_777`

### ドキュメント例

```json
{
  "owner": "worker-abc",
  "leaseUntil": "2026-03-05T12:35:56Z",
  "traceId": "t_xxx",
  "updatedAt": "..."
}
```

### 取得ルール（MUST）

* Firestore TransactionでCAS：

  * leaseがない OR `leaseUntil < now` → 取得して続行
  * それ以外 → 取得失敗 → nack/未ackでリトライ

### TTL（SHOULD）

* 60〜300秒程度（処理時間に合わせる）
* 長い処理は renew（延長）してよい

---

### 5.4 Version(CAS)（楽観ロック）— MUST（世代を持つ書き込み）

書き込みは必ず以下のいずれかで **古い/競合更新を弾く**：

* `draftVersion`（drafts/outlinesのGCS v{n}と一致）
* `outlineVersion`（outlinesのGCS v{n}と一致）
* `generation`（mindtree_nodes）
* `version`（rollupRef / descRef）

### CASルール（MUST）

更新は transaction で

* `currentVersion == expectedVersion` を確認してから更新
* 不一致なら **ACKしてスキップ**（既に先に進んだのが正常）

---

### 5.5 遅延/古いイベントの扱い（MUST）

* version付きイベントは、現在状態より古い場合 **スキップ**してACK

  * `draft.updated`：payload.draftVersion < drafts.latestDraft.version → skip
  * `outline.updated`：payload.outlineVersion < outlines.latestOutline.version → skip

---

## 6. DLQ / Retry / Ack ルール

### 6.1 共通ルール（MUST）

* **成功時のみACK**
* 一時エラー（外部API、タイムアウト等）：nack/未ackで再試行
* 永続エラー（入力不正、参照欠損など）：明示的に failed を ledger に記録し、最終的に DLQ へ

### 6.2 Subscription設定（SHOULD）

* Dead Letter Topic：`mind-events-dlq`
* maxDeliveryAttempts：10（重いAgentは20も可）
* ackDeadline：処理時間に合わせる（60〜600秒）

---
