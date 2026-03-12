# A3 CleanerAgent 仕様

## 1. 責務

* `bundle.created` を Outline と topic graph に反映

## 2. I/O

* Input: `bundle.created`
* Output: `topics/{topicId}/outlines/{outlineVersion}`, `topics/{topicId}/nodes/*`, `topics/{topicId}/edges/*`
* Emit: `outline.updated`, `topic.node_changed`, `atom.reissued`

## 3. 処理段階

1. `bundleId` から蒸留済み claim 群と current `schema_version` を取得する
2. claim を normalize し、node候補 / edge候補 / unresolved候補へ分解する
3. 既存 `topics/{topicId}/nodes/*` と照合して entity resolution を行う
4. merge / create / reject を判定し、必要な node upsert を確定する
5. relation を `topics/{topicId}/edges/*` に upsert する
6. graph 反映結果から outline を再構成し、新しい `outlineVersion` を発行する
7. 変更 node ごとに `topic.node_changed` を emit する
8. schema不整合や確定不能な候補は `atom.reissued` で再評価へ戻す

### 判定責務

* 新規 node を作るか、既存 node に merge するかを決める
* 親子関係にするか、関連 relation にするかを決める
* outline に昇格できる確度かを決める
* current schema で表現不能なら再評価に戻す
* 変更 downstream が必要な node を選ぶ

## 4. Idempotency / 競合対策

* ledger: `type:bundle.created/bundleId:{bundleId}/purpose:apply`
* lease: `topic:{topicId}`
* bundle二重適用防止（`appliedAt` CAS など）
* outlineVersion CAS

## 5. 未決定事項

* merge/rewrite戦略
* edge upsert key 設計
* `atom.reissued` の閾値
