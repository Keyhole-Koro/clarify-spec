# Context Domain

`context/` は `topic` と直交する「ユーザー文脈（personalization overlay）」の正本仕様を置く。

## Purpose

* `topic` の知識正本を壊さずに、個人差・セッション差を扱う
* Act Context Assembly が参照する overlay ルールを固定する

## Files

* `context/model/topic-model.md`
* `context/model/context-model.md`
* `context/personalization/overlay.md`
* `context/assembly/bundle-schema.md`
* `context/assembly/core.md`
* `context/assembly/implementation.md`

## Boundary

* `topic_id` は知識正本キーのまま維持
* `context` は read-time overlay（上書きではなく補助）
* Act memory は `context/` の personalization 正本とは別で、RunAct 実行中の揮発状態として扱う
