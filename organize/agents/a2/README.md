# A2 DraftAppenderAgent 仕様

## 1. 責務

* `topic.resolved` を受け取り、解決済み Topic の Draft に追記して `draft.updated` を発火する

## 2. I/O

* Input: `topic.resolved`
* Output: `workspaces/{workspaceId}/topics/{resolvedTopicId}`（`latestDraftVersion` 更新）, `mind/drafts/{resolvedTopicId}/v{n}.md`
* Emit: `draft.updated`

## 3. LLM モデル

* **LLM 不使用** — topic 解決後の append を決定論的に実行する

## 4. TopicResolver との境界

A2 は topic routing を担当しない。

前段の `TopicResolver` が次を解決済みであることを前提にする:

1. `resolvedTopicId`
2. `resolutionMode` (`existing | new`)
3. `resolutionConfidence`
4. `resolutionReason`

## 5. Draft 追記ルール

1. 現在の `latestDraftVersion` を Firestore から取得する
2. CAS で version を +1 して進める
3. 新しい draft を Markdown として生成する（追記分のみ記載）
4. GCS に保存する: `mind/drafts/{resolvedTopicId}/v{n}.md`

## 6. Draft Markdown フォーマット

Draft の1バージョンは、その版で追記された Atom 群だけを含む差分ドキュメントとする。

各 Atom エントリは以下の構造で記載する:

* 見出し: Atom の `title`（`kind` と `confidence` を付記）
* 本文: `claim` の全文
* メタデータ: `sourceInputId` と `entities` の列挙

過去の Draft 内容は前バージョンの GCS オブジェクトを参照する（累積しない）。

## 7. Idempotency / 競合対策

* ledger: `type:topic.resolved/inputId:{inputId}/topicId:{resolvedTopicId}`
* lease: `topic:{resolvedTopicId}`
* CAS: `latestDraft.version == expectedPrevVersion`
