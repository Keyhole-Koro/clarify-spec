# Organize Domain

`organize` を機能ルートとして管理する。

## Structure

* `organize/specs/`
  * `README.md`（読み順）
  * `model/`（`topic-model.md`）
  * `pipeline/`（`summary/spec/core/agents/ops`）
* `organize/agents/`
  * Agentごとの個別仕様（`a0/`, `a1/` などの下位specを含む）
* `firestore/`
  * Firestoreスキーマ仕様（ルート共有）
