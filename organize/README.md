# Organize Domain

`organize` を機能ルートとして管理する。

## Structure

* `organize/specs/`
  * Pub/Subパイプライン仕様（中核、agent、運用、要約）
* `organize/agents/`
  * Agentごとの個別仕様（`a0/`, `a1/` などの下位specを含む）
* `organize/data/`
  * Firestoreスキーマ仕様
