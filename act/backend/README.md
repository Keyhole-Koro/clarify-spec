# Act Backend

Go実装ディレクトリ（Connect RPC / Vertex AI / stream orchestration）。

## Role

* `act/specs/contracts/*` を契約正本として実装する
* `act/specs/behavior/*` の状態遷移と制約を満たす
* `act/specs/quality/*` のテスト/運用基準を満たす

## Source Of Truth

* Contracts: `act/specs/contracts/rpc-connect-schema.md`
* Runtime behavior: `act/specs/behavior/act-langgraph-runtime.md`
* Connect behavior: `act/specs/behavior/connect-server.md`
* Test plan: `act/specs/quality/act-e2e-test-plan.md`

## Implementation Scope

* 実装コード（Go）
* テストコード（unit/integration/e2e）
* ローカル実行手順

仕様そのものは `act/specs/` に追加・更新する。
