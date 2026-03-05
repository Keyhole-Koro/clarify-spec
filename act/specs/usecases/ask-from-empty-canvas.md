# Usecase: Ask From Empty Canvas

## 目的

空のキャンバスから最初の `RunAct` を実行し、最低1つの `ACT_ROOT` を描画する。

## トリガー入力

* Askフォームで `user_message` を送信
* `anchor_node_ids=[]`
* `context_node_ids=[]`

## 前提条件

* `tree_id`, `workspace_id`, `uid` が有効
* フロントは `RunAct` stream購読可能

参照:

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/behavior/act-flow.md`
* `act/specs/behavior/frontend-stream-integration.md`

## 正常フロー

1. フロントが `RunActRequest` を送信
2. バックエンドが `patch_ops(upsert)` を返す
3. フロントがノードを描画
4. 必要に応じて `append_md` が追記される
5. 最終イベントで `done=true`

## 異常フロー

* 入力不正時は初手 `error.code=INVALID_ARGUMENT` で終端
* 外部依存障害時は `error.code=UNAVAILABLE` または `DEADLINE_EXCEEDED`

## ストリーム期待値（RunActEvent）

* `patch_ops` に `upsert` が1回以上含まれる
* `upsert.blocks` に `kind=ACT_ROOT` を最低1件含む
* 終端は `done` または `error` のどちらか一方

## 受け入れ条件（UI / Backend）

* UIが空状態から破綻せず1ノード以上表示できる
* `append_md` は置換でなく追記になる
* Backendは `upsert/append_md` 以外を返さない

## テスト対応（E2EケースID）

* `UC-ASK-EMPTY-01` -> `E2E-STREAM-SUCCESS`
