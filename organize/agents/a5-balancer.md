# A5 BalancerAgent 仕様

## 1. 責務

* `topic.metrics.updated` を受けて偏り是正の Operation を生成

## 2. I/O

* Input: `topic.metrics.updated`（または定期実行）
* Output: `organizeOps/{opId}`
* Emit: `topic.node_changed`, `topic.metrics.updated`

## 3. Idempotency / 競合対策

* 変更適用時 lease 推奨（outline or scopeNode）
* generation CAS 推奨

## 4. 未決定事項

* 偏り指標の定義
* 自動是正の上限
* 人手レビュー閾値
