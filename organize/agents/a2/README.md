# A2 RouterAgent 仕様

## 1. 責務

* `atom.created` を受け取り、適切な Topic の Draft に追記して `draft.updated` を発火する

## 2. I/O

* Input: `atom.created`
* Output: `workspaces/{workspaceId}/topics/{topicId}`（`latestDraftVersion` 更新）, `mind/drafts/{topicId}/v{n}.md`
* Emit: `draft.updated`

## 3. LLM モデル

* **LLM 不使用** — ルールベースまたは embedding 類似度による決定論的ルーティング

## 4. ルーティング戦略

MVP では `topicId` は Envelope に含まれている（入力時にユーザーが指定済み）ため、ルーティングは検証のみ行う。

将来的な自動ルーティング時の判定基準:

1. Atom の `entities` を取得する
2. 既存 topic それぞれの outline / nodes から entity 集合を取得する
3. Jaccard 類似度 or embedding cosine 類似度を計算する
4. 閾値 0.3 以上の topic に追記する
5. どの topic にも該当しない場合は新規 topic 作成を提案する

## 5. Draft 追記ルール

1. 現在の `latestDraftVersion` を Firestore から取得する
2. CAS で version を +1 して進める
3. 新しい draft を Markdown として生成する（追記分のみ記載）
4. GCS に保存する: `mind/drafts/{topicId}/v{n}.md`

## 6. Draft Markdown フォーマット

Draft の1バージョンは、その版で追記された Atom 群だけを含む差分ドキュメントとする。

各 Atom エントリは以下の構造で記載する:

* 見出し: Atom の `title`（`kind` と `confidence` を付記）
* 本文: `claim` の全文
* メタデータ: `sourceInputId` と `entities` の列挙

過去の Draft 内容は前バージョンの GCS オブジェクトを参照する（累積しない）。

## 7. Idempotency / 競合対策

* ledger: `type:atom.created/inputId:{inputId}`
* lease: `topic:{topicId}`
* CAS: `latestDraft.version == expectedPrevVersion`
