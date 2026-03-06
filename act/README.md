# Act Domain

`act` を機能ルートとして管理する。

## Structure

* `act/specs/README.md`
  * 仕様の粒度ルールと命名ルール
* `act/specs/overview/`
  * 全体像・文書構成ガイド
* `act/specs/context/`
  * Context Assembly と Bundle 契約
* `act/specs/contracts/`
  * RPC契約・Gemini/Vertexレスポンス契約
* `act/specs/behavior/`
  * 実行挙動仕様（RunAct、ADK連携、Connect公開、Frontend統合）
* `act/specs/usecases/`
  * ユーザー操作起点のシナリオ仕様
* `act/specs/quality/`
  * テスト戦略・LLM API試験・運用設定
* `act/act-api/`
  * Go API 実装ルート（Connect公開、auth/authz、stream）
* `act/act-adk-worker/`
  * Python ADK Worker 実装ルート（assembly、model、normalization）
* `act/frontend/`
  * フロント実装ルート（UIコード、実装補助資料）
