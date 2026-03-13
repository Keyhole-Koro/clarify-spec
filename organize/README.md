# Organize Domain

`organize` を機能ルートとして管理する。

## Structure

* `organize/readable/`
  * `README.md`（人間向けの読み順）
  * `organize-guide.md`（役割、Agent、入出力、Actとの境界の解説）
* `organize/specs/`
  * `README.md`（読み順）
  * `pipeline/`（`summary/spec/core/agents/ops`）
* `organize/agents/`
  * Agentごとの個別仕様
  * 各 Agent は `a0/`, `a1/`, `a2/` のようなディレクトリ単位で管理
  * 各 Agent の入口は `README.md`
* 共有正本
  * Topic モデル: `context/model/topic-model.md`
  * Firestore スキーマ: `firestore/schema.md`
