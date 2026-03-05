# A5 BalancerAgent 仕様

## 1. 責務

* `mindtree.metrics.updated` を受けて偏り是正のOperationを生成

## 2. I/O

* Input: `mindtree.metrics.updated`（または定期実行）
* Output: `organizeOps/{opId}`
* Emit: `mindtree.node_changed`, `mindtree.metrics.updated`

## 3. Idempotency / 競合対策

* 変更適用時 lease 推奨（outline or scopeNode）
* generation CAS 推奨

## 4. 未決定事項

* 偏り指標の定義
* 自動是正の上限
* 人手レビュー閾値
