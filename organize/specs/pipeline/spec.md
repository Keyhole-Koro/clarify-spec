# Organize パイプライン仕様（目次 / 正本エントリ）

Version: 1.1 / schemaVersion: v1

`organize/specs/pipeline/spec.md` は入口のみを担い、詳細は分割先を正本とする。

## 1. 参照順序

1. `context/model/topic-model.md`
2. `context/assembly/bundle-schema.md`
3. `context/assembly/core.md`
4. `organize/specs/pipeline/core.md`
5. `organize/specs/pipeline/agents.md`
6. `organize/specs/pipeline/ops.md`

## 2. 分割方針

* `context/model/topic-model.md` + `context/assembly/bundle-schema.md` + `context/assembly/core.md`: Act/Organize共通の topic / context assembly 正本
* `pipeline-core.md`: Topic/Subscription、Envelope/attributes、冪等・Lease・CAS、DLQ/Ackの共通ルール
* `pipeline-agents.md`: A0〜A7/A5 の入出力・emit・必須競合対策
* `pipeline-ops.md`: Firestore状態の衝突回避、監視運用、整合性チェックポイント

## 2.1 境界ルール（MUST）

* Organize は write path 専任（Firestore/GCS更新）とする
* Organize は `act-adk-worker` を呼び出さない
* Act の `PromptBundle` と Organize の `PipelineBundle` を混同しない
* Context Assembly の共通仕様は参照するが、実行主体は Act と Organize で独立させる

## 3. 用語・命名の固定

* 知識正本キーは `topic_id`
* ルーティングキーは `attributes.type` に固定
* 競合対策の3点は `event_ledger` / `leases` / `version(CAS)` に固定
* Subscription 名は短縮形 `sub-a0` 〜 `sub-a7` / `sub-a5` に固定
* `bundle.created` の fan-out は `sub-a6` と `sub-a3` の2購読で固定

## 4. 変更ルール

* 仕様追加は必ず分割先の該当ファイルに行う
* ここには方針や参照関係のみ記載し、詳細仕様は書かない
* `pipeline-summary.md` は常に分割先に整合する要約として維持する
