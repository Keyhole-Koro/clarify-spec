# Context Model

## 目的

`topic` とは別に、ユーザーごとの実行文脈をモデル化する。

## スコープ

* profile context（長期）
* session context（短期）
* memory context（中期）

## エンティティ

### 1. `profile_context`

* key: `(workspace_id, uid)`
* 用途: 言語、表示密度、好みの出力形式、禁止事項
* 更新頻度: 低

### 2. `session_context`

* key: `(workspace_id, uid, session_id)`
* 用途: 直近選択ノード、現在モード、一時制約
* 更新頻度: 高（TTLあり）

### 3. `memory_context`

* key: `(workspace_id, uid)`
* 用途: よく使う比較軸、再利用したい観点、最近の作業傾向
* 更新頻度: 中

## 参照優先順位

1. `topic`（知識正本）
2. `session_context`
3. `profile_context`
4. `memory_context`

補足:

* `context` は `topic` を上書きしない
* 矛盾時は `topic` を優先し、`diagnostics` に衝突を残す
* Organize は personalization を read-only 参照し、MVPでは更新しない

## 受け入れ条件

* `topic_id` 正本を維持したまま personalization が注入できる
* 同一topicでもユーザーごとに異なる補助文脈を使える
