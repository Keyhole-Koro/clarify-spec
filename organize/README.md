# Organize Domain

`organize` を機能ルートとして管理する。

## Structure

* `organize/readable/`
  * `README.md`（人間向けの読み順）
  * `organize-guide.md`（役割、Agent、入出力、Actとの境界の解説）
* `organize/specs/`
  * `README.md`（読み順）
  * `model/`（topic正本は `context/model/topic-model.md` を参照）
  * `pipeline/`（`summary/spec/core/agents/ops`）
* `organize/agents/`
  * Agentごとの個別仕様（`a0/`, `a1/` などの下位specを含む）
* `firestore/`
  * Firestoreスキーマ仕様（ルート共有）
