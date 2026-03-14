# Upload Progress Spec

目的: フロントから upload した input が Organize パイプラインのどこまで進んだかを、人間に分かる形で表示する。

## 1. Source of Truth

* `frontend/frontend-spec.md`
* `firestore/schema.md`
* `organize/specs/pipeline/agents.md`

進捗の正本:

* Firestore `workspaces/{workspaceId}/topics/{topicId}/inputProgress/{inputId}`

フロントは `inputProgress` を snapshot 購読して表示する。
Pub/Sub, event ledger, trace log を直接 UI 正本にしてはならない。

## 2. 対象ユーザー

* upload 後に「今どうなっているか」を知りたい利用者
* 障害時にどこで止まったかを知りたい運用者

## 3. ユーザー向け status

UI は内部 event を直接見せず、次の status に写像する。

* `uploaded`
* `extracting`
* `atomizing`
* `resolving_topic`
* `updating_draft`
* `completed`
* `failed`

補助表示:
* なし

## 4. ステップ定義

### `uploaded`

意味:

* フロントが upload を受理し、backend に処理が渡った

表示文言例:

* 「アップロード済み」
* 「処理を開始しています」

### `extracting`

意味:

* A0 が原文抽出中

表示文言例:

* 「内容を抽出中」

### `atomizing`

意味:

* A1 が atom を生成中

表示文言例:

* 「内容を分解中」

### `resolving_topic`

意味:

* TopicResolver が attach/create_new を判定中

表示文言例:

* 「関連トピックを判定中」

### `updating_draft`

意味:

* A2 / A3b / A6 / A3 により topic へ反映中

表示文言例:

* 「知識を反映中」

### `completed`

意味:

* 少なくとも `outline.updated` まで到達した

表示文言例:

* 「反映完了」
* 「既存トピックに追加しました」
* 「新しいトピックを作成しました」

### `failed`

意味:

* 永続失敗、または運用上 retry 継続ではなく失敗確定とみなす状態

表示文言例:

* 「処理に失敗しました」

## 5. 表示領域

### 5.1 Header

役割:

* 最新 1 件の upload 進捗を簡易表示する

表示項目:

* 現在 status
* stepper 進捗
* 最新更新時刻

非表示でよいもの:

* `traceId`
* `errorCode`
* internal event name

### 5.2 Topic Activity Panel

役割:

* recent uploads を時系列に詳細表示する

表示項目:

* input title または filename
* 現在 status
* 各完了 step
* `completed` 時の attach/create_new 結果
* `resolvedTopicId` に対応する topic title
* `failed` の補助メッセージ

## 6. 完了時の表示

`completed` では次を表示してよい。

* `resolutionMode`
* `resolvedTopicId`
* topic title
* `draft updated` の要約
* `outline updated` の要約

優先順位:

1. topic title
2. `attach_existing | create_new`
3. 反映サマリー

## 7. 失敗の扱い

### `failed`

条件:

* `inputProgress.status == failed`

UI:

* 明確に失敗表示
* retry 導線または debug/support 導線を必須表示

## 8. Firestore 依存項目

最低限必要:

* `status`
* `currentPhase`
* `phaseUpdatedAt`
* `resolvedTopicId`
* `resolutionMode`
* `traceId`
* `errorCode`
* `errorMessage`

## 9. UI に出さないもの

原則として primary 表示しない:

* Pub/Sub event 名
* ledger 状態
* lease 情報
* raw confidence 数値

開発モードのみ補助表示してよい:

* `traceId`
* `lastEventType`
* `currentPhase`

## 10. 固定ルール

* header は既定で最新 1 件だけを表示する
* 長時間処理は異常扱いしない
* `failed` は backend が明示設定した場合にのみ表示する

## 11. 未決事項

人間判断が必要:

* retry 導線をユーザー向けボタンにするか、運用向けリンクだけにするか
