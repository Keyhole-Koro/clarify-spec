# Firestore Indexes (Topic-Centric)

`firestore/schema.md` に基づく推奨インデックス定義。

## Composite Index Candidates

1. `workspaces/{workspaceId}/topics`
* fields: `updated_at desc`

2. `workspaces/{workspaceId}/topics/{topicId}/nodes`
* fields: `updated_at desc`

3. `workspaces/{workspaceId}/topics/{topicId}/edges`
* fields: `source_id asc`, `order_key asc`

4. `workspaces/{workspaceId}/topics/{topicId}/actRuns`
* fields: `status asc`, `started_at desc`

5. `workspaces/{workspaceId}/topics/{topicId}/drafts`
* fields: `version desc`

6. `workspaces/{workspaceId}/topics/{topicId}/outlines`
* fields: `version desc`

7. `workspaces/{workspaceId}/eventLedger`
* fields: `idempotency_key_hash asc`, `topic_id asc`

## Notes

* 実際の `firestore.indexes.json` 作成時は collection group 設定に合わせて調整
* 旧 tree/outlineId 基準のインデックスは新規追加しない
