# A6 BundleDescriptionAgent 仕様

## 1. 責務

* `bundle.created` を説明HTMLへ変換

## 2. I/O

* Input: `bundle.created`
* Output: `mind/bundle_desc/{bundleId}/v{n}.html`, `bundles/{bundleId}.descRef`
* Emit: `bundle.described`（任意）

## 3. Idempotency / 競合対策

* ledger: `type:bundle.created/bundleId:{bundleId}/purpose:desc`
* `descRef.version` CAS

## 4. 未決定事項

* HTMLテンプレート方針
* 画像埋め込みの扱い
* CSS配信戦略
