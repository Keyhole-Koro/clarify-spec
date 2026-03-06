# A3b BundlerAgent 仕様

## 1. 責務

* `draft.updated` 差分から Bundle を作り `bundle.created` を発火

## 2. I/O

* Input: `draft.updated`
* Output: `bundles/{bundleId}`
* Emit: `bundle.created`

## 3. Idempotency / 競合対策

* ledger: `type:draft.updated/topicId:{topicId}/draftVersion:{draftVersion}`
* 推奨 lease: `topic:{topicId}`
* 同一 `(topicId, draftVersion)` の重複 bundle を禁止

## 4. 未決定事項

* Bundleサイズ上限
* bundle分割アルゴリズム
* 下流A3/A6へのpayload最適化
