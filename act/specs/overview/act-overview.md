# Act Overview

## 目的

Actドメインの読み順と責務境界を定義する。

## 読み順

1. `act/specs/overview/act-architecture.md`
2. `context/README.md`
3. `act/specs/contracts/rpc-connect-schema.md`
4. `act/act-api/specs/runtime/runact-implementation.md`
5. `act/act-api/specs/runtime/adk-service-integration.md`
6. `act/act-adk-worker/specs/act-adk-runtime.md`
7. `act/specs/usecases/README.md`
8. `act/specs/quality/e2e-test-strategy.md`

## 責務境界

* Go Act API: Connect契約、認証認可、stream返却
* ADK Worker: context assembly、tool orchestration、model実行
* Organize: write path（正本更新）

## MUST

* `RunAct` 契約は後方互換を保つ
* ADK導入時も `PatchOp` 制約（`upsert`/`append_md`）を維持する
* read-only境界（ADK/Assembly）を破らない
