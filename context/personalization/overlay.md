# Personalization Overlay Policy

## 目的

Act の Context Assembly で `topic context` に personalization を重ねる手順を固定する。

## 入力

* `topic_id`
* `workspace_id`
* `token_uid`
* `session_id`
* `AssembleContextInput`

## 合成順序（MUST）

1. `topic` 由来の focus/related/evidence を構築
2. `session_context` で一時制約を適用
3. `profile_context` で出力嗜好を適用
4. `memory_context` で軽い重み補正を適用
5. `PromptBundle` と `diagnostics` を返す

## 禁止事項

* `topic` の factual content を profile/memory で改変しない
* overlay層から Firestore/GCS write を行わない
* Organize pipeline から `act-adk-worker` を経由して overlay を実行しない

## diagnostics 必須項目

* `overlay_applied: boolean`
* `overlay_sources: [session, profile, memory]`
* `conflicts: string[]`

## 参照

* `context/assembly/core.md`
* `act/act-adk-worker/specs/context/context-assembly-execution-profile.md`
* `context/model/topic-model.md`
