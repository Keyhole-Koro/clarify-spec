# Act Context Specs Guide

このファイルは移行ブリッジ。  
Context正本は `context/` 配下へ移動済み。

## Read Order

1. `context/assembly/bundle-schema.md`
2. `context/assembly/core.md`
3. `context/assembly/implementation.md`

## Scope

* `bundle-schema.md`: 入出力契約
* `core.md`: 段階フローとMUST
* `assembly-implementation.md`: 実装者向けの決定済み手順

## Rule

* ここは read-only Assembly の仕様のみ扱う
* Firestore/GCS への write 規約は `organize/specs/pipeline/*` を参照する
