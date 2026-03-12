# A4 IndexerAgent 仕様

## 1. 責務

* `outline.updated` から Index/Map を再構成

## 2. I/O

* Input: `outline.updated`
* Output: `index_items/*`, `mind/maps/{topicId}/v{n}.md`, `topics/{topicId}.latestMap`
* Emit: なし（任意）

## 3. Idempotency / 競合対策

* ledger: `type:outline.updated/topicId:{topicId}/outlineVersion:{outlineVersion}`
* lease: `topic:{topicId}`
* 古い版は skip + ACK

## 4. 未決定事項

* indexスキーマ（keyword/vector/hybrid）
* map更新頻度
* 全量再計算 vs 差分更新
