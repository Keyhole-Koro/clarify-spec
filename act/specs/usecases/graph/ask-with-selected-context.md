# Usecase: Ask With Selected Context

## 目的

複数選択ノードを文脈として `RunAct` に渡し、選択元から新規 `ACT_ROOT` へ依存エッジを引く。

## トリガー入力

* Graph上で複数ノードを選択
* Askフォーム送信
* `request_id`（UUID）

## 前提条件

* `selectedNodeIds` が1件以上
* 選択ノード本文を `user_message` 拡張文脈に含める実装がある

参照:

* `act/specs/behavior/frontend-canvas-phases.md`
* `act/specs/behavior/frontend-stream-integration.md`
* `act/specs/contracts/rpc-connect-schema.md`

## 正常フロー

1. フロントが `selectedNodeIds` と本文文脈を収集
2. `RunActRequest` 送信
3. 新規 `ACT_ROOT` を受信
4. `selectedNodeIds -> newActRoot` の `contextEdges` を追加
5. 送信後に `selectedNodeIds` をクリア

## 異常フロー

* 選択対象が欠損していても送信自体は継続（欠損分はスキップ）
* stream失敗時も既存描画済みノードは保持

## ストリーム期待値（RunActEvent）

* 初期 `upsert` で `ACT_ROOT` を識別できる
* `done` 終端または `error` 終端が明確

## 受け入れ条件（UI / Backend）

* 複数選択が視覚的にハイライトされる
* 生成開始後に依存エッジが描画される
* Backendの構造エッジ（`parent_id`）とUI依存エッジが混同されない

## テスト対応（E2EケースID）

* `UC-ASK-CONTEXT-01` -> `E2E-CONTEXT-EDGE-LINK`
