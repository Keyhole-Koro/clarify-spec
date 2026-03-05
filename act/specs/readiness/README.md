# Act 実装準備仕様

このディレクトリは、ハッカソン当日に実装するための準備仕様をまとめる。

## Documents

* `runact-implementation.md`
  * RunAct本体（LangGraph）
* `connect-server.md`
  * Connect RPC公開境界
* `act/frontend/specs/frontend-stream-integration.md`
  * フロント受信統合
* `e2e-test-strategy.md`
  * 最小E2E戦略
* `operations-config-observability.md`
  * 設定・ログ・メトリクス運用

## 共通原則

* `act/specs/rpc-connect-schema.md` の契約を優先
* Firestore直接更新は禁止（永続化は Organize）
* `PatchOp` は `upsert` / `append_md` のみ
