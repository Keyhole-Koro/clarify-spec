# A6 BundleDescriptionAgent 仕様

## 1. 責務

* `bundle.created` を受け取り、Bundle の内容を人間が読める説明 HTML に変換する
* UI での差分プレビューに使用される

## 2. I/O

* Input: `bundle.created`
* Output: `mind/bundle_desc/{bundleId}/v{n}.html`, `workspaces/{workspaceId}/topics/{topicId}/pipelineBundles/{bundleId}.descRef`
* Emit: `bundle.described`（任意）

## 3. LLM モデル

* **Gemini Flash** — 要約 HTML 生成。創造性は不要

## 4. 処理フロー

1. `workspaces/{workspaceId}/topics/{topicId}/pipelineBundles/{bundleId}` を Firestore から読む
2. claims を種類ごとにグループ化する
3. LLM で自然言語の説明文を生成する
4. テンプレート HTML に埋め込む
5. GCS に保存する: `mind/bundle_desc/{bundleId}/v{n}.html`

## 5. LLM プロンプト

> 以下のナレッジ更新内容を、研究者向けの簡潔な更新レポートとして日本語の説明文にまとめてください。
>
> **変更内容:**
> - 新規ノード候補: {proposedNodes.length}件
> - 新規関係候補: {proposedEdges.length}件
> - 統合されたClaim: {claims.length}件
>
> **ルール:**
> - 箇条書きで構造化する
> - 確信度の低い項目は「（要検証）」と注記する
> - 200語以内

## 6. HTML の構造

記事形式とし、以下の3パートで構成する:

1. **ヘッダー** — タイトル（topic名）、日時、claim数バッジ
2. **要約セクション** — LLM 生成文
3. **詳細セクション** — ノード候補一覧、関係候補一覧

CSS はインラインまたはシステム共通スタイルシートを参照する。

## 7. Idempotency / 競合対策

* ledger: `type:bundle.created/bundleId:{bundleId}/purpose:desc`
* `descRef.version` CAS
