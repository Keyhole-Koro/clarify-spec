# A3 CleanerAgent 仕様

## 1. 責務

* `bundle.created` を Outline/MindTree に反映

## 2. I/O

* Input: `bundle.created`
* Output: `outlines/{outlineId}`, `mindtree_nodes/*`, `mindtree_edges/*`
* Emit: `outline.updated`, `mindtree.node_changed`, `atom.reissued`

## 3. Idempotency / 競合対策

* ledger: `type:bundle.created/bundleId:{bundleId}/purpose:apply`
* lease: `outline:{outlineId}`
* bundle二重適用防止（`appliedAt` CAS など）
* outlineVersion CAS

## 4. 未決定事項

* merge/rewrite戦略
* edge upsert key 設計
* `atom.reissued` の閾値
