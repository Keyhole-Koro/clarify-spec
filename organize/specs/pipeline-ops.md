# Organize 運用仕様（状態管理 / 監視）

## 1. Firestore 状態フィールドの整理（status衝突を防ぐ）

`bundles/{bundleId}` の `status` が “created/described/applied” で衝突しやすいので、分離を仕様化する。

### bundles/{bundleId} 推奨フィールド

* `bundleStatus`: `created|applied|error`
* `descStatus`: `pending|described|error`
* `appliedAt`: timestamp（CleanerがCASに使う）
* `descRef`:（A6）
* `sourceDraftVersion` / `sourceIdempotencyKey`（Bundler冪等）

---

## 2. 監視・運用（最低限）

* DLQに落ちたメッセージは `idempotencyKey` と `traceId` を軸に原因追跡
* 各Agentはログに必ず出す：

  * `traceId`, `idempotencyKey`, `workspaceId`, `type`, (outlineId/nodeId/bundleId)
* 「CAS失敗」はエラーではなく正常系としてカウント（競合吸収）

---

## 3. この仕様での「矛盾が起きない」ポイント（要点）

* type分岐は attributes.type に一本化 → filterが確実に動く
* 同じ仕事は event_ledger で“1回”に収束
* 同一エンティティ更新は lease + CAS → Draft/Outline/MindTreeの整合が崩れない
* 遅延・古いイベントは version/watermark で無害化
* `bundle.created` の fan-out（A6/A3）は subscription分離で自然に成立
* bundles.status衝突を分離して更新競合を回避

---
