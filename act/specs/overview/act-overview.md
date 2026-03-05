# Act Overview

## 1. 目的

Actの責務・契約・挙動・品質基準を一貫した粒度で参照できるようにする。

## 2. スコープ / 非スコープ

* スコープ: ActのRPC契約、実行挙動、Frontend統合、品質基準
* 非スコープ: Organize永続化ロジックの詳細実装

## 3. 前提 / 参照

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
* `act/specs/overview/act-architecture.md`
* `act/specs/behavior/access-control-middleware.md`
* `act/specs/behavior/act-langgraph-runtime.md`
* `act/specs/usecases/README.md`
* `act/specs/quality/act-e2e-test-plan.md`

## 4. 仕様構成

* `contracts/`: 入出力契約（変更影響が大きい）
* `behavior/`: 状態遷移と実行ルール
* `usecases/`: ユーザー操作起点の受け入れ導線
* `quality/`: テスト戦略・運用設定

## 5. エラー / 例外

* `RunAct` の終端は `done` / `error` 排他
* 外部依存失敗はエラーコードへ正規化

## 6. 完了条件（DoD）

* contracts / behavior / quality の各文書が相互参照可能
* 実装時に迷わない最小導線がある

## 7. 未決定事項

* Deep Research本番運用時の時間上限
* thought表示のUI標準
