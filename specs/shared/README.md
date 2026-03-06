# Shared Specs Guide

`specs/shared` は Act / Organize が共同で参照する正本仕様を置く。

## Read Order

1. `specs/shared/topic-model.md`
2. `specs/shared/context-bundle-schema.md`
3. `specs/shared/context-assembly-core.md`

## Rules

* ここには実装言語依存の記述を置かない
* Firestore/GCS の write は Organize 側仕様、read-only assembly は Act 側仕様から参照する
* 変更時は `act/specs/*` と `organize/specs/*` の参照先を同時更新する
