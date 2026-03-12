# Usecase: Thinking Stream Visible

## 目的

モデルの思考増分（thought）と通常回答を分離して表示し、ストリーミング中の進行を可視化する。

## トリガー入力

* Ask送信時に `thinking_config.include_thoughts=true`

## 前提条件

* Backendが `stream_parts[].thought` を送出可能
* Frontendが thought/answer バッファを分離管理
* thought 表示の既定は OFF とする

参照:

* `act/specs/contracts/rpc-connect-schema.md`
* `act/specs/contracts/gemini-vertex-response-schemas.md`
* `act/specs/behavior/frontend-stream-integration.md`

## 正常フロー

1. thought有効で `RunAct` 実行
2. streamで `stream_parts[].thought=true/false` を受信
3. フロントが thought と answer を別表示
4. `done=true` で終端
5. 終端後は thought セクションを既定で折りたたむ

## 異常フロー

* thoughtが空でも answer stream は継続
* stream失敗時は部分表示を保持して `error` を表示
* 再実行時に前回 request の thought と混ざらない

## ストリーム期待値（RunActEvent）

* `stream_parts[].thought=true` を1件以上受信（有効時）
* `stream_parts[].thought=false` も受信
* 既存互換として `text_delta` が同居しても破綻しない

## 受け入れ条件（UI / Backend）

* thought表示ON/OFFが切替可能
* thoughtとanswerが視覚的に混ざらない
* thought無効時に `stream_parts[].thought=true` が返らない
* thought は node 本文へ混ざらない
* 終端後は既定で折りたたまれるが、request 単位で保持される

## テスト対応（E2EケースID）

* `UC-THINK-STREAM-01` -> `E2E-THOUGHT-STREAM`
