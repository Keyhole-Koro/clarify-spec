# A2 RouterAgent 仕様

## 1. 責務

* `atom.created` を Draft に追記し `draft.updated` を発火

## 2. I/O

* Input: `atom.created`
* Output: `topics/{topicId}/drafts/{draftVersion}`, `mind/drafts/{topicId}/v{n}.md`
* Emit: `draft.updated`

## 3. Idempotency / 競合対策

* ledger: `type:atom.created/inputId:{inputId}`
* lease: `topic:{topicId}`
* CAS: `latestDraft.version == expectedPrevVersion`

## 4. 未決定事項

* どの topic にルーティングするかの判定戦略
* 追加位置（末尾/挿入）
* context window 切り詰め方
