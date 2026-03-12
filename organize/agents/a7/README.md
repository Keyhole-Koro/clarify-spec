# A7 NodeRollupAgent 仕様

## 1. 責務

* `topic.node_changed` / `node.rollup_requested` でノード要約を更新

## 2. I/O

* Input: `topic.node_changed` or `node.rollup_requested`
* Output: `mind/node_rollup/{nodeId}/v{n}.html`, `topics/{topicId}/nodes/{nodeId}.rollupRef`, `rollupWatermark`
* Emit: `node.rollup_updated`（任意）

## 3. Idempotency / 競合対策

* ledger: `type:topic.node_changed/nodeId:{nodeId}/generation:{gen}` 推奨
* lease: `node:{nodeId}`
* watermark以下の古い要求は skip + ACK

## 4. 未決定事項

* rollupトリガー条件
* 再要約の頻度上限
* 子ノード集約ルール
