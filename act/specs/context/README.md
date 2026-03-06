# Act Context Specs Guide

Act実行前の文脈生成に関する正本をまとめる。

## Read Order

1. `act/specs/context/bundle-schema.md`
2. `act/specs/context/core.md`
3. `act/specs/context/implementation.md`

## Scope

* `bundle-schema.md`: 入出力契約
* `core.md`: 段階フローとMUST
* `implementation.md`: 実装者向けの決定済み手順

## Rule

* ここは read-only Assembly の仕様のみ扱う
* Firestore/GCS への write 規約は `organize/specs/pipeline/*` を参照する
