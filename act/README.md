# Act Domain

`act` を機能ルートとして管理する。

## Structure

* `act/specs/`
  * Actの外部仕様（UI状態遷移、Patch適用方針）
* `act/specs/rpc-connect-schema.md`
  * ActのConnect RPC契約（Markdown）
* `act/specs/readiness/`
  * 実装準備仕様（RunAct実装、Connect公開、Frontend統合、E2E、運用設定）
* `act/backend/`
  * LangGraph実行仕様、E2Eテスト計画
* `act/frontend/`
  * フロント受け入れ基準、実装フェーズ資料、AI実装プロンプト
* `act/frontend/specs/`
  * フロント統合実装仕様（RunAct stream受信など）
