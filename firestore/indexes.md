# Firestore Indexes (Draft)

`firestore/schema.md` に基づく推奨インデックス定義メモ。

## Composite Index Candidates

1. `workspaces/{workspaceId}/trees`
* fields: `updated_at desc`

2. `workspaces/{workspaceId}/trees/{treeId}/nodes`
* fields: `updated_at desc`

3. `workspaces/{workspaceId}/trees/{treeId}/edges`
* fields: `source_id asc`, `order_key asc`

4. `workspaces/{workspaceId}/actRequests`
* fields: `uid asc`, `created_at desc`

5. `workspaces/{workspaceId}/actRuns`
* fields: `status asc`, `started_at desc`

6. `workspaces/{workspaceId}/eventLedger`
* fields: `idempotency_key_hash asc`

## Notes

* 実際の `firestore.indexes.json` へ落とす際に collection group / subcollection の設計と合わせて調整する
