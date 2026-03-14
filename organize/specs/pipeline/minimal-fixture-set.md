# Minimal Fixture Set

Version: 1.0

目的: `run-phase` と golden test のための最小 fixture セットを固定する。

## 1. 対象

first pass では次の 4 phase を必須とする。

* `a1`
* `topic-resolution`
* `a3b`
* `a3`

## 2. ディレクトリ構成

```text
fixtures/
  a1/
  topic-resolution/
  a3b/
  a3/
```

各 fixture case:

```text
fixtures/<phase>/<case>/
  input.json
  expected.json
  notes.md
```

## 3. A1 Fixture

必須 case:

* `single-claim`
* `multi-claim-in-one-sentence`
* `bullet-list`

確認項目:

* atom 数
* reject 数
* `kind`
* `confidence`

## 4. TopicResolver Fixture

必須 case:

* `clear-existing-topic`
* `borderline-create-new`

確認項目:

* `decision`
* `resolvedTopicId`
* `confidence`

## 5. A3b Fixture

必須 case:

* `simple-draft-delta`

確認項目:

* bundle shape
* normalized claims
* emitted events

## 6. A3 Fixture

必須 case:

* `simple-bundle-apply`

確認項目:

* node diff
* edge diff
* outline preview
* `atom.reissued`

## 7. Input Shape 方針

* `input.json` は phase の最小入力だけを持つ
* fixture は deterministic な値を使う
* `workspaceId`, `topicId`, `traceId` は固定値でよい

## 8. Expected Shape 方針

`expected.json` は `run-phase` 共通出力 shape に従う。

最低限含む:

* `result`
* `events`
* `firestoreWrites`
* `gcsWrites`

## 9. 命名ルール

* case 名は動作を説明する短い kebab-case
* phase 名は `run-phase` の `phase` 引数と一致させる

## 10. MUST

* 各 phase は最低 1 case を持つ
* `a1` と `topic-resolution` は最低 2 case 以上持つ
* `expected.json` が無い case は valid fixture とみなさない
