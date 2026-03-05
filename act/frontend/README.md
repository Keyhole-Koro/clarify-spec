# Act Frontend

フロント実装ディレクトリ（canvas stream UI / interaction）。

## Role

* `act/specs/contracts/*` のストリーム契約を受信して表示する
* `act/specs/behavior/*` のUIフローを実装する
* `act/specs/quality/*` の受け入れ・E2E観点を満たす

## Source Of Truth

* RPC contract: `act/specs/contracts/rpc-connect-schema.md`
* Stream integration: `act/specs/behavior/frontend-stream-integration.md`
* Canvas phases: `act/specs/behavior/frontend-canvas-phases.md`
* Acceptance: `act/specs/quality/frontend-stream-acceptance.md`

## Implementation Scope

* 実装コード（Next.js/ReactFlow）
* UIテスト・受け入れ確認
* 開発用プロンプト/作業メモ

仕様そのものは `act/specs/` に追加・更新する。
