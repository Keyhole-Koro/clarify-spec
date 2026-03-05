# Usecase: Run Act From Node Action

## 目的

ノード内 `actions[].on_click.type=run_act` から派生 `RunAct` を実行し、起点ノードとの関係を可視化する。

## トリガー入力

* ノードアクションボタン（run_act）をクリック

## 前提条件

* ノードに `actions[]` が存在
* クリック時に `anchor` へ起点ノードIDを設定できる

参照:

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/act-flow.md`

## 正常フロー

1. ユーザーが run_act ボタンを押す
2. フロントが起点ノードIDを `anchor` に入れて送信
3. バックエンドが派生 `ACT_ROOT` を返す
4. 起点ノード -> 派生ノードの依存エッジを追加
5. `done=true` で終端

## 異常フロー

* action payload 不正は `INVALID_ARGUMENT`
* 中途失敗時は `error` 終端、既存Draftは保持

## ストリーム期待値（RunActEvent）

* `patch_ops` が1回以上返る
* `ACT_ROOT` が派生結果として識別可能
* `done/error` 排他

## 受け入れ条件（UI / Backend）

* ノードアクションから追加質問フローが開始できる
* 派生元をキャンバス上で線として追跡できる
* Backendは anchor 由来の文脈をプロンプトに反映する

## テスト対応（E2EケースID）

* `UC-RUNACT-NODE-01` -> `E2E-RUNACT-ANCHOR-LINK`
